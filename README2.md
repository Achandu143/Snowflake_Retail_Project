cls# Snowflake Retail Data Pipeline

An end-to-end retail data pipeline built on Snowflake. Files land in S3, Snowpipe
auto-ingests them into raw tables, Streams + Tasks handle the staging transforms
on a 5-minute schedule, and the analytics layer ends in a star schema with
reporting views ready for BI tools.

I put this together to get hands-on with Snowflake's ingestion and orchestration
features (Snowpipe, Streams, Tasks) on a realistic dataset instead of toy
examples. It's structured the way I'd actually build it at work — with proper
role separation, audit columns, and a medallion layout.

## Architecture

```
┌──────────────┐     ┌──────────────┐      ┌──────────────────────────┐
│  Source data │     │  Amazon S3   │      │        Snowflake         │
│              │────▶│  (raw zone)  │────▶│                          │
│  CSV / JSON  │     │  customers/  │  SQS│  ┌────────┐  ┌────────┐  │
│  customers,  │     │  products/   │─────▶  │  RAW   │─▶│STAGING │  │
│  products,   │     │  orders/     │  │     │ tables │  │ tables │  │
│  orders,     │     │  order_items/│  │     └────────┘  └────────┘  │
│  order_items │     └──────────────┘  │          │           │      │
└──────────────┘            │          │      Snowpipe   Streams +   │
                            │          │       (auto      Tasks      │
                  upload_to_s3.py      │      ingest)   (5 min ETL) │
                                       │                      │      │
                                       │                      ▼      │
                                       │             ┌──────────────┐│
                                       │             │  ANALYTICS   ││
                                       │             │ Dims + Facts ││
                                       └────────────▶│   + Views    ││
                                                     └──────────────┘│
                                                            │         │
                                                            ▼         │
                                                       BI tools       │
                                                  ────────────────────┘
```

The pipeline follows a medallion layout:

- **Bronze (`RAW`)** — files exactly as they land from S3, plus audit columns
  (load timestamp, file name, row number).
- **Silver (`STAGING`)** — typed, validated, deduplicated. Loaded with `MERGE`
  off Streams on the RAW tables.
- **Gold (`ANALYTICS`)** — conformed dimensions and fact tables in a star
  schema, with reporting views on top.

## Project layout

```
Snowflake_Retail_Project/
├── data/sample/                     # Sample retail data
│   ├── customers.csv
│   ├── products.csv
│   ├── orders.json
│   └── order_items.json
│
├── sql/
│   ├── 01_setup/
│   │   └── 01_create_database_roles.sql
│   ├── 02_raw_layer/
│   │   ├── 01_stages_and_formats.sql      # S3 integration, file formats, stages
│   │   ├── 02_raw_tables.sql              # Landing tables w/ audit columns
│   │   └── 03_snowpipes.sql               # Snowpipe defs (auto_ingest)
│   ├── 03_staging_layer/
│   │   ├── 01_staging_tables_and_streams.sql
│   │   └── 02_staging_tasks.sql
│   └── 04_analytics_layer/
│       ├── 01_dimensions.sql              # DIM_DATE, DIM_CUSTOMER, DIM_PRODUCT
│       └── 02_facts_and_reports.sql       # FACT tables + RPT_* views
│
├── python/
│   ├── upload_to_s3.py                    # Pushes sample files to S3
│   └── monitor_pipeline.py                # Pipe / task / row-count checks
│
├── .github/workflows/deploy.yml           # Lint + deploy on push to main
├── .gitignore
├── requirements.txt
└── README.md
```

## Getting it running

### Prerequisites
- A Snowflake account (the free trial is fine)
- An AWS account with an S3 bucket
- Python 3.10+

### 1. Clone and install
```bash
git clone https://github.com/Achandu143/Snowflake_Retail_Project.git
cd Snowflake_Retail_Project
pip install -r requirements.txt
```

### 2. Run the SQL in order
The scripts are numbered for a reason — run them top to bottom. Either paste
them into Snowflake worksheets or use SnowSQL:

```bash
snowsql -a <account> -u <user> -f sql/01_setup/01_create_database_roles.sql
snowsql -a <account> -u <user> -f sql/02_raw_layer/01_stages_and_formats.sql
# ...and so on
```

### 3. Wire up S3 + Snowpipe (one-time setup)

In `sql/02_raw_layer/01_stages_and_formats.sql` you'll need to swap a few
placeholders before running it:

- `<YOUR_AWS_ACCOUNT_ID>` → your AWS account ID
- `your-retail-bucket` → your bucket name
- `snowflake-retail-role` → the IAM role you'll create

After the storage integration is created:

```sql
DESC INTEGRATION S3_RETAIL_INTEGRATION;
```

Grab `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID` and paste them
into your IAM role's trust policy. Then create the pipes and run `SHOW PIPES;`
— you'll need each pipe's `notification_channel` ARN to set up the S3 event
notification → SQS.

> Heads up: the SQS event notification piece tripped me up the first time. If
> the pipes show no activity, double-check that the bucket events are pointing
> at the right SQS ARN and that the prefix matches.

### 4. Upload sample data
```bash
python python/upload_to_s3.py --bucket your-retail-bucket --env dev
```

Snowpipe should pick the files up within ~1 minute and load them into the RAW
tables.

### 5. Build the analytics layer
The staging tasks run every 5 minutes on their own. To build the dims and facts:

```sql
USE ROLE RETAIL_ENGINEER_ROLE;
USE WAREHOUSE RETAIL_TRANSFORM_WH;

-- dimensions first
\i sql/04_analytics_layer/01_dimensions.sql
-- then facts + reporting views
\i sql/04_analytics_layer/02_facts_and_reports.sql
```

### 6. Check that everything's flowing
```bash
python python/monitor_pipeline.py --account <account> --user <user> --check all
```

This script checks pipe status, task history, row counts per layer, and a few
basic data-quality assertions.

## Reporting views

| View | What it gives you |
|------|-------------------|
| `RPT_SALES_SUMMARY_DAILY` | Daily revenue, orders, AOV, gross profit |
| `RPT_CUSTOMER_LIFETIME_VALUE` | CLV, retention status, order frequency |
| `RPT_PRODUCT_PERFORMANCE` | Revenue rank, margin %, revenue share |
| `RPT_CATEGORY_SALES_MONTHLY` | MoM growth by category / sub-category |

A few example queries:

```sql
-- Top 5 customers by lifetime value
SELECT FULL_NAME, CUSTOMER_SEGMENT, TOTAL_ORDERS, LIFETIME_VALUE
FROM RETAIL_DB.ANALYTICS.RPT_CUSTOMER_LIFETIME_VALUE
ORDER BY LIFETIME_VALUE DESC
LIMIT 5;

-- Monthly revenue trend
SELECT YEAR, MONTH_NAME, GROSS_REVENUE, GROSS_PROFIT, AVG_ORDER_VALUE
FROM RETAIL_DB.ANALYTICS.RPT_SALES_SUMMARY_DAILY
GROUP BY YEAR, MONTH_NAME
ORDER BY YEAR, MIN(FULL_DATE);

-- Top 10 products by revenue
SELECT PRODUCT_NAME, CATEGORY, TOTAL_REVENUE, AVG_MARGIN_PCT, REVENUE_RANK
FROM RETAIL_DB.ANALYTICS.RPT_PRODUCT_PERFORMANCE
ORDER BY REVENUE_RANK
LIMIT 10;
```

## End-to-end data flow

```
File lands in S3
      │
      ▼   (SQS event, ~1 min)
Snowpipe COPY INTO RAW_*
      │
      ▼   (Stream captures the inserts, Task runs every 5 min)
TASK_LOAD_STG_* → MERGE INTO STAGING.*    (cast types, dedupe)
      │
      ▼   (manual run or scheduled Task)
DIM_CUSTOMER / DIM_PRODUCT / FACT_ORDERS / FACT_ORDER_ITEMS
      │
      ▼
RPT_* reporting views → BI / dashboards
```

## A few notes on security

Nothing fancy, just the basics I wanted to be deliberate about:

- No credentials in the repo — set them via env vars or use the Snowflake
  storage integration (which uses short-lived IAM tokens, not access keys)
- Roles are scoped: `RETAIL_LOADER_ROLE` can only write to `RAW`, analysts only
  read from `ANALYTICS`
- S3 uploads use server-side encryption (`AES256`)

## CI/CD

`.github/workflows/deploy.yml` does the following on push to `main`:

1. Lints SQL with `sqlfluff` (Snowflake dialect)
2. Runs `flake8` + `black --check` on the Python
3. Deploys the SQL layers to Snowflake in order

Repo secrets you'll need to set:

| Secret | What it is |
|--------|------------|
| `SNOWFLAKE_ACCOUNT` | e.g. `xy12345.us-east-1` |
| `SNOWFLAKE_USER` | Snowflake username |
| `SNOWFLAKE_PASSWORD` | Snowflake password |

## Stack

| Tool | What it does here |
|------|-------------------|
| Snowflake | Warehouse, transformations, orchestration |
| Snowpipe | Auto-ingest from S3 |
| Streams | CDC on RAW tables |
| Tasks | Scheduled SQL (cron-like) |
| Amazon S3 | Landing zone |
| boto3 | Python S3 uploads |
| GitHub Actions | Lint + deploy |

## Things I'd still like to add

- [ ] dbt layer over the analytics models (or at least dbt tests on the dims/facts)
- [ ] Late-arriving dimension handling on `DIM_CUSTOMER`
- [ ] A `DEV` / `PROD` split in the deploy workflow instead of single-environment
- [ ] Slack alert on failed tasks via a separate monitoring task

## Pushing to GitHub

```bash
cd Snowflake_Retail_Project

git init
git add .
git commit -m "Initial commit: retail snowflake pipeline"

git remote add origin https://github.com/Achandu143/Snowflake_Retail_Project.git
git branch -M main
git push -u origin main
```
