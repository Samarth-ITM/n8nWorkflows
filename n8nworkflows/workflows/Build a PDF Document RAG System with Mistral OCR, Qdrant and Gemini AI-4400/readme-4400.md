Build a PDF Document RAG System with Mistral OCR, Qdrant and Gemini AI

https://n8nworkflows.xyz/workflows/build-a-pdf-document-rag-system-with-mistral-ocr--qdrant-and-gemini-ai-4400


# Build a PDF Document RAG System with Mistral OCR, Qdrant and Gemini AI

### 1. Workflow Overview

This workflow implements a comprehensive Retrieval-Augmented Generation (RAG) system centered on PDF documents. It leverages Mistral’s OCR API to extract text from PDFs, stores vectorized document embeddings in a Qdrant vector database, and uses Google Gemini AI for conversational querying and summarization. The system targets use cases such as document search, Q&A from PDF content, and automated summarization in Italian. The logical flow is structured into four main blocks:

- **1.1 Initialization and Qdrant Collection Setup:** Prepares the Qdrant vector store by creating or refreshing collections.
- **1.2 PDF Ingestion and OCR Processing:** Downloads PDFs from Google Drive, uploads them to Mistral API, and obtains OCR text.
- **1.3 Document Vectorization and Storage:** Processes extracted text into chunks, generates embeddings with OpenAI, and inserts vectors into Qdrant.
- **1.4 RAG Query Handling:** Listens for chat queries, retrieves relevant vectors from Qdrant, and generates AI responses using Google Gemini Chat models.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Qdrant Collection Setup

- **Overview:**  
  This block ensures the Qdrant vector database is ready by creating a collection (if needed) and clearing existing points to start fresh.

- **Nodes Involved:**  
  - Create collection  
  - Refresh collection  
  - Search PDFs  
  - Loop Over Items1  
  - Edit Fields1  
  - Execute Workflow  
  - When clicking ‘Test workflow’  
  - Sticky Note3  
  - Sticky Note4  

- **Node Details:**

  - **Create collection**  
    - Type: HTTP Request  
    - Role: Creates a new Qdrant collection with specified vector size (1536) and cosine similarity distance.  
    - Config: HTTP PUT to `http://QDRANTURL/collections/COLLECTION` with JSON body defining vector params and shard settings.  
    - Inputs: Manual Trigger ("Test workflow") triggers refresh and collection creation.  
    - Outputs: None explicitly; initiates workflow.  
    - Failure Modes: HTTP errors, authentication failure, incorrect URL or collection name.  
    - Sticky Note3 explains the need to configure QDRANTURL and COLLECTION.

  - **Refresh collection**  
    - Type: HTTP Request  
    - Role: Deletes all existing points in the Qdrant collection to clear old data before ingestion.  
    - Config: HTTP POST to `http://QDRANTURL/collections/COLLECTION/points/delete` with empty filter `{}`.  
    - Inputs: Triggered by manual trigger node.  
    - Outputs: On success, triggers "Search PDFs" to start ingestion.  
    - Failure Modes: Network issues, invalid credentials, wrong collection name.

  - **Search PDFs**  
    - Type: Google Drive node  
    - Role: Lists PDF files from a specific Google Drive folder (`folderId = 1LWVo3yn_1bWQJsLskBIbWTGwlfObvtUK`) for processing.  
    - Inputs: Triggered after collection cleanup.  
    - Outputs: Passes files to "Loop Over Items1" for batch processing.  
    - Failure Modes: OAuth token expiry, folder permission errors.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Iterates through each PDF file from Google Drive one by one for sequential processing.  
    - Inputs: Files from Google Drive node.  
    - Outputs: Each file triggers "Edit Fields1" node.  
    - Failure Modes: Large file list causing long batch processing time.

  - **Edit Fields1**  
    - Type: Set  
    - Role: Adds a new field `file_id` with the Google Drive file ID, preparing input for sub-workflow execution.  
    - Inputs: Each file batch item.  
    - Outputs: Passes to "Execute Workflow" node.  
    - Failure Modes: Missing or malformed file ID.

  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Runs a sub-workflow (ID: AdVUaHTE9Jk1KO72) that performs Mistral OCR on each PDF.  
    - Config: Runs in "each" mode, waits for sub-workflow completion before proceeding.  
    - Inputs: Receives `file_id` from Edit Fields1.  
    - Outputs: Returns OCR-processed data for further batching.  
    - Failure Modes: Sub-workflow errors, timeouts.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual testing; starts collection refresh.  
    - Outputs: Triggers "Refresh collection".

  - **Sticky Notes**  
    - Sticky Note3: Instructions on setting QDRANTURL and COLLECTION for collection creation.  
    - Sticky Note4: Notes about vectorization and Google Drive folder customization.

---

#### 2.2 PDF Ingestion and OCR Processing

- **Overview:**  
  Downloads each PDF by ID, uploads it to Mistral for OCR processing, and retrieves OCR text and metadata.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Get PDF  
  - Mistral Upload  
  - Mistral Signed URL  
  - Mistral DOC OCR  
  - Code  
  - Loop Over Items  

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point triggered by parent workflow passing `file_id`.  
    - Outputs: Triggers "Get PDF" node.

  - **Get PDF**  
    - Type: Google Drive  
    - Role: Downloads the PDF file binary from Google Drive using `file_id`.  
    - Inputs: `file_id` from trigger node.  
    - Outputs: Binary data passed to "Mistral Upload".  
    - Failure Modes: File missing, permission denied, download failure.

  - **Mistral Upload**  
    - Type: HTTP Request  
    - Role: Uploads PDF binary to Mistral API for OCR processing with "purpose" set to "ocr".  
    - Config: POST multipart/form-data to `https://api.mistral.ai/v1/files` with file binary in `data` field.  
    - Authentication: Uses predefined "Mistral Cloud account".  
    - Outputs: Returns file ID from Mistral for signed URL generation.  
    - Failure Modes: Upload failure, API auth error, file size limits.

  - **Mistral Signed URL**  
    - Type: HTTP Request  
    - Role: Retrieves a signed URL valid for 24 hours to access the uploaded PDF on Mistral side.  
    - Config: GET `https://api.mistral.ai/v1/files/{{ $json.id }}/url` with expiry query param.  
    - Outputs: Passes URL to OCR node.  
    - Failure Modes: URL retrieval failure, expired token.

  - **Mistral DOC OCR**  
    - Type: HTTP Request  
    - Role: Requests OCR on the document at the signed URL using Mistral’s latest OCR model.  
    - Config: POST JSON body specifying model and document URL.  
    - Outputs: Receives OCR result including extracted text and page structure.  
    - Failure Modes: OCR model errors, API timeout, malformed JSON response.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Extracts the markdown text from OCR pages, mapping each page to JSON with `markdown` property.  
    - Inputs: OCR JSON output from previous node.  
    - Outputs: List of page markdown strings for batch processing.  
    - Failure Modes: Unexpected OCR response format.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each page’s markdown text to process them individually downstream.  
    - Outputs: Two branches: one goes to "Set page", another to "Wait".

---

#### 2.3 Document Vectorization and Storage

- **Overview:**  
  Converts OCR text pages into vector embeddings, splits text into chunks, and inserts vectors into the Qdrant collection.

- **Nodes Involved:**  
  - Set page  
  - Qdrant Vector Store  
  - Wait  
  - Token Splitter  
  - Default Data Loader  
  - Embeddings OpenAI  
  - Summarization Chain  
  - Set summary  
  - Google Gemini Chat Model1  
  - Qdrant Vector Store1  
  - Embeddings OpenAI1  
  - Vector Store Retriever  

- **Node Details:**

  - **Set page**  
    - Type: Set  
    - Role: Sets the `text` field to the current page's markdown text.  
    - Outputs: Feeds into "Qdrant Vector Store" for vector insertion.  
    - Failure Modes: Missing markdown content.

  - **Qdrant Vector Store**  
    - Type: LangChain Vector Store Qdrant node  
    - Role: Inserts vector embeddings into the `ocr_mistral_test` Qdrant collection in insert mode.  
    - Credentials: Uses Qdrant API account (Hetzner).  
    - Inputs: Text data to embed and store.  
    - Outputs: Triggers "Wait" node to manage pacing.  
    - Failure Modes: API errors, indexing failures.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow to avoid rate limits or overload.  
    - Outputs: Returns to "Loop Over Items" to continue batch processing.  
    - Failure Modes: Delay misconfiguration.

  - **Token Splitter**  
    - Type: Text Splitter (Token-based)  
    - Role: Splits large text documents into chunks of 400 tokens with 40 token overlap for embedding.  
    - Outputs: Feeds chunks to "Default Data Loader".  
    - Failure Modes: Incorrect chunk size causing data loss or overlap issues.

  - **Default Data Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Prepares text chunks into documents compatible with LangChain embedding workflows.  
    - Outputs: Feeds into "Embeddings OpenAI".  
    - Failure Modes: Data format issues.

  - **Embeddings OpenAI**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Computes vector embeddings using OpenAI API for document chunks.  
    - Credentials: OpenAI API key required.  
    - Outputs: Sends embeddings to "Qdrant Vector Store".  
    - Failure Modes: API rate limits, auth errors.

  - **Summarization Chain**  
    - Type: LangChain Chain Summarization  
    - Role: Optionally summarizes text content in Italian using prompt templates.  
    - Outputs: Sends summary text to "Set summary".  
    - Failure Modes: Model errors or empty input.

  - **Set summary**  
    - Type: Set  
    - Role: Stores summarized text in `text` field for further use.  
    - Outputs: Can be connected to vector store or chat model for lighter RAG.

  - **Google Gemini Chat Model1**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Uses Google Gemini 2.0 Flash model for improved summarization or chat tasks.  
    - Credentials: Google PaLM API account.  
    - Failure Modes: API quota, model availability.

  - **Qdrant Vector Store1**  
    - Type: LangChain Vector Store Qdrant  
    - Role: Used in retrieval to get relevant documents from `ocr_mistral_test`.  
    - Credentials: Same Qdrant API account.  
    - Outputs: Provides vector store to "Vector Store Retriever".

  - **Embeddings OpenAI1**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Similar to Embeddings OpenAI but used in retrieval workflows.  
    - Credentials: OpenAI account.  
    - Outputs: Feeds embeddings to "Qdrant Vector Store1".

  - **Vector Store Retriever**  
    - Type: LangChain Retriever Vector Store  
    - Role: Retrieves relevant document chunks from Qdrant for query answering.  
    - Inputs: Vector store and user query.  
    - Outputs: Passes retrieval results to QA chain.

---

#### 2.4 RAG Query Handling

- **Overview:**  
  Listens for chat messages, runs retrieval-augmented QA chain with Google Gemini chat model, producing answers based on stored document vectors.

- **Nodes Involved:**  
  - When chat message received  
  - Question and Answer Chain  
  - Google Gemini Chat Model  
  - Vector Store Retriever  
  - Sticky Note1  
  - Sticky Note  

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Webhook trigger for incoming chat messages to start RAG query flow.  
    - Outputs: Passes user message to "Question and Answer Chain".  
    - Failure Modes: Webhook connectivity, message parsing.

  - **Question and Answer Chain**  
    - Type: LangChain Chain Retrieval QA  
    - Role: Combines retrieved documents with language model to generate answers.  
    - Inputs: User query and retrieved relevant documents.  
    - Outputs: Answers passed back to chat interface.  
    - Failure Modes: Retrieval failure, model errors.

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Uses Gemini 1.5 Flash model to generate conversational answers.  
    - Credentials: Google PaLM API.  
    - Failure Modes: API limits, model errors.

  - **Vector Store Retriever**  
    - Role: Retrieves vectors matching the query from Qdrant (as described above).

  - **Sticky Notes:**  
    - Sticky Note1: Marks this as STEP 4 for RAG testing.  
    - Sticky Note (near summarization): Notes that replacing "Set page" with "Summarization Chain" yields a lighter, faster RAG using main content summaries.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                   |
|----------------------------|-----------------------------------|----------------------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                    | Manual start for collection refresh    | -                             | Refresh collection             |                                                                                                                              |
| Refresh collection          | HTTP Request                      | Deletes all points in Qdrant collection| When clicking ‘Test workflow’  | Search PDFs                   |                                                                                                                              |
| Search PDFs                | Google Drive                      | Lists PDFs in Google Drive folder      | Refresh collection             | Loop Over Items1               |                                                                                                                              |
| Loop Over Items1           | SplitInBatches                   | Batches PDF files for processing       | Search PDFs                   | Edit Fields1                  |                                                                                                                              |
| Edit Fields1               | Set                              | Adds `file_id` to data                  | Loop Over Items1              | Execute Workflow              |                                                                                                                              |
| Execute Workflow           | Execute Workflow                  | Runs OCR sub-workflow per file          | Edit Fields1                  | Loop Over Items1              |                                                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger      | Entry point for sub-workflow (OCR)     | -                             | Get PDF                      |                                                                                                                              |
| Get PDF                   | Google Drive                      | Downloads file binary by file_id        | When Executed by Another Workflow | Mistral Upload               |                                                                                                                              |
| Mistral Upload             | HTTP Request                     | Uploads PDF to Mistral for OCR          | Get PDF                      | Mistral Signed URL            |                                                                                                                              |
| Mistral Signed URL         | HTTP Request                     | Obtains signed URL for uploaded PDF     | Mistral Upload               | Mistral DOC OCR              |                                                                                                                              |
| Mistral DOC OCR            | HTTP Request                     | Performs OCR on document via Mistral   | Mistral Signed URL           | Code                         |                                                                                                                              |
| Code                      | Code                             | Extracts markdown text from OCR pages   | Mistral DOC OCR              | Loop Over Items               |                                                                                                                              |
| Loop Over Items            | SplitInBatches                   | Iterates over OCR pages                  | Code                         | Set page, Wait                |                                                                                                                              |
| Set page                  | Set                              | Sets page text for vector insertion     | Loop Over Items              | Qdrant Vector Store           | Sticky Note: Light/faster RAG option by replacing with "Summarization Chain"                                                  |
| Qdrant Vector Store        | LangChain Vector Store Qdrant    | Inserts embeddings into Qdrant           | Set page                     | Wait                         |                                                                                                                              |
| Wait                      | Wait                             | Delays to manage processing pace         | Qdrant Vector Store          | Loop Over Items               |                                                                                                                              |
| Token Splitter            | LangChain Text Splitter Token    | Splits text into chunks for embeddings   | -                           | Default Data Loader           |                                                                                                                              |
| Default Data Loader       | LangChain Document Loader        | Prepares documents for embedding         | Token Splitter               | Embeddings OpenAI             |                                                                                                                              |
| Embeddings OpenAI         | LangChain Embeddings OpenAI      | Generates vector embeddings               | Default Data Loader          | Qdrant Vector Store           |                                                                                                                              |
| Summarization Chain       | LangChain Chain Summarization    | Summarizes text in Italian                | Google Gemini Chat Model1    | Set summary                  | Sticky Note: Step 3 summarization optional workflow block                                                                   |
| Set summary               | Set                              | Stores summarized text                     | Summarization Chain          | -                            |                                                                                                                              |
| Google Gemini Chat Model1 | LangChain LM Chat Google Gemini  | Summarization and chat model (Gemini 2) | Summarization Chain          | Set summary                  |                                                                                                                              |
| Qdrant Vector Store1      | LangChain Vector Store Qdrant    | Vector storage for retrieval              | Embeddings OpenAI1           | Vector Store Retriever        |                                                                                                                              |
| Embeddings OpenAI1        | LangChain Embeddings OpenAI      | Embeddings for retrieval                   | -                           | Qdrant Vector Store1          |                                                                                                                              |
| Vector Store Retriever    | LangChain Retriever Vector Store | Retrieves relevant documents for QA       | Qdrant Vector Store1         | Question and Answer Chain     |                                                                                                                              |
| When chat message received | LangChain Chat Trigger           | Webhook for incoming chat queries         | -                           | Question and Answer Chain     | Sticky Note1: Step 4 - Test the RAG                                                                                          |
| Question and Answer Chain | LangChain Chain Retrieval QA     | Executes RAG with documents and LLM       | When chat message received, Vector Store Retriever | Google Gemini Chat Model |                                                                                                                              |
| Google Gemini Chat Model  | LangChain LM Chat Google Gemini  | Language model generates answers          | Question and Answer Chain    | -                            |                                                                                                                              |
| Create collection         | HTTP Request                    | Creates Qdrant collection (initial setup) | -                           | -                            | Sticky Note3: Step 1 - Create Qdrant collection; configure QDRANTURL and COLLECTION                                         |
| Sticky Note3              | Sticky Note                     | Instructions to set QDRANTURL and COLLECTION | -                           | -                            | # STEP 1: Create Qdrant Collection; Change QDRANTURL and COLLECTION                                                        |
| Sticky Note4              | Sticky Note                     | Notes for Step 2 vectorization and Google Drive setup | -                           | -                            | # STEP 2: Documents vectorization with Qdrant and Google Drive; Change QDRANTURL and COLLECTION                             |
| Sticky Note               | Sticky Note                     | Notes on Step 3 summarization alternative | -                           | -                            | ## STEP 3: If you want a "light" and faster rag replace "Set page" with "Summarization Chain"                                |
| Sticky Note1              | Sticky Note                     | Step 4 testing reminder                    | -                           | -                            | ## STEP 4: Test the RAG                                                                                                       |
| Sticky Note2              | Sticky Note                     | Workflow description                       | -                           | -                            | ## Complete RAG system from PDF Documents with Mistral OCR, Qdrant, and Gemini AI                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Qdrant Collection Node (HTTP Request)**
   - Method: PUT  
   - URL: `http://QDRANTURL/collections/COLLECTION` (replace placeholders)  
   - Body: JSON with vectors size 1536, distance "Cosine", shard_number 1, replication_factor 1, write_consistency_factor 1  
   - Headers: Content-Type: application/json  
   - Authentication: HTTP Header Auth with Qdrant API key  
   - Position: top-left (optional)  

2. **Manual Trigger Node ("When clicking ‘Test workflow’")**  
   - Used to start collection refresh and ingestion process.

3. **Refresh Collection Node (HTTP Request)**
   - Method: POST  
   - URL: `http://QDRANTURL/collections/COLLECTION/points/delete`  
   - Body: `{ "filter": {} }`  
   - Headers: Content-Type: application/json  
   - Auth: Same as above

4. **Google Drive Node ("Search PDFs")**  
   - Resource: fileFolder  
   - Filter: folderId set to specific folder containing PDFs  
   - Credentials: Google Drive OAuth2 configured  
   - Output: List of PDF files

5. **SplitInBatches Node ("Loop Over Items1")**  
   - Purpose: Iterate over each PDF file sequentially

6. **Set Node ("Edit Fields1")**  
   - Add field `file_id` with value `={{ $json.id }}`  
   - Prepare input for sub-workflow call

7. **Execute Workflow Node**  
   - Mode: each  
   - Workflow ID: Mistral OCR Sub-Workflow (create separately)  
   - Wait for sub-workflow completion

8. **Sub-Workflow Setup: OCR Processing**

   - Entry Trigger: Execute Workflow Trigger ("When Executed by Another Workflow")  
   - Google Drive Node ("Get PDF")  
     - Operation: download  
     - File ID: `={{ $json.file_id }}`  
     - Credentials: Google Drive OAuth2  
   - HTTP Request Node ("Mistral Upload")  
     - Method: POST  
     - URL: `https://api.mistral.ai/v1/files`  
     - Body: Multipart form-data with `purpose=ocr` and file binary as `data`  
     - Authentication: Mistral Cloud API credentials  
   - HTTP Request Node ("Mistral Signed URL")  
     - Method: GET  
     - URL: `https://api.mistral.ai/v1/files/{{ $json.id }}/url`  
     - Query: expiry=24 hours  
     - Auth: Mistral Cloud API  
   - HTTP Request Node ("Mistral DOC OCR")  
     - Method: POST  
     - URL: `https://api.mistral.ai/v1/ocr`  
     - JSON Body: `{ "model": "mistral-ocr-latest", "document": { "type": "document_url", "document_url": "{{ $json.url }}" }, "include_image_base64": true }`  
     - Auth: Mistral Cloud API  
   - Code Node ("Code")  
     - JavaScript: Extracts markdown pages from OCR response and outputs array of markdown strings  
   - SplitInBatches Node ("Loop Over Items")  
     - Iterates over markdown pages

9. **Vectorization and Storage**

   - Set Node ("Set page")  
     - Assign field `text` = `{{ $json.markdown }}`  
   - LangChain Vector Store Qdrant Node ("Qdrant Vector Store")  
     - Mode: insert  
     - Collection: `ocr_mistral_test` (or your collection)  
     - Credentials: Qdrant API  
   - Wait Node ("Wait")  
     - Optional delay between inserts to manage rate limits  
   - Loop back to "Loop Over Items" to process all pages

10. **Optional Summarization Flow**

    - LangChain Chain Summarization Node ("Summarization Chain")  
      - Prompt: Concise summary in Italian  
      - Model: Google Gemini 2.0 Flash  
    - Set Node ("Set summary")  
      - Store summary text  
    - Connect summarization output as alternative to "Set page" for lighter RAG workflow

11. **RAG Query Handling**

    - LangChain Chat Trigger Node ("When chat message received")  
      - Webhook ID configured for chat interface  
    - LangChain Chain Retrieval QA Node ("Question and Answer Chain")  
      - Connects retrieval and language model  
    - LangChain Vector Store Qdrant Node ("Qdrant Vector Store1")  
      - Used for searching relevant vectors  
    - LangChain Embeddings OpenAI Node ("Embeddings OpenAI1")  
      - Used for embedding queries  
    - LangChain Retriever Vector Store Node ("Vector Store Retriever")  
      - Retrieves relevant document vectors  
    - LangChain LM Chat Google Gemini Node ("Google Gemini Chat Model")  
      - Model: Gemini 1.5 Flash for generating answers  
      - Credentials: Google PaLM API

12. **Credentials Setup**

    - Google Drive OAuth2 with access to target folder  
    - Mistral Cloud API credentials for OCR  
    - Qdrant API key with write/read access  
    - OpenAI API key for embeddings  
    - Google PaLM API credentials for Gemini models  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                         | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Complete RAG system from PDF Documents with Mistral OCR, Qdrant and Gemini AI.                                                                       | Sticky Note2 in workflow description.                                                          |
| Change `QDRANTURL` and `COLLECTION` placeholders in HTTP Request nodes for Qdrant API calls.                                                          | Sticky Note3 and Sticky Note4 instructions.                                                    |
| For faster, lighter RAG, replace the "Set page" node with "Summarization Chain" node to use concise summaries rather than full text chunks.          | Sticky Note near "Set page" node.                                                              |
| Workflow uses Google Gemini PaLM API for both summarization (model gemini-2.0-flash-exp) and answering (model gemini-1.5-flash).                      | Configured in respective LangChain LM Chat Google Gemini nodes.                                |
| Requires Google Drive folder with PDFs accessible by configured OAuth2 account.                                                                       | Folder ID: 1LWVo3yn_1bWQJsLskBIbWTGwlfObvtUK                                                  |
| The sub-workflow for Mistral OCR processing must be created separately with expected input/output as documented in step 8.                          | See "Execute Workflow" node configuration.                                                     |
| Ensure rate limits and API quotas for OpenAI, Google PaLM, and Mistral APIs are monitored to avoid workflow failures during high volume processing. |                                                                                               |

---

This documentation enables developers and AI agents to fully understand, reproduce, and extend the workflow, anticipating key integration points and failure modes.