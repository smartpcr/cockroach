# CockroachDB Transaction and Isolation Implementation

## Overview

CockroachDB implements distributed transactions with strong consistency guarantees using a multi-layered architecture that combines MVCC (Multi-Version Concurrency Control), distributed locking, and hybrid logical clocks.

## Architecture Layers

### 1. SQL Layer
- **Location**: `/pkg/sql/txn_state.go`
- **Purpose**: Manages SQL-level transaction state and coordinates with the KV layer

### 2. KV Transaction Layer
- **Location**: `/pkg/kv/txn.go`
- **Purpose**: Provides the main `Txn` type that applications use for distributed transactions
- **Key Features**:
  - Thread-safe for concurrent use
  - Automatic retry handling
  - Commit triggers
  - Transaction priorities

### 3. Transaction Coordination Layer
- **Location**: `/pkg/kv/kvclient/kvcoord/txn_coord_sender.go`
- **Purpose**: Coordinates distributed transaction operations across multiple ranges
- **Components**:
  - Transaction state management
  - Heartbeating for long-running transactions
  - Retry logic for transient failures

### 4. Concurrency Control Layer
- **Location**: `/pkg/kv/kvserver/concurrency/`
- **Purpose**: Manages locks and resolves conflicts between concurrent transactions
- **Components**:
  - **Concurrency Manager** (`concurrency_manager.go`): Main coordinator
  - **Lock Table** (`lock_table.go`): In-memory tracking of locks
  - **Lock Table Waiter** (`lock_table_waiter.go`): Queue management for waiting transactions
  - **Latch Manager** (`latch_manager.go`): Non-blocking concurrency for non-conflicting operations

### 5. Storage Layer (MVCC)
- **Location**: `/pkg/storage/mvcc.go`
- **Purpose**: Implements multi-version concurrency control
- **Features**:
  - Versioned key-value storage
  - Timestamp-based visibility
  - Write intent management
  - Conflict detection

## Isolation Levels

CockroachDB supports three isolation levels (defined in `/pkg/kv/kvserver/concurrency/isolation/levels.go`):

### 1. Serializable (Default)
- **Implementation**: SSI (Serializable Snapshot Isolation)
- **Properties**:
  - Prevents all anomalies including write skew
  - Uses per-transaction read snapshots
  - Strongest isolation guarantee

### 2. Snapshot
- **Implementation**: SI (Snapshot Isolation)
- **Properties**:
  - Allows write skew
  - Uses per-transaction read snapshots
  - Good balance of performance and consistency

### 3. Read Committed
- **Properties**:
  - Allows write skew
  - Uses per-statement read snapshots
  - Highest performance, weakest consistency

## Key Implementation Details

### Hybrid Logical Clocks (HLC)
- **Location**: `/pkg/util/hlc/`
- **Purpose**: Provides causality tracking with physical time relation
- **Benefits**:
  - Handles clock skew between nodes
  - Maintains causality across the cluster
  - Enables consistent snapshots

### Transaction Timestamps
Each transaction maintains several timestamps:
- **Read Timestamp**: Determines what data versions are visible
- **Write Timestamp**: Timestamp at which writes are performed
- **Global Uncertainty Limit**: Upper bound for clock uncertainty
- **Observed Timestamps**: Per-node timestamps to reduce uncertainty

### Write Intents
- Provisional writes that act as distributed locks
- Stored inline with MVCC data
- Resolved (committed or aborted) during transaction finalization
- Visible to other transactions as locks

### Lock Types
1. **Replicated Locks**: Write intents stored in MVCC
2. **Unreplicated Locks**: Stored in memory lock table for better performance

### Transaction Interceptors
Multiple interceptors handle different aspects of transaction processing:

1. **Heartbeater** (`txn_interceptor_heartbeater.go`)
   - Keeps long-running transactions alive
   - Prevents transaction expiration

2. **Committer** (`txn_interceptor_committer.go`)
   - Implements the distributed commit protocol
   - Handles parallel commits optimization

3. **Pipeliner** (`txn_interceptor_pipeliner.go`)
   - Optimizes write performance through pipelining
   - Reduces latency for write-heavy workloads

4. **Span Refresher** (`txn_interceptor_span_refresher.go`)
   - Enables read refresh for serializable isolation
   - Allows transactions to retry at higher timestamps

5. **Sequence Number Allocator** (`txn_interceptor_seq_num_allocator.go`)
   - Manages operation ordering within transactions

## Conflict Resolution

### Lock Conflict Handling
- Transactions queue when encountering locks
- Configurable maximum queue length (`kv.lock_table.maximum_lock_wait_queue_length`)
- Deadlock detection prevents circular dependencies

### Write-Write Conflicts
- Detected through write intents
- Later transaction pushes earlier transaction's timestamp
- May trigger transaction retry

### Read-Write Conflicts (Serializable Only)
- Tracked to prevent write skew
- May trigger read refresh or transaction retry

## Performance Optimizations

### Parallel Commits
- Allows committing without waiting for all write intents to be resolved
- Significantly reduces commit latency for distributed transactions

### Batch Pushed Lock Resolution
- Non-locking readers can defer and batch resolution of pushed locks
- Reduces overhead for read-heavy workloads

### Pipelining
- Writes can be pipelined without waiting for individual acknowledgments
- Improves throughput for write-heavy transactions

## Configuration and Tuning

### Key Settings
- `kv.transaction.internal.max_auto_retries`: Maximum automatic retries (default: 100)
- `kv.lock_table.maximum_lock_wait_queue_length`: Maximum lock queue length
- `storage.mvcc.max_intents_per_error`: Maximum locks returned in errors
- `kv.lock_table.batch_pushed_lock_resolution.enabled`: Enable batched lock resolution

### Reliability Features
- Unreplicated lock preservation during:
  - Range splits (`kv.lock_table.unreplicated_lock_reliability.split.enabled`)
  - Lease transfers (`kv.lock_table.unreplicated_lock_reliability.lease_transfer.enabled`)
  - Range merges (`kv.lock_table.unreplicated_lock_reliability.merge.enabled`)

## Summary

CockroachDB's transaction implementation provides strong consistency guarantees in a distributed environment through:
- Multi-version concurrency control (MVCC)
- Hybrid logical clocks for distributed timestamp ordering
- Sophisticated lock management with both replicated and unreplicated locks
- Multiple isolation levels to balance consistency and performance
- Extensive optimization for common patterns (parallel commits, pipelining, batched resolution)

The system is designed to provide ACID guarantees while maintaining high performance and availability in a distributed setting.
