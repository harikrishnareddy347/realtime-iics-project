# Error Handling Example: Retry & Recovery Strategies

## Overview
Comprehensive error handling patterns including retries, checkpointing, and rollback strategies.

## Scenario: Customer Load with Error Recovery

## Error Handling Strategy 1: Exponential Backoff Retry

### Configuration

```yaml
Retry Strategy: EXPONENTIAL_BACKOFF
Max Attempts: 5
Initial Delay: 1 second
Max Delay: 60 seconds
Backoff Factor: 2
Jitter: Enabled (±10%)
```

### Attempt Schedule

```
Attempt 1: Immediate (0 seconds)
Attempt 2: 1 second + jitter
Attempt 3: 2 seconds + jitter
Attempt 4: 4 seconds + jitter
Attempt 5: 8 seconds + jitter

Total Max Wait Time: ~15 seconds
```

### When to Use
- Database connection failures
- Network timeouts
- Temporary resource unavailability
- Transient system errors

### Example Implementation

```json
{
  "error_handling": {
    "retry": {
      "enabled": true,
      "strategy": "EXPONENTIAL_BACKOFF",
      "max_attempts": 5,
      "initial_delay_ms": 1000,
      "max_delay_ms": 60000,
      "backoff_factor": 2,
      "jitter_enabled": true
    }
  }
}
```

---

## Error Handling Strategy 2: Checkpoint & Resume

### Configuration

```yaml
Checkpoint Strategy: ROW_BASED
Interval: Every 5000 rows
Storage: /var/iics/checkpoints/customer_load/
Compression: GZIP
Cleanup: Delete on success
Resume Policy: Resume from last checkpoint on failure
```

### How It Works

```
Processing Records:
  0 - 5000 rows: Write checkpoint 1
  5000 - 10000 rows: Write checkpoint 2
  10000 - 15000 rows: FAIL at row 12345

Recovery:
  On restart: Resume from checkpoint 2 (row 10000)
  Skip rows 0-10000 (already processed)
  Continue from row 10001 to completion
```

### Checkpoint Content

```
Checkpoint_ID: CP_001
Timestamp: 2024-05-26 14:35:22
Task: MAP_CUSTOMER_LOAD
Rows Processed: 10000
Last Row ID: CUST_10000
Session State: Serialized session variables
Source Position: Input stream offset
Target Position: Last commit ID
Status: SUCCESSFULLY_COMPLETED
```

### Implementation

```iics
// In mapping expression
IF CHECKPOINT_EXISTS() THEN
  RESUME_FROM_CHECKPOINT()
ELSE
  START_NEW_PROCESSING()
END IF

// Create checkpoint every 5000 rows
IF MOD(ROW_COUNT, 5000) = 0 THEN
  CREATE_CHECKPOINT('CP_' || ROW_COUNT)
END IF
```

---

## Error Handling Strategy 3: Compensating Transactions

### Scenario: Transaction Rollback on Failure

```
Normal Flow:
  1. Lock resources
  2. Extract data
  3. Transform data
  4. Load to staging
  5. Validate
  6. Load to production
  7. Update metadata
  8. Unlock resources
  9. SUCCESS

Error Flow (at step 5):
  1. Validation fails
  2. Execute compensation tasks:
     - DELETE from production (undo step 6)
     - DELETE from staging (undo step 4)
     - Restore metadata (undo step 7)
     - Release locks (undo step 8)
  3. FAIL and rollback complete
```

### Compensating Transaction Script

```sql
-- Compensation Script: rollback_customer_load.sql

BEGIN TRANSACTION;

-- Undo production load
DELETE FROM CUSTOMER_TARGET 
WHERE creation_date = TRUNC(SYSDATE) 
AND batch_id = ?batch_id;

-- Undo staging data
DELETE FROM CUSTOMER_STAGING 
WHERE batch_id = ?batch_id;

-- Update metadata
UPDATE LOAD_METADATA
SET status = 'FAILED',
    failure_reason = ?error_message,
    rolled_back_timestamp = SYSTIMESTAMP
WHERE batch_id = ?batch_id;

-- Release locks
DELETE FROM RESOURCE_LOCKS
WHERE batch_id = ?batch_id;

COMMIT TRANSACTION;
```

### Configuration

```yaml
Compensating Transaction:
  Name: ROLLBACK_CUSTOMER_LOAD
  Trigger: ON_TASK_FAILURE
  Script: scripts/rollback_customer_load.sql
  Execute: BEFORE_RETRY (automatic rollback)
  Log: All compensation actions
```

---

## Error Handling Strategy 4: Error Routing

### Configuration

```yaml
Error Routing:
  Valid Records: CUSTOMER_VALID → CUSTOMER_TARGET
  Invalid Records: CUSTOMER_INVALID → ERROR_LOG
  Duplicate Records: CUSTOMER_DUPLICATES → DUP_LOG
  Null Records: CUSTOMER_NULLS → NULL_LOG
```

### Filter Expressions

```iics
// Route 1: Valid Records
FILTER: NOT ISNULL(cust_id) AND 
        LENGTH(email) > 0 AND 
        LENGTH(phone_clean) = 10

// Route 2: Invalid Records
FILTER: ISNULL(cust_id) OR 
        LENGTH(email) = 0

// Route 3: Duplicate Detection
FILTER: ROW_NUMBER() OVER (PARTITION BY cust_id ORDER BY creation_date DESC) > 1
```

---

## Error Handling Strategy 5: Data Quality Gates

### Pre-Load Validation

```sql
-- Check 1: Source data exists
SELECT COUNT(*) FROM CUSTOMER_SOURCE
MUST BE > 0

-- Check 2: Target table writable
SELECT 1 FROM CUSTOMER_TARGET WHERE 1=0
MUST SUCCEED

-- Check 3: Storage available
SELECT free_space FROM DISK_USAGE
WHERE mount_point = '/data'
MUST BE > 1GB
```

### Post-Load Validation

```sql
-- Check 1: Row count matches
SELECT COUNT(*) FROM CUSTOMER_TARGET
WHERE creation_date = TRUNC(SYSDATE)
MUST EQUAL source_row_count

-- Check 2: Data quality score
SELECT data_quality_score FROM LOAD_QUALITY
WHERE batch_id = ?batch_id
MUST BE >= 95

-- Check 3: Reconciliation
SELECT CASE WHEN target_count = source_count 
             THEN 'PASS' 
             ELSE 'FAIL' 
        END
FROM RECONCILIATION_VIEW
```

---

## Error Handling Strategy 6: Circuit Breaker Pattern

### States and Transitions

```
CLOSED (Normal)
  ├─ Success: Stay CLOSED
  └─ 5 consecutive failures: Go to OPEN

OPEN (Broken)
  ├─ After 30 seconds: Go to HALF_OPEN
  └─ New requests: REJECTED immediately

HALF_OPEN (Testing)
  ├─ Success on test: Go to CLOSED
  └─ Failure on test: Go to OPEN
```

### Configuration

```yaml
Circuit Breaker:
  Name: DATABASE_CONNECTION_CB
  Failure Threshold: 5 consecutive errors
  Success Threshold: 2 successful calls
  Timeout: 30 seconds
  Monitored Resource: Database Connection
  
  Actions:
    On Open: Log alert, notify ops
    On Half-Open: Test with single connection
    On Close: Resume normal operations
```

---

## Error Handling Strategy 7: Null Handling

### Expression Examples

```iics
// Safe Addition
IIF(ISNULL(amount1) OR ISNULL(amount2), 
    NULL, 
    amount1 + amount2)

// Default Value
IIF(ISNULL(country), 'USA', country)

// Conditional Default
IIF(ISNULL(end_date), 
    ADD_MONTHS(start_date, 12), 
    end_date)

// Chain Multiple Defaults
COALESCE(primary_email, 
         secondary_email, 
         'no-email@company.com')
```

---

## Comprehensive Error Handling Example

```yaml
Mapping: MAP_CUSTOMER_WITH_FULL_ERROR_HANDLING

Error Handling Configuration:
  
  1. Pre-Execution:
     - Check source connectivity
     - Verify table accessibility
     
  2. During Execution:
     - Null handling in expressions
     - Type conversion with defaults
     - Data validation checks
     
  3. Retry Strategy:
     - Exponential backoff: 5 attempts
     - Delay: 1s to 60s
     - Log each attempt
     
  4. Checkpointing:
     - Every 5000 rows
     - Resume capability
     
  5. Error Routing:
     - Valid → CUSTOMER_VALID
     - Invalid → CUSTOMER_INVALID
     - Errors → ERROR_LOG
     
  6. Post-Execution:
     - Validation checks
     - Reconciliation
     - Compensation if needed
     
  7. Notifications:
     - On failure: Alert ops
     - On success: Log completion
     - SLA tracking
```

---

## Practice Exercise

Implement error handling for:
1. **Policy Load with retry and checkpointing**
2. **Claims processing with compensation transactions**
3. **Data validation with error routing**
4. **Multi-step process with circuit breaker pattern**
5. **Complex null handling scenarios**
