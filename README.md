# SmartDisplay

Two wall-mounted smart controllers for the living room, with their designs, plans,
and as-built config kept as code.

## Devices
- **M5Dial (ESP32-S3)** — rotary + touch dial: Denon media (source-aware) + living-room
  climate (LG AC summer / Tado sync winter). Custom ESPHome + LVGL firmware.
- **SONOFF NSPanel** — home-summary panel (NSPanel Easy blueprint): bedroom/office light
  status + AC control, kitchen + living-room blinds on the hardware buttons. **Deployed & working.**

## Layout
```
docs/superpowers/specs/   design specs (M5Dial, NSPanel)
docs/superpowers/plans/   implementation plans (M5Dial firmware)
nspanel/                  NSPanel as-built config (blueprint automation + light groups)
m5dial/                   M5Dial ESPHome firmware (m5dial-salon.yaml + images/ + fonts/)
```

## Status
- NSPanel: ✅ done — see [`nspanel/`](nspanel/).
- M5Dial: ✅ done — firmware in [`m5dial/m5dial-salon.yaml`](m5dial/m5dial-salon.yaml).

## M5Dial controls

Two LVGL screens on the round display; **double-click** the front button switches between them.

**Media screen** (Denon, source-aware)
- Dial → Denon volume (`media_player.denon`), shown on the ring.
- Single press → play/pause Spotify/HEOS (`media_player.denon_2`).
- Long press → turn the AVR off.
- On-screen ▶/⏸ ⏮ ⏭ buttons for transport.
- Source artwork: Spotify, Apple TV apps (YouTube/Max/Netflix/Disney/Prime/SkyShowtime/Plex),
  PS5; plain "DENON" when the AVR is off.

**Climate screen** (summer = LG AC, winter = Tado)
- Season switches on `input_boolean.heating_season`.
- Dial → target temperature (0.5° step). Summer sets the AC; winter syncs both
  Tado thermostats (`climate.living_room` + `climate.kitchen`).
- Top status row: mode icon (recolors per hvac mode), fan-strength bars, swing-direction
  arrow, and a red power-off button. When off, a single big button turns the AC back on.
- Fan and swing open full-screen pickers; swing has independent vertical + horizontal toggles.

## Flashing
Built locally (not on the HA add-on, which OOMs):
```
cd SmartDisplay
esphome run m5dial/m5dial-salon.yaml   # pick OTA once it's on Wi-Fi, USB for first flash
```

> **Required once in HA:** enable *"Allow the device to perform Home Assistant actions"*
> on the M5Dial's ESPHome integration (Settings → Devices & Services → ESPHome →
> M5Dial Salon → Configure). Without it HA silently drops every control the device sends.

## Secrets
`m5dial/secrets.yaml` (Wi-Fi/API/OTA keys) is git-ignored. Secrets, HA `secrets.yaml`,
and `.storage` are never committed here.
