Document RAG & Chat Agent: Google Drive to Qdrant with Mistral OCR

https://n8nworkflows.xyz/workflows/document-rag---chat-agent--google-drive-to-qdrant-with-mistral-ocr-10096


# Document RAG & Chat Agent: Google Drive to Qdrant with Mistral OCR

---

### 1. Workflow Overview

This workflow automates the ingestion, processing, and querying of documents stored in a designated Google Drive folder to create a searchable knowledge base integrated with an AI chat assistant. It transforms various document formats into text via OCR, enriches documents with metadata, splits text into manageable chunks, converts them into semantic vector embeddings, and stores them in a Qdrant vector database. The chat agent then leverages this knowledge base for retrieval-augmented generation (RAG), optionally supplemented by external web search results.

The workflow contains two main logical blocks:

**1.1 Document Ingestion and Indexing Pipeline**  
- Lists files from a specific Google Drive folder  
- Downloads files and processes them with Mistral OCR to extract text  
- Applies AI-based metadata extraction for categorization  
- Cleans, chunks, and embeds the text data using OpenAI embeddings  
- Stores embeddings in Qdrant vector store for semantic retrieval

**1.2 Interactive AI Chat Agent with RAG and Optional Web Search**  
- Receives chat messages via a webhook trigger  
- Processes user queries using an AI chat agent configured with a system prompt  
- Uses Qdrant vector search as the primary knowledge retrieval tool  
- Converts queries to embeddings for similarity search  
- Falls back to external web search (Tavily API) upon user consent if no internal data matches  
- Maintains conversation memory for context continuity

---

### 2. Block-by-Block Analysis

---

#### 2.1 Document Ingestion and Indexing Pipeline

**Overview:**  
This block automates fetching all files from a Google Drive folder, extracting text content via OCR, enriching documents with metadata, chunking text for manageable processing, embedding chunks with OpenAI, and finally storing the embeddings into Qdrant.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Google Drive (List Files)  
- Loop Over each file in gdrive folder (SplitInBatches)  
- add metadata (Set)  
- Google Drive1 (Download File)  
- Mistral Upload (HTTP Request)  
- Mistral Signed URL (HTTP Request)  
- Mistral DOC OCR (HTTP Request)  
- If NODE (If)  
- based file name it assign differ metadata (Information Extractor)  
- set all metadata (Set)  
- clean output (Code)  
- convert data into smaller chunks (Code)  
- Embeddings OpenAI (LangChain Embeddings)  
- Qdrant Vector Store1 (LangChain Vector Store Insert)  
- Character Text Splitter (Text Splitter)  
- Default Data Loader (Document Data Loader)  

---

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Kicks off the ingestion pipeline for testing or manual execution  
  - Configuration: Default manual trigger, no parameters  
  - Inputs: None  
  - Outputs: Google Drive (List Files)  
  - Edge Cases: None

- **Google Drive (List Files)**  
  - Type: Google Drive node (List files in folder)  
  - Role: Retrieves the list of files from the specified Google Drive folder (`knowledgebaseforaibot`)  
  - Configuration: Queries folder ID `1C1zD1XefBltEAocX6kfHFbzQtzzAxo_E`, requests file metadata fields including id, name, webViewLink, mimeType  
  - Inputs: Manual Trigger  
  - Outputs: SplitInBatches (Loop Over each file)  
  - Edge Cases: Folder empty, permission errors, API rate limits

- **Loop Over each file in gdrive folder**  
  - Type: SplitInBatches  
  - Role: Processes each file individually downstream  
  - Configuration: Default batch size (default 1) for individual file processing  
  - Inputs: Google Drive file list  
  - Outputs: add metadata (Set) and also loops back if necessary  
  - Edge Cases: Large file lists may affect performance

- **add metadata**  
  - Type: Set node  
  - Role: Extracts and assigns key metadata from each file object to structured fields for use downstream  
  - Configuration: Assigns fields like file_id, file_type, file_title, file_url, last_modified_date from file JSON  
  - Inputs: Loop Over each file  
  - Outputs: Google Drive1 (Download File)  
  - Edge Cases: Missing metadata fields in Google Drive response

- **Google Drive1 (Download File)**  
  - Type: Google Drive node (Download file)  
  - Role: Downloads the actual content/binary of the current file for OCR processing  
  - Configuration: Uses fileId from `add metadata` node's file_url field  
  - Inputs: add metadata  
  - Outputs: Mistral Upload (HTTP Request)  
  - Edge Cases: File not found, permission denied, unsupported file types

- **Mistral Upload**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded file binary to Mistral API for OCR processing  
  - Configuration: POST to `https://api.mistral.ai/v1/files` with multipart-form-data including purpose=`ocr` and binary file data  
  - Authentication: Mistral Cloud API credential  
  - Inputs: Google Drive1 (file binary)  
  - Outputs: Mistral Signed URL  
  - Edge Cases: Upload failures, API authentication errors, large file size issues

- **Mistral Signed URL**  
  - Type: HTTP Request  
  - Role: Fetches a temporary signed URL from Mistral for accessing the uploaded file  
  - Configuration: GET request to `https://api.mistral.ai/v1/files/{{ file_id }}/url` with expiry parameter set to 24 hours  
  - Authentication: Mistral Cloud API credential  
  - Inputs: Mistral Upload (file upload response with file ID)  
  - Outputs: Mistral DOC OCR  
  - Edge Cases: Expired URLs, permission errors

- **Mistral DOC OCR**  
  - Type: HTTP Request  
  - Role: Calls Mistral OCR service to extract text content from the document via the signed URL  
  - Configuration: POST to `https://api.mistral.ai/v1/ocr` with JSON body specifying model `mistral-ocr-latest` and document URL  
  - Authentication: Mistral Cloud API credential  
  - Inputs: Mistral Signed URL (signed URL JSON)  
  - Outputs: If NODE  
  - Edge Cases: OCR failures, unsupported file formats, API timeouts

- **If NODE**  
  - Type: If node  
  - Role: Checks whether OCR extraction succeeded and data is valid (no skipped flag)  
  - Configuration: Condition checks that `data[0].parseJson().skipped` does not exist  
  - Inputs: Mistral DOC OCR  
  - Outputs: based file name it assign differ metadata (true path) or loops back to Loop Over each file (false)  
  - Edge Cases: False negatives if OCR response malformed

- **based file name it assign differ metadata (Information Extractor)**  
  - Type: LangChain Information Extractor  
  - Role: Uses AI to analyze extracted text (first page markdown) and assign metadata fields: document_type, project, assigned_to  
  - Configuration: System prompt defines extraction rules and expected attributes  
  - Inputs: If NODE (OCR success)  
  - Outputs: set all metadata  
  - Edge Cases: Extraction inaccuracies, ambiguous inputs

- **set all metadata (Set node)**  
  - Type: Set node  
  - Role: Combines extracted metadata fields with file metadata to create a unified document metadata object  
  - Configuration: Assigns document name, document data (full markdown), source file ID, assigned persons (as list), project, document type  
  - Inputs: based file name it assign differ metadata  
  - Outputs: clean output  
  - Edge Cases: Missing or inconsistent metadata

- **clean output (Code node)**  
  - Type: Code node (JavaScript)  
  - Role: Formats the document data and metadata into a consistent JSON structure for embedding  
  - Configuration: Extracts content and metadata, wraps in array of objects with `content` and `metadata` keys  
  - Inputs: set all metadata  
  - Outputs: convert data into smaller chunks  
  - Edge Cases: Empty or invalid content

- **convert data into smaller chunks (Code node)**  
  - Type: Code node (JavaScript)  
  - Role: Splits document text into overlapping chunks (~1000 characters, 100 overlap) for embedding  
  - Configuration: Iterates text, slices chunks, creates new items with content and preserved metadata  
  - Inputs: clean output  
  - Outputs: Embeddings OpenAI  
  - Edge Cases: Very short documents, encoding errors (note: code references undefined variable `updatedText` which should be fixed to `text`)

- **Embeddings OpenAI**  
  - Type: LangChain Embeddings OpenAI  
  - Role: Converts each text chunk into a vector embedding using `text-embedding-3-small` model  
  - Configuration: Default options, OpenAI API credentials provided  
  - Inputs: convert data into smaller chunks  
  - Outputs: Qdrant Vector Store1 (insert mode)  
  - Edge Cases: API rate limits, token limits

- **Qdrant Vector Store1**  
  - Type: LangChain Vector Store Qdrant (insert mode)  
  - Role: Inserts embeddings and metadata chunks into Qdrant collection `docaiauto` for semantic search  
  - Configuration: Batch size 200, Qdrant API credentials provided  
  - Inputs: Embeddings OpenAI  
  - Outputs: Loop Over each file in gdrive folder (to continue processing next file)  
  - Edge Cases: Qdrant connectivity, insertion errors

- **Character Text Splitter** (appears unused, possibly for alternative chunking)  
  - Type: LangChain Text Splitter (Character-based)  
  - Role: Optionally splits text by character count for downstream processing  
  - Inputs: Default Data Loader (which loads JSON data)  
  - Outputs: Default Data Loader → Qdrant Vector Store1 for insertion  
  - Edge Cases: None specified

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Loads JSON document data and metadata into LangChain document format for embedding or indexing  
  - Inputs: Character Text Splitter  
  - Outputs: Qdrant Vector Store1  
  - Edge Cases: Input data must be JSON-compliant

---

#### 2.2 Interactive AI Chat Agent with RAG and Optional Web Search

**Overview:**  
This block manages real-time user interactions via a chat interface, utilizing an AI agent that queries the internal Qdrant knowledge base first and optionally performs a web search via Tavily API if no internal data is found or user consents.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- ai chat agent (LangChain Agent)  
- OpenAI Chat Model1 (LangChain Chat Model)  
- Simple Memory1 (LangChain Memory Buffer)  
- Qdrant Vector Store (LangChain Vector Store Retrieve as tool)  
- Embeddings OpenAI1 (LangChain Embeddings)  
- Web Search (LangChain HTTP Request to Tavily API)  

---

**Node Details:**

- **When chat message received (Chat Trigger)**  
  - Type: LangChain Chat Trigger  
  - Role: Initiates workflow when a new message is received on the chat interface webhook  
  - Configuration: Public webhook enabled, default options  
  - Inputs: Incoming chat messages from users  
  - Outputs: ai chat agent  
  - Edge Cases: Webhook connectivity, malformed messages

- **ai chat agent (LangChain Agent)**  
  - Type: LangChain Agent  
  - Role: Core conversational AI orchestrator managing user input, memory, tool usage, and response generation  
  - Configuration:  
    - System prompt defines detailed rules for knowledge search prioritization, response style, image inclusion, citation format, fallback web search logic, and proactive engagement  
    - Tools include Qdrant vector store and optional web search  
    - Language model: OpenAI GPT-4.1-mini or Mistral Cloud Chat Model (depending on flow)  
  - Inputs: When chat message received, Simple Memory1, Qdrant Vector Store, Web Search  
  - Outputs: Final chat responses  
  - Edge Cases: Tool invocation failures, memory overflow, prompt errors

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the language understanding and generation capabilities powering the agent  
  - Configuration: Model `gpt-4.1-mini`, temperature 0.5 for balanced creativity and accuracy, OpenAI API credentials configured  
  - Inputs: ai chat agent (as language model)  
  - Outputs: ai chat agent  
  - Edge Cases: API rate limits, token limits, model availability

- **Simple Memory1**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains short-term conversation history to provide context for follow-up questions  
  - Configuration: Default window size (not specified)  
  - Inputs: ai chat agent  
  - Outputs: ai chat agent  
  - Edge Cases: Memory overflow or loss

- **Qdrant Vector Store**  
  - Type: LangChain Vector Store Qdrant (retrieve-as-tool mode)  
  - Role: Provides semantic search over the internal knowledge base collection `docaiauto` for relevant information retrieval during conversation  
  - Configuration: Retrieves top 3 most relevant chunks, configured as a tool with name and description for the agent  
  - Inputs: ai chat agent (tool invocation)  
  - Outputs: ai chat agent  
  - Edge Cases: No relevant documents found, connectivity issues

- **Embeddings OpenAI1**  
  - Type: LangChain Embeddings OpenAI  
  - Role: Converts user query text into vector embeddings for similarity search in Qdrant  
  - Configuration: Default OpenAI embedding options, OpenAI API credentials  
  - Inputs: ai chat agent (tool embedding requests)  
  - Outputs: Qdrant Vector Store  
  - Edge Cases: API limits, embedding errors

- **Web Search**  
  - Type: LangChain HTTP Request node (Tavily API)  
  - Role: Performs external web search to supplement internal knowledge base when user consents  
  - Configuration:  
    - POST to https://api.tavily.com/search with JSON body containing query and parameters for advanced search  
    - Headers include Authorization (to be replaced with valid Tavily token) and Content-Type application/json  
    - Returns optimized structured results with answers and URLs  
  - Inputs: ai chat agent (tool invocation on user approval)  
  - Outputs: ai chat agent  
  - Edge Cases: API authentication failures, rate limits, no results

---

### 3. Summary Table

| Node Name                     | Node Type                                     | Functional Role                                      | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                                                                                      |
|-------------------------------|-----------------------------------------------|-----------------------------------------------------|---------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                                | Starts ingestion pipeline manually                   | -                               | Google Drive                    |                                                                                                                                                                                  |
| Google Drive                  | Google Drive (List files)                      | Lists files in Google Drive folder                   | When clicking ‘Test workflow’    | Loop Over each file in gdrive folder | ## GET ALL FILE DATA FROM SELECTED GOOGLE DRIVE FOLDER                                                                                                                           |
| Loop Over each file in gdrive folder | SplitInBatches                        | Loops over each file individually                     | Google Drive                   | add metadata, (loop back)        | ## loop over google drive folder items                                                                                                                                           |
| add metadata                  | Set                                            | Extracts and assigns file metadata                    | Loop Over each file              | Google Drive1                   | ## GET  individual files from selected gdrive                                                                                                                                     |
| Google Drive1                 | Google Drive (Download file)                    | Downloads file content                                | add metadata                   | Mistral Upload                 | ## GET  individual files from selected gdrive                                                                                                                                     |
| Mistral Upload               | HTTP Request                                   | Uploads file to Mistral for OCR processing           | Google Drive1                  | Mistral Signed URL             | ## MISTRAL OCR [OCR Guide](https://mistral.ai/news/mistral-ocr) 1. UPLOAD FILE 2. GET SIGNED URL 3. GET EXTRACT DATA AFTER USING MISTRAL OCR                                        |
| Mistral Signed URL           | HTTP Request                                   | Retrieves signed URL for uploaded file                | Mistral Upload                | Mistral DOC OCR               | ## MISTRAL OCR [OCR Guide](https://mistral.ai/news/mistral-ocr) 1. UPLOAD FILE 2. GET SIGNED URL 3. GET EXTRACT DATA AFTER USING MISTRAL OCR                                        |
| Mistral DOC OCR              | HTTP Request                                   | Extracts OCR text from document                        | Mistral Signed URL            | If NODE                       | ## MISTRAL OCR [OCR Guide](https://mistral.ai/news/mistral-ocr) 1. UPLOAD FILE 2. GET SIGNED URL 3. GET EXTRACT DATA AFTER USING MISTRAL OCR                                        |
| If NODE                      | If                                             | Checks OCR success                                    | Mistral DOC OCR               | based file name it assign differ metadata, Loop Over each file in gdrive folder | ## Remove  empty data fields                                                                                                                                                        |
| based file name it assign differ metadata | LangChain Information Extractor     | Extracts document type, project, assigned_to metadata | If NODE                       | set all metadata              | ## assignment agent for any given file this node assign which type documents it is ,which project its related too and who are working on it                                        |
| set all metadata             | Set                                            | Combines AI metadata with file metadata               | based file name it assign differ metadata | clean output                 |                                                                                                                                                                                  |
| clean output                 | Code                                           | Formats and cleans document data for chunking         | set all metadata              | convert data into smaller chunks | ## clean all extracted data and convert them to smaller chunks                                                                                                                    |
| convert data into smaller chunks | Code                                      | Splits text into smaller overlapping chunks           | clean output                  | Embeddings OpenAI             | ## clean all extracted data and convert them to smaller chunks                                                                                                                    |
| Embeddings OpenAI            | LangChain Embeddings OpenAI                     | Creates vector embeddings from text chunks             | convert data into smaller chunks | Qdrant Vector Store1          | ## QDRANT VCETOR AND OPEN API EMBEDDING [QDRANT Guide](https://qdrant.tech/documentation/)                                                                                          |
| Qdrant Vector Store1         | LangChain Vector Store Qdrant (insert mode)    | Inserts embeddings into Qdrant collection              | Embeddings OpenAI             | Loop Over each file in gdrive folder | ## load all chunks into qdrant vector database                                                                                                                                     |
| Character Text Splitter      | LangChain Text Splitter CharacterTextSplitter  | Optional text chunking by character count              | Default Data Loader           | Default Data Loader           |                                                                                                                                                                                  |
| Default Data Loader          | LangChain Document Default Data Loader          | Loads document JSON data for LangChain processing      | Character Text Splitter       | Qdrant Vector Store1          |                                                                                                                                                                                  |
| When chat message received   | LangChain Chat Trigger                           | Starts chat workflow on incoming message               | -                           | ai chat agent                | ## Hosted Chat interface                                                                                                                                                           |
| ai chat agent                | LangChain Agent                                  | Main AI chat orchestrator with tools and memory        | When chat message received, Simple Memory1, Qdrant Vector Store, Web Search | -                           | ## AI chat agent interact with user and process user input and provide appropriate response using different tools.                                                                |
| OpenAI Chat Model1           | LangChain OpenAI Chat Model                      | Provides core language model for chat agent             | ai chat agent (language model) | ai chat agent               |                                                                                                                                                                                  |
| Simple Memory1               | LangChain Memory Buffer Window                   | Maintains conversation context memory                   | ai chat agent               | ai chat agent               |                                                                                                                                                                                  |
| Qdrant Vector Store          | LangChain Vector Store Qdrant (retrieve mode)   | Searches internal knowledge base for relevant info     | ai chat agent (tool)         | ai chat agent               | ## QDRANT VCETOR AND OPEN API EMBEDDING [QDRANT Guide](https://qdrant.tech/documentation/)                                                                                          |
| Embeddings OpenAI1           | LangChain Embeddings OpenAI                      | Embeds user queries for semantic search                  | ai chat agent (tool)         | Qdrant Vector Store          | ## QDRANT VCETOR AND OPEN API EMBEDDING [QDRANT Guide](https://qdrant.tech/documentation/)                                                                                          |
| Web Search                  | LangChain HTTP Request (Tavily API)               | Performs external web search on user consent             | ai chat agent (tool)         | ai chat agent               | ## WEB SEARCH using tavily (http node) [Tavily setup Guide](https://docs.tavily.com/welcome)                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node** named `When clicking ‘Test workflow’` to start the ingestion pipeline manually.

2. **Add Google Drive node** (List Files) named `Google Drive` configured to list files in folder ID `1C1zD1XefBltEAocX6kfHFbzQtzzAxo_E`. Request fields `id`, `name`, `webViewLink`, `mimeType`, and others. Connect output from manual trigger.

3. **Add SplitInBatches node** named `Loop Over each file in gdrive folder` to iterate over each file individually. Connect from Google Drive node.

4. **Add Set node** named `add metadata` to extract and assign metadata fields from each file item:  
   - `file_id` = `{{$json.id}}`  
   - `file_type` = `{{$json.mimeType}}`  
   - `file_title` = `{{$json.name}}`  
   - `file_url` = `{{$json.webViewLink}}`  
   - `last_modified_date` = `{{$json.modifiedTime}}`  
   Connect from Loop Over each file node.

5. **Add Google Drive node** (Download file) named `Google Drive1` to download the current file using `file_url` from `add metadata`. Connect from `add metadata`.

6. **Add HTTP Request node** named `Mistral Upload` to upload the downloaded file to Mistral OCR API:  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/files`  
   - Authentication: Use Mistral Cloud API credentials  
   - Content Type: multipart-form-data  
   - Body Parameters:  
     - `purpose` = `ocr` (text)  
     - `file` = binary data input from `Google Drive1`  
   Connect from `Google Drive1`.

7. **Add HTTP Request node** named `Mistral Signed URL` to get signed URL for uploaded file:  
   - Method: GET  
   - URL: `https://api.mistral.ai/v1/files/{{ $json.id }}/url` (use file ID from upload response)  
   - Query Parameter: `expiry=24` (hours)  
   - Authentication: Mistral Cloud API credentials  
   Connect from `Mistral Upload`.

8. **Add HTTP Request node** named `Mistral DOC OCR` to request OCR extraction:  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/ocr`  
   - JSON Body:  
     ```json
     {
       "model": "mistral-ocr-latest",
       "document": {
         "type": "document_url",
         "document_url": "{{ $json.url }}"
       },
       "include_image_base64": true
     }
     ```  
   - Authentication: Mistral Cloud API credentials  
   Connect from `Mistral Signed URL`.

9. **Add If node** named `If NODE` to check OCR success:  
   - Condition: Check that `data[0].parseJson().skipped` field does not exist (not true)  
   - True output: Connect to AI metadata extractor  
   - False output: Connect back to loop to skip or handle error  
   Connect from `Mistral DOC OCR`.

10. **Add LangChain Information Extractor node** named `based file name it assign differ metadata` to extract metadata:  
    - Input Text: `{{ $json.pages[0].markdown }}` (first page OCR text)  
    - Attributes to extract: `document_type`, `project`, `assigned_to`  
    - System prompt: Custom expert extraction algorithm prompt  
    Connect from `If NODE` (true branch).

11. **Add Set node** named `set all metadata` to combine extracted metadata with file info:  
    - Assign fields:  
      - Document name = from Google Drive file title  
      - Document data = OCR text markdown  
      - source = Google Drive file ID  
      - ASSIGNEDTO = split assigned_to from AI extractor  
      - PROJECT = project from AI extractor  
      - DOCUMENT_TYPE = document_type from AI extractor  
    Connect from `based file name it assign differ metadata`.

12. **Add Code node** named `clean output` to format data for chunking:  
    - Extract content and metadata into array of objects with keys `content` and `metadata`  
    - Return array of JSON items  
    Connect from `set all metadata`.

13. **Add Code node** named `convert data into smaller chunks` to split text into ~1000 character chunks with 100 character overlap:  
    - Iterate over text content, slice chunks, preserve metadata  
    - Return array of chunked JSON items  
    Connect from `clean output`.

14. **Add LangChain Embeddings OpenAI node** named `Embeddings OpenAI`:  
    - Model: `text-embedding-3-small`  
    - Credentials: OpenAI API  
    Connect from `convert data into smaller chunks`.

15. **Add LangChain Vector Store Qdrant node** named `Qdrant Vector Store1`:  
    - Mode: Insert  
    - Collection: `docaiauto`  
    - Batch size: 200  
    - Credentials: Qdrant API  
    Connect from `Embeddings OpenAI`.

16. **Connect output from `Qdrant Vector Store1` back to `Loop Over each file in gdrive folder`** to continue processing next file until all files processed.

---

17. **Create Chat Trigger node** named `When chat message received` with public webhook enabled.

18. **Add LangChain Agent node** named `ai chat agent` configured with a detailed system prompt that:  
    - Defines knowledge access rules using Qdrant vector store as primary tool  
    - Uses OpenAI Chat Model and optionally web search as fallback  
    - Specifies response formatting, image inline rules, citation style  
    - Includes proactive conversation behavior  
    Connect from `When chat message received`.

19. **Add LangChain OpenAI Chat Model node** named `OpenAI Chat Model1` configured with model `gpt-4.1-mini`, temperature 0.5, OpenAI credentials. Connect as language model to `ai chat agent`.

20. **Add LangChain Memory Buffer Window node** named `Simple Memory1` with default window size, connected as conversation memory in `ai chat agent`.

21. **Add LangChain Embeddings OpenAI node** named `Embeddings OpenAI1` to embed user queries, connected as embedding tool in `ai chat agent`.

22. **Add LangChain Vector Store Qdrant node** named `Qdrant Vector Store` in retrieve-as-tool mode, configured to search top 3 results from collection `docaiauto`, connected as tool for `ai chat agent`.

23. **Add HTTP Request node** named `Web Search` configured to call Tavily API for external web search:  
    - POST to `https://api.tavily.com/search`  
    - JSON body includes query placeholder `{query}` and search parameters for general topic, advanced depth, max 20 results, include_answer true  
    - Headers include Authorization (replace with valid Tavily token) and Content-Type: application/json  
    Connect as optional tool in `ai chat agent` invoked on user approval.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                           | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates knowledge ingestion from Google Drive, OCR text extraction via Mistral, metadata enrichment, embedding with OpenAI, storage in Qdrant, and AI chat. | Workflow Title and Description                                                                 |
| Web Search uses Tavily API for external search augmentation. Setup guide: https://docs.tavily.com/welcome                                                              | Sticky Note on Web Search node                                                                 |
| Mistral OCR guide and usage explained at: https://mistral.ai/news/mistral-ocr                                                                                        | Sticky Note on Mistral OCR nodes                                                               |
| Qdrant vector store setup and documentation: https://qdrant.tech/documentation/                                                                                      | Sticky Notes on Qdrant Vector Store and Embeddings nodes                                        |
| AI Chat Agent designed to proactively engage users, include multiple inline images, cite document sources, and fallback to web search only on user consent.          | Agent system prompt and Sticky Notes                                                           |
| Workflow designed by DIGITAL BIZ TECH                                                                                                                                | Branding sticky note                                                                           |

---

**Disclaimer:** The provided text and workflow are generated solely from an automated n8n workflow. All data processed is legal, public, and compliant with content policies. No illegal or offensive content is included.

---