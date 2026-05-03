Binance SM 15min Indicators Tool

https://n8nworkflows.xyz/workflows/binance-sm-15min-indicators-tool-4743


# Binance SM 15min Indicators Tool

---

### 1. Workflow Overview

This workflow, titled **Binance SM 15min Indicators Tool**, is designed as a specialized module for short-term technical analysis on Binance Spot Market trading pairs using 15-minute candlestick data. Its main purpose is to calculate and interpret key trading indicators (RSI, MACD, Bollinger Bands, SMA/EMA, ADX) to assess intraday volatility, momentum shifts, and generate actionable trading signals formatted for Telegram or internal agents.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Triggering:** Receives requests exclusively from other workflows, ensuring controlled execution and input validation.
- **1.2 Indicator Calculation Backend Call:** Sends symbol data to an external HTTP webhook that fetches raw kline data and computes the trading indicators.
- **1.3 AI Interpretation and Analysis:** Uses a GPT-4.1-mini language model to interpret raw numeric indicator outputs into structured, human-readable trading insights.
- **1.4 Session Context Management:** Maintains session-specific memory for multi-turn interactions and symbol consistency across agents.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:**  
  This block exclusively triggers the workflow upon receiving input from authorized external workflows. It accepts a JSON payload with a trading pair symbol and a session identifier, enforcing access control and ensuring session context propagation.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Binance SM 15min Indicators Agent

- **Node Details:**  

  - **When Executed by Another Workflow**  
    - *Type & Role:* Trigger node; starts workflow execution when called by another workflow with inputs.  
    - *Configuration:* Accepts inputs named `message` (trading symbol) and `sessionId` (context ID).  
    - *Inputs & Outputs:* No incoming connections; outputs connected to the Binance SM 15min Indicators Agent.  
    - *Edge Cases:* Missing or invalid inputs, unauthorized workflow calls, or malformed JSON could cause failures.  
    - *Sticky Note:* Explains that this node only triggers from two workflows: "Binance SM Financial Analyst Tool" and "Binance Quant AI Agent."  

  - **Binance SM 15min Indicators Agent**  
    - *Type & Role:* Langchain Agent node; core processing agent interpreting input and orchestrating downstream calls.  
    - *Configuration:* Uses GPT-4.1-mini model for reasoning, expects input as `{{$json.message}}` (the symbol).  
    - *System Message:* Contains detailed instructions describing expected input format, access control, tools usage (HTTP POST to indicator backend), behavior (fetching 15m kline data, calculating indicators), and use cases.  
    - *Inputs & Outputs:* Receives input from the trigger node; outputs trigger memory, language model, and HTTP tool nodes.  
    - *Edge Cases:* Input symbol validation failures, sessionId context loss, model API errors, or misconfiguration of tools.  
    - *Sticky Note:* Describes the agent’s GPT-based reasoning to detect specific indicator patterns such as RSI overbought, MACD crossovers, Bollinger Band squeezes, EMA/SMA relationships, and ADX readings.

---

#### 2.2 Indicator Calculation Backend Call

- **Overview:**  
  This block sends the requested trading pair symbol to an external REST endpoint, which fetches the last 100 15-minute candles and calculates core technical indicators. The backend returns cleaned JSON with indicator values and signal tags.

- **Nodes Involved:**  
  - HTTP Request 15m Indicators Tool

- **Node Details:**  

  - **HTTP Request 15m Indicators Tool**  
    - *Type & Role:* HTTP Request node; performs POST request to external webhook for indicator calculation.  
    - *Configuration:*  
      - URL: `https://treasurium.app.n8n.cloud/webhook/15m-indicators`  
      - Method: POST  
      - Body: JSON containing `"symbol": "<trading_pair>"` where the symbol is dynamically injected from AI override parameter.  
    - *Inputs & Outputs:* Input from Binance SM 15min Indicators Agent; output typically carries the JSON response with indicator data.  
    - *Edge Cases:* HTTP request failures, timeout, invalid symbol causing backend errors, malformed JSON payloads or responses, authentication issues if endpoint changes.  
    - *Sticky Note:* Details the backend’s responsibilities including fetching 100 recent 15m candles and calculating RSI(14), MACD(12/26/9), Bollinger Bands(20/2), SMA/EMA(20), ADX(14).

---

#### 2.3 AI Interpretation and Analysis

- **Overview:**  
  This block uses the OpenAI GPT-4.1-mini chat model to translate raw numeric indicator results into structured, human-readable summaries optimized for Telegram. It highlights key signals such as overbought/oversold RSI, MACD crossovers, Bollinger Band status, EMA/SMA relationships, and ADX strength.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**  

  - **OpenAI Chat Model**  
    - *Type & Role:* Langchain OpenAI Chat Model node; performs language model inference.  
    - *Configuration:* Model set to `gpt-4.1-mini`. No additional options specified.  
    - *Inputs & Outputs:* Receives input from Binance SM 15min Indicators Agent; output is the interpreted textual summary.  
    - *Edge Cases:* API key invalid or rate limited, model response latency, malformed input data from previous steps causing poor interpretation.  
    - *Sticky Note:* Explains the role of the node in providing language interpretation of numeric indicators with example outputs.

---

#### 2.4 Session Context Management

- **Overview:**  
  Maintains session-specific memory buffer to store context such as sessionId, symbol, and last query, enabling persistent multi-turn interactions and symbol consistency across agents.

- **Nodes Involved:**  
  - Simple Memory

- **Node Details:**  

  - **Simple Memory**  
    - *Type & Role:* Langchain Memory Buffer Window node; stores session context.  
    - *Configuration:* Default memory buffer with no additional parameters; automatically manages session context.  
    - *Inputs & Outputs:* Connected from Binance SM 15min Indicators Agent’s ai_memory output; feeds back context for multi-turn conversation.  
    - *Edge Cases:* Memory overflow if too many turns, sessionId mismatch causing incorrect context associations.  
    - *Sticky Note:* Notes that this node stores session context including sessionId, symbol, and last query for multi-turn interactions and cross-agent symbol consistency.

---

### 3. Summary Table

| Node Name                      | Node Type                                | Functional Role                            | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                                               |
|-------------------------------|-----------------------------------------|--------------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger               | Workflow trigger from external workflows   | None                             | Binance SM 15min Indicators Agent | This workflow does not start on its own — it is always triggered by: Binance SM Financial Analyst Tool, Binance Quant AI Agent |
| Binance SM 15min Indicators Agent | Langchain Agent                        | Core processing agent with GPT instructions | When Executed by Another Workflow | Simple Memory, OpenAI Chat Model, HTTP Request 15m Indicators Tool | Uses GPT-4o-mini to interpret raw indicator output and detect patterns such as Overbought RSI, MACD crossovers, Bollinger squeezes, EMA/SMA relations, ADX readings |
| Simple Memory                  | Langchain Memory Buffer Window          | Maintains session-specific context         | Binance SM 15min Indicators Agent | Binance SM 15min Indicators Agent | Stores session context (sessionId, symbol, last query) for persistent multi-turn interactions and cross-agent consistency |
| OpenAI Chat Model             | Langchain OpenAI Chat Model              | Interprets numeric data into human language | Binance SM 15min Indicators Agent | None                             | Provides language interpretation of numeric indicators (e.g., RSI: 71 → Overbought)                                        |
| HTTP Request 15m Indicators Tool | HTTP Request                          | Calls external webhook to calculate indicators | Binance SM 15min Indicators Agent | None                             | Makes POST request to https://treasurium.app.n8n.cloud/webhook/15m-indicators; backend calculates RSI, MACD, BB, SMA/EMA, ADX |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure inputs: two JSON keys, `message` (string) and `sessionId` (string).  
   - This node has no incoming connections and serves as the workflow entry point.

2. **Add Langchain Agent Node:**  
   - Add a **Langchain Agent** node named `Binance SM 15min Indicators Agent`.  
   - Set input text parameter to `={{ $json.message }}` to capture the symbol from trigger.  
   - Insert a detailed system message specifying:  
     - Role as short-term technical analysis agent for Binance 15m intervals.  
     - Access control requiring triggering by other workflows and sessionId context.  
     - Instructions to POST symbol to `https://treasurium.app.n8n.cloud/webhook/15m-indicators` with payload format `{ "symbol": "BTCUSDT" }`.  
     - Expected calculations: RSI, MACD, Bollinger Bands, SMA/EMA, ADX.  
     - Output formatted for Telegram with trading signals summary.  
   - Connect output of `When Executed by Another Workflow` node to this agent’s input.

3. **Add Simple Memory Node:**  
   - Add a **Langchain Memory Buffer Window** node named `Simple Memory`.  
   - Connect this node’s input to the agent’s memory output port (ai_memory).  
   - No special parameters needed; this stores session context for multi-turn interactions.

4. **Add OpenAI Chat Model Node:**  
   - Add a **Langchain OpenAI Chat Model** node named `OpenAI Chat Model`.  
   - Select model `gpt-4.1-mini` (or equivalent).  
   - Connect this node’s input to the agent’s language model output (ai_languageModel).  
   - Provide credentials for OpenAI API (OAuth or API key).  
   - No additional options needed unless tuning is desired.

5. **Add HTTP Request Node:**  
   - Add an **HTTP Request** node named `HTTP Request 15m Indicators Tool`.  
   - Configure as POST request to URL: `https://treasurium.app.n8n.cloud/webhook/15m-indicators`.  
   - Set body to JSON with parameter `symbol`, dynamically assigned via expression referencing AI override or agent input (e.g., `={{ $json.message }}`).  
   - Connect this node’s input to the agent’s HTTP tool output (ai_tool).  
   - No authentication required for this endpoint as per current config.  
   - Set sensible timeout and error handling policies.

6. **Connect Nodes Appropriately:**  
   - `When Executed by Another Workflow` → `Binance SM 15min Indicators Agent`  
   - `Binance SM 15min Indicators Agent` → `Simple Memory` (ai_memory)  
   - `Binance SM 15min Indicators Agent` → `OpenAI Chat Model` (ai_languageModel)  
   - `Binance SM 15min Indicators Agent` → `HTTP Request 15m Indicators Tool` (ai_tool)

7. **Credential Setup:**  
   - Configure OpenAI API credentials for the `OpenAI Chat Model` node.  
   - No credentials needed for HTTP Request node unless endpoint changes.

8. **Testing:**  
   - Trigger the workflow via the `When Executed by Another Workflow` node manually or from a parent workflow.  
   - Provide sample input JSON: `{ "message": "BTCUSDT", "sessionId": "123456789" }`.  
   - Confirm that the HTTP Request node receives the symbol, backend returns indicator data, and the OpenAI Chat Model produces a formatted summary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow is designed as a sub-module for larger projects like the "Binance SM Financial Analyst Tool" and "Binance Quant AI Agent". It is *not* intended to be triggered manually or standalone.                             | Sticky Note on `When Executed by Another Workflow` node          |
| The backend endpoint https://treasurium.app.n8n.cloud/webhook/15m-indicators calculates all indicators and returns cleaned JSON with signal tags, centralizing heavy computation outside n8n for scalability and stability. | Sticky Note on `HTTP Request 15m Indicators Tool` node            |
| GPT-4.1-mini (gpt-4o-mini) model is chosen for balanced performance and cost-efficiency, providing structured summaries that highlight technical indicator signals in an easily readable Telegram-friendly format.           | Sticky Note on `Binance SM 15min Indicators Agent` and OpenAI node |
| Session memory management enables persistent context for multi-turn conversations and helps maintain symbol consistency when the workflow is used in multi-agent systems or chatbots.                                        | Sticky Note on `Simple Memory` node                               |
| Comprehensive documentation and licensing info available in the large sticky note node; includes deployment instructions and contact info.                                                                                   | Sticky Note near workflow bottom referencing Don Jayamaha LinkedIn |

---

**Disclaimer:** The content provided originates exclusively from an automated workflow created with n8n, adhering strictly to applicable content policies and containing no illegal or protected elements. All data handled is lawful and public.

---