# Farm Monitoring and Pilot Alerts

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / JR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#14 (A14) |
| Builds on | [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md), [`docs/connectivity.md`](connectivity.md) (A10), [`docs/landing-zone.md`](landing-zone.md) (A08), [`docs/security-and-identity.md`](security-and-identity.md) (A09), [`docs/clarification-questions.md`](clarification-questions.md) (A06), `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13) |
| Feeds | A15 drone service (aroma-pacifico/aroma-pacifico-roadmap#15), A16 sack and lookup (aroma-pacifico/aroma-pacifico-roadmap#16), A17 functions (aroma-pacifico/aroma-pacifico-roadmap#17), A19 architecture document (aroma-pacifico/aroma-pacifico-roadmap#19), A21 observability and SLOs (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36), B09 ingestion module (aroma-pacifico/aroma-pacifico-roadmap#29) |
| Decision-log entries | PD-074 and PD-085 (closed here), ASM-130 to ASM-138, PD-130 to PD-135 |

## 1. Purpose and requirements

This document defines the functional and operational behavior of crop and drying monitoring in the pilot: the load the ingestion path must carry, how the Federation configures the alert ranges as data, how each reading is evaluated, how a farmer is alerted once and not once per reading, what happens with readings that arrive late from the 24-hour buffer, what the farmer sees, and what the farmer history and the pilot report read. It covers Part A of the [case statement](case/case-statement.pdf) and the crop alert of the pilot figures. It builds no application and no firmware: the case statement excludes both ("Quien presenta no implementa lógica de negocio, aplicaciones ni firmware"). Quotations are in the original Spanish, which prevails over the English rendering (ASM-001). No figure here is a measurement; latencies are design targets.

| Case statement text | Where it is met |
|---|---|
| Part A: "Cultivo: 200 fincas × 6 lecturas al día." and "Secado activo, en cosecha: unas 100 fincas × 24 lecturas al día." | [Section 3](#3-load) |
| "El patrón de red y de ingesta escala de 200 a 1.000 fincas sin rediseñar el direccionamiento ni el punto de ingesta." | [Section 3.4](#34-scale-to-1000-farms) |
| "Los rangos de alerta los configura la Federación. No quedan fijos en el dispositivo." and the table "Rangos iniciales, configurables" | [Section 4](#4-thresholds-as-federation-data) |
| Part A: "Ingesta de telemetría de cultivo y de secado, con los umbrales de la tabla de cifras." | [Sections 4](#4-thresholds-as-federation-data) and [5](#5-evaluation-of-a-reading) |
| "Latencia de alerta, con ruta de entrega disponible: menos de 5 minutos desde la lectura fuera de rango. Sin señal, la alerta se emite al drenar el buffer, dentro de las 24 horas." | [Sections 5](#5-evaluation-of-a-reading), [6](#6-alerts-and-deduplication) and [7](#7-farmer-notification-channel) |
| "Sin señal, el dispositivo hace store-and-forward. El buffer local no retiene más de 24 horas." | [Section 6.4](#64-readings-drained-from-the-buffer) |
| Part A: "Historial para el caficultor y para el reporte del piloto." and "Retención de lecturas [...]: 5 años." | [Section 9](#9-history-and-the-pilot-report) |
| Part A: "El síntoma visible del cafeto entra por el servicio de confirmación del dron, no por una revisión semanal aparte." | [Section 6.3](#63-what-an-episode-produces) |
| Part E: "Lote de telemetría. La función marca la lectura fuera de umbral y persiste el resultado. Las reglas configurables siguen vigentes." | [Section 5](#5-evaluation-of-a-reading) |
| Point 1: "Auditoría de [...] cambio de un umbral, publicación de telemetría [...]" | [Section 4.5](#45-audit-of-a-threshold-change) |
| Restriction: "La aplicación del caficultor tiene pocas pantallas." | [Section 8](#8-what-the-farmer-sees) |

## 2. Accepted inputs

The design below does not reopen any accepted decision; it fills the parts they leave to this task.

| Source | What it fixes | What it leaves to this task |
|---|---|---|
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | AWS IoT Core endpoint, MQTT over TLS with X.509 per node, a topic rule that writes each reading to the DynamoDB telemetry table keyed by node identifier and original reading timestamp and invokes the threshold function, an SQS dead-letter queue as the rule's error action, an SNS alert topic, thresholds as data read by the function and never in the rule SQL. | The farmer channel and the threshold data. |
| [A10 connectivity](connectivity.md) | Two nodes per farm (ASM-070); nodes send every reading and never evaluate thresholds (section 4.2); 24-hour encrypted buffer, live reading first then oldest first, release on PUBACK, drop counter and gap report (ASM-074, section 4.3); the 210-second per-hop alert budget (section 8.2, ASM-075); the generation-neutral rule (section 11). | Hops 6 and 7 of the budget and the channel (PD-085). |
| [A08 landing zone](landing-zone.md#72-event-by-event-design) | Threshold change audited by an application record written in the same transaction as the change, with the fields who, what, when and from where; telemetry publication audited by the stored reading and the IoT Core `ERROR` log; 5-year audit retention (ASM-067). | The threshold store and its secondary audit source (PD-074). |
| [A09 security and identity](security-and-identity.md#42-matrix-of-role-data-object-and-action) | The farmer reads alerts, closure notices and the ranges of their own farms; the agronomist reads by department; only the platform operator writes thresholds, on the Federation's instruction with a recorded reason and a fresh second factor (ASM-099, ASM-092); KMS key per data class (ASM-097); retention by TTL (section 9.2). | Nothing; this document applies it. |
| [A06 fallbacks](clarification-questions.md#4-important-questions) | ASM-053 (pH from a soil analysis), ASM-054 (NPK ranges are Federation data for the whole pilot, no alert until published) and ASM-059 (about 12 % is a lot-closure notice, not an alert). | Nothing; they are used as written. |
| A13 data paths (`aroma-pacifico-diagrams/data-paths/data-paths.md`) | The crop and drying data path: input fields, protocol, buffer, persistence for 5 years, the threshold function persisting `crop_alert_raised` and emitting through the SNS alert topic; the ordering of the store write and the function invocation is open (PD-112). | The evaluation and alert behavior, which section 5 makes independent of that ordering. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | `crop_alert_raised` opens a pest case for a node inside a delimited plot and switches the plot to precision mode. | How often that event is produced, which section 6 limits to once per episode. |

## 3. Load

### 3.1 Readings per day and per hour

Every reading is one MQTT publication, because the node publishes each reading as soon as it is taken and does not batch (A10 section 8.2, hop 1). Each publication causes two topic rule actions, the store write and the message that invokes the threshold function through its queue (ADR-003, ASM-161). The figures below follow from the case statement alone.

| Flow | Arithmetic | Per day | Average per hour | Per 30-day month |
|---|---|---|---|---|
| Crop, all year | 200 farms x 6 readings | 1,200 | 1,200 / 24 = 50 | 36,000 |
| Drying, harvest | 100 farms x 24 readings | 2,400 | 2,400 / 24 = 100 | 72,000 |
| Harvest day total | 1,200 + 2,400 | 3,600 | 150 | 108,000 |
| Outside harvest | Crop only; the case statement gives drying load only in harvest ("Secado activo, en cosecha") | 1,200 | 50 | 36,000 |
| Bound if all 200 farms dried at once | 1,200 + 200 x 24 | 6,000 | 250 | 180,000 |

The harvest day of 3,600 readings is the design load, the same figure A08 uses for the publication audit. The case statement says "unas 100 fincas"; the last row shows that even every farm drying at once stays within the same order of magnitude, so no design element depends on the exact number.

| Derived quantity | Harvest day | Basis |
|---|---|---|
| Threshold function invocations | 3,600 | One per reading (ADR-003). |
| Topic rule actions | 7,200 | Two per reading. |
| Telemetry table writes | 3,600, plus duplicates absorbed by the key | One per reading; a QoS 1 redelivery or a drain after a lost PUBACK rewrites the same item (A10 section 2). |
| Average rate | 3,600 / 86,400 s = 0.042 readings per second | |

### 3.2 Peak rate

The average hides two peaks, and the design is checked against both without assuming any node schedule.

| Peak | How it arises | Bound at 200 farms |
|---|---|---|
| Aligned slot | The case statement says "cada 4 horas" and "cada hora" but not at which minute. If every crop node reports in the same minute as every drying node, one minute carries 200 + 100 readings. | 300 readings in one minute, 5 per second. |
| Drain after a regional outage | A cellular outage of up to 24 hours that affects every node leaves at most one day of readings in the buffers, because nothing older is kept (ASM-074). If every node reconnects at once and the whole backlog arrives within one minute: | 3,600 readings in one minute, 60 per second. |

Neither figure is a measured rate; both are upper bounds from the case figures. The drain bound is deliberately extreme, since nodes reconnect as their own radios attach. The bounds go to the verification of AWS service quotas (IoT Core inbound publications, rule actions, Lambda concurrency and the notification channels), which is PD-130; no quota value is asserted here. Per node, the load stays trivial: a drying node writes one item per hour and drains at most 24 items, a crop node at most 6, each to its own partition key.

### 3.3 Five-year volume

The case statement keeps readings 5 years ("Retención de lecturas [...]: 5 años"). The number of harvest days per year is set by the Federation's harvest calendar (ASM-024), so the stored volume is given as bounds and a formula, and A23 converts it into storage with its own item-size assumption.

| Quantity | Arithmetic | Readings at 200 farms |
|---|---|---|
| Per year, lower bound (crop only) | 1,200 x 365 | 438,000 |
| Per year, upper bound (every day a harvest day) | 3,600 x 365 | 1,314,000 |
| Per year, formula | 1,200 x 365 + 2,400 x D, where D is the number of drying days at harvest load | |
| Five years, lower and upper bound | x 5 | 2,190,000 to 6,570,000 |

### 3.4 Scale to 1,000 farms

At 1,000 farms the readings per farm and the share of farms drying stay as at 200 (ASM-036).

| Quantity | Arithmetic | 1,000 farms |
|---|---|---|
| Crop per day | 1,000 x 6 | 6,000 (250 per hour) |
| Drying per harvest day | 500 x 24 | 12,000 (500 per hour) |
| Harvest day total | 6,000 + 12,000 | 18,000 (750 per hour, 0.21 per second) |
| Aligned-slot peak | 1,000 + 500 in one minute | 1,500 per minute, 25 per second |
| Drain bound | 18,000 in one minute | 300 per second |
| Five-year upper bound | 18,000 x 365 x 5 | 32,850,000 readings |

Nothing in this design changes with scale. The endpoint, topic scheme, rule, table key and threshold function are the same for 200 or 1,000 farms (ADR-003); the threshold table grows only if the Federation sets farm-specific ranges; alert episodes and drying lots are keyed by node, so their write load spreads over more partition keys rather than concentrating. The quota check of PD-130 is made at the 1,000-farm bounds, so the 200-farm pilot never approaches it.

## 4. Thresholds as Federation data

### 4.1 Initial ranges from the case statement

The case statement table "Rangos iniciales, configurables" is reproduced exactly, in its original Spanish, followed by its rendering.

| Variable | Cultivo, alerta | Secado |
|---|---|---|
| pH | Fuera de 4,5 a 6,0. Recomendado entre 4,9 y 5,6 | No aplica |
| Humedad | Fuera de 40 % a 80 % | Cierre de lote cerca del 12 % |
| Temperatura | Fuera de 14 °C a 30 °C. Recomendado entre 17 °C y 23 °C | Grano por encima de 40 °C |
| Nitrógeno, fósforo, potasio | Fuera del rango que publique la Federación para el piloto | No aplica |

| Variable | Crop, alert | Drying |
|---|---|---|
| pH | Outside 4.5 to 6.0. Recommended between 4.9 and 5.6 | Not applicable |
| Moisture | Outside 40 % to 80 % | Lot closure near 12 % |
| Temperature | Outside 14 °C to 30 °C. Recommended between 17 °C and 23 °C | Grain above 40 °C |
| Nitrogen, phosphorus, potassium | Outside the range the Federation publishes for the pilot | Not applicable |

The table becomes the initial rows of the threshold store below. Three cells need a reading the case statement does not give, and the A06 fallbacks are used as written: pH is not measured by the node but recorded from a periodic soil analysis and evaluated against 4.5 to 6.0 (ASM-053); NPK ranges are Federation data for the whole pilot, with the unit stored with each range, and until a range is published readings are stored and no NPK alert is raised (ASM-054); and reaching about 12 % drying moisture raises a lot-closure notice to the farmer, not an out-of-range alert, with the closure value and its tolerance as Federation data (ASM-059). Three further readings are this task's own assumptions: the crop moisture and temperature ranges apply to both the air and the soil reading, as separate rows with the same initial values (ASM-131); the recommended bands are shown to the farmer and raise no alert, because the column that defines alerts is "Fuera de" (ASM-134); and the initial closure tolerance is zero, so the notice is sent when coffee moisture reaches 12 % or less, until the Federation publishes a tolerance (ASM-133).

| Row | Zone | Variable | Source | Rule | Initial values | Unit | Initial status |
|---|---|---|---|---|---|---|---|
| T-01 | Crop | `ph` | Soil analysis (ASM-053) | Alert outside range | 4.5 to 6.0; recommended 4.9 to 5.6 | pH | Active |
| T-02 | Crop | `air_moisture` | Crop node | Alert outside range | 40 to 80 | % relative humidity | Active |
| T-03 | Crop | `soil_moisture` | Crop node | Alert outside range | 40 to 80 | % | Active |
| T-04 | Crop | `air_temperature` | Crop node | Alert outside range | 14 to 30; recommended 17 to 23 | °C | Active |
| T-05 | Crop | `soil_temperature` | Crop node | Alert outside range | 14 to 30; recommended 17 to 23 | °C | Active |
| T-06 | Crop | `nitrogen` | Crop node | Alert outside range | None until published (ASM-054) | Set with the range | Pending publication |
| T-07 | Crop | `phosphorus` | Crop node | Alert outside range | None until published (ASM-054) | Set with the range | Pending publication |
| T-08 | Crop | `potassium` | Crop node | Alert outside range | None until published (ASM-054) | Set with the range | Pending publication |
| T-09 | Drying | `coffee_moisture` | Drying node | Closure notice (ASM-059) | Closure value 12, tolerance 0 (ASM-133) | % | Active |
| T-10 | Drying | `grain_temperature` | Drying node | Alert above | Above 40 | °C | Active |

### 4.2 Configuration model

A threshold is a versioned record, never a constant in the device, the rule SQL or the function code. Each change writes a new immutable version; nothing is updated in place, so every past alert can be explained by the version that raised it.

| Field | Content |
|---|---|
| `rule_key` | Identity of the rule: zone, variable and scope, for example `crop#soil_moisture#pilot` or `crop#soil_moisture#farm#<farm_id>`. |
| `zone` | `crop` or `drying`, as in A08's audit record. |
| `variable` | One of the variable codes of section 4.1. |
| `rule_type` | `alert_outside_range`, `alert_above` or `closure_notice`. |
| `lower`, `upper` | Range bounds for `alert_outside_range`; `upper` alone for `alert_above`. A reading equal to a bound is in range. Empty while `status` is `pending_publication`. |
| `recommended_lower`, `recommended_upper` | Optional band shown to the farmer; never evaluated (ASM-134). |
| `closure_value`, `closure_tolerance` | For `closure_notice` only. |
| `unit` | Unit of the bounds. The function refuses to evaluate a reading whose unit differs and records it as not evaluated. |
| `scope_type`, `scope_id` | `pilot` (no `scope_id`) or `farm` with the farm identifier, the two scopes A08 names (ASM-138). |
| `version` | Integer, starting at 1 per `rule_key`, incremented by each change. |
| `effective_from` | Server time of the change, in UTC. No backdating and no future dating (ASM-138). |
| `status` | `active`, `pending_publication` or `retired`. A retired farm-scope rule falls back to the pilot rule. |
| `changed_by`, `changed_by_role` | Cognito `sub` and group of the platform operator who wrote it (ASM-099). |
| `federation_reference` | Reference of the Federation's instruction that the operator executed, such as a memo or ticket number (ASM-099). |
| `reason` | Free text, mandatory. |
| `previous_version` | The version it replaces, with its values, so the change record carries old and new value together. |
| `source_ip`, `channel`, `user_agent` | Where the change came from (A08). |

**Store (closes PD-074).** Thresholds live in a DynamoDB table, `thresholds`, in the workload account of each environment, encrypted with the `records` customer-managed key (ASM-097), with point-in-time recovery. Its partition key is `rule_key` and its sort key `effective_from`, so the version in force at any instant is one query: the latest item whose `effective_from` is at or before that instant. A head item per `rule_key` holds the current version number for optimistic concurrency. DynamoDB is chosen because the telemetry and transactional stores already use it (ADR-003, A09 section 7.2), so no new service, key or backup scheme is added; because a version write and its audit record fit in one transaction; and because the table is small (ten pilot rows plus any farm overrides) and read once per reading.

**Scope resolution.** For a reading of farm F, the function uses the active farm-scope rule of F for that zone and variable if one exists, otherwise the pilot rule. NPK is pilot-wide (ASM-054); the farm scope exists so that the Federation can correct a range for one farm without touching the other 199. If the Federation answers Q-08 with a department granularity, a `department` scope is one more step in the same resolution, with no change to the reading path.

### 4.3 How a threshold is changed

Only the platform operator writes thresholds, on the Federation's instruction (ASM-099), through the threshold screen of the reports web. The operation requires a `staff` token with MFA whose `auth_time` is at most 10 minutes old (ASM-092); otherwise the screen asks for the TOTP code again. The farmer reads the ranges applied to their own farms and the agronomist reads all ranges of their departments (A09 section 4.2); neither can change them.

1. The operator selects the rule, enters the new values, the unit, the Federation reference and the reason.
2. The threshold service validates the input: `lower` below `upper`, unit consistent with the variable, the recommended band inside the alert range, the closure tolerance not negative.
3. In one DynamoDB `TransactWriteItems` call, the service puts the new version item, which carries every field of section 4.2 including the previous values, and updates the head item on the condition that its version is still the one the operator saw. A concurrent change makes the condition fail and the operator reloads; two changes can never both claim the same version.
4. The new version is in force from its `effective_from`. Readings taken from then on are evaluated with it; readings taken before keep the version that was in force when they were taken (section 5).

A change never re-evaluates readings already evaluated, never deletes a version and never reaches a device: the nodes hold no threshold, so a change needs no firmware update, no device shadow and no visit ("No quedan fijos en el dispositivo").

### 4.4 How the threshold function reads it

The function reads the threshold table on every evaluation, with a strongly consistent query, and holds no cache. At 3,600 evaluations on a harvest day (section 3.1) the extra reads are negligible, and the absence of a cache means a change applies to the next reading taken after it without any propagation delay to explain. The function's role has read-only access to `thresholds` and decrypt on the `records` key; it cannot write a threshold. Every evaluation result and every alert records the `rule_key` and `version` it used.

### 4.5 Audit of a threshold change

A08 requires the application record to be authoritative and written in the same transaction as the change; a CloudTrail data event on the store is the secondary source.

| Layer | Design |
|---|---|
| Authoritative record | The version item itself. It is written in the same `TransactWriteItems` as the head update (section 4.3), so a change cannot exist without its who (`changed_by`, `changed_by_role`), what (zone, variable, scope, old and new values, reason, Federation reference), when (`effective_from`, server UTC) and from where (`source_ip`, `channel`, `user_agent`), which are the A08 fields. |
| Copy to the audit trail | A DynamoDB Streams consumer of `thresholds` writes each new version item as a structured audit record to the `pdn` application audit log group, from which the A08 subscription delivers it to the audit archive under Object Lock (A08 section 7.3). If the consumer fails, the stream keeps the record for retry and the version item stays authoritative. |
| Secondary source | The organization trail adds CloudTrail data events on the `thresholds` table of `pdn`, with an advanced event selector by table ARN restricted to write events (`readOnly` false). Writes are rare, so this costs little, while logging the function's 3,600 daily reads would add cost and no audit value. The trail shows the service role of the threshold service, not the person (A08 section 7), which is why it is secondary. |
| Retention | The audit record 5 years (ASM-067). Version items are kept 5 years after they are superseded, so any alert in the 5-year telemetry history can still be traced to its version (ASM-137). |

## 5. Evaluation of a reading

The threshold function is the Part E telemetry trigger ("Lote de telemetría. La función marca la lectura fuera de umbral y persiste el resultado"), invoked by the topic rule for every reading. Its implementation, retries and failure queue are designed by A17 (PD-014, PD-135); this section fixes the contract it implements.

| Step | Behavior |
|---|---|
| 1. Identify | Take the node identifier from the topic and the payload; look up the node registry for its farm, its zone and its status. A reading of a node that is not `active` is not evaluated (it is still stored by the rule, and A09 revocation stops new ones). |
| 2. Choose the version | For each variable in the reading, resolve the rule (section 4.2) and take the version in force at the reading's original timestamp, not at arrival time. A drained reading is therefore evaluated exactly as it would have been live, which is the A10 principle that "A drained record is evaluated as if it had arrived live" (A10 section 2). |
| 3. Classify | Each variable is `in_range`, `below`, `above`, `closure_reached` or `not_evaluated` (no published range, unit mismatch, or value outside the sensor's physical range). |
| 4. Persist | Out-of-range and closure results are written to the alert store as evaluation items keyed by node, variable and reading timestamp, with a condition that the item does not exist. A duplicate delivery therefore finds the item and stops, so it never opens a second episode or sends a second notification. |
| 5. Update the episode | Every classified variable, in range or not, updates the state item of its node and variable, which holds the open episode, if any, and the timestamp of the latest in-range reading; section 6.2 gives the rules. |
| 6. Emit | Only the evaluation that opens, reminds or closes an episode publishes a message to the SNS alert topic (section 7.3). |

The function never updates the telemetry item and writes only to the alert store. The ordering and coupling that A13 left open in PD-112 was later closed by the inference design: the rule sends the reading key to the threshold queue, and the function evaluates a reading only after it reads the stored item with a strongly consistent read, so a reading is evaluated only once it is persisted ([inference functions](inference-functions.md#64-ordering-and-failure-coupling-of-the-telemetry-rule-closes-pd-112) section 6.4, ASM-161). The contract of this section is unchanged by that choice: the result of an evaluation is the same whichever rule action completes first.

## 6. Alerts and deduplication

### 6.1 Why deduplicate

A crop variable that leaves its range usually stays out for hours, and a drying lot above 40 °C stays hot until the farmer acts. Without deduplication the farmer would receive one message every 4 hours per crop variable, or every hour for the drying lot, and a 24-hour drain could deliver up to 24 messages at once. Each message would also produce a `crop_alert_raised` event and so a new pest case in ADR-002. The unit of alerting is therefore the episode, not the reading.

### 6.2 Episode model

An episode is identified by node, variable and condition (`below`, `above` or `closure_reached`); a reading that moves from below to above closes one episode and opens another (ASM-135).

| Event | Rule |
|---|---|
| Open | The first out-of-range reading for a key with no open episode opens one, with a conditional put, so that two concurrent evaluations of a drain cannot both open it. The winner emits the notification; the loser attaches its reading. |
| Attach | Further out-of-range readings of the same key attach to the open episode: they update the reading count, the latest value and the extreme value, and send nothing. |
| Close | The episode closes on the first in-range reading whose timestamp is later than the latest out-of-range reading of the episode. An older in-range reading arriving from the buffer never closes an episode opened by a newer reading. Conversely, an out-of-range reading older than the latest in-range reading in the state item opens an episode that is already over (section 6.4). |
| Reminder | If an episode is still open 24 hours after it opened, one reminder is sent by push, and then once per further 24 hours (ASM-135). |
| Closure notice | `closure_reached` is not an alert (ASM-059). It produces one notice per drying lot, the first time coffee moisture reaches the closure value plus tolerance, and never repeats in that lot. |
| pH | A soil analysis entered for a farm (ASM-132) is evaluated by the same contract when it is recorded; its episode closes with the next analysis in range. |

### 6.3 What an episode produces

| Episode | Persisted event | Effect beyond the farmer notification |
|---|---|---|
| Crop variable from a crop node opens | `crop_alert_raised`, once per episode | ADR-002: opens a pest case for the plot that contains the node and switches it to precision mode; the drone confirmation flight goes to the node location. The visible symptom of the plant enters through that confirmation service, not through a separate weekly check, as Part A requires. |
| pH from a soil analysis opens | `soil_ph_alert_raised` | Notification only. It has no node location, so it is not a focus and does not open a pest case. |
| Grain temperature above 40 °C opens | `drying_alert_raised` | Notification only; the drying zone is not a plot of the drone service. |
| Drying moisture reaches closure | `drying_closure_notice` | Marks the drying lot as ready to close (section 6.5). |
| Any episode closes | `alert_resolved` | The episode shows as resolved in the app. |
| Gap report received | `telemetry_gap` | Records the number of readings the node evicted after 24 hours (A10 section 4.3), so the history shows a loss rather than an empty period. |

### 6.4 Readings drained from the buffer

Drained readings enter by the same topic, rule, store and function as live ones (A10 section 4.3), so they are evaluated and deduplicated in the same way, and the case statement's rule is met: "Sin señal, la alerta se emite al drenar el buffer, dentro de las 24 horas". Three behaviors are specific to them.

| Behavior | Design |
|---|---|
| Marked as delayed | A reading is `delayed` when its receive time is more than 300 seconds after its original timestamp, the case statement's live limit. The alert carries both times, and the farmer sees "measured at" and "received at" (section 8), so a delayed alert is never mistaken for a live one. A21 measures delayed alerts against the 24-hour limit and live alerts against the 5-minute limit (A10 section 8.3). |
| One notification per drain | Because the node drains oldest first, the first drained out-of-range reading of a key opens the episode and notifies immediately; the following drained readings of the same key attach to it. A day out of range drained at once is one notification, not 24. |
| Episode already over | If a live reading taken at reconnection was in range and was published first (A10 section 4.3), the drained out-of-range readings are older than that in-range reading, so their episode is opened already over and its later drained readings attach to it in timestamp order. The farmer still receives one notification that says the value was out of range from the first to the last affected reading and is now back in range, because the case statement requires the alert to be emitted at drain. |

A node silent for more than 24 hours is not an alert to the farmer: it is detected by the observability of point 6 ("Nodos y fincas sin telemetría por más de 24 horas"), designed by A21. The farmer app shows the last time each node was heard from (section 8).

### 6.5 Drying lot

The case statement measures drying "mientras el lote está en secado", but does not say how the node knows a lot is drying, and the farmer's phone usually has no data at the farm. The drying node therefore has a local start and stop control: the farmer presses it when spreading a lot and when collecting it, and the node then reads hourly while the lot is open and stops when it is closed (ASM-130). Each reading carries a per-node lot sequence number. The platform opens a drying lot record at the first reading of a new sequence number and closes it on the node's stop marker, or marks it `stale` if no reading has arrived for 24 hours.

| Lot field | Content |
|---|---|
| `node_id`, `lot_seq` | Key of the lot. |
| `farm_id` | From the node registry. |
| `started_at`, `ended_at` | Original timestamps of the first reading and of the stop marker. |
| `closure_notice_at` | When the closure notice was raised, if it was. |
| `final_coffee_moisture` | Coffee moisture of the last reading of the lot. |
| `grain_temperature_min`, `_mean`, `_max` | Over the lot's readings. |
| `alerts` | Identifiers of the grain-temperature episodes of the lot. |

These summary fields are exactly the drying variables of the buyer view of a sack (ASM-096), so A16 links each sack to its drying lot and reads them from this record rather than from raw telemetry.

### 6.6 Stores

| Store | Key | Retention | KMS key |
|---|---|---|---|
| Telemetry table (ADR-003, A13) | Node identifier, original reading timestamp | 5 years from the reading, by TTL (A09 section 9.2) | `telemetry` |
| Daily summaries (section 9) | Node identifier, Colombia day | 5 years | `telemetry` |
| `thresholds` (section 4.2) | `rule_key`, `effective_from` | 5 years after supersession (ASM-137) | `records` |
| Alert store: evaluation items, episodes, drying lots, soil analyses | Node (or farm for pH) and variable; lots by node and lot sequence | 5 years from the event, by TTL (ASM-098) | `records` |
| SNS alert topic, notification queue and dead-letter queue | Messages in flight | Queue retention only | `records` |

## 7. Farmer notification channel

### 7.1 Decision (closes PD-085)

**Both channels: an app push notification and an SMS for every alert opening and every closure notice; push only for reminders and resolutions** (ASM-136).

| Criterion | Push only | SMS only | Push and SMS (chosen) |
|---|---|---|---|
| Phone without a data connection | Not delivered until the phone has data, which the case statement places at the municipal seat or on a main road. | Delivered whenever the phone has any cellular signal, including a phone without a data balance. | SMS reaches the farmer on cellular signal; push fills the app when data returns. |
| Few screens, distrust of leaving the notebook | Needs the app to be installed and opened. | Readable without opening the app, in plain text. | The SMS is the alert; the app adds the history behind it without a new screen. |
| Content | Rich: values, range, chart link inside the app. | Short text only. | Each channel carries what it can. |
| Generation-neutral rule (A10 section 11) | Rides on any data bearer. | Must work over LTE or later without a 2G or 3G circuit-switched fallback; this is a procurement and verification criterion, not a property assumed here (PD-130). | Push keeps the alert path working even where SMS over LTE is not confirmed. |
| Delivery evidence for the SLO | App receipt when the phone has data. | Carrier delivery receipt. | The first receipt from either channel marks the alert delivered. |
| Cost | No per-message fee. | Per-message fee. | SMS limited by deduplication to openings and closure notices; A23 prices it (PD-133). |

Push alone would fail exactly the farmer the case statement describes, one with data only in town. SMS alone would lose the history and the "measured at" detail that make a delayed alert understandable and would cost a message for every resolution. Sending both for the actions that need the farmer, and push only for the rest, gives the farmer the alert on the most available bearer and keeps the SMS volume bounded by episodes rather than by readings.

### 7.2 Latency within the budget

The A10 budget allows 30 seconds for hop 6, alert topic to channel provider, and 60 seconds for hop 7, provider to phone (A10 section 8.2). The dispatcher below sits in hop 6, and both channel providers sit in hop 7; they are sent in parallel, so the slower one does not delay the faster. These remain design targets (ASM-075), to be validated by the alert-delivery SLO of A21 (PD-082).

### 7.3 Path

| Step | Component | Design |
|---|---|---|
| 1 | SNS alert topic (ADR-003, A13) | The threshold function publishes one message per episode opening, closure notice, reminder or resolution: episode identifier, farm, node, zone, variable, value, range and version, reading and receive time, `delayed` flag. The message carries no name and no phone number. |
| 2 | Notification queue | An SQS queue subscribed to the topic, with a dead-letter queue, decouples the provider calls from the function and keeps a message if a provider is unavailable. |
| 3 | Notification dispatcher function | Resolves the farmer of the farm, reads the registered phone number from the personal-data table and the app's push endpoint, renders the message and sends both channels. Its role reads only those two items and the alert store, with decrypt on `personal` and `records`; the threshold function never sees a phone number. |
| 4a | Push | Amazon SNS mobile push to the farmer app's platform endpoint (Firebase Cloud Messaging for Android, Apple Push Notification service for iOS). |
| 4b | SMS | SMS through AWS End User Messaging SMS to the phone number registered at enrollment, the same number that receives the sign-in code (ASM-091). The origination identity that Colombia requires is verified before procurement (PD-130). |
| 5 | Delivery record | The dispatcher records the send time per channel; the SMS delivery receipt and the app's receipt update the episode. An alert without a receipt is an undelivered alert for point 6 ("Alertas de cultivo no entregadas"), which A21 observes (PD-134). |

SMS coalescing: the first SMS for a farm is sent at once; further alert openings of the same farm within the next 10 minutes, as during a drain, are combined into one follow-up SMS, while each still gets its own push (ASM-135). A farmer whose app is not installed or has no push endpoint receives SMS only.

The push provider credentials of the SNS platform application are the one secret on this path. Following ADR-003, they live in AWS Secrets Manager and never in the repository or a Terraform state; PD-130 checks whether the platform application can be created with a write-only argument, and otherwise it is created operationally from the secret.

## 8. What the farmer sees

The farmer app has few screens ("La aplicación del caficultor tiene pocas pantallas"), so monitoring adds three views and no settings: the farmer configures nothing, because the Federation owns the ranges. The views below define content only; layout, color of the crop alert and icons belong to the design system task, and farmer-facing text is rendered in Spanish there.

| View | Content |
|---|---|
| Farm status | One card per zone with a status word: normal, alert, drying (with the open lot), or no data since a given time. The last time each node was heard from. Open alerts first. |
| Alerts | One line per episode, newest first: variable, value, the range it left, "measured at" and "received at" when delayed, open or resolved, and for a drying lot the closure notice. |
| History | Per variable, a chart of the last 24 hours, 7 days, 30 days or the harvest, with the alert range as a band and the recommended band when one exists (ASM-134). Gaps are drawn as gaps, labeled "no data received" or "readings lost after 24 hours offline" from the gap report. Drying lots show coffee moisture falling toward 12 % and grain temperature against 40 °C. pH shows the soil analyses as points. |

Example alert text, in English here:

| Channel | Text |
|---|---|
| SMS, live alert | "Aroma Pacifico: farm <alias>, soil moisture 34 % (range 40-80 %), measured 14:02. Check the app." |
| SMS, delayed alert | "Aroma Pacifico: farm <alias>, drying grain 43 C (limit 40 C), measured yesterday 23:10, received 06:15." |
| SMS, closure notice | "Aroma Pacifico: farm <alias>, drying lot at 12 % moisture. The lot can be closed." |
| Push, resolution | "Soil moisture back in range (52 %) at 18:02." |

Messages carry the farm alias chosen at enrollment, never the farmer's name, national identity number or coordinates, and no link. The app keeps the last synchronized history on the phone, so the farmer can read it at the farm without data. Times are shown in Colombia time (ASM-137).

## 9. History and the pilot report

Both readers read persisted data; neither reads from the devices or the rule.

| Reader | Reads | Path | Scope (A09 section 4.2) |
|---|---|---|---|
| Farmer app, history | Raw readings for up to 30 days; daily summaries beyond; the farm's episodes, drying lots and soil analyses. | Application API, query by node identifier and time range on the telemetry table, which its key serves directly; no scan. | Own farms. |
| Pilot report (reports web) | Daily summaries, episodes, drying lots, gap records and threshold versions; never raw readings of a named farmer. | Application API over the summary and alert stores. | Agronomist by department; platform operator operational metadata only (counts, last seen). |

A daily summary function, started by Amazon EventBridge Scheduler after midnight Colombia time, writes one item per node and day: readings received, readings expected, readings lost from gap reports, and the minimum, mean and maximum of each variable. Expected readings are 6 per crop node and 24 per drying node for each day its lot was open, so data completeness is a ratio the report can compute without any figure outside the case statement.

| Pilot report section | Measure |
|---|---|
| Coverage | Farms and nodes reporting per department; readings received against expected; readings lost after 24 hours offline. |
| Alerts | Episodes opened per variable and department; median time open; share delayed; share delivered within 300 seconds (from the A21 SLI). |
| Drying | Lots opened and closed; closure notices; final coffee moisture distribution; grain-temperature episodes. |
| Thresholds | Versions in force and changes in the period, from the threshold store. |

**Retention.** Readings 5 years, as the case statement requires; daily summaries, episodes, drying lots and soil analyses 5 years from their date (ASM-098); threshold versions 5 years after supersession (ASM-137). Deletion is by DynamoDB TTL, and point-in-time recovery limits any remaining copy to 35 days (A09 section 9.2). The data stays in `us-east-1` (ASM-002).

## 10. Observability hooks

A21 designs observability; this design guarantees that the signals it needs exist.

| Point 6 requirement | Signal produced here |
|---|---|
| "Nodos y fincas sin telemetría por más de 24 horas" | Last reading per node in the telemetry table; decommissioned nodes excluded by registry status (A09 section 10). |
| "Alertas de cultivo no entregadas" | Episodes without a delivery receipt after 300 seconds (live) or after the receipt window A21 sets for delayed alerts; messages in the notification dead-letter queue. |
| SLO "entrega de la alerta de cultivo" | Reading, receive, evaluation, publication, send and receipt timestamps on each episode, the trace points of A10 section 8.3. |
| Ingestion metrics and errors | Topic rule error queue (ADR-003); threshold function failures and their failure queue (A17, PD-135); readings `not_evaluated` by cause. |

## 11. Compliance with the issue

| Acceptance criterion of aroma-pacifico/aroma-pacifico-roadmap#14 | Where it is met |
|---|---|
| The design covers 200 farms with 6 crop readings per day. | Section 3.1 (1,200 per day, 50 per hour), 3.2 (peaks), 3.3 (5 years), 3.4 (1,000 farms). |
| The design covers, during harvest, 100 farms with 24 drying readings per day. | Section 3.1 (2,400 per day, 3,600 harvest-day total), 3.2 and 3.4. |
| Crop and drying thresholds are configurable by the Federation and not fixed on the device. | Section 4: versioned threshold store, change procedure, audit, and the function reading it; the node holds no threshold. |
| History, alerts and handling of the no-signal case with a buffer of up to 24 hours are included. | Sections 6 (episodes, drained readings, gaps), 7 (channel), 8 (farmer views) and 9 (history, pilot report, 5-year retention). |
| Closing the drying batch near 12 % moisture and the alert for grain above 40 °C are reflected. | Section 4.1 rows T-09 and T-10, section 6.2 (closure notice), 6.3 and 6.5 (drying lot). |

## 12. Decision-log entries

| ID | Summary |
|---|---|
| PD-074 | Threshold store and its secondary audit source. Closed by sections 4.2 and 4.5. |
| PD-085 | Farmer alert channel. Closed by section 7: push and SMS. |
| ASM-130 | Drying lots are opened and closed with a local control on the drying node. |
| ASM-131 | Crop moisture and temperature ranges apply to air and soil as separate rows. |
| ASM-132 | Soil analysis pH is entered by the agronomist with the laboratory reference. |
| ASM-133 | Initial closure tolerance is zero. |
| ASM-134 | Recommended bands are displayed, not alerted. |
| ASM-135 | Episode deduplication, reminders and SMS coalescing. |
| ASM-136 | Push and SMS channel; SMS to the enrollment phone; SMS over LTE as a procurement criterion. |
| ASM-137 | Threshold versions kept 5 years after supersession; Colombia time for display and daily summaries. |
| ASM-138 | Threshold scopes, version in force at the reading timestamp, no backdating. |
| PD-130 | Verify AWS quotas and features used here against live documentation, and the SMS origination identity for Colombia. |
| PD-131 | The Federation publishes NPK ranges and units and the closure tolerance. |
| PD-132 | The A09 role matrix adds the soil analysis object and the dispatcher's read of the farmer phone. |
| PD-133 | A23 prices SMS and push with its own alert-rate assumption, and the drying node control. |
| PD-134 | A21 adopts the delivery definition and the undelivered-alert signal. |
| PD-135 | A17 implements the evaluation contract of section 5 in the threshold function. |

## 13. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Pilot figures, table of initial ranges, assumptions, restrictions, Part A, Part E, points 1, 2 and 6. |
| [`docs/decision-log.md`](decision-log.md) | ASM-001, ASM-002, ASM-024, ASM-036, ASM-053, ASM-054, ASM-059, ASM-067, ASM-070, ASM-074, ASM-075, ASM-091, ASM-092, ASM-096 to ASM-099, and the entries of section 12. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | `crop_alert_raised`, pest case and mode switch. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | Ingestion pattern, topic rule, telemetry table, alert topic, thresholds as data. |
| [`docs/connectivity.md`](connectivity.md) | Sections 2, 4.2, 4.3, 5, 8 and 11. |
| [`docs/landing-zone.md`](landing-zone.md) | Section 7, audit of the threshold change and the telemetry publication. |
| [`docs/security-and-identity.md`](security-and-identity.md) | Sections 3, 4, 7, 9 and 10. |
| [`docs/clarification-questions.md`](clarification-questions.md) | Q-07, Q-08 and Q-14 and their fallbacks. |
| `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13, aroma-pacifico/aroma-pacifico-diagrams#2) | Crop and drying data path; PD-112. |
