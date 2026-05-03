Sync KPI Metrics from ClickUp and Google Sheets to Slack and Gmail

https://n8nworkflows.xyz/workflows/sync-kpi-metrics-from-clickup-and-google-sheets-to-slack-and-gmail-9464


# Sync KPI Metrics from ClickUp and Google Sheets to Slack and Gmail

### 1. Workflow Overview

This workflow automates the collection, computation, and distribution of KPI metrics sourced from ClickUp task data and Google Sheets lead data. It targets business teams and managers who want to monitor task progress and lead generation performance daily, summarized in Slack and Gmail reports.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Executes the workflow daily at a configured time.
- **1.2 Data Fetching:** Retrieves the latest task data from ClickUp and lead data from Google Sheets concurrently.
- **1.3 Data Merging:** Consolidates task and lead datasets into a single unified dataset.
- **1.4 KPI Computation:** Processes the merged data to calculate task and lead KPIs, including sentiment analysis and trend indicators.
- **1.5 Data Formatting:** Structures computed KPI results into variables formatted for Slack and email reports.
- **1.6 Multi-Channel Distribution:** Posts a KPI snapshot to Slack and sends a detailed KPI report via Gmail.
- **1.7 Error Handling:** Monitors workflow failures and sends alert notifications to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Initiates the workflow automatically on a daily schedule.
- **Nodes Involved:**  
  - Daily Cron Trigger  
  - Cron Trigger Note (sticky note)
- **Node Details:**

  - **Daily Cron Trigger**  
    - Type: Cron trigger node  
    - Configuration: Default daily schedule (time and timezone to be set by user)  
    - Inputs: None (trigger node)  
    - Outputs: Triggers parallel downstream nodes  
    - Edge Cases: Misconfigured timezones or disabled schedules will prevent execution  
    - Sticky Note: Describes purpose and configuration hints for scheduled execution

---

#### 2.2 Data Fetching

- **Overview:** Retrieves task metrics from ClickUp and lead generation data from Google Sheets concurrently.
- **Nodes Involved:**  
  - ClickUp - Fetch Tasks  
  - Google Sheets - Fetch Lead Data  
  - ClickUp Tasks Note (sticky note)  
  - Google Sheets Note (sticky note)
- **Node Details:**

  - **ClickUp - Fetch Tasks**  
    - Type: ClickUp node  
    - Role: Fetches task data from a specific list within a ClickUp workspace  
    - Config:  
      - Team ID, Space ID, Folder ID, List ID specified  
      - Limit of 5 tasks fetched (adjustable)  
      - Filters to exclude subtasks  
      - OAuth2 authentication required  
    - Inputs: Trigger output  
    - Outputs: Task items to merge node  
    - Edge Cases: API rate limits, auth token expiration, empty task list  
    - Sticky Note: Explains data fields fetched (status, assignees, time tracking, due dates)

  - **Google Sheets - Fetch Lead Data**  
    - Type: Google Sheets node  
    - Role: Reads lead data rows from a specified sheet and document  
    - Config:  
      - Document ID and Sheet ID set to specific KPI data sheet  
      - OAuth2 authentication required  
    - Inputs: Trigger output (runs in parallel to ClickUp node)  
    - Outputs: Lead data items to merge node  
    - Edge Cases: Access permission issues, empty or invalid sheet, API quota limits  
    - Sticky Note: Describes lead data fields retrieved (contact, reply, source, status)

---

#### 2.3 Data Merging

- **Overview:** Combines ClickUp tasks and Google Sheets leads into a single dataset for unified analysis.
- **Nodes Involved:**  
  - Merge - Consolidate Datasets  
  - Merge Data Note (sticky note)
- **Node Details:**

  - **Merge - Consolidate Datasets**  
    - Type: Merge node  
    - Role: Combines two input streams (tasks and leads) into one output stream  
    - Config: Defaults to merge mode (probably 'append')  
    - Inputs: Two inputs — from ClickUp and Google Sheets fetch nodes  
    - Outputs: Combined dataset to KPI computation node  
    - Edge Cases: Mismatched data structures or empty inputs  
    - Sticky Note: Explains the purpose of merging datasets

---

#### 2.4 KPI Computation

- **Overview:** Computes detailed KPI metrics and trends from the combined task and lead dataset.
- **Nodes Involved:**  
  - Code - Compute KPI Trends  
  - KPI Computation Note (sticky note)
- **Node Details:**

  - **Code - Compute KPI Trends**  
    - Type: Code (JavaScript) node  
    - Role: Processes JSON data to calculate:  
      - Task KPIs: status distribution, priority counts, assignee task counts, overdue tasks, average time spent, completion stats  
      - Lead KPIs: source counts, status counts, sentiment analysis (positive, negative, neutral), replies by date, top sources  
      - Overall KPIs: data quality metrics, counts, and trends (task and lead trends, urgent tasks, new replies)  
    - Inputs: Merged dataset from previous node  
    - Outputs: Single JSON object containing all KPI metrics  
    - Key Expressions: Uses array filtering, aggregation, sentiment keyword matching, date comparisons  
    - Edge Cases: Missing fields, unexpected data formats, empty inputs, date parsing errors  
    - Sticky Note: Advises customization of JavaScript code to add custom KPIs

---

#### 2.5 Data Formatting

- **Overview:** Maps KPI computation output into structured variables for report generation.
- **Nodes Involved:**  
  - Set - Format Output Data  
  - Data Formatter Note (sticky note)
- **Node Details:**

  - **Set - Format Output Data**  
    - Type: Set node  
    - Role: Extracts KPI metrics into named fields for downstream reporting nodes  
    - Config: Assigns multiple fields such as tasks, overdue counts, average times, leads, sentiment rates, trends, etc.  
    - Inputs: KPI JSON output from code node  
    - Outputs: Structured JSON for Slack and Gmail nodes  
    - Edge Cases: Missing keys or null values in input JSON  
    - Sticky Note: Highlights purpose of formatting for Slack and Gmail

---

#### 2.6 Multi-Channel Distribution

- **Overview:** Sends KPI summaries to Slack channel and detailed reports via Gmail.
- **Nodes Involved:**  
  - Slack - Post Dashboard Snapshot  
  - Gmail - Send KPI Report  
  - Slack Dashboard Note (sticky note)  
  - Gmail Report Note (sticky note)
- **Node Details:**

  - **Slack - Post Dashboard Snapshot**  
    - Type: Slack node  
    - Role: Posts a brief KPI summary message to a configured Slack channel  
    - Config:  
      - Channel ID set to specific channel (e.g. #general)  
      - Message text composed with expressions to display task and lead metrics, top sources, and overall status  
      - OAuth2 Slack API credentials configured  
    - Inputs: Formatted KPI data  
    - Outputs: None (end node)  
    - Edge Cases: Slack API rate limits, invalid channel ID, auth failure  
    - Sticky Note: Reminds to update channel ID in config

  - **Gmail - Send KPI Report**  
    - Type: Gmail node  
    - Role: Sends a detailed HTML email with KPI metrics, charts, and insights  
    - Config:  
      - Recipient email address parameterized or set via environment variable  
      - Rich HTML body with styled KPI report and dynamic data using expressions  
      - Subject line includes current date  
      - OAuth2 Gmail credentials configured  
    - Inputs: Formatted KPI data  
    - Outputs: None (end node)  
    - Edge Cases: Email send failure, invalid recipient, auth issues  
    - Sticky Note: Advises updating recipient email address in settings

---

#### 2.7 Error Handling

- **Overview:** Detects workflow execution errors and sends alerts to Slack for team notification.
- **Nodes Involved:**  
  - Error Trigger Handler  
  - Slack - Send Error Alert  
  - Error Handling Note (sticky note)
- **Node Details:**

  - **Error Trigger Handler**  
    - Type: Error Trigger node  
    - Role: Listens for failures anywhere in the workflow  
    - Inputs: Implicit (monitors workflow)  
    - Outputs: Triggers Slack alert node on error

  - **Slack - Send Error Alert**  
    - Type: Slack node  
    - Role: Sends formatted error alert message to a designated Slack alerts channel  
    - Config:  
      - Channel ID for alerts channel must be set  
      - Message includes error details, failed node name, execution ID, timestamp, and recommended actions  
      - OAuth2 Slack API credentials configured  
    - Inputs: Error data from error trigger  
    - Outputs: None (end node)  
    - Edge Cases: Slack API errors, incorrect channel ID, missing error info  
    - Sticky Note: Describes alert purpose and configuration hints

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                        | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                     |
|-------------------------------|----------------------|-------------------------------------|--------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow Overview             | Sticky Note          | Describes overall workflow           | -                              | -                                 | Contains detailed workflow overview, features, setup requirements, and execution flow                           |
| Cron Trigger Note             | Sticky Note          | Explains daily scheduled trigger    | -                              | -                                 | Explains daily trigger configuration                                                                            |
| ClickUp Tasks Note            | Sticky Note          | Describes ClickUp data fetch         | -                              | -                                 | Describes fetched task data fields and OAuth2 authentication                                                    |
| Google Sheets Note            | Sticky Note          | Describes Google Sheets data fetch   | -                              | -                                 | Describes lead data fields retrieved                                                                             |
| Merge Data Note               | Sticky Note          | Explains data merging step            | -                              | -                                 | Explains consolidation of datasets                                                                              |
| KPI Computation Note          | Sticky Note          | Explains KPI computation logic        | -                              | -                                 | Advises on KPI customization in JavaScript code                                                                |
| Data Formatter Note           | Sticky Note          | Explains data formatting before output| -                              | -                                 | Describes structuring KPI data for Slack and Gmail                                                             |
| Slack Dashboard Note          | Sticky Note          | Explains Slack posting node           | -                              | -                                 | Reminds to update Slack channel ID                                                                              |
| Gmail Report Note             | Sticky Note          | Explains Gmail reporting node         | -                              | -                                 | Reminds to update recipient email address                                                                       |
| Error Handling Note           | Sticky Note          | Explains error alerting mechanism     | -                              | -                                 | Describes error alerts and Slack channel configuration                                                         |
| Daily Cron Trigger            | Cron Trigger         | Starts workflow daily                 | -                              | Google Sheets - Fetch Lead Data, ClickUp - Fetch Tasks |                                                                                                                 |
| ClickUp - Fetch Tasks         | ClickUp              | Fetches ClickUp task data             | Daily Cron Trigger             | Merge - Consolidate Datasets       |                                                                                                                 |
| Google Sheets - Fetch Lead Data| Google Sheets        | Fetches lead data from Google Sheets | Daily Cron Trigger             | Merge - Consolidate Datasets       |                                                                                                                 |
| Merge - Consolidate Datasets  | Merge                | Merges ClickUp and Google Sheets data| ClickUp - Fetch Tasks, Google Sheets - Fetch Lead Data | Code - Compute KPI Trends          |                                                                                                                 |
| Code - Compute KPI Trends     | Code                 | Computes KPI metrics and trends       | Merge - Consolidate Datasets   | Set - Format Output Data           |                                                                                                                 |
| Set - Format Output Data      | Set                  | Formats KPI data for reports          | Code - Compute KPI Trends      | Slack - Post Dashboard Snapshot, Gmail - Send KPI Report |                                                                                                                 |
| Slack - Post Dashboard Snapshot| Slack                | Posts KPI summary message to Slack   | Set - Format Output Data       | -                                 |                                                                                                                 |
| Gmail - Send KPI Report       | Gmail                 | Sends detailed KPI report via email  | Set - Format Output Data       | -                                 |                                                                                                                 |
| Error Trigger Handler         | Error Trigger        | Triggers on workflow errors           | -                              | Slack - Send Error Alert           |                                                                                                                 |
| Slack - Send Error Alert      | Slack                | Sends error alert to Slack channel    | Error Trigger Handler          | -                                 |                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node: "Daily Cron Trigger"**  
   - Set to trigger daily at your preferred time and timezone.

2. **Create a ClickUp node: "ClickUp - Fetch Tasks"**  
   - Operation: `getAll` tasks  
   - Set Team ID, Space ID, Folder ID, List ID to target your ClickUp task list  
   - Limit: 5 (adjust as needed)  
   - Filters: Exclude subtasks  
   - Authentication: Connect with OAuth2 credentials for ClickUp

3. **Create a Google Sheets node: "Google Sheets - Fetch Lead Data"**  
   - Set Document ID and Sheet Name to your leads data spreadsheet  
   - Authentication: Connect with Google OAuth2 credentials

4. **Connect "Daily Cron Trigger" to both "ClickUp - Fetch Tasks" and "Google Sheets - Fetch Lead Data" nodes in parallel.**

5. **Create a Merge node: "Merge - Consolidate Datasets"**  
   - Connect the outputs of both data fetch nodes to the two inputs of the Merge node  
   - Use default or "append" merge mode to combine datasets

6. **Create a Code node: "Code - Compute KPI Trends"**  
   - Paste the JavaScript code from the workflow to compute KPIs from merged data  
   - Connect the output of Merge node to this Code node

7. **Create a Set node: "Set - Format Output Data"**  
   - Map all KPI fields from the code node output to named variables (e.g., tasks, leads, trends) as per the workflow  
   - Connect Code node output to this Set node

8. **Create a Slack node: "Slack - Post Dashboard Snapshot"**  
   - Configure with Slack OAuth2 credentials  
   - Set the target channel ID (e.g., general channel)  
   - Use expressions in the message to display KPI summary from Set node data  
   - Connect Set node output to Slack node

9. **Create a Gmail node: "Gmail - Send KPI Report"**  
   - Configure with Gmail OAuth2 credentials  
   - Set recipient email (can use environment variable or static email)  
   - Paste the provided detailed HTML email template, using expressions to populate KPI data  
   - Connect Set node output to Gmail node

10. **Create an Error Trigger node: "Error Trigger Handler"**  
    - No configuration needed, it listens to workflow errors

11. **Create a Slack node: "Slack - Send Error Alert"**  
    - Configure Slack OAuth2 credentials and set alerts channel ID  
    - Use an error message template with expressions for error details, node name, execution ID, and timestamp  
    - Connect Error Trigger Handler output to this Slack node

12. **Add Sticky Notes to document each major block for clarity and future maintenance: Scheduled Trigger, Data Fetching (ClickUp, Google Sheets), Data Merging, KPI Computation, Data Formatting, Multi-Channel Distribution (Slack, Gmail), and Error Handling.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                               |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| Workflow requires OAuth2 credentials for ClickUp, Google Sheets, Slack, and Gmail integrations. | Setup prerequisite for all API nodes                                        |
| KPI computation code includes basic sentiment analysis using keyword matching for leads.      | Can be customized to improve accuracy or add more sophisticated NLP          |
| Slack channels for dashboard posting and error alerts must be configured with valid channel IDs. | Ensure Slack app permissions include chat:write scope                        |
| Gmail node uses HTML email template styled with inline CSS for rich report presentation.       | Can be customized or replaced with alternative email templates               |
| Cron node must be set carefully to match your timezone and desired report generation time.     | Avoid overlaps or missed runs                                                |
| Error handling ensures failure notifications via Slack, improving monitoring and response time.| Critical for production reliability                                          |
| For troubleshooting, check n8n execution logs and API rate limits on all connected services.  | Helps identify integration issues                                            |

---

*Disclaimer:* The provided text is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.