Technical stock analysis with Telegram, Airtable and a GPT-powered AI Agent

https://n8nworkflows.xyz/workflows/technical-stock-analysis-with-telegram--airtable-and-a-gpt-powered-ai-agent-3053


# Technical stock analysis with Telegram, Airtable and a GPT-powered AI Agent

### 1. Workflow Overview

This workflow automates technical stock analysis by integrating Telegram messaging, Airtable data storage, and AI-powered analysis using OpenAI models within n8n. It targets traders, financial analysts, and developers who want to interact with an AI trading agent via Telegram for real-time stock chart generation and technical evaluation without coding.

The workflow is logically divided into two main scenarios:

- **Scenario 1: AI Agent Interaction via Telegram**  
  Handles incoming Telegram messages (text or voice), processes user queries, transcribes voice messages, invokes an AI agent to analyze stocks using a chart generation tool, and sends back analysis results and charts.

- **Scenario 2: Scheduled Batch Analyses**  
  Periodically retrieves stored tickers from Airtable, loops through them to request stock analysis from the AI agent, and sends results accordingly.

Logical blocks:

- **1.1 Input Reception and Preprocessing**  
  Telegram Trigger, Switch node to differentiate voice vs text, Download File, Transcribe, and Set Text nodes.

- **1.2 AI Agent Processing**  
  AI Agent node configured with system instructions and integrated with the Get Chart tool and memory buffer.

- **1.3 Chart Generation and Technical Analysis**  
  Get Chart tool (sub-workflow), HTTP requests to chart API, Download Chart, and Technical Analysis node using OpenAI image analysis.

- **1.4 Output Delivery**  
  Send Analysis and Send Chart nodes to deliver results back to Telegram.

- **1.5 Scheduled Batch Processing**  
  Schedule Trigger, Airtable nodes to fetch tickers, Loop Over Items, and Run Agent HTTP request to invoke the AI agent workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

**Overview:**  
This block receives messages from Telegram, distinguishes between voice and text inputs, downloads voice files if needed, transcribes audio to text, and prepares the text for AI processing.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Download File  
- Transcribe  
- Set Text

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages (text or voice).  
  - Config: Listens to "message" updates; uses Telegram API credentials.  
  - Inputs: Telegram messages  
  - Outputs: Raw Telegram message JSON  
  - Edge cases: Telegram API downtime, invalid message formats, missing voice file IDs.

- **Switch**  
  - Type: Switch  
  - Role: Routes messages based on content type (voice or text).  
  - Config: Checks existence of `message.voice.file_id` for voice, or `message.text` for text.  
  - Inputs: Telegram Trigger output  
  - Outputs: Two branches: "Voice" and "Text"  
  - Edge cases: Messages without voice or text fields, unexpected message types.

- **Download File**  
  - Type: Telegram node (file download)  
  - Role: Downloads voice message file from Telegram servers.  
  - Config: Uses `message.voice.file_id` to fetch file.  
  - Inputs: Voice branch from Switch  
  - Outputs: Binary audio file  
  - Edge cases: File not found, Telegram API errors, network issues.

- **Transcribe**  
  - Type: OpenAI Audio Transcription node  
  - Role: Transcribes downloaded audio to text using OpenAI API.  
  - Config: Uses OpenAI credentials, operation set to "transcribe".  
  - Inputs: Audio file from Download File  
  - Outputs: Transcribed text  
  - Edge cases: Audio format unsupported, transcription errors, API rate limits.

- **Set Text**  
  - Type: Set node  
  - Role: Forwards text messages as-is by extracting `message.text`.  
  - Config: Assigns `text` field from incoming JSON.  
  - Inputs: Text branch from Switch  
  - Outputs: JSON with `text` property  
  - Edge cases: Empty or malformed text messages.

---

#### 2.2 AI Agent Processing

**Overview:**  
Processes the prepared text input through an AI agent configured with domain-specific instructions for stock analysis. The agent can call the Get Chart tool and uses a memory buffer for session context.

**Nodes Involved:**  
- AI Agent  
- Window Buffer Memory  
- Get Chart (tool workflow)

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent node  
  - Role: Processes user queries, performs stock analysis, calls Get Chart tool as needed.  
  - Config:  
    - Input text from previous block.  
    - System message defines agent role, instructions, and SOP for stock analysis.  
    - Uses GPT-4o model.  
    - Integrates with Get Chart tool and Window Buffer Memory.  
  - Inputs: Text from Set Text or Transcribe nodes  
  - Outputs: Analysis text to be sent back to Telegram  
  - Edge cases: Model API errors, tool invocation failures, session memory issues.

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context for the AI agent using a fixed session key.  
  - Config: Uses a custom session key `"335458847"` to track conversation state.  
  - Inputs: AI Agent  
  - Outputs: AI Agent (memory context)  
  - Edge cases: Memory overflow, session key conflicts.

- **Get Chart**  
  - Type: LangChain Tool Workflow (sub-workflow)  
  - Role: Generates stock charts based on ticker and chart style parameters.  
  - Config:  
    - Workflow ID references a sub-workflow named "Trading Agent".  
    - Inputs: `ticker` and optional `chart_style` (default "candle").  
    - Outputs: URL of generated chart in markdown format.  
  - Inputs: Called by AI Agent as a tool  
  - Outputs: Chart URL for analysis  
  - Edge cases: Invalid ticker symbols, unsupported chart styles, API failures.

---

#### 2.3 Chart Generation and Technical Analysis

**Overview:**  
Generates chart images from the chart URL, downloads them, and performs detailed technical analysis using OpenAI's image analysis capabilities.

**Nodes Involved:**  
- Set Values  
- Get Chart URL (HTTP Request)  
- Download Chart (HTTP Request)  
- Technical Analysis (OpenAI Image Analysis)

**Node Details:**

- **Set Values**  
  - Type: Set node  
  - Role: Prepares parameters `ticker` and `chart_style` for chart generation.  
  - Config: Assigns values from workflow inputs or AI outputs.  
  - Inputs: Workflow Input Trigger or AI Agent outputs  
  - Outputs: JSON with `ticker` and `chart_style`  
  - Edge cases: Missing or invalid parameters.

- **Get Chart URL**  
  - Type: HTTP Request  
  - Role: Calls external chart image API to generate chart URL.  
  - Config:  
    - POST request to `https://api.chart-img.com/v2/tradingview/advanced-chart/storage`.  
    - JSON body includes style, theme, interval, symbol (prefixed with NASDAQ), and studies (Volume, RSI, Stochastic RSI).  
    - Uses HTTP header authentication with API key.  
  - Inputs: Parameters from Set Values  
  - Outputs: JSON containing chart URL  
  - Edge cases: API key invalid, network errors, malformed request.

- **Download Chart**  
  - Type: HTTP Request  
  - Role: Downloads the chart image from the URL obtained.  
  - Config: GET request to the chart URL.  
  - Inputs: Chart URL from Get Chart URL  
  - Outputs: Binary image data (base64)  
  - Edge cases: URL expired, download failures.

- **Technical Analysis**  
  - Type: OpenAI Image Analysis node  
  - Role: Performs detailed technical analysis on the chart image.  
  - Config:  
    - Uses GPT-4o model.  
    - Input type: base64 image.  
    - Prompt instructs the model to analyze candlestick patterns, RSI, Stochastic RSI, MACD, volume, and market sentiment with detailed explanations.  
  - Inputs: Chart image from Download Chart  
  - Outputs: Textual analysis of the chart  
  - Edge cases: Image decoding errors, model API failures.

---

#### 2.4 Output Delivery

**Overview:**  
Sends the AI-generated analysis text and chart images back to the user via Telegram.

**Nodes Involved:**  
- Send Chart  
- Send Analysis

**Node Details:**

- **Send Chart**  
  - Type: Telegram node (sendPhoto)  
  - Role: Sends the generated chart image to the Telegram chat.  
  - Config:  
    - Uses Telegram API credentials.  
    - Sends photo using the chart URL.  
    - Chat ID is hardcoded (replaceable).  
  - Inputs: Chart URL or image from Technical Analysis output  
  - Outputs: Confirmation of message sent  
  - Edge cases: Invalid chat ID, Telegram API errors.

- **Send Analysis**  
  - Type: Telegram node (sendMessage)  
  - Role: Sends the textual stock analysis to Telegram chat.  
  - Config:  
    - Text content from AI Agent output.  
    - Chat ID hardcoded (replaceable).  
  - Inputs: AI Agent output  
  - Outputs: Confirmation of message sent  
  - Edge cases: Message too long, API errors.

---

#### 2.5 Scheduled Batch Processing

**Overview:**  
Periodically triggers batch analysis of stored tickers from Airtable, looping through each ticker to request AI analysis via HTTP webhook.

**Nodes Involved:**  
- Schedule Trigger  
- Get tokens (Airtable)  
- Loop Over Items  
- Run Agent (HTTP Request)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the batch process on a defined interval (default: every minute).  
  - Config: Interval set to 1 minute (default).  
  - Inputs: None  
  - Outputs: Trigger signal to start batch.

- **Get tokens**  
  - Type: Airtable node  
  - Role: Retrieves stored ticker symbols from Airtable "Tickers" table.  
  - Config: Uses Airtable API credentials, searches all records.  
  - Inputs: Schedule Trigger output  
  - Outputs: List of ticker records  
  - Edge cases: Airtable API limits, empty tables.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Iterates over each ticker record for processing.  
  - Config: Default batch size (1)  
  - Inputs: Airtable records  
  - Outputs: Single ticker per iteration

- **Run Agent**  
  - Type: HTTP Request  
  - Role: Calls the AI agent webhook with a prompt to analyze each ticker.  
  - Config:  
    - POST request to local webhook URL of AI agent.  
    - Body includes text: "Please analyze {{ticker}} stocks".  
    - Content-Type header set to application/json.  
  - Inputs: Each ticker from Loop Over Items  
  - Outputs: Response from AI agent webhook  
  - Edge cases: Webhook unavailable, network errors.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                           | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                 |
|-----------------------|----------------------------------|-----------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger                 | Receives Telegram messages               |                             | Switch                     | Replace Telegram connection                                                                |
| Switch                | Switch                          | Routes voice vs text messages            | Telegram Trigger             | Download File, Set Text     |                                                                                             |
| Download File          | Telegram (file download)         | Downloads voice message audio             | Switch (Voice)               | Transcribe                 |                                                                                             |
| Transcribe             | OpenAI Audio Transcription       | Transcribes audio to text                 | Download File                | AI Agent                   |                                                                                             |
| Set Text               | Set                             | Extracts text from text messages          | Switch (Text)                | AI Agent                   |                                                                                             |
| AI Agent               | LangChain AI Agent               | Processes queries, calls Get Chart tool  | Set Text, Transcribe         | Send Analysis              | Replace Chat ID                                                                             |
| Window Buffer Memory   | LangChain Memory Buffer Window   | Maintains conversation context            | AI Agent                    | AI Agent                   |                                                                                             |
| Get Chart              | LangChain Tool Workflow          | Generates stock chart URL                  | AI Agent (tool call)         | AI Agent                   |                                                                                             |
| Set Values             | Set                             | Prepares ticker and chart style params    | Workflow Input Trigger       | Get Chart URL              |                                                                                             |
| Get Chart URL          | HTTP Request                    | Calls chart image API                      | Set Values                  | Download Chart             | Replace API key (header = x-api-key) and chart settings                                    |
| Download Chart         | HTTP Request                    | Downloads chart image                      | Get Chart URL               | Technical Analysis         |                                                                                             |
| Technical Analysis     | OpenAI Image Analysis            | Performs detailed technical chart analysis| Download Chart              | Send Chart                 |                                                                                             |
| Send Chart             | Telegram (sendPhoto)             | Sends chart image to Telegram chat        | Technical Analysis          | response                   | Replace Chat ID                                                                             |
| Send Analysis          | Telegram (sendMessage)           | Sends textual analysis to Telegram chat   | AI Agent                   |                            | Replace Chat ID                                                                             |
| Schedule Trigger       | Schedule Trigger                | Triggers scheduled batch analyses         |                             | Get tokens                 |                                                                                             |
| Get tokens             | Airtable                       | Retrieves stored tickers                   | Schedule Trigger            | Loop Over Items            |                                                                                             |
| Loop Over Items        | Split In Batches                | Iterates over tickers                      | Get tokens                  | Run Agent                  |                                                                                             |
| Run Agent              | HTTP Request                   | Calls AI agent webhook for batch analysis | Loop Over Items             | Loop Over Items (continue) |                                                                                             |
| Workflow Input Trigger | Execute Workflow Trigger        | Accepts external workflow inputs           |                             | Set Values                 |                                                                                             |
| Set Text1              | Set                             | Extracts text from webhook input           | Webhook                     | AI Agent                   |                                                                                             |
| Webhook                | Webhook                        | Receives external HTTP requests            |                             | Set Text1                  |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Set Telegram API credentials.  
   - Position: Input node for user messages.

2. **Add Switch Node**  
   - Type: Switch  
   - Configure two outputs:  
     - "Voice": Condition checks if `message.voice.file_id` exists.  
     - "Text": Condition checks if `message.text` exists.  
   - Connect Telegram Trigger output to Switch input.

3. **Add Download File Node** (Voice branch)  
   - Type: Telegram node (download file)  
   - Configure to download file using `message.voice.file_id`.  
   - Connect Switch "Voice" output to Download File input.

4. **Add Transcribe Node**  
   - Type: OpenAI Audio Transcription  
   - Set operation to "transcribe".  
   - Use OpenAI API credentials.  
   - Connect Download File output to Transcribe input.

5. **Add Set Text Node** (Text branch)  
   - Type: Set  
   - Assign `text` field from `message.text`.  
   - Connect Switch "Text" output to Set Text input.

6. **Add AI Agent Node**  
   - Type: LangChain AI Agent  
   - Set input text to `={{ $json.text }}`.  
   - Configure system message with detailed instructions for stock analysis and tool usage.  
   - Use GPT-4o model.  
   - Add Get Chart as a tool (link to sub-workflow).  
   - Connect outputs of Transcribe and Set Text nodes to AI Agent input.

7. **Add Window Buffer Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Set session key to `"335458847"` (or unique session ID).  
   - Connect AI Agent memory input/output accordingly.

8. **Create Get Chart Sub-Workflow**  
   - Inputs: `ticker` (string), `chart_style` (string, default "candle").  
   - Logic: Calls external chart API to generate chart URL.  
   - Outputs: Chart URL in markdown format.  
   - Integrate this sub-workflow as a tool in AI Agent node.

9. **Add Send Analysis Node**  
   - Type: Telegram (sendMessage)  
   - Text: `={{ $json.output }}` from AI Agent.  
   - Set chat ID to your Telegram chat.  
   - Connect AI Agent output to Send Analysis input.

10. **Add Set Values Node** (for chart generation)  
    - Type: Set  
    - Assign `ticker` and `chart_style` from workflow inputs or AI outputs.  
    - Connect to Get Chart URL node.

11. **Add Get Chart URL Node**  
    - Type: HTTP Request  
    - POST to chart image API endpoint.  
    - Set JSON body with symbol, style, interval, studies, etc.  
    - Use HTTP header authentication with API key.  
    - Connect Set Values output to this node.

12. **Add Download Chart Node**  
    - Type: HTTP Request  
    - GET request to URL from Get Chart URL output.  
    - Connect Get Chart URL output to Download Chart input.

13. **Add Technical Analysis Node**  
    - Type: OpenAI Image Analysis  
    - Input: base64 image from Download Chart.  
    - Model: GPT-4o.  
    - Prompt: Detailed instructions for candlestick, RSI, Stochastic RSI analysis.  
    - Connect Download Chart output to Technical Analysis input.

14. **Add Send Chart Node**  
    - Type: Telegram (sendPhoto)  
    - File: Chart URL or image from Technical Analysis.  
    - Set chat ID.  
    - Connect Technical Analysis output to Send Chart input.

15. **Add Schedule Trigger Node**  
    - Type: Schedule Trigger  
    - Set interval (e.g., every minute).  
    - Connect to Get tokens node.

16. **Add Get tokens Node**  
    - Type: Airtable  
    - Configure to read tickers from Airtable base and table.  
    - Use Airtable API credentials.  
    - Connect Schedule Trigger output to this node.

17. **Add Loop Over Items Node**  
    - Type: Split In Batches  
    - Connect Get tokens output to Loop Over Items input.

18. **Add Run Agent Node**  
    - Type: HTTP Request  
    - POST to AI Agent webhook URL.  
    - Body: JSON with text `"Please analyze {{ $json.Name }} stocks"`.  
    - Set content-type header to application/json.  
    - Connect Loop Over Items output to Run Agent input.

19. **Add Webhook Node**  
    - Type: Webhook  
    - Configure path and POST method.  
    - Connect to Set Text1 node.

20. **Add Set Text1 Node**  
    - Type: Set  
    - Assign `text` from webhook body.  
    - Connect to AI Agent node.

21. **Replace all hardcoded chat IDs and API keys** with your own credentials in Telegram nodes, HTTP Request nodes, and Airtable nodes.

22. **Test the workflow** by sending messages to your Telegram bot and verifying AI responses and chart images.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Video guide demonstrating the complete process of building this trading agent automation using n8n and Telegram.                                                                                                                                                                                                                                                                                                                                                                                  | [YouTube Video](https://youtu.be/94vh6hSiP9s)                                                   |
| Workflow made by Mark Shcherbakov from community 5minAI.                                                                                                                                                                                                                                                                                                                                                                                                                                            | [LinkedIn](https://www.linkedin.com/in/marklowcoding/), [5minAI Community](https://www.skool.com/5minai) |
| Setup instructions: Prepare Airtable table for tickers, configure Telegram bot, replace credentials and API keys, configure API endpoints, then start interaction by messaging the bot with ticker symbols and chart styles.                                                                                                                                                                                                                                                                       | Sticky Note in workflow                                                                         |
| Chart styles supported: bar, candle (default), line, area, heikinAshi, hollowCandle, baseline, hiLo, column.                                                                                                                                                                                                                                                                                                                                                                                        | Mentioned in Get Chart tool description                                                        |
| Replace all Telegram chat IDs and API keys in nodes labeled with sticky notes "Replace Chat ID" and "Replace API key".                                                                                                                                                                                                                                                                                                                                                                            | Sticky notes on Telegram and HTTP Request nodes                                                |

---

This documentation provides a detailed, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.