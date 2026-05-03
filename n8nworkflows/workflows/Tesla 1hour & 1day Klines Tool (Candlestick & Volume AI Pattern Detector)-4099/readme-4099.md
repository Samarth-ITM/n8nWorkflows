Tesla 1hour & 1day Klines Tool (Candlestick & Volume AI Pattern Detector)

https://n8nworkflows.xyz/workflows/tesla-1hour---1day-klines-tool--candlestick---volume-ai-pattern-detector--4099


# Tesla 1hour & 1day Klines Tool (Candlestick & Volume AI Pattern Detector)

### 1. Workflow Overview

This workflow, **Tesla 1hour & 1day Klines Tool (Candlestick & Volume AI Pattern Detector)**, is designed to analyze Tesla (TSLA) stock market data by fetching real-time OHLCV (Open, High, Low, Close, Volume) candlestick data for two timeframes—1-hour and 1-day—and applying AI reasoning to detect key candlestick reversal patterns and volume divergence signals. It uses GPT-4.1 for pattern recognition and outputs a structured JSON report summarizing detected signals.

**Target Use Cases:**  
- Mid- and long-term trading decision support for Tesla stock.  
- Integration as a sub-agent within the broader Tesla Financial Market Data Analyst Tool.  
- Automated detection of technical signals like Doji, Engulfing, Hammer, Shooting Star, and volume anomalies.

---

**Logical Blocks:**

- **1.1 Input Reception & Trigger**  
  Receives execution trigger and inputs from the parent workflow.  
  Node: *When Executed by Another Workflow*

- **1.2 AI Reasoning Core**  
  Acts as the main reasoning engine, orchestrating AI logic to interpret fetched data and produce insights.  
  Node: *Tesla 1hour and 1day Klines Agent*

- **1.3 Memory Management**  
  Maintains short-term context and multi-turn session memory to enhance AI pattern detection continuity.  
  Node: *Simple Memory*

- **1.4 Data Fetching (Alpha Vantage API)**  
  Fetches Tesla OHLCV candlestick data for 1-hour and 1-day intervals using the Alpha Vantage Premium API.  
  Nodes: *Candlestick Data Hour*, *Candlestick Data Day*

- **1.5 AI Model Integration**  
  Connects to OpenAI GPT-4.1 as the core language model for candlestick and volume pattern analysis.  
  Node: *OpenAI Chat Model*

- **1.6 Documentation and Metadata**  
  Sticky notes provide contextual information, usage instructions, and architectural notes to assist users.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Trigger

- **Overview:**  
  This block enables the workflow to start execution only when triggered by another workflow, specifically the Tesla Financial Market Data Analyst Tool. It ensures required inputs (`message` and `sessionId`) are passed for AI processing and memory continuity.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry trigger for the workflow. Listens for calls from parent workflow with optional `message` and mandatory `sessionId`.  
    - *Configuration:* Accepts workflow inputs named `message` and `sessionId`.  
    - *Inputs:* External trigger from the parent workflow.  
    - *Outputs:* Passes trigger and inputs to the AI agent node.  
    - *Version Requirements:* Standard n8n version supporting Execute Workflow Trigger node.  
    - *Potential Failures:* Missing or malformed inputs; no trigger received; workflow locked or inactive.  
    - *Sticky Note:* Explains trigger purpose and input expectations.

---

#### 2.2 AI Reasoning Core

- **Overview:**  
  The central AI agent node runs prompt logic via LangChain, combining data from Alpha Vantage and memory buffer, using GPT-4.1 to analyze candlestick and volume patterns and produce structured JSON output.

- **Nodes Involved:**  
  - Tesla 1hour and 1day Klines Agent

- **Node Details:**  
  - **Tesla 1hour and 1day Klines Agent**  
    - *Type:* LangChain Agent (AI agent node)  
    - *Role:* Orchestrates AI logic, consumes input messages, accesses tools (data fetch nodes), uses memory buffer, and calls OpenAI GPT-4.1 to analyze TSLA candlestick data.  
    - *Configuration:*  
      - Text input set to incoming `message` from trigger.  
      - System message defines role as TSLA Candlestick & Volume Analyst, detailing pattern detection tasks and required output JSON format.  
      - Tools connected: 1-Hour and 1-Day Candlestick Data Tools.  
      - Memory buffer connected for session context.  
      - Language model set to OpenAI GPT-4.1.  
    - *Expressions/Variables:* Uses input JSON field `$json.message` to feed prompt text.  
    - *Inputs:* Trigger node, data fetch nodes, memory buffer, OpenAI chat model.  
    - *Outputs:* JSON structured analysis with detected patterns and volume divergences.  
    - *Potential Failures:* API rate limits (Alpha Vantage or OpenAI), malformed data responses, prompt parsing errors, memory context loss, timeouts.  
    - *Sticky Note:* Details AI agent role, pattern types detected, volume divergence logic, and output format.

---

#### 2.3 Memory Management

- **Overview:**  
  Maintains short-term memory buffer to preserve conversational context or prior analysis state across multiple executions, enabling improved multi-turn reasoning and consistency in pattern detection.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**  
  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window  
    - *Role:* Stores and retrieves short-term session data to maintain context across runs.  
    - *Configuration:* Default window-based memory buffer, no special parameters set.  
    - *Inputs:* Connected from AI agent to provide memory context.  
    - *Outputs:* Feeds memory context back to AI agent for enhanced reasoning.  
    - *Potential Failures:* Memory overflow if context grows too large; session ID mismatch causing loss of context.  
    - *Sticky Note:* Explains use of memory for multi-turn logic and tracking across timeframes.

---

#### 2.4 Data Fetching (Alpha Vantage API)

- **Overview:**  
  Two nodes independently fetch Tesla OHLCV candlestick data from Alpha Vantage API: one for 1-hour intraday candles, another for daily candles. These provide the raw data inputs for AI pattern analysis.

- **Nodes Involved:**  
  - Candlestick Data Hour  
  - Candlestick Data Day

- **Node Details:**  
  - **Candlestick Data Hour**  
    - *Type:* LangChain HTTP Request Tool  
    - *Role:* Queries Alpha Vantage API for TSLA 1-hour interval OHLCV data.  
    - *Configuration:*  
      - URL: `https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=TSLA&interval=60min&outputsize=compact`  
      - Auth: HTTP Query Auth using Alpha Vantage Premium credential.  
      - Tool description: "1-Hour Candles".  
    - *Inputs:* Called as AI tool from LangChain agent.  
    - *Outputs:* JSON response containing recent 1-hour OHLCV candlestick data.  
    - *Potential Failures:* API key limits, invalid API key, network errors, malformed JSON, Alpha Vantage rate limiting, data unavailability during off-market hours.  
    - *Sticky Note:* Describes purpose and data source for 1-hour candles.

  - **Candlestick Data Day**  
    - *Type:* LangChain HTTP Request Tool  
    - *Role:* Fetches TSLA daily OHLCV candlestick data from Alpha Vantage.  
    - *Configuration:*  
      - URL: `https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=TSLA&outputsize=compact`  
      - Auth: HTTP Query Auth with Alpha Vantage Premium credential.  
      - Tool description: "1-Day Candles".  
    - *Inputs:* Called as AI tool from LangChain agent.  
    - *Outputs:* JSON response with daily OHLCV data.  
    - *Potential Failures:* Same as above, plus possible data lag on weekends/holidays.  
    - *Sticky Note:* Explains purpose and data source for daily candles.

---

#### 2.5 AI Model Integration

- **Overview:**  
  Connects to OpenAI GPT-4.1 model to perform natural language reasoning and pattern detection on fetched candlestick and volume data.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**  
  - **OpenAI Chat Model**  
    - *Type:* LangChain Language Model Chat OpenAI Node  
    - *Role:* Provides GPT-4.1 AI model for the LangChain agent to generate reasoning outputs.  
    - *Configuration:*  
      - Model: GPT-4.1 (cached result option enabled).  
      - No additional options set (e.g., temperature defaults).  
    - *Inputs:* Receives prompt and context from AI agent node.  
    - *Outputs:* AI-generated JSON analysis of candlestick patterns and volume divergence.  
    - *Credentials:* OpenAI API key configured in n8n.  
    - *Potential Failures:* API key invalid/expired, quota exceeded, network issues, large prompt causing truncation.  
    - *Sticky Note:* Describes GPT model usage for candlestick and volume pattern interpretation.

---

#### 2.6 Documentation and Metadata (Sticky Notes)

- **Overview:**  
  Sticky notes provide detailed explanations, usage instructions, architectural context, and licensing information to assist users in understanding and maintaining the workflow.

- **Nodes Involved:**  
  - Sticky Note (Trigger explanation)  
  - Sticky Note1 (Memory explanation)  
  - Sticky Note2 (AI Agent purpose)  
  - Sticky Note3 (1-Hour Data explanation)  
  - Sticky Note4 (1-Day Data explanation)  
  - Sticky Note5 (GPT Model explanation)  
  - Sticky Note6 (Overall workflow purpose, setup, licensing, and support)

- **Node Details:**  
  - *Type:* n8n Sticky Note  
  - *Role:* Non-executable nodes purely for user guidance.  
  - *Configuration:* Text content with markdown formatting, links, and instructions.  
  - *Sticky Note6* is a large comprehensive note with detailed installation steps, architecture, and support info.  
  - *Potential Issues:* None (informational only).

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                                   | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                      |
|-------------------------------|-----------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger           | Entry trigger, receives inputs from parent      | External Trigger                 | Tesla 1hour and 1day Klines Agent | ## Trigger from Parent Workflow This node activates the workflow when called to execute Workflow action from the **Tesla Financial Market Analyst Tool**. Ensure proper inputs **message, sessionId** are passed. |
| Tesla 1hour and 1day Klines Agent | LangChain Agent                   | Main AI logic node analyzing patterns          | When Executed by Another Workflow, Simple Memory, Candlestick Data Hour, Candlestick Data Day, OpenAI Chat Model | Simple Memory                   | ## Tesla Klines Pattern & Volume AI This AI agent processes candlestick data (1h and 1d) to detect: **• Reversal Patterns (e.g. Doji, Engulfing)** **• Volume Divergence (e.g. rising price with falling volume)** It returns structured JSON for integration into the broader Tesla Quant system. |
| Simple Memory                  | LangChain Memory Buffer Window    | Maintains short-term session memory              | Tesla 1hour and 1day Klines Agent | Tesla 1hour and 1day Klines Agent | ## Short-Term Memory Module Maintains **conversation context** and **prior state across requests**. Useful for **multi-turn logic** or tracking **pattern analysis** across **timeframes**. |
| Candlestick Data Hour          | LangChain HTTP Request Tool       | Fetches 1-hour TSLA OHLCV data from Alpha Vantage | Tesla 1hour and 1day Klines Agent | Tesla 1hour and 1day Klines Agent | ## 1-Hour OHLCV Data from Alpha Vantage Fetches **TSLA hourly candles** using **Alpha Vantage API**. Output is processed by the AI agent to detect **short-term patterns** and **volume signals**. |
| Candlestick Data Day           | LangChain HTTP Request Tool       | Fetches 1-day TSLA OHLCV data from Alpha Vantage | Tesla 1hour and 1day Klines Agent | Tesla 1hour and 1day Klines Agent | ## 1-Day OHLCV Data from Alpha Vantage Pulls **TSLA daily candlestick data** **for macro-pattern recognition**. This feeds into **long-term reversal** and **divergence detection logic**. |
| OpenAI Chat Model              | LangChain Language Model Chat OpenAI | Provides GPT-4.1 model for candlestick & volume pattern analysis | Tesla 1hour and 1day Klines Agent | Tesla 1hour and 1day Klines Agent | ## GPT Model for Reasoning Uses **OpenAI GPT (e.g. GPT-4.1)** to interpret **candlestick and volume data**, generate insights, and return **JSON-formatted pattern analysis**. |
| Sticky Note                   | Sticky Note                      | Documentation: Trigger explanation                | None                            | None                            | ## Trigger from Parent Workflow This node activates the workflow when called to execute Workflow action from the **Tesla Financial Market Analyst Tool**. Ensure proper inputs **message, sessionId** are passed. |
| Sticky Note1                  | Sticky Note                      | Documentation: Short-term memory explanation      | None                            | None                            | ## Short-Term Memory Module Maintains **conversation context** and **prior state across requests**. Useful for **multi-turn logic** or tracking **pattern analysis** across **timeframes**. |
| Sticky Note2                  | Sticky Note                      | Documentation: AI agent purpose                    | None                            | None                            | ## Tesla Klines Pattern & Volume AI This AI agent processes candlestick data (1h and 1d) to detect: **• Reversal Patterns (e.g. Doji, Engulfing)** **• Volume Divergence (e.g. rising price with falling volume)** It returns structured JSON for integration into the broader Tesla Quant system. |
| Sticky Note3                  | Sticky Note                      | Documentation: 1-hour data fetching explanation   | None                            | None                            | ## 1-Hour OHLCV Data from Alpha Vantage Fetches **TSLA hourly candles** using **Alpha Vantage API**. Output is processed by the AI agent to detect **short-term patterns** and **volume signals**. |
| Sticky Note4                  | Sticky Note                      | Documentation: 1-day data fetching explanation    | None                            | None                            | ## 1-Day OHLCV Data from Alpha Vantage Pulls **TSLA daily candlestick data** **for macro-pattern recognition**. This feeds into **long-term reversal** and **divergence detection logic**. |
| Sticky Note5                  | Sticky Note                      | Documentation: GPT model explanation               | None                            | None                            | ## GPT Model for Reasoning Uses **OpenAI GPT (e.g. GPT-4.1)** to interpret **candlestick and volume data**, generate insights, and return **JSON-formatted pattern analysis**. |
| Sticky Note6                  | Sticky Note                      | Documentation: Overall workflow purpose, setup, licensing | None                            | None                            | ## 📘 Tesla 1hour and 1day Klines Tool ... (full detailed instructions, branding, licensing, and support) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create new workflow named:** `Tesla_1hour_and_1day_Klines_Tool`

2. **Add Node:** *Execute Workflow Trigger*  
   - Name: `When Executed by Another Workflow`  
   - Configure inputs: define workflow inputs `message` (optional), `sessionId` (required)  
   - Purpose: accept trigger and input from parent workflow (Tesla Financial Market Data Analyst Tool).

3. **Add Node:** *LangChain Agent*  
   - Name: `Tesla 1hour and 1day Klines Agent`  
   - Configure:  
     - Text input: expression `={{ $json.message }}`  
     - System Message: Paste detailed system prompt describing the AI role as Tesla candlestick and volume analyst with pattern detection tasks and output JSON format.  
     - Tools: leave empty for now, connect later to HTTP request tools (candlestick data)  
     - Memory: connect to a memory buffer node  
     - Language Model: connect to OpenAI GPT-4.1 node  
   - Connect "When Executed by Another Workflow" output to this node input.

4. **Add Node:** *LangChain Memory Buffer Window*  
   - Name: `Simple Memory`  
   - Default configuration (window buffer)  
   - Connect output of the `Tesla 1hour and 1day Klines Agent` to input of `Simple Memory` (ai_memory connection)  
   - Connect output of `Simple Memory` back to input of `Tesla 1hour and 1day Klines Agent` (ai_memory connection).

5. **Add Node:** *LangChain HTTP Request Tool* for 1-hour candles  
   - Name: `Candlestick Data Hour`  
   - URL: `https://www.alphavantage.co/query?function=TIME_SERIES_INTRADAY&symbol=TSLA&interval=60min&outputsize=compact`  
   - Authentication: Select or create HTTP Query Auth credential named `Alpha Vantage Premium` with your Alpha Vantage Premium API key  
   - Tool Description: "1-Hour Candles"  
   - Connect output as AI tool input to `Tesla 1hour and 1day Klines Agent`.

6. **Add Node:** *LangChain HTTP Request Tool* for 1-day candles  
   - Name: `Candlestick Data Day`  
   - URL: `https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=TSLA&outputsize=compact`  
   - Authentication: same `Alpha Vantage Premium` credential  
   - Tool Description: "1-Day Candles"  
   - Connect output as AI tool input to `Tesla 1hour and 1day Klines Agent`.

7. **Add Node:** *LangChain Language Model Chat OpenAI*  
   - Name: `OpenAI Chat Model`  
   - Model: Select `gpt-4.1` or equivalent GPT-4 variant  
   - Credentials: configure OpenAI API key in n8n credentials (`OpenAi account`)  
   - Connect this node as AI language model input for `Tesla 1hour and 1day Klines Agent`.

8. **Connect all nodes as follows:**  
   - `When Executed by Another Workflow` → `Tesla 1hour and 1day Klines Agent` (main input)  
   - `Tesla 1hour and 1day Klines Agent` ↔ `Simple Memory` (ai_memory bidirectional)  
   - `Candlestick Data Hour` → `Tesla 1hour and 1day Klines Agent` (ai_tool)  
   - `Candlestick Data Day` → `Tesla 1hour and 1day Klines Agent` (ai_tool)  
   - `OpenAI Chat Model` → `Tesla 1hour and 1day Klines Agent` (ai_languageModel)

9. **Add Sticky Notes** as per documentation needs (optional but recommended). Include key instructions on:  
   - Trigger usage  
   - Memory function  
   - Data fetch nodes  
   - AI model roles  
   - Overall workflow architecture and licensing

10. **Save and activate workflow.**

11. **Test Execution:**  
    - Trigger this workflow by invoking it from the Tesla Financial Market Data Analyst Tool, passing `message` (optional) and `sessionId` parameters.  
    - Expect structured JSON output summarizing detected candlestick patterns and volume divergence for both 1-hour and 1-day TSLA candles.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow triggered only as a sub-agent by Tesla Financial Market Data Analyst Tool.                                                                     | [Tesla Financial Market Data Analyst Tool](https://n8n.io/workflows/4094-tesla-financial-market-data-analyst-tool-multi-timeframe-technical-ai-agent/) |
| Requires Alpha Vantage Premium API Key for reliable, real-time TSLA OHLCV data retrieval.                                                               | Alpha Vantage API Documentation: https://www.alphavantage.co/documentation/                                  |
| Uses OpenAI GPT-4.1 model for advanced reasoning on candlestick and volume data.                                                                        | OpenAI API info: https://platform.openai.com/docs/models/gpt-4                                                 |
| Detects key reversal candlestick patterns: Doji, Engulfing (Bullish/Bearish), Hammer/Inverted Hammer, Shooting Star.                                   | Standard candlestick pattern references: https://www.investopedia.com/terms/c/candlestick.asp                |
| Volume divergence defined as Bullish (price down, volume stable/rising) or Bearish (price up, volume fading).                                           | Technical analysis concepts: https://www.investopedia.com/terms/v/volumedivergence.asp                        |
| Short-term memory buffer enables multi-turn reasoning and context retention within session identified by `sessionId`.                                  | LangChain Memory concept documentation: https://python.langchain.com/en/latest/modules/memory.html           |
| Proprietary IP and licensing by Treasurium Capital Limited Company. Unauthorized reproduction prohibited.                                                | © 2025 Treasurium Capital Limited Company                                                                    |
| Workflow creator & support contact: Don Jayamaha (LinkedIn profile and n8n creator profile provided).                                                   | https://linkedin.com/in/donjayamahajr and https://n8n.io/creators/don-the-gem-dealer/                         |
| Important: This tool is not standalone and must be integrated with the parent Tesla Financial Market Data Analyst Tool for triggering and context.      | See Setup Instructions section for integration details                                                       |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated n8n workflow. All data handled is legal and public. This workflow respects current content policies and contains no illegal or offensive elements.