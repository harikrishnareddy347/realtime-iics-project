# Runtime Options Guide - Zurich Insurance Project

## Overview

Runtime options control how mappings and taskflows execute, providing fine-grained control over performance, resource allocation, and behavior.

## Table of Contents

1. [Mapping Runtime Options](#mapping-runtime-options)
2. [Task Runtime Options](#task-runtime-options)
3. [Taskflow Runtime Options](#taskflow-runtime-options)
4. [Resource Management](#resource-management)
5. [Execution Policies](#execution-policies)
6. [Best Practices](#best-practices)

## Mapping Runtime Options

### 1. Session Configuration

```yaml
mapping_runtime:
  name: "MAP_CUSTOMER_LOAD"
  session_config:
    # Connection pooling
    connection_pool_size: 5
    connection_timeout_ms: 30000
    
    # Performance tuning
    rows_per_commit: 10000
    buffer_memory_mb: 256
    sort_memory_mb: 512
    pipeline_buffer_mb: 128
    
    # Parallel processing
    pipeline_parallelism: 4
    partition_type: "HASH"
    partition_key: "customer_id"
    
    # Logging
    trace_level: "NORMAL"  # MINIMAL, NORMAL, VERBOSE, DEBUG
    log_performance_metrics: true
    log_row_counts: true
    
    # Behavior
    fail_on_error: true
    continue_on_lookup_error: false
    sort_order: "ASC"
```

### 2. Source Optimization

```yaml
source_options:
  source_name: "CUSTOMER_SOURCE"
  
  # Data extraction
  fetch_size: 10000
  fetch_strategy: "PARALLEL"  # SEQUENTIAL, PARALLEL
  parallel_connections: 4
  
  # Query optimization
  push_down_filter: true
  push_down_projection: true
  
  # Caching
  enable_caching: true
  cache_size_mb: 100
  cache_ttl_seconds: 300
  
  # Sampling (for testing)
  sampling_enabled: false
  sample_percentage: 1  # 1% for testing
```

### 3. Target Optimization

```yaml
target_options:
  target_name: "CUSTOMER_TARGET"
  
  # Write strategy
  write_mode: "APPEND"  # APPEND, REPLACE, UPDATE
  write_batch_size: 5000
  parallel_write_connections: 4
  
  # Transaction control
  commit_type: "AUTO_COMMIT"  # AUTO_COMMIT, MANUAL_COMMIT
  transaction_isolation_level: "READ_COMMITTED"
  
  # Constraint handling
  disable_constraints_on_load: true
  disable_triggers_on_load: true
  enable_constraints_after_load: true
  
  # Error handling
  truncate_on_error: false
  store_rejected_records: true
  rejected_records_table: "CUSTOMER_TARGET_ERRORS"
```

---

## Task Runtime Options

### 1. Mapping Task Configuration

```yaml
mapping_task_runtime:
  task_name: "TASK_CUSTOMER_LOAD"
  mapping: "MAP_CUSTOMER_LOAD"
  
  # Execution strategy
  execution_mode: "STANDALONE"  # STANDALONE, CLUSTER
  execution_environment: "SECURE_AGENT"
  secure_agent_group: "insurance_agents"
  
  # Performance
  max_parallel_instances: 2
  timeout_minutes: 60
  
  # Resource allocation
  memory_allocation_mb: 1024
  cpu_cores: 2
  
  # Monitoring
  monitor_performance: true
  collect_performance_stats: true
  alert_on_slowdown: true
  slowdown_threshold_seconds: 300
  
  # Logging
  debug_mode: false
  collect_lineage_data: true
```

### 2. Task Scheduling

```yaml
task_scheduling:
  task_name: "TASK_CUSTOMER_LOAD"
  
  # Trigger configuration
  trigger_type: "SCHEDULED"  # SCHEDULED, EVENT_BASED, MANUAL
  
  # Cron expression for scheduling
  cron_schedule: "0 2 * * *"  # 2 AM daily
  
  # Time zone
  schedule_timezone: "America/New_York"
  
  # Window configuration
  success_window_minutes: 120  # Must complete within 2 hours
  recovery_window_days: 7      # Can be recovered within 7 days
```

---

## Taskflow Runtime Options

### 1. Execution Strategy

```yaml
taskflow_runtime:
  taskflow_name: "TF_INSURANCE_LOAD"
  
  # Execution mode
  execution_mode: "SEQUENTIAL"  # SEQUENTIAL, PARALLEL, ADAPTIVE
  
  # Task sequencing
  task_execution_order:
    - "Load_Customers"   # Task 1
    - "Load_Policies"    # Task 2 (parallel if allowed)
    - "Load_Claims"      # Task 3 (parallel if allowed)
    - "Consolidate"      # Task 4 (depends on 1-3)
  
  # Error handling
  on_task_error: "STOP"  # STOP, SKIP, CONTINUE
  max_failures_allowed: 1
  failure_handling_strategy: "ROLLBACK_ALL"  # ROLLBACK_ALL, ROLLBACK_TASK, CONTINUE
  
  # Timeout
  overall_timeout_minutes: 180
  task_timeout_minutes: 60
```

### 2. Parallelization Configuration

```yaml
parallelization:
  taskflow_name: "TF_INSURANCE_LOAD"
  
  # Parallel execution
  enable_parallel_tasks: true
  max_parallel_tasks: 3
  parallel_task_groups:
    - group_id: 1
      tasks: ["Load_Customers", "Load_Policies"]
      max_parallel: 2
    - group_id: 2
      tasks: ["Load_Claims"]
      max_parallel: 1
  
  # Resource coordination
  shared_resources:
    - resource: "database_connection_pool"
      max_connections: 10
    - resource: "file_system"
      max_concurrent_accesses: 5
```

### 3. Conditional Execution

```yaml
conditional_execution:
  - task_name: "Load_Policies"
    condition: "previous_task_status == 'SUCCESS'"
    skip_if_condition_false: false  # Execute alternative
    alternative_task: "Log_Warning"
  
  - task_name: "Consolidate"
    condition: "all_tasks_completed == true"
    skip_if_condition_false: true  # Skip if condition false
```

---

## Resource Management

### 1. Memory Management

```yaml
resource_management:
  memory:
    # Heap memory
    heap_memory_mb: 2048
    heap_memory_initial_mb: 512
    heap_memory_max_mb: 2048
    
    # Off-heap memory
    off_heap_memory_mb: 512
    
    # Buffer management
    buffer_pool_size_mb: 256
    sort_memory_mb: 512
    pipeline_buffer_mb: 128
    
    # Garbage collection
    gc_strategy: "G1GC"  # CMS, G1GC, ZGC
    gc_pause_target_ms: 200
```

### 2. CPU Management

```yaml
resource_management:
  cpu:
    # Core allocation
    allocated_cores: 4
    thread_count: 4
    
    # Thread pool
    thread_pool_size: 16
    queue_capacity: 1000
    
    # CPU affinity
    enable_cpu_affinity: true
    cpu_set: "0-3"  # Use cores 0-3
```

### 3. Disk I/O Management

```yaml
resource_management:
  disk_io:
    # Cache configuration
    enable_disk_cache: true
    disk_cache_location: "/var/iics/cache"
    disk_cache_size_mb: 1000
    
    # I/O optimization
    io_batch_size: 10000
    enable_compression: true
    compression_level: 6  # 1-9
    
    # Spill to disk
    enable_spill: true
    spill_location: "/var/iics/spill"
    spill_threshold_mb: 1000
```

---

## Execution Policies

### 1. Concurrency Policy

```yaml
execution_policies:
  concurrency_policy:
    name: "CONTROLLED_CONCURRENCY"
    max_concurrent_jobs: 5
    max_concurrent_per_type:
      mapping_tasks: 3
      taskflows: 2
    queue_priority:
      - priority: "HIGH"
        max_queue_wait_minutes: 5
      - priority: "NORMAL"
        max_queue_wait_minutes: 15
      - priority: "LOW"
        max_queue_wait_minutes: 60
```

### 2. SLA Policy

```yaml
execution_policies:
  sla_policy:
    name: "INSURANCE_DATA_SLA"
    
    objectives:
      - objective: "Daily customer load completes by 3 AM"
        metric: "execution_time"
        threshold_minutes: 120
        priority: "CRITICAL"
      
      - objective: "99.9% success rate"
        metric: "success_rate"
        threshold_percentage: 99.9
        priority: "CRITICAL"
      
      - objective: "Data freshness within 1 hour"
        metric: "data_lag"
        threshold_minutes: 60
        priority: "HIGH"
    
    # Enforcement
    enforcement_action: "AUTO_ROLLBACK"  # AUTO_ROLLBACK, ALERT, MANUAL
```

### 3. Data Quality Policy

```yaml
execution_policies:
  data_quality_policy:
    name: "INSURANCE_DATA_QUALITY"
    
    quality_gates:
      - gate: "COMPLETENESS_CHECK"
        rule: "null_count <= 1%"
        action_on_failure: "FAIL_TASK"
      
      - gate: "ACCURACY_CHECK"
        rule: "validation_error_rate <= 0.5%"
        action_on_failure: "LOG_WARNING"
      
      - gate: "UNIQUENESS_CHECK"
        rule: "duplicate_count == 0"
        action_on_failure: "FAIL_TASK"
      
      - gate: "CONSISTENCY_CHECK"
        rule: "orphaned_records == 0"
        action_on_failure: "QUARANTINE_RECORDS"
```

---

## Best Practices

### 1. Performance Tuning

✅ **DO**:
- Start with default settings
- Monitor actual performance
- Tune based on metrics
- Test changes in non-prod first
- Document all changes
- Review quarterly

❌ **DON'T**:
- Over-allocate resources
- Use aggressive caching without validation
- Change multiple settings simultaneously
- Forget to measure impact
- Leave debug settings enabled in production

### 2. Resource Allocation

```yaml
resource_allocation_guidelines:
  small_job:
    memory_mb: 512
    cpu_cores: 1
    timeout_minutes: 15
  
  medium_job:
    memory_mb: 1024
    cpu_cores: 2
    timeout_minutes: 60
  
  large_job:
    memory_mb: 2048
    cpu_cores: 4
    timeout_minutes: 180
```

### 3. Monitoring and Tuning

```yaml
monitoring_metrics:
  - metric: "cpu_utilization"
    optimal_range: "50-80%"
    alert_if_above: "90%"
    alert_if_below: "20%"
  
  - metric: "memory_utilization"
    optimal_range: "60-85%"
    alert_if_above: "95%"
    alert_if_below: "40%"
  
  - metric: "execution_time"
    optimal_range: "< baseline * 1.2"
    alert_if_exceeds: "baseline * 1.5"
  
  - metric: "success_rate"
    optimal_target: "99.9%"
    alert_if_below: "99%"
```

---

## Practice Exercises

1. Configure runtime options for a mapping task
2. Create a taskflow with parallel execution
3. Implement resource allocation policies
4. Set up SLA and monitoring
5. Tune performance based on metrics

---

## Next Steps

1. Review configuration files in `config/` directory
2. Explore example implementations in `runtime_options/` directory
3. Study `ERROR_HANDLING_GUIDE.md` for integrated approach
