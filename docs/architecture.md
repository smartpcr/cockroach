# CockroachDB Architecture Document

## Table of Contents
1. [Introduction](#introduction)
2. [System Architecture](#system-architecture)
3. [Component Architecture](#component-architecture)
4. [Data Flow Architecture](#data-flow-architecture)
5. [Network Architecture](#network-architecture)
6. [Storage Architecture](#storage-architecture)
7. [Transaction Architecture](#transaction-architecture)
8. [Query Processing Architecture](#query-processing-architecture)
9. [Replication Architecture](#replication-architecture)
10. [Deployment Architecture](#deployment-architecture)
11. [Integration Points](#integration-points)

## Introduction

This document provides a comprehensive overview of CockroachDB's architecture, detailing how components interact to provide a distributed, scalable, and resilient SQL database. CockroachDB's architecture is designed to be cloud-native, horizontally scalable, and operationally simple.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Client Applications                         │
│                    (JDBC, psql, ORMs, Applications)                 │
└─────────────────────┬───────────────────────────┬───────────────────┘
                      │                           │
                      ▼                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        SQL Gateway Layer                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  │
│  │   Node 1    │  │   Node 2    │  │   Node 3    │  │   Node N  │  │
│  │  (Gateway)  │  │  (Gateway)  │  │  (Gateway)  │  │ (Gateway) │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘  │
│         │                 │                 │               │        │
└─────────┼─────────────────┼─────────────────┼───────────────┼────────┘
          │                 │                 │               │
          ▼                 ▼                 ▼               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Distributed SQL Execution                        │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              Query Planning & Optimization                     │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │              Distributed Query Execution                       │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Transaction Layer                               │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │          Distributed Transaction Coordination                   │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │              MVCC & Concurrency Control                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Distribution Layer                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              Range Location & Routing                          │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │              Load Balancing & Rebalancing                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Replication Layer                                │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                   Raft Consensus                               │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │                  Range Management                              │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Storage Layer                                   │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                RocksDB Storage Engine                          │  │
│  ├───────────────────────────────────────────────────────────────┤  │
│  │              MVCC Storage & Garbage Collection                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

1. **SQL Gateway Layer**
   - Client connection handling
   - Protocol translation (PostgreSQL wire protocol)
   - Session management
   - Initial query routing

2. **Distributed SQL Execution**
   - Query parsing and validation
   - Query planning and optimization
   - Distributed execution coordination
   - Result aggregation

3. **Transaction Layer**
   - Transaction lifecycle management
   - Distributed commit protocol
   - Isolation and consistency enforcement
   - Conflict resolution

4. **Distribution Layer**
   - Range discovery and routing
   - Load distribution
   - Automatic rebalancing
   - Locality awareness

5. **Replication Layer**
   - Raft consensus implementation
   - Replica management
   - Lease management
   - Consistency guarantees

6. **Storage Layer**
   - Persistent storage (RocksDB)
   - MVCC implementation
   - Data compression
   - Garbage collection

## Component Architecture

### Node Components

Each CockroachDB node contains the following major components:

```
┌─────────────────────────────────────────────────────────┐
│                    CockroachDB Node                      │
│                                                          │
│  ┌────────────────┐    ┌────────────────────────────┐  │
│  │   SQL Server   │    │    Admin UI Server          │  │
│  └────────┬───────┘    └────────────┬───────────────┘  │
│           │                          │                   │
│  ┌────────▼───────────────────────────────────────────┐ │
│  │              Executor & Planner                     │ │
│  └────────┬───────────────────────────────────────────┘ │
│           │                                              │
│  ┌────────▼───────────────────────────────────────────┐ │
│  │           Transaction Coordinator                   │ │
│  └────────┬───────────────────────────────────────────┘ │
│           │                                              │
│  ┌────────▼───────────────────────────────────────────┐ │
│  │            Distribution Sender                      │ │
│  └────────┬───────────────────────────────────────────┘ │
│           │                                              │
│  ┌────────▼───────────────────────────────────────────┐ │
│  │                Node Engine                          │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────┐ │ │
│  │  │   Store 1   │  │   Store 2    │  │  Store N  │ │ │
│  │  └─────────────┘  └──────────────┘  └───────────┘ │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Gossip Protocol                        │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │           Time Series Database                      │ │
│  └─────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

### Store Architecture

Each store within a node manages multiple ranges:

```
┌─────────────────────────────────────────────┐
│                   Store                      │
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │          Store Metadata                  ││
│  │  - Store ID                              ││
│  │  - Capacity                              ││
│  │  - Node ID                               ││
│  └─────────────────────────────────────────┘│
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │           Range Replicas                 ││
│  │  ┌────────┐ ┌────────┐ ┌────────┐       ││
│  │  │Range 1 │ │Range 2 │ │Range N │       ││
│  │  │Replica │ │Replica │ │Replica │       ││
│  │  └────────┘ └────────┘ └────────┘       ││
│  └─────────────────────────────────────────┘│
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │           Raft Engine                    ││
│  │  - Log entries                           ││
│  │  - Consensus state                       ││
│  └─────────────────────────────────────────┘│
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │          RocksDB Instance                ││
│  │  - SST files                             ││
│  │  - Write-ahead log                       ││
│  │  - Block cache                           ││
│  └─────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

### Range Architecture

Ranges are the fundamental unit of data distribution:

```
┌─────────────────────────────────────────────┐
│                  Range                       │
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │          Range Metadata                  ││
│  │  - Range ID                              ││
│  │  - Start/End Keys                        ││
│  │  - Replica Locations                     ││
│  └─────────────────────────────────────────┘│
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │           Raft Group                     ││
│  │  - Leader Replica                        ││
│  │  - Follower Replicas                     ││
│  │  - Raft Log                              ││
│  └─────────────────────────────────────────┘│
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │          Lease Holder                    ││
│  │  - Serves reads                          ││
│  │  - Coordinates writes                    ││
│  │  - Timestamp cache                       ││
│  └─────────────────────────────────────────┘│
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │            Data                          ││
│  │  - Key-Value pairs                       ││
│  │  - MVCC versions                         ││
│  │  - Intents                               ││
│  └─────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

## Data Flow Architecture

### Read Operation Flow

```
Client Read Request
        │
        ▼
┌─────────────────┐
│   SQL Gateway   │
│  Parse & Plan   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   DistSender    │
│  Route to Range │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Lease Holder   │
│   Serve Read    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Storage      │
│  Retrieve Data  │
└────────┬────────┘
         │
         ▼
    Return Result
```

### Write Operation Flow

```
Client Write Request
        │
        ▼
┌─────────────────┐
│   SQL Gateway   │
│  Parse & Plan   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Transaction   │
│   Coordinator   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   DistSender    │
│  Route to Range │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Lease Holder   │
│  Propose Write  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Raft Group    │
│  Replicate Log  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Apply to State │
│  Machine (All)  │
└────────┬────────┘
         │
         ▼
    Acknowledge
```

### Distributed Query Flow

```
Complex SQL Query
        │
        ▼
┌─────────────────────┐
│   Query Planner     │
│  Create Logical Plan│
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Physical Planner   │
│ Distribute Execution│
└──────────┬──────────┘
           │
     ┌─────┴─────┬─────────┬─────────┐
     ▼           ▼         ▼         ▼
┌─────────┐ ┌─────────┐ ┌────────┐ ┌────────┐
│ Node 1  │ │ Node 2  │ │ Node 3 │ │ Node N │
│TableScan│ │TableScan│ │  Join  │ │  Agg   │
└────┬────┘ └────┬────┘ └───┬────┘ └───┬────┘
     │           │          │          │
     └───────────┴──────────┴──────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │   Gateway    │
                 │ Merge Results│
                 └──────┬──────┘
                        │
                        ▼
                   Return to Client
```

## Network Architecture

### Inter-Node Communication

```
┌──────────────────────────────────────────────────────┐
│                  Node 1                               │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ gRPC Server  │  │ gRPC Client  │  │   Gossip   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
└─────────┼──────────────────┼────────────────┼────────┘
          │                  │                │
          │                  │                │
     ┌────▼──────────────────▼────────────────▼────┐
     │            Network (TCP/TLS)                 │
     └────┬──────────────────┬────────────────┬────┘
          │                  │                │
┌─────────┼──────────────────┼────────────────┼────────┐
│         │                  │                │        │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌────▼──────┐ │
│  │ gRPC Server  │  │ gRPC Client  │  │   Gossip   │ │
│  └──────────────┘  └──────────────┘  └────────────┘ │
│                  Node 2                               │
└───────────────────────────────────────────────────────┘
```

### Communication Protocols

1. **Client-Node Communication**
   - PostgreSQL wire protocol
   - TLS encryption support
   - Connection pooling
   - Load balancing

2. **Node-Node Communication**
   - gRPC for RPC calls
   - Gossip protocol for cluster state
   - Raft protocol for consensus
   - Streaming for bulk data

3. **Service Discovery**
   - Gossip-based node discovery
   - Range location caching
   - Automatic failover routing
   - Health checking

## Storage Architecture

### RocksDB Integration

```
┌─────────────────────────────────────────────┐
│           CockroachDB Storage Layer          │
│                                              │
│  ┌─────────────────────────────────────────┐│
│  │         MVCC Layer                       ││
│  │  - Version management                    ││
│  │  - Timestamp ordering                    ││
│  │  - Intent resolution                     ││
│  └────────────────┬─────────────────────────┘│
│                   │                           │
│  ┌────────────────▼─────────────────────────┐│
│  │         Key Encoding                     ││
│  │  - Table/Index prefixes                  ││
│  │  - Column families                       ││
│  │  - Version suffixes                      ││
│  └────────────────┬─────────────────────────┘│
│                   │                           │
│  ┌────────────────▼─────────────────────────┐│
│  │         RocksDB API                      ││
│  │  - Get/Put/Delete                        ││
│  │  - Iterators                             ││
│  │  - Snapshots                             ││
│  └────────────────┬─────────────────────────┘│
│                   │                           │
│  ┌────────────────▼─────────────────────────┐│
│  │      RocksDB Storage Engine              ││
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ││
│  │  │ MemTable │ │ SST Files│ │   WAL    │ ││
│  │  └──────────┘ └──────────┘ └──────────┘ ││
│  └─────────────────────────────────────────┘│
└──────────────────────────────────────────────┘
```

### Data Organization

1. **Key Structure**
   ```
   /Table/IndexID/PrimaryKey/ColumnID/Timestamp
   ```

2. **Value Structure**
   - Encoded column data
   - MVCC metadata
   - Transaction information

3. **Storage Optimization**
   - Compression (Snappy/Zstd)
   - Bloom filters
   - Block cache
   - Compaction strategies

## Transaction Architecture

### Transaction Lifecycle

```
┌─────────────────────────────────────────────┐
│              Transaction Start               │
│         - Assign transaction ID              │
│         - Select coordinator node            │
│         - Initialize timestamp               │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│           Write Intents Phase                │
│         - Write provisional values           │
│         - Track write locations              │
│         - Check conflicts                    │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│            Commit Phase                      │
│         - Write transaction record           │
│         - Resolve timestamps                 │
│         - Parallel commit optimization       │
└────────────────┬────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────┐
│         Intent Resolution Phase              │
│         - Convert intents to values          │
│         - Cleanup transaction record         │
│         - Asynchronous resolution            │
└──────────────────────────────────────────────┘
```

### Concurrency Control

1. **MVCC Implementation**
   - Multiple versions per key
   - Timestamp-based ordering
   - Garbage collection of old versions

2. **Lock-Free Reads**
   - Read from consistent snapshots
   - No read locks required
   - Timestamp cache for write protection

3. **Write Conflict Resolution**
   - Optimistic concurrency control
   - Transaction priorities
   - Automatic retry logic

## Query Processing Architecture

### Query Planning Pipeline

```
SQL Query
    │
    ▼
┌───────────────┐
│    Parser     │ → Abstract Syntax Tree
└───────┬───────┘
        │
        ▼
┌───────────────┐
│   Analyzer    │ → Semantic Analysis
└───────┬───────┘
        │
        ▼
┌───────────────┐
│   Optimizer   │ → Logical Plan
└───────┬───────┘
        │
        ▼
┌───────────────┐
│Physical Planner│ → Physical Plan
└───────┬───────┘
        │
        ▼
┌───────────────┐
│   Executor    │ → Distributed Execution
└───────────────┘
```

### Distributed SQL Components

1. **Processors**
   - TableReader: Scan ranges
   - Joiner: Distributed joins
   - Aggregator: Group by operations
   - Sorter: Order by operations
   - Distinct: Unique filtering

2. **Flow Scheduling**
   - Node selection
   - Processor placement
   - Data routing
   - Result streaming

3. **Optimization Techniques**
   - Predicate pushdown
   - Join reordering
   - Index selection
   - Statistics-based planning

## Replication Architecture

### Raft Implementation

```
┌─────────────────────────────────────────────┐
│              Raft Group                      │
│                                              │
│  ┌─────────────┐                            │
│  │   Leader    │                            │
│  │  - Accept writes                         │
│  │  - Replicate log                         │
│  │  - Heartbeat followers                   │
│  └──────┬──────┘                            │
│         │                                    │
│         ▼                                    │
│  ┌─────────────┐  ┌─────────────┐          │
│  │  Follower   │  │  Follower   │          │
│  │ - Receive log│  │ - Receive log│         │
│  │ - Apply entries│ │ - Apply entries│      │
│  │ - Vote in elections│ │ - Vote in elections│ │
│  └─────────────┘  └─────────────┘          │
└──────────────────────────────────────────────┘
```

### Replication Flow

1. **Write Proposal**
   - Client sends write to any node
   - Route to range lease holder
   - Lease holder acts as Raft leader

2. **Log Replication**
   - Leader appends to log
   - Replicate to followers
   - Wait for majority acknowledgment

3. **State Machine Application**
   - Apply committed entries
   - Update local storage
   - Maintain consistency

4. **Failure Handling**
   - Automatic leader election
   - Log catch-up for lagging replicas
   - Split-brain prevention

## Deployment Architecture

### Cluster Topology

```
┌─────────────────────────────────────────────────────┐
│                  Region 1 (US-East)                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │   Node 1   │  │   Node 2   │  │   Node 3   │   │
│  │    AZ-1    │  │    AZ-2    │  │    AZ-3    │   │
│  └────────────┘  └────────────┘  └────────────┘   │
└─────────────────────────┬───────────────────────────┘
                          │
                          │ WAN
                          │
┌─────────────────────────┴───────────────────────────┐
│                  Region 2 (US-West)                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐   │
│  │   Node 4   │  │   Node 5   │  │   Node 6   │   │
│  │    AZ-1    │  │    AZ-2    │  │    AZ-3    │   │
│  └────────────┘  └────────────┘  └────────────┘   │
└──────────────────────────────────────────────────────┘
```

### Zone Configuration

1. **Replication Zones**
   - Configure per database/table/index
   - Specify replica count
   - Set location constraints
   - Define lease preferences

2. **Locality Awareness**
   - Rack-aware placement
   - Datacenter distribution
   - Region preferences
   - Cloud provider integration

3. **Multi-Region Strategies**
   - Geo-partitioned replicas
   - Follow-the-workload
   - Regional by row tables
   - Global tables

## Integration Points

### External Interfaces

1. **Client Drivers**
   - PostgreSQL compatibility
   - JDBC/ODBC support
   - Language-specific drivers
   - ORM integration

2. **Monitoring & Observability**
   - Prometheus metrics
   - OpenTelemetry tracing
   - Structured logging
   - Admin UI dashboards

3. **Operational Tools**
   - Backup/Restore
   - Change Data Capture (CDC)
   - Import/Export utilities
   - Schema migration tools

4. **Cloud Integration**
   - Kubernetes operators
   - Cloud storage backends
   - Load balancer integration
   - Service mesh compatibility

### Extension Points

1. **SQL Functions**
   - User-defined functions
   - Stored procedures
   - Custom aggregates
   - Extension framework

2. **Storage Backends**
   - Pluggable storage engines
   - Cloud object storage
   - Encryption providers
   - Compression algorithms

3. **Security Providers**
   - Authentication plugins
   - Authorization systems
   - Audit frameworks
   - Key management systems

## Conclusion

CockroachDB's architecture is designed to provide a scalable, consistent, and highly available distributed SQL database. The layered architecture separates concerns while maintaining efficiency, and the component design ensures that the system can scale horizontally while maintaining strong consistency guarantees. The architecture supports deployment patterns from single-region clusters to globally distributed databases, making it suitable for a wide range of applications and use cases.
