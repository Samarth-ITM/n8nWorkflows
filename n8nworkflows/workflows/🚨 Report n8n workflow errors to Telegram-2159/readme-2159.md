🚨 Report n8n workflow errors to Telegram

https://n8nworkflows.xyz/workflows/---report-n8n-workflow-errors-to-telegram-2159


# 🚨 Report n8n workflow errors to Telegram

### 1. Workflow Overview

This workflow is designed to serve as a centralized error reporting mechanism for n8n workflows in production environments. When any workflow that has this error workflow set as its error handler encounters a failure, this workflow triggers automatically to send a notification message via Telegram. The notification includes the name of the failed workflow and a direct link to the execution details, enabling quick access for debugging.

The workflow is structured into three logical blocks:

- **1.1 Error Trigger Reception:** Captures the error event from any workflow using this as their error workflow.
- **1.2 Message Preparation:** Constructs a formatted message containing the workflow name and execution link.
- **1.3 Telegram Notification:** Sends the constructed message to a predefined Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Trigger Reception

- **Overview:**  
  This block listens for any workflow error events and triggers this workflow. It acts as the entry point for the error reporting process.

- **Nodes Involved:**  
  - On Error

- **Node Details:**

  - **Node Name:** On Error  
    - **Type:** Error Trigger  
    - **Technical Role:** Initiates the workflow when an error occurs in any workflow that uses this error workflow.  
    - **Configuration Choices:** Default settings, no parameters required.  
    - **Input/Output:** No input; outputs the error execution data to the next node.  
    - **Version Requirements:** Compatible with n8n versions that support error triggers (v0.154.0+).  
    - **Potential Failure Types:** Rare failure; possible issues if the node is not properly set as an error workflow in other workflows.  
    - **Sub-workflow Reference:** This node is the entry point of this error reporting sub-workflow.

#### 1.2 Message Preparation

- **Overview:**  
  Constructs a user-friendly message string that includes the name of the failed workflow and a clickable link to the failed execution in the n8n UI.

- **Nodes Involved:**  
  - Set message

- **Node Details:**

  - **Node Name:** Set message  
    - **Type:** Set  
    - **Technical Role:** Creates a new JSON field `message` with a templated string containing error info.  
    - **Configuration Choices:** Uses an expression to dynamically insert workflow name and execution URL from the error trigger data. The message format is:  
      `⚠️ Workflow <workflow name> failed to run! [execution](<execution url>)`  
    - **Key Expressions:**  
      - `{{$json["workflow"]["name"]}}` — inserts the workflow name.  
      - `{{$json.execution.url}}` — inserts the execution URL.  
    - **Input/Output:** Takes the error data from On Error node; outputs JSON with the new `message` field for Telegram node.  
    - **Version Requirements:** No special requirements; expression syntax compatible with recent n8n versions.  
    - **Potential Failure Types:** Expression failures if the input JSON structure changes or if keys are missing (e.g., missing workflow name or execution URL).  
    - **Sub-workflow Reference:** Part of this error reporting workflow.

#### 1.3 Telegram Notification

- **Overview:**  
  Sends the prepared error notification message to a specified Telegram chat using the Telegram API credentials.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Node Name:** Telegram  
    - **Type:** Telegram  
    - **Technical Role:** Sends messages to a Telegram chat using bot credentials.  
    - **Configuration Choices:**  
      - `Chat ID` is set statically to `1688282582` (must be configured for your target chat).  
      - `Text` field uses the expression `={{ $json.message }}` to send the custom error message.  
      - `appendAttribution` is disabled to avoid adding "Sent via n8n" attribution.  
    - **Input/Output:** Receives input from Set message node; outputs Telegram API response.  
    - **Version Requirements:** Requires Telegram API credentials configured in n8n.  
    - **Potential Failure Types:**  
      - Authentication errors if Telegram bot token is invalid or expired.  
      - Chat ID errors if the chat ID is incorrect or bot is not added to the chat.  
      - Network timeouts or API rate limits.  
    - **Sub-workflow Reference:** Final node in this error reporting workflow.

---

### 3. Summary Table

| Node Name    | Node Type       | Functional Role              | Input Node(s)    | Output Node(s) | Sticky Note                                           |
|--------------|-----------------|-----------------------------|------------------|----------------|-------------------------------------------------------|
| On Error     | Error Trigger   | Error event reception       | —                | Set message    |                                                       |
| Set message  | Set             | Prepare error message       | On Error         | Telegram       |                                                       |
| Telegram     | Telegram        | Send message to Telegram    | Set message      | —              | ### 👆🏽 Set chat id here                              |
| Sticky Note3 | Sticky Note     | Setup instructions          | —                | —              | ### 👨‍🎤 Setup<br>1. Add Telegram creds<br>2. Set chat id in **Telegram** node<br>2. Add this error workflow to other workflows<br>https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow |
| Sticky Note  | Sticky Note     | Chat id reminder            | —                | —              | ### 👆🏽 Set chat id here                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Error Trigger node:**  
   - Add a new node of type **Error Trigger** named `On Error`.  
   - No parameters need to be set. This node will start the workflow when an error occurs.

2. **Create the Set node to prepare the message:**  
   - Add a **Set** node named `Set message`.  
   - Configure it to add a new string field named `message`.  
   - Set the value to the following expression:  
     ```  
     =⚠️ Workflow `{{$json["workflow"]["name"]}}` failed to run! [execution]({{ $json.execution.url }})  
     ```  
   - Enable "Keep Only Set" to ensure only this field is passed on.  
   - Connect the output of `On Error` to the input of `Set message`.

3. **Create the Telegram node to send the message:**  
   - Add a **Telegram** node named `Telegram`.  
   - Set the **Text** parameter to the expression: `={{ $json.message }}`.  
   - Set the **Chat ID** parameter to the Telegram chat ID where you want to receive alerts (e.g., `1688282582`).  
   - Under **Additional Fields**, disable `appendAttribution` to prevent the "Sent via n8n" tag.  
   - Connect the output of `Set message` to the input of `Telegram`.

4. **Configure Telegram credentials:**  
   - In n8n, create or select existing Telegram API credentials using your bot token.  
   - Assign these credentials to the Telegram node.

5. **Add sticky notes (optional but recommended):**  
   - Add a sticky note near the start with setup instructions:  
     ```
     1. Add Telegram creds  
     2. Set chat id in Telegram node  
     3. Add this error workflow to other workflows  
     https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow
     ```  
   - Add another sticky note near the Telegram node reminding to set the chat ID.

6. **Set this workflow as the error workflow in other workflows:**  
   - In other workflows, go to **Settings > Error Workflow** and select this workflow to capture errors.

7. **Save and activate the error workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Error workflows are critical for production-grade automation to ensure quick error visibility and handling.   | n8n Documentation: https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow       |
| Telegram chats require bot membership and correct chat ID for successful message delivery.                     | Telegram Bot API: https://core.telegram.org/bots/api                                                      |
| Disabling `appendAttribution` in Telegram node prevents cluttering messages with automated tags.               | n8n Telegram node docs                                                                                    |

---

This document provides a full understanding of the “🚨 Report n8n workflow errors to Telegram” workflow, enabling reproduction, customization, and troubleshooting.