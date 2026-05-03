Personal Finance Assistant: Query Notion Data via Telegram & WhatsApp with Claude AI

https://n8nworkflows.xyz/workflows/personal-finance-assistant--query-notion-data-via-telegram---whatsapp-with-claude-ai-7528


# Personal Finance Assistant: Query Notion Data via Telegram & WhatsApp with Claude AI

### 1. Workflow Overview

This workflow is designed as a **Personal Finance Assistant** that enables users to query their personal finance data stored in Notion databases via **Telegram** and **WhatsApp** messaging platforms. It leverages an AI-powered **Accountant Agent** (using Claude AI via OpenRouter) to interpret natural language queries, dynamically interact with Notion databases, and return summarized financial data such as expenses, income, budgets, and assets.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures user queries sent over Telegram or WhatsApp.
- **1.2 AI Processing & Notion Data Query:** The Accountant Agent interprets queries, uses memory for context, accesses Notion databases via three specialized tools to fetch schema and data, and composes a response.
- **1.3 Response Delivery:** Sends the summarized answer back to the user on the originating platform (Telegram or WhatsApp).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives incoming user messages from Telegram and serves as the entry point for queries.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger node for Telegram messages  
  - *Configuration:* Listens for new messages (`updates` set to `"message"`). Webhook configured for incoming updates.  
  - *Key expressions:* None beyond default message capture.  
  - *Input Connections:* None (trigger)  
  - *Output Connections:* Connects to Accountant Agent node.  
  - *Edge cases:* Possible failure if webhook is not properly configured, invalid bot token, or permissions denied.  
  - *Credentials:* Uses Telegram API credentials ("Accountant AI").  

---

#### 1.2 AI Processing & Notion Data Query

**Overview:**  
This core block interprets the user's query using an AI Agent, accesses Notion databases with three dedicated tools, and maintains conversational context with memory.

**Nodes Involved:**  
- Accountant Agent  
- Simple Memory  
- claude-3.5-sonnet (Language Model)  
- Get Databases Tool  
- Get Notion Database Properties Tool  
- Query Notion Database Pages Tool  

**Node Details:**

- **Accountant Agent**  
  - *Type:* Langchain AI Agent node  
  - *Role:* Central AI logic interpreting queries and orchestrating Notion API calls.  
  - *Configuration:*  
    - Input text includes the user message and current date.  
    - System message defines the agent’s role as an accountant with access to three Notion tools.  
    - Instructions specify how to select databases, create filters, and summarize data.  
    - Returns intermediate steps for debugging/traceability.  
  - *Expressions:*  
    - `=Query: {{ $json.message.text}}`  
    - `Current Date {{ $now }}`  
  - *Input:* From Telegram Trigger and Simple Memory.  
  - *Outputs:* Connects to Send a text message (Telegram) and Send message (WhatsApp) nodes.  
  - *Edge cases:* AI model timeout, misunderstanding user input, Notion API limit reached, incorrect filter creation.  
  - *Integrations:* Uses three Notion tools and Simple Memory as AI tool & memory inputs.

- **Simple Memory**  
  - *Type:* Langchain memory buffer window  
  - *Role:* Maintains conversation context to provide continuity.  
  - *Configuration:*  
    - Session key "1" with custom session ID type.  
    - Context window length of 10 messages.  
  - *Input:* Feeds memory of previous interaction to Accountant Agent.  
  - *Output:* Connected to Accountant Agent’s memory input.  
  - *Edge cases:* Memory overflow or loss if session keys are inconsistent.

- **claude-3.5-sonnet**  
  - *Type:* Langchain language model node (Anthropic Claude 3.5 Sonnet)  
  - *Role:* Provides AI language model responses for the Accountant Agent.  
  - *Configuration:* Uses OpenRouter API with credentials.  
  - *Input:* Connected from Accountant Agent as language model input.  
  - *Output:* Returns generated text to Accountant Agent.  
  - *Edge cases:* API rate limits, authentication failures.

- **Get Databases Tool**  
  - *Type:* Notion Tool node  
  - *Role:* Retrieves list of all Notion databases accessible by the integration.  
  - *Configuration:*  
    - Resource: database  
    - Operation: getAll (return all databases)  
  - *Input:* Used as AI tool input by Accountant Agent.  
  - *Edge cases:* Notion API failures, permission errors.

- **Get Notion Database Properties Tool**  
  - *Type:* HTTP Request node (custom Notion API call)  
  - *Role:* Retrieves schema (properties) of a selected Notion database by ID.  
  - *Configuration:*  
    - GET request to `https://api.notion.com/v1/databases/{database_id}`  
    - Uses Notion API credentials.  
  - *Input:* Invoked by Accountant Agent as tool.  
  - *Edge cases:* Invalid database ID, permission denied, API limit.

- **Query Notion Database Pages Tool**  
  - *Type:* HTTP Request node  
  - *Role:* Queries rows from a Notion database with filters built from user input.  
  - *Configuration:*  
    - POST request to `https://api.notion.com/v1/databases/{database_id}/query`  
    - JSON body contains filters constructed dynamically by the AI agent.  
    - Uses Notion API credentials.  
  - *Input:* Invoked by Accountant Agent with constructed filter JSON.  
  - *Edge cases:* Incorrect filter format, large result sets, API throttling.

---

#### 1.3 Response Delivery

**Overview:**  
This block sends the AI-generated answer back to the user on the platform they queried from: Telegram or WhatsApp.

**Nodes Involved:**  
- Send a text message (Telegram)  
- Send message (WhatsApp)  

**Node Details:**

- **Send a text message**  
  - *Type:* Telegram node  
  - *Role:* Sends the response text back to the Telegram chat.  
  - *Configuration:*  
    - Text body mapped from `$json.output` (AI response).  
    - `chatId` set dynamically from incoming Telegram message chat ID.  
  - *Input:* From Accountant Agent output.  
  - *Edge cases:* Message send failures due to invalid chat ID, rate limits, or network issues.  
  - *Credentials:* Uses Telegram API credentials "Accountant AI".

- **Send message**  
  - *Type:* WhatsApp node  
  - *Role:* Sends the response text to WhatsApp user.  
  - *Configuration:*  
    - Text body mapped from `$json.output`.  
    - Phone number ID and recipient phone number configured via credentials and expression.  
  - *Input:* From Accountant Agent output.  
  - *Edge cases:* Token expiration, invalid phone number, WhatsApp API errors.  
  - *Credentials:* WhatsApp API credentials "Accountant AI".

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                               | Input Node(s)           | Output Node(s)                    | Sticky Note                                                                                                           |
|--------------------------------|--------------------------------------|-----------------------------------------------|-------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger               | telegramTrigger                      | Receive user queries from Telegram             | None                    | Accountant Agent                 | ## Step 1 - Trigger                                                                                                   |
| Accountant Agent              | langchain.agent                      | AI Agent processing query and orchestrating Notion calls | Telegram Trigger, Simple Memory, claude-3.5-sonnet, Get Databases Tool, Get Notion Database Properties Tool, Query Notion Database Pages Tool | Send a text message, Send message | ## Step 2 - Agent                                                                                                     |
| Simple Memory                 | langchain.memoryBufferWindow         | Maintains conversation context                  | Accountant Agent (ai_memory) | Accountant Agent (ai_memory)    |                                                                                                                       |
| claude-3.5-sonnet             | langchain.lmChatOpenRouter           | Language model providing AI responses           | Accountant Agent (ai_languageModel) | Accountant Agent (ai_languageModel) |                                                                                                                       |
| Get Databases Tool            | notionTool                          | Fetch all Notion databases                       | Accountant Agent (ai_tool) | Accountant Agent (ai_tool)       |                                                                                                                       |
| Get Notion Database Properties Tool | httpRequestTool                   | Retrieve schema of selected Notion database     | Accountant Agent (ai_tool) | Accountant Agent (ai_tool)       |                                                                                                                       |
| Query Notion Database Pages Tool | httpRequestTool                   | Query rows from Notion database with filters    | Accountant Agent (ai_tool) | Accountant Agent (ai_tool)       |                                                                                                                       |
| Send a text message           | telegram                            | Send response back to Telegram user             | Accountant Agent         | None                            | ## Step 3 - Send Message                                                                                               |
| Send message                 | whatsApp                            | Send response back to WhatsApp user              | Accountant Agent         | None                            | ## Step 3 - Send Message                                                                                               |
| Sticky Note                  | stickyNote                         | Documentation note on Agent step                 | None                    | None                            | ## Step 2 - Agent                                                                                                     |
| Sticky Note1                 | stickyNote                         | Documentation note on Trigger step               | None                    | None                            | ## Step 1 - Trigger                                                                                                   |
| Sticky Note2                 | stickyNote                         | Documentation note on Send Message step          | None                    | None                            | ## Step 3 - Send Message                                                                                               |
| Sticky Note3                 | stickyNote                         | Detailed workflow overview and explanation       | None                    | None                            | ## 📝 Query personal finance data in Notion via Telegram and WhatsApp (full descriptive note)                         |
| Sticky Note4                 | stickyNote                         | Setup instructions and credentials guide         | None                    | None                            | Detailed step-by-step setup instructions for Telegram, OpenRouter, Notion, WhatsApp API credentials and security notes |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot Token credentials.  
   - Set to listen to `message` updates.  
   - Position: Start of workflow.

2. **Create Simple Memory node:**  
   - Type: Langchain Memory Buffer Window  
   - Session Key: `"1"`  
   - Session ID Type: `customKey`  
   - Context Window Length: `10`  
   - Connect output of Telegram Trigger to input of Simple Memory.

3. **Create claude-3.5-sonnet node (Language Model):**  
   - Type: Langchain LM Chat Open Router  
   - Model: `anthropic/claude-3.5-sonnet`  
   - Credentials: OpenRouter API key  
   - Connect output of Simple Memory to this node as memory input.

4. **Create Get Databases Tool node:**  
   - Type: Notion Tool  
   - Resource: `database`  
   - Operation: `getAll` (returnAll = true)  
   - Credentials: Notion API token with access to finance workspace.

5. **Create Get Notion Database Properties Tool node:**  
   - Type: HTTP Request node  
   - Method: `GET`  
   - URL Template: `https://api.notion.com/v1/databases/{database_id}` (to be dynamically provided)  
   - Authentication: Notion credentials  
   - Set headers: Authorization, Notion-Version `2022-06-28`, Content-Type `application/json`.

6. **Create Query Notion Database Pages Tool node:**  
   - Type: HTTP Request node  
   - Method: `POST`  
   - URL Template: `https://api.notion.com/v1/databases/{database_id}/query`  
   - Body Type: JSON  
   - Body: dynamically created filter JSON from AI agent  
   - Authentication: Notion credentials.

7. **Create Accountant Agent node:**  
   - Type: Langchain Agent  
   - Inputs:  
     - Text: `Query: {{ $json.message.text}} Current Date {{ $now }}`  
     - System message: Instructions describing role, tools, and usage as in the overview.  
   - Tools inputs linked:  
     - Get Databases Tool  
     - Get Notion Database Properties Tool  
     - Query Notion Database Pages Tool  
   - Memory input: Simple Memory output  
   - Language model input: claude-3.5-sonnet output  
   - Connect Telegram Trigger main output to Accountant Agent main input.  

8. **Create Send a text message node (Telegram):**  
   - Type: Telegram (Send Message)  
   - Text: `={{ $json.output }}` (response from Accountant Agent)  
   - chatId: `={{ $json.message.chat.id }}` (dynamic)  
   - Credentials: Telegram API  
   - Connect Accountant Agent main output to this node.

9. **Create Send message node (WhatsApp):**  
   - Type: WhatsApp  
   - Operation: `send`  
   - Text body: `={{ $json.output }}`  
   - phoneNumberId: your WhatsApp Business phone number ID  
   - recipientPhoneNumber: dynamically passed or set  
   - Credentials: WhatsApp API  
   - Connect Accountant Agent main output to this node.

10. **Link all nodes according to the structure:**  
    - Telegram Trigger → Accountant Agent  
    - Simple Memory → Accountant Agent (memory input)  
    - claude-3.5-sonnet → Accountant Agent (language model input)  
    - Get Databases Tool → Accountant Agent (tool input)  
    - Get Notion Database Properties Tool → Accountant Agent (tool input)  
    - Query Notion Database Pages Tool → Accountant Agent (tool input)  
    - Accountant Agent → Send a text message (Telegram) & Send message (WhatsApp)

11. **Credential Setup:**  
    - Telegram: Bot token with appropriate chat permission  
    - OpenRouter: API key for Claude 3.5-sonnet model  
    - Notion: Integration token with read access to personal finance workspace  
    - WhatsApp: Phone number ID, WABA account, and permanent access token

12. **Testing:**  
    - Send a test message to Telegram bot (e.g., “Total expenses for August 2025”)  
    - Confirm responses are returned correctly on Telegram and WhatsApp (if enabled).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow uses three Notion databases as sources: Financial Transactions, Budget, Income, and Assets & Liabilities. The AI agent dynamically selects and queries the appropriate database based on user query context. | Workflow design principle                                                                          |
| For Notion API calls, use Notion-Version header `2022-06-28` and ensure integration token has appropriate read access to all finance-related databases.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Notion API versioning and access requirements                                                     |
| Telegram Bot setup steps include creating a bot with BotFather, obtaining bot token, and getting chat ID either via sending messages or API calls (`getUpdates`).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Telegram Bot setup instructions                                                                   |
| OpenRouter API key is required for the Claude 3.5 Sonnet model used in this workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | OpenRouter API setup                                                                              |
| WhatsApp Business API requires Facebook Business Manager account, WhatsApp Business Account, phone number ID, and permanent access tokens to send messages programmatically. | WhatsApp Business API setup instructions                                                         |
| Security best practices: never commit tokens to source control, use environment variables or n8n credentials, rotate tokens regularly, and restrict access to Telegram bot by chat IDs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Security best practices                                                                            |
| Recommended credential names in n8n: `Accountant AI` for Telegram and WhatsApp, `OpenRouter account` for AI, and `Notion account` for Notion API to maintain clarity and avoid confusion when wiring nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Credential naming conventions                                                                     |
| Helpful external links:  - [Personal Finance System Notion template](https://www.notion.so/templates/personal-finance-system) - Telegram BotFather documentation - OpenRouter platform docs - Facebook Developer WhatsApp API docs | Useful resources for setup and customization                                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow solution. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.