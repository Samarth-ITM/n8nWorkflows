🚨 Report n8n workflow errors directly to your email

https://n8nworkflows.xyz/workflows/---report-n8n-workflow-errors-directly-to-your-email-2160


# 🚨 Report n8n workflow errors directly to your email

### 1. Workflow Overview

This n8n workflow is designed to capture errors occurring in other workflows and send immediate email alerts to a designated recipient. Its primary use case is in production environments where quick awareness of workflow failures is critical for timely resolution.

The workflow is logically divided into two blocks:

- **1.1 Error Detection:** Listens for any error events triggered by connected workflows.
- **1.2 Notification Dispatch:** Sends a detailed email alert about the error to a predefined email address using Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Detection

**Overview:**  
This block acts as the entry point. It triggers automatically whenever an error occurs in any workflow that has this error workflow linked as its error handler.

**Nodes Involved:**  
- On Error

**Node Details:**

- **Node Name:** On Error  
- **Type and Technical Role:** `Error Trigger` node; listens for workflow error events and starts this workflow when an error is detected.  
- **Configuration Choices:** Default configuration; no parameters needed as it inherently triggers on error events.  
- **Key Expressions or Variables Used:** None; it receives the error context automatically.  
- **Input and Output Connections:** No input; outputs error data to the next node.  
- **Version-Specific Requirements:** Available in n8n from version 0.139.0 and above.  
- **Edge Cases / Potential Failures:**  
  - If not linked properly to other workflows as their error handler, it will never trigger.  
  - Errors in this node itself are unlikely because it is event-driven with minimal config.  
- **Sub-Workflow Reference:** This node triggers this entire error notification workflow when an error occurs in any other workflow.

#### 1.2 Notification Dispatch

**Overview:**  
Sends an email notification to a configured recipient with detailed information about the workflow error, including workflow name, execution URL, last node executed, and the error message and stack trace.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Node Name:** Gmail  
- **Type and Technical Role:** Gmail node; sends email notifications via Gmail SMTP using OAuth2 credentials.  
- **Configuration Choices:**  
  - `sendTo`: Placeholder text "SET YOUR EMAIL HERE" to be replaced by the user with the target recipient email.  
  - `subject`: Uses expressions to dynamically include the workflow name causing the error.  
  - `message`: Composes a multiline plaintext email containing:  
    - Workflow name  
    - Execution URL (link to the error execution in n8n)  
    - The last node executed before the error  
    - The error message and stack trace  
  - `emailType`: Text (plaintext email).  
  - OAuth2 credentials linked for Gmail authentication.  
- **Key Expressions or Variables Used:**  
  - `{{$json["workflow"]["name"]}}` for workflow name  
  - `{{ $json.execution.url }}` for execution link  
  - `{{ $json.execution.lastNodeExecuted }}` for last node executed  
  - `{{ $json.execution.error.message }}` and `{{ $json.execution.error.stack }}` for error details  
- **Input and Output Connections:** Receives error data from the `On Error` node; no further output.  
- **Version-Specific Requirements:** Gmail OAuth2 credential setup requires n8n version supporting OAuth2 workflows (generally 0.130+).  
- **Edge Cases / Potential Failures:**  
  - Email sending can fail due to invalid or expired OAuth2 credentials.  
  - If the target email is not set or incorrectly set, no email will be sent.  
  - Expression evaluation failures if input JSON structure changes or is incomplete.  
  - Gmail API rate limits or quota issues could block sending.  
- **Sub-Workflow Reference:** This node acts as the notification sender within this error workflow.

---

### 3. Summary Table

| Node Name   | Node Type       | Functional Role          | Input Node(s) | Output Node(s) | Sticky Note                                  |
|-------------|-----------------|-------------------------|---------------|----------------|----------------------------------------------|
| On Error    | Error Trigger   | Detect workflow errors  | -             | Gmail          |                                              |
| Gmail       | Gmail           | Send error email alert  | On Error      | -              | 👆🏽 Set target email here                     |
| Sticky Note3| Sticky Note     | Setup instructions      | -             | -              | 👨‍🎤 Setup 1. Add your Gmail creds 2. Add your target email 2. Add this error workflow to other workflows https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow |
| Sticky Note | Sticky Note     | Email target reminder   | -             | -              | 👆🏽 Set target email here                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Error Trigger Node**  
   - Add a new node of type **Error Trigger**.  
   - Name it `On Error`.  
   - No parameters need setup.  
   - This node will listen for errors from other workflows that link to this workflow as their error workflow.

2. **Create the Gmail Node**  
   - Add a **Gmail** node connected from `On Error`.  
   - Name it `Gmail`.  
   - Under **Credentials**, select or create a new Gmail OAuth2 credential with the appropriate Gmail account.  
   - Set **Send To** field to the target email address that should receive error alerts (replace "SET YOUR EMAIL HERE").  
   - Set **Subject** field to:  
     ```
     =🚨 Error in workflow: {{ $json.workflow.name }}
     ```  
   - Set **Message** field to the following expression (plaintext):  
     ```
     =⚠️ Workflow {{$json["workflow"]["name"]}} failed to run!
     You can find the execution here: {{ $json.execution.url }}

     error message from node {{ $json.execution.lastNodeExecuted }}: {{ $json.execution.error.message }}

     {{ $json.execution.error.stack }}
     ```  
   - Set **Email Type** to `text`.  
   - Save node.

3. **Connect Nodes**  
   - Connect the output of `On Error` node to the input of `Gmail` node.

4. **Add Sticky Notes for User Guidance** (Optional but recommended)  
   - Add a sticky note near the top with setup instructions:  
     ```
     👨‍🎤 Setup
     1. Add your Gmail creds
     2. Add your target email
     3. Add this error workflow to other workflows
     https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow
     ```  
   - Add a sticky note near the Gmail node reminding where to set the target email:  
     ```
     👆🏽 Set target email here
     ```

5. **Save and Activate the Workflow**  
   - Save the workflow with a descriptive name, e.g., "🚨 Report n8n workflow errors directly to your email".  
   - In other workflows where you want to catch errors, set this workflow as their **Error Workflow** in the workflow settings.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| For detailed instructions on linking this error workflow to other workflows, refer to n8n official docs on error handling.                                | https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow                         |
| This workflow requires Gmail OAuth2 credentials configured in n8n for sending email notifications securely without username/password in plaintext.          | n8n Gmail node documentation and OAuth2 credential setup                                              |
| Use this workflow as a template to customize email formatting or extend error notifications with additional integrations (e.g., Slack, SMS).                | n8n community forums and marketplace for example workflows                                             |
| Ensure the user email address in the Gmail node is updated to the actual recipient's email to receive alerts.                                              | Critical for proper alert delivery                                                                    |

---

This completes the comprehensive technical documentation for the “🚨 Report n8n workflow errors directly to your email” workflow, enabling full understanding, reproduction, and troubleshooting.