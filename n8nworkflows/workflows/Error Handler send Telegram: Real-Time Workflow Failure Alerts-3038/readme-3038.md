Error Handler send Telegram: Real-Time Workflow Failure Alerts

https://n8nworkflows.xyz/workflows/error-handler-send-telegram--real-time-workflow-failure-alerts-3038


# Error Handler send Telegram: Real-Time Workflow Failure Alerts

---

### 1. Workflow Overview

This workflow is designed as an error handler that sends real-time failure alerts to a Telegram chat or channel whenever any other n8n workflow encounters an error. It captures detailed error information such as the workflow name, timestamp, execution URL, last executed node, and the error message itself. The workflow is intended for developers, teams, and n8n users who want immediate notifications of workflow failures for faster troubleshooting.

The workflow logic is organized into three main blocks:

- **1.1 Error Detection:** Listens for errors triggered by other workflows.
- **1.2 Configuration Setup:** Holds the Telegram chat ID configuration.
- **1.3 Notification Dispatch:** Sends a formatted message to Telegram with error details.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection

- **Overview:**  
  This block detects when any workflow in the n8n instance fails and triggers the error handling process.

- **Nodes Involved:**  
  - Error Trigger

- **Node Details:**

  - **Error Trigger**  
    - **Type:** `Error Trigger` node (special trigger node in n8n)  
    - **Role:** Listens globally for workflow errors and outputs error data when a failure occurs.  
    - **Configuration:** Default settings; no parameters needed.  
    - **Key Expressions/Variables:** Outputs error metadata including workflow name, execution URL, last executed node, and error message.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Connected to the **Config** node.  
    - **Version Requirements:** Available in n8n versions supporting error workflows (generally v0.148+).  
    - **Potential Failures:** None internal; however, if error data is malformed or missing expected fields, downstream nodes may fail.  
    - **Sub-workflow:** None.

#### 2.2 Configuration Setup

- **Overview:**  
  This block sets the Telegram chat ID dynamically to be used by the Telegram node for sending messages.

- **Nodes Involved:**  
  - Config

- **Node Details:**

  - **Config**  
    - **Type:** `Set` node  
    - **Role:** Stores configuration data, specifically the Telegram chat ID, as a workflow variable.  
    - **Configuration:** Contains one string field `telegramChatId` with a default placeholder value `"123456789"`. This should be replaced with the actual Telegram chat or channel ID (e.g., `-123456789` for channels).  
    - **Key Expressions/Variables:** Outputs JSON with `telegramChatId` used by the Telegram node.  
    - **Input Connections:** Receives input from **Error Trigger** node.  
    - **Output Connections:** Connected to the **Telegram** node.  
    - **Version Requirements:** Uses `Set` node version 3.4 or higher for assignment features.  
    - **Potential Failures:** If the chat ID is invalid or missing, Telegram message sending will fail.  
    - **Sub-workflow:** None.

#### 2.3 Notification Dispatch

- **Overview:**  
  This block sends a detailed error notification message to the configured Telegram chat/channel.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - **Type:** `Telegram` node (messaging node)  
    - **Role:** Sends a message to Telegram with error details formatted in HTML.  
    - **Configuration:**  
      - `text` parameter uses expressions to dynamically build the message with data from the **Error Trigger** node, including:  
        - Workflow name  
        - Current date and time (`$now`)  
        - Execution URL  
        - Last node executed  
        - Error message  
      - `chatId` is dynamically set from the `telegramChatId` output of the **Config** node.  
      - Additional fields: `parse_mode` set to `HTML` for message formatting, and `appendAttribution` disabled to avoid extra text.  
      - Retry enabled with 3-second intervals between tries to handle transient failures.  
    - **Key Expressions/Variables:**  
      - `={{ $('Error Trigger').first().json.workflow.name }}`  
      - `{{ $now }}`  
      - `={{ $('Error Trigger').first().json.execution.url }}`  
      - `={{ $('Error Trigger').first().json.execution.lastNodeExecuted }}`  
      - `={{ $('Error Trigger').first().json.execution.error.message }}`  
      - `={{ $('Config').item.json.telegramChatId }}`  
    - **Input Connections:** Receives input from **Config** node.  
    - **Output Connections:** None (end node).  
    - **Version Requirements:** Uses Telegram node version 1.2 or higher for retry and additional fields support.  
    - **Potential Failures:**  
      - Authentication errors if bot token is invalid or revoked.  
      - Permission errors if bot is not added to the chat or lacks send message rights.  
      - Network timeouts or API rate limits.  
      - Expression evaluation errors if error data is incomplete.  
    - **Sub-workflow:** None.

#### 2.4 Documentation Block (Non-executable)

- **Overview:**  
  Provides detailed usage instructions, prerequisites, setup steps, example notifications, and troubleshooting tips.

- **Nodes Involved:**  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note1**  
    - **Type:** `Sticky Note` node  
    - **Role:** Contains comprehensive documentation and instructions for users to configure and use the workflow properly.  
    - **Content Highlights:**  
      - Telegram bot creation and chat ID retrieval instructions.  
      - Workflow configuration steps including setting chat ID and credentials.  
      - How to set this workflow as an error workflow in other workflows.  
      - Testing instructions and example notification format.  
      - Troubleshooting common issues.  
      - Embedded link to official Telegram BotFather documentation: https://core.telegram.org/bots#botfather  
    - **Input/Output Connections:** None (informational only).  
    - **Version Requirements:** None.  
    - **Potential Failures:** None.

---

### 3. Summary Table

| Node Name       | Node Type          | Functional Role                  | Input Node(s)    | Output Node(s) | Sticky Note                                                                                          |
|-----------------|--------------------|--------------------------------|------------------|----------------|----------------------------------------------------------------------------------------------------|
| Error Trigger   | Error Trigger      | Detects workflow errors         | None             | Config         |                                                                                                    |
| Config          | Set                | Holds Telegram chat ID config   | Error Trigger    | Telegram       |                                                                                                    |
| Telegram        | Telegram           | Sends error notification to Telegram | Config           | None           |                                                                                                    |
| Sticky Note1    | Sticky Note        | Documentation and usage instructions | None             | None           | Contains detailed usage instructions, setup steps, example notification, troubleshooting tips, and link to https://core.telegram.org/bots#botfather |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it `Error Handler send Telegram`.

2. **Add an `Error Trigger` node:**  
   - Drag the **Error Trigger** node onto the canvas.  
   - No configuration needed; it listens for any workflow errors globally.

3. **Add a `Set` node named `Config`:**  
   - Connect the output of **Error Trigger** to **Config**.  
   - In **Config**, add one field:  
     - Name: `telegramChatId`  
     - Type: String  
     - Value: Set a placeholder like `"123456789"` (replace this later with your actual Telegram chat or channel ID).  
   - Use node version 3.4 or higher to ensure assignment features.

4. **Add a `Telegram` node:**  
   - Connect the output of **Config** to **Telegram**.  
   - Configure the **Telegram** node as follows:  
     - **Operation:** Send Message  
     - **Chat ID:** Use expression `={{ $('Config').item.json.telegramChatId }}` to dynamically get the chat ID.  
     - **Text:** Use the following expression to build the message:  
       ```
       Workflow: {{ $('Error Trigger').first().json.workflow.name }}
       Data & Time: {{ $now }}
       URL: {{ $('Error Trigger').first().json.execution.url }}
       Last Node: {{ $('Error Trigger').first().json.execution.lastNodeExecuted }}
       Error Detail: {{ $('Error Trigger').first().json.execution.error.message }}
       ```  
     - **Additional Fields:**  
       - `parse_mode`: `HTML` (to allow formatting if needed)  
       - `appendAttribution`: `false` (to avoid extra attribution text)  
     - **Credentials:** Add or select your Telegram bot credentials (bot token).  
     - **Retry on Fail:** Enable with 3 seconds wait between tries to handle transient errors.

5. **Add a `Sticky Note` node (optional but recommended):**  
   - Add a sticky note with detailed instructions on how to configure the workflow, including:  
     - Creating the Telegram bot via BotFather.  
     - Setting the correct `telegramChatId`.  
     - Assigning Telegram credentials.  
     - How to set this workflow as the error workflow in other workflows.  
     - Testing and troubleshooting tips.  
     - Include the link: https://core.telegram.org/bots#botfather

6. **Activate the workflow:**  
   - Toggle the workflow to active to enable error monitoring.

7. **Set as Error Workflow in other workflows:**  
   - In any workflow you want to monitor, go to **Settings** > **Error Workflow** and select this `Error Handler send Telegram` workflow.

8. **Test the setup:**  
   - Trigger an error in any monitored workflow.  
   - Confirm that a detailed error message is sent to the configured Telegram chat/channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                         |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| Telegram bot creation and management is done via BotFather.                                                                      | https://core.telegram.org/bots#botfather               |
| The workflow requires the Telegram and Error Trigger nodes installed in your n8n instance.                                       | n8n node documentation                                 |
| Retry mechanism in the Telegram node helps mitigate transient network or API issues.                                             | n8n Telegram node settings                              |
| The error message uses HTML parse mode for better formatting in Telegram messages.                                               | Telegram Bot API documentation                          |
| Ensure the Telegram bot has permission to send messages in the target chat or channel.                                           | Telegram group/channel settings                         |
| The workflow is designed for n8n versions supporting error workflows and Set node version 3.4+.                                  | n8n changelog and version notes                         |

---

This completes the comprehensive reference documentation for the "Error Handler send Telegram" workflow.