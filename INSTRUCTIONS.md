📝 Pool & Garden Controller Project Brief
🚀 Overview
A local-only, high-reliability smart controller built on an ESP32 (ESP-IDF Framework) using ESPHome. It manages real-time temperature tracking, an active brass water flow sensor, and a 4-channel relay board driving 24V AC Orbit irrigation valves. It features embedded safety cutoffs and autonomous data tracking that functions independently of Home Assistant connectivity.

📌 Hardware Architecture & Pin Mapping
Component	ESP32 Pin	Electrical Details	Function / Behavior
Dallas 1-Wire Bus	GPIO4	3.3V Data Line	Interfaces with dual DS18B20 temperature probes.
Water Flow Sensor	GPIO5	5V Pulse Input	Connects to a brass YF-B5 (DN20) Hall-effect flow sensor.
Pool Valve 1	GPIO16	Digital Output (inverted: true)	Drives Relay 1 for the Pool Auto Fill line. Includes safety timers.
Garden Valve 2	GPIO17	Digital Output (inverted: true)	Drives Relay 2 for Garden Zone 1.
Garden Valve 3	GPIO18	Digital Output (inverted: true)	Drives Relay 3 for Garden Zone 2.
Garden Valve 4	GPIO19	Digital Output (inverted: true)	Drives Relay 4 for Garden Zone 3.
⚠️ Note on Hardware Inversion: All relay pins use inverted: true. This ensures that if the ESP32 loses power or reboots, the pins default to a state that keeps the physical 24V AC irrigation circuits Normally Open (OFF), preventing accidental flooding.

🧠 Core Software Logic & Features
1. Networking (Local-Only & Static)
Static IP Configuration: Locked to 10.7.0.2 on a custom subnet to ensure lightning-fast boot times and permanent local tracking.

Wireless Fallback: If the local Wi-Fi drops, the chip automatically spins up an ad-hoc Access Point named Pool-Controller-Fallback for local browser access.

2. Autonomous Safety Interlocks
Dynamic Auto Fill Timeout: When Pool Valve 1 turns ON, an internal ESP32 countdown timer pulls the runtime limit from an editable slider (id: autofill_timeout_setting). Once reached, it forces the valve closed and writes a safety alert to the local log.

On-Chip Freeze Warning Flag: The ambient air probe monitors temperature locally. If it drops below an editable threshold (defaults to 34°F), it flips a binary sensor (id: freeze_warning_status) to ON instantly, ready for Home Assistant notification automations.

3. Error-Free Volumetric Water Tracking
To bypass framework limitations with esp-idf, water consumption uses a Global Lambda Pulse Accumulator.

Raw high-speed pulses from the flow sensor are captured via on_raw_value and added directly to a volatile memory variable (total_water_pulses).

A template sensor continuously calculates total volume by dividing the pulse count by the sensor's physical constant (25000.0 pulses per gallon).

The accumulator syncs with Home Assistant's local time engine to automatically wipe the memory back to 0.0 gal at exactly midnight daily.