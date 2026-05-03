Daily Cash Flow Reports with Google Sheets, Slack & Email for Finance Teams

https://n8nworkflows.xyz/workflows/daily-cash-flow-reports-with-google-sheets--slack---email-for-finance-teams-10109


# Daily Cash Flow Reports with Google Sheets, Slack & Email for Finance Teams

### 1. Workflow Overview

This workflow automates the daily generation and distribution of a financial cash flow report for finance teams. It is designed for finance professionals who need an end-of-day summary of cash inflows and outflows to monitor the organization's cash position.

The workflow executes every day at 6:00 PM, fetching today's cash transaction data (inflows and outflows) from an accounting API, calculating totals and category breakdowns, then compiling a net cash flow summary. The resulting report is saved into Google Sheets for historical tracking, converted into a PDF, emailed to finance stakeholders, posted as a summary message on Slack, and backed up to Google Drive.

**Logical blocks:**

- **1.1 Scheduling Trigger:** Initiates the workflow daily at a fixed time.
- **1.2 Data Retrieval:** Fetches today's cash inflows and outflows from an external accounting API.
- **1.3 Data Calculations:** Processes and summarizes inflows and outflows by category and counts.
- **1.4 Data Merging and Net Calculation:** Combines inflow and outflow data and computes net cash flow and status.
- **1.5 Report Generation:** Creates an HTML report summarizing cash flows by category.
- **1.6 Persistence and Delivery:** Saves data to Google Sheets, converts the report to PDF, emails it, posts a Slack message, and backs up the report to Google Drive.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling Trigger

- **Overview:**  
Triggers the entire workflow once daily at 6:00 PM to generate and distribute the cash flow report.

- **Nodes Involved:**  
  - Daily at 6 PM

- **Node Details:**  
  - **Daily at 6 PM**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression set to trigger daily at 18:00 (6 PM).  
    - Input: None (trigger node).  
    - Output: Initiates parallel requests to get inflows and outflows nodes.  
    - Potential Failures: Cron misconfiguration (unlikely if properly set), n8n server downtime at trigger time.

#### 1.2 Data Retrieval

- **Overview:**  
Retrieves today's cash inflows and outflows from the accounting API endpoints.

- **Nodes Involved:**  
  - Get Cash Inflows  
  - Get Cash Outflows

- **Node Details:**  
  - **Get Cash Inflows**  
    - Type: HTTP Request  
    - Role: Fetches all incoming payments (deposits, revenue) for the day.  
    - URL: `https://api.accounting.com/transactions` (likely with query filters in real use, not shown here).  
    - Input: Trigger from schedule node.  
    - Output: JSON array of inflow transactions.  
    - Potential Failures: Network errors, API authentication issues, empty or malformed response.  
    - Notes: Retrieves only today's deposits and revenue.  
  - **Get Cash Outflows**  
    - Type: HTTP Request  
    - Role: Fetches all outgoing payments (expenses, bills) for the day.  
    - Configuration similar to inflows node.  
    - Same potential failure considerations.

#### 1.3 Data Calculations

- **Overview:**  
Processes raw inflow and outflow data to calculate totals, group by categories, and count transactions.

- **Nodes Involved:**  
  - Calculate Inflows  
  - Calculate Outflows

- **Node Details:**  
  - **Calculate Inflows**  
    - Type: Code (JavaScript)  
    - Role: Sums total inflow amount, groups by category, counts transactions, attaches current date.  
    - Key Expressions: Parses amounts safely, defaults missing categories to 'Other'.  
    - Inputs: JSON array from Get Cash Inflows node.  
    - Outputs: Object with total_inflow, inflow_categories (object with category sums), inflow_count, date.  
    - Edge Cases: Empty input array, malformed amounts, missing categories handled by defaults.  
  - **Calculate Outflows**  
    - Type: Code (JavaScript)  
    - Role: Similar to Calculate Inflows but processes outflows data.  
    - Same considerations on input validation and defaults.

#### 1.4 Data Merging and Net Calculation

- **Overview:**  
Merges inflow and outflow summaries and calculates the net cash flow and status.

- **Nodes Involved:**  
  - Merge Data  
  - Calculate Net Cash Flow

- **Node Details:**  
  - **Merge Data**  
    - Type: Merge Node  
    - Role: Combines outputs of Calculate Inflows and Calculate Outflows by position.  
    - Input: Two streams from inflows and outflows calculation nodes.  
    - Output: Paired inflow and outflow data objects.  
    - Potential Issues: Misalignment of data if nodes produce different item counts (unlikely here).  
  - **Calculate Net Cash Flow**  
    - Type: Code (JavaScript)  
    - Role: Subtracts total outflows from inflows, determines cash flow status (Positive/Negative), preserves breakdowns and counts.  
    - Inputs: Array with two items from merged node (inflow data at index 0, outflow data at index 1).  
    - Outputs: Final summary object with net_cash_flow, status, and all relevant data fields.  
    - Edge Cases: Non-numeric or missing totals handled by defaults (0).

#### 1.5 Report Generation

- **Overview:**  
Generates a styled HTML report including summaries and detailed category breakdown tables.

- **Nodes Involved:**  
  - Save to Google Sheets  
  - Generate HTML Report  
  - Convert to PDF

- **Node Details:**  
  - **Save to Google Sheets**  
    - Type: Google Sheets Node  
    - Role: Appends daily cash flow summary data to a dedicated sheet for historical tracking.  
    - Configuration: Uses service account authentication, targets sheet "Daily_Cash_Flow" in document ID `9x8w7v6u5t4s3r2q`.  
    - Mapping: Date, Total Inflow, Total Outflow, Net Cash Flow, Status, Inflow Count, Outflow Count.  
    - Potential Issues: Authentication errors, API quota limits, sheet access permissions.  
  - **Generate HTML Report**  
    - Type: Code (JavaScript)  
    - Role: Builds HTML string with embedded CSS styles summarizing the cash flow report with tables for inflows and outflows by category.  
    - Inputs: JSON summary from Calculate Net Cash Flow.  
    - Outputs: Object including `html_report` string plus all summary fields.  
    - Edge Cases: Empty categories produce empty tables; all numbers formatted to two decimals.  
  - **Convert to PDF**  
    - Type: PDFMonkey Node  
    - Role: Converts the generated report into a professional PDF using a specified template.  
    - Configuration: Uses documentTemplateId parameter referencing a stored template.  
    - Inputs: Report data (likely via expression or previous node’s output).  
    - Potential Issues: API key or template errors, PDF generation failures.

#### 1.6 Persistence and Delivery

- **Overview:**  
Delivers the report via email, posts a summary to Slack, and backs up the report to Google Drive.

- **Nodes Involved:**  
  - Email Report  
  - Post to Slack  
  - Backup to Google Drive

- **Node Details:**  
  - **Email Report**  
    - Type: Email Send Node  
    - Role: Sends the daily report email to finance@company.com and cfo@company.com, CC to accounting@company.com, with PDF attachment.  
    - Configuration: Uses SMTP credentials, dynamic subject line with current date, attaches base64 PDF from Convert to PDF node.  
    - Inputs: PDF data and summary fields.  
    - Potential Failures: SMTP authentication errors, attachment handling, network issues.  
  - **Post to Slack**  
    - Type: Slack Node  
    - Role: Posts a concise summary message into #daily-reports channel for team visibility.  
    - Configuration: Uses Slack API credentials, formatted message with date, totals, net cash flow, and status.  
    - Inputs: JSON summary from Generate HTML Report node.  
    - Potential Failures: Slack API rate limits, authentication.  
  - **Backup to Google Drive**  
    - Type: Google Drive Node  
    - Role: Uploads report backup files to a designated folder `/finance/reports/` in Google Drive for archival.  
    - Configuration: Uses OAuth2 credentials, target folder is root by default (likely configured in real scenario).  
    - Potential Issues: Permission issues, storage limits.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                         | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                   |
|---------------------|----------------------------|---------------------------------------|-----------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------|
| Daily at 6 PM       | Schedule Trigger           | Triggers workflow daily at 6 PM       | None                              | Get Cash Inflows, Get Cash Outflows | ⏰ Triggers daily at 6 PM                                                                     |
| Get Cash Inflows    | HTTP Request               | Fetches today's cash inflows          | Daily at 6 PM                    | Calculate Inflows                   | 💵 Fetches cash inflows (deposits, revenue)                                                  |
| Get Cash Outflows   | HTTP Request               | Fetches today's cash outflows         | Daily at 6 PM                    | Calculate Outflows                  | 💸 Fetches cash outflows (expenses, bills)                                                   |
| Calculate Inflows   | Code                       | Calculates inflow totals and counts   | Get Cash Inflows                 | Merge Data                        | 🧮 Calculates totals by category for both                                                    |
| Calculate Outflows  | Code                       | Calculates outflow totals and counts  | Get Cash Outflows                | Merge Data                        | 🧮 Calculates totals by category for both                                                    |
| Merge Data          | Merge                      | Combines inflow and outflow data      | Calculate Inflows, Calculate Outflows | Calculate Net Cash Flow         | 🔀 Merges the data together                                                                  |
| Calculate Net Cash Flow | Code                   | Computes net cash flow and status      | Merge Data                      | Save to Google Sheets, Generate HTML Report | 📊 Calculates net cash flow (Inflow - Outflow)                                              |
| Save to Google Sheets | Google Sheets             | Saves daily summary to spreadsheet    | Calculate Net Cash Flow          | Convert to PDF                    | 💾 Saves to Google Sheets for tracking                                                      |
| Generate HTML Report | Code                      | Creates HTML formatted report          | Calculate Net Cash Flow          | Post to Slack                    | 📄 Generates formatted HTML report                                                          |
| Convert to PDF      | PDFMonkey                  | Converts report to professional PDF   | Save to Google Sheets            | Email Report                    | 📑 Converts to professional PDF                                                             |
| Email Report        | Email Send                 | Emails report with PDF attachment      | Convert to PDF                  | None                           | 📧 Emails to finance team with PDF attached                                                  |
| Post to Slack       | Slack                      | Posts summary message to Slack channel | Generate HTML Report           | Backup to Google Drive           | 💬 Posts summary to Slack                                                                    |
| Backup to Google Drive | Google Drive             | Archives report to Google Drive       | Post to Slack                  | None                           | ☁️ Backs up to Google Drive                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure: Set to trigger daily at 18:00 (6 PM).  
   - Name it "Daily at 6 PM".

2. **Create HTTP Request node for inflows:**  
   - Type: HTTP Request  
   - Name: "Get Cash Inflows"  
   - Configure URL: `https://api.accounting.com/transactions` (add query parameters to filter today's incoming transactions as needed).  
   - Connect input from "Daily at 6 PM".

3. **Create HTTP Request node for outflows:**  
   - Type: HTTP Request  
   - Name: "Get Cash Outflows"  
   - Configure URL: same as inflows or appropriate endpoint, filter for today's outgoing transactions.  
   - Connect input from "Daily at 6 PM".

4. **Create Code node to calculate inflows:**  
   - Type: Code  
   - Name: "Calculate Inflows"  
   - Paste JavaScript code that sums amounts, groups by category, counts transactions, and adds date (see 2.3 for logic).  
   - Connect input from "Get Cash Inflows".

5. **Create Code node to calculate outflows:**  
   - Type: Code  
   - Name: "Calculate Outflows"  
   - Similar to inflows node, but for outflows data.  
   - Connect input from "Get Cash Outflows".

6. **Create Merge node:**  
   - Type: Merge  
   - Name: "Merge Data"  
   - Set mode to "Combine" with combination mode "Merge By Position".  
   - Connect inputs from "Calculate Inflows" (first input) and "Calculate Outflows" (second input).

7. **Create Code node to calculate net cash flow:**  
   - Type: Code  
   - Name: "Calculate Net Cash Flow"  
   - Paste JavaScript to subtract outflows from inflows, calculate status, and assemble full summary.  
   - Connect input from "Merge Data".

8. **Create Google Sheets node:**  
   - Type: Google Sheets  
   - Name: "Save to Google Sheets"  
   - Configure:  
     - Operation: Append  
     - Sheet Name: "Daily_Cash_Flow"  
     - Document ID: Use your Google Sheets document ID.  
     - Authentication: Service Account credentials or OAuth2 with write permission.  
     - Map columns to data fields: Date, Total_Inflow, Total_Outflow, Net_Cash_Flow, Status, Inflow_Count, Outflow_Count.  
   - Connect input from "Calculate Net Cash Flow".

9. **Create Code node to generate HTML report:**  
   - Type: Code  
   - Name: "Generate HTML Report"  
   - Paste JavaScript code that produces a styled HTML document with summary and tables for inflows and outflows by category.  
   - Connect input from "Calculate Net Cash Flow".

10. **Create PDFMonkey node:**  
    - Type: PDFMonkey  
    - Name: "Convert to PDF"  
    - Configure your PDFMonkey API credentials.  
    - Set the Document Template ID referencing your PDF template.  
    - Connect input from "Save to Google Sheets" (or from the node that outputs HTML if you send HTML there).  
    - Map data as needed to template variables.

11. **Create Email Send node:**  
    - Type: Email Send  
    - Name: "Email Report"  
    - Configure SMTP credentials.  
    - Set To: finance@company.com, cfo@company.com  
    - CC: accounting@company.com  
    - From: reports@company.com  
    - Subject: "Daily Cash Flow Report - {{ $now.format('MMM dd, yyyy') }}"  
    - Attach PDF: Use base64 content from "Convert to PDF" node for attachment.  
    - Connect input from "Convert to PDF".

12. **Create Slack node:**  
    - Type: Slack  
    - Name: "Post to Slack"  
    - Configure Slack API credentials.  
    - Set channel to #daily-reports (use channel ID if preferred).  
    - Compose message with date, total inflows, outflows, net cash flow, and status using expressions.  
    - Connect input from "Generate HTML Report".

13. **Create Google Drive node:**  
    - Type: Google Drive  
    - Name: "Backup to Google Drive"  
    - Configure OAuth2 credentials with access to your Google Drive.  
    - Set folder to `/finance/reports/` or appropriate folder ID.  
    - Configure to upload report files (PDF or raw data) as backup.  
    - Connect input from "Post to Slack".

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                           |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Workflow triggers at 6 PM daily to ensure end-of-day financial data capture and reporting.        | Scheduling best practice for daily finance reports.      |
| Uses service account authentication for Google Sheets to enable automated writes without user interaction. | Google Sheets API docs: https://developers.google.com/sheets/api |
| Slack notifications keep finance teams instantly updated with key cash flow metrics.              | Slack API docs: https://api.slack.com/messaging          |
| PDFMonkey integration creates professional PDF reports based on custom templates.                 | PDFMonkey: https://pdfmonkey.io/                          |
| Email notifications target finance, CFO, and accounting teams to ensure visibility and action.    | SMTP setup and email best practices.                      |
| Google Drive backup ensures archival and disaster recovery for financial reports.                 | Google Drive API docs: https://developers.google.com/drive/api |
| Consider adding error handling and retry mechanisms for API calls to improve resilience.          | n8n docs on error workflows: https://docs.n8n.io/nodes/   |

---

**Disclaimer:** The above documentation is derived exclusively from an n8n automated workflow JSON export. The workflow respects all content policies and only processes lawful and public financial data. No illegal or offensive content is included.