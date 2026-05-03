Generate Financial Reports with AI Insights, Budget Analysis & Smart Alerts

https://n8nworkflows.xyz/workflows/generate-financial-reports-with-ai-insights--budget-analysis---smart-alerts-10677


# Generate Financial Reports with AI Insights, Budget Analysis & Smart Alerts

---
### 1. Workflow Overview

This workflow automates the generation and distribution of monthly financial reports enriched with AI-generated insights and anomaly detection. It is designed for finance teams to streamline month-end reporting, integrate data from accounting systems, and deliver actionable executive summaries while ensuring critical reports receive CFO review.

Logical blocks include:

- **1.1 Scheduling & Period Calculation:** Automatic monthly trigger and calculation of current and previous reporting periods.
- **1.2 Data Fetching:** Pulls Profit & Loss (P&L) data for current and previous periods from the accounting system API.
- **1.3 Data Merging & Analysis:** Combines datasets, calculates growth metrics, detects anomalies, and computes a financial health score.
- **1.4 AI Insight Generation:** Prepares context for AI, sends data to an AI agent for executive summary, insights, and recommendations.
- **1.5 Report Preparation:** Parses AI output, compiles report metadata and stakeholder info, and formats report content.
- **1.6 Approval Workflow:** Conditional routing for CFO review based on report criticality.
- **1.7 Report Generation & Archival:** Generates HTML report, converts to PDF, saves to Google Drive.
- **1.8 Notification & Distribution:** Sends emails to stakeholders and posts context-aware Slack alerts with criticality-based formatting.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Period Calculation

- **Overview:** Triggers workflow monthly on the 1st at 9 AM and computes reporting periods for current and previous months.
- **Nodes Involved:** `Schedule Monthly`, `Calculate Period`, `Sticky Note1`
- **Node Details:**

  - **Schedule Monthly**
    - Type: Schedule Trigger
    - Role: Workflow start trigger based on cron expression (`0 9 1 * *` = 9 AM on 1st of each month)
    - Inputs: None
    - Outputs: Triggers `Calculate Period`
    - Potential Failures: Scheduler misconfiguration
    
  - **Calculate Period**
    - Type: Code (JavaScript)
    - Role: Calculates start/end dates for current reporting month and previous comparison month.
    - Configuration: Uses JavaScript Date API to derive ISO-formatted dates and month/year labels.
    - Inputs: Trigger from `Schedule Monthly`
    - Outputs: JSON with `reportPeriod`, `previousPeriod`, `reportDate`
    - Edge Cases: Timezone differences, month boundaries (e.g., January edge case handled by Date API)
    
  - **Sticky Note1**
    - Type: Sticky Note
    - Role: Documents scheduling and data collection logic for user clarity.
    - Inputs/Outputs: None

#### 1.2 Data Fetching

- **Overview:** Retrieves P&L data for current and previous periods from accounting system APIs.
- **Nodes Involved:** `Fetch Current P&L`, `Fetch Previous P&L`, `Merge Data`
- **Node Details:**

  - **Fetch Current P&L**
    - Type: HTTP Request
    - Role: Calls accounting API endpoint for P&L data over current reporting period.
    - Configuration: URL parameterized with `reportPeriod.startDate` and `reportPeriod.endDate`.
    - Authentication: Generic HTTP Header
    - Inputs: Output from `Calculate Period`
    - Outputs: JSON P&L data for current period
    - Failure Types: API authentication errors, network timeouts, invalid date parameters
    
  - **Fetch Previous P&L**
    - Type: HTTP Request
    - Role: Calls same API for previous period data using `previousPeriod` dates.
    - Configuration: Similar to current P&L node with different date parameters.
    - Inputs: Output from `Calculate Period`
    - Outputs: JSON P&L data for previous period
    - Failure Types: Same as above
    
  - **Merge Data**
    - Type: Merge
    - Role: Combines current and previous period data into single array for analysis.
    - Configuration: Mode "combine" to merge parallel inputs.
    - Inputs: Outputs from both HTTP Request nodes
    - Outputs: Combined JSON array with two data points.
    - Edge Cases: Missing data from either period, malformed responses

#### 1.3 Data Merging & Analysis

- **Overview:** Analyzes merged financial data to compute growth rates, detect anomalies, and calculate a health score.
- **Nodes Involved:** `Analyze Financial Data`, `Sticky Note2`
- **Node Details:**

  - **Analyze Financial Data**
    - Type: Code (JavaScript)
    - Role: Processes P&L data; calculates revenue, expenses, net income, growth percentages, and anomalies.
    - Key Logic:
      - Growth thresholds: revenue > 20%, expenses > 15%, budget variance > 25% trigger anomalies.
      - Health score starts at 50, adjusted by conditions (net income positive +20, revenue growth +15, etc.)
      - Anomaly array with severity tags and messages.
    - Inputs: Combined data from `Merge Data`
    - Outputs: JSON including all metrics, anomalies, health score, and review flag.
    - Edge Cases: Zero or missing values, division by zero in growth calculations, empty categories list.

  - **Sticky Note2**
    - Type: Sticky Note
    - Role: Explains financial analysis and anomaly logic for users.
    - Inputs/Outputs: None

#### 1.4 AI Insight Generation

- **Overview:** Prepares a formatted context string and uses AI (GPT-4) to generate executive summary, insights, and actionable recommendations.
- **Nodes Involved:** `Prepare AI Context`, `AI Financial Insights`, `OpenAI Chat Model`
- **Node Details:**

  - **Prepare AI Context**
    - Type: Set
    - Role: Constructs a multi-line string embedding key financial metrics, anomalies, and instructions for AI output.
    - Inputs: Output from `Analyze Financial Data`
    - Outputs: JSON with property `aiContext` containing prompt text for AI.
    
  - **AI Financial Insights**
    - Type: LangChain Agent (Converstional Agent)
    - Role: Sends context prompt to AI agent with system message as CFO advisor.
    - Configuration: Uses custom prompt with strict JSON response format.
    - Inputs: `aiContext` from `Prepare AI Context`
    - Outputs: AI-generated JSON with executive summary, insights, and recommendations.
    - Potential Failures: API rate limits, malformed responses, JSON parse errors.
    
  - **OpenAI Chat Model**
    - Type: LangChain OpenAI Chat Model
    - Role: Backend model set to `gpt-4-turbo` used by AI Financial Insights node.
    - Inputs: Connected internally by LangChain agent.
    - Outputs: AI response to agent node.

#### 1.5 Report Preparation

- **Overview:** Parses AI output, merges with company metadata, and prepares data for report generation.
- **Nodes Involved:** `Prepare Report Data`
- **Node Details:**

  - **Prepare Report Data**
    - Type: Code (JavaScript)
    - Role: Parses AI JSON safely, sets default fallback if parsing fails; adds company info, report ID, and formatted dates.
    - Inputs: Output from `AI Financial Insights` and `Analyze Financial Data`
    - Outputs: JSON combining analysis, AI insights, and metadata for report rendering.
    - Edge Cases: Malformed AI JSON, missing fields, date formatting errors.

#### 1.6 Approval Workflow

- **Overview:** Decides if CFO review is required (based on health score and anomaly count) and routes accordingly.
- **Nodes Involved:** `Requires CFO Review?`, `Request CFO Review`, `Sticky Note7`
- **Node Details:**

  - **Requires CFO Review?**
    - Type: If
    - Role: Checks if `requiresReview` flag is true (anomalies > 2 or health score < 50).
    - Inputs: Output from `Prepare Report Data`
    - Outputs: Routes to `Request CFO Review` if true; else to report generation.
    
  - **Request CFO Review**
    - Type: Gmail (Send Email)
    - Role: Sends report for CFO approval before distribution.
    - Inputs: From `Requires CFO Review?`
    - Outputs: Proceeds to `Generate Report HTML` after approval.
    - Failure Types: Email auth errors, invalid email addresses.
    
  - **Sticky Note7**
    - Type: Sticky Note
    - Role: Documents approval routing logic.
    - Inputs/Outputs: None

#### 1.7 Report Generation & Archival

- **Overview:** Generates detailed HTML report, converts to PDF, and saves to Google Drive.
- **Nodes Involved:** `Generate Report HTML`, `HTML to PDF`, `Save to Google Drive`, `Sticky Note6`
- **Node Details:**

  - **Generate Report HTML**
    - Type: Code (JavaScript)
    - Role: Creates comprehensive HTML including color-coded health banner, AI insights, anomalies, P&L tables, and budget variance.
    - Inputs: Output from `Request CFO Review` or directly from `Requires CFO Review?`
    - Outputs: JSON with `html`, `fileName`, and report content.
    - Edge Cases: Large data sets may produce large HTML; malformed input fields.
    
  - **HTML to PDF**
    - Type: HTML to PDF (htmlcsstopdf)
    - Role: Converts generated HTML to PDF format.
    - Inputs: HTML content from `Generate Report HTML`
    - Outputs: PDF binary for archival and distribution.
    - Failure Types: API limits, HTML rendering issues.
    
  - **Save to Google Drive**
    - Type: Google Drive
    - Role: Uploads PDF file to Google Drive root folder.
    - Inputs: PDF from `HTML to PDF`
    - Outputs: Metadata including web view link for sharing.
    - Edge Cases: Insufficient permissions, quota limits.
    
  - **Sticky Note6**
    - Type: Sticky Note
    - Role: Explains report generation and archival steps.

#### 1.8 Notification & Distribution

- **Overview:** Sends emails to stakeholders and posts Slack alerts with formatting based on report criticality.
- **Nodes Involved:** `Send to Stakeholders`, `Health Critical?`, `Alert - Critical`, `Notify - Standard`, `Sticky Note8`
- **Node Details:**

  - **Send to Stakeholders**
    - Type: Gmail (Send Email)
    - Role: Distributes report PDF via email to predefined stakeholders (CEO, CFO, Board).
    - Inputs: Drive link and report metadata from `Save to Google Drive`
    - Outputs: Triggers health check for alert routing.
    
  - **Health Critical?**
    - Type: If
    - Role: Evaluates if health score < 60 to route to critical or standard Slack alerts.
    - Inputs: Output from `Send to Stakeholders`
    
  - **Alert - Critical**
    - Type: HTTP Request (Slack Webhook)
    - Role: Sends urgent Slack message with immediate attention flag.
    - Inputs: From `Health Critical?` true branch
    - Configuration: Slack webhook URL, JSON payload with report details and link.
    - Failure Types: Webhook invalid, network errors.
    
  - **Notify - Standard**
    - Type: HTTP Request (Slack Webhook)
    - Role: Sends normal Slack notification summarizing report.
    - Inputs: From `Health Critical?` false branch
    - Configuration: Slack webhook URL, JSON payload with report summary.
    
  - **Sticky Note8**
    - Type: Sticky Note
    - Role: Describes Slack alert routing and formatting.

---

### 3. Summary Table

| Node Name            | Node Type                     | Functional Role                          | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                           |
|----------------------|-------------------------------|----------------------------------------|---------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note                   | Workflow intro and setup instructions  |                           |                               | Explains overall workflow purpose, setup steps, and requirements.                                  |
| Schedule Monthly     | Schedule Trigger              | Monthly trigger at 9 AM on 1st         |                           | Calculate Period              | Scheduling & Data Collection description.                                                          |
| Sticky Note1         | Sticky Note                   | Scheduling & data fetch explanation    |                           |                               | Scheduling & Data Collection description.                                                          |
| Calculate Period     | Code                          | Calculates current and previous periods| Schedule Monthly           | Fetch Current P&L, Fetch Previous P&L |                                                                                                     |
| Fetch Current P&L    | HTTP Request                  | Fetch current period P&L data           | Calculate Period           | Merge Data                    |                                                                                                     |
| Fetch Previous P&L   | HTTP Request                  | Fetch previous period P&L data          | Calculate Period           | Merge Data                    |                                                                                                     |
| Merge Data           | Merge                         | Combine current and previous P&L data  | Fetch Current P&L, Fetch Previous P&L | Analyze Financial Data          |                                                                                                     |
| Analyze Financial Data| Code                          | Calculate metrics, anomalies, health   | Merge Data                 | Prepare AI Context            | Financial Analysis & AI explanation.                                                               |
| Sticky Note2         | Sticky Note                   | Financial analysis & anomaly logic     |                           |                               | Financial Analysis & AI explanation.                                                               |
| Prepare AI Context   | Set                           | Prepare AI prompt context string        | Analyze Financial Data     | AI Financial Insights         |                                                                                                     |
| AI Financial Insights| LangChain Conversational Agent| Request AI-generated insights in JSON | Prepare AI Context         | Prepare Report Data           |                                                                                                     |
| OpenAI Chat Model    | LangChain OpenAI Chat         | GPT-4-turbo model for AI insights       | AI Financial Insights (internal) | AI Financial Insights (internal) |                                                                                                     |
| Prepare Report Data  | Code                          | Parse AI output, add metadata, format  | AI Financial Insights, Analyze Financial Data | Requires CFO Review?           |                                                                                                     |
| Requires CFO Review? | If                            | Decide if CFO approval needed           | Prepare Report Data        | Request CFO Review, Generate Report HTML | Approval Routing description.                                                                        |
| Request CFO Review   | Gmail (Send Email)            | Send report for CFO approval            | Requires CFO Review?       | Generate Report HTML          | Approval Routing description.                                                                        |
| Sticky Note7         | Sticky Note                   | Approval routing explanation            |                           |                               | Approval Routing description.                                                                        |
| Generate Report HTML | Code                          | Generate full HTML financial report     | Request CFO Review or Requires CFO Review? | HTML to PDF                   | Report Generation & Distribution explanation.                                                      |
| HTML to PDF          | HTML to PDF                   | Convert HTML report to PDF               | Generate Report HTML       | Save to Google Drive          | Report Generation & Distribution explanation.                                                      |
| Save to Google Drive | Google Drive                  | Archive PDF report to Drive              | HTML to PDF                | Send to Stakeholders          | Report Generation & Distribution explanation.                                                      |
| Sticky Note6         | Sticky Note                   | Report generation & archival description|                           |                               | Report Generation & Distribution explanation.                                                      |
| Send to Stakeholders | Gmail (Send Email)            | Email report to stakeholders             | Save to Google Drive       | Health Critical?              |                                                                                                     |
| Health Critical?     | If                            | Check if health score < 60 for alert    | Send to Stakeholders       | Alert - Critical, Notify - Standard | Context-Aware Alerts description.                                                                   |
| Alert - Critical     | HTTP Request (Slack Webhook)  | Post urgent Slack alert                  | Health Critical? (true)    |                               | Context-Aware Alerts description.                                                                   |
| Notify - Standard    | HTTP Request (Slack Webhook)  | Post regular Slack notification          | Health Critical? (false)   |                               | Context-Aware Alerts description.                                                                   |
| Sticky Note8         | Sticky Note                   | Slack alert routing explanation          |                           |                               | Context-Aware Alerts description.                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add a **Schedule Trigger** node named `Schedule Monthly`.
   - Set cron expression to `0 9 1 * *` (9 AM on the 1st day of every month).

2. **Calculate Reporting Period:**
   - Add a **Code** node named `Calculate Period`.
   - Connect input from `Schedule Monthly`.
   - Paste JavaScript to calculate:
     - Current month start/end dates.
     - Previous month start/end dates.
     - Format dates as ISO strings.
   - Output JSON with `reportPeriod`, `previousPeriod`, `reportDate`.

3. **Fetch Current P&L Data:**
   - Add an **HTTP Request** node named `Fetch Current P&L`.
   - Connect input from `Calculate Period`.
   - Configure:
     - URL: `https://api.youraccounting.com/reports/pl`
     - Query Parameters: `start_date={{ $json.reportPeriod.startDate }}`, `end_date={{ $json.reportPeriod.endDate }}`
     - Authentication: HTTP Header with your accounting system API key.
   - Set to `GET`.

4. **Fetch Previous P&L Data:**
   - Add an **HTTP Request** node named `Fetch Previous P&L`.
   - Connect input from `Calculate Period`.
   - Configure similarly but with `start_date={{ $json.previousPeriod.startDate }}`, `end_date={{ $json.previousPeriod.endDate }}`.

5. **Merge P&L Data:**
   - Add a **Merge** node named `Merge Data`.
   - Connect inputs from both P&L fetch nodes.
   - Set mode to `combine`.

6. **Analyze Financial Data:**
   - Add a **Code** node named `Analyze Financial Data`.
   - Connect input from `Merge Data`.
   - Paste JavaScript to:
     - Extract revenue, expenses, net income for both periods.
     - Calculate growth rates.
     - Analyze budget categories for variances.
     - Detect anomalies based on thresholds.
     - Compute health score and review necessity flag.

7. **Prepare AI Context:**
   - Add a **Set** node named `Prepare AI Context`.
   - Connect input from `Analyze Financial Data`.
   - Create a string assignment `aiContext` containing all key metrics, anomalies, and instructions for AI in JSON format.

8. **Add OpenAI Chat Model:**
   - Add a **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.
   - Set model to `gpt-4-turbo`.
   - No direct connections; used internally by agent.

9. **Configure AI Financial Insights Agent:**
   - Add a **LangChain Conversational Agent** node named `AI Financial Insights`.
   - Connect input from `Prepare AI Context`.
   - Set prompt type to "define" with system message: "You are a CFO advisor analyzing financial reports. Provide concise, actionable insights. Always respond with valid JSON only."
   - Use `OpenAI Chat Model` as underlying model.

10. **Prepare Report Data:**
    - Add a **Code** node named `Prepare Report Data`.
    - Connect input from `AI Financial Insights`.
    - Paste JavaScript to:
      - Parse AI JSON safely.
      - Add company name, logo, report title, fiscal year, prepared by info.
      - Generate unique report ID.
      - Format report period dates.
      - Include stakeholders list (CEO, CFO emails).
      - Combine all into JSON output.

11. **Add Approval Conditional:**
    - Add an **If** node named `Requires CFO Review?`.
    - Connect input from `Prepare Report Data`.
    - Condition: Check Boolean `requiresReview` equals `true`.

12. **Request CFO Review Email:**
    - Add a **Gmail Send Email** node named `Request CFO Review`.
    - Connect from `Requires CFO Review?` true branch.
    - Configure to send an email with report details to CFO for approval.

13. **Generate Report HTML:**
    - Add a **Code** node named `Generate Report HTML`.
    - Connect from `Request CFO Review` (after approval) and also from `Requires CFO Review?` false branch.
    - Use provided HTML and CSS template to render report with:
      - Health score banner.
      - AI insights and recommendations.
      - Anomalies and alerts.
      - P&L statement tables.
      - Budget variance analysis.
      - Footer with report metadata.

14. **Convert HTML to PDF:**
    - Add **HTML to PDF** node.
    - Connect input from `Generate Report HTML`.
    - Use default settings.

15. **Save PDF to Google Drive:**
    - Add **Google Drive** node named `Save to Google Drive`.
    - Connect input from `HTML to PDF`.
    - Set file name from `Generate Report HTML`.
    - Select folder (e.g., root or custom).
    - Authenticate with Google Drive OAuth2.

16. **Email Stakeholders:**
    - Add **Gmail Send Email** node named `Send to Stakeholders`.
    - Connect input from `Save to Google Drive`.
    - Configure recipients: CEO, CFO, Board emails.
    - Attach PDF or provide Drive link.

17. **Health Score Check for Alerts:**
    - Add **If** node named `Health Critical?`.
    - Connect input from `Send to Stakeholders`.
    - Condition: `healthScore < 60`.

18. **Slack Alert - Critical:**
    - Add **HTTP Request** node named `Alert - Critical`.
    - Connect input from `Health Critical?` true branch.
    - Configure URL with Slack Incoming Webhook.
    - Send JSON payload with urgent alert format including report ID, period, health score, anomalies, and Drive link.

19. **Slack Alert - Standard:**
    - Add **HTTP Request** node named `Notify - Standard`.
    - Connect input from `Health Critical?` false branch.
    - Configure URL with Slack Incoming Webhook.
    - Send JSON payload summarizing report publication.

20. **Add Sticky Notes:**
    - Add sticky notes as per original positioning to document key process blocks for clarity and user instructions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow automates month-end financial reporting with AI-powered insights and anomaly detection. Requires API access to accounting system with P&L endpoints. | Workflow overview sticky note at start. |
| Setup requires API credentials for accounting system, OpenAI API key (GPT-4 recommended), Google Drive and Gmail OAuth authentication, Slack webhook URL. | Setup instructions sticky note. |
| Adjust anomaly detection thresholds in the "Analyze Financial Data" code node as needed. | Sticky note near analysis nodes. |
| HTML to PDF conversion uses htmlcsstoimage.com API; obtain API key accordingly. | Report generation sticky note. |
| Stakeholders list is editable in "Prepare Report Data" node; defaults include CEO and CFO. | Report preparation node note. |
| Slack alerts use different formatting for critical (health score <60) vs normal reports. | Alert sticky note. |

---

**Disclaimer:**  
The provided content is extracted solely from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected materials. All data processed is legal and public.