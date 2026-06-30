# NSPanel Salon — as-built config

SONOFF NSPanel running **NSPanel Easy** (`edwardtfn/NSPanel-Easy`), a home-summary
panel for the living room. Device `nspanel-salon` (ESP/TFT/blueprint = 2026.6.2).

Design + as-built details: [`../docs/superpowers/specs/2026-06-29-nspanel-home-summary-design.md`](../docs/superpowers/specs/2026-06-29-nspanel-home-summary-design.md)

## What's here
- `automation.nspanel_salon.yaml` — the blueprint automation (all panel config: tiles, hardware buttons, screensaver, yellow on-state).
- `light_groups.yaml` — the two per-room light groups the tiles use.

## Reproduce from scratch
1. **Firmware:** flash NSPanel Easy per https://edwardtfn.github.io/NSPanel-Easy/install.html (USB-TTL 3.3V serial first flash, then OTA + auto-TFT).
2. **Blueprint:** import NSPanel Easy blueprint into HA.
3. **Light groups:** create the two groups from `light_groups.yaml`.
4. **Automation:** create an automation from the blueprint and apply the inputs in `automation.nspanel_salon.yaml` — re-point `nspanel_name` to your device.
5. Applying input changes later: `automation.reload` + press `button.<device>_restart` (panel re-pulls layout on reboot).

## Behaviour
- Screensaver: clock/date/weather (`weather.home`) + panel temp.
- Tiles: Bedroom / AC Bedroom / Office / AC Office. Light tiles = any-on groups, **yellow when on**, tap = turn room off.
- Hardware buttons: left = kitchen tilt blinds (`cover.blind_tilt_ccd4`, tilt toggle), right = living-room shades (`cover.shadeslivingroom`).
