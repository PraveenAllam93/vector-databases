# 6. Multi-Region & High Availability — Surviving Anything

---

## ELI5

If your pizza shop is in one city and an earthquake hits, all customers lose access. But if you have shops in three cities, one can go down and the others keep serving. That's multi-region.

High availability (HA) means your system stays up even when things break. The question isn't IF things break, but WHEN — and whether your users notice.

---

## Availability Targets

```
Availability %    Downtime/year    Downtime/month    What it means
─────────────────────────────────────────────────────────────────
99%               3.65 days        7.3 hours         "Two nines"
99.9%             8.76 hours       43.8 minutes      "Three nines"
99.95%            4.38 hours       21.9 minutes      "Three and a half nines"
99.99%            52.6 minutes     4.38 minutes      "Four nines"
99.999%           5.26 minutes     26.3 seconds      "Five nines"
```

**Realistic targets for vector search systems:**

```
Non-critical (internal tools):     99.9% (8.76 hours/year downtime)
Production (customer-facing):      99.95% (4.38 hours/year)
Mission-critical (search, ads):    99.99% (52.6 minutes/year)
```

### How to Calculate System Availability

```
Sequential dependencies (all must work):
  System = Component_A × Component_B × Component_C

  API (99.99%) × Qdrant (99.95%) × LLM API (99.9%)
  = 0.9999 × 0.9995 × 0.999
  = 0.9984 = 99.84%

  Even with individually reliable components, the system is LESS reliable
  than its weakest link.

Parallel redundancy (any one works):
  System = 1 - (1 - Component_A) × (1 - Component_B)

  Two Qdrant nodes, each 99.95%:
  = 1 - (1 - 0.9995)²
  = 1 - 0.00000025
  = 0.99999975 = 99.99997%

  Redundancy dramatically improves availability.
```

---

## Single-Region HA Architecture

```
┌────────────────────────────────────────────────────────┐
│  Region: us-east-1                                      │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │  AZ-1a    │  │  AZ-1b    │  │  AZ-1c    │             │
│  │           │  │           │  │           │             │
│  │ API Pod 1 │  │ API Pod 2 │  │ API Pod 3 │             │
│  │ Qdrant N1 │  │ Qdrant N2 │  │ Qdrant N3 │             │
│  │ Redis R1  │  │ Redis R2  │  │           │             │
│  │           │  │           │  │           │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│        │              │              │                  │
│        └──────────────┼──────────────┘                  │
│                       │                                 │
│              ┌────────▼────────┐                        │
│              │  Load Balancer   │                        │
│              │  (ALB/NLB)      │                        │
│              └─────────────────┘                        │
│                                                         │
│  Sharding: 3 shards, each replicated across 3 AZs      │
│  Shard 1: [N1-leader, N2-follower, N3-follower]        │
│  Shard 2: [N2-leader, N3-follower, N1-follower]        │
│  Shard 3: [N3-leader, N1-follower, N2-follower]        │
└────────────────────────────────────────────────────────┘
```

**Failure scenarios handled:**

```
Single node failure:
  → Qdrant: automatic failover (leader election)
  → API: load balancer routes to healthy pods
  → Redis: sentinel promotes replica
  → Impact: none (zero downtime)

Single AZ failure:
  → 2 of 3 AZs still running
  → Qdrant: majority quorum intact (2 of 3 nodes)
  → Reduced capacity but fully operational
  → Impact: possible brief latency spike during failover
```

---

## Multi-Region Architecture

### Active-Passive (Warm Standby)

```
┌─────────────────────┐          ┌─────────────────────┐
│  US-EAST (Active)    │          │  EU-WEST (Passive)   │
│                      │  async   │                      │
│  ┌────┐ ┌────┐      │ ──────▶  │  ┌────┐ ┌────┐      │
│  │ N1 │ │ N2 │ │N3│ │ replica  │  │ N4 │ │ N5 │ │N6│ │
│  └────┘ └────┘      │  tion    │  └────┘ └────┘      │
│                      │          │                      │
│  Handles ALL traffic │          │  Standby (read-only) │
└─────────────────────┘          └─────────────────────┘

Normal: All traffic → US-EAST
US-EAST fails: DNS failover → EU-WEST takes over

Pros: Simple, cost-effective (passive region is idle)
Cons: Failover takes minutes (DNS propagation), EU region may have stale data
```

### Active-Active (Both Serve Traffic)

```
┌─────────────────────┐          ┌─────────────────────┐
│  US-EAST (Active)    │          │  EU-WEST (Active)    │
│                      │ bi-dir   │                      │
│  ┌────┐ ┌────┐      │◀──────▶ │  ┌────┐ ┌────┐      │
│  │ N1 │ │ N2 │ │N3│ │ replic  │  │ N4 │ │ N5 │ │N6│ │
│  └────┘ └────┘      │  ation  │  └────┘ └────┘      │
│                      │          │                      │
│  US users → here     │          │  EU users → here     │
└─────────────────────┘          └─────────────────────┘

Pros: Low latency for all users, no failover needed
Cons: Write conflicts, higher complexity, 2x infrastructure cost
```

**Conflict resolution for active-active:**

```
Write conflict: User A updates doc in US, User B updates same doc in EU

Strategy 1: Region-pinned writes
  Each document "belongs" to one region based on tenant
  Reads from any region (replicated)
  Writes only to the owning region → no conflicts

Strategy 2: Last-Writer-Wins (LWW)
  Attach timestamp to every write
  When regions sync, latest timestamp wins
  Simple but can lose updates

Strategy 3: Conflict-free operations
  Vector upserts are naturally idempotent
  If both regions write the same vector → last one wins, both are valid
  Payload updates: merge (not overwrite) when possible
```

### Active-Active Implementation

```python
class MultiRegionClient:
    """Client that writes to local region, reads from nearest."""

    def __init__(self, local_region: str, regions: dict):
        self.local = regions[local_region]
        self.all_regions = regions

    def upsert(self, collection_name, points):
        """Write to local region (async replication handles the rest)."""
        return self.local.upsert(collection_name=collection_name, points=points)

    def search(self, collection_name, query, **kwargs):
        """Search local region (lowest latency)."""
        return self.local.query_points(
            collection_name=collection_name,
            query=query,
            **kwargs,
        )

    def search_global(self, collection_name, query, **kwargs):
        """Search ALL regions and merge (highest recall, highest latency)."""
        import concurrent.futures

        results = []
        with concurrent.futures.ThreadPoolExecutor() as executor:
            futures = {
                executor.submit(
                    client.query_points,
                    collection_name=collection_name,
                    query=query,
                    **kwargs,
                ): region
                for region, client in self.all_regions.items()
            }
            for future in concurrent.futures.as_completed(futures):
                try:
                    result = future.result()
                    results.extend(result.points)
                except Exception:
                    pass  # region unavailable, skip

        # Deduplicate and re-rank
        seen = set()
        unique = []
        for point in sorted(results, key=lambda p: p.score, reverse=True):
            if point.id not in seen:
                seen.add(point.id)
                unique.append(point)

        return unique[:kwargs.get("limit", 10)]
```

---

## DNS and Traffic Routing

### Geo-Based Routing

```
User in New York → DNS resolves to us-east-1 IP
User in London   → DNS resolves to eu-west-1 IP
User in Tokyo    → DNS resolves to ap-northeast-1 IP

Implementation: Route53 (AWS), Cloud DNS (GCP), Cloudflare
```

### Failover Routing

```
Normal:
  example.com → us-east-1 (primary, health check: passing)

Primary fails:
  Health check: failing (3 consecutive failures)
  DNS TTL expires (60 seconds)
  example.com → eu-west-1 (secondary)

Primary recovers:
  Health check: passing again
  Traffic routes back to primary (or stays on secondary — configurable)
```

### Weighted Routing

```
example.com:
  70% → us-east-1 (more capacity)
  20% → eu-west-1
  10% → ap-northeast-1

Use for:
  - Gradual migration between regions
  - Canary deployments
  - Load distribution
```

---

## Data Synchronization Between Regions

### Async Replication Stream

```
US-EAST (Primary)                    EU-WEST (Secondary)
┌─────────────┐                      ┌─────────────┐
│  WAL         │ ──── Replication ──▶ │  Apply WAL   │
│  [op1,op2,  │      Stream          │  entries     │
│   op3,op4]  │      (Kafka/direct)  │  [op1,op2]  │
└─────────────┘                      └─────────────┘
                                         ↑
                                     Lag: 2 ops
```

**Replication lag between regions:**

```
Same AZ:        < 1ms
Same region:    1-5ms
Cross-region:   50-200ms (depends on distance)
Cross-continent: 100-500ms

US-East to EU-West: ~100ms
US-East to AP-Northeast: ~200ms
```

### Handling Replication Lag

```
Scenario: User writes in US, reads in EU immediately

Option 1: Sticky routing
  After a write, route that user to the write region for 5 seconds
  After 5 seconds, replication is likely complete → route normally

Option 2: Version-aware reads
  Write returns a version token
  Read includes the version token
  EU region: "I need version >= 42, but I'm at version 40"
  EU region waits until replication catches up (or redirects to US)

Option 3: Accept staleness
  For search results, slightly stale data is usually acceptable
  Document uploaded 200ms ago not appearing in search? Acceptable.
```

---

## Chaos Engineering

Test your HA setup by intentionally breaking things.

### What to Test

```
1. Node failure:     Kill a Qdrant pod randomly
2. Network partition: Block traffic between nodes
3. Disk full:        Fill storage volume
4. High latency:     Add 500ms delay to Qdrant
5. CPU saturation:   stress-ng on Qdrant nodes
6. Memory pressure:  Reduce memory limits
7. Clock skew:       Change system time on a node
8. Region failure:   Disable an entire region
```

### Tools

```
Chaos Monkey (Netflix):   Random instance termination
Litmus Chaos:             Kubernetes-native chaos
Toxiproxy:                Network fault injection
tc (traffic control):     Add latency/packet loss

Example with tc (add 200ms latency):
  tc qdisc add dev eth0 root netem delay 200ms

Example pod kill (Kubernetes):
  kubectl delete pod qdrant-0 --grace-period=0
```

### Chaos Testing Script

```python
"""
Simple chaos test: verify system survives node failure.
"""
import subprocess
import time
import requests

QDRANT_PODS = ["qdrant-0", "qdrant-1", "qdrant-2"]
API_URL = "http://localhost:8000"

def test_node_failure():
    # Verify system is healthy
    resp = requests.get(f"{API_URL}/health/ready")
    assert resp.status_code == 200

    # Kill one Qdrant node
    pod = QDRANT_PODS[1]
    print(f"Killing {pod}...")
    subprocess.run(["kubectl", "delete", "pod", pod, "--grace-period=0"])

    # Wait for failover
    time.sleep(10)

    # Verify system still works
    resp = requests.post(f"{API_URL}/ask", json={"question": "test query"})
    assert resp.status_code == 200, f"System down after killing {pod}!"
    print(f"System survived killing {pod} ✓")

    # Wait for pod to restart
    time.sleep(30)

    # Verify full recovery
    resp = requests.get(f"{API_URL}/health/ready")
    assert resp.status_code == 200
    print("Full recovery ✓")
```

---

## Kubernetes HA Configuration

### Qdrant StatefulSet (Production)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
spec:
  serviceName: qdrant-headless
  replicas: 3
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      # Anti-affinity: never schedule 2 Qdrant pods on same node
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: [qdrant]
              topologyKey: kubernetes.io/hostname

      # Spread across availability zones
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: qdrant

      containers:
        - name: qdrant
          image: qdrant/qdrant:latest
          ports:
            - containerPort: 6333
            - containerPort: 6334
          resources:
            requests:
              memory: "4Gi"
              cpu: "2"
            limits:
              memory: "8Gi"
              cpu: "4"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 6333
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /readyz
              port: 6333
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          volumeMounts:
            - name: qdrant-storage
              mountPath: /qdrant/storage

  volumeClaimTemplates:
    - metadata:
        name: qdrant-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3  # AWS EBS gp3
        resources:
          requests:
            storage: 100Gi
```

### Pod Disruption Budget

```yaml
# Prevent Kubernetes from killing too many pods at once
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: qdrant-pdb
spec:
  minAvailable: 2  # Always keep at least 2 of 3 pods running
  selector:
    matchLabels:
      app: qdrant
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| Availability math | Sequential = multiply, Parallel = dramatically better |
| Multi-AZ | Survive AZ failure with pods spread across zones |
| Active-passive | Simple, one region serves, other is standby |
| Active-active | Both regions serve, lowest latency, highest complexity |
| Write conflicts | Region-pinned writes or LWW for active-active |
| Cross-region lag | 100-500ms; handle with sticky routing or version-aware reads |
| DNS routing | Geo-based, failover, weighted routing |
| Chaos engineering | Test failures BEFORE they happen in production |
| Pod anti-affinity | Never run 2 Qdrant pods on the same Kubernetes node |
| PDB | Prevent Kubernetes from evicting too many pods at once |

---

## Resources

- [AWS Multi-Region Architecture](https://aws.amazon.com/solutions/implementations/multi-region-application-architecture/) — reference architecture
- [Qdrant Distributed Mode](https://qdrant.tech/documentation/guides/distributed_deployment/) — cluster setup
- [Netflix Chaos Engineering](https://netflixtechblog.com/the-netflix-simian-army-16e57fbab116) — pioneering chaos testing
- [Kubernetes Topology Spread](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) — AZ spreading
- [Google SRE: Handling Overload](https://sre.google/sre-book/handling-overload/) — production reliability
