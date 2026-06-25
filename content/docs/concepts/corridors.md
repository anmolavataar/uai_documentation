---
title: "Corridors"
linkTitle: "Corridors"
weight: 2
description: "Domain-specific UAI deployments driven entirely by configuration."
---

A **corridor** is a domain-specific deployment of UAI covering a particular sector — agriculture, health, finance, education, or language. Each corridor is defined entirely by a JSON configuration file (the Risk Policy). No code changes to the UAI platform are required to add a new corridor.

## Active Corridors

| Corridor | Key Scopes | Tier Range | Regulatory Context |
|----------|-----------|------------|--------------------|
| Agriculture | `read:crop_diagnosis`, `read:soil_data`, `read:fpo_info`, `read:mandi_prices` (T1); `read:pmkisan_status` (T2) | 1–2 | ICAR, NAFPO FPO records |
| Education | `read:curriculum_data` (T1); `read:student_progress` (T2) | 1–2 | DIKSHA, National Curriculum Framework |
| Health | `read:health_profile` (T2); `read:diagnostic_result` (T3) | 2–3 | ABDM / NHA; requires Insurance VC |
| Finance | `read:credit_data` (T2); `write:loan_application` (T3) | 2–3 | RBI / Account Aggregator; requires Insurance VC |
| Language (Bhashini) | `read:translation` (T1) | 1 | MeitY Bhashini programme |

## Anatomy of a Corridor (Agriculture Example)

The agriculture corridor policy defines six scopes:

```json
{
  "policy_id": "uai-risk-policy:agri:2026-04",
  "version": "1.0.0",
  "scopes": [
    {
      "scope": "read:crop_diagnosis",
      "tier": 1,
      "consent": { "required": false },
      "receipts": { "required": true, "payload_hashing": "sha256" }
    },
    {
      "scope": "read:pmkisan_status",
      "tier": 2,
      "consent": {
        "required": true,
        "allowed_methods": ["otp", "consent-manager"],
        "dpdp_properties_required": ["free", "specific", "informed", "unambiguous", "affirmative", "revocable"]
      },
      "logging": {
        "min_retention_days": 180,
        "jurisdiction": "IN"
      }
    }
  ]
}
```

## How the Trust Server Uses Corridor Policy

When a Seeker requests a token, the Trust Server:

1. Looks up the scope in the active policy file
2. Finds the matching rule (exact match beats wildcard; longest wildcard wins)
3. Reads the tier, consent requirements, and provider requirements from that rule
4. Enforces them before issuing a token

The policy file path is set via the `POLICY_FILE` environment variable. **Restarting only the Trust Server** is sufficient to activate a new policy — Registry, Audit Ledger, and all providers continue running unaffected.

## Corridor Onboarding (30 Minutes)

Adding a new corridor takes four steps:

### Step 1 — Add Scope Entries (2 min)

Edit `schemas/uai-risk-policy/valid/multi-corridor.json` and add your scope rules. Restart the Trust Server. Verify the scope is recognized:

```bash
curl -X POST http://localhost:8002/trust/v1/token \
  -H "Content-Type: application/json" \
  -d '{"seeker_did": "did:web:test.local", "provider_did": "did:web:p.local", "scope": "read:your_new_scope"}'
# Expect 404 (provider not found) — not 403 (scope not in policy)
```

### Step 2 — Implement a Provider Router (10 min)

Copy an existing provider (e.g., `services/demo-providers/education/router.py`) and adapt it:

```python
PROVIDER_KEY: ed25519.Ed25519PrivateKey = None
PROVIDER_DID = "did:web:localhost%3A8005:your-corridor"
_SUPPORTED_SCOPES = {"read:your_new_scope"}

async def _mock_response(scope: str, body: dict) -> dict:
    # Replace with real backend call, DB query, or ML inference
    return {"result": "your data here"}
```

The auth boilerplate (JWT verification, receipt generation, DPoP checking) is identical across all providers — you only implement `_mock_response`.

### Step 3 — Wire into `main.py` (2 min)

```python
from .your_corridor import router as your_router

# In lifespan
app.state.providers["your-corridor"] = ProviderIdentity(
    did=f"did:web:localhost%3A8005:your-corridor",
    private_key=generate_ed25519_key(),
)
app.include_router(your_router, prefix="/your-corridor")
```

### Step 4 — Register and Test (15 min)

```bash
# Register the new provider
curl -X POST http://localhost:8005/your-corridor/admin/self-register

# Verify discovery
curl "http://localhost:8000/registry/v1/discover?scope=read:your_new_scope"

# Get a token
curl -X POST http://localhost:8002/trust/v1/token \
  -d '{"seeker_did":"did:web:seeker.local","provider_did":"did:web:localhost%3A8005:your-corridor","scope":"read:your_new_scope"}'

# Call the provider P2P
curl -X POST http://localhost:8005/your-corridor/uai/v1/query \
  -H "Authorization: Bearer {token}" \
  -d '{"your": "payload"}'
```

## Production DID

In production, use your actual domain:

```
did:web:your-domain.gov.in
```

Publish the DID Document at:

```
https://your-domain.gov.in/.well-known/did.json
```

The DID Document must contain your Ed25519 public key in `Ed25519VerificationKey2020` format with `publicKeyMultibase` encoding.
