RFC Environmental Dashboard — ESP32 Firmware

Production/commissioning firmware for the RFC Environmental Dashboard built around an ESP32. The firmware acquires environmental, air-quality, light/UV, water-quality, rain and sound data, serves a local dashboard/API from LittleFS, maintains RTC time using NTP + DS3231, and publishes the latest snapshot to Firebase Realtime Database.

Security: Never commit Wi-Fi passwords, Firebase device passwords, private keys, or other credentials to Git. The source supplied for this README contained credentials; those values are intentionally omitted here. Rotate any credentials that were previously exposed and move secrets to a local/private configuration file.

1. Features

ESP32-based environmental monitoring

I²C sensor support:

SHT31

BME680

BMP280

BH1750

VEML6075

DS3231 RTC

UART sensors:

PMS7003 particulate matter sensor

MH-Z19C CO₂ sensor

Analog sensors:

TDS

pH

Rain

MAX9814 sound level

AQI calculation from PM2.5 and PM10

TDS temperature compensation

Configurable pH calibration

Rain wetness percentage calculation

RTC synchronization from NTP

Local web dashboard stored in LittleFS

Local JSON APIs

Firebase Realtime Database publishing using REST API

Firebase Email/Password authentication

Persistent Firebase refresh token using ESP32 Preferences

Sensor health diagnostics

Startup self-test

Forced refresh API

Browser dashboard caching/fallback for stale or temporarily unavailable Firebase data

The supplied firmware is configured for commissioning with a 1-minute dashboard publish interval and 5-second physical sensor acquisition interval. The source comments specify changing the dashboard interval to 60 minutes only after commissioning is complete.

2. System Architecture

                    ┌─────────────────────┐
                    │       ESP32         │
                    │                     │
                    │ Sensor acquisition  │
                    │ Validation          │
                    │ AQI calculation      │
                    │ RTC / NTP            │
                    │ Web server           │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        Local Dashboard    Local REST API    Firebase
          LittleFS          /api/*           Realtime DB
              │                                 │
              │                                 ▼
              └──────────────────────────► Web Dashboard

The browser dashboard subscribes to:

devices/rfc-environment-001/latest

and is designed to retain the previous valid snapshot when Firebase temporarily disconnects or returns invalid/stale data.

3. Hardware / Pin Map

I²C

Device

Address

ESP32

SHT31

0x44

SDA GPIO21 / SCL GPIO22

BME680

0x77

SDA GPIO21 / SCL GPIO22

BMP280

0x76

SDA GPIO21 / SCL GPIO22

BH1750

0x23

SDA GPIO21 / SCL GPIO22

VEML6075

0x10

SDA GPIO21 / SCL GPIO22

DS3231

0x68

SDA GPIO21 / SCL GPIO22

The firmware scans the I²C bus during startup and detects the BME680/BMP280 presence before initialization.

UART

Sensor

ESP32 RX

ESP32 TX

Baud

PMS7003

GPIO16

GPIO17

9600

MH-Z19C

GPIO27

GPIO26

9600

UART connections must be crossed:

Sensor TX → ESP32 RX
Sensor RX → ESP32 TX
Sensor GND → ESP32 GND

Analog

Sensor

ESP32 ADC

TDS

GPIO34

pH

GPIO35

Rain

GPIO33

MAX9814

GPIO32

ESP32 GPIOs are not 5 V tolerant. Confirm that every analog output presented to an ESP32 ADC remains within the allowed ESP32 input range.

4. Required Libraries

Install these Arduino libraries:

Adafruit Sensor

Adafruit SHT31

Adafruit BME680

Adafruit BMP280

Adafruit VEML6075

BH1750

RTClib

The following are provided by the ESP32 Arduino core / standard environment:

Arduino

WiFi

WebServer

Wire

LittleFS

Preferences

HTTPClient

WiFiClientSecure

math

time

5. Recommended Repository Structure

rfc-environmental-dashboard/
│
├── firmware/
│   └── esp32/
│       ├── src/
│       │   └── main.cpp
│       │
│       ├── data/
│       │   ├── index.html
│       │   └── rfc1.jpeg
│       │
│       ├── platformio.ini
│       └── README.md
│
├── firebase/
│   └── database.rules.json
│
├── docs/
│   ├── wiring/
│   ├── calibration/
│   └── commissioning/
│
├── .gitignore
└── README.md

If Arduino IDE is used instead of PlatformIO, keep the firmware source and LittleFS data organized separately and upload the LittleFS filesystem after compiling the firmware.

6. Configuration

Before flashing, configure these items locally:

Wi-Fi

Do not hard-code production credentials in Git.

Recommended:

const char* WIFI_SSID = "YOUR_WIFI_SSID";
const char* WIFI_PASSWORD = "YOUR_WIFI_PASSWORD";

Firebase

Configure:

constexpr bool FIREBASE_ENABLED = true;

const char* FIREBASE_API_KEY =
  "YOUR_FIREBASE_WEB_API_KEY";

const char* FIREBASE_DATABASE_URL =
  "YOUR_FIREBASE_DATABASE_URL";

const char* FIREBASE_DEVICE_EMAIL =
  "YOUR_DEVICE_EMAIL";

const char* FIREBASE_DEVICE_PASSWORD =
  "YOUR_DEVICE_PASSWORD";

const char* FIREBASE_DEVICE_ID =
  "rfc-environment-001";

Firebase Email/Password authentication must be enabled and the device account must exist.

The database rules should allow the intended authenticated ESP32 account to write the device path while allowing only the intended browser read access.

7. Sensor Calibration

TDS

The firmware contains:

constexpr float TDS_CALIBRATION_FACTOR = 1.00f;

This is a starting value and must be calibrated against a known reference solution and the actual TDS module.

TDS calculation also uses water temperature compensation.

pH

The current firmware uses:

constexpr float PH_VOLTAGE_AT_PH7 = 2.50f;
constexpr float PH_SLOPE_V_PER_PH = -0.18f;

These are calibration parameters, not universal constants.

Calibrate the actual pH module using appropriate standard buffer solutions and replace these values with the measured calibration values.

For stable operation, retain the last valid pH reading when a single ADC acquisition is invalid rather than replacing a valid reading with NaN/—.

Rain

The firmware uses:

constexpr int RAIN_ADC_DRY = 4095;
constexpr int RAIN_ADC_WET = 1200;

These must be measured and calibrated for the actual rain sensor/module.

Sound

The MAX9814 value is a relative ADC peak-to-peak measurement. It is not a calibrated SPL/dB measurement unless the hardware and firmware are separately calibrated against a reference microphone/SPL meter.

8. Water Hardness

The current hardware/software does not directly measure calcium, magnesium, or total hardness.

TDS and pH must not be treated as a scientifically valid direct measurement of total hardness.

If an estimated hardness value is required for the dashboard, clearly label it:

Estimated Hardness

and document the estimation model and calibration against laboratory hardness measurements.

Do not present a TDS-derived estimate as measured Total Hardness.

For a production analytical value, use a valid hardness measurement method or a suitable calcium/magnesium measurement system.

9. Sensor Acquisition

The commissioning configuration is:

constexpr uint32_t DASHBOARD_UPDATE_INTERVAL_MINUTES = 1UL;
constexpr uint32_t SENSOR_ACQUISITION_INTERVAL_SECONDS = 5UL;

Meaning:

Sensors are physically sampled every 5 seconds.

The dashboard/Firebase snapshot is published approximately every 60 seconds.

After complete commissioning, the source comments specify changing the dashboard interval to:

constexpr uint32_t DASHBOARD_UPDATE_INTERVAL_MINUTES = 60UL;

Do this only after verifying the entire system.

10. RTC / Time

The firmware uses:

NTP server → ESP32 time → DS3231

NTP server:

pool.ntp.org

Timezone:

UTC+05:30

The DS3231 is used as the persistent local clock.

At startup:

Initialize DS3231.

Connect to Wi-Fi.

Synchronize from NTP when available.

Write the synchronized time to DS3231.

Continue using the RTC.

Periodically resynchronize from NTP.

11. Local Web Server

The ESP32 serves the dashboard from LittleFS.

Dashboard

GET /

Returns:

/index.html

Sensor data

GET /api/sensors

Returns the current dashboard JSON.

Compatibility alias

GET /api/data

Uses the same sensor-data handler as /api/sensors.

Health

GET /api/health

Returns health/diagnostic information including sensor status, Wi-Fi state, memory and ADC diagnostics.

Forced refresh

POST /api/refresh

Immediately performs sensor acquisition and publishes a new snapshot.

RFC image

GET /rfc1.jpeg

The image must exist in LittleFS:

/data/rfc1.jpeg

12. LittleFS

The firmware expects:

/index.html

and optionally:

/rfc1.jpeg

Startup diagnostics explicitly check whether /index.html exists.

If the file is missing, the ESP32 reports:

ERROR: /index.html missing from ESP32 LittleFS

Upload the filesystem after placing the dashboard assets in the correct LittleFS data directory.

13. Firebase Data Path

The device publishes to:

devices/
└── rfc-environment-001/
    └── latest/

The browser dashboard is configured to read the same path.

The dashboard expects a valid updatedAt timestamp and a structured snapshot containing the dashboard slides/rows.

Conceptually:

{
  "updatedAt": "2026-08-17T00:00:00",
  "slides": [
    {
      "title": "Air Quality",
      "rows": [
        {
          "label": "PM2.5",
          "value": "12 µg/m³"
        }
      ]
    }
  ]
}

14. Firebase Authentication Flow

The firmware uses Firebase Email/Password authentication through the REST API.

Flow:

ESP32
  │
  ├── Sign in with device email/password
  │
  ├── Receive ID token + refresh token
  │
  ├── Store refresh token in Preferences
  │
  ├── Refresh ID token when required
  │
  └── PUT latest snapshot to Realtime Database

The firmware detects HTTP 401/403 upload errors and invalidates the current authentication state so the next operation can authenticate again.

15. Security Requirements

Never commit secrets

Add local configuration files to .gitignore:

.env
.env.*
secrets.h
credentials.h
*.secret

Do not commit:

Wi-Fi passwords

Firebase device passwords

private keys

access tokens

refresh tokens

service-account JSON files

Rotate exposed credentials

If a password or device credential has already been committed to a repository, changing the source code alone is not enough.

Change the Firebase device password.

Revoke/replace affected tokens if applicable.

Update the local ESP32 configuration.

Remove secrets from Git history if they were committed.

Push only sanitized configuration to the repository.

TLS

Production Firebase communication should use certificate validation:

client.setCACert(FIREBASE_GTS_ROOT_R1);

Do not leave:

client.setInsecure();

enabled in production code. The supplied firmware contains setInsecure() in diagnostic authentication code; this must be removed or restricted to controlled troubleshooting builds before production deployment.

16. Build and Flash

Arduino IDE

Install ESP32 board support.

Install all required libraries.

Open the firmware source.

Select the correct ESP32 board.

Select the correct COM port.

Configure Wi-Fi and Firebase locally.

Compile.

Upload firmware.

Upload LittleFS contents.

Open Serial Monitor at:

115200 baud

PlatformIO

Recommended structure:

firmware/esp32/
├── src/main.cpp
├── data/index.html
├── data/rfc1.jpeg
└── platformio.ini

Typical commands:

pio run
pio run -t upload
pio run -t uploadfs
pio device monitor -b 115200

17. Startup Verification

A successful boot should verify:

LittleFS /index.html
I2C bus
SHT31
BME680
BMP280
BH1750
VEML6075
DS3231
PMS7003 UART
MH-Z19C UART
TDS ADC
pH ADC
Rain ADC
MAX9814 ADC
Wi-Fi
RTC/NTP
Firebase authentication
Firebase upload

The firmware includes a startup self-test and reports the configured pins and update intervals over Serial.

18. First Commissioning Procedure

Step 1 — Hardware

Verify:

Common ground

Correct 3.3 V/5 V requirements

I²C SDA/SCL

UART RX/TX crossing

Analog output voltage safety

Stable power supply

Step 2 — Flash

Upload firmware.

Step 3 — Upload LittleFS

Upload:

index.html
rfc1.jpeg

Step 4 — Serial Monitor

Open:

115200 baud

Check the I²C scan and startup self-test.

Step 5 — Wi-Fi

Verify:

[WiFi] Connected

and note the ESP32 IP address.

Step 6 — Local dashboard

Open:

http://ESP32_IP/

Step 7 — Local API

Open:

http://ESP32_IP/api/sensors

Then:

http://ESP32_IP/api/health

Step 8 — Firebase

Verify:

[Firebase] Authenticated
[Firebase] Upload OK

Then confirm:

devices/rfc-environment-001/latest

is updated.

Step 9 — Sensor calibration

Calibrate:

pH

TDS

rain

sound if a physical dB value is required

Step 10 — Long-duration test

Run the system continuously before declaring production readiness.

19. Troubleshooting

Dashboard says "Waiting for valid data"

Check:

Firebase path
updatedAt
slides structure
Firebase READ rules
ESP32 upload status

Firebase permission denied

Check:

Realtime Database rules

Browser read permissions

ESP32 authenticated account

Device ID/path

Firebase Authentication status

Firebase upload returns 401/403

Check:

Device email/password

Firebase API key

Token refresh

Database rules

Device authentication status

pH becomes blank/—

Check:

pH module power

common ground

ADC output voltage

GPIO35 wiring

ADC noise

pH calibration

buffer-solution calibration

The firmware exposes:

phRaw
phVoltage

in diagnostics, which should be used to identify ADC instability.

TDS is incorrect

Check:

probe wiring

TDS module supply

temperature compensation

calibration factor

reference solution

ADC voltage

PMS7003 not responding

Check:

PMS7003 TX → ESP32 GPIO16
PMS7003 RX → ESP32 GPIO17
GND common
9600 baud

MH-Z19C not responding

Check:

MH-Z19C TX → ESP32 GPIO27
MH-Z19C RX → ESP32 GPIO26
GND common
9600 baud

/index.html missing

Verify that the file is present in the LittleFS filesystem and that the filesystem image was uploaded successfully.

20. API Examples

Get sensors

curl http://ESP32_IP/api/sensors

Get health

curl http://ESP32_IP/api/health

Force refresh

curl -X POST http://ESP32_IP/api/refresh

21. Keyboard Controls — Dashboard

The dashboard source documents these controls:

Key

Function

Left Arrow

Previous slide

Right Arrow

Next slide

P

Pause

F

Fullscreen

D

Toggle diagnostics

The dashboard also provides Previous/Next buttons and slide indicators.

22. Data Reliability

The browser dashboard is designed so that a temporary Firebase failure does not immediately erase a valid display.

It can:

retain the last valid snapshot

restore cached data after a browser restart

distinguish live, cached, stale and offline states

reject malformed Firebase snapshots

avoid replacing valid displayed data with invalid payloads

The dashboard treats an ESP32 snapshot as stale after approximately 5 minutes and periodically checks freshness.

23. Production Checklist

Before production deployment:

Remove all hard-coded credentials

Rotate any credentials previously exposed

Configure Firebase Authentication

Configure strict Realtime Database rules

Remove setInsecure() from production builds

Verify TLS certificate validation

Calibrate pH

Calibrate TDS

Calibrate rain sensor

Verify PMS7003

Verify MH-Z19C

Verify all I²C devices

Verify DS3231/NTP

Verify LittleFS

Verify /api/sensors

Verify /api/health

Verify Firebase uploads

Verify browser dashboard

Test Wi-Fi disconnect/reconnect

Test Firebase disconnect/reconnect

Test power-cycle recovery

Test long-duration operation

Change dashboard publish interval to the intended production value only after commissioning

Document final sensor calibration constants

Record final hardware wiring and device IDs

24. Current Limitations

The supplied firmware currently does not directly measure:

Calcium

Magnesium

laboratory Total Hardness

direct solar-generation power/energy

direct CO₂ savings

tree count

These values should not be represented as measured sensor values unless a valid source is added.

If estimated values are implemented, they must be clearly labelled as estimates and the estimation method documented.

25. License

Add the project's actual license before publishing this repository.

Example:

Copyright (c) GYAM Technologies

All rights reserved.

Replace this section with the legal license selected for the project.

26. Project Status

Stage: ESP32 commissioning / production-preparation

The firmware contains production-oriented diagnostics, validation, Firebase authentication, local APIs and persistent state, but final production deployment requires completion of credential management, sensor calibration, security review, hardware validation and long-duration commissioning.
