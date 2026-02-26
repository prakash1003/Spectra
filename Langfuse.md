# Langfuse — Role and Usage Across the Codebase

---

## What is Langfuse Here

Langfuse serves **two completely separate roles** in this system. They are independent of each other:

1. **Prompt Management** — Langfuse is the single source of truth for all LLM prompt text
2. **LLM Call Tracing** — Every LLM call is automatically traced (inputs, outputs, tokens, latency)

---

## Role 1: Prompt Management

### The Problem It Solves

LLM prompts (the instructions you send to GPT/Gemini/Bedrock) are not hardcoded in the Python files. They are stored in Langfuse with versioning. This allows:
- Prompts to be updated without redeploying the service
- Every prompt version to be tracked and auditable
- Different prompt versions to be assigned to different org/cohort configs

### How the Storage Works

The database `PromptTbl` does **not** store the actual prompt text. It stores a reference key pointing to Langfuse:

```
DB Column: ref_prompt_key = "system/_/vlm_inference/system:3"
                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^  ^
                              Langfuse prompt name          version number
```

At runtime, when a prompt is needed:
1. Fetch `ref_prompt_key` from DB → `"system/_/vlm_inference/system:3"`
2. Split on `:` → name = `"system/_/vlm_inference/system"`, version = `3`
3. Call Langfuse API → `GET /api/public/v2/prompts/system%2F_%2Fvlm_inference%2Fsystem?version=3`
4. Cache the result in Redis with TTL
5. Return the prompt text to the caller

```
DB (ref_prompt_key)
       |
       | rsplit(':', 1)
       v
prompt_handler.get_prompt(name, version)
       |
       +---> Redis cache hit? → return immediately
       |
       +---> Redis miss → Langfuse API call
                               |
                               v
                         prompt text returned
                               |
                               v
                         Cache in Redis (TTL)
                               |
                               v
                         Return to caller
```

### Prompts Stored in Langfuse

These are created by the seed script `backend/src/seed/ingest_prompts_to_langfuse.py`:

| Langfuse Prompt Name | Used By | Purpose |
|---------------------|---------|---------|
| `system/_/vlm_inference/system` | M1 Service | System prompt sent to VLM before video analysis |
| `system/_/vlm_inference/user` | M1 Service | User prompt instructing VLM what to extract |
| `system/_/event_detection/system` | M2 Service | System prompt for event detection LLM |
| `system/_/event_detection/user` | M2 Service | User prompt template with variables (`{{vlm_summary}}`, `{{events_list}}`) |
| `system/_/event_detection/events_list` | M2 Service | Comma-separated list of event types to detect |
| `system/_/chat_agent/system` | Backend Chat Agent | System prompt for the Spectra conversational AI |

### Where Prompt Fetching Happens in Code

**`core/prompt_handler.py`** — the only place that talks to the Langfuse API for prompts

```python
# Fetches prompt by name + version with Redis caching
await prompt_handler.get_prompt("system/_/vlm_inference/system", "3")
```

**`backend/src/services/org_process_resolvers/base.py`** — called when building inference payloads

```python
# _resolve_prompt_content() is called by VLM and event detection resolvers
ref_prompt_key = prompt_record.ref_prompt_key         # e.g. "system/_/vlm_inference/system:3"
base_key, version = ref_prompt_key.rsplit(':', 1)
prompt = await prompt_handler.get_prompt(base_key, version)
```

**`backend/src/services/agent/chat_agent/chat_agent.py`** — chat agent fetches its system prompt

```python
prompt_content = await prompt_handler.get_prompt(CHAT_SYSTEM_PROMPT_KEY, "latest")
# Falls back to hardcoded default if Langfuse unavailable
```

**`backend/src/api/v1/graphql/spectra_ops/prompt_tbl.py`** — when a user creates/updates a prompt via the API, it writes to Langfuse first, then DB

```python
# Create prompt in Langfuse first
lf_result = langfuse.create_prompt(
    name=ref_prompt_key,
    prompt=input.prompt_content,
    tags=langfuse_tags,
    labels=["production", "latest"]
)
# Only if Langfuse succeeds, write metadata to DB
# If DB fails, warns: "Langfuse prompt may need manual cleanup"
```

---

## Role 2: LLM Call Tracing (Observability)

### The Problem It Solves

Every time M1 calls Bedrock/Gemini to analyze a video, or M2 calls an LLM to detect events, or the backend runs the chat agent — Langfuse automatically captures:
- The exact input messages sent to the LLM
- The exact output text returned
- Which model was used
- Token counts (input + output)
- Latency
- Cost estimate
- User ID and session ID for audit trail

This is done via **LangChain's `CallbackHandler`** — it hooks into the LangChain call and reports to Langfuse automatically without changing inference code.

### How It Is Set Up

**`core/observability.py`** — initializes the `CallbackHandler` at service startup

```python
Langfuse(public_key=..., secret_key=..., base_url=...)   # registers global client
_callbacks = [CallbackHandler()]                          # LangChain callback
```

**`core/llm_client.py`** — attaches the callback to every single LLM call

```python
callbacks = get_observability_callbacks()    # gets [CallbackHandler()]

# attached to chat(), chat_stream(), structured() — all three
response = await llm_provider.chat(
    messages,
    model,
    callbacks=callbacks,           # ← Langfuse traces this call
    metadata=langfuse_metadata,    # ← user_id, session_id, tags
    run_name=trace_name
)
```

**`core/llm_client.py`** — metadata attached to every trace

```python
def _build_langfuse_metadata(user_id, session_id, tags, extra_metadata):
    metadata["langfuse_user_id"] = user_id        # who triggered it
    metadata["langfuse_session_id"] = session_id  # conversation/job thread
    metadata["langfuse_tags"] = tags              # searchable labels
```

### Where Tracing Is Active

| Service | File | What Gets Traced |
|---------|------|-----------------|
| M1 | `services/vlm_inference_service.py` | Every VLM video analysis call |
| M2 | `services/event_detection_service.py` | Every event detection LLM call |
| M2 | `services/embedding_service.py` | Every embedding generation call |
| Backend | `api/v1/endpoints/chat/router.py` | Every user chat message + LLM response |
| Backend | `services/agent/chat_agent/chat_agent.py` | Chat agent LangGraph reasoning steps |

### Chat Endpoint Trace Setup (`chat/router.py`)

The chat endpoint explicitly names its Langfuse trace:

```python
callbacks = get_observability_callbacks()

config = {
    "langfuse_session_id": thread_id,        # ties all messages in a conversation
    "langfuse_trace_name": "spectra_chat",   # searchable name in Langfuse UI
    ...
}
```

---

## Lifecycle — When Langfuse Is Initialized and Cleaned Up

| Service | Initialize | Cleanup |
|---------|-----------|---------|
| M1 | startup lifespan: `initialize_observability()` | shutdown: `cleanup_observability()` |
| M2 | startup lifespan: `initialize_observability()` | shutdown: `cleanup_observability()` |
| Backend | `prompt_handler._get_auth()` at startup (prompt role only) | — |

`cleanup_observability()` does this before shutdown:

```python
client = get_client()
client.flush()    # sends any buffered traces to Langfuse
client.shutdown()
```

This ensures no trace data is lost if the service is stopping.

---

## Configuration

All Langfuse config comes from environment variables or cloud secrets:

| Config Key | Secret Key | Purpose |
|-----------|------------|---------|
| `langfuse_enabled` | `LANGFUSE_ENABLED` | Toggle on/off (default: `true`) |
| `langfuse_public_key` | `LANGFUSE-PUBLIC-KEY` | API authentication |
| `langfuse_secret_key` | `LANGFUSE-SECRET-KEY` | API authentication |
| `langfuse_base_url` | `LANGFUSE-BASE-URL` | Self-hosted or `https://cloud.langfuse.com` |
| `prompt_cache_ttl` | — | How long fetched prompts stay in Redis |

If `langfuse_enabled=false` or credentials are missing:
- Tracing: silently disabled, `_callbacks = []`, LLM calls still work normally
- Prompt fetching: raises `RuntimeError("Langfuse is disabled")` — this is a hard failure because prompts are required for inference

---

## Prompt Naming Convention

All prompt keys follow this pattern:

```
{scope}/{user_role_cohort_hash}/{org_process}/{prompt_type}
```

Examples:
```
system/_/vlm_inference/system      ← system-level, all users, VLM process, system role
system/_/event_detection/user      ← system-level, all users, event detection, user role
system/_/chat_agent/system         ← system-level, all users, chat agent, system role
```

The `_` in the second segment means "applies to all cohorts/roles" (not org-specific).

The `ref_prompt_key` stored in the DB always appends the version:
```
system/_/vlm_inference/system:3    ← pinned to version 3
```

When the chat agent fetches its prompt, it uses `"latest"` instead of a pinned version:
```python
prompt_handler.get_prompt(CHAT_SYSTEM_PROMPT_KEY, "latest")
```

---

## Seed Script — Initial Prompt Upload

`backend/src/seed/ingest_prompts_to_langfuse.py`

This is a one-time script run during environment setup. It reads prompt content from `tests/example_payload-aws.json` and pushes 6 prompts to Langfuse:

```bash
python backend/src/seed/ingest_prompts_to_langfuse.py
```

After running, each prompt gets a version number from Langfuse (e.g., version `1`). These version numbers are then stored in the DB `PromptTbl.ref_prompt_key` column as `<name>:1`.

---

## Summary

```
Langfuse
   │
   ├── PROMPT STORE (source of truth for all LLM instructions)
   │      │
   │      ├── prompt_handler.py → Langfuse API → Redis cache → caller
   │      ├── org_process_resolvers/base.py → uses prompt_handler
   │      ├── chat_agent.py → uses prompt_handler
   │      └── prompt_tbl.py GraphQL → writes to Langfuse on create/update
   │
   └── TRACING (automatic capture of every LLM call)
          │
          ├── observability.py → initializes CallbackHandler
          ├── llm_client.py → attaches callbacks to chat/stream/structured/embed
          ├── M1 startup → initialize_observability()
          ├── M2 startup → initialize_observability()
          └── chat/router.py → tags traces with session_id + trace_name
```
