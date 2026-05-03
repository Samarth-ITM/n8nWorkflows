Multi-channel Error Monitoring with Telegram, Discord, Slack, WhatsApp and Gmail

https://n8nworkflows.xyz/workflows/multi-channel-error-monitoring-with-telegram--discord--slack--whatsapp-and-gmail-5630


# Multi-channel Error Monitoring with Telegram, Discord, Slack, WhatsApp and Gmail

### 1. Workflow Overview

This workflow is designed for **multi-channel error monitoring** in n8n. Its main purpose is to automatically detect any workflow failures within the n8n instance and send detailed error notifications across several communication platforms: Telegram, Discord, Slack, WhatsApp, and Gmail. This allows DevOps teams or users to be instantly alerted about issues in their workflows, improving incident response times.

The workflow is logically divided into the following blocks:

- **1.1 Error Detection:** Using the built-in Error Trigger node to capture any workflow errors occurring in the n8n environment.
- **1.2 Error Message Preparation:** A Code node formats the error information into an informative, human-readable message with HTML formatting.
- **1.3 Multi-Channel Notification Dispatch:** Several messaging nodes send the prepared error message simultaneously to configured channels (Telegram, Discord, Slack, WhatsApp, Gmail). Most are disabled by default but can be enabled and configured per user preference.
- **1.4 Informational Sticky Note:** Provides workflow context and author information for users importing or editing the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection

- **Overview:**  
  Captures any error events from other workflows running in the same n8n instance to trigger this monitoring workflow.

- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**  
  - **Error Trigger**  
    - *Type & Role:* System trigger node that activates when any workflow execution fails within n8n.  
    - *Configuration:* No parameters needed; listens globally for errors.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* One, passes error execution data downstream.  
    - *Version requirements:* Requires n8n supporting Error Trigger (available in n8n v1.99.1+).  
    - *Edge cases:* If n8n error handling system is disabled or misconfigured, errors might not be captured.  
    - *Sub-workflow:* None.

#### 2.2 Error Message Preparation

- **Overview:**  
  Receives raw error data from the Error Trigger node and constructs a formatted HTML message summarizing the error details.

- **Nodes Involved:**  
  - Prepare Messages (Code node)

- **Node Details:**  
  - **Prepare Messages**  
    - *Type & Role:* Code node that processes JSON error data and returns a formatted message object.  
    - *Configuration:*  
      - JavaScript code extracts relevant error and workflow info such as workflow name, node name, error message, description, timestamp, execution ID, and error level.  
      - Formats a multi-line string with HTML tags (`<b>`, `<code>`, `<i>`, `<a>`) for rich notification display.  
      - Returns an object with `message`, `parse_mode` set to "HTML", and a `chat_id` (default placeholder).  
    - *Key expressions:* Uses `$input.first().json` to access the first input item (error data).  
    - *Inputs:* Error Trigger node output.  
    - *Outputs:* JSON object with formatted message and metadata.  
    - *Version requirements:* Uses v2 Code node syntax, requires n8n supporting this.  
    - *Edge cases:* If error data is missing fields or malformed, message formatting may fail or produce incomplete info. Timestamp conversion depends on valid ISO date.  
    - *Sub-workflow:* None.

#### 2.3 Multi-Channel Notification Dispatch

- **Overview:**  
  Sends the prepared error message to multiple communication channels simultaneously. Each channel node is individually configurable and mostly disabled by default, letting users enable preferred notification methods.

- **Nodes Involved:**  
  - Notify Telegram  
  - Notify Discord (disabled)  
  - Notify Slack (disabled)  
  - Notify WhatsApp (disabled)  
  - Notify Gmail (disabled)

- **Node Details:**  

  - **Notify Telegram**  
    - *Type & Role:* Telegram node sends the error message as a chat message.  
    - *Configuration:*  
      - Text: Uses expression `{{$json.message}}` to get the formatted message from Prepare Messages.  
      - Chat ID: Placeholder `<your_chat_id>` to be replaced by user.  
      - Additional fields: `parse_mode` set to "HTML" to render message formatting.  
    - *Credentials:* Linked to a Telegram Bot API credential ("Khaisa Monitoring Bot").  
    - *Inputs:* Prepare Messages output.  
    - *Outputs:* None downstream.  
    - *Edge cases:* Invalid chat ID or revoked bot token will cause message send failure.  
    - *Sub-workflow:* None.

  - **Notify Discord** (disabled)  
    - *Type & Role:* Discord node posts message via webhook.  
    - *Configuration:*  
      - Content: Expression `{{$json.message}}`  
      - Authentication: Webhook-based.  
    - *Credentials:* Discord webhook credential.  
    - *Inputs:* Prepare Messages output.  
    - *Outputs:* None downstream.  
    - *Edge cases:* Incorrect webhook URL or permission issues cause failures.

  - **Notify Slack** (disabled)  
    - *Type & Role:* Slack node sends message to Slack channel or user.  
    - *Configuration:* Uses default options with message content from Prepare Messages.  
    - *Credentials:* Slack OAuth2 credential.  
    - *Inputs:* Prepare Messages output.  
    - *Outputs:* None downstream.  
    - *Edge cases:* Invalid token or channel ID issues.

  - **Notify WhatsApp** (disabled)  
    - *Type & Role:* WhatsApp node sends messages via WhatsApp Business API or supported integration.  
    - *Configuration:* Operation set to "send". No additional fields by default.  
    - *Credentials:* WhatsApp API credential.  
    - *Inputs:* Prepare Messages output.  
    - *Outputs:* None downstream.  
    - *Edge cases:* API quota limits, invalid phone number, or auth errors.

  - **Notify Gmail** (disabled)  
    - *Type & Role:* Gmail node sends an email notification with error details.  
    - *Configuration:* No parameters set by default; requires setup for email content and recipient.  
    - *Credentials:* Gmail OAuth2 credential.  
    - *Inputs:* Prepare Messages output.  
    - *Outputs:* None downstream.  
    - *Edge cases:* OAuth token expiration, Gmail API limits.

#### 2.4 Informational Sticky Note

- **Overview:**  
  Provides user-facing documentation and credit information within the workflow editor.

- **Nodes Involved:**  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note1**  
    - *Type & Role:* Visual note node in the editor for documentation.  
    - *Configuration:* Contains markdown text introducing the workflow, usage instructions, author contact, and a donation link.  
    - *Inputs/Outputs:* None, purely informational.  
    - *Edge cases:* None.

---

### 3. Summary Table

| Node Name       | Node Type            | Functional Role                | Input Node(s)   | Output Node(s)                                       | Sticky Note                                                                                                 |
|-----------------|----------------------|-------------------------------|-----------------|-----------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Error Trigger   | Error Trigger        | Detects workflow failures      | None            | Prepare Messages                                    |                                                                                                             |
| Prepare Messages| Code                 | Formats error message           | Error Trigger   | Notify Telegram, Notify Discord, Notify Slack, Notify Whatsapp, Notify Gmail |                                                                                                             |
| Notify Telegram | Telegram             | Sends error notification to Telegram | Prepare Messages | None                                                |                                                                                                             |
| Notify Discord  | Discord (disabled)   | Sends error notification to Discord  | Prepare Messages | None                                                |                                                                                                             |
| Notify Slack    | Slack (disabled)     | Sends error notification to Slack    | Prepare Messages | None                                                |                                                                                                             |
| Notify Whatsapp | WhatsApp (disabled)  | Sends error notification to WhatsApp | Prepare Messages | None                                                |                                                                                                             |
| Notify Gmail    | Gmail (disabled)     | Sends error notification via email   | Prepare Messages | None                                                |                                                                                                             |
| Sticky Note1    | Sticky Note          | In-workflow documentation       | None            | None                                                | ## 🚨 Error Monitoring Workflow<br>This workflow automatically sends notifications when any other workflow fails.<br>Simply download copy and paste into your workflow then add your credential with prefered channel<br>This workflow is free.<br>want more? [contact me](khmuhtadin.com)<br>have something to helps me chill, [a coffee](coff.ee/khmuhtadin) would be great! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Error Trigger node**  
   - Type: Error Trigger  
   - Configuration: Leave default; no parameters needed.  
   - Purpose: To start the workflow when any n8n workflow fails.

2. **Add a Code node named "Prepare Messages"**  
   - Connect the Error Trigger output to this node input.  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
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
       chat_id: '621412350'  // Replace with your Telegram chat ID or remove if not needed.
     };
     ```
   - Purpose: To prepare a rich-text error message for notifications.

3. **Add a Telegram node named "Notify Telegram"**  
   - Connect "Prepare Messages" output to this node input.  
   - Type: Telegram  
   - Parameters:  
     - Text: Use expression `{{$json.message}}` to send the formatted message.  
     - Chat ID: Replace `<your_chat_id>` with your actual Telegram chat ID.  
     - Additional Fields: Set `parse_mode` to "HTML" to preserve message formatting.  
   - Credentials: Create or select a Telegram Bot API credential linked to your bot token.  
   - Purpose: To send error notifications via Telegram.

4. **(Optional) Add Discord node "Notify Discord"**  
   - Connect from "Prepare Messages" similarly.  
   - Type: Discord  
   - Parameters:  
     - Content: Use expression `{{$json.message}}`.  
     - Authentication: Use Webhook.  
   - Credentials: Configure Discord webhook credential.  
   - Purpose: To send error notifications to a Discord channel.

5. **(Optional) Add Slack node "Notify Slack"**  
   - Connect from "Prepare Messages".  
   - Type: Slack  
   - Parameters: Configure message content to use `{{$json.message}}`.  
   - Credentials: Slack OAuth2 credential with chat post permissions.  
   - Purpose: To notify Slack channels/users.

6. **(Optional) Add WhatsApp node "Notify Whatsapp"**  
   - Connect from "Prepare Messages".  
   - Type: WhatsApp  
   - Parameters: Operation set to "send", configure phone numbers as needed.  
   - Credentials: WhatsApp Business API or supported credential.  
   - Purpose: To send WhatsApp alerts.

7. **(Optional) Add Gmail node "Notify Gmail"**  
   - Connect from "Prepare Messages".  
   - Type: Gmail  
   - Parameters: Configure email recipient, subject, and body using message data.  
   - Credentials: Gmail OAuth2 credential.  
   - Purpose: To send email alerts.

8. **Add a Sticky Note**  
   - Create a sticky note in any position convenient.  
   - Paste the following markdown content:  
     ```
     ## 🚨 Error Monitoring Workflow

     This workflow automatically sends notifications when any other workflow fails.

     Simply download copy and paste into your workflow then add your credential with preferred channel.

     This workflow is free.  
     want more? [contact me](khmuhtadin.com)

     have something to help me chill, [a coffee](coff.ee/khmuhtadin) would be great!
     ```

9. **Workflow Settings**  
   - Set execution order to "v1".  
   - Activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                           |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------|
| This workflow is created by Khaisa. For tailored support and more workflows, visit shop.khaisa.com.            | Author website                           |
| The workflow uses the Error Trigger node introduced in n8n v1.99.1 for optimal error capturing.                 | n8n Documentation                        |
| Telegram messages use HTML formatting for rich text display; ensure bots have appropriate permissions.        | Telegram Bot API docs                    |
| The workflow includes disabled nodes for optional channels; enable and configure them as needed.               | n8n Node documentation                   |
| Support and contact available via [khmuhtadin.com](https://khmuhtadin.com).                                     | Author contact link                      |
| Donations appreciated to support ongoing development: [coff.ee/khmuhtadin](https://coff.ee/khmuhtadin)         | Donation link                           |

---

This documentation enables both human operators and automated systems to fully understand, reproduce, and adapt the multi-channel error monitoring workflow in n8n.