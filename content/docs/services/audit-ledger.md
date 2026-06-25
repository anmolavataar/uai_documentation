---
title: "Audit Ledger"
linkTitle: "Audit Ledger"
weight: 3
description: "Append-only, tamper-proof receipt storage backed by PostgreSQL."
---

**Location:** `services/audit-ledger/`  
**Port:** 8001  
**Backend:** PostgreSQL 16 (table: `receipts`)

The Audit Ledger provides non-repudiation for all UAI interactions. Once a receipt is written, it cannot be modified or deleted — enforced at the database level via PostgreSQL triggers.

## Immutability Guarantee

PostgreSQL triggers reject any `UPDATE` or `DELETE` operation on the `receipts` table. The append-only constraint is enforced at the database level, not just in application code — it cannot be bypassed even with direct database access (without dropping the trigger).

## Receipt Submission

`POST /audit/v1/receipts` accepts a UAI Receipt document:

1. Validates the receipt against `schemas/uai-receipt/schema.json`
2. Checks for duplicate `transaction_id` → returns `409 Conflict` if already exists
3. INSERTs the receipt into the PostgreSQL `receipts` table
4. Returns `202 Accepted` (asynchronous — the receipt is durably stored but the response is non-blocking)

Write access is currently open (Phase 1). Phase 2 will add provider authentication at the write endpoint.

## Receipt Querying

`GET /audit/v1/receipts` requires authentication via HS256 JWT:

```python
import jwt, time
token = jwt.encode(
    {"sub": "did:web:your.agent.did", "iat": int(time.time()), "exp": int(time.time()) + 300},
    "dev-secret-change-in-prod",
    algorithm="HS256"
)
```

**Access control:** Only receipts where the authenticated DID (`sub`) is the `seeker_did` or `provider_did` are returned. If a `transaction_id` filter is specified and the requester is not a participant, the response is `403 Forbidden` (not `404` — the ledger does not leak existence of transactions to non-participants).

**Query parameters:**

| Parameter | Description |
|-----------|-------------|
| `transaction_id` | Filter by specific transaction |
| `date_from` | ISO 8601 start date |
| `date_to` | ISO 8601 end date |

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PG_DSN` | `postgresql://uai:uai@127.0.0.1:5433/audit_ledger` | PostgreSQL connection string |
| `PG_TABLE` | `receipts` | Table name |
| `AUDIT_JWT_SECRET` | `dev-secret-change-in-prod` | HS256 secret for query authentication |

{{% alert title="Production Security" color="warning" %}}
Change `AUDIT_JWT_SECRET` in production. The default value is public and provides no security.
{{% /alert %}}

## Start Command

```bash
make start-audit-ledger
# or manually:
cd services/audit-ledger
PYTHONPATH=../.. uvicorn main:app --port 8001
```

## Requires PostgreSQL

The Audit Ledger requires PostgreSQL to be running:

```bash
make up  # starts PostgreSQL (and other infrastructure) via Docker Compose
```

Wait for the health check: `docker compose ps` shows `audit_ledger_db (healthy)`.

## Health Check

```bash
curl http://localhost:8001/docs
# Returns OpenAPI documentation page
```
