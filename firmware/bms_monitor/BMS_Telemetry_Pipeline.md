# BMS Telemetry Pipeline

**Verze:** 1.0 | **Datum:** 2026-07-26
**Zdroje:** Claude_session003, Claude_session012, Gemini_session003, Gemini_session085, Gemini_session107, Gemini_session110
**Komponenty:** JK BMS JK-PB2A16S20P (V19), LiFePO4 630 Ah / ~16 kWh (8S2P)

---

## 1. Komunikační protokol

**Rychlost:** 115200 baud, 8N1
**Úroveň:** TTL 3.3V (JK BMS port)
**Frame start:** `0x4E 0x57` (ASCII "NW")
**Formát:** Big-endian binární frame, ~220+ bajtů

### Hardwarové připojení

```
JK BMS COM port (JST PH 2.0mm)
  1: TX (BMS → ESP)
  2: RX (ESP → BMS)
  3: GND
  4: 3.3V VCC

ESP32 GPIO (UART1)
  9: RX (← BMS TX)
  10: TX (→ BMS RX)  -- pouzita pouze pro inicializaci
```

> **⚠️ Blocker:** JST PH 2.0mm 4pin kabel není fyzicky k dispozici. Dupont (2.54mm) nesedí. Nutno objednat (GME-01).

### Level shifter

ESP32 GPIO jsou 3.3V tolerantní. JK BMS port dává 3.3V TTL — level shifter není nutný (na rozdíl od 5V systémů). Nicméně pro dlouhodobou spolehlivost:
- Doporučen: bi-directional level shifter (TXS0108E)
- Povinné: TVS dioda 3.3V na RX lince pro ochranu proti EMI od 3.2kW měniče

---

## 2. Data Flow

```
[JK BMS] --UART TTL--> [ESP32] --HTTPS POST--> [GCP Cloud Run] --> [Firestore]
                         |                            |
                    [Dell 5590]                   [BigQuery]
                   lokální buffer                  analýza
                    (LittleFS)
```

### Synchronní větev (Edge)
1. ESP32 UART1 čte JK BMS frame každých 60s
2. Parsuje binární protokol → JSON
3. JSON uložen do LittleFS (lokální buffer)
4. Pokud WiFi dostupná: POST na `iot-ingest-beta` Cloud Function
5. Pokud ne: buffering v LittleFS, odeslání při příští konektivitě

### Asynchronní větev (Cloud)
1. `iot-ingest-beta` (Cloud Run) přijímá JSON payload
2. Validace + timestamp přidání
3. Uložení do Firestore (collection: `bms_telemetry`)
4. BigQuery export pro analytiku
5. Telegram notifikace při anomáliích

---

## 3. MicroPython UART Parser (referenční)

```python
import ujson
from machine import UART, Pin
import time

FRAME_START = b'\x4E\x57'  # 0x4E 0x57

class JKBMS:
    def __init__(self, uart_id=1, tx_pin=10, rx_pin=9):
        self.uart = UART(uart_id, baudrate=115200, tx=Pin(tx_pin), rx=Pin(rx_pin))
        self.uart.init(bits=8, parity=None, stop=1)

    def read_frame(self):
        """Precte jeden BMS frame."""
        buf = b''
        while True:
            byte = self.uart.read(1)
            if byte == b'\x4E':
                buf += byte
                byte = self.uart.read(1)
                if byte == b'\x57':
                    buf += byte
                    break
            # timeout
            if len(buf) > 300:  # max frame length
                return None
        # cti zbytek frame (220+ bytu)
        buf += self.uart.read(220)
        return buf

    def parse_frame(self, data):
        """Parsuje frame do JSON."""
        if not data or data[:2] != FRAME_START:
            return None
        result = {
            'cell_voltages': [],  # 8x float
            'total_voltage': 0.0,
            'current': 0.0,
            'soc': 0,
            'capacity_ah': 0,
            'temperatures': [],   # 2x (BMS + battery)
            'balance_status': 0,
        }
        # byte-by-byte parsing dle JK BMS spec
        # (offsety zavisí na verzi firmware BMS)
        # TODO: naplnit z RAW_FIRST testu
        return result

    def get_telemetry(self):
        raw = self.read_frame()
        if raw:
            return self.parse_frame(raw)
        return None
```

---

## 4. RS485 alternativa (pro EMI-resistant vzdálenosti)

Pri vzdálenosti >30 cm od 3.2kW měniče selhává I2C i TTL UART. Resení:

```
[JK BMS] --TTL--> [MAX485 TTL-RS485] --FTP Cat5e 25m--> [MAX485 RS485-TTL] --> [ESP32]
```

**Topologie:** Daisy Chain

| Parametr | Hodnota |
|----------|---------|
| Kabel | FTP Cat5e (stíněný) |
| Max délka | 25 m |
| Ukončení | 120 Ω terminátor pouze u hubu |
| Uzemnění stínění | Pouze u hubu (prevence zemních smyček) |
| Galvanické oddělení | Povinné (izolovaný převodník) |

> **Rozhodnutí:** Fáze 1 používá přímý UART TTL (kabel <1m). RS485 přechod až Fáze 3 pokud EMI způsobí problémy.

---

## 5. SOC Prediction Model

Z historických dat (Claude_session012, Gemini_session107):

| Parametr | Hodnota |
|----------|---------|
| Model | Lineární regrese solárního zisku |
| Korelace | r=0.953 (R²=0.907) |
| Noční úbytek | 3.0 % fixních (standby měniče + BMS) |
| Kalibrace | Automatická (calculate_avg_drop) |
| Výstup | SOC % + Ah (z 630 Ah jmenovité kapacity) |

```python
def calculate_avg_drop(bms_log):
    """Automatická kalibrace nočního úbytku."""
    evening_max = bms_log['evening_soc']
    morning_min = bms_log['morning_soc']
    return (evening_max - morning_min).mean()

def predict_soc(current_soc, solar_kwh, load_kwh, night_drop):
    net = solar_kwh - load_kwh
    soc_change = (net / 15.12) * 100  # 15.12 kWh capacity
    return current_soc + soc_change - night_drop
```

---

## 6. Alarmové prahy

| Parametr | Warning | Critical | Action |
|----------|---------|----------|--------|
| Cell voltage | <3.0V | <2.8V | Load shedding |
| Cell temp | >45°C | >55°C | Reduce charge |
| Pack temp | <0°C | <-5°C | Stop charging |
| SoC | <20% | <10% | Telegram alert |
| Balance delta | >50 mV | >100 mV | Alert |

---

## 7. Seznam DEBT (nevyřešené blokátory)

| ID | Popis | Stav |
|----|-------|------|
| DEBT_001 | BMS TTL kabel (JST PH 2.0mm) | ❌ čeká na objednávku |
| DEBT_IOT_01 | RAW_FIRST test BMS UART dump | ⏳ nutno po kabelu |
| DEBT_IOT_03 | Binary frame offsety z RAV_FIRST | ⏳ závisí na DEBT_IOT_01 |
| — | Galvanické oddělení RS485 | ⏳ Fáze 3 |

---

*Syntetizováno z Claude_session003/012 (data pipeline), Gemini_session003 (ESP32 kód), Gemini_session085 (RS485), Gemini_session107 (SOC model), Gemini_session110 (RS485 topologie)*