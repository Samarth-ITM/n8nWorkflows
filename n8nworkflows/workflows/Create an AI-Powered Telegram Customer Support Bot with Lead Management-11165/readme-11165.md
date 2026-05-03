Create an AI-Powered Telegram Customer Support Bot with Lead Management

https://n8nworkflows.xyz/workflows/create-an-ai-powered-telegram-customer-support-bot-with-lead-management-11165


# Create an AI-Powered Telegram Customer Support Bot with Lead Management

### 1. Workflow Overview

This workflow implements an AI-powered Telegram customer support assistant integrated with lead management. It is designed to handle incoming user messages on Telegram, maintain conversation logs, manage lead records in a database, and provide intelligent, context-aware responses using an AI agent. The workflow is suitable for businesses aiming to automate customer support, streamline lead capture, and deliver personalized replies.

The workflow’s logic is organized into the following main blocks:

- **1.1 Telegram Input & Message Logging:** Receives incoming Telegram messages and logs user messages.
- **1.2 Lead Management:** Checks for existing lead records; creates new leads if none exist.
- **1.3 Context Building:** Constructs a comprehensive context object combining Telegram user info, lead data, and the user message.
- **1.4 AI Processing & Response Generation:** Sends the context to an AI agent that interprets the message, manages lead updates, fetches relevant data (FAQ, services, settings), and generates responses.
- **1.5 Lead Update & Logging Bot Responses:** Updates lead records based on AI output and logs bot replies.
- **1.6 Telegram Output:** Sends the AI-generated response back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input & Message Logging

**Overview:**  
This block handles receiving messages from Telegram users via a webhook trigger and logs the incoming messages into a chat log database table for record-keeping.

**Nodes Involved:**  
- Telegram - Incoming Message  
- Log - User Message  

**Node Details:**  

- **Telegram - Incoming Message**  
  - Type: Telegram Trigger  
  - Role: Listens for new Telegram messages (update type: message).  
  - Configuration: Uses Telegram Bot API credentials; triggers on all message updates.  
  - Inputs: External Telegram webhook trigger.  
  - Outputs: JSON containing full Telegram message object.  
  - Possible failures: Network issues, invalid credentials, Telegram API downtime.  
  - No sub-workflow.

- **Log - User Message**  
  - Type: Data Table (for database logging)  
  - Role: Stores the user’s message in the `chat_logs` table with fields: `chat_id`, `direction` (user), and `message_text`.  
  - Configuration: Maps `chat_id` and `message_text` from Telegram message JSON.  
  - Inputs: Connected from Telegram - Incoming Message.  
  - Outputs: Passes data to the next node to fetch lead info.  
  - Possible failures: Database connection issues, schema mismatches.

---

#### 2.2 Lead Management

**Overview:**  
Checks if a lead record exists for the user’s Telegram chat ID. If none is found, creates a new lead entry with basic Telegram user info.

**Nodes Involved:**  
- DB - Get Lead by User ID  
- Check – Lead Record (If node)  
- DB - Create Lead  

**Node Details:**  

- **DB - Get Lead by User ID**  
  - Type: Data Table (database query)  
  - Role: Retrieves existing lead record matching the Telegram chat ID.  
  - Configuration: Filters by `chat_id` equal to Telegram user chat ID, returns 1 entry.  
  - Inputs: Connected from Log - User Message.  
  - Outputs: Lead record JSON or empty if none found.  
  - Failure modes: DB timeout, no record found (handled downstream).

- **Check – Lead Record**  
  - Type: If node  
  - Role: Checks if `chat_id` field in lead record is non-empty (i.e., lead exists).  
  - Configuration: Condition checks `chat_id` not empty.  
  - Inputs: From DB - Get Lead by User ID.  
  - Outputs:  
    - True: Lead exists → proceed to build context.  
    - False: Lead missing → create new lead.  
  - Edge cases: Empty or null fields.

- **DB - Create Lead**  
  - Type: Data Table  
  - Role: Creates a new lead record with basic Telegram user details (`chat_id`, first and last name, status “new”).  
  - Configuration: Fields mapped from Telegram message chat object.  
  - Inputs: False branch of Check – Lead Record.  
  - Outputs: Data passed to context builder.  
  - Failure modes: DB write errors, missing Telegram user info.

---

#### 2.3 Context Building

**Overview:**  
Constructs a clean, structured context object including Telegram user info, lead data (if any), and the user’s latest message text, preparing it for AI processing.

**Nodes Involved:**  
- Build Assistant Context  

**Node Details:**  

- **Build Assistant Context**  
  - Type: Code (JavaScript)  
  - Role: Extracts Telegram user fields, merges lead data if present, and includes user message text into a single JSON context object.  
  - Configuration:  
    - Reads Telegram chat info (`id`, `first_name`, `last_name`).  
    - Reads lead fields if found: `full_name`, `phone`, `email`, `status`, `notes`.  
    - Prepares object with keys: `telegram_user`, `lead_data`, `user_message`.  
  - Inputs: From DB - Create Lead or Check – Lead Record (true branch).  
  - Outputs: Provides context JSON to AI Agent.  
  - Edge cases: Missing lead data defaults to null; missing Telegram info handled gracefully.

---

#### 2.4 AI Processing & Response Generation

**Overview:**  
An AI Agent receives the context object, interprets user intents, decides on lead updates or information retrieval (services, FAQ, settings), and generates a contextual response.

**Nodes Involved:**  
- AI - Smart Support Assistant  
- OpenRouter Chat Model  
- Simple Memory  
- DB - Get FAQ  
- DB - Get Services  
- DB - Get Settings  

**Node Details:**  

- **AI - Smart Support Assistant**  
  - Type: Langchain Agent  
  - Role: Core AI logic node that processes user input with context, decides actions, and generates replies.  
  - Configuration:  
    - Input text is user message from context.  
    - System prompt defines detailed instructions for lead completion, tool usage, response style, and strict data privacy rules.  
    - Allows calling specific tools to get data (FAQ, services, settings) or update leads.  
  - Inputs: From Build Assistant Context.  
  - Outputs: AI-generated reply and data for lead updates.  
  - Failure modes: API errors, prompt parsing issues, unexpected AI output format.

- **OpenRouter Chat Model**  
  - Type: Language Model (OpenRouter)  
  - Role: Provides the underlying language model API for AI - Smart Support Assistant.  
  - Configuration: Temperature 0.2, max tokens 400, penalties set for frequency and presence, responseFormat as text.  
  - Inputs: Connected as AI language model for AI agent.  
  - Credentials: OpenRouter API key for “UmrahAssistantBot”.  
  - Failure modes: API key invalid, rate limits, timeout.

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversation context history using Telegram chat ID as session key, allowing context window of last 25 interactions.  
  - Inputs: AI agent memory input.  
  - Failure modes: Memory overflow, key misconfiguration.

- **DB - Get FAQ**  
  - Type: Data Table Tool  
  - Role: Retrieves all active FAQ entries to support AI answering questions.  
  - Configuration: Filter for active entries only, returns all matching records.  
  - Inputs: AI agent tool call.  
  - Failure modes: DB access issues.

- **DB - Get Services**  
  - Type: Data Table Tool  
  - Role: Gets all active services/programs data for AI to provide details on pricing and offers.  
  - Configuration: Filter active only, return all.  
  - Inputs: AI agent tool call.  
  - Failure modes: DB errors.

- **DB - Get Settings**  
  - Type: Data Table Tool  
  - Role: Retrieves company info such as contact details, hours, etc., used by AI for related inquiries.  
  - Inputs: AI agent tool call.  
  - Failure modes: DB access errors.

---

#### 2.5 Lead Update & Logging Bot Responses

**Overview:**  
Updates lead records with new data extracted by AI and logs bot-generated messages in the chat logs.

**Nodes Involved:**  
- DB - Update Lead  
- Log - Bot Message  

**Node Details:**  

- **DB - Update Lead**  
  - Type: Data Table Tool  
  - Role: Updates lead fields such as email, phone, full name, status, notes, etc., based on AI-extracted data.  
  - Configuration: Uses `chat_id` filter to update the correct lead record. Fields are dynamically populated from AI outputs.  
  - Inputs: AI tool output from AI agent.  
  - Failure modes: Partial data updates, DB write errors.

- **Log - Bot Message**  
  - Type: Data Table  
  - Role: Logs the bot’s reply in `chat_logs` with direction “bot” and message text from AI output.  
  - Inputs: From AI - Smart Support Assistant node.  
  - Outputs: Passes data to Telegram output node.  
  - Failure modes: DB logging errors.

---

#### 2.6 Telegram Output

**Overview:**  
Sends the AI-generated reply message back to the user on Telegram.

**Nodes Involved:**  
- Telegram - Send Response  

**Node Details:**  

- **Telegram - Send Response**  
  - Type: Telegram node  
  - Role: Sends text messages to Telegram users using bot credentials.  
  - Configuration:  
    - Message text taken from AI assistant output.  
    - Target chat ID matches original Telegram user chat ID.  
    - Attribution disabled to avoid additional bot metadata.  
  - Inputs: From Log - Bot Message.  
  - Failure modes: Telegram API errors, network issues.

---

### 3. Summary Table

| Node Name                | Node Type                            | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                    |
|--------------------------|------------------------------------|---------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------|
| Telegram - Incoming Message | Telegram Trigger                  | Receives Telegram user messages       | External webhook               | Log - User Message             | ## Telegram Input: Handles incoming Telegram messages and forwards them to the workflow. Also stores each user message in chat_logs for tracking. |
| Log - User Message        | Data Table                         | Logs user messages to chat_logs       | Telegram - Incoming Message    | DB - Get Lead by User ID       | ## Logging: Stores all bot replies in chat_logs for auditing and history.                                      |
| DB - Get Lead by User ID  | Data Table                        | Fetches existing lead record by chat_id | Log - User Message             | Check – Lead Record            | ## Lead Management: Checks if the user has an existing lead record. Creates a new lead if missing or updates it using AI-extracted data.                |
| Check – Lead Record       | If                                | Checks if lead exists                  | DB - Get Lead by User ID       | Build Assistant Context (true), DB - Create Lead (false) |                                                                                                               |
| DB - Create Lead          | Data Table                        | Creates new lead record                | Check – Lead Record (false)    | Build Assistant Context        |                                                                                                               |
| Build Assistant Context   | Code                              | Builds context object for AI           | Check – Lead Record (true), DB - Create Lead | AI - Smart Support Assistant  | ## Context Builder: Builds the assistant context object (lead data, telegram user, message) before sending it to the AI.                                |
| AI - Smart Support Assistant | Langchain Agent                 | Processes user input with AI, manages lead updates and tools | Build Assistant Context        | DB - Update Lead, DB - Get FAQ, DB - Get Services, DB - Get Settings, Log - Bot Message | ## AI Agent: Receives context, determines intent, updates leads if needed, retrieves services/FAQ/settings, and prepares the final reply.                |
| OpenRouter Chat Model     | Language Model                    | Provides language model API for AI    | AI - Smart Support Assistant (lmChatOpenRouter) | AI - Smart Support Assistant (ai_languageModel) |                                                                                                               |
| Simple Memory             | Memory Buffer Window              | Maintains chat memory context         | AI - Smart Support Assistant (ai_memory) | AI - Smart Support Assistant  |                                                                                                               |
| DB - Get FAQ              | Data Table Tool                  | Retrieves active FAQ entries           | AI - Smart Support Assistant (ai_tool) | AI - Smart Support Assistant  |                                                                                                               |
| DB - Get Services         | Data Table Tool                  | Retrieves active service entries       | AI - Smart Support Assistant (ai_tool) | AI - Smart Support Assistant  |                                                                                                               |
| DB - Get Settings         | Data Table Tool                  | Retrieves company settings             | AI - Smart Support Assistant (ai_tool) | AI - Smart Support Assistant  |                                                                                                               |
| DB - Update Lead          | Data Table Tool                  | Updates lead records with AI-extracted data | AI - Smart Support Assistant (ai_tool) | AI - Smart Support Assistant  |                                                                                                               |
| Log - Bot Message         | Data Table                      | Logs bot-generated messages            | AI - Smart Support Assistant   | Telegram - Send Response       | ## Logging: Stores all bot replies in chat_logs for auditing and history.                                      |
| Telegram - Send Response  | Telegram Node                   | Sends AI reply to Telegram user        | Log - Bot Message              |                               | ## Telegram Output: Sends the assistant’s final reply back to the user through Telegram.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure credentials with your Telegram Bot token (OAuth2).  
   - Set updates to listen for `message`.  
   - Position this as the workflow start node.

2. **Create Data Table Node: Log - User Message**  
   - Type: Data Table  
   - Connect from Telegram Trigger main output.  
   - Configure to write to `chat_logs` table with columns:  
     - `chat_id`: `={{ $json.message.chat.id }}`  
     - `direction`: `"user"`  
     - `message_text`: `={{ $json.message.text }}`

3. **Create Data Table Node: DB - Get Lead by User ID**  
   - Connect from Log - User Message.  
   - Configure to query `umrah_leads` table, limit 1.  
   - Filter: `chat_id` equals `={{ $json.message.chat.id }}`.

4. **Create If Node: Check – Lead Record**  
   - Connect from DB - Get Lead by User ID.  
   - Condition: Check if `chat_id` field is not empty in the returned data.

5. **Create Data Table Node: DB - Create Lead**  
   - Connect from Check – Lead Record false branch.  
   - Configure to insert into `umrah_leads` table with fields:  
     - `status`: `"new"`  
     - `chat_id`: `={{ $json.message.chat.id }}`  
     - `telegram_first_name`: `={{ $json.message.chat.first_name }}`  
     - `telegram_last_name`: `={{ $json.message.chat.last_name }}`

6. **Create Code Node: Build Assistant Context**  
   - Connect from Check – Lead Record true branch and DB - Create Lead node.  
   - Add JavaScript code:  
     - Extract Telegram chat info (`id`, `first_name`, `last_name`).  
     - Load lead data if available (full_name, phone, email, status, notes).  
     - Extract user message text.  
     - Return JSON object with keys: `telegram_user`, `lead_data`, `user_message`.

7. **Create Langchain Agent Node: AI - Smart Support Assistant**  
   - Connect from Build Assistant Context main output.  
   - Configure with system prompt specifying:  
     - AI responsibilities for support, lead completion logic, tools usage (services, FAQ, settings, lead update).  
     - Strict data privacy and response style requirements.  
   - Set input text as `={{ $json.user_message }}`.

8. **Create OpenRouter Chat Model Node**  
   - Configure with API key credential for OpenRouter.  
   - Set model options: temperature 0.2, max tokens 400, topP 1, frequency and presence penalties 0.2.  
   - Connect as AI language model for the Langchain Agent node.

9. **Create Simple Memory Node**  
   - Configure memory buffer window with session key: `={{ $('Telegram - Incoming Message').item.json.message.chat.id }}`.  
   - Context window length: 25.  
   - Connect as AI memory for Langchain Agent.

10. **Create Data Table Tool Nodes for AI Tools:**  
    - **DB - Get FAQ:** Filter `umrah_faq` table by `is_active` true, return all.  
    - **DB - Get Services:** Filter `umrah_services` table by `is_active` true, return all.  
    - **DB - Get Settings:** Get all records from `umrah_settings` table.  
    - Connect all as AI tools for Langchain Agent.

11. **Create Data Table Tool Node: DB - Update Lead**  
    - Connect as AI tool for Langchain Agent.  
    - Configure update operation on `umrah_leads` table, filter by `chat_id` from Telegram message.  
    - Map fields (`email`, `phone`, `full_name`, `status`, `notes`, etc.) dynamically based on AI output.

12. **Create Data Table Node: Log - Bot Message**  
    - Connect from AI - Smart Support Assistant main output.  
    - Log bot’s reply in `chat_logs` with:  
      - `chat_id`: Telegram user chat ID.  
      - `direction`: `"bot"`  
      - `message_text`: AI assistant output text.

13. **Create Telegram Node: Telegram - Send Response**  
    - Connect from Log - Bot Message.  
    - Configure to send message text from AI output to Telegram user `chat_id`.  
    - Use same Telegram API credentials as the trigger node.  
    - Disable append attribution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                             | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is a generic template for AI-powered Telegram customer support assistants with lead management, easily adaptable by modifying prompts or integrating with other data tables.                                                                 | Sticky Note in workflow main overview.                                                         |
| The system prompt is crafted to ensure strict privacy, no AI system disclosures, and clear stepwise lead data collection logic.                                                                                                                        | AI - Smart Support Assistant node configuration.                                              |
| For more advanced AI model features, ensure your OpenRouter API credentials are valid and properly scoped.                                                                                                                                                | OpenRouter Chat Model node credential setup.                                                  |
| Database tables involved: `umrah_chat_logs` (for message logs), `umrah_leads` (lead records), `umrah_faq` (FAQ data), `umrah_services` (service/program info), `umrah_settings` (company info).                                                           | Data Table node configurations.                                                               |
| Video introduction and detailed project explanation available at https://n8n.io/blog/ai-telegram-support-bot (example link for reference).                                                                                                             | Workflow sticky notes or project documentation.                                               |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow and fully complies with applicable content policies. It contains no illegal, offensive, or protected elements. All data handled is lawful and public.