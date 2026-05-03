Forecast sales trends and weekly reports with Stripe, Sheets, Slack, and Gemini AI

https://n8nworkflows.xyz/workflows/forecast-sales-trends-and-weekly-reports-with-stripe--sheets--slack--and-gemini-ai-13982


# Forecast sales trends and weekly reports with Stripe, Sheets, Slack, and Gemini AI

# 1. Workflow Overview

This workflow automates a weekly sales intelligence process using Stripe data, Google Sheets history, AI-driven analysis with Gemini, and multi-channel reporting through Slack, Gmail, and Notion.

Its primary purpose is to:
- Collect recent Stripe sales activity
- Aggregate and analyze revenue trends
- Estimate metrics such as week-over-week growth and MRR
- Detect significant changes or anomalies
- Generate AI-based business commentary
- Distribute reports and alerts to stakeholders

Typical use cases include:
- Weekly executive revenue reporting
- Early detection of unusual sales swings
- Lightweight revenue forecasting
- Maintaining a historical analysis log for future reporting

## 1.1 Trigger and Configuration
The workflow starts on a weekly schedule and passes through a configuration node intended as a central place for environment-specific values.

## 1.2 Data Collection
It retrieves Stripe charges, subscriptions, and refunds in parallel, while also loading historical sales data from Google Sheets.

## 1.3 Data Processing and Trend Computation
A Code node normalizes and merges the Stripe data, then another Code node calculates weekly metrics, growth rates, moving averages, MRR estimate, and anomaly indicators.

## 1.4 AI Analysis
A Gemini chat model is connected to an LLM chain node that generates structured JSON analysis from the computed trend metrics.

## 1.5 Decision and Distribution
An IF node checks whether a significant change occurred. The workflow always sends standard outputs on the true branch in this implementation, while the false branch sends an anomaly alert. This is logically inconsistent with the node naming and likely requires correction.

## 1.6 Reporting and Persistence
The workflow appends analysis results to Google Sheets, sends a Slack summary, emails leadership, and attempts to update a Notion dashboard.

---

# 2. Block-by-Block Analysis

## Block 1 — Trigger and Configuration

### Overview
This block launches the workflow every Monday at 9:00 UTC and routes execution through a Set node meant to act as a centralized configuration point. In the current JSON, the Set node does not actually define values, so it serves mainly as a fan-out connector.

### Nodes Involved
- Weekly Sales Analysis
- Configuration Settings

### Node Details

#### 1. Weekly Sales Analysis
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; scheduled entry point.
- **Configuration choices:** Configured to run on a cron schedule: every Monday at 09:00 UTC.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none, as it is a trigger node
  - Output: Configuration Settings
- **Version-specific requirements:** Uses Schedule Trigger version `1.2`.
- **Edge cases or potential failure types:**
  - Workflow inactive, so schedule never fires
  - Timezone misunderstandings if the team expects local time instead of UTC
- **Sub-workflow reference:** None.

#### 2. Configuration Settings
- **Type and technical role:** `n8n-nodes-base.set`; intended as a central configuration node.
- **Configuration choices:** No fields are currently defined. Despite the sticky note suggesting centralized IDs and credentials, the actual node is empty.
- **Key expressions or variables used:** None in the current state.
- **Input and output connections:**
  - Input: Weekly Sales Analysis
  - Outputs: Get Stripe Charges, Get Stripe Subscriptions, Get Stripe Refunds, Load Historical Sales
- **Version-specific requirements:** Uses Set node version `3.4`.
- **Edge cases or potential failure types:**
  - Misleading design: users may assume IDs are stored here, but they are hardcoded elsewhere
  - If later converted into a true config node, downstream expressions must be updated to reference it
- **Sub-workflow reference:** None.

---

## Block 2 — Data Collection

### Overview
This block retrieves operational sales data from Stripe and historical records from Google Sheets. All four retrieval nodes run in parallel from the configuration fan-out.

### Nodes Involved
- Get Stripe Charges
- Get Stripe Subscriptions
- Get Stripe Refunds
- Load Historical Sales

### Node Details

#### 3. Get Stripe Charges
- **Type and technical role:** `n8n-nodes-base.httpRequest`; API call to Stripe charges endpoint.
- **Configuration choices:**
  - URL: `https://api.stripe.com/v1/charges`
  - Authentication: generic credential type using HTTP Header Auth
  - No query parameters are configured, so it currently fetches Stripe’s default page rather than explicitly filtering to the last 7 days
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: Configuration Settings
  - Output: Merge and Process Data
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Missing or invalid Stripe API credential
  - Pagination not handled; only first page likely retrieved
  - No date filtering, despite workflow description saying “last 7 days”
  - Stripe rate limits or network errors
- **Sub-workflow reference:** None.

#### 4. Get Stripe Subscriptions
- **Type and technical role:** `n8n-nodes-base.httpRequest`; API call to Stripe subscriptions endpoint.
- **Configuration choices:**
  - URL: `https://api.stripe.com/v1/subscriptions`
  - Authentication: HTTP Header Auth via generic credential
  - No filters or pagination configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: Configuration Settings
  - Output: Merge and Process Data
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid Stripe auth
  - Subscription list may include records outside the intended reporting range
  - No pagination support
  - Data model assumptions in downstream code may fail if `items.data[0]` is missing
- **Sub-workflow reference:** None.

#### 5. Get Stripe Refunds
- **Type and technical role:** `n8n-nodes-base.httpRequest`; API call to Stripe refunds endpoint.
- **Configuration choices:**
  - URL: `https://api.stripe.com/v1/refunds`
  - Authentication: HTTP Header Auth via generic credential
  - No date filtering or pagination configured
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: Configuration Settings
  - Output: Merge and Process Data
- **Version-specific requirements:** HTTP Request version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid Stripe auth
  - Pagination not handled
  - Refund structure may not contain expanded charge/customer context as assumed later
- **Sub-workflow reference:** None.

#### 6. Load Historical Sales
- **Type and technical role:** `n8n-nodes-base.googleSheets`; loads historical sales data from a Google Sheet.
- **Configuration choices:**
  - Document ID is placeholder: `YOUR_SPREADSHEET_ID`
  - Sheet name: `Historical_Sales_Data`
  - Authentication: Google OAuth2
  - Operation is implicitly read/list depending on node defaults, but the JSON does not specify advanced filters
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: Configuration Settings
  - Output: Merge and Process Data
- **Version-specific requirements:** Google Sheets version `4.5`.
- **Edge cases or potential failure types:**
  - Spreadsheet ID not replaced
  - OAuth credentials missing or unauthorized
  - Sheet name mismatch
  - Downstream code expects `json.values`, which may not match actual output format depending on operation/settings
- **Sub-workflow reference:** None.

---

## Block 3 — Data Processing and Metric Calculation

### Overview
This block merges Stripe objects into a normalized sales dataset and computes business metrics such as weekly revenue, week-over-week growth, moving average, estimated MRR, and anomalies.

### Nodes Involved
- Merge and Process Data
- Calculate Trends

### Node Details

#### 7. Merge and Process Data
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript transformation and aggregation.
- **Configuration choices:**
  - Reads the four upstream inputs using positional access:
    - first input = charges
    - second = subscriptions
    - third = refunds
    - fourth = historical data
  - Normalizes Stripe data into common records with date, amount, currency, customer, type, and status
  - Aggregates totals by date
  - Returns:
    - `processed_sales_data`
    - `raw_data`
- **Key expressions or variables used:**
  - `$input.first().json.data`
  - `$input.all()[1]?.json.data`
  - `$input.all()[2]?.json.data`
  - `$input.all()[3]?.json.values`
- **Input and output connections:**
  - Inputs: Get Stripe Charges, Get Stripe Subscriptions, Get Stripe Refunds, Load Historical Sales
  - Output: Calculate Trends
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Reliance on input order is brittle
  - Historical data is loaded but not actually merged into calculations
  - Refund mapping uses `refund.charge?.customer`, but Stripe refunds typically do not expose nested customer unless charge is expanded
  - `dailyTotals[item.type + 's']` creates `refunds`, `charges`, `subscriptions`; this works only because those exact keys are initialized
  - Currency mixing is not handled; all amounts are summed as if same currency
  - Empty API responses produce empty arrays and later metrics may become misleading
- **Sub-workflow reference:** None.

#### 8. Calculate Trends
- **Type and technical role:** `n8n-nodes-base.code`; computes reporting metrics from normalized daily sales data.
- **Configuration choices:**
  - Defines current week as all records from now minus 7 days onward
  - Defines previous week as the 7 days before that
  - Calculates:
    - current week revenue
    - current week transaction count
    - previous week revenue
    - week-over-week growth
    - 7-day moving average
    - estimated MRR from recent subscription totals
    - anomaly list
    - `has_significant_change` when absolute WoW growth exceeds 20%
- **Key expressions or variables used:**
  - `$json.processed_sales_data`
- **Input and output connections:**
  - Input: Merge and Process Data
  - Output: Generate Sales Analysis
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - If there are fewer than 7 days, moving average divides by 7 anyway
  - If previous 7-day average is 0, anomaly division can produce `Infinity` or `NaN`
  - `currentWeekTransactions` counts daily rows, not individual transactions
  - MRR estimate is simplistic and based on summed subscription amounts from recent daily totals, not active recurring commitments
  - Historical data from Google Sheets is not incorporated, despite description implying it is
- **Sub-workflow reference:** None.

---

## Block 4 — AI Analysis

### Overview
This block uses a Gemini chat model connected to an LLM chain to transform computed metrics into structured executive commentary and forecasting in JSON format.

### Nodes Involved
- Gemini AI Model
- Generate Sales Analysis

### Node Details

#### 9. Gemini AI Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; provides the language model backend.
- **Configuration choices:** No model-specific options are shown in the JSON beyond defaults.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - AI language model output to: Generate Sales Analysis
- **Version-specific requirements:** Version `1`.
- **Edge cases or potential failure types:**
  - Missing Gemini/Google AI credentials
  - Model quota/rate limits
  - Output variability despite prompt requesting strict JSON
- **Sub-workflow reference:** None.

#### 10. Generate Sales Analysis
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; prompts the LLM with trend metrics and requests JSON output.
- **Configuration choices:**
  - Prompt includes:
    - current week revenue
    - previous week revenue
    - growth rate
    - 7-day moving average
    - estimated MRR
    - anomaly count and details
    - significant change flag
  - Requires the LLM to return JSON only in a defined schema:
    - executive summary
    - key insights
    - trend analysis
    - forecasting
    - risk assessment
    - recommendations
- **Key expressions or variables used:**
  - `{{ $json.current_week.revenue }}`
  - `{{ $json.previous_week.revenue }}`
  - `{{ $json.growth_metrics.week_over_week_growth }}`
  - `{{ $json.anomalies.length }}`
  - Template loops and conditions over anomalies
- **Input and output connections:**
  - Main input: Calculate Trends
  - AI model input: Gemini AI Model
  - Main output: Check for Anomalies
- **Version-specific requirements:** Chain LLM version `1.4`.
- **Edge cases or potential failure types:**
  - LLM may return invalid JSON despite prompt
  - Downstream nodes use `JSON.parse(...response)` and will fail if format is not exact
  - Template syntax must be supported as written; mixed templating constructs can behave differently across n8n versions
- **Sub-workflow reference:** None.

---

## Block 5 — Decision and Standard Outputs

### Overview
This block decides whether to branch based on `has_significant_change`, then sends the standard weekly outputs and persistence actions. However, the current true/false wiring appears reversed relative to node names and intended behavior.

### Nodes Involved
- Check for Anomalies
- Log to Historical Sheet
- Update Notion Dashboard
- Send Weekly Report to Slack
- Email Executive Report

### Node Details

#### 11. Check for Anomalies
- **Type and technical role:** `n8n-nodes-base.if`; conditional branching.
- **Configuration choices:**
  - Condition: `{{ $json.has_significant_change }}` equals `true`
- **Key expressions or variables used:**
  - `={{ $json.has_significant_change }}`
- **Input and output connections:**
  - Input: Generate Sales Analysis
  - True output: Log to Historical Sheet, Update Notion Dashboard, Send Weekly Report to Slack, Email Executive Report
  - False output: Send Anomaly Alert
- **Version-specific requirements:** IF node version `2`.
- **Edge cases or potential failure types:**
  - Logical inconsistency: false branch sends anomaly alert, while true branch sends routine outputs
  - `has_significant_change` is not present in the LLM output; the IF node receives data from Generate Sales Analysis, so unless the chain preserves input fields, this may fail or evaluate unexpectedly
- **Sub-workflow reference:** None.

#### 12. Log to Historical Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends the weekly analysis summary to a logging sheet.
- **Configuration choices:**
  - Operation: append
  - Spreadsheet ID: `YOUR_SPREADSHEET_ID`
  - Sheet name: `Sales_Analysis_Log`
  - Maps columns:
    - date
    - growth_rate
    - mrr_estimate
    - anomalies_count
    - analysis_summary
    - current_week_revenue
    - previous_week_revenue
  - Pulls values from Calculate Trends and Generate Sales Analysis by node reference
- **Key expressions or variables used:**
  - `$now.format('yyyy-MM-dd')`
  - `$('Calculate Trends').item.json...`
  - `JSON.parse($('Generate Sales Analysis').item.json.response).executive_summary`
- **Input and output connections:**
  - Input: Check for Anomalies (true branch)
  - Output: none
- **Version-specific requirements:** Google Sheets version `4.5`.
- **Edge cases or potential failure types:**
  - Spreadsheet ID placeholder not replaced
  - Append fails if columns or headers mismatch
  - Invalid JSON from LLM breaks `JSON.parse`
- **Sub-workflow reference:** None.

#### 13. Update Notion Dashboard
- **Type and technical role:** `n8n-nodes-base.notion`; intended dashboard update.
- **Configuration choices:**
  - Operation: update
  - No target page/database properties are configured in the JSON shown
- **Key expressions or variables used:** None shown.
- **Input and output connections:**
  - Input: Check for Anomalies (true branch)
  - Output: none
- **Version-specific requirements:** Notion version `2.2`.
- **Edge cases or potential failure types:**
  - Incomplete configuration; as-is, this node is not reproducible without manually setting page/database target and fields
  - Missing Notion credentials or insufficient permissions
- **Sub-workflow reference:** None.

#### 14. Send Weekly Report to Slack
- **Type and technical role:** `n8n-nodes-base.slack`; posts the weekly summary to a Slack channel.
- **Configuration choices:**
  - Authentication: Slack OAuth2
  - Channel selection: fixed channel ID placeholder `YOUR_SALES_CHANNEL`
  - Message includes metrics and parsed AI summary plus top three insights
  - Includes placeholder Notion link: `YOUR_NOTION_URL`
- **Key expressions or variables used:**
  - `$('Calculate Trends').item.json...`
  - `JSON.parse($('Generate Sales Analysis').item.json.response)...`
- **Input and output connections:**
  - Input: Check for Anomalies (true branch)
  - Output: none
- **Version-specific requirements:** Slack node version `2.2`.
- **Edge cases or potential failure types:**
  - Invalid Slack OAuth token
  - Invalid channel ID
  - JSON parsing failure if AI output is malformed
  - Slack formatting may render differently because Markdown support varies by field
- **Sub-workflow reference:** None.

#### 15. Email Executive Report
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the executive email report.
- **Configuration choices:**
  - Subject includes formatted current date
  - Message body uses AI-generated executive summary, recommendations, and risk assessment
  - Includes placeholder Notion dashboard URL variable text rather than actual expression-backed config
  - No recipient field is visible in the provided JSON; likely incomplete
- **Key expressions or variables used:**
  - `JSON.parse($('Generate Sales Analysis').item.json.response)...`
  - `$now.format('MMM dd, yyyy')`
- **Input and output connections:**
  - Input: Check for Anomalies (true branch)
  - Output: none
- **Version-specific requirements:** Gmail node version `2.1`.
- **Edge cases or potential failure types:**
  - Missing recipient configuration
  - Gmail OAuth credentials missing
  - The expression using `| upper` may not work as expected outside supported template filters
  - JSON parsing failure on malformed AI output
- **Sub-workflow reference:** None.

---

## Block 6 — Alerting

### Overview
This block sends a Slack alert intended for anomalous sales changes. Due to the current IF wiring, it is triggered on the false branch, which likely contradicts intended behavior.

### Nodes Involved
- Send Anomaly Alert

### Node Details

#### 16. Send Anomaly Alert
- **Type and technical role:** `n8n-nodes-base.slack`; posts an urgent alert to an alerts channel.
- **Configuration choices:**
  - Channel ID placeholder: `YOUR_ALERTS_CHANNEL`
  - Message highlights unusual revenue spike or decline based on growth direction
  - Uses AI risk level and top two recommendations
  - Mentions `@channel`
- **Key expressions or variables used:**
  - `$('Calculate Trends').item.json...`
  - `JSON.parse($('Generate Sales Analysis').item.json.response)...`
- **Input and output connections:**
  - Input: Check for Anomalies (false branch)
  - Output: none
- **Version-specific requirements:** Slack node version `2.2`.
- **Edge cases or potential failure types:**
  - Most important issue is branch inversion: likely sent when there is *no* significant change
  - Invalid Slack auth or channel ID
  - JSON parsing failure if LLM output is malformed
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Sales Analysis | Schedule Trigger | Weekly workflow entry point |  | Configuration Settings | ## Data collection Scheduled trigger fetches last 7 days of Stripe data (charges, subscriptions, refunds) and loads historical sales records from Google Sheets. |
| Configuration Settings | Set | Fan-out node intended for centralized configuration | Weekly Sales Analysis | Get Stripe Charges; Get Stripe Subscriptions; Get Stripe Refunds; Load Historical Sales | ## Data collection Scheduled trigger fetches last 7 days of Stripe data (charges, subscriptions, refunds) and loads historical sales records from Google Sheets. |
| Get Stripe Charges | HTTP Request | Fetch Stripe charges | Configuration Settings | Merge and Process Data | ## Data collection Scheduled trigger fetches last 7 days of Stripe data (charges, subscriptions, refunds) and loads historical sales records from Google Sheets. |
| Get Stripe Subscriptions | HTTP Request | Fetch Stripe subscriptions | Configuration Settings | Merge and Process Data | ## Data collection Scheduled trigger fetches last 7 days of Stripe data (charges, subscriptions, refunds) and loads historical sales records from Google Sheets. |
| Get Stripe Refunds | HTTP Request | Fetch Stripe refunds | Configuration Settings | Merge and Process Data | ## Data collection Scheduled trigger fetches last 7 days of Stripe data (charges, subscriptions, refunds) and loads historical sales records from Google Sheets. |
| Load Historical Sales | Google Sheets | Load historical records from spreadsheet | Configuration Settings | Merge and Process Data | ## Data collection Scheduled trigger fetches last 7 days of Stripe data (charges, subscriptions, refunds) and loads historical sales records from Google Sheets. |
| Merge and Process Data | Code | Normalize and aggregate Stripe data by date | Get Stripe Charges; Get Stripe Subscriptions; Get Stripe Refunds; Load Historical Sales | Calculate Trends | ## Processing & analysis Code nodes merge and calculate daily totals, trends, MRR estimates. Gemini AI provides comprehensive insights, forecasting, and strategic recommendations. |
| Calculate Trends | Code | Compute weekly metrics, growth, MRR, anomalies | Merge and Process Data | Generate Sales Analysis | ## Processing & analysis Code nodes merge and calculate daily totals, trends, MRR estimates. Gemini AI provides comprehensive insights, forecasting, and strategic recommendations. |
| Gemini AI Model | Google Gemini Chat Model | Supplies LLM backend |  | Generate Sales Analysis | ## Processing & analysis Code nodes merge and calculate daily totals, trends, MRR estimates. Gemini AI provides comprehensive insights, forecasting, and strategic recommendations. |
| Generate Sales Analysis | LLM Chain | Generate structured AI analysis in JSON | Calculate Trends; Gemini AI Model | Check for Anomalies | ## Processing & analysis Code nodes merge and calculate daily totals, trends, MRR estimates. Gemini AI provides comprehensive insights, forecasting, and strategic recommendations. |
| Check for Anomalies | IF | Branch on significant change flag | Generate Sales Analysis | Log to Historical Sheet; Update Notion Dashboard; Send Weekly Report to Slack; Email Executive Report; Send Anomaly Alert | ## Output & alerts IF node detects significant changes, triggers alerts via Slack. All results logged to Sheets, dashboard updated in Notion, weekly reports sent via Slack and Gmail. |
| Log to Historical Sheet | Google Sheets | Append weekly analysis to history log | Check for Anomalies |  | ## Output & alerts IF node detects significant changes, triggers alerts via Slack. All results logged to Sheets, dashboard updated in Notion, weekly reports sent via Slack and Gmail. |
| Update Notion Dashboard | Notion | Update reporting dashboard | Check for Anomalies |  | ## Output & alerts IF node detects significant changes, triggers alerts via Slack. All results logged to Sheets, dashboard updated in Notion, weekly reports sent via Slack and Gmail. |
| Send Weekly Report to Slack | Slack | Post weekly sales summary | Check for Anomalies |  | ## Output & alerts IF node detects significant changes, triggers alerts via Slack. All results logged to Sheets, dashboard updated in Notion, weekly reports sent via Slack and Gmail. |
| Email Executive Report | Gmail | Email leadership report | Check for Anomalies |  | ## Output & alerts IF node detects significant changes, triggers alerts via Slack. All results logged to Sheets, dashboard updated in Notion, weekly reports sent via Slack and Gmail. |
| Send Anomaly Alert | Slack | Post anomaly alert to Slack | Check for Anomalies |  | ## Output & alerts IF node detects significant changes, triggers alerts via Slack. All results logged to Sheets, dashboard updated in Notion, weekly reports sent via Slack and Gmail. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Forecast sales trends and weekly reports with Stripe, Sheets, Slack, and Gemini AI`.

2. **Add a Schedule Trigger node** named `Weekly Sales Analysis`.
   - Set timezone to `UTC`.
   - Configure cron schedule for every Monday at 09:00.
   - Equivalent cron: `0 9 * * 1`.

3. **Add a Set node** named `Configuration Settings`.
   - Connect `Weekly Sales Analysis` to it.
   - In the provided workflow this node is empty.
   - Recommended improvement: define reusable fields such as:
     - spreadsheet_id
     - historical_sheet_name
     - log_sheet_name
     - notion_page_id
     - sales_slack_channel
     - alerts_slack_channel
     - notion_url
   - If you do this, update downstream nodes to reference these values instead of hardcoded placeholders.

4. **Add an HTTP Request node** named `Get Stripe Charges`.
   - Connect from `Configuration Settings`.
   - Method: `GET`
   - URL: `https://api.stripe.com/v1/charges`
   - Authentication: `Generic Credential Type` → `HTTP Header Auth`
   - Create credential with header:
     - `Authorization: Bearer YOUR_STRIPE_SECRET_KEY`
   - Recommended improvement: add query parameters such as:
     - `limit=100`
     - `created[gte]=<unix timestamp for 7 days ago>`
   - Enable pagination manually if needed.

5. **Add another HTTP Request node** named `Get Stripe Subscriptions`.
   - Connect from `Configuration Settings`.
   - Method: `GET`
   - URL: `https://api.stripe.com/v1/subscriptions`
   - Use the same Stripe header credential.
   - Recommended: add status/date filters if relevant to your reporting logic.

6. **Add another HTTP Request node** named `Get Stripe Refunds`.
   - Connect from `Configuration Settings`.
   - Method: `GET`
   - URL: `https://api.stripe.com/v1/refunds`
   - Use the same Stripe header credential.
   - Recommended: add `created[gte]` and pagination.

7. **Add a Google Sheets node** named `Load Historical Sales`.
   - Connect from `Configuration Settings`.
   - Authentication: Google OAuth2.
   - Spreadsheet ID: replace `YOUR_SPREADSHEET_ID`.
   - Sheet name: `Historical_Sales_Data`.
   - Configure it to read rows from the sheet.
   - Ensure the output structure matches what your processing code expects. If necessary, adapt the code to consume actual returned rows rather than `json.values`.

8. **Add a Code node** named `Merge and Process Data`.
   - Connect the outputs of:
     - `Get Stripe Charges`
     - `Get Stripe Subscriptions`
     - `Get Stripe Refunds`
     - `Load Historical Sales`
   - Paste the JavaScript logic that:
     - reads charges, subscriptions, refunds, historical rows
     - transforms them into normalized records
     - groups by date
     - calculates daily totals
     - returns `processed_sales_data` and `raw_data`
   - Important: preserve input order if you keep positional access with `$input.all()[index]`.
   - Recommended improvement: use named merges or guard clauses so node order changes do not break the code.

9. **Add a Code node** named `Calculate Trends`.
   - Connect `Merge and Process Data` to it.
   - Paste the JavaScript that computes:
     - current week revenue
     - previous week revenue
     - week-over-week growth
     - 7-day moving average
     - estimated MRR
     - anomalies
     - `has_significant_change`
   - Recommended improvements:
     - protect against divide-by-zero
     - count transactions from raw records rather than daily aggregates
     - incorporate historical sheet data if intended

10. **Add a Google Gemini Chat Model node** named `Gemini AI Model`.
    - Configure Google/Gemini credentials.
    - Select the desired Gemini model available in your n8n environment.
    - Leave options default unless you need temperature/max token tuning.

11. **Add an LLM Chain node** named `Generate Sales Analysis`.
    - Connect `Calculate Trends` to its main input.
    - Connect `Gemini AI Model` to its AI language model input.
    - Set prompt type to define text manually.
    - Paste a prompt equivalent to the provided one:
      - include revenue metrics
      - include anomaly details
      - request strict JSON only
      - require keys:
        - executive_summary
        - key_insights
        - trend_analysis
        - forecasting
        - risk_assessment
        - recommendations
    - Test carefully to verify the model returns valid JSON.

12. **Add an IF node** named `Check for Anomalies`.
    - Connect `Generate Sales Analysis` to it.
    - Add condition:
      - left value: `={{ $json.has_significant_change }}`
      - operator: equals
      - right value: `true`
    - Important correction: because the IF currently sits after the LLM node, make sure `has_significant_change` is still present in the item reaching this node. If not:
      - either move the IF immediately after `Calculate Trends`
      - or merge the trend data with the LLM output before the IF
    - Recommended correction: true branch should go to `Send Anomaly Alert`, and standard reporting can happen independently or on both branches.

13. **Add a Google Sheets node** named `Log to Historical Sheet`.
    - Connect from the standard-reporting branch.
    - Authentication: Google OAuth2.
    - Operation: `Append`.
    - Spreadsheet ID: same spreadsheet or another dedicated one.
    - Sheet name: `Sales_Analysis_Log`.
    - Map columns:
      - `date` → current date
      - `growth_rate` → from `Calculate Trends`
      - `mrr_estimate` → from `Calculate Trends`
      - `anomalies_count` → from `Calculate Trends`
      - `analysis_summary` → parsed `executive_summary` from the LLM response
      - `current_week_revenue`
      - `previous_week_revenue`
    - Ensure the target sheet has matching headers.

14. **Add a Notion node** named `Update Notion Dashboard`.
    - Connect from the standard-reporting branch.
    - Authentication: Notion credential with access to the target page/database.
    - Operation: `Update`.
    - Manually configure the target object because the JSON does not include enough detail.
    - Typical setup:
      - target page/database entry ID
      - fields/properties such as revenue, growth rate, MRR, summary, anomaly count, report date

15. **Add a Slack node** named `Send Weekly Report to Slack`.
    - Connect from the standard-reporting branch.
    - Authentication: Slack OAuth2.
    - Select fixed channel.
    - Replace `YOUR_SALES_CHANNEL` with the real channel ID.
    - Compose a message containing:
      - current revenue
      - growth rate
      - MRR
      - transaction count
      - executive summary
      - top 3 insights
      - Notion dashboard link
    - If needed, simplify formatting for Slack compatibility.

16. **Add a Gmail node** named `Email Executive Report`.
    - Connect from the standard-reporting branch.
    - Authentication: Gmail OAuth2.
    - Set recipients explicitly; the provided JSON does not include them.
    - Set subject similar to:
      - `Weekly Sales Analysis - {{ $now.format('MMM dd, yyyy') }}`
    - Use a body including:
      - executive summary
      - key metrics
      - recommendations
      - risk assessment
      - dashboard URL
    - Validate all expressions, especially any uppercase formatting logic.

17. **Add a Slack node** named `Send Anomaly Alert`.
    - Connect from the anomaly branch.
    - Authentication: Slack OAuth2.
    - Select fixed channel and replace `YOUR_ALERTS_CHANNEL`.
    - Create an alert message including:
      - spike/decline wording based on sign of growth
      - current vs previous week
      - percent change
      - risk level
      - top 2 recommendations
      - optional `@channel`

18. **Connect the workflow exactly as intended.**
    - Current JSON wiring:
      - `Check for Anomalies` true → Log to Historical Sheet, Update Notion Dashboard, Send Weekly Report to Slack, Email Executive Report
      - `Check for Anomalies` false → Send Anomaly Alert
    - Recommended corrected wiring:
      - true → Send Anomaly Alert
      - both branches or a separate always-run route → log sheet, Notion, weekly Slack report, Gmail report

19. **Configure credentials.**
    - **Stripe:** HTTP Header Auth with secret key bearer token
    - **Google Sheets:** Google OAuth2
    - **Gemini:** Google AI / Gemini credential
    - **Slack:** Slack OAuth2 with permission to post messages to selected channels
    - **Gmail:** Gmail OAuth2 with send permission
    - **Notion:** Notion integration token with page/database access

20. **Replace all placeholders.**
    - `YOUR_SPREADSHEET_ID`
    - `YOUR_SALES_CHANNEL`
    - `YOUR_ALERTS_CHANNEL`
    - `YOUR_NOTION_URL`
    - `YOUR_NOTION_DASHBOARD_URL`
    - any missing Notion record/page ID
    - Gmail recipients

21. **Test each section incrementally.**
    - First test Stripe calls
    - Then test sheet loading
    - Then inspect Code node outputs
    - Then validate LLM response JSON
    - Finally test Slack/Gmail/Notion outputs

22. **Add robustness improvements before production use.**
    - Pagination for Stripe endpoints
    - Explicit last-7-days filtering
    - JSON validation after LLM response
    - fallback behavior when AI output is invalid
    - proper merge of historical Google Sheets data
    - consistent currency handling
    - corrected anomaly branch logic

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ### 📊 Automated Sales Intelligence Transform your Stripe payment data into actionable business insights with AI-powered analysis and automated reporting. | Main workflow description |
| 1. **Data Collection**: Pulls weekly Stripe charges, subscriptions, and refunds | Main workflow description |
| 2. **Historical Context**: Merges with Google Sheets historical data | Main workflow description |
| 3. **Trend Analysis**: Calculates growth rates, MRR, and moving averages | Main workflow description |
| 4. **AI Insights**: Gemini AI analyzes patterns and predicts future performance | Main workflow description |
| 5. **Smart Alerts**: Automatically detects revenue anomalies (±50% variance) | Main workflow description |
| 6. **Multi-Channel Delivery**: Updates Notion dashboard, posts to Slack, emails executives | Main workflow description |
| Setup steps noted in the workflow: configure Stripe API credentials in HTTP Request nodes | Main workflow description |
| Set Google Sheets ID in Configuration Settings | Main workflow description; note that the actual Set node is currently empty |
| Update Notion page ID and Slack channel IDs | Main workflow description |
| Test with Gmail SMTP credentials | Main workflow description; note that the actual node is Gmail OAuth-based, not SMTP |
| Customize alert thresholds in Calculate Trends node | Main workflow description |
| **Runs every Monday at 9 AM UTC** | Main workflow description |

## Additional Implementation Notes
- The workflow contains **one entry point**: `Weekly Sales Analysis`.
- The workflow contains **no sub-workflow nodes** and does not invoke other workflows.
- Several placeholders and incomplete node configurations must be completed before use.
- The current branch logic for anomalies appears reversed and should be reviewed.
- The historical Google Sheets data is fetched but not meaningfully incorporated into downstream calculations in the current code.