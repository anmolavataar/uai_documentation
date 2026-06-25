---
title: "Receipts and Audit"
linkTitle: "Receipts & Audit"
weight: 4
description: "How UAI creates tamper-proof, privacy-preserving audit records of every interaction."
---

Every UAI interaction produces a **receipt** — a cryptographically signed, append-only record that proves the interaction happened, what was authorized, and what the outcome was, without ever storing the raw data exchanged.

## Privacy-First Design

UAI stores **hashes, not payloads**. The receipt contains:

- SHA-256 hash of the JCS-canonicalized request body
- SHA-256 hash of the JCS-canonicalized response body
- The token hash (not the token itself)
- Participant DIDs
- Scope and tier
- Outcome status

No citizen data ever enters the Audit Ledger.

## JCS Canonicalization (RFC 8785)

Before hashing, both request and response bodies are **JCS-canonicalized**:

- Keys sorted lexicographically at every nesting level
- No insignificant whitespace
- Shortest number representation
- The `proof` field is excluded from the input before canonicalization

This ensures the same logical document always produces the same hash, regardless of key ordering or whitespace differences in the original JSON.

Hash format: `sha256:{hex-encoded-digest}`

## Receipt Structure

```json
{
  "receipt_version": "1.0.0",
  "transaction_id": "txn_abc123",
  "message_id": "msg_xyz789",
  "issued_at": "2026-06-25T10:00:00Z",
  "participants": {
    "seeker_did": "did:web:seeker.example.com",
    "provider_did": "did:web:provider.example.com",
    "trust_server_did": "did:web:trust.uai.local"
  },
  "authorization": {
    "token_hash": "sha256:e3b0c44298fc1c14...",
    "scope": "read:crop_diagnosis",
    "tier": 1
  },
  "artifacts": {
    "request_hash": "sha256:6b86b273ff34fce...",
    "response_hash": "sha256:d4735e3a265e16e...",
    "canonicalization": "jcs:rfc8785"
  },
  "outcome": {
    "status": "success",
    "http_status": 200
  },
  "signature": {
    "signer": "did:web:provider.example.com",
    "alg": "EdDSA",
    "sig": "<base64url-encoded-signature>"
  }
}
```

**Field rules:**
- `transaction_id` must start with `txn_`
- `message_id` must start with `msg_`
- All DID fields must start with `did:`
- All hash fields must start with `sha256:`
- `outcome.status` is one of: `success`, `failure`, `partial`
- `signature.alg` is `EdDSA` or `ES256`

## Who Signs the Receipt

The **Provider** signs the receipt with its own Ed25519 private key, using its DID as `signature.signer`. This means the receipt is the Provider's signed attestation that the interaction occurred and the outcome is as stated. The Seeker cannot forge a receipt.

## Audit Ledger Immutability

The Audit Ledger (`services/audit-ledger/`, port 8001) enforces immutability at the database level via PostgreSQL triggers:

- **No UPDATE** — once written, a receipt cannot be modified
- **No DELETE** — receipts cannot be removed
- **Duplicate `transaction_id`** — returns HTTP 409 Conflict

This means once the Provider submits a receipt, the record is permanent and tamper-proof.

## Querying Receipts

The Audit Ledger enforces participant-only access. To query receipts, a DID holder authenticates with an HS256 JWT signed with the `AUDIT_JWT_SECRET`:

```bash
# Create an auth JWT (sub = your DID)
TOKEN=$(python3 -c "
import jwt, time
print(jwt.encode({'sub':'did:web:your.did','iat':int(time.time()),'exp':int(time.time())+300}, 'dev-secret-change-in-prod', algorithm='HS256'))
")

# Query receipts where you are a participant
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8001/audit/v1/receipts"

# Query a specific transaction
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8001/audit/v1/receipts?transaction_id=txn_abc123"
```

**Access control rules:**
- Only receipts where `sub` (your DID) is the `seeker_did` or `provider_did` are returned
- If you query a `transaction_id` where you are not a participant → 403 (the ledger does not reveal the receipt exists)
- Date range filters: `date_from`, `date_to` (ISO 8601)

## Retention Requirements

Per CERT-In compliance, receipts must be retained for a minimum of **180 days** in Indian jurisdiction. The Audit Ledger does not auto-expire receipts — retention policy enforcement is an operational concern.

## Receipt Flow (Sequence)

```
Provider processes query
        │
        ├─ Build UAIReceipt:
        │    - JCS-canonicalize request + response
        │    - SHA-256 hash each
        │    - Fill all required fields
        │    - Sign with provider's Ed25519 key
        │
        └─ POST /audit/v1/receipts  (async, non-blocking)
                │
                ▼
         Audit Ledger
                │
                ├─ Validate against receipt JSON Schema
                ├─ Check for duplicate transaction_id → 409 if exists
                └─ INSERT into postgres receipts table
                   (trigger prevents future UPDATE/DELETE)
```

The Provider fires this request in the background — it does **not** block the response to the Seeker. If the Audit Ledger is temporarily unavailable, the receipt is lost for that transaction. Phase 2 will add a local write-ahead queue.
