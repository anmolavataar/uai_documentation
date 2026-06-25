---
title: "Trust Server API"
linkTitle: "Trust Server"
weight: 2
description: "Complete endpoint reference for the UAI Trust Server (port 8002)."
---

Base URL: `http://localhost:8002` (development)

---

## GET /.well-known/did.json

Returns the Trust Server's DID Document containing all active public keys. Providers use this to verify delegation tokens offline.

**Auth:** None  
**Response: `200 OK`**

```json
{
  "@context": ["https://www.w3.org/ns/did/v1"],
  "id": "did:web:localhost%3A8002",
  "verificationMethod": [
    {
      "id": "did:web:localhost%3A8002#key-1",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:web:localhost%3A8002",
      "publicKeyMultibase": "z6Mkf5..."
    }
  ],
  "authentication": ["did:web:localhost%3A8002#key-1"],
  "assertionMethod": ["did:web:localhost%3A8002#key-1"]
}
```

```bash
curl http://localhost:8002/.well-known/did.json
```

---

## POST /trust/v1/token

Issues a delegation JWT after evaluating the risk policy, validating consent (Tier 2+), and verifying DPoP (Tier 3+).

**Auth:** None for Tier 1; Consent required for Tier 2+; DPoP proof required for Tier 3+  
**Content-Type:** `application/json`  
**Headers (Tier 3+):** `DPoP: {proof-jwt}`

**Request body:**

```json
{
  "seeker_did": "did:web:seeker.example.com",
  "provider_did": "did:web:localhost%3A8005:agriculture",
  "scope": "read:crop_diagnosis",
  "consent_id": "cns_valid_tier2_agri"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `seeker_did` | Yes | Seeker's DID |
| `provider_did` | Yes | Provider's DID |
| `scope` | Yes | Exactly one scope string |
| `consent_id` | Tier 2+ | Consent artifact ID (must start with `cns_`) |

**Response: `200 OK`**

```json
{
  "access_token": "eyJhbGciOiJFZERTQSIsImtpZCI6...",
  "token_type": "Bearer",
  "expires_in": 300,
  "scope": "read:crop_diagnosis",
  "transaction_id": "txn_abc123",
  "tier": 1
}
```

### Error Codes

**400 Bad Request:**

| `error` | Cause |
|---------|-------|
| `consent_required` | Tier 2+ scope requested without `consent_id` |
| `consent_expired` | Consent artifact's `expires_at` is in the past |
| `consent_revoked` | `revocation_status` is not `"active"` |
| `consent_dpdp_invalid` | One or more `dpdp_properties` is `false` |
| `dpop_invalid` | DPoP proof signature invalid or missing required fields |
| `dpop_proof_expired` | DPoP proof `iat` is more than 30 seconds old |
| `dpop_replay` | DPoP proof `jti` has been used before |

**401 Unauthorized:**

| `error` | Cause |
|---------|-------|
| `dpop_required` | Tier 3+ scope requested without `DPoP` header |

**403 Forbidden:**

| `error` | Cause |
|---------|-------|
| `scope_not_in_policy` | Scope not defined in active risk policy |
| `scope_not_supported_by_provider` | Provider's `supported_scopes` doesn't include the requested scope |
| `consent_scope_mismatch` | Requested scope not in consent artifact's `allowed_scopes` |
| `consent_did_mismatch` | Seeker or provider DID in consent doesn't match request |

**404 Not Found:**

| `error` | Cause |
|---------|-------|
| `provider_not_found` | Provider DID not in Registry |
| `seeker_not_found` | Seeker DID not in Registry |
| `consent_not_found` | `consent_id` not found at Seeker's `consent_endpoint` |

**503 Service Unavailable:**

| `error` | Cause |
|---------|-------|
| `registry_unavailable` | Could not reach Registry |
| `consent_fetch_failed` | HTTP error fetching consent artifact |

---

## POST /trust/v1/admin/rotate-key

Rotates the Trust Server's signing key. Generates a new Ed25519 key pair, adds it to the DID Document, and uses it for all future token signing. Up to 3 keys are maintained simultaneously.

**Auth:** None (Phase 1)  
**Request body:** None  
**Response: `200 OK`**

```json
{
  "new_kid": "did:web:localhost%3A8002#key-2",
  "active_keys": 2
}
```

```bash
curl -X POST http://localhost:8002/trust/v1/admin/rotate-key
```
