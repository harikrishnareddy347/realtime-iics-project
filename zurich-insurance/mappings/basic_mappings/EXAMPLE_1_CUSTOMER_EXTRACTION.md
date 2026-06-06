# Example 1: Customer Data Extraction

## Overview
Extract customer data from source system with basic filtering and field mapping.

## Source Table: CUSTOMER_SOURCE
```
Columns:
- customer_id (INTEGER)
- first_name (VARCHAR)
- last_name (VARCHAR)
- email (VARCHAR)
- phone (VARCHAR)
- created_date (DATE)
- status (VARCHAR)
```

## Target Table: CUSTOMER_EXTRACTED
```
Columns:
- cust_id (INTEGER)
- fname (VARCHAR)
- lname (VARCHAR)
- email_address (VARCHAR)
- phone_number (VARCHAR)
- creation_timestamp (TIMESTAMP)
- cust_status (VARCHAR)
```

## Mapping Logic

### Step 1: Filter Active Customers
```
Condition: status = 'ACTIVE'
```

### Step 2: Field Mappings
| Source Field | Target Field | Transformation |
|---|---|---|
| customer_id | cust_id | Direct mapping |
| first_name | fname | Direct mapping |
| last_name | lname | Direct mapping |
| email | email_address | Direct mapping |
| phone | phone_number | Direct mapping |
| created_date | creation_timestamp | Direct mapping |
| status | cust_status | Direct mapping |

## Key Concepts Learned
1. ✅ Source and target configuration
2. ✅ Field-to-field mapping
3. ✅ Filtering data
4. ✅ Direct data transformations

## Practice Exercise
Modify this mapping to:
1. Include ALL customer statuses (not just ACTIVE)
2. Add a new column `extraction_date` with current date
3. Create error routing for customers with null email addresses
