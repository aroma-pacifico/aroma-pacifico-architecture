# ADR-004: Drone capability orchestration

| Field | Value |
|---|---|
| Status | Proposed |
| Date | 2026-11-11 |
| Owner | SR1 |
| Roadmap issue | aroma-pacifico/aroma-pacifico-roadmap#45 |
| Supersedes | none |
| Superseded by | none |

## Context

The drone service has five capabilities that the case statement keeps separate: delimit the plot, confirm the sensor alert, fumigate, verify after fumigating and estimate the harvest. Something has to decide which capability runs when a mission changes state, and has to keep that decision observable and recoverable. In the feedback session the reference architecture showed this as an explicit orchestrator that coordinates each drone capability, with a database and events handled through queues. The pilot design already contains every part of that picture, but no diagram or document names the role, so a reader can conclude that the coordination is missing.

The forces that shape the decision come from the case statement and from earlier ADRs. Part E makes inference event-driven ("La función se invoca con un disparador y termina al persistir el resultado. [...] No hay cómputo persistente en espera de objetos."). A failed invocation must leave the source object and the case state unchanged ("Si la invocación falla, el objeto de origen permanece y el estado del caso no cambia"). A fumigation stays open until its verification object arrives, and its authorization is a human identity that no trigger executes (ADR-002). Between the flight and the verification the pilot may be without a data link, and the verification is a separate mission flown in a later review campaign, so the time between two capabilities of the same case is hours to weeks, not seconds (ASM-025, ASM-026, ASM-028). Cost stays a design driver (`docs/pilot-costs.md`).

The design that answers these forces is documented in [`docs/drone-service.md`](../drone-service.md) (mission and order state machines, section 5) and [`docs/inference-functions.md`](../inference-functions.md) (triggers, queues, retries and dead-letter queues, sections 3 and 6). This ADR does not add a component to it. It names the role that the existing components play together and records why that role is not played by a central workflow engine.

## Decision

The pilot orchestrates the drone capabilities with events, and names the group that does it the Drone capability orchestrator (event-driven). It is made of four parts that already exist in the design:

1. The mission state machine in Amazon DynamoDB is the source of truth. A mission and an order change state only by a guarded transition written to the mission table, and a transition whose guard fails is refused and leaves the state unchanged ([drone service](../drone-service.md#5-state-machines)).
2. The change of a mission into `record_persisted` reaches an Amazon EventBridge Pipe, which puts a `mission_record_persisted` event on the custom event bus of the drone service (ASM-149, ASM-161).
3. The routing rules of that bus, one per mission type, decide which capability runs: `confirmation`, `verification` or `estimation` ([inference functions](../inference-functions.md#3-the-four-triggers)).
4. One Amazon SQS queue per function buffers each routed event and gives it retries with backoff and its own dead-letter queue, so a failure leaves the event, the object and the state where they were ([inference functions](../inference-functions.md#6-failure-handling)).

The bus with its rules is the router, the queues are the buffers, and the state machine in the database is the memory. No component keeps a workflow instance open while it waits: the next capability starts when the next event arrives, and a human step such as the fumigation authorization is a state of the order that only an agronomist identity can change. The functions never authorize a fumigation and never decide the next capability themselves; they write their result and the platform applies the transition (ASM-163).

## Alternatives considered

| Alternative | Advantages | Disadvantages | Reason rejected |
|---|---|---|---|
| Event-driven orchestration: custom bus, one rule per mission type, one queue and dead-letter queue per function, mission state machine in DynamoDB (chosen) | Reuses the components Part E and ADR-003 already require. Each function is independent, idempotent and scaled and retried on its own. No component runs or bills while a case waits. A new consumer, such as observability, subscribes to the bus without touching the functions. | The end-to-end path of one case is not visible in one place; it is reconstructed from the mission and order records and from the events. Ordering between capabilities rests on the state guards, not on a declared sequence. | Chosen. |
| Central workflow engine, for example AWS Step Functions, with one execution per case or per order | One graph shows the whole path of a case, with built-in retries, error branches and an execution history. | It would bill for each state transition of every case, including the waiting states. It would couple every capability to one definition that must change whenever a mission type or a guard changes. Its long wait between the fumigation flight and the verification mission (a review campaign, possibly with no link in between) and its human step (the agronomist authorization) would keep an execution open for days or weeks, or force a callback design, while the state machine in the database already holds that state. It would be a second place that owns the case state next to the mission table. | Not chosen for the pilot. |
| Direct invocation of each function by the reception function | Fewest components. | The reception function would need to know every capability and would lose the event if a call failed. No buffer, no retry and no dead-letter queue per function, which Part E asks the design to define. | Contradicts the failure handling of Part E. |

## When a workflow engine would make sense

The pilot does not need one, and this decision is not permanent. A workflow engine becomes worth reconsidering if one of these conditions appears: a capability needs a sequence of several automated steps with compensation when a later step fails; the number of automated steps between two human or flight steps grows enough that the path of a case can no longer be read from the state guards; or an operator needs a single execution view of a case that the mission and order records cannot give. In that case the engine would coordinate only the automated steps of one capability, would be started by the same event, and would not replace the mission state machine as the source of truth. Reconsidering it needs a new ADR that supersedes this one (PD-390).

## Consequences

| Dimension | Impact |
|---|---|
| Network | None. Every part runs inside AWS managed services in `us-east-1`; no new endpoint or route is added. |
| Security | The authorization of a fumigation stays a human identity: no rule, queue or function can move an order to `authorized` ([ADR-002](adr-002-drone-flight-mode-and-governance.md)). Each function keeps its own execution role and its own queue, so a failure or a compromise of one capability does not reach another. |
| IaC | None. No resource is added: the bus, the rules, the queues and the dead-letter queues are those of the B10 function pattern and the drone stack. The change is the name of the role in the diagrams and documents. |
| Costs | No component is added, so the cost formula of ASM-169 is unchanged. The central engine was not priced for the pilot; its price per state transition would have to be read from the AWS pricing page with its date, as ASM-169 requires for the other unit prices (PD-390). |
| Observability | The single view of a case is built from the mission and order records and from the alarms on each dead-letter queue (`docs/observability.md`), not from an execution history. |

## Compliance with the case statement

| Case statement passage | How this decision complies |
|---|---|
| "La función se invoca con un disparador y termina al persistir el resultado. [...] No hay cómputo persistente en espera de objetos." (The function is invoked by a trigger and ends when the result is persisted. No persistent compute waits for objects.) | The orchestrator holds no running workflow while a case waits; each function starts on an event and ends when it persists its result. |
| "Si la invocación falla, el objeto de origen permanece y el estado del caso no cambia. El diseño define reintentos, cola de fallos y dónde se observa el error." (If the invocation fails, the source object remains and the case state does not change. The design defines retries, a failure queue and where the error is observed.) | One queue with retries and one dead-letter queue per function, and the source of truth in the mission table that a failed function does not change. |
| "La autorización de fumigación es una identidad humana. Ningún disparador la ejecuta." (Fumigation authorization is a human identity. No trigger executes it.) | No rule, queue or function has a transition toward `authorized`; the orchestrator routes only inference events. |
| "El estado de la fumigación no pasa a cerrado hasta persistir ese registro" (The fumigation state does not become closed until that record is persisted.) | The state guard in the mission table, not the router, closes an order, and only after the verification record is persisted. |

## References

- Decision log: ADR-004 (index row), ASM-390, ASM-391, PD-390; ASM-025, ASM-026, ASM-028, ASM-149, ASM-161, ASM-163 and ASM-169 as cited above.
- Roadmap issue: aroma-pacifico/aroma-pacifico-roadmap#45 (G05).
- [`docs/drone-service.md`](../drone-service.md) section 5 (state machines).
- [`docs/inference-functions.md`](../inference-functions.md) sections 3 and 6 (triggers, retries, dead-letter queues).
- [ADR-002](adr-002-drone-flight-mode-and-governance.md) and [ADR-003](adr-003-iac-tool-remote-state-and-pattern-module.md).
- Diagrams: `complete/architecture.drawio` and `flows/inference-functions.drawio` in `aroma-pacifico-diagrams`, group label "Drone capability orchestrator (event-driven) · ADR-004".
