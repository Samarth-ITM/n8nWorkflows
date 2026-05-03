Build RAG-Powered WhatsApp Chatbots for Docs with GPT-4o-mini and MongoDB

https://n8nworkflows.xyz/workflows/build-rag-powered-whatsapp-chatbots-for-docs-with-gpt-4o-mini-and-mongodb-6315


# Build RAG-Powered WhatsApp Chatbots for Docs with GPT-4o-mini and MongoDB

---

### 1. Workflow Overview

This n8n workflow, titled **"Build RAG-Powered WhatsApp Chatbots for Docs with GPT-4o-mini and MongoDB"**, constructs a sophisticated AI-powered chatbot interface for WhatsApp. It leverages Retrieval-Augmented Generation (RAG) principles by integrating document ingestion, vector embedding, and semantic search with GPT-4o-mini for context-aware responses. The workflow supports multiple WhatsApp message types (text, audio, images, documents), processes them accordingly, and uses MongoDB Atlas as a vector store for efficient document retrieval.

**Target Use Cases:**

- Enterprises wanting to provide interactive, AI-driven WhatsApp support chatbots.
- Automating document-based question answering using corporate or product documentation.
- Handling diverse media types from WhatsApp messages with AI understanding.
- Maintaining conversational context with memory buffers for richer interactions.

**Logical Blocks:**

- **1.1 Document Ingestion & Indexing**  
  Import and process Google Docs product documentation, chunk text, generate embeddings, and insert into MongoDB vector store for later retrieval.

- **1.2 WhatsApp Message Trigger & Routing**  
  Listen for incoming WhatsApp messages and route processing based on message type (text, audio, image, document).

- **1.3 Media Retrieval & Preprocessing**  
  Download media content (audio, images, documents) from WhatsApp URLs, process file extensions and MIME types to determine specific handling.

- **1.4 Document Parsing & Extraction**  
  Extract content from documents (PDF, XLS, XLSX) and map JSON or text data for embedding or language model processing.

- **1.5 Embedding Generation & Vector Search**  
  Generate embeddings for incoming user input or document chunks and perform vector similarity search in MongoDB to retrieve relevant document context.

- **1.6 AI Agent Processing with GPT-4o-mini**  
  Use LangChain's Knowledge Base Agent with GPT-4o-mini model and conversation memory to generate context-aware answers based on retrieved documents and user input.

- **1.7 Response Sending**  
  Send generated answers or fallback responses back to the user via WhatsApp.

---

### 2. Block-by-Block Analysis

#### 1.1 Document Ingestion & Indexing

**Overview:**  
This block imports Google Docs product documentation, splits the text into chunks, generates embeddings, and inserts the data into a MongoDB vector store for semantic search.

**Nodes Involved:**  
- When clicking "Execute Workflow"  
- Google Docs Importer  
- Document Chunker  
- Document Section Loader  
- OpenAI Embeddings Generator  
- MongoDB Vector Store Inserter  
- Sticky Note (instructional)

**Node Details:**  
- **When clicking "Execute Workflow"** (Manual Trigger)  
  - Triggers manual execution to initiate document ingestion.  
  - No parameters.  
  - Output: starts Google Docs Importer.  
  - Edge cases: none; manual trigger.

- **Google Docs Importer** (Google Docs node)  
  - Operation: get document content from a specified Google Docs URL.  
  - Configured with a public document URL containing product docs.  
  - Output: raw document content JSON.  
  - Requires Google Docs OAuth credential.  
  - Potential errors: auth failure, invalid URL, rate limits.

- **Document Chunker** (Text Splitter)  
  - Splits imported document text into chunks (3,000 characters with 200 overlap) using markdown split code.  
  - Input: raw document content.  
  - Output: array of text chunks.  
  - Edge cases: large documents may produce many chunks; chunk size tuned for embedding limits.

- **Document Section Loader** (Document Data Loader)  
  - Converts chunked text into document format with metadata (e.g., doc_id).  
  - Input: chunked text array.  
  - Output: document array ready for embedding.

- **OpenAI Embeddings Generator** (Embeddings Node)  
  - Generates vector embeddings for document chunks using OpenAI embeddings API.  
  - Uses OpenAI API credentials.  
  - Edge cases: API rate limits, timeout, embedding size limits.

- **MongoDB Vector Store Inserter** (Vector Store Node)  
  - Inserts embeddings with metadata into MongoDB Atlas collection "n8n-template" and index "data_index".  
  - Mode: insert.  
  - Input: embeddings with metadata.  
  - Requires MongoDB Atlas credentials.  
  - Possible failures: connection issues, indexing errors.

- **Sticky Note**  
  - Content: "Run this workflow manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search."  
  - Provides user instruction on document ingestion.

---

#### 1.2 WhatsApp Message Trigger & Routing

**Overview:**  
This block listens for incoming WhatsApp messages of various types and routes processing based on message type (text, audio, image, document).

**Nodes Involved:**  
- WhatsApp Trigger  
- Route Types (Switch)  
- Sticky Note1 (description)

**Node Details:**  
- **WhatsApp Trigger** (WhatsApp Trigger Node)  
  - Listens for WhatsApp message updates (messages only).  
  - Uses OAuth2 credential for WhatsApp API.  
  - Output: raw WhatsApp message JSON.  
  - Edge cases: webhook misconfiguration, auth failure, message format changes.

- **Route Types** (Switch)  
  - Routes based on `$json.messages[0].type` to outputs: Text, Audio, Image, Document.  
  - Ensures message is processed by correct handler.  
  - Each output connects to media retrieval or direct text processing.

- **Sticky Note1**  
  - Content: "This workflow listens for WhatsApp messages (text, audio, image, documents), converts them into embeddings, searches MongoDB, and uses GPT-4o-mini to provide context-aware answers with conversation memory."  
  - Contextual overview for users.

---

#### 1.3 Media Retrieval & Preprocessing

**Overview:**  
This block retrieves media URLs from WhatsApp messages, downloads media content, and normalizes file extensions/MIME types for downstream processing.

**Nodes Involved:**  
- Gets WhatsApp Voicemail Source URL  
- Download Voicemail  
- OpenAI (Audio Translation)  
- Gets WhatsApp Image Source URL  
- Download Image  
- OpenAI1 (Image Analysis)  
- Gets WhatsApp Document Source URL  
- Download Document  
- Map file extensions  
- Route Document Types (Switch)

**Node Details:**  
- **Gets WhatsApp Voicemail Source URL** (WhatsApp Node)  
  - Retrieves media URL for audio messages using audio media ID.  
  - Requires WhatsApp API credential.  
  - Edge cases: invalid media ID, auth failures.

- **Download Voicemail** (HTTP Request)  
  - Downloads audio from retrieved media URL.  
  - Uses HTTP header auth credential.  
  - Output: binary audio data.  
  - Possible failures: download timeout, auth failure.

- **OpenAI** (OpenAI Node for Audio)  
  - Operation: translate audio to text using OpenAI audio translation.  
  - Input: downloaded audio.  
  - Uses OpenAI API credential.  
  - Edge cases: audio format issues, API limits.

- **Gets WhatsApp Image Source URL** (WhatsApp Node)  
  - Retrieves media URL for image messages using image media ID.  
  - Requires WhatsApp API credential.

- **Download Image** (HTTP Request)  
  - Downloads image from URL.  
  - Uses HTTP header auth credential.

- **OpenAI1** (OpenAI Node for Image)  
  - Analyzes image (base64 input) using GPT-4o-mini for image captioning or understanding.  
  - Uses OpenAI API credential.

- **Gets WhatsApp Document Source URL** (WhatsApp Node)  
  - Retrieves media URL for document messages using document media ID.  
  - Requires WhatsApp API credential.

- **Download Document** (HTTP Request)  
  - Downloads document file from URL.  
  - Uses HTTP header auth credential.

- **Map file extensions** (Code Node)  
  - Normalizes MIME types for downloaded files for consistent routing (e.g., calendar, XML mappings).  
  - Handles missing MIME types by copying from original WhatsApp metadata.  
  - Edge cases: unknown MIME types, missing data.

- **Route Document Types** (Switch)  
  - Routes based on normalized MIME type to handlers for CSV, HTML, Calendar, RTF, TXT, XML, PDF, JSON, XLS, XLSX, or ELSE unsupported types.  
  - Ensures correct parsing and extraction method per document type.

---

#### 1.4 Document Parsing & Extraction

**Overview:**  
Parses and extracts textual content from various document types to prepare them for embedding and knowledge base insertion.

**Nodes Involved:**  
- Extract from PDF  
- Extract from XLS  
- Extract from XLSX  
- Map JSON  
- Map document prompt  
- Map text prompt  
- Map image prompt

**Node Details:**  
- **Extract from PDF** (Extract From File)  
  - Extracts text content from PDF documents.  
  - Input: binary PDF data.  
  - Output: extracted text JSON.

- **Extract from XLS** (Extract From File)  
  - Extracts data from XLS files.  
  - Input: binary XLS data.

- **Extract from XLSX** (Extract From File)  
  - Extracts data from XLSX files.  
  - Input: binary XLSX data.

- **Map JSON** (Set Node)  
  - Converts extraction output to a "text" field.  
  - Prepares text for embedding or prompt construction.

- **Map document prompt** (Set Node)  
  - Constructs prompt text combining parsed text, document captions, and MIME type for Knowledge Base Agent input.

- **Map text prompt** (Set Node)  
  - Extracts WhatsApp text message content into a "text" field for AI processing.

- **Map image prompt** (Set Node)  
  - Constructs prompt with image description and caption for AI agent.

---

#### 1.5 Embedding Generation & Vector Search

**Overview:**  
Generates embeddings from user input or document chunks and performs vector similarity search within MongoDB to retrieve relevant documents.

**Nodes Involved:**  
- Embeddings OpenAI  
- MongoDB Vector Search

**Node Details:**  
- **Embeddings OpenAI** (Embeddings Node)  
  - Generates embeddings for incoming text or document content.  
  - Uses OpenAI API credentials.  
  - Input: text from mapped prompts.

- **MongoDB Vector Search** (Vector Store Node)  
  - Searches MongoDB Atlas vector store using embedding similarity.  
  - Mode: retrieve-as-tool with toolName "productDocs".  
  - Queries the collection "n8n-template" and index "data_index".  
  - Returns relevant document snippets for AI agent context.

---

#### 1.6 AI Agent Processing with GPT-4o-mini

**Overview:**  
Uses LangChain's Knowledge Base Agent with GPT-4o-mini and conversation memory to generate contextual answers based on retrieved documents and user input.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  
- Knowledge Base Agent

**Node Details:**  
- **Simple Memory** (Memory Buffer Window)  
  - Maintains conversation memory keyed by WhatsApp user ID (wa_id).  
  - Session key uses expression referencing WhatsApp Trigger node.  
  - Provides prior conversation context to AI agent.

- **OpenAI Chat Model** (Language Model Node)  
  - Uses GPT-4o-mini model for chat completions.  
  - Inputs user text and retrieved documents context.  
  - Requires OpenAI API credentials.

- **Knowledge Base Agent** (LangChain Agent Node)  
  - Orchestrates the whole AI reasoning: integrates input text, memory, retrieved documents, and language model to produce answers.  
  - Uses "define" prompt type with variable `text` from mapped prompts.  
  - Inputs: text, memory, tools (MongoDB Vector Search).  
  - Outputs: generated chatbot reply text.

---

#### 1.7 Response Sending

**Overview:**  
Sends generated answers or fallback unsupported type messages back to WhatsApp users.

**Nodes Involved:**  
- Send Response  
- Send Unsupported Response

**Node Details:**  
- **Send Response** (WhatsApp node)  
  - Sends text message response to user's WhatsApp phone number.  
  - Uses WhatsApp API credential.  
  - Text body set dynamically to output from Knowledge Base Agent.

- **Send Unsupported Response** (WhatsApp node)  
  - Sends a predefined message "The File type you provided is unsupported."  
  - Triggered when document type is unrecognized or unsupported.  
  - Recipient phone number dynamically from WhatsApp Trigger node.

---

### 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                              | Input Node(s)                                | Output Node(s)                               | Sticky Note                                                                                  |
|--------------------------------|---------------------------------------|----------------------------------------------|---------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow"| Manual Trigger                        | Manual start for document ingestion          | -                                           | Google Docs Importer                         | Run this workflow manually to import and index Google Docs product documentation into MongoDB with vector embeddings for fast search. |
| Google Docs Importer            | Google Docs                           | Import product documentation from Google Docs| When clicking "Execute Workflow"             | MongoDB Vector Store Inserter (via chunker) |                                                                                              |
| Document Chunker               | Text Splitter                        | Split document text into manageable chunks   | Google Docs Importer                         | Document Section Loader                       |                                                                                              |
| Document Section Loader        | Document Data Loader                 | Prepare document chunks with metadata        | Document Chunker                            | OpenAI Embeddings Generator                   |                                                                                              |
| OpenAI Embeddings Generator    | Embeddings Node                     | Generate embeddings for document chunks      | Document Section Loader                     | MongoDB Vector Store Inserter                 |                                                                                              |
| MongoDB Vector Store Inserter  | Vector Store Node                   | Insert embeddings into MongoDB vector store  | OpenAI Embeddings Generator                 | -                                            |                                                                                              |
| WhatsApp Trigger               | WhatsApp Trigger                    | Trigger on incoming WhatsApp messages        | -                                           | Route Types                                  | This workflow listens for WhatsApp messages (text, audio, image, documents), converts them into embeddings, searches MongoDB, and uses GPT-4o-mini to provide context-aware answers with conversation memory. |
| Route Types                   | Switch                             | Route message processing by WhatsApp message type | WhatsApp Trigger                            | Gets WhatsApp Voicemail/Image/Document Source URL, Map Text Prompt |                                                                                              |
| Gets WhatsApp Voicemail Source URL | WhatsApp                            | Retrieve audio media URL from WhatsApp message| Route Types (Audio)                         | Download Voicemail                           |                                                                                              |
| Download Voicemail             | HTTP Request                       | Download audio file from media URL            | Gets WhatsApp Voicemail Source URL          | OpenAI (Audio Translation)                    |                                                                                              |
| OpenAI                       | OpenAI (Audio Translation)          | Transcribe audio to text                      | Download Voicemail                          | Knowledge Base Agent                          |                                                                                              |
| Gets WhatsApp Image Source URL | WhatsApp                            | Retrieve image media URL from WhatsApp message| Route Types (Image)                         | Download Image                              |                                                                                              |
| Download Image                | HTTP Request                       | Download image file from media URL            | Gets WhatsApp Image Source URL               | OpenAI1 (Image Analysis)                       |                                                                                              |
| OpenAI1                      | OpenAI (Image Analysis)              | Analyze image content                          | Download Image                              | Knowledge Base Agent                          |                                                                                              |
| Gets WhatsApp Document Source URL | WhatsApp                            | Retrieve document media URL from WhatsApp message| Route Types (Document)                      | Download Document                           |                                                                                              |
| Download Document             | HTTP Request                       | Download document file from media URL         | Gets WhatsApp Document Source URL            | Map file extensions                          |                                                                                              |
| Map file extensions           | Code                              | Normalize MIME types for documents            | Download Document                           | Route Document Types                         |                                                                                              |
| Route Document Types          | Switch                            | Route documents by MIME type for parsing      | Map file extensions                         | Extract from PDF/XLS/XLSX, Map JSON, Send Unsupported Response |                                                                                              |
| Extract from PDF              | Extract From File                  | Extract text from PDF documents                | Route Document Types (PDF)                   | Map document prompt                          |                                                                                              |
| Extract from XLS              | Extract From File                  | Extract data from XLS files                     | Route Document Types (XLS)                   | Map JSON                                     |                                                                                              |
| Extract from XLSX             | Extract From File                  | Extract data from XLSX files                    | Route Document Types (XLSX)                  | Map JSON                                     |                                                                                              |
| Map JSON                     | Set                               | Map extracted JSON data to text prompt         | Extract from XLS/XLSX                        | Map document prompt                          |                                                                                              |
| Map document prompt          | Set                               | Prepare prompt text for Knowledge Base Agent  | Extract from PDF, Map JSON, Route Document Types | Knowledge Base Agent                        |                                                                                              |
| Map text prompt              | Set                               | Extract WhatsApp text message content          | Route Types (Text)                          | Knowledge Base Agent                          |                                                                                              |
| Map image prompt             | Set                               | Prepare image description prompt                | OpenAI1                                     | Knowledge Base Agent                          |                                                                                              |
| Embeddings OpenAI            | Embeddings Node                   | Generate embeddings for queries and documents | Map text/document prompt                     | MongoDB Vector Search                         |                                                                                              |
| MongoDB Vector Search        | Vector Store Node                 | Search MongoDB vector store with embeddings    | Embeddings OpenAI                           | Knowledge Base Agent                          |                                                                                              |
| Simple Memory                | Memory Buffer Window             | Maintain conversation memory per WhatsApp user| WhatsApp Trigger                            | Knowledge Base Agent (ai_memory input)       |                                                                                              |
| OpenAI Chat Model            | Language Model Node              | Generate chat completions with GPT-4o-mini     | Knowledge Base Agent (ai_languageModel input) | Knowledge Base Agent                          |                                                                                              |
| Knowledge Base Agent         | LangChain Agent Node            | Compose AI responses integrating memory, search, and model | OpenAI Chat Model, MongoDB Vector Search, Simple Memory | Send Response                               |                                                                                              |
| Send Response               | WhatsApp Node                   | Send chatbot reply back to WhatsApp user       | Knowledge Base Agent                         | -                                            |                                                                                              |
| Send Unsupported Response    | WhatsApp Node                   | Send fallback message for unsupported files    | Route Document Types (ELSE)                  | -                                            |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking "Execute Workflow"`  
   - Purpose: Manually start document ingestion.

2. **Create a Google Docs Node**  
   - Name: `Google Docs Importer`  
   - Operation: `Get` document  
   - Parameter: Set document URL to your Google Docs product documentation URL.  
   - Connect: `When clicking "Execute Workflow"` → `Google Docs Importer`  
   - Credentials: Google Docs OAuth2

3. **Create a Text Splitter Node**  
   - Name: `Document Chunker`  
   - Chunk Size: 3000 characters  
   - Chunk Overlap: 200 characters  
   - Split Code: `markdown`  
   - Connect: `Google Docs Importer` → `Document Chunker`

4. **Create Document Data Loader Node**  
   - Name: `Document Section Loader`  
   - Metadata: Add metadata field `doc_id` referencing `documentId` from input JSON.  
   - JSON Data: Use expression `{{$json.content}}`  
   - Connect: `Document Chunker` → `Document Section Loader`

5. **Create OpenAI Embeddings Node**  
   - Name: `OpenAI Embeddings Generator`  
   - Use OpenAI API credentials  
   - Connect: `Document Section Loader` → `OpenAI Embeddings Generator`

6. **Create MongoDB Vector Store Node (Insert Mode)**  
   - Name: `MongoDB Vector Store Inserter`  
   - Mode: `insert`  
   - Mongo Collection: Select or create collection named `n8n-template`  
   - Vector Index Name: `data_index`  
   - Connect: `OpenAI Embeddings Generator` → `MongoDB Vector Store Inserter`  
   - Credentials: MongoDB Atlas connection

7. **Create WhatsApp Trigger Node**  
   - Name: `WhatsApp Trigger`  
   - Listen for message updates only  
   - Credentials: WhatsApp OAuth2 API  
   - This triggers the main chatbot interaction.

8. **Create Switch Node for Message Routing**  
   - Name: `Route Types`  
   - Condition on `$json.messages[0].type` with cases: `text`, `audio`, `image`, `document`  
   - Connect: `WhatsApp Trigger` → `Route Types`

9. **For Text Messages:**  
   - Create Set Node `Map text prompt`  
     - Set field `text` to `{{$json.messages[0].text.body}}`  
   - Connect: `Route Types` (Text output) → `Map text prompt`

10. **For Audio Messages:**  
    - WhatsApp Node `Gets WhatsApp Voicemail Source URL`  
      - Use mediaGetId from audio message ID  
      - Credentials: WhatsApp API  
    - HTTP Request Node `Download Voicemail`  
      - URL from previous node output `url`  
      - Authentication: HTTP Header Auth with appropriate credentials  
    - OpenAI Node (Audio Translation)  
      - Resource: audio  
      - Operation: translate  
      - Credentials: OpenAI API  
    - Connect chain: `Route Types` (Audio) → `Gets WhatsApp Voicemail Source URL` → `Download Voicemail` → OpenAI audio node.

11. **For Image Messages:**  
    - WhatsApp Node `Gets WhatsApp Image Source URL`  
      - Use mediaGetId from image message ID  
    - HTTP Request Node `Download Image`  
      - URL from previous output  
      - Auth as above  
    - OpenAI Node `OpenAI1` (Image Analysis)  
      - Model: GPT-4o-mini  
      - Resource: image  
      - InputType: base64  
      - Operation: analyze  
    - Set Node `Map image prompt`  
      - Compose text prompt with image content and caption  
    - Connect chain: `Route Types` (Image) → `Gets WhatsApp Image Source URL` → `Download Image` → `OpenAI1` → `Map image prompt`

12. **For Document Messages:**  
    - WhatsApp Node `Gets WhatsApp Document Source URL`  
    - HTTP Request Node `Download Document`  
    - Code Node `Map file extensions`  
      - Normalize MIME types for calendar, XML, or missing types  
    - Switch Node `Route Document Types` based on MIME type  
      - Cases for CSV, HTML, Calendar, RTF, TXT, XML, PDF, JSON, XLS, XLSX, ELSE  
    - Connect chain: `Route Types` (Document) → `Gets WhatsApp Document Source URL` → `Download Document` → `Map file extensions` → `Route Document Types`

13. **Document Extraction Nodes:**  
    - For PDF: `Extract from PDF`  
    - For XLS: `Extract from XLS`  
    - For XLSX: `Extract from XLSX`  
    - For JSON/other: `Map JSON`  
    - Set Node `Map document prompt`  
      - Compose prompt with parsed text, captions, and MIME type for AI agent  
    - Connect each extraction output to `Map document prompt`

14. **Embedding Generation & Search:**  
    - Embeddings Node `Embeddings OpenAI`  
      - Generate embeddings from prompts (text or document)  
    - MongoDB Vector Store Node `MongoDB Vector Search`  
      - Mode: retrieve-as-tool  
      - ToolName: productDocs  
      - Connect: `Embeddings OpenAI` → `MongoDB Vector Search`

15. **AI Agent Setup:**  
    - Memory Node `Simple Memory`  
      - Session key: `"memory_{{$json.contacts[0].wa_id}}"` (custom key by WhatsApp user)  
    - OpenAI Chat Model Node  
      - Model: GPT-4o-mini  
      - Credentials: OpenAI API  
    - LangChain Agent Node `Knowledge Base Agent`  
      - Prompt type: define  
      - Input text from mapped prompts  
      - Inputs: AI language model, memory, MongoDB Vector Search as tool  
    - Connect:  
      - `Map text prompt`, `Map image prompt`, `Map document prompt` → `Knowledge Base Agent`  
      - `OpenAI Chat Model` → `Knowledge Base Agent (ai_languageModel)`  
      - `MongoDB Vector Search` → `Knowledge Base Agent (ai_tool)`  
      - `Simple Memory` → `Knowledge Base Agent (ai_memory)`

16. **Send Response:**  
    - WhatsApp Node `Send Response`  
      - Text body: dynamic output from `Knowledge Base Agent`  
      - Recipient: WhatsApp sender phone number from trigger node  
      - Credentials: WhatsApp API  
    - Connect: `Knowledge Base Agent` → `Send Response`

17. **Fallback for Unsupported Document Types:**  
    - WhatsApp Node `Send Unsupported Response`  
      - Text: "The File type you provided is unsupported."  
      - Recipient: WhatsApp sender phone number  
    - Connect: `Route Document Types` (ELSE output) → `Send Unsupported Response`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Search Index Example JSON schema provided in Sticky Note2 describes MongoDB vector store index mappings, including fields `_id`, `text`, `embedding` (knnVector with 1536 dimensions, cosine similarity), `source`, and `doc_id`.                                                                                                                                                                                                                                                                                                                                                                                              | See Sticky Note2 node for JSON structure                                                        |
| The workflow uses LangChain nodes and OpenAI GPT-4o-mini model, balancing powerful language understanding with cost-efficient embedding and chat generation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow AI node configurations                                                                 |
| The WhatsApp integration uses OAuth2 credentials and requires appropriate webhook setup for message updates and media retrieval.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | WhatsApp Trigger and WhatsApp nodes                                                              |
| The workflow supports a variety of document types by MIME type normalization and conditional parsing, enhancing robustness for diverse customer inputs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Route Document Types and Map file extensions nodes                                              |
| Video and blog resources for building RAG-based chatbots with n8n and LangChain are recommended for further understanding (not included explicitly here).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Suggested external learning resources                                                          |

---

**Disclaimer:**  
The content analyzed is fully derived from an automated n8n workflow designed for legal and public data processing. All data handled respects content policies and contains no illegal or protected material.

---