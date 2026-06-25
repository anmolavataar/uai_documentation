---
title: "Registry"
linkTitle: "Registry"
weight: 1
description: "Agent registration, discovery, and proof-of-control validation."
---

**Location:** `services/registry/`  
**Port:** 8000  
**Backend:** Elasticsearch 8.13.0 (index: `uai-agents`)

The Registry is the authoritative directory of all UAI-participating agents. It handles agent registration with cryptographic proof-of-control, capability-based discovery, and ETag-based caching.

## Startup Sequence

1. Connects to Elasticsearch at `ES_HOST` (default: `http://localhost:9200`)
2. Creates the `uai-agents` index if it doesn't exist (with appropriate field mappings)
3. Initializes `NonceStore` for challenge-response registration (in-memory, 5-min TTL per nonce)
4. Starts FastAPI on port 8000

## Key Dependencies

- `adapters/bridge/did_resolver` — resolves `did:web` DIDs to DID Documents
- `adapters/bridge/signature_verifier` — verifies Ed25519 and P-256 signatures
- `jsonschema` (Draft202012Validator) — validates records against `schemas/registry-agent-record/schema.json`

## Registration Validation

When `POST /registry/v1/agents` is called, the Registry:

1. Looks up the nonce in `NonceStore` — rejects if expired or already used
2. Resolves the agent's `did:web` DID to fetch the DID Document
3. Finds the verification method matching the `proof.verificationMethod` claim
4. Verifies the Ed25519 (or P-256) signature over `"register:{did}:nonce:{nonce}"`
5. Validates the full agent record against the JSON Schema
6. Normalizes the DID (decodes `%3A` to `:`)
7. Stores in Elasticsearch and returns the record with `ETag` header

## Discovery Query

`GET /registry/v1/discover` queries Elasticsearch with AND-combined filters. All parameters are optional:

```
GET /registry/v1/discover?scope=read:crop_diagnosis&tier_max=1&data_residency=IN&page_size=10
```

Returns a paginated list of matching agent records with an `ETag` header. Include `If-None-Match: {etag}` to get `304 Not Modified` when the result hasn't changed.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ES_INDEX` | `uai-agents` | Elasticsearch index name |
| `ES_HOST` | `http://localhost:9200` | Elasticsearch connection URL |

## Start Command

```bash
make start-registry
# or manually:
uvicorn services.registry.main:app --port 8000
```

## Health Check

```bash
curl http://localhost:8000/registry/v1/discover
# Returns: { "results": [], "next_cursor": null }
```
