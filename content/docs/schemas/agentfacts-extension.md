---
title: "AgentFacts UAI Extension"
linkTitle: "AgentFacts Extension"
weight: 4
description: "The UAI extension block that makes a NANDA AgentFacts document UAI-compliant."
---

**File:** `schemas/agentfacts-uai-extension/schema.json`

NANDA (Network of AI Networked Distributed Agents) defines the global **AgentFacts** document format for AI agent discovery. UAI extends this with a `uai` block that carries UAI-specific registration information, enabling global discovery while preserving UAI trust metadata.

## Purpose

An agent that publishes an AgentFacts document with a valid `uai` extension block can be discovered through both the NANDA global directory and the UAI national Registry. The bridge adapter validates this block before publishing.

## Extension Fields

The `uai` block is added inside an existing AgentFacts document at the top level.

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `uai_registration` | object | UAI Registry registration details |
| `delegation_scopes` | array | Non-empty array of scope strings this agent supports |
| `risk_tier_max` | integer | Maximum risk tier (1–4) |

### Conditional Fields

| Field | Required when | Description |
|-------|--------------|-------------|
| `vc_refs` | `risk_tier_max >= 3` | Non-empty array of DID URL strings pointing to VC credentials |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| `india_compliance` | object | `{dpdp_category, data_residency}` |
| `consent_requirements` | object | Per-tier consent method map |

### UAI Registration Object

```json
{
  "registry": "https://registry.uai.gov.in",
  "registry_record_id": "did:web:nafpo.gov.in",
  "last_verified": "2026-04-01T00:00:00Z"
}
```

## Complete Example (Agriculture Provider)

```json
{
  "id": "did:web:nafpo.gov.in",
  "name": "NAFPO FPO Finder",
  "description": "Provides FPO discovery and crop diagnostics for Indian agriculture sector",
  "uai": {
    "uai_registration": {
      "registry": "https://registry.uai.gov.in",
      "registry_record_id": "did:web:nafpo.gov.in",
      "last_verified": "2026-04-01T00:00:00Z"
    },
    "delegation_scopes": [
      "read:crop_diagnosis",
      "read:fpo_info",
      "read:mandi_prices"
    ],
    "risk_tier_max": 1,
    "india_compliance": {
      "dpdp_category": "agricultural",
      "data_residency": "IN"
    }
  }
}
```

## Publishing AgentFacts

The `BridgeAdapter` class in `adapters/bridge/publish_agentfacts.py` handles publishing:

```python
from adapters.bridge.publish_agentfacts import BridgeAdapter

adapter = BridgeAdapter(
    registry_url="http://localhost:8000",
    nanda_endpoint="https://nest.projectnanda.org",  # optional
)

await adapter.publish_agentfacts(
    agent_did="did:web:nafpo.gov.in",
    facts=agentfacts_dict,        # must include "uai" block
    facts_url="https://nafpo.gov.in/.well-known/agentfacts.json",
)
```

**Publication sequence:**
1. Validate `facts["id"] == agent_did` — raises `DIDMismatchError` if not
2. Validate `facts["uai"]` against the AgentFacts UAI Extension schema — raises `AgentFactsValidationError` if invalid
3. PATCH `facts_ref` field into the UAI Registry record
4. POST to NANDA NEST (if `nanda_endpoint` configured) — failure here does NOT prevent UAI success

## NANDA Compatibility Note

{{% alert title="NANDA Limitation" color="info" %}}
NANDA NEST currently ignores all UAI-specific fields in the `uai` extension block. The extension was designed by the UAI team. No live NEST agent currently has a `uai` block. NANDA discovery only supports text search on name/description — it cannot filter by scope, capability, tier, or VC requirements. The UAI Registry (Elasticsearch) is the primary discovery layer; NANDA is a secondary global directory for visibility.
{{% /alert %}}
