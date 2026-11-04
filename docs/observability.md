# Observability and SLOs of the Aroma Pacífico Pilot

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR2 / JR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#34 (A21) |
| Builds on | [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md), [architecture](architecture.md#8-point-6-observability) section 8, [landing zone](landing-zone.md), [connectivity](connectivity.md), [collection-center network](collection-center-network.md), [firewall policy](firewall-policy.md), [farm monitoring](farm-monitoring.md), [drone service](drone-service.md), [sack and trip](sack-and-trip.md), [inference functions](inference-functions.md), [automation and pipelines](automation-and-pipelines.md) |
| Dashboards | [`dashboards/README.md`](dashboards/README.md), [`dashboards/operational.json`](dashboards/operational.json), [`dashboards/cost.json`](dashboards/cost.json) |
| Decision-log entries | PD-077, PD-082, PD-094, PD-134, PD-141, PD-151, PD-162 and PD-173 (closed here), ASM-220 to ASM-229, PD-220 to PD-223 |

## 1. Purpose and requirements

This document designs point 6 of the [case statement](case/case-statement.pdf): the metrics, logs and traces of the pilot, the five detections the case statement names, the two SLOs with their SLIs, and the operational and cost dashboards. The earlier designs already guarantee that every signal exists in a named place (the observability hooks of each design and [architecture](architecture.md#8-point-6-observability) section 8); this document sets the thresholds, windows, objectives and recipients on those signals and adds only what the hooks cannot provide by themselves: one scheduled detector function, one synthetic probe and one daily cost export. Nothing is deployed. The alarms and the two dashboard bodies are definitions that the IaC would apply; Point 6 is outside the 100 hours ("Observabilidad, SLO y las dos plantillas de release notes se entregan con el proyecto. No forman parte de las 100 horas"). Every threshold, window and objective is a design value, never a measurement; quotations are in the original Spanish, which prevails (ASM-001).

| Case statement text (point 6) | Where it is met |
|---|---|
| "Métricas, logs y trazas de la ingesta, de las aplicaciones, de la base, de los tres enlaces WAN y de la llegada de objetos de vuelo." | [Section 4](#4-metrics-logs-and-traces-by-domain) |
| "Nodos y fincas sin telemetría por más de 24 horas." | [Section 5.1](#51-nodes-and-farms-without-telemetry-for-more-than-24-hours) |
| "Alertas de cultivo no entregadas." | [Section 5.2](#52-crop-alerts-not-delivered-closes-pd-134) |
| "Vuelos sin registro persistido, fumigaciones sin cierre e invocaciones fallidas que no mutaron el estado." | [Sections 5.3 to 5.5](#53-flights-without-a-persisted-record-closes-pd-141-with-54) |
| "Dos SLO, cada uno con su SLI: entrega de la alerta de cultivo y disponibilidad de la consulta del saco." | [Sections 7](#7-slo-method-and-burn-rate-alerts), [8](#8-slo-1-crop-alert-delivery-closes-pd-082) and [9](#9-slo-2-sack-lookup-availability-closes-pd-151) |
| "Tablero operativo y tablero de costo, partidos por las etiquetas de fincas, drones, acopio y pruebas." | [Section 11](#11-dashboards-closes-pd-077) and [`dashboards/README.md`](dashboards/README.md) |

## 2. Services used

Every service below is already implied by the design: CloudWatch is the observability platform named by ADR-003 and the inventory of the architecture document, CloudTrail is the organization trail of the landing zone, and the AWS cost services are those of the landing zone budgets. No third-party product is used (ASM-220).

| Service | Used for | Why this and not another |
|---|---|---|
| Amazon CloudWatch metrics | AWS service metrics, and custom metrics emitted as CloudWatch embedded metric format (EMF) lines in the structured function logs | EMF turns the log line every function already writes (A17 section 7) into a metric without a second API call. |
| CloudWatch alarms and composite alarms | Every detection, the burn-rate alerts and the CodeDeploy rollback set | Composite alarms implement the two-window burn-rate rule without code. |
| CloudWatch Logs, metric filters and Logs Insights | Function, API access, WAF, IoT (`AWSIotLogsV2`), VPC Flow Logs, center firewall and CodeDeploy event logs; item-level drill-down of every detection | Log groups under the `logs` key with 365 days of retention already exist in the inventory; Logs Insights answers "which farm, which mission" without a second store. |
| AWS X-Ray, with the AWS Distro for OpenTelemetry (ADOT) Lambda layer for code instrumentation | Traces of the in-cloud hops: API Gateway, Lambda, SNS, SQS and EventBridge | X-Ray is the AWS-native trace backend; ADOT is the instrumentation the platform code would use when it exists. Hops outside AWS (the node, the radio, the phone) cannot carry a trace header, so business trace points on the records cover them (section 4.7). |
| CloudWatch Synthetics | One canary for the public sack lookup (section 9.4) | The lookup must be detected down within 15 minutes (ASM-157) even when nobody is reading a sack. |
| CloudWatch cross-account observability (Observability Access Manager) | The network account shares its VPN and Transit Gateway metrics with the `pdn` account | The tunnels live in the network account (A10 section 9); the operational dashboard lives in `pdn`. |
| AWS CloudTrail | Audit of the four mandatory events (A08 section 7), forensics, and change events on the observability resources themselves (section 6.6) | It is already the organization trail; observability reads it, it does not configure a second trail. |
| Amazon EventBridge and EventBridge Scheduler | CodeDeploy state-change events to a log group; the schedules of the detector function | Already used by the drone and summary designs. |
| AWS Billing metrics, AWS Cost Explorer, AWS Budgets, Cost Category | The cost dashboard (section 11.2) | The landing zone already defines the tags, the cost category with its `unallocated` value and the budgets (A08 sections 5 and 6). |

Two components are added (ASM-221). The **detector function** (`<prefix>-pdn-lambda-obsdetector`) is a Lambda function in each workload account, run by EventBridge Scheduler every 5 minutes for the alert-delivery checks and every hour for the state-age checks. It reads, with a read-only role, the node registry, the telemetry table, the alert store and the drone tables, through sparse secondary indexes on an `open_state` attribute set while a record is in a non-terminal state; these are additive attributes in the sense of ASM-175. It writes nothing to any data store: it emits EMF metrics and one structured log line per item it finds, and the alarms of section 5 read the metrics. The **cost export function** (`<prefix>-shared-lambda-costexport`) runs daily in the management account, reads Cost Explorer and Budgets, and publishes the cost metrics of section 11.2.

## 3. Conventions

### 3.1 Metrics and dimensions

Custom metrics live in namespaces `AromaPacifico/<Domain>`: `Telemetry`, `Alerts`, `Drone`, `Inference`, `Sack`, `Lookup`, `CenterNetwork`, `Detector` and `Cost`. Their dimensions are the tag dimensions of the case statement: `CollectionCenter` (`cauca`, `huila`, `tolima`, used for the department of a farm and for a center, as A08 section 5.1 maps them one to one), `Zone` (`crop`, `drying`), `function`, `MissionType`, `State` and, for cost, `CostLine` and `Driver`. `FarmId` is never a metric dimension: 200 farms, 1,000 at scale, times every metric would multiply the number of custom metrics for no alarm that needs it. The farm split is served by Logs Insights over the `farm_id` field that every structured line carries, and by the usage allocation of A08 section 5.3 for cost (ASM-220). The environment split, which is also the testing split ("pruebas"), is the account: every alarm and the operational dashboard exist in `dev`, `qa` and `pdn`, rendered from the same definitions.

### 3.2 Names

Names follow PC-IAC-003, `{client}-{project}-{environment}-{type}-{key}`, with `{client}-{project}` written `<prefix>` until PD-070 fixes it, and `pdn` shown. The names A17 and A18 proposed (`infthresholddlq`, `<function>-errors` and the others) are kept as written. In the dashboard JSON the same names appear as `${prefix}` and `${env}`, which the IaC renders with `templatefile`.

### 3.3 Recipients and severities

| Topic | Account | Subscriber | Severity and expected response |
|---|---|---|---|
| `<prefix>-pdn-sns-opspage` | `pdn` | Platform operator on call | Page: harm is happening now to alerts, sack events, the lookup or the ingestion; act at once, including in the harvest window under the incident path of ASM-183. |
| `<prefix>-pdn-sns-opsticket` | `pdn` | Platform operator | Ticket: reviewed in the daily operations review and at the start of every harvest-day shift. |
| `<prefix>-pdn-sns-agrocauca`, `agrohuila`, `agrotolima` | `pdn` | Agronomist of the department (ADR-002) | Field action: a farm silent, a flight or fumigation stuck, an order flagged. One topic per department, because a CloudWatch alarm notification carries no attribute a subscription filter could use. |
| `<prefix>-shared-sns-netpage` | Network | Platform operator on call | Page for a center without any tunnel. |
| `<prefix>-dev-sns-opsticket`, `<prefix>-qa-sns-opsticket` | `dev`, `qa` | Platform operator | Ticket only; nothing in `dev` or `qa` pages. |
| Budget topic of A08 section 6 | Management | Platform operator and cost owner | Cost alerts; unchanged. |

Subscriber addresses are chosen by the lead at deployment and are not written here, as in A08. Topics are encrypted with the `logs` customer-managed key, whose key policy lets CloudWatch alarms publish (ASM-222). The collection-center operator has no AWS identity; for a center link incident the platform operator informs the center. A CloudWatch alarm notifies on state change only, so a detection that stays in `ALARM` while new items join it does not notify again; the item list is read on the dashboard (section 11).

## 4. Metrics, logs and traces by domain

### 4.1 Ingestion: IoT Core, topic rule, telemetry table, threshold queue and function

| Kind | Signals |
|---|---|
| Metrics | `AWS/IoT` `PublishIn.Success`, `PublishIn.AuthError`, `Connect.AuthError`, `RuleMessageThrottled`, and `Success` and `Failure` per `RuleName` and `ActionType` (DynamoDBv2 and SQS actions); `AWS/SQS` depth and `ApproximateAgeOfOldestMessage` of the threshold queue, its dead-letter queue and the rule error queue; `AWS/Lambda` `Invocations`, `Errors`, `Throttles`, `Duration` and `ConcurrentExecutions` of the threshold function alias; custom `AromaPacifico/Telemetry` `ReadingsEvaluated` (by `Zone`, `CollectionCenter` and `delayed`), `ReadingsDelayed`, `IngestLagSeconds` (receive time minus reading time), `ReadingsNotEvaluated` by cause, `ReadingsEvicted` from gap reports (A10 section 4.3), `NodeClockAhead`, and `ReadingNotPersisted` (A17 section 7). |
| Logs | `AWSIotLogsV2` at level `ERROR` (A08 section 7.2); the threshold function's one structured line per reading (node, farm, zone, variable, outcome, `rule_key` and `version`, trace identifier); the rule error queue messages, inspected with the redrive console. |
| Traces | X-Ray from the SQS message that the rule writes to the threshold function and from there to the SNS alert topic. The node-side hops 1 to 3 have no trace header; they are covered by the business trace points of section 4.7. |

### 4.2 Alert path: alert topic, notification queue, dispatcher and channels

| Kind | Signals |
|---|---|
| Metrics | `AWS/SNS` `NumberOfMessagesPublished` and `NumberOfNotificationsFailed` of the alert topic and of the mobile push platform applications; depth, age and dead-letter queue of the notification queue; dispatcher alias `Errors`, `Throttles`, `Duration`; custom `AromaPacifico/Alerts` `AlertsPublished`, `SmsSent`, `PushSent`, `SmsDeliveryFailed` by carrier status class, `AlertsPendingDelivery`, `AlertsUndelivered`, and the SLO metrics of section 8. |
| Logs | Dispatcher lines (episode, farm, channels sent, provider message identifiers; never a phone number); the SMS delivery events of AWS End User Messaging SMS, delivered by its configuration set to a log group; app receipts recorded by the API. |
| Traces | X-Ray from the threshold function through SNS and SQS to the dispatcher and its two provider calls (hop 6). |

### 4.3 Applications and APIs: API Gateway, sack and trip functions, public lookup

| Kind | Signals |
|---|---|
| Metrics | `AWS/ApiGateway` `Count`, `4XXError`, `5XXError`, `Latency` and `IntegrationLatency` per stage; `AWS/Lambda` metrics of every API function alias (`sack-events`, trips and positions, mission package and upload grant, thresholds, reports queries), the `pre-auth` trigger, the `sack-applier` and `sack-exporter` (including `IteratorAge`) and the lookup function; `AWS/Cognito` sign-in and throttle metrics of the `staff` and `field` pools; `AWS/CloudFront` `Requests`, `4xxErrorRate`, `5xxErrorRate` and `OriginLatency` of the lookup and reports distributions; `AWS/WAFV2` `AllowedRequests` and `BlockedRequests` per rule; custom `AromaPacifico/Sack` `SackEventsAccepted`, `SackEventsRefused` by reason, `SackEventsPending`, `SackEventsAnomaly`, `SackEventsClockSkewed` (sack and trip section 3.3), `PositionsAccepted`, `PositionUploadDelay`; `AromaPacifico/Lookup` `LookupFound`, `LookupNotFound`, the SLO metrics of section 9 and the probe result. |
| Logs | API Gateway JSON access logs (request identifier, route, status, latency, caller role, never a token); one line per function invocation; CloudFront standard access logs of the lookup distribution delivered to CloudWatch Logs, the measurement source of SLO 2; WAF logs in a log group named `aws-waf-logs-<prefix>-pdn-lookup`, as WAF requires; the application audit records of A08 section 7.2. |
| Traces | X-Ray active tracing on the API stage and every function, and on the lookup function behind CloudFront; the X-Ray trace identifier is written on every log line, so a Logs Insights result opens its trace. |

### 4.4 Databases

The tables are those of [farm monitoring](farm-monitoring.md#66-stores) section 6.6 (telemetry, daily summaries, `thresholds`, alert store), [sack and trip](sack-and-trip.md#10-stores) section 10 (`sack_event_log`, `sack_history`, `sacks`, `sack_codes`, `trips`, `transport_positions`, `buyer_view`), the drone records (cases, orders, missions, campaigns, upload grants) and the result records of [inference functions](inference-functions.md#5-result-record) section 5.

| Kind | Signals |
|---|---|
| Metrics | `AWS/DynamoDB` per table: `ConsumedReadCapacityUnits`, `ConsumedWriteCapacityUnits`, `ReadThrottleEvents`, `WriteThrottleEvents`, `SystemErrors`, `SuccessfulRequestLatency` and `ConditionalCheckFailedRequests` (expected on idempotent puts, watched for change); account-level `UserErrors`; stream consumer `IteratorAge` through the Lambda metrics of the applier, exporter and the drone pipe. |
| Logs | CloudTrail data events of the `thresholds` writes (A14 section 4.5); DynamoDB management events in the organization trail (deletion protection, point-in-time recovery changes). |
| Traces | DynamoDB calls appear as subsegments of the calling function's trace. |

### 4.5 The three collection-center WAN links (closes PD-094)

The IPsec tunnels of each center terminate on the hub in the network account ([connectivity](connectivity.md#9-collection-center-wan) section 9), and the center firewall exports its logs over HTTPS to the CloudWatch Logs endpoint in `pdn` `administration` (CC-ADM-02, ASM-123). Both halves are observed, per center and per traffic class.

| Kind | Signals |
|---|---|
| Metrics, cloud side | `AWS/VPN` `TunnelState`, `TunnelDataIn` and `TunnelDataOut` per VPN connection and tunnel of each center; `AWS/TransitGateway` `BytesIn`, `BytesOut` and `PacketDropCountNoRoute` per center attachment. They live in the network account and are shared with `pdn` through cross-account observability. |
| Metrics, center side | `AromaPacifico/CenterNetwork`, produced by CloudWatch Logs metric filters on the firewall's periodic statistics lines: `ClassThroughputBits` and `ClassDrops` by `CollectionCenter`, `TrafficClass` (the six classes of [collection-center network](collection-center-network.md#8-non-operational-traffic-control) section 8) and direction; `WanActiveLink` (primary or backup); `FirewallLogLines` as the export heartbeat. Metric filters work on the line whichever way it arrives, so the design does not change if PD-123 replaces CC-ADM-02 with a syslog collector. |
| Logs | One log group per center firewall, `/<prefix>/pdn/cc/<center>/firewall`; VPC Flow Logs of rejected traffic (CL-DEF-02). |
| Traces | None; a tunnel carries no trace context. Sack events recorded at a center keep `recorded_at` and `received_at`, so a WAN outage appears as upload delay in `PositionUploadDelay` and in the sack clock fields. |

Drops of the non-operational class are expected and shown, not alarmed, because they are the traffic control of A11 working. Use of the backup link is shown and alarmed when it lasts (section 6.4).

### 4.6 Arrival of flight objects

| Kind | Signals |
|---|---|
| Metrics | S3 request metrics of the drone object bucket with a filter on `farms/`: `PutRequests`, `4xxErrors`, `5xxErrors`, `BytesUploaded`; reception function alias metrics; custom `AromaPacifico/Drone` `ObjectsPersisted`, `ObjectsRejected` (checksum mismatch), `ObjectsQuarantined`, `MissionRecordPersisted` by `MissionType`, and the state-age metrics of sections 5.3 and 5.4. |
| Logs | Reception function lines (mission, object key and version, checksum result, grant check); CloudTrail S3 data events on the bucket, which remain the audit of "entrega de un objeto de vuelo" (A08 section 7.2) and are queried with Athena in the security account, not alarmed. |
| Traces | X-Ray from the reception function through the mission stream, the EventBridge Pipe and bus to the function queues and the inference functions. |

### 4.7 Business trace points

Where a hop runs outside AWS, the trace is a set of timestamps on the record itself, written by the component that observes the hop. These are the trace points that let a breach be attributed to a hop.

| Flow | Trace points, in order | Record |
|---|---|---|
| Crop alert (A10 section 8.3) | `reading_at` (node), `iot_received_at` (rule), `evaluated_at`, `published_at` (threshold function), `sent_at` per channel (dispatcher), `delivered_at` and `delivered_channel` (SMS delivery event or app receipt) | Episode in the alert store |
| Flight | `dispatched_at`, `flown_at`, `first_object_at`, `record_persisted_at`, `result_at` | Mission and result record |
| Sack event | `recorded_at` (device), `received_at` (API), `applied_at` (applier), `exported_at` (exporter) | Event log and history |

## 5. The five detections

The detector function (section 2) evaluates the state-age detections; the alarms read its metrics with a 1-hour period, one datapoint to alarm, and missing data treated as not breaching. The item list of each detection is a Logs Insights query on the detector log group, shown on the operational dashboard. All values are design values (ASM-223, ASM-226, ASM-228).

### 5.1 Nodes and farms without telemetry for more than 24 hours

| Item | Design |
|---|---|
| Signal | Last receive time per node: the newest item of the node in the telemetry table, read with a one-item descending query on its key (ADR-003), for every node whose registry status is `active`. Decommissioned and revoked nodes are excluded by status (A09 section 10). A crop node is always expected to publish; a drying node only while it has an open drying lot ([farm monitoring](farm-monitoring.md#65-drying-lot) section 6.5). |
| Metric | `AromaPacifico/Detector` `NodesSilent24h` and `FarmsSilent24h` by `CollectionCenter`; `NodesSilent8h` as early warning; `ActiveNodes` as the denominator. A farm is silent when every expected node of it is silent. |
| Threshold and window | A node is silent when its last receive time is more than 24 hours old, the retention limit of the buffer, after which its readings are being evicted. Early warning at 8 hours, two crop cadences. Evaluated every hour. |
| Alarms | `<prefix>-pdn-alarm-nodesilent` on `NodesSilent24h` at or above 1 (ticket); `<prefix>-pdn-alarm-farmsilent<department>` on `FarmsSilent24h` at or above 1 per department (agronomy topic of the department, which decides on a farmer call or a technician visit); `<prefix>-pdn-alarm-nodesilentmass` when 10 % or more of the active nodes of one department are silent for 8 hours (page: a carrier or satellite outage, or an ingestion fault). `ReadingsEvicted` above 0 in a day is a ticket, because it is data the buffer lost. |
| Query | `fields node_id, farm_id, zone, department, last_received_at, silent_hours \| filter detector = "node_silence" and silent_hours > 24 \| sort silent_hours desc` |

### 5.2 Crop alerts not delivered (closes PD-134)

The delivery definition of the farm monitoring design is adopted unchanged: an alert opening or closure notice is delivered at the first receipt from either channel, the SMS delivery event or the app receipt ([farm monitoring](farm-monitoring.md#73-path) section 7.3). The dispatcher records the send; the receipt sets `delivered_at` on the episode.

| Item | Design |
|---|---|
| Signal | Episodes whose opening or closure notice was published and has no `delivered_at`; messages in the notification dead-letter queue; SMS sends with a failure delivery status; push sends that SNS reports as failed. |
| Metrics | `AlertsPendingDelivery` (published, no receipt, more than 300 seconds after the reading for a live alert or after publication for a delayed one); `AlertsUndelivered` (no receipt 24 hours after publication), both by `CollectionCenter`, from the detector every 5 minutes. |
| Receipt window for delayed alerts | A delayed alert, published when a buffer drains, has 300 seconds from its publication to be delivered on time and 24 hours from its publication before it counts as undelivered; a live alert has 300 seconds from its reading for the SLO and the same 24 hours for this detection (ASM-224). The case statement's 24-hour limit applies to emission ("la alerta se emite al drenar el buffer, dentro de las 24 horas"), which `DelayedAlertEmittedLate` checks: publication more than 24 hours after the reading. |
| Alarms | `<prefix>-pdn-alarm-notifydlq` on the notification dead-letter queue above 0 (page); `<prefix>-pdn-alarm-alertpending` on `AlertsPendingDelivery` at or above 3 for 30 minutes (ticket: a provider or channel fault affects several farmers); `<prefix>-pdn-alarm-alertundelivered<department>` on `AlertsUndelivered` at or above 1 (agronomy topic of the department and ticket, so that someone reaches the farmer by another way); `<prefix>-pdn-alarm-smsfailed` on `SmsDeliveryFailed` of class `carrier` or `blocked` at or above 3 in 1 hour (ticket); `<prefix>-pdn-alarm-pushfailed` on `NumberOfNotificationsFailed` above 0 for 1 hour (ticket); `<prefix>-pdn-alarm-alertlate` on `DelayedAlertEmittedLate` above 0 (ticket). |
| Query | `fields episode_id, farm_id, department, variable, delayed, published_at, pending_minutes, sms_status, push_status \| filter detector = "alert_pending_delivery" \| sort pending_minutes desc` |

### 5.3 Flights without a persisted record (closes PD-141 with 5.4)

A mission in `dispatched` or `flown` without `record_persisted` is a "vuelo sin registro persistido" ([drone service](drone-service.md#53-mission) section 5.3). The record is published when the link returns, at the latest at the preload point (ASM-028), so a normal flight reaches `record_persisted` the same day.

| Condition | Threshold | Notified |
|---|---|---|
| Mission in `flown`, age since `flown_at` | More than 24 hours | Agronomy topic of the department; ticket. |
| Mission in `dispatched`, age since `dispatched_at` | More than 72 hours, or 24 hours after the package `valid_until`, whichever comes first | Agronomy topic; ticket. |
| Either, beyond the first threshold | More than 7 days | Ticket marked escalated, read by SR1 in the weekly review; the case stays open. |
| Fumigation order in `flown` | 24 hours after `window_end` | Agronomy topic; the order does not expire ([drone service](drone-service.md#52-fumigation-order) section 5.2). |

The metric is `AromaPacifico/Detector` `MissionsWithoutRecord` by `MissionType`, `State` and `CollectionCenter`, with alarms `<prefix>-pdn-alarm-flightnorecord<department>`. The arrival side is watched too: `ObjectsQuarantined` above 0 is a ticket, because it means an object without a valid grant or from a revoked pilot or kit; `ObjectsRejected` at or above 3 in 1 hour is a ticket.

### 5.4 Fumigations without closure

A fumigation counts as open in every order state before `closed` ([drone service](drone-service.md#52-fumigation-order) section 5.2, ADR-002). The ages below are from the state timestamps; the case statement gives none (ASM-226).

| Condition | Threshold | Notified |
|---|---|---|
| Order in `draft` (awaiting the agronomist's authorization) | More than 72 hours | Agronomy topic. |
| Order in `authorized`, not dispatched | 24 hours after `window_start` | Agronomy topic; the order still expires at `window_end`. |
| Order in `executed`, verification not persisted | When the first review campaign of the department that starts after `executed_at` has ended without a persisted verification for the order (the verification flies in a campaign or as an extra mission, [drone service](drone-service.md#11-review-campaign-cadence-pd-012) section 11) | Agronomy topic and ticket: "Si ese archivo no llega, la fumigación sigue abierta". |
| Case in `verification_not_closable` without an agronomist decision | More than 7 days | Agronomy topic. |
| Order flagged `out_of_window`, `batch_mismatch`, `aborted_closure` or `late_record` | Any new flag | Agronomy topic, at the next hourly run. |

The metric is `FumigationsOpen` by `State` and `CollectionCenter`, with an `overdue` dimension value for the items past their threshold, and `OrdersFlagged`; alarms `<prefix>-pdn-alarm-fumopen<department>` and `<prefix>-pdn-alarm-fumflag<department>`. The open count by state, overdue or not, is always shown on the dashboard.

### 5.5 Failed invocations that did not change state (closes PD-162)

The observation point is the one A17 proposed: each function's dead-letter queue together with the missions in `record_persisted` without a result ([inference functions](inference-functions.md#7-error-observation-point-proposed-to-a21) section 7). The alarm names are A17's; this table sets their thresholds and windows (ASM-228). A message reaches a dead-letter queue only after 5 attempts with backoff (ASM-162), so its first message is already a final failure.

| Alarm | Metric | Threshold and window | Severity |
|---|---|---|---|
| `infthresholddlq` | Threshold dead-letter queue `ApproximateNumberOfMessagesVisible` | Above 0, one 1-minute period | Page: a reading was not evaluated, so an alert may be missing. |
| `infconfirmdlq`, `infverifydlq`, `infestimatedlq` | Same, per drone function | Above 0, one 1-minute period | Ticket: a case or estimate is waiting; no farmer is waiting. |
| `infroutedlq` | Pipe and EventBridge target dead-letter queues; `AWS/Events` `FailedInvocations` | Above 0, one 5-minute period | Ticket. |
| `infthresholdage` | Threshold queue `ApproximateAgeOfOldestMessage` | Above 120 seconds for 2 of 2 periods of 1 minute | Page: the alert budget allows 10 seconds for this hop. |
| `infconfirmage`, `infverifyage`, `infestimateage` | Same, per drone queue | Above 2,700 seconds, one 5-minute period: longer than the full retry ladder of about 40 minutes | Ticket. |
| `inffailed` | `AromaPacifico/Inference` `InferenceFailed` by `function` | At or above 5 in 15 minutes for one function | Ticket; the retries are still running. |
| `infawaiting` | `MissionsAwaitingResult`: missions in `record_persisted` of type confirmation, verification or estimation without a result | Age above 1 hour, counted hourly by the detector; at or above 1 | Ticket and agronomy topic: a case cannot move. |
| `infthresholdstore` | `ReadingNotPersisted`; IoT rule error queue depth | Above 0, one 5-minute period | Page: the telemetry store write is failing. |
| `infthrottle` | `InferenceThrottled`; `AWS/Bedrock` throttles for the model | At or above 10 in 15 minutes | Ticket; reserved concurrency already bounds the calls. |
| `infperson` | `ModelPersonDetected` | Above 0 | Page, handled as a privacy incident ([inference functions](inference-functions.md#104-backstop-when-the-screen-misses) section 10.4). |

A17's `guard_skipped`, emitted as `GuardSkipped` by `function` and `reason`, is not a failure when its reason is `state_changed`; it is counted on the dashboard only.

## 6. Supporting alarms

### 6.1 Ingestion

| Alarm | Rule | Severity |
|---|---|---|
| `ingestdrop` | `ReadingsEvaluated` of `Zone = crop` over 4 hours below 50 % of the active crop nodes, for 2 consecutive 4-hour periods; crop nodes read every 4 hours, so a 4-hour period holds about one reading per node | Ticket |
| `iotautherr` | `Connect.AuthError` plus `PublishIn.AuthError` at or above 20 in 1 hour | Ticket; a revoked node still trying, or a wrong certificate |
| `iotrulefail` | `AWS/IoT` `Failure` of any action of the telemetry rule above 0 in 5 minutes | Page, with `infthresholdstore` |
| `ingestclock` | `NodeClockAhead` (reading time later than receive time) at or above 1 in a day | Ticket; a node clock distorts the SLO |

### 6.2 Applications and APIs

| Alarm | Rule | Severity |
|---|---|---|
| `api5xxrate` | `5XXError` divided by `Count` above 5 % over 5 minutes, with at least 20 requests | Page: field apps keep events in their outbox, but a sustained 5XX stops every upload. |
| `apilatency` | `Latency` p95 above 3,000 ms for 3 of 3 periods of 5 minutes | Ticket |
| `cognitothrottle` | Any Cognito throttle in 5 minutes | Ticket |
| `sackrefused` | `SackEventsRefused` at or above 20 in 1 hour | Ticket: an app release or a code batch problem. |
| `sackpending` | Sack events held as pending for more than 24 hours, at or above 1 (detector) | Ticket |
| `sackanomaly` | `SackEventsAnomaly` above 0 in a day | Ticket |
| `sackclockskew` | `SackEventsClockSkewed` at or above 10 in a day | Ticket |

### 6.3 Databases and the sack write path

| Alarm | Rule | Severity |
|---|---|---|
| `sackiterage` | `IteratorAge` of the applier or the exporter above 300 seconds, one 1-minute period (ASM-157) | Page: 10 minutes remain before the 15-minute RPO is at risk. |
| `sackdlq` | Applier or exporter dead-letter queue above 0 | Page: the buyer view or the export copy is behind. |
| `ddbthrottle` | `ReadThrottleEvents` or `WriteThrottleEvents` above 0 for 5 minutes on any table | Ticket |
| `ddbsystemerr` | `SystemErrors` above 0 for 5 minutes on any table | Ticket |

### 6.4 WAN links (PD-094)

| Alarm | Rule | Account | Severity |
|---|---|---|---|
| `wandown<center>` | Maximum `TunnelState` over every tunnel of the center equal to 0 for 5 of 5 periods of 1 minute | Network | Page on `netpage`: reception continues offline (ASM-079), but sack events, printing of new codes and flight uploads from that center wait. |
| `wantunnel<center>` | Any one tunnel of the center at 0 for 15 minutes while another is up | Network | Ticket: redundancy lost. |
| `wanbackup<center>` | `WanActiveLink` reports `backup` for more than 60 minutes | `pdn` | Ticket: non-operational traffic is suspended (ASM-086); the primary service needs a claim. |
| `wansat<center>` | WAN egress throughput above 72 Mbps, 80 % of the 90 Mbps plan of A11, for 3 of 3 periods of 5 minutes | `pdn` | Ticket |
| `wanlogistics<center>` | `ClassDrops` of the logistics or drone class above 0 for 15 minutes | `pdn` | Ticket: a guaranteed class is being dropped, which A11 section 8 says cannot happen at the design load. |
| `fwexport<center>` | `FirewallLogLines` equal to 0 for 15 minutes while a tunnel is up | `pdn` | Ticket: the WAN may be fine, but it is no longer observed. |

### 6.5 Public lookup (PD-151)

The lookup signals the sack design proposed are alarmed as follows; the availability itself is SLO 2 (section 9).

| Alarm | Rule | Severity |
|---|---|---|
| `lookupnotfound` | `LookupNotFound` divided by all lookups above 50 % over 15 minutes, with at least 30 lookups | Ticket: enumeration, or a label printed wrong. |
| `lookupwafblock` | WAF rate-based rule `BlockedRequests` at or above 100 in 5 minutes | Ticket: the defense of [sack and trip](sack-and-trip.md#73-protection-against-enumeration-of-sack-codes) section 7.3 is engaged. |
| `lookupthrottle` | Lookup function `Throttles` above 0 for 5 minutes | Ticket; each throttle is also a bad event of SLO 2. |
| `lookupprobe` | Section 9.4 | Page |

The stream iterator age, the sack dead-letter queue and the pending, anomalous and clock-skewed events of PD-151 are the alarms of sections 6.2 and 6.3.

### 6.6 Observability of observability

An EventBridge rule in each account matches CloudTrail management events that disable what this document relies on (`DeleteAlarms`, `DisableAlarmActions`, `DeleteDashboards`, `DeleteLogGroup`, `PutRetentionPolicy`, `DeleteSubscriptionFilter`, `StopLogging`, `DeleteSchedule`) made by any principal other than the apply role of ADR-003, and sends them to `opsticket`. Observability resources change only through the pipeline, as every other `pdn` resource does.

## 7. SLO method and burn-rate alerts

Both SLOs are event-based: an SLI is the ratio of good events to valid events, counted as two custom metrics, `...Good` and `...Valid`, so that every window is metric math on the same counts. The window is a rolling 28 days, four whole weeks, so that every window holds the same mix of weekdays and harvest-day shifts. The error budget is `(1 - objective) x valid events in the window`; the burn rate over a window is `(bad / valid) / (1 - objective)`.

Alerts follow the multiwindow rule: each alert fires only when the burn rate exceeds its factor over both a long and a short window, the long window giving significance and the short window ensuring the alert stops soon after the problem does. Each window is one CloudWatch alarm on a metric-math expression whose period is the window, and the two are joined by a composite alarm with `AND`. Pilot volumes are small, so a single bad event in a quiet hour would otherwise produce a burn rate of 20; each expression therefore requires a minimum number of bad events (ASM-225).

| Alert | Long window | Short window | Burn-rate factor | Budget spent when it fires | Notified |
|---|---|---|---|---|---|
| Fast burn | 1 hour | 5 minutes | 14.4 | 14.4 x 1 / 672 = 2.1 % | `opspage` |
| Slow burn | 6 hours | 30 minutes | 6 | 6 x 6 / 672 = 5.4 % | `opspage` |
| Steady burn | 24 hours | 2 hours | 3 | 3 x 24 / 672 = 10.7 % | `opsticket` |

The longest window is 24 hours so that each alarm evaluates a single period. The 28-day attainment and the remaining budget are shown on the dashboard, not alarmed. When the budget of an SLO is exhausted, the error-budget policy applies (ASM-227): the next `pdn` change of the path the SLO covers must be a fix or carry the SLO owner's agreement in its release note (A22), and a retrospective is written as for an incident (ASM-183).

## 8. SLO 1: crop alert delivery (closes PD-082)

The SLI adopts the A10 proposal ([connectivity](connectivity.md#83-tie-to-the-alert-delivery-slo) section 8.3) and makes its events exact.

| Item | Definition |
|---|---|
| Valid event | An alert opening of an episode of a crop variable (crop zone, from a crop node), in `pdn`, of a node `active` at the reading time, whose reading was published live (`delayed = false`, receive time at most 300 seconds after `reading_at`), excluding an opening whose route to the farmer was not available (below). Soil pH from a laboratory analysis is excluded: it has no reading time on the live path. |
| Good event | A valid event whose first `delivered_at`, from either channel, is at most 300 seconds after `reading_at`, the case statement's limit ("menos de 5 minutos desde la lectura fuera de rango"). |
| Route not available | The case statement measures only "con ruta de entrega disponible". A10 section 8.1 places the node side in the live publication, which the valid definition already requires, and the farmer side in the phone. An opening is excluded only when no receipt arrived within 300 seconds and the SMS delivery event reported the handset unreachable (absent subscriber, phone off, out of coverage) and no push receipt arrived. A carrier, provider or platform failure is never an exclusion. Excluded openings are counted in `AlertDeliveryExcluded` and still followed by section 5.2. |
| Clock rule | `reading_at` comes from the node clock. If it is later than `iot_received_at`, the node clock is ahead and the measurement uses `iot_received_at` instead, counted in `NodeClockAhead` (section 6.1). A device receipt time later than the API time of the receipt is replaced by the API time. |
| Where it is measured | On the episode in the alert store (section 4.7). Every 5 minutes the detector judges each live opening whose 300 seconds have elapsed and that is not yet judged, and emits `AromaPacifico/Alerts` `AlertDeliveryValid`, `AlertDeliveryGood` and `AlertDeliveryExcluded` by `CollectionCenter`; it writes no data store, so a judgement is reproducible from the episode at any time. |
| Objective | 95 % of valid events good (ASM-224). |
| Window | Rolling 28 days. |
| Error budget | 5 % of the valid events of the window. No alert rate is given by the case statement, so no count is computed here; A23 assumes a rate for cost (PD-133). |
| Burn-rate alerts | Section 7, with at least 2 bad events in the long window for fast burn and 3 for slow and steady burn. Bad ratio thresholds: 72 %, 30 % and 15 %. Alarms `<prefix>-pdn-alarm-sloalertfast`, `sloalertslow` and `sloalertticket`, each a composite of two windows. |
| Attribution | For each bad event, the hop that exceeded its A10 budget (10, 60, 20, 10, 20, 30 and 60 seconds) is read from the trace points and logged by the detector as `breach_hop`; the dashboard shows bad events by hop, and the in-cloud hops 4 to 6 open their X-Ray traces. |

Drying alerts and closure notices use the same budget (A10 section 8.2). They are measured with the same definitions as a secondary indicator, `Zone = drying`, shown on the dashboard and not part of the SLO, because point 6 names the crop alert. Delayed alerts are measured against their own windows (section 5.2) and shown beside the SLO: the share emitted within 24 hours of the reading, and the share delivered within 300 seconds of publication.

## 9. SLO 2: sack lookup availability (closes PD-151)

The SLI scope is the lookup path of the sack design: CloudFront, AWS WAF, the lookup function and `buyer_view` ([sack and trip](sack-and-trip.md#71-path) section 7.1).

### 9.1 SLI

| Item | Definition |
|---|---|
| Valid event | An HTTP `GET` or `HEAD` request for `/s/<code>` received by the lookup distribution in `pdn` and allowed by AWS WAF. Requests that WAF blocked are excluded, because blocking them is the enumeration defense working; they are watched by `lookupwafblock`. Probe requests (section 9.4) are excluded. |
| Good event | A valid request answered with status 200 (the buyer view) or 404 (the uniform "not found" of section 7.1 of the sack design, which is a correct answer). |
| Bad event | A valid request answered with any 5XX status, or with 429 because the lookup function's reserved concurrency was exhausted. |
| Where it is measured | At the edge, from the CloudFront standard access logs of the lookup distribution delivered to CloudWatch Logs: two metric filters on `cs-uri-stem` beginning with `/s/` and on `sc-status` produce `AromaPacifico/Lookup` `LookupValid` and `LookupGood`, excluding the WAF-blocked status (403 with the WAF edge result) and the probe user agent. The edge sees failures the function never sees: an origin timeout, a function URL error, a throttle. |
| Cross-check | `AWS/CloudFront` `5xxErrorRate` and `Requests` of the same distribution, and the lookup function `Errors` and `Throttles`, on the same dashboard row; a difference between them points to the log delivery, not to the lookup. |

### 9.2 Objective, window and budget

| Item | Value |
|---|---|
| Objective | 99.5 % of valid requests good (ASM-225). |
| Window | Rolling 28 days. |
| Error budget | 0.5 % of the valid requests of the window. At an even request rate this equals 0.005 x 672 hours, about 3.4 hours of total unavailability in 28 days. |
| Relation to the RTO | The RTO of 4 hours ("RTO de la consulta pública: 4 horas") is the ceiling for one incident; the SLO asks that unavailability over 28 days stay below the budget. One incident that uses the whole RTO therefore spends the budget and triggers the error-budget policy of section 7, which is intended: an outage of that length is not the normal state of the lookup. |
| Burn-rate alerts | Section 7, with at least 10 bad requests in the long window for fast burn and 20 for slow and steady burn. Bad ratio thresholds: 7.2 %, 3 % and 1.5 %. Alarms `<prefix>-pdn-alarm-slolookupfast`, `slolookupslow` and `slolookupticket`. |

### 9.3 Why not measure at the function or the table

A function-side SLI would count only requests that reached the function, so a broken origin access control or a CloudFront behavior error would read as 100 % available. A table-side SLI would ignore the WAF and the function. The edge is where the buyer is.

### 9.4 Synthetic probe

The 4-hour RTO budget starts with 15 minutes of detection (ASM-157), and outside harvest the lookup may receive no request for hours. A CloudWatch Synthetics canary in `us-east-1` requests `/s/<probe code>` every 5 minutes, expects status 200 and the probe sack's state, and fails on anything else. The probe code is a reserved `buyer_view` item that belongs to no farm, flagged `synthetic`, so that its audit records are marked and excluded from the farm allocation of A08 section 5.3 (ASM-225). Alarm `<prefix>-pdn-alarm-lookupprobe` fires at 3 consecutive failed runs, 15 minutes, on `opspage`. The probe is not part of the SLI, whose events are buyers' requests.

## 10. Rollback alarm set for the CodeDeploy canary (closes PD-173)

ASM-174 deploys functions in `qa` and `pdn` with `CodeDeployDefault.LambdaCanary10Percent5Minutes` and automatic rollback on the alarm set of each function ([automation and pipelines](automation-and-pipelines.md#43-how-function-code-is-deployed) section 4.3). An alarm in that set must reach `ALARM` well inside the 5-minute canary, so every alarm below uses 1-minute periods, one datapoint to alarm and missing data treated as not breaching: with no traffic it stays `OK`, and the first failing minute raises it within about 2 minutes of the failure, leaving time for the rollback target of 10 minutes (ASM-174). Thresholds are deliberately strict, because during a canary a false rollback costs a re-run while a missed one costs a defect in production (ASM-229).

| Alarm of the set | Applies to | Metric | Threshold |
|---|---|---|---|
| `<function>-errors` | Every function | `AWS/Lambda` `Errors` of the `live` alias (`Resource` dimension) | At or above 1 in 1 minute |
| `<function>-throttles` | Every function | `Throttles` of the alias | At or above 1 in 1 minute |
| `<function>-dlqgrowth` | Functions with a dead-letter queue (the four inference functions, the dispatcher, the reception function, the applier and the exporter) | Metric math `DIFF` of the dead-letter queue's `ApproximateNumberOfMessagesVisible` | Above 0 in 1 minute |
| `api-5xx` | API functions | API Gateway `5XXError` divided by `Count` | Above 5 % in 1 minute, with at least 3 requests |
| `<function>-duration` | Threshold function and dispatcher, the alert path | `Duration` p99 of the alias | Above 20,000 ms (threshold) and 30,000 ms (dispatcher), the A10 budgets of hops 5 and 6, in 1 minute |
| `<function>-iterage` | Applier and exporter | `IteratorAge` | Above 300,000 ms in 1 minute |
| `lookup-5xx` | Lookup function | `AWS/CloudFront` `5xxErrorRate` of the lookup distribution | Above 5 % in 1 minute, with at least 3 requests; and `lookupprobe` |

The set uses the growth of a dead-letter queue, not its depth. The operational alarms of section 5.5 alarm on depth and stay in `ALARM` until a redrive; if they were in the CodeDeploy group, an old message would block the deployment that fixes its cause. Before starting a deployment, the `pdn` job checks that every alarm of the set is `OK`.

Two limits are stated, not hidden. First, CodeDeploy watches the alarms only while the deployment runs; after the shift to 100 % the operational alarms and the SLO burn-rate alerts take over, and a rollback is the rollback deployment of A18 section 10. Second, a function invoked rarely (the drone functions, the estimation function, the lookup outside harvest) may receive no canary traffic in 5 minutes, so its canary proves nothing; for those the `qa` acceptance suite with recorded sample missions (ASM-177) is the evidence, and a pre-traffic hook that invokes the new version on a sample is left to the platform repository (PD-221).

**Deployment view.** An EventBridge rule in each environment sends the CodeDeploy deployment state-change events to the log group `/<prefix>/pdn/deployments`, and the operational dashboard lists them with their outcome and rollback (section 11). The GitHub deployment record of A18 section 12 stays in GitHub and is linked from the release note.

## 11. Dashboards (closes PD-077)

### 11.1 Operational dashboard

`<prefix>-pdn-dash-ops`, defined in [`dashboards/operational.json`](dashboards/operational.json), lives in each workload account; its `dev` and `qa` copies are the testing split. Its rows answer, from top to bottom: are the two SLOs met and how much budget is left; what is in `ALARM`; is telemetry arriving from every department and which farms are silent; are alerts reaching farmers; which flights and fumigations are stuck; are the inference functions failing without state change; are sack events and the lookup healthy; are the databases throttling; how are the three WAN links and their traffic classes; and what was deployed. The split by the case statement's tags is: farms by Logs Insights tables on `farm_id`; drones by the drone and inference rows; collection centers by the `CollectionCenter` dimension on every row; testing by the environment account.

### 11.2 Cost dashboard

`<prefix>-shared-dash-cost`, defined in [`dashboards/cost.json`](dashboards/cost.json), lives in the management account, where the billing data, Cost Explorer and the budgets are. It shows no price: it shows what the bill says. Its sources are `AWS/Billing` `EstimatedCharges` (month to date, by linked account and by service, in `us-east-1`), and the metrics `AromaPacifico/Cost` that the cost export function publishes once a day from Cost Explorer and Budgets: `DailyCost` and `MonthToDateCost` by `CostLine` (the values of the A08 cost category: farms, drones, collection centers, trips, testing, shared and `unallocated`), by `CollectionCenter` and by `Driver`; `DailyUsage` by `Driver`; and `BudgetActualPercent` and `BudgetForecastPercent` by budget. The drivers are those the design names: IoT Core messages, Lambda, Bedrock, DynamoDB, S3, Site-to-Site VPN and Transit Gateway, NAT gateway (none exists, ASM-127, so the widget expects zero and alarm `<prefix>-shared-alarm-natcost` fires on any NAT gateway hour), CloudFront, WAF, SMS and push, CloudWatch and KMS. The mapping of Cost Explorer services and usage types to drivers is configuration of the function (ASM-221). `unallocated` above 5 % of the month-to-date cost raises `<prefix>-shared-alarm-unallocated` to the budget topic, so a tagging gap is fixed in the month it appears. The per-farm cost is the usage allocation of A08 section 5.3, computed monthly from the CUR 2.0 export; the dashboard shows the farms line and its mean per enrolled farm. Amounts are A23's (PD-076, PD-222).

Every widget of both dashboards, and the question it answers, is described in [`dashboards/README.md`](dashboards/README.md).

## 12. Plan-only boundary

Nothing in this document is deployed. The alarms, composite alarms, metric filters, schedules, canary, topics, the cross-account link and the two dashboard bodies are definitions that the IaC would apply in `aroma-pacifico-infrastructure`; the detector and cost export functions are code of the platform repository (ASM-170), which the pilot does not write. Whether the D10 function pattern of Epic 6 codes its own alarm set (errors, throttles, dead-letter queue) is decided by the 100-hour plan (PD-223). No figure in this document is a measurement, and no SLO attainment is reported: the first attainment is read after 28 days of production traffic.

## 13. Decisions closed and what remains

| PD | Status | Where it is met, or what remains |
|---|---|---|
| PD-077 | Closed | Sections 3.1 and 11 and [`dashboards/README.md`](dashboards/README.md): both dashboards, split by farms, drones, collection centers and testing, with the `unallocated` cost shown and alarmed. |
| PD-082 | Closed | Section 8: SLI adopted with exact valid, good and excluded events, 95 % over 28 days, burn-rate alerts and per-hop attribution. |
| PD-094 | Closed | Sections 4.5 and 6.4: each WAN link by tunnel and by traffic class, drops of non-operational traffic shown, backup use alarmed. |
| PD-134 | Closed | Section 5.2: delivery definition adopted, undelivered alerts and the notification dead-letter queue alarmed, receipt windows set. |
| PD-141 | Closed | Sections 5.3 and 5.4: ages for `dispatched`, `flown`, each order state before `closed` and flagged orders. |
| PD-151 | Closed | Sections 6.5 and 9: SLI on the lookup path measured at the edge, 99.5 % over 28 days, and every proposed signal alarmed. |
| PD-162 | Closed | Section 5.5: every A17 alarm with threshold and window, and the 1-hour age for missions awaiting a result. |
| PD-173 | Closed | Section 10: rollback alarm set, thresholds, 1-minute windows inside the 5-minute canary, and the deployment view. |
| Remaining | Open | PD-220: the AWS features relied on here are re-checked against live documentation. PD-221: pre-traffic hooks for rarely invoked functions. PD-222: A23 prices observability itself. PD-223: the 100-hour plan decides whether D10 codes its alarm set. The objectives of ASM-224 and ASM-225 are assumptions to be confirmed with the Federation. |

## 14. Compliance with the issue

| Acceptance criterion of aroma-pacifico/aroma-pacifico-roadmap#34 | Where it is met |
|---|---|
| The proposal covers metrics, logs and traces for ingestion, applications, the database, the 3 WAN links and flight object arrival. | Section 4 (4.1 to 4.7), with the services of section 2. |
| It detects nodes or farms without telemetry for more than 24 hours, undelivered crop alerts, flights without a persisted record, fumigations without closure and failed invocations with no state change. | Section 5 (5.1 to 5.5), each with signal, metric or query, threshold, window and recipient. |
| An SLO is defined for crop alert delivery, with its SLI. | Sections 7 and 8. |
| An SLO is defined for sack lookup availability, with its SLI. | Sections 7 and 9. |
| An operational dashboard and a cost dashboard are delivered. | Section 11, [`dashboards/operational.json`](dashboards/operational.json), [`dashboards/cost.json`](dashboards/cost.json) and [`dashboards/README.md`](dashboards/README.md). |

## 15. Decision-log entries

| ID | Summary |
|---|---|
| PD-077, PD-082, PD-094, PD-134, PD-141, PD-151, PD-162, PD-173 | Closed by sections 11, 8, 4.5 and 6.4, 5.2, 5.3 and 5.4, 9, 5.5 and 10. |
| ASM-220 | AWS-native observability only; metric namespaces and dimensions; no `FarmId` metric dimension. |
| ASM-221 | Detector function and cost export function. |
| ASM-222 | Notification topics, recipients and severities. |
| ASM-223 | Silence thresholds for nodes and farms. |
| ASM-224 | SLO 1 objective, window, exclusion and receipt windows. |
| ASM-225 | SLO 2 objective and window, burn-rate policy and minimum bad counts, synthetic probe. |
| ASM-226 | Ages for flights without record and fumigations without closure. |
| ASM-227 | Error-budget policy. |
| ASM-228 | Thresholds of the inference, ingestion, application, database, WAN and lookup alarms. |
| ASM-229 | Rollback alarm set and its evaluation windows. |
| PD-220 to PD-223 | Live re-check, pre-traffic hooks, observability cost, D10 alarm scope. |

## 16. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Point 6, the alert latency, the RPO and RTO, Part E. |
| [`docs/decision-log.md`](decision-log.md) | Every ASM, ADR and PD cited. |
| [`docs/architecture.md`](architecture.md) | Section 8 hooks, component inventory and section 17.3. |
| [`docs/connectivity.md`](connectivity.md) | Alert budget per hop (section 8), WAN (section 9). |
| [`docs/farm-monitoring.md`](farm-monitoring.md) | Delivery definition, dispatcher, drying lots, observability hooks. |
| [`docs/drone-service.md`](drone-service.md) | Mission and order states, observability hooks. |
| [`docs/inference-functions.md`](inference-functions.md) | Retries, dead-letter queues, observation point. |
| [`docs/sack-and-trip.md`](sack-and-trip.md) | Lookup path, offline rules, recovery objectives. |
| [`docs/collection-center-network.md`](collection-center-network.md), [`docs/firewall-policy.md`](firewall-policy.md) | Traffic classes, WAN failure behavior, CC-ADM-02. |
| [`docs/automation-and-pipelines.md`](automation-and-pipelines.md) | Canary configuration, rollback, hooks. |
| [`docs/landing-zone.md`](landing-zone.md) | Tags, cost category, budgets, audit. |
| [`docs/dashboards/README.md`](dashboards/README.md) | Widget catalog of both dashboards. |
