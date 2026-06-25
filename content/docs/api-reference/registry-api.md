---
title: "Registry API"
linkTitle: "Registry"
weight: 1
description: "Complete endpoint reference for the UAI Registry service (port 8000)."
---

Base URL: `http://localhost:8000` (development)

---

## POST /admin/clear-did-cache

Clears the in-process DID resolver cache for all cached DID Documents.

**Auth:** None  
**Request body:** None  
**Response:** `200 OK`

```bash
curl -X POST http://localhost:8000/admin/clear-did-cache
```

---

## GET /registry/v1/challenge

Returns a one-time nonce for the proof-of-control registration flow.

**Auth:** None  
**Query parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `did` | string | Yes | The DID of the agent requesting registration |

**Response: `200 OK`**

```json
{
  "nonce": "abc123def456",
  "expires_in": 300,
  "challenge": "register:did:web:agent.gov.in:nonce:abc123def456"
}
```

**Error responses:**
- `400` — missing or invalid `did` parameter

```bash
curl "http://localhost:8000/registry/v1/challenge?did=did:web:my-agent.gov.in"
```

---

## POST /registry/v1/agents

Registers a new agent. Requires proof-of-control via Ed25519 signature.

**Auth:** Ed25519 proof-of-control (in request body)  
**Content-Type:** `application/json`

**Request body:**

```json
{
  "did": "did:web:my-agent.gov.in",
  "role": "provider",
  "endpoints": [
    { "uri": "https://my-agent.gov.in/uai/v1/query", "transport": "https", "auth": "bearer" }
  ],
  "capabilities": [
    { "id": "urn:capability:crop-disease-detection:v1", "version": "1.0" }
  ],
  "supported_scopes": ["read:crop_diagnosis"],
  "risk_tier_max": 1,
  "registered_at": "2026-06-25T10:00:00Z",
  "proof": {
    "type": "Ed25519Signature2020",
    "verificationMethod": "did:web:my-agent.gov.in#key-1",
    "signature": "<base64url-encoded-Ed25519-signature-over-challenge>"
  }
}
```

**Response: `201 Created`** — returns the stored agent record with `ETag` header.

**Error responses:**
- `400` — nonce expired, nonce replayed, or invalid signature
- `409` — DID already registered
- `422` — record fails JSON Schema validation

---

## GET /registry/v1/agents/{did}

Retrieves an agent record by DID.

**Auth:** None  
**Path parameter:** `did` — the agent DID (URL-encoded if it contains colons)

**Response: `200 OK`** — agent record with `ETag` header.

**Error responses:**
- `404` — DID not found in Registry

```bash
curl "http://localhost:8000/registry/v1/agents/did:web:my-agent.gov.in"
```

---

## PATCH /registry/v1/agents/{did}

Updates patchable fields on an existing agent record.

**Auth:** None (Phase 1)  
**Content-Type:** `application/json`

**Patchable fields only:**

```json
{
  "facts_ref": "https://my-agent.gov.in/.well-known/agentfacts.json",
  "india_compliance": { "dpdp_category": "agricultural", "data_residency": "IN" },
  "consent_endpoint": "https://my-agent.gov.in/consent"
}
```

**Response: `200 OK`** — updated agent record.

**Error responses:**
- `404` — DID not found
- `422` — attempted to patch a non-patchable field (e.g., `supported_scopes`, `risk_tier_max`)

---

## GET /registry/v1/discover

Discovers agents matching optional filters.

**Auth:** None  
**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `scope` | string | Filter by supported scope (e.g., `read:crop_diagnosis`) |
| `capability` | string | Filter by capability URN |
| `tier_max` | integer | Maximum risk tier (inclusive) |
| `data_residency` | string | Filter by data residency (e.g., `IN`) |
| `page_size` | integer | Results per page (default: 10) |
| `cursor` | string | Pagination cursor from previous response |

**Headers:**
- `If-None-Match: {etag}` — returns `304 Not Modified` if result set unchanged

**Response: `200 OK`**

```json
{
  "results": [
    {
      "did": "did:web:my-agent.gov.in",
      "role": "provider",
      "supported_scopes": ["read:crop_diagnosis"],
      "risk_tier_max": 1,
      ...
    }
  ],
  "next_cursor": null
}
```

**Response: `304 Not Modified`** — when `If-None-Match` header matches current ETag.

```bash
curl "http://localhost:8000/registry/v1/discover?scope=read:crop_diagnosis&tier_max=1"
```
