# IoT Development Methodology

**Verze:** 1.0 | **Datum:** 2026-07-26 | **Účel:** Pravidla a workflow pro LLM-asistovaný vývoj IoT systémů
**Zdroje:** Claude_session010, Gemini_session109, Gemini_session111, Claude_session002

---

## 1. Základní principy

### Pravidlo 1: RAW_FIRST

Před jakýmkoli kódem ověř surový výstup senzoru/protokolu v terminálu.

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│ Senzor   │────►│ Serial monitor │────►│ Teprve ted   │
│ (fyzický)│     │  (surová data) │     │ psát parser  │
└──────────┘     └──────────────┘     └──────────────┘
```

**Příklady:**
- JK BMS: RAW UART dump před implementací parseru (ověřit 0x4E 0x57 frame)
- POW Modbus: RAW Modbus rámce v terminálu před ModbusMaster knihovnou
- DS18B20: OneWire scan před DallasTemperature

### Pravidlo 2: Binary MVP

Každý úkol má **binární MVP kritérium** definované **písemně** před fyzickou implementací.

**Formát:** "Vidím [konkrétní data] v [konkrétním umístění]"

| Úkol | Binární MVP | Splněno |
|------|------------|---------|
| T02 | "Teplota heatsinku v sériovém terminálu každou sekundu" | ☐ |
| T03 | "Raw Modbus data z POW střídače v sériovém terminálu" | ☐ |
| T05 | "0x4E 0x57 frame z BMS v sériovém terminálu" | ☐ |
| T06 | "Teplota a tlak z BMP180 v terminálu" | ☐ |

### Pravidlo 3: Stop kritérium

Pokud task nedosáhne binárního MVP do 30 minut → STOP. Neiterovat hluboko do jednoho problému. Delegovat (LLM) nebo přeskočit a vrátit se později.

### Pravidlo 4: Jeden projekt najednou

Dokončit jeden blok před otevřením dalšího. Periferní vidění (paralelní úkoly) generuje 80% completion pattern a scope creep.

---

## 2. Vývojový workflow

```
┌──────────────────────────────────────────────────┐
│               LLM-asistovaný vývoj                │
├──────────────────────────────────────────────────┤
│                                                    │
│  1. Architecture diagram → souhlas → implementace  │
│  2. RAW_FIRST test (surová data v terminálu)       │
│  3. Binary MVP splněno? → ANO: hotovo, NE: krok 4 │
│  4. ESPHome YAML first (ne raw C++)                │
│  5. Pokud ESPHome nepodporuje → MicroPython/C++    │
│  6. Test na fyzickém HW (ne emulátor)              │
│  7. Commit + dokumentace                           │
│                                                    │
└──────────────────────────────────────────────────┘
```

### Dual-track development

| Track | Platforma | Nástroje |
|-------|-----------|---------|
| **Track A** | Dell 5590 (Windows) | Python, GCP SDK, Tera Term |
| **Track B** | ESP32 / D1 mini (fyzický HW) | esptool, mpremote, Arduino IDE |

LLM asistuje v obou trackách, ale fyzický HW test probíhá vždy na reálném zařízení.

---

## 3. Anti-patterny

### ❌ "Nejprve se naučím nástroj dokonale"

**Utilitární vztah k nástrojům:** Skloň do hloubkového studia nástroje při blokaci místo inkrementálního testování.

✅ **Řešení:** "Rozumím dostatečně pro tento konkrétní výstup." Software nemá přirozený strop pochopení — implicitní standard úplnosti je pohyblivý cíl.

### ❌ "Udělám to perfektně hned"

**Implicitní standard úplnosti:** Frustrace z nedokonalého řešení.

✅ **Řešení:** Stabilita je post-MVP iterace. První verze je "dostatečná pro tento výstup".

### ❌ Scope creep při 80% hotovo

**80% completion pattern:** Při blízkosti MVP se otevře 5 nových podúkolů.

✅ **Řešení:** Definovat MVP písemně předem. Pokud MVP splněn → hotovo. Nové úkoly jdou do backlogu.

---

## 4. Komunikační protokol s LLM

### Před každým krokem

```
Context:
- fyzické zapojení: [schema / foto]
- očekávaný výstup: [RAW data / log / HTTP response]
- binární MVP: [jedna věta]
- blocker: [co nevíš / nemáš]
```

### Během debugování

1. Ukaž LLM **surový výstup** (ne interpretaci)
2. Ukaž **schema zapojení** (pinout, napětí)
3. Uveď **co jsi zkoušel a výsledek**
4. LLM navrhne hypotézu → ty otestuješ → LLM analyzuje výsledek

### Po dokončení

```markdown
## [Task ID] — [Název]
**Status:** ✅ splněno / ❌ blokováno
**Binární MVP:** [kritérium]
**Splněno:** [datum]
**Poznámky:** [co se naučeno, co překvapilo]
```

---

## 5. Pitevní kniha (Pit-evni kniha)

Každý nečekaný problém nebo chyba se zapisuje do pitevní knihy.

### Formát záznamu

```yaml
id: PB-001
date: 2026-07-26
symptom: [co se stalo]
cause: [proč se to stalo]
fix: [co problém vyřešilo]
prevention: [jak předejít]
```

### Kategorie

| Kategorie | Prefix | Příklady |
|-----------|--------|----------|
| IoT/Embedded | PB-IOT- | EMI, UART timing, heap fragmentation |
| General Coding | PB-GEN- | Python edge case, API změna |
| Cloud/Infra | PB-CLD- | IAM timeout, cold start latency |

---

*Syntetizováno z Claude_session010 (Pravidla 6+7, RAW_FIRST, binary MVP), Gemini_session109 (metodika vývoje), Gemini_session111 (dual-track, Imerze), Claude_session002 (DEBT tracking)*