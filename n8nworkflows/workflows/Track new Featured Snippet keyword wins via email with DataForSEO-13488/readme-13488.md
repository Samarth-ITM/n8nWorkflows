Track new Featured Snippet keyword wins via email with DataForSEO

https://n8nworkflows.xyz/workflows/track-new-featured-snippet-keyword-wins-via-email-with-dataforseo-13488


# Track new Featured Snippet keyword wins via email with DataForSEO

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Track new Featured Snippet keyword wins via email with DataForSEO  
**Internal workflow name:** Get new Featured Snippet keyword wins for your domain via email with DataForSEO

**Purpose:**  
On a weekly schedule, the workflow pulls the latest **Featured Snippet** ranked keywords for one or multiple target domains from **DataForSEO Labs**, compares them to the keywords previously stored in **Google Sheets**, writes the refreshed keyword dataset back to the sheet, and emails a summary listing only the **newly detected** Featured Snippet keywords per target.

**Typical use cases:**
- SEO monitoring: detect newly earned Featured Snippet visibility for your domain(s)
- Weekly reporting to SEO/marketing teams
- Maintaining a historical log (via Google Sheets) of featured snippet wins

### Logical blocks
**1.1 Scheduled start + load previous snapshot (Keywords sheet)**  
Runs weekly, reads the stored keyword snapshot and aggregates it into a usable array.

**1.2 Reset output sheet + load targets (Targets sheet)**  
Clears the Keywords sheet (keeps header row), then loads target rows (domain, language, location).

**1.3 Per-target loop: initialize pagination accumulator**  
Iterates through each target domain, initializes an `items` array to accumulate paginated DataForSEO results.

**1.4 DataForSEO Labs pagination + item merging**  
Calls DataForSEO “get-ranked-keywords” for Featured Snippet results, accumulating items across pages (1000/page).

**1.5 Persist refreshed keyword snapshot (Keywords sheet)**  
Splits accumulated items into rows and appends them to Google Sheets (rebuilding the sheet content after clearing).

**1.6 Compare old vs new + email new wins**  
Computes keyword diff (new items not in old snapshot for the same target). If non-empty, sends an email summary, then continues with the next target.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled start + load previous snapshot (Keywords sheet)
**Overview:**  
Triggers weekly. Loads the previous snapshot of keywords from Google Sheets and aggregates them into one array for later comparison.

**Nodes involved:**  
- Run every Monday  
- Get previous keywords  
- Aggregate

#### Node: Run every Monday
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — workflow entry point.
- **Configuration:** Weekly schedule; triggers **Mondays at 09:00** (instance timezone).
- **Outputs:** To **Get previous keywords**.
- **Edge cases / failures:**
  - Timezone differences between instance and user expectation.
  - Workflow is **inactive** in provided JSON (`active: false`), so it won’t run until activated.

#### Node: Get previous keywords
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads existing keyword snapshot.
- **Configuration choices:**
  - Document: “New Ranked Keywords for featured_snippet from Google with DataForSEO”
  - Sheet: “Keywords”
  - Operation not explicitly set in JSON; by default this node is used as a **read/get many rows** step.
  - `alwaysOutputData: true` ensures downstream nodes run even if the sheet is empty.
- **Outputs:** To **Aggregate**.
- **Key dependencies:** Google Sheets OAuth2 credential.
- **Edge cases / failures:**
  - Empty sheet: returns no rows; downstream logic must handle missing data (it does).
  - Permissions / OAuth expiry.
  - Incorrect sheet columns can break later expectations (comparison relies on `Target` and `Keyword` fields).

#### Node: Aggregate
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — converts multiple rows into a single item.
- **Configuration:** `aggregateAllItemData` (collect all incoming item JSON into one array field, typically under `data`).
- **Outputs:** To **Clear sheet**.
- **Edge cases / failures:**
  - If upstream returns no items, aggregate may produce an item without `data`; downstream code checks for that.

**Sticky note (applies to this block):**  
“Get previous keywords and clear the sheet … sheet must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=1392477424#gid=1392477424).”

---

### 2.2 Reset output sheet + load targets (Targets sheet)
**Overview:**  
Clears the Keywords sheet (keeping the header row), then loads the list of target domains to iterate through.

**Nodes involved:**  
- Clear sheet  
- Get targets

#### Node: Clear sheet
- **Type / role:** Google Sheets — clears previous “Keywords” content so the run can rebuild the snapshot.
- **Configuration:**
  - Operation: **Clear**
  - `keepFirstRow: true` (preserves header row)
  - Same document/sheet as “Get previous keywords” (Keywords sheet).
- **Input:** From **Aggregate**.
- **Output:** To **Get targets**.
- **Edge cases / failures:**
  - If header row isn’t actually the header, you may keep the wrong row.
  - Clearing the sheet means this workflow is **stateful**: failure mid-run can leave sheet partially rebuilt.

#### Node: Get targets
- **Type / role:** Google Sheets — loads target configuration rows.
- **Configuration:**
  - Document: same spreadsheet
  - Sheet: “Targets”
  - Expected columns used later by expressions: `Domain`, `Language`, `Location`
- **Input:** From **Clear sheet**.
- **Output:** To **Loop over targets**.
- **Edge cases / failures:**
  - Missing required columns will break DataForSEO node expressions.
  - Empty targets sheet -> loop does nothing.

**Sticky note (applies to this block):**  
“Get targets … must have the same columns as in [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=0#gid=0).”

---

### 2.3 Per-target loop: initialize pagination accumulator
**Overview:**  
Iterates target rows one by one. For each target, initializes an `items` array that will accumulate all DataForSEO results across pages.

**Nodes involved:**  
- Loop over targets  
- Initialize "items" field  
- Set "items" field

#### Node: Loop over targets
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — iterates through target rows.
- **Configuration:** Default options; batch size not explicitly set (n8n default is typically 1).
- **Inputs:** From **Get targets** and later from **Send a message** to continue to next target.
- **Outputs:**
  - Output 0: unused (empty)
  - Output 1: to **Initialize "items" field** (this is the “current batch item(s)” path)
- **Edge cases / failures:**
  - If batch size > 1, expressions using `$('Get targets').item` may not behave as intended (they assume a current “target item” context).

#### Node: Initialize "items" field
- **Type / role:** Set (`n8n-nodes-base.set`) — creates an empty accumulator array.
- **Configuration:** Sets field `items` (type array) to `[]`.
- **Input:** From **Loop over targets** (output 1).
- **Output:** To **Set "items" field**.
- **Edge cases / failures:** None typical; very low risk.

#### Node: Set "items" field
- **Type / role:** Set — normalizes the accumulator payload for downstream merging.
- **Configuration:** Sets `items` to `{{$json.items}}` (passes through the current `items` array).
- **Input:** From **Initialize "items" field** and from **Has more pages?** when paginating.
- **Output:** To **Get ranked keywords**.
- **Edge cases / failures:**
  - If `items` is missing (unexpected path), this node would set `items` to `undefined`, breaking later spreads.

---

### 2.4 DataForSEO Labs pagination + item merging
**Overview:**  
Fetches Featured Snippet ranked keywords from DataForSEO Labs API in pages of 1000 items, merging each response into the `items` accumulator until the last page.

**Nodes involved:**  
- Get ranked keywords  
- Merge "items" with DFS response  
- Has more pages?  
- Merge "items" with last response

#### Node: Get ranked keywords
- **Type / role:** DataForSEO Labs API (`n8n-nodes-dataforseo.dataForSeoLabsApi`) — queries ranked keywords.
- **Configuration choices:**
  - Operation: `get-ranked-keywords`
  - `item_types`: `["featured_snippet"]` (only Featured Snippet presence)
  - `limit: 1000`
  - `offset: {{$runIndex * 1000}}` (pagination step)
  - `target_any: {{ $('Get targets').item.json.Domain }}`
  - `language_name: {{ $('Get targets').item.json.Language }}`
  - `location_name: {{ $('Get targets').item.json.Location }}`
- **Input:** From **Set "items" field** (ensures `items` exists before merging).
- **Output:** To **Merge "items" with DFS response**.
- **Edge cases / failures:**
  - Auth errors (DataForSEO credentials).
  - API quota/rate limits.
  - Response shape differences: expressions assume `tasks[0].result[0].items` and `total_count` exist.
  - Large `total_count` can cause long looping.

#### Node: Merge "items" with DFS response
- **Type / role:** Set — appends current page items to accumulator.
- **Configuration (key expression):**
  - `items = [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items ]`
- **Input:** From **Get ranked keywords**.
- **Output:** To **Has more pages?**
- **Edge cases / failures:**
  - If `tasks[0].result[0].items` is absent, spread will fail.
  - If DataForSEO returns empty items, it still works (appends nothing).

#### Node: Has more pages?
- **Type / role:** IF (`n8n-nodes-base.if`) — controls pagination loop.
- **Condition (interpreted):**  
  Continue paging while:
  - `{{$runIndex}} < {{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
- **Outputs:**
  - **True path (index 0):** to **Set "items" field** (fetch next page)
  - **False path (index 1):** to **Merge "items" with last response** (finalize)
- **Edge cases / failures:**
  - If `total_count` missing or not numeric, condition fails (type validation is strict).
  - Off-by-one risks: relies on `total_count/1000 - 1` with `$runIndex`. If `total_count` is exactly multiple of 1000, this still generally works, but rounding behavior can be tricky (e.g., 1000/1000 - 1 = 0).

#### Node: Merge "items" with last response
- **Type / role:** Set — creates the final merged `items` array after pagination.
- **Configuration (key expression):**
  - `items = [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]`
- **Input:** From **Has more pages?** (false branch).
- **Output:** To **Split out (items)**.
- **Edge cases / failures:**
  - Similar risks if DataForSEO response path is missing.

**Sticky note (applies to this block):**  
“Get newly earned Featured Snippet keywords with DataForSEO … set up additional parameters if needed.”

---

### 2.5 Persist refreshed keyword snapshot (Keywords sheet)
**Overview:**  
Rebuilds the Keywords sheet content for the current run: split the accumulated items into individual rows and append them.

**Nodes involved:**  
- Split out (items)  
- Append keyword in sheet  
- Aggregate1

#### Node: Split out (items)
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — converts one item with an array field into many items.
- **Configuration:** `fieldToSplitOut: "items"`.
- **Input:** From **Merge "items" with last response**.
- **Output:** To **Append keyword in sheet**.
- **Edge cases / failures:**
  - If `items` is missing or not an array, node may output nothing.

#### Node: Append keyword in sheet
- **Type / role:** Google Sheets — appends each ranked keyword record to the Keywords sheet.
- **Configuration:**
  - Operation: **Append**
  - Mapping mode: define columns explicitly
  - Column mappings (key expressions):
    - `Target = {{ $('Get targets').item.json.Domain }}`
    - `Keyword = {{ $json.keyword_data.keyword }}`
    - `Date = {{ new Date().toLocaleDateString('en-CA') }}` (YYYY-MM-DD in most environments)
    - `Rank = {{ $json.ranked_serp_element.serp_item.rank_group }}`
    - `Search Volume = {{ $json.keyword_data.keyword_info.search_volume }}`
    - `Url = {{ $json.ranked_serp_element.serp_item.url }}`
    - `SERP Item Types = {{ $json.ranked_serp_element.serp_item_types.join(', ') }}`
- **Input:** From **Split out (items)** (each item is one DataForSEO “ranked keyword” record).
- **Output:** To **Aggregate1**.
- **Edge cases / failures:**
  - If any nested fields are missing (`ranked_serp_element`, `keyword_data`, etc.), expressions can error.
  - Google Sheets API limits / throttling when appending many rows.
  - Sheet column headers must match mapping intention (as per example sheet).

#### Node: Aggregate1
- **Type / role:** Aggregate — collects all appended items into one item so the workflow can compute a single diff.
- **Configuration:** `aggregateAllItemData`
- **Input:** From **Append keyword in sheet**.
- **Output:** To **Find new keywords**.
- **Edge cases / failures:** Very large item sets can increase memory usage.

---

### 2.6 Compare old vs new + email new wins
**Overview:**  
Computes which Featured Snippet keywords are new for the current target compared to the previous snapshot. If there are new wins, sends an email summary, then continues looping to the next target.

**Nodes involved:**  
- Find new keywords  
- Filter (has new keywords)  
- Send a message

#### Node: Find new keywords
- **Type / role:** Code (`n8n-nodes-base.code`) — calculates set difference.
- **Logic summary:**
  - Builds `oldKeywords` set from `Aggregate.first().json.data`, filtered to rows where `Target == current Domain`, then maps `Keyword`.
  - Builds `newKeywords` list from `Merge "items" with last response`.first().json.items mapped to `item.keyword_data.keyword`.
  - `diff = newKeywords.filter(x => !oldKeywords.has(x))` if oldKeywords exists; otherwise `diff = []`.
  - Returns a single item `{ diff }`.
- **Inputs:** From **Aggregate1** but references other nodes directly via `$('...')`.
- **Output:** To **Filter (has new keywords)**.
- **Edge cases / failures:**
  - If `Aggregate.first().json.data` is missing, it handles it.
  - If `Merge "items" with last response`.first().json.items is missing, it handles it (diff becomes empty).
  - If keywords contain casing/whitespace differences, they will be treated as different (no normalization).

#### Node: Filter (has new keywords)
- **Type / role:** Filter (`n8n-nodes-base.filter`) — only proceed when `diff` is non-empty.
- **Condition:** `{{$json.diff}}` is **not empty**.
- **Input:** From **Find new keywords**.
- **Output:** To **Send a message**.
- **Edge cases / failures:**
  - If `diff` is missing/not an array, strict validation could block. (Current code always returns an array.)

#### Node: Send a message
- **Type / role:** Gmail (`n8n-nodes-base.gmail`) — sends email notification.
- **Configuration:**
  - To: `user@example.com` (placeholder)
  - Subject: “New Featured Snippet Keywords for Your Domain”
  - HTML message includes:
    - Target domain: `{{ $('Get targets').item.json.Domain }}`
    - Keyword list: `{{ $json.diff.join(', ') }}`
- **Input:** From **Filter (has new keywords)**.
- **Output:** To **Loop over targets** (continues loop to next domain).
- **Edge cases / failures:**
  - Gmail OAuth expired / insufficient scopes.
  - Very large `diff` may exceed email length or be unreadable; consider truncation.
  - If the workflow should continue even when no new keywords, currently it will **not** send mail and also **will not loop forward from this node**; continuation depends on `Split In Batches` configuration and whether another path calls it again (in this design, only “Send a message” feeds the loop).

**Sticky note (applies to this block):**  
“Save new Featured Snippet keywords … and send an email … Create a Gmail connection …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every Monday | Schedule Trigger | Weekly entry point | — | Get previous keywords | This workflow uses DataForSEO Labs API to identify new search queries where your domain started appearing in Google’s Featured Snippet results. Newly detected keywords are saved to Google Sheets, and a brief email summary is sent to you so you can quickly review recent wins. \| Setup: schedule weekly; connect Sheets, DataForSEO, Gmail. |
| Get previous keywords | Google Sheets | Read previous keyword snapshot | Run every Monday | Aggregate | ## Get previous keywords and clear the sheet. Sheet must match [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=1392477424#gid=1392477424). |
| Aggregate | Aggregate | Combine prior rows into one array | Get previous keywords | Clear sheet | ## Get previous keywords and clear the sheet. Sheet must match [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=1392477424#gid=1392477424). |
| Clear sheet | Google Sheets | Clear Keywords sheet (keep header) | Aggregate | Get targets | ## Get previous keywords and clear the sheet. Sheet must match [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=1392477424#gid=1392477424). |
| Get targets | Google Sheets | Load targets list | Clear sheet | Loop over targets | ## Get targets. Must match [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=0#gid=0). |
| Loop over targets | Split In Batches | Iterate each target row | Get targets; Send a message | Initialize "items" field | ## Get targets. Must match [this Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=0#gid=0). |
| Initialize "items" field | Set | Initialize accumulator array | Loop over targets | Set "items" field | ## Get newly earned Featured Snippet keywords with DataForSEO. Create DataForSEO connection and adjust parameters if needed. |
| Set "items" field | Set | Maintain accumulator payload across pages | Initialize "items" field; Has more pages? (true) | Get ranked keywords | ## Get newly earned Featured Snippet keywords with DataForSEO. Create DataForSEO connection and adjust parameters if needed. |
| Get ranked keywords | DataForSEO Labs API | Fetch ranked keywords (Featured Snippet) with paging | Set "items" field | Merge "items" with DFS response | ## Get newly earned Featured Snippet keywords with DataForSEO. Create DataForSEO connection and adjust parameters if needed. |
| Merge "items" with DFS response | Set | Append current page items into accumulator | Get ranked keywords | Has more pages? | ## Get newly earned Featured Snippet keywords with DataForSEO. Create DataForSEO connection and adjust parameters if needed. |
| Has more pages? | IF | Determine whether to request next page | Merge "items" with DFS response | Set "items" field (true); Merge "items" with last response (false) | ## Get newly earned Featured Snippet keywords with DataForSEO. Create DataForSEO connection and adjust parameters if needed. |
| Merge "items" with last response | Set | Final merged items array for this target | Has more pages? (false) | Split out (items) | ## Get newly earned Featured Snippet keywords with DataForSEO. Create DataForSEO connection and adjust parameters if needed. |
| Split out (items) | Split Out | Convert accumulated array into individual items | Merge "items" with last response | Append keyword in sheet | ## Save new Featured Snippet keywords to a spreadsheet and send an email. Use same spreadsheet; connect Gmail; set receiver. |
| Append keyword in sheet | Google Sheets | Append refreshed keyword rows | Split out (items) | Aggregate1 | ## Save new Featured Snippet keywords to a spreadsheet and send an email. Use same spreadsheet; connect Gmail; set receiver. |
| Aggregate1 | Aggregate | Combine appended items for diff computation | Append keyword in sheet | Find new keywords | ## Save new Featured Snippet keywords to a spreadsheet and send an email. Use same spreadsheet; connect Gmail; set receiver. |
| Find new keywords | Code | Compute new keyword wins vs previous snapshot | Aggregate1 | Filter (has new keywords) | ## Save new Featured Snippet keywords to a spreadsheet and send an email. Use same spreadsheet; connect Gmail; set receiver. |
| Filter (has new keywords) | Filter | Only proceed if diff is non-empty | Find new keywords | Send a message | ## Save new Featured Snippet keywords to a spreadsheet and send an email. Use same spreadsheet; connect Gmail; set receiver. |
| Send a message | Gmail | Email summary for the current target | Filter (has new keywords) | Loop over targets | ## Save new Featured Snippet keywords to a spreadsheet and send an email. Use same spreadsheet; connect Gmail; set receiver. |
| Sticky Note | Sticky Note | Comment | — | — | ## Get previous keywords and clear the sheet … [Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=1392477424#gid=1392477424). |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Get targets … [Example](https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=0#gid=0). |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Get newly earned Featured Snippet keywords with DataForSEO … |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Save new Featured Snippet keywords … Create Gmail connection … |
| Sticky Note6 | Sticky Note | Comment | — | — | Uses DataForSEO Labs API … Setup includes [API login and password](https://app.dataforseo.com/api-access). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and set its name to:  
   “Get new Featured Snippet keyword wins for your domain via email with DataForSEO”.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: Weekly → Monday → 09:00 (adjust timezone as needed).
   - This is the entry node.

3. **Add Google Sheets node: “Get previous keywords”**
   - Node type: **Google Sheets**
   - Credentials: create/select **Google Sheets OAuth2**.
   - Document: your spreadsheet
   - Sheet: `Keywords`
   - Operation: read/get all rows (default “Read” behavior in the UI).
   - Enable **Always Output Data** (so downstream runs even if empty).
   - Connect: Schedule Trigger → Get previous keywords.

4. **Add Aggregate node: “Aggregate”**
   - Node type: **Aggregate**
   - Operation: **Aggregate All Item Data**
   - Connect: Get previous keywords → Aggregate

5. **Add Google Sheets node: “Clear sheet”**
   - Operation: **Clear**
   - Document: same spreadsheet
   - Sheet: `Keywords`
   - Enable: **Keep First Row** = true
   - Connect: Aggregate → Clear sheet

6. **Add Google Sheets node: “Get targets”**
   - Operation: read/get rows
   - Document: same spreadsheet
   - Sheet: `Targets`
   - Ensure target columns exist: `Domain`, `Language`, `Location`
   - Connect: Clear sheet → Get targets

7. **Add Split In Batches node: “Loop over targets”**
   - Node type: **Split In Batches**
   - Batch size: 1 (recommended)
   - Connect: Get targets → Loop over targets
   - Use the “continue” input later by connecting Gmail node back into it.

8. **Add Set node: “Initialize "items" field”**
   - Add field `items` of type **Array** with value `[]`
   - Connect: Loop over targets (batch output) → Initialize "items" field

9. **Add Set node: “Set "items" field”**
   - Set `items` (Array) = `{{$json.items}}`
   - Connect: Initialize "items" field → Set "items" field

10. **Add DataForSEO node: “Get ranked keywords”**
   - Node type: **DataForSEO Labs API**
   - Credentials: create/select DataForSEO (API login/password from: https://app.dataforseo.com/api-access)
   - Operation: `get-ranked-keywords`
   - Parameters:
     - `item_types`: `featured_snippet`
     - `limit`: `1000`
     - `offset`: `{{$runIndex * 1000}}`
     - `target_any`: `{{ $('Get targets').item.json.Domain }}`
     - `language_name`: `{{ $('Get targets').item.json.Language }}`
     - `location_name`: `{{ $('Get targets').item.json.Location }}`
   - Connect: Set "items" field → Get ranked keywords

11. **Add Set node: “Merge "items" with DFS response”**
   - Set `items` (Array) to:  
     `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items ] }}`
   - Connect: Get ranked keywords → Merge "items" with DFS response

12. **Add IF node: “Has more pages?”**
   - Condition (Number):  
     Left: `{{$runIndex}}`  
     Operator: “is less than”  
     Right: `{{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
   - Connect: Merge "items" with DFS response → Has more pages?
   - True output → Set "items" field (to fetch next page)
   - False output → next step (final merge)

13. **Add Set node: “Merge "items" with last response”**
   - Set `items` (Array) to:  
     `{{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items] }}`
   - Connect: Has more pages? (false) → Merge "items" with last response

14. **Add Split Out node: “Split out (items)”**
   - Field to split out: `items`
   - Connect: Merge "items" with last response → Split out (items)

15. **Add Google Sheets node: “Append keyword in sheet”**
   - Operation: **Append**
   - Document: same spreadsheet
   - Sheet: `Keywords`
   - Map columns (create headers to match):
     - Target, Keyword, Date, Rank, Search Volume, Url, SERP Item Types
   - Use expressions:
     - Target: `{{ $('Get targets').item.json.Domain }}`
     - Keyword: `{{ $json.keyword_data.keyword }}`
     - Date: `{{ new Date().toLocaleDateString('en-CA') }}`
     - Rank: `{{ $json.ranked_serp_element.serp_item.rank_group }}`
     - Search Volume: `{{ $json.keyword_data.keyword_info.search_volume }}`
     - Url: `{{ $json.ranked_serp_element.serp_item.url }}`
     - SERP Item Types: `{{ $json.ranked_serp_element.serp_item_types.join(', ') }}`
   - Connect: Split out (items) → Append keyword in sheet

16. **Add Aggregate node: “Aggregate1”**
   - Operation: **Aggregate All Item Data**
   - Connect: Append keyword in sheet → Aggregate1

17. **Add Code node: “Find new keywords”**
   - Paste logic that:
     - Builds old keyword set from `Aggregate` output, filtered by current target domain
     - Extracts new keyword list from merged items
     - Returns `{ diff }`
   - Connect: Aggregate1 → Find new keywords

18. **Add Filter node: “Filter (has new keywords)”**
   - Condition: `{{$json.diff}}` is **not empty**
   - Connect: Find new keywords → Filter (has new keywords)

19. **Add Gmail node: “Send a message”**
   - Credentials: **Gmail OAuth2**
   - To: your recipient(s)
   - Subject: “New Featured Snippet Keywords for Your Domain”
   - HTML Message example:  
     `New keywords for which your target {{ $('Get targets').item.json.Domain }} ranks in featured snippet: {{ $json.diff.join(', ') }}`
   - Connect: Filter (has new keywords) → Send a message

20. **Close the loop**
   - Connect: Send a message → Loop over targets (so the next target is processed)

21. **Activate the workflow** (it is inactive by default in the provided JSON).

**Important design note:** As built, the loop continues via the Gmail node connection. If there are **no new keywords**, the Gmail node is skipped and the loop may not advance. To ensure it always advances, add an additional connection from a “no new keywords” branch (or from “Find new keywords”) back into **Loop over targets**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Example Keywords sheet structure required (columns must match). | https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=1392477424#gid=1392477424 |
| Example Targets sheet structure required. | https://docs.google.com/spreadsheets/d/1pRNkz1us8N_w_Sw-8axiMxUDu7to2tIVFXyY5W7gH3U/edit?gid=0#gid=0 |
| DataForSEO API credentials source (login/password). | https://app.dataforseo.com/api-access |
| Workflow behavior summary: scheduled run → read old snapshot → fetch new Featured Snippet rankings → compare → store → email. | Sticky note “This workflow uses DataForSEO Labs API…” (included in workflow) |