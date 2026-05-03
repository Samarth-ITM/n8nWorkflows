Automated Slack IT Helpdesk with GPT, Supabase Vector Search, and JIRA Ticketing

https://n8nworkflows.xyz/workflows/automated-slack-it-helpdesk-with-gpt--supabase-vector-search--and-jira-ticketing-6549


# Automated Slack IT Helpdesk with GPT, Supabase Vector Search, and JIRA Ticketing

---

### 1. Workflow Overview

This n8n workflow implements an **Automated Slack IT Helpdesk** empowered by AI capabilities, leveraging GPT (OpenAI), Supabase vector search for knowledge retrieval, and JIRA for ticket management. The workflow is designed to enable IT support via Slack by understanding user requests, searching internal knowledge bases, and managing incident tickets in JIRA.

The workflow logically decomposes into the following functional blocks:

- **1.1 Document Ingestion (RAG Pipeline):**  
  Handles ingestion of CSV/PDF knowledge base documents uploaded via a web form, converts them to vector embeddings with OpenAI, and stores them in Supabase for later retrieval.

- **1.2 Slack Message Intake and Preprocessing:**  
  Listens to Slack events (mentions, messages, reactions) in a specified channel, formats incoming data, and filters to distinguish initial mentions from thread replies and bot messages.

- **1.3 AI Helpdesk Agent with Memory:**  
  Uses LangChain-based AI agent with GPT-4.1 to process user queries, maintaining session memory for threaded conversations, and orchestrates knowledge base searches and JIRA ticket operations via an MCP (Model Context Protocol) Server.

- **1.4 MCP Server Integration for Tools:**  
  Acts as a centralized interface exposing Supabase vector search and multiple JIRA operations (create ticket, search tickets, get status, change priority, changelog). This decouples AI logic from backend operations.

- **1.5 Slack Response Sending:**  
  Posts AI-generated responses back to Slack in the appropriate thread, ensuring conversational flow.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Document Ingestion (RAG Pipeline)

**Overview:**  
This block enables uploading CSV and PDF documents containing IT helpdesk knowledge, converts them into vector embeddings using OpenAI, and stores these embeddings in a Supabase vector database. It builds a searchable knowledge repository for use by the AI helpdesk.

**Nodes Involved:**  
- On form submission  
- Default Data Loader  
- Embeddings OpenAI  
- Supabase Vector Store  
- Sticky Note (Document Ingestion Flow explanation)

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Accepts user uploads of CSV/PDF files through a web form interface.  
  - *Config:* Accepts only `.csv` and `.pdf` files; form titled “Document Uploader”.  
  - *Input:* User-submitted files.  
  - *Output:* Binary file data forwarded for processing.  
  - *Edge cases:* Invalid file types rejected; no files submitted.  

- **Default Data Loader**  
  - *Type:* Document Data Loader  
  - *Role:* Converts binary uploaded files into document format compatible with embedding generation.  
  - *Config:* Uses default loaders; data type set to binary.  
  - *Input:* Binary files from form trigger.  
  - *Output:* Parsed document data for embedding.  
  - *Edge cases:* Parsing failures for malformed files.  

- **Embeddings OpenAI**  
  - *Type:* OpenAI Embeddings Node  
  - *Role:* Generates 1536-dimensional vector embeddings from document content.  
  - *Config:* Dimensions set to 1536 (compatible with OpenAI embeddings API).  
  - *Input:* Parsed document text.  
  - *Output:* Embeddings vector data.  
  - *Credentials:* OpenAI API key required.  
  - *Edge cases:* API rate limits, network timeouts.  

- **Supabase Vector Store**  
  - *Type:* Vector Store (Supabase)  
  - *Role:* Stores generated embeddings in Supabase table `documents`.  
  - *Config:* Mode set to “insert” to add new vectors; uses `match_documents` query.  
  - *Input:* Embeddings from OpenAI node.  
  - *Output:* Confirmation of storage.  
  - *Credentials:* Supabase API connection needed.  
  - *Edge cases:* Authentication errors, DB connectivity issues.  

- **Sticky Note (Document Ingestion Flow)**  
  - *Purpose:* Documentation describing the ingestion process, accepted formats, and usage notes.

---

#### 2.2 Slack Message Intake and Preprocessing

**Overview:**  
This block captures Slack events (mentions, messages, reactions) from a specified Slack channel, extracts and formats relevant message data, and applies filters to identify new mentions versus thread messages, while ignoring bot messages to prevent loops.

**Nodes Involved:**  
- Slack Trigger  
- Format Output  
- Check app_mention (If node)  
- Check If thread (If node)  
- Sticky Note (Main Helpdesk Flow explanation)

**Node Details:**  

- **Slack Trigger**  
  - *Type:* Slack Trigger  
  - *Role:* Listens for Slack events `app_mention`, `message`, and `reaction_added` in channel `C0963H18JCX`.  
  - *Config:* Resolves IDs, channel is fixed to a specific channel ID.  
  - *Credentials:* Slack API OAuth2 token with appropriate scopes.  
  - *Output:* Raw Slack event JSON.  
  - *Edge cases:* Missing permissions, Slack API rate limits, event subscription misconfigurations.  

- **Format Output**  
  - *Type:* Set Node  
  - *Role:* Cleans and consolidates Slack event JSON into normalized fields such as `channel`, `ts`, `thread_ts`, `text`, `is_bot`, `user`, and reaction details.  
  - *Config:* Extracts text from message blocks or fallback text fields; determines if message is from bot or emoji reaction.  
  - *Input:* Slack Trigger output.  
  - *Output:* Structured JSON with relevant keys for downstream logic.  
  - *Edge cases:* Complex message block structures; missing fields.  

- **Check app_mention**  
  - *Type:* If Node  
  - *Role:* Checks if incoming message is an `app_mention` type, not from a bot, and not an emoji reaction.  
  - *Config:* Conditions: `type == app_mention`, `is_bot == false`, `is_emoji == false`.  
  - *Input:* Formatted Slack output.  
  - *Output:* Two branches: true (mention) leads to AI processing, false leads to thread check.  
  - *Edge cases:* False positives if fields missing or malformed.  

- **Check If thread**  
  - *Type:* If Node  
  - *Role:* Checks if message is a thread continuation, i.e., type `message`, non-bot, and `thread_ts` present.  
  - *Config:* Conditions: `type == message`, `thread_ts` not empty, and `is_bot == false`.  
  - *Input:* False branch from app_mention check.  
  - *Output:* True branch feeds AI processing; false branch ignored (e.g., bot messages).  
  - *Edge cases:* Messages without thread_ts miscategorized.  

- **Sticky Note (Main Helpdesk Flow)**  
  - *Purpose:* Detailed explanation of Slack event handling logic, filtering rules, and conversation flow management to avoid loops and maintain session context.

---

#### 2.3 AI Helpdesk Agent with Memory

**Overview:**  
Processes filtered Slack messages with a LangChain AI agent using GPT-4.1. Maintains conversation context via a memory buffer keyed by Slack thread ID. The agent uses custom system prompts to act as an expert IT helpdesk, performs vector searches, interacts with JIRA, and formats responses accordingly.

**Nodes Involved:**  
- Simple Memory  
- AIhelpdesk (LangChain Agent)  
- helpdesk_tools (MCP Client Tool)  
- OpenAI  
- Send a message

**Node Details:**  

- **Simple Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Stores last 20 messages per session, keyed by Slack thread or message ID (`sessionKey = thread_ts || ts`).  
  - *Config:* Custom session ID type, context window size 20.  
  - *Input:* Slack message session ID from formatted output.  
  - *Output:* Provides conversation history to AIhelpdesk node.  
  - *Edge cases:* Memory overflow, session ID missing.  

- **AIhelpdesk**  
  - *Type:* LangChain Agent Node  
  - *Role:* Core AI agent that processes user input, applies system prompt for IT helpdesk behavior, manages vector and JIRA searches via MCP tools, and generates response text.  
  - *Config:* Uses GPT-4.1 model with temperature 0.7; system message defines detailed rules for thread awareness, reaction options, search strategies, escalation, and ticketing.  
  - *Input:* Cleaned Slack message text and session memory.  
  - *Output:* Generates response text and tool API calls.  
  - *Edge cases:* Model API errors, context window limits, malformed input.  
  - *Notes:* The system message enforces no greeting in thread continuations, reaction-driven interaction, and multi-step search and ticket logic.  

- **helpdesk_tools**  
  - *Type:* MCP Client Tool  
  - *Role:* Connects AI agent to MCP Server endpoint (`heldpesk`), enabling vector search and JIRA ticket operations as AI tools.  
  - *Config:* SSE endpoint URL configured for MCP Server.  
  - *Input:* Calls from AIhelpdesk agent to invoke external tools.  
  - *Output:* Returns search results, ticket IDs, statuses.  
  - *Edge cases:* Connectivity errors, wrong endpoint, tool response delays.  

- **OpenAI**  
  - *Type:* LangChain Language Model (Chat OpenAI)  
  - *Role:* Executes GPT-4.1 chat completions as backend for AIhelpdesk node.  
  - *Config:* Model GPT-4.1, temperature 0.7, top_p 1.  
  - *Input:* Formatted prompts from AIhelpdesk.  
  - *Output:* AI-generated text completions.  
  - *Credentials:* OpenAI API key.  
  - *Edge cases:* API rate limits, network issues.  

- **Send a message**  
  - *Type:* Slack Node  
  - *Role:* Posts AI-generated replies back to Slack channel and thread identified by original message.  
  - *Config:* Sends to channel and thread timestamp (`thread_ts` or `ts`), text set from AIhelpdesk output.  
  - *Input:* AIhelpdesk response text.  
  - *Output:* Confirmation of Slack message posted.  
  - *Credentials:* Slack API OAuth2 token.  
  - *Edge cases:* Slack API rate limits, invalid thread_ts.  

---

#### 2.4 MCP Server Integration for Tools

**Overview:**  
This block represents the MCP Server that exposes external services like Supabase vector search and various JIRA ticketing operations as API tools accessible by the AI agent.

**Nodes Involved:**  
- MCP Server Trigger  
- Supabase Vector Store1  
- Embeddings OpenAI1  
- Create Ticket  
- Search (JIRA)  
- Get Status (JIRA)  
- Change Priority (JIRA)  
- changelog (JIRA)  
- Sticky Note (MCP Server Overview)

**Node Details:**  

- **MCP Server Trigger**  
  - *Type:* MCP Trigger  
  - *Role:* Listens for incoming MCP client requests at path `/heldpesk`.  
  - *Config:* Path configured as `heldpesk`.  
  - *Input:* Tool invocation requests from AIhelpdesk agent.  
  - *Output:* Routes requests to proper tool nodes.  
  - *Edge cases:* Invalid tool requests, network issues.  

- **Supabase Vector Store1**  
  - *Type:* Vector Store (Supabase)  
  - *Role:* Retrieves relevant documents from Supabase vector table `documents` based on query embeddings.  
  - *Config:* Mode `retrieve-as-tool`; includes descriptive toolDescription for AI agent use.  
  - *Input:* Embeddings generated by Embeddings OpenAI1 node.  
  - *Output:* Retrieved documents matched to query.  
  - *Credentials:* Supabase API.  
  - *Edge cases:* Query failures, empty results.  

- **Embeddings OpenAI1**  
  - *Type:* OpenAI Embeddings Node  
  - *Role:* Generates embeddings for MCP server queries with 1536 dimensions.  
  - *Config:* Same as main embeddings node.  
  - *Credentials:* OpenAI API.  
  - *Edge cases:* API limits.  

- **Create Ticket**  
  - *Type:* JIRA Tool  
  - *Role:* Creates new helpdesk tickets in JIRA project ID 10000 (Helpdesk) with issue type 10004 (Service Request).  
  - *Config:* Summary and description are AI-generated overrides; uses JIRA Software Cloud API credentials.  
  - *Input:* Ticket details from AI agent.  
  - *Output:* Confirmation and ticket key.  
  - *Edge cases:* JIRA auth errors, invalid fields.  

- **Search (JIRA)**  
  - *Type:* JIRA Tool  
  - *Role:* Retrieves tickets matching criteria; can return all or limited results.  
  - *Config:* Operation `getAll`; returnAll parameter driven by AI override boolean.  
  - *Input:* Search queries from AI agent.  
  - *Output:* List of matching tickets.  
  - *Edge cases:* No results, API errors.  

- **Get Status (JIRA)**  
  - *Type:* JIRA Tool  
  - *Role:* Fetches status transitions for a given issue key.  
  - *Config:* Operation `transitions`.  
  - *Input:* Issue key from AI agent.  
  - *Output:* Status info.  
  - *Edge cases:* Invalid issue key.  

- **Change Priority (JIRA)**  
  - *Type:* JIRA Tool  
  - *Role:* Updates priority level of JIRA ticket using IDs passed from AI agent.  
  - *Config:* Priority IDs mapped from priority level strings; manual description type.  
  - *Input:* Issue key and priority level from AI.  
  - *Output:* Confirmation of update.  
  - *Edge cases:* Invalid priority values, permission errors.  

- **changelog (JIRA)**  
  - *Type:* JIRA Tool  
  - *Role:* Retrieves changelog/history of a JIRA ticket.  
  - *Config:* Operation `changelog`.  
  - *Input:* Issue key.  
  - *Output:* Change history data.  
  - *Edge cases:* Permissions, missing ticket.  

- **Sticky Note (MCP Server Overview)**  
  - *Purpose:* Explains MCP Server’s role as centralized API interface for vector search and JIRA operations.

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                                    | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                         |
|---------------------|--------------------------------------|---------------------------------------------------|---------------------------|--------------------------|--------------------------------------------------------------------------------------------------|
| Slack Trigger       | Slack Trigger                        | Captures Slack events (mentions, messages, reactions) | -                         | Format Output            | See Sticky Note2 for Slack integration overview                                                  |
| Format Output       | Set                                 | Normalizes Slack event data                        | Slack Trigger             | Check app_mention         | See Sticky Note2                                                                                  |
| Check app_mention   | If                                  | Filters app mentions, non-bot, non-emoji          | Format Output             | AIhelpdesk (true), Check If thread (false) | See Sticky Note2                                                                                  |
| Check If thread     | If                                  | Filters thread continuation messages               | Check app_mention (false) | AIhelpdesk (true)         | See Sticky Note2                                                                                  |
| Simple Memory       | LangChain Memory Buffer Window       | Maintains conversation history per Slack thread   | Format Output             | AIhelpdesk                |                                                                                                  |
| AIhelpdesk          | LangChain Agent                     | Processes user query, manages conversation, calls MCP tools | Check app_mention, Check If thread | Send a message           | See Sticky Note2                                                                                  |
| helpdesk_tools      | MCP Client Tool                     | Connects AI agent to MCP Server tools              | AIhelpdesk                | MCP Server Trigger        | See Sticky Note4                                                                                  |
| OpenAI              | LangChain Language Model (Chat OpenAI) | Provides GPT-4.1 completions                       | AIhelpdesk                | AIhelpdesk                |                                                                                                  |
| Send a message      | Slack                              | Sends AI response back to Slack                     | AIhelpdesk                | -                        |                                                                                                  |
| On form submission  | Form Trigger                       | Accepts CSV/PDF knowledge base uploads              | -                         | Default Data Loader       | See Sticky Note                                                                                   |
| Default Data Loader | Document Data Loader               | Converts uploaded files to documents                 | On form submission        | Embeddings OpenAI         |                                                                                                  |
| Embeddings OpenAI   | OpenAI Embeddings Node             | Generates embeddings from document text             | Default Data Loader       | Supabase Vector Store     |                                                                                                  |
| Supabase Vector Store | Supabase Vector Store             | Stores document embeddings in Supabase vector DB    | Embeddings OpenAI         | -                        |                                                                                                  |
| MCP Server Trigger  | MCP Trigger                       | Handles MCP tool requests from AI agent             | helpdesk_tools            | Supabase Vector Store1, Create Ticket, Search, Get Status, Change Priority, changelog | See Sticky Note4                                                                                  |
| Supabase Vector Store1 | Supabase Vector Store             | Retrieves relevant documents from Supabase          | Embeddings OpenAI1        | MCP Server Trigger        |                                                                                                  |
| Embeddings OpenAI1  | OpenAI Embeddings Node             | Generates embeddings for MCP Server queries          | MCP Server Trigger        | Supabase Vector Store1    |                                                                                                  |
| Create Ticket       | JIRA Tool                        | Creates new JIRA tickets                             | MCP Server Trigger        | MCP Server Trigger        |                                                                                                  |
| Search              | JIRA Tool                        | Searches JIRA tickets                                | MCP Server Trigger        | MCP Server Trigger        |                                                                                                  |
| Get Status          | JIRA Tool                        | Gets status transitions for JIRA tickets             | MCP Server Trigger        | MCP Server Trigger        |                                                                                                  |
| Change Priority     | JIRA Tool                        | Updates priority on JIRA tickets                      | MCP Server Trigger        | MCP Server Trigger        |                                                                                                  |
| changelog           | JIRA Tool                        | Retrieves changelog of JIRA tickets                   | MCP Server Trigger        | MCP Server Trigger        |                                                                                                  |
| Sticky Note         | Sticky Note                      | Document Ingestion Flow explanation                   | -                         | -                        | Covers Document Ingestion Flow                                                                    |
| Sticky Note1        | Sticky Note                      | Workflow Overview and Slack Configuration             | -                         | -                        | Covers overall Workflow Overview and Slack Integration                                            |
| Sticky Note2        | Sticky Note                      | Main Helpdesk Flow explanation                         | -                         | -                        | Covers Slack integration and AI Helpdesk logic                                                   |
| Sticky Note4        | Sticky Note                      | MCP Server overview and tool description              | -                         | -                        | Covers MCP Server and JIRA/Supabase integration                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node:**  
   - Type: Slack Trigger  
   - Configure to listen for events: `app_mention`, `message`, `reaction_added`  
   - Set channel ID to your target Slack channel (e.g., `C0963H18JCX`)  
   - Use Slack API credentials with required OAuth scopes (app_mentions:read, channels:history, users:read, etc.)  
   - Enable resolve IDs option.  

2. **Add Format Output Node (Set):**  
   - Extract and normalize Slack message fields: `channel`, `ts`, `thread_ts`, `text` (from blocks or text), `is_bot`, `user`, `reaction`, `is_emoji`  
   - Use expressions to handle nested JSON safely.  
   - Connect Slack Trigger output to this node.  

3. **Add Check app_mention Node (If):**  
   - Condition: `type == 'app_mention'` AND `is_bot == false` AND `is_emoji == false`  
   - Connect Format Output main output to this node.  

4. **Add Check If thread Node (If):**  
   - Condition: `type == 'message'` AND `thread_ts` is not empty AND `is_bot == false`  
   - Connect Check app_mention false output to this node.  

5. **Add Simple Memory Node (LangChain Memory Buffer Window):**  
   - Session key: `{{$json.thread_ts || $json.ts}}`  
   - Context window length: 20  
   - Connect Format Output to this node (for session context).  

6. **Create AIhelpdesk Node (LangChain Agent):**  
   - Model: GPT-4.1  
   - Temperature: 0.7, topP: 1, presence and frequency penalty 0  
   - System prompt: Use the detailed helpdesk system message covering thread awareness, reaction options, search instructions, escalation, and ticketing logic.  
   - Enable session memory linking to Simple Memory node output.  
   - Connect Check app_mention true output and Check If thread true output to AIhelpdesk node.  

7. **Add helpdesk_tools Node (MCP Client Tool):**  
   - SSE endpoint URL: your MCP Server URL + `/heldpesk` path  
   - Connect AIhelpdesk tool output to this node.  

8. **Add OpenAI Node (LangChain LM Chat OpenAI):**  
   - Model: GPT-4.1  
   - Same parameters as AIhelpdesk node.  
   - Connect AIhelpdesk language model output to this node.  

9. **Add Send a message Node (Slack):**  
   - Channel: `{{$item(0).$node["Slack Trigger"].json["channel"]}}`  
   - Thread timestamp: `{{$node["Format Output"].json["thread_ts"] || $node["Format Output"].json["ts"]}}`  
   - Text: `{{$json.output}}` (the AI agent’s response)  
   - Connect AIhelpdesk main output to this node.  

10. **Setup Document Ingestion Form Trigger:**  
    - Type: Form Trigger  
    - Form title: “Document Uploader”  
    - Add file upload field accepting `.csv,.pdf` only (required)  
    - Connect to Default Data Loader node.  

11. **Add Default Data Loader Node:**  
    - Data type: binary  
    - Connect Form Trigger output to this node.  

12. **Add Embeddings OpenAI Node:**  
    - Dimensions: 1536  
    - Connect Default Data Loader output.  
    - Use OpenAI API credentials.  

13. **Add Supabase Vector Store Node:**  
    - Mode: Insert  
    - Table: `documents`  
    - Use Supabase credentials.  
    - Connect Embeddings OpenAI output.  

14. **Setup MCP Server Trigger Node:**  
    - Type: MCP Trigger  
    - Path: `heldpesk`  
    - Connect to Supabase Vector Store1, Create Ticket, Search, Get Status, Change Priority, and changelog nodes.  

15. **Add Supabase Vector Store1 Node:**  
    - Mode: Retrieve as tool  
    - Table: `documents`  
    - Connect MCP Server Trigger input to Embeddings OpenAI1 node, then to this node.  

16. **Add Embeddings OpenAI1 Node:**  
    - Same config as main embeddings node  
    - Connect MCP Server Trigger input to this node.  

17. **Add JIRA Tool Nodes:**  
    - Create Ticket: Project ID 10000 (Helpdesk), Issue Type 10004 (Service Request), summary and description parameters from AI overrides.  
    - Search: Operation getAll, returnAll configurable.  
    - Get Status: Operation transitions, issueKey from AI.  
    - Change Priority: Update priority field with AI provided ID.  
    - changelog: Operation changelog, issueKey from AI.  
    - Connect all to MCP Server Trigger node as tool endpoints.  
    - Use JIRA Software Cloud API credentials.  

18. **Add Sticky Notes at relevant points:**  
    - Document ingestion explanation  
    - Slack integration overview  
    - Main helpdesk flow description  
    - MCP server overview  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Minimum Slack app configuration requires subscribing to bot events: `app_mention`, `message.channels`, and optionally `message.groups` for private channels. OAuth scopes must cover `app_mentions:read`, `channels:history`, `channels:read`, `groups:history`, `groups:read`, `im:history`, `im:read`, `mpim:history`, `mpim:read`, and `users:read`. Invite bot to Slack channels to activate. Testing can be done by mentioning the bot with `@botname <message>`. | Slack app setup instructions                                          |
| MCP Server centralizes operations with Model Context Protocol, allowing the AI agent to access Supabase vector search and JIRA ticketing tools via a unified API endpoint. This architecture enables scalable integration and modular tool management.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | MCP Server design notes                                               |
| The AIhelpdesk system prompt enforces strict thread-aware conversational rules, reaction-based interaction patterns, vector search before response, and escalation/ticketing workflows, ensuring professional and efficient IT helpdesk assistance in Slack.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | AIhelpdesk prompt and behavior design                                |
| Only CSV and PDF document uploads are supported in the ingestion pipeline by default. To extend file format support, adjust the accepted file types and document loaders accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Document upload format notes                                          |
| Reaction emoji options for user interaction include: 👍 (create ticket/yes), ✅ (confirm), ❌ (cancel/no), ℹ️ (need more info), 👀 (escalate), 🆘 (urgent). These are integral to the AI’s interactive responses and should be preserved in Slack client UI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Interaction design via Slack reactions                               |
| For reproducing or extending the AI agent’s capabilities, consult LangChain and n8n documentation on MCP integration, memory buffers, and LangChain agents. Credentials for OpenAI, Supabase, Slack, and JIRA must be securely configured in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Integration and credential requirements                             |

---

**Disclaimer:**  
The text above is exclusively derived from an automated n8n workflow. It respects all applicable content policies and contains no illegal, offensive, or protected content. All processed data are legal and publicly available.

---