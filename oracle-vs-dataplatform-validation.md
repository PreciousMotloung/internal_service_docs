# Oracle vs Data Platform Validation Guide

## Background

This project logs portfolio data from two sources side-by-side to validate that the **Data Platform returns the same data as Oracle** before Oracle is decommissioned.

Every time an Astute portfolio request is made via the REST endpoint, the service automatically fetches from both Oracle and Data Platform and saves both XML responses to the `astute_call_logger` Postgres table.

Your task is to trigger requests for a list of client ID numbers, then query the database to identify any mismatches.

---

## What You Need

| Tool | Purpose |
|---|---|
| Postman | Get auth token + trigger requests per ID |
| DBeaver (or any Postgres client) | Query the comparison results |
| Your Excel file | List of client ID numbers to validate |
| `ids.csv` | CSV version of your Excel (you will create this) |

---

## Task Checklist

- [ ] Prepare `ids.csv` from Excel
- [ ] Get Auth0 token in Postman
- [ ] Create Astute request in Postman
- [ ] Run Collection Runner for all IDs
- [ ] Connect to dev Postgres in DBeaver
- [ ] Run summary query to identify mismatches
- [ ] Run detail queries for mismatched IDs
- [ ] Check for any IDs that were not triggered
- [ ] Report results back to senior

---

## Step 1 — Prepare Your CSV File

In Excel, ensure your ID numbers are in a single column with a header, then save as CSV:

```
idNumber
6202135212088
7001234567890
8501234567891
```

Save the file as `ids.csv` somewhere easy to find (e.g. your Desktop).

---

## Step 2 — Get an Auth Token in Postman

Create a new **POST** request:

- **URL:** `https://auth.momentuminv-dev.co.za/oauth/token`
- **Body:** `x-www-form-urlencoded`

| Key | Value |
|---|---|
| `grant_type` | `client_credentials` |
| `client_id` | `VzoeQuA4rplbYsR90coDgES5AJxwK5TS` |
| `client_secret` | `usGSvmt-wniQP4hZ-PbIWk6AuqsS8sXuqe_ghDNGXOyUeok7nmLiIpZZdq-W-gGe` |
| `scope` | `read:astute` |
| `audience` | `api://dm-datamesh` |

Send the request. Copy the `access_token` value from the response — you will use it in Step 3.

> **Note:** Tokens expire. If your Postman runner fails mid-way with 401 errors, repeat this step and update the token.

---

## Step 3 — Create the Astute Request in Postman

Create a new **POST** request inside the **same Postman Collection** as your token request:

- **URL:** `https://internal-dev-investments.mmiholdings.com/astute/`
- **Headers:**

| Key | Value |
|---|---|
| `Content-Type` | `application/json` |
| `Authorization` | `Bearer <paste your token here>` |

- **Body:** `raw` → `JSON`:

```json
{
  "idNumber": "{{idNumber}}"
}
```

The `{{idNumber}}` is a Postman variable. The Collection Runner will replace it with each row from your CSV automatically.

---

## Step 4 — Run the Collection Runner

1. Click your **Collection name** in the left sidebar
2. Click **Run collection**
3. Under **Data**, click **Select File** and choose your `ids.csv`
4. Set **Iterations** to match the number of IDs in your file
5. Click **Run**

Postman will fire one request per ID. Each request automatically triggers the Oracle vs Data Platform comparison on the server and saves the result to Postgres.

---

## Step 5 — Connect to Dev Postgres in DBeaver

Create a new PostgreSQL connection with these details:

| Field | Value |
|---|---|
| Host | `postgres.pre.investments.momentum.co.za` |
| Port | `5432` |
| Database | `astute-logging-dev` |
| Username | `astute-logging-dev` |
| Password | `5pB%G32ms1zMHCWu` |

---

## Step 6 — Run the Comparison Queries

### Query 1 — Summary per ID (start here)

```sql
SELECT
    id_number,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN oracle_mapping = dataplatform_mapping THEN 1 ELSE 0 END) AS matches,
    SUM(CASE WHEN oracle_mapping != dataplatform_mapping THEN 1 ELSE 0 END) AS mismatches,
    SUM(CASE WHEN dataplatform_mapping IS NULL THEN 1 ELSE 0 END) AS dp_no_data
FROM astute_call_logger
ORDER BY mismatches DESC;
```

This gives you a scoreboard showing which IDs match, which differ, and which returned no Data Platform data.

---

### Query 2 — List only mismatched IDs

```sql
SELECT
    id,
    id_number,
    call_method,
    oracle_mapping_duration,
    created_at
FROM astute_call_logger
WHERE dataplatform_mapping IS NOT NULL
  AND oracle_mapping IS NOT NULL
  AND oracle_mapping != dataplatform_mapping
ORDER BY created_at DESC;
```

---

### Query 3 — Side-by-side XML for a specific ID

Replace the ID number with one from your mismatch list:

```sql
SELECT
    id_number,
    call_method,
    oracle_mapping,
    dataplatform_mapping,
    created_at
FROM astute_call_logger
WHERE id_number = '6202135212088'
ORDER BY created_at DESC
LIMIT 1;
```

> In DBeaver, click on the `oracle_mapping` or `dataplatform_mapping` cell to open the full XML in a viewer for easier reading.

---

### Query 4 — Check for IDs that were never triggered

Paste all your IDs into the `VALUES` list to find any that are missing from the database:

```sql
SELECT id_number
FROM (VALUES
    ('6202135212088'),
    ('7001234567890'),
    ('8501234567891')
    -- add all your IDs here
) AS excel_ids(id_number)
WHERE id_number NOT IN (
    SELECT DISTINCT id_number FROM astute_call_logger
);
```

If any IDs are returned, re-trigger those in Postman before finalising your report.

---

## Step 7 — Report Results

Use this format to report back to your senior:

| ID Number | Result | Notes |
|---|---|---|
| 6202135212088 | MATCH | |
| 7001234567890 | MISMATCH | Oracle has data, DP differs |
| 8501234567891 | NO DP DATA | Data Platform returned nothing |

Results meaning:
- **MATCH** — Oracle and Data Platform returned identical data
- **MISMATCH** — Both returned data but it differs
- **NO DP DATA** — Data Platform returned nothing for this ID

---

## Environment Reference

| Environment | JDBC URL | Username |
|---|---|---|
| dev | `jdbc:postgresql://postgres.pre.investments.momentum.co.za:5432/astute-logging-dev` | `astute-logging-dev` |
| tst | `jdbc:postgresql://postgres.pre.investments.momentum.co.za:5432/astute-logging-tst` | `astute-logging-tst` |
| pre | `jdbc:postgresql://postgres.pre.investments.momentum.co.za:5432/astute-logging-pre` | `astute-logging-pre` |
| prod | `jdbc:postgresql://postgres.investments.momentum.co.za:5432/astute-logging-prd` | `astute-logging-prd` |
