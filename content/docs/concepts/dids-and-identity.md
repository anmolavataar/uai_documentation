---
title: "DIDs and Identity"
linkTitle: "DIDs & Identity"
weight: 3
description: "How UAI uses Decentralized Identifiers for agent identity and cryptographic authentication."
---

Every agent in the UAI ecosystem has a **DID (Decentralized Identifier)** as its permanent, cryptographically verifiable identity. UAI uses the `did:web` method for Phase 1.

## DID Format

```
did:web:{domain}[:{path}]
```

**Examples:**

| DID | DID Document URL |
|-----|-----------------|
| `did:web:nafpo.gov.in` | `https://nafpo.gov.in/.well-known/did.json` |
| `did:web:localhost%3A8005:education` | `http://localhost:8005/education/did.json` |
| `did:web:trust.uai.local` | `http://trust.uai.local/.well-known/did.json` |
| `did:web:example.com:users:alice` | `https://example.com/users/alice/did.json` |

**URL mapping rules:**
- Root DID (`did:web:{domain}`) → `{scheme}://{domain}/.well-known/did.json`
- Path DID (`did:web:{domain}:{path}`) → `{scheme}://{domain}/{path}/did.json`
- Port encoding: `%3A` is decoded to `:` before URL construction
- `did:web:localhost` uses `http://` (not `https://`) in development

## DID Document Structure

A DID Document published at `/.well-known/did.json` contains the agent's public key(s):

```json
{
  "@context": ["https://www.w3.org/ns/did/v1"],
  "id": "did:web:example.gov.in",
  "verificationMethod": [
    {
      "id": "did:web:example.gov.in#key-1",
      "type": "Ed25519VerificationKey2020",
      "controller": "did:web:example.gov.in",
      "publicKeyMultibase": "z6Mkf5..."
    }
  ],
  "authentication": ["did:web:example.gov.in#key-1"],
  "assertionMethod": ["did:web:example.gov.in#key-1"]
}
```

## Supported Key Types

| Type | Algorithm | Encoding | Use case |
|------|-----------|----------|---------|
| `Ed25519VerificationKey2020` | Ed25519 | `publicKeyMultibase` (z-prefix, base58btc + multicodec 0xed01) | Primary signing for all UAI agents |
| `JsonWebKey2020` | P-256 (SECP256R1) | `publicKeyJwk` | External compatibility |

The **z-prefix** in `publicKeyMultibase` indicates base58btc encoding with a 2-byte multicodec prefix (`0xed 0x01`) prepended to the 32-byte raw Ed25519 public key.

## Key Rotation (Trust Server)

The Trust Server maintains up to **3 active keys simultaneously**, publishing all of them in its DID Document. This ensures that tokens signed with any recently issued key remain verifiable during the rotation period. Rotate via:

```bash
curl -X POST http://localhost:8002/trust/v1/admin/rotate-key
```

When the 4th key is added, the oldest is removed. Any token signed by the removed key becomes unverifiable — which is intentional, as those tokens will have already expired.

## Proof-of-Control at Registration

When an agent registers in the UAI Registry, it must prove it controls the DID (i.e., holds the corresponding private key):

1. Fetch a nonce: `GET /registry/v1/challenge?did={your_did}`
2. Sign the challenge string `"register:{did}:nonce:{nonce}"` with your Ed25519 private key
3. Submit registration: `POST /registry/v1/agents` with the signed `proof` block

The Registry resolves your DID Document to get your public key, verifies the signature, and only then stores your record.

```json
{
  "did": "did:web:your-agent.gov.in",
  "role": "provider",
  "proof": {
    "type": "Ed25519Signature2020",
    "verificationMethod": "did:web:your-agent.gov.in#key-1",
    "signature": "<base64url-encoded-Ed25519-signature>"
  }
}
```

## DID Caching

The DID resolver in `adapters/bridge/did_resolver.py` caches resolved DID Documents for **5 minutes** to avoid repeated HTTP calls. The Registry exposes an admin endpoint to clear this cache if a DID Document changes:

```bash
curl -X POST http://localhost:8000/admin/clear-did-cache
```

## Offline JWT Verification

Because every Trust Server's public key is published in its DID Document, providers can verify delegation tokens **without any callback to the Trust Server**:

1. Decode the JWT header to extract `kid` (the key ID)
2. Fetch the Trust Server's DID Document from the well-known URL
3. Find the matching verification method by `kid`
4. Verify the Ed25519 signature using that public key
5. Validate standard claims (`aud`, `exp`, `scope`)

This design ensures providers remain operational even if the Trust Server is temporarily unreachable.
