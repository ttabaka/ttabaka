# UGREEN NAS – ASPM/PCIe & SATA Power Management Settings

Reference checklist for disabling ASPM and related link power management
settings on UGREEN NAS units, to address PCIe/SATA link power state issues
(drive dropouts, resets, etc.). Covers two different units — one fixed at
the BIOS level, one where the BIOS doesn't expose these settings at all and
the fix had to be done at the OS level instead.

---

## NAS #1 — BIOS-level fix (AMI Aptio, Alder Lake-N–style reference board)

### Entering the BIOS

- Reboot the NAS and tap `Ctrl+F2` (or `Ctrl+F12` on some units) repeatedly
  right at power-on. There is no on-screen prompt for this.
- Alternative: enable SSH in UGOS, then run:

  ```sh
  sudo systemctl reboot --firmware-setup
  ```

  This sets an EFI flag so the next reboot drops straight into BIOS setup.

### Settings to change

### Chipset → System Agent (SA) Configuration → PCI Express Configuration

Per-port ASPM control. Enter each of the following and set **ASPM** to
**Disabled**:

- [x] PCIE M.2 SSD1 — confirmed already `Disabled`
- [ ] PCI Express Root Port 1
- [ ] PCI Express Root Port 2
- [ ] PCI Express Root Port 3

### Advanced → SATA Configuration

- [ ] **Aggressive LPM Support** → set to `Disabled` (was `Enabled`)
- Leave `SATA Mode Selection` on `AHCI` and `SATA Test Mode` on `Disabled` — correct as-is.

### Still to resolve

- **BIOS watchdog timer** — not yet located this session. Forces a
  reboot if UGOS isn't detected for a couple of minutes; find and disable
  it (often under Advanced, sometimes a Super I/O/EC-related submenu) so
  it doesn't interrupt future BIOS sessions.
- All three Serial ATA ports showed `No Install`. If actual drive bays
  were expected here and didn't show up, they're likely on a separate
  controller (PCIe-attached SATA controller or backplane/expander) with
  its own ASPM/LPM settings menu — check for that separately.

### If problems persist after the above

- **Chipset → System Agent (SA) Configuration → PCI Express Configuration →
  SA PCIe LTR Configuration → Force LTR Override** → `Enabled`
  (forces the platform to ignore bad latency-tolerance signaling from a device)
- **PCI Express Clock Gating** / **PCI Express Power Gating** on the
  M.2 SSD1 port → `Disabled`
- CPU C-state limit, under `CPU Configuration` or `Power & Performance`

### Notes

- BIOS: AMI Aptio, version 2.22.1287, Alder Lake-N–style reference
  implementation (evidenced by TCSS Platform Setting / VTIO / PMIC
  charging entries under Advanced → Platform Settings, which are
  leftovers from a laptop/tablet reference design and not relevant to
  the NAS itself — don't touch those).
- Save changes with `F10` before exiting.

---

## NAS #2 — OS-level fix (TrueNAS SCALE / HexOS, hostname `XenoHOS`)

This unit's BIOS (AMI Aptio) Chipset tab exposes only **Onboard Devices**
(per-slot/NIC enable-disable) — no PCH-IO, System Agent, or PCI Express
Configuration submenu at all. None of the BIOS steps from NAS #1 apply
here; everything had to be done at the OS level instead.

- Hardware: Intel Alder Lake-P–based (per `lspci`), reused laptop/mobile
  silicon, same pattern as NAS #1's reference-design BIOS.
- OS: TrueNAS SCALE 25.10.7 (kernel `6.12.105-production+truenas`) under
  HexOS.
- Reach the underlying TrueNAS shell directly, even though HexOS's own
  interface doesn't expose it:
  ```
  https://<nas-ip-or-hostname>/ui/system/shell
  ```

### Diagnosis

- `dmesg` shows this platform's ACPI FADT explicitly declares ASPM
  unsupported:
  ```
  ACPI FADT declares the system doesn't support PCIe ASPM, so disable it
  acpi PNP0A08:00: FADT indicates ASPM is unsupported, using BIOS configuration
  ```
  This locks the OS out of managing ASPM entirely, regardless of kernel
  parameters — confirmed by `cat /sys/module/pcie_aspm/parameters/policy`
  staying on `[default]` even after setting `pcie_aspm.policy=performance`.
- Checked actual per-link state anyway (since the OS lockout doesn't mean
  ASPM is actually *enabled* — it means the OS won't touch whatever the
  BIOS already set):
  ```
  sudo lspci -vv | grep -B4 'LnkCtl.*ASPM'
  ```
  Nearly every link already showed `ASPM Disabled` by BIOS default. Only
  two showed `ASPM L1 Enabled`; identified which devices with:
  ```
  sudo lspci -vv | awk '/^[0-9a-f][0-9a-f]:/{dev=$0} /ASPM L1 Enabled/{print dev}'
  ```
  → both were Intel Alder Lake-P **Thunderbolt 4 Root Ports** (#0 and #2),
  unused and irrelevant to storage. Decided **not** to use `pcie_aspm=force`
  to override the FADT lockout — no actual storage-relevant device was
  affected, and forcing OS ASPM management on hardware whose firmware
  opted out carries real risk of PCIe link instability.

### Kernel boot parameters

Set via `system.advanced.update` (persists across reboots and updates,
unlike hand-editing GRUB):
```
midclt call system.advanced.update '{"kernel_extra_options": "nvme_core.default_ps_max_latency_us=0 pcie_aspm.policy=performance pcie_port_pm=off ahci.mobile_lpm_policy=1"}'
```
then `sudo reboot`.

| Parameter | Purpose |
|---|---|
| `nvme_core.default_ps_max_latency_us=0` | Disables NVMe APST (autonomous power state transitions) |
| `pcie_aspm.policy=performance` | Attempts to force ASPM off in software — supersedes `pcie_aspm=off`, which per current kernel docs only means "leave ASPM untouched," not "disable it" (a common misconception; see [kernel patch](https://lkml.iu.edu/hypermail/linux/kernel/2404.3/06648.html)) |
| `pcie_port_pm=off` | Disables PCIe port runtime power management |
| `ahci.mobile_lpm_policy=1` | Attempted fix for the AHCI "mobile chipset" quirk; did **not** visibly take effect on this controller (still showed `keep_firmware_settings`) — likely this SATA controller's PCI ID isn't in the kernel's mobile-quirk table. Harmless to leave set. |

### SATA link power management — explicit override

The kernel parameter alone left all hosts at `keep_firmware_settings`
(≈ "OS isn't managing this, whatever firmware set stands"). Fixed with a
direct sysfs write, confirmed working, then made persistent:

```
sudo sh -c 'for i in /sys/class/scsi_host/host*/link_power_management_policy; do echo max_performance > "$i"; done'
```

Persisted via TrueNAS Init/Shutdown Script (fires every boot, survives
updates):
```
midclt call initshutdownscript.create '{"type": "COMMAND", "command": "for i in /sys/class/scsi_host/host*/link_power_management_policy; do echo max_performance > $i; done", "when": "POSTINIT", "enabled": true, "comment": "Force SATA link power management to max_performance"}'
```

⚠️ This NAS already has an existing script, `UGREEN LED Controller`
(id 2) — don't remove it when managing scripts via `initshutdownscript.*`.

### Verification (re-run after any reboot)

```
cat /proc/cmdline
cat /sys/module/pcie_aspm/parameters/policy        # [performance] expected, but stays [default] here — see Diagnosis
cat /sys/class/scsi_host/host*/link_power_management_policy   # all → max_performance
sudo dmesg | grep -i aspm
midclt call initshutdownscript.query               # confirm the script is still registered
```

Confirmed working state (as of last check): all 8 SATA hosts →
`max_performance`; kernel cmdline carries all four parameters; script
`id: 3` registered and enabled.

### Storage → Disks (HexOS/TrueNAS GUI) — confirmed ✓

Checked on every disk:

- **HDD Standby:** `Always On`
- **Advanced Power Management:** `Disabled`

No changes needed — all disks were already configured correctly.

- Seagate drives, if any are present, largely ignore standard ATA APM
  commands — the APM setting won't do much for them either way (not a
  concern here since APM is already Disabled on all disks).

### Status: complete

Every item for NAS #2 is now confirmed in place — PCIe/NVMe kernel
parameters set and verified, SATA link power management forced to
`max_performance` and self-reapplying via the persistent Init/Shutdown
Script, and per-disk standby/APM confirmed already correct. Nothing
further outstanding on this unit.
