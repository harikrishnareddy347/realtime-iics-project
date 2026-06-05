# Taskflow Guide - Zurich Insurance Project

## Overview

Taskflows orchestrate mappings and tasks into complete ETL processes. This guide covers taskflow creation and management.

## Table of Contents

1. [Basic Taskflows](#basic-taskflows)
2. [Parallel Execution](#parallel-execution)
3. [Conditional Logic](#conditional-logic)
4. [Error Handling in Taskflows](#error-handling-in-taskflows)
5. [Best Practices](#best-practices)

## Basic Taskflows

### 1. Sequential Taskflow

**Scenario**: Customer data load process

**Taskflow Steps**:
1. Extract customer data from source
2. Transform customer data
3. Validate customer data
4. Load to target database
5. Log completion

**Taskflow Definition**:
```yaml
taskflow:
  name: "TF_CUSTOMER_LOAD_BASIC"
  description: "Basic sequential customer data loading"
  tasks:
    - task_id: 1
      name: "Extract_Customer_Data"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_EXTRACT"
      status: "Pending"
    
    - task_id: 2
      name: "Transform_Customer"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_TRANSFORM"
      depends_on: [1]
      status: "Pending"
    
    - task_id: 3
      name: "Validate_Customer"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_VALIDATE"
      depends_on: [2]
      status: "Pending"
    
    - task_id: 4
      name: "Load_Customer"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_LOAD"
      depends_on: [3]
      status: "Pending"
    
    - task_id: 5
      name: "Log_Completion"
      type: "Script_Task"
      script: "log_process_completion.sh"
      depends_on: [4]
      status: "Pending"
```

**Execution Flow**:
```
1 → 2 → 3 → 4 → 5
```

---

### 2. Parallel Taskflow

**Scenario**: Load customer, policy, and claims data simultaneously

**Taskflow Definition**:
```yaml
taskflow:
  name: "TF_INSURANCE_DATA_LOAD_PARALLEL"
  description: "Parallel loading of multiple data entities"
  tasks:
    - task_id: 1
      name: "Extract_All_Data"
      type: "Script_Task"
      depends_on: []
      status: "Pending"
    
    - task_id: 2
      name: "Load_Customer"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_LOAD"
      depends_on: [1]
      status: "Pending"
    
    - task_id: 3
      name: "Load_Policy"
      type: "Mapping_Task"
      mapping: "MAP_POLICY_LOAD"
      depends_on: [1]
      status: "Pending"
    
    - task_id: 4
      name: "Load_Claims"
      type: "Mapping_Task"
      mapping: "MAP_CLAIMS_LOAD"
      depends_on: [1]
      status: "Pending"
    
    - task_id: 5
      name: "Consolidate"
      type: "Script_Task"
      depends_on: [2, 3, 4]
      status: "Pending"
```

**Execution Flow**:
```
       ┌── 2 ──┐
1 ──→ ┤── 3 ──┤── 5
       └── 4 ──┘
```

---

## Parallel Execution

### Designing Parallel Tasks

✅ **Good Candidates for Parallelization**:
- Independent data extractions
- Different target loads
- Non-overlapping transformations
- Read-only operations

❌ **NOT Suitable for Parallelization**:
- Tasks with mutual dependencies
- Tasks updating same table
- Tasks requiring sequential consistency
- Database locking scenarios

---

## Conditional Logic

### 1. Decision Tree in Taskflow

**Scenario**: Premium calculation based on customer type

**Taskflow with Decisions**:
```yaml
taskflow:
  name: "TF_PREMIUM_CALC_CONDITIONAL"
  description: "Premium calculation with conditional routing"
  tasks:
    - task_id: 1
      name: "Load_Customer_Data"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_LOAD"
      status: "Pending"
    
    - task_id: 2
      name: "Classify_Customer"
      type: "Mapping_Task"
      mapping: "MAP_CUSTOMER_CLASSIFY"
      depends_on: [1]
      status: "Pending"
      outputs:
        - variable: "customer_type"
          type: "STRING"
    
    - task_id: 3
      name: "Decision_Customer_Type"
      type: "Decision"
      depends_on: [2]
      conditions:
        - condition: "customer_type == 'CORPORATE'"
          next_task: 4
        - condition: "customer_type == 'INDIVIDUAL'"
          next_task: 5
        - condition: "customer_type == 'BROKER'"
          next_task: 6
        - default: 7
    
    - task_id: 4
      name: "Calc_Corporate_Premium"
      type: "Mapping_Task"
      mapping: "MAP_CALC_CORPORATE_PREMIUM"
      depends_on: [3]
      status: "Pending"
    
    - task_id: 5
      name: "Calc_Individual_Premium"
      type: "Mapping_Task"
      mapping: "MAP_CALC_INDIVIDUAL_PREMIUM"
      depends_on: [3]
      status: "Pending"
    
    - task_id: 6
      name: "Calc_Broker_Premium"
      type: "Mapping_Task"
      mapping: "MAP_CALC_BROKER_PREMIUM"
      depends_on: [3]
      status: "Pending"
    
    - task_id: 7
      name: "Log_Invalid_Type"
      type: "Script_Task"
      depends_on: [3]
      status: "Pending"
```

**Execution Flow**:
```
1 → 2 → 3 ──┬→ 4
             ├→ 5
             ├→ 6
             └→ 7
```

---

## Error Handling in Taskflows

### 1. Task-Level Error Handling

```yaml
task:
  name: "Load_Customer"
  type: "Mapping_Task"
  mapping: "MAP_CUSTOMER_LOAD"
  error_handling:
    on_error: "STOP_TASKFLOW"  # or "SKIP_TASK" or "CONTINUE"
    retry:
      enabled: true
      max_attempts: 3
      delay_seconds: 60
    email_on_error: "admin@company.com"
```

### 2. Taskflow-Level Error Handling

```yaml
taskflow:
  name: "TF_CUSTOMER_LOAD"
  error_handling:
    mode: "FAIL_FAST"  # or "CONTINUE_ON_ERROR"
    notification:
      on_failure: true
      email: "ops@company.com"
      slack: "#insurance-alerts"
    recovery_tasks:
      - name: "Rollback_Changes"
        trigger: "on_failure"
      - name: "Send_Alert"
        trigger: "on_failure"
```

---

## Best Practices

### 1. Taskflow Design

✅ **DO**:
- Keep taskflows focused on single business process
- Use meaningful names with prefixes: `TF_`, `WF_`
- Document dependencies clearly
- Version your taskflows
- Implement proper error handling
- Add logging at key steps

❌ **DON'T**:
- Create overly complex taskflows
- Skip error handling
- Hard-code values
- Ignore performance implications
- Skip validation steps

### 2. Performance Optimization

✅ **Best Practices**:
- Maximize parallelization where possible
- Minimize data intermediate storage
- Use efficient lookup strategies
- Pre-aggregate data when possible
- Cache reference data

### 3. Logging and Monitoring

```yaml
taskflow:
  name: "TF_CUSTOMER_LOAD"
  monitoring:
    log_level: "INFO"
    track_row_counts: true
    track_execution_time: true
    alerting:
      slow_task_threshold_seconds: 300
      high_error_rate_threshold: 5
```

---

## Practice Exercises

1. Create a simple sequential taskflow (3 tasks)
2. Design a parallel taskflow with 4 independent tasks
3. Implement conditional routing based on a variable
4. Add error handling with retry logic
5. Create a complex taskflow combining parallel and conditional logic

---

## Next Steps

1. Study `PARAMETERIZATION_GUIDE.md` for dynamic taskflows
2. Review `ERROR_HANDLING_GUIDE.md` for advanced error strategies
3. Learn `RUNTIME_OPTIONS_GUIDE.md` for execution control
