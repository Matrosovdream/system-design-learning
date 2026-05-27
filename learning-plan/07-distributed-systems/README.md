# Step 07 — Distributed Systems

This is the deep end. Once you have multiple nodes, multiple regions, partial failures, and asynchronous communication, classic single-machine intuition breaks. You need to know how nodes **agree** on things, how they **coordinate** across failures, and how to design business workflows that span services.

## Goals

- Explain why **clocks** cannot be trusted in distributed systems.
- State what **consensus** means and why it's hard.
- Describe how **Raft** elects a leader and replicates entries (mental model).
- Tell when to use **2PC** (and why most modern systems avoid it).
- Build a multi-step workflow as a **saga** with compensating actions.
- Recognize and avoid **split-brain** and **gray-failure** scenarios.

## Key concepts

1. **Time and clocks** — wall clocks lie, monotonic clocks help, logical clocks are needed for ordering.
2. **Logical clocks** — Lamport, vector clocks, hybrid logical clocks (HLC).
3. **Consensus** — N nodes agree on one value, even with crashes.
4. **Raft and Paxos** — the two consensus algorithms that matter in practice.
5. **Leader election** — choosing one node as the coordinator.
6. **Two-phase commit (2PC)** — distributed transactions; blocking and fragile.
7. **Sagas** — long-running workflows with compensating actions instead of rollback.
8. **Split-brain** — two halves of a partitioned cluster both think they're leader.
9. **Quorum** — majority must agree (N/2 + 1) for correctness in many algorithms.
10. **FLP impossibility** — you can't have consensus, no missed messages, and termination, all three.

## Reading

- **DDIA**: Chapter 8 (*The Trouble with Distributed Systems*), Chapter 9 (*Consistency and Consensus*). Foundational.
- **Raft paper**: "In Search of an Understandable Consensus Algorithm" — readable, recommended.
- **Pat Helland's papers** on sagas and idempotency.

## Examples in this folder

- `01-clocks-and-time.md` — why time isn't reliable, what to use instead.
- `02-raft-leader-election.md` — the algorithm explained without math.
- `03-two-phase-commit.md` — what it is, why it's avoided.
- `04-saga-pattern.md` — multi-service workflows that can roll back.
- `05-vector-clocks.md` — how leaderless systems detect concurrent updates.
- `06-split-brain.md` — how cluster partitions create dual leaders and what stops them.

## Self-check

1. Why is `time.Now()` not safe for ordering events from two different servers?
2. In Raft, what happens when the leader's network is briefly partitioned from a majority?
3. Why is 2PC called a "blocking" protocol?
4. You have a saga that books a flight, hotel, and rental car. The car booking fails. What happens?
5. Two nodes in a 3-node Postgres cluster both believe they are the primary. How could this happen and what prevents it?
