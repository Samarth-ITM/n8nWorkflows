Multi-Language Telegram RAG Chatbot with Supervisor AI & Automated Google Drive Pipeline

https://n8nworkflows.xyz/workflows/multi-language-telegram-rag-chatbot-with-supervisor-ai---automated-google-drive-pipeline-6158


# Multi-Language Telegram RAG Chatbot with Supervisor AI & Automated Google Drive Pipeline

### 1. Workflow Overview

This workflow implements a **Multi-Language Telegram Retrieval-Augmented Generation (RAG) Chatbot** combined with a **Supervisor AI** and an **Automated Google Drive document processing pipeline**. It is designed for managing documents and chat interactions in multiple languages, enriching chatbot responses with contextual data stored in vector databases, and maintaining synchronized Google Sheets and Drive files.

**Primary use cases include:**
- Automated ingestion and indexing of documents from Google Drive (Word, PDF, Google Docs).
- Vector embedding and storage of document content in Supabase for fast semantic search.
- Multi-language chat interactions via Telegram with AI-powered translation and supervision.
- Supervisory AI agent managing context and routing queries to specialized AI agents (Product, Academy, News).
- Periodic scheduled document updates and cleaning of old data.
- Integration with Google Sheets for managing links and manual document metadata.

**Logical blocks:**
- **1.1 Trigger & Input Reception:** Telegram messages and Google Drive file triggers, plus manual and scheduled triggers.
- **1.2 Document Processing Pipeline:** Downloading, converting, extracting text from files, splitting, embedding, and vector store updating.
- **1.3 Data Management:** Google Sheets data retrieval and update, Supabase vector store maintenance, and cleaning old data.
- **1.4 AI Agents & Chatbot Logic:** OpenAI chat models, Langchain agents for content formatting, translation, supervision, and specialized domain agents.
- **1.5 Output & Response:** Message formatting, translation output, and sending responses back to Telegram.
- **1.6 Control Flow & Utilities:** Wait nodes, conditionals (If/Switch), split batches, and merges to manage workflow logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Reception

**Overview:**  
This block handles the reception of input via Telegram, Google Drive triggers, and manual or scheduled triggers, initiating the workflow processes accordingly.

**Nodes Involved:**  
- Telegram Trigger  
- Google Drive File Created  
- Google Drive File Updated  
- Schedule Trigger1  
- When clicking ‘Test workflow’  
- When Executed Supervisor  

**Node Details:**

- **Telegram Trigger**  
  - Type: Trigger node for Telegram webhook messages.  
  - Config: Listens for incoming Telegram chat messages.  
  - Inputs: External via webhook. Outputs: To message preprocessing nodes.  
  - Failures: Telegram API downtime, webhook misconfiguration.  
- **Google Drive File Created / Updated**  
  - Type: Trigger for Google Drive file creation and update events.  
  - Config: Monitors specified Drive folders for new/changed files.  
  - Inputs: External webhook triggers. Outputs: To file processing batch nodes.  
  - Failures: Google Drive API permission errors, webhook failures.  
- **Schedule Trigger1**  
  - Type: Scheduled trigger node.  
  - Config: Runs periodically to initiate batch processing of manual documents and website links.  
  - Inputs: None. Outputs: To Google Sheets retrieval nodes.  
- **When clicking ‘Test workflow’**  
  - Type: Manual trigger node for testing workflow execution.  
  - Config: For manual start during design or debug.  
  - Outputs: To Google Sheets Website Links retrieval block.  
- **When Executed Supervisor**  
  - Type: Execute Workflow Trigger (sub-workflow call).  
  - Config: Triggers Supervisor AI agent workflow for chat memory and context management.  
  - Inputs: Internal triggers from conditions or higher-level logic.

---

#### 1.2 Document Processing Pipeline

**Overview:**  
Processes documents originating from Google Drive or manual input: downloading files, converting Word to Google Docs, extracting text from various formats, splitting text, generating embeddings, and updating vector stores.

**Nodes Involved:**  
- Download File / Download File1  
- Switch File Type / Switch File Type1  
- Extract From Document / Extract From Document1  
- Extract From PDF / Extract From PDF1  
- Convert Word to Google Docs  
- Google Drive - Delete File / Google Drive - Delete File1 / Google Drive - Delete File (second)  
- Set File Data / Set File Data1 / Set File Data3 / Set File Data4  
- Loop Over Files / Loop Over Files1 / Loop Over Files3 / Loop Over Files4  
- Delete Old Doc Rows / Delete Old Doc Rows1 / Delete Old Doc Rows3 / Delete Old Doc Rows4  
- Default Data Loader / Default Data Loader2  
- Character Text Splitter / Character Text Splitter1  
- Embeddings OpenAI / Embeddings OpenAI1 / Embeddings OpenAI2 / Embeddings OpenAI3 / Embeddings OpenAI4 / Embeddings OpenAI5  
- Insert Content into Supabase Vectorstore / Insert Content Into Supabase Vectorstore  
- Google Sheets Get Document  

**Node Details:**

- **Download File / Download File1**  
  - Type: Google Drive node to download files.  
  - Config: Downloads files based on IDs passed from triggers or batch loops.  
  - Inputs: File ID from prior node. Outputs: File binary for extraction.  
  - Failures: Access permission errors, file not found, large file size timeout.  
- **Switch File Type / Switch File Type1**  
  - Type: Switch node to split logic based on file type (e.g., docx, pdf).  
  - Config: Routes files to appropriate extraction nodes.  
  - Inputs: File metadata. Outputs: To extract nodes accordingly.  
  - Edge Cases: Unrecognized file types, corrupted files.  
- **Extract From Document / Extract From Document1**  
  - Type: Extract text from Google Docs, Word, or similar document formats.  
  - Outputs: Extracted plain text for further processing.  
  - Failures: Unsupported formats, extraction errors.  
- **Extract From PDF / Extract From PDF1**  
  - Type: Extract text from PDF files.  
  - Outputs: Extracted text for embedding.  
  - Failures: Scanned PDFs without OCR, extraction inaccuracies.  
- **Convert Word to Google Docs**  
  - Type: HTTP Request node calling Google Docs API or similar service.  
  - Config: Converts Word files to Google Docs format for easier text extraction.  
  - Failures: API rate limits, conversion errors.  
- **Google Drive - Delete File(s)**  
  - Type: Deletes files from Google Drive after processing or cleanup.  
  - Config: Runs on error continue mode to avoid workflow stops.  
- **Set File Data (multiple)**  
  - Type: Set node to configure or modify variables and data fields.  
  - Used for: Preparing metadata or tracking file states during loops.  
- **Loop Over Files (multiple)**  
  - Type: SplitInBatches node to process multiple files in batches for scalability and API quota control.  
- **Delete Old Doc Rows (multiple)**  
  - Type: Supabase node to delete old document metadata or vectorstore rows to maintain freshness.  
- **Default Data Loader / Default Data Loader2**  
  - Type: Langchain document loader node to prepare text chunks for embedding.  
- **Character Text Splitter / Character Text Splitter1**  
  - Type: Langchain text splitter node to chunk long texts into manageable pieces.  
- **Embeddings OpenAI (multiple)**  
  - Type: OpenAI embeddings node generating vector representations of text chunks.  
  - Requires: OpenAI API credentials.  
- **Insert Content into Supabase Vectorstore (multiple)**  
  - Type: Inserts embeddings and associated metadata into Supabase vector database for semantic search.  
- **Google Sheets Get Document**  
  - Retrieves metadata or document links from Google Sheets to drive processing.

---

#### 1.3 Data Management

**Overview:**  
Manages Google Sheets data for website links and manual documents, updates links, and cleans old data from Sheets and Supabase. Also handles aggregation of processed data and updates to Sheets.

**Nodes Involved:**  
- Google Sheets Website Links  
- Google Sheets Website Links - Update  
- Google Sheets - Manual Documents  
- Google Sheets - Manual Documents - Get Rows  
- Google Sheets Manual Documents - Delete Rows  
- Google Sheets Website Links - Delete Rows  
- Aggregate  
- Filter Links  
- Edit Fields Settings  
- Edit Fields Telegram Message1  
- Edit Fields Translator  
- Edit Fields Output  

**Node Details:**

- **Google Sheets Nodes**  
  - Types: Read (Get Rows), Update, and Delete rows in Google Sheets.  
  - Config: Manage two main sheets - Website Links and Manual Documents.  
  - Failures: API quota exceeded, permission errors, sheet not found.  
- **Aggregate**  
  - Type: Aggregates data from multiple inputs into combined output for batch updates.  
- **Filter Links**  
  - Type: Filter node to remove or select specific rows based on conditions.  
- **Edit Fields (multiple)**  
  - Type: Set nodes to edit or enrich data fields for consistency before passing to downstream nodes.

---

#### 1.4 AI Agents & Chatbot Logic

**Overview:**  
This block orchestrates multiple AI agents including OpenAI Chat Models, Langchain agents for translation, supervision, content formatting, and domain-specific RAG agents (Product, Academy, News). It manages chat memory and routing of queries to appropriate AI tools.

**Nodes Involved:**  
- OpenAI Chat Model (multiple instances)  
- Content Formatter  
- Structured Output Parser / Structured Output Parser1 / Structured Output Parser2  
- Translator1  
- Output First Translator  
- Output Translator  
- Supervisor AI Agent1  
- RAG AI Agent - Product  
- RAG AI Agent - Academy  
- RAG AI Agent - News  
- Postgres Chat Memory1  
- Supabase Vector Store / Supabase Vector Store1 / Supabase Vector Store2 / Supabase Vector Store3  
- News AI Agent  
- Academy AI Agent  
- Product AI Agent  
- Telegram Waiting  
- If Question  
- If English  
- If  

**Node Details:**

- **OpenAI Chat Model Nodes**  
  - Type: Langchain OpenAI chat completion nodes.  
  - Config: Used for generating AI responses, translation, and supervision.  
  - Inputs: Formatted user queries, context from vector stores or memory.  
  - Outputs: AI-generated text or structured data.  
  - Failures: API rate limits, invalid prompt formatting, connectivity issues.  
- **Langchain Agents (Content Formatter, Translator, Supervisors, RAG AI Agents)**  
  - Type: Langchain agent nodes orchestrating multi-step AI workflows and tool usage.  
  - Role: Format content, translate between languages, supervise query processing, fetch domain-specific knowledge from vector stores.  
  - Inputs: Chat inputs, vector search results, memory context.  
  - Outputs: Structured responses or translated messages.  
- **Structured Output Parser Nodes**  
  - Type: Parses AI output into structured JSON or predefined formats for further processing.  
- **Postgres Chat Memory1**  
  - Type: Stores chat history and context in a PostgreSQL memory backend.  
  - Role: Maintains conversation state for more coherent AI responses.  
- **Supabase Vector Store Nodes**  
  - Type: Connects to Supabase vector database for semantic similarity search.  
  - Role: Retrieves relevant document embeddings to augment AI responses.  
- **If Question / If English / If**  
  - Type: Conditional nodes to branch logic based on message content or language detection.  
- **Telegram Waiting**  
  - Type: Telegram node sending intermediate messages or typing indicators to users.

---

#### 1.5 Output & Response

**Overview:**  
Formats final AI-generated replies, optionally translates them, merges outputs, and sends the resulting messages back to the Telegram user.

**Nodes Involved:**  
- Edit Fields Output  
- Merge Output  
- Send a text message  

**Node Details:**

- **Edit Fields Output**  
  - Type: Set node to prepare final message fields before sending.  
- **Merge Output**  
  - Type: Merge node combines multiple translation or AI agent outputs into a single message.  
- **Send a text message**  
  - Type: Telegram node to send the final chat message to the user.  
  - Config: Uses Telegram credentials and chat ID from incoming message.  
  - Failures: Telegram API errors, invalid chat ID.

---

#### 1.6 Control Flow & Utilities

**Overview:**  
Contains utility nodes managing delays, batch processing, merging datasets, and sticky notes for documentation.

**Nodes Involved:**  
- Wait / Wait1 / Wait2 / Wait4 / Wait5  
- SplitInBatches (Loop Over Links, Loop Over Files, Loop Over Items)  
- Merge (Merge Document Output, Merge Output)  
- Sticky Notes (multiple)  

**Node Details:**

- **Wait Nodes**  
  - Type: Pauses workflow execution, often used to space API calls or wait for asynchronous processes.  
- **SplitInBatches Nodes**  
  - Type: Splits large datasets into smaller batches for processing to avoid API limits.  
- **Merge Nodes**  
  - Type: Combine multiple data branches back into a single stream.  
- **Sticky Notes**  
  - Used for inline documentation and developer notes. No functional role.

---

### 3. Summary Table

| Node Name                         | Node Type                              | Functional Role                             | Input Node(s)                        | Output Node(s)                       | Sticky Note                        |
|----------------------------------|--------------------------------------|---------------------------------------------|------------------------------------|------------------------------------|----------------------------------|
| When clicking ‘Test workflow’    | Manual Trigger                       | Manual start for testing workflow            |                                    | Edit Fields Settings               |                                  |
| Edit Fields Settings             | Set                                 | Prepare initial settings/data                 | When clicking ‘Test workflow’      | Google Sheets Website Links        |                                  |
| Google Sheets Website Links      | Google Sheets                       | Retrieve list of website links                | Edit Fields Settings               | Filter Links                      |                                  |
| Filter Links                    | Filter                              | Filter website links as per criteria          | Google Sheets Website Links        | Loop Over Links                   |                                  |
| Loop Over Links                 | SplitInBatches                      | Process website links in batches               | Filter Links                      | HTTP Request - Crawl4AI            |                                  |
| HTTP Request - Crawl4AI          | HTTP Request                       | Crawl links for AI content                      | Loop Over Links                   | Edit Fields                      |                                  |
| Edit Fields                    | Set                                 | Format and prepare crawled data                | HTTP Request - Crawl4AI            | Content Formatter                |                                  |
| Content Formatter              | Langchain Agent                    | Format AI content from crawled data            | Edit Fields                      | If Document ID                   |                                  |
| If Document ID                 | If                                  | Check if document ID exists                      | Content Formatter                | Get Text From Doc / Google Docs - Create |                                  |
| Get Text From Doc              | Google Docs                        | Retrieve text from existing Google Docs         | If Document ID                   | Update Text To Doc               |                                  |
| Update Text To Doc             | Google Docs                        | Update existing Google Docs with new text       | Get Text From Doc                | Merge Document Output            |                                  |
| Google Docs - Create           | Google Docs                        | Create new Google Docs document                  | If Document ID (false path)       | Save Text To Doc                |                                  |
| Save Text To Doc              | Google Docs                        | Save extracted text into Google Docs             | Google Docs - Create             | Merge Document Output            |                                  |
| Merge Document Output          | Merge                              | Combine document update/create outputs           | Update Text To Doc / Save Text To Doc | Google Sheets Website Links - Update |                                  |
| Google Sheets Website Links - Update | Google Sheets                       | Update website links metadata                      | Merge Document Output            | Wait                             |                                  |
| Wait                         | Wait                                | Delay between batches                             | Google Sheets Website Links - Update | Loop Over Links               |                                  |
| Loop Over Files               | SplitInBatches                      | Process files from Google Drive in batches         | Google Drive File Created/Updated | Set File Data                   |                                  |
| Set File Data                | Set                                 | Prepare file metadata for processing               | Loop Over Files                  | Google Sheets Get Document       |                                  |
| Google Sheets Get Document    | Google Sheets                       | Retrieve document metadata from Sheets             | Set File Data                   | Delete Old Doc Rows              |                                  |
| Delete Old Doc Rows           | Supabase                          | Clean old document vectorstore rows                | Google Sheets Get Document       | Download File                   |                                  |
| Download File                | Google Drive                      | Download files from Drive                           | Delete Old Doc Rows              | Switch File Type                |                                  |
| Switch File Type             | Switch                             | Route files based on type (doc/pdf)                | Download File                   | Extract From Document / Extract From PDF |                                  |
| Extract From Document        | Extract From File                 | Extract text from docs                              | Switch File Type                | Insert Content into Supabase Vectorstore |                                  |
| Extract From PDF             | Extract From File                 | Extract text from PDFs                              | Switch File Type                | Insert Content into Supabase Vectorstore |                                  |
| Insert Content into Supabase Vectorstore | Langchain VectorStore Supabase    | Store embeddings and metadata                        | Extract From Document / Extract From PDF | Aggregate                     |                                  |
| Aggregate                    | Aggregate                         | Aggregate vectorstore insert outputs                 | Insert Content into Supabase Vectorstore | Google Sheets Manual Documents |                                  |
| Google Sheets - Manual Documents | Google Sheets                      | Manage manual document metadata                       | Aggregate                      | Wait2                           |                                  |
| Wait1                        | Wait                              | Delay for batch processing                            | Insert Content into Supabase Vectorstore | Loop Over Files               |                                  |
| Loop Over Files1             | SplitInBatches                    | Process second batch of files                          | Wait1                         | Set File Data1                 |                                  |
| Set File Data1               | Set                               | Prepare metadata for second batch files               | Loop Over Files1              | Delete Old Doc Rows1           |                                  |
| Delete Old Doc Rows1         | Supabase                        | Clean old rows for second batch                         | Set File Data1               | Download File1                |                                  |
| Download File1              | Google Drive                    | Download files for second batch                         | Delete Old Doc Rows1         | Switch File Type1             |                                  |
| Switch File Type1           | Switch                           | Route second batch files by type                        | Download File1              | Extract From Document1 / Extract From PDF1 |                                  |
| Extract From Document1      | Extract From File               | Extract text from document (second batch)               | Switch File Type1           | Insert Content Into Supabase Vectorstore |                                  |
| Extract From PDF1           | Extract From File               | Extract text from PDF (second batch)                    | Switch File Type1           | Insert Content Into Supabase Vectorstore |                                  |
| Insert Content Into Supabase Vectorstore | Langchain VectorStore Supabase  | Insert embeddings (second batch)                         | Extract From Document1 / Extract From PDF1 | Aggregate                     |                                  |
| Wait2                       | Wait                            | Delay before processing manual documents                 | Google Sheets - Manual Documents | Loop Over Files1              |                                  |
| Loop Over Items             | SplitInBatches                  | Process items from sheets in batches                       | Set File Get Data / Set File Get Data1 | If / Loop Over Files1        |                                  |
| Set File Get Data           | Set                             | Prepare data fields for batch processing                   | Google Drive File Created1 / Google Drive File Updated1 | Loop Over Items          |                                  |
| Edit Fields Telegram Message1 | Set                             | Prepare Telegram message data                              | Telegram Trigger             | Translator1                   |                                  |
| Translator1                 | Langchain Agent                | Translate incoming message                               | Edit Fields Telegram Message1 | Edit Fields Translator        |                                  |
| Edit Fields Translator      | Set                             | Prepare translated fields                                | Translator1                 | Output First Translator        |                                  |
| Output First Translator     | Langchain Agent                | Initial output translation                               | Edit Fields Translator      | Telegram Waiting              |                                  |
| Telegram Waiting            | Telegram                        | Send typing indicator or waiting message                   | Output First Translator     | If Question                  |                                  |
| If Question                | If                              | Check if message is a question                            | Telegram Waiting            | Supervisor AI Agent1          |                                  |
| Supervisor AI Agent1       | Langchain Agent                | Supervise AI interactions, manage chat memory              | If Question                | If English                   |                                  |
| If English                 | If                              | Check if response language is English                      | Supervisor AI Agent1        | Edit Fields Output / Output Translator |                                  |
| Edit Fields Output         | Set                             | Prepare final output fields                               | If English                 | Merge Output                 |                                  |
| Output Translator          | Langchain Agent                | Translate response if needed                               | If English (false path)    | Merge Output                 |                                  |
| Merge Output               | Merge                           | Merge translated and original outputs                      | Edit Fields Output / Output Translator | Send a text message         |                                  |
| Send a text message        | Telegram                       | Send final chatbot response to user                        | Merge Output               |                              |                                  |
| Postgres Chat Memory1      | Langchain Memory Postgres     | Store and retrieve chat history                            |                              | Supervisor AI Agent1          |                                  |
| Supabase Vector Store (various) | Langchain VectorStore Supabase | Connect to different vector stores for Product, Academy, News agents |                              | AI agents                    |                                  |
| RAG AI Agent - Product     | Langchain Agent               | Product domain knowledge retrieval                          | OpenAI Chat Model4          | Supervisor AI Agent1          |                                  |
| RAG AI Agent - Academy     | Langchain Agent               | Academy domain knowledge retrieval                          | OpenAI Chat Model6          | Supervisor AI Agent1          |                                  |
| RAG AI Agent - News       | Langchain Agent               | News domain knowledge retrieval                             | OpenAI Chat Model7          | Supervisor AI Agent1          |                                  |
| News AI Agent              | Langchain ToolWorkflow        | Specialized news AI agent workflow                          | Supervisor AI Agent1        |                              |                                  |
| Academy AI Agent           | Langchain ToolWorkflow        | Specialized academy AI agent workflow                       | Supervisor AI Agent1        |                              |                                  |
| Product AI Agent           | Langchain ToolWorkflow        | Specialized product AI agent workflow                       | Supervisor AI Agent1        |                              |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up credentials:**  
   - Configure Google Drive, Google Sheets, Telegram Bot, OpenAI, and Supabase credentials in n8n.

2. **Triggers:**
   - Create a **Telegram Trigger** node to listen to incoming messages.  
   - Add **Google Drive File Created** and **Google Drive File Updated** triggers for monitoring files.  
   - Add a **Schedule Trigger** node to run periodic batch jobs.  
   - Add a **Manual Trigger** node for manual testing.

3. **Initial Data Retrieval:**
   - Add **Google Sheets** nodes to retrieve lists of website links and manual documents.  
   - Use **Set** nodes to prepare and edit fields from trigger data.

4. **Batch Processing:**
   - Add **SplitInBatches** nodes to process files and links in manageable chunks.

5. **File Download and Type Routing:**
   - Add **Google Drive Download File** nodes for downloading files.  
   - Add **Switch** nodes to route files based on type (docx, pdf, etc.).

6. **File Conversion and Text Extraction:**
   - For Word files, add an **HTTP Request** node for conversion to Google Docs.  
   - Add **Extract From File** nodes for document and PDF text extraction.

7. **Text Processing:**
   - Add Langchain **Text Splitter** nodes to divide large text into chunks.  
   - Add Langchain **Default Data Loader** nodes to prepare chunks for embedding.

8. **Embeddings and Vector Store:**
   - Add **OpenAI Embeddings** nodes to generate vector embeddings.  
   - Add **Supabase Vector Store** nodes to insert embeddings and metadata.

9. **Data Cleaning:**
   - Add **Supabase** nodes to delete old document rows before inserting new ones.  
   - Add **Google Sheets Delete Rows** nodes to clean outdated records.

10. **AI Agents:**
    - Add multiple **OpenAI Chat Model** nodes for chat completions.  
    - Add Langchain **Agent** nodes for:  
      - Content formatting  
      - Translation  
      - Supervisor AI  
      - Domain-specific RAG agents (Product, Academy, News)  
    - Add **Structured Output Parser** nodes to parse AI outputs.

11. **Chat Memory:**
    - Add **Postgres Chat Memory** node to maintain conversation context.

12. **Message Preparation and Routing:**
    - Add **Set** nodes to format messages for Telegram.  
    - Add **If** nodes to check message language and question presence.  
    - Add **Merge** nodes to combine multiple AI outputs.

13. **Sending Responses:**
    - Add **Telegram** node to send final text messages to users.

14. **Control Flow:**
    - Insert **Wait** nodes as needed to manage API rate limits and asynchronous processes.

15. **Testing and Debugging:**
    - Use **Manual Trigger** to start workflow runs for testing.  
    - Add **Sticky Notes** for documentation and clarification.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                   |
|-------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow is a Market Template called "AIAutomationPro Ultimate RAG Chatbot v1" | Found in n8n Marketplace under AI Automation templates.                                         |
| Uses Langchain nodes for advanced AI orchestration with OpenAI integration    | Requires n8n Langchain nodes and OpenAI API access.                                             |
| Integrates with Supabase vector store for semantic search                     | Supabase configuration is mandatory for vector storage and retrieval.                           |
| Multi-language support with translators and output parsers                   | Enables seamless conversation in various languages, useful for global chatbot applications.    |
| Telegram API webhook requires proper setup and bot token                      | Telegram bot must be created and webhook URL configured in Telegram Botfather.                  |
| Google Drive and Sheets API require OAuth2 credentials                        | Ensure OAuth credentials allow file read/write and spreadsheet access.                          |
| Workflow includes error handling with "continue on fail" for file deletions  | Prevents workflow halts due to non-critical errors during cleanup steps.                        |
| Scheduled triggers enable regular updates and cleaning of document indexes   | Helps maintain freshness and relevancy of indexed data for chatbot responses.                   |
| Workflow is designed for scalability with batch processing and delays        | Avoids hitting API rate limits and manages large document volumes efficiently.                  |

---

This document provides a thorough, structured reference for the workflow “Multi-Language Telegram RAG Chatbot with Supervisor AI & Automated Google Drive Pipeline,” enabling advanced users and AI agents to understand, reproduce, and extend the system effectively.