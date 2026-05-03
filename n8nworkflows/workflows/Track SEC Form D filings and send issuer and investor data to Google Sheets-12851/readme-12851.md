Track SEC Form D filings and send issuer and investor data to Google Sheets

https://n8nworkflows.xyz/workflows/track-sec-form-d-filings-and-send-issuer-and-investor-data-to-google-sheets-12851


# Track SEC Form D filings and send issuer and investor data to Google Sheets

## 1. Workflow Overview

**Title:** Track SEC Form D filings and send issuer and investor data to Google Sheets

**Purpose:**  
This workflow monitors the SEC EDGAR “current filings” Atom feed for **Form D** (and **D/A**) submissions, queues newly discovered filings into a Google Sheet, then downloads each filing’s full text document, extracts key issuer/personnel/offering fields from the embedded XML, and finally updates the same Google Sheet row as **processed**. It runs on a schedule during business hours and includes a delay to reduce load on SEC endpoints.

### 1.1 Scheduled intake (feed polling)
Poll the SEC Atom feed for recent Form D filings every 10 minutes during business hours.

### 1.2 Normalize + de-duplicate + queue in Google Sheets
Convert feed entries into a flat structure, remove items already processed in previous workflow executions, and append new rows into a tracking sheet with an empty `status`.

### 1.3 Select unprocessed filings + batch processing loop
Filter queued rows to only those with `status` empty and form type matching `D` or `D/A`, then process items in batches (Split in Batches pattern).

### 1.4 Retrieve and parse filing document XML
Download the `.txt` filing document, extract the `<XML>...</XML>` segment, and parse issuer details, key personnel, and offering financial details.

### 1.5 Consolidate + update Google Sheets + rate limiting
Build a summary payload and update the matching Google Sheets row (matching on `filingLinkTxt`), mark it `processed`, then wait before looping to the next filing.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled intake (feed polling)

**Overview:** Triggers periodically and fetches the SEC EDGAR Atom feed for Form D filings.  
**Nodes involved:** `Schedule Trigger`, `Fetch SEC Form D Feed`, `Parse XML Feed`

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger; entry point to run on a cron-like schedule.
- **Configuration choices:**
  - Cron expression: `*/10 6-21 * * 1-5` → every 10 minutes, 06:00–21:59, Monday–Friday.
- **Inputs/outputs:** No input; outputs one trigger item to `Fetch SEC Form D Feed`.
- **Version notes:** typeVersion `1.2` (standard schedule behavior).
- **Failure/edge cases:**
  - Timezone depends on n8n instance settings; confirm expected timezone for “business hours”.

#### Node: Fetch SEC Form D Feed
- **Type / role:** HTTP Request; downloads SEC Atom feed.
- **Configuration choices:**
  - GET `https://www.sec.gov/cgi-bin/browse-edgar?action=getcurrent&CIK=&type=D&company=&dateb=&owner=include&start=0&count=40&output=atom`
  - Sends headers:
    - `User-Agent: Nick Automations user@example.com` (SEC expects a descriptive UA)
    - `Accept-Encoding: gzip, deflate`
- **Inputs/outputs:** Input from trigger; output XML text to `Parse XML Feed`.
- **Version notes:** typeVersion `4.2`.
- **Failure/edge cases:**
  - SEC can return `403`/`429` if User-Agent is missing/insufficient or if rate-limited.
  - Network timeouts, transient 5xx errors.
  - Feed format changes could break parsing.

#### Node: Parse XML Feed
- **Type / role:** XML node; converts Atom XML into JSON.
- **Configuration choices:** Default XML parsing options.
- **Inputs/outputs:** Input from `Fetch SEC Form D Feed`; output JSON to `Transform Feed Entries`.
- **Version notes:** typeVersion `1`.
- **Failure/edge cases:**
  - If SEC returns HTML error page instead of XML, parsing fails.
  - Atom entries may be a single object vs array in some parsers; downstream code assumes `feed.entry` exists.

**Sticky note context (applies to nodes above):**  
“📥 STEP 1: Collect New Filings … Fetches the SEC EDGAR atom feed for Form D filings and parses the XML response into structured data…”

---

### Block 2 — Normalize + de-duplicate + queue in Google Sheets

**Overview:** Flattens each feed entry to a consistent object, removes entries already seen in previous runs, and appends new filings to a Google Sheet for tracking/queuing.  
**Nodes involved:** `Transform Feed Entries`, `Filter Previously Processed Filings`, `Add New Filing to Tracking Sheet`

#### Node: Transform Feed Entries
- **Type / role:** Code node; maps Atom feed entries into flat items used throughout workflow.
- **Configuration choices (interpreted):**
  - Reads `$('Parse XML Feed').first().json.feed.entry`.
  - Extracts **CIK** from `entry.title` using regex `\((\d{10})\)` then strips leading zeros.
  - Builds TXT link from HTML link by replacing `-index.htm` with `.txt`.
  - Outputs fields:
    - `cikNumber`, `title`, `formType` (from `entry.category.term`), `filingLinkHtml`, `filingLinkTxt`, `updated`, `status: ""`.
- **Inputs/outputs:** Input from `Parse XML Feed`; outputs multiple items to `Filter Previously Processed Filings`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - If SEC changes title format and CIK not in parentheses, `cikNumber` becomes empty.
  - If link format changes (no `-index.htm`), TXT conversion may be wrong.

#### Node: Filter Previously Processed Filings
- **Type / role:** Remove Duplicates; prevents re-processing the same filing across workflow executions.
- **Configuration choices:**
  - Operation: “remove items seen in previous executions”
  - Dedupe value: `{{ $json.filingLinkTxt }}`
  - History size: `1000000` (very large memory of seen items).
- **Inputs/outputs:** Input from `Transform Feed Entries`; output unique new items to `Add New Filing to Tracking Sheet`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - Large `historySize` increases persisted history; can impact storage over time.
  - If `filingLinkTxt` is empty, many items could be treated as duplicates incorrectly.

#### Node: Add New Filing to Tracking Sheet
- **Type / role:** Google Sheets; appends rows to create a processing queue.
- **Configuration choices:**
  - Operation: **Append**
  - Document: “SEC Data” (Spreadsheet ID: `1VoGfVpk1mMrqKIc5hsO7peYuLx0SwhsbW7uUeYJCmrU`)
  - Sheet tab: `Sheet1` (`gid=0`)
  - Writes (notable mappings):
    - `updated`: `{{ $json.updated.toDateTime().format('yyyy-MM-dd') }}`
    - `status`: initially empty string
    - plus `cikNumber`, `title`, `formType`, `filingLinkTxt`, `filingLinkHtml`
- **Inputs/outputs:** Input from `Filter Previously Processed Filings`; output rows to `Filter Unprocessed Form D Filings`.
- **Credentials:** Google Sheets OAuth2 (`[Naveen]Google Sheets account`).
- **Version notes:** typeVersion `4.6`.
- **Failure/edge cases:**
  - Missing/expired Google OAuth token.
  - Sheet schema mismatch: columns in sheet must exist and match expected names.
  - The expression `.toDateTime()` requires `updated` to be a parseable date-time string.

**Sticky note context (applies to nodes in this block):**  
“🔍 STEP 2: Filter & Track … removes duplicates based on filing URL, and adds new filings to Google Sheets with empty status…”

---

### Block 3 — Select unprocessed filings + batch processing loop

**Overview:** Filters the queued Google Sheet rows for unprocessed Form D / D/A entries, then uses Split In Batches to iterate through filings safely.  
**Nodes involved:** `Filter Unprocessed Form D Filings`, `Process Filings Batch`

#### Node: Filter Unprocessed Form D Filings
- **Type / role:** IF; selects which rows should be processed.
- **Configuration choices:**
  - Condition 1: `status` is empty (`{{ $json.status }}` “empty”)
  - Condition 2: `formType` matches regex `^D(/A)?$` to allow `D` or `D/A`.
- **Inputs/outputs:**
  - Input from `Add New Filing to Tracking Sheet`.
  - **True** → `Process Filings Batch`
  - **False** → no downstream connection (ignored).
- **Version notes:** typeVersion `2.2`.
- **Failure/edge cases:**
  - If `status` column contains whitespace instead of empty, it will not match “empty”.
  - If the Atom feed uses a different form representation (e.g., `D/A` vs `D/A `), regex may fail unless trimmed.

#### Node: Process Filings Batch
- **Type / role:** Split In Batches; controls iterative processing of multiple filings.
- **Configuration choices:** Default options (batch size not explicitly set in JSON; n8n default is typically 1 unless configured).
- **Inputs/outputs:**
  - Input from IF (true path).
  - Output **index 1** is connected to `Retrieve Form D Document` (this indicates the “current batch item(s)” output is used).
  - Output **index 0** is fed from `Rate Limit Delay` back into this node, forming the iteration loop.
- **Version notes:** typeVersion `3`.
- **Failure/edge cases:**
  - Miswiring SplitInBatches outputs can cause either only one item processed or infinite loop. Here, the “continue” loop is driven by `Rate Limit Delay → Process Filings Batch`.

**Sticky note context:**  
“⚡ STEP 3: Batch Processing … processes them in batches to avoid overwhelming the SEC servers…”

---

### Block 4 — Retrieve and parse filing document XML

**Overview:** Downloads the TXT filing, extracts the embedded XML portion, then parses issuer details, key personnel, and offering financial details using regex-based extraction.  
**Nodes involved:** `Retrieve Form D Document`, `Parse JSON and Extract SEC Document`, `Parse Issuer Details`, `Extract Key Personnel`, `Extract Offering Financial Details`

#### Node: Retrieve Form D Document
- **Type / role:** HTTP Request; fetches the full filing text document.
- **Configuration choices:**
  - URL: `{{ $json.filingLinkTxt }}`
  - Headers:
    - `User-Agent: iRocket VC user@example.com`
    - `Accept-Encoding: gzip, deflate`
  - Response options: configured to return the raw body (default-like behavior in v4).
- **Inputs/outputs:** Input from `Process Filings Batch`; output to `Parse JSON and Extract SEC Document`.
- **Version notes:** typeVersion `4.2`.
- **Failure/edge cases:**
  - SEC throttling (429), forbidden (403) if UA unacceptable.
  - Filing link may be stale or invalid.
  - Response may be compressed/encoded unexpectedly.

#### Node: Parse JSON and Extract SEC Document
- **Type / role:** Code node; normalizes different possible HTTP response shapes and extracts `<XML>...</XML>`.
- **Configuration choices (interpreted):**
  - `onError: continueRegularOutput` but code uses `throw new Error('No XML content found')`; behavior depends on n8n handling—node will output an error item or continue with previous data depending on runtime settings.
  - Determines `docText` based on input structure:
    - If input is array: `inputData[0].data`
    - Else if `inputData.data`: use it
    - Else uses `inputData.body || inputData`
  - Extracts:
    - `rawXml`: first `<XML> ... </XML>` block
    - `filingDate`: via `FILED AS OF DATE:\s*(\d+)`
    - `accessionNumber`: via `ACCESSION NUMBER: ...`
    - `submissionType`: via `CONFORMED SUBMISSION TYPE: ...`
  - Also pulls `filingLinkTxt` from `$("Process Filings Batch").first().json`.
- **Inputs/outputs:** Input from `Retrieve Form D Document`; output to `Parse Issuer Details`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - Some SEC filings contain multiple XML blocks; code only takes the first match.
  - If `<XML>` tag casing differs or missing, node throws.
  - Reliance on `Process Filings Batch` data means if the batch item doesn’t include `filingLinkTxt`, summary will break.

#### Node: Parse Issuer Details
- **Type / role:** Code node; extracts issuer/company fields from `rawXml`.
- **Configuration choices (interpreted):**
  - Regex extracts for: `entityName`, `cik`, `street1`, `street2`, `city`, `stateOrCountry`, `zipCode`, `issuerPhoneNumber`, `jurisdictionOfInc`, first 4-digit `<value>` (year), `entityType`.
  - Fund-specific: `industryGroupType`, `investmentFundType`.
  - Builds a single formatted address string.
  - Outputs `companyInfo` object.
- **Inputs/outputs:** Input from `Parse JSON and Extract SEC Document`; output to `Extract Key Personnel`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - If any address fields are missing, address string may include `undefined` segments (street1 is assumed present).
  - The year regex `/<value>(\d{4})<\/value>/` might match unrelated `<value>` tags if the XML structure changes.

#### Node: Extract Key Personnel
- **Type / role:** Code node; extracts “related person” entries (executives, promoters, etc.).
- **Configuration choices (interpreted):**
  - Reads `rawXml` from `Parse JSON and Extract SEC Document`.
  - Finds each `<relatedPersonInfo> ... </relatedPersonInfo>` block.
  - Extracts:
    - `firstName`, `lastName`, `relationshipClarification`
    - multiple `<relationship>...</relationship>` roles
  - Special handling:
    - If `firstName === '-'`, treats as **Entity** and uses lastName as name.
    - Else treats as **Individual** and uses `firstName lastName`.
  - Outputs `keyPersonnel` array with `{name, roles, clarification, type}`.
- **Inputs/outputs:** Input from `Parse Issuer Details` (though it actually references the earlier node directly); output to `Extract Offering Financial Details`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - XML may omit `relationship` tags; roles becomes empty array.
  - `firstName` sometimes blank rather than `-`; entity detection may fail.
  - Regex parsing XML is fragile if SEC adds namespaces or formatting changes.

#### Node: Extract Offering Financial Details
- **Type / role:** Code node; extracts offering amounts, investors, exemptions, and timing flags.
- **Configuration choices (interpreted):**
  - Extracts:
    - `totalOfferingAmount`, `totalAmountSold`, `totalRemaining`
    - `totalNumberAlreadyInvested`
    - `hasNonAccreditedInvestors`, `yetToOccur`, `moreThanOneYear`
  - Extracts exemptions by scanning every `<item>...</item>` in the XML (note: this may capture unrelated lists).
  - Formats currency for `totalAmountSold` only, with special handling for `Indefinite` and `0`.
  - Normalizes exemption codes:
    - `06b` → `Rule 506(b)`
    - `3C` → `Investment Company Act Section 3(c)`
    - `3C.7` → `Section 3(c)(7)`
  - Determines:
    - `allAccreditedInvestors` = `hasNonAccredited === 'false'`
    - `saleStatus` and `offeringDuration` from boolean text.
- **Inputs/outputs:** Input from `Extract Key Personnel`; output to `Consolidate Filing Data`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - `totalOfferingAmount` is not currency-formatted (unlike amount sold); may be raw numeric or “Indefinite”.
  - `<item>` regex is broad and can over-collect.
  - Booleans may be `true/false` vs `1/0`; current logic expects string `'true'/'false'`.

**Sticky note context:**  
“🔬 STEP 4: Extract Filing Details … Downloads the full Form D document and parses the XML content to extract …”

---

### Block 5 — Consolidate + update Google Sheets + rate limiting

**Overview:** Combines all extracted data, selects a “key executive”, updates the row in Google Sheets, and waits before processing the next filing.  
**Nodes involved:** `Consolidate Filing Data`, `Update Filing Status in Sheet`, `Rate Limit Delay`

#### Node: Consolidate Filing Data
- **Type / role:** Code node; merges issuer/personnel/offering/document fields into a summary and detailed object.
- **Configuration choices (interpreted):**
  - Pulls:
    - `companyInfo` from `Parse Issuer Details`
    - `personnel` from `Extract Key Personnel`
    - `offering` from `Extract Offering Financial Details`
    - `docInfo` from `Parse JSON and Extract SEC Document`
  - Splits personnel into `individuals` vs `entities`.
  - Determines `keyExecutive` preference order:
    1. Role contains “chief executive” or “ceo”
    2. Role contains “executive officer”
    3. First individual in list
  - Produces:
    - `summary`: fields needed for sheet update (companyName, keyExecutive, fundType, amounts, investorCount, exemptions, filingDate, etc.)
    - `fullDetails`: richer nested object (not written to Sheets in this workflow, but useful for extensions).
- **Inputs/outputs:** Input from `Extract Offering Financial Details`; output to `Update Filing Status in Sheet`.
- **Version notes:** typeVersion `2`.
- **Failure/edge cases:**
  - If no individuals exist, `keyExecutive` and `executiveRole` remain empty.
  - If upstream nodes error/return empty due to `onError` behavior, cross-node references like `$('Parse Issuer Details').all()[0]` can throw.

#### Node: Update Filing Status in Sheet
- **Type / role:** Google Sheets; updates the existing queued row and marks it processed.
- **Configuration choices:**
  - Operation: **Update**
  - Document: “SEC Data” (same spreadsheet)
  - Sheet tab: `Summary` (`gid=0` as cached name indicates “Summary”, but gid=0 typically first tab—confirm in your sheet)
  - **Matching column:** `filingLinkTxt` (this is critical)
  - Writes:
    - `status: processed`
    - `fundType`, `exemptions`, `filingDate`, `companyName`, `salesStatus`, `amountRaised`, `keyExecutive`, `targetAmount`, `executiveRole`, `investorCount`
    - `filingLinkTxt`: `{{ $json.summary.filingLinkText }}`
- **Inputs/outputs:** Input from `Consolidate Filing Data`; output to `Rate Limit Delay`.
- **Credentials:** Google Sheets OAuth2 (`[Naveen]Google Sheets account`).
- **Version notes:** typeVersion `4.6`.
- **Failure/edge cases:**
  - If the sheet tab or column names differ, update will not find matching row.
  - If `filingLinkTxt` is not unique, multiple rows could be updated unexpectedly (depending on n8n behavior).
  - Schema mismatch or missing permissions.

#### Node: Rate Limit Delay
- **Type / role:** Wait node; introduces a delay between processed filings.
- **Configuration choices:** No explicit duration configured in parameters (requires verification in UI; could be “wait until resumed” or default mode depending on node settings/version).
- **Inputs/outputs:** Input from `Update Filing Status in Sheet`; output loops back to `Process Filings Batch` to continue with the next item.
- **Version notes:** typeVersion `1.1`.
- **Failure/edge cases:**
  - If configured as “Wait for webhook” (indicated by presence of `webhookId`), it may pause indefinitely unless externally resumed. In many SEC-rate-limit patterns, a fixed time delay is preferred—verify the wait mode in the editor.

**Sticky note context:**  
“✅ STEP 5: Update & Rate Limit … updates the Google Sheet row … The Wait node adds a delay between filings…”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Time-based workflow start | — | Fetch SEC Form D Feed | 📊 SEC Form D Filing Tracker… Runs every 10 minutes, Mon-Fri, 6 AM - 9 PM… |
| Fetch SEC Form D Feed | HTTP Request | Download SEC Atom feed for Form D | Schedule Trigger | Parse XML Feed | 📥 STEP 1: Collect New Filings… |
| Parse XML Feed | XML | Parse Atom XML to JSON | Fetch SEC Form D Feed | Transform Feed Entries | 📥 STEP 1: Collect New Filings… |
| Transform Feed Entries | Code | Flatten entries; compute CIK and .txt link | Parse XML Feed | Filter Previously Processed Filings | 🔍 STEP 2: Filter & Track… |
| Filter Previously Processed Filings | Remove Duplicates | Avoid reprocessing across executions | Transform Feed Entries | Add New Filing to Tracking Sheet | 🔍 STEP 2: Filter & Track… |
| Add New Filing to Tracking Sheet | Google Sheets | Append new filings as queue rows | Filter Previously Processed Filings | Filter Unprocessed Form D Filings | 🔍 STEP 2: Filter & Track… |
| Filter Unprocessed Form D Filings | IF | Select status empty + D/D-A | Add New Filing to Tracking Sheet | Process Filings Batch (true) | ⚡ STEP 3: Batch Processing… |
| Process Filings Batch | Split In Batches | Iterate filings sequentially | Filter Unprocessed Form D Filings (true), Rate Limit Delay (loop) | Retrieve Form D Document | ⚡ STEP 3: Batch Processing… |
| Retrieve Form D Document | HTTP Request | Download full filing text | Process Filings Batch | Parse JSON and Extract SEC Document | 🔬 STEP 4: Extract Filing Details… |
| Parse JSON and Extract SEC Document | Code | Extract `<XML>` and doc metadata | Retrieve Form D Document | Parse Issuer Details | 🔬 STEP 4: Extract Filing Details… |
| Parse Issuer Details | Code | Extract issuer/company info | Parse JSON and Extract SEC Document | Extract Key Personnel | 🔬 STEP 4: Extract Filing Details… |
| Extract Key Personnel | Code | Extract related persons and roles | Parse Issuer Details | Extract Offering Financial Details | 🔬 STEP 4: Extract Filing Details… |
| Extract Offering Financial Details | Code | Extract amounts, investors, exemptions | Extract Key Personnel | Consolidate Filing Data | 🔬 STEP 4: Extract Filing Details… |
| Consolidate Filing Data | Code | Merge outputs; choose key executive | Extract Offering Financial Details | Update Filing Status in Sheet | ✅ STEP 5: Update & Rate Limit… |
| Update Filing Status in Sheet | Google Sheets | Update row; mark processed | Consolidate Filing Data | Rate Limit Delay | ✅ STEP 5: Update & Rate Limit… |
| Rate Limit Delay | Wait | Pause between filings; loop control | Update Filing Status in Sheet | Process Filings Batch | ✅ STEP 5: Update & Rate Limit… |
| Sticky Note | Sticky Note | Comment / documentation | — | — | 📊 SEC Form D Filing Tracker… (contains requirements, schedule, tip) |
| Sticky Note1 | Sticky Note | Comment / documentation | — | — | 📥 STEP 1: Collect New Filings… |
| Sticky Note2 | Sticky Note | Comment / documentation | — | — | 🔍 STEP 2: Filter & Track… |
| Sticky Note3 | Sticky Note | Comment / documentation | — | — | ⚡ STEP 3: Batch Processing… |
| Sticky Note4 | Sticky Note | Comment / documentation | — | — | 🔬 STEP 4: Extract Filing Details… |
| Sticky Note5 | Sticky Note | Comment / documentation | — | — | ✅ STEP 5: Update & Rate Limit… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add “Schedule Trigger”**
   - Interval rule → Cron expression: `*/10 6-21 * * 1-5`
   - Ensure instance timezone matches your expectations.

3. **Add “HTTP Request” → name it “Fetch SEC Form D Feed”**
   - Method: GET
   - URL: `https://www.sec.gov/cgi-bin/browse-edgar?action=getcurrent&CIK=&type=D&company=&dateb=&owner=include&start=0&count=40&output=atom`
   - Headers:
     - `User-Agent`: set to **your org/name and your email** (SEC requirement)
     - `Accept-Encoding`: `gzip, deflate`
   - Connect: `Schedule Trigger → Fetch SEC Form D Feed`

4. **Add “XML” → name it “Parse XML Feed”**
   - Use default options
   - Connect: `Fetch SEC Form D Feed → Parse XML Feed`

5. **Add “Code” → name it “Transform Feed Entries”**
   - Paste logic to:
     - read `feed.entry`
     - extract CIK from title
     - derive `.txt` link by replacing `-index.htm` with `.txt`
     - output `status: ""`
   - Connect: `Parse XML Feed → Transform Feed Entries`

6. **Add “Remove Duplicates” → name it “Filter Previously Processed Filings”**
   - Operation: *Remove items seen in previous executions*
   - Dedupe value: `{{ $json.filingLinkTxt }}`
   - History size: `1000000` (adjust to your retention needs)
   - Connect: `Transform Feed Entries → Filter Previously Processed Filings`

7. **Prepare your Google Sheet**
   - Create a spreadsheet (e.g., “SEC Data”).
   - Ensure columns exist for at least:
     - `cikNumber`, `title`, `formType`, `filingLinkHtml`, `filingLinkTxt`, `updated`, `status`
     - plus the later update fields: `companyName`, `keyExecutive`, `executiveRole`, `fundType`, `targetAmount`, `amountRaised`, `investorCount`, `salesStatus`, `filingDate`, `exemptions`, `comment`
   - Ensure `filingLinkTxt` is unique per row (recommended).

8. **Add “Google Sheets” → name it “Add New Filing to Tracking Sheet”**
   - Credentials: Google Sheets OAuth2 (connect your Google account)
   - Operation: **Append**
   - Choose Document (your spreadsheet) and Sheet (queue tab, e.g., `Sheet1`)
   - Map fields:
     - `title = {{ $json.title }}`
     - `status = {{ $json.status }}`
     - `updated = {{ $json.updated.toDateTime().format('yyyy-MM-dd') }}`
     - `formType, cikNumber, filingLinkTxt, filingLinkHtml`
   - Connect: `Filter Previously Processed Filings → Add New Filing to Tracking Sheet`

9. **Add “IF” → name it “Filter Unprocessed Form D Filings”**
   - Conditions (AND):
     - `status` is empty (`{{ $json.status }}` operator “empty”)
     - `formType` regex matches `^D(/A)?$`
   - Connect: `Add New Filing to Tracking Sheet → Filter Unprocessed Form D Filings`

10. **Add “Split In Batches” → name it “Process Filings Batch”**
    - Set batch size (recommend **1** to be gentle with SEC).
    - Connect: `Filter Unprocessed Form D Filings (true) → Process Filings Batch`

11. **Add “HTTP Request” → name it “Retrieve Form D Document”**
    - Method: GET
    - URL: `{{ $json.filingLinkTxt }}`
    - Headers: provide your own SEC-friendly `User-Agent` + `Accept-Encoding`
    - Connect: `Process Filings Batch → Retrieve Form D Document`

12. **Add “Code” → name it “Parse JSON and Extract SEC Document”**
    - Configure **On Error**: “Continue regular output” (as in the workflow)
    - Code should:
      - normalize response body to a string
      - extract `<XML>...</XML>`
      - extract filingDate/accessionNumber/submissionType from the raw text
      - carry forward `filingLinkTxt` from the batch item
    - Connect: `Retrieve Form D Document → Parse JSON and Extract SEC Document`

13. **Add “Code” → name it “Parse Issuer Details”**
    - Extract issuer details from `rawXml` and output `companyInfo`.
    - Connect: `Parse JSON and Extract SEC Document → Parse Issuer Details`

14. **Add “Code” → name it “Extract Key Personnel”**
    - Extract `<relatedPersonInfo>` blocks and output `keyPersonnel`.
    - Connect: `Parse Issuer Details → Extract Key Personnel`

15. **Add “Code” → name it “Extract Offering Financial Details”**
    - Extract offering amounts, investors, exemptions, sale flags.
    - Connect: `Extract Key Personnel → Extract Offering Financial Details`

16. **Add “Code” → name it “Consolidate Filing Data”**
    - Merge `companyInfo`, `keyPersonnel`, `offeringDetails`, and doc info.
    - Choose `keyExecutive` using the CEO → executive officer → first person logic.
    - Connect: `Extract Offering Financial Details → Consolidate Filing Data`

17. **Add “Google Sheets” → name it “Update Filing Status in Sheet”**
    - Operation: **Update**
    - Same document; choose the sheet tab where the queue rows exist (must contain `filingLinkTxt`).
    - Matching column: `filingLinkTxt`
    - Set:
      - `status = processed`
      - Map remaining summary fields (`companyName`, `fundType`, `amountRaised`, etc.)
    - Connect: `Consolidate Filing Data → Update Filing Status in Sheet`

18. **Add “Wait” → name it “Rate Limit Delay”**
    - Configure a fixed delay (recommended, e.g., 2–5 seconds) to respect SEC rate limits.
    - Connect: `Update Filing Status in Sheet → Rate Limit Delay`

19. **Close the batch loop**
    - Connect: `Rate Limit Delay → Process Filings Batch` (this continues to next item until done).

20. **Activate the workflow**
    - Run once manually to verify:
      - Atom feed parsing works
      - Rows append correctly
      - Update finds the correct row via `filingLinkTxt`
      - Wait node behaves as intended (does not pause indefinitely)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Replace nchoudhary110792@gmail.com with your email in HTTP nodes” | From the workflow’s main sticky note requirement. Update all SEC HTTP `User-Agent` headers accordingly. |
| “This workflow respects SEC rate limits. The Wait node prevents overwhelming the EDGAR system.” | Rate limiting is essential; also consider handling 429 with retries/backoff. |
| Schedule: “Runs every 10 minutes, Mon-Fri, 6 AM - 9 PM (Customizable via Schedule Trigger node)” | Controlled by the cron expression in `Schedule Trigger`. |
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé…” | Context: confirms content is from an automation workflow and intended to be compliant. |