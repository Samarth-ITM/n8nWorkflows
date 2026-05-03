Automatically Create Google Tasks from Gmail Labeled Emails

https://n8nworkflows.xyz/workflows/automatically-create-google-tasks-from-gmail-labeled-emails-3768


# Automatically Create Google Tasks from Gmail Labeled Emails

### 1. Workflow Overview

This workflow automates the creation of Google Tasks from new Gmail emails labeled "To-Do." It is designed for individuals and teams aiming to enhance productivity by converting important emails directly into actionable tasks without manual intervention.

**Target Use Cases:**  
- Automatically tracking follow-ups from emails marked as "To-Do."  
- Reducing manual task creation from emails.  
- Streamlining task management integrated with Gmail and Google Tasks.

**Logical Blocks:**  
- **1.1 Input Reception:** Watches Gmail for new emails labeled "To-Do."  
- **1.2 Task Creation:** Creates Google Tasks using email data.  
- **1.3 Documentation & Setup Guidance:** Provides user instructions and setup notes via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block continuously monitors Gmail for new emails with the label "To-Do" using a trigger node. It initiates the workflow when such emails arrive.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**  

- **Gmail Trigger**  
  - Type: Trigger node for Gmail  
  - Role: Watches Gmail inbox for new emails matching a specific label filter.  
  - Configuration:  
    - Filter query: `label:To-Do` (only emails tagged with "To-Do" label trigger the workflow)  
    - Polling interval: Every minute (checks for new emails every minute)  
    - Authentication: Uses a Google Service Account credential for OAuth2 authentication.  
  - Key Expressions/Variables: None beyond the filter query.  
  - Input Connections: None (trigger node).  
  - Output Connections: Connects to the Google Tasks node.  
  - Version Requirements: Uses node version 1.2; requires n8n version supporting Gmail Trigger with service account authentication.  
  - Potential Failures:  
    - Authentication errors if OAuth2 credentials expire or lack required scopes.  
    - Gmail API rate limits or connectivity issues.  
    - Label "To-Do" missing in Gmail will result in no triggers firing.  
  - Sub-workflow: None.

---

#### 1.2 Task Creation

**Overview:**  
This block creates a new task in Google Tasks for each triggered email, using the email subject as the task title, the email snippet as notes, and setting a due date 24 hours after the email is received.

**Nodes Involved:**  
- Google Tasks

**Node Details:**  

- **Google Tasks**  
  - Type: Action node for Google Tasks API  
  - Role: Creates a new task in the authenticated user's Google Tasks list.  
  - Configuration:  
    - Title: Set dynamically from the email subject (`{{$json["subject"]}}`).  
    - Notes: Set dynamically from the email snippet (`{{$json["snippet"]}}`).  
    - Due Date: Set to 24 hours after the current time (`{{$now.plus(1, day).toLocaleString()}}`).  
  - Key Expressions/Variables:  
    - `{{$json["subject"]}}` — extracts email subject from trigger data.  
    - `{{$json["snippet"]}}` — extracts email snippet (short preview).  
    - `{{$now.plus(1, day).toLocaleString()}}` — calculates due date one day ahead.  
  - Input Connections: Receives input from Gmail Trigger node.  
  - Output Connections: None (end node).  
  - Version Requirements: Uses node version 1; requires OAuth2 credentials for Google Tasks.  
  - Potential Failures:  
    - Authentication errors if OAuth2 token invalid or expired.  
    - API rate limits or quota exceeded errors.  
    - Invalid date formatting causing task creation failure.  
  - Sub-workflow: None.

---

#### 1.3 Documentation & Setup Guidance

**Overview:**  
This block provides visual instructions and setup notes to assist users in configuring and understanding the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**  

- **Sticky Note**  
  - Type: Informational node (visual aid)  
  - Role: Describes the workflow purpose and high-level function.  
  - Configuration: Content explains the workflow’s function: "Create Todo in Google Tasks whenever receives new email with 'To Do' label."  
  - Input/Output: None (standalone).  
  - Version Requirements: None.  
  - Potential Failures: None.

- **Sticky Note1**  
  - Type: Informational node (visual aid)  
  - Role: Provides setup instructions including Gmail label creation and OAuth2 credential connection.  
  - Configuration: Content includes:  
    - Creating Gmail label "To-Do" if missing.  
    - Connecting Gmail and Google Tasks accounts via OAuth2 in n8n.  
    - Granting necessary API access scopes.  
  - Input/Output: None (standalone).  
  - Version Requirements: None.  
  - Potential Failures: None.

---

### 3. Summary Table

| Node Name     | Node Type               | Functional Role                     | Input Node(s)   | Output Node(s) | Sticky Note                                                                                 |
|---------------|-------------------------|-----------------------------------|-----------------|----------------|---------------------------------------------------------------------------------------------|
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Watches Gmail for "To-Do" emails  | None            | Google Tasks   |                                                                                             |
| Google Tasks  | n8n-nodes-base.googleTasks  | Creates Google Task from email    | Gmail Trigger   | None           |                                                                                             |
| Sticky Note   | n8n-nodes-base.stickyNote   | Workflow purpose description      | None            | None           | 📦 📦 New Email → Create Todo in Google Tasks. Create Todo in Google Tasks whenever receives new email with "To Do" label. |
| Sticky Note1  | n8n-nodes-base.stickyNote   | Setup instructions and requirements | None            | None           | Required Setup: Make sure the Gmail label "To-Do" exists. Connect Gmail and Google Tasks via OAuth2. Grant necessary access scopes. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Add a new node of type **Gmail Trigger**.  
   - Configure the trigger to watch for new emails with the label "To-Do": set filter query to `label:To-Do`.  
   - Set polling interval to every 1 minute.  
   - Authenticate using a Google Service Account OAuth2 credential connected to your Gmail account.  
   - Position this node as the workflow entry point.

2. **Create Google Tasks Node**  
   - Add a new node of type **Google Tasks**.  
   - Connect the output of the Gmail Trigger node to this node’s input.  
   - Configure the task creation parameters:  
     - Title: Use expression `{{$json["subject"]}}` to set the task title from the email subject.  
     - Notes: Use expression `{{$json["snippet"]}}` to add the email snippet as notes.  
     - Due Date: Use expression `{{$now.plus(1, day).toLocaleString()}}` to set the due date 24 hours from now.  
   - Authenticate using OAuth2 credentials connected to your Google Tasks account.

3. **Add Sticky Note for Workflow Description**  
   - Add a **Sticky Note** node.  
   - Enter content describing the workflow purpose:  
     ```
     📦 📦 New Email → Create Todo in Google Tasks  
     Create Todo in Google Tasks whenever receives new email with "To Do" label.
     ```  
   - Position it near the Gmail Trigger node for visibility.

4. **Add Sticky Note for Setup Instructions**  
   - Add another **Sticky Note** node.  
   - Enter setup instructions:  
     ```
     Required Setup:  
     Make sure the Gmail label "To-Do" exists. (Create it manually in Gmail if needed.)  
     Connect your Gmail and Google Tasks accounts via OAuth2 in n8n credentials.  
     Grant necessary access scopes to read emails and manage tasks.
     ```  
   - Position it near the bottom or side for easy reference.

5. **Activate the Workflow**  
   - Save and activate the workflow to begin automatic task creation.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To customize which emails trigger task creation, modify the Gmail label in the Gmail Trigger node. | Workflow customization instructions.                                                           |
| To adjust task due date timing, modify the expression in the Google Tasks node’s dueDate field.     | Example: `{{$now.add(2, 'days').toISOString()}}` for 2 days instead of 1.                       |
| Ensure OAuth2 credentials have scopes for Gmail read access and Google Tasks management.            | Credential setup requirement.                                                                   |
| Gmail label "To-Do" must exist in Gmail; create it manually if missing.                             | Gmail account setup prerequisite.                                                              |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the workflow that automatically creates Google Tasks from Gmail emails labeled "To-Do."