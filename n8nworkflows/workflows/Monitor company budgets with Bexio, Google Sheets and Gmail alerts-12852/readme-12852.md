Monitor company budgets with Bexio, Google Sheets and Gmail alerts

https://n8nworkflows.xyz/workflows/monitor-company-budgets-with-bexio--google-sheets-and-gmail-alerts-12852


# Monitor company budgets with Bexio, Google Sheets and Gmail alerts

## 1. Workflow Overview

**Purpose:** Monitor company budgets proactively by syncing Bexio accounting journal entries into Google Sheets, computing monthly actual costs per budget metric, comparing them to budget targets stored in Google Sheets, and sending a weekly email report via Gmail.

**Primary use cases**
- Keep a Google Sheets “Journal” sheet continuously updated from Bexio.
- Define monthly budgets per metric in a “Budgets” sheet.
- Receive an automated weekly “Budget Report” showing where spending exceeds budgets.

### 1.1 Scheduling & Initialization
Weekly trigger starts the workflow, defines a batch size for Bexio pagination, and initializes an offset used to page through the Bexio journal API.

### 1.2 Bexio → Google Sheets Journal Sync (Paginated)
Fetches Bexio journal entries using offset pagination (limit 1000), upserts them into the Google Sheets “Journal” sheet, waits briefly, then increments the offset and repeats.

### 1.3 Budget Input Loop (Per Metric)
Reads all budget rows from the “Budgets” sheet and loops over them. Each row typically represents a metric/category (e.g., “IT costs”, “Rentals”) linked to an “Account ID”.

### 1.4 Actual Cost Calculation (Per Metric, Monthly)
For each budget metric/account:
- pulls matching Journal rows where the account appears as **Debit** and **Credit**
- aggregates monthly totals for credits and debits
- computes monthly “costs” as **credit − debit**

### 1.5 Budget Comparison & Message Generation
Merges the computed monthly actuals with the budget row and outputs either:
- one or more “Budget exceeded…” messages per month, or
- “All costs within the budget”

### 1.6 Final Report Composition & Email Delivery
Aggregates all metric messages into one text report and emails it via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Initial settings
**Overview:** Starts weekly and prepares variables for pagination (batch size and offset).

**Nodes involved:**
- Schedule Trigger
- Batch Size
- Set Offset

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point on a time schedule.
- **Config (interpreted):** Runs every week (interval field: `weeks`).
- **Connections:** Outputs to **Batch Size**.
- **Failure/edge cases:** n8n instance timezone affects perceived “weekly” boundary; missed executions if n8n is down.

#### Node: Batch Size
- **Type / role:** `Set` — defines constant pagination limit.
- **Config:** Sets `batchSize = 1000`.
- **Connections:** Outputs to **Set Offset**.
- **Failure/edge cases:** If Bexio API max page size differs, requests may fail or ignore the value (the workflow also attempts to set `limit` via query parameters—see “Get Records” notes).

#### Node: Set Offset
- **Type / role:** `Set` — initializes/retains the pagination offset.
- **Config:** Sets `offset = {{$json["new_offset"] || 0}}`.
  - First run: `new_offset` not present → offset becomes `0`.
  - Subsequent loop: fed by **Add Offset** via `new_offset`.
- **Execution setting:** `executeOnce: true` (important: it may prevent offset updating across repeated executions within one run, depending on n8n behavior and how the loop is intended).
- **Connections:** Outputs to **Get Records**.
- **Failure/edge cases:**
  - If `executeOnce` prevents re-evaluation when looping, pagination may stick at offset 0.
  - If `new_offset` is not a number, expression may yield unexpected offset.

**Sticky note(s) covering this block:**
- “## Initial settings”

---

### Block 2 — Update accounting records (Bexio → Google Sheets) with pagination loop
**Overview:** Pulls journal entries from Bexio, upserts them into the “Journal” sheet, pauses, then increases offset and repeats.

**Nodes involved:**
- Get Records
- Update Records
- Delay
- Add Offset

#### Node: Get Records
- **Type / role:** `HTTP Request` — calls Bexio accounting journal endpoint.
- **Config (interpreted):**
  - URL: `https://api.bexio.com/3.0/accounting/journal`
  - Auth: Bearer token via predefined credential (`httpBearerAuth`)
  - Query params:
    - `from = 2025-01-01`
    - `to = 2025-12-31`
    - `=limit = =1000`
    - `=offset = {{$json.offset}}`
  - Header: `Accept: application/json`
- **Key concern:** The query parameter names are entered as `=limit` and `=offset` (including the `=`). Unless intentionally required by Bexio (unlikely), this can cause the API to ignore pagination params.
- **Connections:** Input from **Set Offset**; output to **Update Records**.
- **Version-specific:** Node typeVersion 4.2.
- **Failure/edge cases:**
  - 401/403 if token invalid/expired or insufficient scopes.
  - 429 rate limiting if too many requests (mitigated partially by **Delay**).
  - Wrong query param names → only default page returned → incomplete sync.
  - Large responses → memory pressure in n8n if many items.
  - Date range hard-coded to 2025.

#### Node: Update Records
- **Type / role:** `Google Sheets` — append or update rows in “Journal”.
- **Config (interpreted):**
  - Operation: **Append or Update**
  - Matching column: `ID`
  - Target spreadsheet: **Accounting journal (n8n template)**  
    https://docs.google.com/spreadsheets/d/1ZnaXFqOhcrgMptM2K8k-W0VZTxgyL99_D569GiwTZZk/edit
  - Sheet: **Journal** (gid=0)
  - Maps Bexio fields into columns:
    - `ID` = `{{$json.id}}`
    - `Date` = `DateTime.fromISO($json.date).toFormat('yyyy-MM-dd')`
    - `Amount` = `{{$json.amount}}`
    - `Debit ID` / `Credit ID` = corresponding account ids
    - `Sync Date` = current date
    - `Date Formated` = `MM yyyy`
    - `Year Formatted` = `yyyy`
    - `Amount in basic currency` = `{{$json.base_currency_amount}}`
    - etc.
- **Connections:** Input from **Get Records**; output to **Delay**.
- **Version-specific:** typeVersion 4.6.
- **Failure/edge cases:**
  - OAuth permission issues, sheet not shared with the Google account.
  - Schema mismatch (missing columns, renamed headers) breaks mapping/matching.
  - DateTime usage requires Luxon availability in n8n expressions (standard in n8n).
  - If Bexio returns nested objects/arrays, mapping may need adjustment.

#### Node: Delay
- **Type / role:** `Wait` — throttles loop to reduce API pressure.
- **Config:** Waits `amount = 2` (units depend on node defaults; commonly seconds).
- **Execution setting:** `executeOnce: true` (may cause throttling to run only once).
- **Connections:** Output to **Add Offset**.
- **Failure/edge cases:** If “wait” is too short, Bexio API rate limits may still occur.

#### Node: Add Offset
- **Type / role:** `Set` — computes the next offset.
- **Config:** `new_offset = $('Set Offset').item.json.offset + $('Batch Size').item.json.batchSize`
- **Execution setting:** `executeOnce: true` (can interfere with repeated pagination updates).
- **Connections:** Outputs to **Set Offset** (continue pagination) and also to **Read Budgets** (starts budget analysis).
- **Failure/edge cases:**
  - If referenced nodes have no item in scope, expressions can fail.
  - Pagination has no stop condition: there is **no check** for “no more records returned”, so it may loop indefinitely or keep re-syncing the last page.

**Sticky note(s) covering this block:**
- “## Update accounting records Bexio to Google Sheets”

---

### Block 3 — Read budgets & iterate per budget metric
**Overview:** Loads budget definitions from the “Budgets” sheet and processes each row (metric) in a loop.

**Nodes involved:**
- Read Budgets
- Loop Over Items

#### Node: Read Budgets
- **Type / role:** `Google Sheets` — reads budget targets.
- **Config (interpreted):**
  - Spreadsheet: **Accounting journal (n8n template)**
  - Sheet: **Budgets** (gid=1462771045)
  - Reads rows (default read operation for this node, as operation is not explicitly shown).
- **Connections:** Input from **Add Offset**; output to **Loop Over Items**.
- **Failure/edge cases:**
  - If the sheet structure differs (missing “Account ID”, “Metric”, month columns), downstream code will fail or produce empty results.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — iterates over budget rows.
- **Config:** Uses default batch settings (not explicitly set).
- **Connections:**
  - Main output (each batch/item) → **Add Budgets**, **Get Debits**, **Get Credits**
  - “No more items” output → **Compose Report**
  - Also receives feedback loop from **Check Budgets** to continue processing next item.
- **Failure/edge cases:**
  - If “batch size” default is too large/small, performance changes.
  - If any iteration errors, later metrics may not be processed.

**Sticky note(s) covering this block:**
- “## Compare budgets to actual costs” (contextually covers the budget comparison stage, but this loop is the entry into that stage)

---

### Block 4 — Calculate monthly actuals from Journal (per metric/account)
**Overview:** For the current budget row’s Account ID, fetches Journal rows where that account is debit/credit, aggregates monthly totals, and computes monthly costs.

**Nodes involved:**
- Get Debits
- Get Credits
- Calculate Credits
- Calculate debit
- Merge Credits/Debits
- Calculate Costs

#### Node: Get Debits
- **Type / role:** `Google Sheets` — filters Journal rows where account appears in **Debit ID**.
- **Config:**
  - Sheet: **Journal** (gid=0)
  - Filter: `Debit ID` equals `{{$json['Account ID']}}` (value comes from the current budget row).
  - Combine filters: OR (even though only one filter is defined).
- **Connections:** Output to **Calculate Credits** (node naming mismatch; see below).
- **Failure/edge cases:**
  - If “Account ID” field is missing/empty in Budgets row → filter returns none.
  - If Journal column header differs (“Debit ID” renamed) → filter fails.

#### Node: Get Credits
- **Type / role:** `Google Sheets` — filters Journal rows where account appears in **Credit ID**.
- **Config:** Filter `Credit ID` equals `{{$json['Account ID']}}`.
- **Special setting:** `alwaysOutputData: true` (ensures downstream nodes run even if no rows matched).
- **Connections:** Output to **Calculate debit** (again, naming mismatch).
- **Failure/edge cases:** Same as Get Debits; plus if returning empty, downstream code must tolerate empty arrays (it does).

#### Node: Calculate Credits
- **Type / role:** `Code` — aggregates monthly totals into a single object with keys `MM yyyy`.
- **Config (logic):**
  - Detects year from `items[0].json["Date Formated"]` or defaults to current year.
  - Initializes all 12 months to 0.
  - Sums `Number(json["Amount in basic currency"])` grouped by `Date Formated`.
  - Returns one item: `{ "01 2025": 0, ..., "12 2025": 123.45 }`
- **Connections:** Output to **Merge Credits/Debits** (index 0).
- **Edge cases:**
  - If “Date Formated” is missing or not `MM yyyy`, grouping fails (months remain 0).
  - Year inference from first item can be wrong if dataset spans multiple years.
  - Node name suggests credits, but it is fed by **Get Debits**.

#### Node: Calculate debit
- **Type / role:** `Code` — same aggregation logic as above.
- **Connections:** Output to **Merge Credits/Debits** (index 1).
- **Edge cases:** Same as Calculate Credits.
- **Naming mismatch:** This node is fed by **Get Credits**.

#### Node: Merge Credits/Debits
- **Type / role:** `Merge` — combines the two aggregation results into one item set.
- **Config:** Default merge behavior (no explicit mode shown).
- **Connections:** Output to **Calculate Costs**.
- **Edge cases:** If one side doesn’t produce an item (should not happen because both code nodes always return one item), merge can behave unexpectedly depending on merge mode defaults.

#### Node: Calculate Costs
- **Type / role:** `Code` — computes monthly actual “costs”.
- **Config (logic):**
  - `const [credit, debit] = items.map(i => i.json);`
  - For each month: `credit[month] - (debit[month] || 0)`
  - Returns one item with monthly deltas.
- **Connections:** Output to **Add Budgets** (into merge input index 1).
- **Edge cases:**
  - Ordering dependency: assumes first merged item is “credit” and second is “debit”. If upstream naming/order changes, sign flips.
  - Costs formula may need domain validation (some accounting setups might want debit − credit).

**Sticky note(s) covering this block:**
- “## Calculate costs for each period based on accounting records”

---

### Block 5 — Compare budgets to actuals & generate metric messages
**Overview:** Merges the current budget row with computed actuals and creates human-readable exceedance messages.

**Nodes involved:**
- Add Budgets
- Check Budgets

#### Node: Add Budgets
- **Type / role:** `Merge` — combines:
  - Input 0: current budget row from **Loop Over Items**
  - Input 1: computed monthly actuals from **Calculate Costs**
- **Connections:** Output to **Check Budgets**.
- **Edge cases:** Merge behavior depends on default mode; it must output both items so **Check Budgets** can access `items[0]` and `items[1]`.

#### Node: Check Budgets
- **Type / role:** `Code` — compares actuals vs budgets and creates messages.
- **Config (logic):**
  - `const actuals = items[1].json;`
  - `const budget = items[0].json;`
  - Uses `Metric` field as dynamic key: `items[0].json.Metric || "messages"`
  - For each month key in `budget`: if `actuals[month] > budget[month]`, emits message
  - If none exceeded → `["All costs within the budget"]`
  - Output: `{ [Metric]: [messages...] }`
- **Connections:** Output back to **Loop Over Items** (to continue loop and accumulate results).
- **Edge cases / failure types:**
  - If budget row contains non-month columns (e.g., “Metric”, “Account ID”), `budget[month]` may be non-numeric; comparisons may yield false positives/negatives.
  - If budgets are strings (“200”) comparisons work in JS with coercion but can be error-prone; safer to cast numbers.
  - Assumes actuals object has matching month keys.

**Sticky note(s) covering this block:**
- “## Compare budgets to actual costs”

---

### Block 6 — Compose and send email report
**Overview:** After all budget rows are processed, composes a single text report and emails it.

**Nodes involved:**
- Compose Report
- Send Report

#### Node: Compose Report
- **Type / role:** `Code` — builds a plain-text email body from all collected metric message objects.
- **Config (logic):**
  - Iterates over all input items (each one is like `{ "IT costs": [ ... ] }`)
  - Produces:
    - Header “Budget Report:”
    - For each metric: bullet list of messages
  - Outputs one item: `{ email_body: "..." }`
- **Connections:** Output to **Send Report**.
- **Edge cases:** If any metric key maps to non-array, formatting may break; if no items, email will only contain header.

#### Node: Send Report
- **Type / role:** `Gmail` — sends the report.
- **Config (interpreted):**
  - Subject: `Budget Report {{$now.format("dd/LL/yyyy")}}`
  - Body: `{{$json.email_body}}`
  - Email type: text
  - Attribution disabled
  - **Missing required field in JSON shown:** The “To” recipient is not present in the exported parameters; it must be configured in the node UI.
- **Connections:** Terminal node.
- **Failure/edge cases:**
  - OAuth token expired/invalid.
  - Missing “To” address → send fails.
  - Gmail sending limits.

**Sticky note(s) covering this block:**
- “## Send final reports”

---

### Sticky-note-only nodes (documentation content)
These nodes do not execute logic but provide important context:

- Sticky Note (large intro/requirements/setup/customization)
- Sticky Note1 (example report)
- Sticky Note2 (“Initial settings”)
- Sticky Note3 (“Update accounting records Bexio to Google Sheets”)
- Sticky Note4 (“Calculate costs…”)
- Sticky Note5 (“Send final reports”)
- Sticky Note6 (“Compare budgets to actual costs”)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Weekly entry point | — | Batch Size | ## Initial settings |
| Batch Size | Set | Define pagination page size | Schedule Trigger | Set Offset | ## Initial settings |
| Set Offset | Set | Initialize/hold offset for pagination | Batch Size, Add Offset | Get Records | ## Initial settings |
| Get Records | HTTP Request | Fetch Bexio journal page | Set Offset | Update Records | ## Update accounting records Bexio to Google Sheets |
| Update Records | Google Sheets | Upsert journal rows into “Journal” | Get Records | Delay | ## Update accounting records Bexio to Google Sheets |
| Delay | Wait | Throttle between pages | Update Records | Add Offset | ## Update accounting records Bexio to Google Sheets |
| Add Offset | Set | Compute new_offset and branch to budgets | Delay | Set Offset, Read Budgets | ## Update accounting records Bexio to Google Sheets |
| Read Budgets | Google Sheets | Load budget rows from “Budgets” sheet | Add Offset | Loop Over Items | ## Compare budgets to actual costs |
| Loop Over Items | Split In Batches | Iterate per budget metric; collect results | Read Budgets, Check Budgets | Add Budgets, Get Debits, Get Credits; Compose Report | ## Compare budgets to actual costs |
| Get Debits | Google Sheets | Filter Journal by Debit ID = Account ID | Loop Over Items | Calculate Credits | ## Calculate costs for each period based on accounting records |
| Get Credits | Google Sheets | Filter Journal by Credit ID = Account ID | Loop Over Items | Calculate debit | ## Calculate costs for each period based on accounting records |
| Calculate Credits | Code | Monthly aggregation (fed by debits) | Get Debits | Merge Credits/Debits | ## Calculate costs for each period based on accounting records |
| Calculate debit | Code | Monthly aggregation (fed by credits) | Get Credits | Merge Credits/Debits | ## Calculate costs for each period based on accounting records |
| Merge Credits/Debits | Merge | Combine both monthly aggregates | Calculate Credits, Calculate debit | Calculate Costs | ## Calculate costs for each period based on accounting records |
| Calculate Costs | Code | Compute monthly delta (credit − debit) | Merge Credits/Debits | Add Budgets | ## Calculate costs for each period based on accounting records |
| Add Budgets | Merge | Combine budget row + actuals | Loop Over Items, Calculate Costs | Check Budgets | ## Compare budgets to actual costs |
| Check Budgets | Code | Generate exceedance messages per metric | Add Budgets | Loop Over Items | ## Compare budgets to actual costs |
| Compose Report | Code | Build single email body | Loop Over Items | Send Report | ## Send final reports |
| Send Report | Gmail | Send email report | Compose Report | — | ## Send final reports |
| Sticky Note | Sticky Note | Context: purpose/setup/customization | — | — | **This n8n template allows you to automatically monitor your company's budget...** (includes link to the Google Sheet and setup checklist) |
| Sticky Note1 | Sticky Note | Example output formatting | — | — | ## Example report … |
| Sticky Note2 | Sticky Note | Section label | — | — | ## Initial settings |
| Sticky Note3 | Sticky Note | Section label | — | — | ## Update accounting records Bexio to Google Sheets |
| Sticky Note4 | Sticky Note | Section label | — | — | ## Calculate costs for each period based on accounting records |
| Sticky Note5 | Sticky Note | Section label | — | — | ## Send final reports |
| Sticky Note6 | Sticky Note | Section label | — | — | ## Compare budgets to actual costs |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “Schedule Trigger”**
   - Set interval to **every week**.
3. **Add “Set” node: “Batch Size”**
   - Add field `batchSize` (Number) = `1000`.
   - Connect: **Schedule Trigger → Batch Size**.
4. **Add “Set” node: “Set Offset”**
   - Add field `offset` (Number) = expression: `{{$json["new_offset"] || 0}}`
   - (Optional but recommended) disable “Execute once” if you expect pagination to re-evaluate.
   - Connect: **Batch Size → Set Offset**.
5. **Add “HTTP Request” node: “Get Records”**
   - Method: GET
   - URL: `https://api.bexio.com/3.0/accounting/journal`
   - Authentication: **Bearer Auth** credential (create `httpBearerAuth` with your Bexio token).
   - Query parameters (recommended names):
     - `from` = `2025-01-01`
     - `to` = `2025-12-31`
     - `limit` = `1000`
     - `offset` = `{{$json.offset}}`
   - Header: `Accept: application/json`
   - Connect: **Set Offset → Get Records**
6. **Add “Google Sheets” node: “Update Records”**
   - Credential: Google Sheets OAuth2 (connect Google account).
   - Document: your cloned spreadsheet (see notes below).
   - Sheet: **Journal**
   - Operation: **Append or Update**
   - Matching column: `ID`
   - Map columns:
     - `ID` = `{{$json.id}}`
     - `Date` = `{{DateTime.fromISO($json.date).toFormat('yyyy-MM-dd')}}`
     - `Amount` = `{{$json.amount}}`
     - `Debit ID` = `{{$json.debit_account_id}}`
     - `Credit ID` = `{{$json.credit_account_id}}`
     - `Sync Date` = `{{DateTime.now().toFormat('yyyy-MM-dd')}}`
     - `Currency ID` = `{{$json.currency_id}}`
     - `Description` = `{{$json.description}}`
     - `Date Formated` = `{{DateTime.fromISO($json.date).toFormat('MM yyyy')}}`
     - `Year Formatted` = `{{DateTime.fromISO($json.date).toFormat('yyyy')}}`
     - `Conversion factor` = `{{$json.currency_factor}}`
     - `Amount in basic currency` = `{{$json.base_currency_amount}}`
   - Connect: **Get Records → Update Records**
7. **Add “Wait” node: “Delay”**
   - Wait for 2 (seconds, or set explicit unit if available).
   - Connect: **Update Records → Delay**
8. **Add “Set” node: “Add Offset”**
   - Add field `new_offset` (Number) =  
     `{{ $('Set Offset').item.json.offset + $('Batch Size').item.json.batchSize }}`
   - Connect: **Delay → Add Offset**
   - Connect **Add Offset → Set Offset** (to continue pagination)
9. **Add “Google Sheets” node: “Read Budgets”**
   - Credential: same Google Sheets OAuth2.
   - Document: same spreadsheet
   - Sheet: **Budgets**
   - Operation: Read rows (default)
   - Connect: **Add Offset → Read Budgets**
10. **Add “Split In Batches” node: “Loop Over Items”**
    - Default options are acceptable.
    - Connect: **Read Budgets → Loop Over Items**
11. **Add “Google Sheets” node: “Get Debits”**
    - Document: same spreadsheet; Sheet: **Journal**
    - Filter: column **Debit ID** equals `{{$json['Account ID']}}`
    - Connect: **Loop Over Items (main output) → Get Debits**
12. **Add “Google Sheets” node: “Get Credits”**
    - Same but filter column **Credit ID** equals `{{$json['Account ID']}}`
    - Enable “Always output data” (so downstream runs on empty).
    - Connect: **Loop Over Items (main output) → Get Credits**
13. **Add “Code” node: “Calculate Credits”** (monthly aggregation)
    - Paste the monthly initialization + sum code (as in workflow).
    - Connect: **Get Debits → Calculate Credits**
14. **Add “Code” node: “Calculate debit”** (monthly aggregation)
    - Paste the same aggregation code.
    - Connect: **Get Credits → Calculate debit**
15. **Add “Merge” node: “Merge Credits/Debits”**
    - Keep default merge settings (but verify it outputs two items for the next code node).
    - Connect: **Calculate Credits → Merge Credits/Debits (input 0)**
    - Connect: **Calculate debit → Merge Credits/Debits (input 1)**
16. **Add “Code” node: “Calculate Costs”**
    - Paste code computing `credit - debit` per month.
    - Connect: **Merge Credits/Debits → Calculate Costs**
17. **Add “Merge” node: “Add Budgets”**
    - Connect: **Loop Over Items → Add Budgets (input 0)** (budget row)
    - Connect: **Calculate Costs → Add Budgets (input 1)** (actuals)
18. **Add “Code” node: “Check Budgets”**
    - Paste comparison code that emits messages under the `Metric` key.
    - Connect: **Add Budgets → Check Budgets**
    - Connect: **Check Budgets → Loop Over Items** (to continue loop)
19. **Add “Code” node: “Compose Report”**
    - Paste code generating `email_body`.
    - Connect: **Loop Over Items (no more items output) → Compose Report**
20. **Add “Gmail” node: “Send Report”**
    - Credential: Gmail OAuth2.
    - Set **To**: your recipient email address (required).
    - Subject: `Budget Report {{$now.format("dd/LL/yyyy")}}`
    - Message/body: `{{$json.email_body}}`
    - Email type: Text
    - Connect: **Compose Report → Send Report**
21. **Create/prepare the Google Sheet**
    - Must contain at least:
      - **Journal** sheet with headers matching the mapping (ID, Date, Debit ID, Credit ID, Amount in basic currency, Date Formated, etc.).
      - **Budgets** sheet with:
        - `Metric` column (name used in report sections)
        - `Account ID` column (used for Journal filtering)
        - month columns like `01 2025`, `02 2025`, … that match `Date Formated`.
22. **Credentials**
    - Bexio: create Bearer token credential and ensure API access.
    - Google Sheets: OAuth2, spreadsheet shared/owned by that account.
    - Gmail: OAuth2, account allowed to send emails.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically monitors budgets by comparing live Bexio accounting data vs targets in Google Sheets; sends weekly email reports; includes pagination, aggregation, comparison, reporting. | From the main sticky note description in the canvas |
| Spreadsheet: “Accounting journal (n8n template)” | https://docs.google.com/spreadsheets/d/1ZnaXFqOhcrgMptM2K8k-W0VZTxgyL99_D569GiwTZZk/edit |
| Setup checklist: connect Bexio Bearer Token, Google Sheets OAuth2, Gmail; set budgets in “Budgets”; Journal populated automatically; set Gmail “To”. | From the main sticky note |
| Customization: replace Gmail with Slack/Teams; adjust schedule frequency. | From the main sticky note |
| Example report format showing per-metric messages. | From “## Example report” sticky note |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.