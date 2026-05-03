Build a Local AI Assistant with Llama 3.2, RAG, and Search using Ollama & MCP

https://n8nworkflows.xyz/workflows/build-a-local-ai-assistant-with-llama-3-2--rag--and-search-using-ollama---mcp-5398


# Build a Local AI Assistant with Llama 3.2, RAG, and Search using Ollama & MCP

### 1. Workflow Overview

This workflow builds a Local AI Assistant leveraging Llama 3.2, Retrieval-Augmented Generation (RAG), and search capabilities using Ollama and MCP (Multi-Channel Processing) clients. It targets use cases requiring conversational AI with access to both a knowledge base (via RAG) and up-to-date information retrieval (via a search engine). The assistant intelligently routes queries to appropriate tools depending on the context, enabling dynamic, context-aware responses.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Captures incoming chat messages triggering the assistant.
- **1.2 AI Agent Processing:** Core intelligent agent that decides which tools or data sources to use for answering queries.
- **1.3 Language Model:** The local Ollama Llama 3.2 chat model used for natural language understanding and generation.
- **1.4 Memory Management:** Maintains conversational context via a sliding window memory buffer.
- **1.5 External Tools Integration:** Connects to MCP clients for RAG database querying and search engine operations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages to start the AI assistant workflow.

- **Nodes Involved:**  
  - `When chat message received`

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Entry point webhook that activates the workflow when a chat message is received.  
    - *Configuration:* Default options, no special filters or conditions.  
    - *Input:* External chat message (via webhook).  
    - *Output:* Passes message data to the AI Agent node for processing.  
    - *Version:* 1.1  
    - *Failure cases:* Webhook connectivity failure or malformed chat payloads may cause trigger failure.

#### 2.2 AI Agent Processing

- **Overview:**  
  Central decision-making node implementing the AI assistant logic. Routes queries to either the RAG database or the search engine MCP client based on the type of question (general knowledge vs current events). Uses a system prompt to define behavior.

- **Nodes Involved:**  
  - `AI Agent`

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Processes input messages, manages tool usage, and generates the assistant’s responses.  
    - *Configuration:* Uses a system message instructing the agent to use two MCP servers: one connected to a RAG database and one to a search engine. It explicitly directs to use search for current events and RAG for general queries.  
    - *Key Expression:* System prompt text specifying tool selection logic.  
    - *Input:* Receives the chat message from the trigger node and conversational memory.  
    - *Output:* Sends commands to language model and MCP tools for execution.  
    - *Version:* 2  
    - *Edge cases:* Misclassification of queries can lead to suboptimal tool usage; network issues with MCP clients may cause tool invocation failures.

#### 2.3 Language Model

- **Overview:**  
  Provides natural language understanding and generation capabilities using a local Llama 3.2 model served by Ollama.

- **Nodes Involved:**  
  - `Ollama Chat Model`

- **Node Details:**  
  - **Ollama Chat Model**  
    - *Type:* LangChain Ollama Chat Model  
    - *Role:* Generates and processes chat completions locally using the Llama 3.2 model.  
    - *Configuration:* Connected to a local Ollama service instance via stored API credentials. No extra options configured.  
    - *Input:* Receives prompts and context from the AI Agent.  
    - *Output:* Returns generated chat completions to the AI Agent.  
    - *Version:* 1  
    - *Failure cases:* Local Ollama service downtime, credential misconfiguration, or model loading issues.

#### 2.4 Memory Management

- **Overview:**  
  Maintains a conversation buffer to provide context continuity across chat turns.

- **Nodes Involved:**  
  - `Simple Memory`

- **Node Details:**  
  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Stores a sliding window of recent conversation messages to maintain context for the AI Agent.  
    - *Configuration:* Default buffer window, no custom size set.  
    - *Input:* Receives new chat messages and AI responses.  
    - *Output:* Feeds memory context back into the AI Agent node.  
    - *Version:* 1.3  
    - *Edge cases:* Buffer overflow or memory size misconfigurations could lead to loss of important context.

#### 2.5 External Tools Integration

- **Overview:**  
  Interfaces with two MCP client tools: one for querying the RAG knowledge database and another for executing external search engine operations via Bright Data.

- **Nodes Involved:**  
  - `MCP Client: RAG`  
  - `MCP Client:BD_Tools`  
  - `MCP Client:BD_Execute`

- **Node Details:**  

  - **MCP Client: RAG**  
    - *Type:* LangChain MCP Client Tool  
    - *Role:* Queries the RAG database for general knowledge questions.  
    - *Configuration:* Uses a local SSE endpoint URL for connection.  
    - *Input:* Receives tool invocation requests from AI Agent.  
    - *Output:* Returns retrieved documents or answers to AI Agent.  
    - *Version:* 1  
    - *Failure cases:* Endpoint unreachable, timeouts, or malformed query parameters.

  - **MCP Client:BD_Tools**  
    - *Type:* MCP Client Tool  
    - *Role:* Interface node representing Bright Data tools. Acts as a tool provider for the AI Agent.  
    - *Configuration:* Uses MCP client API credentials for authentication.  
    - *Input:* Receives tool requests from AI Agent.  
    - *Output:* Passes execution commands downstream.  
    - *Version:* 1  
    - *Failure cases:* Authentication errors, API limits.

  - **MCP Client:BD_Execute**  
    - *Type:* MCP Client Tool  
    - *Role:* Executes the search engine tool operation, using parameters dynamically injected via AI override expressions.  
    - *Configuration:* Tool name explicitly set to "search_engine" with operation "executeTool". Tool parameters are taken from AI-generated JSON expressions.  
    - *Input:* Triggered by AI Agent tool requests routed from MCP Client:BD_Tools.  
    - *Output:* Returns search engine results back to AI Agent.  
    - *Version:* 1  
    - *Failure cases:* Execution errors, malformed parameters, or API timeouts.

- **Sticky Notes:**  
  - MCP Client: RAG node has a sticky note titled "## MCP Client: RAG".  
  - MCP Client:BD_Tools and MCP Client:BD_Execute nodes share a sticky note titled "## MCP Client: Bright Data" with wider layout, emphasizing their grouping.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                               | Input Node(s)               | Output Node(s)            | Sticky Note               |
|-------------------------|--------------------------------|-----------------------------------------------|-----------------------------|---------------------------|---------------------------|
| When chat message received | LangChain Chat Trigger          | Entry point for incoming chat messages       | —                           | AI Agent                  |                           |
| AI Agent                | LangChain Agent                | Core AI decision-making and routing           | When chat message received, Simple Memory, Ollama Chat Model, MCP Clients | Ollama Chat Model, MCP Clients |                           |
| Ollama Chat Model       | LangChain Ollama Chat Model    | Local LLM for natural language processing     | AI Agent                    | AI Agent                  |                           |
| Simple Memory           | LangChain Memory Buffer Window | Maintains conversational context              | —                           | AI Agent                  |                           |
| MCP Client: RAG         | LangChain MCP Client Tool      | Queries RAG knowledge database                 | AI Agent                    | AI Agent                  | ## MCP Client: RAG        |
| MCP Client:BD_Tools     | MCP Client Tool                | Provides Bright Data tools interface            | AI Agent                    | MCP Client:BD_Execute     | ## MCP Client: Bright Data|
| MCP Client:BD_Execute   | MCP Client Tool                | Executes search engine tool operation           | MCP Client:BD_Tools         | AI Agent                  | ## MCP Client: Bright Data|

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **LangChain Chat Trigger** node named `When chat message received`.  
   - Leave default webhook settings. This node will start the workflow on incoming chat messages.

2. **Add the AI Agent Node:**  
   - Add a **LangChain Agent** node named `AI Agent`.  
   - Connect the output of `When chat message received` to the input of `AI Agent`.  
   - Configure the agent’s system message with the text:  
     ```
     You are a helpful assistant. You have access to two MCP Servers. One which has access to a RAG Database, and one which has access to a tool to search google.

     When you get a question about current events, you use the search engine MCP Server to fetch information.

     When people ask more general questions, you use your RAG Database.
     ```

3. **Add the Ollama Chat Model Node:**  
   - Add a **LangChain Ollama Chat Model** node named `Ollama Chat Model`.  
   - Configure credentials using your local Ollama API credential (e.g., point to your local Ollama LLM service with valid API key).  
   - Connect the `ai_languageModel` input of `AI Agent` to this node’s output.

4. **Add the Memory Buffer Node:**  
   - Add a **LangChain Memory Buffer Window** node named `Simple Memory`.  
   - Use default parameters for buffer size and window.  
   - Connect its output `ai_memory` to the corresponding input on the `AI Agent`.

5. **Add the MCP Client for RAG:**  
   - Add a **LangChain MCP Client Tool** node named `MCP Client: RAG`.  
   - Set the SSE endpoint URL to your local or remote RAG database MCP server, e.g., `http://localhost:5678/mcp/8d7910ab-f0db-4042-9da9-f580647a8a8e`.  
   - Connect the `ai_tool` input of `AI Agent` to this node.

6. **Add the MCP Client for Bright Data Tools:**  
   - Add an **MCP Client Tool** node named `MCP Client:BD_Tools`.  
   - Configure credentials with your MCP Client API credentials (OAuth or API key).  
   - Connect the `ai_tool` input of `AI Agent` to this node.

7. **Add the MCP Client to Execute Search Tool:**  
   - Add an **MCP Client Tool** node named `MCP Client:BD_Execute`.  
   - Set parameters as follows:  
     - Tool Name: `"search_engine"`  
     - Operation: `"executeTool"`  
     - Tool Parameters: Use an expression that pulls parameters dynamically from AI override like:  
       ```
       {{$fromAI('Tool_Parameters', ``, 'json')}}
       ```  
   - Use the same MCP Client API credentials as `MCP Client:BD_Tools`.  
   - Connect `ai_tool` output of `MCP Client:BD_Tools` to `MCP Client:BD_Execute`.  
   - Connect `ai_tool` output of `MCP Client:BD_Execute` back to `AI Agent`.

8. **Add Sticky Notes for Documentation (Optional):**  
   - Create a sticky note near `MCP Client: RAG` titled "## MCP Client: RAG".  
   - Create a sticky note spanning `MCP Client:BD_Tools` and `MCP Client:BD_Execute` titled "## MCP Client: Bright Data".

9. **Validate Connections:**  
   - Ensure the flow is:  
     `When chat message received` → `AI Agent`  
     `Simple Memory` → `AI Agent` (ai_memory)  
     `Ollama Chat Model` → `AI Agent` (ai_languageModel)  
     `MCP Client: RAG` → `AI Agent` (ai_tool)  
     `MCP Client:BD_Tools` → `MCP Client:BD_Execute` → `AI Agent` (ai_tool)

10. **Configure Credentials:**  
    - Set up credentials for:  
      - Ollama API (local Llama 3.2 service)  
      - MCP Client API (for Bright Data tools)  
    - Ensure local MCP server for RAG is reachable at configured SSE endpoint.

11. **Test the Workflow:**  
    - Send a chat message to the webhook URL exposed by `When chat message received`.  
    - Verify that the AI Agent correctly routes queries to RAG or Search MCP clients.  
    - Confirm that Ollama model responses are generated accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                             |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow leverages local Llama 3.2 model served via Ollama for privacy and performance.      | Ollama local LLM service                                     |
| MCP clients provide modular integration with RAG and external search tools via API endpoints. | MCP (Multi-Channel Processing) architecture                  |
| System prompt guides AI agent decision-making by distinguishing current events vs general queries. | Critical for accurate tool routing                            |
| Sticky notes visually group MCP client nodes for clarity.                                   | Workflow documentation aid                                   |

---

**Disclaimer:**  
The provided description and workflow stem solely from an automated n8n workflow generation. It respects all applicable content policies and includes only lawful, public data sources.