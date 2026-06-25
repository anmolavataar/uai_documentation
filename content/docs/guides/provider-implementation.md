---
title: "Provider Implementation"
linkTitle: "Provider Implementation"
weight: 3
description: "Build a compliant UAI provider agent from scratch."
---

This guide walks through implementing a UAI-compliant provider. The demo providers in `services/demo-providers/` are the reference implementation — copy and adapt them.

## Provider Anatomy

A UAI provider must:

1. Publish a DID Document at its `/.well-known/did.json` (or path-based equivalent)
2. Self-register in the UAI Registry with proof-of-control
3. Expose `POST /uai/v1/query` that:
   - Verifies the Bearer JWT offline
   - Executes domain logic
   - Returns a structured response
   - Submits a receipt to the Audit Ledger asynchronously

## Step 1 — Identity Setup

Generate an Ed25519 key pair for your provider. Each provider in the demo service has its own identity:

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey

PROVIDER_PRIVATE_KEY = Ed25519PrivateKey.generate()
PROVIDER_DID = "did:web:your-provider.gov.in"
```

The corresponding public key must be published in your DID Document:

```python
from adapters.bridge.signature_verifier import encode_ed25519_public_key_multibase

public_key_bytes = PROVIDER_PRIVATE_KEY.public_key().public_bytes_raw()
public_key_multibase = encode_ed25519_public_key_multibase(public_key_bytes)
```

## Step 2 — DID Document Endpoint

Expose `GET /.well-known/did.json` (or `GET /{path}/did.json` for path-based DIDs):

```python
@router.get("/did.json")
async def get_did_document():
    return {
        "@context": ["https://www.w3.org/ns/did/v1"],
        "id": PROVIDER_DID,
        "verificationMethod": [{
            "id": f"{PROVIDER_DID}#key-1",
            "type": "Ed25519VerificationKey2020",
            "controller": PROVIDER_DID,
            "publicKeyMultibase": public_key_multibase,
        }],
        "authentication": [f"{PROVIDER_DID}#key-1"],
        "assertionMethod": [f"{PROVIDER_DID}#key-1"],
    }
```

## Step 3 — Self-Registration

Implement `POST /admin/self-register` to handle the challenge-sign-register flow:

```python
@router.post("/admin/self-register")
async def self_register(request: Request):
    registry_url = os.getenv("REGISTRY_URL", "http://localhost:8000")

    # 1. Get challenge
    resp = await client.get(f"{registry_url}/registry/v1/challenge", params={"did": PROVIDER_DID})
    challenge_data = resp.json()
    nonce = challenge_data["nonce"]

    # 2. Sign challenge
    challenge_str = f"register:{PROVIDER_DID}:nonce:{nonce}"
    signature = PROVIDER_PRIVATE_KEY.sign(challenge_str.encode())
    signature_b64 = base64.urlsafe_b64encode(signature).decode()

    # 3. Submit registration
    agent_record = {
        "did": PROVIDER_DID,
        "role": "provider",
        "endpoints": [{"uri": f"http://localhost:8005/your-path/uai/v1/query", "transport": "https", "auth": "bearer"}],
        "capabilities": [{"id": "urn:capability:your-capability:v1", "version": "1.0"}],
        "supported_scopes": ["read:your_scope"],
        "risk_tier_max": 1,
        "registered_at": datetime.utcnow().isoformat() + "Z",
        "proof": {
            "type": "Ed25519Signature2020",
            "verificationMethod": f"{PROVIDER_DID}#key-1",
            "signature": signature_b64,
        }
    }

    resp = await client.post(f"{registry_url}/registry/v1/agents", json=agent_record)
    if resp.status_code in (200, 201):
        return {"status": "registered", "did": PROVIDER_DID}
    elif resp.status_code == 409:
        return {"status": "already_registered"}
```

## Step 4 — JWT Verification

Before processing any query, verify the Bearer token offline:

```python
from services.demo_providers.shared.jwt_validator import verify_uai_token

async def verify_request(authorization: str) -> dict:
    token = authorization.removeprefix("Bearer ")
    trust_server_did = os.getenv("TRUST_SERVER_DID", "did:web:trust.uai.local")

    claims = await verify_uai_token(
        token=token,
        expected_audience=PROVIDER_DID,
        trust_server_did=trust_server_did,
        supported_scopes=_SUPPORTED_SCOPES,
    )
    return claims
```

`verify_uai_token` performs:
1. Decode JWT header → get `kid`
2. Fetch Trust Server DID Document (cached 5 min)
3. Find matching verification method
4. Verify Ed25519 signature
5. Check `aud == PROVIDER_DID`
6. Check `exp > now`
7. Check `scope in _SUPPORTED_SCOPES`

## Step 5 — Query Handler

```python
@router.post("/uai/v1/query")
async def handle_query(request: Request, authorization: str = Header(...)):
    claims = await verify_request(authorization)
    scope = claims["scope"]
    txn_id = claims["txn"]

    body = await request.json()

    # Build response
    response_data = await _your_domain_logic(scope, body)

    # Submit receipt asynchronously
    asyncio.create_task(submit_receipt(
        txn_id=txn_id,
        seeker_did=claims["sub"],
        provider_did=PROVIDER_DID,
        trust_server_did=claims["iss"],
        scope=scope,
        tier=claims["tier"],
        token=authorization.removeprefix("Bearer "),
        request_body=body,
        response_body=response_data,
    ))

    return {"transaction_id": txn_id, "scope": scope, "data": response_data}
```

## Step 6 — Receipt Submission

```python
from services.demo_providers.shared.receipt_builder import build_and_submit_receipt

async def submit_receipt(txn_id, seeker_did, provider_did, trust_server_did,
                          scope, tier, token, request_body, response_body):
    audit_url = os.getenv("AUDIT_LEDGER_URL", "http://localhost:8001")
    await build_and_submit_receipt(
        audit_ledger_url=audit_url,
        transaction_id=txn_id,
        seeker_did=seeker_did,
        provider_did=provider_did,
        trust_server_did=trust_server_did,
        scope=scope,
        tier=tier,
        bearer_token=token,
        request_body=request_body,
        response_body=response_body,
        provider_private_key=PROVIDER_PRIVATE_KEY,
    )
```

`build_and_submit_receipt` handles:
- JCS canonicalization of request and response
- SHA-256 hashing
- Building the full receipt structure
- Signing with the provider's Ed25519 key
- `POST /audit/v1/receipts`

## Implementing Domain Logic

The `_your_domain_logic()` function is the only part you need to customize. Replace the mock response with any real implementation:

```python
async def _your_domain_logic(scope: str, body: dict) -> dict:
    if scope == "read:your_scope":
        # Database query, ML inference, API call — anything
        result = await your_database.query(body["input"])
        return {"output": result}
    raise ValueError(f"Unsupported scope: {scope}")
```

The UAI framing (authentication, consent validation, receipt generation) is identical regardless of what your domain logic does.

## Tier 3 Provider Requirements

If your provider serves Tier 3 scopes:

1. Set `risk_tier_max: 3` in your Registry record
2. Include `vc_refs` with a valid `UAIInsuranceCredential` DID URL
3. Set `auth: "dpop"` in your endpoints array
4. Verify DPoP proofs on every query (pending `dpop_verifier.py` merge)

```json
{
  "risk_tier_max": 3,
  "vc_refs": ["did:web:your-regulatory-body.gov.in#vc-insurance-2026"],
  "endpoints": [{"uri": "...", "transport": "https", "auth": "dpop"}]
}
```
