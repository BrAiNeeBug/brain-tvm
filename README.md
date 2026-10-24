# brain-tvm
<img width="920" height="945" alt="image" src="https://github.com/user-attachments/assets/46394f3d-b300-46db-aedd-f6e7f2244ef0" />


ESPHome firmware for a small ESP32 display device (**NM-TV**): a 240x240 ST7789 screen with one touch button, showing a clock, system status, a BLE device counter, a free-form info card and a light page. Everything is controlled from Home Assistant or the built-in web UI.

One YAML file, no custom C++ components.

## Features

- **6 display pages** in neon green: Welcome, System status, Card, Clock, BLE scanner, Light
- **Touch button**: short tap = next page, hold = push button of the current page (`PushLight` on the light page)
- **Spike filter** for the touch input (`touch_delay`), so a piezo lighter or similar noise does not switch pages
- **Auto page rotation**: choose pages and interval at runtime
- **Clock page** with English/German switch (`ClockLanguage`), NTP time
- **Card page**: send a JSON to the `Card` text entity and it shows title and up to 8 lines in 4 sizes
- **BLE scanner** counting nearby devices (current and peak), can be switched on and off
- **Web UI** (v3) with groups, plus OTA, safe mode and encrypted API
- Brightness, rotate interval and more as runtime entities, no reflash needed

## Hardware

| Part | Pin |
|---|---|
| ESP32 (`esp32dev`, chip revision 3.1+) | |
| ST7789 240x240 display, SPI CLK / MOSI | GPIO14 / GPIO13 |
| Display CS / DC | GPIO15 / GPIO2 |
| Display power (inverted) | GPIO21 |
| Backlight (PWM, inverted) | GPIO19 |
| Touch pad | GPIO32 |

## Installation

1. Copy `brain-tvm.yaml` into your ESPHome config folder.
2. Add these secrets to your `secrets.yaml`:

   ```yaml
   brain_tvm__encryption_key: "<base64 32 byte key>"
   wifi_ssid: "..."
   wifi_password: "..."
   ap_password: "..."
   webgui_password: "..."
   ```

3. Adjust the `substitutions` at the top (`device_name`, `localdomain`, `touch_delay`, `long_press_ms`).
4. Flash it. The first flash must be over USB, later updates via OTA.
5. Add the device in Home Assistant (ESPHome integration).

## Pages

| # | Page | Content |
|---|---|---|
| 1 | Welcome | "WELCOME", "TOUCH ME" and a hint to tap for the next page |
| 2 | Status | Host, IP, SSID, signal, MAC, uptime, free heap, ESPHome version, build date, BLE count |
| 3 | Card | Content you send to the `Card` entity (only shown once something was sent) |
| 4 | Clock | Weekday, time and date |
| 5 | BLE | Number of BLE devices around, with peak |
| 6 | Light | Static light page, `PushLight` is active while you hold the touch |

Pages can be changed with the `DisplayPage` select. `RotatePages` takes a list like `3,4` or `1-4` or `1,3-6`, `RotateInterval` sets the time.

## Touch

- **Short tap** (on release): next page
- **Hold** (`long_press_ms`, default 600 ms): the push button of the current page goes active until you release. Currently only the light page has one (`PushLight`), use it as a trigger in Home Assistant.
- `touch_delay` (default 80 ms): the touch must be active at least this long. Raise it if short noise spikes still switch pages.

## Card

Send a JSON string to the `Card` text entity (max 255 characters). Every send replaces the whole card.

```json
{"title": "Server", "line1": "CPU 12%", "size1": "xl", "line2": "RAM 45%", "size2": "l", "show": true}
```

| Key | Meaning |
|---|---|
| `title` | Header, max 24 chars |
| `line1` ... `line8` | Text lines, only sent lines are drawn, centered vertically |
| `size1` ... `size8` | `s` (14 px, 26 chars), `m` (20 px, 18), `l` (30 px, 12, default), `xl` (40 px, 9) |
| `show` | `true` jumps to the card page, otherwise it only updates |

All keys are optional, missing keys are cleared. If the text is not valid JSON it is shown as one single line (max 12 chars).

Home Assistant:

```yaml
action: text.set_value
target:
  entity_id: text.brain_tvm1_card
data:
  value: >
    {"title": "Server", "line1": "CPU {{ states('sensor.cpu') }}%", "size1": "xl", "show": true}
```

## Entities (excerpt)

| Entity | Type | Purpose |
|---|---|---|
| `NM TV Touch` | binary sensor | Touch input (filtered) |
| `PushLight` | binary sensor | Active while the touch is held on the light page |
| `Card` | text | Card JSON, see above |
| `DisplayPage` | select | Jump to a page |
| `ClockLanguage` | select | English / German on the clock page |
| `RotatePages`, `RotateInterval` | text, number | Auto rotation |
| `Brightness` | number | Backlight |
| `BLEScanner` | switch | BLE scan on/off |
| `BLEDevices`, `BLEPeak` | sensor | BLE counter, `ResetBLEPeak` resets the peak |
| `WiFiSignal`, `HeapFree`, `NM TV Uptime`, ... | sensors | Diagnostics |
| `ResetReason`, `DeviceInfo` | text sensors | Boot info |

## Known issue: resets from sparks

A piezo lighter or gas stove igniter close to the device can reset the whole chip. This is an electrical issue (the touch wire acts as an antenna, or noise gets in through the power supply) and cannot be fixed in software. Things that can help but this is only an AI-Dream and not tested by myself:

- Short touch wire, with a 470 Ω to 1 kΩ series resistor near the pin
- 100 nF at EN and 3V3, plus a bulk capacitor (470 µF or more) at 5 V
- A decent power supply, a ferrite on the USB cable, more distance from the source

Note: the `ResetReason` text can show a stale message from the last OTA, so check the uptime to see if the device really restarted.

## Notes

- UI texts and entity names are English on purpose, only the clock page can be switched to German.
- Logging is set to `initial_level: NONE` to save resources. Change it with the `LogLevel` select when you need logs.
