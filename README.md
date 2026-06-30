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
m5dial/                   M5Dial ESPHome firmware (added during implementation)
```

## Status
- NSPanel: ✅ done — see [`nspanel/`](nspanel/).
- M5Dial: 📄 spec + plan ready, firmware implementation pending — see [`docs/superpowers/plans/2026-06-30-m5dial-firmware.md`](docs/superpowers/plans/2026-06-30-m5dial-firmware.md).

Secrets (Wi-Fi/API keys, HA `secrets.yaml`, `.storage`) are never committed here.
