---
title: "Consent Artifact"
linkTitle: "Consent Artifact"
weight: 5
description: "Schema for DPDP Act 2023-compliant consent records required for Tier 2+ token issuance."
---

**File:** `schemas/consent-artifact/schema.json`

A consent artifact is the machine-readable proof that a citizen consented to a specific data access. The Trust Server fetches and validates a consent artifact for every Tier 2+ token request.

## Required Fields

| Field | Type | Constraint | Description |
|-------|------|-----------|-------------|
| `consent_id` | string | Must start with `cns_` | Unique identifier for this consent |
| `subject_ref` | string | Must be masked (e.g., `masked_aadhaar_8765`) | Reference to the consenting citizen — never raw PII |
| `seeker_did` | string | Must start with `did:` | DID of the agent requesting the data |
| `provider_did` | string | Must start with `did:` | DID of the agent providing the data |
| `purpose` | string | Machine-readable | Specific purpose string that binds this consent |
| `allowed_scopes` | array | Non-empty | Scope strings this consent covers |
| `issued_at` | string | ISO 8601 | When consent was collected |
| `expires_at` | string | ISO 8601 | When consent expires |
| `collection_method` | enum | See below | How consent was collected |
| `dpdp_properties` | object | All 6 must be `true` | DPDP Act 2023 compliance properties |
| `revocation_status` | string | Must be `"active"` | Current status of this consent |

## Conditional Fields

| Field | Required when |
|-------|--------------|
| `revocation_check_url` | `collection_method` is `biometric` or `manual-approval` |

## Collection Methods

| Value | Tier | Description |
|-------|------|-------------|
| `otp` | 2 | OTP-based consent (e.g., mobile OTP) |
| `biometric` | 3 | Biometric authentication (fingerprint, iris) |
| `consent-manager` | 2–3 | Account Aggregator / consent manager platform |
| `manual-approval` | 4 | Human review and sign-off |

## DPDP Properties

All six DPDP properties must be `true` for the consent to be valid:

```json
{
  "dpdp_properties": {
    "free": true,
    "specific": true,
    "informed": true,
    "unambiguous": true,
    "affirmative": true,
    "revocable": true
  }
}
```

A consent artifact where any property is `false` will fail validation check 5 and result in a `consent_dpdp_invalid` error.

## Revocation

The `revocation_status` field can only be `"active"` in a valid consent artifact. A revoked consent is schema-invalid — meaning the consent store should not serve it over HTTP at all. If a consent has been revoked, `GET {consent_endpoint}/{consent_id}` should return 404 or 410, causing a `consent_not_found` error rather than a `consent_revoked` error.

This design ensures instant, fail-safe revocation: you do not need to wait for a polling cycle.

## Complete Examples

### Tier 2 — OTP Consent (Agriculture)

```json
{
  "consent_id": "cns_valid_tier2_agri",
  "subject_ref": "masked_aadhaar_8765",
  "seeker_did": "did:web:seeker.example.com",
  "provider_did": "did:web:localhost:8005:agriculture",
  "purpose": "pmkisan_benefit_status_check",
  "allowed_scopes": ["read:pmkisan_status"],
  "issued_at": "2026-01-01T00:00:00Z",
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

### Tier 3 — Biometric Consent (Health)

```json
{
  "consent_id": "cns_valid_tier3_health",
  "subject_ref": "abha_xxxx_5678",
  "seeker_did": "did:web:clinic.example.com",
  "provider_did": "did:web:localhost:8005:health",
  "purpose": "diagnostic_result_retrieval_for_treatment",
  "allowed_scopes": ["read:diagnostic_result"],
  "issued_at": "2026-06-01T00:00:00Z",
  "expires_at": "2026-07-01T23:59:59Z",
  "collection_method": "biometric",
  "revocation_check_url": "https://abdm.gov.in/consent/revocation/check",
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

## Invalid Document Examples

The `invalid/` directory tests rejection of:
- `dpdp-property-false.json` — any `dpdp_properties` value is `false`
- `revoked-consent.json` — `revocation_status` is not `"active"`
