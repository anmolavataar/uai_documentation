---
title: "Services"
linkTitle: "Services"
weight: 5
description: "Detailed documentation for all UAI microservices."
---

The UAI platform consists of six FastAPI microservices and a Python bridge adapter package.

| Service | Port | Role |
|---------|------|------|
| [Registry](/docs/services/registry/) | 8000 | Agent registration and discovery |
| [Trust Server](/docs/services/trust-server/) | 8002 | Token issuance and policy evaluation |
| [Audit Ledger](/docs/services/audit-ledger/) | 8001 | Immutable receipt storage |
| [Demo Providers](/docs/services/demo-providers/) | 8005 | Five reference domain providers |
| [Adapters](/docs/services/adapters/) | — | Bridge adapter and MCP bridge |

**Toy Provider** (port 8003) and **Mock Consent** (port 8004) are development fixtures used in the steel thread and integration tests. They follow the same patterns as Demo Providers.
