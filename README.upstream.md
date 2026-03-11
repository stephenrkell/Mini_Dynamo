# Mini DynamoDB — Leaderless, Tunable Consistency Key–Value Store

A **Mini Dynamo-style distributed key–value store** implemented in Python, inspired by Amazon DynamoDB and the original Dynamo paper. This project demonstrates **production-grade distributed systems concepts** including leaderless replication, quorum-based consistency, vector clocks, and eventual consistency — all runnable **locally without any infrastructure costs**.

---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
- [Core Concepts Implemented](#core-concepts-implemented)
- [Read & Write Flow](#read--write-flow)
- [Tunable Consistency (R, W, N)](#tunable-consistency-r-w-n)
- [Conflict Detection & Resolution](#conflict-detection--resolution)
- [Failure Handling](#failure-handling)
- [Project Structure](#project-structure)
- [Running Locally](#running-locally)
- [API Examples](#api-examples)
- [Failure Simulation](#failure-simulation)
- [Design Trade-offs](#design-trade-offs)
- [Future Improvements](#future-improvements)

---

## Overview

This project implements a **simplified version of Amazon Dynamo**, the foundational paper behind DynamoDB, Cassandra, and Riak. The focus is on demonstrating:

- **Leaderless replication** — any node can handle requests
- **High availability** — system stays operational during failures
- **Eventual consistency** — data converges over time
- **Quorum-based reads and writes** — tunable consistency guarantees

Each node operates as a peer in a symmetric architecture with **no leader** and **no single point of failure**. Any node can accept reads and writes and coordinate with replicas to satisfy quorum requirements.

---

## Key Features

- ✅ **Leaderless replication** — symmetric peer-to-peer architecture
- ✅ **Tunable consistency** — configure `R`, `W`, `N` per request
- ✅ **Quorum-based protocol** — configurable read/write quorums
- ✅ **Consistent hashing** with virtual nodes for load distribution
- ✅ **Vector clocks** for causal versioning and conflict detection
- ✅ **Conflict resolution** — returns sibling versions when concurrent writes occur
- ✅ **Read repair** — automatic replica synchronization
- ✅ **Failure tolerance** — handles node crashes and network partitions
- ✅ **REST API** — simple HTTP interface for all operations
- ✅ **Zero infrastructure** — runs entirely on localhost

---

## System Architecture

```
                          Client
                            |
                            v
        +-------------------+-------------------+
        |                   |                   |
   +----------+        +----------+        +----------+
   |  Node A  |  <-->  |  Node B  |  <-->  |  Node C  |
   |  :5001   |        |  :5002   |        |  :5003   |
   +----------+        +----------+        +----------+
        \                   |                   /
         \                  |                  /
          \                 |                 /
           +----------------+----------------+
                Consistent Hash Ring
                (with Virtual Nodes)
```

**Each node contains:**
- HTTP REST API server
- Local in-memory key–value storage
- Consistent hash ring membership
- Replication and quorum coordination logic
- Vector clock versioning

---

## Core Concepts Implemented

| Concept | Description | Benefit |
|---------|-------------|---------|
| **CAP Theorem** | System favors **AP** (Availability + Partition-tolerance) | Stays operational during network splits |
| **Leaderless Replication** | All nodes are equal; any can coordinate | No single point of failure |
| **Consistent Hashing** | Keys distributed via hash ring | Minimal data movement during scaling |
| **Virtual Nodes** | Each physical node owns multiple ring positions | Better load distribution |
| **Quorum Protocol** | `R + W > N` ensures overlap | Tunable consistency guarantees |
| **Vector Clocks** | Tracks causality per node | Detects concurrent writes |
| **Eventual Consistency** | Replicas converge over time | High write availability |
| **Read Repair** | Stale replicas updated on read | Self-healing system |

---

## Read & Write Flow

### Write Path (PUT)

1. Client sends `PUT` request to any node
2. That node becomes the **coordinator**
3. Coordinator:
   - Computes hash of the key
   - Identifies `N` replica nodes via consistent hashing
   - Sends write requests to all replicas in parallel
4. Waits for `W` successful acknowledgements
5. Returns success (200) or failure (5xx) to client

```
Client → Node A (Coordinator)
           ├─→ Node A (stores locally)
           ├─→ Node B (replica 1)
           └─→ Node C (replica 2)
           
         Waits for W=2 responses
```

**Guarantees:** Write succeeds if at least `W` replicas confirm storage.

![quorom](https://github.com/user-attachments/assets/bac045e3-1be5-4aa0-93b7-55f749c8303b)

---

### Read Path (GET)

1. Client sends `GET` request to any node
2. Coordinator:
   - Identifies `N` replicas for the key
   - Requests values from all replicas in parallel
3. Waits for `R` successful responses
4. Compares vector clocks to detect conflicts
5. Returns the most recent version (or multiple versions if conflict detected)
6. **Asynchronously triggers read repair** if replicas have stale data

```
Client → Node B (Coordinator)
           ├─→ Node A (replica)
           ├─→ Node B (replica)
           └─→ Node C (replica)
           
         Waits for R=2 responses
         Resolves conflicts via vector clocks
         Returns latest version(s)
```

**Guarantees:** Read sees the most recent write if `R + W > N`.

---

## Tunable Consistency (R, W, N)

| Parameter | Meaning | Typical Value |
|-----------|---------|---------------|
| **N** | Replication factor (total replicas) | 3 |
| **W** | Write quorum (replicas that must acknowledge) | 2 |
| **R** | Read quorum (replicas that must respond) | 2 |

### Consistency Rule

```
R + W > N  ⇒  Guarantees read sees latest write (strong consistency)
R + W ≤ N  ⇒  Eventual consistency (stale reads possible)
```

### Common Configurations

| Use Case | N | W | R | Behavior |
|----------|---|---|---|----------|
| Strong consistency | 3 | 2 | 2 | Reads always see latest writes |
| Write-heavy | 3 | 1 | 3 | Fast writes, slower reads |
| Read-heavy | 3 | 3 | 1 | Fast reads, slower writes |
| High availability | 3 | 1 | 1 | Maximum availability, eventual consistency |

**Example:**
```
N = 3, W = 2, R = 2
- Tolerates 1 node failure
- Guarantees consistency overlap
- Balanced read/write performance
```

---

## Conflict Detection & Resolution

### Why Conflicts Occur
- **Concurrent writes** to the same key from different clients
- **Network partitions** causing split-brain scenarios
- **Failed nodes** coming back online with stale data

### Vector Clocks

Each stored value includes a vector clock tracking causality:

```json
{
  "value": "Alice",
  "vector_clock": {
    "node_5001": 3,
    "node_5002": 1,
    "node_5003": 2
  }
}
```

### Conflict Resolution Strategy

1. **Compare vector clocks** between replicas
2. **If one dominates** (all counters ≥), that version wins
3. **If concurrent** (neither dominates), return **both versions as siblings**
4. **Client resolves** conflict (semantic reconciliation)

**Example of concurrent writes:**
```
Version A: {node_5001: 2, node_5002: 1}
Version B: {node_5001: 1, node_5002: 2}

Neither dominates → Both returned as siblings
```

**No silent data loss** — conflicts are always surfaced to the client.

![vclock](https://github.com/user-attachments/assets/70291fd0-d128-434c-968a-26e5d82d4171)

---

## Failure Handling

| Failure Type | System Behavior | Recovery |
|--------------|-----------------|----------|
| **Node crash** | Operations succeed if quorum still reachable | Manual restart + read repair |
| **Network partition** | Both sides accept writes independently | Conflicts resolved via vector clocks |
| **Slow replica** | Timed out and excluded from quorum | Healed via read repair |
| **Replica divergence** | Detected during reads | Read repair synchronizes stale nodes |
| **Coordinator failure** | Client retries with different node | Any node can coordinate |

**Design philosophy:** Prioritize **availability** over strict consistency. The system continues operating through failures and converges eventually.

---

## Project Structure

```
mini_dynamo/
├── node.py              # Node entry point and server initialization
├── config.py            # Cluster configuration (ports, N/R/W defaults)
├── api.py               # REST API endpoints (GET, PUT, DELETE)
├── coordinator.py       # Coordinates read/write operations across replicas
├── hash_ring.py         # Consistent hashing implementation with virtual nodes
├── replication.py       # Replica selection logic
├── quorum.py            # Quorum validation and response aggregation
├── storage.py           # Local in-memory key-value store
├── vector_clock.py      # Vector clock implementation and comparison
├── client_rpc.py        # HTTP client for node-to-node communication
├── read_repair.py       # Background replica synchronization
├── failure.py           # Timeout handling and failure detection
├── metrics.py           # Performance and availability metrics
└── utils.py             # Shared utility functions
```

---

## Running Locally

### Prerequisites

- Python 3.9 or higher
- Standard library only (no external dependencies required)

### Start a 3-Node Cluster

Open three terminal windows and run:

```bash
# Terminal 1
python node.py --port 5001

# Terminal 2
python node.py --port 5002

# Terminal 3
python node.py --port 5003
```

Each node will:
- Start an HTTP server on the specified port
- Join the consistent hash ring
- Begin accepting requests

---

## API Examples

### Write a Value

```bash
curl -X PUT localhost:5001/kv/test123 \
  -H "Content-Type: application/json" \
  -d '{"value":"test","N":3,"W":2}'
```

**Response:**
```json
{
  "status": "success",
}
```

---

### Read a Value

```bash
curl "localhost:5002/kv/test123?R=2"
```

**Response (no conflicts):**
```json
{
  "status": "success",
  "value": "test",
  "vector_clock": {
    "node_5001": 1,
    "node_5002": 1
  }
}
```

**Response (with conflicts):**
```json
{
  "status": "conflict",
  "siblings": [
    {
      "value": "Alice",
      "vector_clock": {"node_5001": 2}
    },
    {
      "value": "Bob",
      "vector_clock": {"node_5002": 2}
    }
  ]
}
```

---

### Delete a Value

```bash
curl -X DELETE http://localhost:5003/kv/user123 \
  -H "Content-Type: application/json" \
  -d '{"W": 2}'
```

---

## Failure Simulation

### Simulate Node Failure

While the cluster is running, kill one node:

```bash
# Find the process ID
ps aux | grep node.py

# Kill node (e.g., port 5003)
kill -9 <PID>
```

### Observe Behavior

**Writes still succeed** (if `W ≤ 2` with 2 remaining nodes):
```bash
curl -X PUT http://localhost:5001/kv/test \
  -d '{"value":"data","N":3,"W":2}'
# ✅ Returns success
```

**Reads still succeed** (if `R ≤ 2`):
```bash
curl -X GET http://localhost:5001/kv/test \
  -d '{"R":2}'
# ✅ Returns value
```

### Recovery

Restart the failed node:
```bash
python node.py --port 5003
```

**Read repair** will automatically update the restarted node when subsequent reads occur.

---

## Design Trade-offs

| Design Choice | Rationale | Trade-off |
|---------------|-----------|-----------|
| **No leader** | Eliminates single point of failure | More complex coordination |
| **Eventual consistency** | Maximizes availability during failures | Temporary stale reads possible |
| **Vector clocks** | Preserves all concurrent versions | Clients must handle conflicts |
| **REST API** | Simple, language-agnostic interface | Higher latency than binary protocols |
| **In-memory storage** | Fast, simple implementation | Data lost on restart |
| **Synchronous replication** | Simpler implementation | Could use async for better performance |

---

## Future Improvements

### Near-term Enhancements
- [ ] **Gossip protocol** for membership and failure detection
- [ ] **Hinted handoff** to buffer writes for temporarily failed nodes
- [ ] **Disk persistence** with write-ahead log
- [ ] **Merkle trees** for efficient anti-entropy

### Long-term Enhancements
- [ ] **gRPC** instead of REST for lower latency
- [ ] **Dynamic cluster membership** (add/remove nodes at runtime)
- [ ] **Monitoring dashboard** with Prometheus/Grafana
- [ ] **Compression** for network bandwidth optimization
- [ ] **Authentication and authorization**

---

## References

- [Amazon Dynamo Paper (2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Designing Data-Intensive Applications](https://dataintensive.net/) by Martin Kleppmann
- [CAP Theorem](https://en.wikipedia.org/wiki/CAP_theorem)
- [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing)

---

## License

MIT License - Feel free to use this project for learning, portfolios, and interviews.

---
