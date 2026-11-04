# ADR-003: IaC tool, remote state and pattern module

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Owner | SR1 / SR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#4 |
| Supersedes | none |
| Superseded by | none |

## Context

Epic 6 is the only part of the challenge that is implemented in code, and the case statement bounds it tightly: the infrastructure as code must fit in 2 weeks and 100 hours, its scope is "modules, variables, remote state and an executable plan of one environment", and it must contain a network module, a security module, one pattern module ("ingestion or bag lookup, whichever the diagram leaves as the pattern"), the pattern of one inference function, test and production variables with no secrets in the repository or the state, and a pipeline that runs the plan and requires approval before applying to production. Challenge point 4 adds that there is no production change between 05:00 and 18:00 during harvest week, except for an incident.

The team is four people (SR1, SR2, JR1, JR2; ASM-005). The Epic 6 tasks in the roadmap (B01 to B13, aroma-pacifico/aroma-pacifico-roadmap#21 to #33) carry planning estimates that add up to 98 hours; the 100-hour plan itself is built and verified by A20 (aroma-pacifico/aroma-pacifico-roadmap#20), so this ADR uses those estimates only to show that the chosen tool and layout add no hidden work.

Three earlier inputs constrain the choice. The Terraform rules in [`standards/iac-rules/`](../../standards/iac-rules/README.md) (PC-IAC-001 to PC-IAC-026) and [`standards/terraform-iac-best-practices.md`](../../standards/terraform-iac-best-practices.md) are binding (`AGENTS.md` section 8) and are written for Terraform on AWS. ADR-001 makes AWS the primary cloud and prepares Azure as the second one. The academic delivery is plan-only: no `terraform apply`, no `terraform destroy` and no live cloud resources (`AGENTS.md` section 4).

On 2026-11-04 the owner decided that Terraform is the IaC tool, that telemetry ingestion is the single pattern module, and that the code lives in two repositories: reusable modules in the public `aroma-pacifico-terraform-modules`, consumed by git tag, and the pattern-environment root stacks in the private `aroma-pacifico-infrastructure`. This ADR records those decisions, designs the remote state and the CI and CD constraints around them, and walks through every binding rule.

The pattern matters beyond Epic 6 because the 2020 design is what the case statement says must be updated: it "assumed data coverage on most farms and a device publishing telemetry straight to the ingestion point". The record of what is kept from 2020 and what is retired is reserved at `docs/legacy-2020.md` (aroma-pacifico/aroma-pacifico-roadmap#5); the ingestion pattern below keeps the 2020 requirement of farm telemetry and alerts and replaces its connectivity assumption with store-and-forward.

## Decision

We use Terraform as the IaC tool, keep its state in one encrypted, versioned, natively locked S3 backend per environment with one state key per domain stack, and implement telemetry ingestion as the single Epic 6 pattern module. Reusable modules live in the public `aroma-pacifico-terraform-modules` repository and are consumed by SemVer git tag; root stacks, environment variables and pipelines live in the private `aroma-pacifico-infrastructure` repository.

### Tool and versions

Terraform core is constrained with `required_version = ">= 1.14.0"` in every root stack and module, following PC-IAC-006 and the floor set in the best-practices standard (section 23). That floor covers every feature this design relies on: native tests (1.6), mock providers (1.7), ephemeral resources (1.10), write-only arguments (1.11) and S3 native state locking, which became generally available in 1.11.0 when the DynamoDB locking arguments were deprecated. The CI pins one exact Terraform patch release at or above the floor; B01 (aroma-pacifico/aroma-pacifico-roadmap#21) records which one. Root stacks pin the AWS provider with a pessimistic `~>` range on the 6.x line and commit `.terraform.lock.hcl`; reference modules declare a compatible `>= 6.0.0` minimum and do not commit a lock file.

### Repository layout

| Repository | Visibility | Holds | Consumed as |
|---|---|---|---|
| `aroma-pacifico-terraform-modules` | Public | One directory per reference module under `modules/<name>/` (network, collection-center association, security baseline, firewall rules, farm, telemetry ingestion, inference function), each with the PC-IAC-001 layout, `sample/` and `tests/`, plus a module CI that never holds cloud credentials. | `git::https://github.com/aroma-pacifico/aroma-pacifico-terraform-modules.git//modules/<name>?ref=vX.Y.Z` |
| `aroma-pacifico-infrastructure` | Private | One root stack per domain (`stacks/network`, `stacks/security`, `stacks/workload`) and a `stacks/state-backend` bootstrap stack; per-environment `.tfvars` and partial backend files; the CI and CD workflows. | Not consumed; it is the only place where plans are produced. |

The modules repository is public because nothing in it is environment-specific or secret, and because a public source lets the private pipeline fetch modules over plain HTTPS without a deploy key or token, which removes one credential from the design. A single module repository with repository-wide SemVer tags deviates from the "one module, one repository" wording of PC-IAC-015; seven repositories, each with its own branch protection, CI and release flow, would consume a share of the 100 hours that the code itself needs. The rule's purpose, an immutable and audited version in every shared environment, is kept: every root stack references a `vX.Y.Z` tag, never a branch or a commit.

### Remote state

| Property | Design |
|---|---|
| Backend | `backend "s3" {}` declared empty in each root stack (PC-IAC-008, dynamic mode); bucket, key, region, `encrypt`, `kms_key_id` and `use_lockfile` are injected at `terraform init -backend-config=backends/<env>.s3.tfbackend`. The `.tfbackend` files hold names only, never credentials. |
| Separation by environment | One state bucket per environment (`dev`, `qa`, `pdn`; ASM-002), owned by that environment's account in the landing zone of A08 (aroma-pacifico/aroma-pacifico-roadmap#8). A mistake in `qa` cannot touch the `pdn` state, and the `pdn` bucket policy admits only the `pdn` roles. |
| Separation by domain | One key per root stack inside each bucket: `network/terraform.tfstate`, `security/terraform.tfstate`, `workload/terraform.tfstate`, `state-backend/terraform.tfstate`. |
| Locking | `use_lockfile = true`: S3 native locking writes a `.tflock` object next to the state key. No DynamoDB lock table is created, because its arguments are deprecated since Terraform 1.11. The plan role therefore needs `s3:PutObject` and `s3:DeleteObject` on the `*.tflock` keys, and nothing else that writes. |
| Encryption | `encrypt = true` with SSE-KMS under a customer-managed key per environment, a bucket policy that denies requests without TLS (`aws:SecureTransport`), and Block Public Access on the bucket. |
| Versioning and protection | Bucket versioning is enabled so that a corrupted state is recoverable, and the bucket carries `lifecycle { prevent_destroy = true }` (PC-IAC-010). |
| Bootstrap | The bucket and its key are declared in `stacks/state-backend`, which belongs to the security domain (PC-IAC-022 section 6.1). Its own state is migrated into the bucket it declares, so no stack keeps local state. |
| Cross-stack data | No `terraform_remote_state` (PC-IAC-019). The network and security stacks publish identifiers as tags and non-secret SSM parameters; consumers read them with filtered data sources in their root (PC-IAC-011, PC-IAC-017). |

### Secrets out of the repository and out of the state

The pattern is designed to need no secret at all. CI reaches AWS through GitHub OIDC federation, so there is no access key to store; DynamoDB, SNS, SQS and Lambda authorize through IAM roles, so there is no database password; and each farm node's private key is generated on the device and never leaves it. The controls below make the "no secrets in the repository or the state" requirement checkable rather than a promise.

| Risk | Control |
|---|---|
| A credential committed to git | No static keys exist (OIDC). `.tfvars` and `.tfbackend` files carry non-secret values only (PC-IAC-002 section 5.3, PC-IAC-016). A secret scanner runs on every pull request in both repositories. |
| A device private key in state | Device certificates are registered from a certificate signing request (`aws_iot_certificate` with `csr`), so Terraform only ever sees the public certificate. Creating a certificate without a CSR, which would put the private key in state, is forbidden by this ADR and must be rejected in review. |
| A future secret that a resource genuinely needs | It lives in AWS Secrets Manager or as an SSM `SecureString`. Terraform reads it through an `ephemeral` resource (Terraform 1.10+) and passes it only to a write-only argument (`*_wo` with `*_wo_version`, Terraform 1.11+), such as `value_wo` on `aws_ssm_parameter`, so it is never persisted in the plan or the state. A variable that carries a secret is `sensitive = true`, and no output exposes it (PC-IAC-007, PC-IAC-016). |
| A secret in a published plan | Saved plan files are never committed. The plan artifact is kept only in the private repository's CI run with short retention, and only its rendered text is attached to the pull request. |

### Pattern module: telemetry ingestion

The module codes the boundary of the farm telemetry pattern; the full design belongs to A13 and A14 (aroma-pacifico/aroma-pacifico-roadmap#13, #14) and the identities to A9 (aroma-pacifico/aroma-pacifico-roadmap#9). Every service named here was checked on 2026-11-04 against public AWS availability notices; none of them is closed to new customers (see PD-020 for the residual check). The services deliberately not used are listed under the alternatives.

| Boundary | Design | Coded where |
|---|---|---|
| Device identity per farm node | One AWS IoT Core thing per farm node, named from a stable node identifier, with one X.509 client certificate registered from a CSR (ASM-006). An IoT policy lets a connection publish only to its own topic, using the `${iot:Connection.Thing.ThingName}` policy variable. | Policy in the ingestion module (B09); thing and certificate attachment as connection points of the farm module (B08, aroma-pacifico/aroma-pacifico-roadmap#28). |
| Revocation | Decommissioning a node or recovering from a lost device sets its certificate to `REVOKED` and detaches it from the thing and the policy; from then on IoT Core refuses the TLS session, so the device stops publishing. The procedure is operational (A9), not a Terraform apply. | Architecture (A9). |
| Transport | MQTT over TLS with mutual X.509 authentication to the IoT Core data endpoint, with a TLS 1.2 minimum security policy. Telemetry is never published in clear text. | Ingestion module. |
| Ingestion endpoint | The account's regional IoT Core data endpoint. It is the same endpoint for 200 or 1,000 farms and for farms reached through a gateway chosen by A10 (aroma-pacifico/aroma-pacifico-roadmap#10), so scaling does not redesign addressing or the ingestion point. | Ingestion module (outputs the endpoint and topic scheme). |
| Routing rule | An IoT topic rule on the telemetry topic writes each reading to the telemetry store and invokes the threshold function. Its error action sends failed messages to an SQS dead-letter queue. Thresholds are not written in the rule SQL: the Federation configures them as data, which the function reads. | Ingestion module; role ARNs come from the security stack. |
| Telemetry store | An Amazon DynamoDB table, encrypted with a customer-managed KMS key, keyed by node identifier and the original reading timestamp, so a reading published twice is stored once. The case statement allows "a transactional database or a time series" for crop, drying and position telemetry. | Ingestion module. |
| Alert path, under 5 minutes | Rule, then the threshold function (the "lote de telemetría" trigger of Part E, designed by A17, aroma-pacifico/aroma-pacifico-roadmap#17), then an Amazon SNS alert topic toward the farmer channel designed by A14. The 5-minute figure is a target from the case statement, to be measured by the alert-delivery SLO of A21; it is not a measured result. | Function pattern in B10 (aroma-pacifico/aroma-pacifico-roadmap#30) if A17 selects this trigger; topic and channel in architecture. |
| 24-hour store-and-forward drain | Without signal, the node buffers readings for at most 24 hours. When the link returns it publishes them oldest first, each carrying its original reading timestamp (ASM-007). They enter by the same rule; the store's key makes the drain idempotent, and the function evaluates each drained reading, so an out-of-range reading raises its alert at drain time, within the 24 hours the case statement allows. At the case figures (crop every 4 hours, drying every hour), a node holds at most 30 readings after 24 hours offline. | Connection points in the farm module (B08); device behaviour is out of scope (firmware). |

### CI and CD constraints

| Constraint | Design |
|---|---|
| Gates on every pull request | In both repositories, cheapest first: `terraform fmt -check -recursive`, `terraform validate`, `tflint`, Trivy or Checkov, and plan-mode `terraform test` with mock providers. In the infrastructure repository, a `terraform plan` of each affected stack for the target environment follows, and its text is attached to the pull request. |
| No static keys | GitHub OIDC federation into per-environment roles. The plan role trusts only `repo:aroma-pacifico/aroma-pacifico-infrastructure` pull request and `main` tokens and can read state and write only lock files. The apply role trusts only the `pdn` or `qa` deployment environment subject. The public modules repository never receives cloud credentials, so a pull request from a fork cannot reach AWS. |
| Identity separation | Agents and the plan role cannot apply. Applying is a human-approved pipeline job (best practices section 18), and during the academic delivery it is never run. |
| Promotion | `dev` to `qa` to `pdn` with the same module tags; an environment differs only by its `.tfvars` and its backend file. |
| Approval before production | The `pdn` apply job runs in a GitHub deployment environment with required reviewers, and it applies the saved plan reviewed in the same run, never a fresh plan of whatever `main` holds at apply time. The approval record (who, when) is kept by GitHub. Whether the organization's plan allows required reviewers on a private repository is PD-019. |
| Harvest-week window | Before applying to `pdn`, the job checks the harvest calendar (ASM-004) and refuses to run between 05:00 and 18:00, Colombia time (ASM-003), on a harvest-week day. An incident overrides it only with an incident reference recorded in the approval. Plans are never blocked. |

## Alternatives considered

| Alternative | Advantages | Disadvantages | Reason rejected |
|---|---|---|---|
| Terraform (chosen) | The 26 binding rules and the best-practices standard are written for it; one tool also covers Azure for the prepared second cloud (ADR-001); native tests with mock providers give credential-free evidence. | Business Source License; state must be managed. | Chosen. The license allows internal use, and state is handled by the backend above. |
| OpenTofu | Open-source fork, largely compatible syntax. | The binding standards, examples and verified feature versions are for Terraform; feature lines have diverged since the fork. | Re-validating 26 rules against a second tool does not fit in 100 hours. |
| AWS CloudFormation or AWS CDK | Native to AWS; no separate state to operate. | AWS-only, so the prepared Azure cloud would need another tool; the PC-IAC rules would not apply. | Conflicts with ADR-001 and with the binding rules. |
| Pulumi | General-purpose languages. | Different state model and learning curve for two junior members; the rules would not apply. | Learning cost does not fit the budget. |
| Sack lookup as the pattern module | Exercises the public lookup, the buyer view, the RPO of 15 minutes and the RTO of 4 hours. | Does not connect to the farm module or to the connectivity update that is the core of the 2026 case. | The owner selected ingestion; it ties to B08 and to Epic 3. |
| S3 with a DynamoDB lock table | Long-established pattern. | Deprecated since Terraform 1.11; one more table to secure and pay for. | Superseded by `use_lockfile`. |
| HCP Terraform as the backend | Managed state, runs and approvals. | External SaaS outside the landing zone; PC-IAC-008 admits only S3. | Contradicts PC-IAC-008. |
| One state bucket for all environments | One bucket to bootstrap. | A shared bucket crosses the production and test accounts, and one policy error exposes production state. | Weaker isolation than one bucket per environment. |
| Modules and roots in one repository, modules referenced by local path | No tagging overhead. | Violates PC-IAC-015 for `qa` and `pdn`; module versions are not immutable. | Contradicts PC-IAC-015. |
| One repository per module | Literal reading of "one module, one repository". | Seven repositories with their own protection, CI and releases. | Overhead does not fit in 100 hours; tags keep immutability. |
| AWS IoT Events, IoT Analytics or Timestream for LiveAnalytics in the pattern | Purpose-built alerting, analytics or time-series storage. | All three closed to new customers (IoT Events and IoT Analytics have also reached end of support). | Not available to a new pilot. |
| AWS IoT Device Defender Detect for the 24-hour silent-node alarm | Built-in anomaly detection. | Announced as closed to new customers from 2026-08-31. | Not available; silent-node detection is designed by A21 on CloudWatch. |

## Consequences

The team works with one tool whose rules are already written, and the pattern module plugs directly into the farm module, so the alert path the defense must explain is the one the code implements. Each domain stack can be planned on its own, which keeps every plan small enough to review. On the other side, the design deliberately leaves three things to later tasks that it cannot close itself: how B13 obtains an executable plan without creating live resources (PD-018), whether the GitHub plan supports the production approval gate on a private repository (PD-019), and a re-check of the AWS lifecycle page before coding (PD-020).

| Dimension | Impact |
|---|---|
| Network | Farm telemetry reaches the public IoT Core endpoint over TLS; whether the "telemetry ingress" subnet required by challenge point 3 hosts an IoT Core data-plane interface endpoint for private paths, such as collection-center WAN links, is decided by A7 and A13. Lambda does not need a VPC for this pattern. |
| Security | No static credentials; per-node X.509 identity with revocation; private keys never in state; CMK encryption for state, store and queues; least-privilege roles owned by the security stack; plan and apply identities separated. |
| IaC | Three domain stacks and a bootstrap stack per environment, seven reference modules in one tagged repository, `>= 1.14.0` core floor, `~>` provider pins in roots, plan-mode tests with mock providers. |
| Costs | One S3 bucket and one KMS key per environment for state; no lock table. The pattern's cost drivers are IoT Core messages, rule actions, DynamoDB storage and Lambda invocations; no price is given here, and A23 (aroma-pacifico/aroma-pacifico-roadmap#36) estimates them under PD-017. |

### Compliance with the binding Terraform rules

| Rule | How this design complies, or why it does not apply |
|---|---|
| PC-IAC-001 Module structure | Every module in `modules/<name>/` has the ten mandatory files and a `sample/` directory. `providers.tf` exists but holds only a comment, because PC-IAC-005 forbids provider blocks in child modules. |
| PC-IAC-002 Variables | `type`, `description` and at least one `validation` on every variable; `client`, `project` and `environment` accepted by every module; `map(object)` for `for_each` collections; no environment-specific defaults. |
| PC-IAC-003 Naming | `snake_case` HCL identifiers; physical names `{client}-{project}-{environment}-{type}-{key}` built in the root's `locals.tf`. The `client` and `project` values are fixed in B01. |
| PC-IAC-004 Tagging | Roots set `default_tags` from `common_tags` (Client, Project, Environment, Owner, CostCenter); modules add `Name` and `additional_tags`. The cost tags for farms, drones, collection and environments (challenge point 1) come from A8. |
| PC-IAC-005 Providers | Roots declare the `principal` aliased provider with `region`, `assume_role` and `default_tags`; modules declare `configuration_aliases = [aws.project]` and set `provider = aws.project` on every resource. Roots contain no unaliased resources, so the default-provider recommendation of best practices section 8 is not needed. |
| PC-IAC-006 Versions | `>= 1.14.0` core everywhere; `~>` AWS 6.x pin in roots, `>= 6.0.0` in modules; backend only in roots, with encryption and locking. |
| PC-IAC-007 Outputs | Granular outputs with descriptions (endpoint, topic scheme, table ARN, queue ARN); no full objects and no sensitive outputs. |
| PC-IAC-008 State | Remote S3 backend only, in roots only, `encrypt = true`, `use_lockfile = true`, dynamic partial configuration injected by the pipeline. No `profile` static backend is used. |
| PC-IAC-009 Types and logic | Conversions and dynamic ID injection happen in `locals.tf` with `try()`, `can()` and explicit conversions. |
| PC-IAC-010 for_each and lifecycle | `for_each` over maps for collections; `count` only for on/off switches; `prevent_destroy` on the state bucket and the telemetry table; no `ignore_changes` on managed configuration. |
| PC-IAC-011 Data sources | Data sources only in roots, filtered by name tags or IDs, injected through variables. Modules hold only generic data sources such as `aws_region`. |
| PC-IAC-012 Locals | One `locals` block per file, ordered from the governance prefix to final payloads. |
| PC-IAC-013 Module block order | Source, providers, governance, transversal, configuration, meta-arguments, separated by blank lines. |
| PC-IAC-014 Dynamic blocks and splats | `dynamic` for variable nested blocks such as rule actions; `values(...)[*]` for collection outputs. |
| PC-IAC-015 Remote versioned modules | Every root source is a git URL with `?ref=vX.Y.Z`; no branch refs. The single repository with `//modules/<name>` subdirectories is the documented deviation above. `sample/` uses `source = "../"`, which the rule allows for local development and which is never applied. |
| PC-IAC-016 Secrets | No secrets in git, `.tfvars` or state; `sensitive = true` where a secret could appear; ephemeral resources and write-only arguments for any future secret; CSR-based certificates. |
| PC-IAC-017 Cross-domain communication | Tags and non-secret SSM parameters read through filtered data sources in the consumer root. |
| PC-IAC-018 Testing | `sample/` serves as the examples directory and `tests/` holds plan-mode `.tftest.hcl` tests; CI runs fmt, validate, a security scan and a published plan; production requires manual approval. |
| PC-IAC-019 Remote state restricted | Not used anywhere; no exception is requested. |
| PC-IAC-020 Hardening | KMS encryption at rest, TLS in transit, least-privilege IAM, Block Public Access on every bucket. WAF does not apply to the IoT Core endpoint, which is not an ALB, API Gateway or CloudFront endpoint; TLS mutual authentication and the IoT policy are its perimeter. Any VPC access to AWS services goes through VPC endpoints. |
| PC-IAC-021 Locals for module input | Complex payloads reach module blocks only through `local.*` values. |
| PC-IAC-022 Domain separation | Network, security and workload are separate root stacks with separate state keys, applied in that order. IAM roles, KMS keys, security groups and the state bucket belong to the security stack; the ingestion module and the function receive role and key ARNs as inputs. The IoT policy stays in the ingestion module because it is intrinsic to the IoT service and is not one of the resources the rule assigns to security. |
| PC-IAC-023 Single responsibility | The ingestion module has one responsibility, accepting a farm reading and persisting it durably: IoT policy, topic rule, error queue, telemetry table and log group. It creates no IAM role, security group, VPC or load balancer. |
| PC-IAC-024 Configuration traceability | Environment-varying collections come from `dev.tfvars`, `qa.tfvars` and `pdn.tfvars` through `locals.tf` to `main.tf`. |
| PC-IAC-025 Root governance | The root builds final resource names and injects them into module payloads; modules only append internal suffixes. |
| PC-IAC-026 sample/ pattern | Each module's `sample/` follows the tfvars, variables, data, locals, main flow with no hardcoded IDs. |
| Best practices sections 8 to 14 | Lock files in roots only; thin roots; state isolated per environment and domain; secrets kept out of state; `moved` blocks for any rename; plan-mode tests; `.tfstate`, `.terraform/` and saved plans never committed. |
| Best practices sections 16 to 22 | Agents plan only; OIDC pipeline identity; the human approves a specific saved plan; gates run cheapest first; plan review starts with destroys and replacements. |

## Compliance with the case statement

| Case statement passage | How this decision complies |
|---|---|
| "La implementación de infraestructura como código cabe en 2 semanas y en 100 horas. El alcance de ese esfuerzo es el código: módulos, variables, estado remoto y un plan ejecutable de un ambiente." (The IaC implementation fits in 2 weeks and 100 hours; its scope is the code: modules, variables, remote state and an executable plan of one environment.) | Terraform, for which the binding rules already exist; roadmap estimates for B01 to B13 total 98 hours (A20 verifies the plan); remote state per environment; one pattern environment. How the executable plan is produced under the plan-only boundary is PD-018. |
| "Módulo de ingesta o de consulta del saco. Uno de los dos, el que el diagrama deje como patrón." (Ingestion or bag lookup module, one of the two, whichever the diagram leaves as the pattern.) | Telemetry ingestion only; PD-016 checks that B09 codes nothing else. |
| "Variables de pruebas y de producción, sin secretos en el repositorio ni en el estado." and "Secretos fuera del repositorio y fuera del estado." (Test and production variables, with no secrets in the repository or the state; secrets outside the repository and the state.) | `qa.tfvars` and `pdn.tfvars` with non-secret values; OIDC; CSR-based certificates; ephemeral resources and write-only arguments; no sensitive outputs. |
| "Pipeline que ejecuta el plan y exige aprobación antes de aplicar en producción." (A pipeline that runs the plan and requires approval before applying to production.) | Plan on every pull request; `pdn` apply gated by required reviewers on the reviewed saved plan (PD-019). |
| "No hay cambio productivo entre las 05:00 y las 18:00 en semana de cosecha, salvo incidente." (No production change between 05:00 and 18:00 in harvest week, except for an incident.) | Pre-apply window check with an incident override (ASM-003, ASM-004). |
| "Sin señal, el dispositivo hace store-and-forward. El buffer local no retiene más de 24 horas." (Without signal the device does store-and-forward; the local buffer keeps at most 24 hours.) | Idempotent drain through the same rule and store, connection points in the farm module. |
| "Latencia de alerta, con ruta de entrega disponible: menos de 5 minutos desde la lectura fuera de rango. Sin señal, la alerta se emite al drenar el buffer, dentro de las 24 horas." (Alert latency with a delivery path: under 5 minutes from the out-of-range reading; without signal, the alert is issued when the buffer drains, within 24 hours.) | Rule, function and SNS alert path; the 5 minutes is a target measured later by A21. |
| "Baja de un nodo de finca o de un dron perdido: se revoca la identidad y el dispositivo deja de publicar." (Decommissioning a farm node or a lost drone: the identity is revoked and the device stops publishing.) | Per-node X.509 identity, revoked and detached. |
| "El patrón de red y de ingesta escala de 200 a 1.000 fincas sin rediseñar el direccionamiento ni el punto de ingesta." (The network and ingestion pattern scales from 200 to 1,000 farms without redesigning addressing or the ingestion point.) | One regional IoT Core endpoint and one topic scheme for any number of nodes. |
| "Los rangos de alerta los configura la Federación. No quedan fijos en el dispositivo." (The Federation configures the alert ranges; they are not fixed on the device.) | Thresholds are data read by the function, not rule SQL or device firmware. |
| "Telemetría de cultivo, secado y posición: base transaccional o serie de tiempo." (Crop, drying and position telemetry: transactional database or time series.) | DynamoDB telemetry store. |

## References

| Reference | Use |
|---|---|
| [`docs/decision-log.md`](../decision-log.md) | ADR-003 index row; ASM-002 to ASM-007; PD-003, PD-016, PD-017, PD-018, PD-019, PD-020. |
| [`docs/case/case-statement.pdf`](../case/case-statement.pdf) | Epic 6, challenge points 2 to 4, Parts A and E, pilot figures. |
| [`standards/iac-rules/`](../../standards/iac-rules/README.md), [`standards/terraform-iac-best-practices.md`](../../standards/terraform-iac-best-practices.md) | Binding Terraform rules walked through above. |
| aroma-pacifico/aroma-pacifico-roadmap#4 | This decision. |
| aroma-pacifico/aroma-pacifico-roadmap#21 to #33 | Epic 6 tasks B01 to B13 that implement it. |
| `docs/adr/adr-001-primary-and-second-cloud.md` (reserved) | Primary and second cloud. |
| `docs/legacy-2020.md` (reserved) | 2020 requirements kept and components retired. |
| [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3) | `use_lockfile`, `.tflock` permissions, deprecated DynamoDB arguments. |
| [Upgrading to Terraform v1.11](https://developer.hashicorp.com/terraform/language/v1.11.x/upgrade-guides) | S3 native locking generally available in 1.11. |
| [Terraform 1.11: ephemeral values and write-only arguments](https://www.hashicorp.com/en/blog/terraform-1-11-ephemeral-values-managed-resources-write-only-arguments) | Ephemeral resources (1.10), write-only arguments (1.11), `aws_ssm_parameter` `value_wo`. |
| [AWS IoT Events end of support](https://docs.aws.amazon.com/iotevents/latest/developerguide/iotevents-end-of-support.html) | Closed to new customers 2025-05-20, end of support 2026-05-20. |
| [AWS service availability updates, June 2026](https://aws.amazon.com/about-aws/whats-new/2026/06/aws-service-availability/) | IoT Device Defender Detect closed to new customers from 2026-08-31. |
| [Amazon Pinpoint end of support](https://docs.aws.amazon.com/pinpoint/latest/userguide/migrate.html) | Pinpoint not used; SMS APIs continue under AWS End User Messaging. |
| [GitHub deployment environments](https://docs.github.com/actions/reference/environments) | Required reviewers and plan availability (PD-019). |
