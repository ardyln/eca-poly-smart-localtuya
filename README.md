# E.C.A. Poly Smart thermostat with LocalTuya

This repo gets the **E.C.A. Poly Smart** Wi-Fi boiler thermostat (a receiver wired to the boiler, plus a wireless RF room unit) working locally in Home Assistant. It contains a device template for [hass-localtuya](https://github.com/xZetsubou/hass-localtuya) and a datapoint (DP) map, tested on a real device.

E.C.A. sells the thermostat with its own app, "E.C.A. Poly Smart". Underneath it is a Tuya device, and it pairs just as well with the Tuya Smart / Smart Life apps. That is what makes local control possible.

| | |
|---|---|
| Tuya product ID | `rufjdpbdogijhq89` |
| Protocol | 3.4 |
| Tested with | hass-localtuya 2026.10.0, Home Assistant 2026.10 |

## What you get

- **A climate entity** with on/off, heat/cool, current and target temperature, and a "heating" indicator from the boiler relay. Its presets are:
  - **comfort**, **eco**, **away** (Holiday) and **Program**, the four modes the app offers
  - **Thermostat**, which hands control back to the RF room unit
  - **Manual**, which the device switches to by itself when a target temperature is set
- **Sensors** for the work state, the RF link to the room unit, heat/cool mode, the boiler relay, the window-open flag, and the cooling-mode temperatures.
- **Numbers** for the comfort, eco, holiday and location heating temperatures.

## Install

1. Install [hass-localtuya](https://github.com/xZetsubou/hass-localtuya) (HACS custom repository). This template is **not** for the original rospogrigio/localtuya.
2. Copy `templates/eca_poly_smart.yaml` into `custom_components/localtuya/templates/` and restart Home Assistant.
3. Add the device in LocalTuya: device ID, local key, host, protocol **3.4**. Get the local key the usual way (Tuya IoT platform, tinytuya, etc.).
4. When asked to pick an entity type, choose **Import template** → `eca_poly_smart.yaml`.

## DP map

**Confirmed** means checked against the Tuya app: each value was changed from Home Assistant and the app was seen to show the same change.

| DP | Tuya code | Type | Meaning | Status |
|---|---|---|---|---|
| 1 | `switch` | bool | Power | confirmed |
| 2 | `mode` | enum | Operating mode. The app sets `comfort`, `eco`, `holiday`, `program`; the device also reports `rf_thermostat` and `manual` | confirmed |
| 3 | `work_state` | enum | Current mode (read-only mirror of DP 2) | confirmed |
| 16 | `temp_set` | int ×10 | Target temperature | confirmed |
| 24 | `temp_current` | int ×10 | Room temperature (from the RF room unit) | confirmed |
| 101 | `relay_status` | bool | Boiler relay on/off | confirmed |
| 102 | `eco_mode_temp_heat` | int ×10 | Eco temperature, heating | confirmed |
| 103 | `comfort_mode_temp_heat` | int ×10 | Comfort temperature, heating | confirmed |
| 104 | `holiday_mode_temp_heat` | int ×10 | Holiday temperature, heating | confirmed |
| 106 | `rf_thermostat_pair_status` | enum | RF link: `online` (app: "Paired"), `offline`, `timeout`, `match_mode`, `not_match` | confirmed |
| 108 | `heating_mode` | enum | `heat` / `cool` | confirmed |
| 111 | `location_inside_temp_heat` | int ×10 | Location mode (geofence), inside temperature, heating | from the code name |
| 112 | `location_outside_temp_heat` | int ×10 | Location mode, outside temperature, heating | from the code name |
| 113 | `eco_mode_temp_cool` | int ×10 | Eco temperature, cooling | confirmed |
| 114 | `comfort_mode_temp_cool` | int ×10 | Comfort temperature, cooling | confirmed |
| 115 | `holiday_mode_temp_cool` | int ×10 | Holiday temperature, cooling | confirmed |
| 116 | probably `location_inside_temp_cool` | int ×10 | Location inside, cooling | inferred |
| 117 | probably `location_outside_temp_cool` | int ×10 | Location outside, cooling | inferred |
| 118 | probably `window_open_status` | bool | Open-window detection | inferred, not yet seen to trigger |
| 120 | probably `set_location_inside` | bool | In/out of location | inferred; **ignores writes**, so it's read-only here |

### How the map was made

The Tuya app downloads two things for each product: a translation file listing every DP code and enum label, and the device's control panel (a React Native bundle). Both were read from the Tuya Smart app's data on a rooted phone. They give the codes and values but not the code → DP number table, which sits in the app's encrypted storage. The DP numbers were matched to codes by their live values. Each mapping was then confirmed by changing it from Home Assistant and comparing with the Tuya app's screen. Neither Tuya file is included in this repo.

## How the device behaves

- **Setting a temperature switches the mode to `manual`.** This is the same thing that happens when you drag the dial in the app.
- **Going back to Thermostat (`rf_thermostat`) keeps the last target.** For example, after Holiday the target stays at 5 °C until the room unit sends a new setpoint. To return cleanly, set the target first, then select the Thermostat preset.
- **When the RF room unit is not `online`, the receiver won't heat.** The app's own panel blocks heating in that state, and also when the power is off or the window flag is set. If the room unit's batteries run low, DP 106 goes to `offline`/`timeout` and the room temperature freezes.
- **⚠️ Cool mode on a gas boiler.** In `cool`, the relay closes when the room is *warmer* than the target. A boiler wired to that relay will then fire, heating a room that is already too warm. The template keeps `cool` because the device supports it, but don't use it unless your relay drives a cooling unit. Home Assistant will also label that relay state "heating".

## Known issues (hass-localtuya 2026.10.0)

- **The temperature numbers are raw ×10.** Number scaling re-applies on every read, so 18.0 °C would show as 1.8, then 0.18 ([xZetsubou/hass-localtuya#893](https://github.com/xZetsubou/hass-localtuya/issues/893)). Until that's fixed, the numbers stay unscaled (180 = 18.0 °C). Sensors are not affected, which is why DP 113–117 show °C.
- **Eco has to go through `eco_dp`.** The climate entity sends the HA preset name `eco` to `eco_dp`/`eco_value`, never to `preset_set`. So eco is configured as `eco_dp: 2`, `eco_value: eco` and left out of the preset map. Otherwise selecting Eco silently does nothing.
- **Renaming an entity while editing the device recreates it**, so the entity can get a new entity_id. To change display names, use the entity settings in Home Assistant instead.
- **Use DP 1 for the climate entity.** Its `id` is the power DP. If the climate sits on DP 16 instead, "off" would write `False` into the target temperature.

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).
