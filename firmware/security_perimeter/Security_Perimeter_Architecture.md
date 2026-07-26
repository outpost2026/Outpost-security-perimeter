# Security Perimeter Architecture

**Verze:** 1.0 | **Datum:** 2026-07-26
**Zdroje:** Claude_session002, Gemini_session047, Gemini_session081, Grok_session010, Grok_session012
**Prostředí:** Off-grid dřevostavba, Drahanské údolí, 95% RH, -7°C až +35°C

---

## 1. Vícevrstvá detekce

```
┌─────────────────────────────────────────────────────┐
│                   KASKÁDOVÁ ESKALACE                  │
│                                                       │
│  L0: Otřesové čidlo SW-520D (dveře/okna)             │
│       ↓                                                │
│  L1: Mikrovlnný radar RCWL-0516 (perimetr 5-7m)      │
│       ↓                                                │
│  L2: HW-MS03 doppler (verifikace pohybu)              │
│       ↓                                                │
│  L3: HY-SRF05 ultrazvuk (vzdálenost 20-400 cm)       │
│       ↓                                                │
│  L4: GSM notifikace (A7670E) + bzučák + relé siréna  │
└─────────────────────────────────────────────────────┘
```

## 2. Detekční vrstvy

### L0 — Otřesové čidlo SW-520D (vstupní dveře)

| Parametr | Hodnota |
|----------|---------|
| Typ | Mechanický spínač (kovová kulička) |
| Napětí | 3.3V / 5V |
| Výstup | Digital (HIGH=klid, LOW=otřes) |
| Pull-up | INPUT_PULLUP na ESP |
| Umístění | Rám dveří / okna |

Zapojení: `SW-520D OUT → D3 (GPIO0) → GND`

### L1 — Mikrovlnný radar RCWL-0516 (perimetr)

| Parametr | Hodnota |
|----------|---------|
| Frekvence | 3.2 GHz (Doppler) |
| Dosah | 5-7 m |
| Detekce | Přes Guttafol, dřevo, nekovové materiály |
| Výstup | Digital (HIGH=pohyb, LOW=klid) |
| Výhoda | Imunní vůči změnám teploty (na rozdíl od PIR) |
| Riziko | False positive 30-50 % (zvěř, vítr) |

Zapojení: `RCWL-0516 OUT → D0 (GPIO16) → VCC=5V → GND`

### L2 — HW-MS03 Doppler radar (verifikace)

| Parametr | Hodnota |
|----------|---------|
| Typ | Mikrovlnný Doppler |
| Dosah | ~3 m |
| Role | Druhý senzor pro potvrzení L1 |

### L3 — HY-SRF05 ultrazvuk (vzdálenost)

| Parametr | Hodnota |
|----------|---------|
| Rozsah | 2-400 cm |
| Přesnost | ±3 mm |
| GPIO | TRIG=D1, ECHO=D2 |
| 5V issue | ECHO pin dává 5V → nutný voltage divider pro ESP8266 (3.3V tolerantní) |

**Voltage divider pro HY-SRF05 ECHO (5V→3.3V):**
```
ECHO (5V) --[1kΩ]-- GPIO --[2.2kΩ]-- GND
```

## 3. Kaskádová eskalace

```python
# Eskalacni logika
def security_loop():
    while True:
        l0 = digitalRead(TILT_PIN)      # SW-520D
        l1 = digitalRead(RCWL_PIN)      # RCWL-0516

        if l0 == LOW:                    # otřes
            log("L0: Otres!")
            buzzer(500)                  # krátký píp
            send_telegram("L0: Otres na vstupu")

        if l1 == HIGH:                   # pohyb v perimetru
            l2 = check_hw_ms03()         # verifikace
            distance = measure_ultrasonic()

            if l2 and distance < 200:     # potvrzený pohyb + vzdálenost <2m
                log("L3: Chodec!")
                buzzer_CONTINUOUS()       # trvalý alarm
                siren(ON)                 # siréna (relé)
                send_telegram("ALARM: Naruseni perimetru!")
                delay(30000)              # 30s alarm
                siren(OFF)
            else:
                log("L1/L2: Zver / falesny poplach")
                send_telegram("INFO: Zver v perimetru")
```

### Eskalační matice

| L0 (otres) | L1 (radar) | L2 (MS03) | L3 (vzdálenost) | Akce |
|-----------|-----------|-----------|----------------|------|
| ✅ | — | — | — | Info: otres |
| — | ✅ | — | — | Warning: pohyb |
| — | ✅ | ✅ | — | Warning: verifikovaný pohyb |
| — | ✅ | ✅ | <200 cm | **ALARM** |
| ✅ | ✅ | ✅ | <200 cm | **ALARM + siréna** |
| — | — | — | — | Klid |

## 4. Dead Man's Switch (cloud)

Viz `cloud/Cloud_Architecture.md` §5. Pokud ESP přestane odesílat heartbeat na >15 min:
- Telegram alert: "Node offline"
- Bez ohledu na příčinu (rušička, výpadek, zničení)

## 5. GSM notifikace (budoucí)

Pro případ výpadku WiFi / LTE:

| Komponenta | Účel |
|-----------|-------|
| A7670E Cat-1 | GSM modem (SMS notifikace) |
| SIM karta | Libovolný operátor (O2, T-Mobile) |
| Anténa | Externí (pro zlepšení signálu v údolí) |

> **Fáze 1:** Pouze WiFi + Telegram. GSM modul není zakoupen.

## 6. Umístění senzorů

```
       ┌─────────────────────────┐
       │    Dřevostavba 7×5m     │
       │                         │
       │  ┌─────────────────┐    │
       │  │ Technická míst. │    │
       │  │  ESP32 (hub)    │    │
       │  │  RCWL-0516      │    │
       │  │  HW-MS03        │    │
       │  └─────────────────┘    │
       │                         │
  ─────┤  Vstupní dveře          │
  SW-520D  HY-SRF05              │
       │                         │
       └─────────────────────────┘
                │
           Guttafol folie
           (radar detekuje skrz)
```

## 7. BOM dostupných komponent

| Senzor | Qty k dispozici | Použito v perimetru |
|--------|----------------|---------------------|
| RCWL-0516 | **2 ks** | ✅ L1 (1 ks) + rezerva (1 ks) |
| HW-MS03 | **1 ks** | ✅ L2 |
| HY-SRF05 | **1 ks** | ✅ L3 |
| SW-520D | **5 ks** | ✅ L0 (2×dveře, 3×rezerva) |
| Bzučák shield | **1 ks** | ✅ Lokální alarm |
| Relé shield | **1 ks** | ✅ Siréna (12V, z DC-DC) |

> **Všechny komponenty security perimetru jsou fyzicky k dispozici.** Lze sestavit ihned.

---

*Syntetizováno z Claude_session002 (DEBT tracking), Gemini_session047 (security pipeline), Gemini_session081 (perimetr RCWL + GSM), Grok_session010/012 (HW alokace)*