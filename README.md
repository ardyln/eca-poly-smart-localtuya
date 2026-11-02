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
  - **Location**, the app's geofence mode (driven by DP 110, which the template does not expose)
- **Sensors** for the work state, the RF link to the room unit, heat/cool mode, the boiler relay, the window-open flag, the user mode switch status, and the cooling-mode temperatures.
- **Numbers** for the comfort, eco, holiday and location heating temperatures.

## Install

1. Install [hass-localtuya](https://github.com/xZetsubou/hass-localtuya) (HACS custom repository). This template is **not** for the original rospogrigio/localtuya.
2. Copy `templates/eca_poly_smart.yaml` into `custom_components/localtuya/templates/` and restart Home Assistant.
3. Add the device in LocalTuya: device ID, local key, host, protocol **3.4**. Get the local key the usual way (Tuya IoT platform, tinytuya, etc.).
4. When asked to pick an entity type, choose **Import template** → `eca_poly_smart.yaml`.

## DP map

This is the device's official thing model from the Tuya Cloud API (`/v2.0/cloud/thing/{device_id}/model`). All temperatures are integers ×10 with step 5 (0.5 °C). The temperature settings range from 50 to 400 (5–40 °C); the room temperature ranges from −200 to 600.

| DP | Tuya code | Access | Type | Meaning | In template |
|---|---|---|---|---|---|
| 1 | `switch` | rw | bool | Power | climate on/off |
| 2 | `mode` | rw | enum | Operating mode: `eco`, `comfort`, `holiday`, `program`, `manual`, `location`, `rf_thermostat` | climate presets |
| 3 | `work_state` | ro | enum | Current mode (same values as DP 2) | sensor |
| 16 | `temp_set` | rw | value | Target temperature | climate target |
| 24 | `temp_current` | ro | value | Room temperature (from the RF room unit) | climate current |
| 72 | `week_program_5` | rw | raw | Weekly program (binary) | – |
| 101 | `relay_status` | ro | bool | Boiler relay on/off | binary sensor, climate action |
| 102 | `eco_mode_temp_heat` | rw | value | Eco temperature, heating | number |
| 103 | `comfort_mode_temp_heat` | rw | value | Comfort temperature, heating | number |
| 104 | `holiday_mode_temp_heat` | rw | value | Holiday temperature, heating | number |
| 105 | `unpair_rf_thermostat` | wr | bool | Unpair the RF room unit | – |
| 106 | `rf_thermostat_pair_status` | ro | enum | RF link: `online` (app: "Paired"), `offline`, `timeout`, `match_mode`, `not_match` | sensor |
| 107 | `clear_program_mode` | wr | bool | Clear the weekly program | – |
| 108 | `heating_mode` | rw | enum | `heat` / `cool` | climate hvac mode |
| 109 | `set_mode_settings_default` | wr | bool | Reset the mode temperatures to defaults | – |
| 110 | `set_location_inside` | wr | bool | Set in/out of location (geofence) | – |
| 111 | `location_inside_temp_heat` | rw | value | Location inside temperature, heating | number |
| 112 | `locationoutside_temp_heat` | rw | value | Location outside temperature, heating | number |
| 113 | `eco_mode_temp_cool` | rw | value | Eco temperature, cooling | sensor |
| 114 | `comfort_mode_temp_cool` | rw | value | Comfort temperature, cooling | sensor |
| 115 | `holiday_mode_temp_cool` | rw | value | Holiday temperature, cooling | sensor |
| 116 | `location_inside_temp_cool` | rw | value | Location inside temperature, cooling | sensor |
| 117 | `locationoutside_temp_cool` | rw | value | Location outside temperature, cooling | sensor |
| 118 | `window_open_status` | ro | bool | Open window detected | binary sensor |
| 119 | `rf_message` | ro | raw | RF message (binary) | – |
| 120 | `user_mode_switch_status` | ro | bool | User mode switch status | binary sensor |

"wr" DPs are write-only commands. The device never reports them, so local discovery can't see them. The template leaves out the commands and the raw DPs. DP 120 is read-only, which is why writing to it does nothing.

### How the map was made

The device's DP numbers, codes and types come from the thing model in the Tuya Cloud API (Tuya IoT platform, IoT Core). The labels come from the Tuya Smart app's translation file for this product. The mapping was then tested on the real device: each value was changed from Home Assistant and the Tuya app was checked to show the same change. The Tuya files themselves are not included in this repo.

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
