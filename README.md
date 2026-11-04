# aroma-pacifico-architecture

Architecture of the Aroma Pacífico pilot: case statement, decision log, ADRs, design documents, 100-hour plan, observability, release notes and costs

Part of the Aroma Pacífico pilot, the 2026 update of the coffee traceability pilot for Cauca, Huila and Tolima. Work is tracked in [aroma-pacifico-roadmap](https://github.com/aroma-pacifico/aroma-pacifico-roadmap) and on the [Aroma Pacífico Pilot board](https://github.com/orgs/aroma-pacifico/projects/1).

## Documentation index

The [case statement](docs/case/case-statement.pdf) is the only source of requirements. Every assumption, architecture decision and pending decision is registered in the [decision log](docs/decision-log.md), and full architecture decision records live in [`docs/adr/`](docs/adr/README.md).

| Document | Purpose |
|---|---|
| [`docs/case/case-statement.pdf`](docs/case/case-statement.pdf) | Case statement; the only source of requirements. |
| [`docs/decision-log.md`](docs/decision-log.md) | Master log of assumptions (`ASM-NNN`), the ADR index (`ADR-NNN`) and pending decisions (`PD-NNN`). |
| [`docs/adr/README.md`](docs/adr/README.md) | ADR lifecycle: Proposed, Accepted, Superseded. |
| [`docs/adr/template.md`](docs/adr/template.md) | Template every ADR copies. |
| [`docs/adr/adr-001-primary-and-second-cloud.md`](docs/adr/adr-001-primary-and-second-cloud.md) | ADR-001: primary cloud and prepared second cloud. |
| [`docs/adr/adr-002-drone-flight-mode-and-governance.md`](docs/adr/adr-002-drone-flight-mode-and-governance.md) | ADR-002: drone flight mode and operational governance. |
| [`docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md`](docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | ADR-003: Terraform, remote state per environment and the telemetry ingestion pattern module. |
| [`docs/legacy-2020.md`](docs/legacy-2020.md) | 2020 legacy: which 2020 requirement is kept, retired or replaced, and which task designs the replacement. |

```text
aroma-pacifico-architecture/
|-- AGENTS.md
|-- CLAUDE.md
|-- CONTRIBUTING.md
|-- GEMINI.md
|-- README.md
|-- docs/
|   |-- adr/
|   |   |-- README.md
|   |   |-- adr-001-primary-and-second-cloud.md
|   |   |-- adr-002-drone-flight-mode-and-governance.md
|   |   |-- adr-003-iac-tool-remote-state-and-pattern-module.md
|   |   `-- template.md
|   |-- case/
|   |   `-- case-statement.pdf
|   |-- decision-log.md
|   `-- legacy-2020.md
`-- standards/
    `-- iac-rules/
```

## What lives here

This repository holds the architecture of the Aroma Pacífico pilot: the case statement, the decision log, architecture decision records, the design documents, the 100-hour IaC plan, observability, release notes and cost estimates. The AI agent rules and standards are synchronized here from the canonical `aroma-pacifico-ai-agents` repository; do not edit them in this copy.

## Tasks delivered here

- [A01](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/1)
- [A02](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2)
- [A03](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3)
- [A04](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4)
- [A05](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/5)
- [A06](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/6)
- [A07](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/7)
- [A08](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/8)
- [A09](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/9)
- [A10](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/10)
- [A11](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/11)
- [A12](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/12)
- [A14](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/14)
- [A15](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/15)
- [A16](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/16)
- [A17](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/17)
- [A18](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/18)
- [A19](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/19)
- [A20](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/20)
- [A21](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/34)
- [A22](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/35)
- [A23](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/36)
