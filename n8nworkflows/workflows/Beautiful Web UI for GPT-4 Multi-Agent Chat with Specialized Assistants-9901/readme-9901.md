Beautiful Web UI for GPT-4 Multi-Agent Chat with Specialized Assistants

https://n8nworkflows.xyz/workflows/beautiful-web-ui-for-gpt-4-multi-agent-chat-with-specialized-assistants-9901


# Beautiful Web UI for GPT-4 Multi-Agent Chat with Specialized Assistants

---

## 1. Workflow Overview

This n8n workflow provides a **Beautiful Web UI for GPT-4 Multi-Agent Chat with Specialized Assistants**. It enables end-users to interact with multiple AI agents via a clean, user-friendly web interface served by n8n, without requiring frontend coding skills. Users can send messages either generally or routed to one of three specialized AI agents — Database Search, Web Search, or RAG Knowledge Agent — each designed for specific types of queries.

The workflow consists of two main logical blocks:

- **1.1 UI Interface Delivery Block:**  
  Handles HTTP GET requests to serve a fully styled HTML/JavaScript frontend that users interact with. This interface includes message input, agent selection, dark/light mode toggle, loading states, and response display.

- **1.2 AI Agent Processing Block:**  
  Handles HTTP POST requests containing user messages and agent type, routes requests to the appropriate AI agent node (general, database, web, or RAG), gathers AI responses, formats them, and returns JSON responses to the frontend.

---

## 2. Block-by-Block Analysis

### 2.1 UI Interface Delivery Block

**Overview:**  
This block serves the HTML web UI for user interaction. It listens for GET requests and responds with a complete HTML page embedding CSS and JS to render a chat interface with specialized AI agents.

**Nodes Involved:**  
- Webhook (GET)  
- Code in JavaScript  
- Respond to Webhook

**Node Details:**

- **Webhook (GET)**  
  - *Type & Role:* HTTP webhook node, entry point for UI requests.  
  - *Configuration:* HTTP method GET; path uniquely set (e.g., `/b6f698e9-c16c-4273-8af2-20a958f691c1`).  
  - *Input/Output:* Receives HTTP GET, outputs to Code node.  
  - *Edge Cases:* Invalid/malformed requests or path mismatch result in no UI served.

- **Code in JavaScript**  
  - *Type & Role:* Code node generates the entire HTML page with embedded CSS/JS.  
  - *Configuration:* Contains a large JavaScript string that builds a styled, responsive interface with:  
    - Header including project title and theme toggle  
    - Text input area for messages  
    - Buttons for sending messages and selecting specialized agents  
    - Response display area with loading and error states  
    - Modal with project guide and instructions  
    - JavaScript functions: sending messages, agent selection, response display, error handling, theme toggling, modal controls.  
  - *Key Expressions:* Uses template strings for dynamic content insertion (project name and version).  
  - *Edge Cases:* If generating HTML fails, UI won't render. The embedded JS expects a configurable webhook URL which must be customized before deployment.  
  - *Credentials:* None.  
  - *Sub-workflow:* None.

- **Respond to Webhook**  
  - *Type & Role:* Sends the generated HTML as a binary HTTP response.  
  - *Configuration:* Respond with binary data; property named `data`.  
  - *Input/Output:* Receives HTML buffer from Code node, sends HTTP response to client.  
  - *Edge Cases:* Failure to respond properly may cause client timeouts or broken UI.

---

### 2.2 AI Agent Processing Block

**Overview:**  
Handles POST requests from the frontend containing user messages and chosen agent type. Routes to the correct AI agent workflow, processes the message through GPT-4 powered Langchain agents, formats the AI response, and returns JSON to the frontend.

**Nodes Involved:**  
- Webhook1 (POST)  
- Switch  
- AI Agent - General  
- AI Agent Database  
- AI Agent - Web  
- AI Agent - Rag  
- OpenAI Chat Model (4 instances)  
- Format Response - Code (4 instances)  
- Respond to Webhook1, Respond to Webhook2, Respond to Webhook3, Respond to Webhook4

**Node Details:**

- **Webhook1 (POST)**  
  - *Type & Role:* HTTP webhook node receiving chat POST requests.  
  - *Configuration:* HTTP method POST; path `/webhook-endpoint`. Uses responseMode `responseNode` for async response.  
  - *Input/Output:* Receives JSON body with `message`, `agent_type`, and timestamp; outputs to Switch node.  
  - *Edge Cases:* Invalid requests or missing fields may cause downstream errors.

- **Switch**  
  - *Type & Role:* Routes messages based on `agent_type` field in POST JSON body.  
  - *Configuration:* Four conditions matching `agent_type` values: `general`, `database`, `web`, `rag`.  
  - *Input/Output:* Receives webhook JSON; outputs to respective AI Agent node.  
  - *Edge Cases:* Unknown `agent_type` values are not handled explicitly, may cause dropped requests.

- **AI Agent - General / Database / Web / Rag**  
  - *Type & Role:* Langchain AI Agent nodes interfacing with OpenAI GPT-4 model for processing user messages.  
  - *Configuration:* Each has a `text` parameter (placeholder like "Configure Database Agent") and prompt type `define`.  
  - *Input:* Receives user message JSON from Switch node.  
  - *Output:* AI-generated response JSON.  
  - *Credentials:* Uses shared OpenAI API credentials (`OpenAi account`) with GPT-4.1-mini model configured.  
  - *Edge Cases:* Possible OpenAI API auth failures, rate limits, or timeouts; fallback/error handling not shown here.

- **OpenAI Chat Model (4 nodes)**  
  - *Type & Role:* Language model nodes providing GPT-4 responses for each AI Agent node via Langchain integration.  
  - *Configuration:* Model set to `gpt-4.1-mini`, no additional options.  
  - *Input/Output:* Connected as language model backend for each AI Agent node.  
  - *Credentials:* OpenAI API credentials required and configured.  

- **Format Response - Code (4 nodes)**  
  - *Type & Role:* Code nodes to normalize AI agent output and prepare standardized JSON response.  
  - *Configuration:* Extracts AI response text from multiple possible keys (`output`, `text`, `response`, `message`), applies fallback JSON stringify. Includes original user message, agent type, and timestamp.  
  - *Input/Output:* Input from AI Agent nodes; output to Respond to Webhook nodes.  
  - *Edge Cases:* Handles diverse AI output formats gracefully.

- **Respond to Webhook1, ... Respond to Webhook4**  
  - *Type & Role:* Final nodes to respond to the original HTTP POST request with formatted JSON containing AI response and metadata.  
  - *Configuration:* Default JSON response mode.  
  - *Input/Output:* Input from Format Response code nodes; output HTTP response to frontend.  
  - *Edge Cases:* Failure to respond leads to frontend request failures.

---

## 3. Summary Table

| Node Name              | Node Type                              | Functional Role                      | Input Node(s)    | Output Node(s)           | Sticky Note                                                                                                                  |
|------------------------|--------------------------------------|------------------------------------|------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Webhook                | n8n-nodes-base.webhook                | Serve GET HTTP UI interface         | -                | Code in JavaScript        | ## Module - Create UI interface                                                                                              |
| Code in JavaScript     | n8n-nodes-base.code                   | Generate HTML/JS/CSS UI              | Webhook          | Respond to Webhook        | ## Module - Create UI interface                                                                                              |
| Respond to Webhook     | n8n-nodes-base.respondToWebhook       | Return binary HTML response          | Code in JavaScript| -                        | ## Module - Create UI interface                                                                                              |
| Sticky Note            | n8n-nodes-base.stickyNote             | Annotation (UI Interface module)    | -                | -                        | # 🎨 n8n AI Agent Interface Template... (detailed setup and features explained)                                              |
| Sticky Note1           | n8n-nodes-base.stickyNote             | Annotation (Setup instructions)     | -                | -                        | # 🎨 n8n AI Agent Interface Template... (quick setup and agent details)                                                      |
| Sticky Note2           | n8n-nodes-base.stickyNote             | Annotation (AI Agent module)         | -                | -                        | ## Module - AI Agent                                                                                                         |
| Webhook1               | n8n-nodes-base.webhook                | Receive POST chat requests           | -                | Switch                   |                                                                                                                              |
| Switch                 | n8n-nodes-base.switch                 | Route by agent_type                  | Webhook1         | AI Agent - General, AI Agent Database, AI Agent - Web, AI Agent - Rag |                                                                                                                              |
| AI Agent - General     | @n8n/n8n-nodes-langchain.agent       | General AI agent processing          | Switch           | Format Response - Code    |                                                                                                                              |
| AI Agent Database      | @n8n/n8n-nodes-langchain.agent       | Database specialized AI agent        | Switch           | Format Response - Code1   |                                                                                                                              |
| AI Agent - Web         | @n8n/n8n-nodes-langchain.agent       | Web search specialized AI agent      | Switch           | Format Response - Code2   |                                                                                                                              |
| AI Agent - Rag         | @n8n/n8n-nodes-langchain.agent       | RAG knowledge base specialized agent| Switch           | Format Response - Code3   |                                                                                                                              |
| OpenAI Chat Model      | @n8n/n8n-nodes-langchain.lmChatOpenAi| GPT-4 model backend for AI agents   | -                | AI Agent - General        |                                                                                                                              |
| OpenAI Chat Model1     | @n8n/n8n-nodes-langchain.lmChatOpenAi| GPT-4 model backend for AI agents   | -                | AI Agent Database         |                                                                                                                              |
| OpenAI Chat Model2     | @n8n/n8n-nodes-langchain.lmChatOpenAi| GPT-4 model backend for AI agents   | -                | AI Agent - Web            |                                                                                                                              |
| OpenAI Chat Model3     | @n8n/n8n-nodes-langchain.lmChatOpenAi| GPT-4 model backend for AI agents   | -                | AI Agent - Rag            |                                                                                                                              |
| Format Response - Code | n8n-nodes-base.code                   | Normalize AI response for General   | AI Agent - General| Respond to Webhook1       |                                                                                                                              |
| Format Response - Code1| n8n-nodes-base.code                   | Normalize AI response for Database  | AI Agent Database| Respond to Webhook2       |                                                                                                                              |
| Format Response - Code2| n8n-nodes-base.code                   | Normalize AI response for Web       | AI Agent - Web   | Respond to Webhook3       |                                                                                                                              |
| Format Response - Code3| n8n-nodes-base.code                   | Normalize AI response for RAG       | AI Agent - Rag   | Respond to Webhook4       |                                                                                                                              |
| Respond to Webhook1    | n8n-nodes-base.respondToWebhook       | Respond with JSON (General agent)   | Format Response - Code | -                    |                                                                                                                              |
| Respond to Webhook2    | n8n-nodes-base.respondToWebhook       | Respond with JSON (Database agent)  | Format Response - Code1| -                    |                                                                                                                              |
| Respond to Webhook3    | n8n-nodes-base.respondToWebhook       | Respond with JSON (Web agent)       | Format Response - Code2| -                    |                                                                                                                              |
| Respond to Webhook4    | n8n-nodes-base.respondToWebhook       | Respond with JSON (RAG agent)       | Format Response - Code3| -                    |                                                                                                                              |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Workflow for UI Interface:**

   1.1 Add a **Webhook** node:  
       - HTTP Method: GET  
       - Path: (e.g.) `/your-interface` (unique path)  
       - Response Mode: `Response Node`

   1.2 Add a **Code** node connected to the Webhook node:  
       - Paste the provided JavaScript code that generates the full HTML page with embedded CSS and JS for the UI.  
       - Ensure the embedded `WEBHOOK_URL` JavaScript constant is set to your POST webhook URL (e.g., `/webhook-endpoint`).

   1.3 Add a **Respond to Webhook** node connected to the Code node:  
       - Respond With: Binary  
       - Property Name: `data` (matches the binary data key from Code node output)

   1.4 Connect nodes in order: Webhook → Code → Respond to Webhook.

2. **Create Workflow for AI Agent Processing:**

   2.1 Add a **Webhook** node:  
       - HTTP Method: POST  
       - Path: `/webhook-endpoint` (must match the UI JS webhook URL)  
       - Response Mode: `Response Node`

   2.2 Add a **Switch** node connected to the Webhook node:  
       - Add 4 rules based on expression: `{{$json.body.agent_type}}` equals one of `general`, `database`, `web`, or `rag`.

   2.3 Add four **Langchain AI Agent** nodes:  
       - Names: `AI Agent - General`, `AI Agent Database`, `AI Agent - Web`, `AI Agent - Rag`  
       - Set `text` parameter for each with appropriate description or prompt setup.  
       - Set `promptType` to `define`.  
       - Connect each to the corresponding Switch output for routing.  
       - Attach a GPT-4 model node (`OpenAI Chat Model`) to each agent node:  
         - Model: `gpt-4.1-mini`  
         - Credentials: Your configured OpenAI API credential

   2.4 Create four **Code** nodes (one per agent) to format responses:  
       - Use provided JavaScript code to extract AI response text from multiple keys and return JSON with `response`, `agent_type`, `user_message`, and `timestamp`.  
       - Connect each AI Agent node output to its respective Format Response Code node.

   2.5 Add four **Respond to Webhook** nodes (one per agent):  
       - Connect each Format Response Code node to a Respond to Webhook node to return JSON responses.

3. **Connect and arrange nodes:**

   - For each agent type branch:  
     Switch → AI Agent → Format Response Code → Respond to Webhook

4. **Credential Setup:**

   - Configure OpenAI API credentials in n8n, assign them to all OpenAI Chat Model nodes.

5. **Finalize:**

   - Deploy both workflows.  
   - Update the frontend UI JavaScript constant `WEBHOOK_URL` to point to your AI Agent Processing workflow webhook path.  
   - Test by accessing the UI GET endpoint, sending messages, and selecting different agents.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is a graphical input template for n8n that allows clients to interact with AI agents via a web UI instead of direct API calls.                   | Sticky Note1 contains detailed setup instructions and features summary.                                                                                                      |
| The UI supports dark/light mode toggling, responsive design, and multiple specialized AI agents with loading animations and error handling.                   | Embedded in the Code in JavaScript node's HTML/JS code.                                                                                                                     |
| The project is powered by n8n and OpenAI GPT-4 (with Langchain integration) using a multi-agent architecture.                                                 | See Sticky Note and the Langchain AI Agent nodes configuration.                                                                                                             |
| To customize the UI or add more agents, modify the HTML/CSS/JS code inside the Code node and adjust the Switch node and AI Agent nodes accordingly.           | Instructions and code comments are embedded in the workflow.                                                                                                                |
| Update the JavaScript constant `WEBHOOK_URL` inside the UI code before deployment to match your n8n webhook endpoint for POST requests.                       | Highlighted in Sticky Note1 and in the main UI Code node script.                                                                                                            |
| The workflow handles multiple AI agent types: general assistant, database queries, web searches, and retrieval-augmented generation (RAG) from knowledge base.| Each agent uses the same GPT-4 model but can be customized with different prompt configurations or data sources externally.                                                  |
| For advanced usage, extend the AI Agent nodes with additional tools, external databases, or APIs as needed.                                                  | Requires editing the Langchain agent configurations or adding nodes in the AI Agent Processing workflow.                                                                     |

---

**Disclaimer:**  
The provided description and analysis are based exclusively on the supplied n8n automated workflow JSON. The workflow respects content policies and handles only legal, public data.

---