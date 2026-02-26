# S2 Service — Live Camera Stream Processing (KVS)

**Service Name**: `sightapi_s2`
**Port**: `7084`
**Role in Pipeline**: Parallel Live Stream Branch — Camera Ingestion via AWS KVS
**Queue In**: `S2_PROCESSING_QUEUE`
**Queue Out**: M0 pipeline (publishes camera clip for M0 → M1 → M2 processing)

---

## 1. Overview

The S2 service is a **specialized live stream processing branch** that handles real-time video from physical cameras connected to **AWS Kinesis Video Streams (KVS)**. While M0 handles pre-recorded batch videos, S2 is the entry point for continuously-streaming camera feeds.

S2 periodically or on-demand extracts short video clips from live KVS streams, uploads those clips to cloud storage, and then feeds them into the same M0 → M1 → M2 pipeline that processes batch videos — making live and batch analysis functionally equivalent once the clip is extracted.

### Why S2 Exists

AWS Kinesis Video Streams provides a managed service for ingesting live camera data. However, the M0 pipeline cannot directly consume a live HLS stream — it needs a discrete, downloadable video file. S2 bridges this gap by:

- Discovering and monitoring active camera streams
- Extracting fixed-duration video clips from KVS using timestamp-based APIs
- Uploading clips to cloud storage
- Triggering the standard M0 → M1 → M2 analysis pipeline on each clip

### Problems It Solves

| Problem | Solution |
|---------|----------|
| M0 requires a file URL, not a live stream | S2 extracts clips from KVS and uploads them |
| Live cameras need continuous monitoring | S2 consumer processes tasks on a schedule |
| Camera discovery is manual | `camera_discovery_service` auto-discovers KVS streams |
| Different cameras need independent processing | Each KVS task is processed independently |
| Redis caching prevents duplicate discovery | Camera list cached in Redis |

---

## 2. Service Responsibilities

- Consume camera processing tasks from `S2_PROCESSING_QUEUE`
- Connect to **AWS Kinesis Video Streams** via the KVS client
- Discover all active camera streams in the AWS account
- Extract video clips from KVS streams using time-range queries
- Upload extracted clips to cloud storage (S3 / Azure Blob)
- Generate thumbnail images from clips
- Publish clip details to M0's queue (or directly to the backend) for AI analysis
- Cache camera discovery results in Redis to avoid redundant API calls
- Expose manual trigger endpoints for on-demand clip extraction

---

## 3. Detailed Execution Flow

```
Step 1:  A task is published to S2_PROCESSING_QUEUE
         (by the backend scheduler, user action, or S2 itself)
Step 2:  S2Consumer picks up the KVSTask message
Step 3:  JSON payload parsed to KVSTask Pydantic model
Step 4:  kvs_processor.process_camera_task(task) called:
           a. Validate stream ARN or stream name
           b. Check stream status (must be ACTIVE)
           c. Extract video clip using time range:
              kvs_client.get_video_clip(stream_id, start_time, end_time)
           d. Upload clip bytes to cloud storage:
              storage_client.upload(container, key, clip_bytes)
           e. Generate presigned URL for the clip
           f. (Optional) Generate thumbnail from clip
           g. Trigger M0 processing:
              Publish to M0_PRIORITY_QUEUE with clip URL
Step 5:  On success → log timing metrics
Step 6:  On error → route message to S2_PROCESSING_DLQ
```

---

## 4. Architecture Flow Diagram

```
Physical Camera / IP Camera
         |
         | RTSP / WebRTC
         v
AWS Kinesis Video Streams (KVS)
[Managed live video ingestion + storage]
         |
         | Task published (by scheduler / backend)
         v
RabbitMQ: S2_PROCESSING_QUEUE
         |
         | S2Consumer.process_message()
         v
S2 Consumer Service
         |
         +---> Parse KVSTask (stream_arn, start_time, end_time)
         |
         +---> KVS Client: check_stream_status(stream_arn)
         |
         +---> KVS Client: get_video_clip(stream_arn, start, end)
         |     [Returns binary MP4 clip data]
         |
         +---> Cloud Storage: upload(clip.mp4)
         |     [s3://spectra-videos/live/camera_id/clip_timestamp.mp4]
         |
         +---> (Optional) Generate thumbnail
         |
         +---> Publish to M0_PRIORITY_QUEUE:
               { cloud_stream_path: "s3://.../clip.mp4",
                 inference_modality: "live",
                 live_start_time: clip.start_time }
         |
         v
Standard M0 → M1 → M2 Pipeline
(Same as batch video processing)
```

---

## 5. Connections and Integrations

### RabbitMQ

| Queue | Direction | Purpose |
|-------|-----------|---------|
| `S2_PROCESSING_QUEUE` | Inbound | Receives camera clip extraction tasks |
| `S2_PROCESSING_DLQ` | Outbound (on error) | Failed processing messages |
| `M0_PRIORITY_QUEUE` | Outbound | Triggers M0 pipeline with extracted clip |

### AWS Kinesis Video Streams — via `core/kvs_client.py`

| Operation | Purpose |
|-----------|---------|
| `get_hls_streaming_url(stream_arn)` | Get live HLS URL for monitoring |
| `check_stream_status(stream_arn)` | Validate stream is ACTIVE before extracting |
| `get_video_clip(stream_id, start, end)` | Extract MP4 clip between timestamps |
| `get_all_cameras()` | List all KVS streams in the AWS account |

- **Data sent**: Stream ARN + time range (start_time, end_time as datetime)
- **Data received**: Binary MP4 video data or stream URL

### Redis — via `core/redis_client.py`

- **Purpose**: Cache camera discovery results (list of KVS streams)
- **TTL**: Configurable; prevents hammering the KVS ListStreams API
- **Key pattern**: `s2:cameras:discovered` → JSON list of stream ARNs + names

### Cloud Storage (S3 / Azure Blob)

| Operation | Direction | Purpose |
|-----------|-----------|---------|
| `upload(clip.mp4)` | Outbound | Store extracted KVS clip |
| `generate_presigned_url()` | Outbound | Create time-limited URL for downstream |

### Backend Service (HTTP / RabbitMQ)

- S2 publishes the extracted clip URL to `M0_PRIORITY_QUEUE`
- M0 picks it up and treats the live clip like any other batch video
- The `inference_modality: "live"` flag tells M0/M1/M2 to use live-specific DB mutations

### Manual Trigger Endpoints (REST)

S2 exposes REST endpoints for on-demand operations:

| Endpoint | Purpose |
|----------|---------|
| `POST /trigger/clip` | Manually extract a clip from a stream |
| `POST /trigger/discover` | Force camera discovery refresh |

---

## 6. Data Flow

```
KVS Stream
  stream_arn: "arn:aws:kinesisvideo:us-east-1:123456789:stream/CameraFront/..."
  start_time: 2026-02-26T14:30:00Z
  end_time:   2026-02-26T14:31:00Z  (60 second clip)
         |
         v
KVS API: GetClip → binary MP4 data
         |
         v
Upload to Cloud Storage:
  s3://spectra-videos/live/camera_front/2026-02-26T14-30-00Z.mp4
         |
         v
Publish to M0_PRIORITY_QUEUE:
{
  "process_id": 201,
  "inference_modality": "live",
  "cloud_stream_path": "s3://spectra-videos/live/camera_front/2026-02-26T14-30-00Z.mp4",
  "destination_prefix": "processed/live/201/",
  "cache_container_name": "spectra-cache",
  "live_start_time": "2026-02-26T14:30:00Z",
  "video_preprocessing": {
    "chunk_duration": 60,
    "fps": 15,
    "width": 1280,
    "height": 720
  }
}
         |
         v
M0 Service picks up message → same pipeline as batch
M1: VLM inference on the live clip
M2: Event detection + embedding + indexing
```

---

## 7. Code-Level Explanation

### Entry Point: `main.py`

```
sightapi_s2/src/main.py
```

Startup sequence:
1. `setup_environment()` — load cloud secrets
2. `setup_storage()` — S3/Blob client
3. `setup_http_client()` — async HTTP
4. `connect_rabbitmq()` → `declare_application_queues()`
5. `setup_kvs_client()` — initialize AWS KVS boto3 session
6. `setup_redis_client()` — for camera discovery cache
7. `asyncio.create_task(consumer.run())`

`/health` endpoint reports RabbitMQ connection status.

### Consumer: `services/consumer_service.py`

**Class**: `S2Consumer(BaseConsumer)`

```python
async def process_message(self, message):
    t_start = time.monotonic()
    payload = json.loads(message.body)
    task = KVSTask(**payload)
    await kvs_processor.process_camera_task(task)
    logger.info(f"S2 task {task.camera_id} processed in {time.monotonic()-t_start:.2f}s")
```

Queue config:
- Input: `S2_PROCESSING_QUEUE`
- DLQ: `S2_PROCESSING_DLQ`

### KVS Processor: `services/kvs_processing_service.py`

**Function**: `process_camera_task(task: KVSTask)`

1. Parse stream identifier (ARN or name)
2. `kvs_client.check_stream_status(stream_arn)` — abort if not ACTIVE
3. `kvs_client.get_video_clip(stream_arn, task.start_time, task.end_time)` — get MP4 bytes
4. Generate storage key: `live/{camera_id}/{timestamp}.mp4`
5. `storage_client.upload(container, key, clip_bytes)` — upload
6. `publish_to_m0_priority_queue(message)` — trigger standard pipeline
7. Return storage URL

### Camera Discovery: `services/camera_discovery_service.py`

**Function**: `discover_cameras() → List[CameraInfo]`

1. Check Redis cache: `redis.get("s2:cameras:discovered")`
2. If cache miss: `kvs_client.get_all_cameras()` — list all KVS streams
3. Filter to ACTIVE streams only
4. Cache result in Redis with TTL
5. Return list of `CameraInfo` objects (stream_arn, stream_name, status)

**Function**: `get_active_camera_arns() → List[str]`

- Thin wrapper around `discover_cameras()`
- Used by the scheduler when publishing batch tasks to `S2_PROCESSING_QUEUE`

### Schema: `schemas/kvs_task.py`

**Model**: `KVSTask`

```python
class KVSTask(BaseModel):
    camera_id: str           # Unique camera identifier
    stream_arn: str          # AWS KVS stream ARN
    stream_name: str         # Human-readable stream name
    start_time: datetime     # Clip extraction start
    end_time: datetime       # Clip extraction end
    process_id: int          # DB process record ID
    inference_config: dict   # Passed through to M0/M1/M2
```

### Manual Trigger: `api/v1/endpoints/trigger.py`

**Route**: `POST /trigger/clip`

```python
async def trigger_clip(request: ClipTriggerRequest):
    task = KVSTask(
        camera_id=request.camera_id,
        stream_arn=request.stream_arn,
        start_time=request.start_time,
        end_time=request.end_time,
        ...
    )
    await publish_to_s2_processing_queue(task.dict())
    return {"status": "QUEUED", "camera_id": request.camera_id}
```

---

## 8. Sequence Flow

```
Backend Scheduler / User Action
  │
  │  publish_to_s2_processing_queue({
  │    camera_id: "camera_front",
  │    stream_arn: "arn:aws:kinesisvideo:...",
  │    start_time: "2026-02-26T14:30:00Z",
  │    end_time:   "2026-02-26T14:31:00Z",
  │    process_id: 201
  │  })
  ↓
RabbitMQ [S2_PROCESSING_QUEUE]
  │
  │  S2Consumer.process_message()
  ↓
S2 Consumer
  │
  │  1. Parse KVSTask
  │  2. kvs_client.check_stream_status("arn:aws:kinesisvideo:...") → ACTIVE
  │  3. kvs_client.get_video_clip("arn:...", 14:30, 14:31) → clip_bytes (MP4)
  │  4. storage_client.upload("spectra-videos", "live/camera_front/14-30.mp4", clip_bytes)
  │  5. publish_to_m0_priority_queue({
  │       process_id: 201,
  │       inference_modality: "live",
  │       cloud_stream_path: "s3://spectra-videos/live/camera_front/14-30.mp4",
  │       live_start_time: "2026-02-26T14:30:00Z"
  │     })
  ↓
RabbitMQ [M0_PRIORITY_QUEUE]
  │
  ↓
M0 → M1 → M2 Pipeline (standard batch flow with live_start_time offset)
```

---

## 9. Dependency Explanation

| Dependency | Purpose |
|------------|---------|
| `FastAPI` | REST API + manual trigger endpoints |
| `aio-pika` | Async RabbitMQ consumer |
| `aiobotocore` | Async AWS SDK for KVS and S3 |
| `aioredis` | Async Redis for camera discovery cache |
| `core/kvs_client.py` | KVS stream operations (clip extraction, discovery) |
| `core/storage_client.py` | Upload clips to S3/Blob |
| `core/rmq_client.py` | Queue connection + publish to M0 |
| `core/rmq_consumer.py` | `BaseConsumer` abstract class |
| `core/redis_client.py` | Redis connection management |
| `core/http_client.py` | Optional GraphQL calls |
| `core/config.py` | KVS config, clip duration, storage containers |

---

## 10. Error Handling Flow

```
Error in process_camera_task()
         |
         +---> Log: f"S2 failed for camera_id={camera_id}: {error}"
         |
         Cases:
         | - Stream INACTIVE → log warning, skip processing (no DLQ)
         | - KVS clip extraction fails → exception → DLQ
         | - Storage upload fails → exception → DLQ
         | - M0 publish fails → exception → DLQ
         |
         +---> BaseConsumer: nack message → routes to S2_PROCESSING_DLQ
```

**Stream not active**: If KVS stream is not in ACTIVE state, the task is dropped gracefully (camera may be offline).
**KVS API throttling**: AWS KVS has rate limits; failed calls propagate as exceptions → DLQ.
**Clip too short**: If start_time == end_time or clip is empty, exception → DLQ.
**Redis failure**: Camera discovery falls back to direct KVS API call (Redis is a cache, not a dependency).

---

## 11. Deployment

- **Container**: `sightapi_s2/deployment/docker/Dockerfile`
- **Port**: `7084`
- **AWS Region**: Must be configured to match KVS stream region
- **IAM Permissions required**:
  - `kinesisvideo:GetDataEndpoint`
  - `kinesisvideo:GetClip`
  - `kinesisvideo:GetHLSStreamingSessionURL`
  - `kinesisvideo:ListStreams`
  - `kinesisvideo:DescribeStream`
  - `s3:PutObject` (for clip upload)
- **Key env vars**:
  - `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
  - `REDIS_HOST`, `REDIS_PASSWORD`
  - `S2_CLIP_DURATION` — default clip length in seconds
  - `S2_KVS_MAX_FFMPEG_WORKERS` — FFmpeg worker count for clip processing

---

## 12. Real Example Flow

**Scenario**: A security camera (Camera Front) streams continuously to KVS. Every 60 seconds, the backend publishes a task to S2 to extract and analyze the latest clip.

```
Task published to S2_PROCESSING_QUEUE (by backend scheduler, every 60s):
{
  "camera_id": "camera_front_entrance",
  "stream_arn": "arn:aws:kinesisvideo:us-east-1:123456789012:stream/FrontEntrance/9999",
  "stream_name": "FrontEntrance",
  "start_time": "2026-02-26T14:30:00Z",
  "end_time":   "2026-02-26T14:31:00Z",
  "process_id": 501,
  "inference_config": {
    "vlm_provider": "bedrock",
    "vlm_model": "amazon.nova-pro-v1",
    "system_prompt_key": "security_live_v1",
    "user_prompt": "Detect any security events, unauthorized access, or unusual behavior."
  }
}

Step 1: S2Consumer picks up message
Step 2: KVS status check: FrontEntrance stream → ACTIVE
Step 3: KVS GetClip API call:
  start: 2026-02-26T14:30:00Z
  end:   2026-02-26T14:31:00Z
  → Returns 60 seconds of MP4 data (~15MB)

Step 4: Upload to S3:
  s3://spectra-videos/live/camera_front_entrance/2026-02-26T14-30-00Z.mp4

Step 5: Publish to M0_PRIORITY_QUEUE:
{
  "process_id": 501,
  "inference_modality": "live",
  "cloud_stream_path": "s3://spectra-videos/live/camera_front_entrance/2026-02-26T14-30-00Z.mp4",
  "destination_prefix": "processed/live/501/",
  "cache_container_name": "spectra-cache",
  "live_start_time": "2026-02-26T14:30:00Z",
  "video_preprocessing": { "chunk_duration": 60, "fps": 15 },
  "inference_config": { ... same as above ... }
}

Step 6: M0 picks up message:
  - Since clip is already 60s = 1 chunk → single chunk created
  - Uploads to persistent + cache storage
  - GraphQL: createLiveChunk (live modality)
  - Publishes to M0_RESULTS_QUEUE

Step 7: M1 inference:
  VLM output: "No unusual activity detected. One security guard walks
               through the entrance at 14:30:45."

Step 8: M2 event detection:
  Events: [{ type: "authorized_personnel", timestamp: "14:30:45",
             description: "Security guard enters" }]
  Embedding indexed in Qdrant
  Cache chunk deleted
  createLiveEvent saved to DB

Result: Security team can see real-time event feed and search
        "unauthorized access" across all cameras using semantic search.
```
