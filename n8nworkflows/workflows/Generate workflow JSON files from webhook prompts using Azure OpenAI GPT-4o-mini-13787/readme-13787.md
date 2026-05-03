Generate workflow JSON files from webhook prompts using Azure OpenAI GPT-4o-mini

https://n8nworkflows.xyz/workflows/generate-workflow-json-files-from-webhook-prompts-using-azure-openai-gpt-4o-mini-13787


# Generate workflow JSON files from webhook prompts using Azure OpenAI GPT-4o-mini

# 1. Workflow Overview

This workflow acts as an automated "n8n Architect" that transforms plain-text descriptions of automations into valid, import-ready n8n workflow `.json` files. It exposes a webhook endpoint to receive a prompt, utilizes Azure OpenAI's GPT-4o-mini via a structured LangChain agent to generate the logic, and returns a binary file response that the user can directly import into their n8n instance.

The logic is organized into three functional blocks:
*   **1.1 Input & Prompt Prep:** Receives the raw HTTP POST request and cleans the text input.
*   **1.2 AI Generation:** Uses a Large Language Model with strict system instructions to construct the n8n JSON structure.
*   **1.3 Output Processing & Response:** Validates the AI's response, converts the object into a file, and delivers it via HTTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Prompt Prep
**Overview:** This block handles the entry point of the workflow and ensures the input data is formatted correctly for the AI model.

*   **Receive Workflow Prompt**
    *   **Type:** Webhook (v2.1)
    *   **Configuration:** HTTP POST method; `responseMode` set to "Last Node" (Response Node) to allow the workflow to return a file.
    *   **Input/Output:** Receives a JSON body with a `prompt` field; outputs the raw body data.
    *   **Failure Cases:** 404 if path is incorrect; 400 if the body is not valid JSON.

*   **Sanitize & Normalize Prompt**
    *   **Type:** Code (v2)
    *   **Configuration:** JavaScript snippet that extracts the prompt, normalizes Windows line breaks (`\r\n` to `\n`), and trims whitespace.
    *   **Key Expression:** `let prompt = $json.body?.prompt || "";`
    *   **Failure Cases:** Expression failure if the input body is unexpectedly null.

#### 2.2 AI Generation
**Overview:** The core logic where the text prompt is translated into technical n8n nodes and connections.

*   **Generate Workflow via AI Agent**
    *   **Type:** AI Agent (LangChain v3)
    *   **Configuration:** Set to "Define" prompt type. The system message contains extensive rules: outputting only raw JSON, no markdown backticks, specific node naming conventions, and required n8n workflow metadata (nodes, connections, settings).
    *   **Key Logic:** It instructs the AI to map keywords (e.g., "time", "email") to specific n8n node types (e.g., `n8n-nodes-base.scheduleTrigger`, `n8n-nodes-base.gmail`).
    *   **Failure Cases:** AI "hallucinations" (invalid JSON), token limits, or timeout during generation.

*   **Azure OpenAI Chat Model1**
    *   **Type:** Azure OpenAI Chat Model (v1)
    *   **Configuration:** Uses the `gpt-4o-mini` model.
    *   **Credentials:** Requires `azureOpenAiApi` credentials with a deployment matching the model name.
    *   **Role:** Acts as the processing engine for the AI Agent.

#### 2.3 Output Processing & Response
**Overview:** Finalizes the data by ensuring it is a valid file and sending it back to the user.

*   **Parse & Validate AI JSON Output**
    *   **Type:** Code (v2)
    *   **Configuration:** JavaScript `JSON.parse()` wrapper.
    *   **Key Role:** It validates that the AI actually returned a string that can be parsed as JSON.
    *   **Failure Cases:** Throws a hard error if the AI returns non-JSON text or markdown, which stops the workflow.

*   **Convert Workflow to JSON File**
    *   **Type:** Convert to File (v1.1)
    *   **Configuration:** Operation set to `toJson`.
    *   **Output:** Creates a binary file buffer from the validated JSON object.

*   **Return JSON File to Caller**
    *   **Type:** Respond to Webhook (v1.5)
    *   **Configuration:** `respondWith` set to "Binary".
    *   **Input:** Receives the binary file from the previous node.
    *   **Output:** Returns the `.json` file as the HTTP response.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Workflow Prompt | Webhook | Entry Point | (None) | Sanitize & Normalize Prompt | Receives the raw POST request and extracts the prompt field. |
| Sanitize & Normalize Prompt | Code | Data Cleaning | Receive Workflow Prompt | Generate Workflow via AI Agent | Normalizes line breaks and trims whitespace so the AI always gets clean, consistent input. |
| Generate Workflow via AI Agent | AI Agent | Logic Generation | Sanitize & Normalize Prompt | Parse & Validate AI JSON Output | The AI Agent uses Azure OpenAI GPT-4o-mini with a strict system prompt to produce a complete, import-ready n8n workflow JSON. |
| Azure OpenAI Chat Model1 | Azure OpenAI Chat Model | AI Engine | (None) | Generate Workflow via AI Agent | Requires a valid azureOpenAiApi credential with an active deployment named gpt-4o-mini. |
| Parse & Validate AI JSON Output | Code | Validation | Generate Workflow via AI Agent | Convert Workflow to JSON File | Parses and validates the AI's raw string output into structured JSON. |
| Convert Workflow to JSON File | Convert to File | File Formatting | Parse & Validate AI JSON Output | Return JSON File to Caller | Converts JSON into a downloadable file. |
| Return JSON File to Caller | Respond to Webhook | HTTP Response | Convert Workflow to JSON File | (None) | Returns it to the caller as a binary HTTP response. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:**
    *   Add a **Webhook** node. Set the HTTP Method to `POST`. Set `Response Mode` to `When Last Node Finishes`.
2.  **Input Sanitation:**
    *   Add a **Code** node. Use JavaScript to extract `$json.body.prompt` and apply `.trim()` and `.replace()` to clean the text.
3.  **AI Integration:**
    *   Add an **AI Agent** node (LangChain).
    *   Add an **Azure OpenAI Chat Model** node and connect it to the Agent's "Language Model" input.
    *   In the Chat Model node, select your Azure credentials and set the model to `gpt-4o-mini`.
    *   In the Agent node, set the **System Message**. Instruct the model to:
        *   Output *only* valid JSON.
        *   Include `nodes`, `connections`, and `active: false`.
        *   Use official n8n node types (e.g., `n8n-nodes-base.httpRequest`).
        *   Avoid all markdown formatting (no backticks).
4.  **Validation Logic:**
    *   Add another **Code** node after the Agent.
    *   Use `JSON.parse($json.output)` to ensure the string is valid. Return the parsed object.
5.  **File Transformation:**
    *   Add a **Convert to File** node. Set the operation to `Convert to JSON`.
6.  **Final Response:**
    *   Add a **Respond to Webhook** node. Set `Respond With` to `Binary File`.
7.  **Connectivity:**
    *   Connect the nodes in sequence: Webhook -> Code -> Agent -> Code -> Convert to File -> Respond to Webhook.
    *   Ensure the Azure OpenAI model is connected to the Agent.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Credential Prerequisite** | Requires `azureOpenAiApi` with a specific deployment named `gpt-4o-mini`. |
| **Input Format** | Expects a JSON body: `{"prompt": "text description"}` via POST request. |
| **AI Reliability** | If the AI includes introductory text or markdown backticks despite instructions, the "Parse & Validate" node will intentionally fail to prevent corrupted file downloads. |
| **Customization** | To support newer n8n nodes, update the "Logic Rules" inside the AI Agent's system message. |