# Idempotency Keys

**Problem:** a client calls `POST /payments`, the request succeeds, but the response gets
lost (timeout, dropped connection). The client retries. Without protection, the payment
gets charged twice.

**Fix:** the client generates a unique key per *logical* operation (not per HTTP attempt)
and sends it in a header. The server remembers the result for that key and, on a retry,
replays the stored response instead of re-executing the operation.

```python
import hashlib
import json
from dataclasses import dataclass
from time import time

@dataclass
class StoredResult:
    status_code: int
    body: dict
    request_hash: str
    created_at: float

# In production this is a Redis/DB table with a TTL, not an in-memory dict.
_store: dict[str, StoredResult] = {}

def _hash_request(payload: dict) -> str:
    return hashlib.sha256(json.dumps(payload, sort_keys=True).encode()).hexdigest()

def handle_idempotent_request(idempotency_key: str, payload: dict, execute):
    """
    execute: callable that performs the actual side-effecting operation and
    returns (status_code, body). Only called if this key hasn't been seen before.
    """
    request_hash = _hash_request(payload)
    existing = _store.get(idempotency_key)

    if existing is not None:
        if existing.request_hash != request_hash:
            # Same key reused with a different payload — a client bug, not a safe retry.
            return 422, {"error": "Idempotency-Key reused with a different request body"}
        return existing.status_code, existing.body

    status_code, body = execute(payload)
    _store[idempotency_key] = StoredResult(status_code, body, request_hash, time())
    return status_code, body
```

Usage in a FastAPI-style handler:

```python
@app.post("/payments")
def create_payment(payload: dict, idempotency_key: str = Header(...)):
    def execute(p):
        charge = process_charge(p)  # the actual side-effecting call
        return 201, {"charge_id": charge.id}

    status_code, body = handle_idempotent_request(idempotency_key, payload, execute)
    return JSONResponse(status_code=status_code, content=body)
```

## When to use it
- Any endpoint that has a real-world side effect and could plausibly be retried:
  payments, order creation, sending an email/SMS, provisioning a resource.

## When not to
- Pure `GET`s and other naturally idempotent operations — they don't need this, retrying
  them is already safe.
- Don't key on request content alone (two different orders with the same payload would
  collide) — the key must come from the client and represent one logical attempt.
