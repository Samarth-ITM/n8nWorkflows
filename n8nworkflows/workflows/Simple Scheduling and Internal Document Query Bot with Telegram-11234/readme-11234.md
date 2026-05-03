Simple Scheduling and Internal Document Query Bot with Telegram

https://n8nworkflows.xyz/workflows/simple-scheduling-and-internal-document-query-bot-with-telegram-11234


# Simple Scheduling and Internal Document Query Bot with Telegram

### 1. Workflow Overview

This workflow implements a **Simple Scheduling and Internal Document Query Bot** integrated with **Telegram**. It serves as an AI-driven assistant for an internal IT shop, enabling users to:

- Schedule maintenance or services via Google Calendar
- Check prices of services and products from Google Sheets
- Retrieve company information and troubleshooting guidance from Google Docs and Sheets

The workflow is structured into three main logical blocks:

**1.1 Message Reception, Processing, and Output**  
- Captures user input through Telegram  
- Uses an AI Agent with memory (Redis) to interpret intent and maintain context  
- Routes requests to appropriate MCP Clients for external service integration  
- Sends back clear, contextual text responses to users  

**1.2 Scheduling Management (Google Calendar via MCP)**  
- Handles creation, update, deletion, and querying of calendar events  
- Uses Google Calendar through MCP Client nodes for all scheduling operations  

**1.3 Internal Document Queries (Google Docs & Google Sheets via MCP)**  
- Retrieves service pricing from Google Sheets  
- Provides company information from Google Docs  
- Offers troubleshooting steps from a dedicated Google Sheets document  

These blocks work together to provide an integrated conversational experience where the AI Agent decides the appropriate action and tool to use, based on the user's intent.

---

### 2. Block-by-Block Analysis

#### 2.1 Message Reception, Processing, and Output

**Overview:**  
This block captures user messages from Telegram, processes them with an AI Agent that maintains conversation context using Redis memory, and finally sends a text response back through Telegram.

**Nodes Involved:**  
- Telegram Trigger  
- AI Agent  
- Redis Chat Memory  
- Date & Time  
- Google Gemini Chat Model  
- Send a text message  

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Trigger node for Telegram incoming messages  
  - *Configuration:* Listens for "message" updates, uses Telegram API credentials  
  - *Input/Output:* Starts the flow, outputs message JSON with chat id and text  
  - *Failure Modes:* Telegram API connection issues, webhook misconfiguration  
  - *Credentials:* Telegram API OAuth token required  

- **AI Agent**  
  - *Type:* Langchain AI Agent node  
  - *Configuration:*  
    - Input text: user message text from Telegram  
    - System message: Defines role as IT shop AI agent with detailed instructions and context  
    - Uses Google Gemini Chat Model as language model  
    - Calls MCP Clients for Google Calendar, Docs, Sheets tools  
    - Incorporates Date & Time tool for relative date processing  
  - *Expressions:* Reads `$json.message.text` for input, outputs response text to be sent  
  - *Inputs:* Telegram Trigger (main), Redis Chat Memory (ai_memory), Date & Time (ai_tool), MCP Clients (ai_tool), Google Gemini Chat Model (ai_languageModel)  
  - *Outputs:* Connects to Telegram Send node for reply  
  - *Failure Modes:* AI model API errors, expression evaluation errors, MCP endpoint unavailability  

- **Redis Chat Memory**  
  - *Type:* Langchain Redis memory node  
  - *Configuration:*  
    - Session Key: chat id string from Telegram message to maintain per-user context  
    - Context window length: 10 messages for conversation history  
  - *Inputs:* feeds into AI Agent for memory context  
  - *Failure Modes:* Redis connection or authentication failure, session key errors  
  - *Credentials:* Redis account with Upstash or compatible Redis instance  

- **Date & Time**  
  - *Type:* Date/time utility node  
  - *Configuration:* Outputs current date/time `$now` in ISO format  
  - *Usage:* Used by AI Agent to interpret relative dates like “tomorrow” or “next week”  
  - *Failure Modes:* N/A (local date/time)  

- **Google Gemini Chat Model**  
  - *Type:* Language model node using Google PaLM API  
  - *Configuration:* Uses Google Palm API credentials  
  - *Purpose:* Provides AI chat completions for natural language understanding and generation  
  - *Failure Modes:* API limits, authentication errors  

- **Send a text message**  
  - *Type:* Telegram node for sending messages  
  - *Configuration:* Sends text output from AI Agent back to user chat  
  - *Expressions:* Uses chat id from Telegram Trigger to address user  
  - *Failure Modes:* Telegram API down, invalid chat id  

---

#### 2.2 Scheduling (Google Calendar via MCP)

**Overview:**  
Handles scheduling intents detected by the AI Agent. Supports creating, retrieving, updating, and deleting Google Calendar events via MCP clients.

**Nodes Involved:**  
- MCP Client Google Calendario  
- Create an event in Google Calendar  
- Get many events in Google Calendar  
- Delete an event in Google Calendar  
- Update an event in Google Calendar (disabled)  

**Node Details:**

- **MCP Client Google Calendario**  
  - *Type:* MCP Client tool to interface with Google Calendar services  
  - *Configuration:* Connects to a specific MCP endpoint URL managing calendar operations  
  - *Input/Output:* Receives commands from AI Agent, calls Google Calendar actions  
  - *Failure Modes:* MCP server unavailability, network errors  

- **Create an event in Google Calendar**  
  - *Type:* Google Calendar Tool node  
  - *Operation:* Create event  
  - *Parameters:* Takes event start/end times, summary, description from AI Agent outputs via expressions  
  - *Credentials:* Google Calendar OAuth2 account  
  - *Failure Modes:* Invalid date formats, API rate limits, permission errors  

- **Get many events in Google Calendar**  
  - *Type:* Google Calendar Tool node  
  - *Operation:* Get all events between timeMin and timeMax, derived from AI Agent input (dates converted to ISO)  
  - *Credentials:* Google Calendar OAuth2 account  
  - *Failure Modes:* API errors, invalid date range requests  

- **Delete an event in Google Calendar**  
  - *Type:* Google Calendar Tool node  
  - *Operation:* Delete event by Event ID specified by AI Agent  
  - *Credentials:* Google Calendar OAuth2 account  
  - *Failure Modes:* Event not found, permission denied  

- **Update an event in Google Calendar** (disabled)  
  - *Type:* Google Calendar Tool node  
  - *Operation:* Update event fields  
  - *Note:* Currently disabled, possibly reserved for future enhancements  

---

#### 2.3 Internal Queries (Google Docs & Google Sheets via MCP)

**Overview:**  
Responds to queries about prices, company information, and troubleshooting by querying Google Sheets and Docs through MCP Clients.

**Nodes Involved:**  
- MCP Google Docs e Sheet (MCP Trigger)  
- Valores dos Serviços (Google Sheets Tool)  
- Guia da Loja de TI (Google Docs Tool)  
- Troubleshooting - Lista de Problemas e Soluções (Google Sheets Tool)  
- MCP Client Google Docs e Sheet  

**Node Details:**

- **MCP Google Docs e Sheet**  
  - *Type:* MCP Trigger node  
  - *Purpose:* Receives AI Agent calls to query Google Docs and Sheets documents  
  - *Configuration:* Webhook path for MCP communication  
  - *Failure Modes:* MCP server down, webhook misconfiguration  

- **Valores dos Serviços**  
  - *Type:* Google Sheets Tool node  
  - *Document:* Spreadsheet with service prices  
  - *Sheet:* "Sheet1"  
  - *Credentials:* Google Sheets OAuth2 account  
  - *Usage:* Returns pricing data for service-related queries  
  - *Failure Modes:* Sheet not accessible, invalid credentials  

- **Guia da Loja de TI**  
  - *Type:* Google Docs Tool node  
  - *Document:* Institutional company guide document URL  
  - *Operation:* Get document content  
  - *Credentials:* Google Docs OAuth2 account  
  - *Usage:* Answers company info questions (history, mission, service area)  
  - *Failure Modes:* Document access denied, API errors  

- **Troubleshooting - Lista de Problemas e Soluções**  
  - *Type:* Google Sheets Tool node  
  - *Document:* Spreadsheet with troubleshooting problems and solutions  
  - *Sheet:* "Sheet1"  
  - *Credentials:* Google Sheets OAuth2 account  
  - *Usage:* Provides diagnostic steps for technical issues  
  - *Failure Modes:* API errors, document missing  

- **MCP Client Google Docs**  
  - *Type:* MCP Client tool  
  - *Purpose:* Used by AI Agent to query Google Docs via MCP  
  - *Failure Modes:* MCP server failure  

---

### 3. Summary Table

| Node Name                            | Node Type                         | Functional Role                            | Input Node(s)                     | Output Node(s)               | Sticky Note                                                                                                                         |
|------------------------------------|----------------------------------|-------------------------------------------|----------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger                   | Telegram Trigger                 | Capture user messages from Telegram       | -                                | AI Agent                    | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| AI Agent                          | Langchain AI Agent               | Interpret user intent, control flow       | Telegram Trigger, Redis Chat Memory, Date & Time, MCP Clients, Google Gemini Chat Model | Send a text message          | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| Redis Chat Memory                 | Redis Memory                    | Store conversation context for AI Agent   | -                                | AI Agent                    | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| Date & Time                      | Date/time Tool                  | Provide current date/time for relative date processing | -                                | AI Agent                    | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| Google Gemini Chat Model           | Google PaLM Chat Model           | AI language model for chat completions    | -                                | AI Agent                    | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| Send a text message               | Telegram                        | Send response back to Telegram user       | AI Agent                        | -                           | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| MCP Client Google Calendario      | MCP Client Tool                 | Interface to Google Calendar via MCP      | AI Agent                        | Google Calendar nodes        | ## 2. Scheduling (Google Calendar via MCP)                                                                                        |
| Create an event in Google Calendar | Google Calendar Tool            | Create calendar events                     | MCP Google Calendario           | MCP Google Calendario        | ## 2. Scheduling (Google Calendar via MCP)                                                                                        |
| Get many events in Google Calendar | Google Calendar Tool            | Retrieve multiple calendar events          | MCP Google Calendario           | MCP Google Calendario        | ## 2. Scheduling (Google Calendar via MCP)                                                                                        |
| Delete an event in Google Calendar | Google Calendar Tool            | Delete calendar event                      | MCP Google Calendario           | MCP Google Calendario        | ## 2. Scheduling (Google Calendar via MCP)                                                                                        |
| Update an event in Google Calendar | Google Calendar Tool (disabled) | Update calendar events                     | MCP Google Calendario           | MCP Google Calendario        | ## 2. Scheduling (Google Calendar via MCP)                                                                                        |
| MCP Google Docs e Sheet           | MCP Trigger                    | Receive MCP queries for Docs/Sheets       | Guia da Loja de TI, Valores dos Serviços, Troubleshooting Sheet | AI Agent                    | ## 3. Internal Queries (Google Docs & Google Sheets via MCP)                                                                       |
| Valores dos Serviços              | Google Sheets Tool             | Provide pricing info from Sheets           | MCP Google Docs e Sheet         | MCP Google Docs e Sheet      | ## 3. Internal Queries (Google Docs & Google Sheets via MCP)                                                                       |
| Guia da Loja de TI               | Google Docs Tool               | Provide company info from Docs              | MCP Google Docs e Sheet         | MCP Google Docs e Sheet      | ## 3. Internal Queries (Google Docs & Google Sheets via MCP)                                                                       |
| Troubleshooting - Lista de Problemas e Soluções | Google Sheets Tool             | Provide troubleshooting info from Sheets  | MCP Google Docs e Sheet         | MCP Google Docs e Sheet      | ## 3. Internal Queries (Google Docs & Google Sheets via MCP)                                                                       |
| MCP Client Google Docs           | MCP Client Tool                | Interface to Google Docs via MCP            | AI Agent                      | AI Agent                    | ## 3. Internal Queries (Google Docs & Google Sheets via MCP)                                                                       |
| Sticky Note7                    | Sticky Note                   | General overview and usage instructions     | -                              | -                           | ## Try it ! This n8n template shows how to use AI with MCP to query internal IT shop documents, enabling scheduling, price checks, and company information in a single workflow.|
| Sticky Note6                    | Sticky Note                   | Block 1 description                          | -                              | -                           | ## 1. Message Reception, Processing, and Output (Telegram + AI Agent)                                                              |
| Sticky Note2                    | Sticky Note                   | Block 2 description                          | -                              | -                           | ## 2. Scheduling (Google Calendar via MCP)                                                                                        |
| Sticky Note                     | Sticky Note                   | Block 3 description                          | -                              | -                           | ## 3. Internal Queries (Google Docs & Google Sheets via MCP)                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure updates to listen for: "message"  
   - Set Telegram API credentials (OAuth2 token)  
   - Position: Entry node  

2. **Create Redis Chat Memory node**  
   - Type: Langchain Redis Memory  
   - Configure sessionKey as: `{{$json.message.chat.id.toString()}}`  
   - Context window length: 10  
   - Set Redis credentials (Upstash or compatible Redis)  

3. **Create Date & Time node**  
   - Type: Date/time Tool  
   - Configure to output current date/time as ISO string in field `{{$now}}`  

4. **Create Google Gemini Chat Model node**  
   - Type: Langchain Google Gemini Chat Model  
   - Set Google Palm API credentials  

5. **Create MCP Client nodes**  
   - MCP Client Google Calendario  
     - Type: MCP Client Tool  
     - Configure endpoint URL for Google Calendar MCP service  
   - MCP Client Google Docs  
     - Type: MCP Client Tool  
     - Configure endpoint URL for Google Docs MCP service  

6. **Create MCP Trigger node**  
   - MCP Google Docs e Sheet  
   - Configure webhook path (unique UUID) for MCP server communication  

7. **Create Google Sheets Tool nodes**  
   - Valores dos Serviços:  
     - Document ID: Google Sheets with service prices  
     - Sheet Name: "Sheet1"  
     - Credentials: Google Sheets OAuth2  
   - Troubleshooting - Lista de Problemas e Soluções:  
     - Document ID: Google Sheets with troubleshooting info  
     - Sheet Name: "Sheet1"  
     - Credentials: Google Sheets OAuth2  

8. **Create Google Docs Tool node**  
   - Guia da Loja de TI  
   - Operation: Get document content  
   - Document URL: Institutional company guide Google Docs URL  
   - Credentials: Google Docs OAuth2  

9. **Create AI Agent node**  
   - Type: Langchain AI Agent  
   - Input text: `{{$json.message.text}}` from Telegram Trigger  
   - System message: Define role, instructions, context, output format, and examples as per original content  
   - Connect AI language model input to Google Gemini Chat Model node  
   - Connect ai_memory input to Redis Chat Memory node  
   - Connect ai_tool inputs to MCP Client Google Calendario, MCP Client Google Docs, Date & Time node, and MCP Trigger node  
   - Configure the AI Agent to receive and send data with these connections  

10. **Create Google Calendar Tool nodes**  
    - Create an event in Google Calendar  
      - Operation: Create  
      - Parameters: Event start/end, summary, description from AI Agent outputs  
      - Credentials: Google Calendar OAuth2  
    - Get many events in Google Calendar  
      - Operation: Get all events between date ranges from AI Agent  
      - Credentials: Google Calendar OAuth2  
    - Delete an event in Google Calendar  
      - Operation: Delete event by ID from AI Agent  
      - Credentials: Google Calendar OAuth2  
    - (Optional) Update an event in Google Calendar (disabled)  

11. **Create Send a text message node**  
    - Type: Telegram  
    - Parameters: Text from AI Agent output  
    - Chat ID from Telegram Trigger message chat id  
    - Credentials: Telegram API  

12. **Connect nodes**  
    - Telegram Trigger → AI Agent (main)  
    - Redis Chat Memory → AI Agent (ai_memory)  
    - Date & Time → AI Agent (ai_tool)  
    - Google Gemini Chat Model → AI Agent (ai_languageModel)  
    - MCP Clients (Google Calendario, Google Docs) → AI Agent (ai_tool)  
    - MCP Google Docs e Sheet → AI Agent (ai_tool)  
    - AI Agent → Send a text message (main)  
    - AI Agent → Google Calendar Tool nodes (create/get/delete) via MCP Google Calendario node (ai_tool)  
    - Google Docs and Sheets nodes connected to MCP Google Docs e Sheet (ai_tool)  

13. **Credentials Setup**  
    - Telegram: Telegram Bot API token  
    - Redis: Upstash Redis credentials or compatible Redis instance  
    - Google Calendar, Docs, Sheets: OAuth2 credentials with appropriate scopes  
    - Google Palm API: API key or OAuth for Google Gemeni Chat Model  

14. **Testing**  
    - Test Telegram messages for scheduling, price queries, and info requests  
    - Verify AI Agent correctly routes queries and replies  
    - Monitor logs for errors or failed API calls  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates AI integration with MCP to query internal IT shop documents for scheduling, pricing, and company info through Telegram.                                                                                      | Sticky Note7 content                                                                                                                                                         |
| Telegram trigger is an example; other input/output channels like WhatsApp, Webhooks, or forms can replace it.                                                                                                                             | Sticky Note7 content                                                                                                                                                         |
| Requires Google account with access to Calendar, Docs, Sheets, MCP Server configured, Telegram integration, and Upstash Redis for memory.                                                                                                | Sticky Note7 content                                                                                                                                                         |
| Planned enhancements include segmented specialized agents, voice support, advanced RAG with vector databases, expanded document integration.                                                                                            | Sticky Note7 content                                                                                                                                                         |
| For help, join the n8n Discord or Forum communities.                                                                                                                                                                                     | Sticky Note7 content                                                                                                                                                         |
| Block 1 covers text input, AI processing, and output via Telegram; includes memory and AI models.                                                                                                                                         | Sticky Note6 content                                                                                                                                                         |
| Block 2 covers scheduling via Google Calendar using MCP clients.                                                                                                                                                                         | Sticky Note2 content                                                                                                                                                         |
| Block 3 handles internal document queries using Google Docs and Sheets via MCP clients.                                                                                                                                                   | Sticky Note content                                                                                                                                                          |
| For further information on usage of MCP clients and Langchain AI Agents in n8n, see official n8n documentation and Langchain integration guides.                                                                                        | n8n Docs and Langchain official docs (not linked here)                                                                                                                      |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow designed with strict adherence to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.