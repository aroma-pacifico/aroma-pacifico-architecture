# Security and Identity

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR2 / JR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#9 (A09) |
| Builds on | [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md), [`docs/landing-zone.md`](landing-zone.md) (A08), [`docs/connectivity.md`](connectivity.md) (A10), [`docs/clarification-questions.md`](clarification-questions.md) (A06) |
| Feeds | A13 data paths (aroma-pacifico/aroma-pacifico-roadmap#13), A15 drone service (aroma-pacifico/aroma-pacifico-roadmap#15), A16 sack and public lookup (aroma-pacifico/aroma-pacifico-roadmap#16), A19 architecture document (aroma-pacifico/aroma-pacifico-roadmap#19), B06 security module (aroma-pacifico/aroma-pacifico-roadmap#26) |
| Decision-log entries | PD-075 (closed here), ASM-090 to ASM-099, PD-100 to PD-104 |

## 1. Purpose and requirements

This document defines the minimum security and identity model of the pilot: who the identities are, what each one may read and write and at which scope, where MFA is required, how telemetry, drone objects and personal data are encrypted, what the buyer sees of a sack, how long personal data lives, and how a device or a person is revoked. Point 2 (Seguridad) of the [case statement](case/case-statement.pdf) is the requirement. Quotations are in the original Spanish, which prevails over the English rendering (ASM-001).

| Case statement text | Where it is met |
|---|---|
| "Identidades separadas y mínimo privilegio: caficultor, transportador, operario de acopio, comprador de solo lectura del saco, operador de drones y operador de plataforma." (Separate identities and least privilege for the six roles.) | [Section 3](#3-application-identity-directory), [section 4](#4-roles-and-least-privilege) |
| "MFA para operario de acopio, operador de drones y operador de plataforma." (MFA for collection-center operator, drone operator and platform operator.) | [Section 5](#5-mfa-and-sign-in) |
| "Cifrado en tránsito (TLS) y en reposo. Telemetría de finca y objetos del dron no se publican en claro." (Encryption in transit and at rest; farm telemetry and drone objects are never published in clear.) | [Section 7](#7-encryption) |
| "El identificador del saco no expone la cédula del caficultor ni la traza completa al comprador. El comprador recibe origen, fechas de proceso, estado y variables de secado. Acopio recibe el viaje. El caficultor recibe sus fincas." (The sack identifier exposes neither the farmer's national ID nor the full trace; the buyer receives origin, processing dates, status and drying variables; the center receives the trip; the farmer receives their farms.) | [Section 4.2](#42-matrix-of-role-data-object-and-action), [section 8](#8-sack-code-and-buyer-view) |
| "Datos personales de caficultores y transportadores. Retención: 5 años." (Personal data of farmers and transporters; retention 5 years.) | [Section 9](#9-personal-data-and-retention) |
| "Baja de un nodo de finca o de un dron perdido: se revoca la identidad y el dispositivo deja de publicar." (Decommissioning a farm node or a lost drone: the identity is revoked and the device stops publishing.) | [Section 10](#10-revocation) |
| Part E: "La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta." (Fumigation authorization is a human identity; no trigger executes it.) | [Section 4.3](#43-separation-of-duties) |

The accepted inputs fix the frame and this document does not reopen them. ADR-002 adds the Federation agronomist as a seventh identity with MFA and requires that the person who authorizes a fumigation is not the person who flies it. ADR-003 gives each farm node an X.509 certificate registered from a CSR, revokes it by certificate status, reaches AWS from CI only through GitHub OIDC and keeps every secret out of the repository and the state. The landing zone (A08) places humans who operate AWS in IAM Identity Center, forbids IAM users, keeps devices in AWS IoT Core and leaves the application identity directory to this task; its section 7 designs the audit, which is referenced here and not redesigned. The connectivity design (A10) fixes TLS 1.2 or later end to end, encrypted device buffers and app outboxes, and short-lived pre-signed URLs for drone uploads. The clarification questions (A06) record ASM-058: the lookup by sack code is unauthenticated and returns only the buyer view, and the read-only buyer identity is used by registered buyers in the web reports.

## 2. Kinds of identity

Every actor of the pilot enters through exactly one kind of identity, and the kinds never mix: a farmer or a buyer never receives an AWS identity, and a device never receives a human one.

| Kind | Who or what | Directory | Authentication | Designed in |
|---|---|---|---|---|
| Application users | Farmer, transporter, collection-center operator, read-only buyer, drone operator, agronomist, platform operator (application side) | Amazon Cognito user pools, one `field` pool and one `staff` pool per environment, in the workload account (ASM-090) | Section 5 | This document |
| Anonymous public reader | Anyone holding a sack code | None | None; returns only the buyer view (ASM-058) | This document, A16 |
| Humans who operate AWS | Platform operator, auditor, break-glass | IAM Identity Center, organization instance | Identity Center MFA on every sign-in | A08 section 4; permission sets in section 5.3 |
| Field devices | Each farm node (two per farm, ASM-070) and each drone kit (ASM-093) | AWS IoT Core registry, in each environment account | X.509 client certificate registered from a CSR, mutual TLS | ADR-003; drones in section 6 |
| CI and CD pipeline | Plan and apply roles per environment | GitHub OIDC identity provider | OIDC token exchanged for short-lived role credentials | ADR-003 |
| Platform services | Functions, topic rules, the application API | IAM roles in the workload account | AWS service role credentials | Least-privilege roles from the security stack (ADR-003, B06) |

No identity holds a long-term secret that a person can copy: there are no IAM users or access keys (A08 section 3.1), no shared application accounts, and device private keys never leave the device (ADR-003).

## 3. Application identity directory

### 3.1 Choice

The application identity directory is **Amazon Cognito user pools**, on the **Essentials** feature plan, in `us-east-1` (ASM-090). Each environment has two user pools in its workload account: a `staff` pool with MFA set to required, and a `field` pool without MFA. Device identities stay in AWS IoT Core and are never Cognito users.

Cognito is chosen because it is the managed AWS directory built for application end users, it lives inside the landing zone and the `us-east-1` Region of ADR-001, it issues standard OAuth 2.0 and OpenID Connect tokens that any API front door chosen by A16 can validate, and it provides each feature the roles need without a server to run: TOTP software-token MFA, passwordless one-time codes by SMS and email (Essentials), groups, Lambda triggers, refresh-token revocation and administrative disable and sign-out operations used by revocation (section 10). It is priced per monthly active user, which suits a pilot of a few hundred users; the price is left to A23 under PD-017.

MFA in a Cognito user pool is a setting of the whole pool, not of a group. Two pools make the case statement's split enforceable by configuration: a staff user cannot sign in without a second factor because their pool refuses it, not because a rule in the application remembers to check. A misconfiguration of the field pool cannot remove MFA from a staff role.

| Alternative | Reason rejected |
|---|---|
| IAM Identity Center for application users | It is the workforce entry point to AWS accounts (A08); farmers and buyers must never receive an AWS identity. |
| One Cognito user pool with MFA optional and an application check per role | MFA for the three mandated roles would depend on application code instead of the pool configuration. |
| One Cognito user pool per role (seven pools) | Seven sets of clients, triggers and runbooks for the same two sign-in policies; no added isolation. |
| Self-hosted directory (for example Keycloak) | Servers, patching and availability to operate, against the no-persistent-compute direction of Part E and the 100 hours. |
| A third-party identity SaaS | Personal data would be held by another processor outside the landing zone, adding a transfer to justify under ASM-002. |

### 3.2 Pools, groups and app clients

| Pool | MFA | Groups (one per role) | Sign-in | App clients |
|---|---|---|---|---|
| `field` | Off | `farmer`, `transporter`, `buyer` | Passwordless one-time code: by SMS to the registered phone for farmers and transporters, by email for buyers (section 5.2) | Farmer app, transporter app (public clients, no secret); reports web for buyers |
| `staff` | Required, TOTP software token | `collection_center_operator`, `drone_operator`, `agronomist`, `platform_operator` | Password plus TOTP from an authenticator app (section 5.1) | Logistics app of the collection center, drone ground-station app, reports web |

Every app client has token revocation enabled and uses the authorization code flow with PKCE (web) or the Cognito user authentication flow from the SDK (mobile). A Cognito pre-authentication Lambda trigger refuses the sign-in of any user who belongs to more than one role group, so one person who holds two roles holds two separate identities (ASM-099). Token lifetimes are in ASM-092.

Cognito holds only what sign-in needs: the immutable `sub`, the phone number or email used for the one-time code or recovery, and the group. Names, national ID numbers (cédula), vehicle plates and every other personal field live in the application's personal-data store under its own customer-managed key (section 7.2), because the user pool cannot be encrypted with a customer-managed key.

### 3.3 How authorization is enforced

Tokens prove who the caller is and which role group they belong to. They do not carry scopes such as farm or trip lists. Every API operation denies by default and allows only when the caller's role permits the action on that data object (section 4.2) and the target is inside the caller's scope, which the API reads from assignment records in the transactional store:

| Scope | Assignment record | Maintained by |
|---|---|---|
| Own farms | Farm to farmer membership, written at enrollment. | Platform operator |
| Own trips | Trip to transporter, created by the transporter app when custody is accepted (A10 section 7). | Transporter app, checked by the API |
| Own center | Collection-center operator to center (`cauca`, `huila` or `tolima`). | Platform operator |
| Assigned missions | Fumigation or survey order to pilot and drone kit, set when the order is authorized or scheduled (ADR-002). | Agronomist |
| Department | Agronomist to one or more departments (ASM-021). | Platform operator |

Because scope lives in data rather than in tokens, removing an assignment takes effect on the next request, not at token expiry. The API also rejects any token whose `sub` has a `revoked_after` timestamp later than the token's `auth_time`, and any request from an app installation marked revoked; these two checks make revocation immediate without depending on token expiry (section 10).

## 4. Roles and least privilege

### 4.1 Role catalog

| Role | Who | Application | Scope | MFA | Pool |
|---|---|---|---|---|---|
| Farmer (caficultor) | Farmer of an enrolled farm | Farmer app (few screens) | Own farms | No (section 5.2) | `field` |
| Transporter (transportador) | Driver who carries sacks | Transporter app | Own trips | No (section 5.2) | `field` |
| Collection-center operator (operario de acopio) | Logistics staff of one center | Logistics app; reports web | Own center | Yes | `staff` |
| Read-only buyer (comprador de solo lectura) | Registered buyer | Reports web | Sacks by code, buyer view only | No | `field` |
| Anonymous public reader | Anyone with a sack code | Public lookup page | One sack by code, buyer view only | Not authenticated | None |
| Drone operator (operador de drones) | Certified pilot of the contracted operator (ASM-023) | Drone ground-station app | Assigned missions | Yes | `staff` |
| Agronomist (ADR-002) | Federation agronomist (ASM-021) | Reports web | Assigned departments | Yes | `staff` |
| Platform operator (operador de plataforma) | Federation platform team | Reports web (operations screens); AWS through IAM Identity Center (section 5.3) | Whole platform, no business writes | Yes | `staff` |

### 4.2 Matrix of role, data object and action

Actions: **R** read, **C** create, **U** update, **A** authorize or reject, **-** no access. The scope in brackets is the one of section 3.3; anything outside it is denied. Writes by devices and functions are not human roles and are shown in the last column.

| Data object | Farmer | Transporter | Center operator | Buyer and public | Drone operator | Agronomist | Platform operator | Written by devices or functions |
|---|---|---|---|---|---|---|---|---|
| Farm record, zones, plots, polygons, no-fly zones | R (own farms) | R pickup point only (own trips) | R farm name and municipality (farms delivering to own center) | - | R polygon and no-fly zones (assigned missions) | R; C and U plots and preliminary no-fly zones (department, ASM-027) | C and U farm record at enrollment | Delimitation flight results |
| Farm node registry | R status (own farms) | - | - | - | - | R (department) | C, U, revoke | - |
| Crop and drying telemetry | R (own farms) | - | - | - | - | R (department) | R operational metadata only: counts, last seen | Farm nodes, through the IoT topic rule |
| Alerts and drying closure notices | R (own farms) | - | - | - | - | R (department) | R operational metadata only | Threshold function |
| Thresholds (alert ranges) | R ranges applied to own farms | - | - | - | - | R | U, with step-up (ASM-099) | - |
| Sack registration (code, weight, farm) | C, R (own farms) | R code and weight (own trips) | R (own center); C code binding for a sack that arrives without code | - | - | - | - | - |
| Sack custody events (full trace) | R own sacks, without transporter positions or personal data | C, R (own trips), including port delivery (ASM-057) | C reception accepted or rejected and departure; R (own center) | - | - | - | - | - |
| Buyer view of a sack | R | R | R | R by code only | R | R | R | Buyer-view projection (section 8.4) |
| Trip and transporter positions | - | C, R (own trips) | R trips to or from own center, while open | - | - | - | - | - |
| Code sheets and label batches | R codes allocated to own farms | - | C, R (own center) | - | - | - | - | - |
| Harvest estimate (sacks expected) | R (own farms) | - | R (farms delivering to own center) | - | - | R (department) | - | Estimation function |
| Pest case and inference results | R case status (own farms) | - | - | - | - | R; U close a case without confirmed focus (department) | - | Confirmation and verification functions |
| Fumigation order | R window and status (own farms) | - | - | - | R (assigned missions) | A authorize or reject, sets window and pilot (department) | - | Order state changes from flight records |
| Mission package | - | - | - | - | R (assigned missions) | R (department) | - | - |
| Flight record and drone objects | - | - | - | - | C through upload grants (assigned missions) | R (department) | - | Reception and inference functions |
| Personal profile (name, cédula, phone, plate) | R own | R own | R display name and plate of transporters of trips to own center | R and U own (buyer) | R own | R own | C, U, R for enrollment and support | - |
| Application identities and assignments | - | - | - | - | - | C and U pilot assignment on an order | C, U, disable, sign out, group and scope assignment | - |
| Application audit records | - | - | - | - | - | - | R log groups (A08 section 7.3) | Every audited service |

Nobody reads another person's cédula except the platform operator during enrollment and support; it is not shown to the collection center, the transporter, the agronomist, the drone operator or the buyer. Deletion is not an action of any human role in the matrix: data leaves only through the retention process of section 9.

### 4.3 Separation of duties

| Rule | Enforcement |
|---|---|
| The agronomist who authorizes a fumigation does not fly it, and no service identity authorizes (ADR-002). | Only a `staff` token of group `agronomist` can call the authorize operation; the API rejects it when the chosen pilot's `sub` equals the authorizer's; `authorizer_id` is never writable by a function role. |
| A drone operator cannot create or authorize an order. | No create or authorize operation is allowed to group `drone_operator`. |
| The platform operator cannot authorize a fumigation and does not write sacks, custody events or orders. | No business write is allowed to group `platform_operator` (ADR-002, "it cannot authorize a fumigation"). |
| One identity, one role. | Pre-authentication trigger of section 3.2 (ASM-099). |
| Sensitive staff operations need a fresh second factor. | Fumigation authorization, threshold change, revocation and identity administration require a token whose `auth_time` is at most 10 minutes old (ASM-092); otherwise the app asks for the TOTP code again. |

## 5. MFA and sign-in

### 5.1 Roles with MFA

MFA is required for the collection-center operator, the drone operator and the platform operator, as the case statement requires, and for the agronomist, as ADR-002 requires. All four live in the `staff` pool, where MFA is set to required. The second factor is a TOTP software token from an authenticator app on the person's phone. TOTP works without a data connection, which matters for a drone operator preparing a mission at a municipal seat, and avoids SMS as a second factor, which can be taken over by a SIM swap. A lost TOTP phone is handled in section 10.

### 5.2 Farmer and transporter: no MFA, as a justified decision

The case statement requires MFA for three roles and not for the farmer or the transporter. This design keeps them without MFA on purpose (ASM-091), for three reasons from the case statement. Part of the farmers distrust leaving the notebook and the farmer app has few screens ("Parte de los caficultores desconfía de dejar el cuaderno. La aplicación del caficultor tiene pocas pantallas"), so every extra sign-in step costs adoption, and an app that is not used leaves the sack registration on paper. Farmers and transporters have data only at the municipal seat or on main roads (case statement, assumptions), so a factor that must be renewed online would lock them out of an app that has to keep recording offline (A10, outboxes). And their scope is small: a farmer sees and writes only their own farms and a transporter only their own trips (section 4.2), so a stolen session cannot reach other people's data, and nothing they write authorizes a fumigation or changes a threshold.

Instead of MFA they use the following controls:

| Control | Design |
|---|---|
| No password | Passwordless one-time code by SMS to the phone number registered at enrollment; nothing to remember or write in the notebook. |
| Long-lived session on one phone | Refresh token valid 30 days, access token 60 minutes (ASM-092), so the app keeps working offline and signs in again only at the next connection after expiry. |
| App bound to the installation | Each event carries the app installation identifier (`device_id`, A10 section 7); a revoked installation is rejected immediately (section 10). |
| Local unlock | The app opens only after the phone's screen lock or biometric unlock, and its outbox is encrypted with a key in the phone's hardware-backed keystore (ASM-095). |
| Server-side checks | The API checks every event against the caller's own farms or trips; the outbox cannot inject events for another farm or trip. |

The read-only buyer also has no MFA, because the case statement does not require it and a buyer reads only the buyer view, which the public lookup returns without authentication anyway (ASM-058). Buyers sign in with a one-time code by email.

### 5.3 Humans who operate AWS

People who operate AWS enter through IAM Identity Center with MFA required on every sign-in (A08 section 4). The permission sets below are the narrow operations sets that the production-change SCP of A08 section 3.1 exempts.

| Permission set | Holder | Accounts | Allows | Session |
|---|---|---|---|---|
| `PlatformReadOnly` | Platform operators | Workload accounts, network account | Read-only on resources, CloudWatch metrics and logs, `AWSIotLogsV2`, application audit log groups. | 4 hours |
| `DeviceRevocation` | Platform operators | Workload accounts | `iot:UpdateCertificate`, `iot:DetachPolicy`, `iot:DetachThingPrincipal`, `iot:UpdateThing`, `iot:Describe*`, `iot:List*` on things and certificates. Fallback path of section 10 when the operations screen is unavailable. | 1 hour |
| `IdentitySupport` | Platform operators | Workload accounts | `cognito-idp:AdminDisableUser`, `AdminEnableUser`, `AdminUserGlobalSignOut`, `AdminAddUserToGroup`, `AdminRemoveUserFromGroup`, `AdminGetUser`, `AdminSetUserMFAPreference` on the environment's two pools. Fallback path of section 10. | 1 hour |
| `Auditor` | A Federation person outside the platform team, named at deployment (ASM-099) | Security account | Read the audit archive through Amazon Athena and the organization trail (A08 section 7.3). | 4 hours |
| `BreakGlass` | Two named Federation people, used only under an incident reference | All accounts | Administrator access; every use alerts the platform operators and the auditor. | 1 hour |

Platform operators change production infrastructure only through the pipeline of ADR-003, never through a permission set. The pipeline identities themselves are the GitHub OIDC roles of ADR-003 and are not repeated here.

## 6. Device identities

Farm nodes and drones are devices, so they live in AWS IoT Core: one thing per device and one X.509 certificate registered from a CSR generated on the device, so the private key is never seen by Terraform, the pipeline or a person (ADR-003, ASM-010). The policy of each device is written with policy variables, so one policy serves 200 or 1,000 farms and no policy names a device.

| Device | Thing | Key held in | IoT policy (summary) | What it reaches |
|---|---|---|---|---|
| Farm node, crop or drying (ASM-070) | One per node, named from its stable node identifier | Node non-volatile secure storage | `iot:Connect` only with client ID equal to `${iot:Connection.Thing.ThingName}` and `iot:Connection.Thing.IsAttached` true; `iot:Publish` only to its own telemetry topic, built from the same variable (ADR-003). No subscribe. | IoT Core data endpoint, MQTT over TLS (A10 section 4.2) |
| Drone kit: one drone and its paired ground station (ASM-093) | One per kit | Ground station secure storage | `iot:Connect` with the same client ID rule; `iot:Subscribe` and `iot:Receive` only on its own grant topic (for example `drones/${iot:Connection.Thing.ThingName}/grants`); `iot:Publish` only on its own status topic. | IoT Core data endpoint while the ground station has a link |

**Drone uploads bind the pilot and the drone.** A10 fixes that the drone operator downloads the mission from the application API under the pilot identity with MFA and uploads flight records and objects to Amazon S3 with short-lived pre-signed URLs for the order's own prefix. This design adds the device to that path, so that both a revoked pilot and a revoked drone stop the upload:

1. The pilot, signed in to the `staff` pool, asks the API for an upload grant for an order and a drone kit.
2. The API checks that the pilot is assigned to the order, that the pilot's permit reference is valid (ASM-020), that the drone kit is assigned to the order, and that the kit's certificate is `ACTIVE` in the IoT registry.
3. The API writes an upload-grant record in the transactional store with the pilot `sub`, the kit's thing name, the order, the object keys and the expiry, and publishes the pre-signed URLs, valid 15 minutes (ASM-093), on the kit's grant topic. Only a connection authenticated with that kit's certificate can subscribe to it.
4. The ground station uploads over TLS with the URLs, as A10 designs; the reception function accepts an object only if its key matches an unexpired grant whose pilot and kit were both active when the object arrived, and otherwise moves it to a quarantine prefix without linking it to the order.

The grant record is the application record that carries the individual pilot into the audit of flight object delivery, next to the CloudTrail S3 data event of A08 section 7.2. This closes PD-075.

## 7. Encryption

### 7.1 In transit

Every flow uses TLS 1.2 or later from the device or app to the AWS endpoint, and no gateway, relay or third party terminates TLS (A10 section 2). The mechanisms per endpoint are:

| Endpoint | Used by | Mechanism |
|---|---|---|
| AWS IoT Core data endpoint | Farm nodes, drone kits | MQTT over TLS with mutual X.509 authentication; the domain configuration uses a TLS security policy with TLS 1.2 as minimum, for example `IoTSecurityPolicy_TLS13_1_2_2022_10` (PD-100). |
| Application API and public lookup front door | Apps, reports web, public lookup | HTTPS only; minimum TLS 1.2 policy on the front door chosen by A16 (PD-051), for example the `TLS_1_2` security policy of an Amazon API Gateway custom domain or `TLSv1.2_2021` on Amazon CloudFront. HTTP is redirected or refused. |
| Amazon S3 (drone objects, exports, audit archive) | Drone ground stations, functions | Bucket policies deny any request with `aws:SecureTransport` false and with `s3:TlsVersion` below 1.2; uploads carry per-part checksums (A10 section 6). |
| Amazon Cognito endpoints | Apps, reports web | AWS-managed HTTPS endpoints, TLS 1.2 or later. |
| Collection-center WAN | Logistics posts, ground stations at a center | IPsec site-to-site VPN to the network-account hub (ASM-078), carrying traffic that is itself TLS to interface endpoints. |
| Service to service inside AWS | Functions, rules, stores | AWS SDK calls over TLS; VPC access to AWS services through interface endpoints (ADR-003, PC-IAC-020). |

Data on the edge is also encrypted: the farm node buffer (ASM-074), the app outboxes (ASM-079) with a key in the phone keystore, and the storage of the drone ground station, which the contracted operator encrypts as a contract requirement (ASM-095).

### 7.2 At rest, per store

Every store uses an AWS KMS customer-managed key of its own data class, so that each service role is granted only the key of the data it needs; for example the inference functions can decrypt drone objects but not personal data.

| Store | Holds | Mechanism | Key |
|---|---|---|---|
| Telemetry table (Amazon DynamoDB) | Crop and drying readings | DynamoDB encryption at rest with a customer-managed key | `telemetry` |
| Position store | Transporter positions | Same, in the store Part E allows for position (A13) | `telemetry` |
| Alert topic and dead-letter queues (Amazon SNS, Amazon SQS) | Alerts and failed messages | Server-side encryption with a customer-managed key | Key of the data they carry |
| Drone object bucket (Amazon S3) | Photos, maps, polygons, flight records; sack history export prefix (ADR-001) | Default encryption SSE-KMS with S3 Bucket Keys, Block Public Access, bucket-owner-enforced object ownership, versioning | `objects` |
| Transactional tables (DynamoDB) | Sacks, custody events, trips, cases, orders, upload grants, assignments, device registry | DynamoDB encryption with a customer-managed key | `records` |
| Personal-data table (DynamoDB) | Name, cédula, phone, vehicle plate, linked by a pseudonymous `subject_id` | DynamoDB encryption with a customer-managed key | `personal` |
| Cognito user pools | `sub`, phone or email, group, MFA secret | Encrypted at rest by Amazon Cognito with AWS-managed keys; customer-managed keys are not offered, so only sign-in data is stored there (section 3.2) | AWS managed |
| CloudWatch Logs groups | Application audit, function logs, `AWSIotLogsV2` | Log group encryption with a customer-managed key | `logs` |
| Terraform state bucket | State per environment | SSE-KMS (ADR-003) | `state` |
| Audit archive (security account) | Trail and application audit records | SSE-KMS, Object Lock (A08 section 7.1) | `audit` (A08) |

A pre-signed URL uploads with the permissions of the role that signed it, so the API role that issues upload grants holds `kms:GenerateDataKey` on the `objects` key and `s3:PutObject` on the order prefixes only.

### 7.3 Key ownership and rotation

| Account | Keys | Administered by | Used by |
|---|---|---|---|
| Production, test, development (one set each) | `telemetry`, `objects`, `records`, `personal`, `logs`, `state` | The security stack apply role of that environment (ADR-003, B06) and `BreakGlass` | Named service roles of that environment only; no key is shared across environments, so a test role can never decrypt production data |
| Security | `audit` | Security account administration (A08) | CloudTrail, Firehose delivery, `Auditor` read |
| Management | Budget notification topic key | Management account (A08 section 6) | AWS Budgets, SNS |
| Network | None; it holds no workload data (A08) | - | - |

Every customer-managed key has automatic rotation enabled with a rotation period of 365 days; KMS keeps the earlier key material, so data never needs re-encryption (ASM-097). Key policies name their principals explicitly, never a wildcard, and deny `kms:ScheduleKeyDeletion` and `kms:DisableKey` to every principal except `BreakGlass`; a scheduled deletion uses the maximum waiting period of 30 days, which alerts the platform operators. Keys are created by the B06 security module, not by hand.

## 8. Sack code and buyer view

### 8.1 Sack code format

The sack code is the only identifier a buyer or the public sees. It is a random opaque value that contains no information and cannot be derived from personal data (ASM-094):

| Property | Design |
|---|---|
| Content | 80 random bits from a cryptographically secure generator. |
| Encoding | 16 symbols of Crockford Base32 (digits and capital letters without I, L, O and U, so it can be read aloud and typed), plus one check symbol that catches a mistyped character, printed as `XXXX-XXXX-XXXX-XXXX-C`. |
| Printed form | Human-readable code plus a QR code that encodes the public lookup URL with the code. |
| Issued by | The platform only, in label batches allocated to a farm and printed at its collection center, and on the code sheets printed at the center for sacks that arrive without a code (case statement, Part D). A farm registering offline uses a pre-printed label, so no code is generated on the phone. |
| Never derived from | The cédula, a name, a farm identifier, a node identifier, a date, a sequence number, a department or center, or any hash of these; a hash of a cédula could be reversed by trying every number. |
| Internal identity | The platform keys the sack by an internal `sack_id`; the code is a separate, unique index, so a damaged label can be replaced without changing the sack's history. |
| Guessing | 2^80 possible codes against a peak of 30 reads per minute; the public lookup front door also rate-limits by source (PD-102). A wrong or unknown code returns the same "not found" answer. |

### 8.2 Fields the buyer view returns

The buyer view returns exactly the four groups of the case statement ("origen, fechas de proceso, estado y variables de secado"), with the granularity of ASM-096:

| Group | Fields |
|---|---|
| Origin | `sack_code`, `department`, `municipality` |
| Processing dates | `drying_start_date`, `drying_end_date`, `packing_date` (calendar dates, no time of day) |
| Status | `status`: the current sack state of Part C only (in preparation, en route to the center, at the center accepted or rejected, en route to the port, delivered at the port accepted or rejected) |
| Drying variables | `final_coffee_moisture_pct`, `grain_temperature_min_c`, `grain_temperature_mean_c`, `grain_temperature_max_c` for the drying lot of the sack |

### 8.3 Fields the buyer view never returns

| Never returned | Why |
|---|---|
| Farmer name, cédula, phone, `subject_id` | Personal data; point 2 forbids exposing the cédula. |
| Farm name, farm identifier, vereda, coordinates, plot or polygon | Would identify the farmer or locate the farm. |
| Node identifiers and raw telemetry series | Device identities and full readings are not drying summary variables. |
| Custody history: event times, transporter identity, trip, positions, vehicle plate, receiving operator, receiver reference | That is the full trace, which point 2 withholds from the buyer. |
| Collection center, rejection reasons, weights | Not in the four groups of point 2. |
| Pest cases, fumigation orders, product batch, flight objects, inference results | Not in the four groups of point 2. |
| Internal `sack_id`, audit data | Internal identifiers. |

### 8.4 How the view is enforced

The buyer view is a separate read model: a projection, written by the platform when a sack changes state or its drying lot closes, that contains only the fields of section 8.2. The public lookup and the buyer role read that projection only, with a service role that has no permission on the transactional tables, the personal-data table or their keys. A coding error in the lookup therefore cannot return a field that the projection does not hold. A16 designs the projection and the lookup front door (PD-102); the audit of each read is A08 section 7.2.

## 9. Personal data and retention

### 9.1 Inventory

| Person | Personal data | Where |
|---|---|---|
| Farmer | Name, cédula, phone, farms | Personal-data table; phone also in the `field` pool |
| Transporter | Name, cédula, phone, vehicle plate; positions while en route | Personal-data table; phone in the `field` pool; positions in the position store |
| Staff and buyers | Name, email, phone, organization | Personal-data table; email in the user pool |
| Any public reader | Source IP of a sack read | Audit records (ASM-069) |

Every business record (sack events, trips, orders, flight records) references a person only by the pseudonymous `subject_id`, a random identifier that is not derived from personal data. Application logs carry `subject_id` or `sub`, never a name, cédula or phone.

### 9.2 Retention and what happens after it

The case statement fixes 5 years for personal data of farmers and transporters and for readings, photos, flights and sack history. This design counts the 5 years for the personal profile from the person's deactivation in the pilot, and for every event record from the event date (ASM-098). The data is stored in `us-east-1`, outside Colombia (ASM-002); whether the Federation accepts that transfer, and on which legal basis, is question Q-02 of the clarification session and PD-101.

| Data | At the end of retention | Mechanism |
|---|---|---|
| Personal profile | Anonymized: the profile item (name, cédula, phone, plate) is deleted and the Cognito user is deleted; the `subject_id` in the business records stays, now linked to nobody. | A monthly anonymization function started by Amazon EventBridge Scheduler, with its own role (delete on the personal-data table, `cognito-idp:AdminDeleteUser`). |
| Event records and positions | Deleted. | DynamoDB time to live on an `expires_at` attribute set at write time. |
| Drone objects and sack history export | Deleted. | S3 lifecycle expiration of current and noncurrent versions after 5 years. |
| Audit records with source IP | Deleted when Object Lock retention ends. | S3 lifecycle on the audit archive (A08 section 7.3). |
| Backups | Overwritten. | DynamoDB point-in-time recovery keeps at most 35 days, so a deletion is complete in every copy within 35 days. |

A drone object that contains a person is not sent to inference; its treatment is designed by A17 under PD-015.

## 10. Revocation

The case statement requires that "se revoca la identidad y el dispositivo deja de publicar". The table covers the two cases it names and the two human cases that follow from section 5. The primary path is the operations screen of the reports web, used by a platform operator with a fresh TOTP (section 4.3), which invokes a revocation function whose role holds only the actions listed; the fallback path is the same AWS actions through the `DeviceRevocation` or `IdentitySupport` permission set (section 5.3). The target is to complete a revocation within 4 hours of the report (ASM-098); it is a target, not a measurement.

| Case | Who triggers | Exact AWS and platform actions | How it stops publishing |
|---|---|---|---|
| Farm node decommissioned (farm leaves the pilot, node replaced, node stolen) | The Federation installer or the farmer, through the collection center, reports it; a platform operator executes. | 1. `iot:UpdateCertificate` with new status `REVOKED`. 2. `iot:DetachPolicy` from the certificate. 3. `iot:DetachThingPrincipal` from the thing. 4. `iot:UpdateThing` sets the attribute `status=decommissioned`, so the silent-node alarm of point 6 ignores it. The certificate is kept, revoked, for the audit trail. | Within minutes of leaving `ACTIVE`, IoT Core disconnects sessions that used the certificate and refuses any new TLS session with it (PD-100). Even on an open session, the detached policy denies every publish. The node's buffer can no longer drain, and its readings expire after 24 hours on the device (ASM-074). A replacement node gets a new CSR-registered certificate. |
| Drone lost | The contracted drone operator company reports it (ASM-098); a platform operator executes. | The same four IoT actions on the drone kit's thing, with `status=lost`. The API refuses mission packages and upload grants for the kit. Orders already dispatched to it keep their state until their window expires and need a new authorization (ASM-029). | The kit cannot connect, so it receives no upload grant; URLs already issued expire within 15 minutes; any object that arrives referencing the kit after revocation is quarantined and never linked to an order (section 6). |
| Drone pilot leaves the contract | The contracted operator company reports it; a platform operator executes. For an agronomist who leaves, the Federation reports it. | `cognito-idp:AdminDisableUser` (the user can no longer sign in), `AdminUserGlobalSignOut` (refresh tokens revoked), `AdminRemoveUserFromGroup` from `drone_operator`; the application sets `revoked_after` and removes the pilot from pending orders, which need a new authorization naming another pilot. The user is not deleted, so the orders it flew keep a valid `operator_id`. | Every API call is refused at once through `revoked_after`, without waiting for token expiry; no upload grant can be issued in the pilot's name. The drone kit stays active for other pilots. |
| Phone of a farmer or transporter lost | The user reports it to the collection center or the Federation support line; a platform operator executes. | `AdminUserGlobalSignOut`; the application marks the app installation `device_id` as revoked; `AdminDisableUser` until the user re-enrolls in person, because whoever holds the phone also holds the SIM that receives the one-time code. On re-enrollment, `AdminEnableUser`, update of the phone number if it changed, and sign-in on the new phone. | Any request from the revoked installation is refused, and the refresh token no longer works. Events left in the lost phone's outbox never enter the platform; a custody step that was not uploaded is recorded again by the transporter or the center. A staff member's lost TOTP phone follows the same steps plus `AdminSetUserMFAPreference` to reset the software token, which is associated again at the next sign-in. |

Each action is a management API call recorded by the organization trail, and the operations screen also writes an application audit record with the requester, the reason and the incident reference (A08 section 7).

## 11. Auditing

The audit of the four mandatory events (sack read, threshold change, telemetry publication, flight object delivery) is designed in [A08 section 7](landing-zone.md#7-audit-of-the-four-mandatory-events) and not redesigned here. This document supplies three inputs to it: the identity fields it records come from the identities of section 2 (Cognito `sub` and group, IoT thing name and certificate, `anonymous` for public reads); the individual pilot of a flight object delivery comes from the upload-grant record of section 6 (PD-075); and the holder of the auditor role is defined in section 5.3.

## 12. Compliance with the case statement and the issue

| Requirement | Where it is met |
|---|---|
| Six least-privilege roles: farmer, transporter, collection-center operator, read-only buyer, drone operator, platform operator | Sections 4.1 and 4.2, plus the agronomist of ADR-002 and the anonymous reader of ASM-058. |
| MFA for collection-center operator, drone operator and platform operator | Section 5.1 (`staff` pool, MFA required), and section 5.3 for AWS access; the agronomist too. |
| What farmer and transporter use instead of MFA, and why | Section 5.2. |
| Application identity directory chosen and justified | Section 3 (Amazon Cognito user pools, Essentials). |
| Encryption in transit and at rest for telemetry and drone objects, and for personal data | Section 7: per-endpoint TLS, per-store mechanism, keys per environment, ownership and rotation. |
| The buyer view shows only origin, processing dates, status and drying variables | Section 8.2 and 8.3, enforced by the projection of section 8.4. |
| The sack identifier does not expose the cédula nor the full trace | Section 8.1. |
| Personal data, 5-year retention, deletion or anonymization | Section 9. |
| Revocation of a decommissioned node and a lost drone, the device stops publishing; departing pilot; lost phone | Section 10. |
| Auditing referenced, not redesigned | Section 11. |

## 13. Decision-log entries

| ID | Summary |
|---|---|
| PD-075 | Pilot identity carried into flight object uploads. Closed by section 6 (upload-grant record). |
| ASM-090 | Application directory: Amazon Cognito user pools, Essentials plan, `staff` and `field` pools per environment in the workload account. |
| ASM-091 | Farmer, transporter and buyer sign in without MFA, by passwordless one-time code. |
| ASM-092 | Token lifetimes and the 10-minute step-up for sensitive staff operations. |
| ASM-093 | Drone kit identity in IoT Core; upload grants on the kit topic; 15-minute pre-signed URLs. |
| ASM-094 | Sack code format and issuance. |
| ASM-095 | Edge encryption: outbox key in the phone keystore; ground-station storage encrypted by the operator. |
| ASM-096 | Granularity of the buyer view fields. |
| ASM-097 | One customer-managed key per data class and environment, rotated every 365 days. |
| ASM-098 | Retention clock, reporting duty for lost devices and departing users, 4-hour revocation target. |
| ASM-099 | One role per identity; thresholds written by the platform operator; enrollment by the platform operator; auditor holder. |
| PD-100 | Re-check the AWS features named here against live documentation. |
| PD-101 | Legal basis for personal data transfer and retention, with Q-02. |
| PD-102 | A16 implements the sack code, the buyer-view projection and the lookup rate limit. |
| PD-103 | B06 codes the keys, key policies, rotation and the Identity Center permission sets of this document within its scope. |
| PD-104 | A15 adds the drone kit assignment and the upload grant to the drone service. |

## 14. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Point 2, Part C, Part E, restrictions and assumptions. |
| [`docs/decision-log.md`](decision-log.md) | ASM-001, ASM-002, ASM-010, ASM-020, ASM-021, ASM-023, ASM-027, ASM-029, ASM-057, ASM-058, ASM-069, ASM-070, ASM-074, ASM-078, ASM-079, PD-015, PD-017, PD-051, and the entries of section 13. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | Agronomist identity, separation of duties, order fields and states. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | X.509 from CSR, revocation by certificate status, OIDC, no secrets in state, CMK encryption. |
| [`docs/landing-zone.md`](landing-zone.md) | Identity entry points (section 4), SCPs (section 3.1), audit (section 7). |
| [`docs/connectivity.md`](connectivity.md) | TLS end to end, outboxes, drone pre-signed URLs, transporter events. |
| [`docs/clarification-questions.md`](clarification-questions.md) | Q-02 (residency) and Q-13 (buyer access). |
| [Amazon Cognito user pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html), [feature plans](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sign-in-feature-plans.html), [MFA](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-mfa.html) | Directory, Essentials plan, passwordless sign-in, TOTP. |
| [AWS IoT Core `UpdateCertificate`](https://docs.aws.amazon.com/iot/latest/apireference/API_UpdateCertificate.html) and [policy variables](https://docs.aws.amazon.com/iot/latest/developerguide/thing-policy-variables.html) | Revocation and per-thing policies. |
| [AWS KMS key rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html) | Automatic rotation period. |
