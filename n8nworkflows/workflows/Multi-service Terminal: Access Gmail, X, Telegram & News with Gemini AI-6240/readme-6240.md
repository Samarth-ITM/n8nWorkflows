Multi-service Terminal: Access Gmail, X, Telegram & News with Gemini AI

https://n8nworkflows.xyz/workflows/multi-service-terminal--access-gmail--x--telegram---news-with-gemini-ai-6240


# Multi-service Terminal: Access Gmail, X, Telegram & News with Gemini AI

### 1. Workflow Overview

This workflow, titled **"Multi-service Terminal: Access Gmail, X, Telegram & News with Gemini AI"**, implements a multi-functional AI-driven console interface. It enables users to interact via webhook with a single AI agent that can query multiple external services and APIs, including Gmail, X (formerly Twitter), Telegram, news RSS feeds, stock market data, and Google Calendar. The AI agent processes natural language commands and dispatches queries to the appropriate service nodes, then synthesizes and returns concise, often cynical and dry, responses back to the user.

**Target Use Cases:**

- Unified access to diverse information sources via a chat-like console interface.
- Quick retrieval of emails, tweets, news headlines, stock quotes, calendar events.
- Sending Telegram messages through natural language commands.
- Using a conversational AI assistant powered by Google Gemini 2.5 Flash model for understanding and orchestrating the multi-service queries.
- Maintaining conversational context across sessions with a simple memory buffer.

**Logical Blocks:**

- **1.1 Input Reception:** Receives user input via webhook ("Console In").
- **1.2 Conversational Memory:** Maintains session context with "Simple Memory".
- **1.3 AI Processing:** Handles natural language understanding and orchestration with the "AI Agent", powered by Google Gemini 2.5 Flash.
- **1.4 External Service Queries:** Multiple nodes perform specific external API calls on behalf of the AI agent:
  - Gmail inbox search
  - X (Twitter) search
  - Telegram message sending
  - News headlines via RSS feed
  - Stock market data lookup
  - Google Calendar event search
  - Date & Time retrieval
- **1.5 Output Delivery:** Sends processed AI responses back to the requester via webhook response ("Console out").
- **1.6 Documentation Aids:** Sticky notes document input/output roles for clarity.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block receives user commands from an external client (e.g., a chat interface) via an HTTP webhook.

**Nodes Involved:**  
- Console In (Webhook)

**Node Details:**  

- **Console In**  
  - Type: Webhook  
  - Role: Entry point receiving user commands as HTTP POST requests.  
  - Configuration:  
    - Webhook path unique identifier for incoming queries.  
    - Response mode set to respond from a downstream node ("Console out").  
  - Inputs: External HTTP requests.  
  - Outputs: Sends JSON containing user query to "AI Agent".  
  - Edge Cases:  
    - Invalid or malformed requests may cause webhook errors.  
    - Network or authorization issues blocking webhook access.  
  - Notes:  
    - Linked sticky note indicates example input: "what's in my inbox?"  

---

#### 1.2 Conversational Memory

**Overview:**  
Maintains short-term conversational context per user session to enable contextual AI responses.

**Nodes Involved:**  
- Simple Memory

**Node Details:**  

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Stores and retrieves last 10 messages in conversation, keyed by session ID.  
  - Configuration:  
    - Session key extracted from incoming JSON at `$json.query.sessionId`.  
    - Context window length set to 10 messages.  
  - Inputs: User queries and AI responses.  
  - Outputs: Contextual conversation history sent to "AI Agent".  
  - Edge Cases:  
    - Missing or malformed session ID disables memory context.  
    - Excessive context length may cause latency or token limit issues.  

---

#### 1.3 AI Processing

**Overview:**  
Central AI node interprets user queries, decides which external tool to invoke, and generates concise, dry, cynical responses. It uses a Google Gemini 2.5 Flash LLM and integrates multiple tool plugins.

**Nodes Involved:**  
- AI Agent  
- LLM Gemini 2.5 Flash

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Orchestrates multi-tool AI processing using defined tools and memory context.  
  - Configuration:  
    - Input text extracted from `{{$json.query.prompt}}`.  
    - System message instructs AI on available tools, tone (short, cynical, dry), and usage constraints (e.g., X rate limiting).  
    - Uses tools: News search, Stock market, X search, Date/time, Telegram send, Gmail inbox, Google Calendar, RSS headlines, default knowledge.  
  - Inputs: User prompt, memory context, LLM output, and tool outputs.  
  - Outputs: Final AI response text sent to "Console out".  
  - Edge Cases:  
    - Rate limiting on X searches requires delays or user notification.  
    - API failures from external tools must be handled gracefully.  
    - Complex or ambiguous user queries may cause misrouting or nonsensical results.  
- **LLM Gemini 2.5 Flash**  
  - Type: Google Gemini Chat LLM  
  - Role: Provides natural language generation and understanding capabilities for the AI Agent.  
  - Configuration:  
    - Model: "models/gemini-2.5-flash"  
    - Max output tokens: 2048  
    - Credentials: Google Palm API key required.  
  - Inputs: Prompt and context from AI Agent.  
  - Outputs: Generated text and tool invocation instructions.  
  - Edge Cases:  
    - API quota or authentication errors.  
    - Token limit truncation.  
    - Network latency or failure.  

---

#### 1.4 External Service Queries

**Overview:**  
This block contains nodes that perform specific API calls to external services, triggered by AI Agent tool requests.

**Nodes Involved:**  
- Date & Time  
- Search Tweets (X)  
- Stock info  
- RSS news  
- Send Telegram  
- Read Gmail  
- Check Google Calendar

**Node Details:**  

- **Date & Time**  
  - Type: DateTime Tool  
  - Role: Provides current date/time information.  
  - Config: Default options, no parameters.  
  - Inputs: Triggered by AI Agent tool request.  
  - Outputs: Date/time data returned to AI Agent.  
  - Edge Cases: None significant.  

- **Search Tweets**  
  - Type: Twitter Tool  
  - Role: Searches X (Twitter) for up to 10 tweets matching user query.  
  - Config:  
    - Search text dynamically provided by AI output from key 'Search_Term'.  
    - Includes tweet text and author ID fields.  
    - Manual description clarifies function.  
    - Credentials: Twitter OAuth2 required.  
  - Inputs: AI Agent tool request.  
  - Outputs: Tweet data to AI Agent.  
  - Edge Cases:  
    - Strict rate limits on X API calls, must implement throttling.  
    - Possible 'Too Many Requests' errors.  

- **Stock info**  
  - Type: MarketStack Tool  
  - Role: Retrieves stock market ticker info (quotes, news).  
  - Config:  
    - Stock symbol dynamically from AI key 'Ticker'.  
    - Manual description provided.  
    - Credentials: MarketStack API key required.  
  - Inputs: AI Agent tool request.  
  - Outputs: Stock data to AI Agent.  
  - Edge Cases:  
    - Invalid ticker symbols.  
    - API key limits or failures.  

- **RSS news**  
  - Type: RSS Feed Read Tool  
  - Role: Fetches latest news headlines from Clarin newspaper RSS.  
  - Config:  
    - URL: https://www.clarin.com/rss/lo-ultimo/  
    - SSL verification enabled.  
  - Inputs: AI Agent tool request.  
  - Outputs: News headlines sent to AI Agent.  
  - Edge Cases:  
    - Network or RSS feed availability issues.  

- **Send Telegram**  
  - Type: Telegram Tool  
  - Role: Sends text messages to a predefined Telegram chat ID.  
  - Config:  
    - Text dynamically sourced from AI key 'Text'.  
    - Chat ID statically set (placeholder "YOUR_CHAT_ID").  
    - Append attribution disabled.  
    - Credentials: Telegram Bot API key required.  
  - Inputs: AI Agent tool request.  
  - Outputs: Confirmation or error to AI Agent.  
  - Edge Cases:  
    - Invalid chat ID or bot token.  
    - Telegram API downtime or limits.  

- **Read Gmail**  
  - Type: Gmail Tool  
  - Role: Searches Gmail inbox, returning up to 10 messages.  
  - Config:  
    - No filters applied (fetch all).  
    - Credentials: OAuth2 Gmail account required.  
  - Inputs: AI Agent tool request.  
  - Outputs: Email data to AI Agent.  
  - Edge Cases:  
    - OAuth token expiration or denial.  
    - Gmail API quota exceeded.  

- **Check Google Calendar**  
  - Type: Google Calendar Tool  
  - Role: Retrieves up to 10 upcoming calendar events for specified email.  
  - Config:  
    - Email address statically set (placeholder "YOUR_EMAIL").  
    - Credentials: Google Calendar OAuth2 required.  
  - Inputs: AI Agent tool request.  
  - Outputs: Calendar events to AI Agent.  
  - Edge Cases:  
    - Permission or authentication failures.  
    - Empty or inaccessible calendar.  

---

#### 1.5 Output Delivery

**Overview:**  
Sends the AI agent’s final response back to the requester via webhook HTTP response.

**Nodes Involved:**  
- Console out (Respond to Webhook)

**Node Details:**  

- **Console out**  
  - Type: Respond to Webhook  
  - Role: Sends final AI-generated text back as HTTP response to original webhook call.  
  - Config: Default response options.  
  - Inputs: AI Agent output text.  
  - Outputs: HTTP response to user client.  
  - Edge Cases:  
    - Network timeouts or client disconnects.  

---

#### 1.6 Documentation Aids

**Overview:**  
Sticky note nodes provide inline documentation visible in the n8n editor to clarify input/output semantics.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**  

- **Sticky Note**  
  - Content: "## In\nReceives console command. Example: \"what's in my inbox?\""  
  - Position: Near Console In node.  

- **Sticky Note1**  
  - Content: "## Out\nSend AI agent response back to the console. Example: \"AI $ you have 3 unread emails\""  
  - Position: Near Console out node.  

---

### 3. Summary Table

| Node Name          | Node Type                           | Functional Role                                  | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                   |
|--------------------|-----------------------------------|-------------------------------------------------|-------------------------|-------------------------|-----------------------------------------------------------------------------------------------|
| Console In         | Webhook                           | Receives user commands via HTTP                  | -                       | AI Agent                | ## In Receives console command. Example: "what's in my inbox?"                               |
| Simple Memory      | LangChain Memory Buffer Window    | Maintains conversational context                  | Console In              | AI Agent                |                                                                                               |
| LLM Gemini 2.5 Flash| LangChain Google Gemini LLM       | Provides AI natural language understanding/generation | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                               |
| AI Agent           | LangChain Agent                   | Orchestrates multi-tool AI response               | Console In, Simple Memory, LLM Gemini 2.5 Flash, External Tools | Console out             | ## Out Send AI agent response back to the console. Example: "AI $ you have 3 unread emails"  |
| Date & Time        | DateTime Tool                    | Provides current date and time                     | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| Search Tweets      | Twitter Tool                     | Searches X/Twitter for tweets                      | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| Stock info         | MarketStack Tool                 | Retrieves stock market data                         | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| RSS news           | RSS Feed Read Tool               | Retrieves latest news headlines                     | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| Send Telegram      | Telegram Tool                   | Sends Telegram messages                             | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| Read Gmail         | Gmail Tool                      | Searches Gmail inbox                                | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| Check Google Calendar| Google Calendar Tool            | Retrieves calendar events                           | AI Agent (ai_tool)      | AI Agent                |                                                                                               |
| Console out        | Respond to Webhook              | Sends AI response back via webhook HTTP response  | AI Agent                | -                       |                                                                                               |
| Sticky Note        | Sticky Note                      | Provides documentation on input                    | -                       | -                       | ## In Receives console command. Example: "what's in my inbox?"                               |
| Sticky Note1       | Sticky Note                      | Provides documentation on output                   | -                       | -                       | ## Out Send AI agent response back to the console. Example: "AI $ you have 3 unread emails"  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Console In":**  
   - Type: Webhook  
   - Set a unique webhook path (e.g., "8d6b9886-7cc4-470f-84fa-02954655b297")  
   - Response mode: set to "responseNode"  
   - This node receives the user query JSON with field `query.prompt` and `query.sessionId`.

2. **Create LangChain Memory Node "Simple Memory":**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: expression `{{$json.query.sessionId}}`  
   - Session ID type: custom key  
   - Context window length: 10 messages

3. **Create Google Gemini LLM Node "LLM Gemini 2.5 Flash":**  
   - Type: LangChain Model Chat Google Gemini  
   - Model Name: `models/gemini-2.5-flash`  
   - Max output tokens: 2048  
   - Credentials: Configure Google Palm API credentials with your Google Gemini API key.

4. **Create LangChain Agent Node "AI Agent":**  
   - Type: LangChain Agent  
   - Text input: expression `={{ $json.query.prompt }}`  
   - System message: Provide detailed instructions on tool usage, tone, and constraints as in the original workflow.  
   - Tools: Enable and configure the following tools:  
     - Date & Time (link to Date & Time node)  
     - Search Tweets (X) (link to Search Tweets node)  
     - Stock Info (link to Stock info node)  
     - RSS News (link to RSS news node)  
     - Send Telegram (link to Send Telegram node)  
     - Read Gmail (link to Read Gmail node)  
     - Check Google Calendar (link to Check Google Calendar node)  
   - Memory: link to "Simple Memory" node  
   - Language Model: link to "LLM Gemini 2.5 Flash"

5. **Create External Tool Nodes:**

   - **Date & Time:**  
     - Type: DateTime Tool  
     - Default configuration.

   - **Search Tweets:**  
     - Type: Twitter Tool  
     - Operation: Search  
     - Limit: 10  
     - Search text: expression `={{ $fromAI('Search_Term', 'Use the search term provided', 'string') }}`  
     - Tweet fields: text, author_id  
     - Credentials: Twitter OAuth2 set with your X account.

   - **Stock info:**  
     - Type: MarketStack Tool  
     - Resource: ticker  
     - Symbol: expression `={{ $fromAI('Ticker', '', 'string') }}`  
     - Credentials: MarketStack API key.

   - **RSS news:**  
     - Type: RSS Feed Read Tool  
     - URL: https://www.clarin.com/rss/lo-ultimo/  
     - Ignore SSL: false (default)

   - **Send Telegram:**  
     - Type: Telegram Tool  
     - Text: expression `={{ $fromAI('Text', '', 'string') }}`  
     - Chat ID: set your Telegram chat ID string  
     - Append attribution: false  
     - Credentials: Telegram Bot API key.

   - **Read Gmail:**  
     - Type: Gmail Tool  
     - Operation: getAll  
     - Limit: 10  
     - Filters: none  
     - Credentials: Gmail OAuth2.

   - **Check Google Calendar:**  
     - Type: Google Calendar Tool  
     - Operation: getAll  
     - Limit: 10  
     - Calendar email: set your Google Calendar email address  
     - Credentials: Google Calendar OAuth2.

6. **Create Webhook Response Node "Console out":**  
   - Type: Respond to Webhook  
   - Default configuration  
   - Connect input from "AI Agent" node.

7. **Connect nodes as follows:**  
   - Console In → Simple Memory → AI Agent → Console out  
   - AI Agent → LLM Gemini 2.5 Flash  
   - AI Agent → Date & Time, Search Tweets, Stock info, RSS news, Send Telegram, Read Gmail, Check Google Calendar (as ai_tool inputs)  
   - LLM Gemini 2.5 Flash → AI Agent (ai_languageModel)  
   - Simple Memory → AI Agent (ai_memory)  

8. **Add Sticky Note nodes for documentation (optional):**  
   - Near "Console In": Content "## In\nReceives console command. Example: \"what's in my inbox?\""  
   - Near "Console out": Content "## Out\nSend AI agent response back to the console. Example: \"AI $ you have 3 unread emails\""

9. **Set all credential nodes with your actual API keys and OAuth2 credentials before activating the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini 2.5 Flash model for advanced conversational AI capabilities.                   | [Google Gemini API](https://developers.generativeai.google/api/gemini)                         |
| Rate limiting on X (Twitter) API calls is significant; implement delays or inform users on "Too Many Requests".| See sticky note and system message in AI Agent node configuration                              |
| RSS feed source is Clarin newspaper's latest headlines: https://www.clarin.com/rss/lo-ultimo/                  | Live RSS feed URL                                                                              |
| Telegram messages require a bot token and chat ID; ensure bot permissions to post in target chat.              | Telegram Bot API documentation: https://core.telegram.org/bots/api                              |
| Gmail and Google Calendar nodes require OAuth2 credentials with appropriate scopes for inbox and calendar read. | Google API Console setup required                                                              |
| The AI Agent enforces a dry, cynical, and short response style for user interaction.                            | System prompt in AI Agent node                                                                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.