Tesla Quant Trading AI Agent using Telegram + GPT-4.1 (Main InterFace)

https://n8nworkflows.xyz/workflows/tesla-quant-trading-ai-agent-using-telegram---gpt-4-1--main-interface--4092


# Tesla Quant Trading AI Agent using Telegram + GPT-4.1 (Main InterFace)

### 1. Workflow Overview

This workflow, titled **Tesla Quant Trading AI Agent n8n**, is a master orchestration agent designed to deliver daily and on-demand Tesla (TSLA) stock trading signals via Telegram, powered by GPT-4.1 and real-time market data. It integrates multiple specialized sub-workflows that analyze technical indicators, price patterns, and news sentiment to produce a comprehensive, structured trading report optimized for traders.

The workflow logically divides into the following blocks:

- **1.1 Input Reception & User Validation**: Captures incoming Telegram messages, validates authorized users, and timestamps the session.
- **1.2 Session Metadata & Memory Management**: Assigns session identifiers and maintains short-term memory for context in AI reasoning.
- **1.3 AI Processing & Sub-Agent Invocation**: Calls GPT-4.1-based AI agent that orchestrates several sub-agents for market data and news sentiment analysis.
- **1.4 Output Processing & Telegram Delivery**: Handles output message size constraints by splitting large reports and sends the final formatted HTML report back to Telegram.

This master workflow requires seven additional sub-workflows to be deployed and linked for full functionality, including financial market data analysis, news sentiment analysis with DeepSeek Chat API, and various time-frame specific technical indicator tools.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & User Validation

**Overview:**  
This block receives incoming Telegram messages, ensures only authorized users can trigger the workflow, and timestamps the request.

**Nodes Involved:**  
- Telegram Trigger  
- User Authentication (Replace Telegram ID)  
- Date & Time  
- Add SessionId  

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* Telegram Trigger Node  
  - *Role:* Listens for new Telegram messages (updates: "message") to start the workflow.  
  - *Configuration:* Uses Telegram API credentials linked to the user's bot.  
  - *Inputs:* HTTP webhook from Telegram  
  - *Outputs:* Emits incoming message JSON  
  - *Edge Cases:* Unauthorized users sending messages; network interruptions; malformed messages.  
  - *Sticky Note:* "This node listens for new Telegram messages to initiate report generation."

- **User Authentication (Replace Telegram ID)**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Filters incoming messages to allow only authorized Telegram user IDs.  
  - *Configuration:* Compares the Telegram message `from.id` against a placeholder Telegram ID ("<<replace your ID here>>"). Rejects unauthorized users by returning `{unauthorized: true}`.  
  - *Inputs:* Telegram Trigger output  
  - *Outputs:* Passes data only for authorized users  
  - *Edge Cases:* Must replace placeholder with actual numeric Telegram ID; unauthorized access attempts; expression failures if Telegram message structure changes.  
  - *Sticky Note:* "Ensures only authorized Telegram user IDs can trigger the agent."

- **Date & Time**  
  - *Type:* DateTime Node  
  - *Role:* Captures current system date and time for session tracking.  
  - *Configuration:* Default settings, outputs current timestamp.  
  - *Inputs:* Output of User Authentication  
  - *Outputs:* Adds timestamp data to workflow context  
  - *Edge Cases:* System clock errors or timezone mismatches.  
  - *Sticky Note:* "Captures the current system date and time for session tracking and report stamping."

- **Add SessionId**  
  - *Type:* Set Node  
  - *Role:* Assigns `sessionId` based on Telegram chat ID and stores current date/time for context continuity.  
  - *Configuration:*  
    - `sessionId` set from `Telegram Trigger` message chat ID  
    - `DateandTime` set from current date/time JSON field  
  - *Inputs:* Date & Time output  
  - *Outputs:* Passes enriched data with session metadata  
  - *Sticky Note:* "Assigns sessionId based on Telegram chat ID and stores current date/time for context continuity."

---

#### 2.2 Session Metadata & Memory Management

**Overview:**  
Maintains short-term memory buffer to provide context for AI reasoning and manages the GPT model used for generating the final report.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  

**Node Details:**  

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window Node  
  - *Role:* Maintains session state and message history during the conversation to improve contextual understanding.  
  - *Configuration:* Default window buffer settings, no custom parameters configured.  
  - *Inputs:* From AI Agent node (ai_memory connection)  
  - *Outputs:* Memory context for AI prompt generation  
  - *Edge Cases:* Memory overflow or data loss if session exceeds buffer size.  
  - *Sticky Note:* "Maintains state during session. Helps agent track message history and previous context."

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model Node  
  - *Role:* Provides GPT-4.1 (model variant `gpt-4o-mini`) language model for text generation and reasoning.  
  - *Configuration:*  
    - Model: GPT-4o-mini (OpenAI GPT-4.1 variant)  
    - No additional options configured  
  - *Inputs:* AI Agent node (ai_languageModel connection)  
  - *Outputs:* Generated text responses for AI agent  
  - *Credentials:* OpenAI API key configured  
  - *Edge Cases:* API rate limits, authentication errors, timeout, or prompt size limits.  
  - *Sticky Note:* "Uses OpenAI GPT (e.g., GPT-4o) to process signal analysis and produce final report text."

---

#### 2.3 AI Processing & Sub-Agent Invocation

**Overview:**  
This core block runs the main Tesla Quant Trading AI Agent node which orchestrates multiple sub-workflows (technical analysis and news sentiment) and composes the final structured trading report.

**Nodes Involved:**  
- Tesla Quant Trading AI Agent  
- Tesla Financial Market Data Analyst Tool  
- Tesla News and Sentiment Analyst Tool  

**Node Details:**  

- **Tesla Quant Trading AI Agent**  
  - *Type:* Langchain Agent Node  
  - *Role:* Master AI agent combining multi-timeframe technical analysis, candlestick pattern detection, and news sentiment into a professional Tesla trading report.  
  - *Configuration Highlights:*  
    - System message defines role and responsibilities, including tool connections and output formatting in Telegram HTML.  
    - Calls two sub-agents as Langchain Tool Workflows:  
      - Financial Market Data Analyst Tool  
      - News and Sentiment Analyst Tool  
    - Receives structured data inputs (indicators, sentiment scores, headlines).  
    - Produces final report text with spot and leveraged trade recommendations, confidence score, and sentiment summary.  
  - *Inputs:* Session metadata and user message text  
  - *Outputs:* Generated report text (HTML-formatted)  
  - *Edge Cases:* Sub-agent failures, incomplete data, malformed inputs, OpenAI API errors.  
  - *Sticky Note:* "Uses GPT model to combine technical and sentiment data into a structured TSLA swing-trading report. Calls Financial Market Analyst + Sentiment Analyst."

- **Tesla Financial Market Data Analyst Tool**  
  - *Type:* Langchain Tool Workflow Node  
  - *Role:* Aggregates multi-timeframe technical indicators and candlestick pattern signals for Tesla stock.  
  - *Configuration:* Connects to a published sub-workflow (`Tesla_Financial_Market_Data_Analyst`), accepts inputs for message and sessionId.  
  - *Inputs:* Provided by main AI agent node as tool input  
  - *Outputs:* Structured market data analysis results  
  - *Edge Cases:* Sub-workflow not deployed, API key missing, data fetch errors.  
  - *Sticky Note:* "Fetch technical indicator insights. Calls all 15m, 1h, 1d, and Klines sub-agents via webhook to retrieve structured market analysis."

- **Tesla News and Sentiment Analyst Tool**  
  - *Type:* Langchain Tool Workflow Node  
  - *Role:* Scrapes Tesla-related news headlines and computes sentiment scores using DeepSeek Chat API.  
  - *Configuration:* Connects to a published sub-workflow (`Tesla_News_and_Sentiment_Analyst`), accepts inputs for message and sessionId.  
  - *Inputs:* Provided by main AI agent node as tool input  
  - *Outputs:* Sentiment summary and top headlines  
  - *Edge Cases:* Missing DeepSeek API key, RSS feed failures, sub-workflow offline.  
  - *Sticky Note:* "Scrape Tesla News & Generate Sentiment. Pulls latest Tesla-related headlines from trusted RSS feeds and returns structured sentiment summary."

---

#### 2.4 Output Processing & Telegram Delivery

**Overview:**  
Handles the final report output by checking message length, splitting large messages into Telegram-compatible chunks, and sending formatted HTML reports back to the user.

**Nodes Involved:**  
- Splits message is more than 4000 characters  
- Telegram  

**Node Details:**  

- **Splits message is more than 4000 characters**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Splits the generated report text into 4000-character chunks to comply with Telegram message size limits.  
  - *Configuration:* Custom JS function that checks message length and splits if needed.  
  - *Inputs:* Output from Tesla Quant Trading AI Agent (report text in `json.output`)  
  - *Outputs:* One or more messages ready for sending  
  - *Edge Cases:* Unexpectedly large outputs; splitting may cut in middle of HTML tags (could affect formatting).  
  - *Sticky Note:* "Checks output message length and splits into 4000-character chunks if needed."

- **Telegram**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends the final formatted HTML trading report (or split parts) back to the originating Telegram user.  
  - *Configuration:*  
    - Message text set from the split message chunks  
    - Chat ID dynamically taken from the original Telegram Trigger message  
    - HTML formatting enabled, disables attribution append to keep report clean  
  - *Inputs:* Output from message splitting node  
  - *Outputs:* Sent Telegram messages  
  - *Credentials:* Telegram API credentials for the bot  
  - *Edge Cases:* Network errors, invalid chat ID, Telegram API limits.  
  - *Sticky Note:* "Sends formatted HTML report (or split parts) back to the original Telegram user."

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                                      | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                             |
|-----------------------------------|-----------------------------------|-----------------------------------------------------|----------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------|
| Telegram Trigger                  | Telegram Trigger                  | Listen for incoming Telegram messages                | (Webhook HTTP trigger)            | User Authentication                  | This node listens for new Telegram messages to initiate report generation.                             |
| User Authentication (Replace Telegram ID) | Code Node (JavaScript)             | Validate Telegram user ID                            | Telegram Trigger                 | Date & Time                         | Ensures only authorized Telegram user IDs can trigger the agent.                                      |
| Date & Time                      | DateTime Node                    | Capture current timestamp                            | User Authentication              | Add SessionId                      | Captures the current system date and time for session tracking and report stamping.                   |
| Add SessionId                    | Set Node                        | Assign sessionId and date/time metadata              | Date & Time                     | Tesla Quant Trading AI Agent        | Assigns sessionId based on Telegram chat ID and stores current date/time for context continuity.       |
| Simple Memory                   | Langchain Memory Buffer Window   | Maintain session memory for AI context               | Tesla Quant Trading AI Agent (ai_memory) | Tesla Quant Trading AI Agent (ai_memory) | Maintains state during session. Helps agent track message history and previous context.                |
| OpenAI Chat Model                | Langchain OpenAI Chat Model      | GPT model for language generation and reasoning     | Tesla Quant Trading AI Agent (ai_languageModel) | Tesla Quant Trading AI Agent (ai_languageModel) | Uses OpenAI GPT (e.g., GPT-4o) to process signal analysis and produce final report text.              |
| Tesla Quant Trading AI Agent     | Langchain Agent                  | Main AI agent orchestrating sub-agents and report creation | Add SessionId                  | Splits message is more than 4000 characters | Uses GPT model to combine technical and sentiment data into a structured TSLA swing-trading report. Calls Financial Market Analyst + Sentiment Analyst. |
| Tesla Financial Market Data Analyst Tool | Langchain Tool Workflow           | Fetch technical indicators and patterns              | Tesla Quant Trading AI Agent (ai_tool) | Tesla Quant Trading AI Agent (ai_tool) | Fetch technical indicator insights. Calls all 15m, 1h, 1d, and Klines sub-agents via webhook to retrieve structured market analysis.                |
| Tesla News and Sentiment Analyst Tool | Langchain Tool Workflow           | Fetch news sentiment and headlines                    | Tesla Quant Trading AI Agent (ai_tool) | Tesla Quant Trading AI Agent (ai_tool) | Scrape Tesla News & Generate Sentiment. Pulls latest Tesla-related headlines from trusted RSS feeds and returns structured sentiment summary.        |
| Splits message is more than 4000 characters | Code Node (JavaScript)             | Split output report into Telegram-compatible chunks  | Tesla Quant Trading AI Agent    | Telegram                          | Checks output message length and splits into 4000-character chunks if needed.                         |
| Telegram                        | Telegram Node (Send Message)     | Send final report messages back to Telegram user     | Splits message is more than 4000 characters | (End)                           | Sends formatted HTML report (or split parts) back to the original Telegram user.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates  
   - Credentials: Connect your Telegram Bot API token (created via @BotFather)  
   - Purpose: Receive user messages to initiate workflow.  

2. **Add User Authentication Node (Code Node)**  
   - Type: Code Node (JavaScript)  
   - Code: Compare incoming message user ID against your Telegram numeric ID (replace placeholder `<<replace your ID here>>`)  
   - Purpose: Allow only authorized users to proceed.  

3. **Insert Date & Time Node**  
   - Type: DateTime Node  
   - Default config to insert current timestamp  
   - Purpose: Timestamp the session for report context.  

4. **Add Set Node for Session Metadata**  
   - Type: Set Node  
   - Assign `sessionId` from Telegram chat ID (`{{$json["message"]["chat"]["id"]}}`)  
   - Assign `DateandTime` from current date output  
   - Purpose: Track session continuity.  

5. **Add Langchain Agent Node - Tesla Quant Trading AI Agent**  
   - Type: Langchain Agent Node  
   - Parameters:  
     - Text input: Use Telegram message text  
     - System message: Define agent role and output format as per the detailed system message in the original node  
     - Configure to call two tool workflows:  
       - Tesla Financial Market Data Analyst Tool  
       - Tesla News and Sentiment Analyst Tool  
   - Purpose: Orchestrate sub-agents and generate the final trading report.  

6. **Create Langchain Tool Workflow Nodes for Sub-Agents**  
   - Tesla Financial Market Data Analyst Tool: Link to published workflow ID for financial analysis  
   - Tesla News and Sentiment Analyst Tool: Link to published workflow ID for news sentiment  
   - Provide input mapping (message text and sessionId)  
   - Purpose: Fetch technical and sentiment data for the AI agent.  

7. **Insert Langchain Memory Buffer Window Node**  
   - Type: Memory Buffer Window  
   - Connect to AI agent’s memory port to maintain session context  
   - Purpose: Maintain conversation context for AI reasoning.  

8. **Add Langchain OpenAI Chat Model Node**  
   - Type: OpenAI Chat Model  
   - Set model to GPT-4o-mini (or GPT-4.1 equivalent)  
   - Provide OpenAI API credentials  
   - Purpose: Language model for AI agent text generation.  

9. **Add Code Node to Split Long Messages**  
   - Type: Code Node  
   - JavaScript to split messages >4000 chars into chunks  
   - Input: Output from AI agent (report text)  
   - Purpose: Comply with Telegram message size limits.  

10. **Add Telegram Send Node**  
    - Type: Telegram Node (Send message)  
    - Parameters:  
      - Text: Use chunked message parts  
      - Chat ID: Use original Telegram chat ID  
      - Disable append attribution  
    - Credentials: Same Telegram API token  
    - Purpose: Send final report to user.  

11. **Connect All Nodes in Order:**  
    Telegram Trigger → User Authentication → Date & Time → Add SessionId → Tesla Quant Trading AI Agent  
    Tesla Quant Trading AI Agent (ai_tool) → Tesla Financial Market Data Analyst Tool & Tesla News and Sentiment Analyst Tool  
    Tesla Quant Trading AI Agent (ai_memory) → Simple Memory  
    Tesla Quant Trading AI Agent (ai_languageModel) → OpenAI Chat Model  
    Tesla Quant Trading AI Agent → Splits message is more than 4000 characters → Telegram  

12. **Configure Credentials:**  
    - Telegram Bot API token (for both trigger and send nodes)  
    - OpenAI API key for GPT-4.1 model  
    - DeepSeek Chat API key in News & Sentiment Analyst sub-workflow  
    - Alpha Vantage Premium API key in sub-workflows for technical indicators  

13. **Deploy and Test:**  
    - Ensure all 7 required sub-workflows are imported and published as per instructions.  
    - Replace placeholder Telegram ID in User Authentication node.  
    - Trigger via Telegram message and verify output formatting and correctness.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Full system requires 7 connected sub-workflows including technical indicator tools and news sentiment tools.                  | See "🔌 Required Sub-Workflows" section in workflow description.                                     |
| DeepSeek Chat API key needed for News & Sentiment Analyst Tool.                                                                | https://deepseek.com                                                                                 |
| Alpha Vantage Premium API key required for technical indicator webhooks.                                                       | https://www.alphavantage.co/premium/                                                                |
| Telegram Bot creation instructions via @BotFather.                                                                            | https://t.me/BotFather                                                                               |
| Output reports are formatted as Telegram-ready HTML with bold tags and bullet points.                                          | System message in Tesla Quant Trading AI Agent node.                                                |
| Workflow architecture and prompts are intellectual property of Treasurium Capital Limited Company, no unauthorized rebranding. | Licensing section in workflow description.                                                          |
| Live demo video available showcasing the Tesla Quant Trading AI Agent in action.                                               | https://youtu.be/4638p9XDnF4 (YouTube demo video)                                                   |
| Support and contact via LinkedIn - Don Jayamaha.                                                                               | https://linkedin.com/in/donjayamahajr                                                               |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated n8n workflow respecting all current content policies and involving only legal and public data sources. No illegal or offensive material is included.