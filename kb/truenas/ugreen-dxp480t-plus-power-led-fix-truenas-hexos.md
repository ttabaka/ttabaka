# Fixing the Blinking Power LED on a UGREEN DXP480T Plus Running TrueNAS SCALE / HexOS

## TL;DR

On the UGREEN DXP480T Plus, replacing the stock UGOS firmware with TrueNAS SCALE (or a HexOS install on top of it) leaves the front power LED blinking forever, because nothing is telling the board's embedded LED controller to stop. UGOS normally has a background service for this; TrueNAS doesn't know the controller exists.

The fix is a single `i2cset` command that talks to the controller directly, wrapped in a script that TrueNAS re-runs on every boot via its Init/Shutdown Scripts system. This has been verified working — including surviving a full reboot — on a DXP480T Plus running TrueNAS SCALE with HexOS.

```bash
i2cset -y 0 0x26 0xb1 2 b
```

That's it. One command, solid white LED, confirmed stable for over a minute of observation and across a reboot.

## Applies to

- **Hardware:** UGREEN DXP480T Plus (the all-NVMe, 4-bay model)
- **OS:** TrueNAS SCALE, including systems running the HexOS management layer on top of it
- **Symptom:** Front power button/LED blinks continuously and never settles to solid, even though the system is otherwise healthy

This is *not* the same issue as UGOS's normal "flashing white while shutting down" or "slow orange = device error" behavior — those are documented, expected LED states on the stock firmware. This article is specifically about the LED being stuck blinking under a third-party OS.

## Why this happens

The DXP480T Plus's front LED isn't driven by a simple GPIO pin — it's controlled by a small embedded controller sitting on the motherboard's I2C bus, which UGOS talks to through its own proprietary service. When you flash TrueNAS (or any non-stock OS) onto the box, that service doesn't exist, so the controller is left in whatever default state it powers on in — which, on this hardware, is blinking.

Note: the popular community project [`ugreen_leds_controller`](https://github.com/miskcoo/ugreen_leds_controller) does **not** officially support the DXP480T Plus (it's built for the DXP4800/6800/8800 Plus family, which uses different hardware and a more complex, checksummed multi-byte command protocol). Its CLI tool will likely misdetect or fail to control this board. The DXP480T Plus's controller appears to use a much simpler scheme, closer to a plain register-write interface — see below.

## Step 1 — Load the I2C kernel modules

By default, TrueNAS SCALE doesn't expose `/dev/i2c-*` device nodes. Load the driver that creates them:

```bash
sudo modprobe i2c-dev
```

Then check what buses/adapters exist:

```bash
sudo i2cdetect -l
```

On the DXP480T Plus, you should see an entry like:

```
i2c-0   smbus   SMBus I801 adapter at efa0   SMBus adapter
```

If bus 0 doesn't show up, also load the Intel SMBus driver:

```bash
sudo modprobe i2c-i801
```

## Step 2 — Find the LED controller on the bus

Scan bus 0 for devices:

```bash
sudo i2cdetect -y 0
```

You're looking for a device at address **`0x26`** — that's the LED controller. On the DXP480T Plus it shows up in the `20:` row of the scan output.

## Step 3 — Test the fix manually

With the LED currently blinking, run:

```bash
sudo i2cset -y 0 0x26 0xb1 2 b
```

The power LED should go solid white and stay that way.

### A dead end worth knowing about, in case you're troubleshooting a different unit

An earlier community writeup for this exact model (see Credits below) suggested a second command — writing a "mode" value to register `0x50` — to explicitly select a solid/permanent-on state. On the unit this article was tested against, that register did *not* behave as documented:

| Value written to `0x50` | Result |
|---|---|
| `0` | LED off |
| `1` | Blinking / fast flash |
| `2`, `3`, `4` | Slow pulse / breathing (not solid) |

None of those produced a genuinely steady LED — only the single channel-select command above did, with no follow-up write at all. If you're working on a different DXP480T Plus and the single command above doesn't give you solid on the first try, it's worth testing whether *skipping* any second/mode-setting command produces a better result than trying more values at `0x50`, since on this unit the extra write was actively counterproductive rather than just "not the right value."

## Step 4 — Make it persistent across reboots

The controller resets to blinking on every power cycle, so the fix has to be re-applied at every boot.

### 4a. Write the script to a data pool

TrueNAS SCALE's boot pool is read-only, so the script has to live on a regular data pool, not on the boot environment:

```bash
mkdir -p /mnt/<your-pool>/scripts
cat << 'EOF' | sudo tee /mnt/<your-pool>/scripts/fix-led.sh
#!/bin/bash
modprobe i2c-dev
modprobe i2c-i801
sleep 2
i2cset -y 0 0x26 0xb1 2 b
EOF
sudo chmod +x /mnt/<your-pool>/scripts/fix-led.sh
```

Replace `<your-pool>` with the name of one of your storage pools (e.g. `SSDs`, `tank`, etc.). The `sleep 2` gives the I2C subsystem a moment to be ready before the script tries to write to it.

### 4b. Register it as a Post-Init script

If your system exposes **System Settings → Advanced → Init/Shutdown Scripts** in the web UI, add an entry there:

- **Type:** Script
- **Script:** `/mnt/<your-pool>/scripts/fix-led.sh`
- **When:** Post Init
- **Timeout:** 10
- **Enabled:** ✅

If that page isn't reachable (some HexOS builds don't surface it), register the same thing from the shell using the middleware client:

```bash
sudo midclt call initshutdownscript.create '{
  "type": "SCRIPT",
  "script": "/mnt/<your-pool>/scripts/fix-led.sh",
  "when": "POSTINIT",
  "enabled": true,
  "timeout": 10,
  "comment": "Fix LED blink"
}'
```

Verify it registered correctly:

```bash
midclt call initshutdownscript.query
```

You should see your entry listed with `"enabled": true`.

## Step 5 — Confirm it survives a real reboot

Manual testing only proves the commands work — the actual fix isn't confirmed until it survives a cold boot on its own:

```bash
sudo reboot
```

Wait a couple of minutes for the system to fully come up, then check the front panel. Watch it for at least 30-60 seconds — a slow breathing pulse can look solid at a glance, and only sustained observation rules it out. It should be genuinely steady white with no manual intervention required.

## Known tradeoffs

- This locks the LED to solid white at all times, including during shutdown/reboot — you lose the visual "it's safe to unplug now" cue that UGOS normally gives you during power-off. If you want that back, you can register a second script under a `SHUTDOWN` Init/Shutdown Script entry that writes to the LED's red channel before the system halts — not covered here, but the same `i2cset` approach applies; expect to re-derive the correct value(s) through the same trial-and-observe process described in Step 3.
- The exact register behavior here was reverse-engineered against one specific unit's firmware through trial and error. If your board behaves differently, use the same step-through method to find what works, and consider documenting what you find for others (e.g. as a comment on the upstream project's compatibility issue).

## Credits / further reading

- [`miskcoo/ugreen_leds_controller`](https://github.com/miskcoo/ugreen_leds_controller) — the community project this approach is adjacent to (does not officially support the DXP480T Plus; uses a different, checksummed command protocol for the models it does support)
- [DXP480T Plus compatibility discussion, `ugreen_leds_controller` issue tracker](https://github.com/miskcoo/ugreen_leds_controller/issues)
- [UGREEN DXP480T Plus blinking LED thread (v64.tech forum)](https://v64.tech/t/ugreen-dxp-480t-plus-blinkende-led/4293) — original community writeup that identified I2C address `0x26` on this model (using Proxmox rather than TrueNAS); its specific register-`0x50` mode values did not reproduce cleanly on the unit this article documents
