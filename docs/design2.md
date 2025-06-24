# CockroachDB Design Document

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [System Overview](#system-overview)
3. [Core Design Principles](#core-design-principles)
4. [Architecture Overview](#architecture-overview)
5. [Key Design Decisions](#key-design-decisions)
6. [Data Model](#data-model)
7. [Consistency Model](#consistency-model)
8. [Scalability Design](#scalability-design)
9. [Fault Tolerance](#fault-tolerance)
10. [Performance Considerations](#performance-considerations)
11. [Security Design](#security-design)
12. [Future Considerations](#future-considerations)

## Executive Summary

CockroachDB is a distributed SQL database designed to provide horizontal scalability, strong consistency, and survivability. It combines the familiarity of SQL with the scalability and resilience of NoSQL systems. The system is designed to automatically handle machine, rack, and datacenter failures with minimal latency disruption and no manual intervention.

### Key Features
- **Distributed SQL**: Full SQL interface with ACID transactions
- **Horizontal Scalability**: Linear scaling by adding nodes
- **Strong Consistency**: Serializable isolation with distributed consensus
- **Survivability**: Automatic failover and self-healing
- **Geo-Distribution**: Multi-region deployment support
- **PostgreSQL Compatibility**: Wire protocol compatibility

## System Overview

CockroachDB implements a layered architecture that abstracts complexity at each level:

```
┌─────────────────────────────────────┐
│          SQL Layer                  │ ← SQL parsing, planning, execution
├─────────────────────────────────────┤
│    Distributed KV Layer             │ ← Transaction coordination, range addressing
├─────────────────────────────────────┤
│      Replication Layer              │ ← Raft consensus, range management
├─────────────────────────────────────┤
│       Storage Layer                 │ ← RocksDB, MVCC, persistence
└─────────────────────────────────────┘
```

## Core Design Principles

### 1. **No Single Point of Failure**
- Every component is distributed and replicated
- Any node can serve any request
- Automatic failover without manual intervention

### 2. **Consistency First**
- Strong consistency guarantees (Serializable by default)
- No data loss or corruption under any failure scenario
- Clear consistency semantics for developers

### 3. **Minimal Configuration**
- Single binary deployment
- Self-organizing cluster
- Automatic data distribution and rebalancing

### 4. **Incremental Scalability**
- Add nodes to increase capacity and throughput
- Automatic data redistribution
- No downtime for scaling operations

### 5. **SQL Compatibility**
- Standard SQL interface
- PostgreSQL wire protocol
- Familiar tooling and ecosystem

## Architecture Overview

### Layered Architecture

1. **SQL Layer**
   - Parses and optimizes SQL queries
   - Implements distributed SQL execution
   - Manages client connections and sessions
   - Handles transaction coordination

2. **Distributed Key-Value Layer**
   - Maps SQL data to key-value pairs
   - Routes requests to appropriate ranges
   - Implements distributed transactions
   - Manages range metadata

3. **Replication Layer**
   - Implements Raft consensus protocol
   - Manages range replicas
   - Handles lease management
   - Coordinates rebalancing

4. **Storage Layer**
   - Uses RocksDB for local storage
   - Implements MVCC for concurrency
   - Manages on-disk data format
   - Handles garbage collection

### Node Architecture

Each CockroachDB node contains:
- **SQL Gateway**: Accepts client connections
- **Distributed SQL Engine**: Executes queries
- **Transaction Coordinator**: Manages distributed transactions
- **Range Management**: Handles local ranges
- **Storage Engine**: RocksDB instance

## Key Design Decisions

### 1. **Raft for Consensus**
- Chosen for simplicity and proven correctness
- Each range forms a Raft group
- Provides strong consistency guarantees
- Handles leader election and log replication

### 2. **Ranges as Unit of Replication**
- Data split into 64MB ranges by default
- Each range replicated 3x (configurable)
- Ranges can split/merge dynamically
- Enable fine-grained data distribution

### 3. **Hybrid Logical Clocks (HLC)**
- Combines physical and logical time
- Provides causality tracking
- Enables consistent snapshots
- Handles clock skew gracefully

### 4. **MVCC for Concurrency**
- Multiple versions of each key
- Lock-free reads
- Snapshot isolation support
- Efficient garbage collection

### 5. **Distributed SQL Execution**
- Push computation to data
- Parallel query execution
- Distributed joins and aggregations
- Minimize network traffic

## Data Model

### Key-Value Mapping

SQL data is mapped to a sorted key-value store:

```
Table Schema:
CREATE TABLE users (
    id INT PRIMARY KEY,
    name TEXT,
    email TEXT
)

Key-Value Mapping:
/table/users/primary/1/name  → "Alice"
/table/users/primary/1/email → "alice@example.com"
/table/users/primary/2/name  → "Bob"
/table/users/primary/2/email → "bob@example.com"
```

### Index Structure

Secondary indexes are stored as separate key-value pairs:

```
CREATE INDEX idx_email ON users(email)

Index Mapping:
/table/users/idx_email/"alice@example.com" → 1
/table/users/idx_email/"bob@example.com"   → 2
```

### Range Distribution

Data is automatically split into ranges:
- Ranges grow to ~64MB before splitting
- Each range covers a contiguous key span
- Ranges are distributed across nodes
- Metadata tracks range locations

## Consistency Model

### Transaction Guarantees

1. **ACID Compliance**
   - Atomicity: All or nothing execution
   - Consistency: Maintains invariants
   - Isolation: Serializable by default
   - Durability: Committed data persists

2. **Isolation Levels**
   - Serializable (default): Full serializability
   - Snapshot: Snapshot isolation for performance

3. **Distributed Transactions**
   - Two-phase commit protocol
   - Automatic deadlock detection
   - Optimistic concurrency control
   - Transaction record for coordination

### Time and Ordering

- **HLC Timestamps**: Every transaction gets HLC timestamp
- **Causality Preservation**: Related events maintain order
- **Uncertainty Windows**: Handle clock skew between nodes
- **Commit Wait**: Optional for strict linearizability

## Scalability Design

### Horizontal Scaling

1. **Adding Nodes**
   - New nodes join gossip network
   - Receive range replicas automatically
   - Start serving traffic immediately
   - No manual data distribution

2. **Load Balancing**
   - Automatic range rebalancing
   - Consider multiple factors:
     - Replica count per node
     - Storage capacity
     - CPU/memory usage
     - Network topology

3. **Data Distribution**
   - Ranges distributed evenly
   - Hot ranges split automatically
   - Cold ranges merged to save resources
   - Locality-aware placement

### Performance Scaling

- **Linear Scalability**: Throughput increases with nodes
- **Distributed Execution**: Queries parallelized
- **Local Reads**: Served from nearest replica
- **Batch Operations**: Efficient bulk operations

## Fault Tolerance

### Failure Scenarios

1. **Node Failures**
   - Detected via gossip protocol
   - Ranges re-replicated automatically
   - No data loss with proper replication
   - Transparent to applications

2. **Network Partitions**
   - Raft handles split-brain scenarios
   - Majority quorum required
   - Minority partitions go read-only
   - Automatic recovery on heal

3. **Datacenter Failures**
   - Multi-region replication
   - Configurable placement constraints
   - Survive entire DC loss
   - Geo-distributed consensus

### Recovery Mechanisms

- **Automatic Re-replication**: Replace lost replicas
- **Self-Healing**: No manual intervention required
- **Consistent Recovery**: Maintain ACID guarantees
- **Progress Monitoring**: Track recovery status

## Performance Considerations

### Optimization Strategies

1. **Distributed SQL**
   - Push filters to data nodes
   - Distributed joins and aggregations
   - Parallel execution
   - Result streaming

2. **Caching**
   - Range location cache
   - Prepared statement cache
   - Connection pooling
   - Read timestamp cache

3. **Batching**
   - Batch multiple operations
   - Coalesce network requests
   - Pipeline Raft commands
   - Efficient bulk imports

### Performance Trade-offs

- **Consistency vs. Latency**: Serializable adds overhead
- **Replication vs. Throughput**: More replicas increase cost
- **Distribution vs. Locality**: Balance for optimal placement
- **Durability vs. Speed**: Sync replication has latency

## Security Design

### Authentication
- Client certificate authentication
- Password authentication
- SCRAM-SHA-256 support
- Role-based access control

### Encryption
- TLS for all network traffic
- Encryption at rest (optional)
- Certificate rotation support
- Key management integration

### Authorization
- SQL privilege system
- Fine-grained permissions
- Audit logging
- Compliance features

## Future Considerations

### Planned Enhancements

1. **Performance**
   - Query optimization improvements
   - Better statistics collection
   - Advanced caching strategies
   - Hardware acceleration

2. **Features**
   - Enhanced geo-partitioning
   - Improved multi-tenancy
   - Advanced analytics support
   - Streaming CDC improvements

3. **Operations**
   - Better observability
   - Automated tuning
   - Simplified deployment
   - Cloud-native integrations

### Research Areas

- **Consensus Optimization**: Improve Raft performance
- **Clock Synchronization**: Better time handling
- **Machine Learning**: Intelligent data placement
- **Edge Computing**: Support edge deployments

## Conclusion

CockroachDB's design prioritizes correctness, survivability, and ease of use while providing the scalability needed for modern applications. The layered architecture, combined with proven distributed systems techniques, creates a database that can handle the demanding requirements of cloud-native applications while maintaining the familiarity and power of SQL.

The system continues to evolve, but the core design principles remain constant: no single point of failure, strong consistency, minimal configuration, and incremental scalability. These principles guide all design decisions and ensure CockroachDB remains true to its mission of being a scalable, survivable SQL database.
