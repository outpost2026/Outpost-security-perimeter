# Outpost IoT bezpečnostní perimeter

## Deterministické zabezpečení perimetru – váhový filtr jako jádro detekce neoprávněného vstupu

Projekt řeší zásadní problém zabezpečení odlehlé chaty v jednom z mnoha **pražských údolí** – prostředí, kde běžná PIR čidla, Doppler radary generují nepřijatelné množství falešných poplachů (vítr, listí, pavouci, zvěř). Namísto slepého spoléhání na jeden senzorický princip staví systém na **dvoustupňové validaci**, jejíž jádro tvoří **váhový detektor** – prvek, který tento projekt odlišuje od standardních IoT security řešení.

**Základní odlišnost systému:**
- **Váha (tenzometrická nebo pneumatická)** fyzicky měří hmotnost objektu, který vstupuje do chráněného prostoru.
- **Hmotnostní práh >40 kg** je deterministickým kritériem pro rozlišení dospělého člověka od zvěře (kočka do 5 kg, srnec 15–35 kg).
- Bez potvrzení hmotností **nedojde k aktivaci kamery ani záznamu** – eliminace falešných poplachů na fyzikální úrovni.

---



Systém je navržen pro **off‑grid provoz** (napájení z 24V baterie 15,12 kWh) s cílovou spotřebou **<1 Wh/den**, a je odolný vůči vlhkosti, teplotním výkyvům (−7 °C až +20 °C) a přítomnosti zvěře.

---

## Cíl PoC

Ověřit, zda lze v prostředí údolí se silnou složkou nepředvídatelnosti (zvěř, ptáci, drobná zvířena, domácí mazlíčci) spolehlivě detekovat vstup dospělé osoby do chráněného prostoru (vstup do místnosti 2 nebo dveře chaty) a pořídit kamerový záznam, **přičemž systém ignoruje pohyb zvěře** (kočka, srnka, jelen).

PoC se soustředí na jeden kritický bod vstupu – **dveře izolované místnosti 2** (10 m²), kde je umístěno technické centrum a cennosti. Systém musí být energeticky soběstačný z off‑grid zdroje (24V baterie 15,12 kWh) a odolný vůči vlhkosti, teplotním výkyvům a falešným poplachům.

---

## Princip detekce – dvoustupňová validace

### Úroveň 1 – probuzení
Pasivní infračervený (PIR) nebo mikrovlnný (Doppler) senzor s nízkou spotřebou (řádově µA) neustále monitoruje prostor před vstupem. Při pohybu probudí řídicí jednotku (ESP32) z deep‑sleep.

### Úroveň 2 – potvrzení hmotnosti
Po probuzení ESP32 aktivuje **váhu** (pneumatickou nebo tenzometrickou) umístěnou přímo před dveřmi. Váha změří hmotnost objektu, který na ni stoupne:
- **Hmotnost >40 kg** (dospělý člověk) → systém považuje poplach za reálný a spustí kameru.
- **Hmotnost <35 kg** (zvěř, dítě) → poplach se ignoruje, systém se vrátí do spánku (pouze zaloguje).

### Třetí vrstva (volitelná pro PoC)
Pokud je k dispozici kamera s možností rozpoznávání osob (např. ESP32‑CAM s Edge ML), lze provést dodatečné vizuální potvrzení. Pro PoC však postačí **hmotnostní filtr** jako primární rozhodovací mechanismus.

---

## Architektura

Navržená architektura PoC využívá **kombinaci pohybového senzoru a hmotnostní váhy** k dosažení vysoké spolehlivosti detekce osoby při minimalizaci falešných poplachů. Systém je navržen pro off‑provoz s minimální energetickou náročností a respektuje specifika prostředí (vlhkost, teplotní výkyvy, přítomnost zvěře).

**Po úspěšném ověření lze:**
- Rozšířit na celý perimetr (okna, severní hrana pozemku).
- Nahradit tenzometry robustnější **pneumatickou pastí** (uzavřený vzduchový okruh + tlakový senzor) dle původního návrhu.

---

## Repozitář – struktura a obsah

| Cesta | Popis |
|-------|-------|
| `README.md` | Tento dokument – úvod, princip, cíl PoC, architektura. |
| `LICENSE` | MIT licence. |
| `docs/handoff_security_poc.json` | Architektonický handoff – moduly, prahy, rizika, debt. |
| `docs/differential_analysis.md` | Srovnání řešení s typickými IoT projekty a komerčními alternativami. |
| `docs/roadmap.md` | Plánované iterace – od tenzometrů po pneumatickou pastu, rozšíření perimetru, cloudová integrace. |
| `docs/test_scenarios.md` | Testovací scénáře (kočka, srnec, člověk, teplotní drift) a kritéria úspěšnosti. |
| `cloud/gcp_stack_ingest_v3.md` | Detailní popis serverless GCP infrastruktury (zero idle cost). |
| `hardware/README.md` | Placeholder – bude obsahovat schémata, pinout, kalibraci po ověření prototypu. |
| `firmware/README.md` | Placeholder – bude obsahovat zdrojový kód ESP32 po ověření prototypu. |

---

## Aktuální stav

- **Hardware PoC:** Tenzometrická váha + PIR + ESP32‑CAM – návrh dokončen, čeká na instalaci v terénu.
- **Cloud backend:** Plně funkční serverless stack na GCP – aktuálně využíván pro scrapování, meteodata a orchestrací. Připraven pro příjem dat z ESP32.
- **Dokumentace:** Kompletní – handoff, testovací scénáře, roadmapa, diferenciální analýza, GCP stack.

---

## Další kroky

1. **Terénní instalace** tenzometrického prototypu (vstup do místnosti 2).
2. **Kalibrace prahů** na základě reálných dat (závaží, simulace zvěře).
3. **Integrace s GCP** – ESP32 bude odesílat události HTTP POST do Cloud Run.
4. **Nahrazení tenzometrů pneumatickou pastí** – vyšší odolnost vůči vlhkosti a teplotě.
5. **Rozšíření perimetru** o další vstupní body (okna, severní hrana).
6. **Notifikační vrstva** – napojení na Telegram / e-mail přes Make nebo n8n.

---

## Autor

Projekt je vyvíjen jako součást portfolia autora – autodidakta s praktickými zkušenostmi v oblasti off‑grid energetiky, automatizace a cloudové infrastruktury. Tento repozitář slouží jako **živý archiv** a demonstrace inženýrského přístupu k řešení reálných problémů.

*První commit: 2026-04-01*  
*Licence: MIT*
