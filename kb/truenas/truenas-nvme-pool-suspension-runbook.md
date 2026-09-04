# TrueNAS: Diagnosing a Suspended Mirror Pool Caused by NVMe Power Management (ASPM/D3cold)

> **TL;DR:** A scrub triggers a PCIe power-management bug that knocks out both disks in an NVMe mirror at once. ZFS reports it as a dead drive, but it's usually just power-saving states misbehaving. Disable ASPM, reboot (hard cycle if needed), verify.

## Contents

- [Symptom pattern](#symptom-pattern)
- [Step 1 — Check pool and disk status](#step-1--check-pool-and-disk-status)
- [Step 2 — Check kernel logs](#step-2--check-kernel-logs-for-the-actual-hardware-event)
- [Step 3 — Apply the fix](#step-3--apply-the-fix-kernel-boot-parameters)
- [Step 4 — Reboot](#step-4--reboot-and-expect-it-may-need-to-be-a-hard-power-cycle)
- [Step 5 — Verify recovery](#step-5--verify-recovery)
- [Step 6 — Recover Apps](#step-6--recover-apps-if-applicable)
- [Root cause summary](#root-cause-summary-for-the-record)
- [Quick checklist](#quick-checklist-for-next-time)

---

## Symptom pattern

You'll likely see this sequence of alerts, in this order:

1. **Degraded pool, disk removed**
   > Pool `[X]` state is DEGRADED: One or more devices has been removed by the administrator. Sufficient replicas exist for the pool to continue functioning in a degraded state.

2. **Pool suspends shortly after** (often during a scheduled scrub)
   > Pool `[X]` state is SUSPENDED: One or more devices are faulted in response to IO failures.

3. **Cascading failures downstream of the pool being unavailable** — these are symptoms, not separate problems:
   - `zfs_open() failed - cannot open '[X]/[dataset]': pool I/O is currently suspended`
   - Apps/Docker jobs failing (`app.update`, `catalog.sync`)
   - SMB shares showing path-related errors
   - "Apps Service Not Configured" after reboot

> **Key diagnostic clue:** if only one disk in a mirror had actually failed, the pool would stay DEGRADED and keep serving I/O — it would not SUSPEND. A full suspension means the *other* mirror member is also faulted/erroring, not just the one flagged as REMOVED.

## Step 1 — Check pool and disk status

```bash
zpool status -v [poolname]
```

Look at:
- How many disks are in the affected vdev (usually `mirror-N`)
- Whether more than one disk shows errors, not just the one reported as REMOVED
- READ/WRITE/CKSUM counts on the "surviving" disk — non-zero counts there are a red flag, even if it still shows ONLINE

## Step 2 — Check kernel logs for the actual hardware event

```bash
sudo dmesg -T | grep -iE 'nvme|ata|scsi|pcie|aer|reset' | tail -150
```

If `dmesg` gives `Operation not permitted`, use:

```bash
journalctl -k --since "-3 hours" | grep -iE 'nvme|ata|scsi|pcie|aer|reset'
```

**What confirms this specific issue** — look for this exact pattern, possibly on more than one NVMe controller at slightly different timestamps:

```text
nvme nvmeX: controller is down; will reset: CSTS=0xffffffff, PCI_STATUS=0xffff
nvme nvmeX: Does your device have a faulty power saving mode enabled?
nvme nvmeX: Try "nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off" and report a bug
nvme 0000:XX:00.0: Unable to change power state from D3cold to D0, device inaccessible
nvme nvmeX: Disabling device after reset failure: -19
```

If you see this identical signature on **two different NVMe controllers/PCIe addresses**, that's strong evidence of a shared power-management bug affecting both mirror members at once — not two coincidentally failing drives, and not necessarily bad hardware at all.

## Step 3 — Apply the fix (kernel boot parameters)

There is no dedicated GUI field for this in TrueNAS SCALE/Community Edition — it's set via a middleware command in Shell.

```bash
# Check for any existing kernel_extra_options first (don't overwrite blindly)
midclt call system.advanced.config | grep kernel_extra_options

# Set the parameters (include any existing options in this string too, if step above wasn't empty)
midclt call system.advanced.update '{ "kernel_extra_options": "nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off" }'

# Verify it saved
midclt call system.advanced.config | grep kernel_extra_options
```

Also check the BIOS/UEFI for ASPM / PCIe power management settings and disable them there too if possible — it's a more durable fix than the OS-level workaround alone.

## Step 4 — Reboot (and expect it may need to be a hard power cycle)

```bash
shutdown -r now
```

Notes:
- Safe to do even with the pool suspended — ZFS's copy-on-write design means nothing is left half-written.
- If shutdown hangs for 15+ minutes with no sign of progress, it's very likely stuck trying to cleanly export the already-suspended pool. A hard power cycle at that point is reasonable and should be safe.
- **A soft reboot may not be enough.** If the NVMe controllers are truly wedged, only a full power-off + power flush (not just an OS restart) may fully re-initialize the PCIe bus and clear them. If drives are still missing from `nvme list` / `zpool import` after a normal reboot, try a full shutdown, then physically power off, wait, and power back on.

## Step 5 — Verify recovery

```bash
zpool status -v [poolname]
```

You want to see:
- Both/all disks `ONLINE`
- READ/WRITE/CKSUM at `0`, or only small residual CKSUM counts from the resilver self-healing a few blocks (this is normal and not a sign of ongoing failure)
- `errors: No known data errors`

Then:

```bash
zpool clear [poolname]
```

...to reset error counters to a clean baseline going forward. Let the next scrub run to completion — if errors stay at 0, that confirms the ASPM fix resolved it and the drives themselves are healthy.

## Step 6 — Recover Apps if applicable

If Apps/Docker was pointed at the affected pool, it may show **"Apps Service Not Configured"** after the outage:

1. Go to **Apps** in the left nav.
2. Click **Settings > Choose Pool**.
3. Select the same pool the apps were already using.
4. Click **Save**. Do **not** choose "Migrate existing applications" — you're re-pointing at the same pool, not moving to a new one.
5. Confirm under **Apps > Installed** that existing apps and their data came back, rather than fresh/empty installs.

If the catalog also needs a manual refresh: **Apps > Manage Catalogs > [catalog name] > Refresh**.

## Root cause summary (for the record)

This was not a failing drive. Two NVMe controllers in the same mirror vdev independently dropped into a deep PCIe power state (D3cold) and failed to wake back up when the scheduled scrub's sustained I/O hit them, causing the kernel to disable both "unresponsive" devices. ZFS saw this as devices disappearing — one showed as REMOVED, the other accumulated read/write errors before also going offline — which triggered pool suspension. The fix disables the OS/PCIe-level power saving states responsible (`nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off`), and a full power cycle was needed to clear the already-wedged controllers.

## Quick checklist for next time

- [ ] `zpool status -v [pool]` — check all disks, not just the flagged one
- [ ] `sudo dmesg -T | grep -iE 'nvme|pcie|aer|reset'` (or `journalctl -k`) — look for the D3cold/reset signature
- [ ] If confirmed: set kernel params via `midclt call system.advanced.update`
- [ ] Reboot; hard power cycle if it hangs or drives don't come back
- [ ] `zpool status -v [pool]` again — confirm ONLINE, clear errors, watch next scrub
- [ ] Reconfigure Apps pool if it shows "Not Configured"
