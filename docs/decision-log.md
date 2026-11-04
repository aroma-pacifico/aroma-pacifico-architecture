# Decision Log

This is the single master log of assumptions, architecture decisions and pending decisions for the Aroma Pacífico pilot. Every choice that the final defense has to justify is traceable to one entry here, and every entry is traceable either to the case statement or to an explicit, numbered assumption.

## Source of requirements

**[`docs/case/case-statement.pdf`](case/case-statement.pdf) is the only source of requirements for this challenge.** No other document, issue, conversation, tutorial or prior design adds a requirement. The 2020 design is input to be reviewed, not a requirement in itself; the case statement says the 2026 update keeps only the 2020 requirements that remain valid ("Se conserva el requisito que siga vigente").

The case statement also allows the team to assume what it leaves open, provided the assumption is written down ("Se puede asumir lo que falte, si el supuesto queda escrito"; "Todo supuesto que no esté en este enunciado queda escrito en la propuesta"). This log is where those assumptions are written. A value, dimension, retention period, measurement or design choice that is not in the case statement must appear here with an ID before any document, diagram or Terraform code uses it. Latency, failover, rollout and recovery figures are recorded as targets or future tests, never as measured results.

## How to use this log

Entries live in three sections, one per ID family. IDs are sequential within a family, are never reused, and are never renumbered; an entry that no longer applies keeps its ID and changes its status.

| ID family | Section | Meaning | Statuses |
|---|---|---|---|
| `ASM-NNN` | [Assumptions](#assumptions) | A fact or value the case statement does not give, which the team adopts so that work can proceed. | Active, Validated, Invalidated, Superseded |
| `ADR-NNN` | [Architecture decision records](#architecture-decision-records) | An index of decisions; the full record lives in [`docs/adr/`](adr/). | Decided by owner, ADR pending; Proposed; Accepted; Superseded (see the [ADR lifecycle](adr/README.md#lifecycle)) |
| `PD-NNN` | [Pending decisions](#pending-decisions) | A decision or acceptance criterion that only later work can close. | Open, Closed |

Every entry has an ID, a date in ISO 8601 form (`YYYY-MM-DD`), an owner, a rationale and a status. Owners are role codes (SR1, SR2, JR1, JR2), never people's names. Roadmap issues are cited as `aroma-pacifico/aroma-pacifico-roadmap#<n>`, abbreviated in the tables below as `roadmap#<n>` with a link to the issue.

## Assumptions

| ID | Date | Owner | Assumption | Rationale | Status |
|---|---|---|---|---|---|
| ASM-001 | 2026-11-04 | SR1 / SR2 | The case statement is written in Spanish. Where a project document renders a passage in English, the original Spanish text of the PDF prevails over the rendering. | The project works in English (`AGENTS.md` section 2), but the PDF is the only source of requirements, so a translation must not be able to change a requirement. | Active |
| ASM-040 | 2026-11-04 | SR1 / JR1 | The 2020 design is known only through what the case statement says about it. The legacy table lists only the 2020 elements that the case statement names, and no other 2020 document, figure or component is used. | The case statement describes the 2020 design in a few sentences and calls its figures, pilot and design class material ("Las cifras, el piloto y el diseño de 2020 son material de clase") ([`docs/legacy-2020.md`](legacy-2020.md)). | Active |
| ASM-041 | 2026-11-04 | SR1 / JR1 | The case statement does not say which mobile network generation the 2020 design relied on. The legacy table retires any dependence on a generation that the operator has switched off or will switch off during the pilot, without naming one; the access technology is chosen by roadmap#10. | The case statement only says that the mobile access network changed since 2020 ("La red de acceso móvil cambió desde 2020") ([`docs/legacy-2020.md`](legacy-2020.md), row L-13). | Active |

## Architecture decision records

This section is the index only. Each ADR is written from the [template](adr/template.md) in [`docs/adr/`](adr/), following the [ADR lifecycle](adr/README.md#lifecycle). The file names below are reserved; the files are created by the tasks that write the ADRs.

On 2026-11-04 the owner took the three decisions below. They are recorded with the interim status `Decided by owner, ADR pending`, so that the ADR tasks write the context, alternatives and consequences and move each one to Proposed.

| ID | Date | Owner | Decision | Rationale | Status | Record |
|---|---|---|---|---|---|---|
| ADR-001 | 2026-11-04 | SR1 / SR2 | Primary and second cloud: AWS is the primary cloud; Azure is prepared as the second cloud. | The case statement leaves the choice to the presenter, who must justify it, and requires a second cloud to be prepared without duplicating the platform ("Elige la nube principal y deja preparada una segunda, sin duplicar la plataforma"). The justification is written in the ADR. | Decided by owner, ADR pending | `docs/adr/adr-001-primary-and-second-cloud.md` (reserved), [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2) |
| ADR-002 | 2026-11-04 | SR1 / SR2 | Drone flight mode and governance: combined mode; a contracted certified operator flies; a Federation agronomist authorizes each fumigation. | The case statement requires the design to choose precision, area or combined mode and sustain it, and to state who flies, who authorizes a fumigation, with what permit and how often; fumigation authorization must be a human identity. The permit and frequency are written in the ADR. | Decided by owner, ADR pending | `docs/adr/adr-002-drone-flight-mode-and-governance.md` (reserved), [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) |
| ADR-003 | 2026-11-04 | SR1 / SR2 | IaC tool, remote state and pattern module: Terraform; the pattern module is telemetry ingestion. | Epic 6 requires modules, variables, remote state and an executable plan of one environment, and one module for either ingestion or bag lookup, whichever the diagram leaves as the pattern. The remote state design and the repository layout are written in the ADR. | Decided by owner, ADR pending | `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` (reserved), [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |

## Pending decisions

Each pending decision is an acceptance criterion or a decision that only later work can close. It is closed by the pull request of the named task, which sets the status to Closed and cites the file and section where the criterion is met.

| ID | Date | Owner | Pending decision | Rationale | Status | Closed by |
|---|---|---|---|---|---|---|
| PD-001 | 2026-11-04 | SR1 / SR2 | ADR-001 is written from the template and reaches Accepted. | The decision was taken but not yet justified; the case statement requires the presenter to justify the primary cloud. | Open | [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2) |
| PD-002 | 2026-11-04 | SR1 / SR2 | ADR-002 is written from the template and reaches Accepted, including the permit and the pilot frequency of flights. | The case statement requires the design to state who flies, who authorizes, with what permit and how often, and the defense must sustain the flight mode. | Open | [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) |
| PD-003 | 2026-11-04 | SR1 / SR2 | ADR-003 is written from the template and reaches Accepted, including the remote state design and the IaC repository layout. | Epic 6 requires remote state and an executable plan; the IaC repository layout is decided in ADR-003 (`CLAUDE.md` section 1). | Open | [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |
| PD-004 | 2026-11-04 | SR1 / JR1 | The 2020 legacy table in `docs/legacy-2020.md` records which 2020 requirement is kept and which component is retired. | Challenge point 1 (Gobierno) requires this record ("Registro de qué requisito de 2020 se conserva y qué componente se retira"). Met in [`docs/legacy-2020.md`](legacy-2020.md#keep-retire-and-replace-table). | Closed | [roadmap#5](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/5) |
| PD-005 | 2026-11-04 | SR2 / JR2 | The questions for the clarification session are prepared in `docs/clarification-questions.md`, cover the second cloud (ADR-001), the flight mode (ADR-002) and the IaC pattern (ADR-003), and each answer is fed back into this log. | The case statement provides 30 minutes of clarifications before the defense, with the instructor representing the Federation. An answer may validate or invalidate an `ASM-NNN` entry or reopen an ADR. | Open | [roadmap#6](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/6) |
| PD-006 | 2026-11-04 | SR1 / SR2 | The complete diagram references the 2020 legacy table. | Criterion of roadmap#5 ("referenced from at least one ADR and from the complete diagram"), which only the complete diagram of roadmap#19 can satisfy. | Open | [roadmap#19](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/19) |
| PD-007 | 2026-11-04 | SR1 / JR1 | At least one ADR references the 2020 legacy table. | Criterion of roadmap#5 ("referenced from at least one ADR"). The ADRs are written by roadmap#2, roadmap#3 and roadmap#4, so the legacy task cannot close it alone. ADR-001, in pull request aroma-pacifico/aroma-pacifico-architecture#8, links `docs/legacy-2020.md` in its Context and References sections; this entry closes when that ADR is merged. | Open | [roadmap#5](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/5) with an ADR from [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2), [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) or [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |
| PD-008 | 2026-11-04 | SR1 / SR2 | The architecture document links the 2020 legacy table and states which 2020 requirements are kept and which components are retired. | Criteria of roadmap#5 ("linked to the architecture document") and roadmap#19; challenge point 1 places this record in the architecture document ("Va en el documento de arquitectura"). | Open | [roadmap#19](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/19) |
| PD-009 | 2026-11-04 | SR1 / JR1 | The cloud CIDRs for development, test and production are chosen, without overlapping 10.41.0.0/16, 10.42.0.0/16 or 10.43.0.0/16. | The case statement fixes the reserved ranges but not the cloud ranges, so the chosen values must be recorded as a decision before the network module of roadmap#24 codes them. | Open | [roadmap#7](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/7) |
| PD-010 | 2026-11-04 | SR1 / JR1 | The transporter position interval and the event that persists when the phone loses data are fixed. | The case statement leaves both to the design ("El diseño fija el intervalo y qué evento queda persistido"); criterion of roadmap#10. | Open | [roadmap#10](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/10) |
| PD-011 | 2026-11-04 | SR2 / JR2 | Each firewall rule is classified as permanent or temporary and the policy table is approved. | Criterion of roadmap#12; roadmap#27 may code only the openings justified by the approved table, and Epic 6 codes the rules marked as permanent. | Open | [roadmap#12](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/12) |
| PD-012 | 2026-11-04 | SR2 / JR2 | The drone review campaign cadence is kept (monthly during harvest, quarterly outside harvest) or a changed cadence is justified, consistently with ADR-002. | The case statement allows the design to change the cadence if it justifies it; criterion of roadmap#15. | Open | [roadmap#15](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/15) |
| PD-013 | 2026-11-04 | SR1 / SR2 | The managed inference service region and the objects that leave the country are decided. | The case statement requires the design to state them ("El diseño indica la región del servicio, qué objetos salen del país"); criterion of roadmap#17. | Open | [roadmap#17](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/17) |
| PD-014 | 2026-11-04 | SR1 / SR2 | The retry policy, the dead-letter queue and the error observation point of the inference functions are defined. | The case statement leaves them to the design ("El diseño define reintentos, cola de fallos y dónde se observa el error"); criterion of roadmap#17, coded by roadmap#30. | Open | [roadmap#17](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/17) |
| PD-015 | 2026-11-04 | SR1 / SR2 | The handling of a drone object that contains a person is defined. | The case statement requires that such an object is not sent to inference and that its treatment is documented; criterion of roadmap#17. | Open | [roadmap#17](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/17) |
| PD-016 | 2026-11-04 | SR2 | The pattern module coded in the IaC repository is only the one approved in ADR-003. | Criterion of roadmap#29 ("implements only the pattern approved in A04, not both"); it cannot close until ADR-003 is Accepted. | Open | [roadmap#29](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/29) |
| PD-017 | 2026-11-04 | JR1 / SR1 | Every cost assumption (prices, hardware, drone campaign, invocations) is registered as an `ASM-NNN` entry and shown separately from the cost data. | Criterion of roadmap#36 ("Assumptions are visible and separate from the cost data"); the case statement says 2020 prices are not a valid quote, so no figure may be used without a recorded source or assumption. | Open | [roadmap#36](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/36) |

## Reserved paths

The paths below are reserved by this log for the tasks that create them. Until a file exists it is cited in code form, not as a link, so that no link in this log is broken.

| Path | Purpose | Task |
|---|---|---|
| [`docs/legacy-2020.md`](legacy-2020.md) | Table of 2020 requirements kept and components retired (PD-004, PD-006, PD-007, PD-008); created. | [roadmap#5](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/5) |
| `docs/clarification-questions.md` | Questions for the 30-minute clarification session and their answers (PD-005). | [roadmap#6](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/6) |
| `docs/adr/adr-001-primary-and-second-cloud.md` | ADR-001 (PD-001). | [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2) |
| `docs/adr/adr-002-drone-flight-mode-and-governance.md` | ADR-002 (PD-002). | [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) |
| `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` | ADR-003 (PD-003). | [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |
