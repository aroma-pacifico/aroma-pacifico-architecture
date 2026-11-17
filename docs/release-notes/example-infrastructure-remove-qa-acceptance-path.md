# Example Release Note: Infrastructure, Removal of the Temporary `qa` Acceptance Path

> **EXAMPLE ONLY.** This note is a completed copy of the [infrastructure template](template-infrastructure.md) for a fictional change. It describes a planned release: the academic delivery is plan-only, nothing has been applied, no approval has been given, no notice has been sent, no audit has been made and no check has been run. Commit SHAs, backup references and the issue and pull request numbers are obvious placeholders, and the plan counts are expected values, not plan output. Dates are written relative to a release day `D`, never as a real date. A filled note would live at `release-notes/<release_id>.md` of `aroma-pacifico-infrastructure` (ASM-180).

> **Harvest-week restriction.** "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." Day `D` is a harvest-week day in this example. The hub and the three center firewalls carry production traffic, so the change follows the restriction even before `pdn` go-live (ASM-184), and the window is placed in the evening (section 2).

## 0. Summary

| Field | Value | Source |
|---|---|---|
| `release_id` * | `RN-EXAMPLE-I01` | Owner |
| `class` | Infrastructure | Owner |
| `change_type` * | `standard` | Owner |
| `title` * | Remove the temporary path from the Logística zones to `qa` `application` (CC-LNK-06 and CL-LNK-05) | Owner |
| `change_owner_role` * | SR2 | Owner |
| `status` | `draft` (example; never released) | Owner |
| `related_note` | Not applicable: no user-visible application change | Owner |

## 1. Change

**What changes and why.** During the acceptance tests of each collection center, the Logística zone reaches the `qa` application tier through the tunnel: rule CC-LNK-06 on each center firewall and rule CL-LNK-05 on the hub and in `qa`. The [firewall policy](../firewall-policy.md#6-rules-to-audit-before-production-and-temporary-rules) marks both as temporary, "removed when the center's acceptance is signed off, or at the latest before `pdn` go-live", and requires the removal to be released with its own note and audited (ASM-128, PD-125). In this example the acceptance of the three centers is signed off, so the whole path is removed: afterwards a center reaches only the four `pdn` tiers of CL-LNK-01, and the existing blackholes CL-LNK-02 and CL-LNK-04 cover the removed routes. Epic 6 codes only permanent rules (PD-122); this example assumes the temporary declarations were added later by their own release.

| Field | Value | Source |
|---|---|---|
| `issue` * | `aroma-pacifico/aroma-pacifico-roadmap#<example-issue>` (placeholder; no such issue exists) | Owner |
| `pull_request` * | `aroma-pacifico-infrastructure#<example-pr>` (placeholder); no module tag changes | Owner |
| `path` * | `firewall` and `network` | Owner |
| `pipeline` * | Shared foundation stacks (A18 section 3.1) and `center_device` (ASM-184) | Owner |
| `release_candidate_commit` * | `0000000000000000000000000000000000example` (placeholder) | Pipeline |
| `stacks_and_accounts` * | `stacks/security` in the `qa` account (security group rule); `stacks/network` of the hub in the network account (transit gateway route tables `centers` and `qa`), reviewed and approved as a `pdn` plan through the two-step approval (ASM-184, ASM-321); configuration of the three center firewalls | Owner |
| `module_tags` | Not applicable: the declarations are removed from the stacks; no module changes | Owner |
| `diagram_boxes` | D05 (center association: hub VPN and attachments) and D07 (security group rules and Transit Gateway route-table routes, ASM-200), from the catalog in [architecture](../architecture.md) section 14 | Owner |
| `qa_acceptance` * | Not run (plan-only). In a real release the `qa` stage removes the security group rule; the ASM-177 suites do not use the center path, and the reachability checks of section 7 replace them. | Pipeline |

**Firewall and routing rules touched.**

| Rule ID | Action in this change | Validity | Before production | Enforced by |
|---|---|---|---|---|
| CL-LNK-05 | Remove: static route 10.81.48.0/20 to the `qa` VPC attachment in route table `centers`; static routes 10.41.3.0/24, 10.42.3.0/24 and 10.43.3.0/24 to the VPN attachments in route table `qa`; the `qa` security group rule from `cc-logistics` on TCP 443 | Temporary: removed with CC-LNK-06 | Audit | Terraform: transit gateway routes and a security group rule |
| CC-LNK-06 | Remove from the firewall of each of the three centers | Temporary: removed when the center's acceptance is signed off, or at the latest before `pdn` go-live | Audit | Center device configuration |
| CL-LNK-02, CL-LNK-04 | Unchanged; they now decide the removed destinations (blackhole) | Permanent | Audit, Review | Terraform |

**Terraform plan.** Expected values; the real plan text is attached to the pull request by the pipeline.

| Stack and environment | Resources destroyed | Resources replaced | Added or changed |
|---|---|---|---|
| `stacks/security`, `qa` | 1: the `qa` security group rule from `cc-logistics` on TCP 443 | None | None |
| `stacks/network`, network account | 4: one transit gateway route in `centers`, three in `qa` | None | None |
| `stacks/network` and `stacks/security`, `dev` and `pdn` | None | None | None: no change expected |

**Center device configuration.**

| Center | Rule IDs | Configuration backup before | Configuration backup after | Applied by |
|---|---|---|---|---|
| Cauca 10.41 | CC-LNK-06 | `<example-backup-cauca-before>` | `<example-backup-cauca-after>` | Platform operator, through the session of CC-ADM-01 |
| Huila 10.42 | CC-LNK-06 | `<example-backup-huila-before>` | `<example-backup-huila-after>` | Platform operator, through the session of CC-ADM-01 |
| Tolima 10.43 | CC-LNK-06 | `<example-backup-tolima-before>` | `<example-backup-tolima-after>` | Platform operator, through the session of CC-ADM-01 |

## 2. Window

| Field | Value | Source |
|---|---|---|
| `planned_start` * | `D 19:00` Colombia time: the approval run accepts the reviewed hub plan, then the platform operator changes the three center firewalls one after the other | Owner |
| `planned_end` * | `D 22:00` Colombia time | Owner |
| `harvest_week` * | `yes`: in this example at least one department is in harvest week on `D` (ASM-055) | Owner |
| `outside_restricted_hours` * | `yes`: 19:00 to 22:00 does not touch 05:00 to 18:00. The `qa` security group stage is not a `pdn` change and is not blocked by the window. | Owner |
| `expected_end_before_0500` * | `yes`: one approved hub plan without canary and three device configuration changes; a device change not finished by 22:00 is stopped and continued in a later window with its own notice (ASM-182) | Owner, checked by Pipeline |
| `window_decision` | Not run (plan-only) | Pipeline |

## 3. Impact

| Audience | Affected | How, and what they must do |
|---|---|---|
| Producers (farmers) | No | Farm telemetry never crosses a center (CC-TEL-01). |
| Collection centers | Yes | Logística can no longer reach `qa`; acceptance testing from the centers ends. Reception, sack reads and code sheets keep using `pdn` through CC-LOG-01 and CL-LNK-01, which do not change. Nothing to do. |
| Transporters | No | Their positions use the public API through the local breakout (CC-TEL-02). |
| Buyers | No | The public lookup is not reached through the tunnel. |
| The Federation | Yes | Informed that the test path is closed; agronomists and the drone ground stations are unaffected (CC-DRN rules unchanged). |

| Field | Value | Source |
|---|---|---|
| `expected_downtime` * | None expected. Removing a static route does not renegotiate the tunnels. Whether removing a rule reloads the firewall policy depends on the device class still to be procured (ASM-088), so the operator checks it before `D`. | Owner |
| `expected_degradation` * | None expected on the `pdn` path | Owner |
| `data_and_schema_compatibility` * | Not applicable: no table, queue or bucket is touched | Owner |
| `buffered_data` * | None: farm node buffers, ground station buffers and app outboxes do not use the removed path | Owner |

## 4. Approval

| Field | Value | Source |
|---|---|---|
| `qa_approver_role` | One of SR1, JR1, JR2 other than the author SR2 (ASM-172) | Approver |
| `pdn_approver_role` * | SR1, the only `pdn` approver who is not the author SR2 (ASM-172), listed in `PDN_APPROVERS` and different from the person who starts the promotion run (ASM-321); the same approval authorizes the three center device changes (ASM-184) | Owner states the role; Approver |
| `approval_record` | None: no approval has been given for this example | Pipeline |
| `audit_of_rules` | Required: a role other than the author (SR1, JR1 or JR2) confirms against the configuration backups after the change and the route tables that CC-LNK-06 and CL-LNK-05 are gone (PD-125). Not performed (example). | Auditor |

## 5. Notice to the Federation

| Field | Value | Source |
|---|---|---|
| `notice_recipient_role` * | The Federation role to be named under Q-15 (PD-171). A real note cannot pass the `pdn` check until it is named. | Owner |
| `notice_channel` * | The channel to be named under Q-15, plus publication of this note to the `pdn` release topic of A18 section 9 | Owner |
| `notice_sent_at` * | Planned: `D-2` business days, 10:00 Colombia time. Not sent (example). | Owner |
| `lead_time_required` * | 2 business days before `planned_start` (ASM-173) | Owner |
| `lead_time_met` * | `yes` if sent as planned | Owner, checked by Pipeline |
| `notice_summary` * | "On day D, between 19:00 and 22:00 Colombia time, outside the harvest-week restricted hours, the temporary test path from the three collection centers to the test environment is closed. Reception and every production path are unchanged. If a production path is affected, the previous routes and firewall configuration are restored within the 2-hour target." | Owner |
| `outcome_notice` | Not run (plan-only) | Pipeline |

## 6. Rollback

| Field | Value | Source |
|---|---|---|
| `mechanism` * | For the hub and `qa`: a revert commit through the promotion that restores the removed declarations. For a center: restore of the configuration backup taken before the change (ASM-184). Only what broke a production path is restored; reopening the test path is not a goal of a rollback. | Owner |
| `trigger` * | Reception posts of a center cannot reach `pdn` `application`, a tunnel goes down, or the WAN link alarms of A21 fire after the change; the decision is human | Owner |
| `target_time` * | Revert commit: within 2 hours of the decision (ASM-174); center restore: within 2 hours of the decision (ASM-184). Targets, not measurements. | Owner |
| `rollback_target` | The commit before the release candidate; the three "before" backups of section 1 | Pipeline or Operator |
| `decided_by_role` * | SR1 or SR2; run by the platform operator | Owner |
| `held_messages` | Not applicable: no queue is on this path. Sack events entered at a center during an outage wait in the outbox (ASM-079). | Owner |

| Cannot be rolled back | How it is handled |
|---|---|
| Nothing on this path holds data | The audit records of the change and the firewall logs are kept as written. |

## 7. Verification after release

| Check | Expected | Result | Source |
|---|---|---|---|
| Approved plan equals the reviewed plan | The approval run recomputed the digest of the reviewed plan of both stacks and it equals the approved digest; nothing was applied | Not run (plan-only) | Pipeline |
| `pdn` path from each center | A reception post at each center reaches the application API in `pdn` `application` (CC-LOG-01) | Not run (plan-only) | Operator |
| `qa` path from each center is closed | An HTTPS attempt from a Logística post to `qa` `application` fails and appears in the firewall log as a logged deny (CC-LOG-04 or CC-DEF-01) | Not run (plan-only) | Operator |
| Hub route tables | `centers` holds only the routes of CL-LNK-01 and the blackholes of CL-LNK-02; `qa` holds only the blackhole of CL-LNK-04 | Not run (plan-only) | Operator |
| WAN links | No tunnel or WAN link alarm of A21 during or after the window | Not run (plan-only) | Operator |

**Records required before `pdn` go-live.**

| Record | Result | Source |
|---|---|---|
| Audit of every rule marked Audit and confirmation that every temporary rule is removed (PD-125) | This note records the removal of CC-LNK-06 and CL-LNK-05; the full audit is recorded in the go-live note. Not performed (example). | Auditor |
| Comparison of inference outcomes with the agronomist's review (PD-167) | Not applicable to this note | Owner |
| Rollback rehearsal in `qa` against the targets of ASM-174 (PD-176) | Not applicable to this note | Owner |

## 8. Incident exception

Not applicable: standard change inside an allowed window.

## 9. Release record

Not applicable: the example is never released. After a real release the `design_documents_updated` field would cite the pull request that marks CC-LNK-06 and CL-LNK-05 as removed in the firewall policy, keeping their IDs.

## 10. Retrospective note (incident only)

Not applicable.
