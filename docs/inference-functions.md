# Event-Driven Inference Functions

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / SR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#17 (A17) |
| Builds on | [ADR-001](adr/adr-001-primary-and-second-cloud.md) (Region, Region availability check), [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) (services, events, flight volumes), [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) (ingestion pattern, routing rule), [`docs/drone-service.md`](drone-service.md) (A15, sections 3 to 8), [`docs/farm-monitoring.md`](farm-monitoring.md) (A14, sections 3, 5, 6 and 10), [`docs/sack-and-trip.md`](sack-and-trip.md) (A16), [`docs/security-and-identity.md`](security-and-identity.md) (A09, sections 4, 7 and 9), [`docs/landing-zone.md`](landing-zone.md) (A08, sections 3.1 and 5), [`docs/connectivity.md`](connectivity.md) (A10, section 8.2), `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13) |
| Feeds | A19 architecture document and complete diagram (aroma-pacifico/aroma-pacifico-roadmap#19), A21 observability (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36), B09 ingestion module (aroma-pacifico/aroma-pacifico-roadmap#29), B10 function pattern (aroma-pacifico/aroma-pacifico-roadmap#30) |
| Decision-log entries | PD-013, PD-014, PD-015, PD-111, PD-112 and PD-135 (closed here), PD-018 (stays open), ASM-160 to ASM-169, PD-160 to PD-167 |

## Diagrams

- [Event-Driven Inference Functions Architecture (SVG)](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/flows/inference-functions.svg) ([PNG](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/flows/inference-functions.png)): Architecturally diagrams the four event-driven inference triggers, Amazon Bedrock foundation model invocation, SQS dead-letter queues, and on-device person screening.

## 1. Purpose and requirements

Part E of the [case statement](case/case-statement.pdf) makes inference event-driven: "La función se invoca con un disparador y termina al persistir el resultado. [...] No hay cómputo persistente en espera de objetos." It also separates the function from the store: "La función no es el almacén. Lee el objeto ya persistido, llama a un servicio de inferencia administrado y escribe el resultado en el expediente." This document designs the four functions that Part E names, and is the function design that A13, A14 and A15 defer to (PD-014, PD-111, PD-112, PD-135). Quotations are in the original Spanish, which prevails over the English rendering (ASM-001). No figure in this document is a measurement; every count is arithmetic on the case figures and every other value is a design value recorded in the decision log.

| Requirement of Part E | Where it is met |
|---|---|
| Four triggers: "Objeto de confirmación del foco", "Objeto de verificación", "Objeto del vuelo de estimación", "Lote de telemetría". | Sections 3 and 4 |
| The function reads the already persisted object, calls managed inference and writes the result in the same record, "con marca de tiempo, confianza e identificador del objeto de origen". | Sections 4 and 5 |
| Confirmation "No autoriza la fumigación"; estimation "No genera una orden de empaque"; telemetry "Las reglas configurables siguen vigentes"; "La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta." | Section 4, column "Never does" |
| "Si la invocación falla, el objeto de origen permanece y el estado del caso no cambia. El diseño define reintentos, cola de fallos y dónde se observa el error." | Sections 6 and 7 |
| "No se entrena un modelo. Se usa inferencia administrada. El diseño indica la región del servicio, qué objetos salen del país y el costo por mil invocaciones en cosecha." | Sections 2, 8 and 9 |
| "Los objetos son del cultivo. Si el objeto contiene una persona, no se envía a inferencia y se documenta el tratamiento del archivo." | Section 10 |
| Epic 6: "Patrón de una función: disparador, permiso de lectura del objeto, permiso de escritura del resultado y permiso de invocación al servicio de inferencia." | Section 11 |

## 2. Managed inference service

### 2.1 Choice

**Amazon Bedrock, invoked on demand in `us-east-1` with one vision-capable foundation model and versioned zero-shot prompts** (ASM-160). The initial model is Amazon Nova Pro, the model that ADR-001 recorded as offering in-Region inference in `us-east-1`; the model identifier is a configuration value of each function, so a later model change is a configuration change with its own release note, not a redesign. The function calls the Bedrock Runtime `Converse` operation with the images (or, for telemetry, the readings) as content blocks, a system prompt held as versioned configuration data, and a requested JSON output schema. Nothing is trained, fine-tuned, imported or provisioned: no custom model, no provisioned throughput, no model endpoint. Bedrock is billed per request in tokens and keeps no compute waiting for objects, which is what Part E asks for.

| Option | No training | Managed, no persistent compute | Can judge a coffee pest focus or a crop load | Verdict |
|---|---|---|---|---|
| Amazon Bedrock foundation model, zero-shot prompt (chosen) | Yes: the prompt describes the task; the model is used as published. | Yes: per-request invocation, no endpoint to run. | Yes, as a described visual task; quality is a future test (PD-167). | Chosen. |
| Amazon Rekognition `DetectLabels` | Yes | Yes | No: it returns a generic label taxonomy (for example plant or leaf), not the presence of a specific pest or disease on a coffee plant. | Rejected for crop judgement. Also rejected as a person screen (section 10.2). |
| Amazon Rekognition Custom Labels | No: a Custom Labels model must be trained on labelled images before it can be used. | No: a trained model must be started and is billed while running. | Yes, after training. | Rejected: "No se entrena un modelo" and "No hay cómputo persistente". |
| Amazon SageMaker AI endpoint with a pre-trained model | Possibly | No: a real-time endpoint is persistent compute, and the platform would operate the model. | Depends on the model. | Rejected: persistent compute and not a managed inference service in the sense of Part E. |

### 2.2 How each function uses the model

| Function | Input to the model | Output schema requested from the model |
|---|---|---|
| Confirmation | Up to 8 focus photos of the confirmation mission (ASM-168), the focus coordinates and the alert variables of the case. | `pest_signs_present` (true or false), `signs` (free text), `person_detected` (true or false), `confidence` in [0, 1] per image and overall. |
| Verification | Up to 8 photos or frames of the short pass of the verification mission (ASM-168), and the confirmation result of the case for comparison. | `active_signs_at_focus` (true or false), `person_detected`, `confidence`. |
| Estimation | Up to 20 photos sampled across the plots and the plot map of the estimation mission (ASM-168). | Per plot: `productive_cover_fraction` in [0, 1], `crop_load_class` (`low`, `medium`, `high`), `person_detected`, `confidence`. |
| Telemetry threshold | The reading that opened an alert episode, the node's readings of the previous 24 hours read from the telemetry table, and the threshold version applied (ASM-167). | `plausibility` (`field_condition`, `possible_sensor_fault`, `indeterminate`), `confidence`. |

The `confidence` is the score the model reports in the requested schema. A foundation model does not return a calibrated probability, so the platform treats the score as an uncalibrated self-report, records it with the model identifier and the prompt version, and uses it only against a Federation-configured decision threshold (ASM-163). Calibrating it against the agronomist's own review of the same photos is a future test in `qa`, not a result (PD-167).

## 3. The four triggers

Every function follows the same shape: an event that points to a record already persisted, an Amazon SQS queue per function in front of an AWS Lambda function, a read of the persisted object or record, one Bedrock call, and one conditional write of the result into the record the case statement calls the "expediente". The queue is what gives each function its own retries with backoff and its own dead-letter queue (section 6).

| Trigger of Part E | Event that starts it | AWS delivery path to the function | Batch per invocation |
|---|---|---|---|
| Objeto de confirmación del foco | `mission_record_persisted` for a `confirmation` mission (ASM-149): the reception function of A15 has persisted the flight record and every object of its manifest. | The mission table's DynamoDB stream feeds one Amazon EventBridge Pipe that keeps only the change of a mission into `record_persisted` and puts a `mission_record_persisted` event, with `mission_id`, `mission_type`, `farm_id`, `case_id` and `order_id`, on the custom event bus of the drone service. An EventBridge rule per mission type sends the event to the function's SQS queue. A Lambda event source mapping reads the queue (ASM-161). | 1 message, that is one mission. |
| Objeto de verificación | `mission_record_persisted` for a `verification` mission. | Same path, rule on `mission_type = verification`. | 1 mission. |
| Objeto del vuelo de estimación | `mission_record_persisted` for an `estimation` mission. | Same path, rule on `mission_type = estimation`. | 1 mission. |
| Lote de telemetría | A reading accepted by AWS IoT Core on the telemetry topic (ADR-003). | The telemetry topic rule keeps its two actions: the DynamoDB write of the reading to the telemetry table, and, in place of a direct Lambda invocation, an SQS action that sends the reading key and payload to the threshold function's queue. A Lambda event source mapping reads the queue; the function then reads the stored reading with a strongly consistent `GetItem` before it evaluates it (ASM-161, section 4.4). | Up to 10 readings, the "lote": a live reading usually arrives alone, a drained buffer fills the batch. |

The drone path uses the stream of the mission table rather than an S3 event notification because the trigger is not the arrival of a raw object but the linked, complete mission (ASM-149): the reception function writes `record_persisted` once, in its own transaction, and the stream turns that write into exactly the event the functions need, without a second write the reception function could lose. One pipe reads the stream, so the table has a single stream reader, and the event bus lets A21 and later consumers subscribe without touching the functions.

## 4. Function design

### 4.1 Common contract

| Step | Behavior |
|---|---|
| 1. Receive | Take the event from the queue. The event carries identifiers only, never an object or personal data. |
| 2. Check for a prior result | Look up the result item by its idempotency key (section 6.3). If it exists, delete the message and stop: a duplicated or retried event never calls the model again and never writes twice. |
| 3. Check the guard | Read the case, order, mission or node and check the guard of section 4.2 to 4.4. A failed guard is not an error: the function writes nothing, records a `guard_skipped` metric with the reason and deletes the message. |
| 4. Read the persisted input | Read the objects listed in the mission's persistence receipt (key, version identifier and SHA-256) with `GetObject` on that exact version, and verify the checksum; for telemetry, read the stored reading and its 24-hour context. Objects whose manifest entry is not `person_screen = clear` are excluded before any call (section 10). |
| 5. Invoke | One Bedrock `Converse` call on the model of ASM-160 in `us-east-1`, with the prompt version of the function. |
| 6. Persist | One DynamoDB `TransactWriteItems`: the result item with the common fields of section 5, conditional on its idempotency key not existing, and, where the function may change state, the state transition with the guard of A15 as a condition. Either both are written or neither is. |
| 7. Finish | Delete the message and emit the metrics of section 7. The function ends when the result is persisted, as Part E requires. |

The function role has no `s3:PutObject` or `s3:DeleteObject` on the drone object bucket and no write on the telemetry table, so no function can alter, move or delete its source. That is the mechanism behind "el objeto de origen permanece".

### 4.2 Confirmation function

| Aspect | Design |
|---|---|
| Reads | The `photos/` objects of the confirmation mission listed in its receipt, by version, with `person_screen = clear`. |
| Guard | Case in `confirming`; the mission is a `confirmation` mission of this case with `record_persisted`. |
| Decision | `focus_confirmed` if the model reports `pest_signs_present = true` with overall confidence at or above the confirmation threshold; otherwise `focus_not_confirmed`, with the reason `no_signs` or `below_threshold`. The threshold is Federation-configured data, initially 0.70 (ASM-163). If no image is usable (all excluded, or the model reports a person), the outcome is `not_evaluable` (section 10.3). |
| Writes | `confirmation_result` on the pest case (A15 section 4.1) with the fields of section 5, and the transition `confirming` to `confirmed` or `not_confirmed` in the same transaction, conditional on the case still being in `confirming`. `not_evaluable` writes the result and no transition. |
| State it may cause | `confirmed`, which lets the platform create the draft fumigation order (A15 section 5.2); or `not_confirmed`, which hands the case to the agronomist. |
| Never does | Authorize, reject or create a fumigation order. The function role has no permission on the authorize operation, and `authorizer_id` is never writable by a function role (A09 section 4.3). |

### 4.3 Verification function

| Aspect | Design |
|---|---|
| Reads | The `photos/` objects of the verification mission, including the frames of a short pass (ASM-168), by version, with `person_screen = clear`; and the `track/` geometry of the mission, only to check that the pass covered the focus. The track is not sent to the model. |
| Guard | Case in `verifying`; order of the mission in `executed`; the mission is a `verification` mission of this order with `record_persisted`. |
| Decision | `can_close` if the pass covered the focus and the model reports `active_signs_at_focus = false` with confidence at or above the verification threshold, initially 0.70 (ASM-163); otherwise `cannot_close`, with the reason `signs_present`, `below_threshold` or `focus_not_covered`. `not_evaluable` as in section 4.2. |
| Writes | A new entry in `verification_results` of the case, and in the same transaction: for `can_close`, case `verifying` to `closed` and order `executed` to `closed`; for `cannot_close`, case `verifying` to `verification_not_closable` and order `executed` to `closed` (ASM-141). `not_evaluable` writes the result and no transition. |
| State it may cause | Case `closed` or `verification_not_closable`; order `closed`. |
| Never does | Close a case without a persisted verification object, or change state when the object did not arrive or the call failed: then no result exists and the case stays in `verifying` ("Si el objeto no llega o la invocación falla, el estado no cambia"). |

### 4.4 Estimation function

| Aspect | Design |
|---|---|
| Reads | The `photos/` and `maps/` objects of the estimation mission, by version, with `person_screen = clear`; the active polygon version of the farm for the plot areas. |
| Guard | Farm has an active polygon version; the mission is an `estimation` mission of a review campaign with `record_persisted`. |
| Decision | For each plot, `estimated_sacks = area_ha x productive_cover_fraction x yield_factor(crop_load_class)`, where the yield factor in sacks of 60 kg per hectare and load class is Federation-configured data; the farm estimate is the sum over its plots, with a low and a high bound from the lowest and highest class factor (ASM-166). Until the Federation publishes the factors, the function stores the model outputs and the result `estimate_pending_factors`, and the center sees no number. |
| Writes | The harvest estimate record of the farm and campaign, with the fields of section 5, and the collection center of the farm's department; the center's expected-sacks view is the sum of the latest estimate of each farm for the harvest period. |
| State it may cause | None. The estimate is information for the collection center. |
| Never does | Create a packing order, a sack, a code batch or any sack event. The function role has no permission on the sack, trip or code tables (A16). |

### 4.5 Telemetry threshold function

The telemetry function implements the evaluation contract of A14 section 5 without change, and adds the managed inference call that Part E requires of every function in a way that cannot weaken the configurable rules.

| Aspect | Design |
|---|---|
| Reads | Each reading of the batch from the telemetry table by its key (node identifier and original reading timestamp) with a strongly consistent read; if the item is not yet there, the read is retried 3 times within the invocation (after 200, 400 and 800 ms) and then the reading is returned to the queue as a failed item (section 6.1). For a reading that opens an episode, the node's readings of the previous 24 hours from the same table. |
| Guard | Node `active` in the registry (A14 section 5, step 1). |
| Decision | The Federation's rules decide, exactly as A14 section 5 specifies: version in force at the reading timestamp, classification, evaluation item written with a condition that it does not exist, episode update, and publication to the SNS alert topic only on episode opening, reminder, closure notice or resolution. The rule result is a comparison, so its confidence is 1.0 with `confidence_basis = rule`. |
| Inference | Only for a reading that opens an episode, and only after its alert message has been published, the function calls Bedrock with the reading, its 24-hour context and the threshold version, and writes the `plausibility` label and model confidence onto the opening evaluation item (ASM-167). The agronomist sees it next to the case. It never suppresses, delays or reclassifies an alert, and never changes the case. If the call fails after the SDK retries, the item records `plausibility = unavailable` and the batch does not fail. |
| Writes | The alert store only (A14 section 6.6). It never updates the telemetry item. |
| State it may cause | `crop_alert_raised`, which opens or joins a pest case (A15 section 5.1), exactly as A14 and ADR-002 define. |
| Never does | Override a Federation range ("Las reglas configurables siguen vigentes"), authorize anything, or send a phone number or name to the model: the payload carries node, farm and readings only. |

Placing inference after publication keeps the telemetry function inside hop 5 of the alert budget (A10 section 8.2, 20 seconds as a design target, ASM-075): evaluation and publication do not wait for the model, so the model's response time is outside the 5-minute path.

## 5. Result record

Every result carries the same fields, so A21 and the defense can trace any outcome to its source.

| Field | Content |
|---|---|
| `result_id` | Idempotency key of section 6.3. |
| `function` | `confirmation`, `verification`, `estimation` or `threshold`. |
| `outcome` | Section 4 outcome, including `not_evaluable` with its reason. |
| `result_at` | UTC timestamp at which the result was persisted ("marca de tiempo"). |
| `confidence`, `confidence_basis` | Score in [0, 1] and `model` or `rule` ("confianza"). For a drone result, the overall model confidence; per-image scores are kept in `per_object`. |
| `source` | "Identificador del objeto de origen": for drone results, bucket, key, version identifier and SHA-256 of every object sent to the model, and the `mission_id` and flight record key and version; for telemetry, table, node identifier and original reading timestamp. |
| `excluded_objects` | Key, version and reason (`person_present`, `not_screened`, `model_person_detected`, `checksum_mismatch`) of every object of the mission not used. |
| `model_id`, `model_region`, `prompt_version` | The Bedrock model identifier, `us-east-1`, and the version of the prompt and schema. |
| `bedrock_request_id`, `input_tokens`, `output_tokens` | For audit and for the cost of section 9. |
| `event_id` | Identifier of the triggering event or SQS message, for tracing. |

Results live where the case statement says the record lives: pest case results on the case in the transactional store under the `records` key (A09 section 7.2), the harvest estimate in the same store, and telemetry results in the alert store of A14. Retention is 5 years from the result, as for the record it belongs to (ASM-098).

## 6. Failure handling

### 6.1 Retries with backoff and dead-letter queues

The function never retries by itself beyond the SDK call; the queue holds the event while the function is retried, so a failure always leaves the event, the object and the state where they were. All values below are design values, not measurements (ASM-162).

| Layer | Confirmation, verification, estimation | Telemetry threshold |
|---|---|---|
| Bedrock SDK call | AWS SDK adaptive retry mode, 3 attempts in total, with the SDK's exponential backoff and jitter, on throttling and server errors. | Same, for the plausibility call only. |
| Function timeout | 300 s; memory 512 MB. | 60 s; memory 256 MB. |
| Event source mapping | Batch size 1; reserved concurrency 5 per function, which bounds the parallel calls to Bedrock and the spend of a flood of missions. | Batch size 10, batching window 0 s, partial batch responses (`ReportBatchItemFailures`), reserved concurrency 10. |
| Queue visibility timeout | 1,800 s, six times the function timeout. | 360 s, six times the function timeout. |
| Backoff between attempts | On a retryable failure the function reports the message as failed and sets its visibility to 60 x 2^(n-1) s, where n is the receive count: 60, 120, 240, 480 s. | Same rule with a 10 s base: 10, 20, 40, 80 s, so that a transient failure still leaves the alert near the 5-minute target. |
| Attempts before dead-letter | `maxReceiveCount` 5. | `maxReceiveCount` 5. |
| Dead-letter queue | One SQS queue per function, encrypted with the `records` key, retention 14 days. | Same. The IoT rule's own error action queue (ADR-003) stays as the dead-letter queue of the store write. |
| Upstream delivery | The EventBridge rule targets have a retry policy and their own SQS dead-letter queue for events that could not reach the function queue; the pipe has a dead-letter queue for stream records it could not deliver. | The IoT rule error action receives a failed SQS or DynamoDB action. |
| Not retried | Guard failures, `not_evaluable` outcomes and duplicates are normal outcomes: the message is deleted. A model answer that does not match the schema is retried once inside the invocation, then counted as a failure. | Readings of a node that is not `active`, and duplicates. |

A message in a dead-letter queue is redriven to its source queue by a platform operator with SQS dead-letter queue redrive, after the cause is fixed; because every function is idempotent (section 6.3), a redrive of an event that later succeeded is harmless. Fourteen days is the longest an SQS queue keeps a message, so A21 alarms on the first message (section 7), not on age.

### 6.2 What a failure leaves behind

| Failure | Source object or record | Case, order, alert | Where it is seen |
|---|---|---|---|
| Bedrock throttles or errors on every attempt | Unchanged; functions cannot write it. | Unchanged: no result, no transition. The mission stays in `record_persisted` without a result. | Function dead-letter queue; `InferenceFailed` metric; missions awaiting a result (section 7). |
| The function times out or crashes after the model answered | Unchanged. | Unchanged: the transaction did not commit. The retry calls the model again (a cost, not a state change). | Lambda `Errors`; queue receive count. |
| The transaction condition fails because the state moved (for example the agronomist closed the case) | Unchanged. | Unchanged; the result is not written. | `guard_skipped` metric with reason `state_changed`. |
| The telemetry store write fails | The reading is not in the table; the rule error action queue holds it. | The evaluation message finds no item, retries with backoff and reaches the threshold dead-letter queue if the store is not repaired in time; no alert is raised from an unpersisted reading. | Rule error queue depth; threshold dead-letter queue; `ReadingNotPersisted` metric. |
| The plausibility call fails | Unchanged. | Alert already published; item marked `plausibility = unavailable`. | `PlausibilityUnavailable` metric. |

### 6.3 Idempotency

| Function | Idempotency key | Why a duplicate cannot act twice |
|---|---|---|
| Confirmation, verification | `{function}#{mission_id}`. | One result per mission: the result put is conditional on the key not existing, and the transition is conditional on the case and order still being in the source state. A duplicated `mission_record_persisted`, an SQS redelivery or a redrive finds the result and stops before calling the model. |
| Estimation | `estimation#{mission_id}`. | Same; a new estimation mission of the same farm writes a new result, and the latest one wins in the center's view. |
| Telemetry threshold | Evaluation item key of A14: node, variable and original reading timestamp. | A14 section 5 step 4 already makes the evaluation conditional; the episode update and publication follow only the first successful write. |

### 6.4 Ordering and failure coupling of the telemetry rule (closes PD-112)

The topic rule runs its two actions independently, and the threshold function no longer runs on the rule's copy of the reading but on the stored reading. The ordering is therefore fixed by the data, not by the rule: **a reading is evaluated only after it is persisted**, which is what "Lee el objeto ya persistido" requires. The coupling is one-directional. A failed evaluation never affects the stored reading, because the function cannot write the telemetry table. A failed store write blocks the evaluation of that reading until the reading is persisted, either by the rule's own retry or by the replay of the rule error queue into the table, which is idempotent by the table key (ADR-003); the evaluation message waits in the queue with backoff and, if the store stays down beyond its 5 attempts, ends in the threshold dead-letter queue to be redriven after the replay. A drained reading is still evaluated with the version in force at its original timestamp, so a late evaluation gives the same result as a live one (A14 section 5).

## 7. Error observation point (proposed to A21)

Point 6 asks for "invocaciones fallidas que no mutaron el estado". The observation point is the function's dead-letter queue together with the missions that reached `record_persisted` without a result; both are needed, because a message can be lost upstream of the queue. Names follow PC-IAC-003, `{client}-{project}-{environment}-{type}-{key}`, with `{client}-{project}` written `<prefix>` until PD-070 fixes it, and `pdn` shown. A21 sets the thresholds and windows (PD-162).

| Signal | Source | Proposed alarm name |
|---|---|---|
| Any message in a function dead-letter queue | `AWS/SQS` `ApproximateNumberOfMessagesVisible` > 0 on each dead-letter queue | `<prefix>-pdn-alarm-infconfirmdlq`, `<prefix>-pdn-alarm-infverifydlq`, `<prefix>-pdn-alarm-infestimatedlq`, `<prefix>-pdn-alarm-infthresholddlq` |
| Events that never reached a function queue | Dead-letter queues of the pipe and of the EventBridge rule targets; `AWS/Events` `FailedInvocations` | `<prefix>-pdn-alarm-infroutedlq` |
| Oldest pending event | `AWS/SQS` `ApproximateAgeOfOldestMessage` on each function queue | `<prefix>-pdn-alarm-infconfirmage` (and `infverifyage`, `infestimateage`, `infthresholdage`) |
| Failed invocations that did not change state | Custom metric `InferenceFailed` in namespace `AromaPacifico/Inference`, dimensions `function` and `error_class` | `<prefix>-pdn-alarm-inffailed` |
| Missions awaiting a result | Custom metric `MissionsAwaitingResult`: missions in `record_persisted` of type confirmation, verification or estimation with no result after the age A21 sets, counted by a scheduled query | `<prefix>-pdn-alarm-infawaiting` |
| Readings not persisted before evaluation | Custom metric `ReadingNotPersisted`; depth of the IoT rule error queue | `<prefix>-pdn-alarm-infthresholdstore` |
| Model throttling | Custom metric `InferenceThrottled`; `AWS/Bedrock` invocation throttles for the model | `<prefix>-pdn-alarm-infthrottle` |
| Objects excluded for a person | Custom metrics `PersonExcluded`, `NotScreenedExcluded`, `ModelPersonDetected` | `<prefix>-pdn-alarm-infperson` on `ModelPersonDetected` > 0, an incident (section 10.4) |
| Cost drivers | Custom metrics `InputTokens`, `OutputTokens`, `ImagesSent` per function | Cost dashboard of A21, no alarm |

Each function logs one structured line per event to its log group under the `logs` key (A09 section 7.2), with `event_id`, `mission_id` or reading key, outcome, `result_id`, model request identifier and error class, never an image, a name or a phone number.

## 8. Region and objects that leave Colombia (closes PD-013)

### 8.1 Region

Inference runs in **US East (N. Virginia), `us-east-1`**, the single Region of ADR-001, in the same Region as the bucket, the tables and the functions. Each function invokes the model by its **foundation-model identifier**, which runs the request in the Region called, and never by a cross-Region inference profile: a geographic or global inference profile may route a request to another Region than the one called, so the design could no longer say where an object is processed. Three controls keep it so: the function role allows `bedrock:InvokeModel` only on the resource `arn:aws:bedrock:us-east-1::foundation-model/<model_id>` and on no `inference-profile` resource; the root SCP denies any action whose requested Region is not `us-east-1` (A08 section 3.1); and the model identifier is validated by the pipeline as a foundation-model identifier.

Whether the chosen model is offered in `us-east-1` for in-Region on-demand invocation was recorded by ADR-001 from indexed copies of the AWS pages. This task tried to read the live pages on 2026-11-04, but the authoring session could not reach `docs.aws.amazon.com` or `aws.amazon.com` (the proxy refused the connection), so **PD-018 stays open**. If the re-check shows that the model is offered only through an inference profile, the model identifier is changed to a vision-capable model that is offered in-Region, and the design does not change; if none is, ASM-160 is reopened. The re-check is listed in PD-160.

### 8.2 What leaves the country

No AWS Region exists in Colombia (ADR-001), so **every object and record of the platform leaves Colombia** and is stored and processed in `us-east-1`, in the United States, under ASM-002; whether the Federation accepts that transfer is question Q-02 and PD-101. The table separates what is only stored from what is also sent to the model.

| Object or record | Stored in `us-east-1` | Sent to Bedrock in `us-east-1` |
|---|---|---|
| Confirmation photos with `person_screen = clear` | Yes | Yes |
| Verification photos and short-pass frames with `person_screen = clear` | Yes | Yes |
| Estimation photos and plot maps with `person_screen = clear` | Yes | Yes |
| Telemetry reading that opens an episode, with the node's 24-hour readings and the threshold version | Yes | Yes |
| Other telemetry readings | Yes | No |
| Flight records, tracks, polygons, mission packages | Yes | No (the verification track is read by the function, not sent) |
| Delimitation objects | Yes | No: no function runs on a delimitation (A15 section 8). |
| Objects flagged `person_present` or not screened | Yes, in the restricted prefix or excluded (section 10) | No |
| Personal data (names, cédula, phone, plate) | Yes, in the personal-data table (A09) | No: no function role holds the `personal` key, and no payload carries personal data. |

Amazon Bedrock is described by AWS as not using prompts and outputs to train models and not sharing them with model providers; that statement is part of the PD-160 re-check and is not relied on as verified here.

## 9. Cost per thousand invocations in harvest

### 9.1 Harvest volume of each trigger

The volumes follow the case figures through A14 section 3.1 and ADR-002; where the case gives no rate, the volume stays a named variable for A23 (PD-031, PD-133).

| Trigger | Harvest volume | Basis |
|---|---|---|
| Telemetry batch | 3,600 readings a day, 108,000 in a 30-day harvest month; invocations at most 108,000 (fewer when readings share a batch). Model calls: E, the episode openings of the month. | 200 farms x 6 + 100 farms x 24 readings a day (A14 section 3.1). The case gives no alert rate, so E is A23's (PD-133); E is at most the number of readings. |
| Confirmation object | N_conf, one per confirmation mission. | One confirmation per `crop_alert_raised` case (ADR-002); the case gives no rate of pest cases (PD-031). |
| Verification object | N_ver, at most one per fumigated case, plus re-verifications. | ADR-002 flight volumes. |
| Estimation flight object | 200 in the month of the first campaign of each harvest period. | One estimation flight per farm per harvest period (ADR-002, ASM-024). |
| Harvest month total | 108,000 + 200 + N_conf + N_ver invocations; E + 200 + N_conf + N_ver model calls. | Sum of the rows. |

### 9.2 Formula

No unit price could be read on 2026-11-04 (section 8.1), so **no number is written**: each unit price and each model-dependent quantity is a named assumption for A23 under PD-017 (ASM-169). For a function f, the cost of one thousand invocations is:

`C_1000(f) = 1000 x [ P_req + M_f x D_f x P_gbs + Q_f x P_sqs + R_f x (P_s3get + P_kms) + W_f x P_ddbw + K_f x P_ddbr + p_f x ( (I_f x T_img + T_prompt_f + T_ctx_f) x P_in + T_out_f x P_out ) / 1000 + X_f ]`

| Symbol | Meaning | Value or source |
|---|---|---|
| `P_req`, `P_gbs` | Lambda price per request and per GB-second in `us-east-1` | ASM-169, from the AWS Lambda pricing page, read by A23 with its date |
| `M_f`, `D_f` | Memory in GB and billed duration in seconds | `M_f` from section 6.1 (0.5 or 0.25); `D_f` unmeasured, bounded above by the timeout (300 s or 60 s) for a worst case |
| `Q_f`, `P_sqs` | SQS requests per invocation (receive, delete, visibility change) and price per request | `Q_f` about 3 per message; ASM-169 |
| `R_f`, `P_s3get`, `P_kms` | Objects read per invocation and the S3 GET and KMS request prices | `R_f` is `I_f` plus the flight record (and the track for verification); 0 for telemetry, whose reads are DynamoDB; ASM-169 |
| `W_f`, `K_f`, `P_ddbw`, `P_ddbr` | DynamoDB write and read request units per invocation and their on-demand prices | About 3 writes and 3 reads for a drone function, 2 writes and 1 to 25 reads per reading for telemetry; ASM-169 |
| `p_f` | Share of invocations that call the model | 1 for the drone functions; E / 108,000 for telemetry |
| `I_f` | Images per model call | At most 8 (confirmation, verification) and 21 (estimation: 20 photos and 1 map) (ASM-168); 0 for telemetry |
| `T_img`, `T_prompt_f`, `T_ctx_f`, `T_out_f` | Input tokens per image, prompt tokens, context tokens (24-hour readings for telemetry), output tokens | Model-dependent; ASM-169, from the model documentation, read by A23; measured later from `input_tokens` and `output_tokens` of section 5 |
| `P_in`, `P_out` | Bedrock on-demand price per 1,000 input and output tokens of the chosen model in `us-east-1` | ASM-169, from the Amazon Bedrock pricing page, read by A23 with its date |
| `X_f` | Event delivery per invocation: EventBridge Pipes request, custom bus event and rule delivery for drone functions; IoT rule action for telemetry | ASM-169 |

The harvest month cost of inference is `C_1000(threshold) x 108 + C_1000(confirmation) x N_conf / 1000 + C_1000(verification) x N_ver / 1000 + C_1000(estimation) x 0.2`. The telemetry term counts every reading but calls the model only on openings, which is the first of the cost decisions below; the reserved concurrency of section 6.1 caps the spend of a flood of missions.

| Decision that lowers or avoids cost | Effect |
|---|---|
| The model is called on episode openings, not on every reading. | The model term of telemetry scales with E, not with 108,000 readings. |
| Idempotency is checked before the model call. | Duplicates and redrives of a completed event cost a lookup, not a model call. |
| Image caps per call and exclusion of unscreened objects. | Bounds `I_f`; nothing is sent twice. |
| On-demand invocation, no provisioned throughput, no endpoint. | No cost outside harvest except the invocations that occur. |

## 10. Objects that contain a person (closes PD-015)

### 10.1 Rule

"Si el objeto contiene una persona, no se envía a inferencia y se documenta el tratamiento del archivo." The rule needs a way to know that an object contains a person before any inference service sees it. The design answers it with a screen that runs where the object is created, a fail-closed default in the platform, a restricted store for the file, and a backstop for a screen that misses.

### 10.2 Detection before inference

| Option | Sends the object to inference? | Verdict |
|---|---|---|
| A managed face or person detector in the cloud (Amazon Rekognition `DetectLabels` person label, `DetectFaces`) before the crop model | Yes: it is itself a managed inference service, and it would process the image of the person in `us-east-1`, which is exactly what the rule is meant to avoid. | Rejected. |
| Prompting the crop model to say whether a person is present | Yes, by definition. | Rejected as the control; kept only as a backstop (section 10.4). |
| A person screen on the ground station, before upload (chosen) | No: the object is screened on the operator's equipment, where it was captured, and no service receives it. | Chosen (ASM-164). |
| Prevention by flight geometry only (no-fly zones exclude houses, wet mill and roads) | No | Kept as a reducing factor, not as a control: workers can be in a plot. |

The ground-station software of the contracted operator runs an on-device person detector on every photo, frame and map before upload, the pilot can also mark an object, and the result is written per object in the flight record manifest as `person_screen` = `clear` or `person_present` (ASM-164). This extends the interface the operator contract already fixes (PD-143) and is required by PD-164. In the platform the default is **fail closed**: only an object whose manifest entry says `clear` is ever sent to the model; an entry that is missing, unknown or from an older schema is treated as `not_screened` and excluded.

### 10.3 Treatment of the file

| Aspect | Treatment |
|---|---|
| Where it goes | When it links the object, the reception function of A15 copies an object marked `person_present` to `restricted/person/{yyyy}/{mm}/{dd}/{original key}` and deletes the original versions, as it already does for `quarantine/` (A15 section 7). The mission still counts the object as delivered, so the flight record can reach `record_persisted`. A `not_screened` object stays in place and is only excluded. |
| Encryption | The restricted prefix is encrypted with a separate customer-managed key class, `restricted`, an eighth class next to the seven of ASM-097 and ASM-154. No inference role, and no human role, holds a grant on it, and the bucket policy explicitly denies every function role on `restricted/*` (ASM-165). |
| Access | No routine reader. A platform operator may read an object only for a recorded incident or a request of the person concerned, with the reason in the application audit record; the agronomist sees only that the object exists and why it was excluded. |
| Retention and deletion | 5 years from capture, by S3 lifecycle, because the case statement keeps photos 5 years; then deleted. An earlier deletion, for example on a request of the person or by a Federation rule, is a decision of the Federation with Q-02 (PD-166). |
| Effect on the case | If no object of a mission is usable, the function writes `outcome = not_evaluable` with reason `person_excluded` or `not_screened` and **no transition**: a confirmation case stays in `confirming`, a verification case stays in `verifying`, and an estimation gives no number. The agronomist plans a new mission of the same type, which needs no new case transition. If some objects are usable, the function uses only those and lists the others in `excluded_objects`. |

### 10.4 Backstop when the screen misses

Every model schema includes `person_detected`. If the model reports a person in an object that the ground station screened as `clear`, the rule has already been broken for that object; the function discards the whole result, writes `not_evaluable` with reason `model_person_detected`, asks the reception function to move the object to the restricted prefix, and emits `ModelPersonDetected`, which A21 treats as an incident. The incident is reported to the operator against the contract requirement of PD-164. It is a detection of a failure, not the control.

## 11. Epic 6 function pattern

Epic 6 codes one function pattern: "disparador, permiso de lectura del objeto, permiso de escritura del resultado y permiso de invocación al servicio de inferencia". The pattern module of ADR-003 is telemetry ingestion, so **the coded pattern is the telemetry threshold function** (PD-161). The other three functions stay in the architecture and are not coded in the 100 hours, as Epic 6 says.

| Element of Epic 6 | Telemetry threshold function (coded by B10) | Drone functions (architecture only) |
|---|---|---|
| Trigger | SQS action on the telemetry topic rule (B09), the threshold queue, its dead-letter queue and the Lambda event source mapping. | Mission table stream, pipe, custom bus, rule per function, queue and dead-letter queue. |
| Read permission | `dynamodb:GetItem` and `dynamodb:Query` on the telemetry table, `dynamodb:GetItem` on the `thresholds` table and the node registry, `kms:Decrypt` on `telemetry` and `records`. | `s3:GetObject` and `s3:GetObjectVersion` on the mission prefixes of its own type only (for example `farms/*/cases/*/confirmation/*` for the confirmation function, `farms/*/campaigns/*/estimation/*` for estimation), `kms:Decrypt` on `objects`; explicit Deny on `restricted/*`. |
| Result write permission | `dynamodb:PutItem`, `dynamodb:UpdateItem` and `dynamodb:ConditionCheckItem` on the alert store, `sns:Publish` on the alert topic, `kms:GenerateDataKey` on `records`. | Transactional write on the case, order and estimate items, conditional; no permission on authorize, sack, trip or code operations. |
| Inference permission | `bedrock:InvokeModel` on `arn:aws:bedrock:us-east-1::foundation-model/<model_id>` only. | Same. |
| Not granted | Any write on the telemetry table; `personal` key; any inference profile. | `s3:PutObject`, `s3:DeleteObject`; `personal` and `restricted` keys; any inference profile. |

## 12. Compliance with the issue

| Acceptance criterion | Where it is met |
|---|---|
| The 4 required triggers are described: outbreak confirmation object, verification object, estimation flight object and telemetry batch. | Section 3 (trigger and delivery path) and sections 4.2 to 4.5 (one design per function). |
| Each function reads an already persisted object, invokes managed inference and writes a result with timestamp, confidence and source object identifier. | Section 4.1 steps 4 to 6, section 4.5 (stored reading, inference on openings), section 5 (`result_at`, `confidence`, `source` with key and version). |
| The design specifies retries, a dead-letter queue and an error observation point. | Section 6.1 (retries, backoff, one dead-letter queue per function), sections 6.2 to 6.4, section 7 (metrics and alarm names for A21). |
| The inference service region and which objects leave the country are documented. | Section 8: `us-east-1`, foundation-model invocation without cross-Region profiles, table of what is stored and what is sent. PD-018 stays open for the live availability check. |
| The cost per thousand invocations during harvest is estimated, and the handling of an object that contains a person is defined. | Section 9 (harvest volumes, formula, named unit prices for A23 under PD-017; no price could be read) and section 10. |

## 13. Decision-log entries

| ID | Summary |
|---|---|
| PD-013 | Region and objects leaving the country decided. Closed by section 8. |
| PD-014 | Retries, dead-letter queues and observation point defined. Closed by sections 6 and 7. |
| PD-015 | Treatment of an object with a person defined. Closed by section 10. |
| PD-111 | Trigger on a persisted object, retry, dead-letter and observation. Closed by sections 3, 6 and 7. |
| PD-112 | Ordering and coupling of the rule's store write and the threshold function. Closed by section 6.4. |
| PD-135 | Threshold function implements the A14 contract with its retries and failure queue. Closed by sections 4.5 and 6. |
| PD-018 | Stays open: the live AWS pages could not be read on 2026-11-04. |
| ASM-160 | Amazon Bedrock, one vision-capable foundation model in-Region in `us-east-1`, zero-shot versioned prompts, no training or provisioned capacity. |
| ASM-161 | Delivery paths: mission stream, pipe, bus, rule and queue per drone function; IoT rule SQS action and consistent read for telemetry. |
| ASM-162 | Retry, backoff, timeout, concurrency and dead-letter design values. |
| ASM-163 | Model confidence is an uncalibrated self-report; decision thresholds 0.70, Federation-configured. |
| ASM-164 | On-device person screen at the ground station and the `person_screen` manifest field; fail closed. |
| ASM-165 | Restricted prefix and `restricted` key class for objects with a person; access, retention. |
| ASM-166 | Estimation formula with Federation yield factors per load class. |
| ASM-167 | Telemetry inference on episode openings only, after publication, never changing the alert. |
| ASM-168 | Image caps per call; short pass delivered as frames in `photos/`. |
| ASM-169 | Unit prices and model token quantities as named parameters for A23. |
| PD-160 to PD-167 | Live re-check of the AWS features named here, B09 and B10 coding, A21 alarms, A23 pricing, operator contract, role matrix and key class, Federation parameters, confidence calibration test. |

## 14. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Part E, Part A, Part B, drone section, point 6, Epic 6, "Cifras del piloto". |
| [`docs/decision-log.md`](decision-log.md) | ASM-001, ASM-002, ASM-024, ASM-075, ASM-097, ASM-098, ASM-141, ASM-149, ASM-154, PD-017, PD-018, PD-031, PD-070, PD-101, PD-133, PD-143 and the entries of section 13. |
| [ADR-001](adr/adr-001-primary-and-second-cloud.md) | Region, Region availability check of 2026-11-04. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | Events, services and flight volumes. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | Pattern module, routing rule and its error action, telemetry store key. |
| [`docs/drone-service.md`](drone-service.md) | Sections 4, 5, 7 and 8: records, state machines, prefixes, triggers. |
| [`docs/farm-monitoring.md`](farm-monitoring.md) | Sections 3, 5, 6 and 10: load, evaluation contract, stores, observability. |
| [`docs/security-and-identity.md`](security-and-identity.md) | Sections 4.2, 4.3, 7.2 and 9: role matrix, separation of duties, keys, personal data. |
| [`docs/landing-zone.md`](landing-zone.md) | Section 3.1 (Region SCP), section 5 (cost lines for `drones` and `farms`). |
| [`docs/connectivity.md`](connectivity.md) | Section 8.2: alert latency budget. |
| [`standards/iac-rules/pc-iac-003.md`](../standards/iac-rules/pc-iac-003.md) | Physical naming of the proposed alarms. |
| [Amazon Bedrock supported Regions and models](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html), [Cross-Region inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html), [Amazon Bedrock pricing](https://aws.amazon.com/bedrock/pricing/), [AWS Lambda pricing](https://aws.amazon.com/lambda/pricing/), [Amazon SQS pricing](https://aws.amazon.com/sqs/pricing/), [Amazon EventBridge pricing](https://aws.amazon.com/eventbridge/pricing/) | Pages to read for PD-018, PD-160 and ASM-169. Not readable from the authoring session on 2026-11-04; no content of them is relied on here. |
