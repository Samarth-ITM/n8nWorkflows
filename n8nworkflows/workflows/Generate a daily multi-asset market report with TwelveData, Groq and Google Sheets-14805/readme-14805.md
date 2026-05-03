Generate a daily multi-asset market report with TwelveData, Groq and Google Sheets

https://n8nworkflows.xyz/workflows/generate-a-daily-multi-asset-market-report-with-twelvedata--groq-and-google-sheets-14805


# Generate a daily multi-asset market report with TwelveData, Groq and Google Sheets

# 1. Workflow Overview

This workflow generates a daily multi-asset market snapshot by collecting latest market data from TwelveData, normalizing the results, sending the normalized dataset to a Groq-hosted LLM for analysis, parsing the AI response into structured fields, logging the final report to Google Sheets, and emailing the summary through Gmail.

Its primary use cases are:
- Daily internal market monitoring
- Automated pre-market or post-market multi-asset reporting
- Lightweight AI-assisted macro commentary generation
- Audit-friendly logging of both successful reports and API failures

The workflow is organized into the following logical blocks:

## 1.1 Trigger & Environment Setup
The workflow starts manually and initializes basic configuration values, especially the TwelveData endpoint and API key placeholder.

## 1.2 Asset Definition & Symbol Expansion
Asset groups are defined as comma-separated lists, then expanded into one item per symbol so each instrument can be fetched individually.

## 1.3 Rate-Controlled Market Data Fetching
Symbols are processed one at a time through a batching loop with a wait node to reduce API pressure. Each symbol is requested from TwelveData and validated.

## 1.4 API Error Logging & Recovery Loop
Failed API responses are converted into log rows, appended to a Google Sheet, and an email alert is sent. The flow then returns to the batch loop so processing can continue with remaining symbols.

## 1.5 Market Data Normalization
Successful responses collected across batch iterations are aggregated and transformed into a normalized `market_data` array containing prices, percent changes, trend labels, and metadata.

## 1.6 AI Analysis & Structured Parsing
The normalized market data is passed to a LangChain Agent node using a Groq chat model. The generated text is then parsed into structured report fields.

## 1.7 Report Persistence & Email Delivery
The parsed report is appended to Google Sheets and then sent as a plain-text email.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger & Environment Setup

**Overview:**  
This block starts the workflow and provides configuration values required later in the execution. It is the foundation for API access and downstream symbol processing.

**Nodes Involved:**  
- Generate Daily Market Report
- Environment Config

### Node: Generate Daily Market Report
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for the workflow.
- **Configuration choices:**  
  No custom parameters; execution starts when manually run.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: Environment Config
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None significant; it simply requires a manual run.
- **Sub-workflow reference:**  
  None.

### Node: Environment Config
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines reusable configuration values.
- **Configuration choices:**  
  Assigns:
  - `api_base_url` = `https://api.twelvedata.com/time_series`
  - `twelve_api_key` = empty string placeholder
  `includeOtherFields` is enabled, so incoming fields are preserved.
- **Key expressions or variables used:**  
  Later referenced by:
  - `$('Environment Config').first().json.api_base_url`
  - `$('Environment Config').first().json.twelve_api_key`
- **Input and output connections:**  
  - Input: Generate Daily Market Report
  - Output: Set Market Assets
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If `twelve_api_key` remains empty, TwelveData requests will fail.
  - If users replace the base URL incorrectly, all HTTP calls may fail.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Asset Definition & Symbol Expansion

**Overview:**  
This block defines which instruments should be tracked and expands grouped comma-separated lists into one execution item per symbol. It prepares the data structure required for batched API requests.

**Nodes Involved:**  
- Set Market Assets
- split assests & Symbols

### Node: Set Market Assets
- **Type and technical role:** `n8n-nodes-base.set`  
  Stores the instrument groups to be monitored.
- **Configuration choices:**  
  Defines three string fields:
  - `Indices` = `SPY,QQQ,DIA`
  - `Forex` = `EUR/USD,USD/INR,GBP/USD,USD/JPY`
  - `Commodities` = `XAU/USD,USO`
  Incoming fields are preserved.
- **Key expressions or variables used:**  
  Consumed by the next Code node as `items[0].json`.
- **Input and output connections:**  
  - Input: Environment Config
  - Output: split assests & Symbols
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Symbol formatting matters; invalid or unsupported TwelveData symbols will produce API errors.
  - The node name contains the canonical place where users customize the coverage universe.
- **Sub-workflow reference:**  
  None.

### Node: split assests & Symbols
- **Type and technical role:** `n8n-nodes-base.code`  
  Splits grouped asset strings into individual symbol items and tags them with asset class metadata.
- **Configuration choices:**  
  The code:
  - Reads `Indices`, `Forex`, and `Commodities`
  - Splits each group on commas
  - Emits one item per symbol with:
    - `asset_class`
    - `symbol`
- **Key expressions or variables used:**  
  Uses `items[0].json` as source input.  
  Outputs fields like:
  - `json.asset_class`
  - `json.symbol`
- **Input and output connections:**  
  - Input: Set Market Assets
  - Output: Rate Controlled Queue
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If any asset group field is missing or not a string, `.split(',')` will fail.
  - Extra commas may create empty symbols unless cleaned upstream.
  - The node name contains a typo ("assests"), but this does not affect execution.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Rate-Controlled Market Data Fetching

**Overview:**  
This block processes symbols one at a time, enforces a delay between API calls, fetches daily price data from TwelveData, and validates the response. It forms the workflow’s main collection loop.

**Nodes Involved:**  
- Rate Controlled Queue
- API Throttle
- Fetch Asset Prices
- Validate API Response

### Node: Rate Controlled Queue
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through one symbol at a time and controls the loop.
- **Configuration choices:**  
  - `batchSize` = `1`
  This ensures strictly sequential processing.
- **Key expressions or variables used:**  
  `={{ 1 }}`
- **Input and output connections:**  
  - Inputs:
    - split assests & Symbols
    - Validate API Response (success path continuation)
    - Alert API Faliure (failure path continuation)
  - Outputs:
    - Normalize Market Data
    - API Throttle
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - Split in Batches looping can be confusing to maintain if branches are modified.
  - Because Normalize Market Data is connected directly from this node, it is triggered during loop progression and relies on cross-run item access.
- **Sub-workflow reference:**  
  None.

### Node: API Throttle
- **Type and technical role:** `n8n-nodes-base.wait`  
  Introduces a fixed pause before each API call.
- **Configuration choices:**  
  - `amount` = `15`
  This appears to be a 15-unit wait based on node defaults; in practice this should be verified in the UI because Wait node time unit settings matter.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: Rate Controlled Queue
  - Output: Fetch Asset Prices
- **Version-specific requirements:**  
  Type version `1.1`.
- **Edge cases or potential failure types:**  
  - If the wait unit is not what the designer expects, throttling may be too short or too long.
  - Long waits can slow overall execution significantly.
- **Sub-workflow reference:**  
  None.

### Node: Fetch Asset Prices
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls TwelveData’s time series endpoint for one symbol.
- **Configuration choices:**  
  - URL comes from Environment Config
  - Query parameters:
    - `symbol` = current item symbol
    - `interval` = `1day`
    - `outputsize` = `1`
    - `apikey` = TwelveData API key from config
- **Key expressions or variables used:**  
  - `={{ $('Environment Config').first().json.api_base_url }}`
  - `={{ $json.symbol }}`
  - `={{ $('Environment Config').first().json.twelve_api_key }}`
- **Input and output connections:**  
  - Input: API Throttle
  - Output: Validate API Response
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Rate limiting
  - Unsupported symbols
  - Network timeout
  - API schema changes
  - Forex symbol formatting may depend on TwelveData’s accepted notation
- **Sub-workflow reference:**  
  None.

### Node: Validate API Response
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes successful and failed API responses.
- **Configuration choices:**  
  Condition checks whether:
  - `$json.status` equals `ok`
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: Fetch Asset Prices
  - Outputs:
    - True branch: Rate Controlled Queue
    - False branch: Log API Error
- **Version-specific requirements:**  
  Type version `2.3`.
- **Edge cases or potential failure types:**  
  - If the API returns success data without `status: ok`, valid payloads may be treated as failures.
  - If the HTTP node is configured to throw on HTTP errors in some environments, this IF node may never receive the response.
- **Sub-workflow reference:**  
  None.

---

## 2.4 API Error Logging & Recovery Loop

**Overview:**  
This block handles API failures by converting the failed payload into a compact log format, storing it in a Google Sheet, emailing an alert, and then returning control to the batch loop.

**Nodes Involved:**  
- Log API Error
- Append row in sheet
- Alert API Faliure

### Node: Log API Error
- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms failed API responses into a standard error record.
- **Configuration choices:**  
  Creates:
  - `error_time`
  - `error_status`
  - `error_message`
  - `failed_symbol`
- **Key expressions or variables used:**  
  - `$json.status`
  - `$json.message`
  - `$json.meta?.symbol`
- **Input and output connections:**  
  - Input: Validate API Response (false branch)
  - Output: Append row in sheet
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If the error payload shape differs, fields may fall back to defaults like `Unknown API error` or `N/A`.
- **Sub-workflow reference:**  
  None.

### Node: Append row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends failure information to an error log worksheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `Multi Asset Daily Snapshot`
  - Sheet/tab: `Error logs`
  - Writes:
    - `execution_date`
    - `error_message`
    - `failed_symbol`
    - `status` = `FAILED`
    - `error_code`
    - `workflow_version` = `v2.0-production`
- **Key expressions or variables used:**  
  Includes:
  - `={{ 'FAILED' }}`
  - `={{ $json.code || 'N/A' }}`
  - `={{ $json.message || 'Unknown API error' }}`
  - `={{ $json.meta?.symbol || $json.symbol || 'N/A' }}`
  - `={{ $now }}`
- **Input and output connections:**  
  - Input: Log API Error
  - Output: Alert API Faliure
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - OAuth permission problems
  - Spreadsheet or sheet not found
  - Column mismatches if the sheet schema changes
  - Note that this node expects raw error fields, but it receives the transformed output from Log API Error; as a result, `error_code`, `message`, or `meta.symbol` may not always exist as expected. This is a mild field mapping inconsistency.
- **Sub-workflow reference:**  
  None.

### Node: Alert API Faliure
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an immediate email when a market data API request fails.
- **Configuration choices:**  
  - Plain text email
  - Subject: `API Failure Alert`
  - Message includes status and message fields
- **Key expressions or variables used:**  
  - `{{ $json.status }}`
  - `{{ $json.message }}`
- **Input and output connections:**  
  - Input: Append row in sheet
  - Output: Rate Controlled Queue
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth expiration
  - Because upstream transformed fields are named `error_status` and `error_message`, this node’s references to `$json.status` and `$json.message` may render blank unless the incoming payload still contains those fields. This is another data-shape inconsistency.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Market Data Normalization

**Overview:**  
This block aggregates all successful fetches across loop iterations and converts them into a consistent array used by the AI analysis stage. It also gracefully records incomplete data as error-like entries inside the normalized array.

**Nodes Involved:**  
- Normalize Market Data

### Node: Normalize Market Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Aggregates outputs from multiple executions of the fetch node and computes derived metrics.
- **Configuration choices:**  
  The code:
  - Iterates through all execution runs of `Fetch Asset Prices` using `$items("Fetch Asset Prices", 0, runIndex)`
  - Extracts latest candle from `values[0]`
  - Computes:
    - `price`
    - `open`
    - `high`
    - `low`
    - `date`
    - `change_percent`
    - `trend` = `Bullish` or `Bearish`
    - `status`
  - If no latest data exists, pushes an error-style object instead of dropping the symbol
  - Returns one item:
    - `market_data: [...]`
- **Key expressions or variables used:**  
  - `$items("Fetch Asset Prices", 0, runIndex)`
  - `data.values?.[0]`
  - `data.meta.symbol`
  - `data.meta.type`
- **Input and output connections:**  
  - Input: Rate Controlled Queue
  - Output: AI Market Insights
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Relies on execution-run indexing behavior; this makes the design powerful but somewhat brittle for future edits.
  - If TwelveData changes response shape, parsing will break.
  - If `open` is zero or invalid, change percent calculation can produce `Infinity` or `NaN`.
  - Flat moves where `changePercent === 0` are labeled `Bearish`, because the code only checks `> 0`; there is no neutral branch.
- **Sub-workflow reference:**  
  None.

---

## 2.6 AI Analysis & Structured Parsing

**Overview:**  
This block sends normalized market data to an LLM with strict formatting rules, then parses the generated text into structured report fields suitable for storage and email delivery.

**Nodes Involved:**  
- AI Market Insights
- Insights
- Parse AI Output

### Node: AI Market Insights
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates prompt execution using a connected chat model.
- **Configuration choices:**  
  - Prompt type: defined directly in the node
  - User text includes serialized market data:
    - `{{ JSON.stringify($json.market_data, null, 2) }}`
  - Strong system message enforces exact plain-text structure:
    - Market Summary
    - Key Movers
    - Risk Sentiment
    - Actionable Insight
    - Outlook
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.market_data, null, 2) }}`
- **Input and output connections:**  
  - Main input: Normalize Market Data
  - AI language model input: Insights
  - Output: Parse AI Output
- **Version-specific requirements:**  
  Type version `3.1`. Requires n8n LangChain-compatible nodes.
- **Edge cases or potential failure types:**  
  - Model may ignore format instructions partially.
  - Large market datasets can increase token usage.
  - AI output variability is the main downstream parsing risk.
- **Sub-workflow reference:**  
  None.

### Node: Insights
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Provides the chat language model backend for the agent.
- **Configuration choices:**  
  - Model: `llama-3.3-70b-versatile`
  - Uses Groq API credentials
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output (AI language model connector): AI Market Insights
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - Invalid Groq credentials
  - Model availability changes
  - Provider-side rate limiting or timeout
- **Sub-workflow reference:**  
  None.

### Node: Parse AI Output
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the LLM’s free-text response into structured fields.
- **Configuration choices:**  
  The code:
  - Reads `$json.output`
  - Replaces escaped newlines
  - Uses regex extraction for each required section
  - Cleans markdown remnants and whitespace
  - Splits key movers into an array
  - Trims summary, insight, and outlook to at most two sentences
  - Returns:
    - `market_summary`
    - `key_movers`
    - `risk_sentiment`
    - `insight`
    - `outlook`
- **Key expressions or variables used:**  
  - `$json.output`
  - Regex section matching
- **Input and output connections:**  
  - Input: AI Market Insights
  - Output: Log Daily Market Report
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - If headings differ from the enforced format, extraction may fail partially.
  - If the model outputs all text on one line, parsing may still work for sections but mover splitting may degrade.
  - The sentence regex is designed to avoid breaking decimal percentages like `1.46%`, which is a useful implementation detail.
- **Sub-workflow reference:**  
  None.

---

## 2.7 Report Persistence & Email Delivery

**Overview:**  
This final block stores the parsed market report in Google Sheets and sends a plain-text summary email.

**Nodes Involved:**  
- Log Daily Market Report
- Send Today's market summary

### Node: Log Daily Market Report
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a successful report row to the main reporting worksheet.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet: `Multi Asset Daily Snapshot`
  - Sheet/tab: `Sheet1`
  - Writes:
    - `execution_date`
    - `market_summary`
    - `key_movers`
    - `risk_sentiment`
    - `insight`
    - `outlook`
    - `total_assets`
    - `status` = `SUCCESS`
    - `workflow_version` = `v2.0-production`
- **Key expressions or variables used:**  
  - `={{ "SUCCESS" }}`
  - `={{ $json.insight }}`
  - `={{ $json.outlook }}`
  - `={{ $json.key_movers.join(', ') }}`
  - `={{ $items("Normalize Market Data")[0].json.market_data.length }}`
  - `={{ $now }}`
  - `={{ $json.market_summary }}`
  - `={{ $json.risk_sentiment }}`
- **Input and output connections:**  
  - Input: Parse AI Output
  - Output: Send Today's market summary
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - OAuth failure
  - Spreadsheet schema mismatch
  - If `key_movers` is not an array, `.join(', ')` will fail
  - `total_assets` includes all items inside normalized output, including any embedded error-style objects
- **Sub-workflow reference:**  
  None.

### Node: Send Today's market summary
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the final plain-text report via Gmail.
- **Configuration choices:**  
  - Subject: `today's market summary`
  - Body composed from parsed AI fields
  - Plain text email type
- **Key expressions or variables used:**  
  - `{{ $('Parse AI Output').item.json.market_summary }}`
  - `{{ $('Parse AI Output').item.json.key_movers.join('\n') }}`
  - `{{ $('Parse AI Output').item.json.risk_sentiment }}`
  - `{{ $('Parse AI Output').item.json.insight }}`
  - `{{ $json.outlook }}`
- **Input and output connections:**  
  - Input: Log Daily Market Report
  - Output: none
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth issues
  - Missing recipient configuration in the node would block actual sending if not set in UI fields not visible here
  - Email body uses `$json.outlook` from current input and direct references to Parse AI Output for other fields; this mixed reference style is valid but less consistent for maintenance
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Generate Daily Market Report | Manual Trigger | Manual workflow start |  | Environment Config | ## Workflow Trigger & Configuration\n\nThis section initializes the workflow execution and sets up all required environment variables such as API base URL and authentication keys. It ensures that the workflow has the correct configuration before starting the data processing pipeline. |
| Environment Config | Set | Stores TwelveData endpoint and API key | Generate Daily Market Report | Set Market Assets | ## Workflow Trigger & Configuration\n\nThis section initializes the workflow execution and sets up all required environment variables such as API base URL and authentication keys. It ensures that the workflow has the correct configuration before starting the data processing pipeline. |
| Set Market Assets | Set | Defines tracked asset groups | Environment Config | split assests & Symbols | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| split assests & Symbols | Code | Expands grouped symbols into one item per asset | Set Market Assets | Rate Controlled Queue | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| Rate Controlled Queue | Split In Batches | Sequential batching and loop control | split assests & Symbols; Validate API Response; Alert API Faliure | Normalize Market Data; API Throttle | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| API Throttle | Wait | Delays each API request | Rate Controlled Queue | Fetch Asset Prices | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| Fetch Asset Prices | HTTP Request | Calls TwelveData for latest daily candle | API Throttle | Validate API Response | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| Validate API Response | If | Routes success vs failure API payloads | Fetch Asset Prices | Rate Controlled Queue; Log API Error | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| Normalize Market Data | Code | Aggregates and standardizes fetched market data | Rate Controlled Queue | AI Market Insights | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |
| AI Market Insights | LangChain Agent | Produces AI-written market report from normalized data | Normalize Market Data; Insights (AI model connector) | Parse AI Output | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |
| Insights | Groq Chat Model | Supplies LLM backend for the agent |  | AI Market Insights | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |
| Parse AI Output | Code | Extracts structured fields from LLM text | AI Market Insights | Log Daily Market Report | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |
| Log Daily Market Report | Google Sheets | Stores successful market report in spreadsheet | Parse AI Output | Send Today's market summary | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |
| Send Today's market summary | Gmail | Emails final market summary | Log Daily Market Report |  | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |
| Log API Error | Code | Converts failed API response into log format | Validate API Response | Append row in sheet | ## Error Logging & Failure Alerts\n\nThis section manages **API failure handling and error tracking** by capturing failed responses and recording them for monitoring. It logs detailed error information into **Google Sheets** to maintain a history of issues for debugging and analysis.\nAdditionally, it triggers **real-time email alerts via Gmail** to notify about failures instantly, ensuring quick action and maintaining **workflow reliability and transparency**. |
| Append row in sheet | Google Sheets | Appends API failure record to error log sheet | Log API Error | Alert API Faliure | ## Error Logging & Failure Alerts\n\nThis section manages **API failure handling and error tracking** by capturing failed responses and recording them for monitoring. It logs detailed error information into **Google Sheets** to maintain a history of issues for debugging and analysis.\nAdditionally, it triggers **real-time email alerts via Gmail** to notify about failures instantly, ensuring quick action and maintaining **workflow reliability and transparency**. |
| Alert API Faliure | Gmail | Sends API failure alert email and resumes loop | Append row in sheet | Rate Controlled Queue | ## Error Logging & Failure Alerts\n\nThis section manages **API failure handling and error tracking** by capturing failed responses and recording them for monitoring. It logs detailed error information into **Google Sheets** to maintain a history of issues for debugging and analysis.\nAdditionally, it triggers **real-time email alerts via Gmail** to notify about failures instantly, ensuring quick action and maintaining **workflow reliability and transparency**. |
| Sticky Note | Sticky Note | Canvas documentation |  |  | # Multi-Asset Daily Market Snapshot\n\nThis workflow automates the **end-to-end process** of collecting, analyzing and delivering **daily financial market insights** across multiple asset classes including **equities, forex and commodities**. It integrates **real-time market data**, processes it into a **structured format** and uses **AI-powered analysis** to generate a **professional-grade market report** for faster and smarter decision-making.\n\n## How it Works:\nThe workflow begins by defining **asset groups** and fetching **live market data via API**. It then performs **data normalization and transformation** to create a clean dataset, which is analyzed using an **AI model** to generate insights like **market summary, key movers and sentiment**. Finally, the report is **stored and automatically delivered via email**.\n\n## Setup Steps:\n- **TwelveData API** – Add your API key in Environment Config to enable market data fetching\n- **Groq API (Llama Model)** – Configure credentials to power AI-based market insights\n- **Gmail OAuth** – Connect your Gmail account to send automated reports\n- **Asset Configuration** – Customize symbols in "Set Market Assets" as per your needs |
| Sticky Note4 | Sticky Note | Canvas documentation |  |  | ## Workflow Trigger & Configuration\n\nThis section initializes the workflow execution and sets up all required environment variables such as API base URL and authentication keys. It ensures that the workflow has the correct configuration before starting the data processing pipeline. |
| Sticky Note5 | Sticky Note | Canvas documentation |  |  | ## Asset Splitting & API Fetch Pipeline\nThis section handles **asset breakdown and controlled data fetching** by splitting grouped assets into **individual symbols** for processing. It ensures smooth execution using **rate-controlled batching** and **API throttling**, preventing overload or failures while fetching real-time market data.\nThe pipeline guarantees **efficient API utilization**, maintains **data consistency** and enables reliable retrieval of **latest asset prices** for further analysis. |
| Sticky Note6 | Sticky Note | Canvas documentation |  |  | ## Error Logging & Failure Alerts\n\nThis section manages **API failure handling and error tracking** by capturing failed responses and recording them for monitoring. It logs detailed error information into **Google Sheets** to maintain a history of issues for debugging and analysis.\nAdditionally, it triggers **real-time email alerts via Gmail** to notify about failures instantly, ensuring quick action and maintaining **workflow reliability and transparency**. |
| Sticky Note7 | Sticky Note | Canvas documentation |  |  | ## AI Analysis & Report Generation\n\nThis section transforms structured market data into **actionable financial insights** using an **AI-powered model**. It generates key outputs like **market summary, key movers, risk sentiment and outlook**, providing a clear understanding of market conditions.\n\nThe AI response is then **parsed and structured**, stored in **Google Sheets for tracking** and delivered via **automated email notifications**, ensuring consistent and timely reporting. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Multi-Asset Daily Snapshot`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name it: `Generate Daily Market Report`

3. **Add a Set node for environment settings**
   - Node type: `Set`
   - Name it: `Environment Config`
   - Add fields:
     - `api_base_url` as string: `https://api.twelvedata.com/time_series`
     - `twelve_api_key` as string: your TwelveData API key
   - Enable preserving other input fields if needed.
   - Connect:
     - `Generate Daily Market Report` → `Environment Config`

4. **Add a Set node for asset groups**
   - Node type: `Set`
   - Name it: `Set Market Assets`
   - Add fields:
     - `Indices` = `SPY,QQQ,DIA`
     - `Forex` = `EUR/USD,USD/INR,GBP/USD,USD/JPY`
     - `Commodities` = `XAU/USD,USO`
   - Connect:
     - `Environment Config` → `Set Market Assets`

5. **Add a Code node to split grouped assets into symbols**
   - Node type: `Code`
   - Name it: `split assests & Symbols`
   - Paste this logic conceptually:
     - Read the first input item
     - Build an array of asset groups
     - Split each comma-separated group
     - Output one item per symbol with `asset_class` and `symbol`
   - Connect:
     - `Set Market Assets` → `split assests & Symbols`

6. **Add a Split In Batches node**
   - Node type: `Split In Batches`
   - Name it: `Rate Controlled Queue`
   - Set batch size to `1`
   - Connect:
     - `split assests & Symbols` → `Rate Controlled Queue`

7. **Add a Wait node**
   - Node type: `Wait`
   - Name it: `API Throttle`
   - Set delay amount to `15`
   - Verify the time unit in the UI and choose the intended unit consistently.
   - Connect:
     - `Rate Controlled Queue` → `API Throttle`

8. **Add an HTTP Request node for TwelveData**
   - Node type: `HTTP Request`
   - Name it: `Fetch Asset Prices`
   - Method: GET
   - URL:
     - `{{ $('Environment Config').first().json.api_base_url }}`
   - Enable query parameters:
     - `symbol` = `{{ $json.symbol }}`
     - `interval` = `1day`
     - `outputsize` = `1`
     - `apikey` = `{{ $('Environment Config').first().json.twelve_api_key }}`
   - Connect:
     - `API Throttle` → `Fetch Asset Prices`

9. **Add an If node to validate the API response**
   - Node type: `If`
   - Name it: `Validate API Response`
   - Condition:
     - left value: `{{ $json.status }}`
     - operator: equals
     - right value: `ok`
   - Connect:
     - `Fetch Asset Prices` → `Validate API Response`

10. **Create the batch success loop**
    - Connect the **true** output of `Validate API Response` back to `Rate Controlled Queue`
    - This allows the next symbol to be processed.

11. **Add a Code node for error logging**
    - Node type: `Code`
    - Name it: `Log API Error`
    - Configure it to output:
      - `error_time`
      - `error_status`
      - `error_message`
      - `failed_symbol`
    - Connect:
      - `Validate API Response` false branch → `Log API Error`

12. **Prepare Google Sheets for error logging**
    - Create or reuse a spreadsheet, e.g. `Multi Asset Daily Snapshot`
    - Create an error sheet/tab, e.g. `Error logs`
    - Suggested columns:
      - `execution_date`
      - `error_message`
      - `failed_symbol`
      - `status`
      - `error_code`
      - `workflow_version`

13. **Add a Google Sheets node for failure logs**
    - Node type: `Google Sheets`
    - Name it: `Append row in sheet`
    - Operation: `Append`
    - Select the spreadsheet and the `Error logs` tab
    - Use OAuth2 Google Sheets credentials
    - Map columns manually:
      - `execution_date` = `{{ $now }}`
      - `error_message` = ideally `{{ $json.error_message || $json.message || 'Unknown API error' }}`
      - `failed_symbol` = ideally `{{ $json.failed_symbol || $json.symbol || 'N/A' }}`
      - `status` = `FAILED`
      - `error_code` = `{{ $json.code || 'N/A' }}`
      - `workflow_version` = `v2.0-production`
    - Connect:
      - `Log API Error` → `Append row in sheet`

14. **Add a Gmail node for API failure alerts**
    - Node type: `Gmail`
    - Name it: `Alert API Faliure`
    - Authenticate with Gmail OAuth2 credentials
    - Set email type to plain text
    - Subject: `API Failure Alert`
    - Message body should preferably reference normalized error fields:
      - `Twelve Data API failed.`
      - `Status: {{ $json.error_status || $json.status }}`
      - `Message: {{ $json.error_message || $json.message }}`
    - Configure recipient fields in the node UI
    - Connect:
      - `Append row in sheet` → `Alert API Faliure`

15. **Return failed items to the batch loop**
    - Connect:
      - `Alert API Faliure` → `Rate Controlled Queue`
    - This ensures processing continues after a failure.

16. **Add a Code node to normalize all fetched market data**
    - Node type: `Code`
    - Name it: `Normalize Market Data`
    - The node should:
      - iterate across execution runs of `Fetch Asset Prices`
      - collect latest `values[0]`
      - compute open/close percent change
      - label trend as bullish/bearish
      - retain fallback error objects when data is incomplete
      - return one item containing `market_data`
    - Connect:
      - `Rate Controlled Queue` → `Normalize Market Data`

17. **Add the Groq chat model node**
    - Node type: `Groq Chat Model`
    - Name it: `Insights`
    - Credentials: create Groq API credentials
    - Model: `llama-3.3-70b-versatile`

18. **Add the LangChain Agent node**
    - Node type: `AI Agent` / `LangChain Agent`
    - Name it: `AI Market Insights`
    - Set prompt type to directly define the prompt
    - User prompt should include serialized normalized data:
      - `Here is today's normalized multi-asset market data:`
      - `{{ JSON.stringify($json.market_data, null, 2) }}`
      - Ask for a professional daily market report
    - Add a system instruction enforcing exact sections:
      - `Market Summary:`
      - `Key Movers:`
      - `Risk Sentiment:`
      - `Actionable Insight:`
      - `Outlook:`
    - Require plain text only, no markdown or JSON
    - Connect:
      - `Normalize Market Data` → `AI Market Insights`
      - `Insights` AI model connector → `AI Market Insights`

19. **Add a Code node to parse the AI output**
    - Node type: `Code`
    - Name it: `Parse AI Output`
    - Logic should:
      - read `$json.output`
      - normalize escaped line breaks
      - extract sections with regex
      - clean spacing and markdown remnants
      - split key movers into an array
      - trim summary, insight, and outlook to two sentences
      - return:
        - `market_summary`
        - `key_movers`
        - `risk_sentiment`
        - `insight`
        - `outlook`
    - Connect:
      - `AI Market Insights` → `Parse AI Output`

20. **Prepare Google Sheets for successful report logging**
    - In the same spreadsheet, create the main sheet, e.g. `Sheet1`
    - Suggested columns:
      - `execution_date`
      - `market_summary`
      - `key_movers`
      - `risk_sentiment`
      - `insight`
      - `outlook`
      - `total_assets`
      - `status`
      - `workflow_version`

21. **Add a Google Sheets node for successful report storage**
    - Node type: `Google Sheets`
    - Name it: `Log Daily Market Report`
    - Operation: `Append`
    - Select the main report sheet
    - Use Google Sheets OAuth2 credentials
    - Map columns:
      - `execution_date` = `{{ $now }}`
      - `market_summary` = `{{ $json.market_summary }}`
      - `key_movers` = `{{ $json.key_movers.join(', ') }}`
      - `risk_sentiment` = `{{ $json.risk_sentiment }}`
      - `insight` = `{{ $json.insight }}`
      - `outlook` = `{{ $json.outlook }}`
      - `total_assets` = `{{ $items("Normalize Market Data")[0].json.market_data.length }}`
      - `status` = `SUCCESS`
      - `workflow_version` = `v2.0-production`
    - Connect:
      - `Parse AI Output` → `Log Daily Market Report`

22. **Add a Gmail node for final report delivery**
    - Node type: `Gmail`
    - Name it: `Send Today's market summary`
    - Use Gmail OAuth2 credentials
    - Set email type to plain text
    - Subject: `today's market summary`
    - Build the body from parsed fields:
      - `Market Summary:`
      - `{{ $('Parse AI Output').item.json.market_summary }}`
      - `Key Movers:`
      - `{{ $('Parse AI Output').item.json.key_movers.join('\n') }}`
      - `Risk Sentiment:`
      - `{{ $('Parse AI Output').item.json.risk_sentiment }}`
      - `Insight:`
      - `{{ $('Parse AI Output').item.json.insight }}`
      - `outlook:`
      - `{{ $json.outlook }}`
    - Configure recipients explicitly in the node
    - Connect:
      - `Log Daily Market Report` → `Send Today's market summary`

23. **Add credentials**
    - **Groq API**
      - Create Groq API credential in n8n
      - Attach it to `Insights`
    - **Google Sheets OAuth2**
      - Create Google Sheets OAuth2 credential
      - Attach it to:
        - `Log Daily Market Report`
        - `Append row in sheet`
    - **Gmail OAuth2**
      - Create Gmail OAuth2 credential
      - Attach it to:
        - `Send Today's market summary`
        - `Alert API Faliure`

24. **Recommended fixes while rebuilding**
    - In the failure branch, align field names consistently:
      - Use `error_status`, `error_message`, `failed_symbol` everywhere after `Log API Error`
    - Add a neutral trend option for zero price change
    - Confirm whether the Wait node uses seconds/minutes and set explicitly
    - Consider adding a Schedule Trigger for daily automation if you want production scheduling

25. **Test the workflow**
    - Run manually
    - Verify:
      - TwelveData fetches data for all symbols
      - Failed symbols are logged without stopping the run
      - `Normalize Market Data` returns a complete `market_data` array
      - AI output respects the requested format
      - Parsed fields are non-empty
      - Google Sheets rows are appended correctly
      - Emails are delivered

26. **Optional production enhancement**
    - Replace the Manual Trigger with a Schedule Trigger for daily runs
    - Store API keys in environment variables or credentials instead of plain Set fields
    - Add explicit recipient lists and sheet schema documentation

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Multi-Asset Daily Market Snapshot: This workflow automates the end-to-end process of collecting, analyzing and delivering daily financial market insights across equities, forex and commodities. | General workflow description |
| Setup step: Add your TwelveData API key in Environment Config to enable market data fetching. | TwelveData integration |
| Setup step: Configure Groq API credentials for the Llama model used in AI analysis. | Groq integration |
| Setup step: Connect Gmail OAuth to send automated reports and alerts. | Gmail integration |
| Setup step: Customize the symbols in `Set Market Assets` to change the tracked universe. | Workflow customization |
| Spreadsheet used for logging: `Multi Asset Daily Snapshot` with a main report sheet and an `Error logs` tab. | Google Sheets logging structure |

## Additional implementation observations
- There is only one workflow entry point: `Generate Daily Market Report`.
- There are no sub-workflows or Execute Workflow nodes in this workflow.
- The workflow uses loop-back connections through `Split In Batches`, which is central to its execution logic.
- The error branch currently contains field-name inconsistencies between `Log API Error`, `Append row in sheet`, and `Alert API Faliure`; these should be normalized if reliability is important.
- The success path depends on AI output matching strict section headings; if model behavior drifts, parsing quality will drop.