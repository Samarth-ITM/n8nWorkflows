Decide multi‑agent vs simple workflows using Azure OpenAI GPT‑4o‑mini

https://n8nworkflows.xyz/workflows/decide-multi-agent-vs-simple-workflows-using-azure-openai-gpt-4o-mini-13792


# Decide multi‑agent vs simple workflows using Azure OpenAI GPT‑4o‑mini

# AI-Powered n8n Workflow Architecture Decision Engine

This workflow acts as an **intelligent advisor** for n8n developers. It evaluates a problem description submitted via a web request and determines whether the solution requires a sophisticated **Multi-Agent Architecture** or a **Simple Linear Workflow**. Using Azure OpenAI's GPT-4o-mini, it designs the necessary agents (including node types and purposes) or provides a streamlined logic plan, returning the result as a professionally styled HTML dashboard.

---

### 1. Workflow Overview

The workflow is structured into four functional phases:

*   **1.1 Input Reception:** A webhook endpoint receives a JSON payload containing the problem description.
*   **1.2 AI Decision Logic:** An AI Agent, powered by Azure OpenAI, applies specific architectural rules to decide the complexity of the required solution.
*   **1.3 Data Parsing:** A custom script transforms the AI's natural language response into structured JSON objects (Decision, Agents, and Steps).
*   **1.4 Presentation & Delivery:** The structured data is injected into an HTML/CSS template and returned as an immediate response to the initial caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Extraction
This block handles the entry point and ensures the data is in the correct format for the AI.

*   **Nodes Involved:** `Receive Problem Description via POST`, `Extract Request Body`.
*   **Node Details:**
    *   **Receive Problem Description via POST (Webhook):** Listens for `POST` requests. Configured to `Respond to Webhook` mode, meaning it waits for the final node to provide a response to the client.
    *   **Extract Request Body (Code):** A simple JavaScript snippet that isolates the `body` of the incoming request to prevent downstream nodes from dealing with unnecessary metadata.
    *   **Edge Cases:** If the incoming request is not JSON or lacks a `description` field, the AI prompt may fail or produce irrelevant results.

#### 2.2 AI Architecture Decision
The core "brain" of the workflow that performs the technical analysis.

*   **Nodes Involved:** `Multi-Agent Architecture Decision Agent`, `Azure OpenAI GPT-4o-mini`.
*   **Node Details:**
    *   **Multi-Agent Architecture Decision Agent (AI Agent):** Uses a "Define" prompt type. It contains strict system instructions to differentiate between multi-step reasoning (Multi-Agent) and direct transformations (Simple).
    *   **Azure OpenAI GPT-4o-mini (Language Model):** Connected to the agent. It uses the `gpt-4o-mini` model for cost-effective yet high-speed reasoning.
    *   **Key Expressions:** The prompt dynamically injects `{{ $json.description }}` from the input.
    *   **Potential Failures:** Requires valid Azure OpenAI credentials and an active deployment named `gpt-4o-mini`. Authentication errors or quota limits here will halt the workflow.

#### 2.3 Parse, Build & Respond
This block converts AI-generated text into a user-friendly visual format.

*   **Nodes Involved:** `Parse Decision, Agents & Steps`, `Build HTML Architecture Report`, `Return HTML Report to Caller`.
*   **Node Details:**
    *   **Parse Decision, Agents & Steps (Code):** Uses Regular Expressions (Regex) to extract the "Decision," "Agent Name," "Purpose," "Node," and "Reason" from the AI's text output.
    *   **Build HTML Architecture Report (Code):** Takes the parsed JSON and maps it into an HTML template. It uses conditional CSS classes (e.g., green for Multi-Agent, orange for Simple) and creates a responsive grid for agent cards.
    *   **Return HTML Report to Caller (Respond to Webhook):** Sends the final `html` string back to the user who triggered the webhook.
    *   **Edge Cases:** If the AI changes its output format slightly, the Regex in the parsing node might fail to capture all agents.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Problem Description via POST | Webhook | Entry point for requests | None | Extract Request Body | ## 📥 Input & Extraction Receives a POST request with a description field and extracts the raw request body for downstream processing. |
| Extract Request Body | Code | Data cleaning | Receive Problem Description via POST | Multi-Agent Architecture Decision Agent | ## 📥 Input & Extraction Receives a POST request with a description field and extracts the raw request body for downstream processing. |
| Multi-Agent Architecture Decision Agent | AI Agent | Architectural reasoning | Extract Request Body | Parse Decision, Agents & Steps | ## 🤖 AI Architecture Decision The AI Agent evaluates the problem description and decides if a multi-agent workflow is needed. |
| Azure OpenAI GPT-4o-mini | Azure Chat Model | LLM Provider | None | Multi-Agent Architecture Decision Agent | ⚠️ Azure OpenAI Credentials Required |
| Parse Decision, Agents & Steps | Code | Text to JSON parsing | Multi-Agent Architecture Decision Agent | Build HTML Architecture Report | ## 📊 Parse, Build & Respond Parses the AI output into structured decision, agents, and steps. |
| Build HTML Architecture Report | Code | UI Generation | Parse Decision, Agents & Steps | Return HTML Report to Caller | ## 📊 Parse, Build & Respond Renders a styled HTML dashboard. |
| Return HTML Report to Caller | Respond to Webhook | Final response delivery | Build HTML Architecture Report | None | ## 📊 Parse, Build & Respond Returns it as the final webhook response. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:**
    *   Create a **Webhook** node. Set HTTP Method to `POST`.
    *   Set **Response Mode** to `When Last Node Finishes`.
2.  **Data Extraction:**
    *   Connect a **Code** node. Use JavaScript to return only `$input.first().json.body`.
3.  **AI Engine Setup:**
    *   Create an **AI Agent** node.
    *   Set **Prompt Type** to `Define`.
    *   **System Message:** Paste instructions defining when to use Multi-Agent (complex reasoning) vs. Simple (single-step). Define the required output format (Decision, Agent Name, Purpose, Node, Reason, Workflow Flow).
    *   **Prompt:** Use: `Analyze the following: {{ $json.description }}`.
4.  **Model Configuration:**
    *   Attach an **Azure OpenAI Chat Model** node to the Agent.
    *   Input your Azure Credentials and set the Model/Deployment name to `gpt-4o-mini`.
5.  **Parsing Logic:**
    *   Connect a **Code** node after the Agent.
    *   Use Regex (`match` and `exec`) to extract the specific fields from the `output` string into a structured JSON array of agents and steps.
6.  **Visual Builder:**
    *   Connect another **Code** node.
    *   Create a template string containing `<html>`, `<style>`, and `<body>`.
    *   Map the JSON data into HTML `div` elements for the Decision card, Agent cards, and Workflow steps.
7.  **Response Delivery:**
    *   Connect a **Respond to Webhook** node.
    *   Set **Respond With** to `Text`.
    *   Set **Response Body** to the `html` property from the previous node using an expression.
8.  **Activation:**
    *   Save and activate the workflow. Use a tool like Postman or cURL to send a POST request with a JSON body: `{"description": "I want to classify incoming support emails and route them to different departments."}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Model Optimization** | Replace `gpt-4o-mini` with `gpt-4o` for higher reasoning accuracy on complex architecture problems. |
| **Customization** | Update CSS styles in the "Build HTML Architecture Report" node to match internal company branding. |
| **Webhook Response** | Ensure the Webhook node is specifically set to "Response Mode: responseNode" or "When Last Node Finishes" for the HTML to render in the browser. |