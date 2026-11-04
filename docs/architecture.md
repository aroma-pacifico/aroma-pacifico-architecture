# Architecture of the Aroma Pacífico Pilot

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / SR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#19 (A19, document half) |
| Complete diagram | `aroma-pacifico-diagrams/complete/architecture.drawio`, exported as `aroma-pacifico-diagrams/complete/architecture.svg` and `aroma-pacifico-diagrams/complete/architecture.png`, drawn in parallel in the diagrams repository with the D01 to D13 boxes of [section 14](#14-diagram-boxes-d01-to-d13-and-traceability-to-the-iac) |
| Builds on | [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) and every design document of the [README index](../README.md#documentation-index) |
| Decision-log entries | PD-008 and PD-132 (closed here), ASM-200, PD-200, PD-201; re-check notes on PD-079, PD-100, PD-130, PD-142, PD-150, PD-160, PD-165 and PD-172 |

## 1. Executive summary

The Federation restarts in 2026 a coffee traceability pilot designed in 2020 and never deployed: 200 farms in Cauca, Huila and Tolima, three collection centers, 60 transporters and delivery at the port of Buenaventura. The [case statement](case/case-statement.pdf) keeps the 2020 capabilities (crop and drying monitoring with an alert to the farmer, harvest and packing in 60 kg sacks, trip tracking, sack lookup by code, and a drone service) and replaces the connectivity assumption that no longer holds. This document states the whole architecture in the order of the seven points of the challenge and points to the section of the detail document that holds each decision. It is the text the final defense rests on; it adds no requirement and no new design.

**Cloud and accounts.** AWS is the primary cloud, every environment runs in one Region, US East (N. Virginia) `us-east-1`, and Azure is prepared only as an undeployed export target for the open-format sack history and the drone objects, never as a second platform ([ADR-001](adr/adr-001-primary-and-second-cloud.md)). One AWS Organization holds a management account and five member accounts: security, network, production (`pdn`), test (`qa`) and development (`dev`), with service control policies, a tag policy, budgets and one organization trail applied from the root ([landing zone](landing-zone.md#2-organization-structure)).

**Connectivity without the 2020 assumption.** Each farm receives two small solar nodes, crop and drying, that publish every reading with MQTT over TLS and an X.509 certificate to the regional AWS IoT Core endpoint over an LTE-based bearer with a multi-network SIM. Without signal a node keeps at most 24 hours of readings and drains them oldest first; farms out of coverage use the same node with a satellite IP bearer, without a new radio base or a weekly visit. Transporter, farmer and logistics apps keep an encrypted outbox that releases an event only when the platform confirms it. The drone mission is preloaded at the municipal seat or the collection center, and its flight record is published when the link returns. Each collection center gets a segmented LAN and two IPsec tunnels to an AWS Transit Gateway with a cellular backup; the port gets no Federation equipment, only the delivery event recorded by the transporter ([connectivity](connectivity.md#3-summary-of-the-flows)).

**Workloads.** Telemetry, sacks, trips, positions, cases and orders live in Amazon DynamoDB; photos, maps, polygons and flight records live in Amazon S3; all are kept 5 years. Alert thresholds are Federation data, versioned and audited, read by an event-driven Lambda function that alerts the farmer by push and SMS, with a design target of under 5 minutes. The drone service runs in combined mode: area mode delimits and estimates, precision mode confirms, fumigates and verifies; a contracted certified operator flies and a Federation agronomist authorizes each fumigation as a human identity with MFA ([ADR-002](adr/adr-002-drone-flight-mode-and-governance.md)). Four inference functions call Amazon Bedrock in `us-east-1` by foundation-model identifier, persist each result with timestamp, confidence and source object, and never authorize anything ([inference functions](inference-functions.md#3-the-four-triggers)).

**Security.** Application users sign in to two Amazon Cognito user pools: `staff`, with MFA required for the collection-center operator, drone operator, agronomist and platform operator, and `field`, passwordless, for farmer, transporter and buyer. People who operate AWS use IAM Identity Center with MFA; no IAM user or access key exists. Every store except the Cognito pools, which hold only sign-in data under AWS-managed keys, is encrypted with a customer-managed key of its data class, eight classes per environment. The buyer sees only origin, processing dates, status and drying variables through a separate projection keyed by an 80-bit random sack code ([security and identity](security-and-identity.md#2-kinds-of-identity)).

**Infrastructure as code.** Terraform, one natively locked S3 state per environment, GitHub OIDC with separate plan and apply roles, and telemetry ingestion as the single pattern module of Epic 6 ([ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md)). The thirteen boxes D01 to D13 of the complete diagram mark what the code builds, one box per Epic 6 task, and [section 14](#14-diagram-boxes-d01-to-d13-and-traceability-to-the-iac) names the document section that defines each one. The delivery is plan-only: nothing is applied or deployed.

**What is still open.** The answers of the clarification session, the Federation's thresholds and contracts, the live re-check of the AWS features this design relies on, the observability design, the release notes, the cost document and the design system are pending. [Section 17](#17-open-decisions-for-the-defense) lists every open decision by who closes it and what it blocks.

| Question the defense will be asked | Decision | Record |
|---|---|---|
| Which cloud, which Region, and what does "prepared second cloud" mean? | AWS in `us-east-1`; Azure as a declared, undeployed export target; recovery inside one Region. | [ADR-001](adr/adr-001-primary-and-second-cloud.md#what-prepared-without-duplicating-the-platform-means), ASM-002 to ASM-005 |
| How does a farm without signal or coverage deliver? | 24-hour encrypted buffer, drain oldest first; the same node with a satellite IP bearer where no LTE network attaches. | [Connectivity](connectivity.md#43-without-signal-store-and-forward-up-to-24-hours) sections 4.3 and 4.4, ASM-073, ASM-074 |
| Which flight mode, who flies, who authorizes? | Combined mode; contracted certified operator; Federation agronomist with MFA. | [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md#mode-boundary-and-switching-events) |
| Which IaC pattern, and why does it fit 100 hours? | Terraform, telemetry ingestion pattern, one environment planned (`qa`); the 100-hour plan is A20's. | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#pattern-module-telemetry-ingestion), ASM-051, aroma-pacifico/aroma-pacifico-roadmap#20 |
| What leaves Colombia? | Every object and record, stored and processed in `us-east-1`; only screened crop images and alert-opening readings go to the model. | [Inference functions](inference-functions.md#82-what-leaves-the-country) section 8.2, ASM-002, PD-101 |

## 2. How to read this document

Each of sections 3 to 9 is one point of the challenge ("Reto"), in the order of the case statement. Each one opens with what was decided and why in a few paragraphs and ends with a table of the documents and sections that hold the detail; nothing in those sections is redesigned here. Sections 10 to 17 are the cross-cutting views the defense needs: the component inventory, the pest-to-fumigation flow, the store-and-forward and no-signal behavior, the 2020 legacy, the diagram boxes, the decisions that lower cost, a defense guide and the open decisions.

The [case statement](case/case-statement.pdf) is the only source of requirements, and its Spanish text prevails over any English rendering (ASM-001). Every value that the case statement does not give is an `ASM-NNN` entry of the [decision log](decision-log.md), cited where it is used. No figure in this document is a measurement: latencies, recovery times, capacities and rollback times are design targets or arithmetic on the case figures, and the tests that would verify them are future work listed in section 17. Decision-log IDs are cited bare (ASM-074); roadmap issues are cited as `aroma-pacifico/aroma-pacifico-roadmap#<n>`; design tasks are also named by their code (A07, B09) as the detail documents do.

The complete diagram lives in the diagrams repository as `aroma-pacifico-diagrams/complete/architecture.png` (preview) and `aroma-pacifico-diagrams/complete/architecture.svg` (publication), from the source `aroma-pacifico-diagrams/complete/architecture.drawio`. It shows the addressing, the firewall, the store-and-forward paths, the pest-to-fumigation flow and the inference functions, and carries a numbered badge D01 to D13 on each component that Epic 6 codes. The numbers in this document and in the diagram are the same.

## 3. Point 1: Governance

**What we decided and why.** The landing zone is one AWS Organization, built directly with AWS Organizations and without AWS Control Tower, because Control Tower adds Log Archive and Audit accounts that do not match the single security account of ADR-001 (ASM-061). The security, network, production and test accounts that the case statement requires are joined by a development account (ASM-005) and by the management account that every Organization has, which runs no workload (ASM-060). Organizational units Security, Infrastructure and Workloads (Production, NonProduction) carry the policies once per group (ASM-062).

Seven SCPs are applied from the root or an OU: no leaving the organization, no Region other than `us-east-1`, protection of the audit trail, no long-term credentials, no root user in member accounts, cost tags required on creation, and production changes only through the pipeline. Tags reuse the five transversal keys of PC-IAC-004, with the cost line carried by `CostCenter` (`farms`, `drones`, `collection-centers`, `trips`, `shared`, ASM-064) plus `FarmId` and `CollectionCenter`. A shared resource such as the IoT endpoint or the lookup cannot be split by a tag, so its cost is allocated per farm, trip or center by usage counts the platform already stores (ASM-065). Budgets exist for farms, drones, collection centers, trips, testing and each account, alerting at 80 % and 100 % of actual and 100 % of forecast cost (ASM-066); their amounts come from the cost document (PD-076).

The four audited events have the layer that sees the real actor: the lookup service writes the sack-read record, the threshold service writes the version item of a threshold change in the same transaction, the telemetry rule stores the certificate principal and source of each publication, and CloudTrail S3 data events plus the upload-grant record cover each flight object delivery. Audit records are kept 5 years under S3 Object Lock in compliance mode in the security account (ASM-067). The record of which 2020 requirement is kept and which component is retired, which point 1 places in this document, is [section 13](#13-the-2020-pilot-what-is-kept-changed-and-retired).

| Topic | Detail |
|---|---|
| Accounts, OUs, what each account holds | [Landing zone](landing-zone.md#2-organization-structure) section 2 |
| SCPs, tag policy, budgets and the trail from the root | [Landing zone](landing-zone.md#3-controls-applied-from-the-root) section 3 |
| Tag keys, cost dimensions, allocation of shared services | [Landing zone](landing-zone.md#5-tagging-scheme) section 5 |
| Budgets and cost anomaly detection | [Landing zone](landing-zone.md#6-budgets-and-cost-alerts) section 6 |
| Audit of the four mandatory events, retention, readers | [Landing zone](landing-zone.md#7-audit-of-the-four-mandatory-events) section 7; threshold audit in [farm monitoring](farm-monitoring.md#45-audit-of-a-threshold-change) section 4.5 |
| 2020 kept and retired | [2020 legacy](legacy-2020.md#keep-retire-and-replace-table) and [section 13](#13-the-2020-pilot-what-is-kept-changed-and-retired) |

## 4. Point 2: Security

**What we decided and why.** Every actor enters through exactly one kind of identity: application users through Amazon Cognito, people who operate AWS through IAM Identity Center, devices through AWS IoT Core certificates, the pipeline through GitHub OIDC and services through IAM roles. Cognito needs two pools because MFA is a setting of the whole pool: the `staff` pool requires a TOTP second factor for the collection-center operator, the drone operator, the platform operator and the agronomist that ADR-002 adds, and the `field` pool signs in farmers and transporters with a one-time SMS code and buyers with an email code (ASM-090, ASM-091). Farmers and transporters do not use MFA on purpose: their scope is their own farms or trips, they have data only in town, and the farmer app must stay simple for people who distrust leaving the notebook; installation binding, the phone lock, a 30-day session and server-side scope checks compensate.

Least privilege is a matrix of role, data object and action, with scope (own farms, own trips, own center, assigned missions, department) held as data and checked on every request, so a removed assignment acts on the next call. Separation of duties is enforced by the API: only an agronomist with an MFA session at most 10 minutes old authorizes a fumigation, never the pilot who flies it and never a function (ASM-092). Encryption uses TLS 1.2 or later end to end with no third party terminating it, and a customer-managed KMS key per data class and environment: `telemetry`, `objects`, `records`, `personal`, `logs`, `state` (ASM-097), `public_view` (ASM-154) and `restricted` (ASM-165).

The sack code is 80 random bits in Crockford Base32 with a check symbol, issued only by the platform (ASM-094). It exposes neither the cédula nor the trace: the buyer view is a separate projection with only department, municipality, three processing dates, the current state and four drying variables (ASM-096), read by a role with no grant on any other table or key. Personal data is kept 5 years from deactivation and then anonymized (ASM-098). A decommissioned node or a lost drone kit has its certificate revoked and detached, after which IoT Core refuses its sessions; a departing pilot is disabled and signed out, with a 4-hour completion target (ASM-098).

| Topic | Detail |
|---|---|
| Kinds of identity, Cognito pools and groups | [Security and identity](security-and-identity.md#2-kinds-of-identity) sections 2 and 3 |
| Role catalog and least-privilege matrix | [Security and identity](security-and-identity.md#42-matrix-of-role-data-object-and-action) section 4 |
| MFA, and the justified absence of MFA for farmer and transporter | [Security and identity](security-and-identity.md#5-mfa-and-sign-in) section 5 |
| Device identities, drone kit and upload grants | [Security and identity](security-and-identity.md#6-device-identities) section 6 |
| Encryption in transit and at rest, key ownership and rotation | [Security and identity](security-and-identity.md#7-encryption) section 7 |
| Sack code and buyer view | [Security and identity](security-and-identity.md#8-sack-code-and-buyer-view) section 8; projection in [sack and trip](sack-and-trip.md#10-stores) section 10 |
| Personal data, retention and revocation | [Security and identity](security-and-identity.md#9-personal-data-and-retention) sections 9 and 10 |

## 5. Point 3: Network and connectivity

**What we decided and why: addressing.** The cloud never touches 10.40.0.0/13, which holds the three centers (10.41 Cauca, 10.42 Huila, 10.43 Tolima) and room for a fourth department at 10.44.0.0/16 (ASM-030), so one summary route reaches every center. The AWS supernet is 10.80.0.0/13 with one /16 per environment, so the second octet names the environment: `dev` 10.80.0.0/16, `qa` 10.81.0.0/16, `pdn` 10.82.0.0/16; 10.83.0.0/16 and 10.84.0.0/14 are held in reserve (ASM-034), and 10.88.0.0/13 is reserved for a future Azure network that is not deployed (ASM-035). Each VPC has the seven tiers of the case statement plus a `transit` tier (ASM-033), each tier a /20 split into four /22 slots, three in use across three Availability Zones and one reserved (ASM-031). Farms, phones and drones hold no cloud address and are authenticated by identity, so 1,000 farms add identities, not subnets (ASM-032).

| Tier | Block (E = 80, 81, 82) | What it holds in the pilot |
|---|---|---|
| `public_query` | 10.E.0.0/20 | Created and empty; the lookup front door is CloudFront, with no load balancer and no NAT gateway (PD-051 closed). |
| `telemetry_ingress` | 10.E.16.0/20 | Created and empty; farm telemetry uses the public IoT Core endpoint (CL-TEL-04). |
| `drone_object_ingress` | 10.E.32.0/20 | Created and empty; the ground station uses endpoints in `application` and `object_storage` (ASM-140, PD-140). |
| `application` | 10.E.48.0/20 | `execute-api` and Cognito interface endpoints for the centers; VPC-attached application functions. |
| `data` | 10.E.64.0/20 | Reserved for stores that need a subnet; no route to the internet; never reachable from a center (CL-DAT-01). |
| `object_storage` | 10.E.80.0/20 | S3 interface endpoint reachable from the centers. |
| `administration` | 10.E.96.0/20 | Route 53 Resolver inbound endpoint, CloudWatch Logs, CloudWatch and IAM Roles Anywhere endpoints, management session host (PD-123). |
| `transit` | 10.E.240.0/28 slots | Transit Gateway attachment, one /28 per Availability Zone. |

**What we decided and why: centers and firewall.** One LAN pattern serves the three centers with two parameters, the center key and the second octet. The seven zones of the case statement keep their /24, with VLAN 100 plus the third octet and one SSID per client zone; staff zones use WPA3-Enterprise with EAP-TLS validated by RADIUS on the center firewall, so a center works while its WAN is down, and visitors and transporters use WPA3-Personal in an internet-only routing context that has no route to logistics or management (ASM-080, ASM-081). The drone ground station sits in Soporte at 10.4x.2.10 (ASM-082) and the code-sheet printers in Servidores locales (ASM-087). At an assumed 50 % of planned clients active at the harvest peak, 34 of 66 clients need 68 Mbps down and 17 up against 90 Mbps planned of the 100 Mbps WAN (ASM-084, ASM-085); six traffic classes guarantee logistics, the drone uploads and staff their full reference and cap non-operational traffic (ASM-086).

Each center reaches the cloud through one AWS Site-to-Site VPN connection with two IKEv2 tunnels on a Transit Gateway in the network account, with static routing so that a center cannot widen its routes, and certificate authentication so that the cellular backup can re-establish the tunnels from another address (ASM-078, ASM-120). The centers reach only four `pdn` tiers (`application`, `drone_object_ingress`, `object_storage`, `administration`); `qa` only temporarily for acceptance tests, `dev` never (ASM-121). The policy tables give each rule its source, destination, port, action, validity (Permanent or Temporary with its removal condition) and whether it is audited before production; three rules are temporary. The cloud has no firewall appliance: security groups, network ACLs, endpoint policies, Transit Gateway route tables, bucket policies and AWS WAF enforce each rule where traffic enters, and there is no NAT gateway, no internet egress from private tiers and no AWS Network Firewall (ASM-127).

**What we decided and why: data paths.** The three data paths the case statement allows to differ are different. Farm telemetry goes from the node over cellular or satellite to the public IoT Core endpoint and never crosses a center. Transporter positions and custody events go from the phone over the public mobile network to the application API, one GNSS fix every 5 minutes en route, persisted in the outbox even without data (PD-010 closed). Drone objects go from the ground station to Amazon S3 with 15-minute pre-signed URLs and checksums, over a public link at the municipal seat or through the tunnel to the S3 interface endpoint at a center (ASM-093). The port has no Federation network; the transporter records the delivery and the receiver's outcome (ASM-057, ASM-077). No flow depends on 2G, 3G or circuit-switched data; the lowest terrestrial bearer is LTE-based and LTE continuity is an assumption to verify (ASM-072, PD-080).

| Topic | Detail |
|---|---|
| CIDRs, tiers, Availability Zones, non-overlap check, scale to 1,000 farms | [Addressing plan](addressing-plan.md#3-top-level-allocation) sections 3 to 6 |
| Farm, wet mill, drone, transporter, center and port flows | [Connectivity](connectivity.md#3-summary-of-the-flows) sections 3 to 10 |
| Mobile network generations | [Connectivity](connectivity.md#11-mobile-network-generations) section 11 |
| Center zones, VLANs, SSIDs, authentication, capacity, traffic control | [Collection-center network](collection-center-network.md#3-zones-vlans-and-ssids) sections 3 to 8 |
| Perimeter and cloud firewall tables, permanent and temporary rules, audit before production | [Firewall policy](firewall-policy.md#4-perimeter-firewall-of-each-collection-center) sections 4 to 6 |
| Ground station endpoints | [Drone service](drone-service.md#9-ingress-endpoints-of-the-ground-station-pd-121) section 9 |

## 6. Point 4: Automation and change

**What we decided and why.** Terraform is the tool because the 26 binding rules are written for it and one tool also covers the prepared Azure target. Modules live in the public `aroma-pacifico-terraform-modules`, consumed by SemVer tag, and root stacks per domain (network, security, workload, state backend) live in the private `aroma-pacifico-infrastructure` with one state key per stack inside one encrypted, versioned, natively locked S3 bucket per environment ([ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#remote-state)). The 200 farms are not instantiated: the farm module is the reusable pattern of one farm (ASM-050), and the one environment whose plan is evaluated is `qa` (ASM-051).

The three pipelines that point 4 names (platform, ingestion and flight object reception) share one release model: a release candidate is one merge commit of the infrastructure repository that pins module tags and, in `releases.tfvars`, the version and SHA-256 digest of every Lambda package and static bundle, so the same bytes move from `dev` to `qa` to `pdn` by construction (ASM-171). Terraform publishes each Lambda version and AWS CodeDeploy shifts the `live` alias with a 10 % canary for 5 minutes and automatic rollback in `qa` and `pdn` (ASM-174). A `pdn` run needs `qa` success for the same commit, a release note whose notice to the Federation was sent at least 2 business days ahead by default (ASM-173), a window check, and approval by SR1 or SR2 who is neither the author nor the trigger (ASM-172). The window check refuses any `pdn` job between 05:00 and 18:00 Colombia time on a harvest-week day of any department, or whose expected end would pass 05:00, except under an incident reference (ASM-007, ASM-055, ASM-178).

No static key exists anywhere: CI uses OIDC, device keys are generated on the device and registered from a CSR, and any future secret is read through an ephemeral resource into a write-only argument, so secrets stay out of the repository and the state. Rollback is by alias, by a revert commit through the promotion or by dead-letter redrive, with design targets of 10 minutes, 30 minutes and 2 hours; what cannot be rolled back (sack events, sent alerts, inference results) is corrected forward and listed. The two release notes templates of point 4, each with a completed example, are in [release notes](release-notes/README.md) (A22, aroma-pacifico/aroma-pacifico-roadmap#35); they take their fields from the pipeline hooks (PD-174 closed) and carry the harvest-week restriction with its incident path.

| Topic | Detail |
|---|---|
| Tool, repository layout, remote state, secrets, pattern module, CI and CD constraints, binding rules | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#decision) |
| Pipelines, repositories and identities | [Automation and pipelines](automation-and-pipelines.md#2-scope-of-each-pipeline) sections 2 and 3 |
| Release model, gates, function deployment, promotion and window | [Automation and pipelines](automation-and-pipelines.md#4-common-release-model) section 4 |
| Platform, ingestion and flight object reception pipelines | [Automation and pipelines](automation-and-pipelines.md#5-platform-pipeline) sections 5 to 7 |
| Approval, notice, rollback, secrets, hooks for alarms and release notes | [Automation and pipelines](automation-and-pipelines.md#8-approval) sections 8 to 12 |

## 7. Point 5: Workloads

**Part A, farm monitoring.** The harvest design load is 3,600 readings a day (200 farms x 6 crop readings plus 100 farms x 24 drying readings), and the deliberately extreme drain bound is 3,600 readings in one minute; at 1,000 farms the harvest day is 18,000 readings, on the same endpoint, rule, table and function. Thresholds are a versioned DynamoDB table `thresholds` with pilot and farm scopes; each change is a new immutable version written with its audit fields in one transaction, by the platform operator on the Federation's instruction, and each reading is evaluated with the version in force at its original timestamp (ASM-138). The function alerts once per episode (node, variable and condition), not per reading, with a 24-hour push reminder, so a drained day out of range is one alert (ASM-135). Alerts open by app push and SMS, because the farmer often has signal but no data (ASM-136). pH comes from a soil analysis entered by the agronomist (ASM-053, ASM-132), NPK alerts wait for the Federation's ranges (ASM-054), and drying lots are opened and closed with a local control on the drying node (ASM-130). The visible symptom of the plant enters through the drone confirmation that the alert opens, not through a weekly visit.

**Part B, drones.** The cadence of the case statement is kept: one review campaign a month in harvest, one every three months outside it, plus the fumigations and urgent confirmations of open cases as extra missions; campaigns are records per department and period (ASM-148, PD-012 closed). The five services, their gates and the flow from an alert to a closed case are in [section 11](#11-pest-to-fumigation-flow).

**Part C, sack and trip.** The sack has seven states, the five of Part C with their accepted or rejected outcome, recorded by eight transitions whose actor and scope the API checks. Every event keeps date, time and responsible, is generated with a client identifier and is never edited; corrections are compensating events (ASM-151). Because every actor records offline, the state is recomputed from the whole history ordered by the time of the act, and the party that physically receives the sack is authoritative (ASM-152). The public lookup is Amazon CloudFront with AWS WAF in front of a Lambda function URL that reads one item of the `buyer_view` projection and writes the audit record; the peak of 30 reads a minute is half a request a second. "Un despliegue no pierde eventos de saco" holds because the API acknowledges an event only after a conditional put in the immutable `sack_event_log`, and the outbox releases it only then. The RPO of 15 minutes rests on that log, point-in-time recovery and a 60-second export to S3 in JSON Lines; the RTO of 4 hours of the lookup is split into detection, decision, restore and switch targets (ASM-157). Both are design targets until the rehearsal of PD-153.

**Part D, collection centers.** The logistics posts and phones record receptions through the tunnel with MFA and an outbox; code sheets print locally on IPP; each center keeps a reserve of at least 400 platform-issued codes printed while the WAN is up, so a sack without a code can be labeled during an outage (ASM-153). The same pattern serves the three centers ([section 5](#5-point-3-network-and-connectivity)).

**Part E, functions and inference.** The four functions of Part E are event-driven Lambda functions behind one SQS queue each, which gives each its own retries with backoff and its own dead-letter queue. The drone functions start on `mission_record_persisted`, emitted only when the flight record and every object of its manifest are persisted and linked (ASM-149), through the mission table's stream, an EventBridge Pipe, the drone service bus and one rule per mission type; the telemetry function reads each reading from the table after the IoT rule stores it (ASM-161). Each function reads the persisted object or record, calls Amazon Bedrock with one vision-capable model, initially Amazon Nova Pro, by foundation-model identifier in `us-east-1` with zero-shot versioned prompts, and writes the result with timestamp, confidence and source object version in one conditional transaction (ASM-160). Nothing is trained and no compute waits for objects. A failure leaves the object and the state unchanged and ends in the dead-letter queue after five attempts, observed by named alarms (ASM-162). Objects that contain a person are screened on the ground station before upload, never sent to the model, and moved to a restricted prefix under its own key (ASM-164, ASM-165). The cost per thousand invocations in harvest is a formula with named unit prices for the cost document, because no price page could be read (ASM-169).

| Topic | Detail |
|---|---|
| Load, thresholds, evaluation, episodes, channel, farmer views, history | [Farm monitoring](farm-monitoring.md#3-load) sections 3 to 9 |
| Five services, records, state machines, mission package, flight record, prefixes, cadence | [Drone service](drone-service.md#3-the-five-services-as-the-platform-runs-them) sections 3 to 11 |
| Sack registration, states, events, trip, code, lookup, peaks, RPO and RTO, stores | [Sack and trip](sack-and-trip.md#3-sack-states-and-transitions) sections 2 to 10 |
| Logistics posts, code-sheet printing, WAN failure | [Collection-center network](collection-center-network.md#5-logistics-workstations-and-code-sheet-printing) sections 5 and 10; [sack and trip](sack-and-trip.md#63-codes-for-a-sack-that-arrives-without-one-with-or-without-wan) section 6.3 |
| Inference service, triggers, function design, result record, failures, Region, cost formula, persons | [Inference functions](inference-functions.md#2-managed-inference-service) sections 2 to 10 |

## 8. Point 6: Observability

**What we decided and why.** The observability design itself, with its objectives, thresholds, windows and dashboards, is A21 (aroma-pacifico/aroma-pacifico-roadmap#34) and is not yet written. What the architecture fixes is that every signal point 6 asks for already exists in a named place, so A21 configures alarms on data the platform keeps rather than adding instrumentation. The two SLOs have their SLIs proposed: alert delivery is the share of live out-of-range readings whose notification the farmer received within 300 seconds of the reading timestamp, with drained readings measured against 24 hours and per-hop trace points to attribute a breach (PD-082); lookup availability is measured on the lookup path of CloudFront, WAF, the lookup function and `buyer_view`, with its alarms on not-found ratio, WAF blocks, throttles and stream iterator age (PD-151). No objective percentage is set here.

| Point 6 requirement | Signal and where it is defined | Open in |
|---|---|---|
| Metrics, logs and traces of ingestion | Topic rule error queue, threshold queue age and dead-letter queue, `ReadingNotPersisted`, readings `not_evaluated` by cause ([farm monitoring](farm-monitoring.md#10-observability-hooks) section 10, [inference functions](inference-functions.md#7-error-observation-point-proposed-to-a21) section 7); `AWSIotLogsV2` | PD-162 |
| Applications and database | API 5XX and function errors and throttles of each alias ([automation and pipelines](automation-and-pipelines.md#12-hooks-for-observability-and-release-notes) section 12); stream iterator age of `sack_event_log`; pending, anomalous and clock-skewed sack events ([sack and trip](sack-and-trip.md#33-rules-that-make-the-table-work-offline) section 3.3) | PD-151, PD-173 |
| The three WAN links | Firewall logs and metrics exported over HTTPS to CloudWatch through the tunnel (CC-ADM-02), per traffic class, with drops and backup use ([collection-center network](collection-center-network.md#10-behavior-when-the-wan-fails) section 10) | PD-094, PD-123 |
| Arrival of flight objects | CloudTrail S3 data events and the reception function's persisted, rejected and quarantined counts ([drone service](drone-service.md#12-observability-hooks) section 12) | PD-141 |
| Nodes and farms without telemetry for more than 24 hours | Last reading per node, decommissioned nodes excluded by registry status ([farm monitoring](farm-monitoring.md#10-observability-hooks) section 10) | A21 |
| Undelivered crop alerts | Episodes without a delivery receipt from either channel; notification dead-letter queue ([farm monitoring](farm-monitoring.md#73-path) section 7.3) | PD-134 |
| Flights without a persisted record; fumigations without closure | Missions in `dispatched` or `flown`; orders in any state before `closed`, by age ([drone service](drone-service.md#12-observability-hooks) section 12) | PD-141 |
| Failed invocations that did not change state | Function dead-letter queues, `InferenceFailed`, `MissionsAwaitingResult` ([inference functions](inference-functions.md#7-error-observation-point-proposed-to-a21) section 7) | PD-162 |
| Operational and cost dashboards by farms, drones, collection centers and testing | `CostCenter`, `FarmId`, `CollectionCenter`, the account split and the cost category with its `unallocated` value ([landing zone](landing-zone.md#52-which-cost-dimension-each-tag-answers) section 5.2) | PD-077 |

## 9. Point 7: Design system

**What we decided and why.** The design system of the three applications is a separate deliverable of Epic 7, held in `aroma-pacifico-design-system`, and is not designed in this repository; no 2020 manual is inherited (legacy row L-18). The architecture hands it four inputs that it must respect. The sack has seven platform states, which map to the four sack badges of point 7 (en preparación, en viaje, aceptado, rechazado) as the sack design proposes (PD-154). The buyer view shows only the fields of ASM-096, so the buyer screen has no slot for farm, farmer, trip or weights. The farmer app adds only three monitoring views (farm status, alerts, history) and no settings, because the Federation owns the ranges ([farm monitoring](farm-monitoring.md#8-what-the-farmer-sees) section 8). The crop alert reaches the farmer as an SMS readable without the app and a push into the app, with "measured at" and "received at" shown when it was delayed. Outdoor contrast, icons understandable without a long manual, the logo and the palette are the design system's own decisions.

| Input to the design system | Source |
|---|---|
| Seven sack states to four badges | [Sack and trip](sack-and-trip.md#31-states) section 3.1, PD-154 |
| Buyer view fields and fields never shown | [Security and identity](security-and-identity.md#82-fields-the-buyer-view-returns) sections 8.2 and 8.3 |
| Farmer views, alert texts and the crop alert | [Farm monitoring](farm-monitoring.md#8-what-the-farmer-sees) section 8 |
| Label and code sheet content | [Sack and trip](sack-and-trip.md#62-the-label-and-the-code-sheet) section 6.2 |

## 10. Component inventory

Each workload component exists once in each of `dev`, `qa` and `pdn`, in that environment's account, unless the row says otherwise; `qa` is the pattern environment whose plan Epic 6 evaluates (ASM-051). Retention is 5 years wherever the case statement sets it (readings, photos, flights, sack history and personal data). The box column names the D box of the complete diagram whose Epic 6 task creates the resource (ASM-200); a dash means the component is architecture only and is not coded in the 100 hours.

| Component | AWS service or technology | Account and environment | Tier | Data store, key and retention | Box |
|---|---|---|---|---|---|
| Infrastructure repository and module repository | GitHub, Terraform | Not in AWS | None | Git history; no state, secret or saved plan committed | D01 |
| Remote state | Amazon S3 with native locking, AWS KMS | Each environment account | None | State per stack key, versioned, `state` key, `prevent_destroy` | D02 |
| Per-environment configuration and secrets | `.tfvars` and backend files, SSM parameters, AWS Secrets Manager | Infrastructure repository and each environment account | None | Non-secret values only in git; any secret read ephemerally into a write-only argument | D03 |
| Organization, SCPs, tag policy, budgets, cost categories, anomaly detection | AWS Organizations, AWS Budgets, AWS Cost Explorer | Management account, `shared` | None | Budget notifications on an SNS topic | Not coded (PD-078) |
| Organization trail and audit archive | AWS CloudTrail, Amazon S3 Object Lock, Amazon Data Firehose, Amazon Athena | Security account, `shared` | None | Audit archive, `audit` key, 5 years in compliance mode | Not coded (PD-078) |
| Human access to AWS | IAM Identity Center with MFA, permission sets | Management instance, administered from security | None | None | D06 (permission sets within its scope, PD-103) |
| Environment VPC, subnets, route tables, network ACLs | Amazon VPC | Each environment account | All eight tiers | VPC Flow Logs of rejected traffic, `logs` key | D04 |
| Interface endpoints and Resolver inbound endpoint | AWS PrivateLink endpoints (`execute-api`, `cognito-idp`, S3, CloudWatch Logs, CloudWatch, IAM Roles Anywhere), Amazon Route 53 Resolver | `pdn` (centers); same layout in `qa` and `dev` | `application`, `object_storage`, `administration` | None | D04 |
| Hub and center links | AWS Transit Gateway, AWS Site-to-Site VPN, customer gateways, managed prefix lists, AWS RAM | Network account, `shared`; one attachment per environment | `transit` | None | D05 |
| Center LAN, Wi-Fi and perimeter firewall | Firewall, PoE switch, access points, printers (device classes) | Each collection center | 10.4x.0.0/16 | Device configuration, not Terraform (PD-122) | D05 marks the association only |
| Cloud firewall rules | Security groups, network ACL entries, endpoint policies, Transit Gateway route tables, AWS WAF rules | Network account and each environment account | Per rule | None | D07 |
| Keys and service roles | AWS KMS customer-managed keys (eight classes), IAM roles | Each environment account | None | Key policies, 365-day rotation, 30-day deletion window | D06 |
| Application identities | Amazon Cognito user pools `staff` and `field` | Each environment account | Public endpoint; `application` endpoint for centers | Sign-in data only; profiles in the personal-data table | - |
| Farm node identity (one pattern farm) | AWS IoT Core thing and certificate from a CSR | Each environment account | None (public endpoint) | Node registry, `records` key | D08 |
| Telemetry ingestion | AWS IoT Core endpoint, IoT policy, topic rule with DynamoDB and SQS actions, error queue, log group | Each environment account | None (public endpoint) | Telemetry table, `telemetry` key, 5 years by TTL, `prevent_destroy` | D09 |
| Threshold function (the coded function pattern) | AWS Lambda, Amazon SQS queue and dead-letter queue, event source mapping, `bedrock:InvokeModel` on one foundation model | Each environment account | Not VPC-attached | Writes the alert store, `records` key, 5 years | D10 |
| Thresholds | Amazon DynamoDB `thresholds` | Each environment account | None | `records` key, versions kept 5 years after supersession | - (read by D10) |
| Alert topic, notification queue and dispatcher | Amazon SNS, Amazon SQS, AWS Lambda, SNS mobile push, AWS End User Messaging SMS | Each environment account | None | Messages in flight only | - |
| Daily summaries | AWS Lambda, Amazon EventBridge Scheduler | Each environment account | None | Summary items, `telemetry` key, 5 years | - |
| Drone object bucket | Amazon S3 with versioning, Block Public Access, lifecycle | Each environment account | `object_storage` endpoint for centers | `objects` key, 5 years, including `quarantine/`; `restricted/person/` under the `restricted` key | - |
| Reception function | AWS Lambda | Each environment account | Not VPC-attached | Writes missions, orders and polygon versions, `records` key | - |
| Cases, orders, missions, campaigns, upload grants | Amazon DynamoDB with a stream on the mission table | Each environment account | None | `records` key, 5 years | - |
| Drone inference functions | Amazon EventBridge Pipes, custom bus and rules, Amazon SQS, AWS Lambda, Amazon Bedrock | Each environment account | Not VPC-attached | Results on the case and the estimate, `records` key | - |
| Application API | Amazon API Gateway (public and private through `execute-api`), AWS Lambda, AWS WAF | Each environment account | `application` endpoint for centers | Writes the transactional tables | - |
| Sack and trip stores | Amazon DynamoDB `sack_event_log`, `sack_history`, `sacks`, `sack_codes`, `trips`, `transport_positions` | Each environment account | None | `records` key (positions `telemetry`), point-in-time recovery, deletion protection, 5 years by TTL | - |
| Sack history export | AWS Lambda exporter to an S3 prefix | Each environment account | None | JSON Lines in the drone object bucket, `objects` key, 5 years; declared Azure Blob target, not deployed | - |
| Public lookup | Amazon CloudFront, AWS WAF, Lambda function URL with origin access control, S3 assets bucket, `buyer_view` | Each environment account | None (`public_query` stays empty) | `buyer_view`, `public_view` key; audit record per read | - |
| Personal data | Amazon DynamoDB personal-data table | Each environment account | None | `personal` key, 5 years from deactivation, then anonymized | - |
| Observability | Amazon CloudWatch logs, metrics and alarms, `AWSIotLogsV2` | Each environment account | `administration` endpoints for centers | Log groups under the `logs` key, 365 days | - (A21) |
| CI pipeline | GitHub Actions, OIDC plan role | Infrastructure repository and each environment account | None | Saved plan as a short-lived run artifact | D11 |
| CD pipeline | GitHub deployment environments, OIDC apply role; AWS CodeDeploy if PD-175 includes it | Each environment account | None | Deployment record in GitHub | D12 |
| Validation and evidence of the executable plan | `terraform fmt`, `validate`, `tflint`, Trivy or Checkov, `terraform test`, `terraform plan` | Infrastructure and module repositories | None | Evidence in the pull requests (PD-019) | D13 |

## 11. Pest-to-fumigation flow

The flow below is the event chain of ADR-002 as the drone service, the farm monitoring design and the inference functions implement it. It is sequential, so it is a numbered list. Every arrow that changes a state is a persisted record, never a message that a device sends directly; every human step names its identity.

1. A crop node takes a reading outside a Federation range and publishes it at once with MQTT over TLS, or later from its 24-hour buffer with its original timestamp ([connectivity](connectivity.md#42-exit-protocol-and-path-to-the-cloud) section 4.2).
2. The IoT topic rule stores the reading in the telemetry table and sends its key to the threshold queue ([inference functions](inference-functions.md#3-the-four-triggers) section 3, ASM-161).
3. The threshold function reads the stored reading, evaluates it with the threshold version in force at the reading time, opens an episode and persists `crop_alert_raised`; the alert topic and the dispatcher notify the farmer by push and SMS, with a design target of under 5 minutes for a live reading ([farm monitoring](farm-monitoring.md#5-evaluation-of-a-reading) sections 5 to 7, ASM-075).
4. The case service opens a pest case for the plot that contains the node, or attaches the alert to the plot's open case, and switches the plot from area mode to precision mode. If the farm has no active polygon, a delimitation mission is planned first and the case waits in `awaiting_delimitation` ([drone service](drone-service.md#51-pest-case) section 5.1, ASM-027, ASM-141).
5. A confirmation mission is planned with the crop node location as the focus, in the next review campaign or as an extra mission, and the agronomist assigns a certified pilot ([drone service](drone-service.md#11-review-campaign-cadence-pd-012) section 11, ASM-026, ASM-148).
6. The pilot, signed in with MFA where there is a link (municipal seat or collection center), downloads the mission package with the polygon version, no-fly zones and focus; the mission is `dispatched` ([drone service](drone-service.md#61-mission-package) section 6.1).
7. The drone flies the focus. Without a link, photos, track and flight record stay in the encrypted buffer; when the link returns the kit receives an upload grant, uploads objects then the record, and the reception function verifies the manifest, quarantines anything outside the grant and moves objects with a person to the restricted prefix. The mission reaches `record_persisted` ([drone service](drone-service.md#10-without-a-link-mission-preloaded-record-published-on-restore) section 10).
8. `mission_record_persisted` reaches the confirmation function, which sends the screened photos to Bedrock and persists `focus_confirmed` or `focus_not_confirmed` with confidence and source object; it never authorizes anything ([inference functions](inference-functions.md#42-confirmation-function) section 4.2, ASM-163).
9. On `focus_not_confirmed` the agronomist closes the case without fumigation, and the plot returns to area mode, or asks for a new confirmation.
10. On `focus_confirmed` the platform creates a draft fumigation order with farm, plot, reason and the polygon version; the order cannot exist without a polygon and a closed confirmation record ([drone service](drone-service.md#52-fumigation-order) section 5.2, ASM-142).
11. The agronomist of the department, with an MFA session at most 10 minutes old, sets the window, the pilot and the planned product batch and authorizes the order; the authorizer is taken from the token and can never be the pilot or a function ([security and identity](security-and-identity.md#43-separation-of-duties) section 4.3).
12. The assigned pilot downloads the fumigation package and flies inside the window; the order moves to `dispatched` and then `flown` when the kit reports `flight_completed` or the first object arrives (ASM-143).
13. When the fumigation flight record is persisted, with who flew, the times, the batch used and the closure, the order becomes `executed` (flagged, not refused, if the times or batch differ) and a verification mission is planned for the same plot and focus (ASM-146). An order whose window ends without a record expires and needs a new authorization, unless it is `flown` (ASM-029).
14. The verification mission follows steps 6 and 7. The verification function persists `can_close` or `cannot_close`; the order becomes `closed` either way, and the case becomes `closed`, returning the plot to area mode, only on `can_close`. After `cannot_close` the agronomist asks for a new fumigation or a new verification ([inference functions](inference-functions.md#43-verification-function) section 4.3).
15. If any record or object does not arrive, or a function fails, nothing changes state: the fumigation stays open and is reported by observability, and the failed event waits in its dead-letter queue for redrive ([inference functions](inference-functions.md#62-what-a-failure-leaves-behind) section 6.2).

The mode switch from area to precision is the pair `crop_alert_raised` then `focus_confirmed`; the return is `focus_not_confirmed` with the agronomist's closure, or `case_closed` ([ADR-002](adr/adr-002-drone-flight-mode-and-governance.md#mode-boundary-and-switching-events)). Delimitation and estimation stay in area mode: a delimitation activates a new polygon version and starts no inference (ASM-147), and an estimation persists the expected sacks for the farm's collection center and never creates a packing order (ASM-166).

## 12. Store-and-forward and no-signal behavior

| Actor and device | Exit protocol | Local buffer | Retention or eviction | Without signal | Released when | What the platform keeps meanwhile | Source |
|---|---|---|---|---|---|---|---|
| Crop node | MQTT over TLS 1.2 or later, X.509, QoS 1, port 8883 (443 with ALPN) | Non-volatile encrypted ring buffer, at least 24 hours at the node cadence (6 readings) | No reading older than 24 hours; evicted readings counted and reported as a gap | Keeps reading; on reconnection publishes a live reading first, then drains oldest first | PUBACK of each reading | Nothing until drain; then the reading at its original timestamp, an alert raised at drain within 24 hours, marked delayed | [Connectivity](connectivity.md#43-without-signal-store-and-forward-up-to-24-hours) 4.3, ASM-074 |
| Drying node | Same | Same, 24 readings while a lot is open | Same | Same; the lot opens and closes with the local control | Same | Drying lot record; marked stale after 24 hours without a reading | [Connectivity](connectivity.md#5-wet-mill) 5, [farm monitoring](farm-monitoring.md#65-drying-lot) 6.5 |
| Node on a farm out of coverage | Same MQTT over TLS over a satellite IP bearer | Same | Same; procurement requires a contact at least every 4 hours | Readings wait for the next contact | Same | Same; no new radio base, no visit | [Connectivity](connectivity.md#44-farms-out-of-coverage) 4.4, ASM-073, PD-081 |
| Farmer app (sack registration) | HTTPS, TLS 1.2 or later | Encrypted outbox with client event identifiers | No time-based eviction | Records the sack with weight, code and time | API confirms persistence | A pickup for a sack not yet registered is held pending | [Connectivity](connectivity.md#5-wet-mill) 5, ASM-079, ASM-152 |
| Transporter app | HTTPS, TLS 1.2 or later | Encrypted outbox, at least 7 days of positions plus every custody event | Custody events never evicted; positions thinned to one per 15 minutes before any drop | GNSS fix every 5 minutes persisted as `transport_position`; custody acts as `sack_custody` | API confirms persistence; custody first, then positions | Trip rebuilt by `recorded_at` and `sequence_no`; out-of-order events applied by the receiver rule | [Connectivity](connectivity.md#7-transporter) 7, ASM-076 |
| Drone kit and ground station | HTTPS to API and S3 with pre-signed URLs and checksums; MQTT for grants | Drone and ground station storage, encrypted by the operator | No time limit | Mission already loaded; flight proceeds; record and objects stay in the buffer | Persistence receipt listing each object | Order stays `flown` and open; no `executed` or `closed` without the record | [Connectivity](connectivity.md#6-drone) 6, [drone service](drone-service.md#10-without-a-link-mission-preloaded-record-published-on-restore) 10 |
| Collection-center logistics posts and phones | HTTPS through the IPsec tunnel; cellular backup | Encrypted outbox | No time-based eviction | Receptions recorded with time and responsible; staff still join Wi-Fi through local RADIUS | API confirms persistence | Sack without code labeled from the 400-code reserve and bound when the link returns | [Collection-center network](collection-center-network.md#10-behavior-when-the-wan-fails) 10, ASM-153 |
| Port of Buenaventura | Transporter app, HTTPS over the public mobile network | Transporter outbox | Same as the transporter | Delivery and receiver outcome recorded with their real time | Same | Last sack state until the event drains | [Connectivity](connectivity.md#10-port-of-buenaventura) 10, ASM-077 |
| Farmer phone receiving an alert | SMS and push | None on the platform side; app keeps the last synchronized history | Not applicable | SMS arrives with any cellular signal; push when data returns | Delivery receipt from either channel | Alert persisted and counted as undelivered until a receipt | [Farm monitoring](farm-monitoring.md#71-decision-closes-pd-085) 7.1, ASM-136 |

## 13. The 2020 pilot: what is kept, changed and retired

Point 1 asks this document to record which 2020 requirement is kept and which component is retired. The 2020 design is known only through the case statement (ASM-040); the full record, with the source page of each row, is the [2020 legacy table](legacy-2020.md#keep-retire-and-replace-table). "Changed" below is the legacy decision Replace: the 2020 element is dropped and the case statement requires a 2026 element in its place.

| ID | 2020 element | Decision | 2026 answer | Where |
|---|---|---|---|---|
| L-01 | Traceability from farm to buyer | Kept | The sack is the traced unit from registration to port delivery. | [Sack and trip](sack-and-trip.md#3-sack-states-and-transitions) |
| L-02 | Cauca, Huila and Tolima | Kept | Three centers, second octets 41, 42 and 43. | [Addressing plan](addressing-plan.md#2-reserved-collection-center-space) |
| L-03 | Crop and drying monitoring with an alert | Kept | Two nodes per farm, Federation thresholds, episodes, push and SMS. | [Farm monitoring](farm-monitoring.md#1-purpose-and-requirements) |
| L-04 | Harvest record and 60 kg sacks | Kept | Registration with measured weight; 60 kg is the reference (ASM-150). | [Sack and trip](sack-and-trip.md#2-registering-the-sack-at-the-farm) |
| L-05 | Trip tracking to the center and to Buenaventura | Kept | Two trip types, 5-minute positions, custody events. | [Sack and trip](sack-and-trip.md#5-the-trip) |
| L-06 | Sack lookup by code | Kept | Unauthenticated lookup limited to the buyer view (ASM-058). | [Sack and trip](sack-and-trip.md#7-public-lookup) |
| L-07 | Drone service, five services | Kept | Combined mode, contracted operator, agronomist authorizes. | [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) |
| L-08 | Three applications | Kept | Platform ready for them; not built. | [Automation and pipelines](automation-and-pipelines.md#2-scope-of-each-pipeline) |
| L-09 | Data coverage on most farms | Changed | Every flow states its no-signal behavior; satellite IP bearer for farms out of coverage. | [Connectivity](connectivity.md#44-farms-out-of-coverage) |
| L-10 | Device publishing directly to ingestion | Changed | Store-and-forward with at most 24 hours of buffer. | [Connectivity](connectivity.md#43-without-signal-store-and-forward-up-to-24-hours) |
| L-11 | Collection center without LAN or WAN | Changed | Segmented LAN pattern and IPsec WAN to the Transit Gateway with cellular backup. | [Collection-center network](collection-center-network.md) |
| L-12 | Port without LAN or WAN | Changed | No Federation equipment; the delivery event comes from the transporter app. | [Connectivity](connectivity.md#10-port-of-buenaventura) |
| L-13 | Dependence on a mobile generation | Retired | No flow depends on 2G, 3G or circuit-switched data (ASM-041, ASM-072). | [Connectivity](connectivity.md#11-mobile-network-generations) |
| L-14 | 2020 prices | Retired | Costs at current prices by the cost document (PD-017). | aroma-pacifico/aroma-pacifico-roadmap#36 |
| L-15 | 2020 devices | Retired | Device classes only; models and prices by the cost document. | [Connectivity](connectivity.md#41-field-equipment) |
| L-16 | 2020 cloud services | Retired | AWS chosen and justified anew. | [ADR-001](adr/adr-001-primary-and-second-cloud.md) |
| L-17 | The 2020 architecture as a whole | Retired | This document and the complete diagram. | This document |
| L-18 | 2020 design manual | Retired | New design system of Epic 7. | [Section 9](#9-point-7-design-system) |

## 14. Diagram boxes D01 to D13 and traceability to the IaC

Each box marks, in the complete diagram, one component that an Epic 6 task codes. The table names, for each box, the documents and sections that define what the code must build and the pending decisions it must close or respect. Where a component spans two boxes, the box that creates the resource is the one named, as recorded in ASM-200. The 100-hour plan of A20 (aroma-pacifico/aroma-pacifico-roadmap#20) assigns hours, owners and order to the same boxes.

| Box | Track B issue | What the box marks | Documents and sections that define it | Pending decisions |
|---|---|---|---|---|
| D01 | aroma-pacifico/aroma-pacifico-roadmap#21 (B01) | `aroma-pacifico-infrastructure` with one root per environment, consuming `aroma-pacifico-terraform-modules` by tag | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#repository-layout) Tool and versions, Repository layout, Compliance with the binding Terraform rules; [automation and pipelines](automation-and-pipelines.md#31-repositories-and-triggering-paths) section 3.1 | PD-170 |
| D02 | aroma-pacifico/aroma-pacifico-roadmap#22 (B02) | One S3 state bucket per environment with native locking | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#remote-state) Remote state; [landing zone](landing-zone.md#2-organization-structure) section 2; [security and identity](security-and-identity.md#73-key-ownership-and-rotation) section 7.3 (`state` key) | PD-019 |
| D03 | aroma-pacifico/aroma-pacifico-roadmap#23 (B03) | Per-environment configuration and secrets, out of the repository and the state | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#secrets-out-of-the-repository-and-out-of-the-state) Secrets; [automation and pipelines](automation-and-pipelines.md#11-secrets-out-of-the-repository-and-out-of-the-state) section 11; [landing zone](landing-zone.md#51-keys) section 5.1 (`common_tags`) | PD-070 |
| D04 | aroma-pacifico/aroma-pacifico-roadmap#24 (B04) | VPC and tiers of the pattern environment | [Addressing plan](addressing-plan.md#4-subnet-layout-inside-each-environment-vpc) sections 3, 4 and 4.3; [firewall policy](firewall-policy.md#52-policy-table) CL-LNK-07, CL-APP-03, CL-DRN-04, CL-ADM-04, CL-PUB-04, CL-DAT-01 to 03, CL-EGR-01, CL-DEF-02; [drone service](drone-service.md#9-ingress-endpoints-of-the-ground-station-pd-121) section 9; [sack and trip](sack-and-trip.md#72-front-door-cloudfront-no-load-balancer-closes-pd-051) section 7.2 | PD-050, PD-120, PD-122, PD-123, PD-140 |
| D05 | aroma-pacifico/aroma-pacifico-roadmap#25 (B05) | Association of the environment with one pattern collection center | [Connectivity](connectivity.md#9-collection-center-wan) section 9; [collection-center network](collection-center-network.md#2-one-pattern-three-centers) section 2; [firewall policy](firewall-policy.md#51-enforcement-mechanisms) section 5.1 and CL-LNK-01 to 06, CL-ADM-01; [landing zone](landing-zone.md#51-keys) section 5.1 (`CollectionCenter`) | PD-072, PD-122 |
| D06 | aroma-pacifico/aroma-pacifico-roadmap#26 (B06) | Identities, KMS keys and initial access controls | [Security and identity](security-and-identity.md#2-kinds-of-identity) sections 2, 4.3, 5.3, 6, 7.2 and 7.3; [inference functions](inference-functions.md#11-epic-6-function-pattern) section 11 (function roles); [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#ci-and-cd-constraints) CI and CD constraints (OIDC roles) | PD-103, PD-165 |
| D07 | aroma-pacifico/aroma-pacifico-roadmap#27 (B07) | The pattern firewall rules | [Firewall policy](firewall-policy.md#52-policy-table) section 5.2 (rules marked Permanent) and section 6; center rules of section 4 are device configuration | PD-122, PD-125 |
| D08 | aroma-pacifico/aroma-pacifico-roadmap#28 (B08) | The reusable farm module: one pattern farm | [Connectivity](connectivity.md#41-field-equipment) sections 4.1 and 4.3; [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#pattern-module-telemetry-ingestion) Pattern module (device identity, drain); [security and identity](security-and-identity.md#6-device-identities) section 6; [landing zone](landing-zone.md#51-keys) section 5.1 (`FarmId`) | PD-071 |
| D09 | aroma-pacifico/aroma-pacifico-roadmap#29 (B09) | Telemetry ingestion, the pattern module of ADR-003 | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#pattern-module-telemetry-ingestion) Pattern module; [inference functions](inference-functions.md#3-the-four-triggers) section 3 (SQS action); [farm monitoring](farm-monitoring.md#3-load) sections 3 and 6.6; [landing zone](landing-zone.md#72-event-by-event-design) section 7.2 | PD-016, PD-021, PD-073, PD-161 |
| D10 | aroma-pacifico/aroma-pacifico-roadmap#30 (B10) | The event-driven inference function pattern | [Inference functions](inference-functions.md#11-epic-6-function-pattern) sections 11, 4.5, 6.1 and 7; [farm monitoring](farm-monitoring.md#5-evaluation-of-a-reading) section 5 | PD-161, PD-018 |
| D11 | aroma-pacifico/aroma-pacifico-roadmap#31 (B11) | CI pipeline producing an executable, reviewable plan | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#ci-and-cd-constraints) CI and CD constraints; [automation and pipelines](automation-and-pipelines.md#42-gates) section 4.2 | PD-019 |
| D12 | aroma-pacifico/aroma-pacifico-roadmap#32 (B12) | CD pipeline with the `pdn` approval and the harvest-week window | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#ci-and-cd-constraints) CI and CD constraints; [automation and pipelines](automation-and-pipelines.md#44-promotion-procedure-and-the-harvest-week-window) sections 4.4, 8 and 9 | PD-020, PD-171, PD-175 |
| D13 | aroma-pacifico/aroma-pacifico-roadmap#33 (B13) | Final validation and evidence of the executable plan | [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md#compliance-with-the-binding-terraform-rules) Compliance with the binding Terraform rules; [automation and pipelines](automation-and-pipelines.md#1-purpose-and-requirements) section 1 (plan-only boundary); ASM-051 | PD-016, PD-019, PD-200 |

## 15. Decisions that lower or avoid cost

Criterion 6 of the case statement asks for at least three decisions that lower or avoid spend. No price is quoted here; the cost document (aroma-pacifico/aroma-pacifico-roadmap#36) prices each line under PD-017.

| Decision | What it avoids | Record |
|---|---|---|
| No NAT gateway and no AWS Network Firewall; AWS services through endpoints | Hourly charges per Availability Zone and per gigabyte for egress and inspection that no rule needs | ASM-127, [firewall policy](firewall-policy.md#53-aws-network-firewall-is-not-proposed) section 5.3 |
| CloudFront, WAF and a Lambda function URL for the lookup, no load balancer | An hourly load balancer in every Availability Zone for half a request a second | [Sack and trip](sack-and-trip.md#72-front-door-cloudfront-no-load-balancer-closes-pd-051) section 7.2 |
| No interface endpoint in `drone_object_ingress` or `telemetry_ingress` | Duplicate endpoints charged per hour and Availability Zone | ASM-140, CL-TEL-04 |
| Model called on alert episode openings only, idempotency checked before the call, image caps per call | Model cost scaling with 108,000 readings a harvest month, and repeated calls on retries | ASM-167, ASM-168, [inference functions](inference-functions.md#92-formula) section 9.2 |
| One node module class for every farm, satellite subscription activated only on farms out of coverage | A site survey per farm and a satellite fee on every farm | ASM-073 |
| Contracted drone service instead of a Federation fleet | Capital cost of drones, certification and maintenance | [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md#alternatives-considered) |
| Azure prepared as an export target, not a second platform; one Region | A second platform, pipeline, identity set and Region | [ADR-001](adr/adr-001-primary-and-second-cloud.md#what-prepared-without-duplicating-the-platform-means) |
| On-demand DynamoDB, one database technology; no AWS Control Tower | Capacity planning, a second database and its backups; two extra accounts | [Sack and trip](sack-and-trip.md#10-stores) section 10, ASM-061 |
| Alert deduplication by episode and SMS coalescing | One SMS per reading or per drained reading | ASM-135 |

## 16. Defense guide

The case statement gives 20 minutes before a Federation architect and names three things the team must sustain: the connectivity, the flight mode and why the infrastructure code fits in 100 hours. The table below points each of them, and each evaluation criterion, to the section of this document and the record behind it.

| What the defense must sustain | Short answer | Sections |
|---|---|---|
| Connectivity | Identity, not address; every flow has an exit protocol, a buffer and a no-signal behavior; nothing depends on a switched-off generation; out-of-coverage farms use the same node with a satellite IP bearer. | [5](#5-point-3-network-and-connectivity), [12](#12-store-and-forward-and-no-signal-behavior) |
| Flight mode | Combined mode, because only combined mode covers the five mandatory services; the switch is `crop_alert_raised` then `focus_confirmed`; a contracted certified pilot flies and a Federation agronomist authorizes with MFA. | [7](#7-point-5-workloads), [11](#11-pest-to-fumigation-flow) |
| Why the IaC fits in 100 hours | One tool whose rules exist, one pattern environment (`qa`), one pattern module (ingestion), one function pattern (threshold), no landing zone, no 200 farms and no three centers in code; the hours themselves are A20's plan. | [6](#6-point-4-automation-and-change), [14](#14-diagram-boxes-d01-to-d13-and-traceability-to-the-iac) |

| Evaluation criterion | Sections |
|---|---|
| 1. Strategy coherent with the updated 2020 pilot: farm without cable, low investment, limited public lookup | [12](#12-store-and-forward-and-no-signal-behavior), [13](#13-the-2020-pilot-what-is-kept-changed-and-retired), [15](#15-decisions-that-lower-or-avoid-cost), [7](#7-point-5-workloads) (Part C) |
| 2. Connectivity closed for farm, drone, transporter, center and port, including no signal | [5](#5-point-3-network-and-connectivity), [12](#12-store-and-forward-and-no-signal-behavior) |
| 3. Viability from the case figures; flight mode sustained; IaC within 100 hours and 2 weeks | [7](#7-point-5-workloads), [11](#11-pest-to-fumigation-flow), [14](#14-diagram-boxes-d01-to-d13-and-traceability-to-the-iac) |
| 4. Automation and change: code matches the diagram; templates approve, notify and roll back | [6](#6-point-4-automation-and-change), [14](#14-diagram-boxes-d01-to-d13-and-traceability-to-the-iac) |
| 5. Security: separate identities, encrypted readings and photos, firewall, buyer view | [4](#4-point-2-security), [5](#5-point-3-network-and-connectivity) |
| 6. Current cost: visible assumptions, farm hardware, drones, function invocations in harvest, three cost-saving decisions | [15](#15-decisions-that-lower-or-avoid-cost); prices in the cost document (PD-017) |
| 7. Design system applied to farmer, transporter and sack lookup | [9](#9-point-7-design-system) |
| 8. Clarity of the defense | [1](#1-executive-summary), this section |

## 17. Open decisions for the defense

Every pending decision of the [decision log](decision-log.md#pending-decisions) that is still open on 2026-11-04 is listed once, grouped by who closes it. None is closed here unless this task actually resolved it (PD-008 and PD-132, section 19).

### 17.1 Closed by the Federation

Most of these are asked in the 30-minute clarification session ([clarification questions](clarification-questions.md#3-critical-questions)); each has a fallback assumption, so the design is complete without the answer and the answer can validate or replace it.

| PD | Decision | Blocks |
|---|---|---|
| PD-005 | Answers of the clarification session recorded and fed back into the log. | Validation of ASM-050 to ASM-059 and of ADR-001 to ADR-003 as presented. |
| PD-030 | Aerocivil regulation for agricultural spraying with unmanned aircraft verified (Q-10). | The permit sentence of the defense; ASM-020. |
| PD-080 | LTE continuity in the territory verified against MinTIC or CRC, or answered in the session. | ASM-072 and the bearer class of the nodes. |
| PD-101 | Legal basis for personal data in `us-east-1`, start of the 5-year clock, anonymization (Q-02). | ASM-002 and ASM-098; the statement of what leaves Colombia. |
| PD-131 | NPK ranges with units and the drying closure tolerance published. | NPK alerts (rows T-06 to T-08) and the closure notice tolerance. |
| PD-155 | Weight tolerance around 60 kg; rejected sack does not continue to the port. | Weight flags and the terminal rejected state. |
| PD-166 | Confirmation and verification thresholds, yield factors, retention of objects with a person. | Confirmation and closure decisions; the expected-sacks number for the centers. |
| PD-171 | Recipient, channel and lead time of the change notice (Q-15). | The `pdn` notice check; ASM-173. |
| PD-143 | Drone operator contract: ground-station software for the mission package, record, manifest and receipt. | Every drone flow in practice. |
| PD-164 | Same contract: on-device person screen and incident reporting. | Sending any drone object to inference. |

### 17.2 Closed by the owner and the lead

| PD | Decision | Blocks |
|---|---|---|
| PD-019 | How the executable plan is evidenced: a real plan against a human-provisioned account, or plan-mode tests with mock providers. | D13 and the evidence of D02, D11. |
| PD-020 | GitHub plan allows required reviewers on the private infrastructure repository, or an alternative gate that binds a named human. | D12, the `pdn` approval. |
| PD-170 | Home of application and function code (`aroma-pacifico-platform` or another). | The platform, ingestion and reception code pipelines. |
| PD-200 | Assign a session that can read `docs.aws.amazon.com` and `docs.github.com` to run the consolidated live re-check (section 18). | Coding of D04, D06, D09, D10 and D12 on verified feature behavior. |

### 17.3 Closed by the team

| Closed by | PD | Decision | Blocks |
|---|---|---|---|
| A19 diagram half (aroma-pacifico/aroma-pacifico-roadmap#19) | PD-006 | The complete diagram references the 2020 legacy table. | Closure of roadmap#19. |
| A19 follow-up | PD-201 | Sections 8, 9 and 15 of this document link the observability, design system and cost deliverables once they exist, and section 17.1 is updated with the session answers. | A complete document for the defense. |
| Live re-check (PD-200) | PD-018, PD-079, PD-100, PD-130, PD-142, PD-150, PD-160, PD-172 | AWS and GitHub features named by ADR-001, the landing zone, security, farm monitoring, drone, sack, inference and automation designs re-checked against live documentation. | Confidence in the stated behavior; ASM-160 if the model is not offered in-Region. |
| A20 (aroma-pacifico/aroma-pacifico-roadmap#20) | PD-078, PD-175 | Landing zone stays out of the 100 hours; B13 scope (infrastructure path only, or also CodeDeploy and the notice check). | The 100-hour plan. |
| B03 to B10 (aroma-pacifico/aroma-pacifico-roadmap#23 to #30) | PD-016, PD-021, PD-050, PD-070, PD-071, PD-072, PD-073, PD-103, PD-120, PD-122, PD-123, PD-140, PD-161, PD-165 | Coding choices and checks of D03 to D10 listed in section 14. PD-165: the role matrix part is done in this task; the `restricted` key in B06 or its exclusion in the plan remains. | The Epic 6 code. |
| A13 (aroma-pacifico/aroma-pacifico-roadmap#13) | PD-081 | Satellite IoT service with an IP bearer and dual-mode modules verified, or a republishing bridge designed. | The out-of-coverage farm path. |
| A09 follow-up (aroma-pacifico/aroma-pacifico-roadmap#9) | PD-090 | Private CA and enrollment of center devices and the ground station for EAP-TLS. | Staff Wi-Fi admission at the centers. |
| A21 (aroma-pacifico/aroma-pacifico-roadmap#34) | PD-077, PD-082, PD-094, PD-134, PD-141, PD-151, PD-162, PD-173 | SLO objectives, alarm thresholds and windows, dashboards. | Point 6 and the rollback alarms of CodeDeploy. |
| A23 (aroma-pacifico/aroma-pacifico-roadmap#36) | PD-017, PD-031, PD-076, PD-084, PD-095, PD-124, PD-133, PD-152, PD-163 | Every price, hardware cost, drone campaign cost and invocation cost, with its assumptions. | Epic 7 cost and budget amounts. |
| Design system task (Epic 7) | PD-154 | Seven sack states mapped to four badges. | Point 7. |
| Before `pdn` go-live, no roadmap task | PD-092, PD-125, PD-153, PD-167, PD-176 | Site survey of each center; audit of the rules marked Audit; recovery rehearsal; model evaluation; rollback rehearsal. | Go-live; none blocks the defense, and none may be reported as done. |

## 18. Live documentation re-check (PD-172 and the other re-check entries)

PD-172 asks that the features this design relies on are re-checked against live documentation before this document is consolidated, and PD-079, PD-100, PD-130, PD-142, PD-150 and PD-160 ask the same of the earlier designs. On 2026-11-04 this task tried to read the official pages for the PD-172 features: [CodeDeploy deployment configurations](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-configurations.html), [CodeDeploy Lambda deployments](https://docs.aws.amazon.com/codedeploy/latest/userguide/deployment-steps-lambda.html), [Lambda aliases](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html), [Lambda `PublishVersion`](https://docs.aws.amazon.com/lambda/latest/api/API_PublishVersion.html), [IoT `ReplaceTopicRule`](https://docs.aws.amazon.com/iot/latest/apireference/API_ReplaceTopicRule.html), [CloudFront `Origin`](https://docs.aws.amazon.com/cloudfront/latest/APIReference/API_Origin.html) and [GitHub deployment environments](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment). None could be read: the session proxy refused the connection to `docs.aws.amazon.com` and `docs.github.com`, and the fetch tool could not resolve either host. No feature was verified, so PD-172 and the other re-check entries stay open, and PD-200 asks the lead to assign a session that can read them. The design does not change until that check reports.

## 19. Decision-log entries

| ID | Summary |
|---|---|
| PD-008 | The architecture document links the 2020 legacy table and states what is kept and retired. Closed by section 13. |
| PD-132 | The role matrix adds the soil analysis object and the dispatcher's read of the farmer phone. Closed by the update of [security and identity](security-and-identity.md#42-matrix-of-role-data-object-and-action) section 4.2 in this task. |
| ASM-200 | When a component spans two diagram boxes, the inventory names the box whose task creates the resource: VPC, subnets, route tables, network ACLs and VPC endpoints in D04; Transit Gateway attachment, VPN connections, customer gateways and prefix lists in D05; KMS keys, IAM roles and permission sets in D06; security group rules, endpoint policies, Transit Gateway route-table routes and WAF rules in D07. |
| PD-200 | The lead assigns a session that can read the live AWS and GitHub documentation to run the re-checks of PD-018, PD-021, PD-079, PD-100, PD-120, PD-130, PD-142, PD-150, PD-160 and PD-172. |
| PD-201 | This document is updated with links to the observability, design system and cost deliverables and with the clarification session answers. |
| PD-079, PD-100, PD-130, PD-142, PD-150, PD-160, PD-172 | Re-check attempted on 2026-11-04 and not possible from this session (section 18); still open. |
| PD-165 | Role matrix part met in [security and identity](security-and-identity.md#42-matrix-of-role-data-object-and-action) sections 4.2, 7.2 and 7.3; still open for B06. |

## 20. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | The seven points, Parts A to E, the drone section, Epic 6, the delivery tasks and the evaluation criteria. |
| [`docs/decision-log.md`](decision-log.md) | Every ASM, ADR and PD cited. |
| [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | Cloud, flight mode and governance, IaC. |
| [`docs/legacy-2020.md`](legacy-2020.md) | 2020 kept, replaced and retired. |
| [`docs/landing-zone.md`](landing-zone.md), [`docs/security-and-identity.md`](security-and-identity.md) | Points 1 and 2. |
| [`docs/addressing-plan.md`](addressing-plan.md), [`docs/connectivity.md`](connectivity.md), [`docs/collection-center-network.md`](collection-center-network.md), [`docs/firewall-policy.md`](firewall-policy.md) | Point 3. |
| [`docs/automation-and-pipelines.md`](automation-and-pipelines.md) | Point 4. |
| [`docs/farm-monitoring.md`](farm-monitoring.md), [`docs/drone-service.md`](drone-service.md), [`docs/sack-and-trip.md`](sack-and-trip.md), [`docs/inference-functions.md`](inference-functions.md) | Point 5. |
| [`docs/clarification-questions.md`](clarification-questions.md) | Questions and fallbacks behind section 17.1. |
| `aroma-pacifico-diagrams/complete/architecture.drawio`, `.svg`, `.png` | The complete diagram with boxes D01 to D13. |
| `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13) | Data paths of telemetry, positions and drone objects. |
