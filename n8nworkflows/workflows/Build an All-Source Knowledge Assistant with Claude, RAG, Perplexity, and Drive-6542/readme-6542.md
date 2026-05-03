Build an All-Source Knowledge Assistant with Claude, RAG, Perplexity, and Drive

https://n8nworkflows.xyz/workflows/build-an-all-source-knowledge-assistant-with-claude--rag--perplexity--and-drive-6542


# Build an All-Source Knowledge Assistant with Claude, RAG, Perplexity, and Drive

---

### 1. Workflow Overview

This workflow implements an advanced All-Source Knowledge Assistant designed for enterprise use, combining Claude (Anthropic), Retrieval-Augmented Generation (RAG), Perplexity AI, and Google Drive integration. Its primary goal is to provide a robust conversational AI assistant that can answer queries by leveraging multiple knowledge sources, including internal vector stores, structured databases, Google Drive documents, and live web information.

The workflow is logically organized into the following functional blocks:

- **1.1 Input Reception:** Captures incoming chat messages or manual triggers to start processing.
- **1.2 AI Agent Orchestration:** Uses a central Knowledge Agent node powered by Claude (Anthropic Chat Model) to orchestrate reasoning, memory recall, and tool invocation.
- **1.3 Memory Management:** Maintains conversation context and history using a Postgres-backed chat memory.
- **1.4 Semantic Search & Knowledge Retrieval:** Embeddings generation, vector-store search via Supabase, and reranking with Cohere for relevant internal knowledge.
- **1.5 Structured Data Access:** Queries structured tabular data stored in Postgres.
- **1.6 Google Drive Document Search & Processing:** Locates files in Google Drive, downloads and processes multi-format documents via a sub-workflow.
- **1.7 File Content Extraction:** Extracts text from PDFs, CSVs, images, audio, or video files using specialized nodes and AI models.
- **1.8 External Web Search:** Uses Perplexity AI to fetch live external information when internal data is insufficient.
- **1.9 Vector Store Building (Setup Phase):** Downloads and loads documents into Supabase vector store for internal knowledge indexing.

These blocks work together to enable a multi-modal, multi-source knowledge assistant that reasons before acting and provides well-structured Markdown responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Triggers the workflow on new chat messages or manual execution.

**Nodes Involved:**  
- When chat message received  
- When clicking ‘Execute workflow’

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point for chat messages from users. It invokes the workflow when a new chat message arrives over the MCP protocol.  
  - Configuration: Uses default webhook ID and options.  
  - Inputs: External chat message events.  
  - Outputs: Passes message to Knowledge Agent.  
  - Edge cases: Missing or malformed chat input; webhook authentication recommended (see Sticky Note3).

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual testing or execution of the workflow.  
  - Configuration: No parameters.  
  - Inputs: User manually triggers.  
  - Outputs: Starts document loading sub-flow.  
  - Edge cases: None significant.

---

#### 2.2 AI Agent Orchestration

**Overview:**  
Central orchestrator node that manages reasoning, tool calls, memory, and response generation using Claude.

**Nodes Involved:**  
- Knowledge Agent  
- Anthropic Chat Model  
- Think

**Node Details:**

- **Knowledge Agent**  
  - Type: LangChain Agent Node  
  - Role: Core AI agent managing the conversation flow, deciding which tools to invoke.  
  - Configuration:  
    - Uses Claude-sonnet-4-20250514 model via Anthropic Chat Model node.  
    - System message configures it as an AI assistant for the company, with instructions on reasoning, tool usage, and response formatting in Markdown.  
    - Tools registered include General knowledge vector store, structured data query, Google Drive search and file reading, and Perplexity for external info.  
  - Inputs: Chat messages from trigger node, results from tools.  
  - Outputs: Final AI response to user.  
  - Edge cases: Model rate limits, malformed inputs, tool invocation errors, context overflow.  
  - Version: 2.1

- **Anthropic Chat Model**  
  - Type: Language Model (LLM)  
  - Role: Provides Claude model responses used by Knowledge Agent.  
  - Configuration: Model set to "claude-sonnet-4-20250514" for advanced capabilities.  
  - Inputs: Prompts from Knowledge Agent.  
  - Outputs: Text responses.  
  - Edge cases: API quota limits, network failures.

- **Think**  
  - Type: Tool Think Node  
  - Role: Forces the AI to outline reasoning or ask clarifying questions before acting, reducing hallucinations.  
  - Configuration: Descriptive prompt instructing internal chain-of-thought processing.  
  - Inputs: User query and extracted data.  
  - Outputs: Reasoning output fed back to Knowledge Agent.  
  - Edge cases: Timeout or empty output.

---

#### 2.3 Memory Management

**Overview:**  
Stores and recalls conversation history to maintain context across turns.

**Nodes Involved:**  
- Postgres Chat Memory

**Node Details:**

- **Postgres Chat Memory**  
  - Type: Memory Node (Postgres-backed)  
  - Role: Persists chat context in PostgreSQL database for continuity.  
  - Configuration: Uses configured Postgres credentials.  
  - Inputs: Conversation turns from Knowledge Agent.  
  - Outputs: Context back to Knowledge Agent for prompt building.  
  - Edge cases: DB connection failures, data consistency issues.

---

#### 2.4 Semantic Search & Knowledge Retrieval

**Overview:**  
Transforms queries into embeddings, reranks results, and retrieves relevant internal knowledge snippets.

**Nodes Involved:**  
- Embeddings OpenAI  
- Reranker Cohere  
- General knowledge

**Node Details:**

- **Embeddings OpenAI**  
  - Type: Embeddings Node  
  - Role: Generates 1536-dimensional vector embeddings for text queries.  
  - Configuration: Uses OpenAI API with specified dimensions.  
  - Inputs: Text from user queries or documents.  
  - Outputs: Embeddings vectors.  
  - Edge cases: API limits, malformed input text.

- **Reranker Cohere**  
  - Type: Reranker Node  
  - Role: Reorders candidate search results to prioritize the most contextually relevant content.  
  - Configuration: Uses Cohere API credentials.  
  - Inputs: Search hits embeddings.  
  - Outputs: Reranked results.  
  - Edge cases: API failure, low confidence scores.

- **General knowledge**  
  - Type: Vector Store (Supabase) Retrieval  
  - Role: Retrieves relevant knowledge snippets from internal company data indexed in Supabase.  
  - Configuration: Queries "danelfin" table with reranking enabled and tool description provided.  
  - Inputs: Embeddings and reranked results.  
  - Outputs: Text passages to Knowledge Agent.  
  - Edge cases: DB connectivity, empty results.

---

#### 2.5 Structured Data Access

**Overview:**  
Provides access to relational structured data in a Postgres database for tabular queries.

**Nodes Involved:**  
- structured data (Postgres Tool)

**Node Details:**

- **structured data**  
  - Type: Postgres Tool Node  
  - Role: Executes SQL queries on a specified Postgres table to answer structured data questions.  
  - Configuration: Uses Postgres credentials, targets a dynamic table name (AI override enabled).  
  - Inputs: Query from Knowledge Agent.  
  - Outputs: Query result to Knowledge Agent for response generation.  
  - Edge cases: SQL injection risk if input unchecked, query errors, DB failures.

---

#### 2.6 Google Drive Document Search & Processing

**Overview:**  
Searches Google Drive for relevant files and downloads them for content extraction.

**Nodes Involved:**  
- search about any doc in google drive (MCP Client Tool)  
- Google Drive MCP Server  
- Search Files from Gdrive  
- Read File From GDrive (sub-workflow trigger)  
- When Executed by Another Workflow (sub-workflow entry)  
- Operation (switch node)  
- Download File1 (Google Drive download)  
- FileType (switch node)

**Node Details:**

- **search about any doc in google drive**  
  - Type: MCP Client Tool  
  - Role: Queries Google Drive documents matching user query.  
  - Configuration: Uses SSE endpoint placeholder, requires proper setup.  
  - Inputs: User query text.  
  - Outputs: File metadata to Google Drive MCP Server.  
  - Edge cases: SSE endpoint misconfiguration, empty search results.

- **Google Drive MCP Server**  
  - Type: MCP Trigger  
  - Role: Receives requests from MCP clients for Drive operations.  
  - Configuration: Webhook path configured.  
  - Inputs: MCP client requests.  
  - Outputs: Passes to sub-workflows for file reading.  
  - Edge cases: Webhook auth needed, network issues.

- **Search Files from Gdrive**  
  - Type: Google Drive Tool  
  - Role: Performs file/folder search in Google Drive with query string.  
  - Configuration: Limits results to 10, searches in "My Drive".  
  - Inputs: Query from MCP client tool.  
  - Outputs: File metadata list.  
  - Edge cases: API quota limits, invalid queries.

- **Read File From GDrive**  
  - Type: Tool Workflow Node  
  - Role: Calls sub-workflow `ReadFile` to download and process files.  
  - Configuration: Requires `fileId`, `folderId`, and `operation` inputs.  
  - Inputs: File identifiers and operation mode.  
  - Outputs: Extracted content.  
  - Edge cases: Parameter misconfiguration.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for the sub-workflow handling file reading.  
  - Configuration: Accepts `operation`, `folderId`, `fileId`.  
  - Inputs: From Read File From GDrive node.  
  - Outputs: Routes to operation switch.  
  - Edge cases: Missing parameters.

- **Operation**  
  - Type: Switch  
  - Role: Determines the requested operation, here specifically `readFile`.  
  - Configuration: Routes only when operation equals "readFile".  
  - Inputs: Operation parameter.  
  - Outputs: Passes to file download node.  
  - Edge cases: Unsupported operations.

- **Download File1**  
  - Type: Google Drive Node  
  - Role: Downloads the target file in its binary form.  
  - Configuration: Uses dynamic `fileId`, converts Google Docs to plain text, Slides to PDF.  
  - Inputs: File ID from operation node.  
  - Outputs: Binary data of the file.  
  - Edge cases: API failures, permission errors.

- **FileType**  
  - Type: Switch  
  - Role: Routes file processing based on MIME type: pdf, csv, image, audio, video.  
  - Configuration: Checks MIME type from binary data.  
  - Inputs: File binary.  
  - Outputs: Routes to appropriate extraction node.  
  - Edge cases: Unsupported MIME types.

---

#### 2.7 File Content Extraction

**Overview:**  
Extracts text or transcription from files of various formats to produce text answers.

**Nodes Involved:**  
- Extract from PDF  
- Extract from CSV  
- Get PDF Response  
- Get CSV Response  
- Analyse Image  
- Transcribe Audio

**Node Details:**

- **Extract from PDF**  
  - Type: Extract From File  
  - Role: Extracts text from PDF binary files.  
  - Inputs: PDF binary from FileType node.  
  - Outputs: Extracted text for setting response.  
  - Edge cases: Encrypted or malformed PDFs.

- **Extract from CSV**  
  - Type: Extract From File  
  - Role: Extracts CSV data into rows and columns with UTF-8 encoding.  
  - Inputs: CSV binary.  
  - Outputs: Row arrays for conversion.  
  - Edge cases: Malformed CSVs, encoding issues.

- **Get PDF Response**  
  - Type: Set Node  
  - Role: Converts extracted PDF text to `response` property.  
  - Inputs: Extracted PDF text.  
  - Outputs: Response string for Knowledge Agent.  
  - Edge cases: Empty text.

- **Get CSV Response**  
  - Type: Set Node  
  - Role: Converts CSV rows into CSV string format as `response`.  
  - Inputs: Extracted CSV rows.  
  - Outputs: Response string.  
  - Edge cases: Empty data.

- **Analyse Image**  
  - Type: OpenAI Node (Image Analysis)  
  - Role: Sends base64 image data to GPT-4o-mini for visual description.  
  - Inputs: Image binary from FileType node.  
  - Outputs: Textual image analysis.  
  - Edge cases: Unsupported image formats, API limits.

- **Transcribe Audio**  
  - Type: OpenAI Node (Audio Transcription)  
  - Role: Uses OpenAI Whisper to transcribe audio or video files.  
  - Inputs: Audio/video binary.  
  - Outputs: Text transcript.  
  - Edge cases: Long audio files, poor audio quality.

---

#### 2.8 External Web Search

**Overview:**  
When internal knowledge is insufficient, queries Perplexity AI for up-to-date external web information.

**Nodes Involved:**  
- Message a model in Perplexity

**Node Details:**

- **Message a model in Perplexity**  
  - Type: Perplexity Tool Node  
  - Role: Sends user query messages to Perplexity AI for live web search answers.  
  - Configuration: Dynamic message content from AI override.  
  - Inputs: Query text.  
  - Outputs: Web-sourced answers.  
  - Edge cases: API failures, rate limits.

---

#### 2.9 Vector Store Building (Setup Phase)

**Overview:**  
Loads documents from Google Drive, splits text, generates embeddings, and inserts into Supabase vector store.

**Nodes Involved:**  
- Download file  
- Default Data Loader1  
- Recursive Character Text Splitter1  
- Embeddings OpenAI1  
- Add to Supabase Vector DB

**Node Details:**

- **Download file**  
  - Type: Google Drive Node  
  - Role: Downloads zip file containing documents for indexing.  
  - Inputs: Manual trigger.  
  - Outputs: Binary data to data loader.  
  - Edge cases: File access issues.

- **Default Data Loader1**  
  - Type: Document Default Data Loader  
  - Role: Loads and prepares documents for processing.  
  - Inputs: Binary data from download.  
  - Outputs: Document text to splitter.  
  - Edge cases: Unsupported file contents.

- **Recursive Character Text Splitter1**  
  - Type: Text Splitter  
  - Role: Splits large documents recursively into manageable chunks.  
  - Inputs: Loaded text.  
  - Outputs: Text chunks.  
  - Edge cases: Over-splitting or loss of context.

- **Embeddings OpenAI1**  
  - Type: Embeddings Node  
  - Role: Converts text chunks into vector embeddings.  
  - Inputs: Text chunks.  
  - Outputs: Embeddings to vector store inserter.  
  - Edge cases: API limits.

- **Add to Supabase Vector DB**  
  - Type: Vector Store Insert  
  - Role: Inserts embeddings into Supabase for later retrieval.  
  - Inputs: Embeddings.  
  - Outputs: Confirmation or errors.  
  - Edge cases: DB connection errors.

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                     | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                                                           |
|-------------------------------|--------------------------------------|-----------------------------------|-----------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received    | LangChain Chat Trigger                | Chat message input trigger        | External webhook                  | Knowledge Agent                   | ### Advanced model of claude or Grok 4 for better results                                                                             |
| Knowledge Agent              | LangChain Agent                      | Core AI orchestrator               | When chat message received, Think, Tools  | Sends response                   | Contains detailed system instructions for AI assistant behavior, reasoning, and tool use                                               |
| Anthropic Chat Model         | Language Model (Anthropic Claude)    | LLM response generation            | Knowledge Agent                   | Knowledge Agent                   |                                                                                                                                      |
| Think                       | Tool Think Node                      | Forces AI chain-of-thought        | Knowledge Agent                   | Knowledge Agent                   |                                                                                                                                      |
| Postgres Chat Memory        | Memory (Postgres)                    | Conversation context persistence   | Knowledge Agent                   | Knowledge Agent                   |                                                                                                                                      |
| Embeddings OpenAI           | Embeddings Node (OpenAI)             | Generates vector embeddings        | Text input                      | Reranker Cohere, General knowledge |                                                                                                                                      |
| Reranker Cohere             | Reranker Node (Cohere)               | Reranks search results             | Embeddings OpenAI                | General knowledge                |                                                                                                                                      |
| General knowledge           | Vector Store Retrieval (Supabase)   | Retrieves internal knowledge       | Reranker Cohere                  | Knowledge Agent                   |                                                                                                                                      |
| structured data             | Postgres Tool                       | Queries structured tabular data    | Knowledge Agent                   | Knowledge Agent                   | It can be google sheets/ airtable ...                                                                                                |
| search about any doc in google drive | MCP Client Tool                  | Google Drive file search           | Knowledge Agent                   | Google Drive MCP Server           |                                                                                                                                      |
| Google Drive MCP Server     | MCP Trigger                        | Receives MCP client requests       | Read File From GDrive            | When Executed by Another Workflow |                                                                                                                                      |
| Search Files from Gdrive    | Google Drive Tool                  | Searches files/folders in Drive    | search about any doc in google drive | Google Drive MCP Server           |                                                                                                                                      |
| Read File From GDrive       | Tool Workflow Node                 | Calls sub-workflow to read files   | Knowledge Agent                   | Google Drive MCP Server           |                                                                                                                                      |
| When Executed by Another Workflow | Execute Workflow Trigger           | Sub-workflow entry point           | Google Drive MCP Server          | Operation                       |                                                                                                                                      |
| Operation                  | Switch                            | Routes operation types             | When Executed by Another Workflow | Download File1                  |                                                                                                                                      |
| Download File1             | Google Drive                      | Downloads file binary              | Operation                       | FileType                       |                                                                                                                                      |
| FileType                   | Switch                            | Routes file processing by MIME type | Download File1                  | Extract from PDF, CSV, etc.      | ## 2. Handle Multiple Binary Formats via Conversion and AI [Read more about the PostgreSQL Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/) MCP clients (or rather, the AI agents) still expect and require text responses from our MCP server. N8N can provide the right conversion tools to parse most text formats such as PDF, CSV and XML. For images, audio and video, consider using multimodal LLMs to describe or transcribe the file instead. |
| Extract from PDF           | Extract From File                 | Extracts text from PDFs             | FileType (pdf output)            | Get PDF Response               |                                                                                                                                      |
| Extract from CSV           | Extract From File                 | Extracts CSV content                | FileType (csv output)            | Get CSV Response               |                                                                                                                                      |
| Get PDF Response           | Set Node                        | Converts extracted PDF text to response | Extract from PDF               | Knowledge Agent               |                                                                                                                                      |
| Get CSV Response           | Set Node                        | Converts extracted CSV rows to response | Extract from CSV               | Knowledge Agent               |                                                                                                                                      |
| Analyse Image              | OpenAI (Image Analysis)          | Generates textual image description | FileType (image output)          | Knowledge Agent               |                                                                                                                                      |
| Transcribe Audio           | OpenAI (Audio Transcription)     | Transcribes audio/video to text    | FileType (audio/video output)   | Knowledge Agent               |                                                                                                                                      |
| Message a model in Perplexity | Perplexity Tool                 | Fetches live external web info     | Knowledge Agent                   | Knowledge Agent               | ### Search for live data in the Web                                                                                                  |
| Download file              | Google Drive                   | Downloads documents for indexing   | When clicking ‘Execute workflow’ | Default Data Loader1          | ## Load data to vector store                                                                                                          |
| Default Data Loader1       | Document Default Data Loader     | Loads document binaries            | Download file                   | Recursive Character Text Splitter1 |                                                                                                                                      |
| Recursive Character Text Splitter1 | Text Splitter               | Splits documents into text chunks  | Default Data Loader1            | Embeddings OpenAI1            |                                                                                                                                      |
| Embeddings OpenAI1         | Embeddings Node (OpenAI)         | Converts text chunks to embeddings | Recursive Character Text Splitter1 | Add to Supabase Vector DB   |                                                                                                                                      |
| Add to Supabase Vector DB  | Vector Store Insert (Supabase)   | Inserts embeddings into vector DB  | Embeddings OpenAI1              |                               |                                                                                                                                      |
| Sticky Notes (various)    | n8n Sticky Note                  | Inline documentation and instructions | Various                        |                               | See detailed notes in nodes and summary table                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: LangChain Chat Trigger  
   - Configure webhook ID and default options.  
   - Purpose: Entry point for user chat messages.

2. **Create "Knowledge Agent" node**  
   - Type: LangChain Agent  
   - Configure system message to instruct AI assistant behavior, referencing your company.  
   - Register tools: Anthropic Chat Model, Postgres Chat Memory, Think tool, General knowledge vector store, structured data Postgres tool, Google Drive search and file reading tools, Perplexity.  
   - Connect input from "When chat message received".  
   - Connect outputs to Anthropic Chat Model and tools.

3. **Create "Anthropic Chat Model" node**  
   - Type: Language Model (Anthropic)  
   - Select model "claude-sonnet-4-20250514".  
   - Connect input from Knowledge Agent LLM input.  
   - Connect output back to Knowledge Agent.

4. **Create "Think" node**  
   - Type: Tool Think Node  
   - Add descriptive prompt: "Use the tool to think about the user query and the actual data extracted."  
   - Connect input from Knowledge Agent tool input.  
   - Connect output back to Knowledge Agent.

5. **Create "Postgres Chat Memory" node**  
   - Type: LangChain Postgres Chat Memory  
   - Configure with Postgres credentials.  
   - Connect input/output to/from Knowledge Agent memory.

6. **Create Semantic Search pipeline:**  
   - Add "Embeddings OpenAI" node with 1536 dimensions; connect input from Knowledge Agent embeddings input.  
   - Connect output embeddings to "Reranker Cohere" node (configure with Cohere credentials).  
   - Connect reranked results to "General knowledge" node (Supabase vector store retrieval).  
   - Configure "General knowledge" table name as "danelfin" and enable reranking.  
   - Connect output to Knowledge Agent tool input.

7. **Create "structured data" node (Postgres Tool):**  
   - Configure with Postgres credentials.  
   - Set table name dynamically with AI override enabled.  
   - Connect tool input/output to Knowledge Agent.

8. **Set up Google Drive integration:**  
   - Create "search about any doc in google drive" MCP Client Tool; configure SSE endpoint placeholder.  
   - Connect output to "Google Drive MCP Server" (MCP Trigger) with webhook path configured.  
   - Create "Search Files from Gdrive" node (Google Drive Tool), configure to search "My Drive" with query from MCP Client Tool.  
   - Connect to MCP Server.  
   - Create "Read File From GDrive" Tool Workflow node; configure with parameters `fileId`, `folderId`, `operation`.  
   - Connect output to MCP Server.  
   - Create sub-workflow with "When Executed by Another Workflow" trigger accepting `operation`, `fileId`, `folderId`.  
   - Add "Operation" switch node; route only if `operation` = "readFile".  
   - Connect to "Download File1" Google Drive node; configure to download files with Google Docs converted to text/plain and Slides to PDF.  
   - Connect to "FileType" switch node; route processing by MIME type (`application/pdf`, `text/csv`, image formats, audio, video).  
   - For each MIME type, connect to corresponding extraction node: "Extract from PDF", "Extract from CSV", "Analyse Image", "Transcribe Audio".  
   - Connect extraction outputs to "Get PDF Response" or "Get CSV Response" set nodes that assign extracted content to `response`.  
   - Return `response` to Knowledge Agent.

9. **Create "Message a model in Perplexity" node:**  
   - Configure with Perplexity API credentials.  
   - Connect tool input/output to Knowledge Agent.

10. **Setup vector store building (optional / setup phase):**  
    - Add manual trigger node "When clicking ‘Execute workflow’".  
    - Connect to "Download file" node (Google Drive) with file ID pointing to zipped documents.  
    - Connect to "Default Data Loader1" (Document loader for binary).  
    - Connect to "Recursive Character Text Splitter1" (Text splitter).  
    - Connect to "Embeddings OpenAI1" node (no dimension override).  
    - Connect to "Add to Supabase Vector DB" node to insert embeddings.  
    - Configure all nodes with appropriate credentials.

11. **Add Sticky Notes throughout the workflow** as inline documentation and guidance.

12. **Configure credentials:**  
    - Anthropic API account  
    - OpenAI API account  
    - Cohere API account  
    - Perplexity API account  
    - Postgres database credentials  
    - Supabase credentials  
    - Google Drive OAuth2 credentials  

13. **Test workflow end-to-end:**  
    - Send chat messages to "When chat message received" webhook.  
    - Verify Knowledge Agent responses.  
    - Test file search and reading from Google Drive.  
    - Validate structured data querying.  
    - Confirm external Perplexity web search fallback.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                            | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Always Authenticate Your Server! Before going to production, it's always advised to enable authentication on your MCP server trigger.                                                                                                                                  | Sticky Note3                                                                                     |
| It can be google sheets/ airtable ...                                                                                                                                                                                                                                  | Sticky Note2                                                                                     |
| https://n8n.io/creators/jimleuk/ (Jimleuk build this) - https://n8n.io/workflows/3634-build-your-own-google-drive-mcp-server/ (click the link for more detailed explanation)                                                                                            | Sticky Note5                                                                                     |
| Advanced model of claude or Grok 4 for better results                                                                                                                                                                                                                   | Sticky Note6                                                                                     |
| 2. Handle Multiple Binary Formats via Conversion and AI [Read more about the PostgreSQL Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/) MCP clients (or rather, the AI agents) still expect and require text responses from our MCP server. | Sticky Note1                                                                                     |
| Search for live data in the Web                                                                                                                                                                                                                                        | Sticky Note4                                                                                     |
| Detailed workflow description and architecture, including main flow, memory, semantic retrieval, file processing, external search, design highlights, and system benefits.                                                                                              | Sticky Note7 (extensive multi-page content)                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow, respecting applicable content policies and containing no illegal or protected elements. All handled data is legal and public.

---