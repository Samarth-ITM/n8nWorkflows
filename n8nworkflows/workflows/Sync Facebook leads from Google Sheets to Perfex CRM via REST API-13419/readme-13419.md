Sync Facebook leads from Google Sheets to Perfex CRM via REST API

https://n8nworkflows.xyz/workflows/sync-facebook-leads-from-google-sheets-to-perfex-crm-via-rest-api-13419


# Sync Facebook leads from Google Sheets to Perfex CRM via REST API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync Facebook leads from Google Sheets to Perfex CRM via REST API

**Purpose:**  
Automatically sync newly collected Facebook Lead Ads rows (stored in Google Sheets) into Perfex CRM via its REST API, **preventing duplicates** by searching leads by email before creating new ones. It also **writes back** to Google Sheets to mark processed rows as `ADDED` and stores a **clickable CRM hyperlink**.

**Target use cases:**
- Sales/support teams importing Facebook leads into Perfex CRM
- Any lead intake process where Google Sheets is a staging queue and Perfex is the CRM
- Deduplication via email before creation

### Logical blocks (by dependencies)

**1.1 Scheduling & lead intake (Google Sheets filter)**
- Runs every minute, fetches only rows where `lead_status = CREATED`.

**1.2 Batch processing loop & throttling**
- Iterates leads one-by-one (SplitInBatches pattern) with a Wait buffer to reduce rate-limit risk.

**1.3 Deduplication check (Perfex search by email)**
- Calls Perfex “search lead by email” endpoint and branches depending on whether a lead exists or the API returns 404.

**1.4 Upsert outcome handling (create or mark existing)**
- If found: mark row `ADDED` and store CRM link with existing lead id  
- If not found (404): create lead, then mark row `ADDED` and store CRM link with new record id

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & lead intake (Google Sheets filter)

**Overview:**  
Triggers every minute and queries the Google Sheet for leads that have not been synced yet (status `CREATED`).

**Nodes involved:**
- Schedule trigger (every minute)
- Google Sheets: Get leads with status CREATED

#### Node: Schedule trigger (every minute)
- **Type / role:** `Schedule Trigger` — workflow entry point on a fixed interval.
- **Configuration choices:** Runs every **1 minute**.
- **Inputs/Outputs:** No inputs. Output to Google Sheets fetch node.
- **Edge cases / failures:**
  - High-frequency schedule can overlap if processing takes > 1 minute (depends on n8n concurrency settings).
  - Increased API usage (Google + Perfex) may hit quotas/rate limits.

#### Node: Google Sheets: Get leads with status CREATED
- **Type / role:** `Google Sheets` — reads rows from a sheet with a filter.
- **Configuration choices (interpreted):**
  - Targets `Sheet1` in a specified spreadsheet URL (placeholder in JSON).
  - Applies a filter: `lead_status` equals `CREATED`.
- **Key fields expected in each row (based on later expressions):**
  - `email`, `full_name`, `phone_number`, `company_name`, `platform`, `what_are_you_looking_for?`, `Note`
  - `row_number` (used later to update the exact row)
  - `lead_status`
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`).
- **Inputs/Outputs:**
  - Input from Schedule Trigger.
  - Output is a list of matching rows, passed to Split in batches.
- **Edge cases / failures:**
  - Missing/invalid OAuth credentials.
  - Spreadsheet URL or Sheet gid mismatch.
  - Filter column `lead_status` missing or named differently (returns 0 rows or fails).
  - If `row_number` is not returned by the node (depends on node config/version), downstream updates cannot match rows reliably.

---

### 2.2 Batch processing loop & throttling

**Overview:**  
Processes each lead row in a loop using `Split in batches`. A `Wait` node is used as a buffer between batches to reduce the chance of rate limiting.

**Nodes involved:**
- Split in batches
- Wait (rate limit buffer)

#### Node: Split in batches
- **Type / role:** `SplitInBatches` — iterates items in controlled chunks (commonly used for loop patterns).
- **Configuration choices:** Batch options left default (batch size not explicitly set in provided JSON).
- **Connections (important):**
  - Receives items from “Get leads…”.
  - **Loop behavior in this workflow:**  
    - Its **second output** (index 1) is connected to “Perfex: Search lead by email”.  
    - After each item is handled and written back, the workflow routes back into Split in batches (via Wait / Mark as ADDED (new)), pulling the next item.
- **Edge cases / failures:**
  - If batch size is too high, Perfex or Google Sheets may rate limit.
  - If no items are received, the loop does nothing (expected).

#### Node: Wait (rate limit buffer)
- **Type / role:** `Wait` — pauses the workflow before continuing.
- **Configuration choices:** No explicit duration shown (defaults depend on how the node is configured; in this JSON it’s effectively used as a “buffer step”).
- **Inputs/Outputs:**
  - Input from “Google Sheets: Mark as ADDED (existing)”.
  - Output goes back to “Split in batches” to continue processing.
- **Edge cases / failures:**
  - If configured to “wait for webhook” accidentally, it could block the workflow; verify it’s a time-based wait if the intent is throttling.
  - Adds latency; with high volume and 1-minute schedule it may backlog.

---

### 2.3 Deduplication check (Perfex search by email)

**Overview:**  
Calls Perfex CRM to determine whether the lead already exists (by email). The workflow branches based on successful response containing an `id` or an error response with status `404`.

**Nodes involved:**
- Perfex: Search lead by email
- IF: Lead exists
- IF: Lead not found (404)

#### Node: Perfex: Search lead by email
- **Type / role:** `HTTP Request` — calls Perfex REST API.
- **Configuration choices (interpreted):**
  - **Method:** GET (implicit by not specifying)
  - **URL expression:** `https://crm.example.com/api/leads/search/{{ $json.email }}`
  - Sends headers:
    - `accept: application/json`
    - `authtoken: your api key` (hardcoded placeholder)
  - **Error handling:** `onError: continueErrorOutput`  
    This is crucial: failures are emitted on the error output instead of stopping the workflow.
- **Key expressions/variables:**
  - `{{ $json.email }}` — must exist and be URL-safe.
- **Outputs / branching:**
  - **Main output (success)** → `IF: Lead exists`
  - **Error output** → `IF: Lead not found (404)`
- **Edge cases / failures:**
  - Missing/empty email: results in calling `/search/` with blank, likely 400/404.
  - Email needs URL encoding if it contains special chars; otherwise request may fail.
  - Auth token invalid/expired → 401/403; will go to the error branch but **will not be treated as “not found”** (404 check will fail), effectively dropping the item unless handled.
  - Perfex API may return 200 with empty body instead of 404 depending on implementation—this workflow assumes 404 for “not found”.

#### Node: IF: Lead exists
- **Type / role:** `IF` — checks if the Perfex response contains a lead identifier.
- **Condition:** string “exists” on `={{ $json.id }}`
- **True path:** “Google Sheets: Mark as ADDED (existing)”
- **False path:** not connected (no action)
- **Edge cases / failures:**
  - If Perfex returns an array or nested structure (e.g., `data[0].id`), `$json.id` won’t exist and the workflow will do nothing for that record.
  - If Perfex returns `id` as number but n8n treats it differently, “exists” still usually passes; but structure mismatch is the main risk.

#### Node: IF: Lead not found (404)
- **Type / role:** `IF` — confirms the error is specifically a 404 not-found.
- **Condition:** number equals on `={{ $json.error.status }}` equals `404`
- **True path:** “Perfex: Create lead”
- **False path:** not connected (no action)
- **Edge cases / failures:**
  - Some HTTP errors may not populate `$json.error.status` in the same way (depends on n8n version and node error structure).
  - If Perfex returns 200 with “not found” message instead of 404, this branch never triggers.

---

### 2.4 Upsert outcome handling (create or mark existing)

**Overview:**  
If the lead exists, update the Google Sheet row with `ADDED` and a hyperlink to the lead in Perfex. If it doesn’t exist (404), create the lead, then update the row similarly using the returned `record_id`.

**Nodes involved:**
- Perfex: Create lead
- Google Sheets: Mark as ADDED (existing)
- Google Sheets: Mark as ADDED (new)

#### Node: Perfex: Create lead
- **Type / role:** `HTTP Request` — creates a lead record in Perfex.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://crm.example.com/api/leads`
  - **Body type:** `multipart-form-data`
  - Headers:
    - `accept: application/json`
    - `authtoken: your api key`
  - **Body parameters mapped from sheet row:**
    - `name` ← `$json.full_name`
    - `email` ← `$json.email`
    - `company` ← `$json.company_name`
    - `phonenumber` ← `$json.phone_number.replace('p:', '')`
    - `tags` ← `$json['what_are_you_looking_for?']`
    - `source` ← `$json.platform === 'fb' ? 2 : 3`
    - `description` ← `$json.Note`
    - `assigned` fixed `"5"`
    - `status` fixed `"1"`
    - `custom_fields` is set to a JSON-like string:
      ```text
      {
        "note": true
      }
      ```
    - `custom_contact_date` is set to `=` (appears incomplete / placeholder)
    - `file` parameter exists but has no value
- **Inputs/Outputs:**
  - Input from “IF: Lead not found (404)” (true path).
  - Output to “Google Sheets: Mark as ADDED (new)”.
- **Edge cases / failures:**
  - If Perfex expects JSON (`application/json`) rather than multipart, request may fail; this depends on the Perfex REST module.
  - `custom_fields` formatting may need to match Perfex API spec (often a nested object or specific keys like `custom_fields[leads][FIELD_ID]`).
  - `custom_contact_date` value `=` is likely invalid and may cause API validation errors.
  - `phone_number.replace('p:', '')` fails if `phone_number` is null/non-string.
  - Hardcoded `assigned=5` and `status=1` may not exist in the target Perfex instance.

#### Node: Google Sheets: Mark as ADDED (existing)
- **Type / role:** `Google Sheets (Update)` — marks a row as processed and stores CRM link for an existing lead.
- **Configuration choices (interpreted):**
  - Operation: **Update**
  - Matching column: `row_number` (read-only row index from earlier read)
  - Sets:
    - `lead_status` = `ADDED`
    - `row_number` = `{{ $('Split in batches').item.json.row_number }}`
    - `crm_id` = a **HYPERLINK** formula pointing to the existing lead id:
      - `https://crm.example.com/admin/leads/index/{{ $json.id }}`
- **Inputs/Outputs:**
  - Input from IF: Lead exists (true).
  - Output to Wait (rate limit buffer) → Split in batches (loop continues).
- **Key expressions/variables:**
  - Uses `$('Split in batches').item.json.row_number` to ensure it updates the correct sheet row for the current batch item.
- **Edge cases / failures:**
  - If `row_number` is missing, update will not match and may fail or update the wrong row.
  - The formula string begins with `==HYPERLINK(...)` (double equals). In Google Sheets formulas typically start with a single `=`; double `==` may render as text depending on how n8n writes values.
  - If `$json.id` is missing due to Perfex response format, hyperlink will be broken/empty.

#### Node: Google Sheets: Mark as ADDED (new)
- **Type / role:** `Google Sheets (Update)` — marks row processed and stores CRM link for a newly created lead.
- **Configuration choices (interpreted):**
  - Operation: **Update**
  - Matching column: `row_number`
  - Sets:
    - `lead_status` = `ADDED`
    - `row_number` = `{{ $('Split in batches').item.json.row_number }}`
    - `crm_id` = hyperlink using `{{ $json.record_id }}`
- **Inputs/Outputs:**
  - Input from “Perfex: Create lead”.
  - Output back to “Split in batches” (continues loop).
- **Edge cases / failures:**
  - In the provided JSON, `documentId` and `sheetName.cachedResultUrl` appear empty placeholders—this node must be pointed to the same spreadsheet/sheet as the read node.
  - Same `==HYPERLINK` double-equals concern.
  - Assumes Perfex create response returns `record_id`. If it returns `id` instead, the link will be blank.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule trigger (every minute) | Schedule Trigger | Periodic entry point | — | Google Sheets: Get leads with status CREATED | ## Who’s it for<br>This workflow is for sales and support teams who collect leads from Facebook Lead Ads into Google Sheets and want to automatically sync those leads into Perfex CRM without creating duplicates.<br><br>## What it does<br>Every minute, the workflow reads new rows from a Google Sheet where `lead_status` is `CREATED`. For each lead, it searches Perfex CRM by email. If a lead already exists, it updates the Google Sheet row to `ADDED` and stores a clickable CRM link. If the lead is not found (404), it creates a new lead in Perfex CRM and then marks the Sheet row as `ADDED` with the new CRM link.<br><br>## How to set up<br>1. Connect your Google Sheets credentials.<br>2. Add your Perfex CRM API token using n8n credentials or environment variables (do not hardcode tokens in the HTTP node).<br>3. In the “Config (Edit these)” node, set your Perfex base URL, assignee ID, and Sheet details.<br>4. Ensure your Google Sheet has at least: `email`, `full_name`, `phone_number`, and `lead_status`.<br><br>## Requirements<br>- Google Sheets access<br>- Perfex CRM REST API token (Perfex “Rest API module”)<br><br>## How to customize<br>Map additional fields (tags, source, description, custom fields) in the “Perfex: Create lead” node and adjust the batch size or schedule interval for your volume. |
| Google Sheets: Get leads with status CREATED | Google Sheets | Read unprocessed leads from Sheet | Schedule trigger (every minute) | Split in batches | Expected columns + example values + lead_status logic. |
| Split in batches | Split In Batches | Iteration/loop over leads | Google Sheets: Get leads with status CREATED; Wait (rate limit buffer); Google Sheets: Mark as ADDED (new) | Perfex: Search lead by email |  |
| Perfex: Search lead by email | HTTP Request | Dedup search in Perfex by email | Split in batches | IF: Lead exists; IF: Lead not found (404) | How to generate API token, where to put it (credentials/env), what 404 means. |
| IF: Lead exists | IF | Branch: existing lead | Perfex: Search lead by email (success) | Google Sheets: Mark as ADDED (existing) |  |
| IF: Lead not found (404) | IF | Branch: not found | Perfex: Search lead by email (error) | Perfex: Create lead |  |
| Perfex: Create lead | HTTP Request | Create new lead in Perfex | IF: Lead not found (404) | Google Sheets: Mark as ADDED (new) |  |
| Google Sheets: Mark as ADDED (existing) | Google Sheets | Update sheet row for existing lead | IF: Lead exists | Wait (rate limit buffer) | Writes back ADDED + CRM hyperlink, prevents duplicates. |
| Wait (rate limit buffer) | Wait | Throttle/buffer and continue loop | Google Sheets: Mark as ADDED (existing) | Split in batches |  |
| Google Sheets: Mark as ADDED (new) | Google Sheets | Update sheet row for new lead | Perfex: Create lead | Split in batches | Writes back ADDED + CRM hyperlink, prevents duplicates. |
| Sticky Note | Sticky Note | Documentation/comment | — | — | ## Who’s it for<br>This workflow is for sales and support teams who collect leads from Facebook Lead Ads into Google Sheets and want to automatically sync those leads into Perfex CRM without creating duplicates.<br><br>## What it does<br>Every minute, the workflow reads new rows from a Google Sheet where `lead_status` is `CREATED`. For each lead, it searches Perfex CRM by email. If a lead already exists, it updates the Google Sheet row to `ADDED` and stores a clickable CRM link. If the lead is not found (404), it creates a new lead in Perfex CRM and then marks the Sheet row as `ADDED` with the new CRM link.<br><br>## How to set up<br>1. Connect your Google Sheets credentials.<br>2. Add your Perfex CRM API token using n8n credentials or environment variables (do not hardcode tokens in the HTTP node).<br>3. In the “Config (Edit these)” node, set your Perfex base URL, assignee ID, and Sheet details.<br>4. Ensure your Google Sheet has at least: `email`, `full_name`, `phone_number`, and `lead_status`.<br><br>## Requirements<br>- Google Sheets access<br>- Perfex CRM REST API token (Perfex “Rest API module”)<br><br>## How to customize<br>Map additional fields (tags, source, description, custom fields) in the “Perfex: Create lead” node and adjust the batch size or schedule interval for your volume. |
| Sticky Note1 | Sticky Note | Documentation/comment | — | — | Expected columns + example values + lead_status logic. |
| Sticky Note2 | Sticky Note | Documentation/comment | — | — | How to generate API token, where to put it (credentials/env), what 404 means. |
| Sticky Note3 | Sticky Note | Documentation/comment | — | — | Writes back ADDED + CRM hyperlink, prevents duplicates. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “Schedule trigger (every minute)”**
   - Node: **Schedule Trigger**
   - Set rule: **Interval → Minutes → every 1 minute**
   - This is the entry node.

2) **Create “Google Sheets: Get leads with status CREATED”**
   - Node: **Google Sheets**
   - Credentials: connect **Google Sheets OAuth2**
   - Resource/Operation: **Read / Get Many** (wording varies by n8n version)
   - Select your **Spreadsheet** (by URL) and **Sheet** (e.g., `Sheet1`)
   - Add a **Filter**:
     - Column: `lead_status`
     - Value: `CREATED`
   - Ensure the output includes `row_number` (or enable “Return All / Include Row Number” depending on node UI).

3) **Create “Split in batches”**
   - Node: **Split In Batches**
   - Set **Batch Size** (recommended: 1–20 depending on API limits; the provided JSON implies default).
   - Connect: `Google Sheets: Get leads...` → `Split in batches`

4) **Create “Perfex: Search lead by email”**
   - Node: **HTTP Request**
   - Method: **GET**
   - URL: `https://<your-perfex-domain>/api/leads/search/{{$json.email}}`
   - Headers:
     - `accept: application/json`
     - `authtoken: <YOUR_TOKEN>`
   - Error handling: set **On Error → Continue (Send to Error Output)** (n8n label varies)
   - Connect:
     - `Split in batches` → `Perfex: Search lead by email` (use the looping output that processes items)

   **Credential recommendation:**  
   Prefer **HTTP Request credentials**, environment variables, or an n8n “Credential” pattern rather than hardcoding tokens in the node. If you use an env var, reference it in the header value with an expression (e.g., `{{$env.PERFEX_TOKEN}}`).

5) **Create “IF: Lead exists”**
   - Node: **IF**
   - Condition: **String → Exists**
   - Value: `{{$json.id}}`
   - Connect:
     - `Perfex: Search lead by email` **success output** → `IF: Lead exists`

6) **Create “Google Sheets: Mark as ADDED (existing)”**
   - Node: **Google Sheets**
   - Credentials: same Google Sheets OAuth2
   - Operation: **Update**
   - Select same Spreadsheet + Sheet
   - Matching:
     - Match column: `row_number`
     - Set `row_number` value to: `{{$('Split in batches').item.json.row_number}}`
   - Set columns:
     - `lead_status` = `ADDED`
     - `crm_id` = `=HYPERLINK("https://<your-perfex-domain>/admin/leads/index/{{$json.id}}","{{$json.id}}")`
       - (Use a **single** leading `=` unless you have a specific reason for `==`.)
   - Connect:
     - `IF: Lead exists` **true** → `Google Sheets: Mark as ADDED (existing)`

7) **Create “Wait (rate limit buffer)”**
   - Node: **Wait**
   - Configure as a **time-based wait** (e.g., 200–1000 ms) if the intent is rate-limit buffering.
   - Connect:
     - `Google Sheets: Mark as ADDED (existing)` → `Wait`
     - `Wait` → `Split in batches` (to continue loop)

8) **Create “IF: Lead not found (404)”**
   - Node: **IF**
   - Condition: **Number → Equals**
   - Left value: `{{$json.error.status}}`
   - Right value: `404`
   - Connect:
     - `Perfex: Search lead by email` **error output** → `IF: Lead not found (404)`

9) **Create “Perfex: Create lead”**
   - Node: **HTTP Request**
   - Method: **POST**
   - URL: `https://<your-perfex-domain>/api/leads`
   - Body content type: **multipart/form-data** (only if required by your Perfex REST module)
   - Headers:
     - `accept: application/json`
     - `authtoken: <YOUR_TOKEN>`
   - Body fields mapping (minimum viable):
     - `name` = `{{$json.full_name}}`
     - `email` = `{{$json.email}}`
     - `phonenumber` = `{{$json.phone_number.replace('p:', '')}}`
     - `company` = `{{$json.company_name}}`
     - plus fixed values like `assigned`, `status` as needed by your CRM
   - Connect:
     - `IF: Lead not found (404)` **true** → `Perfex: Create lead`

10) **Create “Google Sheets: Mark as ADDED (new)”**
   - Node: **Google Sheets**
   - Operation: **Update**
   - Same Spreadsheet + Sheet as the read node
   - Matching:
     - `row_number` = `{{$('Split in batches').item.json.row_number}}`
   - Set:
     - `lead_status` = `ADDED`
     - `crm_id` = `=HYPERLINK("https://<your-perfex-domain>/admin/leads/index/{{$json.record_id}}","{{$json.record_id}}")`
       - Adjust `record_id` if your Perfex response returns a different field (commonly `id`).
   - Connect:
     - `Perfex: Create lead` → `Google Sheets: Mark as ADDED (new)`
     - `Google Sheets: Mark as ADDED (new)` → `Split in batches` (continue loop)

11) **(Optional but recommended) Add failure handling**
   - Add a branch for:
     - Search error != 404 (401/403/500) → log/alert and mark row as `ERROR`
     - Create lead failures → mark row as `ERROR` with message

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Expected columns + example values + lead_status logic.” | Sticky note near Google Sheets intake block |
| “How to generate API token, where to put it (credentials/env), what 404 means.” | Sticky note near Perfex API block |
| “Writes back ADDED + CRM hyperlink, prevents duplicates.” | Sticky note near Google Sheets update nodes |
| Do not hardcode Perfex API tokens in HTTP nodes; prefer credentials or environment variables. | Mentioned in the main sticky note (“How to set up”) |
| The sticky note references a “Config (Edit these)” node, but **no such node exists** in the provided workflow. Consider adding a Set node for base URL, token, IDs, and sheet IDs to centralize configuration. | Consistency check based on provided JSON |

