# giga-r1-j1939-simulator
Arduino GIGA R1 based vehicle simulator with J1939 CAN, Web UI controls,

# Vehicle Simulator (Arduino GIGA)

Stable vehicle simulator for **Arduino GIGA R1** with **J1939 CAN**, **Web UI**, and **persistent odometer/engine hours**.

## Highlights
- J1939 broadcasts (VIN BAM, EEC1, CCVS, Engine Hours, Distance Legacy, Fuel, Temps, Voltage)
- Web UI (STA + AP fallback; typical AP IP `192.168.3.1`)
- Persistence (KVStore): saves hourly (Δ ≥ 1 h or 10 km, 10-min guard) and at end-of-drive (IGN OFF after ≥2 min or ≥1 km)
- Simple, modifiable simulation loop

## Hardware
- Arduino **GIGA R1**
- Seeed **CAN-BUS Shield v2 (MCP2515, 16 MHz)**
- 12 V bench supply (OBD-II pin 16 for brief tests)
- usb cable and power supply
- db 9 to output of you choosing j1939 or obd2

## Software
- Arduino IDE (latest GIGA Mbed core)
- Libraries: **ArduinoJson**, **WiFi** (GIGA), **mcp2515**

## Quick start
1. Put your sketch in `src/VehicleSimulator_GIGA.ino` (or rename your `.cpp` to `.ino` and place it there).
2. Open in Arduino IDE → Board: **Arduino GIGA R1** → Upload.
3. Serial @115200: expect `"[KV] Loaded persistent values."`
4. Connect to the device (STA or AP) → open the **Web UI** → simulate a drive.

More details: see `docs/OVERVIEW.md` and `docs/SETUP.md`.

# 

# Changelog


---

## v1.20 — Auto Mode & Large-Number Counters Fix


### Fixed
- **Counters freezing at large values in Auto mode.** Web UI was repeatedly POSTing displayed (large) values back to the device, overwriting live increments.  
  - Web POST now **does not overwrite** `odometer` or `engineHours` unless explicitly allowed (guard pattern).  
  - Optional rule supported: accept edits only when **ignition is OFF**.

### Added
- **Auto-drive helper (optional):** Auto mode can automatically turn **Ignition On** and ramp speed toward a target (e.g., 45 km/h), ensuring both engine hours and odometer advance hands-free.

### Improvements
- Guidance to keep **wide internal accumulators** (`uint64_t` tenths for distance, `double` for hours) and convert only for UI, to remain robust at very large values.

### Notes
- Compatible with v5_20 persistence thresholds and end-of-drive save logic.

---

## v1.10 — Persistence (KVStore) + End-of-Drive Save


### Added
- **Persistent state** via mbed **KVStore**:
  - Saves **Odometer** (tenths of km) and **Engine Hours** (hours) to flash.
  - **Periodic saves** when **Δhours ≥ 1.0** or **Δodo ≥ 10.0 km** with a **10-minute guard** to protect flash.
  - **Load on boot** restores the last saved values.

- **End-of-drive save:**
  - On **IGN ON → OFF**, if the session lasted **≥ 2 minutes** or **≥ 1.0 km**.  
  - Uses a relaxed guard (≥ half the periodic interval) to catch shutdowns.

### Changed
- Serial logs now indicate save reasons:
  - `"[KV] Saved (periodic)."` and `"[KV] Saved (end-of-drive)."`.

### Optional Hardening
- Web handler guidance to **avoid constant POST overwrites** of `odometer`/`engineHours` (disable, guard with a flag, or only allow when ignition is OFF).

### Known Limitations
- Uses mbed **KVStore**; headers may differ across core versions. A **FlashStorage_STM32** variant can be provided if required.
- CRC/versioning of the persisted struct deferred to a future release.

---

## v1.0 — Baseline Stable Release


### Features
- **J1939 CAN broadcasts** suitable for ELD/telematics testing:
  - VIN via BAM.
  - EEC1 (PGN 61444), CCVS (PGN 65265), Engine Hours (PGN 65257),
  - Vehicle Distance (legacy), Fuel/MPG (PGN 65266), Temperatures, Battery Voltage.
- **Web UI** (Wi-Fi STA with AP fallback) for real-time control:
  - Ignition, Cruise, Throttle, Speed, Fuel, Temperature, Voltage.
- **Simulation loop** for generating realistic signals (speed, RPM, MPG, temps).
- **Odometer & Engine Hours** counters (non-persistent in this version).

### Notes
- Values reset on reboot (persistence added later in v5_20).

---

## Upgrade Notes

- **v1.0 → v1.1**
  - Add/enable KVStore persistence block.
  - Ensure thresholds are set:
    ```cpp
    static const unsigned long SAVE_INTERVAL_MS = 600000; // 10 min
    static const uint64_t      ODO_DELTA_SAVE   = 100;    // 10.0 km
    static const float         HRS_DELTA_SAVE   = 1.0f;   // 1.0 hour
    ```
  - Call `Persist_updateFromSim(...)`, `Persist_maybeSave()`, and `Persist_endOfDrive(...)` from the main loop.

- **v1.1 → v1.2**
  - Guard Web POST assignments for `odometer`/`engineHours` (disable, flag-gate, or accept only when ignition is OFF).
  - (Optional) Add Auto-drive helper so counters advance automatically in Auto mode.

---

## Known Issues / Future Work
- Add **CRC + version** to persisted struct with transparent migration.
- Optional **wear-leveling** slots for extra resilience.
- Expanded Auto-drive profiles (idle/accelerate/cruise/decel cycles).
