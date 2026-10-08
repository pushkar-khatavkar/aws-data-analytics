# AWS for Data Analytics — Student Lab Steps

You will build a serverless analytics pipeline on real NYC Yellow Taxi data:

```
S3 (raw CSV)  →  Glue Data Catalog (schema)  →  Athena (SQL)  →  CTAS (processed → S3)  →  chart
```

Work through the steps in order. Do the **Cleanup** section before you end the lab.

---

## 0. Start your lab & sign in

1. In **CloudKida**, open the lab and click **START LAB**. Wait ~2–3 minutes.
2. When the dashboard shows your credentials, note:
   - **AWS Account ID**
   - **IAM Username:** `student`
   - **Password** (click to reveal)
   - **Console sign-in URL** (`https://<account-id>.signin.aws.amazon.com/console`)
3. Open the sign-in URL and log in as `student` with the password.
4. **Set your region first — this is critical.** In the top-right of the console, select **Asia Pacific (Mumbai) ap-south-1** and **do not change it**.
   - If you pick any other region, your bucket/database/workgroup will **not be visible** and every action will be **denied**. Just switch back to **Mumbai**.

Your environment is already created for you:

| Resource | Name |
|----------|------|
| S3 bucket | `aws-analytics-lab-student01-<account-id>` |
| Glue database | `taxi_analytics_student01` |
| Athena workgroup | `analytics-lab-student01` |
| Glue crawler role | `analytics-lab-glue-role-student01` |

**Files to upload** (from your instructor): `yellow_taxi_sample.csv`, `taxi_zone_lookup.csv`, `payment_type_lookup.csv`.

> Wherever you see `<account-id>`, replace it with your AWS Account ID (it's part of your bucket name).

---

## 1. Upload the data to S3

1. Console → **S3** → open `aws-analytics-lab-student01-<account-id>`.

![Open the S3 bucket](images/1.1.1.png)
![Bucket contents view](images/1.1.2.png)

2. **Create folder** → `raw` → **Create folder**.

![Create folder button](images/1.2.1.png)
![Name the folder raw](images/1.2.2.png)

3. Inside `raw/`, create three subfolders:
   - `raw/trips/`
   - `raw/zones/`
   - `raw/payments/`

![Open the raw folder](images/1.3.1.png)
![Create the trips subfolder](images/1.3.2.png)
![Create the zones subfolder](images/1.3.3.png)
![Create the payments subfolder](images/1.3.4.png)

4. Upload the files into their folders:
   - `raw/trips/` → **`yellow_taxi_sample.csv`**
   - `raw/zones/` → **`taxi_zone_lookup.csv`**
   - `raw/payments/` → **`payment_type_lookup.csv`**

![Open a subfolder to upload](images/1.4.1.png)
![Click Upload](images/1.4.2.png)
![Add the CSV file](images/1.4.3.png)
![Confirm the upload](images/1.4.4.png)
![Upload succeeded](images/1.4.5.png)

> Each CSV goes in its own folder — the crawler turns **one folder into one table**.

**Checkpoint:** Three CSVs are in three separate folders under `raw/`.

---

## 2. Create the schema with a Glue Crawler

A crawler scans your files and auto-detects the columns (the schema) so Athena can run SQL on them.

1. Console → **AWS Glue** → **Crawlers** → **Create crawler**.

![Open AWS Glue Crawlers](images/2.1.1.png)
![Crawlers page](images/2.1.2.png)
![Create crawler](images/2.1.3.png)

2. Name: `taxi-raw-crawler-student01` → **Next**.

![Name the crawler](images/2.2.1.png)

3. Data source → **Add a data source** → **S3**:
   - Location: `s3://aws-analytics-lab-student01-<account-id>/raw/`
   - Choose **Crawl all sub-folders** → **Add** → **Next**.

![Add a data source](images/2.3.1.png)
![Choose S3 as the source](images/2.3.2.png)
![Browse to the raw location](images/2.3.3.png)
![Set crawl all sub-folders](images/2.3.4.png)
![Add the data source](images/2.3.5.png)
![Data source added](images/2.3.6.png)

4. IAM role → **choose an existing role** → select **`analytics-lab-glue-role-student01`** → **Next**.

![Choose the existing IAM role](images/2.4.1.png)

5. Target database → select **`taxi_analytics_student01`**. Leave table prefix blank → **Next** → **Create crawler**.

![Select the target database](images/2.5.1.png)
![Review and create the crawler](images/2.5.2.png)

6. Select the crawler → **Run crawler**. Wait ~1–2 minutes until it shows **tables created: 3**.

![Run the crawler](images/2.6.1.png)
![Crawler finished — 3 tables created](images/2.6.2.png)

**Verify:** Glue → **Tables**. You should see `trips`, `zones`, `payments`. Click `trips` and review the columns.

![Three tables in the database](images/2.Verify.png)

**Checkpoint:** Three tables exist in `taxi_analytics_student01`.

---

## 3. Query the data with Athena

### 3.1 Open the query editor
1. Console → **Athena** → **Query editor**.
2. Top-right → **Workgroup** → select **`analytics-lab-student01`**.
3. Left panel → **Database** → select **`taxi_analytics_student01`**.

![Open the Athena query editor](images/3.1.1.png)
![Select the workgroup](images/3.1.2.png)
![Select the database](images/3.1.3.png)

### 3.2 Run these queries (one at a time)

**Q1 — Peek at the data:**
```sql
SELECT * FROM trips LIMIT 10;
```

![Running a query in Athena](images/3.2.1.png)

**Q2 — Total trips:**
```sql
SELECT COUNT(*) AS total_trips FROM trips;
```
**Q3 — Average fare & distance:**
```sql
SELECT ROUND(AVG(trip_distance), 2) AS avg_distance_miles,
       ROUND(AVG(fare_amount), 2)   AS avg_fare
FROM trips;
```
**Q4 — Busiest pickup hours:**
```sql
SELECT HOUR(CAST(tpep_pickup_datetime AS timestamp)) AS pickup_hour,
       COUNT(*) AS trips
FROM trips GROUP BY 1 ORDER BY trips DESC;
```
**Q5 — Tips by payment type (JOIN):**
```sql
SELECT p.payment_desc, COUNT(*) AS trips, ROUND(AVG(t.tip_amount), 2) AS avg_tip
FROM trips t JOIN payments p ON t.payment_type = p.payment_type
GROUP BY p.payment_desc ORDER BY avg_tip DESC;
```
**Q6 — Trips by pickup borough (JOIN):**
```sql
SELECT z.borough, COUNT(*) AS trips, ROUND(AVG(t.total_amount), 2) AS avg_total
FROM trips t JOIN zones z ON t.pu_location_id = z.locationid
GROUP BY z.borough ORDER BY trips DESC;
```

> Column names are **lowercase** in Athena (e.g. `locationid`, not `LocationID`). The SQL above already uses the right names.

**Checkpoint:** You can run SQL directly on data in S3 — no database server.

---

## 4. Deeper analytics + build a processed layer

### 4.1 Analytical queries

**Q7 — Revenue by borough & payment method:**
```sql
SELECT z.borough, p.payment_desc, COUNT(*) AS trips, ROUND(SUM(t.total_amount), 2) AS revenue
FROM trips t
JOIN zones z    ON t.pu_location_id = z.locationid
JOIN payments p ON t.payment_type   = p.payment_type
GROUP BY z.borough, p.payment_desc ORDER BY revenue DESC;
```
**Q8 — Long trips only (filter + HAVING):**
```sql
SELECT z.zone, COUNT(*) AS long_trips, ROUND(AVG(t.trip_distance), 2) AS avg_distance
FROM trips t JOIN zones z ON t.pu_location_id = z.locationid
WHERE t.trip_distance > 10
GROUP BY z.zone HAVING COUNT(*) > 20 ORDER BY long_trips DESC;
```

### 4.2 Create a processed table with CTAS

**CTAS** (CREATE TABLE AS SELECT) runs a query and saves the result as a new table + files in S3 — your `processed/` layer.

**Q9 — Daily revenue summary** (replace `<account-id>` with your real account ID):
```sql
CREATE TABLE daily_revenue_summary
WITH (
  format = 'PARQUET',
  external_location = 's3://aws-analytics-lab-student01-<account-id>/processed/daily_revenue/'
) AS
SELECT DATE(CAST(tpep_pickup_datetime AS timestamp)) AS trip_date,
       COUNT(*)                     AS trips,
       ROUND(SUM(total_amount), 2)  AS revenue,
       ROUND(AVG(tip_amount), 2)    AS avg_tip
FROM trips
GROUP BY DATE(CAST(tpep_pickup_datetime AS timestamp))
ORDER BY trip_date;
```

![CTAS table created and Parquet files in S3](images/4.2.Q9.png)

Verify it:
```sql
SELECT * FROM daily_revenue_summary ORDER BY trip_date LIMIT 10;
```
Then open **S3 → your bucket → `processed/daily_revenue/`** — you'll see the Parquet files Athena wrote. You just built a mini ETL pipeline.


### 4.3 Make a chart
1. Run Q9's verify query (or Q4).
2. In the Athena results panel → **Download results** (CSV).
3. Open in **Google Sheets** (Make A Copy of below sheet in your Account's Google Sheets)
   
**Link:** [Click Here](https://docs.google.com/spreadsheets/d/1XABF3umAwVCb8geM-rli2_N7_C9qBiZmxMOUPxxvQ-A/edit?usp=sharing)

![Download results and build a chart](images/4.3.1.png)

**Checkpoint:** You built raw → catalog → query → processed, and made a chart.

---

## 5. Cleanup (do this before ending the lab)

Do these in order.

1. **Athena — drop the CTAS table:**
   ```sql
   DROP TABLE IF EXISTS daily_revenue_summary;
   ```

![Drop the CTAS table](images/5.1.1.png)

2. **S3 — empty your bucket (required):**
   - Console → **S3** → open `aws-analytics-lab-student01-<account-id>`.
   - Click **Empty** → type `permanently delete` → **Empty**.
   - This clears `raw/`, `processed/`, and `athena-results/`.

   > This step is manual — it will not happen on its own, and the lab can't be torn down while the bucket still has files in it.

![Open the bucket to empty it](images/5.2.1.png)
![Confirm empty the bucket](images/5.2.2.png)
![Bucket emptied](images/5.2.3.png)

3. **(Optional) Glue — delete the crawler and tables:**
   - Glue → **Crawlers** → delete `taxi-raw-crawler-student01`.
   - Glue → **Tables** → delete `trips`, `zones`, `payments`, `daily_revenue_summary`.

![Delete the crawler](images/5.3.1.png)
![Delete the tables](images/5.3.2.png)

4. **End the lab in CloudKida** — click **End Lab**. The rest is cleaned up for you.

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| "Access denied", or you can't see your bucket/database/workgroup | You're in the wrong region. Select **Asia Pacific (Mumbai) ap-south-1** (top-right). |
| Crawler wizard won't let you create a new IAM role | Choose the existing role `analytics-lab-glue-role-student01`. |
| Glue tables show columns like `col0, col1` | The crawler missed the CSV header — ask your instructor; re-run after adding a CSV classifier with "has header". |
| Only one messy table was created | Each CSV must be in its own subfolder under `raw/` (trips / zones / payments). |
| Athena: "no output location" | Select the workgroup `analytics-lab-student01`. |
| CTAS fails: "location already exists" | Delete the `processed/daily_revenue/` folder in S3, then re-run Q9. |
| "Query exceeded scan limit" | Avoid `SELECT *` on large scans; use `LIMIT` while exploring. The sample data is small, so normal queries pass. |
