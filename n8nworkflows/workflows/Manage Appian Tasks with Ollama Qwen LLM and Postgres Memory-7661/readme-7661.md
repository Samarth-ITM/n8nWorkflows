Manage Appian Tasks with Ollama Qwen LLM and Postgres Memory

https://n8nworkflows.xyz/workflows/manage-appian-tasks-with-ollama-qwen-llm-and-postgres-memory-7661


# Manage Appian Tasks with Ollama Qwen LLM and Postgres Memory

### 1. Workflow Overview

This n8n workflow implements a conversational AI agent that manages Appian tasks via an Ollama-hosted Qwen large language model (LLM) and maintains conversation state in a Postgres memory. It is designed for use cases where users interact by chat or webhook to list, create, and query Appian tasks through an API compatible with Appian’s task management system.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Receives chat messages either via a chat trigger or webhook, then normalizes incoming data.
- **1.2 AI Processing:** Uses the Ollama Qwen LLM as the core language model, enhanced with memory storage in Postgres to maintain conversational context, and processes the input through an AI agent.
- **1.3 Appian Task API Integration:** Provides tools for listing tasks, task types, and creating new tasks by interacting with the Appian API.
- **1.4 Response Preparation and Delivery:** Structures the AI’s output and responds back to the webhook caller.
- **1.5 Configuration and Notes:** Contains template variables for API base URL and batch size, plus explanatory notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures incoming chat messages from two types of sources—a chat message trigger or a generic HTTP webhook—and normalizes the input data into consistent fields (`chatInput`, `sessionId`, `username`) for downstream processing.

**Nodes Involved:**  
- When chat message received  
- Webhook  
- Normalize Chat Input  

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Initiates workflow on new chat messages from a chat integration.  
  - Configuration: Default options; listens for incoming chat.  
  - Inputs: External chat service  
  - Outputs: Normalized chat input node  
  - Edge cases: Missing or malformed chat data, connection issues with chat provider.

- **Webhook**  
  - Type: HTTP Webhook (POST)  
  - Role: Receives external POST requests at path `/your-webhook-path` with header authentication.  
  - Configuration:  
    - HTTP method: POST  
    - Auth: Header-based authentication  
    - Response mode: Waits to respond with data from "Respond to Webhook" node.  
  - Inputs: External HTTP clients  
  - Outputs: Normalize Chat Input  
  - Edge cases: Authentication failures, invalid payloads, timeouts.

- **Normalize Chat Input**  
  - Type: Set node  
  - Role: Extracts and normalizes input fields from various potential payload formats including `body.text`, `chatInput`, or session/conversation IDs.  
  - Configuration:  
    - Sets variables:  
      - `chatInput`: prioritizes message text from different JSON paths  
      - `sessionId`: conversation/session identifier from multiple fields  
      - `username`: user identifier with default fallback `user@example.com`  
  - Inputs: Chat Trigger or Webhook  
  - Outputs: AI Agent  
  - Edge cases: Missing fields causing empty variables, inconsistent input formats.

---

#### 2.2 AI Processing

**Overview:**  
The core AI logic uses the Ollama Qwen LLM to process user inputs with contextual memory stored in Postgres. The AI agent orchestrates the conversation, invoking specific tools for task management.

**Nodes Involved:**  
- Ollama Chat Model  
- Postgres Chat Memory  
- AI Agent  

**Node Details:**

- **Ollama Chat Model**  
  - Type: Langchain Ollama LLM Chat node  
  - Role: Provides the large language model backend using Ollama Qwen 2.5 (7b parameter model).  
  - Configuration:  
    - Model: `qwen2.5:7b`  
    - Context length: 8147 tokens (large context window)  
  - Inputs: AI Agent (as languageModel)  
  - Outputs: AI Agent  
  - Edge cases: Model unavailability or slow response, token limit exceeded, model errors.

- **Postgres Chat Memory**  
  - Type: Langchain Postgres Chat Memory  
  - Role: Maintains conversational history in a Postgres database to provide context across interactions.  
  - Configuration: Default (assumes external credential setup)  
  - Inputs: AI Agent (as memory)  
  - Outputs: AI Agent  
  - Edge cases: Database connectivity issues, data synchronization delays.

- **AI Agent**  
  - Type: Langchain Agent node  
  - Role: Coordinates the workflow logic; uses language model, memory, and tools to interpret chat inputs and produce structured outputs.  
  - Configuration:  
    - Max iterations: 15 (limits reasoning cycles)  
    - System message: Defines assistant behavior as concise and date formatting rule.  
    - Returns intermediate steps for transparency/debugging.  
  - Inputs: Normalized chat input, Ollama Model (languageModel), Postgres Memory (memory), and tools (task APIs)  
  - Outputs: Prepare Response  
  - Edge cases: Iteration limit reached without resolution, tool invocation failures, expression errors.

---

#### 2.3 Appian Task API Integration

**Overview:**  
This block implements tools that the AI agent can invoke to interact programmatically with the Appian API to list tasks, list task types, and create tasks. These HTTP request tools are authenticated via OAuth2.

**Nodes Involved:**  
- List Tasks (Appian)  
- List Task Types (Appian)  
- Create Task (Appian)  

**Node Details:**

- **List Tasks (Appian)**  
  - Type: HTTP Request Tool  
  - Role: Retrieves a paginated list of tasks assigned to a user.  
  - Configuration:  
    - URL: `{baseUrl}/tasks` (from Template Vars)  
    - Method: GET (default)  
    - Query parameters:  
      - `username` from normalized input  
      - `startIndex` from AI parameters or defaults to 1  
      - `batchSize` from Template Vars (default 10)  
    - Authentication: OAuth2 (genericCredentialType)  
  - Inputs: AI Agent tool invocation  
  - Outputs: AI Agent  
  - Edge cases: Authentication failures, API errors, pagination errors.

- **List Task Types (Appian)**  
  - Type: HTTP Request Tool  
  - Role: Lists all available task types from Appian.  
  - Configuration:  
    - URL: `{baseUrl}/tasktypes`  
    - Method: GET  
    - Authentication: OAuth2  
  - Inputs: AI Agent tool  
  - Outputs: AI Agent  
  - Edge cases: Authentication errors, API unavailability.

- **Create Task (Appian)**  
  - Type: HTTP Request Tool  
  - Role: Creates a new task for a user with provided title, description, and optional estimates.  
  - Configuration:  
    - URL: `{baseUrl}/tasks`  
    - Method: POST  
    - Body parameters:  
      - `username` from normalized input  
      - `title`, `description`, `estimatedHours`, `estimatedCost` extracted from AI parameters (user input parsed by AI agent)  
    - Authentication: OAuth2  
  - Inputs: AI Agent tool  
  - Outputs: AI Agent  
  - Edge cases: Duplicate task creation (should be executed once per task), validation errors, auth failures.

---

#### 2.4 Response Preparation and Delivery

**Overview:**  
This block formats the AI agent’s output into a JSON response and sends it back to the webhook caller.

**Nodes Involved:**  
- Prepare Response  
- Respond to Webhook  

**Node Details:**

- **Prepare Response**  
  - Type: Set node  
  - Role: Constructs the response JSON payload using the AI agent’s output and includes session ID for continuity.  
  - Configuration:  
    - Sets `output` from AI agent’s output field  
    - Sets `sessionId` from normalized input sessionId  
  - Inputs: AI Agent output  
  - Outputs: Respond to Webhook  
  - Edge cases: Missing output causing empty response.

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends the prepared JSON response back to the HTTP request originator (webhook client).  
  - Configuration: Default response mode, no special headers.  
  - Inputs: Prepared response  
  - Outputs: None (terminal node)  
  - Edge cases: Client disconnects, response timeout.

---

#### 2.5 Configuration and Notes

**Overview:**  
Holds user-configurable variables and informational notes to guide setup and usage.

**Nodes Involved:**  
- Template Vars  
- Notes  

**Node Details:**

- **Template Vars**  
  - Type: Set node  
  - Role: Defines reusable variables such as Appian base URL and default pagination batch size.  
  - Configuration:  
    - `baseUrl`: Placeholder for Appian API base URL (e.g., `https://YOUR-APP-BASE/suite/webapi`)  
    - `defaultBatchSize`: Default number of tasks per page (default 10)  
  - Inputs: None (start node)  
  - Outputs: Used by HTTP request nodes  
  - Edge cases: Missing or incorrect URL will cause API failures.

- **Notes**  
  - Type: Sticky Note  
  - Role: Provides descriptive text about the template and setup instructions.  
  - Content: "Appian Tasks Chat Agent\nThis template shows a local AI chat agent that lists and creates tasks via an Appian-compatible API. Configure Template Vars and attach your own credentials after import."  
  - Inputs/Outputs: None

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                          |
|-----------------------|----------------------------------|------------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Template Vars         | Set                              | Configuration variables             | None                     | List Tasks, List Types, Create Task |                                                                                                    |
| When chat message received | Langchain Chat Trigger          | Receive chat messages               | External chat service     | Normalize Chat Input    |                                                                                                    |
| Webhook               | HTTP Webhook                     | Receive webhook HTTP POST           | External client           | Normalize Chat Input    |                                                                                                    |
| Normalize Chat Input  | Set                              | Normalize incoming input            | When chat message received, Webhook | AI Agent                |                                                                                                    |
| Ollama Chat Model     | Langchain Ollama LLM Chat        | Core LLM for AI processing          | AI Agent (languageModel)  | AI Agent                |                                                                                                    |
| Postgres Chat Memory  | Langchain Postgres Memory        | Maintains conversation context     | AI Agent (memory)         | AI Agent                |                                                                                                    |
| AI Agent              | Langchain Agent                  | Orchestrates AI logic and tools    | Normalize Chat Input, Ollama Chat Model, Postgres Chat Memory, Tools | Prepare Response        |                                                                                                    |
| List Tasks (Appian)   | HTTP Request Tool                | Lists user tasks                   | AI Agent (tool)           | AI Agent                |                                                                                                    |
| List Task Types (Appian) | HTTP Request Tool                | Lists available task types          | AI Agent (tool)           | AI Agent                |                                                                                                    |
| Create Task (Appian)  | HTTP Request Tool                | Creates new task                   | AI Agent (tool)           | AI Agent                |                                                                                                    |
| Prepare Response      | Set                              | Formats AI output for response      | AI Agent                  | Respond to Webhook      |                                                                                                    |
| Respond to Webhook    | Respond to Webhook               | Sends HTTP response                 | Prepare Response          | None                    |                                                                                                    |
| Notes                 | Sticky Note                     | Workflow description & setup info  | None                     | None                    | Appian Tasks Chat Agent template; configure Template Vars and credentials after import.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Template Vars" node (Set type):**  
   - Add variables:  
     - `baseUrl` (string) = `"https://YOUR-APP-BASE/suite/webapi"`  
     - `defaultBatchSize` (number) = `10`

2. **Create "When chat message received" node (Langchain Chat Trigger):**  
   - Use default settings to listen for incoming chat messages.

3. **Create "Webhook" node (HTTP Webhook):**  
   - HTTP Method: POST  
   - Path: `your-webhook-path`  
   - Authentication: Header Auth (configure credentials as needed)  
   - Response Mode: `responseNode`

4. **Create "Normalize Chat Input" node (Set):**  
   - Assign these variables using expressions:  
     - `chatInput` = `{{$json?.body?.text || $json?.chatInput || $json.body?.chatInput}}`  
     - `sessionId` = `{{$json?.body?.conversation?.id || $json?.sessionId || $json.body?.sessionId}}`  
     - `username` = `{{$json?.body?.user || $json?.username || 'user@example.com'}}`  
   - Connect "When chat message received" and "Webhook" nodes to this node.

5. **Create "Ollama Chat Model" node (Langchain Ollama LLM Chat):**  
   - Model: `qwen2.5:7b`  
   - Options: `numCtx = 8147` (context tokens)

6. **Create "Postgres Chat Memory" node (Langchain Postgres Chat Memory):**  
   - Configure Postgres credentials externally  
   - Use default settings for chat memory

7. **Create "AI Agent" node (Langchain Agent):**  
   - Max Iterations: 15  
   - System Message:  
     ```
     You are a helpful assistant that can list and create Appian tasks using the provided tools. Keep responses concise and format dates as 'mmmm d, yyyy'.
     ```  
   - Return Intermediate Steps: true  
   - Connect inputs:  
     - `main` from "Normalize Chat Input"  
     - `ai_languageModel` from "Ollama Chat Model"  
     - `ai_memory` from "Postgres Chat Memory"  
     - `ai_tool` from each Appian API tool (see next steps)

8. **Create "List Tasks (Appian)" node (HTTP Request Tool):**  
   - URL: `{{$('Template Vars').item.json.baseUrl}}/tasks`  
   - Authentication: OAuth2 (genericCredentialType)  
   - Query Parameters:  
     - `username` = `{{$('Normalize Chat Input').item.json.username}}`  
     - `startIndex` = `{{parseInt($fromAI('parameters1_Value', 'First result index', 'string')) || 1}}`  
     - `batchSize` = `{{$('Template Vars').item.json.defaultBatchSize || 10}}`  
   - Connect output to AI Agent `ai_tool`

9. **Create "List Task Types (Appian)" node (HTTP Request Tool):**  
   - URL: `{{$('Template Vars').item.json.baseUrl}}/tasktypes`  
   - Authentication: OAuth2  
   - Connect output to AI Agent `ai_tool`

10. **Create "Create Task (Appian)" node (HTTP Request Tool):**  
    - URL: `{{$('Template Vars').item.json.baseUrl}}/tasks`  
    - Method: POST  
    - Authentication: OAuth2  
    - Body Parameters:  
      - `username` = `{{$('Normalize Chat Input').item.json.username}}`  
      - `title` = `{{$fromAI('parameters1_Value', 'Short task title', 'string')}}`  
      - `description` = `{{$fromAI('parameters2_Value', 'Detailed task description', 'string')}}`  
      - `estimatedHours` = `{{$fromAI('parameters3_Value', 'Estimated hours (optional)', 'string')}}`  
      - `estimatedCost` = `{{$fromAI('parameters4_Value', 'Estimated cost (optional)', 'string')}}`  
    - Connect output to AI Agent `ai_tool`

11. **Create "Prepare Response" node (Set):**  
    - Assign:  
      - `output` = `{{$json.output}}` (from AI Agent output)  
      - `sessionId` = `{{$('Normalize Chat Input').item.json.sessionId}}`  
    - Connect AI Agent main output to this node.

12. **Create "Respond to Webhook" node:**  
    - Connect output from "Prepare Response" node.  
    - Use default settings to send response back to webhook caller.

13. **Create "Notes" node (Sticky Note):**  
    - Content:  
      ```
      Appian Tasks Chat Agent
      This template shows a local AI chat agent that lists and creates tasks via an Appian-compatible API. Configure Template Vars and attach your own credentials after import.
      ```

14. **Connect all nodes accordingly:**  
    - "When chat message received" → "Normalize Chat Input"  
    - "Webhook" → "Normalize Chat Input"  
    - "Normalize Chat Input" → "AI Agent" (main)  
    - "Ollama Chat Model" → "AI Agent" (ai_languageModel)  
    - "Postgres Chat Memory" → "AI Agent" (ai_memory)  
    - "List Tasks (Appian)", "List Task Types (Appian)", "Create Task (Appian)" → "AI Agent" (ai_tool)  
    - "AI Agent" → "Prepare Response"  
    - "Prepare Response" → "Respond to Webhook"

15. **Configure Credentials:**  
    - OAuth2 credentials for Appian API on HTTP Request nodes  
    - Header authentication credentials for Webhook node  
    - Postgres credentials for memory node  
    - Ollama LLM API access if required

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow demonstrates integrating a local AI chat agent with Appian-compatible task APIs.          | Workflow description and design rationale                                                           |
| Configure Template Vars to match your Appian API base URL and batch size preferences.                    | Template Vars node                                                                                   |
| Attach your OAuth2 credentials for Appian API access and Header Auth for webhook security.              | Credential setup instructions                                                                        |
| Postgres memory requires a configured Postgres connection with appropriate tables for Langchain memory. | Database setup for conversation persistence                                                         |
| Ollama Qwen 2.5 LLM provides a local large language model with extensive context window (8147 tokens).  | Ollama LLM node                                                                                      |
| The AI agent system message instructs concise responses with date formatting as “mmmm d, yyyy.”         | AI Agent node configuration                                                                          |
| The workflow includes both chat trigger and webhook input methods for flexible integration.              | Input Reception block                                                                                |
| Reference: [https://docs.appian.com](https://docs.appian.com) for Appian API details and OAuth2 setup.  | External documentation                                                                               |

---

This document provides a detailed, structured reference for understanding, reproducing, and extending the "Manage Appian Tasks with Ollama Qwen LLM and Postgres Memory" n8n workflow. It facilitates troubleshooting, credential configuration, and future enhancements.