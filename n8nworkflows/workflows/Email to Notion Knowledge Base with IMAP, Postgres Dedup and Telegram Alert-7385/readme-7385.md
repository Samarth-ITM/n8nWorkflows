Email to Notion Knowledge Base with IMAP, Postgres Dedup and Telegram Alert

https://n8nworkflows.xyz/workflows/email-to-notion-knowledge-base-with-imap--postgres-dedup-and-telegram-alert-7385


# Email to Notion Knowledge Base with IMAP, Postgres Dedup and Telegram Alert

### 1. Workflow Overview

This workflow automates the process of capturing unseen emails from an IMAP mailbox, normalizing and deduplicating their content, saving unique messages into a Notion knowledge base, and sending Telegram notifications on successful entries. It is designed primarily for knowledge management scenarios where email content needs to be archived and searchable, while avoiding duplicates.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:**  
  Captures new unseen emails using an IMAP Email Trigger node.

- **1.2 Email Normalization and Deduplication:**  
  Processes raw email data to extract key fields (title, snippet, date, sender, URLs), normalizes content (HTML to plain text), generates slugs, and deduplicates emails based on unique message IDs.

- **1.3 Duplicate Check (Postgres):**  
  Checks a Postgres database table to verify if the email's unique message ID has been processed before.

- **1.4 Conditional Flow:**  
  Continues only if the email is new (not previously processed).

- **1.5 Notion Integration:**  
  Creates a new page in the Notion database with extracted email data.

- **1.6 Telegram Alert:**  
  Sends a notification message to a Telegram chat indicating a successful save.

- **1.7 (Advised but not implemented) Postgres Insert:**  
  Suggests inserting processed message IDs into the Postgres table to mark them as handled.

The workflow includes disabled duplicate sets of similar nodes, likely older versions or test variants, but the active flow is consistent and linear.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Monitors an IMAP email inbox for new unseen messages and triggers the workflow when such emails arrive.

- **Nodes Involved:**  
  - Email Trigger (IMAP)

- **Node Details:**

  - **Email Trigger (IMAP)**  
    - Type: `emailReadImap` (Trigger)  
    - Configuration:  
      - Filters emails with the IMAP flag `UNSEEN` only (customEmailConfig: `["UNSEEN"]`).  
      - Uses standard IMAP server settings, e.g., Gmail’s imap.gmail.com, port 993 with SSL/TLS.  
      - Forces reconnect every 60 seconds to maintain stable connection.  
    - Input: Triggered automatically on new unseen emails.  
    - Output: Emits an array of email items with raw fields such as subject, HTML/text body, date, messageId, from, headers.  
    - Potential Failures:  
      - IMAP authentication failure (wrong credentials or network issues).  
      - Timeout or connection drops.  
      - No emails matching filter (outputs empty).  
    - Version: 2.1

#### 1.2 Email Normalization and Deduplication

- **Overview:**  
  Processes each email item to extract and normalize key information, convert HTML to text, generate slugs, and create a unique message ID hash if missing.

- **Nodes Involved:**  
  - Code (JavaScript Function)

- **Node Details:**

  - **Code**  
    - Type: `code` (Function node)  
    - Configuration:  
      - Implements helper functions for date formatting (ISO), slug generation (lowercase, URL-safe), text limiting, URL extraction, email address extraction, HTML to text stripping, and a simple hash function.  
      - Extracts subject, body text (prefers HTML converted to text, falls back to plain text), sent date, sender address, first URL in body, message ID (with fallback to custom hash), and debug metadata.  
      - Output fields:  
        - `title` (max 140 chars),  
        - `snippet` (max 260 chars),  
        - `bodyText`,  
        - `slug`,  
        - `messageId`,  
        - `sentAt` (ISO string),  
        - `fromAddress`,  
        - `sourceUrl`,  
        - `debugFields` (source metadata for troubleshooting).  
    - Input: Raw email items from IMAP trigger.  
    - Output: Normalized email items with standardized fields.  
    - Potential Failures:  
      - Malformed or missing fields in input JSON causing expression or code errors.  
      - Unexpected HTML structures causing incomplete text extraction.  
      - Date parsing errors default to current date/time.  
    - Version: 2

#### 1.3 Duplicate Check (Postgres)

- **Overview:**  
  Queries a Postgres database table `email_kb_index` to check if the message ID already exists, to prevent duplicate storage.

- **Nodes Involved:**  
  - Execute a SQL query (Postgres)

- **Node Details:**

  - **Execute a SQL query**  
    - Type: `postgres`  
    - Configuration:  
      - Query:  
        ```sql
        SELECT EXISTS(
          SELECT 1
          FROM email_kb_index
          WHERE message_id = '{{ $json.messageId }}'
        ) AS exists;
        ```  
      - No additional parameters; uses n8n expression to inject `messageId` dynamically.  
    - Input: Normalized email JSON with `messageId`.  
    - Output: JSON with boolean field `exists`.  
    - Potential Failures:  
      - Postgres connection/authentication errors.  
      - SQL syntax errors if messageId malformed (unlikely here).  
      - Table `email_kb_index` missing or misconfigured.  
    - Version: 2.6

#### 1.4 Conditional Flow (If)

- **Overview:**  
  Determines the continuation of the workflow only if the message ID does not exist in Postgres (i.e., new email).

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: `if`  
    - Configuration:  
      - Condition: `$json.exists` equals `false` (strict boolean check).  
      - Case sensitive and strict type validation enabled.  
    - Input: Postgres query result with `exists` field.  
    - Output:  
      - True branch continues to Notion page creation.  
      - False branch ends workflow or could be used to log duplicates (not connected here).  
    - Potential Failures:  
      - If input JSON missing `exists` field.  
      - Expression evaluation errors if data malformed.  
    - Version: 2.2

#### 1.5 Notion Integration (Create a database page)

- **Overview:**  
  Creates a new page in the specified Notion database with extracted email data.

- **Nodes Involved:**  
  - Create a database page (Notion)

- **Node Details:**

  - **Create a database page**  
    - Type: `notion`  
    - Configuration:  
      - Resource: `databasePage`  
      - Database ID: Selected from list; corresponds to a Notion database shared with the integration.  
      - Properties mapped from the code node output:  
        - Title → `title`  
        - Date → `sentAt`  
        - Summary (Rich text) → `snippet`  
        - Source URL → `sourceUrl` (optional, ignore if empty)  
        - From → `fromAddress`  
        - Slug → `slug`  
      - No tags or notes configured here but can be added.  
    - Input: Normalized email JSON from code node.  
    - Output: Notion page creation response JSON, including `id` (page ID).  
    - Potential Failures:  
      - Notion API authentication failure.  
      - Missing permissions or database not shared with integration.  
      - Invalid property mappings or missing required fields.  
      - Rate limiting by Notion API.  
    - Version: 2.2

#### 1.6 Telegram Alert (Send a text message)

- **Overview:**  
  Sends a Telegram message notifying about the successful saving of the email content to Notion.

- **Nodes Involved:**  
  - Send a text message (Telegram)

- **Node Details:**

  - **Send a text message**  
    - Type: `telegram`  
    - Configuration:  
      - Sends to chat ID from environment variable `TELEGRAM_CHAT_ID`.  
      - Message text uses Markdown formatting:  
        ```
        ✅ Saved to Notion
        *{{ $json.title }}*
        From: {{ $json.fromAddress }}
        Date: {{ $json.sentAt }}
        Link: {{ $json.sourceUrl || '-' }}
        ```  
      - No additional fields set.  
    - Input: Output of Notion page creation node.  
    - Output: Telegram API response JSON.  
    - Potential Failures:  
      - Invalid or missing Telegram bot token credentials.  
      - Invalid or blocked chat ID.  
      - Telegram API rate limits or outages.  
    - Version: 1.2

#### 1.7 (Recommended) Postgres Insert (Not implemented)

- **Overview:**  
  Not implemented in the active workflow, but sticky notes recommend inserting processed message IDs into Postgres to mark them as processed.

- **Details:**  
  - Suggested SQL:  
    ```sql
    INSERT INTO email_kb_index (message_id, slug, created_at, notion_page_id)
    VALUES ($1, $2, $3, $4)
    ON CONFLICT (message_id) DO NOTHING;
    ```  
  - Parameters to bind from JSON and Notion node outputs.  
  - Ensures deduplication on subsequent runs.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                     | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                          |
|-------------------------|----------------------|-----------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Email Trigger (IMAP)     | emailReadImap (Trigger) | Input Reception (new unseen emails) | -                     | Code                    | Options → customEmailConfig: ["UNSEEN"]<br>Key output fields: subject, textHtml/textPlain/html, date, messageId, from. |
| Code                    | code (Function)      | Email Normalization & Deduplication | Email Trigger (IMAP)   | Execute a SQL query      | Converts HTML to plain text fallback.<br>Outputs normalized fields: title, snippet, slug, messageId, sentAt, fromAddress, sourceUrl. |
| Execute a SQL query      | postgres             | Check for Duplicate in DB          | Code                  | If                      | Safe query checking if message_id exists.                                                          |
| If                      | if                   | Conditional flow based on duplicate check | Execute a SQL query    | Create a database page   | Condition: proceed if exists=false (new message).                                                  |
| Create a database page   | notion               | Create Notion database page        | If (true branch)       | Send a text message      | Map email fields to Notion columns.<br>Database must be shared with integration.                   |
| Send a text message      | telegram             | Send Telegram notification         | Create a database page | -                       | Chat ID from env `TELEGRAM_CHAT_ID`.<br>Markdown message confirming save.                          |
| Sticky Note              | stickyNote           | Documentation and instructions     | -                     | -                       | Detailed setup instructions and SQL table creation.                                               |

*Note: Several disabled duplicate nodes exist (Email Trigger (IMAP)1, Code1, etc.) but are not active and omitted here.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create Postgres Table (Run once externally):**  
   Execute SQL to create deduplication table:  
   ```sql
   CREATE TABLE IF NOT EXISTS email_kb_index (
     message_id TEXT PRIMARY KEY,
     slug TEXT,
     created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
     notion_page_id TEXT
   );
   ```

2. **Create IMAP Email Trigger Node:**  
   - Add `emailReadImap` trigger node.  
   - Set options → `customEmailConfig`: `["UNSEEN"]` to fetch only unread emails.  
   - Configure host (e.g., `imap.gmail.com`), port (993), SSL/TLS, and authentication credentials.  
   - Set `forceReconnect` to 60 seconds to maintain connection.

3. **Add Code Node for Normalization:**  
   - Add a `code` node connected to the IMAP trigger output.  
   - Paste the provided JavaScript code that:  
     - Extracts subject, body (converts HTML to plain text), date, from address, and first URL.  
     - Generates a slug from title.  
     - Creates a unique messageId (using header or fallback hash).  
     - Limits title and snippet length.  
   - No additional input parameters required.  
   - Outputs normalized JSON with fields: `title`, `snippet`, `bodyText`, `slug`, `messageId`, `sentAt`, `fromAddress`, `sourceUrl`.

4. **Add Postgres Query Node to Check Duplicate:**  
   - Add `postgres` node connected to Code node output.  
   - Operation: Execute SQL query.  
   - Query:  
     ```sql
     SELECT EXISTS(
       SELECT 1
       FROM email_kb_index
       WHERE message_id = '{{ $json.messageId }}'
     ) AS exists;
     ```  
   - Configure Postgres credentials with access to the defined table.

5. **Add If Node for Conditional Branch:**  
   - Add `if` node connected to Postgres query output.  
   - Condition: `$json.exists` equals `false` (boolean false).  
   - True branch continues workflow; False branch ends or can be used for logging duplicates.

6. **Add Notion Node to Create Database Page:**  
   - Add `notion` node connected to If node’s True output.  
   - Resource: `databasePage`.  
   - Select or paste the Notion database ID connected to your Notion account integration.  
   - Map properties:  
     - Title: `={{ $json.title }}`  
     - Date: `={{ $json.sentAt }}`  
     - Summary (rich text): `={{ $json.snippet }}`  
     - Source URL (optional): `={{ $json.sourceUrl }}` (ignore if empty)  
     - From (rich text): `={{ $json.fromAddress }}`  
     - Slug (rich text): `={{ $json.slug }}`  
   - Ensure the Notion database is shared with your integration.

7. **Add Telegram Node to Send Notification:**  
   - Add `telegram` node connected to Notion node output.  
   - Configure Telegram Bot credentials.  
   - Set Chat ID to environment variable `TELEGRAM_CHAT_ID`.  
   - Compose message (Markdown):  
     ```
     ✅ Saved to Notion
     *{{ $json.title }}*
     From: {{ $json.fromAddress }}
     Date: {{ $json.sentAt }}
     Link: {{ $json.sourceUrl || '-' }}
     ```  
   - No additional fields needed.

8. **(Recommended) Add Postgres Insert Node (Optional):**  
   - After Notion node, add a Postgres node to insert processed message IDs to prevent duplicates.  
   - Query:  
     ```sql
     INSERT INTO email_kb_index (message_id, slug, created_at, notion_page_id)
     VALUES ($1, $2, $3, $4)
     ON CONFLICT (message_id) DO NOTHING;
     ```  
   - Parameters:  
     - $1 = `={{ $json.messageId }}`  
     - $2 = `={{ $json.slug }}`  
     - $3 = `={{ $json.sentAt }}`  
     - $4 = `={{ $node["Create a database page"].json.id }}`  
   - Connect this node before the Telegram node or in parallel after successful Notion page creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| 0) Create Postgres Table (run once) - SQL provided in sticky notes.<br>1) IMAP Email Trigger: Set `customEmailConfig` to `["UNSEEN"]` to fetch only new emails.<br>2) Function Node normalizes email content, including HTML to text conversion, slug creation, and message ID hashing.<br>3) Postgres query safely checks for duplicates.<br>5) IF node filters to new emails only.<br>6) Notion node creates database page with mapped properties.<br>7) Recommended Postgres insert to mark processed emails (not implemented in workflow).<br>Telegram sends notification for each new saved email.<br>Ensure database and Notion integration permissions are correctly configured.<br>Use environment variables for sensitive info like Telegram chat ID.<br>See sticky notes inside workflow for detailed instructions. | Detailed implementation and configuration notes embedded inside workflow Sticky Notes nodes.                  |
| Telegram notification message includes Markdown formatting and useful email metadata for quick human review.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Telegram node content                                                                                             |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.