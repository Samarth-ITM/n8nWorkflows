Generate weekly Brevo campaign analytics reports with Groq and Google Sheets

https://n8nworkflows.xyz/workflows/generate-weekly-brevo-campaign-analytics-reports-with-groq-and-google-sheets-14671


# Generate weekly Brevo campaign analytics reports with Groq and Google Sheets

# 1. Workflow Overview

This workflow automates a weekly email performance reporting cycle for Brevo campaigns. It retrieves recently sent email campaigns from the Brevo API, computes campaign-level metrics, stores those metrics in Google Sheets for historical tracking, filters the last 7 days of records, uses a Groq-hosted LLM to generate a concise executive summary, and sends the result as an HTML email via Gmail.

It is designed for marketing reporting use cases such as:
- Weekly campaign performance summaries
- Executive reporting for marketing leads
- Long-term campaign metric archiving in Google Sheets
- AI-assisted interpretation of engagement and bounce trends

## 1.1 Scheduled Trigger and Brevo Data Retrieval
The workflow starts on a weekly schedule and fetches the latest sent Brevo email campaigns through the Brevo API.

## 1.2 Campaign Data Normalization and Metric Calculation
The raw Brevo response is flattened into simple campaign rows, and derived metrics such as open rate, click rate, and bounce rate are calculated.

## 1.3 Historical Logging in Google Sheets
Each campaign row is appended to a Google Sheet to build a persistent reporting dataset over time.

## 1.4 Historical Readback and 7-Day Filtering
After writing the latest rows, the workflow reads the full sheet back, then filters it to only retain entries from the past 7 days.

## 1.5 Weekly Aggregation and AI Analysis
The 7-day rows are aggregated into arrays and passed to an AI Agent that generates a 3-sentence executive summary using a Groq chat model.

## 1.6 HTML Report Composition and Email Delivery
The AI summary and campaign table are injected into an HTML email template and sent to a recipient via Gmail.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Brevo Data Retrieval

### Overview
This block initiates the workflow once per week and pulls recent sent campaigns from Brevo. It is the workflow’s only execution entry point.

### Nodes Involved
- Schedule Trigger
- HTTP Request

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger node that starts the workflow automatically.
- **Configuration choices:**
  - Configured to run every week
  - Trigger day: Monday
  - Trigger hour: 08:00
- **Key expressions or variables used:** None
- **Input and output connections:**
  - No input node
  - Outputs to `HTTP Request`
- **Version-specific requirements:**
  - Uses node type version `1.3`
- **Edge cases or potential failure types:**
  - Timezone differences between n8n instance and business reporting expectation
  - Workflow will not run unless activated
- **Sub-workflow reference:** None

#### HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Brevo REST API to retrieve email campaign data.
- **Configuration choices:**
  - Method is implicitly GET
  - URL: `https://api.brevo.com/v3/emailCampaigns`
  - Authentication: predefined credential type
  - Credential type: `sendInBlueApi`
  - Query parameters:
    - `status=sent`
    - `limit=10`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Schedule Trigger`
  - Output to `Data Cleaning1`
- **Version-specific requirements:**
  - Uses node type version `4.3`
  - Requires a configured Brevo / Sendinblue API credential in n8n
- **Edge cases or potential failure types:**
  - Invalid API key or missing credential
  - Brevo API rate limit or temporary outage
  - Response schema changes, especially if `campaigns` or nested statistics fields move
  - The `limit=10` parameter may omit older campaigns if more than 10 were sent recently
- **Sub-workflow reference:** None

---

## 2.2 Campaign Data Normalization and Metric Calculation

### Overview
This block transforms the nested Brevo response into one row per campaign and computes reporting metrics. It makes the data compatible with Google Sheets storage and later AI analysis.

### Nodes Involved
- Data Cleaning1

### Node Details

#### Data Cleaning1
- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript transformation node that flattens campaign data and computes derived KPIs.
- **Configuration choices:**
  - Reads `items[0].json.campaigns`
  - Iterates through each campaign
  - Pulls statistics from `campaign.statistics.campaignStats[0]`
  - Extracts:
    - Campaign Name
    - Date
    - Sent
    - Delivered
    - Open Rate
    - Click Rate
    - Bounce Rate
  - Converts `sentDate` to `YYYY-MM-DD` using `split('T')[0]`
  - Calculates:
    - `openRate = uniqueViews / delivered`
    - `clickRate = uniqueClicks / delivered`
    - `bounceRate = (softBounces + hardBounces) / delivered`
  - Falls back to `0` if values are missing
- **Key expressions or variables used:**
  - `items[0].json.campaigns || []`
  - `campaign.statistics.campaignStats[0] || {}`
  - `campaign.sentDate.split('T')[0]`
- **Input and output connections:**
  - Input from `HTTP Request`
  - Output to `Append row in sheet`
- **Version-specific requirements:**
  - Uses node type version `2`
- **Edge cases or potential failure types:**
  - If `campaign.sentDate` is null or missing, `split('T')` will fail
  - If `campaign.statistics` or `campaignStats` is missing, metrics default to zero, which may hide data quality issues
  - Bounce rate is calculated against `delivered`, not `sent`; depending on business logic, this may or may not be desired
  - Multiple `campaignStats` entries are ignored except for index `0`
- **Sub-workflow reference:** None

---

## 2.3 Historical Logging in Google Sheets

### Overview
This block appends each cleaned campaign record into a Google Sheet. The sheet serves as the workflow’s historical analytics store.

### Nodes Involved
- Append row in sheet

### Node Details

#### Append row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends campaign metrics into a spreadsheet row-by-row.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet document: `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `gid=0` / `Sheet1`
  - Mapping mode: define below
  - Explicit column mapping:
    - Date
    - Sent
    - Delivered
    - Open Rate
    - Click Rate
    - Bounce Rate
    - Campaign Name
- **Key expressions or variables used:**
  - `{{ $json.Date }}`
  - `{{ $json.Sent }}`
  - `{{ $json.Delivered }}`
  - `{{ $json["Open Rate"] }}`
  - `{{ $json["Click Rate"] }}`
  - `{{ $json["Bounce Rate"] }}`
  - `{{ $json["Campaign Name"] }}`
- **Input and output connections:**
  - Input from `Data Cleaning1`
  - Output to `Aggregate`
- **Version-specific requirements:**
  - Uses node type version `4.7`
  - Requires Google Sheets credentials with edit access to the target file
  - The target sheet must already contain matching headers
- **Edge cases or potential failure types:**
  - Missing or mismatched headers in the sheet will cause mapping problems
  - Invalid spreadsheet ID or insufficient permissions
  - Re-runs can create duplicate rows because there is no deduplication logic
  - Type conversion is disabled, so non-string/numeric inconsistencies may persist
- **Sub-workflow reference:** None

---

## 2.4 Historical Readback and 7-Day Filtering

### Overview
This block reads the historical sheet data and restricts the reporting scope to the last 7 days. It separates archival storage from weekly analysis.

### Nodes Involved
- Aggregate
- Get row(s) in sheet
- Filtered 7 days data

### Node Details

#### Aggregate
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Consolidates all incoming items after the append step.
- **Configuration choices:**
  - Aggregate mode: `aggregateAllItemData`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Append row in sheet`
  - Output to `Get row(s) in sheet`
- **Version-specific requirements:**
  - Uses node type version `1`
- **Edge cases or potential failure types:**
  - This node does not appear to contribute meaningful business transformation for the subsequent sheet read; it mainly collapses prior items
  - If append returns no items, downstream behavior may be affected
- **Sub-workflow reference:** None

#### Get row(s) in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads historical data from the same spreadsheet used for storage.
- **Configuration choices:**
  - Reads from document: `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `gid=0` / `Sheet1`
  - Default row retrieval behavior, with no explicit filters configured
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Aggregate`
  - Output to `Filtered 7 days data`
- **Version-specific requirements:**
  - Uses node type version `4.7`
  - Requires Google Sheets read access
- **Edge cases or potential failure types:**
  - Large sheets may slow execution
  - Empty sheet produces no rows
  - Date values must be parseable by JavaScript in the next node
- **Sub-workflow reference:** None

#### Filtered 7 days data
- **Type and technical role:** `n8n-nodes-base.code`  
  Filters sheet rows to those with dates within the last 7 days and enriches each row with a lightweight AI context string.
- **Configuration choices:**
  - Computes `sevenDaysAgo`
  - Sets comparison time to midnight
  - Iterates through all sheet rows
  - Includes rows where `rowDate >= sevenDaysAgo`
  - Adds:
    - all original row fields
    - `aiContext`
- **Key expressions or variables used:**
  - `new Date(item.json.Date)`
  - `sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7)`
  - `aiContext: \`${item.json['Campaign Name']} sent on ${item.json.Date} had a ${item.json['Open Rate']} open rate.\``
- **Input and output connections:**
  - Input from `Get row(s) in sheet`
  - Output to `Aggregate weekly data`
- **Version-specific requirements:**
  - Uses node type version `2`
- **Edge cases or potential failure types:**
  - If the sheet date format is locale-specific or inconsistent, `new Date()` may parse incorrectly
  - The comparison uses the system date/time of the n8n runtime
  - `aiContext` is generated but not used later in the workflow
- **Sub-workflow reference:** None

---

## 2.5 Weekly Aggregation and AI Analysis

### Overview
This block compiles the filtered weekly rows into arrays and prompts a large language model to produce a compact executive summary. It is the main interpretation layer of the workflow.

### Nodes Involved
- Aggregate weekly data
- AI Agent
- Groq Chat Model

### Node Details

#### Aggregate weekly data
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Collects selected fields from multiple weekly rows into arrays on a single item.
- **Configuration choices:**
  - Aggregates fields:
    - Campaign Name
    - Sent
    - Delivered
    - Open Rate
    - Click Rate
    - Bounce Rate
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Filtered 7 days data`
  - Output to `AI Agent`
- **Version-specific requirements:**
  - Uses node type version `1`
- **Edge cases or potential failure types:**
  - If no rows match the last 7 days, aggregated arrays may be empty and the AI prompt will contain an empty table
  - Inconsistent data types from the sheet can produce formatting errors later
- **Sub-workflow reference:** None

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based agent node that generates the executive summary from aggregated campaign data.
- **Configuration choices:**
  - Prompt type: `define`
  - Custom text prompt instructs the model to:
    1. Identify the best campaign by engagement depth using click-to-open logic
    2. Alert if any bounce rate is greater than or equal to 2%
    3. Suggest a strategy for next week
  - Builds a Markdown table from aggregated arrays
- **Key expressions or variables used:**
  - `{{ $json["Campaign Name"].map((name, i) => ... ).join('\n') }}`
  - `{{ $json["Sent"][i] }}`
  - `{{ $json["Delivered"][i] }}`
  - `{{ ($json["Open Rate"][i] * 100).toFixed(1) }}%`
  - `{{ ($json["Click Rate"][i] * 100).toFixed(1) }}%`
  - `{{ ($json["Bounce Rate"][i] * 100).toFixed(1) }}%`
- **Input and output connections:**
  - Main input from `Aggregate weekly data`
  - AI language model input from `Groq Chat Model`
  - Main output to `Edit Fields`
- **Version-specific requirements:**
  - Uses node type version `3`
  - Requires compatible LangChain support in the n8n instance
- **Edge cases or potential failure types:**
  - If any aggregated field is missing or not an array, `.map()` will fail
  - Empty data may still produce text, but the summary may be low quality or misleading
  - Prompt refers to engagement depth conceptually, but click-to-open ratio is not precomputed; the model must infer it from click and open rates
  - The sticky note mentions Gemini, but the actual model used is Groq
- **Sub-workflow reference:** None

#### Groq Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Provides the underlying LLM used by the AI Agent.
- **Configuration choices:**
  - Model: `llama-3.3-70b-versatile`
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Outputs as `ai_languageModel` to `AI Agent`
- **Version-specific requirements:**
  - Uses node type version `1`
  - Requires Groq API credentials configured in n8n
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model availability or rate limiting
  - Output variability typical of LLM-based summaries
- **Sub-workflow reference:** None

---

## 2.6 HTML Report Composition and Email Delivery

### Overview
This block constructs a styled HTML email containing the AI summary and a campaign breakdown table, then delivers it by Gmail. It is the final presentation and distribution layer.

### Nodes Involved
- Edit Fields
- Send a message

### Node Details

#### Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a single HTML body field used by the email node.
- **Configuration choices:**
  - Assigns a string field named `emailBoady`
  - Generates a complete HTML email template
  - Includes:
    - reporting period derived from `$today`
    - AI summary from `{{ $json.output }}`
    - campaign metric table using data from `Aggregate weekly data`
    - conditional red text for bounce rates `>= 0.02`
- **Key expressions or variables used:**
  - `{{ $today.minus({days: 7}).toFormat('dd LLL') }}`
  - `{{ $today.toFormat('dd LLL yyyy') }}`
  - `{{ $json.output }}`
  - `{{ $node["Aggregate weekly data"].json["Campaign Name"].map(...) }}`
  - `{{ $node["Aggregate weekly data"].json["Delivered"][i] }}`
  - `{{ ($node["Aggregate weekly data"].json["Open Rate"][i] * 100).toFixed(1) }}%`
  - `{{ ($node["Aggregate weekly data"].json["Click Rate"][i] * 100).toFixed(1) }}%`
  - conditional color based on `Bounce Rate >= 0.02`
- **Input and output connections:**
  - Input from `AI Agent`
  - Output to `Send a message`
- **Version-specific requirements:**
  - Uses node type version `3.4`
- **Edge cases or potential failure types:**
  - Field name is misspelled as `emailBoady`; this is not fatal because the Gmail node uses the same misspelling, but it is easy to break during maintenance
  - If `Aggregate weekly data` has no arrays, the HTML table may render empty
  - If AI output contains unexpected characters, email formatting may degrade
  - Relies on cross-node reference to `Aggregate weekly data`, which couples this node tightly to upstream naming
- **Sub-workflow reference:** None

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the final HTML report email.
- **Configuration choices:**
  - Recipient: `user@example.com`
  - Subject: `🚀 Weekly Marketing Performance Brief: {{ $today.toFormat('dd/MM/yyyy') }}`
  - Message body: `{{ $json.emailBoady }}`
- **Key expressions or variables used:**
  - `{{ $json.emailBoady }}`
  - `{{ $today.toFormat('dd/MM/yyyy') }}`
- **Input and output connections:**
  - Input from `Edit Fields`
  - No downstream nodes
- **Version-specific requirements:**
  - Uses node type version `2.2`
  - Requires Gmail credentials with send permission
- **Edge cases or potential failure types:**
  - Gmail authentication or OAuth scope issues
  - HTML body may be sent as plain content depending on node behavior and account configuration
  - Invalid recipient email address
  - Sending limits on Gmail accounts
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Weekly workflow entry point |  | HTTP Request | ## STEP 1: Data Acquisition & Cleaning<br>Nodes: Schedule → HTTP Request → Data Cleaning (Code)<br>Execution: Triggers the workflow to fetch "Sent" campaigns from the Brevo API. The code immediately flattens the nested JSON and calculates the Open, Click, and Bounce rates for the current run. |
| HTTP Request | n8n-nodes-base.httpRequest | Fetch sent campaigns from Brevo API | Schedule Trigger | Data Cleaning1 | ## STEP 1: Data Acquisition & Cleaning<br>Nodes: Schedule → HTTP Request → Data Cleaning (Code)<br>Execution: Triggers the workflow to fetch "Sent" campaigns from the Brevo API. The code immediately flattens the nested JSON and calculates the Open, Click, and Bounce rates for the current run. |
| Data Cleaning1 | n8n-nodes-base.code | Flatten API response and calculate metrics | HTTP Request | Append row in sheet | ## STEP 1: Data Acquisition & Cleaning<br>Nodes: Schedule → HTTP Request → Data Cleaning (Code)<br>Execution: Triggers the workflow to fetch "Sent" campaigns from the Brevo API. The code immediately flattens the nested JSON and calculates the Open, Click, and Bounce rates for the current run. |
| Append row in sheet | n8n-nodes-base.googleSheets | Archive cleaned campaign rows into Google Sheets | Data Cleaning1 | Aggregate | ## STEP 2: Database Logging<br>Node: Append Row (Sheets)<br>Execution: Maps the cleaned data fields to a Google Sheet. It appends the current campaign statistics as a new row to create a permanent, historical archive for long-term tracking. |
| Aggregate | n8n-nodes-base.aggregate | Collapse append output before sheet readback | Append row in sheet | Get row(s) in sheet | ## STEP 3: Historical Retrieval & Filtering<br>Nodes: Aggregate → Get Rows (Sheets) → Filtered 7 Days (Code)<br>Execution: Pulls the entire database from Google Sheets. The code then strips away any data older than one week, leaving only the relevant "7-Day Window" for the current report. |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Read historical campaign archive from Google Sheets | Aggregate | Filtered 7 days data | ## STEP 3: Historical Retrieval & Filtering<br>Nodes: Aggregate → Get Rows (Sheets) → Filtered 7 Days (Code)<br>Execution: Pulls the entire database from Google Sheets. The code then strips away any data older than one week, leaving only the relevant "7-Day Window" for the current report. |
| Filtered 7 days data | n8n-nodes-base.code | Keep only rows from the last 7 days | Get row(s) in sheet | Aggregate weekly data | ## STEP 3: Historical Retrieval & Filtering<br>Nodes: Aggregate → Get Rows (Sheets) → Filtered 7 Days (Code)<br>Execution: Pulls the entire database from Google Sheets. The code then strips away any data older than one week, leaving only the relevant "7-Day Window" for the current report. |
| Aggregate weekly data | n8n-nodes-base.aggregate | Group weekly metrics into arrays for AI and email rendering | Filtered 7 days data | AI Agent | ## STEP 4: Weekly Intelligence Analysis<br>Nodes: Aggregate Weekly Data → AI Agent (Gemini) → Edit Fields<br>Execution: Bundles the 7-day data into a single object. Gemini analyzes the trends to write an executive summary, and Edit Fields cleans the text for the final email layout. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate executive summary from weekly data | Aggregate weekly data, Groq Chat Model | Edit Fields | ## STEP 4: Weekly Intelligence Analysis<br>Nodes: Aggregate Weekly Data → AI Agent (Gemini) → Edit Fields<br>Execution: Bundles the 7-day data into a single object. Gemini analyzes the trends to write an executive summary, and Edit Fields cleans the text for the final email layout. |
| Groq Chat Model | @n8n/n8n-nodes-langchain.lmChatGroq | LLM backend for AI Agent |  | AI Agent | ## STEP 4: Weekly Intelligence Analysis<br>Nodes: Aggregate Weekly Data → AI Agent (Gemini) → Edit Fields<br>Execution: Bundles the 7-day data into a single object. Gemini analyzes the trends to write an executive summary, and Edit Fields cleans the text for the final email layout. |
| Edit Fields | n8n-nodes-base.set | Build final HTML email body | AI Agent | Send a message | ## STEP 4: Weekly Intelligence Analysis<br>Nodes: Aggregate Weekly Data → AI Agent (Gemini) → Edit Fields<br>Execution: Bundles the 7-day data into a single object. Gemini analyzes the trends to write an executive summary, and Edit Fields cleans the text for the final email layout. |
| Send a message | n8n-nodes-base.gmail | Deliver the final report by email | Edit Fields |  | ## STEP 5: Executive Email Delivery<br>Node: Gmail<br>Execution: Injects the AI-generated insights and the campaign performance table into a responsive HTML template. It sends the finalized intelligence report to the designated stakeholders. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note on overall workflow |  |  | ## Brevo Email Campaign Stats → Weekly Google Sheets Report<br><br>### HOW IT WORKS<br>This workflow serves as an automated "Bridge" between marketing execution and executive reporting. It triggers weekly to fetch sent campaign data from the Brevo API. A custom JavaScript layer cleans the raw JSON and calculates high-level metrics (Open, Click, and Bounce rates). This data is archived in Google Sheets to build a long-term performance history. Finally, Google Gemini AI analyzes the latest 7-day trends and generates a 3-sentence executive summary, which is delivered via a professionally formatted HTML Gmail report.<br><br>### HOW TO SET UP<br><br>**Credentials**: Ensure the HTTP Request node has a valid api-key header from Brevo.<br>**Sheet Mapping**: Your Google Sheet must have headers matching the "Data Cleaning" node keys (e.g., Campaign Name, Open Rate).<br>**AI Logic**: The Gemini node must be in "Expression" mode to map the aggregated arrays into a Markdown table.<br>**Trigger**: Set the "Schedule Trigger" to your preferred reporting day (e.g., Mondays at 08:00).<br><br>### CUSTOMIZATION<br><br>**Thresholds**: Change the 0.02 (2%) value in the HTML code to adjust the "Red Alert" for Bounce Rates.<br>**Channels**: Add a Slack or Discord node at the end for instant team notifications.<br>**Depth**: Adjust the "Filter" node to analyze the last 30 days instead of 7 for monthly reviews. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for Step 1 |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for Step 2 |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for Step 3 |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note for Step 4 |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation note for Step 5 |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Brevo Email Campaign Stats → Weekly Google Sheets Report`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure a weekly schedule
   - Set:
     - Day: Monday
     - Hour: 08:00
   - This becomes the workflow entry point.

3. **Add an HTTP Request node**
   - Node type: `HTTP Request`
   - Connect `Schedule Trigger` → `HTTP Request`
   - Configure:
     - Method: `GET`
     - URL: `https://api.brevo.com/v3/emailCampaigns`
     - Authentication: use Brevo / Sendinblue predefined credential if available
     - Query parameters:
       - `status` = `sent`
       - `limit` = `10`
   - Credential setup:
     - Create or select Brevo API credentials
     - Ensure the request includes a valid Brevo API key

4. **Add a Code node named `Data Cleaning1`**
   - Node type: `Code`
   - Connect `HTTP Request` → `Data Cleaning1`
   - Paste this logic conceptually:
     - Read `items[0].json.campaigns`
     - Loop through campaigns
     - Extract campaign statistics from the first `campaignStats` entry
     - Compute:
       - Sent
       - Delivered
       - Open Rate
       - Click Rate
       - Bounce Rate
     - Return one item per campaign
   - Use these output field names exactly:
     - `Campaign Name`
     - `Date`
     - `Sent`
     - `Delivered`
     - `Open Rate`
     - `Click Rate`
     - `Bounce Rate`
   - Important:
     - The `Date` field is expected in `YYYY-MM-DD` form
     - Guard against null `sentDate` if you harden the flow

5. **Prepare Google Sheets**
   - Create a Google Sheet
   - Add a tab such as `Sheet1`
   - Create header columns exactly matching:
     - `Date`
     - `Campaign Name`
     - `Sent`
     - `Delivered`
     - `Open Rate`
     - `Click Rate`
     - `Bounce Rate`

6. **Add a Google Sheets node named `Append row in sheet`**
   - Node type: `Google Sheets`
   - Connect `Data Cleaning1` → `Append row in sheet`
   - Configure:
     - Operation: `Append`
     - Select your spreadsheet
     - Select the target sheet/tab
     - Use manual field mapping
   - Map fields:
     - Date ← `{{$json.Date}}`
     - Sent ← `{{$json.Sent}}`
     - Delivered ← `{{$json.Delivered}}`
     - Open Rate ← `{{$json["Open Rate"]}}`
     - Click Rate ← `{{$json["Click Rate"]}}`
     - Bounce Rate ← `{{$json["Bounce Rate"]}}`
     - Campaign Name ← `{{$json["Campaign Name"]}}`
   - Credential setup:
     - Configure Google Sheets OAuth2 or service account access
     - Ensure edit permission on the spreadsheet

7. **Add an Aggregate node named `Aggregate`**
   - Node type: `Aggregate`
   - Connect `Append row in sheet` → `Aggregate`
   - Configure aggregate mode as `Aggregate All Item Data`
   - This mirrors the source workflow, even though it is not strictly essential.

8. **Add a Google Sheets node named `Get row(s) in sheet`**
   - Node type: `Google Sheets`
   - Connect `Aggregate` → `Get row(s) in sheet`
   - Configure it to read rows from the same spreadsheet and same tab
   - No additional filter is required in the original design
   - Ensure the node returns all existing rows

9. **Add a Code node named `Filtered 7 days data`**
   - Node type: `Code`
   - Connect `Get row(s) in sheet` → `Filtered 7 days data`
   - Configure logic to:
     - Compute the date 7 days ago at 00:00:00
     - Loop through all rows
     - Parse `item.json.Date`
     - Keep rows where the date is within the last 7 days
     - Add an `aiContext` field for each kept row
   - Expected output:
     - Same row fields as input
     - Additional `aiContext`
   - Note:
     - `aiContext` is not consumed later, but it exists in the workflow and should be reproduced

10. **Add an Aggregate node named `Aggregate weekly data`**
    - Node type: `Aggregate`
    - Connect `Filtered 7 days data` → `Aggregate weekly data`
    - Configure field aggregation so that these fields become arrays:
      - `Campaign Name`
      - `Sent`
      - `Delivered`
      - `Open Rate`
      - `Click Rate`
      - `Bounce Rate`

11. **Add a Groq Chat Model node**
    - Node type: `Groq Chat Model`
    - Configure:
      - Model: `llama-3.3-70b-versatile`
    - Credential setup:
      - Add Groq API credentials in n8n

12. **Add an AI Agent node named `AI Agent`**
    - Node type: `AI Agent`
    - Connect:
      - `Aggregate weekly data` → `AI Agent` as the main input
      - `Groq Chat Model` → `AI Agent` as the language model input
    - Set prompt type to `Define`
    - Add a prompt that:
      - Presents the last 7 days’ campaign data as a Markdown table
      - Asks for a 3-sentence executive summary
      - Requires:
        1. Best campaign by engagement depth / click-to-open logic
        2. Warning if bounce rate is `>= 2%`
        3. A strategy suggestion for next week
    - Use expressions referencing the aggregated arrays, for example:
      - `{{$json["Campaign Name"].map((name, i) => ... ).join('\n')}}`
    - Make sure percentage values are multiplied by 100 and formatted.

13. **Add a Set node named `Edit Fields`**
    - Node type: `Set`
    - Connect `AI Agent` → `Edit Fields`
    - Add a string field named exactly `emailBoady`
      - Preserve the original spelling if you want exact compatibility
    - Build an HTML email template that includes:
      - A title banner
      - Reporting period using `$today.minus({days: 7})` and `$today`
      - AI summary from `{{$json.output}}`
      - A metrics table generated from `{{$node["Aggregate weekly data"].json[...]}}`
      - Conditional formatting for bounce rates:
        - red if `>= 0.02`
    - Important:
      - This node depends on the upstream node being named exactly `Aggregate weekly data`

14. **Add a Gmail node named `Send a message`**
    - Node type: `Gmail`
    - Connect `Edit Fields` → `Send a message`
    - Configure:
      - To: your recipient email, replacing `user@example.com`
      - Subject: `🚀 Weekly Marketing Performance Brief: {{$today.toFormat('dd/MM/yyyy')}}`
      - Message/body: `{{$json.emailBoady}}`
    - Credential setup:
      - Configure Gmail OAuth2 with permission to send email

15. **Verify node connections**
   - Final connection order:
     1. Schedule Trigger
     2. HTTP Request
     3. Data Cleaning1
     4. Append row in sheet
     5. Aggregate
     6. Get row(s) in sheet
     7. Filtered 7 days data
     8. Aggregate weekly data
     9. AI Agent
     10. Edit Fields
     11. Send a message
   - Additional model connection:
     - Groq Chat Model → AI Agent

16. **Add optional sticky notes**
   - Create documentation notes for each functional step if you want to match the original structure:
     - Step 1: acquisition and cleaning
     - Step 2: sheet logging
     - Step 3: retrieval and filtering
     - Step 4: AI analysis
     - Step 5: email delivery

17. **Activate and test**
   - Run the workflow manually first
   - Check:
     - Brevo API returns `campaigns`
     - Google Sheet rows are appended correctly
     - Date parsing works for sheet data
     - AI summary is generated
     - Gmail email renders HTML correctly
   - Then activate the workflow for weekly execution

## Reproduction Notes and Constraints
- Replace `YOUR_GOOGLE_SHEET_ID` with the actual spreadsheet ID.
- Replace `user@example.com` with the actual stakeholder email.
- If you rename nodes, update all cross-node expressions accordingly.
- If you want to avoid duplicate rows, add deduplication logic before appending to the sheet.
- If there are no campaigns in the last 7 days, consider adding an IF node or fallback summary before the AI step.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow description in notes says “Google Gemini AI,” but the actual configured model is a Groq Chat Model using `llama-3.3-70b-versatile`. | Important implementation discrepancy |
| The HTML field is named `emailBoady`, which is likely a typo for `emailBody`. It works only because the Gmail node references the same misspelled field. | Maintenance note |
| Bounce-rate highlighting threshold is `0.02`, representing 2%. | Used in the HTML formatting logic |
| The workflow stores campaign metrics historically in Google Sheets, then reads the full sheet back for weekly analysis. | Architectural pattern |
| The reporting window is implemented in code, not in the Google Sheets query itself. | Filtering behavior |
| The sheet must contain headers matching the cleaned output fields exactly. | Google Sheets setup requirement |