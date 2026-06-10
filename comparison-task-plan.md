# Task Plan — Validate Oracle → Data Platform Mapping

**Goal:** Prove that the **Data Platform** returns the same Astute portfolio data as
**Oracle** for a list of client IDs, so Oracle can be safely decommissioned. Produce a
report of matches, mismatches, and missing data — and for each *real* mismatch, say
which field mapped incorrectly.

> This is the right project. The comparison engine lives in
> `DefaultAstuteCallLoggerService` (`makeDatabaseCall` = Oracle, `makeDataPlatformCall`
> = Data Platform). Both responses are serialised to XML and saved into the
> `astute_call_logger` Postgres table for side-by-side comparison.

---

## How the comparison actually works (read first)

```
Postman (SOAP, one request per ID, astute_ids.csv)
      |
      v  POST /WealthGemStoneService/services/GemStoneConnection   (DEV environment)
InternalService.performWithArgList()
      |
      +--> Oracle        -> AstuteMapper -> XML  -> column: oracle_mapping
      +--> Data Platform -> AstuteMapper -> XML  -> column: dataplatform_mapping
      |
      v
astute_call_logger  (Postgres: astute-logging-dev)
      |
      v  DBeaver: compare oracle_mapping vs dataplatform_mapping
```

**Key facts that shape the whole task:**

1. **Run against the DEPLOYED dev service, not your local run.** The Postman collection
   targets `http://internal-dev-investments.mmiholdings.com/...` and the results land in
   the central `astute-logging-dev` database. Your local instance (Postgres on `:5433`)
   is for development only and cannot produce valid Data Platform data without real Auth0
   credentials — do **not** use it for the validation itself.
2. **Comparison is an exact string match.** Query 1 does
   `oracle_mapping = dataplatform_mapping`. That means **element ordering, whitespace,
   encoding (ISO-8859-1), and null vs empty** can all flag a row as a "mismatch" even
   when the underlying data is identical. So a mismatch is a *candidate*, not a verdict —
   every mismatch must be eyeballed (Query 3) before you call it a real mapping defect.
3. **No auth needed.** The SOAP endpoint is unauthenticated (`Content-Type: text/xml`
   only), confirmed in the session notes.

---

## What you need

| Tool | Purpose | Notes |
|---|---|---|
| Postman | Fire one SOAP request per ID | Collection Runner with `astute_ids.csv` |
| DBeaver (or psql) | Query `astute_call_logger` | dev DB creds in `docs/comparison-queries.md` |
| `astute_ids.csv` | 898 IDs to validate | repo root |
| `astute-soap-validation-dev.postman_collection.json` | the requests | repo root |
| `astute-oracle-mapping.md` / `astute.md` | field-by-field mapping reference | use to diagnose real mismatches |

---

## Task checklist

- [ ] **1. Confirm environment access** — can you reach the dev SOAP host and the dev Postgres?
- [ ] **2. Import** `astute-soap-validation-dev.postman_collection.json` into Postman
- [ ] **3. Smoke test** one ID (`8003180537082`) — expect HTTP 200 + SOAP body
- [ ] **4. Run** the Collection Runner over all 898 IDs (`astute_ids.csv`)
- [ ] **5. Connect** DBeaver to `astute-logging-dev`
- [ ] **6. Coverage check** (Query 4 / Query 5) — every ID produced a row; re-run any misses
- [ ] **7. Scoreboard** (Query 1) — counts of match / mismatch / DP-no-data
- [ ] **8. Triage mismatches** (Query 2 → Query 3) — real defect vs false positive
- [ ] **9. Classify real mismatches by field** using `astute-oracle-mapping.md`
- [ ] **10. Report** results to your senior in the standard table format

---

## Step-by-step (how)

### 1. Confirm environment access
- You must be on the corporate network / VPN.
- Quick reachability check (optional):
  ```bash
  curl -s -o /dev/null -w "%{http_code}\n" \
    http://internal-dev-investments.mmiholdings.com/WealthGemStoneService/services/GemStoneConnection
  ```
  Anything other than a connection error means the host is reachable.
- Confirm the dev Postgres login from `docs/comparison-queries.md` works in DBeaver
  (host `postgres.pre.investments.momentum.co.za:5432`, db `astute-logging-dev`).

### 2. Import the Postman collection
- Postman → **Import** → `astute-soap-validation-dev.postman_collection.json`.
- It appears as **Astute SOAP Validation (dev)** with one request: *Get Portfolio by ID (SOAP)*.
- No auth header required.

### 3. Smoke test one ID
- Open the request, set `idNumber` = `8003180537082`, **Send**.
- **Expect:** HTTP 200, a SOAP envelope. A populated `<return>` means Oracle had data;
  a SOAP `Fault` with `2102` / "Nothing available" means no active contract (still a
  valid, logged outcome).
- This proves the pipe works before you commit to 898 calls.

### 4. Run the Collection Runner
- Click the collection → **Run collection**.
- **Data** → Select File → `astute_ids.csv`; **Iterations** = `898`.
- Consider a small **delay** (e.g. 100–250 ms) between iterations to avoid hammering the
  dev service; let it run to completion.
- Each call triggers the Oracle + Data Platform comparison server-side and writes a row.

### 5. Connect DBeaver
- Use the dev connection details in `docs/comparison-queries.md`.

### 6. Coverage check (do this BEFORE analysing)
- **Query 5** — confirm today's run wrote rows with both columns populated.
- **Query 4** — paste the CSV IDs and find any that produced no row; **re-run those few in
  Postman** before reporting. (Don't analyse an incomplete dataset.)

### 7. Scoreboard
- **Query 1** — per-ID counts of `matches`, `mismatches`, `dp_no_data`.
- This tells you the size of the problem and where to focus.

### 8. Triage mismatches (the important judgement step)
- **Query 2** lists mismatched IDs; **Query 3** shows the two XML blobs side by side.
- For each mismatch, open both cells in DBeaver, **Format XML**, and diff them. Decide:
  - **False positive** — same data, different *form* (element order, whitespace,
    encoding, empty vs null, trailing zeros on amounts, date formatting). Record as
    "cosmetic" and exclude from the defect list — but note the pattern, because a
    systematic formatting difference is itself worth raising.
  - **Real mapping defect** — a value, field, or whole product/holding present in one
    source and wrong/missing in the other.
- **DP-no-data** (Oracle has data, DP returned a Fault) is its own category — usually
  means the Data Platform has no record for that ID yet, or short-circuited (e.g. error
  `2102`). Capture the `errorNumber`.

### 9. Classify real mismatches by field
- Use `astute-oracle-mapping.md` (and `astute.md`) to name *which* field diverged —
  e.g. investor bio (`birthDate`, `title`, address), holding (`investmentValue`,
  `productCode`, `inceptionDate`, fund detail), or broker (`brokerCode`).
- Group defects by field/category — a single mapping bug usually shows up across many IDs,
  so the per-field grouping is what your senior actually needs to action.

### 10. Report
Use the format from `docs/comparison-queries.md`:

| ID Number | Result | Field / Category | Notes |
|---|---|---|---|
| 8003180537082 | MATCH | — | |
| 7001234567890 | MISMATCH | Holding.investmentValue | Oracle 12345.00 vs DP 12345 (trailing-zero only → cosmetic) |
| 8501234567891 | NO DP DATA | — | DP returned fault 2102 |

Plus a short summary: totals (matched / real-mismatch / cosmetic / no-DP-data), and the
**top 3 systematic issues** found, each with an example ID.

---

## Gotchas / things to watch

- **Exact-string comparison inflates the mismatch count.** Expect many cosmetic
  "mismatches"; budget time for triage in Step 8. If most mismatches are formatting-only,
  that finding (normalise before comparing) is itself a useful recommendation.
- **Don't validate against your local instance** — it lacks real Data Platform output.
- **Re-runs append rows.** `created_at` is set per call, so the same ID can have multiple
  rows over time. Filter by `created_at >= CURRENT_DATE` (Query 5) to isolate *your* run,
  and use the latest row per ID (Query 3 uses `ORDER BY created_at DESC LIMIT 1`).
- **Blacklisted IDs are skipped** (`AstuteBlackListRepository`) — if an ID never logs,
  check it isn't blacklisted before assuming Postman missed it.
- **Rate** — 898 sequential SOAP calls hit a shared dev service; pace them and avoid peak
  hours if you can.

---

## Reference map (existing assets — don't recreate)

| File | What it gives you |
|---|---|
| `docs/comparison-queries.md` | All 6 SQL queries + dev DB creds + reporting format |
| `docs/oracle-vs-dataplatform-validation.md` | Original step-by-step guide |
| `docs/soap-validation-session-notes.md` | Why SOAP, architecture, prior session record |
| `docs/investigation-notes.md` | Auth/endpoint dead-ends already ruled out (save yourself the detour) |
| `astute-oracle-mapping.md`, `astute.md` | Field-level Oracle↔Astute mapping for diagnosis |
| `astute_ids.csv` | The 898 IDs |
| `astute-soap-validation-dev.postman_collection.json` | The dev SOAP request |