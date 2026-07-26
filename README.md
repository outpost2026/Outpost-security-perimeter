# Outpost IoT bezpečnostní perimeter

## Deterministické zabezpečení perimetru – váhový filtr jako jádro detekce

Projekt řeší zásadní problém zabezpečení odlehlé chaty v jednom z mnoha **pražských údolí** – prostředí, kde běžná PIR čidla, Doppler radary generují nepřijatelné množství falešných poplachů. Namísto slepého spoléhání na jeden senzorický princip staví systém na **dvoustupňové validaci**, jejíž jádro tvoří **váhový detektor**.

**Základní odlišnost systému:**
* **Fyzikální validace:** Váha (tenzometrická nebo pneumatická) měří hmotnost objektu přímo v místě vstupu.
* **Deterministický práh (>40 kg):** Jasné kritérium pro rozlišení člověka od zvěře (srnec, kočka, pes).
* **Energetická efektivita:** Bez potvrzení hmotnosti nedochází k aktivaci kamery ani energeticky náročnému přenosu dat.

---

## Cíl PoC (Proof of Concept)

Ověřit schopnost systému spolehlivě detekovat dospělou osobu a ignorovat pohyb zvěře v off-grid režimu.
* **Lokalita:** Vstup do technické místnosti (izolovaná sekce s cennostmi).
* **Napájení:** 24V baterie (15,12 kWh), cílová spotřeba systému **<1 Wh/den**.
* **Odolnost:** Provoz v teplotách −7 °C až +20 °C a vysoké vlhkosti.

---

## Princip detekce – dvoustupňová validace

1.  **Úroveň 1 (Probuzení):** Ultra-low-power senzor (PIR/Doppler) neustále monitoruje okolí. Při detekci pohybu probudí řídicí jednotku **ESP32** z hlubokého spánku.
2.  **Úroveň 2 (Potvrzení):** ESP32 aktivuje váhu. Pokud je naměřená hmotnost **vyšší než 40 kg**, je vyhlášen poplach (aktivace kamery, logování do cloudu). Pokud je nižší, systém událost pouze zaloguje a vrací se do spánku.

---

## Repozitář – struktura a obsah

| Soubor | Popis |
| :--- | :--- |
| Soubor | Popis |
| :--- | :--- |
| [`docs/koncepce_zabezpeceni.md`](docs/koncepce_zabezpeceni.md) | Detailní rozbor strategie ochrany perimetru a metodiky detekce. |
| [`docs/differential_analysis.md`](docs/differential_analysis.md) | Srovnání s komerčními systémy a typickými IoT projekty. |
| [`docs/Testovaci_scenare.md`](docs/Testovaci_scenare.md) | Metodika testování (simulace zvěře vs. člověka, teplotní drift). |
| [`docs/Outpost_IoT_Session_Handoff_Security_v2.json`](docs/Outpost_IoT_Session_Handoff_Security_v2.json) | Architektonický handoff – definice modulů, prahů a rizik. |
| [`docs/Outpost_IoT_Security_PoC_v0.1.json`](docs/Outpost_IoT_Security_PoC_v0.1.json) | Konfigurační data a parametry aktuální verze prototypu. |
| [`docs/cloud.md`](docs/cloud.md) | Popis serverless infrastruktury na Google Cloud Platform. |
| [`docs/hardware.md`](docs/hardware.md) | Specifikace komponent (PIR, ESP32, tenzometry) a schéma zapojení. |
| [`docs/Firmware.md`](docs/Firmware.md) | Dokumentace k logice kódu pro ESP32. |
| [`docs/analyza_revitalizace_repozitare_v1.0.docx`](docs/analyza_revitalizace_repozitare_v1.0.docx) | Hluboká analýza repozitáře + P(úspěch) = 0.91. |
| [`docs/revitalizacni_plan_v1.0.docx`](docs/revitalizacni_plan_v1.0.docx) | Plán revitalizace – 5 fází, ~30 dní. |

---

## Rozšířený scope — Outpost IoT Hub

Repozitář byl revitalizován z původního "security perimeter" na **Outpost IoT Hub** — monitoring, automatizace a bezpečnost off-grid uzlu.

**Nové subsystémy:**
| ID | Subsystém | Priorita | Stav |
| :--- | :--- | :--- | :--- |
| T01 | Security perimetr (PIR + weight) | HIGH | koncept |
| T02 | DS18B20 teplota heatsinku | CRITICAL | MVP |
| T03 | Modbus telemetrie (POW-HVM3.2H) | HIGH | MVP |
| T04 | Náhradní NTC termistor | MEDIUM | blocker |
| T05 | JK BMS monitoring (JST PH TTL) | HIGH | blocker |
| T06 | BMP180 klimatická stanice | LOW | koncept |
| T07 | GCP cloud + Telegram notifikace | MEDIUM | plán |

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

## Fáze revitalizace

| Fáze | Název | Deadline | Status |
| :--- | :--- | :--- | :--- |
| F0 | Konsolidace struktury | den 1 | ✅ HOTOVO |
| F1 | IoT MVP (DS18B20, Modbus, BMS) | den 1-3 | ⏳ |
| F2 | Firmware vrstva (ESPHome) | den 4-10 | ⏳ |
| F3 | Security perimetr | den 11-20 | ⏳ |
| F4 | Cloud + notifikace | den 21-30 | ⏳ |

---

## Autor

Projekt vyvíjí autodidakt se zaměřením na off-grid systémy, automatizaci a cloudovou infrastrukturu. Tento repozitář slouží jako živá dokumentace vývoje.

*Poslední aktualizace: 1. 4. 2026* *Licence: MIT*
