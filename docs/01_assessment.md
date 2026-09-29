# 01 · Migration Assessment: Northwind Oracle Back Office

**Author:** vision1402 · **Ticket:** NWM-143 · **Status:** Draft

## 1. Source system inventory

| Object | Type | Rows | Notes |
|--------|------|------|-------|
| CUSTOMERS | Table | 2,000 | PII: email, phone |
| EMPLOYEES | Table | 25 | Sensitive: salary, national_id |
| PRODUCTS | Table | 60 | |
| ORDERS | Table | 20,000 | CLOB `notes`; unconstrained `order_total` |
| ORDER_LINES | Table | 50,101 | Composite PK (order_id, line_no) |
| MONTHLY_SALES_SUMMARY | Table | 186 | Rebuilt nightly |
| BATCH_LOG | Table | 2 | Identity column |
| CALC_LOYALTY_TIERS | Procedure | | Business logic, must be converted |
| REFRESH_SALES_SUMMARY | Procedure | | Business logic, must be converted |
| V_CUSTOMER_ORDERS | View | | Uses `\|\|` concatenation |
| *_BIU (5) | Triggers | | ID assignment + `last_updated` stamping; **no trigger equivalent in Snowflake** |
| *_SEQ (4) | Sequences | | Must restart at Oracle's current value |
| NIGHTLY_BATCH | Scheduler job | | Daily 02:00 |

## 2. Findings and decisions

| # | Finding | Risk | Decision |
|---|---------|------|----------|
| 1 | `DATE` columns hold a time (e.g. `2021-03-14 16:42:09`) | Mapping to Snowflake `DATE` drops the time | Map to `TIMESTAMP_NTZ` |
| 2 | Oracle stores `''` as NULL | Snowflake treats `''` and NULL differently, so queries diverge | Convert `''` → NULL in Silver |
| 3 | `order_total` / `discount` have up to 5 decimals (`NUMBER` with no precision) | Rounding breaks SUM reconciliation | `NUMBER(38,10)`, no rounding; business asked whether 2dp is intended |
| 4 | `created_date = 1900-01-01` placeholder (5 rows) | Fake dates distort tenure/cohort reports | Keep in Bronze; Silver → NULL + `created_date_unknown` flag |
| 5 | `notes` CLOB with commas, quotes, line breaks, 6,000+ chars | CSV extraction splits rows/columns; truncation at 4,000 | Extract to Parquet; verify lengths after load |
| 6 | Accented names (Müller, Nguyễn), apostrophes (O'Brien) | Encoding corruption; broken hand-built SQL | UTF-8 end to end; parameterised queries only |
| 7 | `\|\|` with NULL returns text in Oracle, NULL in Snowflake | Silent wrong results in converted views/procs | Use `CONCAT_WS` / `COALESCE` |
| 8 | Orders dated before their customer's `created_date` | Source data-quality issue; negative "signup to first order" | Migrate as-is; log in DQ issues list; business to decide |
| 9 | PL/SQL procedures, triggers, sequences | Logic lost if only data moves | Convert to Snowflake Scripting / MERGE logic; reset sequences |
| 10 | NIGHTLY_BATCH updates many customers nightly (touches `last_updated`) | Inflates CDC volume; ordering matters | Snowflake Task at 02:30, CDC scheduled after it |
| 11 | Loyalty tiers depend on "last 12 months from today" | Counts shift daily | Compare Oracle vs Snowflake on the same run date |
| 12 | Salary, national_id, email, phone | Exposure of sensitive data | Masking policies + role-based access |

## 3. Golden numbers (must match after migration)

| Check | Oracle value |
|-------|-------------|
| CUSTOMERS rows | 2,000 |
| EMPLOYEES rows | 25 |
| PRODUCTS rows | 60 |
| ORDERS rows | 20,000 |
| ORDER_LINES rows | 50,101 |
| Loyalty tiers (run date: ____ ) | BRONZE 1,011 · SILVER 596 · GOLD 328 · PLATINUM 65 |

## 4. Open questions for the business

| # | Question | Owner | Answer |
|---|----------|-------|--------|
| Q1 | Should `order_total` be stored at 2 decimals going forward? | Sarah (PO) | |
| Q2 | Can 1900-01-01 dates be shown as "unknown" in reports? | Sarah (PO) | |
| Q3 | Who fixes orders that predate their customer record? | Sarah (PO) | |
