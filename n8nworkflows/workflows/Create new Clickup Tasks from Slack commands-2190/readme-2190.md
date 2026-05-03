Create new Clickup Tasks from Slack commands

https://n8nworkflows.xyz/workflows/create-new-clickup-tasks-from-slack-commands-2190


# Create new Clickup Tasks from Slack commands

### 1. Workflow Overview

This workflow automates the creation of ClickUp tasks directly from Slack slash commands, streamlining task management for teams by removing the need for manual entry into ClickUp. It listens for specific Slack commands, extracts task details from the command text, and creates tasks in a designated ClickUp list and workspace.

The workflow’s main logical blocks are:

- **1.1 Slack Command Reception:** Receives and validates incoming slash commands from Slack via webhook.
- **1.2 Data Extraction and Preparation:** Extracts relevant data from the Slack command payload and prepares it for ClickUp task creation.
- **1.3 ClickUp Task Creation:** Uses the ClickUp API to create a new task using the extracted data.
- **1.4 Response Handling:** Sends a confirmation response back to Slack indicating successful task creation.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Command Reception

- **Overview:**  
Receives HTTP POST requests from Slack slash commands via a configured webhook endpoint. This node acts as the entry point for the workflow and waits for Slack to send command data.

- **Nodes Involved:**  
  - Receives slack command

- **Node Details:**

  - **Receives slack command**  
    - Type: Webhook  
    - Role: Listens for Slack slash command POST requests.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `09d30853-66a3-4494-ba4b-115d28402811/slackcommand` (unique webhook URL path)  
      - Response Mode: Uses a separate response node to handle replies (responseNode)  
    - Inputs: External HTTP POST from Slack  
    - Outputs: Passes Slack payload JSON forward  
    - Edge Cases:  
      - Invalid Slack request payloads (missing expected body parameters)  
      - Unauthorized requests if Slack signing secrets are not validated (not configured here)  
      - Network timeouts or webhook URL misconfiguration  
    - Notes: Must be publicly accessible URL for Slack to POST slash commands

#### 2.2 Data Extraction and Preparation

- **Overview:**  
Extracts key fields from the Slack command payload (channel name, command, user name, task text) and sets them as workflow variables for downstream use.

- **Nodes Involved:**  
  - Set your nodes

- **Node Details:**

  - **Set your nodes**  
    - Type: Set  
    - Role: Maps Slack webhook JSON payload fields into named variables for clarity and usage.  
    - Configuration:  
      - Assigns:  
        - `channel_name` from `body.channel_name`  
        - `command` from `body.command`  
        - `user_name` from `body.user_name`  
        - `text` from `body.text` (contains the task description/title)  
    - Inputs: Payload from webhook node  
    - Outputs: JSON with assigned variables for next node  
    - Edge Cases:  
      - Missing or malformed `text` field may cause empty task names  
      - Unexpected Slack payload structure changes  
    - Version Notes: Uses version 3.3 of Set node for stable assignment features

#### 2.3 ClickUp Task Creation

- **Overview:**  
Creates a new task in ClickUp using the extracted task text as both title and content. The task is assigned to a predefined list, team, and space, with OAuth2 authentication.

- **Nodes Involved:**  
  - Create new clickup task

- **Node Details:**

  - **Create new clickup task**  
    - Type: ClickUp  
    - Role: Calls ClickUp API to create a new task.  
    - Configuration:  
      - Target List ID: `900900727522`  
      - Team ID: `9009074051`  
      - Space ID: `90090146907`  
      - Folderless: true (task not placed in a folder)  
      - Task Name: Set dynamically from `text` extracted from Slack command  
      - Additional Fields:  
        - Content (description) set to same `text` value  
        - Assignees: Empty array (no automatic assignment)  
      - Authentication: OAuth2 (configured credential named "ClickUp account")  
    - Inputs: JSON from Set node with task text  
    - Outputs: ClickUp API response with task details (including task ID)  
    - Edge Cases:  
      - Invalid or expired OAuth2 tokens leading to authentication errors  
      - Invalid List, Team, or Space IDs causing API failures  
      - Empty or invalid task text causing API rejection  
      - API rate limits or network outages  
    - Version Notes: Version 1 of ClickUp node used

#### 2.4 Response Handling

- **Overview:**  
Sends a confirmation text response back to Slack indicating that the task was created, including the new task ID.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Returns an HTTP response to Slack to confirm the command was processed.  
    - Configuration:  
      - Respond with: Text response  
      - Response Body: Template string `"Task Created: ID  {{ $json.id }}"` where `id` is the ClickUp task ID from previous node  
    - Inputs: Output from ClickUp task creation node (must contain `id`)  
    - Outputs: HTTP response to Slack's slash command POST request  
    - Edge Cases:  
      - Missing `id` field if ClickUp creation failed silently  
      - Timeout if workflow slow, causing Slack to retry or error  
    - Version Notes: Version 1

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                  |
|----------------------|-----------------------|-------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------|
| Receives slack command| Webhook               | Receives Slack slash commands | —                     | Set your nodes       |                                                                                              |
| Set your nodes       | Set                   | Extracts Slack payload fields | Receives slack command | Create new clickup task |                                                                                              |
| Create new clickup task | ClickUp               | Creates task in ClickUp       | Set your nodes        | Respond to Webhook    |                                                                                              |
| Respond to Webhook   | Respond to Webhook     | Sends confirmation back to Slack | Create new clickup task | —                    |                                                                                              |
| Sticky Note          | Sticky Note            | Describes workflow purpose    | —                     | —                    | ## Create new tasks to airtable from a slack command                                         |
| Sticky Note2         | Sticky Note            | Contains detailed workflow description and setup steps | —                     | —                    | ## Create new Clickup Tasks from Slack commands This workflow aims to make it easy to create new tasks on Clickup from normal Slack messages using simple slack command. For example We can have a slack command as /newTask Set task to update new contacts on CRM and assign them to the sales team This will have an new task on Clickup with the same title and description on Clickup For most teams, getting tasks from Slack to Clickup involves manually entering the new tasks into Clickup. What if we could do this with a simple slash command? ## Step 1 The first step is to Create an endpoint URL for your slack command by creating an events API from the link [below] https://api.slack.com/apps/) ## STEP 2 Next step is defining the endpoint for your URL Create a new webhook endpoint from your n8n with a POST and paste the endpoint URL to your event API. This will send all slash commands associated with the Slash to the desired endpoint Once you have tested the webhook slash command is working with the webhook, create a new Clickup API that can be used to create new tasks in ClickUp This workflow creates a new task with the start dates on Clikup that can be assigned to the respective team members More details about the document setup can be found on this document [below](https://docs.google.com/document/d/1jw_UP6sXmGsIMktW0Z-b-yQB1leDLatUY2393bA4z8s/edit?usp=sharing) #### Happy Productivity |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Slack Command Webhook Node:**  
   - Add a **Webhook** node named `Receives slack command`.  
   - Set HTTP Method to `POST`.  
   - Define the path as a unique identifier (e.g., `your_unique_id/slackcommand`).  
   - Set Response Mode to `responseNode` to defer the response.  
   - Save and activate the webhook to obtain a public URL.

2. **Create the Data Extraction Node:**  
   - Add a **Set** node named `Set your nodes`.  
   - Connect the webhook node output to this node.  
   - Configure to assign the following variables from the webhook JSON:  
     - `channel_name` = `{{$json["body"]["channel_name"]}}`  
     - `command` = `{{$json["body"]["command"]}}`  
     - `user_name` = `{{$json["body"]["user_name"]}}`  
     - `text` = `{{$json["body"]["text"]}}`  
   - This prepares task-related data for the ClickUp node.

3. **Create the ClickUp Task Node:**  
   - Add a **ClickUp** node named `Create new clickup task`.  
   - Connect the output of the Set node to this node.  
   - Configure the node with:  
     - Authentication: OAuth2 (create and select the ClickUp OAuth2 credential).  
     - Team ID: `9009074051` (replace with your actual team ID).  
     - Space ID: `90090146907` (replace with your actual space ID).  
     - List ID: `900900727522` (replace with your actual list ID).  
     - Folderless: `true`.  
     - Name: Set to `{{$json["text"]}}` to use the Slack command text as task title.  
     - Additional Fields:  
       - Content: `{{$json["text"]}}` for task description.  
       - Assignees: Leave empty or configure as needed.  

4. **Create the Response Node:**  
   - Add a **Respond to Webhook** node named `Respond to Webhook`.  
   - Connect output of the ClickUp node to this node.  
   - Set Response Type to `Text`.  
   - Set Response Body to: `Task Created: ID {{ $json.id }}` (maps ClickUp task ID).  
   - This sends confirmation back to Slack confirming task creation.

5. **Connect nodes in order:**  
   - `Receives slack command` → `Set your nodes` → `Create new clickup task` → `Respond to Webhook`.

6. **Configure Slack Application and Slash Command:**  
   - Create a Slack app at https://api.slack.com/apps/.  
   - Define a Slash Command (e.g., `/newTask`) with the webhook URL pointing to the n8n Webhook URL from step 1.  
   - Deploy and test the slash command in Slack.

7. **Test the Workflow:**  
   - Use the slash command in Slack with a test task description.  
   - Confirm the task appears in ClickUp and Slack returns the confirmation message.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| To create the Slack slash command endpoint, refer to Slack’s API documentation for creating apps and slash commands: https://api.slack.com/apps/                                                                                                                                                                                                                                                         | Slack API Documentation                                                                                        |
| Workflow description and setup instructions with screenshots are available in the Google Document: https://docs.google.com/document/d/1jw_UP6sXmGsIMktW0Z-b-yQB1leDLatUY2393bA4z8s/edit?usp=sharing                                                                                                                                                                                                         | Project Documentation                                                                                           |
| OAuth2 credentials for ClickUp must be created and authorized to allow API access. Ensure scopes include task creation permissions.                                                                                                                                                                                                                                                                        | ClickUp OAuth2 Setup                                                                                            |
| This workflow assumes a folderless list setup in ClickUp. Adjust folder and space IDs as per your ClickUp organization structure if needed.                                                                                                                                                                                                                                                             | ClickUp Workspace Setup                                                                                         |
| Slack slash commands require a publicly accessible webhook URL. If running n8n locally, use a tunneling service like ngrok for testing.                                                                                                                                                                                                                                                                  | n8n Webhook Endpoint Exposure                                                                                   |
| For enhanced security, consider validating Slack requests using signing secrets (not implemented here).                                                                                                                                                                                                                                                                                                   | Slack Request Verification Best Practices                                                                       |
| Sticky notes in the workflow contain detailed explanations and setup steps for user reference inside n8n editor.                                                                                                                                                                                                                                                                                          | Embedded Workflow Documentation                                                                                 |

---

This document provides a thorough understanding and stepwise reconstruction of the "Create new Clickup Tasks from Slack commands" workflow, enabling advanced users and AI agents to analyze, modify, or replicate it efficiently.