# SONOFF NSPanel — panel-summary domu + osłony — design

*Data: 2026-06-29 · Autor: Mateusz + Claude Code · Status: zatwierdzony design, gotowy do planu wdrożenia*

## Kontekst i zakres

Drugi z dwóch podprojektów naściennych w salonie (pierwszy: M5Dial — patrz
`2026-06-29-m5dial-denon-climate-design.md`). NSPanel stoi **obok M5Diala**, więc
świadomie się nie dubluje: M5Dial robi media + klimatyzację salonu; NSPanel pokazuje
**te pokoje, których nie widać stojąc przy panelu** (sypialnia, biuro) i steruje
dwiema osłonami okiennymi w pobliżu.

**Rola NSPanela:** panel-glance („rzut oka na dom") + szybkie włączenie AC w 2 pokojach
+ 2 osłony pod przyciskami fizycznymi. Nie jest pełnym kontrolerem — to status + skróty.

### Sprzęt
SONOFF NSPanel (nie Pro): ekran dotykowy landscape + 2 przyciski fizyczne na dole,
2 wbudowane przekaźniki.

### Decyzja architektoniczna
**NSPanel Easy (`edwardtfn/NSPanel-Easy`) — ESPHome + blueprint HA.**
- Aktualnie utrzymywany następca Blackymas `NSPanel_HA_Blueprint` (ten przeszedł w tryb utrzymaniowy — tylko krytyczne fixy). Ta sama filozofia: blueprint + ESPHome, w pełni lokalnie, bez AppDaemon. Framework ESP-IDF.
- Konfiguracja przez blueprint w UI HA; ekran spoczynkowy z zegarem/datą/pogodą wbudowany; strony kart encji + karty termostatu.
- Docs: https://edwardtfn.github.io/NSPanel-Easy/ · komponenty (ESPHome firmware + TFT + blueprint) trzymać na tej samej wersji.

Odrzucone: **B) joBr99 + AppDaemon** (większa moc, ale dokłada utrzymanie AppDaemon —
nadmiar do panelu-summary); **C) custom ESPHome (Nextion lambda) / Tasmota Berry**
(display Nextion czyni własne UI bardzo żmudnym, odtwarza to co A daje gratis).

---

## Encje HA (potwierdzone)

| Rola | entity_id | Wartość przykładowa |
|---|---|---|
| Temp sypialnia | `sensor.bedroom_bedroom_temperature` | 24.2° |
| Temp biuro | `sensor.officetemperaturesensor_temperature` | 26.9° |
| AC sypialnia | `climate.bedroom_2` („Bedroomac") | cool/auto, fan auto/low/med/high/turbo, preset windFree |
| AC biuro | `climate.office` („officeac") | jw. |
| Pogoda na dworze | `weather.home` | sunny, 39° |
| Żaluzje kuchnia (lewy przycisk) | `cover.blind_tilt_ccd4` („Tilt Blinds", SwitchBot) | open/close |
| Rolety salon (prawy przycisk) | `cover.shadeslivingroom` | open/close |

**Do utworzenia (zależność):** grupy świateł per pokój — `light`/`group` dla **sypialni**
i **biura** (agregują wszystkie światła pokoju; wskaźnik „świeci się" + toggle).
Członków rozwiązać z obszarów HA lub listy użytkownika podczas wdrożenia; rozważyć reuse
istniejących `light.bedroom` / `light.office` jeśli już agregują.

---

## UX / struktura ekranów

### Ekran spoczynkowy (domyślny — widoczny ~90% czasu)
- Zegar (duży) + data.
- Pogoda na dworze z `weather.home`: temperatura + stan (ikona).
- **Wiersz summary (dół ekranu):** dwa wskaźniki — **Sypialnia** i **Biuro**, każdy:
  - ikona światła: zapalona (kolor akcent) gdy grupa świateł pokoju `on`, wygaszona gdy `off`;
  - temperatura pokoju.
- Dzięki temu summary („gdzie świeci się, jaka temperatura, data, pogoda") jest dostępne
  **bez dotykania ekranu**.

### Strona sterowania (po tapnięciu / wybudzeniu)
- Toggle świateł: **Sypialnia — światło** (grupa), **Biuro — światło** (grupa).
- Dwa kafle AC: **AC Sypialnia**, **AC Biuro**. Gdy off → „włącz"; gdy on → pokazuje setpoint/stan.
- Tap kafla AC → **pełna karta termostatu** (cardThermo): on/off, temperatura, tryb (cool/auto),
  fan, preset (windFree). Sterowanie `climate.bedroom_2` / `climate.office`.

### Przyciski fizyczne (relay detach)
2 przekaźniki NSPanela **odpięte** od przycisków (decoupled), aby klik wołał akcję HA, nie
lokalne obciążenie:
- **Lewy** → `cover.blind_tilt_ccd4` toggle open/close (żaluzje kuchnia).
- **Prawy** → `cover.shadeslivingroom` toggle open/close (rolety salon).

Logika toggle osłony: jeśli otwarte → zamknij; jeśli zamknięte → otwórz; (jeśli w ruchu → stop,
o ile firmware wspiera).

---

## Stany brzegowe
- **Encja `unavailable`:** wskaźnik światła neutralny (wygaszony), temp pokazuje „—"; kafel AC
  nieaktywny zamiast wysyłać śmieci.
- **HA reconnect:** po odzyskaniu łączności panel dociąga stany.
- **Grupa świateł pusta/błędna:** wskaźnik traktuje brak członków `on` jako „off".

---

## Definition of Done (checklista akceptacyjna)
**Flash i łączność:**
- [ ] NSPanel sflashowany (NSPanel Easy: ESPHome firmware + TFT), bootuje, w HA online, czas/data OK.
- [ ] Blueprint NSPanel Easy zaimportowany i skonfigurowany (firmware/TFT/blueprint na tej samej wersji).

**Ekran spoczynkowy:**
- [ ] Zegar + data poprawne (strefa Europe/Warsaw).
- [ ] Pogoda: temperatura + stan z `weather.home`.
- [ ] Wskaźnik Sypialnia/Biuro: ikona zgodna z realnym stanem grupy światła; temperatura widoczna i aktualna.

**Strona sterowania:**
- [ ] Toggle świateł Sypialnia/Biuro przełącza grupę (sprawdzone w HA).
- [ ] Kafel AC pokazuje off/on; tap otwiera kartę termostatu.
- [ ] Z karty można włączyć AC i ustawić temp/tryb/fan/preset (`bedroom_2`, `office`).

**Przyciski fizyczne:**
- [ ] Lewy: open/close `cover.blind_tilt_ccd4`.
- [ ] Prawy: open/close `cover.shadeslivingroom`.
- [ ] Przekaźniki odpięte — klik nie klika lokalnego obciążenia.

**Stany brzegowe:**
- [ ] Encja unavailable → wskaźnik neutralny / „—", brak wysyłki śmieci.

---

## Poza zakresem (świadomie)
- Salon i kuchnia w summary (są na miejscu + media/klimat salonu na M5Dialu).
- Pełne sterowanie jasnością/kolorem świateł (tu tylko toggle on/off).
- Bezpieczeństwo (alarm/zamki), sceny, sterowanie pozostałymi roletami z dotyku (poza 2 przyciskami).
- Dopasowanie 1:1 do motywu Caule Black Rose (Nextion ma ograniczoną paletę; kolory per-encja OK).

---

## Implementacja (as-built, 2026-06-30)

Wdrożone i działające. Sterowane blueprintem `edwardtfn/nspanel_easy_blueprint.yaml`
przez automatyzację `automation.nspanel_salon` (inputy edytowane programowo przez MCP).

**Firmware:** NSPanel Easy v2026.6.2 (ESPHome firmware + TFT 20.0 + blueprint, wszystko 2026.6.2).
Urządzenie `nspanel-salon`, IP 192.168.1.51, WiFi `Czaja_IoT`.

**Ekran spoczynkowy:** zegar/data + pogoda `weather.home` + temp z panelu
(`indoortemp = sensor.nspanel_salon_temperature`). Temperatury pokoi i chipsy-żarówki
**usunięte** (na życzenie — było nieczytelne). Outdoor temp z `weather.home`.

**Home page — 4 kafle (custom buttons), żółty `[255,215,0]` gdy ON
(`advanced_settings.icon_color_fallback_on`):**
- `home_custom_button01` = `light.bedroom_lights` — „Bedroom", `mdi:bed`
- `home_custom_button02` = `climate.bedroom_2` — „AC Bedroom", `mdi:air-conditioner`
- `home_custom_button03` = `light.office_lights` — „Office", `mdi:desk-lamp`
- `home_custom_button04` = `climate.office` — „AC Office", `mdi:air-conditioner`
Kafle świateł = grupy „any-on": świecą żółto gdy jakiekolwiek światło w pokoju gra, tap = gasi całość.

**Grupy świateł (helper `group`, group_type light, any-on):**
- `light.bedroom_lights`: bedroom, shellybedroommain (MAIN), shellybedroomed (LED),
  bedroom_bed2 (Bedroom Bed), hue_ambiance_spot_1 / _1_2 / _1_3 / _1_4. (pominięto BedroomTV Ambilight)
- `light.office_lights`: shellyswitch25_3c6105e3695d_channel_1 (MAIN),
  yeelight_strip6_0x13f31f78 (Desktop), hue_color_lamp_1 (Ball), sofalampikea, printerlamp.
  (pominięto Bambu P1S — drukarka)
- Źródło składu pokoi: `script.shortcut_office` / `script.shortcut_bedroom` (część świateł nie była otagowana do obszaru HA).

**Przyciski fizyczne (relay decouple — `relay_X_local_fallback` OFF):**
- Lewy „Kuchnia" → `cover.blind_tilt_ccd4` (SwitchBot, **tylko TILT**, supported_features 240).
  Custom action toggle nachylenia metodą z `automation.ps5`:
  tilt > 10 → `cover.set_cover_tilt_position` 0; inaczej → `cover.open_cover_tilt`.
  (Zwykłe open/close NIE działa — ta żaluzja nie wspiera OPEN/CLOSE, tylko tilt.)
- Prawy „Salon" → `cover.shadeslivingroom` (zwykły cover, supported_features 15, toggle).

**Stosowanie zmian configu:** edycja inputów blueprintu wymaga `automation.reload` +
`button.nspanel_salon_restart` (panel pobiera layout po reboot). Zmiana SKŁADU grup
świateł nie wymaga restartu (encja `light.*_lights` ta sama).

**Build/flash (gotchas — patrz też pamięć projektu):** VM HA na Proxmoxie miała 2 GB →
OOM przy kompilacji (`Killed cc1plus`); podbito do ≥4 GB + `default_compile_process_limit: 1`.
Pierwszy flash serial (USB-TTL 3.3V, IO0→GND) przez web.esphome.io; potem OTA + auto-TFT.
Build robić przez LOKALNY adres HA (tunel Cloudflare ucinał WebSocket logu).
