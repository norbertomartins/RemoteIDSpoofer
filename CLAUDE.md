# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

RIDS is an Arduino sketch for ESP8266/ESP32 (and partial ESP32/RP2040/nRF52 support) that spoofs Remote ID
broadcasts for drones. On boot it serves a small captive-portal-style web UI to configure a GPS origin and
a drone count, then simulates that many "drones" flying random walks around that origin and transmits
ASTM F3411 / ASD-STAN 4709-002 Remote ID packets over Wi-Fi (and optionally BLE) for each of them.
It is educational security-research code (see disclaimer in README.md) — do not add features aimed at
evading detection or making the spoofing harder to identify as fake.

## Build / upload

There is no test suite, linter, or CI in this repo (see the To-Do list in README.md). The only tooling is
`arduino-cli`:

```bash
./build.sh    # arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 RemoteIDSpoofer/RemoteIDSpoofer.ino --clean
./upload.sh   # arduino-cli upload --port /dev/ttyUSB0 --fqbn esp8266:esp8266:nodemcuv2 RemoteIDSpoofer/RemoteIDSpoofer.ino
```

Both scripts hardcode the ESP8266 NodeMCU FQBN and `/dev/ttyUSB0`. To build/upload for a different board
(e.g. ESP32), pass a different `--fqbn` manually rather than editing these scripts unless the user asks —
the target architecture also changes which `id_open_*.cpp` file gets compiled (see below).

The sketch itself lives entirely under `RemoteIDSpoofer/` and is opened as `RemoteIDSpoofer.ino` in the
Arduino IDE; `arduino-cli` compiles the same `.ino` + supporting `.c`/`.cpp`/`.h` files in that directory.

## Architecture

Everything is single-threaded Arduino `setup()`/`loop()` code (`RemoteIDSpoofer.ino`). There is no RTOS
task structure — concurrency is simulated by round-robin updating each spoofed drone once per `loop()`
iteration with a small `delay()` in between.

Flow:
1. **`Frontend`** (`frontend.h`/`frontend.cpp`) brings up a Wi-Fi soft-AP (`ESP_RIDS` / `makkauhijau`) and
   an HTTP server on `192.168.4.1`. It serves a single self-contained HTML page (inlined as a C++ string
   literal in `Frontend::HTML()`) with forms to set GPS coordinates and drone count, and a "Start Spoofing"
   button. Settings persist across power cycles via `EEPROM` (fixed byte offsets `latitude_addr`/
   `longitude_addr`/`num_drones_addr`, with byte 42 used as a "has-been-configured" sentinel). `setup()`
   blocks on `frontend.handleClient()` in a loop until either the user hits Start or a 2-minute idle
   timer (`maxtime`) elapses, at which point it auto-starts spoofing with whatever config is stored.
2. Once spoofing starts, the AP/HTTP server is torn down (`WiFi.softAPdisconnect`, `server.stop()`) and
   `num_spoofers` **`Spoofer`** instances (`spoofer.h`/`spoofer.cpp`) are created — one per simulated
   drone, capped at 16 (`Spoofer spoofers[16]` in the `.ino`; the array is fixed-size because `std::vector`
   wasn't getting linked, per the comment in the `.ino`).
3. Each `Spoofer::update()` (called every `loop()` iteration, rate-limited internally to ~2 Hz per drone)
   does a random-walk physics simulation: random acceleration with a restoring bias back toward the origin
   (`- 0.05 * x`), clamped speed/climb-rate, converts local x/y/z meters into lat/lon/alt deltas via
   `UTM_Utilities::calc_m_per_deg`, then hands the resulting `UTM_data` struct to `squitter.transmit()`.
4. **`ID_OpenDrone`** (`id_open.h`/`id_open.cpp`, vendored from sxjack/uav_electronic_ids) is the Remote ID
   protocol layer. It owns the ASTM/ASD-STAN `ODID_*` structs (defined in `opendroneid.h`/`opendroneid.c`,
   the vendored OpenDroneID reference encoder) and packs `UTM_data`/`UTM_parameters` into Basic ID,
   Location, Self ID, System, and Operator ID messages, then transmits them as Wi-Fi Beacon frames
   (`id_open_beacon.cpp`, using raw 802.11 frame construction in `wifi.c`) and/or BLE advertisements,
   depending on which `id_open_<arch>.cpp` file is compiled in for the target chip:
   - `id_open_esp8266.cpp` — Wi-Fi beacon only (current default target, no BT on ESP8266)
   - `id_open_esp32.cpp` — Wi-Fi beacon and/or BLE (`ID_OD_WIFI_NAN`/`ID_OD_WIFI_BEACON`/`ID_OD_BT` in
     `id_open.h`)
   - `id_open_nrf52.cpp` — BLE only

   The compile-time capability flags (`ID_OD_WIFI_NAN`, `ID_OD_WIFI_BEACON`, `ID_OD_BT`, `USE_BEACON_FUNC`,
   `ID_NATIONAL`/`ID_JAPAN`) at the top of `id_open.h` are selected per-architecture with `#if
   defined(ARDUINO_ARCH_*)` — check this file first when changing target boards or transmission mode.
5. `alt_unix_time.c` provides a `mktime`/`settimeofday` fallback for platforms whose libc lacks it; each
   `Spoofer` fakes a fixed system time on `init()` since there's no RTC/NTP source.

Key structs to know when tracing data flow: `UTM_parameters` (static per-drone identity: operator ID,
region, EU category/class) and `UTM_data` (per-update telemetry: lat/lon, altitude, speed, heading,
satellite count) are defined in `utm.h` and are the interface between the physics simulation (`Spoofer`)
and the protocol encoder (`ID_OpenDrone`).

## Conventions

- This is vendored/adapted third-party protocol code (opendroneid, uav_electronic_ids, esp8266_deauther) —
  match the existing (non-Google, tab-heavy in some files) style local to whichever file you're editing
  rather than imposing a single house style across the whole tree.
- Per-architecture code is split into separate `id_open_<arch>.cpp` files selected by `ARDUINO_ARCH_*`
  preprocessor defines, not runtime branching — new architecture support should follow this same pattern.
- The 16-drone cap and fixed-size `Spoofer` array in `RemoteIDSpoofer.ino` are intentional workarounds
  (see comment), not an oversight.
