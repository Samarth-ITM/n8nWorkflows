Get Gmail alerts for dropped top 10 keyword rankings with DataForSEO

https://n8nworkflows.xyz/workflows/get-gmail-alerts-for-dropped-top-10-keyword-rankings-with-dataforseo-13537


# Get Gmail alerts for dropped top 10 keyword rankings with DataForSEO

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Get email alerts for dropped top 10 keyword rankings with DataForSEO  
**Purpose:** Runs weekly, fetches your domain’s currently ranked keywords from DataForSEO, stores the latest Top 10 snapshot in Google Sheets, compares it with the previous snapshot, detects keywords that dropped out of Top 1 / Top 3 / Top 10, enriches “dropped” keywords with fresh SERP competitor results, and sends a structured Gmail summary email.

### 1.1 Scheduling & Previous Snapshot Retrieval
- Triggered weekly (Monday at 09:00).
- Reads the previous snapshot (Keyword + Rank) from Google Sheets and aggregates it for later comparison.

### 1.2 Clear Snapshot & Fetch Current Ranked Keywords (Paged)
- Clears the sheet (keeps header).
- Calls DataForSEO Labs “get-ranked-keywords” for the target domain, paging 1000 results per runIndex.
- Accumulates all pages into a single `items` array.

### 1.3 Keep Current Top 10 + Save New Snapshot
- Splits accumulated `items`.
- Filters to keep rank positions 1–10 only.
- Appends these rows (Keyword, Rank) to Google Sheets as the new snapshot.
- Computes Top1/Top3/Top10 keyword arrays (from the new snapshot) for comparison.

### 1.4 Compare With Previous Snapshot & Classify Drops
- If previous snapshot data exists, split previous keywords into individual rows.
- Switches by the previous rank bucket (Top1 vs Top3 vs Top10).
- For each bucket: keep only keywords that are *not* present in the current corresponding bucket (meaning they dropped out of that bucket).

### 1.5 SERP Enrichment & Email Assembly
- For each dropped keyword (per bucket), fetch live Google SERP via DataForSEO SERP API at depths 1/3/10.
- Builds HTML tables with competitor domains/URLs/titles.
- Aggregates per bucket and sends one Gmail email with all sections.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Load Previous Snapshot
**Overview:** Starts on a weekly schedule and loads the prior week’s snapshot from Google Sheets to enable “drop” detection later.  
**Nodes involved:** `Schedule Trigger`, `Get previous keywords`, `Aggregate`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Config:** Weekly; `triggerAtDay: [1]` (Monday), `triggerAtHour: 9`.
- **Outputs:** To `Get previous keywords`.
- **Failure modes:** n8n timezone mismatch, missed executions if instance down.

#### Node: Get previous keywords
- **Type / role:** `n8n-nodes-base.googleSheets` — reads existing snapshot.
- **Config:** Reads from Spreadsheet **Dropped Keywords from Top10/3/1**, sheet tab **Keywords**.
- **Credentials:** Google Sheets OAuth2.
- **Special setting:** `alwaysOutputData: true` (workflow continues even if sheet returns no rows).
- **Outputs:** To `Aggregate`.
- **Failure modes:** OAuth expiration, spreadsheet permission issues, sheet/tab renamed, API quota.

#### Node: Aggregate
- **Type / role:** `n8n-nodes-base.aggregate` — collects all rows into one item.
- **Config:** `aggregateAllItemData` into `data` (default aggregate output structure).
- **Outputs:** To `Clear sheet (previous keywords)`.
- **Failure modes:** Large sheets can increase memory usage.

---

### Block 2 — Clear Sheet & Initialize Current Items Accumulator
**Overview:** Clears the sheet for the new snapshot (keeps headers), and initializes an accumulator array `items` used to store all paged DataForSEO results.  
**Nodes involved:** `Clear sheet (previous keywords)`, `Initialize "items" field`, `Set "items" field`

#### Node: Clear sheet (previous keywords)
- **Type / role:** `n8n-nodes-base.googleSheets` — clears contents for refresh.
- **Config:** Operation `clear`, `keepFirstRow: true` on the same sheet.
- **Outputs:** To `Initialize "items" field`.
- **Failure modes:** Clearing wrong sheet if documentId/sheetName changed; permission issues.

#### Node: Initialize "items" field
- **Type / role:** `n8n-nodes-base.set` — creates empty accumulator.
- **Config:** sets `items` (array) to `[]`.
- **Outputs:** To `Set "items" field`.
- **Failure modes:** None typical (expression errors only).

#### Node: Set "items" field
- **Type / role:** `n8n-nodes-base.set` — carries accumulator forward per loop.
- **Config:** sets `items` to `{{$json.items}}`.
- **Outputs:** To `Get ranked keywords`.
- **Edge cases:** If upstream item lacks `items`, expression resolves to `undefined`; here it’s safe because it is initialized.

---

### Block 3 — Fetch Ranked Keywords from DataForSEO (Paged) & Accumulate
**Overview:** Calls DataForSEO Labs for ranked keywords, 1000 at a time, and accumulates all pages into one `items` array.  
**Nodes involved:** `Get ranked keywords`, `Merge "items" with DFS response`, `Has more pages?`, `Merge "items" with last response`

#### Node: Get ranked keywords
- **Type / role:** `n8n-nodes-dataforseo.dataForSeoLabsApi` — retrieves ranked keywords for a domain.
- **Config (interpreted):**
  - Operation: `get-ranked-keywords`
  - Target domain: `dataforseo.com` (parameter `target_any`)
  - Location: United States; Language: English
  - Pagination: `limit: 1000`, `offset: {{$runIndex * 1000}}`
- **Credentials:** DataForSEO API (login/password).
- **Outputs:** To `Merge "items" with DFS response`.
- **Failure modes:** Invalid credentials, API rate limits, plan limits, empty results, location/language mismatch.

#### Node: Merge "items" with DFS response
- **Type / role:** `n8n-nodes-base.set` — appends current page results to accumulator.
- **Key expression:**
  - `items = [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items ]`
- **Outputs:** To `Has more pages?`.
- **Edge cases:** If DataForSEO returns no `tasks[0].result[0].items`, expression may throw; consider guarding with `|| []`.

#### Node: Has more pages?
- **Type / role:** `n8n-nodes-base.if` — controls paging loop.
- **Logic:** Checks whether `$runIndex < (total_count / 1000 - 1)`.
  - Left: `{{$runIndex}}`
  - Right: `{{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 -1 }}`
- **Outputs:**
  - **True:** to `Set "items" field` (fetch next page)
  - **False:** to `Merge "items" with last response` (finalize)
- **Edge cases:** `total_count` missing or `0` can break math; floating values can cause off-by-one; prefer `Math.ceil(total_count/1000)`.

#### Node: Merge "items" with last response
- **Type / role:** `n8n-nodes-base.set` — final merge ensuring latest page included.
- **Key expression:**
  - `items = [...$('Set "items" field').item.json.items, ...$('Get ranked keywords').item.json.tasks[0].result[0].items]`
- **Outputs:** To `Split out (items)`.

---

### Block 4 — Keep Current Top 10 & Write New Snapshot to Sheets
**Overview:** Splits accumulated items, filters to Top 10 positions only, appends them into Google Sheets, and computes “Top1/Top3/Top10” fields for later comparison.  
**Nodes involved:** `Split out (items)`, `Filter items with rank > 10`, `Append row in sheet`, `Set Top 1, Top 3, Top 10`, `Aggregate1`, `Filter (data for comparing is not empty)`, `Filter Top 1, Top 2, Top 3`

#### Node: Split out (items)
- **Type / role:** `n8n-nodes-base.splitOut` — converts `items[]` into individual items.
- **Config:** `fieldToSplitOut: items`
- **Outputs:** To `Filter items with rank > 10`.
- **Edge cases:** If `items` is empty/missing, produces no output items.

#### Node: Filter items with rank > 10
- **Type / role:** `n8n-nodes-base.filter` — retains only ranks 1–10.
- **Condition:** `rank_group < 11` using:
  - Left: `{{$json.ranked_serp_element.serp_item.rank_group}}`
- **Outputs:** To `Append row in sheet`.
- **Note:** Node name suggests “rank > 10” but condition keeps **Top 10**.

#### Node: Append row in sheet
- **Type / role:** `n8n-nodes-base.googleSheets` — writes new snapshot rows.
- **Config:** Operation `append` into sheet tab **Keywords** with mapping:
  - `Rank = {{ $('Split out (items)').item.json.ranked_serp_element.serp_item.rank_group }}`
  - `Keyword = {{ $('Split out (items)').item.json.keyword_data.keyword }}`
- **Outputs:** To `Set Top 1, Top 3, Top 10`.
- **Failure modes:** Sheet schema mismatch (missing columns Keyword/Rank), quota, permission issues.

#### Node: Set Top 1, Top 3, Top 10
- **Type / role:** `n8n-nodes-base.set` — marks bucket membership for each Top10 keyword.
- **Key expressions (per item):**
  - `top1`: keyword if `rank_group == 1` else `null`
  - `top3`: keyword if `rank_group < 4` else `null`
  - `=top10` **(field name includes leading `=`)**: keyword if `rank_group < 11` else `null`
- **Outputs:** To `Aggregate1`.
- **Important edge case:** The third field is named `=top10`, but later nodes expect `top10`. This mismatch can cause Top10 comparisons to fail (or become empty). Fix by renaming the field to `top10`.

#### Node: Aggregate1
- **Type / role:** `n8n-nodes-base.aggregate` — aggregates all current Top10 items into a single `data` array.
- **Outputs:** To `Filter (data for comparing is not empty)`.

#### Node: Filter (data for comparing is not empty)
- **Type / role:** `n8n-nodes-base.filter` — ensures comparison is possible.
- **Condition:** `$('Aggregate').item.json.data` is `notEmpty`.
  - This checks **previous snapshot** exists before attempting drop detection.
- **Outputs:** To `Filter Top 1, Top 2, Top 3`.
- **Edge cases:** If previous sheet is empty (first run), the workflow stops here and sends no email.

#### Node: Filter Top 1, Top 2, Top 3
- **Type / role:** `n8n-nodes-base.set` — builds arrays of current keywords per bucket and injects previousItems.
- **Key expressions:**
  - `top1 = $json.data.map(item => item.top1).filter(item => item !== null)`
  - `top3 = $json.data.map(item => item.top3).filter(item => item !== null)`
  - `top10 = $json.data.map(item => item.top10).filter(item => item !== null)` (will break if field is `=top10`)
  - `previousItems = $('Aggregate').item.json.data`
- **Outputs:** To `Split out (previous keywords)`.

---

### Block 5 — Compare Previous Snapshot vs Current Snapshot & Route by Bucket
**Overview:** Iterates over last week’s keywords, classifies them by their previous rank bucket, and keeps only those that dropped out of the corresponding current bucket.  
**Nodes involved:** `Split out (previous keywords)`, `Switch`, `Filter (dropped from Top 1)`, `Filter (dropped from Top 3)`, `Filter (dropped from Top 10)`

#### Node: Split out (previous keywords)
- **Type / role:** `n8n-nodes-base.splitOut` — turns `previousItems[]` into individual items.
- **Expected input fields:** Each item should contain `Keyword` and `Rank` columns (as read from Sheets).
- **Outputs:** To `Switch`.
- **Edge cases:** If sheet headers are different (e.g., `keyword` not `Keyword`), downstream expressions fail.

#### Node: Switch
- **Type / role:** `n8n-nodes-base.switch` — routes based on previous Rank.
- **Rules / outputs:**
  1. **previous Top 1**: `Rank == 1` → output 0 → `Filter (dropped from Top 1)`
  2. **privious Top 3**: `Rank < 4` → output 1 → `Filter (dropped from Top 3)`
  3. **previous Top 10**: `Rank < 11` → output 2 → `Filter (dropped from Top 10)`
- **Edge cases:** Overlap: Rank=1 also satisfies Rank<4 and Rank<11; Switch rule order matters. In n8n Switch, matching behavior depends on configuration; ensure it stops at first match (typical) to avoid duplicates.

#### Node: Filter (dropped from Top 1)
- **Type / role:** `n8n-nodes-base.filter` — keeps keywords not in current `top1`.
- **Condition:** `notContains( currentTop1Array, $json.Keyword )`.
- **Outputs:** To `Loop Over Top 1 Items`.

#### Node: Filter (dropped from Top 3)
- **Type / role:** `n8n-nodes-base.filter` — keeps keywords not in current `top3`.
- **Outputs:** To `Loop Over Top 2 Items`.

#### Node: Filter (dropped from Top 10)
- **Type / role:** `n8n-nodes-base.filter` — keeps keywords not in current `top10`.
- **Outputs:** To `Loop Over Top 3 Items`.
- **Edge case:** If `top10` array is empty due to the `=top10` naming bug, this filter will treat **every** previous Top10 keyword as “dropped”.

---

### Block 6 — SERP Enrichment for Dropped Keywords (Top1 / Top3 / Top10)
**Overview:** For each dropped keyword bucket, fetches live Google organic SERP results at different depths, formats each SERP item into an HTML row, aggregates them, and produces a per-keyword HTML section.  
**Nodes involved:**  
Top1 path: `Loop Over Top 1 Items`, `Get live google organic SERP regular`, `Split out (SERP results)`, `Prepare email content`, `Aggregate2`, `Prepare email content1`  
Top3 path: `Loop Over Top 2 Items`, `Get live google organic SERP regular1`, `Split out (SERP results)1`, `Prepare email content2`, `Aggregate3`, `Prepare email content3`  
Top10 path: `Loop Over Top 3 Items`, `Get live google organic SERP regular2`, `Split out (SERP results)2`, `Prepare email content4`, `Aggregate4`, `Prepare email content5`

#### Node: Loop Over Top 1 Items / Loop Over Top 2 Items / Loop Over Top 3 Items
- **Type / role:** `n8n-nodes-base.splitInBatches` — batch iteration.
- **Config:** default batch size (not set).
- **Connections:** Each has **two outputs**:
  - Output 0 → `Merge` (used later to combine the three groups)
  - Output 1 → corresponding SERP fetch node
- **Edge cases:** If no dropped items in a bucket, that branch produces no items; downstream Merge may wait depending on merge mode.

#### Node: Get live google organic SERP regular / regular1 / regular2
- **Type / role:** `n8n-nodes-dataforseo.dataForSeoSerpApi` — fetches live SERP.
- **Config differences:**
  - Top1 branch: `depth: 1`
  - Top3 branch: `depth: 3`
  - Top10 branch: `depth: 10`
- **Keyword:** `{{$json.Keyword}}` (from previous snapshot row)
- **Language/location:** pulled from the earlier Labs response:
  - `language_name = {{ $('Get ranked keywords').item.json.tasks[0].data.language_name }}`
  - `location_name = {{ $('Get ranked keywords').item.json.tasks[0].data.location_name }}`
- **Outputs:** To corresponding split-out node.
- **Failure modes:** API quota/rate limits, SERP task errors, location/language invalid, timeouts.

#### Node: Split out (SERP results) / (SERP results)1 / (SERP results)2
- **Type / role:** `n8n-nodes-base.splitOut` — one item per SERP entry.
- **Config:** `tasks[0].result[0].items`.
- **Outputs:** To corresponding “Prepare email content” node.
- **Edge cases:** If SERP returns no items, nothing downstream; consider fallback “no competitors found”.

#### Node: Prepare email content / Prepare email content2 / Prepare email content4
- **Type / role:** `n8n-nodes-base.set` — formats one SERP row as HTML.
- **Creates field:** `competitor` containing:
  - `<tr><td>{{domain}}</td><td>{{url}}</td><td>{{rank_group}}</td><td>{{title}}</td><td>{{description}}</td></tr>`
- **Outputs:** To Aggregate2/Aggregate3/Aggregate4 respectively.

#### Node: Aggregate2 / Aggregate3 / Aggregate4
- **Type / role:** `n8n-nodes-base.aggregate` — collects competitor rows into `data[]`.
- **Outputs:** To `Prepare email content1` / `Prepare email content3` / `Prepare email content5`.
- **Edge cases:** Very large SERP depth could create many rows; here limited to 1/3/10.

#### Node: Prepare email content1 (Top1 section) / Prepare email content3 (Top3 section) / Prepare email content5 (Top10 section)
- **Type / role:** `n8n-nodes-base.set` — composes an HTML block for one dropped keyword including competitor table.
- **Key elements:**
  - Uses loop item fields (e.g., `$('Loop Over Top 1 Items').item.json.Keyword` and `.Rank`)
  - Attempts to compute “Current Rank” by searching the final accumulated items:
    - `$('Merge "items" with last response').item.json.items.filter(...).map(...).first() || null`
- **Outputs:** Back into the corresponding Loop node (`Loop Over Top X Items`), enabling iteration.
- **Edge cases / failures:**
  - `.first()` is not standard JS; in n8n, this only works if n8n extends arrays (it often doesn’t). Safer: `[0]` after mapping, e.g. `...map(...)[0]`.
  - If keyword not found in current items, current rank becomes `null` (expected for “dropped out”, but could also indicate missing data).

---

### Block 7 — Merge Sections & Send Gmail
**Overview:** Merges the three bucket streams (Top1/Top3/Top10), aggregates the HTML blocks, and sends a single email.  
**Nodes involved:** `Merge`, `Aggregate5`, `Send a message`

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines 3 inputs.
- **Config:** `numberInputs: 3`
- **Inputs:** from:
  - `Loop Over Top 1 Items` (index 0)
  - `Loop Over Top 2 Items` (index 1)
  - `Loop Over Top 3 Items` (index 2)
- **Outputs:** To `Aggregate5`.
- **Edge cases:** If one branch produces no items, merge behavior can stall depending on merge settings; verify merge mode in UI and consider “Pass-through” patterns if needed.

#### Node: Aggregate5
- **Type / role:** `n8n-nodes-base.aggregate` — gathers all merged items into `data[]` for email templating.
- **Outputs:** To `Send a message`.

#### Node: Send a message
- **Type / role:** `n8n-nodes-base.gmail` — sends HTML email.
- **Config:**
  - To: `user@example.com`
  - Subject: `Dropped keywords`
  - Message is HTML and uses:
    - `{{ $json.data.map(item => item.top1).join('') }}`
    - `{{ $json.data.map(item => item.top3).join('') }}`
    - `{{ $json.data.map(item => item.top10).join('') }}`
- **Credentials:** Gmail OAuth2.
- **Edge cases:**
  - If `top1/top3/top10` fields are absent in merged items, sections render empty.
  - Gmail may sanitize some HTML; tables generally ok.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Weekly entry point | — | Get previous keywords | This workflow uses the DataForSEO APIs to spot keywords for which your domain dropped out of Google’s top 10 results. It compares the latest rankings with the previous week’s snapshot, enriches each lost keyword with fresh SERP data and competitor insights, and sends you a structured Gmail alert. / Setup steps include DataForSEO API access: https://app.dataforseo.com/api-access |
| Get previous keywords | Google Sheets | Read previous snapshot | Schedule Trigger | Aggregate | ## Get previous keywords and clear the sheet / Example sheet: https://docs.google.com/spreadsheets/d/10G-tFbJC__V6_dBjGTGYR50DQPwUtX29_M3zohTz4WE/edit?usp=sharing |
| Aggregate | Aggregate | Aggregate previous snapshot rows | Get previous keywords | Clear sheet (previous keywords) | ## Get previous keywords and clear the sheet / Example sheet: https://docs.google.com/spreadsheets/d/10G-tFbJC__V6_dBjGTGYR50DQPwUtX29_M3zohTz4WE/edit?usp=sharing |
| Clear sheet (previous keywords) | Google Sheets | Clear sheet to store new snapshot | Aggregate | Initialize "items" field | ## Get previous keywords and clear the sheet / Example sheet: https://docs.google.com/spreadsheets/d/10G-tFbJC__V6_dBjGTGYR50DQPwUtX29_M3zohTz4WE/edit?usp=sharing |
| Initialize "items" field | Set | Init accumulator array | Clear sheet (previous keywords) | Set "items" field | ## Get ranked keywords keywords with DataForSEO / Create a DataForSEO connection and specify the Target Domain, Location, and Language. |
| Set "items" field | Set | Carry accumulator between pages | Initialize "items" field; Has more pages? | Get ranked keywords | ## Get ranked keywords keywords with DataForSEO / Create a DataForSEO connection and specify the Target Domain, Location, and Language. |
| Get ranked keywords | DataForSEO Labs API | Fetch ranked keywords (paged) | Set "items" field | Merge "items" with DFS response | ## Get ranked keywords keywords with DataForSEO / Create a DataForSEO connection and specify the Target Domain, Location, and Language. |
| Merge "items" with DFS response | Set | Append DFS page items to accumulator | Get ranked keywords | Has more pages? | ## Get ranked keywords keywords with DataForSEO / Create a DataForSEO connection and specify the Target Domain, Location, and Language. |
| Has more pages? | IF | Decide to fetch next page or finalize | Merge "items" with DFS response | Set "items" field; Merge "items" with last response | ## Get ranked keywords keywords with DataForSEO / Create a DataForSEO connection and specify the Target Domain, Location, and Language. |
| Merge "items" with last response | Set | Finalize accumulator | Has more pages? (false) | Split out (items) | ## Get ranked keywords keywords with DataForSEO / Create a DataForSEO connection and specify the Target Domain, Location, and Language. |
| Split out (items) | Split Out | One item per keyword | Merge "items" with last response | Filter items with rank > 10 | ## Add new Top 10 keywords to Google Sheets and calculate Top 1, Top 3, and Top 10 results |
| Filter items with rank > 10 | Filter | Keep ranks 1–10 | Split out (items) | Append row in sheet | ## Add new Top 10 keywords to Google Sheets and calculate Top 1, Top 3, and Top 10 results |
| Append row in sheet | Google Sheets | Write new snapshot rows | Filter items with rank > 10 | Set Top 1, Top 3, Top 10 | ## Add new Top 10 keywords to Google Sheets and calculate Top 1, Top 3, and Top 10 results |
| Set Top 1, Top 3, Top 10 | Set | Tag each keyword into top buckets | Append row in sheet | Aggregate1 | ## Add new Top 10 keywords to Google Sheets and calculate Top 1, Top 3, and Top 10 results |
| Aggregate1 | Aggregate | Aggregate current snapshot tags | Set Top 1, Top 3, Top 10 | Filter (data for comparing is not empty) | ## Add new Top 10 keywords to Google Sheets and calculate Top 1, Top 3, and Top 10 results |
| Filter (data for comparing is not empty) | Filter | Stop if no previous snapshot | Aggregate1 | Filter Top 1, Top 2, Top 3 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Filter Top 1, Top 2, Top 3 | Set | Build current bucket arrays + attach previousItems | Filter (data for comparing is not empty) | Split out (previous keywords) | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Split out (previous keywords) | Split Out | Iterate last week’s snapshot | Filter Top 1, Top 2, Top 3 | Switch | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Switch | Switch | Route by previous rank bucket | Split out (previous keywords) | Filter (dropped from Top 1); Filter (dropped from Top 3); Filter (dropped from Top 10) | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Filter (dropped from Top 1) | Filter | Detect Top1 drops | Switch | Loop Over Top 1 Items | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Loop Over Top 1 Items | Split In Batches | Iterate Top1 dropped keywords | Filter (dropped from Top 1); Prepare email content1 | Merge; Get live google organic SERP regular | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Get live google organic SERP regular | DataForSEO SERP API | Fetch SERP depth=1 | Loop Over Top 1 Items | Split out (SERP results) | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Split out (SERP results) | Split Out | One item per SERP result | Get live google organic SERP regular | Prepare email content | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Prepare email content | Set | Format competitor row (Top1 branch) | Split out (SERP results) | Aggregate2 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Aggregate2 | Aggregate | Collect competitor rows (Top1) | Prepare email content | Prepare email content1 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Prepare email content1 | Set | Build HTML block (Top1) | Aggregate2 | Loop Over Top 1 Items | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Filter (dropped from Top 3) | Filter | Detect Top3 drops | Switch | Loop Over Top 2 Items | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Loop Over Top 2 Items | Split In Batches | Iterate Top3 dropped keywords | Filter (dropped from Top 3); Prepare email content3 | Merge; Get live google organic SERP regular1 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Get live google organic SERP regular1 | DataForSEO SERP API | Fetch SERP depth=3 | Loop Over Top 2 Items | Split out (SERP results)1 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Split out (SERP results)1 | Split Out | One item per SERP result | Get live google organic SERP regular1 | Prepare email content2 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Prepare email content2 | Set | Format competitor row (Top3 branch) | Split out (SERP results)1 | Aggregate3 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Aggregate3 | Aggregate | Collect competitor rows (Top3) | Prepare email content2 | Prepare email content3 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Prepare email content3 | Set | Build HTML block (Top3) | Aggregate3 | Loop Over Top 2 Items | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Filter (dropped from Top 10) | Filter | Detect Top10 drops | Switch | Loop Over Top 3 Items | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Loop Over Top 3 Items | Split In Batches | Iterate Top10 dropped keywords | Filter (dropped from Top 10); Prepare email content5 | Merge; Get live google organic SERP regular2 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Get live google organic SERP regular2 | DataForSEO SERP API | Fetch SERP depth=10 | Loop Over Top 3 Items | Split out (SERP results)2 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Split out (SERP results)2 | Split Out | One item per SERP result | Get live google organic SERP regular2 | Prepare email content4 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Prepare email content4 | Set | Format competitor row (Top10 branch) | Split out (SERP results)2 | Aggregate4 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Aggregate4 | Aggregate | Collect competitor rows (Top10) | Prepare email content4 | Prepare email content5 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Prepare email content5 | Set | Build HTML block (Top10) | Aggregate4 | Loop Over Top 3 Items | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Merge | Merge | Combine 3 bucket streams | Loop Over Top 1 Items; Loop Over Top 2 Items; Loop Over Top 3 Items | Aggregate5 | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Aggregate5 | Aggregate | Gather all HTML blocks for email | Merge | Send a message | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |
| Send a message | Gmail | Send summary email | Aggregate5 | — | ## Find keywords that dropped from Top 1, Top 3, Top 10. Prepare data for email and send it |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set to **Weekly**, Monday, **09:00** (adjust timezone as needed).

2. **Create Google Sheets: Read previous snapshot**
   - Node: **Google Sheets** named `Get previous keywords`
   - Select Spreadsheet and Sheet tab (e.g., `Keywords`)
   - Ensure the sheet has columns **Keyword** and **Rank**
   - Enable **Always Output Data**.

3. **Aggregate previous snapshot**
   - Node: **Aggregate** named `Aggregate`
   - Mode: **Aggregate All Item Data** (produces `data[]`).

4. **Clear the sheet for new snapshot**
   - Node: **Google Sheets** named `Clear sheet (previous keywords)`
   - Operation: **Clear**
   - Same spreadsheet/tab as above
   - Enable **Keep First Row** (header).

5. **Initialize accumulator**
   - Node: **Set** named `Initialize "items" field`
   - Add field: `items` (Type: Array) = `[]`.

6. **Accumulator passthrough**
   - Node: **Set** named `Set "items" field`
   - Set `items` = `{{$json.items}}`.

7. **Fetch ranked keywords from DataForSEO Labs**
   - Node: **DataForSEO Labs API** named `Get ranked keywords`
   - Operation: **get-ranked-keywords**
   - `target_any`: your domain (e.g., `example.com`)
   - `language_name`: e.g., `english`
   - `location_name`: e.g., `united states`
   - `limit`: `1000`
   - `offset`: `{{$runIndex * 1000}}`
   - Configure **DataForSEO credentials** (API login + password from https://app.dataforseo.com/api-access).

8. **Append page results to accumulator**
   - Node: **Set** named `Merge "items" with DFS response`
   - Set `items` to: combine prior `items` with `tasks[0].result[0].items`.

9. **Paging control**
   - Node: **IF** named `Has more pages?`
   - Condition: `$runIndex < (total_count/1000 - 1)` (or implement a safer `Math.ceil`-based check).
   - True path → back to `Set "items" field` (step 6)
   - False path → proceed.

10. **Finalize accumulator**
   - Node: **Set** named `Merge "items" with last response`
   - Set `items` = prior items + last response items.

11. **Split accumulated items**
   - Node: **Split Out** named `Split out (items)`
   - Field: `items`.

12. **Filter to Top 10**
   - Node: **Filter** named `Filter items with rank > 10`
   - Condition: `rank_group < 11` using field `ranked_serp_element.serp_item.rank_group`.

13. **Append new snapshot rows into Google Sheets**
   - Node: **Google Sheets** named `Append row in sheet`
   - Operation: **Append**
   - Map:
     - `Keyword` = `{{$json.keyword_data.keyword}}`
     - `Rank` = `{{$json.ranked_serp_element.serp_item.rank_group}}`

14. **Compute bucket fields per item**
   - Node: **Set** named `Set Top 1, Top 3, Top 10`
   - Create fields:
     - `top1` (string): keyword if rank==1 else null
     - `top3` (string): keyword if rank<4 else null
     - `top10` (string): keyword if rank<11 else null  
   - Important: do **not** name it `=top10`.

15. **Aggregate current snapshot**
   - Node: **Aggregate** named `Aggregate1`
   - Aggregate All Item Data → `data[]`.

16. **Guard: only compare if previous snapshot exists**
   - Node: **Filter** named `Filter (data for comparing is not empty)`
   - Condition: previous aggregate `$('Aggregate').item.json.data` is not empty.

17. **Build arrays for comparison**
   - Node: **Set** named `Filter Top 1, Top 2, Top 3`
   - `top1/top3/top10` arrays from current `data[]`
   - `previousItems = $('Aggregate').item.json.data`.

18. **Split previous snapshot**
   - Node: **Split Out** named `Split out (previous keywords)`
   - Field: `previousItems`.

19. **Route by previous rank**
   - Node: **Switch** named `Switch`
   - Rules:
     - Rank == 1 → Top1 path
     - Rank < 4 → Top3 path
     - Rank < 11 → Top10 path
   - Ensure it doesn’t duplicate matches (rule order / stop-on-first-match).

20. **For each path, filter for dropped keywords**
   - Create three **Filter** nodes:
     - Dropped Top1: previous Keyword not in current `top1` array
     - Dropped Top3: not in current `top3`
     - Dropped Top10: not in current `top10`

21. **For each path, iterate + fetch SERP**
   - Add **Split In Batches** per path (Top1/Top3/Top10).
   - Add **DataForSEO SERP API** per path with depth 1/3/10, keyword = `{{$json.Keyword}}`.
   - Language/location: reuse from Labs response or hardcode consistently.

22. **Split SERP items and format competitor rows**
   - Split Out `tasks[0].result[0].items`
   - Set node to create `competitor` HTML row.

23. **Aggregate competitor rows**
   - Aggregate node to get `data[]` of competitor rows.

24. **Build per-keyword HTML block**
   - Set node to assemble HTML for that dropped keyword + competitor table.
   - Replace any unsupported `.first()` usage with `[0]` indexing.

25. **Merge 3 paths and send email**
   - Merge node with 3 inputs.
   - Aggregate all merged items.
   - Gmail node:
     - OAuth2 credentials
     - To/Subject
     - HTML body concatenating `top1/top3/top10` blocks.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DataForSEO API credentials (login/password) are required to use Labs + SERP endpoints. | https://app.dataforseo.com/api-access |
| Google Sheets file must include the same columns as the provided example (notably `Keyword` and `Rank`). | https://docs.google.com/spreadsheets/d/10G-tFbJC__V6_dBjGTGYR50DQPwUtX29_M3zohTz4WE/edit?usp=sharing |
| The workflow description notes it sends a structured Gmail alert and logs dropped keywords into Sheets. | Sticky note in canvas (“How it works / Setup steps”). |

