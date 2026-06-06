# Parameterization Example: Dynamic Configuration

## Overview
Demonstrates how to use parameters to make taskflows flexible and reusable across environments.

## Mapping Task: MAP_CUSTOMER_WITH_PARAMETERS

### Parameter Definitions

#### 1. Source Table Parameter
```yaml
Name: pm_source_table
Type: STRING
Data Type: VARCHAR(100)
Default Value: CUSTOMER_SOURCE
Required: true
Description: Source table name
Environment Override:
  DEV: CUSTOMER_SOURCE_DEV
  TEST: CUSTOMER_SOURCE_TEST
  PROD: CUSTOMER_SOURCE
```

#### 2. Target Table Parameter
```yaml
Name: pm_target_table
Type: STRING
Data Type: VARCHAR(100)
Default Value: CUSTOMER_TARGET
Required: true
Description: Target table name
Environment Override:
  DEV: CUSTOMER_TARGET_DEV
  TEST: CUSTOMER_TARGET_TEST
  PROD: CUSTOMER_TARGET
```

#### 3. Batch Size Parameter
```yaml
Name: pm_batch_size
Type: INTEGER
Data Type: INTEGER
Default Value: 10000
Required: false
Min Value: 1
Max Value: 50000
Description: Rows per commit
Recommendation: Larger for prod (20000), smaller for dev (5000)
```

#### 4. Filter Status Parameter
```yaml
Name: pm_filter_status
Type: STRING
Data Type: VARCHAR(20)
Default Value: ACTIVE
Allowed Values: [ACTIVE, INACTIVE, PENDING, ALL]
Required: false
Description: Filter records by status
```

#### 5. Date Range Parameters
```yaml
Name: pm_start_date
Type: DATE
Data Type: DATE
Default Value: NULL
Required: false
Description: Filter from this date
Format: YYYY-MM-DD

Name: pm_end_date
Type: DATE
Data Type: DATE
Default Value: NULL
Required: false
Description: Filter to this date
Format: YYYY-MM-DD
```

---

## Using Parameters in Mapping

### Example 1: Dynamic Table Names in Source Query

```sql
-- Traditional (Hard-coded)
SELECT * FROM CUSTOMER_SOURCE WHERE status = 'ACTIVE'

-- Parameterized
SELECT * FROM ${pm_source_table} WHERE status = '${pm_filter_status}'
```

### Example 2: Conditional Date Filtering

```iics
-- In Filter Transform
IIF(
  ISNULL('${pm_start_date}'),
  1=1,  -- No date filter
  creation_date >= TO_DATE('${pm_start_date}', 'YYYY-MM-DD')
)
```

### Example 3: Dynamic Batch Size in Session

```
Session Configuration:
- Rows Per Commit: ${pm_batch_size}
- Buffer Memory: ${IIF(${pm_batch_size} > 20000, '512MB', '256MB')}
```

---

## Taskflow Parameters

### Parameter Definitions

```yaml
Taskflow: TF_INSURANCE_LOAD_PARAMETERIZED

Parameters:
  - Name: pf_processing_date
    Type: DATE
    Required: true
    Description: Date to process
    
  - Name: pf_environment
    Type: STRING
    Allowed Values: [DEV, TEST, PROD]
    Default Value: DEV
    Description: Target environment
    
  - Name: pf_load_mode
    Type: STRING
    Allowed Values: [FULL, INCREMENTAL, DELTA]
    Default Value: INCREMENTAL
    Description: Load strategy
    
  - Name: pf_batch_size
    Type: INTEGER
    Min: 1000
    Max: 50000
    Default: 10000
    Description: Records per batch
    
  - Name: pf_notify_email
    Type: STRING
    Required: false
    Default: ops@zurich.com
    Description: Alert recipient
```

---

## Parameter Passing Methods

### Method 1: Command Line

```bash
# Single execution
iics-cli run-taskflow \
  --taskflow TF_INSURANCE_LOAD \
  --param pf_processing_date="2024-05-26" \
  --param pf_environment="PROD" \
  --param pf_load_mode="INCREMENTAL" \
  --param pf_batch_size="20000"

# Scheduled execution (cron)
0 2 * * * iics-cli run-taskflow TF_INSURANCE_LOAD \
  --param pf_processing_date="$(date +%Y-%m-%d)" \
  --param pf_environment="PROD"
```

### Method 2: Configuration File

**File**: config/load_parameters.json
```json
{
  "dev": {
    "pf_environment": "DEV",
    "pm_source_table": "CUSTOMER_SOURCE_DEV",
    "pm_target_table": "CUSTOMER_TARGET_DEV",
    "pm_batch_size": 5000,
    "pf_notify_email": "dev-team@zurich.com"
  },
  "test": {
    "pf_environment": "TEST",
    "pm_source_table": "CUSTOMER_SOURCE_TEST",
    "pm_target_table": "CUSTOMER_TARGET_TEST",
    "pm_batch_size": 10000,
    "pf_notify_email": "qa-team@zurich.com"
  },
  "prod": {
    "pf_environment": "PROD",
    "pm_source_table": "CUSTOMER_SOURCE",
    "pm_target_table": "CUSTOMER_TARGET",
    "pm_batch_size": 20000,
    "pf_notify_email": "ops@zurich.com"
  }
}
```

**Usage**:
```bash
# Read from config file
source config/load_parameters.json

# Run with config parameters
iics-cli run-taskflow TF_INSURANCE_LOAD \
  --param pf_environment="${ENVIRONMENT}" \
  --params-file config/load_parameters.json
```

### Method 3: Environment Variables

```bash
# Set environment variables
export PM_SOURCE_TABLE="CUSTOMER_SOURCE"
export PM_TARGET_TABLE="CUSTOMER_TARGET"
export PM_BATCH_SIZE="20000"
export PF_ENVIRONMENT="PROD"
export PF_PROCESSING_DATE="$(date +%Y-%m-%d)"

# Run with environment variables
iics-cli run-taskflow TF_INSURANCE_LOAD \
  --param pm_source_table="$PM_SOURCE_TABLE" \
  --param pm_target_table="$PM_TARGET_TABLE" \
  --param pm_batch_size="$PM_BATCH_SIZE" \
  --param pf_environment="$PF_ENVIRONMENT" \
  --param pf_processing_date="$PF_PROCESSING_DATE"
```

### Method 4: API Call

```json
POST /api/v1/taskflows/TF_INSURANCE_LOAD/run

{
  "parameters": {
    "pf_processing_date": "2024-05-26",
    "pf_environment": "PROD",
    "pf_load_mode": "INCREMENTAL",
    "pm_source_table": "CUSTOMER_SOURCE",
    "pm_target_table": "CUSTOMER_TARGET",
    "pm_batch_size": 20000,
    "pf_notify_email": "ops@zurich.com"
  },
  "runtime_options": {
    "max_parallel_tasks": 3,
    "memory_allocation_mb": 2048
  }
}
```

---

## Parameter Validation

### Built-in Validation

```yaml
Parameter: pm_batch_size
Validation:
  - Type Check: INTEGER
  - Min Value: 1000 (must be > 1000)
  - Max Value: 50000 (must be < 50000)
  - Not Null: true
  Error Handling: FAIL on validation error
```

### Custom Validation

```iics
-- In Pre-Load Script
IF '${pf_processing_date}' > CURRENT_DATE() THEN
  RAISE EXCEPTION 'Processing date cannot be in future'
END IF

IF ${pm_batch_size} < 1000 THEN
  RAISE EXCEPTION 'Batch size must be at least 1000'
END IF
```

---

## Environment-Specific Configuration

### DEV Environment
```
pm_source_table: CUSTOMER_SOURCE_DEV
pm_target_table: CUSTOMER_TARGET_DEV
pm_batch_size: 5000
pf_environment: DEV
Retry: 3 times
Timeout: 30 minutes
```

### TEST Environment
```
pm_source_table: CUSTOMER_SOURCE_TEST
pm_target_table: CUSTOMER_TARGET_TEST
pm_batch_size: 10000
pf_environment: TEST
Retry: 3 times
Timeout: 60 minutes
```

### PROD Environment
```
pm_source_table: CUSTOMER_SOURCE
pm_target_table: CUSTOMER_TARGET
pm_batch_size: 20000
pf_environment: PROD
Retry: 2 times
Timeout: 90 minutes
```

---

## Practice Exercise

Create parameterized mappings for:
1. **Policy Load Mapping**
   - Parameter for source database
   - Parameter for lookup strategy
   - Parameter for premium threshold
   - Parameter for date range

2. **Claims Processing Taskflow**
   - Parameter for claim types to process
   - Parameter for approval threshold
   - Parameter for batch processing window
   - Parameter for notification recipients

3. **Incremental Load Taskflow**
   - Parameter for load mode (FULL/INCREMENTAL)
   - Parameter for CDC strategy
   - Parameter for lookback period
   - Parameter for target partition
