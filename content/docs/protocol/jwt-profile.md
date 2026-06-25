---
title: "JWT Profile"
linkTitle: "JWT Profile"
weight: 2
description: "Claims reference and validation checklist for UAI delegation tokens."
---

## Token Format

UAI delegation tokens are **JSON Web Tokens (JWT)** signed with **EdDSA (Ed25519)**. They are issued by the Trust Server and presented by the Seeker to the Provider in the `Authorization: Bearer` header.

## JWT Header

```json
{
  "alg": "EdDSA",
  "kid": "did:web:trust.uai.local#key-1",
  "typ": "JWT"
}
```

- `alg`: Always `EdDSA` (Ed25519 curve)
- `kid`: Full DID URL of the signing key in the Trust Server's DID Document

## JWT Payload Claims

| Claim | Type | Required | Description |
|-------|------|----------|-------------|
| `iss` | string | Yes | Trust Server DID (e.g., `did:web:trust.uai.local`) |
| `sub` | string | Yes | Seeker agent DID |
| `aud` | string | Yes | Provider agent DID — token is only valid for this exact provider |
| `iat` | number | Yes | Issued-at timestamp (Unix epoch) |
| `nbf` | number | Yes | Not-before timestamp (same as `iat` in practice) |
| `exp` | number | Yes | Expiry timestamp — determined by tier TTL |
| `jti` | string | Yes | UUID v4 — unique per token, used for replay detection |
| `scope` | string | Yes | Exactly one scope string (e.g., `read:crop_diagnosis`) |
| `txn` | string | Yes | Transaction ID starting with `txn_` |
| `tier` | number | Yes | Trust tier (1–4) |
| `consent_id` | string | Tier 2+ | Consent artifact ID (starting with `cns_`) |
| `cnf.jkt` | string | Tier 3+ | JWK thumbprint (RFC 7638) of the Seeker's DPoP key |

## Token Lifetimes

| Tier | TTL |
|------|-----|
| 1 | 300 seconds (5 minutes) |
| 2 | 120 seconds (2 minutes) |
| 3 | 60 seconds (1 minute) |
| 4 | 30 seconds (session-managed) |

## Example Token

```
eyJhbGciOiJFZERTQSIsImtpZCI6ImRpZDp3ZWI6dHJ1c3QudWFpLmxvY2FsI2tleS0xIiwidHlwIjoiSldUIn0
.
eyJpc3MiOiJkaWQ6d2ViOnRydXN0LnVhaS5sb2NhbCIsInN1YiI6ImRpZDp3ZWI6c2Vla2VyLmV4YW1wbGUuY29tIiwiYXVkIjoiZGlkOndlYjpwcm92aWRlci5leGFtcGxlLmNvbSIsImlhdCI6MTcxOTMwMjQwMCwibmJmIjoxNzE5MzAyNDAwLCJleHAiOjE3MTkzMDI3MDAsImp0aSI6IjU1MGU4NDAwLWUyOWItNDFkNC1hNzE2LTQ0NjY1NTQ0MDAwMCIsInNjb3BlIjoicmVhZDpjcm9wX2RpYWdub3NpcyIsInR4biI6InR4bl9hYmMxMjMiLCJ0aWVyIjoxfQ
.
<Ed25519-signature>
```

Decoded payload:

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
  "tier": 1
}
```

## Provider Validation Checklist

When a Provider receives a Bearer token, it must perform all of the following checks offline (no callback to Trust Server):

**Signature verification:**
- [ ] Decode JWT header → extract `kid`
- [ ] Fetch `{trust_server_base}/.well-known/did.json` (cached up to 5 min)
- [ ] Find the verification method with matching `id`
- [ ] Verify EdDSA signature using the extracted Ed25519 public key

**Claim validation:**
- [ ] `aud` == provider's own DID
- [ ] `exp` > current time (not expired)
- [ ] `iss` == known Trust Server DID
- [ ] `scope` is in the provider's `_SUPPORTED_SCOPES` set
- [ ] `tier` matches the scope's expected tier

**DPoP validation (Tier 3+):**
- [ ] `cnf.jkt` claim is present
- [ ] `DPoP` header is present
- [ ] All DPoP proof checks pass (see [DPoP](/docs/protocol/dpop/))

**Replay prevention:**
- [ ] `jti` has not been seen before (providers maintain a JTI cache)

## Token Transport

- Transport: `Authorization: Bearer {token}` header (RFC 6750)
- Transmission security: TLS 1.3 mandatory
- Tokens must not be logged in plaintext
- Tokens must not be stored on the Seeker side longer than their TTL

## Key Rotation

The Trust Server maintains up to 3 active signing keys simultaneously. During rotation:

1. A new key is generated and added to the DID Document
2. New tokens are signed with the new key
3. Old tokens signed with previous keys remain verifiable (the old key is still in the DID Document)
4. When a 4th key is added, the oldest is removed from the DID Document

Once a key is removed from the DID Document, any token signed with it becomes unverifiable — but those tokens will have already expired due to their short TTLs.
