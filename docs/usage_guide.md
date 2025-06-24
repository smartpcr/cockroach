# CockroachDB Usage Guide

## Table of Contents
1. [Getting Started](#getting-started)
2. [Installation](#installation)
3. [Basic Operations](#basic-operations)
4. [SQL Features](#sql-features)
5. [Performance Tuning](#performance-tuning)
6. [Multi-Region Deployment](#multi-region-deployment)
7. [Backup and Recovery](#backup-and-recovery)
8. [Monitoring and Observability](#monitoring-and-observability)
9. [Security Best Practices](#security-best-practices)
10. [Troubleshooting](#troubleshooting)
11. [Best Practices](#best-practices)
12. [Common Use Cases](#common-use-cases)

## Getting Started

### Quick Start

```bash
# Download CockroachDB
curl https://binaries.cockroachdb.com/cockroach-latest.linux-amd64.tgz | tar -xz

# Start a single-node cluster
./cockroach start-single-node --insecure --store=node1 --listen-addr=localhost:26257

# Connect with SQL client
./cockroach sql --insecure --host=localhost:26257

# In another terminal, open the Admin UI
# Navigate to http://localhost:8080
```

### Production Deployment

```bash
# Start first node
./cockroach start \
  --certs-dir=certs \
  --store=node1 \
  --listen-addr=localhost:26257 \
  --http-addr=localhost:8080 \
  --join=localhost:26257,localhost:26258,localhost:26259

# Start additional nodes
./cockroach start \
  --certs-dir=certs \
  --store=node2 \
  --listen-addr=localhost:26258 \
  --http-addr=localhost:8081 \
  --join=localhost:26257,localhost:26258,localhost:26259

# Initialize the cluster
./cockroach init --certs-dir=certs --host=localhost:26257
```

## Installation

### System Requirements

- **Operating System**: Linux, macOS, Windows
- **Memory**: Minimum 2GB RAM per node
- **Storage**: SSD recommended, minimum 10GB free space
- **Network**: Low-latency network between nodes

### Installation Methods

#### Binary Download

```bash
# Linux
curl https://binaries.cockroachdb.com/cockroach-latest.linux-amd64.tgz | tar -xz

# macOS
curl https://binaries.cockroachdb.com/cockroach-latest.darwin-10.9-amd64.tgz | tar -xz

# Move to PATH
sudo mv cockroach /usr/local/bin/
```

#### Docker

```bash
# Pull the image
docker pull cockroachdb/cockroach:latest

# Run a single node
docker run -d \
  --name=roach1 \
  --hostname=roach1 \
  -p 26257:26257 \
  -p 8080:8080 \
  cockroachdb/cockroach:latest start-single-node --insecure
```

#### Kubernetes

```yaml
# cockroachdb-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cockroachdb
spec:
  serviceName: cockroachdb
  replicas: 3
  selector:
    matchLabels:
      app: cockroachdb
  template:
    metadata:
      labels:
        app: cockroachdb
    spec:
      containers:
      - name: cockroachdb
        image: cockroachdb/cockroach:latest
        command:
          - "/bin/bash"
          - "-ecx"
          - |
            exec /cockroach/cockroach start \
              --logtostderr \
              --certs-dir=/cockroach/cockroach-certs \
              --join=cockroachdb-0.cockroachdb,cockroachdb-1.cockroachdb,cockroachdb-2.cockroachdb \
              --cache=.25 \
              --max-sql-memory=.25
        volumeMounts:
        - name: datadir
          mountPath: /cockroach/cockroach-data
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

## Basic Operations

### Database Management

```sql
-- Create a database
CREATE DATABASE myapp;

-- List databases
SHOW DATABASES;

-- Use a database
USE myapp;

-- Drop a database
DROP DATABASE myapp;
```

### Table Operations

```sql
-- Create a table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username STRING UNIQUE NOT NULL,
    email STRING UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT current_timestamp(),
    updated_at TIMESTAMP DEFAULT current_timestamp()
);

-- Add columns
ALTER TABLE users ADD COLUMN age INT;

-- Create indexes
CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_created ON users (created_at DESC);

-- Show table structure
SHOW CREATE TABLE users;
SHOW COLUMNS FROM users;
SHOW INDEXES FROM users;
```

### Data Manipulation

```sql
-- Insert data
INSERT INTO users (username, email) VALUES 
    ('alice', 'alice@example.com'),
    ('bob', 'bob@example.com');

-- Insert with conflict resolution
INSERT INTO users (username, email) 
VALUES ('alice', 'alice@example.com')
ON CONFLICT (username) DO UPDATE 
SET email = excluded.email, 
    updated_at = current_timestamp();

-- Update data
UPDATE users 
SET age = 25, updated_at = current_timestamp()
WHERE username = 'alice';

-- Delete data
DELETE FROM users WHERE username = 'bob';

-- Bulk operations
IMPORT INTO users (username, email, age)
CSV DATA ('s3://bucket/users.csv');
```

### Querying Data

```sql
-- Basic SELECT
SELECT * FROM users WHERE age > 21;

-- Joins
SELECT u.username, p.title
FROM users u
JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2023-01-01';

-- Aggregations
SELECT 
    age, 
    COUNT(*) as user_count,
    AVG(age) as avg_age
FROM users
GROUP BY age
HAVING COUNT(*) > 1
ORDER BY user_count DESC;

-- Window functions
SELECT 
    username,
    age,
    ROW_NUMBER() OVER (ORDER BY age DESC) as age_rank,
    RANK() OVER (ORDER BY created_at) as join_rank
FROM users;

-- Common Table Expressions (CTEs)
WITH active_users AS (
    SELECT * FROM users 
    WHERE last_login > current_timestamp() - INTERVAL '30 days'
)
SELECT 
    age, 
    COUNT(*) as active_count 
FROM active_users 
GROUP BY age;
```

## SQL Features

### Data Types

```sql
-- Numeric types
CREATE TABLE numeric_types (
    id INT PRIMARY KEY,
    small_int INT2,
    regular_int INT4,
    big_int INT8,
    decimal_num DECIMAL(10,2),
    float_num FLOAT8
);

-- String types
CREATE TABLE string_types (
    id INT PRIMARY KEY,
    fixed_char CHAR(10),
    var_char VARCHAR(255),
    text_field TEXT,
    json_data JSONB
);

-- Date/Time types
CREATE TABLE datetime_types (
    id INT PRIMARY KEY,
    date_only DATE,
    time_only TIME,
    timestamp_tz TIMESTAMPTZ,
    interval_type INTERVAL
);

-- Special types
CREATE TABLE special_types (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    binary_data BYTES,
    ip_address INET,
    array_data INT[],
    bool_flag BOOL
);
```

### Constraints

```sql
-- Primary key constraints
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    order_date DATE,
    PRIMARY KEY (order_id, customer_id)
);

-- Foreign key constraints
CREATE TABLE order_items (
    item_id INT PRIMARY KEY,
    order_id INT,
    customer_id INT,
    FOREIGN KEY (order_id, customer_id) 
        REFERENCES orders (order_id, customer_id)
        ON DELETE CASCADE
);

-- Check constraints
CREATE TABLE products (
    id INT PRIMARY KEY,
    name STRING NOT NULL,
    price DECIMAL(10,2) CHECK (price > 0),
    quantity INT CHECK (quantity >= 0),
    CHECK (price < 10000 OR quantity < 100)
);

-- Unique constraints
CREATE TABLE employees (
    id INT PRIMARY KEY,
    email STRING UNIQUE,
    ssn STRING,
    UNIQUE INDEX idx_ssn (ssn)
);
```

### Transactions

```sql
-- Basic transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Transaction with savepoints
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
SAVEPOINT transfer_start;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Rollback to savepoint if needed
ROLLBACK TO SAVEPOINT transfer_start;
COMMIT;

-- Transaction isolation levels
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- Your operations here
COMMIT;

-- Read-only transactions
BEGIN READ ONLY;
SELECT * FROM accounts WHERE balance > 1000;
COMMIT;
```

### Advanced Features

```sql
-- Computed columns
CREATE TABLE products (
    id INT PRIMARY KEY,
    price DECIMAL(10,2),
    tax_rate DECIMAL(5,4),
    total_price DECIMAL(10,2) AS (price * (1 + tax_rate)) STORED
);

-- Partial indexes
CREATE INDEX idx_active_users 
ON users (username) 
WHERE active = true;

-- Expression indexes
CREATE INDEX idx_lower_email 
ON users (lower(email));

-- Inverted indexes for JSON
CREATE INVERTED INDEX idx_metadata 
ON documents (metadata);

-- Table partitioning
CREATE TABLE events (
    id UUID PRIMARY KEY,
    created_at TIMESTAMPTZ,
    event_type STRING,
    data JSONB
) PARTITION BY RANGE (created_at) (
    PARTITION p2023 VALUES FROM ('2023-01-01') TO ('2024-01-01'),
    PARTITION p2024 VALUES FROM ('2024-01-01') TO ('2025-01-01')
);
```

## Performance Tuning

### Query Optimization

```sql
-- Use EXPLAIN to analyze queries
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';

-- Use EXPLAIN ANALYZE for execution statistics
EXPLAIN ANALYZE SELECT * FROM users WHERE age > 25;

-- View query plan in detail
EXPLAIN (VERBOSE) SELECT * FROM users u 
JOIN posts p ON u.id = p.user_id;

-- Force index usage
SELECT * FROM users@idx_users_email 
WHERE email = 'alice@example.com';

-- Optimize JOIN order
SET optimizer_join_hint_ordering = true;
SELECT /*+ LEADING(u p) */ * 
FROM users u 
JOIN posts p ON u.id = p.user_id;
```

### Index Strategies

```sql
-- Create covering index
CREATE INDEX idx_users_covering 
ON users (age) 
STORING (username, email);

-- Multi-column index for sorting
CREATE INDEX idx_posts_user_date 
ON posts (user_id, created_at DESC);

-- Index for WHERE and ORDER BY
CREATE INDEX idx_products_category_price 
ON products (category, price DESC) 
WHERE active = true;

-- Monitor index usage
SELECT 
    table_name,
    index_name,
    total_reads,
    last_read
FROM crdb_internal.index_usage_statistics
WHERE table_name = 'users'
ORDER BY total_reads DESC;
```

### Connection Pooling

```go
// Go example with connection pooling
import (
    "database/sql"
    _ "github.com/lib/pq"
)

func setupDB() (*sql.DB, error) {
    db, err := sql.Open("postgres", 
        "postgresql://user@localhost:26257/mydb?sslmode=require")
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(100)
    db.SetMaxIdleConns(10)
    db.SetConnMaxLifetime(time.Minute * 5)
    
    return db, nil
}
```

### Batch Operations

```sql
-- Use batch inserts
INSERT INTO events (id, type, data) VALUES
    (gen_random_uuid(), 'click', '{"page": "home"}'),
    (gen_random_uuid(), 'view', '{"page": "products"}'),
    ... -- up to 1000 rows
;

-- Use UPSERT for bulk updates
UPSERT INTO inventory (product_id, quantity, updated_at)
SELECT 
    product_id, 
    new_quantity,
    current_timestamp()
FROM staging_inventory;

-- Batch deletes with limit
DELETE FROM old_logs 
WHERE created_at < '2023-01-01' 
LIMIT 1000;
```

## Multi-Region Deployment

### Regional Configuration

```sql
-- Set up multi-region database
CREATE DATABASE multi_region_db PRIMARY REGION "us-east1" REGIONS "us-west1", "eu-west1";

-- Create regional by row table
CREATE TABLE users (
    id UUID PRIMARY KEY,
    region STRING AS (
        CASE 
            WHEN country IN ('US', 'CA', 'MX') THEN 'us-east1'
            WHEN country IN ('GB', 'DE', 'FR') THEN 'eu-west1'
            ELSE 'us-east1'
        END
    ) STORED,
    country STRING,
    email STRING
) LOCALITY REGIONAL BY ROW AS region;

-- Create global table
CREATE TABLE products (
    id UUID PRIMARY KEY,
    name STRING,
    price DECIMAL
) LOCALITY GLOBAL;

-- Create regional table
CREATE TABLE regional_config (
    region STRING PRIMARY KEY,
    settings JSONB
) LOCALITY REGIONAL BY TABLE IN PRIMARY REGION;
```

### Zone Configuration

```sql
-- Configure replication zones
ALTER DATABASE mydb CONFIGURE ZONE USING 
    num_replicas = 5,
    constraints = '[+region=us-east1:2, +region=us-west1:2, +region=eu-west1:1]',
    lease_preferences = '[[+region=us-east1]]';

-- Table-specific zones
ALTER TABLE critical_data CONFIGURE ZONE USING
    num_replicas = 7,
    gc.ttlseconds = 3600,
    constraints = '[+ssd]';

-- Index-specific zones
ALTER INDEX users@idx_users_email CONFIGURE ZONE USING
    num_replicas = 3,
    constraints = '[+region=us-east1]';
```

### Follow-the-Workload

```sql
-- Enable follow-the-workload
SET CLUSTER SETTING kv.allocator.load_based_lease_rebalancing.enabled = true;

-- Configure lease preferences for tables
ALTER TABLE users CONFIGURE ZONE USING
    lease_preferences = '[[+region=us-east1], [+region=us-west1]]',
    num_replicas = 3;

-- Monitor lease distribution
SELECT 
    range_id,
    lease_holder,
    replicas
FROM crdb_internal.ranges
WHERE table_name = 'users';
```

## Backup and Recovery

### Full Backup

```sql
-- Backup to cloud storage
BACKUP DATABASE mydb 
TO 's3://bucket/backups/mydb?AWS_ACCESS_KEY_ID=...&AWS_SECRET_ACCESS_KEY=...'
WITH revision_history;

-- Backup specific tables
BACKUP TABLE users, posts 
TO 's3://bucket/backups/tables'
AS OF SYSTEM TIME '-10s';

-- Backup with encryption
BACKUP DATABASE mydb 
TO 's3://bucket/backups/mydb'
WITH encryption_passphrase = 'secret';
```

### Incremental Backup

```sql
-- Initial full backup
BACKUP DATABASE mydb 
TO 's3://bucket/backups/mydb-full'
AS OF SYSTEM TIME '-10s';

-- Incremental backup
BACKUP DATABASE mydb 
TO 's3://bucket/backups/mydb-inc1'
INCREMENTAL FROM 's3://bucket/backups/mydb-full'
WITH revision_history;

-- Chain incremental backups
BACKUP DATABASE mydb 
TO 's3://bucket/backups/mydb-inc2'
INCREMENTAL FROM 
    's3://bucket/backups/mydb-full',
    's3://bucket/backups/mydb-inc1';
```

### Restore Operations

```sql
-- Restore full database
RESTORE DATABASE mydb 
FROM 's3://bucket/backups/mydb';

-- Restore to different database
RESTORE DATABASE mydb 
FROM 's3://bucket/backups/mydb'
WITH new_db_name = 'mydb_restored';

-- Point-in-time restore
RESTORE DATABASE mydb 
FROM 's3://bucket/backups/mydb'
AS OF SYSTEM TIME '2024-01-15 10:00:00';

-- Restore specific tables
RESTORE TABLE users, posts 
FROM 's3://bucket/backups/tables'
WITH into_db = 'mydb_restore';
```

### Scheduled Backups

```sql
-- Create backup schedule
CREATE SCHEDULE daily_backup
FOR BACKUP DATABASE mydb 
TO 's3://bucket/backups/scheduled/mydb'
RECURRING '@daily'
WITH SCHEDULE OPTIONS first_run = 'now';

-- Create incremental schedule
CREATE SCHEDULE hourly_incremental
FOR BACKUP DATABASE mydb 
TO 's3://bucket/backups/incremental/mydb'
RECURRING '@hourly'
FULL BACKUP '@daily'
WITH SCHEDULE OPTIONS first_run = 'now';

-- View schedules
SHOW SCHEDULES;

-- Pause/resume schedule
PAUSE SCHEDULE 12345;
RESUME SCHEDULE 12345;
```

## Monitoring and Observability

### Built-in Monitoring

```sql
-- View cluster health
SELECT * FROM crdb_internal.cluster_health;

-- Monitor node status
SELECT 
    node_id,
    address,
    is_available,
    is_live
FROM crdb_internal.gossip_nodes;

-- Check range distribution
SELECT 
    table_name,
    COUNT(*) as range_count,
    SUM(range_size) as total_size
FROM crdb_internal.ranges
GROUP BY table_name
ORDER BY total_size DESC;

-- View active queries
SELECT 
    query,
    start,
    application_name,
    client_address
FROM crdb_internal.cluster_queries
WHERE start < now() - INTERVAL '5 seconds';
```

### Metrics Export

```yaml
# Prometheus configuration
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'cockroachdb'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/_status/vars'
```

### Logging Configuration

```bash
# Start with custom logging
./cockroach start \
  --log-dir=/var/log/cockroach \
  --log-file-verbosity=INFO \
  --log-sql-statements=true \
  --log-slow-queries=2s

# Configure structured logging
./cockroach start \
  --log='{"sinks": {"file-groups": {"default": {"dir": "/logs", "format": "json"}}}}'
```

### Tracing

```sql
-- Enable tracing for session
SET tracing = on;
SELECT * FROM users WHERE id = 123;
SELECT * FROM [SHOW TRACE FOR SESSION];

-- Enable verbose tracing
SET tracing = verbose;
-- Run your query
SET tracing = off;

-- Export traces to Jaeger
SET CLUSTER SETTING trace.jaeger.agent = 'localhost:6831';
```

## Security Best Practices

### Authentication

```bash
# Generate certificates
./cockroach cert create-ca \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key

./cockroach cert create-node \
  localhost \
  node1.example.com \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key

./cockroach cert create-client \
  root \
  --certs-dir=certs \
  --ca-key=my-safe-directory/ca.key
```

### User Management

```sql
-- Create users
CREATE USER app_user WITH PASSWORD 'secure_password';
CREATE USER read_only_user;

-- Grant privileges
GRANT CREATE, SELECT, INSERT, UPDATE, DELETE 
ON DATABASE myapp 
TO app_user;

GRANT SELECT ON TABLE users TO read_only_user;

-- Create roles
CREATE ROLE developers;
GRANT ALL ON DATABASE dev TO developers;
GRANT developers TO alice, bob;

-- Revoke privileges
REVOKE INSERT ON TABLE sensitive_data FROM app_user;
```

### Encryption

```sql
-- Enable encryption at rest
./cockroach start \
  --enterprise-encryption=path=/mnt/data,key=/keys/key1.key,old-key=plain \
  --store=/mnt/data

-- Rotate encryption keys
./cockroach start \
  --enterprise-encryption=path=/mnt/data,key=/keys/key2.key,old-key=/keys/key1.key \
  --store=/mnt/data

-- Use encrypted connections
psql "postgresql://user@host:26257/db?sslmode=require&sslrootcert=ca.crt"
```

### Audit Logging

```sql
-- Enable audit logging
SET CLUSTER SETTING sql.audit.enabled = true;

-- Configure audit logs for specific tables
ALTER TABLE sensitive_data EXPERIMENTAL_AUDIT SET READ WRITE;

-- Configure audit logs for users
ALTER USER suspicious_user EXPERIMENTAL_AUDIT SET ALL;

-- View audit logs
SELECT * FROM crdb_internal.cluster_audit_log 
WHERE "user" = 'suspicious_user'
ORDER BY "timestamp" DESC;
```

## Troubleshooting

### Common Issues

#### Connection Issues
```bash
# Check if CockroachDB is running
ps aux | grep cockroach

# Test connectivity
./cockroach sql --insecure --host=localhost:26257 -e "SELECT 1"

# Check certificates
./cockroach cert list --certs-dir=certs

# Debug connection
./cockroach sql --url="postgresql://user@host:26257/db?sslmode=require" --verbose
```

#### Performance Issues
```sql
-- Identify slow queries
SELECT 
    query,
    execution_count,
    service_latency,
    cpu_time
FROM crdb_internal.statement_statistics
WHERE aggregation_interval = '1h'
ORDER BY service_latency DESC
LIMIT 10;

-- Check for missing indexes
SELECT 
    table_name,
    index_recommendations
FROM crdb_internal.table_indexes
WHERE index_recommendations IS NOT NULL;

-- Analyze table statistics
CREATE STATISTICS stats_users ON id, email FROM users;
SHOW STATISTICS FOR TABLE users;
```

#### Storage Issues
```bash
# Check disk usage
./cockroach node status --insecure

# Clean up old data
./cockroach sql --insecure -e "
  SET CLUSTER SETTING jobs.retention_time = '24h';
  SET CLUSTER SETTING sql.stats.cleanup.recurrence = '@hourly';
"

# Force garbage collection
./cockroach sql --insecure -e "
  ALTER TABLE my_table CONFIGURE ZONE USING gc.ttlseconds = 600;
"
```

### Debug Commands

```bash
# Debug range issues
./cockroach debug range-data ./node1 --range=1

# Check raft status
./cockroach debug raft-log ./node1 --range=1

# Export debug data
./cockroach debug zip --insecure --host=localhost:26257 debug.zip

# Decode keys
./cockroach debug decode-key '\x12\x34\x56'
```

## Best Practices

### Schema Design

1. **Use UUIDs for Primary Keys**
   ```sql
   CREATE TABLE orders (
       id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       ...
   );
   ```

2. **Design for Distribution**
   ```sql
   -- Avoid hotspots with composite keys
   CREATE TABLE events (
       tenant_id UUID,
       event_id UUID,
       PRIMARY KEY (tenant_id, event_id)
   );
   ```

3. **Normalize Carefully**
   - Balance between normalization and join performance
   - Consider denormalization for read-heavy workloads
   - Use JSONB for flexible schema requirements

### Application Patterns

1. **Retry Logic**
   ```python
   import time
   import psycopg2
   from psycopg2 import errorcodes
   
   def execute_with_retry(conn, query, max_retries=3):
       for attempt in range(max_retries):
           try:
               with conn.cursor() as cur:
                   cur.execute(query)
                   conn.commit()
                   return cur.fetchall()
           except psycopg2.Error as e:
               if e.pgcode == errorcodes.SERIALIZATION_FAILURE:
                   conn.rollback()
                   time.sleep(0.1 * (2 ** attempt))
                   continue
               raise
       raise Exception("Max retries exceeded")
   ```

2. **Connection Management**
   - Use connection pooling
   - Set appropriate timeouts
   - Handle connection failures gracefully

3. **Batch Operations**
   - Group related operations in transactions
   - Use bulk inserts for large datasets
   - Implement pagination for large result sets

### Operational Excellence

1. **Regular Maintenance**
   ```sql
   -- Update table statistics
   CREATE STATISTICS ON ALL COLUMNS FROM users;
   
   -- Monitor cluster health
   SELECT * FROM crdb_internal.cluster_health;
   
   -- Clean up old jobs
   SELECT * FROM [SHOW JOBS] WHERE status = 'failed';
   ```

2. **Capacity Planning**
   - Monitor storage growth trends
   - Plan for replica placement
   - Scale before hitting limits

3. **Disaster Recovery**
   - Regular backup testing
   - Document recovery procedures
   - Multi-region deployment for critical data

## Common Use Cases

### E-Commerce Platform

```sql
-- Product catalog (global table)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    description TEXT,
    price DECIMAL(10,2),
    inventory INT DEFAULT 0,
    metadata JSONB
) LOCALITY GLOBAL;

-- Orders (regional by row)
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    region STRING NOT NULL,
    status STRING DEFAULT 'pending',
    total DECIMAL(10,2),
    created_at TIMESTAMPTZ DEFAULT current_timestamp(),
    FOREIGN KEY (user_id) REFERENCES users(id)
) LOCALITY REGIONAL BY ROW AS region;

-- Shopping cart with TTL
CREATE TABLE cart_items (
    user_id UUID,
    product_id UUID,
    quantity INT,
    added_at TIMESTAMPTZ DEFAULT current_timestamp(),
    PRIMARY KEY (user_id, product_id)
) WITH (ttl_expiration_expression = 'added_at + INTERVAL ''7 days''');
```

### Time-Series Data

```sql
-- Partitioned time-series table
CREATE TABLE metrics (
    timestamp TIMESTAMPTZ,
    device_id UUID,
    metric_name STRING,
    value FLOAT,
    tags JSONB,
    PRIMARY KEY (device_id, timestamp, metric_name)
) PARTITION BY RANGE (timestamp) (
    PARTITION p_2024_01 VALUES FROM ('2024-01-01') TO ('2024-02-01'),
    PARTITION p_2024_02 VALUES FROM ('2024-02-01') TO ('2024-03-01')
);

-- Efficient queries with time bounds
SELECT 
    device_id,
    avg(value) as avg_value,
    max(value) as max_value
FROM metrics
WHERE timestamp >= now() - INTERVAL '1 hour'
    AND metric_name = 'temperature'
GROUP BY device_id;
```

### Multi-Tenant SaaS

```sql
-- Tenant isolation with row-level security
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name STRING NOT NULL,
    plan STRING DEFAULT 'free',
    settings JSONB
);

-- Tenant data with composite primary key
CREATE TABLE tenant_data (
    tenant_id UUID,
    id UUID DEFAULT gen_random_uuid(),
    data JSONB,
    created_at TIMESTAMPTZ DEFAULT current_timestamp(),
    PRIMARY KEY (tenant_id, id),
    FOREIGN KEY (tenant_id) REFERENCES tenants(id)
);

-- Efficient tenant queries
CREATE INDEX idx_tenant_data_created 
ON tenant_data (tenant_id, created_at DESC);
```

## Conclusion

This usage guide covers the essential aspects of working with CockroachDB. Key points to remember:

1. **Start Simple**: Begin with single-region deployments and expand as needed
2. **Design for Distribution**: Consider data locality and access patterns
3. **Monitor Actively**: Use built-in tools and external monitoring
4. **Plan for Scale**: Design schemas and applications for horizontal scaling
5. **Test Thoroughly**: Include failure scenarios in your testing

For more detailed information, refer to the official CockroachDB documentation and join the community forums for support and best practices sharing.
