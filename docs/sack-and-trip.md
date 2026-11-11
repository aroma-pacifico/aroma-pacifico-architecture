# Sack, Trip and Public Lookup

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-04 |
| Owner | SR1 / JR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#16 (A16) |
| Builds on | [`docs/connectivity.md`](connectivity.md) (A10, sections 5, 7 and 10), [`docs/security-and-identity.md`](security-and-identity.md) (A09, sections 3.3, 4.2, 7.2 and 8), [`docs/landing-zone.md`](landing-zone.md) (A08, section 7.2), [`docs/firewall-policy.md`](firewall-policy.md) (A12, PUB rules), [`docs/collection-center-network.md`](collection-center-network.md) (A11, section 5), [`docs/clarification-questions.md`](clarification-questions.md) (ASM-057, ASM-058), [`docs/addressing-plan.md`](addressing-plan.md) (A07, section 4.2), [ADR-001](adr/adr-001-primary-and-second-cloud.md), `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13) |
| Feeds | A19 architecture document and complete diagram (aroma-pacifico/aroma-pacifico-roadmap#19), A21 observability and SLOs (aroma-pacifico/aroma-pacifico-roadmap#34), A23 cost (aroma-pacifico/aroma-pacifico-roadmap#36), the release notes template of an application change, the design system |
| Decision-log entries | PD-051, PD-096 and PD-102 (closed here), PD-110 (closed here; copied into the log from A13), ASM-150 to ASM-159, PD-150 to PD-155 |

## Diagrams

- [Sack and Trip Custody Sequence (SVG)](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/flows/sack-and-trip-sequence.svg) ([PNG](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/flows/sack-and-trip-sequence.png)): Models the interaction lifecycle for sack registration, transporter custody transfers across transit legs, and unauthenticated buyer QR code verification.
- [Sack Lifecycle State Machine (SVG)](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/flows/sack-states.svg) ([PNG](https://github.com/aroma-pacifico/aroma-pacifico-diagrams/blob/main/flows/sack-states.png)): Specifies the finite state transitions for 60 kg coffee sacks from wet mill creation to port delivery and commercial sale.

## 1. Purpose and requirements

Part C of the [case statement](case/case-statement.pdf) is the requirement, together with the sack lookup of the pilot capabilities ("Consulta de un saco, por código, para el caficultor, el centro de acopio y el comprador") and the code sheets of Part D. Quotations are in the original Spanish, which prevails over the English rendering (ASM-001). No figure in this document is a measurement: capacities are arithmetic on the case figures, and recovery times are design targets to be verified by a future rehearsal (PD-153).

| Case statement text | Where it is met |
|---|---|
| "Estados del saco: en preparación, en viaje al acopio, en acopio (aceptado o rechazado), en viaje al puerto, entregado en puerto (aceptado o rechazado)." | [Section 3](#3-sack-states-and-transitions) |
| "Alta del saco en la finca, con peso e identificador." and "El café sale del beneficiadero en sacos de 60 kg." | [Section 2](#2-registering-the-sack-at-the-farm) |
| "Paso del saco al transportador y llegada al centro de acopio." "Salida hacia Buenaventura y entrega." | [Section 3.2](#32-transition-table), [section 5](#5-the-trip) |
| "Cada evento guarda fecha, hora y responsable." | [Section 4](#4-sack-events-date-time-and-responsible) |
| "La consulta pública lee un saco y muestra solo la vista del comprador." | [Section 7](#7-public-lookup) |
| "En cosecha, cada centro de acopio recibe hasta 400 sacos al día." "Consulta del saco en cosecha: picos de 30 lecturas por minuto." | [Section 8](#8-peak-load-arithmetic) |
| "Un despliegue no pierde eventos de saco. RPO de esos eventos: 15 minutos. RTO de la consulta pública: 4 horas." | [Section 9](#9-recovery-objectives) |
| "Impresión de hojas de códigos para los sacos que lleguen sin código." (Part D) | [Section 6](#6-sack-code-and-printed-code-sheet) |
| "Eventos de saco y órdenes de fumigación: base transaccional." (Part E) | [Section 10](#10-stores) |

The accepted inputs fix the frame and are not reopened. A09 fixes the sack code (80 random bits, ASM-094), the buyer-view fields (ASM-096), the projection read by a role without access to transactional or personal data, the scopes of each role and the `subject_id` pseudonym. A10 fixes the transporter events `transport_position` and `sack_custody`, their fields, the 5-minute interval, the outboxes that release on acknowledgement (ASM-076, ASM-079) and the port delivery recorded by the transporter (ASM-057, ASM-077). A08 fixes the audit of each sack read, written by the lookup service with `anonymous` for a public read (ASM-069). A12 fixes the WAF of the lookup front door (CL-PUB-01, ASM-129) and leaves its threshold here. ASM-058 makes the lookup unauthenticated. ADR-001 places everything in `us-east-1` and meets the RPO and RTO inside that Region (ASM-004). ADR-003 makes ingestion, not the lookup, the coded pattern module, so this design is not coded in the 100 hours.

## 2. Registering the sack at the farm

The sack is born at the wet mill, where the coffee leaves in 60 kg sacks. The farmer registers it in the farmer app, which has few screens, with one action per sack:

1. Scan the QR of a pre-printed label from a batch allocated to the farm (section 6). The phone camera reads it without data, so no code is ever generated on the phone (A09 section 8.1).
2. Enter the weight shown by the scale, in kilograms with one decimal. The 60 kg of the case statement is the reference, not a forced value: the measured weight is stored, and a weight outside a tolerance around 60 kg set by the Federation is flagged to the farmer and to the center, never refused (ASM-150).
3. Confirm the drying lot the coffee came from, which the app proposes as the farm's last closed lot. The lot links the sack to its drying dates and drying variables for the buyer view (ASM-096).

The app writes a `sack_registered` event to its encrypted outbox and sends it to the application API when it has data (A10 section 5). The event carries the identifier (the code), the weight, the farm, the drying lot, the farmer as responsible and the date and time it was recorded. The platform assigns an internal `sack_id` when the event is persisted; the code is an index to it, so a replaced label never changes the history (A09 section 8.1). From that moment the sack is in the state `in_preparation`.

## 3. Sack states and transitions

### 3.1 States

The seven states below are the five states of Part C with their accepted or rejected outcome split out, so that every state the buyer sees is exactly one Part C state (A09 section 8.2). The palette of the design system has four sack states (en preparación, en viaje, aceptado, rechazado); the last column maps each state to them (PD-154).

| State | Part C state | Terminal | Design system badge |
|---|---|---|---|
| `in_preparation` | en preparación | No | en preparación |
| `in_transit_to_center` | en viaje al acopio | No | en viaje |
| `at_center_accepted` | en acopio (aceptado) | No | aceptado |
| `at_center_rejected` | en acopio (rechazado) | Yes (ASM-151) | rechazado |
| `in_transit_to_port` | en viaje al puerto | No | en viaje |
| `delivered_at_port_accepted` | entregado en puerto (aceptado) | Yes | aceptado |
| `delivered_at_port_rejected` | entregado en puerto (rechazado) | Yes | rechazado |

A sack rejected at the center does not travel to the port in the pilot. If the farmer reprocesses that coffee and presents it again, it is registered as a new sack with a new code; the rejected sack keeps its history (ASM-151).

### 3.2 Transition table

Each transition is caused by one event, recorded by one actor whose identity and scope the API checks (A09 sections 3.3 and 4.2). The guard is the condition on the sack and the trip that the transition requires.

| # | From | Event | Actor and app | Guard | To |
|---|---|---|---|---|---|
| T1 | (none) | `sack_registered` | Farmer, farmer app | The code is `issued` to a batch of one of the caller's farms and is not bound to a sack; weight greater than zero; the drying lot belongs to the same farm. | `in_preparation` |
| T2 | `in_preparation` | `picked_up_at_farm` (`sack_custody`) | Transporter, transporter app | The trip is open, of type `farm_to_center`, owned by the caller, and the sack's farm is a pickup point of the trip. | `in_transit_to_center` |
| T3 | `in_transit_to_center` or `in_preparation` | `received_at_center` with `outcome` `accepted` | Collection-center operator (MFA), logistics app | The caller's center is the trip's destination center; for a sack still in `in_preparation`, the missing pickup is flagged (section 3.3). | `at_center_accepted` |
| T4 | Same as T3 | `received_at_center` with `outcome` `rejected` and a `reason` | Same as T3 | Same as T3; a reason is required. | `at_center_rejected` |
| T5 | (none) | `bound_at_center` with `outcome` | Collection-center operator (MFA), logistics app | A sack that arrives without a code: the code is from the caller's center reserve (section 6.3), the farm of origin is one that delivers to that center, and the weight is measured at reception. | `at_center_accepted` or `at_center_rejected` |
| T6 | `at_center_accepted` | `departed_to_port` (`sack_custody`) | Transporter, transporter app | The trip is open, of type `center_to_port`, owned by the caller, and its origin is the sack's center; the center operator has assigned the sack to the trip (`assigned_to_trip`). | `in_transit_to_port` |
| T7 | `in_transit_to_port` or `at_center_accepted` | `delivered_at_port` (`sack_custody`) with `outcome` `accepted`, `receiver_reference` | Transporter, transporter app (ASM-057, ASM-077) | The sack is on the caller's open `center_to_port` trip; for a sack still in `at_center_accepted`, the missing departure is flagged. | `delivered_at_port_accepted` |
| T8 | Same as T7 | `delivered_at_port` with `outcome` `rejected`, `receiver_reference`, `reason` | Same as T7 | Same as T7; a reason is required. | `delivered_at_port_rejected` |

Five further events are recorded with date, time and responsible but do not change the state: `arrived_at_center` (the transporter hands the load over at the center, A10 section 7), `assigned_to_trip` (the center operator puts accepted sacks on an outbound trip), `code_replaced` (a damaged label is replaced by a code from the center reserve and the old code becomes `void`), `weight_corrected` and `outcome_corrected` (section 3.3).

### 3.3 Rules that make the table work offline

Every actor of the chain can record while offline, so events reach the platform out of order: a center often records a reception before the transporter's pickup has drained from a phone that lost data on a feeder road. Four rules keep the state correct (ASM-152).

| Rule | Design |
|---|---|
| Refuse only what the caller cannot do | The API refuses an event, with an error the app shows, only when the caller is outside its role or scope, the code is unknown, `void` or allocated to another farm or center, or a required field is missing. Everything else is persisted, because it records a physical fact that already happened. |
| Order by the time of the act | The state of a sack is recomputed from its whole history, ordered by `recorded_at` and then `event_id`, each time an event for it is persisted. The result does not depend on arrival order. |
| The physical receiver is authoritative | A transition recorded by the party that now holds the sack (T3, T4, T7, T8) is applied even if the previous step is missing, and the missing step is flagged for follow-up. A guard failure that is not covered by this rule leaves the state unchanged and marks the event `anomaly`. |
| Events before the sack exists wait | An event for a code that is issued but not yet registered is held as pending and applied when the registration drains (A10 section 5). Pending and anomalous events, and events whose `recorded_at` is more than 5 minutes after their `received_at` (a phone clock ahead), are listed for observability (PD-151). |

No event is edited or deleted. A mistake is corrected by a compensating event (`weight_corrected`, `outcome_corrected`) recorded by the same role and scope that recorded the original, with a reason; the history shows both (ASM-151).

## 4. Sack events: date, time and responsible

Every sack event, whichever app records it, has the fields below. They extend the `sack_custody` fields of A10 section 7 to the farm and center events.

| Field | Content |
|---|---|
| `event_id` | UUID generated by the app; idempotency key (A10 section 2). |
| `event_type` | One of the events of section 3.2. |
| `sack_codes` | Sack codes read in the act; one event can cover a whole load. |
| `trip_id` | Trip of the act; empty for `sack_registered`. |
| `recorded_at` | UTC date and time of the act, to the millisecond, taken on the device. Displayed in Colombia time (`America/Bogota`, ASM-007). |
| `received_at` | UTC time the API persisted the event; set by the API. |
| `responsible_subject_id`, `responsible_role` | The authenticated person who performed the act, by pseudonym and role (A09 section 9.1); checked against the token, never taken from the payload. |
| `device_id` | App installation identifier (A09 section 5.2). |
| `center_id` | Center of the act, for center events. |
| `latitude`, `longitude` | Position of the act, for transporter events (A10 section 7). |
| `weight_kg`, `drying_lot_id` | For `sack_registered`, `bound_at_center` and `weight_corrected`. |
| `outcome`, `reason`, `receiver_reference` | For receptions, deliveries and corrections. |
| `sequence_no`, `link_state` | Per-trip counter and link state at the time of the act (A10 section 7). |

The four acts the issue names are therefore recorded as follows.

| Act | Event | Date and time | Responsible |
|---|---|---|---|
| Handover to the transporter | `picked_up_at_farm` | `recorded_at` on the transporter phone | Transporter |
| Arrival at the collection center | `arrived_at_center` (handover) and `received_at_center` (acceptance or rejection) | `recorded_at` on each device | Transporter and collection-center operator |
| Departure to the port | `assigned_to_trip` and `departed_to_port` | `recorded_at` on each device | Collection-center operator and transporter |
| Delivery at the port | `delivered_at_port` | `recorded_at` on the transporter phone | Transporter, with the port receiver's reference (ASM-077) |

## 5. The trip

A trip is one leg of custody carried by one transporter (ASM-159). A `farm_to_center` trip opens when the transporter accepts the first sack at a farm, may pick up at several farms, and closes when its load is handed over at one destination center. A `center_to_port` trip opens at the departure from a center and closes when its last sack is delivered at Buenaventura. "En route" in A10 section 7 is the time a trip is open, so positions are taken only then.

The trip record holds the type, the transporter's `subject_id`, the vehicle plate reference (personal data, kept in the personal-data table), the origin and destination, the opening and closing times and the codes of its sacks. Its trace is the ordered set of its `transport_position` events plus its custody events, rebuilt by `recorded_at` and `sequence_no` (A13). The collection-center operator reads trips to or from the own center while they are open, which is "Acopio recibe el viaje" of point 2; the farmer reads the custody events of their own sacks without positions; the buyer reads none of it (A09 section 4.2).

## 6. Sack code and printed code sheet

### 6.1 Code format

The code is the A09 format (ASM-094), fixed here to the symbol:

| Element | Specification |
|---|---|
| Payload | 80 bits from a cryptographically secure generator, written as 16 symbols of the Crockford Base32 alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ`. |
| Check symbol | The 80-bit value modulo 37, written with the Crockford check alphabet, which adds `*`, `~`, `$`, `=` and `U` to the 32 symbols. It detects any single wrong symbol and any swap of two adjacent symbols. All five extra symbols are legal in a URL path. |
| Human form | `XXXX-XXXX-XXXX-XXXX-C`, 21 characters with hyphens. |
| Input normalization | Hyphens and spaces removed, lowercase raised, `I` and `L` read as `1`, `O` read as `0`, as the Crockford encoding defines, so a code typed from the label is accepted. |
| URL form | `https://<lookup-host>/s/XXXXXXXXXXXXXXXXC`, the 17 symbols without hyphens. The host is a Federation-owned name, not fixed here (ASM-158). |

### 6.2 The label and the code sheet

The code sheet is a sheet of adhesive labels, one label per sack, printed on the code-sheet printers of the center (A11 section 5, IPP on TCP 631, ASM-124). The number of labels per sheet is set by the label stock bought with the printer class, not by this design.

| Part | Content | Not shown |
|---|---|---|
| Each label | The QR code of the URL form; the human form in large monospaced type; the words "Consulta del saco" and the lookup host. | Farm, farmer, municipality, center, date, weight, batch, or anything else, so that a label found on a sack reveals nothing beyond the buyer view. |
| Sheet margin, outside the labels | Batch identifier, allocation (the farm, or the center reserve), print time, the printing operator, and "Sheet n of m". | It stays with the center or the farmer and is never stuck on a sack. |

A sheet can be printed again only while its codes are `issued` and not bound; each print and reprint is an application audit record with the operator. Codes follow one life cycle: `issued` to a batch, `bound` to one sack, or `void` when a label is damaged, lost or its batch is cancelled. A `void` code answers "not found" at the lookup.

### 6.3 Codes for a sack that arrives without one, with or without WAN

Codes are issued only by the platform (A09 section 8.1), and the center prints them locally. To let a center label a sack that arrives without a code while the WAN is down, which A11 left open, each center holds a reserve of codes issued to the center and printed in advance while the link is up: at least 400 unbound codes, one peak day of receptions, replenished by a new batch whenever the reserve falls below that number (ASM-153). The operator sticks a reserve label on the sack and records `bound_at_center` in the logistics app; the event waits in the outbox (ASM-079) and binds the code when the link returns. Because reserve codes are already issued and unique, no code is ever invented locally and no collision can occur. This closes PD-096.

## 7. Public lookup

### 7.1 Path

```text
buyer phone or browser
  -> https://<lookup-host>/s/<code>             (QR scan or manual entry)
  -> Amazon CloudFront distribution             (TLS 1.2 minimum, certificate in us-east-1)
       AWS WAF web ACL, scope CloudFront        (CL-PUB-01, rules of section 7.3)
       /assets/* and /       -> Amazon S3 bucket of static assets, origin access control
       /s/*                  -> Lambda function URL of the lookup function, origin access control
  -> lookup function (not attached to a VPC)
       GetItem on the buyer_view table          (key public_view, ASM-154)
       audit record to the pdn application audit log group (A08 section 7.2)
```

The lookup function validates the code (format, then check symbol), reads one item of the `buyer_view` projection by code, writes the audit record of A08 section 7.2 (`anonymous`, source IP from the `CloudFront-Viewer-Address` header, user agent, channel `web`, code, view `buyer`, result, request identifier) and returns a small server-rendered HTML page with the fields of ASM-096, or JSON when the request asks for `application/json`. A malformed code, a code with a wrong check symbol, an unknown code, an unbound code and a `void` code all return the same "not found" page with the same status, so the answer reveals nothing about which codes exist (A09 section 8.1); only the audit record and the metrics tell them apart. The page loads no third-party resource.

Registered buyers, farmers and collection-center operators read the same projection through the application API and their own apps, under their identities (ASM-058); those reads are audited with their `sub` and role by the same audit writer.

### 7.2 Front door: CloudFront, no load balancer (closes PD-051)

The front door is an Amazon CloudFront distribution with an AWS WAF web ACL, whose origins are a Lambda function URL with `AWS_IAM` authentication and a private S3 bucket, both reachable only through CloudFront origin access control (PD-150). The `public_query` tier therefore hosts neither an internet-facing load balancer nor NAT gateways: there is no load balancer to place there, and ASM-127 already removes NAT gateways from the pilot. The `public_query` subnets are still created by the network module, keep their network ACL (CL-PUB-04) and stay empty, so that a later load balancer would not need new addresses; rule CL-PUB-03 is not instantiated. This closes PD-051.

| Alternative | Reason rejected |
|---|---|
| CloudFront, WAF and a Lambda function URL with origin access control (chosen) | No hourly-charged resource; the WAF sits at the edge in front of every request; the origin cannot be called around the WAF; static assets are cached at the edge; one distribution serves the page and the data. |
| Amazon API Gateway regional REST API with WAF, behind or without CloudFront | Works, but adds a second managed front door and its request charge to a single GET operation; CloudFront is still needed for the static assets. |
| Application Load Balancer in `public_query` with WAF and a Lambda target | Charged per hour in every Availability Zone for a peak of half a request per second, and needs CL-PUB-03 to keep the origin behind CloudFront. Against "El caficultor invierte poco". |
| Lookup page served from the reports web, signed in | ASM-058 makes the lookup unauthenticated; a buyer scanning a QR at the port has no account. |

### 7.3 Protection against enumeration of sack codes

Enumeration is defended first by the code itself and then by the front door. Each layer is independent of the others.

| Layer | Control | Value |
|---|---|---|
| Code space | 80 random bits; at most 2^22 codes are ever issued in 5 years (ASM-156) | Probability that one random guess hits an issued code: at most 2^22 / 2^80 = 2^-58, about 3.5 x 10^-18. |
| Check symbol | Validated before any read | 36 of 37 random guesses are rejected without touching the table. |
| WAF, in evaluation order | `AWSManagedRulesAmazonIpReputationList`, `AWSManagedRulesKnownBadInputsRuleSet`, `AWSManagedRulesCommonRuleSet` (ASM-129); a rate-based rule per source IP, scoped down to `/s/`, blocking above 100 requests in a 5-minute window (ASM-155, closes the threshold of PD-102); an allow rule for GET and HEAD on `/`, `/assets/*` and `/s/*` with a path pattern of 17 code symbols; default action Block (CL-PUB-01). | A legitimate buyer checking a load of 400 sacks at 10 per minute stays at 50 per window; the whole pilot peak is 150 requests per window (section 8). |
| Function | Reserved concurrency of 10 (ASM-155) | Caps the cost and the load an attack can push past the WAF. |
| Uniform answers | Same "not found" for every miss; `X-Robots-Tag: noindex`; `Referrer-Policy: no-referrer`; `Cache-Control: no-store` | No oracle, no indexing of codes by search engines, no code leaked to another site. |
| Observation | Not-found ratio, WAF blocks and throttles per minute | Proposed to A21 as alarms (PD-151). |

The worst case shows the margin. An attacker with 10,000 source addresses, each kept just under the rate limit, makes 10,000 x 100 / 5 = 200,000 guesses per minute, about 1.05 x 10^11 a year. The expected number of issued codes found is at most 1.05 x 10^11 x 3.5 x 10^-18, about 3.7 x 10^-7 per year. Even a hit returns only the buyer view, which contains no personal data and no trace.

### 7.4 Caching

Static assets (style sheet, logo, icons, the manual entry page) are versioned files cached by CloudFront and by the browser. Lookup answers are not cached anywhere (`Cache-Control: no-store`), for three reasons: every sack read must reach the lookup service so that it is audited (A08 section 7.2), a cached answer would show a stale state after a reception or a delivery, and a cached "not found" would hide a sack that has just been registered. At the peak of 30 reads per minute, one read of one item per request needs no cache for capacity (section 8).

## 8. Peak load arithmetic

All inputs are case figures; the upper bounds deliberately assume that every step of every sack happens on the same peak day.

| Quantity | Arithmetic | Result |
|---|---|---|
| Sacks received per day, pilot | 400 sacks per center x 3 centers | 1,200 sacks per day |
| Sack event records per center per day, upper bound | 400 sacks x 7 events that can touch a sack in a day at one center (registration, pickup, arrival, reception, assignment, departure, delivery) | 2,800 records per day |
| Same, pilot | 2,800 x 3 | 8,400 records per day, 0.1 per second on average |
| Writes per applied sack event | One history item, one sack item, one projection item | 3 writes, so at most 25,200 writes per day |
| Burst of one load | One custody event for L sacks becomes 3 L writes, applied from the stream in batches; for L = 400 | 1,200 writes, absorbed by the stream and retried on throttling |
| Sacks registered in 5 years, upper bound (ASM-156) | 1,200 x 365 x 5 | 2,190,000, below 2^22 = 4,194,304 |
| Transporter positions per day | 60 transporters x 12 positions per hour x 10 hours (Part A) | 7,200 positions per day |
| Positions in 5 years, upper bound | 7,200 x 365 x 5 | 13,140,000 items |
| Drain burst of one phone | 7 days of positions (ASM-076) | 2,016 positions, sent in batches |
| Public lookup peak | 30 reads per minute | 0.5 reads per second, 150 per 5 minutes, at most 43,200 per day if the peak lasted 24 hours |
| Work per lookup | 1 WAF evaluation, 1 function invocation, 1 item read, 1 audit record | 0.5 item reads per second at peak |
| Lookup concurrency | Concurrency = rate x duration of one invocation = 0.5 x d | Reserved concurrency of 10 covers the peak for any duration d under 20 seconds. |

Even at these upper bounds the sack path averages about 0.3 writes per second and the lookup 0.5 reads per second; five times these figures would still be a few requests per second. On-demand DynamoDB tables therefore follow the load without capacity planning, and no part of this design changes with the number of farms. Prices are left to A23 (PD-152).

## 9. Recovery objectives

The case statement sets three things: no deployment loses sack events, sack events have an RPO of 15 minutes, and the public lookup has an RTO of 4 hours. They are met inside `us-east-1` across Availability Zones (ADR-001, ASM-004). Every time below is a design target, not a measurement; the recovery rehearsal that verifies them is a future test (PD-153).

### 9.1 Write path that cannot lose an acknowledged event

1. The app writes the event to its encrypted outbox before any upload (ASM-079).
2. The `sack-events` function of the application API checks identity, scope and code (section 3.3) and writes the event to `sack_event_log` with a conditional put on `event_id`. A repeated `event_id` returns success without a second write.
3. Only after that write succeeds does the API answer 2xx, and only then does the app remove the event from its outbox. DynamoDB acknowledges a write after it is stored durably across Availability Zones of the Region.
4. DynamoDB Streams carries each new log item to two consumers: the `sack-applier` function (history, sack state, trip and projection, section 3.3) and the `sack-exporter` function (section 9.2). Both use batch bisection, retries and an on-failure destination in an Amazon SQS dead-letter queue under the `records` key; the stream keeps items for 24 hours, so a failing consumer delays processing without losing anything.

### 9.2 RPO of 15 minutes for sack events

| Failure | What protects acknowledged events | Loss target |
|---|---|---|
| A deployment of the API or functions with a defect | Event log writes are a single conditional put that no function role can update or delete (IAM grants `dynamodb:PutItem` with the condition only; no `UpdateItem` or `DeleteItem` on the log). Functions ship as new Lambda versions behind an alias with a canary shift and automatic rollback on error alarms (ASM-157). A defective API answers an error, so the event stays in the outbox. A defective applier leaves events in the stream and the dead-letter queue, and the derived tables are rebuilt from the log. | 0 |
| Loss of an Availability Zone | DynamoDB and Lambda are Regional services spread over Availability Zones. | 0 |
| A derived table corrupted (`sacks`, `sack_history`, `trips`, `buyer_view`) | Rebuilt by replaying `sack_event_log` through the applier, whose fold is deterministic (section 3.3). | 0 |
| The event log itself deleted or corrupted | Deletion protection on the table and `prevent_destroy` in Terraform; point-in-time recovery, restorable to any second of the last 35 days into a new table; and the export copy in Amazon S3. | At most 15 minutes (design target) |
| Loss of the whole Region | Not covered inside the pilot (ASM-004); the export prefix and its declared Azure Blob target are the exit path of ADR-001. | Residual risk accepted by ASM-004 |

The export copy is what turns "point-in-time recovery exists" into a 15-minute bound that does not depend on the table. The `sack-exporter` function writes each stream batch as an append-only JSON Lines object to the sack history export prefix of ADR-001, in the S3 bucket that A09 encrypts with the `objects` key, with versioning and a 5-year lifecycle. Its batching window is 60 seconds and an alarm fires when the stream iterator age exceeds 5 minutes (ASM-157), so under normal operation every acknowledged event has a second copy outside DynamoDB within about a minute, and the alarm leaves 10 minutes to act before the 15-minute objective is at risk. The export is also the open-format, provider-neutral history that ADR-001 prepares for the second cloud.

### 9.3 RTO of 4 hours for the public lookup

The lookup depends only on CloudFront, AWS WAF, the lookup function, the static assets bucket and the `buyer_view` table. It does not depend on the application API, Cognito, the event log or the collection-center links, so a failure of the write path never takes it down: the buyer sees the last projected state.

| Scenario | Recovery | Target |
|---|---|---|
| Defective lookup function release | Move the alias back to the previous version, automatically on alarm or by hand. | Minutes |
| Defective front door or WAF change | Revert the infrastructure change through the pipeline of ADR-003; an incident is the exception to the harvest freeze ("salvo incidente"). | Within the 4-hour budget below |
| `buyer_view` table deleted or corrupted | Restore it by point-in-time recovery into a new table, or rebuild it from `sacks` by the projection job; point the lookup alias to a version configured with the new table. | Within the 4-hour budget below |
| Static assets bucket damaged | Redeploy the versioned assets from the repository; the HTML page of a lookup is rendered by the function, so lookups keep answering meanwhile. | Within the 4-hour budget below |
| Region-wide outage of `us-east-1` | Not covered inside the pilot (ASM-004). | Residual risk |

The 4-hour budget is split into design targets (ASM-157): detection by the lookup availability alarm within 15 minutes, decision and incident declaration within 15 minutes, restore or rebuild within 2 hours, switch and verification within 30 minutes, and 1 hour of margin. The lookup availability SLO of point 6 is defined by A21 on the same path (PD-151).

## 10. Stores

Part E requires a transactional database for sack events and allows a transactional or time-series database for positions. All stores are Amazon DynamoDB tables in on-demand capacity mode, in the workload account of each environment, with point-in-time recovery, deletion protection and a time-to-live attribute `expires_at` set to 5 years after the event date (ASM-098). DynamoDB is already the telemetry store of ADR-003, so the pilot runs one database technology, one backup mechanism and one Terraform pattern; it is serverless, which suits Part E's event-driven functions; and at the loads of section 8 it needs no capacity planning.

| Table | Key | Holds | KMS key | Stream |
|---|---|---|---|---|
| `sack_event_log` | `event_id` | Every accepted sack event as received; immutable. | `records` | Yes, new image, to the applier and the exporter |
| `sack_history` | `sack_id`; sort `recorded_at#event_id` | The ordered history of each sack, input of the fold. | `records` | No |
| `sacks` | `sack_id`; secondary indexes on `sack_code` and on `farm_id` | Current state, flags, version and last applied event. Updated with a condition on the version. | `records` | No |
| `sack_codes` | `sack_code` | Batch, allocation (farm or center reserve), life cycle state, bound `sack_id`. | `records` | No |
| `trips` | `trip_id` | Trip record of section 5. | `records` | No |
| `transport_positions` | `trip_id`; sort `sequence_no`, zero-padded | One item per GNSS fix with the A10 fields. A conditional put on the pair makes a drained duplicate harmless; a different `event_id` on the same pair is flagged. | `telemetry` (A09 section 7.2) | No |
| `buyer_view` | `sack_code` | Only the fields of ASM-096 and the state. | `public_view` (ASM-154) | No |

**Position store (closes PD-110).** Positions are written to the `transport_positions` DynamoDB table above. The trace of a trip is a single query on `trip_id`, ordered by the per-trip `sequence_no`, which also exposes gaps; idempotency comes from the conditional put, as the telemetry table does with node and timestamp. A time-series database was not chosen because it would be a second database technology for 7,200 small items a day, with its own backup, encryption and Terraform module, and its availability to new accounts would have to be verified. Amazon Location Service trackers were not chosen because their position history is not kept for the 5 years of the case statement (to re-check under PD-150).

**Least privilege.** The lookup role can call `dynamodb:GetItem` on `buyer_view` and decrypt with `public_view` only, and write to the audit log group; it holds no grant on the `records`, `personal` or `telemetry` keys, so a coding error in the lookup cannot return a field that the projection does not hold (A09 section 8.4). The applier is the only writer of `buyer_view`. Cost allocation follows A08: positions and trips carry `CostCenter = trips`; the sack store and the lookup are `shared`, allocated to farms by sack reads per farm of origin (ASM-065).

## 11. Compliance with the case statement and the issue

| Criterion | Where it is met |
|---|---|
| Sack registered at the farm with weight and identifier, with reference to 60 kg sacks | Section 2; ASM-150. |
| States in preparation, in transit to center, at center, in transit to port, delivered at port, with accepted or rejected | Section 3.1, and the transition table with event, actor and guard in section 3.2. |
| Events with date, time and responsible for handover, arrival at the center, departure to the port and delivery at the port | Section 4. |
| Peaks of 400 sacks per day per center and 30 reads per minute | Section 8, with the arithmetic. |
| RPO of 15 minutes for sack events and RTO of 4 hours for the public lookup, with the mechanism | Section 9, as design targets (ASM-157, PD-153). |
| "Un despliegue no pierde eventos de saco" | Sections 9.1 and 9.2. |
| Stores for sacks, trips and positions chosen and justified | Section 10; closes PD-110. |
| Public lookup path, front door, caching, enumeration protection, printed code format | Sections 6 and 7; closes PD-051, PD-096 and PD-102. |

## 12. Decision-log entries

| ID | Summary |
|---|---|
| PD-051 | Front door of `public_query`. Closed by section 7.2: CloudFront and WAF in front of a Lambda function URL; no load balancer, no NAT gateway. |
| PD-096 | Codes for a sack that arrives without one while the WAN is down. Closed by section 6.3. |
| PD-102 | Sack code, projection, lookup role and rate limit. Closed by sections 6.1, 7.3 and 10. |
| PD-110 | Position store (A13). Closed by section 10 once the lead copies the entry into the log. |
| ASM-150 | Weight recorded as measured, flagged outside a Federation tolerance; registration names the drying lot. |
| ASM-151 | Rejection at the center is terminal; corrections are compensating events. |
| ASM-152 | Offline ordering, receiver-authoritative transitions, pending and anomaly flags. |
| ASM-153 | Center reserve of at least 400 issued codes. |
| ASM-154 | `public_view` key for the projection. |
| ASM-155 | WAF rate limit of 100 per source IP per 5 minutes; reserved concurrency 10. |
| ASM-156 | Upper bound of 2^22 codes issued in 5 years. |
| ASM-157 | Recovery design targets: exporter window, iterator age alarm, canary release, RTO split. |
| ASM-158 | Federation-owned lookup host name. |
| ASM-159 | A trip is one leg of custody by one transporter. |
| PD-150 | Re-check the AWS features named here. |
| PD-151 | A21 adopts the lookup SLI and the sack and lookup alarms. |
| PD-152 | A23 prices the lookup and the stores. |
| PD-153 | Rehearse the RPO and RTO before `pdn` go-live. |
| PD-154 | The design system maps the seven states to its four sack badges. |
| PD-155 | The Federation confirms the weight tolerance and the rejected-sack rule. |

## 13. References

| Reference | Use |
|---|---|
| [`docs/case/case-statement.pdf`](case/case-statement.pdf) | Part C, Part D, Part E, point 2, pilot figures, capabilities. |
| [`docs/decision-log.md`](decision-log.md) | ASM-001, ASM-004, ASM-007, ASM-036, ASM-057, ASM-058, ASM-065, ASM-069, ASM-076, ASM-077, ASM-079, ASM-094, ASM-096, ASM-098, ASM-124, ASM-127, ASM-129 and the entries of section 12. |
| [ADR-001](adr/adr-001-primary-and-second-cloud.md) | `us-east-1`, recovery inside the Region, sack history export prefix and Azure Blob target. |
| [ADR-003](adr/adr-003-iac-tool-remote-state-and-pattern-module.md) | DynamoDB as the telemetry store, pipeline, the coded pattern module. |
| [`docs/connectivity.md`](connectivity.md) | Sections 5, 7 and 10: registration, custody and position events, outboxes, port delivery. |
| [`docs/security-and-identity.md`](security-and-identity.md) | Sections 3.3, 4.2, 7.2, 8 and 9: scopes, keys, code, buyer view, pseudonyms, retention. |
| [`docs/landing-zone.md`](landing-zone.md) | Section 7.2: audit of a sack read; section 5.3: cost allocation. |
| [`docs/firewall-policy.md`](firewall-policy.md) | CL-PUB-01 to CL-PUB-04, CC-PUB-01, CC-LOG-01, CC-LOG-02. |
| [`docs/collection-center-network.md`](collection-center-network.md) | Section 5: receiving posts and code-sheet printers; section 10: WAN failure. |
| [`docs/addressing-plan.md`](addressing-plan.md) | Section 4.2: the `public_query` tier. |
| `aroma-pacifico-diagrams/data-paths/data-paths.md` (A13) | Transporter position data path, PD-110. |
| [Crockford Base32](https://www.crockford.com/base32.html) | Alphabet, decoding of `I`, `L` and `O`, check symbol modulo 37. |
| [Amazon CloudFront origin access control for Lambda function URLs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-lambda.html) | Origin reachable only through the distribution (PD-150). |
| [AWS WAF rate-based rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html) | Per-source rate limit with a scope-down statement (PD-150). |
| [DynamoDB point-in-time recovery](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/PointInTimeRecovery.html) and [DynamoDB Streams with Lambda](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.Lambda.html) | Restore to a second within 35 days; stream consumers, retention and failure destinations (PD-150). |
