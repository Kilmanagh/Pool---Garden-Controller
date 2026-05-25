# Smart Pool & Garden Irrigation Controller

An autonomous local controller built on an **ESP32** using **ESPHome** on **ESP-IDF**. It monitors pool and ambient temperature, drives a 4-channel relay board for 24V AC Orbit irrigation valves, and supports optional inline flow monitoring for pool autofill protection and daily water usage tracking.

---

## System Features

* **Local-only operation:** Uses a fixed local IP of `10.7.0.2` and a self-referential gateway for isolated network use.
* **No forced Wi-Fi reboot loop:** The controller keeps running locally even if the primary SSID is unavailable.
* **Fallback access point:** Exposes `Pool-Controller-Fallback` if the main Wi-Fi connection is unavailable, protected by its own password.
* **Safe relay defaults:** All valves use `restore_mode: ALWAYS_OFF` so reboot and power recovery default to valves off.
* **Autofill timeout protection:** Valve 1 starts a restart-safe timeout guard and shuts off automatically when the configured run limit is exceeded.
* **Optional flow-based protection:** If the flow sensor is connected and reporting, Valve 1 also enforces a no-flow / low-flow shutdown after a configurable startup grace period.
* **Daily water totalization:** Water use is tracked with `pulse_meter` totalization and reset to zero at midnight using Home Assistant time.
* **Freeze warning:** Ambient temperature drives a local binary freeze warning using a configurable threshold.
* **Startup readiness checks:** Valve starts are blocked until the temperature probes are reporting. Flow monitoring becomes active automatically once the flow sensor is wired and publishing state.
* **Runtime observability:** Tracks Wi-Fi state, API state, disconnect counters, boot count, heap health, reset reason, and an aggregated device health summary.

---

## Requirements Status

### Implemented

* Static local network configuration with fallback AP.
* Dual DS18B20 temperature monitoring on `GPIO4`.
* Four relay outputs on `GPIO16` through `GPIO19` with safe boot defaults.
* Valve 1 timeout protection with configurable runtime.
* Local freeze warning flag with configurable threshold.
* Flow-based daily water tracking and no-flow protection when the flow sensor is installed.
* Local diagnostics for Wi-Fi/API state, reset reason, memory, uptime, and device health.

### Still Open Against The Original Brief

* Midnight daily reset still uses Home Assistant time. That means the water-total reset is not yet fully independent of Home Assistant connectivity.
* The current configuration supports running without the flow sensor installed, but the original project brief assumes that sensor is part of the final hardware.

---

## Pinout & Hardware Architecture

| Component | ESP32 Pin | Logic Profile | Physical / Electrical Description |
| :--- | :--- | :--- | :--- |
| **Dallas 1-Wire Bus** | `GPIO4` | Bus Master | Connects to dual **DS18B20** waterproof probes (Pool & Air) |
| **Water Flow Sensor** | `GPIO27` | Pulse Input | Connected to an inline **DN20 / 3/4 NPT brass hall-effect water flow sensor** |
| **Pool Fill Valve 1** | `GPIO16` | Active HIGH | Relay 1 — Dedicated pool auto-fill line with safety interlocks |
| **Garden Valve 2** | `GPIO17` | Active HIGH | Relay 2 — Garden Zone 1 |
| **Garden Valve 3** | `GPIO18` | Active HIGH | Relay 3 — Garden Zone 2 |
| **Pool Purge Valve** | `GPIO19` | Active HIGH | Relay 4 — Dedicated pool purge / drain path with timeout protection |

## Wiring Notes

### 1. Relay Module Assembly

Verified project infographic: [assets/relay-module-assembly-accurate.svg](assets/relay-module-assembly-accurate.svg)

This project now targets the common **4-channel 5V optocoupler relay board** with input terminals labeled `DC+`, `DC-`, `IN1`, `IN2`, `IN3`, and `IN4`, plus per-channel jumper selectors for **high-level** or **low-level** trigger.

ESP32 to relay-module control wiring:

* `GPIO16` -> `IN1` for **Pool Fill Valve 1**
* `GPIO17` -> `IN2` for **Garden Valve 2**
* `GPIO18` -> `IN3` for **Garden Valve 3**
* `GPIO19` -> `IN4` for **Pool Purge Valve**
* ESP32 `GND` -> relay board `DC-`

Relay board power:

* Relay board `DC+` -> `5V`
* Relay board `DC-` -> ESP32 `GND`
* Do **not** power this board from the ESP32 `3V3` rail. This is a `5V` relay board.

Trigger mode for this board:

* Set all four jumper selectors to **high-level trigger**.
* On boards with `H / L` markings, that means the jumper should connect the center pin to the pin marked `H` for each channel.
* This matches the YAML, which uses `inverted: false` and therefore expects an **active-high relay input**.

This YAML uses `inverted: false` on every relay output, so the design assumes an **active-high relay input**: driving the input high energizes the relay, and boot / reset defaults leave valves off.

In practical operation, that means the ESPHome / Home Assistant valve switch entities are now **active-high**: turning a switch `ON` drives the GPIO high and energizes the matching relay, and turning it `OFF` releases the relay.

Relay output terminals:

* Use `COM` and `NO` for each valve circuit.
* Do **not** use `NC` unless you intentionally want a valve energized when the controller is idle or powered down.

Exact 24VAC terminal mapping:

* 24VAC transformer **hot / line** -> jumper or distribute to all four relay `COM` terminals
* Relay 1 `NO` -> **Pool Fill Valve 1** control wire
* Relay 2 `NO` -> **Garden Valve 2** control wire
* Relay 3 `NO` -> **Garden Valve 3** control wire
* Relay 4 `NO` -> **Pool Purge Valve** control wire
* 24VAC transformer **common / return** -> shared common wire that goes to the second wire on **every** valve

Valve-side wiring list:

* **Pool Fill Valve 1**: one solenoid wire to relay 1 `NO`, other solenoid wire to the shared 24VAC common
* **Garden Valve 2**: one solenoid wire to relay 2 `NO`, other solenoid wire to the shared 24VAC common
* **Garden Valve 3**: one solenoid wire to relay 3 `NO`, other solenoid wire to the shared 24VAC common
* **Pool Purge Valve**: one solenoid wire to relay 4 `NO`, other solenoid wire to the shared 24VAC common

Simple terminal diagram:

```text
24VAC transformer hot  ----+---- COM1
                           +---- COM2
                           +---- COM3
                           +---- COM4

NO1 ---------------------------- Pool Fill Valve 1 lead A
NO2 ---------------------------- Garden Valve 2 lead A
NO3 ---------------------------- Garden Valve 3 lead A
NO4 ---------------------------- Pool Purge Valve lead A

24VAC transformer common ---+--- Pool Fill Valve 1 lead B
                            +--- Garden Valve 2 lead B
                            +--- Garden Valve 3 lead B
                            +--- Pool Purge Valve lead B
```

Timeout safety rules:

* **Pool Fill Valve 1** has a configurable auto-off timeout using **Auto Fill Max Run Time**, default `20` minutes.
* **Pool Purge Valve** has its own configurable auto-off timeout using **Pool Purge Max Run Time**, default `15` minutes.
* The purge timeout exists specifically to reduce the chance of draining too much water if the purge valve is turned on and forgotten.

Recommended bench test before connecting the 24V valves:

1. Power the ESP32 and relay board only.
2. Toggle each valve switch from Home Assistant or the ESPHome web interface.
3. Confirm the matching relay clicks and its indicator LED changes.
4. Confirm all relays remain off during boot and reset.

Dry-bench checklist before adding 24VAC:

1. Reflash the current YAML and confirm the board boots cleanly.
2. Verify all four relay entities switch cleanly: `ON` should energize the relay, `OFF` should release it.
3. Meter `COM` to `NO` on each relay channel with no 24VAC connected.
4. Confirm both DS18B20 probes report sane temperatures.
5. Leave the controller powered for a while and confirm there is no relay chatter or reboot loop.

Valve status verification:

The controller now exposes a text status entity for each valve.

* `running` means that relay output is currently on.
* `idle` means the relay is currently off and the controller self-test is ready.
* `blocked` means the relay is off because the controller self-test is not ready yet, usually because one or both temperature probes are not reporting.

How to verify `blocked`, `idle`, and `running` on the bench:

1. Boot normally with both temperature probes reporting: each valve status should settle at `idle` when off.
2. Turn a valve on: that valve status should change to `running`.
3. Turn it back off: that valve status should return to `idle`.
4. To verify `blocked`, disconnect a temperature probe or otherwise force the controller self-test to fail, then try turning on a valve. The valve should refuse to start and the corresponding status should read `blocked`.

### 2. 24V AC Valve Switching Loop

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

### 3. Optional Flow Sensor Interface (`GPIO27`)

The controller is compatible with the common **DN20 / 3/4 NPT brass hall-effect flow sensor** class, including parts sold as **1-30 L/min**, **DC 5-15V**, and similar to the Lazyfun unit you referenced. If it is not connected, the controller still runs, but Valve 1 no-flow protection and daily water tracking remain inactive.

Recommended safe wiring for this sensor class:

* **Red:** Sensor power to `5V DC`
* **Black:** Sensor ground to `GND`
* **Yellow / Signal:** Sensor signal to a resistor divider, then to `GPIO27`
* **Common ground:** ESP32 ground and sensor ground must be tied together

`GPIO27` is still the right ESP32 pin for the signal wire. It is a safer general-purpose input than `GPIO5`, which was previously used and is a strapping pin.

For the common DN20 1-30 L/min hall sensors, the safest assumption is that the output high level can be near the sensor supply voltage when powered from `5V`. That means the signal should be treated as a `5V` pulse unless you personally verify otherwise with a meter or datasheet.

Recommended divider:

* Sensor signal to `10k` resistor
* Other side of that resistor to `GPIO27`
* `GPIO27` to `20k` resistor
* Other side of the `20k` resistor to `GND`

That scales a `5V` pulse down to about `3.3V`, which is appropriate for the ESP32 input.

Practical resistor alternatives that also work:

* **Preferred if you have these on hand:** `4.7k` from sensor signal to `GPIO27`, and `10k` from `GPIO27` to `GND`
* **Also acceptable:** `10k` from sensor signal to `GPIO27`, and two `10k` resistors in series from `GPIO27` to `GND` to make `20k`

The `4.7k` and `10k` option scales a `5V` pulse to about `3.4V`, which is close enough for this use and is a practical off-the-shelf pairing.

Simple wiring diagram:

```text
Sensor red    -> 5V
Sensor black  -> GND
Sensor yellow -> 10k resistor -> GPIO27
GPIO27        -> 20k resistor -> GND
ESP32 GND     -> same GND as sensor


                    DN20 Flow Sensor                       ESP32
                 +------------------+              +----------------+
                 |                  |              |                |
5V  ------------>| Red / VCC        |              |                |
                 |                  |              |                |
GND -------------+ Black / GND      +--------------+ GND            |
                 |                  |              |                |
                 | Yellow / Signal  +---[ 10k ]----+ GPIO27         |
                 +------------------+              |      |         |
                                                   |    [ 20k ]     |
                                                   |      |         |
                                                   +------+---------+
                                                          |
                                                         GND
```

If you later prove that your exact sensor output is open-collector and only needs a pull-up, you can simplify the wiring. Until then, the divider is the safer default.

The advertised `DC 5-15V` operating range is acceptable for the sensor power input, but it does **not** guarantee that the signal output is automatically safe for a `3.3V` MCU input.

The current flow conversion constants are a starting point only. Expect to tune `Water Flow Rate` and `Daily Water Total` against real measured water volume once the sensor is installed.

---

## Full Wiring Checklist

Use this as the final point-to-point wiring list for the whole controller.

| From | To | Notes |
| :--- | :--- | :--- |
| ESP32 `5V` | Relay board `DC+` | Powers the 5V relay board |
| ESP32 `GND` | Relay board `DC-` | Required common ground |
| ESP32 `GPIO16` | Relay board `IN1` | Pool Fill Valve 1 control |
| ESP32 `GPIO17` | Relay board `IN2` | Garden Valve 2 control |
| ESP32 `GPIO18` | Relay board `IN3` | Garden Valve 3 control |
| ESP32 `GPIO19` | Relay board `IN4` | Pool Purge Valve control |
| Relay board jumpers | `H` / high-level trigger position | Required for `inverted: false` YAML behavior |
| 24VAC transformer hot | Relay `COM1` | Relay 1 common input |
| 24VAC transformer hot | Relay `COM2` | Relay 2 common input |
| 24VAC transformer hot | Relay `COM3` | Relay 3 common input |
| 24VAC transformer hot | Relay `COM4` | Relay 4 common input |
| Relay `NO1` | Pool Fill Valve 1 lead A | Switched hot for autofill valve |
| Relay `NO2` | Garden Valve 2 lead A | Switched hot for garden zone 1 |
| Relay `NO3` | Garden Valve 3 lead A | Switched hot for garden zone 2 |
| Relay `NO4` | Pool Purge Valve lead A | Switched hot for controlled purge path |
| 24VAC transformer common | Pool Fill Valve 1 lead B | Shared common return |
| 24VAC transformer common | Garden Valve 2 lead B | Shared common return |
| 24VAC transformer common | Garden Valve 3 lead B | Shared common return |
| 24VAC transformer common | Pool Purge Valve lead B | Shared common return |
| ESP32 `GPIO4` | DS18B20 data bus | Shared 1-Wire data line for both temp probes |
| ESP32 `3V3` | DS18B20 VCC | Power for both temperature probes if they are 3.3V-compatible |
| ESP32 `GND` | DS18B20 GND | Common ground for both temperature probes |
| DS18B20 data bus | `4.7k` pull-up to `3V3` | Standard 1-Wire pull-up resistor |
| ESP32 `5V` | Flow sensor red / VCC | Flow sensor power |
| ESP32 `GND` | Flow sensor black / GND | Flow sensor ground |
| Flow sensor yellow / signal | `10k` or `4.7k` resistor to `GPIO27` | Top leg of divider |
| `GPIO27` | `20k` or `10k` resistor to `GND` | Bottom leg of divider |

Quick sanity rules:

* Use relay `COM` and `NO`, not `NC`.
* Keep the relay board in **high-level trigger** mode.
* Keep all grounds common on the low-voltage side.
* Keep the 24VAC valve wiring electrically separate from the ESP32 signal wiring except through the relay contacts.

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

1. Wire the flow sensor to `5V`, `GND`, and `GPIO27` through the documented `10k/20k` resistor divider.
2. Confirm `Pool Auto Fill Flow Monitor Ready` changes on once the sensor is publishing.
3. Run water through the line and verify `Water Flow Rate` and `Daily Water Total` update.
4. Tune `Auto Fill Flow Start Delay` and `Auto Fill Minimum Flow Rate` using real readings.
5. Calibrate the flow conversion against a known water volume, because marketplace listings for DN20 hall sensors often omit or misstate the pulse constant.
6. Confirm Valve 1 now trips `Pool Auto Fill Flow Fault` if the valve opens without valid flow.

## Flow Sensor Calibration

The current configuration uses a starting multiplier of `0.00004` for both live flow and total gallons. That assumes roughly `25,000 pulses per gallon`, which is a reasonable placeholder for a generic DN20 hall sensor but should not be treated as final calibration.

You cannot fully calibrate the sensor until water is actually flowing. Before that, you can only verify the wiring, the pulse input path, and that ESPHome recognizes the sensor once pulses are present.

When water is available, calibrate it like this:

1. Start with the current multiplier in the YAML.
2. Run a known measured volume through the sensor, preferably `1` to `5` gallons.
3. Note the reported `Daily Water Total`.
4. Update the multiplier using this formula:

   `new multiplier = old multiplier * (actual volume / reported volume)`

Example:

* Old multiplier: `0.00004`
* Actual measured volume: `5.00 gal`
* Reported volume: `4.20 gal`
* New multiplier: `0.00004 * (5.00 / 4.20) = 0.00004762`

After adjusting the multiplier, reflash the controller and repeat the test until the reported total is close to the measured volume. Once total gallons are calibrated, the live `Water Flow Rate` reading will usually become much more believable as well, since both values are derived from the same pulse stream.

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
* **Daily Water Total:** Daily gallon total from the flow sensor when connected.
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
