# Installation

**Read [DISCLAIMER.md](DISCLAIMER.md) first.** You are responsible for what you do to your own printer.

## 0. Before anything: back up

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

## 1. Get SSH access to your printer

This repo is **IP-agnostic** — it does not assume or publish any specific IP address. Find your printer's IP on your own LAN (check your router's DHCP client list, or the printer's network settings screen), then:

```sh
ssh root@<your-printer-ip>
```

Default credentials for this model are documented by Creality for the Creality Hi line — check your printer's manual or support page if you don't already have them. Treat these credentials as sensitive; this printer should only ever be reachable on your local network, not exposed to the internet.

> Note: this printer doesn't ship `sshpass` or an SFTP server, so plain `scp` will fail. To copy files, use `cat`:
> ```sh
> ssh root@<ip> "cat /path/to/remote/file" > local_file      # download
> ssh root@<ip> "cat > /path/to/remote/file" < local_file    # upload
> ```

## 2. Confirm the printer is idle before editing

Never edit config or restart Klipper mid-print. Check first:

```sh
curl -s http://<your-printer-ip>:7125/printer/info
```

Look for `"state": "ready"`.

## 3. Apply the patches you want

Each file in `patches/` shows the exact section to find and what to replace it with. Open the relevant config file (`vi` is available on the printer, or edit locally and re-upload with the `cat` trick above), locate the "before" block, and replace it with the "after" block exactly as shown. Apply them **one at a time**, testing a real print between each, especially the first time.

- `01-fix-double-z-homing.patch` → edits `sensorless.cfg`
- `02-faster-nozzle-clean.patch` → edits `printer.cfg`
- `03-no-preheat-wait-before-homing.patch` → edits `gcode_macro.cfg`
- `04-optional-bed-mesh-density.patch` → edits `printer.cfg` (optional, your call)

## 4. Restart Klipper and check for errors

```sh
ssh root@<your-printer-ip> "/etc/init.d/klipper restart"
```

Wait ~10-15 seconds, then check:

```sh
curl -s http://<your-printer-ip>:7125/printer/info
```

You want `"state": "ready"`. If you see an error state, check `/mnt/UDISK/printer_data/logs/klippy.log` for the parse error — it will point at the exact line. Restore your backup if needed.

## 5. Test with a real, low-stakes print

Ideally a **cold start** (printer at room temperature) for patch 3, since its effect is invisible if the bed/nozzle are already hot. Watch it home, clean, and level — confirm nothing looks wrong before trusting it on something you care about.

## Replicating across multiple printers of the same model

If you own more than one of this printer model, these config *values* (homing speeds, nozzle-clean parameters, macro logic) are safe to copy between units. **Never copy**:

- Input shaping values (`shaper_type_x/y`, `shaper_freq_x/y`) — mechanical resonance is per-unit.
- Saved bed mesh points — recalibrate on each printer.
- `cut_pos_x` (filament cutter position) and any auto-generated hardware address tables — these are per-unit hardware calibration.

## Firmware updates

A future Creality OTA update may overwrite these config files. If your printer suddenly reverts to double-homing, slow nozzle cleaning, or the pre-homing wait comes back, just re-apply the patches — check this repo for an updated version matching your new firmware, or open an issue if something no longer applies cleanly.
