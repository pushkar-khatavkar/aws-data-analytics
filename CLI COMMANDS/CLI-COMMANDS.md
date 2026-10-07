# AWS for Data Analytics — CLI Guide

The same lab as [`HANDS-ON-LAB-GUIDE.md`](HANDS-ON-LAB-GUIDE.md), done entirely from the
terminal with the AWS CLI. Every command below was executed end-to-end against a real
account; the outputs shown are the actual results.

**Pipeline:** `S3 (raw CSV) → Glue Crawler (catalog) → Athena (SQL) → CTAS (Parquet to S3) → CSV export → chart`

| Item | Value used in this run |
|------|------------------------|
| Region | `us-east-1` |
| Account | `006833004365` |
| Caller | `arn:aws:iam::006833004365:user/student` |
| CloudFormation stack | `test` |
| S3 bucket | `aws-analytics-lab-student01-006833004365` |
| Glue database | `taxi_analytics_student01` |
| Athena workgroup | `analytics-lab-student01` |
| Glue crawler role | `arn:aws:iam::006833004365:role/analytics-lab-glue-role-student01` |
| AWS CLI | 2.36.30 |

> Replace `student01` / the account id with your own values. Everything else is copy-paste.

---

## 0. Prerequisites

```bash
aws --version
aws sts get-caller-identity        # who am I?
aws configure get region           # must be us-east-1
```

`jq` is required by the query helper in this guide:

```bash
jq --version
```

Actual output of the identity check:

```json
{
    "UserId": "AIDAQDF2HL5G4UZ3DY25Q",
    "Account": "006833004365",
    "Arn": "arn:aws:iam::006833004365:user/student"
}
```

---

## HOUR 1 — Foundations

### 1.1 Find or deploy the base infrastructure

First check whether the stack already exists. **Do this before deploying** — in this run the
infrastructure was already present under a stack named `test`, so nothing new was created.

```bash
# list every live stack
aws cloudformation list-stacks --region us-east-1 \
  --query 'StackSummaries[?StackStatus!=`DELETE_COMPLETE`].[StackName,StackStatus,CreationTime]' \
  --output table
```

```
+------+-------------------+------------------------------------+
|  test|  CREATE_COMPLETE  |  2026-10-06T08:31:47.995000+00:00  |
+------+-------------------+------------------------------------+
```

If you need to deploy it yourself:

```bash
cd cloudformation
aws cloudformation create-stack \
  --stack-name analytics-lab-student01 \
  --template-body file://analytics-lab-base.yaml \
  --parameters ParameterKey=StudentName,ParameterValue=student01 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1

# block until the stack finishes
aws cloudformation wait stack-create-complete \
  --stack-name analytics-lab-student01 --region us-east-1
```

### 1.2 Read the stack outputs

```bash
aws cloudformation describe-stacks --stack-name test --region us-east-1 \
  --query 'Stacks[0].Outputs' --output table
```

```
| RawDataPrefix         | s3://aws-analytics-lab-student01-006833004365/raw/                |
| GlueCrawlerRoleArn    | arn:aws:iam::006833004365:role/analytics-lab-glue-role-student01  |
| BucketName            | aws-analytics-lab-student01-006833004365                          |
| ProcessedDataPrefix   | s3://aws-analytics-lab-student01-006833004365/processed/          |
| GlueDatabaseName      | taxi_analytics_student01                                          |
| AthenaWorkGroupName   | analytics-lab-student01                                           |
| AthenaResultsLocation | s3://aws-analytics-lab-student01-006833004365/athena-results/     |
```

Capture the bucket name into a shell variable — the rest of the guide uses `$BUCKET`:

```bash
BUCKET=$(aws cloudformation describe-stacks --stack-name test --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='BucketName'].OutputValue" --output text)
echo "$BUCKET"
```

### 1.3 Cost awareness — inspect the guardrail

The console lesson about budgets has a CLI counterpart: read the scan cutoff baked into the
workgroup. Athena bills per byte scanned, and this workgroup refuses any single query that
would scan more than 200 MB.

```bash
aws athena get-work-group --work-group analytics-lab-student01 --region us-east-1 \
  --query 'WorkGroup.Configuration.[BytesScannedCutoffPerQuery,EnforceWorkGroupConfiguration,ResultConfiguration.OutputLocation]' \
  --output table
```

```
|  200000000                                                      |
|  False                                                          |
|  s3://aws-analytics-lab-student01-006833004365/athena-results/   |
```

`EnforceWorkGroupConfiguration=False` is deliberate: it lets the Hour 4 CTAS write to its own
`external_location`.

✅ **Checkpoint:** credentials work, region is `us-east-1`, and you know the bucket, database,
workgroup, and the cost guardrail.

---

## HOUR 2 — Storage & Ingestion (S3)

### 2.1 Inspect the dataset before uploading

```bash
head -3 dataset/yellow_taxi_sample.csv
head -4 dataset/taxi_zone_lookup.csv
cat  dataset/payment_type_lookup.csv
wc -l dataset/*.csv
```

```
trip_id,tpep_pickup_datetime,tpep_dropoff_datetime,passenger_count,trip_distance,pu_location_id,do_location_id,payment_type,fare_amount,tip_amount,tolls_amount,mta_tax,improvement_surcharge,total_amount
1,2025-01-21 07:47:17,2025-01-21 07:58:17,1,2.42,79,107,2,12.9,0.0,6.55,0.5,0.3,20.25

LocationID,Borough,Zone
4,Manhattan,Alphabet City

payment_type,payment_desc
1,Credit card
2,Cash
3,No charge
4,Dispute

    5 dataset/payment_type_lookup.csv
   52 dataset/taxi_zone_lookup.csv
20001 dataset/yellow_taxi_sample.csv
```

So: 20,000 trips, 51 zones, 4 payment types (each file has one header row).

### 2.2 Upload the raw data

There is no "create folder" step in S3 via CLI — prefixes come into being with the object.
Each CSV goes into **its own prefix**, because a Glue crawler turns each folder into one table.

```bash
aws s3 cp dataset/yellow_taxi_sample.csv  "s3://$BUCKET/raw/trips/"    --region us-east-1
aws s3 cp dataset/taxi_zone_lookup.csv    "s3://$BUCKET/raw/zones/"    --region us-east-1
aws s3 cp dataset/payment_type_lookup.csv "s3://$BUCKET/raw/payments/" --region us-east-1
```

Verify:

```bash
aws s3 ls "s3://$BUCKET/raw/" --recursive --human-readable --region us-east-1
```

```
2026-10-06 14:17:46   74 Bytes raw/payments/payment_type_lookup.csv
2026-10-06 14:17:23    1.7 MiB raw/trips/yellow_taxi_sample.csv
2026-10-06 14:17:35    1.5 KiB raw/zones/taxi_zone_lookup.csv
```

✅ **Checkpoint:** three CSVs in three separate prefixes under `raw/`.

---

## HOUR 3 — Catalog & Query (Glue + Athena)

### 3.1 Create the Glue crawler

```bash
aws glue create-crawler \
  --name taxi-raw-crawler-student01 \
  --role arn:aws:iam::006833004365:role/analytics-lab-glue-role-student01 \
  --database-name taxi_analytics_student01 \
  --targets "{\"S3Targets\":[{\"Path\":\"s3://$BUCKET/raw/\"}]}" \
  --recrawl-policy '{"RecrawlBehavior":"CRAWL_EVERYTHING"}' \
  --schema-change-policy '{"UpdateBehavior":"UPDATE_IN_DATABASE","DeleteBehavior":"DEPRECATE_IN_DATABASE"}' \
  --region us-east-1
```

Pointing the single S3 target at `raw/` is the CLI equivalent of the console's
"crawl all sub-folders" — the crawler walks down and finds `trips/`, `zones/`, `payments/`.

Confirm it registered:

```bash
aws glue get-crawler --name taxi-raw-crawler-student01 --region us-east-1 \
  --query 'Crawler.[Name,State,Role,DatabaseName,Targets.S3Targets[0].Path]' --output table
```

### 3.2 Run the crawler and wait

There is no `glue wait crawler-ready`, so poll the state yourself.

```bash
aws glue start-crawler --name taxi-raw-crawler-student01 --region us-east-1

for i in $(seq 1 60); do
  ST=$(aws glue get-crawler --name taxi-raw-crawler-student01 --region us-east-1 \
        --query 'Crawler.State' --output text)
  echo "[$(date +%H:%M:%S)] state=$ST"
  [ "$ST" = "READY" ] && break
  sleep 10
done
```

```
[14:19:09] state=RUNNING
[14:19:29] state=RUNNING
[14:19:46] state=RUNNING
[14:20:02] state=READY
```

Check the outcome and the "tables created: 3" the console shows:

```bash
aws glue get-crawler --name taxi-raw-crawler-student01 --region us-east-1 \
  --query 'Crawler.LastCrawl.[Status,ErrorMessage,StartTime]' --output table

aws glue get-crawler-metrics --crawler-name-list taxi-raw-crawler-student01 --region us-east-1 \
  --query 'CrawlerMetricsList[0].[TablesCreated,TablesUpdated,TablesDeleted,LastRuntimeSeconds]' \
  --output table
```

```
|  SUCCEEDED  |  None  |  2026-10-06T14:19:07+05:30  |

|  3        |   <- TablesCreated
|  0        |
|  0        |
|  39.639   |   <- seconds
```

### 3.3 Verify the tables and schemas

```bash
aws glue get-tables --database-name taxi_analytics_student01 --region us-east-1 \
  --query 'TableList[].[Name,StorageDescriptor.Location,Parameters.classification]' --output table
```

```
|  payments|  s3://aws-analytics-lab-student01-006833004365/raw/payments/  |  csv |
|  trips   |  s3://aws-analytics-lab-student01-006833004365/raw/trips/     |  csv |
|  zones   |  s3://aws-analytics-lab-student01-006833004365/raw/zones/     |  csv |
```

```bash
aws glue get-table --database-name taxi_analytics_student01 --name trips --region us-east-1 \
  --query 'Table.StorageDescriptor.Columns[].[Name,Type]' --output table
```

```
|  trip_id               |  bigint  |
|  tpep_pickup_datetime  |  string  |
|  tpep_dropoff_datetime |  string  |
|  passenger_count       |  bigint  |
|  trip_distance         |  double  |
|  pu_location_id        |  bigint  |
|  do_location_id        |  bigint  |
|  payment_type          |  bigint  |
|  fare_amount           |  double  |
|  tip_amount            |  double  |
|  tolls_amount          |  double  |
|  mta_tax               |  double  |
|  improvement_surcharge |  double  |
|  total_amount          |  double  |
```

```bash
aws glue get-table --database-name taxi_analytics_student01 --name zones --region us-east-1 \
  --query 'Table.StorageDescriptor.Columns[].[Name,Type]' --output table
```

```
|  locationid |  bigint  |
|  borough    |  string  |
|  zone       |  string  |
```

Two things to notice: real column names (not `col0, col1…`), which means the header was
detected; and `LocationID` came back as **`locationid`** — Glue lowercases column names, which
is why the lab SQL joins on `z.locationid`.

✅ **Checkpoint:** 3 tables, correct schemas, header row honoured.

---

## 3.4 Running Athena queries from the CLI

Athena is asynchronous over the API: you start a query, poll for its state, then fetch results.
The raw three-step form looks like this.

```bash
# 1. start
QID=$(aws athena start-query-execution \
  --query-string "SELECT COUNT(*) AS total_trips FROM trips;" \
  --work-group analytics-lab-student01 \
  --query-execution-context "Database=taxi_analytics_student01" \
  --region us-east-1 \
  --query 'QueryExecutionId' --output text)

# 2. poll
aws athena get-query-execution --query-execution-id "$QID" --region us-east-1 \
  --query 'QueryExecution.Status.State' --output text

# 3. fetch
aws athena get-query-results --query-execution-id "$QID" --region us-east-1
```

Selecting `--work-group analytics-lab-student01` is the CLI equivalent of picking the workgroup
in the console: it supplies the results location, so you never pass `--result-configuration`.

### The helper script

Repeating those three steps for every query gets old, so this repo has
[`athena-query.sh`](../athena-query.sh). It starts the query, polls to a terminal state, prints
the scanned bytes, and aligns the result into columns.

```bash
chmod +x athena-query.sh
./athena-query.sh "SQL" [STUDENT_NAME] [REGION]   # defaults: student01, us-east-1
```

> `run_athena.sh` in the repo root does the same job with no `jq` dependency, but it flattens
> all result cells into one whitespace-separated stream, which is hard to read for multi-column
> output. Use `athena-query.sh` when you want a table.

---

## 3.5 Hour 3 queries (Q1–Q6)

### Q1 — Peek at the data

```bash
./athena-query.sh "SELECT * FROM trips LIMIT 10;"
```

```
QueryExecutionId=24eca1e2-dff9-4803-9502-118511cad810  STATE=SUCCEEDED  scanned=592896B  time=745ms
trip_id  tpep_pickup_datetime  tpep_dropoff_datetime  passenger_count  trip_distance  pu_location_id  do_location_id  payment_type  fare_amount  tip_amount  tolls_amount  mta_tax  improvement_surcharge  total_amount
1        2025-01-21 07:47:17   2025-01-21 07:58:17    1                2.42           79              107             2             12.9         0.0         6.55          0.5      0.3                    20.25
2        2025-01-01 16:45:41   2025-01-01 17:27:41    1                8.14           164             137             1             38.05        2.46        0.0           0.5      0.3                    41.31
3        2025-01-04 06:06:22   2025-01-04 06:28:22    1                3.81           186             145             2             20.23        0.0         0.0           0.5      0.3                    21.03
4        2025-01-19 08:04:02   2025-01-19 08:53:02    1                11.9           138             79              1             49.9         5.32        0.0           0.5      0.3                    56.02
5        2025-01-23 22:41:04   2025-01-23 22:49:04    2                1.41           186             65              1             9.32         2.36        0.0           0.5      0.3                    12.48
```

`LIMIT` kept the scan to 593 KB instead of the full 1.77 MB — the cheap-exploration habit.

### Q2 — How many trips total?

```bash
./athena-query.sh "SELECT COUNT(*) AS total_trips FROM trips;"
```

```
scanned=1773908B  time=694ms
total_trips
20000
```

### Q3 — Average fare and distance

```bash
./athena-query.sh "SELECT ROUND(AVG(trip_distance), 2) AS avg_distance_miles, ROUND(AVG(fare_amount), 2) AS avg_fare FROM trips;"
```

```
avg_distance_miles  avg_fare
4.65                21.77
```

### Q4 — Busiest pickup hours

```bash
./athena-query.sh "SELECT HOUR(CAST(tpep_pickup_datetime AS timestamp)) AS pickup_hour, COUNT(*) AS trips FROM trips GROUP BY 1 ORDER BY trips DESC;"
```

```
pickup_hour  trips
17           1566
18           1480
8            1383
16           1340
19           1254
20           1115
7            1102
9            1083
...
4            144
```

Evening rush (17–18h) leads, morning rush (7–9h) follows, 4 AM is the quietest hour.

### Q5 — Tips by payment type (JOIN #1)

```bash
./athena-query.sh "SELECT p.payment_desc, COUNT(*) AS trips, ROUND(AVG(t.tip_amount), 2) AS avg_tip FROM trips t JOIN payments p ON t.payment_type = p.payment_type GROUP BY p.payment_desc ORDER BY avg_tip DESC;"
```

```
payment_desc  trips  avg_tip
Credit card   8439   3.28
Cash          5791   0.0
Dispute       2835   0.0
No charge     2935   0.0
```

Only card trips carry a recorded tip. That is data semantics, not a bug — cash tips never reach
the meter.

### Q6 — Trips by pickup borough (JOIN #2)

```bash
./athena-query.sh "SELECT z.borough, COUNT(*) AS trips, ROUND(AVG(t.total_amount), 2) AS avg_total FROM trips t JOIN zones z ON t.pu_location_id = z.locationid GROUP BY z.borough ORDER BY trips DESC;"
```

```
borough        trips  avg_total
Manhattan      14392  25.28
Queens         3725   25.23
Brooklyn       1013   25.2
Bronx          522    25.01
Staten Island  348    25.44
```

✅ **Checkpoint:** SQL running over files in S3, no database server anywhere.

---

## HOUR 4 — Deeper analytics, CTAS, export

### Q7 — Revenue by borough and payment method

```bash
./athena-query.sh "SELECT z.borough, p.payment_desc, COUNT(*) AS trips, ROUND(SUM(t.total_amount), 2) AS revenue FROM trips t JOIN zones z ON t.pu_location_id = z.locationid JOIN payments p ON t.payment_type = p.payment_type GROUP BY z.borough, p.payment_desc ORDER BY revenue DESC;"
```

```
borough        payment_desc  trips  revenue
Manhattan      Credit card   6078   165498.01
Manhattan      Cash          4168   98713.27
Manhattan      No charge     2091   51415.87
Manhattan      Dispute       2055   48239.44
Queens         Credit card   1570   42118.49
Queens         Cash          1071   26080.14
...
Staten Island  No charge     55     1353.06
```

20 rows (5 boroughs × 4 payment types), summing to **505,294.62** total revenue.

### Q8 — Long trips only (filter + HAVING)

```bash
./athena-query.sh "SELECT z.zone, COUNT(*) AS long_trips, ROUND(AVG(t.trip_distance), 2) AS avg_distance FROM trips t JOIN zones z ON t.pu_location_id = z.locationid WHERE t.trip_distance > 10 GROUP BY z.zone HAVING COUNT(*) > 20 ORDER BY long_trips DESC;"
```

```
zone                          long_trips  avg_distance
JFK Airport                   218         15.72
Midtown North                 214         15.89
Midtown Center                202         15.63
Midtown South                 201         16.17
Midtown East                  194         15.49
Penn Station/Madison Sq West  183         16.34
Times Sq/Theatre District     177         15.52
LaGuardia Airport             172         15.85
Upper East Side South         37          14.78
...
```

42 zones clear the `HAVING` bar. Airports and Midtown dominate long trips — exactly what you'd
expect from real taxi behaviour.

### 4.2 Build the processed layer with CTAS

CTAS fails if the target prefix already has objects, so check first:

```bash
aws s3 ls "s3://$BUCKET/processed/" --recursive --region us-east-1 || echo "(empty - good)"
```

```bash
./athena-query.sh "CREATE TABLE daily_revenue_summary
WITH (
  format = 'PARQUET',
  external_location = 's3://$BUCKET/processed/daily_revenue/'
) AS
SELECT
  DATE(CAST(tpep_pickup_datetime AS timestamp)) AS trip_date,
  COUNT(*)                     AS trips,
  ROUND(SUM(total_amount), 2)  AS revenue,
  ROUND(AVG(tip_amount), 2)    AS avg_tip
FROM trips
GROUP BY DATE(CAST(tpep_pickup_datetime AS timestamp))
ORDER BY trip_date;"
```

```
QueryExecutionId=7ffb5892-1908-4a2f-8280-f5173169e22f  STATE=SUCCEEDED  scanned=1773908B  time=1411ms
```

A CTAS returns no rows — it produced files and a table instead. Confirm both:

```bash
aws s3 ls "s3://$BUCKET/processed/daily_revenue/" --recursive --human-readable --region us-east-1
```

```
2026-10-06 14:31:39    1.3 KiB processed/daily_revenue/20261006_090137_00043_3etdm_2720657f-e666-4b98-9b2e-cedd2b69da1f
```

```bash
aws glue get-tables --database-name taxi_analytics_student01 --region us-east-1 \
  --query 'TableList[].[Name,Parameters.classification,StorageDescriptor.Location]' --output table
```

```
|  daily_revenue_summary|  None |  .../processed/daily_revenue/   |
|  payments             |  csv  |  .../raw/payments/              |
|  trips                |  csv  |  .../raw/trips/                 |
|  zones                |  csv  |  .../raw/zones/                 |
```

Query the new table:

```bash
./athena-query.sh "SELECT * FROM daily_revenue_summary ORDER BY trip_date LIMIT 10;"
```

```
QueryExecutionId=a4c0fb3e-bfeb-428e-928c-21669fbf169b  STATE=SUCCEEDED  scanned=728B  time=665ms
trip_date   trips  revenue   avg_tip
2025-01-01  658    16464.36  1.52
2025-01-02  634    16149.42  1.39
2025-01-03  634    15358.01  1.17
2025-01-04  616    15879.19  1.52
2025-01-05  614    15053.35  1.45
2025-01-06  664    16723.14  1.34
2025-01-07  677    17639.11  1.56
2025-01-08  625    15197.19  1.16
2025-01-09  631    16043.98  1.36
2025-01-10  680    18005.4   1.41
```

**728 bytes scanned, down from 1,773,908.** That's the whole point of a processed layer — and it
lands squarely on the "Athena bills per byte scanned" lesson. You just built a mini ETL pipeline
from the command line.

### 4.3 Export results (the CLI "Download results")

The console's download button just fetches `<QueryExecutionId>.csv` from the workgroup's results
prefix. Do the same with `aws s3 cp`.

```bash
mkdir -p exports

QID_HOURS=$(aws athena start-query-execution \
  --query-string "SELECT HOUR(CAST(tpep_pickup_datetime AS timestamp)) AS pickup_hour, COUNT(*) AS trips FROM trips GROUP BY 1 ORDER BY pickup_hour;" \
  --work-group analytics-lab-student01 \
  --query-execution-context "Database=taxi_analytics_student01" \
  --region us-east-1 --query 'QueryExecutionId' --output text)

QID_DAILY=$(aws athena start-query-execution \
  --query-string "SELECT * FROM daily_revenue_summary ORDER BY trip_date;" \
  --work-group analytics-lab-student01 \
  --query-execution-context "Database=taxi_analytics_student01" \
  --region us-east-1 --query 'QueryExecutionId' --output text)

# wait for both
for QID in $QID_HOURS $QID_DAILY; do
  for i in $(seq 1 30); do
    ST=$(aws athena get-query-execution --query-execution-id "$QID" --region us-east-1 \
          --query 'QueryExecution.Status.State' --output text)
    case "$ST" in SUCCEEDED|FAILED|CANCELLED) break;; esac
    sleep 2
  done
  echo "$QID -> $ST"
done

aws s3 cp "s3://$BUCKET/athena-results/${QID_HOURS}.csv" exports/busiest_hours.csv        --region us-east-1
aws s3 cp "s3://$BUCKET/athena-results/${QID_DAILY}.csv" exports/daily_revenue_summary.csv --region us-east-1
```

```bash
cat exports/busiest_hours.csv
```

```
"pickup_hour","trips"
"0","464"
"1","319"
"2","159"
"3","150"
"4","144"
"5","272"
"6","595"
"7","1102"
"8","1383"
"9","1083"
"10","761"
"11","779"
"12","928"
"13","916"
"14","868"
"15","1035"
"16","1340"
"17","1566"
"18","1480"
"19","1254"
"20","1115"
"21","890"
"22","814"
"23","583"
```

Open either CSV in Excel or Google Sheets and insert a chart — line chart for busiest hours,
bar chart for daily revenue. That's the dashboard, no QuickSight subscription needed.

### 4.4 Cost lesson, measured

Athena reports `DataScannedInBytes` for every query, so you can prove the cost argument instead
of asserting it.

```bash
./athena-query.sh "SELECT COUNT(*) AS n FROM trips;"                                  # 1,773,908 B
./athena-query.sh "SELECT ROUND(SUM(total_amount),2) AS rev FROM trips;"              # 1,773,908 B
./athena-query.sh "SELECT ROUND(SUM(revenue),2) AS rev FROM daily_revenue_summary;"   #       244 B
```

| Query | Source | Bytes scanned |
|-------|--------|---------------|
| `COUNT(*)` | CSV | 1,773,908 |
| `SUM(total_amount)` — one column | CSV | 1,773,908 |
| `SUM(revenue)` — same answer | Parquet summary | **244** |

Both CSV queries read the entire file because CSV is row-oriented and uncompressed: asking for
one column saves nothing. The Parquet processed layer answers the same business question with
**~7,270× less data scanned**. Columnar format plus pre-aggregation is where the savings live;
on real data you add partitioning on top.

The third query returned `505294.62`, which matches the sum of the Q7 borough/payment
breakdown — the processed layer and the raw layer agree.

---

## Validation — expected answers

These numbers are deterministic (the generator uses a fixed seed), so every run should match.

```bash
./athena-query.sh "SELECT (SELECT COUNT(*) FROM trips) AS trips_rows, (SELECT COUNT(*) FROM zones) AS zones_rows, (SELECT COUNT(*) FROM payments) AS payments_rows;"
```

```
trips_rows  zones_rows  payments_rows
20000       51          4
```

| Check | Expected | Got |
|-------|----------|-----|
| `trips` rows | 20,000 | 20,000 ✅ |
| `zones` rows | 51 | 51 ✅ |
| `payments` rows | 4 | 4 ✅ |
| Busiest hours | rush hours 17–18, 7–9 | 17, 18, 8, 16 ✅ |
| Tips | non-zero only for Credit card | ✅ |
| Boroughs | Manhattan dominates | 14,392 of 20,000 ✅ |
| Column names | real names, not `col0…` | ✅ |

### Mini-assessment answers

```bash
# 1. Which borough has the highest average tip?
./athena-query.sh "SELECT z.borough, ROUND(AVG(t.tip_amount), 2) AS avg_tip FROM trips t JOIN zones z ON t.pu_location_id = z.locationid GROUP BY z.borough ORDER BY avg_tip DESC;"

# 2. What is the busiest single hour of the day?
./athena-query.sh "SELECT HOUR(CAST(tpep_pickup_datetime AS timestamp)) AS pickup_hour, COUNT(*) AS trips FROM trips GROUP BY 1 ORDER BY trips DESC LIMIT 1;"

# 3. How much total revenue came from Credit card trips?
./athena-query.sh "SELECT ROUND(SUM(t.total_amount), 2) AS credit_card_revenue FROM trips t JOIN payments p ON t.payment_type = p.payment_type WHERE p.payment_desc = 'Credit card';"
```

| # | Question | Answer |
|---|----------|--------|
| 1 | Highest average tip by borough | **Brooklyn, 1.43** (Manhattan 1.40, Bronx 1.35, Queens 1.33, Staten Island 1.29) |
| 2 | Busiest single hour | **17:00 — 1,566 trips** |
| 3 | Credit card revenue | **229,417.10** |

---

## CLEANUP

> These commands delete data and infrastructure. Run them only when you're finished.

```bash
# 1. Drop the CTAS table (removes the Glue table, not the S3 files)
./athena-query.sh "DROP TABLE IF EXISTS daily_revenue_summary;"

# 2. Delete the crawler
aws glue delete-crawler --name taxi-raw-crawler-student01 --region us-east-1

# 3. Delete the catalog tables
for T in trips zones payments daily_revenue_summary; do
  aws glue delete-table --database-name taxi_analytics_student01 --name "$T" --region us-east-1 2>/dev/null
done

# 4. Empty the bucket (CloudFormation cannot delete a non-empty bucket)
aws s3 rm "s3://$BUCKET" --recursive --region us-east-1

# 5. Delete the stack (removes bucket, Glue database, workgroup, IAM role)
aws cloudformation delete-stack --stack-name test --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name test --region us-east-1
```

Confirm everything is gone:

```bash
aws s3 ls | grep analytics-lab || echo "bucket gone"
aws glue get-databases --region us-east-1 --query 'DatabaseList[].Name' --output text
aws athena list-work-groups --region us-east-1 --query 'WorkGroups[].Name' --output text
```

If the instructor is deleting the stack for the whole class, students only need step 4.

---

## Command cheat sheet

| Task | Command |
|------|---------|
| Who am I | `aws sts get-caller-identity` |
| Stack outputs | `aws cloudformation describe-stacks --stack-name test --query 'Stacks[0].Outputs' --output table` |
| Upload a file | `aws s3 cp local.csv s3://$BUCKET/raw/trips/` |
| List a prefix | `aws s3 ls s3://$BUCKET/raw/ --recursive --human-readable` |
| Create crawler | `aws glue create-crawler --name … --role … --database-name … --targets '{"S3Targets":[{"Path":"…"}]}'` |
| Run crawler | `aws glue start-crawler --name taxi-raw-crawler-student01` |
| Crawler state | `aws glue get-crawler --name … --query 'Crawler.State' --output text` |
| Tables created | `aws glue get-crawler-metrics --crawler-name-list … --query 'CrawlerMetricsList[0].TablesCreated'` |
| List tables | `aws glue get-tables --database-name taxi_analytics_student01` |
| Table schema | `aws glue get-table --database-name … --name trips --query 'Table.StorageDescriptor.Columns'` |
| Start query | `aws athena start-query-execution --query-string "…" --work-group … --query-execution-context Database=…` |
| Query state | `aws athena get-query-execution --query-execution-id $QID --query 'QueryExecution.Status.State'` |
| Bytes scanned | `aws athena get-query-execution --query-execution-id $QID --query 'QueryExecution.Statistics.DataScannedInBytes'` |
| Query results | `aws athena get-query-results --query-execution-id $QID` |
| Download results | `aws s3 cp s3://$BUCKET/athena-results/$QID.csv ./out.csv` |
| Workgroup config | `aws athena get-work-group --work-group analytics-lab-student01` |
| Run a query (helper) | `./athena-query.sh "SELECT COUNT(*) FROM trips;"` |

## Gotchas hit or worth knowing

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Stack with id analytics-lab-student01 does not exist` | Stack deployed under a different name | `aws cloudformation list-stacks` and use the real name (here: `test`) |
| Athena error: no output location | `--work-group` omitted | Always pass `--work-group analytics-lab-student01`, or supply `--result-configuration` |
| CTAS: "location already exists" | Target prefix has objects | `aws s3 rm s3://$BUCKET/processed/daily_revenue/ --recursive` or use a new path |
| Join on `locationid` fails with `LocationID` | Glue lowercases column names | Use lowercase in all SQL |
| Columns named `col0, col1…` | Crawler missed the CSV header | Re-run the crawler, or add a Glue CSV classifier with "has header" |
| One messy combined table | All CSVs in the same prefix | One CSV type per prefix under `raw/` |
| Query blocked on byte limit | 200 MB workgroup cutoff | Intentional guardrail; narrow the query |
| `glue wait` not found | No waiter exists for crawlers | Poll `Crawler.State` until `READY` |
