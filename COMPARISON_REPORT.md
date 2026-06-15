# Oracle vs Data Platform — Comparison Report

**Files compared:**
- `sorted.dp.json` — Data Platform (DP) responses, 1 651 IDs
- `sorted.oracle.json` — Oracle responses, 1 651 IDs

**How to use this report with the sorted JSON files:**
Open both files side by side in VS Code:
```
code --diff sorted.dp.json sorted.oracle.json
```
The sections below tell you exactly what to look for and where to find it in the diff.

---

## Summary Table

| Category | Count | % of total |
|---|---|---|
| Both sources returned data | 51 | 3.1% |
| Both sources returned fault 2102 | 378 | 22.9% |
| Both sources returned null | 3 | 0.2% |
| **DP has data — Oracle returned fault 2102** | **847** | **51.3%** |
| DP returned fault 2102 — Oracle has data | 32 | 1.9% |
| DP is null — Oracle returned fault 2102 | 312 | 18.9% |
| DP is null — Oracle has data | 28 | 1.7% |
| **TOTAL** | **1 651** | **100%** |

---

## Finding 1 — DP covers 847 IDs that Oracle cannot serve (critical gap)

**Count:** 847 IDs  
**What you see in the diff:**
- DP side: a full `return.result.OLifE` payload with `errorNumber: "0"`
- Oracle side: `Envelope.Body.Fault` with `faultcode: 2102` and message `"No active contract on oracle for request [ID]"`

**What it means:**  
Oracle returns a "no active contract" fault for these IDs. DP has live portfolio data for them. These are clients that Oracle has lost visibility of but DP can still serve. **DP is the only viable source for this group.**

**Sample IDs to inspect in the diff:**
```
1112047772324   1307216568327   1812046405327
1905078259321   2101210788323
```

**What to look for:**  
Search the diff for `"faultcode": "2102"` on the Oracle side where the DP side shows `"errorNumber": "0"`.

---

## Finding 2 — Oracle covers 32 IDs that DP returns a fault for

**Count:** 32 IDs  
**What you see in the diff:**
- DP side: `Envelope.Body.Fault` with `faultcode: 2102`
- Oracle side: a full `return.result.OLifE` payload

**What it means:**  
For these 32 IDs, DP is returning "no active contract" but Oracle has valid data. These are cases where DP migration is **incomplete or incorrect** — Oracle should remain the source of truth until DP is fixed.

**Sample IDs to inspect in the diff:**
```
3907175091183   4306280010080   4312010015084
4403045033083   4509230493187
```

**What to look for:**  
Search the diff for `"faultcode": "2102"` on the DP side where Oracle side shows `"errorNumber": "0"`.

---

## Finding 3 — 51 IDs where both sources have data but DP is always missing fields

**Count:** 51 IDs — **every single one** has fields that Oracle returns and DP does not.  
DP never has a field that Oracle is missing (zero DP-only fields across all 51).

**Top fields consistently missing from DP responses:**

| Times missing | Field path |
|---|---|
| 51 | `return.result.OLifE.Relation` |
| 38 | `return.result.OLifE.Holding.HoldingName` |
| 38 | `return.result.OLifE.Holding.HoldingStatus` |
| 38 | `return.result.OLifE.Holding.HoldingTypeCode` |
| 38 | `return.result.OLifE.Holding.CurrencyTypeCode` |
| 38 | `return.result.OLifE.Holding.ComponentOfPackage` |
| 38 | `return.result.OLifE.Holding.Investment.AccountValue` |
| 38 | `return.result.OLifE.Holding.Investment.AcctNum` |
| 38 | `return.result.OLifE.Holding.Investment.AcctOpenDate` |
| 38 | `return.result.OLifE.Holding.Investment.CarrierCode` |
| 38 | `return.result.OLifE.Holding.Investment.InvestType` |
| 38 | `return.result.OLifE.Holding.Investment.QualPlanType` |
| 38 | `return.result.OLifE.Holding.Investment.@CarrierPartyID` |
| 36 | `return.result.OLifE.Holding.Investment.OLifEExtension` |

**What it means:**  
DP responses are structurally incomplete. Key holding-level data — account number, account value, open date, holding name, status, currency, investment type — is systematically absent. Consumers relying on DP for these fields will receive incomplete or broken data.

**Sample ID with the largest gap:**  
`3602180050080` — Oracle returns 65 fields that DP does not.

**What to look for in the diff:**  
For any ID in the `3602180050080` range, look at the Oracle side and find blocks of fields under `Holding` and `Relation` that are entirely absent on the DP side.

---

## Finding 4 — 343 IDs where DP returns null (no response at all)

**Count:** 343 IDs  
**Breakdown:**

| DP null sub-category | Count |
|---|---|
| Oracle also fault 2102 | 312 |
| Oracle has live data | 28 |
| Oracle also null | 3 |

**What it means:**  
- The 312 null+fault cases are IDs with no active contracts on either side — expected.
- The **28 IDs where DP is null but Oracle has data** are the most concerning: DP is returning nothing (not even a fault), while Oracle has a valid response. These may indicate IDs that were never loaded into DP.
- The 3 both-null cases are likely test or invalid IDs.

**Sample IDs where DP is null but Oracle has data:**
```
21576258   4601025024085   5003114169088
5110265076085   5203010061088
```

**What to look for in the diff:**  
Search for `"ID": null` on the DP side (the value for that key will be `null`) where the Oracle side has a `return` object with data.

---

## Finding 5 — 378 IDs where both sources agree: fault 2102

**Count:** 378 IDs  
Both DP and Oracle return `faultcode: 2102` — "No active contract". The fault messages are **identical** across both sources for all 378 IDs.

**What it means:**  
These are confirmed inactive clients with no portfolio data on either system. No discrepancy, no action needed.

---

## What to look for when opening the diff

| What you see | Meaning |
|---|---|
| Oracle side has `"Fault"` block, DP side has `"return"` with data | Finding 1 — DP is the only source |
| DP side has `"Fault"` block, Oracle side has `"return"` with data | Finding 2 — Oracle is the only source, DP migration gap |
| Both sides have `"return"` but Oracle side has more fields | Finding 3 — DP response is incomplete |
| DP value is `null`, Oracle side has any response | Finding 4 — DP never responded for this ID |
| Both sides have identical `"Fault"` blocks | Finding 5 — No active contract, expected |

---

## Recommended Actions

1. **Investigate the 847 IDs (Finding 1):** Confirm whether Oracle truly has no contract or whether there is a lookup/routing failure. DP should be used as the fallback source for these clients.
2. **Fix the 32 IDs (Finding 2):** DP is incorrectly returning fault 2102 for clients Oracle can serve. Investigate the DP migration or ID mapping for these IDs.
3. **Fix the missing fields in DP (Finding 3):** The 14 systematically missing fields under `Holding` and `Relation` indicate a mapping gap in the DP response builder. These fields are present on Oracle for all 51 dual-data IDs.
4. **Investigate the 28 null IDs (Finding 4):** These IDs were never sent to DP or failed silently during ingestion. Cross-reference with the DP ingestion logs.
