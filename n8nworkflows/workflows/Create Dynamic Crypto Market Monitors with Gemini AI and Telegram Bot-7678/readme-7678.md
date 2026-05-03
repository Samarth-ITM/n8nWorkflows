Create Dynamic Crypto Market Monitors with Gemini AI and Telegram Bot

https://n8nworkflows.xyz/workflows/create-dynamic-crypto-market-monitors-with-gemini-ai-and-telegram-bot-7678


# Create Dynamic Crypto Market Monitors with Gemini AI and Telegram Bot

---
### 1. Workflow Overview

This workflow enables dynamic monitoring of cryptocurrency market conditions using Gemini AI analysis and Telegram Bot interaction. It allows authorized Telegram users to manage custom "watchdogs" for crypto symbols with specific monitoring intervals and user-defined prompts. The workflow integrates user commands via Telegram, stores monitoring configurations in PostgreSQL, creates and activates dedicated sub-workflows for each symbol, analyzes chart images with Gemini AI, and sends alert notifications back to Telegram.

**Target use cases:**  
- Crypto traders or analysts monitoring multiple crypto symbols with AI-assisted chart analysis  
- Automating alert generation and delivery via Telegram based on custom user prompts  
- Managing dynamic sets of market monitors with add/list/delete commands

**Logical blocks:**  
1.1 Telegram Command Reception and Authorization  
1.2 User Command Parsing and Routing  
1.3 Watchdog Configuration Management (PostgreSQL Storage)  
1.4 Dynamic Watchdog Sub-Workflow Creation and Activation  
1.5 Scheduled Market Data Retrieval and AI Image Analysis  
1.6 Alert Decision and Notification Delivery via Telegram

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Command Reception and Authorization  
**Overview:**  
Receives Telegram messages and verifies if the user is authorized to interact with the bot. Unauthorized users receive a polite rejection message.

**Nodes Involved:**  
- Trigger (Telegram Trigger)  
- Authorization (If)  
- Error Authorization (Telegram)  

**Node Details:**  

- **Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point for Telegram messages with update type "message"  
  - Config: Listens for all incoming messages from users  
  - Inputs: Telegram webhook  
  - Outputs: Passes message JSON downstream  
  - Edge cases: Telegram API downtime, webhook misconfiguration  

- **Authorization**  
  - Type: If  
  - Role: Checks if the Telegram message sender ID equals a predefined authorized ID (111111) and that the message entity is of type "bot_command"  
  - Config: Strict number equality for user ID and string equality for message entity type  
  - Inputs: Output from Trigger  
  - Outputs: Two branches - authorized (true) and unauthorized (false)  
  - Edge cases: If user ID or message entity absent, evaluation fails  

- **Error Authorization**  
  - Type: Telegram  
  - Role: Sends an unauthorized message reply to the user  
  - Config: Sends message "Hi ~ {username}\nYou are not authorized, thk !" replying to the original message  
  - Inputs: Unauthorized branch from Authorization node  
  - Outputs: Ends unauthorized user flow  
  - Edge cases: Telegram API errors, invalid chat ID  

---

#### 1.2 User Command Parsing and Routing  
**Overview:**  
Parses the user message text to extract commands and parameters, then routes based on the command type.

**Nodes Involved:**  
- Parse user text (Code)  
- Switch (Switch node for routing commands)  

**Node Details:**  

- **Parse user text**  
  - Type: Code  
  - Role: Parses Telegram message text for commands like /add, /delete, /list, /search, /start, /help with regex and extracts parameters (symbol, interval, prompt)  
  - Config: JavaScript code with regex matching and switch-case for command validation  
  - Inputs: Authorized Telegram messages  
  - Outputs: JSON with fields: chatId, command, symbol, interval, prompt, error  
  - Edge cases: Input format errors, unknown commands, missing parameters  

- **Switch**  
  - Type: Switch  
  - Role: Routes workflow execution to different branches based on parsed command value (start, help, add, list, delete)  
  - Config: Case-sensitive string equality for command  
  - Inputs: Output from Parse user text  
  - Outputs: Multiple branches, one per command  
  - Edge cases: Unmatched commands lead to errors downstream  

---

#### 1.3 Watchdog Configuration Management (PostgreSQL Storage)  
**Overview:**  
Manages storage of watch configurations in PostgreSQL, supporting add, list, and delete operations.

**Nodes Involved:**  
- Add Data (Postgres)  
- Get list (Postgres)  
- Merge All list (Code)  
- Send list (Telegram)  
- Get workflow id (Postgres)  
- Delete the Workflow (HTTP Request)  
- Delete the row (Postgres)  
- Update workflow id (Postgres)  

**Node Details:**  

- **Add Data**  
  - Type: Postgres  
  - Role: Inserts or updates a watchdog record with symbol and user_prompt  
  - Config: Upsert SQL query with ON CONFLICT on symbol  
  - Inputs: Command "add" branch from Switch  
  - Outputs: Passes to sub-workflow creation step  
  - Edge cases: DB connection errors, SQL syntax errors  

- **Get list**  
  - Type: Postgres  
  - Role: Retrieves all watchdog records for listing  
  - Config: Simple SELECT * FROM watchdog  
  - Inputs: Command "list" branch  
  - Outputs: Passes data to formatting node  
  - Edge cases: Empty table, DB errors  

- **Merge All list**  
  - Type: Code  
  - Role: Formats the list of watchdogs into a multiline Markdown text string for Telegram display  
  - Config: Converts each row into a line "`symbol`: user_prompt"  
  - Inputs: Output of Get list  
  - Outputs: JSON with listText string  
  - Edge cases: Empty input array  

- **Send list**  
  - Type: Telegram  
  - Role: Sends formatted list text message to user with MarkdownV2 parse mode  
  - Config: Replies to original user message  
  - Inputs: Output of Merge All list  
  - Outputs: Ends list command flow  
  - Edge cases: Telegram message length limits, API errors  

- **Get workflow id**  
  - Type: Postgres  
  - Role: Retrieves watchdog record by symbol to get associated workflow_id (for deletion)  
  - Config: SELECT by symbol  
  - Inputs: Command "delete" branch  
  - Outputs: Passes workflow_id to deletion HTTP request  
  - Edge cases: Missing symbol, no matching record  

- **Delete the Workflow**  
  - Type: HTTP Request  
  - Role: Calls n8n API to delete the sub-workflow identified by workflow_id  
  - Config: DELETE request to local n8n server with API key auth  
  - Inputs: Output of Get workflow id  
  - Outputs: Triggers deletion of DB row  
  - Edge cases: HTTP errors, invalid workflow_id  

- **Delete the row**  
  - Type: Postgres  
  - Role: Deletes watchdog record from DB by symbol  
  - Config: DELETE with WHERE symbol = ...  
  - Inputs: Output of Delete the Workflow  
  - Outputs: Sends success message  
  - Edge cases: DB errors, missing symbol  

- **Update workflow id**  
  - Type: Postgres  
  - Role: Updates watchdog record with newly created sub-workflow ID after creation  
  - Config: UPDATE with symbol match  
  - Inputs: Output of Create Workflow HTTP request  
  - Outputs: Ends add command flow  
  - Edge cases: DB errors, missing workflow_id  

---

#### 1.4 Dynamic Watchdog Sub-Workflow Creation and Activation  
**Overview:**  
Generates and activates a new n8n workflow dynamically for monitoring a specific symbol and interval based on user prompt.

**Nodes Involved:**  
- Code (Code node generating workflow JSON)  
- Create Workflow (HTTP Request)  
- Activate (HTTP Request)  
- WatchDog Success (Telegram)  
- Test return success (Telegram)  

**Node Details:**  

- **Code**  
  - Type: Code  
  - Role: Creates a JSON definition of a sub-workflow for the specified symbol and interval with schedule trigger, DB queries, HTTP request for chart image, Gemini AI analysis, and alert logic  
  - Config: JavaScript builds a workflow JSON dynamically, using parameters from Switch node  
  - Inputs: Output of Add Data or other command handling  
  - Outputs: Workflow JSON for creation  
  - Edge cases: Code errors, JSON parse errors, missing inputs  

- **Create Workflow**  
  - Type: HTTP Request  
  - Role: Sends the generated workflow JSON to local n8n API to create new workflow  
  - Config: POST /api/v1/workflows with API key auth  
  - Inputs: Output of Code node  
  - Outputs: Workflow creation response including ID  
  - Edge cases: API errors, network issues  

- **Activate**  
  - Type: HTTP Request  
  - Role: Activates the newly created workflow by ID  
  - Config: POST /api/v1/workflows/{id}/activate with API key auth  
  - Inputs: Output of Create Workflow  
  - Outputs: Triggers success notification  
  - Edge cases: Activation failures  

- **WatchDog Success**  
  - Type: Telegram  
  - Role: Notifies user that watchdog creation was successful  
  - Config: Personalized message referencing symbol  
  - Inputs: Output of Activate  
  - Outputs: Ends add command flow  

- **Test return success**  
  - Type: Telegram  
  - Role: Responds to /start command to confirm bot authorization and functionality  
  - Config: Replies with "Test return Success!"  
  - Inputs: Switch branch "start"  
  - Outputs: Ends start command flow  

---

#### 1.5 Scheduled Market Data Retrieval and AI Image Analysis  
**Overview:**  
Scheduled sub-workflow for each watchdog retrieves chart image URL and analyzes it with Gemini AI to generate alerts.

**Nodes Involved:**  
- Schedule Trigger  
- Prompt (Set)  
- Symbol (Set)  
- Execute a SQL query (Postgres)  
- Get Image URL (HTTP Request)  
- Analyze image (Google Gemini AI)  
- Code (parsing AI output)  
- Switch (alert decision)  
- Send a text message (Telegram)  
- Send a photo message (Telegram)  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution at intervals defined by user (e.g., 1h, 15m, 1d)  
  - Config: Dynamic interval setting based on user input  
  - Inputs: None (trigger)  
  - Outputs: Passes to Prompt node  

- **Prompt**  
  - Type: Set  
  - Role: Defines the AI prompt text including instructions and user prompt from DB  
  - Config: Static prompt with placeholders for user_prompt and image analysis instructions  
  - Inputs: Schedule Trigger  
  - Outputs: Passes to Symbol node  

- **Symbol**  
  - Type: Set  
  - Role: Sets symbol variable for DB queries and HTTP requests  
  - Config: Assigns symbol from parameters  
  - Inputs: Prompt node  
  - Outputs: Executes SQL query  

- **Execute a SQL query**  
  - Type: Postgres  
  - Role: Retrieves user prompt for the symbol from watchdog table  
  - Config: SELECT * WHERE symbol=...  
  - Inputs: Symbol node  
  - Outputs: Passes image request  

- **Get Image URL**  
  - Type: HTTP Request  
  - Role: Retrieves chart image URL from external Chart Image API with symbol and interval parameters  
  - Config: GET request with HTTP header authentication  
  - Inputs: Execute SQL query output  
  - Outputs: Passes image URL to AI analysis  

- **Analyze image**  
  - Type: Google Gemini AI node  
  - Role: Sends image URL and prompt text to Gemini AI for advanced chart analysis  
  - Config: Uses Gemini model "gemini-2.5-flash" with text and image URL inputs  
  - Inputs: Get Image URL output  
  - Outputs: AI response JSON  

- **Code**  
  - Type: Code  
  - Role: Extracts JSON alert data from Gemini AI text output, parses it into usable JSON  
  - Config: Regex to find JSON block in AI response, robust error handling  
  - Inputs: Analyze image output  
  - Outputs: Parsed alert data  

- **Switch**  
  - Type: Switch  
  - Role: Routes alert based on is_alert field ("Y" or "N")  
  - Config: String equals condition on is_alert  
  - Inputs: Code node output  
  - Outputs: If alert ("Y") sends notification; if no alert ("N") no output  

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends alert text message with HTML formatting to Telegram channel/thread  
  - Config: Targets channel ID with message thread  
  - Inputs: Switch alert branch "Y"  

- **Send a photo message**  
  - Type: Telegram  
  - Role: Sends chart image photo to Telegram channel/thread  
  - Config: Uses URL from Get Image URL node output  
  - Inputs: Send a text message node  

---

#### 1.6 Help and List Command Responses  
**Overview:**  
Provides users with command assistance and lists current watchdogs.

**Nodes Involved:**  
- Return help manu (Telegram)  
- Get list (Postgres)  
- Merge All list (Code)  
- Send list (Telegram)  

**Node Details:**  

- **Return help manu**  
  - Type: Telegram  
  - Role: Replies with a static help message listing available commands  
  - Config: Markdown text, replies to user message  
  - Inputs: Switch branch "help"  

- **Get list, Merge All list, Send list**  
  - As described in Block 1.3  

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                               | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                         |
|---------------------|---------------------------|-----------------------------------------------|--------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Trigger             | Telegram Trigger          | Receives Telegram messages                     | -                        | Authorization                 |                                                                                                   |
| Authorization       | If                        | Checks if user is authorized                    | Trigger                   | Parse user text, Error Authorization |                                                                                                   |
| Error Authorization | Telegram                  | Sends unauthorized message                      | Authorization            | -                             |                                                                                                   |
| Parse user text     | Code                      | Parses Telegram commands and parameters        | Authorization            | Switch                       |                                                                                                   |
| Switch              | Switch                    | Routes based on command                         | Parse user text           | Add Data, Get list, Get workflow id, Return help manu, Test return success |                                                                                                   |
| Add Data            | Postgres                  | Adds/updates watchdog config                    | Switch (add)              | Code                         |                                                                                                   |
| Code                | Code                      | Generates sub-workflow JSON for watchdog       | Add Data                  | Create Workflow               |                                                                                                   |
| Create Workflow     | HTTP Request              | Creates new workflow in n8n                     | Code                      | Activate, Update workflow id  |                                                                                                   |
| Activate            | HTTP Request              | Activates created workflow                      | Create Workflow           | WatchDog Success             |                                                                                                   |
| WatchDog Success    | Telegram                  | Notifies user of successful watchdog creation  | Activate                  | -                           |                                                                                                   |
| Test return success | Telegram                  | Replies to /start command                        | Switch (start)            | -                           |                                                                                                   |
| Get list            | Postgres                  | List all watchdogs                              | Switch (list)             | Merge All list               |                                                                                                   |
| Merge All list      | Code                      | Formats watchdog list text                       | Get list                  | Send list                   |                                                                                                   |
| Send list           | Telegram                  | Sends formatted list to user                    | Merge All list            | -                           |                                                                                                   |
| Get workflow id     | Postgres                  | Gets workflow ID for symbol (for deletion)     | Switch (delete)           | Delete the Workflow          |                                                                                                   |
| Delete the Workflow | HTTP Request              | Deletes watchdog workflow                        | Get workflow id           | Delete the row               |                                                                                                   |
| Delete the row      | Postgres                  | Removes watchdog DB entry                        | Delete the Workflow       | WatchDog Success1            |                                                                                                   |
| WatchDog Success1   | Telegram                  | Notifies user of successful deletion            | Delete the row            | -                           |                                                                                                   |
| Update workflow id  | Postgres                  | Updates watchdog record with workflow ID        | Create Workflow           | -                           |                                                                                                   |
| Schedule Trigger    | Schedule Trigger          | Triggers scheduled watchdog execution           | -                        | Prompt                      |                                                                                                   |
| Prompt              | Set                       | Sets AI prompt text for Gemini                   | Schedule Trigger          | Symbol                      |                                                                                                   |
| Symbol              | Set                       | Sets symbol for queries and requests             | Prompt                    | Execute a SQL query          |                                                                                                   |
| Execute a SQL query | Postgres                  | Retrieves user prompt for symbol                  | Symbol                    | Get Image URL                |                                                                                                   |
| Get Image URL       | HTTP Request              | Retrieves chart image URL                         | Execute a SQL query       | Analyze image               |                                                                                                   |
| Analyze image       | Google Gemini AI           | Analyzes chart image according to prompt         | Get Image URL             | Code                        |                                                                                                   |
| Code (AI parsing)   | Code                      | Extracts JSON alert data from AI response         | Analyze image             | Switch                      |                                                                                                   |
| Switch (alert)      | Switch                    | Routes based on alert determination              | Code (AI parsing)         | Send a text message          |                                                                                                   |
| Send a text message | Telegram                  | Sends alert text to Telegram channel/thread       | Switch (alert Y)          | Send a photo message         |                                                                                                   |
| Send a photo message| Telegram                  | Sends chart image photo to Telegram channel       | Send a text message       | -                           |                                                                                                   |
| Return help manu    | Telegram                  | Sends help message listing commands               | Switch (help)             | -                           |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook to receive "message" updates  
   - Connect output to Authorization node  

2. **Create Authorization Node (If)**  
   - Type: If  
   - Condition 1: `$json.message.from.id` equals authorized user ID (e.g., 111111, number)  
   - Condition 2: `$json.message.entities[0].type` equals "bot_command"  
   - Connect true output to Parse user text node  
   - Connect false output to Error Authorization node  

3. **Create Error Authorization Node (Telegram)**  
   - Type: Telegram  
   - Text: `"Hi ~ {{ $json.message.chat.username }}\nYou are not authorized, thk !"`  
   - Chat ID: `{{$json.message.chat.id}}`  
   - Reply to original message ID  
   - Credential: Telegram Bot API  

4. **Create Parse User Text Node (Code)**  
   - Type: Code  
   - Use the provided JavaScript code to parse commands (/start, /help, /add, /list, /delete, /search), extract parameters (symbol, interval, prompt), and validate format  
   - Connect output to Switch node  

5. **Create Switch Node**  
   - Type: Switch  
   - Add branches for commands: start, help, add, list, delete  
   - Connect each branch accordingly (see steps below)  

6. **Handle /start Command**  
   - Create Telegram node "Test return success"  
   - Text: "Hi ~ {{ $json.message.chat.username }}\nTest return Success !"  
   - Chat ID from Authorization node message  
   - Connect Switch "start" output to this node  

7. **Handle /help Command**  
   - Create Telegram node "Return help manu"  
   - Text lists all commands and usage  
   - Connect Switch "help" output to this node  

8. **Handle /add Command**  
   - Create Postgres node "Add Data"  
   - Query: Upsert symbol and user_prompt into watchdog table  
   - Credentials: PostgreSQL  
   - Connect Switch "add" output to this node  

   - Create Code node "Code" to generate new workflow JSON dynamically (copy provided JS)  
   - Connect Add Data output to Code node  

   - Create HTTP Request node "Create Workflow"  
   - POST to n8n local API: /api/v1/workflows with JSON body from Code node  
   - API Key authentication  
   - Connect Code node output to Create Workflow  

   - Create HTTP Request node "Activate"  
   - POST to /api/v1/workflows/{id}/activate  
   - Connect Create Workflow output to Activate  

   - Create Postgres node "Update workflow id"  
   - Update watchdog record with new workflow ID  
   - Connect Create Workflow output to Update workflow id  

   - Create Telegram node "WatchDog Success"  
   - Notify user of success with symbol name  
   - Connect Activate output to this node  

9. **Handle /list Command**  
   - Create Postgres node "Get list"  
   - Query: SELECT * FROM watchdog  
   - Connect Switch "list" output to this node  

   - Create Code node "Merge All list"  
   - Format rows into Markdown text with symbols and prompts  
   - Connect Get list output to Merge All list  

   - Create Telegram node "Send list"  
   - Send formatted list with MarkdownV2 formatting  
   - Connect Merge All list output to Send list  

10. **Handle /delete Command**  
    - Create Postgres node "Get workflow id"  
    - Query: SELECT * FROM watchdog WHERE symbol = '{{ $json.symbol }}'  
    - Connect Switch "delete" output to this node  

    - Create HTTP Request node "Delete the Workflow"  
    - DELETE /api/v1/workflows/{workflow_id}  
    - Connect Get workflow id output to Delete the Workflow  

    - Create Postgres node "Delete the row"  
    - Delete watchdog record by symbol  
    - Connect Delete the Workflow output to Delete the row  

    - Create Telegram node "WatchDog Success1"  
    - Notify user deletion success  
    - Connect Delete the row output to this node  

11. **Create Scheduled Sub-Workflow Template (for dynamic creation)**  
    - Schedule Trigger node with interval set dynamically based on user input  
    - Set node "Prompt" with AI prompt text and user prompt placeholder  
    - Set node "Symbol" with symbol variable  
    - Postgres node to retrieve user prompt by symbol  
    - HTTP Request node to retrieve chart image URL from Chart Image API  
    - Google Gemini AI node to analyze image with prompt  
    - Code node to parse Gemini AI JSON output with error handling  
    - Switch node to route based on `is_alert` value (Y/N)  
    - Telegram node to send alert text message in HTML format to channel/thread  
    - Telegram node to send photo message of chart image  

12. **Connect all nodes as per logical flow**  
    - Trigger -> Authorization -> Parse user text -> Switch -> command-specific nodes  

13. **Configure all credentials**  
    - Telegram Bot API credentials for all Telegram nodes  
    - PostgreSQL credentials  
    - HTTP Header Auth credentials for Chart Image API and n8n API  
    - Google Palm API credentials for Gemini AI node  

14. **Set appropriate retry and error handling options**  
    - Enable retry on fail for critical external API calls (HTTP Requests, AI node)  
    - Validate input data and handle missing or malformed commands gracefully  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses Gemini AI model "gemini-2.5-flash" for advanced image analysis.                                      | Google Gemini AI node configuration                                                                |
| Telegram Bot commands follow slash "/" format with strict parameter requirements to ensure parsing success.           | User command parsing in "Parse user text" node                                                     |
| The workflow dynamically creates and manages sub-workflows via n8n API on localhost, requiring API key permissions.  | HTTP Request nodes Create Workflow, Activate, Delete the Workflow                                  |
| PostgreSQL table "watchdog" schema includes fields: symbol (PK), user_prompt, workflow_id                            | Used for storing configurations and associating dynamic workflows                                  |
| Chart Image API requires HTTP header authentication and expects parameters "symbol" (prefixed with "BINANCE:") and "interval". | HTTP Request node "Get Image URL"                                                                    |
| Telegram messages for alerts use HTML parse mode and message threading within a specific channel                       | Telegram nodes Send a text message and Send a photo message                                        |
| The user prompt for AI analysis is a strict JSON output format with fields "is_alert" (Y/N) and "alert_content".      | AI prompt text constructed in "Prompt" node                                                       |
| The workflow currently authorizes a single Telegram user ID; extend Authorization node conditions for multiple users. | Authorization node                                                                                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.