New TheHive Case Slack Notification Bot

https://n8nworkflows.xyz/workflows/new-thehive-case-slack-notification-bot-2577


# New TheHive Case Slack Notification Bot

---

## 1. Workflow Overview

This workflow, titled **New TheHive Case Slack Notification Bot**, integrates TheHive case management system with Slack to empower SOC analysts to manage and update case attributes directly from Slack. It streamlines SOC operations by reducing context switching, speeding up case handling, and maintaining up-to-date case data visible to the entire team.

The workflow logically consists of the following blocks:

- **1.1 TheHive Case Event Reception:** Listens for new case creations in TheHive and triggers workflow execution.
- **1.2 Formatting and Posting New Cases to Slack:** Prepares case details and posts a rich Slack message with interactive actions.
- **1.3 Slack Interaction Processing:** Receives and parses interactive Slack events (button clicks, form submissions) for case updates.
- **1.4 Case Attribute Update Handlers:** Handles updates for assignee, severity, status, TLP, PAP, and case closure, updating TheHive accordingly.
- **1.5 Slack Message Update:** Dynamically rebuilds and updates the Slack message to reflect changes.
- **1.6 Task Management via Slack Modal:** Enables adding tasks to cases through Slack modal dialogs, processing inputs and creating tasks in TheHive.
- **1.7 Response and Acknowledgment:** Provides HTTP responses to Slack to acknowledge interactive events promptly.

---

## 2. Block-by-Block Analysis

### 1.1 TheHive Case Event Reception

**Overview:**  
This block listens for new case creation events in TheHive and initiates the workflow.

**Nodes Involved:**
- TheHive Trigger
- Formatting Dictionaries
- HTTP Request

**Node Details:**

- **TheHive Trigger**  
  - Type: TheHiveProjectTrigger node  
  - Role: Listens to "case_create" events from TheHive via webhook.  
  - Configuration: Subscribed to `case_create` event. Uses webhook URL configured in TheHive.  
  - Inputs: External webhook event from TheHive.  
  - Outputs: Emits new case event JSON.  
  - Edge Cases: Webhook misconfiguration, TheHive downtime, payload format changes.

- **Formatting Dictionaries**  
  - Type: Set node  
  - Role: Provides emoji mappings and TheHive base URL for later formatting.  
  - Configuration: Assigns dictionaries for PAP, Severity, TLP, STATUS emojis, and TheHive URL.  
  - Input: TheHive Trigger output.  
  - Output: Enriched JSON with formatting dictionaries.  
  - Edge Cases: Hardcoded URL requires update if TheHive URL changes.

- **HTTP Request**  
  - Type: HTTP Request node  
  - Role: Looks up Slack user info by email corresponding to TheHive assignee email.  
  - Configuration: Slack API call to `users.lookupByEmail` with email from TheHive event.  
  - Input: Enriched JSON from Formatting Dictionaries.  
  - Output: Slack user profile JSON.  
  - Edge Cases: Slack API failures, missing Slack user for email, rate limits.

---

### 1.2 Formatting and Posting New Cases to Slack

**Overview:**  
Transforms TheHive case data into Slack message blocks and posts a rich interactive message to Slack.

**Nodes Involved:**
- Prep Fields For Slack
- Post New Case To Slack

**Node Details:**

- **Prep Fields For Slack**  
  - Type: Set node  
  - Role: Extracts and formats TheHive case fields into Slack-friendly strings and emojis.  
  - Configuration: Extracts title, case number, date created, severity, TLP, PAP, description, assignee, profile pic, tags, and status with emojis.  
  - Inputs: HTTP Request output with Slack user info and TheHive data.  
  - Outputs: JSON with formatted fields for Slack block kit.  
  - Edge Cases: Missing or malformed input fields could cause empty or incorrect formatting.

- **Post New Case To Slack**  
  - Type: Slack node  
  - Role: Posts the formatted case message to a specified Slack channel with interactive buttons.  
  - Configuration: Posts message with blocks UI including image, header, details sections, and buttons ("Close Case as False Positive", "Add a Task").  
  - Inputs: JSON from Prep Fields For Slack node.  
  - Outputs: Slack message response including channel and timestamp for future updates.  
  - Edge Cases: Slack API authentication errors, permission issues, channel misconfiguration.

---

### 1.3 Slack Interaction Processing

**Overview:**  
Receives Slack interactive events such as button clicks and form submissions, parses action types, and routes to appropriate handlers.

**Nodes Involved:**
- Receive Button Press (Webhook)
- Edit Fields (Set)
- Parse Message Type (Switch)
- Respond to Slack with 200 response

**Node Details:**

- **Receive Button Press**  
  - Type: Webhook node  
  - Role: Receives POST requests from Slack interactive components (buttons, selects, modals).  
  - Configuration: Webhook path `/slackthehivewebhook` with POST method and responseMode set to wait for response node.  
  - Inputs: Incoming Slack interactive event payload.  
  - Outputs: Raw Slack payload JSON.

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts payload from Slack and prepares a structured object (`response`) for downstream logic.  
  - Configuration: Assigns `response` from incoming Slack payload, includes dictionaries and TheHive URL for context.  
  - Inputs: Receive Button Press output.  
  - Outputs: JSON with extracted response data and dictionaries.

- **Parse Message Type**  
  - Type: Switch node  
  - Role: Determines interaction type by inspecting action IDs and routes to specific action handlers such as "Assign to User", "Close Case", "Update Severity", "Add Task", etc.  
  - Configuration: Multiple condition rules on `$json.response.actions[0].action_id` or `$json.response.view.type` for modals.  
  - Inputs: Edit Fields output.  
  - Outputs: Routes to specific branches based on Slack action type.

- **Respond to Slack with 200 response**  
  - Type: Respond To Webhook node  
  - Role: Immediately responds to Slack with HTTP 200 to acknowledge receipt of interaction, preventing Slack timeouts.  
  - Configuration: HTTP 200 with no data content.  
  - Inputs: Parse Message Type output.  
  - Outputs: Ends webhook response cycle.

---

### 1.4 Case Attribute Update Handlers

**Overview:**  
Processes updates for various case attributes (Assignee, Severity, Status, TLP, PAP, Close Case) by updating TheHive cases and preparing updated Slack messages.

**Nodes Involved:**
- Prep Fields For Slack - Assign / Severity / Status / TLP / PAP / Close
- Get Slack User's Email From Slack
- Update TheHive Case with new Assignee
- Update Case Severity
- Update Status in TheHive
- Update Case TLP
- Update Case PAP
- Close Case as False Positive
- Various "Case Block Rebuild" nodes for Slack message update
- Acknowledge nodes for Slack (respond to Slack with 200 or 204)

**Node Details:**

- **Prep Fields For Slack - Assign / Severity / Status / TLP / PAP / Close**  
  - Type: Set nodes  
  - Role: Extract and prepare the necessary fields from Slack message blocks and interaction state for each attribute update.  
  - Configuration: Pulls data from Slack blocks and selected options, maps emojis and descriptive labels.  
  - Inputs: Edit Fields response JSON.  
  - Outputs: JSON ready for attribute update.  
  - Edge Cases: Malformed Slack message blocks or missing fields can cause failures.

- **Get Slack User's Email From Slack**  
  - Type: Slack node  
  - Role: Retrieves Slack user profile info by user ID to get email for TheHive assignee update.  
  - Inputs: User ID from Slack interaction.  
  - Outputs: Slack user profile with email.  
  - Edge Cases: Slack API errors, user ID not found.

- **Update TheHive Case with new Assignee**  
  - Type: TheHiveProject node  
  - Role: Updates TheHive case assignee email based on Slack user email.  
  - Inputs: Prepared JSON with case ID and assignee email.  
  - Outputs: Confirmation from TheHive API.  
  - Edge Cases: Email mismatch between Slack and TheHive users causes update failure.

- **Update Case Severity / Status / TLP / PAP / Close Case as False Positive**  
  - Type: TheHiveProject node  
  - Role: Updates corresponding case attribute in TheHive.  
  - Inputs: Case ID and attribute value to update.  
  - Outputs: Confirmation from TheHive API.  
  - Edge Cases: API errors, invalid values, permissions.

- **Case Block Rebuild Nodes**  
  - Type: Set nodes  
  - Role: Rebuild Slack message blocks to reflect updated case details after attribute changes.  
  - Inputs: Updated attribute values and existing Slack message data.  
  - Outputs: JSON Slack block kit for message update.  
  - Edge Cases: JSON syntax errors if variables are missing or incorrect.

- **Acknowledge Nodes**  
  - Type: Respond To Webhook nodes  
  - Role: Send HTTP 200 or 204 responses to Slack to confirm processing of interactive events.  
  - Inputs: After update operation completes.  
  - Outputs: HTTP response to Slack.  
  - Edge Cases: Missing acknowledgment causes Slack to retry or show errors.

---

### 1.5 Slack Message Update

**Overview:**  
Combines updated interactive blocks and buttons into a final Slack message and posts an update to the original Slack message to reflect changes in real-time.

**Nodes Involved:**
- Map Actions
- Build Final Block
- Update Message with new Assignee

**Node Details:**

- **Map Actions**  
  - Type: Set node  
  - Role: Maps the rebuilt Slack blocks and dynamic action buttons into JSON strings.  
  - Inputs: Output from Case Block Rebuild nodes.  
  - Outputs: JSON with `actions` (case details blocks) and `buttons` (interactive elements).  
  - Edge Cases: Incorrect JSON formatting breaks Slack message update.

- **Build Final Block**  
  - Type: Set node  
  - Role: Combines `actions` and `buttons` into a single Slack message JSON object under `blocks`.  
  - Inputs: Map Actions output.  
  - Outputs: Final JSON Slack message payload.  
  - Edge Cases: Syntax errors if input JSON is malformed.

- **Update Message with new Assignee**  
  - Type: HTTP Request node  
  - Role: Uses Slack `chat.update` API to update the original Slack message with new blocks.  
  - Configuration: Posts to Slack with channel ID, message timestamp, and updated blocks JSON.  
  - Inputs: Build Final Block output, plus channel and timestamp from original message.  
  - Edge Cases: Slack API rate limits, message timestamp or channel invalid.

---

### 1.6 Task Management via Slack Modal

**Overview:**  
Allows users to add tasks to TheHive cases via Slack modal popups, processing modal submissions and creating TheHive tasks.

**Nodes Involved:**
- Acknowledge Modal Request to Slack
- Task Modal
- Check if Case Options
- Close Modal with 204 response
- Get Email From Slack to assign the task to in TheHive
- Add a task to TheHive

**Node Details:**

- **Acknowledge Modal Request to Slack**  
  - Type: Respond To Webhook node  
  - Role: Sends HTTP 200 immediately to Slack to acknowledge modal open request.  
  - Inputs: Parse Message Type output for "Task Modal Details".  
  - Outputs: HTTP 200 response.

- **Task Modal**  
  - Type: HTTP Request node  
  - Role: Opens a Slack modal dialog with inputs for task title, description, group, due date, assignee, and checkboxes.  
  - Configuration: Calls Slack API endpoint `views.open` using trigger_id from Slack payload.  
  - Inputs: Acknowledge Modal Request node output with Slack trigger ID and case context.  
  - Outputs: Slack API response confirming modal open.  
  - Edge Cases: Slack API errors, invalid trigger_id.

- **Check if Case Options**  
  - Type: If node  
  - Role: Checks if the Slack interaction is not just a checkbox submit with no task details, to prevent premature processing.  
  - Inputs: Parse Message Type output for modal submission.  
  - Outputs: Routes to either close modal or continue processing.

- **Close Modal with 204 response**  
  - Type: Respond To Webhook node  
  - Role: Sends HTTP 204 to Slack to close modal when no action needed.  
  - Inputs: Check if Case Options output.  
  - Outputs: HTTP 204 response.

- **Get Email From Slack to assign the task to in TheHive**  
  - Type: Slack node  
  - Role: Retrieves Slack user email for the selected assignee in the modal.  
  - Inputs: User ID from modal submission.  
  - Outputs: Slack user profile JSON.

- **Add a task to TheHive**  
  - Type: TheHiveProject node  
  - Role: Creates a new task on the specified case with task details from modal inputs.  
  - Configuration: Uses caseId, title, description, group, due date, assignee email, flag, mandatory fields.  
  - Inputs: Slack user profile and modal submission data.  
  - Outputs: Confirmation of task creation.  
  - Edge Cases: API errors, invalid assignee email, missing required fields.

---

### 1.7 Response and Acknowledgment

**Overview:**  
Ensures all Slack interactive events receive timely HTTP responses to prevent timeouts and ensure smooth user experience.

**Nodes Involved:**
- Respond positive to Slack when someone clicks a link
- No Action Needed
- Respond 204 to Slack

**Node Details:**

- **Respond positive to Slack when someone clicks a link**  
  - Type: Respond To Webhook node  
  - Role: Responds with HTTP 204 No Content to Slack after link click events to acknowledge interaction.  
  - Inputs: Routed from Parse Message Type "View Link" branch.  
  - Outputs: HTTP 204 response.

- **No Action Needed**  
  - Type: NoOp node  
  - Role: Placeholder to end workflow branch with no operation.  
  - Inputs: Respond positive to Slack output.  
  - Outputs: None.

- **Respond 204 to Slack**  
  - Type: Respond To Webhook node  
  - Role: Sends HTTP 204 to Slack to acknowledge no further action is needed.  
  - Inputs: Check if Case Options output when no details submitted.  
  - Outputs: HTTP 204 response.

---

## 3. Summary Table

| Node Name                            | Node Type                 | Functional Role                               | Input Node(s)                  | Output Node(s)                            | Sticky Note                                                                                                                                                                                                                                          |
|------------------------------------|---------------------------|-----------------------------------------------|-------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| TheHive Trigger                    | TheHiveProjectTrigger     | Listens for new case creation in TheHive      |                               | Formatting Dictionaries                   | ![theHive](https://uploads.n8n.io/templates/thehive.png) Setup TheHive 5 webhook using this node's URL.                                                                                                                                               |
| Formatting Dictionaries            | Set                       | Provides emoji dictionaries and TheHive URL   | TheHive Trigger               | HTTP Request                             |                                                                                                                                                                                                                                                      |
| HTTP Request                      | HTTP Request              | Looks up Slack user by email                   | Formatting Dictionaries       | Prep Fields For Slack                    |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack             | Set                       | Formats TheHive case data for Slack message   | HTTP Request                  | Post New Case To Slack                   |                                                                                                                                                                                                                                                      |
| Post New Case To Slack            | Slack                     | Posts formatted new case message to Slack     | Prep Fields For Slack         |                                       |                                                                                                                                                                                                                                                      |
| Receive Button Press              | Webhook                   | Receives Slack interactive events              |                               | Edit Fields                             | ![slack](https://uploads.n8n.io/templates/slack.png) Receives interactive Slack events.                                                                                                                                                               |
| Edit Fields                      | Set                       | Extracts Slack payload, prepares response data | Receive Button Press          | Parse Message Type                      |                                                                                                                                                                                                                                                      |
| Parse Message Type               | Switch                    | Routes Slack interactions by action type       | Edit Fields                  | Multiple (Respond nodes, handlers)      | ![n8n](https://uploads.n8n.io/templates/n8n.png) Simplifies routing of Slack actions.                                                                                                                                                                 |
| Respond to Slack with 200 response| Respond To Webhook        | Acknowledges Slack interaction with HTTP 200  | Parse Message Type            | Prep Fields For Slack - Assign           |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack - Assign   | Set                       | Prepares case fields for assignee update       | Respond to Slack with 200 response | Get Slack User's Email From Slack     |                                                                                                                                                                                                                                                      |
| Get Slack User's Email From Slack| Slack                     | Retrieves Slack user profile by user ID        | Prep Fields For Slack - Assign | Update TheHive Case with new Assignee   |                                                                                                                                                                                                                                                      |
| Update TheHive Case with new Assignee| TheHiveProject           | Updates TheHive case assignee                   | Get Slack User's Email From Slack | Case Slack Block Rebuild               | ![Slack](https://uploads.n8n.io/templates/slack.png) Slack and TheHive user emails must match exactly.                                                                                                                                               |
| Case Slack Block Rebuild         | Set                       | Rebuilds Slack message blocks after assignee update | Update TheHive Case with new Assignee | Map Actions                         |                                                                                                                                                                                                                                                      |
| Map Actions                     | Set                       | Maps interactive Slack blocks and buttons      | Case Slack Block Rebuild      | Build Final Block                      | ![slack](https://uploads.n8n.io/templates/slack.png) Maps user actions to Slack interactive elements.                                                                                                                                                |
| Build Final Block               | Set                       | Combines blocks and buttons into Slack message | Map Actions                  | Update Message with new Assignee       |                                                                                                                                                                                                                                                      |
| Update Message with new Assignee | HTTP Request              | Updates Slack message with new blocks           | Build Final Block            |                                       |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack - Severity | Set                       | Prepares fields for severity update             |                               | Update Case Severity                   |                                                                                                                                                                                                                                                      |
| Update Case Severity            | TheHiveProject            | Updates TheHive case severity                    | Prep Fields For Slack - Severity | Severity Case Block Rebuild1          |                                                                                                                                                                                                                                                      |
| Severity Case Block Rebuild1    | Set                       | Rebuilds Slack message blocks after severity update | Update Case Severity          | Map Actions                           |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack - Status  | Set                       | Prepares fields for status update               |                               | Update Status in TheHive              |                                                                                                                                                                                                                                                      |
| Update Status in TheHive        | TheHiveProject            | Updates TheHive case status                      | Prep Fields For Slack - Status | Status Case Block Rebuild             |                                                                                                                                                                                                                                                      |
| Status Case Block Rebuild       | Set                       | Rebuilds Slack message blocks after status update | Update Status in TheHive      | Map Actions                           |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack - TLP     | Set                       | Prepares fields for TLP update                   |                               | Update Case TLP                      |                                                                                                                                                                                                                                                      |
| Update Case TLP                | TheHiveProject            | Updates TheHive case TLP                         | Prep Fields For Slack - TLP   | TLP Case Block Rebuild               |                                                                                                                                                                                                                                                      |
| TLP Case Block Rebuild          | Set                       | Rebuilds Slack message blocks after TLP update  | Update Case TLP              | Map Actions                           |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack - PAP     | Set                       | Prepares fields for PAP update                   |                               | Update Case PAP                      |                                                                                                                                                                                                                                                      |
| Update Case PAP                | TheHiveProject            | Updates TheHive case PAP                         | Prep Fields For Slack - PAP   | PAP Case Block Rebuild               |                                                                                                                                                                                                                                                      |
| PAP Case Block Rebuild          | Set                       | Rebuilds Slack message blocks after PAP update  | Update Case PAP              | Map Actions                           |                                                                                                                                                                                                                                                      |
| Prep Fields For Slack - Close   | Set                       | Prepares fields for closing case as false positive |                               | Close Case as False Positive         |                                                                                                                                                                                                                                                      |
| Close Case as False Positive   | TheHiveProject            | Closes TheHive case as false positive            | Prep Fields For Slack - Close | Close Case Block Rebuild             |                                                                                                                                                                                                                                                      |
| Close Case Block Rebuild        | Set                       | Rebuilds Slack message blocks after case close  | Close Case as False Positive | Map Actions                           |                                                                                                                                                                                                                                                      |
| Acknowledge Close Case to Slack | Respond To Webhook        | Sends HTTP 200 to Slack after close case action | Parse Message Type           | Prep Fields For Slack - Close         |                                                                                                                                                                                                                                                      |
| Acknowledge Severity Update to Slack | Respond To Webhook    | Sends HTTP 200 to Slack after severity update   | Parse Message Type           | Prep Fields For Slack - Severity      |                                                                                                                                                                                                                                                      |
| Acknowledge PAP Update to Slack | Respond To Webhook        | Sends HTTP 200 to Slack after PAP update         | Parse Message Type           | Prep Fields For PAP Slack             |                                                                                                                                                                                                                                                      |
| Acknowledge TLP Update to Slack | Respond To Webhook        | Sends HTTP 200 to Slack after TLP update         | Parse Message Type           | Prep Fields For TLP Slack             |                                                                                                                                                                                                                                                      |
| Acknowledge Status Update to Slack | Respond To Webhook      | Sends HTTP 200 to Slack after status update      | Parse Message Type           | Prep Fields For Status Slack          |                                                                                                                                                                                                                                                      |
| Acknowledge Modal Request to Slack | Respond To Webhook      | Sends HTTP 200 to Slack to acknowledge modal open | Parse Message Type           | Task Modal                          |                                                                                                                                                                                                                                                      |
| Task Modal                     | HTTP Request              | Opens Slack modal dialog for adding a task      | Acknowledge Modal Request to Slack |                               |                                                                                                                                                                                                                                                      |
| Check if Case Options          | If                        | Checks if the modal submission is actionable    | Parse Message Type           | Close Modal with 204 response / Respond 204 to Slack |                                                                                                                                                                                                                                                      |
| Close Modal with 204 response  | Respond To Webhook        | Sends HTTP 204 to close Slack modal              | Check if Case Options        | Get Email From Slack to assign the task to in TheHive |                                                                                                                                                                                                                                                      |
| Get Email From Slack to assign the task to in TheHive | Slack   | Retrieves Slack user email to assign task       | Close Modal with 204 response | Add a task to TheHive               | ![slack](https://uploads.n8n.io/templates/slack.png) Slack user emails must match TheHive user emails for assignment.                                                                                                                               |
| Add a task to TheHive          | TheHiveProject            | Creates a new task in TheHive                     | Get Email From Slack to assign the task to in TheHive |                               |                                                                                                                                                                                                                                                      |
| Respond positive to Slack when someone clicks a link | Respond To Webhook | Sends HTTP 204 to Slack after link click         | Parse Message Type           | No Action Needed                     | ![slack](https://uploads.n8n.io/templates/slack.png) Ensure quick positive responses to maintain Slack interaction flow.                                                                                                                            |
| No Action Needed               | NoOp                      | Ends workflow branch with no operation           | Respond positive to Slack when someone clicks a link |                               |                                                                                                                                                                                                                                                      |
| Respond 204 to Slack           | Respond To Webhook        | Sends HTTP 204 to Slack when no further action is needed | Check if Case Options       |                               |                                                                                                                                                                                                                                                      |

---

## 4. Reproducing the Workflow from Scratch

1. **Create TheHive Trigger Node**  
   - Node Type: TheHiveProjectTrigger  
   - Configure event subscription to `case_create`.  
   - Obtain webhook URL and add it to TheHive settings.

2. **Create a Set Node for Formatting Dictionaries**  
   - Assign emoji dictionaries for PAP, Severity, TLP, STATUS, and TheHive base URL (e.g., `http://<TheHive-IP>:9000`).

3. **Create HTTP Request Node to Lookup Slack User by Email**  
   - Method: GET  
   - URL: `https://slack.com/api/users.lookupByEmail`  
   - Query parameter: `email` from TheHive Trigger assignee email.  
   - Authentication: Slack API credential.

4. **Create Set Node "Prep Fields For Slack"**  
   - Extract and format TheHive case fields and Slack user profile info into strings and emojis for Slack block kit.

5. **Create Slack Node "Post New Case To Slack"**  
   - Post message with blocks UI to desired Slack channel (e.g., alerts channel).  
   - Use blocks formatted from previous Set node.  
   - Configure Slack API credential.

6. **Create Webhook Node "Receive Button Press"**  
   - Path: `/slackthehivewebhook`  
   - HTTP method: POST  
   - Response mode: Response Node (wait for explicit response).

7. **Create Set Node "Edit Fields"**  
   - Extract payload from webhook JSON into a structured object.  
   - Add dictionaries and TheHive URL.

8. **Create Switch Node "Parse Message Type"**  
   - Add rules to branch by action_id or modal type:  
     - `change-assignee` → "Assign to User" branch  
     - `close_case` → "Close Case" branch  
     - `update_severity` → "Update Severity" branch  
     - `add_task` → "Add Task" branch  
     - Modal view type `modal` → "Task Modal Details" branch  
     - `update_pap` → "Update PAP" branch  
     - `update-status` → "Update Status" branch  
     - `update_tlp` → "Update TLP" branch  
     - `viewlink` → "View Link" branch

9. **Create Respond To Webhook Node**  
   - Name: "Respond to Slack with 200 response"  
   - Response code: 200  
   - Connect from Switch node output for immediate acknowledgment.

10. **For Assignee Change Branch:**  
    - Set Node "Prep Fields For Slack - Assign": Extract relevant Slack message fields and new assignee user ID.  
    - Slack Node "Get Slack User's Email From Slack": Retrieve Slack user profile by ID.  
    - TheHive Node "Update TheHive Case with new Assignee": Update case assignee with Slack user email.  
    - Set Node "Case Slack Block Rebuild": Rebuild Slack message blocks with updated assignee info.  
    - Set Node "Map Actions": Prepare interactive buttons and blocks JSON.  
    - Set Node "Build Final Block": Combine blocks and buttons into Slack message JSON.  
    - HTTP Request Node "Update Message with new Assignee": Update original Slack message using Slack API chat.update.

11. **For Close Case Branch:**  
    - Set Node "Prep Fields For Slack - Close": Prepare close case fields.  
    - TheHive Node "Close Case as False Positive": Update case status to `FalsePositive`.  
    - Set Node "Close Case Block Rebuild": Rebuild Slack message blocks.  
    - Map Actions → Build Final Block → Update Message with new Assignee (same as above).  
    - Respond To Webhook Node "Acknowledge Close Case to Slack" with HTTP 200.

12. **For Severity Update Branch:**  
    - Set Node "Prep Fields For Slack - Severity": Prepare severity update data.  
    - TheHive Node "Update Case Severity": Update severity attribute.  
    - Set Node "Severity Case Block Rebuild1": Rebuild Slack message blocks.  
    - Map Actions → Build Final Block → Update Message with new Assignee.  
    - Respond To Webhook Node "Acknowledge Severity Update to Slack".

13. **For Status Update Branch:**  
    - Set Node "Prep Fields For Slack - Status": Prepare status update data.  
    - TheHive Node "Update Status in TheHive": Update status attribute.  
    - Set Node "Status Case Block Rebuild": Rebuild Slack message blocks.  
    - Map Actions → Build Final Block → Update Message with new Assignee.  
    - Respond To Webhook Node "Acknowledge Status Update to Slack".

14. **For TLP Update Branch:**  
    - Set Node "Prep Fields For Slack - TLP": Prepare TLP update data.  
    - TheHive Node "Update Case TLP": Update TLP attribute.  
    - Set Node "TLP Case Block Rebuild": Rebuild Slack message blocks.  
    - Map Actions → Build Final Block → Update Message with new Assignee.  
    - Respond To Webhook Node "Acknowledge TLP Update to Slack".

15. **For PAP Update Branch:**  
    - Set Node "Prep Fields For PAP Slack": Prepare PAP update data.  
    - TheHive Node "Update Case PAP": Update PAP attribute.  
    - Set Node "PAP Case Block Rebuild": Rebuild Slack message blocks.  
    - Map Actions → Build Final Block → Update Message with new Assignee.  
    - Respond To Webhook Node "Acknowledge PAP Update to Slack".

16. **For Add Task Branch:**  
    - Respond To Webhook Node "Acknowledge Modal Request to Slack" (HTTP 200) to acknowledge modal open.  
    - HTTP Request Node "Task Modal": Open Slack modal with form fields for adding a task. Use `views.open` API with trigger_id.  
    - If Node "Check if Case Options": Checks if modal submission is actionable; routes accordingly.  
    - Respond To Webhook Node "Close Modal with 204 response" or "Respond 204 to Slack" for no action or closing modal.  
    - Slack Node "Get Email From Slack to assign the task to in TheHive": Get assignee email from user ID.  
    - TheHive Node "Add a task to TheHive": Create a new task with modal inputs.  
    - Respond To Webhook Node "Respond positive to Slack when someone clicks a link" (HTTP 204) for link clicks.

17. **Configure Credentials:**  
    - TheHive API credentials for all TheHiveProject nodes.  
    - Slack API credentials for all Slack and HTTP Request nodes interacting with Slack APIs.

18. **Additional Setup Notes:**  
    - Ensure Slack App has permissions for interactive components, views, user info lookup.  
    - Ensure TheHive webhook is configured to send case_create events.  
    - Match Slack user emails with TheHive user emails exactly for assignee updates.  
    - Configure Slack channel for posting case notifications.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ![theHive](https://uploads.n8n.io/templates/thehive.png) TheHive 5 webhook URL must be configured in TheHive settings to trigger the workflow.                                                                                                                                                                                                                   | https://docs.thehive-project.org                                                                |
| ![slack](https://uploads.n8n.io/templates/slack.png) Slack events subscription setup is required for interactive components to send payloads to the webhook.                                                                                                                                                                                                     | https://api.slack.com/apis/connections/events-api                                               |
| Slack user emails must match TheHive user emails exactly for successful assignee updates; no error handling exists if mismatched.                                                                                                                                                                                                                                |                                                                                                                                                                 |
| This workflow requires Slack app permissions for users:read.email, chat:write, views:write, and interactive components enabled.                                                                                                                                                                                                                                  | Slack App Configuration                                                                         |
| Adding tasks via Slack modal requires Slack OAuth token with views:write and users:read scopes.                                                                                                                                                                                                                                                                  | Slack API documentation                                                                         |
| The workflow uses advanced Slack Block Kit JSON; modifying interactive elements requires understanding of JSON and Slack Block Kit format.                                                                                                                                                                                                                      | https://api.slack.com/block-kit                                                                 |

---

This detailed analysis and reproduction guide will enable advanced users and automation agents to understand, recreate, and adapt the workflow for TheHive and Slack integration, ensuring efficient SOC case management directly from Slack.