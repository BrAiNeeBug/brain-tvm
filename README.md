# brain-tvm
<img width="920" height="945" alt="image" src="https://github.com/user-attachments/assets/46394f3d-b300-46db-aedd-f6e7f2244ef0" />


ESPHome firmware for a small ESP32 display device (**NM-TV**): a 240x240 ST7789 screen with one touch button, showing a clock, system status, a BLE device counter with tracker detection, a free-form info card and a light page. Everything is controlled from Home Assistant or the built-in web UI.

One YAML file, no custom C++ components.

## Features

- **6 display pages** in neon green: Welcome, System status, Card, Clock, BLE scanner, Light
- **Touch button**: short tap = next page, hold = action of the current page (`PushLight` on the light page, dimmer toggle on the clock page)
- **Spike filter** for the touch input (`touch_delay`), so a piezo lighter or similar noise does not switch pages
- **Auto page rotation**: choose pages and interval at runtime
- **Clock page** with English/German switch (`ClockLanguage`), NTP time, hold to toggle the dimmer between 30 % and 100 %
- **Card page**: send a JSON to the `Card` text entity and it shows title and up to 8 lines in 4 sizes
- **BLE scanner** counting nearby devices (current and peak), can be switched on and off
- **Tracker detection**: spots AirTag / Find My, Samsung SmartTag, Tile, Chipolo and Google Find My Device trackers and raises an alert when one stays around
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

3. Adjust the `substitutions` at the top, see [Substitutions](#substitutions).
4. Flash it. The first flash must be over USB, later updates via OTA.
5. Add the device in Home Assistant (ESPHome integration).

## Substitutions

| Name | Default | Meaning |
|---|---|---|
| `device_name`, `friendly_name`, `localdomain` | | Device naming and local domain |
| `touch_delay` | `80ms` | Touch must be active this long to count |
| `long_press_ms` | `600` | Hold time in ms |
| `dim_low`, `dim_high` | `30`, `100` | Backlight levels (%) toggled by holding on the clock page |
| `ble_interval`, `ble_window` | `160ms`, `120ms` | BLE scan timing, the window must be smaller than the interval so WiFi keeps some airtime |
| `ble_timeout_s` | `45` | A device counts as present this many seconds after its last advertisement |
| `tracker_timeout_s` | `60` | A tracker is forgotten after this many seconds without advertisement |
| `tracker_alert_min` | `5` | Alert when a tracker was present this many minutes |

## Pages

| # | Page | Content |
|---|---|---|
| 1 | Welcome | "WELCOME", "TOUCH ME" and a hint to tap for the next page |
| 2 | Status | Host, IP, SSID, signal, MAC, uptime, free heap, ESPHome version, build date, BLE count |
| 3 | Card | Content you send to the `Card` entity (only shown once something was sent) |
| 4 | Clock | Weekday, time and date |
| 5 | BLE | Number of BLE devices around, peak and tracker count (frame turns orange on alert) |
| 6 | Light | Static light page, `PushLight` is active while you hold the touch |

Pages can be changed with the `DisplayPage` select. `RotatePages` takes a list like `3,4` or `1-4` or `1,3-6`, `RotateInterval` sets the time.

## Touch

- **Short tap** (on release): next page
- **Hold** (`long_press_ms`, default 600 ms), depends on the page:
  - Clock page: toggles the backlight between `dim_low` and `dim_high`. It goes through the `Brightness` entity, so the slider follows and the value is stored.
  - Light page: `PushLight` goes active until you release, use it as a trigger in Home Assistant.
  - Other pages: nothing, the hold does not change the page.
- `touch_delay` (default 80 ms): the touch must be active at least this long. Raise it if short noise spikes still switch pages.

## BLE scanner

The counter shows every address that advertised within the last `ble_timeout_s` seconds, plus the peak since boot (`ResetBLEPeak` resets it).

The count always moves a bit, because many phones and wearables rotate their random MAC address. A longer `ble_timeout_s` calms the number, a shorter one reacts faster.

## Tracker detection

Trackers are recognized by their advertisement signature, passive scanning is enough:

| Type | Signature |
|---|---|
| Apple Find My (AirTag and other Find My accessories) | Manufacturer data 0x004C, type 0x12, length 0x19 |
| Samsung SmartTag | Service data 0xFD5A |
| Tile | Service data / UUID 0xFEED or 0xFD84 |
| Chipolo | Service data 0xFE65 |
| Google Find My Device network | Service data 0xFEAA with frame type 0x40 or 0x41 |

Every tracker is followed per address. If it keeps advertising without a gap longer than `tracker_timeout_s`, its presence time grows. Once a tracker has been around for `tracker_alert_min` minutes, the alert is raised, the BLE page shows `! TRACKER: n !` with an orange frame, and `TrackerAlert` turns on in Home Assistant.

Limits:

- An AirTag in separated mode rotates its address about every 15 minutes, so keep `tracker_alert_min` well below that.
- Other Find My accessories (for example separated AirPods) look the same as an AirTag and can cause false alerts.
- The device is stationary, so this tells you "a tracker is parked near me", not "a tracker follows me".
- The UUIDs for Tile and Chipolo may need adjusting if a tracker is not detected.

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
| `Brightness` | number | Backlight, also toggled by holding on the clock page |
| `BLEScanner` | switch | BLE scan on/off |
| `BLEDevices`, `BLEPeak` | sensor | BLE counter, `ResetBLEPeak` resets the peak |
| `TrackerAlert` | binary sensor | On when a tracker was present longer than `tracker_alert_min` |
| `TrackerCount` | sensor | Trackers currently in range |
| `TrackerInfo` | text sensor | Type and presence time per tracker, e.g. `FindMy 6:12, Tile 0:40` |
| `WiFiSignal`, `HeapFree`, `NM TV Uptime`, ... | sensors | Diagnostics |
| `ResetReason`, `DeviceInfo` | text sensors | Boot info |

## Known issue: resets from sparks

A piezo lighter or gas stove igniter close to the device can reset the whole chip. This is an electrical issue (the touch wire acts as an antenna, or noise gets in through the power supply) and cannot be fixed in software. Things that help:

- Short touch wire, with a 470 Ω to 1 kΩ series resistor near the pin
- 100 nF at EN and 3V3, plus a bulk capacitor (470 µF or more) at 5 V
- A decent power supply, a ferrite on the USB cable, more distance from the source

Note: the `ResetReason` text can show a stale message from the last OTA, so check the uptime to see if the device really restarted.

## Notes

- UI texts and entity names are English on purpose, only the clock page can be switched to German.
- Logging is set to `initial_level: NONE` to save resources. Change it with the `LogLevel` select when you need logs.
