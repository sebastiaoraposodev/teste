# CourtMaster AI Pipeline — Sandbox Documentation

This document is a **hands-on onboarding + operations guide** for the CourtMaster AI Pipeline in **Sandbox**.

## 1. Scope

Covers:
- Sandbox environment only
- End-to-end execution and operational runbook
- Concrete message schemas and queue naming used by the code
- Common troubleshooting and recovery flows

Does not cover:
- Model training or ML internals
- Production SLAs or incident response
- Infrastructure code walkthrough (Terraform)

---

## 2. System Overview

The pipeline is an **event-driven, stateful** video processing system orchestrated by **Apache Airflow**.

### Mental model 

- **Airflow orchestrates.** It schedules tasks, runs operators, and triggers DAGs.
- **MongoDB is the source of truth.** Session/chunk lifecycle and decisions live here.
- **SQS provides events.** Messages define “what should happen next”. 
- **S3 stores artefacts.** Videos, chunk videos, and module outputs.

In practice:
- A “process session” request arrives (SQS).
- Airflow launches a **chunker** (ECS) that writes chunk videos to S3 and publishes **new-chunk** messages.
- Each chunk is ingested + processed (ECS + SageMaker).
- When state indicates readiness, Airflow triggers **session aggregation** (ECS), and optionally overlays.

### Core components (Sandbox)

- **Airflow (CeleryExecutor)** — scheduler/worker execution
- **MongoDB** — pipeline state + metadata
- **LocalStack (SQS only)** — local SQS queues
- **Grafana** — dashboards (MongoDB datasource)
- **Docker Compose** — local environment orchestration
- **AWS (optional, real)** — required for ECS/S3/SageMaker execution when you want to run, requires credetials from organization

---

## 3. Execution Model

### 3.1 Units of work

**Session**
- Identifier: `session_id`
- Session DAG runs use `run_id == session_id`

**Chunk**
- Identifier format: `{session_id}_{chunk_idx:03d}` (e.g., `match_001_003`)
- Chunk DAG runs use `run_id == chunk_id`

### 3.2 Event-driven orchestration

The pipeline is not cron-based.

Triggers used by the DAGs:
- SQS → Airflow Assets via `MessageQueueTrigger`
- DAG-to-DAG via `TriggerDagRunOperator`

### 3.3 State is authoritative (MongoDB)

MongoDB stores:
- sessions
- chunks
- status transitions and execution metadata

Status enums enforced by the domain model:
- Chunk: `pending`, `processing`, `processed`, `failed`
- Session: `waiting_chunks`, `pending`, `processing`, `processed`, `failed`

This enables:
- idempotency and safe retries
- partial reprocessing (chunk-only) without restarting the whole pipeline

---

## 4. End-to-End Flow

### 4.1 Full session flow 

1. An external system publishes a **process-session-full-video** message to SQS:

   - `ai-pipeline-process-session-full-video-{ENVIRONMENT}`

2. The Airflow DAG `session_video_chunker_listener` is triggered via `MessageQueueTrigger`
   (SQS → Airflow Asset watcher).

3. `session_video_chunker_listener` launches an ECS task:

   - `SessionVideoChunkerTaskDefinition`

   This task is responsible for video chunking. The DAG itself does **not** trigger downstream DAGs directly.

4. The chunker (ECS task) performs two actions:
   - Writes chunk videos to S3
   - Publishes one **new-chunk** message per chunk to:
     - `ai-pipeline-new-chunk-{ENVIRONMENT}`

5. The DAG `ingest_chunks_dag` (file: `load_chunk.py`) is triggered via `MessageQueueTrigger`
   on the **new-chunk** queue.

6. `ingest_chunks_dag`:
   - Parses the SQS message (`SQSNewChunkMessage`)
   - Constructs:
     - `chunk_id = f"{session_id}_{chunk_idx}"`
     - (Note: zero-padding is **not** enforced in this layer.)
   - Triggers `process_chunk_dag` with:
     - `run_id = chunk_id`

7. `process_chunk_dag`:
   - Runs ECS task:
     - `ProcessChunkTaskDefinition`
   - Executes a readiness check (`ready_session_checker`)
   - If all required chunks are processed, triggers `process_session_dag` with:
     - `run_id = session_id`

8. `process_session_dag`:
   - Runs ECS tasks:
     - `ProcessSessionTaskDefinition`
     - `ExtraProcessSessionTaskDefinition`
   - Aggregates session-level results
   - Publishes final results to:
     - `ai-pipeline-session-results-{ENVIRONMENT}`

### 4.2 Chunk-only flow 

The chunker can be bypassed.

1. Manually publish a **new-chunk** message to:
   - `ai-pipeline-new-chunk-{ENVIRONMENT}`

2. `ingest_chunks_dag` will:
   - Parse the message
   - Create `chunk_id`
   - Trigger `process_chunk_dag`

3. If needed, trigger `process_session_dag` manually using `session_id` as `run_id`.

---

## 5. DAG Overview

### Primary DAGs

- `session_video_chunker_listener` (file: `session_video_chunker_listener.py`)
  - Trigger: SQS “process session full video” (via Airflow Asset + `MessageQueueTrigger`)
  - Runs: ECS `video-chunker-{ENVIRONMENT}`
  - Publishes: “new-chunk” messages (published by the chunker container; the DAG passes `--sqs-queue`)

- `ingest_chunks_dag` (file: `load_chunk.py`)
  - Trigger: SQS “new-chunk” (via Airflow Asset + `MessageQueueTrigger`)
  - Validates: `SQSNewChunkMessage` schema 
  - Persists: chunk state in MongoDB (`register_new_chunk(...)`)
  - Triggers: `process_chunk` (`TriggerDagRunOperator`, `trigger_run_id = chunk_id`, `reset_dag_run=True`)

- `process_chunk` (file: `process_chunk.py`)
  - Trigger: `TriggerDagRunOperator` (run_id = chunk_id)
  - Runs:
    - ECS: `video-chunker-{ENVIRONMENT}` (tail chunker; runs only when the chunk is NOT the last one)
    - ECS: `court-detector-{ENVIRONMENT}`
    - SageMaker: `player-tracker-{chunk_id}` job
    - SageMaker: `ball-tracker-{chunk_id}` job
  - Triggers: `process_session` when the session is considered ready (state-driven)

- `process_session` (file: `process_session.py`)
  - Trigger: `TriggerDagRunOperator` (run_id = session_id)
  - ECS tasks:
    - `chunk-gluer-{ENVIRONMENT}` (players)
    - `chunk-gluer-{ENVIRONMENT}` (ball)
    - `manual-reid-{ENVIRONMENT}`
    - `impact-detection-{ENVIRONMENT}`
    - `projector-2d-{ENVIRONMENT}`
    - `game-awareness-{ENVIRONMENT}`
    - `shot-mapping-{ENVIRONMENT}`
    - `statistics-processor-{ENVIRONMENT}`
  - Triggers: `extra_process_session`

- `extra_process_session` (file: `extra_process_game.py`)
  - Trigger: manual or `TriggerDagRunOperator`
  - ECS task: `video-overlay` 

### Utility DAGs

- `manual_dag_trigger` (file: `manual_trigger_dag.py`)
  - Manual entrypoint for controlled reprocessing
  - Parameters:
    - `dag_id`: one of `process_chunk`, `process_session`, `extra_process_session`
    - `session_ids`: list of session ids (for session DAGs)
    - `chunk_ids`: list of chunk ids (for chunk DAG)
  - Uses `TriggerDagRunOperator(..., reset_dag_run=True)` to re-run cleanly

- `update_database` (file: `update_database.py`)
  - Manual: seed/update reference data in MongoDB

---

## 6. State & Data

### 6.1 Environment variables 

The DAGs **assert** that these exist:
- `ENVIRONMENT=sandbox`
- `AWS_ACCOUNT_ID=000000000000`

### 6.2 MongoDB (state)

Default sandbox URI:
- `mongodb://mongo:mongo@mongodb:27017`

Access:
```bash
docker exec -it mongodb mongosh -u mongo -p mongo
```

Useful queries (collection names may vary; adjust if needed):
```javascript
use courtmaster

db.sessions.find().sort({_id:-1}).limit(10)
db.sessions.find({session_id:"onboarding_demo"})

db.chunks.find({session_id:"onboarding_demo"}).sort({chunk_idx:1})
db.chunks.find({status:"failed"}).limit(20)
```

### 6.3 S3 (artefacts)

Buckets used by code:
- Data bucket: `courtmaster-ai-data-{ENVIRONMENT}` (e.g., `courtmaster-ai-data-sandbox`)
- Model bucket: `courtmaster-ai`

Common paths:
- Videos:
  - `s3://courtmaster-ai-data-<env>/videos/{session_id}/{session_id}.mp4`
  - `s3://courtmaster-ai-data-<env>/videos/{session_id}/chunks/{chunk_idx}.mp4`
- Models:
  - `s3://courtmaster-ai/models/court-detector/...`
  - `s3://courtmaster-ai/models/player-tracker/...`
- Module outputs:
  - `s3://courtmaster-ai-data-<env>/modules/<module-name>/{session_id}/...`

### 6.4 SQS (events)

#### What the code expects (important)

When `ENVIRONMENT == "sandbox"`, the code uses:
- `SQS_BASE_URL = http://localstack:4566`
- `SQS_AWS_CONN_ID = aws_localstack`

Queue naming in code:
- `ai-pipeline-new-chunk-{ENVIRONMENT}`
- `ai-pipeline-process-session-full-video-{ENVIRONMENT}`
- `ai-pipeline-session-results-{ENVIRONMENT}`

So, in sandbox, the expected queue names are:
- `ai-pipeline-new-chunk-sandbox`
- `ai-pipeline-process-session-full-video-sandbox`
- `ai-pipeline-session-results-sandbox`

#### LocalStack init script note

The orchestrator repo includes `localstack/init-queues.sh` which currently creates:
- `new-chunk-sandbox`
- `process-session-full-video-sandbox`
- `session-results-sandbox`

These do **not** match the `ai-pipeline-*` names used by the DAG code.  
For sandbox runs, ensure the `ai-pipeline-*` queues exist (see commands below).

---

## 7. Sandbox Onboarding (Revised & Verified)

### 7.1 Prerequisites

Before starting:

- Docker Desktop installed
- WSL2 backend enabled (if running linux commands on Windows)
- Docker Desktop running
- Git installed
- Access to:
  - `courtmaster-ai-orchestrator`
  - `dags`

---

### 7.2 Repository Setup

Clone orchestrator:

```bash
git clone https://github.com/<org>/courtmaster-ai-orchestrator.git
cd courtmaster-ai-orchestrator
```

Clone DAGs **in the root of the orchestrator repo**:

```bash
git clone https://github.com/<org>/dags.git
```

This automatically creates the structure required by Docker volume mapping:

Expected:

```
./dags/dags/process_chunk.py
./dags/dags/process_session.py
```

No manual folder creation is required.

---

### 7.3 Environment Variables

#### Linux / WSL

```bash
export ENVIRONMENT=sandbox
export AWS_ACCOUNT_ID=000000000000
```

#### Windows PowerShell

```powershell
$env:ENVIRONMENT="sandbox"
$env:AWS_ACCOUNT_ID="000000000000"
```

These are required by DAG code (`dags/utils/config.py`).

---

### 7.4 Start Services

#### Linux / WSL (Make available)

```bash
make up
```

#### Windows PowerShell (no make)

```powershell
docker compose -f docker-compose.airflow.yaml up -d
docker compose -f docker-compose.localstack.yaml up -d
docker compose -f docker-compose.mongo.yaml up -d
docker compose -f docker-compose.grafana.yaml up -d
```

Verify:

```powershell
docker ps
```

Expected services:

- Airflow UI → http://localhost:8080
- MongoDB → localhost:27017
- LocalStack → localhost:4566
- Grafana → http://localhost:3000

---

### 7.5 Airflow Setup

URL: http://localhost:8080  
Credentials: `airflow / airflow`

Unpause:

- session_video_chunker_listener
- ingest_chunks_dag
- process_chunk
- process_session

---

### 7.6 Configure Connections

Airflow → Admin → Connections

#### aws_localstack

Type: Amazon Web Services  
Access Key: test  
Secret Key: test  

Extra:

```json
{
  "endpoint_url": "http://localstack:4566",
  "region_name": "eu-central-1"
}
```

---

### 7.7 Seed Database

```bash
uv venv --python 3.13
uv run scripts/dbinit.py
```

---

### 7.8 LocalStack SQS Setup

List queues:

```bash
docker exec -it localstack awslocal sqs list-queues
```

Create required queues (matching DAG config):

```bash
docker exec -it localstack awslocal sqs create-queue --queue-name ai-pipeline-new-chunk-sandbox
docker exec -it localstack awslocal sqs create-queue --queue-name ai-pipeline-process-session-full-video-sandbox
docker exec -it localstack awslocal sqs create-queue --queue-name ai-pipeline-session-results-sandbox
```

Verify:

```bash
docker exec -it localstack awslocal sqs list-queues
```

---

## 8. Trigger Examples (Tested)

### 8.1 Trigger Chunk-Only Flow

```bash
docker exec -it localstack awslocal sqs send-message   --queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox   --message-body '{"session_id":"onboarding_demo","chunk_idx":0,"absolute_start_frame_idx":0,"video_uri":"s3://courtmaster-ai-data-sandbox/videos/onboarding_demo/chunks/0.mp4","is_last_chunk":true,"camera_id":"camera_001","camera_model":"DS-2CD2443G2-I","club_id":"club_001","court_id":"court_001","nr_frames":1800,"fps":30.0}'
```

Expected:

- ingest_chunks_dag run appears
- process_chunk triggered with run_id onboarding_demo_000

---

### 8.2 Trigger Full Session Flow

```bash
docker exec -it localstack awslocal sqs send-message   --queue-url http://localhost:4566/000000000000/ai-pipeline-process-session-full-video-sandbox   --message-body '{"session_id":"onboarding_demo","video_uri":"s3://courtmaster-ai-data-sandbox/videos/onboarding_demo/onboarding_demo.mp4","club_id":"club_001","court_id":"court_001","camera_model":"DS-2CD2443G2-I"}'
```

Expected:

- session_video_chunker_listener run appears
- new-chunk messages generated

---

## 9. Where to Observe

### 9.1 Airflow CLI

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list-runs -d ingest_chunks_dag
```

---

### 9.2 Logs

Linux:

```bash
make logs
```

Windows:

```powershell
docker logs -f courtmaster-ai-orchestrator-airflow-scheduler-1
docker logs -f courtmaster-ai-orchestrator-airflow-worker-1
docker logs -f localstack
docker logs -f mongodb
```

---

### 9.3 MongoDB

```bash
docker exec -it mongodb mongosh -u mongo -p mongo
```

```javascript
use courtmaster
db.sessions.find().limit(5)
db.chunks.find().limit(5)
```

---

## 10. Reset & Recovery

### Stop Containers

Linux:

```bash
make down
```

Windows:

```powershell
docker compose down
```

---

### Purge SQS

```bash
docker exec -it localstack awslocal sqs purge-queue   --queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox
```

---

### Reset MongoDB (Sandbox Only)

```bash
docker exec -it mongodb mongosh -u mongo -p mongo --eval "use courtmaster; db.dropDatabase()"
```
