PDF Report Monitor with GPT-3.5 Insights and Slack/Email Alerts

https://n8nworkflows.xyz/workflows/pdf-report-monitor-with-gpt-3-5-insights-and-slack-email-alerts-7361


# PDF Report Monitor with GPT-3.5 Insights and Slack/Email Alerts

### 1. Workflow Overview

This workflow automates the monitoring of incoming PDF reports from an FTP folder, extracting key insights using GPT-3.5, and sending alerts via Slack and Email. It is designed for organizations that receive periodic PDF reports and want to automate alerting based on the report content, including identifying critical findings and sentiments.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger and Input Retrieval:** Periodically checks an FTP folder for new PDF reports.
- **1.2 PDF Filtering and Parsing:** Filters incoming files to PDFs only and parses their content using the PDF Vector node.
- **1.3 AI-Based Insight Extraction:** Uses OpenAI GPT-3.5 to extract structured insights from the parsed PDF content.
- **1.4 Alert Formatting and Distribution:** Formats the extracted insights into Slack message blocks and email content, then sends alerts accordingly.
- **1.5 Logging Processed Reports:** Records processed reports and their metadata into a Postgres database for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Input Retrieval

- **Overview:**  
  This block initiates the workflow every 15 minutes and retrieves the list of files from the FTP folder `/reports/incoming`.

- **Nodes Involved:**  
  - Check Every 15 Minutes  
  - Check Report Folder  
  - Filter New PDFs

- **Node Details:**

  - **Check Every 15 Minutes**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow every 15 minutes at the 15th minute of the hour (e.g., 00:15, 01:15).  
    - Configuration: Polls every minute with a minute offset of 15.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Check Report Folder".  
    - Edge Cases: Trigger timing issues if n8n instance is down; possible missed triggers.  

  - **Check Report Folder**  
    - Type: FTP Node  
    - Role: Lists files in `/reports/incoming` folder on FTP server.  
    - Configuration: Operation set to "list" on the specified folder. Credentials must be configured for FTP access.  
    - Inputs: Trigger from schedule.  
    - Outputs: File list passed to "Filter New PDFs".  
    - Edge Cases: FTP connection failures, authentication errors, folder not found, empty folder.  

  - **Filter New PDFs**  
    - Type: IF Node  
    - Role: Filters files to only those with `.pdf` extension.  
    - Configuration: Condition checks if filename ends with `.pdf`.  
    - Inputs: List of files from FTP node.  
    - Outputs: Only PDF files forwarded to parsing.  
    - Edge Cases: Files without extensions, case sensitivity issues (e.g., `.PDF` vs `.pdf`), empty input list.  

#### 2.2 PDF Filtering and Parsing

- **Overview:**  
  Parses the filtered PDF reports using PDF Vector node to extract the document content in a structured way.

- **Nodes Involved:**  
  - PDF Vector - Parse Report

- **Node Details:**

  - **PDF Vector - Parse Report**  
    - Type: PDF Vector Node  
    - Role: Parses PDF document content to extract raw text and semantic vectors.  
    - Configuration:  
      - Operation: Parse  
      - Resource: Document  
      - Document URL: Dynamically set from the JSON property containing the file URL (`{{$json.url}}`).  
      - LLM usage: Auto (automatic decision on whether to use language model assistance).  
    - Inputs: PDF files filtered by extension.  
    - Outputs: Parsed content passed to AI insight extraction.  
    - Edge Cases: Invalid or inaccessible document URL, parsing failures, unsupported PDF formats, large files causing timeouts.  

#### 2.3 AI-Based Insight Extraction

- **Overview:**  
  Sends the parsed PDF content to OpenAI GPT-3.5 to extract key structured insights such as report type, date, KPIs, findings, action items, and sentiment.

- **Nodes Involved:**  
  - Extract Key Insights

- **Node Details:**

  - **Extract Key Insights**  
    - Type: OpenAI Node  
    - Role: Uses GPT-3.5 Turbo model to analyze extracted PDF content and return structured JSON insights.  
    - Configuration:  
      - Model: `gpt-3.5-turbo`  
      - Message content includes a prompt requesting extraction of report metadata and key information formatted as JSON.  
      - Response format explicitly set to JSON object.  
    - Inputs: Parsed document content.  
    - Outputs: JSON structured insights passed to alert formatting.  
    - Credentials: Requires valid OpenAI API credentials.  
    - Edge Cases: API rate limits, authentication errors, malformed JSON response, timeouts, incomplete or ambiguous input content.  

#### 2.4 Alert Formatting and Distribution

- **Overview:**  
  Formats extracted insights into Slack message blocks and HTML email content, then sends alerts to Slack channel and email recipients. Also determines alert priority.

- **Nodes Involved:**  
  - Format Alerts  
  - Send Slack Alert  
  - Send Email Alert

- **Node Details:**

  - **Format Alerts**  
    - Type: Code Node (JavaScript)  
    - Role: Parses AI insights JSON, determines alert priority based on sentiment and alerts presence, and builds Slack message blocks and email content.  
    - Configuration:  
      - Parses JSON from AI node output.  
      - Priority set to "high" if sentiment is negative or alerts found; else "normal".  
      - Constructs Slack header, sections for date, priority, metrics, findings.  
      - Prepares email HTML content with report details, metrics, findings, and action items.  
      - Outputs multiple JSON fields used downstream.  
    - Inputs: AI insights JSON.  
    - Outputs: Formatted alert data to Slack and Email nodes.  
    - Edge Cases: Malformed AI JSON, missing fields, empty arrays, code execution errors.  

  - **Send Slack Alert**  
    - Type: Slack Node  
    - Role: Sends formatted Slack alert message to a designated channel.  
    - Configuration:  
      - Channel: `#reports`  
      - Message blocks populated from code node output.  
      - Attachments empty.  
    - Inputs: Formatted Slack message blocks.  
    - Outputs: None (end node for alert).  
    - Credentials: Slack OAuth credentials required.  
    - Edge Cases: Slack API rate limits, invalid channel, connection issues, message format errors.  

  - **Send Email Alert**  
    - Type: Email Send Node  
    - Role: Sends email alert with formatted report details.  
    - Configuration:  
      - Recipient: `team@company.com`  
      - Subject includes report type dynamically.  
      - HTML body built from code node output including metrics, findings, action items.  
    - Inputs: Formatted email content.  
    - Outputs: None (end node for alert).  
    - Credentials: Email SMTP or service credentials required.  
    - Edge Cases: SMTP authentication issues, invalid recipient address, email formatting errors.  

#### 2.5 Logging Processed Reports

- **Overview:**  
  Logs metadata and insights of processed reports into a Postgres database table for record-keeping and audit.

- **Nodes Involved:**  
  - Log Processed Report

- **Node Details:**

  - **Log Processed Report**  
    - Type: Postgres Node  
    - Role: Inserts a record of the processed report with filename, type, processing timestamp, insights JSON, and priority.  
    - Configuration:  
      - Operation: Insert  
      - Table: `processed_reports`  
      - Columns: `filename, report_type, processed_at, insights, priority`  
      - Values extracted from previous node outputs.  
    - Inputs: Formatted alert data.  
    - Outputs: None (terminal logging node).  
    - Credentials: Requires valid Postgres DB credentials.  
    - Edge Cases: Database connection failures, constraint violations, data type mismatches, large JSON payloads.  

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                       | Input Node(s)       | Output Node(s)                  | Sticky Note                                                                                                    |
|-----------------------|--------------------|------------------------------------|---------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------|
| Alert System Info     | Sticky Note        | Provides workflow overview          | None                | None                          | ## Document Alert System  Monitors: - FTP folders - Email attachments - Cloud storage - Direct URLs Sends alerts via Slack & Email |
| Check Every 15 Minutes| Schedule Trigger   | Triggers workflow every 15 minutes | None                | Check Report Folder           |                                                                                                               |
| Check Report Folder   | FTP                | Lists files in FTP reports folder   | Check Every 15 Minutes| Filter New PDFs               |                                                                                                               |
| Filter New PDFs       | IF                 | Filters only PDF files              | Check Report Folder  | PDF Vector - Parse Report     |                                                                                                               |
| PDF Vector - Parse Report | PDF Vector       | Parses PDF content                  | Filter New PDFs      | Extract Key Insights          |                                                                                                               |
| Extract Key Insights  | OpenAI             | Extracts structured insights via AI| PDF Vector - Parse Report | Format Alerts             |                                                                                                               |
| Format Alerts         | Code               | Formats alerts for Slack/email     | Extract Key Insights | Send Slack Alert, Send Email Alert, Log Processed Report |                                                                                                               |
| Send Slack Alert      | Slack              | Sends alert message to Slack channel| Format Alerts       | None                         |                                                                                                               |
| Send Email Alert      | Email Send         | Sends alert email to team          | Format Alerts        | None                         |                                                                                                               |
| Log Processed Report  | Postgres           | Logs processed report metadata     | Format Alerts        | None                         |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note:**  
   - Name: `Alert System Info`  
   - Content:  
     ```
     ## Document Alert System

     Monitors:
     - FTP folders
     - Email attachments
     - Cloud storage
     - Direct URLs

     Sends alerts via Slack & Email
     ```  
   - Position it at the top-left area for documentation.

2. **Add Schedule Trigger:**  
   - Node Type: `Schedule Trigger`  
   - Name: `Check Every 15 Minutes`  
   - Configure to trigger every hour at minute 15 (e.g., 00:15, 01:15).  
   - No credentials needed.

3. **Add FTP Node:**  
   - Node Type: `FTP`  
   - Name: `Check Report Folder`  
   - Operation: `List`  
   - Folder: `/reports/incoming`  
   - Configure FTP credentials with access to the server and folder.  
   - Connect output from `Check Every 15 Minutes` to this node.

4. **Add IF Node:**  
   - Node Type: `IF`  
   - Name: `Filter New PDFs`  
   - Condition: Check if filename ends with `.pdf` (case-sensitive).  
     - Expression: `{{$json["name"].endsWith('.pdf')}} == true`  
   - Connect output from `Check Report Folder` to this node.

5. **Add PDF Vector Node:**  
   - Node Type: `PDF Vector` (requires PDF Vector node installed)  
   - Name: `PDF Vector - Parse Report`  
   - Operation: `Parse`  
   - Resource: `Document`  
   - Document URL: Use expression `{{$json.url}}` or equivalent property containing the PDF URL.  
   - Set LLM usage to `auto`.  
   - Connect output from `Filter New PDFs` (true path) to this node.

6. **Add OpenAI Node:**  
   - Node Type: `OpenAI`  
   - Name: `Extract Key Insights`  
   - Model: `gpt-3.5-turbo`  
   - Message:  
     ```
     Extract key information from this report:

     {{$json.content}}

     Identify:
     1. Report type and date
     2. Key metrics or KPIs
     3. Important findings or alerts
     4. Action items
     5. Overall sentiment (positive/neutral/negative)

     Return as structured JSON.
     ```  
   - Response format: JSON object  
   - Configure OpenAI credentials.  
   - Connect output from `PDF Vector - Parse Report` to this node.

7. **Add Code Node:**  
   - Node Type: `Code` (JavaScript)  
   - Name: `Format Alerts`  
   - Paste the following code (adapted for context):
     ```javascript
     const insights = JSON.parse($json.content);
     const fileName = $node['Check Report Folder'].json.name;

     let priority = 'normal';
     if (insights.sentiment === 'negative' || (insights.alerts && insights.alerts.length > 0)) {
       priority = 'high';
     }

     const slackBlocks = [
       {
         type: 'header',
         text: {
           type: 'plain_text',
           text: `📄 New Report: ${insights.reportType || fileName}`
         }
       },
       {
         type: 'section',
         fields: [
           {
             type: 'mrkdwn',
             text: `*Date:* ${insights.date || 'Not specified'}`
           },
           {
             type: 'mrkdwn',
             text: `*Priority:* ${priority}`
           }
         ]
       }
     ];

     if (insights.metrics) {
       slackBlocks.push({
         type: 'section',
         text: {
           type: 'mrkdwn',
           text: `*Key Metrics:*\n${Object.entries(insights.metrics).map(([k,v]) => `• ${k}: ${v}`).join('\n')}`
         }
       });
     }

     if (insights.findings) {
       slackBlocks.push({
         type: 'section',
         text: {
           type: 'mrkdwn',
           text: `*Key Findings:*\n${insights.findings.map(f => `• ${f}`).join('\n')}`
         }
       });
     }

     return {
       fileName,
       insights,
       priority,
       slackBlocks,
       emailContent: insights
     };
     ```
   - Connect output from `Extract Key Insights` to this node.

8. **Add Slack Node:**  
   - Node Type: `Slack`  
   - Name: `Send Slack Alert`  
   - Channel: `#reports` (or your desired Slack channel)  
   - Message blocks: Use expression `{{$json.slackBlocks}}`  
   - Configure Slack OAuth credentials.  
   - Connect output from `Format Alerts` to this node.

9. **Add Email Send Node:**  
   - Node Type: `Email Send`  
   - Name: `Send Email Alert`  
   - To Email: `team@company.com` (customize as needed)  
   - Subject: `New Report Alert: {{$json.insights.reportType}}`  
   - HTML Body (use this template):
     ```html
     <h2>New Report Available</h2>
     <p><strong>Report:</strong> {{ $json.fileName }}</p>
     <p><strong>Type:</strong> {{ $json.insights.reportType }}</p>
     <p><strong>Date:</strong> {{ $json.insights.date }}</p>

     <h3>Key Metrics</h3>
     <ul>
     {{ Object.entries($json.insights.metrics || {}).map(([k,v]) => `<li>${k}: ${v}</li>`).join('\n') }}
     </ul>

     <h3>Findings</h3>
     <ul>
     {{ ($json.insights.findings || []).map(f => `<li>${f}</li>`).join('\n') }}
     </ul>

     <h3>Action Items</h3>
     <ul>
     {{ ($json.insights.actionItems || []).map(a => `<li>${a}</li>`).join('\n') }}
     </ul>
     ```
   - Configure SMTP or email service credentials.  
   - Connect output from `Format Alerts` to this node.

10. **Add Postgres Node:**  
    - Node Type: `Postgres`  
    - Name: `Log Processed Report`  
    - Operation: `Insert`  
    - Table: `processed_reports`  
    - Columns: `filename, report_type, processed_at, insights, priority`  
    - Map columns to corresponding values from `$json.fileName`, `$json.insights.reportType`, current timestamp (use expression like `{{ $now }}`), JSON stringified insights, and priority.  
    - Configure Postgres DB credentials.  
    - Connect output from `Format Alerts` to this node.

11. **Connect nodes as follows:**  
    - `Check Every 15 Minutes` → `Check Report Folder` → `Filter New PDFs` → `PDF Vector - Parse Report` → `Extract Key Insights` → `Format Alerts` → (`Send Slack Alert`, `Send Email Alert`, `Log Processed Report`)

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Slack message formatting follows Slack Block Kit standards for rich alert messages.           | [Slack Block Kit](https://api.slack.com/block-kit)             |
| PDF Vector node requires proper configuration and access to PDF URLs or files.                | PDF Vector documentation in n8n community                      |
| OpenAI API requires valid credentials and may incur costs based on usage.                     | [OpenAI Pricing](https://openai.com/pricing)                   |
| Email node requires SMTP or 3rd party service credentials (e.g., Gmail, SendGrid).             | n8n Email node docs                                            |
| Postgres table `processed_reports` should be pre-created with appropriate schema.             | Example schema: `filename TEXT, report_type TEXT, processed_at TIMESTAMP, insights JSONB, priority TEXT` |

---

This document describes the full structure, logic, and configuration of the "PDF Report Monitor with GPT-3.5 Insights and Slack/Email Alerts" workflow, enabling thorough understanding, reproduction, and maintenance.