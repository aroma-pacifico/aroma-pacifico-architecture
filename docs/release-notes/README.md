# Release Notes

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | JR2 / SR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#35 (A22) |
| Builds on | [`docs/automation-and-pipelines.md`](../automation-and-pipelines.md) (A18, sections 4.4, 8, 9, 10 and 12), [ADR-003](../adr/adr-003-iac-tool-remote-state-and-pattern-module.md) (`pdn` approval and harvest-week window), [`docs/firewall-policy.md`](../firewall-policy.md) (A12), [`docs/farm-monitoring.md`](../farm-monitoring.md) (A14), [`docs/clarification-questions.md`](../clarification-questions.md) (Q-09, Q-15) |
| Decision-log entries | ASM-180 to ASM-185, PD-180, PD-181; closes PD-174 |

## 1. Purpose and requirements

Challenge point 4 of the [case statement](../case/case-statement.pdf) asks for "Dos plantillas de release notes, llenas con un ejemplo cada una. Aplicación: una salida de la consulta del saco o de las alertas. Infraestructura: un cambio de red, del firewall, de la ingesta o del camino del dron. En ambas se lee qué cambia, la ventana, el impacto, la aprobación, el aviso a la Federación y cómo se vuelve atrás." The same point adds "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente", and evaluation criterion 4 asks that "las plantillas permiten aprobar, avisar y volver atrás". The delivery list repeats "Plantillas de release notes diligenciadas". Quotations are in the original Spanish, which prevails over any English rendering (ASM-001).

This directory delivers those two templates and one completed example of each. The templates do not redesign the pipelines: the promotion, approvers, notice lead time, canary configuration and rollback mechanisms are those of A18, and each template row says whether the change owner or the pipeline fills it. The examples are fictional, describe planned releases and claim no approval, notice, audit, measurement or test result, because the academic delivery is plan-only (`AGENTS.md` section 4).

## 2. Files

| File | Content |
|---|---|
| [`template-application.md`](template-application.md) | Blank template for a change that a user sees: sack lookup, reports web, alert content or channel, inference outcome. |
| [`template-infrastructure.md`](template-infrastructure.md) | Blank template for a change of the network, a firewall, the ingestion or the drone path, including center firewall configuration. |
| [`example-application-crop-alert-recommended-band.md`](example-application-crop-alert-recommended-band.md) | Example: the push of a crop temperature or pH alert shows the recommended band of the case statement's range table. |
| [`example-infrastructure-remove-qa-acceptance-path.md`](example-infrastructure-remove-qa-acceptance-path.md) | Example: removal of the temporary path from the Logística zones to `qa` (CC-LNK-06 and CL-LNK-05) after center acceptance. |

## 3. Which template, and what a note contains

The class of a note follows the case statement's two scopes (ASM-181). A change that a user sees uses the application template; a change of the network, a cloud or center firewall, the ingestion or the drone path uses the infrastructure template. A18 already tells, per pipeline, which changes fall on each side (sections 5 to 7). A release candidate that does both carries two notes, cross-linked through `related_note`, with one window and one notice. A Federation data change, such as a threshold, is not a deployment and needs no note (ASM-179).

| Pipeline (A18) | Application note | Infrastructure note |
|---|---|---|
| Platform | Lookup page and buyer view, reports web, API behavior seen in the apps | Front door, WAF, tables, Cognito pools |
| Ingestion | Alert content or channel | Topic rule, IoT policy, telemetry table, queues |
| Flight object reception | Inference outcome or what the agronomist sees; prompt, schema or model change with the PD-167 result | Drone object bucket, its policy, endpoints, routing |
| Shared foundation and center devices | Not applicable | Network and security stacks, hub route tables and VPN connections, center firewall rules (ASM-184) |

Both templates have the same sections, so a reader finds the same answer in the same place. The case statement's six elements are sections 1 to 6; verification after release, the incident exception, the release record and the retrospective follow.

| Section | Case statement element | Content in both templates | Specific to infrastructure |
|---|---|---|---|
| 1. Change | "qué cambia" | What and why, linked issue and pull request, pipeline, release candidate commit, components, diagram boxes D01 to D13 (PD-180), `qa` acceptance | Rule IDs with validity and audit class, Terraform plan with destroys and replacements first, center device backups |
| 2. Window | "la ventana" | Planned start and end in Colombia time, harvest-week check, expected end before 05:00 | Device changes count in the expected end |
| 3. Impact | "el impacto" | Producers, collection centers, transporters, buyers, the Federation; downtime, degradation, data and schema compatibility (ASM-175, ASM-185) | Buffered data on nodes, ground stations and outboxes |
| 4. Approval | "la aprobación" | Approver roles by stage (ASM-172), approval record from GitHub | Audit of rules by a person other than the author (PD-125) |
| 5. Notice | "el aviso a la Federación" | Recipient role, channel, sent at, lead time required and met, summary, outcome notice | None |
| 6. Rollback | "cómo se vuelve atrás" | Mechanism, trigger, target time (ASM-174), rollback target, deciding role, what cannot be rolled back | Revert commit and center configuration restore (ASM-184) |
| 7. Verification | Not named; needed to know a rollback is required | Checks with expected and observed result | Records before `pdn` go-live: PD-125, PD-167, PD-176 |
| 8. Incident exception | "salvo incidente" | Reference, declaring role, why it cannot wait, shortened notice, due dates (ASM-183) | None |

## 4. Who fills a note and where it is stored

The change owner, the author of the release candidate (one of SR1, SR2, JR1, JR2), copies the template and fills every row marked **Owner** before the notice is sent. The pipeline never writes to a repository (A18 section 4.4), so rows marked **Pipeline** are copied by the change owner from the deployment event of A18 section 12, which GitHub keeps as the deployment status and which is also published to the `pdn` release topic. The `pdn` approver approves the run that the note describes; the platform operator applies center device changes and runs the verification; an auditor role other than the author records the audit of rules (PD-125).

The templates live here, in the architecture repository. A filled note is committed at `release-notes/<release_id>.md` of `aroma-pacifico-infrastructure`, in the pull request of the release candidate, and linked from its body, because that is the pull request the `pdn` job reads. The release record (section 9 of each template) is added in a follow-up pull request within 1 business day of the release (ASM-180). The layout of the infrastructure repository is fixed by B01, which must reserve that directory (PD-181).

## 5. When a note is sent

The procedure below follows the promotion of A18 section 4.4; the steps are sequential.

1. The change owner opens the pull request of the release candidate with its filled note, status `draft`, and with the planned window of section 2 checked against the harvest calendar (ASM-008).
2. When the note is complete, the change owner sends the notice through the channel named under Q-15, records `notice_sent_at` and sets the status to `notice_sent`. For a standard change this happens at least 2 business days, Colombia time, before `planned_start` (ASM-173).
3. After review the pull request is merged; the promotion plans and applies `dev`, then `qa` with its approval and acceptance suite.
4. The `pdn` job reads the note's tables by field key and refuses to run while a mandatory field is empty, the lead time is not met or the window is not allowed. When it starts waiting for approval, the pipeline publishes the note to the `pdn` release topic (ASM-182).
5. A `pdn` approver approves; the job applies and deploys, and any center device change of the note is made by the platform operator in the same window (ASM-184).
6. The pipeline publishes the outcome, including any automatic rollback, to the same topic; the operator runs the verification of section 7.
7. Within 1 business day the change owner adds the release record in a follow-up pull request.

If the job cannot start inside the planned window, the window is rescheduled and the notice is sent again with its full lead time (ASM-182).

## 6. Harvest-week restriction and the incident exception

Both templates and both examples state the restriction at the top and check it in section 2. No `pdn` change runs between 05:00 and 18:00 Colombia time (ASM-007) on a day when any of Cauca, Huila or Tolima is in harvest week (ASM-008, ASM-055), and a job whose expected end, including every canary and bake and every center device change, would pass 05:00 does not start (ASM-178, ASM-184). Plans and the `dev` and `qa` stages are never blocked. Both examples place their window on a harvest-week evening to show the check.

An incident is the only exception. It is declared by SR1 or SR2, on a report from the platform operator, an alarm or the Federation, and only to stop or repair harm that cannot wait until 18:00, never to ship a feature (ASM-183). The incident path of section 8 of each template then applies:

| Step | Rule | Source |
|---|---|---|
| Declaration | Reference `INC-<YYYYMMDD>-<NN>`, declaring role, time and why the change cannot wait | ASM-183 |
| Approval | SR1 or SR2 with the incident reference in the approval; GitHub still prevents self-approval | A18 section 8, ASM-172 |
| Shortened notice | Sent at the decision, before the apply, on the same channel, marked as incident | ASM-173 |
| Completed note | Within 1 business day of the change | ASM-173 |
| Retrospective note | Within 5 business days: what happened, why inside the restricted hours, whether a rollback was needed, follow-up issues | ASM-183 |

## 7. Fields taken from the pipeline (closes PD-174)

PD-174 asked that the templates take from the pipeline the commit, digests, window, approval record, notice fields and rollback mechanism with its target, and keep recipient, channel and lead time mandatory. Every item is a row of both templates.

| PD-174 item | Template row | Filled from |
|---|---|---|
| Commit | `release_candidate_commit` (section 1) | Deployment event |
| Digests | Artifact table (application section 1); `module_tags` and plan table (infrastructure section 1) | `releases.tfvars`, checked by the `CodeSha256` step (A18 section 4.3) |
| Window | `planned_start`, `planned_end`, `harvest_week`, `outside_restricted_hours`, `expected_end_before_0500`, `window_decision` (section 2) | Owner, then the window decision of the deployment event |
| Approval record | `approval_record` (section 4) | GitHub deployment |
| Notice fields | `notice_recipient_role`, `notice_channel`, `notice_sent_at`, `lead_time_required`, `lead_time_met`, `outcome_notice` (section 5); the first five are mandatory | Owner; outcome from the release topic |
| Rollback mechanism and target | `mechanism`, `trigger`, `target_time`, `rollback_target` (section 6) | A18 section 10 and ASM-174; target from the deployment record |

## 8. Compliance with the case statement and the issue

| Passage or acceptance criterion | Where it is met |
|---|---|
| "Aplicación: una salida de la consulta del saco o de las alertas." Criterion: "A release notes template exists for sack lookup or alerts, completed with an example." | [`template-application.md`](template-application.md); [`example-application-crop-alert-recommended-band.md`](example-application-crop-alert-recommended-band.md) (alerts) |
| "Infraestructura: un cambio de red, del firewall, de la ingesta o del camino del dron." Criterion: "A release notes template exists for network, firewall, ingestion or the drone path, completed with an example." | [`template-infrastructure.md`](template-infrastructure.md); [`example-infrastructure-remove-qa-acceptance-path.md`](example-infrastructure-remove-qa-acceptance-path.md) (firewall and network) |
| "qué cambia, la ventana, el impacto, la aprobación, el aviso a la Federación y cómo se vuelve atrás." Criterion: "Both templates include the change, window, impact, approval, notice to the Federation and rollback." | Sections 1 to 6 of both templates and both examples; section 3 above |
| "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." Criterion: "The restriction ... is reflected." | Header and section 2 of both templates and both examples; section 8 of both templates; section 6 above |
| Criterion 4 of the evaluation: "las plantillas permiten aprobar, avisar y volver atrás." | Sections 4, 5 and 6 of both templates, checked by the `pdn` job (section 5 above) |

## 9. Decision-log entries

| ID | Summary |
|---|---|
| ASM-180 | Templates here; filled notes in the release-candidate pull request of the infrastructure repository; release record in a follow-up pull request. |
| ASM-181 | Application template for user-visible changes, infrastructure template for network, firewall, ingestion and drone path; two notes when both. |
| ASM-182 | The `pdn` job reads the note by field key; the owner sends the notice; the pipeline publishes the note and the outcome; a missed window needs a new notice. |
| ASM-183 | Who declares an incident, what qualifies, the reference format and the 5-business-day retrospective. |
| ASM-184 | Hub and center firewall changes are production changes; center rules are applied by the platform operator with backups and restored from them. |
| ASM-185 | Readers ignore unknown attributes, so an additive attribute is safe in both directions of a rollback. |
| PD-180 | Diagram box catalog D01 to D13, to fill `diagram_boxes` in the examples. |
| PD-181 | B01 reserves `release-notes/` in the infrastructure repository; whether the note check is coded is recorded with PD-175. |
| PD-174 | Closed by this directory. |

## 10. References

| Reference | Use |
|---|---|
| [`docs/decision-log.md`](../decision-log.md) | ASM-180 to ASM-185, PD-180, PD-181, PD-174; ASM-007, ASM-008, ASM-055, ASM-172 to ASM-179 used as written. |
| [`docs/automation-and-pipelines.md`](../automation-and-pipelines.md) | Promotion, approval, notice, rollback and the deployment event the templates read. |
| [`docs/clarification-questions.md`](../clarification-questions.md) | Q-15: recipient, channel and lead time stay mandatory fields. |
| [`docs/firewall-policy.md`](../firewall-policy.md) | Rule IDs, temporary rules and their audit used by the infrastructure example. |
| [`docs/farm-monitoring.md`](../farm-monitoring.md) | Threshold rows, alert message and dispatcher used by the application example. |
