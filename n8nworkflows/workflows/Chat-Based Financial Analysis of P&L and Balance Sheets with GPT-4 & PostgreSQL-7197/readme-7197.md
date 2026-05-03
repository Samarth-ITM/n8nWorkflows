Chat-Based Financial Analysis of P&L and Balance Sheets with GPT-4 & PostgreSQL

https://n8nworkflows.xyz/workflows/chat-based-financial-analysis-of-p-l-and-balance-sheets-with-gpt-4---postgresql-7197


# Chat-Based Financial Analysis of P&L and Balance Sheets with GPT-4 & PostgreSQL

---

### 1. Workflow Overview

This workflow, titled **"Chat-Based Financial Analysis of P&L and Balance Sheets with GPT-4 & PostgreSQL"**, is designed to provide conversational financial analysis through natural language chat interactions. It targets financial analysts or business users who want to query Profit & Loss (P&L) reports and Balance Sheet data from a PostgreSQL database using an AI-powered chat interface. The AI agent leverages GPT-4 capabilities to interpret user queries, retrieve relevant financial data, and maintain conversational context.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Captures incoming chat messages from users to initiate the analysis.
- **1.2 AI Processing:** Uses a Langchain-based AI agent integrated with OpenAI’s GPT-4 model and memory to interpret queries and generate responses.
- **1.3 Data Retrieval:** Interfaces with PostgreSQL databases to fetch P&L reports and Balance Sheet data based on AI agent queries.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a chat message is received, providing the initial input for the financial analysis.

- **Nodes Involved:**  
  - *When chat message received*

- **Node Details:**  

  - **When chat message received**  
    - *Type & Role:* Chat trigger node specialized for Langchain chat interface; entry point of the workflow.  
    - *Configuration:* Listens for incoming chat messages via a webhook with ID "aef942c6-87aa-4dd8-9ea1-4b9b4140d843". No additional parameters configured.  
    - *Expressions/Variables:* Receives user message content as input data to be passed downstream.  
    - *Connections:* Outputs to the "AI Agent" node on the main channel.  
    - *Version Requirements:* Version 1.3 of the chat trigger node used.  
    - *Potential Failures:* Webhook connection failures, malformed input data, or missing messages.  
    - *Sub-Workflows:* None.

---

#### 1.2 AI Processing

- **Overview:**  
  This block processes the user's chat input using an AI agent that integrates GPT-4 language understanding, conversation memory, and orchestrates calls to database query nodes.

- **Nodes Involved:**  
  - *AI Agent*  
  - *OpenAI Chat Model*  
  - *Simple Memory*

- **Node Details:**  

  - **AI Agent**  
    - *Type & Role:* Langchain Agent node responsible for managing AI logic, orchestrating calls to language models, memory, and external tools.  
    - *Configuration:* No direct parameters specified in this export, suggesting default or externally configured agent setup. It connects AI language model, memory, and tools (PostgreSQL query nodes).  
    - *Expressions/Variables:* Receives chat input from "When chat message received" node. Uses references to other nodes for memory and tool use.  
    - *Connections:*  
      - Receives main input from "When chat message received" node.  
      - Connects to "OpenAI Chat Model" as the language model (`ai_languageModel`) input.  
      - Connects to "Simple Memory" as memory buffer (`ai_memory`).  
      - Connects to "P_L_Reports" and "Balance_Sheets" nodes as AI tools (`ai_tool`).  
    - *Version Requirements:* Type version 2.1, compatibility with Langchain v0.13+ recommended.  
    - *Potential Failures:* Misconfiguration of AI model or memory nodes, failures in invoking connected tool nodes, API quota limits, or malformed queries causing parsing errors.  
    - *Sub-Workflows:* None.

  - **OpenAI Chat Model**  
    - *Type & Role:* Provides GPT-4 based chat language model backend for the AI agent.  
    - *Configuration:* Default or externally configured OpenAI credentials and parameters (temperature, max tokens, etc.) not shown here.  
    - *Expressions/Variables:* Processes prompts and returns chat completions to the AI agent.  
    - *Connections:* Outputs to "AI Agent" via the `ai_languageModel` input.  
    - *Version Requirements:* Type version 1.2, requires OpenAI API key configured in n8n credentials.  
    - *Potential Failures:* API authentication errors, request timeouts, rate limiting, or invalid parameters.  
    - *Sub-Workflows:* None.

  - **Simple Memory**  
    - *Type & Role:* Implements a sliding window memory buffer to maintain conversational context for the AI agent.  
    - *Configuration:* Default parameters imply a window size or token limit to manage conversation history.  
    - *Expressions/Variables:* Stores and retrieves chat history for the agent’s context.  
    - *Connections:* Inputs and outputs memory data to the "AI Agent" via the `ai_memory` port.  
    - *Version Requirements:* Type version 1.3.  
    - *Potential Failures:* Memory overflow if window size is too small or state synchronization issues.  
    - *Sub-Workflows:* None.

---

#### 1.3 Data Retrieval

- **Overview:**  
  This block interfaces with PostgreSQL to query financial datasets (P&L reports and Balance Sheets) as requested by the AI agent during analysis.

- **Nodes Involved:**  
  - *P_L_Reports*  
  - *Balance_Sheets*

- **Node Details:**  

  - **P_L_Reports**  
    - *Type & Role:* PostgreSQL Tool node configured to query Profit & Loss report data from the connected database.  
    - *Configuration:* No explicit query or parameters shown, implying dynamic SQL queries are constructed or passed by the AI agent.  
    - *Expressions/Variables:* Receives query parameters or SQL text from the AI agent node’s tool invocation.  
    - *Connections:* Outputs directly to the AI Agent node’s `ai_tool` input.  
    - *Version Requirements:* Type version 2.6; requires PostgreSQL credentials configured in n8n.  
    - *Potential Failures:* Database connection errors, SQL syntax errors, permission issues, or empty result sets.  
    - *Sub-Workflows:* None.

  - **Balance_Sheets**  
    - *Type & Role:* PostgreSQL Tool node configured to query Balance Sheet data.  
    - *Configuration:* Similar to P_L_Reports, likely supports dynamic queries issued by the AI agent.  
    - *Expressions/Variables:* Receives SQL or parameters from the AI agent to fetch relevant balance sheet records.  
    - *Connections:* Outputs to AI Agent node’s `ai_tool` input.  
    - *Version Requirements:* Type version 2.6; requires PostgreSQL credentials.  
    - *Potential Failures:* Same as P_L_Reports – connectivity, permissions, query validity.  
    - *Sub-Workflows:* None.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                      | Input Node(s)               | Output Node(s)             | Sticky Note |
|-------------------------|--------------------------------------|------------------------------------|-----------------------------|----------------------------|-------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Input reception - triggers workflow | -                           | AI Agent                   |             |
| AI Agent                | @n8n/n8n-nodes-langchain.agent        | Central AI processing & orchestration | When chat message received, OpenAI Chat Model, Simple Memory, P_L_Reports, Balance_Sheets | -                          |             |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi | GPT-4 language model backend        | AI Agent                    | AI Agent                   |             |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context   | AI Agent                    | AI Agent                   |             |
| P_L_Reports             | n8n-nodes-base.postgresTool            | Queries P&L financial data          | AI Agent                    | AI Agent                   |             |
| Balance_Sheets          | n8n-nodes-base.postgresTool            | Queries Balance Sheet financial data | AI Agent                    | AI Agent                   |             |
| Sticky Note             | n8n-nodes-base.stickyNote              | (Empty content)                     | -                           | -                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.chatTrigger` named "When chat message received".  
   - Configure a webhook (or use default) to accept incoming chat messages.  
   - No additional parameters needed.

2. **Set Up the AI Agent Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.agent` named "AI Agent".  
   - Leave most parameters default but ensure it can accept inputs from the chat trigger as main input.  
   - Configure the AI Agent to integrate with the language model, memory, and external data tools (next steps).

3. **Add the OpenAI Chat Model Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.lmChatOpenAi` named "OpenAI Chat Model".  
   - Configure OpenAI credentials with an API key capable of GPT-4 usage.  
   - Set model parameters such as temperature (e.g., 0.7), max tokens, and any relevant advanced settings.  
   - Connect its output to the AI Agent node via the `ai_languageModel` input.

4. **Add the Simple Memory Node**  
   - Add a node of type `@n8n/n8n-nodes-langchain.memoryBufferWindow` named "Simple Memory".  
   - Configure memory window size (token or message count) as needed for context retention (e.g., last 5 messages).  
   - Connect its input and output to the AI Agent node via the `ai_memory` input.

5. **Add PostgreSQL Tool Nodes for Data Retrieval**  
   - Add a node of type `n8n-nodes-base.postgresTool` named "P_L_Reports".  
   - Configure PostgreSQL credentials with access to the database containing P&L data.  
   - Leave SQL query dynamic or prepare queries depending on AI agent instructions.  
   - Connect the node output to the AI Agent node on the `ai_tool` input.

   - Add a second node of the same type named "Balance_Sheets".  
   - Configure PostgreSQL credentials for the balance sheet database or same database if unified.  
   - Setup similar to "P_L_Reports".  
   - Connect output to the AI Agent node on the `ai_tool` input.

6. **Establish Connections**  
   - Connect "When chat message received" node’s main output to the AI Agent node main input.  
   - Connect OpenAI Chat Model to AI Agent using the `ai_languageModel` input.  
   - Connect Simple Memory to AI Agent using the `ai_memory` input.  
   - Connect both PostgreSQL nodes (P_L_Reports and Balance_Sheets) to AI Agent using the `ai_tool` inputs.

7. **Test the Workflow**  
   - Activate the workflow.  
   - Send a chat message to the webhook URL.  
   - Verify that the AI agent interprets the query, retrieves financial data from PostgreSQL nodes, and returns a contextual response.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow leverages Langchain integration nodes for n8n, facilitating complex AI orchestration. | See https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                               |
| Requires valid OpenAI GPT-4 API credentials with sufficient quota for chat completions.           | https://platform.openai.com/account/api-keys                                                     |
| PostgreSQL connectivity needs proper credential setup and firewall access for database nodes.     | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/                      |
| For optimal memory management, adjust the sliding window size in the Simple Memory node as needed.|                                                                                                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---