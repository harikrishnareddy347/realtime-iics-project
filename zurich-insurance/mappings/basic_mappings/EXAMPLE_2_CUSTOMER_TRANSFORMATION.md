# Example 2: Customer Data Transformation with Expressions

## Overview
Transform customer data using business rules, expressions, and data quality checks.

## Source Data
```
Columns:
- customer_id (INTEGER)
- first_name (VARCHAR)
- last_name (VARCHAR)
- email (VARCHAR)
- phone (VARCHAR)
- country (VARCHAR)
- annual_spending (DECIMAL)
- created_date (DATE)
- status (VARCHAR)
```

## Transformations Applied

### Step 1: Filter by Status
```
Condition: status = 'ACTIVE'
Purpose: Only process active customers
```

### Step 2: Clean Phone Number
```
Expression: REGEX_REPLACE(phone, '[^0-9]', '')
Purpose: Remove formatting characters from phone
Input: (123) 456-7890
Output: 1234567890
```

### Step 3: Validate Email Format
```
Expression: IIF(
  REGEX_MATCH(email, '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'),
  'VALID',
  'INVALID'
)
Purpose: Check if email matches standard format
```

### Step 4: Customer Segmentation
```
Expression: IIF(
  annual_spending > 50000, 'PLATINUM',
  IIF(annual_spending > 20000, 'GOLD',
    IIF(annual_spending > 5000, 'SILVER', 'BRONZE')
  )
)
Purpose: Classify customers by spending level
Logic:
  > $50,000 → PLATINUM
  > $20,000 → GOLD
  > $5,000 → SILVER
  ≤ $5,000 → BRONZE
```

### Step 5: Add Default Country
```
Expression: IIF(ISNULL(country), 'USA', country)
Purpose: Provide default value for null country
```

### Step 6: Add Processing Timestamp
```
Expression: CURRENT_TIMESTAMP()
Purpose: Track when record was processed
```

## Target Table Output
| Column | Source | Transformation |
|--------|--------|---|
| cust_id | customer_id | Direct |
| fname | first_name | Direct |
| email_address | email | Direct |
| phone_clean | phone | Remove non-numeric |
| email_valid_flag | email | Validation check |
| customer_segment | annual_spending | Nested IIF logic |
| country_code | country | Default to USA |
| processed_timestamp | N/A | Current timestamp |

## Key IICS Functions Used
1. **IIF()** - Conditional logic (Immediate IF)
2. **REGEX_REPLACE()** - Pattern replacement
3. **REGEX_MATCH()** - Pattern matching
4. **ISNULL()** - Null checking
5. **CURRENT_TIMESTAMP()** - Current date/time

## Practice Exercise

Modify this mapping to add:
1. Age group classification (18-25, 26-40, 41-60, 60+)
2. Phone validation (must be 10 digits after cleaning)
3. Email domain extraction (e.g., @gmail.com)
4. Risk score calculation (0-100 based on customer profile)
