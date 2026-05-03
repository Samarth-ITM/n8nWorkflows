Export AI Agent Conversation Logs from Postgres to Google Sheets

https://n8nworkflows.xyz/workflows/export-ai-agent-conversation-logs-from-postgres-to-google-sheets-4464


# Export AI Agent Conversation Logs from Postgres to Google Sheets

### 1. Workflow Overview

This n8n workflow exports AI Agent conversation logs stored in a Postgres database to Google Sheets, creating a dedicated sheet tab for each distinct chat session. It is designed for product teams and developers who manage AI chat agents and want to review or analyze conversation history collaboratively in Google Sheets.

The workflow is structured into the following logical blocks:

- **1.1 Trigger Block:** Starts the workflow manually or on a schedule.
- **1.2 Session Retrieval Block:** Queries distinct `session_id`s from the Postgres chat history table.
- **1.3 Session Loop Block:** Iterates over each session ID to process conversations individually.
- **1.4 Google Sheets Preparation Block:** Clears existing sheet content, duplicates a template sheet, and renames it to match the current session ID.
- **1.5 Conversations Retrieval Block:** Fetches all messages for the current session from Postgres.
- **1.6 Conversation Logging Block:** Appends the session’s messages into the corresponding Google Sheets tab with structured columns.
- **1.7 Setup and Schema Initialization Block:** (Optional) Adds a `created_at` timestamp column to the Postgres table if it does not exist.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:** Initiates the workflow either manually via a button or automatically on a schedule.
- **Nodes Involved:** 
  - `When clicking ‘Test workflow’`
  - `Schedule Trigger`
- **Node Details:**

  - **When clicking ‘Test workflow’**
    - Type: Manual Trigger
    - Role: Allows manual execution of the workflow.
    - Inputs: None
    - Outputs: Triggers the next node to start the workflow.
    - Edge cases: None.
  
  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Automatically triggers the workflow daily at 12:00 PM.
    - Parameters: Runs once daily at hour 12.
    - Inputs: None
    - Outputs: Starts the workflow execution.
    - Edge cases: Timezone considerations; ensure server timezone matches expectations.

#### 1.2 Session Retrieval Block

- **Overview:** Queries the Postgres database to retrieve distinct chat session IDs.
- **Nodes Involved:** 
  - `Postgres - Get session ids`
- **Node Details:**

  - **Postgres - Get session ids**
    - Type: Postgres node (executeQuery operation)
    - Role: Executes SQL: `SELECT DISTINCT(session_id) FROM n8n_chat_histories`
    - Inputs: Trigger input from manual or scheduled trigger.
    - Outputs: List of unique `session_id`s for further processing.
    - Credentials: Uses configured Postgres credentials (`PG - Chat Memory POC`).
    - Edge cases: Database connection errors, empty table, permission issues.
    - Notes: Replace table name if different.

#### 1.3 Session Loop Block

- **Overview:** Iterates over each retrieved session ID to process conversations per session individually.
- **Nodes Involved:** 
  - `Loop Over Session IDs`
- **Node Details:**

  - **Loop Over Session IDs**
    - Type: SplitInBatches
    - Role: Iterates over each session ID one by one.
    - Inputs: List of session IDs from `Postgres - Get session ids`.
    - Outputs: Single session ID per iteration to downstream nodes.
    - Edge cases: Large number of sessions could impact performance or rate limits.

#### 1.4 Google Sheets Preparation Block

- **Overview:** Prepares a dedicated Google Sheets tab for each session by clearing it if existing, duplicating a template tab, and renaming the new tab.
- **Nodes Involved:** 
  - `Clear Sheet Content`
  - `Duplicate template sheet`
  - `Rename Sheet`
  - `Set session_id`
- **Node Details:**

  - **Clear Sheet Content**
    - Type: Google Sheets node (clear operation)
    - Role: Clears rows A2:C10000 in the sheet named after the current session ID.
    - Inputs: Session ID from loop.
    - Parameters: Clear range A2:C10000 on sheet named `={{ $json.session_id }}` in the specified Google Sheets document.
    - Credentials: Google Sheets OAuth2 credentials.
    - Edge cases: If the sheet does not exist, node fails silently and continues.
  
  - **Duplicate template sheet**
    - Type: HTTP Request node (Google Sheets API)
    - Role: Copies the first sheet (index 0) in the Google Sheets document to create a blank template for the session.
    - Inputs: After clearing sheet content.
    - Parameters: POST to `https://sheets.googleapis.com/v4/spreadsheets/{documentId}/sheets/0:copyTo` with body containing destination spreadsheet ID.
    - Credentials: Google Sheets OAuth2.
    - Edge cases: API rate limits, permission errors.
  
  - **Rename Sheet**
    - Type: HTTP Request node (Google Sheets API)
    - Role: Renames the newly duplicated sheet using the current session ID.
    - Inputs: Output from duplicating the template sheet.
    - Parameters: POST to `https://sheets.googleapis.com/v4/spreadsheets/{documentId}:batchUpdate` with JSON body specifying sheet ID and new title.
    - Credentials: Google Sheets OAuth2.
    - Edge cases: Sheet ID retrieval failures, API errors.
  
  - **Set session_id**
    - Type: Set node
    - Role: Assigns the current session ID to the workflow context for downstream use.
    - Inputs: Output from renaming.
    - Outputs: Passes the session ID to `Clear Sheet Content` for next iteration.
    - Edge cases: Expression evaluation failures.

#### 1.5 Conversations Retrieval Block

- **Overview:** Retrieves all conversation messages for the current session from Postgres.
- **Nodes Involved:** 
  - `Get conversations by sessionId`
- **Node Details:**

  - **Get conversations by sessionId**
    - Type: Postgres node (select operation)
    - Role: Selects all rows from `n8n_chat_histories` where `session_id` matches the current session.
    - Inputs: Session ID from loop.
    - Parameters: Uses where clause filtering on session_id.
    - Credentials: Postgres credentials.
    - Edge cases: Large result sets causing timeouts, connection errors.

#### 1.6 Conversation Logging Block

- **Overview:** Appends each message from the session into the corresponding Google Sheets tab with columns for speaker, message content, and timestamp.
- **Nodes Involved:** 
  - `Add conversations`
- **Node Details:**

  - **Add conversations**
    - Type: Google Sheets node (append operation)
    - Role: Appends rows to the sheet named after the session ID.
    - Inputs: Conversation messages from Postgres.
    - Parameters:
      - Columns:
        - Who: from `message.type`
        - Message: from `message.content`
        - Date: `created_at` timestamp formatted as `yyyy-MM-dd hh:mm:ss`
      - Document ID: fixed Google Sheets file ID
      - Sheet name: session ID
    - Credentials: Google Sheets OAuth2.
    - Edge cases: Rate limits, formatting errors, missing data.

#### 1.7 Setup and Schema Initialization Block

- **Overview:** Ensures the Postgres table has a `created_at` column to store message timestamps.
- **Nodes Involved:** 
  - `add create_at column`
- **Node Details:**

  - **add create_at column**
    - Type: Postgres node (executeQuery)
    - Role: Runs SQL to add a `created_at` timestamp column with default current timestamp.
    - Parameters: `ALTER TABLE ONLY "n8n_chat_histories" ADD COLUMN "created_at" TIMESTAMP DEFAULT NOW();`
    - Credentials: Postgres credentials.
    - Edge cases: Column already exists error, permission issues.
    - Notes: Run once before starting the workflow; update table name if needed.

---

### 3. Summary Table

| Node Name                   | Node Type              | Functional Role                                 | Input Node(s)                | Output Node(s)               | Sticky Note                                                             |
|-----------------------------|------------------------|------------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger         | Manual start of workflow                        | —                           | Postgres - Get session ids   | Triggers workflow manually                                              |
| Schedule Trigger            | Schedule Trigger        | Scheduled automatic start at 12:00 daily       | —                           | Postgres - Get session ids   | Triggers workflow daily at noon                                         |
| Postgres - Get session ids  | Postgres               | Retrieves distinct session IDs                  | Trigger nodes               | Loop Over Session IDs        | Replace table name if different                                         |
| Loop Over Session IDs       | SplitInBatches          | Iterates over each session ID                    | Postgres - Get session ids  | Clear Sheet Content          | Iterates sessions to process individually                               |
| Clear Sheet Content          | Google Sheets           | Clears old data in session-named sheet          | Loop Over Session IDs       | Get conversations by sessionId, Duplicate template sheet | Error on missing sheet is expected                                      |
| Duplicate template sheet    | HTTP Request            | Duplicates template Google Sheet tab            | Clear Sheet Content         | Rename Sheet                | Creates blank template sheet copy                                      |
| Rename Sheet               | HTTP Request            | Renames duplicated sheet with session ID        | Duplicate template sheet    | Set session_id              | Uses Google Sheets API batchUpdate                                    |
| Set session_id             | Set                     | Holds session_id for downstream nodes           | Rename Sheet                | Clear Sheet Content         | Passes session_id to clear node                                       |
| Get conversations by sessionId | Postgres            | Retrieves all messages for current session       | Clear Sheet Content         | Add conversations           | Filters messages by session_id                                        |
| Add conversations          | Google Sheets           | Appends messages to session sheet                 | Get conversations by sessionId | Loop Over Session IDs    | Columns: Who, Message, Date                                           |
| add create_at column       | Postgres               | Adds created_at timestamp column to chat table   | —                          | —                          | Run once before first use; replace table name                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**
   - Type: Manual Trigger
   - Name: `When clicking ‘Test workflow’`
   - No parameters needed.

2. **Create Schedule Trigger node:**
   - Type: Schedule Trigger
   - Name: `Schedule Trigger`
   - Set to run daily at hour 12 (noon).

3. **Create Postgres node to get distinct session IDs:**
   - Type: Postgres
   - Name: `Postgres - Get session ids`
   - Operation: Execute Query
   - Query: `SELECT DISTINCT(session_id) FROM n8n_chat_histories`
   - Use your Postgres credentials connected to the AI chat memory DB.

4. **Create SplitInBatches node:**
   - Type: SplitInBatches
   - Name: `Loop Over Session IDs`
   - No special parameters.

5. **Create Google Sheets node to clear sheet content:**
   - Type: Google Sheets
   - Name: `Clear Sheet Content`
   - Operation: Clear
   - Clear Range: A2:C10000
   - Sheet Name: Use expression `{{$json["session_id"]}}` (current session ID)
   - Document ID: Use your Google Sheets document ID (template file)
   - Credentials: Google Sheets OAuth2

6. **Create HTTP Request node to duplicate template sheet:**
   - Type: HTTP Request
   - Name: `Duplicate template sheet`
   - Method: POST
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $parameters.documentId }}/sheets/0:copyTo`
   - Body Parameters: JSON with `destinationSpreadsheetId` set to your Google Sheets document ID.
   - Authentication: Use Google Sheets OAuth2 credentials.

7. **Create HTTP Request node to rename duplicated sheet:**
   - Type: HTTP Request
   - Name: `Rename Sheet`
   - Method: POST
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $parameters.documentId }}:batchUpdate`
   - Body (JSON): Includes `updateSheetProperties` request setting `sheetId` from duplicated sheet response and `title` to current session ID.
   - Authentication: Google Sheets OAuth2

8. **Create Set node to store session_id:**
   - Type: Set
   - Name: `Set session_id`
   - Assign variable `session_id` from `Clear Sheet Content` node output (e.g., first JSON item's `session_id`).

9. **Create Postgres node to get conversations by session ID:**
   - Type: Postgres
   - Name: `Get conversations by sessionId`
   - Operation: Select
   - Table: `n8n_chat_histories`
   - Where clause: `session_id` equals current session ID (from loop)
   - Credentials: Postgres DB

10. **Create Google Sheets node to append conversations:**
    - Type: Google Sheets
    - Name: `Add conversations`
    - Operation: Append
    - Document ID: Same Google Sheets file
    - Sheet Name: Current session ID
    - Columns mapped:
      - Who: `{{$json["message"]["type"]}}`
      - Message: `{{$json["message"]["content"]}}`
      - Date: `{{$json["created_at"].toDateTime().format('yyyy-MM-dd hh:mm:ss')}}`
    - Credentials: Google Sheets OAuth2

11. **Create Postgres node to add created_at column (run once):**
    - Type: Postgres
    - Name: `add create_at column`
    - Operation: Execute Query
    - Query: `ALTER TABLE ONLY "n8n_chat_histories" ADD COLUMN "created_at" TIMESTAMP DEFAULT NOW();`
    - Credentials: Postgres DB

12. **Connect nodes:**
    - Manual Trigger and Schedule Trigger both connect to `Postgres - Get session ids`
    - `Postgres - Get session ids` connects to `Loop Over Session IDs`
    - `Loop Over Session IDs` connects to `Clear Sheet Content`
    - `Clear Sheet Content` connects to both `Get conversations by sessionId` and `Duplicate template sheet`
    - `Duplicate template sheet` connects to `Rename Sheet`
    - `Rename Sheet` connects to `Set session_id`
    - `Set session_id` connects back to `Clear Sheet Content` (for next loop iteration)
    - `Get conversations by sessionId` connects to `Add conversations`
    - `Add conversations` connects back to `Loop Over Session IDs` for next batch

13. **Credentials:**
    - Set up Postgres credentials with access to your AI chat memory database.
    - Set up Google Sheets OAuth2 credentials with access to your Google Sheets template file.

14. **Template Setup:**
    - Prepare a Google Sheets file with a template sheet in the first tab (index 0).
    - Template sheet must have column headers `Who`, `Message`, `Date` in row 1.
    - Share this file with the Google account used in OAuth2 credentials.

15. **Optional:**
    - Run the `add create_at column` node once before first workflow execution if `created_at` does not exist.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow expects a Google Sheets template file with the first tab acting as a blank template to be copied and renamed for each session.                                                                                                                                              | [Google Sheets Template File](https://docs.google.com/spreadsheets/d/14bKI5J0h18Nv48jbe1IXpZWma6EtqYLFWnpKoCB5Bgc/edit?usp=sharing)                                          |
| For Supabase users, it is recommended to connect via Postgres credentials from the Supabase connection pooler rather than using native Supabase nodes for better stability.                                                                                                               | Supabase project > Connect > Transaction pooler > View parameters                                                                                                             |
| Each workflow run clears all conversation sheets and rebuilds them to ensure up-to-date logs. This handles cases where session IDs are overridden or persistent across multiple AI Agent sessions.                                                                                         | N/A                                                                                                                                                                           |
| Make sure to replace the Google Sheets document ID and Postgres table names with your own values before running.                                                                                                                                                                         | N/A                                                                                                                                                                           |
| The `created_at` column in the chat history table stores the timestamp of each message; add it before first use to avoid all messages having the workflow run time as timestamp.                                                                                                         | N/A                                                                                                                                                                           |
| The workflow can be triggered manually for testing or scheduled to run automatically at any interval (hourly, daily, weekly) depending on your needs.                                                                                                                                     | N/A                                                                                                                                                                           |
| The Google Sheets API is used via HTTP request nodes to duplicate and rename sheets since these operations are not natively supported by the Google Sheets node in n8n.                                                                                                                  | Google Sheets API Docs: https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets/sheets/copyTo and https://developers.google.com/sheets/api/reference/rest/v4/spreadsheets/batchUpdate |
| The workflow is designed and tested with n8n version supporting Google Sheets node v4.5, HTTP Request node v4.1, and Postgres node v2.5.                                                                                                                                                | N/A                                                                                                                                                                           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data manipulated are legal and public.