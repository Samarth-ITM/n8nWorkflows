Monitor Commercial Real Estate Opportunities from LoopNet with ScrapeGraphAI & Telegram

https://n8nworkflows.xyz/workflows/monitor-commercial-real-estate-opportunities-from-loopnet-with-scrapegraphai---telegram-6625


# Monitor Commercial Real Estate Opportunities from LoopNet with ScrapeGraphAI & Telegram

### 1. Workflow Overview

This workflow automates the daily monitoring and analysis of Commercial Real Estate (CRE) listings from LoopNet using ScrapeGraphAI and delivers actionable insights via Telegram alerts and Google Sheets logging. It is designed for real estate investors, analysts, and brokers who want to track market opportunities, pricing trends, and listings dynamically without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Automates daily execution to maintain fresh, consistent data.
- **1.2 Data Collection:** Uses AI-powered scraping (ScrapeGraphAI) to extract structured CRE listings data from LoopNet.
- **1.3 Data Analysis & Dashboard:** Processes scraped data to generate market intelligence, identify below-market opportunities, calculate averages, and create alerts.
- **1.4 Decision Logic:** Evaluates whether actionable opportunities exist to reduce unnecessary notifications.
- **1.5 Notification & Logging:** Sends Telegram alerts if opportunities are found and logs all daily metrics to Google Sheets for historical tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block initiates the entire workflow automatically every 24 hours, ensuring daily market data collection without manual input.

- **Nodes Involved:**  
  - 📋 Schedule Info (Sticky Note)  
  - Daily CRE Scanner (Schedule Trigger)

- **Node Details:**

  - **📋 Schedule Info**  
    - Type: Sticky Note  
    - Role: Documentation of scheduling logic and benefits.  
    - Configuration: Describes a 24-hour interval schedule, server default timezone, built-in error retry, and expected downstream actions.  
    - No input/output connections (informational only).

  - **Daily CRE Scanner**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow automatically every 24 hours.  
    - Configuration: Interval set to 24 hours, no specific start time or timezone configured (uses server default).  
    - Output Connections: Triggers → CRE Data Collector HTTP request node.  
    - Failure modes: Potential failure includes scheduler misconfiguration or server downtime. n8n handles retry internally.

---

#### 2.2 Data Collection

- **Overview:**  
  Retrieves commercial real estate listings from LoopNet using the ScrapeGraphAI API, which intelligently scrapes and returns structured data in JSON format.

- **Nodes Involved:**  
  - 📋 Data Collection (Sticky Note)  
  - CRE Data Collector (HTTP Request)

- **Node Details:**

  - **📋 Data Collection**  
    - Type: Sticky Note  
    - Role: Describes scraping logic, targeted data fields, API endpoint, and output format.  
    - Configuration: Highlights use of ScrapeGraphAI with JSON schema specifying listings array with property fields.  
    - No input/output connections (informational only).

  - **CRE Data Collector**  
    - Type: HTTP Request  
    - Role: Sends POST request to ScrapeGraphAI smart scraper endpoint to extract listings.  
    - Configuration:  
      - URL: `https://api.scrapegraphai.com/v1/smartscraper`  
      - Method: POST with JSON body including target URL (`https://www.loopnet.com/search/commercial-real-estate/`), extraction prompt, and output schema.  
      - Headers: Content-Type `application/json` and header-based authentication (credentials configured separately).  
      - Timeout: 60 seconds (default or implied).  
    - Inputs: Triggered by schedule node.  
    - Outputs: JSON containing structured listings data.  
    - Failure modes: Auth errors if header token invalid, network timeouts, API rate limits, or malformed responses.

---

#### 2.3 Data Analysis & Dashboard

- **Overview:**  
  Processes the scraped listings data to produce comprehensive market intelligence including summary statistics, opportunity detection, alerts, and a prioritized dashboard.

- **Nodes Involved:**  
  - 📋 Analysis Engine (Sticky Note)  
  - CRE Analyzer & Dashboard (Code Node)

- **Node Details:**

  - **📋 Analysis Engine**  
    - Type: Sticky Note  
    - Role: Explains analytical goals such as property type distribution, location ranking, average lease rate, and alert triggers.  
    - Configuration: Contains summary of KPIs and alert conditions.  
    - No input/output connections (informational only).

  - **CRE Analyzer & Dashboard**  
    - Type: Code Node (JavaScript)  
    - Role: Runs custom JS code to analyze the listings array.  
    - Configuration:  
      - Reads input JSON's `listings` array.  
      - Calculates:  
        - Total listings count.  
        - Average lease rate (excluding missing data).  
        - Frequency counts of property types and locations.  
        - Identifies below-market opportunities (lease_rate < 20).  
      - Generates alerts if:  
        - More than 10 opportunities found (high priority).  
        - Average lease rate exceeds $30/sqft (medium priority).  
      - Outputs structured dashboard JSON with summaries, top 5 property types & locations, top 10 opportunities sorted by rate, alerts, and recommended next actions.  
    - Inputs: Output from CRE Data Collector.  
    - Outputs: Dashboard JSON for downstream decision and logging.  
    - Failure modes: Code errors if input data malformed or missing fields; possible runtime exceptions in JavaScript environment.

---

#### 2.4 Decision Logic

- **Overview:**  
  Uses conditional logic to decide whether to send Telegram alerts based on presence of actionable opportunities, minimizing notification noise.

- **Nodes Involved:**  
  - 📋 Decision Logic (Sticky Note)  
  - Check for Opportunities (If Node)

- **Node Details:**

  - **📋 Decision Logic**  
    - Type: Sticky Note  
    - Role: Describes condition: send alert only if `opportunities_found > 0`.  
    - Configuration: Details advantages of conditional routing to save API calls and reduce communication noise.  
    - No input/output connections (informational only).

  - **Check for Opportunities**  
    - Type: If Node  
    - Role: Evaluates if `opportunities_found` in input JSON is greater than zero.  
    - Configuration:  
      - Condition: Numeric comparison, `{{ $json.market_summary.opportunities_found }} > 0`  
      - Case sensitive and strict validation enabled.  
    - Inputs: Dashboard JSON from CRE Analyzer & Dashboard.  
    - Outputs:  
      - TRUE branch → Send Opportunity Alert (Telegram node).  
      - FALSE branch → No action, but downstream logging continues.  
    - Failure modes: Expression failures if JSON path missing; logic errors if data structure changes.

---

#### 2.5 Notification & Logging

- **Overview:**  
  Sends a formatted Telegram message with key CRE market insights when opportunities exist; logs all daily metrics to Google Sheets for historical analysis.

- **Nodes Involved:**  
  - 📋 Telegram Setup (Sticky Note)  
  - Send Opportunity Alert (Telegram Node)  
  - 📋 Data Logging (Sticky Note)  
  - Log to Google Sheets (Google Sheets Node)

- **Node Details:**

  - **📋 Telegram Setup**  
    - Type: Sticky Note  
    - Role: Describes Telegram notification setup including bot token, chat ID, message format, and benefits.  
    - Configuration: Notes bot must be authorized and added to chat with proper permissions.  
    - No input/output connections (informational only).

  - **Send Opportunity Alert**  
    - Type: Telegram Node  
    - Role: Sends CRE daily report message to Telegram chat.  
    - Configuration:  
      - Operation: Send message.  
      - Uses preconfigured Telegram Bot credentials.  
      - Message content dynamically built upstream (likely via expressions or code, not explicitly shown in JSON).  
    - Inputs: TRUE output from Check for Opportunities node.  
    - Outputs: None downstream.  
    - Failure modes: Invalid bot token, chat ID, network issues, or Telegram API errors.

  - **📋 Data Logging**  
    - Type: Sticky Note  
    - Role: Explains logging of daily CRE metrics to Google Sheets including required sheet structure.  
    - Configuration: Lists columns logged: date, total properties, average lease rate, opportunity count, top location, alert count.  
    - No input/output connections (informational only).

  - **Log to Google Sheets**  
    - Type: Google Sheets Node  
    - Role: Appends or updates a row in a specified Google Sheet with daily CRE analysis metrics.  
    - Configuration:  
      - Resource: Spreadsheet  
      - Operation: Append or Update  
      - Requires Google Sheets document ID, sheet named 'CRE_Analysis', and service account credentials.  
    - Inputs: Always runs after CRE Analyzer & Dashboard (parallel with decision logic).  
    - Outputs: None downstream.  
    - Failure modes: Auth errors, missing sheet or columns, API limits.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                  | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                  |
|------------------------|-----------------------|---------------------------------|------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| 📋 Schedule Info        | Sticky Note           | Scheduling documentation         | None                   | None                          | Describes daily 24h schedule, benefits, and pipeline trigger                                |
| Daily CRE Scanner       | Schedule Trigger      | Triggers workflow every 24h      | None                   | CRE Data Collector             |                                                                                              |
| 📋 Data Collection      | Sticky Note           | Describes scraping setup         | None                   | None                          | Details ScrapeGraphAI usage, target data, and API config                                    |
| CRE Data Collector      | HTTP Request          | Scrapes CRE listings from LoopNet| Daily CRE Scanner      | CRE Analyzer & Dashboard       |                                                                                              |
| 📋 Analysis Engine      | Sticky Note           | Describes analysis & KPIs        | None                   | None                          | Explains analysis metrics, alerts, and dashboard output                                    |
| CRE Analyzer & Dashboard| Code Node             | Analyzes listings, creates dashboard| CRE Data Collector    | Check for Opportunities, Log to Google Sheets |                                                                                              |
| 📋 Decision Logic       | Sticky Note           | Conditional logic for alerts     | None                   | None                          | Explains flow to send alerts only if opportunities exist                                   |
| Check for Opportunities | If Node               | Checks if opportunities exist    | CRE Analyzer & Dashboard| Send Opportunity Alert (TRUE)  |                                                                                              |
| 📋 Telegram Setup       | Sticky Note           | Telegram message setup info      | None                   | None                          | Describes Telegram bot setup and message structure                                         |
| Send Opportunity Alert  | Telegram Node         | Sends CRE alert to Telegram      | Check for Opportunities (TRUE) | None                    |                                                                                              |
| 📋 Data Logging         | Sticky Note           | Describes Google Sheets logging  | None                   | None                          | Explains historical data logging setup and benefits                                        |
| Log to Google Sheets    | Google Sheets Node    | Logs daily metrics to Google Sheets| CRE Analyzer & Dashboard| None                          |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Schedule Trigger node:**  
   - Name: `Daily CRE Scanner`  
   - Type: Schedule Trigger  
   - Set interval to every 24 hours (hoursInterval = 24).

3. **Add HTTP Request node:**  
   - Name: `CRE Data Collector`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.scrapegraphai.com/v1/smartscraper`  
   - Headers:  
     - Content-Type: application/json  
     - Authorization: Set via credentials (create/use Header Auth credential with API token for ScrapeGraphAI).  
   - Body Parameters (JSON):  
     ```json
     {
       "website_url": "https://www.loopnet.com/search/commercial-real-estate/",
       "user_prompt": "Extract commercial real estate listings: property type, size, location, price, lease rate, contact info",
       "output_schema": "{\"listings\": [{\"id\": \"string\", \"type\": \"string\", \"size_sqft\": \"number\", \"location\": \"string\", \"price\": \"number\", \"lease_rate\": \"number\", \"contact\": \"string\"}]}"
     }
     ```  
   - Enable sending body and headers.

4. **Connect Schedule Trigger output to HTTP Request input.**

5. **Add Code node:**  
   - Name: `CRE Analyzer & Dashboard`  
   - Type: Code (JavaScript)  
   - Paste the provided script that:  
     - Reads `listings` array from input JSON.  
     - Calculates total listings, average lease rate, property types count, top locations count.  
     - Identifies below-market opportunities (lease_rate < 20).  
     - Generates alerts if opportunities > 10 or avg lease rate > 30.  
     - Creates a dashboard JSON output with summaries, top 5 property types and locations, top 10 opportunities, alerts, and next actions.  
   - Connect HTTP Request output to this Code node input.

6. **Add If node:**  
   - Name: `Check for Opportunities`  
   - Condition: Number comparison  
   - Expression: `{{ $json.market_summary.opportunities_found }} > 0`  
   - Strict validation enabled.  
   - Connect Code node output to this If node.

7. **Add Telegram node:**  
   - Name: `Send Opportunity Alert`  
   - Type: Telegram  
   - Operation: Send message  
   - Configure credentials with your Telegram Bot Token (created via @BotFather).  
   - Set Chat ID (your Telegram user or group chat ID).  
   - Construct message text (via expressions or a preceding Function/Set node if needed) to include market summary, top opportunities, and alerts.  
   - Connect TRUE output of If node to this Telegram node.

8. **Add Google Sheets node:**  
   - Name: `Log to Google Sheets`  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Configure with service account credentials authorized to your Google Sheets document.  
   - Specify spreadsheet ID and sheet name (e.g., `CRE_Analysis`).  
   - Map columns for date, total properties, average lease rate, opportunities count, top location, alert count.  
   - Connect Code node output directly (independent of If condition) to this node.

9. **Set up Sticky Notes for documentation:**  
   - Add five sticky notes with the exact content as in the workflow for:  
     - Schedule Info  
     - Data Collection  
     - Analysis Engine  
     - Decision Logic  
     - Telegram Setup  
     - Data Logging

10. **Validate all credential setups:**  
    - ScrapeGraphAI header authentication token.  
    - Telegram Bot token and chat ID.  
    - Google Sheets service account with access to target spreadsheet.

11. **Activate the workflow and monitor initial runs for errors or data validation.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses ScrapeGraphAI's smart scraper API for dynamic content extraction from LoopNet commercial real estate listings. | ScrapeGraphAI API: https://api.scrapegraphai.com/v1/smartscraper                               |
| Telegram Bot integration requires a bot created via @BotFather and added to desired chats with proper permissions.               | Telegram Bot API docs: https://core.telegram.org/bots/api                                      |
| Google Sheets logging enables historical market trend analysis and requires service account authentication with API access.      | Google Sheets API docs: https://developers.google.com/sheets/api                               |
| The code node uses JavaScript and expects listings data structured as per the output schema; modifications require JS knowledge.  | n8n JavaScript Code Node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.code          |
| Scheduling uses server default timezone; adjust if necessary via n8n or server settings to align with local business hours.      | n8n Scheduling docs: https://docs.n8n.io/nodes/n8n-nodes-base.schedule-trigger/                 |

---

This completes the detailed analysis and documentation of the "Monitor Commercial Real Estate Opportunities from LoopNet with ScrapeGraphAI & Telegram" workflow. It is fully described to enable reproduction, modification, and integration error anticipation.