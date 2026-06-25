---
title: "Adapters"
linkTitle: "Adapters"
weight: 5
description: "Bridge Adapter (DID resolution, discovery, negotiation, publishing) and MCP Bridge."
---

**Location:** `adapters/bridge/`

The Bridge Adapter is a Python package that provides the integration layer between global NANDA discovery and the UAI national trust infrastructure. It is used by both the Registry and any third-party agent integration.

## Module Overview

| Module | Class / Function | Description |
|--------|-----------------|-------------|
| `did_resolver.py` | `resolve_did()` | W3C `did:web` resolution with 5-minute caching |
| `signature_verifier.py` | `verify_signature()` | Ed25519 and P-256 signature verification |
| `discover.py` | `discover_agents()` | Registry + NANDA fallback discovery |
| `negotiate.py` | `negotiate_protocol()` | Best endpoint selection from AgentFacts |
| `publish_agentfacts.py` | `BridgeAdapter` class | Validate and publish AgentFacts to Registry + NANDA |
| `mcp_bridge.py` | `MCPBridge` class | MCP ↔ UAI translation |

---

## DID Resolver (`did_resolver.py`)

Resolves `did:web` identifiers to DID Documents per the W3C spec.

**URL mapping:**
```python
"did:web:example.com"               → "https://example.com/.well-known/did.json"
"did:web:example.com:users:alice"   → "https://example.com/users/alice/did.json"
"did:web:localhost"                 → "http://localhost/.well-known/did.json"
"did:web:localhost%3A8005:health"   → "http://localhost:8005/health/did.json"
```

**Caching:** Results are cached in-process for 5 minutes. Clear with `clear_cache()` (called via `POST /admin/clear-did-cache` on the Registry).

```python
from adapters.bridge.did_resolver import resolve_did

doc = await resolve_did("did:web:nafpo.gov.in")
# doc.verification_method[0].public_key_multibase
```

---

## Signature Verifier (`signature_verifier.py`)

Verifies signatures against a resolved DID Document.

**Supported key types:**

| Type | Encoding | Algorithm |
|------|----------|-----------|
| `Ed25519VerificationKey2020` | `publicKeyMultibase` (z-prefix, base58btc + multicodec) | Ed25519 |
| `JsonWebKey2020` | `publicKeyJwk` (P-256 curve) | ECDSA P-256 |

```python
from adapters.bridge.signature_verifier import verify_signature, VerificationResult

result: VerificationResult = verify_signature(
    did_document=doc,
    message=b"register:did:web:agent.gov.in:nonce:abc123",
    signature=base64url_signature,
    verification_method_id="did:web:agent.gov.in#key-1",
)
# result.valid → True/False
# result.reason → explanation if False
```

The `encode_ed25519_public_key_multibase()` helper generates the `z`-prefixed base58btc string from a raw 32-byte Ed25519 public key, for use when building DID Documents.

---

## Agent Discovery (`discover.py`)

Queries the UAI Registry and optionally falls back to NANDA NEST.

```python
from adapters.bridge.discover import discover_agents, DiscoveryQuery, ComplianceFilter

results = await discover_agents(
    registry_url="http://localhost:8000",
    query=DiscoveryQuery(
        scope="read:crop_diagnosis",
        capability="urn:capability:crop-disease-detection:v1",
        page_size=10,
    ),
    compliance=ComplianceFilter(
        tier_max=1,
        data_residency="IN",
    ),
    nanda_endpoint="https://nest.projectnanda.org",  # optional fallback
)
```

**Scoring algorithm:**

| Condition | Score delta |
|-----------|------------|
| Base | 1.0 |
| India data residency | +0.3 |
| Each tier above 1 | −0.2 |
| `facts_ref` present | +0.1 |
| NANDA fallback result | fixed 0.5 |

Results are returned sorted by score (highest first).

**Compliance post-filtering:** `required_vc_types` filters out providers whose `vc_refs` count is less than the required count.

---

## Protocol Negotiation (`negotiate.py`)

Selects the best compatible endpoint from an agent's AgentFacts document.

```python
from adapters.bridge.negotiate import negotiate_protocol, NegotiationResult

result: NegotiationResult = await negotiate_protocol(
    agent_facts=facts_dict,
    preferred_protocols=["uai-native", "mcp"],
)
# result.protocol → "uai-native"
# result.endpoint_uri → "https://provider.example.com/uai/v1/query"
# result.transport → "https"
# result.auth → "bearer"
```

**Priority list (default):** `["uai-native", "mcp"]`

- `uai-native`: transport in `{https, wss}` AND auth in `{bearer, dpop}`
- `mcp`: not yet implemented (logged and skipped)

Raises `NegotiationError` if no compatible endpoint is found.

---

## AgentFacts Publisher (`publish_agentfacts.py`)

The `BridgeAdapter` class wraps the publish workflow:

```python
from adapters.bridge.publish_agentfacts import BridgeAdapter

adapter = BridgeAdapter(
    registry_url="http://localhost:8000",
    nanda_endpoint="https://nest.projectnanda.org",  # optional
)

await adapter.publish_agentfacts(
    agent_did="did:web:your-agent.gov.in",
    facts=agentfacts_dict,
    facts_url="https://your-agent.gov.in/.well-known/agentfacts.json",
)
```

**Validation and write sequence:**
1. `facts["id"] == agent_did` — raises `DIDMismatchError` if not
2. Validate `facts["uai"]` block against schema — raises `AgentFactsValidationError` if invalid
3. PATCH `facts_ref` into Registry record
4. POST to NANDA NEST (fail-graceful — NANDA failure does NOT prevent UAI success)

---

## MCP Bridge (`mcp_bridge.py`)

Translates between MCP tool calls and UAI envelopes.

```python
from adapters.bridge.mcp_bridge import MCPBridge

bridge = MCPBridge(
    registry_url="http://localhost:8000",
    scope_map={
        "diagnose_crop": "read:crop_diagnosis",
        "translate_text": "read:translation",
    }
)

# MCP tool_call → UAI envelope
envelope = bridge.tool_call_to_uai(tool_call={
    "name": "diagnose_crop",
    "arguments": {"crop": "wheat", "symptoms": ["yellowing"]}
})
# envelope.scope → "read:crop_diagnosis"
# envelope.intent → "data-query"
# envelope.payload → {"crop": "wheat", "symptoms": ["yellowing"]}
```

**Scope resolution order:**
1. Explicit `scope_map` lookup
2. `__` convention: `"read__crop_diagnosis"` → `"read:crop_diagnosis"`
3. Dynamic Registry discovery for unmapped tool names

**Intent inference:**
- `"read:"` prefix → `"data-query"`
- `"infer:"` prefix → `"inference"`
- Other → `"data-query"`

**Error handling:**

```python
from adapters.bridge.mcp_bridge import MCPBridgeError

try:
    envelope = bridge.tool_call_to_uai(tool_call)
except MCPBridgeError as e:
    tool_result = bridge.error_to_tool_result(e)
```

---

## Gateway Proxy (`adapters/gateway-proxy/`)

An Nginx/Envoy reverse proxy that wraps legacy government APIs to speak the UAI protocol without any changes to the underlying service. The proxy handles token validation and receipt generation transparently.

**Phase 1 scope:** NAFPO FPO Finder API.

**Not yet fully implemented.** See `adapters/gateway-proxy/README.md` for the planned architecture.
