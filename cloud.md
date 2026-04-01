\# GCP Stack Ingest v3 – Serverless Infrastructure



This document describes the Google Cloud Platform (GCP) resources used by the Outpost project. The stack is designed for \*\*zero idle cost\*\* (no always‑running VMs) and is fully managed via serverless services.



\## Overview



| Service | Purpose |

|---------|---------|

| \*\*Cloud Scheduler\*\* | Cron‑like triggers for periodic jobs (6 schedules). |

| \*\*Cloud Run\*\* | Serverless container execution for orchestrator, scrapers, and pipelines. |

| \*\*Firestore\*\* | Primary NoSQL database for IoT telemetry, events, and configuration. |

| \*\*BigQuery\*\* | Data warehouse for long‑term analytics and reporting. |

| \*\*Cloud Storage (GCS)\*\* | Archival storage for exported data and backups. |

| \*\*Cloud Functions\*\* | Lightweight event‑driven functions (used for specific integrations). |

| \*\*IAM\*\* | Fine‑grained access control. |



\## Components



\### 1. Cloud Scheduler Jobs

\- \*\*`scraper-bazos`\*\* – Runs daily at 06:00, triggers Cloud Run `scraper-bazos` to fetch new listings.

\- \*\*`scraper-notebooky`\*\* – Runs daily at 07:00, scrapes notebook offers.

\- \*\*`meteo-pipeline`\*\* – Runs every 30 minutes, fetches data from ČHMÚ Kbely and writes to Firestore.

\- \*\*`orchestrator-daily`\*\* – Daily at 23:00, runs cleanup, backup, and summary tasks.

\- \*\*`iot-ingest`\*\* – (Reserved for future ESP32 POST ingestion) – runs every 5 minutes to process queued events.

\- \*\*`health-check`\*\* – Every hour, verifies that all services are responsive.



\### 2. Cloud Run Services

All services are containerised (Docker) and scale to zero when idle.



| Service | Description | Runtime |

|---------|-------------|---------|

| `orchestrator` | Main entry point for scheduled jobs; coordinates sub‑tasks. | Python 3.11 |

| `scraper-bazos` | Fetches and parses Bazoš listings; writes to Firestore. | Python 3.11 |

| `scraper-notebooky` | Similarly for notebooky.cz. | Python 3.11 |

| `meteo-ingest` | Polls ČHMÚ API, transforms, stores in Firestore. | Python 3.11 |

| `iot-ingestor` | Receives HTTP POST from ESP32; validates and stores events. | Python 3.11 + FastAPI |

| `export-bigquery` | Daily export of Firestore collections to BigQuery. | Python 3.11 |



\### 3. Data Model (Firestore)



Collections:



\- `telemetry/` – IoT sensor readings (time‑series).

\- `events/` – Security events (trigger source, weight, photo path).

\- `scraped/` – Raw scraped data from external sites.

\- `weather/` – Hourly weather data from ČHMÚ.

\- `config/` – System configuration (thresholds, flags).



\### 4. BigQuery Dataset

\- `outpost\_analytics`

&#x20; - Tables: `telemetry`, `events`, `weather`, `scraped`

&#x20; - Partitioned by ingestion date.

&#x20; - Used for dashboards and anomaly detection.



\### 5. Cloud Storage Buckets

\- `outpost-backups` – Daily exports of Firestore and BigQuery.

\- `outpost-images` – Reserved for security camera images (future).



\## Deployment \& Management

\- Infrastructure is defined manually via GCP Console (no Terraform yet, planned for Iteration 4).

\- All Cloud Run services use \*\*`--cpu 1 --memory 256Mi`\*\* and are configured to \*\*`--min-instances 0`\*\*.

\- Scheduler jobs are authenticated using OIDC to invoke Cloud Run.



\## Cost Analysis

\- Cloud Scheduler: 6 jobs × 30 days = 180 invocations (free tier covers 3 jobs, remaining \~$0.30/month).

\- Cloud Run: Each invocation < 1 minute, scaled to zero → typical monthly cost < $1.

\- Firestore: Small document count (<10k) stays within free tier.

\- BigQuery: Queries and storage within free tier (10 GB, 1 TB queries).

\- \*\*Total monthly cost: < $1 (practically $0).\*\*



\## Future Integrations

\- \*\*Make / n8n\*\* – Connect to Firestore via webhook to trigger Telegram/email alerts.

\- \*\*Gemini API\*\* – Image classification of security photos.

\- \*\*Webhook endpoint\*\* for ESP32 to send events directly to `iot-ingestor`.



\*Snapshot date: 2026-04-01\*

