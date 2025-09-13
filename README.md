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

# Changelog

## v1.00 — 2025-09-13
- First stable baseline release
- Web UI for controls (ignition, cruise, throttle, speed)
- J1939 CAN broadcasts (VIN BAM, EEC1, CCVS, Engine Hours, Distance Legacy, Fuel, Temps, Voltage)
- Basic odometer & engine hours counters

