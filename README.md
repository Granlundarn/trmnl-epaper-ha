# TRMNL E-Paper Calendar Display — ESPHome + Home Assistant

A battery-powered 7.5" e-paper calendar display using ESPHome and Home Assistant. Wakes twice daily (06:00 and 18:00), fetches today's and tomorrow's calendar events, renders them via LVGL, then returns to deep sleep.

![Display showing calendar events, date, week number and battery level](images/display.jpg)

---

## Hardware

| Component | Details |
|---|---|
| Device | [TRMNL](https://usetrmnl.com/) or compatible |
| Display | Waveshare 7.5" e-paper V2 (800×480, `7.50inv2`) |
| MCU | ESP32-S3 with octal PSRAM |
| Battery | LiPo via onboard charging circuit |

The GPIO pinout matches the TRMNL hardware:

| Function | GPIO |
|---|---|
| SPI CLK | GPIO7 |
| SPI MOSI | GPIO9 |
| Display CS | GPIO44 |
| Display DC | GPIO10 |
| Display RST | GPIO38 |
| Display BUSY | GPIO4 (active low) |
| Battery enable | GPIO6 |
| Battery ADC | GPIO1 |

---

## Features

- **E-paper display** rendered natively via LVGL (no screenshot/image approach)
- **Deep sleep** between updates — very low power consumption
- **Wakes at 06:00 and 18:00** daily (configurable)
- **Today & tomorrow** calendar sections with events from Home Assistant
- **Date, day name and week number** formatted directly on device
- **Battery percentage** shown in footer
- **OTA updates** supported (see flashing notes below)

---

## Requirements

- [ESPHome](https://esphome.io/) (tested with 2026.2.x)
- [Home Assistant](https://www.home-assistant.io/) with:
  - A calendar entity (e.g. `calendar.family`)
  - Two `input_text` helpers
  - One automation

---

## Quick Start

### 1. Home Assistant — Create Helpers

In HA: **Settings → Helpers → + Create helper → Text**

| Name | Entity ID | Max length |
|---|---|---|
| `trmnl_today_event` | `input_text.trmnl_today_event` | 255 |
| `trmnl_tomorrow_event` | `input_text.trmnl_tomorrow_event` | 255 |

### 2. Home Assistant — Create Automation

Copy [`homeassistant/automation.yaml`](homeassistant/automation.yaml) into your automation editor.

> **Important:** Change `calendar.family` to your calendar entity ID.

Run the automation manually once to populate the helpers before first flash.

### 3. ESPHome — Configure

Copy [`esphome/trml.yaml`](esphome/trml.yaml) into your ESPHome config folder.

Copy [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) to `secrets.yaml` and fill in your values.

**Key settings to change** at the top of `trml.yaml`:

```yaml
substitutions:
  device_name: trmnl
  friendly_name: TRMNL Display
  timezone: "America/New_York"   # ← your timezone
```

For language/locale, find the `LOCALIZATION` comment in the YAML and update the day/month name arrays and label prefixes.

### 4. Flash

> ⚠️ **Read the flashing section below before proceeding.**

---

## Flashing — Important Notes

The device goes to deep sleep ~60–90 seconds after boot. ESPHome compilation takes 5–10 minutes, so **OTA will always fail** on a sleeping device.

### First flash / after deep sleep

You must put the device into bootloader mode manually:

1. Hold the **Boot** button
2. Press and release **Reset** (keep holding Boot)
3. Release **Boot**
4. **Turn off** the battery switch and unplug the USB cable
5. **Plug in** the USB cable again
6. Flash via ESPHome (USB port)

Steps 4–5 are required on the TRMNL/XIAO ESP32-S3 — the device must re-enumerate as a serial port in bootloader mode.

### Subsequent OTA updates

If you need to OTA flash, keep the device awake by temporarily increasing `run_duration` in the YAML:

```yaml
deep_sleep:
  run_duration: 600s   # 10 minutes — enough for OTA
```

Flash, then revert to `300s`.

---

## Customization

### Timezone

Set your [POSIX timezone string](https://github.com/nayarsystems/posix_tz_db/blob/master/zones.csv) in `substitutions`:

```yaml
substitutions:
  timezone: "Europe/Stockholm"   # Sweden
  # timezone: "America/New_York"
  # timezone: "Australia/Sydney"
```

### Language / Locale

Find the `# LOCALIZATION` block in `trml.yaml` and update the arrays and format strings:

```cpp
// Day names starting from Sunday (index 0)
static const char* DAYS[] = {
  "Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"
};
// Swedish: {"Söndag","Måndag","Tisdag","Onsdag","Torsdag","Fredag","Lördag"}

// Month names (index 0 = January)
static const char* MONTHS[] = {
  "Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"
};
// Swedish: {"jan","feb","mar","apr","maj","jun","jul","aug","sep","okt","nov","dec"}

// Header format strings
// "Today: %s %d %s w.%s"    (English)
// "Idag: %s %d %s v.%s"     (Swedish)
```

### Wake times

Change the two wake times in the sleep calculation lambda:

```cpp
if      (min_now <  6 * 60) next_wake =  6 * 60;   // 06:00
else if (min_now < 18 * 60) next_wake = 18 * 60;   // 18:00
else                        next_wake = 24 * 60 + 6 * 60;
```

### Calendar entity

Change `calendar.family` in the HA automation to your calendar entity.

### Display model

If you use a different Waveshare model, change `model: 7.50inv2` in the display section.

---

## How It Works

```
Boot
 ├─ Priority 600: Enable battery circuit → measure voltage
 ├─ Priority 0:   Disable LVGL scrollbar on screen object
 └─ Priority -100:
      ├─ Prevent deep sleep
      ├─ Wait: WiFi connected     (max 30s)
      ├─ Wait: HA API connected   (max 30s)
      ├─ Wait: Calendar events received from HA  (max 30s)
      ├─ Format date/time labels via ESPTime
      ├─ Copy calendar events to LVGL labels
      ├─ delay 500ms  (LVGL render buffer settle)
      ├─ Refresh e-paper display
      ├─ Calculate sleep duration to next wake time
      └─ Allow deep sleep → sleep until 06:00 or 18:00
```

**Time sources:** `platform: homeassistant` (primary) + SNTP (fallback). Both are valid immediately after API connection in normal conditions.

**E-paper color note:** LVGL `0x000000` renders as *white* and `0xFFFFFF` as *black* on this display — all colors are inverted compared to standard LVGL usage.

---

## File Structure

```
trmnl-epaper-ha/
├── README.md
├── esphome/
│   ├── trml.yaml               # Main ESPHome configuration
│   └── secrets.yaml.example    # Secrets template
└── homeassistant/
    └── automation.yaml         # HA automation for calendar events
```

---

## License

MIT
