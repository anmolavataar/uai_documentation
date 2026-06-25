---
title: "Overview"
linkTitle: "Overview"
weight: 1
description: "Mission, problem statement, and core concepts of the UAI Protocol."
---

## Mission

UAI is India's sovereign Digital Public Infrastructure (DPI) for AI agent interoperability, built by Avataar.ai for MeitY / IndiaAI Mission. Its goal is to let any AI agent — regardless of the platform, ministry, or enterprise that built it — call any other AI agent securely, with citizen consent, and with a tamper-proof record of every interaction.

## The Problem

AI agents built on different platforms cannot interoperate securely today. A government agriculture agent has no trusted mechanism to call a health agent from a different ministry. There is no sovereign trust layer that:

- Lets an agent prove its identity without sharing secrets
- Enforces citizen consent before any sensitive data is accessed
- Provides a tamper-proof, privacy-preserving audit trail compliant with Indian law
- Works across ministries, enterprises, and citizen-facing applications

## How UAI Solves It

**UAI is a control-plane authority — a notary, not a proxy.**

UAI never stores or routes raw payload data. All data flows peer-to-peer (P2P) directly between the Seeker agent and the Provider agent. UAI only ever sees hashes, not payloads.

```
Seeker Agent ──[ token request ]──> Trust Server ──[ issues JWT ]──> Seeker Agent
Seeker Agent ──[ Bearer JWT + P2P call ]──────────────────────────> Provider Agent
Provider Agent ──[ async receipt with hashes ]──────────────────> Audit Ledger
```

## Key Concepts

### Seeker Agent
An AI agent that wants to consume a service or data from a Provider. A citizen-facing chatbot, a ministry's analytics agent, or an enterprise workflow agent can all be Seekers.

### Provider Agent
An AI agent that exposes services — crop diagnosis, health records, credit data, translation, benefit status checks. Providers register in the UAI Registry and declare what scopes they support.

### DID (Decentralized Identifier)
Every agent has a `did:web` identifier (e.g., `did:web:nafpo.gov.in`). The DID Document — published at `/.well-known/did.json` or `/{path}/did.json` — contains the agent's public key(s). All authentication is cryptographic; no passwords are used.

### Delegation Token
A short-lived Ed25519-signed JWT issued by the Trust Server, authorizing a specific Seeker to call a specific Provider for a specific scope. Token lifetime is determined by the trust tier:

| Tier | TTL |
|------|-----|
| Tier 1 | 5 minutes |
| Tier 2 | 2 minutes |
| Tier 3 | 30 seconds |
| Tier 4 | Session only (manual gate) |

### Scope
A fine-grained permission string such as `read:crop_diagnosis` or `write:loan_application`. Every token covers exactly one scope. The Trust Server evaluates scope against the active Risk Policy to determine the tier and consent requirements.

### Corridor
A domain-specific configuration (agriculture, health, finance, education, language) that defines which scopes exist, what tier they require, and what consent or VC requirements apply. Corridors are JSON configuration files — not code changes.

### Receipt
A cryptographically signed, append-only record of every interaction. Contains SHA-256 hashes of the JCS-canonicalized request and response — never raw data.

### Consent Artifact
A DPDP Act 2023-compliant document that proves a citizen gave informed, free, specific, unambiguous, and revocable consent for a Tier 2+ data access.

## Design Principles

1. **Privacy by design** — UAI never stores raw payloads. Only hashes enter the audit ledger.
2. **Citizen sovereignty** — every sensitive access requires a verifiable consent artifact.
3. **Offline verifiability** — providers verify delegation tokens without calling back to the Trust Server, using published DID Documents.
4. **Configuration-driven corridors** — adding a new sector requires a JSON policy update and a provider implementation, not platform changes.
5. **Indian sovereignty** — data residency in India, NTP sync to NIC/NPL, 180-day audit retention, DPDP Act and CERT-In compliance.
