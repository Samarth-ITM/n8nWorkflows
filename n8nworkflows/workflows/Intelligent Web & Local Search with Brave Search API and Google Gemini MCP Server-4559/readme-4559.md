Intelligent Web & Local Search with Brave Search API and Google Gemini MCP Server

https://n8nworkflows.xyz/workflows/intelligent-web---local-search-with-brave-search-api-and-google-gemini-mcp-server-4559


# Intelligent Web & Local Search with Brave Search API and Google Gemini MCP Server

### 1. Workflow Overview

This workflow, titled **"Intelligent Web & Local Search with Brave Search API and Google Gemini MCP Server"**, serves as a multi-agent collaboration protocol (MCP) server exposing an AI agent specialized in performing intelligent web and local searches via the Brave Search platform. It is designed to accept user queries over an MCP SSE endpoint, decide whether to perform a general web search or a local search, execute the appropriate Brave Search MCP client tool accordingly, and return summarized results.

**Target Use Cases:**  
- Integrating with MCP-capable clients (e.g., Roo Code, Cline) to provide AI-powered search capabilities.  
- Allowing external clients to query a centralized AI agent that intelligently manages web/local search decisions and results aggregation.  
- Efficiently managing token usage and logic by centralizing complex AI prompting and tool orchestration.

**Logical Blocks:**  
- **1.1 MCP Server Trigger (Public Entry Point):** Listens for external MCP client POST requests and exposes the AI agent tool.  
- **1.2 Tool Workflow Node (Interface for AI Agent):** Acts as the callable tool by the MCP Server Trigger; passes input query to sub-workflow.  
- **1.3 Brave Search AI Agent Sub-Workflow:** Core logic that processes the query, uses AI to decide search type, calls Brave Search MCP client tools, and summarizes results.  
- **1.4 AI Model and Memory:** Google Gemini LLM model and a session-based simple memory buffer support agent context and reasoning.  
- **1.5 Brave Search MCP Client Tools:** Two MCP client nodes invoke Brave Search’s web search and local search APIs.  
- **1.6 Sticky Notes:** Documentation nodes explaining configuration, usage, testing, and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger (Public Entry Point)

**Overview:**  
This block exposes an MCP SSE endpoint that external clients connect to for querying the Brave Search AI Agent.

**Nodes Involved:**  
- Brave Search MCP Server Trigger  
- call_brave_search_agent (Tool Workflow Node)

**Node Details:**

- **Brave Search MCP Server Trigger**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Listens on a unique SSE path `/cc8cc827-3e72-4029-8a9d-76519d1c136d` for POST requests from MCP clients.  
  - Configuration: Webhook path matches the unique ID; exposes the tool `call_brave_search_agent`.  
  - Input: JSON with `{"input": {"query": "user's search query"}}`.  
  - Output: Streams SSE events with AI agent responses.  
  - Edge cases: HTTP errors, malformed JSON input, authentication failures (if any), client disconnects.  
  - Notes: Clients must replace `YOUR_N8N_INSTANCE` in URLs accordingly.

- **call_brave_search_agent**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Acts as the callable tool for the MCP Trigger node; invokes the sub-workflow with the user's query.  
  - Configuration: Points to sub-workflow ID `QclY1XQ8GEstkvk7` (the Brave Search AI Agent MCP Server sub-workflow).  
  - Input Mapping: Accepts `query` string parameter from external client.  
  - Output: Forwards AI agent’s final response back to MCP Trigger for client streaming.  
  - Edge cases: Sub-workflow invocation failures, parameter mapping issues.

---

#### 2.2 Brave Search AI Agent Sub-Workflow (Core Logic)

**Overview:**  
Processes the user query, decides (via AI) whether to use web or local search, calls the respective Brave Search MCP client tools, and generates a summarized response with context.

**Nodes Involved:**  
- Brave Search Workflow Start (Execute Workflow Trigger)  
- Brave Search AI Agent (Agent node using Google Gemini)  
- Simple Memory (Memory Buffer)  
- brave_web_search (MCP Client Tool)  
- brave_local_search (MCP Client Tool)

**Node Details:**

- **Brave Search Workflow Start**  
  - Type: `n8n-nodes-base.executeWorkflowTrigger`  
  - Role: Entry point for the sub-workflow; receives `query` input from main workflow.  
  - Configuration: Defines `query` as required input parameter.  
  - Input: `query` string from main workflow’s tool node.  
  - Output: Forwards `query` to Brave Search AI Agent node.  
  - Edge cases: Missing or empty `query`, malformed input.

- **Brave Search AI Agent**  
  - Type: `@n8n/n8n-nodes-langchain.agent`  
  - Role: Core AI logic using Google Gemini LLM; interprets query, decides search type, orchestrates tool usage, summarizes results.  
  - Configuration:  
    - Uses `models/gemini-2.5-flash-preview-05-20` model.  
    - System prompt instructs the AI to choose between `brave_web_search` and `brave_local_search` based on query intent (local vs. general).  
    - Tool integrations: `brave_web_search` and `brave_local_search` MCP client nodes.  
    - Memory: Linked to `Simple Memory` node with session key `{{ $execution.id }}` to maintain conversational context without cross-talk.  
    - Final output: Summarized and user-friendly answer.  
  - Key Expressions: `text` input taken from `$('Brave Search Workflow Start').item.json.query`.  
  - Edge cases: LLM errors, malformed tool parameters, tool call failures, no results returned, ambiguous queries causing fallback or clarification needed.  
  - Version: Uses agent type version 2.

- **Simple Memory**  
  - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  - Role: Maintains sliding window conversational memory unique per execution.  
  - Configuration: SessionKey set to current execution id (`{{ $execution.id }}`).  
  - Input: Receives AI agent’s conversation history.  
  - Output: Provides memory context back to AI agent node.  
  - Edge cases: Memory overflow or loss if execution id changes unexpectedly.

- **brave_web_search**  
  - Type: `n8n-nodes-mcp.mcpClientTool`  
  - Role: Calls Brave Search’s web search MCP tool via Smithery credentials.  
  - Configuration:  
    - Uses credential `smithery brave search`.  
    - Accepts JSON parameters with keys:  
      - `query` (string, required)  
      - `count` (number, optional, max 20)  
      - `offset` (number, optional, max 9)  
    - AI generates these params dynamically.  
  - Input: Parameters from AI Agent.  
  - Output: Web search results back to AI agent.  
  - Edge cases: API errors, invalid parameters, rate limits, empty results.

- **brave_local_search**  
  - Type: `n8n-nodes-mcp.mcpClientTool`  
  - Role: Calls Brave Search’s local search MCP tool via Smithery credentials.  
  - Configuration:  
    - Uses same credential `smithery brave search`.  
    - Parameters:  
      - `query` (string, required)  
      - `count` (number, optional, max 20)  
    - No `offset` parameter supported.  
    - May fallback automatically to web search if no local results found.  
  - Input: Parameters from AI Agent.  
  - Output: Local search (or fallback web) results to AI agent.  
  - Edge cases: API failures, no local results fallback, invalid parameters.

---

#### 2.3 Documentation and Configuration Notes (Sticky Notes)

**Overview:**  
Multiple sticky note nodes provide essential information to developers and operators about configuration, client integration, troubleshooting, and testing.

**Key Sticky Notes:**

- MCP Client Configuration example for Roo Code and Cline clients (node "Sticky Note").  
- MCP Server Trigger description and usage (node "Sticky Note1").  
- Tool Workflow explanation for `call_brave_search_agent` (node "Sticky Note2").  
- Main MCP Server Workflow overview and data flow diagram (node "Sticky Note3").  
- Sub-Workflow start and input expectations (node "Sticky Note4").  
- Brave Search AI Agent node explanation including system prompt and tool usage (node "Sticky Note5").  
- Simple Memory purpose and session key details (node "Sticky Note6").  
- Details on MCP client tools `brave_web_search` and `brave_local_search` (nodes "Sticky Note7" and "Sticky Note8").  
- Rationale for this AI agent MCP pattern and benefits (node "Sticky Note9").  
- Testing instructions for MCP endpoint using curl/Postman (node "Sticky Note10").  
- Example prompt/response screenshots for Roo Code integration (nodes "Sticky Note11" and "Sticky Note17").  
- Core LLM and agent settings overview (node "Sticky Note12").  
- Sub-workflow purpose and sample inputs (node "Sticky Note13").  
- Troubleshooting tips for AI agent and tool calls (node "Sticky Note14").  
- Workflow metadata and author info (node "Sticky Note15").  
- TODO and future enhancements list (node "Sticky Note16").

---

### 3. Summary Table

| Node Name                     | Node Type                                   | Functional Role                            | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                  |
|-------------------------------|---------------------------------------------|--------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Brave Search MCP Server Trigger| @n8n/n8n-nodes-langchain.mcpTrigger         | Public entry point SSE for MCP clients     |                                  | call_brave_search_agent          | Explains MCP Server Trigger usage, path, inputs/outputs, and client integration details (Sticky Note1)       |
| call_brave_search_agent        | @n8n/n8n-nodes-langchain.toolWorkflow       | Tool node exposing AI Agent sub-workflow  | Brave Search MCP Server Trigger  | Sub-workflow (Brave Search AI Agent) | Describes as callable tool for MCP clients, passes user query to sub-workflow (Sticky Note2)                  |
| Brave Search Workflow Start    | n8n-nodes-base.executeWorkflowTrigger       | Sub-workflow entry point, receives queries| call_brave_search_agent (via sub-workflow) | Brave Search AI Agent          | Describes role as sub-workflow start, input expectations (Sticky Note4)                                      |
| Brave Search AI Agent          | @n8n/n8n-nodes-langchain.agent               | Core AI logic using Google Gemini model   | Brave Search Workflow Start, Simple Memory | brave_web_search, brave_local_search | Details AI model, system prompt, tool orchestration, memory usage (Sticky Note5)                             |
| Simple Memory                 | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context          | Brave Search AI Agent            | Brave Search AI Agent            | Explains memory type and session key use (Sticky Note6)                                                      |
| brave_web_search              | n8n-nodes-mcp.mcpClientTool                  | Calls Brave Web Search MCP tool            | Brave Search AI Agent            | Brave Search AI Agent            | Details tool parameters, credential, and purpose for web search (Sticky Note7)                               |
| brave_local_search            | n8n-nodes-mcp.mcpClientTool                  | Calls Brave Local Search MCP tool          | Brave Search AI Agent            | Brave Search AI Agent            | Details tool parameters, credential, fallback behavior, and usage notes (Sticky Note8)                        |
| Sticky Note                   | n8n-nodes-base.stickyNote                     | Documentation and configuration notes      |                                  |                                 | MCP Client config example for Roo Code/Cline clients (Sticky Note)                                           |
| Sticky Note1                  | n8n-nodes-base.stickyNote                     | Documentation for MCP Server Trigger        |                                  |                                 | MCP Server Trigger details (Sticky Note1)                                                                    |
| Sticky Note2                  | n8n-nodes-base.stickyNote                     | Documentation for call_brave_search_agent  |                                  |                                 | Tool Workflow node explanation (Sticky Note2)                                                                |
| Sticky Note3                  | n8n-nodes-base.stickyNote                     | Workflow overview and dataflow              |                                  |                                 | Main MCP Server Workflow overview and client data flow (Sticky Note3)                                        |
| Sticky Note4                  | n8n-nodes-base.stickyNote                     | Sub-Workflow start and input expectations  |                                  |                                 | Sub-workflow entry and input (Sticky Note4)                                                                   |
| Sticky Note5                  | n8n-nodes-base.stickyNote                     | AI Agent core logic description             |                                  |                                 | AI Agent node description and system prompt details (Sticky Note5)                                           |
| Sticky Note6                  | n8n-nodes-base.stickyNote                     | Memory usage and settings                    |                                  |                                 | Simple Memory explanation (Sticky Note6)                                                                      |
| Sticky Note7                  | n8n-nodes-base.stickyNote                     | brave_web_search MCP client details         |                                  |                                 | Web search tool usage and parameters (Sticky Note7)                                                          |
| Sticky Note8                  | n8n-nodes-base.stickyNote                     | brave_local_search MCP client details       |                                  |                                 | Local search tool usage and fallback explanation (Sticky Note8)                                              |
| Sticky Note9                  | n8n-nodes-base.stickyNote                     | Pattern rationale and benefits               |                                  |                                 | Explains why this AI agent MCP pattern is useful (Sticky Note9)                                              |
| Sticky Note10                 | n8n-nodes-base.stickyNote                     | Testing instructions for MCP endpoint       |                                  |                                 | How to test MCP endpoint externally (Sticky Note10)                                                          |
| Sticky Note11                 | n8n-nodes-base.stickyNote                     | Example prompt/response screenshot           |                                  |                                 | Roo Code example screenshot (Sticky Note11)                                                                   |
| Sticky Note12                 | n8n-nodes-base.stickyNote                     | Key LLM and agent configuration notes       |                                  |                                 | Core model and prompt configuration notes (Sticky Note12)                                                    |
| Sticky Note13                 | n8n-nodes-base.stickyNote                     | Sub-workflow purpose and input examples     |                                  |                                 | Sub-workflow input and purpose (Sticky Note13)                                                                |
| Sticky Note14                 | n8n-nodes-base.stickyNote                     | Troubleshooting tips                          |                                  |                                 | Troubleshooting advice for sub-workflow/operator (Sticky Note14)                                             |
| Sticky Note15                 | n8n-nodes-base.stickyNote                     | Workflow metadata and author                  |                                  |                                 | Workflow info: version, author, last updated (Sticky Note15)                                                 |
| Sticky Note16                 | n8n-nodes-base.stickyNote                     | TODO and future enhancement ideas             |                                  |                                 | Planned improvements and enhancements (Sticky Note16)                                                         |
| Sticky Note17                 | n8n-nodes-base.stickyNote                     | Example raw output screenshot                   |                                  |                                 | Raw output example from Brave Search MCP for Roo Code client (Sticky Note17)                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Main MCP Server Workflow**  
   - Create a new workflow named "Brave Search Smithery AI Agent MCP Server".  

2. **Add MCP Server Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set webhook path to a unique ID (e.g., `cc8cc827-3e72-4029-8a9d-76519d1c136d`).  
   - Configure it to expose the tool named `call_brave_search_agent`.  
   - No credentials needed here.  

3. **Add Tool Workflow Node `call_brave_search_agent`**  
   - Node Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Set to call a separate sub-workflow (to be built next).  
   - Configure input parameter `query` as string, mapped from MCP Trigger input.  
   - This node acts as a bridge between MCP Trigger and AI Agent sub-workflow.

4. **Create the Sub-Workflow: Brave Search AI Agent MCP Server**  
   - Create new workflow with ID `QclY1XQ8GEstkvk7` or any name.  
   - Add **Execute Workflow Trigger** node as entry point.  
     - Define input parameter `query` (string).  

5. **Add AI Agent Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Configure:  
     - Use language model node: Google Gemini Chat Model with model name `models/gemini-2.5-flash-preview-05-20`.  
     - System prompt instructs to analyze query intent and decide between two tools `brave_web_search` and `brave_local_search`.  
     - Input text is the `query` from trigger node (`={{ $('Brave Search Workflow Start').item.json.query }}`).  
   - Connect to memory and tools as below.

6. **Add Simple Memory Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Set session key to `{{ $execution.id }}` to isolate memory per execution.  
   - Connect this memory buffer to the AI Agent node’s memory input.

7. **Add Google Gemini Chat Model Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Configure modelName as `models/gemini-2.5-flash-preview-05-20`.  
   - Connect as AI language model for the Agent node.  

8. **Add MCP Client Tool Nodes**  
   - **brave_web_search**  
     - Node Type: `n8n-nodes-mcp.mcpClientTool`  
     - Tool Name: `brave_web_search`  
     - Operation: `executeTool`  
     - Set credentials: Use valid Smithery Brave Search MCP HTTP API credential.  
     - Tool parameters: JSON with keys `query` (string, required), optional `count` (max 20), optional `offset` (max 9).  
     - Connect output back to AI Agent as a tool response.  

   - **brave_local_search**  
     - Node Type: `n8n-nodes-mcp.mcpClientTool`  
     - Tool Name: `brave_local_search`  
     - Operation: `executeTool`  
     - Use same Smithery Brave Search credential.  
     - Tool parameters: JSON with keys `query` (string, required), optional `count` (max 20). No `offset`.  
     - Connect output back to AI Agent as a tool response.  

9. **Connect Nodes in Sub-Workflow**  
   - Execute Workflow Trigger → Brave Search AI Agent  
   - Simple Memory linked to AI Agent memory input/output.  
   - Google Gemini Chat Model linked as AI language model.  
   - brave_web_search and brave_local_search linked as AI agent tools.  

10. **Configure Credentials**  
    - Create or import Smithery Brave Search MCP HTTP API credentials with required access tokens or API keys.  
    - Create or import Google Palm API credentials for Google Gemini model.  

11. **Connect Main Workflow to Sub-Workflow**  
    - From `call_brave_search_agent` tool workflow node, set workflow ID to new sub-workflow.  
    - Map input `query` from MCP Trigger node input.  

12. **Test Workflow**  
    - Deploy the main workflow.  
    - Use an MCP client (Roo Code, Postman) to POST JSON with `{"input":{"query":"your test query"}}` to URL:  
      `https://YOUR_N8N_INSTANCE/mcp/cc8cc827-3e72-4029-8a9d-76519d1c136d/sse`  
    - Observe streamed AI responses.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| MCP client configuration example JSON for Roo Code, Cline, etc., showing how to connect to this Brave Search AI Agent MCP server endpoint.                    | Sticky Note content with example client config snippet.                                                         |
| Video explaining MCP agent pattern and usage: [MCP Agent Video](https://youtu.be/dudvmyp7Pyg?si=N59DYTe2WE6GPlvf)                                              | Useful for understanding the multi-agent collaboration approach.                                               |
| Troubleshooting advice: Check AI Agent logs, verify MCP client tool inputs/outputs, test external Smithery Brave Search MCP API access, check credentials.       | Sticky Note14 content for operational diagnostics.                                                              |
| Workflow authored by Jeremy Dawes, Jezweb; last updated 2025-06-01; official site: [www.jezweb.com.au](https://www.jezweb.com.au/)                            | Metadata and credit info (Sticky Note15).                                                                        |
| Example screenshots of Roo Code client prompt/response and raw output display from Brave Search MCP are available for reference and UI integration insights.   | Sticky Notes11 and 17.                                                                                            |
| Planned future enhancements include adding detailed error handling, caching common queries, refining system prompts for ambiguous queries, and freshness params.| Sticky Note16.                                                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, conforming strictly to all applicable content policies. It contains no illegal, offensive, or protected content. All processed data is legal and public.