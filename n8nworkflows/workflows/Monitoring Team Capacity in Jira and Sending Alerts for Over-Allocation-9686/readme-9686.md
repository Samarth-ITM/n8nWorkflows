Monitoring Team Capacity in Jira and Sending Alerts for Over-Allocation

https://n8nworkflows.xyz/workflows/monitoring-team-capacity-in-jira-and-sending-alerts-for-over-allocation-9686


# Monitoring Team Capacity in Jira and Sending Alerts for Over-Allocation

### 1. Workflow Overview

This n8n workflow is designed for **monitoring team capacity within Jira** and **sending alerts when team members are over-allocated**. It targets project managers and team leads who want to proactively manage workload distribution, prevent burnout, and maintain visibility of resource utilization.

The workflow is logically divided into the following main blocks:

- **1.1 Input Reception & Jira Data Retrieval**  
  Triggered manually, it queries Jira for all active (in-progress) issues to obtain real-time workload data.

- **1.2 Data Validation & Error Handling**  
  Validates whether Jira returned results and logs errors if no data is retrieved or the query fails.

- **1.3 Team Capacity Calculation**  
  Processes Jira issues to aggregate time spent per assignee, calculates utilization percentages against a defined daily capacity, and flags over-allocated users.

- **1.4 Capacity Data Logging**  
  Appends calculated capacity data to a Google Sheet for historical tracking and trend analysis.

- **1.5 Over-Allocation Detection & Report Generation**  
  Filters over-allocated members and generates a consolidated alert report detailing the issue and recommended corrective actions.

- **1.6 Notification Dispatch**  
  Sends an email alert to the project manager/team lead with the over-allocation report for immediate intervention.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Jira Data Retrieval

- **Overview:**  
  Manually triggered entry point that fetches all active Jira issues with specific JQL criteria to provide current workload data.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Jira Get Issues Node

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger, no parameters.  
    - Input: None (start node)  
    - Output: Triggers workflow execution.  
    - Edge cases: None; manual execution only.

  - **Jira Get Issues Node**  
    - Type: Jira node (API integration)  
    - Configuration:  
      - Operation: Get all issues  
      - JQL: `statusCategory != Done AND status = "In Progress"` (fetches all non-completed, in-progress issues)  
    - Credentials: Jira Software Cloud API (configured with valid OAuth/token)  
    - Input: Trigger from manual node  
    - Output: JSON array of Jira issues  
    - Edge cases: API authentication failure, invalid JQL syntax, Jira downtime, empty result set.

#### 2.2 Data Validation & Error Handling

- **Overview:**  
  Validates that Jira returned issue data; if no data, logs the failure in a Google Sheet for monitoring.

- **Nodes Involved:**  
  - Data Validation  
  - Log Query Failures to Error Sheet

- **Node Details:**

  - **Data Validation**  
    - Type: IF node  
    - Configuration: Checks if the input array length > 0 (issues returned)  
    - Input: Issues from Jira Get Issues Node  
    - Output:  
      - True path: Proceed to capacity calculation  
      - False path: Route to error logging  
    - Edge cases: Empty dataset, expression evaluation errors.

  - **Log Query Failures to Error Sheet**  
    - Type: Google Sheets node  
    - Configuration: Appends a new row to the “error log sheet” with error details (e.g., no data from Jira)  
    - Credentials: Google Sheets OAuth2  
    - Input: Error data from Data Validation false path  
    - Output: None  
    - Edge cases: Google Sheets API quota exceeded, invalid credentials, network issues.

#### 2.3 Team Capacity Calculation

- **Overview:**  
  Aggregates time spent on tasks per team member, converts time tracking to hours, calculates utilization percentage vs. 8-hour daily capacity, and flags over-allocated members.

- **Nodes Involved:**  
  - Capacity Calculator

- **Node Details:**

  - **Capacity Calculator**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Runs once for all input items (all Jira issues)  
      - Extracts assignee name or assigns "Unassigned" if none  
      - Converts Jira time spent (seconds) to hours  
      - Sums total hours per assignee  
      - Calculates utilization % = (total hours / 8) * 100  
      - Flags status "Overallocated" if utilization > 100%, else "OK"  
    - Input: Validated Jira issues  
    - Output: Structured array of team member capacity objects, one per assignee  
    - Edge cases: Missing assignee data, zero time spent, division by zero (not applicable here), empty input array.

#### 2.4 Capacity Data Logging

- **Overview:**  
  Logs each team member’s capacity data to a Google Sheet for historical tracking, facilitating retrospective analysis and trend visualization.

- **Nodes Involved:**  
  - Log Capacity Data to Tracking Sheet

- **Node Details:**

  - **Log Capacity Data to Tracking Sheet**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append row  
      - Sheet name: “Team Capacity Tracking”  
      - Columns: Assignee, Total Hours, Utilization %, Status, Timestamp (auto-generated)  
    - Credentials: Google Sheets OAuth2  
    - Input: Output from Capacity Calculator  
    - Output: None  
    - Edge cases: API quota limits, credential expiration, malformed data.

#### 2.5 Over-Allocation Detection & Report Generation

- **Overview:**  
  Filters out team members flagged as “Overallocated” and generates a comprehensive alert report message with a dynamic subject line and recommendations.

- **Nodes Involved:**  
  - Over-Allocation Check  
  - Alert Report Generator

- **Node Details:**

  - **Over-Allocation Check**  
    - Type: IF node  
    - Configuration: Filters where status == "Overallocated"  
    - Input: Capacity data (multiple items)  
    - Output:  
      - True path: Over-allocated members for alert report  
      - False path: No over-allocation, no alert needed  
    - Edge cases: Case sensitivity, empty input, expression parsing errors.

  - **Alert Report Generator**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Aggregates all over-allocated users into one alert report message  
      - Includes: Warning header with emoji, list of members with hours and utilization %, timestamp, and recommended corrective actions  
      - Generates dynamic email subject line indicating number of over-allocated members  
      - Handles “all clear” scenario with appropriate message  
    - Input: Filtered over-allocated members  
    - Output: JSON object with subject, message, overallocatedCount, hasIssues, and details array  
    - Edge cases: Empty input (no over-allocations), formatting errors.

#### 2.6 Notification Dispatch

- **Overview:**  
  Sends an email notification with the over-allocation alert report to the project manager/team lead, enabling immediate action and documentation.

- **Nodes Involved:**  
  - Send Over-Allocation Alert to Manager

- **Node Details:**

  - **Send Over-Allocation Alert to Manager**  
    - Type: Gmail node  
    - Configuration:  
      - To: `newscctv22@gmail.com` (project manager’s email)  
      - Subject: Dynamic, from Alert Report Generator output  
      - Message body: Detailed alert message from Alert Report Generator  
      - CC: None by default  
    - Credentials: Gmail OAuth2 (configured)  
    - Input: Alert report from Alert Report Generator  
    - Output: Email sent confirmation  
    - Edge cases: Email sending failure, OAuth token expiration, invalid email address.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                                  | Input Node(s)                      | Output Node(s)                          | Sticky Note                                                                                                 |
|----------------------------------|---------------------|-------------------------------------------------|----------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Entry point to start workflow                    | None                             | Jira Get Issues Node                   |                                                                                                             |
| Jira Get Issues Node              | Jira API node       | Fetch active Jira issues (in-progress)           | When clicking ‘Execute workflow’ | Data Validation                       | ## 📋 Fetch Active Jira Issues: Retrieves all non-done, in-progress Jira issues with full metadata          |
| Data Validation                  | IF node             | Validate Jira query returned data                 | Jira Get Issues Node              | Capacity Calculator, Log Query Failures to Error Sheet | ## ✅ Validate Issues Retrieved Successfully: Checks for non-empty Jira issue dataset before further processing |
| Log Query Failures to Error Sheet | Google Sheets       | Log Jira query failures/errors                     | Data Validation (false path)     | None                                  | ## 📊 Log Query Failures to Error Sheet: Records Jira API failures for troubleshooting and monitoring        |
| Capacity Calculator              | Code (JavaScript)   | Calculate total hours and utilization per member | Data Validation (true path)      | Over-Allocation Check, Log Capacity Data to Tracking Sheet | ## 📊 Calculate Team Member Utilization: Aggregates time spent and calculates utilization % with overflow flag |
| Log Capacity Data to Tracking Sheet | Google Sheets       | Append capacity data to tracking sheet             | Capacity Calculator              | Over-Allocation Check                 | ## 📈 Log Capacity Data to Tracking Sheet: Stores time-series capacity data for trend analysis                |
| Over-Allocation Check            | IF node             | Filter over-allocated members                      | Capacity Calculator, Log Capacity Data to Tracking Sheet | Alert Report Generator               | ## ⚠️ Detect Over-Allocated Team Members: Filters members with utilization > 100% for alerting               |
| Alert Report Generator           | Code (JavaScript)   | Generate consolidated alert report                 | Over-Allocation Check            | Send Over-Allocation Alert to Manager | ## 📢 Generate Over-Allocation Alert Report: Creates detailed alert message with recommendations and timestamp |
| Send Over-Allocation Alert to Manager | Gmail               | Send alert email to manager/team lead              | Alert Report Generator           | None                                  | ## 📧 Send Over-Allocation Alert to Manager: Sends formatted email alert with over-allocation details         |
| Sticky Note                     | Sticky Note         | Documentation for Send Over-Allocation Alert node | None                             | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note1                    | Sticky Note         | Documentation for Alert Report Generator            | None                             | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note2                    | Sticky Note         | Documentation for Over-Allocation Check             | None                             | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note3                    | Sticky Note         | Documentation for Log Capacity Data to Tracking Sheet | None                          | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note4                    | Sticky Note         | Documentation for Capacity Calculator                | None                             | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note5                    | Sticky Note         | Documentation for Log Query Failures to Error Sheet  | None                             | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note6                    | Sticky Note         | Documentation for Data Validation                     | None                             | None                                  | See detailed notes in node positions                                                                        |
| Sticky Note7                    | Sticky Note         | Documentation for Jira Get Issues Node                | None                             | None                                  | See detailed notes in node positions                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Jira Get Issues Node**  
   - Name: `Jira Get Issues Node`  
   - Type: Jira  
   - Operation: Get All Issues  
   - Parameters:  
     - JQL: `statusCategory != Done AND status = "In Progress"`  
   - Credentials: Configure Jira Software Cloud API credentials with valid OAuth or API token.  
   - Connect `When clicking ‘Execute workflow’` main output to this node.

3. **Create IF Node for Data Validation**  
   - Name: `Data Validation`  
   - Type: IF  
   - Condition: Check if number of input items > 0 (expression: `{{$input.all().length > 0}}`)  
   - Connect Jira Get Issues Node output to this node.

4. **Create Google Sheets Node to Log Query Failures**  
   - Name: `Log Query Failures to Error Sheet`  
   - Type: Google Sheets  
   - Operation: Append Row  
   - Google Sheets Document ID: Use your Google Sheet ID  
   - Sheet Name: `error log sheet` (or your chosen sheet)  
   - Columns: `error_id`, `error`  
   - Credentials: Configure Google Sheets OAuth2 credentials.  
   - Connect Data Validation false output (no issues) to this node.

5. **Create Code Node for Capacity Calculation**  
   - Name: `Capacity Calculator`  
   - Type: Code (JavaScript)  
   - Parameters: Paste the following code:

```javascript
const items = $input.all();

if (items.length === 0) {
  return { json: { message: "No issues found." } };
}

const userCapacity = {};

items.forEach(item => {
  const issue = item.json;
  const assignee = issue.fields?.assignee?.displayName || 'Unassigned';
  const timeSpent = issue.fields?.timespent || 0;
  const hoursSpent = timeSpent / 3600;
  if (!userCapacity[assignee]) {
    userCapacity[assignee] = 0;
  }
  userCapacity[assignee] += hoursSpent;
});

const utilizationData = [];
const maxCapacity = 8;

for (let assignee in userCapacity) {
  const totalHours = userCapacity[assignee];
  const utilizationPercentage = (totalHours / maxCapacity) * 100;

  utilizationData.push({
    assignee,
    totalHours: parseFloat(totalHours.toFixed(2)),
    utilizationPercentage: parseFloat(utilizationPercentage.toFixed(2)),
    status: utilizationPercentage > 100 ? 'Overallocated' : 'OK'
  });
}

return utilizationData.map(data => ({ json: data }));
```

   - Set to run once for all items.  
   - Connect Data Validation true output (issues present) to this node.

6. **Create Google Sheets Node to Log Capacity Data**  
   - Name: `Log Capacity Data to Tracking Sheet`  
   - Type: Google Sheets  
   - Operation: Append Row  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: `Team Capacity Tracking`  
   - Columns mapped:  
     - Assignee: `{{$json.assignee}}`  
     - Total Hours: `{{$json.totalHours}}`  
     - Utilization %: `{{$json.utilizationPercentage}}`  
     - Status: `{{$json.status}}`  
     - Timestamp: Use Google Sheets auto-timestamp or add current date string via a Set node if required  
   - Credentials: Google Sheets OAuth2  
   - Connect Capacity Calculator output to this node.

7. **Create IF Node for Over-Allocation Check**  
   - Name: `Over-Allocation Check`  
   - Type: IF  
   - Condition: Check if `{{$json.status}}` equals `"Overallocated"`  
   - Connect Capacity Calculator output to this node **and** also connect Log Capacity Data to Tracking Sheet output to this node (both feed into it).

8. **Create Code Node for Alert Report Generation**  
   - Name: `Alert Report Generator`  
   - Type: Code (JavaScript)  
   - Parameters: Paste the following code:

```javascript
const items = $input.all();

if (items.length === 0) {
  return [{
    json: {
      subject: "✅ Team Capacity Report - All Clear",
      message: "No team members are currently over-allocated.",
      hasIssues: false
    }
  }];
}

let reportLines = items.map(item =>
  `• ${item.json.assignee}: ${item.json.totalHours}h logged (${item.json.utilizationPercentage}% capacity)`
);

const message = `⚠️ TEAM OVER-ALLOCATION ALERT\n\n` +
  `The following team members are over-allocated:\n\n` +
  `${reportLines.join('\n')}\n\n` +
  `Recommended Action: Please review workload distribution and consider:\n` +
  `- Moving lower priority tasks to next sprint\n` +
  `- Redistributing work to available team members\n` +
  `- Extending deadlines if necessary\n\n` +
  `Report generated: ${new Date().toLocaleString()}`;

return [{
  json: {
    subject: `⚠️ Team Over-Allocation Alert - ${items.length} Member(s)`,
    message,
    overallocatedCount: items.length,
    hasIssues: true,
    details: items.map(i => i.json)
  }
}];
```

   - Connect Over-Allocation Check true output (over-allocated members) to this node.

9. **Create Gmail Node to Send Alert Email**  
   - Name: `Send Over-Allocation Alert to Manager`  
   - Type: Gmail  
   - To: `newscctv22@gmail.com` (replace with your manager’s email)  
   - Subject: `{{$json.subject}}` (dynamic from Alert Report Generator)  
   - Message: `{{$json.message}}` (dynamic from Alert Report Generator)  
   - Credentials: Gmail OAuth2 with appropriate permissions  
   - Connect Alert Report Generator output to this node.

10. **Set Workflow Activation**  
    - After verifying all nodes and connections, activate the workflow or run manually to test.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                         | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| This workflow enables early detection of team member over-allocation to prevent burnout and improve sprint planning accuracy.                                                       | Overall workflow purpose                                                                           |
| Google Sheets integration requires OAuth2 credentials with write access to designated spreadsheets “Team Capacity Tracking” and “error log sheet”.                                   | Google Sheets API setup                                                                            |
| Jira integration depends on valid Jira Software Cloud API credentials and appropriate permissions to query issues with JQL.                                                        | Jira API setup and permissions                                                                    |
| Email notifications sent via Gmail require OAuth2 credentials with SMTP send access. Adjust recipient address in the Gmail node as needed.                                         | Gmail OAuth2 configuration                                                                        |
| Dynamic subject lines and messages improve alert visibility and prioritization in email clients.                                                                                    | Alert Report Generator node                                                                        |
| The workflow handles empty Jira responses gracefully by logging errors and sending all-clear notifications, ensuring robustness and preventing silent failures.                    | Data Validation and Alert Report Generator nodes                                                  |
| Recommended corrective actions in alerts include redistribution of tasks, moving tasks to future sprints, and deadline adjustments.                                                 | Alert Report Generator message content                                                           |
| This automation facilitates data-driven resource management and provides audit trails via Google Sheets for retrospective analysis and SLA monitoring.                             | Capacity logging and error logging nodes                                                          |

---

**Disclaimer:** The text provided is generated exclusively from an automated n8n workflow. It strictly complies with content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.