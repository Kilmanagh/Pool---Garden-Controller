# 🏊‍♂️ Smart Pool & Garden Irrigation Controller

An autonomous, ultra-reliable local automation hub built on an **ESP32 microcontroller** using the **ESPHome** framework under **ESP-IDF**. This system controls real-time pool and ambient temperatures, handles a high-speed brass volumetric water flow meter, and interfaces directly with a 4-channel relay board to safely drive 24V AC Orbit irrigation valves. 

---

## 🛠️ System Features

*   **🔒 Local-Only Operations:** Completely stripped of cloud dependencies. Operates on a dedicated static IP (`10.7.0.2`) for immediate boot recovery and offline local API performance.
*   **📡 Resilient Fallback Network:** Automatically deploys an ad-hoc captive portal wireless Access Point (`Pool-Controller-Fallback`) if the primary network drops.
*   **🚰 On-Chip Volumetric Tracking:** Bypasses framework bugs by using a native C++ variable pulse accumulator to track **Combined Daily Water Consumption** in exact gallons across all zones, automatically wiping back to `0.0 gal` at midnight.
*   **🛡️ Fail-Safe Hardware Mapping:** Every valve switch utilizes digital inversion (`inverted: true`). If the chip loses power, crashes, or reboots, the physical loops default to a state that keeps your valves **Closed**, preventing yard flooding.
*   **⏱️ Auto-Fill Runaway Protection:** Includes an autonomous runtime protective interlock loop. If the pool fill line is left open, it drops into a localized countdown script that automatically closes the valve at a threshold pulled directly from your dashboard slider.
*   **❄️ Dynamic Freeze Alert Engine:** Real-time evaluation of the physical air sensor against an adjustable temperature slider. Drops into an active warning state if temperatures threaten outdoor plumbing.

---

## 📌 Pinout & Hardware Architecture

| Component | ESP32 Pin | Logic Profile | Physical / Electrical Description |
| :--- | :--- | :--- | :--- |
| **Dallas 1-Wire Bus** | `GPIO4` | Bus Master | Connects to dual **DS18B20** waterproof probes (Pool & Air) |
| **Water Flow Sensor** | `GPIO5` | Pulse Input | Connected to an inline **YF-B5 (DN20)** Brass Hall-Effect sensor |
| **Pool Valve 1** | `GPIO16` | Active LOW | Relay 1 — Dedicated Pool Auto-Fill line with safety interlocks |
| **Garden Valve 2** | `GPIO17` | Active LOW | Relay 2 — Garden Zone 1 |
| **Garden Valve 3** | `GPIO18` | Active LOW | Relay 3 — Garden Zone 2 |
| **Garden Valve 4** | `GPIO19` | Active LOW | Relay 4 — Garden Zone 3 |

---

## ⚡ Wiring Blueprint

### 1. The 24V AC Valve Switching Loop
Orbit valves utilize a **24V Alternating Current** solenoid coil. Because the ESP32 operates purely on Direct Current (DC), the mechanical relay channels act as isolated dry contact switches to bridge the two lines safely:
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

 ## 2. High-Speed Flow Sensor Interface (`GPIO5`)
The **YF-B5 Brass Flow Sensor** monitors real-time velocity. It operates perfectly using the 5V rail off your primary project power distribution block.
*   🔴 **Red:** Connect to **5V DC**
*   ⚫ **Black:** Connect to **GND**
*   🟡 **Yellow/Blue (Signal):** Connect directly to **GPIO5** 
*(Note: The ESP32's internal hardware layout and dev board configuration keep this strapping pin safe from bootloader interference during startup).*

---

## 🎛️ Local Dashboard Controls Reference

*   **Pool Water Temperature (`sensor`):** Direct hardware evaluation of Dallas probe `0xaa000000127b0928`.
*   **Pool Equipment Ambient Temperature (`sensor`):** Direct hardware evaluation of Dallas probe `0x3e000000156d5d28`.
*   **Auto Fill Max Run Time (`number`):** Slider (1 to 120 minutes) setting the safety cutoff threshold for `valve_1`.
*   **Freeze Warning Threshold (`number`):** Slider (30°F to 45°F) evaluating the runtime trigger point for environmental alerts.
*   **Water Flow Rate (`sensor`):** Tracks realtime fluid speed in Gallons Per Minute (GPM). Uses a conversion filter of `0.04015` optimized for the YF-B5 scale.
*   **Combined Daily Water Consumption (`sensor`):** Calculated accumulation of total gallons pushed through the sensor pipe array across all 4 zones.