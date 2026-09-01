# AirTap vent controller — ESPHome (XIAO ESP32-C6)

ESPHome firmware for AC Infinity AirTap T-series register booster fans whose
controller has been replaced with a Seeed XIAO ESP32-C6.

This supersedes the BLE HACS integration in [`../custom_components`](../custom_components),
which is kept for reference only.

## Layout

```
esphome/
├── packages/airtap-vent.yaml   # all the logic — shared by every unit
├── airtap-office.yaml          # per-device: substitutions + include
├── airtap-guest.yaml
└── secrets.yaml.example
```

Adding another vent is a five-line file. Copy `airtap-guest.yaml`, change `name`,
`device_address` and `room_temp_entity`.

Copy `secrets.yaml.example` to `secrets.yaml` in your ESPHome directory and fill
it in. The real `secrets.yaml` is gitignored.

## Control model

The **device** owns the decision. Home Assistant supplies three inputs over the
native API — room temperature, the thermostat's two setpoints, and what the
thermostat is currently doing — and everything else is computed on-device, so the
vent keeps running on its last known inputs if HA restarts.

AUTO calls for the fan when the HVAC is actively moving air **and** boosting this
register would actually help:

```
room > cool_setpoint  AND  vent < (room - buffer)     -> cooling assist
room < heat_setpoint  AND  vent > (room + buffer)     -> heating assist
```

### Resting state

Every boot lands in **AUTO at `default_speed`** (4/10 = 40%). `auto_mode` and
`auto_speed` are deliberately *not* restored across reboots; the tuning values
(buffer, deadband, min run/off, brightness) are, because only a person changes
those.

Exactly three things drop the device to MANUAL:

| Action | Result |
|---|---|
| Panel Power / Up / Down button | MANUAL, fan follows the button |
| Panel Mode button | toggles AUTO ↔ MANUAL |
| Anyone turning the fan on/off from HA | MANUAL |

Nothing else can. The **Reset To Auto Default** button returns to AUTO at 40% on
demand, and a reboot does the same.

## What changed from rev 1.4.2

### 1. Reboots silently dropped the device into MANUAL — fixed

This was the "they keep going into manual mode and changing the speed on their
own" bug. It was not the hardware.

`fan.restore_mode: RESTORE_DEFAULT_OFF` restores the previous fan state during
`setup()`. In ESPHome, `FanRestoreState::apply()` calls `fan.publish_state()`,
which calls `state_callback_.call()`, which is what `FanTurnOnTrigger` listens
on. The trigger is edge-triggered with `last_on_` initialised to `false`, so a
fan that restores to **on** looks like a genuine off→on transition and fires
`on_turn_on` — whose lambda is:

```yaml
on_turn_on:
  - lambda: 'if (!id(auto_applying)) { id(auto_mode) = false; }'
```

`auto_applying` is false during boot, so every reboot that restored a running fan
turned AUTO off. From then on `apply_auto` returns immediately (it is gated on
`id(auto_mode)`), so the fan sat at its restored speed indefinitely — which is
also where the mystery speed came from.

Two fixes, belt and braces:

- `restore_mode: ALWAYS_OFF` — there is no reason to restore a state the auto
  logic is about to recompute.
- A `booting` global, true until `on_boot` finishes, added to both fan trigger
  guards so no boot-time state publish can ever change the mode.

### 2. Panel buttons are now debounced

The GPIO buttons had no filters. Contact bounce on the **Mode** button toggles
`auto_mode` more than once per press — which also looks exactly like the device
changing mode on its own. All four buttons now have `delayed_on/off: 20ms`.

### 3. The auto decision no longer acts on a stale value

Previously every input (`room_temp`, both setpoints, `temp_sensor`) had
`on_value: script.execute: apply_auto`, but `apply_auto` reads
`id(auto_demand).state` — and `auto_demand` is a separate template binary sensor
that re-evaluates in its own `loop()`. So `apply_auto` ran against the demand
computed from the *previous* inputs, then ran again a moment later once the
sensor caught up. Brief, but it means a real spurious fan command on every input
change.

`auto_demand.on_state` is now the single entry point into `apply_auto`, and the
input sensors have no triggers at all. One decision, one place, always fresh.

### 4. Hysteresis and anti short-cycling

`buffer` only ever guarded the vent-vs-room comparison. The flappy test is
room-vs-setpoint — a room sitting on setpoint would chatter the fan. Added:

- **Room Deadband** (default 0.5 °F) — asymmetric thresholds, so it is harder to
  start than to keep running.
- **Min Run Time** / **Min Off Time** (default 180 s / 120 s) — a state change
  that arrives too soon is held, and the 30 s interval retries it once the hold
  expires.

### 5. Only boost when the HVAC is actually running

New `hvac_action` import from the thermostat. Running a booster fan while the
system is idle just pushes stale duct air around and makes noise. Gated behind a
**Require HVAC Active** switch (default on) in case your thermostat does not
report the attribute; if HA has not reported it yet the gate falls through to the
temperature test rather than blocking forever.

### 6. BLE removed, WiFi modem sleep disabled

`esp32_improv` kept a BLE stack running permanently. The C6 has one 2.4 GHz radio
shared by WiFi and BLE, which is why the old config had to leave
`power_save_mode` at its default. The unit is provisioned from `secrets.yaml`, so
BLE provisioning was dead weight — it is gone, and `power_save_mode: NONE` is now
set instead. That matters here because the auto logic is fed by HA-imported
sensors, so native API latency is control-loop latency. `improv_serial` is kept;
it is UART and costs nothing.

### 7. Assorted

- ADC sampled every 10 s with a 5-sample median filter, instead of the 60 s
  default with no filtering — a much steadier vent temperature.
- Dropped the three `*_display` template sensors that echoed HA's own data back
  to HA every 10 s. The OLED reads the imported values directly.
- OLED now shows mode, vent temp, room temp, fan speed and whether AUTO is
  currently calling for the fan.
- `WiFi Signal` marked `entity_category: diagnostic`.

## Why not Zigbee

Evaluated and rejected. The blocker is not radio quality, it is that
[ESPHome's Zigbee component](https://github.com/luar123/zigbee_esphome) supports
only `light`, `switch`, `binary_sensor` and `sensor`:

- **No `sensor: platform: homeassistant`.** The entire control model depends on HA
  pushing room temperature and setpoints to the device. That is a native-API
  feature. Zigbee has no equivalent, so the on-device logic could not work at all.
- **No `fan` and no `number` platform.** The fan would have to be smuggled as a
  dimmable `light` (brightness → speed) and re-wrapped in a template fan on the HA
  side, and every config number (buffer, deadband, min run/off, brightness) would
  have nowhere to live.
- **No OTA, explicitly "not planned."** Keeping WiFi alongside for OTA is possible
  on the C6, but ESPHome warns that WiFi station + Zigbee **router** destabilises
  the Zigbee network — and a mains-powered vent controller is exactly what you
  want to be a router.
- ESP32-H2 is reported as more reliable than the C6 for Zigbee, with C6 boards
  that only work next to the coordinator. On the XIAO C6 specifically, GPIO3 must
  be driven low to power the RF switch and GPIO14 selects internal vs external
  u.FL — easy to get wrong, and it presents as exactly that symptom.

WiFi + the native API gives real `fan`/`number`/`switch` entities, OTA, logs, the
local web UI, and HA-imported sensors. Zigbee's only genuine advantage here is
mesh range; if a vent has weak WiFi, a u.FL external antenna is the cheaper fix.

## Flashing

```bash
esphome run airtap-office.yaml
```

`device_address` must point at whatever the device answers on **right now**. After
a rename flash lands, update it to `<name>-<mac suffix>.local`.
