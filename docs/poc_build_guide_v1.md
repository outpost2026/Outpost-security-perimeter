# IoT PoC Build Guide — Outpost 2026

**Verze:** 1.0 | **Datum:** 2026-07-26
**Účel:** Podrobný návod na sestavení prvního IoT PoC z **existujícího a fyzicky dostupného HW** (Laskakit obj. 202610447 + doma nalezené komponenty).
**Restrikce:** Pouze HW, který je aktuálně na stole / v Outpostu. Bez čekání na objednávky.

---

## Available HW Inventory (fyzicky k dispozici)

| ID | Komponenta | Množství | Varianta |
|----|-----------|----------|----------|
| HW-01 | Wemos D1 Mini ESP8266 WiFi modul | **1 ks** | V1, V3, V4, V5 |
| HW-02 | MH-ET LIVE ESP32 MiniKIT (D1 mini kompat.) | **1 ks** | V2 |
| HW-03 | Espressif ESP-01/01S | **1 ks** | V6 (čeká na kabel) |
| HW-04 | DS18B20 shield pro D1 mini | **2 ks** | V1, V5 |
| HW-05 | DS18B20 vodotěsné čidlo 1m | **5 ks** | V1, V5 |
| HW-06 | BMP180 shield pro D1 mini | **1 ks** | V3, V5 |
| HW-07 | MAX3232 TTL→RS232 převodník (female DB9) | **1 ks** | V2 |
| HW-08 | RJ45 patch kabel (již odstřižený, připravený) | **1 ks** | V2 |
| HW-09 | RCWL-0516 mikrovlnný Doppler radar | **2 ks** | V4 |
| HW-10 | HW-MS03 mikrovlnný Doppler radar | **1 ks** | V4 |
| HW-11 | HY-SRF05 ultrazvukový měřič vzdálenosti | **1 ks** | V4 |
| HW-12 | SW-520D otřesové čidlo | **5 ks** | V4 |
| HW-13 | D1 mini bzučák shield | **1 ks** | V1, V4, V5 |
| HW-14 | D1 mini 1-kanál relé shield | **1 ks** | V4, V5 |
| HW-15 | Dupont kabely M-F 40žil (20cm) | **2 bal.** | všechny |
| HW-16 | DC-DC step-down 7-40V→1.2-32V 8A 200W | **1 ks** | finální instalace |
| HW-17 | Micro-USB kabel | **1 ks** | všechny |
| HW-18 | Kalafuna | **1 ks** | pájení |

---

## Varianta 1 — Thermal Sentinel (T02)
**Priorita: CRITICAL** | **Závažnost:** Prevence fault 02, ověření Rds(on) hypotézy
**Binární MVP:** "Teplota heatsinku v sériovém terminálu každou sekundu"

### Cíl
Nezávislý teplotní monitoring H-bridge heatsinku střídače POW-HVM3.2H. DS18B20 nalepený přímo na Al blok. D1 mini samostatně napájený.

### Schéma zapojení

```
┌─────────────────┐
│  Wemos D1 Mini  │
│  (ESP8266)      │
│                 │
│  [DS18B20 SHIELD]  ← stackovatelný shield
│  GPIO(D4)=DATA  │────┐
│  VCC=3.3V       │    │    ┌──────────────────┐
│  GND            │    ├────│ DS18B20 čidlo     │
│                 │    │    │ (vodotěsné, 1m)   │
│  USB→5V power   │    │    │ červená=VCC       │
└────────┬────────┘    │    │ žlutá=DATA        │
         │             │    │ černá=GND         │
         │ USB         │    └────────┬─────────┘
         ▼             │             │
   ┌──────────┐        │   tepelná pasta/páska
   │ Notebook  │       │             │
   │ Dell 5590 │       │             ▼
   │ Tera Term │       │    ┌──────────────────┐
   │ 115200 bd │       │    │ H-bridge heatsink │
   └──────────┘       │    │ (velký Al blok)   │
                      │    └──────────────────┘
                      └──── 4.7kΩ pull-up (je na shield)
```

### Postup krok za krokem

**Krok 1 — Fyzické sestavení**
1. Nasaď DS18B20 shield na Wemos D1 Mini (piny pasují)
2. Připoj vodotěsné DS18B20 čidlo do svorkovnice shield:
   - **Červená** → VCC (3.3V)
   - **Žlutá** → DATA (D4/GPIO2)
   - **Černá** → GND
3. Přilep kovové pouzdro čidla **teplovodivou páskou** (nebo kapkou pasty) na H-bridge heatsink (velký Al blok, pravá strana měniče)
4. Propoj D1 mini s notebookem přes micro-USB

**Krok 2 — Nahrání firmwaru (Arduino IDE)**
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

#define ONE_WIRE_BUS D4

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

DeviceAddress sensorAddr;

void setup() {
  Serial.begin(115200);
  sensors.begin();

  if (sensors.getAddress(sensorAddr, 0)) {
    Serial.print("DS18B20 nalezen: ");
    for (uint8_t i = 0; i < 8; i++) {
      if (sensorAddr[i] < 0x10) Serial.print("0");
      Serial.print(sensorAddr[i], HEX);
    }
    Serial.println();
    sensors.setResolution(sensorAddr, 12);
  } else {
    Serial.println("CHYBA: DS18B20 nenalezen!");
  }
}

void loop() {
  sensors.requestTemperatures();
  float tempC = sensors.getTempC(sensorAddr);

  Serial.print("Heatsink: ");
  Serial.print(tempC, 1);
  Serial.print(" °C  ");

  if (tempC > 75.0) {
    Serial.println("!!! SHUTDOWN !!!");
  } else if (tempC > 65.0) {
    Serial.println("!!! CRITICAL !!!");
  } else if (tempC > 55.0) {
    Serial.println("!! WARNING !!");
  } else {
    Serial.println("OK");
  }

  delay(1000);
}
```

**Krok 3 — Ověření**
1. Otevři Tera Term / PuTTY: 115200 baud, 8N1
2. Očekávaný výstup:
   ```
   DS18B20 nalezen: 287253A1000000A5
   Heatsink: 24.5 °C  OK
   Heatsink: 24.5 °C  OK
   ```
3. Pokud svítí "CHYBA: DS18B20 nenalezen":
   - Zkontroluj zapojení (DATA na D4?)
   - Zkontroluj 4.7kΩ pull-up (je na shield, ale multimetrem ověř)
   - Zkus jiný OneWire pin (D3/GPIO0, D2/GPIO4)

**Krok 4 — Zátěžový test**
1. Zapni měnič s ~50W zátěží (např. žárovka)
2. Sleduj teplotu v terminálu
3. Očekávaný trend: ~4-5°C/min nárůst
4. Po cca 10-13 min: fault 02 → DS18B20 ukáže skutečnou teplotu heatsinku:
   - >90°C → Rds(on) degradace potvrzena
   - <60°C → NTC termistor je skutečně vadný

### Troubleshooting

| Problém | Pravděpodobná příčina | Řešení |
|---------|----------------------|--------|
| "CHYBA: DS18B20 nenalezen" | Pull-up chybí / DATA pin | Změř 4.7kΩ mezi DATA a VCC |
| "CHYBA: DS18B20 nenalezen" | Adresa 0x0000... | Zkus pin D3 (GPIO0) místo D4 |
| Teplota +85°C | DS18B20 v parazitním režimu | Připoj VCC (ne jen DATA+GND) |
| Teplota -127°C | Čidlo odpojeno | Zkontroluj konektor |
| Serial nevidíš nic | Špatný baud rate / port | Zkus 9600, 74880 (ESP8266 boot) |

---

## Varianta 2 — GridWatch (T03)
**Priorita: HIGH** | **Závažnost:** Kompletní telemetrie měniče
**Binární MVP:** "Raw Modbus data z POW-HVM3.2H v sériovém terminálu"

### Cíl
Modbus RTU telemetrie ze střídače přes MAX3232 (TTL↔RS232). RJ45 kabel již odstřižen a připraven. Důležité: **2400 baud** (ne 9600, ne 115200).

### Schéma zapojení

```
┌─────────────────┐     RJ45 (odstřižen)     ┌─────────────────┐
│  POW-HVM3.2H    │◄────────────────────────►│  MAX3232        │
│  RS-232 port    │                          │  (female DB9)   │
│                 │  bílo-oranžový=TX (pin1)  │                 │
│                 │  hnědý=GND (pin8)         │  DB9 pin2=R1IN  │
│                 │                          │  DB9 pin5=GND   │
└─────────────────┘                          └────────┬────────┘
                                                       │ TTL
                                                       ▼
                                              ┌─────────────────┐
                                              │  Wemos D1 Mini  │
                                              │  (ESP8266)      │
                                              │  GPIO3(RX)=RXD  │
                                              │  VCC=3.3V       │
                                              │  GND            │
                                              │                 │
                                              │  USB→notebook   │
                                              │  Tera Term      │
                                              │  2400 baud      │
                                              └─────────────────┘
```

### Zapojení pinů (detail)

| MAX3232 pin | Signál | Připojení |
|-------------|--------|-----------|
| DB9 pin 2 | R1IN (RS-232 in) | RJ45 bílo-oranžový (TX z měniče) |
| DB9 pin 5 | GND | RJ45 hnědý (GND měniče) |
| Pin 16 | VCC | WeMos 3.3V |
| Pin 15 | GND | WeMos GND |
| Pin 12 | T1OUT (TTL out) | WeMos GPIO3 (RX pin) — volitelně |
| Pin 14 | T1IN (TTL in) | **volný** (jen posloucháme, neposíláme) |

> **⚠️ Bezpečnost:** MAX3232 VCC na WeMos 3.3V (ne 5V). ESP8266 není 5V tolerantní. RS232 je ±12V, MAX3232 to srazí na 3.3V TTL.

### Postup krok za krokem

**Krok 1 — Fyzické sestavení**
1. RJ45 kabel má jeden konec odstřižený (bílo-oranžový + hnědý vodič)
2. Připoj vodiče k MAX3232 female DB9:
   - **Bílo-oranžový** → DB9 pin 2 (R1IN)
   - **Hnědý** → DB9 pin 5 (GND)
3. Propoj MAX3232 s D1 mini:
   - MAX3232 VCC → D1 mini 3.3V
   - MAX3232 GND → D1 mini GND
   - MAX3232 T1OUT (pin12) → D1 mini GPIO3 (RX pin) — pro pasivní poslech
4. Zapoj nepoužitý konec RJ45 do měniče
5. D1 mini připoj k notebooku micro-USB

**Krok 2 — RAW_FIRST test (pouze serial pass-through)**

Než začneš psát Modbus kód, **jen poslouchej** surová data z měniče:

```cpp
void setup() {
  Serial.begin(115200);   // komunikace s PC
  Serial.begin(2400);     // komunikace s měničem (přes MAX3232)
  // DEBUG: ve skutečnosti ESP8266 má JEDEN HW UART
  // Použijeme SoftwareSerial:
}

#include <SoftwareSerial.h>

SoftwareSerial inverter(D3, D4, false, 256);  // RX=D3(GPIO0), TX=D4(GPIO2)

void setup() {
  Serial.begin(115200);
  inverter.begin(2400);
  Serial.println("RAW_FIRST: nasloucham 2400 baud...");
}

void loop() {
  if (inverter.available()) {
    byte b = inverter.read();
    Serial.print("0x");
    if (b < 0x10) Serial.print("0");
    Serial.print(b, HEX);
    Serial.print(" ");
  }
}
```

**Očekávaný výstup:**
```
0x01 0x03 0x02 0x00 0xE6 0xB9 0xAF ...
```
Modbus rámce: slave ID (0x01), function code (0x03 = read holding registers), data, CRC.

**Pokud nevidíš nic:**
1. Prohoď RX/TX piny (known issue #25 — TX/RX na POW jsou přehozené)
2. Zkus jiný baud rate (4800, 9600 — i když manuál říká 2400)
3. Zkontroluj multimetrem: je na RJ45 pin1 napětí? (±5-12V oproti GND = RS232 signál)

**Krok 3 — Modbus master test**

Až uvidíš raw rámce, přejdi na Modbus knihovnu:

```cpp
#include <ModbusMaster.h>
#include <SoftwareSerial.h>

SoftwareSerial swSerial(D3, D4, false, 256);
ModbusMaster node;

void setup() {
  Serial.begin(115200);
  swSerial.begin(2400);
  node.begin(1, swSerial);  // slave ID = 1
}

void loop() {
  uint8_t result;
  uint16_t regs[8];

  result = node.readHoldingRegisters(4502, 8);
  if (result == node.ku8MBSuccess) {
    Serial.print("AC V: "); Serial.println(regs[0] * 0.1);
    Serial.print("AC Hz: "); Serial.println(regs[1] * 0.1);
    Serial.print("Bat V: "); Serial.println(regs[4] * 0.1);
    Serial.print("SoC: "); Serial.println(regs[5]);
    // ... atd
  } else {
    Serial.print("Modbus fail: 0x");
    Serial.println(result, HEX);
  }
  delay(5000);
}
```

### Troubleshooting

| Problém | Příčina | Řešení |
|---------|---------|--------|
| Nic v terminálu | HW UART kolize | ESP8266 má 1 HW UART → používej SoftwareSerial |
| 0x00 0x00 ... | Špatný baud | Zkus 4800, 9600 |
| 0xFF 0xFF ... | RX/TX přehozeny | Prohoď D3↔D4 |
| Modbus fail 0xE2 | Timeout | RX pin není připojen / špatné GND |
| Modbus fail 0x05 | Bad CRC | Little-endian režim (byte swap) |

---

## Varianta 3 — Climate Station (T06)
**Priorita: LOW** | **Závažnost:** Doplňková — teplota + tlak v technické místnosti
**Binární MVP:** "Teplota a tlak v sériovém terminálu"

### Cíl
BMP180 shield na D1 mini → teplota + barometrický tlak. Užitečné pro monitoring prostředí technické místnosti (vlhkost, kondenzace, teplotní extrémy).

### Schéma zapojení

```
┌─────────────────┐
│  Wemos D1 Mini  │
│  (ESP8266)      │
│                 │
│  [BMP180 SHIELD]   ← stackovatelný shield
│  I2C: D1=SCL   │
│       D2=SDA   │
│  VCC=3.3V      │
│  GND           │
│                 │
│  USB→notebook  │
└─────────────────┘
```

BMP180 shield se nasadí přímo na D1 mini (piny pasují). I2C adresa: **0x77** (default).

### Postup

```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;

void setup() {
  Serial.begin(115200);
  if (!bmp.begin()) {
    Serial.println("CHYBA: BMP180 nenalezen!");
    while (1);
  }
}

void loop() {
  Serial.print("Teplota: "); Serial.print(bmp.readTemperature(), 1); Serial.println(" °C");
  Serial.print("Tlak: "); Serial.print(bmp.readPressure() / 100.0, 1); Serial.println(" hPa");
  Serial.print("Nadm. vyska: "); Serial.print(bmp.readAltitude(1013.25), 1); Serial.println(" m");
  Serial.println("---");
  delay(5000);
}
```

---

## Varianta 4 — Motion Sentinel (perimetr)
**Priorita: MEDIUM** | **Závažnost:** Detekce pohybu + lokální alarm
**Binární MVP:** "Detekce chodce vs. zvěř v sériovém terminálu"

### Cíl
Dvoustupňová detekce pohybu s lokálním alarmem. D1 mini + RCWL-0516 (doppler) + HY-SRF05 (ultrazvuk) + bzučák. Všechny komponenty k dispozici.

### Schéma zapojení

```
┌─────────────────────┐
│  Wemos D1 Mini      │
│  (ESP8266)          │
│                     │
│  D0(GPIO16)=RCWL    │◄── RCWL-0516 (OUT)
│  D1(GPIO5)=TRIG     │──► HY-SRF05 (TRIG)
│  D2(GPIO4)=ECHO     │◄── HY-SRF05 (ECHO)
│  D3(GPIO0)=SW-520D  │◄── Otřesové čidlo
│  D5(GPIO14)=BUZZER  │──► Bzučák shield
│  VCC=5V             │──► senzory
│  GND                │──► společná zem
│                     │
│  USB→5V power       │
└─────────────────────┘
```

### Postup

```cpp
#define RCWL_PIN   D0
#define TRIG_PIN   D1
#define ECHO_PIN   D2
#define TILT_PIN   D3
#define BUZZER_PIN D5

void setup() {
  Serial.begin(115200);
  pinMode(RCWL_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(TILT_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  // 1. Doppler radar
  bool motion = digitalRead(RCWL_PIN);
  // RCWL-0516: HIGH = detekce pohybu, LOW = klid

  // 2. Ultrazvuk (pouze pokud je motion)
  if (motion) {
    long duration, distance;
    digitalWrite(TRIG_PIN, LOW);
    delayMicroseconds(2);
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);
    duration = pulseIn(ECHO_PIN, HIGH);
    distance = duration * 0.034 / 2;

    Serial.print("Pohyb! Vzdalenost: ");
    Serial.print(distance);
    Serial.println(" cm");

    // Člověk = 40-200 cm, zvěř = obvykle >200 cm nebo rychlý průchod
    if (distance < 200) {
      digitalWrite(BUZZER_PIN, HIGH);  // ALARM
      Serial.println("!!! CHODEC !!!");
      delay(3000);                     // bzučí 3s
      digitalWrite(BUZZER_PIN, LOW);
    } else {
      Serial.println("Zver / falesny poplach");
    }
  }

  // 3. Otřesové čidlo
  if (digitalRead(TILT_PIN) == LOW) {
    Serial.println("Otres / vibrace!");
  }

  delay(100);
}
```

### Rozšíření — relé + siréna
Až bude funkční, přidej D1 mini relé shield (HW-14) → 12V siréna napájená z DC-DC měniče (HW-16).

---

## Varianta 5 — All-in-One Hub (D1 mini + stack shields)
**Priorita: FUTURE** | **Závažnost:** Kombinace více funkcí na jednom D1 mini
**Binární MVP:** "Teplota + tlak + detekce v jednom terminálu"

### Cíl
Využít stackovatelné shiely: DS18B20 + BMP180 + bzučák na jednom D1 mini. Úspora ESP modulů pro jiné účely.

### Kompatibilita shieldů

| Shield | I2C adresa | GPIO | Kolize |
|--------|-----------|------|--------|
| DS18B20 shield | — | D4 (GPIO2) | — |
| BMP180 shield | 0x77 | D1(SCL), D2(SDA) | — |
| Bzučák shield | — | D5 (GPIO14) | — |
| Relé shield | — | D6 (GPIO12) | — |

Všechny shiely lze stackovat, pokud nepoužívají stejný pin. DS18B20 (D4) + BMP180 (D1/D2) + bzučák (D5) = bez kolize.

### Postup

Stackuj shiely na D1 mini v pořadí (zdola nahoru):
1. D1 mini základ
2. BMP180 shield
3. Bzučák shield
4. DS18B20 shield

Kombinovaný kód: spočti kódy z Variant 1, 3, 4 do jednoho `.ino`.

---

## Varianta 6 — BMS Sniffer (T05)
**Priorita: HIGH** (blokována) | **Stav: NELZE realizovat bez JST PH kabelu**
**Binární MVP:** "0x4E 0x57 frame v sériovém terminálu"

### Proč je blokovaná
JK BMS má **JST PH 2.0mm 4pin konektor**. Standardní Dupont kabely (2.54mm) fyzicky nezapadnou. Bez JST PH kabelu (GME-01, 50 Kč) nelze BMS připojit.

### Až bude kabel k dispozici

```
┌─────────────────┐     JST PH 2.0mm 4pin    ┌─────────────────┐
│  JK BMS         │◄─────────────────────────►│  ESP-01S        │
│  COM port       │                           │  TX=GPIO1       │
│  1=TX (BMS→ESP) │                           │  RX=GPIO3       │
│  2=RX (ESP→BMS) │                           │  VCC=3.3V(zBMS)│
│  3=GND          │                           │  GND            │
│  4=3.3V VCC     │                           │                 │
└─────────────────┘                           │  USB→serial    │
                                               └─────────────────┘
```

BMS dává 3.3V na pinu 4 → ESP-01S může být napájen přímo z BMS.

---

## Srovnání variant

| Varianta | Název | Priorita | Složitost | Dostupný HW | Binární MVP | Blokuje |
|----------|-------|----------|-----------|-------------|-------------|---------|
| **V1** | Thermal Sentinel | CRITICAL | ★☆☆☆ | ✅ vše | Teplota v terminálu | — |
| **V2** | GridWatch | HIGH | ★★★☆ | ✅ vše (RJ45 hotov) | Modbus v terminálu | V1 (priority) |
| **V3** | Climate Station | LOW | ★☆☆☆ | ✅ vše | Tlak+teplota | — |
| **V4** | Motion Sentinel | MEDIUM | ★★☆☆ | ✅ vše | Pohyb v terminálu | — |
| **V5** | All-in-One Hub | FUTURE | ★★★★ | ✅ vše | Vícesenzor | V1-V4 hotovy |
| **V6** | BMS Sniffer | HIGH | ★★☆☆ | ❌ JST PH kabel | Frame v terminálu | Kabel |

---

## Doporučená roadmap (chronologicky)

### Fáze 1A — Okamžitě (dnes)
```
[V1] Thermal Sentinel ──── 1-2 hod
  └→ Sestavit D1 mini + DS18B20 shield
  └→ Nahrát firmware
  └→ Ověřit teplotu v terminálu ✅
  └→ Nalepit čidlo na H-bridge (tepelná páska/pasta)
```

### Fáze 1B — Dnes/zítra
```
[V2] GridWatch ─────────── 2-3 hod
  └→ Propojit MAX3232 + RJ45 + D1 mini
  └→ RAW_FIRST test (2400 baud)
  └→ Modbus master test
  └→ Ověřit registry ✅
```

### Fáze 1C — Paralelně
```
[V3] Climate Station ───── 30 min
  └→ Nasadit BMP180 shield na druhý D1 mini
  └→ Ověřit tlak+teplotu ✅
```

### Fáze 1D — Pokud zbude čas
```
[V4] Motion Sentinel ───── 2-3 hod
  └→ Sestavit RCWL-0516 + HY-SRF05
  └→ Test: chodec vs. zvěř ✅
```

### Fáze 2 (po potvrzení MVP)
```
[V5] All-in-One Hub ────── 1 hod
  └→ Stackovat shiely na jeden D1 mini
  └→ Kombinovaný firmware

[GCP] Cloud pipeline ───── 4 hod
  └→ ESP32 → Cloud Run → BigQuery
  └→ Telegram notifikace
```

---

## Pravidlo pro deva

1. **RAW_FIRST** — před jakýmkoli kódem vidět surová data v terminálu
2. **Binární MVP** — jakmile vidíš data, task je hotový. Perfektionismus až v Fázi 2
3. **Fail fast** — pokud něco nefunguje do 30 min, deleguj nebo přeskoč
4. **Dokumentuj** — screenshot terminálu = důkaz MVP

---

## Odkazy na související dokumenty

- `docs/analyza_revitalizace_repozitare_v1.0.docx` — Analýza výchozího stavu
- `docs/revitalizacni_plan_v1.0.docx` — Plán revitalizace (5 fází)
- `Outpost_IoT_Security_PoC_v0.1.json` — PoC konfigurace a thresholdy
- `Outpost_IoT_Session_Handoff_Security_v2.json` — Architektonický handoff
- `docs/hardware.md` — Původní HW specifikace
- `docs/Firmware.md` — Původní firmware dokumentace
- `docs/Testovaci_scenare.md` — Testovací scénáře

---

*Klasifikace: Operativní — Outpost 2026 IoT PoC*
