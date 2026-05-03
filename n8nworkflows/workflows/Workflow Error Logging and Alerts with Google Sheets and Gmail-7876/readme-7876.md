Workflow Error Logging and Alerts with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/workflow-error-logging-and-alerts-with-google-sheets-and-gmail-7876


# Workflow Error Logging and Alerts with Google Sheets and Gmail

### 1. Workflow Overview

This workflow provides a centralized error logging and alerting mechanism for all workflows running within an n8n instance. It automatically triggers whenever any workflow execution encounters an error, captures detailed error data, logs this information into a Google Sheets spreadsheet, and sends an immediate email notification via Gmail to alert concerned parties.

The workflow is logically divided into three main functional blocks:

- **1.1 Error Detection:** Listens for error events triggered by any workflow execution.
- **1.2 Error Logging:** Extracts comprehensive error details and appends them as a new row in a Google Sheets document for persistent tracking.
- **1.3 Error Notification:** Sends a formatted email alert via Gmail summarizing the error details and providing a direct link to the failed execution for quick investigation.

Additional nodes provide setup instructions and contextual information in the form of sticky notes to assist users with configuration and understanding.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection Block

- **Overview:**  
  This block initiates the workflow by automatically triggering whenever any error occurs in any workflow execution within the n8n instance.

- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**

  - **Error Trigger**  
    - *Type & Role:* Trigger node specialized to activate on workflow errors.  
    - *Configuration:* Default settings; triggers on all workflow errors globally without filters.  
    - *Expressions/Variables:* Provides detailed execution and error context under `$json.execution.error` and workflow info under `$json.workflow`.  
    - *Input/Output:* No input; outputs error execution data to downstream nodes.  
    - *Version Requirements:* n8n v0.160+ recommended for full error trigger support.  
    - *Potential Failures:* Rare; possible issues if the n8n instance’s error reporting system is disabled or misconfigured.  
    - *Sub-workflow:* None.

#### 2.2 Error Logging Block

- **Overview:**  
  Takes error details from the trigger and appends a structured log entry into a Google Sheets spreadsheet to maintain a persistent error history.

- **Nodes Involved:**  
  - Google Sheets - Create Error Log

- **Node Details:**

  - **Google Sheets - Create Error Log**  
    - *Type & Role:* Google Sheets node configured to append new rows.  
    - *Configuration:*  
      - Operation: Append row to existing sheet.  
      - Target Sheet: "Sheet1" in the Google Sheets document with ID `1JprCxz5IRZzuiYs-NNkYLffQIkeTiC0sd-3CyrcOKV4`.  
      - Columns mapped:  
        - Workflow ID  
        - Workflow Name  
        - Node Name (last node executed)  
        - Node ID  
        - Timestamp (ISO string converted from error timestamp)  
        - Execution URL  
        - Log (full error stack trace)  
    - *Expressions/Variables:*  
      Uses rich expressions to extract error stack, node information, timestamps, etc. from the incoming JSON error context, e.g., `{{$json.execution.error.stack}}`, `{{$json.execution.lastNodeExecuted}}`.  
    - *Input/Output:* Receives error details from Error Trigger; outputs appended row confirmation (not used downstream).  
    - *Version Requirements:* Google Sheets node version 4.5+ supports this mapping and append operation.  
    - *Potential Failures:*  
      - Authentication issues with Google Sheets OAuth2 credential.  
      - Spreadsheet ID or sheet name misconfiguration.  
      - API rate limits or quota exhaustion.  
      - Errors if the error context data is missing or malformed.  
    - *Sub-workflow:* None.

#### 2.3 Error Notification Block

- **Overview:**  
  Sends an immediate email notification summarizing the error details, enabling quick response and troubleshooting.

- **Nodes Involved:**  
  - Gmail - Send Notification

- **Node Details:**

  - **Gmail - Send Notification**  
    - *Type & Role:* Gmail node sending a plain text email.  
    - *Configuration:*  
      - Recipient: `n8n_log_template@yopmail.com` (to be customized).  
      - Subject includes workflow name and node that caused error, e.g., `🚨 n8n Error in Workflow "{{ $json.workflow.name }}" (Node: {{ $json.execution.error.context.nodeCause }})`.  
      - Message body includes workflow name, node causing error, error type, description, execution ID, and direct execution link.  
      - Email type: plain text.  
    - *Expressions/Variables:*  
      Uses multiple expressions to format message dynamically based on error details, such as `{{$json.execution.error.description}}`, `{{$json.execution.id}}`.  
    - *Input/Output:* Receives error data from Error Trigger; outputs email send confirmation (not used downstream).  
    - *Version Requirements:* Gmail node version 2.1+ recommended for OAuth2 and templating features.  
    - *Potential Failures:*  
      - Gmail OAuth2 authentication errors or token expiration.  
      - SMTP sending limits or blocking by Gmail.  
      - Invalid recipient email address.  
    - *Sub-workflow:* None.

#### 2.4 Informational Sticky Notes Block

- **Overview:**  
  Provides setup instructions, workflow description, and process overview to users for easier understanding and configuration.

- **Nodes Involved:**  
  - Sticky Note4  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**

  - **Sticky Note4**  
    - Displays contact information and promotional details for workflow author Billy, including email and URLs.  
    - No inputs or outputs.

  - **Sticky Note**  
    - Provides setup instructions:  
      - Google Sheets template link.  
      - Reminder to update Google Sheets document ID and Gmail recipient.  
      - Required OAuth2 credentials for Google Sheets and Gmail.  
    - No inputs or outputs.

  - **Sticky Note1**  
    - Describes the purpose and capabilities of the workflow: error capture, logging, email alerts, and execution link inclusion.  
    - No inputs or outputs.

  - **Sticky Note2**  
    - Summarizes the workflow process step-by-step in plain language.  
    - No inputs or outputs.

---

### 3. Summary Table

| Node Name                   | Node Type                  | Functional Role                 | Input Node(s)      | Output Node(s)             | Sticky Note                                                                                     |
|-----------------------------|----------------------------|--------------------------------|--------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Error Trigger               | errorTrigger               | Detects any workflow error     | (none)             | Gmail - Send Notification, Google Sheets - Create Error Log |                                                                                                |
| Google Sheets - Create Error Log | googleSheets              | Logs error details to spreadsheet | Error Trigger      | (none)                     |                                                                                                |
| Gmail - Send Notification   | gmail                      | Sends email alert on error     | Error Trigger      | (none)                     |                                                                                                |
| Sticky Note4                | stickyNote                 | Author contact info             | (none)             | (none)                     |                                                                                                |
| Sticky Note                 | stickyNote                 | Setup instructions             | (none)             | (none)                     | Setup instructions with link: https://docs.google.com/spreadsheets/d/11-vLBAKolEvaL0qQDjckHmvC1S6_hxHbgSP8CLyngSs/edit?gid=0#gid=0 |
| Sticky Note1                | stickyNote                 | Workflow purpose & summary     | (none)             | (none)                     |                                                                                                |
| Sticky Note2                | stickyNote                 | Workflow process overview      | (none)             | (none)                     |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Error Trigger node**  
   - Type: `Error Trigger`  
   - Purpose: To capture any error occurring in any workflow execution.  
   - Configuration: Use default settings to listen globally. No additional parameters needed.

2. **Create a Google Sheets node for error logging**  
   - Type: `Google Sheets`  
   - Operation: `Append`  
   - Document ID: Use your own Google Sheets document ID where you want to store logs.  
   - Sheet Name: Set to `"Sheet1"` or your target sheet name.  
   - Columns to map:  
     - Workflow ID → `{{$json.workflow.id}}`  
     - Workflow Name → `{{$json.workflow.name}}`  
     - Node Name → `{{$json.execution.lastNodeExecuted}}`  
     - Node ID → `{{$json.execution.id}}`  
     - Timestamp → `{{new Date($json.execution.error.timestamp).toISOString()}}`  
     - Execution Url → `{{$json.execution.url}}`  
     - Log → `{{$json.execution.error.stack}}`  
   - Credentials: Connect with a valid Google Sheets OAuth2 credential authorized for the target spreadsheet.

3. **Create a Gmail node to send error notification**  
   - Type: `Gmail`  
   - Operation: Send email (plain text)  
   - Recipient: Set to your desired notification email address (default is `n8n_log_template@yopmail.com`)  
   - Subject template: `🚨 n8n Error in Workflow "{{ $json.workflow.name }}" (Node: {{ $json.execution.error.context.nodeCause }})`  
   - Message template:  
     ```
     🚨 n8n Workflow Error Alert

     • Workflow Name: {{ $json.workflow.name }}
     • Node: {{ $json.execution.error.context.nodeCause }}
     • Error Type: {{ $json.execution.error.name }}
     • Message: {{ $json.execution.error.description }}
     • Execution ID: {{ $json.execution.id }}
     • View Execution: {{ $json.execution.url }}
     ```  
   - Credentials: Use a Gmail OAuth2 credential configured with your Gmail account.

4. **Connect nodes**  
   - Connect `Error Trigger` node output to both `Google Sheets - Create Error Log` and `Gmail - Send Notification` nodes as parallel downstream nodes.

5. **Add sticky notes for documentation (optional but recommended)**  
   - Add notes for:  
     - Setup instructions including Google Sheets template and credential requirements.  
     - Workflow purpose summary.  
     - Error process overview steps.  
     - Author contact information for support.

6. **Credentials setup**  
   - Google Sheets OAuth2 Credential: Must have edit rights to target spreadsheet.  
   - Gmail OAuth2 Credential: Must have sending rights for the notification email.

7. **Testing**  
   - Trigger an error in any workflow to verify the error is caught, logged in Google Sheets, and triggers an email notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Workflow requires copying the Google Sheets structure from the template for correct column mapping.                                                        | https://docs.google.com/spreadsheets/d/11-vLBAKolEvaL0qQDjckHmvC1S6_hxHbgSP8CLyngSs/edit?gid=0#gid=0                     |
| Update Google Sheets document ID and Gmail recipient address to personalize alerts.                                                                         | Configuration step detailed in sticky notes                                                                             |
| Gmail and Google Sheets OAuth2 credentials must be configured with appropriate permissions.                                                                | n8n credential management                                                                                                |
| Error notification email uses a clear format with direct links to execution for fast troubleshooting.                                                      | Email subject and message templates in Gmail node                                                                        |
| Author contact for professional n8n workflow and AI automation assistance: Billy Chartanto, billychartanto@gmail.com, https://n8n.io/creators/billy         | https://n8n.io/creators/billy                                                                                            |
| This workflow enables centralized error monitoring, useful for teams managing complex automation environments to maintain high reliability and quick fixes. | General workflow benefit                                                                                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.