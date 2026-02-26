# M1 Service — VLM Inference

**Service Name**: `sightapi_m1`
**Port**: `7081`
**Role in Pipeline**: Stage 2 — AI Visual Understanding
**Queue In**: `M0_RESULTS_QUEUE`
**Queue Out**: `M1_INFERENCE_QUEUE`

---

## 1. Overview

The M1 service is the **AI brain of the video analysis pipeline**. It receives individual video chunks from M0 and runs them through a **Vision Language Model (VLM)** — a multimodal AI capable of "watching" a video and producing a natural-language description of what it sees.

### Why M1 Exists

Raw video data is not directly usable for event detection, search, or reporting. The M1 service bridges the gap between raw video chunks and structured intelligence by:

- Feeding each video chunk to a VLM and asking "what is happening in this video?"
- Generating rich, descriptive summaries of visual content
- Storing those summaries in cloud storage for downstream use
- Forwarding chunk + summary data to M2 for event detection and indexing

### Problems It Solves

| Problem | Solution |
|---------|----------|
| Raw video cannot be searched semantically | VLM produces text summaries |
| Each video segment needs AI analysis | Per-chunk inference pipeline |
| LLM calls need observability | Langfuse integration for tracing |
| Different LLM providers for different deployments | Provider-agnostic LLM client |
| VLM output needs to be persisted | Summary JSON uploaded to blob storage |

---

## 2. Service Responsibilities

- Consume chunk processing messages from `M0_RESULTS_QUEUE`
- Download or reference the video chunk in cloud storage
- Invoke the configured **Vision Language Model** with the video URL and system/user prompts
- Receive the VLM's natural-language output (summary/description)
- Upload the VLM output as a JSON file to blob storage
- Save a `ChunkOutput` record to the database linking the chunk to its summary
- Update the chunk's processing status to `m1_completed`
- Forward an enriched message (with summary path) to `M1_INFERENCE_QUEUE` for M2
- Route failures to `M1_DEADLETTER_QUEUE`

---

## 3. Detailed Execution Flow

```
Step 1:  M0 publishes chunk message to M0_RESULTS_QUEUE
Step 2:  M1Consumer picks up the message (semaphore-controlled concurrency)
Step 3:  JSON payload parsed to ChunkProcessingRequest Pydantic model
Step 4:  Check if VLM inference is configured in the message (inference_config)
Step 5:  process_vlm_inference() called:
           a. Build message list: SystemMessage + HumanMessage (with video_url content)
           b. Call core/llm_client.py → chat(messages, provider, model)
           c. LLM provider (Bedrock / Azure OpenAI / Gemini) processes video
           d. VLM returns text summary of the video chunk
Step 6:  Upload summary as JSON to blob storage
           Path: <destination_prefix>/<chunk_hash>/summary.json
Step 7:  GraphQL mutation: createChunkOutput (links chunk_id → summary_path)
Step 8:  GraphQL mutation: updateChunkStatus(chunk_id, m1_completed)
Step 9:  Build enriched result message (original + summary path added)
Step 10: Publish to M1_INFERENCE_QUEUE for M2
Step 11: On error → update chunk status to failed_m1 + route to DLQ
```

---

## 4. Architecture Flow Diagram

```
RabbitMQ: M0_RESULTS_QUEUE
         |
         | M1Consumer.process_message()
         v
M1 Consumer Service
         |
         +---> Parse ChunkProcessingRequest
         |
         +---> Build VLM Message:
         |       [SystemMessage: "You are a video analysis assistant..."]
         |       [HumanMessage: { type:text, text:"Describe this video" },
         |                      { type:video_url, url:"s3://..." }]
         |
         +---> LLM Client → VLM Provider
         |       (AWS Bedrock / Azure OpenAI / Google Gemini)
         |
         +---> VLM Response: "The video shows a person entering through
         |                    the main gate at 14:32 carrying a package..."
         |
         +---> Upload summary.json → Cloud Storage
         |       s3://bucket/chunks/<hash>/summary.json
         |
         +---> GraphQL: createChunkOutput + updateChunkStatus(m1_completed)
         |
         +---> Publish enriched message to M1_INFERENCE_QUEUE
         v
RabbitMQ: M1_INFERENCE_QUEUE
         |
         v
M2 Service (Event Detection)
```

---

## 5. Connections and Integrations

### RabbitMQ

| Queue | Direction | Purpose |
|-------|-----------|---------|
| `M0_RESULTS_QUEUE` | Inbound | Receives chunk messages from M0 |
| `M1_INFERENCE_QUEUE` | Outbound | Forwards chunk + summary to M2 |
| `M1_DEADLETTER_QUEUE` | Outbound (on error) | Failed inference messages |

### LLM Providers (via `core/llm_client.py`)

M1 supports multiple VLM backends, selected via `LLM_PROVIDERS_TO_INIT` config:

| Provider | Model Examples | Notes |
|----------|---------------|-------|
| **AWS Bedrock** | `amazon.nova-pro-v1` | Multimodal video support |
| **Azure OpenAI** | `gpt-4o` | Vision-enabled GPT |
| **Google Gemini** | `gemini-1.5-pro` | Native video understanding |

- **Data sent**: System prompt + user prompt + video URL (as `video_url` content block)
- **Data received**: Text summary string from VLM

### Cloud Storage (S3 / Azure Blob)

| Operation | Direction | Purpose |
|-----------|-----------|---------|
| `upload()` | Outbound | Store VLM summary JSON |

### PostgreSQL (via GraphQL)

- **`createChunkOutput`** — creates a DB record linking chunk_id to the summary storage path
- **`updateChunkStatus`** — sets chunk status to `m1_completed`
- **`updateChunkStatus(failed_m1)`** — on error path

### Langfuse (Observability)

- Every VLM call is wrapped with Langfuse callbacks
- Traces capture: input messages, output, model, token counts, latency
- Session ID and user ID attached per trace for audit purposes

---

## 6. Data Flow

```
Input (from M0_RESULTS_QUEUE):
{
  "process_id": 101,
  "chunk_id": 1,
  "chunk_hash": "d4f3a9...",
  "cloud_stream_path": "s3://spectra-videos/processed/101/chunk_001.mp4",
  "cache_path": "s3://spectra-cache/temp/101/chunk_001.mp4",
  "start_time": 0.0,
  "end_time": 60.0,
  "inference_modality": "batch",
  "inference_config": {
    "vlm_provider": "bedrock",
    "vlm_model": "amazon.nova-pro-v1",
    "system_prompt_key": "video_analysis_v1",
    "user_prompt": "Describe all events and activities in this video segment."
  }
}
         |
         v
VLM Message Construction:
  messages = [
    SystemMessage("You are a security video analysis assistant. ..."),
    HumanMessage([
      { "type": "text", "text": "Describe all events and activities in this video segment." },
      { "type": "video_url", "video_url": { "url": "s3://.../chunk_001.mp4" } }
    ])
  ]
         |
         v
VLM Inference (AWS Bedrock / Azure OpenAI / Google Gemini):
  response = "Between 0:00 and 1:00, a person wearing a blue jacket
              enters through the main entrance. At 0:35, they stop at
              the reception desk. No unusual activity detected."
         |
         v
Summary JSON Uploaded to Storage:
  s3://spectra-videos/processed/101/chunk_001_summary.json
  Content: { "chunk_id": 1, "summary": "Between 0:00 and 1:00, ..." }
         |
         v
GraphQL Mutations:
  createChunkOutput({ chunk_id: 1, summary_path: "s3://...summary.json" })
  updateChunkStatus(chunk_id: 1, status: m1_completed)
         |
         v
Output (published to M1_INFERENCE_QUEUE):
{
  ... (all original fields from input) ...
  "inference_result": {
    "vlm_summary": "Between 0:00 and 1:00, a person wearing a blue jacket..."
  }
}
```

---

## 7. Code-Level Explanation

### Entry Point: `main.py`

```
sightapi_m1/src/main.py
```

Startup sequence:
1. `setup_environment()` — load secrets
2. `setup_storage()` — blob client
3. `setup_http_client()` — async HTTP
4. `setup_llm()` — initializes all configured LLM providers
5. `initialize_observability()` — Langfuse callbacks
6. `asyncio.create_task(consumer.run())` — start consuming

Registers `/health` endpoint that reports `{"status": "healthy", "llm_provider": "bedrock"}`.

### Consumer: `services/consumer_service.py`

**Class**: `M1Consumer(BaseConsumer)`

Key method: `process_message(message: aio_pika.IncomingMessage)`

```python
async def process_message(self, message):
    payload = json.loads(message.body)
    chunk_request = ChunkProcessingRequest(**payload)

    if chunk_request.inference_config.vlm_provider:
        await process_vlm_inference(chunk_request)

    result_payload = { ...chunk_request fields..., "inference_result": { "vlm_summary": summary } }
    await publish_to_m1_inference_queue(result_payload)
```

### VLM Inference: `services/vlm_inference_service.py`

**Function**: `run_vlm_inference(messages, config)`

- Calls `core/llm_client.py → chat(messages, provider, model, callbacks=[langfuse_handler])`
- Returns raw text string from VLM

**Function**: `process_vlm_inference(chunk_request: ChunkProcessingRequest)`

1. Build `SystemMessage` from prompt config
2. Build `HumanMessage` with text + `video_url` content blocks
3. Call `run_vlm_inference()`
4. Serialize response to JSON
5. `storage_client.upload(container, key, json_bytes)` — save summary
6. GraphQL: `createChunkOutput` + `updateChunkStatus(m1_completed)`
7. Return summary string

### Schema: `schemas/inference.py`

**Model**: `ChunkProcessingRequest`

```python
class ChunkProcessingRequest(BaseModel):
    process_id: int
    chunk_id: int
    chunk_hash: str
    cloud_stream_path: str
    cache_path: str
    start_time: float
    end_time: float
    inference_modality: str          # "batch" | "live"
    inference_config: InferenceConfig
    destination_prefix: str
```

**Model**: `InferenceConfig`

```python
class InferenceConfig(BaseModel):
    vlm_provider: Optional[str]      # "bedrock" | "azure_openai" | "google"
    vlm_model: Optional[str]
    system_prompt_key: Optional[str]
    user_prompt: Optional[str]
```

---

## 8. Sequence Flow

```
M0 Service
  │
  │  publish_to_m0_results_queue({ chunk_001, s3 path, inference_config })
  ↓
RabbitMQ [M0_RESULTS_QUEUE]
  │
  │  M1Consumer.process_message()
  ↓
M1 Consumer
  │
  │  1. Parse ChunkProcessingRequest
  │  2. build_messages(system_prompt, user_prompt, video_url)
  │  3. llm_client.chat(messages, "bedrock", "amazon.nova-pro-v1")
  ↓
AWS Bedrock / Azure OpenAI / Google Gemini
  │
  │  Returns: "A person enters the gate at 0:05..."
  ↓
M1 Consumer (continued)
  │
  │  4. serialize to JSON
  │  5. storage_client.upload(summary.json)
  │  6. GraphQL: createChunkOutput
  │  7. GraphQL: updateChunkStatus(m1_completed)
  │  8. publish_to_m1_inference_queue({ ...original + vlm_summary })
  ↓
RabbitMQ [M1_INFERENCE_QUEUE]  →  M2 Service
```

---

## 9. Dependency Explanation

| Dependency | Purpose |
|------------|---------|
| `FastAPI` | REST API server |
| `aio-pika` | Async RabbitMQ consumer |
| `langchain-core` | Message types (SystemMessage, HumanMessage) |
| `langchain-aws` / `langchain-openai` / `langchain-google-genai` | VLM provider integrations |
| `langfuse` | LLM call observability and tracing |
| `core/llm_client.py` | Provider-agnostic LLM interface |
| `core/rmq_client.py` | Queue publish helpers |
| `core/rmq_consumer.py` | `BaseConsumer` abstract class |
| `core/storage_client.py` | Cloud blob upload |
| `core/http_client.py` | GraphQL mutations |
| `core/observability.py` | Langfuse callback setup |
| `core/config.py` | Centralized configuration |

---

## 10. Error Handling Flow

```
Error in process_vlm_inference()
         |
         +---> Log: f"M1 inference failed for chunk_id={chunk_id}: {error}"
         |
         +---> GraphQL: updateChunkStatus(chunk_id, failed_m1)
         |
         +---> Publish original message to M1_DEADLETTER_QUEUE
         |
         +---> BaseConsumer: nack message (not re-queued)
```

**LLM Timeout**: If LLM provider times out, exception propagates → DLQ route.
**Upload Failure**: If summary.json upload fails, exception propagates → chunk status = `failed_m1`.
**Invalid Payload**: `ChunkProcessingRequest(**payload)` raises `ValidationError` → DLQ route.
**Provider Not Configured**: If `vlm_provider` is None, VLM step is skipped; chunk forwarded as-is.

---

## 11. Deployment

- **Container**: `sightapi_m1/deployment/docker/Dockerfile`
- **Port**: `7081`
- **GPU**: Not required — VLM inference is remote (Bedrock/Azure/Gemini APIs)
- **Key env vars**:
  - `LLM_PROVIDERS_TO_INIT=bedrock` (or `azure_openai`, `google`)
  - `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`
  - `AWS_*` credentials (for Bedrock + S3)
- **Scaling**: Horizontal scaling supported — multiple M1 instances consume from same queue

---

## 12. Real Example Flow

**Scenario**: Chunk 1 of the security camera batch job (process 101) arrives for VLM analysis.

```
Message received from M0_RESULTS_QUEUE:
{
  "process_id": 101,
  "chunk_id": 1,
  "chunk_hash": "d4f3a9c1",
  "cloud_stream_path": "s3://spectra-videos/processed/101/chunk_001.mp4",
  "cache_path": "s3://spectra-cache/temp/101/chunk_001.mp4",
  "start_time": 0.0,
  "end_time": 60.0,
  "inference_config": {
    "vlm_provider": "bedrock",
    "vlm_model": "amazon.nova-pro-v1",
    "system_prompt_key": "security_analysis_v2",
    "user_prompt": "Identify and describe all people, objects, and events."
  }
}

Step 1: Build messages:
  SystemMessage("You are an expert security video analyst...")
  HumanMessage([
    { "type": "text", "text": "Identify and describe all people, objects, and events." },
    { "type": "video_url", "video_url": { "url": "s3://.../chunk_001.mp4" } }
  ])

Step 2: AWS Bedrock (amazon.nova-pro-v1) processes video:
  Response: "At 0:05, one adult male in blue workwear enters from the north
             entrance. At 0:42, a forklift moves through the loading bay.
             No unidentified individuals detected."

Step 3: Upload to s3://spectra-videos/processed/101/chunk_001_summary.json

Step 4: DB mutations:
  createChunkOutput({ chunk_id:1, summary_path:"s3://.../chunk_001_summary.json" })
  updateChunkStatus(chunk_id:1, status:m1_completed)

Step 5: Publish to M1_INFERENCE_QUEUE:
{
  ... (original fields) ...,
  "inference_result": {
    "vlm_summary": "At 0:05, one adult male in blue workwear enters..."
  }
}

M2 receives this message for event detection and embedding.
```
