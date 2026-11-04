# Landing Zone, Tags, Budgets and Auditing

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / JR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#8 (A08) |
| Depends on | [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md), `docs/legacy-2020.md` (A05, aroma-pacifico/aroma-pacifico-architecture#11), `docs/addressing-plan.md` (A07, aroma-pacifico/aroma-pacifico-architecture#13) |
| Feeds | aroma-pacifico/aroma-pacifico-roadmap#9 (A09, identities), aroma-pacifico/aroma-pacifico-roadmap#34 (A21, dashboards), aroma-pacifico/aroma-pacifico-roadmap#36 (A23, cost), aroma-pacifico/aroma-pacifico-roadmap#23, #25, #28 and #29 (B03, B05, B08, B09) |
| Decision-log entries | ASM-060 to ASM-069, PD-070 to PD-079 |

## 1. Purpose and requirements

This document defines the cloud operational baseline of the pilot: how accounts are separated, what is applied from the root of the organization, how resources are tagged so that cost can be split, which budgets and cost alerts exist, and how the four mandatory events are audited. Point 1 of the [case statement](case/case-statement.pdf) (Gobierno) is the requirement:

| Case statement text | Where it is met |
|---|---|
| "Landing zone: cuentas, suscripciones o proyectos separados para seguridad, red, producción y pruebas." (Separate accounts for security, network, production and test.) | [Section 2](#2-organization-structure) |
| "Políticas, identidades, etiquetas y presupuestos aplicados desde la raíz." (Policies, identities, tags and budgets applied from the root.) | [Section 3](#3-controls-applied-from-the-root), [section 4](#4-identity-entry-points) |
| "Las etiquetas alimentan el costeo por fincas, drones, acopio, viajes y ambientes." (Tags feed costing by farms, drones, collection centers, trips and environments.) | [Section 5](#5-tagging-scheme) |
| "Auditoría de lectura de un saco, cambio de un umbral, publicación de telemetría y entrega de un objeto de vuelo." (Audit of a sack read, a threshold change, a telemetry publication and the delivery of a flight object.) | [Section 7](#7-audit-of-the-four-mandatory-events) |
| Point 6: "Tablero operativo y tablero de costo, partidos por las etiquetas de fincas, drones, acopio y pruebas." (Operational and cost dashboards split by the tags of farms, drones, collection centers and testing.) | Tags and cost categories here; dashboards in A21 (aroma-pacifico/aroma-pacifico-roadmap#34, PD-077) |

The accepted inputs fix the frame and this document does not reopen them. ADR-001 places every environment in AWS, Region US East (N. Virginia), `us-east-1`, in one AWS Organization with separate accounts for security, network, production, test and development (ASM-005), and keeps Azure prepared but not deployed. ADR-003 gives each environment, coded `dev`, `qa` and `pdn` (ASM-006), its own state bucket in its own account, and reaches AWS from CI only through GitHub OIDC federation, with no static keys. The addressing plan of A07 gives `dev` 10.80.0.0/16, `qa` 10.81.0.0/16 and `pdn` 10.82.0.0/16, places the collection-center hub in the network account (ASM-033) and holds 10.83.0.0/16 as the first candidate for any VPC the network or security account needs (ASM-034). ADR-002 adds the agronomist identity and requires that each delivered flight object is audited.

The landing zone is designed here, not coded in Epic 6. The case statement limits the 100 hours to the network module, the security module, one ingestion or lookup module, one function pattern, variables and the pipeline of one environment; none of them creates the Organization. Whether any later task codes it is PD-078.

## 2. Organization structure

The pilot uses one AWS Organization with all features enabled, which is the mode that allows service control policies (SCPs) and tag policies. It is built directly with AWS Organizations, without AWS Control Tower, because Control Tower creates its own Log Archive and Audit accounts, which would not match the single security account of ADR-001 (ASM-061). AWS Organizations always has a management account; ADR-001 does not list it because it runs no workload, and this document records it as ASM-060.

```text
Root  (SCPs, tag policy)
|-- management account          (not in an OU; consolidated billing, budgets, cost allocation tags)
|-- Security OU
|   `-- security account        (audit archive, delegated administrator for CloudTrail and IAM Identity Center)
|-- Infrastructure OU
|   `-- network account         (collection-center hub, WAN attachments; ASM-033)
`-- Workloads OU
    |-- Production OU
    |   `-- production account  (pdn, 10.82.0.0/16)
    `-- NonProduction OU
        |-- test account        (qa, 10.81.0.0/16)
        `-- development account (dev, 10.80.0.0/16)
```

The organizational units (OUs) group accounts by the policies they receive, so a policy is attached once to an OU rather than to each account (ASM-062). Production has its own OU because it alone receives the production-change SCP of section 3; test and development share the NonProduction OU because they receive the same controls and together form the testing cost line of section 6.

| Account | OU | Environment tag | Holds | Does not hold |
|---|---|---|---|---|
| Management | Root | `shared` | AWS Organizations, consolidated billing, AWS Budgets, cost allocation tag activation, AWS Cost Categories, AWS Cost Anomaly Detection, the IAM Identity Center organization instance. | Any workload, any state bucket, any application identity. |
| Security | Security | `shared` | The audit archive bucket and its AWS KMS key, the delegated administration of AWS CloudTrail and IAM Identity Center, the audit query workspace (Amazon Athena). | Any workload. |
| Network | Infrastructure | `shared` | The hub that terminates the WAN links of the three collection centers and attaches the environment VPCs (ASM-033); its address space comes from the reserve of ASM-034. | Any workload data. |
| Production | Production | `pdn` | The `pdn` VPC, the `pdn` workloads, the `pdn` state bucket and its key, the `pdn` GitHub OIDC roles (ADR-003), the `pdn` application audit log group. | Test or development data. |
| Test | NonProduction | `qa` | The same layout for `qa`. | Production data. |
| Development | NonProduction | `dev` | The same layout for `dev`. | Production data. |

Account names, account identifiers and root email addresses are assigned when the accounts are created and are not written in this repository. Every account is in `us-east-1` (ADR-001). No Azure subscription or management group exists (ADR-001).

## 3. Controls applied from the root

The case statement asks for policies, identities, tags and budgets applied from the root. On AWS this means four mechanisms owned by the management account: SCPs and a tag policy attached through AWS Organizations, budgets created in the management account over the linked accounts, and one organization trail. SCPs never restrict the management account itself or service-linked roles, which is one more reason that the management account runs no workload (ASM-060).

### 3.1 Service control policies

| SCP | Attached to | What it denies | Why |
|---|---|---|---|
| Deny leaving the organization | Root | `organizations:LeaveOrganization`. | An account outside the organization escapes the trail, the budgets and the tag policy. |
| Deny Regions other than `us-east-1` | Root | Any action whose `aws:RequestedRegion` is not `us-east-1`, exempting global services (for example IAM, AWS Organizations, AWS Budgets and Cost Explorer). The exact exemption list is fixed when the policy is coded. | ADR-001 runs every environment in `us-east-1` only. |
| Protect the audit trail | Root | `cloudtrail:StopLogging`, `cloudtrail:DeleteTrail`, `cloudtrail:UpdateTrail`, `cloudtrail:PutEventSelectors` and `cloudtrail:PutInsightSelectors`, except for the trail administration role in the security account; in the security account, changes to the audit archive bucket policy, Object Lock configuration and the deletion or disabling of its KMS key. | The four mandatory events and every management action must stay recorded. |
| Deny long-term credentials | Root | `iam:CreateUser`, `iam:CreateAccessKey` and `iam:CreateLoginProfile`. | ADR-003 forbids static keys; humans enter through IAM Identity Center and CI through GitHub OIDC (section 4). |
| Deny the root user of member accounts | Root | Every action when the principal is the root user of a member account. Root credentials of member accounts are also removed with IAM centralized root access management. | No shared, unattributable identity can act in a workload account. |
| Require cost tags on creation | Workloads OU | Create actions, for the resource types that support tagging on creation, when the request carries no `Environment` or `CostCenter` tag (`aws:RequestTag` condition). | A tag policy validates values but does not make a tag mandatory; this SCP and the IaC policy gate of `standards/terraform-iac-best-practices.md` make the cost tags present. |
| Production changes only through the pipeline | Production OU | Write actions by human sessions from IAM Identity Center, except the narrow operations permission sets that A09 defines (for example revoking a device certificate, which ADR-003 makes an operational procedure) and a break-glass permission set used under an incident reference; reads are allowed. | ADR-003 applies production only from the approved pipeline job, inside the harvest-week window rule (ASM-007, ASM-008). |

### 3.2 Tag policy

One tag policy is attached to the root. It fixes the capitalization of every key of section 5 and the allowed values of `Environment`, `CostCenter` and `CollectionCenter`, and it is enforced (`enforced_for`) on the resource types the pilot creates that tag policies support. Enforcement makes a tagging operation with a non-compliant value fail; it does not create a missing tag, which is why presence is required by the SCP of section 3.1 and by the CI gate. The values of `Client` and `Project` are added to the policy once the infrastructure repository fixes them (PD-070).

### 3.3 Budgets and the organization trail

The budgets of [section 6](#6-budgets-and-cost-alerts) are created in the management account, which sees the cost of every linked account. The organization trail of [section 7.1](#71-organization-trail) is created from the security account as delegated administrator for CloudTrail; member accounts can see it but cannot change or stop it.

## 4. Identity entry points

The identities themselves, their permissions and MFA are designed by A09 (aroma-pacifico/aroma-pacifico-roadmap#9). This section only fixes where each kind of identity lives, so that the landing zone has exactly one entry point per kind.

| Kind of identity | Entry point | Lives in | Source |
|---|---|---|---|
| Humans who operate AWS (platform operator, auditor, break-glass) | IAM Identity Center, organization instance, with MFA; one permission set per role, assigned per account. | Organization instance in the management account, administered from the security account as delegated administrator. | This document; roles and MFA by A09. |
| CI and CD pipeline | GitHub OIDC identity provider and per-environment plan and apply roles. | Each environment account (`dev`, `qa`, `pdn`). | ADR-003. |
| Farm nodes and drones as devices | AWS IoT Core thing with an X.509 certificate registered from a CSR; revocation by certificate status. | Each environment account. | ADR-003, ASM-010; drones by A09. |
| Users of the three applications (farmer, transporter, collection-center operator, read-only buyer, drone operator, agronomist, platform operator in the reports web) | Application identity directory chosen by A09. | The workload account of each environment. | Case statement point 2, ADR-002, A09. |

No IAM user exists in any account (section 3.1). Human access to AWS and application access to the platform are separate paths: a farmer or a buyer never receives an AWS console identity.

## 5. Tagging scheme

### 5.1 Keys

The keys reuse the five transversal tags of `standards/iac-rules/pc-iac-004.md` section 2.2 (`Client`, `Project`, `Environment`, `Owner`, `CostCenter`) and its `Name` tag, with the same PascalCase spelling, and add only the two keys the case statement needs that no rule provides: one for the farm and one for the collection center. The cost line of the case statement (farms, drones, collection centers, trips) is carried by the existing `CostCenter` key, not by a parallel key.

| Key | Origin | Allowed values | Set by | Tag policy | Cost allocation tag |
|---|---|---|---|---|---|
| `Client` | PC-IAC-004 | Fixed by the infrastructure repository (PD-070). | Root stack `common_tags`, applied through provider `default_tags`. | Key capitalization; value after PD-070. | No |
| `Project` | PC-IAC-004 | Fixed by the infrastructure repository (PD-070). | Root stack `common_tags`. | Key capitalization; value after PD-070. | Yes |
| `Environment` | PC-IAC-004, PC-IAC-002 | `dev`, `qa`, `pdn` (ASM-006); `shared` in the management, security and network accounts (ASM-063). | Root stack `common_tags`, from `var.environment`. | Enforced values. | Yes |
| `Owner` | PC-IAC-004 | Role code of the accountable owner: `SR1`, `SR2`, `JR1`, `JR2`, as in the decision log. Never a person's name. | Root stack `common_tags`. | Key capitalization. | No |
| `CostCenter` | PC-IAC-004 | `farms`, `drones`, `collection-centers`, `trips`, `shared` (ASM-064). | Root stack `common_tags` sets the stack default; a module overrides it per resource through `additional_tags`, because a resource tag takes precedence over the provider default tag with the same key. | Enforced values. | Yes |
| `FarmId` | A08 | The stable farm identifier the platform registers; its format is fixed by the data design (A16). | Farm module (B08), through `additional_tags`, on every resource it creates for one farm (PD-071). | Key capitalization. | Yes |
| `CollectionCenter` | A08 | `cauca`, `huila`, `tolima`, matching the second octets 41, 42 and 43 of the case statement. | Collection-center association (B05) and any resource that serves one center only (PD-072). | Enforced values. | Yes |
| `Name` | PC-IAC-004 section 3.1 | Physical name of PC-IAC-003. | Each reference module, explicitly. | No | No |

`Environment`, `CostCenter`, `CollectionCenter`, `FarmId` and `Project` are activated as cost allocation tags in the management account. Activation applies only from that moment forward and takes up to a day to appear in billing data, so it is done before the first workload is planned for apply. `Name`, `Owner` and `Client` are not activated because they do not split cost.

### 5.2 Which cost dimension each tag answers

| Case statement dimension | Primary split | Tag used | Notes |
|---|---|---|---|
| Environments (ambientes) | Linked account: production, test, development. | `Environment` | The account split is exact; the tag confirms it and separates the `shared` accounts. |
| Farms (fincas) | `CostCenter = farms` for the farm line; `FarmId` per farm for resources dedicated to one farm. | `CostCenter`, `FarmId` | Shared ingestion is allocated per farm by usage (section 5.3). |
| Drones | `CostCenter = drones`: drone object ingress, object storage of flight objects, reception and inference functions. | `CostCenter` | Drones are a contracted service (ADR-002); the cloud cost is the platform side of the service. Per campaign or per farm, by usage (section 5.3). |
| Collection centers (acopio) | `CostCenter = collection-centers`; `CollectionCenter` per center. | `CostCenter`, `CollectionCenter` | WAN attachments and center-specific resources carry the center value. |
| Trips (viajes) | `CostCenter = trips`: transporter position ingestion and store. | `CostCenter` | A trip is a data record, not a resource, so it cannot carry a tag; per-trip cost is allocated by usage (section 5.3). |

An AWS Cost Category named for the pilot cost lines maps `CostCenter` values and linked accounts to the lines farms, drones, collection centers, trips, testing and shared; its default value `unallocated` catches any untagged cost, which A21 shows on the cost dashboard so that tagging gaps are visible.

### 5.3 Allocation of shared managed services

Tags alone cannot split one shared resource. The AWS IoT Core endpoint, the telemetry table, the threshold function and the public lookup each serve all 200 farms (1,000 at scale) from a single resource, and the bill shows that resource once. These resources carry `CostCenter` of their line (for example `farms` for the telemetry path) and no `FarmId`. Their monthly cost is then attributed per farm, per trip or per collection center in proportion to a usage count that the platform already stores, as recorded in ASM-065:

| Shared resource family | Cost line | Usage unit per entity | Where the count comes from |
|---|---|---|---|
| Telemetry ingestion, telemetry store, threshold function | `farms` | Readings accepted per farm in the month. | Telemetry store, keyed by node identifier (ADR-003), mapped to the farm. |
| Drone object ingress, flight object storage, inference functions | `drones` | Flight objects and stored bytes per farm and per campaign. | Object keys in Amazon S3 (one prefix per farm, A13) and the flight records of ADR-002. |
| Transporter position ingestion and store | `trips` | Position messages per trip. | Position store, keyed by trip. |
| Public lookup and sack store | `shared`, allocated to farms | Sack reads per farm of origin. | Application audit records of section 7 (sack read). |
| Network hub and WAN attachments | `collection-centers` | Direct by `CollectionCenter` where the resource is per center; shared hub cost split equally by the three centers. | Tags and the hub attachments. |

The formula is: cost of entity e = monthly cost of the shared resource family x (units of e / total units in the month). Cost comes from the AWS Data Exports Cost and Usage Report (CUR 2.0) filtered by `CostCenter`; units come from the stores above. A21 builds the dashboard and A23 applies the method to the harvest and normal months; neither may replace it by a per-farm tag on a shared resource.

## 6. Budgets and cost alerts

All budgets are monthly AWS Budgets cost budgets in the management account. No amount is given here: the case statement fixes none and states that 2020 prices are not a valid quote, so every amount comes from the cost document of A23 under PD-017 and is set through PD-076. A budget may use planned amounts per month, so a harvest month receives the harvest-month figure of A23 and a normal month the normal-month figure.

| Budget | Filter | Amount | Alerts (ASM-066) | Purpose |
|---|---|---|---|---|
| Farms | `CostCenter = farms`, production account. | A23 (PD-076) | Actual 80 %, actual 100 %, forecasted 100 %. | Cost of the farm line, required by the issue. |
| Drones | `CostCenter = drones`, production account. | A23 (PD-076) | Same. | Cost of the drone line, required by the issue. |
| Collection centers | `CostCenter = collection-centers`, production account; one child budget per `CollectionCenter` value. | A23 (PD-076) | Same. | Cost of the collection-center line, required by the issue. |
| Testing | Linked accounts test and development, all cost (ASM-064). | A23 (PD-076) | Same, plus a budget action described below. | Cost of testing, required by the issue. |
| Trips | `CostCenter = trips`, production account. | A23 (PD-076) | Same. | Completes the five case dimensions. |
| One budget per account | Linked account, all cost; six budgets. | A23 (PD-076) | Same. | A budget per account applied from the root; catches cost that escapes the tag filters. |

Notifications go to one Amazon SNS topic in the management account, encrypted with AWS KMS, whose subscribers are the platform operator role and the cost owner chosen by the lead at deployment; no address is written here. AWS Cost Anomaly Detection adds two monitors, one of type linked account and one of type cost allocation tag on `CostCenter`, which alert on unexpected spend before a percentage threshold is reached.

The testing budget carries an AWS Budgets action that, at actual 100 % and only after manual approval, applies an SCP to the NonProduction OU denying the creation of new resources. Production never receives an automatic budget action, because stopping production would stop alerts to farmers and the public lookup, which the case statement protects with an RTO of 4 hours.

## 7. Audit of the four mandatory events

Two layers write audit records, and the event decides which one is authoritative. AWS CloudTrail records calls to AWS APIs, so it is authoritative when the event is itself an AWS API call made by an identifiable principal, as an upload to Amazon S3 is. A sack read by the public lookup and a threshold change made by a Federation user are actions inside an application: the AWS call underneath them, if any, is made by the application's service role, so CloudTrail would show the service role and not the person or the reader. For those events the application writes the audit record. A device telemetry publication is an MQTT message to AWS IoT Core, which is not an AWS API call made by a person, so its audit comes from IoT Core itself.

### 7.1 Organization trail

One multi-Region organization trail records management events (read and write) of every account, with log file integrity validation, and delivers to the audit archive bucket in the security account. The bucket has versioning, S3 Object Lock in compliance mode with a default retention of 5 years (ASM-067), Block Public Access, SSE-KMS under a customer-managed key of the security account, and a bucket policy that denies requests without TLS. Data events are added only where an event of section 7.2 needs them, with advanced event selectors that name the resource by ARN prefix, so that no account pays for data events it does not need. The trail also records every action of the pipeline plan and apply roles of ADR-003, which makes infrastructure changes auditable as a side effect.

### 7.2 Event-by-event design

| Event | Writing layer | Source of the record | Who | What | When | From where | Kept in |
|---|---|---|---|---|---|---|---|
| Sack read | Application (lookup service) | Structured audit record written by the lookup service for every read through the public web lookup, the farmer app and the collection-center app. | Authenticated user identifier and role (buyer, farmer, collection-center operator); `anonymous` for an unauthenticated public read (ASM-069). | Sack code, view returned (buyer, collection center or farmer, as point 2 restricts), result (found, not found, denied), request identifier. No national identity number and no full trace, as point 2 requires. | Server timestamp in UTC. | Source IP, channel (web, farmer app, collection-center app), user agent. | `pdn` application audit log group, then the audit archive (section 7.3). |
| Threshold change | Application (reports web, threshold configuration) | Structured audit record written by the service that changes a threshold, in the same transaction as the change. Secondary: CloudTrail data events on the threshold store once A14 selects it (PD-074). | Federation or platform operator identity, with MFA (point 2). | Variable, zone (crop or drying), scope (pilot or farm), old value, new value, reason. | Server timestamp in UTC. | Source IP, channel, user agent. | `pdn` application audit log group, then the audit archive. |
| Telemetry publication | AWS IoT Core | The accepted reading itself: the telemetry topic rule writes, with each reading, the certificate principal, client identifier, source IP and the receive timestamp next to the original reading timestamp (ASM-011, PD-073). AWS IoT Core logging at level `ERROR` records rejected connections and publishes (authorization or TLS failures) in the `AWSIotLogsV2` log group. | Thing name and certificate principal of the farm node (ADR-003). | Topic, node identifier, reading. | Original reading timestamp and IoT Core receive timestamp. | Source IP of the connection (cellular carrier or gateway). | Telemetry store in the workload account (accepted publications); `AWSIotLogsV2` log group (rejections). |
| Flight object delivery | AWS (Amazon S3) and application | CloudTrail S3 data events `PutObject` and `CompleteMultipartUpload` on the drone object bucket of `pdn`, selected by ARN prefix. The reception function then links the object to its flight record (ADR-002). | IAM principal of the upload; the individual drone pilot is carried into the session (for example as source identity) or recorded when the upload authorization is issued (PD-075). | Bucket, object key, version identifier, size; flight record and order identifiers. | `eventTime`. | `sourceIPAddress`, `userAgent`. | Audit archive (trail prefix); the flight record in the transactional store. |

The volume follows the case figures. At harvest, telemetry is 200 farms x 6 readings plus 100 farms x 24 readings, that is 3,600 publications a day, each one audited by its stored record, and sack reads peak at 30 per minute. Neither figure needs a data-event trail on the telemetry table or the sack table, which is why those two events are audited where they happen rather than through CloudTrail data events.

### 7.3 Retention and readers

The case statement fixes 5 years for readings, photos, flights and sack history, and for personal data, but says nothing about audit records. This design keeps each audit record at least as long as the data it audits, 5 years (ASM-067). Application audit records of `pdn` are written to an AWS KMS-encrypted Amazon CloudWatch Logs log group with a 365-day retention for operation, and are copied through a CloudWatch Logs cross-account subscription to Amazon Data Firehose in the security account, which delivers them to the `application/` prefix of the audit archive under the same Object Lock retention. Records of `dev` and `qa` stay only in their log groups and are not archived, because they contain no production event (ASM-068). Accepted telemetry readings are kept 5 years in the telemetry store, as the case statement requires for readings. Data stays in `us-east-1` (ASM-002).

| Reader | How | Can read | Cannot |
|---|---|---|---|
| Auditor | IAM Identity Center permission set in the security account, read-only. | The audit archive through Amazon Athena; the organization trail. | Write, delete or change retention. |
| Platform operator | IAM Identity Center permission set in each workload account, with MFA. | Application audit log groups and `AWSIotLogsV2` for operations. | The archive bucket, the trail configuration, the KMS key policy. |
| Any identity, including the management account | None during retention. | None beyond its own role. | Delete or shorten a record under Object Lock compliance mode. |

The roles behind these permission sets, and who in the Federation holds the auditor role, are designed by A09.

## 8. Alternatives considered

| Alternative | Advantages | Disadvantages | Decision |
|---|---|---|---|
| AWS Control Tower | Managed account factory, guardrails and log archive. | Adds Log Archive and Audit accounts that do not match ADR-001's single security account; more moving parts than a six-account pilot needs. | Not used (ASM-061). |
| A per-farm tag on shared resources | Simple to explain. | One resource can hold one value; a shared endpoint or table serves all farms. | Replaced by usage allocation (ASM-065). |
| CloudTrail data events on the telemetry and sack tables | One audit source for everything. | Records the service role, not the device or the reader; pays for a data event per read and per reading. | Not used; application and IoT Core records instead. |
| A new parallel cost key such as `CostLine` | Leaves `CostCenter` free. | Duplicates a key PC-IAC-004 already makes mandatory. | Not used; `CostCenter` carries the line. |

## 9. Gaps and follow-up

| ID | Gap | Closed by |
|---|---|---|
| PD-070 | `Client` and `Project` values and their tag policy entries. | aroma-pacifico/aroma-pacifico-roadmap#23 (B03) |
| PD-071 | Farm module tags with `FarmId` and `CostCenter = farms`. | aroma-pacifico/aroma-pacifico-roadmap#28 (B08) |
| PD-072 | Collection-center association tags with `CollectionCenter`. | aroma-pacifico/aroma-pacifico-roadmap#25 (B05) |
| PD-073 | Telemetry rule stores the publication audit fields. | aroma-pacifico/aroma-pacifico-roadmap#29 (B09) with A13 |
| PD-074 | Threshold store and its secondary audit source. | aroma-pacifico/aroma-pacifico-roadmap#14 (A14) |
| PD-075 | Pilot identity carried into flight object uploads. | aroma-pacifico/aroma-pacifico-roadmap#9 (A09) |
| PD-076 | Budget amounts. | aroma-pacifico/aroma-pacifico-roadmap#36 (A23) |
| PD-077 | Operational and cost dashboards by farms, drones, collection centers and testing. | aroma-pacifico/aroma-pacifico-roadmap#34 (A21) |
| PD-078 | Whether the landing zone is ever coded, outside the 100 hours. | aroma-pacifico/aroma-pacifico-roadmap#20 (A20) |
| PD-079 | Re-check of the AWS features named here against live documentation. | aroma-pacifico/aroma-pacifico-roadmap#19 (A19) |

## 10. References

| Reference | Used for |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Points 1, 2 and 6; retention of 5 years; ingestion and lookup figures. |
| [`docs/decision-log.md`](decision-log.md) | ASM-060 to ASM-069, PD-070 to PD-079; ASM-002, ASM-005 to ASM-008, ASM-010, ASM-011, PD-017. |
| [ADR-001](adr/adr-001-primary-and-second-cloud.md) | Organization, accounts, Region, Azure not deployed. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | Agronomist and drone operator identities; audit of each flight object. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | State bucket per environment, GitHub OIDC, telemetry pattern, pipeline approval. |
| `docs/addressing-plan.md` (A07, aroma-pacifico/aroma-pacifico-architecture#13) | Environment CIDRs, network account hub, reserve blocks. |
| `docs/legacy-2020.md` (A05, aroma-pacifico/aroma-pacifico-architecture#11) | 2020 prices and cloud services retired (rows L-14 and L-16), so no 2020 figure is used for budgets. |
| `standards/iac-rules/pc-iac-002.md`, `pc-iac-003.md`, `pc-iac-004.md`, `pc-iac-025.md` | Environment values, naming, mandatory tags, root governance injection. |
| `standards/terraform-iac-best-practices.md` | Mandatory tags enforced as policy as code; audit logging. |
