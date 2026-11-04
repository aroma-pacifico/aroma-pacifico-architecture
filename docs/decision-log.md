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
| ASM-002 | 2026-11-04 | SR1 / SR2 | Entries seeded by task A01 on behalf of other tasks carry SR1 / SR2, the A01 owners, as provisional owner. The task that closes an entry replaces the provisional owner with its own role code. | The role code assigned to roadmap#2 to roadmap#6 and roadmap#19 was not available to A01, and the log requires an owner on every entry. Recording the owner as provisional avoids inventing an assignment. | Active |

## Architecture decision records

This section is the index only. Each ADR is written from the [template](adr/template.md) in [`docs/adr/`](adr/), following the [ADR lifecycle](adr/README.md#lifecycle). The file names below are reserved; the files are created by the tasks that write the ADRs.

On 2026-11-04 the owner took the three decisions below. They are recorded with the interim status `Decided by owner, ADR pending`, so that the ADR tasks write the context, alternatives and consequences and move each one to Proposed.

| ID | Date | Owner | Decision | Rationale | Status | Record |
|---|---|---|---|---|---|---|
| ADR-001 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | Primary and second cloud: AWS is the primary cloud; Azure is prepared as the second cloud. | The case statement leaves the choice to the presenter, who must justify it, and requires a second cloud to be prepared without duplicating the platform ("Elige la nube principal y deja preparada una segunda, sin duplicar la plataforma"). The justification is written in the ADR. | Decided by owner, ADR pending | `docs/adr/adr-001-primary-and-second-cloud.md` (reserved), [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2) |
| ADR-002 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | Drone flight mode and governance: combined mode; a contracted certified operator flies; a Federation agronomist authorizes each fumigation. | The case statement requires the design to choose precision, area or combined mode and sustain it, and to state who flies, who authorizes a fumigation, with what permit and how often; fumigation authorization must be a human identity. The permit and frequency are written in the ADR. | Decided by owner, ADR pending | `docs/adr/adr-002-drone-flight-mode-and-governance.md` (reserved), [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) |
| ADR-003 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | IaC tool, remote state and pattern module: Terraform; the pattern module is telemetry ingestion. | Epic 6 requires modules, variables, remote state and an executable plan of one environment, and one module for either ingestion or bag lookup, whichever the diagram leaves as the pattern. The remote state design and the repository layout are written in the ADR. | Decided by owner, ADR pending | `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` (reserved), [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |

## Pending decisions

Each pending decision is an acceptance criterion or a decision that only later work can close. It is closed by the pull request of the named task, which sets the status to Closed and cites the file and section where the criterion is met.

| ID | Date | Owner | Pending decision | Rationale | Status | Closed by |
|---|---|---|---|---|---|---|
| PD-001 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | ADR-001 is written from the template and reaches Accepted. | The decision was taken but not yet justified; the case statement requires the presenter to justify the primary cloud. | Open | [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2) |
| PD-002 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | ADR-002 is written from the template and reaches Accepted, including the permit and the pilot frequency of flights. | The case statement requires the design to state who flies, who authorizes, with what permit and how often, and the defense must sustain the flight mode. | Open | [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) |
| PD-003 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | ADR-003 is written from the template and reaches Accepted, including the remote state design and the IaC repository layout. | Epic 6 requires remote state and an executable plan; the IaC repository layout is decided in ADR-003 (`CLAUDE.md` section 1). | Open | [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |
| PD-004 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | The 2020 legacy table in `docs/legacy-2020.md` records which 2020 requirement is kept and which component is retired. | Challenge point 1 (Gobierno) requires this record ("Registro de qué requisito de 2020 se conserva y qué componente se retira"). | Open | [roadmap#5](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/5) |
| PD-005 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | The questions for the clarification session are prepared in `docs/clarification-questions.md`, and each answer is fed back into this log. | The case statement provides 30 minutes of clarifications before the defense, with the instructor representing the Federation. An answer may validate or invalidate an `ASM-NNN` entry. | Open | [roadmap#6](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/6) |
| PD-006 | 2026-11-04 | SR1 / SR2 (provisional, ASM-002) | The complete diagram references the 2020 legacy table. | The architecture deliverable must carry the record of what is kept and retired from 2020, and the diagram is designed in full. | Open | [roadmap#19](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/19) |

## Reserved paths

The paths below are reserved by this log for the tasks that create them. Until a file exists it is cited in code form, not as a link, so that no link in this log is broken.

| Path | Purpose | Task |
|---|---|---|
| `docs/legacy-2020.md` | Table of 2020 requirements kept and components retired (PD-004, PD-006). | [roadmap#5](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/5) |
| `docs/clarification-questions.md` | Questions for the 30-minute clarification session and their answers (PD-005). | [roadmap#6](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/6) |
| `docs/adr/adr-001-primary-and-second-cloud.md` | ADR-001 (PD-001). | [roadmap#2](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/2) |
| `docs/adr/adr-002-drone-flight-mode-and-governance.md` | ADR-002 (PD-002). | [roadmap#3](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/3) |
| `docs/adr/adr-003-iac-tool-remote-state-and-pattern-module.md` | ADR-003 (PD-003). | [roadmap#4](https://github.com/aroma-pacifico/aroma-pacifico-roadmap/issues/4) |
