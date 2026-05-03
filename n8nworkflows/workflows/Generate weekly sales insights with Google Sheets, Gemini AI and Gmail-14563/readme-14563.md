Generate weekly sales insights with Google Sheets, Gemini AI and Gmail

https://n8nworkflows.xyz/workflows/generate-weekly-sales-insights-with-google-sheets--gemini-ai-and-gmail-14563


# Generate weekly sales insights with Google Sheets, Gemini AI and Gmail

# 1. Workflow Overview

This workflow generates a weekly sales performance report from Google Sheets, enriches it with an AI-written summary using Gemini, creates a category comparison chart with QuickChart, formats everything into an HTML email, and sends it through Gmail.

Its main use case is automated weekly business reporting for retail or sales teams that track order performance in a spreadsheet. The workflow is designed for recurring execution and produces a polished email containing KPI highlights, AI commentary, category-level comparisons, and visual reporting.

## 1.1 Trigger & Data Retrieval

The workflow starts on a weekly schedule and retrieves raw rows from a Google Sheets document.

## 1.2 Data Cleaning & Date Filtering

The raw sheet data is cleaned to remove empty rows and convert currency and quantity fields into numeric values. A date filter then narrows the dataset to a fixed two-week reporting window.

## 1.3 Sales Analytics Computation

A custom code node compares the current week against the previous week, computes totals and growth, aggregates category sales, builds HTML table rows, and prepares structured context for downstream AI and chart generation.

## 1.4 AI Insight Generation & Chart Request

Gemini generates a short business summary from the computed metrics. A QuickChart request is then made to render a bar chart comparing category performance between last week and this week.

## 1.5 Email Formatting & Delivery

The workflow builds an HTML email using analytics outputs and AI text, then sends the report to a configured Gmail recipient.

---

# 2. Block-by-Block Analysis

## Block 1: Trigger & Fetch

### Overview

This block launches the workflow automatically every week and loads the sales dataset from Google Sheets. It is the entry point for the entire reporting pipeline.

### Nodes Involved

- Schedule Trigger1
- Get row(s) in sheet1

### Node Details

#### 1. Schedule Trigger1

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry node that starts the workflow automatically.
- **Configuration choices:**  
  Configured to run every week on day `1` at hour `21`. In n8n schedule semantics, day `1` typically means Monday, depending on instance locale/settings.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get row(s) in sheet1`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Timezone mismatch between expected business timezone and n8n instance timezone
  - Unexpected run day if schedule interpretation differs
- **Sub-workflow reference:**  
  None.

#### 2. Get row(s) in sheet1

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheets worksheet.
- **Configuration choices:**  
  - Document ID is set to `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`
  - Sheet selected is `gid=0` / `Sheet1`
  - No filtering options are configured, so it fetches available rows from the sheet
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: `Schedule Trigger1`
  - Output: `Removing Empty Rows1`
- **Version-specific requirements:**  
  Type version `4.7`. Requires Google Sheets credentials authorized in n8n.
- **Edge cases or potential failure types:**  
  - Invalid spreadsheet ID
  - Missing worksheet or wrong sheet selection
  - Google authentication/token expiration
  - Header mismatch if the sheet columns differ from expected names
- **Sub-workflow reference:**  
  None.

---

## Block 2: Data Cleaning & Date Filtering

### Overview

This block standardizes the sheet data into a usable form and restricts the report to a specific reporting period. It ensures later analytics use numeric sales values and only process rows inside the hardcoded comparison window.

### Nodes Involved

- Removing Empty Rows1
- Filter Latest Week1

### Node Details

#### 1. Removing Empty Rows1

- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation node that filters invalid rows and normalizes field values.
- **Configuration choices:**  
  The script:
  - Loads all items via `$input.all()`
  - Keeps only rows where `Date` and `Product type` exist and `Date` is not blank
  - Converts `Total sales (₹)` from strings like `₹28,000` into numeric values like `28000`
  - Converts `Net quantity` into a number
  - Preserves original row fields and adds `IsCleaned: true`
- **Key expressions or variables used:**  
  - `$input.all()`
  - Expected columns:
    - `Date`
    - `Product type`
    - `Total sales (₹)`
    - `Net quantity`
- **Input and output connections:**  
  - Input: `Get row(s) in sheet1`
  - Output: `Filter Latest Week1`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Column names must match exactly, including capitalization and special characters
  - If sales values contain unexpected text, `Number(...)` may produce `NaN`
  - Rows with missing product type are fully removed
- **Sub-workflow reference:**  
  None.

#### 2. Filter Latest Week1

- **Type and technical role:** `n8n-nodes-base.code`  
  JavaScript filter node that keeps only rows within a fixed date range.
- **Configuration choices:**  
  The script:
  - Defines `rangeStart = 19-Mar-2026`
  - Defines `rangeEnd = 01-Apr-2026 23:59:59`
  - Parses sheet dates expected in `DD-MMM-YYYY` format such as `26-Mar-2026`
  - Returns only items whose date falls within that inclusive range
- **Key expressions or variables used:**  
  - `item.json["Date"]`
  - Internal helper `parseSheetDate(dateStr)`
- **Input and output connections:**  
  - Input: `Removing Empty Rows1`
  - Output: `Sales Analytics Engine`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - The code uses `return items.filter(...)`; in n8n Code node behavior, this assumes `items` is available in scope. It usually is, but consistency may vary by execution mode/version. Safer code would explicitly reference input items.
  - Any date not matching `DD-MMM-YYYY` may parse incorrectly or become invalid
  - The date range is hardcoded, so future scheduled runs will eventually become stale unless manually updated
- **Sub-workflow reference:**  
  None.

---

## Block 3: Sales Analytics Computation

### Overview

This block performs the core business logic. It splits the reporting window into previous week and current week, computes totals and growth, aggregates category-level sales, builds HTML rows for the email table, and prepares AI input context.

### Nodes Involved

- Sales Analytics Engine

### Node Details

#### 1. Sales Analytics Engine

- **Type and technical role:** `n8n-nodes-base.code`  
  Custom JavaScript analytics engine for KPI calculation and report payload generation.
- **Configuration choices:**  
  The script:
  - Initializes totals for current and previous week
  - Tracks total quantity sold
  - Builds category maps for current and previous week
  - Aggregates a daily trend for the current week
  - Uses `26-Mar-2026` as the split date:
    - dates on or after split date = current week
    - earlier dates in filtered range = previous week
  - Calculates:
    - formatted sales totals
    - growth percentage
    - growth status text
    - growth color
  - Creates a category comparison HTML table
  - Produces serialized arrays/maps for chart generation
  - Produces `aiContext` for Gemini
  - Sets a display string for the reporting range
- **Key expressions or variables used:**  
  Important output fields:
  - `currentTotalFormatted`
  - `previousTotalFormatted`
  - `totalQty`
  - `growth`
  - `growthStatus`
  - `growthColor`
  - `isGrowthPositive`
  - `prevCatMap`
  - `aiContext`
  - `chartData`
  - `catLabels`
  - `catValues`
  - `categoryTableHtml`
  - `weekRange`
- **Input and output connections:**  
  - Input: `Filter Latest Week1`
  - Output: `Message a model`
- **Version-specific requirements:**  
  Type version `2`.
- **Edge cases or potential failure types:**  
  - Uses `items` directly, so input availability must conform to Code node runtime expectations
  - Hardcoded split date means logic is not dynamic week-over-week
  - If previous week total is zero, growth is forced to `100`
  - Category names are assumed to be present and consistent; spelling variants create separate categories
  - HTML generation is string-based and may break formatting if category names include unexpected characters
- **Sub-workflow reference:**  
  None.

---

## Block 4: AI Insight Generation & Chart Request

### Overview

This block converts raw analytics into a business narrative and triggers a chart render request. The Gemini node generates the sales summary, and the QuickChart request produces a visual comparison asset, although the final email actually embeds a QuickChart URL directly rather than using the downloaded file.

### Nodes Involved

- Message a model
- HTTP Request1

### Node Details

#### 1. Message a model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Sends a prompt to Gemini to generate a concise analytical summary.
- **Configuration choices:**  
  - Model: `models/gemini-2.5-flash`
  - Prompt instructs the model to produce a detailed 3-sentence summary
  - Input prompt includes:
    - `aiContext`
    - `growthStatus`
    - `growth`
    - `totalQty`
  - The prompt asks for:
    1. overall performance
    2. strongest and weakest category mention
    3. total units sold
    4. a focused business recommendation
- **Key expressions or variables used:**  
  - `{{ $json.aiContext }}`
  - `{{ $json.growthStatus }}`
  - `{{ $json.growth }}`
  - `{{ $json.totalQty }}`
- **Input and output connections:**  
  - Input: `Sales Analytics Engine`
  - Output: `HTTP Request1`
- **Version-specific requirements:**  
  Type version `1`. Requires Gemini/Google AI credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Credential/authentication issues
  - Model availability or quota exhaustion
  - AI response schema differences could affect downstream reference to `content.parts[0].text`
  - Prompt may not strictly guarantee only 3 sentences
- **Sub-workflow reference:**  
  None.

#### 2. HTTP Request1

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls QuickChart to generate a chart as a file response.
- **Configuration choices:**  
  - URL: `https://quickchart.io/chart`
  - Sends query parameter `c` containing a Chart.js bar chart definition
  - Response format is configured as `file`
  - Chart compares:
    - Last Week (₹)
    - This Week (₹)
  - Labels come from `catLabels`
  - Previous-week values are rebuilt from `prevCatMap`
  - Current-week values come from `catValues`
- **Key expressions or variables used:**  
  - `$('Sales Analytics Engine').item.json.catLabels`
  - `$('Sales Analytics Engine').item.json.prevCatMap`
  - `$('Sales Analytics Engine').item.json.catValues`
- **Input and output connections:**  
  - Input: `Message a model`
  - Output: `Format Email1`
- **Version-specific requirements:**  
  Type version `4.3`.
- **Edge cases or potential failure types:**  
  - QuickChart service outage or rate limiting
  - Invalid chart JSON if labels or values are malformed
  - This node’s binary output is not actually consumed in the email body
  - If the AI node output structure changes, using `.item` cross-reference to analytics still works, but the current item flow may not include intended binary usage
- **Sub-workflow reference:**  
  None.

**Important implementation note:**  
Although this node downloads a chart file, `Format Email1` does not use the binary output. Instead, the email body constructs a separate QuickChart image URL inline. This makes `HTTP Request1` functionally redundant in the current design unless it is retained for future attachment handling.

---

## Block 5: Email Formatting & Delivery

### Overview

This block composes a styled HTML report and emails it via Gmail. It merges AI-generated commentary with analytics metrics and renders a chart and comparison table inside the email body.

### Nodes Involved

- Format Email1
- Send Email1

### Node Details

#### 1. Format Email1

- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the email subject and full HTML body as structured fields.
- **Configuration choices:**  
  Defines two fields:
  - `emailSubject`
  - `emailBody`
  
  The HTML body includes:
  - report title and week range
  - AI insight text from Gemini
  - KPI cards for current week total and growth
  - embedded QuickChart image using a generated URL
  - detailed category breakdown table from `categoryTableHtml`
- **Key expressions or variables used:**  
  - `{{ $node["Sales Analytics Engine"].json.weekRange }}`
  - `{{ $('Sales Analytics Engine').item.json.weekRange }}`
  - `{{ $json.content.parts[0].text }}`
  - `{{ $('Sales Analytics Engine').item.json.currentTotalFormatted }}`
  - `{{ $('Sales Analytics Engine').item.json.growthColor }}`
  - `{{ $('Sales Analytics Engine').item.json.growthStatus }}`
  - `{{ $('Sales Analytics Engine').item.json.growth }}`
  - `{{ JSON.parse($('Sales Analytics Engine').item.json.catLabels) }}`
  - `{{ JSON.parse($('Sales Analytics Engine').item.json.prevCatMap) }}`
  - `{{ $('Sales Analytics Engine').item.json.catValues }}`
  - `{{ $('Sales Analytics Engine').item.json.categoryTableHtml }}`
- **Input and output connections:**  
  - Input: `HTTP Request1`
  - Output: `Send Email1`
- **Version-specific requirements:**  
  Type version `3`.
- **Edge cases or potential failure types:**  
  - If Gemini returns a different payload shape, `$json.content.parts[0].text` may fail
  - JSON parsing in the embedded chart URL may fail if upstream values are malformed
  - Some email clients may sanitize CSS or render HTML inconsistently
  - Large HTML body or long chart URLs may affect deliverability/rendering
- **Sub-workflow reference:**  
  None.

#### 2. Send Email1

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the generated report through Gmail.
- **Configuration choices:**  
  - Recipient: `YOUR_GMAIL_RECIPIENT_EMAIL`
  - Subject: `={{ $json.emailSubject }}`
  - Message body: `={{ $json.emailBody }}`
  - No extra send options configured
- **Key expressions or variables used:**  
  - `{{ $json.emailSubject }}`
  - `{{ $json.emailBody }}`
- **Input and output connections:**  
  - Input: `Format Email1`
  - Output: none
- **Version-specific requirements:**  
  Type version `2`. Requires Gmail OAuth credentials.
- **Edge cases or potential failure types:**  
  - Gmail authentication or token expiration
  - Invalid recipient address
  - HTML may be sent but rendered differently depending on Gmail/client behavior
  - Sending quotas or account restrictions
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger1 | n8n-nodes-base.scheduleTrigger | Weekly entry point |  | Get row(s) in sheet1 | # Weekly Google Sheets Report → Email Summary<br>### How it works<br>This workflow automatically generates a weekly sales performance report. It starts with a scheduled trigger, fetches raw sales data from Google Sheets, and cleans the dataset by removing empty rows and converting values into usable numeric formats. A custom analytics engine then compares current week vs previous week performance, calculates growth, and builds category-level insights.<br>Next, an AI model generates a short business summary based on the computed metrics. A chart is created using QuickChart to visually compare category performance. Finally, everything is assembled into a clean HTML email including insights, KPIs, and tables, and sent via Gmail.<br>### Setup steps<br>1. Connect your Google Sheets account and select your sheet<br>2. Add your Gemini API credentials for AI summaries<br>3. Connect your Gmail account to send emails<br>4. Update the recipient email address<br>5. Adjust schedule timing if needed<br>### Customization tips<br>- Modify date logic inside the analytics node for custom ranges<br>- Adjust email design styles inside the Set node<br>- Change schedule trigger timing as needed<br>## Step 1: Trigger & Fetch<br>Runs weekly and pulls raw sales data from Google Sheets |
| Get row(s) in sheet1 | n8n-nodes-base.googleSheets | Reads source sales rows from Google Sheets | Schedule Trigger1 | Removing Empty Rows1 | # Weekly Google Sheets Report → Email Summary<br>### How it works<br>This workflow automatically generates a weekly sales performance report. It starts with a scheduled trigger, fetches raw sales data from Google Sheets, and cleans the dataset by removing empty rows and converting values into usable numeric formats. A custom analytics engine then compares current week vs previous week performance, calculates growth, and builds category-level insights.<br>Next, an AI model generates a short business summary based on the computed metrics. A chart is created using QuickChart to visually compare category performance. Finally, everything is assembled into a clean HTML email including insights, KPIs, and tables, and sent via Gmail.<br>### Setup steps<br>1. Connect your Google Sheets account and select your sheet<br>2. Add your Gemini API credentials for AI summaries<br>3. Connect your Gmail account to send emails<br>4. Update the recipient email address<br>5. Adjust schedule timing if needed<br>### Customization tips<br>- Modify date logic inside the analytics node for custom ranges<br>- Adjust email design styles inside the Set node<br>- Change schedule trigger timing as needed<br>## Step 1: Trigger & Fetch<br>Runs weekly and pulls raw sales data from Google Sheets |
| Removing Empty Rows1 | n8n-nodes-base.code | Cleans rows and normalizes numeric fields | Get row(s) in sheet1 | Filter Latest Week1 | ## Step 2: Clean & Analyze<br>Cleans data and calculates weekly performance metrics |
| Filter Latest Week1 | n8n-nodes-base.code | Filters rows to a hardcoded reporting window | Removing Empty Rows1 | Sales Analytics Engine | ## Step 2: Clean & Analyze<br>Cleans data and calculates weekly performance metrics |
| Sales Analytics Engine | n8n-nodes-base.code | Computes totals, growth, category comparisons, and AI context | Filter Latest Week1 | Message a model | ## Step 2: Clean & Analyze<br>Cleans data and calculates weekly performance metrics |
| Message a model | @n8n/n8n-nodes-langchain.googleGemini | Generates AI summary from analytics context | Sales Analytics Engine | HTTP Request1 | ## Step 3: AI & Chart<br>Generates insights and builds visual performance chart |
| HTTP Request1 | n8n-nodes-base.httpRequest | Requests QuickChart chart image/file | Message a model | Format Email1 | ## Step 3: AI & Chart<br>Generates insights and builds visual performance chart |
| Format Email1 | n8n-nodes-base.set | Builds HTML email subject and body | HTTP Request1 | Send Email1 | ## Step 4: Format & Send<br>Formats report into HTML and sends via Gmail |
| Send Email1 | n8n-nodes-base.gmail | Sends final report email via Gmail | Format Email1 |  | ## Step 4: Format & Send<br>Formats report into HTML and sends via Gmail |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas documentation for cleaning/analysis block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas documentation for AI/chart block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas documentation for email block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Global workflow description and setup notes |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Canvas documentation for trigger/fetch block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Generate weekly sales insights with Google Sheets, Gemini AI and Gmail`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure it to run weekly
   - Set:
     - interval field: `weeks`
     - trigger day: Monday
     - trigger hour: `21`
   - Confirm your n8n instance timezone matches the business reporting timezone.

3. **Add a Google Sheets node**
   - Node type: `Google Sheets`
   - Operation: get rows / read sheet rows
   - Connect it after the Schedule Trigger
   - Select or paste your spreadsheet ID in `documentId`
   - Select the worksheet, here `Sheet1` / `gid=0`
   - Authenticate with Google Sheets credentials
   - Ensure the sheet has these expected columns:
     - `Date`
     - `Day`
     - `Product type`
     - `Total sales (₹)`
     - `Net quantity`

4. **Add a Code node for data cleaning**
   - Node type: `Code`
   - Connect it after the Google Sheets node
   - Paste logic equivalent to:
     - read all input items
     - remove rows with missing `Date` or `Product type`
     - convert `Total sales (₹)` from formatted currency string to a number
     - convert `Net quantity` to number
     - keep all original fields
     - add `IsCleaned: true`
   - Use the current workflow logic exactly if you want matching output behavior.

5. **Add a second Code node for date filtering**
   - Node type: `Code`
   - Connect it after the cleaning node
   - Configure it to:
     - parse dates in `DD-MMM-YYYY` format
     - keep only rows between:
       - `19-Mar-2026`
       - `01-Apr-2026 23:59:59`
   - Important: this logic is hardcoded. If you want the workflow to remain useful over time, replace this with dynamic weekly date logic.

6. **Add a third Code node named Sales Analytics Engine**
   - Node type: `Code`
   - Connect it after the filter node
   - Configure it to:
     - split records at `26-Mar-2026`
     - treat rows on/after split date as current week
     - treat rows before split date as previous week
     - calculate:
       - current total sales
       - previous total sales
       - total quantity
       - growth percent
       - growth direction text
       - growth color
     - aggregate sales by category for both weeks
     - build HTML rows for a category comparison table
     - produce serialized chart arrays/maps
     - generate an `aiContext` string
     - output a `weekRange` string like `26-Mar-2026 to 01-Apr-2026`
   - Make sure the node outputs one item with all required report fields.

7. **Add a Gemini node**
   - Node type: `Google Gemini` from the LangChain/n8n AI nodes
   - Connect it after `Sales Analytics Engine`
   - Authenticate with Google Gemini credentials/API access
   - Select model: `models/gemini-2.5-flash`
   - Add one message prompt instructing the model to write a 3-sentence retail analysis using:
     - `aiContext`
     - `growthStatus`
     - `growth`
     - `totalQty`
   - Keep the prompt aligned with the original behavior:
     - summarize overall performance
     - mention strongest and weakest category performance
     - mention total units sold
     - end with a recommendation

8. **Add an HTTP Request node for QuickChart**
   - Node type: `HTTP Request`
   - Connect it after the Gemini node
   - Set:
     - Method: `GET`
     - URL: `https://quickchart.io/chart`
     - Send Query Parameters: enabled
   - Add query parameter:
     - Name: `c`
     - Value: a Chart.js JSON config using expressions from `Sales Analytics Engine`
   - Configure response format as `File`
   - Note: in the provided workflow, this file is not used later. You may skip this node if you only want the inline image URL method, or keep it for parity with the original workflow.

9. **Add a Set node for email formatting**
   - Node type: `Set`
   - Connect it after the HTTP Request node
   - Create fields:
     - `emailSubject`
     - `emailBody`
   - Subject should reference the week range, for example:  
     `📊 Weekly Sales Report: {{ weekRange }}`
   - Body should be HTML and include:
     - header with reporting week
     - AI summary text using Gemini output, typically `content.parts[0].text`
     - KPI cards for:
       - current week total
       - growth indicator
     - image tag using QuickChart URL with embedded Chart.js config
     - category comparison table using `categoryTableHtml`
   - Use expressions referencing `Sales Analytics Engine` outputs for all metrics and chart inputs.

10. **Add a Gmail node**
    - Node type: `Gmail`
    - Connect it after the Set node
    - Authenticate with Gmail OAuth2 credentials
    - Configure:
      - recipient email address
      - subject from `emailSubject`
      - message body from `emailBody`
    - Ensure the node sends HTML email content.

11. **Verify all node connections in this order**
    - `Schedule Trigger1` → `Get row(s) in sheet1`
    - `Get row(s) in sheet1` → `Removing Empty Rows1`
    - `Removing Empty Rows1` → `Filter Latest Week1`
    - `Filter Latest Week1` → `Sales Analytics Engine`
    - `Sales Analytics Engine` → `Message a model`
    - `Message a model` → `HTTP Request1`
    - `HTTP Request1` → `Format Email1`
    - `Format Email1` → `Send Email1`

12. **Configure credentials**
    - Google Sheets credentials for spreadsheet access
    - Gemini credentials for AI generation
    - Gmail OAuth2 credentials for email sending

13. **Replace placeholders**
    - `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`
    - `YOUR_GMAIL_RECIPIENT_EMAIL`

14. **Test with manual execution**
    - Run from the trigger or from the Google Sheets node
    - Confirm:
      - rows are returned
      - cleaned numeric fields are correct
      - date filtering keeps the intended rows
      - AI summary appears in the expected response field
      - HTML preview renders correctly
      - Gmail sends successfully

15. **Optional hardening improvements**
    - Replace hardcoded date ranges with dynamic “last 7 days vs prior 7 days”
    - Add an IF node for empty datasets
    - Add error handling for missing columns or bad dates
    - Remove the unused QuickChart file request if unnecessary
    - Validate Gemini output before referencing `content.parts[0].text`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Weekly Google Sheets Report → Email Summary | Workflow-level title shown on the canvas |
| This workflow automatically generates a weekly sales performance report. It starts with a scheduled trigger, fetches raw sales data from Google Sheets, and cleans the dataset by removing empty rows and converting values into usable numeric formats. A custom analytics engine then compares current week vs previous week performance, calculates growth, and builds category-level insights. Next, an AI model generates a short business summary based on the computed metrics. A chart is created using QuickChart to visually compare category performance. Finally, everything is assembled into a clean HTML email including insights, KPIs, and tables, and sent via Gmail. | Global workflow description |
| Connect your Google Sheets account and select your sheet | Setup guidance |
| Add your Gemini API credentials for AI summaries | Setup guidance |
| Connect your Gmail account to send emails | Setup guidance |
| Update the recipient email address | Setup guidance |
| Adjust schedule timing if needed | Setup guidance |
| Modify date logic inside the analytics node for custom ranges | Customization guidance |
| Adjust email design styles inside the Set node | Customization guidance |
| Change schedule trigger timing as needed | Customization guidance |
| QuickChart chart rendering service | https://quickchart.io/chart |

## Additional implementation observations

- The reporting dates are fully hardcoded to a specific period in 2026. Without modification, the workflow will not behave as a true rolling weekly report.
- The QuickChart HTTP node is currently not required for email rendering because the email body generates a direct chart URL independently.
- The Gemini output path is assumed to be `content.parts[0].text`; if the model response format differs in your n8n version, update the expression in the Set node.
- The workflow has a single entry point and does not call any sub-workflows.