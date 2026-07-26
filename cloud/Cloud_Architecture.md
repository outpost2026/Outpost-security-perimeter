# Cloud Architecture — GCP Pipeline

**Verze:** 1.0 | **Datum:** 2026-07-26
**Zdroje:** Gemini_session008, Gemini_session041, Gemini_session047, aktualGCPinfo_14_03.md, Gemini_session002
**Paradigma:** SERVERLESS_ONLY — žádné VM, scale-to-zero

---

## 1. Architektura

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  ESP32 / D1 Mini │────►│  iot-ingest-beta  │────►│  Firestore   │
│  (HTTPS POST)    │     │  Cloud Run        │     │  Native      │
└─────────────────┘     │  europe-west1     │     └──────┬───────┘
                         └──────────────────┘            │
                                                          ▼
┌─────────────────┐     ┌──────────────────┐     ┌──────────────┐
│  Meteo ETL       │────►│  Scheduler jobs   │     │  BigQuery    │
│  (Open-Meteo)    │     │  6× cron          │     │  Analytics   │
└─────────────────┘     └──────────────────┘     └──────────────┘
                               │
                               ▼
                        ┌──────────────┐
                        │  Telegram    │
                        │  Bot API     │
                        │  Notifikace  │
                        └──────────────┘
```

## 2. GCP Stack — aktuální stav

| Service | Název | Status | Účel |
|---------|-------|--------|------|
| **Cloud Run** | `iot-ingest-beta` | ACTIVE | HTTPS ingest z ESP32 |
| **Cloud Run** | `miner-orchestrator` | ACTIVE | ETL orchestrace |
| **Cloud Functions** | `miner-orchestrator` | ACTIVE | Crawler pipeline |
| **Cloud Scheduler** | `pipeline-meteo-hourly` (6× cron) | ENABLED | Periodické úlohy |
| **Cloud Scheduler** | `iot-heartbeat-monitor` | PLANNED | Heartbeat polling |
| **Firestore** | `(default)` | ACTIVE | NoSQL telemetrie |
| **BigQuery** | `outpost_analytics` | ACTIVE | Analytika |
| **Cloud Storage** | `gcp-miner-rag-data-01` | ACTIVE | RAG data |
| **Cloud Storage** | `outpost-material-czwebs` | ACTIVE | Scraped data |

### Enabed APIs

- `cloudfunctions.googleapis.com`
- `cloudscheduler.googleapis.com`
- `compute.googleapis.com`
- `run.googleapis.com`
- `firestore.googleapis.com`
- `pubsub.googleapis.com`
- `bigquery.googleapis.com`
- `storage.googleapis.com`

---

## 3. IAM Model

### Service Account: `iot-outpost-sa`

```yaml
# iot-outpost-sa IAM binding
roles:
  - roles/datastore.user      # Firestore read/write
  - roles/pubsub.publisher    # PubSub pro event-driven processing
  - roles/logging.logWriter   # Cloud Logging

# Zákaz:
# - roles/compute.*           (SERVERLESS_ONLY)
# - roles/iam.*               (zásadní Least Privilege)
```

### Security pravidla

1. **Žádné VM** (`SERVERLESS_ONLY`) — absolutní zákaz nových Compute Engine instancí
2. **IAM Least Privilege** — `iot-outpost-sa` má pouze role nutné pro svou funkci
3. **Public access prevention** — všechny buckety mají `publicAccessPrevention: enforced`
4. **Region lock** — všechno v `europe-west1`

---

## 4. IoT Ingest Pipeline

### ESP32 → Cloud Run

```python
# iot-ingest-beta (Cloud Run, Python 3.11)
from flask import Flask, request, jsonify
from google.cloud import firestore
import datetime

app = Flask(__name__)
db = firestore.Client()

@app.route('/ingest', methods=['POST'])
def ingest():
    data = request.get_json()
    data['timestamp'] = datetime.datetime.utcnow().isoformat()
    data['source'] = data.get('source', 'unknown')

    # Value-aware deduplication
    if is_duplicate(data):
        return jsonify({'status': 'duplicate', 'id': data['id']}), 200

    doc_ref = db.collection('telemetry').document(data['id'])
    doc_ref.set(data)
    return jsonify({'status': 'ok', 'id': data['id']}), 201

def is_duplicate(data):
    """Prevence duplicit pri opakovanem odeslani."""
    existing = db.collection('telemetry').document(data['id']).get()
    return existing.exists
```

### Heartbeat monitoring

```yaml
# Cloud Scheduler: iot-heartbeat-monitor
schedule: "*/5 * * * *"       # kazdych 5 minut
target: https POST
uri: https://europe-west1-...cloudfunctions.net/iot-heartbeat-check
```

Pokud heartbeat chybí >15 min → Telegram alert. Toto je implementace **Dead Man's Switch** — alarm nevyvolává zpráva, ale její absence.

---

## 5. Dead Man's Switch (inverzní security)

```python
# iot-heartbeat-check (Cloud Function)
from google.cloud import firestore
import datetime

TIMEOUT_MINUTES = 15

def check_heartbeat(request):
    db = firestore.Client()
    now = datetime.datetime.utcnow()

    nodes = db.collection('heartbeat').stream()
    alerts = []
    for node in nodes:
        data = node.to_dict()
        last = data.get('last_seen')
        if last:
            delta = (now - last).total_seconds() / 60
            if delta > TIMEOUT_MINUTES:
                alerts.append(f"⚠️ {node.id}: heartbeat chybi {delta:.0f} min")

    if alerts:
        send_telegram("\n".join(alerts))
    return "OK"
```

### Princip

| Stav | Výsledek |
|------|----------|
| ESP32 odesílá heartbeat každých 5 min | ✅ Klid |
| ESP32 silent >15 min | 🚨 Telegram alert: "Node X offline" |
| GSM rušička aktivní | 🚨 Heartbeat nedorazí → alert |
| Výpadek LTE | 🚨 Alert (ale false positive s LTE) |

---

## 6. Telegram Notifikace

### Schéma

```
ESP32 ──fault──> Cloud Run ──POST──> https://api.telegram.org/bot<TOKEN>/sendMessage
```

### Alarmové kanály

| Level | Kanál | Latence |
|-------|-------|---------|
| INFO | Telegram kanál #info | 5 min |
| WARNING | Telegram kanál #alerts | <30 s |
| CRITICAL | Telegram přímá zpráva | <10 s |
| SHUTDOWN | Telegram + SMS (GSM) | <5 s |

---

## 7. Nákladová optimalizace

| Service | Režim | Měsíční náklad |
|---------|-------|---------------|
| Cloud Run | Scale-to-zero | ~0 Kč (v rámci free tier) |
| Cloud Functions gen2 | Scale-to-zero | ~0 Kč |
| Cloud Scheduler | 6× job, minuta | ~0 Kč |
| Firestore | Free tier (1 GB) | ~0 Kč |
| BigQuery | 10 TB/měsíc free | ~0 Kč |
| **Celkem** | | **~0 Kč/měsíc** |

---

*Syntetizováno z Gemini_session008 (serverless architektura), Gemini_session041 (ETL pipeline), Gemini_session047 (Dead Man's Switch), aktualGCPinfo (GCP snimek)*