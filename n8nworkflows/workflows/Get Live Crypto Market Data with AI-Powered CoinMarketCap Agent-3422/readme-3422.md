Get Live Crypto Market Data with AI-Powered CoinMarketCap Agent

https://n8nworkflows.xyz/workflows/get-live-crypto-market-data-with-ai-powered-coinmarketcap-agent-3422


# Get Live Crypto Market Data with AI-Powered CoinMarketCap Agent

### 1. Workflow Overview

This n8n workflow, titled **Get Live Crypto Market Data with AI-Powered CoinMarketCap Agent**, is designed to provide real-time cryptocurrency market data, metadata, rankings, and conversion functionalities by leveraging the CoinMarketCap API and GPT-4o-mini AI. It serves crypto analysts, traders, and developers who require up-to-date and comprehensive crypto insights.

The workflow is modular and AI-driven, intelligently routing user queries to the appropriate CoinMarketCap API endpoint via six specialized tools. It can operate standalone or be triggered by a supervisor AI agent for multi-agent orchestration.

**Logical Blocks:**

- **1.1 Input Reception:** Receives external triggers with user queries and session context.
- **1.2 AI Processing Core:** Uses GPT-4o-mini as the brain and session memory to interpret queries and maintain context.
- **1.3 AI Agent Tool Router:** Routes queries to the correct CoinMarketCap API tool based on AI decision.
- **1.4 CoinMarketCap API Tools:** Six HTTP Request nodes each representing a CoinMarketCap API endpoint for specific data retrieval or conversion.
- **1.5 Documentation & Support:** Sticky notes providing usage instructions, error handling, licensing, and agent overview.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when called by another workflow, accepting user input and session context.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: `Execute Workflow Trigger`  
    - Role: Entry point for external workflows to trigger this agent with inputs `message` (user query) and `sessionId` (session context ID).  
    - Configuration: Defines two workflow inputs named `message` and `sessionId`.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to the AI Agent node.  
    - Edge Cases: Missing or malformed inputs may cause downstream errors; ensure supervisor workflows provide both inputs.  
    - Version: 1.1

#### 1.2 AI Processing Core

- **Overview:**  
  This block interprets the user query using GPT-4o-mini and maintains session context for multi-turn conversations.

- **Nodes Involved:**  
  - Crypto Agent Brain  
  - Crypto Agent Memory

- **Node Details:**  
  - **Crypto Agent Brain**  
    - Type: `lmChatOpenAi` (OpenAI Chat Language Model)  
    - Role: Processes the user message with GPT-4o-mini to understand intent and generate AI responses.  
    - Configuration: Uses model `gpt-4o-mini` with default options.  
    - Credentials: OpenAI API key configured.  
    - Inputs: Receives `message` from trigger node via AI Agent.  
    - Outputs: Feeds AI Agent for tool routing.  
    - Edge Cases: API rate limits, network issues, or invalid API key may cause failures.  
    - Version: 1.2

  - **Crypto Agent Memory**  
    - Type: `memoryBufferWindow` (Langchain Memory Node)  
    - Role: Stores session conversation history keyed by `sessionId` to maintain context across calls.  
    - Configuration: Default window buffer memory (no custom parameters).  
    - Inputs: Receives conversation data from AI Brain.  
    - Outputs: Provides memory context back to AI Agent.  
    - Edge Cases: Memory overflow or missing `sessionId` may cause context loss.  
    - Version: 1.3

#### 1.3 AI Agent Tool Router

- **Overview:**  
  This node acts as the AI-powered router, receiving the user query and session memory, then selecting the appropriate CoinMarketCap API tool to fulfill the request.

- **Nodes Involved:**  
  - CoinMarketCap Crypto Agent

- **Node Details:**  
  - **CoinMarketCap Crypto Agent**  
    - Type: `langchain.agent` (AI Agent Node)  
    - Role: Central AI agent that receives the user message and session memory, uses the AI Brain to interpret intent, and routes the query to one of six connected CoinMarketCap API tools.  
    - Configuration:  
      - Input text expression: `={{ $json.message }}` (dynamic user query)  
      - System message defines the agent role as an AI cryptocurrency analyst with access to six live CoinMarketCap tools, each described with endpoint, purpose, and supported inputs.  
      - Prompt type: `define` (fixed system prompt)  
    - Inputs: Receives `message` and `sessionId` from trigger node and memory node.  
    - Outputs: Routes to one of six tool HTTP Request nodes based on AI decision.  
    - Edge Cases: Misinterpretation of user query, missing or invalid parameters, or API tool failures may cause incomplete or incorrect responses.  
    - Version: 1.8

#### 1.4 CoinMarketCap API Tools

- **Overview:**  
  Six HTTP Request nodes represent live API endpoints from CoinMarketCap, each specialized for a particular type of crypto data or conversion. The AI Agent routes queries to these tools.

- **Nodes Involved:**  
  - Crypto Map  
  - Crypto Info  
  - Crypto Listings  
  - CoinMarketCap Price  
  - Global Metrics  
  - Price Conversion

- **Node Details:**  

  - **Crypto Map**  
    - Type: `langchain.toolHttpRequest`  
    - Role: Retrieves CoinMarketCap IDs, symbols, and names of cryptocurrencies.  
    - Configuration:  
      - URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/map`  
      - Query parameters: `symbol`, `listing_status`, `start`, `limit` (all optional, passed from AI Agent)  
      - Headers: `Accept: application/json`  
      - Authentication: HTTP Header Auth with CoinMarketCap API key  
    - Inputs: Query parameters from AI Agent  
    - Outputs: JSON map data of cryptocurrencies  
    - Edge Cases: Invalid symbols, rate limiting, or missing API key errors.  
    - Version: 1.1

  - **Crypto Info**  
    - Type: `langchain.toolHttpRequest`  
    - Role: Fetches metadata including descriptions, whitepapers, logos, and social links for specified cryptocurrencies.  
    - Configuration:  
      - URL: `https://pro-api.coinmarketcap.com/v2/cryptocurrency/info`  
      - Query parameter: `symbol` (required)  
      - Headers: `Accept: application/json`  
      - Authentication: HTTP Header Auth  
    - Inputs: Symbols from AI Agent  
    - Outputs: Metadata JSON  
    - Edge Cases: Missing or invalid symbols, API errors.  
    - Version: 1.1

  - **Crypto Listings**  
    - Type: `langchain.toolHttpRequest`  
    - Role: Retrieves ranked lists of cryptocurrencies sorted by market cap with pagination and currency conversion.  
    - Configuration:  
      - URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest`  
      - Query parameters: `start`, `limit`, `convert` (optional)  
      - Headers: `Accept: application/json`  
      - Authentication: HTTP Header Auth  
    - Inputs: Pagination and currency parameters from AI Agent  
    - Outputs: Ranked listings JSON  
    - Edge Cases: Invalid pagination, unsupported convert currency, rate limits.  
    - Version: 1.1

  - **CoinMarketCap Price**  
    - Type: `langchain.toolHttpRequest`  
    - Role: Provides real-time price, volume, and market cap data for specified cryptocurrencies.  
    - Configuration:  
      - URL: `https://pro-api.coinmarketcap.com/v2/cryptocurrency/quotes/latest`  
      - Query parameters: `symbol` (required), `convert` (optional)  
      - Headers: `Accept: application/json`  
      - Authentication: HTTP Header Auth  
    - Inputs: Symbols and convert currency from AI Agent  
    - Outputs: Price and volume JSON  
    - Edge Cases: Missing symbols, invalid convert currency, API errors.  
    - Version: 1.1

  - **Global Metrics**  
    - Type: `langchain.toolHttpRequest`  
    - Role: Returns global cryptocurrency market metrics such as total market cap, 24h volume, BTC dominance.  
    - Configuration:  
      - URL: `https://pro-api.coinmarketcap.com/v1/global-metrics/quotes/latest`  
      - No query parameters required  
      - Headers: `Accept: application/json`  
      - Authentication: HTTP Header Auth  
    - Inputs: None  
    - Outputs: Global market stats JSON  
    - Edge Cases: API downtime, rate limits.  
    - Version: 1.1

  - **Price Conversion**  
    - Type: `langchain.toolHttpRequest`  
    - Role: Converts amounts between cryptocurrencies and fiat currencies.  
    - Configuration:  
      - URL: `https://pro-api.coinmarketcap.com/v1/tools/price-conversion`  
      - Query parameters: `amount`, `symbol`, `convert` (all required)  
      - Headers: `Accept: application/json`  
      - Authentication: HTTP Header Auth  
    - Inputs: Conversion parameters from AI Agent  
    - Outputs: Conversion result JSON  
    - Edge Cases: Missing or invalid parameters, unsupported currencies, rate limits.  
    - Version: 1.1

#### 1.5 Documentation & Support

- **Overview:**  
  Sticky notes provide essential documentation, usage examples, error handling guidance, and licensing information embedded within the workflow canvas.

- **Nodes Involved:**  
  - Crypto Agent Guide  
  - Usage & Examples  
  - Errors & Licensing

- **Node Details:**  

  - **Crypto Agent Guide**  
    - Type: `stickyNote`  
    - Role: Overview of the agent, supported endpoints, node roles, and input requirements.  
    - Content: Detailed description of the agent’s purpose, endpoints, AI brain, memory, and input parameters.  
    - Position: Top-left corner for easy reference.

  - **Usage & Examples**  
    - Type: `stickyNote`  
    - Role: Step-by-step usage instructions and sample API queries for common tasks.  
    - Content: Instructions on inputs, triggering, and sample GET requests for conversions, listings, and metrics.

  - **Errors & Licensing**  
    - Type: `stickyNote`  
    - Role: Lists common API error codes, troubleshooting tips, and licensing/IP rights information.  
    - Content: HTTP error codes explanation, tips to avoid rate limits, and copyright/licensing notes with external LinkedIn link.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                                | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                      |
|------------------------------|----------------------------------|-----------------------------------------------|--------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger         | Entry point receiving `message` and `sessionId` | None                           | CoinMarketCap Crypto Agent     |                                                                                                |
| CoinMarketCap Crypto Agent    | Langchain Agent                  | AI router selecting appropriate CoinMarketCap tool | When Executed by Another Workflow, Crypto Agent Brain, Crypto Agent Memory | Crypto Map, Crypto Info, Crypto Listings, CoinMarketCap Price, Global Metrics, Price Conversion |                                                                                                |
| Crypto Agent Brain            | OpenAI Chat Language Model       | Processes user query with GPT-4o-mini          | CoinMarketCap Crypto Agent     | Crypto Agent Memory            |                                                                                                |
| Crypto Agent Memory           | Langchain Memory Buffer          | Maintains session context for multi-turn chat | Crypto Agent Brain             | CoinMarketCap Crypto Agent     |                                                                                                |
| Crypto Map                   | HTTP Request (Langchain Tool)    | Retrieves CoinMarketCap IDs and active coins  | CoinMarketCap Crypto Agent     | None                          |                                                                                                |
| Crypto Info                  | HTTP Request (Langchain Tool)    | Fetches metadata, whitepapers, social links   | CoinMarketCap Crypto Agent     | None                          |                                                                                                |
| Crypto Listings              | HTTP Request (Langchain Tool)    | Retrieves ranked coin listings by market cap  | CoinMarketCap Crypto Agent     | None                          |                                                                                                |
| CoinMarketCap Price          | HTTP Request (Langchain Tool)    | Provides live price, volume, and supply data  | CoinMarketCap Crypto Agent     | None                          |                                                                                                |
| Global Metrics               | HTTP Request (Langchain Tool)    | Returns global crypto market stats             | CoinMarketCap Crypto Agent     | None                          |                                                                                                |
| Price Conversion             | HTTP Request (Langchain Tool)    | Converts crypto/fiat amounts                    | CoinMarketCap Crypto Agent     | None                          |                                                                                                |
| Crypto Agent Guide           | Sticky Note                     | Documentation: agent overview and endpoints    | None                           | None                          | # 🧠 CoinMarketCap_Crypto_Agent_Tool Guide: agent overview, endpoints, inputs                   |
| Usage & Examples             | Sticky Note                     | Usage instructions and sample prompts          | None                           | None                          | Usage instructions, sample API queries for conversions, listings, and metrics                   |
| Errors & Licensing           | Sticky Note                     | API error codes, troubleshooting, licensing    | None                           | None                          | API error codes, troubleshooting tips, licensing info, LinkedIn link: https://linkedin.com/in/donjayamahajr |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add `Execute Workflow Trigger` node named `When Executed by Another Workflow`.  
   - Configure inputs: `message` (string), `sessionId` (string).  
   - Position: Left side.

2. **Add AI Brain Node**  
   - Add `lmChatOpenAi` node named `Crypto Agent Brain`.  
   - Set model to `gpt-4o-mini`.  
   - Configure OpenAI credentials with your API key.  
   - Position: Below trigger node.

3. **Add Memory Node**  
   - Add `memoryBufferWindow` node named `Crypto Agent Memory`.  
   - Default settings (no parameters).  
   - Position: Next to AI Brain node.

4. **Add AI Agent Node**  
   - Add `langchain.agent` node named `CoinMarketCap Crypto Agent`.  
   - Set input text expression to `={{ $json.message }}`.  
   - Paste the system message describing the six CoinMarketCap tools with their endpoints, purposes, and supported inputs (see overview section).  
   - Set prompt type to `define`.  
   - Connect inputs:  
     - From `When Executed by Another Workflow` (main input)  
     - From `Crypto Agent Brain` (ai_languageModel)  
     - From `Crypto Agent Memory` (ai_memory)  
   - Position: Center.

5. **Add HTTP Request Nodes for Each API Tool**  
   For each of the six tools below, create a `langchain.toolHttpRequest` node with the specified configuration and connect its `ai_tool` input to the `CoinMarketCap Crypto Agent` node:

   - **Crypto Map**  
     - URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/map`  
     - Query parameters: `symbol`, `listing_status`, `start`, `limit` (all optional)  
     - Headers: `Accept: application/json`  
     - Authentication: HTTP Header Auth with CoinMarketCap API key  
     - Position: Right of AI Agent node.

   - **Crypto Info**  
     - URL: `https://pro-api.coinmarketcap.com/v2/cryptocurrency/info`  
     - Query parameter: `symbol` (required)  
     - Headers: `Accept: application/json`  
     - Authentication: HTTP Header Auth  
     - Position: Next to Crypto Map node.

   - **Crypto Listings**  
     - URL: `https://pro-api.coinmarketcap.com/v1/cryptocurrency/listings/latest`  
     - Query parameters: `start`, `limit`, `convert` (optional)  
     - Headers: `Accept: application/json`  
     - Authentication: HTTP Header Auth  
     - Position: Next to Crypto Info node.

   - **CoinMarketCap Price**  
     - URL: `https://pro-api.coinmarketcap.com/v2/cryptocurrency/quotes/latest`  
     - Query parameters: `symbol` (required), `convert` (optional)  
     - Headers: `Accept: application/json`  
     - Authentication: HTTP Header Auth  
     - Position: Next to Crypto Listings node.

   - **Global Metrics**  
     - URL: `https://pro-api.coinmarketcap.com/v1/global-metrics/quotes/latest`  
     - No query parameters  
     - Headers: `Accept: application/json`  
     - Authentication: HTTP Header Auth  
     - Position: Next to CoinMarketCap Price node.

   - **Price Conversion**  
     - URL: `https://pro-api.coinmarketcap.com/v1/tools/price-conversion`  
     - Query parameters: `amount`, `symbol`, `convert` (all required)  
     - Headers: `Accept: application/json`  
     - Authentication: HTTP Header Auth  
     - Position: Next to Global Metrics node.

6. **Connect Nodes**  
   - Connect `When Executed by Another Workflow` main output to `CoinMarketCap Crypto Agent` main input.  
   - Connect `Crypto Agent Brain` output to `CoinMarketCap Crypto Agent` ai_languageModel input.  
   - Connect `Crypto Agent Memory` output to `CoinMarketCap Crypto Agent` ai_memory input.  
   - Connect `CoinMarketCap Crypto Agent` ai_tool output to each of the six HTTP Request nodes.

7. **Configure Credentials**  
   - Create an HTTP Header Auth credential in n8n with your CoinMarketCap API key.  
   - Assign this credential to all six HTTP Request nodes.  
   - Configure OpenAI API credentials for the `Crypto Agent Brain` node.

8. **Add Sticky Notes (Optional but Recommended)**  
   - Add three sticky notes with the content from the documentation and support section:  
     - Crypto Agent Guide  
     - Usage & Examples  
     - Errors & Licensing  
   - Position them for easy reference.

9. **Test the Workflow**  
   - Trigger the workflow manually or via a supervisor workflow passing `message` and `sessionId`.  
   - Use example queries like:  
     - “Convert 5 ETH to USD”  
     - “What is the CoinMarketCap ID for PEPE?”  
     - “Show me the top 10 cryptocurrencies by market cap.”

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This agent is part of the modular **CoinMarketCap AI Analyst System** for crypto market analysis.             | Creator profile: [https://n8n.io/creators/don-the-gem-dealer/](https://n8n.io/creators/don-the-gem-dealer/) |
| API Key registration required at CoinMarketCap: [https://coinmarketcap.com/api/](https://coinmarketcap.com/api/) | Official CoinMarketCap API documentation and key registration.                                     |
| Troubleshooting tips include checking API keys, respecting rate limits, and validating query parameters.      | See Errors & Licensing sticky note for detailed error codes and tips.                              |
| Licensing: Intellectual property of Treasurium Capital Limited Company; unauthorized use prohibited.           | Licensing details and contact via LinkedIn: [https://linkedin.com/in/donjayamahajr](https://linkedin.com/in/donjayamahajr) |
| Recommended prompt style: descriptive queries specifying symbols, amounts, and desired outputs for best results. | Examples included in Usage & Examples sticky note.                                                 |

---

This comprehensive documentation enables advanced users and AI agents to fully understand, reproduce, and extend the CoinMarketCap Crypto Agent workflow with confidence, anticipating potential errors and integration nuances.