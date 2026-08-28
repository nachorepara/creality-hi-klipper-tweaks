# ⚡ Creality Hi / F018 — Klipper Startup Speed-Up

**Stop waiting. Start printing.**

A small set of surgical Klipper config tweaks for the **Creality Hi** (internal model **F018**, config profile `F018_CR4NU200360C20`) that cut a huge chunk of dead time out of the pre-print routine — without touching print quality, bed leveling accuracy, or reliability.

If you own this printer and have ever stared at the screen while it homes Z twice, cleans the nozzle forever, and waits to hit temperatures it doesn't even need yet — this is for you.

> Tested on firmware **OTA 1.1.0.74** (OpenWrt 21.02-SNAPSHOT base, Klipper `09faed31-dirty`). Should apply to any Creality Hi / F018 running a similar firmware line. Works over your **local network only** — this repo is IP-agnostic, you just need SSH access to *your own* printer (see [Requirements](#requirements)).

---

## ⏱️ Results

Measured on a real cold-start print (bed/nozzle at room temp), same `.gcode` file, before and after:

| Stage | Stock | Tweaked | Saved |
|---|---:|---:|---:|
| Wait for nozzle temp *before* the first homing move | ~35 s (blocking) | ~0 s (heats in background) | **~35 s** |
| First full Z homing (mechanical dual-motor sync) | ~52 s | ~52 s | *(unchanged — needs to happen once)* |
| Second Z homing (before bed mesh) | ~57–61 s (full re-sync) | ~17–23 s (skipped, already synced) | **~35–40 s** |
| Nozzle cleaning routine | ~168 s | ~102 s | **~66 s** |
| **Total time from "Print" to first layer** | **~7–8 min** | **~5–6 min** | **~90–140 s (≈20–25%)** |

No change to bed mesh accuracy, Z offset, or print quality was observed across multiple test prints.

---

## 🔧 What's actually fixed

### 1. Double full Z-homing → single homing
The printer's Z-align routine (`ZDOWN`) does a full mechanical dual-motor optical-sensor sync — slow, and normally triggered by **every** `G28 Z`. Stock behavior runs it **twice** per print start (once for XY clearance homing, once again right before bed mesh calibration), even though nothing moved the axis out of sync in between.

**Fix:** a session flag (`xyz_ready.z_ready`) set after the first successful sync. Any subsequent `G28 Z` in the *same print-prep sequence* skips the mechanical re-sync and does a fast probe-only home instead. Patched at the real common choke point (`_HOME_Z` in `sensorless.cfg`), so it works regardless of whether the touchscreen, an app, or the sliced `.gcode` file is driving the print.

> Note: the flag resets after every print (Klipper un-homes everything on `M84`), so this saves time *per print start*, not cumulatively across prints — that's expected and correct.

📄 [`patches/01-fix-double-z-homing.patch`](patches/01-fix-double-z-homing.patch)

### 2. Faster, still-effective nozzle cleaning
Stock `[nozzle_clear]` does ~9–11 probe touches against the rear metal plate and waits for the nozzle to passively cool all the way down to 120 °C before finishing ("closure" step, meant to stop oozing).

**Fix:**
- `touch_cnt: 3 → 1` (cuts touches from ~9 to ~3 — same clean result, much less mechanical back-and-forth)
- `closure_temp: 160` (previously implicit 120) — still fully seals the tip, no oozing observed, just doesn't wait to cool as far

📄 [`patches/02-faster-nozzle-clean.patch`](patches/02-faster-nozzle-clean.patch)

### 3. No more blocking heat-waits before homing
Before the first `G28`, the printer issues a blocking `M109` (wait for **exact** nozzle temp) and would do the same for `M190` (bed) if triggered — despite the fact that **homing needs zero heat**. That wait was pure dead time serialized in front of a move that doesn't care about temperature at all.

**Fix:** global `M109`/`M190` overrides (the same `rename_existing` pattern Creality already uses for `M204`/`PAUSE`/`CANCEL_PRINT`) that check `printer.toolhead.homed_axes`. If **nothing is homed yet**, `M109`/`M190` fall through to non-blocking `M104`/`M140` — target is set, heating happens *in parallel* with homing instead of before it. Once anything is homed, every later temperature wait (nozzle cleaning, bed mesh, actual print temp) behaves exactly as stock — this only touches the one useless pre-homing wait.

📄 [`patches/03-no-preheat-wait-before-homing.patch`](patches/03-no-preheat-wait-before-homing.patch)

### 4. Fewer bed mesh points (optional)
Stock ships with a fairly dense probe grid. Dropping `probe_count` (e.g. `9,9 → 5,5`) cuts calibration time noticeably. This is a accuracy/speed trade-off specific to your bed's flatness — tune it yourself, it's not a "fix" so much as a dial. Included here as a documented option, not a forced change.

---

## 🚫 What we deliberately did *not* touch

- **The ~60 s `M109 S120` wait before bed mesh calibration** (`PRINT_TEMP_SET`). The pressure-based probe (`prtouch_v3`) has a temperature-compensation curve (`prth_tmp_comp`), so changing this wait risks skewing your Z-offset/mesh accuracy. Not worth the ~1 minute saved.
- **The camera/bed LED dimming after startup.** Confirmed via firmware inspection and [Creality's own community forum](https://forum.creality.com) that this is a known, unresolved closed-firmware limitation (driven by the proprietary `display-server` binary). No config lever exists for it — don't waste time looking.
- **Input shaping, per-axis mechanical calibration (`cut_pos_x`), and any auto-generated hardware address tables.** These are specific to *your individual printer's* mechanics — never copy these values between two printers, even the same model.

---

## 📦 What's in this repo

```
patches/
  01-fix-double-z-homing.patch
  02-faster-nozzle-clean.patch
  03-no-preheat-wait-before-homing.patch
  04-optional-bed-mesh-density.patch
INSTALL.md
DISCLAIMER.md
```

Each patch file shows the exact **before → after** for the relevant section, with the file it belongs to and why. They're written as human-readable "find this block, replace with this block" diffs — apply them by hand over SSH (see [INSTALL.md](INSTALL.md)), since exact line numbers can drift between firmware versions.

---

## Requirements

- A Creality Hi / F018 printer running Klipper (check via the printer's OTA version — `1.1.0.74` is what this was built and tested against).
- SSH access to **your own printer** on **your own local network**. Default factory credentials are typically `root` / a printer-specific password shown in Creality's docs for this model — this repo does not publish any IP address or password, because none of that is fixed: your printer's LAN IP depends on your own router/DHCP, and you should treat SSH credentials as sensitive.
- Basic comfort with SSH and editing text config files. If you're not comfortable doing this yourself, ask someone who is — see the disclaimer below.

See [INSTALL.md](INSTALL.md) for step-by-step instructions, including **how to back up your config before touching anything** (do this — every patch here assumes you did).

---

## ⚠️ Disclaimer

**Use at your own risk.** These are unofficial, community-made modifications to your 3D printer's firmware configuration, based on reverse-engineering the stock Klipper setup on one specific printer model and firmware version. They are **not affiliated with, endorsed by, or supported by Creality**.

Editing 3D printer firmware can affect print quality, mechanical calibration, or in rare misconfigured cases, safety-relevant heater/homing behavior. **Always back up your config files before making any change**, and test with a non-critical print first.

**The author(s) of this repository accept no responsibility or liability for any damage, malfunction, data loss, failed prints, or hardware damage resulting from the use of the information or files in this repository.** You are solely responsible for any change you make to your own printer.

Full text in [DISCLAIMER.md](DISCLAIMER.md).

---

## Contributing

Found this useful on a different firmware version, or found a config path that changed? PRs and issues welcome — especially reports of what firmware/OTA version you tested against, since Creality can (and does) change these internals between updates.

## License

MIT — see [LICENSE](LICENSE). The code is free to use and adapt; the disclaimer above still applies regardless of license.
