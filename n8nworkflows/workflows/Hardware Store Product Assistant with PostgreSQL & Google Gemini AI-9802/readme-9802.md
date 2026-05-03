Hardware Store Product Assistant with PostgreSQL & Google Gemini AI

https://n8nworkflows.xyz/workflows/hardware-store-product-assistant-with-postgresql---google-gemini-ai-9802


# Hardware Store Product Assistant with PostgreSQL & Google Gemini AI

### 1. Workflow Overview

This workflow implements a conversational AI agent designed for a hardware store environment, specifically specialized in drywall, ceiling systems, and finishing materials. Its primary purpose is to assist customers with detailed product queries, provide project advice, and generate precise quotations by leveraging a PostgreSQL product database and a Google Gemini AI language model.

The workflow logic is structured into the following functional blocks:

- **1.1 Input Reception and Triggering:** Handles incoming chat messages through a web-accessible chat trigger.
- **1.2 AI Agent Processing:** Processes the conversation using a custom-configured AI agent with business logic, integrating chat memory and invoking an external AI language model (Google Gemini).
- **1.3 Database Query Routing:** Based on AI agent tool requests, routes queries to specific PostgreSQL database nodes targeting product information by various search criteria.
- **1.4 Database Query Execution:** Executes targeted PostgreSQL queries to fetch product data in real-time.
- **1.5 AI Tools and Memory Integration:** Maintains conversational context and routes data between the AI agent and the database via MCP Client tooling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:**  
  This block captures user messages from an external chat interface via a webhook, enabling real-time conversational input.

- **Nodes Involved:**  
  - Chat Trigger  
  - Database Tools Trigger

- **Node Details:**

  - **Chat Trigger**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.chatTrigger` — Webhook-based chat input node that listens for incoming messages.  
    - *Configuration:* Publicly accessible webhook with file upload allowed and a welcome screen enabled.  
    - *Key Parameters:* `webhookId`, `public: true`, `allowFileUploads: true`.  
    - *Connections:* Output to AI Agent node.  
    - *Edge Cases:* Webhook downtime or invalid input formats could cause failure.  
    - *Version:* 1.3  

  - **Database Tools Trigger**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.mcpTrigger` — An MCP Server trigger that listens for AI tool requests to query the database.  
    - *Configuration:* Path set to `servidorbasesdedatos`.  
    - *Connections:* Outputs to multiple PostgreSQL query nodes.  
    - *Edge Cases:* Server availability and path correctness critical; network failures possible.  
    - *Version:* 2  

#### 2.2 AI Agent Processing

- **Overview:**  
  This block processes the chat messages with an AI agent configured to serve as a sales assistant specialized in drywall and construction materials. It uses contextual chat memory and a Google Gemini language model to understand user intents and generate responses, while interacting with the database via MCP Client tooling.

- **Nodes Involved:**  
  - AI Agent  
  - Chat Memory  
  - Language Model (Google Gemini)  
  - DB Tools Client

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` — Central AI agent node orchestrating conversation flow, tool usage, and response generation.  
    - *Configuration:*  
      - System message sets a detailed role as a construction materials sales expert with access to PostgreSQL product data.  
      - Defines extensive product schema fields and conversational guidelines.  
      - Supports queries by multiple product attributes (ID, name, category, etc.) and provides project advice and quotations.  
    - *Key Expressions:* Uses AI tool input to trigger database queries dynamically.  
    - *Connections:* Receives input from Chat Trigger, outputs to Chat Memory and DB Tools Client, consumes Language Model.  
    - *Edge Cases:* Misinterpretation of user input; failure to generate meaningful queries if data is missing; dependency on downstream nodes' availability.  
    - *Version:* 2.2  

  - **Chat Memory**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` — Maintains conversational context with a sliding window of 50 messages.  
    - *Configuration:* Context window length set to 50 messages to keep relevant history.  
    - *Connections:* Input from AI Agent output; feeds back into AI Agent.  
    - *Edge Cases:* Memory overflow or loss could disrupt context continuity.  
    - *Version:* 1.3  

  - **Language Model (Google Gemini)**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — Integrates Google Gemini AI as the language model backend for generating natural language responses.  
    - *Configuration:* Uses Google Gemini credentials configured with OAuth2 or API key.  
    - *Connections:* Used by AI Agent as the language model component.  
    - *Edge Cases:* API rate limits, credential expiration, or network issues.  
    - *Version:* 1  

  - **DB Tools Client**  
    - *Type & Role:* `@n8n/n8n-nodes-langchain.mcpClientTool` — Client tool that connects to the MCP Server endpoint to perform database queries as requested by the AI agent.  
    - *Configuration:* Endpoint URL set to `http://localhost:5678/mcp/servidorbasesdedatos`.  
    - *Connections:* Connected as the AI tool for the AI Agent node.  
    - *Edge Cases:* MCP Server downtime, connectivity issues, endpoint misconfiguration.  
    - *Version:* 1.1  

#### 2.3 Database Query Routing and Execution

- **Overview:**  
  This block contains multiple PostgreSQL query nodes. Each node targets product data filtered by a specific attribute such as ID, name, description, category, subcategory, or note. The nodes execute SQL SELECT operations to fetch matching product records from the `productos` table in the `public` schema.

- **Nodes Involved:**  
  - Query Product by ID  
  - Query Product by Name  
  - Query Product by Description  
  - Query Product by Category  
  - Query Product by Subcategory  
  - Query Product by Note

- **Node Details:**

  Each node shares similar configuration patterns with differences in the `where` clause column:

  - *Type & Role:* `n8n-nodes-base.postgresTool` — Executes parameterized SQL SELECT queries on PostgreSQL.  
  - *Configuration:*  
    - Table: `productos` in schema `public`.  
    - Operation: `select`.  
    - Where clause uses dynamic expressions to get the search value from the AI tool input variable `values0_Value`.  
    - Combine conditions with `OR` (though typically only one condition is present).  
  - *Credentials:* Uses PostgreSQL credentials linked to `bd_ferreteria`.  
  - *Connections:*  
    - Input from Database Tools Trigger node (MCP trigger).  
    - Outputs back to MCP server for AI agent consumption.  
  - *Edge Cases:*  
    - Empty or malformed query values.  
    - Database connection failures.  
    - No results found (empty response).  
    - Potential SQL injection mitigated by parameterization.  
  - *Version:* 2.6  

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                        | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                     |
|-------------------------|--------------------------------------------|-------------------------------------|--------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Database Tools Trigger   | @n8n/n8n-nodes-langchain.mcpTrigger        | MCP Server trigger for AI tool calls | —                        | Query Product nodes      |                                                                                                                                |
| Query Product by ID      | n8n-nodes-base.postgresTool                 | Query products by unique ID          | Database Tools Trigger    | MCP Server               |                                                                                                                                |
| Query Product by Name    | n8n-nodes-base.postgresTool                 | Query products by name               | Database Tools Trigger    | MCP Server               |                                                                                                                                |
| Query Product by Description | n8n-nodes-base.postgresTool             | Query products by description        | Database Tools Trigger    | MCP Server               |                                                                                                                                |
| Query Product by Category| n8n-nodes-base.postgresTool                 | Query products by category           | Database Tools Trigger    | MCP Server               |                                                                                                                                |
| Query Product by Subcategory | n8n-nodes-base.postgresTool              | Query products by subcategory        | Database Tools Trigger    | MCP Server               |                                                                                                                                |
| Query Product by Note    | n8n-nodes-base.postgresTool                 | Query products by note               | Database Tools Trigger    | MCP Server               |                                                                                                                                |
| Chat Trigger            | @n8n/n8n-nodes-langchain.chatTrigger        | Receives chat messages via webhook  | —                        | AI Agent                 |                                                                                                                                |
| AI Agent                | @n8n/n8n-nodes-langchain.agent              | Conversational AI processing & tools | Chat Trigger             | Chat Memory, DB Tools Client |                                                                                                                                |
| Language Model (Google Gemini) | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides AI language model responses | AI Agent                 | AI Agent                 |                                                                                                                                |
| Chat Memory             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context      | AI Agent                  | AI Agent                 |                                                                                                                                |
| DB Tools Client         | @n8n/n8n-nodes-langchain.mcpClientTool      | Calls MCP Server for DB queries     | AI Agent                  | MCP Server               |                                                                                                                                |
| Sticky Note             | n8n-nodes-base.stickyNote                    | Workflow description note            | —                        | —                       | ## 📝 Descripción del Flujo / Workflow Description... (content as per node)                                                    |
| Sticky Note1            | n8n-nodes-base.stickyNote                    | Required configuration note          | —                        | —                       | ## ⚙️ Configuración Requerida / Required Configuration... (content as per node)                                                 |
| Sticky Note2            | n8n-nodes-base.stickyNote                    | Workflow summary note                | —                        | —                       | ## AI Agent with PostgreSQL & MCP for Hardware Store Chatbot... (content as per node)                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configure webhook: set `public` to `true`, enable file uploads, and enable welcome screen.  
   - Save webhook ID for external access.  

2. **Create the Database Tools Trigger node:**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set `path` parameter to `servidorbasesdedatos`.  
   - This node will listen for AI tool requests.  

3. **Create PostgreSQL query nodes (6 total):**  
   For each node, configure:  
   - Type: `n8n-nodes-base.postgresTool`  
   - Credentials: Select or create PostgreSQL credentials named `bd_ferreteria`.  
   - Operation: `select`  
   - Table: `productos` in schema `public`  
   - Where clause column differs per node:  
     - Query Product by ID → column `id`  
     - Query Product by Name → column `nombre`  
     - Query Product by Description → column `descripcion`  
     - Query Product by Category → column `categoria`  
     - Query Product by Subcategory → column `subcategoria`  
     - Query Product by Note → column `nota`  
   - Where clause value: Use expression `{{$fromAI('values0_Value', '', 'string')}}` to accept dynamic inputs.  
   - Combine conditions with `OR`.  
   - Connect each node’s input from Database Tools Trigger node’s `ai_tool` output.  

4. **Create the AI Agent node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Paste the provided detailed system message defining the AI agent’s role, product schema, and conversational logic.  
   - Connect input from Chat Trigger node.  
   - Configure AI tools: add `DB Tools Client` node as an AI tool.  
   - Configure AI language model: select `Language Model (Google Gemini)` node.  

5. **Create the Language Model (Google Gemini) node:**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Configure with valid Google Gemini (PaLM) API credentials.  
   - Connect output to AI Agent node’s AI language model input.  

6. **Create the Chat Memory node:**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set `contextWindowLength` to 50 messages.  
   - Connect output from AI Agent node and feed back into AI Agent node input to maintain context.  

7. **Create the DB Tools Client node:**  
   - Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
   - Set `endpointUrl` to `http://localhost:5678/mcp/servidorbasesdedatos`.  
   - Connect as AI tool input for the AI Agent node.  

8. **Connect nodes:**  
   - Chat Trigger → AI Agent → Chat Memory (bidirectional)  
   - AI Agent → DB Tools Client → MCP Server (Database Tools Trigger) → PostgreSQL query nodes  
   - PostgreSQL query nodes respond back to MCP Server → DB Tools Client → AI Agent  

9. **Add Sticky Notes (optional):**  
   - Add descriptive notes for workflow overview, configuration requirements, and summary as per original content for documentation clarity.  

10. **Activate workflow and test:**  
    - Ensure all credentials are correctly configured.  
    - Deploy MCP Server endpoint at the specified URL.  
    - Test the chat interface and verify product queries return expected results.  

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This AI agent provides specialized sales assistance for drywall and finishing materials, including quotes and project advice. | Workflow purpose and scope description.                                                                  |
| Credentials for PostgreSQL and Google Gemini must be properly configured for workflow operation.            | See Sticky Note1 in the workflow for detailed setup instructions.                                        |
| MCP (Modular Chat Plugin) tooling is critical for bridging AI agent tool calls and PostgreSQL database.    | MCP Client and MCP Trigger nodes implement this integration pattern.                                     |
| Product database schema includes fields like `id`, `nombre`, `descripcion`, `categoria`, `subcategoria`, etc. | Enables flexible product queries by multiple attributes.                                                 |
| Google Gemini API usage requires valid API credentials and proper quota management to avoid rate limits.    | Credential configuration in Language Model node.                                                         |
| The agent’s system message includes detailed conversational guidelines that ensure professional, clear, and proactive customer interactions. | Helps maintain consistent tone and quality of responses.                                                 |
| The workflow respects all data privacy and legality requirements; only public, legal data is processed.     | Disclaimer: workflow content complies with all applicable content policies.                               |

---

**Disclaimer:** This document is generated exclusively from an n8n workflow automation JSON. It strictly adheres to content policies and contains no illegal or offensive elements. All data handled is legal and publicly accessible.