# 📘 Instrukcja konfiguracji YamBMS — Seplos V3 + V2 + Victron

## Wersja: 1.6.0 | Data: 2026-04-11

---

## Spis treści

1. [Opis systemu](#1-opis-systemu)
2. [Wymagany sprzęt i akcesoria](#2-wymagany-sprzęt-i-akcesoria)
3. [Schemat połączeń fizycznych](#3-schemat-połączeń-fizycznych)
4. [Konfiguracja DIP switch (Seplos V2)](#4-konfiguracja-dip-switch-seplos-v2)
5. [Instalacja ESPHome](#5-instalacja-esphome)
6. [Przygotowanie plików konfiguracyjnych](#6-przygotowanie-plików-konfiguracyjnych)
7. [Wgranie firmware na LilyGo T-Connect](#7-wgranie-firmware-na-lilygo-t-connect)
8. [Podłączenie do Home Assistant](#8-podłączenie-do-home-assistant)
9. [Pierwsze uruchomienie i weryfikacja](#9-pierwsze-uruchomienie-i-weryfikacja)
10. [Monitorowanie i diagnostyka](#10-monitorowanie-i-diagnostyka)
11. [Troubleshooting — typowe problemy](#11-troubleshooting--typowe-problemy)
12. [Przydatne linki](#12-przydatne-linki)

---

## 1. Opis systemu

System YamBMS (Yet another multi-BMS Merging Solution) agreguje dane z wielu systemów BMS i prezentuje je jako jeden wirtualny system bateryjny dla invertera.

### Twoja konfiguracja:

| Element | Szczegóły |
|---------|-----------|
| **Kontroler** | LilyGo T-Connect (ESP32-S3, 8MB PSRAM, 16MB Flash) |
| **Grupa 1** | 3× Seplos V3 (Master + 2× Slave) → RS485 Port 1 (sniffer) |
| **Grupa 2** | 2× Seplos V2 (Master + Slave) → RS485 Port 2 (Modbus) |
| **Inverter** | Victron → CAN Port 4 (500 kbps) |
| **Łącznie** | 5 BMS → 1 Virtual BMS → Victron |

### Jak to działa:

```
┌─────────────┐                    ┌──────────────────┐                    ┌─────────────┐
│ Seplos V3 ×3│──── RS485_1 ────→  │                  │                    │             │
│ (sniffer)   │  Port 1 (GPIO4/5)  │   LilyGo         │──── CAN ────────→ │  Victron    │
│             │                    │   T-Connect       │  Port 4 (GPIO9/10)│  Inverter   │
│ Seplos V2 ×2│──── RS485_2 ────→  │   (YamBMS)       │                    │             │
│ (Modbus)    │  Port 2 (GPIO6/7)  │                  │                    │             │
└─────────────┘                    └──────────────────┘                    └─────────────┘
```

---

## 2. Wymagany sprzęt i akcesoria

### Sprzęt główny

| Lp. | Element | Ilość | Uwagi |
|-----|---------|-------|-------|
| 1 | LilyGo T-Connect (3×RS485 + 1×CAN) | 1 szt. | ⚠️ Wersja z 3×RS485 + 1×CAN! |
| 2 | Seplos V3 BMS (Master + Slave) | 3 szt. | Połączone w konfiguracji Master-Slave |
| 3 | Seplos V2 BMS (Master + Slave) | 2 szt. | Połączone w konfiguracji Master-Slave |
| 4 | Inverter Victron z portem CAN | 1 szt. | Np. MultiPlus-II, Cerbo GX |

### Akcesoria i kable

| Lp. | Element | Ilość | Uwagi |
|-----|---------|-------|-------|
| 5 | Kabel USB-C (do programowania) | 1 szt. | Musi obsługiwać dane (nie tylko ładowanie!) |
| 6 | Kabel skrętka RS485 (2-żyłowy + masa) | 2 szt. | Dla Port 1 i Port 2, zalecana skrętka ekranowana |
| 7 | Kabel CAN bus (2-żyłowy + masa) | 1 szt. | Do połączenia z Victron, impedancja 120Ω |
| 8 | Rezystor terminacyjny 120Ω (1/4W) | 4-6 szt. | Po 2 na każdą magistralę (RS485_1, RS485_2, CAN) |
| 9 | Zasilacz 7-12V DC | 1 szt. | Do zasilania stałego (opcja zamiast USB) |
| 10 | Obudowa / montaż DIN | 1 szt. | Opcjonalnie, otwory montażowe 2mm |

### Narzędzia

- Śrubokręt krzyżakowy (mały, do zacisków śrubowych)
- Multimetr (do weryfikacji terminacji i połączeń)
- Komputer z przeglądarką (do ESPHome Dashboard)

---

## 3. Schemat połączeń fizycznych

### Widok portów LilyGo T-Connect

```
    ┌─────────────────────────────────────────────────────┐
    │               LilyGo T-Connect                      │
    │                 (widok z góry)                       │
    │                                                     │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
    │  │  PORT 1   │  │  PORT 2   │  │  PORT 3   │  │  PORT 4   │
    │  │  RS485    │  │  RS485    │  │  RS485    │  │   CAN     │
    │  │ GPIO 4/5  │  │ GPIO 6/7  │  │ GPIO17/18 │  │ GPIO 9/10 │
    │  │           │  │           │  │           │  │           │
    │  │ SG L H DG │  │ SG L H DG │  │ SG L H DG │  │ SG L H DG │
    │  └──────────┘  └──────────┘  └──────────┘  └──────────┘
    │       ↓              ↓           (wolny)         ↓
    │   Seplos V3      Seplos V2                   Victron
    │   (sniffer)      (Modbus)                   (CAN bus)
    │                                                     │
    │  [USB-C]                              [DC 7-12V]    │
    └─────────────────────────────────────────────────────┘
```

### Podłączenie RS485 Port 1 → Seplos V3 (sniffer)

Tryb sniffer wymaga podłączenia ESP32 do istniejącej magistrali RS485 między BMS Master a Slave.

```
    Seplos V3                     LilyGo T-Connect
    Master ←──── RS485 ────→ Slave(s)       PORT 1
       │                                ┌──────────┐
       │         ┌──────────────────────│ SG       │
       ├── A+ ───┼── skrętka RS485 ─────│ L  (A+)  │
       ├── B- ───┼──────────────────────│ H  (B-)  │
       └── GND ──┼──────────────────────│ DG (GND) │
                 │                      └──────────┘
                 │
                 │ ESP32 "podsłuchuje" komunikację
                 │ Master ↔ Slave na tej samej magistrali
                 │
    ⚠️ WAŻNE: Podłącz ESP32 równolegle do istniejącej magistrali!
    ⚠️ Rezystor 120Ω na obu końcach magistrali RS485
```

### Podłączenie RS485 Port 2 → Seplos V2 (Modbus)

ESP32 aktywnie odpytuje każdy BMS po adresie Modbus.

```
    Seplos V2 #1         Seplos V2 #2         LilyGo T-Connect
    (addr: 0x01)         (addr: 0x02)              PORT 2
    ┌──────────┐         ┌──────────┐         ┌──────────┐
    │ A+ ──────┼─────────┼── A+ ────┼─────────│ L  (A+)  │
    │ B- ──────┼─────────┼── B- ────┼─────────│ H  (B-)  │
    │ GND ─────┼─────────┼── GND ───┼─────────│ DG (GND) │
    └──────────┘         └──────────┘         └──────────┘
     [120Ω]                                    [120Ω]
     (na końcu)                              (na końcu)

    ⚠️ Magistrala typu "daisy-chain" (łańcuchowe)
    ⚠️ Rezystor 120Ω na PIERWSZYM i OSTATNIM urządzeniu
    ⚠️ Każdy BMS V2 musi mieć unikalny adres (DIP switch)
```

### Podłączenie CAN Port 4 → Victron

```
    LilyGo T-Connect        Victron Inverter / Cerbo GX
         PORT 4                    CAN Port
    ┌──────────┐             ┌──────────┐
    │ L (CAN_L)│─────────────│ CAN L    │
    │ H (CAN_H)│─────────────│ CAN H    │
    │ DG (GND) │─────────────│ GND      │
    └──────────┘             └──────────┘
     [120Ω]                   [120Ω]*

    * Sprawdź czy Victron ma wbudowaną terminację!
      Jeśli tak, nie dodawaj zewnętrznego rezystora po stronie Victrona.

    ⚠️ Prędkość: 500 kbps
    ⚠️ Mierz multimetrem: ~60Ω między CAN_H a CAN_L = OK
```

---

## 4. Konfiguracja DIP switch (Seplos V2)

Każdy BMS Seplos V2 w trybie Modbus musi mieć unikalny adres ustawiony za pomocą przełączników DIP switch.

### Tabela adresów

| BMS | Adres Modbus | DIP Switch (1-2-3-4) | W konfiguracji |
|-----|-------------|----------------------|----------------|
| BMS 4 (Master V2) | 0x01 | ON-OFF-OFF-OFF | `bms_address: '0x01'` |
| BMS 5 (Slave V2) | 0x02 | OFF-ON-OFF-OFF | `bms_address: '0x02'` |

> **⚠️ UWAGA:** Dokładny układ DIP switchów może się różnić w zależności od wersji sprzętowej Seplos V2. Sprawdź dokumentację swojego modelu BMS!

### Procedura ustawiania adresu:

1. **Wyłącz** zasilanie BMS
2. Znajdź przełączniki DIP switch na płycie BMS
3. Ustaw odpowiednią kombinację zgodnie z tabelą powyżej
4. **Włącz** zasilanie BMS
5. Zweryfikuj adres w oprogramowaniu Seplos (opcjonalnie)

> **💡 Wskazówka:** Seplos V3 w trybie sniffer NIE wymaga konfiguracji adresów — ESP32 pasywnie nasłuchuje komunikacji Master-Slave.

---

## 5. Instalacja ESPHome

> **Wymagana wersja ESPHome:** `2025.11.0` lub nowsza (zalecana: najnowsza stabilna).
> Starsze wersje mogą zwracać mylące błędy walidacji YAML dla pakietów YamBMS.

### Opcja A: ESPHome jako dodatek Home Assistant (zalecana)

1. Otwórz **Home Assistant** → **Ustawienia** → **Dodatki** → **Sklep z dodatkami**
2. Wyszukaj **ESPHome** i zainstaluj
3. Uruchom dodatek ESPHome
4. Otwórz panel ESPHome (zazwyczaj: `http://homeassistant.local:6052`)

### Opcja B: ESPHome standalone (bez Home Assistant)

```bash
# Instalacja przez pip (Python 3.9+)
pip install esphome

# Lub przez Docker
docker run -d \
  --name esphome \
  -v /ścieżka/do/konfiguracji:/config \
  -p 6052:6052 \
  esphome/esphome
```

### Opcja C: ESPHome przez Docker Compose

```yaml
version: '3'
services:
  esphome:
    image: esphome/esphome:latest
    container_name: esphome
    volumes:
      - ./esphome_config:/config
    ports:
      - "6052:6052"
    restart: unless-stopped
```

---

## 6. Przygotowanie plików konfiguracyjnych

### Krok 1: Stwórz plik `secrets.yaml`

W katalogu konfiguracji ESPHome stwórz plik `secrets.yaml`:

```yaml
# secrets.yaml — Dane poufne (NIE udostępniaj tego pliku!)

wifi_ssid: "NazwaTwojejSieciWiFi"
wifi_password: "HasłoDoWiFi"
domain: ".local"

# Opcjonalnie dla MQTT:
# mqtt_host: "192.168.1.10"
# mqtt_username: "mqtt_user"
# mqtt_password: "mqtt_password"
```

### Krok 2: Skopiuj plik konfiguracyjny

Skopiuj plik `yambms_config.yaml` do katalogu konfiguracji ESPHome. Możesz go nazwać dowolnie, np. `yambms-seplos.yaml`.

### Krok 2a: Tryb pakietów (KLUCZOWE)

Ta konfiguracja działa w **trybie zdalnym (remote packages)** — sekcja `packages:` używa `url:` i `files:`.

- ✅ **Tryb zdalny (aktualny):** **NIE kopiujesz** lokalnie folderu `packages/`.
- ✅ Wystarczy internet + poprawna wersja ESPHome.
- ⚠️ Jeśli przejdziesz na **tryb lokalny** (`!include packages/...`), wtedy musisz skopiować **CAŁY** katalog `packages/` z repozytorium YamBMS, nie pojedyncze pliki.

### Krok 3: Zweryfikuj parametry

Otwórz plik i sprawdź/dostosuj:

- [ ] `hostname` — unikalna nazwa w sieci (np. `yambms-seplos`)
- [ ] Parametry napięć (`yambms_bulk_v`, `yambms_float_v`, `yambms_rebulk_v`) — zgodne z twoimi bateriami
- [ ] Parametry prądów (`yambms_max_requested_charge_current/discharge`) — zgodne z inverterem
- [ ] `yambms_cell_count` — liczba ogniw w szeregu (16 dla 48V LFP)
- [ ] `bms_cell_ovp` / `bms_cell_uvp` — progi zabezpieczeń ogniw
- [ ] `bms_max_charge_current` / `bms_max_discharge_current` — limity prądowe per BMS (Seplos V2)

---

## 7. Wgranie firmware na LilyGo T-Connect

### Pierwsze wgranie (przez USB)

1. **Podłącz** LilyGo T-Connect kablem USB-C do komputera
2. **Przytrzymaj** przycisk **BOOT** na płytce
3. W panelu ESPHome kliknij **„Install"** przy swoim urządzeniu
4. Wybierz **„Plug into this computer"** (lub „Manual download")
5. Wybierz port COM/Serial
6. Poczekaj na zakończenie kompilacji i wgrywania (~3-5 minut)
7. **Zwolnij** przycisk BOOT po rozpoczęciu wgrywania

### Alternatywa: Kompilacja z linii poleceń

```bash
# Kompilacja
esphome compile yambms-seplos.yaml

# Wgranie przez USB
esphome upload yambms-seplos.yaml

# Lub kompilacja + wgranie w jednym kroku
esphome run yambms-seplos.yaml
```

### Kolejne aktualizacje (OTA — bezprzewodowo)

Po pierwszym wgraniu możesz aktualizować firmware bezprzewodowo:

1. W panelu ESPHome kliknij **„Install"** → **„Wirelessly"**
2. ESPHome automatycznie znajdzie urządzenie w sieci
3. Aktualizacja trwa ~1-2 minuty

---

## 8. Podłączenie do Home Assistant

### Automatyczne wykrywanie

1. Po wgraniu firmware i podłączeniu do WiFi, Home Assistant powinien **automatycznie wykryć** nowe urządzenie ESPHome
2. Przejdź do **Ustawienia** → **Urządzenia i usługi** → **Integracje**
3. Powinieneś zobaczyć powiadomienie: **„Wykryto nowe urządzenie: YamBMS"**
4. Kliknij **„Konfiguruj"** i potwierdź dodanie

### Ręczne dodawanie

Jeśli automatyczne wykrywanie nie zadziała:

1. **Ustawienia** → **Urządzenia i usługi** → **Dodaj integrację**
2. Wyszukaj **ESPHome**
3. Wpisz adres: `yambms-seplos.local` (lub adres IP urządzenia)
4. Potwierdź dodanie

### Konfiguracja protokołu CAN w Home Assistant

Po dodaniu urządzenia do HA, musisz ustawić protokół CAN:

1. Przejdź do encji urządzenia YamBMS
2. Znajdź **„CANBUS 1 CAN Protocol"** → ustaw na **„Victron"** lub **„PYLON 1.2"**
3. Znajdź **„CANBUS 1 BMS Name"** → ustaw na **„Victron"** (lub zgodnie z twoim inverterem)

> **💡 Wskazówka:** Dla Victron MultiPlus-II zazwyczaj najlepiej działa protokół „PYLON 1.2" z nazwą „PYLON".

---

## 9. Pierwsze uruchomienie i weryfikacja

### Lista kontrolna uruchomienia

1. **[ ]** Podłącz kable RS485 i CAN (bez zasilania!)
2. **[ ]** Sprawdź polaryzację kabli (A→L, B→H, GND→DG)
3. **[ ]** Zainstaluj rezystory terminacyjne 120Ω
4. **[ ]** Włącz zasilanie BMS (wszystkie 5 jednostek)
5. **[ ]** Włącz zasilanie LilyGo T-Connect
6. **[ ]** Sprawdź logi ESPHome (panel → „Logs")
7. **[ ]** Zweryfikuj dane w Home Assistant

### Co sprawdzić w logach:

```
# Prawidłowe działanie — szukaj tych komunikatów:
[I][wifi:...] WiFi Connected!
[I][app:...] ESPHome version X.X.X compiled
[I][seplos_parser:...] Seplos V3 parser initialized
[I][seplos_modbus:...] Seplos Modbus initialized

# Problemy — zwróć uwagę na:
[E][uart:...] Reading from UART timed out
[W][canbus:...] CAN bus error
[E][seplos_modbus:...] No response from BMS
```

### Weryfikacja danych w HA:

Po ~30 sekundach od uruchomienia powinieneś widzieć:

- **Napięcie** wszystkich 5 BMS (indywidualnie i łącznie)
- **Prąd** (ładowania/rozładowania)
- **SoC** (stan naładowania) — średnia ze wszystkich BMS
- **Temperatury** — min/max ze wszystkich BMS
- **Napięcia ogniw** — min/max ze wszystkich BMS
- **Status CAN** — połączenie z inverterem

---

## 10. Monitorowanie i diagnostyka

### Encje w Home Assistant

Po dodaniu urządzenia dostępne będą m.in.:

#### Sensory główne (Virtual BMS)
| Encja | Opis |
|-------|------|
| `sensor.yambms_total_voltage` | Średnie napięcie całkowite |
| `sensor.yambms_current` | Łączny prąd |
| `sensor.yambms_power` | Łączna moc |
| `sensor.yambms_state_of_charge` | Średni SoC [%] |
| `sensor.yambms_min_cell_voltage` | Najniższe napięcie ogniwa |
| `sensor.yambms_max_cell_voltage` | Najwyższe napięcie ogniwa |
| `sensor.yambms_min_temperature` | Najniższa temperatura |
| `sensor.yambms_max_temperature` | Najwyższa temperatura |

#### Sensory per BMS
| Encja (wzorzec) | Opis |
|-----------------|------|
| `sensor.yambms_bms_X_voltage` | Napięcie BMS X |
| `sensor.yambms_bms_X_current` | Prąd BMS X |
| `sensor.yambms_bms_X_soc` | SoC BMS X |
| `sensor.yambms_bms_X_temperature` | Temperatura BMS X |

#### Kontrolki
| Encja | Opis |
|-------|------|
| `select.yambms_canbus_1_can_protocol` | Wybór protokołu CAN |
| `select.yambms_canbus_1_bms_name` | Wybór nazwy BMS dla CAN |
| `switch.yambms_charging` | Włącz/wyłącz ładowanie |
| `switch.yambms_discharging` | Włącz/wyłącz rozładowanie |

### LED statusowe (APA102 na LilyGo T-Connect)

| Kolor LED | Znaczenie |
|-----------|-----------|
| 🟢 Zielony | WiFi połączone, brak komunikacji CAN |
| 🔵 Niebieski | Brak WiFi, brak komunikacji CAN |
| 🟢🔵 Zielono-niebieski (cyan) | WiFi + CAN — wszystko OK! |
| 🔴 Czerwony | Brak WiFi i brak CAN |

### Panel webowy (opcjonalnie)

Jeśli odkomentowałeś `yambms_web_server.yaml`, dostępny jest panel webowy:
- Adres: `http://yambms-seplos.local` (lub adres IP)
- Wyświetla wszystkie sensory i kontrolki w przeglądarce

---

## 11. Troubleshooting — typowe problemy

### ❌ Problem: `packages/yambms/yambms_web_server.yaml is not a valid YAML file`

**Diagnoza:**
Najczęściej to **nie** jest uszkodzony plik `yambms_web_server.yaml`, tylko problem środowiska lub trybu importu:

1. ESPHome jest za stare (poniżej `2025.11.0`)
2. Pomylenie trybu zdalnego i lokalnego pakietów
3. Próba użycia lokalnego `packages/...` bez pełnego katalogu `packages/`

**Kroki naprawy:**
1. Zaktualizuj ESPHome do najnowszej wersji stabilnej
2. Używaj konfiguracji `url + files` (tryb zdalny) — bez lokalnego `packages/`
3. Jeśli wybierasz tryb lokalny, skopiuj cały `packages/` z repo YamBMS
4. Jeśli włączasz panel WWW, dodaj do `secrets.yaml`:
   - `web_server_username`
   - `web_server_password`

### ❌ Problem: Brak danych z Seplos V3 (sniffer)

**Objawy:** Sensory BMS 1-3 pokazują `NaN` lub `Unknown`

**Rozwiązania:**
1. Sprawdź czy kabel RS485 jest podłączony do magistrali MIĘDZY Master a Slave V3
2. Zweryfikuj polaryzację: A+ → L, B- → H
3. Sprawdź rezystor terminacyjny 120Ω
4. Upewnij się, że Seplos V3 komunikują się między sobą (Master-Slave aktywne)
5. Zmień baud rate w konfiguracji: `seplos_baud_rate: '9600'` (jeśli 19200 nie działa)
6. Sprawdź logi: `logger: level: DEBUG` i szukaj komunikatów `seplos_parser`

### ❌ Problem: Brak danych z Seplos V2 (Modbus)

**Objawy:** Sensory BMS 4-5 pokazują `NaN` lub `Unknown`

**Rozwiązania:**
1. Sprawdź adresy DIP switch na każdym BMS V2 (0x01 i 0x02)
2. Zweryfikuj polaryzację kabla RS485: A+ → L, B- → H
3. Spróbuj zamienić A i B (odwrotna polaryzacja)
4. Sprawdź rezystor terminacyjny 120Ω na obu końcach
5. Zmień baud rate: `seplos_modbus_baud_rate: '19200'` (jeśli 9600 nie działa)
6. Sprawdź `bms_protocol_version`: spróbuj `'0x26'` zamiast `'0x20'`
7. W logach szukaj: `seplos_modbus` i `No response`

### ❌ Problem: Brak komunikacji CAN z Victronem

**Objawy:** LED nie miga na cyan, brak danych w Victron

**Rozwiązania:**
1. Sprawdź okablowanie: CAN_H → H, CAN_L → L, GND → DG
2. Zmierz multimetrem rezystancję między CAN_H i CAN_L: powinna być ~60Ω
3. Sprawdź terminację: rezystor 120Ω po stronie T-Connect + terminacja w Victron
4. Upewnij się, że w HA wybrany jest prawidłowy protokół CAN (Victron/PYLON)
5. W Victron (Cerbo GX): sprawdź czy port CAN jest skonfigurowany na „CAN-bus BMS"
6. Sprawdź prędkość: CAN musi działać na 500 kbps po obu stronach

### ❌ Problem: ESP32 restartuje się / crashuje

**Objawy:** Częste restarty, utrata połączenia WiFi

**Rozwiązania:**
1. Upewnij się, że PSRAM jest aktywny (pakiet `board_options_PSRAM_octal_80Mhz.yaml`)
2. Sprawdź zasilanie: użyj stabilnego zasilacza 7-12V DC zamiast USB
3. Zwiększ `bms_update_interval` do `'5s'`
4. Sprawdź logi przed restartem — szukaj `panic`, `assert`, `stack overflow`
5. Zmniejsz liczbę encji (użyj wersji `_minimal` zamiast `_full` dla BMS)

### ❌ Problem: WiFi się rozłącza

**Rozwiązania:**
1. Ustaw statyczny IP (odkomentuj sekcję `wifi: manual_ip:`)
2. Sprawdź siłę sygnału WiFi (RSSI) w logach — poniżej -80 dBm to za słabo
3. Użyj innego kanału WiFi lub sieci 2.4 GHz (ESP32 nie obsługuje 5 GHz)
4. Rozważ użycie anteny zewnętrznej

### ❌ Problem: Nieprawidłowe wartości SoC / napięcia

**Rozwiązania:**
1. Sprawdź `yambms_cell_count` — musi odpowiadać rzeczywistej liczbie ogniw
2. Zweryfikuj `bms_cell_ovp` i `bms_cell_uvp` — muszą odpowiadać ustawieniom BMS
3. Poczekaj kilka cykli ładowania/rozładowania na kalibrację SoC
4. Porównaj dane z YamBMS z odczytami z natywnego oprogramowania Seplos

---

## 12. Przydatne linki

| Zasób | Link |
|-------|------|
| **YamBMS GitHub** | https://github.com/Sleeper85/esphome-yambms |
| **Dokumentacja YamBMS** | https://github.com/Sleeper85/esphome-yambms/tree/main/documents/README |
| **LilyGo T-Connect Wiki** | https://wiki.lilygo.cc/get_started/en/High_speed/T-Connect/T-Connect.html |
| **LilyGo T-Connect Schematy** | https://github.com/Xinyuan-LilyGO/T-Connect |
| **ESPHome Docs** | https://esphome.io/ |
| **Seplos V3 Sniffer** | https://github.com/DpunktS/seplos_v3_sniffer |
| **Seplos V1/V2 ESPHome** | https://github.com/syssi/esphome-seplos-bms |
| **YamBMS Discord** | Sprawdź README projektu na GitHubie |

---

> **📝 Uwaga:** Ta instrukcja została przygotowana dla wersji YamBMS 1.6.0. Przed użyciem sprawdź czy w repozytorium nie pojawiła się nowsza wersja z istotnymi zmianami.

> **⚠️ Zastrzeżenie:** Praca z systemami magazynowania energii wiąże się z ryzykiem. Zawsze przestrzegaj zasad bezpieczeństwa elektrycznego i postępuj zgodnie z dokumentacją producenta baterii i invertera.
