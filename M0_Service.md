# M0 Service — Video Preprocessing & Chunking

**Service Name**: `sightapi_m0`
**Port**: `9000`
**Role in Pipeline**: Stage 1 — Ingestion & Preprocessing
**Queue In**: `M0_PRIORITY_QUEUE`
**Queue Out**: `M0_RESULTS_QUEUE`

---

## 1. Overview

The M0 service is the **entry point for all video data** in the Spectra Sight pipeline. Its sole purpose is to receive raw video files, break them into fixed-duration chunks, upload those chunks to cloud storage, persist chunk metadata to the database, and forward each chunk downstream for AI inference.

### Why M0 Exists

Vision Language Models (VLMs) cannot efficiently process hour-long videos in a single pass. The M0 service solves this by:

- Dividing long-form video into short, manageable time segments (chunks)
- Making each chunk independently processable by downstream services (M1, M2)
- Enabling parallelism: multiple chunks from the same video can be processed simultaneously
- Supporting both **batch** (pre-recorded) and **live** (real-time stream) modalities

### Problems It Solves

| Problem | Solution |
|---------|----------|
| VLMs have token/frame limits | Chunk video into short segments |
| Long videos block inference | Each chunk processed independently |
| Re-encoding is slow | Supports fast stream-copy mode |
| Chunk loss on failure | Dead-letter queue (DLQ) captures failed messages |
| High latency on large files | Streaming generator — chunks uploaded as they are created |

---

## 2. Service Responsibilities

- Validate incoming video processing requests
- Download source video from cloud storage (S3 / Azure Blob)
- Split video into time-based chunks using **FFmpeg**
- Support two processing modes: **stream copy** (fast, no re-encode) and **transcode** (resize + re-sample)
- Upload each chunk to **two destinations**: persistent storage + cache storage (with TTL)
- Save chunk metadata to the database via **GraphQL mutations**
- Publish each chunk as a message to `M0_RESULTS_QUEUE` for M1 consumption
- Update the video processing job status (QUEUED → PROCESSING → COMPLETED/FAILED)
- Clean up temporary files after processing

---

## 3. Detailed Execution Flow

```
Step 1:  Backend publishes a VideoProcessingRequest to M0_PRIORITY_QUEUE
Step 2:  M0QueueConsumer picks up the message (semaphore-controlled concurrency)
Step 3:  JSON payload is parsed and validated
Step 4:  Source video is downloaded from cloud storage to a temp directory
Step 5:  Chunk duration and processing mode are read from config
Step 6:  create_chunks_streaming() generator begins yielding chunks one at a time
Step 7:  For each chunk:
           a. FFmpeg splits the segment (stream copy or transcode)
           b. A unique hash ID is generated for the chunk
           c. Chunk is uploaded to persistent storage (async)
           d. Chunk is uploaded to cache storage with TTL (async)
           e. Chunk metadata saved to DB via GraphQL mutation (createBatchChunk / createLiveChunk)
           f. Chunk message published to M0_RESULTS_QUEUE
Step 8:  After all chunks: process status updated to PROCESSING
Step 9:  Temp directory and downloaded video file are deleted
Step 10: On any error → DLQ routing + process status set to FAILED
```

---

## 4. Architecture Flow Diagram

```
Client / Backend Service
         |
         | POST /api/v1/process_video
         v
M0 FastAPI Endpoint (port 9000)
         |
         | Publishes JSON message
         v
RabbitMQ: M0_PRIORITY_QUEUE
         |
         | BaseConsumer.start_consuming()
         v
M0QueueConsumer.process_message()
         |
         +---> Download video from S3/Blob Storage
         |
         +---> FFmpeg: split into chunks (streaming generator)
         |
         | For each chunk (parallel per chunk):
         +---> Upload chunk (persistent S3/Blob)
         +---> Upload chunk (cache S3/Blob with TTL)
         +---> GraphQL Mutation: createBatchChunk / createLiveChunk
         +---> Publish to M0_RESULTS_QUEUE
         |
         v
RabbitMQ: M0_RESULTS_QUEUE
         |
         v
M1 Service (VLM Inference)
```

---

## 5. Connections and Integrations

### RabbitMQ

| Queue | Direction | Purpose |
|-------|-----------|---------|
| `M0_PRIORITY_QUEUE` | Inbound | Receives video processing requests from backend |
| `M0_RESULTS_QUEUE` | Outbound | Forwards per-chunk messages to M1 |
| `M0_DEADLETTER_QUEUE` | Outbound (on error) | Failed messages for manual inspection/retry |

- **Data sent to M0_RESULTS_QUEUE**: process_id, chunk_id, chunk hash, video URL in storage, chunk start/end times, inference config, modality (batch/live)
- **Data sent to DLQ**: Original message body + error context

### Cloud Storage (S3 / Azure Blob)

| Operation | Direction | Purpose |
|-----------|-----------|---------|
| `download()` | Inbound | Download source video to local temp dir |
| `upload()` (persistent) | Outbound | Permanent chunk storage for M1 inference |
| `upload()` (cache, with TTL) | Outbound | Temporary cache; deleted by M2 after inference |

### PostgreSQL (via GraphQL)

- **Mutations called**:
  - `createBatchChunk` — saves chunk record for batch video jobs
  - `createLiveChunk` — saves chunk record for live camera stream jobs
  - `updateBatchVideoProcessingStatus` — updates batch job status
  - `updateCamVideoProcessingStatus` — updates live job status
- **Data saved per chunk**: chunk_hash, duration, start_time, end_time, time_offset, cloud storage URL, process_id

### Backend Service (HTTP)

- M0 receives its initial trigger from the backend's `POST /api/v1/process_video` endpoint
- The backend publishes to RabbitMQ; M0 does not call back to the backend directly

### FFmpeg (System Process)

- Invoked as a subprocess for video manipulation
- Used for both stream-copy (fast) and transcode (resize/fps) modes
- All FFmpeg calls are async-wrapped via `asyncio.create_subprocess_exec`

---

## 6. Data Flow

```
Input
  Cloud Storage URL (S3 or Azure Blob)
  e.g. s3://my-bucket/videos/input.mp4
         |
         v
Download to local temp directory
  /tmp/m0_<process_id>/input.mp4
         |
         v
FFmpeg Chunking (streaming generator)
  Mode A: Stream Copy  →  fast, no quality loss
    ffmpeg -i input.mp4 -ss <start> -t <duration> -c copy chunk_001.mp4
  Mode B: Transcode  →  resize + resample FPS
    ffmpeg -i input.mp4 -ss <start> -t <duration>
           -vf "scale=1280:720,fps=30"
           -c:v libx264 -preset ultrafast chunk_001.mp4
         |
         v
Per Chunk Parallel Actions:
  [Upload to Persistent Storage]       [Upload to Cache Storage]
  s3://bucket/chunks/chunk_001.mp4     s3://cache/temp/chunk_001.mp4
                 |                              |
                 +----------+---------+---------+
                            |
                            v
              GraphQL Mutation: createBatchChunk
              {
                chunk_hash: "abc123",
                process_id: 42,
                chunk_duration: 30.0,
                start_time: 0.0,
                end_time: 30.0,
                cloud_path: "s3://bucket/chunks/chunk_001.mp4"
              }
                            |
                            v
              Publish to M0_RESULTS_QUEUE
              {
                "process_id": 42,
                "chunk_id": 1,
                "chunk_hash": "abc123",
                "cloud_stream_path": "s3://bucket/chunks/chunk_001.mp4",
                "cache_path": "s3://cache/temp/chunk_001.mp4",
                "start_time": 0.0,
                "end_time": 30.0,
                "inference_config": { ... }
              }
```

---

## 7. Code-Level Explanation

### Entry Point: `main.py`

```
sightapi_m0/src/main.py
```

- Creates a FastAPI application with a `lifespan` context manager
- Startup sequence:
  1. `setup_environment()` — load cloud secrets
  2. `setup_storage()` — initialize S3/Blob client
  3. `setup_http_client()` — async HTTP session
  4. `connect_rabbitmq()` → `declare_application_queues()` — RabbitMQ setup
  5. `asyncio.create_task(consumer.run())` — start consuming messages in background
- Registers `/health` and video processing router

### REST Endpoint: `api/v1/endpoints/video.py`

**Function**: `process_video(request: VideoProcessingRequest)`
**Route**: `POST /api/v1/process_video`

- Validates the request (source URL, destination prefix, cache container)
- Builds an internal processing message
- Calls `publish_to_m0_priority_queue(message)` from `core/rmq_client.py`
- Returns `{"process_id": ..., "status": "QUEUED", "message": "..."}`
- This endpoint does NOT wait for processing; it returns immediately

### Consumer: `services/queue_consumer.py`

**Class**: `M0QueueConsumer(BaseConsumer)`

Key method: `process_message(message: aio_pika.IncomingMessage)`

Internal flow:
1. `json.loads(message.body)` — decode message
2. `storage_client.download(container, key, local_path)` — fetch video
3. `create_chunks_streaming(...)` — async generator
4. Per-chunk: `asyncio.gather(upload_dual, save_metadata, publish_to_queue)`
5. Update process status via GraphQL
6. `cleanup_temp_files(temp_dir)` in `finally` block

Concurrency: inherits `asyncio.Semaphore` from `BaseConsumer`; limits simultaneous video jobs

### Video Processor: `services/video_service.py`

**Function**: `create_chunks_streaming(input_path, output_dir, config)`

- An `async generator` that yields `(chunk_path, chunk_metadata)` tuples
- Calculates chunk boundaries from total video duration and configured chunk size
- Handles the "last chunk" edge case: if remaining video is too short, merges with previous chunk
- Calls either `split_video_segment()` or `transcode_video_segment()` per chunk

**Function**: `split_video_segment(input_path, output_path, start, duration)`

- FFmpeg stream-copy: `-c copy` — fastest mode, no quality change
- Used when resolution/FPS changes are NOT needed

**Function**: `transcode_video_segment(input_path, output_path, start, duration, fps, width, height)`

- FFmpeg re-encode: `libx264 -preset ultrafast`
- Applies `-vf "scale=W:H,fps=F"` filter
- Used when source video needs normalization

**Function**: `upload_chunk_dual_destination(chunk_path, persistent_container, cache_container, key)`

- `asyncio.gather()` runs both uploads in parallel
- Returns `(persistent_url, cache_url)`

---

## 8. Sequence Flow

```
Backend
  │
  │  1. POST /api/v1/process_video
  │     { process_id, cloud_stream_path, destination_prefix, video_preprocessing }
  ↓
M0 Endpoint
  │
  │  2. Validates fields
  │  3. publish_to_m0_priority_queue(message)
  ↓
RabbitMQ [M0_PRIORITY_QUEUE]
  │
  │  4. M0QueueConsumer picks up message
  ↓
M0 Consumer
  │
  │  5. Download video → /tmp/m0_42/input.mp4
  │  6. FFmpeg chunk 1 → /tmp/m0_42/chunk_001.mp4
  │
  │  7. Parallel:
  │     ├── Upload /tmp/chunk_001.mp4 → s3://bucket/chunk_001.mp4
  │     ├── Upload /tmp/chunk_001.mp4 → s3://cache/chunk_001.mp4
  │     ├── GraphQL: createBatchChunk(chunk_001, process_42)
  │     └── publish_to_m0_results_queue({ chunk_001 details })
  │
  │  8. Repeat for chunk_002, chunk_003 ... chunk_N
  │
  │  9. GraphQL: updateBatchVideoProcessingStatus(42, PROCESSING)
  │  10. cleanup_temp_files(/tmp/m0_42/)
  ↓
RabbitMQ [M0_RESULTS_QUEUE]  →  M1 Service
```

---

## 9. Dependency Explanation

| Dependency | Purpose |
|------------|---------|
| `FastAPI` | REST API server |
| `aio-pika` | Async RabbitMQ client |
| `aiohttp` | Async HTTP client (cloud calls) |
| `aiobotocore` / Azure SDK | Cloud storage (S3 / Blob) |
| `asyncio` | Concurrency primitives |
| `FFmpeg` (system binary) | Video chunking (subprocess) |
| `core/rmq_client.py` | Queue connection + publish helpers |
| `core/storage_client.py` | Cloud storage abstraction |
| `core/http_client.py` | GraphQL mutations to backend |
| `core/config.py` | Centralized configuration |
| `core/rmq_consumer.py` | `BaseConsumer` base class |
| `core/logger_setup.py` | Structured logging |

---

## 10. Error Handling Flow

```
Error in process_message()
         |
         +---> Log full exception with process_id and chunk_id
         |
         +---> GraphQL: updateStatus(process_id, FAILED)
         |
         +---> Publish original message body to M0_DEADLETTER_QUEUE
         |
         +---> finally: cleanup_temp_files() — always runs
         |
         +---> BaseConsumer marks message as nack (rejected)
```

**Retry Logic**: No automatic retry at consumer level. Failed messages go to DLQ for manual inspection.
**Semaphore**: If max concurrent jobs are running, new messages wait in queue (back-pressure).
**FFmpeg failure**: If subprocess returns non-zero exit code, exception raised → DLQ path.
**Upload failure**: If either cloud upload fails, exception raised → DLQ path.

---

## 11. Deployment

- **Container**: Docker image built from `sightapi_m0/deployment/docker/Dockerfile`
- **Runtime**: Uvicorn + FastAPI on port `9000`
- **Workers**: Configured via `config.workers` (default: 1 for consumer-based services)
- **Kubernetes**: Manifest at `sightapi_m0/deployment/k8s/deployment.yaml`
- **FFmpeg**: Must be installed in the Docker image (`apt-get install ffmpeg`)
- **Environment**: Cloud credentials injected as environment variables or from Secrets Manager / Key Vault at startup

---

## 12. Real Example Flow

**Scenario**: A 5-minute MP4 batch video needs to be analyzed. Chunk size = 60 seconds.

```
Input Message (published to M0_PRIORITY_QUEUE):
{
  "process_id": 101,
  "inference_modality": "batch",
  "cloud_stream_path": "s3://spectra-videos/uploads/security_cam_feed.mp4",
  "destination_prefix": "processed/101/",
  "cache_container_name": "spectra-cache",
  "video_preprocessing": {
    "chunk_duration": 60,
    "fps": 30,
    "width": 1280,
    "height": 720
  }
}

Step 1: M0 downloads security_cam_feed.mp4 (300 seconds) → /tmp/m0_101/input.mp4

Step 2: Chunk boundaries calculated:
  chunk_001: 0s   → 60s
  chunk_002: 60s  → 120s
  chunk_003: 120s → 180s
  chunk_004: 180s → 240s
  chunk_005: 240s → 300s

Step 3: For chunk_001 (0→60s):
  FFmpeg: ffmpeg -i input.mp4 -ss 0 -t 60 -c copy chunk_001.mp4
  Upload → s3://spectra-videos/processed/101/chunk_001.mp4
  Upload → s3://spectra-cache/temp/101/chunk_001.mp4
  GraphQL: createBatchChunk({ process_id:101, hash:"d4f3...", start:0, end:60 })
  Publish to M0_RESULTS_QUEUE: { chunk_id:1, cloud_stream_path:"s3://.../chunk_001.mp4" }

... (repeats for chunks 2–5)

Step 4: GraphQL: updateBatchVideoProcessingStatus(101, PROCESSING)
Step 5: rm -rf /tmp/m0_101/

M1 receives 5 messages (one per chunk) and processes them concurrently.
```
