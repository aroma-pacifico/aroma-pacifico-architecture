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
| [`docs/addressing-plan.md`](docs/addressing-plan.md) | Cloud addressing plan: CIDRs for `dev`, `qa` and `pdn`, subnet tiers per Availability Zone, and the non-overlap check against the collection centers. |
| [`docs/adr/adr-001-primary-and-second-cloud.md`](docs/adr/adr-001-primary-and-second-cloud.md) | ADR-001: primary cloud and prepared second cloud. |
| [`docs/adr/adr-002-drone-flight-mode-and-governance.md`](docs/adr/adr-002-drone-flight-mode-and-governance.md) | ADR-002: drone flight mode and operational governance. |
| [`docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md`](docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | ADR-003: Terraform, remote state per environment and the telemetry ingestion pattern module. |
| [`docs/legacy-2020.md`](docs/legacy-2020.md) | 2020 legacy: which 2020 requirement is kept, retired or replaced, and which task designs the replacement. |
| [`docs/landing-zone.md`](docs/landing-zone.md) | Landing zone: accounts, root controls, tagging scheme, budgets and audit of the four mandatory events. |
| [`docs/clarification-questions.md`](docs/clarification-questions.md) | Prioritized questions for the 30-minute Q&A session, with impact and fallback assumptions. |
| [`docs/connectivity.md`](docs/connectivity.md) | End-to-end connectivity of farm, wet mill, drone, transporter, collection center and port, including no-signal behavior. |
| [`docs/collection-center-network.md`](docs/collection-center-network.md) | Collection center LAN and Wi-Fi pattern: zones, VLANs, SSIDs, inter-zone policy intent, capacity and non-operational traffic control. |
| [`docs/security-and-identity.md`](docs/security-and-identity.md) | Security and identity: roles and least-privilege matrix, MFA, Cognito directory, device identities, encryption, sack code and buyer view, personal data retention and revocation. |
| [`docs/firewall-policy.md`](docs/firewall-policy.md) | Perimeter firewall of each collection center and cloud firewall policy: rules by group, permanent or temporary validity, and the rules to audit before production. |
| [`docs/drone-service.md`](docs/drone-service.md) | Drone service and fumigation order: the five services as missions, case, order and mission state machines, mission package, flight record, S3 prefixes, inference triggers, ground station endpoints and campaign cadence. |
| [`docs/farm-monitoring.md`](docs/farm-monitoring.md) | Farm monitoring and pilot alerts: crop and drying load, Federation-configured thresholds and their audit, alert deduplication, push and SMS farmer channel, history and pilot report. |
| [`docs/sack-and-trip.md`](docs/sack-and-trip.md) | Sack, trip and public lookup: registration, states and transitions, events with date, time and responsible, code sheet, lookup front door, peak arithmetic, stores, RPO and RTO. |
| [`docs/inference-functions.md`](docs/inference-functions.md) | Event-driven inference functions: the four triggers, managed inference in `us-east-1`, result record, retries, dead-letter queues and observation point, objects that leave Colombia, cost per thousand invocations and objects that contain a person. |
| [`docs/automation-and-pipelines.md`](docs/automation-and-pipelines.md) | Platform, ingestion and flight object reception pipelines: repositories and paths, artifacts, gates, promotion from `dev` to `qa` to `pdn`, approval, notice to the Federation, rollback and secrets. |
| [`docs/release-notes/README.md`](docs/release-notes/README.md) | Release notes: what the two templates are for, who fills them, where a filled note is stored and when it is sent, and the harvest-week restriction with its incident path. |
| [`docs/release-notes/template-application.md`](docs/release-notes/template-application.md) | Blank application release note template: sack lookup, reports web, alerts, inference outcome. |
| [`docs/release-notes/template-infrastructure.md`](docs/release-notes/template-infrastructure.md) | Blank infrastructure release note template: network, firewall, ingestion, drone path. |
| [`docs/release-notes/example-application-crop-alert-recommended-band.md`](docs/release-notes/example-application-crop-alert-recommended-band.md) | Example application release note: recommended band in crop alert pushes. |
| [`docs/release-notes/example-infrastructure-remove-qa-acceptance-path.md`](docs/release-notes/example-infrastructure-remove-qa-acceptance-path.md) | Example infrastructure release note: removal of the temporary `qa` acceptance path (CC-LNK-06, CL-LNK-05). |

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
|   |-- addressing-plan.md
|   |-- automation-and-pipelines.md
|   |-- case/
|   |   `-- case-statement.pdf
|   |-- clarification-questions.md
|   |-- collection-center-network.md
|   |-- connectivity.md
|   |-- decision-log.md
|   |-- drone-service.md
|   |-- farm-monitoring.md
|   |-- firewall-policy.md
|   |-- inference-functions.md
|   |-- landing-zone.md
|   |-- legacy-2020.md
|   |-- release-notes/
|   |   |-- README.md
|   |   |-- example-application-crop-alert-recommended-band.md
|   |   |-- example-infrastructure-remove-qa-acceptance-path.md
|   |   |-- template-application.md
|   |   `-- template-infrastructure.md
|   |-- sack-and-trip.md
|   `-- security-and-identity.md
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
