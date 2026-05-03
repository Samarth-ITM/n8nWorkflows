Manage Schedule & Contacts with Telegram Bot using GPT-4o-mini & Google Services

https://n8nworkflows.xyz/workflows/manage-schedule---contacts-with-telegram-bot-using-gpt-4o-mini---google-services-7853


# Manage Schedule & Contacts with Telegram Bot using GPT-4o-mini & Google Services

---

### 1. Workflow Overview

This workflow implements an AI-powered personal assistant Telegram bot that integrates GPT-4o-mini with Google services to manage user schedules, contacts, and emails. It is designed to receive user messages via Telegram, process requests with an AI agent enhanced by multiple tools (calendar, contacts, web search, Wikipedia), and respond appropriately. The workflow supports fetching daily calendar events, retrieving and sending emails, searching the web or Wikipedia, and maintaining conversational context.

**Logical Blocks:**

- **1.1 Input Reception:** Receives user messages from Telegram.
- **1.2 AI Processing:** Uses a LangChain-based AI agent running GPT-4o-mini to interpret and respond to user input, including maintaining conversation memory.
- **1.3 AI Tools Integration:** Provides various auxiliary tools (Google Calendar, Gmail, Google Sheets for contacts, Wikipedia, SerpAPI web search) to extend AI capabilities.
- **1.4 Output Delivery:** Sends AI-generated responses back to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming messages on a configured Telegram bot and triggers the workflow.
- **Nodes Involved:** Telegram Trigger
- **Node Details:**

  - **Telegram Trigger**
    - **Type:** Telegram Trigger node
    - **Role:** Entry point for incoming Telegram messages.
    - **Configuration:** Listens for "message" updates only; requires Telegram Bot token credential (from @BotFather).
    - **Key variables:** Extracts message text and chat ID from incoming Telegram updates.
    - **Connections:** Output connected to AI Agent node's main input.
    - **Edge Cases:** Missing or invalid bot token; Telegram API downtime; unsupported update types.
    - **Version:** 1.1

#### 2.2 AI Processing

- **Overview:** Processes input text using a LangChain AI agent with GPT-4o-mini, enriched by conversation memory and multiple AI tools to fulfill complex personal assistant tasks.
- **Nodes Involved:** AI Agent, OpenAI Chat Model, Conversation Memory
- **Node Details:**

  - **AI Agent**
    - **Type:** LangChain AI Agent node
    - **Role:** Central AI logic processor orchestrating language model and tools.
    - **Configuration:**
      - Input text dynamically set from Telegram Trigger message text.
      - System message defines assistant role and constraints, e.g., instructs to address user as "USER_NAME" and details calendar and email handling instructions.
      - Uses GPT-4o-mini as language model.
      - Calls out to integrated tools for contacts, calendar, email, Wikipedia, and web search.
    - **Key expressions:** `{{$json.message.text}}` for input text; system message includes date/time with `{{$now}}`.
    - **Connections:** Input from Telegram Trigger; outputs to Telegram Response; AI tools inputs from Gmail, Google Calendar, Google Sheets (contacts), Wikipedia, SerpAPI; memory input from Conversation Memory; language model input from OpenAI Chat Model.
    - **Edge Cases:** API key issues; timeout or rate limits on OpenAI; failure to parse inputs or tool outputs; malformed user queries.
    - **Version:** 1.7

  - **OpenAI Chat Model**
    - **Type:** LangChain OpenAI Chat Model node
    - **Role:** Provides GPT-4o-mini AI language model backend.
    - **Configuration:** Model set explicitly to "gpt-4o-mini"; requires OpenAI API key credential.
    - **Connections:** Input from AI Agent (ai_languageModel); output back to AI Agent.
    - **Edge Cases:** API quota exceeded; invalid API key; network errors.
    - **Version:** 1.2

  - **Conversation Memory**
    - **Type:** LangChain Memory Buffer (windowed)
    - **Role:** Maintains recent conversation context (last 10 messages) per user session.
    - **Configuration:** Uses Telegram user ID as session key; context window set to 10 messages.
    - **Connections:** Input from Telegram Trigger for session key; output to AI Agent (ai_memory).
    - **Edge Cases:** Losing context on restarts; large conversations exceeding context window.
    - **Version:** 1.3

#### 2.3 AI Tools Integration

- **Overview:** Supplies the AI agent with external data sources and actions: calendar events, email sending, contact lookup, web search, and Wikipedia lookup.
- **Nodes Involved:** Google Calendar, Get Calendar Events, Gmail Tool, Get Contacts, Wikipedia Tool, Web Search (SerpAPI)
- **Node Details:**

  - **Google Calendar**
    - **Type:** Google Calendar Tool node
    - **Role:** Manage calendar events, e.g., creating or updating entries.
    - **Configuration:** Calendar ID set to user email (placeholder "YOUR_EMAIL@gmail.com"); start/end times and event details dynamically injected from AI agent overrides.
    - **Connections:** Input from AI Agent as ai_tool.
    - **Edge Cases:** OAuth token expiry; calendar ID misconfiguration; event creation conflicts.
    - **Version:** 1.3

  - **Get Calendar Events**
    - **Type:** Google Calendar Tool node
    - **Role:** Retrieve existing calendar events for daily schedule summaries.
    - **Configuration:** Calendar ID set to user email (placeholder); operation set to "getAll" to fetch all events.
    - **Connections:** Input from AI Agent as ai_tool.
    - **Edge Cases:** Large number of events causing timeouts; missing calendar permissions.
    - **Version:** 1.3

  - **Gmail Tool**
    - **Type:** Gmail Tool node
    - **Role:** Send emails on behalf of user.
    - **Configuration:** Email recipient, subject, message content dynamically set via AI agent overrides; sender name taken from Telegram user first name; plain text email type; OAuth2 required with send permissions.
    - **Connections:** Input from AI Agent as ai_tool.
    - **Edge Cases:** Missing Gmail OAuth2 tokens; send quota limits; invalid email addresses.
    - **Version:** 2.1

  - **Get Contacts**
    - **Type:** Google Sheets Tool node
    - **Role:** Access and manage contact information stored in Google Sheets.
    - **Configuration:** Document ID placeholder "YOUR_GOOGLE_SHEET_ID"; accesses sheet named "Sheet1" (gid=0); read/write enabled.
    - **Connections:** Input from AI Agent as ai_tool.
    - **Edge Cases:** Incorrect sheet ID; OAuth token expiry; sheet access permissions.
    - **Version:** 4.5

  - **Wikipedia Tool**
    - **Type:** LangChain Wikipedia Tool node
    - **Role:** Provides Wikipedia search results to AI agent.
    - **Configuration:** Default, no special parameters.
    - **Connections:** Input from AI Agent as ai_tool.
    - **Edge Cases:** Wikipedia API rate limiting; ambiguous search terms.
    - **Version:** 1

  - **Web Search (SerpAPI)**
    - **Type:** LangChain SerpAPI Tool node
    - **Role:** Enables web search functionality via SerpAPI.
    - **Configuration:** Requires SerpAPI account and API key; options default.
    - **Connections:** Input from AI Agent as ai_tool.
    - **Edge Cases:** API key invalid or quota exceeded; network issues.
    - **Version:** 1

#### 2.4 Output Delivery

- **Overview:** Sends AI-generated text responses back to the originating Telegram chat.
- **Nodes Involved:** Telegram Response
- **Node Details:**

  - **Telegram Response**
    - **Type:** Telegram node (send message)
    - **Role:** Delivers text output from AI Agent back to Telegram user.
    - **Configuration:** Message text set from AI Agent output; chat ID set dynamically from Telegram Trigger message chat ID; disables attribution.
    - **Connections:** Input from AI Agent main output.
    - **Edge Cases:** Telegram API downtime; invalid chat ID (e.g., user blocked bot).
    - **Version:** 1.2

---

### 3. Summary Table

| Node Name          | Node Type                            | Functional Role                     | Input Node(s)         | Output Node(s)      | Sticky Note                                                                                                  |
|--------------------|------------------------------------|-----------------------------------|-----------------------|---------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger    | n8n-nodes-base.telegramTrigger     | Receive Telegram messages          | -                     | AI Agent            | Receives messages from your Telegram bot. Replace credentials with your bot token from @BotFather           |
| AI Agent           | @n8n/n8n-nodes-langchain.agent     | Central AI processing and orchestration | Telegram Trigger, Gmail Tool, Get Contacts, Wikipedia Tool, Google Calendar, Get Calendar Events, Web Search (SerpAPI), Conversation Memory, OpenAI Chat Model | Telegram Response   | Main AI agent using OpenAI. Customize the system message for your needs.                                     |
| Telegram Response   | n8n-nodes-base.telegram             | Sends AI responses to Telegram    | AI Agent              | -                   | Sends AI agent responses back to Telegram chat                                                              |
| Wikipedia Tool      | @n8n/n8n-nodes-langchain.toolWikipedia | Wikipedia search capabilities     | AI Agent (ai_tool)     | AI Agent            | Provides Wikipedia search capabilities to the AI agent                                                      |
| Web Search (SerpAPI)| @n8n/n8n-nodes-langchain.toolSerpApi | Enables web search functionality  | AI Agent (ai_tool)     | AI Agent            | Enables web search functionality. Requires SerpAPI account and API key.                                     |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | AI language model backend         | AI Agent (ai_languageModel) | AI Agent         | AI language model. Change to gpt-4 or other models as needed. Requires OpenAI API key.                       |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation context    | Telegram Trigger       | AI Agent (ai_memory) | Maintains conversation context. Adjust contextWindowLength as needed (default: 10 messages)                  |
| Google Calendar     | n8n-nodes-base.googleCalendarTool  | Manage calendar events            | AI Agent (ai_tool)     | AI Agent            | Manages calendar events. Replace YOUR_EMAIL@gmail.com with your calendar ID. Requires Google Calendar OAuth2 setup. |
| Gmail Tool          | n8n-nodes-base.gmailTool            | Send emails                      | AI Agent (ai_tool)     | AI Agent            | Sends emails via Gmail. Requires Gmail OAuth2 setup with send permissions.                                   |
| Get Contacts        | n8n-nodes-base.googleSheetsTool    | Access contact database           | AI Agent (ai_tool)     | AI Agent            | Accesses contact database in Google Sheets. Replace YOUR_GOOGLE_SHEET_ID with your sheet ID. Requires OAuth2.|
| Get Calendar Events | n8n-nodes-base.googleCalendarTool  | Retrieve calendar events          | AI Agent (ai_tool)     | AI Agent            | Retrieves calendar events for daily schedule emails. Replace YOUR_EMAIL@gmail.com with your calendar ID.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**
   - Type: Telegram Trigger
   - Configure to listen for "message" updates.
   - Add Telegram Bot credentials (token from @BotFather).
   - Position as starting node.

2. **Create AI Agent node:**
   - Type: LangChain AI Agent
   - Set input text to `{{$json.message.text}}` from Telegram Trigger.
   - Customize the system message with your personalized instructions and constraints (e.g., user name, daily schedule handling).
   - Connect Telegram Trigger main output to AI Agent main input.

3. **Add OpenAI Chat Model node:**
   - Type: LangChain OpenAI Chat Model
   - Select model "gpt-4o-mini" or preferred GPT-4 variant.
   - Add OpenAI API key credentials.
   - Connect AI Agent `ai_languageModel` input to OpenAI Chat Model output.

4. **Add Conversation Memory node:**
   - Type: LangChain Memory Buffer Window
   - Set sessionKey to `{{$json.message.from.id}}` from Telegram Trigger.
   - Set contextWindowLength to 10.
   - Connect Telegram Trigger output to Conversation Memory input.
   - Connect Conversation Memory output to AI Agent `ai_memory` input.

5. **Add Google Calendar node (for event management):**
   - Type: Google Calendar Tool
   - Set calendar ID to your email or calendar ID.
   - Configure start, end, summary, description parameters with overrides from AI agent.
   - Connect AI Agent `ai_tool` input to Google Calendar node input.
   - Add Google Calendar OAuth2 credentials.

6. **Add Get Calendar Events node:**
   - Type: Google Calendar Tool
   - Set calendar ID to your email or calendar ID.
   - Set operation to "getAll".
   - Connect AI Agent `ai_tool` input to this node.
   - Add Google Calendar OAuth2 credentials.

7. **Add Gmail Tool node:**
   - Type: Gmail Tool
   - Configure sendTo, subject, message from AI agent overrides.
   - Set senderName dynamically from Telegram user first name.
   - Set email type to "text".
   - Connect AI Agent `ai_tool` input to Gmail Tool input.
   - Add Gmail OAuth2 credentials with send permissions.

8. **Add Get Contacts node:**
   - Type: Google Sheets Tool
   - Set document ID to your Google Sheet ID containing contacts.
   - Set sheet name to "Sheet1" or your sheet name.
   - Configure for read/write access.
   - Connect AI Agent `ai_tool` input to Get Contacts node input.
   - Add Google Sheets OAuth2 credentials.

9. **Add Wikipedia Tool node:**
   - Type: LangChain Wikipedia Tool
   - No special configuration needed.
   - Connect AI Agent `ai_tool` input to Wikipedia Tool input.

10. **Add Web Search (SerpAPI) node:**
    - Type: LangChain SerpAPI Tool
    - Provide SerpAPI API key credentials.
    - Connect AI Agent `ai_tool` input to Web Search node input.

11. **Add Telegram Response node:**
    - Type: Telegram (send message)
    - Set text to AI Agent output `{{$json.output}}`.
    - Set chatId dynamically from Telegram Trigger message chat id.
    - Disable append attribution.
    - Connect AI Agent main output to Telegram Response input.

12. **Verify all connections:**
    - Telegram Trigger → AI Agent (main)
    - AI Agent → Telegram Response (main)
    - AI Agent → OpenAI Chat Model (ai_languageModel)
    - AI Agent → Conversation Memory (ai_memory)
    - AI Agent → Gmail Tool, Get Contacts, Google Calendar, Get Calendar Events, Wikipedia Tool, Web Search (ai_tool)

13. **Set Workflow Settings:**
    - Timezone: America/Chicago (or your timezone)
    - Execution order: v1 (default)
    - Caller policy: workflowsFromSameOwner

14. **Activate workflow and test:**
    - Send message to Telegram bot.
    - Validate AI responses and integration with calendar, email, contacts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Telegram bot token must be obtained from @BotFather and configured in Telegram Trigger credentials.                                                                            | Telegram Bot setup                                                                                  |
| Google Calendar, Gmail, and Google Sheets nodes require OAuth2 credentials with appropriate scopes (calendar read/write, email send, sheets read/write).                      | Google Cloud Console API & OAuth2 setup                                                           |
| SerpAPI requires account registration and API key for web search functionality.                                                                                                | https://serpapi.com/                                                                               |
| The system message in AI Agent node is customizable to tailor AI assistant responses and behaviors, including naming the user and formatting email/calendar outputs.          | Modify system message parameter in AI Agent node                                                  |
| Conversation memory is essential for maintaining context across messages; adjust window length based on expected conversation complexity and API token limits.                | AI conversation design best practices                                                             |
| Replace placeholders such as YOUR_EMAIL@gmail.com and YOUR_GOOGLE_SHEET_ID with actual values for live deployment.                                                             | Configuration step                                                                                |
| The AI agent uses multiple tools via LangChain integrations to enrich responses beyond language generation, including external data fetching and actions.                     | https://docs.n8n.io/nodes/agents/                                                                 |
| Workflow designed for personal productivity assistants using Telegram as interface and Google services for backend data management.                                          | Use case: personal assistant, calendar/email management                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.

---