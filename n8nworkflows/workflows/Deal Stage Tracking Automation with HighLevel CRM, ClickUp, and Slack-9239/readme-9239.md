Deal Stage Tracking Automation with HighLevel CRM, ClickUp, and Slack

https://n8nworkflows.xyz/workflows/deal-stage-tracking-automation-with-highlevel-crm--clickup--and-slack-9239


# Deal Stage Tracking Automation with HighLevel CRM, ClickUp, and Slack

---

### 1. Workflow Overview

This workflow automates the tracking of deal stage changes from HighLevel CRM and synchronizes these updates with ClickUp tasks while providing alert notifications via Slack for older deals. It is designed for sales or customer success teams who manage deals in HighLevel CRM and need an automated way to reflect deal status changes in ClickUp task management and receive alerts about outdated deal updates.

The workflow logically divides into the following blocks:

- **1.1 Manual Trigger Input**: Starts the workflow manually for on-demand execution.
- **1.2 Fetch Deals from HighLevel CRM**: Retrieves all deals (opportunities) including their full details.
- **1.3 Date-Based Filtering**: Checks if deals were updated recently or prior to a cutoff date.
- **1.4 Process Recent Deals**: For recent updates, fetches related contact information and creates ClickUp tasks.
- **1.5 Process Older Deals**: For outdated updates, sends Slack notifications to alert team members.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger Input

- **Overview:** Initiates the workflow manually on-demand. Useful for testing, bulk processing, or one-time syncs.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Sticky Note - Workflow Start

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point node that requires user interaction to start the workflow.  
    - Configuration: Default manual trigger with no parameters.  
    - Input: None  
    - Output: Triggers the next node to fetch deals.  
    - Edge cases: None typical; user must execute manually.

  - **Sticky Note - Workflow Start**  
    - Type: Sticky Note  
    - Role: Documentation overlay describing the workflow start and its purpose.  
    - Content: Highlights manual trigger purpose for testing, bulk processing, and syncing deals.  
    - No inputs or outputs.

---

#### 1.2 Fetch Deals from HighLevel CRM

- **Overview:** Retrieves all deal opportunities from HighLevel CRM using the HighLevel API. Gathers complete deal details including stages and contact references.
- **Nodes Involved:**  
  - Fetch All Deals from CRM1 (HighLevel node)  
  - Sticky Note - Fetch Deals

- **Node Details:**

  - **Fetch All Deals from CRM1**  
    - Type: HighLevel CRM node (resource: opportunity, operation: getAll)  
    - Role: Calls HighLevel API endpoint GET /opportunities to get all deals without limit.  
    - Configuration: `returnAll=true` to retrieve the complete set of deals.  
    - Credentials: Connected with OAuth2 credentials for HighLevel account.  
    - Input: Triggered by manual trigger node output.  
    - Output: Array of deal objects with all fields including lastStatusChangeAt and contactId.  
    - Edge cases: API rate limits, invalid credentials, network issues.

  - **Sticky Note - Fetch Deals**  
    - Type: Sticky Note  
    - Role: Describes the purpose and API endpoint used to fetch all deals.  
    - No inputs or outputs.

---

#### 1.3 Date-Based Filtering

- **Overview:** Splits deals into two branches based on whether their last status change date is on or after September 30, 2025. This determines if the deal is recent or outdated.
- **Nodes Involved:**  
  - Filter Recent Deal Updates1 (If node)  
  - Sticky Note - Date Filter

- **Node Details:**

  - **Filter Recent Deal Updates1**  
    - Type: If Node  
    - Role: Conditional logic node that evaluates if deal’s `lastStatusChangeAt` datetime is after or equal to `2025-09-30T00:00:00`.  
    - Configuration: Uses expression `={{ $json.lastStatusChangeAt }}` compared to static cutoff date.  
    - Input: Receives array of deals from Fetch All Deals node, processes each item separately.  
    - Output:  
      - True branch: deals updated on or after cutoff date.  
      - False branch: deals updated before cutoff date.  
    - Edge cases: Missing or malformed `lastStatusChangeAt` field; timezone considerations; null values.

  - **Sticky Note - Date Filter**  
    - Type: Sticky Note  
    - Role: Explains the date filtering logic and branch behavior.  
    - No inputs or outputs.

---

#### 1.4 Process Recent Deals (True Branch)

- **Overview:** For deals updated recently, fetch full contact details from HighLevel CRM and create tasks in ClickUp with contextual information.
- **Nodes Involved:**  
  - Get Contact Details1 (HighLevel node)  
  - Create ClickUp Task1 (ClickUp node)  
  - Sticky Note - Get Contact  
  - Sticky Note - Create Task

- **Node Details:**

  - **Get Contact Details1**  
    - Type: HighLevel CRM node (resource: contact, operation: get)  
    - Role: Fetches detailed contact information using contactId from deal data.  
    - Configuration: Expression `={{ $json.contactId }}` passed to `contactId` parameter.  
    - Credentials: Uses same HighLevel OAuth2 credentials.  
    - Input: Each deal from True branch of If node.  
    - Output: Contact data including full name, locationId, and other fields.  
    - Edge cases: Missing contactId, contact not found, API errors.

  - **Create ClickUp Task1**  
    - Type: ClickUp node (operation: create task)  
    - Role: Creates a new task in ClickUp with details about the contact and deal update.  
    - Configuration:  
      - List ID: `901611225384` (predefined ClickUp list)  
      - Task name: Includes contact full name (lowercase), location id, and note “Changed Status recently” via expression.  
      - Team and Space IDs: `90161261705` and `90165174252` respectively.  
      - Folderless: true  
    - Credentials: ClickUp API credentials set.  
    - Input: Contact details from previous node.  
    - Output: ClickUp task creation response.  
    - Edge cases: Invalid list or team IDs, API errors, rate limiting.

  - **Sticky Note - Get Contact**  
    - Type: Sticky Note  
    - Role: Explains the API call to get full contact details and its necessity for task creation.  
    - No inputs or outputs.

  - **Sticky Note - Create Task**  
    - Type: Sticky Note  
    - Role: Describes the ClickUp task creation, main output of the workflow, and data included in the task.  
    - No inputs or outputs.

---

#### 1.5 Process Older Deals (False Branch)

- **Overview:** For deals updated before the cutoff date, send a Slack notification to a specific user to alert about the outdated update.
- **Nodes Involved:**  
  - Notify Old Deal Update1 (Slack node)  
  - Sticky Note - Slack Alert

- **Node Details:**

  - **Notify Old Deal Update1**  
    - Type: Slack node (operation: send message)  
    - Role: Sends a direct message notification to a Slack user about an old deal update.  
    - Configuration:  
      - Text message includes lead ID from deal JSON: `Hi, your lead ID: {{ $json.id }} was updated before the cutoff date`  
      - Recipient user ID: `U09E375JZPA` (n8n_workspace user)  
      - Mode: select user by ID  
    - Credentials: Slack OAuth2 credentials configured.  
    - Input: Deals from False branch of If node.  
    - Output: Slack API message response.  
    - Edge cases: Invalid user ID, Slack API rate limits, connection errors.

  - **Sticky Note - Slack Alert**  
    - Type: Sticky Note  
    - Role: Describes the Slack alert purpose for old deals and summarizes message content.  
    - No inputs or outputs.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                      | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------------|---------------------|------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Entry point for manual execution   | None                          | Fetch All Deals from CRM1       | ## 🚀 WORKFLOW START - Manual trigger workflow for testing, bulk processing, manual syncing       |
| Sticky Note - Workflow Start    | Sticky Note         | Documentation of workflow start    | None                          | None                           | ## 🚀 WORKFLOW START - Manual trigger workflow for testing, bulk processing, manual syncing       |
| Fetch All Deals from CRM1       | HighLevel           | Fetch all deals from HighLevel CRM | When clicking ‘Execute workflow’ | Filter Recent Deal Updates1    | ## 📋 FETCH ALL DEALS - Retrieves all opportunities with full details                             |
| Sticky Note - Fetch Deals       | Sticky Note         | Documentation of fetch deals       | None                          | None                           | ## 📋 FETCH ALL DEALS - Retrieves all opportunities with full details                             |
| Filter Recent Deal Updates1     | If                  | Filters deals by last status change date | Fetch All Deals from CRM1      | Get Contact Details1 (True branch), Notify Old Deal Update1 (False branch) | ## ⏱️ DATE FILTER - Splits deals by update date cutoff                                           |
| Sticky Note - Date Filter       | Sticky Note         | Documentation of date filter logic | None                          | None                           | ## ⏱️ DATE FILTER - Splits deals by update date cutoff                                           |
| Get Contact Details1            | HighLevel           | Fetch contact details for deals    | Filter Recent Deal Updates1 (True branch) | Create ClickUp Task1           | ## 👤 GET CONTACT INFO - Retrieves full contact details                                           |
| Sticky Note - Get Contact       | Sticky Note         | Documentation of contact fetch     | None                          | None                           | ## 👤 GET CONTACT INFO - Retrieves full contact details                                           |
| Create ClickUp Task1            | ClickUp             | Create task in ClickUp for recent deals | Get Contact Details1          | None                           | ## ✅ CREATE CLICKUP TASK - Creates actionable task with contact and deal context                 |
| Sticky Note - Create Task       | Sticky Note         | Documentation of task creation     | None                          | None                           | ## ✅ CREATE CLICKUP TASK - Creates actionable task with contact and deal context                 |
| Notify Old Deal Update1         | Slack               | Notify Slack user about old deals  | Filter Recent Deal Updates1 (False branch) | None                           | ## 💬 SLACK ALERT (OLD DEALS) - Sends alert for outdated deal updates                            |
| Sticky Note - Slack Alert       | Sticky Note         | Documentation of Slack alert       | None                          | None                           | ## 💬 SLACK ALERT (OLD DEALS) - Sends alert for outdated deal updates                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger:**
   - Add a **Manual Trigger** node.
   - Leave default settings.
   - This node starts the workflow on-demand.

2. **Add HighLevel Node to Fetch Deals:**
   - Add a **HighLevel** node.
   - Set Resource: `opportunity`.
   - Set Operation: `getAll`.
   - Enable `Return All` to true (fetch all deals).
   - Connect output of Manual Trigger to this node.
   - Configure HighLevel OAuth2 credentials with proper API access.

3. **Add If Node to Filter Deals by Date:**
   - Add an **If** node.
   - Condition: Check if `$json.lastStatusChangeAt` is after or equal to `2025-09-30T00:00:00`.
   - Use expression: `={{ $json.lastStatusChangeAt }} >= '2025-09-30T00:00:00'`.
   - Connect output of HighLevel fetch deals node to this If node.

4. **Add HighLevel Node to Get Contact Details (True Branch):**
   - Add a **HighLevel** node.
   - Set Resource: `contact`.
   - Set Operation: `get`.
   - Set `contactId` parameter to `={{ $json.contactId }}`.
   - Connect True output of If node to this node.
   - Use same HighLevel OAuth2 credentials.

5. **Add ClickUp Node to Create Task (True Branch):**
   - Add a **ClickUp** node.
   - Set Operation: `create task`.
   - Set List ID: `901611225384`.
   - Set Team ID: `90161261705`.
   - Set Space ID: `90165174252`.
   - Enable Folderless mode.
   - Set Task Name with expression:  
     ```
     Contact: {{ $json.fullNameLowerCase }} 
     Location id:{{ $json.locationId }}
     Changed Status recently
     ```
   - Connect output of Get Contact Details node to this node.
   - Configure ClickUp API credentials.

6. **Add Slack Node to Notify Old Deals (False Branch):**
   - Add **Slack** node.
   - Set operation to send message.
   - Configure message text as:  
     `Hi, your lead ID: {{ $json.id }} was updated before the cutoff date`.
   - Set recipient user to Slack user ID `U09E375JZPA` (n8n_workspace user).
   - Connect False output of If node to Slack node.
   - Configure Slack OAuth2 credentials.

7. **Add Sticky Notes as Documentation:**
   - Add sticky notes to document each block:
     - Workflow start
     - Fetch deals explanation
     - Date filter logic
     - Get contact explanation
     - Create task explanation
     - Slack alert explanation

8. **Verify Connections:**
   - Manual Trigger -> Fetch All Deals from CRM1
   - Fetch All Deals -> Filter Recent Deal Updates1
   - Filter True -> Get Contact Details1 -> Create ClickUp Task1
   - Filter False -> Notify Old Deal Update1

9. **Test Workflow:**
   - Manually execute workflow.
   - Confirm tasks are created in ClickUp for recent deals.
   - Confirm Slack messages sent for old deals.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automates synchronization between HighLevel CRM deals and ClickUp, with Slack alerts for outdated deals.             | Workflow purpose                                                                                 |
| Manual execution useful for bulk processing and testing.                                                                       | Sticky Note - Workflow Start                                                                     |
| HighLevel API documentation: https://developers.gohighlevel.com/docs/api-reference#opportunities                               | HighLevel API reference                                                                          |
| ClickUp API documentation: https://clickup.com/api                                                                          | ClickUp API reference                                                                            |
| Slack API message sending guide: https://api.slack.com/messaging/sending                                                       | Slack API reference                                                                             |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with the current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.*