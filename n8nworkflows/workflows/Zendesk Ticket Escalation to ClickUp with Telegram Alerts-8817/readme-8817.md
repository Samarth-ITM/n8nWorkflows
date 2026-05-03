Zendesk Ticket Escalation to ClickUp with Telegram Alerts

https://n8nworkflows.xyz/workflows/zendesk-ticket-escalation-to-clickup-with-telegram-alerts-8817


# Zendesk Ticket Escalation to ClickUp with Telegram Alerts

---

### 1. Workflow Overview

This workflow automates the escalation process of Zendesk support tickets by creating corresponding tasks in ClickUp and sending immediate alerts via Telegram. It is designed for support teams and managers who need real-time visibility and tracking of urgent tickets requiring escalation. The workflow is logically organized into the following blocks:

- **1.1 Input Reception:** Manual Trigger to start the workflow.
- **1.2 Zendesk Ticket Retrieval:** Fetches all pending tickets from a specific Zendesk group.
- **1.3 Ticket Selection:** Identifies the most recent Zendesk ticket for escalation.
- **1.4 Requester Data Enrichment:** Retrieves detailed information about the ticket requester.
- **1.5 Data Consolidation:** Merges ticket and requester data into a unified dataset.
- **1.6 ClickUp Task Preparation:** Formats the consolidated data into a task payload.
- **1.7 ClickUp Task Creation:** Creates a new task in ClickUp with escalation details.
- **1.8 Telegram Alert Formatting:** Builds a concise escalation alert message.
- **1.9 Telegram Notification:** Sends the alert message to a designated Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Provides a manual trigger to start the escalation workflow on demand.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’
- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Configuration:** No parameters; triggers workflow execution manually.  
  - **Inputs:** None  
  - **Outputs:** Connects to "Fetch Zendesk Tickets" node.  
  - **Edge Cases:** None; manual initiation prevents unintended automatic runs.

#### 2.2 Zendesk Ticket Retrieval

- **Overview:** Retrieves all Zendesk tickets with status "pending" from a specific group, sorted by status descending.
- **Nodes Involved:**  
  - Fetch Zendesk Tickets
- **Node Details:**  
  - **Type:** Zendesk Node (Get All Tickets)  
  - **Configuration:**  
    - Group ID fixed to 22337660284956 to scope the tickets.  
    - Status filter set to "pending" to focus on active tickets.  
    - Sorted by status descending (most urgent first).  
  - **Credentials:** Zendesk API credentials named "Zendesk account vivek".  
  - **Inputs:** Triggered by manual node.  
  - **Outputs:** Ticket list passed downstream to "Select Latest Ticket".  
  - **Edge Cases:**  
    - API rate limits or authentication errors.  
    - Empty ticket list (no pending tickets).  
  - **Sticky Note:** Explains purpose — retrieves active tickets for escalation.

#### 2.3 Ticket Selection

- **Overview:** Selects the single most recent ticket from the retrieved list to focus escalation efforts.
- **Nodes Involved:**  
  - Select Latest Ticket
- **Node Details:**  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Sorts tickets by `created_at` descending.  
    - Returns only the latest ticket with key fields: id, subject, description, requester_id, created_at.  
  - **Inputs:** Receives array of tickets from previous node.  
  - **Outputs:** Sends one ticket object downstream.  
  - **Edge Cases:**  
    - Empty input array could cause undefined access. Handling not explicitly coded.  
  - **Sticky Note:** Describes filtering and sorting rationale.

#### 2.4 Requester Data Enrichment

- **Overview:** Retrieves detailed user information for the requester of the selected ticket.
- **Nodes Involved:**  
  - Fetch Requester Email
- **Node Details:**  
  - **Type:** Zendesk Node (Get User)  
  - **Configuration:**  
    - Uses dynamic expression for user ID: `={{ $json.requester_id }}`.  
  - **Credentials:** Same Zendesk API credentials.  
  - **Inputs:** Receives latest ticket data with requester ID.  
  - **Outputs:** Provides user data downstream.  
  - **Edge Cases:**  
    - Invalid requester ID or API issues returning no user data.  
  - **Sticky Note:** Notes importance of retrieving requester’s name, email, timezone.

#### 2.5 Data Consolidation

- **Overview:** Merges the ticket and requester information into a single enriched dataset for task creation.
- **Nodes Involved:**  
  - Merge Ticket & Requester Data
- **Node Details:**  
  - **Type:** Merge Node  
  - **Configuration:** Default merge with inputs: requester data and ticket data on separate inputs.  
  - **Inputs:**  
    - Input 1: Requester data from "Fetch Requester Email"  
    - Input 2: Ticket data from "Select Latest Ticket"  
  - **Outputs:** Combined data passed to "Prepare ClickUp Task Payload".  
  - **Edge Cases:**  
    - Mismatch or missing data on either input could cause incomplete merges.  
  - **Sticky Note:** Explains the purpose of consolidating metadata for downstream nodes.

#### 2.6 ClickUp Task Preparation

- **Overview:** Formats the merged data into a comprehensive ClickUp task payload including task title, description, priority, and tags.
- **Nodes Involved:**  
  - Prepare ClickUp Task Payload
- **Node Details:**  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Extracts ticket and requester info from merged input.  
    - Constructs task name with "[Escalation]" prefix and ticket subject/ID.  
    - Builds detailed description including escalation message, ticket info, requester info, and a clickable ticket URL.  
    - Sets priority to 3 (default) and tags "zendesk" and "escalation".  
  - **Inputs:** Receives merged ticket/requester JSON.  
  - **Outputs:** Task payload sent to "Create a task".  
  - **Edge Cases:**  
    - Missing fields in input JSON may cause incomplete task descriptions.  
  - **Sticky Note:** Details task structure and escalation context added.

#### 2.7 ClickUp Task Creation

- **Overview:** Creates a new task in ClickUp with the prepared payload, assigning it to a specific user and setting a due date.
- **Nodes Involved:**  
  - Create a task
- **Node Details:**  
  - **Type:** ClickUp Node (Create Task)  
  - **Configuration:**  
    - Team, Space, and List IDs hardcoded to specific values.  
    - Task name dynamically set from input.  
    - Additional fields include description, due date set 7 days ahead, and assignee ID.  
    - Folderless task creation enabled.  
  - **Credentials:** ClickUp API credentials "ClickUp account 3".  
  - **Inputs:** Receives task payload from "Prepare ClickUp Task Payload".  
  - **Outputs:** Created task data sent to "Format Telegram Alert Message".  
  - **Edge Cases:**  
    - API limits, permission issues, or invalid IDs could cause failure.  
  - **Sticky Note:** Explains task creation and assignment details.

#### 2.8 Telegram Alert Formatting

- **Overview:** Constructs a concise and urgent Telegram message containing ticket and ClickUp task details for immediate manager notification.
- **Nodes Involved:**  
  - Format Telegram Alert Message
- **Node Details:**  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Extracts task name, ticket ID, requester name/email from the input.  
    - Builds a formatted message with emojis, bold text, and a clickable link to the ClickUp task.  
    - Includes a call to action for quick review and assignment.  
  - **Inputs:** Receives created task data from "Create a task".  
  - **Outputs:** Sends formatted message to "Send Telegram Escalation Alert".  
  - **Edge Cases:**  
    - Missing or malformed input fields could break message formatting.  
  - **Sticky Note:** Describes message formatting and urgency focus.

#### 2.9 Telegram Notification

- **Overview:** Sends the formatted escalation alert message to a predefined Telegram chat ID.
- **Nodes Involved:**  
  - Send Telegram Escalation Alert
- **Node Details:**  
  - **Type:** Telegram Node (Send Message)  
  - **Configuration:**  
    - Chat ID set to "963318735" (manager’s Telegram chat).  
    - Message text dynamically set from input JSON message field.  
  - **Credentials:** Telegram API credentials "Telegram account".  
  - **Inputs:** Receives message from "Format Telegram Alert Message".  
  - **Outputs:** None (end node).  
  - **Edge Cases:**  
    - Telegram API errors, invalid chat ID, or network issues.  
  - **Sticky Note:** Notes final delivery of alert for immediate manager action.

---

### 3. Summary Table

| Node Name                      | Node Type         | Functional Role                           | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                         |
|--------------------------------|-------------------|-----------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger    | Starts workflow manually                 | None                         | Fetch Zendesk Tickets             |                                                                                                   |
| Fetch Zendesk Tickets           | Zendesk           | Retrieves pending tickets from group    | When clicking ‘Execute workflow’ | Select Latest Ticket             | Retrieves Zendesk tickets from a specific group. Pulls all tickets with status "pending".         |
| Select Latest Ticket            | Code              | Selects most recent ticket               | Fetch Zendesk Tickets         | Fetch Requester Email, Merge Ticket & Requester Data | Chooses the most recent ticket from the Zendesk list.                                            |
| Fetch Requester Email           | Zendesk           | Retrieves requester user details         | Select Latest Ticket          | Merge Ticket & Requester Data     | Retrieves requester details from Zendesk, including name, email, timezone.                        |
| Merge Ticket & Requester Data  | Merge             | Combines ticket and requester info       | Fetch Requester Email, Select Latest Ticket | Prepare ClickUp Task Payload     | Combines ticket and requester information into one dataset.                                       |
| Prepare ClickUp Task Payload   | Code              | Formats data into ClickUp task structure | Merge Ticket & Requester Data | Create a task                    | Formats the ticket and requester data into ClickUp task structure with escalation context.        |
| Create a task                  | ClickUp           | Creates task in ClickUp                   | Prepare ClickUp Task Payload  | Format Telegram Alert Message     | Creates a new task in the designated ClickUp list with due date and assignee.                      |
| Format Telegram Alert Message  | Code              | Builds formatted Telegram alert message  | Create a task                | Send Telegram Escalation Alert    | Builds a concise escalation alert for Telegram with key details and call to action.               |
| Send Telegram Escalation Alert | Telegram          | Sends alert message to Telegram chat     | Format Telegram Alert Message | None                            | Sends the formatted message to Telegram for immediate manager notification.                       |
| Sticky Note                    | Sticky Note       | Informational comments                   | None                         | None                            | Action: Retrieves Zendesk tickets from a specific group.                                          |
| Sticky Note1                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Creates a new task in the designated ClickUp list.                                        |
| Sticky Note2                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Formats the ticket and requester data into ClickUp task structure.                        |
| Sticky Note3                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Combines ticket and requester information into one dataset.                               |
| Sticky Note4                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Retrieves requester details from Zendesk.                                                 |
| Sticky Note5                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Chooses the most recent ticket from the Zendesk list.                                    |
| Sticky Note6                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Retrieves Zendesk tickets from a specific group.                                          |
| Sticky Note7                   | Sticky Note       | Informational comments                   | None                         | None                            | Action: Sends the formatted message to Telegram.                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No special parameters.

2. **Add Zendesk Node to Fetch Tickets**  
   - Name: `Fetch Zendesk Tickets`  
   - Type: Zendesk  
   - Operation: Get All Tickets  
   - Parameters:  
     - Group: `22337660284956` (specific Zendesk group)  
     - Status: `pending`  
     - Sort By: `status`  
     - Sort Order: `desc`  
   - Credentials: Configure Zendesk API credentials (e.g., "Zendesk account vivek")  
   - Connect output of Manual Trigger to this node.

3. **Add Code Node to Select Latest Ticket**  
   - Name: `Select Latest Ticket`  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const tickets = items.map(item => item.json);
     tickets.sort((a, b) => new Date(b.created_at) - new Date(a.created_at));
     const latest = tickets[0];
     return [{
       json: {
         id: latest.id,
         subject: latest.subject,
         description: latest.description,
         requester_id: latest.requester_id,
         created_at: latest.created_at
       }
     }];
     ```  
   - Connect output of Zendesk Tickets node here.

4. **Add Zendesk Node to Fetch Requester Email**  
   - Name: `Fetch Requester Email`  
   - Type: Zendesk  
   - Operation: Get User  
   - Parameters:  
     - User ID: Expression `={{ $json.requester_id }}`  
   - Credentials: Same Zendesk API credentials  
   - Connect output of `Select Latest Ticket` node here.

5. **Add Merge Node to Combine Ticket and Requester Data**  
   - Name: `Merge Ticket & Requester Data`  
   - Type: Merge  
   - Mode: Default (Merge by Index)  
   - Connect outputs:  
     - Input 1 from `Fetch Requester Email`  
     - Input 2 from `Select Latest Ticket` (pass through)  
   - Both inputs are required.

6. **Add Code Node to Prepare ClickUp Task Payload**  
   - Name: `Prepare ClickUp Task Payload`  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const inputData = $input.all();
     const requester = inputData[0].json;
     const ticket = inputData[1].json;
     const taskName = `[Escalation] ${ticket.subject} (Ticket #${ticket.id})`;
     const escalationMessage = `🚨 This ticket has been escalated for immediate review by the support team.`;
     const description = `
     ${escalationMessage}
     
     ---
     
     **Zendesk Ticket Escalation**
     
     **Ticket Info**
     - ID: ${ticket.id}
     - Subject: ${ticket.subject}
     - Description: ${ticket.description}
     - Created At: ${ticket.created_at}
     
     **Requester Info**
     - Name: ${requester.name}
     - Email: ${requester.email}
     - Timezone: ${requester.time_zone}
     
     🔗 [View Ticket](${ticket.url})
     `;
     return [{
       json: {
         name: taskName,
         description: description,
         priority: 3,
         tags: ["zendesk", "escalation"],
       }
     }];
     ```  
   - Connect output of Merge node here.

7. **Add ClickUp Node to Create Task**  
   - Name: `Create a task`  
   - Type: ClickUp  
   - Operation: Create Task  
   - Parameters:  
     - Team: `9014872066`  
     - Space: `90143686913`  
     - List: `901411343468`  
     - Folderless: true  
     - Name: Expression `={{ $json.name }}`  
     - Additional Fields:  
       - Content (Description): Expression `={{ $json.description }}`  
       - Due Date: Expression `={{ new Date(Date.now() + 7*24*60*60*1000).toISOString() }}` (7 days from now)  
       - Assignees: `[224432632]` (user ID)  
   - Credentials: ClickUp API credentials (e.g., "ClickUp account 3")  
   - Connect output of Prepare ClickUp Task Payload node.

8. **Add Code Node to Format Telegram Alert Message**  
   - Name: `Format Telegram Alert Message`  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const inputData = $input.item.json;
     const taskName = inputData.name || "No Task Name";
     const ticketId = taskName.match(/#(\d+)/) ? taskName.match(/#(\d+)/)[1] : "N/A";
     const description = inputData.description || "";
     const requesterMatch = description.match(/Name:\s(.+)\n- Email:\s(.+)/);
     const requesterName = requesterMatch ? requesterMatch[1].trim() : "Unknown";
     const requesterEmail = requesterMatch ? requesterMatch[2].trim() : "Unknown";
     const clickupUrl = inputData.url || "No ClickUp URL";
     const message = `
     🚨 *Escalation Alert – Immediate Attention Required!*

     • *Ticket:* ${taskName}  
     • *Requester:* ${requesterName} (${requesterEmail})  

     🔗 [Open ClickUp Task](${clickupUrl})

     ⚡ Please review and assign *next steps* ASAP.
     `;
     return [{
       json: {
         message: message.trim()
       }
     }];
     ```  
   - Connect output of `Create a task` node.

9. **Add Telegram Node to Send Escalation Alert**  
   - Name: `Send Telegram Escalation Alert`  
   - Type: Telegram  
   - Parameters:  
     - Chat ID: `963318735` (Manager’s Telegram chat ID)  
     - Text: Expression `={{ $json.message }}`  
   - Credentials: Telegram API credentials (e.g., "Telegram account")  
   - Connect output of "Format Telegram Alert Message" node.

10. **Validate and Activate Workflow**  
    - Test each step for valid credentials and data flow.  
    - Ensure API permissions and IDs are correct.  
    - Activate when ready for production use.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow integrates Zendesk, ClickUp, and Telegram to automate escalation and ensure rapid managerial notification.           | Integration overview                                                                                |
| Escalation messages use emojis and Markdown formatting for emphasis in Telegram alerts.                                            | Telegram message formatting                                                                         |
| Due date for ClickUp tasks is set to exactly 7 days after task creation to ensure timely follow-up.                               | ClickUp task due date logic                                                                         |
| API credentials must be configured with appropriate permissions for Zendesk, ClickUp, and Telegram to avoid authorization errors. | Credential configuration best practices                                                           |
| If no pending tickets exist or requester data is missing, the workflow may fail or produce incomplete tasks — consider adding error handling. | Potential edge cases and failure points                                                            |
| Sticky notes within the workflow provide detailed explanations for each functional block and can assist in user onboarding.      | Workflow documentation within n8n                                                                 |
| Telegram chat ID and ClickUp team/space/list IDs are hardcoded and should be updated to match organizational setup.              | Configuration customization instructions                                                          |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.

---