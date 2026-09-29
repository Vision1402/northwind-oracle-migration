# Northwind: Oracle to Snowflake Migration

Migrating Northwind Retail's 15-year-old Oracle back-office system (customers, orders, employees, PL/SQL business logic and a nightly batch job) to Snowflake, with no data lost, no business stopped and nothing broken.

## Architecture

```
ORACLE (legacy)                                SNOWFLAKE
┌──────────────┐  Python     ┌─────────┐  COPY  ┌────────────────────────────────┐
│ CUSTOMERS    │  extractor  │ Parquet │  INTO  │ NORTHWIND_DEV / NORTHWIND_PROD │
│ ORDERS       │ ──────────► │  files  │ ─────► │  BRONZE → SILVER → GOLD        │
│ ORDER_LINES  │             │ (stage) │        │  exact    cleaned   business   │
│ EMPLOYEES    │             └─────────┘        │  copy     & fixed   outputs    │
│ PL/SQL procs │                                └────────────────────────────────┘
└──────┬───────┘                                              │
       └───────── validation: counts · sums · hashes ─────────┘

Code: feature branch → Pull Request → review → merge → GitHub Actions deploys to PROD
```

| Layer  | Holds | Rule |
|--------|-------|------|
| BRONZE | Exact copy of Oracle | Nothing changed; must match Oracle 100% |
| SILVER | Cleaned data | Mapping-document rules applied |
| GOLD   | Loyalty tiers, summaries, views | Must match Oracle's business results |

| Environment | Database | Who changes it |
|-------------|----------|----------------|
| DEV  | `NORTHWIND_DEV`  | Engineers, by hand, freely |
| PROD | `NORTHWIND_PROD` | GitHub Actions only |

## Repository layout

```
docs/                 assessment, mapping, cutover runbook
oracle/               scripts that build the legacy Oracle system (Docker)
snowflake/setup/      one-time admin setup: roles, warehouses, databases
snowflake/migrations/ versioned SQL deployed by schemachange (V1.0__..., V1.1__...)
extract/              Python: Oracle → Parquet → Snowflake
validation/           Python: reconcile Oracle vs Snowflake
.github/workflows/    CI/CD pipelines
```

## Work log

| Ticket  | Description | Status |
|---------|-------------|--------|
| NWM-137 | Build legacy Oracle system in Docker | ✅ Done |
| NWM-138 | Repo + Snowflake DEV/PROD environments | ✅ Done |
| NWM-142 | Migrate CUSTOMERS, ORDERS, ORDER_LINES + loyalty logic | ⏳ To do |

## Running the legacy system locally

```powershell
docker run -d --name oracle-legacy -p 1521:1521 -e ORACLE_PASSWORD=<admin-password> -v oracle-data:/opt/oracle/oradata gvenzl/oracle-free:23-slim
docker cp oracle oracle-legacy:/tmp/oracle
docker exec -it oracle-legacy sqlplus system/<admin-password>@//localhost:1521/FREEPDB1 '@/tmp/oracle/00_create_user.sql'
docker exec -it oracle-legacy sqlplus northwind/Northwind123@//localhost:1521/FREEPDB1 '@/tmp/oracle/01_legacy_schema.sql'
```

> The `northwind` password in `oracle/` is a throwaway credential for a local Docker practice database. Real credentials live in `.env` (never committed).
