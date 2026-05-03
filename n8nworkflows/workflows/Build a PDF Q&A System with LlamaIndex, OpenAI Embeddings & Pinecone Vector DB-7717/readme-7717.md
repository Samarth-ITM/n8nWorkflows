Build a PDF Q&A System with LlamaIndex, OpenAI Embeddings & Pinecone Vector DB

https://n8nworkflows.xyz/workflows/build-a-pdf-q-a-system-with-llamaindex--openai-embeddings---pinecone-vector-db-7717


# Build a PDF Q&A System with LlamaIndex, OpenAI Embeddings & Pinecone Vector DB

### 1. Workflow Overview

This workflow automates the ingestion, parsing, normalization, embedding, and indexing of PDF documents stored in a Google Drive folder, enabling a Retrieval-Augmented Generation (RAG) system using LlamaIndex, OpenAI embeddings, and Pinecone vector database. Its primary use case is to build a PDF-based Q&A or chatbot system that semantically searches documents such as insurance policies, legal files, or compliance documents.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Detect new PDF files in Google Drive and download them.
- **1.2 PDF Parsing:** Upload PDFs to LlamaIndex Cloud API for parsing into clean Markdown.
- **1.3 Parsing Status Monitoring:** Poll for parsing job completion.
- **1.4 Content Extraction and Normalization:** Retrieve Markdown output and sanitize text by removing noise and formatting artifacts.
- **1.5 Text Chunking:** Split normalized text into overlapping chunks optimized for embedding.
- **1.6 Embedding Generation:** Generate semantic embeddings for each text chunk using OpenAI.
- **1.7 Vector Storage:** Insert embeddings and metadata into Pinecone for semantic search retrieval.

Supporting sticky notes provide guidance, context, and usage instructions throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Detects newly created PDF files in a specified Google Drive folder and downloads them for processing.
- **Nodes Involved:** 
  - Google Drive Trigger  
  - Download file  
- **Node Details:**

  - **Google Drive Trigger**  
    - *Type:* Trigger node, watches Google Drive folder for new files.  
    - *Configuration:* Monitors a specific folder (folder ID to be specified by user), triggers every minute on 'fileCreated' event.  
    - *Expressions:* Uses `$json.id` to identify the new file.  
    - *Connections:* Output triggers the Download file node.  
    - *Failures:* Possible auth errors if Google OAuth token expires; folder ID must be set correctly.  

  - **Download file**  
    - *Type:* Google Drive operation node, downloads the detected file.  
    - *Configuration:* Uses file ID from trigger node to download the PDF content.  
    - *Connections:* Passes downloaded binary data to Llama Cloud upload node.  
    - *Failures:* Download failures if file is inaccessible or deleted; auth issues if credentials expire.

---

#### 2.2 PDF Parsing

- **Overview:** Uploads the downloaded PDF file to LlamaIndex Cloud parsing API, which converts the PDF to clean Markdown format asynchronously.
- **Nodes Involved:**  
  - Upload to Llama Cloud  
  - Wait  
  - Check Parsing Status  
  - If  
  - Wait2  
- **Node Details:**

  - **Upload to Llama Cloud**  
    - *Type:* HTTP Request node, uploads PDF as multipart form data.  
    - *Configuration:* POST to `https://api.cloud.llamaindex.ai/api/v1/parsing/upload` with bearer token auth. Binary file data is sent in form field `file`.  
    - *Connections:* On success, triggers Wait node to delay before status check.  
    - *Failures:* Network, API limit, or auth errors.  
    - *Notes:* Retries on failure enabled.  

  - **Wait**  
    - *Type:* Wait node, delays 30 seconds allowing parsing job initialization.  
    - *Connections:* Proceeds to Check Parsing Status node.  
    - *Failures:* None anticipated.  

  - **Check Parsing Status**  
    - *Type:* HTTP Request node, polls parsing job status via API GET request.  
    - *Configuration:* URL dynamically constructed with job ID from Upload response.  
    - *Connections:* Output passes to If node for status evaluation.  
    - *Failures:* API errors or wrong job ID.  
    - *Retries:* Enabled.  

  - **If**  
    - *Type:* Conditional node, checks if parsing job status equals "SUCCESS".  
    - *Configuration:* Checks `$json.status == "SUCCESS"`.  
    - *Connections:* If true, proceeds to extract Markdown; if false, loops back to Wait2.  
    - *Failures:* Logic errors if status keys change.  

  - **Wait2**  
    - *Type:* Wait node, delays 60 seconds before polling status again if not successful.  
    - *Connections:* Loops back to Check Parsing Status node.  
    - *Failures:* None anticipated.

---

#### 2.3 Content Extraction and Normalization

- **Overview:** Retrieves parsed Markdown from Llama Cloud, then normalizes text to remove noise such as page numbers, headers, and formatting artifacts.
- **Nodes Involved:**  
  - Extract Markdown from Llama Cloud  
  - Normalize Text  
- **Node Details:**

  - **Extract Markdown from Llama Cloud**  
    - *Type:* HTTP Request node, downloads Markdown result of parsing job.  
    - *Configuration:* GET request to Llama API endpoint with job ID, bearer authentication.  
    - *Connections:* Outputs JSON containing Markdown text to Normalize Text node.  
    - *Failures:* Parsing job incomplete or errors in job ID.  

  - **Normalize Text**  
    - *Type:* Code node (JavaScript), processes Markdown to clean and standardize text.  
    - *Configuration:*  
      - Removes specific phrases like "Car Insurance Policy" with numbers.  
      - Strips page markers ("Page X").  
      - Replaces divider lines with newlines.  
      - Decodes HTML entities (&, bullets).  
      - Collapses multiple line breaks and spaces.  
      - Trims whitespace.  
    - *Input:* Markdown text from previous node.  
    - *Output:* JSON with key `normalizedText`.  
    - *Failures:* Regex assumptions may fail if document format varies; user needs to customize for their documents.

---

#### 2.4 Text Chunking

- **Overview:** Splits normalized text into manageable chunks (~1200 characters with 150-character overlaps) optimized for embedding and retrieval.
- **Nodes Involved:**  
  - Chunk Text  
  - Default Data Loader  
- **Node Details:**

  - **Chunk Text**  
    - *Type:* LangChain Recursive Character Text Splitter node.  
    - *Configuration:*  
      - Splitting mode: markdown-aware.  
      - Chunk size: 1200 characters.  
      - Overlap: 150 characters.  
    - *Connections:* Outputs text chunks to Default Data Loader.  
    - *Failures:* If input text is empty, splitting fails or outputs empty.  

  - **Default Data Loader**  
    - *Type:* LangChain document loader node.  
    - *Role:* Wraps chunked text into document objects for embedding.  
    - *Connections:* Outputs to Generate Embeddings.  
    - *Failures:* None specific.

---

#### 2.5 Embedding Generation

- **Overview:** Generates OpenAI semantic embeddings for each text chunk to facilitate semantic search.
- **Nodes Involved:**  
  - Generate Embeddings  
- **Node Details:**

  - **Generate Embeddings**  
    - *Type:* LangChain OpenAI Embeddings node.  
    - *Configuration:* Uses OpenAI API credentials.  
    - *Connections:* Outputs embeddings to Store in Pinecone.  
    - *Failures:* API rate limits, quota exceeded, or auth errors.

---

#### 2.6 Vector Storage

- **Overview:** Inserts embeddings and associated metadata into a Pinecone vector index under a specified namespace.
- **Nodes Involved:**  
  - Store in Pinecone  
- **Node Details:**

  - **Store in Pinecone**  
    - *Type:* LangChain Pinecone Vector Store node.  
    - *Configuration:*  
      - Mode: Insert.  
      - Namespace: "rag" (retrieval-augmented generation).  
      - Index ID: user-defined Pinecone index.  
    - *Connections:* Terminal node.  
    - *Failures:* Pinecone API errors, network issues, or namespace/index misconfiguration.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                      | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                                        |
|----------------------------|---------------------------------------|------------------------------------|-----------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Google Drive Trigger        | googleDriveTrigger                    | Detect new PDFs in Google Drive    | -                           | Download file                | ## Try It Out! ... (full usage instructions and overview)                                                          |
| Download file              | googleDrive                           | Download PDF file                  | Google Drive Trigger         | Upload to Llama Cloud        |                                                                                                                    |
| Upload to Llama Cloud      | httpRequest                          | Upload PDF for parsing             | Download file                | Wait                        | ## Prepare data - Parse and Normalize                                                                              |
| Wait                      | wait                                 | Delay before status check          | Upload to Llama Cloud        | Check Parsing Status         |                                                                                                                    |
| Check Parsing Status       | httpRequest                          | Poll parsing job status            | Wait, Wait2                 | If                          |                                                                                                                    |
| If                        | if                                   | Check if parsing succeeded         | Check Parsing Status         | Extract Markdown from Llama Cloud, Wait2 |                                                                                                                    |
| Wait2                     | wait                                 | Wait before next status poll       | If                          | Check Parsing Status         |                                                                                                                    |
| Extract Markdown from Llama Cloud | httpRequest                  | Retrieve parsed Markdown           | If                          | Normalize Text               |                                                                                                                    |
| Normalize Text            | code                                 | Clean and normalize Markdown text | Extract Markdown from Llama | Store in Pinecone            | ## Normalized Content ... (notes on removing noise, preserving context, and user customization)                     |
| Chunk Text                | textSplitterRecursiveCharacterTextSplitter | Split text into chunks          | Default Data Loader          | Default Data Loader          | ## Save to Vector DB                                                                                               |
| Default Data Loader       | documentDefaultDataLoader             | Load chunks as documents           | Chunk Text                  | Generate Embeddings          | ## Save to Vector DB                                                                                               |
| Generate Embeddings       | embeddingsOpenAi                      | Generate embeddings                | Default Data Loader          | Store in Pinecone            | ## Save to Vector DB                                                                                               |
| Store in Pinecone         | vectorStorePinecone                   | Insert embeddings into vector DB  | Normalize Text, Default Data Loader, Generate Embeddings | -                      | ## Save to Vector DB                                                                                               |
| Sticky Note               | stickyNote                           | Informational                     | -                           | -                            | ## Save to Vector DB                                                                                                |
| Sticky Note1              | stickyNote                           | Informational                     | -                           | -                            | ## Prepare data - Parse and Normalize                                                                              |
| Sticky Note2              | stickyNote                           | Informational                     | -                           | -                            | ## Normalized Content ... (notes on noise removal and token efficiency)                                           |
| Sticky Note3              | stickyNote                           | Informational                     | -                           | -                            | ## Try It Out! ... (full usage instructions and overview)                                                          |
| Sticky Note4              | stickyNote                           | Informational                     | -                           | -                            | ## Extract Data                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node:**  
   - Set event to `fileCreated`.  
   - Configure to watch a specific folder (set folder ID).  
   - Poll every minute.  
   - Authenticate with Google Drive OAuth2 credentials.  

2. **Add a Google Drive node to download file:**  
   - Operation: `download`.  
   - File ID: Use expression `{{$json["id"]}}` from trigger.  
   - Connect Google Drive Trigger output to this node.  

3. **Add HTTP Request node "Upload to Llama Cloud":**  
   - Method: POST.  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/upload`.  
   - Authentication: HTTP Bearer (LlamaIndex Cloud API key).  
   - Content Type: multipart/form-data.  
   - Body parameter: binary file data field named `file`.  
   - Connect Download file output to this node.  
   - Enable retry on failure.  

4. **Add a Wait node:**  
   - Delay: 30 seconds.  
   - Connect Upload to Llama Cloud output here.  

5. **Add HTTP Request node "Check Parsing Status":**  
   - Method: GET.  
   - URL: `https://api.cloud.llamaindex.ai/api/parsing/job/{{ $json["id"] }}` (use job ID from upload response).  
   - Authentication: HTTP Bearer.  
   - Connect Wait output to this node.  
   - Enable retry on failure.  

6. **Add an If node:**  
   - Condition: Check if `$json.status` equals `SUCCESS`.  
   - Connect Check Parsing Status output to this node.  

7. **Add a Wait node "Wait2":**  
   - Delay: 60 seconds.  
   - Connect the false branch of If node to Wait2.  

8. **Loop Wait2 back to Check Parsing Status:**  
   - Connect Wait2 output to Check Parsing Status input to poll again if parsing incomplete.  

9. **Add HTTP Request node "Extract Markdown from Llama Cloud":**  
   - Method: GET.  
   - URL: `https://api.cloud.llamaindex.ai/api/v1/parsing/job/{{ $json.id }}/result/markdown`.  
   - Authentication: HTTP Bearer.  
   - Connect true branch of If node to this node.  

10. **Add a Code node "Normalize Text":**  
    - Mode: run once for each item.  
    - JavaScript code: to clean and normalize Markdown text (remove headers, footers, page numbers, replace dividers, decode entities, collapse whitespace).  
    - Input: Use the `markdown` or `text` property from previous node.  
    - Connect Extract Markdown node output to this node.  

11. **Add a LangChain Recursive Character Text Splitter node "Chunk Text":**  
    - Chunk size: 1200.  
    - Overlap: 150.  
    - Split code: markdown.  
    - Connect Normalize Text output to this node.  

12. **Add LangChain Default Data Loader node:**  
    - Use to wrap text chunks as document objects.  
    - Connect Chunk Text output to this node.  

13. **Add LangChain OpenAI Embeddings node "Generate Embeddings":**  
    - Configure OpenAI API credentials.  
    - Connect Default Data Loader output to this node.  

14. **Add LangChain Pinecone Vector Store node "Store in Pinecone":**  
    - Mode: insert.  
    - Pinecone namespace: `rag`.  
    - Select Pinecone index ID (configured with Pinecone API credentials).  
    - Connect Normalize Text, Default Data Loader, and Generate Embeddings outputs to this node (as appropriate for embeddings and metadata ingestion).  

15. **Add sticky notes for documentation at relevant workflow points:**  
    - Use content from Sticky Note nodes in the original workflow to provide guidance on data preparation, normalization, saving to vector DB, and overall usage instructions.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| This workflow demonstrates building a full RAG pipeline for PDFs using Google Drive, LlamaIndex Cloud parsing, OpenAI embeddings, and Pinecone vector DB. Use it for chatbot/Q&A systems on structured documents like insurance policies. Customize folder IDs, regex in normalization, chunk sizes, and namespaces per your needs. Requires accounts and API keys for Google Drive, LlamaIndex, OpenAI, and Pinecone. Ask for help on the [n8n Forum](https://community.n8n.io/). Happy Automating! 🚀                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | [n8n Forum](https://community.n8n.io/)                             |
| Adjust regex in the Normalize Text node to suit your document formats (headers, footers, page numbering). This step is crucial for quality retrieval and preventing token waste.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Important customization note                                        |
| The chunk size of 1200 characters with 150 character overlap balances embedding quality and token usage. Modify according to your document complexity and API quotas.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Embedding configuration advice                                     |
| Pinecone namespace "rag" is used for storing vector embeddings; ensure your Pinecone index is properly configured and accessible with correct credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Pinecone configuration note                                        |
| The LlamaIndex Cloud parsing API is asynchronous; status polling with delays ensures robust handling of parsing job completion. Retry and failure handling are enabled to improve reliability.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow stability and robustness note                             |

---

**Disclaimer:** The provided content originates solely from an automated workflow created with n8n, an integration and automation tool. The process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed are legal and public.