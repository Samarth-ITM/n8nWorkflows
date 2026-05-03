Send Monthly Toggl Time Tracking Summary via Resend Email

https://n8nworkflows.xyz/workflows/send-monthly-toggl-time-tracking-summary-via-resend-email-9864


# Send Monthly Toggl Time Tracking Summary via Resend Email

### 1. Workflow Overview

This workflow automates the monthly generation and emailing of a Toggl Track time tracking summary report. It is designed for users managing time entries in Toggl who want a consolidated, client- and project-level breakdown of tracked hours for the previous month, delivered via email through the Resend service.

The workflow is structured into the following logical blocks:

- **1.1 Schedule Trigger:** Automatically starts the workflow monthly at a specified hour.
- **1.2 Data Retrieval from Toggl:** Fetches projects and detailed time summary data from Toggl for the prior month.
- **1.3 Data Processing and Merging:** Processes raw Toggl data, associates projects with clients, and aggregates tracked time.
- **1.4 HTML Report Generation:** Converts the aggregated data into a clean, readable HTML report.
- **1.5 Email Dispatch via Resend:** Sends the generated report by email using the Resend API.

Supporting sticky notes provide documentation, instructions, and contextual guidance for configuration and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block initiates the workflow automatically once every month at 4 AM server time.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow monthly.  
    - Configuration:  
      - Interval set to "months" with trigger hour at 4 AM.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers two HTTP Request nodes to fetch Toggl data.  
    - Edge Cases:  
      - Server time zone differences may affect trigger time.  
      - If the node is disabled, the workflow will not run automatically.  
    - Sticky Notes:  
      - "Schedule automation (monthly)" — reminds the user to adjust timing as needed.  

---

#### 1.2 Data Retrieval from Toggl

- **Overview:**  
  Fetches necessary data from Toggl API: the list of projects in the workspace and a summary of time entries grouped by clients and projects for the previous month.

- **Nodes Involved:**  
  - Get Toggle Projects  
  - Get Toggle Summary  

- **Node Details:**

  - **Get Toggle Projects**  
    - Type: HTTP Request  
    - Role: Retrieves all projects for the specified Toggl workspace.  
    - Configuration:  
      - URL endpoint: `https://api.track.toggl.com/api/v9/workspaces/TOGGL_WORKSPACE_ID/projects` (replace placeholder with actual workspace ID).  
      - Uses pagination to fetch all pages of projects.  
      - Authentication: Toggl API credential required.  
    - Inputs: Trigger from Schedule Trigger node.  
    - Outputs: Merges with project-related summary data downstream.  
    - Edge Cases:  
      - Invalid or missing workspace ID causes authentication or 404 errors.  
      - Network timeouts or API limits.  

  - **Get Toggle Summary**  
    - Type: HTTP Request  
    - Role: Retrieves summarized time entries for the previous calendar month, grouped by clients and sub-grouped by projects.  
    - Configuration:  
      - URL endpoint: `https://api.track.toggl.com/reports/api/v3/workspace/TOGGL_WORKSPACE_ID/summary/time_entries` (replace placeholder).  
      - Method: POST with JSON body parameters specifying start and end dates dynamically computed as first and last day of previous month.  
      - Grouping parameters: `grouping=clients`, `sub_grouping=projects`.  
      - Authentication: Toggl API credential required.  
      - Headers: Content-Type set to application/json.  
    - Inputs: Trigger from Schedule Trigger node.  
    - Outputs: Passes data to code node for project list extraction.  
    - Edge Cases:  
      - Missing or incorrect workspace ID or credentials leads to 401 or 403 errors.  
      - Date formatting errors in expressions.  
      - API limits or network issues.  
    - Sticky Notes:  
      - "Fetch Toggl clients, projects, and summary data for last month via HTTP nodes" with reminder to enter workspace ID.  

---

#### 1.3 Data Processing and Merging

- **Overview:**  
  Processes the Toggl summary data to extract project-level details, then merges this data with the full project list fetched earlier to enrich entries with project metadata.

- **Nodes Involved:**  
  - Prepare projects list (Code)  
  - Merge data  

- **Node Details:**

  - **Prepare projects list**  
    - Type: Code (JavaScript)  
    - Role: Parses the summary response to extract an array of objects each containing client_id, project_id, and seconds tracked.  
    - Configuration:  
      - Iterates over groups (clients), then over their sub_groups (projects), building a simplified list of entries.  
      - Outputs a flattened array of JSON objects representing project-level time entries with client linkage.  
    - Inputs: From Get Toggle Summary.  
    - Outputs: To Merge data node.  
    - Edge Cases:  
      - Missing or malformed `groups` or `sub_groups` in the response may cause empty or incomplete arrays.  
      - Defensive coding ensures defaults and type conversions.  

  - **Merge data**  
    - Type: Merge  
    - Role: Combines the processed project summary data with the full project list from Get Toggle Projects to enrich entries by matching on project_id.  
    - Configuration:  
      - Mode: Combine  
      - Join Mode: Enrich Input 1 (the project summary data enriched with project metadata).  
      - Merge by fields: project_id from summary data and id from projects list.  
    - Inputs:  
      - Primary: Output from Prepare projects list (summary data).  
      - Secondary: Output from Get Toggle Projects (projects list).  
    - Outputs: To HTML report preparation node.  
    - Edge Cases:  
      - Projects present in summary but missing in projects list or vice versa could result in incomplete merges.  
      - Duplicate project IDs could cause unexpected merges.  

- **Sticky Notes:**  
  - "Merge data to associate projects with clients."  

---

#### 1.4 HTML Report Generation

- **Overview:**  
  Generates a structured HTML report summarizing hours tracked per client and per project, sorted by total hours descending.

- **Nodes Involved:**  
  - Prepare html raport (Code)  

- **Node Details:**

  - **Prepare html raport**  
    - Type: Code (JavaScript)  
    - Role: Aggregates and formats the merged data into an HTML string and email subject line.  
    - Configuration:  
      - Groups records by client, sums project seconds to total client seconds.  
      - Sorts clients by total hours descending.  
      - Sorts projects under each client by tracked time descending.  
      - Converts seconds to hours with two decimal places.  
      - Constructs an HTML email body with header, client sections with projects, and total hours.  
      - Uses the period label "Previous month" dynamically.  
    - Inputs: From Merge data node.  
    - Outputs: JSON with `subject` and `html` properties for the email.  
    - Edge Cases:  
      - Missing client or project names default to placeholders.  
      - Zero or missing time entries handled gracefully.  
      - Large datasets could impact performance.  

- **Sticky Notes:**  
  - "Generate an HTML report with hours per client and per project."  

---

#### 1.5 Email Dispatch via Resend

- **Overview:**  
  Sends the generated HTML report by email through the Resend API.

- **Nodes Involved:**  
  - Send email via Resend  

- **Node Details:**

  - **Send email via Resend**  
    - Type: HTTP Request  
    - Role: Posts the email data to the Resend API endpoint to send the report email.  
    - Configuration:  
      - URL: `https://api.resend.com/emails`  
      - Method: POST  
      - Headers:  
        - Authorization: Bearer token (user must enter valid RESEND_API_KEY).  
        - Content-Type: application/json  
      - Body (JSON):  
        - `from`: sender email (e.g., `<noreplay@yourdomain.com>`)  
        - `to`: array of recipient emails (e.g., `["to@yourdomain.com"]`)  
        - `subject`: email subject from previous node  
        - `html`: email body HTML from previous node  
    - Inputs: From Prepare html raport node.  
    - Outputs: None (final node).  
    - Edge Cases:  
      - Invalid or missing API key leads to 401 Unauthorized.  
      - Invalid email addresses or malformed JSON causes request failure.  
      - Network issues or Resend API downtime.  
    - Sticky Notes:  
      - Instructions to enter RESEND_API_KEY, FROM and TO addresses.  

---

#### Supporting Documentation Nodes

- **Sticky Note1**: Explains fetching Toggl clients, projects, and summary data.  
- **Sticky Note3**: Notes about data merging to associate projects with clients.  
- **Sticky Note4**: Notes about generating the HTML report.  
- **Sticky Note5**: Notes about monthly schedule automation.  
- **Sticky Note (near Send email)**: Instructions for sending report via Resend.  
- **README**: Detailed description, requirements, customization tips, documentation links, and author credit.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                        | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                  |
|---------------------|--------------------|-------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger   | Monthly automation trigger            | —                          | Get Toggle Summary, Get Toggle Projects | Schedule automation (monthly)                                                                |
| Get Toggle Projects  | HTTP Request       | Fetch Toggl projects list             | Schedule Trigger            | Merge data                  | Fetch Toggl clients, projects, and summary data form last month via HTTP nodes               |
| Get Toggle Summary   | HTTP Request       | Fetch Toggl time summary (clients/projects) | Schedule Trigger            | Prepare projects list       | Fetch Toggl clients, projects, and summary data form last month via HTTP nodes               |
| Prepare projects list| Code               | Extract project-level time entries    | Get Toggle Summary          | Merge data                  |                                                                                              |
| Merge data           | Merge              | Combine project summary with project details | Prepare projects list, Get Toggle Projects | Prepare html raport           | Merge data to associate projects with clients.                                              |
| Prepare html raport  | Code               | Generate HTML report from merged data | Merge data                  | Send email via Resend       | Generate an HTML report with hours per client and per project                               |
| Send email via Resend| HTTP Request       | Send summary report email via Resend | Prepare html raport         | —                           | Send the report via Resend email API. 1. Enter RESEND_API_KEY 2. Enter FROM and TO email addresses |
| Sticky Note1         | Sticky Note        | Documentation and instructions       | —                          | —                           | Fetch Toggl clients, projects, and summary data form last month via HTTP nodes 1. Enter your TOGGL_WORKSPACE_ID |
| Sticky Note3         | Sticky Note        | Documentation                       | —                          | —                           | Merge data to associate projects with clients.                                              |
| Sticky Note4         | Sticky Note        | Documentation                       | —                          | —                           | Generate an HTML report with hours per client and per project                               |
| Sticky Note5         | Sticky Note        | Documentation                       | —                          | —                           | Schedule automation (monthly)                                                                |
| README              | Sticky Note        | Detailed workflow description and instructions | —                          | —                           | See detailed README content above                                                           |
| Sticky Note (near Send email) | Sticky Note        | Documentation                       | —                          | —                           | Send the report via Resend email API. 1. Enter RESEND_API_KEY 2. Enter FROM and TO email addresses |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run monthly, trigger hour set to 4 AM.

2. **Create HTTP Request node “Get Toggle Summary”:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.track.toggl.com/reports/api/v3/workspace/TOGGL_WORKSPACE_ID/summary/time_entries` (replace `TOGGL_WORKSPACE_ID`)  
   - Authentication: Use Toggl API credentials (API token or OAuth).  
   - Headers: Content-Type: application/json  
   - Body Parameters (JSON):  
     ```json
     {
       "start_date": "={{ $now.minus({ months: 1 }).startOf('month').toFormat('yyyy-MM-dd') }}",
       "end_date": "={{ $now.minus({ months: 1 }).endOf('month').toFormat('yyyy-MM-dd') }}",
       "grouping": "clients",
       "sub_grouping": "projects"
     }
     ```
   - Connect Schedule Trigger output to this node.

3. **Create HTTP Request node “Get Toggle Projects”:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.track.toggl.com/api/v9/workspaces/TOGGL_WORKSPACE_ID/projects` (replace `TOGGL_WORKSPACE_ID`)  
   - Authentication: Same Toggl API credentials as above.  
   - Enable pagination with page parameter increment.  
   - Connect Schedule Trigger output to this node as well.

4. **Create Code node “Prepare projects list”:**  
   - Type: Code (JavaScript)  
   - Input: From “Get Toggle Summary” node.  
   - Code:  
     ```javascript
     const groups = $json.groups || [];
     const result = [];

     for (const g of groups) {
       const clientId = Number(g.id);
       const subs = g.sub_groups || [];
       for (const s of subs) {
         result.push({
           json: {
             client_id: clientId,
             project_id: Number(s.id),
             seconds: Number(s.seconds) || 0
           }
         });
       }
     }

     return result;
     ```
5. **Create Merge node “Merge data”:**  
   - Mode: Combine  
   - Join Mode: Enrich Input 1  
   - Merge by fields: `project_id` (from Prepare projects list) and `id` (from Get Toggle Projects)  
   - Inputs:  
     - Primary: “Prepare projects list” output  
     - Secondary: “Get Toggle Projects” output  

6. **Create Code node “Prepare html raport”:**  
   - Input: From “Merge data” node.  
   - Code:  
     ```javascript
     const data = $input.all().map(i => i.json);
     const period = $json.period || "Previous month";

     // Group by client
     const clients = {};
     for (const i of data) {
       const clientId = i.client_id || i.cid;
       const clientName = i.client_name || "(missing customer name)";
       if (!clients[clientId]) {
         clients[clientId] = { name: clientName, projects: [] };
       }
       clients[clientId].projects.push({
         project_id: i.project_id,
         project_name: i.name || "(bez nazwy projektu)",
         seconds: Number(i.seconds) || 0
       });
     }

     // Sum clients time
     for (const c of Object.values(clients)) {
       c.totalSeconds = c.projects.reduce((a, p) => a + p.seconds, 0);
     }

     // Sort clients
     const sortedClients = Object.entries(clients)
       .sort(([, a], [, b]) => b.totalSeconds - a.totalSeconds);

     // Generate html report
     let totalSeconds = 0;
     let html = "<html><body>";
     html += "<h3>Time Report per Client and Project - " + period + "</h3>";

     for (const [clientId, c] of sortedClients) {
       const clientHours = (c.totalSeconds / 3600).toFixed(2);
       totalSeconds += c.totalSeconds;

       html += "<p><strong>" + c.name + "</strong> – " + clientHours + " h</p>";
       html += "<ul>";

       const sortedProjects = c.projects.sort((a, b) => b.seconds - a.seconds);
       for (const p of sortedProjects) {
         const hours = (p.seconds / 3600).toFixed(2);
         html += "<li>" + p.project_name + " – " + hours + " h</li>";
       }

       html += "</ul>";
     }

     html += "<p><strong>Total hours: " + (totalSeconds / 3600).toFixed(2) + "</strong></p>";
     html += "</body></html>";

     return [{
       json: {
         subject: "Time Report per Client and Project - " + period,
         html
       }
     }];
     ```
7. **Create HTTP Request node “Send email via Resend”:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.resend.com/emails`  
   - Headers:  
     - Authorization: Bearer `YOUR_RESEND_API_KEY` (replace with actual API key)  
     - Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "from": "<noreplay@yourdomain.com>",
       "to": ["to@yourdomain.com"],
       "subject": "={{$json.subject}}",
       "html": "={{$json.html}}"
     }
     ```
   - Input: Connect from “Prepare html raport” node.

8. **Optional: Add Sticky Notes nodes** to document each part for user instructions, especially to:  
   - Remind about entering TOGGL_WORKSPACE_ID.  
   - Explain data merging.  
   - Explain report generation.  
   - Explain Resend API key and email configuration.  
   - Provide detailed README content with links and author info.

9. **Credentials Setup:**  
   - Create and assign Toggl API credentials with your Toggl API token.  
   - Create and assign Resend API key with your Resend.com API key.

10. **Activate the workflow** to enable monthly scheduling.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Toggl free account required; sign up at https://accounts.toggl.com/track/signup/                            | Toggl account setup                                                                              |
| Resend.com free account required; API docs at https://resend.com/docs/                                    | Resend email sending service                                                                     |
| Workflow author: Krystian Syryca - https://krsweb.pl                                                      | Author and source                                                                                |
| Workflow description and customization instructions included in the README sticky note                    | Internal workflow documentation                                                                  |
| Pagination used in Get Toggle Projects HTTP node to handle large project lists                             | Toggl API pagination info                                                                        |
| Email sending requires valid FROM and TO email addresses configured in the HTTP Request body              | Resend API email sending prerequisites                                                          |
| Period calculation uses dynamic expressions to get the previous calendar month start and end dates        | Date handling with n8n expressions                                                               |
| Archived projects can be included by extending Toggl project API calls with `active=false&archived=true`  | Customization tip for including archived projects                                                |

---

**Disclaimer:**  
The text above is exclusively derived from an automated workflow built with n8n, adhering strictly to current content policies and containing no illegal, offensive, or protected elements. All processed data is lawful and publicly accessible.