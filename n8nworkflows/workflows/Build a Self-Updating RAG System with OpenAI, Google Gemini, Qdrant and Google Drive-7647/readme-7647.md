Build a Self-Updating RAG System with OpenAI, Google Gemini, Qdrant and Google Drive

https://n8nworkflows.xyz/workflows/build-a-self-updating-rag-system-with-openai--google-gemini--qdrant-and-google-drive-7647


# Build a Self-Updating RAG System with OpenAI, Google Gemini, Qdrant and Google Drive

---
### 1. Workflow Overview

This workflow implements a **Retrieval-Augmented Generation (RAG)** system designed to:

- Automatically ingest, vectorize, and store documents from a specified Google Drive folder into a Qdrant vector database.
- Update the Qdrant collection automatically when files in the Google Drive folder are modified.
- Process user questions by retrieving relevant document vectors from Qdrant and generating AI-based answers using Google Gemini.
- Support manual triggering for setup or debugging.

The workflow is logically divided into four main blocks:

**1.1 Initialization and Collection Setup**  
Manually triggered setup that creates and clears the Qdrant collection to prepare for document ingestion.

**1.2 Document Ingestion and Vectorization**  
Fetches files from Google Drive, downloads them, splits text into chunks, generates embeddings with OpenAI, and inserts/updates these vectors into the Qdrant collection.

**1.3 Automatic Updates on File Changes**  
Listens for Google Drive file update events in a specific folder, deletes outdated vectors by file_id in Qdrant, re-downloads updated files, and re-ingests them.

**1.4 Question-Answering Interaction**  
Handles incoming chat messages, retrieves relevant document chunks from Qdrant, and generates answers using a Google Gemini language model.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Collection Setup

- **Overview:**  
  This block prepares the Qdrant vector database by creating a new collection and clearing any existing data. It is triggered manually to initialize or reset the vector store.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Create collection (HTTP Request)  
  - Clear collection (HTTP Request)  
  - Search files (Google Drive)  
  - Loop Over Items (Split in Batches)  

- **Node Details:**

1. **When clicking ‘Test workflow’**  
   - Type: Manual Trigger  
   - Role: Entry point for manual execution.  
   - Config: No parameters.  
   - Outputs: Triggers "Create collection" and "Clear collection".  
   - Edge cases: Manual execution only; no input validation.

2. **Create collection**  
   - Type: HTTP Request  
   - Role: Creates a Qdrant collection with specific vector size (1536) and cosine distance metric.  
   - Config: PUT request to Qdrant API URL (placeholder "http:///collections/test_sparse"), with JSON body defining vector and sparse vector configurations.  
   - Credentials: HTTP Header Auth for Qdrant API.  
   - Inputs: Triggered by manual trigger node.  
   - Outputs: None downstream directly, but logically sets up the collection.  
   - Edge cases: HTTP errors, incorrect URL, auth failure, malformed JSON.

3. **Clear collection**  
   - Type: HTTP Request  
   - Role: Deletes all points in the Qdrant collection to start fresh.  
   - Config: POST to "http://QDRANTURL/collections/COLLECTION/points/delete" with empty filter.  
   - Credentials: Qdrant API HTTP Header Auth.  
   - Inputs: Triggered after collection creation.  
   - Outputs: Triggers "Search files" node.  
   - Edge cases: HTTP errors, incorrect URL, permission issues.

4. **Search files**  
   - Type: Google Drive  
   - Role: Lists files in a specific Google Drive folder ("Test Negozio").  
   - Config: Filters by folder ID and drive ID.  
   - Credentials: Google Drive OAuth2.  
   - Inputs: Triggered after clearing collection.  
   - Outputs: Triggers "Loop Over Items" to process files in batches.  
   - Edge cases: API rate limits, folder permissions.

5. **Loop Over Items**  
   - Type: Split in Batches  
   - Role: Processes files one by one or in small batches.  
   - Config: Default batch size.  
   - Inputs: Receives file list from "Search files".  
   - Outputs: On batch contains, triggers file download nodes.  
   - Edge cases: Batch size misconfiguration, empty file list.

---

#### 2.2 Document Ingestion and Vectorization

- **Overview:**  
  This block downloads each file, splits its content into chunks, generates OpenAI embeddings, and inserts them into the Qdrant collection with appropriate metadata.

- **Nodes Involved:**  
  - Get files (Google Drive)  
  - Default Data Loader1 (Langchain Document Loader)  
  - Recursive Character Text Splitter (Langchain Text Splitter)  
  - Embeddings OpenAI1 (Langchain Embeddings)  
  - Insert file (Qdrant Vector Store)  
  - Wait 5 sec. (Wait node)  

- **Node Details:**

1. **Get files**  
   - Type: Google Drive  
   - Role: Downloads a file content by its ID.  
   - Config: Converts Google Doc to plain text format.  
   - Inputs: Triggered per file batch.  
   - Outputs: Passes binary content to document loader.  
   - Edge cases: File not found, conversion failure, API limits.

2. **Default Data Loader1**  
   - Type: Langchain Document Default Data Loader  
   - Role: Loads text content from binary data for processing.  
   - Config: Sets metadata fields "file_id" and "file_name" from Google Drive file info.  
   - Inputs: Binary data from "Get files".  
   - Outputs: Document objects for text splitting.  
   - Edge cases: Missing binary field, metadata extraction failure.

3. **Recursive Character Text Splitter**  
   - Type: Langchain Text Splitter Recursive Character  
   - Role: Splits document text into chunks (500 chars with 50 overlap).  
   - Inputs: Document from loader node.  
   - Outputs: Chunked text documents.  
   - Edge cases: Empty documents, chunk size misconfiguration.

4. **Embeddings OpenAI1**  
   - Type: Langchain OpenAI Embeddings  
   - Role: Generates vector embeddings for text chunks.  
   - Credentials: OpenAI API.  
   - Inputs: Text chunks.  
   - Outputs: Embeddings with metadata.  
   - Edge cases: API rate limits, network issues.

5. **Insert file**  
   - Type: Langchain Vector Store Qdrant  
   - Role: Inserts embeddings into the specified Qdrant collection.  
   - Config: Insert mode, collection selected from list ("negozio-emporio-verde").  
   - Credentials: Qdrant API.  
   - Inputs: Embeddings.  
   - Outputs: Triggers "Wait 5 sec." node.  
   - Edge cases: Insert failure, API errors.

6. **Wait 5 sec.**  
   - Type: Wait  
   - Role: Introduces delay to avoid rate limits or sync issues.  
   - Inputs: From "Insert file".  
   - Outputs: Loops back to "Loop Over Items" to process next file.  
   - Edge cases: Delay not respected, workflow timeout.

---

#### 2.3 Automatic Updates on File Changes

- **Overview:**  
  This block listens for updates in the Google Drive folder, deletes old vectors associated with the updated file_id in Qdrant, then re-downloads and re-ingests the file.

- **Nodes Involved:**  
  - Update? (Google Drive Trigger)  
  - Set file_id (Set Node)  
  - Delete points by file_id (HTTP Request)  
  - Get file (Google Drive)  
  - Default Data Loader (Langchain Document Loader)  
  - Recursive Character Text Splitter1 (Langchain Text Splitter)  
  - Embeddings OpenAI2 (Langchain Embeddings)  
  - Update file (Langchain Vector Store Qdrant)  

- **Node Details:**

1. **Update?**  
   - Type: Google Drive Trigger  
   - Role: Watches for any file updates in the specified folder ("Test Negozio").  
   - Config: Polls every hour, triggers on all file types.  
   - Credentials: Google Drive OAuth2.  
   - Outputs: Triggers "Set file_id" node.  
   - Edge cases: Delay in trigger, missed events.

2. **Set file_id**  
   - Type: Set  
   - Role: Extracts and assigns "file_id" from trigger event JSON.  
   - Outputs: Triggers "Delete points by file_id" and "Get file".  
   - Edge cases: Missing file_id, JSON path errors.

3. **Delete points by file_id**  
   - Type: HTTP Request  
   - Role: Deletes all Qdrant points matching the updated file_id metadata.  
   - Config: POST with filter querying metadata.file_id equals the current file_id.  
   - Credentials: Qdrant API.  
   - Edge cases: HTTP errors, partial deletion.

4. **Get file**  
   - Type: Google Drive  
   - Role: Downloads updated file content, converts to plain text.  
   - Credentials: Google Drive OAuth2.  
   - Edge cases: File access issues.

5. **Default Data Loader**  
   - Type: Langchain Document Loader  
   - Role: Loads updated file content, sets metadata.  
   - Edge cases: Missing binary field.

6. **Recursive Character Text Splitter1**  
   - Type: Langchain Text Splitter  
   - Role: Splits updated document text into chunks.  
   - Edge cases: Empty content.

7. **Embeddings OpenAI2**  
   - Type: Langchain Embeddings OpenAI  
   - Role: Generates embeddings for updated chunks.  
   - Edge cases: API limits.

8. **Update file**  
   - Type: Langchain Vector Store Qdrant  
   - Role: Inserts updated embeddings into Qdrant collection.  
   - Edge cases: Insert failure.

---

#### 2.4 Question-Answering Interaction

- **Overview:**  
  This block processes user chat inputs, retrieves relevant document vectors from Qdrant, and produces AI-generated answers using Google Gemini.

- **Nodes Involved:**  
  - When chat message received (Langchain chatTrigger)  
  - Question and Answer Chain (Langchain chainRetrievalQa)  
  - Google Gemini Chat Model (Langchain lmChatGoogleGemini)  
  - Vector Store Retriever (Langchain retrieverVectorStore)  
  - Qdrant Vector Store1 (Langchain vectorStoreQdrant)  
  - Embeddings OpenAI (Langchain embeddingsOpenAI)  

- **Node Details:**

1. **When chat message received**  
   - Type: Langchain chatTrigger  
   - Role: Webhook that receives user input queries.  
   - Outputs: Triggers "Question and Answer Chain".  
   - Edge cases: Webhook downtime.

2. **Question and Answer Chain**  
   - Type: Langchain chainRetrievalQa  
   - Role: Combines retrieved documents with LLM to answer questions.  
   - Inputs: From chat trigger and retriever.  
   - Outputs: Final response.  
   - Edge cases: LLM errors, no relevant documents.

3. **Google Gemini Chat Model**  
   - Type: Langchain lmChatGoogleGemini  
   - Role: Language model that generates answer text.  
   - Credentials: Google Palm API (Google Gemini).  
   - Edge cases: API quota, latency.

4. **Vector Store Retriever**  
   - Type: Langchain retrieverVectorStore  
   - Role: Retrieves top 5 relevant vectors from Qdrant based on query embeddings.  
   - Inputs: From Qdrant vector store.  
   - Edge cases: Empty results.

5. **Qdrant Vector Store1**  
   - Type: Langchain vectorStoreQdrant  
   - Role: Provides access to Qdrant collection "negozio-emporio-verde" for retrieval.  
   - Credentials: Qdrant API.  
   - Edge cases: Connection issues.

6. **Embeddings OpenAI**  
   - Type: Langchain embeddingsOpenAI  
   - Role: Creates embeddings for user query to feed retriever.  
   - Credentials: OpenAI API.  
   - Edge cases: API failure.

---

### 3. Summary Table

| Node Name                     | Node Type                              | Functional Role                       | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                 |
|-------------------------------|--------------------------------------|-------------------------------------|----------------------------------|--------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                       | Manual start for setup               |                                  | Create collection, Clear collection | ## Complete RAG System with autoUpdate documents Using Qdrant                              |
| Create collection             | HTTP Request                        | Creates Qdrant vector collection    | When clicking ‘Test workflow’    | Clear collection               | # STEP 1: Create Qdrant Collection. Change QDRANTURL and COLLECTION                         |
| Clear collection             | HTTP Request                        | Clears all points from collection   | Create collection                | Search files                  | # STEP 1: Create Qdrant Collection. Change QDRANTURL and COLLECTION                         |
| Search files                 | Google Drive                       | List files in target folder         | Clear collection                | Loop Over Items               | # STEP 2: Documents vectorization with Qdrant and Google Drive. Change QDRANTURL & COLLECTION |
| Loop Over Items              | Split In Batches                   | Processes files in batches           | Search files                   | Get files (on batch), no output on empty | # STEP 3: Documents vectorization with Qdrant and Google Drive. Change QDRANTURL & COLLECTION |
| Get files                   | Google Drive                       | Download file content                | Loop Over Items                 | Default Data Loader1          | Set as metadata: FILE_ID, FILE_NAME from Google Drive                                       |
| Default Data Loader1        | Langchain Document Loader          | Loads document from binary text     | Get files                      | Recursive Character Text Splitter | Set as metadata: FILE_ID, FILE_NAME from Google Drive                                       |
| Recursive Character Text Splitter | Langchain Text Splitter        | Splits text into chunks              | Default Data Loader1            | Embeddings OpenAI1            |                                                                                             |
| Embeddings OpenAI1          | Langchain Embeddings OpenAI       | Generates embeddings                 | Recursive Character Text Splitter | Insert file                   |                                                                                             |
| Insert file                 | Langchain Vector Store Qdrant     | Inserts embeddings into Qdrant      | Embeddings OpenAI1             | Wait 5 sec.                  |                                                                                             |
| Wait 5 sec.                 | Wait                             | Delay to avoid rate limits           | Insert file                   | Loop Over Items              |                                                                                             |
| Update?                     | Google Drive Trigger              | Triggers on file update events      |                                  | Set file_id                   |                                                                                             |
| Set file_id                 | Set                              | Assigns file_id from trigger event  | Update?                       | Delete points by file_id, Get file |                                                                                             |
| Delete points by file_id    | HTTP Request                     | Deletes vectors matching file_id    | Set file_id                   |                              |                                                                                             |
| Get file                   | Google Drive                     | Downloads updated file content       | Set file_id                   | Default Data Loader           | Set as metadata: FILE_ID, FILE_NAME from Google Drive                                       |
| Default Data Loader         | Langchain Document Loader        | Loads updated document               | Get file                     | Recursive Character Text Splitter1 | Set as metadata: FILE_ID, FILE_NAME from Google Drive                                       |
| Recursive Character Text Splitter1 | Langchain Text Splitter      | Splits updated text into chunks     | Default Data Loader           | Embeddings OpenAI2           |                                                                                             |
| Embeddings OpenAI2          | Langchain Embeddings OpenAI       | Generates embeddings for updates    | Recursive Character Text Splitter1 | Update file                 |                                                                                             |
| Update file                | Langchain Vector Store Qdrant     | Updates embeddings in Qdrant        | Embeddings OpenAI2            |                              |                                                                                             |
| When chat message received  | Langchain chatTrigger             | Receives user queries                |                                  | Question and Answer Chain     | # STEP 4: Try RAG                                                                          |
| Question and Answer Chain   | Langchain chainRetrievalQa        | Retrieves docs and generates answer | When chat message received, Vector Store Retriever, Google Gemini Chat Model |                              | # STEP 4: Try RAG                                                                          |
| Google Gemini Chat Model    | Langchain lmChatGoogleGemini      | Generates AI answer text             | Question and Answer Chain      |                              | # STEP 4: Try RAG                                                                          |
| Vector Store Retriever      | Langchain retrieverVectorStore    | Retrieves relevant vectors           | Qdrant Vector Store1           | Question and Answer Chain     | # STEP 4: Try RAG                                                                          |
| Qdrant Vector Store1        | Langchain vectorStoreQdrant       | Accesses Qdrant collection           | Embeddings OpenAI             | Vector Store Retriever        |                                                                                             |
| Embeddings OpenAI           | Langchain Embeddings OpenAI       | Embeds user query text               | When chat message received    | Qdrant Vector Store1          |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually for setup.

2. **Create HTTP Request Node "Create collection"**  
   - Method: PUT  
   - URL: `http://<QDRANTURL>/collections/test_sparse` (replace `<QDRANTURL>` accordingly)  
   - Body (JSON): Define vectors size=1536, distance=Cosine, sparse_vectors text property, shard_number=1, replication_factor=1, write_consistency_factor=1.  
   - Authentication: HTTP Header Auth with Qdrant API key.  
   - Connect Manual Trigger → Create collection.

3. **Create HTTP Request Node "Clear collection"**  
   - Method: POST  
   - URL: `http://<QDRANTURL>/collections/<COLLECTION>/points/delete`  
   - Body: `{ "filter": {} }` (deletes all points)  
   - Authentication: Same as above.  
   - Connect Create collection → Clear collection.

4. **Create Google Drive Node "Search files"**  
   - Operation: List files  
   - Filter: Folder ID set to your target folder (e.g., "1RO5ByPhq2yvYLmbapTNC_kKdU5lZd4W5")  
   - Credentials: Connect Google Drive OAuth2  
   - Connect Clear collection → Search files.

5. **Create SplitInBatches Node "Loop Over Items"**  
   - Default batch size.  
   - Connect Search files → Loop Over Items.

6. **Create Google Drive Node "Get files"**  
   - Operation: Download  
   - File ID: `={{ $json.id }}` (from input)  
   - Convert Google Docs to plain text.  
   - Credentials: Google Drive OAuth2.  
   - Connect Loop Over Items → Get files.

7. **Create Langchain Document Loader Node "Default Data Loader1"**  
   - Data Type: Binary, Specific Field (binary.data)  
   - Metadata: Set "file_id" to `={{ $('Get files').item.json.id }}` and "file_name" to `={{ $('Get files').item.json.name }}`.  
   - Connect Get files → Default Data Loader1.

8. **Create Langchain Text Splitter Node "Recursive Character Text Splitter"**  
   - Chunk Size: 500 chars  
   - Chunk Overlap: 50 chars  
   - Connect Default Data Loader1 → Recursive Character Text Splitter.

9. **Create Langchain Embeddings Node "Embeddings OpenAI1"**  
   - Credentials: OpenAI API configured.  
   - Connect Recursive Character Text Splitter → Embeddings OpenAI1.

10. **Create Langchain Vector Store Node "Insert file"**  
    - Mode: Insert  
    - Collection: Select your Qdrant collection (e.g., "negozio-emporio-verde")  
    - Credentials: Qdrant API.  
    - Connect Embeddings OpenAI1 → Insert file.

11. **Create Wait Node "Wait 5 sec."**  
    - Default 5 seconds delay.  
    - Connect Insert file → Wait 5 sec.

12. **Connect Wait 5 sec. → Loop Over Items** to continue processing files.

13. **Create Google Drive Trigger Node "Update?"**  
    - Trigger on fileUpdated event for the same folder as "Search files".  
    - Poll every hour.  
    - Credentials: Google Drive OAuth2.

14. **Create Set Node "Set file_id"**  
    - Assign variable "file_id" = `={{ $json.id }}` (from trigger event).  
    - Connect Update? → Set file_id.

15. **Create HTTP Request Node "Delete points by file_id"**  
    - Method: POST  
    - URL: `http://<QDRANTURL>/collections/<COLLECTION>/points/delete`  
    - Body:  
      ```json
      {
        "filter": {
          "must": [
            { "key": "metadata.file_id", "match": { "value": "={{ $json.file_id }}" } }
          ]
        }
      }
      ```  
    - Credentials: Qdrant API.  
    - Connect Set file_id → Delete points by file_id.

16. **Create Google Drive Node "Get file"**  
    - Operation: Download  
    - File ID: `={{ $json.file_id }}`  
    - Convert Google Docs to plain text.  
    - Credentials: Google Drive OAuth2.  
    - Connect Set file_id → Get file.

17. **Create Langchain Document Loader Node "Default Data Loader"**  
    - Data Type: Binary, Specific Field (binary.data)  
    - Metadata: Set "file_id" and "file_name" accordingly.  
    - Connect Get file → Default Data Loader.

18. **Create Langchain Text Splitter Node "Recursive Character Text Splitter1"**  
    - Same chunk size and overlap as before.  
    - Connect Default Data Loader → Recursive Character Text Splitter1.

19. **Create Langchain Embeddings Node "Embeddings OpenAI2"**  
    - Credentials: OpenAI API.  
    - Connect Recursive Character Text Splitter1 → Embeddings OpenAI2.

20. **Create Langchain Vector Store Node "Update file"**  
    - Mode: Insert  
    - Collection: Same Qdrant collection.  
    - Credentials: Qdrant API.  
    - Connect Embeddings OpenAI2 → Update file.

21. **Create Langchain Chat Trigger Node "When chat message received"**  
    - Webhook configured to receive chat input.  

22. **Create Langchain Vector Store Node "Qdrant Vector Store1"**  
    - Collection: Same Qdrant.  
    - Credentials: Qdrant API.

23. **Create Langchain Embeddings Node "Embeddings OpenAI"**  
    - Credentials: OpenAI API.

24. **Create Langchain Retriever Node "Vector Store Retriever"**  
    - TopK: 5  
    - Connect Qdrant Vector Store1 → Vector Store Retriever.

25. **Create Langchain LLM Node "Google Gemini Chat Model"**  
    - Model: "models/gemini-1.5-flash"  
    - Credentials: Google PaLM API (Google Gemini).

26. **Create Langchain Chain Node "Question and Answer Chain"**  
    - Connect When chat message received → Question and Answer Chain.  
    - Connect Vector Store Retriever → Question and Answer Chain.  
    - Connect Google Gemini Chat Model → Question and Answer Chain.

27. **Connect Embeddings OpenAI → Qdrant Vector Store1** to embed queries.

This completes the workflow reconstruction.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow demonstrates a complete RAG system integrating **OpenAI embeddings**, **Google Gemini LLM**, **Qdrant vector database**, and **Google Drive** for automatic document ingestion and updating.                   | Workflow purpose overview                                                                        |
| Replace all `QDRANTURL` and `COLLECTION` placeholders with your actual Qdrant deployment URL and collection name before running.                                                                                                | Critical setup information                                                                       |
| Google Drive folder ID is set to "1RO5ByPhq2yvYLmbapTNC_kKdU5lZd4W5" for demo; change it to your target folder.                                                                                                                  | Google Drive configuration                                                                       |
| Metadata fields "file_id" and "file_name" are attached to each document chunk to enable precise update and deletion in Qdrant.                                                                                                | Metadata design                                                                                  |
| Rate limiting is handled by adding a 5-second wait after each file insertion to Qdrant. Adjust as needed depending on API quotas and latency.                                                                                  | Rate limit mitigation                                                                            |
| Google Drive trigger polls every hour for file updates; for real-time updates consider alternative triggers or API push notifications if supported.                                                                           | Trigger configuration note                                                                      |
| Google Gemini Chat Model uses "models/gemini-1.5-flash" — verify your Google PaLM API access and model availability.                                                                                                           | LLM model usage                                                                                  |
| For error handling: Monitor HTTP request responses for Qdrant and Google Drive nodes; consider adding retry logic on failures for robustness.                                                                                | Suggested error handling                                                                        |
| Documentation and inspiration: [Qdrant official docs](https://qdrant.tech/documentation/), [Google Gemini API](https://developers.generativeai.google/), [OpenAI API](https://platform.openai.com/docs).                       | Useful external references                                                                      |

---

**Disclaimer:**  
The provided text and workflow originate exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.