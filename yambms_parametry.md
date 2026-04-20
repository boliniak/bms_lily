# 🔧 YamBMS — Parametry do dostosowania

## Konfiguracja: LilyGo T-Connect + 3× Seplos V3 + 2× Seplos V2 + Victron

---

## Spis treści

1. [Parametry wymagające dostosowania (⚠️ OBOWIĄZKOWE)](#1-parametry-wymagające-dostosowania--obowiązkowe)
2. [Parametry sieciowe](#2-parametry-sieciowe)
3. [Parametry ładowania YamBMS](#3-parametry-ładowania-yambms)
4. [Parametry BMS — Seplos V3 (sniffer)](#4-parametry-bms--seplos-v3-sniffer)
5. [Parametry BMS — Seplos V2 (Modbus)](#5-parametry-bms--seplos-v2-modbus)
6. [Parametry CAN bus](#6-parametry-can-bus)
7. [Parametry systemowe](#7-parametry-systemowe)
8. [Tabele ograniczeń temperaturowych](#8-tabele-ograniczeń-temperaturowych)
9. [Parametry zmieniane w Home Assistant (runtime)](#9-parametry-zmieniane-w-home-assistant-runtime)

---

## 1. Parametry wymagające dostosowania (⚠️ OBOWIĄZKOWE)

Te parametry **MUSISZ** zweryfikować i dostosować do swojego systemu przed pierwszym uruchomieniem:

| Parametr | Aktualna wartość | Co oznacza | Jak dostosować |
|----------|-----------------|------------|----------------|
| `wifi_ssid` | *(w secrets.yaml)* | Nazwa sieci WiFi | Wpisz nazwę swojej sieci WiFi 2.4 GHz |
| `wifi_password` | *(w secrets.yaml)* | Hasło WiFi | Wpisz hasło do sieci WiFi |
| `yambms_cell_count` | `'16'` | Liczba ogniw w szeregu | Zmień jeśli twoje baterie mają inną konfigurację (np. 15S, 8S) |
| `yambms_bulk_v` | `'55.2'` | Napięcie ładowania Bulk [V] | Oblicz: `liczba_ogniw × napięcie_bulk_per_cell` |
| `yambms_float_v` | `'53.6'` | Napięcie Float [V] | Oblicz: `liczba_ogniw × napięcie_float_per_cell` |
| `yambms_rebulk_v` | `'52.8'` | Napięcie Rebulk [V] | Oblicz: `liczba_ogniw × napięcie_rebulk_per_cell` |
| `bms_cell_ovp` | `'3.650'` | Zabezpieczenie OVP ogniwa [V] | Sprawdź w ustawieniach swojego BMS |
| `bms_cell_uvp` | `'2.800'` | Zabezpieczenie UVP ogniwa [V] | Sprawdź w ustawieniach swojego BMS |
| `bms_max_charge_current` | `'100'` | Maks. prąd ładowania per BMS V2 [A] | Sprawdź specyfikację swoich baterii |
| `bms_max_discharge_current` | `'100'` | Maks. prąd rozładowania per BMS V2 [A] | Sprawdź specyfikację swoich baterii |

---

## 2. Parametry sieciowe

### secrets.yaml

| Parametr | Wartość domyślna | Opis | Zalecenie |
|----------|-----------------|------|-----------|
| `wifi_ssid` | — | Nazwa sieci WiFi | Użyj sieci 2.4 GHz (ESP32 nie obsługuje 5 GHz) |
| `wifi_password` | — | Hasło do WiFi | Silne hasło, min. 8 znaków |
| `domain` | `.local` | Domena mDNS | Zazwyczaj `.local`, nie zmieniaj |

### Identyfikacja urządzenia

| Parametr | Wartość domyślna | Opis | Zalecenie |
|----------|-----------------|------|-----------|
| `friendly_name` | `'YamBMS'` | Prefiks nazw encji w HA | Zmień jeśli masz wiele instancji YamBMS |
| `hostname` | `'yambms-seplos'` | Nazwa DNS urządzenia | Musi być unikalna w sieci! Bez spacji, małe litery |
| `name` | `''` | Dodatkowy prefiks encji | Można zostawić pusty |

### Statyczny IP (opcjonalnie)

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `static_ip` | zakomentowany | Stały adres IP | Ustaw jeśli masz problemy z mDNS lub WiFi |
| `gateway` | zakomentowany | Brama domyślna | Adres routera |
| `subnet` | `255.255.255.0` | Maska podsieci | Zazwyczaj nie zmieniaj |

---

## 3. Parametry ładowania YamBMS

### Główne parametry napięciowe

| Parametr | Wartość | Opis | Obliczanie | Zalecenie LFP 16S |
|----------|---------|------|------------|-------------------|
| `yambms_battery_chemistry` | `'1'` | Chemia ogniw | 1=LFP, 2=Li-ion, 3=LTO | `'1'` dla LFP |
| `yambms_cell_count` | `'16'` | Liczba ogniw | Policz fizycznie | 16 dla 48V |
| `yambms_bulk_v` | `'55.2'` | Napięcie Bulk | 16 × 3.45V | 54.4–55.2V |
| `yambms_float_v` | `'53.6'` | Napięcie Float | 16 × 3.35V | 52.8–53.6V |
| `yambms_rebulk_v` | `'52.8'` | Napięcie Rebulk | 16 × 3.30V | 51.2–52.8V |

> **📝 Tabela napięć per ogniwo LFP (typowe):**
> | Parametr | Na ogniwo | 8S (24V) | 16S (48V) |
> |----------|-----------|----------|-----------|
> | Bulk | 3.45V | 27.6V | 55.2V |
> | Float | 3.35V | 26.8V | 53.6V |
> | Rebulk | 3.30V | 26.4V | 52.8V |

### Timery ładowania

| Parametr | Wartość | Jednostka | Opis | Zalecenie |
|----------|---------|-----------|------|-----------|
| `yambms_eoc_timer` | `'30'` | minuty | Maks. czas fazy cut-off | 15–60 min |
| `yambms_cutoff_timer` | `'60'` | sekundy | Czas stabilizacji końca ładowania | 30–120 s |

### Limity prądowe

| Parametr | Wartość | Jednostka | Opis | Zalecenie |
|----------|---------|-----------|------|-----------|
| `yambms_max_requested_charge_current` | `'500'` | A | Maks. żądany prąd ładowania (suma) | ≤ suma prądów BMS, ≤ limit invertera |
| `yambms_max_requested_discharge_current` | `'500'` | A | Maks. żądany prąd rozładowania (suma) | ≤ suma prądów BMS, ≤ limit invertera |

> **💡 Wyjaśnienie:** YamBMS automatycznie skaluje żądany prąd proporcjonalnie do liczby aktywnych BMS. Wartość `500A` to górny limit — rzeczywisty prąd będzie niższy, zależny od BMS i temperatury.

---

## 4. Parametry BMS — Seplos V3 (sniffer)

### Parser sniffer (wspólny)

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `seplos_uart_id` | `'uart_esp_1'` | Interfejs UART | NIE ZMIENIAJ (Port 1) |
| `seplos_baud_rate` | `'19200'` | Prędkość komunikacji | 9600 lub 19200 (zgodnie z BMS) |
| `seplos_bms_count` | `'3'` | Liczba BMS V3 | Zmień jeśli masz inną liczbę |
| `seplos_update_interval` | `'3'` | Interwał odświeżania [s] | 3–10s, nie mniej niż 3 |

### Parametry per BMS V3

| Parametr | BMS 1 | BMS 2 | BMS 3 | Opis |
|----------|-------|-------|-------|------|
| `bms_id` | `'1'` | `'2'` | `'3'` | Numer porządkowy (ciągły!) |
| `bms_prefix` | `'bms0'` | `'bms1'` | `'bms2'` | ⚠️ NIE ZMIENIAJ! Indeks parsera |
| `bms_name` | `'Seplos V3 BMS 1'` | `'Seplos V3 BMS 2'` | `'Seplos V3 BMS 3'` | Nazwa wyświetlana |
| `bms_cell_ovp` | `'3.650'` | `'3.650'` | `'3.650'` | OVP per ogniwo [V] |
| `bms_cell_uvp` | `'2.800'` | `'2.800'` | `'2.800'` | UVP per ogniwo [V] |
| `bms_balance_trigger_voltage` | `'0.010'` | `'0.010'` | `'0.010'` | Próg balansowania [V] |

> **⚠️ WAŻNE:** Prefiksy `bms0`, `bms1`, `bms2` odpowiadają indeksom w parserze Seplos V3 sniffer. Zmiana tych wartości spowoduje brak odczytu danych!

---

## 5. Parametry BMS — Seplos V2 (Modbus)

### Parser Modbus (wspólny)

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `seplos_modbus_uart_id` | `'uart_esp_2'` | Interfejs UART | NIE ZMIENIAJ (Port 2) |
| `seplos_modbus_baud_rate` | `'9600'` | Prędkość komunikacji | 9600 (standard V2) lub 19200 |

### Parametry per BMS V2

| Parametr | BMS 4 | BMS 5 | Opis | Uwagi |
|----------|-------|-------|------|-------|
| `bms_id` | `'4'` | `'5'` | Numer porządkowy | Kontynuacja po V3! |
| `bms_name` | `'Seplos V2 BMS 4'` | `'Seplos V2 BMS 5'` | Nazwa wyświetlana | Dowolna |
| `bms_address` | `'0x01'` | `'0x02'` | Adres Modbus | ⚠️ Zgodny z DIP switch! |
| `bms_protocol_version` | `'0x20'` | `'0x20'` | Wersja protokołu | 0x20=Seplos, 0x26=Boqiang |
| `bms_max_charge_current` | `'100'` | `'100'` | Maks. prąd ładowania [A] | Wg specyfikacji baterii |
| `bms_max_discharge_current` | `'100'` | `'100'` | Maks. prąd rozładowania [A] | Wg specyfikacji baterii |
| `bms_cell_ovp` | `'3.650'` | `'3.650'` | OVP per ogniwo [V] | Zgodne z ustawieniami BMS |
| `bms_cell_uvp` | `'2.800'` | `'2.800'` | UVP per ogniwo [V] | Zgodne z ustawieniami BMS |
| `bms_balance_trigger_voltage` | `'0.010'` | `'0.010'` | Próg balansowania [V] | 5–20 mV typowo |

> **📝 Uwaga o `bms_protocol_version`:**
> - `0x20` — standardowy protokół Seplos (najczęściej używany)
> - `0x26` — protokół Boqiang (niektóre klony/OEM)
> - Jeśli BMS nie odpowiada, spróbuj zmienić na `0x26`

---

## 6. Parametry CAN bus

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `canbus_id` | `'canbus1'` | ID instancji CAN w YamBMS | NIE ZMIENIAJ |
| `canbus_name` | `'CANBUS Victron'` | Nazwa wyświetlana | Dowolna |
| `canbus_node_id` | `'canbus_inverter_1'` | ID węzła CAN | NIE ZMIENIAJ |
| `canbus_light_id` | `'esp_light'` | ID LED statusowego | NIE ZMIENIAJ |
| `canbus_link_timer` | `'5s'` | Timeout odpowiedzi invertera | 3–10s |
| `esp32_canbus_bitrate` | `'500kbps'` | Prędkość CAN bus | 500kbps (standard Victron) |

> **💡 `canbus_link_timer`:** Jeśli inverter nie odpowie frame 0x305 w ciągu 5 sekund, link CAN zostanie uznany za zerwany. Zwiększ do `'10s'` jeśli masz niestabilne połączenie.

---

## 7. Parametry systemowe

### Interwały odświeżania

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `yambms_update_interval` | `'1s'` | Częstotliwość agregacji Virtual BMS | 1–3s |
| `bms_update_interval` | `'3s'` | Częstotliwość odpytywania BMS | ⚠️ NIE mniej niż 3s! |
| `shunt_update_interval` | `'3s'` | Częstotliwość odpytywania shunta | 3–10s (jeśli używany) |
| `debug_update_interval` | `'5s'` | Częstotliwość danych debugowych | 5–30s |

### Logger

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `level` | `INFO` | Poziom logowania | INFO na co dzień, DEBUG przy problemach |

> **⚠️ Poziomy logowania:**
> - `ERROR` — tylko błędy
> - `WARN` — ostrzeżenia i błędy
> - `INFO` — informacje ogólne (zalecane)
> - `DEBUG` — szczegółowe informacje (do diagnostyki)
> - `VERBOSE` — wszystko (dużo danych, może spowalniać)

### Tryb wyświetlania

| Parametr | Wartość | Opis | Zalecenie |
|----------|---------|------|-----------|
| `yambms_input_number_mode` | `'slider'` | Sposób wyświetlania suwaków w HA | `'slider'` lub `'box'` |

---

## 8. Tabele ograniczeń temperaturowych

### Tabela ładowania (`yambms_charging_rate_table`)

Współczynnik ładowania stosowany do pojemności baterii w zależności od temperatury:

| Temperatura [°C] | Współczynnik | Efekt dla 280Ah | Opis |
|-------------------|-------------|-----------------|------|
| < 0°C | 0.00 | 0A (brak ładowania) | ⛔ Ładowanie zablokowane |
| 0°C | 0.05 | 14A | Minimalne ładowanie |
| 5°C | 0.12 | 33.6A | Ostrożne ładowanie |
| 10°C | 0.30 | 84A | Umiarkowane ładowanie |
| 20–55°C | 0.50 | 140A | Normalne ładowanie |
| ≥ 60°C | 0.00 | 0A (brak ładowania) | ⛔ Przegrzanie! |

### Tabela rozładowania (`yambms_discharging_rate_table`)

| Temperatura [°C] | Współczynnik | Efekt dla 280Ah | Opis |
|-------------------|-------------|-----------------|------|
| < -30°C | 0.00 | 0A (brak rozładowania) | ⛔ Za zimno |
| -20°C do 55°C | 0.50 | 140A | Normalne rozładowanie |
| ≥ 60°C | 0.00 | 0A (brak rozładowania) | ⛔ Przegrzanie! |

> **📝 Jak modyfikować:** Każdy wpis to para `{ temperatura, współczynnik }`. Współczynnik 0.50 oznacza 0.5C rate (połowa pojemności w amperach). Możesz zwiększyć do 1.0 (1C) jeśli twoje baterie na to pozwalają.

---

## 9. Parametry zmieniane w Home Assistant (runtime)

Te parametry można zmieniać **po uruchomieniu** systemu, bezpośrednio w interfejsie Home Assistant:

| Encja w HA | Opis | Wartości |
|------------|------|---------|
| **CAN Protocol** | Protokół komunikacji z inverterem | PYLON 1.2, PYLON V2, SMA, Victron, LuxPower, Deye PCS |
| **BMS Name** | Nazwa BMS wysyłana do invertera | PYLON, GOODWE, SEPLOS, Victron, itp. |
| **Charging** | Włączenie/wyłączenie ładowania | ON/OFF |
| **Discharging** | Włączenie/wyłączenie rozładowania | ON/OFF |
| **Auto CCL** | Automatyczny limit prądu ładowania | ON/OFF |
| **Auto DCL** | Automatyczny limit prądu rozładowania | ON/OFF |
| **Auto CVL** | Automatyczny limit napięcia ładowania | ON/OFF |
| **Temperature Limitation** | Ograniczenie temperaturowe prądów | ON/OFF |
| **EOC Timer** | Timer końca ładowania | ON/OFF |

> **💡 Dla Victron:** Zazwyczaj najlepiej działa:
> - CAN Protocol: **„PYLON 1.2"** lub **„Victron"**
> - BMS Name: **„PYLON"** lub **„Victron"**
> - Przetestuj obie kombinacje i wybierz tę, która daje stabilną komunikację

---

> **📝 Uwaga:** Wszystkie wartości napięć, prądów i temperatur powinny odpowiadać specyfikacji twoich konkretnych baterii. Podane tutaj wartości są typowe dla ogniw LiFePO4 (LFP) ale mogą się różnić w zależności od producenta i modelu.
