Get Exchange & Sentiment Insights with CoinMarketCap AI Agent

https://n8nworkflows.xyz/workflows/get-exchange---sentiment-insights-with-coinmarketcap-ai-agent-3423


# Get Exchange & Sentiment Insights with CoinMarketCap AI Agent

### 1. Workflow Overview

This workflow, titled **Get Exchange & Sentiment Insights with CoinMarketCap AI Agent**, is designed to provide comprehensive cryptocurrency exchange data, market index insights, and community sentiment analysis by leveraging CoinMarketCap’s API and AI-powered processing. It serves as a sub-agent within a larger CoinMarketCap AI Analyst ecosystem or as a standalone tool.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives external triggers with user queries and session context.
- **1.2 AI Processing Core:** Uses GPT-4o-mini to interpret queries, manage conversation memory, and orchestrate API tool calls.
- **1.3 API Tool Integrations:** Five dedicated API nodes fetch specific data from CoinMarketCap endpoints:
  - Exchange Map (lookup exchanges)
  - Exchange Info (metadata)
  - Exchange Assets (token holdings)
  - CMC 100 Index (top 100 crypto assets)
  - Fear & Greed Index (market sentiment)
- **1.4 Documentation & Guidance:** Sticky notes provide user instructions, usage examples, error handling, and licensing information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow execution, accepting inputs from a supervisor workflow or manual invocation. It collects the main query (`message`) and session identifier (`sessionId`) to maintain conversational context.

**Nodes Involved:**  
- When Executed by Another Workflow

**Node Details:**  
- **When Executed by Another Workflow**  
  - Type: `Execute Workflow Trigger`  
  - Role: Entry point triggered by external workflows, passing `message` and `sessionId` as inputs.  
  - Configuration: Defines workflow inputs `message` (user query) and `sessionId` (context key).  
  - Inputs: External trigger with parameters.  
  - Outputs: Passes inputs downstream to AI Agent node.  
  - Edge Cases: Missing inputs may cause downstream errors; ensure supervisor or manual trigger provides both parameters.

---

#### 1.2 AI Processing Core

**Overview:**  
This block interprets the user query using an AI language model, maintains session memory, and orchestrates calls to CoinMarketCap API tools based on the query context.

**Nodes Involved:**  
- Exchange and Community Agent Brain  
- Exchange and Community Agent Memory  
- CoinMarketCap Exchange and Community Agent

**Node Details:**  

- **Exchange and Community Agent Brain**  
  - Type: `LM Chat OpenAI` (GPT-4o-mini)  
  - Role: Processes natural language queries, generates AI-driven responses, and decides which API tools to invoke.  
  - Configuration: Uses GPT-4o-mini model with default options; linked to OpenAI credentials.  
  - Inputs: Receives `message` and memory context.  
  - Outputs: Sends processed prompts and instructions to the AI Agent node.  
  - Edge Cases: API rate limits or timeouts; malformed prompts may cause errors.

- **Exchange and Community Agent Memory**  
  - Type: `Memory Buffer Window`  
  - Role: Maintains conversational state using `sessionId` to provide context continuity across interactions.  
  - Configuration: Default window buffer memory without custom parameters.  
  - Inputs: Receives conversation data from AI Brain.  
  - Outputs: Provides memory context back to AI Brain and Agent node.  
  - Edge Cases: Memory overflow if session data grows too large; sessionId must be consistent.

- **CoinMarketCap Exchange and Community Agent**  
  - Type: `LangChain Agent`  
  - Role: Central orchestrator that receives user queries, uses system instructions to select appropriate API tools, and composes structured responses.  
  - Configuration:  
    - Text input bound to `{{$json.message}}` from trigger.  
    - System message defines agent role, supported tools, usage instructions, error handling, and API endpoint details.  
    - Prompt type: `define` (static system prompt).  
  - Inputs: Receives message and memory context.  
  - Outputs: Calls API tool nodes based on query interpretation.  
  - Edge Cases: If required parameters (e.g., exchange `id`) are missing, agent prompts user to refine query; large API responses may exceed token limits triggering error messages.

---

#### 1.3 API Tool Integrations

**Overview:**  
This block contains five HTTP Request nodes configured as tools to query CoinMarketCap API endpoints. Each node corresponds to a specific data domain and is invoked by the AI Agent as needed.

**Nodes Involved:**  
- Exchange Map  
- Exchange Info  
- Exchange Assets  
- CMC 100 Index  
- Fear and Greed Latest

**Node Details:**  

- **Exchange Map**  
  - Type: `Tool HTTP Request`  
  - Role: Retrieves a list of all exchanges with their CoinMarketCap IDs, names, and slugs.  
  - Configuration:  
    - URL: `https://pro-api.coinmarketcap.com/v1/exchange/map`  
    - Query parameter: `slug` (optional, used for filtering)  
    - Headers: Accept header set, uses HTTP Header Auth with CoinMarketCap API key.  
  - Inputs: Receives `slug` from AI Agent for targeted lookup.  
  - Outputs: Returns exchange mapping data for downstream use.  
  - Edge Cases: Missing or invalid slug returns full list or error; API rate limits apply.

- **Exchange Info**  
  - Type: `Tool HTTP Request`  
  - Role: Fetches metadata for a specific exchange by ID.  
  - Configuration:  
    - URL: `https://pro-api.coinmarketcap.com/v1/exchange/info`  
    - Query parameter: `id` (required)  
    - Headers: Accept header, HTTP Header Auth.  
  - Inputs: Exchange `id` from AI Agent after resolving via Exchange Map.  
  - Outputs: Returns detailed metadata including launch date, social links, country, and status.  
  - Edge Cases: Missing or invalid `id` causes error; API limits apply.

- **Exchange Assets**  
  - Type: `Tool HTTP Request`  
  - Role: Retrieves token holdings, wallet addresses, blockchain platform, and USD value for a given exchange.  
  - Configuration:  
    - URL: `https://pro-api.coinmarketcap.com/v1/exchange/assets`  
    - Query parameter: `id` (required)  
    - Headers: Accept header, HTTP Header Auth.  
  - Inputs: Exchange `id` resolved by Exchange Map.  
  - Outputs: Token balances and asset data.  
  - Edge Cases: Large data sets may exceed token limits; missing `id` triggers error.

- **CMC 100 Index**  
  - Type: `Tool HTTP Request`  
  - Role: Provides constituents and weights of the CoinMarketCap 100 Index.  
  - Configuration:  
    - URL: `https://pro-api.coinmarketcap.com/v3/index/cmc100-latest`  
    - Headers: Accept header, HTTP Header Auth.  
  - Inputs: No query parameters required.  
  - Outputs: Latest index data with asset breakdown.  
  - Edge Cases: API downtime or rate limits.

- **Fear and Greed Latest**  
  - Type: `Tool HTTP Request`  
  - Role: Retrieves the latest crypto market sentiment score and classification.  
  - Configuration:  
    - URL: `https://pro-api.coinmarketcap.com/v3/fear-and-greed/latest`  
    - Headers: Accept header, HTTP Header Auth.  
  - Inputs: No parameters required.  
  - Outputs: Sentiment index score (0-100) and descriptive classification.  
  - Edge Cases: Data updated daily; stale data possible if API delayed.

---

#### 1.4 Documentation & Guidance

**Overview:**  
Sticky notes provide essential user guidance, usage examples, error handling tips, and licensing information to support workflow users and maintainers.

**Nodes Involved:**  
- Exchange & Community Guide  
- Usage & Examples  
- Errors & Licensing

**Node Details:**  

- **Exchange & Community Guide**  
  - Type: `Sticky Note`  
  - Role: Summarizes agent purpose, supported API tools, components, and trigger parameters.  
  - Content Highlights:  
    - Lists 5 API endpoints with descriptions.  
    - Notes on agent brain, memory, and tools.  
    - Emphasizes using Exchange Map first for ID resolution.  
  - Position: Top-left for easy reference.

- **Usage & Examples**  
  - Type: `Sticky Note`  
  - Role: Provides stepwise usage instructions and example queries for common use cases.  
  - Content Highlights:  
    - Input requirements (`slug` or `id`).  
    - Supervisor trigger usage.  
    - Sample API calls for Fear & Greed, Binance holdings, CMC 100 Index, Coinbase info.  

- **Errors & Licensing**  
  - Type: `Sticky Note`  
  - Role: Details common API error codes, large response warnings, and licensing terms.  
  - Content Highlights:  
    - HTTP error codes 400, 401, 429, 500 explained.  
    - Warning message for data exceeding model capacity.  
    - Contact and copyright information.  

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                          | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                   |
|-----------------------------------|-----------------------------------|----------------------------------------|----------------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger           | Entry point receiving `message` & `sessionId` | (External trigger)               | CoinMarketCap Exchange and Community Agent |                                                                                              |
| CoinMarketCap Exchange and Community Agent | LangChain Agent                   | Orchestrates AI query processing and API tool calls | When Executed by Another Workflow, Exchange and Community Agent Brain, Exchange and Community Agent Memory | Exchange Map, Exchange Info, Exchange Assets, CMC 100 Index, Fear and Greed Latest |                                                                                              |
| Exchange and Community Agent Brain | LM Chat OpenAI (GPT-4o-mini)      | Processes natural language queries     | Exchange and Community Agent Memory | CoinMarketCap Exchange and Community Agent |                                                                                              |
| Exchange and Community Agent Memory | Memory Buffer Window               | Maintains session conversation context | Exchange and Community Agent Brain | Exchange and Community Agent Brain |                                                                                              |
| Exchange Map                      | Tool HTTP Request                  | Retrieves exchange IDs, names, slugs   | CoinMarketCap Exchange and Community Agent | CoinMarketCap Exchange and Community Agent |                                                                                              |
| Exchange Info                     | Tool HTTP Request                  | Fetches exchange metadata               | CoinMarketCap Exchange and Community Agent | CoinMarketCap Exchange and Community Agent |                                                                                              |
| Exchange Assets                   | Tool HTTP Request                  | Retrieves exchange token holdings       | CoinMarketCap Exchange and Community Agent | CoinMarketCap Exchange and Community Agent |                                                                                              |
| CMC 100 Index                    | Tool HTTP Request                  | Provides CoinMarketCap 100 Index data   | CoinMarketCap Exchange and Community Agent | CoinMarketCap Exchange and Community Agent |                                                                                              |
| Fear and Greed Latest             | Tool HTTP Request                  | Provides latest market sentiment score  | CoinMarketCap Exchange and Community Agent | CoinMarketCap Exchange and Community Agent |                                                                                              |
| Exchange & Community Guide        | Sticky Note                       | Documentation: agent overview and tools | None                             | None                            | Explains agent purpose and component connections                                             |
| Usage & Examples                  | Sticky Note                       | Documentation: usage instructions and examples | None                             | None                            | Walkthrough for sample use cases                                                             |
| Errors & Licensing                | Sticky Note                       | Documentation: error codes and licensing | None                             | None                            | Includes API error code reference and licensing details                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `Execute Workflow Trigger` node named `When Executed by Another Workflow`.  
   - Configure inputs: `message` (string), `sessionId` (string).

2. **Add AI Language Model Node:**  
   - Add `LM Chat OpenAI` node named `Exchange and Community Agent Brain`.  
   - Set model to `gpt-4o-mini`.  
   - Connect credentials for OpenAI API (OAuth or API key).  
   - No special options needed.

3. **Add Memory Node:**  
   - Add `Memory Buffer Window` node named `Exchange and Community Agent Memory`.  
   - Default settings; no parameters required.

4. **Add LangChain Agent Node:**  
   - Add `LangChain Agent` node named `CoinMarketCap Exchange and Community Agent`.  
   - Bind input text to `{{$json.message}}` from trigger node.  
   - Paste system message describing agent role, supported tools, usage, and error handling (as per overview).  
   - Set prompt type to `define`.  
   - Connect AI Brain node as `ai_languageModel` input.  
   - Connect Memory node as `ai_memory` input.

5. **Add API Tool Nodes:**  
   - For each CoinMarketCap endpoint, add a `Tool HTTP Request` node with the following configurations:

   a. **Exchange Map**  
      - URL: `https://pro-api.coinmarketcap.com/v1/exchange/map`  
      - Query parameter: `slug` (optional)  
      - Authentication: HTTP Header Auth with CoinMarketCap API key  
      - Headers: `Accept: application/json`

   b. **Exchange Info**  
      - URL: `https://pro-api.coinmarketcap.com/v1/exchange/info`  
      - Query parameter: `id` (required)  
      - Authentication and headers as above.

   c. **Exchange Assets**  
      - URL: `https://pro-api.coinmarketcap.com/v1/exchange/assets`  
      - Query parameter: `id` (required)  
      - Authentication and headers as above.

   d. **CMC 100 Index**  
      - URL: `https://pro-api.coinmarketcap.com/v3/index/cmc100-latest`  
      - No query parameters  
      - Authentication and headers as above.

   e. **Fear and Greed Latest**  
      - URL: `https://pro-api.coinmarketcap.com/v3/fear-and-greed/latest`  
      - No query parameters  
      - Authentication and headers as above.

6. **Connect API Tool Nodes:**  
   - Connect the `CoinMarketCap Exchange and Community Agent` node’s `ai_tool` outputs to each API tool node respectively.

7. **Connect Trigger to AI Agent:**  
   - Connect `When Executed by Another Workflow` node’s main output to `CoinMarketCap Exchange and Community Agent` node’s main input.

8. **Connect AI Brain and Memory to Agent:**  
   - Connect `Exchange and Community Agent Brain` node output to `CoinMarketCap Exchange and Community Agent` node’s `ai_languageModel` input.  
   - Connect `Exchange and Community Agent Memory` node output to `CoinMarketCap Exchange and Community Agent` node’s `ai_memory` input.

9. **Add Sticky Notes for Documentation:**  
   - Create three sticky notes with content for:  
     - Exchange & Community Guide  
     - Usage & Examples  
     - Errors & Licensing  
   - Position for easy reference.

10. **Credential Setup:**  
    - Create HTTP Header Auth credential in n8n named e.g. `CoinMarketCap Standard`.  
    - Store your CoinMarketCap API key in the header `X-CMC_PRO_API_KEY`.  
    - Assign this credential to all HTTP Request nodes.

11. **Testing:**  
    - Trigger workflow manually or via supervisor with inputs:  
      - `message`: e.g., "Get all exchanges"  
      - `sessionId`: unique session string  
    - Verify AI Agent processes query and calls appropriate API tool nodes.  
    - Check outputs for correctness.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This agent is part of a broader **CoinMarketCap AI Analyst System** with multiple sub-agents and supervisor flows.   | Visit [Creator profile](https://n8n.io/creators/don-the-gem-dealer/) to download all components. |
| Setup requires a CoinMarketCap API key: [https://coinmarketcap.com/api/](https://coinmarketcap.com/api/)              | Official API key application page.                                                              |
| Use HTTP Header Auth credential type in n8n to securely store API keys.                                              | n8n credential management documentation.                                                        |
| Error codes to watch for: 400 (Bad Request), 401 (Unauthorized), 429 (Rate Limit), 500 (Server Error).                | Included in Errors & Licensing sticky note.                                                     |
| Large API responses may exceed GPT token limits; refine queries with parameters like `limit`, `start`, or `id`.      | Agent system message includes this error handling advice.                                       |
| Connect with the workflow author on LinkedIn: [http://linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr) | For support and updates.                                                                         |
| Copyright © 2025 Treasurium Capital Limited Company. Licensed use only; unauthorized reproduction prohibited.         | Legal and licensing information in sticky note.                                                |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and extend the CoinMarketCap Exchange and Community Agent workflow with confidence.