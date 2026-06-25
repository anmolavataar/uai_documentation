---
title: "UAI Risk Policy"
linkTitle: "Risk Policy"
weight: 3
description: "Schema for the machine-readable policy that governs token issuance."
---

**File:** `schemas/uai-risk-policy/schema.json`

The Risk Policy is a JSON file evaluated by the Trust Server for every token request. One file can cover multiple corridors. Adding new corridors or scopes requires only editing this file and restarting the Trust Server — no code changes.

## Top-Level Fields

| Field | Type | Description |
|-------|------|-------------|
| `policy_id` | string | Unique identifier for this policy (e.g., `uai-risk-policy:agri:2026-04`) |
| `version` | string | Policy version string |
| `scopes` | array | Array of `ScopeRule` objects |

## ScopeRule Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `scope` | string | Yes | Exact scope string, or wildcard ending with `*` |
| `tier` | integer | Yes | Risk tier (1–4) |
| `consent` | object | Yes | Consent requirements |
| `receipts` | object | No | Receipt requirements |
| `logging` | object | No | Audit logging requirements |
| `provider_requirements` | object | No | Provider-side requirements |
| `runtime_controls` | object | No | Human gate and dual control |

### Consent Object

```json
{
  "required": true,
  "allowed_methods": ["otp", "consent-manager"],
  "dpdp_properties_required": ["free", "specific", "informed", "unambiguous", "affirmative", "revocable"],
  "purpose_bound": true
}
```

### Receipts Object

```json
{
  "required": true,
  "payload_hashing": "sha256"
}
```

### Logging Object

```json
{
  "min_retention_days": 180,
  "jurisdiction": "IN"
}
```

### Provider Requirements Object

```json
{
  "required_vc_types": ["UAIInsuranceCredential"],
  "insurance_vc_required": true
}
```

**Constraint:** If `tier >= 3`, `provider_requirements.insurance_vc_required` must be `true`.

### Runtime Controls Object

```json
{
  "human_in_loop": true,
  "dual_control": true
}
```

## Wildcard Scopes

Scope rules support wildcard matching with `*` at the end:

```json
{ "scope": "read:*", "tier": 1 }
```

**Evaluation precedence:**
1. Exact match always beats any wildcard
2. Among wildcards, the longest matching prefix wins
3. Ties (same prefix length) → most restrictive tier wins

**Consistency enforcement:** At policy load time, the Trust Server validates that no specific scope has a lower tier than its covering wildcard. This would create a policy conflict and raises a `PolicyConflictError`.

## Multi-Corridor Policy (Active Configuration)

The default policy file at `schemas/uai-risk-policy/valid/multi-corridor.json` covers all five corridors:

```json
{
  "policy_id": "uai-risk-policy:multi:2026-04",
  "version": "1.0.0",
  "scopes": [
    { "scope": "read:crop_diagnosis", "tier": 1, "consent": { "required": false } },
    { "scope": "read:mandi_prices",   "tier": 1, "consent": { "required": false } },
    { "scope": "read:fpo_info",       "tier": 1, "consent": { "required": false } },
    { "scope": "read:soil_data",      "tier": 1, "consent": { "required": false } },
    { "scope": "read:fpo_data",       "tier": 1, "consent": { "required": false } },
    { "scope": "read:curriculum_data","tier": 1, "consent": { "required": false } },
    { "scope": "read:translation",    "tier": 1, "consent": { "required": false } },
    {
      "scope": "read:pmkisan_status",
      "tier": 2,
      "consent": {
        "required": true,
        "allowed_methods": ["otp", "consent-manager"],
        "dpdp_properties_required": ["free","specific","informed","unambiguous","affirmative","revocable"]
      },
      "logging": { "min_retention_days": 180, "jurisdiction": "IN" }
    },
    {
      "scope": "read:credit_data",
      "tier": 2,
      "consent": { "required": true, "allowed_methods": ["otp", "consent-manager"] }
    },
    {
      "scope": "read:health_profile",
      "tier": 2,
      "consent": { "required": true, "allowed_methods": ["otp", "consent-manager"] }
    },
    {
      "scope": "read:student_progress",
      "tier": 2,
      "consent": { "required": true, "allowed_methods": ["otp", "consent-manager"] }
    },
    {
      "scope": "read:diagnostic_result",
      "tier": 3,
      "consent": { "required": true, "allowed_methods": ["biometric", "consent-manager"] },
      "provider_requirements": { "insurance_vc_required": true, "required_vc_types": ["UAIInsuranceCredential"] }
    },
    {
      "scope": "write:loan_application",
      "tier": 3,
      "consent": { "required": true, "allowed_methods": ["biometric", "consent-manager"] },
      "provider_requirements": { "insurance_vc_required": true },
      "runtime_controls": { "human_in_loop": true }
    }
  ]
}
```

## Changing the Active Policy

Set the `POLICY_FILE` environment variable before starting the Trust Server:

```bash
POLICY_FILE=schemas/uai-risk-policy/valid/agriculture-corridor.json \
  make start-trust-server
```

Only the Trust Server needs to be restarted when changing the policy.
