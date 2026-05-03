Sync Gmail emails to PostgreSQL with S3 attachment storage

https://n8nworkflows.xyz/workflows/sync-gmail-emails-to-postgresql-with-s3-attachment-storage-8871


# Sync Gmail emails to PostgreSQL with S3 attachment storage

---

### 1. Workflow Overview

This workflow is designed for automated processing and archiving of Gmail emails into a PostgreSQL database with attachments backed up to S3/MinIO storage. It targets businesses or individuals who need a searchable and structured archive of email communications, including attachments, to support compliance, analytics, or integration with other business systems.

**Logical Blocks:**

- **1.1 System Setup and Triggers:** Credential and schedule triggers setup for detecting new emails individually or in bulk.
- **1.2 Email Retrieval:** Fetches emails (individual or bulk) including metadata and attachments using Gmail API.
- **1.3 Attachment Processing:** Filters emails with attachments, extracts binary metadata, splits attachments for upload.
- **1.4 Storage Upload and Aggregation:** Uploads attachments to S3/MinIO, aggregates attachments linked to emails.
- **1.5 Data Transformation:** Converts Gmail API email data and attachments into PostgreSQL-compatible structured JSON.
- **1.6 Database Storage:** Inserts or updates transformed email records in PostgreSQL with UPSERT and batch processing.

---

### 2. Block-by-Block Analysis

#### 2.1 System Setup and Triggers

- **Overview:** Establishes prerequisites and triggers to initiate email processing. Supports both real-time and scheduled batch email fetching.
- **Nodes Involved:**
  - `SYSTEM OVERVIEW` (Sticky Note)
  - `SETUP REQUIREMENTS` (Sticky Note)
  - `SYSTEM TRIGGERS` (Sticky Note)
  - `Gmail Trigger`
  - `Schedule Trigger`

- **Node Details:**

  - **SYSTEM OVERVIEW**  
    - Type: Sticky Note  
    - Role: Summarizes workflow purpose and technology stack.  
    - No inputs or outputs. Contains descriptive content only.

  - **SETUP REQUIREMENTS**  
    - Type: Sticky Note  
    - Role: Details credential setup for Gmail OAuth2, PostgreSQL, and S3/MinIO; also includes PostgreSQL schema.  
    - No inputs or outputs.

  - **SYSTEM TRIGGERS**  
    - Type: Sticky Note  
    - Role: Describes trigger node configuration requirements and setup instructions.  
    - No inputs or outputs.

  - **Gmail Trigger**  
    - Type: Gmail Trigger Node  
    - Role: Detects new individual incoming emails every 30 minutes.  
    - Config: `downloadAttachments: true`, `simple: false`, Gmail OAuth2 credential with `gmail.readonly` scope.  
    - Output: Emits new email ID and metadata to downstream nodes.  
    - Potential failures: OAuth token expiration, API rate limits, network errors.

  - **Schedule Trigger**  
    - Type: Schedule Trigger Node  
    - Role: Triggers workflow execution every hour to fetch bulk emails (sent and inbox).  
    - Config: Interval set to 1 hour.  
    - Output: Triggers bulk email retrieval nodes.  
    - Potential failures: None intrinsic; depends on downstream node errors.

#### 2.2 Email Retrieval

- **Overview:** Retrieves emails from Gmail in full detail, including attachments, for both individual and bulk scenarios.
- **Nodes Involved:**
  - `Get Individual Message`
  - `Get Many Sent`
  - `Get Many Inbox`
  - `Flow Convergence`

- **Node Details:**

  - **Get Individual Message**  
    - Type: Gmail Node (Get operation)  
    - Role: Fetches complete email data for a specific message ID triggered by Gmail Trigger.  
    - Config: `simple: false`, `downloadAttachments: true`, Gmail OAuth2 credential.  
    - Input: Message ID from Gmail Trigger.  
    - Output: Full email data including headers, body, labels, and attachments as binary.  
    - Failures: Invalid message ID, API errors, OAuth failures.

  - **Get Many Sent**  
    - Type: Gmail Node (GetAll operation)  
    - Role: Retrieves all sent emails from the last hour triggered by Schedule Trigger.  
    - Config: Query `"in:sent newer_than:1h"`, `returnAll: true`, `downloadAttachments: true`, `simple: false`.  
    - Output: Array of sent emails with full metadata and attachments.  
    - Failures: API limits, network errors.

  - **Get Many Inbox**  
    - Type: Gmail Node (GetAll operation)  
    - Role: Retrieves all inbox emails from the last hour triggered by Schedule Trigger.  
    - Config: Query `"in:inbox newer_than:1h"`, `returnAll: true`, `downloadAttachments: true`, `simple: false`.  
    - Output: Array of inbox emails.  
    - Failures: Same as Get Many Sent.

  - **Flow Convergence**  
    - Type: NoOp Node  
    - Role: Merges output streams from individual and bulk email retrieval nodes into a single flow.  
    - Inputs: From `Get Individual Message`, `Get Many Sent`, `Get Many Inbox`.  
    - Outputs: Passes data to attachment filtering and transformation nodes.  
    - Failures: None; used for flow control.

#### 2.3 Attachment Processing

- **Overview:** Filters emails containing attachments, extracts detailed metadata from binary files, and splits attachments into individual items for upload.
- **Nodes Involved:**
  - `Binary Data Filter`
  - `Extract Binary Metadata`
  - `Split Attachments`
  - `SCRIPT: Extract Binary`

- **Node Details:**

  - **Binary Data Filter**  
    - Type: Filter Node  
    - Role: Filters only items that contain binary data (attachments).  
    - Condition: Checks if binary data exists in any of the possible sources (`Get Individual Message`, `Get Many Sent`, `Get Many Inbox`).  
    - Inputs: Combined email data from Flow Convergence.  
    - Outputs: Items with binary attachments to the next node; others proceed to transformation.  
    - Failures: Expression evaluation errors if input format changes.

  - **Extract Binary Metadata (Code Node)**  
    - Type: Code Node  
    - Role: Iterates over binary attachments to extract metadata such as filename, MIME type, size, extension, encoding, and file type category.  
    - Input: Items filtered with binary data.  
    - Output: Items enriched with attachment metadata summary and list.  
    - Failures: JavaScript runtime errors, missing binary data.

  - **Split Attachments**  
    - Type: SplitOut Node  
    - Role: Splits the list of attachments into separate items to process individually for upload.  
    - Input: Attachment metadata from previous node.  
    - Output: Single attachment per item with reference to original email ID.  
    - Failures: Empty attachment arrays or missing fields.

  - **SCRIPT: Extract Binary** (Sticky Note)  
    - Describes the detailed process of metadata extraction and summary creation from binary attachment data.

#### 2.4 Storage Upload and Aggregation

- **Overview:** Uploads individual attachments to S3/MinIO storage and aggregates attachments by email for database linking.
- **Nodes Involved:**
  - `Merge Attachments`
  - `Upload to Storage`
  - `Structure Fields`
  - `Aggregate Attachments`
  - `Final Merge`

- **Node Details:**

  - **Merge Attachments**  
    - Type: Merge Node (Combine mode)  
    - Role: Merges individual attachment items with their original email using `originalJson.id` = `id` as merge key.  
    - Inputs: Output from `Split Attachments` and emails with attachment metadata.  
    - Output: Combined email-attachment pairs ready for upload.  
    - Failures: Key mismatch or missing merge keys.

  - **Upload to Storage**  
    - Type: S3 Node (Upload operation)  
    - Role: Uploads each attachment to S3/MinIO bucket `gmail-attachments`.  
    - File path format: `/emailuser/{messageId}/{fileName}`  
    - Binary property: Uses attachment binary key dynamically.  
    - Credentials: Configured S3/MinIO credentials.  
    - Failures: Authentication errors, upload failures, network timeouts.

  - **Structure Fields**  
    - Type: Set Node  
    - Role: Prepares structured JSON object containing uploaded attachment data and original email message ID for aggregation.  
    - Output: JSON including `attachments` and `messageId`.  
    - Failures: Expression errors if referenced nodes change.

  - **Aggregate Attachments**  
    - Type: Aggregate Node  
    - Role: Aggregates all attachments grouped by `messageId` into a field named `adjunto`.  
    - Input: Structured attachment items from `Structure Fields`.  
    - Output: Aggregated attachment arrays per email.  
    - Failures: Aggregation errors with empty inputs.

  - **Final Merge**  
    - Type: Merge Node  
    - Role: Combines aggregated attachments with transformed emails for final processing.  
    - Inputs: Aggregated attachments and transformed email data.  
    - Output: Final combined email + attachment dataset.  
    - Failures: Merge key inconsistencies.

  - **Sticky Note: DATA MAPPING**  
    - Describes logic to merge attachments and emails, upload to storage, and prepare final data structure.

#### 2.5 Data Transformation

- **Overview:** Converts raw Gmail API email data into normalized PostgreSQL-compatible JSON including linked attachment metadata.
- **Nodes Involved:**
  - `Email to PostgreSQL Transform`
  - `Link Email Attachments`
  - `SCRIPT: Email Transform`

- **Node Details:**

  - **Email to PostgreSQL Transform (Code Node)**  
    - Type: Code Node  
    - Role: Normalizes sender and recipients, converts HTML to plain text, sets flags (`is_read`, `is_outgoing`), and formats timestamps.  
    - Input: Raw Gmail API email data.  
    - Output: Structured email JSON ready for database insertion.  
    - Key Expressions: Normalizes email address objects; extracts body text prioritizing plain text fields.  
    - Failures: Parsing errors, invalid date formats.

  - **Link Email Attachments (Code Node)**  
    - Type: Code Node  
    - Role: Links aggregated attachments to their respective emails by `messageId`, converts attachments array to JSON string.  
    - Input: Combined datasets of emails and attachments.  
    - Output: Final email records with attachment JSON strings for PostgreSQL.  
    - Failures: JSON stringify errors, missing attachments.

  - **SCRIPT: Email Transform** (Sticky Note)  
    - Details step-by-step transformations from Gmail format to PostgreSQL schema.

#### 2.6 Database Storage

- **Overview:** Inserts or updates the final transformed email records into PostgreSQL using batch processing with UPSERT to avoid duplicates.
- **Nodes Involved:**
  - `Process Items Loop`
  - `Insert or Update Database`
  - `End Loop`

- **Node Details:**

  - **Process Items Loop**  
    - Type: SplitInBatches Node  
    - Role: Processes email records one by one to prevent timeouts on bulk inserts.  
    - Input: Final transformed emails with attachments.  
    - Output: Single email JSON per iteration.  
    - Failures: Timeout or batch size misconfiguration.

  - **Insert or Update Database**  
    - Type: PostgreSQL Node (Upsert operation)  
    - Role: Inserts or updates email record in `messages` table keyed by `message_id`.  
    - Config: Uses PostgreSQL credentials with write permissions; JSONB fields for `sender`, `recipients`, `labels`, `attachments`.  
    - Input: Single email JSON from loop.  
    - Output: Confirmation of DB operation.  
    - Failures: DB connection errors, constraint violations, permission issues.

  - **End Loop**  
    - Type: NoOp Node  
    - Role: Marks completion of batch processing loop.  
    - Input: Loop iteration output.  
    - Output: Workflow end for looped batch.  
    - Failures: None.

  - **Sticky Note: DATABASE STORAGE**  
    - Describes rationale for batch processing, UPSERT usage, and performance considerations.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                          |
|-------------------------|-------------------------|----------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| SYSTEM OVERVIEW         | Sticky Note             | Workflow purpose summary                |                              |                               | Automated Gmail Email Processing System, integration overview                                                        |
| SETUP REQUIREMENTS      | Sticky Note             | Credential and DB schema setup          |                              |                               | Details Gmail OAuth2, PostgreSQL setup, and SQL schema                                                               |
| SYSTEM TRIGGERS         | Sticky Note             | Trigger setup instructions              |                              |                               | Describes Gmail and Schedule triggers configuration                                                                  |
| Gmail Trigger           | Gmail Trigger           | Real-time single email detection        |                              | Get Individual Message         | Requires Gmail OAuth2 with gmail.readonly                                                                             |
| Schedule Trigger        | Schedule Trigger        | Bulk email fetch every hour             |                              | Get Many Sent, Get Many Inbox  |                                                                                                                      |
| Get Individual Message  | Gmail                   | Fetch single detailed email             | Gmail Trigger                | Flow Convergence              | Downloads attachments, full email data                                                                                |
| Get Many Sent           | Gmail                   | Fetch sent emails from last hour        | Schedule Trigger             | Flow Convergence              | Query: in:sent newer_than:1h, downloads attachments                                                                  |
| Get Many Inbox          | Gmail                   | Fetch inbox emails from last hour       | Schedule Trigger             | Flow Convergence              | Query: in:inbox newer_than:1h, downloads attachments                                                                  |
| Flow Convergence        | NoOp                    | Merges multiple retrieval outputs       | Get Individual Message, Get Many Sent, Get Many Inbox | Binary Data Filter, Merge Attachments, Email to PostgreSQL Transform |                                                                                                                      |
| Binary Data Filter      | Filter                  | Filters emails with attachments         | Flow Convergence             | Extract Binary Metadata       | Checks presence of binary data in items                                                                               |
| Extract Binary Metadata | Code                    | Extracts metadata from binary attachments | Binary Data Filter           | Split Attachments             | Extracts filename, mimeType, size, extension, encoding                                                                |
| Split Attachments       | SplitOut                | Splits attachments into individual items | Extract Binary Metadata      | Merge Attachments             |                                                                                                                      |
| Merge Attachments       | Merge                   | Links individual attachments to emails  | Split Attachments            | Upload to Storage             | Merges on originalJson.id = id                                                                                        |
| Upload to Storage       | S3                      | Uploads attachments to S3/MinIO          | Merge Attachments            | Structure Fields              | Bucket: gmail-attachments, path: /emailuser/{messageId}/{fileName}                                                   |
| Structure Fields        | Set                     | Prepares JSON for aggregation            | Upload to Storage            | Aggregate Attachments         | Sets `attachments` and `messageId` fields                                                                             |
| Aggregate Attachments   | Aggregate               | Groups attachments by email              | Structure Fields             | Final Merge                  | Aggregates all attachments into `adjunto` field                                                                       |
| Final Merge             | Merge                   | Combines emails with aggregated attachments | Aggregate Attachments, Email to PostgreSQL Transform | Link Email Attachments         |                                                                                                                      |
| Email to PostgreSQL Transform | Code              | Normalizes and structures email data     | Flow Convergence             | Final Merge                  | Converts Gmail format to DB-ready JSON                                                                                |
| Link Email Attachments  | Code                    | Links attachments JSON to emails         | Final Merge                  | Process Items Loop           | Converts attachments array to JSON string                                                                             |
| Process Items Loop      | SplitInBatches          | Processes emails one by one               | Link Email Attachments       | Insert or Update Database, End Loop | Prevents timeout, batch size default                                                                                   |
| Insert or Update Database | PostgreSQL             | UPSERT email records into DB              | Process Items Loop           | Process Items Loop           | Uses `message_id` as key, supports JSONB fields                                                                       |
| End Loop               | NoOp                    | Marks end of batch processing             | Process Items Loop           |                               |                                                                                                                      |
| SCRIPT: Extract Binary  | Sticky Note             | Details extraction of binary metadata     |                              |                               | Explains metadata extraction logic                                                                                     |
| SCRIPT: Email Transform | Sticky Note             | Details email data transformation         |                              |                               | Explains Gmail to PostgreSQL data conversion logic                                                                     |
| DATA MAPPING            | Sticky Note             | Describes attachment merging and upload   |                              |                               |                                                                                                                      |
| DATABASE STORAGE        | Sticky Note             | Describes DB storage logic and performance |                              |                               |                                                                                                                      |
| Sticky Note             | Sticky Note             | High-level workflow description            |                              |                               | Sync Gmail emails to PostgreSQL with S3 attachment storage, includes setup and customization notes                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**
   - Add notes titled:
     - "SYSTEM OVERVIEW" with purpose and technology stack.
     - "SETUP REQUIREMENTS" listing Gmail OAuth2, PostgreSQL schema, and S3 setup steps.
     - "SYSTEM TRIGGERS" describing trigger nodes and credential requirements.
     - "EMAIL RETRIEVAL" describing the Gmail nodes.
     - "ATTACHMENT PROCESSING", "DATA MAPPING", "DATA TRANSFORMATION", and "DATABASE STORAGE" explaining each block's logic.

2. **Add Triggers:**
   - Create a **Gmail Trigger** node:
     - Configure with Gmail OAuth2 credentials.
     - Set `downloadAttachments` to true.
     - Poll every 30 minutes.
   - Create a **Schedule Trigger** node:
     - Set interval to every 1 hour.
     - No credentials needed.

3. **Create Gmail Retrieval Nodes:**
   - Add **Get Individual Message** node:
     - Operation: Get
     - Message ID: `={{ $json.id }}`
     - Enable `downloadAttachments`
     - Use Gmail OAuth2 credential.
     - Connect Gmail Trigger → Get Individual Message.
   - Add **Get Many Sent** node:
     - Operation: Get All
     - Query: `"in:sent newer_than:1h"`
     - Enable `downloadAttachments`
     - Return all emails.
     - Connect Schedule Trigger → Get Many Sent.
   - Add **Get Many Inbox** node:
     - Operation: Get All
     - Query: `"in:inbox newer_than:1h"`
     - Enable `downloadAttachments`
     - Return all emails.
     - Connect Schedule Trigger → Get Many Inbox.

4. **Add Flow Convergence:**
   - Insert **NoOp** node.
   - Connect outputs of `Get Individual Message`, `Get Many Sent`, and `Get Many Inbox` to this NoOp node.

5. **Filter Emails with Attachments:**
   - Add **Filter** node named `Binary Data Filter`.
   - Condition: Check if binary data exists on the incoming items.
   - Connect Flow Convergence → Binary Data Filter.

6. **Extract Binary Metadata:**
   - Add a **Code** node named `Extract Binary Metadata`.
   - Paste the provided JS code that extracts attachment metadata.
   - Connect Binary Data Filter → Extract Binary Metadata.

7. **Split Attachments:**
   - Add **SplitOut** node named `Split Attachments`.
   - Field to split: `attachments`.
   - Include `originalJson.id` field.
   - Connect Extract Binary Metadata → Split Attachments.

8. **Merge Attachments to Emails:**
   - Add **Merge** node named `Merge Attachments`.
   - Mode: Combine.
   - Merge by fields: `originalJson.id` (left) and `id` (right).
   - Connect Split Attachments → Merge Attachments.

9. **Upload Attachments to S3/MinIO:**
   - Add **S3** node named `Upload to Storage`.
   - Operation: Upload.
   - Bucket: `gmail-attachments`.
   - FileName: `/emailuser/{{ $json["originalJson.id"] }}/{{ $json.attachments.fileName }}`.
   - Binary property name: `={{ $json.attachments.key }}`.
   - Configure S3 credential.
   - Connect Merge Attachments → Upload to Storage.

10. **Prepare Attachment Structure:**
    - Add **Set** node named `Structure Fields`.
    - Set JSON output to include:
      - `attachments`: `{{ $('Merge Attachments').item.json.attachments }}`
      - `messageId`: `{{ $('Merge Attachments').item.json['originalJson.id'] }}`
    - Connect Upload to Storage → Structure Fields.

11. **Aggregate Attachments:**
    - Add **Aggregate** node named `Aggregate Attachments`.
    - Aggregate all item data into field named `adjunto`.
    - Connect Structure Fields → Aggregate Attachments.

12. **Email Data Transformation:**
    - Add **Code** node named `Email to PostgreSQL Transform`.
    - Paste JS code transforming Gmail API emails to PostgreSQL format.
    - Connect Flow Convergence → Email to PostgreSQL Transform.

13. **Final Merge:**
    - Add **Merge** node named `Final Merge`.
    - Connect Aggregate Attachments → Final Merge (input 1).
    - Connect Email to PostgreSQL Transform → Final Merge (input 2).

14. **Link Attachments to Emails:**
    - Add **Code** node named `Link Email Attachments`.
    - Paste JS code linking attachments JSON string to emails.
    - Connect Final Merge → Link Email Attachments.

15. **Batch Processing Loop:**
    - Add **SplitInBatches** node named `Process Items Loop`.
    - Connect Link Email Attachments → Process Items Loop.

16. **Upsert to PostgreSQL:**
    - Add **PostgreSQL** node named `Insert or Update Database`.
    - Operation: Upsert.
    - Table: `messages`.
    - Schema: `public`.
    - Matching column: `message_id`.
    - Configure PostgreSQL credentials with write access.
    - Connect Process Items Loop → Insert or Update Database.

17. **End Loop:**
    - Add **NoOp** node named `End Loop`.
    - Connect Process Items Loop → End Loop (second output).

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                       |
|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------|
| Workflow integrates Gmail API, PostgreSQL, and S3/MinIO for automated email archiving and attachment backup.     | SYSTEM OVERVIEW sticky note                           |
| Gmail OAuth2 requires `gmail.readonly` scope and proper OAuth setup in Google Cloud Console.                      | SETUP REQUIREMENTS sticky note                        |
| PostgreSQL table `messages` must use JSONB fields for flexible storage of sender, recipients, labels, attachments. | SETUP REQUIREMENTS sticky note                        |
| Attachments are stored in bucket `gmail-attachments` under path `/emailuser/{messageId}/{fileName}`.              | DATA MAPPING sticky note                              |
| Batch processing uses SplitInBatches to avoid timeouts with large datasets.                                        | DATABASE STORAGE sticky note                          |
| The workflow processes emails from the last hour to limit volume and system load; adjust time frames as needed.   | Sticky Note at workflow start                          |
| Update `authenticatedUserEmail` variable in transform code node to your email address for correct `is_outgoing` flag. | Email to PostgreSQL Transform code node               |

---

**Disclaimer:** The provided description and analysis are derived exclusively from the n8n workflow JSON data and respect all content policies. No illegal, offensive, or protected data is included; all data handled is legal and public.

---