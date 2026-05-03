💥🛠️Build a Web Search Chatbot with GPT-4o and MCP Brave Search

https://n8nworkflows.xyz/workflows/-----build-a-web-search-chatbot-with-gpt-4o-and-mcp-brave-search-3189


# 💥🛠️Build a Web Search Chatbot with GPT-4o and MCP Brave Search

### 1. Workflow Overview

This workflow implements a web search chatbot leveraging GPT-4o and MCP Brave Search integration within n8n. It is designed to receive user chat messages, process them with an AI agent, retrieve and execute web search tools via Brave Search, and maintain short-term conversational memory. The workflow is ideal for developers and businesses wanting to embed AI-powered chat capabilities enhanced by real-time web search results.

The workflow’s logic is organized into the following functional blocks:

- **1.1 Input Reception:** Captures incoming chat messages via a chat trigger node.
- **1.2 AI Processing:** Uses a GPT-4o powered AI Agent to interpret user queries and orchestrate tool usage.
- **1.3 Tool Retrieval:** Fetches available Brave Search tools through the MCP Get Brave Tools node.
- **1.4 Tool Execution:** Executes specific Brave Search queries using the MCP Execute Brave Search node.
- **1.5 Memory Management:** Maintains short-term chat memory to provide conversational context.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages from users and triggers the workflow to start processing.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: Chat Trigger node (Langchain)  
    - Role: Entry point for chat messages; listens for incoming user inputs via webhook.  
    - Configuration: Default options, webhook ID assigned for external chat interface integration.  
    - Inputs: External chat messages via webhook.  
    - Outputs: Passes received messages to the AI Agent node.  
    - Edge Cases: Webhook misconfiguration, network issues, or malformed incoming messages may cause failures.  
    - Version: 1.1

#### 2.2 AI Processing

- **Overview:**  
  Processes the user input using a GPT-4o language model wrapped in an AI Agent node. This node also coordinates tool usage and memory integration.

- **Nodes Involved:**  
  - AI Agent  
  - gpt-4o

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Central orchestrator that processes chat input, decides on tool usage, and generates AI responses.  
    - Configuration: Uses default options; integrates with language model, memory, and tools via connections.  
    - Inputs: Receives chat messages from the trigger node.  
    - Outputs: Sends instructions to MCP nodes and returns final AI-generated chat responses.  
    - Expressions: Uses dynamic expressions to pass tool names and parameters to MCP Execute Brave Search node.  
    - Edge Cases: AI model API errors, expression evaluation failures, or tool invocation errors.  
    - Version: 1.8

  - **gpt-4o**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides the GPT-4o language model for the AI Agent.  
    - Configuration: Model set explicitly to "gpt-4o"; uses OpenAI API credentials.  
    - Inputs: Receives prompts from AI Agent.  
    - Outputs: Returns AI-generated text responses.  
    - Edge Cases: API key invalidation, rate limits, or network timeouts.  
    - Version: 1.2

#### 2.3 Tool Retrieval

- **Overview:**  
  Retrieves the list of available Brave Search tools from MCP to enable the AI Agent to select appropriate tools for query execution.

- **Nodes Involved:**  
  - MCP Get Brave Tools

- **Node Details:**  
  - **MCP Get Brave Tools**  
    - Type: MCP Client Tool node  
    - Role: Fetches available tools from MCP related to Brave Search.  
    - Configuration: Uses MCP Client API credentials; no additional parameters set.  
    - Inputs: Triggered by AI Agent’s tool request.  
    - Outputs: Provides tool metadata for AI Agent to select.  
    - Edge Cases: API authentication failure, MCP service downtime, or malformed responses.  
    - Version: 1  
    - Credentials: MCP Client API (STDIO) account configured.

#### 2.4 Tool Execution

- **Overview:**  
  Executes specific Brave Search queries as instructed by the AI Agent, using the selected tool and parameters.

- **Nodes Involved:**  
  - MCP Execute Brave Search

- **Node Details:**  
  - **MCP Execute Brave Search**  
    - Type: MCP Client Tool node  
    - Role: Executes the Brave Search tool with parameters dynamically provided by the AI Agent.  
    - Configuration:  
      - Tool name is dynamically set via expression from AI Agent output (`$fromAI('tool', ...)`).  
      - Tool parameters are JSON parsed from AI Agent output (`$fromAI('Tool_Parameters', ...)`).  
      - Operation set to "executeTool".  
    - Inputs: Receives tool name and parameters from AI Agent.  
    - Outputs: Returns search results to AI Agent for response generation.  
    - Edge Cases: Invalid tool names, malformed parameters, API errors, or timeouts.  
    - Version: 1  
    - Credentials: MCP Client API (STDIO) account configured.

#### 2.5 Memory Management

- **Overview:**  
  Maintains a short-term conversational memory buffer to provide context for ongoing chat interactions.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**  
  - **Simple Memory**  
    - Type: Langchain Memory Buffer Window node  
    - Role: Stores recent conversation history to enrich AI responses with context.  
    - Configuration: Default buffer window settings; no custom parameters set.  
    - Inputs: Receives chat messages and AI responses.  
    - Outputs: Provides memory context to AI Agent.  
    - Edge Cases: Memory overflow if buffer size is too small or too large; potential data loss on node failure.  
    - Version: 1.3

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                      | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                  |
|-------------------------|----------------------------------|------------------------------------|-------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| When chat message received | Chat Trigger (Langchain)          | Input Reception (chat message)     | —                       | AI Agent                  |                                                                                              |
| AI Agent                | Langchain Agent                   | AI Processing & orchestration      | When chat message received, gpt-4o, Simple Memory, MCP Get Brave Tools, MCP Execute Brave Search | MCP Get Brave Tools, MCP Execute Brave Search, Simple Memory, gpt-4o |                                                                                              |
| gpt-4o                  | Langchain OpenAI Chat Model       | Cloud LLM (GPT-4o)                 | AI Agent (ai_languageModel) | AI Agent                  | ## Cloud LLM                                                                                 |
| MCP Get Brave Tools     | MCP Client Tool                   | Retrieve Brave Search tools        | AI Agent (ai_tool)       | AI Agent                  | ## 1️⃣ MCP Get Brave Tools                                                                   |
| MCP Execute Brave Search | MCP Client Tool                   | Execute Brave Search queries       | AI Agent (ai_tool)       | AI Agent                  | ## 2️⃣ MCP Execute Brave Search                                                              |
| Simple Memory           | Langchain Memory Buffer Window    | Short-term chat memory             | AI Agent (ai_memory)     | AI Agent                  | ## Short Term Chat Memory                                                                     |
| Sticky Note             | Sticky Note                      | Informational                      | —                       | —                         | # 💥🛠️Your First Simple MCP AI Chatbot using Brave Search https://github.com/nerding-io/n8n-nodes-mcp https://brave.com/search/api/ |
| Sticky Note1            | Sticky Note                      | Informational                      | —                       | —                         | ### **Who is this for?** This workflow is ideal for developers, automation enthusiasts, and businesses looking to integrate AI-powered chat capabilities into their workflows. It's particularly useful for those leveraging Brave Search and MCP tools to enhance user interactions and streamline data retrieval. ### **What problem is this workflow solving?** This workflow addresses the challenge of creating an intelligent chatbot that can process user queries, execute searches using Brave Search, and provide responses enriched by AI. It simplifies the integration of multiple tools into a cohesive system, saving time and effort for users who need a robust conversational AI solution. ### **What this workflow does** - Listens for incoming chat messages using the **Chat Trigger** node. - Processes user input with an **AI Agent** powered by GPT-4o. - Retrieves relevant tools using the **MCP Get Brave Tools** node. - Executes specific search queries via the **MCP Execute Brave Search** node. - Maintains short-term memory of conversations with the **Simple Memory** node. ### **Setup** 1. **Prerequisites**: - Access to an n8n instance (self-hosted). - API credentials for OpenAI and MCP Client Tools. - Brave Search API key. 2. **Steps**: - Import the workflow JSON into your n8n instance. - Configure the API credentials for OpenAI and MCP Client Tools in their respective nodes. - Set up your Brave Search API key in the MCP nodes. https://brave.com/search/api/ 3. **Testing**: - Use the built-in chat interface to send test messages. - Verify that the chatbot processes queries and returns results as expected. ### **How to customize this workflow to your needs** - Modify the AI Agent's prompt settings to tailor responses to your specific use case. - Adjust the memory buffer in the Simple Memory node to retain more or less conversational context. - Replace or add additional tools in the MCP nodes to expand functionality. |
| Sticky Note2            | Sticky Note                      | Informational                      | —                       | —                         | ## 🤖 AI Agent with Tools                                                                     |
| Sticky Note3            | Sticky Note                      | Informational                      | —                       | —                         | ## 2️⃣ MCP Execute Brave Search                                                              |
| Sticky Note4            | Sticky Note                      | Informational                      | —                       | —                         | ## Short Term Chat Memory                                                                     |
| Sticky Note5            | Sticky Note                      | Informational                      | —                       | —                         | ## Cloud LLM                                                                                 |
| Sticky Note6            | Sticky Note                      | Informational                      | —                       | —                         | # 💥🛠️Your First Simple MCP AI Chatbot using Brave Search https://github.com/nerding-io/n8n-nodes-mcp https://brave.com/search/api/ |
| Sticky Note7            | Sticky Note                      | Informational                      | —                       | —                         | ## 🛠️ MCP Toolbox https://github.com/nerding-io/n8n-nodes-mcp https://brave.com/search/api/     |
| Sticky Note8            | Sticky Note                      | Informational                      | —                       | —                         | ## 👍Try Me!                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a **Chat Trigger** node (Langchain).  
   - Configure it with default options.  
   - Note the webhook URL generated; this will be the entry point for chat messages.

2. **Add the AI Agent Node**  
   - Add an **AI Agent** node (Langchain Agent).  
   - Connect the output of the Chat Trigger node to the AI Agent node’s main input.  
   - Leave options as default initially; this node will orchestrate AI and tool usage.

3. **Add the GPT-4o Language Model Node**  
   - Add a **Langchain OpenAI Chat Model** node.  
   - Set the model to **gpt-4o** explicitly.  
   - Configure OpenAI API credentials (create or select existing).  
   - Connect this node’s output to the AI Agent node’s `ai_languageModel` input.

4. **Add the Simple Memory Node**  
   - Add a **Langchain Memory Buffer Window** node.  
   - Use default buffer settings or adjust as needed for conversational context length.  
   - Connect this node’s output to the AI Agent node’s `ai_memory` input.

5. **Add MCP Get Brave Tools Node**  
   - Add an **MCP Client Tool** node.  
   - Name it "MCP Get Brave Tools".  
   - Configure MCP Client API credentials (STDIO account with Brave Search API key).  
   - No additional parameters needed.  
   - Connect this node’s output to the AI Agent node’s `ai_tool` input.

6. **Add MCP Execute Brave Search Node**  
   - Add another **MCP Client Tool** node.  
   - Name it "MCP Execute Brave Search".  
   - Configure MCP Client API credentials (same as above).  
   - Set parameters:  
     - Tool Name: Use expression `={{ $fromAI('tool', 'Set this with the specific tool name') }}`  
     - Operation: `executeTool`  
     - Tool Parameters: Use expression `={{ $fromAI('Tool_Parameters', ``, 'json') }}`  
   - Connect this node’s output to the AI Agent node’s `ai_tool` input.

7. **Connect AI Agent Outputs**  
   - Connect the AI Agent node’s outputs to the inputs of MCP Get Brave Tools, MCP Execute Brave Search, Simple Memory, and GPT-4o nodes as per their roles:  
     - `ai_tool` to MCP nodes  
     - `ai_memory` to Simple Memory  
     - `ai_languageModel` to GPT-4o

8. **Configure Credentials**  
   - Ensure OpenAI API credentials are set up and linked to the GPT-4o node.  
   - Ensure MCP Client API credentials with Brave Search API key are set up and linked to MCP nodes.

9. **Test the Workflow**  
   - Activate the workflow.  
   - Use the webhook URL from the Chat Trigger node to send test chat messages.  
   - Verify that the AI Agent processes input, retrieves tools, executes searches, and returns enriched responses.

10. **Optional: Add Sticky Notes**  
    - Add sticky notes for documentation and clarity as per the original workflow (optional but recommended for maintainability).

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| 💥🛠️Your First Simple MCP AI Chatbot using Brave Search                                            | https://github.com/nerding-io/n8n-nodes-mcp         |
| Brave Search API documentation and key setup                                                       | https://brave.com/search/api/                        |
| MCP Toolbox for n8n integration                                                                    | https://github.com/nerding-io/n8n-nodes-mcp         |
| This workflow requires a self-hosted n8n instance due to MCP node dependencies                      | —                                                   |
| Customize AI Agent prompts and memory buffer size to tailor chatbot behavior                        | —                                                   |
| Use the built-in chat interface or webhook to test chat message reception and response generation  | —                                                   |

---

This documentation provides a detailed, structured reference to understand, reproduce, and customize the "💥🛠️Build a Web Search Chatbot with GPT-4o and MCP Brave Search" workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and automation agents to work effectively with this chatbot system.