Get Blockchain Insights from Chat using GPT-4 and Nansen MCP

https://n8nworkflows.xyz/workflows/get-blockchain-insights-from-chat-using-gpt-4-and-nansen-mcp-6303


# Get Blockchain Insights from Chat using GPT-4 and Nansen MCP

### 1. Workflow Overview

This workflow enables real-time blockchain insights retrieval through conversational AI, combining OpenAI’s GPT-4 language capabilities with Nansen’s MCP (Multi-Chain Platform) tool for blockchain data analysis. It is designed for use cases where users send chat messages requesting blockchain-related information, and the system responds with intelligent, data-driven answers. The workflow logic is divided into three main blocks:

- **1.1 Input Reception:** Receives incoming chat messages as triggers.
- **1.2 AI Processing:** Uses an AI Agent to interpret the message, combining GPT-4 with access to the MCP Tool.
- **1.3 Data Retrieval & Response:** The AI Agent leverages the MCP Client node to fetch blockchain data, and the OpenAI Chat Model to generate natural language responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages that trigger the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - **Type:** Chat Trigger (LangChain)  
    - **Configuration:** Listens for chat messages via a webhook with ID `8d13f3c6-4adf-43df-bbd2-a305f5369ec0`. No additional options configured.  
    - **Expressions/Variables:** None configured explicitly.  
    - **Inputs:** External chat messages via webhook.  
    - **Outputs:** Sends chat message data to AI Agent node.  
    - **Version Requirements:** Requires n8n LangChain integration version ≥ 1.1.  
    - **Potential Failures:** Webhook connectivity issues, malformed chat payloads, or missing authentication if configured externally.  
    - **Sub-workflow:** None.

#### 2.2 AI Processing

- **Overview:**  
  Processes the incoming chat message using an AI agent that integrates GPT-4 and the Nansen MCP Tool to understand blockchain queries.

- **Nodes Involved:**  
  - AI Agent

- **Node Details:**

  - **AI Agent**  
    - **Type:** LangChain AI Agent Node  
    - **Configuration:**  
      - System message prompts the agent as a helpful assistant with access to the Nansen MCP Tool for blockchain insights.  
      - Dynamic date inclusion using expression `{{ $now.toISO() }}` to provide context about the current date/time.  
      - Configured to receive inputs from the chat trigger and to invoke the MCP Client and OpenAI Chat Model nodes for tools and language model respectively.  
    - **Expressions/Variables:**  
      - System message uses ISO timestamp expression for “today.”  
    - **Inputs:** From “When chat message received” node.  
    - **Outputs:** Connects bi-directionally with MCP Client (as AI tool) and OpenAI Chat Model (as language model).  
    - **Version Requirements:** Requires LangChain integration version ≥ 2.  
    - **Potential Failures:**  
      - Expression evaluation errors in system message.  
      - API rate limits or authentication failures when calling downstream nodes.  
      - Timeout or improper chaining if MCP Client or OpenAI nodes fail.  
    - **Sub-workflow:** None.

#### 2.3 Data Retrieval & Response

- **Overview:**  
  This block consists of two nodes: MCP Client fetches blockchain data from Nansen’s MCP API, and OpenAI Chat Model generates the final conversational response using GPT-4.

- **Nodes Involved:**  
  - MCP Client  
  - OpenAI Chat Model

- **Node Details:**

  - **MCP Client**  
    - **Type:** LangChain MCP Client Tool Node  
    - **Configuration:**  
      - Endpoint URL set to `https://mcp.nansen.ai/ra/mcp/`.  
      - Authentication via HTTP header with credentials named “Nansen MCP - HS.”  
      - Server transport set to “httpStreamable” for streaming responses.  
    - **Expressions/Variables:** None.  
    - **Inputs:** Invoked as AI tool by AI Agent node.  
    - **Outputs:** Returns blockchain data to AI Agent.  
    - **Version Requirements:** Requires LangChain MCP Client version ≥ 1.1.  
    - **Potential Failures:**  
      - Authentication failure if header token invalid or expired.  
      - Network timeouts or endpoint unavailability.  
      - Streaming errors or malformed responses.  
    - **Sub-workflow:** None.

  - **OpenAI Chat Model**  
    - **Type:** LangChain OpenAI Chat Model Node  
    - **Configuration:**  
      - Uses “gpt-4.1-mini” model variant.  
      - No extra options set.  
      - Credentials linked to “OpenAi account - hs.”  
    - **Expressions/Variables:** None.  
    - **Inputs:** Invoked as AI language model by AI Agent node.  
    - **Outputs:** Provides generated conversational responses back to AI Agent.  
    - **Version Requirements:** LangChain OpenAI integration version ≥ 1.2.  
    - **Potential Failures:**  
      - API key invalid or rate limited.  
      - Model unavailability or version mismatch.  
      - Latency or timeout issues.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role              | Input Node(s)                | Output Node(s)                | Sticky Note                                               |
|-------------------------|--------------------------------|-----------------------------|-----------------------------|------------------------------|-----------------------------------------------------------|
| When chat message received | LangChain Chat Trigger          | Input Reception             | (Webhook external)           | AI Agent                     |                                                           |
| AI Agent                | LangChain AI Agent              | AI Processing (Core Logic)  | When chat message received   | MCP Client, OpenAI Chat Model |                                                           |
| MCP Client              | LangChain MCP Client Tool       | Blockchain Data Retrieval   | AI Agent (ai_tool)           | AI Agent                     |                                                           |
| OpenAI Chat Model       | LangChain OpenAI Chat Model     | Response Generation         | AI Agent (ai_languageModel)  | AI Agent                     |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Name: `When chat message received`  
   - Configure webhook (auto-generated ID) to receive chat messages. No additional options needed.

2. **Create AI Agent Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `AI Agent`  
   - Parameters:  
     - System Message:  
       ```
       You are a helpful assistant with access to Nansen MCP Tool. Nansen MCP tool allows you to understand everything that is happening with the blockchain.

       today: {{ $now.toISO() }}
       ```  
   - Connect input from `When chat message received` main output.

3. **Create MCP Client Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpClientTool`  
   - Name: `MCP Client`  
   - Parameters:  
     - Endpoint URL: `https://mcp.nansen.ai/ra/mcp/`  
     - Authentication: Header Auth  
     - Server Transport: `httpStreamable`  
   - Credentials: Create or select HTTP Header Auth credential with the Nansen MCP API token.  
   - Connect as AI tool from `AI Agent` (ai_tool input).

4. **Create OpenAI Chat Model Node:**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `OpenAI Chat Model`  
   - Parameters:  
     - Model: Select `gpt-4.1-mini`  
     - Options: Leave default  
   - Credentials: Set with valid OpenAI API key (OAuth2 or API key) under “OpenAi account - hs.”  
   - Connect as AI language model from `AI Agent` (ai_languageModel input).

5. **Connect Nodes:**  
   - Connect `When chat message received` → `AI Agent`.  
   - Connect `AI Agent` → `MCP Client` (ai_tool).  
   - Connect `AI Agent` → `OpenAI Chat Model` (ai_languageModel).  
   - Ensure bi-directional communication paths for AI Agent to receive data from MCP Client and OpenAI Chat Model.

6. **Test Workflow:**  
   - Deploy webhook.  
   - Send test chat message payload to webhook endpoint.  
   - Confirm AI Agent processes input, fetches blockchain data via MCP Client, and responds using OpenAI Chat Model.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                          |
|-------------------------------------------------------------------------------------------------|----------------------------------------|
| Nansen MCP tool enables deep blockchain analytics through a rich API, critical for DeFi projects. | https://www.nansen.ai/mcp               |
| GPT-4 model variant “gpt-4.1-mini” used to balance performance and capability.                   | OpenAI API documentation                |
| The system message dynamically includes the current ISO timestamp to provide temporal context.  | n8n expression syntax: `{{ $now.toISO() }}` |
| For authentication, ensure API keys are securely stored in n8n credentials manager.             | n8n documentation on credentials        |

---

*Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.*