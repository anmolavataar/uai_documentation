---
title: "Schemas"
linkTitle: "Schemas"
weight: 4
description: "JSON Schema definitions for all UAI protocol data structures."
---

UAI defines five JSON Schemas that form the canonical contracts for all data structures in the protocol. All schemas use **JSON Schema Draft 2020-12**.

| Schema | Purpose |
|--------|---------|
| [Registry Agent Record](/docs/schemas/registry-agent-record/) | Agent registration record stored in the UAI Registry |
| [UAI Receipt](/docs/schemas/uai-receipt/) | Tamper-proof receipt of every P2P interaction |
| [UAI Risk Policy](/docs/schemas/uai-risk-policy/) | Machine-readable policy evaluated by the Trust Server |
| [AgentFacts UAI Extension](/docs/schemas/agentfacts-extension/) | UAI-compliance block for NANDA AgentFacts documents |
| [Consent Artifact](/docs/schemas/consent-artifact/) | DPDP Act 2023-compliant citizen consent record |

Schema files live in the `schemas/` directory of the uai-protocol repository. Each schema directory contains:
- `schema.json` — the JSON Schema
- `README.md` — field-by-field documentation
- `valid/` — example documents that must pass validation
- `invalid/` — example documents that must fail validation (negative tests)

Run `make validate-schemas` to verify all examples against their schemas.
