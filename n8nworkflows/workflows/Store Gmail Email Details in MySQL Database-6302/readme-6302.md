Store Gmail Email Details in MySQL Database

https://n8nworkflows.xyz/workflows/store-gmail-email-details-in-mysql-database-6302


# Store Gmail Email Details in MySQL Database

### 1. Workflow Overview

This workflow automates the process of capturing incoming Gmail emails and storing their key details into a MySQL database. It is designed for use cases where email metadata and sender/recipient information need to be tracked or archived systematically, such as CRM contact management or email analytics.

The workflow’s logic is grouped into the following blocks:

- **1.1 Input Reception:** Continuously monitors a Gmail inbox for new emails.
- **1.2 Data Extraction:** Parses sender and recipient names and emails from raw Gmail data.
- **1.3 Database Upsert:** Inserts or updates the email details into a MySQL contacts table, avoiding duplicate entries.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a new email arrives in the linked Gmail account, polling every minute.

- **Nodes Involved:**  
  - Receive Email

- **Node Details:**

  - **Receive Email**  
    - Type: Gmail Trigger (n8n-nodes-base.gmailTrigger)  
    - Role: Watches Gmail inbox for new emails.  
    - Configuration:  
      - Polling every minute (mode: everyMinute)  
      - No specific filter criteria set, so it triggers on all new emails.
    - Credentials: Gmail OAuth2 credentials linked to the user’s Gmail account.  
    - Input: None (trigger node).  
    - Output: Emits full Gmail message data JSON including fields like From, To, Subject, snippet, id, threadId.  
    - Edge Cases / Failures:  
      - OAuth token expiration or invalid credentials may cause trigger failure.  
      - Network or API rate limiting by Gmail could delay or block trigger.  
      - Large inboxes or very frequent emails may cause missed triggers if polling interval is too long or API quota is reached.

#### 1.2 Data Extraction

- **Overview:**  
  Parses and normalizes sender and recipient email addresses and display names from the raw email headers.

- **Nodes Involved:**  
  - Get Client Name and Email

- **Node Details:**

  - **Get Client Name and Email**  
    - Type: Code Node (n8n-nodes-base.code)  
    - Role: Extracts and structures sender and recipient name/email fields from the raw Gmail headers.  
    - Configuration:  
      - Runs once per incoming email item.  
      - JavaScript code parses the `From` and `To` fields, which may be in formats like `"Name <email@example.com>"` or just `"email@example.com"`.  
      - Trims spaces, and separates names and emails accordingly.  
    - Input: Raw email JSON from “Receive Email” node.  
    - Output: JSON object with keys: `sender_name`, `sender_email`, `recipient_name`, `recipient_email`.  
    - Expressions/Variables: Accesses `$json.From` and `$json.To` to extract data.  
    - Edge Cases / Failures:  
      - Missing or malformed `From` or `To` fields may cause undefined or null outputs.  
      - Multiple recipients in `To` field are not handled (only first recipient parsed).  
      - Unexpected email header formats may break parsing logic.

#### 1.3 Database Upsert

- **Overview:**  
  Inserts new or updates existing records in a MySQL database table to store the extracted email and contact details.

- **Nodes Involved:**  
  - Insert New Client in MySQL

- **Node Details:**

  - **Insert New Client in MySQL**  
    - Type: MySQL Node (n8n-nodes-base.mySql)  
    - Role: Performs an UPSERT operation on the `contacts` table to store email metadata and client info.  
    - Configuration:  
      - Table: `contacts` (must exist in the connected MySQL database).  
      - Operation: `upsert` (updates existing record if matched, otherwise inserts).  
      - Matching criteria: Matches on `messageId` column using Gmail message ID (`$('Receive Email').item.json.id`).  
      - Columns updated/inserted: threadId, sender_name, sender_email, recipient_name, recipient_email, subject, snippet.  
      - On error: “continueRegularOutput” to avoid workflow failure if any DB error occurs.  
    - Credentials: MySQL credentials connected to the target database.  
    - Input: Output from “Get Client Name and Email” with extracted fields and original email JSON.  
    - Output: MySQL query result data.  
    - Edge Cases / Failures:  
      - Database connection errors or credential misconfiguration.  
      - Missing required columns in the target table cause insert/update failures.  
      - Duplicate key constraint violations if matching column setup is incorrect.  
      - SQL injection risk is minimal due to parameterized inputs but depends on n8n internals.  
    - Version Note: Uses MySQL node version 2.4 supporting upsert operation.

---

### 3. Summary Table

| Node Name               | Node Type                | Functional Role                   | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                         |
|-------------------------|--------------------------|---------------------------------|----------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------|
| Receive Email           | Gmail Trigger            | Trigger on new incoming emails  | None                 | Get Client Name and Email |                                                                                                                     |
| Get Client Name and Email | Code                     | Parse sender/recipient info     | Receive Email        | Insert New Client in MySQL |                                                                                                                     |
| Insert New Client in MySQL | MySQL                    | Upsert email/contact data       | Get Client Name and Email | None                     |                                                                                                                     |
| Sticky Note             | Sticky Note              | Workflow overview and instructions | None                 | None                     | ## This workflow processes emails received in Gmail and saves detailed information about each email to a MySQL database. <br>### Before using, you need to have:<br>- Gmail credentials<br>- MySQL database credentials<br>- A table in your database with the following columns:<br>  - messageId (Gmail message ID)<br>  - threadId<br>  - snippet<br>  - sender_name (nullable)<br>  - sender_email<br>  - recipient_name (nullable)<br>  - recipient_email<br>  - subject (nullable)<br><br>### How it works:<br>- The Gmail Trigger listens for new emails (checked every minute).<br>- A Code Node extracts the following fields from each email:<br>  - Sender's name and email<br>  - Recipient's name and email<br>- The MySQL Node inserts the extracted data into your database.<br>- If an entry with the same sender email already exists, it updates the record with the new details.<br><br>### How to use:<br>- Make sure your database table has all required columns listed above.<br>- Select the appropriate table and configure the matching column (e.g., id) to avoid duplicates.<br><br>### Customizing this Workflow:<br>- You can further modify the workflow to store attachments, timestamps, labels, or any other Gmail metadata as needed. |
| Sticky Note1            | Sticky Note              | MySQL table setup instructions  | None                 | None                     | ## Please setup this node first<br>- You'll need a MySQL database with a table to store contact and email details.<br>- The table must include the following columns:<br>  - id (Gmail message ID — should be unique)<br>  - threadId<br>  - snippet<br>  - sender_name (nullable)<br>  - sender_email<br>  - recipient_name (nullable)<br>  - recipient_email<br>  - subject (nullable) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Add node of type `Gmail Trigger`.  
   - Configure to poll every minute (`pollTimes` → set mode to `everyMinute`).  
   - Set up Gmail OAuth2 credentials with valid access to the target Gmail inbox.  
   - No filters needed unless you want to restrict emails.  
   - Name this node `Receive Email`.

2. **Add Code Node to Parse Email Addresses:**  
   - Add a `Code` node connected from `Receive Email`.  
   - Set mode to `runOnceForEachItem`.  
   - Paste the JavaScript code:

     ```javascript
     // Extract Sender Info
     let sender_email = $json.From.trim();
     let sender_name = null;

     if (sender_email.includes('<')) {
       sender_name = sender_email.split('<')[0].trim();
       sender_email = sender_email.split('<')[1].replace('>', '').trim();
     }

     // Extract Recipient Info
     let recipient_email = $json.To.trim();
     let recipient_name = null;

     if (recipient_email.includes('<')) {
       recipient_name = recipient_email.split('<')[0].trim();
       recipient_email = recipient_email.split('<')[1].replace('>', '').trim();
     }

     return {
       "sender_name": sender_name,
       "sender_email": sender_email,
       "recipient_name": recipient_name,
       "recipient_email": recipient_email
     }
     ```

   - Name this node `Get Client Name and Email`.

3. **Add MySQL Node to Upsert Data:**  
   - Add a `MySQL` node connected from `Get Client Name and Email`.  
   - Configure operation as `upsert`.  
   - Set database credentials with your MySQL instance (create credentials beforehand).  
   - Select the target database and table named `contacts` (table must exist).  
   - Set `columnToMatchOn` to `messageId` (the unique Gmail message ID column).  
   - Map `valueToMatchOn` to `={{ $('Receive Email').item.json.id }}`.  
   - Define columns/values to insert or update:  
     - `threadId`: `={{ $('Receive Email').item.json.threadId }}`  
     - `sender_name`: `={{ $json.sender_name }}`  
     - `sender_email`: `={{ $json.sender_email }}`  
     - `recipient_name`: `={{ $json.recipient_name }}`  
     - `recipient_email`: `={{ $json.recipient_email }}`  
     - `subject`: `={{ $('Receive Email').item.json.Subject }}`  
     - `snippet`: `={{ $('Receive Email').item.json.snippet }}`  
   - Set `onError` to continue to prevent workflow failure on DB errors.  
   - Name this node `Insert New Client in MySQL`.

4. **Add Sticky Notes for Documentation (Optional):**  
   - Create two Sticky Note nodes describing workflow setup and table requirements.  
   - Position them logically near relevant nodes for clarity.

5. **Connect Nodes:**  
   - Connect `Receive Email` → `Get Client Name and Email` → `Insert New Client in MySQL`.

6. **Validate and Save Workflow:**  
   - Test Gmail trigger with a new email.  
   - Monitor MySQL table for inserted or updated records.  
   - Adjust polling frequency or filters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow requires a MySQL table named `contacts` with columns: `messageId` (unique Gmail message ID), `threadId`, `snippet`, `sender_name` (nullable), `sender_email`, `recipient_name` (nullable), `recipient_email`, and `subject` (nullable). Ensure the table schema aligns exactly to prevent SQL errors.                                                                                                | Table schema requirement                                  |
| Gmail OAuth2 credentials must have the appropriate Gmail API scopes to read emails (e.g., `https://www.googleapis.com/auth/gmail.readonly`). Token refresh setup is recommended to avoid trigger failures due to expired tokens.                                                                                                                                                                               | Gmail OAuth2 setup                                        |
| The current code node parses only one recipient from the `To` field and assumes standard email header formatting. For emails with multiple recipients, or more complex headers (e.g., CC, BCC), additional parsing logic is required.                                                                                                                                                                            | Potential code enhancement                                 |
| To extend functionality, consider adding nodes to extract attachments, labels, or timestamps from Gmail messages and store them in the database.                                                                                                                                                                                                                                                         | Customization suggestion                                  |
| On large volumes of emails, consider rate limits and quotas of Gmail API and MySQL performance when setting polling intervals and batch sizes.                                                                                                                                                                                                                                                           | Performance consideration                                 |

---

Disclaimer:  
The provided text is generated exclusively from an automated n8n workflow and fully complies with content policies. It contains no illegal, offensive, or protected material. All processed data is legal and public.