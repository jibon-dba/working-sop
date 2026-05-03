# FDW Data Flow: Production to Analytics

This guide explains how to use **PostgreSQL Foreign Data Wrapper (FDW)** to safely access data from a **Production source database** and move it into an **Analytics destination**.

---

## Architecture Overview

```
Production DB (Source)
        ↓
Foreign Data Wrapper (FDW)
        ↓
Analytics DB (Destination)
```

---

## Step 1: Prerequisites

- PostgreSQL installed on both source and analytics databases
- Network access from Analytics DB → Production DB
- A **read-only user** on production (recommended)

---

## Step 2: Install FDW Extension (Analytics DB)

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

---

## Step 3: Create FDW Server

```sql
CREATE SERVER prod_server
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (
    host 'prod-db-host',
    port '5432',
    dbname 'production_db'
);
```

---

## Step 4: Create User Mapping

```sql
CREATE USER MAPPING FOR analytics_user
SERVER prod_server
OPTIONS (
    user 'prod_readonly_user',
    password 'strong_password'
);
```

---

## Step 5: Create Schema for Foreign Tables

```sql
CREATE SCHEMA prod_fdw;
```

---

## Step 6: Import Foreign Tables

### Import Entire Schema

```sql
IMPORT FOREIGN SCHEMA public
FROM SERVER prod_server
INTO prod_fdw;
```

### Import Selected Tables Only

```sql
IMPORT FOREIGN SCHEMA public
LIMIT TO (orders, customers)
FROM SERVER prod_server
INTO prod_fdw;
```

---

## Step 7: Query Production Data

```sql
SELECT *
FROM prod_fdw.orders
WHERE created_at >= current_date - interval '1 day';
```

> ⚠️ Queries hit **live production data**.

---

## Step 8: Materialize Data into Analytics Tables

### Initial Load

```sql
CREATE TABLE analytics.orders_daily AS
SELECT *
FROM prod_fdw.orders
WHERE created_at >= current_date - interval '1 day';
```

### Incremental Load

```sql
INSERT INTO analytics.orders_daily
SELECT *
FROM prod_fdw.orders
WHERE updated_at > (
    SELECT max(updated_at) FROM analytics.orders_daily
);
```

---

## Step 9: Schedule Data Refresh

Schedule using:
- Cron
- Airflow
- DB-native scheduler

### Example Cron Job

```bash
0 2 * * * psql -d analytics_db -f refresh_orders.sql
```

---

## Step 10: Performance Optimization

```sql
SET postgres_fdw.use_remote_estimate = true;
```

**Best Practices:**
- Avoid `SELECT *`
- Filter early (WHERE clause)
- Index production tables
- Materialize heavy joins

---

## Production Safety Rules

- Use **read-only** user
- Limit table access
- Run jobs during off-peak hours
- Monitor query performance

---

## When NOT to Use FDW

- Very large analytical joins
- Near real-time analytics at scale
- Heavy transformations

Alternative tools:
- Debezium
- Airbyte
- Fivetran

---

## Recommended Pattern

```
Production → FDW (Read-only) → Staging Tables → Analytics Tables
```

---

## License

MIT License
