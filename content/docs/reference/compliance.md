---
title: "Compliance"
linkTitle: "Compliance"
weight: 3
description: "DPDP Act 2023, CERT-In, ISO 27001, and SOC 2 requirements for UAI deployments."
---

## DPDP Act 2023 (Digital Personal Data Protection)

The DPDP Act 2023 is India's foundational law governing personal data processing. UAI implements compliance at the protocol level.

| Requirement | UAI Implementation |
|-------------|-------------------|
| Informed consent | Consent artifacts carry `informed: true` DPDP property |
| Purpose limitation | `purpose` field in consent artifact binds consent to specific use case |
| Data minimization | UAI stores only hashes — never raw payloads |
| Free consent | `free: true` property; consent cannot be conditional on service |
| Specific consent | `specific: true`; consent tied to exact scope and purpose |
| Unambiguous consent | `unambiguous: true`; explicit opt-in required |
| Affirmative action | `affirmative: true`; no implied consent |
| Revocability | `revocable: true`; `revocation_check_url` for Tier 3+ |
| Consent manager alignment | `consent-manager` is a valid `collection_method` |
| Data principal rights | Receipt query access limited to participants |

**Consent manager categories:** Per the DPDP Act, Tier 2 scopes accept `consent-manager` as a collection method, enabling integration with Account Aggregator and other government consent frameworks.

## CERT-In Requirements

The Indian Computer Emergency Response Team (CERT-In) mandates specific operational controls for ICT services.

| Requirement | UAI Implementation |
|-------------|-------------------|
| NTP sync | Sync to NIC/NPL NTP servers required for all nodes |
| Incident reporting | 6-hour reporting window for security incidents |
| Audit log retention | Minimum 180-day rolling retention in Indian jurisdiction |
| Log jurisdiction | `logging.jurisdiction: "IN"` enforced in Tier 2+ scope rules |

**Retention enforcement:** The `logging` block in risk policy rules specifies `min_retention_days: 180` for sensitive scopes. Operational retention enforcement is outside the Audit Ledger — it must be implemented at the infrastructure level (e.g., PostgreSQL data retention policies).

## Tier-Specific Compliance

| Tier | Required Standards |
|------|-------------------|
| 1 | None (public data) |
| 2 | DPDP Act consent, CERT-In logging |
| 3 | DPDP Act (biometric/CM consent), CERT-In, Insurance VC (ISO 27001 implied by VC issuer) |
| 4 | All above + dual-control sign-off, human audit trail |

## Insurance VC (Tier 3+)

Tier 3 providers must hold a `UAIInsuranceCredential` from a recognized regulatory body. This VC is referenced (not verified cryptographically in Phase 1) in the provider's Registry record `vc_refs` field.

**Current VC references (Phase 1 mock):**
- Health corridor: `did:web:abdm.gov.in#vc-insurance-2026` (issued by ABDM / NHA)
- Finance corridor: `did:web:rbi.gov.in#vc-insurance-2026` (issued by RBI)

Phase 2 will implement cryptographic VC verification using Status List 2021.

## Data Residency

For scopes requiring data residency in India, providers declare:

```json
{
  "india_compliance": {
    "dpdp_category": "health",
    "data_residency": "IN"
  }
}
```

The Registry filters by `data_residency` in discovery queries. The bridge adapter scores India-resident providers +0.3 over non-India providers.

## Conformance Testing

The UAI protocol includes a conformance test plan covering six areas:

| Area | Key Tests |
|------|-----------|
| Registry | Schema validation, discovery filtering, ETag/TTL handling, NANDA federation |
| Trust Server | JWT claim presence, tier policy enforcement, consent validation, DPoP (Tier 3), TTL enforcement |
| Provider/Adapter | Offline JWT verification, exact scope enforcement, receipt emission on success AND failure, idempotency |
| Audit Ledger | Immutability (no UPDATE/DELETE), query correctness, access control, 180-day retention |
| IGM | Issue creation linked to receipt, ONDC lifecycle states |
| Security | TLS 1.3, NTP sync, key rotation, 6-hour incident reporting |

Run conformance validation:

```bash
make validate-schemas    # Schema conformance
make test-all            # Functional conformance
```
