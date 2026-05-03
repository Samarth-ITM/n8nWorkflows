Analyze Shopify orders with Gemini AI and send weekly Slack insights

https://n8nworkflows.xyz/workflows/analyze-shopify-orders-with-gemini-ai-and-send-weekly-slack-insights-13872


# Analyze Shopify orders with Gemini AI and send weekly Slack insights

## 1. Workflow Overview

This workflow performs a **weekly Shopify sales review** and distributes the results through **Slack**, **email**, and **Google Sheets**, while also generating AI-based commentary using **Google Gemini**.

Its main purpose is to help a team monitor recent sales performance, detect negative changes, and receive summarized business insights without manually reviewing Shopify orders.

### 1.1 Scheduled Input Reception
The workflow starts every Monday at 09:00 using a schedule trigger. It then requests Shopify order data intended to represent the last 7 days of sales.

### 1.2 Sales Data Aggregation
The retrieved Shopify orders are processed in a Code node that calculates:
- total revenue
- order count
- average order value
- top-selling products
- daily sales totals
- a simulated previous-week revenue
- percentage revenue change

### 1.3 AI Processing with Gemini
The aggregated metrics are sent to the Gemini API through an HTTP Request node. Gemini is prompted to analyze the week’s performance and return structured JSON containing:
- insights
- trends
- recommendations
- risks

### 1.4 Report Preparation and Distribution
The AI response is parsed and combined with the computed weekly metrics. The workflow then:
- posts a weekly sales summary to Slack
- sends an email report to stakeholders
- appends metrics to Google Sheets

### 1.5 Revenue Drop Alerting
After logging the data, the workflow checks whether revenue dropped by more than 20% versus the previous week. If so, it sends a Slack alert to a dedicated alerts channel.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

**Overview:**  
This block defines when the workflow runs and starts the weekly reporting sequence. It is the single entry point of the workflow.

**Nodes Involved:**
- Weekly Monday Trigger

### Node Details

#### Weekly Monday Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger that launches the workflow automatically.
- **Configuration choices:**  
  Configured with cron expression `0 9 * * MON`, which means **every Monday at 09:00**.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get Last 7 Days Orders`
- **Version-specific requirements:**  
  Uses node type version `1.2`. Cron behavior depends on the workflow/server timezone configured in n8n.
- **Edge cases or potential failure types:**  
  - Runs in server timezone, which may differ from business timezone
  - Daylight saving changes may affect expected reporting hour
  - If the n8n instance is down at trigger time, execution may be missed depending on deployment setup
- **Sub-workflow reference:**  
  None.

---

## 2.2 Data Collection

**Overview:**  
This block retrieves Shopify orders that will feed the analysis stage. The node is named as if it fetches the last 7 days, but its actual configuration does not currently enforce a date filter.

**Nodes Involved:**
- Get Last 7 Days Orders

### Node Details

#### Get Last 7 Days Orders
- **Type and technical role:** `n8n-nodes-base.shopify`  
  Fetches order records from Shopify.
- **Configuration choices:**  
  - Operation: `getAll`
  - Limit: `500`
  - No extra options or filters are configured
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Weekly Monday Trigger`
  - Output: `Aggregate Sales Data`
- **Version-specific requirements:**  
  Uses node version `1`. Requires valid Shopify credentials with permission to read orders.
- **Edge cases or potential failure types:**  
  - **Important mismatch:** despite the node name, there is **no date filter** set for the last 7 days
  - If store has more than 500 matching orders, data may be truncated depending on pagination behavior and node settings
  - Auth failures if Shopify credentials are invalid or expired
  - API throttling/rate-limit issues on high-volume stores
  - Missing `line_items`, `total_price`, or `created_at` fields can break downstream assumptions
- **Sub-workflow reference:**  
  None.

---

## 2.3 Sales Data Aggregation

**Overview:**  
This block converts raw Shopify orders into summarized business metrics suitable for reporting and AI interpretation. It also fabricates a previous-week revenue benchmark for comparison.

**Nodes Involved:**
- Aggregate Sales Data

### Node Details

#### Aggregate Sales Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Runs JavaScript to aggregate all incoming order items into a single metrics object.
- **Configuration choices:**  
  The script:
  - loads all incoming orders with `$input.all()`
  - sums `total_price`
  - counts orders
  - groups revenue by order date
  - counts sold quantities by product name from `line_items`
  - selects top 5 products
  - computes average order value
  - simulates previous week revenue as `current revenue * 0.95`
  - computes revenue change percentage
- **Key expressions or variables used:**  
  - `$input.all()`
  - `order.json.total_price`
  - `order.json.created_at`
  - `order.json.line_items`
  - Returned fields:
    - `totalRevenue`
    - `orderCount`
    - `avgOrderValue`
    - `topProducts`
    - `dailySales`
    - `previousWeekRevenue`
    - `revenueChange`
- **Input and output connections:**  
  - Input: `Get Last 7 Days Orders`
  - Output: `Gemini Sales Analysis`
- **Version-specific requirements:**  
  Uses Code node version `2`.
- **Edge cases or potential failure types:**  
  - If `created_at` is missing or not a string, `split('T')` will fail
  - If order values are non-numeric, `parseFloat` can return `NaN`
  - If no orders are returned, order count becomes `0`; average order value safely becomes `0`
  - `previousWeekRevenue` is synthetic, not historical
  - Because previous-week revenue is calculated as 95% of current revenue, `revenueChange` will almost always be about `5.3%`, making the alert path effectively non-functional unless logic is changed
- **Sub-workflow reference:**  
  None.

---

## 2.4 AI Processing with Gemini

**Overview:**  
This block sends the summarized sales metrics to the Gemini API and requests structured analytical output. The next node assumes Gemini returns valid JSON in plain text form.

**Nodes Involved:**
- Gemini Sales Analysis
- Parse AI Response

### Node Details

#### Gemini Sales Analysis
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Generative Language API directly.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash-latest:generateContent`
  - Authentication: generic credential type using HTTP header auth
  - Header set: `Content-Type: application/json`
  - Body includes a `contents` payload with a long prompt containing weekly metrics
  - Prompt asks Gemini to return JSON with keys:
    - `insights`
    - `trends`
    - `recommendations`
    - `risks`
- **Key expressions or variables used:**  
  In the request body:
  - `$json.totalRevenue`
  - `$json.orderCount`
  - `$json.avgOrderValue`
  - `$json.revenueChange`
  - `$json.topProducts.map(...)`
  - `Object.entries($json.dailySales).map(...)`
- **Input and output connections:**  
  - Input: `Aggregate Sales Data`
  - Output: `Parse AI Response`
- **Version-specific requirements:**  
  Uses HTTP Request node version `4.2`. Requires an HTTP Header Auth credential, typically with an API key header expected by the Gemini API.
- **Edge cases or potential failure types:**  
  - Gemini API authentication may be configured incorrectly if the header name/value is not set exactly as required
  - The prompt requests JSON, but models may still return markdown, commentary, or malformed JSON
  - Large prompt payloads may fail on high-volume weekly summaries
  - API quota/rate-limit errors
  - Network timeout or 4xx/5xx API failures
- **Sub-workflow reference:**  
  None.

#### Parse AI Response
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts and reshapes the Gemini response into a cleaner object for downstream reporting.
- **Configuration choices:**  
  Creates two fields:
  - `aiInsights`: parsed from `candidates[0].content.parts[0].text` using `JSON.parse(...)`
  - `weeklyMetrics`: copied from the `Aggregate Sales Data` node output
- **Key expressions or variables used:**  
  - `={{ JSON.parse($json.candidates[0].content.parts[0].text) }}`
  - `={{ $('Aggregate Sales Data').item.json }}`
- **Input and output connections:**  
  - Input: `Gemini Sales Analysis`
  - Outputs:
    - `Post to Slack Channel`
    - `Email Report to Stakeholders`
    - `Log Metrics to Google Sheets`
- **Version-specific requirements:**  
  Uses Set node version `3.4`.
- **Edge cases or potential failure types:**  
  - `JSON.parse(...)` will fail if Gemini returns non-JSON text
  - If the Gemini response shape changes, `candidates[0].content.parts[0].text` may not exist
  - Referencing another node with `$('Aggregate Sales Data').item.json` assumes one coherent item context
- **Sub-workflow reference:**  
  None.

---

## 2.5 Reporting and Metrics Logging

**Overview:**  
This block distributes the weekly analysis to communication channels and writes data to Google Sheets for historical tracking. Three actions run in parallel after the AI response is parsed.

**Nodes Involved:**
- Post to Slack Channel
- Email Report to Stakeholders
- Log Metrics to Google Sheets

### Node Details

#### Post to Slack Channel
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends a formatted Slack message to a sales channel.
- **Configuration choices:**  
  - Sends a text report to channel `#sales`
  - Includes date range, revenue, order count, AOV, insights, trends, and recommendations
- **Key expressions or variables used:**  
  - `$now.minus({days: 7}).toFormat('MMM dd')`
  - `$now.toFormat('MMM dd')`
  - `$json.weeklyMetrics.totalRevenue`
  - `$json.weeklyMetrics.revenueChange`
  - `$json.weeklyMetrics.orderCount`
  - `$json.weeklyMetrics.avgOrderValue`
  - `$json.aiInsights.insights`
  - `$json.aiInsights.trends`
  - `$json.aiInsights.recommendations`
- **Input and output connections:**  
  - Input: `Parse AI Response`
  - Output: none
- **Version-specific requirements:**  
  Uses Slack node version `2.2`. Requires Slack credentials authorized to post into the specified channel.
- **Edge cases or potential failure types:**  
  - Channel name may not resolve if the bot is not a member
  - Slack markdown rendering may not match the author’s expectations
  - Long AI responses may exceed comfortable message size
  - Null or missing AI fields will produce incomplete messages
- **Sub-workflow reference:**  
  None.

#### Email Report to Stakeholders
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the weekly report as an HTML email.
- **Configuration choices:**  
  - Subject includes current date
  - Body contains formatted metrics, AI commentary, and top products list
  - No explicit recipient field appears in the provided configuration, so this node is incomplete as shown
- **Key expressions or variables used:**  
  - `$now.minus({days: 7}).toFormat('MMM dd')`
  - `$now.toFormat('MMM dd')`
  - `$json.weeklyMetrics.totalRevenue`
  - `$json.weeklyMetrics.revenueChange`
  - `$json.weeklyMetrics.orderCount`
  - `$json.weeklyMetrics.avgOrderValue`
  - `$json.aiInsights.insights`
  - `$json.aiInsights.trends`
  - `$json.aiInsights.recommendations`
  - `$json.weeklyMetrics.topProducts.map(...)`
- **Input and output connections:**  
  - Input: `Parse AI Response`
  - Output: none
- **Version-specific requirements:**  
  Uses Gmail node version `2.1`. Requires Gmail OAuth2 credentials and a configured recipient.
- **Edge cases or potential failure types:**  
  - Missing recipient configuration will prevent sending
  - Gmail credential expiration or re-consent issues
  - HTML rendering varies by email client
  - Null metrics or arrays can break inline expressions if not handled
- **Sub-workflow reference:**  
  None.

#### Log Metrics to Google Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends weekly metrics to a spreadsheet for historical recordkeeping.
- **Configuration choices:**  
  - Operation: `appendRow`
  - Document ID is a placeholder: `your-spreadsheet-id`
  - No visible column mapping is configured in the JSON
- **Key expressions or variables used:**  
  None shown in parameters.
- **Input and output connections:**  
  - Input: `Parse AI Response`
  - Output: `Check Revenue Drop >20%`
- **Version-specific requirements:**  
  Uses Google Sheets node version `4.5`. Requires Google credentials with access to the target spreadsheet.
- **Edge cases or potential failure types:**  
  - Placeholder spreadsheet ID must be replaced
  - Missing sheet/tab selection or column mapping may prevent append operations
  - Data shape may not match sheet columns
  - Permission issues on the spreadsheet
- **Sub-workflow reference:**  
  None.

---

## 2.6 Revenue Drop Alerting

**Overview:**  
This block checks whether weekly revenue fell below a threshold and sends an alert if necessary. In the current implementation, the condition depends on a simulated comparison value from the Code node.

**Nodes Involved:**
- Check Revenue Drop >20%
- Send Revenue Drop Alert

### Node Details

#### Check Revenue Drop >20%
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates whether the revenue change percentage is less than `-20`.
- **Configuration choices:**  
  - Numeric comparison
  - Left value: parsed revenue change from current item
  - Condition: less than `-20`
- **Key expressions or variables used:**  
  - `={{ parseFloat($json.weeklyMetrics.revenueChange) }}`
- **Input and output connections:**  
  - Input: `Log Metrics to Google Sheets`
  - True output: `Send Revenue Drop Alert`
  - False output: none connected
- **Version-specific requirements:**  
  Uses IF node version `2`.
- **Edge cases or potential failure types:**  
  - If `revenueChange` is non-numeric or missing, parsing may result in `NaN`
  - Because `revenueChange` comes from simulated previous-week revenue, this condition is not based on real historical data
  - False branch is intentionally unused
- **Sub-workflow reference:**  
  None.

#### Send Revenue Drop Alert
- **Type and technical role:** `n8n-nodes-base.slack`  
  Sends an escalation message to an alerts channel when a significant revenue decline is detected.
- **Configuration choices:**  
  - Sends to `#alerts`
  - Includes current revenue, revenue change, recommended actions, and AI risk assessment
  - Uses fallback text if `aiInsights.risks` is missing
- **Key expressions or variables used:**  
  - `$json.weeklyMetrics.totalRevenue`
  - `$json.weeklyMetrics.revenueChange`
  - `$json.aiInsights.risks || 'Review recommended for declining performance'`
- **Input and output connections:**  
  - Input: `Check Revenue Drop >20%`
  - Output: none
- **Version-specific requirements:**  
  Uses Slack node version `2.2`.
- **Edge cases or potential failure types:**  
  - Channel access issues
  - Alert likely never triggers with current synthetic comparison logic
  - Missing AI risk field degrades gracefully due to fallback string
- **Sub-workflow reference:**  
  None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Weekly Monday Trigger | Schedule Trigger | Starts the workflow every Monday at 09:00 |  | Get Last 7 Days Orders | ## Data collection<br>Fetch orders from Shopify |
| Get Last 7 Days Orders | Shopify | Retrieves Shopify orders for analysis | Weekly Monday Trigger | Aggregate Sales Data | ## Data collection<br>Fetch orders from Shopify |
| Aggregate Sales Data | Code | Computes weekly metrics from order data | Get Last 7 Days Orders | Gemini Sales Analysis | ## Data processing<br>Aggregate sales metrics and prepare for AI analysis |
| Gemini Sales Analysis | HTTP Request | Sends summarized metrics to Gemini for AI interpretation | Aggregate Sales Data | Parse AI Response | ## AI analysis<br>Gemini AI analyzes trends and provides insights |
| Parse AI Response | Set | Parses AI output and combines it with metrics | Gemini Sales Analysis | Post to Slack Channel; Email Report to Stakeholders; Log Metrics to Google Sheets | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |
| Post to Slack Channel | Slack | Posts the weekly report to Slack | Parse AI Response |  | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |
| Email Report to Stakeholders | Gmail | Sends the weekly sales report by email | Parse AI Response |  | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |
| Log Metrics to Google Sheets | Google Sheets | Appends weekly metrics to a spreadsheet | Parse AI Response | Check Revenue Drop >20% | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |
| Check Revenue Drop >20% | IF | Checks whether revenue declined by more than 20% | Log Metrics to Google Sheets | Send Revenue Drop Alert | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |
| Send Revenue Drop Alert | Slack | Sends a Slack alert for major revenue decline | Check Revenue Drop >20% |  | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |
| Overview | Sticky Note | Documentation note describing workflow purpose and setup |  |  | ## Weekly Shopify sales analysis with Gemini AI<br><br>This workflow analyzes your Shopify sales data weekly and provides AI-powered insights via Slack and email.<br><br>### How it works<br>1. **Data collection**: Pulls last 7 days of Shopify orders every Monday morning<br>2. **Analysis**: Aggregates sales metrics (revenue, top products, order count)<br>3. **AI insights**: Sends data to Gemini AI for trend analysis and recommendations<br>4. **Reporting**: Delivers insights via Slack channel and email to stakeholders<br>5. **Tracking**: Logs metrics to Google Sheets for historical analysis<br>6. **Alerts**: Notifies team if revenue drops >20% from previous week<br><br>### Setup steps<br>1. Configure Shopify credentials with order read permissions<br>2. Set up Gemini API key for AI analysis<br>3. Connect Slack workspace and choose target channel<br>4. Add Gmail credentials for email reports<br>5. Create Google Sheets for data tracking<br>6. Adjust revenue threshold in the filter node<br><br>### Customization<br>- Change schedule frequency in the trigger<br>- Modify revenue drop threshold (default: 20%)<br>- Customize Gemini prompts for different analysis types<br>- Add more Slack channels for department-specific reports |
| Data Collection | Sticky Note | Visual documentation for Shopify retrieval block |  |  | ## Data collection<br>Fetch orders from Shopify |
| Processing | Sticky Note | Visual documentation for aggregation block |  |  | ## Data processing<br>Aggregate sales metrics and prepare for AI analysis |
| AI Analysis | Sticky Note | Visual documentation for Gemini block |  |  | ## AI analysis<br>Gemini AI analyzes trends and provides insights |
| Reporting | Sticky Note | Visual documentation for reporting and alert block |  |  | ## Reporting & alerts<br>Distribute insights via Slack, email, and track in Google Sheets. Alert on revenue drops. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Analyze Shopify orders with Gemini AI and send weekly Slack insights**.

2. **Add a Schedule Trigger node** named **Weekly Monday Trigger**.
   - Set trigger type to cron/schedule.
   - Use cron expression: `0 9 * * MON`
   - Confirm your n8n instance timezone matches the business reporting timezone.

3. **Add a Shopify node** named **Get Last 7 Days Orders**.
   - Connect it after the trigger.
   - Set operation to **Get Many / Get All Orders**.
   - Set limit to **500**.
   - Configure Shopify credentials with order-read permission.
   - Important: if you want the workflow to truly fetch only the last 7 days, add date filtering based on `created_at` or equivalent Shopify query support. The provided workflow JSON does not currently do this.

4. **Add a Code node** named **Aggregate Sales Data**.
   - Connect it after the Shopify node.
   - Paste JavaScript that:
     - reads all orders with `$input.all()`
     - calculates total revenue
     - counts orders
     - aggregates daily sales by order date
     - sums product quantities from `line_items`
     - extracts top 5 products
     - calculates average order value
     - calculates a previous week value and percentage revenue change
   - Use the same output structure:
     - `totalRevenue`
     - `orderCount`
     - `avgOrderValue`
     - `topProducts`
     - `dailySales`
     - `previousWeekRevenue`
     - `revenueChange`
   - Recommended improvement: instead of simulated previous-week data, later read real previous-week metrics from Google Sheets or another datastore.

5. **Add an HTTP Request node** named **Gemini Sales Analysis**.
   - Connect it after **Aggregate Sales Data**.
   - Configure:
     - Method: `POST`
     - URL:  
       `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash-latest:generateContent`
   - Enable sending headers and body.
   - Add header:
     - `Content-Type: application/json`
   - Configure authentication using **HTTP Header Auth** or another compatible Gemini auth method.
   - Set the API key header according to your Gemini credential approach.
   - Build a JSON request body containing `contents` with one text prompt embedding:
     - total revenue
     - order count
     - average order value
     - revenue change
     - top products
     - daily sales
   - In the prompt, explicitly instruct Gemini to return valid JSON with keys:
     - `insights`
     - `trends`
     - `recommendations`
     - `risks`

6. **Add a Set node** named **Parse AI Response**.
   - Connect it after the HTTP Request node.
   - Create field `aiInsights` as an object using:
     - parsed text from the Gemini response
   - Create field `weeklyMetrics` as an object using:
     - the JSON output from **Aggregate Sales Data**
   - Use expressions equivalent to:
     - `JSON.parse(responseText)`
     - reference to the `Aggregate Sales Data` node output
   - Recommended improvement: add error handling before parsing if the model may return fenced JSON or plain prose.

7. **Add a Slack node** named **Post to Slack Channel**.
   - Connect it from **Parse AI Response**.
   - Configure Slack credentials.
   - Set operation to send a message to channel **#sales** or your preferred channel.
   - Build the message body with:
     - reporting period
     - total revenue
     - revenue change
     - order count
     - average order value
     - AI insights
     - trends
     - recommendations

8. **Add a Gmail node** named **Email Report to Stakeholders**.
   - Also connect it from **Parse AI Response**.
   - Configure Gmail OAuth2 credentials.
   - Set:
     - recipient email address(es)
     - subject similar to `Weekly Sales Report - {{ current date }}`
     - HTML message body containing metrics, AI insights, trends, recommendations, and top products
   - Important: the provided workflow JSON does not show a recipient field, so you must define one manually.

9. **Add a Google Sheets node** named **Log Metrics to Google Sheets**.
   - Also connect it from **Parse AI Response**.
   - Configure Google Sheets credentials.
   - Set operation to **Append Row**.
   - Replace placeholder spreadsheet ID `your-spreadsheet-id` with your real spreadsheet ID.
   - Select the target sheet/tab.
   - Map columns for values such as:
     - report date
     - total revenue
     - order count
     - average order value
     - revenue change
     - top products summary
   - If you want future comparisons, also store enough fields to reconstruct previous-week benchmarks.

10. **Add an IF node** named **Check Revenue Drop >20%**.
    - Connect it after **Log Metrics to Google Sheets**.
    - Configure a numeric condition:
      - left value: parsed `weeklyMetrics.revenueChange`
      - operator: less than
      - right value: `-20`
    - This means the true path activates only when revenue fell by more than 20%.

11. **Add another Slack node** named **Send Revenue Drop Alert**.
    - Connect it to the **true** output of the IF node.
    - Configure Slack credentials.
    - Set destination channel to **#alerts** or your incident/ops channel.
    - Build the alert message with:
      - current revenue
      - percentage drop
      - response suggestions
      - AI risk assessment if available

12. **Optionally add sticky notes** for maintainability.
    - Add notes for:
      - overview
      - data collection
      - processing
      - AI analysis
      - reporting and alerts

13. **Configure credentials** before activation:
    - **Shopify**: read access to orders
    - **Gemini / Google AI**: API key or header-based auth
    - **Slack**: bot/user able to post in selected channels
    - **Gmail**: OAuth2 with send-mail permission
    - **Google Sheets**: access to the spreadsheet

14. **Test each stage manually**.
    - Run the workflow with sample orders
    - Verify Shopify returns data
    - Inspect the Code node output
    - Confirm Gemini returns parsable JSON
    - Check Slack formatting
    - Confirm Gmail recipient and email rendering
    - Verify row append behavior in Google Sheets
    - Test the IF branch with mocked negative revenue values

15. **Correct the two major implementation gaps** if you want production-ready behavior:
    - Add an actual Shopify date filter for the last 7 days
    - Replace simulated `previousWeekRevenue` with real historical data from Google Sheets or another data source

### Sub-workflow setup
This workflow does **not** include any Execute Workflow or sub-workflow nodes. No sub-workflow configuration is required.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Weekly Shopify sales analysis with Gemini AI. This workflow analyzes your Shopify sales data weekly and provides AI-powered insights via Slack and email. | Workflow overview |
| Data collection: Pulls last 7 days of Shopify orders every Monday morning. | Overview note |
| Analysis: Aggregates sales metrics (revenue, top products, order count). | Overview note |
| AI insights: Sends data to Gemini AI for trend analysis and recommendations. | Overview note |
| Reporting: Delivers insights via Slack channel and email to stakeholders. | Overview note |
| Tracking: Logs metrics to Google Sheets for historical analysis. | Overview note |
| Alerts: Notifies team if revenue drops >20% from previous week. | Overview note |
| Setup steps: Configure Shopify credentials with order read permissions; set up Gemini API key; connect Slack workspace; add Gmail credentials; create Google Sheets; adjust revenue threshold in the filter node. | Overview note |
| Customization: change schedule frequency; modify revenue drop threshold; customize Gemini prompts; add more Slack channels. | Overview note |

### Additional implementation notes
- The workflow is logically well structured, but **the “last 7 days” claim is not enforced in the Shopify node configuration**.
- The revenue comparison is currently **synthetic**, not historical.
- The Gmail node appears **incomplete** because no recipient is visible in the provided configuration.
- The Google Sheets node contains a **placeholder spreadsheet ID** and will require explicit column mapping in a real deployment.