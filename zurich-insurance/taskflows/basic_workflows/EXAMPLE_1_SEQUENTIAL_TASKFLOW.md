# Sequential Taskflow Example: Customer Daily Load

## Overview
A complete end-to-end sequential taskflow that loads customer data from source to target with validations.

## Taskflow: TF_CUSTOMER_DAILY_LOAD

## Execution Flow

```
Task 1: Extract Customer Source
    ↓
Task 2: Transform Customer Data
    ↓
Task 3: Validate Customer Data
    ↓
Task 4: Load to Target
    ↓
Task 5: Generate Report
```

## Task Details

### Task 1: Extract Customer Source
**Type**: Mapping Task  
**Mapping**: MAP_CUSTOMER_EXTRACT  
**Purpose**: Extract customer data from source system  
**Timeout**: 30 minutes  
**Retry**: Yes, 2 attempts  

**Input**:
- Legacy system CUSTOMER_SOURCE table

**Output**:
- Extracted customer records to staging table

**Success Criteria**:
- row_count > 0
- No connection errors

---

### Task 2: Transform Customer Data
**Type**: Mapping Task  
**Mapping**: MAP_CUSTOMER_TRANSFORM  
**Purpose**: Apply business logic and enrichment  
**Depends On**: Task 1 (must complete successfully)  
**Timeout**: 45 minutes  
**Pass Row Count**: From Task 1  

**Input**:
- Extracted customer records

**Transformations**:
- Filter by status
- Clean phone numbers
- Validate emails
- Segment customers
- Add defaults

**Output**:
- Transformed customer records

**Success Criteria**:
- All transformations applied
- No data quality issues

---

### Task 3: Validate Customer Data
**Type**: Mapping Task  
**Mapping**: MAP_CUSTOMER_VALIDATE  
**Purpose**: Quality gate before loading  
**Depends On**: Task 2  
**Timeout**: 30 minutes  

**Input**:
- Transformed records

**Validation Checks**:
- Mandatory fields
- Data types
- Format validation
- Range validation

**Output**:
- Valid records → CUSTOMER_VALID
- Invalid records → CUSTOMER_INVALID

**Success Criteria**:
- Error rate < 1%
- All records validated

---

### Task 4: Load to Target
**Type**: Mapping Task  
**Mapping**: MAP_CUSTOMER_LOAD  
**Purpose**: Load validated data to production  
**Depends On**: Task 3  
**Timeout**: 60 minutes  

**Input**:
- Valid customer records

**Load Options**:
- Write Mode: APPEND
- Commit Interval: 10000 rows
- Error Handling: STOP_ON_ERROR

**Output**:
- Records loaded to CUSTOMER_TARGET
- Load statistics captured

**Success Criteria**:
- All valid records loaded
- No connection errors

---

### Task 5: Generate Report
**Type**: Script Task  
**Script**: generate_daily_report.sh  
**Purpose**: Create execution summary  
**Depends On**: Task 4  

**Parameters**:
- process_date: $(date +%Y-%m-%d)
- source_count: Extract row count
- valid_count: Valid records count
- invalid_count: Invalid records count
- loaded_count: Successfully loaded count

**Output**:
- Daily report to report directory
- Email notification to team

**Success Criteria**:
- Report generated successfully
- Email sent

---

## Error Handling

### Task Level
```
On Error: STOP_TASKFLOW
Retry: Up to 2 times with 60 second delay
Email: ops@zurich.com on failure
```

### Taskflow Level
```
Mode: FAIL_FAST
On Failure: Stop all tasks, send alerts
Notify: ops@zurich.com, insurance-team@zurich.com
Logs: Collect and archive
```

---

## Monitoring & Metrics

### Track Metrics
- Execution time per task
- Row counts at each stage
- Data quality scores
- Error rates

### Alerts
- Task exceeds timeout: Alert
- Error rate > 1%: Alert
- Load failure: Critical alert
- Slow execution (> 2x baseline): Warning

---

## Pre/Post Execution

### Pre-Execution Checks
1. Database connectivity
2. Source data freshness (< 24 hours old)
3. Target database writable
4. Required table space available

### Post-Execution Actions
1. Update metadata table
2. Archive logs
3. Send completion email
4. Publish metrics

---

## Parameters

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| processing_date | DATE | TODAY | Date to process |
| environment | STRING | DEV | Target env |
| batch_size | INTEGER | 10000 | Rows per commit |
| notify_email | STRING | ops@zurich.com | Alert recipient |

---

## Practice Exercise

Extend this taskflow to:
1. Add a pre-load reconciliation task
2. Include multiple customer sources with parallel loading
3. Add data quality scoring task
4. Create alternate path for error records
5. Add post-load validation task
6. Implement checkpoint and resume capability
