Create Smart Telegram Reminders with GPT-4 Mini and Airtable Cancel Codes

https://n8nworkflows.xyz/workflows/create-smart-telegram-reminders-with-gpt-4-mini-and-airtable-cancel-codes-7921


# Create Smart Telegram Reminders with GPT-4 Mini and Airtable Cancel Codes

### 1. Workflow Overview

This workflow enables users to create, manage, and cancel personal reminders via Telegram, leveraging GPT-4 Mini for natural language understanding and Airtable as a backend storage system. Users send messages to a Telegram bot to set reminders in natural language or cancel existing reminders by replying with a unique cancellation code. The workflow includes:

- **1.1 Input Reception:** Telegram Trigger node listens for user messages.
- **1.2 Message Classification:** If the message is a numeric code, it is treated as a cancellation request; otherwise, it is processed as a new reminder.
- **1.3 AI Processing:** GPT-4 Mini interprets natural language reminder requests into structured JSON data.
- **1.4 Code Generation:** A unique 4-digit cancellation code is generated for each reminder.
- **1.5 Data Storage:** Reminders are stored in Airtable with relevant metadata.
- **1.6 Confirmation Messaging:** Users receive confirmation messages with reminder details and cancellation codes.
- **1.7 Cancellation Handling:** When a code is received, the workflow searches Airtable to delete the corresponding reminder and notifies the user accordingly.
- **1.8 Error Handling:** If a cancellation code is invalid or not found, the user is informed.

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Message Type Branching

**Overview:**  
Detects incoming Telegram messages and determines whether they represent a cancellation code or a new reminder request.

**Nodes Involved:**  
- Telegram Trigger  
- If (checks if message text is 4-6 digit code)  
- Parse the data (extracts code or cancellation intent)

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages to the bot.  
  - Configuration: Triggers on message updates only. Uses Telegram API credentials.  
  - Input: Incoming Telegram updates.  
  - Output: Message JSON including text, chat, and user info.  
  - Edge Cases: Telegram API downtime or webhook misconfiguration could prevent message receipt.

- **If (message is a numeric code)**  
  - Type: If  
  - Role: Determines if the message text matches a 4 to 6-digit number pattern (used as cancel code).  
  - Configuration: Condition checks regex `^\d{4,6}$` on the message text.  
  - Input: Telegram Trigger output.  
  - Output: Two branches — true (code), false (natural language).  
  - Edge Cases: Messages with numbers embedded in text may be misclassified if only the number is sent. Non-standard inputs or empty messages may cause false negatives.

- **Parse the data (extract code or yes-only cancel)**  
  - Type: Code  
  - Role: Extracts cancellation code from message text or detects plain "YES" reply.  
  - Configuration: JavaScript code uses regex to find 4-6 digit code and boolean flag for "yes" reply.  
  - Input: Output from If node (true branch).  
  - Output: JSON with `chat_id`, `raw` text, `code`, `has_code` boolean, and `yes_only` boolean.  
  - Edge Cases: Messages without a valid code set `code` to empty string; "YES" without code is detected separately.

---

#### 2.2 Cancellation Flow (Code Found Path)

**Overview:**  
Validates the cancellation code against stored reminders in Airtable and deletes the matching record if found; otherwise, informs the user no reminder matches that code.

**Nodes Involved:**  
- CHECK IF CODE IS THERE (If)  
- Search records (Airtable)  
- CODE FOUND? (If)  
- Delete a record (Airtable)  
- Merge (conditional branching)  
- SEND DELETED REMINDER (Telegram)  
- SEND CODE NOT FOUND (Telegram)

**Node Details:**

- **CHECK IF CODE IS THERE**  
  - Type: If  
  - Role: Checks if the parsed message contains a valid code (`has_code` is true).  
  - Input: Output from Parse the data.  
  - Output: True branch continues cancellation search, false branch ends this flow (implied elsewhere).  
  - Edge Cases: Missing or malformed code leads to false branch.

- **Search records**  
  - Type: Airtable Search  
  - Role: Searches Airtable for a non-acknowledged reminder matching the chat_id and code.  
  - Configuration: Filter formula with `AND({ack}=0, {chat_id}='...', {code}='...')`. Returns only 1 record.  
  - Input: CHECK IF CODE IS THERE output JSON with `chat_id` and `code`.  
  - Output: Either matching record or empty result.  
  - Edge Cases: Airtable API rate limits, invalid credentials, or missing base/table cause errors.

- **CODE FOUND?**  
  - Type: If  
  - Role: Checks if a matching record was found (non-null `id` field).  
  - Input: Search records output.  
  - Output:  
    - True: Proceed to delete record.  
    - False: Send “code not found” message.  
  - Edge Cases: Unexpected data format or empty results.

- **Delete a record**  
  - Type: Airtable Delete  
  - Role: Deletes the reminder record identified by its Airtable record ID.  
  - Input: CODE FOUND? True branch sends record ID.  
  - Output: Deleted record info.  
  - Edge Cases: Record missing at deletion time, Airtable API errors.

- **Merge**  
  - Type: Merge (Choose Branch mode)  
  - Role: Combines two branches for final notification: successful deletion or code not found.  
  - Input: One branch from Delete a record, one from CODE FOUND? False branch.  
  - Output: Single merged output to Telegram notification nodes.  
  - Edge Cases: Misaligned branches could cause output errors.

- **SEND DELETED REMINDER**  
  - Type: Telegram  
  - Role: Notifies user that the reminder was deleted, includes reminder title and code.  
  - Configuration: Text template with dynamic fields.  
  - Input: Merge output with deleted record data.  
  - Edge Cases: Telegram API errors, invalid chat IDs.

- **SEND CODE NOT FOUND**  
  - Type: Telegram  
  - Role: Notifies user that no reminder was found for the provided code.  
  - Configuration: Text message with code and chat_id from Parse the data node.  
  - Input: Merge output from CODE FOUND? false branch.  
  - Edge Cases: Same as above.

---

#### 2.3 New Reminder Creation Flow (Natural Language Input)

**Overview:**  
Processes natural language reminder requests by parsing them through GPT-4 Mini, generating a unique cancellation code, storing the reminder in Airtable, and confirming to the user.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- OpenAI Chat Model  
- Structured Output Parser  
- Code (custom code for code generation)  
- Create a record (Airtable)  
- Send a text message (Telegram)

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Sends prompt to GPT-4 Mini to interpret reminder input text into structured JSON according to a schema.  
  - Configuration: Constructs prompt embedding the entire user message text, chat_id, user_id, current UTC timestamp, and default timezone. Uses a system message instructing GPT to output only valid JSON per schema with no markdown or extra keys.  
  - Input: False branch output of If node (natural language message).  
  - Output: JSON object containing fields like chat_id, user_id, title, phone, dateText, tz, dueISO, and code (initially empty).  
  - Version: LangChain Agent v2.2, OpenAI model GPT-4.1-mini.  
  - Edge Cases: API rate limits, unexpected or ambiguous user input causing parsing failure or invalid JSON; network errors.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides underlying GPT-4.1-mini model for AI Agent.  
  - Configuration: No additional options set. Requires OpenAI API credentials.  
  - Input: AI Agent node.  
  - Edge Cases: API key issues, model unavailability.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates and parses AI output JSON against defined schema.  
  - Schema includes mandatory fields: chat_id, user_id, title, tz, code; optional fields: phone, dateText, dueISO.  
  - Input: AI Agent output.  
  - Output: Structured JSON confirmed to match schema or error if invalid.  
  - Edge Cases: Parsing errors if AI output is malformed.

- **Code (unique code generation)**  
  - Type: Code  
  - Role: Generates a 4-digit random numeric code to be used as cancellation code for the reminder.  
  - Configuration: JavaScript function generates a random integer between 1000 and 9999 as string. Comments suggest possibility to extend to 5-6 digits or check for collisions.  
  - Input: Output from AI Agent.  
  - Output: Same JSON enriched with field `code`.  
  - Edge Cases: Potential collisions if same code generated for multiple reminders; no current collision detection implemented.

- **Create a record (Airtable)**  
  - Type: Airtable Create Record  
  - Role: Inserts reminder data into Airtable table with fields: tz, ack (false), code, phone, title, due_at, chat_id, user_id, next_due_at.  
  - Configuration: Uses defined Airtable base and table IDs. Mapping mode is manual with field definitions.  
  - Input: Output from Code node.  
  - Output: Created record data including Airtable record ID.  
  - Edge Cases: Airtable API limits, invalid fields, credential expiration.

- **Send a text message (Telegram)**  
  - Type: Telegram  
  - Role: Confirms reminder creation to user with message including reminder title and cancellation code.  
  - Configuration: Text template: "✅ Reminder set: [title]. Reply [code] to stop the reminder."  
  - Input: Output from Create a record node.  
  - Edge Cases: Telegram API errors, invalid chat IDs.

---

#### 2.4 Informational and Setup Notes

**Overview:**  
Two sticky notes provide comprehensive explanations and setup instructions for users and maintainers.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1

**Node Details:**

- **Sticky Note**  
  - Content: Detailed explanation of the entire ReminderBot workflow including features, components, and high-level flow.  
  - Usage: For documentation and user understanding.

- **Sticky Note1**  
  - Content: Step-by-step setup instructions for Telegram bot, Airtable base configuration, OpenAI API key, and scheduler node setup.  
  - Usage: For admins to properly configure the environment.

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                                   | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                     |
|-------------------------|-----------------------------------|-------------------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                  | Listens for incoming Telegram messages           | -                      | If                      |                                                                                                |
| If                     | If                               | Checks if message is 4-6 digit numeric code      | Telegram Trigger       | Parse the data (true), AI Agent (false) |                                                                                                |
| Parse the data          | Code                             | Extracts cancellation code or yes-only reply     | If (true branch)       | CHECK IF CODE IS THERE   |                                                                                                |
| CHECK IF CODE IS THERE  | If                               | Determines if valid code is present               | Parse the data         | Search records (true)    |                                                                                                |
| Search records          | Airtable Search                  | Searches for reminder matching chat_id + code    | CHECK IF CODE IS THERE | CODE FOUND?              |                                                                                                |
| CODE FOUND?             | If                               | Checks if reminder record was found               | Search records         | Delete a record (true), SEND CODE NOT FOUND (false) |                                                                                                |
| Delete a record         | Airtable Delete                 | Deletes reminder record from Airtable             | CODE FOUND? (true)     | Merge                   |                                                                                                |
| Merge                   | Merge (Choose Branch)             | Merges cancellation success and failure branches | Delete a record, CODE FOUND? (false) | SEND DELETED REMINDER (true branch), SEND CODE NOT FOUND (false branch) |                                                                                                |
| SEND DELETED REMINDER   | Telegram                         | Sends confirmation of deleted reminder            | Merge                  | -                       |                                                                                                |
| SEND CODE NOT FOUND     | Telegram                         | Sends message when no matching reminder code found| Merge                  | -                       |                                                                                                |
| AI Agent                | LangChain Agent                  | Parses natural language reminder requests to JSON | If (false branch)      | Code                    |                                                                                                |
| OpenAI Chat Model       | LangChain OpenAI Chat Model     | Provides GPT-4.1-mini model for AI parsing        | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                                                |
| Structured Output Parser| LangChain Structured Output Parser | Validates AI output JSON to schema                | AI Agent (ai_outputParser) | AI Agent (ai_outputParser) |                                                                                                |
| Code                    | Code                             | Generates unique 4-digit cancellation code        | AI Agent               | Create a record          |                                                                                                |
| Create a record         | Airtable Create Record          | Stores reminder data in Airtable                   | Code                   | Send a text message      |                                                                                                |
| Send a text message     | Telegram                         | Sends confirmation message of reminder set        | Create a record         | -                       |                                                                                                |
| Sticky Note             | Sticky Note                      | Documentation: Workflow overview and features     | -                      | -                       | 🔔 ReminderBot with Telegram + Airtable ... (full content in node)                             |
| Sticky Note1            | Sticky Note                      | Documentation: Setup instructions                  | -                      | -                       | Setup Instructions ... (full content in node)                                                 |

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for “message” updates only.  
   - Set Telegram API credentials with your Telegram bot token.

2. **Add an If node for numeric code detection**  
   - Condition: Check if message text matches regex `^\d{4,6}$` (4 to 6 digits).  
   - Connect Telegram Trigger output to this If node.

3. **Add Code node to parse cancellation code or plain "YES"**  
   - JavaScript snippet extracts 4-6 digit code from message text; sets flags for presence of code and plain “YES” only.  
   - Connect If node’s true output to this node.

4. **Add If node "CHECK IF CODE IS THERE"**  
   - Condition: Check if parsed data’s `has_code` is true.  
   - Connect Parse the data node output to this node.

5. **Add Airtable Search node**  
   - Configure with your Airtable base and reminder table.  
   - Filter formula to search for record with `ack=0`, matching `chat_id` and `code` from parsed data.  
   - Limit results to 1.  
   - Connect CHECK IF CODE IS THERE true output to this node.

6. **Add If node "CODE FOUND?"**  
   - Condition: Check if Airtable search result contains a valid record ID.  
   - Connect Search records node output to this node.

7. **Add Airtable Delete node**  
   - Configure to delete record with ID from CODE FOUND? true branch.  
   - Connect CODE FOUND? true to this node.

8. **Add Merge node (Choose Branch mode)**  
   - Connect Delete a record node output to one input branch.  
   - Connect CODE FOUND? false output (no record found) to the other input branch.

9. **Add Telegram node "SEND DELETED REMINDER"**  
   - Configure text: `"🗑️ Deleted reminder: [title] (code [code])."` with dynamic fields from deleted record.  
   - Connect Merge node output branch for successful deletion.

10. **Add Telegram node "SEND CODE NOT FOUND"**  
    - Configure text: `"I couldn't find a pending reminder for code [code]"` with dynamic fields from parse node.  
    - Connect Merge node output branch for no record found.

11. **Add AI Agent node for natural language processing**  
    - Use LangChain Agent node.  
    - Set prompt to include user message text, chat_id, user_id, current UTC timestamp, default timezone (America/New_York).  
    - Set system message to instruct AI to output only valid JSON matching a predefined schema with fields: chat_id, user_id, title, phone, dateText, tz, dueISO, code.  
    - Connect If node false output (message is not a code) to AI Agent node.  
    - Configure with OpenAI Chat Model node using GPT-4.1-mini and OpenAI API credentials.

12. **Add Structured Output Parser node**  
    - Define manual JSON schema corresponding to expected AI output fields.  
    - Connect AI Agent output to this node.

13. **Add Code node to generate unique 4-digit cancellation code**  
    - JavaScript function generates random integer 1000-9999 as string and appends to output JSON.  
    - Connect Structured Output Parser output to this node.

14. **Add Airtable Create Record node**  
    - Configure with your Airtable base and table.  
    - Map output JSON fields to Airtable columns: tz, ack (false), code, phone, title, due_at (dueISO), chat_id, user_id, next_due_at (dueISO).  
    - Connect Code node output to Create a record node.  
    - Use Airtable Personal Access Token credentials.

15. **Add Telegram node "Send a text message"**  
    - Configure text: `"✅ Reminder set: [title]. Reply [code] to stop the reminder."` with dynamic fields from Airtable record.  
    - Connect Create a record output to this node.  
    - Use Telegram API credentials.

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 🔔 ReminderBot with Telegram + Airtable: turns Telegram messages into reminders, manages them with unique cancel codes. | Sticky Note node in workflow contains full feature and flow description.                                     |
| Setup Instructions: includes Telegram bot creation, Airtable base and table schema, OpenAI API key setup, and scheduler.| Sticky Note1 node in workflow details setup steps for proper environment configuration.                      |
| Airtable base URL: https://airtable.com/appAkS7G35AGYejcP                                                              | Found in Airtable nodes’ cachedResultUrl fields.                                                            |
| Telegram Bot creation via @BotFather and token usage instructions.                                                      | Provided in Sticky Note1 for Telegram API credential setup.                                                  |
| OpenAI GPT-4 Mini model used for natural language processing with LangChain integration.                               | Configured in OpenAI Chat Model node, requiring valid OpenAI API key credential.                             |

---

This detailed reference document enables advanced users and AI agents to fully understand, reproduce, and maintain the “Create Smart Telegram Reminders with GPT-4 Mini and Airtable Cancel Codes” workflow.