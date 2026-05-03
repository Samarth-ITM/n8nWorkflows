Track domain expiry dates with Google Sheets and WHOIS API

https://n8nworkflows.xyz/workflows/track-domain-expiry-dates-with-google-sheets-and-whois-api-13211


# Track domain expiry dates with Google Sheets and WHOIS API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Track domain expiry dates with Google Sheets and WHOIS API  
**Purpose:** This workflow reads a list of domains from a Google Sheet, queries the WHOIS API (via RapidAPI) to retrieve DNS SOA data, derives an “expiry date” from the SOA `expire` value, and writes formatted expiry details (date, month, year) back into the same sheet, one domain at a time, with a 30-second pause between requests.

**Primary use cases**
- Maintain a domain portfolio tracker in Google Sheets with automatically refreshed “expiry” fields.
- Batch processing of many domains while rate-limiting requests to an external API.

### Logical blocks
1.1 **Manual start & configuration**  
- Manually triggers the workflow and sets Sheet ID / sheet name used by downstream nodes.

1.2 **Load domains from Google Sheets**  
- Reads all rows from the configured sheet to obtain the `Websites` column (domains).

1.3 **Per-domain loop + WHOIS API request**  
- Iterates over rows (domains), calls the WHOIS DNS records endpoint, and continues even if an error occurs.

1.4 **Expiry parsing, sheet update, and throttling**  
- Extracts `SOA[0].expire`, converts it into a formatted date string, updates the matching row in Google Sheets, waits 30 seconds, and proceeds to the next domain.

---

## 2. Block-by-Block Analysis

### 2.1 Manual start & configuration

**Overview:** Starts the workflow on demand and defines runtime constants (Sheet ID, sheet name) used to locate the Google Sheet to read/update.

**Nodes involved**
- Run Domain Expiry Check
- Set Sheet Configuration

#### Node: Run Domain Expiry Check
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration:** No parameters; user clicks “Execute workflow”.
- **Inputs/Outputs:**  
  - Input: none  
  - Output → **Set Sheet Configuration**
- **Version-specific:** TypeVersion 1.
- **Potential failures:** None (except instance-level execution issues).

#### Node: Set Sheet Configuration
- **Type / role:** Set (`n8n-nodes-base.set`) — defines configuration fields in the JSON for reuse.
- **Configuration choices (interpreted):**
  - Sets:
    - `Sheet ID` = `1JZVzYT__FJ4XRhberEa0NXQplL1jPavyt4xFV7mOEDo`
    - `sites sheet` = `" SiteData "` (note the leading/trailing spaces)
    - `Email Size` = `30` (not used elsewhere in this workflow)
    - An additional empty string field with empty name/value (likely accidental)
- **Key expressions/variables:** Downstream uses `$json['Sheet ID']` and `$json['sites sheet']`.
- **Inputs/Outputs:**  
  - Input ← Manual Trigger  
  - Output → **Read All Domains from Sheet**
- **Version-specific:** TypeVersion 3.4.
- **Edge cases / failures:**
  - **Sheet name whitespace:** `" SiteData "` includes spaces; if the actual sheet tab is `SiteData` (without spaces), Google Sheets node will fail to find it.
  - Empty field name may be harmless but can confuse maintainers.

---

### 2.2 Load domains from Google Sheets

**Overview:** Reads the configured Google Sheet tab to obtain all rows, including the `Websites` column that drives the domain loop.

**Nodes involved**
- Read All Domains from Sheet

#### Node: Read All Domains from Sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — fetch rows from a worksheet.
- **Configuration choices (interpreted):**
  - Uses **Document ID** from expression: `={{ $json['Sheet ID'] }}`
  - Uses **Sheet Name** from expression: `={{ $json['sites sheet'] }}`
  - Operation is not explicitly shown in JSON, but the node is configured to “read all rows” (default read behavior for this node when only doc + sheet are specified).
- **Credentials:** `Google Sheets - Krishna` (OAuth2).
- **Inputs/Outputs:**  
  - Input ← Set Sheet Configuration  
  - Output → **Loop Through Each Domain**
- **Version-specific:** TypeVersion 4.6.
- **Edge cases / failures:**
  - OAuth token expired/invalid, insufficient permissions, or wrong spreadsheet ID.
  - Sheet name mismatch (including whitespace) causing “sheet not found”.
  - Missing `Websites` column: downstream expressions expect `$json.Websites`.

---

### 2.3 Per-domain loop + WHOIS API request

**Overview:** Processes domains one-by-one using a batch loop; for each item it calls the RapidAPI WHOIS DNS endpoint to fetch SOA records.

**Nodes involved**
- Loop Through Each Domain
- Fetch DNS Records via WHOIS API

#### Node: Loop Through Each Domain
- **Type / role:** Split in Batches (`n8n-nodes-base.splitInBatches`) — iterates items in batches (used here as a per-item loop).
- **Configuration choices (interpreted):**
  - Batch size not explicitly set → defaults to **1** in many n8n versions (verify in UI). This matches “one domain at a time”.
- **Connections behavior:**
  - Output **(main, index 1)** → **Fetch DNS Records via WHOIS API** (the “continue” output)
  - Output **(main, index 0)** is unused (commonly the “done/noItemsLeft” output)
- **Inputs/Outputs:**  
  - Input ← Read All Domains from Sheet  
  - Loop-back input ← Pause Before Next Domain  
  - Output → Fetch DNS Records via WHOIS API
- **Version-specific:** TypeVersion 3.
- **Edge cases / failures:**
  - If batch size is not 1, multiple domains could be sent per iteration and break assumptions in later nodes (which use single-item access patterns).
  - If `Websites` field is empty or malformed, HTTP request may fail.

#### Node: Fetch DNS Records via WHOIS API
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — calls external WHOIS API endpoint.
- **Configuration choices (interpreted):**
  - **Method:** POST
  - **URL:** `https://whois-api6.p.rapidapi.com/dns/api/v1/getRecords`
  - **Body:** `query = {{$json.Websites}}`
  - **Headers:**
    - `x-rapidapi-host: whois-api6.p.rapidapi.com`
    - `x-rapidapi-key: <hardcoded key in workflow>` (should be moved to credentials/env)
  - **Error handling:** `onError: continueRegularOutput` (workflow continues even when request fails)
- **Inputs/Outputs:**  
  - Input ← Loop Through Each Domain  
  - Output → Convert Expiry Timestamp to Date
- **Version-specific:** TypeVersion 4.2.
- **Edge cases / failures:**
  - RapidAPI auth errors (invalid key, quota exceeded, host mismatch).
  - Network timeouts / non-2xx responses.
  - API returns unexpected schema (missing `result.records.SOA[0].expire`).
  - Because errors continue as “regular output”, downstream nodes may receive error-shaped JSON and then fail in code.

**Security note:** The RapidAPI key is embedded directly in the node parameters. This is risky for sharing/exporting and should be stored in credentials or environment variables.

---

### 2.4 Expiry parsing, sheet update, and throttling

**Overview:** Extracts an expiry-related value from the WHOIS response, formats it into `DD-MM-YYYY`, updates the matching row in Google Sheets, then waits 30 seconds before continuing to the next domain.

**Nodes involved**
- Convert Expiry Timestamp to Date
- Write Expiry Data to Sheet
- Pause Before Next Domain

#### Node: Convert Expiry Timestamp to Date
- **Type / role:** Code (`n8n-nodes-base.code`) — transforms WHOIS response into a formatted date.
- **Configuration choices (interpreted):**
  - JavaScript logic:
    - Reads: `expireSeconds = $input.first().json.result.records.SOA[0].expire`
    - Computes: `expireDate = new Date(Date.now() + expireSeconds * 1000)`
    - Formats as `DD-MM-YYYY`
    - Outputs: `{ expireDate: formatted }`
  - **Error handling:** `onError: continueRegularOutput`
- **Inputs/Outputs:**  
  - Input ← Fetch DNS Records via WHOIS API  
  - Output → Write Expiry Data to Sheet
- **Version-specific:** TypeVersion 2.
- **Edge cases / failures (important):**
  - If the HTTP node returned an error payload (or different structure), `result.records.SOA[0]` may be undefined → code throws.
  - **Semantic concern:** In DNS SOA records, the `expire` field is typically a **zone transfer expiry timer** (seconds), not the domain registration expiration date. This workflow is effectively computing “today + SOA expire seconds”, which may not represent registrar expiry.
  - If `expireSeconds` is not a number, date becomes invalid.

#### Node: Write Expiry Data to Sheet
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — updates rows with computed expiry fields.
- **Configuration choices (interpreted):**
  - **Operation:** Update
  - **Match on column:** `Websites` (uses `matchingColumns: ["Websites"]`)
  - **Sheet / Document:** pulled from **Set Sheet Configuration** via expressions:
    - Document ID: `={{ $('Set Sheet Configuration').item.json['Sheet ID'] }}`
    - Sheet Name: `={{ $('Set Sheet Configuration').item.json['sites sheet'] }}`
  - **Values written:**
    - `Websites` = `{{ $('Loop Through Each Domain').item.json.Websites }}`
    - `Domain Expiry` = `{{ $json.expireDate }}`
    - `Expiry Year` = `{{ $json.expireDate.split('-')[2] }}`
    - `Expiry Month` = month name lookup using `expireDate.split('-')[1]`
    - `Days Left` = `"="` (a literal equals sign; likely intended to be a formula but currently not)
  - **Error handling:** `onError: continueRegularOutput`
- **Credentials:** `Google Sheets - Krishna` (OAuth2).
- **Inputs/Outputs:**  
  - Input ← Convert Expiry Timestamp to Date  
  - Output → Pause Before Next Domain
- **Version-specific:** TypeVersion 4.6.
- **Edge cases / failures:**
  - If `expireDate` missing/invalid, `.split('-')` may error or produce wrong results.
  - Update will fail or update wrong row if:
    - `Websites` values are not unique,
    - there are leading/trailing spaces or different casing,
    - the column header is not exactly `Websites`.
  - `Days Left` being `"="` may overwrite existing data unintentionally.

#### Node: Pause Before Next Domain
- **Type / role:** Wait (`n8n-nodes-base.wait`) — rate limiting between domains.
- **Configuration choices (interpreted):**
  - Waits **30** (units depend on node UI; typically seconds for “amount” in this context).
- **Inputs/Outputs:**  
  - Input ← Write Expiry Data to Sheet  
  - Output → Loop Through Each Domain (to fetch next batch item)
- **Version-specific:** TypeVersion 1.1.
- **Edge cases / failures:**
  - Wait node pauses execution; long domain lists can create long-running executions.
  - n8n instance execution time limits (cloud or self-host) could stop the run mid-loop.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Domain Expiry Check | Manual Trigger | Manual entry point | — | Set Sheet Configuration | ## Domain Expiry Date Tracker<br>This workflow automatically checks domain expiry dates for<br>all websites listed in your Google Sheet. It loops through<br>each domain one by one, fetches DNS records using the<br>WHOIS API, extracts the expiry timestamp, converts it into<br>a readable date format, and writes the expiry date, month,<nand year directly back into your tracking sheet. Perfect<br>for managing large domain portfolios and staying ahead of<br>renewal deadlines.<br><br>## How it works<br>1. Reads all website domains from your Google Sheet.<br>2. Loops through each domain individually.<br>3. Queries the WHOIS API to retrieve DNS SOA records.<br>4. Extracts the expiry timestamp and converts it to DD-MM-YYYY.<br>5. Writes expiry date, month name, and year to the sheet.<br>6. Pauses 30 seconds before processing the next domain.<br><br>## Setup steps<br>1. Connect your Google Sheets OAuth credentials.<br>2. Add your RapidAPI WHOIS API key in the HTTP Request node.<br>3. Update Sheet ID and sheet name in "Set Sheet Configuration".<br>4. List all domains in the "Websites" column of your sheet.<br>5. Run the workflow manually to update all expiry dates. |
| Set Sheet Configuration | Set | Define spreadsheet config variables | Run Domain Expiry Check | Read All Domains from Sheet | ## Configuration & Data Loading<br>Sets the Google Sheet ID and sheet name,<br>then reads all domain rows from the sheet<br>to prepare for expiry checking. |
| Read All Domains from Sheet | Google Sheets | Load all domain rows from the sheet | Set Sheet Configuration | Loop Through Each Domain | ## Configuration & Data Loading<br>Sets the Google Sheet ID and sheet name,<br>then reads all domain rows from the sheet<br>to prepare for expiry checking. |
| Loop Through Each Domain | Split in Batches | Iterate domains one-by-one | Read All Domains from Sheet; Pause Before Next Domain | Fetch DNS Records via WHOIS API | ## Expiry Data Fetch & Processing<br>Loops through each domain, queries WHOIS API<br>for DNS records, then extracts and converts<br>the expiry timestamp into a readable date format. |
| Fetch DNS Records via WHOIS API | HTTP Request | Call RapidAPI WHOIS DNS endpoint | Loop Through Each Domain | Convert Expiry Timestamp to Date | ## Expiry Data Fetch & Processing<br>Loops through each domain, queries WHOIS API<br>for DNS records, then extracts and converts<br>the expiry timestamp into a readable date format. |
| Convert Expiry Timestamp to Date | Code | Transform SOA expire seconds into formatted date | Fetch DNS Records via WHOIS API | Write Expiry Data to Sheet | ## Expiry Data Fetch & Processing<br>Loops through each domain, queries WHOIS API<br>for DNS records, then extracts and converts<br>the expiry timestamp into a readable date format. |
| Write Expiry Data to Sheet | Google Sheets | Update matched row with expiry fields | Convert Expiry Timestamp to Date | Pause Before Next Domain | ## Update Sheet & Continue Loop<br>Writes the expiry date, month, and year<br>back to the Google Sheet, pauses briefly,<br>then moves to the next domain in the list. |
| Pause Before Next Domain | Wait | Throttle/rate-limit between domains | Write Expiry Data to Sheet | Loop Through Each Domain | ## Update Sheet & Continue Loop<br>Writes the expiry date, month, and year<br>back to the Google Sheet, pauses briefly,<br>then moves to the next domain in the list. |
| Sticky Note | Sticky Note | Documentation (no execution) | — | — | ## Domain Expiry Date Tracker<br>(content shown above) |
| Sticky Note1 | Sticky Note | Documentation (no execution) | — | — | ## Configuration & Data Loading<br>(content shown above) |
| Sticky Note2 | Sticky Note | Documentation (no execution) | — | — | ## Expiry Data Fetch & Processing<br>(content shown above) |
| Sticky Note3 | Sticky Note | Documentation (no execution) | — | — | ## Update Sheet & Continue Loop<br>(content shown above) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: `Run Domain Expiry Check`

3) **Add node: Set**
   - Node type: **Set**
   - Name: `Set Sheet Configuration`
   - Add fields:
     - `Sheet ID` (String): your spreadsheet ID (from Google Sheets URL)
     - `sites sheet` (String): your worksheet/tab name (ensure no unintended spaces)
     - (Optional) `Email Size` (Number): `30` (not used by this workflow)
   - Connect: `Run Domain Expiry Check` → `Set Sheet Configuration`

4) **Add node: Google Sheets (Read)**
   - Node type: **Google Sheets**
   - Name: `Read All Domains from Sheet`
   - Credentials: configure **Google Sheets OAuth2** (Google Cloud project + OAuth consent + scopes as required by n8n)
   - Set:
     - **Document ID**: expression `{{$json["Sheet ID"]}}`
     - **Sheet Name**: expression `{{$json["sites sheet"]}}`
     - Operation: “Read / Get Many” (read all rows)
   - Connect: `Set Sheet Configuration` → `Read All Domains from Sheet`

5) **Add node: Split in Batches**
   - Node type: **Split in Batches**
   - Name: `Loop Through Each Domain`
   - Batch size: set **1** (to process one domain per loop iteration)
   - Connect: `Read All Domains from Sheet` → `Loop Through Each Domain`

6) **Add node: HTTP Request**
   - Node type: **HTTP Request**
   - Name: `Fetch DNS Records via WHOIS API`
   - Method: **POST**
   - URL: `https://whois-api6.p.rapidapi.com/dns/api/v1/getRecords`
   - Send Headers: enabled
     - `x-rapidapi-host`: `whois-api6.p.rapidapi.com`
     - `x-rapidapi-key`: your RapidAPI key (preferably from n8n credentials or environment variables)
   - Send Body: enabled (Body Parameters)
     - `query`: expression `{{$json.Websites}}`
   - Error handling: set **On Error** to **Continue (regular output)** if you want the same behavior.
   - Connect: `Loop Through Each Domain` (the **loop/continue output**) → `Fetch DNS Records via WHOIS API`

7) **Add node: Code**
   - Node type: **Code**
   - Name: `Convert Expiry Timestamp to Date`
   - Paste logic (adapt as needed):
     - Read `result.records.SOA[0].expire`
     - Convert to date and format `DD-MM-YYYY`
     - Output `{ expireDate }`
   - Error handling: **Continue (regular output)** to match the workflow.
   - Connect: `Fetch DNS Records via WHOIS API` → `Convert Expiry Timestamp to Date`

8) **Add node: Google Sheets (Update)**
   - Node type: **Google Sheets**
   - Name: `Write Expiry Data to Sheet`
   - Credentials: same Google Sheets OAuth2
   - Operation: **Update**
   - Document ID: `{{$('Set Sheet Configuration').item.json["Sheet ID"]}}`
   - Sheet Name: `{{$('Set Sheet Configuration').item.json["sites sheet"]}}`
   - Matching:
     - Matching column: `Websites`
   - Columns to set (mapping):
     - `Websites`: `{{$('Loop Through Each Domain').item.json.Websites}}`
     - `Domain Expiry`: `{{$json.expireDate}}`
     - `Expiry Year`: `{{$json.expireDate.split('-')[2]}}`
     - `Expiry Month`: `{{ ['January','February','March','April','May','June','July','August','September','October','November','December'][+($json.expireDate.split('-')[1]) - 1] }}`
     - `Days Left`: set as desired (the original sets a literal `"="`; you may want a formula or computed number instead)
   - Connect: `Convert Expiry Timestamp to Date` → `Write Expiry Data to Sheet`

9) **Add node: Wait**
   - Node type: **Wait**
   - Name: `Pause Before Next Domain`
   - Amount: `30` (seconds)
   - Connect: `Write Expiry Data to Sheet` → `Pause Before Next Domain`

10) **Close the loop**
   - Connect: `Pause Before Next Domain` → `Loop Through Each Domain` (input), so it continues with the next batch item.

11) **Prepare the Google Sheet**
   - Ensure the sheet has a header column named exactly **`Websites`** containing the domains to check.
   - Ensure columns exist for: `Domain Expiry`, `Expiry Month`, `Expiry Year`, `Days Left` (or allow n8n to write into existing columns).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow computes a date using the DNS SOA `expire` timer (seconds) and `Date.now() + expireSeconds`. This may not represent the actual domain registration expiration date from the registrar. | Validate that the RapidAPI endpoint and field used match your intended “domain expiry” meaning. |
| The RapidAPI key is hardcoded in the HTTP Request node. Replace it with a secure credential or environment variable before sharing/exporting. | HTTP Request node: `x-rapidapi-key` header. |
| The configured sheet name includes leading/trailing spaces (`" SiteData "`). Remove spaces if your actual Google Sheets tab is `SiteData`. | Set Sheet Configuration → `sites sheet`. |
| The workflow uses “continue regular output” on HTTP/Code/Sheets update nodes, so later nodes may run even after upstream errors. Consider adding an IF node to handle error payloads safely. | Nodes: Fetch DNS Records via WHOIS API, Convert Expiry Timestamp to Date, Write Expiry Data to Sheet. |