# Outpost IoT Security Perimeter – Proof of Concept

## Přehled

Tento projekt řeší problém zabezpečení odlehlé chaty v údolí proti vniknutí nepovolaných osob, přičemž musí spolehlivě ignorovat pohyb zvěře (kočky, srnky, jeleni). Systém je navržen jako **dvoustupňová validace**:

1. **PIR senzor** trvale monitoruje prostor před vstupem a probouzí ESP32 z deep‑sleep.
2. **Tenzometrická váha** změří hmotnost objektu, který na ni stoupne. Při překročení prahu **40 kg** (dospělý člověk) se aktivuje kamera a pořídí záznam.

Systém je energeticky soběstačný (napájen z off‑grid 24V baterie 15,12 kWh) a ukládá data lokálně na SD kartu. Tento dokument popisuje architekturu, hardware, software a postup pro implementaci **prvního ověřovacího prototypu (PoC)**.

---

## Architektura

![Architektura PoC]

### Hlavní komponenty

| Komponenta | Funkce |
|------------|--------|
| **ESP32** | Řídicí jednotka, deep‑sleep, ovládání periferií |
| **PIR (AM312)** | Probouzeč – detekce pohybu |
| **HX711 + 4× tenzometr 50 kg** | Měření hmotnosti na nášlapné desce |
| **ESP32‑CAM** | Pořízení fotografie/videa při potvrzeném poplachu |
| **DC‑DC měnič 24V→5V** | Napájení celého systému |
| **MOSFETy** | Spínání napájení HX711 a kamery pro úsporu energie |

---

## Logika chování

1. **Deep‑sleep (výchozí stav)** – ESP32 uspán, napájeny pouze PIR (a případně obvody pro udržení RTC).
2. **Pohyb detekován** – PIR probudí ESP32 přes přerušení.
3. **Měření hmotnosti** – ESP32 sepne MOSFET pro HX711, počká 200 ms, provede 5 měření, zprůměruje a převede na kg.
4. **Vyhodnocení**:
   - Hmotnost **< 2 kg** – ignorováno, systém usne.
   - Hmotnost **2–35 kg** – událost se zapíše do logu, nefotí.
   - Hmotnost **> 40 kg** – sepne MOSFET pro kameru, pořídí 3 snímky nebo 10s video, uloží na SD kartu, zapíše do logu.
5. **Návrat do deep‑sleep** – po dokončení všech operací (nebo po timeoutu 10 s bez zatížení) se periferie vypnou a ESP32 usne.

---

## Hardware setup

### Seznam součástek

- ESP32 dev kit (Libre)
- PIR senzor AM312 (nebo RCWL‑0516)
- HX711 modul + 4 ks tenzometrů 50 kg (např. z kuchyňské váhy)
- ESP32‑CAM s OV2640 a slotem na microSD
- MOSFET modul IRF520 nebo samostatné MOSFETy (2 ks)
- DC‑DC měnič 24V→5V (3A)
- Odpory 10kΩ, 20kΩ (dělič)
- TVS dioda 3.3V (SMAJ3.3A)
- IP67 box pro ESP32 a HX711
- IP65 box pro ESP32‑CAM
- microSD karta (8–32 GB)

### Zapojení

| ESP32 pin | Periferie | Poznámka |
|-----------|-----------|----------|
| GPIO 13 | PIR OUT | Interrupt (RTC) |
| GPIO 14 | MOSFET HX711 | Spíná 5V pro HX711 |
| GPIO 15 | MOSFET Camera | Spíná 5V pro ESP32‑CAM |
| GPIO 4 | HX711 DT | Data |
| GPIO 5 | HX711 SCK | Clock |
| 3.3V | PIR VCC | PIR napájen trvale |
| 5V (z DC‑DC) | ESP32 VIN | |

**Poznámka:** Pro HX711 je třeba odporový dělič na výstupu, pokud HX711 pracuje na 5V a ESP32 na 3.3V – použijte 10k/20k dělič na DT a SCK.

---

## Software

Firmware je napsán v Arduino C++ pro ESP32. Klíčové knihovny:

- `HX711` – čtení tenzometrů
- `esp32-camera` – ovládání kamery
- `SD` – ukládání na SD kartu
- `EEPROM` – uložení kalibračních konstant

Zdrojový kód je strukturován do stavového automatu a zahrnuje:

- Nastavení deep‑sleep a wake‑up zdrojů
- Auto‑táru před každým měřením
- Kalibraci pomocí závaží (uložení škálovacího faktoru do EEPROM)
- Logování událostí do CSV souboru
- Ovládání MOSFETů pro úsporu energie

### Kalibrace

1. Umístěte závaží 20 kg na váhu, změřte hodnotu.
2. Umístěte závaží 50 kg, změřte.
3. Vypočítejte faktor (rozdíl hodnot / rozdíl hmotnosti) a offset.
4. Uložte do EEPROM. Při každém startu se faktor načte.

---

## Instalace na místě

1. **Nášlapná deska** – umístěte těsně před vchod do místnosti 2 (izolované jádro). Deska by měla být lehce zapuštěna, aby na ni bylo nutné stoupnout. Tenzometry připevněte ke spodní straně desky a zatmelte (pro PoC postačí provizorní krytí).
2. **PIR senzor** – namontujte nad dveře pod stříšku. Nasměrujte na prostor před deskou. Přidejte mechanickou síťku proti pavoukům.
3. **ESP32 + HX711** – umístěte do suchého boxu uvnitř místnosti 2.
4. **ESP32‑CAM** – upevněte naproti vchodu (např. na stěnu pergoly) v IP65 boxu s průhledným okénkem. Nasměrujte tak, aby zabírala dveře i váhu.
5. **Napájení** – připojte k 24V z off‑grid systému (přes DC‑DC měnič). Dbejte na správné jištění (pojistka 2A).

---

## Testování

### Testovací scénáře

1. **Kočka** – měla by probudit systém, ale nefotit (zaznamená se do logu).
2. **Srnec (simulace 20 kg)** – stejně jako kočka.
3. **Dospělý** – systém musí vyfotit a uložit záznam.
4. **Vítr / listí** – může probudit, ale bez zatížení se do 10 s uspí, žádný záznam.
5. **Teplotní změny** – sledujte drift; v případě problémů implementujte teplotní kompenzaci.

### Kritéria úspěšnosti

- Během 7 dnů testu **0 falešných alarmů** (fotografií bez skutečného člověka).
- **100% detekce** vstupu osoby (při standardním použití).
- Stabilní napájení a žádné pády systému.

---

## Rizika a řešení

| Riziko | Řešení |
|--------|--------|
| Falešné PIR poplachy | Mechanická síťka, logická vazba AND s váhou (bez váhy nefotí) |
| Vlhkost poškodí tenzometry | Pro PoC krátkodobé nasazení; po validaci přechod na pneumatickou pastu |
| Zvíře zůstane stát na váze | Timeout 30 s bez změny hmotnosti → uspání bez záznamu |
| Noční záznamy tmavé | Přidat IR přísvit spínaný společně s kamerou |
| Přepětí na ADC pinu | Odporový dělič + TVS dioda |

---

## Další kroky po PoC

- Vyhodnotit přesnost a spolehlivost.
- Pokud PoC uspěje:
  - Nahradit tenzometry **pneumatickou pastou** (MPS20N0040D, měch, kapilára) pro vyšší odolnost.
  - Rozšířit perimetr o další vstupní body (okna, severní hrana).
  - Implementovat teplotní kompenzaci (DS18B20 + stavová rovnice).
  - Zvážit vzdálené notifikace (GSM/LTE) pro ostrý provoz.

---

## Odkazy na související dokumenty

- [Outpost IoT Session Handoff Security v2](./Outpost_IoT_Session_Handoff_Security_v2.json)
- [Outpost 2026 Datasheet](./Outpost_2026_datasheetv_2.md)
- [Handoff JSON – PoC](./Outpost_IoT_Security_PoC_v0.1.json)

---

## Licence a autorská práva

Tento projekt je vyvíjen pro soukromé účely na pozemku Outpost 2026. Veškerá dokumentace je poskytována „jak je“ bez záruk.

---

# Report: Posouzení PoC „Outpost IoT Security Perimeter“

**Dokument:** Outpost_IoT_Security_PoC_v0.1.json  
**Prostředí:** Outpost_2026 (Praha, off-grid)  
**Datum analýzy:** 2026-04-01

---

## 1. Shrnutí (Executive Summary)

**Hlavní závěr:** PoC je **technicky relevantní a má vysoký potenciál úspěchu**, avšak jeho implementace naráží na dva kritické faktory definované v podkladových materiálech: **extrémní mikroklima (vlhkost, teplotní výkyvy)** a **stavebně-geologický deficit objektu (zaříznutá severní hrana)**. Tyto faktory nebyly v architektuře PoC dostatečně reflektovány, což představuje hlavní riziko.

---

## 2. Originalita a přístup k řešení

| Aspekt | Hodnocení |
|--------|-----------|
| **Skóre** | **9/10 – Vysoká invence v rámci dané třídy problémů** |

Koncept je originální v tom, že opouští standardní přístup zabezpečení (plot, kamerový systém s AI) a místo toho vytváří **fyzický „filtr“** založený na základní fyzikální veličině – hmotnosti. Tento přístup je:

1. **Deterministický:** Váha nelže. Na rozdíl od PIR nebo AI modelu je výsledek (člověk vs. zvěř) jasně definován prahem.
2. **Nízkonákladový:** Využívá dostupné a levné komponenty (ESP32, tenzometry, HX711).
3. **Elegantně řeší problém falešných poplachů:** Logická vazba **AND (PIR A hmotnost > práh)** je extrémně účinná pro eliminaci rušení způsobeného hmyzem, listím nebo malými zvířaty.
4. **Respektuje off-grid realitu:** Koncept „RAW_FIRST“ a priorizace lokálního úložiště nad cloudovou komunikací je naprosto správný v místě bez stabilního internetového připojení.

**Slabinou invence** není myšlenka samotná, ale její **mechanická realizace v terénu**. Použití tenzometrů a HX711 v exteriéru je invenčně odvážné, ale z hlediska dlouhodobé udržitelnosti rizikové.

---

## 3. Analýza technického řešení (Proveditelnost a robustnost)

| Aspekt | Hodnocení |
|--------|-----------|
| **Skóre** | **7/10 – Funkční prototyp, vyžaduje významné úpravy pro produkční nasazení** |

### Silné stránky

- **Architektura spotřeby:** Výpočet energetické bilance (<1 Wh/den) je realistický a systém je perfektně dimenzován na kapacitu baterie (17,52 kWh). MOSFET spínání periferií z ESP32 je správná praxe pro deep-sleep režimy.
- **Softwarová logika:** Stavový automat (state machine) je přehledný a pokrývá všechny důležité scénáře (timeout, auto-tára, logování). Logování na SD kartu je nezbytné pro validaci.
- **Riziková analýza:** Identifikace rizik (pavouci, vlhkost, deep-sleep) a navržená mitigace (mechanická síťka, děliče napětí, zatmelení) svědčí o zkušenosti.

### Kritické slabiny a nesoulad s kontextem Outpost_2026

#### 3.1 Klimatická odolnost (Nedostatečné řešení)

| Problém | Detail |
|---------|--------|
| **Vlhkost** | Dokument uvádí ranní vlhkost až **80 %**. Tenzometry lepené na ocelovém nosníku a nekvalifikované „zatmelení“ (specifikováno jako „pro PoC dočasně“) v tomto prostředí **do týdne zkorodují nebo ztratí kontakt**. Toto je největší technické riziko. |
| **Teplotní fluktuace** | Rozsah -7 °C až +20 °C v březnu způsobí významný teplotní drift HX711 a tenzometrů. Auto-tára před každým měřením tento drift eliminuje *pouze pro nulovou hodnotu*, ale neřeší nelinearitu v celém rozsahu. Bez teplotní kompenzace (DEBT_PoC_001) bude měření váhy v reálném provozu nespolehlivé. |

#### 3.2 Integrace s fyzickým objektem

| Problém | Detail |
|---------|--------|
| **Zaříznutá severní hrana** | Dokumentace zmiňuje, že severní hrana objektu je 0,4 m pod úrovní terénu a vyžaduje **okamžitou sanaci**. Pokud není tato sanace provedena *před* instalací elektroniky, hrozí trvalé podmáčení základů a vzlínání vlhkosti do suchého boxu s ESP32. PoC ignoruje tento strukturální deficit a předpokládá, že vnitřní prostředí je suché. |

#### 3.3 Mechanická integrace (Nášlapná deska)

| Problém | Detail |
|---------|--------|
| **Nášlapná deska** | Deska 50×50 cm je logická, ale její instalace na svažitém terénu (převýšení 4,98 m na pozemku) a před vstupem do objektu bude vyžadovat stabilní a vyrovnaný podklad. Implementace tohoto detailu v terénu bude pravděpodobně časově náročnější, než předpokládá koeficient 1,5×. |

---

## 4. Analýza ROI (Návratnost investice) a strategická hodnota

| Aspekt | Hodnocení |
|--------|-----------|
| **Skóre** | **8/10 – Extrémně vysoká hodnota pro informovanost, nulová pro fyzickou ochranu** |

Návratnost investice je třeba posuzovat v kontextu obchodních vektorů definovaných v `datasheetu` (Vektor C: Asanace, Dřevěné prvky; Vektor E: Diagnostika, Automatizace, IoT).

| Faktor | Zhodnocení |
|--------|------------|
| **Nízké materiální náklady** | Celkové náklady na komponenty (ESP32, ESP32-CAM, HX711, tenzometry, PIR, MOSFETy) se pohybují v řádu 2 000–3 000 Kč. To je zanedbatelná investice. |
| **Vysoká strategická hodnota (Vektor E)** | Tento PoC je **demonstračním artefaktem** pro Vektor E (Automatizace, IoT). Jeho úspěšná implementace a funkčnost v drsném off-grid prostředí je silným prodejním argumentem pro budoucí zakázky v podobném duchu. |
| **Hodnota informace** | Primárním přínosem není fyzické zabránění vstupu (plot, zámek), ale **sběr dat**. Systém poskytne přesný přehled o tom, kdo (a zda vůbec) vstupuje na perimetr. To je pro správce odlehlé nemovitosti klíčové pro situační vědomí. |
| **Nízká fyzická ochrana** | Systém je pasivní. V případě vloupání pouze pořídí záznam, který bude uložen na SD kartu uvnitř objektu. Zloděj, který nalezne hlavní box, si kartu může vzít s sebou. Pro skutečné zabezpečení by byl nutný real-time alert (DEBT_PoC_002), což je v tomto místě bez investice do GSM/LTE konektivity obtížné. |

---

## 5. Vhodnost pro prostředí (Shoda s Operačními Limity)

| Aspekt | Hodnocení |
|--------|-----------|
| **Skóre** | **6/10 – Konceptuálně vhodné, ale podceněny jsou logistické a environmentální vstupní podmínky** |

| Operační limit | Shoda / Nesoulad |
|----------------|------------------|
| **Koeficient 1,5×** | Návrh pravděpodobně podceňuje čas na mechanickou instalaci váhy do svahu a utěsnění kabelových průchodek proti vlhkosti. Odhad času na instalaci by měl být minimálně dvojnásobný oproti laboratornímu sestavení. |
| **Logistické okno (1,5 h)** | Není zohledněna skutečnost, že při případném výpadku nebo nutnosti přeprogramování bude cesta na místo a zpět zabírat 3 hodiny. Firmware by proto měl mít možnost bezdrátové aktualizace (OTA) přes lokální Wi-Fi (pokud je vytvořena), nebo být extrémně robustní. |
| **Stop-stav LiFePO₄ (0 °C)** | Systém je napájen z 24V baterie. Pokud dojde k vybití hlavní baterie během mrazivého období a následnému pokusu o nabíjení solárem bez předehřevu, BMS (Jikong) nabíjení zablokuje. PoC tento scénář neřeší a předpokládá, že 24V je vždy k dispozici. V případě delšího deštivého období a nízkých teplot by mohl systém zůstat bez napájení. |

---

## 6. Závěr a doporučení

### Verdikt

PoC je **schválen k realizaci v omezeném rozsahu**, ale s podmínkou revize technického provedení s ohledem na vlhkost a teplotní stabilitu. Není vhodný pro okamžité produkční nasazení bez zapracování následujících doporučení.

### Doporučený postup (Aktualizace Next Steps)

1.  **Před zahájením (Krok 0):** **Sanovat severní hranu objektu.** Dle `datasheetu` je to kritické rozhraní. Bez zajištění drenáže a izolace (nopová fólie) není vhodné instalovat do objektu jakoukoli citlivou elektroniku s dlouhodobým výhledem.

2.  **Změna mechanického návrhu váhy:** Pro PoC nepoužívejte tenzometry lepené na oceli. Pro rychlé ověření logiky použijte **lacinou osobní váhu s digitálním výstupem** (připájet se na výstup z ADC). To eliminuje riziko vlhkosti a mechanické nestability v prvních týdnech testování. Teprve po validaci logiky řešte robustní mechaniku (pneumatická pasta z DEBT_PoC_003).

3.  **Implementace teplotní kompenzace (DEBT_PoC_001):** Povýšit z "MEDIUM" na **"HIGH" prioritu pro PoC**. Přidat DS18B20 do boxu s HX711 a upravit firmware pro korekci hodnoty hmotnosti. V prostředí s denními výkyvy 27 °C je to nutnost pro získání smysluplných dat.

4.  **Úprava plánu logistiky:** Do rozpočtu času zahrnout 1 den na instalaci a utěsnění kabelových prostupů do suchých boxů (použití PG šroubení, ne silikonu).

---

## 7. Celkové hodnocení projektu

Jedná se o **velmi slibný a inovativní koncept**, který má potenciál stát se vzorovým řešením pro perimetrovou detekci v off-grid lokalitách. Jeho úspěch však nestojí na softwarové logice (která je dobrá), ale na **mechanické robustnosti a odolnosti vůči environmentálním stresorům**, které byly v počátečním návrhu podceněny.

Po zapracování výše uvedených bodů má projekt vysokou šanci na úspěch a poskytne cenná data pro budoucí rozšíření perimetru.

---

| Kritérium | Hodnocení |
|-----------|-----------|
| Invence | 9/10 |
| Technické řešení | 7/10 |
| ROI | 8/10 |
| Vhodnost pro prostředí | 6/10 |
| **Celkem** | **7,5/10** |
