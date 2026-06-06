# Example 3: Customer Data Validation

## Overview
Validate customer data quality with multiple checks and error routing.

## Source Data
Transformed customer data from previous examples

## Validation Rules

### Rule 1: Mandatory Fields Check
```
Condition: NOT ISNULL(cust_id) AND 
           NOT ISNULL(fname) AND 
           NOT ISNULL(email_address) AND 
           NOT ISNULL(phone_clean)

Purpose: Ensure all required fields have values
Action: REJECT if any field is null
```

### Rule 2: Data Type Validation
```
Conditions:
- ISNUMBER(cust_id) - Customer ID must be numeric
- cust_id > 0 - Customer ID must be positive
- LENGTH(fname) > 0 AND LENGTH(fname) <= 100 - Name length validation
- LENGTH(email_address) > 0 AND LENGTH(email_address) <= 255

Purpose: Verify data types and ranges
Action: REJECT if validation fails
```

### Rule 3: Email Validation
```
Condition: email_valid_flag = 'VALID'
Purpose: Ensure email format is correct
Action: REJECT if invalid format
```

### Rule 4: Phone Validation
```
Condition: LENGTH(phone_clean) = 10
Purpose: Ensure phone has correct number of digits
Action: REJECT if not 10 digits
```

### Rule 5: Segmentation Validation
```
Condition: customer_segment IN ('PLATINUM', 'GOLD', 'SILVER', 'BRONZE')
Purpose: Ensure valid segment assignment
Action: REJECT if invalid segment
```

## Processing Logic

### Valid Records Path
```
Input → All Validations Pass 
      → Add validation_status = 'VALID'
      → Add validation_timestamp = CURRENT_TIMESTAMP()
      → Output to CUSTOMER_VALID table
```

### Invalid Records Path
```
Input → Any Validation Fails
      → Add validation_status = 'INVALID'
      → Add validation_error = (reason for failure)
      → Add validation_timestamp = CURRENT_TIMESTAMP()
      → Output to CUSTOMER_INVALID table
```

## Target Tables

### CUSTOMER_VALID
Contains all validated customer records ready for loading

### CUSTOMER_INVALID
Contains rejected records with error descriptions for analysis

| Column | Purpose |
|--------|---------|
| cust_id | Customer identifier |
| fname | First name |
| email_address | Email |
| phone_clean | Cleaned phone |
| customer_segment | Segment classification |
| validation_status | 'VALID' or 'INVALID' |
| validation_error | Description of errors |
| validation_timestamp | When validated |

## Key Concepts
1. ✅ Multiple validation checks
2. ✅ Error routing and categorization
3. ✅ Data quality gates
4. ✅ Audit trail (timestamps)
5. ✅ Rejectability tracking

## Practice Exercise

Add to this mapping:
1. Create detailed error messages for each validation failure
2. Add data quality scoring (0-100 based on validation passes)
3. Add quarantine logic for suspicious data patterns
4. Create separate error codes for each validation type (ERR_001, ERR_002, etc.)
5. Add a column to track which validation(s) failed
