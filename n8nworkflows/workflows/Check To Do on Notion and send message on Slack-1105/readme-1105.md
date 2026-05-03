Check To Do on Notion and send message on Slack

https://n8nworkflows.xyz/workflows/check-to-do-on-notion-and-send-message-on-slack-1105


# Check To Do on Notion and send message on Slack

### 1. Workflow Overview

This workflow automates the daily checking of a To-Do list stored in Notion and sends personalized notifications to a Slack user if specific tasks are assigned to them and remain incomplete. It is designed to streamline task monitoring and ensure timely reminders via Slack direct messaging.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically every day at a fixed hour.
- **1.2 Notion Data Retrieval:** Fetches all To-Do tasks from a specified Notion page.
- **1.3 Task Filtering:** Evaluates each task to determine if it is assigned to a specific user and is incomplete.
- **1.4 Slack Messaging:** Opens a direct message channel with the user and sends the relevant To-Do task as a message.
- **1.5 No Operation Handling:** Handles the case where tasks do not meet the criteria, effectively terminating those branches.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow at a specified time daily, ensuring automated execution without manual intervention.

- **Nodes Involved:**  
  - Cron

- **Node Details:**

  - **Cron**  
    - Type: Trigger node  
    - Configuration: Set to trigger once daily at 08:00 (8 AM)  
    - Key parameters: Hour set to 8, no additional minutes specified (default 0)  
    - Input connections: None (trigger node)  
    - Output connections: Connected to "Get To Dos" node  
    - Edge cases: Workflow will not run if the n8n instance is down at trigger time; time zone considerations should be verified to match intended schedule.  
    - Notes: Reliable for daily scheduling but should be paired with error handling in downstream nodes.

#### 2.2 Notion Data Retrieval

- **Overview:**  
  Fetches all To-Do tasks from a specified Notion block, providing the raw task data for further evaluation.

- **Nodes Involved:**  
  - Get To Dos

- **Node Details:**

  - **Get To Dos**  
    - Type: Notion node (resource: block, operation: getAll)  
    - Configuration:  
      - `blockId`: Identifier of the Notion page/block containing the To-Do list (example shown as "bafdscf")  
      - `returnAll`: true (fetch all matching blocks)  
    - Credentials: Requires configured Notion API credentials with appropriate access rights.  
    - Input connections: Receives trigger from Cron node  
    - Output connections: Feeds data into the IF node  
    - Key expressions: None directly here; raw JSON output includes task details under the "to_do" property.  
    - Edge cases:  
      - API authentication failures  
      - Permissions issues if token lacks read access  
      - Notion API rate limits or downtime  
      - Block ID must be accurate and correspond to a To-Do list page  
    - Notes: Proper credentials setup is critical; see prerequisites for links.

#### 2.3 Task Filtering

- **Overview:**  
  Evaluates each fetched task to determine if it is assigned to a specific user and if the task remains incomplete, thus filtering for relevant tasks.

- **Nodes Involved:**  
  - If task assigned to Harshil?

- **Node Details:**

  - **If task assigned to Harshil?**  
    - Type: IF node (conditional logic)  
    - Configuration:  
      - Conditions:  
        - String condition comparing the assigned user’s name in the task mention to a hardcoded name ("NAME" placeholder)  
        - Boolean condition checking that the task's "checked" status is false (incomplete)  
      - Expression for assigned user name: `{{$json["to_do"]["text"][1]["mention"]["user"]["name"]}}`  
      - Expression for checked status: `{{$json["to_do"]["checked"]}}`  
    - Input connections: From "Get To Dos" node  
    - Output connections:  
      - True: To "Create a Direct Message" node  
      - False: To "NoOp" node  
    - Edge cases:  
      - The expression assumes the second element of the to_do text array contains a mention with user data; if the structure varies, expression may fail.  
      - Task without assigned user mention or unexpected data structure causes evaluation failure.  
      - Case sensitivity and exact matching of user names may cause mismatches.  
    - Notes: User name placeholder "NAME" must be replaced with the actual target user's name (e.g., "Harshil"). This node acts as a filter gate.

#### 2.4 Slack Messaging

- **Overview:**  
  Opens a direct Slack message channel with the user and sends a formatted message containing the filtered To-Do task.

- **Nodes Involved:**  
  - Create a Direct Message  
  - Send a Direct Message

- **Node Details:**

  - **Create a Direct Message**  
    - Type: Slack node (resource: channel, operation: open)  
    - Configuration:  
      - Users: Array containing Slack user ID ("U01JXLAJ6SE") to open DM with  
    - Credentials: Requires Slack API credentials with chat permissions  
    - Input connections: From IF node (true branch)  
    - Output connections: To "Send a Direct Message" node  
    - Edge cases:  
      - Invalid or expired Slack credentials  
      - User ID not correct or user no longer exists  
      - Slack API rate limits or service disruption  
    - Notes: Executes once per matched task; ensure user ID matches intended recipient.

  - **Send a Direct Message**  
    - Type: Slack node (resource: chat message, operation: post message)  
    - Configuration:  
      - Text: Static "# TO DO" header  
      - Channel: Dynamically set to the channel ID returned from "Create a Direct Message" node (`{{$json["id"]}}`)  
      - Attachments: Contains one attachment with title set to the content of the first text segment of the task (checkbox emoji prepended)  
      - Other options: Markdown enabled for formatting  
    - Credentials: Slack API credentials (same as above)  
    - Input connections: From "Create a Direct Message" node  
    - Output connections: None (end node for this path)  
    - Edge cases:  
      - Message send failure due to network or API errors  
      - Incorrect channel ID propagation would cause message delivery failure  
      - Payload formatting errors if task text structure varies  
    - Notes: Message content pulls from the first text element of the to_do field; may need adjustment if task text structure changes.

#### 2.5 No Operation Handling

- **Overview:**  
  Terminates the workflow branch cleanly when tasks do not satisfy the condition, with no further action taken.

- **Nodes Involved:**  
  - NoOp

- **Node Details:**

  - **NoOp**  
    - Type: No Operation node  
    - Configuration: Default (no parameters)  
    - Input connections: From IF node (false branch)  
    - Output connections: None  
    - Edge cases: None; purely passive node  
    - Notes: Used to explicitly end workflow branches with no relevant tasks.

---

### 3. Summary Table

| Node Name                 | Node Type                | Functional Role                              | Input Node(s)        | Output Node(s)                  | Sticky Note                                                                                          |
|---------------------------|--------------------------|----------------------------------------------|----------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Cron                      | Cron Trigger             | Triggers workflow daily at 8 AM               | None                 | Get To Dos                     |                                                                                                    |
| Get To Dos                | Notion                   | Retrieves all To-Do tasks from Notion         | Cron                 | If task assigned to Harshil?   | Requires Notion credentials; ensure blockId matches your To-Do page (see prerequisites).            |
| If task assigned to Harshil? | IF                       | Filters tasks assigned to specific user & incomplete | Get To Dos           | Create a Direct Message (true) <br> NoOp (false) | Check if the task is incomplete. Replace "NAME" with target user’s name.                            |
| Create a Direct Message    | Slack                    | Opens DM channel with the user                 | If (true)            | Send a Direct Message          | Requires Slack API credentials; User ID must be correct for recipient.                              |
| Send a Direct Message      | Slack                    | Sends To-Do task message in DM                  | Create a Direct Message | None                         | Sends message with task content; markdown enabled.                                                 |
| NoOp                      | No Operation             | Ends the branch when no relevant tasks         | If (false)           | None                          | No further action if condition is false.                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Check To Do on Notion and send message on Slack".

2. **Add a Cron node:**  
   - Set trigger time to 08:00 (hour = 8, minutes = 0 by default)  
   - This node will start your workflow daily.

3. **Add a Notion node:**  
   - Set resource to "block"  
   - Set operation to "getAll"  
   - Set `blockId` to your Notion To-Do list page/block ID (e.g., copy from your page URL or Notion integration settings)  
   - Enable "Return All" to fetch all tasks  
   - Connect the Cron node's output to this Notion node's input.  
   - Configure Notion API credentials (create and link credentials with appropriate permissions). Reference: https://docs.n8n.io/credentials/notion/

4. **Add an IF node:**  
   - Purpose: Filter tasks assigned to a specific user and incomplete  
   - Under Conditions:  
     - Add a string condition:  
       - Value 1: Expression `{{$json["to_do"]["text"][1]["mention"]["user"]["name"]}}`  
       - Value 2: Your target username as a string (e.g., "Harshil")  
     - Add a boolean condition:  
       - Value 1: Expression `{{$json["to_do"]["checked"]}}`  
       - Condition: should be false (i.e., task is not checked)  
   - Connect the Notion node output to the IF node input.

5. **Add a Slack node to create a DM channel:**  
   - Resource: Channel  
   - Operation: Open  
   - Under Options > Users: Enter the Slack user ID of the recipient (e.g., "U01JXLAJ6SE")  
   - Connect IF node's "true" output to this Slack node.  
   - Configure Slack API credentials with chat:write and im:write scopes. Reference: https://docs.n8n.io/credentials/slack/

6. **Add a Slack node to send the message:**  
   - Resource: Chat message  
   - Operation: Post message  
   - Text: `# TO DO` (static header)  
   - Channel: Expression `{{$json["id"]}}` (channel ID from previous Slack node output)  
   - Attachments:  
     - Title: Expression `=☑️ {{$node["If task assigned to Harshil?"].json["to_do"]["text"][0]["text"]["content"]}}`  
   - Enable Markdown in message options  
   - Connect the "Create a Direct Message" node output to this node.

7. **Add a No Operation node:**  
   - Place it to handle the "false" output from the IF node  
   - Connect the IF node’s false output to this NoOp node.  
   - No configuration needed.

8. **Double-check all connections:**  
   - Cron → Notion → IF → (true) Slack Open DM → Slack Send message  
   - IF → (false) NoOp

9. **Save and activate the workflow.**

10. **Test the workflow manually** or wait for the scheduled trigger to verify functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Replace `"NAME"` in the IF node string condition with the exact username of the intended user.  | Ensures filtering works correctly for your specific user assignment.                            |
| Notion To-Do list page sample: https://www.notion.so/n8n/To-Do-520ca3bdb6084098a4a80cfddd957488 | Example template for structuring your Notion To-Do list.                                       |
| Notion credentials setup guide: https://docs.n8n.io/credentials/notion/                         | Documentation for creating and configuring Notion API credentials in n8n.                      |
| Slack credentials setup guide: https://docs.n8n.io/credentials/slack/                           | Instructions to create Slack OAuth2 credentials with required permissions for messaging.       |
| Slack user ID must be correct and corresponds to the recipient user in your Slack workspace.    | You can get user ID by Slack API methods or Slack client user info.                             |
| Task text structure in Notion is assumed fixed; if your Notion tasks differ, expressions may fail. | Adjust expressions according to your actual Notion task JSON structure.                         |

---

This documentation provides a complete and detailed reference to understand, maintain, and reproduce the "Check To Do on Notion and send message on Slack" workflow in n8n.