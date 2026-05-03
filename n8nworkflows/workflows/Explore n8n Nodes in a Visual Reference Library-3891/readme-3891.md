Explore n8n Nodes in a Visual Reference Library

https://n8nworkflows.xyz/workflows/explore-n8n-nodes-in-a-visual-reference-library-3891


# Explore n8n Nodes in a Visual Reference Library

### 1. Workflow Overview

This workflow serves as a comprehensive **visual reference library for n8n nodes**, designed to help users explore and understand a wide variety of nodes grouped by their functional roles. It is targeted at both beginners and advanced users who want to quickly grasp the capabilities of n8n nodes, especially those related to AI, app integrations, data transformation, and workflow control.

The workflow is logically divided into the following blocks:

- **1.1 Triggers:** Nodes that initiate workflows based on events or schedules.
- **1.2 AI Agents and AI Tools:** Nodes related to AI models, chains, tools, and memory management.
- **1.3 Vector Memory:** Nodes handling vector stores and embeddings for AI memory.
- **1.4 App Actions:** Nodes integrating with external applications and services.
- **1.5 Data Transformation:** Nodes for manipulating, filtering, and converting data.
- **1.6 Flow & Core:** Nodes for flow control, code execution, HTTP requests, and webhook handling.
- **1.7 Miscellaneous AI Tools:** Additional AI-related nodes including parsers and models.

Each block contains nodes grouped visually and functionally, with sticky notes providing contextual explanations.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers

**Overview:**  
This block contains nodes that start workflow execution based on various triggers such as incoming emails, form submissions, schedules, webhooks, and external service events.

**Nodes Involved:**  
- Calendly Trigger  
- Email Trigger (IMAP)  
- Google Drive Trigger  
- Gumroad Trigger  
- Local File Trigger  
- On form submission  
- Schedule Trigger  
- Webhook  
- When chat message received  
- When clicking 'Test workflow' (Manual Trigger)  
- Workflow Input Trigger  
- Gmail Trigger  
- Google Sheets Trigger  

**Node Details:**

- **Calendly Trigger**  
  - Type: Calendly event webhook trigger  
  - Config: Listens for invitee.created and invitee.canceled events  
  - Input: External Calendly events  
  - Output: Starts workflow on event  
  - Edge cases: Webhook connectivity issues, event filtering errors

- **Email Trigger (IMAP)**  
  - Type: IMAP email polling trigger  
  - Config: Default options, polls email inbox  
  - Input: New emails  
  - Output: Starts workflow on new email  
  - Edge cases: Authentication failure, IMAP server downtime

- **Google Drive Trigger**  
  - Type: Google Drive folder watcher  
  - Config: Polls specific folder every minute  
  - Credentials: Google Drive OAuth2  
  - Input: New or changed files in folder  
  - Output: Starts workflow on file changes  
  - Edge cases: OAuth token expiration, folder permission issues

- **Gumroad Trigger**  
  - Type: Gumroad webhook trigger  
  - Config: Default, listens for Gumroad events  
  - Input: Gumroad event webhook  
  - Output: Starts workflow on event  
  - Edge cases: Webhook registration failure

- **Local File Trigger**  
  - Type: Local folder watcher  
  - Config: Watches folder for changes  
  - Input: File system events  
  - Output: Starts workflow on file changes  
  - Edge cases: File system access permissions

- **On form submission**  
  - Type: Form submission webhook trigger  
  - Config: Accepts form fields (empty in template)  
  - Input: Form data POSTed to webhook  
  - Output: Starts workflow on form submit  
  - Edge cases: Missing form fields, webhook URL misconfiguration

- **Schedule Trigger**  
  - Type: Time-based trigger  
  - Config: Interval-based schedule (default empty)  
  - Input: Time events  
  - Output: Starts workflow on schedule  
  - Edge cases: Incorrect interval setup

- **Webhook**  
  - Type: Generic webhook trigger  
  - Config: Path specified, no extra options  
  - Input: HTTP requests to webhook URL  
  - Output: Starts workflow on request  
  - Edge cases: URL conflicts, security/authentication missing

- **When chat message received**  
  - Type: Chat message webhook trigger (Langchain)  
  - Config: Default options  
  - Input: Incoming chat messages  
  - Output: Starts workflow on chat message  
  - Edge cases: Webhook connectivity, message format errors

- **When clicking 'Test workflow' (Manual Trigger)**  
  - Type: Manual trigger  
  - Config: No parameters  
  - Input: User-initiated trigger  
  - Output: Starts workflow manually  
  - Edge cases: None

- **Workflow Input Trigger**  
  - Type: Execute workflow trigger  
  - Config: No parameters  
  - Input: External workflow execution  
  - Output: Starts workflow on execution call  
  - Edge cases: Workflow execution permission

- **Gmail Trigger**  
  - Type: Gmail polling trigger  
  - Config: Polls every minute, no filters  
  - Input: New Gmail messages  
  - Output: Starts workflow on new email  
  - Edge cases: OAuth token expiration, Gmail API limits

- **Google Sheets Trigger**  
  - Type: Google Sheets polling trigger  
  - Config: Polls every minute on specified sheet and document  
  - Credentials: Google Sheets OAuth2  
  - Input: Changes in sheet  
  - Output: Starts workflow on sheet update  
  - Edge cases: OAuth expiration, sheet access issues

---

#### 1.2 AI Agents and AI Tools

**Overview:**  
This block contains nodes related to AI processing, including AI agents, language models, chains, and memory management nodes.

**Nodes Involved:**  
- AI Agent  
- OpenAI  
- Basic LLM Chain  
- Information Extractor  
- Question and Answer Chain  
- Sentiment Analysis  
- Summarization Chain  
- Text Classifier  
- Chat Memory Manager  

**Node Details:**

- **AI Agent**  
  - Type: Langchain AI agent node  
  - Config: Default options  
  - Input: AI prompts or data  
  - Output: AI-generated responses  
  - Edge cases: API key issues, model availability

- **OpenAI**  
  - Type: Langchain OpenAI node  
  - Config: Model selection list (empty by default), message templates  
  - Credentials: OpenAI API key  
  - Input: Prompt messages  
  - Output: AI completions  
  - Edge cases: API rate limits, invalid model ID

- **Basic LLM Chain**  
  - Type: Langchain LLM chain  
  - Config: Default chain settings  
  - Input: Text input  
  - Output: Processed AI output  
  - Edge cases: Chain configuration errors

- **Information Extractor**  
  - Type: Langchain information extraction node  
  - Config: Default options  
  - Input: Text data  
  - Output: Extracted structured information  
  - Edge cases: Parsing errors

- **Question and Answer Chain**  
  - Type: Langchain retrieval-based QA chain  
  - Config: Default options  
  - Input: Questions and context  
  - Output: Answers  
  - Edge cases: Retrieval failures

- **Sentiment Analysis**  
  - Type: Langchain sentiment analysis node  
  - Config: Default options  
  - Input: Text data  
  - Output: Sentiment classification  
  - Edge cases: Ambiguous sentiment

- **Summarization Chain**  
  - Type: Langchain summarization chain  
  - Config: Default options  
  - Input: Text data  
  - Output: Summaries  
  - Edge cases: Over-summarization

- **Text Classifier**  
  - Type: Langchain text classification node  
  - Config: Default options  
  - Input: Text data  
  - Output: Classification labels  
  - Edge cases: Misclassification

- **Chat Memory Manager**  
  - Type: Langchain memory manager  
  - Config: Default options  
  - Input: Chat context  
  - Output: Managed chat memory  
  - Edge cases: Memory overflow

---

#### 1.3 Vector Memory

**Overview:**  
Nodes in this block provide vector store functionality for AI agents, enabling memory storage and retrieval using various backends.

**Nodes Involved:**  
- Default Data Loader  
- Auto-fixing Output Parser  
- Answer questions with a vector store  
- In-Memory Vector Store  
- Pinecone Vector Store  
- Postgres PGVector Store  
- Supabase Vector Store  

**Node Details:**

- **Default Data Loader**  
  - Type: Langchain document data loader  
  - Config: Default options  
  - Input: Documents  
  - Output: Loaded data for vectorization  
  - Edge cases: Unsupported document formats

- **Auto-fixing Output Parser**  
  - Type: Langchain output parser with auto-fix  
  - Config: Default options  
  - Input: AI output  
  - Output: Corrected structured output  
  - Edge cases: Parsing failures

- **Answer questions with a vector store**  
  - Type: Langchain vector store tool  
  - Config: Default options  
  - Input: Query text  
  - Output: Retrieved answers  
  - Edge cases: Empty vector store

- **In-Memory Vector Store**  
  - Type: Langchain in-memory vector store  
  - Config: Mode set to retrieve-as-tool  
  - Input: Vector queries  
  - Output: Vector search results  
  - Edge cases: Memory limits

- **Pinecone Vector Store**  
  - Type: Langchain Pinecone vector store  
  - Config: Mode retrieve-as-tool, index selected from list  
  - Input: Vector queries  
  - Output: Vector search results  
  - Edge cases: Pinecone API errors, index misconfiguration

- **Postgres PGVector Store**  
  - Type: Langchain Postgres PGVector store  
  - Config: Mode retrieve-as-tool  
  - Input: Vector queries  
  - Output: Vector search results  
  - Edge cases: DB connection issues

- **Supabase Vector Store**  
  - Type: Langchain Supabase vector store  
  - Config: Mode retrieve-as-tool, table name selected from list  
  - Input: Vector queries  
  - Output: Vector search results  
  - Edge cases: Supabase API errors

---

#### 1.4 App Actions

**Overview:**  
This block groups nodes that interact with external applications and services such as Google Workspace apps, Dropbox, Gmail, Reddit, YouTube, and social media platforms.

**Nodes Involved:**  
- Bitly App  
- Dropbox App  
- Gmail App  
- Google Calendar App  
- Google Docs App  
- Google Sheets App  
- Pushbullet App  
- YouTube App  
- Bluesky App  
- Perplexity App  
- ElevenLabs App  
- Reddit App  
- Gmail Trigger App  
- Google Sheets Trigger App  
- X (Twitter)  

**Node Details:**

- **Bitly App**  
  - Type: URL shortening service  
  - Config: Default additional fields  
  - Input: URLs  
  - Output: Shortened URLs  
  - Edge cases: API limits

- **Dropbox App**  
  - Type: Dropbox file operations  
  - Config: Download operation, OAuth2 authentication  
  - Credentials: Dropbox OAuth2  
  - Input: File paths  
  - Output: Downloaded files  
  - Edge cases: OAuth expiration, file not found

- **Gmail App**  
  - Type: Gmail email sending  
  - Config: Default options  
  - Input: Email data  
  - Output: Sent email confirmation  
  - Edge cases: Gmail API limits

- **Google Calendar App**  
  - Type: Google Calendar operations  
  - Config: Calendar selected from list  
  - Credentials: Google Calendar OAuth2  
  - Input: Calendar events  
  - Output: Event creation/update confirmation  
  - Edge cases: Calendar access permissions

- **Google Docs App**  
  - Type: Google Docs operations  
  - Config: Get operation  
  - Input: Document ID  
  - Output: Document content  
  - Edge cases: Document access permissions

- **Google Sheets App**  
  - Type: Google Sheets operations  
  - Config: Append operation, sheet and document selected from list  
  - Credentials: Google Sheets OAuth2  
  - Input: Data rows  
  - Output: Append confirmation  
  - Edge cases: Sheet access permissions

- **Pushbullet App**  
  - Type: Pushbullet notifications  
  - Config: Default  
  - Input: Notification data  
  - Output: Notification sent  
  - Edge cases: API limits

- **YouTube App**  
  - Type: YouTube playlist creation  
  - Config: Create playlist operation  
  - Input: Playlist details  
  - Output: Playlist creation confirmation  
  - Edge cases: API quota limits

- **Bluesky App**  
  - Type: Bluesky social media integration  
  - Config: Default  
  - Input: Post data  
  - Output: Post confirmation  
  - Edge cases: Auth errors

- **Perplexity App**  
  - Type: Perplexity AI tool  
  - Config: Default additional fields  
  - Input: Query data  
  - Output: AI responses  
  - Edge cases: API errors

- **ElevenLabs App**  
  - Type: Text-to-speech voice generation  
  - Config: Voice ID selected from list, additional request options  
  - Credentials: ElevenLabs API  
  - Input: Text data  
  - Output: Audio files  
  - Edge cases: Voice ID invalid, API limits

- **Reddit App**  
  - Type: Reddit API integration  
  - Config: Default  
  - Input: Reddit actions  
  - Output: API responses  
  - Edge cases: Auth errors

- **Gmail Trigger App**  
  - Type: Gmail polling trigger  
  - Config: Poll every minute  
  - Input: New Gmail messages  
  - Output: Workflow trigger  
  - Edge cases: OAuth expiration

- **Google Sheets Trigger App**  
  - Type: Google Sheets polling trigger  
  - Config: Poll every minute on specified sheet and document  
  - Credentials: Google Sheets OAuth2  
  - Input: Sheet updates  
  - Output: Workflow trigger  
  - Edge cases: OAuth expiration

- **X (Twitter)**  
  - Type: Twitter API integration  
  - Config: Default additional fields  
  - Input: Tweet data  
  - Output: Tweet confirmation  
  - Edge cases: API limits, auth errors

---

#### 1.5 Data Transformation

**Overview:**  
Nodes in this block provide data manipulation, filtering, aggregation, and conversion capabilities.

**Nodes Involved:**  
- Code  
- Date & Time  
- Edit Fields  
- Filter  
- Limit  
- Remove Duplicates  
- Split Out  
- Aggregate  
- Summarize  
- Convert to File  
- Extract from File  
- HTML  
- Markdown  
- Rename Keys  
- Sort  
- If  
- Loop Over Items  

**Node Details:**

- **Code**  
  - Type: JavaScript code execution  
  - Config: Adds a new field 'myNewField' with value 1 to each input item  
  - Input: JSON data  
  - Output: Modified JSON data  
  - Edge cases: JS errors

- **Date & Time**  
  - Type: Date/time manipulation  
  - Config: Default options  
  - Input: Date/time data  
  - Output: Modified date/time  
  - Edge cases: Invalid date formats

- **Edit Fields**  
  - Type: Set node to assign fields  
  - Config: Adds "Name Test" string and "Name 1" number fields  
  - Input: JSON data  
  - Output: Modified JSON data  
  - Edge cases: Overwriting existing fields

- **Filter**  
  - Type: Data filtering  
  - Config: Empty condition (placeholder)  
  - Input: JSON data  
  - Output: Filtered data  
  - Edge cases: Invalid filter expressions

- **Limit**  
  - Type: Limits number of items passed  
  - Config: Default (no limit set)  
  - Input: JSON data  
  - Output: Limited data  
  - Edge cases: None

- **Remove Duplicates**  
  - Type: Removes duplicate items  
  - Config: Default options  
  - Input: JSON data  
  - Output: Unique items  
  - Edge cases: Large datasets performance

- **Split Out**  
  - Type: Splits data into individual items  
  - Config: Default options  
  - Input: JSON array  
  - Output: Individual items  
  - Edge cases: Empty arrays

- **Aggregate**  
  - Type: Aggregates data fields  
  - Config: Empty field to aggregate (placeholder)  
  - Input: JSON data  
  - Output: Aggregated results  
  - Edge cases: Missing fields

- **Summarize**  
  - Type: Summarizes data fields  
  - Config: Empty fields (placeholder)  
  - Input: JSON data  
  - Output: Summary  
  - Edge cases: Incomplete data

- **Convert to File**  
  - Type: Converts data to file (text)  
  - Config: Operation set to toText  
  - Input: JSON data  
  - Output: Text file  
  - Edge cases: Large data size

- **Extract from File**  
  - Type: Extracts content from files (PDF)  
  - Config: Operation pdf  
  - Input: PDF files  
  - Output: Extracted text  
  - Edge cases: Unsupported PDF formats

- **HTML**  
  - Type: Extracts HTML content  
  - Config: Extraction values empty (placeholder)  
  - Input: HTML data  
  - Output: Extracted content  
  - Edge cases: Malformed HTML

- **Markdown**  
  - Type: Markdown processing  
  - Config: Default options  
  - Input: Markdown text  
  - Output: Processed markdown  
  - Edge cases: Invalid markdown syntax

- **Rename Keys**  
  - Type: Renames JSON keys  
  - Config: No specific keys defined  
  - Input: JSON data  
  - Output: Renamed keys  
  - Edge cases: Key conflicts

- **Sort**  
  - Type: Sorts data items  
  - Config: Default options  
  - Input: JSON data  
  - Output: Sorted data  
  - Edge cases: Missing sort fields

- **If**  
  - Type: Conditional branching  
  - Config: Empty condition (placeholder)  
  - Input: JSON data  
  - Output: Branches based on condition  
  - Edge cases: Invalid conditions

- **Loop Over Items**  
  - Type: Split in batches  
  - Config: Default options  
  - Input: JSON array  
  - Output: Batches of items  
  - Edge cases: Batch size not set

---

#### 1.6 Flow & Core

**Overview:**  
This block contains nodes that control workflow execution flow, run commands, make HTTP requests, and handle webhook responses.

**Nodes Involved:**  
- Replace Me (NoOp)  
- Execute Workflow  
- Wait  
- Execute Command  
- HTTP Request  
- Execution Data  
- FTP  
- Respond to Webhook  
- Merge  

**Node Details:**

- **Replace Me (NoOp)**  
  - Type: No operation placeholder  
  - Config: Placeholder for future node  
  - Input: Connected from Loop Over Items  
  - Output: Pass-through  
  - Edge cases: None

- **Execute Workflow**  
  - Type: Executes another workflow  
  - Config: Default options  
  - Input: Data to pass to sub-workflow  
  - Output: Sub-workflow output  
  - Edge cases: Sub-workflow not found

- **Wait**  
  - Type: Pauses execution until webhook resume  
  - Config: Resume on webhook  
  - Input: Workflow data  
  - Output: Resumed data  
  - Edge cases: Webhook not called

- **Execute Command**  
  - Type: Runs system commands  
  - Config: Empty command (placeholder)  
  - Input: Workflow data  
  - Output: Command output  
  - Edge cases: Command errors

- **HTTP Request**  
  - Type: Makes HTTP calls  
  - Config: Default options  
  - Input: Request data  
  - Output: Response data  
  - Edge cases: Timeout, connection errors

- **Execution Data**  
  - Type: Accesses execution metadata  
  - Config: Default  
  - Input: Workflow data  
  - Output: Execution info  
  - Edge cases: None

- **FTP**  
  - Type: FTP/SFTP file operations  
  - Config: Protocol set to SFTP  
  - Input: File operation data  
  - Output: Operation result  
  - Edge cases: Connection errors

- **Respond to Webhook**  
  - Type: Sends HTTP response to webhook caller  
  - Config: Default options  
  - Input: Response data  
  - Output: HTTP response  
  - Edge cases: Response formatting errors

- **Merge**  
  - Type: Merges multiple data streams  
  - Config: Default options  
  - Input: Multiple inputs  
  - Output: Merged data  
  - Edge cases: Data conflicts

---

#### 1.7 Miscellaneous AI Tools

**Overview:**  
This block contains various AI-related nodes including language models, embeddings, memory types, and output parsers.

**Nodes Involved:**  
- Call n8n Workflow Tool  
- Code Tool  
- HTTP Request1 (Langchain tool)  
- Calculator  
- SerpAPI  
- Wikipedia  
- Wolfram Alpha  
- gmailTool App  
- googleCalendarTool App  
- googleDocsTool App  
- googleSheetsTool App  
- MCP Client  
- Anthropic Chat Model  
- Google Gemini Chat Model  
- OpenAI Chat Model  
- Window Buffer Memory  
- Postgres Chat Memory  
- Redis Chat Memory  
- Item List Output Parser  
- Structured Output Parser  
- Embeddings Google Gemini  
- Embeddings OpenAI  

**Node Details:**

- **Call n8n Workflow Tool**  
  - Type: Langchain tool to call workflows  
  - Config: Default options  
  - Input: Tool invocation data  
  - Output: Workflow results  
  - Edge cases: Workflow not found

- **Code Tool**  
  - Type: Langchain code execution tool  
  - Config: Default options  
  - Input: Code snippets  
  - Output: Execution results  
  - Edge cases: Code errors

- **HTTP Request1 (Langchain tool)**  
  - Type: Langchain HTTP request tool  
  - Config: Default options  
  - Input: HTTP request data  
  - Output: HTTP response  
  - Edge cases: Timeout

- **Calculator**  
  - Type: Langchain calculator tool  
  - Config: Default options  
  - Input: Mathematical expressions  
  - Output: Calculated results  
  - Edge cases: Invalid expressions

- **SerpAPI**  
  - Type: Langchain search API tool  
  - Config: Default options  
  - Credentials: SerpAPI key  
  - Input: Search queries  
  - Output: Search results  
  - Edge cases: API limits

- **Wikipedia**  
  - Type: Langchain Wikipedia tool  
  - Config: Default options  
  - Input: Search terms  
  - Output: Article summaries  
  - Edge cases: No article found

- **Wolfram Alpha**  
  - Type: Langchain Wolfram Alpha tool  
  - Config: Default options  
  - Input: Queries  
  - Output: Computational answers  
  - Edge cases: API errors

- **gmailTool App**  
  - Type: Langchain Gmail tool  
  - Config: Default options  
  - Input: Email data  
  - Output: Email actions  
  - Edge cases: Auth errors

- **googleCalendarTool App**  
  - Type: Langchain Google Calendar tool  
  - Config: Calendar selection  
  - Credentials: Google Calendar OAuth2  
  - Input: Event data  
  - Output: Calendar actions  
  - Edge cases: Permissions

- **googleDocsTool App**  
  - Type: Langchain Google Docs tool  
  - Config: Default  
  - Input: Document data  
  - Output: Document actions  
  - Edge cases: Permissions

- **googleSheetsTool App**  
  - Type: Langchain Google Sheets tool  
  - Config: Sheet and document selection  
  - Credentials: Google Sheets OAuth2  
  - Input: Sheet data  
  - Output: Sheet actions  
  - Edge cases: Permissions

- **MCP Client**  
  - Type: Langchain MCP client tool  
  - Config: Default options  
  - Input: Client data  
  - Output: Responses  
  - Edge cases: Connection errors

- **Anthropic Chat Model**  
  - Type: Langchain Anthropic chat model  
  - Config: Default options  
  - Input: Chat prompts  
  - Output: Chat completions  
  - Edge cases: API limits

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini chat model  
  - Config: Default options  
  - Input: Chat prompts  
  - Output: Chat completions  
  - Edge cases: API availability

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI chat model  
  - Config: Default options  
  - Credentials: OpenAI API  
  - Input: Chat prompts  
  - Output: Chat completions  
  - Edge cases: API limits

- **Window Buffer Memory**  
  - Type: Langchain window buffer memory  
  - Config: Default options  
  - Input: Chat context  
  - Output: Memory window  
  - Edge cases: Memory overflow

- **Postgres Chat Memory**  
  - Type: Langchain Postgres chat memory  
  - Config: Default options  
  - Input: Chat context  
  - Output: Stored memory  
  - Edge cases: DB connection

- **Redis Chat Memory**  
  - Type: Langchain Redis chat memory  
  - Config: Default options  
  - Input: Chat context  
  - Output: Stored memory  
  - Edge cases: Redis connection

- **Item List Output Parser**  
  - Type: Langchain output parser for item lists  
  - Config: Default options  
  - Input: AI output  
  - Output: Parsed list  
  - Edge cases: Parsing errors

- **Structured Output Parser**  
  - Type: Langchain structured output parser  
  - Config: Default options  
  - Input: AI output  
  - Output: Structured data  
  - Edge cases: Parsing errors

- **Embeddings Google Gemini**  
  - Type: Langchain embeddings using Google Gemini  
  - Config: Default options  
  - Input: Text data  
  - Output: Embeddings  
  - Edge cases: API availability

- **Embeddings OpenAI**  
  - Type: Langchain embeddings using OpenAI  
  - Config: Default options  
  - Credentials: OpenAI API  
  - Input: Text data  
  - Output: Embeddings  
  - Edge cases: API limits

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                         | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                           |
|----------------------------|-----------------------------------------|---------------------------------------|---------------------------|--------------------------|-----------------------------------------------------------------------------------------------------|
| Calendly Trigger            | Calendly Trigger                        | Trigger                               |                           |                          | # TRIGGERS - This section contains all trigger nodes that can start workflow execution.              |
| Email Trigger (IMAP)        | Email IMAP Trigger                      | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Google Drive Trigger        | Google Drive Trigger                    | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Gumroad Trigger             | Gumroad Trigger                        | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Local File Trigger          | Local File Trigger                     | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| On form submission          | Form Submission Trigger                | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Schedule Trigger            | Schedule Trigger                      | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Webhook                    | Webhook Trigger                       | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| When chat message received  | Langchain Chat Trigger                 | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| When clicking 'Test workflow'| Manual Trigger                       | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Workflow Input Trigger      | Execute Workflow Trigger               | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Gmail Trigger              | Gmail Trigger                         | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| Google Sheets Trigger       | Google Sheets Trigger                  | Trigger                               |                           |                          | # TRIGGERS                                                                                          |
| AI Agent                   | Langchain AI Agent                    | AI Agent                             |                           |                          | # AI AGENTS - This section contains all AI related nodes that can attach models, tools and memory.  |
| OpenAI                     | Langchain OpenAI                      | AI Model                             |                           |                          | # AI AGENTS                                                                                         |
| Basic LLM Chain            | Langchain LLM Chain                   | AI Chain                             |                           |                          | # AI AGENTS                                                                                         |
| Information Extractor      | Langchain Information Extractor       | AI Tool                             |                           |                          | # AI AGENTS                                                                                         |
| Question and Answer Chain  | Langchain Retrieval QA Chain           | AI Chain                             |                           |                          | # AI AGENTS                                                                                         |
| Sentiment Analysis         | Langchain Sentiment Analysis           | AI Tool                             |                           |                          | # AI AGENTS                                                                                         |
| Summarization Chain        | Langchain Summarization Chain          | AI Chain                             |                           |                          | # AI AGENTS                                                                                         |
| Text Classifier            | Langchain Text Classifier              | AI Tool                             |                           |                          | # AI AGENTS                                                                                         |
| Chat Memory Manager        | Langchain Memory Manager               | AI Memory                           |                           |                          | # AI AGENTS                                                                                         |
| Default Data Loader        | Langchain Document Data Loader         | Vector Memory                       |                           |                          | # VECTOR MEMORY - This section contains tools for AI Agents related to memory storage.              |
| Auto-fixing Output Parser  | Langchain Output Parser Autofixing     | Vector Memory                       |                           |                          | # VECTOR MEMORY                                                                                     |
| Answer questions with a vector store | Langchain Vector Store Tool    | Vector Memory                       |                           |                          | # VECTOR MEMORY                                                                                     |
| In-Memory Vector Store     | Langchain In-Memory Vector Store       | Vector Memory                       |                           |                          | # VECTOR MEMORY                                                                                     |
| Pinecone Vector Store      | Langchain Pinecone Vector Store        | Vector Memory                       |                           |                          | # VECTOR MEMORY                                                                                     |
| Postgres PGVector Store   | Langchain Postgres PGVector Store      | Vector Memory                       |                           |                          | # VECTOR MEMORY                                                                                     |
| Supabase Vector Store      | Langchain Supabase Vector Store        | Vector Memory                       |                           |                          | # VECTOR MEMORY                                                                                     |
| Bitly App                 | Bitly Node                            | App Action                        |                           |                          | # APP ACTIONS - This section contains nodes for interacting with external apps and services.       |
| Dropbox App               | Dropbox Node                         | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Gmail App                 | Gmail Node                           | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Google Calendar App       | Google Calendar Node                 | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Google Docs App           | Google Docs Node                    | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Google Sheets App         | Google Sheets Node                  | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Pushbullet App            | Pushbullet Node                     | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| YouTube App               | YouTube Node                       | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Bluesky App               | Bluesky Node                       | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Perplexity App            | Perplexity Node                    | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| ElevenLabs App            | ElevenLabs Node                    | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Reddit App                | Reddit Node                       | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Gmail Trigger App         | Gmail Trigger Node                | App Action Trigger              |                           |                          | # APP ACTIONS                                                                                      |
| Google Sheets Trigger App | Google Sheets Trigger Node        | App Action Trigger              |                           |                          | # APP ACTIONS                                                                                      |
| X                         | Twitter Node                      | App Action                        |                           |                          | # APP ACTIONS                                                                                      |
| Code                      | Code Node                         | Data Transformation              |                           |                          | # DATA TRANSFORMATION - This section contains nodes for manipulating, filtering, and converting data. |
| Date & Time               | Date & Time Node                  | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Edit Fields               | Set Node                         | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Filter                    | Filter Node                      | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Limit                     | Limit Node                       | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Remove Duplicates          | Remove Duplicates Node            | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Split Out                 | Split Out Node                   | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Aggregate                 | Aggregate Node                   | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Summarize                 | Summarize Node                   | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Convert to File           | Convert To File Node             | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Extract from File          | Extract From File Node            | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| HTML                      | HTML Node                       | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Markdown                  | Markdown Node                   | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Rename Keys               | Rename Keys Node                | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Sort                      | Sort Node                      | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| If                        | If Node                        | Data Transformation              |                           |                          | # DATA TRANSFORMATION                                                                             |
| Loop Over Items           | Split In Batches Node           | Data Transformation              |                           | Replace Me               | # DATA TRANSFORMATION                                                                             |
| Replace Me                | NoOp Node                      | Flow Control / Placeholder       | Loop Over Items           | Loop Over Items (second output) | # FLOW & CORE - This section contains flow control nodes and core functionality nodes.             |
| Execute Workflow          | Execute Workflow Node           | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| Wait                      | Wait Node                      | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| Execute Command           | Execute Command Node            | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| HTTP Request              | HTTP Request Node               | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| Execution Data            | Execution Data Node             | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| FTP                       | FTP Node                       | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| Respond to Webhook        | Respond To Webhook Node         | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| Merge                     | Merge Node                     | Flow Control                    |                           |                          | # FLOW & CORE                                                                                     |
| Call n8n Workflow Tool    | Langchain Workflow Tool         | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS - This section contains miscellaneous AI tools including models, embeddings, memory and parsers. |
| Code Tool                 | Langchain Code Tool             | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| HTTP Request1             | Langchain HTTP Request Tool     | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Calculator                | Langchain Calculator Tool       | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| SerpAPI                   | Langchain SerpAPI Tool          | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Wikipedia                 | Langchain Wikipedia Tool        | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Wolfram Alpha             | Langchain Wolfram Alpha Tool    | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| gmailTool App             | Langchain Gmail Tool            | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| googleCalendarTool App    | Langchain Google Calendar Tool  | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| googleDocsTool App        | Langchain Google Docs Tool      | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| googleSheetsTool App      | Langchain Google Sheets Tool    | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| MCP Client                | Langchain MCP Client Tool       | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Anthropic Chat Model      | Langchain Anthropic Chat Model  | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Google Gemini Chat Model  | Langchain Google Gemini Chat Model | Miscellaneous AI Tool       |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| OpenAI Chat Model         | Langchain OpenAI Chat Model     | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Window Buffer Memory      | Langchain Window Buffer Memory  | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Postgres Chat Memory      | Langchain Postgres Chat Memory  | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Redis Chat Memory         | Langchain Redis Chat Memory     | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Item List Output Parser   | Langchain Output Parser Item List | Miscellaneous AI Tool         |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Structured Output Parser  | Langchain Output Parser Structured | Miscellaneous AI Tool         |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Embeddings Google Gemini  | Langchain Embeddings Google Gemini | Miscellaneous AI Tool         |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |
| Embeddings OpenAI         | Langchain Embeddings OpenAI     | Miscellaneous AI Tool           |                           |                          | # MISCELLANEOUS AI TOOLS                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes (Block 1.1):**  
   - Add Calendly Trigger node; configure events: invitee.created, invitee.canceled.  
   - Add Email Trigger (IMAP) node with default options.  
   - Add Google Drive Trigger node; set folder to watch and OAuth2 credentials.  
   - Add Gumroad Trigger node with default settings.  
   - Add Local File Trigger node; set to watch folder.  
   - Add Form Submission Trigger node; configure webhook and form fields as needed.  
   - Add Schedule Trigger node; configure interval as desired.  
   - Add Webhook node; set path and options.  
   - Add Langchain Chat Trigger node; configure webhook.  
   - Add Manual Trigger node for testing.  
   - Add Execute Workflow Trigger node.  
   - Add Gmail Trigger node; configure polling interval and OAuth2 credentials.  
   - Add Google Sheets Trigger node; configure polling interval, sheet name, document ID, and OAuth2 credentials.

2. **Add AI Agents and Tools (Block 1.2):**  
   - Add AI Agent node with default options.  
   - Add OpenAI node; select model and provide OpenAI API credentials.  
   - Add Basic LLM Chain node with default settings.  
   - Add Information Extractor node.  
   - Add Question and Answer Chain node.  
   - Add Sentiment Analysis node.  
   - Add Summarization Chain node.  
   - Add Text Classifier node.  
   - Add Chat Memory Manager node.

3. **Add Vector Memory Nodes (Block 1.3):**  
   - Add Default Data Loader node.  
   - Add Auto-fixing Output Parser node.  
   - Add Vector Store tool node for answering questions.  
   - Add In-Memory Vector Store node; set mode to retrieve-as-tool.  
   - Add Pinecone Vector Store node; set mode and select Pinecone index.  
   - Add Postgres PGVector Store node; set mode.  
   - Add Supabase Vector Store node; set mode and table name.

4. **Add App Action Nodes (Block 1.4):**  
   - Add Bitly node.  
   - Add Dropbox node; configure download operation and OAuth2 credentials.  
   - Add Gmail node; configure options.  
   - Add Google Calendar node; select calendar and provide OAuth2 credentials.  
   - Add Google Docs node; set operation to get.  
   - Add Google Sheets node; set operation to append, select sheet and document, provide OAuth2 credentials.  
   - Add Pushbullet node.  
   - Add YouTube node; set resource to playlist and operation to create.  
   - Add Bluesky node.  
   - Add Perplexity node.  
   - Add ElevenLabs node; select voice ID and provide API credentials.  
   - Add Reddit node.  
   - Add Gmail Trigger node; configure polling interval.  
   - Add Google Sheets Trigger node; configure polling interval, sheet, document, and OAuth2 credentials.  
   - Add Twitter (X) node.

5. **Add Data Transformation Nodes (Block 1.5):**  
   - Add Code node; insert JavaScript code to add field "myNewField" with value 1.  
   - Add Date & Time node.  
   - Add Set node; add fields "Name Test" (string) and "Name 1" (number).  
   - Add Filter node; configure conditions as needed.  
   - Add Limit node.  
   - Add Remove Duplicates node.  
   - Add Split Out node.  
   - Add Aggregate node; configure fields to aggregate.  
   - Add Summarize node; configure fields.  
   - Add Convert to File node; set operation to toText.  
   - Add Extract from File node; set operation to pdf.  
   - Add HTML node; configure extraction values.  
   - Add Markdown node.  
   - Add Rename Keys node.  
   - Add Sort node.  
   - Add If node; configure conditions.  
   - Add Split In Batches node (Loop Over Items); default options.

6. **Add Flow & Core Nodes (Block 1.6):**  
   - Add NoOp node (Replace Me); connect input from Loop Over Items output 2.  
   - Add Execute Workflow node.  
   - Add Wait node; set resume to webhook.  
   - Add Execute Command node.  
   - Add HTTP Request node.  
   - Add Execution Data node.  
   - Add FTP node; set protocol to SFTP.  
   - Add Respond to Webhook node.  
   - Add Merge node.

7. **Add Miscellaneous AI Tools (Block 1.7):**  
   - Add Call n8n Workflow Tool node.  
   - Add Code Tool node.  
   - Add HTTP Request Langchain Tool node.  
   - Add Calculator node.  
   - Add SerpAPI node; provide SerpAPI credentials.  
   - Add Wikipedia node.  
   - Add Wolfram Alpha node.  
   - Add Langchain Gmail Tool node.  
   - Add Langchain Google Calendar Tool node; provide OAuth2 credentials.  
   - Add Langchain Google Docs Tool node.  
   - Add Langchain Google Sheets Tool node; provide OAuth2 credentials.  
   - Add MCP Client node.  
   - Add Anthropic Chat Model node.  
   - Add Google Gemini Chat Model node.  
   - Add OpenAI Chat Model node; provide OpenAI credentials.  
   - Add Window Buffer Memory node.  
   - Add Postgres Chat Memory node.  
   - Add Redis Chat Memory node.  
   - Add Item List Output Parser node.  
   - Add Structured Output Parser node.  
   - Add Embeddings Google Gemini node.  
   - Add Embeddings OpenAI node; provide OpenAI credentials.

8. **Add Sticky Notes:**  
   - Add sticky notes with colors and content to visually group nodes by block:  
     - Purple for Vector Memory  
     - Green for App Actions  
     - Gray for AI Tools  
     - Blue for Miscellaneous AI Tools  
     - Red for AI Agents  
     - Brown, Yellow, and others as per original layout for flow and data transformation sections.

9. **Connect Nodes:**  
   - Connect Replace Me node output to Loop Over Items input (second output).  
   - Other nodes are grouped visually and do not have explicit connections in this template; users can connect as needed for their use cases.

10. **Configure Credentials:**  
    - Set up and link credentials for OpenAI, Google OAuth2 (Drive, Calendar, Sheets), Dropbox OAuth2, ElevenLabs API, SerpAPI, Pinecone, Supabase, Postgres, Redis, Gmail, and others as required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| # WATCH THE n8n STARTER GUIDE 👇 [![Click Here!](https://i.imgur.com/GpUl9iS.png)](https://www.youtube.com/watch?v=It3CkokmodE&list=PL1Ylp5hLJfWeL9ZJ0MQ2sK5y2wPYKfZdE&index=1&pp=gAQBiAQBsAQB)                                                                                                                                                                         | Starter guide video on YouTube by @IversusAI                                                                    |
| This template is featured in the n8n Starter Guide series. The template is free, but comes with two additional PDFs and a Quick Start video if you grab the **[full download pack on gumroad](https://iversusai.gumroad.com/l/n8nstartupguidepack)**.                                                                                                                  | Full download pack with additional resources                                                                     |
| This workflow is a visual map of many useful n8n nodes grouped by categories like Triggers, AI tools, and App connectors. It acts as a handy visual reference guide to explore and copy nodes for your own workflows.                                                                                                                                                     | Workflow purpose                                                                                                 |
| Project credits to [@IversusAI](https://www.youtube.com/@IversusAI) on YouTube for providing this resource.                                                                                                                                                                                                                                                             | Project credit                                                                                                   |

---

This document provides a detailed, structured overview and analysis of the "Explore n8n Nodes in a Visual Reference Library" workflow, enabling users and AI agents to understand, reproduce, and extend the workflow effectively.