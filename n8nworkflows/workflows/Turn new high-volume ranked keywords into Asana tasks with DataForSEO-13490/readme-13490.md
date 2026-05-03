Turn new high-volume ranked keywords into Asana tasks with DataForSEO

https://n8nworkflows.xyz/workflows/turn-new-high-volume-ranked-keywords-into-asana-tasks-with-dataforseo-13490


# Turn new high-volume ranked keywords into Asana tasks with DataForSEO

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs on a weekly schedule to detect **new Google-ranked keywords with high search volume (SV > 1000)** for one or more target domains using **DataForSEO Labs API**. It then:
1) stores the latest snapshot into **Google Sheets**,  
2) compares the snapshot with the previous one to find **newly ranked keywords**,  
3) creates a follow-up **Asana task** if new keywords exist, and  
4) sends a **Slack summary**.

**Target use cases:**
- SEO teams tracking new keyword opportunities for domains.
- Content teams generating tasks based on newly ranked, high-volume queries.
- Automated weekly reporting (tasking + notification).

### 1.1 Logical blocks
1. **Scheduling & Previous Snapshot Load (Google Sheets)**
2. **Reset Keywords Sheet & Load Targets**
3. **Per-Target Keyword Fetch (DataForSEO) with Pagination Accumulation**
4. **Write Snapshot to Sheet**
5. **Detect New Keywords vs Previous Snapshot**
6. **Conditional Actions: Asana Task + Slack Notification**
7. **Loop Control (iterate targets)**

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Previous Snapshot Load
**Overview:** Triggers weekly and loads the previous keywords snapshot from Google Sheets, aggregating all rows so later steps can compare “old vs new”.

**Nodes involved:**
- Run every Monday
- Get previous keywords
- Aggregate

#### Node: Run every Monday
- **Type / role:** Schedule Trigger; starts the workflow.
- **Config:** Weekly at **Monday 09:00** (server/workflow timezone).
- **Outputs to:** Get previous keywords
- **Failure modes / edge cases:** time-zone mismatch; workflow inactive means it won’t fire.

#### Node: Get previous keywords
- **Type / role:** Google Sheets node (read); retrieves previously stored keywords snapshot.
- **Config choices:**
  - Spreadsheet: “New Ranked Keywords with SV>1000 from Google with DataForSEO”
  - Sheet: “Keywords”
  - Operation: (implicit read/get all rows; default)
  - **alwaysOutputData: true** so downstream nodes still run even if sheet is empty.
- **Outputs to:** Aggregate
- **Failure modes:** OAuth expired; sheet ID changed; permissions; API quota/429; empty sheet (handled by alwaysOutputData).

#### Node: Aggregate
- **Type / role:** Aggregate; collects all incoming items into one structure.
- **Config:** `aggregateAllItemData`
- **Outputs to:** Clear sheet
- **What it produces:** A single item that typically contains an array of rows under `.json.data` (used later by code).
- **Edge cases:** If Google Sheets returns no rows, `.json.data` may be missing; downstream code checks for presence.

**Sticky note context (applies to this block):**
- “## Get previous keywords and clear the sheet … [this Example](https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=1681593026#gid=1681593026).”

---

### Block 2 — Reset Keywords Sheet & Load Targets
**Overview:** Clears the “Keywords” sheet (keeping headers) and loads the list of target domains (and their language/location) from another sheet.

**Nodes involved:**
- Clear sheet
- Get targets

#### Node: Clear sheet
- **Type / role:** Google Sheets node; clears old snapshot to prepare for writing the new snapshot.
- **Config:**
  - Operation: **Clear**
  - keepFirstRow: **true** (preserves header row)
  - Targets the same “Keywords” sheet used for snapshot storage.
- **Inputs from:** Aggregate
- **Outputs to:** Get targets
- **Failure modes:** same as Google Sheets auth/permissions; risk of clearing wrong sheet if documentId/sheetName misconfigured.

#### Node: Get targets
- **Type / role:** Google Sheets node (read); loads target domains and related parameters.
- **Config:**
  - Spreadsheet: same doc
  - Sheet: “Targets”
  - Operation: (implicit read/get rows)
- **Inputs from:** Clear sheet
- **Outputs to:** Loop over targets
- **Expected columns (used later):** `Domain`, `Language`, `Location`
- **Failure modes:** missing expected columns → expressions in DataForSEO node fail; empty targets list → loop does nothing.

**Sticky note context (applies to this block):**
- “## Get targets … [this Example](https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=0#gid=0).”

---

### Block 3 — Per-Target Keyword Fetch (DataForSEO) + Pagination Accumulation
**Overview:** Iterates through each target domain, requests ranked keywords with SV > 1000 from DataForSEO Labs, and accumulates all returned items across pages.

**Nodes involved:**
- Loop over targets
- Initialize "items" field
- Set "items" field
- Get ranked keywords
- Merge "items" with DFS response
- Has more pages?

#### Node: Loop over targets
- **Type / role:** Split In Batches; used here as a loop controller over target rows.
- **Config:** default batch settings (no explicit batch size shown).
- **Input from:** Get targets
- **Output connections:**
  - Output **0**: unused
  - Output **1**: Initialize "items" field (this is the “current batch” output in many n8n versions)
- **Edge cases / failures:** if no items, loop body never runs; if batch size default is too large, could stress API rate limits.

#### Node: Initialize "items" field
- **Type / role:** Set; initializes an accumulator array for paginated results.
- **Config:** sets `items = []` (array).
- **Inputs from:** Loop over targets
- **Outputs to:** Set "items" field
- **Edge cases:** none significant.

#### Node: Set "items" field
- **Type / role:** Set; passes `items` forward (acts as a staging node for later merges).
- **Config:** sets `items = {{$json.items}}`
- **Inputs from:** Initialize "items" field OR Has more pages? (true branch)
- **Outputs to:** Get ranked keywords
- **Edge cases:** if `items` missing (shouldn’t happen due to initializer), later merge expressions could fail.

#### Node: Get ranked keywords
- **Type / role:** DataForSEO Labs API; fetch ranked keywords for the current target with filter SV > 1000.
- **Config choices:**
  - Operation: **get-ranked-keywords**
  - `target_any`: `{{ $('Get targets').item.json.Domain }}`
  - `language_name`: `{{ $('Get targets').item.json.Language }}`
  - `location_name`: `{{ $('Get targets').item.json.Location }}`
  - Pagination:
    - `limit`: **1000**
    - `offset`: `{{ $runIndex * 1000 }}`
  - Filter: `["keyword_data.keyword_info.search_volume", ">", 1000]`
- **Inputs from:** Set "items" field
- **Outputs to:** Merge "items" with DFS response
- **Failure modes:**
  - Bad credentials (DataForSEO login/password)
  - API quota/rate limits
  - Target parameters invalid (language/location names must match DataForSEO expected values)
  - Response shape changes (expressions using `tasks[0].result[0]...` could break)

#### Node: Merge "items" with DFS response
- **Type / role:** Set; appends the current page of DataForSEO items to the accumulator.
- **Config expression:**
  - `items = {{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`
- **Inputs from:** Get ranked keywords
- **Outputs to:** Has more pages?
- **Edge cases:**
  - If `result[0].items` missing (no results), expression may error unless DataForSEO returns an empty array reliably.
  - If `items` gets very large, memory usage increases.

#### Node: Has more pages?
- **Type / role:** IF; determines whether more paginated API calls are needed.
- **Condition:**  
  `{{$runIndex}} < {{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
- **Outputs:**
  - **True branch →** Set "items" field (continue loop / next page)
  - **False branch →** Merge "items" with last response (finalize)
- **Edge cases:**
  - `total_count` missing or 0 can cause division issues or negative thresholds.
  - Off-by-one risks depending on how DataForSEO counts pages; current formula assumes pages of 1000.

**Sticky note context (applies to this block):**
- “## Getn high-volume ranked keywords with DataForSEO Create a DataForSEO connection and set up additional parameters if needed.”

---

### Block 4 — Finalize Accumulated Items & Split to Rows
**Overview:** After pagination ends for a target, the workflow creates the final `items` array and splits it into individual keyword records for writing to Google Sheets.

**Nodes involved:**
- Merge "items" with last response
- Split out (items)
- Append keyword in sheet

#### Node: Merge "items" with last response
- **Type / role:** Set; produces the final `items` array to process.
- **Config expression:**
  - `items = {{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]}}`
- **Inputs from:** Has more pages? (false branch)
- **Outputs to:** Split out (items)
- **Edge cases:** If final page already merged in prior step, this may duplicate the last page (depends on how n8n executes branches/runIndex). In this workflow design, it’s intended to ensure the last response is included; validate no duplicates by testing with small `total_count`.

#### Node: Split out (items)
- **Type / role:** Split Out; converts the `items` array into one item per keyword record.
- **Config:** `fieldToSplitOut = "items"`
- **Inputs from:** Merge "items" with last response
- **Outputs to:** Append keyword in sheet
- **Edge cases:** if `items` empty → produces no output items; downstream nodes won’t run.

#### Node: Append keyword in sheet
- **Type / role:** Google Sheets append; writes each ranked keyword row into the Keywords sheet (new snapshot).
- **Config choices:**
  - Operation: **Append**
  - Mapping mode: “Define below” with explicit columns.
  - Writes:
    - **Target** = `{{ $('Get targets').item.json.Domain }}`
    - **Keyword** = `{{ $json.keyword_data.keyword }}`
    - **Date** = `{{ new Date().toLocaleDateString('en-CA') }}` (YYYY-MM-DD)
    - **Rank** = `{{ $json.ranked_serp_element.serp_item.rank_group }}`
    - **Search Volume** = `{{ $json.keyword_data.keyword_info.search_volume }}`
    - **Url** = `{{ $json.ranked_serp_element.serp_item.url }}`
    - **SERP Item Types** = `{{ $json.ranked_serp_element.serp_item_types.join(', ') }}`
- **Inputs from:** Split out (items)
- **Outputs to:** Aggregate1
- **Failure modes:** Google Sheets quota; schema mismatch (header names must match); missing nested fields in DataForSEO response (null references in expressions).

**Sticky note context (applies to this block and later actions):**
- “## Save high-volume ranked keywords … create an Asana task and send a Slack notification …”

---

### Block 5 — Detect New Keywords vs Previous Snapshot
**Overview:** Aggregates the newly written snapshot for the target and compares it with the previous snapshot (loaded at the start) to compute the keyword difference.

**Nodes involved:**
- Aggregate1
- Find new keywords
- Filter (has new keywords)

#### Node: Aggregate1
- **Type / role:** Aggregate; collects all appended sheet rows (for the current processing chain) into a single dataset for comparison.
- **Config:** `aggregateAllItemData`
- **Inputs from:** Append keyword in sheet
- **Outputs to:** Find new keywords
- **Edge cases:** Large datasets can increase memory usage.

#### Node: Find new keywords
- **Type / role:** Code; computes `diff` = keywords present in current DataForSEO results but not present in old snapshot for the same target.
- **Key logic (interpreted):**
  - Builds `oldKeywords` from `$('Aggregate').first().json.data` filtered by `Target == current Domain`, mapped to `Keyword`.
  - Extracts `newKeywords` from `$('Merge "items" with last response').first().json.items`, mapped to `keyword_data.keyword`.
  - `diff = newKeywords.filter(x => !oldKeywords.has(x))` (only if oldKeywords non-empty; otherwise diff is empty).
- **Output:** single item `{ diff: [...] }`
- **Inputs from:** Aggregate1
- **Outputs to:** Filter (has new keywords)
- **Edge cases / failure types:**
  - If old sheet is empty, this code returns `diff = []` (so it will *not* create tasks on the first run). This is intentional but may be surprising.
  - If DataForSEO returns items but `Merge "items" with last response"` is empty/undefined, `newKeywords` becomes `[]`.
  - Target matching relies on exact string equality of `Target` vs `Domain` (case/spacing issues can cause false “new” results).

#### Node: Filter (has new keywords)
- **Type / role:** Filter; only continue if `diff` array is not empty.
- **Condition:** `{{$json.diff}}` **notEmpty**
- **Inputs from:** Find new keywords
- **Outputs to:** Create a task
- **Edge cases:** If `diff` is undefined due to upstream code errors, filter may drop everything.

---

### Block 6 — Conditional Actions: Asana Task + Slack Summary
**Overview:** If new keywords exist for the target, creates one Asana task containing the keyword list, then posts a Slack message summary.

**Nodes involved:**
- Create a task
- Send a message

#### Node: Create a task
- **Type / role:** Asana; creates a follow-up task for content creation.
- **Config choices:**
  - Task name: “Create content for new keywords with high search volume”
  - Workspace: `112670807337085`
  - Notes (templated):
    - `New Keyword with SV > 1000 apeared for {{Domain}}: {{ diff.join(', ') }}. Create content for these keywords.`
  - Assignee: `1207548230460014`
  - Projects: includes `1207548+1234567890` (ensure this is a valid project GID; the `+` looks suspicious and may be a placeholder).
- **Inputs from:** Filter (has new keywords)
- **Outputs to:** Send a message
- **Failure modes:** invalid workspace/project/assignee IDs; permission errors; Asana rate limits.

#### Node: Send a message
- **Type / role:** Slack; posts a summary message.
- **Auth:** OAuth2
- **Text:**  
  `New ranked keywords with SV > 1000 for target {{Domain}}: {{ diff.join(', ') }}`
- **Inputs from:** Create a task
- **Outputs to:** Loop over targets (continue to next target)
- **Failure modes:** OAuth expired; missing channel/receiver configuration in node UI; Slack rate limits.

---

### Block 7 — Loop Control / Iteration Completion
**Overview:** After Slack notification (or if no new keywords, note that the current design does not explicitly “continue”), the workflow triggers the next target iteration via SplitInBatches.

**Nodes involved:**
- Loop over targets (re-entry from Send a message)

#### Node: Loop over targets (re-entry)
- **Type / role:** Loop continuation.
- **Inputs from:** Send a message
- **Important behavior:** This is the mechanism to proceed to the next target **only when the Slack node runs**.
- **Edge case (design caveat):** If there are **no new keywords**, the Filter stops the branch before “Send a message”, so **the loop may not continue to the next target**. In practice, this can cause the workflow to process only the first target that yields no new keywords.  
  **Mitigation:** add an “else” path from the Filter back to Loop over targets, or move the loop continuation connection earlier.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run every Monday | Schedule Trigger | Weekly start trigger | — | Get previous keywords | This workflow uses the DataForSEO Labs API to detect new high-volume search queries… (full note content) |
| Get previous keywords | Google Sheets | Read previous keyword snapshot | Run every Monday | Aggregate | ## Get previous keywords and clear the sheet … [this Example](https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=1681593026#gid=1681593026). |
| Aggregate | Aggregate | Aggregate old snapshot rows into one item | Get previous keywords | Clear sheet | ## Get previous keywords and clear the sheet … [this Example](https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=1681593026#gid=1681593026). |
| Clear sheet | Google Sheets | Clear Keywords sheet (keep header) | Aggregate | Get targets | ## Get previous keywords and clear the sheet … [this Example](https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=1681593026#gid=1681593026). |
| Get targets | Google Sheets | Read Targets list (Domain/Language/Location) | Clear sheet | Loop over targets | ## Get targets … [this Example](https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=0#gid=0). |
| Loop over targets | Split In Batches | Iterate over each target row | Get targets; Send a message | Initialize "items" field |  |
| Initialize "items" field | Set | Initialize accumulator array | Loop over targets | Set "items" field | ## Getn high-volume ranked keywords with DataForSEO Create a DataForSEO connection and set up additional parameters if needed. |
| Set "items" field | Set | Pass/maintain accumulator | Initialize "items" field; Has more pages? (true) | Get ranked keywords | ## Getn high-volume ranked keywords with DataForSEO Create a DataForSEO connection and set up additional parameters if needed. |
| Get ranked keywords | DataForSEO Labs API | Fetch ranked keywords (SV>1000), paginated | Set "items" field | Merge "items" with DFS response | ## Getn high-volume ranked keywords with DataForSEO Create a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with DFS response | Set | Append current page to accumulator | Get ranked keywords | Has more pages? | ## Getn high-volume ranked keywords with DataForSEO Create a DataForSEO connection and set up additional parameters if needed. |
| Has more pages? | IF | Pagination control | Merge "items" with DFS response | Set "items" field (true); Merge "items" with last response (false) | ## Getn high-volume ranked keywords with DataForSEO Create a DataForSEO connection and set up additional parameters if needed. |
| Merge "items" with last response | Set | Finalize items array for splitting | Has more pages? (false) | Split out (items) | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Split out (items) | Split Out | Convert items array to individual items | Merge "items" with last response | Append keyword in sheet | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Append keyword in sheet | Google Sheets | Append snapshot rows to Keywords sheet | Split out (items) | Aggregate1 | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Aggregate1 | Aggregate | Aggregate newly appended rows for comparison | Append keyword in sheet | Find new keywords | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Find new keywords | Code | Diff old vs new keywords for current target | Aggregate1 | Filter (has new keywords) | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Filter (has new keywords) | Filter | Continue only if diff not empty | Find new keywords | Create a task | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Create a task | Asana | Create follow-up task listing new keywords | Filter (has new keywords) | Send a message | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Send a message | Slack | Notify summary of new keywords | Create a task | Loop over targets | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Sticky Note | Sticky Note | Comment | — | — | ## Get previous keywords and clear the sheet … |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Get targets … |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Getn high-volume ranked keywords with DataForSEO … |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Save high-volume ranked keywords… create an Asana task and send a Slack notification … |
| Sticky Note6 | Sticky Note | Comment | — | — | This workflow uses the DataForSEO Labs API… (full note content) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n and name it:  
   **“Turn new high-volume ranked keywords into Asana tasks with DataForSEO”**

2) **Add Schedule Trigger**
   - Node: *Schedule Trigger*
   - Set rule: **Every week**, **Monday**, at **09:00**.

3) **Add Google Sheets: Get previous keywords**
   - Node: *Google Sheets*
   - Credentials: Google Sheets OAuth2 (account with access to the spreadsheet)
   - Select Document: your spreadsheet
   - Select Sheet: **Keywords**
   - Operation: read/get rows (default)
   - Enable **Always Output Data** (important for empty sheet runs).

4) **Add Aggregate (old snapshot)**
   - Node: *Aggregate*
   - Operation: **Aggregate All Item Data**.
   - Connect: Schedule Trigger → Get previous keywords → Aggregate.

5) **Add Google Sheets: Clear sheet**
   - Node: *Google Sheets*
   - Operation: **Clear**
   - Document: same spreadsheet
   - Sheet: **Keywords**
   - Enable **Keep First Row** = true
   - Connect: Aggregate → Clear sheet

6) **Add Google Sheets: Get targets**
   - Node: *Google Sheets*
   - Operation: read/get rows (default)
   - Document: same spreadsheet
   - Sheet: **Targets**
   - Ensure Targets sheet has columns at least: **Domain**, **Language**, **Location**
   - Connect: Clear sheet → Get targets

7) **Add Split In Batches: Loop over targets**
   - Node: *Split In Batches*
   - Use default options (or set batch size = 1 for strict per-target processing).
   - Connect: Get targets → Loop over targets
   - Use the loop output that represents “current batch” (commonly output 1 in some versions).

8) **Add Set: Initialize "items" field**
   - Node: *Set*
   - Add field:
     - `items` (type **Array**) = `[]`
   - Connect: Loop over targets → Initialize "items" field

9) **Add Set: Set "items" field**
   - Node: *Set*
   - Add field:
     - `items` (Array) = `{{$json.items}}`
   - Connect: Initialize "items" field → Set "items" field

10) **Add DataForSEO: Get ranked keywords**
   - Node: *DataForSEO Labs API*
   - Credentials: DataForSEO (API login + password from https://app.dataforseo.com/api-access)
   - Operation: **Get ranked keywords**
   - Set:
     - Limit = **1000**
     - Offset = `{{$runIndex * 1000}}`
     - Filters = `["keyword_data.keyword_info.search_volume", ">", 1000]`
     - Target Any = `{{ $('Get targets').item.json.Domain }}`
     - Language Name = `{{ $('Get targets').item.json.Language }}`
     - Location Name = `{{ $('Get targets').item.json.Location }}`
   - Connect: Set "items" field → Get ranked keywords

11) **Add Set: Merge "items" with DFS response**
   - Node: *Set*
   - Field:
     - `items` = `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items] }}`
   - Connect: Get ranked keywords → Merge "items" with DFS response

12) **Add IF: Has more pages?**
   - Node: *IF*
   - Condition (Number, LT):
     - Left: `{{$runIndex}}`
     - Right: `{{ $('Get ranked keywords').item.json.tasks[0].result[0].total_count / 1000 - 1}}`
   - Connect: Merge "items" with DFS response → Has more pages?
   - Connect **True** → Set "items" field (to fetch next page)

13) **Add Set: Merge "items" with last response**
   - Node: *Set*
   - Field:
     - `items` = `{{ [...$('Set "items" field').item.json.items, ... $('Get ranked keywords').item.json.tasks[0].result[0].items]}}`
   - Connect **False** from Has more pages? → Merge "items" with last response

14) **Add Split Out (items)**
   - Node: *Split Out*
   - Field to split out: `items`
   - Connect: Merge "items" with last response → Split Out (items)

15) **Add Google Sheets: Append keyword in sheet**
   - Node: *Google Sheets*
   - Operation: **Append**
   - Document: same spreadsheet
   - Sheet: **Keywords**
   - Map columns exactly to your header names:
     - Target = `{{ $('Get targets').item.json.Domain }}`
     - Keyword = `{{ $json.keyword_data.keyword }}`
     - Date = `{{ new Date().toLocaleDateString('en-CA') }}`
     - Rank = `{{ $json.ranked_serp_element.serp_item.rank_group }}`
     - Search Volume = `{{ $json.keyword_data.keyword_info.search_volume }}`
     - Url = `{{ $json.ranked_serp_element.serp_item.url }}`
     - SERP Item Types = `{{ $json.ranked_serp_element.serp_item_types.join(', ') }}`
   - Connect: Split Out (items) → Append keyword in sheet

16) **Add Aggregate (new snapshot for this run)**
   - Node: *Aggregate*
   - Operation: **Aggregate All Item Data**
   - Connect: Append keyword in sheet → Aggregate1

17) **Add Code: Find new keywords**
   - Node: *Code*
   - Paste logic equivalent to:
     - Build `oldKeywords` from the first Aggregate node (old snapshot), filtered by current Domain.
     - Build `newKeywords` from `Merge "items" with last response`.  
     - Output `{ diff }`.
   - Connect: Aggregate1 → Find new keywords

18) **Add Filter: Filter (has new keywords)**
   - Node: *Filter*
   - Condition: Array `diff` **not empty**
   - Connect: Find new keywords → Filter (has new keywords)

19) **Add Asana: Create a task**
   - Node: *Asana*
   - Credentials: Asana OAuth
   - Workspace: select your workspace
   - Set:
     - Task name as desired
     - Notes: include `{{ $('Get targets').item.json.Domain }}` and `{{ $json.diff.join(', ') }}`
     - Assignee: choose a user
     - Project: choose a project
   - Connect: Filter (has new keywords) → Create a task

20) **Add Slack: Send a message**
   - Node: *Slack*
   - Credentials: Slack OAuth2
   - Choose channel/recipient in node UI
   - Text: `New ranked keywords with SV > 1000 for target {{Domain}}: {{ diff.join(', ') }}`
   - Connect: Create a task → Send a message

21) **Close the loop**
   - Connect: Send a message → Loop over targets (to process next target)

22) **Recommended fix (to ensure loop continues even with no new keywords)**
   - Add an additional connection from **Filter (has new keywords)** “false/no-match” path back to **Loop over targets**, or add a no-op node after Find new keywords that always routes back to the loop.  
   (Without this, targets after a “no new keywords” target may not be processed.)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DataForSEO API access requires API login/password | https://app.dataforseo.com/api-access |
| Example spreadsheet structure for Keywords sheet | https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=1681593026#gid=1681593026 |
| Example spreadsheet structure for Targets sheet | https://docs.google.com/spreadsheets/d/1uCvVHKk8rWeQ_FKpZkBfrCB26Q6UebwwzsOdU7EmUYU/edit?gid=0#gid=0 |
| Workflow intent summary: weekly run, read sheets, fetch DataForSEO, diff, write, Asana task, Slack summary | (From the workflow sticky note “How it works” / “Setup steps”) |