# Installation

**Read [DISCLAIMER.md](DISCLAIMER.md) first.** You are responsible for what you do to your own printer.

> **Skill level:** this guide assumes intermediate/advanced comfort with editing config files, either over SSH or through a web file browser. If terms like "SSH", "config file", or "backup a file before editing it" are unfamiliar, ask someone experienced to do this with/for you rather than improvising — see the disclaimer.

## 0. Enable root / remote access on the printer

By default the printer does not expose SSH or web-based config access — you have to turn it on yourself, once, from the touchscreen:

1. On the printer's touchscreen, go to **Settings → Root account information**.
2. Read the disclaimer shown. You'll need to **wait ~30 seconds** before the confirmation checkbox becomes available.
3. Check **"I have read and understood the risks of root login"** and press **OK**.
4. The screen will now show your **root password** (auto-generated, unique per printer) — write it down.
5. Find your printer's **IP address** under **Settings → Network** (shown under your network name/SSID).

Username is always `root`. You now have everything needed for both options below: `ssh root@<ip>` or the web UI.

Keep this password somewhere safe and don't expose your printer's management ports to the internet — this access level is meant for your local network only.

## 1. Before anything: back up

Every patch in this repo touches one of these files:

```
/mnt/UDISK/printer_data/config/printer.cfg
/mnt/UDISK/printer_data/config/gcode_macro.cfg
/mnt/UDISK/printer_data/config/sensorless.cfg
```

For each file you're about to touch, make a timestamped backup **on the printer itself** before editing, e.g.:

```sh
cp printer.cfg printer.cfg.bak_$(date +%Y%m%d_%H%M%S)_before-tweaks
```

Keep these. If anything looks wrong after a change, restoring is just copying the backup back over the live file and restarting Klipper.

## 2. Get to the config files: pick SSH or the web UI

This repo is **IP-agnostic** — it does not assume or publish any specific IP address; use the one you found in step 0.

### Option A — SSH (recommended if you're comfortable with a terminal)

```sh
ssh root@<your-printer-ip>
```

Enter the root password from step 0 when prompted (it won't show as you type — that's normal).

> Note: this printer doesn't ship `sshpass` or an SFTP server, so plain `scp` will fail. To copy files, use `cat`:
> ```sh
> ssh root@<ip> "cat /path/to/remote/file" > local_file      # download
> ssh root@<ip> "cat > /path/to/remote/file" < local_file    # upload
> ```

### Option B — Web file manager (no terminal needed)

If you'd rather not use SSH, the printer runs a Fluidd/Mainsail-style web UI on **port 4408**. Open in your browser:

```
http://<your-printer-ip>:4408/#/configure
```

This is the same "Configuration" file browser Klipper web UIs normally expose — you'll see `printer.cfg`, `gcode_macro.cfg`, `sensorless.cfg`, etc. listed there. You can open, edit, and save each file directly in the browser, including downloading a copy first (that's your backup — see step 1) before making changes. It uses the same root credentials from step 0 if it prompts for login.

Either option edits the exact same files — use whichever you're more comfortable with. The rest of this guide describes actions in terms of the files themselves, not the specific tool.

## 3. Confirm the printer is idle before editing

Never edit config or restart Klipper mid-print. Check first:

```sh
curl -s http://<your-printer-ip>:7125/printer/info
```

Look for `"state": "ready"`. (Or just check the touchscreen isn't showing an active print.)

## 4. Apply the patches you want

Each file in `patches/` shows the exact section to find and what to replace it with. Open the relevant config file (via SSH with `vi`/`cat`, or via the web file manager from step 2), locate the "before" block, and replace it with the "after" block exactly as shown. Apply them **one at a time**, testing a real print between each, especially the first time.

- `01-fix-double-z-homing.patch` → edits `sensorless.cfg`
- `02-faster-nozzle-clean.patch` → edits `printer.cfg`
- `03-no-preheat-wait-before-homing.patch` → edits `gcode_macro.cfg`
- `04-optional-bed-mesh-density.patch` → edits `printer.cfg` (optional, your call)

## 5. Restart Klipper — then power-cycle the printer

Restart Klipper first (via SSH, or the restart button in the web UI):

```sh
ssh root@<your-printer-ip> "/etc/init.d/klipper restart"
```

Wait ~10-15 seconds, then check:

```sh
curl -s http://<your-printer-ip>:7125/printer/info
```

You want `"state": "ready"`. If you see an error state, check `/mnt/UDISK/printer_data/logs/klippy.log` (or the Console/Log tab in the web UI) for the parse error — it will point at the exact line. Restore your backup if needed.

**Once Klipper comes back clean, do a full power cycle of the printer too** (turn it off and back on from the physical switch, not just a Klipper restart). This isn't strictly required by Klipper itself, but it's cheap insurance against any of the printer's other services (touchscreen `display-server`, camera, box/AMS controller) holding onto stale state from before the config change — worth the extra minute before you trust these changes on a real print.

## 6. Test with a real, low-stakes print

Ideally a **cold start** (printer at room temperature) for patch 3, since its effect is invisible if the bed/nozzle are already hot. Watch it home, clean, and level — confirm nothing looks wrong before trusting it on something you care about.

## Replicating across multiple printers of the same model

If you own more than one of this printer model, these config *values* (homing speeds, nozzle-clean parameters, macro logic) are safe to copy between units. **Never copy**:

- Input shaping values (`shaper_type_x/y`, `shaper_freq_x/y`) — mechanical resonance is per-unit.
- Saved bed mesh points — recalibrate on each printer.
- `cut_pos_x` (filament cutter position) and any auto-generated hardware address tables — these are per-unit hardware calibration.

## Firmware updates

A future Creality OTA update may overwrite these config files. If your printer suddenly reverts to double-homing, slow nozzle cleaning, or the pre-homing wait comes back, just re-apply the patches — check this repo for an updated version matching your new firmware, or open an issue if something no longer applies cleanly.
