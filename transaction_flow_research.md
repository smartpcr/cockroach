# CockroachDB Transaction Coordinator and Execution Flow Research

## Overview

CockroachDB implements a sophisticated distributed transaction system that ensures ACID properties across a distributed cluster. The transaction flow involves multiple layers from SQL execution down to the storage layer, with careful coordination to handle distributed consensus, conflict resolution, and fault tolerance.

## 1. Transaction Initiation (SQL to KV Layer)

### SQL Layer Components
- **connExecutor**: Manages the connection state machine and executes SQL statements
- **txnState**: Maintains transaction state including the KV transaction object, timestamps, priority, and isolation level
- **execStmt**: Entry point for statement execution that dispatches based on current transaction state

### Transaction States
```go
// From txn_state.go
type txnState int
const (
    txnPending      // Normal state for ongoing transactions
    txnRetryableError   // Transaction hit a retryable error
    txnError        // Non-retriable error encountered
    txnPrepared     // EndTxn(commit=true,prepare=true) succeeded
    txnFinalized    // Transaction committed or rolled back
)
```

### Transaction Types
- **Implicit transactions**: Single statement executed outside explicit BEGIN/COMMIT
- **Explicit transactions**: Started with BEGIN statement
- **Upgraded explicit transactions**: Started as implicit but upgraded via BEGIN

## 2. TxnCoordSender and Interceptor Stack

The `TxnCoordSender` is the core component that coordinates distributed transactions. It wraps a `DistSender` and maintains a stack of interceptors that transform requests and responses.

### Key Responsibilities
- Heartbeating transaction records
- Accumulating lock spans
- Handling retriable errors
- Managing transaction state transitions
- Coordinating parallel commits

### Interceptor Stack (in order)
1. **txnHeartbeater**: Manages transaction heartbeats to prevent abandonment
2. **txnSeqNumAllocator**: Assigns sequence numbers to requests
3. **txnWriteBuffer**: Buffers write operations for batching
4. **txnPipeliner**: Enables pipelined writes for performance
5. **txnCommitter**: Handles commit protocol including parallel commits
6. **txnSpanRefresher**: Manages read span refreshing for serializable isolation
7. **txnMetricRecorder**: Records transaction metrics
8. **txnLockGatekeeper**: Controls concurrent request execution

## 3. Transaction Lifecycle

### Begin Transaction
1. SQL parser recognizes BEGIN statement or implicit transaction start
2. `resetForNewSQLTxn` creates new KV transaction with:
   - Transaction ID (UUID)
   - Initial timestamp from HLC clock
   - Priority and isolation level
   - Uncertainty interval
3. Transaction context created with tracing span

### Execute Operations
1. Each SQL statement converted to KV operations (Get, Put, Delete, Scan)
2. Requests flow through interceptor stack:
   - Sequence numbers assigned
   - Writes buffered or pipelined
   - Read spans tracked for refresh
   - Intents written at provisional timestamp
3. Lock spans accumulated for cleanup

### Commit/Rollback
1. EndTxn request prepared with all lock spans
2. Commit protocol executes (standard or parallel commit)
3. Transaction record created/updated
4. Intents resolved asynchronously

## 4. Distributed Transaction Coordination

### Transaction Record
```proto
message TransactionRecord {
    TxnMeta meta;           // ID, key, timestamps
    TransactionStatus status;   // PENDING, STAGING, COMMITTED, ABORTED
    Timestamp last_heartbeat;
    repeated Span lock_spans;
    repeated SequencedWrite in_flight_writes;
    repeated IgnoredSeqNumRange ignored_seqnums;
}
```

### Transaction Status Flow
- **PENDING**: Active transaction with heartbeat
- **STAGING**: Parallel commit in progress
- **COMMITTED**: Successfully committed
- **ABORTED**: Rolled back or abandoned

## 5. Two-Phase Commit Protocol Implementation

### Standard Commit
1. All intents written at provisional timestamp
2. EndTxn creates transaction record in COMMITTED state
3. Intents resolved asynchronously to committed values

### Parallel Commit
1. Enabled by `parallelCommitsEnabled` setting
2. Transaction can be implicitly committed if:
   - Transaction record in STAGING status
   - All in-flight writes successful at or below commit timestamp
3. Benefits: Removes one consensus round-trip
4. Implementation in `txnCommitter` interceptor

### Commit Conditions
- **Explicit commit**: Transaction record with COMMITTED status
- **Implicit commit**: STAGING record with all in-flight writes confirmed

## 6. Transaction Record Management

### Heartbeating Mechanism
- Heartbeat interval prevents transaction abandonment
- Only root transactions send heartbeats
- Heartbeat loop started when first intent written
- Updates `last_heartbeat` field in transaction record

### Anchor Key Selection
- Transaction record stored at "anchor key"
- Can be randomized (`RandomizedTxnAnchorKeyEnabled`) or first locked key
- Determines which range owns the transaction record

## 7. Intent Resolution Flow

### Intent Resolver Components
- **IntentResolver**: Manages asynchronous intent resolution
- **Batching**: Groups intent resolution for efficiency
- **Push/Resolve**: Can push conflicting transactions or resolve intents

### Resolution Process
1. After commit: Intents upgraded to committed values
2. After abort: Intents removed
3. Batched by transaction for efficiency
4. Respects admission control and backpressure

### Resolution Strategies
```go
// Intent resolution settings
var intentResolverBatchSize = 100
var intentResolverRangeBatchSize = 10
var MaxTxnsPerIntentCleanupBatch = 100
```

## 8. Error Handling and Retry Logic

### Retryable Errors
- **TransactionRetryError**: General retry needed
- **TransactionRetryWithProtoRefreshError**: Can retry with updated proto
- **ReadWithinUncertaintyIntervalError**: Timestamp push needed
- **WriteIntentError**: Conflicting intent found

### Retry Mechanisms

#### Client-Side Retries
- Automatic retry with new epoch
- Preserves intents across retries
- Updates transaction proto

#### Span Refreshing
- Avoids full restarts for timestamp pushes
- Verifies no conflicting writes in read spans
- Updates timestamp if refresh succeeds
- Falls back to restart if refresh fails

### Refresh Algorithm
1. Collect read spans during execution
2. On timestamp push, check spans for conflicts
3. If no conflicts, update timestamp cache entries
4. If conflicts found, force transaction restart

## 9. Key Implementation Details

### Concurrency Control
- Pessimistic write locks (intents)
- Optimistic read locks (timestamp cache)
- Lock table tracks waiting transactions
- Deadlock detection via push timestamps

### Performance Optimizations
- Write pipelining reduces latency
- Parallel commits save consensus round-trip
- Read span condensing limits memory usage
- Batch intent resolution for efficiency

### Memory Management
- `MaxTxnRefreshSpansBytes`: Limits refresh span memory (default 4MB)
- Condensable span sets merge overlapping spans
- Falls back to transaction restart if memory exceeded

### Observability
- Transaction metrics tracked throughout lifecycle
- Tracing spans for debugging
- Audit logging for sensitive operations
- Performance counters for monitoring

## Architecture Flow Diagram

```
SQL Statement
    ↓
connExecutor.execStmt()
    ↓
txnState (transaction state machine)
    ↓
TxnCoordSender
    ↓
┌─────────────────────────┐
│  Interceptor Stack:     │
│  - txnHeartbeater      │
│  - txnSeqNumAllocator  │
│  - txnWriteBuffer      │
│  - txnPipeliner        │
│  - txnCommitter        │
│  - txnSpanRefresher    │
│  - txnMetricRecorder   │
│  - txnLockGatekeeper   │
└─────────────────────────┘
    ↓
DistSender
    ↓
Replica/Storage Layer
```

## Key Insights

1. **Distributed Coordination**: Transaction records serve as the source of truth for transaction state across the cluster

2. **Performance vs Correctness**: Parallel commits and pipelining optimize performance while maintaining strict serializability

3. **Failure Handling**: Comprehensive retry mechanisms handle both transient failures and conflicts

4. **Memory Efficiency**: Careful management of read spans and intent tracking prevents memory bloat

5. **Observability**: Rich instrumentation enables debugging and performance analysis

This architecture enables CockroachDB to provide serializable isolation with high performance across a distributed cluster while handling failures gracefully.