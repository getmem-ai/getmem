# getmem-ai

Persistent memory API for AI agents. Two API calls — your agent remembers everything.

[![PyPI version](https://img.shields.io/pypi/v/getmem-ai.svg)](https://pypi.org/project/getmem-ai/)
[![Python](https://img.shields.io/pypi/pyversions/getmem-ai.svg)](https://pypi.org/project/getmem-ai/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## Install

```bash
pip install getmem-ai
```

## Quick Start

```python
import getmem_ai as getmem

mem = getmem.init("gm_live_...")

# Before each LLM call — get relevant context
result = mem.get("user_123", query=user_message)
context = result["context"]  # inject into system prompt

# After each turn — save conversation to memory
mem.ingest("user_123", messages=[
    {"role": "user", "content": user_message},
    {"role": "assistant", "content": reply},
])
```

## Usage with OpenAI

```python
import getmem_ai as getmem
from openai import OpenAI

mem = getmem.init("gm_live_...")
openai = OpenAI(api_key="sk-...")

user_id = "user_123"
user_message = "Tell me something about my preferences"

# 1. Get memory context
result = mem.get(user_id, query=user_message)
context = result["context"]

# 2. Call LLM with memory injected
completion = openai.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": f"You are a helpful assistant.\n\n## Memory\n{context}"},
        {"role": "user", "content": user_message},
    ]
)
reply = completion.choices[0].message.content

# 3. Save conversation back to memory
mem.ingest_conversation(user_id, user_message, reply)
```

## API Reference

### `getmem.init(api_key, base_url=None, timeout=30, max_retries=3)`

Returns a `GetMem` client instance.

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `api_key` | `str` | *required* | Your getmem API key |
| `base_url` | `str` | `https://memory.getmem.ai` | API base URL |
| `timeout` | `int` | `30` | Request timeout in seconds |
| `max_retries` | `int` | `3` | Max retries for retryable errors |

### `mem.get(user_id, query)` → dict

Retrieve relevant memory context. Returns:

```python
{
    "context": str,      # ready for LLM system prompt
    "memories": list,    # individual items with relevance scores
    "meta": dict,        # timings, token count, entities
}
```

### `mem.ingest(user_id, messages, session_id=None)` → dict

Ingest raw messages into memory. Returns immediately — extraction is async.

```python
mem.ingest("user_123", [
    {"role": "user", "content": "I love hiking", "timestamp": "2026-04-17T10:00:00Z"},
    {"role": "assistant", "content": "Great hobby!", "timestamp": "2026-04-17T10:00:01Z"},
])
```

### `mem.ingest_conversation(user_id, user_message, assistant_message, session_id=None)` → dict

Convenience method for a single exchange.

```python
mem.ingest_conversation("user_123", user_msg, reply)
```

### `mem.health()` → dict

Health check. Returns service status and component health.

### Error Handling

```python
from getmem_ai import (
    UnauthorizedError,
    QuotaExceededError,
    RateLimitedError,
    ValidationError,
    InternalError,
    ServiceUnavailableError,
    ConnectionError,
    TimeoutError,
)

try:
    result = mem.get("user_123", query="hello")
except UnauthorizedError:
    print("Check your API key")
except QuotaExceededError as e:
    print(f"Quota reset at: {e.reset_at}")
except RateLimitedError as e:
    print(f"Retry after {e.retry_after}s")  # SDK retries automatically
except (InternalError, ServiceUnavailableError):
    pass  # SDK retries automatically with exponential backoff
```

**Retryable errors** (handled automatically by the SDK):

| Error | Strategy |
|-------|----------|
| `RateLimitedError` (429) | Sleep `retry_after` seconds, retry |
| `InternalError` (500) | Exponential backoff: 1s, 2s, 4s |
| `ServiceUnavailableError` (503) | Exponential backoff: 2s, 4s, 8s |

**Non-retryable errors** (fix required):

| Error | Cause |
|-------|-------|
| `UnauthorizedError` (401) | Invalid or missing API key |
| `QuotaExceededError` (402) | Monthly quota exhausted |
| `ForbiddenError` (403) | API key lacks required scope |
| `NotFoundError` (404) | Resource not found |
| `ValidationError` (422) | Invalid request body |

## Custom Base URL

For local development or self-hosted:

```python
mem = getmem.init("gm_live_...", base_url="http://localhost:8001")
```

## Links

- **Docs & dashboard:** [platform.getmem.ai](https://platform.getmem.ai)
- **Website:** [getmem.ai](https://getmem.ai)
- **npm (JS/TS):** [npmjs.com/package/getmem](https://npmjs.com/package/getmem)
- **GitHub (JS SDK):** [github.com/getmem-ai/getmem-js](https://github.com/getmem-ai/getmem-js)
- **Issues:** [github.com/getmem-ai/getmem/issues](https://github.com/getmem-ai/getmem/issues)
