# Example Release Note: Application, Crop Alert Recommended Band

> **EXAMPLE ONLY.** This note is a completed copy of the [application template](template-application.md) for a fictional change. It describes a planned release: the academic delivery is plan-only, nothing has been deployed, no approval has been given, no notice has been sent and no check has been run. Commit SHAs, digests, versions and the issue and pull request numbers are obvious placeholders. Dates are written relative to a release day `D`, never as a real date. A filled note would live at `release-notes/<release_id>.md` of `aroma-pacifico-infrastructure` (ASM-180).

> **Harvest-week restriction.** "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." Day `D` is a harvest-week day in this example, so the window is placed in the evening and must end before 05:00 of `D+1` (section 2).

## 0. Summary

| Field | Value | Source |
|---|---|---|
| `release_id` * | `RN-EXAMPLE-A01` | Owner |
| `class` | Application | Owner |
| `change_type` * | `standard` | Owner |
| `title` * | Show the recommended band in the push notification of a crop temperature or pH alert | Owner |
| `change_owner_role` * | JR2 | Owner |
| `status` | `draft` (example; never released) | Owner |
| `related_note` | Not applicable: no network, firewall, ingestion or drone path resource changes; only two function packages | Owner |

## 1. Change

**What changes and why.** Today the push notification of a crop alert says which range the reading left, for example "soil temperature 31 C, range 14-30 C". The case statement's table of initial ranges also gives a recommended band for temperature ("Recomendado entre 17 °C y 23 °C") and pH ("Recomendado entre 4,9 y 5,6"), which the farm monitoring design already draws in the history chart (ASM-134) but not in the alert. After this change the threshold function adds the recommended band of the threshold row to the alert message, and the notification dispatcher appends it to the push text, so the farmer sees where to bring the value back without opening the history. The SMS text is unchanged, to keep its length and per-message cost (A14 section 7.1, PD-133).

| Field | Value | Source |
|---|---|---|
| `issue` * | `aroma-pacifico/aroma-pacifico-roadmap#<example-issue>` (placeholder; no such issue exists) | Owner |
| `pull_request` * | `aroma-pacifico-infrastructure#<example-pr>` (placeholder), bumping two entries of `stacks/workload/releases.tfvars` | Owner |
| `pipeline` * | `ingestion` (A18 section 6): `functions/threshold/**` and `functions/notification-dispatcher/**` | Owner |
| `release_candidate_commit` * | `0000000000000000000000000000000000example` (placeholder) | Pipeline |
| `components` * | Threshold function (A17); SNS alert topic message, one added attribute `recommended_range`; notification dispatcher, push rendering only (A14 section 7.3). Rows T-01 (pH), T-04 and T-05 (air and soil temperature) of A14 section 4.1 carry a band; the other rows have none and the attribute is absent. | Owner |
| `diagram_boxes` | D09 (telemetry ingestion: threshold function and SNS alert topic) and D10 (event-driven function pattern), from the catalog in [architecture](../architecture.md) section 14 | Owner |
| `user_visible_change` * | Push, before: "Soil temperature 31 C, out of range 14-30 C, measured 14:02." Push, after: "Soil temperature 31 C, out of range 14-30 C, recommended 17-23 C, measured 14:02." Values are illustrative; the text is rendered in Spanish by the design system. | Owner |
| `qa_acceptance` * | Not run (plan-only). In a real release: the ingestion suite of ASM-177 with an out-of-range soil temperature reading, an out-of-range moisture reading (no band) and a drained reading, checking one push per episode with and without the band. | Pipeline |
| `model_evaluation` | Not applicable: no prompt, schema or model identifier changes | Owner |

| Artifact | From version | To version | SHA-256 digest | Source |
|---|---|---|---|---|
| `threshold` | `1.3.0` (example) | `1.4.0` (example) | `sha256:example0000000000000000000000000000000000000000000000000000001` | Owner, checked by Pipeline |
| `notification-dispatcher` | `1.2.0` (example) | `1.3.0` (example) | `sha256:example0000000000000000000000000000000000000000000000000000002` | Owner, checked by Pipeline |

## 2. Window

| Field | Value | Source |
|---|---|---|
| `planned_start` * | `D 20:00` Colombia time | Owner |
| `planned_end` * | `D 22:00` Colombia time | Owner |
| `harvest_week` * | `yes`: in this example at least one department is in harvest week on `D` (ASM-055) | Owner |
| `outside_restricted_hours` * | `yes`: 20:00 to 22:00 does not touch 05:00 to 18:00 | Owner |
| `expected_end_before_0500` * | `yes`: one apply and two canary deployments of `LambdaCanary10Percent5Minutes` with the configured bake end well before 05:00 of `D+1`; the pipeline computes the expected end from its configuration (ASM-178) | Owner, checked by Pipeline |
| `window_decision` | Not run (plan-only) | Pipeline |

## 3. Impact

| Audience | Affected | How, and what they must do |
|---|---|---|
| Producers (farmers) | Yes | The push of a new crop temperature or pH alert is one phrase longer. SMS, reminders, resolutions and the app screens are unchanged. Nothing to do. |
| Collection centers | No | No alert reaches a center. |
| Transporters | No | Not on the alert path. |
| Buyers | No | The buyer view of ASM-096 has no alert data. |
| The Federation | Yes | Its recommended values (Federation data, A14 section 4.2) become farmer-facing text. The notice asks the Federation to confirm that the recommended bands of T-01, T-04 and T-05 are the ones it wants farmers to read. |

| Field | Value | Source |
|---|---|---|
| `expected_downtime` * | None. The `live` alias moves by canary; ingestion and alerting keep running. | Owner |
| `expected_degradation` * | During the 5-minute canary about 10 % of threshold batches run the new version, so a push may arrive with or without the band. Alert latency should not change: the band travels in the message and the dispatcher makes no new read (hop 6 budget of A10 section 8.2). Not measured; the alert-delivery SLO of A21 observes it. | Owner |
| `data_and_schema_compatibility` * | Additive: one optional attribute in the alert message (ASM-175). The previous dispatcher ignores it and the new dispatcher renders without it when absent (ASM-185), so each function can roll back alone. No table, queue or topic changes. | Owner |
| `installed_apps` * | Unaffected: push text is rendered by the phone's operating system; app outboxes carry no alert data. | Owner |

## 4. Approval

| Field | Value | Source |
|---|---|---|
| `qa_approver_role` | One of SR1, SR2, JR1 other than the author JR2 (ASM-172) | Approver |
| `pdn_approver_role` * | SR1 or SR2 listed in `PDN_APPROVERS`, not the person who starts the promotion run (ASM-321) and not the author (ASM-172) | Owner states the role; Approver |
| `approval_record` | None: no approval has been given for this example | Pipeline |
| `additional_review` | Not applicable: no buyer view, WAF or code format change | Owner |

## 5. Notice to the Federation

| Field | Value | Source |
|---|---|---|
| `notice_recipient_role` * | The Federation role to be named under Q-15 (PD-171). A real note cannot pass the `pdn` check until it is named. | Owner |
| `notice_channel` * | The channel to be named under Q-15, plus publication of this note to the `pdn` release topic of A18 section 9 | Owner |
| `notice_sent_at` * | Planned: `D-3` business days, 10:00 Colombia time. Not sent (example). | Owner |
| `lead_time_required` * | 2 business days before `planned_start` (ASM-173) | Owner |
| `lead_time_met` * | `yes` if sent as planned: 3 business days is more than 2 | Owner, checked by Pipeline |
| `notice_summary` * | "From day D, the push of a crop temperature or pH alert will also show the recommended band that the Federation configured. The change is made between 20:00 and 22:00 Colombia time, outside the harvest-week restricted hours; SMS and the app are unchanged. If an error alarm fires the previous version is restored automatically, and any wrong push is followed by a correction on the same channel." | Owner |
| `outcome_notice` | Not run (plan-only) | Pipeline |

## 6. Rollback

| Field | Value | Source |
|---|---|---|
| `mechanism` * | CodeDeploy automatic rollback of each function during its canary and bake; afterwards a rollback deployment of the alias to the previous version (A18 section 10). No Terraform resource changes, so no revert commit is needed unless both functions must return together. | Owner |
| `trigger` * | `<prefix>-pdn-alarm-threshold-errors`, `<prefix>-pdn-alarm-threshold-throttles` and the `infthresholddlq` alarm; `<prefix>-pdn-alarm-notification-dispatcher-errors`, `<prefix>-pdn-alarm-notification-dispatcher-throttles` and the notification dead-letter queue alarm (thresholds by A21, PD-173). A human decision on a push with a wrong or missing band. | Owner |
| `target_time` * | Automatic: within 10 minutes of the alarm; rollback deployment: within 30 minutes of the decision (ASM-174). Targets, not measurements. | Owner |
| `rollback_target` | `threshold` `1.3.0` and `notification-dispatcher` `1.2.0` (example), the versions recorded as previous | Pipeline |
| `decided_by_role` * | SR1 or SR2; run by the platform operator | Owner |
| `dead_letter_redrive` | Threshold and notification dead-letter queues, redriven by the platform operator after the rollback; both functions are idempotent (A17 section 6.3) | Owner |

| Cannot be rolled back | How it is handled |
|---|---|
| Push notifications already sent with the band | Harmless when the band is correct. If a wrong band was shown, a correction push on the same channel for each affected episode (A18 section 10). |
| Alert episodes and events written by the new version | Unchanged by this release: the episode record and `crop_alert_raised` are written as before. |

## 7. Verification after release

| Check | Expected | Result | Source |
|---|---|---|---|
| Deployed code equals the reviewed artifact | `CodeSha256` of both new versions equals the digests of section 1 | Not run (plan-only) | Pipeline |
| Canary and bake | No alarm of the rollback set in `ALARM` | Not run (plan-only) | Pipeline |
| Push of the next crop temperature or pH alert | The dispatcher log of the push carries the band of its threshold row; a moisture alert carries none | Not run (plan-only) | Operator |
| Alert path health | No growth of the threshold or notification dead-letter queues; delivery receipts keep arriving (A14 section 10) | Not run (plan-only) | Operator |

## 8. Incident exception

Not applicable: standard change inside an allowed window.

## 9. Release record

Not applicable: the example is never released.

## 10. Retrospective note (incident only)

Not applicable.
