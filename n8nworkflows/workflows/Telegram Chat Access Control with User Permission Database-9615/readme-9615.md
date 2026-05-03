Telegram Chat Access Control with User Permission Database

https://n8nworkflows.xyz/workflows/telegram-chat-access-control-with-user-permission-database-9615


# Telegram Chat Access Control with User Permission Database

### 1. Workflow Overview

This workflow implements **Telegram Chat Access Control with User Permission Database**. Its primary purpose is to restrict Telegram bot interactions to authorized users only, based on a predefined user access list stored in a data table. It is ideal for internal bots or automations where only team members or selected users should have access.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages.
- **1.2 User Access Lookup:** Queries a user permission database to check the sender’s access status.
- **1.3 Permission Decision:** Evaluates whether access is granted or denied.
- **1.4 Response Handling:** Sends a denial message if access is denied or allows continuation if granted.
- **1.5 Documentation and User Guidance:** Contains sticky notes with instructions, explanations, and links.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming messages from Telegram users who interact with the bot.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages  
    - Configuration: Set to listen for `message` updates only.  
    - Key expressions: Uses credentials named "Test" for Telegram API authentication.  
    - Input: None (trigger)  
    - Output: Emits the Telegram message JSON object containing message and user info.  
    - Edge cases: Invalid or expired credentials, Telegram API downtime, webhook misconfiguration.  
    - Notes: Requires setting up Telegram Bot credentials through @BotFather and n8n credentials.  
    - Sticky Note Reference: Instructional sticky notes explain setup steps.

---

#### 1.2 User Access Lookup

- **Overview:**  
  Queries a user permission database to identify if the Telegram user has access rights.

- **Nodes Involved:**  
  - Database with employees (Data Table node)  
  - (Optional alternative nodes present but not connected: Google Sheets, Airtable, Notion)

- **Node Details:**

  - **Database with employees**  
    - Type: Data Table node (n8n native data table)  
    - Configuration: Performs a `get` operation with a filter on the `UserName` column matching the Telegram message sender’s username (`{{$json.message.from.username}}`).  
    - Input: Telegram Trigger output  
    - Output: Returns user record(s) with access status.  
    - Edge cases: Username not found, empty results, data table misconfiguration, username case sensitivity issues.  
    - Notes: The data table must be pre-populated with usernames and their access status ("Granted" or "Denied").  
    - Sticky Note Reference: Explanation of data table usage and employee access filter.

  - **Other data source nodes (Google Sheets, Airtable, Notion)**  
    - Present but not connected; alternatives if user prefers those storage solutions.

---

#### 1.3 Permission Decision

- **Overview:**  
  Evaluates the retrieved user access status and routes the workflow accordingly.

- **Nodes Involved:**  
  - Permission (Switch)

- **Node Details:**

  - **Permission (Switch)**  
    - Type: Switch node  
    - Configuration: Checks the value of `$json.Access` field from the user record.  
      - If `"Granted"` → route continues to "No Operation" node.  
      - If `"Denied"` → route sends denial message.  
    - Input: User record from the Data Table  
    - Output: Two outputs labeled "Granted" and "Denied"  
    - Edge cases: Missing or malformed `Access` field, case sensitivity mismatches, no matching user record leading to undefined `$json.Access`.  
    - Notes: This node controls the access flow branching.

---

#### 1.4 Response Handling

- **Overview:**  
  Sends a Telegram response if access is denied or allows the workflow to continue if access is granted.

- **Nodes Involved:**  
  - No Operation, do nothing  
  - Answer Denied (Telegram node)

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp (no operation) node  
    - Configuration: Default, no processing; placeholder for granted users to connect further logic.  
    - Input: Permission node "Granted" output  
    - Output: None connected (can be extended)  
    - Edge cases: None; acts as a pass-through or placeholder.

  - **Answer Denied**  
    - Type: Telegram node (send message)  
    - Configuration: Sends text "Access dinied" (typo in original — should be "Access denied") to the chat ID extracted dynamically from the original Telegram message (`{{$('Input Data').item.json.message.chat.id}}`).  
    - Input: Permission node "Denied" output  
    - Output: Sends Telegram message confirming denial  
    - Edge cases: Telegram API auth failure, invalid chatId, message send failure.  
    - Notes: Text can be customized; attribution disabled.

---

#### 1.5 Documentation and User Guidance

- **Overview:**  
  Provides detailed instructions, explanations, and external resources via sticky notes for users and developers.

- **Nodes Involved:**  
  - Multiple Sticky Note nodes (Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7)

- **Node Details:**

  - Sticky Notes cover:  
    - Input data explanation  
    - Employee access table setup  
    - Access filtering logic  
    - Response on access denial  
    - Workflow description and quick guide  
    - Video tutorial link: https://youtu.be/Blm7iamYaoA  
    - Step-by-step Telegram bot connection instructions  
    - Use case and testing recommendations  

  - Notes are purely informational with no execution role but essential for understanding and reproducing the workflow.

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                        | Input Node(s)          | Output Node(s)               | Sticky Note                                                      |
|-------------------------|------------------|-------------------------------------|-----------------------|-----------------------------|-----------------------------------------------------------------|
| Telegram Trigger        | telegramTrigger  | Receives Telegram messages           | None                  | Database with employees      | Setup instructions in Sticky Note7                              |
| Database with employees | dataTable        | Lookup user access status            | Telegram Trigger      | Permission                   | Data table explained in Sticky Note1, Sticky Note2             |
| Permission             | switch           | Route based on access status         | Database with employees| No Operation, Answer Denied  | Access filter logic in Sticky Note2                             |
| No Operation, do nothing| noOp             | Placeholder for granted users        | Permission (Granted)  | None                        | Response flow continuation                                      |
| Answer Denied          | telegram         | Sends denial message                  | Permission (Denied)   | None                        | Response message explained in Sticky Note3                      |
| Sticky Note            | stickyNote       | Informational                       | None                  | None                        | Input data explanation                                          |
| Sticky Note1           | stickyNote       | Informational                       | None                  | None                        | Employee access table info                                      |
| Sticky Note2           | stickyNote       | Informational                       | None                  | None                        | Access filter explanation                                       |
| Sticky Note3           | stickyNote       | Informational                       | None                  | None                        | Denial response explanation                                    |
| Sticky Note4           | stickyNote       | Informational                       | None                  | None                        | Workflow overview and use case                                 |
| Sticky Note5           | stickyNote       | Informational                       | None                  | None                        | Video tutorial link                                            |
| Sticky Note6           | stickyNote       | Informational                       | None                  | None                        | Additional usage notes                                         |
| Sticky Note7           | stickyNote       | Informational                       | None                  | None                        | Detailed Telegram bot connection instructions and testing guide|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram API Credential**  
   - In n8n, go to Credentials → Create new → Telegram API.  
   - Name it (e.g., “Test”).  
   - Obtain a Telegram Bot Token via @BotFather on Telegram (`/newbot` command).  
   - Paste the Bot Token into credentials and save.

2. **Add Telegram Trigger Node**  
   - Create a new node of type **Telegram Trigger**.  
   - Set **Updates** to include only `message`.  
   - Assign the Telegram API credential created in step 1.  
   - Save.

3. **Create Data Table for User Permissions**  
   - Use n8n Data Table or external source (Google Sheets, Airtable, Notion).  
   - Add records with two fields: `UserName` and `Access` (values: “Granted” or “Denied”).  
   - Example entries: `johndoe` → `Granted`, `janedoe` → `Denied`.

4. **Add Data Table Node to Lookup User**  
   - Add a **Data Table** node named “Database with employees”.  
   - Operation: `get`  
   - Filter: Set condition where `UserName` equals expression `{{$json.message.from.username}}` from Telegram Trigger.  
   - Connect Telegram Trigger output to this node input.

5. **Add Switch Node for Permission Check**  
   - Add a **Switch** node named “Permission”.  
   - Add two rules:  
     - Output “Granted”: Condition equals string `Granted` on `$json.Access`.  
     - Output “Denied”: Condition equals string `Denied` on `$json.Access`.  
   - Connect “Database with employees” output to this node input.

6. **Add No Operation Node for Granted Access**  
   - Add a **No Operation** node named “No Operation, do nothing”.  
   - Connect the “Granted” output of the “Permission” node to this node.

7. **Add Telegram Node to Send Denial Message**  
   - Add a **Telegram** node named “Answer Denied”.  
   - Operation: Send message  
   - Text: “Access denied” (correct the typo)  
   - Chat ID: Expression `{{$json.message.chat.id}}` or `{{$('Telegram Trigger').item.json.message.chat.id}}`  
   - Assign the Telegram API credential.  
   - Connect the “Denied” output of the “Permission” node to this node.

8. **Activate Workflow**  
   - Set the workflow status to active.  
   - Test by sending a message to the Telegram bot from different usernames to verify access control.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                         |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Detailed step-by-step Telegram bot connection instructions, including @BotFather setup and credential creation in n8n.         | See Sticky Note7 content                                |
| Video tutorial for workflow setup and use: [YouTube Video](https://youtu.be/Blm7iamYaoA)                                        | Sticky Note5                                            |
| Use case: restrict internal bots or automations only to authorized team members.                                                | Sticky Note4                                            |
| The workflow supports multiple data sources for user permissions: n8n Data Table (recommended), Google Sheets, Airtable, Notion. | Sticky Note1                                            |
| Customize denial message in the “Answer Denied” node’s text field.                                                              | Sticky Note3                                            |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.