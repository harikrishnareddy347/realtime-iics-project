# Error Handling Guide - Zurich Insurance Project

## Overview

Robust error handling is critical for production systems. This guide covers error detection, handling, and recovery strategies.

## Table of Contents

1. [Error Types](#error-types)
2. [Exception Handling](#exception-handling)
3. [Retry Strategies](#retry-strategies)
4. [Recovery Mechanisms](#recovery-mechanisms)
5. [Monitoring and Alerts](#monitoring-and-alerts)
6. [Best Practices](#best-practices)

## Error Types

### 1. Data Quality Errors

**Examples**:
- Null values in required fields
- Invalid data types
- Out-of-range values
- Duplicate keys

**Handling Strategy**:
```yaml
error_handler:
  error_type: "DATA_QUALITY"
  detection:
    - check: "NOT ISNULL(customer_id)"
      action: "REDIRECT_TO_ERROR_TABLE"
    - check: "ISNUMBER(premium_amount)"
      action: "LOG_AND_SKIP"
    - check: "premium_amount > 0"
      action: "SET_TO_DEFAULT"
      default_value: 0
  error_table: "CUSTOMER_ERRORS"
```

### 2. System Errors

**Examples**:
- Database connection failures
- Out of memory
- File not found
- Network timeouts

**Handling Strategy**:
```yaml
error_handler:
  error_type: "SYSTEM"
  retry:
    enabled: true
    max_attempts: 3
    backoff_strategy: "EXPONENTIAL"
    initial_delay_ms: 1000
    max_delay_ms: 60000
  on_failure: "HALT_PROCESS"
  notification:
    email: "admin@zurich.com"
    severity: "CRITICAL"
```

### 3. Business Logic Errors

**Examples**:
- Invalid customer type
- Invalid policy status transition
- Premium calculation outside expected range

**Handling Strategy**:
```yaml
error_handler:
  error_type: "BUSINESS_LOGIC"
  validation:
    - rule: "customer_type IN ('CORPORATE', 'INDIVIDUAL', 'BROKER')"
      action: "REJECT_RECORD"
    - rule: "annual_premium > 0 AND annual_premium < 1000000"
      action: "LOG_WARNING"
  quarantine_table: "CUSTOMER_VALIDATION_ERRORS"
```

---

## Exception Handling

### 1. Try-Catch Pattern in Mappings

**Scenario**: Safe data type conversion

```iics
// Attempt to convert amount to number
// If fails, set to 0
IIF(ISNUMBER(amount_str),
    TO_NUMBER(amount_str),
    IIF(ISNULL(amount_str), 0, 0)  // Default to 0 on error
)
```

### 2. Null Handling

```iics
// Handle multiple null scenarios
IIF(ISNULL(customer_id),
    -1,  // Special value for missing customer
    IIF(ISNULL(policy_amount),
        0,  // Special value for missing amount
        customer_id * policy_amount
    )
)
```

### 3. Range Validation

```iics
// Validate value within acceptable range
IIF(premium_amount >= 0 AND premium_amount <= 1000000,
    premium_amount,
    NULL  // Mark as invalid
)
```

### 4. Format Validation

```iics
// Validate email format
IIF(
    REGEX_MATCH(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$'),
    email,
    'INVALID_EMAIL'
)
```

---

## Retry Strategies

### 1. Exponential Backoff Retry

```yaml
retry_strategy:
  name: "EXPONENTIAL_BACKOFF"
  config:
    max_attempts: 5
    initial_delay_ms: 1000
    max_delay_ms: 60000
    backoff_factor: 2
    jitter_enabled: true
  
  # Attempt schedule
  # Attempt 1: immediate
  # Attempt 2: 1 second + jitter
  # Attempt 3: 2 seconds + jitter
  # Attempt 4: 4 seconds + jitter
  # Attempt 5: 8 seconds + jitter
```

### 2. Linear Backoff Retry

```yaml
retry_strategy:
  name: "LINEAR_BACKOFF"
  config:
    max_attempts: 3
    initial_delay_ms: 5000
    increment_delay_ms: 5000
  
  # Attempt schedule
  # Attempt 1: immediate
  # Attempt 2: 5 seconds
  # Attempt 3: 10 seconds
```

### 3. Circuit Breaker Pattern

```yaml
circuit_breaker:
  name: "DATABASE_CONNECTION_CB"
  failure_threshold: 5  # Fail after 5 consecutive errors
  success_threshold: 2  # Close after 2 successful attempts
  timeout_ms: 30000     # Open for 30 seconds
  states:
    - state: "CLOSED"
      description: "Normal operation"
      transition: "to OPEN on 5 consecutive failures"
    - state: "OPEN"
      description: "Circuit broken, requests rejected"
      transition: "to HALF_OPEN after timeout"
    - state: "HALF_OPEN"
      description: "Testing if service recovered"
      transition: "to CLOSED on success, to OPEN on failure"
```

---

## Recovery Mechanisms

### 1. Checkpoint and Restart

```yaml
checkpoint:
  enabled: true
  interval: "1000_ROWS"  # Create checkpoint every 1000 rows
  storage_location: "/var/iics/checkpoints/customer_load/"
  recovery_on_failure:
    enabled: true
    resume_from_last_checkpoint: true
    skip_processed_records: true
```

**Usage**:
```bash
# Run normally
iics run -taskflow TF_CUSTOMER_LOAD

# If failed, resume from checkpoint
iics run -taskflow TF_CUSTOMER_LOAD -recovery true
```

### 2. Rollback Strategy

```yaml
rollback:
  strategy: "TRANSACTION_BASED"
  scope: "PER_BATCH"
  on_error: "ROLLBACK_CURRENT_BATCH"
  
  # Alternatively:
  # strategy: "MANUAL"
  # rollback_script: "rollback_customer_load.sql"
  # run_on_failure: true
```

### 3. Compensating Transactions

```yaml
compensating_transaction:
  name: "ROLLBACK_CUSTOMER_INSERT"
  trigger: "ON_MAPPING_FAILURE"
  script: |
    DELETE FROM CUSTOMER_TARGET 
    WHERE creation_date = TRUNC(SYSDATE) 
    AND batch_id = '$g_batch_id'
  execute_before_restart: true
```

---

## Monitoring and Alerts

### 1. Error Logging

```yaml
error_logging:
  enabled: true
  log_level: "ERROR"
  log_destination:
    - "FILE:/var/logs/iics/errors.log"
    - "DATABASE:ERROR_LOG_TABLE"
    - "SYSLOG:insurance_errors"
  
  log_format: "JSON"
  log_fields:
    - timestamp
    - error_code
    - error_message
    - error_source
    - task_name
    - record_id
    - stack_trace
```

### 2. Alert Configuration

```yaml
alerts:
  - alert_id: 1
    name: "TASK_FAILURE"
    trigger: "Task execution failed"
    severity: "CRITICAL"
    notification:
      email: ["ops@zurich.com", "insurance-team@zurich.com"]
      slack: "#insurance-alerts"
      pagerduty: true
  
  - alert_id: 2
    name: "HIGH_ERROR_RATE"
    trigger: "Error rate > 5%"
    severity: "WARNING"
    notification:
      email: ["ops@zurich.com"]
      slack: "#insurance-alerts"
  
  - alert_id: 3
    name: "DATA_QUALITY_ISSUE"
    trigger: "Quality check failed"
    severity: "WARNING"
    notification:
      email: ["data-quality@zurich.com"]
```

### 3. Metrics and Dashboard

```yaml
metrics:
  - metric: "task_execution_time"
    threshold: "300 seconds"
    alert_on_threshold: true
  
  - metric: "error_count_per_task"
    threshold: "10 errors"
    alert_on_threshold: true
  
  - metric: "data_quality_score"
    threshold: "95%"
    alert_below_threshold: true
  
  - metric: "success_rate"
    threshold: "99%"
    alert_below_threshold: true
```

---

## Best Practices

### 1. Comprehensive Error Handling

✅ **DO**:
- Handle all anticipated errors
- Log detailed error information
- Implement multiple recovery strategies
- Test error scenarios thoroughly
- Document error handling logic

❌ **DON'T**:
- Ignore errors silently
- Use generic error messages
- Forget to clean up resources on error
- Skip testing error paths
- Assume errors won't happen

### 2. Logging Best Practices

```iics
// Log before critical operations
LogMessage("Starting customer extraction from: $pm_source_table", "INFO")

// Log errors with context
LogMessage(
    "Error processing customer " || customer_id || ": " || error_message,
    "ERROR"
)

// Log completion status
LogMessage("Completed processing " || processed_count || " records", "INFO")
```

### 3. Test Error Scenarios

✅ **Test Cases**:
- Null value scenarios
- Invalid data types
- Out-of-range values
- Duplicate records
- Connection failures
- Timeout scenarios
- Partial failures
- Full failures

### 4. Error Documentation

```yaml
error_documentation:
  - error_code: "ERR_001"
    error_message: "Customer ID is null"
    cause: "Source data missing required field"
    resolution: "Validate source data quality"
    prevention: "Implement pre-load validation"
  
  - error_code: "ERR_002"
    error_message: "Database connection failed"
    cause: "Network issue or database down"
    resolution: "Check database connectivity and retry"
    prevention: "Implement connection pooling and heartbeat"
```

---

## Practice Exercises

1. Implement null handling in a mapping
2. Create a retry strategy with exponential backoff
3. Implement a checkpoint and restart mechanism
4. Design a comprehensive error logging system
5. Create alert configurations for different error types

---

## Next Steps

1. Study `RUNTIME_OPTIONS_GUIDE.md` for dynamic execution control
2. Review configuration files in `config/` directory
3. Explore example implementations in `error_handling/` directory
