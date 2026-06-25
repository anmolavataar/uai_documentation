---
title: "Make Targets"
linkTitle: "Make Targets"
weight: 1
description: "All available make targets in the uai-protocol repository."
---

Run `make {target}` from the project root with the Python virtual environment activated.

## Infrastructure

| Target | Description |
|--------|-------------|
| `make up` | Start Docker infrastructure (Elasticsearch, PostgreSQL, Redis, Kibana) in detached mode |
| `make down` | Stop and remove Docker containers |

## Starting Services

| Target | Port | Description |
|--------|------|-------------|
| `make start-registry` | 8000 | Start the UAI Registry service |
| `make start-audit-ledger` | 8001 | Start the Audit Ledger service |
| `make start-trust-server` | 8002 | Start the Trust Server |
| `make start-toy-provider` | 8003 | Start the Toy Provider reference implementation |
| `make start-mock-consent` | 8004 | Start the Mock Consent service (for Tier 2 testing) |
| `make start-demo-providers` | 8005 | Start all five demo domain providers |

## Seeding and Registration

| Target | Description |
|--------|-------------|
| `make register-demo-providers` | Register all 5 demo providers in the Registry via self-register |
| `make steel-thread` | Run the 7-step end-to-end integration demo |

## Testing

| Target | Description |
|--------|-------------|
| `make test` | Run core test suite (Registry, Bridge, Demo Providers, Trust Server, Mock Consent, Toy Provider). Does not require PostgreSQL. |
| `make test-all` | Run all tests including Audit Ledger. Requires PostgreSQL on port 5433. |
| `make test-integration` | Run integration tests only (MCP flow, Bhashini chain, citizen journey) |
| `make validate-schemas` | Validate all JSON Schema examples — `valid/` must pass, `invalid/` must fail |

## Detailed Commands Behind Each Target

### `make start-registry`
```bash
uvicorn services.registry.main:app --port 8000
```

### `make start-audit-ledger`
```bash
cd services/audit-ledger && PYTHONPATH=../.. uvicorn main:app --port 8001
```

### `make start-trust-server`
```bash
TRUST_SERVER_DID=did:web:localhost%3A8002 \
REGISTRY_URL=http://localhost:8000 \
POLICY_FILE=schemas/uai-risk-policy/valid/multi-corridor.json \
uvicorn services.trust-server.main:app --port 8002
```

### `make start-toy-provider`
```bash
PROVIDER_DID=did:web:localhost%3A8003 \
PROVIDER_ENDPOINT=http://localhost:8003/uai/v1 \
TRUST_SERVER_DID=did:web:localhost%3A8002 \
REGISTRY_URL=http://localhost:8000 \
AUDIT_LEDGER_URL=http://localhost:8001 \
uvicorn services.toy-provider.main:app --port 8003
```

### `make start-demo-providers`
```bash
REGISTRY_URL=http://localhost:8000 \
TRUST_SERVER_DID=did:web:localhost%3A8002 \
AUDIT_LEDGER_URL=http://localhost:8001 \
uvicorn services.demo-providers.main:app --port 8005
```
