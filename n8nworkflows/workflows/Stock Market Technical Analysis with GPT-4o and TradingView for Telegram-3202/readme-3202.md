Stock Market Technical Analysis with GPT-4o and TradingView for Telegram

https://n8nworkflows.xyz/workflows/stock-market-technical-analysis-with-gpt-4o-and-tradingview-for-telegram-3202


# Stock Market Technical Analysis with GPT-4o and TradingView for Telegram

### 1. Workflow Overview

This n8n workflow, titled **Stock Market Technical Analysis with GPT-4o and TradingView for Telegram**, is designed to deliver AI-driven, professional-grade stock market technical analysis directly to Telegram users. It integrates advanced technical indicators and chart generation with conversational AI to provide actionable trading insights in real-time.

The workflow is logically divided into the following blocks:

- **1.1 User Interaction & Message Reception**: Captures incoming Telegram messages and triggers the workflow.
- **1.2 AI Processing & Context Management**: Uses GPT-4o via LangChain nodes to interpret user queries, maintain conversation context, and decide when to invoke chart generation.
- **1.3 Chart Generation & Download**: Requests and retrieves TradingView-based stock charts with multiple technical indicators.
- **1.4 Technical Analysis & Response Preparation**: Analyzes the downloaded chart data using OpenAI models and formats the response.
- **1.5 Response Delivery**: Sends the generated chart image and AI analysis back to the user on Telegram.
- **1.6 Sub-Workflow & Utility Nodes**: Includes code execution and workflow triggers for modularity and data handling.

---

### 2. Block-by-Block Analysis

#### 2.1 User Interaction & Message Reception

- **Overview:**  
  This block captures incoming messages from Telegram users to initiate the analysis process.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Listens for incoming messages from the Telegram bot webhook.  
    - Configuration: Default settings, webhook ID assigned for Telegram integration.  
    - Inputs: External Telegram messages.  
    - Outputs: Passes message data to the AI Agent node.  
    - Edge Cases: Telegram API downtime, invalid webhook setup, message format inconsistencies.

---

#### 2.2 AI Processing & Context Management

- **Overview:**  
  This block interprets user messages using GPT-4o, maintains conversation context, and determines if chart generation is needed.

- **Nodes Involved:**  
  - AI Agent  
  - Window Buffer Memory  
  - OpenAI Chat Model

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent node  
    - Role: Core conversational AI processor that handles user queries and orchestrates tools.  
    - Configuration: Connected to OpenAI Chat Model (GPT-4o) for language understanding and to Window Buffer Memory for context.  
    - Inputs: Telegram Trigger messages, AI language model, AI memory, and tool workflows.  
    - Outputs: Sends responses to Telegram node or triggers chart generation tool.  
    - Edge Cases: API rate limits, incomplete user input, failure in tool invocation.

  - **Window Buffer Memory**  
    - Type: LangChain Memory Buffer (Window)  
    - Role: Maintains a sliding window of conversation history to provide context-aware responses.  
    - Configuration: Default buffer size, linked to AI Agent.  
    - Inputs: Conversation data from AI Agent.  
    - Outputs: Context data back to AI Agent.  
    - Edge Cases: Memory overflow, context loss if buffer size is too small.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o language model capabilities for natural language understanding and generation.  
    - Configuration: Requires OpenAI API key credential.  
    - Inputs: Prompts from AI Agent.  
    - Outputs: Textual responses to AI Agent.  
    - Edge Cases: API key invalidation, request timeouts, model unavailability.

---

#### 2.3 Chart Generation & Download

- **Overview:**  
  This block generates TradingView charts with technical indicators and downloads the resulting chart images for analysis.

- **Nodes Involved:**  
  - Symbol And ChatId  
  - Get Chart (HTTP Request)  
  - Download Chart (HTTP Request)  
  - GetChart (LangChain Tool Workflow)

- **Node Details:**  
  - **Symbol And ChatId**  
    - Type: Set node  
    - Role: Extracts and sets the stock symbol and Telegram chat ID for downstream requests.  
    - Configuration: Uses expressions to parse input data and prepare parameters for chart requests.  
    - Inputs: Output from Code node.  
    - Outputs: Passes parameters to Get Chart HTTP Request node.  
    - Edge Cases: Missing or malformed symbol, invalid chat ID.

  - **Get Chart**  
    - Type: HTTP Request node  
    - Role: Sends a request to chart-img.com or TradingView API to generate a chart image URL with technical indicators (candlestick, Bollinger Bands, RSI, volume).  
    - Configuration: Configured with chart-img.com API key, URL parameters for indicators, and symbol.  
    - Inputs: Parameters from Symbol And ChatId node.  
    - Outputs: Chart image URL to Download Chart node.  
    - Edge Cases: API quota exceeded, invalid API key, network errors.

  - **Download Chart**  
    - Type: HTTP Request node  
    - Role: Downloads the actual chart image using the URL obtained from Get Chart.  
    - Configuration: Set to download binary data (image).  
    - Inputs: URL from Get Chart node.  
    - Outputs: Binary image data to Analysis node.  
    - Edge Cases: Broken URL, timeout, large image size causing delays.

  - **GetChart**  
    - Type: LangChain Tool Workflow  
    - Role: Invoked by AI Agent as a tool to generate charts on demand within the AI conversation flow.  
    - Configuration: Linked as an AI tool in AI Agent node.  
    - Inputs: Triggered by AI Agent when a stock symbol is detected.  
    - Outputs: Returns chart data or image URL back to AI Agent.  
    - Edge Cases: Tool workflow failure, miscommunication between AI Agent and tool.

---

#### 2.4 Technical Analysis & Response Preparation

- **Overview:**  
  This block processes the downloaded chart image with OpenAI to generate detailed technical analysis and prepares the response message.

- **Nodes Involved:**  
  - Analysis (OpenAI)  
  - Response (Set)  
  - Code

- **Node Details:**  
  - **Analysis**  
    - Type: LangChain OpenAI node  
    - Role: Uses GPT-4o to analyze chart data and generate technical insights including candlestick patterns, MACD signals, volume trends, and support/resistance levels.  
    - Configuration: Requires OpenAI API key, prompt templates tailored for technical analysis.  
    - Inputs: Binary chart image from Download Chart node.  
    - Outputs: Textual analysis to Response node.  
    - Edge Cases: Image processing limitations, API errors, ambiguous chart data.

  - **Response**  
    - Type: Set node  
    - Role: Formats the final message payload including text analysis and chart image for Telegram delivery.  
    - Configuration: Sets message text, attaches image binary data, and includes chat ID.  
    - Inputs: Analysis text and image data.  
    - Outputs: Passes formatted message to Telegram node.  
    - Edge Cases: Incorrect formatting causing Telegram API rejection.

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Performs data extraction, transformation, or preparation tasks such as parsing user input or preparing parameters for Symbol And ChatId node.  
    - Configuration: Custom JavaScript code tailored to workflow needs.  
    - Inputs: Triggered by sub-workflow execution or other nodes.  
    - Outputs: Data to Symbol And ChatId node.  
    - Edge Cases: Script errors, unexpected input data formats.

---

#### 2.5 Response Delivery

- **Overview:**  
  Sends the prepared analysis and chart image back to the user on Telegram.

- **Nodes Involved:**  
  - Telegram  
  - Response2 (Telegram)

- **Node Details:**  
  - **Telegram**  
    - Type: Telegram node (send message)  
    - Role: Delivers the final technical analysis text and chart image to the user chat.  
    - Configuration: Uses Telegram Bot API token, configured to send messages with media attachments.  
    - Inputs: Formatted message from Response node.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Telegram API limits, chat ID errors, message size limits.

  - **Response2**  
    - Type: Telegram node (send message)  
    - Role: Sends immediate AI Agent responses back to Telegram users for conversational interaction.  
    - Configuration: Linked directly to AI Agent output for real-time replies.  
    - Inputs: Text responses from AI Agent.  
    - Outputs: None.  
    - Edge Cases: Same as Telegram node.

---

#### 2.6 Sub-Workflow & Utility Nodes

- **Overview:**  
  Contains nodes for modular workflow execution and sticky notes for documentation.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Sticky Note1  
  - Sticky Note

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows this workflow to be triggered as a sub-workflow by other workflows.  
    - Configuration: Default, no parameters specified.  
    - Inputs: External workflow calls.  
    - Outputs: Triggers Code node.  
    - Edge Cases: Misconfigured sub-workflow calls.

  - **Sticky Note1 & Sticky Note**  
    - Type: Sticky Note  
    - Role: Provide visual documentation or comments within the workflow editor.  
    - Configuration: Content is empty in this export.  
    - Inputs/Outputs: None.  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                             | Input Node(s)                 | Output Node(s)               | Sticky Note                          |
|----------------------------|----------------------------------|---------------------------------------------|------------------------------|------------------------------|------------------------------------|
| Telegram Trigger            | Telegram Trigger                 | Captures incoming Telegram messages         | External Telegram             | AI Agent                     |                                    |
| AI Agent                   | LangChain Agent                  | Processes user input, manages AI conversation | Telegram Trigger, OpenAI Chat Model, Window Buffer Memory, GetChart | Response2                    |                                    |
| Window Buffer Memory        | LangChain Memory Buffer Window  | Maintains conversation context               | AI Agent                     | AI Agent                     |                                    |
| OpenAI Chat Model           | LangChain OpenAI Chat Model     | Provides GPT-4o language model                | AI Agent                     | AI Agent                     |                                    |
| GetChart                   | LangChain Tool Workflow         | Generates TradingView charts on demand       | AI Agent (ai_tool)           | AI Agent                     |                                    |
| Response2                   | Telegram                        | Sends AI Agent conversational replies        | AI Agent                     | None                        |                                    |
| When Executed by Another Workflow | Execute Workflow Trigger       | Allows sub-workflow execution                  | External                     | Code                         |                                    |
| Code                       | Code                            | Parses input and prepares parameters          | When Executed by Another Workflow | Symbol And ChatId           |                                    |
| Symbol And ChatId           | Set                             | Extracts stock symbol and chat ID             | Code                         | Get Chart                   |                                    |
| Get Chart                  | HTTP Request                    | Requests chart image URL from chart-img.com   | Symbol And ChatId            | Download Chart               |                                    |
| Download Chart             | HTTP Request                    | Downloads chart image binary                   | Get Chart                   | Analysis                    |                                    |
| Analysis                   | LangChain OpenAI                | Generates technical analysis from chart image | Download Chart              | Response                    |                                    |
| Response                   | Set                             | Formats message with analysis and image       | Analysis                     | Telegram                    |                                    |
| Telegram                   | Telegram                        | Sends final analysis and chart image to user | Response                    | None                        |                                    |
| Sticky Note1               | Sticky Note                    | Visual comment (empty content)                 | None                        | None                        |                                    |
| Sticky Note                | Sticky Note                    | Visual comment (empty content)                 | None                        | None                        |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API token (via credentials).  
   - Set to listen for incoming messages.  
   - Position: Start of workflow.

2. **Create OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Configure with OpenAI API key credential.  
   - Use GPT-4o model or equivalent.  
   - No special parameters needed beyond API key.

3. **Create Window Buffer Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Default buffer size (e.g., 5 messages).  
   - Connect to AI Agent for conversation context.

4. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Connect inputs:  
     - Telegram Trigger (main input)  
     - OpenAI Chat Model (ai_languageModel)  
     - Window Buffer Memory (ai_memory)  
     - GetChart (ai_tool)  
   - No additional parameters needed; ensure tool workflow is linked.

5. **Create GetChart Node (Tool Workflow)**  
   - Type: LangChain Tool Workflow  
   - This is a sub-workflow that handles chart generation.  
   - Configure to accept stock symbol input and return chart URL or data.  
   - Connect output to AI Agent (ai_tool).

6. **Create Response2 Node**  
   - Type: Telegram node (send message)  
   - Configure with Telegram Bot API credentials.  
   - Connect input from AI Agent main output for conversational replies.

7. **Create When Executed by Another Workflow Node**  
   - Type: Execute Workflow Trigger  
   - Allows this workflow to be triggered externally.

8. **Create Code Node**  
   - Type: Code (JavaScript)  
   - Write code to parse incoming data and extract stock symbol and chat ID.  
   - Connect input from Execute Workflow Trigger node.

9. **Create Symbol And ChatId Node**  
   - Type: Set  
   - Use expressions to set variables: stock symbol and Telegram chat ID.  
   - Connect input from Code node.

10. **Create Get Chart Node (HTTP Request)**  
    - Type: HTTP Request  
    - Configure to call chart-img.com or TradingView API with parameters: symbol, indicators (candlestick, Bollinger Bands, RSI, volume).  
    - Use API key credential for chart-img.com.  
    - Connect input from Symbol And ChatId node.

11. **Create Download Chart Node (HTTP Request)**  
    - Type: HTTP Request  
    - Configure to download binary image from URL returned by Get Chart node.  
    - Connect input from Get Chart node.

12. **Create Analysis Node**  
    - Type: LangChain OpenAI node  
    - Configure with OpenAI API key.  
    - Use prompt templates designed for technical analysis of charts.  
    - Connect input from Download Chart node.

13. **Create Response Node (Set)**  
    - Type: Set  
    - Format the message to include analysis text and attach the chart image binary.  
    - Set chat ID for Telegram delivery.  
    - Connect input from Analysis node.

14. **Create Telegram Node (Send Message)**  
    - Type: Telegram  
    - Configure with Telegram Bot API credentials.  
    - Connect input from Response node.  
    - This node sends the final analysis and chart image to the user.

15. **Connect Workflow**  
    - Telegram Trigger → AI Agent → Response2 (for conversational replies)  
    - When Executed by Another Workflow → Code → Symbol And ChatId → Get Chart → Download Chart → Analysis → Response → Telegram (final delivery)  
    - AI Agent uses GetChart as a tool workflow for chart generation.

16. **Credential Setup**  
    - Telegram Bot API token for Telegram nodes.  
    - OpenAI API key for OpenAI Chat Model and Analysis nodes.  
    - chart-img.com API key for Get Chart HTTP Request node.

17. **Activate Workflow**  
    - Test by sending messages to Telegram bot.  
    - Verify chart generation and AI analysis responses.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow integrates GPT-4o via LangChain for advanced conversational AI capabilities.      | n8n LangChain nodes documentation: https://docs.n8n.io/integrations/builtin/nodes/langchain/    |
| Chart generation uses chart-img.com API for TradingView-style charts with technical indicators. | Chart-img.com API documentation: https://chart-img.com/docs/                                    |
| Telegram Bot setup requires token from @BotFather on Telegram.                                  | Telegram Bot API docs: https://core.telegram.org/bots/api                                       |
| Ensure OpenAI API key has access to GPT-4o or equivalent model for best results.                 | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4                                  |
| Conversation context is managed with a sliding window memory to maintain relevant chat history. | LangChain memory concepts: https://python.langchain.com/docs/modules/memory/                    |
| Troubleshooting tips include checking API credentials, webhook setup, and API quota limits.     | Refer to the Setup Instructions section above for common issues.                                |

---

This document fully describes the workflow structure, node configurations, and operational logic to enable advanced users and AI agents to understand, reproduce, and maintain the Stock Market Technical Analysis Bot in n8n.