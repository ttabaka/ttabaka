# TrueNAS: Suspended Mirror Pool from NVMe Power Management (Quick Reference)

**Symptom:** Pool alert goes DEGRADED (disk "removed") → then SUSPENDED ("faulted in response to IO failures"), usually during a scrub. Apps/SMB/catalog errors follow — these are downstream symptoms, not separate problems.

**Key tell:** If it's a real single-disk failure, a mirror stays DEGRADED and keeps working. Full SUSPENSION means the *other* mirror disk is also faulted — check both, not just the one flagged.

---

### 1. Check the pool
```bash
zpool status -v [poolname]
```
Look for errors on **both** disks in the vdev, not just the one marked REMOVED.

### 2. Check kernel logs for the cause
```bash
sudo dmesg -T | grep -iE 'nvme|pcie|aer|reset' | tail -150
```
If `Operation not permitted`:
```bash
journalctl -k --since "-3 hours" | grep -iE 'nvme|pcie|reset'
```

**Confirms this issue** if you see, on one or more NVMe controllers:
```text
nvme nvmeX: controller is down; will reset: CSTS=0xffffffff...
nvme 0000:XX:00.0: Unable to change power state from D3cold to D0, device inaccessible
nvme nvmeX: Disabling device after reset failure: -19
```
This is a PCIe power-saving bug, not necessarily a dying drive — especially if it hits two controllers at once.

### 3. Fix: disable NVMe/PCIe power saving
No GUI field for this — use Shell:
```bash
# Check existing value first
midclt call system.advanced.config | grep kernel_extra_options

# Set the parameters
midclt call system.advanced.update '{ "kernel_extra_options": "nvme_core.default_ps_max_latency_us=0 pcie_aspm=off pcie_port_pm=off" }'
```
Also disable ASPM/PCIe power management in BIOS if possible.

### 4. Reboot
```bash
shutdown -r now
```
Safe even while suspended (ZFS won't corrupt data). If it hangs 15+ min, hard power cycle. **A full power-off + flush may be needed** — a soft reboot alone doesn't always clear a wedged NVMe controller.

### 5. Verify
```bash
zpool status -v [poolname]      # both disks ONLINE, 0 errors (or minor residual CKSUM from resilver = fine)
zpool clear [poolname]          # reset counters, then watch next scrub stays clean
```

### 6. If Apps shows "Service Not Configured"
Apps → **Settings → Choose Pool** → select the same pool → **Save** (skip "Migrate existing applications" — you're not moving pools).

---
**Root cause:** NVMe drives dropped into deep PCIe power state (D3cold) under scrub load and failed to wake up; kernel disabled them, ZFS saw devices vanish. Fixed by disabling ASPM/power-saving at the kernel level.
