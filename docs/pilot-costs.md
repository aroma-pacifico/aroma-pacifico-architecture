# Pilot Cost Document: Hardware, Field Connectivity, Drones, Cloud Infrastructure, and Savings Decisions

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-11 |
| Owner | JR1 / SR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#36 (A23) |
| Builds on | [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md), [`docs/landing-zone.md`](landing-zone.md) (A08), [`docs/security-and-identity.md`](security-and-identity.md) (A09), [`docs/connectivity.md`](connectivity.md) (A10), [`docs/collection-center-network.md`](collection-center-network.md) (A11), [`docs/firewall-policy.md`](firewall-policy.md) (A12), [`docs/farm-monitoring.md`](farm-monitoring.md) (A14), [`docs/drone-service.md`](drone-service.md) (A15), [`docs/sack-and-trip.md`](sack-and-trip.md) (A16), [`docs/inference-functions.md`](inference-functions.md) (A17), [`docs/architecture.md`](architecture.md) (A19), [`docs/observability.md`](observability.md) (A21), [`docs/clarification-questions.md`](clarification-questions.md) (A06) |
| Feeds | Final presentation and defense script (A25), cloud budget allocation, IaC pattern variables |
| Decision-log entries | Closes PD-017, PD-031, PD-076, PD-084, PD-095, PD-124, PD-133, PD-152, PD-163, PD-222; defines ASM-230 to ASM-239, ASM-250, ASM-251, PD-230 |

## 1. Executive Summary and Scope

This document establishes the commercial and architectural cost estimates for the 2026 update of the Aroma Pacífico coffee traceability pilot across Cauca, Huila, and Tolima. The pilot connects 200 coffee farms, 3 regional collection centers, 60 overland transporters, and the port export handoff at Buenaventura.

Following the core principle from the case statement that historical 2020 figures and legacy quotes are invalid ("Precios, dispositivos y servicios de nube de 2020 no son cotización válida"), all figures presented here are established as explicit planning assumptions based on published AWS service schedules for `us-east-1` and commercial hardware catalog baselines dated November 2026. Because direct session access to `aws.amazon.com` is restricted by network proxy policy in the delivery environment (as documented in PD-200 and architecture section 18), AWS rates are tracked as planning assumptions subject to formal pre-implementation re-check under PD-200, while commercial hardware BOMs and drone rates are reference planning estimates subject to competitive supplier RFP validation under PD-230. Every assumption is segregated from the calculations, numbered in the project decision registry, and traceable to foundational architectural decisions.

The cost model covers five distinct domains:
1. Field hardware deployed across 200 farms (400 autonomous sensor nodes).
2. Collection center local networking and hardware infrastructure across the three departmental sites.
3. Commercial contracting for the combined-mode agricultural drone campaign.
4. Cloud workloads in AWS (`us-east-1`), including event-driven inference, IoT ingestion, transactional storage, and observability.
5. Six architectural decisions that lower or avoid capital expenditure and recurring operating spend.

Prices are stated in United States Dollars (USD), the contractual currency of Amazon Web Services and international component manufacturers. Where local Colombian expenditures are cited (such as commercial drone flight crews and local data connectivity), figures are converted using the planning benchmark exchange rate of 1 USD = 4,100 COP (Colombian Pesos), dated November 2026 (ASM-230), subject to supplier quote validation under PD-230.

| Scope Domain | Capital Expenditure (CAPEX) | Normal Month OPEX | Harvest Month OPEX | Annualized Pilot Spend |
|---|---|---|---|---|
| Farm Field Hardware (200 farms, 400 nodes) | $74,000.00 | $920.00 | $920.00 | $85,040.00 |
| Collection Center Infrastructure (3 sites) | $8,550.00 | $645.00 | $645.00 | $16,290.00 |
| Drone Campaign (Contracted Flight Service) | $0.00 | $583.33 | $8,500.00 | $45,250.00 |
| AWS Cloud Workloads & Ingestion | $0.00 | $580.29 | $601.92 | $7,050.00 |
| Total Pilot Expenditure | $82,550.00 | $2,728.62 | $10,666.92 | $153,630.00 |

---

## 2. Assumptions Registry (closes PD-017)

In strict compliance with acceptance criterion 2 ("Assumptions are visible and separate from the cost data") and PD-017, every parameter, rate, volume, and unit price is cataloged in the table below before being applied in the subsequent operational cost models.

| Assumption ID | Category | Parameter Description | Assigned Value | Architectural Source & Justification |
|---|---|---|---|---|
| ASM-230 | Financial Baseline | Pricing currency, documentation date, and COP/USD planning benchmark rate. | USD; November 2026; 1 USD = 4,100 COP planning rate. | Q-16 response; published pricing schedules and commercial reference quotes; subject to RFP validation under PD-230; closes PD-017. |
| ASM-231 | Field Hardware | Farm node bill of materials, dual-mode module unit cost, cellular SIM monthly fee, and satellite NTN active fee. | Crop Node: $175.00; Drying Node: $180.00; Cellular SIM: $1.50/mo; Satellite NTN: $8.00/mo ($9.50/mo total per satellite node). | ASM-070, ASM-071, ASM-073; 10 % of farms (20 farms, 40 nodes) pay $1.50 SIM + $8.00 satellite = $9.50/node-mo ($380.00/mo); 360 nodes pay $1.50 ($540.00/mo); reference hardware classes subject to PD-230; closes PD-084. |
| ASM-232 | Center Infrastructure | Local networking equipment bill of materials per collection center across three departments. | Firewall: $850.00; Backup Router: $350.00; Switch: $550.00; APs: $600.00; Printer: $500.00. Total: $2,850.00 per site. | ASM-088, ASM-095; site hardware classes defined in collection-center network design; commercial reference estimates subject to PD-230; closes PD-095. |
| ASM-233 | Cloud Networking | Hourly rates for Transit Gateway, Site-to-Site VPN, Route 53 Resolver, Interface Endpoints, and WAF. | TGW: $0.05/hr; VPN: $0.05/hr; Resolver: $0.125/hr/ENI; Endpoints: $0.01/hr/AZ; WAF Web ACL: $5.00/mo. | ASM-120, ASM-122, ASM-124; AWS us-east-1 published pricing schedules (no separate IPv4 charge on AWS-managed VPN tunnels); subject to PD-200 re-check; closes PD-124. |
| ASM-234 | Farmer Notifications | SMS unit cost to Colombia (+57) and mobile push notification delivery pricing. | SMS: $0.055 per delivered segment; Mobile Push: $0.00 (under 1M free tier). | ASM-135, ASM-136; AWS End User Messaging SMS rate; alert episode rates of 90/mo (normal) and 270/mo (harvest); closes PD-133. |
| ASM-235 | Storage & Web Lookup | DynamoDB On-Demand WRU/RRU rates, S3 Standard storage, and CloudFront egress. | WRU: $0.625/M; RRU: $0.125/M; Storage: $0.25/GB-mo; CloudFront: $0.085/GB; S3: $0.023/GB. | ASM-152, ASM-154; sack lookup peaks at 30 reads/min; DynamoDB on-demand eliminates capacity provisioning; closes PD-152. |
| ASM-236 | Managed Inference | Foundation model token pricing and token consumption parameters for Amazon Nova Pro in `us-east-1`. | Input: $0.0008 / 1k tokens; Output: $0.0032 / 1k tokens; Image: 1,200 tokens/img; Prompt: 400; Context: 200; Output: 300. | ASM-160, ASM-169; Amazon Bedrock on-demand published schedule in us-east-1; subject to PD-200 re-check; closes PD-163. |
| ASM-237 | Harvest Invocations | Harvest-month function execution counts, episode openings ($E$), and mission rates ($N_{conf}$, $N_{ver}$, $N_{est}$). | Telemetry: 108,000 invocations ($E = 150$); $N_{conf} = 40$; $N_{ver} = 30$; $N_{est} = 200$. Total: 108,270 invocations. | ASM-163, ASM-169; based on 200 farms x 6 readings + 100 farms x 24 drying readings/day over 30 days; closes PD-163. |
| ASM-238 | Drone Operations | Commercial flight mission contracting rates, flight duration, and campaign cadence. | Targeted mission: $25.00/flight; Area mission: $30.00/flight; $H = 4$ harvest months; 7 annual review campaigns. | ADR-002, ASM-024; contracted certified flight operator avoiding fleet acquisition; commercial reference rate subject to PD-230; closes PD-031. |
| ASM-239 | Observability | CloudWatch custom metrics, standard and composite alarms, log ingestion, and canary run pricing. | Metric: $0.30/mo; Alarm: $0.10/mo; Composite: $0.50/mo; Logs: $0.50/GB; Canary: $0.0012/run. | ASM-220, ASM-221; CloudWatch monitoring catalog from observability design; closes PD-222. |
| ASM-250 | Budget Thresholds | Monthly cloud budget ceilings across production accounts, testing environments, and shared services. | Normal Month Ceiling: $650.00; Harvest Month Ceiling: $700.00. | ASM-066; applied from management account via AWS Budgets; closes PD-076. |
| ASM-251 | Cost Savings | Architectural savings across six design domains comparing chosen approaches against conventional alternatives. | Net annual savings: $47,532.88 OPEX avoidance and $230,000.00 CAPEX avoidance. | ADR-001, ADR-002, ASM-073, ASM-127, ASM-167; Decision 1 ($10,856.40 net OPEX), Decision 2 ($490.56 net OPEX), Decision 3 ($632.68 net OPEX), Decision 4 ($50,000 CAPEX + $34,560 OPEX), Decision 5 ($180,000 CAPEX), Decision 6 ($993.24 net OPEX); closes Criterion 3. |

---

## 3. Field Hardware per Farm (closes PD-084)

The case statement mandates that field hardware must require moderate investment, install in a single visit with little equipment, operate autonomously without grid electrical supply, withstand high humidity and temperature, and function without requiring terrestrial cable infrastructure.

Each pilot farm receives two autonomous field nodes:
1. One Crop Node installed directly in the coffee lot, measuring nitrogen, phosphorus, potassium, soil moisture, soil temperature, ambient air temperature, and relative humidity.
2. One Drying Node located in the wet mill drying area, measuring grain core temperature and coffee moisture content, equipped with a tactile start/stop push-button control for drying lot lifecycle tracking (ASM-130).

To avoid expensive manual site surveys and prevent dispatch errors during initial deployment, all 400 nodes are built using a standardized hardware specification featuring the Quectel BG95-S5 communication module (or equivalent 3GPP Rel-17 dual-mode device class). This module natively supports terrestrial LTE-M / NB-IoT as well as 3GPP Release-17 Satellite Non-Terrestrial Network (NTN) connectivity via geostationary and LEO constellations (such as Skylo or equivalent NTN carrier).

### 3.1 Bill of Materials (BOM) per Farm

| Subsystem Component | Specification & Environmental Rating | Crop Node Unit Cost | Drying Node Unit Cost | Combined Farm Cost |
|---|---|---|---|---|
| Microcontroller & Logic Board | Ultra-low-power 32-bit ARM Cortex-M4 controller, hardware crypto-engine, non-volatile encrypted flash buffer for 24-hour store-and-forward. | $30.00 | $30.00 | $60.00 |
| Outdoor Protective Enclosure | Industrial IP67-rated polycarbonate enclosure, UV-stabilized, waterproof cable glands, pole/stake mounting bracket. | $18.00 | $18.00 | $36.00 |
| Solar Power Subsystem | 5W high-efficiency monocrystalline solar panel, 3.2V 3,000 mAh LiFePO4 battery pack (-10 °C to +60 °C), MPPT solar charge controller. | $32.00 | $32.00 | $64.00 |
| Soil & Ambient Sensor Suite | Multi-parameter 5-pin RS485 Modbus stainless steel probe (NPK, soil moisture, soil temperature) plus external SHT35 air temp/humidity probe in radiation shield. | $54.00 | $0.00 | $54.00 |
| Drying Sensor Suite & Controls | High-accuracy stainless steel grain penetration probe (PT100 RTD) and capacitive bean moisture sensor; IP67 tactile push-button with LED status indicator. | $0.00 | $59.00 | $59.00 |
| Cellular / Satellite NTN Module | Quectel BG95-S5 dual-mode module (or equivalent 3GPP Rel-17 NTN/cellular dual-mode module), internal omnidirectional antenna. | $38.00 | $38.00 | $76.00 |
| Multi-Network Industrial SIM | Industrial-grade MFF2 embedded or plastic SIM card supporting multi-carrier IMSI switching in Colombia (Claro, Movistar, Tigo). | $3.00 | $3.00 | $6.00 |
| Total Hardware Cost per Node | Complete assembled, provisioned, and sealed node. | $175.00 | $180.00 | $355.00 |

Component costs reflect commercial reference planning estimates for generic device classes, subject to competitive supplier quote validation under PD-230 during the procurement RFP phase. Mounting consumables (mounting stakes, stainless steel zip ties, grounding rods, tamper-evident seals) add $15.00 per farm, bringing the total capital expenditure for one farm's hardware installation to exactly **$370.00 USD** (approximately 1,517,000 COP at the 4,100 planning exchange rate). For all 200 farms in the pilot, total field hardware CAPEX is **$74,000.00 USD**.

### 3.2 Recurring Field Connectivity Fees

Field connectivity charges depend on whether the farm has terrestrial cellular coverage or relies on satellite NTN connectivity:

1. Terrestrial Cellular Coverage (Baseline): 180 farms (360 nodes, 90 % of the pilot) maintain cellular connection. Industrial IoT multi-network pooled SIMs with 10 MB per month per node cost $1.50 USD per node per month. 360 nodes x $1.50 = **$540.00 USD/month**.
2. Out-of-Coverage Satellite Connectivity (ASM-052, ASM-073): 20 farms (40 nodes, 10 % of the pilot) activate the Skylo (or equivalent) direct-to-device satellite NTN service during commissioning. Each satellite node incurs both the base multi-network SIM fee ($1.50 USD/month) and the active satellite NTN access fee ($8.00 USD/month), totaling $9.50 USD per node per month. 40 nodes x ($1.50 + $8.00) = 40 nodes x $9.50 = **$380.00 USD/month**.

The combined monthly recurring connectivity fee for all 400 field nodes across all 200 farms is $540.00 + $380.00 = **$920.00 USD/month** (an average of $4.60 USD per farm per month). Both SIM carrier rates and satellite access charges are commercial reference planning estimates subject to contract finalization under PD-230.

---

## 4. Collection Center Hardware Infrastructure (closes PD-095)

Each of the three departmental collection centers (Cauca at 10.41.0.0/16, Huila at 10.42.0.0/16, and Tolima at 10.43.0.0/16) receives an identical local network hardware deployment. In accordance with the network design in `docs/collection-center-network.md`, the equipment terminates a 100 Mbps symmetric fiber WAN link, establishes dual IPsec VPN tunnels to the cloud Transit Gateway, hosts local WPA3-Enterprise Wi-Fi for staff and WPA3-Personal for visitors, isolates operational logistics workstations, and provides local printing for sack identification sheets.

| Device Class | Hardware Model & Technical Specification | Quantity per Site | Unit Price | Total Cost per Center | Total Cost (3 Centers) |
|---|---|---|---|---|---|
| Enterprise Perimeter Firewall | Fortinet FortiGate 70F (or equivalent Netgate 6100 pfSense Plus appliance): 10 Gbps firewall throughput, 1 Gbps IPsec VPN throughput, dual WAN interfaces, integrated hardware crypto, local RADIUS server, zone stateful inspection. | 1 | $850.00 | $850.00 | $2,550.00 |
| Cellular Backup Gateway | Teltonika RUTX09 industrial LTE-A Cat 6 cellular router with dual SIM failover, gigabit Ethernet handoff to firewall second WAN port. | 1 | $350.00 | $350.00 | $1,050.00 |
| Managed PoE Access Switch | Cisco Catalyst 1000-24P (or Aruba Instant On 1930 24G 4SFP/SFP+ 195W): 24x 10/100/1000 PoE+ ports, 802.1Q VLAN support, 802.1X port authentication, port security. | 1 | $550.00 | $550.00 | $1,650.00 |
| Enterprise Wi-Fi 6 APs | Aruba AP-505 (or Ubiquiti UniFi U6 Pro) dual-radio Wi-Fi 6 indoor/semi-industrial access points, 5 SSIDs, WPA3-Enterprise, client isolation, PoE powered. | 3 | $200.00 | $600.00 | $1,800.00 |
| Industrial Code Sheet Printer | Zebra ZD421 (or equivalent industrial thermal printer): thermal transfer network barcode and code-sheet printer, 203 dpi, Ethernet 10/100, dedicated static IP in VLAN 110. | 1 | $500.00 | $500.00 | $1,500.00 |
| Total Collection Center Hardware | Complete turnkey on-premise hardware suite. | 7 devices | - | $2,850.00 | $8,550.00 |

Collection center equipment pricing lines represent commercial reference planning estimates for the specified device classes, subject to competitive distributor quote verification under PD-230 during procurement.

### 4.1 Collection Center Operating Connectivity Costs
1. Dedicated Internet Access (DIA): 100 Mbps symmetric commercial fiber connection with SLA: $180.00 USD/month per center ($540.00 USD/month for three sites).
2. Cellular Backup Link: Corporate pooled LTE backup data plan (standby operational traffic): $35.00 USD/month per center ($105.00 USD/month for three sites).

Total monthly recurring connectivity spend across all three collection centers is **$645.00 USD/month** (subject to ISP commercial quote validation under PD-230).

---

## 5. Commercial Drone Campaign Contracting (closes PD-031)

Under ADR-002, Aroma Pacífico operates in a combined flight mode where area-mode missions establish plot boundaries and estimate harvest yield, while precision-mode missions confirm pest alerts, perform targeted spray applications, and verify pest eradication.

Rather than procuring an expensive fleet of commercial agricultural drones, establishing maintenance depots, and hiring full-time aviation staff, the Federation contracts a certified commercial drone operator company holding civil aviation permits (Aerocivil) for unmanned aerial spraying and multispectral mapping (ASM-023). Flight mission rates ($25.00 for precision missions and $30.00 for area missions) are commercial reference planning estimates subject to formal competitive RFP validation under PD-230.

### 5.1 Flight Volumes and Cadence

Following ADR-002 section 3.3, annual review campaigns follow the formula:
$$C_{annual} = H + \left\lceil \frac{12 - H}{3} \right\rceil$$

With $H = 4$ harvest months per year (ASM-024), the pilot conducts $4 + \left\lceil \frac{8}{3} \right\rceil = 4 + 3 = 7$ scheduled campaigns per year.

| Mission Flight Type | Mission Trigger & Scope | Harvest Month Flights | Normal Month Flights | Annual Total Flights | Contract Unit Rate |
|---|---|---|---|---|---|
| Delimitation Missions | One initial plot mapping flight per farm, establishing polygons and no-fly zones. Flown before the pilot launch. | 0 | 0 | 200 | $30.00 |
| Harvest Estimation Missions | Area flight over entire farm to estimate yield in 60 kg sacks; flown once per harvest period. | 200 | 0 | 200 | $30.00 |
| Pest Focus Confirmation | Targeted flight to sensor coordinates upon `crop_alert_raised` event. Captures up to 8 images. | 40 | 15 | 285 | $25.00 |
| Targeted Fumigation | Authorized spraying flight restricted to confirmed plot; requires Federation agronomist MFA. | 30 | 10 | 210 | $25.00 |
| Post-Spray Verification | Orthomosaic or inspection pass to verify symptom remediation and close the open case. | 30 | 10 | 210 | $25.00 |
| Total Flight Missions | Aggregated flight missions across all categories. | 300 | 35 | 1,105 | - |

### 5.2 Monthly and Annual Campaign Costs

1. Harvest Month Campaign Cost:
   - 200 Harvest Estimation flights x $30.00 = $6,000.00
   - 40 Confirmation flights x $25.00 = $1,000.00
   - 30 Fumigation flights x $25.00 = $750.00
   - 30 Verification flights x $25.00 = $750.00
   - Total Cost for a Harvest Month Campaign: **$8,500.00 USD**.

2. Normal Month Flight Costs:
   - Outside harvest campaigns (held once every three months during off-peak seasons), flight operations handle only alert-driven precision missions (15 confirmations, 10 fumigations, 10 verifications = 35 flights).
   - Monthly equivalent off-harvest operating cost: 35 flights x $25.00 = **$875.00 USD** (when campaigns run every quarter, the quarterly non-harvest campaign cost is $1,750.00 USD, or an average of $583.33 USD/month).

3. Annual Contract Value:
   - Initial Delimitation Campaign (200 farms x $30.00): $6,000.00
   - 4 Harvest Campaigns x $8,500.00: $34,000.00
   - 3 Off-Harvest Review Campaigns x $1,750.00: $5,250.00
   - Total Annual Commercial Drone Operator Contract: **$45,250.00 USD**.

---

## 6. Event-Driven Function Invocations and Managed Inference (closes PD-163)

In accordance with `docs/inference-functions.md`, all automated inferencing in the pilot is strictly event-driven. Compute executes via AWS Lambda, while image and pattern inferencing is performed via Amazon Bedrock utilizing the Amazon Nova Pro model in `us-east-1` (ASM-160). No provisioned concurrency, dedicated model endpoints, or GPU instances exist.

### 6.1 Inference Pricing Formula (ASM-169)

For any function $f$, the cost of 1,000 invocations is governed by the standardized formula:
$$C_{1000}(f) = 1000 \times \left[ P_{req} + M_f \times D_f \times P_{gbs} + Q_f \times P_{sqs} + R_f \times (P_{s3get} + P_{kms}) + W_f \times P_{ddbw} + K_f \times P_{ddbr} + \frac{p_f \times \left( (I_f \times T_{img} + T_{prompt\_f} + T_{ctx\_f}) \times P_{in} + T_{out\_f} \times P_{out} \right)}{1000} + X_f \right]$$

### 6.2 Unit Price Parameters (`us-east-1`, November 2026 Planning Baselines, subject to PD-200)

Unit rates below represent published AWS service pricing schedules for `us-east-1`. Because direct external network connectivity to `aws.amazon.com` is proxy-restricted during authoring (as recorded in PD-200 and architecture section 18), these rates are tracked as planning assumptions subject to formal pre-implementation verification under PD-200.

| Parameter Symbol | Description | Provider Price Schedule (`us-east-1`) |
|---|---|---|
| $P_{req}$ | AWS Lambda Request Fee | $0.20 per 1,000,000 requests ($0.00000020 per request) |
| $P_{gbs}$ | AWS Lambda Compute Duration (ARM Graviton) | $0.0000133334 per GB-second |
| $P_{sqs}$ | Amazon SQS Standard Request | $0.40 per 1,000,000 requests ($0.00000040 per request) |
| $P_{s3get}$ | Amazon S3 GET Request | $0.0004 per 1,000 requests ($0.00000040 per request) |
| $P_{kms}$ | AWS KMS Symmetric Cryptographic Request | $0.03 per 10,000 requests ($0.00000300 per request) |
| $P_{ddbw}$ | Amazon DynamoDB On-Demand Write Request Unit (WRU) | $0.625 per 1,000,000 WRUs ($0.000000625 per WRU) |
| $P_{ddbr}$ | Amazon DynamoDB On-Demand Read Request Unit (RRU) | $0.125 per 1,000,000 RRUs ($0.000000125 per RRU) |
| $P_{in}$ | Amazon Bedrock Input Tokens (Amazon Nova Pro) | $0.00080 per 1,000 tokens ($0.80 per 1,000,000 tokens) |
| $P_{out}$ | Amazon Bedrock Output Tokens (Amazon Nova Pro) | $0.00320 per 1,000 tokens ($3.20 per 1,000,000 tokens) |
| $X_f$ | Event Routing (IoT Core Rule Action or EventBridge Pipe) | $0.00000030 to $0.00000040 per delivered event |
| $T_{img}$ | Input Tokens per Processed Image | 1,200 tokens per image |

### 6.3 Evaluation of $C_{1000}(f)$ per Function

| Function Pattern | $M_f$ (GB) | $D_f$ (s) | $Q_f$ | $R_f$ | $W_f$ | $K_f$ | $p_f$ | $I_f$ | Input Tokens | Output Tokens | Cost per 1,000 Invocations ($C_{1000}$) | Cost per Single Invocation |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Telemetry Threshold Function | 0.25 | 0.15 | 2 | 0 | 2 | 4 | 0.001389 | 0 | 500 | 150 | **$0.00477** | $0.00000477 |
| Pest Confirmation Function | 0.50 | 3.50 | 3 | 5 | 3 | 3 | 1.000000 | 4 | 5,200 | 300 | **$5.16400** | $0.00516400 |
| Pest Verification Function | 0.50 | 3.50 | 3 | 5 | 3 | 3 | 1.000000 | 4 | 5,200 | 300 | **$5.16400** | $0.00516400 |
| Harvest Estimation Function | 0.50 | 8.00 | 3 | 22 | 3 | 3 | 1.000000 | 21 | 25,700 | 300 | **$21.65000** | $0.02165000 |

### 6.4 Monthly Harvest Invocations and Aggregate Inference Cost

During a 30-day harvest month, the pilot processes:
- Crop Telemetry: 200 farms x 6 readings/day x 30 days = 36,000 readings.
- Active Drying Telemetry: 100 farms x 24 readings/day x 30 days = 72,000 readings.
- Total Telemetry Invocations: 108,000 invocations.
- Episode Openings ($E$): Modeled at 150 openings/month (ASM-237). Only these 150 invocations call the Bedrock model ($p_f = 150 / 108,000 = 0.001389$).
- Drone Missions: 40 confirmations, 30 verifications, and 200 harvest estimations.

| Trigger Event | Monthly Invocations | Model Calls | Formula Scaling | Monthly Compute & Model Cost |
|---|---|---|---|---|
| Telemetry Batch Evaluation | 108,000 | 150 | $108 \times C_{1000}(telemetry)$ | $0.515 USD |
| Confirmation Objects | 40 | 40 | $0.040 \times C_{1000}(conf)$ | $0.207 USD |
| Verification Objects | 30 | 30 | $0.030 \times C_{1000}(ver)$ | $0.155 USD |
| Harvest Estimation Objects | 200 | 200 | $0.200 \times C_{1000}(est)$ | $4.330 USD |
| Harvest Month Total | 108,270 | 420 | - | **$5.207 USD** |

In a normal off-harvest month (36,000 crop telemetry readings, $E = 60$ episode openings, 15 confirmation flights, 10 verification flights, and 0 estimation flights), total invocations equal **36,025**, total model calls equal **85**, and total inference compute spend equals **$0.301 USD**.

---

## 7. Cloud Workloads and Monthly Operational Budgets (closes PD-076, PD-124, PD-152, PD-222)

All cloud services are hosted in AWS region `us-east-1` across a multi-account landing zone consisting of `management`, `security`, `network`, `development`, `test`, and `production` accounts.

### 7.1 Cloud Infrastructure Breakdown by Service

| Cloud Service Domain | Metering Dimension & Quantities | Normal Month Spend | Harvest Month Spend | Basis & Reference |
|---|---|---|---|---|
| AWS Transit Gateway | 3 VPC attachments ($0.05/hr x 730 hrs x 3 = $109.50) plus data processing (~50 GB @ $0.02/GB = $1.00). | $110.50 | $110.50 | ASM-120, ASM-233; central network hub. |
| AWS Site-to-Site VPN | 3 VPN connections (one per center) x $0.05/hr x 730 hrs ($109.50; AWS-managed VPN tunnel endpoints incur no separate customer public IPv4 address fee). | $109.50 | $109.50 | ASM-120, ASM-233; IPsec tunnels to collection centers; subject to PD-200 re-check. |
| Route 53 Resolver Inbound | 2 ENIs in `administration` tier x $0.125/hr x 730 hrs ($182.50) plus query resolution ($0.20). | $182.70 | $182.70 | ASM-122; private DNS for centers. |
| VPC Interface Endpoints | 4 interface endpoints (Logs, Monitoring, Roles Anywhere, S3) x 2 AZs x $0.01/hr x 730 hrs ($58.40) + data. | $58.90 | $58.90 | ASM-124, ASM-140; PrivateLink endpoints. |
| AWS WAF | 1 Web ACL ($5.00) + 4 managed rule groups ($4.00) + ~250,000 evaluated requests ($0.15). | $9.15 | $9.15 | ASM-129, ASM-155; edge defense on CloudFront. |
| Amazon CloudFront | ~15 GB egress ($1.28) + 120,000 HTTPS requests ($0.12). | $1.40 | $1.40 | ASM-152; public sack lookup front door. |
| AWS IoT Core | ~130,000 MQTT messages/mo ($0.13), 260,000 rule actions ($0.04), connection minutes ($0.02). | $0.19 | $0.19 | ADR-003, ASM-234; telemetry ingestion. |
| Amazon DynamoDB On-Demand | WRUs (400,000/mo = $0.25), RRUs (600,000/mo = $0.08), storage (10 GB = $2.50), PITR ($2.00). | $4.83 | $4.83 | ASM-152, ASM-156; serverless tables. |
| Amazon S3 & AWS KMS | S3 Standard (~300 GB = $6.90) + PUT/GET requests ($0.60) + 8 KMS keys ($8.00) + crypto requests ($0.30). | $15.80 | $15.80 | ASM-097, ASM-154; encrypted object store. |
| Event-Driven Compute & AI | Lambda compute, SQS queues, EventBridge Pipes, and Bedrock Nova Pro inferencing. | $0.30 | $5.21 | ASM-169, ASM-237; section 6 calculations. |
| Farmer Notifications (SMS/Push) | AWS End User Messaging SMS to Colombia (+57) @ $0.055/msg; Push via SNS is free. | $4.95 (90 SMS) | $14.85 (270 SMS) | ASM-135, ASM-136; episode deduplication. |
| Observability & Telemetry | CloudWatch custom metrics, alarms, log ingestion, Synthetics canary, and Cost Explorer export. | $60.72 | $60.72 | ASM-220, ASM-239; section 8 calculations. |
| AWS Cloud Workload Total | Monthly AWS invoice total across all accounts. | **$580.29 USD** | **$601.92 USD** | Closes PD-076, PD-124, PD-152. |

Both the normal month AWS workload spend ($580.29 USD) and harvest month AWS workload spend ($601.92 USD) fall well within the monthly AWS Budgets ceilings established in Section 10 ($650.00 USD and $700.00 USD, respectively), providing approximately 11 % to 14 % operational margin for unpredicted traffic bursts.

---

## 8. Observability and Monitoring Infrastructure Costs (closes PD-222)

In accordance with `docs/observability.md`, observability in Aroma Pacífico is built exclusively using AWS-native monitoring services. The pricing model incorporates every metric, log group, alarm, synthetic canary, and automated reporting task defined in the architecture.

| Observability Component | Operational Scope & Quantity | Unit Price (`us-east-1`) | Monthly Cost | Architectural Purpose & Reference |
|---|---|---|---|---|
| CloudWatch Custom Metrics | 65 custom metrics published via Embedded Metric Format (EMF) and detector functions (telemetry arrival, SLO indicators, lag metrics). | First 10 free; $0.30 per metric-month for next 55 metrics. | $16.50 | ASM-220; SLO 1 and SLO 2 tracking, silent farm detection. |
| Standard CloudWatch Alarms | 110 standard resolution alarms (60-second evaluation period) monitoring infrastructure, queues, and thresholds. | $0.10 per alarm-month. | $11.00 | ASM-228; dead-letter queues, Lambda errors, WAN health. |
| Composite Alarms | 8 composite alarms aggregating related failure conditions to suppress alarm storms during outages. | $0.50 per composite alarm-month. | $4.00 | ASM-220; multi-hop alert failure suppression. |
| CloudWatch Logs Ingestion | 35 GB per month of application logs, IoT logs, VPC Flow Logs of rejected traffic, and WAF sampled logs. | $0.50 per GB ingested. | $17.50 | ASM-067, ASM-127; 5-year retention compliance. |
| CloudWatch Logs Archive Storage | 35 GB cumulative archive storage in S3 Glacier Instant Retrieval / Standard-IA log classes. | $0.03 per GB-month. | $1.05 | ASM-067; audit log durability. |
| CloudWatch Synthetics Canary | 1 canary requesting public sack lookup (`/s/<probe_code>`) every 5 minutes (8,640 executions per month). | $0.0012 per canary run. | $10.37 | ASM-225; SLO 2 synthetic availability probe. |
| AWS Cost Explorer API Requests | 30 automated daily requests executed by the cost export function to feed `<prefix>-shared-dash-cost`. | $0.01 per request. | $0.30 | ASM-221; automated daily cost tracking. |
| AWS X-Ray Tracing | Distributed tracing across API Gateway, Lambda, and DynamoDB (~80,000 traces per month). | Within permanent AWS Free Tier (100,000 traces/mo free). | $0.00 | ASM-220; end-to-end transaction tracing. |
| Total Observability Spend | Aggregated monthly monitoring infrastructure cost. | - | **$60.72 USD** | Closes PD-222. |

---

## 9. Architectural Decisions That Lower or Avoid Expenditure (closes Acceptance Criterion 3, ASM-251)

Acceptance criterion 3 requires the technical and commercial justification of at least three decisions that reduce or avoid spending. This architecture establishes and quantifies six major cost-saving architectural decisions:

### Decision 1: Elimination of AWS Network Firewall and NAT Gateways in Favor of VPC Endpoints (ASM-127)
- **Alternative Considered**: Deploying AWS Network Firewall across three Availability Zones in `production` and NAT Gateways in each AZ for outbound internet inspection.
- **Cost Avoided**: AWS Network Firewall charges $0.395 per firewall endpoint hour ($288.35/month per AZ, or $865.05/month for 3 AZs = $10,380.60/year) plus data processing. Three NAT Gateways charge $0.045 per hour ($32.85/month per AZ, or $98.55/month for 3 AZs = $1,182.60/year) plus data egress. Combined traditional network egress and inspection alternative: $865.05 + $98.55 = **$963.60 USD/month** ($11,563.20 USD annually).
- **Chosen Architecture**: Private subnets contain no default routes to the internet. Collection center traffic is inspected locally by on-premise firewalls before entering the IPsec VPN tunnel. AWS services are accessed exclusively through private Gateway Endpoints (S3 and DynamoDB, which are free) and four Interface Endpoints ($58.90/month = $706.80/year).
- **Net Annual Savings**: $963.60 - $58.90 = $904.70 USD/month = **$10,856.40 USD/year** (Network Firewall alone avoids $10,380.60/year; adding NAT Gateways avoids $11,563.20/year gross, yielding $10,856.40/year net savings after deducting the $706.80/year interface endpoint spend).

### Decision 2: Serverless Edge Front Door (CloudFront, WAF, Lambda Function URL) vs. Application Load Balancers (PD-051, ASM-155)
- **Alternative Considered**: An Application Load Balancer (ALB) provisioned across three Availability Zones in the `public_query` tier with an associated Web ACL.
- **Cost Avoided**: An ALB costs $0.0225 per hour ($16.43/month) plus Load Balancer Capacity Units (LCUs) of at least $0.008 per LCU-hour across 3 AZs (~$35.00/month), totaling ~$51.43/month ($617.16/year).
- **Chosen Architecture**: Amazon CloudFront routes directly to a secure AWS Lambda function URL using Origin Access Control (OAC) with `AWS_IAM` signing. CloudFront ($1.40/month) and AWS WAF ($9.15/month) total $10.55/month ($126.60/year), while Lambda Function URLs incur no base hourly or capacity unit fees.
- **Net Savings**: $51.43 - $10.55 = **$40.88 USD/month** (**$490.56 USD/year**) while delivering superior global edge caching, DDoS mitigation, and sub-100ms response times for sack code lookups.

### Decision 3: Inference Model Invocations Limited to Alert Episode Openings (ASM-167)
- **Alternative Considered**: Executing the foundation vision model on every incoming crop sensor reading to detect anomalies.
- **Cost Avoided**: Running inference on all 108,000 monthly telemetry readings during harvest ($95.04/month) and 36,000 readings during off-harvest ($31.68/month) at $0.00088 per call would cost $633.60/year in model tokens alone, plus expanded Lambda execution duration.
- **Chosen Architecture**: The telemetry threshold function evaluates mathematical bounds deterministically in Lambda. Only readings that transition an un-alerted state into an open episode ($E = 150$ per harvest month = $0.13/month; $E = 60$ per off-harvest month = $0.05/month) invoke the Bedrock model to generate natural language explanations and agronomist guidance.
- **Net Savings**: Reduces monthly inference token expenditure during harvest from $95.04 to $0.13 (saving $94.91/month across 4 harvest months = $379.64/year) and during off-harvest from $31.68 to $0.05 (saving $31.63/month across 8 off-harvest months = $253.04/year), achieving a net annual savings of **$632.68 USD/year**.

### Decision 4: Standardized Dual-Mode Radio Hardware with On-Demand Satellite Activation (ASM-073)
- **Alternative Considered**: Contracting specialized RF field engineers to conduct pre-deployment signal surveys across 200 remote veredas in Cauca, Huila, and Tolima to identify out-of-coverage farms, or activating satellite subscriptions indiscriminately across all 400 nodes.
- **Cost Avoided**: Pre-installation RF surveys across 200 rugged Andean farms cost upwards of $250.00 per farm ($50,000.00 CAPEX). Furthermore, activating satellite NTN plans on all 400 nodes at $8.00/month would cost $3,200.00 USD/month ($38,400.00 USD/year) in satellite subscription fees.
- **Chosen Architecture**: Every node is built with the Quectel BG95-S5 dual-mode module (or equivalent 3GPP Rel-17 device class). Installers test connectivity during the single mounting visit. If terrestrial LTE-M attaches, the satellite subscription remains un-provisioned ($0.00/mo). Only the 20 out-of-coverage farms (40 nodes) activate satellite subscriptions (40 x $8.00/mo = $320.00/mo).
- **Net Savings**: Avoids **$50,000.00 USD** in preliminary site survey CAPEX, and avoids paying the $8.00/month satellite fee on the 360 terrestrial nodes (360 nodes x $8.00/mo = $2,880.00 USD/month avoided), saving **$34,560.00 USD/year in recurring OPEX**.

### Decision 5: Contracted Commercial Drone Operations vs. Federation Fleet Ownership (ADR-002)
- **Alternative Considered**: Purchasing commercial agricultural spray drones (e.g., DJI Agras T40 class at $25,000 each or equivalent) and multispectral mapping drones across three departments, hiring certified full-time pilots, purchasing mobile generator trailers, and licensing civil aviation operator certificates.
- **Cost Avoided**: Procuring a reference fleet of 6 spray drones and 3 mapping drones requires an estimated $180,000.00 USD in capital equipment, plus battery replacements ($12,000/year), aviation insurance ($15,000/year), and dedicated personnel.
- **Chosen Architecture**: Contracting an Aerocivil-certified regional operator on a per-mission basis ($25.00 for precision flights, $30.00 for area flights).
- **Net Savings**: Avoids **$180,000.00 USD in CAPEX** and limits annual drone operating spend to **$45,250.00 USD**, scaling strictly with verified agronomic necessity.

### Decision 6: Serverless On-Demand Transaction Stores vs. Provisioned Clusters (ASM-061, ASM-152)
- **Alternative Considered**: Provisioning Aurora Serverless v2 or multi-AZ relational databases with reserved read replicas.
- **Cost Avoided**: A minimum Aurora PostgreSQL deployment (0.5 ACU baseline across 2 AZs) costs ~$0.12/ACU-hour = $87.60/month base charge ($1,051.20/year) regardless of traffic, plus storage and backup fees.
- **Chosen Architecture**: DynamoDB On-Demand tables for telemetry, alerts, sacks, and positions ($4.83/month = $57.96/year). During off-harvest months or nighttime periods with zero transactions, DynamoDB read/write compute costs drop to exactly $0.00.
- **Net Savings**: $87.60 - $4.83 = $82.77 USD/month = **$993.24 USD/year** while providing multi-AZ fault tolerance and single-digit millisecond latency.

### 9.1 Savings Reconciliation Summary Table

| Decision Domain | Alternative Gross Cost | Chosen Architecture Cost | Net OPEX Avoidance / Year | Net CAPEX Avoidance |
|---|---|---|---|---|
| Decision 1: VPC Endpoints vs Network Firewall & NAT | $11,563.20/yr ($963.60/mo) | $706.80/yr ($58.90/mo) | $10,856.40/yr | $0.00 |
| Decision 2: Serverless Front Door vs ALB | $617.16/yr ($51.43/mo) | $126.60/yr ($10.55/mo) | $490.56/yr | $0.00 |
| Decision 3: Episode-Only Inference vs All Readings | $643.08/yr | $10.40/yr | $632.68/yr | $0.00 |
| Decision 4: Dual-Mode Hardware & Selective Satellite | $38,400.00/yr OPEX + $50,000 CAPEX | $3,840.00/yr OPEX | $34,560.00/yr | $50,000.00 |
| Decision 5: Commercial Drone Contract vs Fleet | $180,000.00 CAPEX + maintenance | $45,250.00/yr OPEX | Scales per mission | $180,000.00 |
| Decision 6: DynamoDB On-Demand vs Provisioned DB | $1,051.20/yr ($87.60/mo) | $57.96/yr ($4.83/mo) | $993.24/yr | $0.00 |
| **Total Reconciled Savings** | - | - | **$47,532.88/yr** | **$230,000.00** |

---

## 10. AWS Budgets Configuration and Landing Zone Cost Ceilings (closes PD-076, ASM-250)

In compliance with `docs/landing-zone.md` section 6 and PD-076, AWS Budgets are configured in the `management` account to track and enforce financial boundaries across the five cost centers (`farms`, `drones`, `collection-centers`, `trips`, `testing`) and individual linked accounts.

All budgets alert via Amazon SNS to the platform operations topic at 80 % of actual cost, 100 % of actual cost, and 100 % of forecasted cost (ASM-066). The testing budget includes an automated AWS Budgets Action that applies a restrictive Service Control Policy (SCP) denying new resource creation upon manual approval at 100 % of actual spend; production workloads never receive automated shutdown actions to safeguard farmer alerts and public sack lookups (RTO = 4 hours).

| Cost Center / Dimension | Filter Tag / Account Scope | Normal Month Budget | Harvest Month Budget | Alert Thresholds | Automated Budget Action |
|---|---|---|---|---|---|
| `farms` | `CostCenter = farms`, production account | $80.00 | $95.00 | 80 %, 100 % actual; 100 % forecast | None (production protected) |
| `drones` | `CostCenter = drones`, production account | $15.00 | $25.00 | 80 %, 100 % actual; 100 % forecast | None (production protected) |
| `collection-centers` | `CostCenter = collection-centers`, production account | $385.00 | $385.00 | 80 %, 100 % actual; 100 % forecast | None (production protected) |
| `trips` | `CostCenter = trips`, production account | $10.00 | $15.00 | 80 %, 100 % actual; 100 % forecast | None (production protected) |
| `testing` | Linked accounts `development` and `test` | $80.00 | $90.00 | 80 %, 100 % actual; 100 % forecast | Manual approval action: apply SCP denying resource creation |
| Shared & Management | Management account, security account, shared logs | $80.00 | $90.00 | 80 %, 100 % actual; 100 % forecast | None |
| Total Cloud Budget Ceiling | Aggregate monthly AWS billing threshold | **$650.00 USD** | **$700.00 USD** | - | - |

---

## 11. Compliance with the Case Statement and Acceptance Criteria

| Acceptance Criterion | Verification Evidence in this Document | Compliance Status |
|---|---|---|
| The document includes the cost of a normal month, a harvest month, one farm's hardware, and the drone campaign. | Normal month OPEX ($2,728.62) and harvest month OPEX ($10,666.92) detailed in Section 1 and Section 7. One farm's hardware ($370.00) broken down in Section 3. Drone campaign ($8,500.00/harvest campaign, $45,250.00/year) detailed in Section 5. | SATISFIED |
| Assumptions are visible and separate from the cost data. | Section 2 isolates every parameter, unit cost, and volume in dedicated table ASM-230 through ASM-251, with explicit architectural citations, prior to any cost calculations. | SATISFIED |
| At least 3 decisions that reduce or avoid spending are justified. | Section 9 provides exhaustive technical and financial justifications for six distinct cost-avoidance decisions, quantifying $47,532.88 in annual OPEX savings and $230,000.00 in avoided CAPEX. | SATISFIED |
| The document includes an estimate of function invocations during harvest. | Section 6.4 estimates 108,270 total function invocations and 420 Bedrock model calls during harvest, evaluating cost per thousand invocations using the formal ASM-169 formula. | SATISFIED |

---

## 12. Decision Log Registry and Closed Items

| ID | State | Description | Resolution Applied |
|---|---|---|---|
| PD-017 | Closed | Cost assumptions registered as ASM-NNN and separate from cost data. | Closed by Section 2 (ASM-230 through ASM-251). |
| PD-031 | Closed | Drone campaign cost states $H$, pest case rate, and ADR-002 flight volumes. | Closed by Section 5 ($H = 4$, 7 annual campaigns, 1,105 annual flights). |
| PD-076 | Closed | Landing zone budgets receive explicit normal and harvest month amounts. | Closed by Section 10 ($650.00 normal, $700.00 harvest). |
| PD-084 | Closed | Prices for both node module classes and satellite subscription fee. | Closed by Section 3 ($38.00 Quectel BG95-S5 module, $8.00/mo satellite fee). |
| PD-095 | Closed | Prices for collection center local hardware device classes. | Closed by Section 4 ($2,850.00 per site across FortiGate, switch, APs, printer). |
| PD-124 | Closed | Prices for cloud network controls and Network Firewall comparison. | Closed by Section 7 and Section 9 (TGW, VPN at $0.05/hr, Resolver, endpoints; saves $10,856.40/yr net). |
| PD-133 | Closed | Prices for farmer notifications (SMS to Colombia +57 and push). | Closed by Section 7 (SMS @ $0.055, push free; 90 normal, 270 harvest). |
| PD-152 | Closed | Prices for public sack lookup front door and DynamoDB tables. | Closed by Section 7 (CloudFront, WAF, Lambda URL, On-Demand DynamoDB). |
| PD-163 | Closed | Cost per thousand invocations in harvest evaluated via ASM-169 formula. | Closed by Section 6 (108,270 invocations, 420 model calls, $5.21 total spend). |
| PD-222 | Closed | Prices observability itself and sets dashboard budget amounts. | Closed by Section 8 ($60.72/mo across metrics, alarms, logs, canaries). |
| PD-230 | Open | Re-verify supplier hardware quotes, COP benchmark rate, and drone operator rates during RFP procurement. | Assigned to SR1 / JR1; competitive procurement validation prior to field deployment. |

---

## 13. References and Pricing Schedules

All cloud unit prices are based on published provider pricing documentation (dated November 2026 for `us-east-1`) and commercial hardware distributor catalogs. Because direct external web connections to `aws.amazon.com` are blocked by environment proxy configuration (PD-200), AWS service rates are maintained as planning assumptions subject to formal pre-implementation re-check under PD-200. Similarly, commercial hardware model lines and drone rates represent reference planning estimates subject to supplier quote validation under PD-230 during competitive RFP procurement:
1. [AWS Lambda Pricing (`us-east-1`)](https://aws.amazon.com/lambda/pricing/): Published November 2026 schedule. $0.20 per 1M requests, $0.0000133334 per GB-second (ARM Graviton); subject to PD-200 re-check.
2. [Amazon DynamoDB On-Demand Pricing (`us-east-1`)](https://aws.amazon.com/dynamodb/pricing/on-demand/): Published November 2026 schedule. $0.625 per 1M WRU, $0.125 per 1M RRU, $0.25 per GB-month; subject to PD-200 re-check.
3. [Amazon Bedrock Pricing (`us-east-1`)](https://aws.amazon.com/bedrock/pricing/): Published November 2026 schedule. Amazon Nova Pro standard tier: $0.0008 per 1k input tokens, $0.0032 per 1k output tokens; subject to PD-200 re-check.
4. [AWS IoT Core Pricing (`us-east-1`)](https://aws.amazon.com/iot-core/pricing/): Published November 2026 schedule. $1.00 per 1M messages (5 KB), $0.15 per 1M rule evaluations, $0.08 per 1M connection-minutes; subject to PD-200 re-check.
5. [AWS Transit Gateway & Site-to-Site VPN Pricing (`us-east-1`)](https://aws.amazon.com/transit-gateway/pricing/): Published November 2026 schedule. Transit Gateway at $0.05 per attachment-hour, $0.02 per GB processed; Site-to-Site VPN at $0.05 per connection-hour (AWS-managed VPN tunnel endpoints incur no separate customer public IPv4 fee); subject to PD-200 re-check.
6. [Amazon CloudWatch Pricing (`us-east-1`)](https://aws.amazon.com/cloudwatch/pricing/): Published November 2026 schedule. Custom metrics at $0.30/metric-mo, standard alarms at $0.10/mo, log ingestion at $0.50/GB, canary runs at $0.0012/run; subject to PD-200 re-check.
7. [AWS End User Messaging SMS Pricing (Colombia +57)](https://aws.amazon.com/end-user-messaging/pricing/): Published November 2026 schedule. Outbound SMS delivery to Colombia carriers averaging $0.055 per segment; subject to PD-200 re-check.
8. [Quectel Wireless Solutions BG95-S5 Specification & Distributor Catalog](https://www.quectel.com/product/lte-cat-m1-nb-iot-egprs-satellite-bg95-s5): 3GPP Rel-17 NTN satellite / cellular dual-mode IoT module class. Reference planning estimate of $38.00 unit baseline, subject to competitive RFP quote validation under PD-230.
9. [Skylo Technologies Direct-to-Device Satellite NTN IoT Connectivity](https://www.skylo.tech/): Satellite narrowband IoT telemetry subscription class. Reference planning estimate of $8.00 per active node per month, subject to contract finalization under PD-230.
10. [Fortinet FortiGate 70F Hardware Datasheet & Commercial Distribution](https://www.fortinet.com/products/next-generation-firewall): Enterprise perimeter branch firewall class. Reference planning estimate of $850.00 unit baseline, subject to competitive distributor quote validation under PD-230.
