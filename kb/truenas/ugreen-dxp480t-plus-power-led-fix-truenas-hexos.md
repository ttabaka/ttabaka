# Fixing the Blinking Power LED on a UGREEN DXP480T Plus Running TrueNAS SCALE / HexOS

## TL;DR

On the UGREEN DXP480T Plus, replacing the stock UGOS firmware with TrueNAS SCALE (or a HexOS install on top of it) leaves the front power LED blinking forever, because nothing is telling the board's embedded LED controller to stop. UGOS normally has a background service for this; TrueNAS doesn't know the controller exists.

**The single `i2cset` command that circulates for this exact chip address does not fix it.** It was tested repeatedly on this unit — with `sudo`, with `-f` force, with a 15-second read-back loop confirming the write genuinely holds — and the LED kept blinking regardless. The controller needs **five writes in a specific order**: reset both color channels off, disable slow-flash, turn the white channel on, then explicitly force the mode register to solid.

```bash
sudo i2cset -y 0 0x26 0xa0 1 b   # red off
sudo i2cset -y 0 0x26 0xa0 2 b   # white off (reset state)
sudo i2cset -y 0 0x26 0x51 0 b   # slow flash off
sudo i2cset -y 0 0x26 0xb1 2 b   # white on
sudo i2cset -y 0 0x26 0x50 0 b   # mode: solid (1 = fast flash)
```

Run all five, in order, and the LED goes solid white immediately. This is wrapped in a script that TrueNAS re-runs on every boot (and again at shutdown) via its Init/Shutdown Scripts system — see Step 4. **Status: confirmed working manually and registered for boot/shutdown; a full cold-reboot verification is still pending** (see Step 5) — don't take that claim further than that until it's actually been tested.

## Applies to

- **Hardware:** UGREEN DXP480T Plus (the all-NVMe, 4-bay model)
- **OS:** TrueNAS SCALE, including systems running the HexOS management layer on top of it
- **Symptom:** Front power button/LED blinks continuously and never settles to solid, even though the system is otherwise healthy — confirmed via `zpool status` (clean, 0 errors on every pool) and `sensors` (all temperatures well under threshold). It's also not a background process fighting a manual fix: no service, cron job, or Docker container on the box touches i2c or LED state.

This is *not* the same issue as UGOS's normal "flashing white while shutting down" or "slow orange = device error" behavior — those are documented, expected LED states on the stock firmware. This article is specifically about the LED being stuck blinking under a third-party OS.

## Why this happens

The DXP480T Plus's front LED isn't driven by a simple GPIO pin — it's controlled by a small embedded controller sitting on the motherboard's I2C bus, which UGOS talks to through its own proprietary service. When you flash TrueNAS (or any non-stock OS) onto the box, that service doesn't exist, so the controller is left in whatever default state it powers on in — which, on this hardware, is blinking.

The single-line command that circulates for this address is presumably an excerpt of the five-write sequence below with the four supporting writes dropped — a genuine, valid write to a genuine "on" register, just not enough on its own to force the mode out of whatever flash state the chip powered on in.

Note: the popular community project [`ugreen_leds_controller`](https://github.com/miskcoo/ugreen_leds_controller) does **not** officially support the DXP480T Plus (it's built for the DXP4800/6800/8800 Plus family, which uses different hardware and a more complex, checksummed multi-byte command protocol). Its CLI tool will likely misdetect or fail to control this board. The DXP480T Plus's controller uses a much simpler scheme — a plain register-write interface, documented below.

## Step 1 — Load the I2C kernel modules

By default, TrueNAS SCALE doesn't expose `/dev/i2c-*` device nodes. Load the driver that creates them:

```bash
sudo modprobe i2c-dev
sudo modprobe i2c-i801
```

Then check what buses/adapters exist:

```bash
sudo i2cdetect -l
```

You'll see well over a dozen adapters (the CPU's GPU/DisplayPort gmbus buses show up too) — only one matters. Look for:

```
i2c-0   smbus   SMBus I801 adapter at efa0   SMBus adapter
```

## Step 2 — Find the LED controller on the bus

Scan bus 0 for devices:

```bash
sudo i2cdetect -y 0
```

You're looking for a device at address **`0x26`** — that's the LED controller. On the DXP480T Plus it shows up in the `20:` row of the scan output:

```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
20: -- -- -- -- -- -- 26 -- -- -- -- -- -- -- -- --
```

## Step 3 — What doesn't work (save yourself the time)

Before the real fix, here's everything that was ruled out on this unit — worth knowing if you're troubleshooting a different one and land on the same dead ends:

- **The single-command "fix"** — `sudo i2cset -y 0 0x26 0xb1 2 b` — does not stop the blink by itself, run any number of times.
- **Not a silently-reverting register.** Wrote `0x01` to `0xb1`, then polled it back with `i2cget` once a second for 15 seconds — it held `0x01` the entire time. So the chip isn't quietly resetting the byte on a timer; the single write is simply the wrong shape of fix, not a race condition.
- **Also tried, zero effect:** values `0` and `1` at `0xb1`; values `0x04` and `0x08` individually at both `0xb0` and `0xb1` (byte-flag values documented for a related chip on a different UGREEN model — didn't transfer to this one).

## Step 4 — Apply the real fix

The controller needs five writes, in order:

| Register | Value | Effect (best-effort label — see caveat below) |
|---|---|---|
| `0xa0` | `1` | red — off |
| `0xa0` | `2` | white — off (reset before setting) |
| `0x51` | `0` | slow-flash — off |
| `0xb1` | `2` | white — on |
| `0x50` | `0` | mode — solid (`1` = fast flash) |

```bash
sudo i2cset -y 0 0x26 0xa0 1 b
sudo i2cset -y 0 0x26 0xa0 2 b
sudo i2cset -y 0 0x26 0x51 0 b
sudo i2cset -y 0 0x26 0xb1 2 b
sudo i2cset -y 0 0x26 0x50 0 b
```

Run all five, in this order. The LED goes solid white immediately — confirmed on this unit.

> **Caveat:** which of these five lines is actually load-bearing hasn't been isolated — the full sequence is what's proven, not each line individually. The "effect" labels above come from a third-party reverse-engineering writeup, not a datasheet, so treat them as best-effort, not verified truth. Anecdotally, the `0x50 → 0` write looked like the one doing the most visible work; if you're troubleshooting a different unit and want to shortcut this, that's the first line worth isolating — not the last.

## Step 5 — Make it persistent across reboots

The controller needs re-initializing at every boot, so the fix has to run automatically each time, not just once from a shell.

### 5a. Write the scripts to a data pool

TrueNAS SCALE's boot pool is read-only, so the scripts have to live on a regular data pool. Same five writes in both — the boot-time version also loads the kernel modules first:

```bash
mkdir -p /mnt/<your-pool>/scripts

cat << 'EOF' | sudo tee /mnt/<your-pool>/scripts/fix-led.sh
#!/bin/bash
modprobe i2c-dev
modprobe i2c-i801
sleep 2
i2cset -y 0 0x26 0xa0 1 b   # red off
i2cset -y 0 0x26 0xa0 2 b   # white off (reset state)
i2cset -y 0 0x26 0x51 0 b   # slow flash off
i2cset -y 0 0x26 0xb1 2 b   # white on
i2cset -y 0 0x26 0x50 0 b   # mode: solid
EOF
sudo chmod +x /mnt/<your-pool>/scripts/fix-led.sh

cat << 'EOF' | sudo tee /mnt/<your-pool>/scripts/shutdown-led.sh
#!/bin/bash
i2cset -y 0 0x26 0xa0 1 b   # red off
i2cset -y 0 0x26 0xa0 2 b   # white off (reset state)
i2cset -y 0 0x26 0x51 0 b   # slow flash off
i2cset -y 0 0x26 0xb1 2 b   # white on
i2cset -y 0 0x26 0x50 0 b   # mode: solid
EOF
sudo chmod +x /mnt/<your-pool>/scripts/shutdown-led.sh
```

Replace `<your-pool>` with the name of one of your storage pools (e.g. `SSDs`, `tank`, etc.). The `sleep 2` in the boot script gives the I2C subsystem a moment to be ready before it writes; the shutdown script skips that since the modules are already loaded by then.

### 5b. Register both as Init/Shutdown Scripts

If your system exposes **System Settings → Advanced → Init/Shutdown Scripts** in the web UI, add two entries there — one `Type: Script`, `Script: /mnt/<your-pool>/scripts/fix-led.sh`, `When: Post Init`, `Enabled: ✅`, and a second with `shutdown-led.sh` and `When: Shutdown`.

If that page isn't reachable (some HexOS builds don't surface it), register both the same way from the shell using the middleware client:

```bash
sudo midclt call initshutdownscript.create '{
  "type": "SCRIPT",
  "script": "/mnt/<your-pool>/scripts/fix-led.sh",
  "when": "POSTINIT",
  "enabled": true,
  "timeout": 10,
  "comment": "Fix power LED (boot)"
}'

sudo midclt call initshutdownscript.create '{
  "type": "SCRIPT",
  "script": "/mnt/<your-pool>/scripts/shutdown-led.sh",
  "when": "SHUTDOWN",
  "enabled": true,
  "timeout": 10,
  "comment": "Fix power LED (shutdown)"
}'
```

Verify both registered correctly:

```bash
sudo midclt call initshutdownscript.query
```

Confirmed output on this unit:

```json
[{"type": "SCRIPT", "script": ".../fix-led.sh",      "when": "POSTINIT", "enabled": true, "id": 1},
 {"type": "SCRIPT", "script": ".../shutdown-led.sh", "when": "SHUTDOWN",  "enabled": true, "id": 2}]
```

## Step 6 — Confirm it survives a real reboot

**Not yet done on this unit — treat this step as open, not confirmed.** Manual testing and registering the scripts only proves the pieces are in place; the fix isn't actually confirmed until it survives a cold boot with zero manual intervention:

```bash
sudo reboot
```

Wait a couple of minutes for the system to fully come up, then check the front panel. Watch it for at least 30–60 seconds — a slow breathing pulse can look solid at a glance, and only sustained observation rules it out. It should be genuinely steady white with no manual intervention required. Update this document once that's actually been observed.

## Known tradeoffs

- The shutdown script forces the same solid-white state right up to power-off, so the light no longer changes between "running" and "shutting down" — you lose whatever visual "safe to unplug now" cue UGOS originally gave you. If that distinction matters, the shutdown script could instead target a different value (e.g. the red channel, or off) — not attempted here, and would need its own trial-and-observe pass same as Step 3/4.
- The exact register behavior here was reverse-engineered against one specific unit's firmware through trial and error, cross-checked against a community writeup for the same model. If your board behaves differently, use the same step-through method (Step 3's approach) to find what works, and consider documenting what you find for others (e.g. as a comment on the upstream project's compatibility issue).

## Credits / further reading

- [Ugreen DXP 480T Plus — blinkende LED (v64.tech forum)](https://v64.tech/t/ugreen-dxp-480t-plus-blinkende-led/4293) — source of the working five-write sequence (originally documented against Proxmox, not TrueNAS). Applied as a complete sequence, it reproduced cleanly on this unit — an earlier note suggesting the register `0x50` values didn't reproduce was based on testing that register in isolation, not the full documented sequence.
- [`miskcoo/ugreen_leds_controller`](https://github.com/miskcoo/ugreen_leds_controller) — the community project this approach is adjacent to (does not officially support the DXP480T Plus; uses a different, checksummed command protocol for the models it does support)
- [DXP480T Plus compatibility discussion, `ugreen_leds_controller` issue tracker](https://github.com/miskcoo/ugreen_leds_controller/issues)
