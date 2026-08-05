# Návrh hardwaru (Hardware Design)

Tento adresář bude obsahovat veškerou dokumentaci k hardwaru, schémata a diagramy zapojení pinů (pinout) poté, co bude prototyp úspěšně ověřen v terénu.

> [!IMPORTANT]
> **Stav:** Aktuálně prázdné. Dokumentace bude doplňována průběžně po validaci PoC (Proof of Concept).

### Plánovaný obsah:

* `schematics/` – Soubory Fritzing / KiCad pro váhovou podložku a propojení s ESP32.
* `pinout.md` – Detailní přiřazení GPIO pinů pro ESP32 (PIR, HX711, kamera, MOSFETy).
* `calibration.md` – Postup kalibrace tenzometrických snímačů krok za krokem.
* `enclosure.md` – Rozměry boxu s krytím IP67 a pokyny pro montáž.

---

## Předběžné technické informace

Aktuální verze PoC (Proof of Concept) využívá následující komponenty:

| Komponenta | Specifikace a role v systému |
| :--- | :--- |
| **ESP32 (Libre)** | Podpora `deep-sleep`, buzení externím přerušením (interrupt) z PIR senzoru. |
| **PIR senzor AM312** | Miniaturní detektor pohybu s velmi nízkým proudovým odběrem. |
| **HX711 + 4× 50 kg** | Tenzometrické snímače (load cells) zapojené pro měření celkové hmotnosti. |
| **ESP32-CAM (OV2640)** | Kamerový modul pro vizuální verifikaci, napájený selektivně přes MOSFET. |
| **MOSFET moduly (IRF520)** | Slouží ke spínání 5V větve pro HX711 a kameru (úspora energie). |
| **DC-DC měnič 24V→5V** | Hlavní napájecí modul pro celý systém. |

### Umístění a ochrana
Veškerá řídicí elektronika je umístěna v **suchém IP67 boxu** uvnitř objektu. Vnějšímu prostředí (vlivům počasí) je vystavena pouze samotná váhová podložka a PIR senzor.

---

*Tato sekce bude rozšířena po dokončení testování v terénu (předpokládaný termín: Q2 2026).*

---

# APPENDIX A: Salvage Hardware — Indukční vařič Rohnson R-2450

**Zdroj:** Vařič indukční Rohnson R-2450 (rozbitá varná deska)  
**Datum analýzy:** 2026-08-05  
**Účel:** Identifikace komponentů pro sekundární využití v Outpost IoT Hub  
**Stav:** Po demontáži, komponenty připraveny k testování

---

## A.1 Přehled salvage komponentů

| # | Komponent | Zdrojový stav | Kompatibilita s Outpost | Priorita |
|---|-----------|---------------|------------------------|----------|
| 1 | Toroidní tlumivka | Funkční | BMS UART EMI ochrana (T05) | 🔴 A |
| 2 | MKP-X2 8µF / 275V | Funkční | EMI filtr měniče (T03) | 🔴 A |
| 3 | MKP-X2 5µF / 275V | Funkční | EMI filtr vstup (T03) | 🔴 A |
| 4 | Buzzer piezo | Funkční | Lokální alarm (T01) | 🔴 A |
| 5 | Ventilátor JY-020 12V | Funkční | Aktivní chlazení | 🟡 B |
| 6 | 7-segment displej | Funkční | Lokální status display | 🟡 B |
| 7 | Heatsink Al | Funkční | Chlazení DC-DC HW-16 | 🟡 B |
| 8 | Indukční cívka | Funkční | Perimetrální detekce kovů | 🟠 C |
| 9 | EE10 transformátor | Funkční | Galvanické oddělení RS485 | 🟠 C |
| 10 | Elektrolyty (5-10 ks) | Testovat | Prototypování | 🟢 D |

---

## A.2 Detekce perimetru — Indukční senzor kovů (T01)

### Koncept
Indukční cívka z R-2450 jako **LC oscilátor** pro detekci kovových předmětů na vstupu.

### Schéma zapojení
```
                    ┌─────────────────────────────────┐
                    │      LC OSCILÁTOR (555/ESP32)    │
                    │                                 │
  INDUKČNÍ CÍVKA ──┤  L1          R1                  │
  (z R-2450)       │  ~~~~~~~    ┌─┴─┐               │
                    │            │   │ 10kΩ           │
  MKP-X2 8µF ──────┤  C1        └─┬─┘               │
  (z R-2450)       │  |||         │                  │
                    │  |||        GND                 │
                    └──────────────┬─────────────────┘
                                   │
                              GPIO 4 (ESP32)
                              frekvence → ADC/count
```

### Princip fungování
1. LC obvod kmitá na rezonanční frekvenci f₀ = 1/(2π√LC)
2. Přiblížení kovu → změna indukčnosti L → změna f₀
3. ESP32 měří periodu přes timer counter
4. Delta f > práh = detekce kovu

### Parametry (odhad)
| Parametr | Hodnota |
|----------|---------|
| C1 (MKP-X2) | 8 µF |
| L1 (cívka) | ~1-5 mH (závisí na počtu závitů) |
| f₀ | ~800 Hz - 2 kHz |
| Dosah | 5-15 cm (záleží na velikosti kovu) |
| Spotřeba | ~50-100 mA (nutný MOSFET spínač) |

### Integrace do stávající architektury
- Přidat jako **vrstvu L5** do Security Perimeter cascade
- Budit z deep-sleep přes PIR (stávající L0)
- Po probuzení: aktivovat LC senzor přes MOSFET (GPIO 14)
- Měřit frekvenci 100ms → vyhodnotit → případně aktivovat kameru

### Výhody oproti stávajícím senzorům
| Vlastnost | PIR (AM312) | RCWL-0516 | Indukční (nový) |
|-----------|-------------|-----------|-----------------|
| Voděodolnost | ❌ | ⚠️ omezená | ✅ |
| Noční provoz | ✅ | ✅ | ✅ |
| Detekce kovů | ❌ | ❌ | ✅ |
| Falešné poplachy | vysoké | střední | nízké |
| Spotřeba | ~0.02 mA | ~3 mA | ~100 mA* |

*nutný MOSFET spínač pro úsporu energie

---

## A.3 EMI Ochrana — BMS UART (T05)

### Problém
JK BMS komunikuje přes UART na 115200 baud. 3.2kW solární měnič generuje EMI, který způsobuje chyby v přenosu.

### Řešení — Common-Mode Choke
Toroidní tlumivka z R-2450 jako **common-mode choke** na UART linku.

### Schéma
```
ESP32 TX (GPIO 17) ────┐
                       │
                  ┌────┴────┐
                  │ TOROID  │  ← 30-40 závitů na feritovém jádru
                  │ TLUMIVKA│     (z R-2450)
                  └────┬────┘
                       │
JK BMS RX ─────────────┘

ESP32 RX (GPIO 16) ────┐
                       │
                  ┌────┴────┐
                  │ TOROID  │
                  │ TLUMIVKA│
                  └────┬────┘
                       │
JK BMS TX ─────────────┘
```

### Implementace
1. Navinout 2-3 závity TX linky přes toroid
2. Navinout 2-3 závity RX linky přes toroid
3. Žádný firmware update — čistě hardwarové řešení
4. Otestovat stabilitu přenosu po přidání tlumivky

### ROI
- Toroidní tlumivka: ~80 Kč (nová cena)
- RS485 přechod: ~300 Kč + firmware update
- **Úspora: ~220 Kč + 0 hod firmware práce**

---

## A.4 EMI Filtr — Vstup 24V (T03)

### Problém
DC-DC měnič HW-16 (24V→5V) nemá EMI filtr na vstupu. Solární měnič generuje špičky.

### Řešení — Pi-Filter
MKP-X2 kondenzátory + toroidní tlumivka jako **pi-filter**.

### Schéma
```
24V BAT+ ──[MKP-X2 8µF]──┬──[TOROID]──┬──→ DC-DC HW-16 (vin)
                          │             │
                         GND           GND

24V BAT+ ──[MKP-X2 5µF]──┤             │
                          │             │
                         GND           GND
```

### Parametry filtru
| Komponent | Hodnota | Účel |
|-----------|---------|------|
| C1 (MKP-X2) | 8 µF / 275V AC | Primární filtr |
| C2 (MKP-X2) | 5 µF / 275V AC | Sekundární filtr |
| L1 (toroid) | ~1-5 mH | Common-mode tlumení |
| cutoff freq | ~100 Hz - 1 kHz | Potlačení spínaného šumu |

### Bezpečnost
- MKP-X2 jsou **bezpečnostní kondenzátory** (X2 norma)
- Při poruše se nerozpalují, nehoří
- Před montáží: **vypustit přes rezistor 10kΩ/5W!**

---

## A.5 Lokální Alarm (T01)

### Komponent
Piezo bzučák z R-2450 (černý válec, ~5V, 2-4 kHz)

### Připojení na D1 Mini
```
D1 Mini GPIO D5 (GPIO14) ──[100Ω]──→ Buzzer (+)
                                      Buzzer (-) ──→ GND
```

### Integrace do firmware
```cpp
// Po detekci člověka (>40 kg)
if (weight > HUMAN_THRESHOLD) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(500);
    digitalWrite(BUZZER_PIN, LOW);
    // Současně: kamera + Telegram notifikace
}
```

### Funkce
- Zvuková signalizace při detekci >40 kg
- Lokální varování pro případného narušitele
- Auditní signál pro kalibraci (test buzzeru)

---

## A.6 Aktivní Chlazení

### Komponent
Ventilátor JY-020 (JINGYI DC BRUSHLESS FAN), 12V, ~80mm

### Připojení
```
12V SUPPLY ──┬──→ Ventilátor (+)
             │
            [MOSFET IRF520]
             │
ESP32 GPIO 14 (volný) ──[10kΩ]──→ MOSFET Gate
                                    MOSFET Source ──→ GND
                                    MOSFET Drain ──→ Ventilátor (-)
```

### Řízení (PWM)
```cpp
// Teplotní regulace
if (temp_heatsink > 50.0) {
    analogWrite(FAN_PIN, 200);  // 80% výkon
} else if (temp_heatsink < 40.0) {
    analogWrite(FAN_PIN, 0);    // vypnuto
}
```

### Umístění
- Na heatsink DC-DC HW-16
- Nebo v IP67 boxu pro proudění vzduchu

---

## A.7 Lokální Status Display

### Komponent
4-místný 7-segment displej z R-2450 (červený LED)

### Zobrazení
| Formát | Význam |
|--------|--------|
| `88:88` | SoC baterie (%) |
| `t:XX` | Teplota heatsinku (°C) |
| `ALRM` | Alarm aktivní |
| `----` | Systém v deep-sleep |

### Připojení
- Přes **shift register 74HC595** (3 GPIO: DATA, CLK, LATCH)
- Nebo přímo na GPIO (pokud je decode obvod součástí displeje)

### Priorita
- NÍZKÁ — systém je headless (Telegram + GCP)
- Implementovat až po stabilizaci T01-T05

---

## A.8 Heatsink pro DC-DC

### Komponent
Aluminiový heatsink z R-2450 (~80×50×25mm)

### Využití
- Pasivní chlazení DC-DC měniče HW-16
- Alternativa: chlazení MOSFET modulů (IRF520)

### Montáž
- Tepelná pasta + tepelná páska
- Přichytit šrouby nebo tepelným lepidlem

---

## A.9 Indukční cívka — Experimentální projekty

### Varianta 1: Bezdrátové nabíjení (WPT)
- Primární cívka z R-2450
- Sekundární cívka: navinout na stejném jádru
- Frekvence: ~100-200 kHz (Qi standard)
- Výkon: ~5W (pro malé senzory)

### Varianta 2: Indukční topení
- Cívka jako primár spínaného zdroje
- Kovový předmět jako sekundární zátěž
- Ohřev pájek, nástrojů, mincí

### Varianta 3: Detekce kovů (viz A.2)

---

## A.10 Transformátor EE10 — Galvanické oddělení

### Komponent
Malý spíraný transformátor EE10 z R-2450

### Využití
- Galvanické oddělení RS485 linky (Phase 3)
- Puls transformátor pro měřicí aplikace
- Oddělení primární/sekundární strany

### Poznámka
- Vyžaduje převíjení (primární/sekundární poměr)
- Experimentální — nízká priorita

---

## A.11 Ekonomická analýza

### Hodnota komponentů (odhad tržní cena)

| Komponent | Odhad cena | Kde koupit nový |
|-----------|-----------|-----------------|
| Toroidní tlumivka | 50-100 Kč | AliExpress |
| MKP-X2 8µF | 80-150 Kč | TME, GM Electronic |
| MKP-X2 5µF | 60-120 Kč | TME, GM Electronic |
| Ventilátor 12V 80mm | 100-200 Kč | Alza, CZC |
| 7-segment displej | 50-100 Kč | Arduino shopy |
| Heatsink | 50-100 Kč | GM Electronic |
| Buzzer | 20-40 Kč | Arduino shopy |
| Indukční cívka | 200-400 Kč | AliExpress |
| EE10 transformátor | 40-80 Kč | AliExpress |
| Elektrolyty (5-10 ks) | 30-80 Kč | GM Electronic |
| **CELKEM** | **~680-1370 Kč** | |

### Úspora vs. nákup nových komponentů

| Konvergence | Úspora | Čas |
|-------------|--------|-----|
| Toroid → BMS EMI | ~80 Kč | 0 hod |
| MKP-X2 → EMI filtr | ~180 Kč | 2 hod |
| Buzzer → alarm | ~30 Kč | 0.5 hod |
| Ventilátor → chlazení | ~150 Kč | 1 hod |
| Displej → status | ~80 Kč | 3 hod |
| **CELKEM** | **~520 Kč** | **~6.5 hod** |

---

## A.12 Bezpečnostní varování

### ⚠️ KRITICKÉ PŘED DEMONTÁŽÍ

```
1. VYPOUT Z SÍTĚ — ověřit multimetrem (AC napětí na vstupu = 0V)
2. VYPUSTIT KONDENZÁTORY — MKP-X2 přes rezistor 10kΩ/5W
   - Počkat 5 minut po vypnutí
   - Ověřit napětí multimetrem = 0V
3. ELEKTROLYTY — mohou být nabité (až 50V)
   - Vypustit přes rezistor
4. SKLO — ostré střepy, rukavice povinné
5. CÍVKA — NEPOJAT do sítě bez řídící elektroniky!
6. IGBT/MOSFET — může být vadný (důvod rozbití?)
```

### Postup demontáže

| Krok | Komponent | Nářadí | Čas |
|------|-----------|--------|-----|
| 1 | Ventilátor | Křížový šroubovák | 2 min |
| 2 | Indukční cívka | Křížový šroubovák + páječka | 3 min |
| 3 | Ovládací panel | Křížový šroubovák | 2 min |
| 4 | PCB | Křížový šroubovák | 5 min |
| 5 | Heatsink | Klíč + tepelná pasta | 5 min |
| 6 | Kondenzátory | Horkovzdušná páječka | 10 min |
| **CELKEM** | | | **~27 min** |

---

## A.13 Doporučený implementační plán

### Fáze 1 — Okamžitá integrace (1 den)
- [ ] Demontovat komponenty dle postupu A.12
- [ ] Toroidní tlumivka → otestovat na BMS UART
- [ ] Buzzer → připojit na D1 Mini GPIO
- [ ] Ventilátor → připojit na 12V, otestovat

### Fáze 2 — EMI filtr (2-3 hodiny)
- [ ] Namontovat MKP-X2 8µF + toroid jako pi-filter
- [ ] Zapojit mezi 24V baterii a DC-DC HW-16
- [ ] Otestovat stabilitu napájení

### Fáze 3 — Indukční senzor (1-2 dny)
- [ ] Navinout cívku na LC oscilátor (555 timer nebo ESP32 PWM)
- [ ] Kalibrovat baseline (bez kovu)
- [ ] Testovat detekci kovových předmětů

### Fáze 4 — Displej (2-3 hodiny)
- [ ] Ověřit pinout 7-segment displeje
- [ ] Připojit přes shift register na ESP32
- [ ] Zobrazit SoC baterie a stav alarmu

---

## A.14 Reference

| Zdroj | Soubor | Umístění |
|-------|--------|----------|
| Rohnson R-2450 | pcb.jpg, detail01.jpg, detail02.jpg | `C:\Users\PC\Desktop\Vařič indukční Rohnson R-2450\` |
| Salvage report | RECYKLACE_REPORT.md | `C:\Users\PC\Desktop\Vařič indukční Rohnson R-2450\` |
| BMS EMI problém | BMS_Telemetry_Pipeline.md:36 | `firmware/bms_monitor/` |
| Security cascade | Security_Perimeter_Architecture.md:9-24 | `firmware/security_perimeter/` |
| Power budget | ESP32_HW_Architecture.md:76-96 | `firmware/` |
| Pin mapy | koncepce_zabezpeceni.md:62-68 | root |

---

*Tento appendix byl vygenerován 2026-08-05 na základě sémantické analýzy repozitáře a salvage komponentů z Rohnson R-2450.*
