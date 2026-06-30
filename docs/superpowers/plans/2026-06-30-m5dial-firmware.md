# M5Dial Firmware Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build ESPHome + LVGL firmware for the M5Dial (ESP32-S3) that controls Denon media (source-aware) and living-room climate (LG AC summer / Tado sync winter) from one rotary + touch dial.

**Architecture:** On-device LVGL UI with two pages (MEDIA, CLIMATE). Home Assistant is the single source of truth: `homeassistant` text/numeric sensors pull entity state into the device; the rotary encoder, the front button (single/double/long press) and on-screen touch buttons fire `homeassistant.action` service calls. Displayed values follow the HA entity echo, so UI never drifts from HA.

**Tech Stack:** ESPHome (esp-idf framework), LVGL component, `ili9xxx` (GC9A01A) display, `ft5x06` touchscreen, `rotary_encoder`, GPIO `binary_sensor` with `on_multi_click`, `homeassistant` native API.

## Global Constraints

- Board: `esp32` `variant: esp32s3`, `framework: type: esp-idf`. (ESPHome 2025.x+.)
- **Verified M5Dial pinout** (from devices.esphome.io/devices/m5stack-dial): SPI `mosi GPIO5` / `clk GPIO6`; display GC9A01A `cs GPIO7` / `reset GPIO8` / `dc GPIO4`; backlight `GPIO9`; I2C `sda GPIO11` / `scl GPIO12`; touch `ft5x06 @0x38`; rotary encoder `pin_a GPIO40` / `pin_b GPIO41`; front button `GPIO42`; buzzer `GPIO3` (unused); RTC PCF8563 `@0x51` (unused).
- **Touch controller caveat:** official ESPHome M5Dial = `ft5x06`; the spec assumed `cst816`. Task 3 includes an I2C-scan verification and a one-line fallback. Use whatever the scan reports.
- HA entities (exact, confirmed in instance): `media_player.denon` (AVR: master volume, `source`, power — NO transport/metadata), `media_player.denon_2` (HEOS: `media_title`/`media_artist`, transport), `media_player.apple_tv`, `climate.air_conditioner` (LG; off/cool/auto, temp 18–30 step 0.5, fan `nature/low/medium/high`, swing vertical on/off), `climate.living_room` + `climate.kitchen` (Tado), `input_boolean.heating_season` (off=summer/AC, on=winter/Tado).
- Fan display mapping (label → HA value): `auto`→`nature`, `low`→`low`, `med`→`medium`, `high`→`high`.
- Encoder: volume step ~2% (MEDIA), temp step 0.5 °C (CLIMATE). Debounce sends (~150 ms / on movement end) so AVR/Tado aren't flooded.
- Single source of truth = HA. No album art (text only). Out of scope: RFID/RTC/buzzer, horizontal swing, AC modes beyond off/cool/auto.
- Flashing: M5Dial has native USB-C → flash via `esphome run` over USB the first time, OTA after. Wall-mounted, USB-powered (no battery power-hold needed).
- All config lives in one device file `m5dial-salon.yaml` + `secrets.yaml` (ESPHome single-device-file convention; avoids `lvgl:`/package list-merge pitfalls). Each task appends/edits clearly-commented sections of that file.

---

## File Structure

- Create: `m5dial/m5dial-salon.yaml` — the entire device config, built up section by section across tasks. Section order in the file: `substitutions` → `esphome`/`esp32` → `logger`/`wifi`/`api`/`ota`/`time` → `spi`/`i2c` → `output`/`light` (backlight) → `display` → `touchscreen` → `sensor` (encoder) → `binary_sensor` (button) → `globals` → `homeassistant` text/numeric sensors → `script` → `lvgl`.
- Create: `m5dial/secrets.yaml` — `wifi_ssid`, `wifi_password`, `api_key`, `ota_password` (NOT committed; add to `.gitignore`).
- Create: `m5dial/README.md` — flash + OTA instructions, pin reference, HA entity list.
- HA side (created in Task 7 via MCP/UI): `script.m5dial_denon_spotify_resume`.

All verification uses the ESPHome CLI (`esphome config|compile|run|logs m5dial/m5dial-salon.yaml`) plus on-device/HA observation. There is no pytest layer — the "test" for each task is: config validates, compile succeeds, and the named on-device/HA behavior is observed in `esphome logs` or the HA UI.

---

## Task 1: Project skeleton + HA connectivity

**Files:**
- Create: `m5dial/m5dial-salon.yaml`
- Create: `m5dial/secrets.yaml`
- Create: `m5dial/.gitignore`

**Interfaces:**
- Produces: a flashable ESPHome device `m5dial-salon` that connects to Wi-Fi + HA native API. Later tasks add components to the same file.

- [ ] **Step 1: Write `m5dial/secrets.yaml`** (fill real values; generate api_key with `openssl rand -base64 32`)

```yaml
wifi_ssid: "Czaja_IoT"
wifi_password: "REPLACE_ME"
api_key: "REPLACE_WITH_openssl_rand_base64_32"
ota_password: "REPLACE_ME"
```

- [ ] **Step 2: Write `m5dial/.gitignore`**

```gitignore
secrets.yaml
.esphome/
*.bin
```

- [ ] **Step 3: Write `m5dial/m5dial-salon.yaml` skeleton**

```yaml
substitutions:
  name: m5dial-salon
  friendly_name: M5Dial Salon

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

esp32:
  variant: esp32s3
  framework:
    type: esp-idf

logger:
  level: INFO

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

time:
  - platform: homeassistant
    id: ha_time
```

- [ ] **Step 4: Validate config**

Run: `esphome config m5dial/m5dial-salon.yaml`
Expected: `INFO Configuration is valid!`

- [ ] **Step 5: Flash over USB** (M5Dial connected by USB-C)

Run: `esphome run m5dial/m5dial-salon.yaml`
Expected: compile + upload succeed; pick the USB serial port; logs show `WiFi Connected` and `API ... connected`.

- [ ] **Step 6: Verify in HA**

Settings → Devices & Services → ESPHome → the new `m5dial-salon` is discovered/online (enter the api_key if prompted).
Expected: device shows online, no entities yet (fine).

- [ ] **Step 7: Commit**

```bash
git add m5dial/m5dial-salon.yaml m5dial/.gitignore
git commit -m "feat(m5dial): ESPHome skeleton + HA connectivity"
```

---

## Task 2: Display + backlight + LVGL hello

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (add `spi`, `output`, `light`, `display`, minimal `lvgl`)

**Interfaces:**
- Produces: `id(backlight)` light (dimmable, GPIO9); `id(round_display)` display; an `lvgl:` block rendering on it. Later tasks add pages/widgets under `lvgl:`.

- [ ] **Step 1: Add SPI, backlight, display, and a one-label LVGL block** (append to the file)

```yaml
spi:
  id: spi_bus
  mosi_pin: GPIO5
  clk_pin: GPIO6

output:
  - platform: ledc
    pin: GPIO9
    id: backlight_output

light:
  - platform: monochromatic
    output: backlight_output
    id: backlight
    name: Backlight
    restore_mode: ALWAYS_ON
    default_transition_length: 250ms

display:
  - platform: ili9xxx
    id: round_display
    model: GC9A01A
    cs_pin: GPIO7
    dc_pin: GPIO4
    reset_pin: GPIO8
    invert_colors: true
    auto_clear_enabled: false
    update_interval: never

lvgl:
  displays:
    - round_display
  bg_color: 0x000000
  pages:
    - id: media_page
      widgets:
        - label:
            align: CENTER
            text: "M5Dial OK"
            text_color: 0xFFFFFF
```

- [ ] **Step 2: Validate config**

Run: `esphome config m5dial/m5dial-salon.yaml`
Expected: `Configuration is valid!`

- [ ] **Step 3: Compile + flash**

Run: `esphome run m5dial/m5dial-salon.yaml`
Expected: build OK, upload OK.

- [ ] **Step 4: Verify on device**

Expected: screen backlight on; round display shows centered white text "M5Dial OK". If colors look inverted/wrong, toggle `invert_colors` (known GC9A01A quirk) and re-flash.

- [ ] **Step 5: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): display + backlight + LVGL hello"
```

---

## Task 3: Inputs — touch, encoder, button (single/double/long)

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (add `i2c`, `touchscreen`, encoder `sensor`, button `binary_sensor`, link touch to `lvgl`)

**Interfaces:**
- Produces: `id(encoder)` rotary sensor (fires `on_clockwise`/`on_anticlockwise`), `id(front_button)` binary_sensor (fires single/double/long via `on_multi_click`), touch driving LVGL buttons. These are wired to real actions in later tasks; here they only log.

- [ ] **Step 1: Add I2C and scan to confirm the touch controller**

Add:

```yaml
i2c:
  - id: internal_i2c
    sda: GPIO11
    scl: GPIO12
    scan: true
```

Run: `esphome run m5dial/m5dial-salon.yaml` then `esphome logs m5dial/m5dial-salon.yaml`
Expected: I2C scan lists `0x38` (FT5x06) and `0x51` (RTC). **If `0x38` is absent but `0x15` is present, the unit is CST816** — in Step 2 use `platform: cst816` instead of `ft5x06` (same `i2c_id`, no address needed).

- [ ] **Step 2: Add touchscreen, encoder, button; link touch to LVGL**

Add:

```yaml
touchscreen:
  - platform: ft5x06        # if scan showed CST816, use: platform: cst816
    id: my_touch
    i2c_id: internal_i2c

sensor:
  - platform: rotary_encoder
    id: encoder
    pin_a: GPIO40
    pin_b: GPIO41
    on_clockwise:
      - logger.log: "encoder CW"
    on_anticlockwise:
      - logger.log: "encoder CCW"

binary_sensor:
  - platform: gpio
    id: front_button
    pin:
      number: GPIO42
      inverted: true
      mode:
        input: true
        pullup: true
    on_multi_click:
      # long press: held >= 600ms
      - timing:
          - ON for at least 600ms
        then:
          - logger.log: "btn LONG"
      # double press
      - timing:
          - ON for at most 400ms
          - OFF for at most 400ms
          - ON for at most 400ms
          - OFF for at least 50ms
        then:
          - logger.log: "btn DOUBLE"
      # single press
      - timing:
          - ON for at most 400ms
          - OFF for at least 400ms
        then:
          - logger.log: "btn SINGLE"
```

Then add the touch link inside the existing `lvgl:` block (add this key alongside `displays:`):

```yaml
  touchscreens:
    - my_touch
```

- [ ] **Step 3: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; then `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 4: Verify on device via logs**

Run: `esphome logs m5dial/m5dial-salon.yaml`
Expected: rotating the bezel logs `encoder CW`/`encoder CCW`; pressing the dial logs `btn SINGLE`; two quick presses log `btn DOUBLE`; holding ~1s logs `btn LONG`. Touching the screen logs touchscreen activity at DEBUG.

- [ ] **Step 5: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): touch + encoder + multi-click button"
```

---

## Task 4: Home Assistant entity ingestion

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (add `homeassistant` text/numeric sensors + `globals`)

**Interfaces:**
- Produces text_sensors `ha_media_title`, `ha_media_artist`, `ha_heos_state`, `ha_avr_source`, `ha_avr_state`, `ha_appletv_title`, `ha_appletv_app`, `ha_ac_state`, `ha_ac_fan`, `ha_ac_swing`, `ha_season`; numeric sensors `ha_volume`, `ha_ac_target`, `ha_ac_current`, `ha_tado_target`. These feed the LVGL update scripts in Tasks 6 and 8.

- [ ] **Step 1: Add the `homeassistant` platform sensors**

```yaml
text_sensor:
  - platform: homeassistant
    id: ha_media_title
    entity_id: media_player.denon_2
    attribute: media_title
  - platform: homeassistant
    id: ha_media_artist
    entity_id: media_player.denon_2
    attribute: media_artist
  - platform: homeassistant
    id: ha_heos_state
    entity_id: media_player.denon_2
  - platform: homeassistant
    id: ha_avr_source
    entity_id: media_player.denon
    attribute: source
  - platform: homeassistant
    id: ha_avr_state
    entity_id: media_player.denon
  - platform: homeassistant
    id: ha_appletv_title
    entity_id: media_player.apple_tv
    attribute: media_title
  - platform: homeassistant
    id: ha_appletv_app
    entity_id: media_player.apple_tv
    attribute: app_name
  - platform: homeassistant
    id: ha_ac_state
    entity_id: climate.air_conditioner
  - platform: homeassistant
    id: ha_ac_fan
    entity_id: climate.air_conditioner
    attribute: fan_mode
  - platform: homeassistant
    id: ha_ac_swing
    entity_id: climate.air_conditioner
    attribute: swing_mode
  - platform: homeassistant
    id: ha_season
    entity_id: input_boolean.heating_season

sensor:
  - platform: homeassistant
    id: ha_volume
    entity_id: media_player.denon
    attribute: volume_level
  - platform: homeassistant
    id: ha_ac_target
    entity_id: climate.air_conditioner
    attribute: temperature
  - platform: homeassistant
    id: ha_ac_current
    entity_id: climate.air_conditioner
    attribute: current_temperature
  - platform: homeassistant
    id: ha_tado_target
    entity_id: climate.living_room
    attribute: temperature
```

(The existing `sensor:` block from Task 3 already contains the encoder — append these entries under the same `sensor:` key. Same for `text_sensor:` being introduced here.)

- [ ] **Step 2: Add globals for screen/state bookkeeping**

```yaml
globals:
  - id: current_screen          # 0 = media, 1 = climate
    type: int
    restore_value: false
    initial_value: "0"
  - id: pending_volume          # debounced target volume 0..1, -1 = idle
    type: float
    restore_value: false
    initial_value: "-1"
  - id: pending_temp            # debounced target temp, -1 = idle
    type: float
    restore_value: false
    initial_value: "-1"
```

- [ ] **Step 3: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 4: Verify ingestion via logs**

Run: `esphome logs m5dial/m5dial-salon.yaml`
Expected: on connect, lines like `'ha_avr_source': Sending state 'HEOS Music'`, `'ha_volume': Sending state 0.38`. Change AC target in HA → `ha_ac_target` logs the new value. Confirms the echo path works.

- [ ] **Step 5: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): ingest HA entity state (media/climate/season)"
```

---

## Task 5: Two LVGL pages + navigation + idle

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (replace the placeholder `lvgl.pages` with two real pages; add navigation scripts; wire double-press + idle)

**Interfaces:**
- Produces: `lvgl` pages `media_page` and `climate_page`; scripts `nav_to_media`, `nav_to_climate`, `nav_toggle`; an idle timeout that returns CLIMATE→MEDIA after 15 s and dims the backlight after 60 s. Later tasks fill each page's widgets and the per-screen press/encoder dispatch.

- [ ] **Step 1: Replace the `lvgl.pages` list with two real (still sparse) pages**

```yaml
  pages:
    - id: media_page
      widgets:
        - label:
            id: lbl_media_placeholder
            align: CENTER
            text: "MEDIA"
            text_color: 0xFFFFFF
    - id: climate_page
      widgets:
        - label:
            id: lbl_climate_placeholder
            align: CENTER
            text: "CLIMATE"
            text_color: 0xFFFFFF
```

Also add an LVGL-level idle handler key (alongside `displays:`/`touchscreens:`):

```yaml
  on_idle:
    timeout: 60s
    then:
      - light.turn_on:
          id: backlight
          brightness: 20%
```

- [ ] **Step 2: Add navigation scripts**

```yaml
script:
  - id: nav_to_media
    then:
      - lvgl.page.show: media_page
      - lambda: 'id(current_screen) = 0;'
      - light.turn_on: { id: backlight, brightness: 100% }
  - id: nav_to_climate
    then:
      - lvgl.page.show: climate_page
      - lambda: 'id(current_screen) = 1;'
      - light.turn_on: { id: backlight, brightness: 100% }
  - id: nav_toggle
    then:
      - if:
          condition:
            lambda: 'return id(current_screen) == 0;'
          then:
            - script.execute: nav_to_climate
          else:
            - script.execute: nav_to_media
  - id: climate_idle_return     # called by a 15s timer started on entering CLIMATE
    mode: restart
    then:
      - delay: 15s
      - script.execute: nav_to_media
```

- [ ] **Step 3: Wire double-press to navigation and start/stop the idle timer**

In `binary_sensor.front_button.on_multi_click`, replace the DOUBLE handler body:

```yaml
        then:
          - script.execute: nav_toggle
          - if:
              condition:
                lambda: 'return id(current_screen) == 1;'
              then:
                - script.execute: climate_idle_return
              else:
                - script.stop: climate_idle_return
```

Also wake the backlight on ANY input — add to the SINGLE and LONG handlers and to the encoder `on_clockwise`/`on_anticlockwise`:

```yaml
          - light.turn_on: { id: backlight, brightness: 100% }
```

- [ ] **Step 4: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 5: Verify on device**

Expected: boots on MEDIA page. Double-press → CLIMATE; double-press → MEDIA. On CLIMATE, after 15 s with no input it returns to MEDIA. After 60 s idle the backlight dims; any encoder turn/press restores full brightness.

- [ ] **Step 6: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): two LVGL pages + nav + idle/dim"
```

---

## Task 6: MEDIA screen UI + source router

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (build `media_page` widgets; add `update_media` script; trigger it from the relevant text_sensors)

**Interfaces:**
- Consumes: `ha_avr_state`, `ha_avr_source`, `ha_heos_state`, `ha_media_title`, `ha_media_artist`, `ha_appletv_title`, `ha_appletv_app`, `ha_volume` (Task 4).
- Produces: widgets `m_top` (source label), `m_title`, `m_artist`, `m_btn_play`, `m_btn_prev`, `m_btn_next`, `m_vol_arc`, `m_vol_lbl`, `m_logo`; script `update_media` that applies the source-router visibility rules. Touch handlers on prev/next/play are added in Task 7.

- [ ] **Step 1: Replace `media_page.widgets` with the real widget tree**

```yaml
    - id: media_page
      widgets:
        - label:
            id: m_top
            align: TOP_MID
            y: 26
            text: "HEOS · Spotify"
            text_color: 0x9AA0A6
        - label:
            id: m_title
            align: CENTER
            y: -28
            text_align: CENTER
            text_font: montserrat_20
            text_color: 0xFFFFFF
            text: "—"
        - label:
            id: m_artist
            align: CENTER
            y: 2
            text_align: CENTER
            text_color: 0xB0B5BA
            text: ""
        - label:
            id: m_logo
            align: CENTER
            y: -10
            text_font: montserrat_24
            text_color: 0xFFFFFF
            text: "DENON"
            hidden: true
        - button:
            id: m_btn_prev
            align: BOTTOM_LEFT
            x: 36
            y: -36
            width: 48
            height: 48
            widgets:
              - label: { align: CENTER, text: "", text_font: montserrat_20 }
        - button:
            id: m_btn_play
            align: CENTER
            y: 56
            width: 56
            height: 56
            widgets:
              - label: { id: m_play_icon, align: CENTER, text: "", text_font: montserrat_20 }
        - button:
            id: m_btn_next
            align: BOTTOM_RIGHT
            x: -36
            y: -36
            width: 48
            height: 48
            widgets:
              - label: { align: CENTER, text: "", text_font: montserrat_20 }
        - arc:
            id: m_vol_arc
            align: CENTER
            width: 230
            height: 230
            start_angle: 130
            end_angle: 50
            min_value: 0
            max_value: 100
            value: 0
            arc_color: 0x202020
            indicator:
              arc_color: 0x1D9E75
        - label:
            id: m_vol_lbl
            align: BOTTOM_MID
            y: -8
            text_color: 0xFFFFFF
            text: "Vol —"
```

Add the FontAwesome glyphs by declaring a font with those codepoints (append a `font:` block; ``=prev,``=play,``=pause,``=next):

```yaml
font:
  - file: "gfonts://Montserrat"
    id: montserrat_20
    size: 20
  - file: "gfonts://Montserrat"
    id: montserrat_24
    size: 24
  - file: { type: web, url: "https://github.com/FortAwesome/Font-Awesome/raw/6.x/webfonts/fa-solid-900.ttf" }
    id: fa_icons
    size: 20
    glyphs: ["", "", "", "", ""]
```

(Use `text_font: fa_icons` on the icon labels above instead of `montserrat_20`.)

- [ ] **Step 2: Add the `update_media` router script**

```yaml
  - id: update_media
    then:
      - lambda: |-
          std::string avr = id(ha_avr_state).state;
          std::string src = id(ha_avr_source).state;
          std::string heos = id(ha_heos_state).state;
          bool off = (avr == "off" || avr == "unavailable" || avr == "");
          bool appletv = (src == "APPLE TV");
          bool ps5 = (src == "PlayStation 5");
          bool heos_active = (heos == "playing" || heos == "paused");
          // default: hide everything optional
          lv_obj_add_flag((lv_obj_t*)id(m_logo), LV_OBJ_FLAG_HIDDEN);
          lv_obj_clear_flag((lv_obj_t*)id(m_title), LV_OBJ_FLAG_HIDDEN);
          lv_obj_clear_flag((lv_obj_t*)id(m_artist), LV_OBJ_FLAG_HIDDEN);
          auto show = [](lv_obj_t* o, bool v){ if (v) lv_obj_clear_flag(o, LV_OBJ_FLAG_HIDDEN); else lv_obj_add_flag(o, LV_OBJ_FLAG_HIDDEN); };
          bool transport = heos_active && !off && !appletv && !ps5;
          show((lv_obj_t*)id(m_btn_play), transport);
          show((lv_obj_t*)id(m_btn_prev), transport);
          show((lv_obj_t*)id(m_btn_next), transport);
          if (off) {
            lv_label_set_text((lv_obj_t*)id(m_top), "Denon");
            lv_label_set_text((lv_obj_t*)id(m_logo), "DENON  ");
            show((lv_obj_t*)id(m_logo), true);
            show((lv_obj_t*)id(m_title), false);
            show((lv_obj_t*)id(m_artist), false);
          } else if (appletv) {
            lv_label_set_text((lv_obj_t*)id(m_top), id(ha_appletv_app).state.c_str());
            lv_label_set_text((lv_obj_t*)id(m_title), id(ha_appletv_title).state.c_str());
            lv_label_set_text((lv_obj_t*)id(m_artist), "");
          } else if (ps5) {
            lv_label_set_text((lv_obj_t*)id(m_top), "PlayStation 5");
            lv_label_set_text((lv_obj_t*)id(m_logo), "PS5");
            show((lv_obj_t*)id(m_logo), true);
            show((lv_obj_t*)id(m_title), false);
            show((lv_obj_t*)id(m_artist), false);
          } else if (heos_active) {
            lv_label_set_text((lv_obj_t*)id(m_top), "HEOS · Spotify");
            lv_label_set_text((lv_obj_t*)id(m_title), id(ha_media_title).state.c_str());
            lv_label_set_text((lv_obj_t*)id(m_artist), id(ha_media_artist).state.c_str());
          } else {
            lv_label_set_text((lv_obj_t*)id(m_top), src.c_str());
            lv_label_set_text((lv_obj_t*)id(m_title), "Nic nie gra");
            lv_label_set_text((lv_obj_t*)id(m_artist), "");
          }
          // volume arc + label (echo)
          float v = id(ha_volume).has_state() ? id(ha_volume).state : -1.0f;
          if (v >= 0) {
            int pct = (int) roundf(v * 100.0f);
            lv_arc_set_value((lv_obj_t*)id(m_vol_arc), pct);
            char buf[16]; snprintf(buf, sizeof(buf), "Vol %d%%", pct);
            lv_label_set_text((lv_obj_t*)id(m_vol_lbl), buf);
          } else {
            lv_label_set_text((lv_obj_t*)id(m_vol_lbl), "Vol —");
          }
```

- [ ] **Step 3: Trigger `update_media` whenever a source value changes**

Add `on_value: - script.execute: update_media` to each of: `ha_avr_state`, `ha_avr_source`, `ha_heos_state`, `ha_media_title`, `ha_media_artist`, `ha_appletv_title`, `ha_appletv_app`, `ha_volume`. Example:

```yaml
  - platform: homeassistant
    id: ha_avr_source
    entity_id: media_player.denon
    attribute: source
    on_value:
      - script.execute: update_media
```

- [ ] **Step 4: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 5: Verify on device**

Expected: with HEOS playing, title+artist show and the play/prev/next buttons are visible; the volume arc tracks `media_player.denon` volume. In HA, switch the AVR source to APPLE TV → screen shows the Apple TV title (no transport buttons); to PlayStation 5 → "PS5" logo; turn the AVR off → "DENON ⏻" logo. (Widget coordinates may need small nudges on-device — adjust `x/y` and re-flash.)

- [ ] **Step 6: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): MEDIA screen UI + AVR source router"
```

---

## Task 7: MEDIA actions + Spotify-resume HA script

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (encoder→volume, press dispatch, touch transport, debounce)
- Create (HA side): `script.m5dial_denon_spotify_resume`

**Interfaces:**
- Consumes: `update_media`, the button `on_multi_click` SINGLE/LONG handlers, encoder events, `ha_volume`, `ha_avr_state`/`source`/`heos_state`.
- Produces: scripts `media_press`, `media_long`, `volume_apply`. After this task the MEDIA screen is fully interactive.

- [ ] **Step 1: Create the HA script (via MCP `ha_config_set_script` or HA UI)**

`script.m5dial_denon_spotify_resume` sequence: turn on AVR, select a network source, play HEOS. YAML:

```yaml
alias: M5Dial Denon Spotify Resume
sequence:
  - action: media_player.turn_on
    target: { entity_id: media_player.denon }
  - delay: { seconds: 2 }
  - action: media_player.select_source
    target: { entity_id: media_player.denon }
    data: { source: "HEOS Music" }   # tune to "NET" if Spotify Connect needs it
  - action: media_player.media_play
    target: { entity_id: media_player.denon_2 }
mode: single
```

- [ ] **Step 2: Add the encoder→volume debounce (MEDIA only)**

Replace the encoder `on_clockwise`/`on_anticlockwise` bodies:

```yaml
    on_clockwise:
      - light.turn_on: { id: backlight, brightness: 100% }
      - if:
          condition: { lambda: 'return id(current_screen) == 0;' }
          then:
            - lambda: |-
                float base = (id(pending_volume) >= 0) ? id(pending_volume)
                            : (id(ha_volume).has_state() ? id(ha_volume).state : 0.0f);
                float v = base + 0.02f; if (v > 1.0f) v = 1.0f;
                id(pending_volume) = v;
                int pct = (int) roundf(v*100.0f);
                lv_arc_set_value((lv_obj_t*)id(m_vol_arc), pct);
                char buf[16]; snprintf(buf,sizeof(buf),"Vol %d%%",pct);
                lv_label_set_text((lv_obj_t*)id(m_vol_lbl), buf);
            - script.execute: volume_apply
          else:
            - script.execute: climate_temp_up      # defined in Task 9
    on_anticlockwise:
      - light.turn_on: { id: backlight, brightness: 100% }
      - if:
          condition: { lambda: 'return id(current_screen) == 0;' }
          then:
            - lambda: |-
                float base = (id(pending_volume) >= 0) ? id(pending_volume)
                            : (id(ha_volume).has_state() ? id(ha_volume).state : 0.0f);
                float v = base - 0.02f; if (v < 0.0f) v = 0.0f;
                id(pending_volume) = v;
                int pct = (int) roundf(v*100.0f);
                lv_arc_set_value((lv_obj_t*)id(m_vol_arc), pct);
                char buf[16]; snprintf(buf,sizeof(buf),"Vol %d%%",pct);
                lv_label_set_text((lv_obj_t*)id(m_vol_lbl), buf);
            - script.execute: volume_apply
          else:
            - script.execute: climate_temp_down     # defined in Task 9
```

Add the debounced apply script (sends ~150 ms after the last tick):

```yaml
  - id: volume_apply
    mode: restart
    then:
      - delay: 150ms
      - homeassistant.action:
          action: media_player.volume_set
          data:
            entity_id: media_player.denon
            volume_level: !lambda 'return id(pending_volume);'
      - lambda: 'id(pending_volume) = -1;'
```

- [ ] **Step 3: Add press dispatch (SINGLE/LONG) for MEDIA**

In the SINGLE `on_multi_click` handler, after the wake line, branch by screen; for MEDIA call `media_press`:

```yaml
        then:
          - light.turn_on: { id: backlight, brightness: 100% }
          - if:
              condition: { lambda: 'return id(current_screen) == 0;' }
              then: [ script.execute: media_press ]
              else: [ script.execute: climate_press ]   # defined in Task 9
```

In the LONG handler:

```yaml
        then:
          - light.turn_on: { id: backlight, brightness: 100% }
          - if:
              condition: { lambda: 'return id(current_screen) == 0;' }
              then: [ script.execute: media_long ]
```

Add the scripts:

```yaml
  - id: media_press
    then:
      - lambda: |-
          std::string avr = id(ha_avr_state).state;
          std::string heos = id(ha_heos_state).state;
          bool off = (avr == "off" || avr == "unavailable" || avr == "");
          bool heos_active = (heos == "playing" || heos == "paused");
          if (off) { id(do_resume).execute(); }
          else if (heos_active) { id(do_play_pause).execute(); }
  - id: do_resume
    then:
      - homeassistant.action: { action: script.turn_on, data: { entity_id: script.m5dial_denon_spotify_resume } }
  - id: do_play_pause
    then:
      - homeassistant.action: { action: media_player.media_play_pause, data: { entity_id: media_player.denon_2 } }
  - id: media_long
    then:
      - lambda: |-
          std::string avr = id(ha_avr_state).state;
          if (!(avr == "off" || avr == "unavailable" || avr == "")) { id(do_avr_off).execute(); }
  - id: do_avr_off
    then:
      - homeassistant.action: { action: media_player.turn_off, data: { entity_id: media_player.denon } }
```

- [ ] **Step 4: Add touch handlers for prev/next/play**

On `m_btn_prev`, `m_btn_next`, `m_btn_play` add `on_click`:

```yaml
        - button:
            id: m_btn_prev
            ...
            on_click:
              - homeassistant.action: { action: media_player.media_previous_track, data: { entity_id: media_player.denon_2 } }
        - button:
            id: m_btn_next
            ...
            on_click:
              - homeassistant.action: { action: media_player.media_next_track, data: { entity_id: media_player.denon_2 } }
        - button:
            id: m_btn_play
            ...
            on_click:
              - script.execute: do_play_pause
```

- [ ] **Step 5: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 6: Verify (device + HA)**

Expected: turning the bezel on MEDIA changes `media_player.denon` master volume in HA (arc + label follow); single-press toggles HEOS play/pause; prev/next touch change tracks; long-press turns the AVR off; with AVR off, single-press runs the resume script (AVR on + Spotify plays). Confirm each in the HA UI / logbook.

- [ ] **Step 7: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): MEDIA actions + Spotify-resume script"
```

---

## Task 8: CLIMATE screen UI (AC summer / Tado winter)

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (build `climate_page` widgets; add `update_climate` script; trigger from climate/season sensors)

**Interfaces:**
- Consumes: `ha_ac_state`, `ha_ac_target`, `ha_ac_current`, `ha_ac_fan`, `ha_ac_swing`, `ha_season`, `ha_tado_target`.
- Produces: widgets `c_title`, `c_temp`, `c_current`, `c_fan_auto/low/med/high`, `c_swing`, `c_power_off`, `c_on_btn`; script `update_climate` applying off/on visibility and season label. Touch handlers + actions land in Task 9.

- [ ] **Step 1: Replace `climate_page.widgets` with the real tree**

```yaml
    - id: climate_page
      widgets:
        - label:
            id: c_title
            align: TOP_MID
            y: 26
            text: "AC · Salon"
            text_color: 0x9AA0A6
        - label:
            id: c_temp
            align: CENTER
            y: -16
            text_font: montserrat_40
            text_color: 0xFFFFFF
            text: "—"
        - label:
            id: c_current
            align: CENTER
            y: 24
            text_color: 0x9AA0A6
            text: ""
        - button:
            id: c_fan_auto
            align: CENTER
            x: -78
            y: 64
            width: 38
            height: 26
            widgets: [ label: { align: CENTER, text: "auto" } ]
        - button:
            id: c_fan_low
            align: CENTER
            x: -36
            y: 64
            width: 38
            height: 26
            widgets: [ label: { align: CENTER, text: "low" } ]
        - button:
            id: c_fan_med
            align: CENTER
            x: 6
            y: 64
            width: 38
            height: 26
            widgets: [ label: { align: CENTER, text: "med" } ]
        - button:
            id: c_fan_high
            align: CENTER
            x: 48
            y: 64
            width: 38
            height: 26
            widgets: [ label: { align: CENTER, text: "high" } ]
        - button:
            id: c_swing
            align: CENTER
            x: 0
            y: 96
            width: 70
            height: 26
            widgets: [ label: { id: c_swing_lbl, align: CENTER, text: "swing" } ]
        - button:
            id: c_power_off
            align: TOP_RIGHT
            x: -22
            y: 22
            width: 34
            height: 34
            bg_color: 0xA32D2D
            widgets: [ label: { align: CENTER, text: "", text_font: fa_icons } ]
        - button:
            id: c_on_btn
            align: CENTER
            width: 150
            height: 150
            radius: 75
            bg_color: 0x185FA5
            hidden: true
            widgets:
              - label: { align: CENTER, text: "  Włącz", text_font: fa_icons }
```

- [ ] **Step 2: Add the `update_climate` script**

```yaml
  - id: update_climate
    then:
      - lambda: |-
          bool winter = (id(ha_season).state == "on");
          std::string st = id(ha_ac_state).state;
          bool on = !(st == "off" || st == "unavailable" || st == "");
          auto show = [](lv_obj_t* o, bool v){ if (v) lv_obj_clear_flag(o, LV_OBJ_FLAG_HIDDEN); else lv_obj_add_flag(o, LV_OBJ_FLAG_HIDDEN); };
          lv_label_set_text((lv_obj_t*)id(c_title), winter ? "Tado · LR+Kitchen" : "AC · Salon");
          if (winter) {
            // Tado: always "on", show single setpoint, hide AC-only controls
            show((lv_obj_t*)id(c_on_btn), false);
            show((lv_obj_t*)id(c_power_off), false);
            show((lv_obj_t*)id(c_fan_auto), false); show((lv_obj_t*)id(c_fan_low), false);
            show((lv_obj_t*)id(c_fan_med), false); show((lv_obj_t*)id(c_fan_high), false);
            show((lv_obj_t*)id(c_swing), false);
            show((lv_obj_t*)id(c_temp), true); show((lv_obj_t*)id(c_current), false);
            float t = id(ha_tado_target).has_state() ? id(ha_tado_target).state : -100.0f;
            char b[16]; if (t > -50) snprintf(b,sizeof(b),"%.1f°",t); else snprintf(b,sizeof(b),"—");
            lv_label_set_text((lv_obj_t*)id(c_temp), b);
            return;
          }
          // summer / AC
          show((lv_obj_t*)id(c_on_btn), !on);
          show((lv_obj_t*)id(c_temp), on);
          show((lv_obj_t*)id(c_current), on);
          show((lv_obj_t*)id(c_power_off), on);
          show((lv_obj_t*)id(c_fan_auto), on); show((lv_obj_t*)id(c_fan_low), on);
          show((lv_obj_t*)id(c_fan_med), on); show((lv_obj_t*)id(c_fan_high), on);
          show((lv_obj_t*)id(c_swing), on);
          if (on) {
            float t = id(ha_ac_target).has_state() ? id(ha_ac_target).state : -100.0f;
            char b[16]; if (t > -50) snprintf(b,sizeof(b),"%.1f°",t); else snprintf(b,sizeof(b),"—");
            lv_label_set_text((lv_obj_t*)id(c_temp), b);
            float cur = id(ha_ac_current).has_state() ? id(ha_ac_current).state : -100.0f;
            char c[24]; if (cur > -50) snprintf(c,sizeof(c),"akt. %.1f°",cur); else snprintf(c,sizeof(c),"");
            lv_label_set_text((lv_obj_t*)id(c_current), c);
            // highlight active fan: map nature->auto
            std::string fan = id(ha_ac_fan).state;
            lv_obj_set_style_bg_color((lv_obj_t*)id(c_fan_auto), lv_color_hex(fan=="nature"?0x185FA5:0x2C2C2C), 0);
            lv_obj_set_style_bg_color((lv_obj_t*)id(c_fan_low),  lv_color_hex(fan=="low"?0x185FA5:0x2C2C2C), 0);
            lv_obj_set_style_bg_color((lv_obj_t*)id(c_fan_med),  lv_color_hex(fan=="medium"?0x185FA5:0x2C2C2C), 0);
            lv_obj_set_style_bg_color((lv_obj_t*)id(c_fan_high), lv_color_hex(fan=="high"?0x185FA5:0x2C2C2C), 0);
            lv_obj_set_style_bg_color((lv_obj_t*)id(c_swing), lv_color_hex(id(ha_ac_swing).state=="on"?0x185FA5:0x2C2C2C), 0);
          }
```

- [ ] **Step 3: Trigger `update_climate` from the climate/season sensors**

Add `on_value: - script.execute: update_climate` to `ha_ac_state`, `ha_ac_target`, `ha_ac_current`, `ha_ac_fan`, `ha_ac_swing`, `ha_season`, `ha_tado_target`.

- [ ] **Step 4: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 5: Verify on device**

Expected: with `heating_season=off` and AC off → CLIMATE page shows the big "Włącz" button only; turn AC on in HA → temp + current + fan segments + swing + red power-off appear, active fan highlighted (nature shows as "auto"). Set `heating_season=on` → title "Tado · LR+Kitchen", single Tado setpoint, AC-only controls hidden.

- [ ] **Step 6: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): CLIMATE screen UI (AC + Tado)"
```

---

## Task 9: CLIMATE actions + temp overlay

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (climate press/encoder/touch actions + debounce)

**Interfaces:**
- Consumes: `update_climate`, encoder `else` branches from Task 7 (`climate_temp_up`/`climate_temp_down`), `climate_press`, `ha_season`, `ha_ac_state`, `ha_ac_target`, `ha_tado_target`.
- Produces: scripts `climate_temp_up`, `climate_temp_down`, `temp_apply`, `climate_press`; touch handlers on fan/swing/power/on. After this task the CLIMATE screen is fully interactive.

- [ ] **Step 1: Add temp encoder scripts (season-aware base + step 0.5)**

```yaml
  - id: climate_temp_up
    then:
      - lambda: |-
          bool winter = (id(ha_season).state == "on");
          float cur = id(pending_temp) >= -50 ? id(pending_temp)
                    : (winter ? (id(ha_tado_target).has_state()? id(ha_tado_target).state : 21.0f)
                              : (id(ha_ac_target).has_state()? id(ha_ac_target).state : 22.0f));
          float t = cur + 0.5f; float hi = winter ? 25.0f : 30.0f; if (t > hi) t = hi;
          id(pending_temp) = t;
          char b[16]; snprintf(b,sizeof(b),"%.1f°",t);
          lv_label_set_text((lv_obj_t*)id(c_temp), b);
      - script.execute: temp_apply
  - id: climate_temp_down
    then:
      - lambda: |-
          bool winter = (id(ha_season).state == "on");
          float cur = id(pending_temp) >= -50 ? id(pending_temp)
                    : (winter ? (id(ha_tado_target).has_state()? id(ha_tado_target).state : 21.0f)
                              : (id(ha_ac_target).has_state()? id(ha_ac_target).state : 22.0f));
          float t = cur - 0.5f; float lo = winter ? 5.0f : 18.0f; if (t < lo) t = lo;
          id(pending_temp) = t;
          char b[16]; snprintf(b,sizeof(b),"%.1f°",t);
          lv_label_set_text((lv_obj_t*)id(c_temp), b);
      - script.execute: temp_apply
  - id: temp_apply
    mode: restart
    then:
      - delay: 150ms
      - if:
          condition: { lambda: 'return id(ha_season).state == "on";' }
          then:
            - homeassistant.action:
                action: climate.set_temperature
                data:
                  entity_id: [climate.living_room, climate.kitchen]
                  temperature: !lambda 'return id(pending_temp);'
          else:
            - homeassistant.action:
                action: climate.set_temperature
                data:
                  entity_id: climate.air_conditioner
                  temperature: !lambda 'return id(pending_temp);'
      - lambda: 'id(pending_temp) = -100;'
```

- [ ] **Step 2: Add `climate_press` (power toggle; winter no-op)**

```yaml
  - id: climate_press
    then:
      - lambda: |-
          if (id(ha_season).state == "on") return;   // Tado: no power toggle
          std::string st = id(ha_ac_state).state;
          bool on = !(st == "off" || st == "unavailable" || st == "");
          if (on) id(do_ac_off).execute(); else id(do_ac_on).execute();
  - id: do_ac_on
    then:
      - homeassistant.action: { action: climate.turn_on, data: { entity_id: climate.air_conditioner } }
  - id: do_ac_off
    then:
      - homeassistant.action: { action: climate.turn_off, data: { entity_id: climate.air_conditioner } }
```

- [ ] **Step 3: Add touch handlers (fan / swing / power-off / on-button)**

Add `on_click` to each climate button:

```yaml
        - button: { id: c_fan_auto, ..., on_click: [ homeassistant.action: { action: climate.set_fan_mode, data: { entity_id: climate.air_conditioner, fan_mode: "nature" } } ] }
        - button: { id: c_fan_low,  ..., on_click: [ homeassistant.action: { action: climate.set_fan_mode, data: { entity_id: climate.air_conditioner, fan_mode: "low" } } ] }
        - button: { id: c_fan_med,  ..., on_click: [ homeassistant.action: { action: climate.set_fan_mode, data: { entity_id: climate.air_conditioner, fan_mode: "medium" } } ] }
        - button: { id: c_fan_high, ..., on_click: [ homeassistant.action: { action: climate.set_fan_mode, data: { entity_id: climate.air_conditioner, fan_mode: "high" } } ] }
        - button:
            id: c_swing
            ...
            on_click:
              - homeassistant.action:
                  action: climate.set_swing_mode
                  data:
                    entity_id: climate.air_conditioner
                    swing_mode: !lambda 'return id(ha_ac_swing).state == "on" ? "off" : "on";'
        - button: { id: c_power_off, ..., on_click: [ script.execute: do_ac_off ] }
        - button: { id: c_on_btn,    ..., on_click: [ script.execute: do_ac_on ] }
```

- [ ] **Step 4: Validate + flash**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`.

- [ ] **Step 5: Verify (device + HA)**

Expected (summer): on CLIMATE, "Włącz" turns AC on; encoder changes AC target temp (HA follows, step 0.5, clamped 18–30); fan buttons set nature/low/medium/high in HA; swing toggles; power-off turns AC off. Winter (`heating_season=on`): encoder sets the SAME setpoint on `climate.living_room` AND `climate.kitchen` (confirm both in HA); press does nothing.

- [ ] **Step 6: Commit**

```bash
git add m5dial/m5dial-salon.yaml
git commit -m "feat(m5dial): CLIMATE actions (AC + Tado sync) + temp debounce"
```

---

## Task 10: Edge cases + overlay auto-hide + offline indicator + README

**Files:**
- Modify: `m5dial/m5dial-salon.yaml` (overlay revert, offline dot, unavailable guards)
- Create: `m5dial/README.md`

**Interfaces:**
- Consumes: everything above. Produces: `api.connected`-driven offline indicator, a revert-to-echo timer so the optimistic overlay snaps back to HA truth, and operator docs.

- [ ] **Step 1: Add an offline indicator dot to both pages**

Add to each page's widgets:

```yaml
        - led:
            id: ofl_dot          # grey when offline, hidden when connected
            align: TOP_MID
            y: 6
            width: 8
            height: 8
            color: 0x5C5C5C
            hidden: true
```

Add an interval that reflects API connectivity:

```yaml
interval:
  - interval: 2s
    then:
      - lambda: |-
          bool up = api_is_connected();
          lv_obj_t* a = (lv_obj_t*) (id(current_screen)==0 ? id(m_vol_lbl) : id(c_current)); (void)a;
          if (up) { lv_obj_add_flag((lv_obj_t*)id(ofl_dot), LV_OBJ_FLAG_HIDDEN); }
          else    { lv_obj_clear_flag((lv_obj_t*)id(ofl_dot), LV_OBJ_FLAG_HIDDEN); }
```

(Include `#include "esphome/components/api/api_server.h"` is not needed; use `esphome::api::global_api_server->is_connected()` via a small lambda helper if `api_is_connected()` is unavailable in your ESPHome version — confirm at compile and adjust.)

- [ ] **Step 2: Overlay auto-revert** — make `volume_apply`/`temp_apply` re-run the echo refresh after sending, so a stale optimistic value can't stick:

Append to `volume_apply.then` (after the lambda reset): `- script.execute: update_media`
Append to `temp_apply.then` (after the lambda reset): `- script.execute: update_climate`

- [ ] **Step 3: Unavailable guards** — already handled in `update_media`/`update_climate` ("—" when no state). Verify the encoder `base` fallbacks don't send garbage when `ha_volume`/targets are unknown by guarding `volume_apply`/`temp_apply`:

Wrap each `homeassistant.action` in an `if` with `condition: { lambda: 'return id(pending_volume) >= 0;' }` (volume) / `'return id(pending_temp) >= -50;'` (temp).

- [ ] **Step 4: Write `m5dial/README.md`**

```markdown
# M5Dial Salon (ESPHome firmware)

Rotary + touch controller: Denon media (source-aware) + living-room climate.

## Flash
- First time: USB-C → `esphome run m5dial/m5dial-salon.yaml`
- After: OTA over Wi-Fi (same command).

## Pins (M5Dial, ESP32-S3)
display GC9A01A cs7/dc4/rst8, SPI mosi5/clk6, backlight 9; touch ft5x06 @0x38 I2C sda11/scl12; encoder 40/41; button 42.

## HA entities used
media_player.denon (AVR/volume/source), media_player.denon_2 (HEOS), media_player.apple_tv,
climate.air_conditioner (LG), climate.living_room + climate.kitchen (Tado), input_boolean.heating_season.
Requires HA script: script.m5dial_denon_spotify_resume.
```

- [ ] **Step 5: Validate + flash + full DoD pass**

Run: `esphome config m5dial/m5dial-salon.yaml` → valid; `esphome run m5dial/m5dial-salon.yaml`. Then walk the spec's Definition of Done checklist end to end.

- [ ] **Step 6: Commit**

```bash
git add m5dial/m5dial-salon.yaml m5dial/README.md
git commit -m "feat(m5dial): edge cases, offline indicator, overlay revert, docs"
```

---

## Self-Review notes (coverage vs spec)

- Interaction model / state machine → Tasks 5 (nav/idle), 7 (MEDIA press/encoder), 9 (CLIMATE press/encoder). ✅
- MEDIA source router (off / AppleTV / PS5 / HEOS / other) → Task 6. ✅
- MEDIA actions (volume→AVR, play/pause, prev/next, long→off, resume script) → Task 7. ✅
- CLIMATE AC off/on UI + fan(nature→auto)/swing + Tado season → Tasks 8, 9. ✅
- Data flow read/write entities → Task 4 + per-action `homeassistant.action`. ✅
- Edge cases (offline dot, unavailable "—", debounce, optimistic revert) → Tasks 7/9/10. ✅
- DoD checklist → exercised in Task 10 Step 5. ✅

**Known on-device tuning (verification, not placeholders):** LVGL widget coordinates (`x/y/width`), `invert_colors`, the touch-controller platform (ft5x06 vs cst816, Task 3 scan), the Spotify-resume source name ("HEOS Music" vs "NET"), and the exact `api_is_connected()` helper name for your ESPHome version. Each has a concrete starting value and a verification step.
