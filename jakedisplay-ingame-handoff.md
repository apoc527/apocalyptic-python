# Handoff: JakeDisplay as an in-game table display

Paste this into a **local** Claude Code session on the Mac that the display is plugged into. The best place to open it is the GurpsBattleManager checkout.

## Goal
Use the JakeDisplay (Freenove ESP32-S3, 4.0" 480x320 touchscreen) as a GURPS Banestorm table display, driven by Python over its WiFi API. Use the touchscreen as a "next turn" button.

## Why this has to run locally
An earlier cloud session couldn't do the work:
- The display is on the home LAN. Cloud sessions can't reach it.
- The firmware and the `.claude/skills/` (`/flash-jakedisplay`, `/add-screen`, `/add-audio`) aren't on GitHub. They exist only on this Mac.
- Flashing needs USB (`/dev/cu.usbmodem21301`).

## First steps (before writing code)
1. **Find the firmware folder** on this Mac (look for `platformio.ini` with `env: app`). Note its path. You'll need it if the display needs a new screen type.
2. **Look at how GurpsBattleManager stores state.** Find out what language it's in and where initiative order, the current turn, HP/FP and the in-game date live. Can Python read that state (JSON file, SQLite, importable module)?
   - If yes, build the display as a thin adapter inside this repo.
   - If the state lives only in memory or a UI, build a small standalone project instead and don't rebuild the battle manager's state.
3. **Smoke-test the device:** `curl http://<device-ip>/api/status`, then send one `POST /api/update` and one `POST /api/stats`, and check what each looks like on the screen.
4. **Measure touch latency:** poll `GET /api/touch` in a loop and time how long a tap takes to register. Expect a few hundred ms. Decide whether that's acceptable for the table.

## Candidate features (rough priority)
1. **Combat round / initiative tracker:** shows whose turn it is, with HP/FP. A tap advances the turn. Uses the battle manager's data if it's reusable.
2. **Banestorm clock and weather:** in-game date, watch or time of day, and weather. A tap advances a watch. Weather could come from the existing `banestorm-travel-generator` logic.
3. **Doom clock / faction-pressure ticker:** changes color and plays an audio sting (e.g. `wargames`) when a threshold is crossed.

## Known constraints and open questions
- The API only renders text: title, body, an RGB color, and stats label/value pairs. No pixels, images or animation. Check whether `/api/stats` can lay out an initiative list well enough. If not, add a screen in the firmware with `/add-screen`.
- How many label/value pairs does `/api/stats` support? Look at the firmware source.
- The `/api/touch` `ts` field is probably for spotting new taps. Check whether it's milliseconds since boot, and how to debounce double taps.
- Audio is limited to four built-in clips. New stings need `/add-audio` and a reflash.
- A laptop has to keep the Python script running during play.

## Hardware / API reference (from the original brief)
- Board: Freenove FNK0104S, ESP32-S3 dual-core 240 MHz, 16 MB flash, 8 MB PSRAM
- Display: 4.0" IPS 480x320 ST7796 SPI
- Touch: FT6336U, I2C 0x38 (SDA=GPIO16, SCL=GPIO15)
- Audio: ES8311 codec, I2C 0x18. I2S MCLK=GPIO4, BCLK=GPIO5, WS=GPIO7, DOUT=GPIO8. AMP_EN=GPIO1 (active LOW)
- **Wiring rule:** `Wire.begin(16, 15)` MUST come before `lgfx->init()`. Otherwise LovyanGFX reinitializes I2C and breaks the bus.
- Flash: `.venv/bin/pio run -e app -t upload --upload-port /dev/cu.usbmodem21301`

```
POST /api/update  {"title":"…","body":"…","color":[r,g,b]}
POST /api/stats   {"title":"…","body":"…","label1":"…","val1":"…",...}
GET  /api/status  → {"ip":"…","uptime_s":…,"wifi":"SSID"}
POST /api/reset   -- factory restore (reruns WarGames boot)
POST /api/demo    -- non-destructive overlay of boot sequence
POST /api/audio   {"clip":"wargames"|"chess"|"theme"|"sneakers"}
GET  /api/touch   → {"x":…,"y":…,"ts":…}
GET  /api/docs    -- API docs
```

## Opening prompt for the local session
> Read `jakedisplay-ingame-handoff.md`. Do the "First steps" first and report back on (1) where the firmware lives, (2) whether GurpsBattleManager's combat state is reusable from Python, and (3) the smoke test and touch-latency results. Then recommend which candidate feature to build first. Don't write the display code until we've agreed on that.
