# 🔊 Acoustic Guard — Infrastructure Hub

> Complete Docker Compose infrastructure for the **Acoustic Guard** distributed urban acoustic monitoring system — provisioning core backend services, ML classifiers, messaging brokers, and edge agents for both simulation and live testing.

---

## 📐 Architecture Overview

The infrastructure is divided into two logical blocks: **Core Services** and the **Edge Network**.

### Core Services

| Service | Stack | Role |
|---|---|---|
| `rabbitmq` | RabbitMQ 3 | AMQP message broker for high-throughput audio event streams |
| `postgres` | PostgreSQL 15 + PostGIS | Spatial database for alerts, incidents, sensors, telemetry |
| `classifier` | Python + YAMNet + Random Forest | gRPC server with dual ML pipeline for threat classification |
| `java-hub` | Java + Spring Boot | Central orchestrator — REST API, WebSocket broadcaster, DB persistence |

**Classified threat types:** UAV · Explosion · Siren · Truck · Generator

---

## 🚀 Modes of Operation

The system supports two distinct ways to run the infrastructure.

---

### 🎮 Mode 1 — Full Simulation *(default, out-of-the-box)*

Spins up the entire backend along with **13 virtual edge agents** streaming pre-recorded anomalies (explosions, sirens, UAVs) across different geographic zones.

**1. Start the cluster**

```bash
docker compose up -d --build
```

**2. Open the frontend**

```
http://localhost:5173
```

**3. Log in with the default admin credentials**

```
Username: admin
Password: admin123
```

**4. Observe** — the dashboard populates with live simulated incidents, telemetry, and the noise heatmap in real time.

---

### 🎙️ Mode 2 — Live Microphone Testing

Use this mode to test the ML classifier and backend using a **physical microphone** through a local Rust edge agent.

#### Step 1 — Windows Audio Setup *(crucial for accurate classification)*

To prevent OS-level audio processing from distorting ML input:

1. Open **Sound Control Panel → Recording → Microphone Properties**
2. **Enhancements tab** → check *"Disable all sound effects"* (disable any AI noise cancellation/suppression)
3. **Advanced tab** → set Default Format to a clean sample rate without spatial audio

#### Step 2 — Start Core Infrastructure

This starts the Database, RabbitMQ, Classifier, and Hub — **without** the 13 simulated agents, so your microphone is the only data source:

```bash
docker compose -f docker-compose.live.yml up -d --build
```

#### Step 3 — Start the Local Edge Agent

```bash
cd ../edge-agent
cargo run
```

> Ensure the edge agent is configured to use `AUDIO_SOURCE=MIC` rather than `FILE` mode.

#### Step 4 — Test in Real Time

Log into the frontend (`admin` / `admin123`), play a siren or generator sound from your phone near the microphone, and watch the system detect it live.

---

## 🛠 Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Docker | v24.0.0+ | Required for all modes |
| Docker Compose | v2.20.0+ | Required for all modes |
| Rust / Cargo | latest stable | **Mode 2 only** |
| Free RAM | 4–8 GB | 17 containers run concurrently in Mode 1 |

---

## 🔌 Exposed Ports

| Service | Port | Description | Credentials |
|---|---|---|---|
| Java Hub (REST + WS) | `8080` | Main API & WebSocket endpoint | — |
| PostgreSQL | `5432` | Direct database access | `postgres` / `password` |
| RabbitMQ AMQP | `5672` | AMQP port for edge agents | `guest` / `guest` |
| RabbitMQ UI | `15672` | Management web interface | `guest` / `guest` |
| gRPC Classifier | `50051` | Direct ML inference endpoint | — |

---

## 🧹 Maintenance & Cleanup

Stop all services gracefully:

```bash
docker compose down
```

Stop and **wipe all data** (destroys persistent volumes — database, logs, RabbitMQ queues):

```bash
docker compose down -v
```
