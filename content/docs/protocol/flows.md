---
title: "Protocol Flows"
linkTitle: "Flows"
weight: 1
description: "Step-by-step walkthroughs of registration, discovery, token issuance, P2P query, and audit."
---

## 1. Registration Flow (Proof-of-Control)

Before an agent can participate in any UAI interaction, it must register in the Registry and prove it controls its DID.

```
Agent                          Registry (8000)
  │                                 │
  │── GET /registry/v1/challenge ──>│
  │   ?did=did:web:agent.example    │
  │                                 │── store nonce (5 min TTL)
  │<── { nonce, challenge } ────────│
  │                                 │
  │  sign("register:{did}:nonce:{nonce}", privkey)
  │                                 │
  │── POST /registry/v1/agents ────>│
  │   { agent record + proof }      │── verify nonce (not replayed)
  │                                 │── resolve did:web → DID Document
  │                                 │── verify Ed25519 signature
  │                                 │── validate against JSON Schema
  │<── 201 { stored record } ───────│── store in Elasticsearch
```

**Steps:**
1. Agent calls `GET /registry/v1/challenge?did={did}` and receives a one-time nonce (5-minute TTL)
2. Agent signs the string `"register:{did}:nonce:{nonce}"` with its Ed25519 private key
3. Agent calls `POST /registry/v1/agents` with its full record and a `proof` block containing the signature
4. Registry verifies the nonce hasn't been used before, resolves the DID Document, verifies the signature, validates the JSON Schema, and stores the record

For demo providers, `POST /{provider}/admin/self-register` handles all three steps internally. It is idempotent — returns `{"status": "already_registered"}` if the DID is already in the Registry.

---

## 2. Discovery Flow

```
Seeker Agent                   Registry (8000)
     │                              │
     │── GET /registry/v1/discover ─>
     │   ?scope=read:crop_diagnosis  │── Elasticsearch query
     │   &tier_max=1                 │   (AND-combined filters)
     │   &data_residency=IN          │
     │<── { results[], next_cursor }─│
```

**Query parameters:**
| Parameter | Description |
|-----------|-------------|
| `scope` | Filter by supported scope |
| `capability` | Filter by capability URN |
| `tier_max` | Max risk tier (inclusive) |
| `data_residency` | Filter by data residency (`IN`, `EU`, etc.) |
| `page_size` | Results per page |
| `cursor` | Pagination cursor from previous response |

**ETag support:** Include `If-None-Match: {etag}` to get a `304 Not Modified` response when the result set is unchanged.

**Bridge adapter scoring:** When using the bridge adapter's discovery function, results are scored and sorted:
- Base score: 1.0
- +0.3 for India data residency
- −0.2 per tier above 1
- +0.1 for `facts_ref` presence

If the Registry returns no results and `nanda_endpoint` is configured, the adapter falls back to querying the NANDA NEST index (`nest.projectnanda.org`). NANDA results receive a fixed score of 0.5.

---

## 3. Token Issuance Flow

```
Seeker Agent              Trust Server (8002)          Registry (8000)
     │                          │                           │
     │── POST /trust/v1/token ─>│                           │
     │   { seeker_did,          │── GET /discover?did=... ─>│
     │     provider_did,        │<── provider record ───────│
     │     scope,               │                           │
     │     [consent_id],        │── GET {consent_ep}/{id}  (Tier 2+)
     │     [DPoP header] }      │   validate 5 checks
     │                          │
     │                          │   evaluate scope → tier
     │                          │   [verify DPoP proof]   (Tier 3+)
     │                          │   sign JWT (Ed25519)
     │<── { access_token,       │
     │      expires_in,         │
     │      tier,               │
     │      transaction_id } ───│
```

**Policy evaluation:**
- Exact scope match beats any wildcard
- Among wildcards, the longest prefix wins
- Ties are broken by taking the highest (most restrictive) tier

**Token JWT claims:**

```json
{
  "iss": "did:web:trust.uai.local",
  "sub": "did:web:seeker.example.com",
  "aud": "did:web:provider.example.com",
  "iat": 1719302400,
  "nbf": 1719302400,
  "exp": 1719302700,
  "jti": "550e8400-e29b-41d4-a716-446655440000",
  "scope": "read:crop_diagnosis",
  "txn": "txn_abc123",
  "tier": 1,
  "cnf": { "jkt": "thumbprint..." }
}
```

`cnf.jkt` (JWK thumbprint) is only present for Tier 3+ tokens where DPoP was provided.

---

## 4. P2P Query Flow

```
Seeker Agent                         Provider Agent
     │                                    │
     │── POST /uai/v1/query ─────────────>│
     │   Authorization: Bearer {JWT}      │── decode JWT header → kid
     │   [DPoP: {proof}]                  │── fetch Trust Server DID Doc
     │   { query payload }                │── verify Ed25519 signature offline
     │                                    │── check aud == own DID
     │                                    │── check exp
     │                                    │── check scope in supported scopes
     │                                    │── [verify DPoP proof] (Tier 3+)
     │                                    │
     │                                    │── execute domain logic
     │                                    │
     │<── { response data } ──────────────│
     │                                    │
     │                                    │── build receipt (async)
     │                                    │── POST /audit/v1/receipts
     │                                    │   (fire-and-forget)
```

**Provider JWT verification (offline, no callback):**
1. Decode JWT header → get `kid`
2. Fetch `GET {trust_server_base}/.well-known/did.json`
3. Find verification method matching `kid`
4. Verify Ed25519 signature with that public key
5. Check `aud` matches the provider's own DID
6. Check `exp` has not passed
7. Check `scope` is in `_SUPPORTED_SCOPES`

**For Tier 3+ (DPoP):** Also verify the DPoP proof — see [DPoP](/docs/protocol/dpop/).

---

## 5. Audit Receipt Flow

The Provider builds and submits a receipt **after** responding to the Seeker. This is asynchronous and does not block the response.

```
Provider (background task)         Audit Ledger (8001)
        │                               │
        │  JCS-canonicalize(request)    │
        │  SHA-256 → request_hash       │
        │                               │
        │  JCS-canonicalize(response)   │
        │  SHA-256 → response_hash      │
        │                               │
        │  build UAIReceipt             │
        │  sign with provider key       │
        │                               │
        │── POST /audit/v1/receipts ───>│
        │   { receipt }                 │── validate JSON Schema
        │                               │── check duplicate txn_id → 409
        │<── 202 Accepted ──────────────│── INSERT (trigger blocks UPDATE/DELETE)
```

---

## 6. Steel Thread (End-to-End Demo)

The `steel_thread.py` script (`make steel-thread`) demonstrates all five flows in sequence:

| Step | Action |
|------|--------|
| 1 | Register Toy Provider in Registry (`POST /admin/self-register`) |
| 2 | Discover provider via Registry (`GET /registry/v1/discover?scope=read:crop_diagnosis`) |
| 3 | Request Tier 1 token from Trust Server (`POST /trust/v1/token`) |
| 4 | Call Provider P2P with Bearer token (`POST /uai/v1/query`) |
| 5 | Wait 1 second for async receipt submission |
| 6 | Verify receipt in Audit Ledger (`GET /audit/v1/receipts?transaction_id={id}`) |
| 7 | Query all seeker's receipts (`GET /audit/v1/receipts`) |
