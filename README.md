# Outpost IoT bezpečnostní perimeter

## Deterministické zabezpečení perimetru – váhový filtr jako jádro detekce

Projekt řeší zabezpečení odlehlého objektu v náročném terénu (pražská údolí), kde běžná PIR čidla nebo radary generují vysoké množství falešných poplachů kvůli zvěři a vegetaci. Systém staví na **dvoustupňové validaci**, jejímž unikátním prvkem je **váhový detektor**.

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
| [`koncepce_zabezpeceni.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/koncepce_zabezpeceni.md) | Detailní rozbor strategie ochrany perimetru a metodiky detekce. |
| [`differential_analysis.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/differential_analysis.md) | Srovnání s komerčními systémy a typickými IoT projekty. |
| [`Testovaci_scenare.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/Testovaci_scenare.md) | Metodika testování (simulace zvěře vs. člověka, teplotní drift). |
| [`Outpost_IoT_Session_Handoff_Security_v2.json`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/Outpost_IoT_Session_Handoff_Security_v2.json) | Architektonický handoff – definice modulů, prahů a rizik. 
| [`Outpost_IoT_Security_PoC_v0.1.json`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/Outpost_IoT_Security_PoC_v0.1.json) | Konfigurační data a parametry aktuální verze prototypu. |
| [`cloud.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/cloud.md) | Popis serverless infrastruktury na Google Cloud Platform (zero idle cost). |
| [`hardware.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/hardware.md) | Specifikace komponent (PIR, ESP32, tenzometry) a schéma zapojení. |
| [`Firmware.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/Docs/Firmware.md) | Dokumentace k logice kódu pro ESP32 a správy spánkových režimů. |
[`README.md`](https://github.com/outpost2026/Outpost-security-perimeter/blob/main/README.md) | Hlavní přehled projektu, principy a cíle. |

---

## Aktuální stav a další kroky

* **Status:** Návrh HW a cloudového backendu (GCP) je dokončen.
* **Fáze:** Příprava na terénní instalaci tenzometrického prototypu.

**Plán vývoje:**
1.  **Kalibrace v terénu:** Nastavení reálných prahů hmotnosti v místě vstupu.
2.  **Integrace GCP:** Odesílání telemetrie a snímků do Cloud Run/Cloud Storage.
3.  **Pneumatická past:** Vývoj odolnější verze senzoru na bázi tlaku vzduchu (náhrada tenzometrů).
4.  **Notifikace:** Implementace varovných zpráv přes Telegram / Signal.

---

## Autor

Projekt vyvíjí autodidakt se zaměřením na off-grid systémy, automatizaci a cloudovou infrastrukturu. Tento repozitář slouží jako živá dokumentace vývoje.

*Poslední aktualizace: 1. 4. 2026* *Licence: MIT*
