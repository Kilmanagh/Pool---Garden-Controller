# Smart Pool & Garden Irrigation Controller

An autonomous local controller built on an **ESP32** using **ESPHome** on **ESP-IDF**. It monitors pool and ambient temperature, drives a 4-channel relay board for 24V AC Orbit irrigation valves, and supports optional inline flow monitoring for pool autofill protection and daily water usage tracking.

---

## System Features

* **Local-only operation:** Uses a fixed local IP of `10.7.0.2` and a self-referential gateway for isolated network use.
* **No forced Wi-Fi reboot loop:** The controller keeps running locally even if the primary SSID is unavailable.
* **Fallback access point:** Exposes `Pool-Controller-Fallback` if the main Wi-Fi connection is unavailable, protected by its own password.
* **Safe relay defaults:** All valves are `inverted: true` and `restore_mode: ALWAYS_OFF` so reboot and power recovery default to valves off.
* **Autofill timeout protection:** Valve 1 starts a restart-safe timeout guard and shuts off automatically when the configured run limit is exceeded.
* **Optional flow-based protection:** If the flow sensor is connected and reporting, Valve 1 also enforces a no-flow / low-flow shutdown after a configurable startup grace period.
* **Daily water totalization:** Water use is tracked with `pulse_meter` totalization and reset to zero at midnight.
* **Freeze warning:** Ambient temperature drives a local binary freeze warning using a configurable threshold.
* **Startup readiness checks:** Valve starts are blocked until the temperature probes are reporting. Flow monitoring becomes active automatically once the flow sensor is wired and publishing state.
* **Runtime observability:** Tracks Wi-Fi state, API state, disconnect counters, boot count, heap health, reset reason, and an aggregated device health summary.

---

## Pinout & Hardware Architecture

| Component | ESP32 Pin | Logic Profile | Physical / Electrical Description |
| :--- | :--- | :--- | :--- |
| **Dallas 1-Wire Bus** | `GPIO4` | Bus Master | Connects to dual **DS18B20** waterproof probes (Pool & Air) |
| **Water Flow Sensor** | `GPIO5` | Pulse Input | Connected to an inline **YF-B5 (DN20)** Brass Hall-Effect sensor |
| **Pool Valve 1** | `GPIO16` | Active LOW | Relay 1 — Dedicated Pool Auto-Fill line with safety interlocks |
| **Garden Valve 2** | `GPIO17` | Active LOW | Relay 2 — Garden Zone 1 |
| **Garden Valve 3** | `GPIO18` | Active LOW | Relay 3 — Garden Zone 2 |
| **Garden Valve 4** | `GPIO19` | Active LOW | Relay 4 — Garden Zone 3 |

## Wiring Notes

### 1. 24V AC Valve Switching Loop

Orbit valves use a **24V AC** solenoid coil. The ESP32 does not drive the valves directly; the relay board acts as the isolated switching layer:
+-----------------------------------+
              |      24V AC Power Transformer     |
              +-----------------------------------+
                 |                             |
                 | [Line Wire]                 | [Common Return Wire]
                 v                             v
       +-------------------+         +------------------------+
       | Relay Board Terminal |         | Solenoid Parallel Rail |
       +-------------------+         +------------------------+
       | COM (Common Input)|         | Ties to one wire on    |
       +-------------------+         | EVERY Orbit valve      |
                 |                   +------------------------+
                 | [Switched Output]           |
                 v                             |
       +-------------------+                   |
       | NO (Normally Open)|                   |
       +-------------------+                   |
                 |                             |
                 v                             |
       +----------------------------------+    |
       | Dedicated Remaining Valve Wire   | <--+ (Completes the AC loop)
       +----------------------------------+

### 2. Optional Flow Sensor Interface (`GPIO5`)

The **YF-B5 / DN20** flow sensor is optional in the current configuration. If it is not connected, the controller still runs, but Valve 1 no-flow protection and daily water tracking remain inactive.

* **Red:** `5V DC`
* **Black:** `GND`
* **Signal:** `GPIO5`

`GPIO5` is a strapping pin on ESP32 hardware. ESPHome will warn about it during validation, so avoid adding external pull resistors that could interfere with boot.

---

## Hardware Bring-Up Checklist

### Stage 1: Base Controller

1. Wire the ESP32, relay board, power, and both DS18B20 probes.
2. Set `wifi_ssid`, `wifi_password`, and `fallback_ap_password` in `esphome/secrets.yaml`.
3. Flash the configuration and confirm Wi-Fi, API, and fallback AP behavior.
4. Verify `Pool Water Temperature` and `Pool Equipment Ambient Temperature` are both reporting.
5. Confirm `Pool Controller Self-Test Ready` turns on.
6. Confirm `Pool Controller Device Health` settles to `Healthy` or `WiFi OK / API Idle` after boot.
7. Test each relay output with the valves disconnected or otherwise made safe.

### Stage 2: Pool Autofill Safety

1. Connect Valve 1 and confirm it turns off after the configured `Auto Fill Max Run Time`.
2. Verify `Pool Valve 1 Last Stop Reason` changes to `timeout` when the timeout guard trips.
3. Check that a manual shutoff leaves the last-stop reason as `manual_off`.

### Stage 3: Flow Sensor Integration

1. Wire the flow sensor to `5V`, `GND`, and `GPIO5`.
2. Confirm `Pool Auto Fill Flow Monitor Ready` changes on once the sensor is publishing.
3. Run water through the line and verify `Water Flow Rate` and `Combined Daily Water Consumption` update.
4. Tune `Auto Fill Flow Start Delay` and `Auto Fill Minimum Flow Rate` using real readings.
5. Confirm Valve 1 now trips `Pool Auto Fill Flow Fault` if the valve opens without valid flow.

## Dashboard Controls And Telemetry

### Controls

* **Auto Fill Max Run Time:** Timeout threshold for Valve 1, from 1 to 120 minutes.
* **Auto Fill Flow Start Delay:** Grace period before no-flow protection evaluates Valve 1.
* **Auto Fill Minimum Flow Rate:** Minimum valid autofill flow in GPM before Valve 1 is treated as a flow fault.
* **Freeze Warning Threshold:** Ambient freeze-warning threshold in degrees Fahrenheit.

### Sensors

* **Pool Water Temperature:** Pool probe reading.
* **Pool Equipment Ambient Temperature:** Ambient equipment-area probe reading.
* **Water Flow Rate:** Real-time flow in GPM when the sensor is connected.
* **Combined Daily Water Consumption:** Daily gallon total from the flow sensor when connected.
* **Pool Controller Wi-Fi Signal:** Wi-Fi RSSI in dBm.
* **Pool Controller Uptime:** Controller uptime.
* **Pool Controller Internal Chip Temp:** ESP32 internal temperature.
* **Pool Controller Heap Free / Heap Fragmentation / Main Loop Time:** Low-level ESP32 runtime diagnostics.
* **Pool Controller Boot Count / Wi-Fi Disconnect Count / API Client Count / API Disconnect Count:** Long-term operational counters.

### Safety And Status Entities

* **Pool Controller Status:** Standard ESPHome online/offline status entity.
* **Pool Controller Wi-Fi Connected:** Explicit Wi-Fi connectivity state.
* **Pool Controller API Connected:** Explicit Home Assistant API connectivity state.
* **Pool Equipment Freeze Warning:** Set when ambient temperature falls below the configured threshold.
* **Pool Controller Self-Test Ready:** True when the two temperature probes are available and the controller is safe to start valves.
* **Pool Controller Self-Test Detail:** Reports `pool_probe_not_ready`, `ambient_probe_not_ready`, `ready_without_flow_sensor`, or `ready`.
* **Pool Auto Fill Flow Monitor Ready:** True when the flow sensor is connected and publishing state.
* **Pool Auto Fill Timeout Tripped:** Set when Valve 1 is shut down for timeout.
* **Pool Auto Fill Flow Fault:** Set when Valve 1 is shut down for low flow or missing flow.
* **Pool Controller Memory Warning:** Turns on if free heap gets too low or heap fragmentation gets too high.
* **Pool Valve 1 Last Stop Reason:** Latched reason string such as `running`, `manual_off`, `timeout`, `flow_fault`, or `self_test_blocked`.

### Text And Maintenance Entities

* **Pool Controller Reset Reason:** ESP32 reset cause from the debug integration.
* **Pool Controller Device Health:** Aggregated summary such as `Recovering After Boot`, `WiFi Reconnecting`, `WiFi OK / API Idle`, `Sensor Trouble`, `Memory Warning`, or `Healthy`.
* **Pool Controller IP Address / Wi-Fi SSID:** Current network identity values.
* **Pool Controller Restart:** Restart button for maintenance.

## Required Secrets

The configuration expects these keys in `esphome/secrets.yaml`:

* `wifi_ssid`
* `wifi_password`
* `fallback_ap_password`

Choose a real password for the fallback AP before putting the device into service.

## Validation

Use the ESPHome CLI to validate the configuration before flashing:

```powershell
esphome config .\esphome\pool-smart-controller.yaml
```
