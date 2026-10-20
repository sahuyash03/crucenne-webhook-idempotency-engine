# Crucenne Webhook Idempotency Engine

Distributed, production-grade idempotency and de-duplication reference architecture engineered for high-throughput enterprise webhooks, payments, and asynchronous state synchronization pipelines.

---

## The Problem
Distributed webhook senders (Stripe, Shopify, ERP event streams) guarantee **at-least-once delivery**. In high-concurrency environments, network retries and transient timeouts often trigger duplicate concurrent executions, leading to race conditions, double charging, and corrupted database ledger balances.

---

## Architectural Workflow

```mermaid
sequenceDiagram
    autonumber
    actor Provider as Upstream Webhook Source
    participant Gateway as API Gateway
    participant Engine as Idempotency Engine
    participant Redis as Redis (Distributed Lock)
    participant DB as Persistent State Store
    participant Consumer as Core Business Service

    Provider->>Gateway: POST /webhooks/v1/event (Idempotency-Key: evt_123)
    Gateway->>Engine: Intercept Payload + Key
    Engine->>Redis: SETNX lock:evt_123 (TTL: 60s)
    alt Lock Failed (Concurrent Duplicate In-Flight)
        Redis-->>Engine: Key Exists
        Engine-->>Gateway: HTTP 409 Conflict / HTTP 200 (De-duplicated)
        Gateway-->>Provider: Request Accepted (Duplicate Dropped)
    else Lock Acquired
        Engine->>DB: Query processed_events WHERE key = 'evt_123'
        alt Previously Processed
            DB-->>Engine: Cached Response Payload
            Engine-->>Provider: HTTP 200 OK (Replayed Cached Response)
        else Unprocessed Execution
            Engine->>DB: INSERT key INTO processed_events (status: PROCESSING)
            Engine->>Consumer: Dispatch Execution Handler
            Consumer-->>Engine: Success Response Data
            Engine->>DB: UPDATE status = COMPLETED, response = {...}
            Engine->>Redis: RELEASE lock:evt_123
            Engine-->>Provider: HTTP 200 OK (Processed Successfully)
        end
    end
```

---

## Core Enterprise Guardrails

- **Distributed Concurrency Lock**: Redis `SETNX` with a sliding TTL prevents parallel workers from processing identical payloads simultaneously.
- **Payload Signature Verification**: SHA-256 payload hashing ensures that malicious or erroneous key-reuse attempts with modified request bodies fail deterministically.
- **Safe Response Replay**: Fully processed events return the cached execution payload directly from the database without touching internal downstream services.
- **Fail-Safe Crash Recovery**: If a worker crashes mid-transaction, Redis lock TTL expiration paired with persistent state recovery routines unblocks subsequent retries safely.

---

## Project Structure

```text
├── docker-compose.yml           # Redis & State Store local topology
├── contracts/
│   └── sample_payload.json      # Standardized enterprise webhook schema
├── src/
│   ├── middleware/
│   │   └── idempotency.ts       # Core cryptographic hashing & lock gate
│   └── storage/                 # Persistence & distributed lock handlers
└── README.md
```

---

## Running the Mock Environment

```bash
docker compose up -d
```