Multi-Channel Workflow Error Alerts with Telegram, Gmail & Messaging Apps

https://n8nworkflows.xyz/workflows/multi-channel-workflow-error-alerts-with-telegram--gmail---messaging-apps-5629


# Multi-Channel Workflow Error Alerts with Telegram, Gmail & Messaging Apps

---

### 1. Workflow Overview

This workflow is designed to monitor and send multi-channel error notifications whenever any other n8n workflow fails. It captures error details automatically and broadcasts them through several messaging platforms, including Telegram, Gmail, Discord, Slack, and WhatsApp (the last three currently disabled by default).

Logical blocks included:

- **1.1 Error Detection:** Listens for any workflow error events.
- **1.2 Error Message Formatting:** Processes raw error data into a formatted notification message.
- **1.3 Multi-Channel Notifications:** Sends the formatted error message to various configured communication channels (Telegram, Gmail, Discord, Slack, WhatsApp).
- **1.4 Documentation Note:** Provides in-workflow documentation and usage instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Detection

- **Overview:**  
  This block captures any error occurring in other workflows running on the same n8n instance. It acts as the trigger for the entire notification process.

- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**  
  - **Error Trigger**  
    - Type: `Error Trigger` node  
    - Technical Role: Listens globally for any workflow execution errors within n8n.  
    - Configuration: No parameters required; listens to all errors by default.  
    - Input Connections: None (trigger node).  
    - Output Connections: Connected to the `Code` node to pass error data downstream.  
    - Version Requirements: Compatible with n8n v1.99.1 and above.  
    - Potential Failures: None expected on trigger itself; however, if n8n is misconfigured or errors are suppressed globally, this node may not receive events.

#### 1.2 Error Message Formatting

- **Overview:**  
  This block converts raw error data into a human-readable, richly formatted message suitable for multiple messaging apps. It extracts key error properties and formats them using HTML markup for enhanced display.

- **Nodes Involved:**  
  - Code

- **Node Details:**  
  - **Code**  
    - Type: `Code` (JavaScript) node  
    - Technical Role: Processes error JSON to create a structured alert message.  
    - Configuration: Custom JS script that extracts workflow name, node name, error message, description, timestamp (localized Indonesian time), execution URL, and error severity level. It returns a JSON object containing the formatted message, parsing mode (HTML), and a default Telegram chat ID.  
    - Key Expressions/Variables:  
      - `errorData.execution` and `errorData.workflow` for metadata  
      - Template literals with HTML tags for message formatting  
      - `new Date(error.timestamp).toLocaleString('id-ID')` for time display  
    - Input Connections: Receives error data from `Error Trigger`.  
    - Output Connections: Outputs to all notification nodes (`Notify Telegram`, `Notify Discord`, `Notify Slack`, `Notify Whatsapp`, `Notify Gmail`).  
    - Version Requirements: n8n v1.99.1+ recommended for error structure compatibility.  
    - Edge Cases:  
      - Missing or malformed error properties may cause undefined values or message formatting issues.  
      - If the error timestamp is missing or invalid, date conversion may fail or show incorrect values.  
    - Sub-workflow: None.

#### 1.3 Multi-Channel Notifications

- **Overview:**  
  This block distributes the formatted error message to several messaging services, enabling notification redundancy and user preference flexibility.

- **Nodes Involved:**  
  - Notify Telegram  
  - Notify Gmail  
  - Notify Discord (disabled)  
  - Notify Slack (disabled)  
  - Notify Whatsapp (disabled)

- **Node Details:**  

  - **Notify Telegram**  
    - Type: `Telegram` node  
    - Technical Role: Sends the error message to a Telegram chat via a bot.  
    - Configuration:  
      - `text` parameter uses expression `{{$json.message}}` to send the formatted message.  
      - `chatId` is set as a placeholder `<your_chat_id>` and requires user replacement.  
      - Additional field `parse_mode` set to `HTML` for message formatting.  
    - Credentials: Uses Telegram Bot API credentials (`Khaisa Monitoring Bot`).  
    - Input Connections: From `Code` node.  
    - Output Connections: None.  
    - Edge Cases:  
      - Invalid or missing chat ID will cause message delivery failure.  
      - Bot permissions must allow sending messages to the specified chat.  
      - Rate limits or downtime on Telegram API could delay or block notifications.

  - **Notify Gmail**  
    - Type: `Gmail` node  
    - Technical Role: Sends an email notification with the error details.  
    - Configuration: Basic setup with default options; message body and subject can be configured as needed by user (currently minimal).  
    - Credentials: Gmail OAuth2 credentials must be configured (`Gmail account`).  
    - Input Connections: From `Code` node.  
    - Output Connections: None.  
    - Edge Cases:  
      - OAuth2 token expiry or revocation may cause authentication failures.  
      - Email quota limits may block sending.  
      - Email formatting not explicitly specified; may require enhancements.

  - **Notify Discord** (disabled)  
    - Type: `Discord` node  
    - Role: Intended to send notifications via Discord webhook.  
    - Configuration: Sends the message content as `{{$json.message}}` using webhook authentication.  
    - Credentials: Requires configured Discord Webhook credentials.  
    - Input Connections: From `Code` node.  
    - Disabled by default; must be enabled and configured manually.  
    - Potential Issues: Invalid webhook URL or permissions errors.

  - **Notify Slack** (disabled)  
    - Type: `Slack` node  
    - Role: Intended to send notifications to Slack channels.  
    - Configuration: Basic setup with default options; message content not explicitly defined.  
    - Credentials: Requires Slack API credentials.  
    - Input Connections: From `Code` node.  
    - Disabled by default; requires manual activation and configuration.  
    - Potential Issues: Invalid token, insufficient permissions, or API rate limiting.

  - **Notify Whatsapp** (disabled)  
    - Type: `WhatsApp` node  
    - Role: Intended for sending WhatsApp messages.  
    - Configuration: Operation set to "send" but no message or recipient configured.  
    - Disabled by default; requires setup.  
    - Input Connections: From `Code` node.  
    - Potential Issues: Requires proper WhatsApp API credentials and recipient numbers.

#### 1.4 Documentation Note

- **Overview:**  
  Provides embedded user instructions and informational content inside the workflow for easy reference.

- **Nodes Involved:**  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note1**  
    - Type: `Sticky Note` node  
    - Technical Role: Houses documentation text describing the workflow purpose, usage instructions, and contact links.  
    - Configuration:  
      - Text content includes workflow description, usage hints, and links:  
        - [contact me](khmuhtadin.com)  
        - [a coffee](coff.ee/khmuhtadin)  
      - Styling: color set to 5 (blue), size set for readability.  
    - Input and Output: None (informational only).  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name        | Node Type       | Functional Role                      | Input Node(s)   | Output Node(s)                           | Sticky Note                                                                                                                        |
|------------------|-----------------|------------------------------------|-----------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Error Trigger    | Error Trigger   | Detects any workflow errors        | —               | Code                                   | 🚨 Error Notification Workflow: Automatically sends notifications on failures. See sticky note for contact and support links.    |
| Code             | Code            | Formats error message for alerts   | Error Trigger   | Notify Telegram, Notify Discord, Notify Slack, Notify Whatsapp, Notify Gmail | See sticky note on Error Trigger for workflow purpose and links.                                                                 |
| Notify Telegram  | Telegram        | Sends error alert via Telegram bot | Code            | —                                      | Replace `<your_chat_id>` with actual chat ID. Requires Telegram Bot credentials.                                                  |
| Notify Gmail     | Gmail           | Sends error alert via email        | Code            | —                                      | Requires Gmail OAuth2 credentials.                                                                                              |
| Notify Discord   | Discord         | Sends error alert via Discord      | Code            | —                                      | Disabled by default; requires webhook credentials and manual enablement.                                                         |
| Notify Slack     | Slack           | Sends error alert via Slack        | Code            | —                                      | Disabled by default; requires Slack credentials and manual enablement.                                                           |
| Notify Whatsapp  | WhatsApp        | Sends error alert via WhatsApp     | Code            | —                                      | Disabled by default; requires WhatsApp API setup and manual enablement.                                                          |
| Sticky Note1     | Sticky Note     | Workflow description and contacts  | —               | —                                      | ## 🚨 Error Notification Workflow\nThis workflow sends error alerts...\nContact: khmuhtadin.com\nSupport: coff.ee/khmuhtadin     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an "Error Trigger" node:**  
   - Type: `Error Trigger`  
   - Purpose: To listen for all workflow errors automatically.  
   - No configuration needed.  
   - Position it as the starting node.

2. **Add a "Code" node:**  
   - Type: `Code` (JavaScript)  
   - Connect the output of `Error Trigger` to the input of this `Code` node.  
   - Paste the following JS code (adapted for n8n v1.99.1+):  
     ```js
     const errorData = $input.first().json;
     const execution = errorData.execution;
     const workflow = errorData.workflow;
     const error = execution.error;

     const message = `🚨 <b>WORKFLOW ERROR</b>\n\n` +
       `📋 Workflow: <b>${workflow.name}</b>\n` +
       `⚙️ Node: <code>${error.node.name}</code>\n` +
       `❌ Error: <code>${error.message}</code>\n` +
       `📝 Description: <i>${error.description}</i>\n` +
       `🕐 Waktu: ${new Date(error.timestamp).toLocaleString('id-ID')}\n` +
       `🔍 Execution: <a href="${execution.url}">${execution.id}</a>\n` +
       `⚠️ Level: ${error.level.toUpperCase()}`;

     return {
       message: message,
       parse_mode: 'HTML',
       chat_id: '621412350', // default placeholder; override as needed
     };
     ```
   - This node formats the error for messaging platforms.

3. **Set up "Notify Telegram" node:**  
   - Type: `Telegram`  
   - Connect input from the `Code` node.  
   - Configure:  
     - `Text` field: Expression `{{$json.message}}`  
     - `Chat ID`: Replace `<your_chat_id>` with your actual Telegram chat ID.  
     - Additional fields: Set `parse_mode` to `HTML` to preserve formatting.  
   - Add Telegram API credentials (bot token) under credentials.

4. **Set up "Notify Gmail" node:**  
   - Type: `Gmail`  
   - Connect input from the `Code` node.  
   - Configure email message parameters as needed (subject, body), or leave default for initial testing.  
   - Add Gmail OAuth2 credentials.

5. **(Optional) Set up "Notify Discord" node:**  
   - Type: `Discord`  
   - Connect input from the `Code` node.  
   - Configure `Content` field with expression `{{$json.message}}`.  
   - Set authentication to `Webhook`.  
   - Add Discord Webhook URL credentials.  
   - Enable node manually if required.

6. **(Optional) Set up "Notify Slack" node:**  
   - Type: `Slack`  
   - Connect input from the `Code` node.  
   - Configure message content as needed.  
   - Add Slack API credentials.  
   - Enable node manually if required.

7. **(Optional) Set up "Notify Whatsapp" node:**  
   - Type: `WhatsApp`  
   - Connect input from the `Code` node.  
   - Configure recipient and message content (currently empty in original).  
   - Add WhatsApp API credentials.  
   - Enable node manually if required.

8. **Add a "Sticky Note" node:**  
   - Type: `Sticky Note`  
   - Content:  
     ```
     ## 🚨 Error Notification Workflow

     This workflow automatically sends notifications when any other workflow fails.

     Simply download copy and paste into your workflow then add your credential with preferred channel.

     This workflow is free.  
     Want more? [contact me](khmuhtadin.com)

     Have something to help me chill, [a coffee](coff.ee/khmuhtadin) would be great!
     ```
   - Position it for visibility.

9. **Set workflow activation settings:**  
   - Ensure the workflow is active and the `Error Trigger` node is enabled to capture errors globally.

---

### 5. General Notes & Resources

| Note Content                                                                                                                    | Context or Link                |
|---------------------------------------------------------------------------------------------------------------------------------|-------------------------------|
| This workflow automatically sends notifications when any other workflow fails.                                                  | Workflow purpose summary       |
| Replace `<your_chat_id>` in the Telegram node with the actual chat ID to receive messages.                                      | Telegram configuration         |
| The workflow is free and open for modification; author contact: khmuhtadin.com                                                  | Contact link: khmuhtadin.com   |
| Support the author with a coffee donation: coff.ee/khmuhtadin                                                                  | Support link                   |
| Recommended n8n version: v1.99.1 or higher for best compatibility with Error Trigger node structure                            | Version recommendation         |
| Disabled nodes (Slack, Discord, WhatsApp) must be enabled and configured manually depending on user preferences and credentials | Optional channels configuration|

---

**Disclaimer:**  
The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.