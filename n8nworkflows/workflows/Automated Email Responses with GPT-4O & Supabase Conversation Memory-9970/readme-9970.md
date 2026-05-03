Automated Email Responses with GPT-4O & Supabase Conversation Memory

https://n8nworkflows.xyz/workflows/automated-email-responses-with-gpt-4o---supabase-conversation-memory-9970


# Automated Email Responses with GPT-4O & Supabase Conversation Memory

### 1. Workflow Overview

This workflow automates professional email responses by leveraging AI (OpenAI GPT-4O) combined with a Supabase vector database for conversation memory and knowledge retrieval. Its primary goal is to process incoming Outlook emails, categorize and filter them, use historical conversation context and FAQ/email template databases to generate intelligent replies, and update the knowledge base with new interactions.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Preprocessing:** Polls Outlook inbox for unread emails, cleans the HTML content to extract plain text, and prepares raw email data.
- **1.2 Email Categorization and Spam Filtering:** Retrieves existing categories, classifies emails using GPT-4O-Mini, aggregates categories, and filters spam emails.
- **1.3 Contextual AI Response Generation:** Retrieves conversation history from Supabase, formats it, and uses an AI agent with access to FAQ and email template vector stores to draft replies or flag for human review.
- **1.4 Response Delivery and Data Ingestion:** Decides whether to forward or reply to emails, sends responses via Outlook, then invokes a sub-workflow to ingest email and response data into Supabase to update conversation memory.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

- **Overview:**  
  This block triggers on new unread emails in a designated Outlook folder, extracts the email content, cleans the HTML to remove quoted text and formatting, and prepares it for further processing.

- **Nodes Involved:**  
  - Microsoft Outlook Trigger  
  - Loop Over Items1 (split batch processing)  
  - Clean HTML

- **Node Details:**

  - **Microsoft Outlook Trigger**  
    - *Type & Role:* Trigger node to poll Outlook mailbox for unread emails in a specific folder every minute.  
    - *Configuration:* Filters unread emails without attachments in a specified folder (folder ID provided). Downloads no attachments.  
    - *Credentials:* Microsoft Outlook OAuth2.  
    - *Inputs/Outputs:* No input; outputs raw email data including sender, subject, body in HTML, and metadata.  
    - *Potential Failures:* OAuth token expiration, API rate limits, folder ID mismatches.

  - **Loop Over Items1**  
    - *Type & Role:* Splits incoming batch of emails into individual items for processing.  
    - *Configuration:* Default split with no batch size specified (processes items one by one).  
    - *Inputs:* Connected from Outlook Trigger output.  
    - *Outputs:* Individual email items to Clean HTML node.  
    - *Edge Cases:* Large batch size could cause performance issues if not controlled.

  - **Clean HTML**  
    - *Type & Role:* Code node that strips HTML tags from email body, removes quoted replies and signatures, and extracts clean plain text email content.  
    - *Configuration:* JavaScript code uses regex to remove Gmail quotes, blockquotes, reply headers, horizontal rules, styles, scripts, and HTML tags; also removes common HTML entities and trims whitespace. It further removes plain text quotes using regex patterns.  
    - *Inputs:* Individual email items from Loop Over Items1.  
    - *Outputs:* JSON with original email data plus a new `cleanBody` field containing the cleaned plain text.  
    - *Potential Failures:* Complex or malformed HTML might not be fully cleaned; regex might remove important content in edge cases.

---

#### 2.2 Email Categorization and Spam Filtering

- **Overview:**  
  Retrieves existing email categories from Supabase, categorizes incoming emails using GPT-4O-Mini considering spam detection, aggregates category data, and filters out spam emails.

- **Nodes Involved:**  
  - Retrieve Categories (Postgres)  
  - Aggregate1  
  - Categorize (OpenAI GPT-4O-Mini)  
  - Spam Filter (If node)

- **Node Details:**

  - **Retrieve Categories**  
    - *Type & Role:* Executes a Postgres query on Supabase to fetch distinct email categories from the `emailreplies` table.  
    - *Configuration:* Executes once per workflow run to get updated categories.  
    - *Credentials:* Supabase Postgres.  
    - *Outputs:* List of categories passed as context to the Categorize node.  
    - *Potential Failures:* Database connectivity, query syntax errors.

  - **Aggregate1**  
    - *Type & Role:* Aggregates the categories list from the database query into a single array suitable for prompt injection.  
    - *Configuration:* Aggregates on the `category` field.  
    - *Inputs:* Connected from Retrieve Categories.  
    - *Outputs:* Aggregated category list to Categorize node.

  - **Categorize**  
    - *Type & Role:* OpenAI GPT-4O-Mini model node; classifies the email into existing categories or creates a new category based on detailed instructions and spam detection rules embedded in the system prompt.  
    - *Configuration:* Uses detailed system prompt specifying spam rules, business context for construction company, new category criteria, and outputs strict JSON with category name.  
    - *Inputs:* Email metadata from Clean HTML, aggregated categories from Aggregate1.  
    - *Outputs:* JSON with field `category`.  
    - *Credentials:* OpenAI API key.  
    - *Potential Failures:* Model timeout, API quota exceeded, prompt formatting errors.

  - **Spam Filter**  
    - *Type & Role:* Conditional If node that checks if the category returned is "SPAM" and routes emails accordingly.  
    - *Configuration:* Checks `message.content.category` from Categorize output, case-insensitive, excludes spam emails from further processing.  
    - *Inputs:* Categorize output.  
    - *Outputs:* Routes spam emails for exclusion; non-spam continue to conversation retrieval.  
    - *Edge Cases:* Misclassification may occur; depends on prompt accuracy.

---

#### 2.3 Contextual AI Response Generation

- **Overview:**  
  For non-spam emails, retrieves past conversation history from Supabase, formats it, and uses a GPT-4O AI Agent with access to FAQ and Email Template vector databases to draft an intelligent reply or flag for human review if uncertain.

- **Nodes Involved:**  
  - Conversation Retrieval (Postgres)  
  - Format (Code)  
  - Email Manager (Langchain AI Agent)  
  - FAQ DB (Supabase vector store)  
  - Email Template DB (Supabase vector store)  
  - Embeddings OpenAI1 & 2 (OpenAI embeddings for vector stores)

- **Node Details:**

  - **Conversation Retrieval**  
    - *Type & Role:* Executes a SQL SELECT query on `emailreplies` table to fetch past conversation messages related to the current conversation ID.  
    - *Inputs:* Uses `conversationId` from Clean HTML node.  
    - *Outputs:* Conversation history rows to Format node.  
    - *Credentials:* Supabase Postgres.  
    - *Potential Failures:* DB connectivity, missing conversation ID.

  - **Format**  
    - *Type & Role:* Code node that sorts conversation history by date ascending and concatenates date, content, and reply fields into a formatted plain-text string representing conversation context.  
    - *Inputs:* Rows from Conversation Retrieval.  
    - *Outputs:* Single string `conversationHistory` to Email Manager.  
    - *Potential Failures:* Empty conversation history, date parsing errors.

  - **Embeddings OpenAI1 & Embeddings OpenAI2**  
    - *Type & Role:* Compute vector embeddings for querying Email Template DB and FAQ DB respectively.  
    - *Inputs:* N/A (used as embedding providers for vector stores).  
    - *Outputs:* Embeddings passed to their respective Supabase vector store nodes.  
    - *Credentials:* OpenAI API.

  - **FAQ DB**  
    - *Type & Role:* Supabase vector store node configured in "retrieve-as-tool" mode to find relevant FAQ entries by vector similarity.  
    - *Credentials:* Supabase API key.  
    - *Outputs:* FAQ results as a tool accessible by the Email Manager agent.

  - **Email Template DB**  
    - *Type & Role:* Supabase vector store node to find relevant email reply templates by vector similarity, also accessible as a tool for Email Manager.  
    - *Credentials:* Supabase API key.

  - **Email Manager**  
    - *Type & Role:* Langchain AI Agent node using GPT-4O to draft professional replies or flag emails for human review.  
    - *Configuration:* System message describes detailed stepwise logic for:
      - Checking attachments to possibly skip to forwarding,
      - Searching FAQ and email template databases,
      - Deciding whether to reply or forward,
      - Formatting output strictly as JSON with reply body (HTML with <br>) and forward boolean,
      - Addressing sender by name,
      - Using conversation history for context.
    - *Inputs:* Cleaned email content, category, conversation history, FAQ DB and Email Template DB as tools.
    - *Outputs:* JSON with fields `body` and `forward` indicating reply content and forwarding need.
    - *Credentials:* OpenAI API.
    - *Potential Failures:* Model API errors, incomplete FAQ/template data, malformed output JSON.

---

#### 2.4 Response Delivery and Data Ingestion

- **Overview:**  
  Routes AI-generated responses based on forwarding flag, sends replies or forwards via Outlook, then invokes a sub-workflow to ingest the email and response into Supabase, updating conversation memory for future use.

- **Nodes Involved:**  
  - Forward? (If)  
  - Reply1 (Microsoft Outlook Reply)  
  - Reply (Microsoft Outlook Reply)  
  - Outlook Forward (HTTP Request to MS Graph API)  
  - Execute Workflow (Sub-workflow trigger)  
  - Loop Over Items1 (for batch processing in sub-workflow)  
  - When Executed by Another Workflow (Sub-workflow trigger)  
  - Supabase Vector Store4 (Insert vector embedding)  
  - Embeddings OpenAI6 (Embedding generation)  
  - Default Data Loader3 (Document loader)  
  - Recursive Character Text Splitter3 (Text splitter)  
  - Update a row2 (Supabase update)

- **Node Details:**

  - **Forward?**  
    - *Type & Role:* If node that checks the `forward` boolean from Email Manager output.  
    - *Outputs:* True branch triggers forwarding; false branch triggers replying.

  - **Reply1**  
    - *Type & Role:* Microsoft Outlook Reply node that sends the AI-generated reply email.  
    - *Inputs:* Email Manager output JSON body, original email message ID.  
    - *Credentials:* Microsoft Outlook OAuth2.

  - **Reply**  
    - *Type & Role:* Alternate Microsoft Outlook Reply node used when forwarding is not required (used in false branch).  
    - *Credentials:* Microsoft Outlook OAuth2.

  - **Outlook Forward**  
    - *Type & Role:* HTTP Request node using Microsoft Graph API to forward the email with a comment asking for human review.  
    - *Configuration:* POST request to `/me/messages/{id}/forward` with JSON body to specify recipients and comment. Currently, recipient address is empty and should be configured.  
    - *Credentials:* Microsoft Outlook OAuth2.  
    - *Potential Failures:* Missing recipient email, API errors, insufficient permissions.

  - **Execute Workflow**  
    - *Type & Role:* Invokes a sub-workflow (ID `l6VbNoViT2nEeTmN`) to ingest the email and reply data into Supabase for memory building.  
    - *Inputs:* Mapped parameters including cleaned body, reply, category, subject, conversation ID, and date.  
    - *Outputs:* None specified; triggers further processing.

  - **When Executed by Another Workflow**  
    - *Type & Role:* Sub-workflow trigger node receiving input JSON for ingestion.  
    - *Inputs:* JSON with email and reply data fields.

  - **Supabase Vector Store4**  
    - *Type & Role:* Supabase vector store node that inserts the new email-reply pair into the `emailreplies` table with vector embeddings.  
    - *Credentials:* Supabase API key.

  - **Embeddings OpenAI6**  
    - *Type & Role:* Generates embeddings for the new email-reply content before inserting into Supabase.

  - **Default Data Loader3**  
    - *Type & Role:* Prepares the email-reply content as a document with structured JSON fields to insert into vector store.

  - **Recursive Character Text Splitter3**  
    - *Type & Role:* Splits long text fields into chunks for embedding processing.

  - **Update a row2**  
    - *Type & Role:* Updates existing rows in Supabase emailreplies table with new metadata including category, flag, subject, conversation ID, date, and reply text.  
    - *Inputs:* Matches rows by content field and updates accordingly.

---

### 3. Summary Table

| Node Name                  | Node Type                                     | Functional Role                                      | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                         |
|----------------------------|-----------------------------------------------|-----------------------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Microsoft Outlook Trigger   | n8n-nodes-base.microsoftOutlookTrigger         | Polls Outlook inbox for new unread emails           |                            | Loop Over Items1             | # Automation Overview: Polls inbox for incoming emails                                                                             |
| Loop Over Items1            | n8n-nodes-base.splitInBatches                   | Splits batch of emails into individual items        | Microsoft Outlook Trigger   | Clean HTML                  | # Automation Overview: Polls inbox for incoming emails                                                                             |
| Clean HTML                 | n8n-nodes-base.code                              | Cleans and extracts plain text from HTML emails     | Loop Over Items1            | Retrieve Categories          | # Phase 1: Every email gets stripped of HTML and categorized via LLM                                                               |
| Retrieve Categories         | n8n-nodes-base.postgres                          | Retrieves distinct categories from Supabase         | Clean HTML                  | Aggregate1                  | # Phase 1: Every email gets stripped of HTML and categorized via LLM                                                               |
| Aggregate1                 | n8n-nodes-base.aggregate                          | Aggregates categories for LLM prompt                 | Retrieve Categories         | Categorize                  | # Phase 1: Every email gets stripped of HTML and categorized via LLM                                                               |
| Categorize                 | @n8n/n8n-nodes-langchain.openAi                  | Classifies email into categories or spam             | Aggregate1, Clean HTML      | Spam Filter                 | # Phase 1: Every email gets stripped of HTML and categorized via LLM                                                               |
| Spam Filter                 | n8n-nodes-base.if                                 | Filters out spam emails                               | Categorize                  | Conversation Retrieval, Loop Over Items1 | # Automation Overview: Filters out spam emails                                                                                       |
| Conversation Retrieval      | n8n-nodes-base.postgres                          | Retrieves conversation history from Supabase         | Spam Filter                 | Format                      | # Phase 2: Conversation history retrieved to provide AI agent context                                                             |
| Format                     | n8n-nodes-base.code                              | Formats conversation history into plain text string | Conversation Retrieval      | Email Manager               | # Phase 2: Conversation history retrieved to provide AI agent context                                                             |
| Embeddings OpenAI1          | @n8n/n8n-nodes-langchain.embeddingsOpenAi        | Embeddings for Email Template DB                      |                            | Email Template DB           | # Phase 2: AI agent uses FAQ and Email Template DB                                                                                  |
| Email Template DB           | @n8n/n8n-nodes-langchain.vectorStoreSupabase      | Retrieves email templates as tool for AI agent       | Embeddings OpenAI1          | Email Manager               | # Phase 2: AI agent uses FAQ and Email Template DB                                                                                  |
| Embeddings OpenAI2          | @n8n/n8n-nodes-langchain.embeddingsOpenAi        | Embeddings for FAQ DB                                 |                            | FAQ DB                     | # Phase 2: AI agent uses FAQ and Email Template DB                                                                                  |
| FAQ DB                     | @n8n/n8n-nodes-langchain.vectorStoreSupabase      | Retrieves FAQs as tool for AI agent                   | Embeddings OpenAI2          | Email Manager               | # Phase 2: AI agent uses FAQ and Email Template DB                                                                                  |
| Email Manager              | @n8n/n8n-nodes-langchain.agent                     | AI agent drafts email reply or flags for review      | Format, Email Template DB, FAQ DB, Clean HTML, Categorize | Forward?                  | # Phase 3: AI agent drafts replies or flags for review                                                                             |
| Forward?                   | n8n-nodes-base.if                                 | Routes emails to forward or reply branches            | Email Manager               | Reply1, Reply               | # Phase 3: Route emails based on AI decision                                                                                        |
| Reply1                     | n8n-nodes-base.microsoftOutlook                    | Sends reply emails via Outlook                         | Forward? (true branch)      | Outlook Forward             | # Phase 3: Route emails based on AI decision                                                                                        |
| Reply                      | n8n-nodes-base.microsoftOutlook                    | Sends reply emails via Outlook                         | Forward? (false branch)     | Execute Workflow            | # Phase 3: Route emails based on AI decision                                                                                        |
| Outlook Forward            | n8n-nodes-base.httpRequest                         | Forwards email using Microsoft Graph API              | Reply1                      | Execute Workflow            | # Phase 3: Route emails based on AI decision                                                                                        |
| Execute Workflow           | n8n-nodes-base.executeWorkflow                     | Invokes sub-workflow to ingest email-reply into DB   | Reply, Outlook Forward      | Loop Over Items1 (subwf)    | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Loop Over Items1 (subwf)    | n8n-nodes-base.splitInBatches                      | Splits batch for ingestion sub-workflow               | Execute Workflow            | Clean HTML (subwf)          | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger           | Triggers ingestion sub-workflow                        |                            | Supabase Vector Store4      | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Supabase Vector Store4     | @n8n/n8n-nodes-langchain.vectorStoreSupabase       | Inserts email-reply vectors into Supabase DB          | When Executed by Another Workflow | Update a row2             | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Embeddings OpenAI6          | @n8n/n8n-nodes-langchain.embeddingsOpenAi          | Embeddings for ingestion into Supabase DB             |                            | Supabase Vector Store4      | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Default Data Loader3        | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Prepares email-reply document for vector store        |                            | Supabase Vector Store4      | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Recursive Character Text Splitter3 | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits long text into chunks for embedding            |                            | Default Data Loader3        | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Update a row2              | n8n-nodes-base.supabase                            | Updates metadata fields for ingested email-reply row | Supabase Vector Store4      |                             | # Phase 4: Subworkflow to ingest email + response into Supabase via vector embedding                                                |
| Sticky Note (4 total)      | n8n-nodes-base.stickyNote                          | Informational notes for blocks                         |                            |                             | See individual sticky notes for phase descriptions and automation overview                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger: Microsoft Outlook Trigger**  
   - Node type: Microsoft Outlook Trigger  
   - Poll every minute for unread emails in a specific folder (set folder ID).  
   - Filter: unread, no attachments, folder(s) included.  
   - Set credentials with Microsoft Outlook OAuth2.

2. **Add Loop Over Items1**  
   - Node type: Split In Batches  
   - Default configuration to process emails one by one.  
   - Connect output of Outlook Trigger to this node.

3. **Add Clean HTML (Code node)**  
   - Node type: Code  
   - Paste JavaScript code to strip HTML, remove quoted replies and signatures, clean text, and produce `cleanBody`.  
   - Connect input from Loop Over Items1.

4. **Add Retrieve Categories (Postgres node)**  
   - Node type: Postgres  
   - Query: `SELECT DISTINCT category FROM emailreplies;`  
   - Set Supabase Postgres credentials.  
   - Connect from Clean HTML output.

5. **Add Aggregate1 (Aggregate node)**  
   - Node type: Aggregate  
   - Aggregate on field `category` to form an array.  
   - Connect from Retrieve Categories.

6. **Add Categorize (OpenAI GPT-4O-Mini)**  
   - Node type: Langchain OpenAI  
   - Model: gpt-4o-mini  
   - Set detailed system prompt including spam detection, category rules, and output JSON format.  
   - Inject email details and aggregated categories into prompt.  
   - Set OpenAI API credentials.  
   - Connect from Aggregate1 and Clean HTML (for email data).

7. **Add Spam Filter (If node)**  
   - Node type: If  
   - Condition: `category` not equals "spam" (case-insensitive).  
   - Connect from Categorize.  
   - True branch goes to Conversation Retrieval; false branch can stop or archive.

8. **Add Conversation Retrieval (Postgres node)**  
   - Node type: Postgres  
   - Query to fetch conversation history by conversation ID from `emailreplies`.  
   - Use conversationId from Clean HTML node.  
   - Connect from Spam Filter (true branch).  
   - Set Supabase credentials.

9. **Add Format (Code node)**  
   - Node type: Code  
   - Sort conversation rows by date and concatenate date, body, and reply into one string called `conversationHistory`.  
   - Connect from Conversation Retrieval.

10. **Add Embeddings OpenAI1 and Embeddings OpenAI2**  
    - Node type: Langchain Embeddings OpenAI  
    - For Email Template DB and FAQ DB respectively.  
    - Set OpenAI credentials.

11. **Add Email Template DB and FAQ DB (Supabase vector store nodes)**  
    - Retrieve vectors as tools using embeddings from corresponding nodes.  
    - Configure with table names `emailreplies` and `faq`.  
    - Set Supabase API credentials.

12. **Add Email Manager (Langchain AI Agent)**  
    - Use GPT-4O model.  
    - System prompt includes instructions to use conversation history, FAQ DB, Email Template DB, attachment check, and reply/forward logic.  
    - Input: cleaned email, category, conversation history, tools FAQ and Email Template DB.  
    - Output: JSON with `body` (HTML) and `forward` (boolean).  
    - Set OpenAI credentials.  
    - Connect from Format, FAQ DB, Email Template DB, Clean HTML, and Categorize.

13. **Add Forward? (If node)**  
    - Check if `forward` field from Email Manager output is true.  
    - True branch to forwarding nodes; false branch to reply nodes.

14. **Add Reply1 and Reply (Microsoft Outlook nodes)**  
    - Reply1 for forwarding branch, Reply for replying branch.  
    - Set credentials Microsoft Outlook OAuth2.  
    - Use messageId from original email.  
    - Connect Forward? true to Reply1, false to Reply.

15. **Add Outlook Forward (HTTP Request node)**  
    - POST to MS Graph `/me/messages/{id}/forward`.  
    - JSON body includes recipient email(s) and comment "Please review the following email".  
    - Set Microsoft Outlook OAuth2 credentials.  
    - Connect from Reply1.

16. **Add Execute Workflow (Execute Workflow node)**  
    - Call sub-workflow for ingestion into Supabase.  
    - Map inputs: body, date, reply, subject, category, conversationID from Clean HTML, Email Manager, Categorize nodes.  
    - Connect from Reply and Outlook Forward.

17. **Create Sub-Workflow for Ingestion**

    - Add When Executed by Another Workflow trigger node.  
    - Add Supabase Vector Store4 node (vector insert) with table `emailreplies`.  
    - Add Embeddings OpenAI6 to create embeddings for insertion.  
    - Add Default Data Loader3 to prepare document JSON from input data.  
    - Add Recursive Character Text Splitter3 to chunk text.  
    - Connect nodes in order: When Executed → Recursive Splitter → Default Data Loader → Embeddings OpenAI6 → Supabase Vector Store4.  
    - Add Update a row2 node (Supabase) connected post Supabase Vector Store4 to update metadata fields on matching content rows.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates email reply generation for a construction company context using GPT-4O models and vector databases for FAQ and email templates.                                                            | Overview as per sticky note at workflow start.                                                 |
| Phase 1: Email cleaning and categorization, including spam detection.                                                                                                                                         | Sticky Note near Clean HTML and Categorize nodes.                                              |
| Phase 2: Conversation history retrieval and AI agent context enrichment with FAQ and templates stored in Supabase vector DBs.                                                                                 | Sticky Note near Conversation Retrieval and Email Manager nodes.                              |
| Phase 3: AI agent determines routing: auto-reply or human review forwarding.                                                                                                                                    | Sticky Note near Forward? and Email Manager nodes.                                            |
| Phase 4: Sub-workflow ingests processed email and reply into Supabase to build training data and retain conversation memory.                                                                                   | Sticky Note near Execute Workflow and ingestion sub-workflow nodes.                           |
| Supabase and OpenAI credentials must be configured with proper API keys and permissions for Outlook, Supabase Postgres, and vector stores.                                                                      | Credential setup required for all API-based nodes.                                            |
| Forwarding node (`Outlook Forward`) requires configuration of recipient email address to avoid forwarding failures.                                                                                            | Currently set with empty recipient; must be updated in production.                            |
| The GPT-4O-Mini model is specifically used for categorization to reduce token usage and cost, while GPT-4O is used for detailed email drafting.                                                                | Model selection optimizes cost vs quality tradeoff.                                           |
| Email bodies are cleaned of quoted replies and signatures to reduce token bloat and improve AI understanding.                                                                                                   | Clean HTML node uses regex stripping for multiple email client formats.                       |
| The system prompt for categorization includes detailed rules for spam and new category detection, tailored for construction business emails.                                                                    | Ensures relevant and actionable categories.                                                  |
| The AI agent uses two vector stores as "tools" to search FAQs and templates to generate informed replies, reverting to forwarding when it cannot answer confidently or attachments are present.                 | Enhances reply quality and automates triage.                                                 |

---

**Disclaimer:**  
The provided text results exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled are legal and publicly accessible.