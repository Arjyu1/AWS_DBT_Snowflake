# Airbnb Data Pipeline — AWS S3 + Snowflake + dbt

An end-to-end data engineering project that ingests Airbnb data from Amazon S3 into Snowflake and transforms it through a medallion architecture (Bronze → Silver → Gold) using dbt.

---

## What This Project Does

Raw Airbnb data (listings, bookings, hosts) lands in Amazon S3 and is loaded into Snowflake's staging schema via an external stage. From there, dbt takes over — cleaning, enriching, and modelling the data into a structured analytical layer ready for BI and reporting.

---

## Architecture

```
Amazon S3
    │
    │  (external stage / COPY INTO)
    ▼
Snowflake — STAGING schema
    │
    │  dbt
    ▼
┌─────────────────────────────────────────────┐
│  BRONZE  │  Raw ingestion, incremental load  │
├─────────────────────────────────────────────┤
│  SILVER  │  Cleaned, enriched, typed         │
├─────────────────────────────────────────────┤
│  GOLD    │  OBT, fact table, SCD2 dims       │
└─────────────────────────────────────────────┘
```

---

## Tech Stack

| Tool       | Role                              |
|------------|-----------------------------------|
| Amazon S3  | Raw data storage                  |
| Snowflake  | Cloud data warehouse              |
| dbt        | Data transformation & modelling   |
| Python     | Project environment (uv)          |

---

## Data Models

### Bronze — Raw Ingestion
Incremental loads from the Snowflake staging schema. Only new records are pulled on each run using a `CREATED_AT` watermark — no full table scans.

- `bronze_bookings`
- `bronze_listings`
- `bronze_hosts`

### Silver — Cleaned & Enriched
Business logic applied on top of bronze. Each model uses a unique key for idempotent incremental runs.

- `silver_bookings` — calculates `TOTAL_AMOUNT` (nights × nightly rate, rounded)
- `silver_listings` — tags each listing with a price band: `low / medium / high`
- `silver_hosts` — normalises host names, categorises response rate quality

### Gold — Analytical Layer

- `obt` — One Big Table joining bookings, listings, and hosts into a single wide record. Built using a metadata-driven Jinja2 loop — the join logic is defined in config, not hardcoded SQL.
- `fact` — Fact table joining the OBT against SCD2 dimension snapshots for point-in-time accuracy.
- `bookings`, `listings`, `hosts` — Ephemeral (in-memory CTE) dimension extracts used within the gold layer.

### Snapshots — SCD2 Dimensions
Tracks historical changes to listings, hosts, and bookings using dbt snapshots with timestamp strategy. Every change is preserved with `dbt_valid_from` / `dbt_valid_to` fields.

- `dim_listings`
- `dim_hosts`
- `dim_bookings`

---

## Custom Macros

| Macro                  | What it does                                                  |
|------------------------|---------------------------------------------------------------|
| `multiply(x, y, p)`    | Multiplies two columns and rounds to `p` decimal places       |
| `tag(col)`             | Categorises a numeric column into `low / medium / high` bands |
| `trimmer(col)`         | Trims whitespace and uppercases a string column               |
| `generate_schema_name` | Routes models to the correct Snowflake schema (bronze/silver/gold) |

---

## Project Structure

```
aws_dbt_snowflake_project/
├── models/
│   ├── bronze/          # Raw incremental ingestion
│   ├── silver/          # Cleaned & enriched layer
│   └── gold/
│       ├── ephemerals/  # In-memory CTE dimensions
│       ├── obt.sql      # One Big Table
│       └── fact.sql     # Fact table
├── macros/              # Reusable Jinja2 functions
├── snapshots/           # SCD2 dimension tracking
├── tests/               # Data quality tests
├── analyses/            # Ad-hoc exploration queries
└── dbt_project.yml
```

---

## Running Locally

**1. Set your Snowflake password as an environment variable**

```bash
export SNOWFLAKE_PASSWORD="your_snowflake_password"
```

**2. Install dependencies**

```bash
uv sync
```

**3. Test the connection**

```bash
dbt debug
```

**4. Run the full pipeline**

```bash
dbt build
```

Or run individual layers:

```bash
dbt run --select bronze
dbt run --select silver
dbt run --select gold
```

**5. Run snapshots**

```bash
dbt snapshot
```

---

## Snowflake Setup

The project targets the `AIRBNB` database with three schemas created automatically by dbt:

- `AIRBNB.BRONZE`
- `AIRBNB.SILVER`
- `AIRBNB.GOLD`

Source data is expected in `AIRBNB.STAGING` — loaded from S3 via a Snowflake external stage.
