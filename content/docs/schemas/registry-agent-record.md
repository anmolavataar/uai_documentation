---
title: "Registry Agent Record"
linkTitle: "Agent Record"
weight: 1
description: "Schema for agent registration records in the UAI Registry."
---

**File:** `schemas/registry-agent-record/schema.json`

Every registered agent has exactly one Registry Agent Record. It is created via `POST /registry/v1/agents` and stored in Elasticsearch.

## Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `did` | string | Agent DID. Must start with `did:`. |
| `role` | enum | `"seeker"`, `"provider"`, or `"both"` |
| `endpoints` | array | Communication endpoints (see below) |
| `capabilities` | array | Capability URNs supported by this agent |
| `supported_scopes` | array | Scope strings this agent can serve (or request) |
| `risk_tier_max` | integer | Maximum trust tier: 1–4 |
| `registered_at` | string | ISO 8601 datetime of registration |

## Conditional Fields

| Field | Required when | Description |
|-------|--------------|-------------|
| `vc_refs` | `risk_tier_max >= 3` | Array of DID URIs pointing to VC credentials (e.g., `did:web:abdm.gov.in#vc-insurance-2026`) |
| `consent_endpoint` | `role = seeker` AND `risk_tier_max >= 2` | URL where Trust Server fetches consent artifacts |

## Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `facts_ref` | string | URL to the agent's NANDA AgentFacts document |
| `india_compliance` | object | `{dpdp_category, data_residency}` |
| `etag` | string | ETag for optimistic concurrency |
| `registry_version` | string | Schema version |

## Endpoint Object

```json
{
  "uri": "https://provider.example.gov.in/uai/v1/query",
  "transport": "https",
  "auth": "bearer"
}
```

| Field | Values |
|-------|--------|
| `transport` | `https`, `wss` |
| `auth` | `bearer`, `dpop`, `none` |

## Capability Object

```json
{
  "id": "urn:capability:crop-disease-detection:v1",
  "version": "1.0"
}
```

The `id` must start with `urn:capability:`.

## Complete Example (Tier 1 Agriculture Provider)

```json
{
  "did": "did:web:nafpo.gov.in",
  "role": "provider",
  "endpoints": [
    {
      "uri": "https://nafpo.gov.in/uai/v1/query",
      "transport": "https",
      "auth": "bearer"
    }
  ],
  "capabilities": [
    { "id": "urn:capability:crop-disease-detection:v1", "version": "1.0" },
    { "id": "urn:capability:fpo-information:v1", "version": "1.0" }
  ],
  "supported_scopes": ["read:crop_diagnosis", "read:fpo_info", "read:mandi_prices"],
  "risk_tier_max": 1,
  "facts_ref": "https://nafpo.gov.in/.well-known/agentfacts.json",
  "india_compliance": {
    "dpdp_category": "agricultural",
    "data_residency": "IN"
  },
  "registered_at": "2026-04-01T00:00:00Z"
}
```

## Complete Example (Tier 3 Health Provider)

```json
{
  "did": "did:web:abdm.gov.in",
  "role": "provider",
  "endpoints": [
    {
      "uri": "https://abdm.gov.in/uai/v1/query",
      "transport": "https",
      "auth": "dpop"
    }
  ],
  "capabilities": [
    { "id": "urn:capability:health-records:v1", "version": "1.0" }
  ],
  "supported_scopes": ["read:health_profile", "read:diagnostic_result"],
  "risk_tier_max": 3,
  "vc_refs": ["did:web:abdm.gov.in#vc-insurance-2026"],
  "india_compliance": {
    "dpdp_category": "health",
    "data_residency": "IN"
  },
  "registered_at": "2026-04-01T00:00:00Z"
}
```

## DID Normalization

The Registry normalizes DIDs before storage. Percent-encoded colons in DID paths are decoded:

```
did:web:localhost%3A8005:education  →  stored as  did:web:localhost:8005:education
```

This normalization ensures consistent querying regardless of how the DID was encoded in the registration request.

## Patchable Fields

Only three fields can be updated after registration via `PATCH /registry/v1/agents/{did}`:
- `facts_ref`
- `india_compliance`
- `consent_endpoint`

All other fields (including `supported_scopes` and `risk_tier_max`) require re-registration.
