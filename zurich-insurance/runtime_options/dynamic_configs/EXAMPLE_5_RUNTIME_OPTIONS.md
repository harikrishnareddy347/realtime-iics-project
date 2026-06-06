# Runtime Options Example: Performance Tuning

## Overview
Configure runtime options for optimal performance based on data volume and environment.

## Session Configuration for Different Scenarios

### Scenario 1: Small Data Load (< 100K rows)

```yaml
Session Configuration - DEV/TEST:
  
Performance Settings:
  - Rows Per Commit: 5000
  - Buffer Memory: 256 MB
  - Sort Memory: 256 MB
  - Pipeline Buffers: 64 MB
  - Trace Level: VERBOSE
  
Pipeline:
  - Parallelism: 2
  - Thread Count: 4
  - Queue Size: 100
  
Connections:
  - Pool Size: 3
  - Timeout: 30 seconds
  
Caching:
  - Lookup Cache: ENABLED
  - Cache Size: 50 MB
  - Cache TTL: 300 seconds
```

### Scenario 2: Medium Data Load (100K - 1M rows)

```yaml
Session Configuration - TEST/PROD:
  
Performance Settings:
  - Rows Per Commit: 10000
  - Buffer Memory: 512 MB
  - Sort Memory: 512 MB
  - Pipeline Buffers: 128 MB
  - Trace Level: NORMAL
  
Pipeline:
  - Parallelism: 4
  - Thread Count: 8
  - Queue Size: 500
  
Connections:
  - Pool Size: 5
  - Timeout: 60 seconds
  
Caching:
  - Lookup Cache: ENABLED (SHARED)
  - Cache Size: 200 MB
  - Cache TTL: 600 seconds
```

### Scenario 3: Large Data Load (> 1M rows)

```yaml
Session Configuration - PROD:
  
Performance Settings:
  - Rows Per Commit: 20000
  - Buffer Memory: 1024 MB
  - Sort Memory: 1024 MB
  - Pipeline Buffers: 256 MB
  - Trace Level: NORMAL (minimal logging)
  
Pipeline:
  - Parallelism: 8
  - Thread Count: 16
  - Queue Size: 1000
  
Connections:
  - Pool Size: 10
  - Timeout: 120 seconds
  
Caching:
  - Lookup Cache: ENABLED (PERSISTENT)
  - Cache Size: 500 MB
  - Cache TTL: 1800 seconds
```

---

## Source Optimization

### Query Optimization

```yaml
Source: CUSTOMER_SOURCE
Optimization:
  - Push Down Filter: YES
  - Push Down Projection: YES
  - Fetch Strategy: PARALLEL
  - Parallel Connections: 4
  - Fetch Size: 10000
  
Query Tuning:
  - Pre-fetch Statistics: YES
  - Create Source Index: YES
  - Use Parallel Read: YES
```

### Example Source Query

```sql
-- Without optimization
SELECT * FROM CUSTOMER_SOURCE

-- With optimization
SELECT cust_id, fname, lname, email, phone
FROM CUSTOMER_SOURCE
WHERE status = 'ACTIVE'
  AND creation_date >= TRUNC(SYSDATE) - 30
ORDER BY cust_id
```

---

## Target Optimization

### Write Strategy

```yaml
Target: CUSTOMER_TARGET

Write Configuration:
  - Write Mode: APPEND
  - Write Batch Size: 5000
  - Parallel Write Connections: 4
  - Auto-commit: YES
  - Commit Type: BATCH
  
Constraint Handling:
  - Disable Constraints: YES (on load)
  - Enable Constraints: YES (after load)
  - Disable Triggers: YES
  - Enable Triggers: YES
  
Index Management:
  - Disable Indexes: YES (on load)
  - Rebuild Indexes: YES (after load)
  - Rebuild Statistics: YES
```

---

## Memory Management

### Memory Allocation Strategy

```yaml
JVM Configuration:
  Heap Memory:
    - Initial: 512 MB
    - Maximum: 2048 MB
    - Growth: Dynamic (100 MB increments)
  
  Off-Heap Memory: 512 MB
  
Buffer Allocation:
  - Input Buffer: 256 MB
  - Output Buffer: 256 MB
  - Sort Buffer: 512 MB
  - Pipeline Buffer: 128 MB
  
Garbage Collection:
  - Strategy: G1GC (for 2GB+ heap)
  - Pause Target: 200 ms
  - Full GC Threshold: 80% heap
```

### Memory Monitoring

```
Monitor Points:
  - Heap Usage: Alert if > 80%
  - GC Frequency: Alert if > 5 per minute
  - GC Pause Time: Alert if > 500 ms
  - Memory Leak: Alert if gradual increase
```

---

## CPU & Threading Configuration

### Thread Pool Configuration

```yaml
Thread Management:
  Core Threads: 4
  Max Threads: 16
  Queue Capacity: 1000
  Keep-Alive Time: 60 seconds
  
CPU Affinity:
  Enabled: YES (PROD only)
  CPU Set: "0-7" (use cores 0-7)
  
Load Balancing:
  Strategy: WORK_STEALING
  Priority: NORMAL
```

### Parallel Execution Settings

```yaml
Mapping Task Parallelism:
  - Pipeline Parallelism: 4
  - Data Partition: HASH (by cust_id)
  - Min Partition Size: 10000 rows
  
Taskflow Parallelism:
  - Max Concurrent Tasks: 3
  - Resource Reservation: YES
  - Priority Scheduling: YES
```

---

## Disk I/O Optimization

### Spill Configuration

```yaml
Spill Strategy:
  Enabled: YES
  Location: /var/iics/spill/
  Threshold: 1000 MB (switch to disk)
  Compression: GZIP (level 6)
  
Sort Performance:
  - Use In-Memory: Primary
  - Spill to Disk: Fallback
  - Merge Strategy: Multi-way merge
```

### Cache Configuration

```yaml
Lookup Cache:
  Type: SHARED (across instances)
  Location: /var/iics/cache/
  Size: 500 MB (PROD) / 100 MB (DEV)
  TTL: 1800 seconds
  Eviction: LRU (Least Recently Used)
  
Source Cache:
  Enabled: YES
  Type: ROW_CACHE
  Size: 100 MB
  Persistence: Memory only
```

---

## Performance Monitoring Configuration

### Metrics Collection

```yaml
Metrics:
  - Task Execution Time
  - Row Processing Rate (rows/sec)
  - Memory Utilization (%)
  - CPU Utilization (%)
  - Disk I/O Rate (MB/sec)
  - Network Throughput (MB/sec)
  
Collection Interval: 10 seconds
History Retention: 30 days
```

### Alert Configuration

```yaml
Alert Thresholds:
  
  Execution Time:
    Warning: Task > Baseline × 1.2
    Critical: Task > Baseline × 1.5
  
  Memory:
    Warning: Heap > 70%
    Critical: Heap > 85%
  
  Error Rate:
    Warning: > 0.5%
    Critical: > 2%
  
  Data Quality:
    Warning: Score < 90%
    Critical: Score < 80%
```

---

## SLA Configuration

### Service Level Agreements

```yaml
Daily Load SLA:
  Objective: Complete by 3 AM
  Target Time: 120 minutes
  Alert Threshold: 100 minutes (80%)
  Critical: > 150 minutes
  
Data Freshness SLA:
  Objective: < 1 hour lag
  Alert: > 45 minutes
  Critical: > 1.5 hours
  
Quality SLA:
  Objective: > 99% accuracy
  Alert: < 99%
  Critical: < 98%
```

---

## Resource Allocation Policy

### Environment-Based Policy

```yaml
DEV Environment:
  Memory Limit: 512 MB per task
  CPU Cores: 1-2
  Max Concurrent: 1
  Timeout: 30 minutes
  Priority: LOW
  
TEST Environment:
  Memory Limit: 1024 MB per task
  CPU Cores: 2
  Max Concurrent: 2
  Timeout: 60 minutes
  Priority: MEDIUM
  
PROD Environment:
  Memory Limit: 2048 MB per task
  CPU Cores: 4
  Max Concurrent: 3
  Timeout: 90 minutes
  Priority: HIGH
```

---

## Performance Tuning Checklist

```
Pre-Execution:
  ✓ Verify available memory
  ✓ Check CPU utilization
  ✓ Confirm disk space
  ✓ Review source data statistics
  
During Execution:
  ✓ Monitor heap usage
  ✓ Track GC frequency
  ✓ Observe thread utilization
  ✓ Check I/O patterns
  
Post-Execution:
  ✓ Compare vs baseline
  ✓ Identify bottlenecks
  ✓ Analyze logs
  ✓ Tune for next run
```

---

## Practice Exercise

Configure runtime options for:
1. **Policy load handling 2M records**
2. **Real-time customer sync with low latency requirement**
3. **Historical claims archive with minimal resource usage**
4. **Concurrent multi-source ingestion**
5. **Peak hour vs off-peak configurations**
