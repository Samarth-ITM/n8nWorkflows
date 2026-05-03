Build a private Llama chatbot with Ollama, Groq, Slack and Google Sheets

https://n8nworkflows.xyz/workflows/build-a-private-llama-chatbot-with-ollama--groq--slack-and-google-sheets-13811


# Build a private Llama chatbot with Ollama, Groq, Slack and Google Sheets

# 1. Workflow Overview

This workflow implements a private, self-hosted AI chatbot system utilizing Meta’s Llama models. It is designed to run either locally (via Ollama) or through high-performance cloud endpoints (Groq or Together AI) while maintaining control over data flow. The system features intent-based routing, conversation memory management, automated logging, and a human-in-the-loop escalation path.

The workflow is structured into several functional stages:
*   **1.1 Intake & Configuration:** Receives the incoming message and sets global environment variables.
*   **1.2 Session Memory & Intent Routing:** Manages conversation history and classifies the user's intent (Support, Sales, General, or Escalation) to select the appropriate system prompt.
*   **1.3 AI Inference:** Executes the request to the Llama model using a unified API format.
*   **1.4 Post-Processing & Logic:** Parses the AI's response, updates the session history, and determines if human intervention is required.
*   **1.5 Audit & Delivery:** Logs interactions to Google Sheets, alerts staff via Slack if necessary, and returns the response to the user.

---

# 2. Block-by-Block Analysis

### 2.1 Intake & Configuration
**Overview:** This block acts as the entry point, capturing incoming data and defining the operational parameters for the AI model and external integrations.

*   **Receive Chat Message (Webhook):**
    *   **Type:** Webhook
    *   **Configuration:** Listens for `POST` requests at `/llama-chat`.
    *   **Input/Output:** Receives raw JSON; outputs normalized object.
    *   **Edge Cases:** Missing `sessionId` or `message` in the body will cause downstream failures if not handled.
*   **Set Llama Config (Set):**
    *   **Type:** Set
    *   **Role:** Configuration Manager.
    *   **Key Expressions:** Generates unique `messageId` and provides defaults for `LLAMA_ENDPOINT`, `LLAMA_MODEL`, and API keys.
    *   **Failure Types:** Incorrect endpoint URLs or missing API keys will lead to HTTP errors in the inference stage.

### 2.2 Memory & Intent Routing
**Overview:** This block processes the user input to maintain context and decide the "personality" of the bot based on the message content.

*   **Load Memory and Build Request (Code):**
    *   **Type:** Code (JavaScript)
    *   **Role:** Context & Intent Engine.
    *   **Logic:** 
        *   Extracts history from the request body (stateless memory).
        *   Uses Regex to detect keywords (e.g., "refund" -> Support; "price" -> Sales).
        *   Maps intents to specific System Prompts.
        *   Formats the payload for either Ollama (`/api/chat`) or OpenAI-compatible cloud providers (`/chat/completions`).
    *   **Edge Cases:** Empty messages or extremely long conversation histories exceeding token limits.

### 2.3 AI Inference
**Overview:** The communication layer between n8n and the Llama model.

*   **Call Llama Model (HTTP Request):**
    *   **Type:** HTTP Request
    *   **Configuration:** Dynamically switches between Local/Cloud URLs. Set to a 90-second timeout to handle large model inference.
    *   **Authentication:** Uses `Bearer` tokens if a key is provided, or a dummy "ollama" token for local instances.
    *   **Failure Types:** Connection refused (Ollama not running), Timeout, or Rate Limiting (Groq/Together AI).

### 2.4 Response Processing
**Overview:** Normalizes the different AI provider responses and updates the state.

*   **Parse Reply and Update Memory (Code):**
    *   **Type:** Code (JavaScript)
    *   **Role:** Response Parser.
    *   **Logic:** Extracts the text content from different JSON structures (Ollama vs. Groq). It also performs a second check for escalation keywords in the *AI's* response.
    *   **Output:** A clean object containing the assistant's reply and the updated history array.

### 2.5 Audit & Delivery
**Overview:** The final stage handles data persistence, human alerts, and user response.

*   **Log Conversation Turn (Google Sheets):**
    *   **Type:** Google Sheets (Append)
    *   **Role:** Audit Logging. 
    *   **Requirement:** Requires a pre-configured sheet with headers matching the output keys.
*   **Check Escalation Flag (IF):**
    *   **Type:** IF
    *   **Role:** Logical Gate. Checks the `needsEscalation` boolean.
*   **Alert Slack Escalation (HTTP Request):**
    *   **Type:** HTTP Request
    *   **Role:** Notification. Sends a detailed Slack block kit message containing session details and the conversation snippet.
*   **Send Chatbot Reply (Respond to Webhook):**
    *   **Type:** Respond to Webhook
    *   **Role:** Final Output. Returns a structured JSON to the initial requester with the AI reply and metadata.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Chat Message | Webhook | Entry Point | (None) | Set Llama Config | Stage A+B — Intake & Memory |
| Set Llama Config | Set | Config Management | Receive Chat Message | Load Memory... | Stage A+B — Intake & Memory |
| Load Memory and Build Request | Code | Intent & History | Set Llama Config | Call Llama Model | Stage C+D — Route & Infer |
| Call Llama Model | HTTP Request | AI Inference | Load Memory... | Parse Reply... | Stage C+D — Route & Infer |
| Parse Reply and Update Memory | Code | Response Parsing | Call Llama Model | Log Turn, Check Flag | Stage E+F — Parse, Log & Deliver |
| Log Conversation Turn | Google Sheets | Data Logging | Parse Reply... | Send Chatbot Reply | Stage E+F — Parse, Log & Deliver |
| Check Escalation Flag | IF | Logic Branching | Parse Reply... | Alert Slack, Send Reply | Stage E+F — Parse, Log & Deliver |
| Alert Slack Escalation | HTTP Request | Human Alert | Check Escalation Flag | Send Chatbot Reply | Stage E+F — Parse, Log & Deliver |
| Send Chatbot Reply | Respond to Webhook | Final Delivery | Multiple | (None) | Stage E+F — Parse, Log & Deliver |

---

# 4. Reproducing the Workflow from Scratch

1.  **Setup the Webhook:** Create a **Webhook** node. Set the HTTP Method to `POST` and Path to `llama-chat`. Set the Response Mode to "When Last Node Finishes".
2.  **Initialize Variables:** Add a **Set** node. Define strings for `sessionId`, `userMessage`, `LLAMA_ENDPOINT`, `LLAMA_API_KEY`, `LLAMA_MODEL`, `SLACK_WEBHOOK`, and `SHEET_ID`.
3.  **Implement Logic & Memory:** Add a **Code** node. Use JavaScript to:
    *   Slice history to the last 20 messages.
    *   Determine intent via regex (e.g., searching for "help" or "price").
    *   Construct the `messages` array starting with a specialized `system` prompt.
    *   Format the `requestPayload` JSON based on whether the endpoint is local (Ollama) or cloud-based.
4.  **Connect Inference:** Add an **HTTP Request** node.
    *   **Method:** `POST`.
    *   **URL:** `{{ $json.LLAMA_ENDPOINT + $json.apiPath }}`.
    *   **Headers:** `Content-Type: application/json` and `Authorization: Bearer {{ API_KEY }}`.
    *   **Body:** Send the `requestPayload` from the previous step.
5.  **Parse Response:** Add another **Code** node to extract the text from `raw.message.content` (Ollama) or `raw.choices[0].message.content` (Groq). Calculate `needsEscalation` by checking both intent and bot response keywords.
6.  **Storage & Alerts:**
    *   Connect a **Google Sheets** node (Append operation) to log all variables to a spreadsheet.
    *   Connect an **IF** node to check if `needsEscalation` is `true`.
    *   If `true`, connect an **HTTP Request** node to your Slack Webhook URL using a Block Kit payload for rich formatting.
7.  **Final Response:** Connect the **Respond to Webhook** node. Ensure it receives input from the Slack node, the Google Sheets node, or the IF node (false branch). Return the AI's reply and the updated history.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Ollama Documentation** | Use `http://localhost:11434` for local testing. [Ollama API](https://github.com/ollama/ollama/blob/main/docs/api.md) |
| **Groq Cloud Console** | High-speed Llama 3.1 inference. [Groq API](https://console.groq.com/) |
| **Slack Block Kit Builder** | Used to customize the escalation alert appearance. [Slack Block Kit](https://app.slack.com/block-kit-builder) |
| **Data Privacy** | If using Ollama locally, no conversation data is sent to external AI providers. |