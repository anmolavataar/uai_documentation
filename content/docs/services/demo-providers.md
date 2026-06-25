---
title: "Demo Providers"
linkTitle: "Demo Providers"
weight: 4
description: "Five reference domain providers covering agriculture, education, health, finance, and language."
---

**Location:** `services/demo-providers/`  
**Port:** 8005

The Demo Providers service hosts five reference provider implementations in a single FastAPI app using path-based routing. Each provider has its own Ed25519 key pair and DID, but they share the same service process.

## The Five Providers

| Provider | Path Prefix | DID | Max Tier | Scopes |
|----------|-------------|-----|---------|--------|
| Education | `/education/` | `did:web:localhost%3A8005:education` | 2 | `read:curriculum_data` (T1), `read:student_progress` (T2) |
| Agriculture | `/agriculture/` | `did:web:localhost%3A8005:agriculture` | 2 | `read:crop_diagnosis` (T1), `read:pmkisan_status` (T2) |
| Bhashini | `/bhashini/` | `did:web:localhost%3A8005:bhashini` | 1 | `read:translation` (T1) |
| Health | `/health/` | `did:web:localhost%3A8005:health` | 3 | `read:health_profile` (T2), `read:diagnostic_result` (T3) |
| Finance | `/finance/` | `did:web:localhost%3A8005:finance` | 3 | `read:credit_data` (T2), `write:loan_application` (T3) |

## DID Document Resolution

Each provider's DID Document is accessible at its path:

```bash
curl http://localhost:8005/education/did.json
curl http://localhost:8005/agriculture/did.json
curl http://localhost:8005/health/did.json
```

The DID `did:web:localhost%3A8005:education` resolves to `http://localhost:8005/education/did.json` (port-decoded, path appended).

## Registration

Providers self-register by signing their own challenge. The admin endpoint is idempotent:

```bash
# Register all 5 providers at once
make register-demo-providers

# Or individually
curl -X POST http://localhost:8005/education/admin/self-register
curl -X POST http://localhost:8005/agriculture/admin/self-register
curl -X POST http://localhost:8005/health/admin/self-register
curl -X POST http://localhost:8005/finance/admin/self-register
curl -X POST http://localhost:8005/bhashini/admin/self-register
```

Returns `{"status": "registered", "did": "..."}` or `{"status": "already_registered"}` if the DID is already in the Registry.

## Insurance VC Requirement (Health and Finance)

Health and Finance are Tier 3 providers. They require `vc_refs` containing a valid `UAIInsuranceCredential` at registration:

- **Health:** `did:web:abdm.gov.in#vc-insurance-2026`
- **Finance:** `did:web:rbi.gov.in#vc-insurance-2026`

These are currently mock VC references (the VCs themselves are not cryptographically verified in Phase 1).

## Example Requests

### Agriculture — Tier 1 (no consent required)

```bash
# Get a token
TOKEN=$(curl -s -X POST http://localhost:8002/trust/v1/token \
  -H "Content-Type: application/json" \
  -d '{"seeker_did":"did:web:seeker.local","provider_did":"did:web:localhost%3A8005:agriculture","scope":"read:crop_diagnosis"}' \
  | jq -r '.access_token')

# Query the provider
curl -X POST http://localhost:8005/agriculture/uai/v1/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"crop": "wheat", "symptoms": ["yellowing", "spots"]}'
```

### Bhashini — Translation (Tier 1)

```bash
TOKEN=$(curl -s -X POST http://localhost:8002/trust/v1/token \
  -H "Content-Type: application/json" \
  -d '{"seeker_did":"did:web:seeker.local","provider_did":"did:web:localhost%3A8005:bhashini","scope":"read:translation"}' \
  | jq -r '.access_token')

curl -X POST http://localhost:8005/bhashini/uai/v1/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello", "source_lang": "en", "target_lang": "hi"}'
```

The Bhashini provider supports translations from English to: Hindi (`hi`), Tamil (`ta`), Bengali (`bn`), Telugu (`te`), Marathi (`mr`), Kannada (`kn`).

### Agriculture — PMKisan Status (Tier 2 with consent)

```bash
# First, ensure Mock Consent is running and has been seeded
# Then get a Tier 2 token with consent_id
TOKEN=$(curl -s -X POST http://localhost:8002/trust/v1/token \
  -H "Content-Type: application/json" \
  -d '{
    "seeker_did": "did:web:seeker.local",
    "provider_did": "did:web:localhost%3A8005:agriculture",
    "scope": "read:pmkisan_status",
    "consent_id": "cns_valid_tier2_agri"
  }' | jq -r '.access_token')

curl -X POST http://localhost:8005/agriculture/uai/v1/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"farmer_id": "FMR_001", "state": "Maharashtra", "district": "Pune"}'
```

## Agriculture Provider — Registry Integration

The Agriculture provider's `read:pmkisan_status` response demonstrates cross-agent discovery: it performs a live Registry lookup for nearby NAFPO FPO agents matching the requested state/district and includes FPO recommendations in the response.

## Auth and Receipt Boilerplate

All providers share identical JWT verification and receipt generation code via `services/demo-providers/shared/`:

- `jwt_validator.py` — offline JWT verification (fetch Trust Server DID Doc, verify Ed25519)
- `dpop_verifier.py` — DPoP proof verification (Tier 3+, pending merge)
- `receipt_builder.py` — builds and signs UAIReceipt, submits to Audit Ledger async
- `identity.py` — `ProviderIdentity` dataclass holding DID and Ed25519 key pair

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `REGISTRY_URL` | `http://localhost:8000` | Registry base URL |
| `TRUST_SERVER_DID` | `did:web:trust.uai.local` | Trust Server DID for JWT verification |
| `AUDIT_LEDGER_URL` | `http://localhost:8001` | Audit Ledger base URL |

## Start Command

```bash
make start-demo-providers
# or manually:
REGISTRY_URL=http://localhost:8000 \
TRUST_SERVER_DID=did:web:localhost%3A8002 \
AUDIT_LEDGER_URL=http://localhost:8001 \
uvicorn services.demo-providers.main:app --port 8005
```
