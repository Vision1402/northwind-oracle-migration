# 02 · Source-to-Target Mapping

**Ticket:** NWM-143 · **Status:** Template (to be completed in NWM-143)

One row per column. This document is the contract between the assessment and the code:
every rule here must appear in a Bronze, Silver or Gold script.

## CUSTOMERS → BRONZE.CUSTOMERS → SILVER.CUSTOMERS

| Oracle column | Oracle type | Bronze type | Silver column | Silver rule |
|---------------|-------------|-------------|---------------|-------------|
| CUSTOMER_ID | NUMBER(10) | NUMBER(10,0) | customer_id | as-is (PK) |
| CREATED_DATE | DATE | TIMESTAMP_NTZ | created_at | `1900-01-01` → NULL; add `created_date_unknown` flag |
| PHONE | VARCHAR2(30) | VARCHAR(30) | phone | `NULLIF(TRIM(phone), '')` |
| … | … | … | … | … |

## ORDERS

| Oracle column | Oracle type | Bronze type | Silver column | Silver rule |
|---------------|-------------|-------------|---------------|-------------|
| … | … | … | … | … |

## ORDER_LINES

| Oracle column | Oracle type | Bronze type | Silver column | Silver rule |
|---------------|-------------|-------------|---------------|-------------|
| … | … | … | … | … |
