# Analiza projektu ESPHome-YamBMS — Kompatybilność z konfiguracją Seplos v2 + v3 na Lilit to Go

**Data analizy:** 11 kwietnia 2026  
**Wersja YamBMS:** 1.6.0  
**Repozytorium:** [https://github.com/Sleeper85/esphome-yambms](https://github.com/Sleeper85/esphome-yambms)

---

## Podsumowanie konfiguracji użytkownika

| Element | Szczegóły |
|---------|-----------|
| **Urządzenie sterujące** | Lilit to Go (4 porty CAN) |
| **Grupa 1** | 3× Seplos v3 (Master-Slave) → 1 wyjście CAN z Master |
| **Grupa 2** | 2× Seplos v2 (Master-Slave) → 1 wyjście CAN z Master |
| **Magistrale CAN** | 2 magistrale (po jednej z każdej grupy) |
| **Inverter** | Victron |
| **Cel** | YamBMS jako agregator danych → jeden wirtualny BMS dla Victrona |

---

## Odpowiedzi na pytania

### 1. Kompatybilność z "Lilit to Go" z 4 portami CAN

#### ⚠️ Odpowiedź: "Lilit to Go" NIE jest oficjalnie wspierane przez projekt YamBMS

**Kluczowe ustalenia:**
- Projekt YamBMS **nie zawiera żadnych referencji** do urządzenia "Lilit to Go" w kodzie, dokumentacji ani konfiguracji.
- "Lilit to Go" nie jest urządzeniem ESP32 — jest to dedykowana płytka CAN gateway. YamBMS jest **firmware ESPHome przeznaczony dla mikrokontrolerów ESP32** (i RP2040).
- Lista oficjalnie obsługiwanych płytek obejmuje m.in.:
  - **LilyGo T-Connect** (ESP32-S3, 3× RS485 + 1× CAN) — to NIE jest "Lilit to Go"
  - **LilyGo T-CAN485** (ESP32, 1× RS485 + 1× CAN)
  - **Waveshare ESP32-S3-RS485-CAN** (1× RS485 + 1× CAN)
  - **M5Stack AtomS3/AtomS3R** + bazy CAN/RS485
  - **espBerry + 2-CH CAN HAT** (ESP32 + 2× MCP2515 CAN)
  - Różne warianty ESP32 DevKit

> **Źródło:** [Supported_devices.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Supported_devices.md)

**Podsumowanie:** YamBMS wymaga urządzenia z mikrokontolerem ESP32 (lub RP2040), na którym można zaflashować firmware ESPHome. "Lilit to Go" nie jest takim urządzeniem.

---

### 2. Obsługa wielu magistrali CAN jednocześnie

#### ✅ Odpowiedź: TAK — YamBMS obsługuje wiele magistrali CAN, ale z ograniczeniami

**Jak to działa:**

YamBMS obsługuje wiele interfejsów CAN na jednym ESP32. Istnieją dwa scenariusze użycia CAN:

| CAN Bus | ID | Przeznaczenie |
|---------|-----|---------------|
| **CAN do BMS** | `canbus_bms_1` | Odczyt danych z BMS (np. Deye CAN BMS) |
| **CAN do invertera** | `canbus_inverter_1` | Wysyłanie zagregowanych danych do invertera |

**Konfiguracja z wieloma CAN:**
- Płytka **espBerry + 2-CH CAN HAT** (Waveshare) wspiera **2 niezależne magistrale CAN** przez MCP2515 na SPI:
  - `canbus_node1` (CS: GPIO 5)
  - `canbus_node2` (CS: GPIO 17)
- Można też użyć **1× ESP32 TWAI (wbudowany CAN)** + **1× MCP2515** na tym samym ESP32.

**Ograniczenia:**
- Standardowy ESP32 ma tylko **1 wbudowany kontroler TWAI** (CAN).
- ESP32-S3 również ma **1 kontroler TWAI**.
- Aby mieć 2+ CAN, potrzebne są **zewnętrzne kontrolery MCP2515** (przez SPI).
- Obsługa **odczytu danych z BMS przez CAN** jest zaimplementowana tylko dla **DEYE BMS CAN protocol** — NIE dla Seplos CAN.

> **Źródło:** [board_ESP32_DevKitC_espBerry_2-CH-CAN.yaml](https://github.com/Sleeper85/esphome-yambms/blob/main/packages/board/board_ESP32_DevKitC_espBerry_2-CH-CAN.yaml), [board_options_itf_canbus_esp32_can.yaml](https://github.com/Sleeper85/esphome-yambms/blob/main/packages/board/board_options_itf_canbus_esp32_can.yaml)

---

### 3. Obsługa protokołów Seplos v2 i v3

#### ✅ Odpowiedź: TAK — oba protokoły są obsługiwane, ale **przez RS485, NIE CAN**

**Seplos v1/v2 (RS485 Modbus):**
- Pełne wsparcie przez komponent `seplos_modbus`
- Każdy BMS adresowany przez DIP switch (`0x01`, `0x02`, `0x03`...)
- Komunikacja przez UART/RS485
- Pliki: `bms_combine_SEPLOS_V1_V2_RS485_modbus.yaml` + `bms_combine_SEPLOS_V1_V2_RS485_bms_full.yaml`

**Seplos v3 (RS485 Sniffer):**
- Wsparcie przez **sniffer RS485** — ESP32 nasłuchuje komunikację Master-Slave
- Wykorzystuje zewnętrzny komponent: [DpunktS/seplos_v3_sniffer](https://github.com/DpunktS/seplos_v3_sniffer)
- Konfigurowalny parametr `seplos_bms_count` (liczba BMS w grupie)
- Pliki: `bms_combine_SEPLOS_V3_RS485_modbus_sniffer.yaml` + `bms_combine_SEPLOS_V3_RS485_sniffer_bms_full.yaml`

**⚠️ WAŻNE:** Projekt **NIE obsługuje odczytu danych z Seplos przez CAN bus!** Seplos jest obsługiwany wyłącznie przez **RS485**. Oznacza to, że nie można podłączyć wyjścia CAN z Master Seplos do YamBMS i odczytać z niego danych.

> **Źródło:** [RP_BMS_example_SEPLOS_V1_V2_RS485.yaml](https://github.com/Sleeper85/esphome-yambms/blob/main/examples/single-node/RP_BMS_example_SEPLOS_V1_V2_RS485.yaml), [RP_BMS_example_SEPLOS_V3_RS485_sniffer.yaml](https://github.com/Sleeper85/esphome-yambms/blob/main/examples/single-node/RP_BMS_example_SEPLOS_V3_RS485_sniffer.yaml)

---

### 4. Agregacja danych i prezentacja jako jeden wirtualny BMS

#### ✅ Odpowiedź: TAK — to GŁÓWNA funkcja YamBMS

YamBMS został zaprojektowany dokładnie do tego celu. Mechanizm działa następująco:

1. **Zbieranie danych** z wielu BMS (dowolny mix modeli/protokołów)
2. **Agregacja** (combine) — obliczanie:
   - **Napięcie**: średnia ze wszystkich BMS
   - **Prąd**: suma ze wszystkich BMS
   - **Moc**: suma ze wszystkich BMS
   - **SoC**: obliczany na podstawie sumy pojemności pozostałej / sumy pojemności całkowitej
   - **SoH**: średnia ze wszystkich BMS
   - **Min/Max napięcie ogniwa**: globalne minimum/maksimum ze wszystkich BMS
   - **Min/Max temperatura**: globalne wartości
   - **Prądy ładowania/rozładowania**: suma OCP * 0.9, nie więcej niż ustawiony max
3. **Wysyłanie** zagregowanych danych do invertera przez CAN bus w protokole **Victron** (lub PYLON, SMA, LuxPower)

**Obsługiwane protokoły CAN do invertera Victron:**
- Protokół `Victron` — dedykowany, pełny, z ramkami:
  - `0x35E` — nazwa producenta
  - `0x351` — CVL, CCL, DCL, DVL
  - `0x355` — SoC, SoH
  - `0x356` — napięcie, prąd, temperatura
  - `0x35A` — alarmy i ostrzeżenia
  - `0x372` — informacje o modułach baterii
  - `0x373` — min/max napięcia i temperatury ogniw
  - `0x374-0x377` — ID ogniw min/max
  - `0x379` — pojemność zainstalowana
  - `0x382` — identyfikacja produktu
  - `0x360` — wymuszenie ładowania

Victron MultiPlus-II jest na liście oficjalnie potwierdzonych inwerterów.

> **Źródło:** [yambms_combine.yaml](https://github.com/Sleeper85/esphome-yambms/blob/main/packages/yambms/yambms_combine.yaml), [yambms_canbus.yaml](https://github.com/Sleeper85/esphome-yambms/blob/main/packages/yambms/yambms_canbus.yaml), [YamBMS_functions.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/YamBMS_functions.md)

---

### 5. Konfiguracja systemu — jak to zrobić?

#### ⚠️ Wymagana zmiana podejścia — RS485 zamiast CAN do odczytu BMS

Ponieważ YamBMS odczytuje dane z Seplos **wyłącznie przez RS485** (nie CAN), proponowana konfiguracja musi zostać zmodyfikowana:

**Zamiast:** Lilit to Go → CAN z Master Seplos  
**Należy:** ESP32-S3 z RS485 → podpięcie do magistrali RS485 każdej grupy Seplos

#### Rekomendowana konfiguracja sprzętowa

| Element | Rekomendacja |
|---------|-------------|
| **Płytka** | **LilyGo T-Connect** (ESP32-S3, 8MB PSRAM, 3× RS485, 1× CAN) |
| **RS485 Port 1** | → magistrala RS485 grupy Seplos v3 (sniffer) |
| **RS485 Port 2** | → magistrala RS485 grupy Seplos v2 (modbus) |
| **CAN Port** | → inverter Victron |
| **RS485 Port 3** | wolny (zapasowy lub multi-node) |

#### Pliki konfiguracyjne do zmodyfikowania

Główny plik YAML (np. `YamBMS_Seplos_Mixed.yaml`) powinien zawierać:

```yaml
# Bazowy przykład konfiguracji

logger:
  level: INFO

ota:
  - platform: esphome

api:
  reboot_timeout: 0s

substitutions:
  friendly_name: 'YamBMS'
  hostname: 'yambms-seplos'
  name: ''
  yambms_id: 'yambms1'
  yambms_name: 'YamBMS 1'
  yambms_update_interval: '1s'
  yambms_input_number_mode: 'slider'
  yambms_battery_chemistry: '1'  # LFP
  yambms_cell_count: '16'
  yambms_bulk_v: '55.2'
  yambms_float_v: '53.6'
  yambms_rebulk_v: '52.8'
  yambms_eoc_timer: '30'
  yambms_cutoff_timer: '60'
  yambms_max_requested_charge_current: '300'
  yambms_max_requested_discharge_current: '300'
  # ... (tabele temperature-based current - jak w przykładach)
  shunt_update_interval: '3s'
  bms_update_interval: '3s'

packages:

  # ============================================
  # BOARD: LilyGo T-Connect
  # ============================================
  board:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      - path: 'packages/board/board_ESP32-S3_LilyGo-T-Connect.yaml'

  # ============================================
  # BOARD OPTIONS: 2x UART (RS485) + 1x CAN
  # ============================================
  board_options:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      # UART 1 dla Seplos V3 (RS485 Port 1)
      - path: 'packages/board/board_options_itf_uart_esp_1.yaml'
      # UART 2 dla Seplos V2 (RS485 Port 2)
      - path: 'packages/board/board_options_itf_uart_esp_2.yaml'
      # CAN do invertera Victron (Port 4 = CAN)
      - path: 'packages/board/board_options_itf_canbus_esp32_can.yaml'
        vars:
          canbus_node_id: 'canbus_inverter_1'

  # ============================================
  # SEPLOS V3 — Sniffer na RS485 Port 1
  # Podłączenie: RS485 sniffer do magistrali
  # komunikacyjnej Master-Slave grupy Seplos V3
  # ============================================
  bms_seplos_v3:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      # Parser Seplos V3
      - path: 'packages/bms/bms_combine_SEPLOS_V3_RS485_modbus_sniffer.yaml'
        vars:
          seplos_uart_id: 'uart_esp_1'
          seplos_baud_rate: '19200'
          seplos_bms_count: '3'     # 3 BMS Seplos V3
          seplos_update_interval: '3'
      # BMS 1 (Seplos V3 Master)
      - path: 'packages/bms/bms_combine_SEPLOS_V3_RS485_sniffer_bms_full.yaml'
        vars:
          bms_id: '1'
          bms_prefix: 'bms0'       # NIE ZMIENIAĆ!
          bms_name: 'Seplos-V3 Master'
          bms_cell_ovp: '3.650'
          bms_cell_uvp: '2.800'
          bms_balance_trigger_voltage: '0.010'
      # BMS 2 (Seplos V3 Slave 1)
      - path: 'packages/bms/bms_combine_SEPLOS_V3_RS485_sniffer_bms_full.yaml'
        vars:
          bms_id: '2'
          bms_prefix: 'bms1'       # NIE ZMIENIAĆ!
          bms_name: 'Seplos-V3 Slave 1'
          bms_cell_ovp: '3.650'
          bms_cell_uvp: '2.800'
          bms_balance_trigger_voltage: '0.010'
      # BMS 3 (Seplos V3 Slave 2)
      - path: 'packages/bms/bms_combine_SEPLOS_V3_RS485_sniffer_bms_full.yaml'
        vars:
          bms_id: '3'
          bms_prefix: 'bms2'       # NIE ZMIENIAĆ!
          bms_name: 'Seplos-V3 Slave 2'
          bms_cell_ovp: '3.650'
          bms_cell_uvp: '2.800'
          bms_balance_trigger_voltage: '0.010'

  # ============================================
  # SEPLOS V2 — Modbus na RS485 Port 2
  # Podłączenie: RS485 do magistrali
  # komunikacyjnej grupy Seplos V2
  # ============================================
  bms_seplos_v2:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      # Modbus Seplos V2
      - path: 'packages/bms/bms_combine_SEPLOS_V1_V2_RS485_modbus.yaml'
        vars:
          seplos_modbus_uart_id: 'uart_esp_2'
          seplos_modbus_baud_rate: '9600'
      # BMS 4 (Seplos V2 Master)
      - path: 'packages/bms/bms_combine_SEPLOS_V1_V2_RS485_bms_full.yaml'
        vars:
          bms_id: '4'              # KONTYNUACJA numeracji!
          bms_name: 'Seplos-V2 Master'
          bms_address: '0x01'
          bms_protocol_version: '0x20'
          bms_max_charge_current: '100'
          bms_max_discharge_current: '100'
          bms_cell_ovp: '3.650'
          bms_cell_uvp: '2.800'
          bms_balance_trigger_voltage: '0.010'
      # BMS 5 (Seplos V2 Slave)
      - path: 'packages/bms/bms_combine_SEPLOS_V1_V2_RS485_bms_full.yaml'
        vars:
          bms_id: '5'              # KONTYNUACJA numeracji!
          bms_name: 'Seplos-V2 Slave'
          bms_address: '0x02'
          bms_protocol_version: '0x20'
          bms_max_charge_current: '100'
          bms_max_discharge_current: '100'
          bms_cell_ovp: '3.650'
          bms_cell_uvp: '2.800'
          bms_balance_trigger_voltage: '0.010'

  # ============================================
  # YamBMS Core
  # ============================================
  yambms:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      - path: 'packages/yambms/yambms.yaml'

  # ============================================
  # CAN bus → Victron Inverter
  # ============================================
  canbus:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      - path: 'packages/yambms/yambms_canbus.yaml'
        vars:
          canbus_id: 'canbus1'
          canbus_name: 'CAN Victron'
          canbus_node_id: 'canbus_inverter_1'
          canbus_light_id: 'esp_light'
          canbus_link_timer: '5s'

  # ============================================
  # DEBUG
  # ============================================
  debug:
    url: https://github.com/Sleeper85/esphome-yambms
    ref: main
    refresh: 60min
    files:
      - path: packages/base/device_debug_ESP32_PSRAM.yaml
        vars:
          debug_name: 'Debug'
          debug_update_interval: '5s'
          debug_psram_size: '8388608'
```

#### Schemat połączeń

```
┌─────────────────────────────────────────────────────┐
│              LilyGo T-Connect (ESP32-S3)            │
│                                                     │
│  RS485 Port 1 (UART 1)  ←──── Seplos V3 RS485 bus  │
│  [IO4/IO5]                     (Master ↔ Slaves)    │
│                                sniffer mode          │
│                                                     │
│  RS485 Port 2 (UART 2)  ←──── Seplos V2 RS485 bus  │
│  [IO6/IO7]                     (Master ↔ Slaves)    │
│                                modbus mode           │
│                                                     │
│  CAN Port (TWAI)         ────→ Victron Inverter     │
│  [IO9/IO10]                    (protokół Victron)    │
│                                                     │
│  RS485 Port 3 (UART 3)        wolny / zapasowy      │
│  [IO17/IO18]                                        │
└─────────────────────────────────────────────────────┘
```

#### Konfiguracja Victron:
- W menu invertera Victron ustaw: **"CAN-bus BMS LV (500 kbit/s)"**
- W YamBMS wybierz protokół: **"Victron"**

---

## Potencjalne ograniczenia i problemy

### 🔴 Krytyczne

1. **"Lilit to Go" nie jest kompatybilne** — to nie jest urządzenie ESP32, więc nie można na nim uruchomić YamBMS. Konieczne jest użycie innej płytki (rekomendacja: LilyGo T-Connect).

2. **Brak obsługi Seplos CAN w YamBMS** — YamBMS odczytuje dane z Seplos **wyłącznie przez RS485**. Wyjścia CAN z Master Seplos nie mogą być bezpośrednio użyte do odczytu danych przez YamBMS. Jedyny BMS obsługiwany przez CAN to **DEYE BMS**.

3. **Seplos V3 — tryb sniffer** — Komponent `seplos_v3_sniffer` działa w trybie pasywnego nasłuchu magistrali RS485 między Master a Slave'ami. ESP32 musi mieć dostęp do fizycznej magistrali RS485 wewnątrz grupy Seplos V3. Należy upewnić się, że Master Seplos V3 odpytuje Slave'y po RS485 (a nie tylko wewnętrznie).

### 🟡 Potencjalne problemy

4. **Dwie niezależne magistrale RS485** — każda wymaga osobnego interfejsu UART. LilyGo T-Connect z 3× RS485 jest idealnym rozwiązaniem. Inne płytki (np. Waveshare RS485-CAN z 1× RS485) wymagałyby konfiguracji multi-node.

5. **Różne prędkości transmisji** — Seplos V2 zazwyczaj używa 9600 baud, Seplos V3 — 19200 baud. Każdy UART musi być skonfigurowany z odpowiednią prędkością.

6. **Numeracja BMS** — BMS muszą być numerowane **sekwencyjnie od 1**. W przykładzie: V3 = BMS 1-3, V2 = BMS 4-5. Nie można pominąć numerów.

7. **Zużycie pamięci** — 5 BMS to znaczne obciążenie. ESP32-S3 z 8MB PSRAM (jak LilyGo T-Connect) jest **konieczny**. Włączenie Web Servera lub BLE przy takiej liczbie BMS może spowodować niestabilność.

8. **Kompatybilność ogniw** — Jeśli Seplos V2 i V3 mają **różną liczbę ogniw szeregowo** (np. 15S vs 16S), YamBMS użyje globalnego min/max napięcia ogniwa, ale parametry `yambms_cell_count`, `yambms_bulk_v`, `yambms_float_v` muszą być jednakowe. Mieszanie baterii o różnej konfiguracji (np. 15S i 16S) jest **niezalecane**.

### 🟢 Informacyjne

9. **Seplos V3 sniffer** jest komponentem zewnętrznym ([DpunktS/seplos_v3_sniffer](https://github.com/DpunktS/seplos_v3_sniffer)), nie jest częścią oficjalnego ESPHome. Może wymagać osobnej weryfikacji kompatybilności.

10. **Izolacja galwaniczna** — LilyGo T-Connect ma izolowane porty RS485 i CAN (do 2500V), co jest ważne w instalacjach bateryjnych.

---

## Alternatywne podejście: Konfiguracja multi-node

Jeśli nie jest możliwy fizyczny dostęp do magistral RS485 obu grup Seplos, można użyć konfiguracji **multi-node**:

```
┌──────────────┐  RS485    ┌──────────────┐  RS485    ┌──────────────┐
│ ESP32 Node 2 │  Seplos   │ ESP32 Node 3 │  Seplos   │ ESP32 Node 1 │
│ (Seplos V3)  │ ←─RS485─→ │ (Seplos V2)  │           │  (YamBMS)    │
│ modbus server│           │ modbus server│           │ modbus client│
└──────┬───────┘           └──────┬───────┘           └──────┬───────┘
       │ RS485 YamBMS bus         │                          │ CAN
       └──────────────────────────┘──────────────────────────┘
                              ↕                              ↓
                    Dedykowana magistrala           Victron Inverter
                    RS485 dla YamBMS
```

Każdy node jest osobnym ESP32, dane przesyłane są przez dedykowaną magistralę RS485 (modbus) do Node 1 (YamBMS), który agreguje je i wysyła do invertera.

---

## Rekomendacje końcowe

| # | Rekomendacja |
|---|-------------|
| 1 | **Nie używaj "Lilit to Go"** — zastąp go płytką **LilyGo T-Connect** (ESP32-S3, 3× RS485, 1× CAN, 8MB PSRAM) |
| 2 | **Podłącz Seplos przez RS485**, nie CAN — YamBMS nie obsługuje Seplos CAN |
| 3 | **Seplos V3**: użyj trybu sniffer — podłącz ESP32 do magistrali RS485 Master↔Slave |
| 4 | **Seplos V2**: użyj trybu modbus — każdy BMS ma adres DIP switch |
| 5 | **Użyj protokołu Victron** na CAN bus do invertera Victron |
| 6 | **Numeruj BMS sekwencyjnie** (1, 2, 3, 4, 5) niezależnie od modelu |
| 7 | **Upewnij się**, że wszystkie baterie mają tę samą konfigurację ogniw (np. 16S LFP) |
| 8 | **Rozważ multi-node** jeśli nie masz fizycznego dostępu do magistral RS485 obu grup |
| 9 | **Przetestuj** na jednej grupie BMS przed dodaniem drugiej |
| 10 | **Włącz PSRAM** — konieczne przy 5 BMS |

---

## Linki do dokumentacji

| Temat | Link |
|-------|------|
| README główne | [README.md](https://github.com/Sleeper85/esphome-yambms/blob/main/README.md) |
| Obsługiwane urządzenia | [Supported_devices.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Supported_devices.md) |
| Tworzenie YAML | [YamBMS_RP_YAML.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/YamBMS_RP_YAML.md) |
| Funkcje YamBMS | [YamBMS_functions.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/YamBMS_functions.md) |
| CAN bus | [Interface_CAN_bus.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Interface_CAN_bus.md) |
| RS485 bus | [Interface_RS485_bus.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Interface_RS485_bus.md) |
| LilyGo T-Connect | [Board_LilyGo_T-Connect.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Board_LilyGo_T-Connect.md) |
| Hardware & schematy | [Hardware_and_schematic_instructions.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Hardware_and_schematic_instructions.md) |
| Logika ładowania | [Charging_logic.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Charging_logic.md) |
| Zachowanie YamBMS | [YamBMS_behavior.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/YamBMS_behavior.md) |
| Troubleshooting | [Troubleshooting.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Troubleshooting.md) |
| Seplos V3 Sniffer | [DpunktS/seplos_v3_sniffer](https://github.com/DpunktS/seplos_v3_sniffer) |
| Przykłady single-node | [examples/single-node/](https://github.com/Sleeper85/esphome-yambms/tree/main/examples/single-node/) |
| Przykłady multi-node | [examples/multi-node/](https://github.com/Sleeper85/esphome-yambms/tree/main/examples/multi-node/) |
| Forum DIY Solar | [Temat YamBMS](https://diysolarforum.com/threads/yambms-jk-bms-can-with-new-cut-off-charging-logic-open-source.79325/) |
| Changelog | [Changelog.md](https://github.com/Sleeper85/esphome-yambms/blob/main/documents/README/Changelog.md) |
