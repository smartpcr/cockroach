# CockroachDB Transaction Implementation Design

## Overview

CockroachDB implements distributed ACID transactions using a sophisticated architecture that combines Multi-Version Concurrency Control (MVCC), Hybrid Logical Clocks (HLC), and an optimistic concurrency control protocol. This document provides a comprehensive overview of how transactions are implemented, including isolation levels, MVCC mechanics, and the complete transaction lifecycle.

## Architecture Overview

The transaction system is organized into several layers:

1. **SQL Layer** - Manages SQL semantics and session state
2. **Transaction Coordinator Layer** - Coordinates distributed transaction execution
3. **Distribution Layer** - Routes requests to appropriate ranges
4. **Storage Layer** - Implements MVCC and handles actual data storage

## Core Components

### Transaction Structure

The main transaction representation is the `kv.Txn` struct (pkg/kv/txn.go), which provides a thread-safe interface for concurrent goroutines. Key components include:

- **Transaction ID** - Unique identifier (UUID)
- **Timestamp** - HLC timestamp for MVCC ordering
- **Priority** - Used for conflict resolution
- **Isolation Level** - Serializable (default), Snapshot, or Read Committed
- **Read/Write Sets** - Track accessed keys for conflict detection

### Transaction Coordinator

The `TxnCoordSender` (pkg/kv/kvclient/kvcoord/txn_coord_sender.go) is the central coordinator that manages:

- Transaction state machine (pending → staging → committed/aborted)
- Heartbeating to prevent abandonment
- Lock span accumulation
- Retry logic and error handling

#### Interceptor Stack

The coordinator uses a stack of interceptors that add functionality:

1. **MetricRecorder** - Records transaction metrics
2. **SpanRefresher** - Enables timestamp advancement without full restart
3. **Committer** - Handles commit protocol including parallel commits
4. **Pipeliner** - Allows asynchronous intent writes
5. **WriteBuffer** - Buffers writes for performance
6. **SeqNumAllocator** - Manages operation sequence numbers
7. **Heartbeater** - Sends periodic heartbeats

## MVCC Implementation

### Key Concepts

CockroachDB uses MVCC to enable concurrent transactions without locking for reads. Each key can have multiple versions at different timestamps.

### Key Encoding

MVCC keys are encoded as:
```
[user_key][sentinel][timestamp_wall][timestamp_logical][timestamp_length]
```

Keys are stored in decreasing timestamp order, allowing efficient retrieval of the most recent version.

### Value Types

1. **Regular Values** - Committed data with timestamps
2. **Intents** - Uncommitted transactional writes with metadata
3. **Tombstones** - Deletion markers
4. **Inline Values** - Unversioned values at timestamp 0

### Read Operations

`MVCCGet` performs versioned reads:
- Finds the most recent value at or before the read timestamp
- Checks for intents that may conflict
- Handles uncertainty intervals for clock skew
- Returns appropriate errors for conflicts

### Write Operations

`MVCCPut` handles versioned writes:
- Transactional writes create intents
- Non-transactional writes go directly to versioned values
- Maintains intent history for rollback capability
- Updates MVCC statistics

### Intent Resolution

Intents represent uncommitted writes and must be resolved:
- **Commit**: Convert intent to regular versioned value
- **Abort**: Remove intent and restore previous value
- **Push**: Move intent timestamp forward for conflicts

## Isolation Levels

### Serializable (Default)

CockroachDB implements serializable isolation using a variant of Write-Snapshot Isolation:

- **Implementation**: Timestamp ordering with read tracking
- **Conflict Detection**: Write-write via intents, read-write via timestamp cache
- **Optimizations**: Read refresh to avoid restarts
- **Guarantees**: No anomalies (dirty reads, non-repeatable reads, phantoms, write skew)

### Snapshot Isolation

Maps to PostgreSQL's REPEATABLE READ:
- **Implementation**: Fixed read timestamp for entire transaction
- **Conflict Detection**: Write-write conflicts only
- **Trade-offs**: Allows write skew but better performance
- **Use Cases**: Applications that can tolerate write skew

### Read Committed

Weakest isolation level:
- **Implementation**: New read timestamp for each statement
- **Conflict Detection**: Minimal, only direct write conflicts
- **Trade-offs**: Best performance but allows most anomalies
- **Use Cases**: Read-heavy workloads with relaxed consistency needs

### Key Mechanisms

1. **Timestamp Cache**: Tracks reads to detect conflicts
2. **Refresh Spans**: Allows timestamp advancement without restart (4MB limit)
3. **Uncertainty Intervals**: Handle clock skew in distributed setting
4. **Write Too Old Errors**: Prevent lost updates

## Transaction Lifecycle

### 1. Transaction Initiation

```
Client → SQL Layer → connExecutor → txnState → kv.Txn
```

- SQL layer creates transaction state
- Assigns initial timestamp and priority
- Establishes transaction context

### 2. Statement Execution

```
SQL Statement → Plan → Execute → KV Operations → Storage
```

- Each statement flows through the interceptor stack
- Writes create intents at provisional timestamp
- Reads check for conflicts and track spans

### 3. Commit Protocol

#### Standard Commit
1. Client sends EndTxn(commit=true)
2. Create transaction record with COMMITTED status
3. Respond to client
4. Asynchronously resolve intents

#### Parallel Commit Optimization
1. Write intents with STAGING status
2. If all writes succeed, transaction implicitly commits
3. Saves one consensus round-trip
4. Falls back to standard commit on failure

### 4. Abort/Rollback

1. Client sends EndTxn(commit=false) or timeout occurs
2. Create/update transaction record with ABORTED status
3. Asynchronously clean up intents
4. Update abort span to prevent zombie transactions

## Distributed Transaction Coordination

### Transaction Records

Stored at an "anchor key" and serve as authoritative transaction state:
- **Location**: First key in transaction or randomized
- **Contents**: Status, timestamp, intent spans, heartbeat
- **Purpose**: Coordination point for distributed transaction

### Heartbeating

Prevents transaction abandonment:
- Periodic heartbeats (1-5 seconds based on priority)
- Only sent by root transactions
- Timeout triggers abort and cleanup

### Two-Phase Commit

For transactions spanning multiple ranges:
1. **Prepare Phase**: Write intents on all participants
2. **Commit Phase**: Update transaction record and resolve intents
3. **Optimization**: Parallel commits reduce to single phase

## Concurrency Control

### Pessimistic Locking (Writes)

- Intents serve as exclusive locks
- Block other transactions from writing
- Can be pushed by higher priority transactions

### Optimistic Locking (Reads)

- No read locks, tracked in timestamp cache
- Conflicts detected at write time
- Enables high read concurrency

### Conflict Resolution

1. **Write-Write**: Later writer waits or pushes earlier intent
2. **Read-Write**: Write fails if conflicts with earlier read
3. **Priority-Based**: Higher priority transactions can push lower
4. **Deadlock Detection**: Distributed deadlock detection via push ordering

## Performance Optimizations

### Write Pipelining

- Async intent writes while transaction continues
- Proves writes at commit time
- Reduces latency for write-heavy transactions

### Span Refreshing

- Tracks read spans up to 4MB
- Validates no conflicts occurred
- Avoids full transaction restart

### Intent Resolution Batching

- Groups intent resolution by range
- Reduces network round-trips
- Handles both local and remote intents

### Lock Table

- In-memory structure for active locks
- Enables efficient conflict checking
- Supports multiple lock strengths

## Error Handling and Retries

### Retry Mechanisms

1. **Client-Side Retries**: For network errors
2. **Span Refresh**: For timestamp conflicts
3. **Automatic Retries**: For serialization errors
4. **Savepoint Rollback**: For application-level retries

### Error Types

- **Ambiguous Results**: Network failures during commit
- **Write Too Old**: Timestamp pushed beyond read timestamp
- **Transaction Aborted**: Explicit abort or timeout
- **Serialization Failure**: Isolation violation detected

## Observability

### Metrics

- Transaction throughput and latency
- Retry rates by type
- Intent resolution performance
- Conflict and contention metrics

### Tracing

- Distributed tracing support
- Transaction flow visualization
- Performance bottleneck identification

### Debugging

- Transaction status in system tables
- Intent visualization tools
- Deadlock detection logs

## Design Trade-offs

### Optimistic vs Pessimistic

CockroachDB uses a hybrid approach:
- **Optimistic for reads**: No locks, high concurrency
- **Pessimistic for writes**: Intents prevent conflicts
- **Benefits**: Good balance of performance and correctness

### Timestamp Ordering vs Locking

- **Choice**: Timestamp ordering with MVCC
- **Benefits**: No read locks, snapshot reads, time travel
- **Trade-offs**: Clock synchronization requirements

### Distributed Coordination

- **Transaction records**: Single coordination point
- **Parallel commits**: Reduce consensus operations  
- **Trade-offs**: Complexity for performance

## Future Considerations

1. **Read Committed Optimization**: Better performance for weaker isolation
2. **Pessimistic Locking**: Optional for high-contention workloads
3. **Global Transactions**: Cross-region optimizations
4. **Hardware Acceleration**: Leverage modern hardware features

## Conclusion

CockroachDB's transaction implementation provides strong ACID guarantees in a distributed setting through careful design choices:

- MVCC enables high concurrency without read locks
- Hybrid Logical Clocks provide global ordering
- Sophisticated coordination protocols minimize latency
- Multiple isolation levels offer flexibility
- Comprehensive observability aids operations

The system successfully balances correctness, performance, and operational simplicity, making it suitable for demanding distributed database workloads.