Send a message on Mattermost when a workflow is updated

https://n8nworkflows.xyz/workflows/send-a-message-on-mattermost-when-a-workflow-is-updated-1059


# Send a message on Mattermost when a workflow is updated

### 1. Workflow Overview

This workflow is designed to automatically send a notification message to a Mattermost channel whenever a specified workflow within n8n is updated. It targets users who want real-time alerts about workflow changes, facilitating collaboration and change tracking.

The workflow consists of two main logical blocks:

- **1.1 Workflow Update Detection:** Uses the Workflow Trigger node to listen for workflow update events.
- **1.2 Notification Dispatch:** Sends a message to a Mattermost channel notifying about the workflow update.

There is also an inactive Webhook → Set node path present but not connected to the main notification flow, possibly for testing or future extension.

---

### 2. Block-by-Block Analysis

#### 1.1 Workflow Update Detection

- **Overview:**  
  Detects when the current workflow is updated and triggers the subsequent notification process.

- **Nodes Involved:**  
  - Workflow Trigger

- **Node Details:**  

  - **Workflow Trigger**  
    - *Type & Role:* Event trigger node specific to workflow lifecycle events; initiates the workflow on update events.  
    - *Configuration:* Set to listen for the "update" event only, meaning it activates whenever the workflow is modified.  
    - *Expressions / Variables:* None (event-driven).  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to the Mattermost node.  
    - *Version Requirements:* n8n version that supports workflowTrigger node (available in recent versions).  
    - *Edge Cases / Failures:*  
      - Will not trigger if updates are made to other workflows.  
      - Requires appropriate permissions to detect workflow updates.  
      - Possible delays if n8n server is under heavy load.  
    - *Sub-workflow:* None.

#### 1.2 Notification Dispatch

- **Overview:**  
  Sends a message to a specified Mattermost channel informing that the workflow was updated.

- **Nodes Involved:**  
  - Mattermost

- **Node Details:**  

  - **Mattermost**  
    - *Type & Role:* Integration node to send messages to Mattermost channels.  
    - *Configuration:*  
      - Message text uses an expression to dynamically include the updated workflow’s name: `"The workflow {{$workflow.name}}, was updated."`  
      - Sends to channel ID `"toyi3uoycf8rirtm7d5jm15sso"`.  
      - No attachments configured.  
      - Uses stored credentials named `"Mattermost Credentials"`.  
    - *Expressions / Variables:* Uses `{{$workflow.name}}` to insert the updated workflow's name.  
    - *Inputs:* Connected from Workflow Trigger node.  
    - *Outputs:* None (end node).  
    - *Version Requirements:* Requires n8n version with Mattermost node support and valid Mattermost API credentials.  
    - *Edge Cases / Failures:*  
      - Failure if credentials are invalid or expired.  
      - Channel ID must exist and be accessible by the credentials.  
      - Network or API rate limits could delay or block message sending.  
    - *Sub-workflow:* None.

#### Additional Nodes (Not connected in main flow)

- **Webhook**  
  - *Type:* Standard HTTP Webhook node.  
  - *Role:* Currently not connected in the main flow; likely for testing or future extension.  
  - *Config:* Uses a unique path ID as webhook path.  
  - *Outputs:* Connected to Set node.  
  - *Edge Cases:* Unused currently.

- **Set**  
  - *Type:* Data transformation node.  
  - *Role:* Sets a static string message `"Hello!"` but does not feed into the main workflow path.  
  - *Inputs:* From Webhook node.  
  - *Outputs:* None connected further.  
  - *Edge Cases:* Inactive for current logic.

---

### 3. Summary Table

| Node Name         | Node Type                  | Functional Role                | Input Node(s)    | Output Node(s)       | Sticky Note                                                      |
|-------------------|----------------------------|-------------------------------|------------------|----------------------|-----------------------------------------------------------------|
| Webhook           | Webhook                    | Receive external HTTP requests | None             | Set                  |                                                                 |
| Set               | Set                        | Prepare static message         | Webhook          | None                 |                                                                 |
| Mattermost        | Mattermost                 | Send notification message      | Workflow Trigger | None                 |                                                                 |
| Workflow Trigger  | Workflow Trigger           | Trigger workflow on update     | None             | Mattermost           | The Workflow Trigger node will trigger the workflow when updated. <br> The Mattermost node sends notification on update. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `Workflow Trigger` node:**
   - Type: `Workflow Trigger`
   - Parameters:
     - Events: Select only `update`
   - Purpose: To trigger this workflow whenever it is updated.
   - No credentials needed.

3. **Add a `Mattermost` node:**
   - Type: `Mattermost`
   - Credentials: Select or create `Mattermost Credentials` (requires Mattermost API token with permission to post in the target channel).
   - Parameters:
     - Channel ID: Enter the target channel ID (e.g., `"toyi3uoycf8rirtm7d5jm15sso"`)
     - Message: Use an expression to include the workflow name, e.g.,  
       `"The workflow {{$workflow.name}}, was updated."`
     - Attachments: Leave empty.
     - Other options: Leave default.

4. **Connect the `Workflow Trigger` node’s output to the `Mattermost` node’s input.**

5. *(Optional)* Add a `Webhook` node (if needed for testing or extension):
   - Type: `Webhook`
   - Parameters:
     - Path: Use a unique identifier or auto-generate.
   - Connect to a `Set` node to prepare a message if needed.
   - This path is not required for the core functionality.

6. **Save the workflow and activate it.**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                     |
|----------------------------------------------------------------------------------------------|-----------------------------------|
| Screenshot available as `workflow-screenshot` (fileId:487) shows the workflow layout         | Workflow documentation screenshot |
| Workflow requires valid Mattermost API credentials with post permission in the target channel | Credential setup in n8n            |
| Refer to n8n docs for `Workflow Trigger` node for event types and usage                      | https://docs.n8n.io/nodes/n8n-nodes-base.workflowTrigger/ |
| Instructional note: "The Workflow Trigger node will trigger the workflow when updated."      | User-provided description          |
| Instructional note: "Mattermost node sends a message on workflow update."                     | User-provided description          |

---

This document fully describes the "Send a message on Mattermost when a workflow is updated" n8n workflow, enabling reproduction, modification, and troubleshooting.