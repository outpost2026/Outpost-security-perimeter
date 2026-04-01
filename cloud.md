# GCP Stack Ingest v3 – Serverless Infrastructure

This document describes the Google Cloud Platform (GCP) resources used by the **Outpost project**. The stack is designed for **zero idle cost** (no always‑running VMs) and is fully managed via serverless services.

> [!TIP]
> **Cost Efficiency:** By utilizing Cloud Run's scale-to-zero capability and staying within free tiers for Firestore and BigQuery, the total monthly cost is kept near $0.

## 1. Overview of Services

| Service | Purpose |
| :--- | :--- |
| **Cloud Scheduler** | Cron-like triggers for periodic jobs (6 schedules). |
| **Cloud Run** | Serverless container execution for orchestrator, scrapers, and pipelines. |
| **Firestore** | Primary NoSQL database for IoT telemetry, events, and configuration. |
| **BigQuery** | Data warehouse for long-term analytics and reporting. |
| **Cloud Storage** | Archival storage for exported data and backups. |
| **Cloud Functions** | Lightweight event-driven functions for specific integrations. |
| **IAM** | Fine-grained access control across the stack. |

---

## 2. Components

### 2.1. Cloud Scheduler Jobs
* **`scraper-bazos`**: Runs daily at 06:00; triggers scraper for new listings.
* **`scraper-notebooky`**: Runs daily at 07:00; scrapes notebook offers.
* **`meteo-pipeline`**: Runs every 30 minutes; fetches data from ČHMÚ Kbely and writes to Firestore.
* **`orchestrator-daily`**: Daily at 23:00; executes cleanup, backup, and summary tasks.
* **`iot-ingest`**: Runs every 5 minutes to process queued IoT events.
* **`health-check`**: Runs hourly to verify all services are responsive.

### 2.2. Cloud Run Services
All services are containerized (Docker) and configured to scale to zero when idle.

| Service | Description | Runtime |
| :--- | :--- | :--- |
| `orchestrator` | Main entry point; coordinates sub-tasks. | Python 3.11 |
| `scraper-bazos` | Fetches/parses Bazoš listings; writes to Firestore. | Python 3.11 |
| `scraper-notebooky` | Similarly for notebooky.cz. | Python 3.11 |
| `meteo-ingest` | Polls ČHMÚ API; transforms and stores data. | Python 3.11 |
| `iot-ingestor` | Receives HTTP POST from ESP32; uses FastAPI. | Python 3.11 |
| `export-bigquery` | Daily export of Firestore collections to BigQuery. | Python 3.11 |

---

## 3. Data Architecture

### 3.1. Firestore Collections (NoSQL)
* `telemetry/`: IoT sensor readings (time-series).
* `events/`: Security events (trigger source, weight, photo path).
* `scraped/`: Raw scraped data from external sites.
* `weather/`: Hourly weather data from ČHMÚ.
* `config/`: System configuration (thresholds, flags).

### 3.2. BigQuery Dataset (`outpost_analytics`)
* **Tables**: `telemetry`, `events`, `weather`, `scraped`.
* **Partitioning**: Data is partitioned by ingestion date for query efficiency.
* **Usage**: Powering dashboards and anomaly detection algorithms.

### 3.3. Cloud Storage Buckets
* `outpost-backups`: Daily exports of Firestore and BigQuery.
* `outpost-images`: Reserved for future security camera image storage.

---

## 4. Deployment & Management

* **Current State**: Infrastructure is defined manually via GCP Console.
* **Future Plan**: Migration to Terraform for IaC (planned for Iteration 4).
* **Cloud Run Specs**: Services use `--cpu 1 --memory 256Mi` with `--min-instances 0`.
* **Security**: Scheduler jobs use OIDC authentication to invoke Cloud Run endpoints.

---

## 5. Cost Analysis

| Component | Usage Estimate | Monthly Cost |
| :--- | :--- | :--- |
| **Cloud Scheduler** | 180 invocations (6 jobs × 30 days) | ~$0.30 |
| **Cloud Run** | Scaled to zero; <1 min per invocation | < $1.00 |
| **Firestore** | < 10k documents (within free tier) | $0.00 |
| **BigQuery** | Within 10 GB storage / 1 TB query limit | $0.00 |
| **Total** | | **<$1.00 (Practical $0)** |

---

## 6. Future Integrations

* **Make / n8n**: Webhook connection to Firestore for Telegram/email alerts.
* **Gemini API**: AI-based image classification of security camera photos.
* **ESP32 Direct Webhook**: Streamlined event ingestion for IoT devices.

> [!NOTE]
> **Snapshot Date:** 2026-04-01
