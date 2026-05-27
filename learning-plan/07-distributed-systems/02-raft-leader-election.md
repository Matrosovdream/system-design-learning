# Example 02 — Raft: how N nodes agree on a leader and a log

Raft is the consensus algorithm most modern distributed systems use. etcd, Consul, CockroachDB, TiDB, MongoDB, Vault, Kubernetes' control plane — all use Raft (or a close variant). It's worth understanding because **leader election and log replication is the foundation** of HA databases and coordination systems.

## What problem Raft solves

You have N nodes. You want them to agree on **a sequence of commands** even when:
- nodes crash
- network briefly partitions
- messages are reordered or duplicated

The mathematical foundation goes back to Paxos (Leslie Lamport, 1989). Raft (Diego Ongaro, 2013) is functionally similar but **designed to be understandable**.

## The roles

At any moment, every node is in one of three states:
- **Leader** — handles all writes, replicates to followers.
- **Follower** — passive; responds to leader and candidates.
- **Candidate** — trying to become leader (during election).

There is at most **one leader** at a time, per **term**.

## Term: the logical clock of Raft

A monotonically increasing integer. Whenever a new election happens, the term increments. Every node remembers the current term.

If a node sees a message with a term > its own, it updates its term and reverts to follower. This is the **fence** that prevents stale leaders from causing damage.

## How a normal write works (steady state)

```
Client → Leader: "set x=5"

Leader: append (term=4, idx=12, x=5) to my log (uncommitted)
Leader → all followers: AppendEntries (term=4, idx=12, x=5)

Followers: write to log, ack to leader.

Leader: received acks from majority? (in a 3-node cluster: 2 of 3 including self)
   yes → mark idx=12 committed
   apply to state machine (x = 5)
   reply to client: ok

Leader → followers in next heartbeat: "commit up to idx 12"
Followers: apply idx=12 to their state machines.
```

### What does "majority" mean

For N nodes, majority = ⌊N/2⌋ + 1.
- 3 nodes → need 2 ack.
- 5 nodes → need 3 ack.
- 2 nodes → need 2 ack (can't tolerate any failure — why even-sized clusters are bad).

The "majority must agree" rule is what gives Raft its safety properties.

## How leader election works

The leader sends regular **heartbeats** (empty AppendEntries) to followers. If a follower hasn't heard from a leader within its **election timeout** (typically randomized 150-300ms), it suspects the leader is dead.

1. **Follower becomes candidate**: increments term, votes for itself, asks every other node for a vote.
2. **Other nodes vote** if (a) they haven't voted this term, and (b) the candidate's log is at least as up-to-date as theirs.
3. **If candidate gets majority**: becomes leader, starts sending heartbeats.
4. **If split vote** (no majority): wait random timeout, increment term, try again.

The **randomized timeout** prevents repeated split votes — eventually one node times out first and wins.

### Why elections don't accidentally elect bad leaders

A candidate's log must be **at least as up-to-date** as the receiver's, or the receiver refuses to vote. This guarantees the new leader has every committed entry from the previous term.

If you lost a leader after it committed entry 13 but before the follower caught up to 13, the follower won't be elected — only a node that has entry 13 can win.

## A failure scenario walkthrough

3-node cluster: A (leader), B (follower), C (follower). Term 4.

```
t=0: A is leader, all healthy.

t=1: A crashes.

t=200ms: B's election timeout expires. B becomes candidate, term=5,
         votes for self, sends RequestVote to A (dead) and C.

t=210ms: C receives request, hasn't voted this term, B's log is up-to-date,
         votes for B.

t=215ms: B has 2 votes (self + C) = majority. Becomes leader.

t=216ms: B sends heartbeats. C accepts B as leader.

t=2s: A reboots, sees term=5 messages from B, accepts B as leader, becomes follower.
```

The whole election took ~215 ms. Many production systems tune timeouts to ~150 ms for faster failover.

## What if the network partitions?

```
3 nodes: A (leader), B, C.
Partition: A on one side, B + C on the other.
```

- A still thinks it's leader but can't replicate to a majority (only itself).
- A still accepts writes from clients but they never commit (no majority).
- A continues to accept writes — but **they won't be committed**, so they're invisible.

Wait — or does A reject?

In Raft, **A returns success only after a majority commits**. Since A can't reach B/C, writes hang or time out at A. A might still be the "leader" but it's effectively unable to make progress.

Meanwhile:

- B and C don't hear A's heartbeats. B times out, starts an election, term=5, gets C's vote, becomes leader.
- B now serves writes successfully (B and C, that's a majority).

When the partition heals:

- A sees B's heartbeats with term=5 > its own term=4. A steps down to follower.
- Any uncommitted writes A held are discarded.
- A's log syncs with B's.

**The losing side of the partition cannot commit anything.** The winning side has a majority and continues normally. This is how Raft survives partitions while preserving consistency.

## Quorum cluster sizes

| Nodes | Tolerate failure | Notes                                          |
|-------|------------------|------------------------------------------------|
| 1     | 0                | No HA. Single point of failure.                |
| 2     | 0                | Strictly worse than 1; either failure stops    |
| 3     | 1                | **Sweet spot for small clusters**              |
| 4     | 1                | Same as 3 but pays for an extra node          |
| 5     | 2                | **Common production size**, tolerates 2 fails  |
| 7     | 3                | Used for very high availability                |

Even-sized clusters give you no extra fault tolerance over the next-lower odd size. Always pick **odd**.

## Why Raft and not Paxos?

- **Paxos** is the academic original. Provably correct but notoriously hard to implement.
- **Multi-Paxos** extends single-decision Paxos to a log.
- **Raft** decomposes the problem cleanly (leader election + log replication + safety) and adds membership changes that are easier to reason about.

Most implementers find Raft easier to get right. Both algorithms achieve the same correctness; Raft just made consensus accessible.

## Where Raft shows up

- **etcd** (the K/V store backing Kubernetes).
- **Consul** (service discovery).
- **CockroachDB, TiDB, YugabyteDB** (NewSQL): each range/region has its own Raft group.
- **MongoDB replica sets** (3.6+): Raft-inspired.
- **HashiCorp Vault, Nomad**.
- **Apache Kafka KRaft mode** (replacing ZooKeeper).

When you read "the system uses Raft", you can mentally fill in: leader election, log replication, majority quorum, term-numbered safety.

## Architect's takeaway

- **Raft (or equivalent) is the foundation of HA distributed systems.**
- **Odd cluster sizes only.** 3 or 5 is the standard.
- **Network partitions are handled gracefully** — losing side waits, winning side keeps going, no split-brain.
- **There's no free lunch on writes**: every committed write requires majority round-trips. Latency = slowest of majority.
- **Don't implement Raft from scratch.** Use a battle-tested library (etcd's, HashiCorp's, dragonboat in Go).
