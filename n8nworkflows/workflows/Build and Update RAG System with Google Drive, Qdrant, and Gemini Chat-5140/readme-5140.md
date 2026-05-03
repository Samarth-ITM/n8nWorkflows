Build and Update RAG System with Google Drive, Qdrant, and Gemini Chat

https://n8nworkflows.xyz/workflows/build-and-update-rag-system-with-google-drive--qdrant--and-gemini-chat-5140


# Build and Update RAG System with Google Drive, Qdrant, and Gemini Chat

---

### 1. Workflow Overview

This workflow automates the creation, updating, and querying of a Retrieval-Augmented Generation (RAG) system using Google Drive as the document source, Qdrant as the vector store, and Google Gemini Chat for conversational AI interaction.

**Target Use Cases:**  
- Incremental or full updates of document embeddings in a vector database to keep a RAG system current with Google Drive documents.  
- Vectorization of documents and storage in Qdrant for semantic search.  
- Handling chat queries using a retrieval-augmented question answering chain powered by Google Gemini.

**Logical Blocks:**  
- **1.1 Initialization and Collection Setup:** Manual trigger, Qdrant collection creation, and clearing existing data for fresh indexing.  
- **1.2 Document Retrieval and Processing:** Fetching files from Google Drive, downloading, splitting, and embedding documents.  
- **1.3 Vector Store Insertion and Update:** Inserting or updating document embeddings into Qdrant vector store.  
- **1.4 Incremental Update Handling:** Targeted updates and deletions of single documents in the vector store based on file ID.  
- **1.5 Chat Query Handling:** Triggering on chat messages, retrieving relevant vectors, and responding using Google Gemini chat model.  
- **1.6 Control Flow and Orchestration:** Looping over files, waiting for processing, and orchestrating the sequence of operations.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Collection Setup

- **Overview:**  
  This block initializes the RAG system by creating a Qdrant collection and clearing any existing data to prepare for a fresh ingestion of documents.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Create collection  
  - Clear collection  
  - Get files  
  - Loop Over Items  
  - Sticky Note3 (Instruction)  
  - Sticky Note4 (Instruction)  
  - Sticky Note (Instruction)  
  - Sticky Note2 (Overview)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution to start the workflow.  
    - Configuration: No parameters; manual trigger node.  
    - Connections: Outputs to `Clear collection`.  
    - Edge Cases: None.  
      
  - **Create collection**  
    - Type: HTTP Request  
    - Role: Creates a Qdrant collection via REST API.  
    - Configuration:  
      - Method: PUT  
      - URL: `http://QDRANTURL/collections/COLLECTION` (placeholders to be replaced)  
      - Body: JSON specifying vector size (1536), distance metric (Cosine), shard number, replication factor, and write consistency.  
      - Authentication: HTTP header auth with a Qdrant API token.  
    - Input: None (triggered manually or by other nodes).  
    - Output: None specified.  
    - Edge Cases: API connection errors, authentication failures, improper URL or collection name.  
    - Sticky Note3 highlights the need to change QDRANTURL and COLLECTION placeholders.  

  - **Clear collection**  
    - Type: HTTP Request  
    - Role: Deletes all points in the Qdrant collection to empty it before re-indexing.  
    - Configuration:  
      - Method: POST  
      - URL: `http://QDRANTURL/collections/COLLECTION/points/delete`  
      - Body: JSON with empty filter (deletes all points)  
      - Auth: Same as above.  
    - Input: Triggered by manual trigger node.  
    - Output: Outputs to `Get files` node.  
    - Edge Cases: API errors, failure to clear collection.  

  - **Get files**  
    - Type: Google Drive  
    - Role: Lists files from a specific Google Drive folder for processing.  
    - Configuration:  
      - Resource: fileFolder  
      - Filter: Drive ID "My Drive", Folder ID set to a specific folder (Test Negozio)  
      - Auth: Google Drive OAuth2  
    - Input: Triggered after clearing collection.  
    - Output: Outputs to `Loop Over Items`.  
    - Edge Cases: Folder inaccessible, permission issues, empty folder.  

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes files one by one or in batches for downstream processing.  
    - Configuration: Default batch size.  
    - Input: Receives array of files.  
    - Output: Has two outputs; first is empty, second connects to `Download files`.  
    - Edge Cases: Large number of files might cause processing delays.  

  - **Sticky Notes (2,3,4)**  
    - Provide contextual instructions to change URLs, collection names, and file IDs.  
    - Sticky Note2 explains the workflow purpose.  
    - Sticky Notes 3 and 4 provide step-by-step guidance on setup.  

#### 1.2 Document Retrieval and Processing

- **Overview:**  
  Downloads files from Google Drive, converts them to plain text, splits large documents into smaller chunks for embedding.

- **Nodes Involved:**  
  - Download files  
  - Recursive Character Text Splitter  
  - Default Data Loader1  
  - Embeddings OpenAI1  

- **Node Details:**

  - **Download files**  
    - Type: Google Drive  
    - Role: Downloads the file specified by file ID from Google Drive converting Google Docs to plain text.  
    - Configuration:  
      - Operation: Download  
      - File ID from loop item `{{ $json.id }}`  
      - Conversion: Google Docs to text/plain  
      - Auth: Google Drive OAuth2  
    - Input: From `Loop Over Items`  
    - Output: To `Default Data Loader1` through `Recursive Character Text Splitter`  
    - Edge Cases: File not found, permission errors, conversion failures.  

  - **Recursive Character Text Splitter**  
    - Type: Text Splitter (Recursive Character)  
    - Role: Splits long text documents into chunks of 500 characters with 50 character overlap to optimize chunk size for embeddings.  
    - Configuration: Chunk size 500, overlap 50.  
    - Input: Binary text from `Download files` node.  
    - Output: To `Default Data Loader1`.  
    - Edge Cases: Very small documents may produce single chunk; malformed text input.  

  - **Default Data Loader1**  
    - Type: Document Loader  
    - Role: Prepares data with metadata for embedding processing.  
    - Configuration:  
      - Data type: binary  
      - Metadata: attaches file_id and file_name from `Download files` node.  
    - Input: From `Recursive Character Text Splitter`.  
    - Output: To `Embeddings OpenAI1`.  
    - Edge Cases: Missing metadata, binary data corruption.  

  - **Embeddings OpenAI1**  
    - Type: OpenAI Embeddings node (LangChain)  
    - Role: Converts document chunks to vector embeddings using OpenAI embeddings API.  
    - Configuration: Uses default OpenAI embeddings model and credentials.  
    - Input: From `Default Data Loader1`.  
    - Output: To `Qdrant Vector Store`.  
    - Edge Cases: API rate limits, invalid credentials, network errors.  

#### 1.3 Vector Store Insertion and Update

- **Overview:**  
  Inserts the generated embeddings into the Qdrant collection to build the vector store for semantic search.

- **Nodes Involved:**  
  - Qdrant Vector Store  
  - Wait  

- **Node Details:**

  - **Qdrant Vector Store**  
    - Type: Vector Store node (Qdrant with LangChain)  
    - Role: Inserts new vectors into specified Qdrant collection `negozio-emporio-verde`.  
    - Configuration:  
      - Mode: insert  
      - Collection: `negozio-emporio-verde` (selectable from list)  
    - Input: From `Embeddings OpenAI1`.  
    - Output: To `Wait` node.  
    - Credentials: Qdrant API with Hetzner endpoint.  
    - Edge Cases: Connection issues, insertion failures, collection not found.  

  - **Wait**  
    - Type: Wait node  
    - Role: Introduces delay or throttling to avoid API rate limits or to manage flow control.  
    - Configuration: Default parameters (no custom time set), triggered by webhook.  
    - Input: From `Qdrant Vector Store`.  
    - Output: Loops back to `Loop Over Items` to process next file.  
    - Edge Cases: Misconfiguration might cause indefinite waiting or workflow hang.  

#### 1.4 Incremental Update Handling

- **Overview:**  
  Supports updating or deleting individual files in the Qdrant vector store based on a specified file ID, enabling incremental updates without full reindexing.

- **Nodes Involved:**  
  - Edit Fields3  
  - Download file  
  - Delete single file  
  - Default Data Loader  
  - Recursive Character Text Splitter1  
  - Embeddings OpenAI2  
  - Update single file  

- **Node Details:**

  - **Edit Fields3**  
    - Type: Set node  
    - Role: Sets the `file_id` variable to a specified Google Drive file ID (`DRIVEFILE_ID`) for update.  
    - Configuration: Assign string value to `file_id`.  
    - Input: Triggered externally or by manual start (not explicitly connected here).  
    - Output: To `Download file` and `Delete single file`.  
    - Edge Cases: Invalid file ID input will cause downstream failures.  

  - **Download file**  
    - Type: Google Drive  
    - Role: Downloads the specific file by `file_id`.  
    - Configuration:  
      - Operation: Download  
      - File ID: from `Edit Fields3` (`{{$json.file_id}}`)  
      - Conversion: Docs to plain text  
      - Auth: Google Drive OAuth2  
    - Input: From `Edit Fields3`.  
    - Output: To `Update single file` after processing.  
    - Edge Cases: File not found, permission denied.  

  - **Delete single file**  
    - Type: HTTP Request  
    - Role: Deletes all points in Qdrant associated with the specified `file_id` before re-inserting updated embeddings.  
    - Configuration:  
      - Method: POST  
      - URL: `http://QDRANTURL/collections/COLLECTION/points/delete`  
      - Body: JSON filter matching metadata.file_id = `{{$json.file_id}}`  
      - Auth: Qdrant API token.  
    - Input: From `Edit Fields3`.  
    - Output: None explicitly connected but normally precedes update insertion.  
    - Edge Cases: API errors, mismatched file_id filter causing incomplete deletion.  

  - **Default Data Loader**  
    - Type: Document Loader  
    - Role: Loads the downloaded file data and attaches metadata for embedding.  
    - Configuration:  
      - Data type: binary  
      - Metadata: `file_id` and `file_name` from downloaded file.  
    - Input: From `Recursive Character Text Splitter1` (see below).  
    - Output: To `Embeddings OpenAI2`.  
    - Edge Cases: Metadata mismatch, missing binary data.  

  - **Recursive Character Text Splitter1**  
    - Type: Text splitter  
    - Role: Splits the downloaded file text into chunks for embedding.  
    - Configuration: Chunk size 500, overlap 50.  
    - Input: From `Download file`.  
    - Output: To `Default Data Loader`.  
    - Edge Cases: Small files produce minimal chunks.  

  - **Embeddings OpenAI2**  
    - Type: OpenAI Embeddings node  
    - Role: Embeds the chunks from the single file update.  
    - Configuration: Default OpenAI embedding model and credentials.  
    - Input: From `Default Data Loader`.  
    - Output: To `Update single file`.  
    - Edge Cases: API rate limit, authentication errors.  

  - **Update single file**  
    - Type: Vector Store Qdrant node  
    - Role: Inserts updated embeddings for the single file into Qdrant collection `negozio-emporio-verde`.  
    - Configuration: Insert mode  
    - Input: From `Embeddings OpenAI2`.  
    - Output: None explicitly connected.  
    - Edge Cases: Insert failures, collection not found.  

  - **Sticky Note**  
    - Explains the need to set the Google Drive file ID for incremental update.  

#### 1.5 Chat Query Handling

- **Overview:**  
  Handles incoming chat messages, retrieves relevant documents from Qdrant vector store, and generates answer responses using Google Gemini chat model.

- **Nodes Involved:**  
  - When chat message received  
  - Question and Answer Chain  
  - Google Gemini Chat Model  
  - Vector Store Retriever  
  - Qdrant Vector Store1  
  - Embeddings OpenAI  
  - Sticky Note1 (Instruction)

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Webhook trigger for incoming chat messages.  
    - Configuration: Default.  
    - Output: To `Question and Answer Chain`.  
    - Edge Cases: Webhook connectivity, malformed messages.  

  - **Question and Answer Chain**  
    - Type: LangChain Retrieval QA Chain  
    - Role: Coordinates retrieval of relevant documents and generation of answers.  
    - Configuration: Default.  
    - Input: From chat trigger and retriever.  
    - Output: Chat response.  
    - Edge Cases: Retrieval failures, response timeout.  

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Language model for response generation.  
    - Configuration: Model set to `models/gemini-1.5-flash`.  
    - Credentials: Google PaLM API credentials.  
    - Input: From QA Chain.  
    - Output: To QA Chain.  
    - Edge Cases: API limits, authentication errors.  

  - **Vector Store Retriever**  
    - Type: LangChain Retriever Vector Store  
    - Role: Retrieves relevant vectors from Qdrant based on chat query.  
    - Input: From `Qdrant Vector Store1` embeddings.  
    - Output: To QA Chain.  
    - Edge Cases: Retrieval latency, empty results.  

  - **Qdrant Vector Store1**  
    - Type: Vector Store (Qdrant)  
    - Role: Provides vector embeddings from `ocr_mistral_test` collection for retrieval.  
    - Credentials: Same Qdrant API.  
    - Input: From `Embeddings OpenAI`.  
    - Output: To `Vector Store Retriever`.  
    - Edge Cases: Collection existence, connection errors.  

  - **Embeddings OpenAI**  
    - Type: OpenAI Embeddings node  
    - Role: Embeds chat queries for vector retrieval.  
    - Input: From chat input.  
    - Output: To `Qdrant Vector Store1`.  
    - Edge Cases: API limits.  

  - **Sticky Note1**  
    - Instruction to test the RAG system after setup.  

---

### 3. Summary Table

| Node Name                     | Node Type                                    | Functional Role                                     | Input Node(s)               | Output Node(s)              | Sticky Note                          |
|-------------------------------|----------------------------------------------|----------------------------------------------------|-----------------------------|-----------------------------|------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                               | Starts the workflow manually                        |                             | Clear collection            |                                    |
| Create collection             | HTTP Request                                | Creates Qdrant collection                           |                             |                             | # STEP 1 - Change QDRANTURL, COLLECTION |
| Clear collection             | HTTP Request                                | Clears all points in Qdrant collection             | When clicking ‘Test workflow’ | Get files                   |                                    |
| Get files                    | Google Drive                                | Lists files from Google Drive folder                | Clear collection            | Loop Over Items             |                                    |
| Loop Over Items              | SplitInBatches                              | Processes files in batches                           | Get files                   | Download files (2nd output) |                                    |
| Download files               | Google Drive                                | Downloads each file as plain text                    | Loop Over Items             | Recursive Character Text Splitter |                                    |
| Recursive Character Text Splitter | Text Splitter (Recursive)               | Splits large text into chunks for embedding         | Download files              | Default Data Loader1        |                                    |
| Default Data Loader1         | Document Loader                             | Prepares document chunks with metadata               | Recursive Character Text Splitter | Embeddings OpenAI1          |                                    |
| Embeddings OpenAI1           | OpenAI Embeddings (LangChain)               | Converts text chunks to embeddings                    | Default Data Loader1        | Qdrant Vector Store         |                                    |
| Qdrant Vector Store          | Vector Store (Qdrant)                       | Inserts embeddings into Qdrant collection            | Embeddings OpenAI1          | Wait                       |                                    |
| Wait                        | Wait                                        | Controls flow timing and rate limit                   | Qdrant Vector Store         | Loop Over Items             |                                    |
| Edit Fields3                | Set                                          | Sets the file_id for incremental update               |                             | Download file, Delete single file | # STEP 3 - Set Google Drive File ID |
| Download file               | Google Drive                                | Downloads a specific file for incremental update      | Edit Fields3                | Recursive Character Text Splitter1 |                                    |
| Recursive Character Text Splitter1 | Text Splitter (Recursive)               | Splits single file text into chunks                    | Download file               | Default Data Loader         |                                    |
| Default Data Loader         | Document Loader                             | Prepares file chunks with metadata                     | Recursive Character Text Splitter1 | Embeddings OpenAI2          |                                    |
| Embeddings OpenAI2          | OpenAI Embeddings (LangChain)               | Embeds chunks for single file update                   | Default Data Loader         | Update single file          |                                    |
| Update single file          | Vector Store (Qdrant)                       | Inserts updated embeddings for single file             | Embeddings OpenAI2          |                             |                                    |
| Delete single file          | HTTP Request                                | Deletes vectors for specific file_id from Qdrant       | Edit Fields3                |                             |                                    |
| When chat message received  | LangChain Chat Trigger                      | Triggered by incoming chat messages                    |                             | Question and Answer Chain   |                                    |
| Question and Answer Chain   | LangChain Retrieval QA Chain                | Handles retrieval and answer generation                | When chat message received, Vector Store Retriever, Google Gemini Chat Model |                             |                                    |
| Google Gemini Chat Model    | LangChain LM Chat Google Gemini             | Generates chat responses using Google Gemini model     | Question and Answer Chain   |                             |                                    |
| Vector Store Retriever      | LangChain Retriever Vector Store            | Retrieves relevant vectors from Qdrant                 | Qdrant Vector Store1        | Question and Answer Chain   |                                    |
| Qdrant Vector Store1        | Vector Store (Qdrant)                       | Provides vectors from `ocr_mistral_test` collection    | Embeddings OpenAI           | Vector Store Retriever      |                                    |
| Embeddings OpenAI           | OpenAI Embeddings (LangChain)               | Embeds chat queries for retrieval                       |                             | Qdrant Vector Store1        |                                    |
| Sticky Note2               | Sticky Note                                 | Workflow overview and purpose                           |                             |                             | # Enables full or incremental updates to documents in RAG system using Qdrant |
| Sticky Note3               | Sticky Note                                 | Step 1 instructions for collection creation            |                             |                             | # STEP 1 - Change QDRANTURL and COLLECTION |
| Sticky Note4               | Sticky Note                                 | Step 2 instructions for document vectorization          |                             |                             | # STEP 2 - Change QDRANTURL and COLLECTION |
| Sticky Note                | Sticky Note                                 | Step 3 instructions for setting Google Drive file ID    |                             |                             | # STEP 3 - Set Google Drive File ID |
| Sticky Note1               | Sticky Note                                 | Step 4 instructions for testing the RAG system          |                             |                             | ## STEP 4 - Test the RAG              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create HTTP Request Node to Create Qdrant Collection**  
   - Name: `Create collection`  
   - Type: HTTP Request  
   - Method: PUT  
   - URL: `http://QDRANTURL/collections/COLLECTION` (replace placeholders)  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "vectors": {
         "size": 1536,
         "distance": "Cosine"
       },
       "shard_number": 1,
       "replication_factor": 1,
       "write_consistency_factor": 1
     }
     ```  
   - Authentication: HTTP Header Auth with Qdrant API key  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node to Clear Qdrant Collection**  
   - Name: `Clear collection`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `http://QDRANTURL/collections/COLLECTION/points/delete`  
   - Body: `{ "filter": {} }`  
   - Authentication: Same as above  
   - Connect output of Manual Trigger to this node.

4. **Create Google Drive Node to Get Files**  
   - Name: `Get files`  
   - Type: Google Drive  
   - Operation: List files/folders  
   - Filter: Drive ID = "My Drive", Folder ID = target folder ID  
   - Authentication: Google Drive OAuth2 credentials  
   - Connect output of `Clear collection` to this node.

5. **Create SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Type: SplitInBatches  
   - Default batch size  
   - Connect output of `Get files` to this node.

6. **Create Google Drive Node to Download Files**  
   - Name: `Download files`  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: `={{ $json.id }}` (from loop)  
   - Conversion: Google Docs to `text/plain`  
   - Authentication: Google Drive OAuth2 credentials  
   - Connect second output of `Loop Over Items` to this node.

7. **Create Recursive Character Text Splitter Node**  
   - Name: `Recursive Character Text Splitter`  
   - Type: Text Splitter (Recursive)  
   - Parameters: Chunk size 500, Chunk overlap 50  
   - Connect output of `Download files` to this node.

8. **Create Document Default Data Loader Node**  
   - Name: `Default Data Loader1`  
   - Type: Document Data Loader  
   - Data Type: Binary (specific field)  
   - Metadata: Attach `file_id` and `file_name` from `Download files` node  
   - Connect output of `Recursive Character Text Splitter` to this node.

9. **Create OpenAI Embeddings Node**  
   - Name: `Embeddings OpenAI1`  
   - Type: LangChain OpenAI Embeddings  
   - Credentials: OpenAI API  
   - Connect output of `Default Data Loader1` to this node.

10. **Create Qdrant Vector Store Node**  
    - Name: `Qdrant Vector Store`  
    - Type: LangChain Vector Store Qdrant  
    - Mode: Insert  
    - Collection: `negozio-emporio-verde`  
    - Credentials: Qdrant API  
    - Connect output of `Embeddings OpenAI1` to this node.

11. **Create Wait Node**  
    - Name: `Wait`  
    - Type: Wait  
    - Default settings  
    - Connect output of `Qdrant Vector Store` to this node.  
    - Connect output of `Wait` back to `Loop Over Items` to iterate.

12. **Create Set Node for Incremental Update**  
    - Name: `Edit Fields3`  
    - Type: Set  
    - Set `file_id` to the target Google Drive file ID string (placeholder `DRIVEFILE_ID`).  
    - This node initiates incremental updates.  

13. **Create Google Drive Node to Download Single File**  
    - Name: `Download file`  
    - Type: Google Drive  
    - Operation: Download  
    - File ID: `={{ $json.file_id }}` from `Edit Fields3`  
    - Conversion: Docs to `text/plain`  
    - Connect output of `Edit Fields3` to this node.

14. **Create HTTP Request Node to Delete Single File from Qdrant**  
    - Name: `Delete single file`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `http://QDRANTURL/collections/COLLECTION/points/delete`  
    - Body: JSON filter to match `metadata.file_id` = `{{$json.file_id}}`  
    - Authentication: Qdrant API  
    - Connect output of `Edit Fields3` to this node.

15. **Create Recursive Character Text Splitter Node for Single File**  
    - Name: `Recursive Character Text Splitter1`  
    - Type: Text Splitter (Recursive)  
    - Chunk size 500, overlap 50  
    - Connect output of `Download file` to this node.

16. **Create Default Data Loader for Single File**  
    - Name: `Default Data Loader`  
    - Type: Document Data Loader  
    - Data type: Binary (specific field)  
    - Metadata: `file_id` and file name from `Download file` binary data  
    - Connect output of `Recursive Character Text Splitter1` to this node.

17. **Create Embeddings OpenAI Node for Single File**  
    - Name: `Embeddings OpenAI2`  
    - Type: OpenAI Embeddings (LangChain)  
    - Credentials: OpenAI API  
    - Connect output of `Default Data Loader` to this node.

18. **Create Vector Store Update Node for Single File**  
    - Name: `Update single file`  
    - Type: LangChain Vector Store Qdrant  
    - Mode: Insert  
    - Collection: `negozio-emporio-verde`  
    - Credentials: Qdrant API  
    - Connect output of `Embeddings OpenAI2` to this node.

19. **Create Chat Trigger Node**  
    - Name: `When chat message received`  
    - Type: LangChain Chat Trigger  
    - Webhook enabled  
    - No parameters.

20. **Create Question and Answer Chain Node**  
    - Name: `Question and Answer Chain`  
    - Type: LangChain Retrieval QA Chain  
    - Connect output of Chat Trigger and Vector Store Retriever to this node.

21. **Create Google Gemini Chat Model Node**  
    - Name: `Google Gemini Chat Model`  
    - Type: LangChain LM Chat Google Gemini  
    - Model: `models/gemini-1.5-flash`  
    - Credentials: Google PaLM API  
    - Connect to QA Chain node.

22. **Create Vector Store Retriever Node**  
    - Name: `Vector Store Retriever`  
    - Type: LangChain Retriever Vector Store  
    - Connect to QA Chain node.

23. **Create Qdrant Vector Store Node for Chat**  
    - Name: `Qdrant Vector Store1`  
    - Type: LangChain Vector Store Qdrant  
    - Collection: `ocr_mistral_test` (or relevant collection)  
    - Credentials: Qdrant API  
    - Connect output of Embeddings OpenAI (for chat) to this node.  
    - Connect output to Vector Store Retriever.

24. **Create Embeddings OpenAI Node for Chat Queries**  
    - Name: `Embeddings OpenAI`  
    - Type: OpenAI Embeddings  
    - Credentials: OpenAI API  
    - Connect chat input to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automates full or incremental updates of a RAG system combining Google Drive, Qdrant, and Gemini Chat. | Overall workflow purpose explained in Sticky Note2.                                            |
| Step 1 requires careful setup of Qdrant API URL and collection name.                                             | See Sticky Note3 for instructions.                                                             |
| Step 2 involves vectorizing documents fetched from Google Drive.                                                | See Sticky Note4 for detailed instructions.                                                    |
| Step 3 requires setting the specific Google Drive file ID for incremental updates.                              | See Sticky Note for guidance.                                                                   |
| Step 4 is for testing the RAG system via chat queries.                                                          | See Sticky Note1 for instructions.                                                             |
| Qdrant API credentials must be valid and have permissions for collection management and point insertion/deletion.| Ensure Qdrant API tokens are valid and network accessible.                                      |
| Google Drive OAuth2 credentials must have access to the target folder and files.                                 | Ensure proper OAuth scopes and permissions.                                                    |
| OpenAI API credentials are required for embeddings generation.                                                  | Use appropriate API keys with quota available.                                                |
| Google PaLM API credentials are required for Google Gemini chat model.                                          | Ensure access to Gemini model and valid billing.                                              |
| Replace all placeholder strings (`QDRANTURL`, `COLLECTION`, `DRIVEFILE_ID`) with real values before execution. | Critical to avoid failures.                                                                    |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow constructed with n8n, adhering strictly to applicable content policies and containing no illegal or offensive material. All manipulated data are legal and publicly accessible.

---