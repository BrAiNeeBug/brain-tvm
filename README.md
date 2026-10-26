# brain-tvm
<img width="720" height="622" alt="image" src="https://github.com/user-attachments/assets/bf93e311-9ff8-41b3-bc3e-bd3a893a51cb" />

ESPHome firmware for a small ESP32 display device: a 240x240 ST7789 screen with one touch button, showing a colorful clock, system status, a BLE device counter with tracker detection, a free-form info card and a light page. Everything is controlled from Home Assistant or the built-in web UI.

One YAML file, no custom C++ components.

## Features

- **6 display pages** (neon green theme, the clock page is in color): Welcome, System status, Card, Clock, BLE scanner, Light
- **Colorful clock**: rainbow seconds ring, gradient digits with a fading colon, weekday in its own color and a day progress bar. Digits and bar share a palette (green/blue/purple by day, dark red by night) that you can shift with the `ClockBarHue` slider
- **Touch button**: short tap = next page, hold = action of the current page (`PushLight` on the light page, dimmer toggle on the clock page)
- **Spike filter** for the touch input (`touch_delay`), plus an adjustable trigger level (`touch_threshold`) for each device
- **Auto page rotation**: choose pages and interval at runtime
- **Clock language** switch English/German (`ClockLanguage`), NTP time
- **Card page**: send a JSON to the `Card` text entity and it shows title and up to 8 lines in 4 sizes
- **BLE scanner** counting nearby devices (current and peak), can be switched on and off
- **Tracker detection**: spots AirTag / Find My, Samsung SmartTag, Tile, Chipolo and Google Find My Device trackers and raises an alert when one stays around
- **Web UI** (v3) with groups, plus OTA, safe mode and encrypted API
- **Safe mode restart with notice**: the display shows a BOOTMODE hint before the device reboots into safe mode
- Brightness, rotate interval, clock colors and more as runtime entities, no reflash needed

## Hardware

| Part | Pin |
|---|---|
| ESP32 (`esp32dev`, chip revision 3.1+) | |
| ST7789 240x240 display, SPI CLK / MOSI | GPIO14 / GPIO13 |
| Display CS / DC | GPIO15 / GPIO2 |
| Display power (inverted) | GPIO21 |
| Backlight (PWM, inverted) | GPIO19 |
| Touch pad | GPIO32 |

The display runs at 40 MHz SPI (`data_rate`), so a frame is pushed quickly and the main loop stays free for touch and BLE. If the picture glitches (flicker, pixel noise), lower it to `20MHz` in the `display` block.

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

All values are plain text that gets pasted into the config, so they are all quoted. Options that expect a time need the unit (`"50ms"`). Values used inside C++ lambdas are bare numbers, the unit is in the name (`_ms`, `_s`, `_min`).

| Name | Default | Meaning |
|---|---|---|
| `device_name`, `friendly_name`, `localdomain` | | Device naming and local domain |
| `touch_delay` | `50ms` | Touch must be active this long to count |
| `touch_threshold` | `1450` | Touch trigger level, the pad counts as touched when its raw value drops below this |
| `touch_setup_mode` | `false` | `true` logs the raw touch values, see [Touch](#touch) |
| `long_press_ms` | `600` | Hold time in ms |
| `dim_low` | `5` | Backlight level (%) the clock page dimmer goes down to |
| `ble_interval`, `ble_window` | `160ms`, `120ms` | BLE scan timing, the window must be smaller than the interval so WiFi keeps some airtime |
| `ble_timeout_s` | `45` | A device counts as present this many seconds after its last advertisement |
| `tracker_timeout_s` | `60` | A tracker is forgotten after this many seconds without advertisement |
| `tracker_alert_min` | `5` | Alert when a tracker was present this many minutes |

## Pages

| # | Page | Content |
|---|---|---|
| 1 | Welcome | "WELCOME", "HA DASHBOARD", "TOUCH ME" and a hint to tap for the next page |
| 2 | Status | Host, IP, SSID, signal, MAC, uptime, free heap, ESPHome version, build date, BLE count |
| 3 | Card | Content you send to the `Card` entity (only shown once something was sent) |
| 4 | Clock | Seconds ring, weekday, time, day progress bar and date |
| 5 | BLE | Number of BLE devices around, peak and tracker count (frame turns orange on alert) |
| 6 | Light | Static light page, `PushLight` is active while you hold the touch |

Pages can be changed with the `DisplayPage` select. `RotatePages` takes a list like `3,4` or `1-4` or `1,3-6`, `RotateInterval` sets the time.

The display refreshes every 250 ms on the clock page (smooth ring and colon fade) and every 500 ms on all other pages.

On the status page, characters the font cannot draw (for example emoji in an SSID) are shown as `_`, so they do not flood the log with missing-glyph warnings. The real values stay unchanged in the web UI and Home Assistant.

## Clock page

- **Seconds ring**: 60 rainbow dots around the edge, 0 s is at the top center and it runs clockwise. The current second is a white dot with a fading tail of 15 dots.
- **Time (HH:MM, no seconds)**: the digits have a vertical color gradient. The colon fades between bright and dark in a 2 second cycle.
- **Weekday**: its own color per day (Monday red, Tuesday orange, Wednesday yellow, Thursday green, Friday cyan, Saturday purple, Sunday pink), in English or German.
- **Day progress bar**: 0-24 h, filled up to the current time, with a white marker.
- **Date**: `dd.mm.yyyy` in German, `Mon dd, yyyy` in English.

**Colors**: digits and bar use the same palette. By day it runs from green over blue to purple. At night it is a dimmed dark red. The change fades in from 20 h to 22 h and out from 5 h to 7 h, so it is easy on the eyes at the bedside. The `ClockBarHue` slider (0-360°) shifts the hue of both palettes, day and night.

**Dimmer**: hold on the clock page to dim the backlight down to `dim_low` (default 5 %). Hold again to go back to the brightness from before. That previous brightness is stored, so it survives a reboot. It goes through the `Brightness` entity, so the slider follows. Without a stored value it returns to 50 %.

## Touch

- **Short tap** (on release): next page
- **Hold** (`long_press_ms`, default 600 ms), depends on the page:
  - Clock page: dimmer toggle, see above.
  - Light page: `PushLight` goes active until you release, use it as a trigger in Home Assistant.
  - Other pages: nothing, the hold does not change the page.
- `touch_delay` (default 50 ms): the touch must be active at least this long. If short noise spikes switch pages, raise it, for example to 60 ms.

**Calibrating `touch_threshold`**: the raw touch value differs per device, wiring and case. The pad counts as touched when the value drops below the threshold, so it belongs between the idle and the touched value.

1. Set `touch_setup_mode: "true"` and flash.
2. Set the `LogLevel` select to `DEBUG`, the logger starts at `NONE`.
3. Note the value with nothing touching and the value with a finger on the pad.
4. Set `touch_threshold` roughly in the middle, a bit closer to the idle value so it triggers safely.
5. Set `touch_setup_mode` back to `"false"` and lower the log level, the values log several times per second otherwise.

Example from one device: idle about 1620, touched about 1295, threshold 1450.

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

## Entities

Home Assistant sorts them by entity category:

| Section in Home Assistant | Entities |
|---|---|
| **Controls** (everything for the display) | `DisplayPage`, `Card`, `Brightness`, `ClockLanguage`, `ClockBarHue`, `RotateInterval`, `RotatePages` |
| **Configuration** (other things to click) | `BLEScanner` (switch), `ResetBLEPeak` (button) |
| **Sensors** (display only) | `BLEDevices`, `BLEPeak`, `TrackerCount`, `TrackerInfo`, `TrackerAlert`, `Touch`, `PushLight` |
| **Diagnostic** | `IP`, `Uptime`, `DeviceInfo`, `ResetReason`, WiFi info (see below), `Restart`, `SafeModeRestart`, `LogLevel` |

Notes on single entities:

| Entity | Purpose |
|---|---|
| `Touch` | Touch input (filtered) |
| `PushLight` | Active while the touch is held on the light page |
| `Brightness` | Backlight, default 50 % on a fresh install, also toggled by holding on the clock page |
| `ClockBarHue` | Shifts the hue of the clock bar and digits, 0 = default colors |
| `TrackerInfo` | Type and presence time per tracker, e.g. `FindMy 6:12, Tile 0:40` |
| `SafeModeRestart` | Shows the BOOTMODE notice on the display, then reboots into safe mode |
| `ResetReason`, `DeviceInfo` | Boot info |

**WiFi info** (also in the web UI group "Network"): `IP`, `WiFiSSID`, `WiFiBSSID`, `WiFiMAC`, `WiFiDNS`, `WiFiSignal` (dBm), `WiFiQuality` (% derived from the signal, -100 dBm = 0 %, -50 dBm = 100 %) and `WiFiChannel`. There is no SNR or noise value, the ESP32 only reports the signal strength.

**Web UI only**: `HeapFree`, `HeapMaxBlock`, `LoopTime`, `CPUFreq` and `SystemTime` are marked `internal` and not sent to Home Assistant. `include_internal: true` in `web_server` still shows them in the web UI (group "System"). The same setting shows the raw safe mode button `SafeModeRestartRaw_NoNotice` there. It reboots into safe mode without the display hint, press `SafeModeRestart` instead.

**Web UI groups**: Display, Bluetooth, Network, System.

## Known issue: resets from sparks

A piezo lighter or gas stove igniter close to the device can reset the whole chip. This is an electrical issue (the touch wire acts as an antenna, or noise gets in through the power supply) and cannot be fixed in software. Things that help:

- Short touch wire, with a 470 Ω to 1 kΩ series resistor near the pin
- 100 nF at EN and 3V3, plus a bulk capacitor (470 µF or more) at 5 V
- A decent power supply, a ferrite on the USB cable, more distance from the source

To see why it restarted, read the `ResetReason` text after the next boot (brownout, watchdog, panic ...). Note that it can show a stale message from the last OTA, so check the uptime to see if the device really restarted.

## Notes

- UI texts and entity names are English on purpose, only the clock page can be switched to German.
- Logging is set to `initial_level: NONE` to save resources. Change it with the `LogLevel` select when you need logs.
- Renaming an entity changes its Home Assistant ID, the old one then shows as unavailable and can be deleted.
