# 2. Replication & Sharding — Scaling Data Across Machines

---

## ELI5

**Replication** = Make COPIES of data on multiple machines. If one machine dies, others have the data. Like backing up your photos to both your phone and your laptop.

**Sharding** = SPLIT data across multiple machines. Each machine holds a piece. Like splitting a phone book — A-M goes to Server 1, N-Z goes to Server 2. Now you can search twice as fast (in parallel).

You almost always use BOTH together.

---

## Replication

### Why Replicate?

```
Without replication:
  Node 1: [all vectors]
  Node 1 dies → ALL DATA LOST, ALL QUERIES FAIL

With replication (factor=3):
  Node 1: [all vectors] (leader)
  Node 2: [all vectors] (follower)
  Node 3: [all vectors] (follower)
  Node 1 dies → Node 2 becomes leader, zero downtime ✓
```

| Benefit | How |
|---------|-----|
| Fault tolerance | Data survives node failures |
| Read scaling | Multiple nodes can serve read queries in parallel |
| Geo-locality | Replicas in different regions for lower latency |
| Zero-downtime upgrades | Rolling restart — upgrade one node at a time |

---

### Leader-Follower (Primary-Replica) Replication

The most common pattern. One leader handles writes; followers receive copies.

```
                    ┌──────────┐
         writes ──▶ │  Leader   │
                    │  (Node 1) │
                    └─────┬─────┘
                          │ replication stream
                    ┌─────┼─────┐
                    ▼     ▼     ▼
              ┌────────┐ ┌────────┐ ┌────────┐
   reads ──▶  │Follower│ │Follower│ │Follower│
              │(Node 2)│ │(Node 3)│ │(Node 4)│
              └────────┘ └────────┘ └────────┘
```

**Write path:**
```
1. Client sends upsert to Leader
2. Leader writes to its WAL
3. Leader sends WAL entry to all Followers
4. Followers apply the entry
5. Leader waits for quorum ACK (e.g., 2 of 3 followers)
6. Leader confirms write to client
```

**Read path:**
```
1. Client sends query
2. Router picks a replica (round-robin, nearest, least-loaded)
3. Replica searches its local index
4. Returns results

Note: Reading from a follower might return slightly stale data
if replication hasn't caught up yet.
```

### Synchronous vs Asynchronous Replication

**Synchronous (strong consistency):**
```
Client → Leader → [write to WAL] → [send to ALL followers] → [wait for ALL ACK] → [confirm to client]

Timeline:
  T0: Leader receives write
  T1: Leader writes locally (1ms)
  T2: Leader sends to followers
  T3: Follower 1 ACK (5ms network + 1ms write)
  T4: Follower 2 ACK (8ms network + 1ms write)
  T5: Follower 3 ACK (12ms network + 1ms write) ← must wait for slowest!
  T6: Leader confirms to client

Total latency: ~15ms (bottleneck: slowest follower)

Pros: No data loss, strong consistency
Cons: Write latency = slowest replica, one slow node slows everything
```

**Asynchronous (eventual consistency):**
```
Client → Leader → [write to WAL] → [confirm to client] → [async send to followers]

Timeline:
  T0: Leader receives write
  T1: Leader writes locally (1ms)
  T2: Leader confirms to client ← immediate!
  T3-T10: Followers receive and apply (in background)

Total latency: ~2ms

Pros: Fast writes, leader not blocked by followers
Cons: Data loss if leader dies before replication, stale reads from followers
```

**Semi-synchronous (practical compromise):**
```
Client → Leader → [write to WAL] → [send to followers] → [wait for MAJORITY ACK] → [confirm]

Wait for 2 of 3 followers (not all):
  - If fastest two respond in 6ms → confirm in 8ms
  - Third follower catches up asynchronously
  - Can tolerate 1 slow/dead follower without blocking writes

This is what Qdrant uses in production.
```

### Replication Lag

The time between a write on the leader and the write appearing on a follower.

```
Leader:   [op1, op2, op3, op4, op5, op6, op7]
                                              ↑ latest
Follower: [op1, op2, op3, op4, op5]
                                    ↑ lagging by 2 operations

Replication lag = 2 operations = ~10ms (depends on throughput)
```

**Problems caused by replication lag:**

```
Scenario 1: "Phantom reads"
  T1: Client writes new vector V1 to Leader
  T2: Client queries Follower → V1 not there yet (lag)
  T3: Client queries Follower → V1 appears
  T4: Client queries Follower → V1 gone?! (different follower, also lagging)

Scenario 2: "Stale search results"
  T1: Client updates vector V1 with new embedding
  T2: Client searches → gets results based on OLD embedding (follower hasn't updated)

Mitigation: Read-your-writes consistency
  Route reads from the same client back to the same node (sticky sessions)
  Or route to leader for writes + immediate reads
```

**Monitoring replication lag:**

```
Key metrics to watch:
  - Replication lag in seconds/operations
  - WAL position difference (leader vs follower)
  - Follower status (active, catching up, dead)

Alert thresholds:
  Warning: lag > 100ms
  Critical: lag > 1s
  Page: lag > 10s or follower unreachable
```

### Leader Election

When the leader dies, a follower must become the new leader.

```
Healthy state:
  Node1 (Leader) ↔ Node2 (Follower) ↔ Node3 (Follower)

Node1 crashes:
  Node2 & Node3: "Leader heartbeat missing for 5 seconds"
  Node2: "I'll start an election! I vote for myself."
  Node3: "Node2 has all committed data. I vote for Node2."
  Node2: "I have majority (2/3). I'm the new leader!"

  Node2 (New Leader) ↔ Node3 (Follower)
  Node1 (Dead)

Node1 recovers:
  Node1: "I was leader, but I missed some time..."
  Node1 sees Node2 is leader with higher term
  Node1: becomes Follower, syncs from Node2

  Node2 (Leader) ↔ Node3 (Follower) ↔ Node1 (Follower, catching up)
```

**Election safety:**
```
Requirement: A candidate must have ALL committed entries to become leader

Node A: [op1, op2, op3, op4] ← has everything
Node B: [op1, op2, op3]      ← missing op4
Node C: [op1, op2]           ← missing op3, op4

If leader dies:
  Only Node A can become leader (has all committed entries)
  Node B: votes for A (A is more up-to-date)
  Node C: votes for A
  → No data loss guaranteed
```

---

### Multi-Leader Replication

Multiple nodes accept writes. Used for multi-region deployments.

```
US Region:             EU Region:
┌──────────┐           ┌──────────┐
│ Leader A  │ ←──────→  │ Leader B  │
│ (writes)  │ bidirectional│ (writes) │
└──────────┘ replication └──────────┘

US users write to Leader A (low latency)
EU users write to Leader B (low latency)
Leaders sync with each other asynchronously
```

**The big problem: Write Conflicts**

```
Concurrent writes to the same vector:
  T1: US user updates vector V1 to [0.1, 0.2, ...] on Leader A
  T1: EU user updates vector V1 to [0.5, 0.6, ...] on Leader B

Both succeed locally. When they sync → CONFLICT!

Resolution strategies:
  1. Last-Writer-Wins (LWW): Use timestamps, latest write wins
     Problem: Clock skew → wrong version might win

  2. Version Vectors: Track causality, detect true conflicts
     Problem: Application must resolve conflicts

  3. CRDT: Conflict-free data types that auto-merge
     Problem: Not all operations are commutative

  4. Avoid conflicts: Route same key to same leader
     Problem: Reduces multi-leader benefits

Most vector DBs use LWW (acceptable because vector updates are typically
full overwrites, not partial modifications).
```

### Leaderless Replication (Dynamo-style)

No leader at all. Any node can accept reads and writes.

```
Write: Send to ALL replicas, succeed if W replicas ACK
Read:  Send to ALL replicas, succeed if R replicas respond

Client writes V1:
  → Node 1: ACK ✓
  → Node 2: ACK ✓ (W=2 met, success!)
  → Node 3: temporarily down

Client reads V1:
  → Node 1: returns V1 ✓
  → Node 2: returns V1 ✓ (R=2 met, consistent!)
  → Node 3: returns old data (still catching up)

Read-repair: Client notices Node 3 is stale → sends correct version to Node 3
```

---

## Sharding (Partitioning)

### Why Shard?

```
Replication alone:
  3 nodes, 100M vectors each = 100M total (each node holds everything)
  Read throughput: 3x (parallel reads)
  Write throughput: 1x (all writes go to leader)
  Storage: 3x redundancy

Sharding + Replication:
  3 shards × 3 replicas = 9 nodes
  Each shard: 33M vectors
  Total capacity: 100M vectors
  Read throughput: 9x
  Write throughput: 3x (parallel writes to different shards)
  Storage: 3x redundancy per shard
```

### Sharding Strategies

#### 1. Hash-Based Sharding

```python
# Deterministic: same ID always goes to same shard
def get_shard(vector_id: str, num_shards: int) -> int:
    return hash(vector_id) % num_shards

get_shard("doc_123", 4)  # → shard 2
get_shard("doc_456", 4)  # → shard 0
get_shard("doc_789", 4)  # → shard 3
```

```
Shard 0: IDs where hash(id) % 4 == 0
Shard 1: IDs where hash(id) % 4 == 1
Shard 2: IDs where hash(id) % 4 == 2
Shard 3: IDs where hash(id) % 4 == 3
```

**Pros:** Even distribution, simple routing.
**Cons:** Resharding is painful (adding a shard redistributes everything).

#### 2. Consistent Hashing

Solves the resharding problem. Uses a virtual ring:

```
The Hash Ring:
          0
         / \
        /   \
      Node1  Node4
      /       \
    /           \
  Node2        Node3
    \           /
     \         /
      \_______/

Each data point is hashed to a position on the ring.
It belongs to the NEXT node clockwise.

Adding a new node (Node5):
  Only data between Node4 and Node5 needs to move
  Other nodes unaffected!

Traditional hash: adding 1 shard → ~75% of data moves
Consistent hash:  adding 1 shard → ~25% of data moves
```

**Virtual nodes** improve balance:
```
Without virtual nodes:
  Node1: 15% of ring (uneven!)
  Node2: 40% of ring
  Node3: 45% of ring

With virtual nodes (100 per physical node):
  Node1: ~33% of ring ✓
  Node2: ~33% of ring ✓
  Node3: ~34% of ring ✓
```

#### 3. Range-Based Sharding

Split by a key range:

```
Shard 0: user_id 0 - 999,999
Shard 1: user_id 1,000,000 - 1,999,999
Shard 2: user_id 2,000,000 - 2,999,999
```

**Pros:** Range queries are efficient (one shard for a user's data).
**Cons:** Hot spots — if user IDs are sequential, newest shard gets all writes.

#### 4. Tenant-Based Sharding

Each tenant gets their own shard (or set of shards):

```
Shard 0: Tenant "acme_corp" → all their vectors
Shard 1: Tenant "globex_inc" → all their vectors
Shard 2: Tenant "initech" → all their vectors

Pros:
  - Perfect isolation (no cross-tenant leaks)
  - Each tenant can be sized independently
  - Easy to migrate a single tenant

Cons:
  - Small tenants waste shard capacity
  - Large tenants may need multiple shards
```

### Sharding in Qdrant

Qdrant supports automatic and custom sharding:

```python
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, ShardingMethod

client = QdrantClient(host="localhost", port=6333)

# Automatic sharding (Qdrant distributes across nodes)
client.create_collection(
    collection_name="documents",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    shard_number=6,         # 6 shards across the cluster
    replication_factor=2,   # 2 replicas per shard
)

# Custom sharding (you control which shard)
client.create_collection(
    collection_name="tenant_docs",
    vectors_config=VectorParams(size=768, distance=Distance.COSINE),
    sharding_method=ShardingMethod.CUSTOM,
)

# Create shard per tenant
client.create_shard_key("tenant_docs", shard_key="acme_corp")
client.create_shard_key("tenant_docs", shard_key="globex_inc")

# Insert to specific shard
client.upsert(
    collection_name="tenant_docs",
    points=[...],
    shard_key_selector="acme_corp",
)

# Search specific shard only
client.query_points(
    collection_name="tenant_docs",
    query=[...],
    shard_key_selector="acme_corp",
)
```

### How Distributed Search Works

```
Client: "Find top 10 similar vectors"
        │
        ▼
    ┌──────────┐
    │  Router   │
    └──┬───┬───┬┘
       │   │   │   scatter (parallel)
       ▼   ▼   ▼
    ┌────┐┌────┐┌────┐
    │Sh 1││Sh 2││Sh 3│  Each shard: find local top 10
    └──┬─┘└──┬─┘└──┬─┘
       │     │     │    gather
       ▼     ▼     ▼
    ┌──────────────────┐
    │  Merge & Re-rank  │  Global top 10 from 30 candidates
    └──────────────────┘
        │
        ▼
    Top 10 results
```

**Scatter-Gather overhead:**
```
Single shard: 5ms search
3 shards (parallel): max(5ms, 5ms, 5ms) + 1ms merge = 6ms
6 shards (parallel): max(5ms, 5ms, ...) + 2ms merge = 7ms

Overhead scales with number of shards, but search time stays constant
(parallel execution). Main cost is the merge step.
```

### Shard Rebalancing

When adding or removing nodes:

```
Before (3 nodes, 6 shards, replication=2):
  Node 1: [S1-leader, S2-follower, S3-leader, S4-follower]
  Node 2: [S1-follower, S2-leader, S5-leader, S6-follower]
  Node 3: [S3-follower, S4-leader, S5-follower, S6-leader]

Add Node 4:
  1. Qdrant identifies shards to move
  2. Copies shard data to Node 4 (background)
  3. Once synced, switches traffic
  4. Removes shard from old node

After:
  Node 1: [S1-leader, S2-follower, S3-leader]
  Node 2: [S2-leader, S5-leader, S6-follower]
  Node 3: [S3-follower, S4-leader, S5-follower]
  Node 4: [S1-follower, S4-follower, S6-leader]  ← new, balanced
```

---

## Replication + Sharding Combined

```
Collection: 100M vectors, 3 shards, 2 replicas each

Shard 1 (~33M vectors):
  ├── Node A (leader)
  └── Node D (follower)

Shard 2 (~33M vectors):
  ├── Node B (leader)
  └── Node E (follower)

Shard 3 (~33M vectors):
  ├── Node C (leader)
  └── Node F (follower)

Write to vector in Shard 2:
  → Routes to Node B (Shard 2 leader)
  → Node B writes to WAL, replicates to Node E
  → Node E ACKs
  → Write confirmed

Search query:
  → Scatter to all 3 shard leaders (A, B, C) in parallel
  → Each returns local top-K
  → Router merges into global top-K
  → Returns to client

Node B crashes:
  → Shard 2: Node E becomes leader (election)
  → Writes to Shard 2 now go to Node E
  → Reads from Shard 2 now go to Node E
  → Other shards unaffected
  → System continues with zero downtime
```

---

## Summary

| Concept | Key Point | When to Use |
|---------|-----------|-------------|
| Leader-Follower | One writer, many readers | Default replication pattern |
| Sync replication | Wait for all replicas | When data loss is unacceptable |
| Async replication | Confirm before replication | When low latency matters more |
| Semi-sync | Wait for majority | Best balance (Qdrant default) |
| Multi-leader | Multiple writers in different regions | Multi-region deployments |
| Replication lag | Time for follower to catch up | Monitor and alert on this |
| Hash sharding | hash(id) % N | Even distribution, most common |
| Consistent hashing | Virtual ring | When resharding frequently |
| Range sharding | Key ranges | When range queries matter |
| Tenant sharding | Per-customer shards | Multi-tenant isolation |
| Scatter-gather | Parallel search across shards | All distributed searches |

---

## Resources

- [Designing Data-Intensive Applications, Ch 5-6](https://dataintensive.net/) — replication and partitioning
- [Qdrant Distributed Deployment](https://qdrant.tech/documentation/guides/distributed_deployment/) — official guide
- [Qdrant Sharding](https://qdrant.tech/documentation/guides/distributed_deployment/#sharding) — custom sharding docs
- [Consistent Hashing (ByteByteGo)](https://www.youtube.com/watch?v=UF9Iqmg94tk) — visual explainer
- [Dynamo Paper (Amazon)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) — foundational distributed DB paper
