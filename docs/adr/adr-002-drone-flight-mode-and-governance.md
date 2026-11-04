# ADR-002: Drone flight mode and operational governance

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-11-04 |
| Owner | SR1 / SR2 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#3 |
| Supersedes | none |
| Superseded by | none |

## Context

The case statement asks for a drone service, not a loose flight: the platform creates the order, authorizes it, hands it to the flight team and keeps the result ("La plataforma crea la orden, la autoriza, la entrega al equipo de vuelo y guarda el resultado"). Five services are mandatory and none replaces another: delimit the plot, confirm the sensor alert, fumigate, verify after fumigating and estimate the harvest. The design must choose precision agriculture, area agriculture or a combination, sustain it in the defense, and, if combined, name the event that passes from one mode to the other. It must also state who flies, who authorizes a fumigation, with which permit and how often in the pilot, because the farmer does not fly the drone ("El dron no lo opera cada caficultor por su cuenta").

Four rules of the case statement constrain every option. A fumigation order cannot leave without a plot polygon and a closed flight record. A fumigation stays open until a verification object arrives. The mission is preloaded before take-off, at the municipal seat or at the collection center, because the link may drop over the plot; the flight record is published when the link returns, and the fumigation does not close until that record is persisted. Fumigation authorization is a human identity, and no trigger executes it.

The owner took the decision on 2026-11-04: combined mode, a contracted certified drone operator flies, and a Federation agronomist authorizes each fumigation as a human identity with MFA. This record states the mode boundary, maps the five services, fixes the governance and the cadence, and registers what the case statement leaves open as [ASM-020](../decision-log.md#assumptions) to [ASM-029](../decision-log.md#assumptions) and [PD-030](../decision-log.md#pending-decisions) to [PD-032](../decision-log.md#pending-decisions). Operating the drone, dose, mixture, product choice and the flight plan of each farm are out of scope, as the case statement says.

## Decision

The pilot uses combined mode: area mode delimits each farm and estimates the harvest, and precision mode confirms a crop alert, fumigates the confirmed focus and verifies that same plot. A contracted certified drone operator flies every mission, and a Federation agronomist, authenticated as a human identity with MFA, authorizes each fumigation order individually.

### Mode boundary and switching events

Area mode is the steady state of every farm. A farm enters precision mode only through a crop alert, and only a confirmed focus can lead to a fumigation order. The events below are persisted records in the platform, not messages that a device sends directly.

| Event | Produced by | Effect |
|---|---|---|
| `crop_alert_raised` | The telemetry batch function, when it marks a reading outside the Federation-configured threshold of a node located inside a delimited plot. | Opens a pest case for that plot and switches the plot from area mode to precision mode. A confirmation mission targets the node location (the focus), not the whole farm. If the farm has no polygon yet, a delimitation mission is queued first (ASM-027). |
| `focus_confirmed` | The confirmation function, which reads the persisted confirmation object, calls managed vision inference and persists a confirmed result with timestamp, confidence and source object identifier. | Makes the case eligible for a fumigation order. It does not authorize anything: only the agronomist does. |
| `focus_not_confirmed` | The same function, when inference does not confirm the focus. | The agronomist closes the case without fumigation and the plot returns to area mode. |
| `case_closed` | The verification function, when the verification object shows that the case can close, and only after the fumigation flight record is persisted. | The plot returns to area mode. |

The mode switch from area to precision is therefore the pair `crop_alert_raised` then `focus_confirmed`: the alert opens the precision cycle with a confirmation flight, and the confirmation is the only path to a fumigation. The return to area mode is `focus_not_confirmed` or `case_closed`.

### The five mandatory services

| Service | Mode | Trigger | Inputs the platform provides | Record returned | Gate enforced by the platform |
|---|---|---|---|---|---|
| Delimit the plot | Area | Farm enrolled in the pilot, or a plot change reported by the agronomist; flown in the next review campaign. | Farm and crop zone identifiers, declared plots, and the preliminary no-fly zones (houses, wet mill, roads, water sources) registered before the first flight (ASM-027). | Plot polygons, confirmed no-fly zones and map objects in object storage; flight record. | No fumigation order can be created for a plot without a persisted polygon. |
| Confirm the sensor alert | Precision | `crop_alert_raised`. | Case identifier, plot polygon, focus coordinates of the node, no-fly zones. | Confirmation object (photos of the focus) and flight record. The function then persists `focus_confirmed` or `focus_not_confirmed`. | No authorization is possible without `focus_confirmed` and a closed confirmation flight record ("Sin cierre de ese registro no se autoriza la fumigación"). |
| Fumigation | Precision | Fumigation order authorized by the agronomist. | The order fields below, plus the plot polygon and no-fly zones, preloaded as one mission package. | Fumigation flight record: who flew, actual window, product batch identifier used, and closure. | The order does not leave without polygon and closed confirmation record. The order does not reach `executed` until the fumigation flight record is persisted. |
| Verify after fumigating | Precision, same plot | Fumigation flight record persisted. | Case identifier, plot polygon, focus coordinates. | Verification object (photo or short pass) and flight record. The function persists whether the case can close. | Without a persisted verification object, or if the invocation fails, the fumigation stays open and the state does not change. |
| Estimate the harvest | Area | Once per harvest per farm, scheduled in the first review campaign of each harvest period (ASM-024). | Farm polygon, plots, no-fly zones. | Estimation object and flight record. The function persists the estimated number of sacks for the farm's collection center. | Never fumigates and generates no packing order. |

### Fumigation order

The fumigation order is a row in the transactional database. Its fields follow the case statement list (farm, plot, reason, window, operator, authorizer) plus the product batch identifier the platform must keep.

| Field | Content | Rule |
|---|---|---|
| `farm_id` | Farm identifier. | Must be enrolled in the pilot. |
| `plot_id` | Plot identifier. | Must have a persisted polygon. |
| `reason` | Pest case identifier and the `focus_confirmed` result that justifies it (source object identifier, confidence, timestamp). | Must reference a confirmed focus with a closed confirmation flight record. |
| `window` | Start and end of the authorized flight window. | An order whose window ends without a fumigation flight record expires and needs a new authorization (ASM-029). |
| `operator_id` | Identity of the certified pilot of the contracted operator, and the reference and expiry of the permit under which they fly (ASM-020). | Must be a drone operator identity with MFA and an unexpired permit reference. |
| `authorizer_id` | Identity of the Federation agronomist and the time of the MFA-backed authorization. | Must be an agronomist identity; must differ from `operator_id`; never a service identity or a trigger. |
| `product_batch_id` | Identifier of the product batch planned for the application; the flight record carries the batch identifier actually used. | Stored as an identifier only. The product, dose and mixture are not chosen by the platform. |

The order moves through the states `draft`, `authorized`, `dispatched` (mission package preloaded), `flown` (record still in the drone or ground station buffer), `executed` (flight record persisted), `closed` (verification persisted) or `expired`. A fumigation is counted as open, and is reported by the observability of point 6, while it is in any state before `closed`.

### Mission preload and flight record

The drone operates far from the farmer's phone. Before travelling to the farm, the operator downloads the mission package (order, polygon and no-fly zones) to the ground station at the municipal seat or at the collection center, where a data link exists. If the link drops over the plot, the mission is already loaded and the flight record and objects stay in the buffer of the drone or ground station. They are published through the drone object ingress path when the link returns, at the latest when the team is back at the municipal seat or the collection center (ASM-028). Each delivered flight object is audited, as point 1 requires. The connectivity and the data path themselves are designed by the later connectivity tasks; this ADR fixes only the preload points and the closure rule.

### Who flies, who authorizes, with which permit

| Role | Who | Platform identity | What it may do |
|---|---|---|---|
| Flies | Certified pilots of a contracted drone operator company (ASM-023). The farmer never flies. | Drone operator, one identity per pilot, MFA (point 2). | Download preloaded missions assigned to it, upload flight records and objects. It cannot create or authorize an order. |
| Authorizes each fumigation | A Federation agronomist (ASM-021). | Agronomist, one identity per person, MFA. | Authorize or reject one fumigation order at a time, close a case with no confirmed focus, report plot changes. It cannot fly or upload flight records. |
| Runs the platform | Platform operator. | Platform operator, MFA. | Operate the platform; it cannot authorize a fumigation. |

The case statement lists the minimum identities of point 2; the agronomist identity is added because the authorizer must be a human identity distinct from the person who flies. A lost drone, or a pilot who leaves the contract, has its identity revoked so that it stops publishing, as point 2 requires.

The permit framework is that of the Colombian civil aviation authority (Aerocivil) for unmanned aircraft. No regulation or article number is cited here, because none was verified against an official source in this task. The requirement is recorded as ASM-020: the contracted operator holds the Aerocivil authorization that applies to agricultural spraying with unmanned aircraft, its aircraft are registered as Aerocivil requires, and each pilot holds the competence certificate Aerocivil requires. The platform stores the permit reference and expiry for the operator and each pilot, and refuses to assign an order to a pilot whose permit reference is missing or expired. The exact regulation is to be verified in PD-030. Permits outside aviation, such as rules on applying agricultural products, belong to the operator and the Federation and are out of scope (ASM-022).

### Campaign cadence

The pilot keeps the cadence of the case statement: during harvest, one review campaign per month plus the fumigations that open cases produce; outside harvest, one review campaign every three months. The cadence is kept because it already matches the services. Harvest is when the collection centers receive up to 400 sacks a day each and when the estimation flight is useful, so the monthly campaign also carries the once-per-harvest estimation of the 200 farms. Outside harvest there is no estimation to fly, and pest work is driven by alerts rather than by the calendar.

A review campaign visits the farms of a department and flies, in this order, the pending delimitations, the confirmations of open alerts, the verifications of fumigations already flown and, in the first campaign of each harvest, the estimation flights (ASM-025). Fumigations, and the confirmation of an alert the agronomist does not want to wait for, are flown as extra missions between campaigns, as part of the work that open cases produce (ASM-026).

Flight volumes follow from the case statement figures and do not depend on any invented rate:

| Flight type | Volume |
|---|---|
| Delimitation | 200 flights, one per farm, before the first fumigation; repeated only for a reported plot change. |
| Harvest estimation | 200 flights per harvest period, one per farm. |
| Review campaigns per year | H + ceil((12 - H) / 3), where H is the number of harvest months per year that the Federation configures (ASM-024). |
| Confirmation, fumigation and verification | One confirmation per `crop_alert_raised` case, and at most one fumigation and one verification per `focus_confirmed` case. The case statement gives no rate of pest cases, so no count is assumed here; the cost task uses the formula and states its own rate (PD-031). |

## Alternatives considered

| Alternative | Advantages | Disadvantages | Reason rejected |
|---|---|---|---|
| Precision mode only | Smallest flight footprint; every flight targets a focus. | The case statement defines precision as confirm, fumigate and verify; it does not produce the polygon or the harvest estimate. | Two of the five mandatory services would be missing, and they cannot be substituted. |
| Area mode only | Simple scheduled campaigns over whole farms. | The case statement defines area mode as delimit and estimate, and it does not fumigate the whole area; it also contradicts "El dron va al foco que marcó la lectura". | Confirmation, fumigation and verification would be missing. |
| Combined mode (chosen) | Covers the five services with the mode the case statement assigns to each; one event chain from sensor to fumigation. | Requires a defined switching event and a case state machine. | Chosen. |
| The farmer flies | No contractor cost. | Explicitly excluded by the case statement; farmers have little budget and no certification. | Prohibited. |
| Federation in-house pilots and fleet | Direct control of crews. | Capital cost of drones, certification and maintenance for a 200-farm pilot. | A contracted service fits the low-investment criterion and keeps the field team separate from the platform team, as the case statement allows. |
| Automatic authorization by a trigger or rule | Fast. | "La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta." | Prohibited. |
| Authorization by the drone operator or the platform operator | No extra identity. | No separation between who flies and who authorizes; no agronomic competence. | Fails separation of duties. |

## Consequences

Combined mode makes the pest flow traceable from one sensor reading to one closed case, and every fumigation is backed by a polygon, a confirmed focus, a named human authorizer and two persisted flight records. In exchange the platform has to keep a case state machine, an agronomist identity in addition to the six identities of point 2, and a permit register for the contracted operator. A fumigation can stay open for days when the link is down; this is the intended behavior and is made visible by observability rather than hidden.

| Dimension | Impact |
|---|---|
| Network | Uses the drone object ingress subnet of point 3. Missions are preloaded and flight records uploaded at the municipal seat or the collection center, so the collection center design must allow the drone ground station to reach the drone object ingress endpoint over the WAN; the rule itself is classified in PD-011. |
| Security | Two MFA human identities (drone operator, agronomist) with separation of duties enforced on each order; no service identity can write `authorizer_id`. Authorization, delivery of each flight object and permit changes are audited. Lost drones and departing pilots are revoked. |
| IaC | None in the Epic 6 code: the drone path is designed in the architecture but is not the pattern module. The network module still declares the drone object ingress subnet. |
| Costs | No drone is bought; the drone campaign is a contracted service whose count follows the formula above. The campaign cost is priced by aroma-pacifico/aroma-pacifico-roadmap#36 under PD-017 and PD-031. |

## Compliance with the case statement

| Case statement passage | How this decision complies |
|---|---|
| "El diseño elige agricultura de precisión, agricultura de áreas, o una combinación, y lo sostiene." (The design chooses precision, area or combined agriculture, and sustains it.) | Combined mode, justified against the two single-mode alternatives. |
| "Combinación: áreas para el polígono y la estimación, precisión para el foco y la fumigación. El diseño dice qué evento pasa de un modo al otro." (Combined: area for the polygon and the estimate, precision for the focus and the fumigation. The design says which event passes from one mode to the other.) | [Mode boundary and switching events](#mode-boundary-and-switching-events): `crop_alert_raised` then `focus_confirmed`; return on `focus_not_confirmed` or `case_closed`. |
| "Los cinco entran. No se sustituye uno por otro." (All five are in. None replaces another.) | [The five mandatory services](#the-five-mandatory-services) maps each service to its mode, trigger, inputs and record. |
| "El diseño dice quién vuela, quién autoriza una fumigación, con qué permiso y con qué frecuencia en el piloto." (The design says who flies, who authorizes a fumigation, with what permit and how often.) | [Who flies, who authorizes, with which permit](#who-flies-who-authorizes-with-which-permit) and [Campaign cadence](#campaign-cadence); the permit is ASM-020, to verify in PD-030. |
| "La orden de fumigación lleva: finca, lote, motivo, ventana, operador y quién autoriza." (The order carries farm, plot, reason, window, operator and authorizer.) "La plataforma guarda quién autorizó, quién voló, la ventana, el identificador del lote de producto usado y el cierre del vuelo." | [Fumigation order](#fumigation-order) holds every field plus `product_batch_id`; the flight record carries the batch actually used and the closure. |
| "Si no hay polígono del lote o no hay cierre del registro, la orden no sale." (Without the plot polygon or the record closure, the order does not leave.) | Gates of the delimitation, confirmation and fumigation services. |
| "Una foto o un recorrido corto cierra el caso. Si ese archivo no llega, la fumigación sigue abierta." (A photo or a short pass closes the case; if that file does not arrive, the fumigation stays open.) | Verification gate and the `closed` state. |
| "Se preposiciona en cabecera municipal o en el centro de acopio. El registro de vuelo se publica al restablecer el enlace." (The mission is prepositioned at the municipal seat or the collection center; the flight record is published when the link returns.) | [Mission preload and flight record](#mission-preload-and-flight-record). |
| "Esas zonas viven en la plataforma antes del primer vuelo." (No-fly zones live in the platform before the first flight.) | Preliminary no-fly zones are registered before the delimitation flight (ASM-027). |
| "La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta." (Fumigation authorization is a human identity; no trigger executes it.) | Agronomist identity with MFA; functions only persist inference results. |
| "En cosecha hay una campaña de revisión al mes [...] Fuera de cosecha, una campaña de revisión cada tres meses." (Monthly review campaign during harvest, quarterly outside.) | Cadence kept, with volumes derived from the 200 farms. |
| "No se pide operar el dron ni definir dosis, mezcla o producto." (Operating the drone, dose, mixture or product are not requested.) | Out of scope; the platform stores only the product batch identifier. |

## References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](../case/case-statement.pdf) | Drone section, Part B, Part E, points 1, 2, 3 and 6, Scope. |
| [Decision log](../decision-log.md) | ADR-002 row, ASM-020 to ASM-029, PD-002, PD-011, PD-012, PD-015, PD-017, PD-030 to PD-032. |
| aroma-pacifico/aroma-pacifico-roadmap#3 | This task. |
| aroma-pacifico/aroma-pacifico-roadmap#15 | Drone design that keeps this cadence (PD-012) and implements the case state machine (PD-032). |
| aroma-pacifico/aroma-pacifico-roadmap#17 | Inference functions, retries and treatment of objects that contain a person (PD-014, PD-015). |
| aroma-pacifico/aroma-pacifico-roadmap#36 | Drone campaign cost (PD-017, PD-031). |
| Aerocivil, Colombian civil aviation authority | Permit framework for unmanned aircraft; no provision cited until verified (PD-030). |
