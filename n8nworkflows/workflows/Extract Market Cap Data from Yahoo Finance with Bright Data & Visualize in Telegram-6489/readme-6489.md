Extract Market Cap Data from Yahoo Finance with Bright Data & Visualize in Telegram

https://n8nworkflows.xyz/workflows/extract-market-cap-data-from-yahoo-finance-with-bright-data---visualize-in-telegram-6489


# Extract Market Cap Data from Yahoo Finance with Bright Data & Visualize in Telegram

### 1. Workflow Overview

This workflow automates the extraction of market capitalization and related financial data from Yahoo Finance for user-provided keywords, using Bright Data’s API for web scraping, and then visualizes the results by sending a market cap chart to a Telegram chat. It is designed for financial analysts, investors, or businesses monitoring market data in real-time with minimal manual intervention.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & Data Scraping Trigger:** Captures user input keyword(s) via a web form and initiates a data extraction job on Bright Data’s platform.

- **1.2 Monitoring & Fetching Scraped Data:** Waits for the scraping job to complete, monitors the job status, and retrieves the final data snapshot once ready.

- **1.3 Data Processing, Storage & Visualization:** Filters and saves the scraped data to Google Sheets, generates a market capitalization bar chart using QuickChart API, and sends the resulting image to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Scraping Trigger

- **Overview:**  
  This block captures a keyword input from a user through a form trigger, then calls Bright Data’s API to start a new dataset discovery job using that keyword.

- **Nodes Involved:**  
  - 🟩 Form Trigger  
  - 🚀 Trigger Scraping1 (HTTP Request to Bright Data)  
  - Sticky Note (explains this block)

- **Node Details:**

  - **🟩 Form Trigger**  
    - **Type:** Form Trigger (Webhook-based)  
    - **Role:** Receives a keyword input from a user via a web form titled "Yahoo".  
    - **Configuration:** Single form field named "keyword".  
    - **Key Expression:** Uses `{{$json.keyword}}` to pass input to next nodes.  
    - **Input:** Webhook receives HTTP POST on form submission.  
    - **Output:** Triggers next node with JSON containing the keyword.  
    - **Edge Cases:** Missing or empty keyword input may cause no meaningful scraping; no explicit validation shown.  
    - **Version:** 2.2  
  
  - **🚀 Trigger Scraping1 (HTTP Request to Bright Data)**  
    - **Type:** HTTP Request  
    - **Role:** Sends POST request to Bright Data API to trigger a new scraping job with the keyword.  
    - **Configuration:**  
      - URL: `https://api.brightdata.com/datasets/v3/trigger`  
      - Method: POST  
      - Body: JSON array with keyword from form input  
      - Query Parameters:  
        - `dataset_id=gd_lmrpz3vxmz972ghd7` (specific dataset for market data)  
        - `include_errors=true`  
        - `type=discover_new`  
        - `discover_by=keyword`  
        - `limit_per_input=2` (limits results per keyword)  
      - Headers: Authorization bearer token using environment variable `BRIGHT_DATA_API_KEY`  
    - **Input:** Receives JSON with keyword from Form Trigger.  
    - **Output:** Returns scraping job snapshot ID and status information.  
    - **Edge Cases:** Authorization failure, network timeout, invalid keyword format, or Bright Data API errors.  
    - **Version:** 4.2  
  
  - **Sticky Note**  
    - Explains the user input and API call process clearly, aiding maintainability.

---

#### 2.2 Monitoring & Fetching Scraped Data

- **Overview:**  
  This block waits for Bright Data scraping job completion by polling the job status every minute, and once ready, fetches the final snapshot data.

- **Nodes Involved:**  
  - 🕐 Wait 1 minute1  
  - 🟡 Check Delivery Status of Snap ID1 (HTTP Request)  
  - 🟢 Check Final Status (IF Node)  
  - 🔽 final data given (Snapshot Fetch) (HTTP Request)  
  - Sticky Note1 (explains this block)

- **Node Details:**

  - **🟡 Check Delivery Status of Snap ID1**  
    - **Type:** HTTP Request  
    - **Role:** Checks current status of the scraping snapshot job using Bright Data API.  
    - **Configuration:**  
      - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
      - Headers: Uses `Authorization: Bearer BRIGHT_DATA_API_KEY`  
      - Method: GET (default)  
    - **Input:** Snapshot ID from previous Trigger Scraping node.  
    - **Output:** Returns JSON with job progress and status.  
    - **Edge Cases:** API downtime, invalid snapshot_id, expired job, or auth errors.  
  
  - **🕐 Wait 1 minute1**  
    - **Type:** Wait  
    - **Role:** Delays workflow execution by 1 minute between status polls to avoid API spamming.  
    - **Configuration:** Wait for 1 minute.  
    - **Input:** Triggered by status check node when job not ready.  
    - **Output:** Triggers next status check.  
    - **Edge Cases:** If job takes too long, potential timeout or workflow max execution time exceeded.  
  
  - **🟢 Check Final Status**  
    - **Type:** IF Node  
    - **Role:** Branches logic based on whether `status == "ready"` from status check.  
    - **Configuration:** Checks if `$json.status` equals `"ready"`.  
    - **Input:** Status JSON from status check node.  
    - **Output:**  
      - If `ready`: proceeds to snapshot fetch.  
      - Else: loops back to status check (polling).  
    - **Edge Cases:** Unexpected status values, null status, or JSON parsing errors.  
  
  - **🔽 final data given (Snapshot Fetch)**  
    - **Type:** HTTP Request  
    - **Role:** Retrieves the final dataset snapshot in JSON format once scraping job is complete.  
    - **Configuration:**  
      - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
      - Query Parameter: `format=json`  
      - Headers: Auth bearer token  
    - **Input:** Snapshot ID from status check node when ready.  
    - **Output:** Full scraped result JSON with financial data.  
    - **Edge Cases:** Data not available, API errors, or invalid snapshot ID.  
  
  - **Sticky Note1**  
    - Summarizes the monitoring and data fetch logic for clarity and future maintenance.

---

#### 2.3 Data Processing, Storage & Visualization

- **Overview:**  
  Processes the retrieved financial data, stores it in a Google Sheet, generates a bar chart reflecting market caps, converts it to PNG via QuickChart, and sends the chart image to a Telegram chat.

- **Nodes Involved:**  
  - 📊 Filtered Output & Save to Sheet (Google Sheets)  
  - 🧮 Generate Chart Payload (Code)  
  - 🌐 Generate PNG from Chart (HTTP Request)  
  - 📤 Send Chart on Telegram (Telegram Node)  
  - Sticky Note2  
  - Sticky Note3  

- **Node Details:**

  - **📊 Filtered Output & Save to Sheet**  
    - **Type:** Google Sheets  
    - **Role:** Appends the scraped and filtered financial data to a specified Google Sheet.  
    - **Configuration:**  
      - Operation: Append  
      - Document: Google Sheet identified by URL parameter (replace `YOUR_SHEET_ID` with actual ID)  
      - Sheet Name: Defaults to first sheet (`gid=0`)  
      - Columns: Maps many fields from JSON (e.g., name, market_cap, eps, volume, exchange, etc.)  
      - Mapping Mode: Define below (explicit column mappings)  
    - **Input:** JSON array of scraped financial records.  
    - **Output:** Passes same data to chart generation node.  
    - **Edge Cases:** Credential errors, quota limits, malformed data, or Google API downtime.  
  
  - **🧮 Generate Chart Payload**  
    - **Type:** Code (JavaScript)  
    - **Role:** Converts market cap strings (e.g., "1.2B", "3.4T") into numerical values in billions USD and constructs a QuickChart-compatible bar chart configuration JSON.  
    - **Key Logic:**  
      - Parses market caps with suffixes T (trillions), B (billions), M (millions) into numbers.  
      - Prepares horizontal bar chart with labels as company names and values as market cap in billions.  
      - Styles chart with colors, title, and tooltips.  
    - **Input:** Data from Google Sheets node.  
    - **Output:** JSON object with chart configuration.  
    - **Edge Cases:** Market cap missing or malformed; fallback to zero; potential parsing errors.  
  
  - **🌐 Generate PNG from Chart**  
    - **Type:** HTTP Request  
    - **Role:** Calls QuickChart.io API to generate a PNG image from the chart JSON.  
    - **Configuration:**  
      - URL: `https://quickchart.io/chart?c={{ JSON.stringify($json.chart) }}`  
      - Method: GET (default)  
    - **Input:** Chart JSON from Code node.  
    - **Output:** Binary PNG image data.  
    - **Edge Cases:** API downtime, large chart payloads, or network issues.  
  
  - **📤 Send Chart on Telegram**  
    - **Type:** Telegram  
    - **Role:** Sends the generated market cap chart image to a configured Telegram chat.  
    - **Configuration:**  
      - Chat ID: Replace `YOUR_TELEGRAM_CHAT_ID` with target chat or group ID.  
      - Operation: sendPhoto  
      - Uses binary data from previous node for photo upload.  
    - **Input:** PNG image from QuickChart node.  
    - **Output:** Sends message to Telegram; no further output.  
    - **Edge Cases:** Invalid chat ID, Telegram API limits, missing credentials, or blocked bot.  
  
  - **Sticky Note2 & Sticky Note3**  
    - Provide explanations of the data filtering, output saving, chart generation, and Telegram sending steps for clarity.

---

### 3. Summary Table

| Node Name                            | Node Type           | Functional Role                         | Input Node(s)                          | Output Node(s)                      | Sticky Note                                                                                                   |
|------------------------------------|---------------------|---------------------------------------|--------------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| 🟩 Form Trigger                    | Form Trigger        | Captures user keyword input           | Webhook (external user)               | 🚀 Trigger Scraping1               | 🟩 Form Trigger (On form submission1) User form se keyword deta hai (e.g. "AI", "Crypto", "MSFT") Bright Data API ko call karta hai |
| 🚀 Trigger Scraping1 (HTTP Request to Bright Data) | HTTP Request        | Initiates Bright Data scraping job    | 🟩 Form Trigger                      | 🟡 Check Delivery Status of Snap ID1 |                                                                                                              |
| 🟡 Check Delivery Status of Snap ID1 | HTTP Request        | Polls job status from Bright Data API | 🚀 Trigger Scraping1                 | 🕐 Wait 1 minute1, 🟢 Check Final Status |                                                                                                              |
| 🕐 Wait 1 minute1                  | Wait                | Delays workflow 1 minute for polling  | 🟡 Check Delivery Status of Snap ID1 | 🟢 Check Final Status              |                                                                                                              |
| 🟢 Check Final Status              | IF                  | Checks if job status is "ready"       | 🕐 Wait 1 minute1, 🟡 Check Delivery Status | 🔽 final data given, 🟡 Check Delivery Status of Snap ID1 |                                                                                                              |
| 🔽 final data given (Snapshot Fetch) | HTTP Request        | Fetches final scraped data snapshot   | 🟢 Check Final Status (ready branch) | 📊 Filtered Output & Save to Sheet |                                                                                                              |
| 📊 Filtered Output & Save to Sheet | Google Sheets       | Saves filtered financial data to sheet| 🔽 final data given                  | 🧮 Generate Chart Payload          | 📊 Zone 3: Filter & Output Includes Optional IF Node, Code Node, Google Sheets Node                           |
| 🧮 Generate Chart Payload          | Code                | Creates QuickChart bar chart payload  | 📊 Filtered Output & Save to Sheet   | 🌐 Generate PNG from Chart         | 1.🧮 Generate Chart Payload (Code Node) Prepare chart data for market cap visualization                      |
| 🌐 Generate PNG from Chart         | HTTP Request        | Converts chart JSON to PNG image      | 🧮 Generate Chart Payload            | 📤 Send Chart on Telegram          | 2.🌐 Generate PNG from Chart (HTTP Request Node) Uses QuickChart.io API                                      |
| 📤 Send Chart on Telegram          | Telegram            | Sends chart image to Telegram chat    | 🌐 Generate PNG from Chart           | None                             | 3.📤 Send Chart on Telegram (Telegram Node) Sends chart to client/group                                      |
| Sticky Note                       | Sticky Note         | Explanation of the first block         | None                               | None                            | 🟩 Form Trigger (On form submission1) User input and Bright Data API call                                   |
| Sticky Note1                     | Sticky Note         | Explanation of the monitoring block    | None                               | None                            | ⏳ Zone 2: Monitor & Fetch Results: Wait, Status Check, IF, Snapshot API                                     |
| Sticky Note2                     | Sticky Note         | Explanation of the data output & save block | None                               | None                            | 📊 Zone 3: Filter & Output Includes IF Node, Code Node, Google Sheets Node                                  |
| Sticky Note3                     | Sticky Note         | Explanation of chart generation & Telegram | None                               | None                            | 1.🧮 Generate Chart Payload; 2.🌐 Generate PNG from Chart; 3.📤 Send Chart on Telegram                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "🟩 Form Trigger" node:**
   - Type: Form Trigger  
   - Configure form title as "Yahoo"  
   - Add one form field with label "keyword" (string input)  
   - Save webhook URL for external triggering  
   - Output will provide JSON with `keyword` field  

2. **Create "🚀 Trigger Scraping1 (HTTP Request to Bright Data)" node:**
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Add Query Parameters:  
     - `dataset_id` = `gd_lmrpz3vxmz972ghd7`  
     - `include_errors` = `true`  
     - `type` = `discover_new`  
     - `discover_by` = `keyword`  
     - `limit_per_input` = `2`  
   - Body Type: JSON  
   - Body Content: `[{"keyword": "{{ $json.keyword }}"}]` (use expression)  
   - Headers:  
     - `Authorization: Bearer BRIGHT_DATA_API_KEY` (configure credential or environment variable)  
   - Connect input from Form Trigger node output  

3. **Create "🟡 Check Delivery Status of Snap ID1" node:**
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}` (expression)  
   - Headers:  
     - `Authorization: Bearer BRIGHT_DATA_API_KEY`  
   - Connect input from "🚀 Trigger Scraping1" output  

4. **Create "🕐 Wait 1 minute1" node:**
   - Type: Wait  
   - Wait for 1 minute  
   - Connect input from "🟡 Check Delivery Status of Snap ID1" for retry loop  

5. **Create "🟢 Check Final Status" node:**
   - Type: IF  
   - Condition: `$json.status` equals `"ready"`  
   - Connect "true" output to next node (snapshot fetch)  
   - Connect "false" output back to "🟡 Check Delivery Status of Snap ID1" (loop)  
   - Connect input from "🕐 Wait 1 minute1" and also from "🟡 Check Delivery Status of Snap ID1"  

6. **Create "🔽 final data given (Snapshot Fetch)" node:**
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query Parameter: `format=json`  
   - Headers:  
     - `Authorization: Bearer BRIGHT_DATA_API_KEY`  
   - Connect input from "🟢 Check Final Status" true output  

7. **Create "📊 Filtered Output & Save to Sheet" node:**
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Google Sheet URL or ID (replace placeholder with actual sheet ID)  
   - Sheet Name: `Sheet1` or `gid=0` as per your sheet  
   - Map columns manually to fields provided from JSON (e.g., name, market_cap, eps, volume, etc.)  
   - Connect input from "🔽 final data given" output  
   - Configure Google Sheets credentials with appropriate access  

8. **Create "🧮 Generate Chart Payload" node:**
   - Type: Code (JavaScript)  
   - Paste the provided JS code which converts market cap strings to numeric values and builds chart JSON payload  
   - Connect input from Google Sheets node output  

9. **Create "🌐 Generate PNG from Chart" node:**
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://quickchart.io/chart?c={{ JSON.stringify($json.chart) }}` (expression)  
   - Connect input from Code node output  

10. **Create "📤 Send Chart on Telegram" node:**
    - Type: Telegram  
    - Operation: sendPhoto  
    - Chat ID: Replace with your Telegram chat ID or group ID  
    - Set binary data input to receive the PNG image from previous node  
    - Connect input from HTTP Request node output  
    - Configure Telegram credentials (bot token)  

11. **Connect all nodes following the logical flow:**
    - Form Trigger → Trigger Scraping → Check Delivery Status → Wait 1 min → Check Final Status IF  
    - IF true → Snapshot Fetch → Google Sheets → Generate Chart → Generate PNG → Telegram  
    - IF false → Loop back to Check Delivery Status  

12. **Add Sticky Notes (optional) with explanatory text at relevant places for maintainability.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                        | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Bright Data API requires an API key set as environment variable `BRIGHT_DATA_API_KEY` or configured credential in n8n.                                                                                                                             | Bright Data API documentation for authentication and dataset usage.                                |
| Replace `YOUR_SHEET_ID` in Google Sheets node with your actual Google Sheets document ID. Ensure the Google Sheets credentials have write access to this document.                                                                                  | Google Sheets API and credential setup instructions.                                               |
| Replace `YOUR_TELEGRAM_CHAT_ID` with the target Telegram chat or group ID. The Telegram bot must be added to the chat and granted permission to send messages/photos.                                                                               | Telegram Bot API documentation for chat IDs and permissions.                                      |
| QuickChart.io is a free chart image generation API used here to convert chart config JSON to PNG. The URL length and payload size must be kept reasonably small to avoid request failure.                                                           | https://quickchart.io/documentation/                                                               |
| The workflow uses polling with a fixed 1-minute interval; consider adjusting or adding a maximum retry count/timeout to avoid infinite loops in production.                                                                                         | Best practices for API polling and rate limiting.                                                 |
| Market cap parsing in code node supports suffixes T (trillion), B (billion), M (million); unexpected formats may result in zero or NaN values.                                                                                                       | Data validation or enhancement could improve robustness.                                          |
| The workflow assumes stable connectivity to all external APIs (Bright Data, Google Sheets, QuickChart, Telegram); network interruptions can cause failures requiring error handling or retries.                                                     | Consider adding error workflow branches or retry nodes for robustness.                            |

---

**Disclaimer:** The provided text is exclusively extracted from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.