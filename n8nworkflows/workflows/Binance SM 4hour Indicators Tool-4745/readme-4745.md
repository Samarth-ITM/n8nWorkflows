Binance SM 4hour Indicators Tool

https://n8nworkflows.xyz/workflows/binance-sm-4hour-indicators-tool-4745


# Binance SM 4hour Indicators Tool

---

### 1. Workflow Overview

The **Binance SM 4hour Indicators Tool** is a medium-term technical analysis workflow designed to generate 4-hour interval trading signals for Binance Spot Market symbols. It targets use cases such as swing trend evaluation, volatility compression detection, and reversal pressure analysis within a 1-2 day horizon. The workflow is primarily intended to serve as a backend tool for higher-level agents like the Binance SM Financial Analyst Tool and the Binance Quant AI Agent.

The workflow’s logic is divided into the following functional blocks:

- **1.1 Trigger Input Reception:** Receives input messages and session identifiers from parent workflows.
- **1.2 Indicator Calculation Agent:** Processes the input symbol, calls an external webhook to fetch and calculate multiple technical indicators, and interprets the results.
- **1.3 External Indicator Data Fetch:** Issues an HTTP POST request to a dedicated webhook that pulls Binance 4-hour candle data and computes several indicators.
- **1.4 AI Interpretation & Memory:** Uses OpenAI GPT (gpt-4.1-mini) to convert raw indicator outputs into human-readable, Telegram-friendly summaries; maintains session memory for multi-turn context.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Input Reception

- **Overview:**  
  This block waits for execution triggers from other workflows, receiving the trading symbol and session ID as inputs. It acts as the workflow’s entry point.

- **Nodes Involved:**  
  - `When Executed by Another Workflow`

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - *Type & Role:* Execute Workflow Trigger node; initiates this workflow when called by another workflow.  
    - *Configuration:* Accepts workflow inputs named `message` (the trading symbol) and `sessionId` (context identifier).  
    - *Expressions/Variables:* Passes these inputs downstream as JSON fields.  
    - *Connections:* Outputs trigger to `Binance SM 4hour Indicators Tool Agent`.  
    - *Failure Modes:* Missing or invalid inputs could cause downstream errors; no internal validation.  
    - *Sub-workflow:* Triggered externally by Binance SM Financial Analyst Tool and Binance Quant AI Agent.

#### 2.2 Indicator Calculation Agent

- **Overview:**  
  This is the core reasoning node that orchestrates the workflow logic. It interprets incoming messages, calls the external webhook to fetch indicator data, and prepares the context for AI analysis.

- **Nodes Involved:**  
  - `Binance SM 4hour Indicators Tool Agent`

- **Node Details:**  
  - *Type & Role:* Langchain Agent node specialized for text-based AI workflow orchestration.  
  - *Configuration:*  
    - Input text is set dynamically from the received `message` field.  
    - Uses a detailed system prompt defining its role as a swing trend analyzer for Binance 4-hour interval data, specifying expected inputs, output behavior, and connected tools.  
    - The prompt outlines the indicators used (RSI, MACD, Bollinger Bands, SMA, EMA, ADX) and specifies the external webhook endpoint (`https://treasurium.app.n8n.cloud/webhook/4h-indicators`).  
  - *Expressions:* `={{ $json.message }}` to extract input symbol.  
  - *Connections:*  
    - Receives input from the trigger node.  
    - Outputs AI memory data to `Simple Memory`.  
    - Sends AI tool commands to `HTTP Request 4h Indicators Tool`.  
    - Sends AI language model commands to `OpenAI Chat Model`.  
  - *Failure Modes:*  
    - Invalid symbols, API throttling, or insufficient candle data could cause failures.  
    - Expression errors if input is missing or malformed.  
  - *Sub-workflow:* Acts as the central agent coordinating subprocesses.

#### 2.3 External Indicator Data Fetch

- **Overview:**  
  Performs the actual data retrieval and indicator computation by POSTing the symbol to an external webhook service.

- **Nodes Involved:**  
  - `HTTP Request 4h Indicators Tool`

- **Node Details:**  
  - *Type & Role:* HTTP Request node specialized for tool usage within n8n.  
  - *Configuration:*  
    - Sends POST request to `https://treasurium.app.n8n.cloud/webhook/4h-indicators`.  
    - Payload body contains JSON with the symbol field dynamically set from AI tool's instructions.  
  - *Expressions:* Uses AI-generated override expression to dynamically set the symbol parameter.  
  - *Connections:*  
    - Input from `Binance SM 4hour Indicators Tool Agent` as AI tool.  
  - *Failure Modes:*  
    - HTTP errors (timeouts, 4xx or 5xx responses).  
    - Throttling or invalid response formats from the webhook.  
    - Symbol validation failures at the external API.  
  - *Sub-workflow:* Delegates indicator calculation to an external webhook microservice.

#### 2.4 AI Interpretation & Memory

- **Overview:**  
  Interprets the raw indicator data and formats it as a Telegram-friendly message; stores session data for contextual continuity.

- **Nodes Involved:**  
  - `OpenAI Chat Model`  
  - `Simple Memory`

- **Node Details:**  
  - **OpenAI Chat Model**  
    - *Type & Role:* Langchain OpenAI Chat Model node.  
    - *Configuration:*  
      - Uses model `gpt-4.1-mini`.  
      - No additional prompt options specified; input is passed from the agent node.  
      - Connected to OpenAI credentials for API access.  
    - *Connections:* Input from `Binance SM 4hour Indicators Tool Agent` AI language model output.  
    - *Failure Modes:*  
      - API quota exceeded, network issues, or malformed prompts.  
      - Model version compatibility (requires n8n version supporting Langchain 1.2+).  
  - **Simple Memory**  
    - *Type & Role:* Langchain Memory Buffer Window node.  
    - *Purpose:* Stores session variables like the symbol and prior interaction state to maintain conversation context across executions.  
    - *Connections:* Receives AI memory input from `Binance SM 4hour Indicators Tool Agent`.  
    - *Failure Modes:* Memory corruption or missed session IDs may cause inconsistent context.

---

### 3. Summary Table

| Node Name                         | Node Type                                  | Functional Role                           | Input Node(s)                             | Output Node(s)                           | Sticky Note                                                                                             |
|----------------------------------|--------------------------------------------|-----------------------------------------|------------------------------------------|----------------------------------------|-------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger                    | Entry trigger receiving symbol and session ID | —                                        | Binance SM 4hour Indicators Tool Agent | ## Workflow Trigger Node: When Executed by Another Workflow This workflow does not start on its own — it is always triggered by: Binance SM Financial Analyst Tool, Binance Quant AI Agent |
| Binance SM 4hour Indicators Tool Agent | Langchain Agent                            | Core swing trend analysis agent, orchestrates calls and prompts | When Executed by Another Workflow         | Simple Memory, HTTP Request 4h Indicators Tool, OpenAI Chat Model | ## System Reasoning Agent Node: Binance SM 4hour Indicators Tool Agent System prompt defines this as a swing trend analyzer. Interprets indicator outputs and transforms them into labeled sentiment for the 4h window. |
| HTTP Request 4h Indicators Tool   | HTTP Request Tool                          | Fetches Binance 4h candle data and calculates indicators | Binance SM 4hour Indicators Tool Agent (ai_tool) | —                                      | ## Webhook Request (Core Tool) Node: HTTP Request 4h Indicators Tool POSTs to https://treasurium.app.n8n.cloud/webhook/4h-indicators Payload: { "symbol": "BNBUSDT" } Pulls 40 candles of 4h OHLCV from Binance Calculates RSI, MACD, Bollinger Bands, SMA, EMA, ADX Returns clean JSON with all indicators and signal summaries. |
| OpenAI Chat Model                 | Langchain OpenAI Chat Model                | Converts raw indicator data into readable Telegram summaries | Binance SM 4hour Indicators Tool Agent (ai_languageModel) | —                                      | ## OpenAI Chat Model Node: OpenAI Chat Model (gpt-4.1-mini) Interprets indicator values and returns Telegram-friendly summaries like RSI, MACD, BB, EMA/SMA trends, and ADX strength. |
| Simple Memory                    | Langchain Memory Buffer Window             | Stores session context for multi-turn interactions and symbol recall | Binance SM 4hour Indicators Tool Agent (ai_memory) | —                                      | ## Context Memory Node: Simple Memory Stores session variables like symbol and prior state. Useful for inter-agent coordination and multi-turn state tracking. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: When Executed by Another Workflow**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: Add `message` (string) and `sessionId` (string) as expected parameters.  
   - Position as entry point.

2. **Add Langchain Agent Node: Binance SM 4hour Indicators Tool Agent**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Parameters:  
     - Text input: `={{ $json.message }}`  
     - System Message: Paste the detailed system prompt defining the agent’s role, expected input format, behavior, connected tools, and operational notes.  
   - Connect output from trigger node to this agent node input.

3. **Configure AI Memory Node: Simple Memory**  
   - Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - No special parameters needed; defaults suffice.  
   - Connect `ai_memory` output of the agent node here to track session state.

4. **Configure HTTP Request Node: HTTP Request 4h Indicators Tool**  
   - Type: `n8n-nodes-base.httpRequestTool`  
   - Set method to `POST`.  
   - Set URL to `https://treasurium.app.n8n.cloud/webhook/4h-indicators`.  
   - Configure body parameters with JSON: `{ "symbol": "={{ $fromAI('parameters0_Value', '', 'string') }}" }` (to allow AI to specify symbol dynamically).  
   - Connect `ai_tool` output from agent node here.

5. **Configure OpenAI Chat Model Node: OpenAI Chat Model**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Select model: `gpt-4.1-mini`.  
   - Add OpenAI credentials (API key, OAuth, etc).  
   - Connect `ai_languageModel` output from agent node here.

6. **Link Connections Appropriately**  
   - Connect `main` output of trigger node to agent node input.  
   - Connect `ai_memory` output of agent node to Simple Memory.  
   - Connect `ai_tool` output of agent node to HTTP Request node.  
   - Connect `ai_languageModel` output of agent node to OpenAI Chat Model node.

7. **Credential Setup**  
   - Add OpenAI credentials with an active API key for GPT-4.1-mini.  
   - No special credentials required for HTTP Request to the webhook since it is public.

8. **Deploy and Test**  
   - Ensure webhook `https://treasurium.app.n8n.cloud/webhook/4h-indicators` is active externally and supports POST with `symbol`.  
   - Trigger the workflow externally with JSON input:  
     ```json
     {
       "message": "BNBUSDT",
       "sessionId": "telegram_chat_id"
     }
     ```  
   - Confirm that the workflow retrieves indicator data and outputs a Telegram-formatted summary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| This workflow is triggered exclusively by the Binance SM Financial Analyst Tool and Binance Quant AI Agent workflows.                     | See Sticky Note on "When Executed by Another Workflow" node.  |
| The system prompt for the Langchain agent defines the workflow as a swing trend analyzer calculating RSI, MACD, Bollinger Bands, SMA, EMA, and ADX. | See Sticky Note on "Binance SM 4hour Indicators Tool Agent".  |
| The external webhook `https://treasurium.app.n8n.cloud/webhook/4h-indicators` pulls 40 4-hour Binance candles and calculates all indicators. | See Sticky Note on "HTTP Request 4h Indicators Tool".         |
| OpenAI GPT model `gpt-4.1-mini` is used for converting raw data into Telegram-friendly messages.                                          | See Sticky Note on "OpenAI Chat Model".                        |
| Session memory is maintained to track symbol and interaction state for multi-turn or multi-agent coordination.                            | See Sticky Note on "Simple Memory".                            |
| Official LinkedIn credit: Don Jayamaha - http://linkedin.com/in/donjayamahajr                                                             | Licensing & Support section in documentation Sticky Note.      |
| The workflow is proprietary and protected under U.S. copyright and international IP law; redistribution or resale without license is prohibited. | Licensing & Support section.                                    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, respecting all current content policies and containing no illegal or offensive material. All processed data is legal and public.