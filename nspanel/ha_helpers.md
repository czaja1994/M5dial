# NSPanel companion HA helpers

Helpers the NSPanel config depends on. Created as **config-entry helpers**
(Settings → Devices & Services → Helpers), not YAML. Reproduce via the UI or MCP.

## Template switches — smart room toggles (home tiles)

Home tiles point at these instead of the raw light groups, so the tap is asymmetric:
indicator = any bulb on in the room; tap-ON turns everything off; tap-OFF turns on
only a chosen subset.

### `switch.bedroom_room`  (Template → switch, icon mdi:bed)
- **State:** `{{ is_state('light.bedroom_lights','on') }}`
- **Turn on:** `adaptive_lighting.apply` → `switch.adaptive_lighting_bedsideadaptive`, `turn_on_lights: true`
  (Bedroom Bed via Adaptive Lighting)
- **Turn off:** `light.turn_off` → `light.bedroom_lights`

### `switch.office_room`  (Template → switch, icon mdi:desk-lamp)
- **State:** `{{ is_state('light.office_lights','on') }}`
- **Turn on:** `light.turn_on` → `light.printerlamp`, `light.yeelight_strip6_0x13f31f78` (Desktop),
  `light.sofalampikea`, `light.hue_color_lamp_1` (Ball).  Office MAIN stays off.
- **Turn off:** `light.turn_off` → `light.office_lights`

## Light groups — Salon zones (Salon entity page)

Salon has ~15 lights; grouped into zones so they fit one page (Group → light).

| Group | Members |
|-------|---------|
| `light.salon_mood` | `light.mood_top`, `light.mood_mid`, `light.mood_bottom`, `light.moodlamp` |
| `light.salon_hue_tv` | `light.hue_play_1`, `light.hue_play_2`, `light.hue_play_right`, `light.hue_play_2_left` |
| `light.salon_gallery` | `light.shelly1_8caab561c0e6` (Gallery), `light.shelly1_e8db84a92cb1` (LED LR), `light.shelly1_e8db84a92cb1_2` (LED) |

## Screen wake on motion

- **NSPanel:** blueprint input `wake_up_sensors: [binary_sensor.lightsensorlivingroom_occupancy]`.
  Requires `include_action_wake_up: true` in the nspanel ESPHome substitutions.
- **M5Dial:** firmware subscribes to the same sensor (`binary_sensor` platform `homeassistant`)
  and calls `wake_now` on presence — see `m5dial/m5dial-salon.yaml`.
