# Release Note Template: Infrastructure

Copy this file to `release-notes/<release_id>.md` in `aroma-pacifico-infrastructure`, in the pull request of the release candidate (ASM-180), and replace every `<...>`. Use it for a change of the network, a firewall (cloud or collection center), the ingestion or the drone path: routes, security groups, network ACLs, endpoint policies, transit gateway route tables, VPN connections, WAF, the IoT topic rule or policy, queues, tables, the drone object bucket and its policy, and center firewall rules (A18 sections 6 and 7, ASM-181, ASM-184). A change that a user sees uses the [application template](template-application.md).

Every row stays. A row that does not apply says `Not applicable` and why. The **Source** column says who fills the row: **Owner** is the change owner before the notice, **Pipeline** is copied from the deployment event of [A18 section 12](../automation-and-pipelines.md#12-hooks-for-observability-and-release-notes), and **Approver**, **Operator** or **Auditor** is the role named. The `pdn` job refuses to run while a mandatory row (marked `*`) is empty (A18 section 9, ASM-182). Roles are role codes, never people's names.

> **Harvest-week restriction.** "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." No `pdn` change runs between 05:00 and 18:00 Colombia time (`America/Bogota`) on a day when any of Cauca, Huila or Tolima is in harvest week, unless it is an incident change under section 8 (ASM-007, ASM-008, ASM-055, ASM-178). Changes to the hub and to collection center firewalls follow the same rule (ASM-184).

## 0. Summary

| Field | Value | Source |
|---|---|---|
| `release_id` * | `RN-<YYYY>-<NNN>` | Owner |
| `class` | Infrastructure | Owner |
| `change_type` * | `standard` or `incident` (section 8) | Owner |
| `title` * | `<one line, what changes in the network, firewall, ingestion or drone path>` | Owner |
| `change_owner_role` * | `<SR1, SR2, JR1 or JR2>` | Owner |
| `status` | `draft`, `notice_sent`, `approved`, `released`, `rolled_back` or `cancelled` | Owner, then Pipeline |
| `related_note` | `<release_id of the application note of the same candidate, or Not applicable>` | Owner |

## 1. Change

**What changes and why.** `<Two to four sentences: the state before, the state after, and the reason, citing the design document, rule ID or ADR it serves.>`

| Field | Value | Source |
|---|---|---|
| `issue` * | `aroma-pacifico/aroma-pacifico-roadmap#<n>` | Owner |
| `pull_request` * | `aroma-pacifico-infrastructure#<n>`, and `aroma-pacifico-terraform-modules#<n>` if a module changed | Owner |
| `path` * | `network`, `firewall`, `ingestion` or `drone_path` | Owner |
| `pipeline` * | `ingestion`, `flight_object_reception`, or the shared foundation stacks (A18 section 3.1); `center_device` for center firewall configuration (ASM-184) | Owner |
| `release_candidate_commit` * | `<merge commit SHA on main of aroma-pacifico-infrastructure>` | Pipeline |
| `stacks_and_accounts` * | `<stacks/network, stacks/security or stacks/workload, and the account of each: dev, qa, pdn or the network account>` | Owner |
| `module_tags` | `<module>: <vX.Y.Z> -> <vX.Y.Z>`, or Not applicable | Owner |
| `diagram_boxes` | `<D01 to D13 boxes touched, or Not applicable>` (PD-180) | Owner |
| `qa_acceptance` * | `<link to the GitHub deployment of qa with the ASM-177 suite result, or why qa does not carry this change>` | Pipeline |

**Firewall and routing rules touched.** Rule IDs are those of [`docs/firewall-policy.md`](../firewall-policy.md); an ID is never reused.

| Rule ID | Action in this change | Validity | Before production | Enforced by |
|---|---|---|---|---|
| `<CC- or CL- ID>` | `<add, change or remove>` | `<Permanent or Temporary, with the removal condition>` | `<Audit or Review>` | `<Terraform resource, or center device configuration>` |

**Terraform plan.** The plan text of each stack and environment is attached to the pull request (A18 section 4.2). Destroys and replacements are listed here first, and a plan that shows any destroy or replacement not listed here stops the change.

| Stack and environment | Resources destroyed | Resources replaced | Added or changed |
|---|---|---|---|
| `<stack, env>` | `<addresses, or none>` | `<addresses, or none>` | `<count and short description>` |

**Center device configuration.** Only for center firewall rules, which are device configuration and not Terraform (PD-122, ASM-184).

| Center | Rule IDs | Configuration backup before | Configuration backup after | Applied by |
|---|---|---|---|---|
| `<Cauca 10.41, Huila 10.42 or Tolima 10.43>` | `<IDs>` | `<backup reference>` | `<backup reference>` | Platform operator, through the session of CC-ADM-01 |

## 2. Window

| Field | Value | Source |
|---|---|---|
| `planned_start` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `planned_end` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `harvest_week` * | `yes` or `no` for every day the window spans, from the Federation harvest calendar (ASM-008); `yes` if any of the three departments is in harvest week (ASM-055) | Owner |
| `outside_restricted_hours` * | `yes` if the window does not touch 05:00 to 18:00 on a harvest-week day; otherwise this must be an incident (section 8) | Owner |
| `expected_end_before_0500` * | `yes` or `no`: apply, every canary and bake, and every center device change end before 05:00 (ASM-178, ASM-184) | Owner, checked by Pipeline |
| `window_decision` | `allowed`, `refused` or `incident_override`, with the run identifier | Pipeline |

If the `pdn` job cannot start inside the planned window, the window is rescheduled and the notice of section 5 is sent again with its full lead time (ASM-182).

## 3. Impact

| Audience | Affected | How, and what they must do |
|---|---|---|
| Producers (farmers) | `<yes or no>` | `<farm node telemetry, alerts>` |
| Collection centers | `<yes or no>` | `<LAN, Wi-Fi, tunnels, WAN, reception posts, printing>` |
| Transporters | `<yes or no>` | `<positions and custody events>` |
| Buyers | `<yes or no>` | `<public lookup front door>` |
| The Federation | `<yes or no>` | `<reports web, agronomists, drone operator and ground stations>` |

| Field | Value | Source |
|---|---|---|
| `expected_downtime` * | `<none, or duration as a target>` | Owner |
| `expected_degradation` * | `<tunnel renegotiation, rule reload, rule error action holding messages>` | Owner |
| `data_and_schema_compatibility` * | `<tables, queues and buckets touched; prevent_destroy unaffected; attribute changes additive (ASM-175)>` | Owner |
| `buffered_data` * | `<effect on farm node buffers (24 hours), drone ground station buffers and app outboxes; nothing in a buffer may be lost>` | Owner |

## 4. Approval

| Field | Value | Source |
|---|---|---|
| `qa_approver_role` | One of SR1, SR2, JR1, JR2, not the author (ASM-172) | Approver |
| `pdn_approver_role` * | SR1 or SR2, not the author and not the person who triggered the run (ASM-172); the same approval authorizes the center device changes of this note (ASM-184) | Owner states the role; Approver |
| `approval_record` | `<GitHub deployment and run link, saved plan reference, approved at>`; GitHub keeps who approved | Pipeline |
| `audit_of_rules` | For a rule marked Audit, or a temporary rule removed: `<auditor role, other than the author, and the result>` (PD-125) | Auditor |

## 5. Notice to the Federation

| Field | Value | Source |
|---|---|---|
| `notice_recipient_role` * | `<the Federation role named under Q-15 (PD-171)>` | Owner |
| `notice_channel` * | `<the channel named under Q-15>`; the pipeline also publishes this note to the `pdn` release topic of A18 section 9 | Owner |
| `notice_sent_at` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `lead_time_required` * | 2 business days before `planned_start` for a standard change; at the decision for an incident (ASM-173) | Owner |
| `lead_time_met` * | `yes` or `no`; the `pdn` job refuses to run on `no` | Owner, checked by Pipeline |
| `notice_summary` * | `<three sentences for the Federation: which sites or paths change, when, and how it is undone>` | Owner |
| `outcome_notice` | `<deployment outcome sent to the release topic, including any rollback>` | Pipeline |

## 6. Rollback

| Field | Value | Source |
|---|---|---|
| `mechanism` * | `<revert commit through the promotion that restores the previous module tag or declaration; restore of the center configuration backup; CodeDeploy rollback for a function of the path>` (A18 section 10, ASM-184) | Owner |
| `trigger` * | `<alarms by name (A18 section 12 and A21), and the observable failure that starts a human decision>` | Owner |
| `target_time` * | Revert commit: within 2 hours of the decision, including plan review and approval (ASM-174); center configuration restore: within 2 hours of the decision (ASM-184); targets, not measurements | Owner |
| `rollback_target` | `<previous commit, module tag or configuration backup>` | Pipeline or Operator |
| `decided_by_role` * | A `pdn` approver (SR1 or SR2); run by the platform operator (A18 section 8) | Owner |
| `held_messages` | `<error queue or dead-letter queue to redrive after the rollback, or Not applicable>` | Owner |

| Cannot be rolled back | How it is handled |
|---|---|
| `<readings stored with a wrong attribute, revoked certificates, moved objects, a replaced resource>` | `<correction job, re-enrollment, copy back from the retained version, prevent_destroy (A18 section 10)>` |

## 7. Verification after release

| Check | Expected | Result | Source |
|---|---|---|---|
| Applied plan equals the reviewed plan | The saved plan of the run was applied; no drift on the next plan | `<result>` | Pipeline |
| `<path that must still work>` | `<observation from the site or endpoint>` | `<result>` | Operator |
| `<path that must now be closed or open>` | `<observation, and the log line of the rule that decides it>` | `<result>` | Operator |
| `<WAN link, ingestion or drone path metrics of A21>` | `<no alarm>` | `<result>` | Operator |

**Records required before `pdn` go-live.** Fill in the note that releases the go-live, or the note that carries the record; otherwise Not applicable.

| Record | Result | Source |
|---|---|---|
| Audit of every rule marked Audit and confirmation that every temporary rule is removed (PD-125) | `<result and auditor role>` | Auditor |
| Comparison of inference outcomes with the agronomist's review (PD-167) | `<result>` | Owner |
| Rollback rehearsal in `qa` against the targets of ASM-174 (PD-176) | `<timed result>` | Owner |

## 8. Incident exception

Fill only when `change_type` is `incident`; otherwise write `Not applicable` once here.

| Field | Value | Source |
|---|---|---|
| `incident_reference` * | `INC-<YYYYMMDD>-<NN>` (ASM-183) | Owner |
| `declared_by_role` * | SR1 or SR2 (ASM-183) | Owner |
| `declared_at` * | `<YYYY-MM-DD HH:MM>` Colombia time | Owner |
| `why_it_cannot_wait` * | `<harm that waiting until 18:00 would cause: tunnel or WAN down, telemetry or drone objects not accepted, security exposure>` | Owner |
| `shortened_notice_sent_at` * | `<at the decision, before the apply, marked as incident (ASM-173)>` | Owner |
| `completed_note_due` | Within 1 business day of the change (ASM-173) | Owner |
| `retrospective_due` | Within 5 business days of the change (ASM-183) | Owner |

## 9. Release record

Added after the release by the change owner, in a follow-up pull request, within 1 business day (ASM-180).

| Field | Value | Source |
|---|---|---|
| `deployment_outcome` | `<succeeded, rolled_back or failed>` | Pipeline |
| `released_at` | `<YYYY-MM-DD HH:MM>` Colombia time | Pipeline |
| `device_changes_completed_at` | `<YYYY-MM-DD HH:MM>` Colombia time, or Not applicable | Operator |
| `verification_completed_at` | `<YYYY-MM-DD HH:MM>` Colombia time | Operator |
| `design_documents_updated` | `<pull request that updates the firewall policy or design document, or Not applicable>` | Owner |
| `follow_ups` | `<issues opened, or none>` | Owner |

## 10. Retrospective note (incident only)

`<What happened, why the change ran inside the restricted hours, what was changed, whether the rollback was needed, and the follow-up issues that prevent the same incident.>`
