Gmail to Vector Embeddings with PGVector and Ollama

https://n8nworkflows.xyz/workflows/gmail-to-vector-embeddings-with-pgvector-and-ollama-3762


# Gmail to Vector Embeddings with PGVector and Ollama

### 1. Workflow Overview

This workflow enables the extraction, storage, and vectorization of Gmail emails using local open-source tools, allowing semantic search capabilities on your email content. It is designed for anyone who wants to query their email history semantically, for example, to recall specific information like hotel stays or academic marks, by leveraging vector similarity searches.

The workflow is structured into the following logical blocks:

- **1.1 Initialization & Bulk Import Setup**  
  Prepares the environment by creating necessary database tables and defining the time intervals for bulk email import based on the user’s Gmail account creation date.

- **1.2 Bulk Email Import (Manual Triggered)**  
  Retrieves emails in monthly batches from Gmail, extracts relevant fields, stores structured metadata in Postgres, generates vector embeddings using Ollama and nomic-embed-text, and stores these embeddings in a pgvector-enabled Postgres table.

- **1.3 Continuous Email Import (Gmail Trigger)**  
  Listens for new incoming emails in real-time, processes them similarly to the bulk import, and updates the database accordingly.

- **1.4 Data Processing and Vectorization Pipeline**  
  Transforms raw email data into structured metadata and vector embeddings, including splitting email text into manageable chunks for embedding.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Bulk Import Setup

**Overview:**  
This block initializes the database schema and calculates the weekly intervals from the user’s Gmail account creation date to the current date to enable batch processing of emails.

**Nodes Involved:**  
- Manual Trigger  
- Create the table  
- Explode interval into weeks  
- Split Out  
- Loop Over Items  
- Set before and after dates

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Starts the workflow manually for bulk import  
  - Configuration: Default, no parameters  
  - Connections: Outputs to "Create the table"  
  - Edge Cases: None specific; user must trigger manually

- **Create the table**  
  - Type: Postgres node (executeQuery)  
  - Role: Creates the `emails_metadata` table if it does not exist  
  - Configuration: SQL query to create table with columns for email metadata (id, thread id, from, to, cc, date, subject, text, attachments)  
  - Connections: Outputs to "Explode interval into weeks"  
  - Edge Cases: SQL execution errors, permission issues

- **Explode interval into weeks**  
  - Type: Code node (JavaScript)  
  - Role: Generates an array of ISO date strings representing the first day of each week from the Gmail account creation date to now  
  - Configuration: User must edit the `whenDidICreateMyGmailAccount` variable to their Gmail account creation date (e.g., '2013-11-01')  
  - Key Expressions: Uses Luxon DateTime and Interval to calculate weekly intervals  
  - Connections: Outputs to "Split Out"  
  - Edge Cases: Incorrect date format, infinite loops if logic altered

- **Split Out**  
  - Type: SplitOut node  
  - Role: Splits the array of weekly dates into individual items for batch processing  
  - Configuration: Splits on field `weeks` into field `after`  
  - Connections: Outputs to "Loop Over Items"  
  - Edge Cases: Empty input array

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Role: Controls batch processing of weekly intervals  
  - Configuration: Default batch size (processes one week at a time)  
  - Connections: On first output, no connection; on second output, to "Set before and after dates"  
  - Edge Cases: Batch processing errors, empty batches

- **Set before and after dates**  
  - Type: Set node  
  - Role: Defines the `after` and `before` date range for each batch (one week interval)  
  - Configuration:  
    - `after` = current batch date (from split)  
    - `before` = `after` + 1 week (ISO date format)  
  - Connections: Outputs to "Get a batch of messages"  
  - Edge Cases: Date calculation errors

---

#### 1.2 Bulk Email Import (Manual Triggered)

**Overview:**  
Processes emails in weekly batches fetched from Gmail, extracting fields, storing structured data, and vectorizing email content.

**Nodes Involved:**  
- Get a batch of messages  
- Extract email fields  
- Store structured  
- Embeddings Ollama  
- Recursive Character Text Splitter  
- Default Data Loader  
- Store vectorized  
- Was manually triggered?  
- No Operation, do nothing

**Node Details:**

- **Get a batch of messages**  
  - Type: Gmail node (getAll operation)  
  - Role: Retrieves all emails received between `after` and `before` dates  
  - Configuration:  
    - Filters: `receivedAfter` and `receivedBefore` from Set node  
    - Downloads attachments  
    - Returns all matching emails  
  - Connections: Outputs to "Extract email fields"  
  - Edge Cases: Gmail API rate limits, auth errors, empty results

- **Extract email fields**  
  - Type: Set node  
  - Role: Extracts and normalizes key email fields for storage and embedding  
  - Configuration: Extracts fields such as:  
    - `email_text` (email body text)  
    - `email_from`, `email_to`, `email_cc` (email addresses)  
    - `date` (ISO formatted)  
    - `email_id`, `thread_id` (unique identifiers)  
    - `email_subject`  
    - `attachments` (array of attachment filenames)  
  - Connections: Outputs to "Store structured"  
  - Edge Cases: Missing fields, malformed emails

- **Store structured**  
  - Type: Postgres node (upsert operation)  
  - Role: Inserts or updates email metadata into `emails_metadata` table  
  - Configuration:  
    - Upsert on `email_id`  
    - Maps all extracted fields to columns  
    - Continues on error (to avoid stopping batch)  
  - Connections: Outputs to "Store vectorized" and on error back to "Loop Over Items"  
  - Edge Cases: DB connection issues, constraint violations

- **Embeddings Ollama**  
  - Type: Langchain embeddings node (Ollama)  
  - Role: Generates vector embeddings of email text using `nomic-embed-text:latest` model  
  - Configuration: Model set to `nomic-embed-text:latest`  
  - Connections: Outputs embeddings to "Store vectorized" (ai_embedding input)  
  - Edge Cases: Ollama server unavailability, model errors

- **Recursive Character Text Splitter**  
  - Type: Langchain text splitter node  
  - Role: Splits email text into chunks of 2000 characters with 50 character overlap for embedding  
  - Configuration: chunkSize=2000, chunkOverlap=50  
  - Connections: Outputs to "Default Data Loader" (ai_textSplitter input)  
  - Edge Cases: Very short or empty text

- **Default Data Loader**  
  - Type: Langchain document loader node  
  - Role: Prepares chunks of text with metadata for vector embedding  
  - Configuration:  
    - Metadata includes `emails_metadata.id` and `emails_metadata.thread_id` from extracted fields  
    - Loads `email_text` field as document content  
  - Connections: Outputs to "Store vectorized" (ai_document input)  
  - Edge Cases: Metadata missing or malformed

- **Store vectorized**  
  - Type: Langchain vector store node (PGVector)  
  - Role: Inserts vector embeddings into `emails_embeddings` table in Postgres  
  - Configuration:  
    - Insert mode  
    - Table name: `emails_embeddings`  
    - Continues on error  
  - Connections:  
    - On success: to "Was manually triggered?"  
    - On error: to "Loop Over Items"  
  - Edge Cases: DB connection issues, vector store errors

- **Was manually triggered?**  
  - Type: If node  
  - Role: Checks if the workflow was triggered manually or by Gmail trigger  
  - Configuration: Condition checks if `Manual Trigger` node was executed  
  - Connections:  
    - If false: to "No Operation, do nothing"  
    - If true: to "Loop Over Items" (to continue batch processing)  
  - Edge Cases: Expression evaluation errors

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Role: Ends workflow branch when triggered by Gmail Trigger (not manual)  
  - Configuration: Default  
  - Connections: None  
  - Edge Cases: None

---

#### 1.3 Continuous Email Import (Gmail Trigger)

**Overview:**  
Listens for new incoming emails in the Gmail inbox every minute and processes them immediately through the same extraction, storage, and vectorization pipeline.

**Nodes Involved:**  
- Gmail Trigger  
- Extract email fields  
- Store structured  
- Embeddings Ollama  
- Recursive Character Text Splitter  
- Default Data Loader  
- Store vectorized  
- Was manually triggered?  
- No Operation, do nothing

**Node Details:**

- **Gmail Trigger**  
  - Type: Gmail Trigger node  
  - Role: Watches Gmail inbox label "INBOX" every minute for new emails  
  - Configuration:  
    - Filters: labelIds = ["INBOX"]  
    - Downloads attachments  
    - Polls every minute  
  - Connections: Outputs to "Extract email fields"  
  - Edge Cases: Gmail API limits, auth errors, missed triggers

- Remaining nodes are the same as in Bulk Email Import block, processing each new email as it arrives.

---

#### 1.4 Data Processing and Vectorization Pipeline

**Overview:**  
Transforms raw email text into vector embeddings with metadata, enabling semantic search capabilities, while also storing structured metadata for relational queries.

**Nodes Involved:**  
- Recursive Character Text Splitter  
- Default Data Loader  
- Embeddings Ollama  
- Store vectorized

**Node Details:**

- **Recursive Character Text Splitter**  
  - Splits email text into overlapping chunks to optimize embedding quality and handle long emails.

- **Default Data Loader**  
  - Loads chunks with metadata for embedding.

- **Embeddings Ollama**  
  - Generates embeddings using local Ollama model `nomic-embed-text`.

- **Store vectorized**  
  - Inserts embeddings into Postgres pgvector table `emails_embeddings`.

---

### 3. Summary Table

| Node Name                  | Node Type                                    | Functional Role                                  | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                        |
|----------------------------|----------------------------------------------|-------------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger             | Manual trigger                               | Starts bulk import manually                      |                              | Create the table             |                                                                                                                                   |
| Create the table           | Postgres (executeQuery)                      | Creates emails_metadata table                    | Manual Trigger               | Explode interval into weeks  |                                                                                                                                   |
| Explode interval into weeks| Code                                         | Generates weekly intervals from account creation| Create the table             | Split Out                   |                                                                                                                                   |
| Split Out                 | SplitOut                                     | Splits weeks array into individual items        | Explode interval into weeks  | Loop Over Items              |                                                                                                                                   |
| Loop Over Items           | SplitInBatches                               | Controls batch processing of weekly intervals   | Split Out                   | Set before and after dates   |                                                                                                                                   |
| Set before and after dates| Set                                          | Defines date range for batch                      | Loop Over Items             | Get a batch of messages      |                                                                                                                                   |
| Get a batch of messages   | Gmail                                        | Retrieves emails for date range                   | Set before and after dates  | Extract email fields         |                                                                                                                                   |
| Extract email fields      | Set                                          | Extracts and normalizes email metadata           | Get a batch of messages, Gmail Trigger | Store structured            |                                                                                                                                   |
| Store structured          | Postgres (upsert)                            | Stores structured email metadata                  | Extract email fields        | Store vectorized             |                                                                                                                                   |
| Embeddings Ollama         | Langchain embeddings Ollama                   | Generates vector embeddings using Ollama model   | Default Data Loader         | Store vectorized (ai_embedding) |                                                                                                                                   |
| Recursive Character Text Splitter | Langchain text splitter                 | Splits email text into chunks                      |                            | Default Data Loader          |                                                                                                                                   |
| Default Data Loader       | Langchain document loader                     | Loads text chunks with metadata for embedding    | Recursive Character Text Splitter | Store vectorized (ai_document) |                                                                                                                                   |
| Store vectorized          | Langchain vector store PGVector               | Inserts vector embeddings into Postgres pgvector | Store structured, Embeddings Ollama, Default Data Loader | Was manually triggered?      |                                                                                                                                   |
| Was manually triggered?   | If                                            | Checks if workflow was manually triggered        | Store vectorized            | No Operation, do nothing; Loop Over Items |                                                                                                                                   |
| No Operation, do nothing  | NoOp                                           | Ends branch if triggered by Gmail Trigger        | Was manually triggered?     |                              |                                                                                                                                   |
| Gmail Trigger             | Gmail Trigger                                  | Watches for new emails every minute               |                              | Extract email fields         | Activate the workflow and this trigger will check for new mail every minute                                                      |
| Sticky Note               | Sticky Note                                    | Bulk import instructions                          |                              |                              | ## Bulk e-mail import\n\nPress the `Test workflow` button to run this once, and bulk import of all your e-mail\n\n### IMPORTANT\nSpecify your Gmail account creation date by editing the code node |
| Sticky Note1              | Sticky Note                                    | Gmail account creation date reminder              |                              |                              | ## Edit this ⬇️\nAnd specify your Gmail account creation date                                                                     |
| Sticky Note2              | Sticky Note                                    | Workflow activation reminder                       |                              |                              | ## Activate the workflow\nAnd this trigger will check for new mail, every minute                                                  |
| Sticky Note3              | Sticky Note                                    | Explanation of structured and vectorized storage |                              |                              | ## Magic here 🪄\n#### (not really, just statistics)\nE-mail is stored in a `emails_metadata` structured table, and also fed to the [`nomic-embed-text`](https://ollama.com/library/nomic-embed-text) model to be stored in a `emails_embeddings` table as [vector embeddings](https://www.pinecone.io/learn/vector-embeddings/)... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start bulk import manually.

2. **Create Postgres Node "Create the table"**  
   - Type: Postgres (executeQuery)  
   - Credentials: Configure Postgres credentials with access to your database.  
   - Query:  
     ```sql
     CREATE TABLE IF NOT EXISTS public.emails_metadata (
       email_id character varying(64) NOT NULL,
       thread_id character varying(64),
       email_from text,
       email_to text,
       email_cc text,
       date timestamp with time zone NOT NULL,
       email_subject text,
       email_text text,
       attachments text[]
     );
     ```  
   - Connect Manual Trigger → Create the table

3. **Create Code Node "Explode interval into weeks"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     let whenDidICreateMyGmailAccount = DateTime.fromISO('2013-11-01'); // Edit this date
     whenDidICreateMyGmailAccount = whenDidICreateMyGmailAccount.set({day: 1});
     let now = $now.set({day: 1});
     const weeks = [];
     while (Math.floor(Interval.fromDateTimes(whenDidICreateMyGmailAccount, now).length('weeks')) > -1) {
       weeks.push(now.toISODate());
       now = now.minus({weeks: 1});
     }
     return {json: { weeks }};
     ```  
   - Connect Create the table → Explode interval into weeks

4. **Create SplitOut Node "Split Out"**  
   - Type: SplitOut  
   - Field to split out: `weeks`  
   - Destination field name: `after`  
   - Connect Explode interval into weeks → Split Out

5. **Create SplitInBatches Node "Loop Over Items"**  
   - Type: SplitInBatches  
   - Default batch size (1)  
   - Connect Split Out → Loop Over Items

6. **Create Set Node "Set before and after dates"**  
   - Type: Set  
   - Assignments:  
     - `after` = `={{ $json.after }}`  
     - `before` = `={{ DateTime.fromISO($json.after).plus(1, 'week').toISODate() }}`  
   - Connect Loop Over Items (second output) → Set before and after dates

7. **Create Gmail Node "Get a batch of messages"**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials  
   - Operation: getAll  
   - Filters:  
     - receivedAfter: `={{ $json.after }}`  
     - receivedBefore: `={{ $json.before }}`  
   - Options: Download attachments enabled  
   - Return all: true  
   - Connect Set before and after dates → Get a batch of messages

8. **Create Set Node "Extract email fields"**  
   - Type: Set  
   - Assignments:  
     - `email_text` = `={{ $json.text }}`  
     - `email_from` = `={{ $json.from?.text ?? '' }}`  
     - `email_to` = `={{ $json.to?.text ?? '' }}`  
     - `email_cc` = `={{ $json.cc?.text ?? '' }}`  
     - `date` = `={{ DateTime.fromISO($json.date).toISO() }}`  
     - `email_id` = `={{ $json.id }}`  
     - `thread_id` = `={{ $json.threadId }}`  
     - `email_subject` = `={{ $json.subject }}`  
     - `attachments` = `={{ Object.keys($binary).map(item => $binary[item].fileName).filter(item => !!item) }}`  
   - Connect Get a batch of messages → Extract email fields

9. **Create Postgres Node "Store structured"**  
   - Type: Postgres (upsert)  
   - Credentials: Use same Postgres credentials  
   - Table: `emails_metadata` in schema `public`  
   - Matching column: `email_id`  
   - Map all extracted fields accordingly  
   - On error: Continue  
   - Connect Extract email fields → Store structured

10. **Create Langchain Node "Recursive Character Text Splitter"**  
    - Type: Text Splitter Recursive Character  
    - Parameters: chunkSize=2000, chunkOverlap=50  
    - Connect as ai_textSplitter input to Default Data Loader (see next step)

11. **Create Langchain Node "Default Data Loader"**  
    - Type: Document Default Data Loader  
    - Parameters:  
      - jsonData: `={{ $('Extract email fields').item.json.email_text }}`  
      - Metadata:  
        - `emails_metadata.id` = `={{ $('Extract email fields').item.json.email_id }}`  
        - `emails_metadata.thread_id` = `={{ $('Extract email fields').item.json.thread_id }}`  
    - Connect Recursive Character Text Splitter → Default Data Loader

12. **Create Langchain Node "Embeddings Ollama"**  
    - Type: Embeddings Ollama  
    - Model: `nomic-embed-text:latest`  
    - Connect Default Data Loader (ai_document output) → Embeddings Ollama (ai_embedding input)

13. **Create Langchain Node "Store vectorized"**  
    - Type: Vector Store PGVector  
    - Table name: `emails_embeddings`  
    - Mode: Insert  
    - On error: Continue  
    - Connect Store structured → Store vectorized  
    - Connect Embeddings Ollama → Store vectorized (ai_embedding input)  
    - Connect Default Data Loader → Store vectorized (ai_document input)

14. **Create If Node "Was manually triggered?"**  
    - Type: If  
    - Condition: Check if `Manual Trigger` node was executed (expression: `={{ $('Manual Trigger').isExecuted }}`)  
    - True output: Connect to Loop Over Items (to continue batch)  
    - False output: Connect to No Operation node

15. **Create No Operation Node "No Operation, do nothing"**  
    - Type: NoOp  
    - Connect False output of If node here

16. **Create Gmail Trigger Node "Gmail Trigger"**  
    - Type: Gmail Trigger  
    - Credentials: Configure Gmail OAuth2 credentials  
    - Filters: labelIds = ["INBOX"]  
    - Options: Download attachments enabled  
    - Poll times: every minute  
    - Connect output to Extract email fields (same node as bulk import)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is designed to run 100% locally using open-source tools: Ollama for LLM embeddings, nomic-embed-text model, and pgvector extension on Postgres for vector storage.                                                                                                                                                                                               | Workflow description                                                                                |
| Vector similarity searches enable semantic querying of emails, while structured storage allows factual queries; both are linked by `email_id` stored as `emails_metadata.id` in vectors metadata.                                                                                                                                                                              | Sticky Note3                                                                                       |
| Bulk import requires specifying your Gmail account creation date in the code node "Explode interval into weeks" before running the workflow manually.                                                                                                                                                                                                                         | Sticky Note, Explode interval into weeks node                                                     |
| Activate the workflow to enable continuous email ingestion via the Gmail Trigger node, which polls every minute.                                                                                                                                                                                                                                                             | Sticky Note2                                                                                       |
| Learn more about vector embeddings: https://www.pinecone.io/learn/vector-embeddings/                                                                                                                                                                                                                                                                                          | Sticky Note3                                                                                       |
| Ollama nomic-embed-text model details: https://ollama.com/library/nomic-embed-text                                                                                                                                                                                                                                                                                            | Sticky Note3                                                                                       |
| This workflow can be paired with other templates for semantic and structured RAG workflows, including email chatbots and SQL query translation for emails (links forthcoming).                                                                                                                                                                                               | Workflow description                                                                                |

---

This documentation fully describes the workflow structure, node configurations, data flow, and usage instructions, enabling advanced users and AI agents to understand, reproduce, and modify the workflow effectively.