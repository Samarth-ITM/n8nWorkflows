Automated Editorial Task Tracking & Notifications with Motion and Airtable

https://n8nworkflows.xyz/workflows/automated-editorial-task-tracking---notifications-with-motion-and-airtable-7383


# Automated Editorial Task Tracking & Notifications with Motion and Airtable

### 1. Workflow Overview

This workflow automates editorial task tracking and notifications by integrating Airtable project data with Motion task management. It targets project managers, team leads, and agencies who want to monitor editorial project milestones, especially for SEO-related content, and send email alerts upon task completion. The workflow logically divides into these blocks:

- **1.1 Scheduled Trigger:** Periodic initiation of the workflow based on calendar schedule.
- **1.2 Airtable Data Retrieval:** Fetch project records from Airtable to identify active editorial projects.
- **1.3 Active Project Filtering:** Filter projects marked as active and not recently notified.
- **1.4 Batch Processing:** Process projects one by one in batches for scalability.
- **1.5 Motion Projects Retrieval:** Call Motion API to get detailed project data using workspace IDs.
- **1.6 Project Data Transformation:** Parse and flatten Motion API response to usable project items.
- **1.7 Project Filtering for Editorial Tasks:** Filter projects with status "Todo" and names containing "SEO".
- **1.8 Task Completion Check:** Query Motion API for a specific task ("Intégrer les articles de blog") marked as completed.
- **1.9 Conditional Email Notification:** Send notification emails only if the task is completed.
- **1.10 Airtable Update:** Update Airtable with the last notification sent timestamp.
- **1.11 Workflow Looping and Continuation:** Loop handling and sequential processing of all projects.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Starts the workflow automatically every day between the 10th and 31st of each month at 8 AM.
- **Nodes Involved:** Schedule Trigger
- **Node Details:**
  - Type: Schedule Trigger (n8n built-in)
  - Configured with cron expression: `0 8 10-31 * *` (8:00 AM, days 10 to 31 monthly)
  - No inputs; triggers workflow start
  - Output connects to Airtable data retrieval node
  - Edge cases: Workflow will not run outside specified days; cron misconfiguration can cause missed runs.

#### 2.2 Airtable Data Retrieval

- **Overview:** Retrieves projects from an Airtable base to get project metadata including Motion workspace IDs.
- **Nodes Involved:** Récupération infos airtable
- **Node Details:**
  - Type: Airtable node (API v2.1)
  - Operation: Search/list on table "Projets" in base "appk6h4dLarLUXKpR"
  - Credentials: Airtable Personal Access Token
  - Parameters: Default search, no filters applied here
  - Outputs: Array of project records
  - Input: Trigger from Schedule Trigger
  - Output: Connects to Active Project Filtering node
  - Edge cases: API rate limits, invalid credentials, empty tables

#### 2.3 Active Project Filtering

- **Overview:** Filters projects that are active and have not been sent notifications after the start of the current month.
- **Nodes Involved:** Actif (If node)
- **Node Details:**
  - Type: If node (conditional)
  - Conditions:
    - Status field `Status - Calendrier éditorial` equals "Actif"
    - Field `Last sent - Calendrier éditorial` is before the first day of current month
  - Inputs: Project records from Airtable
  - Outputs: True branch continues to batch processing; false branch discards projects
  - Edge cases: Missing fields cause filtering to fail; date parsing errors

#### 2.4 Batch Processing

- **Overview:** Splits project list into individual items to process one at a time.
- **Nodes Involved:** Loop Over Items (SplitInBatches)
- **Node Details:**
  - Type: SplitInBatches
  - Default batch size (not explicitly set, default 1)
  - Input: Filtered projects from Actif node
  - Outputs:
    - Main output: Project item to Motion Projects Retrieval
    - Second output: Empty for loop continuation
  - Edge cases: Large datasets slow down processing; batch size tuning needed

#### 2.5 Motion Projects Retrieval

- **Overview:** Requests detailed project information from Motion API for the given workspace ID.
- **Nodes Involved:** Récupération des projets (HTTP Request)
- **Node Details:**
  - Type: HTTP Request
  - Method: GET to `https://api.usemotion.com/v1/projects`
  - Query parameter: `workspaceId` from current project JSON
  - Authentication: HTTP Header Auth with Motion API token
  - Executes once per item (batch)
  - Output: Motion API response with project details
  - Edge cases: API downtime, invalid token, workspace ID missing or incorrect

#### 2.6 Project Data Transformation

- **Overview:** Parses the Motion API response, extracting the projects array and mapping each project into individual items.
- **Nodes Involved:** Tri des projets (Code node)
- **Node Details:**
  - Type: Code (JavaScript)
  - Logic:
    - Checks if response contains `.projects` array
    - Maps each project object to a new item with project data and metadata
  - Input: JSON response from Motion API
  - Output: Array of project items
  - Edge cases: Malformed API response, empty projects array

#### 2.7 Project Filtering for Editorial Tasks

- **Overview:** Filters projects with status "Todo" and names containing "SEO" to focus on editorial SEO tasks.
- **Nodes Involved:** Todo (Filter node)
- **Node Details:**
  - Type: Filter node
  - Conditions:
    - `status.name` equals `Todo`
    - `name` contains `SEO`
  - Input: Projects from transformation node
  - Output: Projects matching editorial tasks
  - Edge cases: Case sensitivity in string matching, missing fields

#### 2.8 Task Completion Check

- **Overview:** Queries Motion API for tasks within the project filtered by specific task name and completion status.
- **Nodes Involved:** HTTP Request (second HTTP Request node)
- **Node Details:**
  - Type: HTTP Request
  - Endpoint: `https://api.usemotion.com/v1/tasks`
  - Query parameters:
    - `projectId` from current project ID
    - `name` = "Intégrer les articles de blog"
    - `status` = "completed"
  - Authentication: Motion API token in header
  - Output: Tasks matching the criteria
  - Edge cases: No matching tasks, API errors

#### 2.9 Conditional Email Notification

- **Overview:** Sends an email via Gmail only if the targeted task is marked as "Completed".
- **Nodes Involved:** If1 (If node), Send a message (Gmail node)
- **Node Details:**
  - If1:
    - Checks if first task in tasks array has status "Completed"
    - True branch sends email; false branch loops back for next item
  - Send a message:
    - Type: Gmail node (OAuth2)
    - Recipients: Project manager email, client emails, collaborators (all extracted dynamically from current project data)
    - Subject: Includes project name and current month name dynamically
    - Message: HTML formatted email referencing editorial calendar link
    - Sender name set as "Institut du Référencement"
  - Edge cases:
    - Missing email addresses cause failure
    - Gmail authentication errors
    - Email rate limits

#### 2.10 Airtable Update

- **Overview:** Updates the Airtable record for the project with the current date as last notification sent timestamp.
- **Nodes Involved:** Airtable (Update node)
- **Node Details:**
  - Operation: Update project record in Airtable by matching record ID
  - Updates fields:
    - `Budget - Accès` set to 0 (possibly a reset or placeholder)
    - `Last sent - Calendrier éditorial` set to current date formatted as M/d/yyyy
  - Credentials: Airtable Personal Access Token
  - Input: After sending email
  - Output: Loops back to Loop Over Items node for next project
  - Edge cases: Airtable API limits, record not found, permission errors

#### 2.11 Workflow Looping and Continuation

- **Overview:** Ensures continuous processing of all projects sequentially until completion.
- **Nodes Involved:** Loop Over Items (second output branch)
- **Node Details:**
  - The second output branch of Loop Over Items is connected to Motion Projects Retrieval, enabling continuous iteration.
  - Handles flow control for batch processing.
  - Edge cases: Infinite loops if not configured properly, batch size misconfiguration

---

### 3. Summary Table

| Node Name                 | Node Type             | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                             |
|---------------------------|-----------------------|----------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger      | Initiates workflow on schedule         | -                           | Récupération infos airtable |                                                                                                       |
| Récupération infos airtable| Airtable              | Retrieves projects from Airtable       | Schedule Trigger             | Actif                       |                                                                                                       |
| Actif                    | If                    | Filters active projects for notification| Récupération infos airtable | Loop Over Items              |                                                                                                       |
| Loop Over Items          | SplitInBatches        | Processes projects one by one           | Actif                       | Récupération des projets, (loop continuation) |                                                                                                       |
| Récupération des projets  | HTTP Request          | Fetches projects from Motion API       | Loop Over Items             | Tri des projets             |                                                                                                       |
| Tri des projets           | Code                  | Parses & maps Motion API project data  | Récupération des projets    | Todo                        |                                                                                                       |
| Todo                      | Filter                | Filters projects by status and name    | Tri des projets             | HTTP Request                |                                                                                                       |
| HTTP Request             | HTTP Request          | Queries Motion API for completed tasks | Todo                        | If1                        |                                                                                                       |
| If1                       | If                    | Checks if targeted task is completed   | HTTP Request                | Send a message, Loop Over Items |                                                                                                   |
| Send a message            | Gmail                 | Sends notification emails              | If1                        | Airtable                    |                                                                                                       |
| Airtable                  | Airtable              | Updates Airtable with notification date| Send a message              | Loop Over Items             |                                                                                                       |
| Sticky Note               | Sticky Note           | Documentation and overview             | -                           | -                           | # Automated project status tracking with Airtable and Motion... (Full detailed documentation included) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**
   - Type: Schedule Trigger
   - Configure cron expression `0 8 10-31 * *` to run daily at 8 AM from the 10th to 31st.
   - Connect output to Airtable node.

2. **Create Airtable node for project retrieval:**
   - Type: Airtable
   - Operation: Search/List
   - Base: Select your Airtable base (e.g., `appk6h4dLarLUXKpR`)
   - Table: Select "Projets"
   - Credentials: Use Airtable Personal Access Token
   - Connect output to If node for active projects filtering.

3. **Create If node "Actif" for filtering active projects:**
   - Set conditions:
     - `Status - Calendrier éditorial` equals "Actif"
     - `Last sent - Calendrier éditorial` date is before first day of current month (use expression to get first day)
   - True output connects to SplitInBatches node.

4. **Add SplitInBatches node "Loop Over Items":**
   - Default batch size 1
   - Input: from If node True branch
   - Connect main output to HTTP Request node for Motion API projects retrieval
   - Connect second output for loop continuation to same HTTP Request node (to continue processing batches)

5. **Create HTTP Request node "Récupération des projets":**
   - Method: GET
   - URL: `https://api.usemotion.com/v1/projects`
   - Query parameter: `workspaceId` from current item JSON (`{{$json["Motion Workspace ID"]}}`)
   - Credentials: HTTP Header Auth with Motion API token
   - Connect output to Code node.

6. **Add Code node "Tri des projets":**
   - JavaScript code to check for `.projects` array and map each project to individual items.
   - Connect output to Filter node "Todo".

7. **Create Filter node "Todo":**
   - Conditions:
     - `status.name` equals "Todo"
     - `name` contains "SEO"
   - Output to HTTP Request node for task check.

8. **Add HTTP Request node for task completion check:**
   - Method: GET
   - URL: `https://api.usemotion.com/v1/tasks`
   - Query parameters:
     - `projectId` from current project id
     - `name` = "Intégrer les articles de blog"
     - `status` = "completed"
   - Credentials: Motion API token
   - Connect output to If node "If1".

9. **Create If node "If1":**
   - Condition: Check if `tasks[0].status.name` equals "Completed"
   - True output to Gmail node for sending email
   - False output back to SplitInBatches node's second output for next iteration

10. **Add Gmail node "Send a message":**
    - Configure OAuth2 Gmail credentials
    - Recipients: Compose dynamically from project manager email, client emails, collaborators
    - Subject: Include project name and current month dynamically
    - Message: HTML template referencing editorial calendar link and professional signature
    - Connect output to Airtable update node.

11. **Create Airtable node for update:**
    - Operation: Update record by ID
    - Base and Table same as retrieval
    - Set fields:
      - `Budget - Accès` to 0
      - `Last sent - Calendrier éditorial` to current date formatted M/d/yyyy
    - Credentials: Airtable Personal Access Token
    - Connect output back to SplitInBatches node second output for continuous processing

12. **Add Sticky Note for documentation (optional):**
    - Include overview, instructions, and usage notes as provided.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Automated project status tracking with Airtable and Motion. Designed for project managers and agencies to track editorial milestones, send notifications, and synchronize data across Airtable and Motion task management. Includes scheduling, filtering, API integrations, and conditional email notifications. Requirements include Airtable, Motion API access, and Gmail OAuth2. Customizable for multiple projects and notification types. Supports monthly email notifications for SEO editorial projects with task completion checks. | Full workflow description is documented as a comprehensive sticky note within the workflow itself, accessible in the n8n editor.                                                                                                                                                                                                                                                                                                                                                                                               |
| Motion API documentation | https://docs.usemotion.com/reference/api-overview (for API endpoint reference and authentication setup)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| Airtable API documentation | https://airtable.com/api (for base and table configuration, API tokens)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| Gmail OAuth2 setup guide | https://developers.google.com/gmail/api/quickstart/nodejs (to configure OAuth2 credentials for sending emails)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. No illegal, offensive, or protected elements are included. All processed data is legal and public.