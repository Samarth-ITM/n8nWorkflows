Log daily Interactive Brokers trades to a Google Sheets journal

https://n8nworkflows.xyz/workflows/log-daily-interactive-brokers-trades-to-a-google-sheets-journal-12151


# Log daily Interactive Brokers trades to a Google Sheets journal

## 1. Workflow Overview

**Purpose:** This workflow automatically pulls a daily trade report from **Interactive Brokers (IBKR) Flex Statements** and logs each trade into a **Google Sheets trading journal**, avoiding duplicates by matching on `tradeID`.

**Primary use cases**
- Maintain a daily-updated trade journal in Google Sheets
- Automate Flex report retrieval + parsing without manual downloads
- Ensure idempotent logging (update existing rows instead of appending duplicates)

### 1.1 Scheduling & Report Request (IBKR Flex)
A daily cron schedule triggers a request to IBKR to generate a Flex report, then extracts the `ReferenceCode` needed to download the statement.

### 1.2 Report Readiness Wait & Download
The workflow waits a fixed time (10 seconds) to allow IBKR to generate the report, then downloads the Flex statement content (XML).

### 1.3 Parsing & Trade Normalization
The XML is converted to JSON, trades are split into individual items, and a normalized set of fields is prepared for sheet insertion.

### 1.4 Google Sheets Upsert (Journal Write)
Each trade is appended or updated in Google Sheets using `tradeID` as the matching key.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & IBKR Flex Request
**Overview:** Triggers daily, calls IBKR’s FlexStatement `SendRequest` endpoint, and extracts the `ReferenceCode` from the XML response so the statement can later be retrieved.  
**Nodes involved:** `Daily Schedule Trigger`, `1. Request Flex Report`, `2. Extract Reference Code`

#### Node: Daily Schedule Trigger
- **Type / role:** Schedule Trigger (`scheduleTrigger`) — workflow entry point
- **Configuration (interpreted):**
  - Runs on cron: `0 8 * * *` (every day at **08:00** server time)
- **Outputs:** to `1. Request Flex Report`
- **Edge cases / failures:**
  - Timezone confusion (n8n instance timezone vs. expected local time)
  - If the workflow is inactive, it will not run (workflow is currently `active: false`)

#### Node: 1. Request Flex Report
- **Type / role:** HTTP Request — starts Flex report generation
- **Configuration (interpreted):**
  - GET request with query parameters to:
    - `https://gdcdyn.interactivebrokers.com/Universal/servlet/FlexStatementService.SendRequest`
  - Query parameters:
    - `t = YOUR_FLEX_TOKEN` (must be replaced)
    - `q = YOUR_QUERY_ID` (must be replaced)
    - `v = 3`
  - Response format: **text** (important because IBKR returns XML)
- **Key variables / expressions:** none (hard-coded placeholders are used)
- **Input:** from `Daily Schedule Trigger`
- **Output:** to `2. Extract Reference Code`
- **Edge cases / failures:**
  - Invalid token/query ID ⇒ IBKR error XML; extraction step will fail
  - IBKR endpoint rate limits or transient outages
  - Response not being plain text (misconfiguration) can break downstream parsing

#### Node: 2. Extract Reference Code
- **Type / role:** Code node — parses XML response text and extracts `<ReferenceCode>`
- **Configuration (interpreted):**
  - Attempts to locate XML in multiple possible shapes:
    - If `$input.item.json` is a string, uses it directly
    - Else tries `$input.item.json.body` or `.data`
    - Else JSON stringifies the object as a fallback
  - Regex extraction: `/<ReferenceCode>(.*?)<\/ReferenceCode>/`
  - Returns:
    - `referenceCode` (extracted)
    - `token: 'YOUR_FLEX_TOKEN'` (hard-coded placeholder; must match request token)
    - `debug_xml` (first 200 chars)
  - Throws an error if ReferenceCode not found (includes first 500 chars of response)
- **Key variables / expressions:**
  - Uses `$input.item.json` and returns `referenceCode`, `token`
- **Input:** from `1. Request Flex Report`
- **Output:** to `Wait 10 Seconds`
- **Edge cases / failures:**
  - IBKR error payload without `<ReferenceCode>` ⇒ node throws and workflow stops
  - If IBKR returns unexpected XML namespaces/format, regex may fail
  - Token is duplicated here as a hard-coded string; if you update the token in node 1 but not here, downloads will fail

---

### Block 2 — Wait & Download Flex Statement
**Overview:** Pauses to let IBKR generate the report, then downloads the statement XML using the extracted reference code.  
**Nodes involved:** `Wait 10 Seconds`, `3. Download Report`

#### Node: Wait 10 Seconds
- **Type / role:** Wait node — delay for report generation
- **Configuration (interpreted):**
  - Wait duration: `10` seconds
- **Input:** from `2. Extract Reference Code`
- **Output:** to `3. Download Report`
- **Edge cases / failures:**
  - 10 seconds may be insufficient during IBKR delays ⇒ download returns “not ready” / empty / error XML
  - Too long a wait increases execution time and could affect queueing under heavy load

#### Node: 3. Download Report
- **Type / role:** HTTP Request — retrieves the generated statement
- **Configuration (interpreted):**
  - GET request with query parameters to:
    - `https://gdcdyn.interactivebrokers.com/Universal/servlet/FlexStatementService.GetStatement`
  - Query parameters:
    - `t = {{ $json.token }}`
    - `q = {{ $json.referenceCode }}`
    - `v = 3`
  - Response format: **text** (IBKR returns XML)
- **Input:** from `Wait 10 Seconds`
- **Output:** to `4. Convert XML to JSON`
- **Edge cases / failures:**
  - Wrong token/referenceCode ⇒ error XML returned
  - Statement not ready yet ⇒ may return an error or incomplete response
  - Network/timeout issues; consider retry options if needed

---

### Block 3 — Parse, Split, and Normalize Trades
**Overview:** Converts the Flex XML into JSON, splits the list of trades into individual items, and prepares consistent fields for storage.  
**Nodes involved:** `4. Convert XML to JSON`, `5. Split Trades`, `6. Format Trade Fields`

#### Node: 4. Convert XML to JSON
- **Type / role:** XML node — parses XML text into structured JSON
- **Configuration (interpreted):**
  - Default XML parsing options (no custom settings defined)
- **Input:** from `3. Download Report` (expects XML string)
- **Output:** to `5. Split Trades`
- **Edge cases / failures:**
  - If the download returned non-XML or malformed XML, parsing fails
  - If IBKR returns an error XML with a different structure, downstream paths (field paths) may not exist

#### Node: 5. Split Trades
- **Type / role:** Split Out node — turns an array of trades into one item per trade
- **Configuration (interpreted):**
  - Splits out the field path:
    - `FlexQueryResponse.FlexStatements.FlexStatement.Trades.Trade`
- **Input:** from `4. Convert XML to JSON`
- **Output:** to `6. Format Trade Fields`
- **Edge cases / failures:**
  - If there are **no trades**, the path may be missing or not an array (node may output 0 items or error depending on actual structure)
  - If IBKR returns a single trade as an object (not array), split behavior may differ
  - If your Flex query includes different sections/levels, the path must be adjusted

#### Node: 6. Format Trade Fields
- **Type / role:** Set node — normalizes selected fields for Google Sheets
- **Configuration (interpreted):**
  - Creates/sets the following string fields from the trade item:
    - `tradeDate`, `symbol`, `quantity`, `buySell`, `tradeID`, `dateTime`,
      `tradePrice`, `currency`, `fxRateToBase`, `description`, `tradeMoney`
  - All values use direct expressions like `{{ $json.tradeDate }}`
- **Input:** from `5. Split Trades` (expects each item to be a trade record)
- **Output:** to `7. Save to Google Sheet`
- **Edge cases / failures:**
  - Missing fields in some trade rows ⇒ values become empty strings/undefined (sheet write may still succeed but journal quality suffers)
  - Types are forced to “string” by configuration; numeric/date handling will be as text unless Sheets formats them

---

### Block 4 — Upsert Trades into Google Sheets
**Overview:** Writes each normalized trade into Google Sheets, appending new trades or updating existing rows by matching on `tradeID`.  
**Nodes involved:** `7. Save to Google Sheet`

#### Node: 7. Save to Google Sheet
- **Type / role:** Google Sheets node — append or update rows (upsert)
- **Configuration (interpreted):**
  - Operation: **appendOrUpdate**
  - Document: `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - Sheet/tab selection: by ID `gid=0` (first tab in many sheets)
  - Column mapping: explicitly maps workflow fields to sheet columns:
    - `symbol`, `buySell`, `tradeID`, `currency`, `dateTime`, `quantity`,
      `tradeDate`, `tradeMoney`, `tradePrice`, `description`, `fxRateToBase`
  - Matching behavior:
    - `matchingColumns: ["tradeID"]` (prevents duplicates and updates existing rows)
  - Type conversion:
    - `attemptToConvertTypes: false` (values are written as provided)
- **Credentials:**
  - Uses Google Sheets OAuth2 (`Google Sheets - Naviqo`) — must be valid on your instance
- **Input:** from `6. Format Trade Fields`
- **Output:** end
- **Edge cases / failures:**
  - OAuth issues (expired/invalid token) ⇒ auth failures
  - Sheet permissions (service account/user lacks edit access)
  - If the sheet does not have a `tradeID` column header matching exactly, matching/upsert can fail or behave unexpectedly
  - If `tradeID` is not unique in the sheet, updates may target the wrong row(s) depending on n8n behavior

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Schedule Trigger | scheduleTrigger | Daily entry point (08:00 cron) | — | 1. Request Flex Report | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) |
| 1. Request Flex Report | httpRequest | Initiate IBKR Flex statement generation | Daily Schedule Trigger | 2. Extract Reference Code | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 1. Setup Flex Query in Interactive Brokers |
| 2. Extract Reference Code | code | Parse XML response and extract ReferenceCode | 1. Request Flex Report | Wait 10 Seconds | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 1. Setup Flex Query in Interactive Brokers |
| Wait 10 Seconds | wait | Delay to allow IBKR report generation | 2. Extract Reference Code | 3. Download Report | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 2. Download trade journal and format fields |
| 3. Download Report | httpRequest | Download Flex statement XML using referenceCode | Wait 10 Seconds | 4. Convert XML to JSON | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 2. Download trade journal and format fields |
| 4. Convert XML to JSON | xml | Convert IBKR XML to JSON | 3. Download Report | 5. Split Trades | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 2. Download trade journal and format fields |
| 5. Split Trades | splitOut | Create one item per trade record | 4. Convert XML to JSON | 6. Format Trade Fields | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 2. Download trade journal and format fields |
| 6. Format Trade Fields | set | Normalize trade fields for Sheets | 5. Split Trades | 7. Save to Google Sheet | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 2. Download trade journal and format fields |
| 7. Save to Google Sheet | googleSheets | Upsert trades into Google Sheets (match on tradeID) | 6. Format Trade Fields | — | # Automated IBKR Trade Report to Google Sheets Journal / How it works + Setup steps (checklist) ; ## 3. Save to Google Sheet (enter sheet ID + connect credentials) |
| Sticky Note | stickyNote | Documentation | — | — |  |
| Sticky Note1 | stickyNote | Documentation | — | — |  |
| Sticky Note2 | stickyNote | Documentation | — | — |  |
| Sticky Note3 | stickyNote | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `Automated IBKR Trade Report to Google Sheets Journal`
   - Keep it inactive until tested.

2. **Add trigger: “Schedule Trigger”**
   - Node type: **Schedule Trigger**
   - Set rule to **Cron**
   - Cron expression: `0 8 * * *` (adjust as needed)
   - This is the entry point.

3. **Add node: “HTTP Request” → “1. Request Flex Report”**
   - Node type: **HTTP Request**
   - Method: GET
   - URL: `https://gdcdyn.interactivebrokers.com/Universal/servlet/FlexStatementService.SendRequest`
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `t`: your IBKR Flex Web Service token (replace `YOUR_FLEX_TOKEN`)
     - `q`: your Flex Query ID (replace `YOUR_QUERY_ID`)
     - `v`: `3`
   - Set response format to **Text**
   - Connect: **Schedule Trigger → 1. Request Flex Report**

4. **Add node: “Code” → “2. Extract Reference Code”**
   - Node type: **Code**
   - Paste logic that:
     - Reads the XML string from the HTTP response
     - Extracts `<ReferenceCode>...</ReferenceCode>` via regex
     - Outputs JSON containing `referenceCode` and `token`
   - Important: ensure `token` equals your real Flex token (do not leave `YOUR_FLEX_TOKEN`).
   - Connect: **1. Request Flex Report → 2. Extract Reference Code**

5. **Add node: “Wait” → “Wait 10 Seconds”**
   - Node type: **Wait**
   - Wait time: `10 seconds` (increase if needed)
   - Connect: **2. Extract Reference Code → Wait 10 Seconds**

6. **Add node: “HTTP Request” → “3. Download Report”**
   - Node type: **HTTP Request**
   - Method: GET
   - URL: `https://gdcdyn.interactivebrokers.com/Universal/servlet/FlexStatementService.GetStatement`
   - Response format: **Text**
   - Query parameters:
     - `t`: expression `{{ $json.token }}`
     - `q`: expression `{{ $json.referenceCode }}`
     - `v`: `3`
   - Connect: **Wait 10 Seconds → 3. Download Report**

7. **Add node: “XML” → “4. Convert XML to JSON”**
   - Node type: **XML**
   - Input: the XML text from previous node
   - Default options are sufficient for this workflow structure
   - Connect: **3. Download Report → 4. Convert XML to JSON**

8. **Add node: “Split Out” → “5. Split Trades”**
   - Node type: **Split Out**
   - Field to split out:
     - `FlexQueryResponse.FlexStatements.FlexStatement.Trades.Trade`
   - Connect: **4. Convert XML to JSON → 5. Split Trades**

9. **Add node: “Set” → “6. Format Trade Fields”**
   - Node type: **Set**
   - Add fields (as strings) using expressions from the trade item:
     - `tradeDate`: `{{ $json.tradeDate }}`
     - `symbol`: `{{ $json.symbol }}`
     - `quantity`: `{{ $json.quantity }}`
     - `buySell`: `{{ $json.buySell }}`
     - `tradeID`: `{{ $json.tradeID }}`
     - `dateTime`: `{{ $json.dateTime }}`
     - `tradePrice`: `{{ $json.tradePrice }}`
     - `currency`: `{{ $json.currency }}`
     - `fxRateToBase`: `{{ $json.fxRateToBase }}`
     - `description`: `{{ $json.description }}`
     - `tradeMoney`: `{{ $json.tradeMoney }}`
   - Connect: **5. Split Trades → 6. Format Trade Fields**

10. **Add node: “Google Sheets” → “7. Save to Google Sheet”**
    - Node type: **Google Sheets**
    - Credentials: create/select **Google Sheets OAuth2** credentials
      - Ensure the Google account has edit access to the target sheet
    - Operation: **Append or Update** (`appendOrUpdate`)
    - Document/Spreadsheet: select by ID and paste your spreadsheet ID (replace `YOUR_GOOGLE_SHEET_ID`)
    - Sheet/tab: select the desired tab (this workflow uses `gid=0`)
    - Mapping mode: define columns manually and map:
      - `tradeDate`, `dateTime`, `symbol`, `description`, `quantity`, `buySell`,
        `tradePrice`, `tradeMoney`, `currency`, `fxRateToBase`, `tradeID`
    - Matching columns: set to `tradeID`
    - Connect: **6. Format Trade Fields → 7. Save to Google Sheet**

11. **Prepare the Google Sheet**
    - Ensure the header row contains columns matching your mappings (including `tradeID`)
    - Ensure `tradeID` is unique per trade to get reliable upserts

12. **Test execution**
    - Run manually once.
    - If the download returns “not ready”, increase the Wait duration.
    - Verify that rows are inserted/updated and `tradeID` matching works.

13. **Activate the workflow**
    - After successful manual test, activate to run daily.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Automated IBKR Trade Report to Google Sheets Journal” overview + setup checklist: schedule time, IBKR token/query ID, adjust wait time, select sheet/tab, connect OAuth2, confirm mapping and `tradeID` matching, run manual test. | Sticky Note: **Automated IBKR Trade Report to Google Sheets Journal** |
| “Setup Flex Query in Interactive Brokers” (heading placeholder). | Sticky Note: **1. Setup Flex Query in Interactive Brokers** |
| Save to Google Sheet: replace `YOUR_GOOGLE_SHEET_ID`; connect Google OAuth2 credentials in node 7. | Sticky Note: **3. Save to Google Sheet** |
| “Download trade journal and format fields” (block heading). | Sticky Note: **2. Download trade journal and format fields** |