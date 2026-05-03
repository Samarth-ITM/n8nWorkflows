n8n Workflow Error Alerts with Google Sheets, Telegram, and Gmail

https://n8nworkflows.xyz/workflows/n8n-workflow-error-alerts-with-google-sheets--telegram--and-gmail-4407


# n8n Workflow Error Alerts with Google Sheets, Telegram, and Gmail

### 1. Workflow Overview

This workflow is designed for **error handling and alerting** within n8n. Its main purpose is to **capture workflow execution errors**, **log them into a Google Sheets spreadsheet** for record-keeping, and **notify responsible parties via Telegram and Gmail** to ensure prompt awareness and resolution. The workflow is targeted toward teams or individuals who want automated monitoring and alerting of n8n workflow failures with historical tracking.

Logical blocks:

- **1.1 Error Detection & Data Extraction:** Captures errors occurring in any workflow execution.
- **1.2 Error Logging:** Records detailed error information into a Google Sheets document.
- **1.3 Notification Preparation:** Sets required variables such as Telegram chat ID and email recipient.
- **1.4 Alert Dispatch:** Sends error alerts as messages to a Telegram channel and as emails via Gmail.
- **1.5 User Instructions:** Provides notes for configuring chat IDs and emails.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Detection & Data Extraction

- **Overview:** This block triggers whenever an error occurs in any workflow and extracts relevant error details for further processing.
- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**

  - **Error Trigger**  
    - Type: `Error Trigger` (system node)  
    - Role: Listens globally for any workflow execution errors in the n8n instance.  
    - Configuration: Default - no special parameters.  
    - Inputs: None (it's a trigger).  
    - Outputs: Emits error data including workflow name, error message, node where error occurred, and execution URL.  
    - Edge Cases: Will not trigger if error handling is suppressed or if the node is inactive.  
    - Notes: This node is fundamental as it initiates the entire alerting process on error occurrence.

#### 1.2 Error Logging

- **Overview:** Appends a new row to a designated Google Sheets spreadsheet with detailed error information, enabling historical tracking and status management.
- **Nodes Involved:**  
  - Log error

- **Node Details:**

  - **Log error**  
    - Type: `Google Sheets` (append operation)  
    - Role: Logs error metadata into a Google Sheet row.  
    - Configuration:  
      - Document: Specific Google Sheet by ID  
      - Sheet: Default sheet (gid=0)  
      - Operation: Append row  
      - Columns mapped: Timestamp, Workflow name, Execution URL, Node name where error occurred, Error message, Status (preset to "NEW"), Notes (empty)  
      - Timestamp format: Day and hour with AM/PM (e.g., "5 03:24 pm")  
    - Key expressions:  
      - `{{$json.execution.url}}` — execution URL  
      - `{{$json.execution.error.node.name}}` — node name where error occurred  
      - `{{$json.workflow.name}}` — workflow name  
      - `{{$now.format('D hh:mm a')}}` — current timestamp  
      - `{{$json.execution.error.message}}` — error message  
    - Inputs: Error data from "Error Trigger"  
    - Outputs: Passes data downstream unchanged.  
    - Credentials: Google Sheets OAuth2 account configured.  
    - Edge Cases: Failure to append if Google Sheets API quota exceeded, invalid credentials, or sheet permissions insufficient.

#### 1.3 Notification Preparation

- **Overview:** Prepares and sets essential variables such as Telegram chat ID and email recipient to be used in notification messages.
- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**

  - **Edit Fields**  
    - Type: `Set` node  
    - Role: Defines static values for notification recipients (Telegram chat ID and email address).  
    - Configuration:  
      - Sets field `telegramChatID` (default string "chatID")  
      - Sets field `toEmail` (default string "toEmail")  
    - Inputs: Receives error data from previous nodes  
    - Outputs: Passes data including newly set fields downstream  
    - Edge Cases: If these fields are not updated with actual values, notifications will fail or go to incorrect recipients.

#### 1.4 Alert Dispatch

- **Overview:** Sends error notifications to the specified Telegram channel and email address with detailed error information.
- **Nodes Involved:**  
  - Notify in channel  
  - Send email

- **Node Details:**

  - **Notify in channel**  
    - Type: `Telegram` node  
    - Role: Sends a formatted error alert message to a Telegram chat/channel.  
    - Configuration:  
      - Text message includes workflow name, execution URL, node name, and error message using expressions referencing the "Error Trigger" node’s JSON data.  
      - Chat ID is dynamically assigned from the `telegramChatID` field set earlier.  
      - No attribution appended to messages.  
    - Inputs: Receives data from "Edit Fields"  
    - Outputs: Passes data to "Send email" node  
    - Credentials: Configured with a Telegram API token.  
    - Edge Cases: Invalid chat ID, Telegram API errors, network issues.

  - **Send email**  
    - Type: `Gmail` node  
    - Role: Sends an email alert with the error details.  
    - Configuration:  
      - Recipient email dynamically assigned from `toEmail` field.  
      - Email subject includes workflow name.  
      - Message body uses the text result from the Telegram node (to keep messages consistent).  
      - Sender name set to "n8n Error Tracker".  
      - No attribution appended.  
    - Inputs: Receives data from "Notify in channel"  
    - Credentials: Gmail OAuth2 configured.  
    - Edge Cases: Invalid email address, Gmail quota exceeded, authentication errors.

#### 1.5 User Instructions

- **Overview:** Provides a sticky note with instructions for users to update Telegram chat ID and email recipient fields.
- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: `Sticky Note` (UI annotation)  
    - Role: Documentation inside the workflow canvas.  
    - Configuration: Contains instructions:  
      - "Set your Telegram chat id to get notified in a channel"  
      - "Insert the recipient's email"  
    - Inputs/Outputs: None  
    - Edge Cases: None

---

### 3. Summary Table

| Node Name       | Node Type        | Functional Role                            | Input Node(s)      | Output Node(s)         | Sticky Note                                                                                   |
|-----------------|------------------|------------------------------------------|--------------------|------------------------|-----------------------------------------------------------------------------------------------|
| Error Trigger   | Error Trigger    | Captures workflow execution errors       | None               | Log error, Edit Fields  |                                                                                               |
| Log error       | Google Sheets    | Logs error details into Google Sheets     | Error Trigger      | None                   |                                                                                               |
| Edit Fields     | Set              | Sets variables for Telegram chat ID, email | Error Trigger      | Notify in channel       |                                                                                               |
| Notify in channel| Telegram         | Sends error alert message to Telegram     | Edit Fields        | Send email              |                                                                                               |
| Send email      | Gmail            | Sends error alert email                    | Notify in channel  | None                   |                                                                                               |
| Sticky Note     | Sticky Note      | Provides instructions for user setup      | None               | None                   | Update fields: Set your Telegram chat id to get notified in a channel; Insert the recipient's email |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Error Trigger" node:**  
   - Type: Error Trigger  
   - Purpose: Listen for any workflow errors to initiate alerting.  
   - Leave default parameters.

2. **Create "Log error" node:**  
   - Type: Google Sheets  
   - Operation: Append row  
   - Document ID: Use your Google Sheet ID where errors will be logged.  
   - Sheet Name: Default or as per your sheet tab (e.g., "Sheet1").  
   - Map columns with these keys and expressions:  
     - URL: `{{$json.execution.url}}`  
     - Node: `{{$json.execution.error.node.name}}`  
     - STATUS: `"NEW"` (static string)  
     - Workflow: `{{$json.workflow.name}}`  
     - Timestamp: `{{$now.format('D hh:mm a')}}`  
     - Error Message: `{{$json.execution.error.message}}`  
   - Configure Google Sheets OAuth2 credentials.

3. **Create "Edit Fields" node:**  
   - Type: Set  
   - Assign two string fields:  
     - `telegramChatID` with your Telegram chat ID (replace `"chatID"`)  
     - `toEmail` with recipient email address (replace `"toEmail"`)

4. **Connect nodes:**  
   - Connect "Error Trigger" to "Log error" and "Edit Fields" (parallel branches).

5. **Create "Notify in channel" node:**  
   - Type: Telegram  
   - Chat ID: `={{ $json.telegramChatID }}`  
   - Message text:  
     ```
     ⚠️🐛 New bug in n8n

     Workflow: {{ $('Error Trigger').item.json.workflow.name }}
     Execution URL: {{ $('Error Trigger').item.json.execution.url }}
     Node name: {{ $('Error Trigger').item.json.execution.error.node.name }}
     Error message: {{ $('Error Trigger').item.json.execution.error.message }}
     ```  
   - Disable attribution.  
   - Use Telegram API credentials.

6. **Connect "Edit Fields" to "Notify in channel".**

7. **Create "Send email" node:**  
   - Type: Gmail  
   - Send To: `={{ $json.toEmail }}`  
   - Subject: `🐛New n8n bug in "{{ $('Error Trigger').item.json.workflow.name }}"`  
   - Message: Use the output text from "Notify in channel" node (`{{$json.result.text}}`).  
   - Sender Name: "n8n Error Tracker"  
   - Disable attribution.  
   - Use Gmail OAuth2 credentials.

8. **Connect "Notify in channel" to "Send email".**

9. **Create "Sticky Note":**  
   - Add instructions:  
     - "Set your Telegram chat id to get notified in a channel"  
     - "Insert the recipient's email"  
   - Position on canvas for user visibility.

10. **Set workflow timezone to "Europe/Madrid" or as appropriate.**

11. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Ensure Google Sheets API, Telegram Bot API, and Gmail API OAuth2 credentials are correctly set up.   | Credential configuration in n8n                  |
| Telegram chat ID must be numeric and correspond to a chat or channel where the bot has permissions. | Telegram Bot API documentation                    |
| Gmail OAuth2 must have "Send email" permissions enabled.                                            | Google Developer Console Gmail API setup          |
| Timestamp format uses day and 12-hour time with AM/PM (e.g. "5 03:24 pm").                          | Standard moment.js formatting                      |
| This workflow requires n8n version supporting Error Trigger node (v0.157+ recommended).             | https://docs.n8n.io/nodes/n8n-nodes-base.errorTrigger/ |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.