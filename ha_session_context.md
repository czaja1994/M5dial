# Home Assistant — kontekst sesji (dla Claude Code)

*Setup: Mateusz, instancja HA pod `192.168.1.228:8123`, zarządzana zdalnie przez VPN + MCP server.*

## Cel sesji
Naprawa integracji po migracji TADO → tado_ce, budowa nowego mobilnego dashboardu `🏠 Home` (`url_path: mobile-home`), stylizacja motywem Caule Black Rose.

---

## Stan integracji (naprawione w tej sesji)

### TADO
- Stara integracja `tado` (setup_error, pętla re-auth, zjadała 100 req/dzień) — **usunięta**.
- `tado_ce` (HACS, community edition) miała `migration_error` — **usunięta i dodana od nowa**, teraz `loaded`.
- Po reinstalacji strefy climate dostały sufiksy (`climate.bedroom_bedroom`); **przemianowane z powrotem** na czyste ID:
  - `climate.bedroom`, `climate.kitchen`, `climate.living_room`
- Sensory API tado_ce mają teraz prefix `sensor.tado_ce_hub_*` (wcześniej `sensor.tado_ce_*`).
- HomeKit lokalny odczyt: V3+ wspiera, omija limit API. (Bridge dodany w HomeKit.)

### Pogoda
- **AccuWeather usunięty** (płatne API od 2026).
- **Open-Meteo dodany** (darmowy, bez limitu) → encja `weather.home`.
- Odtworzone template sensory z ORYGINALNYMI nazwami (używane w automatyzacjach):
  - `sensor.prandoty_outdoor_temperature` → `{{ state_attr('weather.home','temperature') }}`
  - `sensor.prandoty_weather_condition` → `{{ states('weather.home') }}`

### Automatyzacja "Bedroom Morning AC" (id 1751843763536)
- Miała martwe device_id (ghost devices po starym tado): `f44d8dc9313e241d131fc88ee0a39156`, `da500e7c4124540ccd74c4f847e4307f`.
- **Naprawione** — zamienione na entity conditions:
  - `sensor.bedroom_bedroom_temperature > 17`
  - `sensor.living_room_living_room_temperature > 23`
  - + `device_tracker.czajas_iphone` = home, `binary_sensor.windowsensorbedroom_contact` = off
- Akcja: `climate.bedroom_2` → cool 22°C + preset windFree.

### Better Thermostat (HACS) — DO ZROBIENIA RĘCZNIE
Trzy instancje, wszystkie `loaded` ale 2 używają martwych sensorów tado.
MCP **nie może** edytować options flow zewnętrznych integracji HACS — trzeba w UI:
- **LivingRoomThermostatBT** (`01JNP77RSWMFX2JNRGBSHF3DJV`): temp sensor `sensor.living_room_temperature` (unknown) → zmień na `sensor.living_room_living_room_temperature`
- **KitchenThermostatBT** (`01JNPDAKCCDWHQG2KS3C8215CZ`): temp sensor `sensor.kitchen_temperature` (unknown) → zmień na `sensor.kitchen_kitchen_temperature`
- **OfficeBetterThermo** (`01JQEYG0FQM3FX8AEAQQVVSYSB`): `sensor.officetemperaturesensor_temperature` — OK, działa.

---

## Helper utworzony
- `input_boolean.heating_season` — `off` = lato (pokazuje AC), `on` = sezon grzewczy (pokazuje TADO). Obecnie `off` (czerwiec).

## Skrypty utworzone (shortcuts dzień/wieczór)
Logika "co włączyć" zależnie od pory — dzień vs >1h przed zachodem. Wołane z dashboardu przez `script.turn_on` (template w `target.entity_id` NIE działa, stąd skrypty):
- `script.shortcut_living_kitchen` — dzień: countertop+LED LR / wieczór: Countertop Color+MoodLamp+Sunset
- `script.shortcut_office` — dzień: MAIN / wieczór: Desktop+Ball+SofaLampIkea+PrinterLamp
- `script.shortcut_bedroom` — dzień: MAIN / wieczór: Bedroom Bed

Warunek pory dnia (w skryptach, `condition` blok):
```yaml
- condition: state
  entity_id: sun.sun
  state: above_horizon
- condition: template
  value_template: "{{ (as_timestamp(states('sensor.sun_next_setting')) - as_timestamp(now())) > 3600 }}"
```

---

## Dashboard `🏠 Home` (url_path: mobile-home)
Mobilny, 6 widoków (`type: sections`, `max_columns: 2`):

1. **Home** — sekcje: weather-forecast (animowane ikony) + chips bar (pogoda/osoby/alarm/Roborock), media (conditional media-control gdy gra), shortcuts (3 kafelki per pokój), Climate (conditional wg heating_season), Blinds, Security.
2. **Lights** — 3 duże kafelki 1-kolumnowe (Living+Kitchen, Bedroom, Office) → Bubble Card pop-upy z mushroom-light-card w gridzie 2-kol.
3. **Climate** — heating/AC conditional + TADO API monitor.
4. **Media** — Denon, Apple TV, Spotify, Bedroom TV, HomePod.
5. **Cameras** — NVR (hall/living/office) + peephole doorbell.
6. **Tech** — UPS, Rack (temp+CPU/RAM mini-graph+fany), Bambu P1S (conditional gdy drukuje), Scale.

### Person chips
Pokazują imiona (Mateusz/Ala) zamiast home/away, ikona zielona gdy `home`, szara gdy nie.

### Media widget
`type: media-control` (NIE mushroom/tile) dla `media_player.apple_tv` i `media_player.spotify_mateusz_czajowski`, owinięte w `conditional` (state playing/paused).

### Climate karty
- `tap_action: more-info` (tapnięcie NIE toggluje, otwiera panel).
- AC hvac_modes: tylko `off/cool/auto` (wyrzucone heat/dry/fan_only).
- Bedroom AC + Office AC: feature `climate-preset-modes` (windFree, preset_modes: none/sleep/boost/wind_free/wind_free_sleep).
- Living Room AC (`climate.air_conditioner`): brak preset_modes.

### Security — potwierdzenia
- `lock.audi_s3_limousine_door_lock`: `tap_action: toggle` + `confirmation: "Lock / unlock Audi S3?"`
- alarm `alarm_control_panel.aqara_hub_m1s_272b_security_system`: `more-info` + `confirmation: "Change alarm state?"`
- (PIN NIE jest zapisany w configu — odrzucone ze względów bezpieczeństwa; alarm przez HomeKit nie ma natywnego pola kodu.)

---

## Motyw Caule Black Rose (caule-themes-pack-1, HACS)
- Ustawiony `Caule Black Rose Glass` przez `frontend.set_theme`.
- **PROBLEM**: wygląda flat/szaro, nie jak "Transparent" z repo.
- **Przyczyny**:
  1. Brak `frontend: themes: !include_dir_merge_named themes` w `configuration.yaml` — **user dodaje ręcznie**, potem restart.
  2. Brak plików tła/ikon — trzeba pobrać z https://bit.ly/38G6ptj i wrzucić folder do `config/www/caule-themes-pack-1/`.
  3. Moje card-mod styles mogą walczyć z motywem.
- Profil użytkownika powinien mieć temat ustawiony na **"Backend-selected"**.

### Glow na kartach (card-mod)
Używać `--ha-card-box-shadow` CSS variable (NIE `box-shadow !important` — Mushroom/temat to nadpisuje):
```
ha-card { --ha-card-box-shadow: {{ '0 0 0 1px rgba(R,G,B,0.6), 0 0 20px rgba(R,G,B,0.3)' if (warunek) else 'none' }}; transition: --ha-card-box-shadow 0.4s ease; }
```
Kolory: amber `245,158,11` (Living+Kitchen), purple `139,92,246` (Bedroom), blue `59,130,246` (Office), cyan `6,182,212` (Climate/fany), green `16,185,129` (shortcut Kitchen+Living).

### Bubble Card pop-up — szare tło fix
Bubble Card ma własne CSS vars:
```
:host {
  --bubble-pop-up-background-color: rgba(10,10,14,0.97) !important;
  --bubble-main-background-color: rgba(10,10,14,0.97) !important;
}
```

---

## KLUCZOWE zasady techniczne (MCP `ha_config_set_dashboard` + `python_transform`)
- **Brak `def`** — walidator odrzuca FunctionDef. Używać `lambda` + list comprehension.
- **Brak `import`**.
- `config_hash` trzeba pobrać świeży przez `ha_config_get_dashboard` przed każdym zapisem.
- Duże configi: edytować przez `python_transform` z index-based path traversal, nie full replace.

## Lovelace gotchas (potwierdzone w tej sesji)
- `mushroom-scene-card` **NIE istnieje** w Mushroom v5.1.1 → użyć `mushroom-legacy-template-card` z `tap_action: scene.turn_on`.
- `conditional` card w sekcjach: tylko `condition: state` / `condition: numeric_state`. **`value_template` NIE działa** ("Visibility status unknown"). Dla wielu stanów (playing+paused) → osobne `conditional` karty.
- `vertical-stack` wewnątrz `conditional` z mushroom-media-player → configuration error. Karty bezpośrednio w sekcji.
- Template w `tap_action target.entity_id` **NIE działa** → użyć skryptów HA.
- `which: above_horizon` NIE działa w skryptach — użyć `condition: state` na `sun.sun`.
- `tod` helper nie przyjmuje sunrise/sunset, tylko stałe HH:MM.
- Styling: bez `border-left: 3px` (side-stripe). Uniform `1px solid rgba(255,255,255,0.07)` border, glow ring tylko dla aktywnego stanu.

## Urządzenia offline (świadome, nie błąd)
- Shelly bar + countertop — brak prądu, reset ręczny po powrocie.
- `switch.playstation_5_power` — celowo odcięty (smart plug, fix CEC wake od Apple TV).
- Fire TV — nieużywane.

## Reference
- `MainOversight` (`dashboard-mainoversight`) — kanoniczne źródło entity_id i nazw urządzeń.
- Wszystkie etykiety po angielsku, nazwy świateł 1:1 z friendly_name z HA.
