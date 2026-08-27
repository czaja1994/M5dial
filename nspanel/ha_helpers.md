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

## Light groups — entity-page tiles (Group → light)

Curated, not every bulb. Kitchen is merged into the Salon page.

| Group | Members |
|-------|---------|
| `light.evening_mood` | `light.map`, `light.moodlamp`, `light.sunset`, `light.yeelight_strip6_0x1582d560` (Countertop), `light.hue_play_1`, `light.hue_play_2`, `light.hue_play_right`, `light.hue_play_2_left`, `light.living_room` (HUE Strip) — the "Glitz and glam" set |
| `light.office_mood` | `light.yeelight_strip6_0x13f31f78` (Desktop), `light.hue_color_lamp_1` (Ball), `light.sofalampikea` (Sofa), `light.printerlamp` (Printer) |

Bedroom "Bed" tile uses the existing Hue group `light.bedroom_bed2` (driven by Adaptive
Lighting), so no extra group is needed there.

## Screen-off timeout (kept in sync with the M5Dial)

NSPanel device numbers (runtime, persisted on the panel — not in the blueprint):
`number.nspanel_salon_timeout_sleep = 30` (screen off), matching the M5Dial's
30s `backlight_idle`. `timeout_dimming` stays at 30; `display_brightness_sleep = 0`.

## Screen wake on motion

- **NSPanel:** blueprint input `wake_up_sensors: [binary_sensor.lightsensorlivingroom_occupancy]`.
  Requires `include_action_wake_up: true` in the nspanel ESPHome substitutions.
- **M5Dial:** firmware subscribes to the same sensor (`binary_sensor` platform `homeassistant`)
  and calls `wake_now` on presence — see `m5dial/m5dial-salon.yaml`.
