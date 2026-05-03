Get Binance Spot Market Financial Analysis via Telegram with GPT-4o

https://n8nworkflows.xyz/workflows/get-binance-spot-market-financial-analysis-via-telegram-with-gpt-4o-4741


# Get Binance Spot Market Financial Analysis via Telegram with GPT-4o

### 1. Workflow Overview

This workflow, titled **Get Binance Spot Market Financial Analysis via Telegram with GPT-4o**, is designed to provide comprehensive, AI-driven financial market analysis for Binance spot trading pairs. It accepts natural-language queries (typically via Telegram) referencing cryptocurrency symbols and optional timeframes, then orchestrates multiple specialized sub-agents to deliver a multi-timeframe, indicator-based market snapshot. The final output is a structured, Telegram-optimized summary suitable for traders seeking actionable insights.

The workflow is modular and composed of distinct logical blocks:

- **1.1 Trigger Input**: Receives invocation requests from upstream workflows or bots with user queries and session identifiers.
- **1.2 AI Reasoning & Orchestration Agent**: The core decision engine that parses user input, infers symbol/timeframe, routes calls to multiple indicator and price data agents, and merges results.
- **1.3 Session Memory Management**: Maintains session context to support multi-turn dialogues and stateful analysis.
- **1.4 Price & Market Structure Data Retrieval**: Fetches real-time Binance spot market data including current price, 24h stats, order book, and candlestick (kline) data.
- **1.5 Multi-Timeframe Technical Indicator Agents**: Four parallel sub-agents each analyze a specific timeframe (15min, 1h, 4h, 1d) to compute RSI, MACD, Bollinger Bands, SMA/EMA, and ADX indicators.
- **1.6 AI Model for Final Interpretation**: An OpenAI GPT-4.1-mini chat model interprets raw indicator outputs into meaningful labels and generates the final formatted market analysis report.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger Input

- **Overview:** This block receives external workflow calls, providing the entry point for user queries and session info.
- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: `Execute Workflow Trigger`  
    - Role: Entry trigger node that listens for calls from other workflows (likely Telegram bot or Quant AI Agent).  
    - Configuration: Expects inputs `message` (user query) and `sessionId` (Telegram chat ID).  
    - Inputs: External workflow invocation  
    - Outputs: Passes inputs downstream to the Financial Analyst Agent  
    - Edge cases: Missing or malformed inputs could cause downstream failures; no direct validation here.  
    - Sub-workflow: Triggered externally; this node itself is the entry point.

---

#### 2.2 AI Reasoning & Orchestration Agent

- **Overview:** Acts as the primary orchestrator parsing user input, selecting relevant tools, invoking sub-agents in parallel, and merging results into a final report.
- **Nodes Involved:**  
  - Binance SM Financial Analyst Agent

- **Node Details:**

  - **Binance SM Financial Analyst Agent**  
    - Type: LangChain AI Agent  
    - Role: Core decision engine and router  
    - Configuration:  
      - Receives user message from trigger node  
      - System prompt defines role as a Binance Spot Market Financial Analyst AI Agent  
      - Tasked with parsing natural language to extract symbol and timeframe (with defaults if missing)  
      - Routes calls to connected tools: Price & Market Structure Tool; 15min, 1h, 4h, 1d Technical Indicator Agents  
      - Combines all responses into a structured Telegram-formatted report  
      - Prompt includes detailed instructions and output formatting template  
    - Inputs: `message` and `sessionId` from trigger  
    - Outputs: Final structured market analysis  
    - Edge cases:  
      - User inputs missing symbol trigger prompt for clarification  
      - Timeframe inference fallback logic if none specified  
      - Potential failures if sub-agents return malformed data or timeouts occur  
      - AI interpretation errors if unexpected input is received  
    - Sub-workflow: This node orchestrates calls to multiple sub-agent workflows listed below.

---

#### 2.3 Session Memory Management

- **Overview:** Maintains a session-based memory buffer to track user queries, session context, and past indicator values to support multi-turn dialogue and consistent analysis.
- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Stores session ID, user message, and intermediate indicator values for context retention  
    - Configuration: Default buffer window memory (no additional parameters)  
    - Inputs: Connected from Financial Analyst Agent (AI memory channel)  
    - Outputs: Provides stored context back for AI reasoning  
    - Edge cases: Memory overflow or loss if session data grows too large or is not properly purged.

---

#### 2.4 Price & Market Structure Data Retrieval

- **Overview:** Retrieves raw market data from Binance for the requested trading symbol, including current price, 24h statistics, order book depth, and recent candlestick data.
- **Nodes Involved:**  
  - Binance SM Price-24hrStats-OrderBook-Kline Agent

- **Node Details:**

  - **Binance SM Price-24hrStats-OrderBook-Kline Agent**  
    - Type: LangChain Tool - Sub-agent Workflow  
    - Role: Provides low-level Binance market data on demand  
    - Configuration:  
      - Invokes the sub-workflow named `Binance SM Price-24hrStats-OrderBook-Kline Tool` (workflow ID supplied)  
      - Input: Takes `message` (symbol) and `sessionId` forwarded from AI agent  
      - Output: Returns structured JSON with:  
        - Current price  
        - 24-hour statistics (change %, high, low, volume)  
        - Order book snapshot (top 100 bids & asks)  
        - Various kline intervals for deeper analysis  
    - Inputs: From Financial Analyst Agent (tool call)  
    - Outputs: Market data JSON to AI agent  
    - Edge cases: API rate limits, symbol not found, network errors, malformed data from Binance API.

---

#### 2.5 Multi-Timeframe Technical Indicator Agents

- **Overview:** Four parallel sub-agents each analyze Binance kline data at a specific timeframe to compute key technical indicators for market trend and momentum analysis.
- **Nodes Involved:**  
  - Binance SM 15min Indicators Agent  
  - Binance SM 1hour Indicators Agent  
  - Binance SM 4hour Indicators Agent  
  - Binance SM 1day Indicators Agent

- **Node Details:**

  - **Binance SM 15min Indicators Agent**  
    - Type: LangChain Tool - Sub-agent Workflow  
    - Role: Short-term technical indicator analysis on 15-minute candlesticks  
    - Configuration: Calls dedicated 15min indicator workflow  
    - Inputs: `message` (symbol), `sessionId`  
    - Indicators calculated: RSI(14), MACD(12/26/9), Bollinger Bands (20/2), SMA/EMA(20), ADX(14)  
    - Edge cases: Insufficient data points, API errors, symbol issues

  - **Binance SM 1hour Indicators Agent**  
    - Type: LangChain Tool - Sub-agent Workflow  
    - Role: Intraday momentum and confirmation on 1-hour candles  
    - Same indicator set and inputs as above, different timeframe

  - **Binance SM 4hour Indicators Agent**  
    - Type: LangChain Tool - Sub-agent Workflow  
    - Role: Medium-term swing trend analysis on 4-hour candles  
    - Same indicator set and inputs

  - **Binance SM 1day Indicators Agent**  
    - Type: LangChain Tool - Sub-agent Workflow  
    - Role: Long-term trend and sentiment on daily candles  
    - Same indicator set and inputs

  - Common edge cases for all: Delays or failures in upstream workflows, invalid symbols, partial or missing data, API throttling.

---

#### 2.6 AI Model for Final Interpretation

- **Overview:** Uses OpenAI GPT-4.1-mini model to interpret raw indicator and market data, label technical conditions, and produce a clean, Telegram-ready textual market snapshot.
- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Final reasoning and formatting engine  
    - Configuration:  
      - Model selected: `gpt-4.1-mini` or compatible  
      - No special prompt parameters beyond those embedded in the orchestrator agent  
    - Inputs: Receives merged data from the Financial Analyst Agent outputs  
    - Outputs: Telegram-optimized market snapshot text  
    - Edge cases: OpenAI API rate limits, prompt interpretation errors, incomplete data leading to poor summaries.

---

### 3. Summary Table

| Node Name                                  | Node Type                           | Functional Role                              | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                                             |
|--------------------------------------------|-----------------------------------|----------------------------------------------|-------------------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow           | Execute Workflow Trigger           | Entry trigger for external calls              | External workflow                   | Binance SM Financial Analyst Agent  | Trigger: Parent Workflow Call — Invoked by higher-level agents like Binance Quant AI Agent                                               |
| Binance SM Financial Analyst Agent          | LangChain AI Agent                 | Core orchestrator & router                     | When Executed by Another Workflow   | Simple Memory, OpenAI Chat Model, Price Agent, Indicator Agents | Primary Reasoning Agent — Parses input, selects tools, merges results                                                                    |
| Simple Memory                               | LangChain Memory Buffer Window    | Session state management                        | Binance SM Financial Analyst Agent  | Binance SM Financial Analyst Agent  | Memory Context — Stores session ID, message, and indicator data for multi-turn dialogue                                                  |
| Binance SM Price-24hrStats-OrderBook-Kline Agent | LangChain Tool Workflow           | Fetches Binance price, 24h stats, order book, klines | Binance SM Financial Analyst Agent | Binance SM Financial Analyst Agent  | Price & Structure Tool — Provides raw market data for AI formatting                                                                     |
| Binance SM 15min Indicators Agent           | LangChain Tool Workflow           | Short-term technical indicators (15min candles) | Binance SM Financial Analyst Agent | Binance SM Financial Analyst Agent  | Technical Indicator Agents — 15m Agent for fast intraday signals                                                                         |
| Binance SM 1hour Indicators Agent            | LangChain Tool Workflow           | Mid-term momentum indicators (1h candles)     | Binance SM Financial Analyst Agent | Binance SM Financial Analyst Agent  | Technical Indicator Agents — 1h Agent for momentum confirmation                                                                           |
| Binance SM 4hour Indicators Agent            | LangChain Tool Workflow           | Medium-term swing trend indicators (4h candles) | Binance SM Financial Analyst Agent | Binance SM Financial Analyst Agent  | Technical Indicator Agents — 4h Agent for macro swing trend analysis                                                                     |
| Binance SM 1day Indicators Agent             | LangChain Tool Workflow           | Long-term trend & sentiment indicators (1d candles) | Binance SM Financial Analyst Agent | Binance SM Financial Analyst Agent  | Technical Indicator Agents — 1d Agent for high-timeframe sentiment                                                                       |
| OpenAI Chat Model                           | LangChain OpenAI Chat Model       | Final AI reasoning and report formatting       | Binance SM Financial Analyst Agent  | (External output)                   | AI Engine — Interprets indicator values and formats final Telegram summary                                                               |
| Sticky Note                                 | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Trigger: Parent Workflow Call - Explains overall usage and trigger                                                                      |
| Sticky Note1                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Memory Context - Explains role of Simple Memory node                                                                                    |
| Sticky Note2                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | AI Engine - Explains role of OpenAI Chat Model                                                                                          |
| Sticky Note3                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Price & Structure Tool - Explains Price Agent node                                                                                      |
| Sticky Note4                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Technical Indicator Agents - 15min Agent explanation                                                                                    |
| Sticky Note5                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Technical Indicator Agents - 1h Agent explanation                                                                                       |
| Sticky Note6                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Technical Indicator Agents - 4h Agent explanation                                                                                       |
| Sticky Note7                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Technical Indicator Agents - 1d Agent explanation                                                                                       |
| Sticky Note8                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Primary Reasoning Agent explanation                                                                                                    |
| Sticky Note9                                | Sticky Note                      | Documentation notes                            | N/A                                 | N/A                                 | Full workflow documentation with purpose, connected sub-agents, architecture, timeframe logic, output format, use cases, and credits  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **Execute Workflow Trigger** node  
   - Configure to accept inputs: `message` (string), `sessionId` (string)  
   - Position it as workflow entry point

2. **Create Core AI Orchestrator Agent**  
   - Add **LangChain Agent** node  
   - Name it `Binance SM Financial Analyst Agent`  
   - Set parameters:  
     - Input text: `={{ $json.message }}`  
     - System prompt: Define role as Binance Spot Market Financial Analyst AI Agent with instructions for parsing symbol/timeframe, routing to tools, merging results, and formatting output as per Telegram template  
   - Connect input from trigger node  
   - Configure AI model references and set to version 2

3. **Add Session Memory Node**  
   - Add **LangChain Memory Buffer Window** node named `Simple Memory`  
   - Connect AI memory channel from the Financial Analyst Agent node to this node  
   - No extra configuration needed

4. **Add OpenAI Chat Model Node**  
   - Add **LangChain OpenAI Chat Model** node  
   - Set model to `gpt-4.1-mini` or compatible GPT-4 model  
   - Connect AI language model channel from Financial Analyst Agent node to this node  
   - Configure OpenAI credentials (API key setup required)

5. **Add Price & Market Structure Tool Node**  
   - Add **LangChain Tool Workflow** node named `Binance SM Price-24hrStats-OrderBook-Kline Agent`  
   - Link to sub-workflow `Binance SM Price-24hrStats-OrderBook-Kline Tool` by its workflow ID  
   - Map inputs:  
     - `message`: populated dynamically from AI agent output  
     - `sessionId`: forwarded from main input  
   - Connect AI tool channel from Financial Analyst Agent node to this node

6. **Add Four Technical Indicator Agent Nodes**  
   - Add four **LangChain Tool Workflow** nodes named:  
     - `Binance SM 15min Indicators Agent` (workflow ID: 15min indicators tool)  
     - `Binance SM 1hour Indicators Agent` (workflow ID: 1h indicators tool)  
     - `Binance SM 4hour Indicators Agent` (workflow ID: 4h indicators tool)  
     - `Binance SM 1day Indicators Agent` (workflow ID: 1d indicators tool)  
   - For each:  
     - Map inputs `message` and `sessionId` from AI agent output  
     - Connect AI tool channel from Financial Analyst Agent node to each indicator node

7. **Set Credentials**  
   - Configure OpenAI API credentials for AI nodes  
   - Ensure any Binance API or other API credentials required by sub-workflows are correctly set up

8. **Connect Outputs**  
   - From all sub-agent nodes (price and indicators), route outputs back to the Financial Analyst Agent for aggregation  
   - Output final formatted text from OpenAI Chat Model as the final workflow output

9. **Test and Validate**  
   - Test with sample Telegram-style queries such as “How is BTCUSDT doing today?”  
   - Verify correct symbol parsing, multi-timeframe indicator fetching, and clean output format

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This agent is always triggered by a higher-level workflow, such as the Binance Quant AI Agent. It does not run standalone.                                                                                                                                                                                                                                                                                                                                   | Sticky Note: Trigger: Parent Workflow Call                                                      |
| Session memory enables multi-turn dialogues, trend comparison across timeframes, and report reuse if queries repeat.                                                                                                                                                                                                                                                                                                                                         | Sticky Note: Memory Context                                                                     |
| The core AI model is GPT-4.1-mini, responsible for interpreting indicator values with labels like “Overbought,” “MACD Cross Up,” and generating final Telegram-ready snapshots.                                                                                                                                                                                                                                                                              | Sticky Note: AI Engine                                                                          |
| The price & structure tool returns clean, Telegram-optimized market data including current price, 24h stats, order book depths, and kline snapshots for multiple intervals.                                                                                                                                                                                                                                                                                 | Sticky Note: Price & Structure Tool                                                            |
| Each technical indicator agent computes RSI(14), MACD(12/26/9), Bollinger Bands(20/2), SMA/EMA(20), and ADX(14) for its respective timeframe: 15min (fast intraday), 1h (momentum confirmation), 4h (swing trend), 1d (long-term sentiment).                                                                                                                                                                                                                  | Sticky Notes: Technical Indicator Agents (multiple notes)                                      |
| Final output is formatted as a Telegram-friendly text summary with price, 24h change, volume, multiple timeframe indicator summaries, and key support/resistance levels.                                                                                                                                                                                                                                                                                     | Sticky Note: Full Workflow Documentation                                                      |
| Workflow and intellectual property are proprietary to Treasurium Capital Limited Company. Contact Don Jayamaha on LinkedIn for support and licensing inquiries.                                                                                                                                                                                                                                                                                             | [LinkedIn Profile](http://linkedin.com/in/donjayamahajr)                                      |

---

**Disclaimer:** The provided content is derived exclusively from an n8n automated workflow and complies with all applicable content policies. The workflow processes only legal and public data, without any illegal, offensive, or protected elements.