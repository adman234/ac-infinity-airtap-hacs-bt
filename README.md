# AC Infinity AirTap — Home Assistant

Two generations of control for [AC Infinity AirTap](https://acinfinity.com/register-booster-fans/)
register booster fans.

## Current — ESPHome on a XIAO ESP32-C6 ([`esphome/`](esphome))

The AirTap controller is replaced with a Seeed XIAO ESP32-C6 driving the fan over
PWM, with an SSD1306 OLED, the four original panel buttons, and an NTC probe in
the vent. It talks to Home Assistant over WiFi using the native API.

The **device** owns the control decision. HA supplies room temperature, the
thermostat setpoints and the current `hvac_action`; the vent computes its own
demand and keeps running on its last known inputs if HA restarts. Every boot
lands in AUTO at 40%, and only a panel button or a person in HA can change that.

See [`esphome/README.md`](esphome/README.md) for the control model, the per-device
layout, and the full list of changes from rev 1.4.2.

## Legacy — BLE HACS integration ([`custom_components/`](custom_components))

The original approach: a HACS custom integration talking to the stock AirTap over
Bluetooth LE. Superseded by the ESPHome firmware above and kept for reference.

Worth knowing if you are still running it: the integration's Power switch is what
knocks the vents out of AUTO mode. `ac_infinity_ble`'s `turn_on()` sends
`work_type = 2` (manual on) and `turn_off()` sends `work_type = 1` (manual off),
so every on/off command leaves AUTO. `turn_on()` with no speed also falls back to
`level_on or 10`, which is where the unexplained speed-10 came from.

### Fork changes

- **Simplified entities:** removed auto/climate mode controls; temperature is a
  read-only sensor rather than part of a climate device
- **Single fan speed:** one speed setting (1–10) instead of separate min/max
- **Stability fixes:** a 500 error during discovery caused by nearby
  non-AC-Infinity Bluetooth devices, an assertion error on switch toggle, and a
  config entry serialization issue that could break the integration after a
  restart

To enable debug logging, add to your
[logger config](https://www.home-assistant.io/integrations/logger/):

```yaml
logger:
  default: info
  logs:
    ac_infinity_ble: debug
    custom_components.ac_infinity: debug
```

## Credits

BLE integration built on work by
[mtsphere](https://github.com/mtsphere/ac-infinity-airtap-hacs) and originally
[Jason Hunter (hunterjm)](https://github.com/hunterjm/ac-infinity-hacs), using the
[ac-infinity-ble](https://github.com/hunterjm/ac-infinity-ble/) library.
