---
title: "Getting Started"
linkTitle: "Getting Started"
weight: 1
description: "Run the full UAI stack locally and complete the steel thread in under 10 minutes."
---

## Prerequisites

- Python 3.11+
- Docker and Docker Compose
- `make`
- Git

## 1. Clone the Repository

```bash
git clone https://github.com/avataar-ai/uai-protocol.git
cd uai-protocol
```

## 2. Create a Python Virtual Environment

```bash
python3 -m venv .venv-registry
source .venv-registry/bin/activate

pip install fastapi "uvicorn[standard]" elasticsearch "pyjwt[crypto]" cryptography \
  httpx jsonschema psycopg2-binary pytest
```

## 3. Start Infrastructure

```bash
make up
```

This starts Elasticsearch (port 9200), PostgreSQL (port 5433), Redis (port 6379), and Kibana (port 5601) via Docker Compose.

Wait approximately 15 seconds for health checks to pass:

```bash
docker compose ps
# All containers should show (healthy)
```

## 4. Start Application Services

Open six terminal tabs (or use a process manager). Run each command in the project root with the venv activated.

**Terminal 1 — Registry:**
```bash
make start-registry
```

**Terminal 2 — Audit Ledger:**
```bash
make start-audit-ledger
```

**Terminal 3 — Trust Server:**
```bash
make start-trust-server
```

**Terminal 4 — Toy Provider:**
```bash
make start-toy-provider
```

**Terminal 5 — Mock Consent (for Tier 2 testing):**
```bash
make start-mock-consent
```

**Terminal 6 — Demo Providers:**
```bash
make start-demo-providers
```

## 5. Verify All Services

```bash
curl http://localhost:8000/registry/v1/discover        # Registry — empty results OK
curl http://localhost:8002/.well-known/did.json        # Trust Server DID Document
curl http://localhost:8003/.well-known/did.json        # Toy Provider DID Document
curl http://localhost:8005/education/did.json          # Demo Providers (Education)
curl http://localhost:8001/docs                        # Audit Ledger OpenAPI
```

## 6. Register Demo Providers

```bash
make register-demo-providers
```

This calls `POST /{provider}/admin/self-register` for all five demo providers (education, agriculture, bhashini, health, finance).

Verify discovery:

```bash
curl "http://localhost:8000/registry/v1/discover?scope=read:crop_diagnosis"
# Should return the agriculture provider record
```

## 7. Run the Steel Thread

The steel thread is a 7-step end-to-end demonstration of the full UAI lifecycle:

```bash
make steel-thread
```

**What it does:**

| Step | Action |
|------|--------|
| 1 | Registers Toy Provider in Registry |
| 2 | Discovers it via Registry |
| 3 | Requests a Tier 1 token from Trust Server |
| 4 | Calls the Provider P2P with the Bearer token |
| 5 | Waits 1 second for async receipt submission |
| 6 | Verifies the receipt in the Audit Ledger |
| 7 | Queries all seeker receipts |

On success you'll see all steps printed with ✓.

## 8. Run Tests

```bash
# Core tests (no PostgreSQL required)
make test

# All tests including Audit Ledger (requires PostgreSQL on port 5433)
make test-all

# Integration tests only
make test-integration

# Validate all JSON Schema examples
make validate-schemas
```

## Tier 2 Quick Test

Ensure Mock Consent is running, then:

```bash
TOKEN=$(curl -s -X POST http://localhost:8002/trust/v1/token \
  -H "Content-Type: application/json" \
  -d '{
    "seeker_did": "did:web:seeker.local",
    "provider_did": "did:web:localhost%3A8005:agriculture",
    "scope": "read:pmkisan_status",
    "consent_id": "cns_valid_tier2_agri"
  }' | jq -r '.access_token')

curl -X POST http://localhost:8005/agriculture/uai/v1/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"farmer_id": "FMR_001", "state": "Maharashtra", "district": "Pune"}'
```

## Stopping Services

```bash
# Stop Docker infrastructure
make down

# Stop individual Python services with Ctrl+C in each terminal
```

## Troubleshooting

**Registry returns 500 on startup:** Elasticsearch isn't ready yet. Wait 15 seconds and retry.

**Trust Server returns `scope_not_in_policy`:** The scope you requested isn't in `multi-corridor.json`. Check spelling, or add the scope to the policy file and restart the Trust Server.

**`consent_not_found`:** Mock Consent service isn't running, or the `consent_id` hasn't been seeded. Run `make start-mock-consent`.

**Audit Ledger won't start:** PostgreSQL container isn't healthy. Run `docker compose ps` and wait for `(healthy)`.

**`provider_not_found`:** Demo providers haven't been registered. Run `make register-demo-providers`.
