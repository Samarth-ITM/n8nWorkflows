Automated Sprint Reports from Jira to Stakeholders via Gmail

https://n8nworkflows.xyz/workflows/automated-sprint-reports-from-jira-to-stakeholders-via-gmail-8932


# Automated Sprint Reports from Jira to Stakeholders via Gmail

### 1. Workflow Overview

This workflow automates the generation and distribution of sprint reports from Jira to stakeholders via Gmail. It targets Agile teams using Jira Software Cloud who want to automatically send up-to-date sprint progress summaries on a regular schedule.

The workflow is logically divided into three main blocks:

- **1.1 Trigger and Configuration Setup**: Scheduled trigger initiates the workflow weekly, sets Jira and email configuration parameters.
- **1.2 Jira Data Retrieval and Validation**: Fetches all Jira issues for the current open sprint, extracts and normalizes relevant fields, and validates data to ensure completeness.
- **1.3 Metrics Calculation, Report Generation, and Email Notification**: Calculates key sprint metrics, generates an HTML report, and sends it via Gmail to configured recipients.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Configuration Setup

**Overview:**  
This block initiates the workflow on a weekly schedule (every Friday at 17:00) and sets key Jira and email configuration variables used downstream.

**Nodes Involved:**  
- Weekly trigger  
- Jira & email configuration

**Node Details:**

- **Weekly trigger**  
  - *Type & Role*: Schedule Trigger; starts workflow weekly.  
  - *Configuration*: Runs every Friday at 17:00 hours, triggered weekly by setting interval and day/hour.  
  - *Inputs/Outputs*: No input; outputs trigger event to next node.  
  - *Edge Cases*: Workflow runs only once per week; misconfiguration can cause missed or repeated runs.

- **Jira & email configuration**  
  - *Type & Role*: Set node; defines static configuration for Jira API and email recipients.  
  - *Configuration*: Sets JSON fields including Jira base URL, Jira API email, API token, project key, and email recipients.  
  - *Key Variables*:  
    - `jiraBaseUrl`: Jira Cloud instance URL.  
    - `jiraEmail`: Jira API email address.  
    - `jiraApiToken`: Jira API token for authentication.  
    - `projectKey`: Jira project key used in JQL.  
    - `emailRecipients`: Comma-separated email addresses for report distribution.  
  - *Inputs/Outputs*: Takes trigger input, outputs configuration JSON for next node.  
  - *Edge Cases*: Missing or invalid credentials will cause Jira API calls to fail.

---

#### 1.2 Jira Data Retrieval and Validation

**Overview:**  
Fetches all issues from the currently open sprint in the specified Jira project using JQL, then normalizes and validates critical fields such as story points and sprint data to ensure consistency.

**Nodes Involved:**  
- Get many issues  
- Validation & error handling

**Node Details:**

- **Get many issues**  
  - *Type & Role*: Jira node; fetches all issues matching JQL query.  
  - *Configuration*:  
    - Operation: `getAll` (fetches all matching issues).  
    - JQL: `project = your project AND sprint in openSprints()` to filter issues in the current sprint.  
    - Fields: summary, status, assignee, priority, issuetype, labels, created, updated, customfield_10016 (story points), customfield_10020 (sprint info).  
  - *Credentials*: Jira Software Cloud API OAuth credentials.  
  - *Inputs/Outputs*: Receives configuration JSON; outputs array of Jira issues.  
  - *Edge Cases*: Authentication failure, empty result if no open sprint or no issues.

- **Validation & error handling**  
  - *Type & Role*: Code node; processes raw Jira data to extract and normalize fields, replacing missing values with defaults.  
  - *Configuration*: JavaScript code extracts fields like key, summary, status, assignee, priority, story points (customfield_10016), sprint info (customfield_10020), and dates. It ensures fields like assignee and priority have fallback values such as "Unassigned" or "Undefined".  
  - *Inputs/Outputs*: Accepts Jira issues array; outputs normalized array of simplified issue objects.  
  - *Edge Cases*: Inconsistent Jira data formats (e.g., sprint sometimes an array), missing custom fields.

---

#### 1.3 Metrics Calculation, Report Generation, and Email Notification

**Overview:**  
Aggregates normalized data into key performance indicators (KPIs), creates a formatted HTML sprint report, and sends it by Gmail to stakeholders.

**Nodes Involved:**  
- Metrics calculation  
- HTML report generation  
- Email notification

**Node Details:**

- **Metrics calculation**  
  - *Type & Role*: Code node; computes sprint KPIs such as issue counts by status, story points completed, blockers count, and sprint timeline.  
  - *Configuration*: JavaScript code performs:  
    - Counting total issues, done, in progress, to do.  
    - Summing story points total and done.  
    - Identifying blockers by priority "Highest" or labels/status indicating blocking.  
    - Extracting sprint name and planned end date.  
  - *Inputs/Outputs*: Takes normalized issues; outputs metrics object with counts, story points, blockers, sprint info, and timestamp.  
  - *Edge Cases*: No issues (outputs message "Aucun ticket"), missing sprint info.

- **HTML report generation**  
  - *Type & Role*: Code node; builds an HTML string report containing sprint summary and detailed issues table with inline CSS.  
  - *Configuration*:  
    - Constructs an HTML table with columns: Key (linked to Jira), Summary, Status, Assignee, Priority, Story Points.  
    - Includes header with sprint name, completion stats, blockers count, and metadata such as date/time and sprint planned end.  
    - Uses inline styles for email client compatibility.  
  - *Inputs/Outputs*: Accepts metrics object; outputs JSON with `html` property for email body.  
  - *Edge Cases*: Empty issues list (renders empty table), broken links if Jira URL misconfigured.

- **Email notification**  
  - *Type & Role*: Gmail node; sends the generated report via email.  
  - *Configuration*:  
    - Subject: "Your Sprint Report".  
    - Recipient: configured email address(es).  
    - Message body: includes HTML report content and a link to the sprint board on Jira.  
  - *Credentials*: Gmail OAuth2 credentials.  
  - *Inputs/Outputs*: Receives report HTML; sends email; no output data.  
  - *Edge Cases*: Gmail authentication errors, invalid recipient addresses, email size limits.

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                  |
|-------------------------|------------------|---------------------------------------|------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Weekly trigger          | Schedule Trigger | Workflow start, scheduled trigger     |                        | Jira & email configuration | ## 1) Trigger and Normalize: Scheduled run every Friday at 17:00; triggers configuration setup and Jira fetch. |
| Jira & email configuration | Set              | Defines Jira and email parameters      | Weekly trigger          | Get many issues          | ## 1) Trigger and Normalize: Sets Jira base URL, API credentials, project key, and email recipients.          |
| Get many issues         | Jira             | Fetches all issues in current sprint   | Jira & email configuration | Validation & error handling | ## 1) Trigger and Normalize: Fetches issues using JQL and selects relevant fields for processing.              |
| Validation & error handling | Code             | Normalizes and validates Jira fields    | Get many issues         | Metrics calculation      | ## 1) Trigger and Normalize: Ensures fields like assignee and story points are set; normalizes missing values. |
| Metrics calculation     | Code             | Calculates sprint KPIs and aggregates data | Validation & error handling | HTML report generation   | ## 2) Calculate Metrics: Aggregates counts, story points, blockers, sprint info for the report.                |
| HTML report generation  | Code             | Builds HTML sprint report for email    | Metrics calculation     | Email notification       | ## 3) Generate Report & Send Notification: Creates clean HTML with inline CSS for email compatibility.        |
| Email notification      | Gmail            | Sends sprint report email to recipients | HTML report generation  |                         | ## 3) Generate Report & Send Notification: Sends email with sprint report and Jira sprint link.               |
| Sticky Note1            | Sticky Note      | Notes required credentials and config  |                        |                         | • Jira Cloud project (API email + API token) • Gmail credential                                              |
| Sticky Note             | Sticky Note      | Overview of Trigger and normalization  |                        |                         | ## 1) Trigger and Normalize: Best practice to validate and normalize early to avoid later errors.             |
| Sticky Note2            | Sticky Note      | Overview of metrics calculation         |                        |                         | ## 2) Calculate Metrics: Keep calculations in dedicated node for debugging and KPI adjustments.               |
| Sticky Note3            | Sticky Note      | Overview of report generation and email |                        |                         | ## 3) Generate Report & Send Notification: Use clean HTML template and send to correct recipients.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Weekly trigger`  
   - Type: Schedule Trigger  
   - Configure to run every Friday at 17:00 hours (set interval weeks, triggerAtDay=5 (Friday), triggerAtHour=17).

2. **Create a Set node for Jira & email configuration**  
   - Name: `Jira & email configuration`  
   - Type: Set  
   - Mode: Raw JSON  
   - Add JSON with keys:  
     - `jiraBaseUrl`: your Jira Cloud URL, e.g. `https://yourdomain.atlassian.net`  
     - `jiraEmail`: your Jira API email  
     - `jiraApiToken`: your Jira API token  
     - `projectKey`: your Jira project key  
     - `emailRecipients`: comma-separated email addresses  
   - Connect output of `Weekly trigger` to this node.

3. **Create a Jira node to fetch issues**  
   - Name: `Get many issues`  
   - Type: Jira  
   - Operation: `getAll`  
   - JQL: `project = your project AND sprint in openSprints()` (replace `your project` with actual project key)  
   - Fields: summary, status, assignee, priority, issuetype, labels, created, updated, customfield_10016 (story points), customfield_10020 (sprint info)  
   - Set credentials: Jira Software Cloud API with OAuth2 or API token credentials.  
   - Connect output of `Jira & email configuration` to this node.

4. **Create a Code node for validation and normalization**  
   - Name: `Validation & error handling`  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript that extracts and normalizes fields (see Block 2.2 above).  
   - Connect output of `Get many issues` to this node.

5. **Create a Code node for metrics calculation**  
   - Name: `Metrics calculation`  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript that computes counts, story points, blockers, sprint info, and timestamps.  
   - Connect output of `Validation & error handling` to this node.

6. **Create a Code node for HTML report generation**  
   - Name: `HTML report generation`  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript that builds the HTML table, header, and metadata with inline CSS.  
   - Connect output of `Metrics calculation` to this node.

7. **Create a Gmail node to send the email**  
   - Name: `Email notification`  
   - Type: Gmail  
   - Set credentials: OAuth2 Gmail account connected to your sending email.  
   - Parameters:  
     - Send To: your recipient email(s) from configuration or hardcoded for testing  
     - Subject: "Your Sprint Report"  
     - Message: use expression to insert HTML report:  
       ```
       Hello,  
       This is your current sprint report

       {{$json.html}}

       <p><a href="https://yourdomain.atlassian.net/jira/software/c/projects/TES/boards/3">See the sprint in Jira</a></p>
       ```  
   - Connect output of `HTML report generation` to this node.

8. **Validate all credentials**:  
   - Ensure Jira credentials have API access and correct permissions for the project.  
   - Ensure Gmail OAuth2 credentials have sufficient scope to send emails.

9. **Test the workflow on demand** or wait for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Requires Jira Cloud API email and API token for authentication.                                                                    | Sticky Note1                                                                                     |
| Gmail account must be configured with OAuth2 credentials for sending emails securely.                                               | Sticky Note1                                                                                     |
| Scheduled trigger set for every Friday at 17:00 to generate weekly sprint report.                                                  | Sticky Note                                                                                      |
| Early data validation and normalization prevent errors in metrics calculations and report generation.                              | Sticky Note                                                                                      |
| Calculations are separated in a dedicated node for easier debugging and future KPI additions.                                      | Sticky Note2                                                                                     |
| HTML report uses inline CSS to ensure compatibility with most email clients.                                                      | Sticky Note3                                                                                     |
| The email includes a direct link to the Jira sprint board for quick access by stakeholders.                                         | Email notification node message configuration                                                  |
| Reference Jira sprint board URL and issue links must be adjusted according to your Jira instance domain and project keys.          | Email notification and HTML report generation nodes                                            |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.