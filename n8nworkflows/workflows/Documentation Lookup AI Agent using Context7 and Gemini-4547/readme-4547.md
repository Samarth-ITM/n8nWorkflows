Documentation Lookup AI Agent using Context7 and Gemini

https://n8nworkflows.xyz/workflows/documentation-lookup-ai-agent-using-context7-and-gemini-4547


# Documentation Lookup AI Agent using Context7 and Gemini

---

### 1. Workflow Overview

This workflow implements an AI-powered Documentation Lookup Agent specialized for software libraries using Context7 and Google Gemini technologies. It targets developers and AI systems seeking detailed software library documentation via natural language queries.

The workflow is structured into two main logical blocks:

- **1.1 MCP Server & Client Interface:**  
  This block exposes a public SSE (Server-Sent Events) endpoint via an MCP (Multi-Agent Collaboration Protocol) server trigger. External clients (such as Roo Code or other AI agents) send their documentation queries here. This block routes the requests to an internal AI Agent tool workflow and streams back AI-generated responses.

- **1.2 AI Agent Sub-Workflow:**  
  This is the core AI processing block. It includes a Google Gemini Chat model-based AI agent that interprets user queries, intelligently calls two Context7 MCP tools (to resolve library IDs and fetch documentation), and maintains conversational context with memory. It orchestrates calls to these tools based on the user’s natural language input to produce precise and contextually relevant documentation answers.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server & Client Interface

- **Overview:**  
  This block provides a secure public entry point for external clients to interact with the AI agent via a long, random SSE endpoint URL. It exposes a tool workflow node that calls the AI Agent Sub-Workflow to process queries.

- **Nodes Involved:**  
  - Context7 MCP Server Trigger  
  - call_context7_ai_agent (Tool Workflow Node)  
  - Context7 Workflow Start (Execute Workflow Trigger)  
  - Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4 (for documentation and explanation)

- **Node Details:**

  - **Context7 MCP Server Trigger**  
    - *Type & Role:* MCP SSE Server Trigger; serves as the public API endpoint.  
    - *Configuration:* Path is a unique, long, random string to act as a simple security measure. Listens for HTTP POST requests carrying JSON payloads with user queries.  
    - *Inputs/Outputs:* Receives JSON input { "input": { "query": "..." } }, streams SSE events back.  
    - *Credentials:* None required for the trigger itself.  
    - *Failure Modes:* Client connection issues, malformed JSON input, SSE stream interruptions.  
    - *Sticky Note1:* Explains the trigger’s role, usage, and path purpose.  
    - *Sub-workflow:* Exposes the `call_context7_ai_agent` tool workflow node.

  - **call_context7_ai_agent**  
    - *Type & Role:* Tool Workflow node; acts as a callable tool from MCP clients.  
    - *Configuration:* Calls the sub-workflow “Context7 Smithery AI Agent MCP Server” (ID: vqujIu65SFbZUz0n) passing the user’s query as input.  
    - *Inputs/Outputs:* Input is `query` string; output is the AI agent’s response stream.  
    - *Credentials:* None directly, but the sub-workflow uses credentials.  
    - *Failure Modes:* Sub-workflow invocation failure, input mapping issues.  
    - *Sticky Note2:* Documents the tool workflow’s role as a bridge to the sub-workflow.  
    - *Sub-workflow:* Calls “Context7 Smithery AI Agent MCP Server” sub-workflow.

  - **Context7 Workflow Start**  
    - *Type & Role:* Execute Workflow Trigger; entry point inside the sub-workflow, receiving the `query` parameter.  
    - *Configuration:* Defines `query` as a workflow input parameter.  
    - *Inputs/Outputs:* Receives query from the main workflow; passes it downstream to the AI Agent node.  
    - *Failure Modes:* Missing or malformed query input.  
    - *Sticky Note4:* Describes this node as the sub-workflow’s input gateway.

#### 2.2 AI Agent Sub-Workflow

- **Overview:**  
  This block processes the user query using a Google Gemini-based AI agent. It uses two MCP tools to resolve library IDs and fetch documentation from Context7. The agent maintains conversational context using a sliding window memory buffer keyed by execution ID.

- **Nodes Involved:**  
  - Context7 AI Agent  
  - Google Gemini Chat Model  
  - context7-resolve-library-id (MCP Client Tool)  
  - context7-get-library-docs (MCP Client Tool)  
  - Simple Memory (Memory Buffer Window)  
  - Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8

- **Node Details:**

  - **Context7 AI Agent**  
    - *Type & Role:* Langchain Agent node; central AI logic orchestrator.  
    - *Configuration:*  
      - Uses the user’s `query` as input.  
      - System message instructs the agent to first resolve library IDs using the `resolve-library-id` tool, then fetch documentation with `get-library-docs`.  
      - Has access to both Context7 tools and the Google Gemini model.  
      - Uses “Simple Memory” node for conversation context, scoped by current execution ID (`{{$execution.id}}`).  
    - *Inputs/Outputs:* Input: user's query; Output: AI-generated response including tool calls and final answer.  
    - *Failure Modes:* Model API errors, tool call errors, malformed tool parameter JSON, memory failures.  
    - *Sticky Note5:* Details the AI agent’s core logic and connections.

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node providing the underlying AI.  
    - *Configuration:* Uses "models/gemini-2.5-flash-preview-05-20" model with default options.  
    - *Credentials:* Google Palm API credentials named "Google Gemini Context7".  
    - *Inputs/Outputs:* Receives prompt and context from AI Agent; returns AI completions.  
    - *Failure Modes:* API authentication failure, rate limits, timeouts.

  - **context7-resolve-library-id**  
    - *Type & Role:* MCP Client Tool node; calls Context7’s “resolve-library-id” tool.  
    - *Configuration:*  
      - Uses `smithery_context7` MCP HTTP credential to connect.  
      - Expects JSON parameter `{"libraryName": "string"}` generated dynamically by the AI agent.  
      - Converts general library names to precise Context7-compatible library IDs.  
    - *Inputs/Outputs:* Input: JSON with libraryName; Output: resolved library ID string.  
    - *Failure Modes:* Credential errors, network issues, invalid library name, missing tool parameters.  
    - *Sticky Note7:* Explains tool purpose and AI interaction.

  - **context7-get-library-docs**  
    - *Type & Role:* MCP Client Tool node; calls Context7’s “get-library-docs” tool.  
    - *Configuration:*  
      - Uses `smithery_context7` MCP HTTP credential.  
      - Expects JSON parameters with keys: `context7CompatibleLibraryID` (required), optional `topic` and `tokens`.  
      - Fetches documentation or focused topic snippets.  
    - *Inputs/Outputs:* Input: JSON with library ID and optional topic/tokens; Output: documentation content.  
    - *Failure Modes:* Credential failure, network errors, invalid library ID, token limit issues.  
    - *Sticky Note8:* Describes usage and integration.

  - **Simple Memory**  
    - *Type & Role:* Memory Buffer Window node; preserves recent conversation history.  
    - *Configuration:* Uses the current execution ID as a session key to isolate conversations.  
    - *Inputs/Outputs:* Connected to Context7 AI Agent to provide conversational context.  
    - *Failure Modes:* Memory overflow, key collisions (unlikely due to unique execution ID).  
    - *Sticky Note6:* Explains memory function and scope.

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                       |
|------------------------------|-----------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Context7 MCP Server Trigger   | MCP SSE Server Trigger             | Public SSE endpoint for client queries          |                             | call_context7_ai_agent       | Explains public SSE endpoint usage, security by obscurity, client input/output format            |
| call_context7_ai_agent        | Tool Workflow                     | Tool node invoking AI Agent sub-workflow        | Context7 MCP Server Trigger |                             | Describes invocation of AI Agent sub-workflow passing user query                               |
| Context7 Workflow Start       | Execute Workflow Trigger          | Sub-workflow entry point, receives query input  |                             | Context7 AI Agent            | Entry point for AI Agent sub-workflow, passes query downstream                                  |
| Context7 AI Agent             | Langchain Agent Node              | Core AI logic orchestrating tool usage          | Context7 Workflow Start      |                             | Details AI logic, system prompt, use of tools and memory                                        |
| Google Gemini Chat Model      | Language Model Node               | Provides AI completions for the agent            | Context7 AI Agent            | Context7 AI Agent            | Specifies model and credentials for AI language generation                                     |
| context7-resolve-library-id   | MCP Client Tool                  | Resolves library name to Context7 library ID    | Context7 AI Agent            | Context7 AI Agent            | Explains tool purpose, credentials, AI-generated JSON parameters                               |
| context7-get-library-docs     | MCP Client Tool                  | Fetches documentation for resolved library ID   | Context7 AI Agent            | Context7 AI Agent            | Describes fetching docs with optional topic and token limits                                   |
| Simple Memory                 | Memory Buffer Window             | Maintains conversational context for AI agent  | Context7 AI Agent            | Context7 AI Agent            | Notes memory usage scoped by execution ID                                                      |
| Sticky Note                  | Sticky Note                      | Documentation and explanations                    |                             |                             | Various nodes have sticky notes explaining role, usage, client config, and system overview      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Server Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Set path to a unique, long random string (e.g., `/dafdfsdfsdf6e8-ea19-43b7-b337-a5iuodsf98vjfewf`)  
   - Purpose: To expose an SSE endpoint that listens for client queries.  
   - No credentials needed here.  

2. **Create the Tool Workflow Node (`call_context7_ai_agent`)**  
   - Node Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
   - Configure it to call a sub-workflow (create next) by specifying its workflow ID or name.  
   - Pass workflow input parameter `query` from the MCP Server Trigger’s input.  
   - Add a descriptive name and description for clarity.  

3. **Create the Sub-Workflow (AI Agent Sub-Workflow)**  
   - Entry node: `Execute Workflow Trigger` with input parameter `query` (string).  
   - This node receives the user’s query from the main workflow.  

4. **Add the `Context7 AI Agent` Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Input text expression: `={{ $json["query"] }}` from the Execute Workflow Trigger node.  
   - System Message: Instruct the agent to:  
     - Use `resolve-library-id` tool first with `libraryName` to get Context7 ID.  
     - Then use `get-library-docs` tool with that ID and optional topic and tokens.  
     - Be precise with tool usage and parameter JSON.  
   - Prompt Type: Set to "define".  
   - Connect two tools (next steps) and memory (below).  

5. **Add Google Gemini Chat Model Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Model Name: `models/gemini-2.5-flash-preview-05-20`  
   - Credentials: Configure Google Palm API credentials (name it e.g. “Google Gemini Context7”)  
   - Connect output to `Context7 AI Agent` languageModel input.  

6. **Add the `context7-resolve-library-id` MCP Client Tool Node**  
   - Node Type: `n8n-nodes-mcp.mcpClientTool`  
   - Tool Name: `resolve-library-id`  
   - Operation: `executeTool`  
   - Connection Type: HTTP  
   - Credentials: Configure MCP HTTP API credentials (e.g., “smithery_context7”)  
   - Tool Parameters: Use an expression to get AI-generated JSON with `libraryName` key (e.g., `={{ $fromAI(...) }}`)  
   - Connect as a tool input to the AI Agent node.  

7. **Add the `context7-get-library-docs` MCP Client Tool Node**  
   - Node Type: `n8n-nodes-mcp.mcpClientTool`  
   - Tool Name: `get-library-docs`  
   - Operation: `executeTool`  
   - Connection Type: HTTP  
   - Credentials: Same MCP HTTP API credentials as above.  
   - Tool Parameters: Use an expression generating JSON with `context7CompatibleLibraryID` plus optional `topic` and `tokens`.  
   - Connect as a tool input to the AI Agent node.

8. **Add the `Simple Memory` Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Session Key: Use expression `={{ $execution.id }}` to isolate session context by execution.  
   - Connect memory input and output to the AI Agent node’s memory ports.  

9. **Wire connections:**  
   - Sub-workflow Start -> Context7 AI Agent  
   - Context7 AI Agent -> Google Gemini Chat Model (languageModel)  
   - Context7 AI Agent -> context7-resolve-library-id (ai_tool)  
   - Context7 AI Agent -> context7-get-library-docs (ai_tool)  
   - Simple Memory -> Context7 AI Agent (ai_memory)  

10. **Configure credentials:**  
    - Google Palm API (Google Gemini): Create and assign credentials with API key for Google Gemini model.  
    - MCP HTTP API: Configure with Context7 service endpoint and authentication for MCP tools.  

11. **Test end-to-end:**  
    - Send a POST request with JSON `{"input":{"query":"How do I use authentication in Next.js?"}}` to the MCP Server Trigger SSE endpoint.  
    - Verify streaming AI responses resolving library IDs and fetching documentation.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| To use this workflow's MCP endpoint in clients like Roo Code or Cline, configure: `"n8n-context7": { "url": "https://n8n.coolify.au/mcp/YOUR_LONG_RANDOM_PATH/sse", "alwaysAllow": ["call_context7_ai_agent"] }`                             | Sticky Note on MCP JSON configuration issued for Roo Code / Cline usage                            |
| The workflow exposes a secure SSE endpoint with basic obscurity by a long random path to reduce unauthorized access.                                                                                                                          | Explanation in Sticky Note1 and Sticky Note3                                                       |
| The sub-workflow AI agent uses a sliding window memory keyed by execution ID to maintain context per user request, preventing cross-talk across sessions.                                                                                   | Sticky Note6                                                                                        |
| The Google Gemini model is used in a preview version ("gemini-2.5-flash-preview-05-20"), which may require appropriate API access and may be subject to Google’s rate limits or availability constraints.                                        | Node configuration and Sticky Note5                                                                |
| Video explanation of the workflow’s design and rationale is available: https://youtu.be/dudvmyp7Pyg?si=N59DYTe2WE6GPlvf                                                                                                                     | Sticky Note3                                                                                        |
| This workflow pattern centralizes AI agent capabilities inside n8n, simplifying client logic and reducing token usage by delegating multi-step reasoning and tool calls internally. Ideal for building modular, scalable AI agent ecosystems. | General workflow design note                                                                        |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly accessible.

---