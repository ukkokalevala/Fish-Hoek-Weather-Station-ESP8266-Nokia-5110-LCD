Fish Hoek Weather Station — ESP8266 + Nokia 5110 LCD
A compact, self-contained desk weather station built on a Wemos D1 mini (ESP8266) driving a Nokia 5110 (PCD8544) 84×48 monochrome LCD. It displays live local time (synced via NTP) and current weather conditions (temperature + graphical icon + text description) fetched from the free Open-Meteo API. No API key required.

Features
Live clock — synced over UDP NTP to za.pool.ntp.org, displayed in South African Standard Time (UTC+2, no DST)

Current temperature — fetched every 10 minutes from Open-Meteo

Graphical weather icon — 8×8 bitmap mapped from WMO weather codes (sun, cloud, rain, snow, unknown)

Text description — "Clear", "Partly cloudy", "Rain", etc.

Debug footer — raw WMO code displayed for verification

Resilient — auto-retries weather fetch, hourly NTP resync, survives brief network outages

Hardware
Component	Detail
MCU	Wemos D1 mini (ESP8266)
Display	Nokia 5110 / PCD8544, 84×48 px, monochrome
Connection	Hardware SPI
Power	5 V USB (1 A supply recommended)
Wiring (hardware SPI):

Nokia 5110	Wemos D1	GPIO
RST	D2	4
CE (CS)	D1	5
DC	D6	12
DIN (MOSI)	D7	13
CLK	D5	14
VCC	3V3	—
GND	GND	—
Software Design
The sketch was deliberately built lean to run comfortably on the ESP8266's limited heap (~50 KB free at runtime):

No TLS — plain HTTP/1.0 to api.open-meteo.com:80 avoids the ~40 KB heap cost of WiFiClientSecure

No ArduinoJson — weather JSON is parsed by manual substring scanning, anchored at "current":{ to skip the current_units block

No NTPClient library — raw UDP NTP packet (~48 bytes) sent and decoded directly

Icons in PROGMEM — bitmaps live in flash, not RAM

Serial at 9600 baud — matches the board's stable baud rate (clone firmware freezes at 115200)

Data Flow
text
Boot → WiFi connect (30 s timeout)
     → NTP sync (UDP 123, za.pool.ntp.org)
     → Weather fetch (HTTP/1.0, Open-Meteo)
     → Render frame

Loop every 1 s:
     → Update clock
     → Refresh weather if 10 min elapsed
     → Redraw LCD

Every 1 h:
     → NTP resync
Configuration
Edit only these lines at the top of the sketch:

const char* SSID = "your_wifi";
const char* PASS = "your_password";
const char* LAT  = "-34.136002";       // Fish Hoek, Cape Town
const char* LON  = "18.430000";
const long  TZ_OFFSET = 2 * 3600;      // SAST (UTC+2)
Display Layout
text
┌──────────────────────────────┐
│  18:47              ☁        │  ← time (size 2) + 8×8 icon
│                              │
│  14.2C                       │  ← current temperature
│  Partly cloudy               │  ← WMO description
│  code:2                      │  ← raw weather code (debug)
└──────────────────────────────┘
Engineering Notes
Three non-obvious bugs were discovered and fixed during development — useful lessons for any ESP8266 project:

WiFi.setSleepMode(WIFI_NONE_SLEEP) before WiFi.begin() can prevent association on some clone firmwares. Call it after connection succeeds, or omit it entirely.

Open-Meteo returns "current_units" before "current" — a naive indexOf("temperature_2m") matches the unit string "°C" first, yielding 0.0. Always anchor the search after "current":{.

HTTP response must be read until the server closes (HTTP/1.0 + Connection: close), not until N bytes past the headers — otherwise the JSON body is truncated mid-stream.

Source
Single .ino file, ~280 lines, no external libraries beyond:

Adafruit GFX Library

Adafruit PCD8544 Nokia 5110 LCD

ESP8266WiFi (built-in)

WiFiUdp (built-in)

SPI (built-in)
