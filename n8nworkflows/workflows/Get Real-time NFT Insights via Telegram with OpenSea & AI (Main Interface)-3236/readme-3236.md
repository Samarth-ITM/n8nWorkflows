Get Real-time NFT Insights via Telegram with OpenSea & AI (Main Interface)

https://n8nworkflows.xyz/workflows/get-real-time-nft-insights-via-telegram-with-opensea---ai--main-interface--3236


# Get Real-time NFT Insights via Telegram with OpenSea & AI (Main Interface)

### 1. Workflow Overview

This workflow, **OpenSea AI-Powered Insights via Telegram**, is designed to provide real-time NFT market intelligence directly through a Telegram bot interface. It integrates the OpenSea API, GPT-4o-mini AI, and Telegram to allow users to query NFT market data in natural language and receive structured, actionable insights instantly.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user messages from Telegram and prepares session context.
- **1.2 AI Supervisor Processing:** Uses GPT-4o-mini to interpret user queries, maintain conversation memory, and route requests.
- **1.3 Agent Tool Invocation:** Calls one of three specialized sub-workflows (Analytics, NFT, Marketplace) based on AI routing.
- **1.4 Response Delivery:** Sends the processed and formatted insights back to the user via Telegram.

This modular design allows complex multi-step NFT data queries, combining multiple data sources and AI reasoning, all orchestrated seamlessly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages, captures them, and assigns a session ID based on the chat to maintain context across interactions.

**Nodes Involved:**  
- Telegram Trigger  
- Adds SessionId  
- When chat message received

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Listens for new Telegram messages (updates of type "message").  
  - *Configuration:* Uses Telegram API credentials; triggers on any message.  
  - *Connections:* Outputs to Adds SessionId node.  
  - *Edge Cases:* Telegram API downtime, invalid bot token, message format errors.

- **Adds SessionId**  
  - *Type:* Set node  
  - *Role:* Adds a `sessionId` field equal to the Telegram chat ID to track conversation context.  
  - *Configuration:* Sets `sessionId` = `{{$json.message.chat.id}}`.  
  - *Connections:* Outputs to OpenSea AI-Powered Insights Agent node.  
  - *Edge Cases:* Missing or malformed chat ID in message JSON.

- **When chat message received**  
  - *Type:* LangChain Chat Trigger  
  - *Role:* Captures chat messages for AI processing (likely for LangChain integration).  
  - *Configuration:* Default options; webhook enabled.  
  - *Connections:* Outputs to OpenSea AI-Powered Insights Agent node.  
  - *Edge Cases:* Webhook misconfiguration, message parsing errors.

---

#### 2.2 AI Supervisor Processing

**Overview:**  
This block uses GPT-4o-mini to interpret the user's natural language query, maintain conversation memory, and decide which specialized agent tool(s) to invoke.

**Nodes Involved:**  
- Opensea Supervisor Brain  
- Opensea Supervisor Memory  
- OpenSea AI-Powered Insights Agent

**Node Details:**

- **Opensea Supervisor Brain**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Processes user input with GPT-4o-mini to understand query intent and generate AI responses.  
  - *Configuration:* Model set to `gpt-4o-mini`; uses OpenAI API credentials.  
  - *Connections:* Feeds AI language model input to OpenSea AI-Powered Insights Agent.  
  - *Edge Cases:* API rate limits, network timeouts, invalid API key.

- **Opensea Supervisor Memory**  
  - *Type:* LangChain Memory Buffer Window  
  - *Role:* Stores recent conversation history to maintain context for multi-turn dialogs.  
  - *Configuration:* Default buffer window; no custom parameters.  
  - *Connections:* Provides memory input to OpenSea AI-Powered Insights Agent.  
  - *Edge Cases:* Memory overflow or loss, inconsistent context.

- **OpenSea AI-Powered Insights Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Central AI agent that receives user messages, uses the supervisor brain and memory, and routes queries to the appropriate sub-agent tools.  
  - *Configuration:*  
    - Input text: `{{$json.message.text}}` (user query)  
    - System message defines agent capabilities and instructions, including the use of three sub-agents (Marketplace, Analytics, NFT).  
  - *Connections:*  
    - Receives main input from Adds SessionId and When chat message received nodes.  
    - Connects to AI language model (Opensea Supervisor Brain), AI memory (Opensea Supervisor Memory), and AI tools (three sub-agent workflows).  
    - Outputs to Telegram node.  
  - *Edge Cases:* Expression evaluation errors, AI misinterpretation, sub-agent workflow failures.

---

#### 2.3 Agent Tool Invocation

**Overview:**  
Based on AI routing decisions, this block invokes one or more sub-workflows specialized in different NFT data domains: Analytics, NFT metadata, and Marketplace data.

**Nodes Involved:**  
- OpenSea Analytics Agent Tool  
- OpenSea NFT Agent Tool  
- OpenSea Marketplace Agent Tool

**Node Details:**

- **OpenSea Analytics Agent Tool**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Handles queries related to market trends, sales volume, transaction history, and analytics.  
  - *Configuration:* Calls workflow ID `yRMCUm6oJEMknhbw` (must be installed and published separately).  
  - *Inputs:* Passes `message` (user query) and `sessionId`.  
  - *Connections:* Outputs back to OpenSea AI-Powered Insights Agent.  
  - *Edge Cases:* Sub-workflow not published, API key missing, invalid query parameters.

- **OpenSea NFT Agent Tool**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Provides detailed NFT metadata, ownership info, traits, and payment token data.  
  - *Configuration:* Calls workflow ID `ZBH1ExE58wsoodkZ`.  
  - *Inputs:* Passes `message` and `sessionId`.  
  - *Connections:* Outputs back to OpenSea AI-Powered Insights Agent.  
  - *Edge Cases:* Missing contract addresses or token IDs, invalid blockchain names.

- **OpenSea Marketplace Agent Tool**  
  - *Type:* LangChain Tool Workflow  
  - *Role:* Fetches live marketplace data including listings, offers, orders, and trait-based pricing.  
  - *Configuration:* Calls workflow ID `brRSLvIkYp3mLq0K`.  
  - *Inputs:* Passes `message` and `sessionId`.  
  - *Connections:* Outputs back to OpenSea AI-Powered Insights Agent.  
  - *Edge Cases:* Unsupported blockchain names, pagination issues, API limits.

---

#### 2.4 Response Delivery

**Overview:**  
This block formats the AI-generated insights and sends them back to the user through Telegram.

**Nodes Involved:**  
- Telegram

**Node Details:**

- **Telegram**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends the AI-generated response text back to the Telegram chat.  
  - *Configuration:*  
    - Text: `{{$json.output}}` (output from AI agent)  
    - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}` (target chat)  
    - Credentials: Telegram API credentials.  
    - Additional fields: disables attribution append.  
  - *Connections:* Receives input from OpenSea AI-Powered Insights Agent.  
  - *Edge Cases:* Telegram API errors, invalid chat ID, message size limits.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                           | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                         |
|-------------------------------|---------------------------------|-----------------------------------------|---------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger              | Telegram Trigger                | Receives Telegram messages               | —                               | Adds SessionId                  | See Sticky Note1 for data flow and example queries.                                                                 |
| Adds SessionId               | Set                            | Adds sessionId for context tracking      | Telegram Trigger                | OpenSea AI-Powered Insights Agent | See Sticky Note1 for data flow and example queries.                                                                 |
| When chat message received   | LangChain Chat Trigger          | Captures chat messages for AI processing | —                               | OpenSea AI-Powered Insights Agent |                                                                                                                     |
| Opensea Supervisor Brain     | LangChain OpenAI Chat Model     | Processes user input with GPT-4o-mini    | —                               | OpenSea AI-Powered Insights Agent |                                                                                                                     |
| Opensea Supervisor Memory    | LangChain Memory Buffer Window  | Maintains conversation context           | —                               | OpenSea AI-Powered Insights Agent |                                                                                                                     |
| OpenSea AI-Powered Insights Agent | LangChain Agent             | Central AI agent routing queries          | Adds SessionId, When chat message received, Opensea Supervisor Brain, Opensea Supervisor Memory, Agent Tools | Telegram                      | See Sticky Note for detailed system overview and setup instructions.                                                |
| OpenSea Analytics Agent Tool | LangChain Tool Workflow         | Handles analytics-related NFT queries    | OpenSea AI-Powered Insights Agent | OpenSea AI-Powered Insights Agent |                                                                                                                     |
| OpenSea NFT Agent Tool       | LangChain Tool Workflow         | Handles NFT metadata and ownership queries | OpenSea AI-Powered Insights Agent | OpenSea AI-Powered Insights Agent |                                                                                                                     |
| OpenSea Marketplace Agent Tool | LangChain Tool Workflow       | Handles marketplace listings and offers  | OpenSea AI-Powered Insights Agent | OpenSea AI-Powered Insights Agent |                                                                                                                     |
| Telegram                    | Telegram node                   | Sends AI response back to Telegram user | OpenSea AI-Powered Insights Agent | —                               |                                                                                                                     |
| Sticky Note                 | Sticky Note                    | Provides full integration guide and overview | —                               | —                               | Covers entire workflow architecture and setup instructions.                                                        |
| Sticky Note1                | Sticky Note                    | Details data flow, execution steps, and example queries | —                               | —                               | Covers Input Reception and AI processing blocks with example queries.                                              |
| Sticky Note2                | Sticky Note                    | Critical setup notes and troubleshooting | —                               | —                               | Covers credential setup, chain name warnings, session tracking, pagination, and final thoughts.                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use [@BotFather](https://t.me/BotFather) to create a Telegram bot and obtain the API token.  
   - In n8n, create Telegram API credentials using this token.

2. **Create OpenSea API Credentials**  
   - Register and obtain an OpenSea API key from the [OpenSea Developer Portal](https://docs.opensea.io/reference/api-keys).  
   - Add these credentials in n8n for use by sub-workflows.

3. **Create Telegram Trigger Node**  
   - Add a **Telegram Trigger** node.  
   - Configure it to listen for "message" updates.  
   - Assign the Telegram API credentials.  
   - This node will receive user messages.

4. **Add Set Node to Assign Session ID**  
   - Add a **Set** node named "Adds SessionId".  
   - Configure it to add a new field `sessionId` with value `{{$json.message.chat.id}}`.  
   - Enable "Include Other Fields" to pass through all original data.

5. **Add LangChain Chat Trigger Node**  
   - Add a **When chat message received** node (LangChain Chat Trigger).  
   - Use default settings to capture chat messages for AI processing.

6. **Add LangChain OpenAI Chat Model Node**  
   - Add a node named "Opensea Supervisor Brain" of type LangChain OpenAI Chat.  
   - Set model to `gpt-4o-mini`.  
   - Assign OpenAI API credentials.

7. **Add LangChain Memory Buffer Node**  
   - Add a node named "Opensea Supervisor Memory" of type LangChain Memory Buffer Window.  
   - Use default buffer settings.

8. **Add LangChain Agent Node**  
   - Add a node named "OpenSea AI-Powered Insights Agent" of type LangChain Agent.  
   - Configure input text as `{{$json.message.text}}`.  
   - Paste the detailed system prompt describing the agent’s role, capabilities, and instructions for using three sub-agents (Marketplace, Analytics, NFT).  
   - Connect the following inputs:  
     - AI language model input from "Opensea Supervisor Brain"  
     - AI memory input from "Opensea Supervisor Memory"  
     - AI tool inputs from three Tool Workflow nodes (next steps)  
   - Connect main input from "Adds SessionId" and "When chat message received".

9. **Add Tool Workflow Nodes for Sub-Agents**  
   - Add three **LangChain Tool Workflow** nodes:  
     - "OpenSea Analytics Agent Tool" pointing to workflow ID `yRMCUm6oJEMknhbw`  
     - "OpenSea NFT Agent Tool" pointing to workflow ID `ZBH1ExE58wsoodkZ`  
     - "OpenSea Marketplace Agent Tool" pointing to workflow ID `brRSLvIkYp3mLq0K`  
   - Configure each to accept inputs:  
     - `message`: populated from AI agent message  
     - `sessionId`: passed from main workflow  
   - Ensure these sub-workflows are downloaded, imported, and published in your n8n instance.

10. **Add Telegram Node to Send Responses**  
    - Add a **Telegram** node to send messages.  
    - Configure text as `{{$json.output}}` (AI agent output).  
    - Set chat ID as `{{$('Telegram Trigger').item.json.message.chat.id}}`.  
    - Assign Telegram API credentials.  
    - Disable attribution append.

11. **Connect Nodes**  
    - Connect "Telegram Trigger" → "Adds SessionId" → "OpenSea AI-Powered Insights Agent"  
    - Connect "When chat message received" → "OpenSea AI-Powered Insights Agent"  
    - Connect "Opensea Supervisor Brain" (AI language model) → "OpenSea AI-Powered Insights Agent"  
    - Connect "Opensea Supervisor Memory" (AI memory) → "OpenSea AI-Powered Insights Agent"  
    - Connect each Tool Workflow node (Analytics, NFT, Marketplace) as AI tools input to "OpenSea AI-Powered Insights Agent"  
    - Connect "OpenSea AI-Powered Insights Agent" → "Telegram"

12. **Activate Workflow and Test**  
    - Publish the workflow.  
    - Test by sending queries to your Telegram bot such as:  
      - "Show me the 5 cheapest listings for Azuki."  
      - "Who owns Bored Ape #456?"  
      - "Compare sales volume for BAYC and CloneX last 30 days."

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow requires three sub-workflows to function properly: OpenSea Analytics Agent Tool, OpenSea NFT Agent Tool, and OpenSea Marketplace Agent Tool. These must be downloaded and published separately.                                       | See main description and sub-workflow links at: https://n8n.io/creators/don-the-gem-dealer/                 |
| Supported blockchain names must be used exactly as specified; notably, use `matic` instead of `polygon`. Using unsupported chain names will cause API errors.                                                                                     | See detailed blockchain list in the system prompt within the AI agent node.                                 |
| For large datasets (100+ results), use pagination parameters (`next`) in queries to avoid incomplete data.                                                                                                                                          | Critical for marketplace and analytics queries.                                                             |
| Session ID tracking is critical to maintain conversation context and enable multi-turn dialogs.                                                                                                                                                     | Managed by the "Adds SessionId" node and passed to sub-workflows.                                           |
| Telegram bot setup requires creating a bot via @BotFather and configuring credentials in n8n.                                                                                                                                                      | https://t.me/BotFather                                                                                       |
| OpenSea API key is required and must be configured in all sub-workflows for API authentication.                                                                                                                                                    | https://docs.opensea.io/reference/api-keys                                                                  |
| For help or questions, connect with the creator on LinkedIn: [http://linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr)                                                                                                         | Creator contact                                                                                              |
| The workflow is designed for NFT traders, collectors, and analysts to get AI-powered market intelligence on demand, enabling discovery of undervalued NFTs, trend tracking, and ownership insights—all from Telegram.                              | Workflow purpose and use cases.                                                                              |

---

This documentation provides a complete, structured reference for understanding, reproducing, and extending the **OpenSea AI-Powered Insights via Telegram** workflow and its integration with OpenSea and AI-powered sub-agents.