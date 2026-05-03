Daily Task Reminder System: Airtable to Slack Automated Notifications

https://n8nworkflows.xyz/workflows/daily-task-reminder-system--airtable-to-slack-automated-notifications-5672


# Daily Task Reminder System: Airtable to Slack Automated Notifications

# Daily Task Reminder System: Airtable to Slack Automated Notifications

---

### 1. Workflow Overview

This workflow automates daily task reminders by fetching active tasks from an Airtable base and sending notification messages to a designated Slack channel every morning at 9 a.m. The key use case is to eliminate manual follow-ups on pending work by automatically notifying team members of their outstanding tasks.

The workflow is logically divided into three functional blocks:

- **1.1 Schedule Trigger:** Initiates the workflow execution on a daily schedule at a fixed time (9:00 a.m.).
- **1.2 Airtable Data Retrieval:** Connects to Airtable, queries for tasks marked as “In Progress,” and retrieves relevant task details.
- **1.3 Slack Notification:** Iterates through the fetched tasks and sends formatted reminder messages to a specific Slack channel.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Schedule Trigger

- **Overview:** This block triggers the entire workflow automatically every day at 9 a.m., serving as the entry point.
- **Nodes Involved:**  
  - Schedule Trigger

##### Node: Schedule Trigger

- **Type and Role:** `n8n-nodes-base.scheduleTrigger` — Initiates the workflow on a scheduled basis.
- **Configuration:**  
  - Interval: Daily  
  - Trigger time: 9:00 a.m. (hour = 9, minute = 0)  
  - Runs once per day at the set hour.
- **Expressions/Variables:** None.
- **Input Connections:** None (trigger node).
- **Output Connections:** Connected to the "Search records" node.
- **Version Requirements:** Compatible with n8n version supporting `scheduleTrigger` v1.2 or above.
- **Potential Failures:**  
  - Timezone misconfiguration may cause unexpected trigger times.  
  - n8n instance downtime could prevent scheduled execution.
- **Sub-workflow:** None.

---

#### 1.2 Airtable Data Retrieval

- **Overview:** Queries the Airtable base “Tâches” for tasks currently marked with the status “En cours” (In Progress). Retrieves all matching records.
- **Nodes Involved:**  
  - Search records (Airtable node)
  
##### Node: Search records

- **Type and Role:** `n8n-nodes-base.airtable` — Fetches records from Airtable according to a filter.
- **Configuration:**  
  - Base: "Tâches" (Airtable base identified by appPk6zfIQ1EbBq43)  
  - Table: "Produits" (table id tblLkuoQ9AYsAQ0io)  
  - Operation: Search  
  - Filter formula: `{Statut} = "En cours"` — Filters to only return tasks with status “In Progress”  
  - Return all matching records: Enabled  
  - Output format: Simple (flattened JSON structure)  
- **Credentials:** Uses an Airtable Personal Access Token credential configured with read access to the specific base.
- **Expressions/Variables:** Uses Airtable formula syntax directly in the filter.
- **Input Connections:** Receives trigger from the Schedule Trigger node.
- **Output Connections:** Feeds output into the Slack message node.
- **Version Requirements:** Compatible with Airtable node v2.1 or newer.
- **Potential Failures:**  
  - Authentication errors if token is invalid or expired.  
  - API rate limits or Airtable service downtime.  
  - Incorrect filter formula syntax causing no results or errors.  
  - Empty results if no tasks match the criteria.
- **Sub-workflow:** None.

---

#### 1.3 Slack Notification

- **Overview:** Sends a formatted message for each active task retrieved from Airtable into a specific Slack channel, effectively notifying users of their pending tasks.
- **Nodes Involved:**  
  - Send a message (Slack node)

##### Node: Send a message

- **Type and Role:** `n8n-nodes-base.slack` — Sends messages to Slack channels using OAuth2 authentication.
- **Configuration:**  
  - Operation: Send message  
  - Message destination: Channel (selected from a list)  
  - Channel: "tous-n8n" (channel ID: C0945G1M3T2)  
  - Message text template:  
    ```
    New task for {{ $json.name }}: *{{ $json["Titre"] }}* 👉 Deadline: {{ $json["Date limite"] }}
    ```
    This uses expressions to dynamically insert task owner (`name`), task title (`Titre`), and deadline (`Date limite`) from the Airtable record.
  - Authentication: OAuth2 (Slack Bot credentials)
- **Credentials:** Uses OAuth2 credentials linked to a Slack workspace and bot token with permission to post messages.
- **Expressions/Variables:** Uses n8n expression syntax to dynamically populate messages.
- **Input Connections:** Connected to the output of the Airtable "Search records" node, processing each task record.
- **Output Connections:** None (end of workflow chain).
- **Version Requirements:** Compatible with Slack node v2.3 or newer.
- **Potential Failures:**  
  - Slack API authorization failures due to revoked or expired tokens.  
  - Posting errors if the bot lacks permission to post in the designated channel.  
  - Message formatting errors if expected fields are missing in the Airtable data.  
  - Rate limiting if too many messages sent simultaneously.
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name       | Node Type                    | Functional Role           | Input Node(s)     | Output Node(s) | Sticky Note                                                                                                                           |
|-----------------|------------------------------|--------------------------|-------------------|----------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| scheduleTrigger               | Initiates workflow daily | None              | Search records | Still reminding people about their tasks manually every morning? Let Slack do it automatically, daily at 9 a.m. See detailed instructions in Sticky Note1. |
| Search records  | airtable                     | Retrieves active tasks    | Schedule Trigger  | Send a message | See detailed setup instructions for Airtable token creation and node configuration in Sticky Note1.                                     |
| Send a message  | slack                        | Sends Slack reminders    | Search records    | None           | Use Slack OAuth2 credentials; message template example and Slack channel setup explained in Sticky Note1.                              |
| Sticky Note1    | stickyNote                   | Documentation and guide  | None              | None           | Contains full step-by-step workflow explanation, including Airtable and Slack setup instructions, and workflow logic overview.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Parameters:  
     - Interval: Days  
     - Days Between Triggers: 1  
     - Trigger At Hour: 9  
     - Trigger At Minute: 0  
   - Position: (example: x=500, y=1000)

3. **Add Airtable node:**  
   - Type: Airtable  
   - Name: "Search records"  
   - Credentials: Create new Airtable Personal Access Token credential:  
     - Go to https://airtable.com/create/tokens  
     - Name token (e.g., "TACHES")  
     - Scope: Enable `data.records:read`  
     - Access: Limit to your specific base (e.g., “Tâches”)  
     - Save and copy the token  
     - In n8n, create new credential and paste token  
   - Parameters:  
     - Base: Select your base ("Tâches")  
     - Table: Select your table (e.g., "Produits")  
     - Operation: Search  
     - Filter Formula: `{Statut} = "En cours"`  
     - Return All: Enabled  
     - Output Format: Simple  
   - Position: (e.g., x=620, y=1960)  
   - Connect Schedule Trigger node output to this node’s input.

4. **Add Slack node:**  
   - Type: Slack  
   - Name: "Send a message"  
   - Credentials: Create new Slack OAuth2 credential:  
     - Create a Slack App in your workspace with bot token and appropriate scopes (chat:write)  
     - In n8n, add new Slack OAuth2 credential and authenticate via OAuth2 flow  
   - Parameters:  
     - Operation: Send message  
     - Send Message To: Channel  
     - Channel: Select your Slack channel (e.g., #tous-n8n)  
     - Message Text:  
       ```
       New task for {{ $json.name }}: *{{ $json["Titre"] }}* 👉 Deadline: {{ $json["Date limite"] }}
       ```  
     - Authentication: OAuth2  
   - Position: (e.g., x=680, y=3380)  
   - Connect Airtable node output to this Slack node input.

5. **Save and activate the workflow.**

6. **Test the workflow:**  
   - Execute the Schedule Trigger manually or wait for scheduled trigger.  
   - Verify Airtable node returns tasks with status “En cours.”  
   - Confirm Slack messages are posted correctly in the specified channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Detailed setup instructions and rationale for the workflow are included in the Sticky Note node within the workflow canvas.                        | See "Sticky Note1" node content.                                                                                        |
| Airtable Personal Access Token creation guide: https://airtable.com/create/tokens                                                                   | Required to configure Airtable credentials securely.                                                                    |
| Slack App creation and OAuth2 token setup needed to send messages: https://api.slack.com/authentication/oauth-v2                                 | Ensure bot token has chat:write permission and is installed in target Slack workspace.                                  |
| Recommended field schema in Airtable table: Title (Text), Assignee (Text), Email (Email), Status (Single select), Due Date (Date, format dd/mm/yyyy) | Consistency in field names and types is essential for correct filtering and message formatting.                        |
| Use of n8n expressions (e.g., `{{ $json["Titre"] }}`) to dynamically populate Slack messages from Airtable data.                                  | Expression syntax: https://docs.n8n.io/nodes/expressions/                                                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies with all relevant content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.