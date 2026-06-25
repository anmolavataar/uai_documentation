---
title: "Audit Ledger API"
linkTitle: "Audit Ledger"
weight: 3
description: "Complete endpoint reference for the UAI Audit Ledger (port 8001)."
---

Base URL: `http://localhost:8001` (development)

---

## POST /audit/v1/receipts

Submits a UAI Receipt to the append-only ledger.

**Auth:** None (Phase 1 — open write endpoint)  
**Content-Type:** `application/json`

**Request body:** A complete `UAIReceipt` document — see the [Receipt Schema](/docs/schemas/uai-receipt/).

```json
{
  "receipt_version": "1.0.0",
  "transaction_id": "txn_20260625_001",
  "message_id": "msg_20260625_001",
  "issued_at": "2026-06-25T10:00:00Z",
  "participants": {
    "seeker_did": "did:web:seeker.example.com",
    "provider_did": "did:web:localhost:8005:agriculture",
    "trust_server_did": "did:web:trust.uai.local"
  },
  "authorization": {
    "token_hash": "sha256:e3b0c44...",
    "scope": "read:crop_diagnosis",
    "tier": 1
  },
  "artifacts": {
    "request_hash": "sha256:6b86b2...",
    "response_hash": "sha256:d4735e...",
    "canonicalization": "jcs:rfc8785"
  },
  "outcome": {
    "status": "success",
    "http_status": 200
  },
  "signature": {
    "signer": "did:web:localhost:8005:agriculture",
    "alg": "EdDSA",
    "sig": "base64url..."
  }
}
```

**Response: `202 Accepted`**

```json
{ "status": "accepted", "transaction_id": "txn_20260625_001" }
```

**Error responses:**
- `409 Conflict` — `transaction_id` already exists in the ledger
- `422 Unprocessable Entity` — receipt fails JSON Schema validation (e.g., missing `sha256:` prefix, wrong `receipt_version`)

---

## GET /audit/v1/receipts

Queries receipts from the ledger. Only returns receipts where the authenticated DID is a participant.

**Auth:** `Authorization: Bearer {HS256-JWT}`

The HS256 JWT must be signed with `AUDIT_JWT_SECRET` and contain:
- `sub`: The querying DID (e.g., `did:web:seeker.example.com`)
- `iat`: Issued-at timestamp
- `exp`: Expiry timestamp

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `transaction_id` | string | Filter by specific transaction ID |
| `date_from` | string | ISO 8601 start date for results |
| `date_to` | string | ISO 8601 end date for results |

**Response: `200 OK`**

```json
{
  "receipts": [
    {
      "receipt_version": "1.0.0",
      "transaction_id": "txn_20260625_001",
      ...
    }
  ]
}
```

**Error responses:**
- `401 Unauthorized` — missing or invalid JWT
- `403 Forbidden` — `transaction_id` was specified but the authenticated DID is not a participant (does not reveal that the transaction exists)

### Creating an Audit Query JWT

```python
import jwt, time

token = jwt.encode(
    {
        "sub": "did:web:your.agent.did",
        "iat": int(time.time()),
        "exp": int(time.time()) + 300,
    },
    "dev-secret-change-in-prod",  # AUDIT_JWT_SECRET
    algorithm="HS256",
)
```

```bash
# Get all receipts where you are a participant
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8001/audit/v1/receipts"

# Filter by transaction
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8001/audit/v1/receipts?transaction_id=txn_abc123"

# Filter by date range
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8001/audit/v1/receipts?date_from=2026-06-01T00:00:00Z&date_to=2026-06-30T23:59:59Z"
```
