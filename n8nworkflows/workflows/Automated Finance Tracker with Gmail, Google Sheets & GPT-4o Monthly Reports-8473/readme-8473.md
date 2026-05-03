Automated Finance Tracker with Gmail, Google Sheets & GPT-4o Monthly Reports

https://n8nworkflows.xyz/workflows/automated-finance-tracker-with-gmail--google-sheets---gpt-4o-monthly-reports-8473


# Automated Finance Tracker with Gmail, Google Sheets & GPT-4o Monthly Reports

### 1. Workflow Overview

This workflow automates personal or small business finance tracking by integrating Gmail, Google Sheets, and OpenAI’s GPT-4o. It performs the complete cycle of fetching receipt emails, extracting structured receipt data using AI, avoiding duplicates, appending new entries to a finance sheet, and generating insightful monthly financial reports with charts and summaries emailed automatically.

The workflow logically divides into two main functional blocks:

- **1.1 Receipt Processing Block**: Fetches receipt emails, extracts and cleans receipt data using AI, checks for duplicates, filters incomplete data, and appends validated entries to a Google Sheet.

- **1.2 Monthly Reporting Block**: Triggered monthly, it reads all financial data from the sheet, aggregates it by month, generates AI-driven insights, produces a visual summary chart, and emails the final report.

This design enables continuous automated financial data capture with no manual input, combined with AI-powered analysis and periodic reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Receipt Processing Block

**Overview:**  
Processes incoming receipt emails by fetching them, extracting attachments and email content, using GPT-4o to parse receipt details, cleaning and validating the data, checking for duplicates, filtering out incomplete entries, and finally appending new, validated data to a Google Sheet.

**Nodes Involved:**  
- Fetch Receipt Emails (Gmail)  
- Get Email with Attachments (Gmail)  
- Parse Email Body & Check Attachments (Function)  
- AI: Extract Receipt Data (GPT-4o) (OpenAI)  
- Clean & Parse AI Output (Function)  
- Check for Duplicates (Google Sheets)  
- Filter: Skip if Missing Vendor (If)  
- Append to Finance Sheet (Google Sheets)

**Node Details:**

- **Fetch Receipt Emails**  
  - Type: Gmail node (IMAP fetch)  
  - Configuration: Retrieves recent emails tagged or identified as receipts (e.g., via search criteria).  
  - Input: Trigger from the Monthly Report Trigger node or manual start.  
  - Output: Email metadata and list of emails.  
  - Potential failures: Authentication errors, rate limits, no emails found.

- **Get Email with Attachments**  
  - Type: Gmail node  
  - Configuration: Fetches full email content including attachments for each email fetched.  
  - Input: Output of Fetch Receipt Emails.  
  - Output: Emails enriched with attachments.  
  - Edge cases: Emails without attachments, large attachments causing timeouts.

- **Parse Email Body & Check Attachments**  
  - Type: Function node (JavaScript)  
  - Configuration: Parses the email body text and verifies presence and validity of attachments expected as receipts.  
  - Input: Gmail email data with attachments.  
  - Output: Structured data object containing the parsed receipt text and attachment metadata.  
  - Failure modes: Parsing errors if email format is unexpected, missing attachments flagged here.

- **AI: Extract Receipt Data (GPT-4o)**  
  - Type: OpenAI node  
  - Configuration: Sends parsed receipt text to GPT-4o with a prompt designed to extract structured receipt fields (date, vendor, total, items, tax, etc.).  
  - Input: Parsed email body data.  
  - Output: AI-generated JSON string with receipt details.  
  - Edge cases: AI response delay or timeout; incomplete or ambiguous data extraction.

- **Clean & Parse AI Output**  
  - Type: Function node  
  - Configuration: Cleans the AI output string, parses JSON safely, normalizes date formats, and ensures data consistency.  
  - Input: AI raw output.  
  - Output: Clean structured receipt data object.  
  - Failure: JSON parse errors; malformed AI output requiring fallback logic.

- **Check for Duplicates**  
  - Type: Google Sheets node (read/search)  
  - Configuration: Searches the finance sheet for existing entries matching the new receipt's unique identifiers (e.g., invoice number, date, amount).  
  - Input: Clean receipt data.  
  - Output: Boolean or filtered data indicating duplication.  
  - Failure: Sheet access errors, large data causing slow queries.

- **Filter: Skip if Missing Vendor**  
  - Type: If node  
  - Configuration: Checks if the receipt data contains a vendor name; if missing, skips appending to prevent incomplete records.  
  - Input: Duplicate check output.  
  - Output: Routes to append or discard path.  
  - Edge case: Vendor field missing or null; false negatives if vendor parsing failed.

- **Append to Finance Sheet**  
  - Type: Google Sheets node (append)  
  - Configuration: Appends new validated receipt data as a new row in the finance sheet.  
  - Input: Filtered, non-duplicate, complete receipt data.  
  - Output: Confirmation of append success.  
  - Failure: Write permission errors, sheet quota limits.

---

#### 2.2 Monthly Reporting Block

**Overview:**  
Triggered monthly, this block generates a period range, reads all financial data, aggregates it by month, generates AI-driven insights, creates charts, and emails a comprehensive monthly report.

**Nodes Involved:**  
- Monthly Report Trigger (Cron)  
- Generate Month Range (Function)  
- Read All Finance Data (Google Sheets)  
- Aggregate Monthly Data (Function)  
- AI: Generate Insights (OpenAI)  
- Generate Chart & Final Data (Function)  
- Send Monthly Report (Email Send)

**Node Details:**

- **Monthly Report Trigger**  
  - Type: Cron node  
  - Configuration: Scheduled to run once per month at a specified time (default midnight on 1st of month).  
  - Input: None (time-based trigger).  
  - Output: Triggers both receipt fetching and reporting chains.  
  - Edge cases: Missed trigger due to downtime; time zone considerations.

- **Generate Month Range**  
  - Type: Function node  
  - Configuration: Generates an array or object defining the date range for the report period (e.g., previous calendar month).  
  - Input: Trigger from cron.  
  - Output: Date range parameters for data queries.  
  - Edge cases: Incorrect date calculations around year boundaries.

- **Read All Finance Data**  
  - Type: Google Sheets node (read)  
  - Configuration: Reads all rows from the finance sheet covering the date range generated.  
  - Input: Month range parameters.  
  - Output: Raw finance data array.  
  - Failure: Large data volume causing timeout, sheet access issues.

- **Aggregate Monthly Data**  
  - Type: Function node  
  - Configuration: Processes raw data to aggregate totals, categorize expenses/income, and prepare summary statistics.  
  - Input: Raw finance data.  
  - Output: Aggregated monthly summary object.  
  - Edge cases: Missing or malformed data rows.

- **AI: Generate Insights**  
  - Type: OpenAI node  
  - Configuration: Sends aggregated monthly data along with a prompt to generate narrative insights, trends, and financial advice.  
  - Input: Aggregated data summary.  
  - Output: AI-generated text insights.  
  - Edge cases: AI response latency or incomplete analysis.

- **Generate Chart & Final Data**  
  - Type: Function node  
  - Configuration: Creates visualization data (e.g., base64 encoded charts), compiles final report data structure combining AI insights and chart.  
  - Input: AI insights and aggregated data.  
  - Output: Final report package ready for email.  
  - Failure: Chart generation errors or encoding issues.

- **Send Monthly Report**  
  - Type: Email Send node (SMTP or Gmail)  
  - Configuration: Sends the monthly report email with chart and insights to the configured recipient(s).  
  - Input: Final report content.  
  - Output: Email delivery confirmation.  
  - Edge cases: SMTP auth errors, email delivery failures, spam filtering.

---

### 3. Summary Table

| Node Name                     | Node Type         | Functional Role                      | Input Node(s)             | Output Node(s)                 | Sticky Note                      |
|-------------------------------|-------------------|-----------------------------------|---------------------------|-------------------------------|---------------------------------|
| 📋 Template Description        | Sticky Note       | Documentation placeholder          |                           |                               |                                 |
| Monthly Report Trigger         | Cron              | Monthly trigger for report & fetch |                           | Generate Month Range, Fetch Receipt Emails |                                 |
| User Config                   | Set               | User-specific configuration values |                           |                               |                                 |
| Fetch Receipt Emails          | Gmail             | Fetch receipt-related emails       | Monthly Report Trigger    | Get Email with Attachments     |                                 |
| Get Email with Attachments    | Gmail             | Retrieve email contents & attachments | Fetch Receipt Emails      | Parse Email Body & Check Attachments |                                 |
| Parse Email Body & Check Attachments | Function          | Parse email content & validate attachments | Get Email with Attachments | AI: Extract Receipt Data (GPT-4o) |                                 |
| AI: Extract Receipt Data (GPT-4o) | OpenAI            | Extract structured receipt data    | Parse Email Body & Check Attachments | Clean & Parse AI Output        |                                 |
| Clean & Parse AI Output       | Function          | Clean and parse AI JSON output     | AI: Extract Receipt Data (GPT-4o) | Check for Duplicates           |                                 |
| Check for Duplicates          | Google Sheets     | Check if receipt entry already exists | Clean & Parse AI Output    | Filter: Skip if Missing Vendor |                                 |
| Filter: Skip if Missing Vendor | If                | Skip entries missing vendor name   | Check for Duplicates      | Append to Finance Sheet         |                                 |
| Append to Finance Sheet       | Google Sheets     | Append new receipt data to sheet   | Filter: Skip if Missing Vendor |                               |                                 |
| Generate Month Range          | Function          | Create date range for report       | Monthly Report Trigger    | Read All Finance Data           |                                 |
| Read All Finance Data         | Google Sheets     | Fetch all finance data for period  | Generate Month Range      | Aggregate Monthly Data          |                                 |
| Aggregate Monthly Data        | Function          | Summarize and aggregate data       | Read All Finance Data     | AI: Generate Insights           |                                 |
| AI: Generate Insights         | OpenAI            | Generate narrative financial insights | Aggregate Monthly Data    | Generate Chart & Final Data     |                                 |
| Generate Chart & Final Data   | Function          | Create charts and finalize report  | AI: Generate Insights     | Send Monthly Report             |                                 |
| Send Monthly Report           | Email Send        | Email the monthly report            | Generate Chart & Final Data |                               |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Node named "Monthly Report Trigger"**  
   - Set to run monthly (e.g., at 00:00 on the 1st)  
   - This node triggers the entire workflow monthly.

2. **Create a Function Node named "Generate Month Range"**  
   - Connect from "Monthly Report Trigger"  
   - Code: Generate previous calendar month start and end dates in ISO format.  
   - Outputs the date range for filtering finance data.

3. **Create a Google Sheets Read Node named "Read All Finance Data"**  
   - Connect from "Generate Month Range"  
   - Configure credentials for Google Sheets OAuth2.  
   - Set to read all rows from the finance sheet covering the generated month range.

4. **Create a Function Node named "Aggregate Monthly Data"**  
   - Connect from "Read All Finance Data"  
   - Code: Aggregate data by categories, totals, count transactions.

5. **Create an OpenAI Node named "AI: Generate Insights"**  
   - Connect from "Aggregate Monthly Data"  
   - Use OpenAI API credentials.  
   - Prompt: Provide aggregated data, ask for insights and financial summary text.

6. **Create a Function Node named "Generate Chart & Final Data"**  
   - Connect from "AI: Generate Insights"  
   - Code: Generate a base64 chart (e.g., bar chart of expenses), combine with AI insights.

7. **Create an Email Send Node named "Send Monthly Report"**  
   - Connect from "Generate Chart & Final Data"  
   - Configure SMTP or Gmail OAuth2 credentials.  
   - Set email recipient(s), subject, and body including chart and insights.

8. **Create a Gmail Node named "Fetch Receipt Emails"**  
   - Connect also from "Monthly Report Trigger" (parallel branch)  
   - Configure Gmail OAuth2 credentials.  
   - Set to fetch emails matching receipt criteria (e.g., subject contains "receipt").

9. **Create a Gmail Node named "Get Email with Attachments"**  
   - Connect from "Fetch Receipt Emails"  
   - Retrieves full email content and attachments.

10. **Create a Function Node named "Parse Email Body & Check Attachments"**  
    - Connect from "Get Email with Attachments"  
    - Code: Extract text from email body, verify receipt attachments exist.

11. **Create an OpenAI Node named "AI: Extract Receipt Data (GPT-4o)"**  
    - Connect from "Parse Email Body & Check Attachments"  
    - Use OpenAI credentials.  
    - Prompt: Extract structured receipt info (vendor, date, totals, items).

12. **Create a Function Node named "Clean & Parse AI Output"**  
    - Connect from "AI: Extract Receipt Data (GPT-4o)"  
    - Code: Parse JSON response, normalize data fields.

13. **Create a Google Sheets Read Node named "Check for Duplicates"**  
    - Connect from "Clean & Parse AI Output"  
    - Configure to search finance sheet for existing matching entries by key fields.

14. **Create an If Node named "Filter: Skip if Missing Vendor"**  
    - Connect from "Check for Duplicates"  
    - Condition: Check if vendor field exists and is not empty.

15. **Create a Google Sheets Append Node named "Append to Finance Sheet"**  
    - Connect from the "true" output of the If node (vendor present)  
    - Append the new receipt data row to the finance sheet.

16. **Ensure all credentials are configured:**  
    - Gmail OAuth2 for Gmail nodes  
    - Google Sheets OAuth2 for Sheets nodes  
    - OpenAI API key for OpenAI nodes  
    - SMTP or Gmail OAuth2 for Email Send node

17. **Test the workflow end-to-end:**  
    - Trigger manually or wait for the scheduled cron to verify email fetching, receipt processing, and monthly report generation.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow requires OAuth2 credentials for Gmail and Google Sheets APIs configured in n8n.   | n8n Credential setup documentation: https://docs.n8n.io/credentials/google/                         |
| OpenAI GPT-4o is used for advanced AI extraction and insight generation, requiring API key setup. | OpenAI API docs: https://platform.openai.com/docs/api-reference                                     |
| SMTP or Gmail OAuth2 must be configured for sending emails reliably.                             | Email node docs: https://docs.n8n.io/nodes/n8n-nodes-base.email-send/                               |
| The workflow assumes receipts are emailed and contain attachments or text parsable by AI.       | Adjust Gmail fetch filters as needed for your receipt source.                                      |
| Chart generation is done via JavaScript function nodes; customize for preferred chart libraries. | Consider using libraries like Chart.js in functions or external services for richer charts.        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.