# Comparison Queries — Oracle vs Data Platform

## What These Queries Are For

Every time a SOAP request is made to the service with an ID number, the service
fetches portfolio data from **two sources in parallel**:

- **Oracle** — the current production database (being decommissioned)
- **Data Platform** — the new REST API that must replace Oracle

Both responses are saved side-by-side into the `astute_call_logger` Postgres table.
These queries let you read and compare what was saved after your Postman run.

> **Run order:** Query 1 first (overview), then Query 2 (drill into mismatches),
> then Query 3 (inspect specific IDs), then Query 4 (find any IDs that were missed).

---

## Connection Details

| Environment | Host | Database | Username | Password |
|---|---|---|---|---|
| **dev** | `postgres.pre.investments.momentum.co.za:5432` | `astute-logging-dev` | `astute-logging-dev` | `5pB%G32ms1zMHCWu` |
| pre | `postgres.pre.investments.momentum.co.za:5432` | `astute-logging-pre` | `astute-logging-pre` | Ask your senior |
| tst | `postgres.pre.investments.momentum.co.za:5432` | `astute-logging-tst` | `astute-logging-tst` | Ask your senior |
| prod | `postgres.investments.momentum.co.za:5432` | `astute-logging-prd` | `astute-logging-prd` | Ask your senior |

---

## Query 1 — Overall Scoreboard

**Purpose:** Get a high-level view of every ID that was triggered. Shows how many
calls matched, how many differed, and how many times the Data Platform returned nothing.
**Run this first** — it tells you where to focus.

```sql
SELECT
    id_number,
    COUNT(*) AS total_calls,
    SUM(CASE WHEN oracle_mapping = dataplatform_mapping THEN 1 ELSE 0 END) AS matches,
    SUM(CASE WHEN oracle_mapping != dataplatform_mapping THEN 1 ELSE 0 END) AS mismatches,
    SUM(CASE WHEN dataplatform_mapping IS NULL THEN 1 ELSE 0 END) AS dp_no_data
FROM astute_call_logger
GROUP BY id_number
ORDER BY mismatches DESC;
```

> **Note:** `GROUP BY id_number` is required — omitting it causes a Postgres error.

**How to read the results:**

| Column | Meaning |
|---|---|
| `total_calls` | How many times this ID was triggered |
| `matches` | Oracle and Data Platform returned identical data |
| `mismatches` | Both returned data but the content differs |
| `dp_no_data` | Data Platform returned nothing (Oracle had data, DP did not) |

---

## Query 2 — List Only Mismatched IDs

**Purpose:** Narrow down to the IDs where both sources returned data but it does not
match. These are the highest-priority discrepancies — both sources are active but
disagree on the portfolio content.

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

## Query 3 — Side-by-Side XML for a Specific ID

**Purpose:** See exactly what Oracle returned vs what Data Platform returned for one
ID. Use this after Query 2 to inspect the detail of a specific mismatch.
Replace the ID number with one from your mismatch list.

```sql
SELECT
    id_number,
    call_method,
    oracle_mapping,
    dataplatform_mapping,
    created_at
FROM astute_call_logger
WHERE id_number = '8003180537082'
ORDER BY created_at DESC
LIMIT 1;
```

> In DBeaver, click the `oracle_mapping` or `dataplatform_mapping` cell to open the
> full XML in a viewer. Use **Format XML** to make it readable.

---

## Query 4 — Find IDs That Were Never Triggered

**Purpose:** After the Collection Runner finishes, check whether every ID in your
CSV actually produced a row in the database. If any are missing it means Postman
skipped them or the request failed silently — you need to re-run those.

Paste your IDs into the `VALUES` block. A quick way: copy the `idNumber` column
from `astute_ids.csv` and replace the example values below.

```sql
SELECT id_number
FROM (VALUES
    ('5112291029328'),
    ('8307068379321'),
    ('6405062588327')
    -- paste remaining IDs here
) AS csv_ids(id_number)
WHERE id_number NOT IN (
    SELECT DISTINCT id_number FROM astute_call_logger
);
```

> If rows are returned, re-trigger those IDs in Postman before reporting.

---

## Query 5 — Confirm Rows Were Logged for Your Run

**Purpose:** Verify that your specific Postman run actually wrote data. Filter by
`created_at` to see only rows from today and check that both `oracle_mapping` and
`dataplatform_mapping` are populated (not null).

```sql
SELECT
    id_number,
    call_method,
    created_at,
    CASE WHEN oracle_mapping IS NOT NULL THEN 'YES' ELSE 'NO' END AS oracle_returned,
    CASE WHEN dataplatform_mapping IS NOT NULL THEN 'YES' ELSE 'NO' END AS dp_returned
FROM astute_call_logger
WHERE created_at >= CURRENT_DATE
ORDER BY created_at DESC;
```

---

## Query 6 — Check a Specific ID Across All Calls

**Purpose:** See every historical call for one ID — useful if an ID shows up as a
mismatch and you want to check whether it has ever matched in a previous run.

```sql
SELECT
    id,
    id_number,
    call_method,
    oracle_mapping_duration,
    gemstone_duration,
    created_at,
    CASE WHEN oracle_mapping = dataplatform_mapping THEN 'MATCH'
         WHEN dataplatform_mapping IS NULL THEN 'DP NO DATA'
         ELSE 'MISMATCH'
    END AS result
FROM astute_call_logger
WHERE id_number = '8003180537082'
ORDER BY created_at DESC;
```

---

## Reporting Format

Use this table to report results back to your senior:

| ID Number | Result | Notes |
|---|---|---|
| 8003180537082 | MATCH / MISMATCH / NO DP DATA | e.g. "DP returned fault 2102" |

- **MATCH** — Oracle and Data Platform returned identical XML
- **MISMATCH** — Both returned data but the content differs (investigate with Query 3)
- **NO DP DATA** — Data Platform returned nothing; Oracle had data
- **NO ORACLE DATA** — Oracle returned nothing; check `errorNumber` in the response
