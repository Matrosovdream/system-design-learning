# Example 05 — Vector clocks: detecting concurrent updates in leaderless systems

In a leaderless or multi-leader system, two clients can write to the same key at the same time on different nodes. How do you tell whether two versions are causally related (one came after the other) or **concurrent** (each was made without knowing about the other)?

Vector clocks. They're how Dynamo-class systems decide what to do with conflicting writes.

## The setup

Three nodes: A, B, C. Each maintains a **vector** of counters, one per node. So a vector clock looks like:

```
[A=3, B=7, C=2]
```

A version of a value carries the vector clock at the time it was written.

## The rules

On each write at node N:
```
1. Increment N's own counter.
2. Attach the resulting vector clock to the value.
```

On each read followed by a write:
```
1. Read returns value V with vector clock VC_v.
2. Client modifies V.
3. Client writes back. Node bumps its own counter, attaches new vector clock.
```

On each replica synchronization:
```
Compare incoming vector clock to local one.
If incoming dominates local (every element ≥), incoming is newer; replace.
If local dominates incoming, ignore.
If neither dominates → concurrent writes; conflict.
```

"VC_x dominates VC_y" means for every i: VC_x[i] >= VC_y[i], and at least one is strictly greater.

## Worked example

Initial state: key "color" = "white", VC = [A=0, B=0, C=0].

```
t=1: Client writes "red" at A.
     New VC: [A=1, B=0, C=0]
     Replicated to B and C eventually.

t=2: Client reads from B, gets "red", [A=1, B=0, C=0].
     Modifies to "blue", writes back at B.
     New VC: [A=1, B=1, C=0]
     This dominates [A=1, B=0, C=0] → "blue" is newer than "red".
     Eventually replicates.

t=3: Meanwhile, another client reads from C, gets "red", [A=1, B=0, C=0]
     (because B's write hasn't replicated to C yet).
     Modifies to "green", writes back at C.
     New VC: [A=1, B=0, C=1]

Now B has [A=1, B=1, C=0] = "blue"
        C has [A=1, B=0, C=1] = "green"

When B and C sync:
   B's VC and C's VC: neither dominates the other.
   B has B=1, C has C=1 — they incremented different counters.
   → CONCURRENT WRITES. Conflict.
```

## What do we do with the conflict?

Three common strategies:

### 1. Last-write-wins (LWW)

Pick the value with the larger wall-clock timestamp. **Loses data** (silently). Don't use for important data.

### 2. Application-level reconciliation

Store **both** values (in a "sibling" set). On the next read, the client gets both and must merge them.

```python
def reconcile(siblings):
    # custom logic per data type
    if all siblings are "shopping cart":
        return union of items in each
    elif all siblings are "user profile":
        return... well, ask the user.
```

Used by Amazon's original Dynamo for shopping carts. The reconciliation was "union of items" — if you added an item on one device and a different item on another, you get **both**. Eventually right.

### 3. CRDTs (Conflict-free Replicated Data Types)

Special data structures designed so concurrent updates **always merge correctly** without conflicts.

- **G-Counter** (grow-only counter): each node has its own counter; merge = element-wise max + sum.
- **PN-Counter**: separate increment/decrement counters per node.
- **G-Set** (grow-only set): merge = union.
- **OR-Set** (observed-remove set): handles add+remove correctly.
- **LWW-Element-Set**: per-element timestamp; LWW per element.

CRDTs are the "right" answer for concurrent collaboration. Used in Yjs, Automerge, Redis CRDB, distributed databases.

## Why this matters in real systems

### Cassandra (with vector clocks turned off)

Cassandra by default uses **wall-clock LWW**, not vector clocks. Faster but **loses concurrent updates silently**. You opt into client-side conflict handling for important data, or you use lightweight transactions (`IF NOT EXISTS`) for atomicity.

### DynamoDB

Internally Dynamo (the system) used vector clocks. DynamoDB (the AWS product) uses LWW by default — but with conditional writes (`IF`-style updates) to let you avoid the LWW trap when needed.

### Riak

Native vector clocks, sibling-based reconciliation. The Dynamo paper's ideas in production.

### Real-time collaboration apps

Google Docs, Notion, Figma all use CRDTs (or operational transforms — a related approach) to merge concurrent edits.

## Hybrid Logical Clocks (HLC) vs vector clocks

Vector clocks are expensive: O(N) per write where N is number of nodes. Across a 100-node cluster, that's a lot of metadata per value.

HLCs (used by CockroachDB, MongoDB, YugabyteDB) carry **one** timestamp per value, combining wall-clock with a logical counter. They lose the ability to detect *concurrency* but preserve causal ordering. Cheap and sufficient for most leader-based systems.

You use vector clocks when you have **truly leaderless** writes; HLC when you have a leader per range and want cheap ordering.

## A subtle gotcha: client-side vector clocks

Vector clocks track *replica* writes, not *client* writes. If two clients read the same vector clock and both update, the next replica that sees them detects... a single update from one replica, not the concurrency.

Solution: include the client identity in the vector clock too. Now it's a *client* vector clock — more memory, but accurate.

This is why production systems usually do **conditional writes** ("write only if the value hasn't changed since I read it") on top of vector clocks for important data.

## Architect's takeaway

- **Vector clocks detect concurrent updates** that wall-clock LWW would silently overwrite.
- **They're the right primitive in leaderless / multi-leader systems** where the "real" order is undefined.
- **Production cost is high** — they grow with cluster size.
- **CRDTs** are usually a better fit for known data types (counters, sets, registers) because they merge automatically.
- **Wall-clock LWW is dangerous** for important data. Pick something causally aware.
- **Most apps don't need vector clocks** because most apps use a leader-based system. They become essential when you go leaderless or build collaborative editing.
