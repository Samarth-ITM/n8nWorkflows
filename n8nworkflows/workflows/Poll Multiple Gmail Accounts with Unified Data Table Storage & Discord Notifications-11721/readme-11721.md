Poll Multiple Gmail Accounts with Unified Data Table Storage & Discord Notifications

https://n8nworkflows.xyz/workflows/poll-multiple-gmail-accounts-with-unified-data-table-storage---discord-notifications-11721


# Poll Multiple Gmail Accounts with Unified Data Table Storage & Discord Notifications

### 1. Workflow Overview

This workflow polls multiple Gmail accounts at scheduled intervals, retrieves new email messages based on dynamic time windows, stores the normalized email data in a unified Data Table, and sends notifications on a Discord channel summarizing the number of new emails received. It is designed to centrally manage multiple Gmail credentials, efficiently fetch emails without duplication, and provide real-time updates via Discord.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Scheduling:** Scheduled trigger and retrieval of Gmail account credentials from a data table.
- **1.2 Per-Account Processing Loop:** Iterates through each Gmail account, calculates polling time windows, and dynamically runs Gmail queries with specific OAuth credentials.
- **1.3 Email Processing & Storage:** Normalizes and formats email data, checks for duplicates, upserts new emails into a master data table, and updates polling status per account.
- **1.4 Notification:** Aggregates the count of new emails fetched during the run and sends a summary message to a Discord channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Scheduling

- **Overview:**  
  This block initiates the workflow on a schedule and loads all Gmail accounts to be polled from a centralized Data Table.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get All Email Accounts  
  - Sticky Note (Get Email Accounts)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow automatically every hour.  
    - *Config:* Interval set to every 1 hour.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Triggers downstream fetching of email accounts.  
    - *Failures:* Misconfiguration of schedule, timezone issues.

  - **Get All Email Accounts**  
    - *Type:* Data Table (get operation)  
    - *Role:* Fetches all rows from the `cold_email_accounts` data table containing Gmail account info and credentials.  
    - *Config:* Returns all rows, no filters applied.  
    - *Inputs:* Triggered by Schedule Trigger.  
    - *Outputs:* List of email accounts with credentials and metadata for polling.  
    - *Failures:* Table not accessible, data permissions, empty results.

  - **Sticky Note (Get Email Accounts)**  
    - *Role:* Documentation for this block, explaining purpose.

---

#### 1.2 Per-Account Processing Loop

- **Overview:**  
  Processes each Gmail account individually in a loop, calculates the time window for polling emails, and dynamically executes Gmail API calls with the correct OAuth credentials.

- **Nodes Involved:**  
  - Loop Over Items  
  - Code in JavaScript (Polling Time Calculator)  
  - Run Node With Credentials X (Dynamic Gmail Query Execution)  
  - If new email found  
  - If new email found1  
  - Sticky Notes (Polling Time Calculator, Get Gmail Messages Dynamically)

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Iterates through each Gmail account record fetched from the data table one-by-one for sequential processing.  
    - *Config:* Default batch size (1 by default, implied by iteration).  
    - *Inputs:* List of Gmail accounts.  
    - *Outputs:* Single account item per iteration.  
    - *Failures:* Large batch sizes may cause timeouts.

  - **Code in JavaScript (Polling Time Calculator)**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Computes the polling window for Gmail API queries using the account's last polled timestamp and current time. Converts timestamps to epoch seconds for Gmail query.  
    - *Config:* Runs once per item (account).  
    - *Key Expressions:*  
      - `last_poll = Math.floor(new Date(last_polled).getTime() / 1000)`  
      - `now = Math.floor(Date.now() / 1000)`  
      - Adds `after` and `before` properties to item JSON.  
    - *Inputs:* Single Gmail account item with `last_polled` timestamp.  
    - *Outputs:* Modified item with time window for query.  
    - *Failures:* Invalid `last_polled` format, date parsing errors.

  - **Run Node With Credentials X**  
    - *Type:* Run Node With Credentials (Custom n8n node)  
    - *Role:* Dynamically runs a Gmail node (Get many messages) using the OAuth2 credential of the current account.  
    - *Config:*  
      - Operation: Get all messages within the `after` and `before` time window.  
      - Gmail query excludes emails from specific domains (`-from:(*@getkhatriautomations.com OR *@thekhatriautomations.com OR *@heykhatriautomations.com)`).  
      - Credentials ID set dynamically from current account item.  
    - *Inputs:* Account item with OAuth credentials and query parameters.  
    - *Outputs:* List of messages fetched from Gmail.  
    - *Failures:* OAuth token expiration, Gmail API rate limits, query syntax errors.

  - **If new email found**  
    - *Type:* If Node  
    - *Role:* Checks if any new emails were returned from the Gmail query.  
    - *Config:* Condition tests for non-empty JSON output from Gmail node.  
    - *Inputs:* Output from Run Node With Credentials X.  
    - *Outputs:*  
      - True branch: Proceed to email normalization and storage.  
      - False branch: Update last_polled without saving emails.  
    - *Failures:* Expression evaluation errors if input is malformed.

  - **If new email found1**  
    - *Type:* If Node  
    - *Role:* Checks if retrieved email records exist in Data Table to avoid duplicates before counting new emails.  
    - *Config:* Tests for non-empty results from the "Get new emails if any" node.  
    - *Inputs:* Result of data table lookup for message_id.  
    - *Outputs:*  
      - True branch: Proceed to count emails for notification.  
      - False branch: Skip counting.  
    - *Failures:* Empty or null inputs.

  - **Sticky Notes (Polling Time Calculator, Get Gmail Messages Dynamically)**  
    - Provide high-level explanations of time window calculation and dynamic Gmail API calls with OAuth credential injection.

---

#### 1.3 Email Processing & Storage

- **Overview:**  
  Normalizes fetched email data, checks for duplicates, upserts new records into the central "All Emails" Data Table, and updates the last polled timestamp per email account.

- **Nodes Involved:**  
  - Email Normalization (Code)  
  - Get new emails if any (Data Table Get)  
  - upsert emails (Data Table Upsert)  
  - Update last_polled  
  - Update last_polled1  
  - Sticky Note (Add to Data Tables)

- **Node Details:**

  - **Email Normalization**  
    - *Type:* Code Node (JavaScript)  
    - *Role:* Extracts and formats relevant email fields from raw Gmail message JSON. Normalizes fields like sender name/email, recipients, subject, preview text, body, labels, attachments, read status, etc.  
    - *Config:* Runs once per email item.  
    - *Key Expressions:* JSON construction including attachments array, flags for read/unread, subject fallback.  
    - *Inputs:* Raw Gmail message JSON.  
    - *Outputs:* JSON stringified normalized email data.  
    - *Failures:* Unexpected message structure, missing fields.

  - **Get new emails if any**  
    - *Type:* Data Table (get operation)  
    - *Role:* Checks if the newly fetched email already exists in the "All Emails" table by message ID to prevent duplicates.  
    - *Config:* Filters on `message_id` equal to current Gmail message ID.  
    - *Inputs:* Current message ID.  
    - *Outputs:* Existing record(s) if any.  
    - *Failures:* Table access errors.

  - **upsert emails**  
    - *Type:* Data Table (upsert operation)  
    - *Role:* Inserts new normalized email records or updates existing ones based on email_data uniqueness.  
    - *Config:* Matching column: `email_data`. Columns mapped: `email_data` (normalized email JSON), `message_id`.  
    - *Inputs:* Normalized email data and message ID.  
    - *Outputs:* Upsert result.  
    - *Failures:* Data type mismatches, table write errors.

  - **Update last_polled & Update last_polled1**  
    - *Type:* Data Table (update operation)  
    - *Role:* Updates the `last_polled` timestamp to current time for the processed email account.  
    - *Config:* Filters on account ID, updates `last_polled` field and resets `warmup_enabled` to false.  
    - *Inputs:* Account ID from loop item.  
    - *Outputs:* Updated account record.  
    - *Failures:* Concurrent update conflicts, empty filters.

  - **Sticky Note (Add to Data Tables)**  
    - Explains the upsert logic and last polled updates.

---

#### 1.4 Notification

- **Overview:**  
  After processing all email accounts, this block counts the number of new emails fetched and sends a notification message to a specific Discord channel.

- **Nodes Involved:**  
  - Edit Fields (Set)  
  - Limit  
  - Send a message (Discord)  
  - Sticky Note (Send Notification on Discord Channel)

- **Node Details:**

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Calculates and assigns the total count of new emails processed during the current workflow run.  
    - *Config:* Sets variable `emailCount` to the length of the input array (all new emails).  
    - *Inputs:* Collection of new emails.  
    - *Outputs:* Single object with `emailCount`.  
    - *Failures:* Empty input array.

  - **Limit**  
    - *Type:* Limit  
    - *Role:* Limits the number of items passed to the Discord message node (typically 1).  
    - *Config:* Default limit (1).  
    - *Inputs:* Email count object.  
    - *Outputs:* Single item for message content.  
    - *Failures:* None expected.

  - **Send a message (Discord)**  
    - *Type:* Discord Node  
    - *Role:* Sends a message to a Discord server channel containing the count of new emails (e.g., "15 new emails arrived").  
    - *Config:*  
      - Content templated with `emailCount`.  
      - Uses configured Discord Bot credentials.  
      - Targets specific guild and channel IDs.  
    - *Inputs:* Single item with `emailCount`.  
    - *Outputs:* Discord API response.  
    - *Failures:* Invalid webhook or bot token, network errors, permission errors.

  - **Sticky Note (Send Notification on Discord Channel)**  
    - Describes the notification purpose and summary message content.

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                                   | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                             |
|---------------------------|------------------------------------|-------------------------------------------------|-----------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                   | Initiate workflow every hour                     |                             | Get All Email Accounts         |                                                                                                       |
| Get All Email Accounts    | Data Table (get)                   | Load all Gmail accounts to poll                  | Schedule Trigger            | Loop Over Items                | ## Get Email Accounts Loads all email accounts to be polled.                                          |
| Loop Over Items           | Split In Batches                   | Iterate accounts one by one                       | Get All Email Accounts      | Get new emails if any, Code in JavaScript  |                                                                                                       |
| Code in JavaScript        | Code                              | Calculate polling time window (after/before)    | Loop Over Items             | Run Node With Credentials X    | ## Polling Time Calculator Calculates the polling time range in Epoch.                                |
| Run Node With Credentials X | Run Node With Credentials         | Dynamic Gmail message retrieval per account      | Code in JavaScript          | If new email found             | ## Get Gmail Messages Dynamically Runs Gmail nodes dynamically using OAuth credentials from the data table. |
| If new email found        | If                                | Check if Gmail returned new emails                | Run Node With Credentials X | Email Normalization, Update last_polled1 |                                                                                                       |
| Email Normalization       | Code                              | Normalize raw Gmail email data                    | If new email found          | Update last_polled             |                                                                                                       |
| Update last_polled        | Data Table (update)                | Update last_polled timestamp after email save   | Email Normalization         | upsert emails                 | ## Add to Data Tables Upserts new emails and updates last_polled per account.                         |
| upsert emails             | Data Table (upsert)                | Insert or update email records in master table   | Update last_polled           | Loop Over Items               |                                                                                                       |
| Get new emails if any     | Data Table (get)                   | Check for existing emails to avoid duplicates    | Loop Over Items             | If new email found1            |                                                                                                       |
| If new email found1       | If                                | Check if email exists in data table               | Get new emails if any       | Edit Fields                   |                                                                                                       |
| Edit Fields               | Set                               | Count new emails for notification                 | If new email found1         | Limit                        | ## Send Notification on Discord Channel Sends a summary message with the number of newly received emails. |
| Limit                    | Limit                             | Limit output items to 1                            | Edit Fields                 | Send a message                |                                                                                                       |
| Send a message            | Discord                           | Send new email count summary to Discord channel | Limit                      |                               |                                                                                                       |
| Update last_polled1       | Data Table (update)                | Update last_polled timestamp if no new emails   | If new email found          | Loop Over Items               |                                                                                                       |
| Sticky Note1              | Sticky Note                      | Full workflow node-by-node explanation            |                             |                               | ### How It Works (node-by-node): ... (full explanation)                                               |
| Sticky Note               | Sticky Note                      | Get Email Accounts explanation                     |                             |                               | ## Get Email Accounts Loads all email accounts to be polled.                                          |
| Sticky Note2              | Sticky Note                      | Polling Time Calculator explanation                |                             |                               | ## Polling Time Calculator Calculates the polling time range in Epoch.                                |
| Sticky Note3              | Sticky Note                      | Gmail dynamic message retrieval explanation        |                             |                               | ## Get Gmail Messages Dynamically Runs Gmail nodes dynamically using OAuth credentials from the data table. |
| Sticky Note4              | Sticky Note                      | Data Tables upsert and last_polled update explanation |                             |                               | ## Add to Data Tables Upserts new emails into the All Emails table and avoids duplicates. Updates last_polled after processing. |
| Sticky Note5              | Sticky Note                      | Discord notification explanation                    |                             |                               | ## Send Notification on Discord Channel Sends a summary message with the number of newly received emails.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create a "Schedule Trigger" node:**  
   - Set interval to every 1 hour.

2. **Create a "Data Table" node ("Get All Email Accounts"):**  
   - Operation: Get  
   - Data Table: Select your Gmail accounts data table (e.g., "cold_email_accounts")  
   - Return All: True  
   - Connect Schedule Trigger → Get All Email Accounts.

3. **Add a "Split In Batches" node ("Loop Over Items"):**  
   - Default batch size (1)  
   - Connect Get All Email Accounts → Loop Over Items.

4. **Add a "Code" node ("Polling Time Calculator"):**  
   - Mode: Run Once For Each Item  
   - JavaScript code:  
     ```js
     let last_polled = $json.last_polled;
     const last_poll = Math.floor(new Date(last_polled).getTime() / 1000);
     const now = Math.floor(Date.now() / 1000);
     let item = $json;
     item.after = last_poll;
     item.before = now;
     return { item };
     ```  
   - Connect Loop Over Items (main output 2) → Code in JavaScript.

5. **Create a "Run Node With Credentials" node ("Run Node With Credentials X"):**  
   - Paste a Gmail node JSON configured as:  
     - Operation: Get All Messages  
     - Return All: True  
     - Query Filter: `after:{{ $json.item.after }} before:{{ $json.item.before }} -from:(*@getkhatriautomations.com OR *@thekhatriautomations.com OR *@heykhatriautomations.com)`  
     - Credentials: Set dynamically using `={{ $json.item.cred_id }}` (from current item)  
   - Connect Code in JavaScript → Run Node With Credentials X.

6. **Add an "If" node ("If new email found"):**  
   - Condition: Check if output from Gmail node is not empty.  
   - Connect Run Node With Credentials X → If new email found.

7. **For True branch:**  
   - Add a "Data Table" node ("Get new emails if any"):  
     - Operation: Get  
     - Data Table: "All Emails" table  
     - Filter: `message_id` = `{{ $json.message_id }}`  
     - Connect If new email found (true) → Get new emails if any.

   - Add an "If" node ("If new email found1"):  
     - Condition: Check if output from "Get new emails if any" is not empty.  
     - Connect Get new emails if any → If new email found1.

   - Add a "Set" node ("Edit Fields"):  
     - Assignment: `emailCount` = `={{ $input.all().length }}`  
     - Connect If new email found1 (true) → Edit Fields.

   - Add a "Limit" node:  
     - Limit to 1 item  
     - Connect Edit Fields → Limit.

   - Add a "Discord" node ("Send a message"):  
     - Resource: Message  
     - Channel ID: Your Discord channel ID  
     - Content: `={{ $json.emailCount }} new emails arrived`  
     - Credentials: Discord Bot API OAuth2 credentials  
     - Connect Limit → Send a message.

8. **For True branch of If new email found:**  
   - Add a "Code" node ("Email Normalization"):  
     - JavaScript code extracts and formats email fields (sender, subject, preview, body, attachments, labels, read status).  
     - Connect If new email found (true) → Email Normalization.

   - Add a "Data Table" node ("upsert emails"):  
     - Operation: Upsert  
     - Data Table: "All Emails"  
     - Match Column: `email_data`  
     - Columns: `email_data` (normalized email JSON), `message_id`  
     - Connect Email Normalization → upsert emails.

   - Add a "Data Table" node ("Update last_polled"):  
     - Operation: Update  
     - Data Table: "cold_email_accounts"  
     - Filter: On account ID from current loop item  
     - Update: `last_polled` = current timestamp, `warmup_enabled` = false  
     - Connect Email Normalization → Update last_polled.

   - Connect upsert emails → Loop Over Items (to process next account).

9. **For False branch of If new email found:**  
   - Add a "Data Table" node ("Update last_polled1"):  
     - Same update as above (sets `last_polled` to now)  
     - Connect If new email found (false) → Update last_polled1 → Loop Over Items.

10. **Ensure connections are properly sequenced:**  
    - Schedule Trigger → Get All Email Accounts → Loop Over Items → Code in JavaScript → Run Node With Credentials X → If new email found → (either email processing or last_polled update)  
    - At the end of Loop Over Items, count total new emails and send notification.

11. **Setup credentials:**  
    - For each Gmail account, create Gmail OAuth2 credentials in n8n.  
    - Store credential ID in the `cold_email_accounts` data table as `cred_id`.  
    - Configure Discord Bot API OAuth2 credentials for the Discord node.

12. **Final checks:**  
    - Validate data tables exist with correct schemas (`cold_email_accounts`, `All Emails`).  
    - Confirm Discord channel and guild IDs are correct.  
    - Test schedule trigger manually for correct operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow excludes emails from specific internal domains (e.g., `getkhatriautomations.com`) in Gmail queries to avoid noise. | Gmail query filter syntax in Run Node With Credentials X.                                              |
| Uses dynamic injection of OAuth2 credentials per Gmail account to securely fetch emails without credential overlap.               | Run Node With Credentials X node configuration.                                                        |
| Discord notifications summarize new emails per run, enabling real-time monitoring of email activity.                             | Discord node configured with bot credentials and specific channel.                                    |
| Data Tables used: `cold_email_accounts` (email accounts and credentials), `All Emails` (centralized email storage).              | Data Table nodes for persistence and querying.                                                        |
| The workflow handles polling windows using epoch seconds to avoid duplicate fetching and to maintain consistent time ranges.    | Code in JavaScript node for time window calculation.                                                  |
| Potential failure points include OAuth token expiry, Gmail API rate limits, data table access issues, and expression errors.     | Monitoring recommended on these nodes to ensure reliability.                                          |
| For more on Gmail node usage: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.gmail/                                | n8n official documentation.                                                                            |
| For Discord node setup and webhooks: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.discord/                       | n8n official documentation.                                                                            |
| Data Table management and schema design is critical for avoiding duplicates and ensuring efficient lookups in this workflow.     | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.dataTable/                               |

---

This documentation provides a comprehensive understanding of the "Poll Multiple Gmail Accounts with Unified Data Table Storage & Discord Notifications" workflow, enabling reproduction, modification, and troubleshooting by both human developers and automation agents.