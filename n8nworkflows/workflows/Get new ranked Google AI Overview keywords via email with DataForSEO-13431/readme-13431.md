Get new ranked Google AI Overview keywords via email with DataForSEO

https://n8nworkflows.xyz/workflows/get-new-ranked-google-ai-overview-keywords-via-email-with-dataforseo-13431


# Get new ranked Google AI Overview keywords via email with DataForSEO

## 1. Workflow Overview

**Purpose:** This workflow runs on a weekly schedule (Monday 09:00) to detect **new keywords** that a set of target domains started ranking for in Google’s **AI Overview** (via DataForSEO Labs API). It **stores the latest snapshot** of ranked AIO keywords in Google Sheets and **emails a short summary** listing only the *newly discovered* keywords per target.

**Primary use cases**
- SEO monitoring for Google AI Overview visibility per domain
- Weekly change detection (“newly ranked AIO keywords”) with a persistent spreadsheet log
- Automated reporting via Gmail

### 1.1 Scheduled start + previous snapshot load
Loads the previously stored keyword snapshot from Google Sheets (Keywords tab) and prepares it for comparison.

### 1.2 Snapshot reset + targets load
Clears the Keywords sheet (keeping header row) and loads the list of target domains (Targets tab) to iterate through them.

### 1.3 Per-target DataForSEO pagination + consolidation
For each target, paginates through DataForSEO “ranked keywords” results (1000 per page), accumulating all returned items into a single array.

### 1.4 Persist snapshot + compute diff + email
Writes the current full snapshot back into the Keywords sheet, computes “new keywords vs previous snapshot” for that target, and emails a summary if there are any new keywords.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled start + previous snapshot load

**Overview:** Triggers weekly, reads the prior keyword snapshot from Google Sheets, and aggregates it into a single in-memory array for downstream comparison.

**Nodes involved:**
- Run every Monday
- Get previous keywords
- Aggregate

#### Node: **Run every Monday**
- **Type / role:** Schedule Trigger — workflow entry point
- **Config:** Weekly schedule: **Monday** at **09:00**
- **Outputs to:** Get previous keywords
- **Edge cases / failures:** n8n timezone settings can shift the effective trigger time; ensure instance timezone is correct.

#### Node: **Get previous keywords**
- **Type / role:** Google Sheets — read existing snapshot
- **Config choices:**
  - Reads from Spreadsheet **“New Ranked Keywords with ai_overview from Google with DataForSEO”**
  - Sheet: **Keywords** (gid 1681593026)
  - `alwaysOutputData: true` so the workflow continues even if the sheet is empty
- **Outputs to:** Aggregate
- **Edge cases / failures:**
  - OAuth token expired / permission denied
  - Sheet renamed or columns changed
  - Empty sheet: output may be empty; handled by later code (but see diff logic in Block 4)

#### Node: **Aggregate**
- **Type / role:** Aggregate — combine all rows into one item
- **Config:** `aggregateAllItemData` (collect all incoming items into one structure)
- **Outputs to:** Clear sheet
- **Key data:** Later referenced as `$('Aggregate').first().json.data` in the Code node.
- **Edge cases / failures:** If upstream returns no rows, aggregated `.json.data` may be missing/undefined (the code checks for it).

**Sticky note covering this block**
- **“Get previous keywords and clear the sheet”**  
  “Create a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=1681593026#gid=1681593026).”

---

### Block 2 — Snapshot reset + targets load

**Overview:** Clears the Keywords sheet (keeping headers), then loads the targets list from a separate sheet and starts looping through targets.

**Nodes involved:**
- Clear sheet
- Get targets
- Loop over targets

#### Node: **Clear sheet**
- **Type / role:** Google Sheets — clear existing snapshot
- **Config choices:**
  - Operation: **Clear**
  - `keepFirstRow: true` (preserves header row)
  - Spreadsheet and sheet match the “previous keywords” sheet (Keywords tab)
- **Inputs from:** Aggregate
- **Outputs to:** Get targets
- **Edge cases / failures:**
  - If headers aren’t in the first row, you may lose schema alignment.
  - Permission issues may prevent clearing.

#### Node: **Get targets**
- **Type / role:** Google Sheets — read target domains
- **Config choices:**
  - Spreadsheet: same document as above
  - Sheet: **Targets** (gid=0)
  - Expected columns (implied by expressions used later): **Domain**, **Language**, **Location**
- **Inputs from:** Clear sheet
- **Outputs to:** Loop over targets
- **Edge cases / failures:**
  - Missing columns: expressions like `$('Get targets').item.json.Domain` will resolve to `undefined` and break API filtering.
  - Empty target list => loop does nothing, no snapshot written, no emails sent.

#### Node: **Loop over targets**
- **Type / role:** SplitInBatches — iterate through targets (acts as the per-target loop controller)
- **Config:** Uses default batch behavior (batch size not explicitly set in JSON; depends on node defaults)
- **Inputs from:** Get targets; also receives a “continue loop” signal later from Send a message
- **Outputs:**
  - Output **index 1** goes to Initialize "items" field (this is the “current batch item(s)” path in this workflow wiring)
- **Edge cases / failures:**
  - If batch size > 1, `$('Get targets').item` semantics can be confusing; the workflow appears designed for per-target iteration.
  - Loop continuation relies on the connection from **Send a message** back into the loop.

**Sticky note covering this block**
- **“Get targets”**  
  “Select a spreadsheet with your target domains. The spreadsheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=0#gid=0).”

---

### Block 3 — Per-target DataForSEO pagination + consolidation

**Overview:** Initializes an accumulator array (`items`), then repeatedly calls DataForSEO Labs “ranked keywords” with a 1000-item page size, appending each page to `items` until all pages are fetched.

**Nodes involved:**
- Initialize "items" field
- Set "items" field
- Get ranked keywords
- Merge "items" with DFS response
- Has more pages?
- Merge "items" with last response

#### Node: **Initialize "items" field**
- **Type / role:** Set — initialize accumulator
- **Config:** Sets `items` to an empty array: `items = []`
- **Inputs from:** Loop over targets
- **Outputs to:** Set "items" field
- **Edge cases / failures:** None significant.

#### Node: **Set "items" field**
- **Type / role:** Set — pass/retain accumulator
- **Config:** Sets `items` to the current JSON’s `items` (effectively “carry forward”): `items = $json.items`
- **Inputs from:** Initialize "items" field; also from Has more pages? (looping)
- **Outputs to:** Get ranked keywords
- **Edge cases / failures:** If `items` missing, becomes `undefined` and later merges may fail; initialization should prevent this.

#### Node: **Get ranked keywords**
- **Type / role:** DataForSEO Labs API — fetch ranked keywords for AIO
- **Config choices (interpreted):**
  - Operation: **get-ranked-keywords**
  - Filters:
    - `item_types = ["ai_overview_reference"]` (only AIO-relevant results)
    - `target_any = Domain` from current target row
    - `language_name = Language` from current target row
    - `location_name = Location` from current target row
  - Pagination:
    - `limit = 1000`
    - `offset = {{$runIndex * 1000}}`
      - `$runIndex` increases each loop cycle through **Has more pages? → Set "items" field → Get ranked keywords**
- **Credentials:** DataForSEO account (API login/password)
- **Inputs from:** Set "items" field
- **Outputs to:** Merge "items" with DFS response
- **Edge cases / failures:**
  - Auth failures (wrong DataForSEO credentials)
  - API rate limits / quota exhaustion
  - If DataForSEO returns no `tasks[0].result[0].items`, downstream merge expressions can throw.

#### Node: **Merge "items" with DFS response**
- **Type / role:** Set — append current page items to accumulator
- **Config / expression:**
  - `items = [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items ]`
- **Inputs from:** Get ranked keywords
- **Outputs to:** Has more pages?
- **Edge cases / failures:**
  - If `tasks[0].result[0].items` is undefined/null, spread operator will throw.
  - If the workflow runs with multiple items per target, `.item` resolution may not match expected target row.

#### Node: **Has more pages?**
- **Type / role:** IF — pagination decision
- **Condition (interpreted):**
  - Continue paging while:  
    `$runIndex < ( total_count / 1000 - 1 )`  
    where `total_count` is taken from `$('Get ranked keywords').item.json.tasks[0].result[0].total_count`
- **Outputs:**
  - **True path (index 0):** Set "items" field (fetch next page)
  - **False path (index 1):** Merge "items" with last response (finalize)
- **Edge cases / failures:**
  - `total_count` missing => expression failure
  - Non-integer page calculation: if `total_count` isn’t a multiple of 1000, the formula can be off by one depending on rounding; consider using `Math.ceil(total_count/1000)` logic to be safer.

#### Node: **Merge "items" with last response**
- **Type / role:** Set — finalize accumulator on last page
- **Config / expression:**
  - `items = [ ...$('Set "items" field').item.json.items, ...$('Get ranked keywords').item.json.tasks[0].result[0].items ]`
- **Inputs from:** Has more pages? (false branch)
- **Outputs to:** Split out (items)
- **Edge cases / failures:** Same as merge above if last response has missing `items`.

**Sticky note covering this block**
- **“Get ranked keywords from Google AIO with DataForSEO”**  
  “Create a DataForSEO connection and set up additional parameters if needed.”

---

### Block 4 — Persist snapshot + compute diff + email

**Overview:** Writes each returned keyword item into the Keywords sheet, aggregates the newly written rows, compares keyword lists against the previous snapshot for the same domain, and emails a summary when new keywords exist.

**Nodes involved:**
- Split out (items)
- Append keyword in sheet
- Aggregate1
- Find new AIO keywords
- Filter (has new AIO keywords)
- Send a message

#### Node: **Split out (items)**
- **Type / role:** SplitOut — converts the `items` array into individual items
- **Config:** `fieldToSplitOut = "items"`
- **Inputs from:** Merge "items" with last response
- **Outputs to:** Append keyword in sheet
- **Edge cases / failures:** If `items` is empty or missing, no output rows are produced (and nothing is appended).

#### Node: **Append keyword in sheet**
- **Type / role:** Google Sheets — append current snapshot rows
- **Config choices:**
  - Operation: **Append**
  - Writes columns:
    - **Url** = `ranked_serp_element.serp_item.url`
    - **Date** = `new Date().toLocaleDateString('en-CA')` (YYYY-MM-DD in many locales)
    - **Rank** = `ranked_serp_element.serp_item.rank_absolute`
    - **Target** = current target Domain
    - **Keyword** = `keyword_data.keyword`
    - **Search Volume** = `keyword_data.keyword_info.search_volume`
    - **SERP Item Types** = `keyword_data.serp_info.serp_item_types.join(', ')`
  - Mapping mode explicitly defines schema fields in-node.
- **Inputs from:** Split out (items)
- **Outputs to:** Aggregate1
- **Edge cases / failures:**
  - If any nested fields are missing (e.g., `ranked_serp_element.serp_item.url`), expressions may evaluate to `undefined` and append blank cells.
  - `serp_item_types` might be missing/not an array; `.join()` would throw.
  - Spreadsheet ID mismatch risk: the node’s `sheetName` cached URL differs (shows `1uCvVHK...` in cachedResultUrl but `documentId` is `1w8bTZ...`). In n8n, the actual `documentId.value` is what matters—verify it points to the correct spreadsheet.

#### Node: **Aggregate1**
- **Type / role:** Aggregate — combine appended rows for downstream processing
- **Config:** `aggregateAllItemData`
- **Inputs from:** Append keyword in sheet
- **Outputs to:** Find new AIO keywords
- **Edge cases / failures:** If nothing was appended (no items), aggregation may be empty; downstream code should handle but may lead to empty comparisons.

#### Node: **Find new AIO keywords**
- **Type / role:** Code — computes “diff” between previous and current keywords for the target
- **Logic (interpreted):**
  1. Build `oldKeywords` set from `Aggregate` (previous snapshot), filtered by `Target == current Domain`, collecting `Keyword`
  2. Build `newKeywords` array from consolidated DataForSEO `items`, collecting `keyword_data.keyword`
  3. `diff = newKeywords not in oldKeywords` (only if `oldKeywords.size > 0`, else `diff = []`)
  4. Return one item: `{ diff: [...] }`
- **Important expressions:**
  - Previous snapshot source: `$('Aggregate').first().json.data`
  - Current target domain: `$('Get targets').first().json.Domain` (note: uses `.first()`, not `.item`)
  - Current fetched items: `$('Merge "items" with last response').first().json.items`
- **Inputs from:** Aggregate1
- **Outputs to:** Filter (has new AIO keywords)
- **Edge cases / failures / correctness notes:**
  - **If there were no previous keywords for a domain**, the code sets `diff = []` (so it will not email anything). If you want first-run discovery emails, change this behavior to `diff = newKeywords`.
  - Uses `$('Get targets').first()` instead of the current loop item; with multiple targets, `.first()` can incorrectly pin to the first target domain. Prefer `$('Get targets').item` (or pass Domain forward explicitly) to ensure per-target correctness.
  - If DataForSEO returns duplicate keywords, `diff` may contain duplicates unless deduplicated.

#### Node: **Filter (has new AIO keywords)**
- **Type / role:** Filter — only proceed when there is something to report
- **Condition:** `diff` is **not empty**
- **Inputs from:** Find new AIO keywords
- **Outputs to:** Send a message
- **Edge cases:** None major; if `diff` undefined, treated as empty by condition evaluation depending on node semantics.

#### Node: **Send a message**
- **Type / role:** Gmail — send HTML email summary
- **Config:**
  - To: `user@example.com` (replace)
  - Subject: `New Ranked Keywords for AIO`
  - HTML message includes:
    - target domain: `{{ $('Get targets').item.json.Domain }}`
    - diff list: `{{ $json.diff.join(', ') }}`
- **Inputs from:** Filter (has new AIO keywords)
- **Outputs to:** Loop over targets (to continue processing remaining targets)
- **Edge cases / failures:**
  - Gmail OAuth issues / insufficient scopes
  - Email too long if diff list is huge; consider truncation and/or attaching a file
  - `$('Get targets').item.json.Domain` depends on correct loop item context; ensure it matches the current target being processed.

**Sticky note covering this block**
- **“Save current ranked AIO keywords to the spreadsheet and send new keywords via Email”**  
  “Select the same spreadsheet with your keywords as in the first Google Sheets node.  
  Create a Gmail connection and set a receiver. Add additional information to the message if needed.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every Monday | Schedule Trigger | Starts workflow weekly | — | Get previous keywords | This workflow uses DataForSEO Labs API to automatically find new keywords your domains started ranking for in Google’s AI Overview feature, then saves those keywords into Google Sheets, and sends you a short email summary. |
| Get previous keywords | Google Sheets | Read previous keyword snapshot | Run every Monday | Aggregate | ## Get previous keywords and clear the sheet \nCreate a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=1681593026#gid=1681593026). |
| Aggregate | Aggregate | Consolidate previous snapshot rows | Get previous keywords | Clear sheet | ## Get previous keywords and clear the sheet \nCreate a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=1681593026#gid=1681593026). |
| Clear sheet | Google Sheets | Clear Keywords sheet (keep header) | Aggregate | Get targets | ## Get previous keywords and clear the sheet \nCreate a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=1681593026#gid=1681593026). |
| Get targets | Google Sheets | Load list of domains/language/location | Clear sheet | Loop over targets | ## Get targets \nSelect a spreadsheet with your target domains. The spreadsheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=0#gid=0). |
| Loop over targets | SplitInBatches | Iterate per target | Get targets; Send a message | Initialize "items" field | This workflow uses DataForSEO Labs API to automatically find new keywords your domains started ranking for in Google’s AI Overview feature, then saves those keywords into Google Sheets, and sends you a short email summary. |
| Initialize "items" field | Set | Initialize accumulator array | Loop over targets | Set "items" field | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Set "items" field | Set | Carry accumulator into next step | Initialize "items" field; Has more pages? | Get ranked keywords | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Get ranked keywords | DataForSEO Labs API | Fetch ranked AIO keywords (paged) | Set "items" field | Merge "items" with DFS response | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with DFS response | Set | Append current page to accumulator | Get ranked keywords | Has more pages? | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Has more pages? | IF | Continue pagination or finalize | Merge "items" with DFS response | Set "items" field; Merge "items" with last response | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with last response | Set | Finalize full items list | Has more pages? (false) | Split out (items) | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Split out (items) | SplitOut | Convert items array into rows | Merge "items" with last response | Append keyword in sheet | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Append keyword in sheet | Google Sheets | Append snapshot rows into Keywords sheet | Split out (items) | Aggregate1 | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Aggregate1 | Aggregate | Consolidate appended rows | Append keyword in sheet | Find new AIO keywords | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Find new AIO keywords | Code | Diff current vs previous keywords | Aggregate1 | Filter (has new AIO keywords) | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Filter (has new AIO keywords) | Filter | Only proceed if diff not empty | Find new AIO keywords | Send a message | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Send a message | Gmail | Email diff summary | Filter (has new AIO keywords) | Loop over targets | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Sticky Note | Sticky Note | Comment / instructions | — | — | ## Get previous keywords and clear the sheet \nCreate a Google Sheets connection and select the spreadsheet where keywords should be saved. The sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=1681593026#gid=1681593026). |
| Sticky Note2 | Sticky Note | Comment / instructions | — | — | ## Get targets \nSelect a spreadsheet with your target domains. The spreadsheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=0#gid=0). |
| Sticky Note3 | Sticky Note | Comment / instructions | — | — | ## Get ranked keywords from Google AIO with DataForSEO \nCreate a DataForSEO connection and set up additional parameters if needed. |
| Sticky Note5 | Sticky Note | Comment / instructions | — | — | ## Save current ranked AIO keywords to the spreadsheet and send new keywords via Email \nSelect the same spreadsheet with your keywords as in the first Google Sheets node.\nCreate a Gmail connection and set a receiver. Add additional information to the message if needed. |
| Sticky Note6 | Sticky Note | Comment / overview | — | — | This workflow uses DataForSEO Labs API to automatically find new keywords your domains started ranking for in Google’s AI Overview feature, then saves those keywords into Google Sheets, and sends you a short email summary. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   **“Get new ranked keywords from Google AIO via email with DataForSEO”**

2. **Add Trigger**
   - Add node: **Schedule Trigger**
   - Configure: weekly on **Monday**, at **09:00**
   - Node name: **Run every Monday**

3. **Read previous keyword snapshot**
   - Add node: **Google Sheets**
   - Node name: **Get previous keywords**
   - Credentials: **Google Sheets OAuth2**
   - Select Spreadsheet: your file (with at least `Keywords` and `Targets` sheets)
   - Sheet: **Keywords**
   - Turn on **Always Output Data** (so workflow continues when empty)
   - Connect: **Run every Monday → Get previous keywords**

4. **Aggregate previous snapshot**
   - Add node: **Aggregate**
   - Node name: **Aggregate**
   - Operation/mode: **Aggregate All Item Data**
   - Connect: **Get previous keywords → Aggregate**

5. **Clear Keywords sheet (keep header row)**
   - Add node: **Google Sheets**
   - Node name: **Clear sheet**
   - Operation: **Clear**
   - Sheet: **Keywords**
   - Enable: **Keep First Row**
   - Connect: **Aggregate → Clear sheet**

6. **Load targets**
   - Add node: **Google Sheets**
   - Node name: **Get targets**
   - Operation: read rows from sheet **Targets**
   - Ensure Targets sheet columns exist: **Domain**, **Language**, **Location**
   - Connect: **Clear sheet → Get targets**

7. **Loop through targets**
   - Add node: **Split In Batches**
   - Node name: **Loop over targets**
   - Keep defaults (or set batch size to 1 for clarity)
   - Connect: **Get targets → Loop over targets**

8. **Initialize accumulator**
   - Add node: **Set**
   - Node name: **Initialize "items" field**
   - Add field:
     - `items` (type: Array) = `[]`
   - Connect: **Loop over targets → Initialize "items" field** (use the output that corresponds to iterating items)

9. **Carry accumulator forward**
   - Add node: **Set**
   - Node name: **Set "items" field**
   - Add field:
     - `items` (type: Array) = `{{$json.items}}`
   - Connect: **Initialize "items" field → Set "items" field**

10. **Call DataForSEO Labs: ranked keywords**
    - Add node: **DataForSEO Labs API** (community/partner node as in your instance)
    - Node name: **Get ranked keywords**
    - Credentials: **DataForSEO API** (login + password)
    - Operation: **get-ranked-keywords**
    - Parameters:
      - `limit`: 1000
      - `offset`: `{{$runIndex * 1000}}`
      - `item_types`: `["ai_overview_reference"]`
      - `target_any`: `{{$('Get targets').item.json.Domain}}`
      - `language_name`: `{{$('Get targets').item.json.Language}}`
      - `location_name`: `{{$('Get targets').item.json.Location}}`
    - Connect: **Set "items" field → Get ranked keywords**

11. **Merge accumulator + current page**
    - Add node: **Set**
    - Node name: **Merge "items" with DFS response**
    - Set field:
      - `items` = `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`
    - Connect: **Get ranked keywords → Merge "items" with DFS response**

12. **Add pagination decision**
    - Add node: **IF**
    - Node name: **Has more pages?**
    - Condition (Number):
      - Left: `{{$runIndex}}`
      - Operation: **less than**
      - Right: `{{$('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1}}`
    - Connect: **Merge "items" with DFS response → Has more pages?**
    - Connect **True** output → **Set "items" field** (to fetch next page)

13. **Finalize items on last page**
    - Add node: **Set**
    - Node name: **Merge "items" with last response**
    - Set field:
      - `items` = `{{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]}}`
    - Connect: **Has more pages? (False) → Merge "items" with last response**

14. **Split items into rows**
    - Add node: **Split Out**
    - Node name: **Split out (items)**
    - Field to split out: `items`
    - Connect: **Merge "items" with last response → Split out (items)**

15. **Append snapshot rows to Keywords sheet**
    - Add node: **Google Sheets**
    - Node name: **Append keyword in sheet**
    - Operation: **Append**
    - Sheet: **Keywords**
    - Map columns (Define Below) with these expressions:
      - Target: `{{$('Get targets').item.json.Domain}}`
      - Keyword: `{{$json.keyword_data.keyword}}`
      - Date: `{{ new Date().toLocaleDateString('en-CA') }}`
      - Rank: `{{$json.ranked_serp_element.serp_item.rank_absolute}}`
      - Search Volume: `{{$json.keyword_data.keyword_info.search_volume}}`
      - Url: `{{$json.ranked_serp_element.serp_item.url}}`
      - SERP Item Types: `{{$json.keyword_data.serp_info.serp_item_types.join(', ')}}`
    - Connect: **Split out (items) → Append keyword in sheet**

16. **Aggregate appended rows**
    - Add node: **Aggregate**
    - Node name: **Aggregate1**
    - Mode: **Aggregate All Item Data**
    - Connect: **Append keyword in sheet → Aggregate1**

17. **Compute “new keywords” diff**
    - Add node: **Code**
    - Node name: **Find new AIO keywords**
    - Paste logic (adapt if you fix target scoping; see note below):
      - Builds `oldKeywords` from **Aggregate**
      - Builds `newKeywords` from **Merge "items" with last response**
      - Outputs `{ diff: [...] }`
    - Connect: **Aggregate1 → Find new AIO keywords**
    - (Recommended adjustment for correctness with multiple targets: avoid `first()` and carry `Domain` forward as data, then compare using that value.)

18. **Filter when diff not empty**
    - Add node: **Filter**
    - Node name: **Filter (has new AIO keywords)**
    - Condition: `diff` **is not empty**
    - Connect: **Find new AIO keywords → Filter (has new AIO keywords)**

19. **Send email**
    - Add node: **Gmail**
    - Node name: **Send a message**
    - Credentials: **Gmail OAuth2**
    - To: your recipient
    - Subject: `New Ranked Keywords for AIO`
    - Message (HTML), e.g.:
      - `New ranked keywords in ai_overview_reference for target {{ $('Get targets').item.json.Domain }}: {{ $json.diff.join(', ') }}`
    - Connect: **Filter (has new AIO keywords) → Send a message**

20. **Continue the target loop**
    - Connect: **Send a message → Loop over targets**  
      (This is the loop continuation path in this workflow design. If you also want to continue when no email is sent, you’d need an additional connection from the “no diff” branch back to the loop.)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow uses DataForSEO Labs API to automatically find new keywords your domains started ranking for in Google’s AI Overview feature, then saves those keywords into Google Sheets, and sends you a short email summary.” | Workflow overview sticky note (Sticky Note6) |
| Example Keywords sheet structure | https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=1681593026#gid=1681593026 |
| Example Targets sheet structure | https://docs.google.com/spreadsheets/d/1w8bTZ0hfQ0A0e-GZ3s5oDR8-lWqpepU-FJN_Uv4fWwE/edit?gid=0#gid=0 |

**Disclaimer (provided by you):** Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.