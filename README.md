# LocuSense – Low-Power CO₂ / T / RH / VOC Sensor Node

LocuSense is a **battery-powered environmental sensor node** designed for ultra-low power operation.  
It measures:

- CO₂ (SCD41)  
- Temperature & humidity (SHT41)  
- VOC index (SGP40 + Sensirion VOC algorithm, optional / USB mode)

and can communicate either via:

- **LoRaWAN (Wio-E5)** – confirmed uplinks with change/heartbeat policy  
- **Matter over Thread (ESP32-C6)** – Sleepy End Device, fully integrated with Home Assistant

The node includes a **2.13" e-ink display** for local status and is optimized for **very long runtime** from a primary **Li-SOCl₂** cell, with optional **Li-Ion + USB charging**.

---

## Features

### Hardware

- **MCU:** STM32U0 (ultra-low power)
- **Sensors:**
  - SCD41 – CO₂ + T + RH (CO₂+TRH mode)
  - SHT41 – T + RH (TRH-only mode)
  - SGP40 – VOC raw + VOC index (Sensirion gas index algorithm)
- **Power:**
  - Primary: Li-SOCl₂ cell (long lifetime)
  - Secondary: Li-Ion (1S) with **BQ25185** charger
  - USB (5 V) as power & Li-Ion charging source
  - Separate power switches for:
    - Sensors rail (PS1)
    - RF rail (PS2)
    - E-ink display rail (PS4)
  - Switchable pull-ups, voltage dividers etc. for minimum sleep current
- **Display:**
  - 2.13" e-ink (EPD_2in13_V4, 250×122)
  - Local UI: CO₂ / T / RH / VOC, battery, USB, sleep interval, LoRa/Matter status, emoji air quality indicator
- **Connectivity:**
  - Wio-E5 (LoRaWAN, AT commands over UART)
  - ESP32-C6 (Matter over Thread, UART text protocol with WAKE / READY handshake)

### Power & Sleep

- **LoRaWAN mode (Wio-E5):**  
  - STOP2 sleep with radio in low-power mode  
  - Typical sleep current ≈ **4 µA**
- **Matter over Thread mode (ESP32-C6 SED):**
  - Sleepy End Device with light sleep on ESP side  
  - Average current ≈ **4 mA** (Thread always-on)
- All peripherals can be completely powered down:
  - Sensor rail
  - RF rail
  - E-ink rail
  - I²C pull-ups, battery dividers, etc.

---

## Firmware Overview

The repository contains **two firmware projects** plus hardware design and Home Assistant configuration:

- `firmware/stm32/` – STM32U0 main application
- `firmware/esp32c6/` – ESP32-C6 Matter over Thread application
- `hardware/altium/` – Altium Designer project (baseboard + RF modules)
- `home-assistant/` – Home Assistant, InfluxDB and Grafana configuration / examples

### STM32U0 Application

Main file: `app.c`

**Key responsibilities:**

- System state machine:
  - `STATE_INIT` – cold boot, config load, radio init
  - `STATE_MEASURE` – sensor measurements (CO₂/T/RH or T/RH only)
  - `STATE_BAT_MEASURE` – battery + VDDA measurement via ADC
  - `STATE_SEND_DATA` – send payload via LoRa or ESP
  - `STATE_RECOVER` – LoRa rejoin / ESP HW reset & status
  - `STATE_TIME_REQ` – LoRa time sync (port 8) via WioE5
  - `STATE_GUI` – full screen e-ink redraw (change-based)
  - `STATE_SLEEP` – STOP2 sleep / VOC idle loop
  - `STATE_RECALIBRATION` – SCD41 forced recalibration to 400 ppm
  - `STATE_CONFIG` – UART configuration console
- Payload format (6 bytes):
  - T in 0.01 °C (signed)  
  - RH in 0.01 %RH  
  - CO₂ in ppm  
  ```
  [0..1] = temperature_01C
  [2..3] = humidity_01pct
  [4..5] = co2_ppm
  ```
- **TX policy** (used for both LoRa & Matter):
  - First measurement → always send
  - Then: respect `tx_min_interval_sec`
  - Heartbeat after `tx_max_interval_sec`
  - Or immediate send when thresholds exceeded:
    - `th_temp_01C`
    - `th_rh_01pct`
    - `th_co2_ppm`
- **Battery measurement:**
  - Internal VREF calibration
  - Two ADC channels (Li-Ion, Li-SOCl₂) with common divider
  - Results in mV stored in `app.vbat_liion_mV` and `app.vbat_lsocl2_mV`
  - Optional sending to ESP as `DATA BAT KV ...` frame
- **VOC handling:**
  - VOC mode (`vocMode`):
    - `VOC_DISABLED`
    - `VOC_CONT_USB` – continuous VOC sampling while USB is connected
  - Uses LPTIM2 with 10 s tick
  - SGP40 raw readings processed by `sensirion_gas_index_algorithm`
  - VOC index and timestamp shown on e-ink, optional partial updates only

### UART Text Protocol (STM32 ↔ ESP32-C6)

Communication is done via **simple text frames over UART1** with a **hardware handshake**:

- STM32 drives `WAKE` (to ESP) and reads `READY`
- ESP drives `READY` (to STM32) and reads `WAKE`
- Protocol examples:
  - Environmental data:
    - `DATA HEX AABBCCDDEEFF`
    - `DATA KV T=<i16_01C> RH=<u16_01pct> CO2=<u16_ppm>`
  - Battery / power sources:
    - `DATA BAT KV LI=<mV> LS=<mV> USB=<0|1> CHG=<0..6>`
  - Commands from STM32 to ESP:
    - `PING`
    - `STATUS?`
    - `QR?` / `GET QR`
    - `VERSION?`
    - `FABRICS?`
    - `COMM START` / `COMM STOP`
    - `FACTORYRESET`

The ESP replies with newline-terminated text (`STATUS ONLINE`, `COMM STATE ...`, QR & manual code strings, etc.).

### ESP32-C6 Matter Application

Main file: `app_main()` in `esp32c6` project.

**Key points:**

- Uses **esp-matter** + **Thread SED** configuration
- Creates the following endpoints:

1. **Temperature sensor** (TemperatureMeasurement cluster)
2. **Humidity sensor** (RelativeHumidityMeasurement cluster)
3. **CO₂ / Air Quality sensor**
   - AirQuality cluster (0x005B): AirQuality enum derived from CO₂
   - Custom CO₂ concentration cluster (0x040D) with MEA feature:
     - MeasuredValue (float, ppm)
     - Min/MaxMeasuredValue
     - Uncertainty
     - MeasurementUnit (ppm)
     - MeasurementMedium (air)
4. **Power Source endpoints:**
   - USB (wired power)
   - Li-Ion battery
   - Li-SOCl₂ battery

- Receives `DATA ...` frames from STM32 and updates Matter attributes accordingly
- Implements **commissioning over UART commands**:
  - `COMM START` → opens commissioning window, prints QR + manual code
  - `COMM STOP` → closes window
  - `FABRICS?` → lists fabrics
  - Fabric removal and factory reset through Matter / internal helpers

- Generates Matter **QR code** and **manual code** on the ESP side and sends them to STM32:
  - STM32 draws QR on e-ink display (via `GUI_DrawQR_TextFull()`)

---

## Configuration Console (STM32, UART4)

The node exposes a **minimal CLI** over UART4 for all configuration, including LoRa credentials and ESP pairing info.  

Enter CONFIG mode by long-pressing the button (or via firmware). The console supports:

- General:
  - `HELP` – print help
  - `SHOW` – print current config + Wio-E5 keys + ESP status/QR
  - `SET <key> <value>` – set AppConfig fields (intervals, thresholds, modes)
  - `RESET` – restore firmware defaults (AppConfig only, LoRa/ESP data unchanged)
  - `SAVE` – store AppConfig + Wio OTAA + ESP pairing (QR/manual) to EEPROM
  - `EXIT` – leave config mode
- LoRa / Wio-E5:
  - `WIO SHOW`
  - `WIO SET APP_EUI <16hex>`
  - `WIO SET DEV_EUI <16hex>`
  - `WIO SET APP_KEY <32hex>`
  - `WIO SET ADR <0|1>`
  - `WIO JOIN`
  - `WIO SEND HEX ... / KV ...`
  - `WIO TIME` – network time via downlink
- ESP32-C6:
  - `ESP PING`
  - `ESP STATUS`
  - `ESP VERSION`
  - `ESP QR`
  - `ESP FABRICS`
  - `ESP FACTORYRESET`
  - `ESP COMM START` / `ESP COMM STOP`
  - `ESP SEND HEX ... / KV ...`

The console is idle-timeout protected (default 10 min inactivity → auto exit).

---

## E-Ink UI

The GUI is implemented in `GUI.c`:

- **Main screen** (full refresh):
  - CO₂, temperature, humidity, VOC index
  - Trend arrows (up/down/flat) per parameter
  - Air quality emoji (happy / neutral / sad / dead) based on CO₂+T
  - CO₂ mini gauge with thresholds (800 / 1200 ppm)
  - Bottom bar with:
    - Li-SOCl₂ voltage / presence
    - Li-Ion voltage / presence
    - USB state & charger status
    - Sleep interval formatted (`xh ym`, etc.)
- **Partial updates:**
  - State line (`STATE: MEASURE`, `STATE: SLEEP`, …)
  - VOC line (just VOC + timestamp)
  - Config-mode status strip (`SET key=val : OK/ERR`)
  - QR full-screen render for Matter commissioning

GUI refresh is **change-based** (thresholds aligned with TX policy) to avoid unnecessary e-ink wear and power usage.

---

## Home Assistant & Data Pipeline

In **Matter mode (ESP32-C6)** the device appears in Home Assistant as:

- Temperature sensor
- Humidity sensor
- CO₂ / air quality sensor
- Power source entities (USB, Li-Ion, Li-SOCl₂)

The repository includes (or will include):

- `home-assistant/` – sample HA config / dashboard
- InfluxDB integration (time-series storage)
- Grafana dashboards (visualization of CO₂ / T / RH / VOC / battery)

Typical flow:

1. LocuSense → Matter over Thread → Home Assistant  
2. Home Assistant → InfluxDB → Grafana dashboard

---

## Repository Structure

Suggested structure (adapt if needed):

```text
.
├── firmware
│   ├── stm32/         # STM32U0 application (CubeIDE / Makefile)
│   └── esp32c6/       # ESP32-C6 Matter over Thread project
├── hardware
│   └── altium/        # Altium Designer projects (baseboard + RF modules)
├── home-assistant/
│   ├── dashboards/    # HA dashboards (YAML, Lovelace) and screenshots
│   ├── influxdb/      # Example retention / bucket config
│   └── grafana/       # Example Grafana dashboards (JSON)
└── README.md
```

---

## Building & Flashing

### STM32U0 Firmware

- Toolchain: STM32CubeIDE or any arm-none-eabi-gcc based setup.
- Dependencies:
  - STM32U0 HAL / LL
  - Drivers for:
    - SCD41, SHT41, SGP40
    - Wio-E5 (AT driver)
    - BQ25185
    - EPD_2in13_V4 + paint library
    - M24C02 EEPROM
  - Sensirion VOC algorithm library
- Build, flash via SWD (ST-Link), SWD pins are left enabled in firmware (note: they can be disabled in production).

### ESP32-C6 Firmware

- Toolchain: ESP-IDF (v5.x) + esp-matter  
- Configure project:
  - Enable Thread + SED mode
  - Configure storage / NVS
- Build & flash:
  ```bash
  idf.py set-target esp32c6
  idf.py build
  idf.py flash monitor
  ```

---

## Modes & Configuration Summary

Config keys (via `SET <key> <value>` in console):

- `measure_mode` / `mm`
  - `0` = CO₂ + T + RH (SCD41 + SHT41)
  - `1` = T + RH only (SHT41)
- `voc_mode` / `voc`
  - `0` = VOC_DISABLED
  - `1` = VOC_CONT_USB (VOC only when USB is powered)
- `comms_mode` / `comms`
  - `0` = COMMS_OFFLINE (no radio, GUI only)
  - `1` = COMMS_LORA (Wio-E5 LoRaWAN)
  - `2` = COMMS_MATTER (ESP32-C6, Matter over Thread)
- `interval_measure_sec` / `tm`
- `interval_sleep_sec` / `ts`
- `interval_time_req_sec` / `tt` (LoRa time sync)
- `interval_bat_sec` / `tb`
- `tx_min_interval_sec` / `txmin`
- `tx_max_interval_sec` / `txmax` (heartbeat)
- `th_temp_01C` / `dT`
- `th_rh_01pct` / `dRH`
- `th_co2_ppm` / `dCO2`

Don’t forget to call `SAVE` if you want to persist changes to EEPROM.

---

## License

TBD.  
Choose a license (e.g. MIT, Apache-2.0, GPL) and put it here.

---

## Status

This project is under active development.  
Hardware, STM32 firmware, ESP32-C6 Matter firmware and Home Assistant dashboards are all included in this repository.

Contributions and issue reports are welcome.
