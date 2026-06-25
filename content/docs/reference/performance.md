---
title: "Performance Targets"
linkTitle: "Performance"
weight: 4
description: "Latency SLOs for each UAI component per the technical specification."
---

## Component Latency Targets

Per the UAI Technical Specification v1.0, Section 11:

| Component | P95 Target | P99 Target |
|-----------|-----------|-----------|
| NANDA Discovery | <100ms | <300ms |
| Registry Search | <200ms | <800ms |
| Trust Server Token Issuance | <500ms | <2s |
| Provider Response | <800ms | <3s |
| Receipt Submission | <200ms | <500ms |
| **Total Perceived Latency** | **<2s** | **<5s** |

## NANDA Observed Latency

{{% alert title="NANDA Performance Gap" color="warning" %}}
Observed NANDA NEST latency during exploration was ~1,400ms average for list-all operations — 14× over the <100ms P95 target. This is why the bridge adapter uses the UAI Registry as the primary discovery layer and only falls back to NANDA when the Registry returns no results.
{{% /alert %}}

## Token TTL vs Performance

Shorter token TTLs (Tier 3: 30–60 seconds) mean Seekers must request new tokens more frequently. In a session involving multiple P2P calls, the Seeker should:

1. Request a token at the start of the interaction
2. Reuse it for all calls within the TTL window
3. Request a new token when the current one expires or is within a few seconds of expiry

For Tier 3+ with DPoP, the DPoP proof itself must also be fresh (issued within 30 seconds) for each request — even if the token is still valid.

## Async Receipt Submission

Receipt submission is fire-and-forget from the Provider's perspective. The Provider responds to the Seeker first, then submits the receipt in a background task. This removes receipt submission latency from the critical path, keeping total perceived latency within the 2-second P95 target.

## Caching

| Cache | TTL | Content |
|-------|-----|---------|
| DID resolver cache | 5 minutes | DID Documents |
| Registry discovery ETag | Per-response | Discovery result sets |
| JTI replay cache | Token TTL + clock skew | Used DPoP proof JTIs |
| Nonce store | 5 minutes | Registration challenge nonces |

Aggressive DID Document caching is important: the Trust Server fetches provider DID Documents to verify registrations, and providers fetch the Trust Server DID Document to verify tokens. Without caching, every token issuance and every P2P call would incur an extra HTTP round trip.
