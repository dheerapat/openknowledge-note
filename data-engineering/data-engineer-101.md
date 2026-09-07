# Data Engineering 101: ELT Pipelines & Data Modeling

A foundational guide covering how data moves from source to warehouse (ELT) and how to model it once it's there (dimensional modeling).

> Applied example: [IMDb ELT Pipeline with Meltano + DuckDB](data-engineering-101-workshop.md)

---

## Part 1: What Is Data Engineering?

Data engineering is the discipline of building systems that **move, store, and transform data** so that analysts, data scientists, and applications can reliably use it. At its core, a data engineer answers one question over and over: *"How do I get data from where it's created to where it's needed, in a form people can trust?"*

The typical flow looks like this:

```
Source Systems → Extract → Load → Transform → Model → Consume
   (apps, DBs,      (E)      (L)      (T)      (star     (BI, ML,
    APIs, files)                              schema)     reports)
```

---

## Part 2: ETL vs ELT

### ETL (Extract, Transform, Load) — the old way

Data is transformed **before** it's loaded into the warehouse, usually on a separate processing server.

```
Source → [Transform in a separate engine] → Load into Warehouse
```

- Warehouse only ever sees clean, final data
- Transform logic lives outside the warehouse (harder to see, version, debug)
- Made sense when warehouses were expensive and couldn't handle raw compute

### ELT (Extract, Load, Transform) — the modern way

Raw data is loaded into the warehouse **first**, then transformed *inside* the warehouse using SQL.

```
Source → Load raw data into Warehouse → Transform with SQL (dbt, etc.)
```

- Takes advantage of cheap, scalable cloud/warehouse compute (Snowflake, BigQuery, DuckDB, Redshift)
- Raw data is preserved — you can always re-derive transformed tables if logic changes or bugs are found
- Transform logic lives in SQL/dbt, version-controlled, testable, and visible to the whole team
- This is the dominant pattern in the "Modern Data Stack" today

**Why ELT won:** storage got cheap, warehouse compute got cheap and elastic, and keeping raw data around gives you a safety net — you're never blocked because you "transformed away" information you didn't know you'd need later.

---

## Part 3: The ELT Pipeline, Stage by Stage

### 1. Extract

Pull data out of source systems: application databases (Postgres, MySQL), SaaS APIs (Salesforce, Stripe), event streams (Kafka), flat files (CSV, Parquet), etc.

**Key methods:**

- **Full extraction** — pull the entire table every run. Simple, but doesn't scale as data grows.
- **Incremental extraction** — pull only new/changed rows since the last run, using a timestamp/watermark column (`updated_at > last_run_time`).
- **Change Data Capture (CDC)** — read the source database's transaction log (e.g., via Debezium) to capture every insert/update/delete in near real-time, without querying the source table directly.

**Common tools:** Fivetran, Airbyte, Meltano, Stitch, custom Python scripts.

### 2. Load

Move the extracted data into the warehouse/lake **as-is** — minimal or no transformation. This creates your **raw layer** (sometimes called the "landing zone" or "bronze layer").

**Principles:**

- Keep it as close to the source format as possible.
- Don't throw anything away — you can always filter/clean later, but you can't recover data you never loaded.
- Loads should be **idempotent** — running the same load twice shouldn't duplicate data (use `MERGE`/upsert logic, or fully replace a partition).

**Common targets:** Snowflake, BigQuery, Redshift, DuckDB, or a data lake (S3 + Parquet/Iceberg).

### 3. Transform

This is where the real modeling happens — turning raw, messy data into clean, trusted, analysis-ready tables. Done in SQL, often orchestrated by **dbt**.

A typical layered structure:

| Layer | Purpose | Example |
| --- | --- | --- |
| **Raw / Staging** | 1:1 with source, light cleanup (renaming, casting types) | `stg_customers`, `stg_orders` |
| **Intermediate** | Business logic, joins, aggregations building blocks | `int_orders_with_customer` |
| **Marts / Presentation** | Final, business-facing tables — this is where dimensional modeling lives | `dim_customer`, `fct_orders` |

This staged approach (often called **medallion architecture**: bronze → silver → gold) keeps raw data untouched, makes debugging traceable (you can see exactly where a number changed), and lets multiple downstream models reuse the same clean building blocks.

---

## Part 4: Core Data Modeling Principles

Before you design tables, internalize these principles — they prevent most modeling mistakes.

### Grain

**The single most important concept in data modeling.** Grain = *what does one row in this table represent?*

- "One row per order" ≠ "one row per order line item" ≠ "one row per order per day"
- Mixing grains in a join silently duplicates or inflates numbers (a classic bug called **fan-out**)
- Every table you build should have an explicit, documented grain — decide it *before* writing SQL

### Normalization vs. Denormalization

- **Normalized** (3NF-style): data split into many small tables to eliminate redundancy — great for transactional systems (OLTP) that need fast, safe writes.
- **Denormalized**: data flattened and duplicated across fewer, wider tables — great for analytics (OLAP), because it minimizes joins and makes queries fast and easy to write.

Data warehouses generally favor **denormalized, dimensional models** because analysts query them far more often than they update them.

### Idempotency

Running a transformation twice on the same input should produce the same result — no duplicates, no drift. This is what lets you safely re-run failed pipelines.

### Slowly Changing Dimensions (SCD)

Business attributes change over time (a customer moves, a product's price changes). How you handle that history matters:

| Type | Behavior | Use when |
| --- | --- | --- |
| **Type 0** | Never update — value is fixed forever | Immutable facts (e.g., original signup date) |
| **Type 1** | Overwrite the old value, no history kept | You only care about the current state |
| **Type 2** | Insert a new row with a new surrogate key for each change; keep old rows with `valid_from`/`valid_to` | You need historical accuracy (e.g., "what was the customer's address when this order shipped?") |
| **Type 3** | Add a new column to store the previous value (limited history) | Rare — only need to compare "current vs. previous" |

**Type 2 is the standard for most dimension tables in analytics.**

### Surrogate Keys vs. Natural Keys

- **Natural/business key**: identifier from the source system (email, source UUID, SKU) — meaningful, but can change or collide across systems.
- **Surrogate key**: an artificial key *you* generate in the warehouse (integer or hash) to uniquely identify a row in your model.

Surrogate keys are essential once you implement Type 2 SCD, because the natural key alone can no longer be unique (the same customer now has multiple rows for different time periods).

```sql
-- generating a surrogate key in dbt
select
    {{ dbt_utils.generate_surrogate_key(['customer_uuid', 'valid_from']) }} as customer_sk,
    customer_uuid,   -- natural key, kept as an attribute
    email,
    address,
    valid_from,
    valid_to
from stg_customers
```

---

## Part 5: Dimensional Modeling — Star Schema

Once your data is clean, the classic way to organize it for analytics is the **star schema** (popularized by Ralph Kimball). It's built from two types of tables:

```
              dim_date
                  |
dim_customer — fact_orders — dim_product
                  |
              dim_store
```

### Fact Tables

Store **measurable, numeric events** — things that happened. Each row is a transaction, event, or measurement at a specific grain.

- Contain **foreign keys** to dimension tables, plus **measures** (numbers you aggregate: `sum`, `avg`, `count`)
- Example: `fct_orders` — one row per order line, with `customer_sk`, `product_sk`, `date_sk`, `store_sk`, `quantity`, `unit_price`, `total_amount`
- Fact tables are typically **long and narrow but grow tall** (millions/billions of rows)

**Types of fact tables:**

- **Transaction fact** — one row per event (e.g., one row per sale)
- **Periodic snapshot fact** — one row per entity per time period (e.g., daily account balance)
- **Accumulating snapshot fact** — one row per process, updated as it moves through stages (e.g., order: placed → shipped → delivered)

### Dimension Tables

Store **descriptive context** — the who, what, where, when around a fact.

- Contain **attributes** used for filtering, grouping, and labeling (names, categories, descriptions)
- Wide, but relatively few rows compared to fact tables
- Example: `dim_customer` — `customer_sk`, `customer_uuid`, `name`, `email`, `signup_date`, `segment`, `region`
- Includes a special **date dimension** (`dim_date`) almost universally — pre-built with year, quarter, month, day-of-week, fiscal periods, holidays, etc., so you never have to compute date logic in every query

### Why "star" schema?

Because visually, one fact table sits in the center surrounded by its dimension tables — like a star. It's simple, fast to query, and intuitive for BI tools (Superset, Looker, Tableau) to consume directly.

*(A related pattern, snowflake schema, normalizes dimensions further into sub-tables — e.g., splitting `dim_product` into `dim_product` + `dim_category`. It saves storage but adds joins; star schema is usually preferred for analytics simplicity.)*

---

## Part 6: Putting It All Together — Example Flow

1. **Extract**: pull raw `orders`, `customers`, `products` tables from a Postgres app DB via CDC.
2. **Load**: land them raw into the warehouse as `raw.orders`, `raw.customers`, `raw.products`.
3. **Stage**: clean/rename/cast in `stg_orders`, `stg_customers`, `stg_products` (1:1 with source, light transforms).
4. **Model dimensions**: build `dim_customer` (Type 2 SCD, surrogate key `customer_sk`) and `dim_product`.
5. **Model facts**: build `fct_orders` at the grain of "one row per order line," joining in `customer_sk`, `product_sk`, `date_sk`.
6. **Test**: run dbt tests — uniqueness of surrogate keys, not-null checks, referential integrity between fact and dimension tables.
7. **Consume**: connect Superset/Looker to the marts layer; analysts query `fct_orders` joined to dimensions for reporting.

---

## Part 7: Common Beginner Pitfalls (Quick Reference)

| Pitfall | Why it bites you |
| --- | --- |
| Ignoring grain | Joins silently duplicate rows (fan-out), inflating metrics |
| Non-idempotent loads | Re-running a failed pipeline creates duplicate data |
| Using natural keys as dimension PKs | Breaks the moment you need historical tracking (SCD Type 2) |
| Full-refresh everything | Doesn't scale; slows pipelines and increases warehouse cost |
| No tests on models | Bad data reaches dashboards silently, sometimes for weeks |
| Ignoring late-arriving data | Records get dropped or misattributed to the wrong period |
| Skipping the raw/staging layer | No safety net if transform logic has a bug — raw data is gone |

---

## Next

Worked example: [IMDb ELT Pipeline with Meltano + DuckDB](data-engineering-101-workshop.md) — maps every principle above onto real IMDb datasets.
