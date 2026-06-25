---
title: "UAI Receipt"
linkTitle: "Receipt"
weight: 2
description: "Schema for the tamper-proof interaction receipt submitted to the Audit Ledger."
---

**File:** `schemas/uai-receipt/schema.json`

A UAI Receipt is produced by the Provider after every successful (or failed) P2P interaction. It contains cryptographic hashes of the exchanged data — never the raw data itself.

## Required Fields

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| `receipt_version` | string | Must be `"1.0.0"` | Schema version |
| `transaction_id` | string | Must start with `txn_` | Unique transaction identifier |
| `message_id` | string | Must start with `msg_` | Unique message identifier |
| `issued_at` | string | ISO 8601 | When the receipt was created |
| `participants` | object | — | DIDs of all parties |
| `authorization` | object | — | Token and scope information |
| `artifacts` | object | — | Hashes of exchanged data |
| `outcome` | object | — | Interaction result |
| `signature` | object | — | Provider's cryptographic signature |

## Participants Object

```json
{
  "seeker_did": "did:web:seeker.example.com",
  "provider_did": "did:web:provider.example.com",
  "trust_server_did": "did:web:trust.uai.local"
}
```

All three DID fields must start with `did:`.

## Authorization Object

```json
{
  "token_hash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "scope": "read:crop_diagnosis",
  "tier": 1
}
```

`token_hash` must start with `sha256:` — it is the SHA-256 hash of the raw bearer token, not the token itself.

## Artifacts Object

```json
{
  "request_hash": "sha256:6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b",
  "response_hash": "sha256:d4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35",
  "canonicalization": "jcs:rfc8785"
}
```

- `request_hash` and `response_hash` must start with `sha256:`
- `canonicalization` must be `"jcs:rfc8785"` (RFC 8785 JSON Canonicalization Scheme)
- The `proof` field is excluded before canonicalization

## Outcome Object

```json
{
  "status": "success",
  "http_status": 200
}
```

| Field | Values |
|-------|--------|
| `status` | `"success"`, `"failure"`, `"partial"` |
| `http_status` | HTTP status code (200, 400, 500, etc.) |
| `error_code` | Optional — present on failure |

## Signature Object

```json
{
  "signer": "did:web:provider.example.com",
  "alg": "EdDSA",
  "sig": "<base64url-encoded-Ed25519-signature>"
}
```

| Field | Values |
|-------|--------|
| `signer` | Provider's DID (must start with `did:`) |
| `alg` | `"EdDSA"` or `"ES256"` |

The signature covers the entire receipt body (excluding the `signature.sig` field itself) after JCS canonicalization.

## Complete Example (Success)

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
    "token_hash": "sha256:abc123...",
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

## Invalid Document Examples

The `invalid/` directory tests rejection of:
- `raw-payload-instead-of-hash.json` — artifacts that contain plain text instead of `sha256:` prefixed hashes
- `wrong-receipt-version.json` — any version string other than `"1.0.0"`
