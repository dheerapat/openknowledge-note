# IMDb ELT Pipeline with Meltano + DuckDB

Maps every principle from [Data Engineering 101](./data-engineer-101.md) onto a real dataset — [IMDb's non-commercial datasets](https://developer.imdb.com/non-commercial-datasets/) — and designs a working ELT pipeline around it.

---

## Part 0: Understand the Source Data First

IMDb publishes gzipped TSV files, refreshed daily, at `https://datasets.imdbws.com/`. We'll use four of them:

| File | Grain (1 row = ...) | Key columns |
| --- | --- | --- |
| `title.basics.tsv.gz` | one title (movie/show/episode) | `tconst` (PK), `titleType`, `primaryTitle`, `startYear`, `runtimeMinutes`, `genres` (up to 3, comma-packed) |
| `title.ratings.tsv.gz` | one title's *current* rating snapshot | `tconst` (PK), `averageRating`, `numVotes` |
| `title.principals.tsv.gz` | one person's involvement in one title | `tconst`, `ordering`, `nconst`, `category` (actor/director/etc.), `characters` |
| `name.basics.tsv.gz` | one person | `nconst` (PK), `primaryName`, `birthYear`, `deathYear`, `primaryProfession` |

Two things jump out immediately, and they drive most of our modeling decisions below:

1. `genres` **is a multi-valued field** (a title can have up to 3 genres packed into one string) — this is a classic **many-to-many relationship** disguised as a flat column.
2. `title.ratings` **is a daily-refreshed snapshot**, not an immutable fact — the same title's rating changes over time. That's a strong signal for a **periodic snapshot fact table**, not a simple overwrite.

This is exactly the kind of thing that's invisible in a tutorial but obvious once you look at real data — which is the point of this exercise.

---

## Part 1: Extract & Load — Meltano + DuckDB

### Pipeline shape

```
IMDb .tsv.gz files → Meltano tap (extract) → DuckDB raw schema (load) → dbt (transform)
```

### `meltano.yml`

Since the files are flat, gzipped TSVs over HTTP, `tap-spreadsheets-anywhere` works well — it can read remote, compressed, delimited files directly.

```yaml
version: 1
default_environment: dev
project_id: imdb-mds-in-a-box

plugins:
  extractors:
    - name: tap-spreadsheets-anywhere
      variant: ets
      pip_url: git+https://github.com/ets/tap-spreadsheets-anywhere.git
      config:
        tables:
          - path: https://datasets.imdbws.com
            name: title_basics
            pattern: title.basics.tsv.gz
            format: csv
            csv_options:
              delimiter: "\t"
              compression: gzip
              null_values: ["\\N"]
            key_properties: [tconst]

          - path: https://datasets.imdbws.com
            name: title_ratings
            pattern: title.ratings.tsv.gz
            format: csv
            csv_options:
              delimiter: "\t"
              compression: gzip
              null_values: ["\\N"]
            key_properties: [tconst]

          - path: https://datasets.imdbws.com
            name: title_principals
            pattern: title.principals.tsv.gz
            format: csv
            csv_options:
              delimiter: "\t"
              compression: gzip
              null_values: ["\\N"]
            key_properties: [tconst, ordering]

          - path: https://datasets.imdbws.com
            name: name_basics
            pattern: name.basics.tsv.gz
            format: csv
            csv_options:
              delimiter: "\t"
              compression: gzip
              null_values: ["\\N"]
            key_properties: [nconst]

  loaders:
    - name: target-duckdb
      variant: jwills
      pip_url: target-duckdb~=0.4
      config:
        filepath: /tmp/imdb.duckdb
        default_target_schema: raw

  transformers:
    - name: dbt-duckdb
      variant: jwills
      pip_url: dbt-core~=1.2.0 dbt-duckdb~=1.2.0
      config:
        path: /tmp/imdb.duckdb
```

Run it:

```bash
meltano run tap-spreadsheets-anywhere target-duckdb
```

**ELT principle in action:** notice we load `genres` as a raw comma-packed string and `\N` nulls as-is — no cleanup at extract time. That normalization happens *inside the warehouse*, in dbt, where it's visible, testable, and versioned. That's the E-L-T order, not E-T-L.

**Idempotency check:** IMDb refreshes these files daily and each row has a stable primary key (`tconst`, `nconst`, or `tconst+ordering`). Using `key_properties` lets `target-duckdb` upsert on re-run instead of blindly appending — so re-running the pipeline twice in a day doesn't duplicate rows.

---

## Part 2: Staging Layer — Clean, 1:1 with Source

Grain stays identical to the raw table here — staging just casts types, splits packed fields, and standardizes nulls. No business logic yet.

```sql
-- models/staging/stg_title_basics.sql
select
    tconst,
    titletype                          as title_type,
    primarytitle                       as primary_title,
    cast(startyear as integer)         as start_year,
    cast(runtimeminutes as integer)    as runtime_minutes,
    isadult = '1'                      as is_adult,
    genres                             as genres_raw   -- still packed, split downstream
from raw.title_basics
```

```sql
-- models/staging/stg_title_ratings.sql
select
    tconst,
    cast(averagerating as decimal(3,1)) as average_rating,
    cast(numvotes as integer)           as num_votes,
    current_date                        as snapshot_date  -- captured at load time
from raw.title_ratings
```

```sql
-- models/staging/stg_title_principals.sql
select
    tconst,
    cast(ordering as integer) as ordering,
    nconst,
    category,
    characters
from raw.title_principals
```

```sql
-- models/staging/stg_name_basics.sql
select
    nconst,
    primaryname                as primary_name,
    cast(birthyear as integer) as birth_year,
    cast(deathyear as integer) as death_year,
    primaryprofession           as professions_raw
from raw.name_basics
```

---

## Part 3: Solving the Many-to-Many — Bridge Table for Genres

`genres_raw` (`"Comedy,Drama,Romance"`) is a normalization problem. If we naively `split` and pivot into columns, we lose flexibility. The correct dimensional pattern is a **bridge table** — one row per title-genre pair.

```sql
-- models/intermediate/int_title_genres.sql
select
    tconst,
    trim(unnest(str_split(genres_raw, ','))) as genre
from {{ ref('stg_title_basics') }}
where genres_raw is not null
```

This turns a hidden many-to-many relationship into an explicit table — now `dim_genre` and a bridge can join cleanly to `fct_title_ratings` without duplicating rating rows (a classic **fan-out** trap if done wrong: joining ratings directly to a comma-split genre list, one-to-many, would inflate `numVotes` sums by 2-3x for multi-genre titles).

---

## Part 4: Dimensional Model — Star Schema

### Grain decisions (made explicitly, before writing SQL)

| Table | Grain | Type |
| --- | --- | --- |
| `dim_title` | one row per title (`tconst`) | Dimension |
| `dim_person` | one row per person (`nconst`) | Dimension |
| `dim_genre` | one row per genre | Dimension |
| `bridge_title_genre` | one row per title-genre pair | Bridge (resolves M:N) |
| `fct_title_ratings` | one row per title **per snapshot date** | Fact — periodic snapshot |
| `fct_title_principals` | one row per title-person-role | Fact — factless fact (an event/association, no numeric measure) |

### `dim_title`

```sql
-- models/marts/dim_title.sql
select
    {{ dbt_utils.generate_surrogate_key(['tconst']) }} as title_sk,
    tconst,                        -- natural key, kept as attribute
    title_type,
    primary_title,
    start_year,
    runtime_minutes,
    is_adult
from {{ ref('stg_title_basics') }}
```

**Surrogate key applied:** `tconst` is IMDb's own surrogate key (meaningless alphanumeric ID, generated by *their* system) — from our warehouse's point of view it's a natural/business key. We mint our own `title_sk` on top of it so our fact tables join to a warehouse-controlled key, decoupled from IMDb's ID scheme.

### `dim_person`

```sql
-- models/marts/dim_person.sql
select
    {{ dbt_utils.generate_surrogate_key(['nconst']) }} as person_sk,
    nconst,
    primary_name,
    birth_year,
    death_year
from {{ ref('stg_name_basics') }}
```

### `dim_genre` + `bridge_title_genre`

```sql
-- models/marts/dim_genre.sql
select distinct
    {{ dbt_utils.generate_surrogate_key(['genre']) }} as genre_sk,
    genre
from {{ ref('int_title_genres') }}
```

```sql
-- models/marts/bridge_title_genre.sql
select
    t.title_sk,
    g.genre_sk
from {{ ref('int_title_genres') }} tg
join {{ ref('dim_title') }} t using (tconst)
join {{ ref('dim_genre') }} g using (genre)
```

### `fct_title_ratings` — Periodic Snapshot Fact

This is the most important modeling call in this dataset. Ratings **change daily** as more votes come in. If we just `MERGE`/overwrite by `tconst`, we permanently lose the ability to answer "how has this title's rating trended over time?" — so instead of Type 1 overwrite, we treat every load as a new snapshot row.

```sql
-- models/marts/fct_title_ratings.sql
select
    {{ dbt_utils.generate_surrogate_key(['tconst', 'snapshot_date']) }} as rating_sk,
    t.title_sk,
    r.snapshot_date,
    r.average_rating,
    r.num_votes
from {{ ref('stg_title_ratings') }} r
join {{ ref('dim_title') }} t using (tconst)
```

**Grain:** one row per title *per day loaded* — not one row per title. This is a **periodic snapshot fact table**, directly from the fact-table taxonomy in the [DE 101 guide](./data-engineer-101.md) (transaction / periodic snapshot / accumulating snapshot). Run this pipeline daily for a month and you have a full time series of rating momentum — impossible with an overwrite-in-place design.

### `fct_title_principals` — Factless Fact Table

No numeric measure here (no dollar amount, no count to sum) — this table just records *that an association happened* (person X was credited as director on title Y). This is a well-known dimensional pattern called a **factless fact table**, used for tracking events/relationships rather than quantities.

```sql
-- models/marts/fct_title_principals.sql
select
    {{ dbt_utils.generate_surrogate_key(['tconst', 'ordering']) }} as principal_sk,
    t.title_sk,
    p.person_sk,
    pr.category,
    pr.characters
from {{ ref('stg_title_principals') }} pr
join {{ ref('dim_title') }} t using (tconst)
join {{ ref('dim_person') }} p using (nconst)
```

### The resulting star

```
                dim_genre
                    |
              bridge_title_genre
                    |
dim_person — fct_title_principals — dim_title — fct_title_ratings
                                          |
                                    (snapshot_date)
```

Note there are **two fact tables sharing** `dim_title` — a common real-world pattern. Multiple business processes (ratings tracking, cast/crew tracking) can share the same dimension without duplicating it.

---

## Part 5: Testing — Catching Bad Data Before It Reaches a Dashboard

```yaml
# models/marts/schema.yml
models:
  - name: dim_title
    columns:
      - name: title_sk
        tests: [unique, not_null]
      - name: tconst
        tests: [unique, not_null]

  - name: fct_title_ratings
    columns:
      - name: rating_sk
        tests: [unique, not_null]
      - name: title_sk
        tests:
          - not_null
          - relationships:
              to: ref('dim_title')
              field: title_sk
      - name: average_rating
        tests:
          - dbt_utils.accepted_range:
              min_value: 0
              max_value: 10
```

This catches the exact class of bug the DE 101 guide warned about: a broken join (referential integrity test), a duplicate key (uniqueness test), or a value out of plausible range (e.g., a rating of 15 out of 10 from a parsing bug).

---

## Part 6: Full Principle-to-Implementation Map

| 101 Principle | Where it shows up in this pipeline |
| --- | --- |
| **ELT, not ETL** | Raw TSVs loaded untouched into `raw` schema; all cleanup happens in dbt SQL afterward |
| **Idempotent loads** | `key_properties` in Meltano config enables upsert instead of append-only duplication |
| **Grain** | Explicitly defined per table *before* writing SQL (see Part 4 table) — prevents fan-out bugs |
| **Raw → Staging → Marts layering** | `raw.title_basics` → `stg_title_basics` → `dim_title` |
| **Natural key vs. surrogate key** | `tconst`/`nconst` (IMDb's own surrogate keys) kept as natural/business keys; `title_sk`/`person_sk` minted fresh for warehouse use |
| **Many-to-many resolution** | `genres` packed string → `bridge_title_genre` + `dim_genre`, avoiding join fan-out |
| **Periodic snapshot fact** | `fct_title_ratings` — one row per title per load date, preserving rating history instead of overwriting |
| **Factless fact table** | `fct_title_principals` — records an association (who worked on what) with no numeric measure |
| **Star schema** | Two fact tables sharing `dim_title`, surrounded by `dim_person`, `dim_genre` |
| **Data quality tests** | `unique`, `not_null`, `relationships`, and range tests on the marts layer |
