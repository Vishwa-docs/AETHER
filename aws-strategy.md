# AETHER — AWS Strategy & 24-Hour Goal

This document answers the four key evaluation questions for the **AI for Bharath Hackathon** AWS credit application.

---

## 1. Which AWS Services Do We Need?

AETHER is a distributed elderly-care OS spanning edge hardware, a cloud backend, GenAI reasoning, and caregiver-facing apps. The table below maps each AWS service to the AETHER component it enables.

| AWS Service | Role in AETHER |
|---|---|
| **AWS IoT Core** | Device registry, secure MQTT broker for all ESP32 sensor nodes and the AETHER Hub; rules engine fans events into the processing pipeline |
| **AWS API Gateway** | HTTPS entry-point for the Hub's event uplink and all app clients (mobile, web dashboard) |
| **AWS Lambda** | Serverless event processor: validation, normalization, escalation trigger, and GenAI orchestrator invocations |
| **Amazon DynamoDB** | Timeline event store — on-demand capacity, single-digit-ms reads for caregiver dashboards and LLM evidence retrieval |
| **Amazon RDS (PostgreSQL + TimescaleDB extension)** | Structured resident profiles, care plans, longitudinal analytics, and time-series aggregates |
| **Amazon S3** | Data lake for raw evidence packets, exported PDF reports, OTA firmware bundles, and 90-day event archives |
| **Amazon SQS** | Decouples the Lambda ingest layer from downstream consumers; provides backpressure and replay for burst events |
| **Amazon EventBridge** | Event bus routing IoT rules output to correct Lambda handlers; scheduled rules for daily GenAI summary jobs |
| **AWS Bedrock** | Managed inference endpoint for Gemma, Llama-3 Nemotron, and MedGemma (see §2 for model details) |
| **AWS SageMaker** | Federated learning orchestration — aggregates locally trained model updates without exposing raw sensor data |
| **SageMaker Edge Manager** | Deploys updated edge models (fall classifier, activity classifier) to the AETHER Hub over-the-air |
| **Amazon Cognito** | User authentication and fine-grained authorization for seniors, caregivers, nurses, and clinicians |
| **AWS KMS** | Encryption-key management for data at rest (DynamoDB, RDS, S3) and device-cert rotation |
| **Amazon SNS** | SMS and push notification delivery for emergency escalation alerts |
| **AWS CloudWatch** | Logs, metrics, alarms, and dashboards for both cloud services and edge-device health telemetry |
| **AWS WAF** | Web Application Firewall protecting the API Gateway from abuse |
| **Amazon Pinpoint** | Caregiver engagement tracking (alert acknowledgment rates, app session analytics) |
| **Amazon Q (optional)** | Natural-language query interface over AETHER's DynamoDB/RDS data for clinicians and operators |

---

## 2. GenAI Model Strategy — What We Invoke via Bedrock

AETHER uses a **model router** that selects the right model based on latency requirements, risk tier, and data sensitivity. All cloud inference goes through **AWS Bedrock**.

### Models via AWS Bedrock

| Model | Bedrock Model ID | Task in AETHER |
|---|---|---|
| **Llama 3.1 Nemotron (8B)** | `meta.llama3-1-8b-instruct-v1` *(NIM-compatible)* | Primary "Care Coach" — structured JSON reasoning over sensor timelines; generates evidence-backed health summaries with explicit event citations |
| **Gemma 2 (2B / 7B)** | `google.gemma-2-2b-instruct` / `google.gemma-2-7b-it` | Wearable insight summarizer — converts smartwatch vitals and daily activity streams into plain-language caregiver briefs |
| **MedGemma (post-MVP)** | `google.medgemma-*` *(when available on Bedrock)* | Clinical triage advisory — produces urgency-level assessments (Routine / Elevated / Urgent / Emergency) with medical phrasing; explicitly non-diagnostic |

### On-Device (not Bedrock)

| Model | Runtime | Task |
|---|---|---|
| **MediaPipe Pose / BlazePose-style** | ONNX Runtime on edge | Skeleton keypoint extraction; fall pattern detection — video never leaves the device |
| **Lightweight TCN / 1D-CNN** | TensorFlow Lite | Temporal fall classifier over pose vectors |
| **XGBoost / LightGBM** | ONNX Runtime on edge | Camera-free ADL state classifier (sleeping, eating, bathroom) from PIR/mmWave/pressure sensors |

### Model Router Policy

```
risk=HIGH  AND latency=critical  → Edge model (offline-safe)
risk=HIGH  AND latency=normal    → Nemotron 8B via Bedrock
risk=LOW   AND topic=wearables   → Gemma via Bedrock
risk=ANY   AND topic=clinical    → MedGemma via Bedrock (post-MVP)
```

---

## 3. Data Strategy — Sources, Storage, and Processing

### 3.1 Data Sources

| Source | Data Type | Privacy Handling |
|---|---|---|
| **AETHER SenseMesh nodes** (ESP32) | Temperature, humidity, PIR/mmWave presence, door/fridge contact, pressure film | Raw values processed on Hub; only labelled events (type + timestamp + confidence) uploaded |
| **AETHER MedDock** | Compartment open/close timestamps, duration, NFC bottle scans | Event labels only; no video of person |
| **AETHER BedSense** | Pressure strip presence, bed exit timestamps, sleep fragmentation features | Derived features; no audio stored by default |
| **AETHER MealTrack / HydroTrack** | Load-cell weight deltas for plate and water bottle | Event labels (meal started/ended, hydration count) |
| **Edge pose pipeline** | Skeleton keypoints (17–33 landmarks), fall confidence score | Keypoints only; zero raw video frames reach the cloud |
| **Wearable / Apple Health / Google Fit** | HR, SpO2, step count, sleep stages | Imported via vendor export APIs; stored encrypted in RDS |
| **Caregiver WhatsApp-style group thread** | Text messages, photos, voice notes | Passive ingestion pipeline extracts structured events (med change, symptom, appointment); originals stored in S3 with access controls |
| **Synthetic "Digital Household Generator"** | 90 days of realistic routines (meds, meals, sleep, heat events) | Used for pipeline load-testing and LLM prompt evaluation only |

### 3.2 Storage Architecture

```
Edge (Local SQLite)
    │  event-only HTTPS uplink (60 s cadence)
    ▼
AWS IoT Core  ──►  IoT Rules Engine
                        │
                        ├──► Amazon EventBridge  ──►  Lambda (ingest)
                        │                                    │
                        │                          ┌─────────┴──────────┐
                        │                          ▼                    ▼
                        │                   Amazon DynamoDB       Amazon RDS
                        │                  (Timeline events,    (Resident profiles,
                        │                   on-demand reads)     longitudinal stats)
                        │                          │
                        └──────────────────►  Amazon S3
                                              (Evidence packets,
                                               firmware OTA,
                                               90-day archive,
                                               exported PDFs)
```

**Key storage decisions:**

- **DynamoDB** for the hot timeline path: every sensor event is a DynamoDB item (`residentId` + `timestamp` sort key). Queries by caregiver dashboard need < 10 ms.
- **RDS (PostgreSQL + TimescaleDB)** for analytical time-series: rolling adherence windows, sleep trend graphs, anomaly baselines. TimescaleDB continuous aggregates power the clinician dashboard.
- **S3** as the cold/archive tier: lifecycle policy moves items older than 90 days to S3 Glacier; evidence packets (72-hour event bundles) are stored as JSON + PDF in S3.

### 3.3 Data Processing Pipeline

1. **Ingest Lambda** — validates schema, deduplicates, writes to DynamoDB and publishes to SQS.
2. **Enrichment Lambda** (SQS consumer) — resolves resident context from RDS, tags events with care-plan metadata.
3. **Anomaly Detector** (scheduled Lambda, every 15 min) — Isolation Forest over recent event windows; flags deviations from 14-day baseline.
4. **GenAI Orchestrator** (EventBridge scheduled rule, every 24 h) — assembles last-24-hour evidence window from DynamoDB, invokes Nemotron via Bedrock, stores structured JSON insight in RDS, pushes summary to SNS.
5. **Federated Learning aggregator** (SageMaker) — nightly job collects encrypted model deltas from Hub devices, aggregates using federated averaging, publishes updated edge models back via SageMaker Edge Manager.

---

## 4. 24-Hour Goal — First Technical Milestone

> **Goal:** Within 24 hours of AWS credits being activated, have a live, end-to-end data pipeline running in AWS — from a simulated AETHER Hub sending sensor events to a working caregiver dashboard displaying real-time alerts.

### Milestone Checklist

- [ ] **AWS IoT Core** — register one virtual AETHER Hub device; generate and embed X.509 device certificate.
- [ ] **Event simulator** — Python script running on a laptop acting as the Hub; publishes synthetic sensor events (fall, medication, meal, door) over MQTT to IoT Core every 5 seconds.
- [ ] **IoT Rule → Lambda → DynamoDB** — rule fans events to an ingest Lambda function; Lambda writes each event to a DynamoDB `AetherTimeline` table (partition key: `residentId`, sort key: `timestamp`).
- [ ] **API Gateway + Lambda** — single GET `/timeline/{residentId}` endpoint that queries DynamoDB and returns the last 50 events as JSON.
- [ ] **Bedrock smoke-test** — Lambda calls Nemotron 8B via Bedrock with a hard-coded 5-event evidence window and logs the structured JSON insight response to CloudWatch.
- [ ] **React caregiver dashboard** (minimal) — hosted on S3 + CloudFront; polls the `/timeline` API every 10 seconds and renders a live event feed with colour-coded severity.
- [ ] **SNS alert** — when the simulator sends a fall event, Lambda publishes an SMS to a test phone number via Amazon SNS.

### Why This Milestone Matters

This 24-hour build proves the entire **sense → ingest → reason → notify** loop works end-to-end on real AWS infrastructure before any hardware arrives. Every subsequent sprint (real sensors, wearable connectors, federated learning) plugs into this validated backbone rather than being built in isolation.

**Estimated time breakdown:**

| Task | Time |
|---|---|
| IoT Core device + cert setup | 30 min |
| Event simulator script | 45 min |
| Lambda + DynamoDB wiring | 60 min |
| API Gateway endpoint | 30 min |
| Bedrock Nemotron smoke-test | 30 min |
| React dashboard skeleton | 90 min |
| SNS fall alert integration | 30 min |
| End-to-end demo run + screenshots | 30 min |
| **Total** | **~6 hours** |

---

*Document last updated: 2026-02-26. Aligns with [design.md](design.md), [requirements.md](requirements.md), and [Idea.txt](Idea.txt).*
