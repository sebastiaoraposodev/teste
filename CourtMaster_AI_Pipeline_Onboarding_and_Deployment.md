# CourtMaster AI Pipeline — Sandbox Documentation (Onboarding and Deploy)

## 1. Sandbox Onboarding 

### 1.1 Prerequisites

Before starting:

- Docker Desktop installed
- WSL2 backend enabled (if running linux commands on Windows)
- Docker Desktop running
- Git installed
- Access to:
  - `courtmaster-ai-orchestrator`
  - `dags`
  - `courtmaster-ai-infra`

---

### 1.2 Repository Setup

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

### 1.3 Environment Variables

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

### 1.4 Start Services

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

### 1.5 Airflow Setup

URL: http://localhost:8080  
Credentials: `airflow / airflow`

Ensure the following DAGs are **not paused**:

- session_video_chunker_listener
- ingest_chunks_dag
- process_chunk
- process_session

You can verify via CLI:

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list --output plain
```

Note:
Even if unpaused, Asset-triggered DAGs will not run until a valid SQS message updates their Asset.

---

### 1.6 Configure Connections (UI + CLI) — REQUIRED

 Source of truth: DAG code.
- In sandbox, SQS sensors/triggers use `aws_localstack` (from `dags/utils/config.py`).
- HOWEVER: some tasks hardcode `aws_default` (e.g., results publishing + SageMaker operators).

 Therefore, for the sandbox onboarding to be correct and reproducible, you must configure BOTH:
- `aws_localstack`
- `aws_default`

#### 1.6.1 UI Method 

Airflow UI → Admin → Connections

Create / verify:

A) Connection: aws_localstack
- Conn Id: `aws_localstack`
- Conn Type: Amazon Web Services
- Login: `dummy`
- Password: `dummy`
- Extra: **MUST** include LocalStack SQS endpoint to ensure hooks/operators can reach local SQS:
  ```json
  {
    "endpoint_url": "http://localstack:4566",
    "region_name": "eu-central-1"
  }
  ```

B) Connection: aws_default

- Conn Id: aws_default

- Conn Type: Amazon Web Services

- Login/Password:

- For sandbox-only trigger testing: can be dummy.

- For end-to-end ECS/SageMaker execution: must be real org credentials/role access.

Extra:
```json
{
  "region_name": "eu-central-1"
}
```

#### 1.6.2 CLI Verification

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow connections get aws_localstack
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow connections get aws_default
```

#### 1.6.3 CLI Creation (fallback)

A) Connection: aws_localstack

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow connections add aws_localstack \
  --conn-type aws \
  --conn-login dummy \
  --conn-password dummy \
  --conn-extra '{"endpoint_url":"http://localstack:4566","region_name":"eu-central-1"}'
```  

B) Connection: aws_default
```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow connections add aws_default \
  --conn-type aws \
  --conn-login dummy \
  --conn-password dummy \
  --conn-extra '{"region_name":"eu-central-1"}'
 ```

### 1.7 UI Version & Service Health (UI + CLI)

#### Airflow Version (CLI)

```bash
curl http://localhost:8080/api/v2/version
```

#### Airflow UI Health Checklist

Airflow UI → http://localhost:8080

Credentials : (airflow / airflow)

Browse → DAGs

Confirm DAGs appear (no missing DAG folder mapping)

Browse → Import Errors

Confirm empty. If not empty, open each error and fix before onboarding continues.

Browse → Runs / DAG Runs

Confirm scheduler is active (new runs appear when triggered).

#### Grafana Version

CLI:

```bash
docker exec -it grafana grafana-server -v
```

UI:

Grafana → http://localhost:3000

Confirm you can log in and navigate to “Connections / Data Sources”.

#### LocalStack SQS Health (CLI)

```bash
docker exec -it localstack awslocal sqs list-queues
```

---

### 1.8 Seed Database

If `uv` is installed:

```bash
uv venv --python 3.13
uv run scripts/dbinit.py
```

You can skip this step in sandbox unless explicitly testing database seeding,
or execute the script using a standard Python environment if required.


---

### 1.9 LocalStack SQS Setup

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

#### 1.9.1 List queue URLs 

```bash
docker exec -it localstack awslocal sqs get-queue-url --queue-name ai-pipeline-new-chunk-sandbox
docker exec -it localstack awslocal sqs get-queue-url --queue-name ai-pipeline-process-session-full-video-sandbox
docker exec -it localstack awslocal sqs get-queue-url --queue-name ai-pipeline-session-results-sandbox
```

#### 1.9.2 Peek messages (receive-message)

```bash
docker exec -it localstack awslocal sqs receive-message \
  --queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox \
  --max-number-of-messages 5 \
  --wait-time-seconds 1
```

#### 1.9.3 Purge queues (reset between tests)

```bash
docker exec -it localstack awslocal sqs purge-queue \
  --queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox
```

---

## 2. Trigger Examples 

### 2.1 Test 1 — Trigger Chunk-Only via SQS (ingest_chunks_dag → process_chunk)

#### CLI 

```bash
docker exec -it localstack awslocal sqs send-message \
  --queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox \
  --message-body '{"session_id":"onboarding_demo","chunk_idx":0,"absolute_start_frame_idx":0,"video_uri":"s3://courtmaster-ai-data-sandbox/videos/onboarding_demo/chunks/0.mp4","is_last_chunk":true,"camera_id":"camera_001","camera_model":"DS-2CD2443G2-I","club_id":"club_001","court_id":"court_001","nr_frames":1800,"fps":30.0}'
```

#### UI Validation

Airflow UI → DAGs → ingest_chunks_dag

Open “Runs” → confirm new run appears after message is sent.

Graph view → open logs of:

fetch_messages

register_chunk

trigger_process_chunk

Then validate downstream:
Airflow UI → DAGs → process_chunk

Confirm a run exists with run_id = {session_id}_{chunk_idx}

#### CLI Validation

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list-runs ingest_chunks_dag
```
```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list-runs process_chunk
```

---

### 2.2 Test 2 — Trigger Full Session via SQS (session_video_chunker_listener)

#### CLI 

```bash
docker exec -it localstack awslocal sqs send-message \
  --queue-url http://localhost:4566/000000000000/ai-pipeline-process-session-full-video-sandbox \
  --message-body '{"session_id":"onboarding_demo","video_uri":"s3://courtmaster-ai-data-sandbox/videos/onboarding_demo/onboarding_demo.mp4","club_id":"club_001","court_id":"court_001","camera_model":"DS-2CD2443G2-I"}'
```

#### UI Validation

Airflow UI → DAGs → session_video_chunker_listener

  Confirm run appears

  Graph view → open logs for run_video_chunker

  Confirm operator tried to launch:

  ECS task definition: video-chunker-sandbox

  command includes: --sqs-queue ai-pipeline-new-chunk-sandbox

#### Note : If No DAG Run Appears After Sending SQS Message

If you send a message and nothing appears in Airflow:

1) Verify the queues exist:

```bash
docker exec -it localstack awslocal sqs list-queues
```

Ensure the queue names are:

ai-pipeline-new-chunk-sandbox

ai-pipeline-process-session-full-video-sandbox

2) Verify Airflow connection:

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow connections get aws_localstack
```

3) Verify DAG is not paused:

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list --output plain
```

4) Check scheduler logs:

```bash
docker logs -f courtmaster-ai-orchestrator-airflow-scheduler-1
```

### 2.3 Test 3 — Trigger process_chunk manually 

This validates the DAG structure/logging even when you cannot run ECS/SageMaker.

#### UI Trigger

Airflow UI → DAGs → process_chunk → Trigger DAG

  Choose run_id (example): onboarding_demo_0

Then validate in UI:

  Graph view → open logs per task

  Identify which tasks require AWS resources (ECS, SageMaker)

### 2.4 Test 4 — Trigger process_session manually (UI + CLI)

#### UI Trigger

Airflow UI → DAGs → process_session → Trigger DAG

  run_id = onboarding_demo

#### CLI Trigger

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags trigger process_session --run-id onboarding_demo
```

### 2.5 Test 5 — Validate results queue (process_session output)

process_session publishes final results to:

ai-pipeline-session-results-sandbox

#### CLI: receive results message

```bash
docker exec -it localstack awslocal sqs receive-message \
  --queue-url http://localhost:4566/000000000000/ai-pipeline-session-results-sandbox \
  --max-number-of-messages 10 \
  --wait-time-seconds 1
```

#### UI: validate publish task logs

Airflow UI → process_session → open run → Graph

  Click the final task send_session_results_task

  Logs must show:

    QueueUrl = ai-pipeline-session-results-sandbox

    “results sent to SQS queue …”

Important: this task uses aws_default connection (hardcoded). If aws_default is missing, it fails.

### 2.6 Test 6 — Trigger extra_process_session manually

#### UI Trigger

UI Trigger

Airflow UI → DAGs → extra_process_session → Trigger DAG

  run_id = onboarding_demo

  configuration must include:

  ```json
  {"session_id":"onboarding_demo"}
  ```

#### CLI Trigger

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags trigger extra_process_session \
  --run-id onboarding_demo \
  -c '{"session_id":"onboarding_demo"}'
```

### 2.7 Test 7 — manual_trigger_dag 

This DAG is a UI-friendly entrypoint for reprocessing.^

#### UI Trigger

Airflow UI → DAGs → manual_trigger_dag → Trigger DAG

  Fill params:

    dag_id: process_chunk OR process_session OR extra_process_session

    chunk_ids (array) or session_ids (array)

  Examples:

    Trigger chunk:

      ```json
      {"dag_id":"process_chunk","chunk_ids":["onboarding_demo_0","onboarding_demo_1"]}
      ```

    Trigger session:

      ```json
      {"dag_id":"process_session","session_ids":["onboarding_demo"]}
      ```

    Trigger extra:

    ```json
    {"dag_id":"extra_process_session","session_ids":["onboarding_demo"]}
    ```

#### CLI Trigger

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags trigger manual_trigger_dag \
  -c '{"dag_id":"process_chunk","chunk_ids":["onboarding_demo_0"]}'
```

### 2.8 Test 8 — update_database

This DAG seeds/updates MongoDB reference data.

#### UI Trigger

Airflow UI → DAGs → update_database → Trigger DAG

#### CLI Trigger

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags trigger update_database --run-id db_seed_test
```

Validate:

  Airflow logs for the DAG run

  MongoDB queries 

---

## 3. Where to Observe

### 3.1 Airflow UI (Primary)

Airflow UI → http://localhost:8080

#### 3.1.1 Import Errors
Browse → Import Errors  
- Must be empty for onboarding to be considered successful.

#### 3.1.2 DAG Run Forensics
For any DAG:
1) Open DAG → Runs
2) Open the run → Graph
3) Click task → Logs

Checklist inside Logs:
- Which connection id is used (`aws_localstack` vs `aws_default`)
- Which queue URL is used (must match `ai-pipeline-*`)
- For ECS tasks: task definition name and errors
- For SageMaker tasks: job name and AWS errors

#### 3.1.3 Pause/Unpause and Trigger
- Unpause DAGs: toggle in DAG list
- Trigger manually: “Trigger DAG” button

---

### 3.2 Airflow CLI 

List DAGs:
```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list --output plain
```

List runs:

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags list-runs ingest_chunks_dag
```

Trigger DAG:

```bash
docker exec -it courtmaster-ai-orchestrator-airflow-apiserver-1 airflow dags trigger process_session --run-id onboarding_demo
```

---

### 3.3 LocalStack (SQS) — CLI only

#### List queues:

```bash
docker exec -it localstack awslocal sqs list-queues
```

#### Receive messages:

```bash
docker exec -it localstack awslocal sqs receive-message \
  --queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox \
  --max-number-of-messages 5 \
  --wait-time-seconds 1
```

---

### 3.4 Logs

#### Linux:

```bash
make logs
```

#### Windows:

```bash
docker logs -f courtmaster-ai-orchestrator-airflow-scheduler-1
docker logs -f courtmaster-ai-orchestrator-airflow-worker-1
docker logs -f localstack
docker logs -f mongodb
```

---

### 3.5 MongoDB (State) — UI via Grafana OR CLI via mongosh

#### CLI (mongosh)

```bash
docker exec -it mongodb mongosh -u mongo -p mongo
```

#### Core queries:

use courtmaster
db.sessions.find().sort({_id:-1}).limit(10)
db.chunks.find({session_id:"onboarding_demo"}).sort({chunk_idx:1})
db.chunks.find({status:"failed"}).limit(20)

---

### 3.6 Grafana UI (Mongo datasource) — Visualization

Grafana UI → http://localhost:3000

#### Add datasource:

Type: MongoDB datasource plugin (installed by container)

Host: mongodb:27017

User/Pass: mongo / mongo

---

## 4. Reset & Recovery

### Stop/Remove Containers

Linux:

```bash
make down
```

Windows:

```bash
docker compose -f docker-compose.airflow.yaml -f docker-compose.localstack.yaml -f docker-compose.mongo.yaml -f docker-compose.grafana.yaml down
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


# 5. Deployment and Pipeline Execution

## 5.1 Deployment Process

### 5.1.1 Build and Push Image to ECR

The deployment process starts by opening a Pull Request to the `main` branch of the service repository (e.g., `courtmaster-ai-statistics-processor`).

When the PR is opened:

1. GitHub Actions runs automatically.
2. The workflow builds the Docker image.
3. The image is pushed to Amazon ECR.
4. A version/tag identifier is generated.

#### Retrieve the Image Version (GitHub UI)

1. Open the service repository in GitHub.
2. Navigate to **Actions**.
3. Open the workflow run associated with the PR.
4. Locate the build and push step.
5. Copy the generated image tag/version.

#### Verify the Image in ECR (CLI)

```bash
aws ecr describe-images   --repository-name courtmaster-ai-video-chunker   --image-ids imageTag=<VERSION>
```

---

### 5.1.2 Update Infrastructure (Terraform)

Navigate to the infrastructure repository:

```
terraform/environments/sandbox/pipeline
```

Open:

```
locals.tf
```

Add the new image version to the appropriate service block (e.g., `statistics_processor`):

```hcl
versions = {
  "1.0.0" = {
    cpu    = 512
    memory = 1024
  }

  "<NEW_VERSION_FROM_GITHUB_ACTIONS>" = {
    cpu    = 512
    memory = 1024
  }
}
```

---

### 5.1.3 Apply Terraform Changes

Linux / WSL:
```bash
cd terraform/environments/sandbox/pipeline
terraform init
terraform apply
```

Confirm the changes when prompted.

After successful apply, commit and merge the changes to the `main` branch of the infrastructure repository.

---

### 5.1.4 Update Orchestrator / DAG Version

After infrastructure is updated, update the version used by the orchestrator.

Navigate to the orchestrator repository and update the version reference inside:

```
dags/dags/utils/config.py
```

Ensure the version used by the DAGs matches the version deployed via Terraform so that ECS task definitions resolve correctly.

---

## 5.2 Running the Pipeline via Video Chunker

The recommended method to run the pipeline is using the Video Chunker tool.

### 5.2.1 Command (Linux / WSL)

```bash
uv run video-chunker chunker 26226_first_20min.mp4   --chunk-duration-seconds 600   --aws-sync   --bucket courtmaster-ai-data-sandbox   --session-id test_manual_reid_26226_3   --club-id fakeclub   --court-id fakecourt   --camera-id fakeid   --camera-model DS-2CD2443G2-I   --message-sleep-time 300   --sqs-queue-url http://localhost:4566/000000000000/ai-pipeline-new-chunk-sandbox
```

This command:

1. Splits the video into chunks.
2. Uploads chunks to S3 when `--aws-sync` is enabled.
3. Sends SQS messages to `ai-pipeline-new-chunk-sandbox`.
4. Triggers the Airflow pipeline automatically via `ingest_chunks_dag`.
