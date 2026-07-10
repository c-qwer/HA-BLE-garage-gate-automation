# HA BLE Garage Gate Automation

A simple Home Assistant and ESPHome design for garage / driveway gate automation using BLE vehicle presence.

The project is intended for garages or gates where a vehicle can be detected by a BLE signal, such as a Tesla BLE key, a BLE beacon, or another device that exposes a stable BLE presence signal. When an authorized vehicle approaches the garage, the door can open automatically. When the vehicle leaves, the system can close the door after the path has stayed clear for a configured delay.

## Features

- Automatically opens the garage/gate when an authorized BLE vehicle is detected.
- Automatically closes the garage/gate after a vehicle leaves.
- Uses RSSI threshold and delayed-on filtering to reduce false triggers from weak or noisy BLE signals.
- Uses an obstacle sensor, such as an IR beam, before closing.
- Supports multiple vehicle presence sensors.
- Includes an optional open-gate reminder blueprint with actionable mobile notifications.
- The low cost of upgrading. I spent less than $50 for my garage door.

## Requirements

- Home Assistant.
- ESPHome.
- ESP32, or another ESPHome-compatible device with BLE support.
- Obstacle sensor, such as an IR beam or safety beam.
- Garage/gate controller exposed in Home Assistant as a `cover` entity.
  - The cover should report the current gate state, such as `open` and `closed`.
  - The cover should support open and close commands.
- BLE beacon (if the vehicle does not support BLE)
- Smart Relay connecting to the controller or motor to control the gate

## BLE Vehicle Options

The BLE source matters because different devices behave differently around a garage.

### Persistent BLE Device

Examples include battery-powered BLE beacons or devices that keep advertising all the time.

This is useful for presence detection, but if the vehicle is parked near the ESP32, the vehicle may remain detected even when it is not being driven. In this case, starting the car may not create a new `off -> on` transition, so it may not open the door automatically.

### USB-Powered BLE Beacon

A USB-powered beacon plugged into the vehicle can be a good option.

When the car is off, the beacon may be off. When the car starts, the beacon powers on and the BLE presence sensor changes from `off` to `on`. This can automatically trigger the garage/gate to open.

## How It Works

The ESP32 scans for authorized BLE devices and exposes each detected vehicle as a Home Assistant binary sensor.

The BLE presence sensor uses:

- `min_rssi` to ignore signals weaker than the configured threshold.
- `delayed_on` to confirm the signal is stable before treating it as present.
- `timeout` to turn the sensor off after the BLE signal has not been seen for a while.

The main automation then uses those presence sensors:

- When a vehicle changes from `off` to `on`, the automation opens the garage/gate if it is currently closed.
- When a vehicle changes from `on` to `off`, the automation starts a safe close sequence if the garage/gate is open.
- Before closing, the obstacle sensor must stay clear for the configured delay.
- If an obstacle appears during the countdown, the countdown restarts.
- If a vehicle returns during the countdown, the close attempt is cancelled.

## Repository Structure

```text
Blueprints/
  automation/
    ble_garage_gate_automation.yaml   # Main BLE open/close automation
    gate_close_reminder.yaml          # Optional open-gate reminder

esphome/
  examples/
    garage_esp_example.yaml           # ESPHome BLE and obstacle sensor example
```

## Setup

### 1. Find Your Vehicle BLE MAC Address

If your vehicle or BLE beacon uses a stable BLE MAC address, you need that address for the ESPHome `ble_presence` sensor.

The easiest way is to use a BLE scanner app on your phone.

1. Install a BLE scanner app on your phone.
   - iOS: apps such as `nRF Connect` or `LightBlue`.
   - Android: apps such as `nRF Connect`, `BLE Scanner`, or similar.
2. Move close to the vehicle, vehicle key, or BLE beacon.
3. Open the scanner app and start scanning.
4. Look for a device that appears consistently and has a stronger RSSI when you move closer to the vehicle or beacon.
5. Copy the BLE MAC address shown by the app.

The address usually looks like:

```text
AA:BB:CC:DD:EE:FF
```

Use that address in the ESPHome configuration:

```yaml
binary_sensor:
  - platform: ble_presence
    mac_address: AA:BB:CC:DD:EE:FF
    name: "Vehicle 1 Presence"
```

Tip: stand near the vehicle or beacon first, note the devices with strong RSSI, then move away and scan again. The correct device should become weaker or disappear as distance increases.

Some vehicles, phones, and BLE devices use randomized BLE addresses. If the MAC address changes over time, MAC-based tracking will not be reliable. In that case, use a dedicated BLE beacon with a stable MAC address, or use the iBeacon option if your beacon supports UUID/major/minor broadcasting.

### 2. Configure ESPHome

Use the example file:

```text
esphome/examples/garage_esp_example.yaml
```

Update the substitutions and BLE identifiers:

```yaml
substitutions:
  threshold_min_rssi: -80dB
  ble_timeout: 15s
  ble_delayed_on: 2s
  obstacle_sensor_pin: GPIO13
```

Replace the example BLE MAC address:

```yaml
mac_address: XX:XX:XX:XX:XX:XX
```

with the MAC address of your BLE beacon or vehicle BLE device.

If your device broadcasts iBeacon instead of a stable MAC address, use the commented iBeacon example in the ESPHome file.

### 3. Add Secrets

The ESPHome example expects secrets such as:

```yaml
wifi_ssid: "Your WiFi"
wifi_password: "Your WiFi Password"
garage_ble_node_api_key: "Generated ESPHome API encryption key"
garage_ble_node_ota_password: "OTA password"
garage_ble_node_fallback_password: "Fallback hotspot password"
```

Generate an ESPHome API encryption key with:

```bash
openssl rand -base64 32
```

### 4. Import the Home Assistant Blueprint

Import or copy this blueprint into Home Assistant:

```text
Blueprints/automation/ble_garage_gate_automation.yaml
```

Create an automation from the blueprint and select:

- Garage/gate `cover` entity.
- Obstacle binary sensor.
- One or more vehicle presence binary sensors.
- Close delay in seconds.

### 5. Optional Reminder Automation

You can also import:

```text
Blueprints/automation/gate_close_reminder.yaml
```

This sends an actionable mobile notification if the gate has been open too long, or at a scheduled daily check time.

## Tuning RSSI

RSSI is not a precise distance measurement. It can change because of vehicle position, body blocking, walls, metal garage doors, antenna direction, and BLE advertising behavior.

General guidance:

```text
-60 dBm: close
-70 dBm: medium range
-80 dBm: weaker / further away
-90 dBm: very weak or unreliable
```

Start with:

```yaml
threshold_min_rssi: -80dB
ble_timeout: 15s
ble_delayed_on: 2s
```

Then adjust based on your garage:

- If the door opens too early or from too far away, make the threshold stronger, such as `-75dB`.
- If the vehicle is not detected reliably, make the threshold weaker, such as `-85dB`.
- If detection flickers, increase `ble_timeout`.
- If false positives happen, increase `ble_delayed_on`.

## Safety Notes

This project controls a moving garage door or gate. Use it carefully.

- Always keep a working physical safety mechanism on the garage/gate.
- Use an obstacle sensor before automatic closing.
- Test the automation with the motor disconnected or in a safe condition first.
- Make sure the cover state in Home Assistant is accurate.
- Do not rely on BLE as the only safety signal.

## Limitations

- BLE RSSI is noisy and should not be treated as exact distance.
- Some vehicles or phones use randomized BLE addresses, which may not work with MAC-based tracking.
- A vehicle parked near the ESP32 may stay detected continuously if the BLE source is always powered.
