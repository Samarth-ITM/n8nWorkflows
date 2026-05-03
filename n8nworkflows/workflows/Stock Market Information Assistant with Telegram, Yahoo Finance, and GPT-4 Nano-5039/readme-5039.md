Stock Market Information Assistant with Telegram, Yahoo Finance, and GPT-4 Nano

https://n8nworkflows.xyz/workflows/stock-market-information-assistant-with-telegram--yahoo-finance--and-gpt-4-nano-5039


# Stock Market Information Assistant with Telegram, Yahoo Finance, and GPT-4 Nano

### 1. Workflow Overview

This workflow functions as a **Stock Market Information Assistant** integrating Telegram, Yahoo Finance data retrieval, and GPT-4 Nano AI capabilities via OpenRouter. Its primary purpose is to receive user queries about stocks via Telegram, process these queries with AI, fetch real-time stock information and related data, and respond back through Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Capture user messages from Telegram and trigger the workflow.
- **1.2 AI Processing & Memory:** Use GPT-4 Nano (OpenRouter Chat Model) with memory buffer and a specialized Stock Market Agent to interpret user requests and decide on actions.
- **1.3 Data Retrieval:** Query Yahoo Finance for stock information and perform web searches to extract additional relevant data.
- **1.4 Data Processing:** Extract and refine ticker information and stock details.
- **1.5 Output Delivery:** Send processed information back to the user on Telegram.
- **1.6 External Invocation:** Support execution triggered by other workflows for integration and extensibility.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming messages from Telegram users, serving as the primary input point for stock-related queries.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger node  
    - *Role:* Listens to incoming Telegram messages directed at the bot, triggering the workflow upon receiving a message.  
    - *Configuration:* Uses a webhook ID to receive updates; no additional parameters configured here.  
    - *Inputs:* External Telegram messages via webhook.  
    - *Outputs:* Passes message data downstream to the Stock Market Agent node.  
    - *Potential Failures:* Telegram API downtime, webhook misconfiguration, message format errors.

#### 2.2 AI Processing & Memory

- **Overview:**  
  This block processes user input with GPT-4 Nano via OpenRouter, maintaining conversational context using a memory buffer, and coordinates specific AI-driven tools to interpret and respond to stock queries.

- **Nodes Involved:**  
  - Stock Market Agent  
  - OpenRouter Chat Model  
  - Simple Memory

- **Node Details:**

  - **Stock Market Agent**  
    - *Type:* LangChain Agent node  
    - *Role:* Central AI agent that orchestrates understanding of user input, decision-making, and tool invocation.  
    - *Configuration:* Connected to AI language model, AI memory, and AI tools (Get Stock Info, Get Ticker Name).  
    - *Inputs:* User input from Telegram Trigger; AI memory and tools outputs.  
    - *Outputs:* Final AI-generated response to Telegram node.  
    - *Edge Cases:* Misinterpretation of user intent, failure in invoking tools, latency in AI response.

  - **OpenRouter Chat Model**  
    - *Type:* LangChain OpenRouter Chat Model node  
    - *Role:* Provides GPT-4 Nano language model capabilities for the Stock Market Agent.  
    - *Configuration:* No extra parameters set explicitly; defaults apply.  
    - *Inputs:* Receives prompts from the agent.  
    - *Outputs:* AI-generated text responses.  
    - *Failures:* API key invalidation, rate limits, response timeouts.

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window node  
    - *Role:* Maintains short-term conversational context to enable multi-turn dialogues.  
    - *Configuration:* Default window size and buffer settings.  
    - *Inputs:* Conversation history from the agent.  
    - *Outputs:* Contextual memory for the agent to use.  
    - *Edge Cases:* Memory overflow, context truncation leading to loss of important info.

#### 2.3 Data Retrieval

- **Overview:**  
  This block obtains stock-related data by querying Yahoo Finance and performing Google searches to supplement information.

- **Nodes Involved:**  
  - Get Stock Info  
  - Get Ticker Name  
  - Google Search

- **Node Details:**

  - **Get Stock Info**  
    - *Type:* HTTP Request Tool node  
    - *Role:* Queries Yahoo Finance API or similar data sources for detailed stock information.  
    - *Configuration:* Uses API endpoint configured in HTTP Request Tool; parameters dynamically set by agent tools.  
    - *Inputs:* Requests from Stock Market Agent.  
    - *Outputs:* Raw stock data (price, market cap, etc.) sent back to agent.  
    - *Failures:* API downtime, invalid ticker, network errors.

  - **Get Ticker Name**  
    - *Type:* LangChain Tool Workflow node  
    - *Role:* Resolves ambiguous stock names or queries into valid ticker symbols.  
    - *Configuration:* Calls a sub-workflow designed to map stock names to ticker symbols.  
    - *Inputs:* Agent tool invocation with user input or extracted text.  
    - *Outputs:* Valid ticker symbols for further data retrieval.  
    - *Edge Cases:* Unknown stock names, multiple matches, sub-workflow failures.

  - **Google Search**  
    - *Type:* HTTP Request node  
    - *Role:* Performs web search to gather supplementary information related to the stock or query.  
    - *Configuration:* Uses a generic HTTP GET request to a search API or Google Custom Search.  
    - *Inputs:* Triggered externally or via the "When Executed by Another Workflow" node.  
    - *Outputs:* Search results forwarded for extraction.  
    - *Failures:* API limits, malformed queries, network issues.

#### 2.4 Data Processing

- **Overview:**  
  This block extracts precise information from raw search and stock data, identifies ticker symbols, and prepares data for final output.

- **Nodes Involved:**  
  - Information Extractor  
  - Ticker Name Search  
  - Set Ticker  
  - OpenRouter Chat Model1

- **Node Details:**

  - **Information Extractor**  
    - *Type:* LangChain Information Extractor node  
    - *Role:* Parses Google Search results or other data to extract relevant stock-related entities (e.g., ticker names).  
    - *Configuration:* Uses AI language model (OpenRouter Chat Model1) for extraction logic.  
    - *Inputs:* Data from Google Search.  
    - *Outputs:* Structured information to Ticker Name Search node.  
    - *Edge Cases:* Extraction errors, ambiguous data, unexpected input formats.

  - **Ticker Name Search**  
    - *Type:* HTTP Request node  
    - *Role:* Searches for detailed ticker information based on extracted entities.  
    - *Configuration:* HTTP GET with dynamic parameters from extracted data.  
    - *Inputs:* Extracted ticker info from Information Extractor.  
    - *Outputs:* Results passed to Set Ticker node.  
    - *Failures:* Invalid ticker queries, API errors.

  - **Set Ticker**  
    - *Type:* Set node  
    - *Role:* Finalizes and formats the ticker symbol for use in subsequent queries or responses.  
    - *Configuration:* Defines variables or fields representing the ticker symbol.  
    - *Inputs:* Output from Ticker Name Search.  
    - *Outputs:* Prepared ticker data for further processing or output.  
    - *Edge Cases:* Missing or malformed ticker data.

  - **OpenRouter Chat Model1**  
    - *Type:* LangChain OpenRouter Chat Model node  
    - *Role:* Supports the Information Extractor node with AI-powered text understanding.  
    - *Configuration:* Similar to the primary OpenRouter Chat Model but dedicated to extraction tasks.  
    - *Inputs:* Prompts from Information Extractor.  
    - *Outputs:* Extracted entities or structured text data.  
    - *Failures:* API errors or response misinterpretation.

#### 2.5 Output Delivery

- **Overview:**  
  Sends the final, AI-curated stock information back to the Telegram user.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram node  
    - *Role:* Sends messages to users on Telegram with the final response from the Stock Market Agent.  
    - *Configuration:* Uses appropriate bot credentials and chat identifiers to send messages.  
    - *Inputs:* AI-generated replies from Stock Market Agent.  
    - *Outputs:* External user-facing messages.  
    - *Failures:* Telegram API errors, message size limits, chat ID errors.

#### 2.6 External Invocation

- **Overview:**  
  Allows this workflow to be triggered by other workflows, enabling integration in larger automation scenarios.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Google Search (connected downstream)

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger node  
    - *Role:* Entry point for external workflow calls, triggering Google Search and subsequent data extraction.  
    - *Configuration:* No parameters; simply triggers downstream nodes.  
    - *Inputs:* External workflow execution calls.  
    - *Outputs:* Initiates Google Search node.  
    - *Edge Cases:* Invocation without required parameters, timing issues.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                     | Input Node(s)                     | Output Node(s)                   | Sticky Note                                  |
|-----------------------------|-----------------------------------------|-----------------------------------|----------------------------------|---------------------------------|----------------------------------------------|
| Telegram Trigger            | Telegram Trigger                        | Input reception from Telegram     | —                                | Stock Market Agent              |                                              |
| Stock Market Agent          | LangChain Agent                        | AI processing and orchestration   | Telegram Trigger, Simple Memory, OpenRouter Chat Model, Get Stock Info, Get Ticker Name | Telegram                       |                                              |
| OpenRouter Chat Model       | LangChain OpenRouter Chat Model        | AI language model for agent       | Stock Market Agent               | Stock Market Agent              |                                              |
| Simple Memory               | LangChain Memory Buffer Window          | Conversational context memory     | Stock Market Agent               | Stock Market Agent              |                                              |
| Get Stock Info              | HTTP Request Tool                      | Fetch stock data from Yahoo Finance | Stock Market Agent             | Stock Market Agent              |                                              |
| Get Ticker Name             | LangChain Tool Workflow                | Resolve stock name to ticker      | Stock Market Agent               | Stock Market Agent              |                                              |
| Telegram                   | Telegram                               | Sends final message to user       | Stock Market Agent               | —                              |                                              |
| When Executed by Another Workflow | Execute Workflow Trigger           | External workflow invocation      | —                              | Google Search                  |                                              |
| Google Search              | HTTP Request                          | Web search for supplementary info | When Executed by Another Workflow | Information Extractor         |                                              |
| Information Extractor      | LangChain Information Extractor        | Extract entities from search data | Google Search                   | Ticker Name Search             |                                              |
| Ticker Name Search         | HTTP Request                          | Search for ticker details          | Information Extractor           | Set Ticker                    |                                              |
| Set Ticker                 | Set                                   | Format and finalize ticker symbol | Ticker Name Search             | —                             |                                              |
| OpenRouter Chat Model1     | LangChain OpenRouter Chat Model        | AI model for information extraction | Information Extractor          | Information Extractor          |                                              |
| Sticky Note                | Sticky Note                           | N/A                               | —                              | —                             |                                              |
| Sticky Note1               | Sticky Note                           | N/A                               | —                              | —                             |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook with your Telegram bot credentials.  
   - No additional parameters needed.  
   - Position: Start of the workflow.

2. **Create Stock Market Agent Node**  
   - Type: LangChain Agent  
   - Connect Telegram Trigger node output to this node input.  
   - Configure AI language model, memory, and tools connections (set below).  
   - Position: Next after Telegram Trigger.

3. **Create OpenRouter Chat Model Node**  
   - Type: LangChain OpenRouter Chat Model  
   - Provide OpenRouter API credentials with GPT-4 Nano model access.  
   - Connect this node to the Stock Market Agent's AI language model input.

4. **Create Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Default settings (or configure window size as desired).  
   - Connect output to Stock Market Agent's AI memory input.

5. **Create Get Stock Info Node**  
   - Type: HTTP Request Tool  
   - Configure to call Yahoo Finance API or equivalent with dynamic ticker symbol.  
   - Connect output to Stock Market Agent’s AI tool input.

6. **Create Get Ticker Name Node**  
   - Type: LangChain Tool Workflow  
   - Link to a sub-workflow that resolves stock names to ticker symbols (must be created separately).  
   - Connect output to Stock Market Agent’s AI tool input.

7. **Create Telegram Node**  
   - Type: Telegram  
   - Configure with Telegram bot credentials for sending messages.  
   - Connect Stock Market Agent’s main output to this node input.

8. **Create When Executed by Another Workflow Node**  
   - Type: Execute Workflow Trigger  
   - Place separately to allow external triggering.  
   - Connect to Google Search node.

9. **Create Google Search Node**  
   - Type: HTTP Request  
   - Configure API for web search (e.g., Google Custom Search API) with dynamic query parameters.  
   - Connect output to Information Extractor node.

10. **Create Information Extractor Node**  
    - Type: LangChain Information Extractor  
    - Connect Google Search output to this node input.  
    - Connect this node’s output to Ticker Name Search node.  
    - Link AI language model input to OpenRouter Chat Model1.

11. **Create OpenRouter Chat Model1 Node**  
    - Type: LangChain OpenRouter Chat Model  
    - Configure with same OpenRouter credentials.  
    - Connect to Information Extractor node’s AI language model input.

12. **Create Ticker Name Search Node**  
    - Type: HTTP Request  
    - Configure to query for ticker details based on extracted info.  
    - Connect output to Set Ticker node.

13. **Create Set Ticker Node**  
    - Type: Set  
    - Configure fields to finalize ticker symbol for further processing.  
    - Connect as final step in data processing chain.

14. **Make all connections as per the workflow diagram:**  
    - Telegram Trigger → Stock Market Agent  
    - Stock Market Agent → Telegram  
    - Stock Market Agent → OpenRouter Chat Model (AI language model)  
    - Stock Market Agent → Simple Memory (AI memory)  
    - Stock Market Agent → Get Stock Info (AI tool)  
    - Stock Market Agent → Get Ticker Name (AI tool)  
    - When Executed by Another Workflow → Google Search → Information Extractor → Ticker Name Search → Set Ticker

15. **Set Credentials:**  
    - Telegram nodes: Provide Telegram Bot API credentials.  
    - OpenRouter Chat Model nodes: Provide OpenRouter API key with GPT-4 Nano access.  
    - HTTP Request nodes: Configure API keys for Yahoo Finance, Google Custom Search, or other data sources.  
    - Sub-workflow for Get Ticker Name: Must be created and linked properly.

16. **Verify node versions:**  
    - Use latest compatible versions as indicated (e.g., Telegram Trigger v1.2, LangChain nodes v1+).  
    - Test for any API or credential changes.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow leverages GPT-4 Nano via OpenRouter for advanced AI language understanding.    | OpenRouter: https://openrouter.ai/                                                                   |
| Yahoo Finance data is accessed via HTTP Request Tool; ensure API endpoint and key validity.  | Yahoo Finance API documentation or alternative finance APIs.                                         |
| Telegram integration requires bot creation and webhook setup.                               | Telegram Bot API: https://core.telegram.org/bots/api                                                 |
| The workflow supports multi-turn conversations with memory buffer for better context.       | LangChain Memory Buffer Window node documentation.                                                  |
| Sub-workflow for ticker name resolution is essential and must handle ambiguous inputs well. | Consider building robust name-to-ticker mapping logic with fallback strategies.                      |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.