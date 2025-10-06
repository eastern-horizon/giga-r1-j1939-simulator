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
# v1.3 — quality of life, dual speed control

## Highlights

v1.4 — Dual Speed Control. You can now type an exact speed or use the slider; both stay perfectly in sync.


# v1.3 — Fuel realism, LFE, TP.BAM fix, distance PGNs, persistence

## Highlights

* **More realistic fuel model** with DFCO, idle, linear & aero terms, and UI-safe clamps. Also fixes the fuel-level slider behavior by using a single “source of truth” for liters remaining. &#x20;
* **LFE (Fuel Economy / PGN 65266)** now transmitted with correct SPNs and scaling; hides instant MPG at low speeds to avoid noisy values.&#x20;
* **VIN over TP.BAM**: reserved byte in CM frame corrected to **0xFF** (BAM), not “7”. This aligns with J1939-21. &#x20;
* **Vehicle distance**: supports both legacy Vehicle Distance (PGN 65217) and **Total/Trip Vehicle Distance High Resolution** (PGN 65248). &#x20;
* **Persistent odometer & hours**: values are loaded at boot and continuously synced back to KV; precise integer counters (meters/seconds) drive the floats so they never drift.  &#x20;

## Added

* **PGN 65266 (LFE)** builder with SPN-correct mph→MPG conversion, and DFCO behavior carrying through to instant MPG (shows NA/steady when appropriate).&#x20;
* **High-resolution distance (PGN 65248)** alongside the legacy odometer PGN for broader tool compatibility.&#x20;

## Changed

* **Fuel-use & fuel-level**: model runs entirely in **L/h**, integrates consumption per tick, and uses a single global `LitersRemaining` to derive `% fuelLevel` (fixes slider disagreements & race conditions).&#x20;
* **Instant/Average MPG display**: clamps for UI sanity and hides instant MPG when `vehicleSpeed <= 5 mph`.&#x20;
* **TP.BAM CM byte 5 (reserved)** now `0xFF` for VIN BAM; also standardized the TP.CM/TP.DT identifiers via `j1939Id(...)`.  &#x20;
* **PGN & SPN correctness pass** (selection):

  * **Fuel Level 1** uses PGN **0xFEFC (65276)** with proper 0.4%/bit mapping and byte positions.&#x20;
  * **Vehicle Electrical Power (VEP / 65271)** battery potential **SPN 168** placed at bytes 5–6 with 0.05 V/bit.&#x20;
  * **Engine Temp / ET1 (65262)** encodings clarified (Kelvin-based oil, −40 °C offset where applicable).&#x20;
  * **CCVS (Cruise/Vehicle Speed) flags** and SPN 586 packed per spec. &#x20;

## Fixed

* **Fuel slider** sticking at 100% when toggling Auto MPG: resolved by eliminating local copies and driving `%` from global liters remaining.&#x20;
* **VIN BAM** CM\[4] incorrect value (`7`) in older snapshots—now normalized to `0xFF` everywhere we emit or schedule BAM.&#x20;

## Internals / Stability

* **Precise counters** (`ODO_METERS`, `ENG_SECONDS`) tick in whole units and backfill the user-visible floats during each update cycle. This keeps legacy math and adds accuracy. &#x20;
* **CAN init & interrupt**: explicit flagging and D2 interrupt attach logged at startup for easier bring-up.&#x20;
* **VIN BAM scheduling**: simple state machine with pacing gap to avoid floods on slow busses. &#x20;

## Tunables (fuel model defaults)

Idle 2.2 L/h; base 3.0 L/h; linear 0.50 L/h per mph; aero 0.0025·mph²; DFCO threshold 8 mph. Tweak in code comments if you need a different tractor/trailer profile.&#x20;

## Known Issues / Notes

* Instant MPG is suppressed below \~5 mph by design to prevent division noise; tools expecting a number at 0–5 mph may see last good value or clamp.&#x20;
* If your downstream expects only one distance PGN, you may disable either legacy or HR sender in `sendJ1939_ODOMETER()`.&#x20;

## Upgrade Guide

1. Flash v1.3. On first boot we **load** persisted odo/hours and **seed** precise counters; from then on, the counters are the source of truth and we keep KV in sync automatically. &#x20;
2. Verify CAN @ 250 kbps initializes (console shows ✅) and that VIN is broadcast once via BAM at startup. &#x20;
3. (Optional) Adjust fuel model constants to match your platform’s burn rates.&#x20;



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
