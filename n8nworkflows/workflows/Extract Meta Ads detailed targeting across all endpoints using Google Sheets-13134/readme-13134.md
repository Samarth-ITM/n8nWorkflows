Extract Meta Ads detailed targeting across all endpoints using Google Sheets

https://n8nworkflows.xyz/workflows/extract-meta-ads-detailed-targeting-across-all-endpoints-using-google-sheets-13134


# Extract Meta Ads detailed targeting across all endpoints using Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Meta Ads Detailed Targeting (Universal, Switch by Endpoint)  
**Purpose:** Read “detailed targeting” request rows from Google Sheets, route each row to the correct Meta Ads Targeting endpoint (Search, Suggestions, Browse, Validation), normalize the returned results, and append them to dedicated result tabs in the same spreadsheet.

**Target use cases**
- Build/maintain an internal targeting library (interests, behaviors, demographics, etc.)
- Programmatically validate or expand targeting lists for campaign setup
- Centralize multiple Graph API endpoints behind one Google Sheet-driven interface

### 1.1 Input Reception (manual or polling trigger)
- Two entry points: manual run and a Google Sheets polling trigger.
- Both feed into a single “Read Input” node that reads the request sheet.

### 1.2 Row Validation + Routing
- Filters out incomplete request rows.
- Switch routes each row to one of four branches based on the `endpoint` field.

### 1.3 API Execution per Endpoint
- Calls Meta Graph API (v23.0) endpoint corresponding to the branch:
  - `targetingsearch`
  - `targetingsuggestions`
  - `targetingbrowse`
  - `targetingvalidation`

### 1.4 Context Merge → Array Split → Output Formatting
- Merges original request context with API response (combine-by-position).
- Splits the API `data` array into one item per targeting result.
- Flattens fields into a tabular schema suitable for Google Sheets.

### 1.5 Persist Results to Google Sheets
- Appends results into one of four result sheets:
  - `search_results`
  - `suggestions_results`
  - `browse_results`
  - `validation_results`

---

## 2. Block-by-Block Analysis

### Block A — Input reception (Manual + Google Sheets Trigger → read request sheet)

**Overview:** Starts the workflow either manually or on a schedule (polling Google Sheets), then reads all request rows from the `targeting_requests` tab.

**Nodes involved**
- Manual Trigger
- Google Sheets Trigger
- Read Input (Google Sheets)

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual start for testing or ad-hoc runs.
- **Configuration choices:** Default (no parameters).
- **Input/Output:**
  - Input: none
  - Output → `Read Input (Google Sheets)`
- **Edge cases / failures:** none (except workflow not active / user permissions).
- **Version notes:** typeVersion 1 (standard).

#### Node: Google Sheets Trigger
- **Type / role:** `n8n-nodes-base.googleSheetsTrigger` — polls a Google Sheet for changes.
- **Configuration choices:**
  - Polling: `everyHour`
  - Document: “Meta Ads | Detailed targeting”
  - Sheet/tab: `targeting_requests` (gid=0)
- **Credentials:** `Google Sheets Trigger account` (OAuth2)
- **Input/Output:**
  - Output → `Read Input (Google Sheets)`
- **Edge cases / failures:**
  - OAuth token expiration / revoked access
  - Polling may miss rapid successive updates depending on trigger internals
  - Trigger emits “new row” events, but the workflow *still reads the whole sheet* afterward (see next node), which can reprocess old rows if not otherwise controlled
- **Version notes:** typeVersion 1.

#### Node: Read Input (Google Sheets)
- **Type / role:** `n8n-nodes-base.googleSheets` — reads request data from Google Sheets.
- **Configuration choices (interpreted):**
  - Operation: read from spreadsheet “Meta Ads | Detailed targeting”
  - Sheet: `targeting_requests`
  - Returns each row as one item (used as one API request)
- **Credentials:** `KH | Google Sheets` (OAuth2)
- **Input/Output:**
  - Input: from Manual Trigger or Google Sheets Trigger
  - Output → `Valid rows (ad_account_id + endpoint)`
- **Edge cases / failures:**
  - Auth errors / missing permissions
  - Incorrect sheet name/gid selection
  - Column naming mismatches (expressions later expect specific column names, e.g. `ad_account_id`, `endpoint`, `q (keyword)`)
- **Version notes:** typeVersion 4.6 (Google Sheets node v4+ UI uses schema/mapping options; behavior can differ from older versions).

**Sticky note coverage**
- “Sticky Note - Get input & route” applies conceptually to: Manual Trigger, Google Sheets Trigger, Read Input, Valid rows, Switch by endpoint.

---

### Block B — Validate request rows and route by endpoint

**Overview:** Filters incomplete rows and routes each row to the appropriate API branch based on `endpoint`.

**Nodes involved**
- Valid rows (ad_account_id + endpoint)
- Switch by endpoint

#### Node: Valid rows (ad_account_id + endpoint)
- **Type / role:** `n8n-nodes-base.filter` — keeps only rows with required inputs.
- **Configuration choices:**
  - Condition 1: `$json.ad_account_id` is not empty
  - Condition 2: `$json.endpoint` is not empty
  - Combinator: AND
- **Input/Output:**
  - Input: from `Read Input (Google Sheets)`
  - Output → `Switch by endpoint`
- **Edge cases / failures:**
  - If column headers differ (e.g., `ad account id`), expressions evaluate to empty and rows get dropped.
  - “Strict” type validation: if a field is missing vs empty, results can differ depending on n8n version and sheet parsing.
- **Version notes:** typeVersion 2.2.

#### Node: Switch by endpoint
- **Type / role:** `n8n-nodes-base.switch` — routes items into one of four named outputs.
- **Configuration choices:**
  - Rule “Search”: `$json.endpoint === "search"`
  - Rule “Suggestions”: `$json.endpoint === "suggestions"`
  - Rule “Browse”: `$json.endpoint === "browse"`
  - Rule “Validation”: `$json.endpoint === "validation"`
  - Each rule renames output accordingly (outputKey).
- **Input/Output (important multi-wiring):**
  - Input: from `Valid rows…`
  - Output 0 (“Search”) → `API Search` **and** `Merge Search` (Merge input 2)
  - Output 1 (“Suggestions”) → `API Suggestions` **and** `Merge Suggestions` (Merge input 2)
  - Output 2 (“Browse”) → `API Browse` **and** `Merge Browse` (Merge input 2)
  - Output 3 (“Validation”) → `API Validation` **and** `Merge Validation` (Merge input 2)
- **Edge cases / failures:**
  - Any `endpoint` outside these four values is silently not routed (no default route configured), so the row is effectively dropped.
  - Case sensitivity: “Search” vs “search” will not match.
- **Version notes:** typeVersion 3.3.

---

### Block C — Branch: Search (targetingsearch → normalize → save)

**Overview:** Runs Meta “Targeting Search” using a keyword query, then stores each targeting result row in `search_results`.

**Nodes involved**
- API Search
- Merge Search
- Split Search
- Format Search
- Save search_results

#### Node: API Search
- **Type / role:** `n8n-nodes-base.facebookGraphApi` — calls Meta Graph API.
- **Configuration choices:**
  - Graph API version: v23.0
  - Node: `act_{{$json.ad_account_id}}`
  - Edge: `targetingsearch`
  - Query parameters:
    - `q` = `{{$json["q (keyword)"]}}`
    - `limit` = `{{$json.limit || 1000}}`
    - `limit_type` = `{{$json.limit_type || ''}}`
    - `locale` = `{{$json.locale || ''}}`
- **Credentials:** `KH | Facebook Graph`
- **Input/Output:**
  - Input: from Switch “Search”
  - Output → `Merge Search` (Merge input 1)
- **Edge cases / failures:**
  - Missing/invalid `ad_account_id` → Graph error
  - Wrong column name for query: expects **`q (keyword)`** exactly; if the sheet uses `q`, this will send empty `q` and return broad/empty results.
  - Meta permission/scopes issues (ads_management / business_management depending on account setup)
  - API rate limiting, transient 5xx
- **Version notes:** typeVersion 1 of the node; depends on n8n’s Facebook Graph node availability.

#### Node: Merge Search
- **Type / role:** `n8n-nodes-base.merge` — combines API response with original request row.
- **Configuration choices:**
  - Mode: Combine
  - Combine by: position
- **Connections (critical):**
  - Input 1: API Search response
  - Input 2: Original row from Switch “Search” (wired directly to Merge)
  - Output → Split Search
- **Why this matters:** The API response alone lacks request context (e.g., which query/account produced it). Merge preserves that.
- **Edge cases / failures:**
  - Combine-by-position assumes both streams have same item ordering and counts. If one side emits fewer/more items (e.g., API error producing no item), merge can misalign or produce empty merges.
- **Version notes:** typeVersion 3.2.

#### Node: Split Search
- **Type / role:** `n8n-nodes-base.splitOut` — expands `data[]` into one item per element.
- **Configuration choices:**
  - Field to split: `data`
  - Include: all other fields (keeps merged context alongside each data element)
- **Input/Output:** Merge Search → Split Search → Format Search
- **Edge cases / failures:**
  - If API returns no `data` field or `data` is not an array, node may output nothing or error (depending on n8n version/settings).
- **Version notes:** typeVersion 1.

#### Node: Format Search
- **Type / role:** `n8n-nodes-base.set` — maps to flat columns for Sheets.
- **Key mappings/expressions:**
  - `endpoint` = `"search"`
  - `ad_account_id` = `{{$json.ad_account_id}}`
  - `query` = `{{$json["q (keyword)"]}}`
  - `limit_type` = `{{$json.data.type}}` *(note: this is “type” from returned targeting item, not request limit_type)*
  - `targeting_id` = `{{$json.data.id}}`
  - `targeting_name` = `{{$json.data.name || ''}}`
  - `audience_size_lower_bound` / `upper_bound` from `$json.data.*`
  - `path` = join array with ` > ` if array
  - `description` = `{{$json.data.description || ''}}`
- **Input/Output:** Split Search → Format Search → Save search_results
- **Edge cases / failures:**
  - If `data` element lacks fields (id/name), expressions may yield undefined.
  - `limit_type` naming confusion: you may want to store request `limit_type` separately to avoid overwriting meaning.
- **Version notes:** typeVersion 3.4.

#### Node: Save search_results
- **Type / role:** `n8n-nodes-base.googleSheets` — appends formatted rows to result sheet.
- **Configuration choices:**
  - Operation: Append
  - Sheet: `search_results`
  - Mapping: Auto-map input data to columns (schema defined)
  - Columns include: `targeting_id`, `query`, `limit_type`, `targeting_name`, audience bounds, `path`, `description`, `endpoint`, `ad_account_id`, `type`, `valid`
- **Credentials:** `KH | Google Sheets`
- **Input/Output:** Format Search → Save search_results (end of branch)
- **Edge cases / failures:**
  - Sheet/tab missing or renamed
  - Schema mismatch: if columns absent, Google Sheets node may fail or append partial data depending on node behavior
  - High-volume writes can hit Google API quotas
- **Version notes:** typeVersion 4.6.

---

### Block D — Branch: Suggestions (targetingsuggestions → normalize → save)

**Overview:** Calls “Targeting Suggestions” based on a provided `targeting_list`, expands results, and appends to `suggestions_results`.

**Nodes involved**
- API Suggestions
- Merge Suggestions
- Split Suggestions
- Format Suggestions
- Save suggestions_results

#### Node: API Suggestions
- **Type / role:** Facebook Graph API call.
- **Configuration choices:**
  - Node: `act_{{$json.ad_account_id}}`
  - Edge: `targetingsuggestions`
  - Params:
    - `targeting_list` = `{{$json.targeting_list}}`
    - `limit` = `{{$json.limit || 1000}}`
    - `limit_type` = `{{$json.limit_type}}`
    - `locale` = `{{$json.locale || ''}}`
- **Edge cases / failures:**
  - `targeting_list` must be a JSON array string/structure acceptable by Meta (see notes in sticky).
  - If sheet stores it as plain text, you may need to ensure it’s valid JSON (double quotes, no smart quotes).
- **Other details:** Graph API v23.0, credential `KH | Facebook Graph`.

#### Node: Merge Suggestions
- Same merge pattern as Search (combine by position).
- Inputs: API response + original request row from Switch “Suggestions”.

#### Node: Split Suggestions
- Splits `data` array; includes all other fields.

#### Node: Format Suggestions
- **Mappings:**
  - `endpoint` = `"suggestions"`
  - `ad_account_id` from row
  - `query` = `{{$json.targeting_list || ''}}` (stores the input list as query/context)
  - `limit_type` = `{{$json.data.type}}` *(again: returned item type)*
  - `targeting_id` = `{{$json.data.id}}`
  - `targeting_name` = `{{$json.data.name}}`
  - audience bounds, `path` join, `description`
  - `type` = `{{$json.data.type}}`
- **Edge cases:**
  - `path` expression doesn’t default to empty string if missing (returns undefined), may be fine but could create inconsistent sheet values.

#### Node: Save suggestions_results
- Appends to `suggestions_results` with auto-mapping schema.
- Edge cases: same as other sheets.

---

### Block E — Branch: Browse (targetingbrowse → normalize → save)

**Overview:** Calls “Targeting Browse” (category browsing) and stores results to `browse_results`.

**Nodes involved**
- API Browse
- Merge Browse
- Split Browse
- Format Browse
- Save browse_results

#### Node: API Browse
- **Configuration choices:**
  - Edge: `targetingbrowse`
  - Params: `limit_type`, `locale`
  - Node: `act_{{$json.ad_account_id}}`
- **Edge cases:**
  - Depending on Meta behavior, browse may require additional params to navigate categories (not provided here), so results may be generic or limited.

#### Node: Merge Browse / Split Browse
- Same pattern: combine-by-position, then split `data`.

#### Node: Format Browse
- **Mappings:**
  - `endpoint` = `"browse"`
  - `ad_account_id`
  - `limit_type` = `{{$json.limit_type || ''}}` *(here it uses the request field, unlike Search/Suggestions)*
  - `targeting_name` = `{{$json.data.raw_name}}` (note: browse uses `raw_name` instead of `name`)
  - Others: `id`, audience bounds, `path`, `description`
- **Edge cases:**
  - If `raw_name` missing, `targeting_name` becomes undefined.
  - The result schema includes a `type` column in the Save node, but Format Browse does **not** set `type` explicitly (it will be blank unless carried from upstream).

#### Node: Save browse_results
- Appends to `browse_results` with auto-mapping schema.

---

### Block F — Branch: Validation (targetingvalidation → normalize → save)

**Overview:** Validates a provided `targeting_list` and writes validation results to `validation_results`.

**Nodes involved**
- API Validation
- Merge Validation
- Split Validation
- Format Validation
- Save validation_results

#### Node: API Validation
- **Configuration choices:**
  - Edge: `targetingvalidation`
  - Params:
    - `targeting_list` = `{{$json.targeting_list}}`
    - `locale` = `{{$json.locale || ''}}`
  - Node: `act_{{$json.ad_account_id}}`
- **Edge cases:**
  - `targeting_list` must be valid JSON array; common failure point.
  - Validation endpoint may return different structures depending on item types.

#### Node: Merge Validation / Split Validation
- Same merge/split pattern.

#### Node: Format Validation
- **Mappings:**
  - `endpoint` = `"validation"`
  - `ad_account_id`
  - `limit_type` = `{{$json.limit_type}}` (request field; may be undefined)
  - `targeting_id` / `targeting_name` from `$json.data`
  - audience bounds, `path`
  - `description` = `{{$json.description || ''}}` *(note: this references `$json.description` not `$json.data.description`; likely a bug/inconsistency)*
  - `valid` = `{{$json.data.valid}}`
- **Edge cases:**
  - Likely typo: `description` should probably be `{{$json.data.description || ''}}`.
  - `valid` is set as **string** though may be boolean; fine for Sheets but inconsistent typing.

#### Node: Save validation_results
- Appends to `validation_results`.
- **Schema:** empty in node configuration (auto-map without predefined schema). This can work but increases risk of misaligned columns if sheet headers differ.
- Edge cases: missing headers / append shape mismatch.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual entry point | — | Read Input (Google Sheets) | ## Get input & route by endpoint; Manual runs **Read Input** (whole sheet). Trigger runs when a new row is added to `targeting_requests`. |
| Google Sheets Trigger | Google Sheets Trigger | Scheduled/polling entry point | — | Read Input (Google Sheets) | ## Get input & route by endpoint; Manual runs **Read Input** (whole sheet). Trigger runs when a new row is added to `targeting_requests`. |
| Read Input (Google Sheets) | Google Sheets | Read request rows from `targeting_requests` | Manual Trigger; Google Sheets Trigger | Valid rows (ad_account_id + endpoint) | **Read Input (Google Sheets)** Reads sheet `targeting_requests`. Each row = one API request. |
| Valid rows (ad_account_id + endpoint) | Filter | Drop incomplete rows | Read Input (Google Sheets) | Switch by endpoint | **Valid rows** Keeps only rows where `ad_account_id` and `endpoint` are not empty. |
| Switch by endpoint | Switch | Route to API branch by `endpoint` | Valid rows (ad_account_id + endpoint) | API Search, Merge Search; API Suggestions, Merge Suggestions; API Browse, Merge Browse; API Validation, Merge Validation | **Switch by endpoint** Routes each row to the branch matching `endpoint`: search, suggestions, browse, or validation. |
| API Search | Facebook Graph API | Call `targetingsearch` | Switch by endpoint (Search output) | Merge Search | ## Call API, merge, split, format & save (branch pattern description) |
| Merge Search | Merge | Combine request context + API response | API Search; Switch by endpoint (Search output) | Split Search | ## Call API, merge, split, format & save (branch pattern description) |
| Split Search | Split Out | Expand `data[]` to items | Merge Search | Format Search | ## Call API, merge, split, format & save (branch pattern description) |
| Format Search | Set | Flatten fields for Sheets | Split Search | Save search_results | ## Call API, merge, split, format & save (branch pattern description) |
| Save search_results | Google Sheets | Append to `search_results` | Format Search | — | ## Call API, merge, split, format & save (branch pattern description) |
| API Suggestions | Facebook Graph API | Call `targetingsuggestions` | Switch by endpoint (Suggestions output) | Merge Suggestions | ## Call API, merge, split, format & save (branch pattern description) |
| Merge Suggestions | Merge | Combine request context + API response | API Suggestions; Switch by endpoint (Suggestions output) | Split Suggestions | ## Call API, merge, split, format & save (branch pattern description) |
| Split Suggestions | Split Out | Expand `data[]` | Merge Suggestions | Format Suggestions | ## Call API, merge, split, format & save (branch pattern description) |
| Format Suggestions | Set | Flatten fields for Sheets | Split Suggestions | Save suggestions_results | ## Call API, merge, split, format & save (branch pattern description) |
| Save suggestions_results | Google Sheets | Append to `suggestions_results` | Format Suggestions | — | ## Call API, merge, split, format & save (branch pattern description) |
| API Browse | Facebook Graph API | Call `targetingbrowse` | Switch by endpoint (Browse output) | Merge Browse | ## Call API, merge, split, format & save (branch pattern description) |
| Merge Browse | Merge | Combine request context + API response | API Browse; Switch by endpoint (Browse output) | Split Browse | ## Call API, merge, split, format & save (branch pattern description) |
| Split Browse | Split Out | Expand `data[]` | Merge Browse | Format Browse | ## Call API, merge, split, format & save (branch pattern description) |
| Format Browse | Set | Flatten fields for Sheets | Split Browse | Save browse_results | ## Call API, merge, split, format & save (branch pattern description) |
| Save browse_results | Google Sheets | Append to `browse_results` | Format Browse | — | ## Call API, merge, split, format & save (branch pattern description) |
| API Validation | Facebook Graph API | Call `targetingvalidation` | Switch by endpoint (Validation output) | Merge Validation | ## Call API, merge, split, format & save (branch pattern description) |
| Merge Validation | Merge | Combine request context + API response | API Validation; Switch by endpoint (Validation output) | Split Validation | ## Call API, merge, split, format & save (branch pattern description) |
| Split Validation | Split Out | Expand `data[]` | Merge Validation | Format Validation | ## Call API, merge, split, format & save (branch pattern description) |
| Format Validation | Set | Flatten fields for Sheets | Split Validation | Save validation_results | ## Call API, merge, split, format & save (branch pattern description) |
| Save validation_results | Google Sheets | Append to `validation_results` | Format Validation | — | ## Call API, merge, split, format & save (branch pattern description) |
| Sticky Note - Overview | Sticky Note | Documentation / instructions | — | — | # Meta Ads Detailed Targeting (Universal)… [Connect on LinkedIn](https://www.linkedin.com/in/kirill-khatkevich/) |
| Sticky Note - Get input & route | Sticky Note | Documentation / instructions | — | — | (Contains block description for input + routing) |
| Sticky Note - API & save | Sticky Note | Documentation / instructions | — | — | (Contains branch pattern description) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Meta Ads Detailed Targeting (Universal, Switch by Endpoint)**

2. **Create Google Sheets file structure**
   - Spreadsheet: (any name; in the original: “Meta Ads | Detailed targeting”)
   - Create tabs:
     1) `targeting_requests`
     2) `search_results`
     3) `suggestions_results`
     4) `browse_results`
     5) `validation_results`

3. **Prepare `targeting_requests` columns**
   - Minimum required columns:
     - `endpoint` (values: `search`, `suggestions`, `browse`, `validation`)
     - `ad_account_id` (Meta Ads account numeric ID, without `act_`)
   - Additional columns used by endpoints:
     - For **search**: `q (keyword)`, optional: `limit`, `limit_type`, `locale`
     - For **suggestions**: `targeting_list`, optional: `limit`, `limit_type`, `locale`
     - For **browse**: optional: `limit_type`, `locale`
     - For **validation**: `targeting_list`, optional: `locale`
   - For **suggestions/validation**, set `targeting_list` to a JSON array (string) like:
     - `[{"type":"interests","id":"6003263791114"}]`

4. **Add entry nodes**
   1) Add **Manual Trigger**
   2) Add **Google Sheets Trigger**
      - Poll time: **Every hour**
      - Select the same spreadsheet + `targeting_requests` tab
      - Set OAuth2 credentials for Google Sheets Trigger

5. **Add “Read Input (Google Sheets)”**
   - Node type: **Google Sheets**
   - Operation: **Read/Get rows** (depending on n8n UI wording)
   - Document: select your spreadsheet
   - Sheet: `targeting_requests`
   - Connect:
     - Manual Trigger → Read Input
     - Google Sheets Trigger → Read Input
   - Set OAuth2 credentials for Google Sheets

6. **Add “Valid rows (ad_account_id + endpoint)”**
   - Node type: **Filter**
   - Conditions (AND):
     - `ad_account_id` is not empty
     - `endpoint` is not empty
   - Connect: Read Input → Valid rows

7. **Add “Switch by endpoint”**
   - Node type: **Switch**
   - Add 4 rules (string equals):
     - `{{$json.endpoint}}` equals `search`
     - `{{$json.endpoint}}` equals `suggestions`
     - `{{$json.endpoint}}` equals `browse`
     - `{{$json.endpoint}}` equals `validation`
   - Connect: Valid rows → Switch

8. **Build the Search branch**
   1) Add **Facebook Graph API** node “API Search”
      - Node: `act_{{$json.ad_account_id}}`
      - Edge: `targetingsearch`
      - Graph API version: v23.0
      - Query params: `q`, `limit`, `limit_type`, `locale` using the expressions listed in section 2
      - Set Facebook Graph credentials
   2) Add **Merge** node “Merge Search”
      - Mode: combine, by position
      - Connect:
        - Switch (Search output) → API Search
        - API Search → Merge Search (input 1)
        - Switch (Search output) → Merge Search (input 2)
   3) Add **Split Out** “Split Search”
      - Field to split: `data`
      - Include: all other fields
      - Connect: Merge Search → Split Search
   4) Add **Set** “Format Search”
      - Map fields as in section 2 (endpoint/search, ids, name, bounds, path, description)
      - Connect: Split Search → Format Search
   5) Add **Google Sheets** “Save search_results”
      - Operation: Append
      - Sheet: `search_results`
      - Enable auto-mapping (or manually map columns)
      - Connect: Format Search → Save search_results

9. **Build the Suggestions branch**
   - Repeat the same pattern with:
     - API edge: `targetingsuggestions`
     - Params include `targeting_list`
     - Save sheet: `suggestions_results`

10. **Build the Browse branch**
   - Repeat the pattern with:
     - API edge: `targetingbrowse`
     - Params: `limit_type`, `locale`
     - In formatting, use `data.raw_name` for the name
     - Save sheet: `browse_results`

11. **Build the Validation branch**
   - Repeat the pattern with:
     - API edge: `targetingvalidation`
     - Params include `targeting_list`
     - Format includes `valid` from `data.valid`
     - Save sheet: `validation_results`

12. **Credentials checklist**
   - **Google Sheets OAuth2** credential for read/write nodes (scopes allowing Sheets read/write).
   - **Google Sheets Trigger OAuth2** credential for the trigger node.
   - **Facebook Graph API** credential:
     - Access token with permissions for Ads account targeting endpoints (commonly requires ads-related scopes and correct Business access).
     - Ensure the token has access to the specified `ad_account_id`.

13. **Optional hardening (recommended if you want to avoid duplicates)**
   - Add a “processed” flag column in `targeting_requests` and update it after successful write.
   - Or modify the trigger path to process only the new row payload instead of reading the entire sheet.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Built by Kirill Khatkevich | Included in workflow notes |
| Connect on LinkedIn | https://www.linkedin.com/in/kirill-khatkevich/ |
| Suggestions/Validation `targeting_list` must be a JSON array (example: `[{"type":"interests","id":"6003263791114"}]`) | Sticky note “Overview” |
| Setup steps: set Document ID on all Google Sheets nodes; create request + result tabs; connect credentials | Sticky note “Overview” |
| Important behavior: Trigger runs on new row but workflow reads the whole `targeting_requests` sheet afterward (can reprocess older rows) | Derived from node wiring |

