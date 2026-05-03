Monitor workflow errors via n8n API with Gemini analysis and Telegram alerts

https://n8nworkflows.xyz/workflows/monitor-workflow-errors-via-n8n-api-with-gemini-analysis-and-telegram-alerts-13846


# Monitor workflow errors via n8n API with Gemini analysis and Telegram alerts

# Workflow Reference: AI-Powered n8n Error Monitor

This workflow provides an automated diagnostic system for n8n. It intercepts execution failures from any workflow on the instance, utilizes the n8n API to retrieve the failed workflow's structure, and employs **Google Gemini AI** to generate a technical root-cause analysis and actionable fix steps, delivered instantly via **Telegram**.

---

### 1. Workflow Overview

The workflow logic is organized into four functional blocks:

*   **1.1 Error Detection & Context Definition:** Captures the error event and stores global configuration (API keys, URLs) and error-specific metadata (Node name, error message, stack trace).
*   **1.2 Metadata Enrichment:** Uses the n8n REST API to fetch the full JSON definition of the workflow that failed, providing the AI with the necessary context to understand the node logic.
*   **1.3 AI Diagnostic Engine:** A LangChain-based agent analyzes the error against the workflow definition. It classifies the error (e.g., Auth, Rate Limit, Connection) and generates a structured HTML report.
*   **1.4 Alert Dispatch:** Sends the finalized diagnostic report to a designated Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection & Context Definition
This block acts as the entry point and configuration hub.

*   **Nodes Involved:** `Error Trigger`, `Set Context`.
*   **Node Details:**
    *   **Error Trigger:** Listen for failures across the instance. It outputs the workflow name, ID, and execution error details.
    *   **Set Context (Set Node):** 
        *   **Role:** Configuration hub.
        *   **Manual Parameters:** `n8n_instance_url`, `n8n_api_key`, `telegram_chat_id`.
        *   **Dynamic Parameters:** Captures `error_workflow_id`, `error_node`, `error_message`, and `execution_id` from the trigger.
        *   **Edge Case:** If the API key or URL is incorrect, the subsequent HTTP node will fail.

#### 2.2 Metadata Enrichment
Fetches the "Internal" logic of the failed workflow to help the AI understand *why* it failed.

*   **Nodes Involved:** `Get Workflow Content`.
*   **Node Details:**
    *   **Get Workflow Content (HTTP Request):** 
        *   **Technical Role:** GET request to the n8n API (`/api/v1/workflows/{id}`).
        *   **Configuration:** Sends `X-N8N-API-KEY` in headers. 
        *   **Settings:** "On Error: Continue" is enabled to ensure the flow doesn't crash if the API is unreachable, though AI analysis quality will drop.

#### 2.3 AI Diagnostic Engine
The "brain" of the workflow that transforms raw stack traces into human-readable solutions.

*   **Nodes Involved:** `AI Agent`, `Google Gemini Chat Model`, `Structured Output Parser`.
*   **Node Details:**
    *   **AI Agent (LangChain):**
        *   **Configuration:** Uses a System Message to enforce "Error Classification" (e.g., mapping HTTP 429 to "Rate Limit Error").
        *   **Prompting:** Passes the Workflow Definition JSON and the Stack Trace to the LLM.
        *   **Output Constraint:** Forces a JSON response with a single `explanation` field containing Telegram-compatible HTML tags.
    *   **Google Gemini Chat Model:** Connects to the `googlePalmApi` credential. Processes the reasoning logic.
    *   **Structured Output Parser:** Ensures the AI's response is strictly valid JSON, preventing Telegram errors from unexpected AI text formatting.

#### 2.4 Alert Dispatch
Delivers the final report.

*   **Nodes Involved:** `Send Telegram Notification`.
*   **Node Details:**
    *   **Send Telegram Notification:** 
        *   **Configuration:** Uses HTML parse mode. 
        *   **Input:** Data from the `AI Agent`'s `explanation` field.
        *   **Target:** Uses the `telegram_chat_id` defined in the "Set Context" node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Error Trigger** | n8n-nodes-base.errorTrigger | Entry point for failures | None | Set Context | Catch errors from any workflow. |
| **Set Context** | n8n-nodes-base.set | Configuration & Variable mapping | Error Trigger | Get Workflow Content | Update these 3 fields only: instance_url, api_key, telegram_chat_id. |
| **Get Workflow Content** | n8n-nodes-base.httpRequest | Fetches failed workflow JSON | Set Context | AI Agent | Fetches full workflow definition via n8n API. |
| **AI Agent** | @n8n/n8n-nodes-langchain.agent | Logic & Report Generation | Get Workflow Content | Send Telegram Notification | Classifies error, identifies cause, and writes report. |
| **Google Gemini Chat Model** | langchain.lmChatGoogleGemini | LLM Processing | AI Agent | AI Agent | Connect Google Gemini credential here. |
| **Structured Output Parser** | langchain.outputParserStructured | JSON Schema Enforcement | AI Agent | AI Agent | Ensures AI returns valid JSON with HTML field. |
| **Send Telegram Notification** | n8n-nodes-base.telegram | Message Delivery | AI Agent | None | Deliver the formatted HTML report to your chat. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Credentials Setup:**
    *   Create a **Google Gemini (Palm) API** credential.
    *   Create a **Telegram Bot API** credential and obtain your Chat ID (via `@userinfobot`).
    *   Generate an **n8n API Key** (Settings > API).

2.  **Initial Triggers & Config:**
    *   Add an **Error Trigger** node.
    *   Connect it to a **Set** node named "Set Context". Add String variables: `n8n_instance_url`, `n8n_api_key`, `telegram_chat_id`. 
    *   Map the dynamic variables from the trigger (Workflow ID, Name, Error Message, etc.) into the Set node.

3.  **Data Fetching:**
    *   Add an **HTTP Request** node. Set Method to `GET`. 
    *   URL: `{{ $json.n8n_instance_url }}/api/v1/workflows/{{ $json.error_workflow_id }}`.
    *   Add Header: `X-N8N-API-KEY` with your API key variable.

4.  **AI Integration:**
    *   Add an **AI Agent** node. Set "Agent" to `ReAct` or `Conversational`.
    *   Attach a **Google Gemini Chat Model** node to the Agent's sub-connection.
    *   Attach a **Structured Output Parser** node to the Agent's sub-connection. Define a JSON schema with one property: `explanation` (string).
    *   Paste the system prompt into the Agent (Instruction: Classify error, use HTML tags `<b>`, `<code>`, and return only JSON).

5.  **Output:**
    *   Add a **Telegram** node. Set "Chat ID" to the variable from your "Set Context" node.
    *   Set "Text" to `{{ $json.output.explanation }}`.
    *   **Crucial:** In "Additional Fields," set **Parse Mode** to `HTML`.

6.  **Activation:**
    *   Save and activate the workflow.
    *   Go to any *other* workflow > Settings > **Error Workflow** > Select this workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Credits** | Created by Nguyen Thieu Toan (Jay Nguyen) - Verified n8n Creator. |
| **Donations & Support** | [Support the Creator](https://nguyenthieutoan.com/payment/) |
| **Author Portfolio** | [n8n Creator Profile](https://n8n.io/creators/nguyenthieutoan/) |
| **Customization Tip** | To change language, modify the "Report Template" section inside the AI Agent system message. |
| **Security Note** | Ensure your n8n API key is kept secure and not hardcoded in nodes visible to unauthorized users. |