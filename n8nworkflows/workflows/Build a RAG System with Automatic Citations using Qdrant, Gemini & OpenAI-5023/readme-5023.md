Build a RAG System with Automatic Citations using Qdrant, Gemini & OpenAI

https://n8nworkflows.xyz/workflows/build-a-rag-system-with-automatic-citations-using-qdrant--gemini---openai-5023


# Build a RAG System with Automatic Citations using Qdrant, Gemini & OpenAI

### 1. Workflow Overview

This workflow implements a **Retrieval-Augmented Generation (RAG)** system to provide AI-generated answers with automatic source citations. It targets use cases where knowledge documents stored in Google Drive are vectorized and indexed in a Qdrant vector database. Upon receiving a user query, it retrieves relevant documents, generates an answer using Google Gemini (PaLM API), and cites the original document sources automatically.

The workflow is logically divided into the following blocks:

- **1.1 Collection Initialization**: Creates and clears the Qdrant collection to ensure a fresh vector database state.
- **1.2 Document Ingestion and Vectorization**: Loads documents from a specific Google Drive folder, splits them into chunks, generates embeddings using OpenAI, and inserts vectors into Qdrant with metadata.
- **1.3 Query Handling and Retrieval**: Listens for chat queries, converts input into embeddings, retrieves top relevant documents from Qdrant, and aggregates metadata about the sources.
- **1.4 AI Answer Generation with Citation**: Uses the retrieved documents and Google Gemini chat model to generate an answer, then formats the output with automatic source citations.

---

### 2. Block-by-Block Analysis

#### 2.1 Collection Initialization

- **Overview**: This block ensures the Qdrant collection is ready by creating it if necessary and clearing any existing points before ingestion.
- **Nodes Involved**:  
  - Create collection  
  - Clear collection  
  - Get folder (entry to next block)  
  - Sticky Note3 (instructions)  

- **Node Details**:

  - **Create collection**  
    - Type: HTTP Request  
    - Role: Creates a Qdrant collection with specified vector size (1536) and cosine distance metric.  
    - Configuration: PUT request to `http://QDRANTURL/collections/COLLECTION` with JSON body defining vector size and replication parameters.  
    - Authentication: HTTP Header Auth with Qdrant API credentials.  
    - Input/Output: Triggered manually; outputs to Clear collection.  
    - Edge cases: API endpoint or auth misconfiguration, network timeout, invalid JSON body.  
    - Notes: Requires replacing placeholders `QDRANTURL` and `COLLECTION` with actual values.

  - **Clear collection**  
    - Type: HTTP Request  
    - Role: Deletes all points in the Qdrant collection to start fresh.  
    - Configuration: POST to `http://QDRANTURL/collections/COLLECTION/points/delete` with empty filter `{}` to delete all points.  
    - Authentication: Same as Create collection.  
    - Input/Output: Triggered after Create collection, outputs to Get folder.  
    - Edge cases: Same as Create collection; ensure collection exists.

  - **Get folder**  
    - Type: Google Drive node  
    - Role: Fetches files from a specific Google Drive folder ("Test Negozio") to process.  
    - Configuration: Lists files in folder ID `1RO5ByPhq2yvYLmbapTNC_kKdU5lZd4W5` within "My Drive".  
    - Credentials: Google Drive OAuth2.  
    - Input/Output: Triggered after Clear collection, outputs to Loop Over Items.  
    - Edge cases: Folder ID invalid or permission denied, API quota issues.

  - **Sticky Note3**  
    - Content: Instructions to update Qdrant URL and collection name for this step.

#### 2.2 Document Ingestion and Vectorization

- **Overview**: Processes each document file from Google Drive: downloads content, splits it into chunks, generates embeddings with OpenAI, and inserts them into Qdrant with file metadata for citation.
- **Nodes Involved**:  
  - Loop Over Items  
  - Get file  
  - Default Data Loader1  
  - Recursive Character Text Splitter  
  - Embeddings OpenAI1  
  - Qdrant Vector Store  
  - Wait  
  - Sticky Note4, Sticky Note (metadata explanation)  

- **Node Details**:

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Iterates over each file from Get folder to process sequentially.  
    - Input/Output: Input from Get folder, outputs to Get file (second output).  
    - Edge cases: Large number of files may slow down processing.

  - **Get file**  
    - Type: Google Drive node  
    - Role: Downloads file content as plain text (converted from Google Docs).  
    - Configuration: Uses file ID from current item, converts Google Docs to "text/plain".  
    - Credentials: Google Drive OAuth2.  
    - Input/Output: From Loop Over Items, outputs to Qdrant Vector Store.  
    - Edge cases: File access issues, unsupported file formats.

  - **Default Data Loader1**  
    - Type: LangChain Document Default Data Loader  
    - Role: Loads the binary text data from the file node with metadata fields for file ID and name.  
    - Configuration: Sets metadata fields `file_id` and `file_name` extracted from the Get file node JSON.  
    - Input/Output: Takes binary text, outputs to Qdrant Vector Store.  
    - Edge cases: Missing or malformed metadata fields.

  - **Recursive Character Text Splitter**  
    - Type: LangChain Text Splitter  
    - Role: Splits document text into chunks of 500 characters with 50-character overlap for embedding.  
    - Configuration: Default options, chunkSize=500, chunkOverlap=50.  
    - Input/Output: Connected into Default Data Loader1 (ai_textSplitter).  
    - Edge cases: Very short or empty documents.

  - **Embeddings OpenAI1**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates 1536-dimensional text embeddings using OpenAI API for each chunk.  
    - Credentials: OpenAI API key configured.  
    - Input/Output: From Default Data Loader1, outputs to Qdrant Vector Store (ai_embedding).  
    - Edge cases: API rate limits, invalid API key.

  - **Qdrant Vector Store**  
    - Type: LangChain Vector Store Qdrant  
    - Role: Inserts vector embeddings and associated metadata into Qdrant collection.  
    - Configuration: Insert mode, target collection "negozio-emporio-verde" (example).  
    - Credentials: Qdrant API.  
    - Input/Output: Receives embeddings, outputs to Wait node.  
    - Edge cases: Connectivity issues, malformed vectors.

  - **Wait**  
    - Type: Wait node  
    - Role: Introduces a delay to avoid API throttling or timing issues between insert operations.  
    - Input/Output: From Qdrant Vector Store, loops back to Loop Over Items for next file.  
    - Edge cases: Infinite loops if not configured properly.

  - **Sticky Note4**  
    - Content: Instructions to update Qdrant URL and collection for document vectorization.

  - **Sticky Note (metadata)**  
    - Content: Explains metadata structure added to vectors including file ID and file name for source citation.

#### 2.3 Query Handling and Retrieval

- **Overview**: Listens for chat input, converts the input query to embeddings, retrieves top relevant documents from Qdrant, and aggregates their metadata.
- **Nodes Involved**:  
  - When chat message received  
  - chatInput  
  - Embeddings OpenAI4  
  - Retrive sources (Qdrant Vector Store load mode)  
  - Aggregate  
  - Merge1  
  - Response  
  - Output  

- **Node Details**:

  - **When chat message received**  
    - Type: LangChain Chat Trigger  
    - Role: Webhook listener for incoming chat queries.  
    - Input/Output: Triggers chatInput node.  
    - Edge cases: Webhook connectivity or authentication.

  - **chatInput**  
    - Type: Set  
    - Role: Extracts and sets the chat input string for use in further nodes.  
    - Configuration: Sets variable `chatInput` from incoming JSON.  
    - Input/Output: From chat trigger, outputs to Question and Answer Chain and Retrive sources.  
    - Edge cases: Missing or malformed chat input field.

  - **Embeddings OpenAI4**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates embeddings for the chat query text.  
    - Credentials: OpenAI API.  
    - Input/Output: Feeds into Retrive sources node as ai_embedding.  
    - Edge cases: API rate limits.

  - **Retrive sources**  
    - Type: LangChain Vector Store Qdrant (load mode)  
    - Role: Retrieves top 5 most similar document vectors from Qdrant based on the query embedding.  
    - Configuration: TopK=5, uses chatInput as prompt.  
    - Credentials: Qdrant API.  
    - Input/Output: Outputs to Aggregate.  
    - Edge cases: No data found, connection issues.

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates metadata fields from retrieved documents for unique source identification.  
    - Configuration: Aggregates `document.metadata.file_id` and `document.metadata.file_name`.  
    - Input/Output: Output to Merge1.  
    - Edge cases: Empty results.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines AI generated response with aggregated metadata for final output.  
    - Configuration: Combine all inputs.  
    - Input/Output: Receives from Question and Answer Chain and Aggregate, outputs to Response.  
    - Edge cases: Misaligned data inputs.

  - **Response**  
    - Type: Code node  
    - Role: Processes aggregated metadata to produce unique lists of file IDs and file names for citation.  
    - Script: Creates JS Set to remove duplicates and returns unique file IDs/names.  
    - Input/Output: Output to Output node.  
    - Edge cases: Empty data arrays.

  - **Output**  
    - Type: Set  
    - Role: Formats the final response string including AI answer and the list of source file names for display.  
    - Configuration: Uses `Question and Answer Chain` response and unique file names from Response node.  
    - Edge cases: Formatting issues if data missing.

#### 2.4 AI Answer Generation with Citation

- **Overview**: Uses the retrieved relevant document chunks and the Google Gemini language model to generate a context-aware answer to the user query.
- **Nodes Involved**:  
  - Question and Answer Chain  
  - Google Gemini Chat Model  
  - Vector Store Retriever  
  - Qdrant Vector Store1  
  - Embeddings OpenAI  
  - Sticky Note1 (final output explanation)

- **Node Details**:

  - **Vector Store Retriever**  
    - Type: LangChain Retriever Vector Store  
    - Role: Retrieves relevant document vectors from Qdrant to feed into the QA chain.  
    - Configuration: TopK=5 on collection "negozio-emporio-verde".  
    - Credentials: Qdrant API.  
    - Input/Output: Connects with Qdrant Vector Store1 and Question and Answer Chain.  
    - Edge cases: No relevant documents found.

  - **Qdrant Vector Store1**  
    - Type: LangChain Vector Store Qdrant  
    - Role: Loads vectors from Qdrant for retrieval.  
    - Configuration: Collection "negozio-emporio-verde".  
    - Credentials: Qdrant API.  
    - Input/Output: Feeds into Vector Store Retriever.  
    - Edge cases: Connection or auth failures.

  - **Embeddings OpenAI**  
    - Type: LangChain Embeddings OpenAI  
    - Role: Generates embeddings for AI language model input.  
    - Credentials: OpenAI API.  
    - Input/Output: Feeds into Qdrant Vector Store1.  
    - Edge cases: API limits.

  - **Question and Answer Chain**  
    - Type: LangChain Chain Retrieval QA  
    - Role: Orchestrates retrieval and generation of answer using retrieved document chunks and language model.  
    - Input/Output: Inputs from chatInput (query) and Vector Store Retriever; outputs answer to Merge1.  
    - Edge cases: Language model API failures, empty retrieval results.

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Uses Google Gemini (PaLM) API to generate AI answer in chat format.  
    - Configuration: Model name "models/gemini-1.5-flash".  
    - Credentials: Google Palm API key.  
    - Input/Output: Feeds into Question and Answer Chain as language model.  
    - Edge cases: API quota, authentication issues.

  - **Sticky Note1**  
    - Content: Explains the final output format which includes the AI response followed by the list of source file names.

---

### 3. Summary Table

| Node Name                | Node Type                                  | Functional Role                                   | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                   |
|--------------------------|--------------------------------------------|--------------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                            | Manual start point for collection initialization  |                             | Clear collection            |                                                                                               |
| Create collection        | HTTP Request                              | Creates Qdrant collection                         | When clicking ‘Test workflow’ | Clear collection            | # STEP 1: Change QDRANTURL and COLLECTION                                                     |
| Clear collection         | HTTP Request                              | Clears all points in Qdrant collection            | Create collection            | Get folder                  | # STEP 1: Change QDRANTURL and COLLECTION                                                     |
| Get folder               | Google Drive                              | Lists files in specified Google Drive folder      | Clear collection             | Loop Over Items             | # STEP 2: Change QDRANTURL and COLLECTION                                                    |
| Loop Over Items          | Split In Batches                          | Iterates over each file in folder                  | Get folder                  | Get file                    | # STEP 2: Change QDRANTURL and COLLECTION                                                    |
| Get file                 | Google Drive                              | Downloads single file content as plain text       | Loop Over Items             | Qdrant Vector Store         | # STEP 2: Change QDRANTURL and COLLECTION                                                    |
| Default Data Loader1     | LangChain Document Data Loader            | Loads document content with metadata               | Recursive Character Text Splitter | Qdrant Vector Store         | Metadata includes file_id and file_name for citations                                        |
| Recursive Character Text Splitter | LangChain Text Splitter                  | Splits text into chunks for embedding              | Default Data Loader1        | Default Data Loader1 (ai_textSplitter) | Metadata includes file_id and file_name for citations                                        |
| Embeddings OpenAI1       | LangChain Embeddings OpenAI               | Generates embeddings for document chunks           | Default Data Loader1        | Qdrant Vector Store         | Requires OpenAI credentials                                                                  |
| Qdrant Vector Store      | LangChain Vector Store Qdrant             | Inserts vectors with metadata into Qdrant          | Embeddings OpenAI1 / Default Data Loader1 | Wait                       | Requires Qdrant API credentials, target collection specified                                 |
| Wait                     | Wait                                     | Delay between vector inserts to manage rate limits| Qdrant Vector Store         | Loop Over Items             |                                                                                               |
| When chat message received | LangChain Chat Trigger                    | Webhook trigger for user chat queries              |                             | chatInput                   |                                                                                               |
| chatInput                | Set                                      | Extracts and sets chat input string                 | When chat message received  | Question and Answer Chain, Retrive sources |                                                                                               |
| Embeddings OpenAI4       | LangChain Embeddings OpenAI               | Generates embeddings for chat query                 | chatInput                   | Retrive sources             | Requires OpenAI credentials                                                                  |
| Retrive sources          | LangChain Vector Store Qdrant (load)      | Retrieves top relevant documents from Qdrant       | Embeddings OpenAI4          | Aggregate                   | Requires Qdrant API credentials                                                              |
| Aggregate                | Aggregate                                | Aggregates source metadata fields uniquely         | Retrive sources             | Merge1                     |                                                                                               |
| Question and Answer Chain | LangChain Chain Retrieval QA              | Generates answer using retrieved documents and LM  | chatInput, Vector Store Retriever | Merge1                     |                                                                                               |
| Merge1                   | Merge                                    | Combines AI answer and metadata for final output   | Question and Answer Chain, Aggregate | Response                   |                                                                                               |
| Response                 | Code                                     | Removes duplicate citations, formats metadata      | Merge1                      | Output                      |                                                                                               |
| Output                   | Set                                      | Formats final response string with answer and sources| Response                    |                             |                                                                                               |
| Vector Store Retriever   | LangChain Retriever Vector Store          | Retrieves relevant vectors for Q&A chain            | Qdrant Vector Store1        | Question and Answer Chain   |                                                                                               |
| Qdrant Vector Store1     | LangChain Vector Store Qdrant             | Loads vectors from Qdrant for retrieval             | Embeddings OpenAI           | Vector Store Retriever      |                                                                                               |
| Embeddings OpenAI        | LangChain Embeddings OpenAI               | Generates embeddings for retrieval                   |                             | Qdrant Vector Store1        | Requires OpenAI credentials                                                                  |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini           | Google Gemini language model for answer generation  |                             | Question and Answer Chain   | Final output format: response + list of source file names                                    |
| Sticky Note2             | Sticky Note                              | Workflow description                                |                             |                             | Complete RAG System with Automatic Source Citations Using Qdrant                            |
| Sticky Note3             | Sticky Note                              | Instructions on Qdrant collection creation          |                             |                             | # STEP 1: Change QDRANTURL and COLLECTION                                                   |
| Sticky Note4             | Sticky Note                              | Instructions for document vectorization             |                             |                             | # STEP 2: Change QDRANTURL and COLLECTION                                                   |
| Sticky Note (metadata)   | Sticky Note                              | Metadata format for citation                         |                             |                             | Metadata includes file_id and file_name for source citation                                |
| Sticky Note1             | Sticky Note                              | Final output explanation                             |                             |                             | Final output is: RESPONSE + Sources: [“FILENAME 1”, “FILENAME 2”, ...]                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.  
   - This starts the ingestion process manually.

2. **Create Qdrant Collection:**  
   - Add an **HTTP Request** node named `Create collection`.  
   - Method: PUT  
   - URL: `http://QDRANTURL/collections/COLLECTION` (replace placeholders)  
   - Body (JSON):  
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
   - Authentication: HTTP Header Auth with Qdrant API key.  
   - Connect `When clicking ‘Test workflow’` → `Create collection`.

3. **Clear Qdrant Collection:**  
   - Add an **HTTP Request** node named `Clear collection`.  
   - Method: POST  
   - URL: `http://QDRANTURL/collections/COLLECTION/points/delete`  
   - Body (JSON): `{ "filter": {} }`  
   - Authentication: Same as above.  
   - Connect `Create collection` → `Clear collection`.

4. **Get Google Drive Folder Files:**  
   - Add a **Google Drive** node named `Get folder`.  
   - Operation: List files in folder  
   - Drive ID: "My Drive"  
   - Folder ID: [Set your folder ID] (e.g., `1RO5ByPhq2yvYLmbapTNC_kKdU5lZd4W5`)  
   - Credentials: Google Drive OAuth2.  
   - Connect `Clear collection` → `Get folder`.

5. **Loop Over Files:**  
   - Add a **Split In Batches** node named `Loop Over Items`.  
   - Connect `Get folder` → `Loop Over Items`.

6. **Download File Content:**  
   - Add a **Google Drive** node named `Get file`.  
   - Operation: Download file  
   - File ID: Set to `={{ $json.id }}` from loop item  
   - Enable Google Docs conversion to `text/plain`.  
   - Credentials: Google Drive OAuth2.  
   - Connect second output of `Loop Over Items` → `Get file`.

7. **Split Text into Chunks:**  
   - Add a **LangChain Recursive Character Text Splitter** node named `Recursive Character Text Splitter`.  
   - Chunk size: 500 characters  
   - Chunk overlap: 50 characters  
   - Connect output of `Get file` → `Recursive Character Text Splitter`.

8. **Load Document with Metadata:**  
   - Add a **LangChain Default Data Loader** node named `Default Data Loader1`.  
   - Set data type: binary specific field (text content)  
   - Add metadata fields:  
     - `file_id`: `={{ $('Get file').item.json.id }}`  
     - `file_name`: `={{ $('Get file').item.json.name }}`  
   - Connect `Recursive Character Text Splitter` → `Default Data Loader1`.

9. **Generate Embeddings for Document Chunks:**  
   - Add **LangChain Embeddings OpenAI** node named `Embeddings OpenAI1`.  
   - Use OpenAI API credentials.  
   - Connect `Default Data Loader1` → `Embeddings OpenAI1`.

10. **Insert Vectors into Qdrant:**  
    - Add **LangChain Vector Store Qdrant** node named `Qdrant Vector Store`.  
    - Mode: Insert  
    - Target collection: `negozio-emporio-verde` or your collection name  
    - Use Qdrant API credentials.  
    - Connect `Embeddings OpenAI1` → `Qdrant Vector Store`.

11. **Add Wait Node:**  
    - Add a **Wait** node named `Wait`.  
    - Connect `Qdrant Vector Store` → `Wait`.  
    - Connect `Wait` → `Loop Over Items` (to continue with the next file).

12. **Setup Chat Query Trigger:**  
    - Add **LangChain Chat Trigger** node named `When chat message received`.  
    - Set webhook for chat input.  

13. **Extract Chat Input:**  
    - Add a **Set** node named `chatInput`.  
    - Assign variable `chatInput` from incoming JSON field `chatInput`.  
    - Connect `When chat message received` → `chatInput`.

14. **Generate Query Embeddings:**  
    - Add **LangChain Embeddings OpenAI** node named `Embeddings OpenAI4`.  
    - Use OpenAI API credentials.  
    - Connect `chatInput` → `Embeddings OpenAI4`.

15. **Retrieve Relevant Documents:**  
    - Add **LangChain Vector Store Qdrant** node named `Retrive sources`.  
    - Mode: Load  
    - TopK: 5  
    - Use Qdrant API credentials, target collection same as above.  
    - Connect `Embeddings OpenAI4` → `Retrive sources`.

16. **Aggregate Metadata:**  
    - Add **Aggregate** node named `Aggregate`.  
    - Aggregate fields: `document.metadata.file_id` and `document.metadata.file_name`.  
    - Connect `Retrive sources` → `Aggregate`.

17. **Retrieve Vectors for QA Chain:**  
    - Add **LangChain Embeddings OpenAI** node named `Embeddings OpenAI`.  
    - Use OpenAI API credentials.

18. **Load Vectors for Retrieval:**  
    - Add **LangChain Vector Store Qdrant** node named `Qdrant Vector Store1`.  
    - Use Qdrant API credentials, same collection.  
    - Connect `Embeddings OpenAI` → `Qdrant Vector Store1`.

19. **Vector Store Retriever:**  
    - Add **LangChain Retriever Vector Store** node named `Vector Store Retriever`.  
    - TopK: 5  
    - Connect `Qdrant Vector Store1` → `Vector Store Retriever`.

20. **Google Gemini Chat Model:**  
    - Add **LangChain LM Chat Google Gemini** node named `Google Gemini Chat Model`.  
    - Model: `models/gemini-1.5-flash`  
    - Use Google Palm API credentials.  

21. **Question and Answer Chain:**  
    - Add **LangChain Chain Retrieval QA** node named `Question and Answer Chain`.  
    - Connect `chatInput` and `Vector Store Retriever` to it.  
    - Connect `Google Gemini Chat Model` as language model input.  
    - Output goes to Merge1.

22. **Merge AI Answer and Metadata:**  
    - Add **Merge** node named `Merge1`.  
    - Mode: Combine all inputs.  
    - Connect outputs of `Question and Answer Chain` and `Aggregate` to `Merge1`.

23. **Process Unique Sources:**  
    - Add **Code** node named `Response`.  
    - JS code to extract unique file_ids and file_names from aggregated metadata.  

24. **Format Final Output:**  
    - Add **Set** node named `Output`.  
    - Compose final output string including AI response and source file names.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                         |
|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow implements a **Complete RAG System with Automatic Source Citations Using Qdrant**.             | Workflow description (Sticky Note2)                     |
| Change `QDRANTURL` and `COLLECTION` placeholders in HTTP Request nodes for your Qdrant setup.                 | Collection creation and clearing steps (Sticky Note3)   |
| Change `QDRANTURL` and `COLLECTION` in vector store nodes for document ingestion and retrieval.                | Document vectorization step (Sticky Note4)              |
| Metadata attached to vectors includes `file_id` and `file_name` from Google Drive to enable citations.       | Metadata format explanation (Sticky Note)               |
| Final output includes AI-generated response followed by a list of source document filenames.                  | Output explanation (Sticky Note1)                        |
| Google Gemini (PaLM) model used: `models/gemini-1.5-flash`.                                                   | Language model node configuration                        |
| OpenAI embeddings use 1536-dimensional vectors (default for text-embedding-ada-002 or similar).                | Embeddings nodes                                           |
| Google Drive folder ID used: `1RO5ByPhq2yvYLmbapTNC_kKdU5lZd4W5` (example) — replace with your folder ID.     | Google Drive nodes configuration                          |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, respecting all current content policies. All data processed is legal and publicly accessible.