Indonesian Stock Market Research Agent with Sectors App, Gemini & Tavily

https://n8nworkflows.xyz/workflows/indonesian-stock-market-research-agent-with-sectors-app--gemini---tavily-9551


# Indonesian Stock Market Research Agent with Sectors App, Gemini & Tavily

### 1. Workflow Overview

This workflow is an Indonesian Stock Market Research Agent integrating multiple data sources and AI tools to provide users with financial insights, especially about Indonesian public companies listed on IDX. It targets use cases such as answering user queries about stock market sectors, company financials, investment opportunities, and real-time news validation.

The workflow logically divides into the following blocks:

- **1.1 Input Reception**: Captures user inputs from various channels – native n8n chat UI, Telegram, and an HTTP webhook.
- **1.2 Main AI Agent Processing**: Core AI agent (MainAdvisor) interprets user intent, manages conversation memory, and delegates tasks to specialized sub-agents.
- **1.3 Specialized Agents**: Dedicated agents for:
  - Sectors App API (financial data for Indonesian companies)
  - Web Search (real-time news and data validation via Tavily and Google Grounding)
  - Vector Store Knowledge (Algoritma Data Science School internal knowledge)
- **1.4 Vector Document Processing**: Handles vector embeddings and storage of documents for knowledge retrieval.
- **1.5 Response Delivery**: Sends responses back to users through chat, Telegram, or HTTP webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives user input from three sources: native n8n chat UI, Telegram bot, and HTTP webhook. Inputs are normalized and forwarded for processing.

**Nodes Involved:**  
- When chat message received  
- Telegram Trigger  
- Webhook  
- If (conditions to check presence of file uploads)  
- If1  
- If2  
- Sub-MainAgent (for native chat)  
- Sub-MainAgent1 (for Telegram)  
- Sub-MainAgent2 (for webhook)  
- Respond to Chat  
- Send a text message  
- Respond to Webhook  

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point for native n8n chat UI messages  
  - Config: Public webhook enabled, supports file uploads  
  - Output: JSON with chatInput and sessionId  
  - Edge cases: File uploads presence checked by subsequent If node  
  - Connections: To If and Sub-MainAgent

- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Listen for Telegram messages  
  - Config: Listens to "message" updates, linked with Telegram API credentials  
  - Connections: To Sub-MainAgent1 and If1  
  - Edge cases: Telegram API auth failures, message format inconsistencies

- **Webhook**  
  - Type: HTTP Webhook (POST)  
  - Role: Receive external web requests with chatInput and session_id  
  - Config: Path "n8n_chat_agent", responseMode "responseNode"  
  - Connections: To Sub-MainAgent2 and If2  
  - Edge cases: HTTP request validation, missing parameters

- **If, If1, If2**  
  - Type: Conditional nodes  
  - Role: Check if incoming messages contain file uploads (files[0] exists)  
  - Config: Object existence check on files[0]  
  - Connections: If true, call Sub-VectorStore workflows for vector document processing  
  - Edge cases: No files present, malformed input data

- **Sub-MainAgent, Sub-MainAgent1, Sub-MainAgent2**  
  - Type: Execute Workflow (sub-workflow)  
  - Role: Process chat inputs through main AI agent workflow  
  - Config: Passes chatInput and sessionId from respective triggers  
  - Connections: Output forwarded to Respond to Chat, Telegram message sender, or Respond to Webhook  
  - Edge cases: Sub-workflow failures, missing inputs

- **Respond to Chat**  
  - Type: Langchain Chat node  
  - Role: Return AI response back to native n8n chat UI  
  - Config: Sends message from sub-workflow's output

- **Send a text message**  
  - Type: Telegram node  
  - Role: Send text response to Telegram user  
  - Config: ChatId extracted from Telegram Trigger node, message from sub-workflow output

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Return AI response as JSON to HTTP webhook caller  
  - Config: JSON body with AIResponse field from sub-workflow output

---

#### 2.2 Main AI Agent Processing

**Overview:**  
Central AI agent (MainAdvisor) interprets user queries, manages conversation memory, and delegates to specialized agents for detailed processing.

**Nodes Involved:**  
- Start (execute workflow trigger)  
- AI Agent (Langchain Agent node)  
- Postgres Chat Memory  
- Google Gemini Chat Model  
- VectorStore-CompanyKnowledge  
- Embeddings Google Gemini1  
- Embeddings Google Gemini  

**Node Details:**

- **Start**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for sub-workflows, accepts chatInput and sessionId as inputs

- **AI Agent**  
  - Type: Langchain Agent node  
  - Role: MainAdvisor agent processing user intent and delegating tools  
  - Config: System message defines roles and three tools: Sub-SectorsApp, Sub-WebSearch, Vector Store Knowledge  
  - Uses tools to produce concise, structured, actionable responses  
  - Inputs: chatInput, sessionId, memory from Postgres Chat Memory, relevant vector store data  
  - Outputs: Structured response to caller  
  - Edge cases: Model timeout, invalid session keys, tool invocation failures

- **Postgres Chat Memory**  
  - Type: Langchain Memory node using Postgres  
  - Role: Stores and retrieves chat session history keyed by sessionId + workflowId  
  - Config: Custom sessionKey to isolate data per user session and workflow  
  - Edge cases: DB connection failures, session data corruption

- **Google Gemini Chat Model & Google Gemini Chat Model1, 2**  
  - Type: Langchain LM Chat nodes (Google Gemini/Palm API)  
  - Role: Provide language model capabilities to AI agents  
  - Config: Temperature setting (0.4 for main model), API credentials configured  
  - Edge cases: API quota limits, authentication errors, request timeouts

- **VectorStore-CompanyKnowledge**  
  - Type: Langchain Vector Store Supabase node  
  - Role: Retrieve company-related knowledge vectors for AI agent use  
  - Config: Retrieves top 10 results from Supabase "documents" table  
  - Edge cases: Vector store connectivity, empty results

- **Embeddings Google Gemini, Embeddings Google Gemini1**  
  - Type: Langchain Embeddings nodes using Google Gemini embeddings  
  - Role: Generate embeddings for documents or queries to insert or query vector store  
  - Config: Uses model "models/gemini-embedding-001" with Google Palm API credentials  
  - Edge cases: API errors, embedding generation failures

---

#### 2.3 Specialized Agents

**Overview:**  
Dedicated Langchain agents specialized for specific tasks: querying the Sectors App API for Indonesian financial data, conducting web searches for real-time information, and handling vector store knowledge.

**Nodes Involved:**  
- Sub-SectorsApp (toolWorkflow node)  
- Sub-WebSearch (toolWorkflow node)  
- Spec Agent - Sectors App (Langchain Agent)  
- Spec Agent - Web Search (Langchain Agent)  
- WebSeach - Tavily (HTTP Request Tool)  
- WebSearch - Google Grounding (HTTP Request Tool)  
- GetCompanies (HTTP Request Tool)  
- GetSubsectors (HTTP Request Tool)  
- GetCompanyReport (HTTP Request Tool)  
- GetDailyTransData (HTTP Request Tool)  
- GetTopCompaniesRanked (HTTP Request Tool)  

**Node Details:**

- **Sub-SectorsApp & Sub-WebSearch**  
  - Type: Tool Workflow nodes invoking sub-workflows specialized in Sectors App API and web search respectively  
  - Role: Provide modular, dedicated processing for complex API calls and searches  
  - Config: Pass inputs and receive outputs transparently  
  - Edge cases: Sub-workflow availability, input/output mapping errors

- **Spec Agent - Sectors App**  
  - Type: Langchain Agent node  
  - Role: Handles user queries about Indonesian stock sectors, companies, and rankings using Sectors App API  
  - Config: Detailed system message outlining API endpoints, usage patterns, validation steps, and error handling strategies  
  - Uses HTTP Request Tool nodes (GetSubsectors, GetCompanies, GetCompanyReport, GetDailyTransData, GetTopCompaniesRanked) as "tools" for API calls  
  - Edge cases: API key errors, subsector validation failures, empty data responses

- **Spec Agent - Web Search**  
  - Type: Langchain Agent node  
  - Role: Conducts web search to validate real-time information or gather news  
  - Config: Uses WebSeach - Tavily and WebSearch - Google Grounding HTTP Request Tool nodes  
  - Edge cases: API limits, response delays, insufficient data

- **WebSeach - Tavily**  
  - Type: HTTP Request Tool node  
  - Role: Send POST requests to Tavily API for targeted web search  
  - Config: Accepts parameters like query, topic (general, news, finance), search_depth, start_date, end_date  
  - Auth: HTTP Header Auth with Tavily API key

- **WebSearch - Google Grounding**  
  - Type: HTTP Request Tool node  
  - Role: Calls Google Gemini API for web search and content generation with grounding  
  - Config: Sends structured JSON body with query parts and tools spec for google_search  
  - Auth: Uses Google API key in headers

- **GetCompanies, GetSubsectors, GetCompanyReport, GetDailyTransData, GetTopCompaniesRanked**  
  - Type: HTTP Request Tool nodes  
  - Role: Make authenticated API calls to Sectors App endpoints for retrieving company lists, subsectors, reports, daily transaction data, and rankings  
  - Config: Use HTTP Header Auth with Sectors App API key  
  - Query parameters dynamically injected from AI prompts or workflow inputs  
  - Edge cases: API rate limits, invalid parameters, empty or partial data

---

#### 2.4 Vector Document Processing

**Overview:**  
Processes input documents for vector embedding and inserts them into the Supabase vector store for knowledge retrieval.

**Nodes Involved:**  
- Default Data Loader  
- Embeddings Google Gemini  
- Supabase Vector Store  
- Sticky Note (Vector Document Processing)

**Node Details:**

- **Default Data Loader**  
  - Type: Document Default Data Loader  
  - Role: Loads input documents in binary format to prepare for embedding  
  - Config: Accepts binary data inputs

- **Embeddings Google Gemini**  
  - Type: Langchain Embeddings node  
  - Role: Generate vector embeddings from documents  
  - Config: Uses Gemini embedding model

- **Supabase Vector Store**  
  - Type: Langchain Vector Store Supabase node  
  - Role: Inserts generated embeddings into Supabase "documents" table  
  - Config: Batch size 32 for embedding insertions

- **Sticky Note**  
  - Content: "Vector Document Processing — handling input document to vector store"  
  - Serves as documentation for this block

---

#### 2.5 Response Delivery

**Overview:**  
Sends AI-generated responses back to users via appropriate channels based on the input source.

**Nodes Involved:**  
- Respond to Chat (for native n8n chat UI)  
- Send a text message (for Telegram)  
- Respond to Webhook (for HTTP webhook)  

**Node Details:**

- **Respond to Chat**  
  - Sends text reply message from Sub-MainAgent output to native chat UI

- **Send a text message**  
  - Sends text reply to Telegram chat ID extracted from incoming message

- **Respond to Webhook**  
  - Returns JSON response with AIResponse field for HTTP webhook caller

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                                     | Input Node(s)                          | Output Node(s)                    | Sticky Note                                                                                                           |
|---------------------------|--------------------------------------|----------------------------------------------------|--------------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger                | Receive native n8n chat UI message                  | -                                    | If, Sub-MainAgent                | ## Input - Native n8n Chat UI receive and process input from native n8n ui chat                                        |
| If                        | If                                   | Check if message contains file upload               | When chat message received            | Call Sub-VectorStore             |                                                                                                                       |
| Call Sub-VectorStore       | Execute Workflow                     | Process vector store insertion for file uploads     | If                                  | -                               |                                                                                                                       |
| Sub-MainAgent             | Execute Workflow                     | Process native chat input with AI agent             | When chat message received            | Respond to Chat                 |                                                                                                                       |
| Respond to Chat           | Langchain Chat                      | Reply to native chat UI                               | Sub-MainAgent                       | -                               |                                                                                                                       |
| Telegram Trigger          | Telegram Trigger                    | Receive Telegram messages                            | -                                    | Sub-MainAgent1, If1             | ## Input - Telegram Bot receive and process input from native n8n ui chat                                              |
| If1                       | If                                   | Check if Telegram message contains file upload      | Telegram Trigger                     | Call Sub-VectorStore1            |                                                                                                                       |
| Call Sub-VectorStore1      | Execute Workflow                     | Process vector store insertion for Telegram files   | If1                                 | -                               |                                                                                                                       |
| Sub-MainAgent1            | Execute Workflow                     | Process Telegram chat input with AI agent           | Telegram Trigger                    | Send a text message             |                                                                                                                       |
| Send a text message       | Telegram                            | Send response back to Telegram user                  | Sub-MainAgent1                      | -                               |                                                                                                                       |
| Webhook                   | HTTP Webhook                       | Receive HTTP POST chat input                          | -                                    | Sub-MainAgent2, If2             | ## Input - Web UI (Webhook) receive and process input from native n8n ui chat                                          |
| If2                       | If                                   | Check if webhook input contains file upload          | Webhook                            | Call Sub-VectorStore2            |                                                                                                                       |
| Call Sub-VectorStore2      | Execute Workflow                     | Process vector store insertion for webhook files    | If2                                 | -                               |                                                                                                                       |
| Sub-MainAgent2            | Execute Workflow                     | Process webhook chat input with AI agent             | Webhook                            | Respond to Webhook             |                                                                                                                       |
| Respond to Webhook        | Respond to Webhook                 | Reply to webhook caller with AI response             | Sub-MainAgent2                     | -                               |                                                                                                                       |
| Start                     | Execute Workflow Trigger           | Entry point for sub-workflows                         | -                                    | AI Agent                       |                                                                                                                       |
| AI Agent                  | Langchain Agent                   | MainAdvisor AI agent processing user queries         | Start, Postgres Chat Memory, VectorStore-CompanyKnowledge, LM nodes | Outputs to sub-workflows/tools | ## Main Agent processing request and delegate task to specialized agent                                                 |
| Postgres Chat Memory      | Langchain Memory (Postgres)         | Manage chat session memory                            | -                                    | AI Agent                       |                                                                                                                       |
| Google Gemini Chat Model  | Langchain LM Chat (Google Gemini)  | Language model for AI agent                           | AI Agent                         | AI Agent                       |                                                                                                                       |
| VectorStore-CompanyKnowledge | Langchain Vector Store (Supabase) | Retrieve company knowledge vectors                    | Embeddings Google Gemini1           | AI Agent                       |                                                                                                                       |
| Embeddings Google Gemini1 | Langchain Embeddings (Google Gemini) | Generate embeddings for vector store retrieval       | -                                    | VectorStore-CompanyKnowledge   |                                                                                                                       |
| Embeddings Google Gemini  | Langchain Embeddings (Google Gemini) | Generate embeddings for document insertion            | Default Data Loader                 | Supabase Vector Store          |                                                                                                                       |
| Sub-SectorsApp            | Tool Workflow                    | Sub-workflow for Sectors App API queries              | AI Agent (tool)                   | AI Agent (tool)                | ## Specialized Agent - Sectors App handling data request through Sectors App API                                       |
| Sub-WebSearch             | Tool Workflow                    | Sub-workflow for web search queries                    | AI Agent (tool)                   | AI Agent (tool)                | ## Specialized Agent - Web Grounding search real time information to verify answer                                      |
| Spec Agent - Sectors App  | Langchain Agent                   | Sectors App API specialized AI agent                  | GetCompanies, GetSubsectors, GetCompanyReport, GetDailyTransData, GetTopCompaniesRanked | Sub-SectorsApp tools          | ## Specialized Agent - Sectors App handling data request through Sectors App API                                       |
| Spec Agent - Web Search   | Langchain Agent                   | Web search specialized AI agent                        | WebSeach - Tavily, WebSearch - Google Grounding | Sub-WebSearch tools           |                                                                                                                       |
| WebSeach - Tavily         | HTTP Request Tool                | Query Tavily web search API                            | Spec Agent - Web Search (tool)     | Spec Agent - Web Search       |                                                                                                                       |
| WebSearch - Google Grounding | HTTP Request Tool                | Query Google Gemini web grounding API                  | Spec Agent - Web Search (tool)     | Spec Agent - Web Search       |                                                                                                                       |
| GetCompanies              | HTTP Request Tool                | Sectors App API endpoint for company list             | Spec Agent - Sectors App (tool)    | Spec Agent - Sectors App      |                                                                                                                       |
| GetSubsectors             | HTTP Request Tool                | Sectors App API endpoint for subsector list            | Spec Agent - Sectors App (tool)    | Spec Agent - Sectors App      |                                                                                                                       |
| GetCompanyReport          | HTTP Request Tool                | Sectors App API endpoint for company report            | Spec Agent - Sectors App (tool)    | Spec Agent - Sectors App      |                                                                                                                       |
| GetDailyTransData         | HTTP Request Tool                | Sectors App API endpoint for daily transaction data    | Spec Agent - Sectors App (tool)    | Spec Agent - Sectors App      |                                                                                                                       |
| GetTopCompaniesRanked     | HTTP Request Tool                | Sectors App API endpoint for top companies ranking     | Spec Agent - Sectors App (tool)    | Spec Agent - Sectors App      |                                                                                                                       |
| Default Data Loader       | Document Data Loader             | Load documents for vector embedding                     | -                                    | Embeddings Google Gemini      | ## Vector Document Processing handling input document to vector store                                                  |
| Supabase Vector Store     | Langchain Vector Store (Supabase) | Insert document embeddings into vector store           | Embeddings Google Gemini           | -                             |                                                                                                                       |
| Sticky Note               | Sticky Note                      | Documentation for vector document processing block     | -                                    | -                             | ## Vector Document Processing handling input document to vector store                                                  |
| Sticky Note1              | Sticky Note                      | Documentation for Telegram input block                  | -                                    | -                             | ## Input - Telegram Bot receive and process input from native n8n ui chat                                              |
| Sticky Note2              | Sticky Note                      | Documentation for native n8n chat UI input block        | -                                    | -                             | ## Input - Native n8n Chat UI receive and process input from native n8n ui chat                                        |
| Sticky Note3              | Sticky Note                      | Documentation for HTTP webhook input block              | -                                    | -                             | ## Input - Web UI (Webhook) receive and process input from native n8n ui chat                                          |
| Sticky Note4              | Sticky Note                      | Documentation for Main Agent processing block           | -                                    | -                             | ## Main Agent processing request and delegate task to specialized agent                                                |
| Sticky Note5              | Sticky Note                      | Documentation for Specialized Agent - Sectors App       | -                                    | -                             | ## Specialized Agent - Sectors App handling data request through Sectors App API                                       |
| Sticky Note6              | Sticky Note                      | Documentation for Specialized Agent - Web Grounding     | -                                    | -                             | ## Specialized Agent - Web Grounding search real time information to verify answer                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Reception Nodes:**

   - *When chat message received*: Add Langchain Chat Trigger node. Enable public webhook, allow file uploads, responseMode "responseNodes". Position appropriately.
   - *Telegram Trigger*: Add Telegram Trigger node, configure with Telegram credentials, listen to "message" updates.
   - *Webhook*: Add HTTP Webhook node with POST method, path "n8n_chat_agent", responseMode "responseNode".

2. **Add Conditional Checks for File Uploads:**

   - Add three If nodes ("If", "If1", "If2") each checking if `files[0]` exists on respective input JSON.
   - Connect "When chat message received" to "If", "Telegram Trigger" to "If1", and "Webhook" to "If2".
   - Configure conditions as object existence checks on `files[0]`.

3. **Add Sub-Workflow Calls for Vector Store Processing:**

   - Create three Execute Workflow nodes ("Call Sub-VectorStore", "Call Sub-VectorStore1", "Call Sub-VectorStore2") linked from the true branch of the respective If nodes to handle vector document processing.
   - Set workflow ID to the vector store processing sub-workflow (ID: LGqCjjPrBqibM5YW).
   - Set mappingMode "defineBelow" with empty inputs.

4. **Add Sub-Workflow Calls for Main AI Processing:**

   - Add three Execute Workflow nodes ("Sub-MainAgent", "Sub-MainAgent1", "Sub-MainAgent2").
   - Configure inputs:
     - For native chat: pass `chatInput` and `sessionId` from JSON.
     - For Telegram: map `message.text` as `chatInput` and `message.chat.id` as `sessionId`.
     - For webhook: map `body.chatInput` and `body.session_id`.
   - Connect "When chat message received" to "Sub-MainAgent", "Telegram Trigger" to "Sub-MainAgent1", and "Webhook" to "Sub-MainAgent2".

5. **Add Response Nodes:**

   - Add "Respond to Chat" Langchain Chat node connected from "Sub-MainAgent" output. Set message to `$json.output`.
   - Add "Send a text message" Telegram node connected from "Sub-MainAgent1". Configure chatId from Telegram Trigger's message chat id, text from sub-workflow output.
   - Add "Respond to Webhook" node connected from "Sub-MainAgent2". Set JSON response with field `AIResponse` from output.

6. **Build Main AI Agent Sub-Workflow:**

   - Add "Start" Execute Workflow Trigger with inputs `chatInput` and `sessionId`.
   - Add "Postgres Chat Memory" node for memory storage using Postgres credentials.
   - Add "Google Gemini Chat Model" node with temperature 0.4 and Google Palm API credentials.
   - Add "Embeddings Google Gemini1" and "VectorStore-CompanyKnowledge" nodes connected for vector knowledge retrieval.
   - Add "AI Agent" node configured with system message defining MainAdvisor role and tools (Sub-SectorsApp, Sub-WebSearch, Vector Store Knowledge).
   - Connect nodes to build a pipeline: Start → Postgres Chat Memory → AI Agent (with LM and vector store inputs).

7. **Create Specialized Agent Sub-Workflows:**

   - **Sectors App Agent:**
     - Use HTTP Request Tool nodes for endpoints:
       - GetSubsectors: GET https://api.sectors.app/v1/subsectors with HTTP Header Auth.
       - GetCompanies: GET https://api.sectors.app/v1/companies with dynamic query parameters.
       - GetCompanyReport, GetDailyTransData, GetTopCompaniesRanked similarly configured.
     - Create Langchain Agent node "Spec Agent - Sectors App" with detailed system prompt describing subsector validation, API usage, error handling.
     - Integrate HTTP Request nodes as tools.
     - Wrap in a Tool Workflow node "Sub-SectorsApp" to be called by MainAgent.

   - **Web Search Agent:**
     - Add HTTP Request Tool nodes:
       - WebSeach - Tavily: POST https://api.tavily.com/search with query, topic, dates, authenticated via HTTP Header Auth.
       - WebSearch - Google Grounding: POST to Google Gemini API with structured JSON body and API key header.
     - Create Langchain Agent node "Spec Agent - Web Search" using these tools.
     - Wrap in Tool Workflow node "Sub-WebSearch".

8. **Setup Vector Document Processing:**

   - Add "Default Data Loader" to load binary documents.
   - Add "Embeddings Google Gemini" to generate embeddings.
   - Add "Supabase Vector Store" node to insert embeddings to "documents" table.
   - Connect nodes accordingly.

9. **Add Sticky Notes for Documentation:**

   - Add descriptive sticky notes in the canvas to highlight the purpose of major blocks:
     - Input blocks (Native chat, Telegram, Webhook)
     - Main Agent processing
     - Specialized Agents (Sectors App, Web Grounding)
     - Vector Document Processing

10. **Configure Credentials:**

    - Google Palm API credentials for Gemini model and embeddings.
    - Postgres credentials for chat memory.
    - Supabase credentials for vector store.
    - Telegram API credentials.
    - HTTP Header Auth credentials for Sectors App API and Tavily API.

11. **Test Workflow:**

    - Verify input reception from all three sources.
    - Ensure AI agent correctly delegates to specialized agents.
    - Confirm API calls to Sectors App and web search work and return data.
    - Validate responses are sent back via the correct channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The Sectors App API is a key data provider, offering real-time financial data for most Indonesian IDX-listed companies. The specialized agent uses a rigorous 2-step subsector validation process before querying company lists to ensure accuracy and meaningful responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Detailed in "Spec Agent - Sectors App" system message node                                                       |
| The workflow uses Google Gemini (PaLM) API for language modeling and embedding generation, requiring appropriate API credentials and quota management.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Google Gemini API documentation for language and embedding models                                                |
| For real-time web information validation and news, the workflow integrates Tavily API and Google Gemini’s web grounding, facilitating up-to-date and accurate AI responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Tavily API documentation and Google Gemini grounding search API                                                  |
| Postgres database is used for chat memory to maintain session context across interactions, improving conversational coherence. Ensure reliable DB connectivity and backup strategies.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Postgres Chat Memory node setup                                                                                   |
| Supabase vector store is used for document embedding storage and retrieval, enabling fast knowledge lookup for company data and Algoritma internal information.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Supabase documentation and Langchain vector store integration                                                     |
| The workflow supports file uploads in all input channels, routed to vector store insertion sub-workflows, allowing document ingestion for knowledge expansion.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | File upload handling via If nodes and Sub-VectorStore executions                                                 |
| Telegram integration requires bot API token and proper webhook or polling setup.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Telegram Bot API documentation                                                                                   |
| The workflow is designed for modularity and scalability, with sub-workflows dedicated to specific functions, enabling easy updates or replacements of individual agents or data sources.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Use of Execute Workflow nodes for modular design                                                                  |
| For more information on Indonesian stock market data sources and APIs, visit the Sectors App official site: https://sectors.app/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | https://sectors.app/                                                                                              |
| For detailed API specs and examples for Tavily, see https://tavily.com/api                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | https://tavily.com/api                                                                                            |

---

This comprehensive documentation covers all nodes, their roles, configurations, edge cases, and interconnections, enabling reproduction or modification of the workflow in n8n for Indonesian stock market research and AI-assisted financial advisory.