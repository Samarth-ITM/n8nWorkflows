Generate Meta Ads campaign reports in Google Sheets and send Telegram alerts

https://n8nworkflows.xyz/workflows/generate-meta-ads-campaign-reports-in-google-sheets-and-send-telegram-alerts-13745


# Generate Meta Ads campaign reports in Google Sheets and send Telegram alerts

This document provides a technical breakdown of the n8n workflow designed to automate daily Meta Ads reporting and Telegram alerts for multiple clients.

### 1. Workflow Overview
The workflow automates the collection of Meta Ads performance data (spend, impressions, clicks, CTR, etc.) for various clients stored in a central registry. It processes each client individually, writes campaign-level data to their specific Google Sheets reports, and sends a performance summary with diagnostic alerts (e.g., "Weak Creative" or "Good Candidate to Scale") via Telegram.

**Logical Blocks:**
*   **1.1 Data Initialization:** Triggers the flow and retrieves the list of clients and their Meta/Google credentials from a master sheet.
*   **1.2 Client Iteration & Context:** Loops through each client and prepares a "context" object to maintain metadata across nodes.
*   **1.3 Data Acquisition:** Queries the Meta Graph API for "yesterday's" campaign insights.
*   **1.4 Transformation & Extraction:** Flattens API results, normalizes metrics, and extracts Spreadsheet IDs from URLs.
*   **1.5 Reporting & Notification:** Appends data to specific Google Sheets and generates a formatted summary for Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Initialization
*   **Overview:** Sets the execution schedule and pulls the client configuration (API tokens, Account IDs, and Sheet URLs).
*   **Nodes Involved:** `Schedule Trigger`, `Get row(s) in sheet`.
*   **Node Details:**
    *   **Schedule Trigger:** Set to run daily at 10:00 AM.
    *   **Get row(s) in sheet:** Connects to a master registry. It expects columns like `ad_account_id`, `meta_access_token`, and `report_sheet_url`. Failure here (e.g., Google auth error) stops the entire process.

#### 2.2 Client Iteration & Context
*   **Overview:** Manages the "for-each" logic and ensures client-specific data is preserved.
*   **Nodes Involved:** `Loop Over Items`, `ctx`.
*   **Node Details:**
    *   **Loop Over Items:** Batches the client list to process one client at a time.
    *   **ctx (Code Node):** Creates a nested `ctx` object within the JSON. This is a best practice to prevent data loss when merging results from the Meta API later.
    *   **Edge Cases:** If the client registry has empty rows, downstream nodes may fail due to missing tokens.

#### 2.3 Data Acquisition
*   **Overview:** Fetches campaign-level metrics directly from Meta.
*   **Nodes Involved:** `HTTP Request в ads manager клиента`, `If`.
*   **Node Details:**
    *   **HTTP Request:** Hits `graph.facebook.com/v25.0/act_{{ad_account_id}}/insights`. It uses a Bearer Token for authorization. Query params are set to `level=campaign` and `date_preset=yesterday`.
    *   **If:** Checks if the Meta API actually returned data (`$json.data.length > 0`). If no ads ran yesterday, it skips the reporting for that client to avoid errors.

#### 2.4 Transformation & Extraction
*   **Overview:** Prepares the raw API response for spreadsheet insertion.
*   **Nodes Involved:** `Merge`, `Split campaigns`, `Extract spreadsheetId`.
*   **Node Details:**
    *   **Merge:** Combines the Meta API results with the original client context (`ctx`).
    *   **Split campaigns (Code Node):** Converts the array of campaigns into individual n8n items. It also converts string metrics (like spend) into Numbers to ensure Google Sheets treats them as values, not text.
    *   **Extract spreadsheetId (Code Node):** Uses a Regular Expression (`/\/d\/([a-zA-Z0-9-_]+)/`) to turn a full Google Sheets URL into a specific ID required by the Google Sheets node.

#### 2.5 Reporting & Notification
*   **Overview:** Writes to the database and alerts the user.
*   **Nodes Involved:** `Append row in sheet`, `Code in JavaScript`, `Send a text message`.
*   **Node Details:**
    *   **Append row in sheet:** Uses "Auto Map" to match keys to columns. The Document ID is dynamic (`{{$json.report_spreadsheet_id}}`).
    *   **Code in JavaScript:** Aggregates all campaigns for the client into one summary. It applies **Diagnostic Logic**:
        *   *Weak Creative:* CTR < 1.5%
        *   *Expensive:* CPC > $0.50
        *   *Scaling Candidate:* CTR ≥ 2% AND CPC ≤ $0.50
    *   **Send a text message (Telegram):** Sends the formatted message to a specific Chat ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule | Trigger | None | Get row(s) in sheet | Runs on a schedule to automate daily/weekly reporting. |
| Get row(s) in sheet | Google Sheets | Data Source | Schedule Trigger | Loop Over Items | Loads the [client list](https://docs.google.com/spreadsheets/d/11repzs1T85xH0waYFkngj8nNzqisWZ27K4gjNKVNlZU/edit?usp=sharing). |
| Loop Over Items | SplitInBatches | Iterator | Get row(s)..., Send a text... | ctx | Loops through each client from the register sheet. |
| ctx | Code | Data Prep | Loop Over Items | HTTP Request..., Merge | Builds a context object to simplify mapping. |
| HTTP Request... | HTTP Request | API Fetch | ctx | If | Sends a request to the Meta Ads Insights endpoint. |
| If | If | Filter | HTTP Request... | Merge (True), Loop (False) | Checks if campaign data exists. |
| Merge | Merge | Data Join | ctx, If | Split campaigns | Merges client context with campaign data. |
| Split campaigns | Code | Transformation | Merge | Extract spreadsheetId | Splits API response into individual campaign items. |
| Extract spreadsheetId | Code | Transformation | Split campaigns | Append row in sheet | Parses report_sheet_url for spreadsheetId. |
| Append row in sheet | Google Sheets | Data Storage | Extract spreadsheetId | Code in JavaScript | Appends a row for each campaign. |
| Code in JavaScript | Code | Aggregator | Append row in sheet | Send a text message | Aggregates results and generates a Telegram message. |
| Send a text message | Telegram | Notification | Code in JavaScript | Loop Over Items | Sends a Telegram alert with the workflow summary. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Registry Setup:** Create a Master Google Sheet with columns: `client_name`, `ad_account_id`, `meta_access_token`, `report_sheet_url`, and `sheet_list` (the tab name).
2.  **Trigger:** Add a **Schedule Trigger** set to your preferred daily time.
3.  **Client Fetch:** Add a **Google Sheets: Get Many Rows** node. Select your master sheet.
4.  **Looping:** Add a **Loop Over Items** node (Split in Batches) with batch size 1.
5.  **Context Creation:** Add a **Code Node** (`ctx`) to wrap the current item's properties into a sub-object named `ctx`. This prevents property overwriting during the API call.
6.  **Meta API Call:** Add an **HTTP Request** node:
    *   Method: GET
    *   URL: `https://graph.facebook.com/v25.0/act_{{$json.ctx.ad_account_id}}/insights`
    *   Auth: Header `Authorization: Bearer {{$json.ctx.meta_access_token}}`
    *   Params: `date_preset: yesterday`, `level: campaign`, `fields: campaign_name,spend,impressions,clicks,ctr,cpm,cpc,date_start`.
7.  **Data Filtering:** Add an **If Node** to check if `data.length > 0`. Connect the "False" output back to the **Loop Over Items** node.
8.  **Merging:** Add a **Merge Node** (Mode: Combine by Position). Input 1: The `ctx` node; Input 2: The `If` (True) node.
9.  **Item Splitting:** Add a **Code Node** to loop through `data` (campaigns) and return one item per campaign, injecting `ctx` values (URL, ID) into every item.
10. **ID Extraction:** Add a **Code Node** to parse the `report_sheet_url` using regex to find the alphanumeric ID between `/d/` and the next `/`.
11. **Reporting:** Add a **Google Sheets: Append Row** node.
    *   Document ID: `{{$json.report_spreadsheet_id}}`
    *   Sheet Name: `{{$json.sheet_list}}`
    *   Mapping: Auto-map fields.
12. **Aggregation & Logic:** Add a **Code Node** to loop through all input items (all campaigns for that client), calculate totals, and build a string using template literals including the performance status (Weak/Good/OK).
13. **Alerting:** Add a **Telegram Node** using the aggregated string. Use your bot's credentials and a valid `chatId`.
14. **Close Loop:** Connect the Telegram node back to the **Loop Over Items** node input to process the next client.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| How to Generate Meta Tokens | [Meta Developers Portal](https://developers.facebook.com/) |
| Token Debugger | [Access Token Tool](https://developers.facebook.com/tools/debug/accesstoken/) |
| Status Logic | Weak: CTR < 1.5% \| Expensive: CPC > 0.5 \| Scale: CTR > 2% & CPC < 0.5 |
| API Version used | Meta Graph API v25.0 |