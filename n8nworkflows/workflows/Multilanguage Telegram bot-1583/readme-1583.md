Multilanguage Telegram bot

https://n8nworkflows.xyz/workflows/multilanguage-telegram-bot-1583


# Multilanguage Telegram bot

### 1. Workflow Overview

This workflow implements a multilanguage Telegram bot using n8n. It is designed to respond to Telegram messages in multiple languages dynamically, supporting seamless addition of new languages without modifying the core workflow logic.

The bot identifies the user’s language preference based on their Telegram language setting and stores user data in a NocoDB database, allowing personalized interactions. Due to API changes in NocoDB, some database operations are handled via HTTP Request nodes instead of native NocoDB nodes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Language Detection**: Captures incoming Telegram messages, extracts user chat ID and preferred language, and loads the bot’s multilingual dictionary.
- **1.2 User Verification and Database Interaction**: Checks if the user exists in the database and either adds a new user or updates existing user data with the latest language settings.
- **1.3 Command Processing and Response Generation**: Evaluates user commands (/start, /help, others) and sends appropriate multilingual responses.
- **1.4 Error Handling for Unknown Commands**: Provides default replies for unrecognized commands.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Language Detection

**Overview:**  
This block listens to incoming Telegram messages, extracts user identifiers and language preferences, and loads the multilingual message dictionary from the database.

**Nodes Involved:**  
- Telegram Trigger  
- chatID (Function)  
- LoadDictionary (NocoDB)  
- botmessages (Function)  

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point that listens to incoming Telegram messages (specifically "message" updates).  
  - *Config:* Uses Telegram API credentials.  
  - *Inputs:* External Telegram messages (webhook).  
  - *Outputs:* Message JSON including user info, chat ID, message text, etc.  
  - *Notes:* Requires Telegram Bot API credentials.  
  - *Failure Modes:* Invalid Telegram credentials, webhook misconfiguration, or Telegram downtime.

- **chatID (Function)**  
  - *Type:* Function Node  
  - *Role:* Extracts chat ID and determines user language; defaults to English if language unsupported.  
  - *Config:*  
    - Reads `language_code` from Telegram message data.  
    - Supports languages in `botlang` array (currently ["ru", "en"]).  
    - Sets `lang` to user language or defaults to "en".  
  - *Input:* Telegram message JSON.  
  - *Output:* JSON with `chatID` and `lang` properties.  
  - *Failure Modes:* Missing or malformed language code in Telegram data; fallback logic mitigates this.

- **LoadDictionary (NocoDB)**  
  - *Type:* NocoDB Node (Get All Records)  
  - *Role:* Loads all multilingual bot messages from the `botmessages` table in the NocoDB project.  
  - *Config:* Returns all records without filters.  
  - *Credentials:* NocoDB API credentials.  
  - *Failure Modes:* NocoDB API downtime, authentication errors, or schema changes.

- **botmessages (Function)**  
  - *Type:* Function Node  
  - *Role:* Transforms the array of dictionary entries into a key-value object indexed by message name for easy access.  
  - *Config:* Iterates over loaded dictionary items and maps each `botmessage` key to its JSON content.  
  - *Input:* Array of bot message items from NocoDB.  
  - *Output:* Object mapping message keys (e.g., "greeting", "help") to their localized content.  
  - *Failure Modes:* Empty or malformed dictionary data.

---

#### 2.2 User Verification and Database Interaction

**Overview:**  
This block verifies if the Telegram user is already registered in the database and either creates a new user record or updates the existing one with the current language.

**Nodes Involved:**  
- CheckUser (NocoDB)  
- Merge  
- Switch  
- New user? (Function)  
- IF  
- HTTP AddUser (HTTP Request)  
- HTTP UpdateUser (HTTP Request)  

**Node Details:**

- **CheckUser (NocoDB)**  
  - *Type:* NocoDB Node (Get All Records)  
  - *Role:* Queries the `TG_users` table for a user record matching the current `chatID`.  
  - *Config:* Uses a `where` filter to find the user by `TG_account_ID`.  
  - *Credentials:* NocoDB API credentials.  
  - *Output:* User record if found, empty if not.  
  - *Failure Modes:* Query failures, NocoDB API errors.

- **Merge**  
  - *Type:* Merge Node (Pass Through)  
  - *Role:* Synchronizes data streams from user check and message trigger.  
  - *Input:* Outputs from CheckUser node and Telegram Trigger node (via chatID and LoadDictionary).  
  - *Output:* Merged JSON for downstream processing.

- **Switch**  
  - *Type:* Switch Node  
  - *Role:* Routes flow based on Telegram message text content to handle commands: `/start`, `/help`, or others.  
  - *Config:* Checks if message text is exactly "/start" or "/help", else falls back to default path.  
  - *Input:* Merged message JSON.  
  - *Outputs:*  
    1. `/start` path  
    2. `/help` path  
    3. Default (wrong command) path.

- **New user? (Function)**  
  - *Type:* Function Node  
  - *Role:* Determines if the user is new by checking if the CheckUser result is empty.  
  - *Config:* Returns boolean `empty` indicating user absence.  
  - *Input:* Output from Switch node on `/start`.  
  - *Output:* JSON with `empty` boolean.

- **IF**  
  - *Type:* IF Node  
  - *Role:* Conditional branch on `empty` boolean from New user? node to decide between Add or Update user.  
  - *Output:*  
    - True: New user, proceed to AddUser.  
    - False: Existing user, proceed to UpdateUser.

- **HTTP AddUser (HTTP Request)**  
  - *Type:* HTTP Request Node  
  - *Role:* Creates new user record in `TG_users` table via NocoDB REST API.  
  - *Config:*  
    - POST method to `TG_users` endpoint.  
    - Body includes `TG_account_ID` and `Last_language_used`.  
  - *Credentials:* HTTP Header Auth with NocoDB API key.  
  - *Output:* Confirmation of user creation.  
  - *Failure Modes:* HTTP errors, authentication failure, invalid payload.

- **HTTP UpdateUser (HTTP Request)**  
  - *Type:* HTTP Request Node  
  - *Role:* Updates existing user record language in `TG_users` table.  
  - *Config:*  
    - PATCH method to specific user ID URL.  
    - Body updates `Last_language_used` and `TG_account_ID`.  
  - *Credentials:* HTTP Header Auth with NocoDB API key.  
  - *Output:* Confirmation of user update.  
  - *Failure Modes:* HTTP errors, invalid user ID, auth errors.

*Note:* NocoDB native AddUser and UpdateUser nodes are present but disabled due to API changes; HTTP Request nodes replace their functionality.

---

#### 2.3 Command Processing and Response Generation

**Overview:**  
This block sends multilingual responses to the user based on the command received and user status (new or returning).

**Nodes Involved:**  
- msg_greet (Telegram)  
- msg_welcomeback (Telegram)  
- msg_help (Telegram)  
- msg_wrongcommand (Telegram)  

**Node Details:**

- **msg_greet (Telegram)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends greeting message to new users after registration.  
  - *Config:*  
    - Text is dynamically selected from `botmessages.greeting` for user language.  
    - Chat ID from Telegram Trigger message.  
  - *Credentials:* Telegram API credentials.  
  - *Input:* Output from HTTP AddUser node.  
  - *Failure Modes:* Telegram API errors, invalid chat ID, missing dictionary entry.

- **msg_welcomeback (Telegram)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends welcome back message to returning users after update.  
  - *Config:*  
    - Text is selected from `botmessages.welcomeback` keyed by user language.  
    - Chat ID from Telegram Trigger message.  
  - *Credentials:* Telegram API credentials.  
  - *Input:* Output from HTTP UpdateUser node.  
  - *Failure Modes:* Same as msg_greet.

- **msg_help (Telegram)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends help information when user sends `/help` command.  
  - *Config:*  
    - Text from `botmessages.help`, per user language.  
    - Uses Markdown parse mode for formatting.  
  - *Credentials:* Telegram API credentials.  
  - *Input:* Output from Switch node on `/help` command.  
  - *Failure Modes:* Telegram API or formatting errors.

- **msg_wrongcommand (Telegram)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends message for unknown/unhandled commands.  
  - *Config:*  
    - Text from `botmessages.wrongcommand` for user language.  
  - *Credentials:* Telegram API credentials.  
  - *Input:* Default output from Switch node.  
  - *Failure Modes:* Telegram API errors.

---

#### 2.4 Error and Maintenance Notes

**Nodes Involved:**  
- Note (Sticky Note)  

**Details:**  
- Provides contextual explanation about the replacement of NocoDB native nodes with HTTP Request nodes due to API changes in May 2022.  
- Indicates the workflow’s adherence to functionality despite API disruptions.

---

### 3. Summary Table

| Node Name       | Node Type         | Functional Role                    | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                        |
|-----------------|-------------------|----------------------------------|--------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger  | Entry point for Telegram messages| -                        | chatID, Merge              |                                                                                                                    |
| chatID          | Function          | Extract chat ID and language     | Telegram Trigger         | LoadDictionary             |                                                                                                                    |
| LoadDictionary  | NocoDB            | Load multilingual dictionary     | chatID                   | botmessages                |                                                                                                                    |
| botmessages     | Function          | Transform dictionary data format | LoadDictionary           | CheckUser                  |                                                                                                                    |
| CheckUser       | NocoDB            | Check if user exists in DB       | botmessages              | Merge                      |                                                                                                                    |
| Merge           | Merge             | Synchronize data streams         | CheckUser, Telegram Trigger| Switch                   | Wait for dictionary to load                                                                                        |
| Switch          | Switch            | Route based on command text      | Merge                    | New user?, msg_help, msg_wrongcommand | Check bot commands                                                                                  |
| New user?       | Function          | Check if user is new             | Switch                   | IF                         |                                                                                                                    |
| IF              | IF                | Branch add or update user        | New user?                 | HTTP AddUser, HTTP UpdateUser|                                                                                                                    |
| HTTP AddUser    | HTTP Request      | Add new user to DB               | IF (true branch)         | msg_greet                  |                                                                                                                    |
| HTTP UpdateUser | HTTP Request      | Update existing user             | IF (false branch)        | msg_welcomeback            |                                                                                                                    |
| msg_greet       | Telegram          | Send greeting to new user        | HTTP AddUser             | -                          |                                                                                                                    |
| msg_welcomeback | Telegram          | Send welcome back message        | HTTP UpdateUser          | -                          |                                                                                                                    |
| msg_help        | Telegram          | Send help message                | Switch (/help path)      | -                          |                                                                                                                    |
| msg_wrongcommand| Telegram          | Send unknown command response    | Switch (default path)    | -                          |                                                                                                                    |
| Note            | Sticky Note       | Explain NocoDB node replacement  | -                        | -                          | ## What's this? Due to some breaking API changes in NocoDB some of its node options are not working at the moment (MAY 2022). These two nodes were replaced by HTTP request nodes. Functionality is still the same. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Trigger on "message" updates  
   - Credentials: Connect your Telegram bot API credentials  
   - Position: Entry point

2. **Create chatID Function Node**  
   - Type: Function  
   - Purpose: Extract user chat ID and language from Telegram message  
   - Code snippet:  
     ```javascript
     var data = $node["Telegram Trigger"].json;
     const botlang = ["ru", "en"];
     var curlang = botlang.includes(data.message.from.language_code) ? data.message.from.language_code : "en";
     return [{json: {chatID  : data.message.chat.id, lang : curlang}}];
     ```
   - Connect Telegram Trigger → chatID

3. **Create LoadDictionary NocoDB Node**  
   - Type: NocoDB (Get All Records)  
   - Parameters:  
     - Table: `botmessages`  
     - Return all records  
     - Project ID: your NocoDB project ID (e.g., "n8n_multilang_bot_wzhb")  
   - Credentials: NocoDB API auth  
   - Connect chatID → LoadDictionary

4. **Create botmessages Function Node**  
   - Type: Function  
   - Purpose: Convert dictionary array into key-value object  
   - Code snippet:  
     ```javascript
     let data = {};
     for (item of items) {
       data[item.json.botmessage] = item.json;
     }
     return [{json: data}];
     ```  
   - Connect LoadDictionary → botmessages

5. **Create CheckUser NocoDB Node**  
   - Type: NocoDB (Get All Records)  
   - Parameters:  
     - Table: `TG_users`  
     - Filter: `where=(TG_account_ID,eq,{{$node["chatID"].json["chatID"]}})`  
     - Project ID: your NocoDB project ID  
   - Credentials: NocoDB API auth  
   - Connect botmessages → CheckUser

6. **Create Merge Node**  
   - Type: Merge (Pass Through)  
   - Connect CheckUser → Merge (Input 2)  
   - Connect Telegram Trigger → Merge (Input 1)

7. **Create Switch Node**  
   - Type: Switch  
   - Parameter:  
     - Value1: `={{$node["Merge"].json["message"]["text"]}}`  
     - Rules: Check if equals `/start`, `/help`  
     - Fallback output: default (unknown command)  
   - Connect Merge → Switch

8. **Create New user? Function Node**  
   - Type: Function  
   - Purpose: Check if CheckUser returned empty result  
   - Code snippet:  
     ```javascript
     return [{json: {empty: Object.keys($node["CheckUser"].json).length == 0}}];
     ```  
   - Connect Switch (output for `/start` command) → New user?

9. **Create IF Node**  
   - Type: IF  
   - Condition: Check if `empty` is true (new user) or false (existing user)  
   - Connect New user? → IF

10. **Create HTTP AddUser Node**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `https://database.digigin.eu/api/v1/db/data/noco/n8n_multilang_bot_wzhb/TG_users`  
      - Method: POST  
      - Body Content Type: JSON  
      - Body Parameters:  
        - TG_account_ID = `{{$node["chatID"].json["chatID"]}}`  
        - Last_language_used = `{{$node["Telegram Trigger"].json["message"]["from"]["language_code"]}}`  
    - Authentication: HTTP Header Auth with NocoDB API key  
    - Connect IF (true) → HTTP AddUser

11. **Create HTTP UpdateUser Node**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `https://database.digigin.eu/api/v1/db/data/noco/n8n_multilang_bot_wzhb/TG_users/{{$node["CheckUser"].json["id"]}}`  
      - Method: PATCH  
      - Body Content Type: JSON  
      - Body Parameters:  
        - TG_account_ID = `{{$node["chatID"].json["chatID"]}}`  
        - Last_language_used = `{{$node["Telegram Trigger"].json["message"]["from"]["language_code"]}}`  
    - Authentication: HTTP Header Auth with NocoDB API key  
    - Connect IF (false) → HTTP UpdateUser

12. **Create msg_greet Telegram Node**  
    - Type: Telegram (Send Message)  
    - Text: `={{$evaluateExpression($node["botmessages"].json["greeting"][$node["chatID"].json["lang"]])}}`  
    - Chat ID: `={{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`  
    - Credentials: Telegram API  
    - Connect HTTP AddUser → msg_greet

13. **Create msg_welcomeback Telegram Node**  
    - Type: Telegram (Send Message)  
    - Text: `={{$evaluateExpression($node["botmessages"].json["welcomeback"][$node["chatID"].json["lang"]])}}`  
    - Chat ID: `={{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`  
    - Credentials: Telegram API  
    - Connect HTTP UpdateUser → msg_welcomeback

14. **Create msg_help Telegram Node**  
    - Type: Telegram (Send Message)  
    - Text: `={{$evaluateExpression($node["botmessages"].json["help"][$node["chatID"].json["lang"]])}}`  
    - Chat ID: `={{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`  
    - Additional Fields: parse_mode = Markdown  
    - Credentials: Telegram API  
    - Connect Switch (output `/help`) → msg_help

15. **Create msg_wrongcommand Telegram Node**  
    - Type: Telegram (Send Message)  
    - Text: `={{$evaluateExpression($node["botmessages"].json["wrongcommand"][$node["chatID"].json["lang"]])}}`  
    - Chat ID: `={{$node["Telegram Trigger"].json["message"]["chat"]["id"]}}`  
    - Credentials: Telegram API  
    - Connect Switch (default output) → msg_wrongcommand

16. **Add Sticky Note**  
    - Content: Note explaining replacement of NocoDB nodes by HTTP requests due to May 2022 API changes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                    |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------|
| Due to some breaking API changes in NocoDB (May 2022), native AddUser and UpdateUser nodes do not function properly. They were replaced by HTTP Request nodes. Functionality remains unchanged. | See Sticky Note in the workflow. |

---

This documentation provides a comprehensive understanding of the multilanguage Telegram bot workflow, enabling advanced users and AI agents to reproduce, modify, or troubleshoot the system effectively.