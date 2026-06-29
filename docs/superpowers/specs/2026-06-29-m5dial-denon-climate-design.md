# M5Dial — sterownik Denon (Spotify) + klimatyzacja — design

*Data: 2026-06-29 · Autor: Mateusz + Claude Code · Status: zatwierdzony design, gotowy do planu wdrożenia*

## Kontekst i zakres

Dwa urządzenia naścienne w salonie obok siebie: **SONOFF NSPanel** i **M5Dial (ESP32-S3)**.
Ten spec obejmuje **wyłącznie M5Dial** — pierwszy z dwóch niezależnych podprojektów.
NSPanel zostanie zaprojektowany osobno (własny cykl spec → plan → wdrożenie).

M5Dial pełni rolę dotykowo-pokrętłowego kontrolera dwóch obszarów:
1. **Media** — Spotify Connect / źródła AVR Denon (salon).
2. **Klimatyzacja** — LG AC (lato) lub Tado sync (zima).

### Sprzęt M5Dial
- ESP32-S3, okrągły ekran GC9A01 240×240, dotyk pojemnościowy CST816.
- Enkoder obrotowy na obręczy + przycisk pod pokrętłem (single / double / long press).

### Decyzja architektoniczna
**ESPHome + LVGL on-device.** UI renderowany lokalnie na M5Dial; łączność z HA przez native API.
Single source of truth = Home Assistant; enkoder/dotyk wysyłają komendy, a wartości „dociągają się" z encji (echo), więc stan UI nie rozjeżdża się z HA.

Odrzucone: `display:` lambda bez LVGL (żmudne, brzydsze), hybryda HA-driven (słabo wspierana na M5Dial, zależna od HA online).

---

## Encje HA (potwierdzone w instancji)

| Rola | entity_id | Uwagi |
|---|---|---|
| AVR Denon (master volume, źródło, power) | `media_player.denon` | „Denon AVR-S960H". `volume_set`/`volume_step`, `select_source`, `turn_on/off`. Brak transportu/metadanych. |
| HEOS (Spotify, metadane, transport) | `media_player.denon_2` | „DENON HEOS". `media_title`/`media_artist`, play/pause, next/prev, volume. |
| Apple TV | `media_player.apple_tv` | Tytuł treści (Netflix/YT/HBO) + `app_name`. |
| LG AC salon | `climate.air_conditioner` | „AC Living room". hvac off/cool/auto (UI), temp 18–30 krok 0.5, fan `nature/low/medium/high`, swing pionowy `on/off`. |
| Tado salon | `climate.living_room` | Sync zimowy. |
| Tado kuchnia | `climate.kitchen` | Sync zimowy. |
| Helper sezonu | `input_boolean.heating_season` | `off`=lato (AC), `on`=zima (Tado). |

**Skrypt do utworzenia:** `script.m5dial_denon_spotify_resume` — wybudzenie AVR + wznowienie Spotify (patrz niżej).

---

## Model interakcji i maszyna stanów

Dwa ekrany główne: **MEDIA** (domyślny/home) i **CLIMATE**.

| Wejście | Ekran MEDIA | Ekran CLIMATE |
|---|---|---|
| Obrót enkodera | master volume AVR (`volume_set`, krok ~2%), zawsze niezależnie od źródła | target temp (18–30°C, krok 0.5); zimą oba Tado naraz |
| Single press | zależne od stanu/źródła (patrz tabela MEDIA) | toggle power (włącz gdy off / wyłącz gdy on) |
| Long press (~1s) | wyłącz Denon (gdy AVR on) | — |
| Double press | → przejście na CLIMATE | → powrót na MEDIA |
| Dotyk | prev / next (gdy widoczne) | fan, swing, lub duży „Włącz" gdy off |
| Idle 15s | — (zostaje) | auto-powrót na MEDIA |
| Idle ~60s | przygaszenie ekranu (dim), wybudzenie dowolnym wejściem | jw. |

```
            double press                 double press / idle 15s
  MEDIA  ───────────────►  CLIMATE  ───────────────────────────►  MEDIA
```

Gesty single / double / long rozróżniane w ESPHome (`on_multi_click` + detekcja hold).

---

## Ekran MEDIA (świadomy źródła AVR)

Router ekranu czyta `media_player.denon` attr `source`. Głośność (enkoder) zawsze = master volume AVR.

**Priorytet stanów (od góry):**

| Warunek | Wyświetlanie | Single press | Long press |
|---|---|---|---|
| AVR `off` | Logo Denon + znaczek power | Wybudź Denon + wznów Spotify (`script.m5dial_denon_spotify_resume`) | — |
| source = `APPLE TV` | Tytuł z `media_player.apple_tv` (+ `app_name`), bez transportu | — | Wyłącz Denon |
| source = `PlayStation 5` | Logo PS5, bez transportu | — | Wyłącz Denon |
| HEOS gra (`denon_2` playing/paused) | Pełny Spotify: tytuł/artysta + prev / play-pause / next | Play/pause | Wyłącz Denon |
| inne źródło (TV Audio, Bluetooth, CD, Blu-ray, Tuner…) | Logo Denon + nazwa źródła, bez transportu | — | Wyłącz Denon |

**Layout (pełny Spotify):** górny label „HEOS · Spotify", tytuł (19px) + artysta (14px), centralny przycisk play/pause, dotykowe prev/next u dołu, łuk poziomu głośności na obręczy + „Vol NN%". **Bez okładki albumu** (czytelność na 240×240 + koszt pobierania obrazka w ESPHome). Gdy HEOS nie gra a źródło sieciowe → stan „Nic nie gra" + ikona Spotify; enkoder dalej steruje głośnością.

**`script.m5dial_denon_spotify_resume` (HA):** `media_player.turn_on` na `denon` → wybór źródła sieciowego (NET / HEOS Music — do dostrojenia) → `media_player.media_play` na `denon_2`. Jeden punkt strojenia gdyby Spotify Connect wymagał aktywnej sesji. M5Dial woła `script.turn_on`.

---

## Ekran CLIMATE

Pod-stan wg `input_boolean.heating_season`:
- `off` (lato) → steruje `climate.air_conditioner` (LG).
- `on` (zima) → **Tado sync**: jedno pokrętło ustawia ten sam setpoint na `climate.living_room` **i** `climate.kitchen` jednocześnie (jeden `climate.set_temperature` na oba targety).

**LG AC — pod-stany wg power:**
- **OFF** → pełnoekranowy okrągły przycisk „Włącz" (ikona power w centrum), reszta ukryta. Press lub tap = `climate.turn_on`.
- **ON** → duża temperatura zadana w centrum, pod nią aktualna (`current_temperature`); segmenty fan; toggle swing pionowego; czerwona ikona power-off.

**Mapowanie wyświetlania fan** (wartość do HA w nawiasie): `auto` (`nature`) · `low` (`low`) · `med` (`medium`) · `high` (`high`). Etykieta „auto" jest czysto kosmetyczna — do HA idzie `nature`.

**Swing:** pojedynczy toggle swing **pionowego** `on/off` (`climate.set_swing_mode`). Poziomy pomijamy.

Feedback obrotu enkodera: chwilowy overlay z wartością („Vol 38%" / „22.5°C") znikający po ~1.5 s.

---

## Przepływ danych

**Odczyt (HA → M5Dial), komponenty `homeassistant`:**

| Element UI | Źródło |
|---|---|
| Tytuł / artysta | `text_sensor` ← `denon_2` attr `media_title` / `media_artist` |
| Stan odtwarzania | `text_sensor` ← stan `denon_2` |
| Źródło AVR (router ekranów) | `text_sensor` ← `denon` attr `source` |
| Power AVR | `text_sensor` ← stan `denon` |
| Poziom głośności | `sensor` ← `denon` attr `volume_level` (0–1 → %) |
| Tytuł Apple TV | `text_sensor` ← `apple_tv` attr `media_title` / `app_name` |
| Stan / temp zadana / aktualna AC | `sensor`/`text_sensor` ← `air_conditioner` (state, `temperature`, `current_temperature`) |
| Fan / swing AC | `text_sensor` ← attr `fan_mode` / `swing_mode` |
| Setpoint Tado | `sensor` ← `living_room` attr `temperature` |
| Sezon | `binary_sensor` ← `input_boolean.heating_season` |

**Zapis (M5Dial → HA), `homeassistant.action`:**

| Akcja | Serwis → target |
|---|---|
| Play/pause | `media_player.media_play_pause` → `denon_2` |
| Prev / next | `media_player.media_previous_track` / `media_next_track` → `denon_2` |
| Głośność | `media_player.volume_set` → `denon` |
| Wybudź + wznów Spotify | `script.turn_on` → `script.m5dial_denon_spotify_resume` |
| Wyłącz Denon | `media_player.turn_off` → `denon` |
| Temp AC | `climate.set_temperature` → `air_conditioner` |
| Fan / swing AC | `climate.set_fan_mode` / `set_swing_mode` → `air_conditioner` |
| Power AC | `climate.turn_on` / `turn_off` → `air_conditioner` |
| Temp Tado (zima) | `climate.set_temperature` → `living_room` + `kitchen` (oba) |

---

## Stany brzegowe i obsługa błędów

- **HA API niedostępne:** LVGL renderuje dalej; encje pokazują ostatnią znaną wartość; subtelny wskaźnik „offline" (szara kropka). Po reconnect wartości się dociągają.
- **Encja `unavailable`/`unknown`:** głośność/media pokazują „—"; przyciski nie wysyłają śmieci.
- **Rozjazd enkoder↔encja:** lokalny „optimistic" overlay ~1.5 s, potem korekta z echa encji HA — bez migotania.
- **Debounce enkodera:** szybkie kręcenie batchuje wysyłkę (~co 150 ms / na końcu ruchu), by nie zalać AVR-a/Tado.
- **Tado sync częściowy fail:** jeśli jeden termostat nie odpowie, drugi i tak dostaje setpoint; log w HA, brak twardego błędu na ekranie.
- **Idle/dim:** po ~60 s ekran przygasa (backlight); wybudzenie dowolnym wejściem; stan zachowany.

---

## Definition of Done (checklista akceptacyjna)

**Walidacja konfiguracji (przed flashem):**
- [ ] `esphome config m5dial.yaml` — bez błędów.
- [ ] `esphome compile m5dial.yaml` — pełny build OK.

**Smoke test sprzętu (po flashu):**
- [ ] Ekran się zapala; enkoder liczy w obie strony; dotyk reaguje.
- [ ] Przycisk rozróżnia single / double / long (widać w `esphome logs`).
- [ ] Urządzenie ESPHome widoczne w HA, encje online.

**Przepływy (na żywo):**
- [ ] Spotify: tytuł+artysta widoczne; play/pause; prev/next; enkoder zmienia master volume AVR (sprawdzone w HA); łuk głośności podąża.
- [ ] Router źródeł: Apple TV → tytuł z `apple_tv`; PS5 → logo PS5; AVR off → logo Denon; single press wybudza+gra (skrypt); long press wyłącza Denon.
- [ ] AC lato: off → duży „Włącz", press włącza; temp z enkodera; fan (auto/low/med/high → poprawne wartości w HA); swing toggle; power-off.
- [ ] Tado zima: `heating_season=on` → ekran steruje Tado; enkoder ustawia **oba** (`living_room`+`kitchen`) — potwierdzone w HA.
- [ ] Nawigacja: double-press w obie strony; auto-powrót po 15 s; dim po 60 s.

**Stany brzegowe:**
- [ ] AVR off / encja unavailable → „—"/logo, brak wysyłki śmieci.
- [ ] HA offline → ostatnie wartości + wskaźnik offline; po reconnect dociągnięcie.

---

## Poza zakresem (świadomie)

- NSPanel (osobny podprojekt — żaluzje kuchnia + reszta TBD).
- Okładki albumów na ekranie media.
- Swing poziomy LG.
- Tryby AC inne niż off/cool/auto.
- RFID / RTC / buzzer M5Dial (brak potrzeby w tej wersji).
