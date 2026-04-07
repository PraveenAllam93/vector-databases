# Phase 4: System Design — Distributed Systems Deep Dive

## Reading Order

| # | Topic | File | Key Takeaway |
|---|-------|------|-------------|
| 1 | [Distributed Fundamentals](01-distributed-fundamentals.md) | CAP/PACELC theorems, consistency models, quorums, Raft consensus, clocks, failure modes, split brain |
| 2 | [Replication & Sharding](02-replication-and-sharding.md) | Leader-follower, sync/async/semi-sync replication, replication lag, leader election, hash/consistent/range/tenant sharding, scatter-gather |
| 3 | [Race Conditions & Concurrency](03-race-conditions-and-concurrency.md) | Write-write conflicts, read-after-write, index-data skew, ingestion races, delete-during-search, deadlocks, thundering herd |
| 4 | [Scaling & Load Management](04-scaling-and-load-management.md) | Vertical vs horizontal, capacity planning, load balancing, back-pressure, rate limiting, circuit breakers, health checks, autoscaling, graceful degradation |
| 5 | [Data Integrity & Recovery](05-data-integrity-and-recovery.md) | WAL deep dive, snapshots, RPO/RTO, checksums, disaster recovery scenarios, embedding versioning, audit trails |
| 6 | [Multi-Region & HA](06-multi-region-and-high-availability.md) | Availability math, multi-AZ, active-passive/active-active, write conflicts, DNS routing, cross-region replication lag, chaos engineering, K8s HA config |
| 7 | [Interview Patterns](07-system-design-interview-patterns.md) | 3 sample designs (semantic search, multi-tenant RAG, sub-10ms search), common Q&A, architecture decision records, cheat sheet |

## Interview Quick Reference

**"Explain CAP theorem in the context of vector databases."**
→ During a network partition, you choose availability (serve potentially stale results) or consistency (refuse requests until partition heals). Vector DBs typically choose AP: stale search results are better than no results. Writes lean CP: semi-synchronous replication with majority quorum ensures data safety.

**"How does leader-follower replication work?"**
→ One leader handles all writes, replicates via WAL to followers. Followers serve reads. If leader dies, Raft consensus elects a new leader from the most up-to-date follower. Semi-synchronous: wait for majority ACK before confirming writes (balance of safety and speed).

**"What race conditions can occur in a vector database?"**
→ Write-write conflicts (lost updates), read-after-write inconsistency (stale searches), index-data skew (HNSW graph doesn't match current vectors), ingestion races (duplicate chunks), thundering herd (cache stampede). Solutions: write queues (Kafka), idempotent upserts, immutable segments, content-hash IDs, cache stampede locks.

**"How do you shard a vector database?"**
→ Hash-based (deterministic, even distribution), consistent hashing (minimal redistribution on add/remove), range-based (good for range queries), or tenant-based (custom shard keys for isolation). Search is scatter-gather: query all shards in parallel, merge top-K results.

**"How do you achieve 99.99% availability?"**
→ Multi-AZ deployment (3+ AZs), replication factor >= 2, pod anti-affinity (no two Qdrant pods on same node), PodDisruptionBudget (min 2 available), health checks (liveness + readiness probes), automated failover (Raft leader election), and optionally multi-region active-passive with DNS failover.

**"Walk me through a write path end-to-end."**
→ Client → API (validate, extract tenant) → Kafka (durable queue) → Worker (chunk, embed, deterministic IDs) → Qdrant leader (WAL fsync → replicate to majority → ACK) → mutable segment (immediately searchable via flat scan) → background: seal segment → build HNSW index.

**"How do you prevent data leaks in multi-tenant?"**
→ Defense in depth: custom shard keys per tenant, mandatory tenant_id filter on every query, extract tenant from JWT (never trust client), audit logging, automated cross-tenant access tests.
