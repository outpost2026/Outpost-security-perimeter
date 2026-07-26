# ESP32 Hardware Architecture — Outpost IoT

**Verze:** 1.0 | **Datum:** 2026-07-26 | **Scope:** Chip selection, board allocation, power budget, UART map
**Zdroje:** Claude_session007, Grok_session010, Grok_session012, Laskakit BOM 202610447

---

## 1. Chip Selection Matrix

| Chip | UART | Deep Sleep | WiFi | ESPHome | Role | Status |
|------|------|-----------|------|---------|------|--------|
| **ESP32 OG** | 3× | 5 µA | ✅ | ✅ | BMS TTL + RS232 bridge + debug | **✅ pridelene** |
| **ESP8266 (D1 Mini)** | 1× | 20 µA | ✅ | ✅ | DS18B20, perimetr, klimaticka | **✅ 1 ks k dispozici** |
| ESP32-C3 SuperMini | 1× | 5 µA | ✅ | ✅ | Low-power satellite (folio<0xE1>k) | ❌ nenakoupen |
| ESP32-S3 | 2× | 7 µA | ✅ | ✅ | Faze 3-4: ML inference | ❌ odlozeno |
| ESP32-CAM | 1× (UART0 konflikt) | — | ✅ | ❌ | Kamerovy node (samostatny) | ❌ nenakoupen |
| ESP32-P4 | ? | 5 mA | ❌ | ❌ | Zamitnut (no ESPHome) | ❌ |
| ESP32-H2 | 1× | — | ❌ | ✅ | Zamitnut (no WiFi) | ❌ |

### Klíčové rozhodnutí

**Počet UART je primární filtr.** ESP32 OG (3× UART) je jediný čip schopný souběžně:
- BMS TTL (JK BMS, 115200 baud, 0x4E 0x57 frame)
- RS232 bridge (MAX3232 → POW-HVM3.2H, 2400 baud Modbus)
- UART0 (flash/debug)

Jakýkoli čip s ≤2 UART (C3, S2, H2, 8266) = blocker pro duální sériové role.

---

## 2. Board Allocation (Fáze 1)

| Board | Qty | Role | UART use | Power | Sleep |
|-------|-----|------|---------|-------|-------|
| **ESP32 mini-kit** | 1 | Modbus master (POW-HVM3.2H) | UART2: GPIO16/17 → MAX3232 | 80 mA active / 10 mA idle | Neni (trvaly monitoring) |
| **D1 Mini #1** | 1 | DS18B20 heatsink (T02) | OneWire D4 (neni UART) | 80 mA / 20 mA | Neni (1s polling) |
| **D1 Mini #2** | 0 | (volny pro perimetr/dalsi) | — | — | — |
| **ESP-01S** | 1 | BMS backup / klimaticka | UART0: GPIO1/3 | 60 mA / 15 mA | Deep sleep ~20 µA |

### Alokace dle Laskakit BOM

| Komponenta | Množství | Kam jde | Stav |
|-----------|---------|---------|------|
| MH-ET LIVE ESP32 MiniKIT | 1 | Modbus master (T03) | k dispozici |
| Wemos D1 Mini ESP8266 | 1 z 3 | DS18B20 (T02) | k dispozici |
| Wemos D1 Mini ESP8266 | 0 z 3 | Security perimetr (T01) | nebyl nalezen? |
| ESP-01/01S | 1 | BMS / klimaticka (T05/T06) | k dispozici |
| ESP32-CAM | 0 | Kamera (T01) | nenakoupen |

---

## 3. UART Pin Map

| UART | ESP32 mini-kit | Pripojeno k | Protocol | Baud |
|------|---------------|------------|----------|------|
| **UART0** | GPIO1(TX)/GPIO3(RX) | USB serial (debug) | Console | 115200 |
| **UART1** | GPIO9(TX)/GPIO10(RX) | JK BMS (TTL 3.3V) | Binary 0x4E 0x57 | 115200 |
| **UART2** | GPIO17(TX)/GPIO16(RX) | MAX3232 → POW-HVM3.2H | Modbus RTU | 2400 |

> **Poznámka.** ESP32 mini-kit ma piny UART1 vyvedeny na GPIO9/10 (ne standardnich 19/22). Overit pred prototype.

### D1 Mini (ESP8266) pin map

| Pin | GPIO | Funkce | Pripojeno |
|-----|------|--------|-----------|
| D0 | 16 | RCWL-0516 (OUT) | Doppler radar |
| D1 | 5 | SCL (I2C) | BMP180 |
| D2 | 4 | SDA (I2C) | BMP180 |
| D3 | 0 | SW-520D / HY-SRF05 TRIG | Otres / ultrazvuk |
| D4 | 2 | OneWire DATA | DS18B20 (integrovany pull-up na shield) |
| D5 | 14 | Buzzer | Bzučák shield |
| RX | 3 | SoftwareSerial RX | MAX3232 (alternativa) |

---

## 4. Power Budget

| Modul | Max (mA) | Avg (mA) | Wh/den (24h) |
|-------|---------|---------|-------------|
| ESP32 mini-kit | 180 | 80 | 9.6 (z 5V) |
| D1 Mini (ESP8266) | 170 | 80 | 9.6 (z 5V) |
| ESP-01S | 80 | 30 | 3.6 (z 3.3V) |
| MAX3232 | 2 | 1 | 0.12 |
| DS18B20 (1x) | 4 | 1 | 0.12 |
| **2x D1 + ESP32 + senzory** | **~436** | **~192** | **~23 Wh/den** |

### Zdroj

- 24V trakční baterie → DC-DC step-down (7-40V→1.2-32V, 8A, 92%)
- Pri 23 Wh/den z 15.12 kWh:
  - **0.15 % kapacity baterie/den** na IoT monitoring
  - Pri 100% SoC: ~660 dní výdrže (teoreticky)
  - Reálně: zanedbatelné (<0.2 % denní kapacity)

> **Kritické:** DC-DC měnič je fyzicky k dispozici (HW-16). Lze použit jako central power supply pro všechny ESP moduly.

---

## 5. Architektonická pravidla

### Pravidlo 1: Jeden UART = jedna sériová role
Pokud čip nemá dostatek hardwarových UART pro všechny plánované sériové linky, použij samostatný čip. SoftwareSerial na ESP8266 je použitelný pro low-speed (2400 baud Modbus), ale ne pro BMS (115200 baud).

### Pravidlo 2: ESPHome first
Všechny nové implementace primárně v ESPHome YAML. Raw C++ / MicroPython pouze pokud ESPHome nepodporuje daný protokol (JK BMS binary frame, nestandardní Modbus varianty).

### Pravidlo 3: Deep sleep jako výchozí stav
Senzorické nody (teplota, perimetr) jsou standardně v deep sleep. Probouzejí se pouze pro měření a odeslání dat. Trvale napájené jsou pouze moduly vyžadující kontinuální komunikaci (Modbus master).

### Pravidlo 4: Galvanické oddělení
RS485 Modbus vyžaduje izolované převodníky (zemní smyčky v 24V systému jsou kritické riziko). JK BMS TTL připojení je možné pouze přes level shifter (3.3V tolerance ESP32 vs BMS port).

---

*Syntetizováno z historických session: Claude_session007 (chip selection), Grok_session010/012 (HW alokace), Laskakit BOM*