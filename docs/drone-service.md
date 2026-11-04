# Drone Service and Fumigation Order

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR2 / JR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#15 (A15) |
| Builds on | [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md), [`docs/connectivity.md`](connectivity.md) (A10, section 6), [`docs/collection-center-network.md`](collection-center-network.md) (A11, section 4.1), [`docs/firewall-policy.md`](firewall-policy.md) (A12, DRN rules), [`docs/security-and-identity.md`](security-and-identity.md) (A09, sections 4, 6, 7 and 10), [`docs/landing-zone.md`](landing-zone.md) (A08, sections 5.3 and 7), `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13, drone objects) |
| Feeds | A17 inference functions (aroma-pacifico/aroma-pacifico-roadmap#17), A19 architecture document and complete diagram (aroma-pacifico/aroma-pacifico-roadmap#19), A21 observability (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36), B04 network module (aroma-pacifico/aroma-pacifico-roadmap#24) |
| Decision-log entries | PD-012, PD-032, PD-104 and PD-121 (closed here), ASM-140 to ASM-149, PD-140 to PD-143 |

## 1. Purpose and scope

The drone section of the [case statement](case/case-statement.pdf) asks for a service, not a loose flight: "La plataforma crea la orden, la autoriza, la entrega al equipo de vuelo y guarda el resultado." Epic 4 covers "Los cinco servicios, la orden de fumigación y el enlace", and Part B adds the campaign cadence. [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) already decided the combined flight mode, the five services and their gates, the switching events, the fields and states of the fumigation order, who flies and who authorizes, and the cadence. This document does not reopen any of that. It designs how the platform implements it: the records it keeps, the transitions it allows and who may cause each one, what the ground station downloads and uploads, how objects are named, which event starts each inference function, and which endpoints the ground station uses.

Operating the drone, the flight plan of each farm, dose, mixture and product are out of scope, as the case statement says ("No se pide operar el dron ni definir dosis, mezcla o producto"; "Queda fuera [...] el plan de vuelo de cada finca"). The platform hands the operator the constraints of a flight (polygon, no-fly zones, focus, window) and keeps the evidence it returns. Quotations are in the original Spanish, which prevails over the English rendering (ASM-001). No figure in this document is a measurement.

## 2. Inputs this document builds on

| Source | What it fixes | Used in |
|---|---|---|
| ADR-002 | Combined mode; events `crop_alert_raised`, `focus_confirmed`, `focus_not_confirmed`, `case_closed`; the five services with trigger, inputs, record and gate; order fields and states; mission preload; contracted pilot flies, agronomist authorizes; cadence and campaign content (ASM-020 to ASM-029). | Sections 3 to 6 and 10 |
| A10 section 6 | HTTPS from the ground station, mission download from the application API, multipart upload to Amazon S3 with pre-signed URLs and checksums, deletion from the buffer only after confirmed persistence, paths from a center and from a municipal seat. | Sections 7 and 9 |
| A11 section 4.1 | Ground station in Soporte, VLAN 102, at 10.4x.2.10. | Section 9 |
| A12 DRN and APP rules | CC-DRN-01 to CC-DRN-06, CL-APP-01, CL-APP-02, CL-DRN-01 to CL-DRN-04; PD-121 asks this task to name the `drone_object_ingress` endpoints. | Section 9 |
| A09 sections 4 and 6 | Drone operator and agronomist roles, separation of duties, 10-minute step-up for authorization, drone kit identity, upload-grant record, quarantine of objects from a revoked pilot or kit (PD-104). | Sections 4, 5 and 7 |
| A09 section 10 | Revocation of a lost drone and of a departing pilot, and its effect on pending orders. | Section 5.2 |
| A08 sections 5.3 and 7.2 | One S3 prefix per farm for cost allocation; audit of flight object delivery by CloudTrail S3 data events plus the application record. | Section 7 |
| A13 drone objects | Persistence is the prerequisite for inference and for closure; the reception function links each object to its flight record; the inference trigger mechanism is left to A17 (PD-111). | Sections 7 and 8 |

## 3. The five services as the platform runs them

Every flight is a **mission**: one assignment of one pilot and one drone kit to fly one service over one farm. A mission has a type, a mission package that the ground station downloads before take-off, and exactly one flight record that the ground station returns. Fumigation missions also belong to a fumigation order, and confirmation, fumigation and verification missions belong to a pest case. The table extends the services table of ADR-002 with the platform records and checks; the gates themselves are ADR-002's.

| Service | Mission type | Belongs to | Planned by | Mission package adds | Objects returned (section 7) | When the mission counts as persisted | What persists next |
|---|---|---|---|---|---|---|---|
| Delimit the plot | `delimitation` | Farm and review campaign | Campaign planner, or the agronomist after a reported plot change or for a case on a farm with no polygon (ASM-027) | Declared plots and preliminary no-fly zones | `polygons` (plot polygons, confirmed no-fly zones, GeoJSON), `maps`, `track`, flight record | Flight record and every object of its manifest persisted with matching checksum | The reception function validates the GeoJSON and activates a new polygon version of the farm (ASM-147). |
| Confirm the sensor alert | `confirmation` | Pest case | Platform, on `crop_alert_raised` | Focus coordinates (crop node location, ASM-056), case identifier | `photos` of the focus, `track`, flight record | Same | Confirmation function (section 8) persists `focus_confirmed` or `focus_not_confirmed`. |
| Fumigation | `fumigation` | Fumigation order and pest case | Platform, when the agronomist authorizes the order | Order fields, window, planned product batch identifier | `track`, flight record with the product batch identifier used and the closure | Same | Order moves to `executed`; a verification mission is planned. |
| Verify after fumigating | `verification` | Pest case and the order it verifies | Platform, when the order reaches `executed` | Focus coordinates, the order it verifies | `photos` or a short pass in `track`, flight record | Same | Verification function persists whether the case can close; the order moves to `closed`. |
| Estimate the harvest | `estimation` | Farm and the first review campaign of a harvest period (ASM-024) | Campaign planner | Farm polygon version | `photos`, `maps`, `track`, flight record | Same | Estimation function persists the estimated sacks for the farm's collection center; no packing order. |

No mission of one type can stand in for another: the guard of each transition in section 5 names the mission type it needs, so for example a delimitation map can never satisfy a verification, and an estimation flight never produces an order. This is how the platform keeps "Los cinco entran. No se sustituye uno por otro."

## 4. Records

All the records below live in the transactional store under the `records` key (A09 section 7.2), as Part E requires for fumigation orders ("Eventos de saco y órdenes de fumigación: base transaccional"). Objects live in Amazon S3 (section 7). Persons are referenced by the pseudonymous `subject_id` of A09 section 9.1, never by name or cédula. Every state change appends a row to the record's history with the previous state, the new state, the actor (a `subject_id`, or the name of a function role), the time and the reason, so the whole path of a case can be replayed.

### 4.1 Pest case

| Field | Content |
|---|---|
| `case_id` | Opaque identifier generated by the platform. |
| `farm_id`, `plot_id` | Plot that contains the node whose reading raised the alert. |
| `alert_refs` | Every `crop_alert_raised` record attached to this case: node, reading timestamp, variable, value and range. |
| `focus` | Coordinates of the crop node (ASM-056). |
| `state` | Section 5.1. |
| `confirmation_result` | Result written by the confirmation function: outcome, confidence, timestamp, source object key and version. |
| `order_ids` | Fumigation orders of the case, in order of creation. |
| `verification_results` | Results written by the verification function, one per verification mission. |
| `closed_at`, `closed_by` | Time and actor of the terminal state. |

### 4.2 Fumigation order

The order carries the seven fields of ADR-002 and the attributes the platform needs to enforce them.

| Field | Content | Set by |
|---|---|---|
| `order_id` | Opaque identifier. | Platform |
| `farm_id`, `plot_id` | As ADR-002. | Platform, from the case |
| `reason` | `case_id` and the `focus_confirmed` result: source object key and version, confidence, timestamp, and the `mission_id` of the closed confirmation flight record. | Platform, from the case |
| `polygon_version` | Version of the plot polygon and no-fly zones in force when the order was authorized; the mission package carries exactly this version (ASM-147). | Platform, at authorization |
| `window` | `window_start`, `window_end` in UTC (ASM-029). | Agronomist |
| `operator_id` | Pilot `subject_id`, permit reference and permit expiry (ASM-020). | Agronomist chooses the pilot; platform copies the permit data |
| `authorizer_id` | Agronomist `subject_id` and the `auth_time` of the MFA session used. | Platform, from the authorizing token; never writable by a function role (A09 section 4.3) |
| `product_batch_id` | Planned product batch identifier. | Agronomist |
| `kit_thing_name` | Drone kit that flies the order (A09 section 6). | Pilot, at dispatch |
| `mission_id` | Fumigation mission of the order. | Platform |
| `state`, `state_history` | Section 5.2. | Platform |
| `end_reason` | For `expired`: `window_elapsed`, `rejected`, `pilot_revoked` or `kit_revoked` (ASM-142). | Platform |
| `flags` | `out_of_window`, `batch_mismatch`, `aborted_closure`, `late_record` (section 6). | Reception function |

### 4.3 Mission and flight record

The mission record holds `mission_id`, `mission_type`, `farm_id`, `plot_id` where applicable, `case_id` or `campaign_id`, `order_id` for fumigation and verification, the assigned pilot `subject_id`, the `kit_thing_name`, the mission package digest, its state (section 5.3) and the list of objects persisted so far. The flight record that the ground station returns is defined in section 6.

## 5. State machines

The case statement and ADR-002 fix the events and the order states; the tables below add the guard of each transition and the actor allowed to cause it. A transition whose guard fails is refused and leaves the state unchanged; the refusal is logged with its reason. No transition toward `authorized` has a function or a trigger as its actor ("La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta."), and no transition toward `executed` or `closed` happens without a persisted record ("El estado de la fumigación no pasa a cerrado hasta persistir ese registro").

### 5.1 Pest case

| From | To | Trigger | Guard | Actor |
|---|---|---|---|---|
| (none) | `opened` | `crop_alert_raised` persisted by the threshold function | No case in a non-terminal state exists for the same plot; otherwise the alert is attached to that case's `alert_refs` and no case is opened (ASM-141). | Platform (case service) |
| `opened` | `awaiting_delimitation` | Case opened | The farm has no active polygon version (ASM-027). A `delimitation` mission is planned. | Platform |
| `awaiting_delimitation` | `confirming` | New polygon version activated | The plot of the case exists in the new version. | Platform |
| `opened` | `confirming` | Case opened | The plot has an active polygon. A `confirmation` mission is planned with the focus of the case. | Platform |
| `confirming` | `confirmed` | `focus_confirmed` persisted | Result references a persisted confirmation object of a `confirmation` mission of this case whose flight record is persisted. | Confirmation function (writes result); platform applies the transition |
| `confirming` | `not_confirmed` | `focus_not_confirmed` persisted | Same guard. | Confirmation function; platform |
| `not_confirmed` | `closed_without_fumigation` | Agronomist closes the case | Agronomist of the case's department with MFA; reason recorded. Plot returns to area mode. | Agronomist |
| `not_confirmed` | `confirming` | Agronomist requests a new confirmation | Same identity check; a new `confirmation` mission is planned (ASM-141). | Agronomist |
| `confirmed` | `in_fumigation` | Draft order created | No other order of the case is in `draft` to `executed`. | Platform |
| `in_fumigation` | `verifying` | Order reaches `executed` | Fumigation flight record persisted. A `verification` mission is planned for the same plot and focus. | Platform |
| `in_fumigation` | `confirmed` | Order reaches `expired` | A new draft order can be created for the case; the expired order keeps its history. | Platform |
| `verifying` | `closed` | `case_closed` persisted | Verification result says the case can close; it references a persisted verification object of a `verification` mission of this case whose flight record is persisted; the order is `closed`. Plot returns to area mode. | Verification function (writes result); platform |
| `verifying` | `verification_not_closable` | Verification result says the case cannot close | Same object and record guard. | Verification function; platform |
| `verification_not_closable` | `confirmed` | Agronomist asks for a new fumigation | Agronomist with MFA; a new draft order is created (ASM-141). | Agronomist |
| `verification_not_closable` | `verifying` | Agronomist asks for a new verification | Agronomist with MFA; a new `verification` mission is planned for the same order. | Agronomist |

If the verification object or its flight path never arrives, the verification mission stays in `flown` (section 5.3), no verification result exists, and the case stays in `verifying`: it is open, and A21 reports it ("Si ese archivo no llega, la fumigación sigue abierta"). If the verification function fails, the result is not written and the state does not change (Part E); retries and the failure queue are A17's (PD-014).

### 5.2 Fumigation order

The order states are those of ADR-002: `draft`, `authorized`, `dispatched`, `flown`, `executed`, `closed`, `expired`. A fumigation counts as open in every state before `closed`.

| From | To | Trigger | Guard | Actor |
|---|---|---|---|---|
| (none) | `draft` | Case reaches `confirmed` | Plot has an active polygon version; confirmation flight record persisted ("Si no hay polígono del lote o no hay cierre del registro, la orden no sale"). Farm, plot and reason are filled from the case (ASM-142). | Platform (order service) |
| `draft` | `authorized` | Agronomist authorizes | Caller is in group `agronomist`, scoped to the farm's department, `auth_time` at most 10 minutes old (ASM-092); `window_end` is in the future and after `window_start`; `operator_id` is in group `drone_operator`, differs from the authorizer, and has a permit reference whose expiry is after `window_end` (ASM-020); `product_batch_id` is filled; the guards of `draft` still hold. `authorizer_id` is taken from the token, not from the request body. | Agronomist |
| `draft` | `expired` (`rejected`) | Agronomist rejects | Same identity checks; reason recorded (ASM-142). | Agronomist |
| `authorized` | `dispatched` | Assigned pilot downloads the mission package | Caller is the `operator_id` pilot with MFA and is not revoked; permit still valid; the chosen drone kit is `ACTIVE` in the IoT registry and not revoked; current time is before `window_end`. The platform records the kit, the package digest and the download time. | Drone operator |
| `dispatched` | `flown` | Kit reports `flight_completed` on its status topic, or the first object of the mission arrives (ASM-143) | Message or object authenticated as the order's kit; the status message carries the package digest. | Drone kit (status message) or reception function |
| `dispatched`, `flown` | `executed` | Fumigation flight record of this mission persisted with every object of its manifest | Record matches the order, the package digest and the pilot and kit of the upload grant; both were active when the object arrived (A09 section 6). The order is flagged, not blocked, if the flown times fall outside the window or the batch used differs from the planned one (ASM-142, ASM-146). | Reception function |
| `executed` | `closed` | Verification result persisted | A `verification` mission of this order has its flight record and objects persisted and the verification function has written its result, whatever the outcome (ASM-141). | Verification function; platform |
| `draft`, `authorized`, `dispatched` | `expired` (`window_elapsed`) | `window_end` passes with no fumigation flight record | No fumigation flight record of the order is persisted, and the order is not in `flown`. A new authorization is needed (ASM-029). | Platform scheduler |
| `authorized`, `dispatched` | `expired` (`pilot_revoked`) | Pilot revoked (A09 section 10) | Pilot not yet flown for this order. A new draft names another pilot. | Platform, on the revocation |
| `dispatched` | `expired` (`kit_revoked`) | Kit revoked and `window_end` passes (A09 section 10) | Same as `window_elapsed`; the kit cannot obtain upload grants. | Platform scheduler |
| `expired` (`window_elapsed`) | `executed` | A fumigation flight record of the order arrives late | The record's take-off and landing lie inside the authorized window and its pilot and kit were active at take-off; the order is flagged `late_record` (ASM-142). | Reception function |

An order in `flown` does not expire at `window_end`: the flight happened inside the window and its record is in the buffer, which is the no-signal case the case statement describes. Observability shows how long it has been in `flown` (PD-141).

### 5.3 Mission

| From | To | Trigger | Guard | Actor |
|---|---|---|---|---|
| (none) | `planned` | Section 3, column "Planned by" | Farm enrolled; for case missions, the case is in the state that plans it. | Platform or agronomist |
| `planned` | `assigned` | Pilot assigned | Pilot in group `drone_operator` with a valid permit (ASM-020, ASM-148). For fumigation, the pilot is the order's `operator_id`. | Agronomist (fumigation: at authorization) |
| `assigned` | `dispatched` | Mission package downloaded | Same checks as order `dispatched`; for fumigation the order must be `authorized`. | Drone operator |
| `dispatched` | `flown` | `flight_completed` status or first object (ASM-143) | As the order. | Drone kit or reception function |
| `dispatched`, `flown` | `record_persisted` | Flight record and every object in its manifest persisted with matching checksum | As the order; objects that fail the grant check go to quarantine and do not count. | Reception function |
| `planned`, `assigned`, `dispatched` | `cancelled` | Case closed, order expired, campaign ended without the flight, or pilot or kit revoked | Recorded with reason; a dispatched mission whose record later arrives is still persisted and linked, as evidence. | Platform or agronomist |

A mission in `dispatched` or `flown` without a persisted record is a "vuelo sin registro persistido" of point 6.

## 6. Mission package and flight record

### 6.1 Mission package

The mission package is what "La misión se carga antes del despegue" loads. The pilot downloads it from the application API (CC-DRN-01) at the municipal seat or the collection center, where there is a link, and the ground station keeps it for the flight. It is a JSON document; geometry is GeoJSON (RFC 7946) in WGS 84, which drone ground-station software commonly imports (ASM-144).

| Element | Content |
|---|---|
| Header | `package_id`, `package_version`, `issued_at`, `valid_until` (the order's `window_end` for a fumigation; the end of the campaign, or the date the agronomist sets for an extra mission, otherwise), `package_digest` (SHA-256 of the canonical document, computed by the platform). |
| Mission | `mission_id`, `mission_type`, `farm_id`, `plot_id` where applicable, `case_id`, `order_id`, `campaign_id`. |
| People and device | Pilot `subject_id`, permit reference and expiry, `kit_thing_name`. No name, cédula or phone of anyone, including the farmer. |
| Geometry | Plot polygon and farm boundary of `polygon_version`, no-fly zones (houses, wet mill, roads, water sources) of the same version, focus point for confirmation and verification. |
| Fumigation only | `window_start`, `window_end`, planned `product_batch_id`, `authorizer_id` reference (no personal data). |
| Return instructions | Object key prefix of the mission (section 7), expected object classes, the flight record schema version, and the MQTT grant topic of the kit (A09 section 6). |

The platform stores the digest in the mission record and the flight record must echo it, so the platform can prove which polygon and no-fly zones the drone carried. The flight plan inside those limits is the operator's. A package cannot be downloaded after `valid_until`, and a new package version is issued only before dispatch; a polygon change after dispatch takes effect on the next mission.

### 6.2 Flight record

The flight record is a JSON document that the ground station writes after landing and uploads as `record/flight-record.json` under the mission prefix. It carries what the case statement requires the platform to keep for a fumigation ("quién autorizó, quién voló, la ventana, el identificador del lote de producto usado y el cierre del vuelo") and what the other services need (ASM-146).

| Field | Content | Required for |
|---|---|---|
| `schema_version`, `record_id` | Schema of the package; identifier generated by the ground station. | All |
| `mission_id`, `mission_type`, `order_id`, `case_id`, `farm_id`, `plot_id` | Echo of the package. | All (order and case where they apply) |
| `package_digest` | Digest of the package flown. | All |
| `pilot_subject_id`, `kit_thing_name` | Who flew and with which kit; checked against the upload grant. | All |
| `takeoff_at`, `landing_at` | UTC times recorded by the ground station. | All |
| `link_state` | Whether the ground station had a data link during the flight (`online`, `offline`), and `recorded_at`, the time the record was written. | All |
| `product_batch_id_used` | Batch identifier actually loaded. | Fumigation |
| `closure` | `completed` or `aborted`, and a free-text reason for an abort. This is the "cierre del vuelo". | All |
| `track_key` | Key of the flown path object in `track/`. | All |
| `manifest` | Every object of the flight: key, size, SHA-256 and capture time. | All |

The authorizer and the window are already on the order, so the record does not repeat them; the platform joins them. Any persisted fumigation flight record moves the order to `executed`, also when `closure` is `aborted`, because the platform cannot judge what was applied: the order is flagged `aborted_closure` and the verification decides whether the case can close (ASM-146). Dose and mixture are not recorded.

## 7. Objects and S3 prefixes

Each environment has one drone object bucket in its workload account, encrypted with the `objects` key, versioned and with Block Public Access (A09 section 7.2). Keys are built only from opaque identifiers, never from a name, a cédula or a phone, because keys appear in CloudTrail and in cost reports (ASM-145). The first segment is the farm, as A08 section 5.3 needs to allocate bytes and objects per farm.

| Prefix | Holds | Written by |
|---|---|---|
| `farms/{farm_id}/cases/{case_id}/{mission_type}/{mission_id}/` | Confirmation, fumigation and verification missions of a pest case. | Ground station, through upload grants |
| `farms/{farm_id}/campaigns/{campaign_id}/{mission_type}/{mission_id}/` | Delimitation and estimation missions of a review campaign. | Same |
| `farms/{farm_id}/extra/{mission_id}/delimitation/` | A delimitation flown outside a campaign (plot change, case on a farm without polygon). | Same |
| `.../{mission_id}/package/mission-package-v{n}.json` | The mission package issued. | Application API role |
| `.../{mission_id}/record/flight-record.json` | The flight record. | Ground station |
| `.../{mission_id}/photos/`, `track/`, `maps/`, `polygons/` | Object classes of section 3. Names are `{sequence}.{extension}` with a zero-padded sequence. | Ground station |
| `quarantine/{yyyy}/{mm}/{dd}/{original key}` | Objects whose key matches no unexpired grant, or whose pilot or kit was revoked when they arrived (A09 section 6, PD-104). Never linked to an order and never read by inference. | Reception function (copy, then delete of the original version) |

The upload grant of A09 section 6 lists the exact keys of the mission's manifest, so a ground station cannot write outside its mission prefix. The ground station uploads objects first and the flight record last; the reception function evaluates the manifest whenever either arrives, so the order of arrival does not matter for correctness. When a mission reaches `record_persisted`, the API returns a receipt with each key, version identifier and checksum, and the ground station deletes from its buffer only the objects listed in that receipt (A10 section 6). Retention is 5 years by S3 lifecycle, including the quarantine prefix, which is kept as evidence (A09 section 9.2). The treatment of an object that contains a person is A17's (PD-015).

## 8. Triggers of the inference functions

Part E makes inference event-driven: "La función se invoca con un disparador y termina al persistir el resultado." The four functions are designed by A17 (aroma-pacifico/aroma-pacifico-roadmap#17), which also chooses the invocation mechanism, retries, failure queue and observation point (PD-014, PD-111). This section only names the event that starts each function and its input. A drone object is not sent to inference when it arrives: the trigger is the reception function's record that the object is linked to a mission whose flight record is persisted (ASM-149), so inference never runs on an object of an unknown, revoked or incomplete mission.

| Function | Triggering event | Input object | Guard before invoking | Output (same record, with timestamp, confidence and source object key and version) |
|---|---|---|---|---|
| Confirmation | `mission_record_persisted` for a `confirmation` mission | The `photos/` objects of that mission | Case in `confirming`. | `focus_confirmed` or `focus_not_confirmed` on the case. Never authorizes. |
| Verification | `mission_record_persisted` for a `verification` mission | The `photos/` objects, or the short pass in `track/`, of that mission | Order of the mission in `executed`. | Can close or cannot close, on the case; drives `case_closed` and order `closed`. |
| Estimation | `mission_record_persisted` for an `estimation` mission | The `photos/` and `maps/` objects of that mission | Farm has an active polygon version. | Estimated sacks for the farm's collection center. No packing order. |
| Telemetry threshold | IoT topic rule action on each accepted reading (ADR-003, A13) | The reading, not an object | Range configured by the Federation. | `crop_alert_raised`, which opens or joins a case (section 5.1). |

A `delimitation` mission starts no inference function: its polygons are validated and activated by the reception function (ASM-147). If a function fails, the source objects stay where they are and no state changes (Part E).

## 9. Ingress endpoints of the ground station (PD-121)

The ground station reaches the platform over HTTPS with TLS 1.2 or later (A10 section 6), and its IoT connection uses mutual X.509 (A09 section 6). From a collection center it sits in Soporte at 10.4x.2.10 and uses the tunnel; from a municipal seat it uses public endpoints in `us-east-1`. The complete set of endpoints it calls is:

| Purpose | Endpoint at a collection center | Endpoint at a municipal seat | A12 rules |
|---|---|---|---|
| Pilot sign-in with MFA | Amazon Cognito interface endpoint (`cognito-idp`) in `application` | Public Cognito endpoint | CC-DRN-01, CL-APP-01 |
| Mission package download, upload-grant request, persistence receipt | Application API through the `execute-api` interface endpoint in `application` | Public application API front door | CC-DRN-01, CL-APP-01, CL-APP-02 |
| Upload grant delivery | AWS IoT Core data endpoint, public, through the Soporte local breakout (ASM-126) | Same public endpoint | CC-DRN-04, CL-TEL-01 |
| Object and flight record upload | Amazon S3 interface endpoint in `object_storage`, private DNS answered by the Resolver inbound endpoint (ASM-122) | Public S3 regional endpoint | CC-DRN-03, CL-DRN-01 to CL-DRN-03 |

No endpoint of this set needs to live in the `drone_object_ingress` tier, so **in the pilot that tier holds no interface endpoint** (ASM-140). A second S3 or `execute-api` endpoint there would duplicate one of the endpoints above, at an hourly charge per Availability Zone, without adding a control that the endpoint policy CL-DRN-02, the bucket policy CL-DRN-03 and the upload grant do not already give. The reception function needs no VPC attachment, because it reads S3 and the transactional store through their AWS endpoints and has no network peer in the VPC. The tier stays reserved, as `telemetry_ingress` does under CL-TEL-04, for a later separation of drone uploads from `object_storage`. Accordingly CC-DRN-02 becomes an explicit Deny and CL-DRN-01 is narrowed to the S3 interface endpoint in `object_storage`; the route and network ACL rows that still name the tier (CL-LNK-01, CL-LNK-07, CL-DRN-04) are aligned by the network module (PD-140). The statement in A10 section 6 and A13 that the center path ends at "interface endpoints in `drone_object_ingress` and `object_storage`" therefore resolves to the S3 interface endpoint in `object_storage`.

## 10. Without a link: mission preloaded, record published on restore

| Phase | Link needed | What happens |
|---|---|---|
| Before travelling | Yes, at the municipal seat or the collection center | The pilot signs in, chooses the kit and downloads the package. Without this download the order or mission cannot reach `dispatched`, so no drone leaves without its polygon and no-fly zones. |
| Over the plot | No | The mission is already loaded ("Si el enlace cae en el lote, la misión ya va cargada"). Objects, the track and the flight record are written to the encrypted buffer of the drone and ground station (ASM-095). The platform still shows `dispatched`. |
| After landing, still without link | No | The record and objects stay in the buffer with no time limit (A10 section 6). The order does not reach `executed` and does not expire while it is `flown`; if the kit could not report `flight_completed`, an order still in `dispatched` expires at `window_end` and returns to `executed` when the record arrives (section 5.2). |
| Link restored | Yes | As soon as there is a link, and at the latest at the preload point (ASM-028), the kit connects to IoT Core, reports `flight_completed`, receives its upload grant, uploads the objects and then the flight record by multipart with checksums, and resumes interrupted parts. The reception function persists, verifies and links; the mission reaches `record_persisted`; the order reaches `executed` ("El registro de vuelo se publica al restablecer el enlace"). |
| After persistence | Yes | The ground station deletes only what the persistence receipt lists. Each delivered object is audited by CloudTrail S3 data events and the upload-grant record (A08 section 7.2, A09 section 6). |

## 11. Review campaign cadence (PD-012)

The cadence of the case statement is kept, as ADR-002 decided: during harvest, one review campaign per month plus the fumigations that open cases produce; outside harvest, one review campaign every three months ("En cosecha hay una campaña de revisión al mes [...] Fuera de cosecha, una campaña de revisión cada tres meses"). It is kept because it fits the services: the monthly harvest campaign carries the once-per-harvest estimation, and outside harvest pest work is driven by alerts, not the calendar. A changed cadence would need a figure the case statement does not give, such as a rate of pest cases (PD-031).

The platform implements it with a **review campaign** record per department and period: `campaign_id`, department, start and end dates, harvest flag and the missions it contains. The Federation configures the harvest periods of each department (ASM-024); the campaign planner then creates one campaign per calendar month inside a harvest period and one per three months outside it, with the start dates the Federation sets (ASM-148). When a campaign opens, the planner adds, in the order of ASM-025, the pending delimitations, the confirmations of open cases not yet flown, the verifications of orders already `executed`, and, in the first campaign of a harvest period, one estimation per farm. Fumigations, and confirmations the agronomist does not want to defer, are extra missions outside the campaign (ASM-026). A campaign mission not flown by the campaign end is cancelled and planned again in the next campaign, and an open case is never cancelled with it.

## 12. Observability hooks

Point 6 asks for "Vuelos sin registro persistido, fumigaciones sin cierre e invocaciones fallidas que no mutaron el estado" and for the arrival of flight objects. This design gives A21 the facts; A21 sets the alarms (PD-141).

| Signal | Source in this design |
|---|---|
| Flights without a persisted record | Missions in `dispatched` or `flown`, with the age since `dispatched_at`; orders in `flown`. |
| Fumigations without closure | Orders in any state before `closed`, by state and age; cases in `verifying` or `verification_not_closable`. |
| Failed invocations that did not change state | A17's failure queue (PD-014) matched to missions in `record_persisted` without a result. |
| Arrival of flight objects | CloudTrail S3 data events and the reception function's persisted, rejected and quarantined counts per farm and campaign. |
| Order exceptions | Orders flagged `out_of_window`, `batch_mismatch`, `aborted_closure` or `late_record`. |

## 13. Compliance with the case statement and the issue

| Requirement | Where it is met |
|---|---|
| Issue: the design covers plot delineation, alert confirmation, fumigation, verification and harvest estimation. | Section 3, with records in section 4, transitions in section 5 and triggers in section 8. |
| Issue: the fumigation order requires a polygon, the authorizing human identity, who flies, a time window and the product batch identifier. | Section 4.2 (fields) and section 5.2 (`draft` and `authorized` guards: polygon version, agronomist with fresh MFA, pilot with permit, window, batch). |
| Issue: if the verification photo or flight path does not arrive, the case stays open. | Section 5.1 (`verifying` stays), section 5.2 (`executed` to `closed` guard), section 5.3 (`flown` until the manifest is complete). |
| Issue: the review campaign, monthly in harvest and quarterly outside, is documented or justified. | Section 11; closes PD-012. |
| Issue: the mission is loaded before take-off and the record is published when the link is restored. | Sections 6.1 and 10. |
| "La plataforma crea la orden, la autoriza, la entrega al equipo de vuelo y guarda el resultado." | Draft created by the platform, authorized by the agronomist, dispatched by package download, result kept as flight record and objects (sections 5.2, 6 and 7). |
| "Los cinco entran. No se sustituye uno por otro." | Section 3: each guard names its mission type. |
| "La orden de fumigación lleva: finca, lote, motivo, ventana, operador y quién autoriza." and "La plataforma guarda quién autorizó, quién voló, la ventana, el identificador del lote de producto usado y el cierre del vuelo." | Sections 4.2 and 6.2. |
| "Si no hay polígono del lote o no hay cierre del registro, la orden no sale." | Section 5.2, `draft` guard. |
| "Esas zonas viven en la plataforma antes del primer vuelo." | Section 6.1: no-fly zones of the polygon version travel in every package, including the first delimitation (ASM-027). |
| "La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta." | Section 5: the only actor of `authorized` is the agronomist; `authorizer_id` is taken from the token. |
| Part E: function output in the same record, with timestamp, confidence and source object identifier; on failure the state does not change. | Section 8. |
| Fotos, mapas, polígonos y registros de vuelo en object storage; órdenes en base transaccional; retention 5 years. | Sections 4 and 7. |
| ADR-002 to this task: state machine, mission package, prefixes, flight record, inference triggers, PD-121, PD-032. | Sections 5, 6, 7, 8 and 9; PD-032 closed by sections 3 to 5. |
| A09 to this task: drone kit assignment, upload-grant record and quarantine (PD-104). | Section 4.2 (`kit_thing_name`), section 5.2 (`dispatched` guard), section 7 (grant keys, quarantine prefix). |

## 14. Decision-log entries

| ID | Summary |
|---|---|
| PD-012 | Cadence kept. Closed by section 11. |
| PD-032 | Case and order states, switching events and order fields implemented. Closed by sections 3 to 5. |
| PD-104 | Drone kit assignment, upload grant and quarantine in the order and flight record design. Closed by sections 4.2, 5.2 and 7. |
| PD-121 | Ingress endpoints named; `drone_object_ingress` holds none in the pilot; CC-DRN-02 and CL-DRN-01 narrowed. Closed by section 9. |
| ASM-140 | No interface endpoint in `drone_object_ingress` in the pilot; reception function not VPC-attached. |
| ASM-141 | Alerts on a plot with an open case join it; re-confirmation and re-verification by the agronomist; the order closes on any persisted verification result, the case only on `case_closed`. |
| ASM-142 | The platform creates the draft order; rejection and revocation end in `expired` with an `end_reason`; out-of-window and late records are flagged; a late in-window record turns `expired` into `executed`. |
| ASM-143 | `flown` comes from the kit's `flight_completed` status message or the first object of the mission. |
| ASM-144 | Mission package as JSON with GeoJSON in WGS 84 and a SHA-256 digest echoed by the flight record. |
| ASM-145 | S3 key scheme by farm, case or campaign and mission; opaque identifiers only; quarantine prefix; objects before record; deletion by receipt. |
| ASM-146 | Flight record fields; any persisted fumigation record makes the order `executed`, an aborted closure is flagged. |
| ASM-147 | A delimitation creates a new polygon version activated by the reception function; orders and packages pin the version. |
| ASM-148 | Campaign record per department and period; start dates set by the Federation; campaign pilots assigned by the agronomist. |
| ASM-149 | Inference starts on `mission_record_persisted`, not on raw object arrival. |
| PD-140 | B04 aligns CL-LNK-01, CL-LNK-07 and CL-DRN-04 with the empty `drone_object_ingress` tier. |
| PD-141 | A21 sets alarm ages for missions without a record and open fumigations. |
| PD-142 | Re-check the AWS features named here against live documentation. |
| PD-143 | The drone operator contract requires ground-station software that imports the package and returns the record and manifest as specified. |

## 15. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Drone section, Qué se actualiza (drone flow), Part B, Part E, points 1, 2 and 6, Epic 4, Scope. |
| [`docs/decision-log.md`](decision-log.md) | ASM-001, ASM-020 to ASM-029, ASM-056, ASM-092, ASM-093, ASM-095, ASM-122, ASM-126, PD-014, PD-015, PD-031, PD-111, and the entries of section 14. |
| [ADR-002](adr/adr-002-drone-flight-mode-and-governance.md) | Mode, events, services, order fields and states, governance, cadence. |
| [`docs/connectivity.md`](connectivity.md) | Section 6: drone exit protocol, buffer, paths. |
| [`docs/collection-center-network.md`](collection-center-network.md) | Section 4.1: ground station placement. |
| [`docs/firewall-policy.md`](firewall-policy.md) | DRN, APP and TEL rules; PD-121. |
| [`docs/security-and-identity.md`](security-and-identity.md) | Sections 4, 6, 7, 9 and 10: roles, drone kit, upload grant, keys, retention, revocation. |
| [`docs/landing-zone.md`](landing-zone.md) | Sections 5.3 and 7.2: cost per farm prefix, audit of flight object delivery. |
| `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13, aroma-pacifico/aroma-pacifico-diagrams#2) | Drone object data path and its open points PD-110 to PD-112. |
| [RFC 7946, The GeoJSON Format](https://www.rfc-editor.org/rfc/rfc7946) | Geometry format and WGS 84 coordinates of the mission package. |
| [Uploading objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html) and [Checking object integrity](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity.html) | Upload and checksum mechanism, as cited by A10. |
