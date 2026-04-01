# Diferenční analýza: Outpost IoT Ecosystem vs. Referenční skupiny

## 1. Kontext – backendová infrastruktura

funkční serverless stack na GCP, který slouží pro:

- Automatizované scrapování (Bazoš, notebooky, materiály)
- IoT telemetrii (ESP32 → Firestore)
- Meteo pipeline (ČHMÚ Kbely)
- Centrální orchestrací (Cloud Scheduler + Cloud Run)
- Datové úložiště (GCS, Firestore, BigQuery)
- Energeticky optimalizovaný provoz (zero idle cost, VM jen legacy)

Tento backend **není izolovaný** – je navržen jako **univerzální platforma** pro všechny automatizace Outpostu, včetně budoucího rozšíření perimetrové detekce (notifikace, analýza dat).

---

## 2. Srovnávané entity

| Skupina | Charakteristika | Typická výbava |
|---------|-----------------|----------------|
| **Soused** | Zahradník, technicky minimálně kompetentní | 2× IP kamera, LED reflektor, proprietární cloud, datová SIM |
| **Běžný uživatel** | Zákazník e-shopu, občasný kutil | 1–2 kusové IoT zařízení (chytrá zásuvka, bezpečnostní kamera) s mobilní aplikací |
| **Typický IoT projekt z GitHubu** | Student/hobbyista, jedna úloha, často jen Arduino + MQTT | ESP32 + senzor → MQTT → lokální broker nebo jednoduchý cloud (ThingSpeak) |
| **Potenciální klient** | Správce nemovitosti / malá firma hledající komplexní řešení | Očekává hotové řešení, často se spoléhá na dodavatele (např. zabezpečení, monitoring) |
| **Outpost operátor (tento případ)** | Autodidakt, systematický inženýrský přístup | **Dvoustupňový HW PoC** (PIR+váha) + **plně serverless GCP backend** (Cloud Run, Scheduler, Firestore, BigQuery) + dokumentace, metodika, roadmapa |

---

## 3. Diferenční analýza – komparativní tabulka

| Kritérium | Soused | Běžný uživatel | Typický IoT projekt | Potenciální klient | **Outpost operátor** |
|-----------|--------|----------------|---------------------|---------------------|----------------------|
| **HW detekce** | PIR v kameře | PIR, dveřní senzor | Jeden senzor (PIR, teplota) | Podle dodavatele | **PIR + váha (fyzický filtr)** – eliminuje falešné alarmy |
| **Záznam / data** | Proprietární cloud, měsíční poplatek | Aplikace výrobce | Lokální SD nebo jednoduchý cloud | Různé | **Lokální SD + GCS + Firestore** (dvojí úložiště) |
| **Backend infrastruktura** | Žádná (pouze cloud výrobce) | Žádná | Jednoduchý broker / ThingSpeak | Obvykle outsourcovaná | **Full-stack GCP serverless** – Cloud Run, Scheduler, BigQuery, Firestore, 10+ služeb |
| **Automatizace / orchestrace** | Pouze notifikace v aplikaci | Žádná | Základní cron (ESP8266) | Závisí na dodavateli | **6 scheduler jobů**, orchestrátor, meteo pipeline, scraping, IoT ingest |
| **Energetická optimalizace** | Nízká (kamery 24/7) | Nízká | Často žádná | Neřeší | **Deep-sleep (<0,1 mW), MOSFET spínání, VM terminovaná, zero idle cost** |
| **Provozní nezávislost** | Závislost na SIM a cloudu | Závislost na výrobci | Částečná (může být lokální) | Závislost na dodavateli | **Plně lokální možnost + volitelná cloudová vrstva** – dual mode |
| **Dokumentace** | Žádná | Žádná | Často chybí nebo základní README | Vyžaduje manuál | **Kompletní dokumentace**: architektura, pinout, kalibrace, testovací scénáře, roadmapa, GCP Stack Ingest v3 |
| **Škálovatelnost / rozšiřitelnost** | Fixní počet kamer | Fixní | Omezená | Omezená | **Modulární**: lze přidat senzory, rozšířit perimetr, přepnout na pneumatickou pastu, přidat GSM notifikace |
| **Náklady na provoz** | Měsíční poplatek za SIM | Žádné (kromě elektřiny) | Nízké (elektřina) | Závisí | **0 Kč/měsíc** (bez SIM, GCP v free tieru) |
| **Náročnost nasazení** | Nízká (koupit, zavěsit) | Nízká | Střední (programování, pájení) | Vysoká (dodavatel) | **Vysoká** (mechanická integrace, kalibrace, GCP nastavení) |
| **Přenositelnost know-how** | Nulová | Nulová | Omezená (jen daný projekt) | Závisí na dodavateli | **Vysoká** – metodika, kód, infrastruktura jako kód, dokumentace |
| **Hodnota pro portfolio / zaměstnavatele** | Nízká (spotřebitelská zkušenost) | Nízká | Střední (základní technický projekt) | N/A | **Extrémně vysoká** – komplexní HW↔SW, cloud, metodika, dokumentace |

---

## 4. Detailní rozbor rozdílů

### 4.1. Backendová infrastruktura – největší diferenciace

**Soused, běžný uživatel, typický IoT projekt** – jejich "cloud" je buď proprietární (kamera), nebo jednoduchá datová schránka (ThingSpeak, Firebase). Nikdo z nich nemá:

- **Serverless architekturu** s orchestrací (Cloud Scheduler → Cloud Run → GCS)
- **Víceúčelový backend** sloužící současně pro scrapování, IoT ingest, meteo pipeline a budoucí perimetr
- **Datové sklady** (BigQuery) pro analýzu
- **Formalizovaný stack** s verzováním, snapshoty a direktivami (serverless-first)

**Outpost operátor** má tyto komponenty plně funkční a zdokumentované v `gcp_stack_ingest_v3.md`. To je důkaz, že umí navrhovat a provozovat **produkční cloudové systémy** – nejen jednorázové prototypy.

### 4.2. Energetická optimalizace a náklady

Zatímco ostatní skupiny buď **nepočítají s off‑grid** (soused bere energii ze sítě), nebo **používají jednoduché úsporné režimy** (ESP deep‑sleep), operátor:

- Navrhl systém pro **off‑grid 24V s 15 kWh baterií**
- Dimenzoval spotřebu na <1 Wh/den (HW PoC)
- V GCP využívá **zero idle cost** – VM terminovaná, Cloud Run se škáluje na nulu
- Má **6 scheduler jobů** spouštějících serverless funkce bez trvalých nákladů

Tato kombinace je **unikátní** – většina hobby projektů neřeší náklady na cloud ani energetickou bilanci.

### 4.3. Dokumentace a přenositelnost

Typické IoT projekty na GitHubu obsahují:

- README s pár větami
- Schéma zapojení (často jen foto)
- Kód bez komentářů

Outpost operátor poskytuje:

- **Kompletní architekturu** (stavový automat, pinout, kalibrace)
- **Testovací scénáře** a kritéria úspěšnosti
- **Rizika a mitigace** (vlhkost, teplotní drift, deep‑sleep)
- **Roadmapu** s technickými debt items
- **GCP Stack Ingest v3** – detailní snapshot cloudové infrastruktury s verzemi a poznámkami
- **Metodiku** (anti‑vibe, RAW_FIRST) a pitevní knihu


### 4.4. Potenciální klient – jak by vnímal řešení

Potenciální klient (např. správce nemovitosti, e‑commerce firma s logistikou) hledá:

- **Spolehlivost** – eliminace falešných poplachů
- **Nízké provozní náklady** – žádné měsíční poplatky
- **Možnost rozšíření** – neuzavřený systém
- **Transparentnost** – ví, co systém dělá
- **Podporu a dokumentaci** – aby mohl systém spravovat sám nebo předat

Outpost řešení splňuje **všechny tyto požadavky lépe než běžné IP kamery**:

- Váhový filtr = spolehlivost
- Žádný cloud výrobce = žádné poplatky
- Modulární HW a cloud = škálovatelnost
- Vlastní firmware a dokumentace = transparentnost
- Kompletní dokumentace = předatelnost

Autor může nabídnout **nasazení na míru** – přizpůsobení prahů, integraci s dalšími senzory, napojení na jejich systémy.
