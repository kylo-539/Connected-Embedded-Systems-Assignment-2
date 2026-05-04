# Connected-Embedded-Systems-Assignment-2 

# Assignment 2: MQTT Telemetry + Visualisation + LED Alerts
# Grade: 90%
## Source code and video demonstration available upon request

## What This Project Currently Does
This project is a complete MQTT-based telemetry workflow made up of multiple programs:

1. A Qt desktop dashboard that subscribes to live telemetry and plots it.
2. A publisher that reads ADXL345 orientation plus Raspberry Pi CPU temperature and publishes JSON to MQTT.
3. A roll-angle subscriber that drives three GPIO LEDs based on tilt severity.
4. A CPU temperature subscriber that prints thermal warnings and flashes an LED when hot.

It is an end-to-end sensing, messaging, visualisation, and hardware-alert system.

## Runtime Components

### 1) Qt Dashboard (`assignment2` app)
The Qt app (built from `EEN1071Ass2.pro`) uses Paho MQTT + QCustomPlot and provides:

- Connect/disconnect controls for MQTT.
- Topic selection via radio buttons:
    - `een1071/ADXL345`
    - `een1071/CPUTemp`
- Live plotting with a scrolling time axis.
- A status/log text area that shows received payloads.
- A rolling average display (up to 120 most recent samples).

Current plotting behavior:

- ADXL345 mode:
    - Graph 0: Pitch
    - Graph 1: Roll
    - Y-axis: -180 to 180 degrees
    - Average field: current roll + average roll
- CPU mode:
    - Graph 0: CPU temperature
    - Y-axis: 0 to 60 degC
    - Average field: current temperature + average temperature

The Up/Down buttons exist for manual value adjustment/testing inside the UI.

### 2) Publisher (`publish.cpp`)
The publisher runs on a device with access to:

- ADXL345 over I2C (bus 1, address `0x53`)
- Linux CPU temp file: `/sys/class/thermal/thermal_zone0/temp`

It continuously publishes retained JSON messages every 500 ms with QoS 1:

- Topic `een1071/ADXL345`
    - Payload format: `{"time": <unix>, "pitch": <float>, "roll": <float>}`
- Topic `een1071/CPUTemp`
    - Payload format: `{"time": <unix>, "CPUTemp": <float>}`

It also configures an MQTT Last Will on `een1071/status`.

### 3) Roll Subscriber with LED States (`subscribe.cpp`)
This subscriber listens to `een1071/ADXL345`, parses JSON `roll`, and drives 3 GPIO outputs using `libgpiod`:

- GPIO 23: Low state
- GPIO 25: Medium state
- GPIO 24: High state

State logic (based on `delta = abs(roll - (-5.0))`):

- `delta < 4` => LOW LED on
- `delta <= 20` => MEDIUM LED on
- otherwise => HIGH LED on

Only one state LED is active at a time.

### 4) CPU Subscriber with Thermal Warnings (`sub2.cpp`)
This subscriber listens to `een1071/CPUTemp`, parses JSON `CPUTemp`, and applies thresholds:

- `<= 30`: normal message
- `> 30`: warm warning message
- `> 40`: high warning + flash LED on GPIO 24

This program uses a persistent MQTT session (`cleansession = 0`) for its client.

## MQTT Setup Used in Code

- Desktop Qt app broker address: `tcp://127.0.0.1:1883`
- Publisher/subscribers broker address: `tcp://192.168.1.41:1883`

If you run everything in one environment, align these addresses first.

MQTT credentials are currently hard-coded in source for assignment testing.

## Data Contracts (JSON)

- ADXL345 topic message:

```json
{
    "time": 1710000000,
    "pitch": 12.34,
    "roll": -3.21
}
```

- CPU temperature topic message:

```json
{
    "time": 1710000000,
    "CPUTemp": 41.56
}
```

## Dependencies

### Qt dashboard
- Qt (Core, GUI, Widgets, PrintSupport)
- QCustomPlot (included in repository)
- Eclipse Paho MQTT C client (`libpaho-mqtt3c`)

### Publisher / hardware subscribers (Linux/Raspberry Pi)
- Eclipse Paho MQTT C client
- `libgpiod` (GPIO control)
- `json-c` (JSON parsing in subscribers)
- ADXL345 support code in `ADXL345/`

## Build Notes

### Qt dashboard
1. Open `EEN1071Ass2.pro` in Qt Creator.
2. Ensure linker can find `-lpaho-mqtt3c`.
3. Build and run.

### Publisher/subscribers
These are standalone C/C++ source files and are intended to be built on Linux/Raspberry Pi with the dependencies above installed.

## Typical End-to-End Run Order
1. Start MQTT broker.
2. Start `publish` on the sensor device.
3. Optionally start `subscribe` and/or `sub2` for GPIO/console alerts.
4. Start the Qt dashboard and click Connect.
5. Switch between ADXL345 and CPUTemp topics in the UI.

## Repository Layout (Key Files)
- `EEN1071Ass2.pro`: Qt project file.
- `mainwindow.cpp` / `mainwindow.h`: Dashboard logic, MQTT callbacks, plotting, topic switching, rolling average.
- `mainwindow.ui`: Dashboard UI layout.
- `publish.cpp`: Sensor + CPU temperature publisher.
- `subscribe.cpp`: Roll-based 3-level LED alert subscriber.
- `sub2.cpp`: CPU temperature warning subscriber with LED flash.
- `ADXL345/`: I2C ADXL345 driver and support classes.

## Notes
- The dashboard processes MQTT messages in a thread-safe way by emitting a Qt signal from the MQTT callback and updating the UI in the main thread.
- Topic switching in the dashboard unsubscribes from the previous topic and subscribes to the selected one.
- Retained MQTT messages are enabled in the publisher for both telemetry topics.

---
Developed for EEN1071 Assignment 2.
