# Outpost IoT Hub — bezpečnostní perimeter, monitoring, automatizace

## Deterministické zabezpečení perimetru – váhový filtr jako jádro detekce

Projekt řeší zásadní problém zabezpečení odlehlé chaty v prostředí, kde běžná PIR čidla a Doppler radary generují nepřijatelné množství falešných poplachů. Namísto slepého spoléhání na jeden senzorický princip staví systém na **dvoustupňové validaci**, jejíž jádro tvoří **váhový detektor**.

**Základní odlišnost systému:**
* **Fyzikální validace:** Váha (tenzometrická nebo pneumatická) měří hmotnost objektu přímo v místě vstupu.
* **Deterministický práh (>40 kg):** Jasné kritérium pro rozlišení člověka od zvěře (srnec, kočka, pes).
* **Energetická efektivita:** Bez potvrzení hmotnosti nedochází k aktivaci kamery ani energeticky náročnému přenosu dat.

---

## Rozšířený scope — Outpost IoT Hub

Repozitář byl revitalizován z původního "security perimeter" na **Outpost IoT Hub** — monitoring, automatizace a bezpečnost off-grid uzlu. Zahrnuje teplotní monitoring (DS18B20), Modbus telemetrii, BMS monitoring, IoT senzorové sady pro polní kulturu a cloudovou pipeline (GCP).

---

## Obsah docs/

| Soubor | Popis |
| :--- | :--- |
| **Bezpečnost perimetru** | |
| [`koncepce_zabezpeceni.md`](docs/koncepce_zabezpeceni.md) | Detailní rozbor strategie ochrany perimetru a metodiky detekce. |
| [`differential_analysis.md`](docs/differential_analysis.md) | Srovnání s komerčními systémy a typickými IoT projekty. |
| [`Testovaci_scenare.md`](docs/Testovaci_scenare.md) | Metodika testování (simulace zvěře vs. člověka, teplotní drift). |
| **IoT vývoj a metodika** | |
| [`IoT_Methodology.md`](docs/IoT_Methodology.md) | Pravidla a workflow pro LLM-asistovaný vývoj IoT systémů. |
| [`poc_build_guide_v1.md`](docs/poc_build_guide_v1.md) | Návod na sestavení prvního IoT PoC z existujícího HW. |
| [`crop_IoT_monitoring.md`](docs/crop_IoT_monitoring.md) | IoT senzorová příručka pro polní kulturu (teplota, VPD, půdní vlhkost, GCP pipeline). |
| **Architektura a konfigurace** | |
| [`Outpost_IoT_Session_Handoff_Security_v2.json`](docs/Outpost_IoT_Session_Handoff_Security_v2.json) | Architektonický handoff – definice modulů, prahů a rizik. |
| [`Outpost_IoT_Security_PoC_v0.1.json`](docs/Outpost_IoT_Security_PoC_v0.1.json) | Konfigurační data a parametry aktuální verze prototypu. |
| **Infrastruktura** | |
| [`cloud.md`](docs/cloud.md) | Popis serverless infrastruktury na Google Cloud Platform. |
| [`hardware.md`](docs/hardware.md) | Specifikace komponent (PIR, ESP32, tenzometry) a schéma zapojení. |
| [`Firmware.md`](docs/Firmware.md) | Dokumentace k logice kódu pro ESP32. |
| **Analýzy a plány** | |
| [`analyza_revitalizace_repozitare_v1.0.docx`](docs/analyza_revitalizace_repozitare_v1.0.docx) | Hluboká analýza repozitáře + P(úspěch) = 0.91. |
| [`revitalizacni_plan_v1.0.docx`](docs/revitalizacni_plan_v1.0.docx) | Plán revitalizace – 5 fází, ~30 dní. |

---

## Subsystémy

| ID | Subsystém | Priorita | Stav |
| :--- | :--- | :--- | :--- |
| T01 | Security perimetr (PIR + weight) | HIGH | koncept |
| T02 | DS18B20 teplota heatsinku | CRITICAL | MVP hotovo |
| T03 | Modbus telemetrie (POW-HVM3.2H) | HIGH | MVP |
| T04 | Náhradní NTC termistor | MEDIUM | blocker |
| T05 | JK BMS monitoring (JST PH TTL) | HIGH | blocker |
| T06 | BMP180 klimatická stanice | LOW | koncept |
| T07 | GCP cloud + Telegram notifikace | MEDIUM | plán |

---

## Adresářová struktura

```
├── docs/                  # Dokumentace
├── firmware/              # ESP32/ESP8266 firmware moduly
├── hardware/              # Schemata, BOM, pinouty
├── cloud/                 # GCP infrastruktura
├── tests/                 # Testy
├── data/                  # Vzorová data
└── images/                # Fotky, schemata
```

---

## Fáze revitalizace

| Fáze | Název | Stav |
| :--- | :--- | :--- |
| F0 | Konsolidace struktury | ✅ HOTOVO |
| F1 | IoT MVP (DS18B20, Modbus, BMS) | ⏳ |
| F2 | Firmware vrstva (ESPHome) | ⏳ |
| F3 | Security perimetr | ⏳ |
| F4 | Cloud + notifikace | ⏳ |

---

*Poslední aktualizace: 27. 7. 2026* *Licence: Apache 2.0*