# Clarification Questions for the Q&A Session

| Field | Value |
|---|---|
| Status | Ready for the session |
| Date | 2026-11-04 |
| Owner | SR2 / JR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#6 (A06) |
| Session | 30 minutes of clarifications before the defense; the instructor represents the Federation |
| Decision-log entries | PD-005 (this document), PD-030 (drone permit), ASM-050 to ASM-059 (fallback assumptions created here) |

## 1. Purpose and rules used to select the questions

The case statement grants "30 minutos de aclaraciones" before the defense, with the instructor representing the Federation ("El docente representa a la Federación"). This document is the script for that window. It lists only the ambiguities that remain after reading the [case statement](case/case-statement.pdf), the accepted ADRs ([ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md)), the 2020 legacy table (`docs/legacy-2020.md`, in pull request aroma-pacifico/aroma-pacifico-architecture#11), the addressing plan (`docs/addressing-plan.md`, in pull request aroma-pacifico/aroma-pacifico-architecture#13) and the assumptions and pending decisions of the [decision log](decision-log.md).

Three rules decided what became a question. A question is asked only if the case statement does not already answer it; for example, the alert ranges being configurable by the Federation, the 5-year retention, the three collection centers and "one of the two" pattern modules are stated and are not asked. A question is asked only if the answer can change a design document, a cost or the Epic 6 code; preferences that the case statement leaves to the presenter, such as the primary cloud or the flight mode itself, are presented as decisions to confirm, not reopened. Every question carries the assumption the team will use if the instructor gives no answer, so that no work waits on the session.

Priorities follow the effect on early work. **Critical** questions can change the complete architecture of aroma-pacifico/aroma-pacifico-roadmap#19 or the 100-hour IaC plan of aroma-pacifico/aroma-pacifico-roadmap#20, or reopen an accepted ADR. **Important** questions change one design area, a cost line or a data model, but not the plan. **Nice-to-have** questions refine a template or a document and are asked only if time remains.

## 2. Time budget

The budget keeps the total at 24 minutes, leaving 6 minutes of buffer inside the 30-minute window. Times include stating the question and hearing the answer. If the session starts late or an answer runs long, the moderator (SR2) skips the nice-to-have block first and then the last important questions; the fallback assumption of a skipped question applies unchanged.

| Block | Questions | Minutes | Cumulative minutes |
|---|---|---|---|
| Critical | Q-01 to Q-06 | 13.0 | 13.0 |
| Important | Q-07 to Q-13 | 8.5 | 21.5 |
| Nice-to-have | Q-14 to Q-16 | 2.5 | 24.0 |
| Buffer | none | 6.0 | 30.0 |

How to run the session: SR2 reads each question as written, JR2 writes the answer verbatim in [section 6](#6-answer-record), and nobody debates an answer during the session. Questions that share a topic are grouped so that one answer can close several of them.

## 3. Critical questions

| ID | Topic | Question | Why it matters | Impact if unresolved before roadmap#19 or roadmap#20 | Assumption used if there is no answer | Minutes |
|---|---|---|---|---|---|---|
| Q-01 | Second cloud depth | The case statement asks to leave a second cloud "preparada [...] sin duplicar la plataforma". We read "prepared" as portability and exit: provider-neutral module interfaces, durable data and sack history in open formats, and a declared but undeployed Azure Blob export target, with the RPO of 15 minutes and the RTO of 4 hours met inside one AWS Region. Does the Federation expect anything deployed or coded in the second cloud, such as a warm standby, a copy job or Azure Terraform code? | "Prepared" is not defined, and the answer decides whether a second platform, pipeline or state backend exists. | roadmap#19: the diagram either shows no Azure runtime or adds a recovery path. roadmap#20: any Azure code adds tasks to a 100-hour plan whose Epic 6 estimates already total 98 hours (ADR-003), and would force ADR-001 to be superseded. | ASM-004: portability and exit only; nothing is deployed in Azure; recovery targets are met across Availability Zones of `us-east-1`. | 2.5 |
| Q-02 | Data residency | No AWS Region exists in Colombia, so personal data of farmers and transporters, readings, photos and flight objects would be stored for 5 years and sent to managed inference in US East (N. Virginia), `us-east-1`. Does the Federation accept that this data leaves Colombia, and is there a legal basis or consent model we must state for that international transfer? | The case statement fixes retention and asks which objects leave the country, but gives no residency rule. | roadmap#19: a residency requirement would change the Region of ADR-001, the inference region of PD-013 and the security design. roadmap#20: the Region is a variable in every root stack and the provider configuration. | ASM-002: data may be stored and processed in `us-east-1` for 5 years; the defense states that personal data leaves Colombia. | 1.5 |
| Q-03 | IaC pattern scope | Challenge point 4 says "se entrega el módulo reutilizable de una finca", but the Epic 6 list (network, security, ingestion or sack lookup, one function, variables, pipeline) does not name a farm module. We chose telemetry ingestion as the pattern module and plan the farm module as the connection point between a farm node and that ingestion pattern. Is the reusable farm module part of the 100 hours of Epic 6, or is it a design deliverable outside the code? | Two passages of the case statement disagree on whether the farm module is coded. | roadmap#20: the farm module is one of the 13 Epic 6 tasks; dropping or adding it moves the total either side of the 100-hour limit. roadmap#19: the diagram box that the farm task points to would change. | ASM-050: the reusable farm module is coded inside the 100 hours, as planned in ADR-003, and connects to the ingestion pattern. | 2.5 |
| Q-04 | Executable plan | The code "debe poder generar un plan" of "un ambiente", and Epic 6 also asks for test and production variables and an approval gate before production. Which environment is the one whose plan is evaluated, test or production? Will the Federation provide a cloud account and state bucket for a real `terraform plan`, or is a plan produced with mock providers in plan-mode tests acceptable as the executable plan? | The academic delivery is plan-only and no live account exists, so how the plan is evidenced is open (PD-019). | roadmap#20: decides whether the plan needs an account bootstrap task and credentials, or only plan-mode tests; decides which `.tfvars` file is the primary evidence. Also decides PD-019 for roadmap#33. | ASM-051: the evaluated environment is test (`qa`); production variables and the production approval gate are delivered and validated but not planned against an account; PD-019 stays open until a human provisions an account. | 2.0 |
| Q-05 | Flight mode and operator model | We chose combined mode: area mode delimits farms and estimates harvest, precision mode confirms an alert, fumigates the focus and verifies the same plot; the switch is the pair `crop_alert_raised` then `focus_confirmed` (ADR-002). A contracted certified drone operator flies, and a Federation agronomist authorizes each fumigation with MFA. Can the Federation commit agronomists in each department to authorize orders during campaigns, and does it already contract a drone operator for the three departments, or should the design assume one new contractor? | The case statement requires the design to say who flies and who authorizes, but only the Federation knows whether those people and contracts exist. | roadmap#19: the identity model, the drone service and the case state machine depend on these roles. Cost (roadmap#36): a contracted service versus Federation-owned crews changes the drone campaign line. | ASM-021 and ASM-023: Federation agronomists authorize in each department, and one new contracted operator serves the three departments. | 2.5 |
| Q-06 | Farms outside coverage | The case statement says some pilot farms are outside cellular coverage and their readings and objects enter "sin desplegar una radiobase nueva ni programar una visita semanal". How many of the 200 farms, or which share, are outside coverage, and is a recurring device-borne or satellite path acceptable to the Federation as long as it is not a weekly visit? | The share is not given and drives the field hardware and connectivity mix, which the defense must sustain. | roadmap#19: the connectivity design of roadmap#10 must close the no-signal case; the out-of-coverage path is a separate data path in the diagram. Cost (roadmap#36): the hardware per farm and the monthly fee change with the share. | ASM-052: coverage is a per-farm attribute; the out-of-coverage path is designed and costed per farm, and the cost document states the share it uses as its own assumption. | 2.0 |

## 4. Important questions

| ID | Topic | Question | Why it matters | Impact if unresolved before roadmap#19 or roadmap#20 | Assumption used if there is no answer | Minutes |
|---|---|---|---|---|---|---|
| Q-07 | pH variable | The table of initial ranges includes pH ("Fuera de 4,5 a 6,0"), but the crop variables measured every 4 hours are nitrogen, phosphorus, potassium, humidity and air and soil temperature; pH is not listed. Is pH measured by the farm node, or does it come from a periodic soil analysis entered into the platform? | A pH probe adds fixed hardware to the farm, which the case statement asks to minimize, and changes the telemetry payload. | roadmap#19: the farm node specification and the threshold evaluation. Cost (roadmap#36): hardware per farm. roadmap#20: none in code, because the ingestion store is keyed by node and timestamp, not by variable. | ASM-053: pH is not measured by the farm node; it is recorded from a periodic soil analysis and evaluated against the configurable range. | 1.5 |
| Q-08 | NPK ranges | Nitrogen, phosphorus and potassium alert "fuera del rango que publique la Federación para el piloto". Will the Federation publish those ranges before the defense, and are they set per department, per farm or for the whole pilot, with which units? | The threshold configuration model must hold ranges at the right granularity. | roadmap#19: the threshold data model and the alert path. roadmap#20: none, because thresholds are data read by the function, not code (ADR-003). | ASM-054: NPK ranges are Federation-configured data per pilot, with the unit stored with each range; until a range is published, readings are stored and no NPK alert is raised. | 1.5 |
| Q-09 | Harvest calendar and change freeze | "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha". Which weeks are harvest weeks in Cauca, Huila and Tolima during the pilot? If they differ by department, does the freeze apply to the shared production platform whenever any department is in harvest? | The production pipeline enforces this window (ADR-003; its `pdn` job is plan-only and checks the window at promotion and at approval, ASM-320, ASM-322), and the number of harvest months drives the drone campaigns (ADR-002). | roadmap#20: the window check reads a calendar as pipeline configuration, so code does not wait; only its values do. Cost (roadmap#36): the harvest month cost and H in the campaign formula. | ASM-007, ASM-008 and ASM-024 for time zone, calendar source and harvest months; ASM-055: the freeze applies to production whenever any of the three departments is in harvest week. | 1.5 |
| Q-10 | Drone permit | Aerocivil regulates unmanned aircraft in RAC 100, and its public pages describe a specific category for commercial operations that requires a registered aircraft, a UAS operator certificate and certified pilots. Does the Federation require any permit or certification beyond what Aerocivil requires of the contracted operator, for example an agricultural application license? | The case statement asks "con qué permiso", and the permit fields are part of the fumigation order. | roadmap#19: the permit register and the order fields of the drone design (roadmap#15). | ASM-020 and ASM-022: the operator holds the applicable Aerocivil authorization; non-aviation permits belong to the operator and the Federation. PD-030 stays open until the regulation text is verified. | 1.0 |
| Q-11 | Nodes per farm | Each farm has one crop zone, which may contain several lots, and one drying zone. Is one node per zone (two per farm) acceptable, or does the Federation expect a node per lot? | The node count drives hardware cost and the precision of the focus that a confirmation flight targets. | roadmap#19: the farm node specification and the focus location in the drone flow. Cost (roadmap#36): hardware per farm and device identities. | ASM-056: one crop node and one drying node per farm; the confirmation flight targets the crop node location inside the delimited plot. | 1.0 |
| Q-12 | Port delivery event | The Federation deploys nothing in Buenaventura and the platform only records the sack delivery event, accepted or rejected. Who records that event and its acceptance: the transporter in the app, the shipping line through an integration, or the buyer? | Every sack event stores a responsible person, and the port flow must be closed for the no-signal case. | roadmap#19: the port data path and the identity that writes the last sack state. | ASM-057: the transporter records the delivery and the receiver's acceptance or rejection in the transporter app, with store-and-forward if there is no data; no integration with the shipping line. | 1.0 |
| Q-13 | Buyer access | Point 2 lists a "comprador de solo lectura del saco" identity, while Part C says "La consulta pública lee un saco y muestra solo la vista del comprador". Is the public lookup by sack code anonymous, or must the buyer sign in? | It decides whether the public query tier exposes an unauthenticated endpoint and how it is protected. | roadmap#19: the public lookup front door (PD-051) and the identity design (roadmap#9). | ASM-058: the lookup by sack code is unauthenticated and returns only the buyer view; the read-only buyer identity is used by registered buyers in the web reports, with the same view. | 1.0 |

## 5. Nice-to-have questions

| ID | Topic | Question | Why it matters | Impact if unresolved before roadmap#19 or roadmap#20 | Assumption used if there is no answer | Minutes |
|---|---|---|---|---|---|---|
| Q-14 | Drying closure | For drying humidity the table says "Cierre de lote cerca del 12 %". Is reaching about 12 % a notice to the farmer that the lot can close, or an alert, and what tolerance does "cerca" mean? | The drying flow needs to know whether this value raises an alert or a closure notice. | roadmap#19: only the wording of the drying notice and the SLO scope; no code impact. | ASM-059: it is a lot-closure notice, not an out-of-range alert, sent when the configurable closure value is reached; the tolerance is Federation-configured data. | 1.0 |
| Q-15 | Notice to the Federation | Both release notes templates must show "el aviso a la Federación". Who in the Federation receives the notice, through which channel, and with how much lead time before a change? | The templates must name a recipient and a lead time to be usable. | Release notes templates only; no impact on roadmap#19 or roadmap#20. | No value is assumed: the templates keep the recipient, channel and lead time as mandatory fields of each note, filled by the change owner. | 0.5 |
| Q-16 | Cost currency | Costs are "a precios de ahora". Should they be presented in US dollars from the provider price lists, in Colombian pesos, or both, and at which exchange-rate date? | The cost document must state a currency and a date. | Cost document (roadmap#36) only. | No value is assumed here: under PD-017 the cost document records its currency, exchange-rate source and price date as its own assumptions. | 1.0 |

The time of Q-16 includes 0.5 minutes to close the session by reading back any answer that invalidates an assumption.

## 6. Answer record

JR2 fills this table during the session. After the session, each answer is fed back into the [decision log](decision-log.md): the affected `ASM-NNN` entry changes to Validated or Invalidated, an invalidated assumption that touches an accepted ADR opens a superseding ADR, and PD-005 is closed only when every row below has an answer or is marked "Not asked" or "No answer" with its fallback assumption confirmed.

| ID | Answer, as given | Asked | Decision-log effect |
|---|---|---|---|
| Q-01 | | | ASM-004 |
| Q-02 | | | ASM-002, PD-013 |
| Q-03 | | | ASM-050 |
| Q-04 | | | ASM-051, PD-019 |
| Q-05 | | | ASM-021, ASM-023 |
| Q-06 | | | ASM-052 |
| Q-07 | | | ASM-053 |
| Q-08 | | | ASM-054 |
| Q-09 | | | ASM-007, ASM-008, ASM-024, ASM-055 |
| Q-10 | | | ASM-020, ASM-022, PD-030 |
| Q-11 | | | ASM-056 |
| Q-12 | | | ASM-057 |
| Q-13 | | | ASM-058, PD-051 |
| Q-14 | | | ASM-059 |
| Q-15 | | | none |
| Q-16 | | | PD-017 |

## 7. Coverage of the required topics

| Required topic | Questions |
|---|---|
| Expected depth of the second cloud | Q-01, with data residency in Q-02 |
| Flight mode and operator model | Q-05, with the permit in Q-10 |
| IaC pattern choice | Q-03 and Q-04 |
| Operational ambiguities | Q-06 to Q-09 and Q-11 to Q-16: coverage, pH, NPK ranges published by the Federation, harvest calendar for the change freeze, nodes per farm, port event, buyer access, drying closure, change notice and cost currency |

## 8. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Only source of requirements; every quotation above. |
| [`docs/decision-log.md`](decision-log.md) | Fallback assumptions and pending decisions cited per question. |
| [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | Decisions presented for confirmation in Q-01 to Q-05. |
| `docs/legacy-2020.md` (aroma-pacifico/aroma-pacifico-architecture#11) | Confirms that the 2020 design adds no question beyond the case statement (ASM-040). |
| `docs/addressing-plan.md` (aroma-pacifico/aroma-pacifico-architecture#13) | Public query tier and front door (PD-051) behind Q-13; addressing raises no open question for the Federation. |
| [Aerocivil, Reglamentos Aeronáuticos de Colombia (RAC)](https://www.aerocivil.gov.co/autoridad_aeronautica/normatividad/13-reglamentos-aeronauticos-de-colombia-rac) and [Aerocivil, Drones y Movilidad Urbana Aérea](https://www.aerocivil.gov.co/preguntas-frecuentes/18/drones-y-movilidad-urbana-aerea/) | Basis of the RAC 100 wording in Q-10; the regulation text itself is still to be verified (PD-030). |
