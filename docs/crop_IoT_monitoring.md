# APPENDIX – IoT monitoring & vzdálená správa: polní kultura

**Datum:** 2026-07-27
**Vztah:** Apendix k artefaktům `vysadba_cervenec_zdiby.md` a `vysadba_cervenec_zdiby_APPENDIX_plodiny.md`
**Cíl:** Aspirační projekty s hEROI pro současnou kulturu (2 záhony + 1 květináč, Zdiby)
**Kontext:** Autor disponuje IoT hubem (Outpost Security Perimeter / Outpost IoT Hub) s ESP32/ESP8266 moduly, 24V LiFePO₄ napájením (15.12 kWh), GCP cloud pipeline (Cloud Run → Firestore → BigQuery → Telegram) a WiFi v dosahu ~10 m od záhonů.
**Úroveň:** IoT junior – vše vysvětleno "jako chytrému dítěti"

---

## Obsah – tři projekty krok za krokem

- **Kapitola A:** SHT30 + BH1750 — VPD (klima) a osvětlení
- **Kapitola B:** 3× Capacitive Soil Moisture v1.2 — půdní vlhkost a zálivka
- **Kapitola C:** DS18B20 + BMP180 — teplotní profil (MVP, ihned k dispozici)
- **Kapitola D:** Společný základ – ESP32 firmware a GCP pipeline

---

## Společný základ: Jak ESP32 čte senzory (pro začátečníky)

Než se pustíme do jednotlivých senzorů, pochopme princip. ESP32 je jako malý počítač. Komunikuje se senzory třemi základními způsoby:

1. **GPIO pin (General Purpose Input/Output):** Prostá nožička, která čte 0 V nebo 3.3 V (logická 0 nebo 1). Jako vypínač. Používáme pro OneWire (DS18B20) a tlačítka.

2. **ADC pin (Analog-to-Digital Converter):** Speciální nožička, která čte napětí 0–3.3 V a převádí ho na číslo 0–4095 (12 bitů). Čím vyšší napětí, tím vyšší číslo. Používáme pro půdní vlhkost, pH, EC.

3. **I2C (Inter-Integrated Circuit):** Dvě nožičky – SDA (data) a SCL (hodiny). Na jedny dvě nožičky se dá připojit až stovka různých senzorů, každý má svou adresu (např. 0x44, 0x23). Jako byste měli jedny dveře a každý senzor má své jméno. Používáme pro SHT30, BH1750, BMP180, ADS1115.

ESP32 má WiFi. Každých 15 minut se vzbudí, přečte všechny senzory, pošle data na GCP (Cloud Run → Firestore) a zase usne. Mezi měřeními je v deep sleep režimu (5 µA = prakticky nula).

---

## 8. Kapitola A: SHT30 + BH1750 — klima a světlo pro kulturu

### 8.1 Co to je a k čemu to je?

**SHT30** je luxusní teploměr + vlhkoměr v jednom pouzdře velkém jako nehet. Měří:
- Teplotu vzduchu s přesností ±0.3 °C (dražší verze SHT35 má ±0.1 °C)
- Relativní vlhkost vzduchu s přesností ±2 %
- Komunikuje po I2C (adresa 0x44)

**BH1750** je světloměr. Měří intenzitu osvětlení v luxech (lx). Je to jako foťák měřící, kolik světla dopadá na rostlinu. Komunikuje po I2C (adresa 0x23).

**Proč je to pro kulturu kritické?** Protože z teploty a vlhkosti spočítáme **VPD – Vapor Pressure Deficit**. VPD je "žízeň vzduchu" – rozdíl mezi tím, kolik vody vzduch pojme, a tím, kolik jí skutečně má. Když je VPD moc vysoké (>1.6 kPa), vzduch je suchý, rostlina se dusí (zavírá průduchy, neroste). Když je VPD moc nízké (<0.4 kPa), vzduch je mokrý, číhá plíseň (botrytis). Kultura vyžaduje VPD 0.8–1.4 kPa podle fáze.

**BH1750** říká, jestli rostlina dostává dost světla. Jižní svah ve Zdibech = plné slunce = až 100 000 lx. Při vlnách veder nad 32 °C to může být stres (fotoinhibice). Senzor to potvrdí čísly, ne odhadem.

### 8.2 Jak to zapojit (pinout)

Do ESP32 MiniKIT zapojíme oba senzory na stejné I2C piny (stačí dvě nožičky pro oba senzory):

```
ESP32 MiniKIT     SHT30        BH1750
-----------      -----        ------
GPIO21 (SDA) ─────── SDA (pin 4) ─────── SDA (pin 2)
GPIO22 (SCL) ─────── SCL (pin 3) ─────── SCL (pin 3)
3.3V      ─────── VCC (pin 1) ─────── VCC (pin 1)
GND      ─────── GND (pin 2) ─────── GND (pin 4)
```

**Důležité:** Na I2C lince musí být pull-up rezistory 4.7 kΩ na obou linkách (SDA i SCL). Většina hotových modulů je už má na desce. Pokud kupuješ holý čip, přidej je.

**Adresy (každý senzor má své jméno):**
- SHT30: I2C adresa **0x44** (nebo 0x45, záleží na ADR pinu)
- BH1750: I2C adresa **0x23** (nebo 0x5C, záleží na ADDR pinu)

**Kontrola:** Po zapojení spustit I2C scan – ESP32 najde všechny senzory na lince a vypíše jejich adresy. Pokud nevidí žádný, je špatné zapojení nebo chybí pull-up.

### 8.3 Co říká výrobce (specifikace)

| Parametr | SHT30 | BH1750 |
|----------|-------|--------|
| Teplota | -40 až 125 °C | – |
| Přesnost teploty | ±0.3 °C | – |
| Vlhkost | 0–100 % RH | – |
| Přesnost vlhkosti | ±2 % RH | – |
| Osvětlení | – | 1–65535 lx (H-res mode) |
| Přesnost osvětlení | – | ±1 lx |
| Napájení | 2.4–5.5 V | 2.4–3.6 V |
| Odběr (aktivní) | ~2 mA | ~0.12 mA |
| Rozhraní | I2C (až 1 MHz) | I2C (až 400 kHz) |

### 8.4 Co z toho ESP32 přečte (data v číslech)

```json
{
 "temp_air": 26.4,    // °C
 "humidity_air": 58.2,  // % RH
 "lux": 78200,      // lux
 "vpd": 1.12,       // kPa (vypočítáno softwarem)
 "timestamp": "2026-07-27T14:00:00Z"
}
```

**Vzorec pro VPD (zjednodušeně):**
```
1. Vypočti tlak nasycené páry (Es):
  Es = 0.6108 × exp((17.27 × T) / (T + 237.3))
  kde T = teplota vzduchu ve °C

2. Vypočti aktuální tlak páry (Ea):
  Ea = Es × (RH / 100)

3. VPD = Es - Ea (v kPa)

Prakticky: knihovna pro SHT30 to spočítá sama.
```

### 8.5 Proč to kultura potřebuje (přímý vliv na růst)

| Scénář | VPD | Co se děje | Co dělat |
|--------|:---:|------------|----------|
| Horký suchý den (jižní svah, 35 °C, 30 %) | >1.8 kPa | Rostlina zavírá průduchy, zastavuje růst, stres | Stínicí textilie, zvýšit zálivku |
| Ideál pro ranou fázi růstu (nyní) | 1.0–1.4 kPa | Maximální fotosyntéza, zdravý transport živin | – |
| Vlhký vzduch po dešti (září) | <0.4 kPa | Kondenzace na květenstvích → **botrytis** | Fóliový tunel (rain cover), prořezávka |
| Ráno s rosou | <0.6 kPa | Riziko plísně, nenechat zavřené | Větrání, oddálit zálivku |

**Pro současnou kulturu (27.7., raná fáze, Ø 67 cm):** Cílové VPD je 1.0–1.4 kPa. Pokud SHT30 ukáže VPD > 1.6, je suchý vzduch (červenec na jižním svahu) – zvýšit zálivku, případně stínicí textilie. Pokud VPD < 0.6, hrozí plíseň – zlepšit větrání.

### 8.6 Ukázkový kód (Arduino IDE)

```cpp
#include <Wire.h>
#include <Adafruit_SHT31.h>
#include <BH1750.h>

Adafruit_SHT31 sht30 = Adafruit_SHT31();
BH1750 bh1750;

void setup() {
 Serial.begin(115200);
 Wire.begin(21, 22); // SDA=GPIO21, SCL=GPIO22

 if (!sht30.begin(0x44)) {
  Serial.println("SHT30 nenalezen!");
 }
 if (!bh1750.begin(BH1750::CONTINUOUS_HIGH_RES_MODE, 0x23)) {
  Serial.println("BH1750 nenalezen!");
 }
}

void loop() {
 float temp = sht30.readTemperature();
 float hum = sht30.readHumidity();
 float lux = bh1750.readLightLevel();

 // Výpočet VPD (zjednodušený)
 float es = 0.6108 * exp((17.27 * temp) / (temp + 237.3));
 float ea = es * (hum / 100.0);
 float vpd = es - ea;

 Serial.printf("Teplota: %.1f °C | Vlhkost: %.0f %% | Lux: %.0f | VPD: %.2f kPa\n",
        temp, hum, lux, vpd);

 // Odeslání na GCP (HTTP POST)
 // ...

 delay(60000); // měření každou minutu (pro test)
}
```

### 8.7 Nákupní seznam

| Položka | Cena | Kde koupit | Poznámka |
|---------|:----:|------------|----------|
| SHT30 modul (GY-SHT30) | ~120 Kč | Aliexpress / GM electronic | I2C, 0x44, již s pull-up |
| BH1750 modul (GY-302) | ~40 Kč | Aliexpress / GM electronic | I2C, 0x23, 3.3V |
| 4× dupont kabel M-F | ~10 Kč | jakýkoli hobby market | Propojení s ESP32 |
| **Celkem** | **~170 Kč** | | |

---

## 9. Kapitola B: Capacitive Soil Moisture v1.2 — půdní vlhkost

### 9.1 Co to je a k čemu to je?

Capacitive (kapacitní) půdní vlhkoměr je senzor, který měří, kolik vody je v půdě. Vypadá jako zelená plošná deska s dvěma piny (hroty), které se strčí do země. Na rozdíl od starších rezistivních senzorů (ty vidličkové s odkrytými kovovými pásky) **nekoroduje** – kov není v kontaktu s vodou, měří se kapacita (jako kondenzátor). Vydrží roky, ne týdny.

Funguje na principu: **voda mění dielektrickou konstantu půdy**. Suchá půda má jednu kapacitu, mokrá jinou. ESP32 to změří jako napětí na analogovém pinu (ADC).

**Proč tři kusy:** Jeden do každého záhonu a jeden do květináče AP1. Každé stanoviště může mít jinou vlhkost – záhon 1 (AP6, GG3, GG4, GG5) je na jiném místě než záhon 2 (AP3, AP4, AP5, GG2) a květináč AP1 vysychá úplně jinak (plast, 40 L, tmavý na slunci).

### 9.2 Jak to zapojit

Každý senzor má 3 piny: VCC, GND, AO (analog output). Zapojíme každý na jiný ADC pin ESP32.

```
ESP32 MiniKIT  Sensor 1 (záhon 1)  Sensor 2 (záhon 2)  Sensor 3 (AP1)
-----------   ------------------  ------------------  -----------------
3.3V       VCC          VCC          VCC
GND       GND          GND          GND
GPIO32      AO          –           –
GPIO33      –           AO          –
GPIO34      –           –           AO
```

**Důležité:** 
- Napájet z 3.3V (ne 5V!), protože ESP32 ADC čte max 3.3V.
- Kabely co nejkratší (max 1 m) – delší kabely chytají rušení.
- Senzor zapíchnout kolmo do půdy, alespoň 4 cm hluboko, ne až po pájku (není vodotěsná).
- Pro venkovní použití: pájku a PCB ochránit lakem na nehty nebo silikonem.

### 9.3 Kalibrace (nejdůležitější krok!)

Každý senzor je jiný. Každá půda je jiná. Pokud nezkalibruješ, čísla jsou k ničemu. Kalibrace = zjistit, jakou hodnotu ADC dává senzor v suché půdě a v mokré půdě.

**Postup:**

```
1. Zapoj senzor, nahrej testovací kód (viz 9.6).
2. Nech senzor na suchém vzduchu → zapíš READ_DRY (např. 2850).
3. Zapíchni senzor do nádoby s vodou (ne až po PCB!) → zapíš READ_WET (např. 1250).
4. Tyto dvě hodnoty zadáš do kódu. ESP32 pak přepočítá:
  vlhkost % = map(ADC_hodnota, READ_DRY, READ_WET, 0, 100)
  
  Pozor: suchá půda dává VYŠŠÍ ADC hodnotu než mokrá (voda snižuje napětí).
```

**Kalibrační hodnoty z praxe pro capacitive v1.2 + ESP32 (3.3V napájení):**

| Médium | ADC hodnota (cca) | Poznámka |
|--------|:-----------------:|----------|
| Suchý vzduch | 2800–3000 | Výchozí 0% |
| Suchá zahradní zemina | 2500–2800 | Závisí na typu půdy |
| Optimálně vlhká (60 %) | 1800–2000 | Cíl pro kulturu |
| Mokrá (po dešti) | 1400–1600 | Zálivka hotova |
| Voda (substrát saturován) | 1200–1400 | 100 % (nesmí být trvale) |

**Každý ze tří senzorů musíš zkalibrovat samostatně** – výrobní tolerance jsou velké.

### 9.4 Co říká výrobce (specifikace)

| Parametr | Hodnota |
|----------|---------|
| Napájení | 3.3–5.5 V |
| Analogový výstup | 0–3.0 VDC (při 5V napájení) |
| Odběr | ~5 mA |
| Rozhraní | PH2.0-3P (3pin) |
| Životnost | >3 roky (nekoroduje) |
| Hloubka měření | ~4 cm od špičky |

### 9.5 Jak interpretovat data pro zálivku

| Půdní vlhkost | Význam | Akce |
|:-------------:|--------|------|
| 0–20 % | **Kritické sucho** | Ihned zalít, rostlina vadne |
| 20–40 % | Sucho | Zalít (AP1 květináč vysychá nejrychleji) |
| **40–65 %** | **OPTIMUM** | Nic nedělat |
| 65–85 % | Vlhký | Sledovat, nezalévat |
| 85–100 % | Přemokřeno | Riziko hniloby kořenů, nechat proschnout |

**Pro plodiny v záhonech:** Udržet 40–60 %. V pozdější fázi (srpen–září) mírně sušší (40–55 %) – podporuje zrání a snižuje riziko plísně.

**Pro květináč AP1 (40 L plast, jižní svah):** Bude vysychat 2× rychleji než záhony. Alert při < 30 %.

### 9.6 Ukázkový kód (Arduino IDE)

```cpp
#define SOIL_1 32 // záhon 1
#define SOIL_2 33 // záhon 2
#define SOIL_3 34 // květináč AP1

// Kalibrační hodnoty (změř vlastní!)
#define DRY_1 2850 // záhon 1 – suchý vzduch
#define WET_1 1300 // záhon 1 – ve vodě
#define DRY_2 2800 // záhon 2
#define WET_2 1250
#define DRY_3 2900 // AP1 květináč
#define WET_3 1350

void setup() {
 Serial.begin(115200);
}

int readMoisture(int pin, int dryVal, int wetVal) {
 // Průměr z 10 odečtů (ADC ESP32 je šumivý)
 long sum = 0;
 for (int i = 0; i < 10; i++) {
  sum += analogRead(pin);
  delay(5);
 }
 int raw = sum / 10;
 int pct = map(raw, dryVal, wetVal, 0, 100);
 return constrain(pct, 0, 100);
}

void loop() {
 int m1 = readMoisture(SOIL_1, DRY_1, WET_1);
 int m2 = readMoisture(SOIL_2, DRY_2, WET_2);
 int m3 = readMoisture(SOIL_3, DRY_3, WET_3);

 Serial.printf("Záhon 1: %d %% | Záhon 2: %d %% | AP1: %d %%\n", m1, m2, m3);

 // Alerty
 if (m3 < 30) Serial.println("⚠️ AP1 kritické sucho! Zalít!");
 if (m1 > 85) Serial.println("⚠️ Záhon 1 přemokřeno!");

 delay(60000);
}
```

### 9.7 Nákupní seznam

| Položka | Cena | Kde | Poznámka |
|---------|:----:|-----|----------|
| Capacitive v1.2 (3 ks) | 3× 60 Kč | Aliexpress | Hledej "capacitive soil moisture v1.2" |
| Dupont M-F | ~10 Kč | Hobby market | 3× 3 kabely |
| Lak na nehty / silikon | doma | – | Ochrana PCB před vlhkostí |
| **Celkem** | **~190 Kč** | | |

---

## 10. Kapitola C: DS18B20 + BMP180 — teplotní profil (MVP)

### 10.1 Co to je a k čemu to je?

**DS18B20** je teploměr velký jako tranzistor. Vodotěsný, v kovovém pouzdře s kabelem 1 m. Měří teplotu od -55 do +125 °C s přesností ±0.5 °C. Komunikuje po **OneWire** – speciální protokol, kde všechny senzory visí na jediném pinu a každý má svou unikátní 64-bitovou adresu (jako rodné číslo). Můžeš jich na jeden pin pověsit desítky.

**BMP180** je barometrický senzor. Měří atmosférický tlak (hPa) a teplotu. Komunikuje po I2C. Už ho máš doma (HW-06). Není nejpřesnější na teplotu (±1 °C), ale tlak měří dobře (±1 hPa).

**Proč to potřebuješ hned a v MVP verzi?**
- Půdní teplota ovlivňuje, jak rychle kořeny přijímají živiny (pod 15 °C se příjem P a K zastavuje)
- Teplota vzduchu v koruně + VPD (z SHT30) = kompletní obraz mikroklimatu
- Teplota zálivkové vody – studená voda (pod 10 °C) šokuje kořeny
- BMP180 tlak → predikce počasí (klesající tlak = déšť = nasadit fólii)

### 10.2 MVP rozhodnutí: kolik DS18B20 použít?

Máš **5 ks** DS18B20, ale potřebuješ je i na jiné projekty (T02 – heatsink monitor). Pro garden node navrhuji **MVP = 2 ks**:

| # | Umístění | Proč | Důležitost |
|:-:|----------|------|:----------:|
| **DS18B20 #1** | Půda záhon 2 (mezi AP3 a GG2 – nejsilnější rostliny) | Sledovat, zda půda neklesá pod 15 °C (blokace P) | ★★★★★ |
| **DS18B20 #2** | Vzduch v koruně (zavěsit na oporu u nejvyšší rostliny) | Korelace s SHT30 pro VPD, detekce teplotních extrémů | ★★★★★ |
| DS18B20 #3–5 | Rezerva pro jiné projekty | T02 heatsink, T06 klimatická stanice, záložní | – |

**BMP180** je I2C – stačí připojit na stejný bus jako SHT30 a BH1750 (GPIO21/22). Už ho máš, nic nekupuješ.

### 10.3 Jak to zapojit

**DS18B20 (oba na jeden GPIO pin GPIO4):**

```
ESP32 MiniKIT   DS18B20 #1 (půda)  DS18B20 #2 (vzduch)
-----------    -----------------  ------------------
GPIO4       DATA (žlutý)     DATA (žlutý)
3.3V       VCC (červený)    VCC (červený)
GND        GND (černý)     GND (černý)
```

**Důležité:** OneWire vyžaduje **4.7 kΩ pull-up rezistor** mezi DATA a 3.3V. Bez něj senzory nefungují. Stačí jeden pro všechny DS18B20 na lince.

```
3.3V ──[4.7 kΩ]── GPIO4 (DATA)
```

**BMP180 (I2C – na stejný bus jako SHT30 a BH1750):**

```
ESP32 MiniKIT   BMP180
-----------    ------
GPIO21 (SDA)   SDA
GPIO22 (SCL)   SCL
3.3V       VCC
GND       GND
```

Adresa BMP180: **0x77** (nemění se, na rozdíl od SHT30 0x44 a BH1750 0x23 – všechny tři I2C senzory můžou být na jedné lince, protože mají různé adresy).

### 10.4 Co říká výrobce (specifikace)

| Parametr | DS18B20 | BMP180 |
|----------|---------|--------|
| Teplota | -55 až +125 °C | -40 až +85 °C |
| Přesnost teploty | **±0.5 °C** | ±1 °C |
| Tlak | – | 300–1100 hPa (±1 hPa) |
| Rozlišení | 9–12 bit (volitelné) | 0.01 hPa (ultra-low-power) |
| Napájení | 3.0–5.5 V (parazitní/ externí) | 1.8–3.6 V |
| Odběr | ~1 mA (měření), 0 µA (standby) | 5 µA (standby) |
| Rozhraní | **OneWire** (1 pin pro N senzorů) | I2C (0x77) |

### 10.5 Jak interpretovat data

| Teplota půdy | Význam | Akce |
|:------------:|--------|------|
| < 10 °C | Kořeny nepřijímají P a K | Kritické – na jižním svahu v noci v X |
| 10–15 °C | Zpomalený příjem živin | Sledovat, mulč pomáhá stabilizovat |
| **15–25 °C** | **OPTIMUM – maximální absorpce** | – |
| 25–30 °C | Stále OK, vyšší transpirace | Zvýšit zálivku |
| > 30 °C | Stres, riziko poškození kořenů | Mulč, stínit květináč AP1 |

| Teplota vzduchu v koruně | Význam | Akce |
|:------------------------:|--------|------|
| < 15 °C | Zastavení růstu | Fólie v noci v IX–X |
| 15–20 °C | Pomalý růst | – |
| **20–28 °C** | **OPTIMUM** | – |
| 28–32 °C | Stále OK, vyšší transpirace | Zálivka |
| > 32 °C | Stres, fotoinhibice | Bílá stínicí textilie |
| > 35 °C | **Nebezpečí** – poškození listů | Ihned stínit! |

| Tlak (BMP180) | Význam | Akce |
|:-------------:|--------|------|
| Stoupá (> 3 hPa / 3 h) | Zlepšení počasí | – |
| Stabilní | Status quo | – |
| Klesá (> 3 hPa / 3 h) | Blíží se fronta, déšť | **Připravit fóliový tunel** |
| Rychle klesá (> 5 hPa / h) | Bouřka do 6 h | Zajistit ukotvení, stáhnout fólii |

### 10.6 Ukázkový kód (Arduino IDE)

```cpp
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Wire.h>
#include <Adafruit_BMP085.h>

// OneWire na GPIO4
OneWire oneWire(4);
DallasTemperature sensors(&oneWire);

// BMP180 na I2C (0x77)
Adafruit_BMP085 bmp;

void setup() {
 Serial.begin(115200);
 sensors.begin();   // DS18B20
 bmp.begin(0x77);   // BMP180

 // Počet nalezených DS18B20 na lince
 Serial.printf("Nalezeno %d DS18B20 senzorů\n", sensors.getDeviceCount());
}

void loop() {
 sensors.requestTemperatures();

 // DS18BOT #1 (půda) – první na lince
 float t_soil = sensors.getTempCByIndex(0);
 // DS18B20 #2 (vzduch) – druhý na lince
 float t_air_ds = sensors.getTempCByIndex(1);

 // BMP180
 float t_air_bmp = bmp.readTemperature();
 float pressure = bmp.readPressure() / 100.0; // hPa

 Serial.printf("Půda: %.1f °C | Vzduch DS: %.1f °C | BMP: %.1f °C | Tlak: %.0f hPa\n",
        t_soil, t_air_ds, t_air_bmp, pressure);

 // Alerty
 if (t_soil > 30) Serial.println("⚠️ Půda přehřátá! Mulč funguje?");
 if (t_air_ds > 35) Serial.println("🚨 Teplota v koruně kritická! Stínit!");
 if (pressure < 1005) Serial.println("🌧 Nízký tlak – možný déšť");

 delay(60000);
}
```

### 10.7 Nákupní seznam

| Položka | Cena | Kde | Poznámka |
|---------|:----:|-----|----------|
| DS18B20 vodotěsný 1m | **0 Kč** (máš 5 ks) | – | Použij 2 ks pro garden MVP |
| BMP180 shield pro D1 | **0 Kč** (máš HW-06) | – | I2C, adresa 0x77 |
| 4.7 kΩ rezistor | ~2 Kč | Jakýkoli | Pull-up pro OneWire |
| **Celkem** | **~2 Kč** | | |

---

## 11. Společný firmware pro garden node (všechny senzory dohromady)

### 11.1 Architektura měřicího cyklu

```
┌──────────────────────────────────────────────────────────────┐
│         MĚŘICÍ CYKLUS (každých 15 min)        │
├──────────────────────────────────────────────────────────────┤
│                               │
│ 1. Probuzení z deep sleep                  │
│ 2. Inicializace WiFi, I2C, OneWire             │
│ 3. Čtení SHT30 (teplota + vlhkost)             │
│ 4. Čtení BH1750 (lux)                    │
│ 5. Čtení BMP180 (tlak)                    │
│ 6. Výpočet VPD                       │
│ 7. Čtení 3× capacitive soil moisture (průměr z 10 vzorků) │
│ 8. Čtení 2× DS18B20 (půda + vzduch)            │
│ 9. Sestavení JSON payloadu                 │
│ 10. HTTPS POST na iot-ingest-beta (GCP Cloud Run)      │
│ 11. Vyhodnocení thresholdů → Telegram alert (při anomálii) │
│ 12. Deep sleep (ušetřit baterii)               │
│                               │
│ Celková doba měřicího cyklu: ~3–5 sekund          │
│ Spotřeba: 120 mA × 5 s = 0.17 mAh na cyklus        │
│ Při 96 cyklech/den = 16 mAh/den = 0.08 Wh/den       │
└──────────────────────────────────────────────────────────────┘
```

### 11.2 JSON payload odesílaný na GCP

```json
{
 "device_id": "garden-node-01",
 "timestamp": "2026-07-27T14:00:00Z",
 "measurements": {
  "temp_air": 26.4,
  "humidity_air": 58.2,
  "pressure": 1013.2,
  "lux": 78200,
  "vpd": 1.12,
  "soil_moisture_1": 55,
  "soil_moisture_2": 48,
  "soil_moisture_3": 42,
  "temp_soil": 24.1,
  "temp_air_ds": 26.8
 },
 "battery": {
  "voltage": 5.02,
  "rssi": -48
 }
}
```

### 11.3 Implementační kroky (den 1)

| Krok | Co udělat | Čas |
|------|-----------|:---:|
| 1 | Připojit SHT30 + BH1750 + BMP180 na I2C (GPIO21/22) | 10 min |
| 2 | Naspustit I2C scan – ověřit adresy 0x44, 0x23, 0x77 | 5 min |
| 3 | Připojit 2× DS18B20 na GPIO4 s pull-up 4.7kΩ | 10 min |
| 4 | Naspustit OneWire scan – ověřit 2 adresy | 5 min |
| 5 | Připojit 1× capacitive soil moisture na GPIO32 (test) | 5 min |
| 6 | Kalibrace soil moisture – suchý vzduch + voda | 15 min |
| 7 | Nahrát firmware (všechny senzory + WiFi + GCP POST) | 30 min |
| 8 | Monitorovat Serial output (všechna data čtou?) | 10 min |
| 9 | Ověřit data v Firestore (iot-ingest-beta) | 10 min |
| 10 | Nastavit Telegram alerty (1. alert: AP1 sucho) | 15 min |
| | **Celkem** | **~2 h** |

---

## 12. Shrnutí nákladů a přínosů (všechny tři projekty)

### Pořizovací náklady

| Projekt | Nové komponenty | Cena |
|---------|----------------|:----:|
| A – SHT30 + BH1750 | SHT30 modul, BH1750 modul | 160 Kč |
| B – 3× soil moisture | 3× capacitive v1.2 | 180 Kč |
| C – teplotní profil | 4.7kΩ rezistor | 2 Kč |
| **Celkem** | | **~342 Kč** |

### Co za 342 Kč dostaneš

| Co měříš | K čemu to je | Ušetří / vydělá |
|----------|-------------|:---------------:|
| VPD (SHT30) | Prevence plísně (botrytis) | **Zachráněná sklizeň — prevence ztráty úrody** |
| Osvětlení (BH1750) | Detekce světelného stresu | Kvalitnější květenství |
| Půdní vlhkost (3×) | Přesný timing zálivky, žádné přelévání | Zdravější kořeny, vyšší výnos |
| Teplota půdy (DS18B20) | Detekce blokace P a K pod 15 °C | Prevence deficitu |
| Tlak (BMP180) | Predikce deště → fóliový tunel včas | Prevence botrytis |
| Teplota vzduchu (DS18B20) | Korelace s VPD, detekce extrémů | Včasná reakce na vedra |

### Kdy to začne dávat smysl

První reálný alert, který ti **zachrání rostlinu**, zaplatí celý systém. První úspěšný zásah zaplatí celý systém.

---

## 13. Slovník pojmů (pro úplného začátečníka)

| Pojem | Význam jako pro dítě |
|-------|----------------------|
| **ADC** | Nožička, která čte napětí (0–3.3 V) a převádí na číslo (0–4095). Jako pravítko na elektřinu. |
| **I2C** | Způsob, jak zapojit víc senzorů na dvě dráty. Každý má své jméno (adresu). Jako třída, kde učitel volá jména. |
| **OneWire** | Jeden drát pro víc teploměrů. Každý má unikátní sériové číslo. Jako rodné číslo. |
| **GPIO** | Univerzální nožička na ESP32. Může číst (vstup) nebo posílat signál (výstup). |
| **Pull-up rezistor** | Malá součástka, která drží signál na 3.3 V, když senzor mlčí. Bez ní by linka "plavala" a četla nesmysly. |
| **I2C adresa** | Každý I2C senzor má číslo (např. 0x44). Když ESP32 zavolá "0x44", odpoví jen ten senzor. |
| **Deep sleep** | Režim spánku ESP32, kdy žere skoro nulu (5 µA). Budí se na časovač (např. každých 15 min). |
| **VPD** | "Žízeň vzduchu" – rozdíl mezi tím, kolik vody vzduch pojme a kolik jí má. Suchý vzduch = vysoké VPD = rostlina trpí. Mokrý vzduch = nízké VPD = plíseň. |
| **Lux** | Jednotka osvětlení. Plné slunce = ~100 000 lx. Stín = ~10 000 lx. |
| **hPa** | Jednotka tlaku vzduchu. Normál = 1013 hPa. Když klesá, bude pršet. |
| **Firestore** | Cloudová databáze (GCP), kam ESP32 ukládá měření. Jako nekonečný zápisník v cloudu. |
| **Cloud Run** | Aplikace na GCP, která přijímá data z ESP32 a ukládá je do Firestore. Jako pošťák. |

---

## 14. Kapitola D: Semantická analýza — GO / NO-GO pro sezonu Q3/Q4 2026

### 14.1 Co zkoumáme

Dva dokumenty:
- **A =** `vysadba_cervenec_zdiby_APPENDIX_plodiny.md` — kultivační deepdive, agronomie, rizika, akční seznam
- **B =** tento IoT monitoring dokument

Otázka: **Má autor implementovat IoT řešení pro tuto konkrétní sezonu (Q3/Q4 2026), nebo je to scope creep / overengineering?**

Analyzujeme: kritická rizika kultury, reálnou přidanou hodnotu IoT, opportunity cost a návratnost při zbývajících ~8–10 týdnech do sklizně.

---

### 14.2 Semantická mapa: Co kultura skutečně potřebuje (z dokumentu A)

| Priorita | Potřeba | Riziko při selhání | Lze vyřešit bez IoT? |
|:--------:|---------|:------------------:|:--------------------:|
| **P0** | **Fóliový tunel (skelet nyní, fólie IX)** | Ztráta >50 % úrody vlivem plísně | ✅ Ano — práce, ne technika |
| **P0** | **Přechod na P–K hnojivo (srpen)** | Nízký výnos | ✅ Ano — koupit hnojivo |
| **P0** | **Test pH substrátu** | Blokace Fe/Mn/Zn při pH > 7 | ✅ Ano — pH test strips 50 Kč |
| **P1** | **Denní vizuální kontrola rostlin** | Prošvihnutí choroby nebo škůdců | ✅ Ano — vlastní oči |
| **P1** | **AP1 květináč: stínit + častější zálivka** | Přehřátí kořenů, úhyn | ✅ Ano — textilie + ruka |
| **P1** | **Prořezávka spodních listů** | Plíseň, špatný airflow | ✅ Ano — nůžky |
| **P2** | **Sledovat přehnojení** | Přebytky N = deformace růstu | ✅ Ano — vizuálně |
| **P2** | **Příprava skladovacího prostoru** | Plíseň při sklizni | ✅ Ano — místnost |

**Klíčové zjištění:** Všechna P0 a P1 rizika jsou řešitelná bez jediného senzoru. IoT není kritickou cestou (critical path) k úspěšné sklizni.

---

### 14.3 Semantická mapa: Co IoT nabízí (z dokumentu B)

| Projekt | Cena | Řeší jaké riziko z A? | Alternativa bez IoT |
|---------|:----:|:---------------------:|:-------------------:|
| **DS18B20 + BMP180 (Fáze 0)** | 2 Kč | Teplota půdy < 15 °C → blokace P (nastane v X, sklizeň už probíhá) | Ruka do země |
| **SHT30 + BH1750 (Fáze 1A)** | 160 Kč | VPD > 1.6 → stres (řeší stínicí textilie); VPD < 0.4 → botrytis (řeší fólie v IX) | Koupit digitální teploměr/vlhkoměr 150 Kč, číst ručně |
| **3× Soil moisture (Fáze 1B)** | 180 Kč | Přelití / sucho | Mulč (hotovo) + prst do země |
| **pH + EC (Fáze 2)** | 850 Kč | pH drift, N-load | pH test strips 50 Kč, 1× za 14 dní |
| **Auto-závlaha (Fáze 3)** | 1 000+ Kč | Sucho při nepřítomnosti | Autor je na místě |

---

### 14.4 Scope creep analýza

#### 14.4.1 Časová náročnost vs. zbývající sezona

```
Dnes 27.7.
│
├── Týden 1 (28.7.–3.8.): Implementace IoT Fáze 0 + 1 = 6–10 h
│  ├── Nejvyšší priorita sezony: STAVBA SKELETU TUNELU (P0)
│  └── Druhá nejvyšší: přechod na P–K hnojivo (P0)
│
├── Týden 2 (4.8.–10.8.): Fáze 2 (pH/EC) = 3 dny
│  ├── Vrcholí vegetativní růst → denní monitoring kritický
│  └── Řez + vyvazování
│
├── Týden 3 (11.8.–17.8.): Fáze 3 (auto-závlaha) = 1 týden
│  ├── Rostliny 80–130 cm
│  └── Skelet už by měl stát (jinak se nedá postavit)
│
└── Srpen–září: IoT běží, ale už se nesbírá data pro letošní sezonu
  └── První užitečná data až pro příští rok!
```

**Opportunity cost:** Každá hodina na IoT = hodina, která není na tunel, řez, prořezávku, sklizeň a skladování. V sezoně se 7–10 týdnů do sklizně je čas kritický zdroj.

#### 14.4.2 Poměr přidané hodnoty k úsilí

| IoT komponenta | Instalace | Kalibrace | Integrace | Užitek letos | Užitek příští rok |
|:--------------:|:---------:|:---------:|:---------:|:------------:|:-----------------:|
| **DS18B20 + BMP180** | 30 min | 0 min | 30 min | Střední | Vysoký |
| **SHT30 + BH1750** | 20 min | 0 min | 30 min | Nízký (data bez kontextu) | Vysoký (lze porovnávat) |
| **3× Soil moisture** | 30 min | 60 min | 30 min | Střední (AP1 jen) | Vysoký |
| **pH + EC** | 60 min | 60 min | 60 min | Nízký (test strips rychlejší) | Střední |
| **Auto-závlaha** | 3–5 dní | 1 den | 1 den | Nízký (není potřeba) | Vysoký |

#### 14.4.3 „IoT past" — varování před scope creep

Běžný pattern scope creep u IoT v zahradnictví:

1. **„Dám tam jeden teploměr"** → 30 min
2. **„Když už, tak i vlhkost"** → +160 Kč, +1 h
3. **„A když to posílám na cloud, přidám i půdní vlhkost"** → +180 Kč, +2 h
4. **„Vlastně bych chtěl vědět i pH"** → +850 Kč, +3 dny
5. **„No a když už to mám, mohlo by to samo zalívat"** → +1 000 Kč, +1 týden
6. **„A přidal bych kameru, ať vidím"** → +250 Kč, +1 den

**Výsledek:** Místo 30 min a 0 Kč je z toho 2 týdny práce a 2 440 Kč. Hlavní riziko (botrytis při vlhkém počasí) se neřeší IoT, ale fólií.

---

### 14.5 Verdikt: GO / NO-GO per fáze

#### ✅ Fáze 0 — DS18B20 + BMP180 (MVP, 2 Kč, 30 min)

**GO — Implementovat nyní.**

- Nulové náklady, hardware již k dispozici
- 30 minut práce včetně nahrání firmwaru
- Okamžitý přínos: teplota půdy v záhonu 2 (kontrola blokace P), tlak pro predikci deště
- **Nepřekáží stavbě tunelu** — senzory jsou pod rostlinami
- Cenná data i pro letošní sezonu (zejména predikce deště z BMP180 → včas nasadit fólii)
- **Slouží jako IoT learning pro příští sezony** — nejlepší poměr učení/kč

**Rozsah MVP:**
- 2× DS18B20 (půda záhon 2 + vzduch koruna), ne 5×
- BMP180 (již na I2C)
- Žádné soil moisture, žádný SHT30, žádný cloud
- Pouze Serial output + lokální logování (ESP32 běží na USB power prvních pár dní)

#### ❌ Fáze 1A — SHT30 + BH1750 (160 Kč)

**NO-GO pro letošní sezonu.**

Důvody:
- VPD výpočet je užitečný až při porovnání s daty z minulých let (trendy)
- Na letošní sezonu: suchý vzduch = cítíš to sám, vlhký vzduch = poznáš podle rosy + meteostanice
- Botrytis riziko řeší **fólie** (P0 priorita), ne VPD alerty
- 160 Kč není mnoho, ale **čas na instalaci + integraci + cloud pipeline** = 2–3 h, které chybí na tunel
- **Doporučení:** Koupit nyní, nainstalovat až v září až po dokončení všech P0/P1 agronomických úkolů, nebo rovnou až na jaro 2027

#### ❌ Fáze 1B — 3× Capacitive soil moisture (180 Kč)

**NO-GO pro letošní sezonu.**

Důvody:
- Mulč je hotov (26.7.) — primární mitigace výparu
- Malá výměra není plošný závlahový problém
- Jediný rizikový bod je AP1 (květináč) — a ten řeší **stínění + prst do země**, ne senzor
- Kalibrace 3 senzorů = 1 h času + riziko nepřesnosti (ESP32 ADC je šumivý)
- **Doporučení:** Koupit nyní (cena poroste), ale neinstalovat. Nasadit na jaře 2027, kdy bude celá sezona na IoT.

#### ❌ Fáze 2 — pH + EC + ADS1115 (850 Kč)

**NO-GO pro letošní sezonu. NO-GO pro tuto kulturu vůbec.**

Důvody:
- pH test strips (50 Kč) dají stejnou informaci za 5 minut
- EC monitoring dává smysl při řízení živného roztoku (hydroponie/coco). V záhonech s hnojem + kompostem + organominerálním hnojivem je EC signál příliš komplexní na interpretaci
- 850 Kč = 25× cena pH test strips
- Instalace + kalibrace = 3 dny práce
- **Toto je učebnicový příklad overengineeringu** pro outdoor záhon 3,79 m²

#### ❌ Fáze 3 — Automatická závlaha + ESP32-CAM (1 000+ Kč)

**NO-GO pro letošní sezonu. NO-GO pro tuto kulturu.**

Důvody:
- Rostliny na docházkové vzdálenosti — není potřeba automatizace
- AP1 květináč: stínění + mulč + vědomí, že vysychá rychleji = řešení
- ESP32-CAM: 250 Kč, zábavné, ale 0 přidaná hodnota oproti 1× denně vlastníma očima
- Mechanická instalace čerpadla + ventilů + flow metru = minimálně 1 týden práce
- **Scope creep maximum** — z 30 min teploměru je týdenní projekt

---

### 14.6 GO / NO-GO tabulka

| Projekt | Cena | Čas | GO/NO-GO | Zdůvodnění |
|---------|:----:|:---:|:--------:|------------|
| **DS18B20 + BMP180 (MVP)** | 2 Kč | 30 min | **✅ GO** | Zero cost, okamžitý přínos, learning |
| **SHT30 + BH1750** | 160 Kč | 3 h | **❌ NO-GO** | Koupit teď, nainstalovat až po tunelu / na jaře |
| **3× Soil moisture** | 180 Kč | 2 h | **❌ NO-GO** | Koupit teď, nasadit až 2027 |
| **pH + EC + ADS1115** | 850 Kč | 3 dny | **❌ NO-GO permanent** | Test strips > IoT |
| **Auto-závlaha** | 1 000+ Kč | 1 týden | **❌ NO-GO permanent** | Není potřeba — autor je na místě |
| **ESP32-CAM** | 250 Kč | 1 den | **❌ NO-GO** | Oči > kamera |

---

### 14.7 Doporučený akční plán (co dělat MÍSTO IoT)

| Pořadí | Úkol | Čas | Priorita |
|:------:|------|:---:|:--------:|
| 1 | **Postavit skelet tunelu** (rostliny 52–82 cm, ještě se dá) | 2–4 h | **P0 – NYNÍ** |
| 2 | **Přejít na P–K hnojivo** (koupit, začít aplikovat) | 1 h | **P0 – NYNÍ** |
| 3 | **Otestovat pH substrátu** (test strips 50 Kč) | 10 min | **P0 – NYNÍ** |
| 4 | **Zastínit květináč AP1** (bílá textilie / deska na jižní stranu) | 15 min | **P1 – NYNÍ** |
| 5 | **Prořezávka spodních listů** (zlepšit airflow) | 30 min | **P1 – tento týden** |
| 6 | **Připravit skladovací prostor** (větrání, teplota, vlhkost) | 2 h | **P1 – tento týden** |
| 7 | **DS18B20 + BMP180 MVP** (naučit se IoT, sbírat data pro příští rok) | 30 min | **P2 – až po tunelu** |
| 8 | **Koupit SHT30 + BH1750 + soil moisture** (cena poroste) | 15 min online | **P3 – kdykoli** |
| 9 | **Nainstalovat IoT Fáze 1** | 3 h | **P4 – až po sklizni / jaro 2027** |

---

### 14.8 Shrnutí jedním odstavcem

**Pro sezonu Q3/Q4 2026:** Implementuj pouze DS18B20 + BMP180 MVP (30 min, 2 Kč) jako learning investici. Všechny ostatní IoT projekty odlož na jaro 2027. Tři důvody: (1) všechna P0 rizika kultury řeší agronomie, ne senzory; (2) čas do sklizně (~8–10 týdnů) je příliš krátký na návratnost IoT investice; (3) každá hodina na IoT je hodina, která chybí na stavbu tunelu — a tunel má 100× vyšší EROI než SHT30. **Nenech se chytit do IoT pasti: 30 min a 0 Kč → 2 týdny a 2 440 Kč.** Fólie > senzory. Až bude tunel stát, teprve pak přemýšlej o měření.

---

*Kapitola D – Semantická analýza GO/NO-GO 2026-07-27 | Outpost kontext master – Polní kultura – rychle rostoucí varieta*

### 1.1 K dispozici nyní (bez nákupu)

| Komponenta | Qty | Stav | Využití pro garden |
|------------|:---:|:----:|--------------------|
| **ESP32 MiniKIT** (MH-ET LIVE) | 1 ks | ✅ funkční | Hlavní garden node – 20+ volných GPIO, 3× UART, I2C, ADC |
| **Wemos D1 Mini #1** (ESP8266) | 1 ks | ✅ funkční | DS18B20 termální sentinel → lze rozšířit |
| **Wemos D1 Mini #2** (ESP8266) | 1 ks | ⚠️ plánovaný | Garden sekundární node (klimatická stanice) |
| **ESP-01S** (ESP8266) | 1 ks | ✅ k dispozici | BMS monitoring → alternativa pro dedikovaný senzor |
| **DS18B20 vodotěsné** (1m) | **5 ks** | ✅ k dispozici | Půdní teplota, teplota vzduchu, teplota vody – zdarma |
| **BMP180** (tlak + teplota) | 1 ks | ✅ k dispozici | Atmosférický tlak – již vlastněno |
| **HX711 + load cell** | ? | koncept | Vážení květináče / transpirační monitoring – v konceptu perimetru |
| **DC-DC HW-16 step-down** (7-40V→5V, 8A) | 1 ks | ✅ funkční | Napájení garden node z 24V baterie |
| **24V LiFePO₄** (15.12 kWh) | 1 ks | ✅ funkční | Kapacita zanedbatelná pro garden node (~0,1 % denně) |
| **GCP pipeline** | 1 ks | ✅ funkční | iot-ingest-beta, Firestore, BigQuery, Telegram bot |

### 1.2 Volné GPIO na ESP32 MiniKIT pro garden senzory

| Rozhraní | Piny | Využití |
|----------|:----:|---------|
| **I2C bus** | GPIO21 (SDA), GPIO22 (SCL) | SHT30, BH1750, pH/EC, ADC expander |
| **ADC** | GPIO32, 33, 34, 35, 36, 39 | Půdní vlhkost, pH, EC (6× analog) |
| **OneWire** | GPIO4, 5, 13, 14, 15, 18, 19 | DS18B20 (až 7× na jednom pinu s parazitním napájením) |
| **GPIO/PWM** | GPIO0, 2, 12, 23, 25, 26, 27 | Flow metr, relé, HX711, PWM ventil |
| **UART** | GPIO16/17 (volný) | MH-Z19B CO₂ senzor |

---

## 2. Návrh senzorové sady pro polní kulturu

### 2.1 Fáze 0 – Ihned, zero cost (využití stávajícího HW)

| Senzor | Měří | Rozhraní | ESP pin | Přesnost | Cena |
|--------|------|----------|:-------:|:--------:|:----:|
| **DS18B20 #1** | Teplota půdy – záhon 1 | OneWire | GPIO4 | ±0.5 °C | **0 Kč** (již k dispozici) |
| **DS18B20 #2** | Teplota půdy – záhon 2 | OneWire | GPIO4 (paralelně) | ±0.5 °C | **0 Kč** |
| **DS18B20 #3** | Teplota půdy – květináč AP1 | OneWire | GPIO4 | ±0.5 °C | **0 Kč** |
| **DS18B20 #4** | Teplota vzduchu v koruně | OneWire | GPIO4 | ±0.5 °C | **0 Kč** |
| **DS18B20 #5** | Teplota zálivkové vody | OneWire | GPIO4 | ±0.5 °C | **0 Kč** |
| **BMP180** | Tlak + teplota vzduchu | I2C (0x77) | GPIO21/22 | ±1 hPa | **0 Kč** |
| **HX711 + load cell** | Hmotnost květináče AP1 | HX711 | GPIO14/26 | ±1 g | **0 Kč** (v konceptu) |

**hEROI Fáze 0:** ∞ (nulové náklady, okamžitý přínos)
- 5× teplotní profil půdy a vzduchu
- Základní klimatická data
- Transpirační monitoring vážením (detekce odpařování, přesný timing zálivky)

### 2.2 Fáze 1 – Nízkonákladové senzory (~1 000 Kč)

| Senzor | Měří | Rozhraní | Cena | Přesnost | hEROI |
|--------|------|----------|:----:|:--------:|:----:|
| **Capacitive soil moisture v1.2** (3 ks) | Půdní vlhkost – oba záhony + květináč | Analog ADC | 3× 60 Kč | ±3 % | ★★★★★ |
| **SHT30** | Teplota + vlhkost vzduchu (VPD výpočet) | I2C (0x44) | 120 Kč | ±0.3 °C, ±2 % RH | ★★★★★ |
| **BH1750** | Intenzita osvětlení (lux → PAR) | I2C (0x23) | 40 Kč | ±1 lx | ★★★★☆ |
| **YF-S201** | Průtok / objem zálivky | Pulse GPIO27 | 50 Kč | ±10 % | ★★★☆☆ |

**Celkem Fáze 1:** ~390 Kč
**hEROI:** Vysoký – VPD řízení je kritické pro kulturu (prevence plísně, optimalizace stomatální vodivosti). Půdní vlhkost eliminuje přelévání/podmáčení. BH1750 kvantifikuje světelný stres (jižní svah).

### 2.3 Fáze 2 – Precizní senzory (~1 600 Kč)

| Senzor | Měří | Rozhraní | Cena | Přesnost | hEROI |
|--------|------|----------|:----:|:--------:|:----:|
| **Analog pH meter** (DFRobot Gravity) | pH půdy / zálivky | Analog ADC | 350 Kč | ±0.1 pH | ★★★★☆ |
| **Analog EC meter** (DFRobot Gravity) | EC půdy / živného roztoku | Analog ADC | 350 Kč | ±10 µS/cm | ★★★★☆ |
| **MH-Z19B** (NDIR CO₂) | CO₂ koncentrace | UART GPIO16/17 | 500 Kč | ±50 ppm | ★★★☆☆ |
| **ADS1115** (16-bit ADC, 4ch) | ADC expander pro pH/EC | I2C | 150 Kč | 16-bit | ★★★★☆ |
| **BME280** (náhrada BMP180) | Teplota + vlhkost + tlak (3v1) | I2C | 100 Kč | ±0.5 °C, ±3 % RH | ★★★☆☆ |

**Celkem Fáze 2:** ~1 450 Kč
**hEROI:** Střední–vysoký. pH a EC monitoring je kritický pro absorpci živin (kultura preferuje pH 6.2–6.8). CO₂ je užitečný, ale venkovní kulturu limituje méně než indoor. ADS1115 řeší problém ESP32 ADC nepřesnosti.

### 2.4 Fáze 3 – Automatizace (~1 200 Kč)

| Komponenta | Funkce | Rozhraní | Cena | hEROI |
|------------|--------|----------|:----:|:----:|
| **2× relé modul** (2kanál) | Spínání čerpadla / ventilu | GPIO | 150 Kč | ★★★★★ |
| **12V membránové čerpadlo** | Automatická závlaha dle vlhkosti | 12V DC | 500 Kč | ★★★★☆ |
| **Solenoidový ventil** | Přepínání záhon 1 / záhon 2 | 12V DC | 300 Kč | ★★★☆☆ |
| **ESP32-CAM** | Vizuální monitoring (denní foto) | ESP32 + kamera | 250 Kč | ★★★☆☆ |

**Celkem Fáze 3:** ~1 200 Kč
**hEROI:** Vysoký při častém cestování – automatická závlaha eliminuje riziko sucha. ESP32-CAM umožňuje vizuální kontrolu na dálku.

---

## 3. Architektura garden monitoringu

```
┌─────────────────────────────────────────────────────────────────────┐
│          GARDEN IoT NODE (ESP32 MiniKIT)          │
│                                   │
│ I2C bus (GPIO21/22):                        │
│  ├── SHT30    (teplota + vlhkost → VPD výpočet)       │
│  ├── BH1750    (lux / osvětlení)                │
│  ├── BMP180    (tlak, záložní teplota) – již k dispozici   │
│  └── ADS1115   (16-bit ADC pro pH a EC)            │
│                                   │
│ Analog ADC:                            │
│  ├── GPIO32 → Capacitive soil moisture v1.2 (záhon 1)      │
│  ├── GPIO33 → Capacitive soil moisture v1.2 (záhon 2)      │
│  ├── GPIO34 → Capacitive soil moisture v1.2 (květináč AP1)   │
│  ├── GPIO35 → pH meter (přes ADS1115 nebo přímo)         │
│  └── GPIO36 → EC meter (přes ADS1115 nebo přímo)         │
│                                   │
│ OneWire bus (GPIO4, parazitní napájení):             │
│  ├── DS18B20 #1 → půdní teplota záhon 1             │
│  ├── DS18B20 #2 → půdní teplota záhon 2             │
│  ├── DS18B20 #3 → půdní teplota květináč AP1          │
│  ├── DS18B20 #4 → teplota vzduchu v koruně rostlin        │
│  └── DS18B20 #5 → teplota zálivkové vody             │
│                                   │
│ GPIO (PWM / Interrupt):                      │
│  ├── GPIO27 → YF-S201 (flow meter – průtok zálivky)       │
│  ├── GPIO14 → HX711 DT (hmotnost květináče AP1)         │
│  ├── GPIO26 → HX711 SCK                     │
│  ├── GPIO25 → relé 1 (čerpadlo)                 │
│  └── GPIO23 → relé 2 (ventil záhon 1/2)             │
│                                   │
│ UART:                               │
│  └── GPIO16/17 → MH-Z19B (CO₂)                  │
│                                   │
│ WiFi → HTTPS POST → iot-ingest-beta (GCP Cloud Run)        │
│    → Firestore (telemetrie)                   │
│    → BigQuery (analytika)                    │
│    → Telegram (alerty)                      │
│                                   │
│ Napájení: 5V z DC-DC HW-16 (24V→5V, 8A, již v systému)      │
│ Spotřeba: ~120 mA active, ~10 µA deep sleep            │
│ Režim: měření každých 15 min, deep sleep mezi měřeními      │
│ WiFi: 10 m od obydlí – RSSI ~ -50 dBm (výborný signál)      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.1 Datový tok a vizualizace

```
ESP32 garden node ──HTTPS POST──► iot-ingest-beta (Cloud Run)
                    │
               ┌─────────┴──────────┐
               ▼           ▼
             Firestore       BigQuery
             collection:       dataset:
             garden_telemetry    outpost_analytics
               │           │
               ▼           ▼
          ┌─────────────────┐  ┌──────────────────┐
          │ Telegram Bot   │  │ VPD výpočet   │
          │ (denní report + │  │ (z SHT30 dat)  │
          │ alerty)     │  │         │
          └─────────────────┘  │ Grafana / Looker │
                      │ (budoucí)    │
                      └──────────────────┘
```

**Telegram notifikace (denní report v 8:00):**
```
🌿 Garden Report – 27.7.2026
Záhon 1: 24.2°C / 62% RH / VPD 1.1 kPa / vlhkost půdy 58%
Záhon 2: 23.8°C / 64% RH / VPD 1.0 kPa / vlhkost půdy 52%
AP1 květináč: 25.1°C / vlhkost půdy 45% (↓ pozor!)
Osvětlení: 78,000 lx (jižní svah plné slunce)
CO₂: 415 ppm
Transpirace AP1: -180 ml od včerejška
⚠️ Upozornění: půdní vlhkost AP1 pod 50 %
```

**Alerty při anomáliích:**
- Půdní vlhkost < 30 % → "🚨 Zálivka!"
- Půdní vlhkost > 85 % → "🚨 Přelito!"
- Teplota > 35 °C → "🚨 Horko! Stínicí textilie?"
- VPD > 1.6 kPa → "🚨 Suchý vzduch, stres"
- VPD < 0.4 kPa → "🚨 Vlhký vzduch, riziko plísně"
- pH < 5.5 nebo > 7.5 → "🚨 pH mimo rozsah!"
- EC > 2.5 mS/cm → "🚨 Přehnojeno!"
- Detekce deště (pokles osvětlení + vlhkost) → "🌧 Déšť – zkontrolovat"

### 3.2 VPD (Vapor Pressure Deficit) – klíčový parametr

VPD je pro kulturu kritičtější než samotná teplota nebo vlhkost:

| Fáze | Cílové VPD | T limit | RH limit |
|------|:----------:|:-------:|:--------:|
| **Vegetativní fáze** | 0.8–1.2 kPa | 22–28 °C | 55–70 % |
| **Raná reprodukční fáze (nyní)** | 1.0–1.4 kPa | 24–30 °C | 50–65 % |
| **Plná reprodukční fáze (VIII–IX)** | 1.2–1.6 kPa | 24–28 °C | 40–55 % |
| **Dozrávání (IX–X)** | 1.0–1.4 kPa | 20–26 °C | 45–60 % |

SHT30 + teplota vzduchu z DS18B20 #4 umožňuje VPD výpočet v reálném čase na ESP32.

---

## 4. hEROI analýza – aspirační projekty

### 4.1 Projekt A: Klimatická stanice (VPD + osvětlení)
**Náklady:** SHT30 (120 Kč) + BH1750 (40 Kč) = **160 Kč**
**Přínos:** Prevence plísně (VPD alerty), optimalizace mikroklimatu, kvantifikace světelného stresu
**hEROI:** ★★★★★ (4 měsíce kultivace — prevence ztráty úrody za zlomek její hodnoty)
**Implementace:** 1 den (zapojení + ESPHome/Arduino kód + Firestore ingest)

### 4.2 Projekt B: Půdní vlhkost + transpirační monitoring
**Náklady:** 3× capacitive soil moisture (180 Kč) + HX711 (již k dispozici) = **180 Kč**
**Přínos:** Eliminace přelévání/podmáčení, přesný timing zálivky, křivka transpirace
**hEROI:** ★★★★★ (přelití = plíseň = ztráta celé rostliny)
**Implementace:** 2 dny (kalibrace senzorů pro substrát)

### 4.3 Projekt C: pH + EC monitoring
**Náklady:** pH meter (350 Kč) + EC meter (350 Kč) + ADS1115 (150 Kč) = **850 Kč**
**Přínos:** Kontrola absorpce živin (pH 6.2–6.8), prevence přehnojení (EC)
**hEROI:** ★★★★☆ (při NPK 8-4-8 + hnůj + bobky = riziko kumulace N; EC monitoring by to kvantifikoval)
**Implementace:** 3 dny (kalibrace, montáž, integrace)

### 4.4 Projekt D: Automatická závlaha
**Náklady:** 2× relé (150 Kč) + 12V čerpadlo (500 Kč) + solenoid (300 Kč) + YF-S201 (50 Kč) = **1 000 Kč**
**Přínos:** Nezávislost na přítomnosti, přesné dávkování dle půdní vlhkosti
**hEROI:** ★★★★☆ (při časté nepřítomnosti – eliminuje riziko úhynu suchem)
**Implementace:** 1 týden (mechanická montáž, testování)

### 4.5 Projekt E: Vizuální monitoring (ESP32-CAM)
**Náklady:** ESP32-CAM (250 Kč)
**Přínos:** Denní foto/stream rostlin na dálku, detekce změn v růstu, vizuální kontrola zdraví
**hEROI:** ★★★☆☆ (komfortní, ale neřeší kritické parametry)
**Implementace:** 1 den

### 4.6 Celkový investiční přehled

| Projekt | Náklady | hEROI | Priorita | Časová náročnost |
|---------|:-------:|:-----:|:--------:|:----------------:|
| **A – Klima + VPD** | 160 Kč | ★★★★★ | **P1 – ihned** | 1 den |
| **B – Půdní vlhkost** | 180 Kč | ★★★★★ | **P1 – ihned** | 2 dny |
| **C – pH/EC** | 850 Kč | ★★★★☆ | **P2 – do 1 týdne** | 3 dny |
| **D – Automatická závlaha** | 1 000 Kč | ★★★★☆ | **P3 – do 2 týdnů** | 1 týden |
| **E – Kamera** | 250 Kč | ★★★☆☆ | **P4 – volitelné** | 1 den |
| **Celkem** | **~2 440 Kč** | | | |

---

## 5. Implementační roadmapa

### Týden 1 (28.7. – 3.8.):
1. **Zprovoznit ESP32 MiniKIT** s novým firmwarem (garden node)
  - I2C scan (SHT30 + BH1750 + BMP180)
  - OneWire scan (5× DS18B20)
  - ADC test (3× capacitive soil moisture)
  - HTTPS POST na iot-ingest-beta
2. **Instalace senzorů do záhonů:**
  - DS18B20 #1 do záhonu 1 (hloubka 10 cm)
  - DS18B20 #2 do záhonu 2 (hloubka 10 cm)
  - DS18B20 #3 do květináče AP1
  - DS18B20 #4 do koruny (zavěsit na oporu)
  - SHT30 + BH1750 do výšky koruny (stíněné před přímým sluncem)
3. **Telegram notifikace** – denní report v 8:00, alerty při anomáliích

### Týden 2 (4.8. – 10.8.):
4. **Instalace capacitive soil moisture:**
  - Záhon 1 (mezi AP6 a GG3/GG4)
  - Záhon 2 (mezi AP3 a GG2)
  - Květináč AP1
  - Kalibrace: suchý substrát → voda (každý senzor individuálně)
5. **HX711 + load cell pro AP1:**
  - Vážicí plošina pod květináč
  - Denní křivka transpirace
  - Alert při poklesu hmotnosti pod threshold

### Týden 3 (11.8. – 17.8.):
6. **pH + EC metry:**
  - Instalace ADS1115 (16-bit ADC) na I2C
  - Kalibrace pH (pH 4.0, 7.0, 10.0)
  - Kalibrace EC (1.41 mS/cm standard)
  - Měření pH půdy v obou záhonech + květináči
  - Měření EC zálivkové vody

### Srpen – září (fáze květu):
7. **Automatická závlaha** (volitelné):
  - 12V membránové čerpadlo + YF-S201 flow meter
  - Automatické spuštění při půdní vlhkosti < 40 %
  - Vypnutí při > 70 % nebo po 5 L
8. **ESP32-CAM:**
  - Ranní foto k 8:00 → Cloud Storage
  - Možnost ručního snapshotu přes Telegram

---

## 6. Integrace se stávajícím systémem

### 6.1 GCP – nové kolekce Firestore

```
garden_telemetry/     # Senzorová data
 {device_id}/
  {timestamp}/
   temp_soil_1: float
   temp_soil_2: float
   temp_soil_pot: float
   temp_air: float
   humidity_air: float
   pressure: float
   vpd: float
   lux: int
   soil_moisture_1: int (%)
   soil_moisture_2: int (%)
   soil_moisture_pot: int (%)
   ph_soil: float
   ec_soil: float
   co2: int
   flow_volume: float (L)
   weight_pot: float (g)
   battery_voltage: float
   rssi: int

garden_events/       # Alerty a události
 {timestamp}/
  type: string (alert/info/warning)
  message: string
  value: float
  threshold: float

garden_config/       # Konfigurace
 thresholds/
  soil_moisture_min: 30
  soil_moisture_max: 85
  temp_max: 35
  vpd_min: 0.4
  vpd_max: 1.6
  ph_min: 5.5
  ph_max: 7.5
  ec_max: 2.5
```

### 6.2 BigQuery – analytické dotazy

```sql
-- VPD křivka za posledních 7 dní
SELECT
 TIMESTAMP_TRUNC(timestamp, HOUR) AS hour,
 AVG(temp_air) AS avg_temp,
 AVG(humidity_air) AS avg_hum,
 AVG(vpd) AS avg_vpd
FROM outpost_analytics.garden_telemetry
WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY hour
ORDER BY hour;

-- Korelace půdní vlhkosti a transpirace (AP1)
SELECT
 soil_moisture_pot,
 AVG(weight_pot) AS avg_weight,
 COUNT(*) AS readings
FROM outpost_analytics.garden_telemetry
WHERE timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
GROUP BY soil_moisture_pot
ORDER BY soil_moisture_pot;
```

### 6.3 Telegram – stávající bot (rozšíření)

Stávající Telegram bot (3 kanály: info, alerts, direct) se rozšíří o:
- **Kanál #4:** `garden_info` – denní report v 8:00
- **Kanál #5:** `garden_alerts` – anomálie (sucho, pH, VPD)
- **Direct commands:**
 - `/garden` – aktuální status
 - `/photo` – snapshot z ESP32-CAM
 - `/water` – manuální spuštění závlahy
 - `/calibrate` – spuštění kalibrační sekvence

### 6.4 Napájení – stávající 24V LiFePO₄

| Komponenta | Spotřeba | Denně | % z 15.12 kWh |
|------------|:--------:|:-----:|:-------------:|
| ESP32 MiniKIT (active) | 120 mA @ 5V = 0.6 W | 15 min/h = 6 h = 3.6 Wh | 0.024 % |
| ESP32 deep sleep (18 h) | 10 µA = 0.05 mW | 0.9 mWh | ~0 % |
| SHT30 + BH1750 | 2 mA = 10 mW | 6 h = 60 mWh | ~0 % |
| 3× soil moisture | 15 mA = 75 mW | 6 h = 450 mWh | 0.003 % |
| pH/EC (přes ADS1115) | 5 mA = 25 mW | 6 h = 150 mWh | 0.001 % |
| **Celkem garden node** | | **~4.3 Wh/den** | **0.028 %** |
| Stávající IoT hub (T01–T07) | | ~23 Wh/den | 0.15 % |
| **Rezerva baterie** | | **15 120 Wh – 27.3 Wh = 99.8 %** | |

**Závěr:** Garden node je z hlediska napájení zcela zanedbatelný (0.028 % kapacity). Není potřeba žádná úprava napájecí soustavy.

---

## 7. Rizika a mitigace

| Riziko | Pravděpodobnost | Dopad | Mitigace |
|--------|:--------------:|:-----:|----------|
| **ESP32 ADC nepřesnost** (pH/EC) | Vysoká | Střední | ADS1115 (16-bit ADC, I2C) – 100 Kč |
| **Koroze senzorů půdní vlhkosti** (rezistivní typ) | Vysoká | Vysoký | Použít výhradně **capacitive v1.2** (nekoroduje) |
| **Přehřátí ESP32 na přímém slunci** | Střední | Střední | IP65 krabička pod záhon / ve stínu rostlin |
| **WiFi výpadek** | Nízká (10 m) | Nízký | ESP32 buffering – data se odešlou při obnovení spojení |
| **Poškození kabeláže zvěří** | Střední | Střední | Kabeláž v chráničce, senzory pod mulčem |
| **Selhání čerpadla (automatická závlaha)** | Nízká | Střední | Bypass – možnost ruční zálivky |
| **Degradace pH/EC sond** | Střední | Vysoký | Pravidelná kalibrace (1× měsíčně), skladování v roztoku |

---

## 8. Závěr – aspirační projekty s hEROI

### Okamžitě (týden 1) – 340 Kč
1. **5× DS18B20 + BMP180** – teplotní profil (0 Kč, již k dispozici)
2. **SHT30 + BH1750** – VPD + osvětlení (160 Kč)
3. **3× capacitive soil moisture v1.2** – půdní vlhkost (180 Kč)

### Krátkodobě (týden 2-3) – 1 000 Kč
4. **HX711** – transpirační monitoring (0 Kč, již v konceptu)
5. **pH + EC + ADS1115** – živná kontrola (850 Kč)
6. **YF-S201** – flow monitoring (50 Kč)

### Střednědobě (srpen) – 1 250 Kč
7. **Automatická závlaha** (čerpadlo + relé + solenoid) – 1 000 Kč
8. **MH-Z19B CO₂** – 500 Kč (volitelné)
9. **ESP32-CAM** – 250 Kč (volitelné)

**Celková investice:** ~2 440 Kč
**Denní spotřeba:** +4.3 Wh (0.028 % baterie)
**Návratnost:** I při 10% navýšení výnosu díky IoT je ROI pozitivní.**

**Nejvyšší hEROI položka:** SHT30 + BH1750 (160 Kč) – VPD řízení je kritický faktor pro kulturu a jeho monitoring může zabránit ztrátě celé sklizně plísní při vlhkém vzduchu (botrytis).

---

*Appendix – IoT monitoring rešerše 2026-07-27 | Outpost kontext master – Polní kultura – rychle rostoucí varieta | Navazuje na Outpost Security Perimeter / Outpost IoT Hub*
