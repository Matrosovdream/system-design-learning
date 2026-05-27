# Example 04 — Saga: multi-step workflows that survive partial failures

A **saga** is a long-running business workflow split into a sequence of local transactions. If any step fails, the saga undoes its prior steps with **compensating actions**.

It's the modern alternative to 2PC. Used in virtually every microservices system that has multi-step transactions.

## A canonical example: book a trip

User wants to book: flight + hotel + rental car. Each is a different service with its own DB.

```
Step 1: Reserve flight        →  Flight Service.   compensating: Cancel flight
Step 2: Reserve hotel         →  Hotel Service.    compensating: Cancel hotel
Step 3: Reserve rental car    →  Car Service.      compensating: Cancel car
```

If step 3 fails (no cars available), you've already booked the flight and hotel. The saga **runs compensations** in reverse order: cancel hotel, then cancel flight. The user gets "Sorry, no cars" and **nothing has been charged or reserved** at the end.

## Two ways to coordinate a saga

### 1. Orchestration

A central **orchestrator** (saga controller) drives the workflow. It explicitly calls each step and tracks state.

```
[Orchestrator]
  → Call Flight Service.   ok.
  → Call Hotel Service.    ok.
  → Call Car Service.      fail.

  → Call Hotel.compensate. ok.
  → Call Flight.compensate. ok.
  → Reply to user: failure.
```

**Pros**
- Clear central state — easy to debug.
- Easy to monitor "where is saga #123?"
- Easy to add new steps.

**Cons**
- The orchestrator is a SPOF (mitigated with replication).
- Orchestrator must persist state to survive crashes.

**Used by**: AWS Step Functions, Temporal, Camunda, Netflix Conductor, Uber Cadence.

### 2. Choreography

No central orchestrator. Each service publishes events; others react.

```
Booking Service publishes "trip.requested"
Flight Service reacts → reserves → publishes "flight.reserved"
Hotel Service reacts to "flight.reserved" → reserves → publishes "hotel.reserved"
Car Service reacts to "hotel.reserved" → reserves → publishes "car.reserved"

If Car Service fails → publishes "car.failed"
Hotel Service reacts to "car.failed" → cancels → publishes "hotel.cancelled"
Flight Service reacts to "hotel.cancelled" → cancels → publishes "flight.cancelled"
Booking Service reacts to "flight.cancelled" → marks trip failed
```

**Pros**
- No central coordinator → no SPOF.
- Services are decoupled, can be added/removed independently.

**Cons**
- The flow is implicit — hard to see the whole picture.
- Hard to debug "why didn't the saga progress?"
- Cyclic dependencies emerge if you're not careful.
- Adding a new step means updating multiple services to react to new events.

**Used by**: many event-driven architectures, especially smaller ones.

## Compensating actions: not the same as rollback

A compensating action is a **business-level undo**, not a database rollback.

- **DB rollback**: "pretend this never happened".
- **Compensation**: "make a new transaction that semantically undoes the previous one".

The difference matters because compensation might **not perfectly restore the previous state**:
- Cancelling a flight at noon doesn't undo the email "your flight is booked" that was sent at 11:59.
- Refunding a charge doesn't refund the fee.
- Releasing reserved stock doesn't undo the time it was unavailable to other shoppers.

Compensations are **forward-only**. They make the state correct from now on but don't erase history. Often that's fine — it's what business reality looks like anyway.

## The trickiest case: the partial-failure during compensation

```
Step 1: reserve flight.  ok.
Step 2: reserve hotel.   ok.
Step 3: reserve car.     fail.
Step 4: compensate hotel.   fail.   <-- now what?
```

You can't just give up — flight and hotel are still reserved.

The saga must **retry compensations indefinitely**, possibly with backoff, until they succeed. Compensations must be **idempotent** and **must not fail permanently** (or, if they do, you escalate to human intervention with a clear alert).

In practice: each compensation is logged, retried, and the saga doesn't terminate until everything is either done or compensated.

## Saga state machine: an actual one

```
states:
  pending           (initial)
  flight_reserved
  hotel_reserved
  car_reserved
  completed         (terminal, success)

  car_failed
  cancelling_hotel
  hotel_cancelled
  cancelling_flight
  flight_cancelled
  cancelled         (terminal, failure)

transitions:
  pending           → flight_reserved      on event "flight.reserved"
  flight_reserved   → hotel_reserved       on event "hotel.reserved"
  hotel_reserved    → car_reserved         on event "car.reserved"
  car_reserved      → completed            on event "trip.confirmed"

  flight_reserved   → cancelling_flight    on event "flight.failed"
  hotel_reserved    → cancelling_hotel     on event "hotel.failed"
  car_reserved      → cancelling_car       on event "car.failed"
  cancelling_hotel  → cancelling_flight    on event "hotel.cancelled"
  cancelling_flight → cancelled            on event "flight.cancelled"
```

The saga is a finite-state machine. Each transition is durable (DB row update). On crash, you read the current state and continue.

## When to choose orchestration vs choreography

| Use orchestration when:                    | Use choreography when:                  |
|--------------------------------------------|-----------------------------------------|
| The flow is complex (5+ steps)             | The flow is short (2-3 steps)           |
| You need clear visibility/audit            | Services are highly decoupled           |
| You expect to evolve the flow              | Adding services later is easy           |
| You have a workflow engine available        | Lightweight, no extra infrastructure    |
| Business analysts want to see the workflow | Engineers fully own the flow            |

**Rule of thumb:** start with choreography for small flows; introduce an orchestrator (Temporal is excellent) once you have more than 3-4 steps.

## A real saga: Amazon order flow (simplified)

```
1. Reserve inventory.            comp: Release inventory.
2. Authorize payment.            comp: Release authorization.
3. Charge payment.               comp: Refund.
4. Allocate warehouse.           comp: Deallocate.
5. Ship.                         comp: Issue recall / return.
6. Deliver.                      (no compensation; terminal)
```

Each step is a separate service with its own DB. The orchestrator (or choreography) drives the flow. If anything fails, compensations run backward.

In practice many of these compensations have **time limits** (you can refund within 90 days; you can't compensate a delivered shipment by un-delivering it). The saga model accommodates "non-compensable" terminal states.

## Architect's takeaway

- **Sagas are the right pattern for multi-step workflows across services.**
- **Orchestration is easier to debug**; **choreography is more decoupled**. Pick based on complexity.
- **Compensations are forward-only business undo**, not database rollback. They may leave side effects.
- **Compensations must be idempotent and must not fail permanently** (or you escalate).
- **A saga is a state machine.** Persist state at each step; resume on crash.
- **Use a workflow engine** (Temporal, Camunda, AWS Step Functions) for non-trivial sagas. Don't hand-roll if you can avoid it.
