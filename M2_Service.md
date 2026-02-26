# M2 Service — Event Detection, Embedding & Indexing

**Service Name**: `sightapi_m2`<br>
**Port**: `7082`<br>
**Role in Pipeline**: Stage 3 — Intelligence Extraction & Search Indexing<br>
**Queue In**: `M1_INFERENCE_QUEUE`<br>
**Queue Out**: None (terminal stage of the batch/live pipeline)<br>

---

## 1. Overview

The M2 service is the **final intelligence processing stage** of the video analysis pipeline. It receives enriched chunk messages from M1 (which already include VLM-generated text summaries) and performs three parallel operations:

1. **Event Detection** — Uses an LLM to extract structured events from the VLM summary
2. **Embedding & Indexing** — Converts the summary into a vector embedding and stores it in Qdrant for semantic search
3. **Cache Cleanup** — Deletes the temporary video chunk from cache storage

Once all chunks of a process are marked `completed`, the entire job is considered done.

### Why M2 Exists

After M1 generates raw text descriptions of video chunks, that text still needs to be:
- **Parsed into structured events** (who, what, when, where) for reporting and alerts
- **Vectorized** so users can search across all video content using natural language
- **Cleaned up** to free cache storage consumed by temporary chunks

M2 handles all three concerns in a single, parallelized stage.

### Problems It Solves

| Problem | Solution |
|---------|----------|
| VLM summaries are unstructured text | LLM event detection extracts structured JSON |
| Video content is not searchable | Vector embeddings + Qdrant semantic index |
| Cache storage fills up with temp chunks | Automatic deletion after inference |
| Three independent tasks waste time if sequential | `asyncio.gather()` runs all three in parallel |
| Chunk completion needs tracking | Status updated to `completed` only after all tasks succeed |

---

## 2. Service Responsibilities

- Consume enriched chunk messages from `M1_INFERENCE_QUEUE`
- **Event Detection**: call an LLM with the VLM summary and extract structured event records
- **Save Events**: persist detected events to the database (batch or live event tables)
- **Generate Embedding**: convert VLM summary text into a dense vector using an embedding model
- **Index in Qdrant**: store the vector with chunk metadata for semantic similarity search
- **Delete Cache Chunk**: remove temporary video from cache storage
- **Update Status**: mark the chunk as `completed`
- Determine if all chunks of a process are complete and update process status accordingly
- Route failed messages to `M2_DEADLETTER_QUEUE`

---

## 3. Detailed Execution Flow

```
Step 1:  M1 publishes enriched chunk message to M1_INFERENCE_QUEUE
Step 2:  M2Consumer picks up the message
Step 3:  JSON payload parsed to EventDetectionMessage Pydantic model
Step 4:  Three async tasks launched in PARALLEL via asyncio.gather():

         Task A: Event Detection
           a. Extract VLM summary from inference_result
           b. Build LLM prompt: "Extract structured events from: <summary>"
           c. Call LLM (same provider as M1, or separately configured)
           d. Parse response into list of event dicts
           e. For each event → GraphQL mutation (createBatchEvent / createLiveEvent)

         Task B: Embedding & Indexing
           b1. Call embedding model with VLM summary text
           b2. Receive dense vector (e.g. 1536 dimensions)
           b3. Build Qdrant document with metadata:
               { chunk_id, process_id, start_time, end_time, summary, vector }
           b4. Store in Qdrant collection via search_client.index_document()
           b5. Returns doc_id

         Task C: Cache Deletion
           c1. Extract cache_path from message
           c2. storage_client.delete(cache_container, chunk_key)

Step 5:  All three tasks complete (or one fails → error path)
Step 6:  GraphQL: updateChunkStatus(chunk_id, completed)
Step 7:  Check if all chunks of process are complete:
           If yes → updateProcessStatus(process_id, completed)
Step 8:  On any error → log, update chunk status to failed_m2, DLQ route
```

---

## 4. Architecture Flow Diagram

```
RabbitMQ: M1_INFERENCE_QUEUE
         |
         | M2Consumer.process_message()
         v
M2 Consumer Service
         |
         +----------+-------------------+-------------------+
         |          |                   |                   |
         v          v                   v                   v
   Parse Msg   [Task A]            [Task B]            [Task C]
               Event Detection  Embedding+Indexing   Cache Deletion
               |                |                   |
               v                v                   v
         LLM Provider     Embedding Model      Cloud Storage
         (Bedrock/       (text-embedding)      delete(cache_path)
          Azure/Gemini)         |
               |                v
               v           Qdrant Vector DB
         GraphQL:          index_document()
         createEvent()          |
               |                v
               |         doc_id returned
               |
               +---------- All tasks joined --------+
                                                    |
                                                    v
                                       GraphQL: updateChunkStatus(completed)
                                                    |
                                                    v
                                       Check: all chunks complete?
                                       If yes: updateProcessStatus(completed)
```

---

## 5. Connections and Integrations

### RabbitMQ

| Queue | Direction | Purpose |
|-------|-----------|---------|
| `M1_INFERENCE_QUEUE` | Inbound | Receives chunk + VLM summary from M1 |
| `M2_DEADLETTER_QUEUE` | Outbound (on error) | Failed M2 processing messages |

### LLM Provider (Event Detection) — via `core/llm_client.py`

- Same provider system as M1 (Bedrock / Azure OpenAI / Gemini)
- Configured via `inference_config` in the message or service-level defaults
- **Data sent**: Prompt containing VLM summary + event extraction instructions
- **Data received**: Structured JSON list of events (type, description, timestamp, confidence)

### Embedding Model — via `core/llm_client.py`

- `embed(text)` or `embed_batch(texts)` function
- Returns dense float vector (e.g. 1536D for text-embedding-ada-002, 768D for Gemini)
- Used to represent the semantic meaning of the VLM summary

### Qdrant Vector Database — via `core/search_client.py`

- **Collection**: Configured per `config.qdrant_collection_name`
- **Distance metric**: Cosine similarity
- **`index_document(doc)`**: Stores `{ id, vector, payload }` where payload = chunk metadata + summary
- **Why**: Enables natural-language semantic search across all analyzed video content
- Users can later query: "Show me all events where someone enters a building" → Qdrant returns top-K similar chunks

### Cloud Storage (S3 / Azure Blob)

| Operation | Direction | Purpose |
|-----------|-----------|---------|
| `delete()` | Outbound | Delete temporary cache chunk after inference |

Cache TTL also handles automatic expiry as a backup mechanism.

### PostgreSQL (via GraphQL)

- **`createBatchEvent`** / **`createLiveEvent`** — persist detected events for batch/live jobs
- **`updateChunkStatus(completed)`** — final chunk status update
- **`updateBatchVideoProcessingStatus(completed)`** — marks whole batch process done
- **`updateCamVideoProcessingStatus(completed)`** — marks live process done

### Langfuse (Observability)

- All LLM calls (event detection + embedding) traced via Langfuse callbacks

---

## 6. Data Flow

```
Input (from M1_INFERENCE_QUEUE):
{
  "process_id": 101,
  "chunk_id": 1,
  "chunk_hash": "d4f3a9c1",
  "cloud_stream_path": "s3://spectra-videos/processed/101/chunk_001.mp4",
  "cache_path": "s3://spectra-cache/temp/101/chunk_001.mp4",
  "start_time": 0.0,
  "end_time": 60.0,
  "inference_modality": "batch",
  "inference_result": {
    "vlm_summary": "At 0:05, one adult male in blue workwear enters..."
  }
}
         |
         v (parallel execution)

[Task A: Event Detection]
  LLM Input:
    "Extract structured events from the following video summary:
     At 0:05, one adult male in blue workwear enters..."
  LLM Output:
    [
      { "type": "person_entry", "description": "Adult male enters north entrance",
        "timestamp": "0:05", "confidence": 0.95 },
      { "type": "vehicle_movement", "description": "Forklift moves through loading bay",
        "timestamp": "0:42", "confidence": 0.87 }
    ]
  GraphQL: createBatchEvent × 2

[Task B: Embedding + Indexing]
  Input text: "At 0:05, one adult male in blue workwear enters..."
  Embedding: [0.023, -0.14, 0.88, ..., 0.002]  (1536 dimensions)
  Qdrant Document:
    {
      id: "d4f3a9c1",
      vector: [0.023, -0.14, ...],
      payload: {
        chunk_id: 1, process_id: 101,
        start_time: 0.0, end_time: 60.0,
        summary: "At 0:05, one adult male..."
      }
    }
  → Indexed in Qdrant

[Task C: Cache Deletion]
  storage_client.delete("spectra-cache", "temp/101/chunk_001.mp4")
  → Temporary chunk removed

         |
         v (all tasks complete)

GraphQL: updateChunkStatus(chunk_id:1, completed)
         |
         v
(If last chunk): updateBatchVideoProcessingStatus(101, completed)

Output: None (M2 is the terminal stage — results are in DB + Qdrant)
```

---

## 7. Code-Level Explanation

### Entry Point: `main.py`

```
sightapi_m2/src/main.py
```

Startup sequence:
1. `setup_environment()`
2. `setup_storage()` — S3/Blob
3. `setup_http_client()` — for GraphQL
4. `setup_search()` — initialize Qdrant client, create collection if missing
5. `connect_rabbitmq()` → `declare_application_queues()`
6. `setup_llm()` — LLM + embedding models
7. `initialize_observability()` — Langfuse
8. `asyncio.create_task(consumer.run())`

`/health` endpoint reports status of: LLM, storage, search (Qdrant), RabbitMQ.

### Consumer & Orchestrator: `services/consumer_service.py`

**Class**: `M2Consumer(BaseConsumer)`

Key method: `process_message(message)`

```python
async def process_message(self, message):
    payload = json.loads(message.body)
    event_message = EventDetectionMessage(**payload)

    tasks = [
        event_detection_service.process_event_detection(event_message),
        _process_embedding_and_indexing(event_message),
        cache_deletion_service.delete_cache_chunk(event_message.cache_path)
    ]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # Check for exceptions in results
    for result in results:
        if isinstance(result, Exception):
            raise result

    await update_chunk_status(event_message.chunk_id, "completed")
    await check_and_update_process_completion(event_message.process_id)
```

**Internal helper**: `_process_embedding_and_indexing(event_message)`

```python
async def _process_embedding_and_indexing(event_message):
    embedding = await embedding_service.generate_embedding(event_message.inference_result.vlm_summary)
    doc_id = await search_client.index_document({
        "id": event_message.chunk_hash,
        "vector": embedding,
        "payload": { ... chunk metadata ... }
    })
    return doc_id
```

### Event Detection: `services/event_detection_service.py`

**Function**: `process_event_detection(event_message: EventDetectionMessage)`

- Builds an LLM prompt containing the VLM summary
- Calls `llm_client.chat()` or `llm_client.structured()` to get events
- Parses the JSON response into event objects
- Calls `metadata_service.save_events(events, event_message)` for DB persistence

### Embedding: `services/embedding_service.py`

**Function**: `generate_embedding(text: str) → List[float]`

- Calls `llm_client.embed(text)` from `core/llm_client.py`
- Returns vector representation of the VLM summary

### Metadata Service: `services/metadata_service.py`

**Function**: `save_events(events: List[dict], event_message)`

- Determines modality from `event_message.inference_modality` (batch vs live)
- Calls `createBatchEvent` or `createLiveEvent` GraphQL mutation per event

### Cache Deletion: `services/cache_deletion_service.py`

**Function**: `delete_cache_chunk(cache_path: str)`

- Parses container name and key from the cache_path URL
- Calls `storage_client.delete(container, key)`

### Chunk Output: `services/chunk_output_service.py`

**Function**: `update_chunk_status(chunk_id, status)`

- Calls `updateChunkStatus` GraphQL mutation
- Called for both success (`completed`) and failure (`failed_m2`)

### Schema: `schemas/event_detection.py`

**Model**: `EventDetectionMessage`

```python
class EventDetectionMessage(BaseModel):
    process_id: int
    chunk_id: int
    chunk_hash: str
    cloud_stream_path: str
    cache_path: str
    start_time: float
    end_time: float
    inference_modality: str
    inference_result: InferenceResult
    destination_prefix: str

class InferenceResult(BaseModel):
    vlm_summary: str
```

---

## 8. Sequence Flow

```
M1 Service
  │
  │  publish_to_m1_inference_queue({ chunk_001, vlm_summary: "..." })
  ↓
RabbitMQ [M1_INFERENCE_QUEUE]
  │
  │  M2Consumer.process_message()
  ↓
M2 Consumer
  │
  │  asyncio.gather() — 3 parallel tasks:
  │
  ├──[Task A]─────────────────────────────────────────────────────────
  │    event_detection_service.process_event_detection()
  │      → LLM: "Extract events from: At 0:05, one adult male..."
  │      → Bedrock returns: [{ type: person_entry, ... }]
  │      → GraphQL: createBatchEvent × N
  │
  ├──[Task B]─────────────────────────────────────────────────────────
  │    generate_embedding("At 0:05, one adult male...")
  │      → embedding_model.embed()
  │      → [0.023, -0.14, 0.88, ...]
  │      → search_client.index_document({ id, vector, payload })
  │      → Qdrant stores document
  │
  └──[Task C]─────────────────────────────────────────────────────────
       cache_deletion_service.delete_cache_chunk(cache_path)
         → storage_client.delete("spectra-cache", "temp/101/chunk_001.mp4")
  │
  │  All tasks complete
  │
  │  updateChunkStatus(chunk_id:1, completed)
  │  Check: is this the last chunk?
  │  Yes → updateBatchVideoProcessingStatus(101, completed)
  ↓
Pipeline Complete — Results stored in PostgreSQL + Qdrant
```

---

## 9. Dependency Explanation

| Dependency | Purpose |
|------------|---------|
| `FastAPI` | REST API server |
| `aio-pika` | Async RabbitMQ consumer |
| `qdrant-client` | Async Qdrant vector database client |
| `langchain-*` | LLM provider integrations |
| `langfuse` | LLM observability tracing |
| `core/llm_client.py` | Event detection LLM + embedding model |
| `core/search_client.py` | Qdrant index/search operations |
| `core/storage_client.py` | Cache chunk deletion |
| `core/rmq_client.py` | Queue connection |
| `core/rmq_consumer.py` | `BaseConsumer` abstract class |
| `core/http_client.py` | GraphQL mutations |
| `core/config.py` | Centralized configuration |
| `core/observability.py` | Langfuse setup |

---

## 10. Error Handling Flow

```
Error in any of the 3 parallel tasks
         |
         +---> asyncio.gather(return_exceptions=True) captures it
         |
         +---> Exception detected in results list
         |
         +---> Log: f"M2 failed for chunk_id={chunk_id}: {error}"
         |
         +---> GraphQL: updateChunkStatus(chunk_id, failed_m2)
         |
         +---> Publish original message to M2_DEADLETTER_QUEUE
         |
         +---> BaseConsumer: nack message
```

**Qdrant failure**: If vector indexing fails, event detection still succeeds; only embedding is missing. Both failures are treated as full chunk failure to ensure data consistency.
**Embedding failure**: Prevents Qdrant indexing → DLQ route.
**Event detection failure**: Events not persisted → DLQ route.
**Cache deletion failure**: Treated as non-fatal in some configurations (TTL handles cleanup).
**Partial completion**: No partial success — if any task fails, chunk status = `failed_m2`.

---

## 11. Deployment

- **Container**: `sightapi_m2/deployment/docker/Dockerfile`
- **Port**: `7082`
- **Dependencies**: Qdrant must be reachable at startup (`setup_search()` creates collection)
- **Key env vars**:
  - `QDRANT_HOST`, `QDRANT_PORT`, `QDRANT_COLLECTION_NAME`
  - `LLM_PROVIDERS_TO_INIT`
  - `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`
- **Scaling**: Multiple M2 instances can consume from `M1_INFERENCE_QUEUE` concurrently
- **Qdrant**: Must be deployed as a persistent service with volume mounts for vector storage

---

## 12. Real Example Flow

**Scenario**: Chunk 1 from process 101 arrives at M2 with VLM summary from M1.

```
Message received from M1_INFERENCE_QUEUE:
{
  "process_id": 101,
  "chunk_id": 1,
  "chunk_hash": "d4f3a9c1",
  "cache_path": "s3://spectra-cache/temp/101/chunk_001.mp4",
  "start_time": 0.0,
  "end_time": 60.0,
  "inference_modality": "batch",
  "inference_result": {
    "vlm_summary": "At 0:05, one adult male in blue workwear enters from the
                    north entrance. At 0:42, a forklift moves through the loading bay."
  }
}

asyncio.gather() launches 3 tasks simultaneously:

[Task A — Event Detection]:
  LLM prompt: "Extract structured security events from: ..."
  LLM response:
    [
      { "type": "person_entry", "timestamp": "0:05", "confidence": 0.95,
        "description": "Adult male in blue workwear enters north entrance" },
      { "type": "vehicle_movement", "timestamp": "0:42", "confidence": 0.87,
        "description": "Forklift traverses loading bay" }
    ]
  → createBatchEvent({ process_id:101, chunk_id:1, type:"person_entry", ... })
  → createBatchEvent({ process_id:101, chunk_id:1, type:"vehicle_movement", ... })

[Task B — Embedding + Indexing]:
  text = "At 0:05, one adult male in blue workwear..."
  embedding = embed(text)  → [0.023, -0.14, 0.88, ...]  (1536-dim)
  Qdrant: index_document({
    id: "d4f3a9c1",
    vector: [0.023, -0.14, ...],
    payload: { chunk_id:1, process_id:101, start:0.0, end:60.0, summary:"..." }
  })

[Task C — Cache Deletion]:
  storage_client.delete("spectra-cache", "temp/101/chunk_001.mp4")
  → Chunk removed from cache

All tasks complete:
  updateChunkStatus(chunk_id:1, completed)
  Check: 5 chunks total, 5 completed → updateBatchVideoProcessingStatus(101, completed)

Users can now search: "show me forklift activity"
→ Qdrant finds chunk_001 via vector similarity
→ Returns { chunk_id:1, summary:"...", timestamp:0:42 }
```
