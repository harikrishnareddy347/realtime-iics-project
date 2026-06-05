# Parameterization Guide - Zurich Insurance Project

## Overview

Parameterization enables dynamic configuration of mappings and taskflows, making them reusable and flexible.

## Table of Contents

1. [Parameter Basics](#parameter-basics)
2. [Parameter Types](#parameter-types)
3. [Parameter Configuration](#parameter-configuration)
4. [Using Parameters](#using-parameters)
5. [Parameter Passing](#parameter-passing)
6. [Best Practices](#best-practices)

## Parameter Basics

### What is a Parameter?

A parameter is a variable that can be set at runtime to change the behavior of mappings or taskflows without modifying the underlying code.

**Benefits**:
- ✅ Reusability across environments (Dev, Test, Prod)
- ✅ Dynamic configuration without code changes
- ✅ Flexibility for different data scenarios
- ✅ Easy maintenance and updates
- ✅ Better security (sensitive values in config)

---

## Parameter Types

### 1. Mapping Parameters

**Used within mapping transformations**

**Example**: Database connection parameters

```yaml
parameter:
  name: "param_db_host"
  type: "STRING"
  data_type: "VARCHAR(100)"
  default_value: "localhost"
  description: "Database host address"
```

### 2. Mapping Task Parameters

**Used to control mapping task execution**

```yaml
parameter:
  name: "param_filter_date"
  type: "DATE"
  data_type: "DATE"
  default_value: "2024-01-01"
  required: true
  description: "Filter records from this date onwards"
```

### 3. Taskflow Parameters

**Used at taskflow level**

```yaml
parameter:
  name: "param_environment"
  type: "STRING"
  data_type: "VARCHAR(50)"
  allowed_values: ["DEV", "TEST", "PROD"]
  default_value: "DEV"
  description: "Target environment"
```

### 4. Session Variables

**Dynamic variables during execution**

```yaml
session_variable:
  name: "$sesStartTime"
  type: "DATETIME"
  description: "Session start timestamp"
```

### 5. Global Variables

**Shared across taskflows**

```yaml
global_variable:
  name: "$g_batch_date"
  type: "DATE"
  description: "Current batch processing date"
  scope: "GLOBAL"
```

---

## Parameter Configuration

### 1. Define Parameters in Mapping

**Step 1**: Create parameter definition

```yaml
mapping:
  name: "MAP_CUSTOMER_LOAD"
  parameters:
    - name: "pm_source_table"
      type: "STRING"
      data_type: "VARCHAR(100)"
      default_value: "CUSTOMER_SOURCE"
    
    - name: "pm_target_table"
      type: "STRING"
      data_type: "VARCHAR(100)"
      default_value: "CUSTOMER_TARGET"
    
    - name: "pm_batch_size"
      type: "INTEGER"
      data_type: "INTEGER"
      default_value: 1000
    
    - name: "pm_filter_status"
      type: "STRING"
      data_type: "VARCHAR(20)"
      default_value: "ACTIVE"
      allowed_values: ["ACTIVE", "INACTIVE", "PENDING"]
```

**Step 2**: Use parameters in mapping expressions

```iics
// In source query
SELECT * FROM $pm_source_table 
WHERE status = '$pm_filter_status'

// In filter transform
status = '$pm_filter_status'

// In target query
INSERT INTO $pm_target_table
```

---

### 2. Create Mapping Task with Parameters

```yaml
mapping_task:
  name: "TASK_CUSTOMER_LOAD"
  mapping: "MAP_CUSTOMER_LOAD"
  parameters:
    pm_source_table: "CUSTOMER_SOURCE"
    pm_target_table: "CUSTOMER_TARGET"
    pm_batch_size: 1000
    pm_filter_status: "ACTIVE"
  
  # Parameterized session configuration
  session_config:
    rows_per_commit: "$pm_batch_size"
    buffer_memory: "256M"
    sort_memory: "512M"
```

---

### 3. Taskflow Parameter Definition

```yaml
taskflow:
  name: "TF_INSURANCE_LOAD_PARAMETERIZED"
  description: "Parameterized insurance data load"
  
  parameters:
    - name: "pf_processing_date"
      type: "DATE"
      data_type: "DATE"
      required: true
      description: "Date to process"
    
    - name: "pf_environment"
      type: "STRING"
      data_type: "VARCHAR(10)"
      allowed_values: ["DEV", "TEST", "PROD"]
      default_value: "DEV"
    
    - name: "pf_load_mode"
      type: "STRING"
      data_type: "VARCHAR(20)"
      allowed_values: ["FULL", "INCREMENTAL", "DELTA"]
      default_value: "INCREMENTAL"
    
    - name: "pf_notification_email"
      type: "STRING"
      data_type: "VARCHAR(255)"
      description: "Email for notifications"
  
  tasks:
    - task_id: 1
      name: "Load_Customers"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_LOAD"
      parameters:
        pm_filter_status: "ACTIVE"
        pm_batch_size: 5000
      depends_on: []
    
    - task_id: 2
      name: "Load_Policies"
      type: "Mapping_Task"
      mapping: "MAP_POLICY_LOAD"
      parameters:
        pm_processing_date: "$pf_processing_date"
        pm_load_mode: "$pf_load_mode"
      depends_on: []
    
    - task_id: 3
      name: "Send_Completion_Email"
      type: "Script_Task"
      script: "send_email.sh"
      parameters:
        recipient: "$pf_notification_email"
        environment: "$pf_environment"
      depends_on: [1, 2]
```

---

## Using Parameters

### 1. Using Parameters in Mapping Expressions

**Scenario**: Filter customers by date range

```yaml
mapping:
  name: "MAP_CUSTOMER_BY_DATE"
  parameters:
    - name: "pm_start_date"
      type: "DATE"
      required: true
    - name: "pm_end_date"
      type: "DATE"
      required: true
  
  transforms:
    - name: "Filter_By_Date"
      type: "Filter"
      expression: "creation_date >= '$pm_start_date' AND creation_date <= '$pm_end_date'"
```

### 2. Using Parameters in Conditional Logic

**Scenario**: Apply different calculations based on parameter

```iics
IIF('$pf_load_mode' = 'FULL',
    // Full load: reset all data
    TRUNCATE_TARGET = true,
    // Incremental: append only new records
    TRUNCATE_TARGET = false
)
```

### 3. Using Parameters in Lookups

**Scenario**: Lookup from different tables based on parameter

```iics
Lookup(
  IIF('$pm_lookup_source' = 'CACHED', 'POLICY_TYPE_CACHE', 'POLICY_TYPE_LOOKUP'),
  'policy_type_code',
  policy_type_code
)
```

---

## Parameter Passing

### 1. Command Line Parameter Passing

```bash
# Run mapping task with parameters
iics run -mapping MAP_CUSTOMER_LOAD \
  -param pm_source_table="CUSTOMER_SOURCE" \
  -param pm_target_table="CUSTOMER_TARGET" \
  -param pm_filter_status="ACTIVE"

# Run taskflow with parameters
iics run -taskflow TF_INSURANCE_LOAD \
  -param pf_processing_date="2024-05-26" \
  -param pf_environment="PROD" \
  -param pf_load_mode="INCREMENTAL"
```

### 2. Configuration File Parameter Passing

**File**: `config/parameter_config.json`

```json
{
  "environments": {
    "dev": {
      "pm_source_table": "CUSTOMER_SOURCE_DEV",
      "pm_target_table": "CUSTOMER_TARGET_DEV",
      "pm_db_host": "dev-db.internal",
      "pm_db_port": 5432
    },
    "test": {
      "pm_source_table": "CUSTOMER_SOURCE_TEST",
      "pm_target_table": "CUSTOMER_TARGET_TEST",
      "pm_db_host": "test-db.internal",
      "pm_db_port": 5432
    },
    "prod": {
      "pm_source_table": "CUSTOMER_SOURCE",
      "pm_target_table": "CUSTOMER_TARGET",
      "pm_db_host": "prod-db.internal",
      "pm_db_port": 5432
    }
  }
}
```

### 3. Environment Variable Parameter Passing

```bash
# Set environment variables
export PM_SOURCE_TABLE="CUSTOMER_SOURCE"
export PM_TARGET_TABLE="CUSTOMER_TARGET"
export PM_FILTER_STATUS="ACTIVE"

# Run with environment variables
iics run -taskflow TF_CUSTOMER_LOAD \
  -param pm_source_table="$PM_SOURCE_TABLE" \
  -param pm_target_table="$PM_TARGET_TABLE" \
  -param pm_filter_status="$PM_FILTER_STATUS"
```

### 4. Programmatic Parameter Passing (API)

```json
{
  "taskflow_run": {
    "taskflow_name": "TF_INSURANCE_LOAD",
    "parameters": {
      "pf_processing_date": "2024-05-26",
      "pf_environment": "PROD",
      "pf_load_mode": "INCREMENTAL",
      "pf_notification_email": "ops@zurich.com"
    },
    "runtime_options": {
      "thread_count": 4,
      "memory_allocation": "2GB"
    }
  }
}
```

---

## Best Practices

### 1. Parameter Naming

✅ **DO**:
- Use prefixes: `pm_` for mapping, `pf_` for taskflow, `ps_` for session
- Use descriptive names: `pm_source_database_host`
- Use snake_case: `pm_filter_status`
- Version parameters: `pm_lookup_source_v2`

❌ **DON'T**:
- Use generic names: `param1`, `value`
- Mix naming conventions: `pm_SourceTable` and `pm_target_table`
- Use special characters: `pm-source-table`
- Abbreviate excessively: `pm_src_db_h`

### 2. Parameter Validation

```yaml
parameter:
  name: "pm_batch_size"
  type: "INTEGER"
  validation:
    min_value: 1
    max_value: 10000
    pattern: "^[0-9]+$"
  error_message: "Batch size must be between 1 and 10000"
```

### 3. Default Values

✅ **Best Practices**:
- Always provide sensible defaults
- Test default values thoroughly
- Document default value choices
- Update defaults when requirements change

### 4. Documentation

```yaml
parameter:
  name: "pm_processing_date"
  type: "DATE"
  data_type: "DATE"
  required: true
  default_value: "NULL"
  allowed_values: "Any valid date"
  description: "The date to process customer records from"
  example_value: "2024-05-26"
  constraints: "Must be after 2020-01-01"
  modified_by: "Data Team"
  modified_date: "2024-05-20"
```

---

## Practice Exercises

1. Create a parameterized mapping with 3 parameters
2. Create a mapping task that accepts parameters
3. Build a taskflow with 5 parameters for different environments
4. Implement conditional logic based on parameters
5. Pass parameters via command line and configuration file

---

## Next Steps

1. Study `ERROR_HANDLING_GUIDE.md` for robust parameterization
2. Review `RUNTIME_OPTIONS_GUIDE.md` for dynamic execution
