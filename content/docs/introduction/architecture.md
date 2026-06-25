---
title: "Architecture"
linkTitle: "Architecture"
weight: 2
description: "The five-layer stack and how all UAI services interact."
---

## Five-Layer Stack

UAI is organized into five layers, from global AI discovery at the top down to citizen-facing applications at the bottom.

| Layer | Name | Owner | Purpose |
|-------|------|-------|---------|
| 0 | Global Discovery | NANDA / MIT | Agent identity resolution via AgentFacts documents. UAI federates into this as a participant. |
| 1 | National Registry | MeitY / IndiaAI | India-specific agent registration, capability ledger, and VC verification |
| 2 | Trust Protocol | UAI Trust Server | Delegation token issuance, scope negotiation, risk-tiered authorization, consent management |
| 3 | Transaction & Audit | C-DAC / DIC | Receipt logging, non-repudiation, IGM grievance, and immutable audit ledger |
| 4 | Application | Market | Citizen-facing agents, ministry workflows, enterprise integrations |

## Service Map

The UAI platform consists of six FastAPI microservices:

| Service | Port | Backend | Responsibility |
|---------|------|---------|---------------|
| Registry | 8000 | Elasticsearch 8.13.0 | Agent registration, discovery, proof-of-control validation |
| Audit Ledger | 8001 | PostgreSQL 16 | Append-only receipt store |
| Trust Server | 8002 | In-memory | JWT issuance, policy evaluation, consent validation |
| Toy Provider | 8003 | In-memory | Reference P2P provider implementation |
| Mock Consent | 8004 | In-memory | Pre-seeded consent artifacts for Tier 2 testing |
| Demo Providers | 8005 | In-memory | Five domain providers: education, agriculture, bhashini, health, finance |

## Interaction Flow

```
Seeker Agent                Trust Server (8002)           Provider Agent
     │                           │                              │
     │── POST /trust/v1/token ──>│                              │
     │   {seeker_did,            │── GET /registry/v1/discover ─┤
     │    provider_did,          │   (verify provider exists)   │
     │    scope,                 │                              │
     │    [consent_id],          │── GET {consent_endpoint}/... ┤
     │    [DPoP header]}         │   (Tier 2+: fetch & validate │
     │                           │    consent artifact)         │
     │<── {access_token,         │                              │
     │     expires_in,           │                              │
     │     tier,                 │                              │
     │     transaction_id}  ─────│                              │
     │                           │                              │
     │────── POST /uai/v1/query ────────────────────────────────>
     │        Authorization: Bearer {JWT}                       │
     │        [DPoP: {proof} for Tier 3+]                       │
     │                                    verify JWT offline    │
     │<────────────── {response data} ──────────────────────────│
     │                           │                              │
     │                           │<── POST /audit/v1/receipts ──│
     │                           │    Audit Ledger (8001)       │
     │                           │    (async, background)       │
```

**Key points:**

- The Trust Server contacts the Registry to verify the provider exists and supports the requested scope.
- For Tier 2+, the Trust Server fetches the consent artifact from the Seeker's `consent_endpoint` and runs five validation checks.
- The Provider verifies the JWT **offline** using the Trust Server's published DID Document — no callback required.
- The Provider submits a receipt to the Audit Ledger **asynchronously** (fire-and-forget) after responding to the Seeker.
- The Seeker and Provider communicate **peer-to-peer** — UAI is never in the data path.

## Control Plane vs. Data Plane

| Plane | What it carries | UAI involvement |
|-------|----------------|-----------------|
| Control plane | Token requests, consent validation, registration | Trust Server is the authority |
| Data plane | Actual query payloads and responses | UAI is never involved; P2P only |
| Audit plane | Hashed receipts | Audit Ledger stores hashes, not content |

## Infrastructure Dependencies

```
Docker Compose
├── elasticsearch:8.13.0  (port 9200)  ← Registry backend
├── postgres:16-alpine    (port 5433)  ← Audit Ledger backend
├── redis:7-alpine        (port 6379)  ← Reserved for future use
└── kibana:8.13.0         (port 5601)  ← Elasticsearch UI
```

## Bridge Adapter

The Bridge Adapter (`adapters/bridge/`) is a Python package that sits between NANDA global discovery and the UAI national trust infrastructure. It provides:

- **DID resolution** — `did:web` → DID Document (cached 5 min)
- **Signature verification** — Ed25519 and P-256 key types
- **Agent discovery** — queries Registry, falls back to NANDA NEST
- **Protocol negotiation** — selects best compatible endpoint from an agent's facts
- **AgentFacts publishing** — validates and publishes UAI extension blocks to Registry and NANDA

## MCP Bridge

The MCP Bridge (`adapters/bridge/mcp_bridge.py`) translates MCP `tool_call` dicts into UAI envelopes and back. It resolves tool names to UAI scopes via an explicit scope map, a `__`-separated naming convention, or dynamic Registry discovery.
