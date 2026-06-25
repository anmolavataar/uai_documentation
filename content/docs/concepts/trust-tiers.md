---
title: "Trust Tiers"
linkTitle: "Trust Tiers"
weight: 1
description: "The four risk-based authorization tiers that govern every UAI interaction."
---

Every scope in UAI is assigned a **trust tier** (1–4) that determines the authentication requirements, token lifetime, consent type, and additional controls required before the Trust Server will issue a delegation token.

## Tier Summary

| Tier | Risk | Auth Required | Token TTL | Consent | DPoP | Insurance VC | Human Gate |
|------|------|---------------|-----------|---------|------|-------------|------------|
| 1 | Low | None | 5 min | Not required | No | No | No |
| 2 | Medium | Consent (OTP or consent-manager) | 2 min | Required | No | No | No |
| 3 | High | MFA + Insurance VC | 30 sec | Required (biometric or consent-manager) | Required | Required | No |
| 4 | Critical | Manual + dual-control | Session only | Required (manual-approval) | Required | Required | Yes (2 approvers) |

## Tier 1 — Low Risk (Open Public Data)

Tier 1 covers public or aggregated data that does not require citizen consent. Anyone with a registered DID can call a Tier 1 scope.

**Examples:** `read:crop_diagnosis`, `read:mandi_prices`, `read:fpo_info`, `read:translation`, `read:curriculum_data`

**What the Trust Server checks:**
- Seeker DID is registered in the Registry
- Provider DID is registered and supports the requested scope
- Scope exists in the active risk policy

**No consent artifact required. No DPoP required.**

## Tier 2 — Medium Risk (Personal Data with Consent)

Tier 2 covers personal data access where the citizen has provided verifiable consent through OTP or a consent manager.

**Examples:** `read:pmkisan_status`, `read:land_records`, `read:credit_data`, `read:student_progress`, `read:health_profile`

**Additional requirements beyond Tier 1:**
- A valid `consent_id` must be provided in the token request
- The Trust Server fetches the consent artifact from the Seeker's registered `consent_endpoint`
- The artifact must pass all five DPDP validation checks (see [Consent](/docs/concepts/consent/))
- The Seeker must have a `consent_endpoint` registered in the Registry

**Token TTL is shortened to 2 minutes to limit exposure.**

## Tier 3 — High Risk (Sensitive Data + Proof-of-Possession)

Tier 3 covers highly sensitive data such as health records and financial transactions. It adds sender-constraining via DPoP so that a stolen token cannot be replayed by a different party.

**Examples:** `read:health_records`, `read:diagnostic_result`, `write:loan_application`, `write:insurance_claim`

**Additional requirements beyond Tier 2:**
- The Seeker must send a `DPoP` proof header with the token request
- The Trust Server embeds `cnf.jkt` (JWK thumbprint) in the issued JWT
- The Provider must verify the DPoP proof on every P2P call
- The Provider must have `vc_refs` containing a valid `UAIInsuranceCredential` in its Registry record
- Consent collection method must be `biometric` or `consent-manager` (not just OTP)

**Token TTL is 30 seconds.**

## Tier 4 — Critical Risk (System-Level Operations)

Tier 4 covers irreversible system-level operations. These require a human in the loop with dual-control sign-off from two independent approvers before a token is issued.

**Examples:** `write:system_config`, `admin:*`, `write:benefit_disbursal`

**Additional requirements beyond Tier 3:**
- Human approval gate — no automated issuance
- Dual-control sign-off from two independent approvers
- Consent collection method must be `manual-approval`
- Session-scoped token (no fixed TTL)

{{% alert title="Implementation Status" color="warning" %}}
Tier 4 manual approval workflow is planned for Phase 2. The policy schema supports it, but the Trust Server does not yet implement the human gate.
{{% /alert %}}

## How Tiers Are Assigned

Tiers are assigned per scope in the Risk Policy JSON file — not per agent. The same agent can provide both Tier 1 and Tier 3 scopes.

```json
{
  "scopes": [
    { "scope": "read:crop_diagnosis", "tier": 1, "consent": { "required": false } },
    { "scope": "read:pmkisan_status", "tier": 2, "consent": { "required": true, "allowed_methods": ["otp", "consent-manager"] } },
    { "scope": "read:diagnostic_result", "tier": 3, "consent": { "required": true, "allowed_methods": ["biometric", "consent-manager"] }, "provider_requirements": { "insurance_vc_required": true } }
  ]
}
```

## Provider Ceiling

Every provider declares `risk_tier_max` in its Registry record. The Trust Server will not issue a token for a scope whose tier exceeds the provider's declared ceiling — even if the scope exists in the policy.

```json
{
  "risk_tier_max": 2,
  "supported_scopes": ["read:crop_diagnosis", "read:pmkisan_status"]
}
```

A provider with `risk_tier_max: 2` cannot serve Tier 3 scopes, regardless of what the policy says.
