# Mapping Guide - Zurich Insurance Project

## Overview

Mappings are the core transformation logic in IICS. This guide covers mapping creation from basic to advanced scenarios.

## Table of Contents

1. [Basic Mappings](#basic-mappings)
2. [Intermediate Mappings](#intermediate-mappings)
3. [Advanced Mappings](#advanced-mappings)
4. [Best Practices](#best-practices)

## Basic Mappings

### 1. Simple Column Mapping

**Scenario**: Customer data extract

**Source**: `CUSTOMER_SOURCE` table
- customer_id
- first_name
- last_name
- email
- phone

**Target**: `CUSTOMER_TARGET` table
- cust_id
- fname
- lname
- email_address
- phone_number

**Mapping Logic**:
```
SOURCE FIELDS → TRANSFORMATION → TARGET FIELDS
customer_id   → Direct Map      → cust_id
first_name    → Direct Map      → fname
last_name     → Direct Map      → lname
email         → Direct Map      → email_address
phone         → Direct Map      → phone_number
```

**Key Concepts**:
- Drag and drop field mappings
- Data type conversions
- Null handling
- Default values

---

### 2. Expression Mapping

**Scenario**: Policy premium calculation

**Expression Example**:
```
annual_premium = (base_rate * coverage_amount * risk_factor) / 100
```

**IICS Expression**:
```
annual_premium = (base_rate * coverage_amount * risk_factor) / 100
```

---

## Intermediate Mappings

### 3. Conditional Mapping (IF-THEN-ELSE)

**Scenario**: Customer tier classification

```
IF annual_spending > 50000 THEN 'PLATINUM'
ELSE IF annual_spending > 20000 THEN 'GOLD'
ELSE IF annual_spending > 5000 THEN 'SILVER'
ELSE 'BRONZE'
```

**IICS Expression**:
```iics
IIF(annual_spending > 50000, 'PLATINUM',
    IIF(annual_spending > 20000, 'GOLD',
        IIF(annual_spending > 5000, 'SILVER', 'BRONZE')
    )
)
```

---

### 4. Lookup Mapping

**Scenario**: Policy type description lookup

**Source**: POLICY table (policy_type_code)
**Lookup**: POLICY_TYPE_LOOKUP table
- policy_type_code
- policy_type_description
- coverage_type

**Mapping Logic**:
```
Lookup(POLICY_TYPE_LOOKUP, 'policy_type_code', policy_type_code)
→ Returns policy_type_description
```

---

### 5. Aggregation Mapping

**Scenario**: Customer claims summary

```
GROUP BY: customer_id
AGGREGATE:
- COUNT(*) as total_claims
- SUM(claim_amount) as total_claim_amount
- AVG(claim_amount) as avg_claim_amount
- MAX(claim_date) as latest_claim_date
```

---

## Advanced Mappings

### 6. Joiner Mapping

**Scenario**: Customer and Policy information join

**Master Input**: CUSTOMER (Master)
- customer_id (PK)
- customer_name
- customer_email

**Detail Input**: POLICY (Detail)
- policy_id (PK)
- customer_id (FK)
- policy_amount
- policy_status

**Join Condition**:
```
CUSTOMER.customer_id = POLICY.customer_id
```

**Output**:
```
customer_id, customer_name, customer_email, policy_id, policy_amount, policy_status
```

---

### 7. XML Transformation

**Scenario**: Claims data XML to flat format

**Source XML**:
```xml
<claim>
  <claim_id>CLM001</claim_id>
  <policy_ref>
    <policy_number>POL123</policy_number>
    <customer_id>CUST456</customer_id>
  </policy_ref>
  <claim_items>
    <item>
      <item_id>1</item_id>
      <amount>5000</amount>
    </item>
    <item>
      <item_id>2</item_id>
      <amount>3000</amount>
    </item>
  </claim_items>
</claim>
```

**Mapping Approach**:
- Parse XML structure
- Extract nested values
- Flatten hierarchical data
- Handle repeating elements

---

### 8. Complex Transformation Pipeline

**Scenario**: Premium calculation with multiple factors

**Pipeline Steps**:
1. Load customer risk profile
2. Apply geographical risk factor
3. Calculate base premium
4. Apply discounts based on claim history
5. Add taxes and fees
6. Apply promotional codes
7. Calculate final premium

**Expression Flow**:
```
base_premium = coverage_amount * risk_factor
risk_adjusted = base_premium * (1 + geo_risk_factor)
with_discount = risk_adjusted * (1 - (claim_history_discount/100))
with_tax = with_discount * (1 + tax_rate/100)
final_premium = with_tax - promotional_discount
```

---

## Best Practices

### 1. Naming Conventions

✅ **DO**:
- Use descriptive names: `customer_annual_premium_calculation`
- Use prefixes for mapping type: `MAP_`, `CALC_`, `LOOKUP_`
- Version your mappings: `CUSTOMER_TRANSFORM_V1`, `CUSTOMER_TRANSFORM_V2`

❌ **DON'T**:
- Use generic names: `Map1`, `Transform`
- Use special characters: `Map@Customer`, `Transform#Policy`
- Abbreviate excessively: `cust_prem_calc` (use full names)

### 2. Performance Optimization

✅ **Best Practices**:
- Use filter transforms early to reduce data volume
- Cache lookup tables when possible
- Avoid nested complex expressions
- Use sorter before joins
- Index join keys in source queries

### 3. Error Handling in Mappings

✅ **Recommended Approach**:
```iics
// Handle null values
IIF(ISNULL(customer_id), -1, customer_id)

// Handle invalid conversions
IIF(ISNUMBER(amount_str), TO_NUMBER(amount_str), 0)

// Handle date parsing
IIF(ISVALID(claim_date, 'YYYY-MM-DD'), claim_date, NULL)
```

### 4. Data Quality Checks

✅ **Include in Mappings**:
- Validate email format
- Check phone number length
- Verify policy amounts > 0
- Check customer status values
- Validate date ranges

### 5. Documentation

✅ **Document Each Mapping**:
- Purpose and business context
- Source and target systems
- Transformation rules
- Known limitations
- Contact information
- Last modified date

---

## Common Mistakes

### Mistake 1: Not Handling Nulls

❌ **Wrong**:
```iics
annual_premium = base_rate * coverage_amount
// What if base_rate or coverage_amount is NULL?
```

✅ **Correct**:
```iics
annual_premium = IIF(ISNULL(base_rate) OR ISNULL(coverage_amount), 
                     NULL, 
                     base_rate * coverage_amount)
```

### Mistake 2: Type Mismatches

❌ **Wrong**:
```iics
total = policy_amount + premium_paid  // One is STRING, other is NUMBER
```

✅ **Correct**:
```iics
total = TO_NUMBER(policy_amount) + TO_NUMBER(premium_paid)
```

### Mistake 3: Inefficient Lookups

❌ **Wrong**: Lookup on every row without caching

✅ **Correct**: Use persistent lookup cache or pre-load lookup data

---

## Practice Exercises

1. Create a simple customer mapping (name, email, phone)
2. Build a conditional mapping for customer segmentation
3. Implement a lookup for policy types
4. Create an aggregation for claim statistics
5. Build a joiner for customer-policy relationship

---

## Next Steps

After mastering basic mappings:
1. Study `TASKFLOW_GUIDE.md` to learn how to execute mappings
2. Explore `PARAMETERIZATION_GUIDE.md` for dynamic mappings
3. Review `ERROR_HANDLING_GUIDE.md` for robust transformations
