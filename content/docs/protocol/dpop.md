---
title: "DPoP"
linkTitle: "DPoP"
weight: 3
description: "Sender-constraining for Tier 3+ tokens using RFC 9449 DPoP proofs."
---

**DPoP (Demonstrating Proof-of-Possession)** is a mechanism defined in RFC 9449 that binds a token to a specific key pair, making it useless if stolen by a third party. UAI requires DPoP for all Tier 3+ interactions.

## Why DPoP?

A Bearer token alone is a "bearer instrument" — anyone who gets hold of it can use it. For Tier 3 sensitive data (health records, financial transactions), this is insufficient. DPoP adds **sender-constraining**: only the party that holds the private key used to create the DPoP proof can use the token, even if the token itself is intercepted.

## DPoP Proof JWT

A DPoP proof is a short-lived JWT signed with the Seeker's **ephemeral DPoP key pair** (not the agent's identity key). It is included in the `DPoP` header of both the token request and the P2P query.

### DPoP Proof Header

```json
{
  "typ": "dpop+jwt",
  "alg": "EdDSA",
  "jwk": {
    "kty": "OKP",
    "crv": "Ed25519",
    "x": "<base64url-encoded-public-key>"
  }
}
```

The public key is embedded directly in the DPoP proof header so verifiers can extract it without any external lookup.

### DPoP Proof Payload

```json
{
  "jti": "unique-proof-id",
  "htm": "POST",
  "htu": "http://localhost:8002/trust/v1/token",
  "iat": 1719302400,
  "ath": "base64url(SHA-256(raw_bearer_token_bytes))"
}
```

| Claim | Description |
|-------|-------------|
| `jti` | Unique ID for this proof (prevents proof replay) |
| `htm` | HTTP method of the request this proof is for |
| `htu` | Target URI (scheme + host + path, no query string) |
| `iat` | Proof issue time — must be within 30 seconds of now |
| `ath` | `base64url(SHA-256(ASCII(bearer_token)))` — only required when presenting a token to a provider (not needed at the token endpoint) |

## JWK Thumbprint (RFC 7638)

The Trust Server extracts the Seeker's DPoP public key from the proof header and computes its **RFC 7638 JWK thumbprint**:

```python
# Sort required members by key name, serialize to JSON (no whitespace), SHA-256 hash, base64url encode
members = {"crv": "Ed25519", "kty": "OKP", "x": "<pubkey>"}  # sorted alphabetically
thumbprint = base64url(SHA-256(JSON(members)))
```

Required members by key type:
- OKP (Ed25519): `{crv, kty, x}`
- EC (P-256): `{crv, kty, x, y}`
- RSA: `{e, kty, n}`

This thumbprint is embedded as `cnf.jkt` in the issued delegation token.

## At Token Issuance (Trust Server)

When a Seeker requests a Tier 3+ token:

1. Seeker includes `DPoP: {proof}` header with the token request
2. Trust Server validates the DPoP proof:
   - `htm` == `"POST"`
   - `htu` == token endpoint URI (`http://localhost:8002/trust/v1/token`)
   - `jti` not seen before (replay cache)
   - `iat` within 30-second window
3. Trust Server extracts the JWK from the proof header
4. Trust Server computes the RFC 7638 thumbprint
5. Trust Server embeds `cnf: { "jkt": "{thumbprint}" }` in the issued JWT

**Error codes:**

| Code | Meaning |
|------|---------|
| `dpop_required` | Tier 3+ scope requested without `DPoP` header |
| `dpop_invalid` | Proof signature verification failed or missing fields |
| `dpop_proof_expired` | `iat` is more than 30 seconds old |
| `dpop_replay` | `jti` has been used before |

## At P2P Call (Provider)

When the Seeker calls the Provider with a Tier 3+ token:

1. Provider checks that `cnf.jkt` is present in the JWT payload
2. Provider requires a `DPoP` header in the request
3. Provider validates the DPoP proof:
   - `htm` matches the request method (e.g., `"POST"`)
   - `htu` matches the provider endpoint URI (scheme + host + path, no query string)
   - `ath` == `base64url(SHA-256(raw_bearer_token_bytes))`
   - `iat` within 30-second window (10 seconds for Tier 4)
   - JWK thumbprint from the DPoP header == `cnf.jkt` from the JWT
   - DPoP signature is valid
   - `jti` not seen before

The `ath` claim is critical — it binds the DPoP proof to this specific token. Even if an attacker intercepts both the token and a DPoP proof from a different request, the `ath` mismatch will reject it.

## Complete Tier 3 Request Example

```bash
# 1. Generate a DPoP key pair (ephemeral, per-session or per-request)
# 2. Create DPoP proof for the token endpoint
DPOP_TOKEN_PROOF="eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVkRFNBIiwiandrIjp7Li4ufX0..."

# 3. Request a Tier 3 token
RESPONSE=$(curl -X POST http://localhost:8002/trust/v1/token \
  -H "Content-Type: application/json" \
  -H "DPoP: $DPOP_TOKEN_PROOF" \
  -d '{
    "seeker_did": "did:web:seeker.example.com",
    "provider_did": "did:web:localhost%3A8005:health",
    "scope": "read:diagnostic_result",
    "consent_id": "cns_valid_tier3_health"
  }')

ACCESS_TOKEN=$(echo $RESPONSE | jq -r '.access_token')

# 4. Create DPoP proof for the provider endpoint
#    (includes ath = base64url(SHA-256(ACCESS_TOKEN)))
DPOP_QUERY_PROOF="eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVkRFNBIiwiandrIjp7Li4ufX0..."

# 5. Call the provider P2P
curl -X POST http://localhost:8005/health/uai/v1/query \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "DPoP: $DPOP_QUERY_PROOF" \
  -H "Content-Type: application/json" \
  -d '{"patient_id": "PT_001", "record_type": "diagnostic"}'
```

## Implementation Status

{{% alert title="Phase 2" color="info" %}}
DPoP verification at the **provider side** (`dpop_verifier.py`) is pending merge. The Trust Server correctly validates DPoP at token issuance and embeds `cnf.jkt`. Provider-side enforcement is currently in progress on the `20_ddop_verifier_add` branch.
{{% /alert %}}
