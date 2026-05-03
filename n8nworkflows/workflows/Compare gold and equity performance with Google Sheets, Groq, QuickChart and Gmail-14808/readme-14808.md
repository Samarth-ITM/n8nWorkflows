Compare gold and equity performance with Google Sheets, Groq, QuickChart and Gmail

https://n8nworkflows.xyz/workflows/compare-gold-and-equity-performance-with-google-sheets--groq--quickchart-and-gmail-14808


# Compare gold and equity performance with Google Sheets, Groq, QuickChart and Gmail

# 1. Workflow Overview

This workflow automates a comparative financial analysis between gold and equity over a user-defined date range. It reads historical prices from Google Sheets, aligns the two asset series, calculates percentage returns, asks a Groq-powered AI model to produce investment commentary, generates a QuickChart visualization, builds an HTML report, sends either a normal or alert email depending on the performance gap, and logs the result back into Google Sheets.

## 1.1 Manual Start and Parameter Definition
The workflow begins with a manual trigger and a parameter node where the user defines:
- `startDate`
- `endDate`
- `threshold`

These values control the analysis window and the alert threshold.

## 1.2 Market Data Retrieval
Two Google Sheets reads are used to pull historical price records:
- Gold prices
- Equity prices

Both datasets come from separate tabs in the same spreadsheet.

## 1.3 Data Alignment and Performance Calculation
A custom code node merges the two time series, filters by date range, removes duplicate dates, and produces paired daily records. Another code node calculates:
- Gold return
- Equity return
- Return difference
- Winning asset

## 1.4 AI Analysis
The calculated metrics are sent to an AI Agent using a Groq chat model (`llama-3.3-70b-versatile`). The AI is instructed to return strict JSON containing:
- Summary
- Winner
- Reason
- Investment advice
- Allocation percentages
- Strategy

A parsing code node then converts the AI text response into structured JSON.

## 1.5 Chart Generation
In parallel with the AI branch, another code node converts the aligned daily asset prices into a QuickChart line chart URL for visual comparison.

## 1.6 Report Assembly
A Merge node recombines:
- Parsed AI output
- Chart output

A final code node injects those values into a styled HTML email template.

## 1.7 Conditional Delivery and Audit Logging
An IF node compares the absolute return gap with the configured threshold:
- If below or equal: send standard report email
- If above: send alert email

Both branches end by appending a summary record to a Google Sheet for history tracking.

---

# 2. Block-by-Block Analysis

## 2.1 Manual Start and Analysis Parameters

### Overview
This block starts the workflow manually and defines the date window and threshold used across the rest of the process. These parameters are referenced later by code and conditional logic.

### Nodes Involved
- Run Report
- Set Analysis Parameters

### Node Details

#### Run Report
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for testing or on-demand execution.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Set Analysis Parameters`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None significant; runs only when manually triggered.
- **Sub-workflow reference:** None.

#### Set Analysis Parameters
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the core configuration payload for downstream nodes.
- **Configuration choices:** Assigns:
  - `startDate = "2026-03-01"`
  - `endDate = "2026-03-10"`
  - `threshold = 5`
- **Key expressions or variables used:** These fields are later referenced as:
  - `$('Set Analysis Parameters').first().json.startDate`
  - `$('Set Analysis Parameters').first().json.endDate`
  - `$('Set Analysis Parameters').first().json.threshold`
- **Input and output connections:**  
  - Input: `Run Report`
  - Output: `Fetch Gold Prices`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Invalid date strings can break date parsing in later code nodes.
  - A non-numeric threshold would break the IF comparison.
  - If `startDate > endDate`, the merge block will likely return no data.
- **Sub-workflow reference:** None.

---

## 2.2 Market Data Retrieval

### Overview
This block fetches historical records for the two assets from Google Sheets. The workflow is chained so that gold data is fetched first, then equity data, after which the equity branch passes control into the merge logic.

### Nodes Involved
- Fetch Gold Prices
- Fetch Equity Prices

### Node Details

#### Fetch Gold Prices
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the Gold sheet.
- **Configuration choices:**  
  - Google Sheets document: `Gold Data`
  - Sheet/tab: `Gold`
  - `executeOnce: true`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**  
  - Input: `Set Analysis Parameters`
  - Output: `Fetch Equity Prices`
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Expired or missing Google Sheets OAuth2 credentials
  - Incorrect spreadsheet or tab selection
  - Missing expected columns such as `Date` and `Price`
  - Empty result set
- **Sub-workflow reference:** None.

#### Fetch Equity Prices
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from the Equity sheet.
- **Configuration choices:**  
  - Google Sheets document: `Gold Data`
  - Sheet/tab: `Equity`
  - `executeOnce: true`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:**  
  - Input: `Fetch Gold Prices`
  - Output: `Merge Market Data`
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Same credential and access risks as the previous node
  - If row ordering differs from Gold sheet, the later merge logic may produce incorrect asset pairing because the code aligns by array index, not by explicit date lookup
  - Missing `Date` or `Price` fields
- **Sub-workflow reference:** None.

---

## 2.3 Data Alignment and Performance Calculation

### Overview
This block normalizes and aligns the two market datasets, filters records within the selected date range, and computes comparative performance metrics. It is the core numerical part of the workflow.

### Nodes Involved
- Merge Market Data
- Calculate Performance Metrics

### Node Details

#### Merge Market Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation for pairing Gold and Equity rows.
- **Configuration choices:**  
  The script:
  - Retrieves all Gold rows via `$items("Fetch Gold Prices")`
  - Uses current incoming items as Equity rows
  - Reads `startDate` and `endDate` from `Set Analysis Parameters`
  - Iterates by index across Equity rows
  - Skips rows where either asset lacks `Price`
  - Normalizes Gold date values by removing commas and trimming whitespace
  - Removes duplicate dates using a `Set`
  - Filters records outside the requested date range
  - Emits objects with:
    - `date`
    - `goldPrice`
    - `equityPrice`
  - Returns an error item if no matched data exists
- **Key expressions or variables used:**  
  - `$items("Fetch Gold Prices")`
  - `$("Set Analysis Parameters").first().json`
- **Input and output connections:**  
  - Input: `Fetch Equity Prices`
  - Outputs:
    - `Calculate Performance Metrics`
    - `Generate Chart`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Assumes Gold and Equity rows align by index; if sheets are sorted differently or have missing dates in one sheet, results may be inaccurate
  - Uses Gold dates as the canonical date source; Equity dates are not explicitly checked
  - JavaScript date parsing may behave differently if sheet date strings are not ISO-like or locale-stable
  - If there are no valid rows in range, returns:
    - `error: "No market data found in selected date range"`
- **Sub-workflow reference:** None.

#### Calculate Performance Metrics
- **Type and technical role:** `n8n-nodes-base.code`  
  Computes beginning-to-end returns and determines the outperforming asset.
- **Configuration choices:**  
  The script:
  - Exits early with an error if incoming data is empty or already contains an error
  - Uses the first matched item as the starting point
  - Uses the last matched item as the ending point
  - Calculates:
    - `goldReturn`
    - `equityReturn`
    - `difference = goldReturn - equityReturn`
  - Sets `winner` to `Gold`, `Equity`, or `Equal`
  - Returns values formatted with two decimals
- **Key expressions or variables used:** Uses incoming `items`.
- **Input and output connections:**  
  - Input: `Merge Market Data`
  - Output: `Generate AI Investment Insights`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If first price is zero, division by zero will occur
  - If only one valid row exists, returns will be zero because start and end are the same
  - If rows are unsorted chronologically, the return calculation may be wrong
  - Returns numeric values as strings due to `.toFixed(2)`, which must later be cast if numerical comparison is required
- **Sub-workflow reference:** None.

---

## 2.4 AI Analysis and JSON Parsing

### Overview
This block converts the raw performance metrics into human-readable investment insight using a Groq language model. It then parses the model output into structured JSON suitable for templating and storage.

### Nodes Involved
- Generate AI Investment Insights
- Insights
- Parse AI Output

### Node Details

#### Generate AI Investment Insights
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based AI agent node that sends a prompt to the attached language model.
- **Configuration choices:**  
  - Prompt type: defined directly in the node
  - User prompt includes:
    - `startDate`
    - `endDate`
    - `goldReturn`
    - `equityReturn`
    - `difference`
  - System message instructs the AI to:
    - compare performance
    - explain why one asset performed better
    - suggest an allocation totaling 100%
    - use exact numbers
    - output only valid JSON
- **Key expressions or variables used:**  
  - `{{ $json.startDate }}`
  - `{{ $json.endDate }}`
  - `{{ $json.goldReturn }}`
  - `{{ $json.equityReturn }}`
  - `{{ $json.difference }}`
- **Input and output connections:**  
  - Main input: `Calculate Performance Metrics`
  - AI language model input: `Insights`
  - Main output: `Parse AI Output`
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain nodes installed in the n8n environment.
- **Edge cases or potential failure types:**  
  - Groq API credential/authentication errors
  - Non-JSON or partially valid JSON output from the model
  - Hallucinated fields or missing fields
  - LLM latency or rate limits
  - If upstream data contains an error payload, this node may still run unless additional guards are added
- **Sub-workflow reference:** None.

#### Insights
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Provides the Groq-hosted chat model used by the AI Agent.
- **Configuration choices:**  
  - Model: `llama-3.3-70b-versatile`
  - Temperature: `0.7`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output (AI language model connection): `Generate AI Investment Insights`
- **Version-specific requirements:** Type version `1`; requires Groq API credentials.
- **Edge cases or potential failure types:**  
  - Invalid Groq API key
  - Model availability changes
  - High temperature may increase the chance of malformed JSON
- **Sub-workflow reference:** None.

#### Parse AI Output
- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the text output of the AI node into structured JSON.
- **Configuration choices:**  
  The script:
  - Reads `items[0].json.output`
  - Attempts `JSON.parse`
  - Returns parsed JSON if successful
  - Otherwise returns:
    - `error: "Invalid JSON from AI"`
    - `rawOutput`
- **Key expressions or variables used:** `items[0].json.output`
- **Input and output connections:**  
  - Input: `Generate AI Investment Insights`
  - Output: `Combine AI + Chart Data`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - If the AI node output field is not named `output`, parsing fails
  - Model may wrap JSON in markdown fences, causing parse failure
  - Downstream nodes do not explicitly halt on parse failure, so the report can be generated with missing fields
- **Sub-workflow reference:** None.

---

## 2.5 Chart Generation

### Overview
This block takes the merged market data and builds a QuickChart line chart URL. It produces a visual comparison of both asset price series over the selected period.

### Nodes Involved
- Generate Chart

### Node Details

#### Generate Chart
- **Type and technical role:** `n8n-nodes-base.code`  
  Converts paired daily market data into arrays and a chart URL.
- **Configuration choices:**  
  The script:
  - Stops early if items are empty or contain an error
  - Builds:
    - `labels` from dates
    - `gold` data array
    - `equity` data array
  - Creates a Chart.js-style config for a line chart
  - Encodes that config into a QuickChart URL:
    - width `900`
    - height `450`
    - white background
- **Key expressions or variables used:** Uses incoming `items`.
- **Input and output connections:**  
  - Input: `Merge Market Data`
  - Output: `Combine AI + Chart Data`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Long date ranges can create very long URLs
  - Empty or malformed data arrays produce broken charts
  - QuickChart availability is external and not guaranteed
  - If upstream merge returns an error, this node emits blank `chartUrl`
- **Sub-workflow reference:** None.

---

## 2.6 Report Assembly

### Overview
This block recombines the AI insight stream and the chart stream, then builds a styled HTML report. It centralizes the content used by both email paths.

### Nodes Involved
- Combine AI + Chart Data
- Generate Final Report

### Node Details

#### Combine AI + Chart Data
- **Type and technical role:** `n8n-nodes-base.merge`  
  Rejoins the two branches after AI parsing and chart generation.
- **Configuration choices:**  
  No custom parameters are explicitly set, so default merge behavior applies.
- **Key expressions or variables used:** None directly in the node.
- **Input and output connections:**  
  - Input 1: `Parse AI Output`
  - Input 2: `Generate Chart`
  - Output: `Generate Final Report`
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**  
  - Merge behavior depends on default mode; with mismatched item counts, unexpected pairings may happen
  - Since both upstream branches currently emit a single result item in normal operation, it should behave predictably
- **Sub-workflow reference:** None.

#### Generate Final Report
- **Type and technical role:** `n8n-nodes-base.code`  
  Builds the final HTML body used in outgoing emails.
- **Configuration choices:**  
  The script:
  - Assumes:
    - `items[0].json` = AI data
    - `items[1].json` = chart data
  - Merges them into a single object
  - Uses `chart.url || chart.chartUrl`
  - Creates a styled HTML report including:
    - Winner
    - Summary
    - Market insight
    - Investment advice
    - Gold allocation
    - Equity allocation
    - Strategy
    - Chart image
- **Key expressions or variables used:** Uses ordered `items` array from merge output.
- **Input and output connections:**  
  - Input: `Combine AI + Chart Data`
  - Output: `Check Performance Gap`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Assumes exact item ordering from the merge node
  - If AI output contains an error instead of expected fields, HTML may render undefined values
  - No HTML escaping is performed on AI output, so malformed content could break formatting
  - Broken chart URL still results in email HTML but with a broken image
- **Sub-workflow reference:** None.

---

## 2.7 Conditional Delivery and Audit Logging

### Overview
This block decides whether the report should be sent as a normal notification or an alert, then appends an execution record to Google Sheets.

### Nodes Involved
- Check Performance Gap
- Send Report Email
- Send Alert Email
- Store Report History

### Node Details

#### Check Performance Gap
- **Type and technical role:** `n8n-nodes-base.if`  
  Compares the absolute return difference against the threshold.
- **Configuration choices:**  
  Condition:
  - Left value: `Math.abs(Number($('Calculate Performance Metrics').first().json.difference))`
  - Operator: greater than
  - Right value: `$('Set Analysis Parameters').first().json.threshold`
- **Key expressions or variables used:**  
  - `$('Calculate Performance Metrics').first().json.difference`
  - `$('Set Analysis Parameters').first().json.threshold`
- **Input and output connections:**  
  - Input: `Generate Final Report`
  - True output: `Send Report Email`
  - False output: `Send Alert Email`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - The current wiring appears logically inverted relative to the node names:
    - Condition `difference > threshold` goes to output index 0
    - Output index 0 is connected to `Send Report Email`
    - Output index 1 is connected to `Send Alert Email`
  - In standard n8n IF behavior, output 0 is usually the **true** branch and output 1 the **false** branch. If so, large gaps would send the normal report and small gaps would send the alert, which is opposite of the intended behavior.
  - If `difference` is non-numeric, comparison may fail or coerce unexpectedly.
- **Sub-workflow reference:** None.

#### Send Report Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the standard email version of the report.
- **Configuration choices:**  
  - Subject: `Gold vs Equity Report ({{ $now }})`
  - Body: `{{ $json.html }}`
- **Key expressions or variables used:**  
  - `{{ $json.html }}`
  - `{{ $now }}`
- **Input and output connections:**  
  - Input: `Check Performance Gap`
  - Output: `Store Report History`
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Recipient configuration is not visible in the provided JSON; if not set elsewhere, the node may be incomplete
  - Gmail auth issues
  - HTML body may be treated as plain text depending on node options/version behavior if not configured as HTML content
  - Google sending limits or security blocks
- **Sub-workflow reference:** None.

#### Send Alert Email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the alert variant of the email.
- **Configuration choices:**  
  - Subject: `ALERT: Significant Performance Gap`
  - Body prefixes text:
    - `Alert Triggered!`
    - `A significant difference in asset performance detected.`
    - followed by `{{ $json.html }}`
- **Key expressions or variables used:**  
  - `{{ $json.html }}`
- **Input and output connections:**  
  - Input: `Check Performance Gap`
  - Output: `Store Report History`
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Same Gmail auth and recipient concerns as above
  - Mixing plain text prefix with HTML body may result in inconsistent formatting depending on Gmail node handling
- **Sub-workflow reference:** None.

#### Store Report History
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a historical record of the report to a Google Sheet.
- **Configuration choices:**  
  - Operation: `append`
  - Document: `Gold Data`
  - Sheet/tab: `Sheet3`
  - Explicit column mapping:
    - `Date = {{ $now }}`
    - `Report = {{ $('Parse AI Output').first().json.reason }}`
    - `Winner = {{ $('Parse AI Output').first().json.winner }}`
    - `Summary = {{ $('Parse AI Output').first().json.summary }}`
- **Key expressions or variables used:**  
  - `{{ $now }}`
  - `{{ $('Parse AI Output').first().json.reason }}`
  - `{{ $('Parse AI Output').first().json.winner }}`
  - `{{ $('Parse AI Output').first().json.summary }}`
- **Input and output connections:**  
  - Inputs:
    - `Send Report Email`
    - `Send Alert Email`
  - Output: none
- **Version-specific requirements:** Type version `4.7`; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Destination sheet headers must match schema
  - If AI parsing fails, mapped fields may be blank or undefined
  - Append permissions or quota issues
- **Sub-workflow reference:** None.

---

## 2.8 Sticky Notes / In-Canvas Documentation

### Overview
These nodes do not affect execution, but they document the workflow purpose and each major section directly in the canvas.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  General overview and setup note.
- **Configuration choices:** Large note covering overall purpose, process, and setup.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents setup and market data ingestion block.
- **Configuration choices:** Covers parameter setup, Google Sheets fetch, merge, and cleaning.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents AI insight and parsing block.
- **Configuration choices:** Describes Groq-powered AI analysis and JSON parsing.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents chart generation block.
- **Configuration choices:** Describes QuickChart line graph generation from aligned arrays.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents report generation block.
- **Configuration choices:** Describes merge of AI JSON and chart URL into HTML.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents final conditional email and logging block.
- **Configuration choices:** Describes threshold check, Gmail branching, and historical logging.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Report | Manual Trigger | Manual workflow entry point |  | Set Analysis Parameters | # Gold vs Equity Performance Comparison Tracker with Visual Insights<br><br>This workflow automates financial performance analysis between gold and equity using historical market data. It fetches price data from Google Sheets, processes and compares returns, generates AI-driven investment insights, visualizes performance trends and sends a formatted report via email.<br><br>## How it works:<br>The workflow starts with user-defined date parameters, **retrieves gold and equity data**, merges and cleans it and calculates returns. AI then analyzes performance and suggests portfolio allocation. A **chart is generated for visualization** and a final HTML report is prepared and emailed. Alerts are triggered if performance differences exceed a threshold.<br><br>## Setup steps:<br>1. Google SheetsConnect Google Sheets (Gold + Equity data)<br>2. Configure Groq API for AI insights<br>3. Set Gmail credentials for report delivery<br>4. Adjust date range and threshold in parameters<br>5. Execute workflow manually or via trigger |
| Set Analysis Parameters | Set | Defines start date, end date, and alert threshold | Run Report | Fetch Gold Prices | ## Setup & Market Data Ingestion<br>The workflow begins by setting the analysis parameters (start/end dates and threshold limits). It then connects to Google Sheets to fetch historical daily price data for both Gold and Equity. The data is merged and cleaned by date to ensure an accurate, day-by-day head-to-head comparison before moving to the analysis phase. |
| Fetch Gold Prices | Google Sheets | Reads Gold historical price rows | Set Analysis Parameters | Fetch Equity Prices | ## Setup & Market Data Ingestion<br>The workflow begins by setting the analysis parameters (start/end dates and threshold limits). It then connects to Google Sheets to fetch historical daily price data for both Gold and Equity. The data is merged and cleaned by date to ensure an accurate, day-by-day head-to-head comparison before moving to the analysis phase. |
| Fetch Equity Prices | Google Sheets | Reads Equity historical price rows | Fetch Gold Prices | Merge Market Data | ## Setup & Market Data Ingestion<br>The workflow begins by setting the analysis parameters (start/end dates and threshold limits). It then connects to Google Sheets to fetch historical daily price data for both Gold and Equity. The data is merged and cleaned by date to ensure an accurate, day-by-day head-to-head comparison before moving to the analysis phase. |
| Merge Market Data | Code | Aligns Gold and Equity rows, filters by date range, emits paired daily data | Fetch Equity Prices | Calculate Performance Metrics; Generate Chart | ## Setup & Market Data Ingestion<br>The workflow begins by setting the analysis parameters (start/end dates and threshold limits). It then connects to Google Sheets to fetch historical daily price data for both Gold and Equity. The data is merged and cleaned by date to ensure an accurate, day-by-day head-to-head comparison before moving to the analysis phase. |
| Calculate Performance Metrics | Code | Calculates returns, difference, and winner | Merge Market Data | Generate AI Investment Insights | ## AI Insights & Data Parsing<br>Performance metrics are passed directly to a Groq-powered AI Agent which analyzes the financial returns. The AI evaluates the data to determine the winning asset, provide realistic market context and recommend a strategic portfolio allocation. The resulting raw text from the AI is then parsed into structured JSON, ensuring the generated insights can be seamlessly mapped into the final HTML email report. |
| Generate AI Investment Insights | LangChain Agent | Produces structured AI investment commentary from calculated returns | Calculate Performance Metrics; Insights (AI language model) | Parse AI Output | ## AI Insights & Data Parsing<br>Performance metrics are passed directly to a Groq-powered AI Agent which analyzes the financial returns. The AI evaluates the data to determine the winning asset, provide realistic market context and recommend a strategic portfolio allocation. The resulting raw text from the AI is then parsed into structured JSON, ensuring the generated insights can be seamlessly mapped into the final HTML email report. |
| Insights | Groq Chat Model | Supplies the LLM used by the AI agent |  | Generate AI Investment Insights | ## AI Insights & Data Parsing<br>Performance metrics are passed directly to a Groq-powered AI Agent which analyzes the financial returns. The AI evaluates the data to determine the winning asset, provide realistic market context and recommend a strategic portfolio allocation. The resulting raw text from the AI is then parsed into structured JSON, ensuring the generated insights can be seamlessly mapped into the final HTML email report. |
| Parse AI Output | Code | Parses AI text output into JSON or returns parse error | Generate AI Investment Insights | Combine AI + Chart Data | ## AI Insights & Data Parsing<br>Performance metrics are passed directly to a Groq-powered AI Agent which analyzes the financial returns. The AI evaluates the data to determine the winning asset, provide realistic market context and recommend a strategic portfolio allocation. The resulting raw text from the AI is then parsed into structured JSON, ensuring the generated insights can be seamlessly mapped into the final HTML email report. |
| Generate Chart | Code | Builds QuickChart URL from aligned market data | Merge Market Data | Combine AI + Chart Data | ## Chart Data & Visualization<br>Structures Gold and Equity price data into arrays and generates a QuickChart line graph URL for performance comparison. |
| Combine AI + Chart Data | Merge | Rejoins AI output and chart output | Parse AI Output; Generate Chart | Generate Final Report | ## Report Generation<br>It synchronizes the two separate data streams back together. It takes the structured JSON insights from the AI agent and the generated chart URL, combining them into a single data object. Finally, it injects this combined data into a stylized HTML template to build the final comparative report. |
| Generate Final Report | Code | Builds final HTML report payload | Combine AI + Chart Data | Check Performance Gap | ## Report Generation<br>It synchronizes the two separate data streams back together. It takes the structured JSON insights from the AI agent and the generated chart URL, combining them into a single data object. Finally, it injects this combined data into a stylized HTML template to build the final comparative report. |
| Check Performance Gap | IF | Compares absolute performance difference against threshold | Generate Final Report | Send Report Email; Send Alert Email | ## Conditional Email Delivery & Auditing<br><br>This final section evaluates the calculated performance gap against a predefined threshold. Depending on the outcome, it branches to send either a standard "Report Email" or an urgent "Alert Email" via Gmail. After the notification is dispatched, the workflow appends the AI's summary, the winning asset and the execution date into a Google Sheet to maintain a historical log of the generated reports. |
| Send Report Email | Gmail | Sends standard report email | Check Performance Gap | Store Report History | ## Conditional Email Delivery & Auditing<br><br>This final section evaluates the calculated performance gap against a predefined threshold. Depending on the outcome, it branches to send either a standard "Report Email" or an urgent "Alert Email" via Gmail. After the notification is dispatched, the workflow appends the AI's summary, the winning asset and the execution date into a Google Sheet to maintain a historical log of the generated reports. |
| Send Alert Email | Gmail | Sends alert email variant | Check Performance Gap | Store Report History | ## Conditional Email Delivery & Auditing<br><br>This final section evaluates the calculated performance gap against a predefined threshold. Depending on the outcome, it branches to send either a standard "Report Email" or an urgent "Alert Email" via Gmail. After the notification is dispatched, the workflow appends the AI's summary, the winning asset and the execution date into a Google Sheet to maintain a historical log of the generated reports. |
| Store Report History | Google Sheets | Appends report summary to history sheet | Send Report Email; Send Alert Email |  | ## Conditional Email Delivery & Auditing<br><br>This final section evaluates the calculated performance gap against a predefined threshold. Depending on the outcome, it branches to send either a standard "Report Email" or an urgent "Alert Email" via Gmail. After the notification is dispatched, the workflow appends the AI's summary, the winning asset and the execution date into a Google Sheet to maintain a historical log of the generated reports. |
| Sticky Note | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for setup/data ingestion |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for AI block |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for chart block |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for report generation block |  |  |  |
| Sticky Note5 | Sticky Note | Canvas documentation for delivery/logging block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Gold vs Equity Performance Comparison Tracker`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name: `Run Report`

3. **Add a Set node**
   - Node type: `Set`
   - Name: `Set Analysis Parameters`
   - Add three fields:
     - `startDate` as String, for example `2026-03-01`
     - `endDate` as String, for example `2026-03-10`
     - `threshold` as Number, for example `5`
   - Connect `Run Report` → `Set Analysis Parameters`

4. **Add the first Google Sheets node for Gold**
   - Node type: `Google Sheets`
   - Name: `Fetch Gold Prices`
   - Credential: connect a Google Sheets OAuth2 credential
   - Select the source spreadsheet
   - Select the sheet/tab containing Gold data
   - Expected data structure should include at least:
     - `Date`
     - `Price`
   - Keep it configured to read rows
   - Set `executeOnce` if available in your version
   - Connect `Set Analysis Parameters` → `Fetch Gold Prices`

5. **Add the second Google Sheets node for Equity**
   - Node type: `Google Sheets`
   - Name: `Fetch Equity Prices`
   - Use the same or another Google Sheets OAuth2 credential
   - Select the source spreadsheet
   - Select the sheet/tab containing Equity data
   - Expected columns:
     - `Date`
     - `Price`
   - Set `executeOnce` if available
   - Connect `Fetch Gold Prices` → `Fetch Equity Prices`

6. **Add a Code node to merge the two market datasets**
   - Node type: `Code`
   - Name: `Merge Market Data`
   - Connect `Fetch Equity Prices` → `Merge Market Data`
   - Paste logic equivalent to:
     - pull Gold items using `$items("Fetch Gold Prices")`
     - use current incoming items as Equity items
     - read `startDate` and `endDate` from `Set Analysis Parameters`
     - iterate through rows
     - skip rows without prices
     - normalize date strings
     - deduplicate dates
     - filter outside date window
     - return objects with:
       - `date`
       - `goldPrice`
       - `equityPrice`
     - if nothing matches, return one item with an `error` field
   - Important expectation:
     - both sheets should be sorted consistently
     - both sheets should have corresponding rows by index

7. **Add a Code node for performance calculation**
   - Node type: `Code`
   - Name: `Calculate Performance Metrics`
   - Connect `Merge Market Data` → `Calculate Performance Metrics`
   - Configure logic to:
     - stop if input contains an error
     - use first and last paired dates
     - calculate:
       - gold return %
       - equity return %
       - difference %
       - winner
     - output:
       - `startDate`
       - `endDate`
       - `goldReturn`
       - `equityReturn`
       - `difference`
       - `winner`

8. **Add the Groq chat model node**
   - Node type: `Groq Chat Model` / `lmChatGroq`
   - Name: `Insights`
   - Credential: connect your Groq API credential
   - Model: `llama-3.3-70b-versatile`
   - Temperature: `0.7`

9. **Add the AI Agent node**
   - Node type: `LangChain Agent`
   - Name: `Generate AI Investment Insights`
   - Connect `Calculate Performance Metrics` → `Generate AI Investment Insights`
   - Connect the `Insights` node to the AI Agent via the AI language model port
   - Set the prompt text to include the computed values, for example:
     - Start Date
     - End Date
     - Gold Return
     - Equity Return
     - Difference
   - Add a system message instructing the model to:
     - compare gold vs equity
     - explain the winner
     - give allocation advice totaling 100%
     - output only valid JSON
   - Required output fields:
     - `summary`
     - `winner`
     - `reason`
     - `investmentAdvice`
     - `goldAllocation`
     - `equityAllocation`
     - `strategy`

10. **Add a Code node to parse the AI output**
    - Node type: `Code`
    - Name: `Parse AI Output`
    - Connect `Generate AI Investment Insights` → `Parse AI Output`
    - Configure logic to:
      - read `items[0].json.output`
      - `JSON.parse()` it
      - if parsing fails, return:
        - `error`
        - `rawOutput`

11. **Add a Code node for chart generation**
    - Node type: `Code`
    - Name: `Generate Chart`
    - Connect `Merge Market Data` → `Generate Chart`
    - Configure logic to:
      - extract `labels`, `gold`, and `equity` arrays
      - construct a Chart.js config
      - generate a QuickChart URL using:
        - `https://quickchart.io/chart?...&c=` + encoded config
      - return:
        - `labels`
        - `gold`
        - `equity`
        - `chartUrl`

12. **Add a Merge node**
    - Node type: `Merge`
    - Name: `Combine AI + Chart Data`
    - Connect:
      - `Parse AI Output` → input 1
      - `Generate Chart` → input 2
    - Use the merge mode that preserves one item from each branch for downstream report assembly.
    - Since the next node expects separate items, verify the merge output structure in your n8n version.

13. **Add a Code node to build the final HTML**
    - Node type: `Code`
    - Name: `Generate Final Report`
    - Connect `Combine AI + Chart Data` → `Generate Final Report`
    - Configure logic to:
      - treat one merged item as AI data and the other as chart data
      - combine them
      - create an HTML email body including:
        - winner
        - summary
        - reason
        - advice
        - gold/equity allocations
        - strategy
        - chart image
      - output all combined fields plus `html`

14. **Add an IF node for threshold checking**
    - Node type: `IF`
    - Name: `Check Performance Gap`
    - Connect `Generate Final Report` → `Check Performance Gap`
    - Configure a numeric comparison:
      - left value: absolute numeric value of `difference` from `Calculate Performance Metrics`
      - operation: greater than
      - right value: `threshold` from `Set Analysis Parameters`
    - Recommended expression:
      - `={{Math.abs(Number($('Calculate Performance Metrics').first().json.difference))}}`
      - compared to
      - `={{$('Set Analysis Parameters').first().json.threshold}}`

15. **Add a Gmail node for the standard report**
    - Node type: `Gmail`
    - Name: `Send Report Email`
    - Credential: connect Gmail OAuth2
    - Set recipients as needed
    - Subject:
      - `Gold vs Equity Report ({{ $now }})`
    - Body:
      - `{{ $json.html }}`
    - Configure HTML email mode if your node version requires it

16. **Add a Gmail node for the alert report**
    - Node type: `Gmail`
    - Name: `Send Alert Email`
    - Credential: same Gmail OAuth2 or another Gmail account
    - Set recipients as needed
    - Subject:
      - `ALERT: Significant Performance Gap`
    - Body:
      - include an alert prefix plus the report HTML
    - Configure HTML mode if necessary

17. **Connect the IF branches carefully**
    - Intended logic:
      - **True branch** (`difference > threshold`) should go to `Send Alert Email`
      - **False branch** should go to `Send Report Email`
    - In the provided workflow JSON, the branches appear reversed. Correct this during rebuild.

18. **Add a Google Sheets node for report logging**
    - Node type: `Google Sheets`
    - Name: `Store Report History`
    - Credential: Google Sheets OAuth2
    - Operation: `Append`
    - Destination sheet: a history tab such as `Sheet3`
    - Create columns in the destination sheet:
      - `Date`
      - `Winner`
      - `Summary`
      - `Report`
    - Map values:
      - `Date = {{ $now }}`
      - `Winner = {{ $('Parse AI Output').first().json.winner }}`
      - `Summary = {{ $('Parse AI Output').first().json.summary }}`
      - `Report = {{ $('Parse AI Output').first().json.reason }}`

19. **Connect email nodes to logging**
    - `Send Report Email` → `Store Report History`
    - `Send Alert Email` → `Store Report History`

20. **Add optional sticky notes**
    - Add canvas notes for:
      - workflow purpose
      - data ingestion
      - AI insights
      - chart generation
      - report generation
      - email and audit logging

21. **Configure credentials**
    - **Google Sheets OAuth2**
      - Required for:
        - `Fetch Gold Prices`
        - `Fetch Equity Prices`
        - `Store Report History`
    - **Gmail OAuth2**
      - Required for:
        - `Send Report Email`
        - `Send Alert Email`
    - **Groq API**
      - Required for:
        - `Insights`

22. **Validate source sheet structure**
    - Gold and Equity tabs should each contain at least:
      - `Date`
      - `Price`
    - Keep data sorted by date ascending
    - Ensure row counts and date alignment are consistent if you keep the current index-based merge logic

23. **Test execution**
    - Run manually
    - Check:
      - merged data count
      - calculated returns
      - AI JSON validity
      - chart URL rendering
      - IF branch correctness
      - email delivery
      - logging append result

24. **Recommended hardening improvements**
    - Replace index-based asset matching with date-key matching
    - Add explicit date sorting before return calculation
    - Validate AI JSON with fallback defaults
    - Correct IF branch wiring if reproducing intended alert behavior
    - Reduce model temperature if JSON formatting errors occur
    - Shorten chart data or aggregate weekly if QuickChart URLs become too long

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed for manual execution but can be adapted to a scheduled trigger for regular monitoring. | General implementation note |
| The workflow description recommends compatibility with self-hosted n8n version 2.1.5 or newer. | Environment requirement |
| The AI layer uses Groq with the Llama model `llama-3.3-70b-versatile`. | AI model choice |
| QuickChart is used for chart rendering via URL-encoded Chart.js configuration. | https://quickchart.io/ |
| Suggested customizations include changing AI personality, chart aesthetics, email recipients, and replacing Google Sheets with market APIs. | General customization guidance |
| Suggested add-ons include Slack/Discord delivery, live financial APIs, and PDF generation. | Extension ideas |
| Need implementation help: WeblineIndia n8n automation team | https://www.weblineindia.com/hire-n8n-developers/ |
| Contact WeblineIndia | https://www.weblineindia.com/contact-us.html |

## Notable implementation observations
- The current merge logic pairs Gold and Equity rows by array index, not by explicit date matching. This is fragile if the two sheets differ in row order or missing dates.
- The IF node appears connected opposite to the intended alert behavior. Verify branch direction before production use.
- The Gmail nodes in the JSON do not visibly show recipient fields, so confirm recipient configuration after import.
- The final HTML report depends heavily on valid AI JSON. If the AI returns malformed content, the report and history log may degrade rather than stop cleanly.