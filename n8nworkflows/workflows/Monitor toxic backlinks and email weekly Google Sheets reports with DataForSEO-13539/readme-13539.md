Monitor toxic backlinks and email weekly Google Sheets reports with DataForSEO

https://n8nworkflows.xyz/workflows/monitor-toxic-backlinks-and-email-weekly-google-sheets-reports-with-dataforseo-13539


# Monitor toxic backlinks and email weekly Google Sheets reports with DataForSEO

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs **weekly** to monitor potentially toxic/spam backlinks for a target domain using the **DataForSEO Backlinks API**, compiles the results into a newly created **Google Sheets** spreadsheet, and sends a **Gmail** email containing a link to the report.

**Typical use cases:**
- SEO monitoring for harmful backlinks (negative SEO, spam campaigns).
- Weekly backlink hygiene reporting for agencies or internal SEO teams.
- Building an auditable history of suspicious backlinks (one spreadsheet per week).

### 1.1 Scheduling & pagination setup
Runs weekly, initializes an accumulator array, and paginates through DataForSEO results in batches of 1000.

### 1.2 Backlink retrieval & spam filtering (DataForSEO)
Calls DataForSEO with filters (spam score threshold + last 7 days), merges paginated items into an array, and decides whether to continue paging.

### 1.3 Conditional report generation
Only proceeds if DataForSEO reports at least one spam backlink.

### 1.4 Google Sheets report creation & population
Creates a new spreadsheet, creates/ensures header columns, splits items to rows, maps fields, and appends rows.

### 1.5 Email delivery (Gmail)
Sends a weekly email summary and the spreadsheet link.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & accumulator initialization
**Overview:** Triggers every week, then initializes an `items` array used to accumulate backlinks across pages.  
**Nodes involved:** `Schedule Trigger`, `Initialize "items" field`

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger (cron-like entry point).
- **Config (interpreted):** Weekly schedule: **Mondays at 09:00** (triggerAtDay `[1]`, triggerAtHour `9`).
- **Outputs to:** `Initialize "items" field`
- **Edge cases / failures:**
  - Timezone depends on n8n instance settings; “09:00” may not match user’s local time if instance TZ differs.
  - If executions overlap (long API calls), consider concurrency settings.

#### Node: Initialize "items" field
- **Type / role:** Set node; initializes accumulator.
- **Config:** Sets `items` to an empty array: `[]`.
- **Outputs to:** `Set "items" field`
- **Edge cases:**
  - If later nodes assume `items` exists, this prevents “undefined” merge errors.

---

### Block 2 — Pagination loop & DataForSEO retrieval
**Overview:** Iteratively fetches backlinks from DataForSEO (1000 per runIndex page), filters to “spammy” backlinks (score > 50 and first seen within last 7 days), and merges results into `items`.  
**Nodes involved:** `Set "items" field`, `Get spam backlinks`, `Merge "items" with DFS response`, `Has more pages?`

#### Node: Set "items" field
- **Type / role:** Set node; passes `items` forward each loop iteration.
- **Config:** `items = {{$json.items}}` (effectively a no-op copy).
- **Inputs from:** `Initialize "items" field` (first pass) and `Has more pages?` (loop back).
- **Outputs to:** `Get spam backlinks`
- **Edge cases:**
  - If an upstream node changes the payload structure unexpectedly, `items` may not remain an array.

#### Node: Get spam backlinks
- **Type / role:** DataForSEO Backlinks API node (`n8n-nodes-dataforseo.dataForSeoBacklinksApi`).
- **Operation:** `get-backlinks`
- **Key configuration:**
  - **target:** `dataforseo.com` (hard-coded in this workflow; user should replace with their domain)
  - **limit:** `1000`
  - **offset:** `{{$runIndex * 1000}}` (pagination)
  - **include_indirect_links:** `false`
  - **filters (DataForSEO syntax):**  
    - `backlink_spam_score > 50`  
    - `first_seen > {{$now.minus(7, "days").format("yyyy-MM-dd")}}`
- **Inputs from:** `Set "items" field`
- **Outputs to:** `Merge "items" with DFS response`
- **Credentials:** DataForSEO API (login/password).
- **Edge cases / failures:**
  - Auth failure if API credentials invalid.
  - API quota/rate limiting or billing limits.
  - Response shape assumptions: uses `tasks[0].result[0].items` and `total_count`. If DataForSEO changes schema or returns empty tasks/errors, downstream expressions can fail.
  - `first_seen` filter depends on DataForSEO expected date format; here it’s `yyyy-MM-dd`.

#### Node: Merge "items" with DFS response
- **Type / role:** Set node; merges accumulator with current page’s items.
- **Key expression:**
  - `items = [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items ]`
- **Inputs from:** `Get spam backlinks`
- **Outputs to:** `Has more pages?`
- **Edge cases:**
  - If `tasks[0].result[0].items` is missing (API error), spreading will throw.
  - If `items` is not an array (corrupted state), merge fails.

#### Node: Has more pages?
- **Type / role:** IF node; controls pagination loop.
- **Condition:**  
  - `{{$runIndex}} < {{ $('Get spam backlinks').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
- **Outputs:**
  - **True branch:** back to `Set "items" field` (continue paging)
  - **False branch:** to `Filter (has spam backlinks)` (stop paging and continue workflow)
- **Edge cases / logic pitfalls:**
  - `total_count / 1000 - 1` can be fractional. Example: `total_count=1` → `1/1000 - 1 = -0.999`; condition becomes false immediately (fine).  
  - For `total_count=1001`, expression is `1.001 - 1 = 0.001`; with `$runIndex` starting at 0, `0 < 0.001` is true, so it runs page 2 (offset 1000) once—correct, but relies on floating math.
  - If `total_count` is missing/null, the IF expression can error.

---

### Block 3 — Proceed only if there are spam backlinks
**Overview:** Stops report generation when no suspicious backlinks were found.  
**Nodes involved:** `Filter (has spam backlinks)`

#### Node: Filter (has spam backlinks)
- **Type / role:** Filter node; gates downstream work.
- **Condition:** `total_count > 0` using:  
  - Left: `{{ $('Get spam backlinks').item.json.tasks[0].result[0].total_count }}`
  - Right: `0`
- **Inputs from:** `Has more pages?` (false branch)
- **Outputs to:** `Create spreadsheet` (only if condition passes)
- **Edge cases:**
  - If DataForSEO returned an error but still produced a structure without `total_count`, the expression can fail.
  - This checks `total_count` from the most recent `Get spam backlinks` execution; it assumes `total_count` is stable across pages (it should be).

---

### Block 4 — Spreadsheet creation + header setup
**Overview:** Creates a new spreadsheet for this week’s report and ensures the header columns exist.  
**Nodes involved:** `Create spreadsheet`, `Prepare columns for Google Sheet`, `Append columns`

#### Node: Create spreadsheet
- **Type / role:** Google Sheets node; creates a new spreadsheet.
- **Operation/resource:** Spreadsheet resource; create spreadsheet with an initial sheet.
- **Title expression:**  
  - `Spam Backlinks to {{ $('Get spam backlinks').item.json.tasks[0].data.target }} - {{ $now.format("yyyy-MM-dd") }}`
- **Initial sheet:** Creates a sheet titled **“Spam Backlinks”**.
- **Inputs from:** `Filter (has spam backlinks)`
- **Outputs to:** `Prepare columns for Google Sheet`
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - OAuth token expired/revoked.
  - Google API permission issues (Drive scope, Sheets scope).
  - If user wants to create in a specific Drive folder, this workflow currently does not set a folder option (despite sticky note mentioning Drive/folder).

#### Node: Prepare columns for Google Sheet
- **Type / role:** Set node; defines an empty-object row with the intended columns.
- **Config:** Produces JSON with keys:
  - Date, Target, Backlink, Spam Score, Backlink Rank, Domain from, Domain from Rank, URL from, URL from Rank, Backlink Is Broken, Backlinks Is Dofollow
- **Inputs from:** `Create spreadsheet`
- **Outputs to:** `Append columns`
- **Edge cases:**
  - Column names must match later mapping keys exactly to avoid misalignment.

#### Node: Append columns
- **Type / role:** Google Sheets node; append or update to ensure headers/columns exist.
- **Operation:** `appendOrUpdate`
- **Document ID:** `{{ $('Create spreadsheet').item.json.spreadsheetId }}`
- **Sheet name (by ID):** `{{ $('Create spreadsheet').item.json.sheets[0].properties.sheetId }}`
- **Inputs from:** `Prepare columns for Google Sheet`
- **Outputs to:** `Set final "items" field`
- **Edge cases / failures:**
  - Using `sheetId` where a node expects a sheet “name” is supported in n8n via resource locator (`__rl`), but if the node/UI changes, this could break.
  - `appendOrUpdate` behavior depends on matching columns configuration; here schema is empty, mapping is “auto map input data”.

---

### Block 5 — Final item set, row mapping, and append rows
**Overview:** Builds the final accumulated backlink array, splits it into individual rows, maps each backlink to the sheet schema, appends rows, then aggregates to one item for the email step.  
**Nodes involved:** `Set final "items" field`, `Split out items`, `Prepare data for Google Sheet`, `Append row in sheet`, `Aggregate`

#### Node: Set final "items" field
- **Type / role:** Set node; constructs the final array of items to write.
- **Key expression:**  
  - `items = [...$('Set "items" field').item.json.items, ... $('Get spam backlinks').item.json.tasks[0].result[0].items]`
- **Inputs from:** `Append columns`
- **Outputs to:** `Split out items`
- **Important behavior note:**  
  - The pagination loop already merges pages in `Merge "items" with DFS response`, but this node merges again using `Set "items" field"` + the latest `Get spam backlinks` page. Depending on execution order and data carried forward, this can **duplicate the last page** or otherwise produce unexpected results if not aligned with the intended accumulator state.
- **Edge cases:**
  - Same schema risks as earlier merges (`items` must be an array; DataForSEO path must exist).

#### Node: Split out items
- **Type / role:** Split Out node; converts the `items` array into one item per backlink.
- **Field:** `items`
- **Inputs from:** `Set final "items" field`
- **Outputs to:** `Prepare data for Google Sheet`
- **Edge cases:**
  - If `items` is empty or missing, produces zero output (downstream nodes won’t run).
  - If `items` is not an array, node errors.

#### Node: Prepare data for Google Sheet
- **Type / role:** Set node (raw JSON mode); maps each backlink item into the sheet row format.
- **Key fields mapped (examples):**
  - `Date`: `{{$now.format('yyyy-MM-dd')}}`
  - `Target`: `{{ $('Get spam backlinks').item.json.tasks[0].data.target }}`
  - `Backlink`: `{{$json.url_to}}`
  - `Spam Score`: `{{$json.backlink_spam_score}}`
  - `Backlink Rank`: `{{$json.rank}}`
  - `Domain from`: `{{$json.domain_from}}`
  - `Domain from Rank`: `{{$json.domain_from_rank}}`
  - `URL from`: `{{$json.url_from}}`
  - `URL from Rank`: `{{$json.page_from_rank}}`
  - `Backlink Is Broken`: `{{$json.is_broken}}`
  - `Backlinks Is Dofollow`: `{{$json.dofollow}}`
- **Inputs from:** `Split out items`
- **Outputs to:** `Append row in sheet`
- **Edge cases:**
  - Some fields may be null/missing for certain backlinks; values will become empty strings or “null”-like strings depending on coercion.
  - Note the template includes a leading space in `"Backlink": " {{ $json.url_to }}"` which may create inconsistent values.

#### Node: Append row in sheet
- **Type / role:** Google Sheets node; appends each prepared row.
- **Operation:** `append`
- **Document ID:** `{{ $('Create spreadsheet').item.json.spreadsheetId }}`
- **Sheet (by ID):** `{{ $('Create spreadsheet').item.json.sheets[0].properties.sheetId }}`
- **Columns schema:** Explicit schema with the same column names as the header step; mapping mode auto-maps input fields.
- **Inputs from:** `Prepare data for Google Sheet`
- **Outputs to:** `Aggregate`
- **Edge cases / failures:**
  - Google API rate limits for large backlink counts.
  - If schema column names differ from prepared keys (spelling/case), data may not map.

#### Node: Aggregate
- **Type / role:** Aggregate node; reduces multiple row-append outputs into a single item (so the email is sent once).
- **Mode:** `aggregateAllItemData`
- **Inputs from:** `Append row in sheet`
- **Outputs to:** `Send a message`
- **Edge cases:**
  - For very large executions, aggregated payload can become large (memory considerations), though here it’s mainly used as a “join” barrier.

---

### Block 6 — Email report (Gmail)
**Overview:** Sends a weekly email summarizing the count and linking to the newly created Google Sheet.  
**Nodes involved:** `Send a message`

#### Node: Send a message
- **Type / role:** Gmail node; email delivery.
- **To:** `user@example.com` (placeholder; must be replaced)
- **Subject expression:**  
  - `Spam Backlinks to {{ target }} - {{ today }}`
- **HTML message expression:** Includes:
  - Target domain
  - Total count: `{{ total_count }}`
  - Link to spreadsheet: `{{ $('Create spreadsheet').item.json.spreadsheetUrl }}`
- **Inputs from:** `Aggregate`
- **Credentials:** Gmail OAuth2.
- **Edge cases / failures:**
  - Gmail OAuth token revoked/expired.
  - Sending limits (daily quotas) or restricted scopes.
  - If `Create spreadsheet` didn’t run (no spam backlinks), this node is not reached (by design).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Comment: DataForSEO setup context | — | — | ## Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. |
| Schedule Trigger | Schedule Trigger | Weekly entry point | — | Initialize "items" field | This workflow runs weekly and uses the DataForSEO Backlinks API to automatically retrieve suspicious backlinks (filtered by a spam score threshold) pointing to your domain, then it generates a toxic backlink report and emails the results to you for a quick review.; ## How it works; 1. Triggers automatically on a weekly schedule.; 2. Fetches backlinks for your domain or URL using the DataForSEO Backlinks API.; 3. Filters backlinks by spam score threshold (default: >50).; 4. Extracts suspicious backlinks and key metrics (source URL, spam score, DR, etc.).; 5. Builds a structured weekly report in Google Sheets.; 6. Emails you the report via Gmail.; ## Setup steps; 1. Create or select your DataForSEO connection (use [your API login and password](https://app.dataforseo.com/api-access)).; 2. Indicate your target domain.; 3. Connect Google Drive and choose a folder. ; 4. Connect Gmail and choose who gets the message. |
| Initialize "items" field | Set | Initialize accumulator array | Schedule Trigger | Set "items" field | (same as above) |
| Set "items" field | Set | Carry accumulator through loop | Initialize "items" field; Has more pages? (true) | Get spam backlinks | (same as above) |
| Get spam backlinks | DataForSEO Backlinks API | Fetch filtered backlinks page | Set "items" field | Merge "items" with DFS response | ## Get spam backlinks with DataForSEO; Create a DataForSEO connection, specify a Target Domain, and set up additional parameters if needed. |
| Merge "items" with DFS response | Set | Merge accumulator with current page items | Get spam backlinks | Has more pages? | (same as above) |
| Has more pages? | IF | Pagination control | Merge "items" with DFS response | Set "items" field (true); Filter (has spam backlinks) (false) | (same as above) |
| Filter (has spam backlinks) | Filter | Stop if total_count = 0 | Has more pages? (false) | Create spreadsheet | ## Create a Google Sheet with spam backlinks and send it via email; Create a Google Sheets connection.; Create a Gmail connection and set a receiver. |
| Create spreadsheet | Google Sheets | Create weekly report spreadsheet | Filter (has spam backlinks) | Prepare columns for Google Sheet | ## Create a Google Sheet with spam backlinks and send it via email; Create a Google Sheets connection.; Create a Gmail connection and set a receiver. |
| Prepare columns for Google Sheet | Set | Create header/column keys | Create spreadsheet | Append columns | (same as above) |
| Append columns | Google Sheets | Ensure header row/columns exist | Prepare columns for Google Sheet | Set final "items" field | (same as above) |
| Set final "items" field | Set | Build final list to write | Append columns | Split out items | (same as above) |
| Split out items | Split Out | One item per backlink | Set final "items" field | Prepare data for Google Sheet | (same as above) |
| Prepare data for Google Sheet | Set | Map backlink fields to sheet columns | Split out items | Append row in sheet | (same as above) |
| Append row in sheet | Google Sheets | Append each backlink row | Prepare data for Google Sheet | Aggregate | (same as above) |
| Aggregate | Aggregate | Collapse to single item before email | Append row in sheet | Send a message | (same as above) |
| Send a message | Gmail | Email report with sheet link | Aggregate | — | ## Create a Google Sheet with spam backlinks and send it via email; Create a Google Sheets connection.; Create a Gmail connection and set a receiver. |
| Sticky Note3 | Sticky Note | Comment: Sheets + Gmail setup | — | — | ## Create a Google Sheet with spam backlinks and send it via email; Create a Google Sheets connection.; Create a Gmail connection and set a receiver. |
| Sticky Note1 | Sticky Note | Comment: end-to-end description + setup link | — | — | This workflow runs weekly and uses the DataForSEO Backlinks API to automatically retrieve suspicious backlinks (filtered by a spam score threshold) pointing to your domain, then it generates a toxic backlink report and emails the results to you for a quick review.; ## How it works; 1. Triggers automatically on a weekly schedule.; 2. Fetches backlinks for your domain or URL using the DataForSEO Backlinks API.; 3. Filters backlinks by spam score threshold (default: >50).; 4. Extracts suspicious backlinks and key metrics (source URL, spam score, DR, etc.).; 5. Builds a structured weekly report in Google Sheets.; 6. Emails you the report via Gmail.; ## Setup steps; 1. Create or select your DataForSEO connection (use [your API login and password](https://app.dataforseo.com/api-access)).; 2. Indicate your target domain.; 3. Connect Google Drive and choose a folder. ; 4. Connect Gmail and choose who gets the message. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n (inactive by default while configuring credentials).

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure: Interval **Weeks**, set **Day = Monday**, **Hour = 9** (adjust timezone/locale as needed).
   - Connect to the next node.

3. **Add accumulator initialization**
   - Node: **Set**
   - Name: `Initialize "items" field`
   - Add field:
     - `items` (type: Array) = `[]`
   - Connect to the next node.

4. **Add loop carrier**
   - Node: **Set**
   - Name: `Set "items" field`
   - Set:
     - `items` (Array) = `{{$json.items}}`
   - Connect to DataForSEO node.

5. **Add DataForSEO Backlinks API call**
   - Node: **DataForSEO Backlinks API** (Backlinks endpoint)
   - Name: `Get spam backlinks`
   - Credentials: create **DataForSEO API** credential using API login/password from: https://app.dataforseo.com/api-access
   - Configure:
     - Operation: **Get Backlinks**
     - Target: your domain (example: `example.com`)
     - Limit: `1000`
     - Offset: `{{$runIndex * 1000}}`
     - Include indirect links: **false**
     - Filters (DataForSEO filter syntax):  
       - `backlink_spam_score > 50` AND `first_seen > {{ $now.minus(7, "days").format("yyyy-MM-dd") }}`
   - Connect to merge node.

6. **Merge current page into accumulator**
   - Node: **Set**
   - Name: `Merge "items" with DFS response`
   - Set:
     - `items` (Array) = `{{ [ ...$('Set "items" field').item.json.items, ...$json.tasks[0].result[0].items ] }}`
   - Connect to IF node.

7. **Add pagination IF**
   - Node: **IF**
   - Name: `Has more pages?`
   - Condition (Number):
     - Left: `{{$runIndex}}`
     - Operator: **less than**
     - Right: `{{ $('Get spam backlinks').item.json.tasks[0].result[0].total_count / 1000 - 1 }}`
   - Connect **true** output back to `Set "items" field`.
   - Connect **false** output forward to the filter gate.

8. **Add “has results” gate**
   - Node: **Filter**
   - Name: `Filter (has spam backlinks)`
   - Condition (Number):
     - Left: `{{ $('Get spam backlinks').item.json.tasks[0].result[0].total_count }}`
     - Operator: **greater than**
     - Right: `0`
   - Connect to Google Sheets creation.

9. **Create spreadsheet**
   - Node: **Google Sheets**
   - Name: `Create spreadsheet`
   - Credentials: **Google Sheets OAuth2** (authorize an account with Sheets/Drive access).
   - Resource: **Spreadsheet**
   - Operation: **Create**
   - Title:  
     `Spam Backlinks to {{ $('Get spam backlinks').item.json.tasks[0].data.target }} - {{ $now.format("yyyy-MM-dd") }}`
   - Add a sheet titled: `Spam Backlinks`
   - Connect to header preparation.

10. **Prepare header columns**
    - Node: **Set** (raw JSON)
    - Name: `Prepare columns for Google Sheet`
    - Output JSON containing the desired keys (empty values), matching exactly the column names you’ll write later:
      - Date, Target, Backlink, Spam Score, Backlink Rank, Domain from, Domain from Rank, URL from, URL from Rank, Backlink Is Broken, Backlinks Is Dofollow
    - Connect to sheet append/update.

11. **Append or update header row**
    - Node: **Google Sheets**
    - Name: `Append columns`
    - Operation: **Append or Update**
    - Document: `{{ $('Create spreadsheet').item.json.spreadsheetId }}`
    - Sheet (resource locator by ID): `{{ $('Create spreadsheet').item.json.sheets[0].properties.sheetId }}`
    - Connect to final items builder.

12. **Build final items list**
    - Node: **Set**
    - Name: `Set final "items" field`
    - Set:
      - `items` (Array) = `{{ [...$('Set "items" field').item.json.items, ... $('Get spam backlinks').item.json.tasks[0].result[0].items]}}`
    - Connect to split node.

13. **Split items array into individual items**
    - Node: **Split Out**
    - Name: `Split out items`
    - Field to split out: `items`
    - Connect to row mapping.

14. **Map each backlink item to sheet row format**
    - Node: **Set** (raw JSON)
    - Name: `Prepare data for Google Sheet`
    - Map fields from each backlink item (`$json`) into the column keys used above.
    - Connect to append row.

15. **Append rows to the sheet**
    - Node: **Google Sheets**
    - Name: `Append row in sheet`
    - Operation: **Append**
    - Document: `{{ $('Create spreadsheet').item.json.spreadsheetId }}`
    - Sheet (by ID): `{{ $('Create spreadsheet').item.json.sheets[0].properties.sheetId }}`
    - Define the schema columns matching your header keys (or rely on automapping, but schema is safer).
    - Connect to Aggregate.

16. **Aggregate to a single item**
    - Node: **Aggregate**
    - Name: `Aggregate`
    - Mode: **Aggregate All Item Data**
    - Connect to Gmail.

17. **Send email**
    - Node: **Gmail**
    - Name: `Send a message`
    - Credentials: Gmail OAuth2 (authorize sending account).
    - To: set your recipient address.
    - Subject: `Spam Backlinks to {{ target }} - {{ today }}`
    - Body: HTML including:
      - `total_count`
      - `spreadsheetUrl` from `Create spreadsheet`
    - Final node (no outputs).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| DataForSEO API access requires login/password credentials | https://app.dataforseo.com/api-access |
| Workflow intent and setup steps are described in the canvas note (weekly run, spam score filter, Google Sheets report, Gmail delivery) | (Embedded in workflow sticky note content) |
| Sticky note mentions “Connect Google Drive and choose a folder” but the current Google Sheets node configuration does not explicitly set a Drive folder | Consider adding a folder option in the spreadsheet creation step if needed |

