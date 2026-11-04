# Operational and Cost Dashboards

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR2 / JR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#34 (A21) |
| Design | [`docs/observability.md`](../observability.md) section 11 |
| Decision-log entries | PD-077 (closed by this page and the design), ASM-220, ASM-221, PD-220, PD-222 |

This page describes the two CloudWatch dashboards of point 6 of the [case statement](../case/case-statement.pdf) ("Tablero operativo y tablero de costo, partidos por las etiquetas de fincas, drones, acopio y pruebas"), widget by widget, with the question each widget answers. The two JSON files are CloudWatch dashboard bodies. Nothing is deployed: the IaC would apply each body with an `aws_cloudwatch_dashboard` resource whose `dashboard_body` is `templatefile()` of the file, so the `${...}` tokens below are template variables, not CloudWatch syntax. Every threshold drawn on a widget is a design value of the observability design; no value is a measurement.

| File | Dashboard name | Account | Purpose |
|---|---|---|---|
| [`operational.json`](operational.json) | `<prefix>-pdn-dash-ops` (and the `dev` and `qa` copies) | Each workload account | Operations of point 6: SLOs, detections, ingestion, alerts, drones, inference, sack and lookup, databases, WAN links, deployments. |
| [`cost.json`](cost.json) | `<prefix>-shared-dash-cost` | Management account | Cost by the pilot cost lines, collection center, testing and the drivers the design names, with the budgets. |

## 1. Template variables

| Variable | Value supplied by the IaC |
|---|---|
| `${prefix}` | `{client}-{project}` of PC-IAC-003, fixed by PD-070. |
| `${env}` | `dev`, `qa` or `pdn`. |
| `${account_id}`, `${management_account_id}`, `${network_account_id}` | Account identifiers from the stack configuration of `aroma-pacifico-infrastructure`; never written in this repository. |
| `${vpn_id_cauca}`, `${vpn_id_huila}`, `${vpn_id_tolima}` | VPN connection identifiers of the three centers, outputs of the hub stack in the network account. |
| `${telemetry_rule_name}`, `${api_name}`, `${lookup_distribution_id}` | Outputs of the ingestion and platform modules. |
| `${enrolled_farms}` | Number of enrolled farms used to show the mean farm-line cost; 200 in the pilot. |

Resource names inside the bodies (queues such as `sqs-thresholddlq`, functions such as `lambda-dispatcher`, the bucket `s3-droneobjects`, the web ACL `waf-lookup`, the canary `canary-lookup`) follow PC-IAC-003 with the keys of the earlier designs; the IaC that creates the resources is the authority, and a key it shortens is changed here in the same pull request.

## 2. Conventions the bodies rely on

Every custom metric is emitted by its embedded metric format line twice: once with its dimension set (for example `CollectionCenter`) and once with no dimension, so that a widget can read the pilot total without a search expression. Search expressions (`SEARCH('{namespace,dimension} MetricName="..."', ...)`) draw one line per dimension value, which is how a widget splits by department, collection center, traffic class, cost line or driver without listing the values. Metrics of the network account carry the `accountId` option and are read through cross-account observability (ASM-220). Log widgets are Logs Insights queries over the detector log group, whose lines carry `farm_id`; they are the farm split, because `FarmId` is not a metric dimension (ASM-220). Widgets with their own `start` show a fixed range (28 days for SLO attainment, 7 days for SLO trends, 35 days for daily cost) whatever the dashboard range is. The section headers of the operational dashboard are one-line text widgets and carry no number.

## 3. Operational dashboard

### 3.1 SLOs and alarms

| Widget | Shows | Question it answers |
|---|---|---|
| OP-00 | Title and how the dashboard is split. | What am I looking at, and where are farms, drones, centers and testing? |
| OP-01 | SLO 1 attainment over the last 28 days: `AlertDeliveryGood` divided by `AlertDeliveryValid`. | Are crop alerts reaching farmers within 5 minutes at least 95 % of the time? |
| OP-02 | Share of the SLO 1 error budget left over 28 days. | How much room is left before the error-budget policy applies? |
| OP-03 | SLO 2 attainment over 28 days: `LookupGood` divided by `LookupValid`. | Is the sack lookup answering at least 99.5 % of buyers' requests? |
| OP-04 | Share of the SLO 2 error budget left over 28 days. | Same, for the lookup. |
| OP-05 | State of the six burn-rate composite alarms and the lookup probe. | Is either SLO burning its budget fast enough to page? |
| OP-06 | SLO 1 good ratio per hour for 7 days, with the 95 % line, and the excluded openings (handset unreachable) on the right axis. | When did deliveries degrade, and is the exclusion growing in a way that hides a fault? |
| OP-07 | SLO 1 bad events by the hop that broke its A10 budget, and by department. | Which hop (node attach, IoT Core, function, SNS, provider, phone) causes the misses, and where? |
| OP-08 | SLO 2 good ratio per hour for 7 days and the Synthetics probe success. | Did buyers see failures, and did the probe agree? |
| OP-09 | Every page-level alarm of the observability design. | What is paging now? |

### 3.2 Ingestion and farms without telemetry

| Widget | Shows | Question it answers |
|---|---|---|
| OP-10 | `ReadingsEvaluated` per hour, one line per department and zone. | Is telemetry arriving from Cauca, Huila and Tolima, crop and drying? |
| OP-11 | Ingest lag p95, delayed readings and readings evicted from node buffers. | Are buffers draining, how late, and is data being lost beyond 24 hours? |
| OP-12 | IoT Core connection and publish authentication errors, topic rule action failures, rule error queue depth. | Are revoked or misconfigured nodes knocking, and is the rule failing to store readings? |
| OP-13 | Nodes and farms silent for more than 24 hours, per department. | How many nodes and farms have gone quiet, and where? |
| OP-14 | Table of nodes silent for more than 8 hours, by farm, node, zone and department. | Which farms must the agronomist or a technician reach? |

### 3.3 Crop alerts to farmers

| Widget | Shows | Question it answers |
|---|---|---|
| OP-15 | Alerts published, SMS sent, push sent, SMS failed. | Is the dispatcher sending on both channels? |
| OP-16 | Alerts pending delivery and undelivered, per department. | How many farmers have an alert they have not received? |
| OP-17 | Notification queue oldest age, its dead-letter queue, dispatcher p99 duration. | Is hop 6 within its budget, and has any notification failed for good? |
| OP-18 | Table of alerts without a receipt: episode, farm, department, variable, delayed flag, minutes pending, SMS and push status. | Which farmer must be reached another way? |

### 3.4 Drones: flight objects, flights and fumigations

| Widget | Shows | Question it answers |
|---|---|---|
| OP-19 | Objects persisted, rejected and quarantined; S3 PUT requests and 4xx on `farms/`. | Are flight objects arriving and being accepted? |
| OP-20 | Missions without a persisted record, by mission type and state. | How many flights have not closed their record? |
| OP-21 | Fumigations open by order state, overdue or not, and flagged orders per department. | How many fumigations are open, and which are past their age? |
| OP-22 | Table of overdue missions and orders with farm, department, state and age. | Which flight or fumigation does the agronomist follow up today? |

### 3.5 Inference functions

| Widget | Shows | Question it answers |
|---|---|---|
| OP-23 | Messages in the threshold, confirmation, verification, estimation and routing dead-letter queues. | Has any invocation failed for good without changing state? |
| OP-24 | Oldest message age of the four function queues, with the 120-second and 2,700-second lines. | Is a queue stuck behind failures? |
| OP-25 | `InferenceFailed` by function, missions awaiting a result, and `GuardSkipped` by reason. | Are invocations failing while retries run, and are cases waiting for a result? |
| OP-26 | Bedrock input and output tokens, images sent, throttles. | How much inference is used, and is the model throttling? |
| OP-27 | Objects excluded by the person screen, not screened, and persons detected by the model. | Is the person rule holding, and is there a privacy incident? |

### 3.6 Sack events, API and public lookup

| Widget | Shows | Question it answers |
|---|---|---|
| OP-28 | Sack events accepted per collection center; refused, pending, anomalous and clock-skewed events. | Are receptions flowing, and are the offline rules flagging problems? |
| OP-29 | Iterator age of the applier and exporter, with the 300-second line, and their dead-letter queues. | Is the 15-minute RPO safe, and is the buyer view current? |
| OP-30 | API requests, 5XX, 4XX and p95 latency. | Are the field apps' uploads succeeding? |
| OP-31 | Lookups found and not found; WAF blocks, all rules and the rate-based rule. | Is the lookup being used normally, or enumerated? |
| OP-32 | CloudFront 5xx rate and origin latency of the lookup; lookup function throttles and errors. | Where does a lookup failure come from: edge, origin or function? |

### 3.7 Databases

| Widget | Shows | Question it answers |
|---|---|---|
| OP-33 | Read and write throttles and system errors of every table. | Is any table refusing work? |
| OP-34 | Consumed read and write capacity of every table, per hour. | Which tables carry the load, and does it match the figures of the designs? |

### 3.8 The three collection-center WAN links

| Widget | Shows | Question it answers |
|---|---|---|
| OP-35 | Tunnel state of Cauca, Huila and Tolima (1 is up), read from the network account. | Is each center connected to the cloud? |
| OP-36 | Tunnel bytes in and out per center. | How much each center sends and receives through its tunnel. |
| OP-37 | Backup link in use per center and the firewall log export heartbeat. | Is a center on its cellular backup, and is its firewall still observed? |
| OP-38 | Throughput per center, traffic class and direction, with the 72 Mbps line. | Is the 100 Mbps link near saturation, and which class uses it? |
| OP-39 | Drops per center, traffic class and direction. | Is non-operational traffic being capped as designed, and is any guaranteed class dropped? |

### 3.9 Deployments

| Widget | Shows | Question it answers |
|---|---|---|
| OP-40 | CodeDeploy deployment state changes: application, function deployment group, deployment, outcome. | What was deployed, and did any canary roll back? |

## 4. Cost dashboard

The cost dashboard shows what the bill says and sets no price; prices and budget amounts are A23's (aroma-pacifico/aroma-pacifico-roadmap#36, PD-076, PD-222). `AWS/Billing` metrics are in `us-east-1` and update a few times a day; the `AromaPacifico/Cost` metrics are written once a day by the cost export function from Cost Explorer and Budgets (ASM-221), so they lag up to one day. Cost lines are the values of the landing zone cost category: farms, drones, collection centers, trips, testing, shared and `unallocated`, and `total` for their sum.

| Widget | Shows | Question it answers |
|---|---|---|
| CO-00 | Title, sources and lag. | Where do these figures come from? |
| CO-01 | Month-to-date estimated charges of the organization. | What has the pilot cost so far this month? |
| CO-02 | `unallocated` as a percentage of month-to-date cost; its alarm fires above 5 %. | Is any cost escaping the tags? |
| CO-03 | Month-to-date cost of the testing line (test and development accounts, ASM-064). | What does testing cost? |
| CO-04 | State of the `unallocated` and NAT gateway cost alarms. | Is a tagging gap or an unexpected NAT gateway costing money? |
| CO-05 | Daily cost per cost line, stacked, 35 days. | Which line drives the daily cost, and does a harvest month look different? |
| CO-06 | Month-to-date cost per cost line. | How is this month split between farms, drones, centers, trips, testing and shared? |
| CO-07 | Month-to-date cost per `CollectionCenter` tag value, and cost without the tag. | What does each collection center cost? |
| CO-08 | Farms line month to date, and its mean per enrolled farm. | What does a farm cost on average this month? |
| CO-09 | Estimated charges per linked account. | How is the cost split between production, test, development and the shared accounts? |
| CO-10 | Daily cost per driver, stacked, 35 days. | Which service the design names drives the cost? |
| CO-11 | Estimated charges per AWS service, from the billing metrics. | Does the daily export agree with the bill? |
| CO-12 to CO-23 | For each driver, daily cost and daily usage quantity: IoT Core messages, Lambda, Bedrock tokens, DynamoDB request units, S3, VPN and Transit Gateway, NAT gateway (expected zero, ASM-127), CloudFront, WAF, SMS and push, CloudWatch, KMS. | Is a cost change explained by usage, and is usage in line with the load figures of the designs? |
| CO-24 | Actual spend of each budget as a percentage of its amount, with the 80 % and 100 % lines of ASM-066. | Is any budget near its alert? |
| CO-25 | Forecasted spend of each budget as a percentage of its amount, with the 100 % line. | Will any budget be exceeded this month? |
| CO-26 | How the per-farm cost is obtained: the usage allocation of the landing zone, monthly from the CUR 2.0 export. | Where is the cost of one farm, one trip or one campaign? |

## 5. How the dashboards meet the tag split

| Split of point 6 | Operational dashboard | Cost dashboard |
|---|---|---|
| Farms (fincas) | OP-10 to OP-18 by department, and the farm tables OP-14 and OP-18 by `farm_id` | Farms line in CO-05, CO-06 and CO-08; per farm by the allocation of CO-26 |
| Drones | OP-19 to OP-27 | Drones line in CO-05 and CO-06; Bedrock, S3 and Lambda drivers |
| Collection centers (acopio) | `CollectionCenter` dimension on OP-10, OP-13, OP-16, OP-28; WAN row OP-35 to OP-39 | Collection-centers line, CO-07, VPN and Transit Gateway driver |
| Testing (pruebas) | The same body deployed in the `dev` and `qa` accounts | Testing line, CO-03, linked accounts in CO-09 |

## 6. Verification

Both files parse as JSON (`python3 -m json.tool`), every widget lies inside the 24-column grid, and every widget number is unique. The bodies have not been applied to an account; the dashboard properties they use are listed for the live documentation re-check of PD-220.
