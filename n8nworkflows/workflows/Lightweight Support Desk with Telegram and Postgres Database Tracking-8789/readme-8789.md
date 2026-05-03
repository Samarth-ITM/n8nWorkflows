Lightweight Support Desk with Telegram and Postgres Database Tracking

https://n8nworkflows.xyz/workflows/lightweight-support-desk-with-telegram-and-postgres-database-tracking-8789


# Lightweight Support Desk with Telegram and Postgres Database Tracking

---
### 1. Workflow Overview

This workflow implements a **lightweight support desk system using Telegram as the user interface and PostgreSQL as the backend database**. Its purpose is to allow users to create, update, and query support tickets directly via Telegram bot commands. It is designed for small to medium support teams requiring a simple ticketing mechanism with database tracking and operator-controlled updates.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Command Routing:** Listens to Telegram bot messages and routes commands (/start, /new, /status, /update, /list) to appropriate processing branches.
- **1.2 Ticket Creation:** Parses new ticket details from freeform text, normalizes data, generates unique IDs, and upserts tickets into PostgreSQL.
- **1.3 Ticket Status Retrieval:** Parses status request commands, validates ticket IDs, fetches ticket status from the database, and replies to users.
- **1.4 Ticket Status Update:** Allows authorized operators to update ticket statuses, verifies permissions, updates DB, inserts audit logs, and notifies users.
- **1.5 Admin Commands:** Allows admin users to list recent tickets.
- **1.6 Error Handling & Notifications:** Manages invalid commands, missing or invalid inputs, DB errors, and notifies users or admins accordingly.
- **1.7 Setup & Documentation:** Sticky notes describing database schema, stored functions, credentials, and placeholders for customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Command Routing

- **Overview:**  
Listens for Telegram messages and routes commands to corresponding processing branches. Supports /start, /new, /status, /update, /list commands, with fallback for unrecognized commands.

- **Nodes Involved:**  
  - `01 Telegram Trigger: Intake + Status`  
  - `02 Switch: Route by Command`  
  - `Welcome Message`  
  - `Telegram: Invalid Command`  

- **Node Details:**

  - **01 Telegram Trigger: Intake + Status**  
    - *Type:* Telegram Trigger (Webhook)  
    - *Role:* Entry point, receives Telegram messages  
    - *Configuration:* Listens for "message" updates, uses Telegram Bot credentials named "Ticket Intake"  
    - *Input:* Telegram webhook POST calls  
    - *Output:* JSON with message data  
    - *Potential Failures:* Webhook misconfiguration, Telegram API downtime  

  - **02 Switch: Route by Command**  
    - *Type:* Switch node (Conditional routing)  
    - *Role:* Routes incoming messages based on command extracted from message text (first word, case-insensitive)  
    - *Configuration:* Outputs mapped to /start, /status, /new, /update, /list commands; fallback to "extra" (invalid command)  
    - *Expressions:* Command extracted via `{{$json["message"]["text"].split(" ")[0].toLowerCase()}}`  
    - *Input:* Messages from Telegram Trigger  
    - *Output:* Branches per command  
    - *Failures:* If message text missing or malformed, fallback route executes  

  - **Welcome Message**  
    - *Type:* Telegram node (send message)  
    - *Role:* Sends introductory help text for /start command  
    - *Configuration:* HTML parse mode, no attribution, message includes command usage and instructions  
    - *Input:* /start command branch from Switch  
    - *Output:* Telegram message sent  
    - *Failures:* Telegram API errors  

  - **Telegram: Invalid Command**  
    - *Type:* Telegram node (send message)  
    - *Role:* Replies when command is unrecognized (fallback)  
    - *Configuration:* Text prompts user with valid commands /new or /status <ID>  
    - *Input:* Fallback branch from Switch  
    - *Output:* Telegram message sent  

---

#### 1.2 Ticket Creation

- **Overview:**  
Parses freeform ticket information sent via /new command, extracts fields, generates UUID correlation ID and dedupe key, then inserts or updates the ticket record in the database.

- **Nodes Involved:**  
  - `03a FN: Normalize + Hash`  
  - `04a DB: Upsert Ticket`  
  - `05a Telegram Ack`  

- **Node Details:**

  - **03a FN: Normalize + Hash**  
    - *Type:* Code (Python)  
    - *Role:* Parses text for Name, Email, Phone, Subject, Description using regex; generates UUID and SHA256 hash for deduplication  
    - *Key Variables:*  
      - `requester_name`, `requester_email`, `requester_phone`, `subject`, `description` (default fallbacks)  
      - `correlation_id` (UUID)  
      - `dedupe_key` (hash of email + subject)  
      - `external_id` (Telegram chat ID)  
    - *Input:* Telegram message text from /new command branch  
    - *Output:* JSON with normalized fields for DB insertion  
    - *Failures:* Parsing errors if text format unexpected; missing fields defaulted  

  - **04a DB: Upsert Ticket**  
    - *Type:* Postgres node (execute query)  
    - *Role:* Calls stored function `upsert_ticket` to insert or update ticket record  
    - *Configuration:* SQL with parameters from parsed JSON fields; returns ticket ID and correlation ID  
    - *Input:* Parsed ticket data from previous node  
    - *Output:* DB response with ticket identifiers  
    - *Failures:* DB connection issues, missing stored function, data validation errors  

  - **05a Telegram Ack**  
    - *Type:* Telegram node (send message)  
    - *Role:* Sends acknowledgment with correlation ID to user after ticket creation  
    - *Configuration:* HTML format, includes correlation ID for user reference  
    - *Input:* Confirmation from DB upsert  
    - *Output:* Telegram message sent  

---

#### 1.3 Ticket Status Retrieval

- **Overview:**  
Handles /status commands by parsing the correlation ID, validating format, retrieving ticket info from DB, and sending status or error messages accordingly.

- **Nodes Involved:**  
  - `03b FN: Parse Status Command`  
  - `03b IF: Has Valid Correlation ID Format`  
  - `03b1 IF: Has Correlation ID`  
  - `04b DB: Get Ticket Status`  
  - `04b1 IF: No Ticket Found`  
  - `04b0 IF: DB Lookup Failed?`  
  - `04b1 IF: Ticket Belongs To User`  
  - `05b Telegram: Status Reply`  
  - `05b Telegram: Status Reply (Error)`  
  - `05b0 Telegram: Status DB Error`  
  - `05b1 Telegram: Unauthorized Status Check`  
  - `Send a text message2` (Invalid ID format message)  
  - `Send a text message1` (No ticket found message)  

- **Node Details:**

  - **03b FN: Parse Status Command**  
    - *Type:* Code (Python)  
    - *Role:* Extracts correlation ID from command arguments, checks for UUID regex format  
    - *Input:* Telegram message text from /status command branch  
    - *Output:* JSON with correlation_id and chat_id  
    - *Failures:* Missing or malformed ID yields null correlation_id  

  - **03b IF: Has Valid Correlation ID Format**  
    - *Type:* If node  
    - *Role:* Validates correlation ID against UUID regex pattern  
    - *Output:* Routes valid IDs forward, invalid IDs to error message  

  - **03b1 IF: Has Correlation ID**  
    - *Type:* If node  
    - *Role:* Checks if correlation_id is not empty  
    - *Output:* Continues to DB lookup if present, otherwise sends error message  

  - **04b DB: Get Ticket Status**  
    - *Type:* Postgres node  
    - *Role:* Selects ticket info by correlation ID  
    - *Output:* Ticket data including subject, status, timestamps, chat_id  
    - *Failures:* DB connectivity issues retried; returns empty if no ticket found  

  - **04b1 IF: No Ticket Found**  
    - *Type:* If node  
    - *Role:* Checks if ticket query returned empty (subject is empty)  
    - *Output:* Sends "no ticket" message or continues  

  - **04b0 IF: DB Lookup Failed?**  
    - *Type:* If node  
    - *Role:* Checks for error in DB query response  
    - *Output:* Sends error message or continues  

  - **04b1 IF: Ticket Belongs To User**  
    - *Type:* If node  
    - *Role:* Checks if requesting chat_id matches ticket owner chat_id  
    - *Output:* Sends ticket details or unauthorized message  

  - **05b Telegram: Status Reply**  
    - *Type:* Telegram node  
    - *Role:* Sends formatted ticket status message to user  
    - *Content:* Displays ticket ID, subject, status, created and updated timestamps in readable format  

  - **05b Telegram: Status Reply (Error)**  
    - *Type:* Telegram node  
    - *Role:* Sends message requesting correlation ID if missing  

  - **05b0 Telegram: Status DB Error**  
    - *Type:* Telegram node  
    - *Role:* Sends generic DB error message to user  

  - **05b1 Telegram: Unauthorized Status Check**  
    - *Type:* Telegram node  
    - *Role:* Alerts user if they try to access tickets they don’t own  

  - **Send a text message2**  
    - *Type:* Telegram node  
    - *Role:* Warns about invalid UUID format for correlation ID  

  - **Send a text message1**  
    - *Type:* Telegram node  
    - *Role:* Notifies user that no ticket was found for given ID  

---

#### 1.4 Ticket Status Update

- **Overview:**  
Allows authorized operator users to update ticket statuses via /update command. Validates operator ID, parses input, checks new status validity, updates ticket in DB, inserts audit log, and notifies ticket owner and operator.

- **Nodes Involved:**  
  - `03c FN: Parse Update Command`  
  - `03c0 IF: Is Operator`  
  - `03c1 IF: Has Correlation ID`  
  - `03c2 IF: Valid Status`  
  - `04c DB: Update Ticket Status`  
  - `04c2 DB: Insert Audit Row`  
  - `04c1 DB: Get Ticket Owner`  
  - `04c1a IF: Resolved or In Progress`  
  - `05c Telegram: Update Confirmation`  
  - `05c Telegram: Invalid Status`  
  - `Send a text message` (Invalid or missing correlation ID message)  
  - `05c0 Telegram: Unauthorized Update Attempt`  
  - `05c0 IF: Operator Reply Failed?`  
  - `Telegram: Admin Alert — Operator Reply Failed`  
  - `05c1a Telegram: Notify Resolved`  
  - `05c1b Telegram: Notify In Progress`  
  - `Notify Failed?`  
  - `Send a text message3` (Unauthorized command message)  

- **Node Details:**

  - **03c FN: Parse Update Command**  
    - *Type:* Code (Python)  
    - *Role:* Extracts correlation ID and new status from /update command arguments, verifies UUID format  
    - *Input:* Telegram message text from /update command branch  
    - *Output:* JSON with correlation_id, new_status, chat_id  

  - **03c0 IF: Is Operator**  
    - *Type:* If node  
    - *Role:* Checks if the sender's chat_id matches the configured operator Telegram ID (replace placeholder `YOUR_OPERATOR_ID`)  
    - *Output:* Authorized operators proceed, others receive unauthorized message  

  - **03c1 IF: Has Correlation ID**  
    - *Type:* If node  
    - *Role:* Validates presence of correlation ID  
    - *Output:* Continues if present, else sends invalid ID message  

  - **03c2 IF: Valid Status**  
    - *Type:* If node  
    - *Role:* Checks if new status is one of allowed values: `new`, `in_progress`, `resolved`  
    - *Output:* Proceeds to update DB if valid, else sends invalid status message  

  - **04c DB: Update Ticket Status**  
    - *Type:* Postgres node  
    - *Role:* Updates ticket status and updated_at timestamp in DB by correlation ID  
    - *Output:* Updated ticket info  

  - **04c2 DB: Insert Audit Row**  
    - *Type:* Postgres node  
    - *Role:* Inserts audit trail record for the update action with actor chat ID  
    - *Requires:* `ticket_audit` table  

  - **04c1 DB: Get Ticket Owner**  
    - *Type:* Postgres node  
    - *Role:* Retrieves ticket owner chat_id and status for notification purposes  

  - **04c1a IF: Resolved or In Progress**  
    - *Type:* If node  
    - *Role:* Checks if new status is "resolved" or "in_progress" to trigger user notifications  

  - **05c Telegram: Update Confirmation**  
    - *Type:* Telegram node  
    - *Role:* Confirms ticket update to the operator with status and timestamp  

  - **05c Telegram: Invalid Status**  
    - *Type:* Telegram node  
    - *Role:* Notifies operator of invalid status value  

  - **Send a text message**  
    - *Type:* Telegram node  
    - *Role:* Alerts about invalid or missing correlation ID in update command  

  - **05c0 Telegram: Unauthorized Update Attempt**  
    - *Type:* Telegram node  
    - *Role:* Warns user they lack permission to update tickets  

  - **05c0 IF: Operator Reply Failed?**  
    - *Type:* If node  
    - *Role:* Detects if sending confirmation message failed, triggers admin alert  

  - **Telegram: Admin Alert — Operator Reply Failed**  
    - *Type:* Telegram node  
    - *Role:* Notifies admin if operator reply to update fails  

  - **05c1a Telegram: Notify Resolved**  
    - *Type:* Telegram node  
    - *Role:* Sends user notification when ticket is marked resolved  

  - **05c1b Telegram: Notify In Progress**  
    - *Type:* Telegram node  
    - *Role:* Sends user notification when ticket status changes to in_progress  

  - **Notify Failed?**  
    - *Type:* If node  
    - *Role:* Checks if notification sending failed and logs errors  

  - **Send a text message3**  
    - *Type:* Telegram node  
    - *Role:* Sends unauthorized command message for non-operators trying /update  

---

#### 1.5 Admin Commands

- **Overview:**  
Enables an admin user to list recent tickets with limited metadata.

- **Nodes Involved:**  
  - `Check Admin`  
  - `DB: List Tickets`  
  - `Code in JavaScript`  
  - `Send a text message4`  
  - `Send a text message3` (Unauthorized message)  

- **Node Details:**

  - **Check Admin**  
    - *Type:* If node  
    - *Role:* Checks if sender's Telegram ID matches configured admin ID (`YOUR_ADMIN_ID`)  
    - *Output:* Admins proceed to list tickets; others receive unauthorized message  

  - **DB: List Tickets**  
    - *Type:* Postgres node  
    - *Role:* Queries last 10 tickets ordered by creation date descending  

  - **Code in JavaScript**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Formats ticket list into a user-friendly Telegram message with IDs, status, subjects, and creation dates  

  - **Send a text message4**  
    - *Type:* Telegram node  
    - *Role:* Sends the formatted ticket list to admin user  

  - **Send a text message3**  
    - *Type:* Telegram node  
    - *Role:* Sends unauthorized message if user is not admin  

---

#### 1.6 Error Handling & Notifications

- **Overview:**  
Handles error conditions such as database failures, invalid commands, unauthorized access, and logs workflow errors into a database table.

- **Nodes Involved:**  
  - `Notify Failed?`  
  - `Execute a SQL query` (Insert into workflow_errors)  
  - Various Telegram error message nodes (described above)  

- **Node Details:**

  - **Notify Failed?**  
    - *Type:* If node  
    - *Role:* Detects if notification sending failed (e.g., Telegram API failure)  

  - **Execute a SQL query**  
    - *Type:* Postgres node  
    - *Role:* Inserts error details including workflow ID, execution ID, error message, and JSON payload into `workflow_errors` table for monitoring  

---

#### 1.7 Setup & Documentation

- **Overview:**  
Provides setup instructions, database schema details, stored function code, and placeholders for IDs for easy configuration.

- **Nodes Involved:**  
  - `Sticky Note` (Setup Requirements)  
  - `Sticky Note1` (upsert_ticket function SQL)  

- **Node Details:**

  - **Sticky Note**  
    - Describes required database tables (`tickets`, `ticket_audit`, `workflow_errors`)  
    - Describes the `upsert_ticket` stored function inputs and outputs  
    - Lists required credentials setup (Postgres, Telegram)  
    - Mentions placeholders `YOUR_ADMIN_ID` and `YOUR_OPERATOR_ID` to replace with actual Telegram user IDs  
    - Provides link to Telegram user ID bot for lookup  

  - **Sticky Note1**  
    - Contains full SQL code for `upsert_ticket` function, with explanation of its purpose and return values  

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                         | Input Node(s)                            | Output Node(s)                          | Sticky Note                                                                                                      |
|---------------------------------|---------------------|---------------------------------------|-----------------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 01 Telegram Trigger: Intake + Status | Telegram Trigger    | Entry point, receive Telegram messages | -                                       | 02 Switch: Route by Command             |                                                                                                                  |
| 02 Switch: Route by Command      | Switch              | Route commands to processing branches | 01 Telegram Trigger                      | Welcome Message, 03b FN: Parse Status Command, 03a FN: Normalize + Hash, 03c FN: Parse Update Command, Check Admin, Telegram: Invalid Command |                                                                                                                  |
| Welcome Message                 | Telegram            | Send /start help message               | 02 Switch                               | -                                      |                                                                                                                  |
| Telegram: Invalid Command       | Telegram            | Send message for unrecognized commands | 02 Switch                               | -                                      |                                                                                                                  |
| 03a FN: Normalize + Hash        | Code (Python)       | Parse /new ticket info, generate IDs  | 02 Switch (/new branch)                  | 04a DB: Upsert Ticket                   |                                                                                                                  |
| 04a DB: Upsert Ticket           | Postgres            | Insert or update ticket in DB          | 03a FN: Normalize + Hash                 | 05a Telegram Ack                        |                                                                                                                  |
| 05a Telegram Ack                | Telegram            | Confirm ticket creation to user        | 04a DB: Upsert Ticket                    | -                                      |                                                                                                                  |
| 03b FN: Parse Status Command    | Code (Python)       | Parse /status command arguments        | 02 Switch (/status branch)               | 03b IF: Has Valid Correlation ID Format |                                                                                                                  |
| 03b IF: Has Valid Correlation ID Format | If              | Validate UUID format                    | 03b FN: Parse Status Command             | 03b1 IF: Has Correlation ID, Send a text message2 |                                                                                                                  |
| 03b1 IF: Has Correlation ID     | If                  | Check correlation ID presence          | 03b IF: Has Valid Correlation ID Format  | 04b DB: Get Ticket Status, 05b Telegram: Status Reply (Error) |                                                                                                                  |
| 04b DB: Get Ticket Status       | Postgres            | Fetch ticket info by correlation ID    | 03b1 IF: Has Correlation ID              | 04b1 IF: No Ticket Found                |                                                                                                                  |
| 04b1 IF: No Ticket Found        | If                  | Check if ticket exists                  | 04b DB: Get Ticket Status                | Send a text message1, 04b0 IF: DB Lookup Failed? |                                                                                                                  |
| 04b0 IF: DB Lookup Failed?      | If                  | Check for DB error                      | 04b1 IF: No Ticket Found                 | 05b0 Telegram: Status DB Error, 04b1 IF: Ticket Belongs To User |                                                                                                                  |
| 04b1 IF: Ticket Belongs To User | If                  | Check if requester owns ticket          | 04b0 IF: DB Lookup Failed?               | 05b Telegram: Status Reply, 05b1 Telegram: Unauthorized Status Check |                                                                                                                  |
| 05b Telegram: Status Reply      | Telegram            | Send ticket status to user              | 04b1 IF: Ticket Belongs To User          | -                                      |                                                                                                                  |
| 05b Telegram: Status Reply (Error) | Telegram          | Ask for correlation ID if missing      | 03b1 IF: Has Correlation ID (false branch) | -                                    |                                                                                                                  |
| 05b0 Telegram: Status DB Error  | Telegram            | Notify DB error during status fetch    | 04b0 IF: DB Lookup Failed?                | -                                      |                                                                                                                  |
| 05b1 Telegram: Unauthorized Status Check | Telegram       | Notify unauthorized ticket status access | 04b1 IF: Ticket Belongs To User (false) | -                                     |                                                                                                                  |
| Send a text message2            | Telegram            | Notify invalid correlation ID format   | 03b IF: Has Valid Correlation ID Format (false) | -                                  |                                                                                                                  |
| Send a text message1            | Telegram            | Notify no ticket found                  | 04b1 IF: No Ticket Found (true)          | -                                      |                                                                                                                  |
| 03c FN: Parse Update Command    | Code (Python)       | Parse /update command arguments         | 02 Switch (/update branch)                | 03c0 IF: Is Operator                   |                                                                                                                  |
| 03c0 IF: Is Operator            | If                  | Check if sender is authorized operator | 03c FN: Parse Update Command              | 03c1 IF: Has Correlation ID, 05c0 Telegram: Unauthorized Update Attempt |                                                                                                                  |
| 03c1 IF: Has Correlation ID     | If                  | Check correlation ID presence          | 03c0 IF: Is Operator                      | 03c2 IF: Valid Status, Send a text message |                                                                                                                  |
| 03c2 IF: Valid Status           | If                  | Validate new status value               | 03c1 IF: Has Correlation ID               | 04c DB: Update Ticket Status, 05c Telegram: Invalid Status |                                                                                                                  |
| 04c DB: Update Ticket Status    | Postgres            | Update ticket status in DB              | 03c2 IF: Valid Status                     | 04c2 DB: Insert Audit Row               |                                                                                                                  |
| 04c2 DB: Insert Audit Row       | Postgres            | Insert audit log for the update        | 04c DB: Update Ticket Status              | 04c1 DB: Get Ticket Owner               |                                                                                                                  |
| 04c1 DB: Get Ticket Owner       | Postgres            | Fetch ticket owner info                  | 04c2 DB: Insert Audit Row                 | 04c1a IF: Resolved or In Progress, 05c Telegram: Update Confirmation |                                                                                                                  |
| 04c1a IF: Resolved or In Progress | If                | Check if status is resolved/in_progress | 04c1 DB: Get Ticket Owner                  | 05c1a Telegram: Notify Resolved, 05c1b Telegram: Notify In Progress |                                                                                                                  |
| 05c Telegram: Update Confirmation | Telegram          | Confirm update to operator              | 04c1 DB: Get Ticket Owner                  | 05c0 IF: Operator Reply Failed?         |                                                                                                                  |
| 05c Telegram: Invalid Status    | Telegram            | Notify invalid status value             | 03c2 IF: Valid Status (false)              | -                                      |                                                                                                                  |
| Send a text message             | Telegram            | Notify invalid or missing correlation ID | 03c1 IF: Has Correlation ID (false)      | -                                      |                                                                                                                  |
| 05c0 Telegram: Unauthorized Update Attempt | Telegram   | Notify unauthorized update attempt      | 03c0 IF: Is Operator (false)               | -                                      |                                                                                                                  |
| 05c0 IF: Operator Reply Failed? | If                 | Detect failure sending confirmation     | 05c Telegram: Update Confirmation          | Telegram: Admin Alert — Operator Reply Failed |                                                                                                                  |
| Telegram: Admin Alert — Operator Reply Failed | Telegram | Alert admin if operator reply fails     | 05c0 IF: Operator Reply Failed?             | -                                      |                                                                                                                  |
| 05c1a Telegram: Notify Resolved | Telegram            | Notify user ticket resolved             | 04c1a IF: Resolved or In Progress           | Notify Failed?                        |                                                                                                                  |
| 05c1b Telegram: Notify In Progress | Telegram         | Notify user ticket in progress          | 04c1a IF: Resolved or In Progress           | Notify Failed?                        |                                                                                                                  |
| Notify Failed?                 | If                  | Check notification failure              | 05c1a Telegram: Notify Resolved, 05c1b Telegram: Notify In Progress | Execute a SQL query             |                                                                                                                  |
| Execute a SQL query            | Postgres            | Log workflow errors into DB             | Notify Failed?                            | -                                      |                                                                                                                  |
| Check Admin                   | If                  | Check if sender is admin                 | 02 Switch (/list branch)                   | DB: List Tickets, Send a text message3 |                                                                                                                  |
| DB: List Tickets              | Postgres            | Query last 10 tickets                    | Check Admin                               | Code in JavaScript                     |                                                                                                                  |
| Code in JavaScript            | Code (JavaScript)   | Format ticket list for Telegram          | DB: List Tickets                          | Send a text message4                   |                                                                                                                  |
| Send a text message4          | Telegram            | Send ticket list to admin                | Code in JavaScript                        | -                                      |                                                                                                                  |
| Send a text message3          | Telegram            | Unauthorized message (non-admin or non-operator) | Check Admin, 03c0 IF: Is Operator (false) | -                                      |                                                                                                                  |
| Sticky Note                  | Sticky Note         | Setup instructions & DB schema           | -                                       | -                                      | Contains setup requirements including DB tables, stored function, credentials, and placeholders for IDs         |
| Sticky Note1                 | Sticky Note         | SQL for upsert_ticket function            | -                                       | -                                      | Contains full SQL code for stored function `upsert_ticket`                                                      |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Credentials**  
- Add Telegram Bot credentials with bot token under name "Ticket Intake".  
- Add PostgreSQL credentials matching your support DB under name "Postgres account".  

**Step 2: Create Telegram Trigger**  
- Node: `Telegram Trigger`  
- Parameters: listen for "message" updates only  
- Credentials: Telegram Bot "Ticket Intake"  
- Position: Entry node  
- Output: pass JSON message data  

**Step 3: Add Switch Node to Route Commands**  
- Node: `Switch`  
- Condition: extract first word of message text, lowercased  
- Outputs: `/start`, `/new`, `/status`, `/update`, `/list`, fallback "extra"  
- Connect Telegram Trigger output to Switch input  

**Step 4: Handle /start Command**  
- Node: Telegram (send message)  
- Text: Welcome message with instructions and command usage, HTML parse mode  
- Connect Switch `/start` output to this node  

**Step 5: Handle /new Command - Parse Ticket Data**  
- Node: Code (Python)  
- Parse Name, Email, Phone, Subject, Description from freeform text with regex  
- Generate `correlation_id` (UUID) and `dedupe_key` (SHA256 of email+subject)  
- Extract Telegram chat ID as `external_id`  
- Connect Switch `/new` output to this node  

**Step 6: Insert or Update Ticket in DB**  
- Node: Postgres Execute Query  
- Query: Call stored function `upsert_ticket` with parameters from parsed JSON fields  
- Connect code node output to this DB node  

**Step 7: Acknowledge Ticket Creation**  
- Node: Telegram (send message)  
- Message: Confirm ticket received, include correlation ID (HTML format)  
- Connect DB upsert output to this node  

**Step 8: Handle /status Command - Parse Correlation ID**  
- Node: Code (Python)  
- Extract correlation ID argument from message; validate UUID format with regex  
- Connect Switch `/status` output to this node  

**Step 9: Validate Correlation ID Format**  
- Node: If node  
- Condition: correlation_id matches UUID regex  
- Connect code node output to this node  

**Step 10: Check Correlation ID Presence**  
- Node: If node  
- Condition: correlation_id is not empty  
- Connect valid format output to this node  

**Step 11: Query Ticket Status from DB**  
- Node: Postgres Query  
- Query: Select subject, status, timestamps, chat_id from tickets where correlation_id = $1  
- Parameter: correlation_id from JSON  
- Connect If node "has correlation ID" true output here  

**Step 12: Check for Ticket Existence**  
- Node: If node  
- Condition: subject field empty or not (indicating no ticket found)  
- Connect DB output here  

**Step 13: Handle DB Lookup Failure**  
- Node: If node  
- Condition: Check for error field in DB response  
- Connect "no ticket found" false branch here  

**Step 14: Validate Requester Ownership**  
- Node: If node  
- Condition: ticket chat_id equals Telegram message chat_id (requester)  
- Connect DB lookup success branch here  

**Step 15: Send Ticket Status Reply**  
- Node: Telegram (send message)  
- Format message with ticket ID, subject, status, created and updated timestamps (formatted)  
- Connect ownership validated branch here  

**Step 16: Send Error Messages**  
- Nodes: Telegram nodes for missing correlation ID, invalid format, no ticket found, unauthorized access, and DB errors  
- Connect respective false branches from previous If nodes  

**Step 17: Handle /update Command - Parse Arguments**  
- Node: Code (Python)  
- Extract correlation ID and new status argument, validate correlation ID UUID format  
- Connect Switch `/update` output here  

**Step 18: Check Operator Authorization**  
- Node: If node  
- Condition: Telegram user ID equals configured `YOUR_OPERATOR_ID` (replace placeholder)  
- Connect parse update output here  

**Step 19: Check Correlation ID Presence for Update**  
- Node: If node  
- Condition: correlation ID not empty  
- Connect authorized operator branch here  

**Step 20: Validate Status Value**  
- Node: If node  
- Condition: new_status is one of `new`, `in_progress`, `resolved`  
- Connect correlation ID present branch here  

**Step 21: Update Ticket Status in DB**  
- Node: Postgres Update Query  
- Update tickets set status, updated_at where correlation_id = $1  
- Return updated ticket info  
- Connect valid status branch here  

**Step 22: Insert Audit Log**  
- Node: Postgres Insert Query  
- Insert into ticket_audit with ticket_id, correlation_id, action='update', new_status, actor_chat_id  
- Connect update ticket status output here  

**Step 23: Retrieve Ticket Owner for Notification**  
- Node: Postgres Query  
- Select chat_id, correlation_id, status, subject from tickets where correlation_id = $1  
- Connect audit insert output here  

**Step 24: Check Status for Notification**  
- Node: If node  
- Condition: status equals `resolved` or `in_progress`  
- Connect ticket owner query output here  

**Step 25: Send Update Confirmation to Operator**  
- Node: Telegram (send message)  
- Confirm ticket updated with status and updated timestamp (HTML)  
- Connect ticket owner query output here  

**Step 26: Notify Ticket Owner**  
- Nodes: Telegram nodes to notify resolved or in-progress status  
- Connect If node branches accordingly  

**Step 27: Handle Invalid Status or Missing IDs**  
- Nodes: Telegram nodes for invalid status message, missing/invalid correlation ID, unauthorized update attempts  
- Connect respective false branches  

**Step 28: Detect and Handle Notification Failures**  
- Node: If node  
- Check for errors in sending Telegram messages  
- On failure, insert workflow error record to `workflow_errors` table with details  

**Step 29: Handle /list Command for Admins**  
- Node: If node  
- Condition: Telegram user ID equals `YOUR_ADMIN_ID` (replace placeholder)  
- Connect Switch `/list` output here  

**Step 30: Query Recent Tickets**  
- Node: Postgres Query  
- Select last 10 tickets ordered by created_at desc  
- Connect admin check true branch here  

**Step 31: Format Ticket List**  
- Node: Code (JavaScript)  
- Format result rows into Telegram message text listing ticket IDs, status, subject, creation dates  
- Connect DB list output here  

**Step 32: Send Ticket List to Admin**  
- Node: Telegram (send message)  
- Connect formatting node output here  

**Step 33: Unauthorized Messages**  
- Nodes: Telegram nodes to notify unauthorized users trying /update or /list  
- Connect respective false branches from authorization checks  

**Step 34: Setup Sticky Notes**  
- Create sticky notes with:  
  - Database schema for `tickets`, `ticket_audit`, `workflow_errors` tables  
  - SQL code for `upsert_ticket` stored function  
  - Instructions to replace placeholders `YOUR_ADMIN_ID`, `YOUR_OPERATOR_ID` with actual Telegram IDs  
  - Credential setup instructions for Telegram and Postgres  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Use [@userinfobot](https://t.me/userinfobot) on Telegram to find your user ID for setting `YOUR_ADMIN_ID` and `YOUR_OPERATOR_ID`.                                                                                                                                                                  | Setup Instructions                                                                                     |
| The database requires three tables: `tickets`, `ticket_audit`, and `workflow_errors` with specified key columns to track ticket info, audit trails, and workflow errors respectively.                                                                                                             | Setup Requirements (Sticky Note)                                                                       |
| The stored function `upsert_ticket` must be created in PostgreSQL to handle insert or update logic for tickets by correlation ID. Example SQL is provided in the sticky note.                                                                                                                       | Setup Requirements (Sticky Note1)                                                                      |
| Telegram Bot API credentials are required, along with Postgres DB credentials configured in n8n with matching access permissions to read/write ticket data.                                                                                                                                       | Credentials Setup                                                                                       |
| Operators and Admin users are controlled by Telegram user ID checks embedded in If nodes; replace placeholders to secure access to sensitive commands like ticket updates and listing.                                                                                                              | Security / Access Control                                                                               |
| The workflow includes retry logic on DB queries and error handling nodes to manage failures gracefully and notify users or admins as appropriate.                                                                                                                                                 | Robustness & Error Handling                                                                             |
| Date/time formatting in Telegram messages uses JavaScript's `toLocaleString` with `"en-GB"` locale for readable timestamps.                                                                                                                                                                      | Localization Note                                                                                       |

---

This detailed documentation should empower developers and AI agents to understand, reproduce, and extend the Lightweight Support Desk workflow efficiently and reliably.