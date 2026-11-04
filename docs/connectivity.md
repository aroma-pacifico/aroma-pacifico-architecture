# End-to-End Connectivity

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / JR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#10 (A10) |
| Builds on | [ADR-001](adr/adr-001-primary-and-second-cloud.md), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md), [`docs/legacy-2020.md`](legacy-2020.md) (A05, rows L-09 to L-13), [`docs/addressing-plan.md`](addressing-plan.md) (A07) |
| Feeds | A11 collection-center LAN (aroma-pacifico/aroma-pacifico-roadmap#11), A12 firewall policy table (aroma-pacifico/aroma-pacifico-roadmap#12), A13 and A14 ingestion and alerts (aroma-pacifico/aroma-pacifico-roadmap#13, #14), A21 SLOs (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36) |
| Decision-log entries | PD-010 (closed here), ASM-070 to ASM-079, PD-080 to PD-085 |

## 1. Purpose and scope

Point 3 of the [case statement](case/case-statement.pdf) requires that "El diseño de conectividad cubre finca, dron, transportador, centro de acopio y puerto, con el comportamiento sin señal. No va en un anexo." The section "Qué se actualiza" adds that each flow states its exit protocol, its local buffer and what happens when there is no signal ("Cada uno indica protocolo de salida, buffer local y qué pasa cuando no hay señal"). This document closes those flows for the farm, the wet mill, the drone, the transporter, the collection center and the port, in the order the case statement lists them, and gives for each one the path to its cloud endpoint.

It replaces the two connectivity assumptions of 2020 that the legacy record marks for this task: data coverage on most farms (row L-09) and a device publishing straight to the ingestion point (row L-10). It closes the port flow (row L-12) and retires any dependence on a mobile network generation (row L-13). It does not design the collection-center LAN, which belongs to A11, nor the firewall rules, which belong to A12; for the centers it covers only the WAN egress and the link to the `transit` tier of the addressing plan. Firmware, application logic and device models are out of scope, as the case statement says; devices are specified as classes, enough to cost them and to say how they receive orders and return records.

Quotations are in the original Spanish, which prevails over the English rendering (ASM-001). No figure in this document is a measurement. Latencies are design targets, to be validated by the alert-delivery SLO of A21; coverage is never quantified.

## 2. Design principles

Five rules apply to every flow, and each section below is an instance of them.

| Rule | Statement | Source |
|---|---|---|
| Identity, not address | Field devices and phones hold no cloud private address and are authenticated by their identity: an X.509 certificate per farm node, a human identity per app user. Adding farms adds identities, never subnets or routes. | ADR-003 (device identity, ingestion endpoint); addressing plan section 5; ASM-032 |
| Bearer neutrality | Every field flow needs only an IP bearer that carries TLS. No flow depends on a specific mobile generation, circuit-switched data or a voice channel (section 11). | Case statement, "La red de acceso móvil cambió desde 2020" |
| Original timestamp and idempotency key | Every buffered record carries the time it was produced and a key that makes a second delivery harmless: node identifier and reading timestamp for telemetry, a client-generated event identifier for app events. A drained record is evaluated as if it had arrived live. | ADR-003 (telemetry store key); ASM-011 |
| Release on acknowledgement | A record leaves a local buffer only when the cloud has acknowledged it (MQTT PUBACK, HTTP 2xx after persistence, or a verified object checksum). A lost acknowledgement causes a duplicate, which the idempotency key absorbs, never a loss. | This document |
| Encryption end to end | TLS 1.2 or later from the device or app to the AWS endpoint; no gateway, relay or third party terminates TLS or sees a reading, object or event in clear. Buffers on devices and phones are encrypted at rest. | Case statement point 2, "Telemetría de finca y objetos del dron no se publican en claro" |

## 3. Summary of the flows

| Flow | Actor and device | Exit protocol | Local buffer | Without signal | Cloud endpoint and tier |
|---|---|---|---|---|---|
| Farm (crop zone) | Crop node, autonomous in energy (ASM-070) | MQTT 3.1.1 or 5 over TLS, X.509 client certificate, QoS 1, port 8883 (443 with ALPN as alternative) | Non-volatile ring buffer, retention 24 hours (ASM-074) | Store-and-forward, drain oldest first with original timestamps | AWS IoT Core regional data endpoint in `us-east-1`, public; `telemetry_ingress` reserved for its interface endpoint |
| Farm out of coverage | Same crop and drying nodes with a dual-mode cellular and satellite module (ASM-073) | Same MQTT over TLS, over a satellite IP bearer | Same buffer | Readings wait for the next satellite contact, within 24 hours | Same IoT Core endpoint and topic |
| Wet mill | Drying node (ASM-070); farmer app for the sack record | Drying node: as the farm. Farmer app: HTTPS (TLS 1.2 or later) | Drying node: as the farm. App: encrypted outbox (ASM-079) | Drying node: store-and-forward 24 hours. App: outbox until acknowledgement | IoT Core endpoint; public application API backed by the `application` tier |
| Drone | Drone and ground station of the contracted operator (ADR-002) | HTTPS (TLS 1.2 or later): mission download from the application API, flight record and object upload to Amazon S3 | Drone and ground station storage, no time limit | Mission already loaded; record and objects stay in the buffer; fumigation cannot close | Public S3 and API endpoints, or from a center the API and Cognito interface endpoints in `application` and the S3 interface endpoint in `object_storage`; `drone_object_ingress` holds no endpoint in the pilot (ASM-140) |
| Transporter | Transporter's smartphone and app | HTTPS (TLS 1.2 or later) | Encrypted outbox (ASM-076, ASM-079) | GNSS fix every 5 minutes keeps being persisted; custody events and positions drain on reconnect | Public application API backed by the `application` tier |
| Collection center | Center firewall in 10.4x.20.0/24 | IPsec site-to-site VPN to the hub in the network account (ASM-078) | Logistics app outbox on each post (ASM-079) | Cellular backup link; if both fail, reception continues offline | Hub, then `transit` tier, then interface endpoints of the environment VPC |
| Port (Buenaventura) | Transporter's smartphone and app; no Federation equipment | HTTPS (TLS 1.2 or later) | Same outbox | Same as the transporter | Public application API |

## 4. Farm

### 4.1 Field equipment

Each farm receives two field nodes, one per zone of the case statement ("Cada finca tiene zona de cultivo y zona de secado"): a crop node in the crop zone and a drying node at the drying area of the wet mill (ASM-070). Each node is an AWS IoT Core thing with its own X.509 certificate registered from a CSR (ADR-003, ASM-010), so revoking one node does not affect the other. The crop node is described here; the drying node in section 5.

| Device class | Function | Constraint it answers |
|---|---|---|
| Sealed node enclosure rated for outdoor high humidity and high temperature | Holds the controller, radio module and buffer. | "El equipo de la finca tiene que soportar humedad alta y temperatura alta." |
| Small solar panel with rechargeable battery and charge controller | Powers the node; the radio sleeps between publications. | "El nodo de finca es autónomo en energía. En el lote no hay suministro eléctrico confiable." |
| Multi-parameter soil probe and air temperature and humidity sensor | Nitrogen, phosphorus, potassium, moisture and temperature of air and soil, every 4 hours. | Case figures, crop variables and cadence. |
| Cellular LPWA radio module (LTE-M or NB-IoT, LTE Cat-1 acceptable), dual-mode with satellite where needed | IP bearer for MQTT over TLS. | Section 11; out-of-coverage farms (section 4.4). |
| Multi-network IoT SIM or eSIM profile | Attaches to any LTE network that serves the node location, not only one operator (ASM-071). | More farms in coverage without a new radio base. |

The node is provisioned at a depot before the visit: it generates its key pair, its CSR is registered, and its thing name and farm identifier are set. The installation visit therefore only mounts the node on a stake or pole, inserts the probe, records the node position with the installer's phone and checks that the node attaches and that its first reading reaches the platform. This keeps the installation to a single visit with little equipment ("La instalación en finca se hace en una sola visita y con poco equipo") and keeps fixed hardware to two small nodes ("Instalar la menor cantidad posible de hardware fijo en la finca"). No gateway, mast, repeater or wired link is installed.

### 4.2 Exit protocol and path to the cloud

The crop node publishes every reading as soon as it is taken, with MQTT over TLS 1.2 or later and mutual X.509 authentication, to the regional IoT Core data endpoint of the environment in `us-east-1` (ADR-003, transport and ingestion endpoint). It uses QoS 1, so the broker acknowledges each message, and port 8883; where a network blocks that port, MQTT over TLS on port 443 with the ALPN protocol that IoT Core documents for this case is the alternative. The node connects, publishes and returns to sleep; it holds no persistent session.

The node does not evaluate thresholds. The Federation configures the alert ranges as data ("Los rangos de alerta los configura la Federación. No quedan fijos en el dispositivo"), so the node sends every reading and the cloud decides. Each message carries the node identifier, the original reading timestamp, a per-node sequence number and the values (ASM-011). The IoT policy lets the connection publish only to its own topic. The topic rule writes the reading to the telemetry store, keyed by node identifier and reading timestamp, and invokes the threshold function, which raises the alert of section 8 (ADR-003, routing rule and alert path).

The path is: node, cellular radio access, operator core network, public internet, IoT Core public endpoint in `us-east-1`. Farm traffic does not cross the collection centers, the hub or any VPC subnet. The `telemetry_ingress` tier keeps its reservation for the IoT Core interface endpoint used by private paths (addressing plan section 4.2) and is not on the farm path.

### 4.3 Without signal: store-and-forward up to 24 hours

The case statement fixes the behavior: "Sin señal, el dispositivo hace store-and-forward. El buffer local no retiene más de 24 horas." The buffer is specified as follows (ASM-074).

| Property | Specification |
|---|---|
| Medium | Non-volatile memory in the node, encrypted at rest, that survives a power loss or a reboot. |
| Content | Each reading as it would have been published: node identifier, original reading timestamp, sequence number, values. |
| Capacity | At least 24 hours of readings at the node's cadence: 6 readings for the crop node, 24 for the drying node, at most 30 per farm (ADR-003). Sized with margin so that capacity is never the limit; retention is. |
| Retention | No reading older than 24 hours is kept. When a reading reaches 24 hours without acknowledgement it is evicted and a per-node drop counter is incremented. |
| Trigger to drain | The radio attaches and the TLS session to IoT Core is established. |
| Drain order | A reading taken at the moment of reconnection is published first, so a live out-of-range value never waits behind history; then the buffer drains oldest first. |
| Release | A reading is removed only after its PUBACK. If the session drops mid-drain, the remaining readings stay and the drain resumes at the next session. |
| Gap report | The first message after a drain carries the drop counter and the oldest and newest sequence numbers sent, so the platform can tell an evicted gap from a node that took no reading. |

Drained readings enter by the same topic, rule and store as live ones. The store key makes the drain idempotent, and the threshold function evaluates each drained reading, so an out-of-range reading raises its alert at drain time, within the 24 hours the case statement allows ("Sin señal, la alerta se emite al drenar el buffer, dentro de las 24 horas"). Because the alert is measured from the original timestamp, its delay is visible and is not confused with a live alert (section 8.3). A node silent for more than 24 hours is reported by the observability of point 6 ("Nodos y fincas sin telemetría por más de 24 horas"), designed by A21 on CloudWatch (ADR-003).

### 4.4 Farms out of coverage

The case statement says: "Hay fincas del piloto fuera de cobertura. Sus lecturas y objetos entran sin desplegar una radiobase nueva ni programar una visita semanal." No coverage figure is given, and none is assumed: a farm is out of coverage when its nodes cannot attach to any LTE network at the installation check, using the multi-network SIM (ASM-073). The number of such farms is learnt during installation and is not a design input.

**Mechanism.** Every node is bought with the same dual-mode radio module class, terrestrial LTE-M or NB-IoT plus a satellite IoT bearer that provides IP connectivity, such as 3GPP non-terrestrial NB-IoT (ASM-073). On a farm in coverage the satellite subscription stays inactive. On a farm out of coverage the installer activates it in the same visit, and the node keeps exactly the same protocol, identity, topic, buffer and drain: MQTT over TLS with its X.509 certificate to the same IoT Core endpoint. Nothing changes in the cloud, the addressing or the ingestion point, which is what "El patrón de red y de ingesta escala de 200 a 1.000 fincas sin rediseñar el direccionamiento ni el punto de ingesta" requires. Objects of an out-of-coverage farm are the drone's photos, maps, polygons and flight records; they do not travel over the node and enter through the drone flow (section 6), which already uploads from the municipal seat or the collection center.

**Why this mechanism.** It needs no radio base, no mast and no visit: the satellite is the radio access. It keeps the 24-hour bound under the platform's control, because it does not depend on a person travelling to coverage. It keeps the TLS session end to end, so no third party sees readings in clear, and it keeps the revocation of ADR-003 working unchanged. Buying one module class for all nodes means the installer does not need to know beforehand which farms lack coverage, which preserves the single visit.

**What it costs in delay.** Over a satellite bearer the node publishes during contact windows. While a contact is active the route is available and the alert budget of section 8 applies; the attach and uplink hops of that budget must be verified for the chosen service before procurement (PD-081). Between contacts the node is in the no-signal case of section 4.3: readings wait in the buffer and their alerts are raised at the next contact. The procurement criterion is a contact interval of at most 4 hours, one crop reading interval, so that the 24-hour retention is never reached in normal operation (ASM-073); this is a requirement on the service, not a measured property. A second delay sits on the farmer side: the farmer's phone has data only at the municipal seat or on a main road (case statement, assumptions), so on an out-of-coverage farm the alert is raised in the platform at the contact but reaches the farmer only when the phone next has data. The alert opens the drone confirmation case at once (ADR-002, `crop_alert_raised`), so the pest flow does not wait for the farmer. In money, the cost is a dearer module class on every node and a satellite fee only for the farms that need it; A23 prices both at current prices (PD-084).

**Alternatives rejected.**

| Alternative | Reason rejected |
|---|---|
| Private LTE or LoRaWAN gateway on a hill near the farms | It is a new radio base in all but name, needs a site survey and power, and usually more than one visit. |
| Farmer's phone as a data mule (short-range sync with the node, upload when the phone reaches data) | Delivery depends on when the farmer travels, so it cannot guarantee the 24-hour bound; the phone would hold a second copy of the readings outside the node buffer, beyond its 24-hour retention. |
| Point-to-point radio relay to a covered neighbor farm | Fixed hardware on a second farm, line-of-sight survey, and a dependency on a third party's power and consent. |
| Manual download by a technician | It is the scheduled visit the case statement excludes. |

## 5. Wet mill

The wet mill hosts the drying zone and the packing of the coffee that leaves in 60 kg sacks ("El café sale del beneficiadero en sacos de 60 kg"). It has no wired access ("Finca y beneficiadero no tienen acceso cableado"), and it carries two flows.

**Drying telemetry.** The drying node (ASM-070) is the same device class as the crop node with a probe for grain temperature and coffee moisture, the drying variables of the case statement, read every hour while the lot is drying. It shares the exit protocol, endpoint, topic scheme, buffer, 24-hour store-and-forward, drain and out-of-coverage mechanism of sections 4.2 to 4.4. A drying node holds at most 24 readings after 24 hours offline. Drying alerts, such as grain above 40 °C, follow the same alert path and budget as crop alerts (section 8).

**Sack registration.** The sack is registered at the farm with its weight and identifier ("Alta del saco en la finca, con peso e identificador"), and each sack event keeps date, time and responsible. The farmer app sends it with HTTPS (TLS 1.2 or later) to the platform's public application API, authenticated with the farmer identity of point 2. The managed front door of that API is chosen by the application design; behind it, the API is served from the `application` tier. When the phone has no data, the app writes the event to its encrypted outbox with a client-generated event identifier, the farmer identity, the sack identifier, the weight and the time it was recorded; the outbox keeps it until the API acknowledges persistence (ASM-079). Because the farmer's phone has data at the municipal seat or on a main road, the event normally arrives on the farmer's next trip. The chain of custody does not depend on that upload: the transporter app records the handoff by reading the sack code (section 7), and a handoff for a sack whose registration has not yet arrived is held by the platform as pending until the registration drains.

## 6. Drone

ADR-002 fixes the rules this flow implements: the mission is preloaded before take-off at the municipal seat or the collection center; if the link drops over the plot, the record and objects stay in the drone buffer; they are published when the link returns, at the latest when the team is back at the preload point (ASM-028); and the fumigation does not close until the record is persisted ("Sin cierre de ese registro no se autoriza la fumigación"). The link between the drone and its ground station is the operator's own radio and is out of scope.

| Aspect | Design |
|---|---|
| Exit protocol | HTTPS with TLS 1.2 or later from the ground station. The mission package (order, polygon, no-fly zones) is downloaded from the application API under the drone operator identity with MFA. Flight records and objects are uploaded to Amazon S3 with short-lived pre-signed URLs that the API issues to that identity for the order's own prefix, using multipart upload with per-part checksums so an interrupted upload resumes instead of restarting. Each delivered object is audited (point 1). |
| Local buffer | Storage of the drone and of the ground station. No time limit applies: the case statement says the mission "queda en el buffer del dron" and sets no retention. A record or object is deleted from the ground station only after the platform confirms that it is persisted with a matching checksum. |
| Without signal | The mission is already loaded, so the flight proceeds. The order moves to `flown` and stays open; no authorization depending on that record can be given, and the fumigation does not reach `executed`, until the upload is persisted (ADR-002, order states). Observability reports flights without a persisted record (point 6). |
| Path from a collection center | Ground station on the center LAN (Soporte, 10.4x.2.10, A11 section 4.1), center firewall, IPsec tunnel to the hub, `transit` tier, then the API and Cognito interface endpoints in `application` and the S3 interface endpoint in `object_storage`. The `drone_object_ingress` tier stays reserved and holds no endpoint in the pilot ([drone service](drone-service.md#9-ingress-endpoints-of-the-ground-station-pd-121) section 9, ASM-140). The traffic never leaves the private path. |
| Path from a municipal seat | Ground station on any public data link (phone tethering or a local internet service), public internet, public S3 and API endpoints in `us-east-1`. Integrity and confidentiality come from TLS and the checksums, not from the network. |
| Delay | The record arrives when the team reaches a data link. It does not affect the crop alert budget: a fumigation stays open until then, which is the intended behavior (ADR-002, consequences). |
| Lost drone | Its identity is revoked (point 2); upload URLs are no longer issued for it and the platform rejects records that reference it. |

Large uploads from a center share the 100 Mbps WAN with operations; their traffic class is set by A11 and A12 under the non-operational traffic control of point 3, not here.

## 7. Transporter

The case statement leaves both values to the design: "El transportador publica posición en ruta. El diseño fija el intervalo y qué evento queda persistido si el teléfono pierde datos." This section fixes them and closes PD-010.

**Interval.** The position interval is 5 minutes en route, the figure of the case statement ("En ruta se espera ubicación cada 5 minutos"; "60 transportadores × 12 posiciones por hora, en una jornada de 10 horas"). "En route" starts when the transporter accepts custody of a sack or a load (pickup at the farm or departure from the center) and ends at arrival at the collection center or delivery at the port. Outside a trip no position is taken.

**Exit protocol and path.** The transporter app sends events with HTTPS (TLS 1.2 or later) to the public application API, authenticated with the transporter identity of point 2. The path is phone, public mobile network, public internet, API front door, `application` tier; the phone holds no cloud address (ASM-032). In the yard of a collection center the phone may use the transport SSID (10.4x.4.0/24), whose internet exit is ruled by A11 and A12; the app's behavior is the same. Positions are a separate data path from farm telemetry, which the case statement allows ("Pueden ser distintos"), and they are stored in the transactional or time-series store that Part E allows for position.

**Event persisted when the phone loses data.** Position is taken from the phone's GNSS receiver, which does not need a data connection (ASM-076). Every fix is therefore persisted as a `transport_position` event in the outbox whether or not there is data, and the reference every 5 minutes is never interrupted by a data loss; only its upload is delayed. Custody changes are persisted as `sack_custody` events. Both are written to the outbox before any upload is attempted.

| Field | `transport_position` | `sack_custody` | Content |
|---|---|---|---|
| `event_id` | Yes | Yes | UUID generated on the phone; idempotency key. |
| `event_type` | Yes | Yes | `transport_position`, or the custody step: `picked_up_at_farm`, `arrived_at_center`, `departed_to_port`, `delivered_at_port`. |
| `trip_id` | Yes | Yes | Trip the event belongs to. |
| `transporter_id` | Yes | Yes | Transporter identity of point 2; the API checks it against the authenticated caller. |
| `device_id` | Yes | Yes | App installation identifier, for revocation and audit. |
| `recorded_at` | Yes | Yes | UTC time of the fix or the custody act, taken on the phone. |
| `sequence_no` | Yes | Yes | Per-trip counter, so the platform detects missing events. |
| `latitude`, `longitude`, `horizontal_accuracy_m` | Yes | Yes | GNSS position and its accuracy. |
| `speed_mps`, `heading_deg` | Yes | No | Motion at the fix, when the receiver gives it. |
| `link_state` | Yes | Yes | `online` or `offline` when recorded; the first `offline` fix marks where data was lost. |
| `sack_ids` | No | Yes | Sacks read by code in this act. |
| `outcome`, `receiver_reference` | No | At center and port | `accepted` or `rejected`, and the receiver's reference (section 10, ASM-077). |
| `received_at` | Set by the API | Set by the API | Server time of persistence; the difference with `recorded_at` is the upload delay. |

**Buffer.** The outbox is app storage encrypted at rest, survives app restarts and phone reboots, and has no time-based eviction: an event leaves only when the API confirms persistence (ASM-079). It is sized for at least 7 days of fixes at the 5-minute interval, 2,016 position events, plus every custody event of the trip; custody events are never evicted, and if the position capacity were ever exhausted, the oldest positions would be thinned to one per 15 minutes before any is dropped (ASM-076).

**Without signal.** The app keeps recording, and when data returns it uploads custody events first and then positions oldest first, in batches. The API stores each event once by `event_id` and keeps `recorded_at`, so the trip trace is rebuilt with its original times. A custody event that arrives after a later step is ordered by `recorded_at` and `sequence_no`, not by arrival. The case statement's assumption that the transporter has data on main roads bounds the gaps to the feeder roads between the farms and the main road.

## 8. Crop alert under 5 minutes

The case statement fixes the target: "Latencia de alerta, con ruta de entrega disponible: menos de 5 minutos desde la lectura fuera de rango."

### 8.1 When a route is available

A route is available for a reading when, at the time the reading is taken, the node has an attached bearer (cellular, or an active satellite contact) and the farmer's phone can receive the notification. When the node side is missing, the reading follows the 24-hour drain path. When only the farmer side is missing, the alert is raised and persisted within the budget but its delivery waits for the phone; it counts as an undelivered alert in point 6 until delivered.

### 8.2 Latency budget per hop

The budget runs from the original reading timestamp (ASM-011) to the receipt of the notification on the farmer's phone. Every value is a design target, not a measurement (ASM-075).

| Hop | From and to | Target | Notes |
|---|---|---|---|
| 1 | Reading taken to message ready on the node | 10 s | The node publishes every reading at once; it does not batch or evaluate thresholds. |
| 2 | Radio wake, network attach and TLS connection | 60 s | The largest field hop; for a satellite bearer, to be verified in procurement (PD-081). |
| 3 | MQTT publish to PUBACK at the IoT Core endpoint | 20 s | Includes MQTT retransmission at QoS 1. |
| 4 | Topic rule to threshold function invocation | 10 s | ADR-003 routing rule, through the threshold queue ([inference functions](inference-functions.md#3-the-four-triggers) section 3, ASM-161). |
| 5 | Threshold function: read configured ranges, persist the alert, publish to the alert topic | 20 s | Function designed by A17 (aroma-pacifico/aroma-pacifico-roadmap#17). |
| 6 | Alert topic to the notification channel provider | 30 s | Amazon SNS toward the channel chosen by A14. |
| 7 | Channel provider to the farmer's phone | 60 s | App push notification or SMS, whichever A14 chooses (PD-085). |
| | Total of targets | 210 s | |
| | Margin to the case statement limit | 90 s | Absorbs retries on any hop. |
| | Case statement limit | 300 s | "menos de 5 minutos" |

The network round trip from Colombia to `us-east-1` is inside hops 3 and 5 and is not expected to matter at this scale (ASM-003). Drying alerts use the same budget.

### 8.3 Tie to the alert-delivery SLO

The budget is the design input to the alert-delivery SLO of A21 (point 6, "Dos SLO, cada uno con su SLI: entrega de la alerta de cultivo"). The SLI proposed to A21 is the share of out-of-range readings that were published live, that is, not from the buffer, and whose notification was received by the farmer within 300 seconds of the reading timestamp. Drained readings are measured separately against the 24-hour limit. The hop timestamps above (reading, PUBACK, function start, alert persisted, handoff to the channel, delivery receipt) are the trace points that let A21 attribute a breach to a hop. The objective percentage and the window are set by A21 (PD-082).

## 9. Collection center WAN

The case statement asks each center for a "salida WAN hacia la nube" and sets "WAN de cada centro: 100 Mbps simétricos, compartidos con la operación". The LAN, its VLANs, SSIDs and per-client capacity are A11; this section covers only the egress and its link to the cloud.

| Aspect | Design |
|---|---|
| Edge | The center firewall in the cloud link zone, 10.4x.20.0/24 (10.41 Cauca, 10.42 Huila, 10.43 Tolima). |
| Exit protocol | IPsec site-to-site VPN over the 100 Mbps internet service, two tunnels, terminated on the hub in the network account (ASM-078). The hub attaches each environment VPC through its `transit` tier (addressing plan section 4, ASM-033). |
| Routing | The center advertises its own /16; the cloud reaches all centers with the single summary 10.40.0.0/13 via `transit` (addressing plan sections 2 and 4). Operational traffic reaches the interface endpoints in `application`, `object_storage` and `administration` of `pdn` privately ([firewall policy](firewall-policy.md#52-policy-table), CL-LNK-01, ASM-121); `drone_object_ingress` and `telemetry_ingress` hold no endpoint in the pilot (ASM-140, CL-TEL-04). |
| What crosses the tunnel | Sack reception and code printing from the logistics posts, mission downloads and flight uploads from a ground station at the center, and administration of the center equipment. |
| What does not | Farm telemetry and transporter traffic, which use public endpoints; visitor traffic, which the firewall breaks out to the internet locally (A11, A12). |
| Backup link | A cellular backup link of LTE or a later generation on the center firewall, which re-establishes the same IPsec tunnels if the primary service fails (ASM-078). |
| Local buffer | Each logistics post and logistics phone app keeps an encrypted outbox of sack reception events with client-generated identifiers, released on acknowledgement (ASM-079). |
| Without WAN | If both links fail, reception continues: events are recorded with their time and responsible and drain when the link returns. The three WAN links are observed as point 6 requires. |

The firewall rules that implement this path, and whether each is permanent or temporary, are classified by A12 using the ranges of the addressing plan (PD-083, PD-052).

## 10. Port of Buenaventura

At the port the Federation deploys nothing ("En Buenaventura no se despliega infraestructura de la Federación"), the shipping line keeps its own telemetry, and the platform only registers the sack delivery event ("La plataforma solo registra el evento de entrega del saco"). No integration with the shipping line or the port is designed.

The delivery event enters through the transporter's phone and the transporter app, over the public mobile network available in the city, with HTTPS to the public application API, as a `sack_custody` event of type `delivered_at_port` (section 7). The transporter reads each sack code and records the outcome that the port receiver gives, accepted or rejected, with the receiver's reference for the act, which closes the sack states of Part C, "entregado en puerto (aceptado o rechazado)" (ASM-077). If the phone has no data at the port, the event waits in the outbox like any other custody event and keeps its `recorded_at`, so the delivery time in the sack history is the real one. Tracking of the ship is out of scope ("No hay que seguir el barco por fuera de la entrega en puerto").

## 11. Mobile network generations

The case statement says: "La red de acceso móvil cambió desde 2020. La solución no depende de una generación de red que el operador ya apagó o que vaya a apagar durante el piloto." This document does not assert which operator switches off which generation or when, because no official source was verified for this task. The requirement is stated as a technology-neutral rule plus a named assumption to verify.

| Rule | Applies to |
|---|---|
| No flow depends on 2G or 3G, on circuit-switched data, or on any generation whose switch-off is announced for a date before the end of the pilot. A device or phone that could fall back to such a generation is not relied upon to do so. | Every flow. |
| The lowest terrestrial bearer any field device needs is LTE-based: LTE-M, NB-IoT or LTE Cat-1. Later generations are accepted where present. | Crop and drying nodes, center backup link. |
| Phones need only a data connection of any generation still in service; the apps never use SMS or voice to carry data. | Farmer and transporter apps. |
| Out-of-coverage farms use a satellite bearer, which does not depend on the terrestrial mobile generation at all. | Section 4.4. |
| If an SMS alert channel is chosen, it must work over LTE or later networks without a 2G or 3G circuit-switched fallback. | A14 (PD-085). |

ASM-072 assumes that LTE service, on which the terrestrial bearers rely, stays in service in Cauca, Huila, Tolima and on the corridor to Buenaventura until the pilot ends. It is to be verified against an official Colombian source, such as the Ministry of Information and Communications Technologies (MinTIC) or the Communications Regulation Commission (CRC), with a link, or raised in the clarification session (PD-080). If it were invalidated, only the bearer class of the nodes would change; the protocols, endpoints, buffers and identities of this design would not.

## 12. Compliance with the case statement and the issue

| Requirement | Where it is met |
|---|---|
| "cubre finca, dron, transportador, centro de acopio y puerto, con el comportamiento sin señal" and the wet mill | Sections 4 to 7, 9 and 10; summary in section 3. |
| "Cada uno indica protocolo de salida, buffer local y qué pasa cuando no hay señal" | Each flow section and the table of section 3. |
| "Sin señal, el dispositivo hace store-and-forward. El buffer local no retiene más de 24 horas." | Section 4.3, buffer specification and drain. |
| "Sus lecturas y objetos entran sin desplegar una radiobase nueva ni programar una visita semanal." | Section 4.4, mechanism, justification and delay. |
| "Latencia de alerta, con ruta de entrega disponible: menos de 5 minutos" | Section 8, per-hop budget and SLI for A21. |
| "Sin señal, la alerta se emite al drenar el buffer, dentro de las 24 horas." | Sections 4.3 and 8.3. |
| "El diseño fija el intervalo y qué evento queda persistido si el teléfono pierde datos." | Section 7: 5 minutes, `transport_position` and `sack_custody` events and the outbox. |
| "Si el enlace cae, la misión queda en el buffer del dron. Sin cierre de ese registro no se autoriza la fumigación." | Section 6. |
| "salida WAN hacia la nube" and "100 Mbps simétricos" | Section 9. |
| "En Buenaventura no se despliega infraestructura de la Federación [...] solo registra el evento de entrega del saco." | Section 10. |
| "La solución no depende de una generación de red que el operador ya apagó o que vaya a apagar durante el piloto." | Section 11. |
| "La instalación en finca se hace en una sola visita y con poco equipo." and moderate hardware per farm | Section 4.1, device classes and depot provisioning. |
| "El patrón de red y de ingesta escala de 200 a 1.000 fincas sin rediseñar el direccionamiento ni el punto de ingesta." | Sections 2 and 4.4: one endpoint, identities not addresses, satellite farms on the same endpoint. |

## 13. Decision-log entries

| ID | Summary |
|---|---|
| PD-010 | Transporter interval and persisted event. Closed by section 7. |
| ASM-070 | Two field nodes per farm, crop and drying, each with its own identity. |
| ASM-071 | LTE-based LPWA bearer and a multi-network IoT SIM. |
| ASM-072 | LTE stays in service in the pilot territory until the pilot ends; to verify (PD-080). |
| ASM-073 | Out-of-coverage farms are found at installation and use a dual-mode module with a satellite IP bearer, contact interval of at most 4 hours. |
| ASM-074 | Node buffer specification. |
| ASM-075 | Per-hop alert budget values are design targets. |
| ASM-076 | Transporter position from GNSS without data; outbox of at least 7 days. |
| ASM-077 | The port delivery outcome is recorded by the transporter with the receiver's reference. |
| ASM-078 | Center egress by IPsec VPN to the network-account hub, with a cellular backup link. |
| ASM-079 | App outboxes release events only on acknowledgement and have no time-based eviction. |
| PD-080 | Verify ASM-072 against an official source. |
| PD-081 | Verify the satellite IP service and dual-mode modules; if only non-IP messaging exists, A13 designs a bridge. |
| PD-082 | A21 adopts the SLI and budget and sets the objective. |
| PD-083 | A12 classifies the center WAN and drone upload rules. |
| PD-084 | A23 prices both node module classes and the satellite fee. |
| PD-085 | A14 chooses the farmer alert channel within the hop 7 target. |

## 14. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Qué se actualiza, restrictions, assumptions, pilot figures, drone section, points 2, 3 and 6, Part C. |
| [`docs/decision-log.md`](decision-log.md) | ASM-001, ASM-003, ASM-010, ASM-011, ASM-028 and the entries of section 13; ASM-032 and ASM-033 come from A07, ASM-041 from A05. |
| [ADR-001](adr/adr-001-primary-and-second-cloud.md) | AWS in `us-east-1`; one VPC per environment account; collection centers connect to the network account. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | Mission preload, flight record, order states, `crop_alert_raised`. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | IoT Core, MQTT over TLS, X.509 per node, idempotent 24-hour drain, alert path. |
| [`docs/legacy-2020.md`](legacy-2020.md) | Rows L-09 to L-13 replaced or retired here; ASM-041. |
| [`docs/addressing-plan.md`](addressing-plan.md) | Collection-center /16s, 10.40.0.0/13 summary, tiers including `transit`, managed endpoints per tier. |
| [AWS IoT Core protocols, port mappings and authentication](https://docs.aws.amazon.com/iot/latest/developerguide/protocols.html) | MQTT over TLS on 8883 and on 443 with ALPN for X.509 clients. |
| [Uploading objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html) and [Checking object integrity](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html) | Drone uploads with short-lived URLs and checksums. |
| [AWS Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html) | IPsec tunnels from each center to the hub. |
