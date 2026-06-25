---
title: "Trust Server"
linkTitle: "Trust Server"
weight: 2
description: "Delegation token issuance, policy evaluation, and consent validation."
---

**Location:** `services/trust-server/`  
**Port:** 8002  
**Backend:** Stateless (in-memory JTI replay cache only)

The Trust Server is the authorization authority of the UAI platform. It issues short-lived, Ed25519-signed JWTs after evaluating the risk policy, optionally validating consent, and optionally verifying DPoP proofs.

## Startup State

On startup, the Trust Server initializes:

| Component | Description |
|-----------|-------------|
| `TrustServerIdentity` | Holds the Trust Server's DID and Ed25519 key pair for signing JWTs |
| `PolicyEvaluator` | Loaded from `POLICY_FILE` env var. Validates policy consistency at load. |
| `RegistryClient` | HTTP client for verifying providers in the Registry |
| `ConsentFetcher` / `ConsentValidator` | HTTP fetch + 5-check consent validation |

## Policy Evaluation

The `PolicyEvaluator` (`services/trust-server/policy/evaluator.py`) handles scope lookup with the following precedence:

1. **Exact match** always wins over any wildcard
2. Among wildcards, **longest prefix** wins
3. Ties (same prefix length) → **highest tier** (most restrictive) wins

At load time, the evaluator validates policy consistency: if a specific scope has a lower tier than its covering wildcard, a `PolicyConflictError` is raised. This prevents accidental policy downgrades.

## Token Issuance Logic

```
POST /trust/v1/token
  │
  ├─ 1. Look up scope in policy → get tier, consent requirements
  ├─ 2. Verify seeker is registered (via Registry)
  ├─ 3. Verify provider is registered and supports the scope
  ├─ 4. If Tier 2+: fetch consent → run 5 checks
  ├─ 5. If Tier 3+: validate DPoP proof, extract JWK thumbprint
  ├─ 6. Build JWT claims
  ├─ 7. Sign with Ed25519 key
  └─ 8. Return {access_token, token_type, expires_in, scope, transaction_id, tier}
```

## DID Document and Key Rotation

The Trust Server publishes its public key(s) at `GET /.well-known/did.json`. This is how providers verify delegation tokens offline.

Key rotation (`POST /trust/v1/admin/rotate-key`):
- Generates a new Ed25519 key pair
- Adds it to the DID Document
- Future tokens are signed with the new key
- Up to 3 keys are maintained simultaneously
- When a 4th key is generated, the oldest is removed

## Consent Validation

`ConsentValidator.validate()` runs five sequential checks:

```python
1. artifact.expires_at > datetime.utcnow()          # Not expired
2. artifact.revocation_status == "active"           # Not revoked
3. request.scope in artifact.allowed_scopes         # Scope match
4. artifact.seeker_did == request.seeker_did        # DID match (seeker)
   AND artifact.provider_did == request.provider_did # DID match (provider)
5. all(v == True for v in artifact.dpdp_properties.values())  # DPDP valid
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `TRUST_SERVER_DID` | `did:web:trust.uai.local` | Trust Server's own DID |
| `REGISTRY_URL` | `http://localhost:8000` | Registry base URL |
| `POLICY_FILE` | `schemas/uai-risk-policy/valid/multi-corridor.json` | Active risk policy JSON file |

## Start Command

```bash
make start-trust-server
# or manually:
TRUST_SERVER_DID=did:web:localhost%3A8002 \
REGISTRY_URL=http://localhost:8000 \
uvicorn services.trust-server.main:app --port 8002
```

## Health Check

```bash
curl http://localhost:8002/.well-known/did.json
# Returns DID Document with active public key(s)
```
