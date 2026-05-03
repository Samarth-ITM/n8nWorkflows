Fast-Track VIP Tickets with Google Sheets, ClickUp, and Telegram

https://n8nworkflows.xyz/workflows/fast-track-vip-tickets-with-google-sheets--clickup--and-telegram-8754


# Fast-Track VIP Tickets with Google Sheets, ClickUp, and Telegram

### 1. Workflow Overview

This workflow automates the fast-tracking of VIP customer tickets by integrating Zendesk ticket data retrieval, VIP priority checking, task creation in ClickUp, and instant alerts via Telegram. It targets customer service teams and support departments needing to prioritize VIP tickets efficiently and ensure timely handling through automated notifications and task management.

Logical blocks:

- **1.1 Input Reception**: Manual trigger initiates the workflow.
- **1.2 Ticket Retrieval**: Fetch all tickets from Zendesk.
- **1.3 VIP Priority Check**: Filter tickets for high-priority (VIP) status.
- **1.4 VIP User Detail Retrieval**: Obtain detailed user info from Zendesk.
- **1.5 Task Creation in ClickUp**: Automatically create a priority task with ticket details.
- **1.6 Telegram Alerting**: Send a Telegram notification about the VIP ticket.
- **1.7 Documentation and Setup Notes**: Sticky notes provide instructions, setup checklists, and benefits for users and maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block starts the workflow manually on user command.
- **Nodes Involved:** `When clicking ‘Execute workflow’`
- **Node Details:**
  - Type: Manual Trigger
  - Configuration: No parameters; executes when user clicks ‘Execute workflow’ in n8n.
  - Input: None (start node)
  - Output: Triggers `Get many tickets` node.
  - Edge cases: None significant; manual trigger implies intentional execution.
  - Version: Compatible with n8n v1+

#### 2.2 Ticket Retrieval

- **Overview:** Retrieves all customer tickets from Zendesk for further processing.
- **Nodes Involved:** `Get many tickets`
- **Node Details:**
  - Type: Zendesk Node (Get All Tickets)
  - Configuration: Operation `getAll`, `returnAll` set to true to fetch all tickets.
  - Input: Triggered from manual trigger node.
  - Output: List of tickets forwarded to the VIP priority check.
  - Credentials: Requires Zendesk API credentials with permissions to read tickets.
  - Edge cases: API rate limits, authentication failure, empty ticket list.
  - Version: Zendesk node v1+
  
#### 2.3 VIP Priority Check

- **Overview:** Filters tickets to identify those marked as 'high' priority, representing VIP status.
- **Nodes Involved:** `Check High Priority`
- **Node Details:**
  - Type: If Node
  - Configuration: Checks if the `priority` field of each ticket contains the string `"high"`.
  - Input: Receives all tickets from `Get many tickets`.
  - Output: Only tickets with `priority` containing `"high"` proceed.
  - Expressions: `={{ $('Get many tickets').item.json.priority }} contains "high"`
  - Edge cases: Tickets with missing or null priority fields, case sensitivity of priority values.
  - Version: If node v1+
  
#### 2.4 VIP User Detail Retrieval

- **Overview:** Fetches detailed user information for each VIP ticket requester from Zendesk.
- **Nodes Involved:** `Get the VIP user`
- **Node Details:**
  - Type: Zendesk Node (Get User)
  - Configuration: Gets user by `id` equal to `requester_id` from the ticket JSON.
  - Input: Receives VIP ticket data from `Check High Priority`.
  - Output: User details sent to task creation node.
  - Expressions: `id={{ $json.requester_id }}`
  - Credentials: Zendesk API credentials with user read permissions.
  - Edge cases: Missing user ID, user not found, Zendesk API errors.
  - Version: Zendesk node v1+
  
#### 2.5 Task Creation in ClickUp

- **Overview:** Creates a priority task in ClickUp for each VIP ticket to enable tracking and handling.
- **Nodes Involved:** `Create ClickUp Task1`
- **Node Details:**
  - Type: ClickUp Node (Create Task)
  - Configuration:
    - Team ID: `9016683627`
    - Space ID: `90162844741`
    - Folder ID: `90164394824`
    - List ID: `901611032798`
    - Task Name: `"VIP Ticket: [description]"` from VIP ticket description field.
  - Input: Receives user and ticket info from `Get the VIP user`.
  - Output: Task creation result passed to Telegram alert node.
  - Expressions: `=VIP Ticket:{{ $('Check High Priority').item.json.description }}`
  - Credentials: ClickUp API credentials with task creation permissions.
  - Edge cases: Invalid team/space/folder/list IDs, API limits, permission errors.
  - Version: ClickUp node v1+
  
#### 2.6 Telegram Alerting

- **Overview:** Sends an instant Telegram message notifying the team of the new VIP ticket task.
- **Nodes Involved:** `Send Telegram Alert1`
- **Node Details:**
  - Type: Telegram Node (Send Message)
  - Configuration:
    - Chat ID: Placeholder `<Your Chat ID>` (must be replaced by user)
    - Message Text: Includes VIP ticket subject and requester email.
  - Input: Receives task creation confirmation and user info from `Create ClickUp Task1`.
  - Output: Final node; no further output.
  - Expressions: 
    ```
    🚨 VIP Ticket received!

    Subject:{{ $('Check High Priority').item.json.subject }}
    Requester: {{ $('Get the VIP user').item.json.email }}
    Check ClickUp for the created task.
    ```
  - Credentials: Telegram bot token credentials required.
  - Edge cases: Invalid chat ID, bot token revocation, Telegram API rate limits.
  - Version: Telegram node v1+
  
#### 2.7 Documentation and Setup Notes

- **Overview:** Contains multiple sticky notes providing users with instructions, setup checklists, explanations, and benefits.
- **Nodes Involved:** All sticky notes:
  - `📋 Workflow Overview`
  - `🚀 Trigger Instructions`
  - `📊 Data Source Info`
  - `🔍 VIP Detection Logic`
  - `📝 Task Management`
  - `📱 Alert System`
  - `⚡ Setup Checklist`
  - `🎯 Business Benefits`
- **Node Details:**
  - Type: Sticky Note
  - Configuration: Textual instructions and workflow context.
  - Purpose: Aid users in understanding, configuring, and troubleshooting the workflow.
  - Edge cases: None; static documentation.
  - Version: n8n v1+

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                         | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                          |
|----------------------------|-----------------------|---------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Starts the workflow                    | None                       | Get many tickets          | 🚀 START HERE: Manual trigger to start the VIP processing chain; can be replaced with other triggers |
| Get many tickets           | Zendesk               | Retrieves all tickets                  | When clicking ‘Execute workflow’ | Check High Priority       | 📊 DATA SOURCE: Reads tickets from Zendesk (Google Sheets note is outdated); requires Zendesk credentials |
| Check High Priority        | If                    | Filters tickets for high priority (VIP) | Get many tickets            | Get the VIP user          | 🔍 VIP DETECTION: Filters tickets with priority ‘high’                                            |
| Get the VIP user           | Zendesk               | Gets user details for VIP ticket requester | Check High Priority         | Create ClickUp Task1      |                                                                                                    |
| Create ClickUp Task1       | ClickUp                | Creates priority task in ClickUp       | Get the VIP user            | Send Telegram Alert1      | 📝 TASK CREATION: Creates task named “VIP Ticket: [Description]”; requires ClickUp API credentials  |
| Send Telegram Alert1       | Telegram               | Sends Telegram notification            | Create ClickUp Task1        | None                     | 📱 INSTANT ALERTS: Sends VIP alert message; requires Telegram bot token and chat ID                 |
| 📋 Workflow Overview       | Sticky Note            | Provides workflow summary              | None                       | None                     | Describes purpose and flow                                                                         |
| 🚀 Trigger Instructions    | Sticky Note            | Explains how to start workflow         | None                       | None                     | Instructions for manual trigger; suggests replacing with webhook or schedule                       |
| 📊 Data Source Info        | Sticky Note            | Notes on data source                   | None                       | None                     | Explains data source and setup requirements                                                        |
| 🔍 VIP Detection Logic     | Sticky Note            | Details VIP filtering logic            | None                       | None                     | Explains If Node condition for VIP detection                                                       |
| 📝 Task Management         | Sticky Note            | Explains ClickUp task creation         | None                       | None                     | Details on creating priority tasks in ClickUp                                                     |
| 📱 Alert System            | Sticky Note            | Describes Telegram notifications       | None                       | None                     | Setup instructions for Telegram bot                                                               |
| ⚡ Setup Checklist         | Sticky Note            | Checklist before running workflow      | None                       | None                     | Lists prerequisites and common issues                                                             |
| 🎯 Business Benefits       | Sticky Note            | Explains workflow benefits             | None                       | None                     | Highlights why to use this automation                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking ‘Execute workflow’`
   - No parameters.
   - This node starts the workflow manually.

2. **Add Zendesk Node to Get All Tickets**
   - Type: Zendesk
   - Operation: `getAll`
   - Parameters: Return all tickets (`returnAll=true`)
   - Credentials: Configure Zendesk API credentials with ticket read permission.
   - Connect output of Manual Trigger to this node.

3. **Add If Node to Filter High Priority Tickets**
   - Type: If
   - Condition: String contains
   - Set value1 to the priority field of current item: `={{ $json.priority }}`
   - Set value2 to `"high"`
   - Connect output of Zendesk ‘getAll’ node to this If node.
   - Only tickets with priority containing "high" proceed.

4. **Add Zendesk Node to Get User Details**
   - Type: Zendesk
   - Operation: `get`
   - Resource: `user`
   - ID: Use expression `={{ $json.requester_id }}`
   - Credentials: Same Zendesk API credentials.
   - Connect ‘true’ output of If node to this node.

5. **Add ClickUp Node to Create Task**
   - Type: ClickUp
   - Operation: Create Task
   - Team ID: `9016683627` (replace with your own)
   - Space ID: `90162844741`
   - Folder ID: `90164394824`
   - List ID: `901611032798`
   - Task Name: Use expression `=VIP Ticket:{{ $json.description }}`
   - Credentials: Configure ClickUp API credentials with task creation permissions.
   - Connect output of Zendesk Get User node to this node.

6. **Add Telegram Node to Send Alert**
   - Type: Telegram
   - Operation: Send Message
   - Chat ID: Replace `<Your Chat ID>` with your Telegram chat/channel ID.
   - Text: Use expression:
     ```
     🚨 VIP Ticket received!

     Subject:{{ $json.subject }}
     Requester: {{ $json.email }}
     Check ClickUp for the created task.
     ```
   - Credentials: Configure Telegram bot token credentials.
   - Connect output of ClickUp Create Task node to this node.

7. **Add Sticky Notes for Documentation (Optional)**
   - Add multiple Sticky Note nodes with the provided instructional content:
     - Workflow overview
     - Trigger instructions
     - Data source info
     - VIP detection logic
     - Task management
     - Alert system
     - Setup checklist
     - Business benefits

8. **Test the Workflow**
   - Run manually using the Manual Trigger node.
   - Verify tickets are fetched, filtered for VIP high priority.
   - Confirm tasks are created in ClickUp.
   - Confirm Telegram alert messages are received.
   - Adjust credentials and parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow demonstrates fast-tracking VIP tickets using Zendesk, ClickUp, and Telegram.       | Workflow purpose                                                 |
| Replace manual trigger with webhook or schedule triggers for automated runs.                     | Trigger Instructions sticky note                                 |
| Ensure all credentials (Zendesk, ClickUp, Telegram) are correctly configured and tested.        | Setup Checklist sticky note                                      |
| Telegram chat ID must be updated to the correct ID for message delivery.                         | Alert System sticky note                                         |
| ClickUp IDs (team, space, folder, list) must reflect your workspace structure.                   | Task Management sticky note                                      |
| Common issues include API rate limits, permission errors, and incorrect chat ID or tokens.      | Setup Checklist sticky note                                      |
| Workflow benefits include speed, consistency, visibility, and scalability for customer support. | Business Benefits sticky note                                    |
| For more info on n8n nodes and credential setup, refer to [n8n docs](https://docs.n8n.io).       | General resource                                                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.