Track Employee Attendance with Analytics, Email Reports & Slack Alerts Using Google Sheets

https://n8nworkflows.xyz/workflows/track-employee-attendance-with-analytics--email-reports---slack-alerts-using-google-sheets-10106


# Track Employee Attendance with Analytics, Email Reports & Slack Alerts Using Google Sheets

### 1. Workflow Overview

This workflow automates the tracking of employee attendance by integrating data from Google Sheets, performing analytics, generating alerts, and distributing summarized reports via email and Slack. It is designed for HR or management teams to monitor daily attendance metrics without manual intervention.

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger**: Initiates the workflow on an hourly basis.
- **1.2 Data Retrieval**: Fetches attendance logs and employee master data from Google Sheets.
- **1.3 Analytics Engine**: Merges and analyzes data to produce attendance statistics, detect anomalies, and generate alerts.
- **1.4 Validation & Notification Routing**: Checks data availability, evaluates alert severity, and routes notifications accordingly.
- **1.5 Report Formatting**: Prepares visually rich email and Slack message content based on analytics results.
- **1.6 Distribution & Archiving**: Sends emails and Slack messages, and logs daily summaries back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview**:  
  Triggers the workflow every hour to ensure attendance data is processed regularly and reports are generated consistently.

- **Nodes Involved**:  
  - Schedule Trigger

- **Node Details**:  
  - **Schedule Trigger**  
    - Type: `scheduleTrigger`  
    - Configuration: Set to trigger on an hourly interval (every 1 hour).  
    - Inputs: None (start node).  
    - Outputs: Connects to both "Fetch Attendance Records" and "Fetch Employee Master Data" nodes.  
    - Potential Failures: Scheduler misconfiguration or system downtime could prevent triggering.

---

#### 1.2 Data Retrieval

- **Overview**:  
  Retrieves required data from two Google Sheets documents: one for daily attendance logs and one for employee master records. These datasets form the basis for subsequent analysis.

- **Nodes Involved**:  
  - Fetch Attendance Records  
  - Fetch Employee Master Data

- **Node Details**:  
  - **Fetch Attendance Records**  
    - Type: `googleSheets`  
    - Configuration: Reads from the "AttendanceLogs" sheet in a specified Google Sheets document (spreadsheet ID placeholder used).  
    - Authentication: Uses Google API Service Account credentials.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Sends data to "Analytics Engine".  
    - Edge Cases: Invalid spreadsheet ID, permission errors, empty or malformed sheet data.

  - **Fetch Employee Master Data**  
    - Type: `googleSheets`  
    - Configuration: Reads from the "Employees" sheet in another Google Sheets document (spreadsheet ID placeholder used).  
    - Authentication: Google API Service Account credentials (same as above).  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Sends data to "Analytics Engine".  
    - Edge Cases: As above, plus missing employee records causing downstream mismatches.

---

#### 1.3 Analytics Engine

- **Overview**:  
  Combines attendance and employee data to calculate key attendance metrics, categorize employee statuses (Present, Absent, Late, Leave, WFH), generate department-specific summaries, and detect alerts based on thresholds.

- **Nodes Involved**:  
  - Analytics Engine

- **Node Details**:  
  - **Analytics Engine**  
    - Type: `code` (JavaScript)  
    - Configuration:  
      - Parses input from both Google Sheets nodes.  
      - Builds an employee lookup map.  
      - Filters attendance records for today’s date.  
      - Counts status occurrences and builds lists of late, absent, and WFH employees.  
      - Generates department-wise attendance breakdowns.  
      - Calculates attendance rate, punctuality rate, absenteeism rate.  
      - Detects alerts for high lateness (>10%) and high absence (>15%).  
      - Determines whether management notifications are necessary based on alerts and absolute absence counts.  
    - Inputs: Data arrays from "Fetch Attendance Records" and "Fetch Employee Master Data".  
    - Outputs: JSON object with all computed metrics and alerts, sent to "Records Available".  
    - Edge Cases: Missing or inconsistent employee IDs, no attendance records for the day, data format inconsistencies, JS code runtime errors.

---

#### 1.4 Validation & Notification Routing

- **Overview**:  
  Validates whether any attendance records were processed, then evaluates if critical alerts warrant notification to management. Depending on these checks, routes data to email formatting, Slack formatting, and summary logging.

- **Nodes Involved**:  
  - Records Available (If)  
  - Critical Alerts (If)

- **Node Details**:  
  - **Records Available**  
    - Type: `if`  
    - Checks if the number of attendance records processed (`recordsProcessed`) is greater than zero.  
    - Inputs: From "Analytics Engine".  
    - Outputs:  
      - True branch to "Critical Alerts", "Format Slack", and "Log Summary".  
      - False branch aborts flow (no downstream nodes connected).  
    - Edge Cases: Zero records processed causing workflow early exit.

  - **Critical Alerts**  
    - Type: `if`  
    - Checks if the flag `shouldNotifyManagement` is true (alerts present or absence count > 5).  
    - Inputs: From "Records Available".  
    - Outputs:  
      - True branch to "Format Email".  
      - False branch skips email (no downstream nodes connected).  
    - Edge Cases: Incorrect boolean evaluation or missing flag causing missed notifications.

---

#### 1.5 Report Formatting

- **Overview**:  
  Generates formatted daily attendance reports for email and Slack channels, including metrics, alerts, and employee breakdowns with visual styling for clear comprehension.

- **Nodes Involved**:  
  - Format Email  
  - Format Slack

- **Node Details**:  
  - **Format Email**  
    - Type: `code` (JavaScript)  
    - Generates HTML email body with:  
      - Header with date and time  
      - Alert section if alerts exist  
      - Grid summary of Present, Late, Absent, Leave counts  
      - Key metrics (attendance rate, punctuality rate, total employees)  
      - Email subject dynamically prefixed with urgent indicator if high-priority alerts exist  
    - Inputs: From "Critical Alerts" (only if critical alerts exist).  
    - Outputs: To "Send Email".  
    - Edge Cases: HTML rendering issues, missing data fields.

  - **Format Slack**  
    - Type: `code` (JavaScript)  
    - Creates Slack message blocks with sections for:  
      - Header with date  
      - Fields summarizing Present, Late, Absent, Leave counts  
      - Lists of late and absent employees (up to 5 each) or "None" if empty  
    - Inputs: From "Records Available" (executed regardless of alert presence).  
    - Outputs: To "Post to Slack".  
    - Edge Cases: Slack block formatting errors, empty employee lists.

---

#### 1.6 Distribution & Archiving

- **Overview**:  
  Sends the formatted email report to management, posts Slack messages to a specified channel, and appends a summary record to a Google Sheet for audit and trend analysis.

- **Nodes Involved**:  
  - Send Email  
  - Post to Slack  
  - Log Summary

- **Node Details**:  
  - **Send Email**  
    - Type: `emailSend`  
    - Sends email to `management@company.com` from `hr@company.com`.  
    - Uses SMTP credentials configured in n8n.  
    - Subject and body taken from "Format Email".  
    - Inputs: From "Format Email".  
    - Edge Cases: SMTP authentication failures, email delivery failures.

  - **Post to Slack**  
    - Type: `slack`  
    - Posts to Slack channel with ID `C12345678` (placeholder).  
    - Message content and blocks from "Format Slack".  
    - Uses Slack OAuth credentials.  
    - Inputs: From "Format Slack".  
    - Edge Cases: Slack API rate limits, invalid channel ID, token expiry.

  - **Log Summary**  
    - Type: `googleSheets`  
    - Appends daily summary data to "DailySummary" sheet in a specified Google Sheets document.  
    - Uses Google API Service Account credentials.  
    - Inputs: From "Records Available".  
    - Edge Cases: Permission errors, data mapping issues.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                               | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                 |
|-------------------------|--------------------|-----------------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger        | scheduleTrigger    | Initiates workflow hourly                      | -                            | Fetch Attendance Records, Fetch Employee Master Data | Runs hourly to monitor attendance with zero manual effort.                                  |
| Fetch Attendance Records | googleSheets       | Retrieves daily attendance logs                | Schedule Trigger             | Analytics Engine                 | Fetches attendance and employee master data, then merges them for enriched analytics...     |
| Fetch Employee Master Data | googleSheets     | Retrieves employee master data                  | Schedule Trigger             | Analytics Engine                 | Fetches attendance and employee master data, then merges them for enriched analytics...     |
| Analytics Engine        | code               | Processes data, calculates metrics, generates alerts | Fetch Attendance Records, Fetch Employee Master Data | Records Available                | Fetches attendance and employee master data, then merges them for enriched analytics...     |
| Records Available       | if                 | Checks if there are any attendance records     | Analytics Engine             | Critical Alerts, Format Slack, Log Summary | Validates data, prioritizes alerts, and routes notifications via email, Slack, and database... |
| Critical Alerts         | if                 | Checks if management notifications are needed | Records Available            | Format Email                    | Validates data, prioritizes alerts, and routes notifications via email, Slack, and database... |
| Format Email            | code               | Formats HTML email report                       | Critical Alerts              | Send Email                     | Sends visually formatted reports with metrics, alerts, and detailed employee breakdowns.     |
| Send Email              | emailSend          | Sends email report to management                | Format Email                 | -                              | Sends visually formatted reports with metrics, alerts, and detailed employee breakdowns.     |
| Format Slack            | code               | Formats Slack message blocks                    | Records Available            | Post to Slack                  | Validates data, prioritizes alerts, and routes notifications via email, Slack, and database... |
| Post to Slack           | slack              | Posts report message to Slack                   | Format Slack                 | -                              | Validates data, prioritizes alerts, and routes notifications via email, Slack, and database... |
| Log Summary             | googleSheets       | Appends daily summary to Google Sheets          | Records Available            | -                              | Validates data, prioritizes alerts, and routes notifications via email, Slack, and database... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: `Schedule Trigger`  
   - Set to trigger every 1 hour (Interval: hours).  
   - This node starts the workflow.

2. **Create Fetch Attendance Records node**  
   - Type: `Google Sheets`  
   - Operation: Read data from sheet "AttendanceLogs".  
   - Set Google Sheets document ID to your attendance spreadsheet.  
   - Authentication: Use Google API Service Account credentials.  
   - Connect output from Schedule Trigger to this node.

3. **Create Fetch Employee Master Data node**  
   - Type: `Google Sheets`  
   - Operation: Read data from sheet "Employees".  
   - Set Google Sheets document ID to your employee master spreadsheet.  
   - Authentication: Use same Google API Service Account credentials.  
   - Connect output from Schedule Trigger to this node.

4. **Create Analytics Engine node (Code node)**  
   - Type: `Code`  
   - Insert JavaScript code that:  
     - Reads data from both "Fetch Attendance Records" and "Fetch Employee Master Data" inputs.  
     - Builds employee map for lookup.  
     - Filters today's attendance records.  
     - Counts statuses, compiles lists, calculates attendance and punctuality rates.  
     - Generates department breakdowns and alert list based on thresholds.  
     - Outputs an object with all computed metrics and flags.  
   - Connect outputs of both Google Sheets nodes to this node.

5. **Create Records Available node (If node)**  
   - Type: `If`  
   - Condition: Check if `recordsProcessed` > 0.  
   - Connect output from Analytics Engine to this node.

6. **Create Critical Alerts node (If node)**  
   - Type: `If`  
   - Condition: Boolean check if `shouldNotifyManagement` === true.  
   - Connect true branch from Records Available to this node.

7. **Create Format Email node (Code node)**  
   - Type: `Code`  
   - Input: Data from Critical Alerts node.  
   - Output:  
     - Construct an HTML email with dynamic sections for date/time, alerts, attendance stats, and key metrics.  
     - Compose email subject with an urgent prefix if high-priority alerts exist.  
   - Connect true branch from Critical Alerts to this node.

8. **Create Send Email node (Email Send)**  
   - Type: `Email Send`  
   - SMTP credentials: Use configured SMTP account (e.g., company mail server).  
   - Set From: `hr@company.com`  
   - Set To: `management@company.com`  
   - Subject and HTML body: Use expressions to pull from Format Email node outputs.  
   - Connect output from Format Email node.

9. **Create Format Slack node (Code node)**  
   - Type: `Code`  
   - Input: Data from Records Available node (true branch).  
   - Output: Slack message blocks with summary fields and lists of late/absent employees.  
   - Connect true branch from Records Available to this node.

10. **Create Post to Slack node (Slack)**  
    - Type: `Slack`  
    - Slack credentials: Use configured Slack OAuth credentials.  
    - Channel: Set Slack channel ID where reports are posted.  
    - Text: Static text "Daily Attendance Report" or dynamic.  
    - Blocks: Use Slack blocks from Format Slack node output.  
    - Connect output from Format Slack node.

11. **Create Log Summary node (Google Sheets)**  
    - Type: `Google Sheets`  
    - Operation: Append data to "DailySummary" sheet in your summary spreadsheet.  
    - Authentication: Use Google API Service Account credentials.  
    - Map input fields from Records Available node output metrics.  
    - Connect true branch from Records Available node.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Runs hourly to monitor attendance with zero manual effort.                                                 | Sticky note on Schedule Trigger node                                                               |
| Fetches attendance and employee master data, then merges them for enriched analytics and generates alerts. | Sticky note covering Fetch Attendance Records, Fetch Employee Master Data, and Analytics Engine nodes |
| Validates data, prioritizes alerts, routes notifications via email, Slack, and logs daily summaries.       | Sticky note covering Records Available, Critical Alerts, Format Slack, and Log Summary nodes        |
| Sends visually formatted reports with metrics, alerts, and detailed employee breakdowns.                   | Sticky note covering Format Email and Send Email nodes                                             |

---

**Disclaimer:** The provided content is extracted from a fully automated n8n workflow. The workflow complies strictly with current content policies and handles only legal, public data. It contains no illegal, offensive, or protected elements.