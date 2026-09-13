# TrueNAS: Diagnosing a Suspended Mirror Pool Caused by NVMe Power Management (ASPM/D3cold)

> **TL;DR:** A scrub triggers a PCIe power-management bug that knocks out both disks in an NVMe mirror at once. ZFS reports it as a dead drive, but it's usually just power-saving states misbehaving. Disable ASPM, reboot (hard cycle if needed), verify.
>
> **Update (Sep 13, 2026):** The fix has held — no recurrence of the original suspension. A follow-up alert one week later turned out to be a different, much milder pattern: a single self-corrected checksum error on one of the *same two* NVMe drives, left over from the hard power cycles used to recover from the original incident, not a new failure. See [Update — Follow-up Alert](#update--follow-up-alert-sep-13-2026) at the end for the full diagnostic trail and the general lesson it produced.

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
- [Update — Follow-up alert (Sep 13, 2026)](#update--follow-up-alert-sep-13-2026)

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
>
> **Comment:** the corollary matters just as much — a pool that *stays* ONLINE with a single, already self-corrected error on one device is a much milder signal than this pattern, and shouldn't automatically be treated as a recurrence of this same bug. See the [Update](#update--follow-up-alert-sep-13-2026) below for exactly this situation.

## Step 1 — Check pool and disk status

```bash
zpool status -v [poolname]
```

Look at:
- How many disks are in the affected vdev (usually `mirror-N`)
- Whether more than one disk shows errors, not just the one reported as REMOVED
- READ/WRITE/CKSUM counts on the "surviving" disk — non-zero counts there are a red flag, even if it still shows ONLINE

> **Comment:** also note the `scan:` line. If a scrub/resilver already ran and completed with the errors corrected, ZFS has already fixed the immediate problem before you even looked — the remaining question is *why* the error happened, not whether the pool is currently healthy.

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

> **Comment:** the absence of this signature is just as diagnostic as its presence. If `zpool status` shows an error but dmesg is completely clean around that time — no resets, no AER, no D3cold messages — don't reach for this fix. Check whether the error is instead explained by something that already happened and finished (see the Update below) before concluding it needs a kernel-parameter change.

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

> **Comment — confirmed on XenoHOS as of Sep 13, 2026:** the saved `kernel_extra_options` string on this system is now:
> ```
> nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off ahci.mobile_lpm_policy=1
> ```
> The `ahci.mobile_lpm_policy=1` flag was added on top of the original three, to disable link power management on the SATA/AHCI side as well (this box also has 8 SATA SSDs in a separate raidz2 pool, `STORAGE`). Re-run the `grep` command above any time a new alert comes in, to confirm this hasn't been lost to a firmware update or config rollback before assuming a new alert is a fresh instance of this bug.

## Step 4 — Reboot (and expect it may need to be a hard power cycle)

```bash
shutdown -r now
```

Notes:
- Safe to do even with the pool suspended — ZFS's copy-on-write design means nothing is left half-written.
- If shutdown hangs for 15+ minutes with no sign of progress, it's very likely stuck trying to cleanly export the already-suspended pool. A hard power cycle at that point is reasonable and should be safe.
- **A soft reboot may not be enough.** If the NVMe controllers are truly wedged, only a full power-off + power flush (not just an OS restart) may fully re-initialize the PCIe bus and clear them. If drives are still missing from `nvme list` / `zpool import` after a normal reboot, try a full shutdown, then physically power off, wait, and power back on.

> **Comment:** the hard power cycles used here are not free of side effects. They can leave a torn/incomplete write on a block that was being written at the moment of the cut — which shows up days later as a single self-corrected checksum error on that same drive, once a scrub happens to touch that block. That's expected cleanup, not evidence the fix didn't work or that the drive is failing. See the Update below for a concrete example of this on XenoHOS.

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

> **Comment:** the affected mirror is the `SSDs` pool on this system — specifically `nvme0n1p1` + `nvme1n1p1`. A third NVMe controller (`nvme2n1`) exists on the same box but is used elsewhere; it was not involved in this incident. Worth remembering which physical drives are which the next time an alert on `SSDs` comes in.

## Quick checklist for next time

- [ ] `zpool status -v [pool]` — check all disks, not just the flagged one
- [ ] `sudo dmesg -T | grep -iE 'nvme|pcie|aer|reset'` (or `journalctl -k`) — look for the D3cold/reset signature
- [ ] If confirmed: set kernel params via `midclt call system.advanced.update`
- [ ] Reboot; hard power cycle if it hangs or drives don't come back
- [ ] `zpool status -v [pool]` again — confirm ONLINE, clear errors, watch next scrub
- [ ] Reconfigure Apps pool if it shows "Not Configured"
- [ ] **If the alert is milder** (pool stayed ONLINE, one already-corrected error, clean dmesg) — don't assume a recurrence. Resolve the GUID to a device, check its own SMART/NVMe health log, and check `unsafe_shutdowns` against any recent hard power cycle before concluding anything. See the Update below.

---

## Update — Follow-up alert (Sep 13, 2026)

Exactly one week after the original incident, a new HexOS alert came in on the same box:

> Pool `SSDs` state is ONLINE: One or more devices has experienced an unrecoverable error. An attempt was made to correct the error. Applications are unaffected.

This looked superficially similar (same pool, an NVMe-related error) but turned out to be a distinct, much milder pattern. Walking through it end to end, because the diagnostic trail is a useful template:

**1. Pool status:**

```
  pool: SSDs
 state: ONLINE
status: One or more devices has experienced an unrecoverable error. An
        attempt was made to correct the error. Applications are unaffected.
  scan: resilvered 154M in 00:00:05 with 0 errors on Sun Sep 13 13:18:50 2026
config:
        NAME                                      STATE     READ WRITE CKSUM
        SSDs                                      ONLINE       0     0     0
          mirror-0                                ONLINE       0     0     0
            3a9aee7f-...                          ONLINE       0     0     0
            a59a2107-...                          ONLINE       0     0     1
errors: No known data errors
```

Pool stayed fully `ONLINE` the entire time — no DEGRADED, no SUSPENDED. One mirror member logged a single `CKSUM` error, already resilvered and corrected. This alone is a much softer signal than last week's full suspension.

**2. Kernel logs:** a completely clean, unremarkable boot log for all three NVMe controllers and all eight SATA/AHCI drives. No `controller is down`, no `D3cold`, no AER/reset messages anywhere. The specific signature that confirmed the original bug was entirely absent this time.

**3. Identify the actual physical drive.** `zpool status` only shows partition GUIDs, so:

```bash
$ ls -l /dev/disk/by-partuuid/ | grep a59a2107
a59a2107-... -> ../../nvme1n1p1
$ ls -l /dev/disk/by-partuuid/ | grep 3a9aee7f
3a9aee7f-... -> ../../nvme0n1p1
```

This confirmed the `SSDs` mirror is exactly `nvme0n1p1` + `nvme1n1p1` — the *same two drives* involved in the original suspension. That overlap is what made it worth digging further rather than dismissing the alert outright.

**4. Check the drive's own health, independent of ZFS:**

```bash
$ sudo nvme smart-log /dev/nvme1n1
critical_warning  : 0
available_spare   : 100%
percentage_used   : 0%
media_errors      : 0
num_err_log_entries: 0
unsafe_shutdowns  : 3
```

Completely clean — the drive's own controller isn't logging any hardware fault. But `unsafe_shutdowns: 3` stood out: that count plausibly traces back to the hard power cycles performed last week to clear the wedged NVMe controllers during recovery.

**5. Confirm the ASPM fix is still in place:**

```bash
$ midclt call system.advanced.config | grep kernel_extra_options
"kernel_extra_options": "nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off ahci.mobile_lpm_policy=1"
```

Still applied, ruling out "the mitigation silently reverted" as an explanation.

**6. Sanity-check the other pool on the box:**

```
pool: STORAGE
 state: ONLINE
 scan: scrub in progress ... 0B repaired, 13.51% done
errors: No known data errors
```

The separate 8-drive SATA raidz2 pool was mid-scrub and fully healthy — confirming the issue was isolated to that one NVMe mirror, not systemic.

**Conclusion:** this was not a recurrence of the ASPM/D3cold bug (no reset signature, mitigation confirmed active) and not a sign of drive failure (SMART/NVMe health completely clean). The most likely explanation: one of last week's hard power cycles interrupted a write in progress on `nvme1n1`, leaving one block with mismatched data. Today's scrub found the checksum mismatch, pulled the correct copy from the mirror, and self-healed it automatically — ZFS working as designed. Action taken: `zpool clear SSDs`, no kernel changes, no replacement. If this same drive accumulates further CKSUM errors on future scrubs, or `media_errors`/`num_err_log_entries` start climbing, that would change the assessment — a single, cleanly-explained, already-corrected error does not.

**General lesson for next time:** a milder alert on the *same drives* that were involved in a prior incident deserves this specific check — resolve the GUID to a device, pull its own health log, and see whether a recent hard reboot/power cycle can explain a one-off torn write — before either dismissing it as unrelated noise or over-escalating it to a replacement decision.
