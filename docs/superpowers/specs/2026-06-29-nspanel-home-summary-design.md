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
**NSPanel Lovelace UI (Blackymas) — ESPHome + blueprint HA.**
- Dojrzały projekt społeczności; konfiguracja przez blueprint w UI HA (bez AppDaemon).
- Ekran spoczynkowy z zegarem/datą/pogodą wbudowany; strony kart encji + karty termostatu.

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
- [ ] NSPanel sflashowany (NSPanel Lovelace UI: ESPHome + TFT), bootuje, w HA online, czas/data OK.
- [ ] Blueprint Blackymas zaimportowany i skonfigurowany.

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
