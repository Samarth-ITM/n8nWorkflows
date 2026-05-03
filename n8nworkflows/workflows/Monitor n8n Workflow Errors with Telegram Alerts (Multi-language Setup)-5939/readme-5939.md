Monitor n8n Workflow Errors with Telegram Alerts (Multi-language Setup)

https://n8nworkflows.xyz/workflows/monitor-n8n-workflow-errors-with-telegram-alerts--multi-language-setup--5939


# Monitor n8n Workflow Errors with Telegram Alerts (Multi-language Setup)

### 1. Workflow Overview

This workflow is designed to monitor errors occurring in other n8n workflows and send immediate alerts via Telegram messages. Its primary use case is for active error monitoring in production or critical environments where prompt notification of workflow failures is essential.

The workflow consists of two main logical blocks:

- **1.1 Error Capture:** Listens globally for any error events triggered by other workflows in the n8n instance.
- **1.2 Telegram Alert Dispatch:** Formats and sends a Telegram message containing error details to a specified chat.

Additionally, a comprehensive multilingual setup guide is embedded as a sticky note to facilitate configuration by users in English, Spanish, German, French, and Russian.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Capture

- **Overview:**  
  This block captures any error that occurs in any active workflow within the n8n instance. It serves as the entry point for the alerting process.

- **Nodes Involved:**  
  - *Error Trigger*

- **Node Details:**

  - **Error Trigger**  
    - *Type and Role:* System node that listens for error events across workflows.  
    - *Configuration:* Uses default settings with no filters, meaning it captures all workflow errors globally.  
    - *Key Expressions:* None; the node outputs error metadata automatically.  
    - *Input and Output:* No input; output connects to the Telegram message node.  
    - *Version-specific:* Requires n8n version supporting the `Error Trigger` node (generally v0.175.0+).  
    - *Edge Cases / Failures:*  
      - If the workflow is inactive, no errors will be captured.  
      - Errors occurring within the error-triggered workflow itself will not be captured by this node to avoid infinite loops.

#### 2.2 Telegram Alert Dispatch

- **Overview:**  
  This block sends a Telegram message notification containing error details captured by the previous block. It requires user setup of a Telegram bot and specifying the target chat ID.

- **Nodes Involved:**  
  - *Send a text message with info about error*  
  - *SETUP INSTRUCTIONS* (Sticky Note)

- **Node Details:**

  - **Send a text message with info about error**  
    - *Type and Role:* Telegram node sends text messages via a Telegram Bot API.  
    - *Configuration:*  
      - **Chat ID:** Default demo value `1234567890` that must be replaced by the user’s Telegram chat ID.  
      - **Text:** Template string: `Error en flow  {{ $json.workflow.name }}`. This can be customized. The placeholders available include:  
        - `{{$json.workflow.name}}` — name of the workflow where error occurred  
        - `{{$json.error.message}}` — error message details  
        - `{{$json.execution.id}}` — execution ID of the failed workflow run  
      - **Credentials:** Requires Telegram Bot credentials created via @BotFather.  
    - *Input and Output:* Receives error data from the Error Trigger node via main connection; no output since this is a terminal notification node.  
    - *Version-specific:* Uses Telegram API version supported by n8n Telegram node version 1.2.  
    - *Edge Cases / Failures:*  
      - Invalid or expired Telegram bot token will cause message sending failure.  
      - Incorrect chat ID will prevent message delivery.  
      - Network issues or Telegram API downtime may cause timeouts.  
      - If the message template uses incorrect expressions, sending may fail or produce malformed messages.

  - **SETUP INSTRUCTIONS (Sticky Note)**  
    - *Type and Role:* Informational note providing detailed multilingual setup instructions.  
    - *Content Highlights:*  
      - Step-by-step guide to create a Telegram bot and obtain credentials.  
      - Instructions on how to retrieve the chat ID from Telegram API.  
      - Directions to configure the Telegram message node with credentials and chat ID.  
      - Warnings to keep the workflow active at all times to capture errors.  
      - Available in English, Spanish, German, French, and Russian.  
    - *Input and Output:* None, purely informational.  
    - *Edge Cases:* None, but users may neglect the instructions leading to misconfiguration.

---

### 3. Summary Table

| Node Name                              | Node Type               | Functional Role           | Input Node(s)    | Output Node(s)                      | Sticky Note                                                                                                                              |
|--------------------------------------|-------------------------|--------------------------|------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Error Trigger                        | Error Trigger           | Captures workflow errors | None             | Send a text message with info about error |                                                                                                                                          |
| Send a text message with info about error | Telegram                | Sends Telegram alert     | Error Trigger    | None                              |                                                                                                                                          |
| SETUP INSTRUCTIONS                   | Sticky Note             | Provides setup guide     | None             | None                              | Multi-language setup instructions covering bot creation, chat ID retrieval, node configuration, and activation warnings in 5 languages. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it appropriately (e.g., "ErrorFlow").**

2. **Add an "Error Trigger" node:**  
   - Drag the *Error Trigger* node onto the canvas.  
   - Leave default settings to listen to all workflow errors globally.  
   - This node has no inputs.

3. **Add a "Telegram" node to send messages:**  
   - Drag the *Telegram* node onto the canvas.  
   - Select the "Send a text message" operation.  
   - Configure the *Chat ID* field with a placeholder (e.g., `1234567890`) to be replaced later.  
   - Set the *Text* field to a template string, for example:  
     ```
     Error in flow {{ $json.workflow.name }}  
     Message: {{ $json.error.message }}  
     Execution ID: {{ $json.execution.id }}
     ```  
   - Connect the output of the *Error Trigger* node to the input of this Telegram node.

4. **Create Telegram Bot Credentials:**  
   - Use **@BotFather** on Telegram to create a new bot and obtain the API token.  
   - In n8n, create new credentials for Telegram with this token.  
   - Assign these credentials to the Telegram node.

5. **Obtain your chat ID:**  
   - Send any message to your bot on Telegram.  
   - Visit `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` in your browser.  
   - Locate the `chat → id` number in the response, which is your chat ID.  
   - Replace the placeholder in the Telegram node’s *Chat ID* field with this actual chat ID.

6. **Add a Sticky Note node for setup instructions (optional but recommended):**  
   - Add a *Sticky Note* node to the canvas.  
   - Paste the multilingual setup instructions content into the note.  
   - Position the note visibly near the Telegram node for user reference.

7. **Activate the workflow:**  
   - Save and activate the workflow in n8n.  
   - Test by triggering an error in any other workflow to verify that a Telegram alert is received.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Keep this workflow active 24/7 to ensure it can capture and notify all errors in real time. If it is inactive, no error alerts will be sent.                                                                                   | Setup Instructions (all languages)                                                             |
| Telegram Bot creation and API token generation are done through the official **@BotFather** bot on Telegram.                                                                                                                    | https://core.telegram.org/bots#6-botfather                                                    |
| Retrieving chat ID requires making a GET request to `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` after messaging your bot.                                                                                            | Telegram Bot API documentation                                                                 |
| Placeholder expressions in message templates use n8n’s standard JavaScript expression syntax, allowing dynamic insertion of error data into notifications.                                                                     | Official n8n Expressions documentation                                                        |
| This workflow is language inclusive with setup instructions provided in English, Spanish, German, French, and Russian for wider accessibility.                                                                                  | Embedded in Sticky Note                                                                        |

---

*Disclaimer:* The provided text is exclusively derived from an automated n8n workflow and strictly complies with applicable content policies. All manipulated data is legal and publicly available.