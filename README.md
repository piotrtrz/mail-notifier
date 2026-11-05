# Mail Notifier

Battery-powered ESPHome device that sends a notification whenever the mailbox is opened. An ESP32-C6 sleeps in deep sleep and is woken by a reed switch mounted on the mailbox door; it reports the event over MQTT (and optionally exposes the native Home Assistant API), then goes back to sleep.

## How it works

1. The device is in deep sleep and only wakes when the reed switch on `GPIO7` goes high (`ESP_SLEEP_WAKEUP_EXT1`) — i.e. the mailbox door is opened — or once every 24 h for a routine battery report.
2. On boot there is a 10 s window during which OTA updates are accepted.
3. The device connects to Wi-Fi and MQTT, then publishes:
   - battery voltage and charge percentage (retained),
   - `alarm` = `TRIGGERED` if it was woken by the reed switch (not retained),
   - current reed switch state: `OPEN` or `CLOSED` (retained).
4. If the reed switch is closed again, the device re-enters deep sleep. If it is still open (mailbox door left open), the device stays awake and re-checks every 5 s until the door is closed, so the closing event is not missed.

An MQTT Last Will and Testament on `<device_name>/status` (`online` / `offline`, retained) makes the device's availability visible to consumers.

## Hardware

| Component | Connection |
| --- | --- |
| ESP32-C6 DevKitC-1 | — |
| Reed switch (magnetic sensor) | `GPIO7`, internal pull-up; opens when the mailbox door magnet is away |
| Li-ion / LiPo battery | `GPIO0` (ADC, 12 dB attenuation) through a 2:1 voltage divider (`multiply: 2.0` filter) |

The battery percentage is derived from voltage with a 3.0 V – 4.15 V LiPo curve.

## MQTT topics

All topics are prefixed with `device_name` from `esp32-c6.yaml` (default: `battery-sensor-7f3a`).

| Topic | Payload | Retained | Description |
| --- | --- | --- | --- |
| `<device_name>/status` | `online` / `offline` | yes | Availability (birth / LWT message) |
| `<device_name>/alarm` | `TRIGGERED` | no | Published when woken by the reed switch |
| `<device_name>/reed_switch` | `OPEN` / `CLOSED` | yes | Current state of the mailbox door |
| `<device_name>/battery/voltage` | volts, e.g. `4.02` | yes | Battery voltage |
| `<device_name>/battery/percent` | `0`–`100` | yes | Estimated battery level |

## Configuration

The device is configured entirely with [ESPHome](https://esphome.io/). Secrets are loaded from `secrets.yaml`, which must define:

```yaml
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
mqqt_broker: "192.168.1.10"
mqqt_username: "mqtt-user"
mqqt_password: "mqtt-password"
mail_notifier_ha_api_key: "base64-32-bytes-api-key"
mail_notifier_ota_password: "ota-password"
```

> Note: the MQTT secret names are intentionally spelled `mqqt_*` to match the firmware config.

Adjust `device_name` and `friendly_name` in the `substitutions:` section of `esp32-c6.yaml` if needed.

## Flashing

```bash
esphome run esp32-c6.yaml
```

For the first flash use USB. Later updates can be pushed over the air — the device accepts OTA for 10 s after every boot (it reboots at least once every 24 h, or immediately after a power cycle).

## Home Assistant

The native ESPHome API is enabled with encryption (`mail_notifier_ha_api_key`), so the device can be added to Home Assistant via the ESPHome integration. Because the device is in deep sleep most of the time it is only intermittently reachable over the API; `reboot_timeout: 0s` on both `api:` and `wifi:` prevents needless reboots when the connection is unavailable. For reliable, always-available state, consume the MQTT topics (e.g. with an MQTT binary sensor) and use `device_name/alarm` as a notification trigger.

## Power notes

- Deep sleep with `LIGHT` Wi-Fi power saving and `fast_connect` keeps current draw low; battery life depends mainly on how often the mailbox is opened and on the 24 h wake-ups.
- `sleep_duration: 24h` acts as a watchdog: even if the mailbox is never opened, the device reports battery level and availability once a day.
