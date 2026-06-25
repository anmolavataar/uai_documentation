---
title: "Consent"
linkTitle: "Consent"
weight: 5
description: "DPDP Act 2023-compliant consent artifacts and how UAI validates them."
---

Every Tier 2+ data access in UAI requires a **consent artifact** — a machine-readable document proving that the data principal (citizen) gave informed, free, specific, unambiguous, and revocable consent for this exact data access.

## Legal Basis

UAI consent artifacts are designed to comply with the **Digital Personal Data Protection (DPDP) Act, 2023** (India). The act requires that consent for processing personal data be:

| DPDP Property | Meaning |
|---------------|---------|
| `free` | Not coerced or conditional on a service |
| `specific` | Tied to a specific purpose |
| `informed` | The citizen understands what they are consenting to |
| `unambiguous` | A clear affirmative action (not a default) |
| `affirmative` | Explicit opt-in (not implied) |
| `revocable` | Can be withdrawn at any time |

All six properties must be `true` in every consent artifact UAI validates.

## Consent Artifact Structure

```json
{
  "consent_id": "cns_valid_tier2_agri_001",
  "subject_ref": "masked_aadhaar_8765",
  "seeker_did": "did:web:seeker.example.com",
  "provider_did": "did:web:agriculture.example.gov.in",
  "purpose": "crop_disease_diagnosis_pmkisan_check",
  "allowed_scopes": ["read:pmkisan_status"],
  "issued_at": "2026-06-01T00:00:00Z",
  "expires_at": "2026-12-31T23:59:59Z",
  "collection_method": "otp",
  "dpdp_properties": {
    "free": true,
    "specific": true,
    "informed": true,
    "unambiguous": true,
    "affirmative": true,
    "revocable": true
  },
  "revocation_status": "active"
}
```

**Key field rules:**
- `consent_id` must start with `cns_`
- `subject_ref` must be a **masked** reference (e.g., masked Aadhaar) — never raw PII
- `seeker_did` and `provider_did` must start with `did:`
- `allowed_scopes` is a non-empty array of scope strings
- `collection_method` is one of: `otp`, `biometric`, `consent-manager`, `manual-approval`
- `revocation_status` must be `"active"` (revoked artifacts are schema-invalid)
- `revocation_check_url` is required when `collection_method` is `biometric` or `manual-approval`

## The Five Validation Checks

When the Trust Server receives a token request for a Tier 2+ scope, it fetches the consent artifact from the Seeker's `consent_endpoint` and runs five checks. **All five must pass** or the token request is rejected.

| Check | What is verified | Error code on failure |
|-------|-----------------|----------------------|
| 1. Not expired | `expires_at > now` | `consent_expired` |
| 2. Not revoked | `revocation_status == "active"` | `consent_revoked` |
| 3. Scope match | `requested_scope in allowed_scopes` | `consent_scope_mismatch` |
| 4. DID match | `artifact.seeker_did == request.seeker_did` AND `artifact.provider_did == request.provider_did` | `consent_did_mismatch` |
| 5. DPDP validity | All 6 `dpdp_properties` are `true` | `consent_dpdp_invalid` |

## How Consent Flows

```
Citizen grants consent (OTP / biometric / consent-manager)
        │
        ▼
Consent artifact created with consent_id (e.g., cns_xxx)
        │
        ▼
Seeker stores consent_id, includes it in token request:
  POST /trust/v1/token
  { "seeker_did": "...", "provider_did": "...",
    "scope": "read:pmkisan_status", "consent_id": "cns_xxx" }
        │
        ▼
Trust Server fetches artifact:
  GET {seeker.consent_endpoint}/{consent_id}
        │
        ▼
5-check validation
        │
        ▼
Token issued (if all checks pass)
```

## Consent Endpoint Registration

For a Seeker to use Tier 2+ scopes, it must register a `consent_endpoint` in the UAI Registry. This is the URL the Trust Server will call to fetch consent artifacts:

```json
{
  "did": "did:web:seeker.example.com",
  "role": "seeker",
  "consent_endpoint": "https://seeker.example.com/consent",
  "risk_tier_max": 2
}
```

The Trust Server calls: `GET https://seeker.example.com/consent/{consent_id}`

## Mock Consent Service (Development)

For development and testing, the Mock Consent service (port 8004) serves pre-seeded consent artifacts:

```bash
# The Mock Consent service pre-seeds known consent IDs
# Register your seeker with consent_endpoint = http://localhost:8004/consent

curl http://localhost:8004/consent/cns_valid_tier2_agri
```

The `seed_demo_corridor_policies.py` script populates consent IDs like `cns_valid_tier2_agri`, `cns_valid_tier2_health`, etc.

## Tier-Specific Consent Requirements

| Tier | `collection_method` allowed | Additional requirement |
|------|---------------------------|----------------------|
| 2 | `otp`, `consent-manager` | None |
| 3 | `biometric`, `consent-manager` | `revocation_check_url` required |
| 4 | `manual-approval` | `revocation_check_url` required; human approval gate |

For Tier 3 (`biometric`/`consent-manager`), the Trust Server may check the `revocation_check_url` in real-time to verify the consent has not been withdrawn since it was issued.
