Binance SM 1hour Indicators Tool

https://n8nworkflows.xyz/workflows/binance-sm-1hour-indicators-tool-4744


# Binance SM 1hour Indicators Tool

### 1. Workflow Overview

This workflow, titled **Binance SM 1hour Indicators Tool**, is designed to calculate, interpret, and summarize technical trading indicators on a 1-hour candlestick basis for any Binance Spot Market trading pair. It is primarily used to support swing-trading strategies by providing medium-term trend signals and momentum insights. The workflow does not initiate autonomously; instead, it is triggered by other workflows or agents such as the Binance SM Financial Analyst Tool or Binance Quant AI Agent.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives trigger input from other workflows containing the trading symbol and session ID.
- **1.2 Indicator Retrieval Agent:** Processes the input symbol, calls an internal HTTP webhook service that calculates multiple 1-hour technical indicators for the symbol, and returns raw indicator data.
- **1.3 AI Interpretation and Memory:** Uses OpenAI GPT-4.1-mini to translate raw indicator data into natural-language summaries and signal labels; maintains session context including symbol and previous queries for persistent multi-turn dialogue.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block serves as the entry point, receiving input from external workflows. It expects a JSON payload containing a `message` (the Binance symbol) and a `sessionId` (user or session identifier).

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: Trigger node for sub-workflows  
    - Configuration: Accepts two inputs — `message` (string) and `sessionId` (string) — from the calling workflow.  
    - Inputs: External workflow trigger  
    - Outputs: Passes input data to the next node  
    - Edge Cases: Missing or malformed input JSON; symbol validation is deferred downstream.  
    - Notes: This node ensures the workflow only runs when explicitly triggered by authorized workflows/agents.

#### 2.2 Indicator Retrieval Agent

- **Overview:**  
  This block extracts the trading symbol from the input, formats a POST request to an external webhook service that calculates multiple 1-hour technical indicators for the symbol, and gathers the raw indicator data.

- **Nodes Involved:**  
  - Binance SM 1hour Indicators Agent  
  - HTTP Request 1h Indicators Tool

- **Node Details:**  
  - **Binance SM 1hour Indicators Agent**  
    - Type: LangChain agent node  
    - Configuration:  
      - Receives the raw input text (`message`) containing the symbol.  
      - Uses a detailed system message describing its role as the "Binance 1-Hour Technical Indicator Agent," including the connected tools it will call, expected input format, and behavior.  
      - It formats and triggers the HTTP Request node to fetch indicator data.  
    - Expressions: Extracts symbol from input JSON (`{{$json.message}}`)  
    - Inputs: Data from the trigger node  
    - Outputs: JSON indicator data to next node  
    - Edge Cases: Invalid symbol format, HTTP request failures, insufficient data from Binance (less than 40 candles), partial data availability (some indicators fail).  
    - Notes: Contains detailed internal instructions for error handling and data validation.

  - **HTTP Request 1h Indicators Tool**  
    - Type: HTTP Request node  
    - Configuration:  
      - POST request to `https://treasurium.app.n8n.cloud/webhook/1h-indicators`  
      - Body contains JSON with the `symbol` parameter extracted dynamically via an AI override expression.  
    - Inputs: Triggered internally by the agent node  
    - Outputs: Raw JSON containing calculated indicators: RSI(14), MACD(12/26/9), Bollinger Bands (20, 2 STD), SMA(20), EMA(20), ADX(14) with DI+ and DI−.  
    - Edge Cases: HTTP timeouts, webhook unavailability, malformed response, API rate limits, symbol not found.  
    - Version Requirements: n8n version supporting HTTP Request Tool v4.2 or higher recommended.

#### 2.3 AI Interpretation and Memory

- **Overview:**  
  This block processes the raw indicator data using OpenAI GPT-4.1-mini to generate human-readable technical signal summaries and momentum interpretations. It also maintains session memory to support persistent and multi-turn interactions.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: LangChain Chat Model node using OpenAI GPT model  
    - Configuration:  
      - Model set to `gpt-4.1-mini` for efficient yet high-quality summarization.  
      - Input: Raw indicator JSON data from HTTP request.  
      - Output: Natural-language summaries suitable for display (e.g., Telegram), including overbought/oversold states, crossover signals, trend strength, and short actionable insights.  
    - Inputs: Raw indicator data  
    - Outputs: Text summaries and structured signal labels  
    - Edge Cases: API key invalid or expired, model rate limits, input exceeding token limits, parsing errors.  
    - Credential: Requires valid OpenAI API credentials.  
    - Version: Compatible with LangChain v1.2+ and n8n OpenAI integration.

  - **Simple Memory**  
    - Type: LangChain memory buffer (windowed)  
    - Configuration: Stores session context such as sessionId, symbol, and last queries.  
    - Inputs: Receives session context updates from the agent node.  
    - Outputs: Provides context for multi-turn dialogue and symbol consistency across interactions.  
    - Edge Cases: Memory overflow if session context grows too large, potential data privacy considerations.  
    - Notes: Enables persistent state for user sessions and cross-agent consistency.

---

### 3. Summary Table

| Node Name                       | Node Type                                   | Functional Role                                  | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                   |
|--------------------------------|---------------------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger                    | Entry trigger, receives input symbol & sessionId | External Workflow                | Binance SM 1hour Indicators Agent | This workflow does not start on its own — it is always triggered by: Binance SM Financial Analyst Tool, Binance Quant AI Agent |
| Binance SM 1hour Indicators Agent | LangChain Agent                            | Core logic: processes input, calls indicator webhook | When Executed by Another Workflow | HTTP Request 1h Indicators Tool, Simple Memory, OpenAI Chat Model | You are the Binance 1-Hour Technical Indicator Agent; extracts symbol, calls indicators, returns JSON & labels |
| HTTP Request 1h Indicators Tool | HTTP Request Tool                           | Calls webhook to calculate technical indicators  | Binance SM 1hour Indicators Agent | Binance SM 1hour Indicators Agent (feedback data) | POSTs to https://treasurium.app.n8n.cloud/webhook/1h-indicators; sends symbol; fetches RSI, MACD, BBANDS, SMA, EMA, ADX |
| OpenAI Chat Model               | LangChain OpenAI Chat Model                 | Converts raw indicator data into natural-language summaries | Binance SM 1hour Indicators Agent | Simple Memory                    | Uses GPT-4.1-mini to generate momentum interpretation and Telegram-ready summaries                             |
| Simple Memory                  | LangChain Memory Buffer Window              | Maintains session context for multi-turn interactions | Binance SM 1hour Indicators Agent | N/A                              | Stores session context (sessionId, symbol, last query) for persistent interactions and symbol consistency      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure to accept inputs `message` (string) and `sessionId` (string).  
   - This node will receive input JSON like `{ "message": "ETHUSDT", "sessionId": "telegram_chat_id" }`.

2. **Add LangChain Agent Node**  
   - Add a **LangChain Agent** node named `Binance SM 1hour Indicators Agent`.  
   - Set parameter `text` to `={{ $json.message }}` to extract the symbol.  
   - Define a detailed system message with the following points:  
     - Role as Binance 1-Hour Technical Indicator Agent.  
     - List connected tools (calls to 1h-indicators webhook).  
     - Expected input format with example JSON.  
     - Behavior: Extract symbol, POST to webhook, merge indicator outputs.  
     - Use case scenarios and operational notes.  
   - Connect input from `When Executed by Another Workflow`.

3. **Configure HTTP Request Node**  
   - Add **HTTP Request Tool** node named `HTTP Request 1h Indicators Tool`.  
   - Set method to POST; URL to `https://treasurium.app.n8n.cloud/webhook/1h-indicators`.  
   - Enable sending body parameters; add parameter `symbol` with value `={{ $fromAI('parameters0_Value', ``, 'string') }}` to dynamically receive symbol from AI override in the agent.  
   - Connect output from `Binance SM 1hour Indicators Agent`.

4. **Add OpenAI Chat Model Node**  
   - Add **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.  
   - Select model `gpt-4.1-mini` for balanced performance.  
   - Ensure valid OpenAI API credentials are configured.  
   - Input will be the JSON output of the HTTP Request node containing raw indicators.  
   - Purpose: Generate natural-language signal summaries and momentum interpretations.  
   - Connect output from `Binance SM 1hour Indicators Agent`.

5. **Add Simple Memory Node**  
   - Add **LangChain Memory Buffer Window** node named `Simple Memory`.  
   - Purpose: Store session context (sessionId, symbol, last query).  
   - No specific parameters required, defaults suffice.  
   - Connect to and from `Binance SM 1hour Indicators Agent` for session persistence.

6. **Connect Nodes**  
   - Connect `When Executed by Another Workflow` → `Binance SM 1hour Indicators Agent`.  
   - Connect `Binance SM 1hour Indicators Agent` → `HTTP Request 1h Indicators Tool` (ai_tool).  
   - Connect `Binance SM 1hour Indicators Agent` → `OpenAI Chat Model` (ai_languageModel).  
   - Connect `Binance SM 1hour Indicators Agent` → `Simple Memory` (ai_memory).

7. **Credential Setup**  
   - Configure OpenAI API credentials for `OpenAI Chat Model` node.  
   - No special credentials needed for HTTP Request node as webhook is public or has its own auth logic (not specified).  
   - No credentials needed for trigger or memory nodes.

8. **Testing and Validation**  
   - Trigger the workflow via an external workflow sending JSON with `message` (e.g., "BTCUSDT") and `sessionId`.  
   - Verify that the HTTP webhook responds with indicator data.  
   - Confirm that OpenAI returns summarized signals.  
   - Check memory retains session context properly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow is a core component of swing-trade setup validation, providing structured and interpreted 1-hour technical indicators for Binance Spot Market pairs. It is designed to be triggered only by authorized sub-agents like Binance SM Financial Analyst Tool or Binance Quant AI Agent.                                                                                  | Workflow Trigger Sticky Note                           |
| The backend webhook at `https://treasurium.app.n8n.cloud/webhook/1h-indicators` calculates all key indicators (RSI, MACD, BBANDS, SMA, EMA, ADX) using the latest 40 1-hour candles from Binance.                                                                                                                                                                                | HTTP Request Node Sticky Note                          |
| OpenAI GPT-4.1-mini model is used for translating numeric indicator data into human-readable, Telegram-friendly technical signal summaries, including momentum and trend strength interpretations.                                                                                                                                                                            | OpenAI Chat Model Sticky Note                          |
| Memory node stores session context to allow persistent multi-turn interactions and maintain symbol consistency across queries.                                                                                                                                                                                                                                               | Simple Memory Sticky Note                              |
| Proprietary tool architecture by Don Jayamaha, Treasurium Capital Limited. Commercial use and redistribution require active license. LinkedIn: [http://linkedin.com/in/donjayamahajr](http://linkedin.com/in/donjayamahajr)                                                                                                                                                        | Licensing & Support Note                               |
| The workflow documentation and architecture ensure reliable chaining of API calls, error resilience (e.g., logging failures but returning valid partial data), and consistent timestamp alignment across indicators.                                                                                                                                                           | Comprehensive workflow documentation sticky note      |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.