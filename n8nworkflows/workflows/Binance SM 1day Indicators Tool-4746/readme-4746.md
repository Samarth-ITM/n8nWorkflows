Binance SM 1day Indicators Tool

https://n8nworkflows.xyz/workflows/binance-sm-1day-indicators-tool-4746


# Binance SM 1day Indicators Tool

---

### 1. Workflow Overview

The **Binance SM 1day Indicators Tool** workflow is designed to analyze long-term technical indicators on Binance Spot Market trading pairs using daily (1-day interval) candle data. It targets quantitative analysts, swing traders, and AI agents who require macro-level market sentiment, trend shifts, and support/resistance identification for strategic decision-making.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception & Trigger**: Accepts input requests from external workflows, carrying the trading symbol and session context.
- **1.2 Indicator Calculation Agent**: Orchestrates technical indicator calculations by calling an external webhook that processes 1-day candle data and returns multiple indicators.
- **1.3 HTTP Request Tool**: Sends the trading symbol to the external indicator service and retrieves raw indicator data.
- **1.4 AI Interpretation**: Uses an OpenAI GPT model to interpret raw numerical outputs into human-readable insights and trading signal summaries.
- **1.5 Session Memory Management**: Maintains session context and symbol tracking for multi-turn or multi-timeframe interactions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Trigger

- **Overview:**  
This block triggers the workflow execution upon receiving an external call from parent workflows, forwarding the input data (symbol and session ID) downstream.

- **Nodes Involved:**  
  - `When Executed by Another Workflow`

- **Node Details:**  
  - **Type and Role:** `Execute Workflow Trigger` node; listens for invocation from other workflows and accepts input parameters.  
  - **Configuration:** Accepts two inputs: `message` (the Binance trading symbol) and `sessionId` (for session continuity).  
  - **Expressions/Variables:** None directly; inputs come from external workflow triggers.  
  - **Connections:** Output connects to `Binance SM 1day Indicators Agent`.  
  - **Version:** 1.1  
  - **Potential Failures:** Missing or malformed input parameters; trigger not activated if called improperly.  
  - **Sub-Workflow Reference:** Triggered exclusively by parent workflows such as the Binance SM Financial Analyst Tool and Binance Quant AI Agent.

---

#### 2.2 Indicator Calculation Agent

- **Overview:**  
Serves as the core reasoning agent that receives the input symbol, validates it, and coordinates the retrieval and calculation of long-term daily technical indicators by calling an external service.

- **Nodes Involved:**  
  - `Binance SM 1day Indicators Agent`

- **Node Details:**  
  - **Type and Role:** LangChain Agent node; processes input text and manages calls to external tools.  
  - **Configuration:**  
    - Input text set dynamically from incoming JSON `message` field.  
    - System message defines the agent’s role with detailed instructions: calculate 1-day interval indicators such as RSI, MACD, Bollinger Bands, SMA, EMA, ADX using 40 daily candles.  
    - The node calls a webhook at `https://treasurium.app.n8n.cloud/webhook/1d-indicators` to fetch indicator data.  
  - **Expressions/Variables:** Uses `={{ $json.message }}` to extract symbol from input.  
  - **Connections:** Input from `When Executed by Another Workflow`; outputs to `HTTP Request 1d Indicators Tool` via agent tool interface.  
  - **Version:** 1.9  
  - **Potential Failures:**  
    - Incorrect or malformed symbol input (e.g., lowercase, typos).  
    - Failure to get sufficient Binance 1-day candles (less than 40).  
    - Webhook service unavailability or incorrect response format.  
  - **Sub-Workflow Reference:** Integrates with external webhook for indicator retrieval.

---

#### 2.3 HTTP Request Tool

- **Overview:**  
Executes the external HTTP POST request to the webhook endpoint that calculates and returns the daily technical indicators for the specified Binance symbol.

- **Nodes Involved:**  
  - `HTTP Request 1d Indicators Tool`

- **Node Details:**  
  - **Type and Role:** HTTP Request Tool node; sends POST requests with JSON body to an external API.  
  - **Configuration:**  
    - URL: `https://treasurium.app.n8n.cloud/webhook/1d-indicators`  
    - Method: POST  
    - Body Parameters: JSON object containing the symbol dynamically set via AI override expression to match the agent’s symbol parameter.  
  - **Expressions/Variables:** Uses AI-generated override expression:  
    `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('parameters0_Value', ``, 'string') }}` for the `symbol` field.  
  - **Connections:** Input from `Binance SM 1day Indicators Agent` (as ai_tool); outputs raw indicator data back to the agent for further processing.  
  - **Version:** 4.2  
  - **Potential Failures:**  
    - HTTP errors (timeouts, 4xx/5xx responses).  
    - Invalid or missing symbol parameter leading to API errors.  
    - Malformed response from webhook breaking downstream processing.

---

#### 2.4 AI Interpretation

- **Overview:**  
Interprets the raw numerical indicator data into meaningful trading insights such as trend phases, momentum signals, and generates Telegram-ready summaries.

- **Nodes Involved:**  
  - `OpenAI Chat Model`  
  - `Simple Memory`

- **Node Details:**  

  - **OpenAI Chat Model:**  
    - **Type and Role:** LangChain OpenAI Chat node; uses GPT-4.1-mini model to generate natural language interpretations.  
    - **Configuration:** Uses GPT-4.1-mini with no additional options set.  
    - **Expressions/Variables:** Receives processed numerical data from the agent’s memory buffer.  
    - **Connections:** Input from `Binance SM 1day Indicators Agent` (ai_languageModel), output likely to calling workflow (not shown).  
    - **Version:** 1.2  
    - **Potential Failures:**  
      - OpenAI API rate limits or authorization errors.  
      - Unexpected input format causing response errors.  

  - **Simple Memory:**  
    - **Type and Role:** LangChain Memory Buffer Window node; stores recent session state such as sessionId, symbol, and last-used indicator data.  
    - **Configuration:** No parameters; default session memory buffer window.  
    - **Connections:** Input from `Binance SM 1day Indicators Agent` (ai_memory), outputs back to the agent to maintain context.  
    - **Version:** 1.3  
    - **Potential Failures:**  
      - Memory overflow or session mismatch if sessionId not properly maintained.

---

### 3. Summary Table

| Node Name                      | Node Type                                   | Functional Role                                    | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                                         |
|-------------------------------|---------------------------------------------|---------------------------------------------------|----------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger                     | Workflow trigger receiving external invocation    | -                                | Binance SM 1day Indicators Agent      | This workflow does not start on its own — it is always triggered by: Binance SM Financial Analyst Tool, Binance Quant AI Agent |
| Binance SM 1day Indicators Agent | LangChain Agent                             | Main logic agent orchestrating indicator retrieval | When Executed by Another Workflow | HTTP Request 1d Indicators Tool       | Calculates daily RSI, MACD, Bollinger Bands, SMA, EMA, ADX for any Binance symbol to detect macro trends            |
| HTTP Request 1d Indicators Tool | HTTP Request Tool                           | Calls external webhook to calculate indicators    | Binance SM 1day Indicators Agent  | Binance SM 1day Indicators Agent      | Sends POST request with symbol to webhook; retrieves 40 daily candles indicators                                    |
| OpenAI Chat Model              | LangChain OpenAI Chat Model                 | Interprets raw indicator data into trading signals | Binance SM 1day Indicators Agent  | -                                    | Uses GPT-4.1-mini to generate summaries and trend interpretations                                                   |
| Simple Memory                 | LangChain Memory Buffer Window               | Maintains session and symbol context               | Binance SM 1day Indicators Agent  | Binance SM 1day Indicators Agent      | Stores sessionId, symbol, last-used indicators for multi-turn conversations                                         |
| Sticky Note                   | Sticky Note                                 | Comment and documentation                          | -                                | -                                    | Workflow Trigger: This workflow triggered by parent workflows                                                       |
| Sticky Note8                  | Sticky Note                                 | Comment and documentation                          | -                                | -                                    | Main Reasoning Agent: Calculates key daily indicators to detect macro sentiment and trend setups                    |
| Sticky Note2                  | Sticky Note                                 | Comment and documentation                          | -                                | -                                    | OpenAI Chat Model: GPT-4.1-mini interprets trend status and outputs Telegram-ready summaries                         |
| Sticky Note1                  | Sticky Note                                 | Comment and documentation                          | -                                | -                                    | Simple Memory: Stores session data for multi-timeframe reports                                                      |
| Sticky Note5                  | Sticky Note                                 | Comment and documentation                          | -                                | -                                    | HTTP Request 1d Indicators Tool: Calls webhook, calculates RSI, MACD, Bollinger Bands, SMA, EMA, ADX using 40 candles |
| Sticky Note3                  | Sticky Note                                 | Comment and documentation                          | -                                | -                                    | Complete documentation describing workflow purpose, inputs, outputs, and licensing                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure it to accept two inputs: `message` (string) and `sessionId` (string).  
   - This node will not start on its own and must be invoked externally.

2. **Create Indicator Agent Node:**  
   - Add a **LangChain Agent** node named `Binance SM 1day Indicators Agent`.  
   - Set input text as `={{ $json.message }}` to extract symbol.  
   - Define the System Message with detailed instructions:  
     - Responsible for calculating 1-day technical indicators (RSI, MACD, Bollinger Bands, SMA, EMA, ADX).  
     - Input format and expected behavior as per the documentation.  
   - Configure the agent to call an external HTTP tool (next step).  
   - Connect output from `When Executed by Another Workflow` to this node.

3. **Create HTTP Request Tool Node:**  
   - Add **HTTP Request Tool** node named `HTTP Request 1d Indicators Tool`.  
   - Set Method to POST and URL to `https://treasurium.app.n8n.cloud/webhook/1d-indicators`.  
   - Configure body parameters: JSON with key `symbol` set dynamically via AI override expression, e.g.:  
     `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('parameters0_Value', ``, 'string') }}`  
   - Connect this node as the `ai_tool` output of the agent node.

4. **Create OpenAI Chat Model Node:**  
   - Add **LangChain OpenAI Chat** node named `OpenAI Chat Model`.  
   - Select model `gpt-4.1-mini` and link to valid OpenAI API credentials.  
   - Connect this node as the `ai_languageModel` output of the agent node.

5. **Create Simple Memory Node:**  
   - Add **LangChain Memory Buffer Window** node named `Simple Memory`.  
   - Leave default configuration for session memory.  
   - Connect this node as `ai_memory` input and output of the agent node to maintain session context.

6. **Connect Nodes:**  
   - `When Executed by Another Workflow` (trigger) → `Binance SM 1day Indicators Agent` (main agent)  
   - `Binance SM 1day Indicators Agent` → `HTTP Request 1d Indicators Tool` (ai_tool)  
   - `Binance SM 1day Indicators Agent` → `OpenAI Chat Model` (ai_languageModel)  
   - `Binance SM 1day Indicators Agent` ↔ `Simple Memory` (ai_memory, bidirectional)

7. **Credential Setup:**  
   - For `OpenAI Chat Model`, configure OpenAI API credentials with a valid API key.  
   - For HTTP Request Tool, no credentials needed as it calls a public webhook.

8. **Parent Workflow Integration:**  
   - Ensure this workflow is only triggered by other workflows (e.g., Binance SM Financial Analyst Tool, Binance Quant AI Agent).  
   - Provide correct JSON input format:  
     ```json
     {
       "message": "MATICUSDT",
       "sessionId": "telegram_chat_id"
     }
     ```

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                  |
|--------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow is triggered exclusively by Binance SM Financial Analyst Tool and Binance Quant AI Agent.      | Sticky Note on Workflow Trigger node                            |
| The workflow calculates RSI, MACD, Bollinger Bands, SMA, EMA, and ADX indicators on 1-day Binance candles.  | Sticky Note on Main Reasoning Agent node                        |
| OpenAI GPT-4.1-mini is used to interpret raw indicator data into trading signals and Telegram-ready summaries.| Sticky Note on OpenAI Chat Model                                |
| Session management is performed to track multi-turn conversations with symbols and indicators.               | Sticky Note on Simple Memory node                               |
| The external webhook endpoint: POST https://treasurium.app.n8n.cloud/webhook/1d-indicators                    | Sticky Note on HTTP Request 1d Indicators Tool                  |
| Complete documentation and licensing info available within Sticky Note node (Sticky Note3).                   | Sticky Note with detailed workflow documentation and license   |
| Licensing: © 2025 Treasurium Capital Limited Company. Internal use only; IP protected under U.S. law.        | Licensing section in Sticky Note3                               |
| Author: Don Jayamaha – LinkedIn profile: http://linkedin.com/in/donjayamahajr                               | Licensing section in Sticky Note3                               |

---

**Disclaimer:**  
The provided description is generated solely from the automated n8n workflow JSON export. It respects all content policies and contains no illegal, offensive, or protected elements. All data processed is lawful and public.

---