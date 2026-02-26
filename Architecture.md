# Overall System Architecture — Spectra Sight Service Backend

**Platform**: AI-Powered Video Analysis Platform
**Version**: Production
**Architecture Style**: Event-Driven Microservices

---

## 1. System Overview

Spectra Sight is an **AI-powered video intelligence platform** that ingests video from two sources — pre-recorded batch files and live camera streams — and applies Vision Language Models (VLMs) to extract structured insights, events, and semantic search capabilities from the video content.

The system is composed of **five microservices** connected through an asynchronous message queue (RabbitMQ), with shared infrastructure for storage, databases, vector search, and AI inference.

### Core Value Proposition

| Raw Input | Processed Output |
|-----------|-----------------|
| MP4 video file or live camera stream | Natural-language event descriptions |
| Hours of continuous footage | Searchable, timestamped event database |
| Unstructured visual data | Structured JSON events (person, vehicle, anomaly) |
| Siloed camera recordings | Semantic similarity search across all cameras |

---

## 2. Service Inventory

| Service | Name | Port | Role |
|---------|------|------|------|
| **Backend** | `backend` | 8080/8083 | GraphQL API, orchestration, SSE notifications |
| **M0** | `sightapi_m0` | 9000 | Video preprocessing and chunking |
| **M1** | `sightapi_m1` | 7081 | VLM inference — visual understanding |
| **M2** | `sightapi_m2` | 7082 | Event detection, embedding, vector indexing |
| **S2** | `sightapi_s2` | 7084 | Live camera stream processing via AWS KVS |

---

## 3. Infrastructure Components

| Component | Version | Purpose |
|-----------|---------|---------|
| **PostgreSQL** | 17.6 + pgvector | Primary relational database (3 schemas) |
| **Redis Stack** | Latest | Caching, session management, camera discovery |
| **RabbitMQ** | 3.x | Async message queue for service communication |
| **Qdrant** | Latest | Vector database for semantic similarity search |
| **AWS S3 / Azure Blob** | — | Object storage for videos, chunks, summaries |
| **AWS KVS** | — | Live camera stream ingestion |
| **Langfuse** | — | LLM observability and prompt management |

---

## 4. Full System Architecture Flow Diagram

```
                        ┌────────────────────────────────┐
                        │         CLIENT APPLICATIONS     │
                        │   (Web UI / Mobile / Admin)     │
                        └──────────────┬─────────────────┘
                                       │ HTTPS
                                       ▼
                        ┌─────────────────────────────────┐
                        │      BACKEND SERVICE (8080)      │
                        │                                  │
                        │  ┌──────────┐  ┌─────────────┐  │
                        │  │ GraphQL  │  │  REST APIs   │  │
                        │  │ (Strawb.)│  │  (FastAPI)   │  │
                        │  └──────────┘  └─────────────┘  │
                        │  ┌──────────────────────────┐   │
                        │  │  SSE Notification Manager │   │
                        │  └──────────────────────────┘   │
                        │  ┌──────────────────────────┐   │
                        │  │   LangGraph Chat Agent    │   │
                        │  └──────────────────────────┘   │
                        └────────┬────────────────────────┘
                                 │
              ┌──────────────────┼─────────────────────┐
              │                  │                      │
              ▼                  ▼                      ▼
   ┌─────────────────┐  ┌──────────────┐    ┌─────────────────┐
   │   PostgreSQL    │  │  RabbitMQ    │    │     Redis       │
   │ (3 schemas)     │  │  (queues)    │    │  (cache/SSE)    │
   └─────────────────┘  └──────┬───────┘    └─────────────────┘
                               │
         ┌─────────────────────┼──────────────────────────┐
         │                     │                          │
         ▼                     ▼                          ▼
┌────────────────┐   ┌──────────────────┐      ┌─────────────────┐
│  M0_PRIORITY   │   │  S2_PROCESSING   │      │  SSE_EXCHANGE   │
│    _QUEUE      │   │     _QUEUE       │      │  (RabbitMQ      │
└───────┬────────┘   └────────┬─────────┘      │   fanout)       │
        │                     │                └─────────────────┘
        ▼                     ▼
┌───────────────┐    ┌────────────────┐
│  M0 SERVICE   │    │  S2 SERVICE    │
│   (port 9000) │    │   (port 7084)  │
│               │    │                │
│ Video Chunking│    │ KVS Clip       │
│ FFmpeg        │    │ Extraction     │
│               │    │                │
│  S3/Blob ←───┤    │ AWS KVS ←─────┤
│  Download     │    │ GetClip API    │
│  Upload chunks│    │ Upload clip    │
└───────┬───────┘    └────────┬───────┘
        │                     │
        │ M0_RESULTS_QUEUE    │ (clips fed back into M0_PRIORITY_QUEUE)
        ▼                     ↓
┌──────────────────────────────────────┐
│             M1 SERVICE               │
│             (port 7081)              │
│                                      │
│  Vision Language Model Inference     │
│  ┌──────────────────────────────┐   │
│  │  AWS Bedrock / Azure OpenAI  │   │
│  │  / Google Gemini             │   │
│  └──────────────────────────────┘   │
│                                      │
│  Input: video_url (S3 path)          │
│  Output: natural-language summary    │
│  Stored: summary.json → S3/Blob      │
└──────────────┬───────────────────────┘
               │ M1_INFERENCE_QUEUE
               ▼
┌──────────────────────────────────────┐
│             M2 SERVICE               │
│             (port 7082)              │
│                                      │
│  asyncio.gather() — 3 parallel tasks │
│                                      │
│  ┌──────────┐ ┌─────────┐ ┌───────┐ │
│  │  Event   │ │ Embed + │ │ Cache │ │
│  │Detection │ │  Index  │ │ Clean │ │
│  │  (LLM)   │ │(Qdrant) │ │  (S3) │ │
│  └────┬─────┘ └────┬────┘ └───────┘ │
│       │             │                │
│       ▼             ▼                │
│  PostgreSQL       Qdrant             │
│  createEvent    index_document       │
└──────────────────────────────────────┘
               │
               ▼
        Pipeline Complete
   (Process status → completed)
        │
        ▼
  Backend SSE / GraphQL
  notifies connected clients
```

---

## 5. Message Queue Architecture

### Queue Map

| Queue Name | Producer | Consumer | Message Type |
|------------|----------|----------|-------------|
| `M0_PRIORITY_QUEUE` | Backend, S2 | M0 | VideoProcessingRequest |
| `M0_RESULTS_QUEUE` | M0 | M1 | ChunkProcessingRequest |
| `M1_INFERENCE_QUEUE` | M1 | M2 | EventDetectionMessage |
| `S2_PROCESSING_QUEUE` | Backend, S2 (scheduler) | S2 | KVSTask |
| `M0_DEADLETTER_QUEUE` | M0 (on error) | Ops team | Failed M0 messages |
| `M1_DEADLETTER_QUEUE` | M1 (on error) | Ops team | Failed M1 messages |
| `M2_DEADLETTER_QUEUE` | M2 (on error) | Ops team | Failed M2 messages |
| `S2_PROCESSING_DLQ` | S2 (on error) | Ops team | Failed S2 tasks |
| `CONTEXT_SUMMARIZATION_QUEUE` | Backend | Backend worker | Context summary tasks |
| `SSE_EXCHANGE` | Backend events consumer | Backend SSE manager | Real-time notifications |

### Queue Features

- **Priority**: `M0_PRIORITY_QUEUE` supports message priority (0–10)
- **Dead-letter routing**: All queues have associated DLQs for failure capture
- **Prefetch (QoS)**: Each consumer sets `prefetch_count` to limit in-flight messages
- **Durability**: All queues are durable (survive RabbitMQ restarts)
- **Acknowledgment**: Messages ack'd only on success; nack'd (no requeue) on failure

---

## 6. Database Schema Design

Three PostgreSQL schemas provide logical isolation:

### SPECTRA_CORE
Core identity and configuration entities:
- `users` — user accounts and credentials
- `organizations` — tenant organizations
- `cohorts` — user groups within organizations
- `roles` — access control roles
- `org_config` — organization-level settings

### SPECTRA_OPS
Operational processing records:
- `batch_video_processes` — batch job records (status tracking)
- `batch_video_chunks` — individual chunk records per batch job
- `cam_video_processes` — live camera processing records
- `cam_video_chunks` — individual chunk records per live job
- `process_configs` — AI pipeline configuration per job

### SPECTRA_INSIGHTS
AI analysis results:
- `batch_events` — detected events from batch videos
- `live_events` — detected events from live cameras
- `chunk_outputs` — VLM summary references per chunk
- `chat_context` — conversation context for LangGraph agent
- `context_summaries` — aggregated context summaries

**pgvector** extension is installed for semantic similarity operations directly in SQL where needed.

---

## 7. Two Processing Pipelines

### Pipeline A: Batch Video (Pre-recorded)

```
User uploads video URL via Backend GraphQL
              ↓
Backend: createBatchProcess (DB) + publish to M0_PRIORITY_QUEUE
              ↓
M0: Download → chunk → upload × N → createBatchChunk × N
              ↓
M1: VLM inference per chunk → createChunkOutput
              ↓
M2: Event detection + embedding + cache cleanup per chunk
              ↓
All chunks complete → updateBatchVideoProcessingStatus(completed)
```

**Status progression**: `pending → queued → processing → completed / failed`

### Pipeline B: Live Camera Stream

```
Scheduler / User publishes task to S2_PROCESSING_QUEUE
              ↓
S2: KVS GetClip → upload clip → publish to M0_PRIORITY_QUEUE
              ↓
M0: Single chunk (clip = 1 chunk) → createLiveChunk
              ↓
M1: VLM inference on clip → createChunkOutput
              ↓
M2: Event detection + embedding + cache cleanup
              ↓
updateCamVideoProcessingStatus(completed)
```

**Key difference**: Live jobs use `inference_modality: "live"` which routes to live-specific DB tables.

---

## 8. Shared Core Module Architecture

The `core/` directory at the project root is shared across ALL services via symbolic links:

```
core/
├── config.py              # Single Pydantic Config instance
├── security.py            # JWT, bcrypt, cloud secrets loader
├── database_client.py     # PostgreSQL + SQLAlchemy async sessions
├── rmq_client.py          # RabbitMQ publish helpers + queue declarations
├── rmq_consumer.py        # BaseConsumer abstract class
├── redis_client.py        # Redis connection management
├── storage_client.py      # S3/Blob abstraction (upload/download/delete)
├── kvs_client.py          # AWS KVS operations
├── http_client.py         # Async HTTP + GraphQL client
├── llm_client.py          # Multi-provider LLM interface
├── search_client.py       # Qdrant vector DB client
├── logger_setup.py        # Structured logging
├── observability.py       # Langfuse setup
├── prompt_handler.py      # Prompt management with Redis cache
├── notification_client.py # SSE connection manager
├── sns_client.py          # AWS SNS publishing
├── enums.py               # Shared status enums
└── providers/
    ├── factory.py         # LLM provider factory
    ├── aws/               # Bedrock, Secrets Manager, S3
    ├── azure/             # Azure OpenAI, Key Vault, Blob
    ├── google/            # Gemini
    └── openai/            # OpenAI direct
```

**Symlink pattern** (development):
```bash
sightapi_m0/src/core → ../../core
sightapi_m1/src/core → ../../core
sightapi_m2/src/core → ../../core
sightapi_s2/src/core → ../../core
backend/src/core     → ../../core
```

**Docker build**: `core/` is `COPY`'d directly into each container (no symlinks in containers).

---

## 9. Configuration System

### Bootstrap (Environment Variables)

These are required before secrets can be loaded:

```bash
CLOUD_PROVIDER=aws           # or "azure"
CLOUD_SECRETS_IDENTIFIER=arn:aws:secretsmanager:us-east-1:123:secret:spectra-prod
LLM_PROVIDERS_TO_INIT=bedrock,google
```

### Application Secrets (from Secrets Manager / Key Vault)

After `setup_environment()` runs at startup, `config` is updated with:

```
POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_HOST, POSTGRES_DB
RABBITMQ_USERNAME, RABBITMQ_PASSWORD, RABBITMQ_HOST
REDIS_HOST, REDIS_PASSWORD
JWT_SECRET_KEY
LANGFUSE_PUBLIC_KEY, LANGFUSE_SECRET_KEY
AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (if not using IAM roles)
```

### Service-Specific Config

M0:
- `M0_CHUNK_DURATION` — default chunk size in seconds
- `M0_DEFAULT_FPS`, `M0_DEFAULT_WIDTH`, `M0_DEFAULT_HEIGHT`

M1:
- `M1_VLM_PROVIDER`, `M1_VLM_MODEL`
- `M1_SYSTEM_PROMPT_KEY`

M2:
- `QDRANT_HOST`, `QDRANT_PORT`, `QDRANT_COLLECTION_NAME`
- `M2_EMBEDDING_PROVIDER`, `M2_EMBEDDING_MODEL`

S2:
- `S2_CLIP_DURATION` — live clip extraction duration
- `S2_KVS_MAX_FFMPEG_WORKERS`
- `AWS_REGION` — KVS stream region

---

## 10. Async & Worker Architecture

### Consumer Pattern

All four pipeline services (M0, M1, M2, S2) use the same `BaseConsumer` pattern:

```python
class BaseConsumer(ABC):
    def __init__(self, max_concurrent: int = config.max_concurrent_tasks):
        self._semaphore = asyncio.Semaphore(max_concurrent)
        self._tasks: set[asyncio.Task] = set()

    async def start_consuming(self):
        queue = await channel.declare_queue(self.get_queue_name(), passive=True)
        await queue.consume(self._handle_message)

    async def _handle_message(self, message: aio_pika.IncomingMessage):
        async with self._semaphore:           # Back-pressure control
            async with message.process():    # Auto-ack on success
                try:
                    await self.process_message(message)
                except Exception as e:
                    await self._forward_to_dlq(message, e)

    @abstractmethod
    async def process_message(self, message): ...
```

**Concurrency model**:
- Each service runs a single consumer coroutine in the event loop
- Semaphore limits how many messages can be processed simultaneously
- All I/O operations (storage, DB, LLM) are `await`-ed — no blocking

### M2 Parallel Task Execution

M2's `asyncio.gather()` is the most parallelism-intensive pattern:

```python
results = await asyncio.gather(
    process_event_detection(msg),    # LLM call
    process_embedding_and_index(msg), # embed + Qdrant write
    delete_cache_chunk(msg),          # S3 delete
    return_exceptions=True
)
```

All three complete concurrently. Total M2 latency ≈ max(event_detection_time, embedding_time, deletion_time) — not their sum.

---

## 11. Observability & Monitoring

### Langfuse (LLM Tracing)

- Every LLM call in M1 (VLM inference) and M2 (event detection, embedding) is traced
- Captured data: input messages, output text, model ID, token counts, latency, cost
- Session ID and user ID attached for audit trail
- Prompts versioned and fetched via `PromptHandler` with Redis caching

### Structured Logging

All services use `core/logger_setup.py`:

```
[2026-02-26 14:30:01] [INFO] consumer_service: M0 chunk processed in 2.34s chunk_id=1
[2026-02-26 14:30:03] [INFO] vlm_inference_service: VLM inference complete chunk_id=1 tokens=1243
[2026-02-26 14:30:05] [ERROR] consumer_service: M2 failed for chunk_id=2: ConnectionError
```

### Health Endpoints

Every service exposes `GET /health`:

| Service | Health Reports |
|---------|---------------|
| Backend | DB, Redis, RabbitMQ, LLM |
| M0 | RabbitMQ |
| M1 | RabbitMQ, LLM provider |
| M2 | RabbitMQ, LLM, Qdrant, Storage |
| S2 | RabbitMQ |

---

## 12. Security Architecture

### Authentication Flow

```
Client → POST /api/v1/auth/login
       → JWT access token (short TTL) + refresh token (long TTL)
       → All subsequent requests: Authorization: Bearer <token>
       → Auth middleware validates JWT on every request
       → Role-based access control via middleware
```

### Secret Management

- **AWS**: Secrets Manager → `setup_environment()` fetches at startup
- **Azure**: Key Vault → same pattern
- No secrets in environment variables for production (only bootstrap credentials)
- `JWT_SECRET_KEY` minimum 32 characters enforced at startup

### Network Security

- All services communicate internally (no public exposure except Backend port 8080)
- RabbitMQ: optional SSL/TLS
- S3/Blob: IAM roles or managed identities (preferred over access keys)
- Presigned URLs for temporary client access to stored videos

---

## 13. Deployment Architecture

### Docker

Each service has its own `Dockerfile`:

```
sightapi_m0/deployment/docker/Dockerfile
sightapi_m1/deployment/docker/Dockerfile
sightapi_m2/deployment/docker/Dockerfile
sightapi_s2/deployment/docker/Dockerfile
backend/deployment/docker/Dockerfile
```

Build command pattern:
```bash
docker build -f sightapi_m0/deployment/docker/Dockerfile \
             -t <ecr-uri>/sightapi_m0:<version> .
```

Note: Build context is the **repository root** so `core/` can be `COPY`'d into each image.

### Kubernetes

YAML manifests in each service's `deployment/k8s/` directory:
- `deployment.yaml` — pod spec, replicas, resource limits
- `service.yaml` — ClusterIP / NodePort
- `configmap.yaml` — non-secret configuration
- `hpa.yaml` — Horizontal Pod Autoscaler (optional)

### Infrastructure as Code

CloudFormation templates in `deployment/cloudformation/`:
```bash
make deploy    # Deploy full stack
make update    # Update existing stack
make status    # Check stack status
make outputs   # Show endpoint URLs
```

### Service Communication Matrix

| From → To | Method | Details |
|-----------|--------|---------|
| Client → Backend | HTTPS REST / GraphQL | Port 8080 |
| Client → Backend | SSE | Port 8083 (or same) |
| Backend → M0 | RabbitMQ | M0_PRIORITY_QUEUE |
| Backend → S2 | RabbitMQ | S2_PROCESSING_QUEUE |
| M0 → M1 | RabbitMQ | M0_RESULTS_QUEUE |
| M1 → M2 | RabbitMQ | M1_INFERENCE_QUEUE |
| S2 → M0 | RabbitMQ | M0_PRIORITY_QUEUE |
| M0/M1/M2 → Backend DB | GraphQL HTTP | Internal HTTP call |
| All → PostgreSQL | TCP | SQLAlchemy async |
| All → Redis | TCP | aioredis |
| M0/S2 → Storage | HTTPS | S3/Blob SDK |
| M1 → LLM Provider | HTTPS | Bedrock/Azure/Gemini |
| M2 → Qdrant | HTTP | qdrant-client |

---

## 14. Real End-to-End Example

**Scenario**: A facility manager uploads a 10-minute warehouse security video for analysis.

```
t=0s:   Manager POSTs video URL via GraphQL
        Backend: createBatchProcess(process_id=300, status=pending)
        Backend: publish M0_PRIORITY_QUEUE({ process_id:300, url:"s3://.../warehouse.mp4" })

t=0.5s: M0 picks up message
        Downloads warehouse.mp4 (500MB) → /tmp/m0_300/

t=15s:  Download complete. FFmpeg begins chunking (10 chunks × 60s each):

        Chunk 1 (0–60s):
          FFmpeg → chunk_001.mp4
          Upload → s3://bucket/processed/300/chunk_001.mp4
          Upload → s3://cache/temp/300/chunk_001.mp4
          createBatchChunk(id=1, process_id=300, start=0, end=60)
          Publish to M0_RESULTS_QUEUE

        [M1 starts chunk_001 inference while M0 continues chunking]

        Chunk 2 (60–120s): ... same pattern
        ...
        Chunk 10 (540–600s): created and published

t=18s:  M1 receives chunk_001:
        VLM (Bedrock nova-pro): "A forklift moves from aisle 3 to 7.
                                 Two workers inspect pallets."
        Summary → s3://bucket/processed/300/chunk_001_summary.json
        updateChunkStatus(1, m1_completed)
        Publish to M1_INFERENCE_QUEUE

t=19s:  M2 receives chunk_001 (parallel):
        [Event Detection]: [{ type: vehicle_movement }, { type: worker_activity }]
        [Embedding]: [0.23, -0.11, ...] → Qdrant indexed
        [Cache Delete]: cache/temp/300/chunk_001.mp4 removed
        updateChunkStatus(1, completed)

[Chunks 2–10 follow the same pattern, concurrently]

t=120s: All 10 chunks processed:
        updateBatchVideoProcessingStatus(300, completed)
        Backend SSE → notifies manager: "Analysis complete"

Manager searches: "forklift near aisle 7"
→ Qdrant returns chunk_001 (similarity: 0.94), chunk_004 (0.89)
→ Backend returns { chunk_id, timestamp: 0:00, summary: "..." }
→ Manager watches the relevant 60-second clip
```

---

## 15. Scaling Considerations

| Service | Scaling Strategy | Bottleneck |
|---------|-----------------|-----------|
| Backend | Horizontal (stateless) | DB connections |
| M0 | Horizontal (one consumer per instance) | FFmpeg CPU + download bandwidth |
| M1 | Horizontal (limited by LLM rate limits) | LLM API quotas |
| M2 | Horizontal (Qdrant write throughput) | Qdrant indexing speed |
| S2 | Horizontal (one consumer per instance) | KVS API rate limits |
| RabbitMQ | Cluster mode for HA | Message persistence I/O |
| PostgreSQL | Read replicas for read-heavy queries | Write throughput |
| Qdrant | Distributed cluster for large collections | Vector index memory |
