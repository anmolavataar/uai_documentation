---
title: "Environment Variables"
linkTitle: "Env Vars"
weight: 2
description: "All environment variables across UAI services with defaults and descriptions."
---

## Registry (port 8000)

| Variable | Default | Description |
|----------|---------|-------------|
| `ES_INDEX` | `uai-agents` | Elasticsearch index name for agent records |
| `ES_HOST` | `http://localhost:9200` | Elasticsearch connection URL |

## Trust Server (port 8002)

| Variable | Default | Description |
|----------|---------|-------------|
| `TRUST_SERVER_DID` | `did:web:trust.uai.local` | The Trust Server's own DID (used in issued JWTs as `iss`) |
| `REGISTRY_URL` | `http://localhost:8000` | Registry base URL for provider verification |
| `POLICY_FILE` | `schemas/uai-risk-policy/valid/multi-corridor.json` | Path to active risk policy JSON file |

## Audit Ledger (port 8001)

| Variable | Default | Description |
|----------|---------|-------------|
| `PG_DSN` | `postgresql://uai:uai@127.0.0.1:5433/audit_ledger` | PostgreSQL connection string |
| `PG_TABLE` | `receipts` | PostgreSQL table name for receipts |
| `AUDIT_JWT_SECRET` | `dev-secret-change-in-prod` | HS256 secret for authenticating receipt queries |

{{% alert title="Security" color="warning" %}}
`AUDIT_JWT_SECRET` must be changed in production. The default is public and provides no security.
{{% /alert %}}

## Toy Provider (port 8003)

| Variable | Default | Description |
|----------|---------|-------------|
| `PROVIDER_DID` | `did:web:toy-provider.uai.local` | Provider's DID |
| `PROVIDER_ENDPOINT` | `http://localhost:8003/uai/v1` | Provider's public P2P endpoint base URL |
| `TRUST_SERVER_DID` | `did:web:trust.uai.local` | Trust Server DID for offline JWT verification |
| `REGISTRY_URL` | `http://localhost:8000` | Registry base URL for self-registration |
| `AUDIT_LEDGER_URL` | `http://localhost:8001` | Audit Ledger base URL for receipt submission |

## Demo Providers (port 8005)

| Variable | Default | Description |
|----------|---------|-------------|
| `REGISTRY_URL` | `http://localhost:8000` | Registry base URL |
| `TRUST_SERVER_DID` | `did:web:trust.uai.local` | Trust Server DID for offline JWT verification |
| `AUDIT_LEDGER_URL` | `http://localhost:8001` | Audit Ledger base URL for receipt submission |

## Docker Compose Infrastructure

### Elasticsearch
| Variable | Value |
|----------|-------|
| `discovery.type` | `single-node` |
| `xpack.security.enabled` | `false` |
| `ES_JAVA_OPTS` | `-Xms512m -Xmx512m` |

### PostgreSQL
| Setting | Value |
|---------|-------|
| Database | `audit_ledger` |
| User | `uai` |
| Password | `uai` |
| Host port | `5433` (mapped from container port 5432) |

## Development vs Production

In development, all services use `localhost` URLs and the default credentials shown above.

For production:
- Use real domain-based DIDs (`did:web:your-domain.gov.in`)
- Set `TRUST_SERVER_DID` to your actual Trust Server DID
- Change `AUDIT_JWT_SECRET` to a cryptographically random value
- Use a real PostgreSQL DSN with strong credentials
- Set `POLICY_FILE` to your production policy file path
- Ensure TLS 1.3 for all inter-service communication
