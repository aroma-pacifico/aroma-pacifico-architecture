# Release Note Template: Application

Copy this file to `release-notes/<release_id>.md` in `aroma-pacifico-infrastructure`, in the pull request of the release candidate (ASM-180), and replace every `<...>`. Use it for a change that a user sees: the sack lookup, the reports web, the content or channel of an alert, or an inference outcome that the agronomist reads (A18 sections 5 to 7, ASM-181). A network, firewall, ingestion or drone path change uses the [infrastructure template](template-infrastructure.md).

Every row stays. A row that does not apply says `Not applicable` and why. The **Source** column says who fills the row: **Owner** is the change owner before the notice, **Pipeline** is copied from the deployment event of [A18 section 12](../automation-and-pipelines.md#12-hooks-for-observability-and-release-notes), and **Approver** or **Operator** is the role named. The `pdn` job refuses to run while a mandatory row (marked `*`) is empty (A18 section 9, ASM-182). Roles are role codes, never people's names.

> **Harvest-week restriction.** "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." No `pdn` change runs between 05:00 and 18:00 Colombia time (`America/Bogota`) on a day when any of Cauca, Huila or Tolima is in harvest week, unless it is an incident change under section 8 (ASM-007, ASM-008, ASM-055, ASM-178).

## 0. Summary

| Field | Value | Source |
|---|---|---|
| `release_id` * | `RN-<YYYY>-<NNN>` | Owner |
| `class` | Application | Owner |
| `change_type` * | `standard` or `incident` (section 8) | Owner |
| `title` * | `<one line, what the user will see change>` | Owner |
| `change_owner_role` * | `<SR1, SR2, JR1 or JR2>` | Owner |
| `status` | `draft`, `notice_sent`, `approved`, `released`, `rolled_back` or `cancelled` | Owner, then Pipeline |
| `related_note` | `<release_id of the infrastructure note of the same candidate, or Not applicable>` | Owner |

## 1. Change

**What changes and why.** `<Two to four sentences: the behavior before, the behavior after, and the reason, citing the design document or case statement passage it serves.>`

| Field | Value | Source |
|---|---|---|
| `issue` * | `aroma-pacifico/aroma-pacifico-roadmap#<n>` | Owner |
| `pull_request` * | `<repository>#<n>` of the release candidate | Owner |
| `pipeline` * | `platform`, `ingestion` or `flight_object_reception` (A18 sections 5 to 7) | Owner |
| `release_candidate_commit` * | `<merge commit SHA on main of aroma-pacifico-infrastructure>` | Pipeline |
| `components` * | `<functions, bundles, topics or tables touched>` | Owner |
| `diagram_boxes` | `<D01 to D13 boxes touched, or Not applicable>` (PD-180) | Owner |
| `user_visible_change` * | `<before> -> <after>`, in English; user-facing text is rendered in Spanish by the design system | Owner |
| `qa_acceptance` * | `<link to the GitHub deployment of qa with the ASM-177 suite result>` | Pipeline |
| `model_evaluation` | For a prompt, schema or model identifier change: `<PD-167 comparison result in qa>`; otherwise Not applicable | Owner |

| Artifact | From version | To version | SHA-256 digest | Source |
|---|---|---|---|---|
| `<function or bundle>` | `<semver>` | `<semver>` | `<sha256 from releases.tfvars>` | Owner, checked by Pipeline (A18 section 4.3) |

## 2. Window

| Field | Value | Source |
|---|---|---|
| `planned_start` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `planned_end` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `harvest_week` * | `yes` or `no` for every day the window spans, from the Federation harvest calendar (ASM-008); `yes` if any of the three departments is in harvest week (ASM-055) | Owner |
| `outside_restricted_hours` * | `yes` if the window does not touch 05:00 to 18:00 on a harvest-week day; otherwise this must be an incident (section 8) | Owner |
| `expected_end_before_0500` * | `yes` or `no`: apply, every canary and its bake end before 05:00 (ASM-178) | Owner, checked by Pipeline |
| `window_decision` | `allowed`, `refused` or `incident_override`, with the run identifier | Pipeline |

If the `pdn` job cannot start inside the planned window, the window is rescheduled and the notice of section 5 is sent again with its full lead time (ASM-182).

## 3. Impact

| Audience | Affected | How, and what they must do |
|---|---|---|
| Producers (farmers) | `<yes or no>` | `<farmer app, push, SMS, alert text>` |
| Collection centers | `<yes or no>` | `<logistics app, reception, code sheets, reports web>` |
| Transporters | `<yes or no>` | `<transporter app, positions, custody events>` |
| Buyers | `<yes or no>` | `<public lookup, buyer view fields of ASM-096>` |
| The Federation | `<yes or no>` | `<reports web, thresholds, agronomists, drone service>` |

| Field | Value | Source |
|---|---|---|
| `expected_downtime` * | `<none, or duration as a target>`; a canary is not downtime | Owner |
| `expected_degradation` * | `<what users may notice during the canary and bake>` | Owner |
| `data_and_schema_compatibility` * | `<additive or not; current and previous app release accepted (ASM-175); readers ignore unknown attributes (ASM-185)>` | Owner |
| `installed_apps` * | `<whether apps with unsent outbox events keep working (ASM-076, ASM-079)>` | Owner |

## 4. Approval

| Field | Value | Source |
|---|---|---|
| `qa_approver_role` | One of SR1, SR2, JR1, JR2, not the author (ASM-172) | Approver |
| `pdn_approver_role` * | SR1 or SR2, not the author and not the person who triggered the run (ASM-172) | Owner states the role; Approver |
| `approval_record` | `<GitHub deployment and run link, saved plan reference, approved at>`; GitHub keeps who approved | Pipeline |
| `additional_review` | `<security review against A09 for a buyer view, WAF or code format change (A18 section 5); otherwise Not applicable>` | Owner |

## 5. Notice to the Federation

| Field | Value | Source |
|---|---|---|
| `notice_recipient_role` * | `<the Federation role named under Q-15 (PD-171)>` | Owner |
| `notice_channel` * | `<the channel named under Q-15>`; the pipeline also publishes this note to the `pdn` release topic of A18 section 9 | Owner |
| `notice_sent_at` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `lead_time_required` * | 2 business days before `planned_start` for a standard change; at the decision for an incident (ASM-173) | Owner |
| `lead_time_met` * | `yes` or `no`; the `pdn` job refuses to run on `no` | Owner, checked by Pipeline |
| `notice_summary` * | `<three sentences for the Federation: what users will see, when, and how it is undone>` | Owner |
| `outcome_notice` | `<deployment outcome sent to the release topic, including any automatic rollback>` | Pipeline |

## 6. Rollback

| Field | Value | Source |
|---|---|---|
| `mechanism` * | `<CodeDeploy automatic rollback, rollback deployment, revert commit through the promotion, or a combination>` (A18 section 10) | Owner |
| `trigger` * | `<the alarms of A18 section 12 by name, and the human decision that starts a rollback deployment or a revert>` | Owner |
| `target_time` * | Automatic: within 10 minutes of the alarm; rollback deployment: within 30 minutes of the decision; revert commit: within 2 hours (ASM-174); targets, not measurements | Owner |
| `rollback_target` | `<previous version of each alias from the deployment record>` | Pipeline |
| `decided_by_role` * | A `pdn` approver (SR1 or SR2); run by the platform operator (A18 section 8) | Owner |
| `dead_letter_redrive` | `<queues to redrive after the rollback, or Not applicable>` | Owner |

| Cannot be rolled back | How it is handled |
|---|---|
| `<written events, sent alerts or notices, written results>` | `<compensating events, correction notice, superseding result (A18 section 10)>` |

## 7. Verification after release

| Check | Expected | Result | Source |
|---|---|---|---|
| Deployed code equals the reviewed artifact | `CodeSha256` of each new version equals its digest | `<result>` | Pipeline |
| Canary and bake | No alarm of the rollback set in `ALARM` | `<result>` | Pipeline |
| `<user-visible behavior>` | `<expected observation>` | `<result>` | Operator |
| `<dead-letter queues and audit records>` | `<no growth; audit records written>` | `<result>` | Operator |

## 8. Incident exception

Fill only when `change_type` is `incident`; otherwise write `Not applicable` once here.

| Field | Value | Source |
|---|---|---|
| `incident_reference` * | `INC-<YYYYMMDD>-<NN>` (ASM-183) | Owner |
| `declared_by_role` * | SR1 or SR2 (ASM-183) | Owner |
| `declared_at` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `why_it_cannot_wait` * | `<harm that waiting until 18:00 would cause: lost or wrong events, wrong or undelivered alerts, lookup down, security exposure>` | Owner |
| `shortened_notice_sent_at` * | `<at the decision, before the apply, marked as incident (ASM-173)>` | Owner |
| `completed_note_due` | Within 1 business day of the change (ASM-173) | Owner |
| `retrospective_due` | Within 5 business days of the change (ASM-183) | Owner |

## 9. Release record

Added after the release by the change owner, in a follow-up pull request, within 1 business day (ASM-180).

| Field | Value | Source |
|---|---|---|
| `deployment_outcome` | `<succeeded, rolled_back_automatically, rolled_back_by_decision or failed>` | Pipeline |
| `released_at` | `<YYYY-MM-DD HH:MM>` Colombia time | Pipeline |
| `verification_completed_at` | `<YYYY-MM-DD HH:MM>` Colombia time | Operator |
| `follow_ups` | `<issues opened, or none>` | Owner |

## 10. Retrospective note (incident only)

`<What happened, why the change ran inside the restricted hours, what was changed, whether the rollback was needed, and the follow-up issues that prevent the same incident.>`
