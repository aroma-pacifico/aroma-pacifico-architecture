# Automation and Pipelines

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR2 / JR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#18 (A18) |
| Builds on | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) (Terraform, state, OIDC, gates, `pdn` approval, harvest-week window), [ADR-001](adr/adr-001-primary-and-second-cloud.md) (Region, accounts), [`docs/landing-zone.md`](landing-zone.md) (A08, production-change SCP, audit), [`docs/security-and-identity.md`](security-and-identity.md) (A09, permission sets, keys), [`docs/farm-monitoring.md`](farm-monitoring.md) (A14), [`docs/drone-service.md`](drone-service.md) (A15), [`docs/sack-and-trip.md`](sack-and-trip.md) (A16), [`docs/inference-functions.md`](inference-functions.md) (A17), [`docs/clarification-questions.md`](clarification-questions.md) (A06, Q-09 and Q-15) |
| Feeds | A19 architecture document (aroma-pacifico/aroma-pacifico-roadmap#19), A20 100-hour plan (aroma-pacifico/aroma-pacifico-roadmap#20), A21 observability (aroma-pacifico/aroma-pacifico-roadmap#34), A22 release notes (aroma-pacifico/aroma-pacifico-roadmap#35), B13 pipeline (aroma-pacifico/aroma-pacifico-roadmap#33) |
| Decision-log entries | ASM-170 to ASM-179, PD-170 to PD-176 |

## Diagrams

- [Telemetry Ingestion and Flight-Object Reception Pipelines (SVG)](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/pipelines/ingestion-and-flight-objects.svg) ([PNG](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/pipelines/ingestion-and-flight-objects.png)): Depicts the parallel telemetry ingestion engine and the pre-signed S3 flight object reception pipeline with SHA-256 validation and quarantine routing.
- [Platform CI/CD Deployment Pipeline (SVG)](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/pipelines/platform-pipeline.svg) ([PNG](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/pipelines/platform-pipeline.png)): Illustrates the GitHub Actions CI/CD deployment pipeline, branch promotion gates across environments, reviewable plan generation, and harvest-week freeze windows.

## 1. Purpose and requirements

Challenge point 4 of the [case statement](case/case-statement.pdf) asks for "Pipeline de la plataforma, de la ingesta y de la recepción de objetos de vuelo", "Ambientes de desarrollo, pruebas y producción, con promoción controlada" and "Secretos fuera del repositorio y fuera del estado". It also says "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente", and the release notes it requires must let a reader see "la aprobación, el aviso a la Federación y cómo se vuelve atrás". Criterion 4 of the evaluation repeats it: "las plantillas permiten aprobar, avisar y volver atrás". Quotations are in the original Spanish, which prevails over the English rendering (ASM-001).

ADR-003 already fixes the infrastructure side: Terraform, one natively locked S3 state per environment, GitHub OIDC with separate plan and apply roles, the gates on every pull request, the `pdn` apply in a GitHub deployment environment with required reviewers applying the saved plan, and the harvest-week window (ASM-007, ASM-008, ASM-055). This document does not reopen ADR-003. It extends it to the three pipelines the case statement names and records how each one builds, promotes, is approved, notifies and rolls back. No time in this document is a measurement; every duration is a design target recorded in the decision log.

| Requirement | Where it is met |
|---|---|
| Three pipelines: platform, ingestion, flight object reception | Sections 5, 6 and 7 |
| `dev`, `qa` and `pdn` with controlled promotion | Sections 3 and 4 |
| Approve, notify, roll back | Sections 8, 9 and 10 |
| Secrets out of the repository and out of the state | Section 11 |
| No production change from 05:00 to 18:00 in harvest week, except an incident | Section 4.4 |
| Hooks for the alarms of A21 and the release notes of A22 | Section 12 |

**Plan-only boundary.** The academic delivery never runs `terraform apply`, never creates a deployment and never publishes an artifact to a cloud account (`AGENTS.md` section 4). Every step below that applies, stages or shifts traffic is designed, not executed; where a step would act on an account, the text says "would". How the executable plan of Epic 6 is evidenced is PD-019, and only the infrastructure plan path is coded, by B13.

## 2. Scope of each pipeline

The case statement says "Quien presenta no implementa lógica de negocio, aplicaciones ni firmware", so no application or function code exists in the pilot delivery. The pipelines are designed so that the platform is ready for that code: they define where it would live, what artifact it would produce and how that artifact would reach each environment. The infrastructure side of the ingestion pipeline is the only part coded in Epic 6 (ADR-003, PD-161).

| Pipeline | What it carries | Designed by |
|---|---|---|
| Platform | Application API (Amazon API Gateway, the `execute-api` endpoint of A15 section 9, and its Lambda functions: sack events, trips and positions, mission package and upload grant, thresholds, reports queries), the Cognito pre-authentication trigger, the sack applier and exporter, the public lookup function, the static bundles of the reports web and of the lookup page, the CloudFront distributions and WAF web ACLs, the transactional tables and the Cognito pools. | A09, A14, A15, A16 |
| Ingestion | The telemetry pattern of ADR-003 (IoT policy, topic rule, error queue, telemetry table, log group), the farm module connection points, the threshold queue and its dead-letter queue, the threshold function of A17, the alert topic and the notification dispatcher of A14. | ADR-003, A14, A17 |
| Flight object reception | The drone object bucket, its bucket policy and event notification, the reception function of A15, the mission table stream, the EventBridge Pipe, bus and rules, the function queues and dead-letter queues, and the confirmation, verification and estimation functions of A17 with their prompts and output schemas. | A15, A17 |

The mobile applications (farmer, transporter, center logistics) are distributed through the mobile app stores, not through the cloud; their store test and production tracks are outside these pipelines. What the pipelines guarantee to them is backward compatibility of the API (ASM-175), because an installed app keeps unsent events in its outbox for days (ASM-076, ASM-079). The drone ground-station software belongs to the contracted operator (PD-143).

## 3. Repositories, environments and pipeline identities

### 3.1 Repositories and triggering paths

Infrastructure follows ADR-003. Application and function code would live in one private repository that does not exist yet (ASM-170). Agents never create repositories, so creating it is the lead's decision (PD-170). The paths below are proposed; B01 fixes the exact layout of the infrastructure repository.

| Repository | Path that triggers | Pipeline | Produces |
|---|---|---|---|
| `aroma-pacifico-terraform-modules` | `modules/<name>/**` | Module CI of the pipeline that owns the module (section 5, 6 or 7) | A SemVer git tag `vX.Y.Z`; never cloud credentials (ADR-003). |
| `aroma-pacifico-infrastructure` | `stacks/network/**`, `stacks/security/**`, `stacks/state-backend/**` | Shared foundation of the three pipelines | Saved plans per environment, applied in order network, security, workload (PC-IAC-022). |
| `aroma-pacifico-infrastructure` | `stacks/workload/**`, including `stacks/workload/releases.tfvars` (ASM-171) | Whichever pipeline owns the changed module call or release entry | Saved plans per environment. |
| `aroma-pacifico-platform` (ASM-170) | `services/api/**`, `functions/lookup/**`, `functions/sack-applier/**`, `functions/sack-exporter/**`, `functions/pre-auth/**`, `web/reports/**`, `web/lookup/**` | Platform | Versioned Lambda packages and static bundles. |
| `aroma-pacifico-platform` | `functions/threshold/**`, `functions/notification-dispatcher/**` | Ingestion | Versioned Lambda packages. |
| `aroma-pacifico-platform` | `functions/reception/**`, `functions/confirmation/**`, `functions/verification/**`, `functions/estimation/**`, `inference/prompts/**` | Flight object reception | Versioned Lambda packages, each with its prompt and schema version (ASM-179). |

### 3.2 Environments

| Environment | Account (A08) | GitHub deployment environment | Purpose |
|---|---|---|---|
| `dev` | Development account | `dev`, no reviewer | Integration of every merge to `main`. |
| `qa` | Test account | `qa`, one reviewer (section 8) | Acceptance with the same artifact and the same deployment configuration as `pdn` (ASM-174, ASM-177). The only environment whose plan Epic 6 evaluates (ASM-051). |
| `pdn` | Production account | `pdn`, required reviewers, harvest-week window, notice check (sections 8 and 9) | Production. Changes only through this pipeline (A08 section 3.1). |

An environment differs from another only by its `.tfvars` and its backend file (ADR-003). The artifact, the module tags and the release entries are identical across the three.

### 3.3 Pipeline identities

Every pipeline identity is a GitHub OIDC role; no access key exists anywhere (A08 SCP "Deny long-term credentials"). The plan and apply roles are those of ADR-003 and are not redesigned. This document adds one role.

| Role | Account | Trusted subject | Allows |
|---|---|---|---|
| Plan (ADR-003) | Each environment | Pull request and `main` tokens of `aroma-pacifico-infrastructure` | Read state, write `*.tflock`, read-only describe. |
| Apply (ADR-003) | `qa`, `pdn` (and `dev`) | The repository's deployment environment of the same name | Apply the saved plan, and start and stop CodeDeploy deployments of that environment (section 4.3). |
| Artifact publisher (ASM-171) | Each environment | Tag pushes of `aroma-pacifico-platform` matching a release tag | `s3:PutObject` on `artifacts/*` of that environment's artifact bucket only; no read of any data, no Lambda, no CodeDeploy. |

## 4. Common release model

### 4.1 What a release candidate is

A release candidate is one merge commit on `main` of `aroma-pacifico-infrastructure`. It pins everything that reaches an environment: module tags (`?ref=vX.Y.Z`, PC-IAC-015) and, in `stacks/workload/releases.tfvars`, the version and SHA-256 digest of every Lambda package and static bundle (ASM-171). Because the same commit is planned and applied in `dev`, then `qa`, then `pdn`, the same artifact is promoted by construction: an environment cannot run a digest or module tag that the environments before it did not run.

| Artifact | Built by | Identity of the artifact | Where an environment reads it |
|---|---|---|---|
| Terraform configuration | Module CI and infrastructure CI | Commit SHA plus module tags; one saved plan per environment and run | GitHub Actions run artifact, short retention, never committed (ADR-003) |
| Lambda package | Platform repository build, on a release tag | `<function>-<semver>.zip`, its SHA-256 digest | `artifacts/functions/<function>/<semver>/<sha256>.zip` in the environment's artifact bucket |
| Static bundle (reports web, lookup page) | Platform repository build | `<site>-<semver>.tar`, its SHA-256 digest, unpacked to versioned file names | `releases/<semver>/` in the site's assets bucket (ASM-176) |

### 4.2 Gates

Gates run cheapest first, as ADR-003 sets. A failed gate stops the run; it is never skipped to obtain green.

| Stage | Gates |
|---|---|
| Pull request, modules repository | `terraform fmt -check -recursive`, `terraform validate`, `tflint`, Trivy or Checkov, plan-mode `terraform test` with mock providers. No cloud credentials. |
| Pull request, platform repository | Unit tests, lint, dependency vulnerability scan, secret scan, build of the package, and contract tests of the API and event schemas against the current and previous versions (ASM-175). |
| Release tag, platform repository | Rebuild from the tag, SHA-256 digest, software bill of materials, then the artifact publisher writes the same bytes to the three artifact buckets. |
| Pull request, infrastructure repository | The module gates above, secret scan, a check that every digest in `releases.tfvars` exists in all three artifact buckets, and a `terraform plan` of each affected stack for `dev`, `qa` and `pdn`, attached to the pull request with destroys and replacements listed first. |
| `dev` | Apply the saved `dev` plan on merge, then the function deployments (section 4.3) with `AllAtOnce`. |
| `qa` | One reviewer approves the `qa` plan; apply; function deployments with the `pdn` configuration; the acceptance suite of ASM-177, whose result is recorded in the GitHub deployment. |
| `pdn` | `qa` succeeded for the same commit; the release note exists and its notice was sent with the lead time of ASM-173; the harvest-week window allows the run (section 4.4); required reviewers approve the `pdn` plan; apply the saved plan; function deployments with canary and automatic rollback. |

### 4.3 How function code is deployed

Terraform publishes every new Lambda version, and AWS CodeDeploy moves the `live` alias to it. Concretely, each function is declared with `publish = true` and its code reference (`s3_key` and `source_code_hash`) taken from `releases.tfvars`, so a new digest appears in the plan as a code change and, after apply, as a new numbered version. Terraform also declares the `live` alias, the CodeDeploy application and deployment group with its alarms and automatic rollback, and every event source (API integration, SQS event source mapping, S3 notification, IoT rule SQS action) targets the alias ARN, never `$LATEST`.

Right after the apply, the same approved job starts one CodeDeploy deployment per function whose version changed, from the alias's current version to the new one. The job first checks that the `CodeSha256` that Lambda reports for the new version equals the digest in `releases.tfvars`, and stops before shifting any traffic if it does not. The alias's `function_version` and `routing_config` are then moved by CodeDeploy, an agent outside Terraform, so the alias carries `lifecycle { ignore_changes = [function_version, routing_config] }`. That is the case PC-IAC-010 section 3.2 permits ("atributos que son modificados por agentes externos a Terraform") and the only `ignore_changes` in the design; every other attribute stays under Terraform.

This keeps ADR-003 whole. The code change is reviewed as part of the plan, it is applied only by the apply role from an approved deployment environment, and the traffic shift is a step of the same approved run, not a second path into production. It also delivers the alias, canary and automatic rollback that ASM-157 already assumed for the sack design.

| Environment | CodeDeploy configuration | Alarms that roll back |
|---|---|---|
| `dev` | `CodeDeployDefault.LambdaAllAtOnce` | None; a failure is fixed forward. |
| `qa` | `CodeDeployDefault.LambdaCanary10Percent5Minutes`, as `pdn`, so that rollback is exercised before production (ASM-174) | The alarm set of section 12 in `qa`. |
| `pdn` | `CodeDeployDefault.LambdaCanary10Percent5Minutes` | The alarm set of section 12 in `pdn`. |

### 4.4 Promotion procedure and the harvest-week window

1. The change owner merges a pull request on `main` of `aroma-pacifico-infrastructure` after its plans are reviewed. A pull request that bumps a module tag or a release entry is opened by a person; no workflow writes to another repository, so no repository token is needed.
2. The promotion run plans and applies `dev`, and runs the function deployments.
3. The run plans `qa`. A `qa` reviewer approves; the run applies, deploys and runs the acceptance suite (ASM-177).
4. The run plans `pdn` only after step 3 succeeded, so the saved plan is fresh. The `pdn` job checks the release note and the notice (section 9) and the window, then waits for the required reviewers.
5. After approval the job applies the saved `pdn` plan, then runs the function deployments and records the result as a GitHub deployment status and a deployment event (section 12).

A later merge supersedes a run still waiting for approval; a run already applying is never cancelled. A pull request that reverts a commit is a release candidate like any other and follows the same steps.

The window check is the one ADR-003 designs, applied to the whole `pdn` job and not only to `terraform apply`, because a canary shift is a production change too. On a harvest-week day of any of the three departments (ASM-055), the job refuses to start between 05:00 and 18:00 Colombia time (ASM-007) and also refuses to start if its expected end, the apply plus every canary and its bake time, would pass 05:00 (ASM-178). An incident overrides it only with an incident reference recorded in the approval. Plans and the `dev` and `qa` stages are never blocked by the window.

## 5. Platform pipeline

| Element | Design |
|---|---|
| Trigger | Platform paths of section 3.1; modules of the application API, lookup front door, tables and Cognito pools; their entries in `releases.tfvars`. |
| Artifacts | One Lambda package per API function, the lookup, the applier, the exporter and the pre-authentication trigger; one static bundle each for the reports web and the lookup page. |
| Infrastructure deploy | Terraform: API Gateway stage and routes, CloudFront distributions, WAF web ACLs (ASM-129, ASM-155), DynamoDB tables with point-in-time recovery and `prevent_destroy`, Cognito pools and app clients. |
| Code deploy | Section 4.3 for every function. The lookup function's alias rollback is the first recovery step of A16 section 9.3. |
| Static bundles | Uploaded once to `releases/<semver>/` with versioned file names; the active release is the CloudFront origin path set from `releases.tfvars`, so switching or reverting a bundle is one Terraform change and needs no cache invalidation (ASM-176). |
| Acceptance in `qa` | API contract tests for the current and previous event schemas, a sack lifecycle replay through every state of A16 section 3, a lookup by code that checks only the buyer-view fields of ASM-096, and sign-in with MFA for a `staff` test identity (ASM-177). |
| Approval in `pdn` | Section 8; a change to the buyer view, the WAF or the code format is also reviewed against the security design A09. |
| Notice | Application release note of A22 for any change visible in the lookup, the reports web or the alerts; infrastructure release note for a front door, WAF or table change. |

**Rollback.** A defective function is rolled back by CodeDeploy when an alarm fires during the canary, or by a rollback deployment that moves the alias to the previous version (section 10). A defective bundle or front-door change is reverted by a revert commit through the same promotion, which the 4-hour RTO budget of the lookup absorbs (ASM-157). **What cannot be rolled back:** sack events already written. The event log takes only conditional puts and no function can update or delete it (A16 section 9.1), so a defective release cannot erase events, but it can write wrong ones. They are corrected by compensating events (ASM-151) and the derived tables are rebuilt from the log by the applier (A16 section 9.2). Because a deployment never touches `sack_event_log` and the outbox releases an event only after the API confirms it (ASM-079), "Un despliegue no pierde eventos de saco" holds across a rollback as well.

## 6. Ingestion pipeline

| Element | Design |
|---|---|
| Trigger | `modules/telemetry-ingestion/**` and `modules/farm/**` in the modules repository; their module calls in `stacks/workload`; `functions/threshold/**` and `functions/notification-dispatcher/**`. |
| Artifacts | Module tag `vX.Y.Z` of the ingestion and farm modules; Lambda packages of the threshold function and the dispatcher. |
| Infrastructure deploy | Terraform: IoT policy, topic rule with its DynamoDB action, SQS action and error action (ASM-161), error queue, telemetry table with `prevent_destroy`, threshold queue and dead-letter queue, event source mapping on the `live` alias, alert topic. This path is what B09, B10 and B13 code. |
| Code deploy | Section 4.3. The event source mapping invokes the `live` alias, so during a canary 10 % of the batches run the new version. Both versions are idempotent by the conditional writes of A17 section 6.3. |
| Acceptance in `qa` | A synthetic device in `qa` publishes recorded readings through the real endpoint: in-range and out-of-range live readings, a drained 24-hour buffer with original timestamps, and a duplicate. The suite checks one stored item per reading, one alert per episode, the delayed mark on drained readings and delivery on both channels to test phones (ASM-177). |
| Approval in `pdn` | Section 8. A change of the topic rule, the IoT policy or the telemetry table is an infrastructure change of "la ingesta" in the A22 sense. |
| Notice | Infrastructure release note for the rule, policy, table or queues; application release note for a change of the alert content or channel. |

**What is not a deployment.** A threshold change is Federation data written through the reports web, audited in the same transaction (A14 section 4.5); it does not pass through any pipeline and is not blocked by the change window (ASM-179). Device certificates are registered and revoked operationally (A09 section 10), not by a pipeline run.

**Rollback.** The threshold function and the dispatcher roll back by alias (section 10). A topic rule, policy or queue change rolls back by a revert commit that restores the previous module tag. An IoT topic rule has no versions or aliases, so its rollback is that revert and its apply, and the rule's error action keeps any message the rule could not deliver in the error queue for redrive in the meantime. **What cannot be rolled back:** readings that a defective rule stored with a wrong attribute, because IoT Core keeps no copy of a message after the rule has run, and alerts already sent to a farmer by SMS or push. Wrong readings are corrected by an audited correction job that adds a correction to the affected items where the stored values allow it, and otherwise marks those readings as unreliable in the history, and a wrongly sent alert gets a correction notice on the same channels. A revoked certificate is final: the node is enrolled again with a new certificate from a new CSR. The `prevent_destroy` on the telemetry table makes a plan that would replace it fail instead of losing 5 years of readings.

## 7. Flight object reception pipeline

| Element | Design |
|---|---|
| Trigger | The drone object and reception modules in the modules repository and their module calls; `functions/reception/**`, `functions/confirmation/**`, `functions/verification/**`, `functions/estimation/**`, `inference/prompts/**`. |
| Artifacts | Module tags; one Lambda package per function. The package of an inference function embeds its prompt and output schema with their `prompt_version`, so one Lambda version always runs one prompt (ASM-179). The model identifier is a Terraform variable of the function (A17 section 2.1). |
| Infrastructure deploy | Terraform: object bucket with versioning, Block Public Access, the bucket policy CL-DRN-03 and lifecycle; S3 event notification to the reception alias; mission table stream, EventBridge Pipe, custom bus and one rule per mission type; one queue and one dead-letter queue per function with an event source mapping on the alias (ASM-161, ASM-162). |
| Code deploy | Section 4.3 for the four functions. |
| Acceptance in `qa` | A recorded sample mission of each type is uploaded with an upload grant: objects screened `clear` and one marked `person_present`, then the flight record. The suite checks `record_persisted`, quarantine of an object outside the grant, the move to the restricted prefix, and one result per mission with confidence and source identifiers. The comparison with the agronomist's review is PD-167, not part of every run (ASM-177). |
| Approval in `pdn` | Section 8. A change of a prompt, schema or model identifier is an application change of the drone path and needs the agronomist's comparison result in `qa` recorded in the release note (PD-167). |
| Notice | Infrastructure release note for a change of "el camino del dron" (bucket, policy, endpoints, routing); application release note for a change of the inference outcome or of what the agronomist sees. |

**Rollback.** Functions roll back by alias; a defective version that failed an invocation leaves its event in the dead-letter queue, from which a platform operator redrives it once the previous version is live again, without harm because every function is idempotent (A17 sections 6.1 and 6.3). Routing, bucket and policy changes roll back by a revert commit. A deployment never deletes an uploaded object, and the ground station deletes from its buffer only what the persistence receipt lists (A15 section 7), so a rollback loses no flight object. **What cannot be rolled back:** results written by a defective version, a polygon version activated by the reception function (ASM-147), and objects it moved to `quarantine/` or `restricted/person/`. Results are superseded, never edited: a new evaluation of the same mission writes a newer result, and the defective ones are found by the `function_version` that each result records (ASM-175). A wrongly activated polygon is replaced by activating the previous polygon version as a new version. A wrongly moved object is copied back from its retained version by a platform operator under an incident reference, with an audit record. A defective result can never authorize a fumigation, because authorization is a human identity in every version (Part E, ADR-002).

## 8. Approval

Approvers are role codes of the decision log, never people's names. GitHub prevents the author of a change from approving its own deployment, and the approval record (who, when, which run and which saved plan) is kept by GitHub and copied into the release note (ASM-172). Whether the organization's GitHub plan offers required reviewers on the private repository is still PD-020; if it does not, the fallback must still bind a named human to the reviewed plan.

| Stage | Approves | Condition |
|---|---|---|
| Merge of a pull request, any repository | One reviewer of SR1, SR2, JR1, JR2 other than the author | Gates green; plan text attached for the infrastructure repository. |
| `qa` apply and deploy | One reviewer of SR1, SR2, JR1, JR2 other than the author | Same commit succeeded in `dev`. |
| `pdn` apply and deploy, standard change | One of SR1 or SR2, other than the author and other than the person who triggered the run | `qa` succeeded with acceptance recorded; release note with notice sent; window allows. |
| `pdn` change inside the window, incident | One of SR1 or SR2 with an incident reference in the approval | The notice is sent at the time of the decision (ASM-173). |
| Rule marked Audit in the firewall policy | Audit by a person other than its author before go-live (PD-125) | Recorded in the infrastructure release note. |

The platform operator does not approve deployments; it operates the platform through its permission sets (A09 section 5.3), redrives dead-letter queues and runs rollback deployments that a `pdn` approver authorizes. The `BreakGlass` permission set is not a deployment path.

## 9. Notice to the Federation

The case statement requires "el aviso a la Federación" in both release notes. A06 asked who receives it, through which channel and with how much lead time (Q-15), and its fallback assumes no value: the templates keep recipient, channel and lead time as mandatory fields. This design keeps that rule and gives the pipeline a default to check against until the Federation answers (ASM-173, PD-171).

| Change class | Default lead time | Channel and recipient | Pipeline check |
|---|---|---|---|
| Standard `pdn` change | Notice sent at least 2 business days, Colombia time, before the window in which the `pdn` job runs | The recipient and channel the Federation names under Q-15. The pipeline also publishes the release note to an Amazon SNS topic in the `pdn` account whose subscriber the Federation names at deployment; no address is written here. | The `pdn` job reads the release note linked in the pull request and refuses to run if a mandatory notice field is empty or the notice time is later than the lead time allows. |
| Incident change | At the time of the decision, before the apply | Same channel, marked as an incident | The incident reference is required; the completed release note follows within 1 business day. |
| `dev` and `qa` | No notice | None | None. |

When the `pdn` deployment ends, the deployment event of section 12 sends the outcome, including an automatic rollback, to the same topic, so the Federation is told when a change it was warned about did not stay in place.

## 10. Rollback

Every time below is a design target, to be timed by the rollback rehearsal in `qa` (ASM-174, PD-176). Lambda versions are kept, never pruned below the last five, so that a rollback target always exists.

| Mechanism | Used for | Trigger | Target time |
|---|---|---|---|
| CodeDeploy automatic rollback | Any function in `qa` or `pdn` during its canary and bake | An alarm of section 12 in state `ALARM` | Traffic back on the previous version within 10 minutes of the alarm |
| Rollback deployment | Any function after its deployment completed | A `pdn` approver decides; the job moves the alias to the version recorded as previous in the deployment record | Within 30 minutes of the decision |
| Revert commit through the promotion | Terraform changes: module tag, topic rule, bucket policy, routing, WAF, origin path of a static bundle | A `pdn` approver decides; an incident is the only exception to the window | Within 2 hours of the decision, including plan review and approval; inside the 4-hour lookup RTO (ASM-157) |
| Dead-letter queue redrive | Events that failed while a defective version was live | Platform operator, after the rollback | After the rollback, without a deployment |

| Cannot be rolled back | How it is handled |
|---|---|
| Sack events written | Compensating events; rebuild of derived tables from the log (section 5). |
| Telemetry readings stored with a wrong attribute | Audited correction job, or the readings marked unreliable (section 6). |
| Alerts and notices already sent to a farmer | Correction notice on the same channels (section 6). |
| Inference results and activated polygon versions | Superseded by a newer result or polygon version; defective ones found by `function_version` (section 7). |
| Objects moved to `quarantine/` or `restricted/person/` | Copied back from the retained version under an incident reference (section 7). |
| A revoked device certificate | Re-enrollment with a new certificate from a new CSR (section 6; A09 section 10). |
| Data shape changes in a table | Avoided: attribute changes are additive and a removal waits for the next release (ASM-175); `prevent_destroy` makes a replacing plan fail. |
| A scheduled KMS key deletion | Cancelled within the 30-day waiting period (ASM-097); key policies deny scheduling deletion to every principal except `BreakGlass` (A09 section 7.3). |

## 11. Secrets out of the repository and out of the state

The three pipelines follow ADR-003: no static key exists, Terraform never sees a private key, and any secret a resource needs lives in AWS Secrets Manager or as an SSM `SecureString` read through an ephemeral resource into a write-only argument. A function that needs a secret reads it at run time through its own role; it is never passed as a Lambda environment variable, because environment variables are stored in the Terraform state and in the function configuration. GitHub holds only non-secret variables: role ARNs, bucket names and account-scoped names, set per deployment environment.

| Pipeline | What it needs | How it gets it | Never |
|---|---|---|---|
| All | AWS access for plan, apply, deploy and artifact staging | GitHub OIDC into the roles of section 3.3 | An access key, a GitHub repository secret holding cloud credentials |
| All | Module source | Public HTTPS of the modules repository (ADR-003) | A deploy key or token |
| Platform | Cognito app client identifiers for the apps and the reports web | Non-secret SSM parameters published by the stack; the clients are public, without a client secret (A09 section 3.2) | A client secret in a bundle |
| Platform | Lookup origin credentials | CloudFront origin access control signs with its service identity (A16 section 7.2) | A shared header secret |
| Ingestion | Device identity | Key generated on the node, certificate from a CSR (ASM-010) | A private key in state |
| Ingestion | Push provider credentials of the SNS platform application | Secrets Manager, created operationally or with a write-only argument (A14 section 7.3, PD-130) | The credential in `.tfvars` or state |
| Flight object reception | Upload access for the ground station | 15-minute pre-signed URLs issued at run time by the API role (ASM-093) | A stored credential for the operator |
| Flight object reception | Model access | `bedrock:InvokeModel` on the function role (A17 section 11) | An API key |

Every pull request of the three code-bearing repositories runs a secret scanner, saved plans are never committed and keep short retention in the run, and only the plan text is attached to the pull request (ADR-003).

## 12. Hooks for observability and release notes

A21 owns the alarms and dashboards and A22 owns the two templates. This section defines only the hooks they plug into. Names follow PC-IAC-003 with `<prefix>` until PD-070 fixes it, and `pdn` shown.

| Hook | Content | Consumer |
|---|---|---|
| Rollback alarm set of each function | `<prefix>-pdn-alarm-<function>-errors` on `AWS/Lambda` `Errors` of the alias, `<prefix>-pdn-alarm-<function>-throttles`, the function's dead-letter queue alarm of A17 section 7 (`infthresholddlq`, `infconfirmdlq` and the others), and for the API functions `<prefix>-pdn-alarm-api-5xx` on API Gateway 5XX errors. CodeDeploy reads them through the deployment group. | A21 sets thresholds and windows (PD-173). |
| Deployment event | One record per environment and run: commit, module tags, digests, saved plan reference, approver, incident reference if any, window decision, CodeDeploy outcome, rollback if any. Published as a GitHub deployment status and as a message on the `pdn` release topic of section 9. | A21 deployment dashboard; A22 release notes. |
| CodeDeploy notifications | Deployment start, success, failure and rollback sent to the same topic through the deployment group's triggers. | A21, the Federation notice of section 9. |
| Release note fields filled by the pipeline | What changed (commit and digests), window, approval, notice time and recipient field, rollback mechanism from section 10 and its target. | A22 templates (PD-174). |

## 13. Compliance with the case statement and the issue

| Passage or acceptance criterion | Where it is met |
|---|---|
| "Pipeline de la plataforma, de la ingesta y de la recepción de objetos de vuelo." Criterion: "Automation patterns are defined for the platform, ingestion and flight object reception." | Sections 2, 3.1, 5, 6 and 7 |
| "Ambientes de desarrollo, pruebas y producción, con promoción controlada." Criterion: "The design covers development, test and production with controlled promotion between environments." | Sections 3.2, 4.1, 4.2 and 4.4 |
| "Secretos fuera del repositorio y fuera del estado." Criterion: "The automation is consistent with keeping secrets out of the repository and out of the state." | Section 11 |
| "la aprobación, el aviso a la Federación y cómo se vuelve atrás." Criterion: "The proposal covers approving, notifying and rolling back." | Sections 8, 9 and 10, per pipeline in sections 5, 6 and 7 |
| "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." | Section 4.4 |
| "Pipeline que ejecuta el plan y exige aprobación antes de aplicar en producción." | Sections 4.2 and 8, as ADR-003 |
| "Un despliegue no pierde eventos de saco." | Section 5 |

## 14. Decision-log entries

| ID | Summary |
|---|---|
| ASM-170 | Application and function code would live in a future private `aroma-pacifico-platform` repository with the paths of section 3.1. |
| ASM-171 | Build once on a release tag, stage the same bytes in each environment's artifact bucket, pin version and digest in `releases.tfvars`. |
| ASM-172 | Approvers by stage: team roles for merge and `qa`, SR1 or SR2 for `pdn`, never the author. |
| ASM-173 | Default notice lead time of 2 business days for a standard change, at decision time for an incident, until Q-15 is answered. |
| ASM-174 | CodeDeploy configurations per environment, five retained versions and the rollback time targets. |
| ASM-175 | API and event schemas accept the current and previous app release; table changes are additive; results record `function_version`. |
| ASM-176 | Static bundles under versioned prefixes, the active one selected by the CloudFront origin path. |
| ASM-177 | Contents of the `qa` acceptance suites; results recorded in the deployment, never invented. |
| ASM-178 | The window check covers the whole `pdn` job and its expected end. |
| ASM-179 | Prompts and schemas ship inside the function package; Federation-configured values are data, not deployments. |
| PD-170 to PD-176 | Platform repository, Federation notice, live feature re-check, A21 alarms, A22 fields, B13 scope, rollback rehearsal. |

## 15. References

| Reference | Use |
|---|---|
| [`docs/decision-log.md`](decision-log.md) | ASM-170 to ASM-179, PD-170 to PD-176; ASM-007, ASM-008, ASM-051, ASM-055, ASM-157 used as written. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | Tool, state, OIDC roles, gates, `pdn` approval and window, extended here. |
| [`standards/iac-rules/pc-iac-010.md`](../standards/iac-rules/pc-iac-010.md) | Section 3.2, `ignore_changes` only for attributes changed by an agent outside Terraform. |
| [`docs/inference-functions.md`](inference-functions.md) | Function queues, idempotency, dead-letter redrive and alarm names. |
| [`docs/sack-and-trip.md`](sack-and-trip.md) | Event log write path and lookup recovery budget. |
| [AWS CodeDeploy deployment configurations on AWS Lambda](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html) | `LambdaCanary10Percent5Minutes`, `LambdaAllAtOnce` (PD-172). |
| [GitHub deployment environments](https://docs.github.com/actions/reference/environments) | Required reviewers, prevention of self-review (PD-020, PD-172). |
