---
title: "NANDA Integration"
linkTitle: "NANDA Integration"
weight: 4
description: "How UAI federates with the global NANDA agent registry."
---

NANDA (Network of AI Networked Distributed Agents) is MIT's global AI agent discovery infrastructure. UAI participates in NANDA as a national registry, extending AgentFacts documents with a `uai` block.

## NANDA Architecture vs UAI Registry

| Capability | NANDA NEST | UAI Registry |
|------------|-----------|-------------|
| Discovery by name/description | Yes | No |
| Discovery by scope | No | Yes |
| Discovery by tier | No | Yes |
| Discovery by capability URN | No | Yes |
| Data residency filtering | No | Yes |
| VC requirements filtering | No | Yes |
| P95 discovery latency | ~1,400ms | <200ms |
| UAI extension block support | No (ignored) | Yes (required) |

## NANDA Exploration Findings

An exploration of the NANDA NEST testbed revealed key limitations:

**Working endpoint:** `https://nest.projectnanda.org` (the NANDA NEST testbed). Other endpoints like `chat.nanda-registry.com:6900` are dead.

**Discovery limitation:** NANDA text search operates only on agent `name` and `description` fields — not on capabilities, scopes, tier, or VC refs. Capability-based routing is not supported.

**Latency:** Average NANDA list-all latency is ~1,400ms — 14× over the <100ms P95 target in the UAI spec. This makes NANDA unsuitable as the primary discovery path for real-time interactions.

**Extension block status:** NANDA NEST ignores all UAI-specific fields in the `uai` extension block. No live NEST agent currently has a `uai` block. The extension schema was designed entirely by the UAI team.

**SDK contribution:** All five bridge functions (publish_agentfacts, register_uai, discover, negotiate, mint_token) are fully greenfield implementations. The NANDA SDK packages contribute nothing to the UAI implementation.

## Federation Strategy

UAI uses a **UAI-first, NANDA-fallback** approach:

```
discover_agents(scope="read:crop_diagnosis")
    │
    ├─ Query UAI Registry → results? → return (scored and filtered)
    │
    └─ Results empty? → fall back to NANDA NEST text search
           │
           └─ assign fixed score 0.5, source="nanda"
```

NANDA results bypass scope/tier/VC filtering (NANDA has no such data), so they always appear at a lower priority than UAI Registry results.

## Publishing to NANDA

The `BridgeAdapter.publish_agentfacts()` method handles dual publishing:

```python
adapter = BridgeAdapter(
    registry_url="http://localhost:8000",
    nanda_endpoint="https://nest.projectnanda.org",
)

await adapter.publish_agentfacts(
    agent_did="did:web:your-agent.gov.in",
    facts={
        "id": "did:web:your-agent.gov.in",
        "name": "Your Agent",
        "description": "...",
        "uai": {
            "uai_registration": {...},
            "delegation_scopes": ["read:your_scope"],
            "risk_tier_max": 1,
        }
    },
    facts_url="https://your-agent.gov.in/.well-known/agentfacts.json",
)
```

**Behavior:**
- UAI Registry update is primary and must succeed
- NANDA publication is secondary and fail-graceful
- NANDA failure does NOT cause the overall operation to fail
- A warning is logged if NANDA publication fails

## Conclusion

For production UAI deployments:

- **UAI Registry** is the authoritative, primary discovery layer
- **NANDA** provides optional global visibility for agents that want to appear in international directories
- Don't build critical latency-sensitive flows that depend on NANDA discovery
- The NANDA fallback in the bridge adapter is a best-effort safety net, not a primary path
