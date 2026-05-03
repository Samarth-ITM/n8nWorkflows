Track new ranked keywords in Google Sheets with DataForSEO and Slack alerts

https://n8nworkflows.xyz/workflows/track-new-ranked-keywords-in-google-sheets-with-dataforseo-and-slack-alerts-13332


# Track new ranked keywords in Google Sheets with DataForSEO and Slack alerts

## 1. Workflow Overview

**Purpose:**  
This workflow runs weekly to detect **new Google-ranked keywords** for one or more target domains using **DataForSEO Labs API**, then **stores the latest full keyword set in Google Sheets** and sends a **Slack alert listing only newly discovered keywords** per domain.

**Typical use cases:**
- Weekly SEO monitoring for new keywords a site starts ranking for
- Lightweight “keyword discovery” alerting for multiple domains/markets (location + language)
- Building a historical log of newly ranked keywords while keeping a “current state” sheet refreshed

### 1.1 Schedule Trigger & Baseline Load (previous run)
- Runs every Monday at 09:00.
- Reads the existing keyword rows from Google Sheets (baseline from last run), aggregates them, then clears the sheet to repopulate it with current results.

### 1.2 Target Loop (domains to check)
- Loads a “Targets” sheet containing (at least) Domain, Location, Language.
- Iterates through each target row.

### 1.3 DataForSEO Pagination & Item Accumulation
- Calls DataForSEO “get-ranked-keywords” for each target, paging in chunks of 1000 using `$runIndex`.
- Merges pages into a growing `items` array.

### 1.4 Save Current Keywords + Detect New Ones
- Splits accumulated items into individual rows and appends them into the Keywords sheet (rebuilding the “current state”).
- Aggregates saved items, compares with baseline, produces `diff` = newly ranked keywords.

### 1.5 Slack Notification
- If `diff` is not empty, sends a Slack message listing new keywords for that domain.
- Continues loop to next target.

---

## 2. Block-by-Block Analysis

### Block A — Schedule, Load Previous Keywords, Clear Sheet
**Overview:** Loads the previous run’s keyword list from Google Sheets (used as comparison baseline), then clears the Keywords sheet (keeping headers) so it can be repopulated with the latest DataForSEO results.

**Nodes involved:**
- Run every Monday
- Get previous keywords
- Aggregate
- Clear sheet (Keywords)

#### Node: **Run every Monday**
- **Type / role:** Schedule Trigger — entry point.
- **Config:** Weekly schedule; triggers on **Monday** at **09:00**.
- **Outputs:** Starts the workflow.
- **Edge cases:** Timezone depends on n8n instance settings; verify schedule timezone to avoid off-by-hours.

#### Node: **Get previous keywords**
- **Type / role:** Google Sheets — read existing “Keywords” sheet rows.
- **Config choices:**
- Reads from Spreadsheet **“New Ranked Keywords”** (documentId) and sheet **“Keywords”** (gid 1681593026).
- `alwaysOutputData: true` ensures downstream nodes still execute even if the sheet is empty.
- **Outputs:** Items = rows from the Keywords sheet (each row as JSON).
- **Edge cases / failures:**
  - OAuth permission issues, missing spreadsheet access.
  - Empty sheet: outputs no rows but still continues (important for first run).

#### Node: **Aggregate**
- **Type / role:** Aggregate — collapses all rows into one item containing all row JSON in `data`.
- **Config:** `aggregateAllItemData`.
- **Input:** Rows from “Get previous keywords”.
- **Output:** Single item like `{ data: [ {Target, Keyword, ...}, ...] }`.
- **Edge cases:** If no input rows, `data` may be missing; later code checks for existence.

#### Node: **Clear sheet (Keywords)**
- **Type / role:** Google Sheets — clears the Keywords sheet to rebuild the “current snapshot”.
- **Config:** Operation `clear`, `keepFirstRow: true` (keeps headers).
- **Input:** The aggregated baseline (not strictly required, but used as sequencing).
- **Output:** Confirmation metadata from Sheets.
- **Edge cases / failures:** Clearing requires edit permissions; if the sheet is protected, this fails.

**Sticky note coverage (applies to this block’s Sheets nodes):**  
“Get previous keywords and clear the sheet … must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=1681593026#gid=1681593026).”

---

### Block B — Load Targets & Loop
**Overview:** Reads target domains (plus location/language) and iterates over them one by one.

**Nodes involved:**
- Get targets
- Loop Over Targets

#### Node: **Get targets**
- **Type / role:** Google Sheets — loads target configuration rows.
- **Config:** Reads sheet **“Targets”** (gid=0) from the same spreadsheet.
- **Expected columns used later:** `Domain`, `Location`, `Language`.
- **Output:** Each item is one target row.
- **Edge cases:** Missing columns will break expressions in the DataForSEO node (undefined values).

#### Node: **Loop Over Targets**
- **Type / role:** Split In Batches — iterates over targets.
- **Config:** Default batch behavior (batch size not explicitly set; n8n defaults apply).
- **Connections / behavior:**
  - **Input:** Targets list.
  - **Output (main index 1)** goes to `Initialize "items" field` (this is the path used in the workflow).
  - After Slack message is sent, flow returns to this node to continue the loop.
- **Edge cases:**
  - If there are zero targets, loop will produce nothing (no DataForSEO calls).
  - Large target lists: consider rate limits and execution time.

**Sticky note coverage:**  
“Get targets … spreadsheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=0#gid=0).”

---

### Block C — Initialize & Fetch Ranked Keywords (Paged)
**Overview:** For each target, initializes an empty `items` array, requests ranked keywords from DataForSEO in pages of 1000, and merges pages until finished.

**Nodes involved:**
- Initialize "items" field
- Set "items" field
- Get ranked keywords
- Merge "items" with DFS response
- Has more pages?
- Merge "items" with last response

#### Node: **Initialize "items" field**
- **Type / role:** Set — creates an empty accumulator.
- **Config:** Sets `items = []`.
- **Input:** Current target item from Loop Over Targets.
- **Output:** Same item plus `items: []`.
- **Edge cases:** None; purely internal initialization.

#### Node: **Set "items" field**
- **Type / role:** Set — preserves/forwards the working accumulator field between iterations.
- **Config:** Sets `items = {{$json.items}}`.
- **Input:** From Initialize or from the loop-back branch after pagination.
- **Output:** Ensures `items` is present for the next merge step.
- **Edge cases:** If an upstream branch loses `items`, later merge expressions can fail.

#### Node: **Get ranked keywords**
- **Type / role:** DataForSEO Labs API — fetch ranked keywords for the current target.
- **Operation:** `get-ranked-keywords`
- **Config choices:**
  - `limit: 1000`
  - `offset: {{$runIndex * 1000}}` (pagination uses the execution run index)
  - `target_any: {{ $('Get targets').item.json.Domain }}`
  - `language_name: {{ $('Get targets').item.json.Language }}`
  - `location_name: {{ $('Get targets').item.json.Location }}`
- **Credentials:** DataForSEO account (API login/password via n8n credential).
- **Output shape used later:** expects `tasks[0].result[0].items` and `tasks[0].result[0].total_count`.
- **Edge cases / failures:**
  - Auth errors (wrong login/password).
  - API limits, quota exceeded, timeouts.
  - If DataForSEO response structure changes or task returns errors, the expressions referencing `tasks[0]...` can throw.

#### Node: **Merge "items" with DFS response**
- **Type / role:** Set — concatenates accumulator + current page items.
- **Config expression:**
  - `items = {{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`
- **Input:** DataForSEO response (`$json`) plus access to previously set `items`.
- **Output:** One item with updated `items` array.
- **Edge cases:** If DataForSEO returns no items or tasks missing, this expression may fail.

#### Node: **Has more pages?**
- **Type / role:** IF — decides whether to fetch another page.
- **Condition:** `{{$runIndex}} < {{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
- **Outputs:**
  - **True branch →** `Set "items" field` (continue paging)
  - **False branch →** `Merge "items" with last response` (finalize)
- **Edge cases:**
  - `total_count` missing/0 can create negative thresholds.
  - Non-integer division may cause boundary off-by-one; consider `Math.floor(total_count/1000)` logic if you see missing/extra page calls.

#### Node: **Merge "items" with last response**
- **Type / role:** Set — final merge ensuring `items` contains both accumulator and the last DataForSEO page.
- **Config expression:**
  - `items = {{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]}}`
- **Input:** Runs on IF false branch.
- **Output:** Final `items` array for this target.
- **Potential issue:** This can **double-add the last page** depending on how the true/false branch sequencing and `$runIndex` iterations behave. If you notice duplicate rows in Sheets, this node’s concatenation logic is the first place to check (you may only need the accumulator from the previous merge, not re-adding the current page again).

**Sticky note coverage:**  
“Get ranked keywords from DataForSEO … Create a DataForSEO connection and set up additional parameters if needed.”

---

### Block D — Persist Current Keywords to Sheets
**Overview:** Takes the final `items` array (ranked keyword objects) and appends each item into the Keywords sheet, rebuilding the “current snapshot” after the clear.

**Nodes involved:**
- Split out (items)
- Append keyword in sheet
- Aggregate1

#### Node: **Split out (items)**
- **Type / role:** Split Out — converts array field `items` into individual items.
- **Config:** `fieldToSplitOut: "items"`.
- **Input:** One item containing `items: [ ... ]`.
- **Output:** Many items; each output item is one element from the `items` array.
- **Edge cases:** If `items` is empty or missing, outputs nothing (downstream steps won’t run).

#### Node: **Append keyword in sheet**
- **Type / role:** Google Sheets — append rows into Keywords sheet.
- **Operation:** `append`
- **Mapping:** “Define below” schema with these columns:
  - Target = `{{ $('Get targets').item.json.Domain }}`
  - Keyword = `{{ $json.keyword_data.keyword }}`
  - Date = `{{ new Date().toLocaleDateString('en-CA') }}` (YYYY-MM-DD)
  - Rank = `{{ $json.ranked_serp_element.serp_item.rank_absolute }}`
  - Search Volume = `{{ $json.keyword_data.keyword_info.search_volume }}`
  - Url = `{{ $json.ranked_serp_element.serp_item.url }}`
  - SERP Item Types = `{{ $json.keyword_data.serp_info.serp_item_types.join(', ') }}`
- **Input:** Each split item from DataForSEO `items`.
- **Output:** Append response per row.
- **Edge cases / failures:**
  - If some nested fields are missing (e.g., no `ranked_serp_element`), expressions can fail.
  - `serp_item_types` may be absent or not an array; `.join()` would error.
  - Date uses server locale/timezone; may differ from your business timezone.

#### Node: **Aggregate1**
- **Type / role:** Aggregate — aggregates appended rows (or their source items) into a single item.
- **Config:** `aggregateAllItemData`.
- **Role in logic:** Provides a single “current keywords dataset” to compare against previous baseline in the next block.
- **Edge cases:** If no keywords were appended (no items), downstream diff may be null.

**Sticky note coverage:**  
“Save current ranked keywords to the spreadsheet and send new keywords to Slack …”

---

### Block E — Compute Diff (New Keywords Only) & Filter
**Overview:** Compares the newly fetched keywords for the current target against the previously stored keywords for that same target, producing a list of newly ranked keywords (`diff`). Continues only if diff is not empty.

**Nodes involved:**
- Find new keywords
- Filter (has new keywords)

#### Node: **Find new keywords**
- **Type / role:** Code — computes set difference.
- **Key logic:**
  - Builds `oldKeywords` from `Aggregate.first().json.data`
    - Filters rows where `Target == current Domain`
    - Maps to `Keyword`
  - Builds `newKeywords` from `Merge "items" with last response`.json.items mapped to `keyword_data.keyword`
  - `diff = newKeywords.filter(x => !oldKeywords.has(x))`
  - Returns `{ diff }`
- **Important dependency:** Uses data from:
  - `Aggregate` (previous baseline)
  - `Get targets` (current domain)
  - `Merge "items" with last response` (final DataForSEO items)
- **Output:** One item: `{ diff: string[] | null }`
- **Edge cases / pitfalls:**
  - If there is no baseline (first run), `oldKeywords.size` is 0 so `diff` remains **null** (not an empty array). This is why the next Filter checks “notEmpty”; `null` will fail the filter and no Slack message will be sent on first run.
  - If `Merge "items" with last response` did not execute (pagination path issue), `items` may be undefined and diff becomes null/empty.

#### Node: **Filter (has new keywords)**
- **Type / role:** Filter — only proceed when `diff` contains at least one element.
- **Condition:** Array `notEmpty` on `{{$json.diff}}`.
- **True output:** To Slack message.
- **False output:** Ends for this target (no Slack).
- **Edge cases:** If `diff` is `null`, it is treated as empty and filtered out (intended in this design).

---

### Block F — Slack Alert & Loop Continuation
**Overview:** Sends a Slack message listing the newly ranked keywords for the current domain, then returns control to the loop to process the next target.

**Nodes involved:**
- Send a message

#### Node: **Send a message**
- **Type / role:** Slack — sends a message (OAuth2).
- **Text template:**
  - `New ranked keywords for target {{ $('Loop Over Targets').item.json.Domain }}: {{ $json.diff.join(', ') }}`
- **Inputs:** Filtered diff item.
- **Outputs:** Slack API response then loops back to “Loop Over Targets”.
- **Edge cases / failures:**
  - Slack OAuth scope/channel permissions.
  - Message length limits if diff is very large; consider truncation or sending as attachment/block kit.
  - `diff.join(', ')` will throw if `diff` is not an array (filter should prevent this, but be cautious if node behavior changes).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every Monday | Schedule Trigger | Weekly entry point | — | Get previous keywords | This workflow automatically fetches new keywords your domains started ranking for on Google, saves them into Google Sheets, and sends you a Slack summary so you can quickly see what’s changed.<br>## How it works<br>1. Triggers on your chosen schedule (default: once a week).<br>2. Reads your keywords and target domains from Google Sheets.<br>3. Extracts fresh ranking data from Google via DataForSEO Labs API.<br>4. Compares the results with the previous run.<br>5. Adds newly ranked keywords into a dedicated Google Sheet.<br>6. Sends a weekly summary message to Slack.<br>## Setup steps<br>1. Indicate your spreadsheets with keywords and target domains.<br>2. Create or select your DataForSEO connection (use your API login and password – see here).<br>3. Create a Slack connection and choose who gets the message with the new ranked keywords. |
| Get previous keywords | Google Sheets | Read baseline keywords (previous run) | Run every Monday | Aggregate | ## Get previous keywords and clear the sheet<br>Create a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=1681593026#gid=1681593026). |
| Aggregate | Aggregate | Aggregate baseline rows into one item | Get previous keywords | Clear sheet (Keywords) | ## Get previous keywords and clear the sheet<br>Create a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=1681593026#gid=1681593026). |
| Clear sheet (Keywords) | Google Sheets | Clear current snapshot sheet (keep headers) | Aggregate | Get targets | ## Get previous keywords and clear the sheet<br>Create a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=1681593026#gid=1681593026). |
| Get targets | Google Sheets | Load target domains (Domain/Location/Language) | Clear sheet (Keywords) | Loop Over Targets | ## Get targets<br>Select a spreadsheet with your target domains. The spreadsheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=0#gid=0). |
| Loop Over Targets | Split In Batches | Iterate over each target | Get targets; Send a message | Initialize "items" field | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Initialize "items" field | Set | Initialize accumulator array `items=[]` | Loop Over Targets | Set "items" field | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Set "items" field | Set | Forward current accumulator `items` | Initialize "items" field; Has more pages? (true) | Get ranked keywords | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Get ranked keywords | DataForSEO Labs API | Fetch ranked keywords page (limit/offset) | Set "items" field | Merge "items" with DFS response | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with DFS response | Set | Append current page items into accumulator | Get ranked keywords | Has more pages? | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Has more pages? | IF | Pagination decision | Merge "items" with DFS response | Set "items" field (true); Merge "items" with last response (false) | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with last response | Set | Finalize `items` before persistence | Has more pages? (false) | Split out (items) | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Split out (items) | Split Out | Expand items array into per-keyword items | Merge "items" with last response | Append keyword in sheet | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Append keyword in sheet | Google Sheets | Store current ranked keywords rows | Split out (items) | Aggregate1 | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Aggregate1 | Aggregate | Collect current run items for diff calculation | Append keyword in sheet | Find new keywords | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Find new keywords | Code | Compute `diff` vs baseline for current target | Aggregate1 | Filter (has new keywords) | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Filter (has new keywords) | Filter | Only continue when diff not empty | Find new keywords | Send a message | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Send a message | Slack | Post Slack summary for the target | Filter (has new keywords) | Loop Over Targets | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Sticky Note2 | Sticky Note | Documentation / instructions | — | — | ## Get targets<br>Select a spreadsheet with your target domains. The spreadsheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=0#gid=0). |
| Sticky Note3 | Sticky Note | Documentation / instructions | — | — | ## Get ranked keywords from DataForSEO<br>Create a DataForSEO connection and set up additional parameters if needed. |
| Sticky Note5 | Sticky Note | Documentation / instructions | — | — | ## Save current ranked keywords to the spreadsheet and send new keywords to Slack<br>Select the same spreadsheet with your keywords as in the first Google Sheets node.<br>Add additional information to the Slack message if needed. |
| Sticky Note6 | Sticky Note | Workflow description / setup | — | — | This workflow automatically fetches new keywords your domains started ranking for on Google, saves them into Google Sheets, and sends you a Slack summary so you can quickly see what’s changed.<br>## How it works<br>1. Triggers on your chosen schedule (default: once a week).<br>2. Reads your keywords and target domains from Google Sheets.<br>3. Extracts fresh ranking data from Google via DataForSEO Labs API.<br>4. Compares the results with the previous run.<br>5. Adds newly ranked keywords into a dedicated Google Sheet.<br>6. Sends a weekly summary message to Slack.<br>## Setup steps<br>1. Indicate your spreadsheets with keywords and target domains.<br>2. Create or select your DataForSEO connection (use your API login and password – see here).<br>3. Create a Slack connection and choose who gets the message with the new ranked keywords. |
| Sticky Note7 | Sticky Note | Documentation / instructions | — | — | ## Get previous keywords and clear the sheet<br>Create a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=1681593026#gid=1681593026). |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Google Sheets file**
   1. Create a spreadsheet (or reuse one).
   2. Add sheet **Targets** with columns at least: `Domain`, `Location`, `Language`.
   3. Add sheet **Keywords** with columns matching the append mapping: `Target`, `Keyword`, `Date`, `Rank`, `Search Volume`, `Url`, `SERP Item Types`.
   4. Ensure headers are in row 1.

2) **Create credentials**
   1. **Google Sheets OAuth2** credential in n8n with access to the spreadsheet.
   2. **DataForSEO API** credential (login + password).
   3. **Slack OAuth2** credential with permission to post messages to the chosen channel/user.

3) **Add nodes and configure in this order**
   1. **Schedule Trigger** node named *Run every Monday*  
      - Set interval: weekly, Monday, 09:00 (adjust timezone as needed).
   2. **Google Sheets** node named *Get previous keywords*  
      - Resource: Spreadsheet  
      - Operation: Read/Get Many (default “read rows”)  
      - Select Document: your spreadsheet  
      - Sheet: `Keywords`  
      - Enable “Always Output Data”.
   3. **Aggregate** node named *Aggregate*  
      - Mode: “Aggregate All Item Data”.
   4. **Google Sheets** node named *Clear sheet (Keywords)*  
      - Operation: Clear  
      - Sheet: `Keywords`  
      - Enable “Keep First Row”.
   5. **Google Sheets** node named *Get targets*  
      - Operation: Read/Get Many  
      - Sheet: `Targets`.
   6. **Split In Batches** node named *Loop Over Targets*  
      - Default options are sufficient (or set batch size = 1 explicitly).
      - Use the **second output** (index 1) for the loop body, matching the provided workflow logic.
   7. **Set** node named *Initialize "items" field*  
      - Add field `items` (type Array) = `[]`.
   8. **Set** node named *Set "items" field*  
      - Add field `items` (Array) = expression `{{$json.items}}`.
   9. **DataForSEO** node named *Get ranked keywords*  
      - Operation: `get-ranked-keywords`  
      - Limit: `1000`  
      - Offset: `{{$runIndex * 1000}}`  
      - Target: `{{$('Get targets').item.json.Domain}}`  
      - Language name: `{{$('Get targets').item.json.Language}}`  
      - Location name: `{{$('Get targets').item.json.Location}}`.
   10. **Set** node named *Merge "items" with DFS response*  
       - Field `items` (Array) = `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`.
   11. **IF** node named *Has more pages?*  
       - Condition (Number): `{{$runIndex}}` **is less than** `{{$('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1}}`.
       - True → loop back to *Set "items" field*  
       - False → continue to next step.
   12. **Set** node named *Merge "items" with last response*  
       - Field `items` (Array) = `{{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]}}`.
   13. **Split Out** node named *Split out (items)*  
       - Field to split out: `items`.
   14. **Google Sheets** node named *Append keyword in sheet*  
       - Operation: Append  
       - Sheet: `Keywords`  
       - Map columns (expressions):
         - Target: `{{$('Get targets').item.json.Domain}}`
         - Keyword: `{{$json.keyword_data.keyword}}`
         - Date: `{{new Date().toLocaleDateString('en-CA')}}`
         - Rank: `{{$json.ranked_serp_element.serp_item.rank_absolute}}`
         - Search Volume: `{{$json.keyword_data.keyword_info.search_volume}}`
         - Url: `{{$json.ranked_serp_element.serp_item.url}}`
         - SERP Item Types: `{{$json.keyword_data.serp_info.serp_item_types.join(', ')}}`
   15. **Aggregate** node named *Aggregate1*  
       - Mode: “Aggregate All Item Data”.
   16. **Code** node named *Find new keywords*  
       - Paste the same JS logic to compute `diff` based on:
         - `Aggregate` baseline (`Aggregate.first().json.data`)
         - Current target (`Get targets`)
         - Current items (`Merge "items" with last response`)
   17. **Filter** node named *Filter (has new keywords)*  
       - Condition: Array `notEmpty` on `{{$json.diff}}`.
   18. **Slack** node named *Send a message*  
       - Operation: Post message  
       - Text: `New ranked keywords for target {{ $('Loop Over Targets').item.json.Domain }}: {{ $json.diff.join(', ') }}`

4) **Wire the connections**
   - Run every Monday → Get previous keywords → Aggregate → Clear sheet (Keywords) → Get targets → Loop Over Targets
   - Loop Over Targets (output 1) → Initialize "items" field → Set "items" field → Get ranked keywords → Merge "items" with DFS response → Has more pages?
   - Has more pages? (true) → Set "items" field (continues paging)
   - Has more pages? (false) → Merge "items" with last response → Split out (items) → Append keyword in sheet → Aggregate1 → Find new keywords → Filter (has new keywords) → Send a message → Loop Over Targets

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Example spreadsheet structure for Targets | https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=0#gid=0 |
| Example spreadsheet structure for Keywords | https://docs.google.com/spreadsheets/d/1FO9Btg5y5TmE56La4O-QzJbEjGAZLe3zG0phB7eXnqs/edit?pli=1&gid=1681593026#gid=1681593026 |
| Operational note: workflow clears the Keywords sheet each run (keeps headers) and rebuilds it from DataForSEO results | Applies to the overall data model (“current snapshot” approach) |
| First run behavior: baseline is empty, so `diff` becomes `null` and Slack won’t send any message | Comes from Code node logic (`if (oldKeywords.size) ...`) |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.