# CourtMaster AI Pipeline — Sandbox Documentation

This document is an **operations guide** for the CourtMaster AI Pipeline in **Sandbox**.

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
- Logical identifier format (canonical): `{session_id}_{chunk_idx:03d}` (e.g., `match_001_003`)
- In practice, zero-padding is not strictly enforced at ingestion time.
- The actual `run_id` used by `process_chunk` equals the constructed `chunk_id` from the SQS message.


### 3.2 Event-driven orchestration

The pipeline is not cron-based.

Triggers used by the DAGs:
- SQS message updates an Airflow Asset (dataset), which triggers the downstream DAG(s)
- DAG-to-DAG via `TriggerDagRunOperator`

Some DAGs (e.g., `ingest_chunks_dag`) use an Asset-based timetable.

This means:
- They will NOT create runs automatically when Airflow starts.
- They will only run when the associated Asset is updated (via SQS message + MessageQueueTrigger).
- Seeing the DAG in the UI does not imply that it should already have runs.


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
   - Triggers `process_chunk` with:
     - `run_id = chunk_id`

7. `process_chunk_dag`:
   - Runs ECS task:
     - `ProcessChunkTaskDefinition`
   - Executes a readiness check (`ready_session_checker`)
   - If all required chunks are processed, triggers `process_session` with:
     - `run_id = session_id`

8. `process_session`:
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
   - Trigger `process_chunk`

3. If needed, trigger `process_session` manually using `session_id` as `run_id`.

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

- `manual_trigger_dag` (file: `manual_trigger_dag.py`)
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

