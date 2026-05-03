Automating WhatsApp replies using Go High Level with Redis and Anthropic

https://n8nworkflows.xyz/workflows/automating-whatsapp-replies-using-go-high-level-with-redis-and-anthropic-7191


# Automating WhatsApp replies using Go High Level with Redis and Anthropic

### 1. Workflow Overview

This workflow automates WhatsApp and SMS replies in Go High Level (GHL) using Redis for message buffering and Anthropic's Claude AI for generating accurate, data-driven responses based on a predefined dataset called ClientInfo. It is designed to integrate with the Wazzap plugin for WhatsApp message handling within GHL, ensuring replies are sent through GHL automations triggered by updates to a custom contact field.

**Target Use Cases:**  
- Businesses using Go High Level and Wazzap to manage WhatsApp and SMS conversations.  
- Teams requiring AI-generated, precise responses grounded solely on verified ClientInfo data.  
- Use cases where minimizing additional automation costs in GHL is important by updating a custom field rather than triggering inbound automations on every message.

**Logical Blocks:**

- **1.1 Input Reception & Filtering:** Receives inbound messages from GHL via Webhook, filters audio messages, sanitizes text messages.  
- **1.2 Redis-based Message Buffering:** Buffers multiple messages per contact within a time window using Redis lists to batch process user inputs.  
- **1.3 AI Processing with ClientInfo Tool:** Uses a LangChain AI Agent powered by Anthropic Claude to generate responses strictly from ClientInfo data.  
- **1.4 Output Sanitization & GHL Contact Update:** Cleans AI output and updates the GHL contact custom field (IA_answer) to trigger outbound SMS replies.  
- **1.5 Auxiliary:** Optional sub-workflow example to merge client info from Excel files stored in Google Drive.  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Filtering

**Overview:**  
This block receives WhatsApp/SMS messages from GHL via a webhook, checks if the message is an audio type, and routes appropriately. Text messages are sanitized for safe processing.

**Nodes Involved:**  
- Webhook  
- Switch1  
- Sanitize and Format Message Body  
- Get GHL Custom fields  
- Update Contact No audio  

**Node Details:**  

- **Webhook**  
  - Type: HTTP Webhook (POST)  
  - Configured to receive inbound messages from GHL automation (Customer Replied SMS + Wazzap plugin).  
  - Path fixed for secure, unique webhook URL.  
  - Input: HTTP POST JSON with message details including contact_id and message body.  
  - Output: Forwards data to Switch1.  
  - Edge cases: Malformed payloads, unauthorized access (should be secured externally).  

- **Switch1**  
  - Type: Switch (conditional branching)  
  - Logic: Checks if the message body contains "type message: audio" string to detect audio messages from Wazzap/GHL.  
  - Branch 1 (No audio): Passes sanitized text messages to Sanitize and Format Message Body node.  
  - Branch 2 (Audio): Proceeds to Get GHL Custom fields for fallback reply.  
  - Edge cases: Message body missing or incorrectly formatted; may misclassify message types.

- **Sanitize and Format Message Body**  
  - Type: Code Node (JavaScript)  
  - Function: Fixes incorrectly escaped double quotes, escapes line breaks, carriage returns, and tabs, trims whitespace.  
  - Input: Raw text message body from webhook.  
  - Output: Sanitized text assigned to property `text`.  
  - Edge cases: Unexpected characters, empty messages, Unicode handling.  

- **Get GHL Custom fields**  
  - Type: HTTP Request (GET)  
  - Function: Retrieves all custom fields from GHL sub-account to later find the ID of the custom field `IA_answer`.  
  - Auth: HTTP Bearer with GHL API Key.  
  - Output: JSON array of custom fields.  
  - Edge cases: API authentication failure, network errors, API rate limits.

- **Update Contact No audio**  
  - Type: HTTP Request (PUT)  
  - Function: Updates contact’s custom field `IA_answer` with a fallback message: "Sorry, I can't listen to voice notes at the moment..."  
  - Uses field ID from Get GHL Custom fields node.  
  - Auth: HTTP Bearer with GHL API Key.  
  - Triggered only for audio messages to politely request text input.  
  - Edge cases: API errors, missing custom field ID, malformed JSON.  

---

#### 1.2 Redis-based Message Buffering

**Overview:**  
Buffers multiple messages per contact received within a short timeframe using Redis lists to batch process messages, avoiding fragmented or premature AI responses.

**Nodes Involved:**  
- Sanitize and Format Message Body (output connection)  
- Save message (Redis push)  
- Wait (delay)  
- Get messages (Redis get)  
- If (compare last message)  
- messages (Set node to join messages)  
- Delete messages (Redis delete)  

**Node Details:**  

- **Save message**  
  - Type: Redis node (List Push)  
  - Pushes sanitized message text to a Redis list keyed by contact_id.  
  - Input: Sanitized `text` from previous node.  
  - Output: Triggers Wait node.  
  - Edge cases: Redis connectivity, authentication failure.

- **Wait**  
  - Type: Wait node  
  - Pauses workflow for 15 seconds to allow user to finish typing multiple messages.  
  - Output: Triggers Get messages node after delay.  
  - Edge cases: Workflow timeout or interruption.

- **Get messages**  
  - Type: Redis node (Get list)  
  - Reads all messages from Redis list for contact_id into a collection named `mensajes`.  
  - Output: Passes data to If node.  
  - Edge cases: Redis errors, empty list handling.

- **If**  
  - Type: Conditional node  
  - Compares last message in Redis list with the latest sanitized message; if equal, proceeds to merge messages, otherwise no action to avoid premature response.  
  - Output 1 (true): Passes to `messages` node.  
  - Output 2 (false): Passes to No Operation node (does nothing).  
  - Edge cases: Timing issues if messages arrive very fast.

- **messages (Set node)**  
  - Joins all buffered messages with newline characters into a single string assigned to `message`.  
  - Also assigns `id` as contact_id.  
  - Output: Triggers Delete messages node.  

- **Delete messages**  
  - Redis node (Delete key)  
  - Deletes Redis list for contact_id to clear the message buffer after processing.  
  - Output: Triggers AI Agent node.  
  - Edge cases: Redis errors or key not found.

---

#### 1.3 AI Processing with ClientInfo Tool

**Overview:**  
Invokes a LangChain AI Agent configured with Anthropic Claude Sonnet 4 to generate responses strictly based on ClientInfo data. Maintains conversational context with Simple Memory.

**Nodes Involved:**  
- AI Agent  
- Anthropic Chat Model  
- Simple Memory  
- Call n8n Workflow Tool (ClientInfo sub-workflow)  
- Sanitize and Format Message Output  

**Node Details:**  

- **AI Agent**  
  - Type: LangChain AI Agent node  
  - Input: Joined user messages (`message`) from Redis buffer.  
  - Uses a system prompt that strictly instructs the agent to answer only using ClientInfo data, with detailed rules, examples, and tone instructions.  
  - Calls ClientInfo sub-workflow tool to fetch exact service/branch information.  
  - Maintains conversational context per user phone number using Simple Memory.  
  - Output: Raw AI response forwarded for sanitizing.  
  - Edge cases: API rate limits, prompt failures, unexpected AI output.

- **Anthropic Chat Model**  
  - Type: LLM Chat Model (Anthropic Claude Sonnet 4)  
  - Configured as the underlying language model for AI Agent.  
  - API credentials required.  
  - Output: AI-generated text response.

- **Simple Memory**  
  - Type: LangChain Memory Buffer (Window)  
  - Stores recent conversational history keyed by user's phone number from webhook.  
  - Keeps context short for relevant AI replies.  

- **Call n8n Workflow Tool**  
  - Type: LangChain Workflow Tool node  
  - Invokes a sub-workflow providing the ClientInfo data (not detailed here), which acts as the knowledge base for AI responses.  
  - Edge cases: Sub-workflow failures, timeout.

- **Sanitize and Format Message Output**  
  - Type: Code node  
  - Cleans AI output by removing extraneous quotes, unescaping characters, escaping newlines, carriage returns, and tabs, and trimming whitespace.  
  - Prepares the text safely for JSON embedding.  
  - Output: Triggers GHL Custom fields node for updating contact.  
  - Edge cases: Unexpected AI output formats, encoding issues.

---

#### 1.4 Output Sanitization & GHL Contact Update

**Overview:**  
Updates the Go High Level contact's custom field `IA_answer` with the sanitized AI response to trigger an outbound SMS reply automation.

**Nodes Involved:**  
- GHL Custom fields  
- Update Contact  

**Node Details:**  

- **GHL Custom fields**  
  - Type: HTTP Request (GET)  
  - Retrieves the list of all custom fields from Go High Level sub-account, to find the ID for `IA_answer`.  
  - Authenticated via HTTP Bearer token.  
  - Output: Provides custom fields data to Update Contact.  
  - Edge cases: API errors, token expiry.

- **Update Contact**  
  - Type: HTTP Request (PUT)  
  - Updates the specific contact (by contact_id) setting the `IA_answer` custom field to the sanitized AI response.  
  - Uses the custom field ID found in previous node.  
  - Authenticated via HTTP Bearer token.  
  - This update triggers the outbound SMS automation in GHL.  
  - Edge cases: API errors, missing contact_id, field ID mismatch.

---

#### 1.5 Auxiliary: Google Drive Excel Merge Sub-Workflow Example (Optional)

**Overview:**  
An optional example workflow that downloads an Excel file from Google Drive, extracts two sheets (`Tests` and `Sites`), merges their data, and returns it as a data source for ClientInfo.

**Nodes Involved:**  
- When Executed by Another Workflow (trigger)  
- Download file (Google Drive)  
- Extract Tests (ExtractFromFile node for `Tests` sheet)  
- Extract Sites (ExtractFromFile node for `Sites` sheet)  
- Merge (merges extracted data)  
- Success (final output)  

**Node Details:**  

- **When Executed by Another Workflow**  
  - Trigger node for sub-workflow invocation.  

- **Download file**  
  - Google Drive node downloads specified Excel file by file ID.  
  - Requires Google Drive OAuth2 credentials.  

- **Extract Tests** and **Extract Sites**  
  - ExtractFromFile nodes parse the corresponding Excel sheets.  
  - Output dataset used for merging.

- **Merge**  
  - Combines data from both sheets into a single dataset.  

- **Success**  
  - Sets the final merged dataset as output for sub-workflow return.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                           | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|-----------------------------------------------------------|------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                     | Webhook                          | Receives inbound WhatsApp/SMS messages from GHL           | -                            | Switch1                        | See "GHL to n8n SMS Reply Bridge" note about webhook and filtering setup.                        |
| Switch1                     | Switch                          | Filters audio vs text messages                             | Webhook                      | Sanitize and Format Message Body (text), Get GHL Custom fields (audio) |                                                                                                 |
| Sanitize and Format Message Body | Code                           | Cleans/sanitizes incoming text message                     | Switch1 (text branch)         | Save message                   |                                                                                                 |
| Save message                | Redis (List Push)                 | Buffers messages in Redis list                             | Sanitize and Format Message Body | Wait                         | "Redis Message Buffer Workflow" note explains buffering and Redis use.                          |
| Wait                        | Wait                            | Delays processing to batch multiple messages               | Save message                 | Get messages                  |                                                                                                 |
| Get messages                | Redis (Get list)                 | Retrieves buffered messages from Redis                     | Wait                        | If                            |                                                                                                 |
| If                          | If                              | Checks last message identity for batching                  | Get messages                | messages (true), No Operation (false) |                                                                                                 |
| messages                   | Set                             | Joins buffered messages into one string                    | If (true)                  | Delete messages               |                                                                                                 |
| Delete messages             | Redis (Delete key)               | Clears Redis buffer after messages are processed           | messages                   | AI Agent                     |                                                                                                 |
| AI Agent                   | LangChain AI Agent               | Generates AI response strictly using ClientInfo data       | Delete messages             | Sanitize and Format Message Output | "ClientInfo LLM Response Workflow" note covers AI Agent usage and prompt details.               |
| Anthropic Chat Model        | LLM Chat Model                  | Provides AI language model for AI Agent                    | AI Agent (ai_languageModel) | AI Agent                     |                                                                                                 |
| Simple Memory               | LangChain Memory Buffer         | Maintains short conversational context                     | AI Agent (ai_memory)         | AI Agent                     |                                                                                                 |
| Call n8n Workflow Tool      | LangChain Tool Workflow         | Calls ClientInfo sub-workflow for accurate data            | AI Agent (ai_tool)           | AI Agent                     |                                                                                                 |
| Sanitize and Format Message Output | Code                           | Cleans AI output for JSON embedding                         | AI Agent                    | GHL Custom fields            |                                                                                                 |
| GHL Custom fields           | HTTP Request (GET)              | Retrieves GHL custom fields list                            | Sanitize and Format Message Output | Update Contact              | See "GHL Update Contact Workflow" note for details on custom fields retrieval and usage.         |
| Update Contact              | HTTP Request (PUT)              | Updates contact custom field with AI response               | GHL Custom fields           | -                            |                                                                                                 |
| Get GHL Custom fields       | HTTP Request (GET)              | Retrieves GHL custom fields list for audio fallback         | Switch1 (audio branch)       | Update Contact No audio       |                                                                                                 |
| Update Contact No audio     | HTTP Request (PUT)              | Updates contact with fallback message for audio inputs      | Get GHL Custom fields       | -                            |                                                                                                 |
| No Operation, do nothing    | NoOp                           | Prevents workflow continuation if conditions not met        | If (false branch)           | -                            |                                                                                                 |
| When Executed by Another Workflow | Execute Workflow Trigger         | Trigger for optional Google Drive Excel Merge sub-workflow | -                            | Download file                | See "Google Drive Excel Merge Workflow" note for optional client info enrichment.              |
| Download file              | Google Drive                    | Downloads Excel file from Google Drive                      | When Executed by Another Workflow | Extract Tests, Extract Sites |                                                                                                 |
| Extract Tests              | ExtractFromFile (XLSX sheet)   | Extracts data from Excel "Tests" sheet                      | Download file               | Merge                        |                                                                                                 |
| Extract Sites              | ExtractFromFile (XLSX sheet)   | Extracts data from Excel "Sites" sheet                      | Download file               | Merge                        |                                                                                                 |
| Merge                      | Merge                          | Combines extracted Excel data                               | Extract Tests, Extract Sites | Success                      |                                                                                                 |
| Success                    | Set                            | Outputs merged data for sub-workflow return                 | Merge                       | -                            |                                                                                                 |
| messages                   | Set                            | Joins messages from Redis                                   | If (true)                   | Delete messages              |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook (POST)  
   - Path: Unique string (e.g., `fce352e8-8edf-447c-9b54-c2c2b6649ccc`)  
   - Purpose: Receive inbound WhatsApp/SMS messages from GHL automation.  

2. **Add Switch Node (Switch1)**  
   - Condition: Check if incoming message body contains `"type message: audio"`  
   - Output 1: If *no* audio detected → route to sanitize text node.  
   - Output 2: If audio detected → route to get custom fields node for fallback.  

3. **Add Code Node "Sanitize and Format Message Body"**  
   - JS code:  
     ```js
     return [{
       text: $input.first().json.body.message.body
         .replace(/\\+\"/g, '"')
         .replace(/\n/g, "\\n")
         .replace(/\r/g, "\\r")
         .replace(/\t/g, "\\t")
         .trim()
     }];
     ```  
   - Input: message body from webhook.  
   - Output: sanitized text as `text`.

4. **Add Redis Node "Save message"**  
   - Operation: Push  
   - List: Use contact_id from webhook JSON as key.  
   - Message: Use sanitized `text` from previous node.  

5. **Add Wait Node**  
   - Duration: 15 seconds.  
   - Purpose: Buffer messages to allow multi-part user input.

6. **Add Redis Node "Get messages"**  
   - Operation: Get list by contact_id key.  
   - Output: List of buffered messages.

7. **Add If Node**  
   - Condition: Check if last message in Redis list equals latest sanitized text.  
   - True → proceed to join messages.  
   - False → no operation (skip processing).

8. **Add Set Node "messages"**  
   - Join all messages with newline `\n`.  
   - Assign joined string to `message`.  
   - Assign contact_id to `id`.

9. **Add Redis Node "Delete messages"**  
   - Operation: Delete key by contact_id.  
   - Purpose: Clear Redis buffer after processing.

10. **Add LangChain AI Agent Node "AI Agent"**  
    - Text input: Use joined `message` string.  
    - System prompt: Detailed instructions to answer only from ClientInfo tool data.  
    - Tools: Add "Call n8n Workflow Tool" for ClientInfo sub-workflow.  
    - Memory: Use Simple Memory node keyed by user phone.  
    - Language Model: Use Anthropic Claude Sonnet 4 node.

11. **Add Anthropic Chat Model Node**  
    - Model: Claude Sonnet 4 (or preferred model).  
    - Credentials: Anthropic API key.

12. **Add Simple Memory Node**  
    - Session key: User phone from webhook.  

13. **Add "Call n8n Workflow Tool" Node**  
    - Workflow ID: ClientInfo sub-workflow ID.  
    - Purpose: Provide exact data source to AI Agent.

14. **Add Code Node "Sanitize and Format Message Output"**  
    - JS code to unescape quotes, remove extra chars, escape newlines, carriage returns, tabs.  
    - Output: Cleaned AI response.

15. **Add HTTP Request Node "GHL Custom fields"**  
    - Method: GET  
    - URL: `https://rest.gohighlevel.com/v1/custom-fields/`  
    - Auth: HTTP Bearer with GHL API key.

16. **Add HTTP Request Node "Update Contact"**  
    - Method: PUT  
    - URL: `https://rest.gohighlevel.com/v1/contacts/{{contact_id}}`  
    - JSON Body: Update custom field `IA_answer` with sanitized AI response.  
    - Auth: HTTP Bearer with GHL API key.

17. **Add HTTP Request Node "Get GHL Custom fields"** (for audio messages)  
    - Same as step 15.  

18. **Add HTTP Request Node "Update Contact No audio"**  
    - Method: PUT  
    - URL: `https://rest.gohighlevel.com/v1/contacts/{{contact_id}}`  
    - JSON Body: Set `IA_answer` field to polite fallback message for audio inputs.  
    - Auth: HTTP Bearer with GHL API key.

19. **Add No Operation Node**  
    - Used to terminate paths where no action is needed (e.g., If false branch).

20. **Optional: Setup Google Drive Excel Merge Sub-Workflow**  
    - Set up Google OAuth2 credentials with Drive API enabled.  
    - Create workflow with trigger node "When Executed by Another Workflow".  
    - Download Excel file by fileId.  
    - Extract "Tests" and "Sites" sheets using ExtractFromFile nodes.  
    - Merge extracted data into one dataset.  
    - Return merged dataset as output for ClientInfo tool usage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| **GHL to n8n SMS Reply Bridge**: Requires GoHighLevel sub-account API key. Set up GHL automation to trigger on Customer Replied (SMS) and send payload to n8n Webhook. Use Wazzap plugin for WhatsApp messages. Cost-saving tip: Use a custom field (IA_answer) updated with bot response to avoid triggering extra inbound automations on every message.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note node near Webhook and Switch1.                                                                                                           |
| **Redis Message Buffer Workflow**: Buffers multiple messages per contact within a short window using Redis lists keyed by contact_id. Wait node pauses workflow (15s) to allow batching. After buffer, messages are merged and sent to AI processing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note node near Redis nodes.                                                                                                                   |
| **ClientInfo LLM Response Workflow**: AI Agent uses Anthropic Claude Sonnet 4, with prompts forcing answers only from ClientInfo tool (a sub-workflow). Uses Simple Memory keyed by phone number for context. AI output sanitized before updating GHL.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note node near AI Agent and Sanitize Output nodes.                                                                                            |
| **GHL Update Contact Workflow**: Retrieves GHL custom fields via API, finds IA_answer field ID, and updates contact with AI response. GHL automation triggered on IA_answer update sends SMS reply to user.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note node near GHL Custom fields and Update Contact nodes.                                                                                     |
| **Main Workflow Description**: Integrates GHL + Wazzap with Redis and Anthropic AI to automate accurate WhatsApp and SMS replies based on ClientInfo data. Suitable for businesses wanting AI-driven, data-accurate, cost-effective customer communication automation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note near workflow top left.                                                                                                                  |
| **Google Drive Excel Merge Workflow**: Optional sub-workflow example that downloads Excel file from Google Drive, extracts sheets, merges data, and returns it for enriching ClientInfo tool. Requires Google OAuth2 credentials and Drive API enabled.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note near Download file and Extract nodes.                                                                                                   |

---

**Disclaimer:** The text above is exclusively generated from an n8n automated workflow, respecting current content policies and containing no illegal or protected elements. All data handled is legal and public.