Build a document-upload RAG chatbot with OpenAI, Pinecone and daily analytics

https://n8nworkflows.xyz/workflows/build-a-document-upload-rag-chatbot-with-openai--pinecone-and-daily-analytics-14041


# Build a document-upload RAG chatbot with OpenAI, Pinecone and daily analytics

# 1. Workflow Overview

This workflow implements a document-based RAG system in n8n with three parallel capabilities:

1. **Document ingestion and indexing**: users upload files through a form, files are processed into chunks, embedded with OpenAI, and stored in Pinecone.
2. **Chat-based question answering**: users ask questions through a chat interface, the agent retrieves relevant document chunks from Pinecone, and answers strictly from retrieved content.
3. **Daily analytics and reporting**: chat interactions are logged, summarized once per day, and emailed as an HTML report.

It is designed for use cases such as internal knowledge assistants, document Q&A bots, and lightweight observability of chatbot usage.

## 1.1 Document Upload Interface
Handles incoming uploaded files and initializes reusable configuration values such as Pinecone index name and chunking settings.

## 1.2 Document Processing and Vector Indexing
Loads uploaded files, splits them into chunks, generates embeddings, and inserts those chunks into Pinecone.

## 1.3 Chat Interface and Retrieval-Augmented Answering
Provides a public chat endpoint with conversational memory, retrieves document context from Pinecone as a tool, and generates constrained answers using an OpenAI chat model.

## 1.4 Chat Logging
Captures the question, generated answer, timestamp, session ID, and a best-effort file reference and stores them for later analytics.

## 1.5 Daily Analytics and Email Reporting
Runs on a daily schedule, queries recent chat logs, computes usage metrics, builds an HTML summary, and sends it by email.

---

# 2. Block-by-Block Analysis

## 2.1 Document Upload Interface

**Overview:**  
This block exposes a form where users upload documents and then enriches the incoming payload with central configuration values used later in indexing. It is the entry point for ingestion.

**Nodes Involved:**  
- Document Upload Form
- Workflow Configuration
- Store Form Responses

### Node: Document Upload Form
- **Type and role:** `n8n-nodes-base.formTrigger`  
  Public form trigger that accepts uploaded files.
- **Configuration choices:**  
  - Form title: **Document Upload for RAG**
  - Description: users can upload **PDF, CSV, or JSON**
  - One file field labeled **Upload Documents**
  - Submit button label: **Upload Documents**
  - n8n attribution disabled
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Entry point node
  - Outputs to **Workflow Configuration**
- **Version-specific requirements:**  
  Uses `typeVersion 2.3`; requires n8n version supporting Form Trigger with file upload fields.
- **Edge cases / failures:**  
  - Unsupported file type may be rejected at upload time
  - Large files may exceed n8n upload or instance limits
  - If multiple files are uploaded but downstream nodes expect a single binary field, behavior may vary depending on how n8n stores the upload
- **Sub-workflow reference:** None

### Node: Workflow Configuration
- **Type and role:** `n8n-nodes-base.set`  
  Central configuration node that defines reusable runtime values.
- **Configuration choices:**  
  Adds these fields while preserving prior input data:
  - `pineconeIndex`: placeholder for Pinecone index name
  - `pineconeNamespace`: `documents`
  - `chunkSize`: `1000`
  - `chunkOverlap`: `200`
  - `topK`: `5`
  - `includeOtherFields: true`
- **Key expressions or variables used:**  
  These values are later referenced through expressions such as:
  - `$('Workflow Configuration').first().json.chunkSize`
  - `$('Workflow Configuration').first().json.pineconeIndex`
- **Input and output connections:**  
  - Input from **Document Upload Form**
  - Outputs to **Store Form Responses**
  - Outputs to **Pinecone Insert Documents**
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`
- **Edge cases / failures:**  
  - Placeholder Pinecone index must be replaced, otherwise Pinecone operations will fail
  - `topK` is defined but not actually used in the retrieval node as currently configured
- **Sub-workflow reference:** None

### Node: Store Form Responses
- **Type and role:** `n8n-nodes-base.dataTable`  
  Stores upload form submissions into a Data Table for recordkeeping.
- **Configuration choices:**  
  - Data table ID: `form_responses`
  - Auto-maps input data into columns
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from **Workflow Configuration**
  - No downstream connection
- **Version-specific requirements:**  
  Uses `typeVersion 1`; requires Data Tables feature enabled in n8n.
- **Edge cases / failures:**  
  - Table `form_responses` must already exist or be selectable
  - Auto-mapping may create schema mismatches if upload payload shape changes
  - Binary content may not be stored meaningfully depending on Data Table behavior
- **Sub-workflow reference:** None

---

## 2.2 Document Processing and Vector Indexing

**Overview:**  
This block converts uploaded files into chunked text documents, generates embeddings, and writes them into Pinecone for retrieval later by the chatbot.

**Nodes Involved:**  
- Text Splitter
- Document Loader
- OpenAI Embeddings
- Pinecone Insert Documents

### Node: Text Splitter
- **Type and role:** `@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter`  
  Splits text into overlapping chunks suitable for embedding and retrieval.
- **Configuration choices:**  
  - Chunk size from workflow config
  - Chunk overlap from workflow config
  - Default recursive character splitting behavior
- **Key expressions or variables used:**  
  - `={{ $('Workflow Configuration').first().json.chunkSize }}`
  - `={{ $('Workflow Configuration').first().json.chunkOverlap }}`
- **Input and output connections:**  
  - Connected as `ai_textSplitter` into **Document Loader**
- **Version-specific requirements:**  
  Uses `typeVersion 1`; requires LangChain nodes available in the n8n instance.
- **Edge cases / failures:**  
  - If configuration values are missing or non-numeric, expression resolution fails
  - Too-small chunks may hurt retrieval quality; too-large chunks may exceed embedding/model constraints
- **Sub-workflow reference:** None

### Node: Document Loader
- **Type and role:** `@n8n/n8n-nodes-langchain.documentDefaultDataLoader`  
  Reads uploaded file binary data and converts it into documents.
- **Configuration choices:**  
  - Loader selected: `pdfLoader`
  - Data type: `binary`
  - Binary mode: `specificField`
  - Text splitting mode: `custom`
  - Receives text splitter from **Text Splitter**
- **Key expressions or variables used:** None explicit in parameters
- **Input and output connections:**  
  - Receives `ai_textSplitter` from **Text Splitter**
  - Outputs `ai_document` to **Pinecone Insert Documents**
- **Version-specific requirements:**  
  Uses `typeVersion 1.1`
- **Edge cases / failures:**  
  - Important mismatch: the form accepts PDF, CSV, and JSON, but the loader is configured as **PDF loader only**
  - CSV/JSON uploads are likely to fail or be parsed incorrectly
  - `specificField` binary mode usually requires a binary property name; none is shown explicitly here, so successful execution depends on the node’s defaults or UI state
  - Scanned PDFs without extractable text may produce poor results unless OCR is used externally
- **Sub-workflow reference:** None

### Node: OpenAI Embeddings
- **Type and role:** `@n8n/n8n-nodes-langchain.embeddingsOpenAi`  
  Generates vector embeddings for document chunks and retrieval queries.
- **Configuration choices:**  
  Uses default OpenAI embedding settings.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Outputs `ai_embedding` to **Pinecone Insert Documents**
  - Outputs `ai_embedding` to **Pinecone Vector Store**
- **Version-specific requirements:**  
  Uses `typeVersion 1.2`
- **Edge cases / failures:**  
  - Missing or invalid OpenAI credentials
  - Rate limits or quota exhaustion
  - Large input volumes can increase cost and latency
- **Sub-workflow reference:** None

### Node: Pinecone Insert Documents
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Inserts embedded document chunks into a Pinecone index.
- **Configuration choices:**  
  - Mode: `insert`
  - Namespace from workflow configuration
  - Pinecone index from workflow configuration
  - Receives documents from **Document Loader**
  - Receives embeddings from **OpenAI Embeddings**
  - Receives main flow input from **Workflow Configuration**
- **Key expressions or variables used:**  
  - Namespace: `={{ $('Workflow Configuration').first().json.pineconeNamespace }}`
  - Index: `={{ $('Workflow Configuration').first().json.pineconeIndex }}`
- **Input and output connections:**  
  - Main input from **Workflow Configuration**
  - `ai_document` from **Document Loader**
  - `ai_embedding` from **OpenAI Embeddings**
  - No downstream main node
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`
- **Edge cases / failures:**  
  - Pinecone index must exist and match the embedding dimension expected by the selected OpenAI embeddings model
  - Invalid namespace/index expression causes runtime failure
  - Upserts may duplicate content unless IDs or deduplication strategies are configured elsewhere
- **Sub-workflow reference:** None

---

## 2.3 Chat Interface and Retrieval-Augmented Answering

**Overview:**  
This block provides a public chat experience backed by conversational memory and Pinecone retrieval. The agent is explicitly instructed to answer only from retrieved document context.

**Nodes Involved:**  
- Chat Trigger
- Chat Memory
- OpenAI Chat Model
- Pinecone Vector Store
- RAG Chatbot Agent

### Node: Chat Trigger
- **Type and role:** `@n8n/n8n-nodes-langchain.chatTrigger`  
  Public chat entry point for user questions.
- **Configuration choices:**  
  - Public: enabled
  - Loads previous session from memory
  - Initial message: invites questions about uploaded documents
- **Key expressions or variables used:**  
  Downstream uses:
  - `$json.chatInput`
  - `$json.sessionId`
- **Input and output connections:**  
  - Entry point node
  - Outputs main flow to **RAG Chatbot Agent**
  - Connected to **Chat Memory** on `ai_memory`
- **Version-specific requirements:**  
  Uses `typeVersion 1.4`
- **Edge cases / failures:**  
  - Public exposure means anyone with the URL may access it unless protected externally
  - Session continuity depends on how n8n stores and passes session state
- **Sub-workflow reference:** None

### Node: Chat Memory
- **Type and role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
  Supplies recent conversation memory to the agent.
- **Configuration choices:**  
  - Context window length: `10`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Connected on `ai_memory` between **Chat Trigger** and **RAG Chatbot Agent**
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`
- **Edge cases / failures:**  
  - Memory is limited to 10 turns/windows; earlier context is dropped
  - If session handling breaks, memory may not persist across messages
- **Sub-workflow reference:** None

### Node: OpenAI Chat Model
- **Type and role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  The LLM used by the agent to compose final answers.
- **Configuration choices:**  
  - Model: `gpt-4o-mini`
  - No extra model options configured
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Provides `ai_languageModel` to **RAG Chatbot Agent**
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`
- **Edge cases / failures:**  
  - Requires valid OpenAI credentials and model access
  - Rate limits, quota exhaustion, or provider-side errors
- **Sub-workflow reference:** None

### Node: Pinecone Vector Store
- **Type and role:** `@n8n/n8n-nodes-langchain.vectorStorePinecone`  
  Exposes Pinecone retrieval as a tool callable by the agent.
- **Configuration choices:**  
  - Mode: `retrieve-as-tool`
  - No namespace configured in this node
  - Pinecone index field is effectively empty in the provided JSON (`value: ""`)
  - Receives embeddings from **OpenAI Embeddings**
- **Key expressions or variables used:** None currently
- **Input and output connections:**  
  - Receives `ai_embedding` from **OpenAI Embeddings**
  - Outputs `ai_tool` to **RAG Chatbot Agent**
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`
- **Edge cases / failures:**  
  - As configured, this is incomplete: Pinecone index is not set
  - Namespace mismatch with ingestion can cause no results even if indexing succeeds
  - `topK` from configuration is not wired here, so retrieval depth may use default behavior
- **Sub-workflow reference:** None

### Node: RAG Chatbot Agent
- **Type and role:** `@n8n/n8n-nodes-langchain.agent`  
  Central orchestration node that receives the user prompt, uses Pinecone retrieval, applies memory, and generates the answer.
- **Configuration choices:**  
  - Prompt source: `={{ $json.chatInput }}`
  - Prompt type: `define`
  - Strong system message instructing the assistant to:
    1. review vector-store context
    2. answer only from retrieved documents
    3. return `"I couldn't find that in the uploaded documents."` when context is insufficient
    4. cite documents when possible
    5. be concise and accurate
    6. never use outside knowledge
- **Key expressions or variables used:**  
  - Input text: `={{ $json.chatInput }}`
- **Input and output connections:**  
  - Main input from **Chat Trigger**
  - `ai_memory` from **Chat Memory**
  - `ai_languageModel` from **OpenAI Chat Model**
  - `ai_tool` from **Pinecone Vector Store**
  - Main output to **Prepare Chat Log**
- **Version-specific requirements:**  
  Uses `typeVersion 3`
- **Edge cases / failures:**  
  - If the retrieval tool is misconfigured, the agent may fail or return empty-context answers
  - Citation behavior depends on retrieved metadata quality
  - The model may still produce weakly inferred answers unless prompts are enforced and tool output is reliable
- **Sub-workflow reference:** None

---

## 2.4 Chat Logging

**Overview:**  
This block transforms chatbot output into a log-friendly schema and stores each interaction for analytics, debugging, and future reporting.

**Nodes Involved:**  
- Prepare Chat Log
- Log Chat Interactions

### Node: Prepare Chat Log
- **Type and role:** `n8n-nodes-base.set`  
  Creates a structured chat log record.
- **Configuration choices:**  
  Assigns:
  - `userQuery` from chat input
  - `botResponse` from agent output
  - `timestamp` from current time
  - `sessionId` from chat session
  - `matchedFiles` from metadata filename if available, else `N/A`
- **Key expressions or variables used:**  
  - `={{ $json.chatInput }}`
  - `={{ $json.output }}`
  - `={{ $now.toISO() }}`
  - `={{ $json.sessionId }}`
  - `={{ $json.metadata?.fileName || 'N/A' }}`
- **Input and output connections:**  
  - Input from **RAG Chatbot Agent**
  - Output to **Log Chat Interactions**
- **Version-specific requirements:**  
  Uses `typeVersion 3.4`
- **Edge cases / failures:**  
  - `metadata.fileName` may not exist at the top level of the agent output, so `matchedFiles` may often be `N/A`
  - The field is named `matchedFiles`, but analytics later expects `filesReferenced`
- **Sub-workflow reference:** None

### Node: Log Chat Interactions
- **Type and role:** `n8n-nodes-base.dataTable`  
  Persists chat log records to a Data Table.
- **Configuration choices:**  
  - Data table ID: `chat_logs`
  - Auto-map input fields
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input from **Prepare Chat Log**
  - No downstream connection
- **Version-specific requirements:**  
  Uses `typeVersion 1`
- **Edge cases / failures:**  
  - Data table `chat_logs` must exist
  - Auto-mapping may produce sparse or inconsistent schemas if upstream fields change
- **Sub-workflow reference:** None

---

## 2.5 Daily Analytics and Email Reporting

**Overview:**  
This block runs every morning, pulls the last day of chat logs, computes summary statistics, generates an HTML email report, and sends it to a configured recipient.

**Nodes Involved:**  
- Daily Summary Schedule
- Get Chat Logs
- Analyze Chat Data
- Send Daily Summary Email

### Node: Daily Summary Schedule
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Scheduled entry point for daily reporting.
- **Configuration choices:**  
  - Runs daily at hour `9`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Entry point node
  - Outputs to **Get Chat Logs**
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`
- **Edge cases / failures:**  
  - Trigger time is based on workflow/server timezone settings
  - If the workflow is inactive, schedule will not run
- **Sub-workflow reference:** None

### Node: Get Chat Logs
- **Type and role:** `n8n-nodes-base.dataTable`  
  Retrieves chat logs newer than 24 hours ago.
- **Configuration choices:**  
  - Operation: `get`
  - Return all: true
  - Filter condition:
    - `timestamp` greater than `{{ $now.minus({ days: 1 }).toISO() }}`
  - Data table ID: `chat_logs`
- **Key expressions or variables used:**  
  - `={{ $now.minus({ days: 1 }).toISO() }}`
- **Input and output connections:**  
  - Input from **Daily Summary Schedule**
  - Output to **Analyze Chat Data**
- **Version-specific requirements:**  
  Uses `typeVersion 1`
- **Edge cases / failures:**  
  - Assumes timestamps are stored in ISO string format and compare correctly
  - If timestamps are malformed or stored differently, filtering may be inaccurate
- **Sub-workflow reference:** None

### Node: Analyze Chat Data
- **Type and role:** `n8n-nodes-base.code`  
  Aggregates chat logs and creates HTML summary content.
- **Configuration choices:**  
  JavaScript code:
  - counts total questions
  - tries to count referenced files from `data.filesReferenced`
  - counts failed lookups when:
    - `documentsFound === 0`
    - or `status === 'failed'`
    - or `noResults === true`
  - computes success rate
  - renders a styled HTML email
  - returns summary fields plus `emailBody`
- **Key expressions or variables used:**  
  Uses `$input.all()` to process all log items.
- **Input and output connections:**  
  - Input from **Get Chat Logs**
  - Output to **Send Daily Summary Email**
- **Version-specific requirements:**  
  Uses `typeVersion 2`
- **Edge cases / failures:**  
  - Field mismatch: logger writes `matchedFiles`, but analytics reads `filesReferenced`
  - Field mismatch: logger does not write `documentsFound`, `status`, or `noResults`, so failed lookup metrics will likely stay `0`
  - If there are no logs, the email still sends with zeroed stats
  - HTML generation may fail only if upstream data contains unexpected non-serializable values
- **Sub-workflow reference:** None

### Node: Send Daily Summary Email
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends the generated HTML analytics report by email.
- **Configuration choices:**  
  - Recipient: placeholder email address
  - Subject includes current date
  - Message body from `emailBody`
- **Key expressions or variables used:**  
  - `={{ $json.emailBody }}`
  - `=Daily RAG Chatbot Summary - {{ $now.toFormat('yyyy-MM-dd') }}`
- **Input and output connections:**  
  - Input from **Analyze Chat Data**
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`; requires Gmail credentials configured in n8n.
- **Edge cases / failures:**  
  - Placeholder email must be replaced
  - Gmail OAuth authentication may expire or require re-consent
  - HTML rendering depends on email client
- **Sub-workflow reference:** None

---

## 2.6 Sticky Notes / Documentation Nodes

These nodes are documentation-only and have no execution role, but they are part of the workflow and should be preserved for clarity.

### Node: Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents vector indexing purpose.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents document processing pipeline.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents workflow configuration values.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents document upload interface.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note4
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents RAG answering behavior.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note5
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents chat interface.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note7
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents chat logging purpose.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note8
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents chat log analysis metrics.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note9
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents daily analytics trigger.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note10
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents daily summary report.
- **Connections:** None
- **Edge cases:** None

### Node: Sticky Note6
- **Type and role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Documents the full daily analytics purpose.
- **Connections:** None
- **Edge cases:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Document Upload Form | n8n-nodes-base.formTrigger | Accepts uploaded documents via web form |  | Workflow Configuration | ## Document Upload Interface<br><br>Users upload PDF, CSV, or JSON files through a form. |
| Workflow Configuration | n8n-nodes-base.set | Defines shared Pinecone and chunking settings | Document Upload Form | Store Form Responses; Pinecone Insert Documents | ## Workflow Configuration<br><br>Defines key settings used throughout the workflow:<br>Pinecone index name,namespace,chunk size, chunk overlap, retrieval depth (top-K) |
| Store Form Responses | n8n-nodes-base.dataTable | Stores upload form submissions | Workflow Configuration |  |  |
| Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits extracted text into chunks |  | Document Loader | ## Document Processing Pipeline<br><br>Uploaded files are loaded and converted into text.<br><br>## Vector Database Indexing<br><br>Document chunks are converted into embeddings and stored in Pinecone.<br><br>This creates a searchable vector index that allows the chatbot to retrieve relevant document context during conversations. |
| Document Loader | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads uploaded file binary into LangChain documents | Text Splitter | Pinecone Insert Documents | ## Document Processing Pipeline<br><br>Uploaded files are loaded and converted into text.<br><br>## Vector Database Indexing<br><br>Document chunks are converted into embeddings and stored in Pinecone.<br><br>This creates a searchable vector index that allows the chatbot to retrieve relevant document context during conversations. |
| OpenAI Embeddings | @n8n/n8n-nodes-langchain.embeddingsOpenAi | Generates embeddings for indexing and retrieval |  | Pinecone Insert Documents; Pinecone Vector Store |  |
| Pinecone Insert Documents | @n8n/n8n-nodes-langchain.vectorStorePinecone | Inserts document embeddings into Pinecone | Workflow Configuration; Document Loader; OpenAI Embeddings |  | ## Vector Database Indexing<br><br>Document chunks are converted into embeddings and stored in Pinecone.<br><br>This creates a searchable vector index that allows the chatbot to retrieve relevant document context during conversations. |
| Chat Trigger | @n8n/n8n-nodes-langchain.chatTrigger | Public chat entry point | Chat Memory | RAG Chatbot Agent | ## Chat Interface<br><br>Users interact with the knowledge base through a chat interface. |
| Chat Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains recent conversation context |  | Chat Trigger; RAG Chatbot Agent | ## RAG Question Answering<br><br>The chatbot retrieves relevant document chunks from Pinecone and uses them as context for the LLM.<br><br>The assistant answers using only retrieved information from the uploaded documents. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides LLM for answer generation |  | RAG Chatbot Agent | ## RAG Question Answering<br><br>The chatbot retrieves relevant document chunks from Pinecone and uses them as context for the LLM.<br><br>The assistant answers using only retrieved information from the uploaded documents. |
| RAG Chatbot Agent | @n8n/n8n-nodes-langchain.agent | Coordinates memory, retrieval, and answer generation | Chat Trigger; Chat Memory; OpenAI Chat Model; Pinecone Vector Store | Prepare Chat Log | ## RAG Question Answering<br><br>The chatbot retrieves relevant document chunks from Pinecone and uses them as context for the LLM.<br><br>The assistant answers using only retrieved information from the uploaded documents. |
| Prepare Chat Log | n8n-nodes-base.set | Formats chatbot interaction into log fields | RAG Chatbot Agent | Log Chat Interactions | ## Chat Logging<br><br>User queries and chatbot responses are captured with metadata such as session ID and timestamps.<br><br>This enables conversation tracking, analytics, and debugging. |
| Log Chat Interactions | n8n-nodes-base.dataTable | Stores chat logs for analytics | Prepare Chat Log |  | ## Chat Logging<br><br>User queries and chatbot responses are captured with metadata such as session ID and timestamps.<br><br>This enables conversation tracking, analytics, and debugging. |
| Daily Summary Schedule | n8n-nodes-base.scheduleTrigger | Starts daily analytics run |  | Get Chat Logs | ## Daily Analytics Trigger<br><br>This scheduled trigger runs every morning to generate a usage summary for the RAG chatbot.<br><br>## Daily RAG Chatbot Analytics<br><br>This workflow generates a daily performance report for a Retrieval-Augmented Generation (RAG) chatbot by analyzing recent chat activity stored in the system.<br><br>Every morning the workflow runs automatically and retrieves chat interactions from the last 24 hours. These logs include user questions, referenced documents, timestamps, and whether relevant documents were successfully retrieved.<br><br>The workflow then analyzes the data to calculate key metrics such as the total number of questions asked, failed document lookups, overall success rate, and the most frequently referenced files in the knowledge base.<br><br>Using these statistics, the workflow generates a formatted HTML summary report. The report highlights usage patterns and helps identify which documents are most useful to users.<br><br>Finally, the report is sent via email to the configured recipient, providing a quick daily overview of chatbot activity and knowledge base performance. |
| Get Chat Logs | n8n-nodes-base.dataTable | Retrieves the last 24 hours of chat logs | Daily Summary Schedule | Analyze Chat Data | ## Chat Log Analysis<br><br>The workflow retrieves chat interactions from the last 24 hours and analyzes usage patterns.<br><br>Metrics include:<br>• total questions asked<br>• files referenced<br>• failed document lookups<br>• overall success rate. |
| Analyze Chat Data | n8n-nodes-base.code | Computes metrics and builds HTML email report | Get Chat Logs | Send Daily Summary Email | ## Chat Log Analysis<br><br>The workflow retrieves chat interactions from the last 24 hours and analyzes usage patterns.<br><br>Metrics include:<br>• total questions asked<br>• files referenced<br>• failed document lookups<br>• overall success rate.<br><br>## Daily Summary Report<br><br>The workflow generates a formatted HTML report and sends it via email. |
| Send Daily Summary Email | n8n-nodes-base.gmail | Emails daily chatbot usage report | Analyze Chat Data |  | ## Daily Summary Report<br><br>The workflow generates a formatted HTML report and sends it via email. |
| Pinecone Vector Store | @n8n/n8n-nodes-langchain.vectorStorePinecone | Exposes Pinecone retrieval as an AI tool | OpenAI Embeddings | RAG Chatbot Agent | ## RAG Question Answering<br><br>The chatbot retrieves relevant document chunks from Pinecone and uses them as context for the LLM.<br><br>The assistant answers using only retrieved information from the uploaded documents. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for vector indexing |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for document processing |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for configuration |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for upload interface |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual documentation for RAG block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual documentation for chat interface |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual documentation for chat logging |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual documentation for analytics |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual documentation for daily trigger |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual documentation for daily report |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual documentation for overall analytics description |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create the document upload trigger**
   - Add a **Form Trigger** node.
   - Name it **Document Upload Form**.
   - Set:
     - Form title: `Document Upload for RAG`
     - Description: `Upload PDF, CSV, or JSON files to add them to the knowledge base`
     - Button label: `Upload Documents`
     - Disable attribution if desired
   - Add one form field:
     - Type: `File`
     - Label: `Upload Documents`
     - Accepted file types: `application/pdf,.csv,application/json`

2. **Add a configuration node**
   - Add a **Set** node after the form trigger.
   - Name it **Workflow Configuration**.
   - Enable keeping other incoming fields.
   - Add fields:
     - `pineconeIndex` → string → your Pinecone index name
     - `pineconeNamespace` → string → `documents`
     - `chunkSize` → number → `1000`
     - `chunkOverlap` → number → `200`
     - `topK` → number → `5`
   - Connect **Document Upload Form → Workflow Configuration**.

3. **Store upload form submissions**
   - Add a **Data Table** node.
   - Name it **Store Form Responses**.
   - Set Data Table to `form_responses`.
   - Use automatic input mapping.
   - Connect **Workflow Configuration → Store Form Responses**.
   - Create the `form_responses` table in n8n first if it does not exist.

4. **Create the text splitter**
   - Add a **Recursive Character Text Splitter** node.
   - Name it **Text Splitter**.
   - Set:
     - Chunk size: `={{ $('Workflow Configuration').first().json.chunkSize }}`
     - Chunk overlap: `={{ $('Workflow Configuration').first().json.chunkOverlap }}`

5. **Create the document loader**
   - Add a **Default Data Loader** LangChain node.
   - Name it **Document Loader**.
   - Configure:
     - Loader: `PDF Loader`
     - Data type: `Binary`
     - Binary mode: `Specific Field`
     - Text splitting mode: `Custom`
   - Attach **Text Splitter** to **Document Loader** using the AI text splitter connection.
   - Important: because the workflow advertises CSV and JSON uploads, either:
     - change the loader to support those formats too, or
     - add format-specific branches with dedicated loaders.
   - In the provided workflow, only PDF loading is actually configured.

6. **Create embeddings node**
   - Add an **OpenAI Embeddings** node.
   - Name it **OpenAI Embeddings**.
   - Configure OpenAI credentials.
   - Keep default embedding settings unless you need a specific model.

7. **Create Pinecone insert node**
   - Add a **Pinecone Vector Store** node.
   - Name it **Pinecone Insert Documents**.
   - Set mode to **Insert**.
   - Configure:
     - Pinecone credentials
     - Index: `={{ $('Workflow Configuration').first().json.pineconeIndex }}`
     - Namespace: `={{ $('Workflow Configuration').first().json.pineconeNamespace }}`
   - Connect:
     - **Workflow Configuration → Pinecone Insert Documents** as main input
     - **Document Loader → Pinecone Insert Documents** as AI document input
     - **OpenAI Embeddings → Pinecone Insert Documents** as AI embedding input
   - Ensure the Pinecone index dimension matches the selected OpenAI embedding model.

8. **Create the chat entry point**
   - Add a **Chat Trigger** node.
   - Name it **Chat Trigger**.
   - Configure:
     - Public: enabled
     - Load previous session: `memory`
     - Initial message: `Hi! I can answer questions about the documents you've uploaded. What would you like to know?`

9. **Create chat memory**
   - Add a **Memory Buffer Window** node.
   - Name it **Chat Memory**.
   - Set context window length to `10`.
   - Connect it to the chat flow as AI memory for the agent.

10. **Create chat model**
    - Add an **OpenAI Chat Model** node.
    - Name it **OpenAI Chat Model**.
    - Select model `gpt-4o-mini`.
    - Configure OpenAI credentials.

11. **Create retrieval tool**
    - Add another **Pinecone Vector Store** node.
    - Name it **Pinecone Vector Store**.
    - Set mode to **Retrieve as Tool**.
    - Configure Pinecone credentials.
    - Set the same Pinecone index used for ingestion.
    - Ideally also set the same namespace: `documents`.
    - If available in your node version, set retrieval count/top K to `5` or use the value from configuration.
    - Connect **OpenAI Embeddings → Pinecone Vector Store** as AI embedding input.
    - Note: in the provided JSON, this node’s index is not configured, so you must complete it manually.

12. **Create the RAG agent**
    - Add an **AI Agent** node.
    - Name it **RAG Chatbot Agent**.
    - Set prompt input text to:
      - `={{ $json.chatInput }}`
    - Use a defined prompt/system instruction:
      - Explain that the assistant must answer only from retrieved documents
      - If information is insufficient, return: `I couldn't find that in the uploaded documents.`
      - Ask it to cite document names when possible
      - Instruct it never to invent information
   - Connect:
     - **Chat Trigger → RAG Chatbot Agent** main
     - **Chat Memory → RAG Chatbot Agent** AI memory
     - **OpenAI Chat Model → RAG Chatbot Agent** AI language model
     - **Pinecone Vector Store → RAG Chatbot Agent** AI tool

13. **Create chat log formatter**
   - Add a **Set** node.
   - Name it **Prepare Chat Log**.
   - Add fields:
     - `userQuery` → `={{ $json.chatInput }}`
     - `botResponse` → `={{ $json.output }}`
     - `timestamp` → `={{ $now.toISO() }}`
     - `sessionId` → `={{ $json.sessionId }}`
     - `matchedFiles` → `={{ $json.metadata?.fileName || 'N/A' }}`
   - Connect **RAG Chatbot Agent → Prepare Chat Log**.

14. **Store chat logs**
   - Add a **Data Table** node.
   - Name it **Log Chat Interactions**.
   - Set Data Table to `chat_logs`.
   - Use auto-mapping.
   - Connect **Prepare Chat Log → Log Chat Interactions**.
   - Create the `chat_logs` table in advance if needed.

15. **Create daily analytics trigger**
   - Add a **Schedule Trigger** node.
   - Name it **Daily Summary Schedule**.
   - Configure it to run daily at **09:00**.
   - Confirm server/project timezone so the hour matches expectation.

16. **Retrieve last 24 hours of logs**
   - Add a **Data Table** node.
   - Name it **Get Chat Logs**.
   - Set:
     - Operation: `Get`
     - Return all: enabled
     - Data table: `chat_logs`
     - Filter: `timestamp` greater than `={{ $now.minus({ days: 1 }).toISO() }}`
   - Connect **Daily Summary Schedule → Get Chat Logs**.

17. **Create analytics code node**
   - Add a **Code** node.
   - Name it **Analyze Chat Data**.
   - Paste logic that:
     - reads all input items
     - counts total questions
     - aggregates referenced files
     - counts failed lookups
     - calculates success rate
     - builds an HTML email body
     - returns summary fields plus `emailBody`
   - Connect **Get Chat Logs → Analyze Chat Data**.
   - Important improvement: align field names with logging.
     - Either log `filesReferenced` instead of `matchedFiles`
     - Or update the code to read `matchedFiles`
     - Also log failure indicators if you want meaningful `failedLookups`

18. **Create email delivery node**
   - Add a **Gmail** node.
   - Name it **Send Daily Summary Email**.
   - Configure Gmail OAuth2 credentials.
   - Set:
     - To: your email address
     - Subject: `Daily RAG Chatbot Summary - {{ $now.toFormat('yyyy-MM-dd') }}`
     - Message/body: `={{ $json.emailBody }}`
   - Connect **Analyze Chat Data → Send Daily Summary Email**.

19. **Add optional sticky notes**
   - Add sticky notes to document:
     - upload interface
     - workflow configuration
     - document processing
     - vector indexing
     - chat interface
     - RAG answering
     - chat logging
     - daily analytics
     - reporting

20. **Configure credentials**
   - **OpenAI credentials**
     - Required for **OpenAI Embeddings**
     - Required for **OpenAI Chat Model**
   - **Pinecone credentials**
     - Required for **Pinecone Insert Documents**
     - Required for **Pinecone Vector Store**
   - **Gmail credentials**
     - Required for **Send Daily Summary Email**

21. **Create or verify required external resources**
   - Pinecone index must exist beforehand.
   - Data tables needed:
     - `form_responses`
     - `chat_logs`

22. **Test the ingestion path**
   - Submit a sample PDF through **Document Upload Form**
   - Verify:
     - form submission stored in `form_responses`
     - document chunks inserted into Pinecone

23. **Test the chat path**
   - Open the chat endpoint
   - Ask a question whose answer exists in the uploaded document
   - Verify:
     - retrieval returns relevant context
     - answer follows the “retrieved-documents-only” rule
     - interaction is written to `chat_logs`

24. **Test the analytics path**
   - Insert or generate sample chat logs
   - Manually execute **Get Chat Logs → Analyze Chat Data → Send Daily Summary Email**
   - Confirm email formatting and metrics

25. **Recommended fixes before production**
   - Configure the retrieval Pinecone node with the same index and namespace as ingestion
   - Use file loaders consistent with allowed upload formats
   - Align logging field names with analytics code
   - Optionally store document IDs, file names, retrieval count, and no-result flags explicitly

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The upload form claims support for PDF, CSV, and JSON, but the configured document loader is PDF-only. | Review the ingestion block before relying on non-PDF uploads. |
| The retrieval node does not currently use the configured `topK` value and has no populated Pinecone index in the provided JSON. | Complete the Pinecone retrieval configuration manually. |
| Analytics fields are inconsistent: logging writes `matchedFiles`, while analysis expects `filesReferenced`; failure metrics are also based on fields never logged. | Update either the logging node or the code node for accurate reporting. |
| This workflow has multiple entry points. | Entry points: **Document Upload Form**, **Chat Trigger**, **Daily Summary Schedule**. |
| No sub-workflows are invoked in this workflow. | N/A |