Website Chatbot with Google Drive Knowledge Base using GPT-4 and Mistral AI

https://n8nworkflows.xyz/workflows/website-chatbot-with-google-drive-knowledge-base-using-gpt-4-and-mistral-ai-10142


# Website Chatbot with Google Drive Knowledge Base using GPT-4 and Mistral AI

---

## 1. Workflow Overview

This workflow enables a **website chatbot** that answers user queries based on a **knowledge base of documents stored in Google Drive**. It integrates advanced AI models — GPT-4 and Mistral AI — to process document content, perform OCR, embed document chunks as vectors into a Qdrant vector store, and leverages vector search to provide accurate, context-aware chatbot responses.

The workflow is logically divided into two main functional blocks:

- **1.1 Knowledge Base Preparation**  
  Automates the ingestion of documents from a specified Google Drive folder, processes them via Mistral OCR, chunks the extracted content, converts these chunks into vector embeddings using Mistral AI, and stores them in Qdrant vector database for efficient retrieval.

- **1.2 Website Chatbot Query Handling**  
  Listens for chat messages (both embedded and webhook-triggered), retrieves relevant document vectors from Qdrant using Mistral embeddings, and generates responses using GPT-4 guided by a custom system prompt tailored to the brand’s knowledge base.

---

## 2. Block-by-Block Analysis

### 2.1 Knowledge Base Preparation

**Overview:**  
This block orchestrates loading documents from Google Drive, performing OCR extraction via Mistral AI, splitting large text into chunks, embedding chunks into vector representations, and storing them into the Qdrant vector database to build the website chatbot’s knowledge base.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Google Drive (brand related data for chatbot)  
- Loop Over Items1 (Batch splitter)  
- Set metadata  
- Google Drive(load file)  
- Mistral Upload  
- Mistral Signed URL  
- Mistral DOC OCR  
- If2 (Conditional check)  
- Wait  
- prepare for chunking (Set node)  
- Code(convert to chunks for loading into vector db)  
- Character Text Splitter  
- Default Data Loader  
- Embeddings Mistral Cloud  
- Qdrant Vector Store1  
- Sticky Note4, Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note8 (explanatory notes)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the knowledge base update process on demand.  
  - Inputs: None  
  - Outputs: Triggers Google Drive folder listing.  
  - Failure: None typical; manual start.

- **Google Drive (brand related data for chatbot)**  
  - Type: Google Drive File/Folder Search  
  - Role: Lists all files in specific Google Drive folder (folderId = "1o3DK9Ceka5Lqb8irvFSfEeB8SVGG_OL7") containing documents to ingest.  
  - Parameters: Filter set to folderId, retrieves metadata fields (id, name, webViewLink, mimeType, etc.)  
  - Output: List of files to process.  
  - Credential: Google Drive OAuth2.  
  - Failures: API quota, auth errors, folder access permissions.

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Processes files one by one to avoid memory overload.  
  - Inputs: List of files from Google Drive node.  
  - Outputs: Each file’s metadata sequentially.

- **Set metadata**  
  - Type: Set  
  - Role: Extracts and formats metadata fields from each Google Drive file item for downstream use (file_id, file_type, file_title, file_url, last_modified_date).  
  - Inputs: Single file metadata.  
  - Outputs: Metadata-enriched item.

- **Google Drive (load file)**  
  - Type: Google Drive File Download  
  - Role: Downloads the actual file content using file_id from Set metadata node.  
  - Inputs: file_id from previous node.  
  - Outputs: Binary file data for OCR upload.  
  - Failures: File access denied, missing file, network errors.

- **Mistral Upload**  
  - Type: HTTP Request (POST multipart-form-data)  
  - Role: Uploads downloaded file to Mistral API for OCR processing (purpose = "ocr").  
  - Inputs: Binary data from Google Drive(load file).  
  - Outputs: File ID assigned by Mistral.  
  - Credential: Mistral Cloud API key.  
  - Failures: API errors, network timeout.

- **Mistral Signed URL**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves a signed URL from Mistral API to access the uploaded file for OCR processing.  
  - Inputs: Mistral file ID.  
  - Outputs: URL for OCR request.  
  - Failures: Expired tokens, invalid file ID.

- **Mistral DOC OCR**  
  - Type: HTTP Request (POST JSON)  
  - Role: Sends OCR request to Mistral AI using the signed URL to extract document text and images.  
  - Inputs: Signed URL from previous node.  
  - Outputs: OCR results with extracted blocks and metadata.  
  - OnError: Continues on error to avoid full workflow failure; retries enabled.  
  - Failures: OCR model errors, malformed input, rate limits.

- **If2 (Conditional check)**  
  - Type: If  
  - Role: Checks if OCR result contains skipped data (indicating incomplete or failed OCR).  
  - Condition: Evaluates if 'skipped' field exists and its absence triggers chunking.  
  - Outputs: Directs to chunk processing or Wait node for retry.

- **Wait**  
  - Type: Wait  
  - Role: Provides a delay if OCR is incomplete or requires retry.  
  - Inputs: From If2 fail branch.  
  - Outputs: Loops back to Google Drive(load file) after waiting.

- **prepare for chunking**  
  - Type: Set  
  - Role: Extracts OCR blocks and source metadata, prepares JSON strings for chunking.  
  - Inputs: OCR data from Mistral DOC OCR.  
  - Outputs: Document data and name for chunking.  
  - Expressions used to parse JSON paths.

- **Code(convert to chunks for loading into vector db)**  
  - Type: Code (JavaScript)  
  - Role: Splits large OCR content into manageable chunks (~1000 characters) for vector embedding. Handles oversize content by splitting further, preserves metadata for each chunk.  
  - Inputs: JSON string of OCR blocks and metadata.  
  - Outputs: Array of chunked document data items.  
  - Edge cases: Very large documents, OCR images ignored for chunk length.

- **Character Text Splitter**  
  - Type: Text Splitter (character based)  
  - Role: (Not directly connected in main flow) Possibly legacy or alternative text chunking.  
  - Parameters: chunkSize set to an extremely large number (10,000,000), effectively no splitting.  
  - Note: May be unused or for specific use cases.

- **Default Data Loader**  
  - Type: Document Data Loader  
  - Role: Loads chunked JSON data into a format suitable for vector store insertion.  
  - Inputs: Chunked JSON strings from Code node.  
  - Outputs: Document objects with metadata for embedding.  
  - Parameters: Metadata fields mapped from OCR metadata.

- **Embeddings Mistral Cloud**  
  - Type: Embeddings Node (Mistral AI)  
  - Role: Generates vector embeddings from document chunks for semantic search.  
  - Inputs: Document data from Default Data Loader.  
  - Outputs: Embeddings for each chunk.  
  - Credential: Mistral Cloud API.  
  - Failures: API limits, malformed input.

- **Qdrant Vector Store1**  
  - Type: Vector Store (Qdrant)  
  - Role: Inserts vector embeddings into Qdrant collection "docragtestkb".  
  - Inputs: Embeddings from Embeddings Mistral Cloud.  
  - Outputs: Confirmation of insertion.  
  - Credential: Qdrant API.  
  - Parameters: Batch size 200 embeddings per insert.  
  - Failures: API errors, collection access issues.

- **Loop Over Items1 (Second output branch)**  
  - Type: SplitInBatches continuation  
  - Role: Iterates over files to process each individually.

- **Sticky Notes**  
  - Provide documentation and guidance for OCR flow, chunking, vector store usage, and workflow purpose with useful external links (e.g., [OCR Guide](https://mistral.ai/news/mistral-ocr)) and project context.

---

### 2.2 Website Chatbot Query Handling

**Overview:**  
This block listens for incoming chat messages via webhook or Langchain chat trigger, uses semantic vector search in Qdrant with Mistral embeddings to retrieve relevant document chunks, and generates friendly, brand-focused answers with GPT-4.

**Nodes Involved:**  
- When chat message received (Langchain Chat Trigger)  
- Webhook (HTTP POST listener)  
- Qdrant Vector Store (retrieve)  
- Embeddings Mistral Cloud1  
- OpenAI Chat Model (GPT-4)  
- Simple Memory (Langchain Memory Buffer)  
- website chat agent (Langchain Agent)  
- Sticky Note5, Sticky Note6, Sticky Note7 (explanatory notes)

**Node Details:**

- **When chat message received**  
  - Type: Langchain Chat Trigger (webhook mode)  
  - Role: Listens publicly for chat messages from embedded chat interface or external sources.  
  - Parameters: Response mode set to output from last node (the agent).  
  - Outputs: Message text for chatbot processing.

- **Webhook**  
  - Type: n8n Webhook  
  - Role: Alternative entry point for chat messages (HTTP POST on path "75458eba-ed7b-491d-9fcf-aa8f7440aab8").  
  - Outputs: Forwards data to chatbot agent node.  
  - Failures: HTTP errors, malformed payloads.

- **Qdrant Vector Store (retrieve-as-tool)**  
  - Type: Qdrant vector store retrieval  
  - Role: Performs semantic search in "docragtestkb" collection to find relevant document chunks based on user query embeddings.  
  - Credential: Qdrant API.  
  - Parameters: Tool description used by agent for query context.  
  - Inputs: Query embeddings from Embeddings node.

- **Embeddings Mistral Cloud1**  
  - Type: Embeddings Node (Mistral AI)  
  - Role: Converts incoming user query text into vector embeddings for retrieval.  
  - Credential: Mistral Cloud API.

- **OpenAI Chat Model**  
  - Type: GPT-4 Langchain Chat Model  
  - Role: Generates the final chatbot response text based on retrieved documents and memory context.  
  - Parameters: Model "gpt-4.1-mini", temperature 0.5, presence and frequency penalties set to 1 for focused responses.  
  - Credential: OpenAI API key.  
  - Version: 1.2

- **Simple Memory**  
  - Type: Langchain Memory Buffer Window  
  - Role: Maintains conversation context with a sliding window of 10 recent messages for coherent multi-turn chat.  
  - Parameter: sessionIdType set to custom key (likely user/session ID).  
  - Connected to the agent node.

- **website chat agent**  
  - Type: Langchain Agent  
  - Role: Orchestrates the full chat response flow using:  
    - Input query text  
    - Vector search tool (Qdrant)  
    - Language model (GPT-4)  
    - Memory buffer for context  
  - Parameters:  
    - System message defines brand-specific behavioral rules, tone, and response style.  
    - Enforces answer sourcing strictly from the "chatdbtai" vector store.  
    - Provides fallback replies if no relevant data found.  
  - Version: 1.9  
  - Inputs: Query text from chat trigger or webhook, embeddings, memory.  
  - Outputs: Final chat response.

- **Sticky Notes**  
  - Explain chatbot entry modes (webhook, embedded chat) and overall chat agent purpose.

---

## 3. Summary Table

| Node Name                          | Node Type                               | Functional Role                              | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                                                                            |
|-----------------------------------|---------------------------------------|----------------------------------------------|--------------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                        | Starts knowledge base ingestion               | None                                 | Google Drive(brand related data for chatbot) |                                                                                                                        |
| Google Drive(brand related data for chatbot) | Google Drive (Folder Search)          | Lists documents in Google Drive folder         | When clicking ‘Execute workflow’     | Loop Over Items1                     | ## load folder with all need for website chatbot                                                                       |
| Loop Over Items1                  | SplitInBatches                       | Processes files one by one                      | Google Drive(brand related data for chatbot) | Set metadata (main branch)           |                                                                                                                        |
| Set metadata                     | Set                                  | Extracts file metadata                          | Loop Over Items1                     | Google Drive(load file)              | ## load individual files                                                                                                |
| Google Drive(load file)           | Google Drive (File Download)          | Downloads file content                          | Set metadata                        | Mistral Upload                      |                                                                                                                        |
| Mistral Upload                   | HTTP Request (POST multipart-form-data) | Uploads file to Mistral OCR API                  | Google Drive(load file)              | Mistral Signed URL                  | ## MISTRAL OCR 1. UPLOAD FILE 2. GET SIGNED URL 3. GET EXTRACT DATA AFTER USING MISTRAL OCR [OCR Guide](https://mistral.ai/news/mistral-ocr) |
| Mistral Signed URL              | HTTP Request (GET)                   | Retrieves signed URL for OCR file access        | Mistral Upload                     | Mistral DOC OCR                    |                                                                                                                        |
| Mistral DOC OCR                 | HTTP Request (POST JSON)              | Requests OCR processing on uploaded file        | Mistral Signed URL                 | If2 (main branch)                  |                                                                                                                        |
| If2                            | If                                   | Checks if OCR skipped data                        | Mistral DOC OCR                   | prepare for chunking / Wait         |                                                                                                                        |
| Wait                          | Wait                                 | Delays workflow for retry if OCR incomplete      | If2                              | Google Drive(load file)              |                                                                                                                        |
| prepare for chunking             | Set                                  | Prepares OCR data for chunking                    | If2                              | Code(convert to chunks for loading into vector db) | ## convert ocr output into chunks for loading into vector database                                                      |
| Code(convert to chunks for loading into vector db) | Code (JavaScript)                    | Splits OCR text into chunks for embedding        | prepare for chunking               | Qdrant Vector Store1               |                                                                                                                        |
| Character Text Splitter          | Text Splitter (character-based)       | (Unused/alternative) splits text into chunks     | None                              | Default Data Loader                |                                                                                                                        |
| Default Data Loader              | Document Data Loader                   | Loads chunked JSON into document format           | Code                             | Embeddings Mistral Cloud          |                                                                                                                        |
| Embeddings Mistral Cloud        | Embeddings Node                       | Creates vector embeddings from document chunks   | Default Data Loader               | Qdrant Vector Store1               |                                                                                                                        |
| Qdrant Vector Store1             | Vector Store (Qdrant)                  | Inserts embeddings into vector database            | Embeddings Mistral Cloud          | Loop Over Items1 (second branch)    | ## qdrant vector store                                                                                                 |
| When chat message received       | Langchain Chat Trigger (webhook mode) | Listens for chat messages via webhook            | None                              | website chat agent                | ## embedded chat                                                                                                       |
| Webhook                        | n8n Webhook (HTTP POST listener)       | Alternative chat message entry point              | None                              | website chat agent                | ## chatbot from webhook                                                                                                |
| Embeddings Mistral Cloud1       | Embeddings Node                       | Embeds user query text into vector space          | When chat message received / Webhook | Qdrant Vector Store               |                                                                                                                        |
| Qdrant Vector Store             | Vector Store (Qdrant)                  | Retrieves relevant document vectors for query    | Embeddings Mistral Cloud1         | website chat agent                |                                                                                                                        |
| OpenAI Chat Model               | GPT-4 Langchain Chat Model             | Generates chatbot text responses                   | website chat agent (ai_languageModel) | website chat agent (ai_languageModel) |                                                                                                                        |
| Simple Memory                  | Langchain Memory Buffer Window          | Maintains conversational context                   | website chat agent (ai_memory)    | website chat agent (ai_memory)    |                                                                                                                        |
| website chat agent             | Langchain Agent                        | Orchestrates chat query, retrieval, and response | When chat message received / Webhook, Qdrant Vector Store, OpenAI Chat Model, Simple Memory | Final chatbot response            | ## website chat agent \nreply to user query either from embedded chat or webhook                                         |
| Sticky Note4                   | Sticky Note                          | OCR flow guide and link                             | None                              | None                              | ## MISTRAL OCR\n [OCR Guide](https://mistral.ai/news/mistral-ocr)\n1. UPLOAD FILE\n2. GET SIGNED URL\n3. GET EXTRACT DATA AFTER USING MISTRAL OCR |
| Sticky Note                    | Sticky Note                          | Folder load explanation                             | None                              | None                              | ## load folder with all need for website chatbot                                                                       |
| Sticky Note1                   | Sticky Note                          | Individual file load explanation                    | None                              | None                              | ## load individual files                                                                                                |
| Sticky Note2                   | Sticky Note                          | Chunking explanation                                | None                              | None                              | ## convert ocr output into chunks for loading into vector database                                                      |
| Sticky Note3                   | Sticky Note                          | Qdrant vector store explanation                      | None                              | None                              | ## qdrant vector store                                                                                                 |
| Sticky Note5                   | Sticky Note                          | Website chat agent high-level explanation           | None                              | None                              | ## website chat agent \n\nreply to user query either from embedded chat or webhook                                       |
| Sticky Note6                   | Sticky Note                          | Chatbot webhook entry explanation                    | None                              | None                              | ## chatbot from webhook                                                                                                |
| Sticky Note7                   | Sticky Note                          | Embedded chat entry explanation                       | None                              | None                              | ## embedded chat                                                                                                       |
| Sticky Note8                   | Sticky Note                          | Full workflow summary and tools used                  | None                              | None                              | ## Website Chat + Document Intelligence Workflow\n\nEnables a **website chatbot** to answer user queries using **documents stored in Google Drive**.  \nThe workflow automatically **fetches, OCRs, chunks, and embeds** documents with **Mistral AI**, then stores them in **Qdrant** for vector search — allowing the chatbot to deliver accurate, context-aware replies in real time.\n\n**Flow:** Google Drive → OCR + Chunk → Embed → Qdrant Vector Store → Chat Query → Intelligent Response  \n**Core Tools:** Mistral AI · Qdrant · Supabase · OpenAI |

---

## 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node:**  
   - Node: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: To start document ingestion process manually.

2. **Add Google Drive folder listing node:**  
   - Node: Google Drive (File/Folder Search)  
   - Name: "Google Drive(brand related data for chatbot)"  
   - Parameters:  
     - Filter by Folder ID: "1o3DK9Ceka5Lqb8irvFSfEeB8SVGG_OL7"  
     - Return fields: id, name, webViewLink, mimeType, etc.  
   - Credential: OAuth2 Google Drive account with access to folder.

3. **Add SplitInBatches node:**  
   - Node: SplitInBatches  
   - Name: "Loop Over Items1"  
   - Purpose: Batch process files one by one from folder listing.

4. **Add Set node to extract file metadata:**  
   - Node: Set  
   - Name: "Set metadata"  
   - Assign fields: file_id, file_type, file_title, file_url, last_modified_date from incoming JSON.

5. **Add Google Drive download node:**  
   - Node: Google Drive (File Download)  
   - Name: "Google Drive(load file)"  
   - Parameters: Use file_id from Set metadata node.  
   - Credential: Same Google Drive OAuth2.

6. **Add HTTP Request node to upload file to Mistral OCR:**  
   - Node: HTTP Request (POST multipart-form-data)  
   - Name: "Mistral Upload"  
   - URL: https://api.mistral.ai/v1/files  
   - Body Parameters: purpose = "ocr", file = binary from Google Drive(load file)  
   - Credential: Mistral Cloud API key.

7. **Add HTTP Request node to get signed URL:**  
   - Node: HTTP Request (GET)  
   - Name: "Mistral Signed URL"  
   - URL: Construct as https://api.mistral.ai/v1/files/{{file_id}}/url  
   - Query: expiry=24  
   - Credential: Mistral Cloud API key.

8. **Add HTTP Request node for OCR processing:**  
   - Node: HTTP Request (POST JSON)  
   - Name: "Mistral DOC OCR"  
   - URL: https://api.mistral.ai/v1/ocr  
   - JSON Body:  
     ```json
     {
       "model": "mistral-ocr-latest",
       "document": {
         "type": "document_url",
         "document_url": "{{signed_url}}"
       },
       "include_image_base64": true
     }
     ```  
   - Credential: Mistral Cloud API key.  
   - Set OnError to continue with retries enabled.

9. **Add If node to check for skipped OCR data:**  
   - Node: If  
   - Name: "If2"  
   - Condition: Check if `data[0].parseJson().skipped` does not exist (means OCR successful).

10. **Add Wait node for retry delay:**  
    - Node: Wait  
    - Name: "Wait"  
    - Connect from If2 fail branch to delay before retrying Google Drive(load file).

11. **Add Set node to prepare OCR data for chunking:**  
    - Node: Set  
    - Name: "prepare for chunking"  
    - Assign variables:  
      - Document name = OCR data source field  
      - Document data = OCR blocks array  
      - source = Google Drive file id

12. **Add Code node to chunk OCR data:**  
    - Node: Code (JavaScript)  
    - Name: "Code(convert to chunks for loading into vector db)"  
    - Use provided JS code to split large OCR text into ~1000 char chunks with metadata.

13. **Add Default Data Loader node:**  
    - Node: Document Default Data Loader  
    - Name: "Default Data Loader"  
    - Input: chunked JSON strings from Code node  
    - Map metadata fields accordingly.

14. **Add Embeddings node for text chunks:**  
    - Node: Embeddings Mistral Cloud  
    - Name: "Embeddings Mistral Cloud"  
    - Credential: Mistral Cloud API key.

15. **Add Qdrant Vector Store node for insertion:**  
    - Node: Qdrant Vector Store  
    - Name: "Qdrant Vector Store1"  
    - Mode: Insert  
    - Collection: "docragtestkb"  
    - Credential: Qdrant API key  
    - Batch size: 200

16. **Connect Qdrant Vector Store1 back to Loop Over Items1 to process next file.**

---

**Chatbot Query Handling Setup:**

17. **Add Langchain Chat Trigger node:**  
    - Node: When chat message received  
    - Mode: webhook, public access enabled  
    - Response mode: lastNode

18. **Add Webhook node:**  
    - Node: Webhook  
    - HTTP method: POST  
    - Path: unique webhook path for external chat messages.

19. **Add Embeddings node for query text:**  
    - Node: Embeddings Mistral Cloud1  
    - Credential: Mistral Cloud API key  
    - Input: query text from chat trigger or webhook.

20. **Add Qdrant Vector Store node for retrieval:**  
    - Node: Qdrant Vector Store  
    - Mode: retrieve-as-tool  
    - Collection: "docragtestkb"  
    - Credential: Qdrant API key  
    - Tool description: "use this data answer query"

21. **Add OpenAI Chat Model node:**  
    - Node: OpenAI Chat Model  
    - Model: "gpt-4.1-mini"  
    - Temperature: 0.5  
    - PresencePenalty: 1  
    - FrequencyPenalty: 1  
    - Credential: OpenAI API key.

22. **Add Simple Memory node:**  
    - Node: Simple Memory (MemoryBufferWindow)  
    - sessionIdType: customKey  
    - contextWindowLength: 10

23. **Add Langchain Agent node:**  
    - Node: website chat agent  
    - Parameters:  
      - Text: user query  
      - System message: detailed brand-specific instructions (as per original)  
      - Prompt type: define  
      - Connect ai_languageModel to OpenAI Chat Model node  
      - Connect ai_tool to Qdrant Vector Store node  
      - Connect ai_memory to Simple Memory node  
    - Inputs: From chat trigger and webhook.

24. **Connect “When chat message received” and “Webhook” nodes to “website chat agent”.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| MISTRAL OCR guide: Upload file → Get signed URL → Extract data using Mistral OCR                                                          | https://mistral.ai/news/mistral-ocr                    |
| The workflow uses a Google Drive folder as a source of knowledge documents for the chatbot knowledge base.                                | Google Drive folder ID: 1o3DK9Ceka5Lqb8irvFSfEeB8SVGG_OL7 |
| Qdrant vector store collection name: "docragtestkb"                                                                                       | Qdrant vector database                                 |
| Chatbot system prompt enforces brand tone, strict vector search answers, user-friendly replies, and fallback messages                     | Embedded in “website chat agent” node parameters      |
| Core AI tools integrated: Mistral AI (OCR & embeddings), Qdrant (vector DB), OpenAI GPT-4 (chat generation), Google Drive (document storage) |                                                       |
| Workflow supports both embedded chat messages and webhook-triggered queries                                                                | Nodes: “When chat message received”, “Webhook”        |

---

**Disclaimer:** The provided text exclusively derives from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---