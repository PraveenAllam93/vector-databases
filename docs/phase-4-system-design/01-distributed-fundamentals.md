# 1. Distributed Systems Fundamentals — The Theory Behind Scaling Vector DBs

---

## ELI5

Imagine a pizza shop. One oven can cook 10 pizzas/hour. When demand hits 100 pizzas/hour, you have two choices:

- **Scale up** (vertical): Buy a BIGGER oven → expensive, has a ceiling
- **Scale out** (horizontal): Open 10 shops, each with its own oven → complex but unlimited

Distributed systems is about running MANY machines that work together as ONE system. It sounds simple, but networks fail, machines crash, and clocks disagree — that's where it gets interesting.

---

## Why Distribute a Vector Database?

### Single-Node Limits

```
Single machine with 64GB RAM:
  - Max vectors (768d, float32): ~20M vectors
  - Max QPS: ~500-1000
  - Single point of failure: machine dies = system dies
  - Maintenance: upgrade requires downtime

After distribution (4 nodes):
  - Max vectors: ~80M vectors
  - Max QPS: ~2000-4000
  - Fault tolerant: 1 node dies, others serve traffic
  - Rolling upgrades: no downtime
```

### What Gets Distributed?

```
┌──────────────────────────────────────────────┐
│           Distributed Vector DB               │
│                                               │
│  ┌─────────────────────────────────────────┐ │
│  │  DATA (vectors + payloads)              │ │
│  │  → Sharding: split across nodes         │ │
│  │  → Replication: copy for redundancy     │ │
│  └─────────────────────────────────────────┘ │
│                                               │
│  ┌─────────────────────────────────────────┐ │
│  │  COMPUTE (search, indexing)             │ │
│  │  → Search: parallel across shards       │ │
│  │  → Indexing: can run on dedicated nodes  │ │
│  └─────────────────────────────────────────┘ │
│                                               │
│  ┌─────────────────────────────────────────┐ │
│  │  COORDINATION (who owns what)           │ │
│  │  → Consensus: agree on cluster state    │ │
│  │  → Routing: send queries to right shard │ │
│  └─────────────────────────────────────────┘ │
└──────────────────────────────────────────────┘
```

---

## CAP Theorem

The CAP theorem states that a distributed system can provide at most TWO of three guarantees simultaneously:

```
        Consistency
           /\
          /  \
         /    \
        / PICK \
       /  TWO   \
      /          \
     /____________\
Availability    Partition
                Tolerance
```

### The Three Properties

**Consistency (C)**: Every read returns the most recent write. All nodes see the same data at the same time.

```
Client writes vector V1 to Node A
Client reads from Node B immediately after
  → Consistent: Node B returns V1 (even if it means waiting for replication)
  → Inconsistent: Node B returns old data or nothing
```

**Availability (A)**: Every request gets a response (success or failure). No requests hang forever.

```
Client sends search query
  → Available: always gets results (maybe stale)
  → Unavailable: request times out because system is syncing
```

**Partition Tolerance (P)**: The system continues operating even when network communication between nodes is lost.

```
Node A ←✗ broken network ✗→ Node B
  → Partition tolerant: both nodes keep serving
  → Not partition tolerant: system shuts down
```

### Why Can't You Have All Three?

When a network partition happens (P is forced on you), you must choose:

```
Scenario: Network splits Node A from Node B

Option 1 — Choose CP (Consistency + Partition Tolerance):
  Node A: "I can't confirm Node B has the latest data"
  Node A: "I'll reject writes/reads until partition heals"
  → System is unavailable but consistent

Option 2 — Choose AP (Availability + Partition Tolerance):
  Node A: "I'll keep serving with my local data"
  Node B: "I'll keep serving with my local data"
  → Both serve, but might return different results (inconsistent)
```

### Where Do Vector DBs Land?

| System | CAP Choice | Why |
|--------|-----------|-----|
| Qdrant | CP-leaning | Prioritizes correct search results over availability during partitions |
| Milvus | AP-leaning | Prioritizes availability; eventual consistency for writes |
| Elasticsearch | AP-leaning | Keeps serving; may return stale results |
| PostgreSQL+pgvector | CP | Full ACID transactions |

**For vector search, AP is usually acceptable**: returning slightly stale search results is better than returning nothing. But writes need more care (CP-leaning).

---

## Consistency Models

### Strong Consistency

Every read sees the latest write. Period.

```
Timeline:
  T1: Client A writes vector V1 to Node 1
  T2: Client B reads from Node 2
  T3: Client B sees V1 ✓ (guaranteed)

Cost: High latency (must wait for replication confirmation)
```

### Eventual Consistency

If no new writes happen, all nodes will EVENTUALLY have the same data. No guarantee about WHEN.

```
Timeline:
  T1: Client A writes vector V1 to Node 1
  T2: Client B reads from Node 2 → sees old data (V0)
  T3: Replication happens in background
  T4: Client B reads from Node 2 → sees V1 ✓

Gap: T2 to T4 could be milliseconds or seconds
```

### Read-Your-Writes Consistency

A client always sees its own writes, but might not see other clients' writes immediately.

```
Timeline:
  T1: Client A writes V1 to Node 1
  T2: Client A reads → sees V1 ✓ (routed to same node)
  T3: Client B reads from Node 2 → might see old data (acceptable)
```

**This is the most practical model for vector databases**. The ingestion service always sees its own writes; query services might see slightly stale data.

### Causal Consistency

If operation B depends on operation A, then everyone sees A before B. Unrelated operations can be seen in any order.

```
Client A: writes V1 (cause)
Client A: writes V2 that references V1 (effect)
Client B: if B sees V2, it must also see V1 (causal order preserved)
Client B: might see V1 without V2 yet (that's fine)
```

### Consistency Model Comparison

```
Strongest ───────────────────────────────────────────── Weakest
Linearizable → Sequential → Causal → Read-your-writes → Eventual

More latency ←──────────────────────────────────────── Less latency
Less throughput ←────────────────────────────────────── More throughput
```

---

## The PACELC Theorem (CAP Extended)

CAP only describes behavior DURING a partition. PACELC adds: even when there's NO partition, there's still a tradeoff.

```
If Partition:
  Choose Availability or Consistency (A or C)
Else (normal operation):
  Choose Latency or Consistency (L or C)

PACELC = Partition → A/C, Else → L/C
```

| System | During Partition | Normal Operation |
|--------|-----------------|------------------|
| Qdrant | Consistency (CP) | Low Latency (EL) |
| Milvus | Availability (AP) | Low Latency (EL) |
| DynamoDB | Availability (AP) | Low Latency (EL) |
| PostgreSQL | Consistency (CP) | Consistency (EC) |

**Vector DB sweet spot**: PA/EL — Available during partitions, Low latency normally. Accept eventual consistency for reads.

---

## Quorum Systems

Quorums define how many nodes must agree before an operation succeeds.

### The Formula

```
N = total replicas
W = write quorum (number of nodes that must acknowledge a write)
R = read quorum (number of nodes that must respond to a read)

Rule: W + R > N → guarantees strong consistency
```

### Examples

```
N=3, W=2, R=2: Strong consistency
  Write: 2 of 3 nodes confirm → success
  Read: 2 of 3 nodes respond → at least 1 has latest data
  Overlap guaranteed: W + R = 4 > 3

N=3, W=1, R=1: Eventual consistency (fast!)
  Write: 1 node confirms → success (async replicate to others)
  Read: 1 node responds → might be stale
  No overlap: W + R = 2 ≤ 3

N=3, W=3, R=1: Strong writes, fast reads
  Write: ALL nodes must confirm (slow, but data is everywhere)
  Read: ANY node has latest data (fast)
  Overlap: W + R = 4 > 3
```

### Quorum in Vector DB Context

```
Typical production setup (N=3):
  Write path: W=2 (majority) → confirms after 2 replicas acknowledge
  Read path:  R=1 (fast) → reads from nearest replica

  Tradeoff: Writes are slower but safe. Reads are fast but might
  be slightly stale if the one replica queried hasn't replicated yet.
```

### Sloppy Quorum (Qdrant/Dynamo Style)

When a node is down, write to a temporary substitute node. When the original comes back, transfer the data.

```
Normal: Write to [Node1, Node2, Node3], need 2 ACKs
Node2 is down: Write to [Node1, Node3, Node4(temp)], need 2 ACKs
Node2 recovers: Node4 transfers data to Node2 (hinted handoff)
```

---

## Clocks and Ordering

### The Problem with Wall Clocks

```
Node A clock: 10:00:00.000
Node B clock: 10:00:00.150  ← 150ms ahead (NTP drift)

Event 1 on Node A at 10:00:00.100: write("article_1", vector_v1)
Event 2 on Node B at 10:00:00.050: write("article_1", vector_v2)

By wall clock: Event 2 (10:00:00.050) happened BEFORE Event 1 (10:00:00.100)
In reality: Event 1 happened first, Event 2 second

If we use wall clocks to resolve conflicts → wrong version wins!
```

### Logical Clocks (Lamport Clocks)

Don't trust wall clocks. Use counters that increment on every event:

```
Node A: counter=0
Node A: write V1, counter=1
Node A: sends message to B with counter=1
Node B: receives, sets counter = max(own, received) + 1 = 2
Node B: write V2, counter=3

Now we know: V1 (counter=1) happened before V2 (counter=3)
```

### Vector Clocks

Track counters per node — captures concurrent events:

```
Node A: {A:1, B:0} → write V1
Node B: {A:0, B:1} → write V2 (concurrent! neither knows about the other)

{A:1, B:0} vs {A:0, B:1} → neither dominates → CONFLICT
Must resolve: last-writer-wins, merge, or ask the application
```

### How Qdrant Handles This

Qdrant uses a simpler approach:
- **Raft consensus** for metadata/cluster state (strong ordering)
- **WAL sequence numbers** for data operations (per-shard ordering)
- **Upsert semantics**: last write wins (application must handle conflicts)

---

## Failure Modes

### Types of Failures

```
┌─────────────────────────────────────────────────┐
│  Failure Spectrum                                 │
│                                                   │
│  Crash failure:      Node stops, doesn't respond  │
│  Omission failure:   Node drops messages silently  │
│  Timing failure:     Node responds too slowly      │
│  Byzantine failure:  Node sends WRONG data         │
│                      (corrupted, malicious)        │
│                                                   │
│  Simple ─────────────────────────────── Complex   │
│  (crash)                              (byzantine) │
└─────────────────────────────────────────────────┘
```

### Failure Detection

How do you know a node is down?

```
Heartbeat pattern:
  Every node sends "I'm alive" every 1 second
  If no heartbeat for 5 seconds → suspected failure
  If no heartbeat for 30 seconds → declared dead

Problems:
  - Network is slow, not dead → false positive
  - Node is overloaded, heartbeat delayed → false positive
  - Solution: phi-accrual failure detector (probabilistic)
```

### Qdrant Failure Handling

```
Scenario: 3-node cluster, Node 2 dies

Before:
  Shard 1: [Node1 (leader), Node2 (follower), Node3 (follower)]
  Shard 2: [Node2 (leader), Node3 (follower), Node1 (follower)]

After Node 2 dies:
  Shard 1: [Node1 (leader), Node3 (follower)] ← still works, lost 1 replica
  Shard 2: [Node3 (new leader), Node1 (follower)] ← leader election!

Recovery:
  Node 2 comes back → joins as follower → syncs from leaders
  Eventually: back to 3 replicas per shard
```

---

## Consensus Algorithms

### Why Consensus?

When you have multiple nodes, they need to AGREE on things:
- Who is the leader for each shard?
- What is the current cluster membership?
- What was the last committed operation?

### Raft (Used by Qdrant)

Raft is the consensus algorithm Qdrant uses for cluster coordination.

```
Three roles:
  Leader:    Handles all writes, sends to followers
  Follower:  Receives from leader, applies locally
  Candidate: Trying to become leader (during election)

Normal operation:
  Client → Leader → replicate to Followers → majority ACK → commit

Leader election:
  1. Follower notices leader is missing (timeout)
  2. Becomes Candidate, votes for itself
  3. Requests votes from other nodes
  4. Wins if gets majority → becomes Leader
  5. Notifies all nodes of new leadership
```

**Raft guarantees**:
- At most ONE leader per term
- Committed entries are never lost
- All nodes eventually agree on the same log

```
Term 1: Node A is leader
  A: [op1, op2, op3]  (committed to majority)
  B: [op1, op2, op3]
  C: [op1, op2]        (slightly behind, but catches up)

Node A crashes → Term 2 election:
  B wins election (has all committed entries)
  B becomes leader, C catches up from B
  No data loss ✓
```

### Split Brain

The worst failure mode: two nodes both think they're the leader.

```
Network partition:
  [Node A (leader)] ←✗ broken ✗→ [Node B, Node C]

  Node A: "I'm still leader!" (but can't reach majority)
  Node B: "Leader is gone, I'll hold election" → wins (majority with C)

  Two leaders! If clients write to both → data divergence

Raft prevents this:
  Node A can't commit: needs majority (2 of 3), only has itself
  Node A's writes fail → clients retry on Node B
  When partition heals: Node A steps down, syncs from Node B
```

---

## Network Partitions in Practice

### Types of Partitions

```
Complete partition: Node A can't reach B or C
  [A] ←✗→ [B, C]

Partial partition: A can reach B, A can reach C, but B can't reach C
  [A] ↔ [B]
  [A] ↔ [C]
  [B] ←✗→ [C]

Asymmetric partition: A can send to B, but B can't send to A
  [A] → [B]
  [A] ←✗ [B]
```

### What Happens During a Partition

```
Query path (reads):
  - Queries to available nodes still work
  - Results might be stale if the partition isolated the shard leader
  - Some vector DBs return partial results (data from available shards only)

Write path:
  - Writes to the majority side succeed
  - Writes to the minority side fail (can't achieve quorum)
  - After partition heals, minority side syncs up

Index operations:
  - Index rebuilds on isolated nodes may produce inconsistent indexes
  - After partition heals, indexes re-sync or rebuild
```

---

## Summary

| Concept | Key Point | Vector DB Relevance |
|---------|-----------|-------------------|
| CAP Theorem | Pick 2 of 3: Consistency, Availability, Partition Tolerance | Vector DBs typically choose AP (available, eventually consistent) |
| PACELC | Even without partitions: latency vs consistency tradeoff | Vector search favors low latency → accept eventual consistency |
| Consistency Models | Strong → Eventual spectrum | Read-your-writes is the sweet spot for most vector DB workloads |
| Quorums | W+R>N for strong consistency | W=majority for safe writes, R=1 for fast reads |
| Logical Clocks | Don't trust wall clocks for ordering | WAL sequence numbers order operations within a shard |
| Raft Consensus | Leader election + log replication | Qdrant uses Raft for cluster coordination |
| Split Brain | Two leaders → data divergence | Raft prevents this via majority requirement |
| Failure Detection | Heartbeats + timeouts | False positives cause unnecessary leader elections |

---

## Resources

- [Designing Data-Intensive Applications (Martin Kleppmann)](https://dataintensive.net/) — THE book for distributed systems
- [Raft Visualization](https://thesecretlivesofdata.com/raft/) — interactive Raft explainer
- [Jepsen: Distributed Systems Safety Research](https://jepsen.io/) — formal testing of distributed DBs
- [CAP Theorem Explained (ByteByteGo)](https://www.youtube.com/watch?v=BHqjEjzAicY) — visual overview
- [Qdrant Distributed Deployment](https://qdrant.tech/documentation/guides/distributed_deployment/) — official docs
