# CockroachDB Implementation Document

## Table of Contents
1. [Introduction](#introduction)
2. [Development Environment](#development-environment)
3. [Code Organization](#code-organization)
4. [Key Implementation Patterns](#key-implementation-patterns)
5. [SQL Layer Implementation](#sql-layer-implementation)
6. [Storage Implementation](#storage-implementation)
7. [Transaction Implementation](#transaction-implementation)
8. [Distribution Implementation](#distribution-implementation)
9. [Replication Implementation](#replication-implementation)
10. [Performance Optimizations](#performance-optimizations)
11. [Testing Strategy](#testing-strategy)
12. [Debugging and Troubleshooting](#debugging-and-troubleshooting)

## Introduction

This document provides detailed implementation guidance for CockroachDB, covering the key patterns, algorithms, and techniques used throughout the codebase. It serves as a reference for developers working on or with CockroachDB.

## Development Environment

### Prerequisites

1. **Go Version**: 1.23.7 or later
2. **Build Tools**:
   - Bazel (primary build system)
   - Make (wrapper around dev tool)
   - Docker (for testing)
   - Git (version control)

### Setting Up Development Environment

```bash
# Clone the repository
git clone https://github.com/cockroachdb/cockroach.git
cd cockroach

# Install dependencies and verify environment
./dev doctor

# Build CockroachDB
./dev build

# Run tests
./dev test

# Generate code
./dev generate
```

### Build System

CockroachDB uses Bazel as its primary build system:

```python
# Example BUILD.bazel file structure
go_library(
    name = "storage",
    srcs = ["storage.go"],
    importpath = "github.com/cockroachdb/cockroach/pkg/storage",
    deps = [
        "//pkg/roachpb",
        "//pkg/util/log",
        "@com_github_cockroachdb_pebble//:pebble",
    ],
)

go_test(
    name = "storage_test",
    srcs = ["storage_test.go"],
    embed = [":storage"],
    deps = ["//pkg/testutils"],
)
```

## Code Organization

### Package Structure

```
cockroach/
├── pkg/                    # Main source code
│   ├── sql/               # SQL layer
│   ├── kv/                # Key-value layer
│   ├── storage/           # Storage engine
│   ├── roachpb/           # Protocol buffers
│   ├── server/            # Server implementation
│   ├── cli/               # Command-line interface
│   ├── ccl/               # Commercial features
│   └── util/              # Utilities
├── docs/                  # Documentation
├── build/                 # Build scripts
└── scripts/               # Development scripts
```

### Key Packages

1. **pkg/sql**: SQL parsing, planning, and execution
2. **pkg/kv**: Distributed key-value operations
3. **pkg/storage**: Local storage and MVCC
4. **pkg/roachpb**: Protocol buffer definitions
5. **pkg/rpc**: RPC framework
6. **pkg/gossip**: Gossip protocol
7. **pkg/testutils**: Testing utilities

## Key Implementation Patterns

### Error Handling

```go
// Standard error handling pattern
func processData(data []byte) error {
    if len(data) == 0 {
        return errors.New("empty data")
    }
    
    // Use errors.Wrap for context
    if err := validateData(data); err != nil {
        return errors.Wrap(err, "validation failed")
    }
    
    // Use typed errors for specific conditions
    if isCorrupted(data) {
        return &CorruptionError{
            Data: data,
            Reason: "checksum mismatch",
        }
    }
    
    return nil
}
```

### Context Usage

```go
// Context propagation pattern
func executeQuery(ctx context.Context, query string) (Result, error) {
    // Add tracing span
    ctx, span := tracing.ChildSpan(ctx, "execute-query")
    defer span.Finish()
    
    // Check for cancellation
    if err := ctx.Err(); err != nil {
        return nil, err
    }
    
    // Add timeout
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    // Execute with context
    return db.QueryContext(ctx, query)
}
```

### Logging

```go
// Structured logging pattern
import "github.com/cockroachdb/cockroach/pkg/util/log"

func processRequest(req Request) {
    log.Infof(ctx, "processing request: id=%d type=%s", req.ID, req.Type)
    
    if log.V(2) {
        log.VEventf(ctx, 2, "detailed request info: %+v", req)
    }
    
    // Log with key-value pairs
    log.Event(ctx, "request_processed", 
        log.Safe("request_id", req.ID),
        log.Safe("duration_ms", duration.Milliseconds()))
}
```

### Protobuf Usage

```protobuf
// Example protobuf definition
syntax = "proto3";
package cockroach.sql;

message TableDescriptor {
    uint32 id = 1;
    string name = 2;
    repeated ColumnDescriptor columns = 3;
    repeated IndexDescriptor indexes = 4;
    
    message ColumnDescriptor {
        uint32 id = 1;
        string name = 2;
        Type type = 3;
        bool nullable = 4;
    }
}
```

## SQL Layer Implementation

### Query Processing Pipeline

```go
// Simplified query processing
func (p *planner) Execute(ctx context.Context, query string) (Result, error) {
    // 1. Parse SQL
    stmts, err := parser.Parse(query)
    if err != nil {
        return nil, err
    }
    
    // 2. Analyze and type-check
    typed, err := p.analyzeStatement(ctx, stmts[0])
    if err != nil {
        return nil, err
    }
    
    // 3. Optimize
    optimized, err := p.optimize(ctx, typed)
    if err != nil {
        return nil, err
    }
    
    // 4. Generate physical plan
    plan, err := p.makePhysicalPlan(ctx, optimized)
    if err != nil {
        return nil, err
    }
    
    // 5. Execute
    return p.execPlan(ctx, plan)
}
```

### Plan Node Implementation

```go
// Example plan node
type scanNode struct {
    table     *TableDescriptor
    spans     roachpb.Spans
    filter    tree.TypedExpr
    columns   []ColumnOrdinal
    
    // Runtime state
    fetcher   row.Fetcher
    rowsRead  int64
}

func (n *scanNode) startExec(params runParams) error {
    // Initialize fetcher
    if err := n.fetcher.Init(
        params.ctx,
        n.table,
        n.spans,
        n.columns,
    ); err != nil {
        return err
    }
    
    return n.fetcher.StartScan(params.ctx, params.txn)
}

func (n *scanNode) Next(params runParams) (bool, error) {
    for {
        // Fetch next row
        row, err := n.fetcher.NextRow(params.ctx)
        if err != nil || row == nil {
            return false, err
        }
        
        // Apply filter
        if n.filter != nil {
            passed, err := sqlbase.RunFilter(n.filter, row)
            if err != nil {
                return false, err
            }
            if !passed {
                continue
            }
        }
        
        n.rowsRead++
        return true, nil
    }
}
```

### Type System Implementation

```go
// Type checking example
func typeCheckExpr(
    ctx context.Context,
    expr tree.Expr,
    desired *types.T,
) (tree.TypedExpr, error) {
    // Handle constants
    if d, ok := expr.(*tree.DInt); ok {
        return d, nil
    }
    
    // Handle binary operations
    if b, ok := expr.(*tree.BinaryExpr); ok {
        left, err := typeCheckExpr(ctx, b.Left, nil)
        if err != nil {
            return nil, err
        }
        
        right, err := typeCheckExpr(ctx, b.Right, nil)
        if err != nil {
            return nil, err
        }
        
        // Find overload
        op, err := findBinaryOverload(b.Op, left.ResolvedType(), right.ResolvedType())
        if err != nil {
            return nil, err
        }
        
        return &tree.TypedBinaryExpr{
            Op:    op,
            Left:  left,
            Right: right,
        }, nil
    }
    
    return nil, errors.Newf("unsupported expression: %T", expr)
}
```

## Storage Implementation

### MVCC Implementation

```go
// MVCC key encoding
type MVCCKey struct {
    Key       roachpb.Key
    Timestamp hlc.Timestamp
}

func EncodeMVCCKey(key roachpb.Key, timestamp hlc.Timestamp) []byte {
    // Encode key + timestamp
    buf := make([]byte, len(key)+mvccEncodedTimeLen)
    copy(buf, key)
    encodeTimestamp(buf[len(key):], timestamp)
    return buf
}

// MVCC value structure
type MVCCValue struct {
    Value   roachpb.Value
    Deleted bool
    IntentHistory []IntentHistoryEntry
}

// MVCC scan implementation
func MVCCScan(
    ctx context.Context,
    reader Reader,
    key, endKey roachpb.Key,
    timestamp hlc.Timestamp,
    opts MVCCScanOptions,
) ([]roachpb.KeyValue, error) {
    iter := reader.NewMVCCIterator(MVCCIterOptions{
        UpperBound: endKey,
        Timestamp:  timestamp,
    })
    defer iter.Close()
    
    var results []roachpb.KeyValue
    for iter.SeekGE(MVCCKey{Key: key}); iter.Valid(); iter.Next() {
        // Check timestamp
        if iter.Key().Timestamp.Less(timestamp) {
            continue
        }
        
        // Handle intents
        if iter.IsIntent() {
            if err := handleIntent(ctx, iter); err != nil {
                return nil, err
            }
        }
        
        results = append(results, roachpb.KeyValue{
            Key:   iter.Key().Key,
            Value: iter.Value(),
        })
        
        if opts.MaxKeys > 0 && len(results) >= opts.MaxKeys {
            break
        }
    }
    
    return results, iter.Error()
}
```

### RocksDB Integration

```go
// RocksDB wrapper
type RocksDB struct {
    db     *C.DBEngine
    opts   *Options
    mu     sync.RWMutex
    closed bool
}

func (r *RocksDB) Put(key MVCCKey, value []byte) error {
    r.mu.RLock()
    defer r.mu.RUnlock()
    
    if r.closed {
        return errors.New("RocksDB is closed")
    }
    
    cKey := C.CString(string(EncodeMVCCKey(key.Key, key.Timestamp)))
    defer C.free(unsafe.Pointer(cKey))
    
    cValue := C.CBytes(value)
    defer C.free(cValue)
    
    status := C.DBPut(r.db, cKey, C.int(len(key)), cValue, C.int(len(value)))
    return statusToError(status)
}
```

### Write Batch Implementation

```go
// Batch operations for efficiency
type Batch struct {
    repr    []byte
    entries []BatchEntry
    
    // Statistics
    count     int
    size      int64
    timestamp hlc.Timestamp
}

func (b *Batch) Put(key MVCCKey, value []byte) {
    b.entries = append(b.entries, BatchEntry{
        Kind:  BatchEntryPut,
        Key:   key,
        Value: value,
    })
    
    b.count++
    b.size += int64(len(key.Key) + len(value))
}

func (b *Batch) Commit(writer Writer) error {
    // Build RocksDB batch
    batch := writer.NewBatch()
    defer batch.Close()
    
    for _, entry := range b.entries {
        switch entry.Kind {
        case BatchEntryPut:
            if err := batch.Put(entry.Key, entry.Value); err != nil {
                return err
            }
        case BatchEntryDelete:
            if err := batch.Delete(entry.Key); err != nil {
                return err
            }
        }
    }
    
    return batch.Commit()
}
```

## Transaction Implementation

### Transaction Coordinator

```go
// Transaction coordinator implementation
type TxnCoordSender struct {
    clock           *hlc.Clock
    heartbeatInterval time.Duration
    
    mu struct {
        sync.Mutex
        txn         *roachpb.Transaction
        intentSpans []roachpb.Span
    }
}

func (tc *TxnCoordSender) Send(
    ctx context.Context,
    ba roachpb.BatchRequest,
) (*roachpb.BatchResponse, error) {
    // Start transaction if needed
    if tc.mu.txn == nil {
        tc.mu.txn = &roachpb.Transaction{
            ID:        uuid.MakeV4(),
            Priority:  roachpb.NormalUserPriority,
            Timestamp: tc.clock.Now(),
        }
    }
    
    // Add transaction to request
    ba.Txn = tc.mu.txn
    
    // Track intents for cleanup
    tc.trackIntents(ba)
    
    // Send request
    br, err := tc.wrapped.Send(ctx, ba)
    if err != nil {
        return nil, err
    }
    
    // Update transaction state
    if br.Txn != nil {
        tc.mu.txn.Update(br.Txn)
    }
    
    // Handle transaction retry
    if br.Error != nil && br.Error.TransactionRestart != roachpb.TransactionRestart_NONE {
        return nil, roachpb.NewTransactionRetryError(br.Error.TransactionRestart)
    }
    
    return br, nil
}
```

### Intent Resolution

```go
// Intent resolution
type IntentResolver struct {
    stores      *Stores
    pushTxnQueue *PushTxnQueue
}

func (ir *IntentResolver) ResolveIntents(
    ctx context.Context,
    intents []roachpb.Intent,
    opts ResolveOptions,
) error {
    // Group intents by range
    grouped := ir.groupIntentsByRange(intents)
    
    // Resolve in parallel
    g, gCtx := errgroup.WithContext(ctx)
    for rangeID, rangeIntents := range grouped {
        rangeID := rangeID
        rangeIntents := rangeIntents
        
        g.Go(func() error {
            return ir.resolveRangeIntents(gCtx, rangeID, rangeIntents, opts)
        })
    }
    
    return g.Wait()
}

func (ir *IntentResolver) resolveRangeIntents(
    ctx context.Context,
    rangeID roachpb.RangeID,
    intents []roachpb.Intent,
    opts ResolveOptions,
) error {
    // Build resolve request
    req := &roachpb.ResolveIntentRangeRequest{
        RangeID:  rangeID,
        IntentTxn: intents[0].Txn,
        Status:    opts.Status,
    }
    
    // Send to range
    return ir.stores.Send(ctx, req)
}
```

### Concurrency Control

```go
// Latch manager for concurrency control
type LatchManager struct {
    mu    sync.Mutex
    latchSets map[roachpb.Key]*latchSet
}

type latchSet struct {
    mu      sync.Mutex
    waiters list.List
}

func (lm *LatchManager) Acquire(ctx context.Context, spans []roachpb.Span) (*Guard, error) {
    g := &Guard{
        spans: spans,
        done:  make(chan struct{}),
    }
    
    // Try to acquire all latches
    for _, span := range spans {
        if err := lm.acquireSpan(ctx, span, g); err != nil {
            g.Release()
            return nil, err
        }
    }
    
    return g, nil
}

func (lm *LatchManager) acquireSpan(
    ctx context.Context,
    span roachpb.Span,
    g *Guard,
) error {
    lm.mu.Lock()
    ls, ok := lm.latchSets[span.Key]
    if !ok {
        ls = &latchSet{}
        lm.latchSets[span.Key] = ls
    }
    lm.mu.Unlock()
    
    ls.mu.Lock()
    defer ls.mu.Unlock()
    
    // Check for conflicts
    for e := ls.waiters.Front(); e != nil; e = e.Next() {
        other := e.Value.(*Guard)
        if g.conflicts(other) {
            // Wait for conflicting guard
            ls.waiters.PushBack(g)
            return g.wait(ctx)
        }
    }
    
    // No conflicts, acquire immediately
    ls.waiters.PushBack(g)
    return nil
}
```

## Distribution Implementation

### Range Addressing

```go
// Range descriptor cache
type RangeCache struct {
    db    *kv.DB
    mu    sync.RWMutex
    cache map[string]*roachpb.RangeDescriptor
}

func (rc *RangeCache) LookupRangeDescriptor(
    ctx context.Context,
    key roachpb.Key,
) (*roachpb.RangeDescriptor, error) {
    // Check cache
    rc.mu.RLock()
    if desc, ok := rc.cache[string(key)]; ok {
        rc.mu.RUnlock()
        return desc, nil
    }
    rc.mu.RUnlock()
    
    // Meta lookup
    metaKey := keys.RangeMetaKey(key)
    kvs, err := rc.db.Scan(ctx, metaKey, metaKey.Next(), 1)
    if err != nil {
        return nil, err
    }
    
    if len(kvs) == 0 {
        return nil, errors.Newf("no range descriptor for key %s", key)
    }
    
    var desc roachpb.RangeDescriptor
    if err := kvs[0].Value.GetProto(&desc); err != nil {
        return nil, err
    }
    
    // Update cache
    rc.mu.Lock()
    rc.cache[string(key)] = &desc
    rc.mu.Unlock()
    
    return &desc, nil
}
```

### Load Balancing

```go
// Load-based replica selection
type ReplicaSlice []roachpb.ReplicaDescriptor

func (rs ReplicaSlice) OptimalReplica(
    nodeID roachpb.NodeID,
    latencies map[roachpb.NodeID]time.Duration,
) roachpb.ReplicaDescriptor {
    // Prefer local replica
    for _, replica := range rs {
        if replica.NodeID == nodeID {
            return replica
        }
    }
    
    // Choose replica with lowest latency
    var best roachpb.ReplicaDescriptor
    minLatency := time.Duration(math.MaxInt64)
    
    for _, replica := range rs {
        if latency, ok := latencies[replica.NodeID]; ok && latency < minLatency {
            best = replica
            minLatency = latency
        }
    }
    
    return best
}
```

### Distributed Sender

```go
// DistSender routes requests to ranges
type DistSender struct {
    nodeDescs    *NodeDescStore
    rangeCache   *RangeCache
    transportFactory TransportFactory
}

func (ds *DistSender) Send(
    ctx context.Context,
    ba roachpb.BatchRequest,
) (*roachpb.BatchResponse, error) {
    // Determine target ranges
    ranges, err := ds.divideByRange(ctx, ba)
    if err != nil {
        return nil, err
    }
    
    // Send to each range
    var responses []*roachpb.BatchResponse
    for _, rangeReq := range ranges {
        br, err := ds.sendToRange(ctx, rangeReq)
        if err != nil {
            return nil, err
        }
        responses = append(responses, br)
    }
    
    // Combine responses
    return ds.combineResponses(responses), nil
}

func (ds *DistSender) sendToRange(
    ctx context.Context,
    ba roachpb.BatchRequest,
) (*roachpb.BatchResponse, error) {
    // Lookup range descriptor
    desc, err := ds.rangeCache.LookupRangeDescriptor(ctx, ba.Key)
    if err != nil {
        return nil, err
    }
    
    // Select replica
    replica := desc.Replicas.OptimalReplica(ds.nodeDescs.GetNodeID(), ds.getLatencies())
    
    // Get transport
    transport, err := ds.transportFactory.GetTransport(replica.NodeID)
    if err != nil {
        return nil, err
    }
    
    // Send RPC
    return transport.Send(ctx, ba)
}
```

## Replication Implementation

### Raft Integration

```go
// Replica with Raft
type Replica struct {
    RangeID   roachpb.RangeID
    store     *Store
    raftGroup *raft.RawNode
    
    mu struct {
        sync.RWMutex
        state      replicaState
        desc       *roachpb.RangeDescriptor
        leaseHolder bool
    }
}

func (r *Replica) propose(ctx context.Context, ba roachpb.BatchRequest) (*roachpb.BatchResponse, error) {
    // Check if we're the lease holder
    if !r.mu.leaseHolder {
        return nil, roachpb.NewNotLeaseHolderError(r.mu.desc.Replicas[0])
    }
    
    // Encode proposal
    prop := ProposalData{
        Batch:     ba,
        Timestamp: r.store.Clock().Now(),
    }
    
    data, err := prop.Marshal()
    if err != nil {
        return nil, err
    }
    
    // Propose through Raft
    if err := r.raftGroup.Propose(ctx, data); err != nil {
        return nil, err
    }
    
    // Wait for application
    return r.waitForApplication(ctx, prop)
}

func (r *Replica) handleRaftMessage(msg raftpb.Message) error {
    // Process different message types
    switch msg.Type {
    case raftpb.MsgApp:
        // Leader appending entries
        return r.raftGroup.Step(msg)
        
    case raftpb.MsgVote:
        // Election vote request
        return r.handleVoteRequest(msg)
        
    case raftpb.MsgHeartbeat:
        // Leader heartbeat
        r.updateLeaseHolder(msg.From)
        return r.raftGroup.Step(msg)
        
    default:
        return r.raftGroup.Step(msg)
    }
}
```

### State Machine

```go
// Apply committed Raft entries
func (r *Replica) applyCommittedEntries(ctx context.Context) error {
    for {
        committed := r.raftGroup.CommittedEntries()
        if len(committed) == 0 {
            break
        }
        
        for _, entry := range committed {
            if err := r.applyEntry(ctx, entry); err != nil {
                return err
            }
        }
        
        r.raftGroup.Advance()
    }
    
    return nil
}

func (r *Replica) applyEntry(ctx context.Context, entry raftpb.Entry) error {
    // Decode command
    var prop ProposalData
    if err := prop.Unmarshal(entry.Data); err != nil {
        return err
    }
    
    // Create batch for application
    batch := r.store.Engine().NewBatch()
    defer batch.Close()
    
    // Apply each request
    for _, req := range prop.Batch.Requests {
        if err := r.applyRequest(ctx, batch, req); err != nil {
            return err
        }
    }
    
    // Commit batch
    return batch.Commit()
}
```

### Snapshot Handling

```go
// Snapshot generation and application
func (r *Replica) generateSnapshot(ctx context.Context) (*roachpb.RaftSnapshotData, error) {
    // Get consistent view
    snap := r.store.Engine().NewSnapshot()
    defer snap.Close()
    
    // Iterate over range data
    iter := snap.NewMVCCIterator(MVCCIterOptions{
        StartKey: r.mu.desc.StartKey,
        EndKey:   r.mu.desc.EndKey,
    })
    defer iter.Close()
    
    var data roachpb.RaftSnapshotData
    for iter.SeekGE(MVCCKey{Key: r.mu.desc.StartKey}); iter.Valid(); iter.Next() {
        data.KV = append(data.KV, roachpb.KeyValue{
            Key:   iter.Key().Key,
            Value: iter.Value(),
        })
    }
    
    return &data, iter.Error()
}

func (r *Replica) applySnapshot(ctx context.Context, snap *roachpb.RaftSnapshotData) error {
    // Clear existing data
    batch := r.store.Engine().NewBatch()
    defer batch.Close()
    
    if err := batch.ClearRange(r.mu.desc.StartKey, r.mu.desc.EndKey); err != nil {
        return err
    }
    
    // Write snapshot data
    for _, kv := range snap.KV {
        if err := batch.Put(kv.Key, kv.Value); err != nil {
            return err
        }
    }
    
    return batch.Commit()
}
```

## Performance Optimizations

### Batching Strategies

```go
// Request batching for efficiency
type Batcher struct {
    maxSize     int
    maxDelay    time.Duration
    
    mu struct {
        sync.Mutex
        pending  []Request
        timer    *time.Timer
        resultCh chan Result
    }
}

func (b *Batcher) Add(req Request) <-chan Result {
    b.mu.Lock()
    defer b.mu.Unlock()
    
    resultCh := make(chan Result, 1)
    b.mu.pending = append(b.mu.pending, req)
    
    // Start timer if first request
    if len(b.mu.pending) == 1 {
        b.mu.timer = time.AfterFunc(b.maxDelay, b.flush)
    }
    
    // Flush if batch is full
    if len(b.mu.pending) >= b.maxSize {
        b.flush()
    }
    
    return resultCh
}

func (b *Batcher) flush() {
    b.mu.Lock()
    pending := b.mu.pending
    b.mu.pending = nil
    b.mu.Unlock()
    
    // Process batch
    results := b.processBatch(pending)
    
    // Send results
    for i, result := range results {
        pending[i].ResultCh <- result
    }
}
```

### Caching

```go
// Multi-level caching
type Cache struct {
    l1 *ristretto.Cache  // In-memory cache
    l2 *diskCache        // Disk-based cache
}

func (c *Cache) Get(ctx context.Context, key string) (interface{}, error) {
    // Check L1 cache
    if val, found := c.l1.Get(key); found {
        return val, nil
    }
    
    // Check L2 cache
    val, err := c.l2.Get(ctx, key)
    if err == nil {
        // Promote to L1
        c.l1.Set(key, val, 1)
        return val, nil
    }
    
    return nil, ErrCacheMiss
}

func (c *Cache) Set(ctx context.Context, key string, val interface{}) error {
    // Set in both levels
    c.l1.Set(key, val, 1)
    return c.l2.Set(ctx, key, val)
}
```

### Memory Management

```go
// Memory accounting
type MemoryMonitor struct {
    pool     *MemoryPool
    reserved int64
    used     int64
    mu       sync.Mutex
}

func (m *MemoryMonitor) Acquire(ctx context.Context, size int64) error {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    // Check if we can allocate
    if m.used+size > m.reserved {
        // Try to reserve more from pool
        additional := size - (m.reserved - m.used)
        if err := m.pool.Reserve(ctx, additional); err != nil {
            return err
        }
        m.reserved += additional
    }
    
    m.used += size
    return nil
}

func (m *MemoryMonitor) Release(size int64) {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    m.used -= size
    
    // Release back to pool if we have excess
    excess := m.reserved - m.used
    if excess > m.reserved/2 {
        m.pool.Release(excess / 2)
        m.reserved -= excess / 2
    }
}
```

## Testing Strategy

### Unit Testing

```go
// Example unit test
func TestMVCCScan(t *testing.T) {
    defer leaktest.AfterTest(t)()
    
    ctx := context.Background()
    engine := storage.NewDefaultInMem()
    defer engine.Close()
    
    // Write test data
    testData := []struct {
        key   roachpb.Key
        value roachpb.Value
        ts    hlc.Timestamp
    }{
        {roachpb.Key("a"), roachpb.MakeValueFromString("1"), hlc.Timestamp{WallTime: 1}},
        {roachpb.Key("b"), roachpb.MakeValueFromString("2"), hlc.Timestamp{WallTime: 2}},
        {roachpb.Key("c"), roachpb.MakeValueFromString("3"), hlc.Timestamp{WallTime: 3}},
    }
    
    for _, d := range testData {
        if err := storage.MVCCPut(ctx, engine, nil, d.key, d.ts, d.value, nil); err != nil {
            t.Fatal(err)
        }
    }
    
    // Test scan
    kvs, err := storage.MVCCScan(ctx, engine, roachpb.Key("a"), roachpb.Key("d"), 
        hlc.Timestamp{WallTime: 10}, storage.MVCCScanOptions{})
    require.NoError(t, err)
    require.Len(t, kvs, 3)
    
    // Verify results
    for i, kv := range kvs {
        require.Equal(t, testData[i].key, kv.Key)
        require.Equal(t, testData[i].value, kv.Value)
    }
}
```

### Integration Testing

```go
// Integration test example
func TestDistributedTransaction(t *testing.T) {
    defer leaktest.AfterTest(t)()
    
    // Start test cluster
    tc := testcluster.StartTestCluster(t, 3, base.TestClusterArgs{})
    defer tc.Stopper().Stop(context.Background())
    
    // Get SQL DB
    db := tc.ServerConn(0)
    
    // Create test table
    _, err := db.Exec(`
        CREATE TABLE test (
            id INT PRIMARY KEY,
            value TEXT
        )
    `)
    require.NoError(t, err)
    
    // Test distributed transaction
    tx, err := db.Begin()
    require.NoError(t, err)
    
    // Insert data
    _, err = tx.Exec("INSERT INTO test VALUES (1, 'a'), (2, 'b'), (3, 'c')")
    require.NoError(t, err)
    
    // Read in same transaction
    var count int
    err = tx.QueryRow("SELECT COUNT(*) FROM test").Scan(&count)
    require.NoError(t, err)
    require.Equal(t, 3, count)
    
    // Commit
    require.NoError(t, tx.Commit())
    
    // Verify data is persisted
    err = db.QueryRow("SELECT COUNT(*) FROM test").Scan(&count)
    require.NoError(t, err)
    require.Equal(t, 3, count)
}
```

### Benchmark Testing

```go
// Benchmark example
func BenchmarkMVCCPut(b *testing.B) {
    ctx := context.Background()
    engine := storage.NewDefaultInMem()
    defer engine.Close()
    
    value := roachpb.MakeValueFromBytes(bytes.Repeat([]byte("a"), 1024))
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        key := roachpb.Key(fmt.Sprintf("key-%d", i))
        ts := hlc.Timestamp{WallTime: int64(i)}
        
        if err := storage.MVCCPut(ctx, engine, nil, key, ts, value, nil); err != nil {
            b.Fatal(err)
        }
    }
    
    b.StopTimer()
}
```

## Debugging and Troubleshooting

### Debug Commands

```bash
# Start with debug logging
./cockroach start --vmodule=replica=2,store=3

# Debug specific subsystem
./cockroach debug range-data <store-dir> --range=<range-id>

# Analyze Raft log
./cockroach debug raft-log <store-dir> --range=<range-id>

# Check range descriptors
./cockroach debug range-descriptors <store-dir>
```

### Tracing

```go
// Add tracing to operations
func (s *Store) executeRequest(ctx context.Context, req Request) (Response, error) {
    // Start span
    ctx, span := tracing.ChildSpan(ctx, "store.executeRequest")
    defer span.Finish()
    
    // Log span metadata
    span.LogKV(
        "request.type", req.Type(),
        "request.key", req.Key(),
        "store.id", s.StoreID(),
    )
    
    // Record timing
    start := timeutil.Now()
    defer func() {
        span.LogKV("duration.ns", timeutil.Since(start).Nanoseconds())
    }()
    
    // Execute with tracing context
    return s.impl.Execute(ctx, req)
}
```

### Metrics and Monitoring

```go
// Define metrics
var (
    metaReplicaCount = metric.Metadata{
        Name:        "replicas.count",
        Help:        "Number of replicas",
        Measurement: "Replicas",
        Unit:        metric.Unit_COUNT,
    }
    
    metaRangeSize = metric.Metadata{
        Name:        "range.size",
        Help:        "Size of range in bytes",
        Measurement: "Bytes",
        Unit:        metric.Unit_BYTES,
    }
)

// Store metrics
type StoreMetrics struct {
    ReplicaCount *metric.Gauge
    RangeSize    *metric.Counter
}

func newStoreMetrics() *StoreMetrics {
    return &StoreMetrics{
        ReplicaCount: metric.NewGauge(metaReplicaCount),
        RangeSize:    metric.NewCounter(metaRangeSize),
    }
}

// Update metrics
func (s *Store) updateMetrics() {
    s.metrics.ReplicaCount.Update(int64(s.replicaCount()))
    s.metrics.RangeSize.Inc(s.totalRangeSize())
}
```

## Conclusion

This implementation document provides a comprehensive guide to CockroachDB's internal implementation. The codebase follows consistent patterns for error handling, concurrency, and resource management. Understanding these patterns and the overall architecture is crucial for contributing to or extending CockroachDB.

Key takeaways:
- Layered architecture with clear separation of concerns
- Extensive use of Go idioms and best practices
- Strong emphasis on testing and observability
- Performance optimization through batching and caching
- Robust error handling and recovery mechanisms

For the most up-to-date implementation details, refer to the source code and inline documentation in the CockroachDB repository.
