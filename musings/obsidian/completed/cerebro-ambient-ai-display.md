# Cerebro — Ambient AI Display on ESP32-S3

**Status:** Completed  
**Updated:** 2026-05-01T20:00:00+05:30  
**Tags:** embedded, esp32, display, ai-assistant, hardware

## TL;DR

Cerebro is a concept for an ambient AI display built on the ESP32-S3 microcontroller — a small, always-on screen that shows contextual information like weather, calendar, reminders, and AI-generated summaries without requiring user interaction. It acts as an ambient information radiator rather than an interactive device, updating its display periodically based on configured data sources and an LLM backend.

## Concept

The idea is a small e-ink or LCD display, wall-mounted or desk-standing, that:
- Shows a rotating carousel of information cards
- Pulls data from APIs (weather, calendar, news headlines, stocks)
- Summarizes and contextualizes using an LLM
- Updates every few minutes
- Uses WiFi for connectivity
- Runs on ESP32-S3 for cost efficiency (~$5-10 BOM)

### Key Design Principles

1. **Ambient, not interactive** — No touch, no buttons. Information comes to you.
2. **Glanceable** — Designed for 2-3 second reads, not deep study
3. **Contextual** — Shows what's relevant based on time of day and configured interests
4. **Low power** — ESP32 deep sleep between updates, e-ink retains image without power
5. **Private** — Data processing can happen locally or through a self-hosted LLM endpoint

## Hardware

### Core Components
- **MCU:** ESP32-S3 (dual-core Xtensa LX7, WiFi/BLE, PSRAM)
- **Display options:**
  - Waveshare 4.2" e-ink (400×300, B/W or 3-color)
  - Waveshare 5.65" 7-color e-ink (600×448)
  - ILI9341 TFT LCD 2.8" as cheaper alternative
- **Power:** USB-C or LiPo battery with charging circuit
- **Enclosure:** 3D-printed frame, picture-frame style

### BOM Estimate (e-ink variant)
| Component | Cost |
|-----------|------|
| ESP32-S3 Dev Board | $4-6 |
| 4.2" e-ink display | $15-20 |
| 3D-printed case | $2-3 |
| USB-C cable + misc | $2 |
| **Total** | **$23-31** |

## Software Architecture

```
┌─────────────────────────────────────────┐
│              Cerebro Display             │
│  ┌───────────┐  ┌────────────────────┐  │
│  │ WiFi Mgr  │  │   Display Driver   │  │
│  │ (connect, │  │   (e-ink partial   │  │
│  │  reconnect)│  │    refresh, layout)│  │
│  └─────┬─────┘  └────────┬───────────┘  │
│        │                 │               │
│  ┌─────▼─────────────────▼───────────┐  │
│  │         Content Manager           │  │
│  │  (card generation, scheduling,    │  │
│  │   API polling, cache management)  │  │
│  └─────────────┬─────────────────────┘  │
│                │ HTTPS                   │
└────────────────┼────────────────────────┘
                 │
    ┌────────────▼────────────┐
    │    Backend / LLM API    │
    │  (OpenAI, Ollama, etc)  │
    └─────────────────────────┘
```

### Firmware (Arduino / ESP-IDF)

Key libraries:
- **GxEPD2** or **WaveShare E-Paper** for display
- **ArduinoJson** for API response parsing
- **WiFiClientSecure** for HTTPS API calls
- **NTPClient** for time sync
- **Preferences** for persistent config storage

### Card Types

1. **Weather Card** — Current conditions, forecast, temperature trend
2. **Calendar Card** — Upcoming events (today + tomorrow)
3. **Quote Card** — Daily quote or thought-provoking excerpt
4. **Summary Card** — AI-generated digest of configured topics
5. **News Card** — Top headline from configured categories
6. **Status Card** — System info (wifi, battery, last update)

### LLM Integration

The display calls a backend server (could be a Raspberry Pi, cloud function, or direct API) that:
1. Fetches raw data from configured sources
2. Sends to LLM with a prompt template for summarization
3. Returns a formatted card (title + body, constrained to display dimensions)

Example prompt template:
```
You are a display card generator. Summarize the following 
information into a card with a title (max 40 chars) and 
body text (max 300 chars) suitable for a 400x300 display.

Data: {weather_data}

Format:
TITLE: <title>
BODY: <body>
```

## Implementation Status

### Completed
- Hardware selection and BOM finalized
- Display driver code for Waveshare 4.2" e-ink working
- WiFi connection manager with reconnection logic
- NTP time sync
- Basic card rendering (text only, 2 fonts)

### In Progress
- Multi-card carousel with configurable rotation timing
- Weather API integration (OpenWeatherMap)
- LLM backend integration (Ollama local + OpenAI fallback)

### Planned
- Calendar integration (Google Calendar / CalDAV)
- 3D-printable enclosure design
- Web-based configuration portal (captive portal on first boot)
- OTA firmware updates
- BLE-based phone companion for configuration

## Development Setup

```bash
# Clone the firmware repo
git clone https://github.com/bagdeabhishek/cerebro-firmware
cd cerebro-firmware

# PlatformIO build
pio run -e esp32-s3-devkitc-1

# Flash and monitor
pio run -e esp32-s3-devkitc-1 -t upload -t monitor
```

## Configuration

Configuration is stored in SPIFFS as `config.json`:

```json
{
  "wifi": {
    "ssid": "",
    "password": ""
  },
  "backend": {
    "url": "http://192.168.1.100:8000",
    "api_key": ""
  },
  "cards": ["weather", "calendar", "quote", "summary"],
  "rotation_interval_seconds": 30,
  "timezone": "Asia/Kolkata",
  "location": {
    "lat": 19.0760,
    "lon": 72.8777
  }
}
```

## Alternatives Considered

| Alternative | Pros | Cons |
|------------|------|------|
| Raspberry Pi Zero W | Full Linux, easier dev | Higher cost ($15+), higher power |
| ESP32-C3 | Cheaper | No PSRAM, less capable |
| Inkplate | All-in-one e-ink ESP32 | Expensive ($99+) |
| Magic Mirror (RPi) | Established ecosystem | Larger, needs monitor, higher power |

Cerebro targets the sweet spot: cheap ESP32-S3 hardware with just enough capability for an ambient display, offloading heavy processing to a backend.

## References

- [ESP32-S3 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf)
- [Waveshare 4.2inch e-Paper Module](https://www.waveshare.com/wiki/4.2inch_e-Paper_Module)
- [GxEPD2 Library](https://github.com/ZinggJM/GxEPD2)
- [PlatformIO ESP32-S3](https://docs.platformio.org/en/latest/boards/espressif32/esp32-s3-devkitc-1.html)
