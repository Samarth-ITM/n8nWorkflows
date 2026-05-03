Automated Book Summarization with DeepSeek AI, Qdrant Vector DB & Google Drive

https://n8nworkflows.xyz/workflows/automated-book-summarization-with-deepseek-ai--qdrant-vector-db---google-drive-4566


# Automated Book Summarization with DeepSeek AI, Qdrant Vector DB & Google Drive

### 1. Workflow Overview

This workflow automates the process of summarizing books using DeepSeek AI, Qdrant Vector Database, and Google Drive. It is designed to ingest a book file uploaded to Google Drive, process the content by splitting and embedding it, store it in a vector database, perform semantic retrieval and question-answering, and finally save the summarized output back to Google Drive. The workflow is structured into the following logical blocks:

- **1.1 Input Reception and File Loading:** Triggered by a new file in Google Drive and initial loading of document data.
- **1.2 Text Processing and Vectorization:** Splitting the text recursively and embedding it to store in Qdrant vector store.
- **1.3 Vector Store Management and Retrieval:** Managing the Qdrant collections, performing vector search and retrieval.
- **1.4 AI Processing and Question Answering:** Running AI agents powered by DeepSeek Chat Model for summarization and information extraction.
- **1.5 Output and Cleanup:** Creating the summarized document in Google Drive and clearing temporary vector collections.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Loading

**Overview:**  
This block listens for new file events in Google Drive, downloads the file, and prepares it as input for further processing.

**Nodes Involved:**  
- File Created (Google Drive Trigger)  
- Google Drive (file download)  
- input (Code)  
- Qdrant Vector Store (initial setup)

**Node Details:**  
- **File Created**  
  - Type: Google Drive Trigger  
  - Role: Watches for new files added to Google Drive to start the workflow.  
  - Configuration: Default trigger on file creation.  
  - Input: External (Google Drive event)  
  - Output: File metadata and content references.  
  - Potential failures: Auth errors, network issues, no files detected.

- **Google Drive**  
  - Type: Google Drive node  
  - Role: Downloads the file triggered in the previous node to provide raw content.  
  - Configuration: Uses OAuth2 credentials to access Google Drive; configured to download triggered file.  
  - Input: File Created node output  
  - Output: File content and metadata.  
  - Edge cases: File access denied, large files timeout.

- **input (Code)**  
  - Type: Code node  
  - Role: Prepares/structures incoming file content as input for LangChain nodes.  
  - Configuration: Custom JavaScript to format input data for vector store.  
  - Input: Google Drive node output  
  - Output: Structured data object for downstream nodes.  
  - Edge cases: Parsing errors, unexpected file content.

- **Qdrant Vector Store**  
  - Type: LangChain Qdrant Vector Store node  
  - Role: Acts as initial vector store interface receiving prepared data for embedding.  
  - Configuration: Connected with Cohere embeddings downstream.  
  - Input: input node output  
  - Output: Embedded vectors to be stored.  
  - Edge cases: DB connection failure, auth failures.

---

#### 2.2 Text Processing and Vectorization

**Overview:**  
This block recursively splits the loaded document’s text into manageable chunks, embeds them using Cohere embeddings, and prepares them for storage in Qdrant.

**Nodes Involved:**  
- Recursive Character Text Splitter  
- Default Data Loader1  
- Embeddings Cohere  
- Qdrant Vector Store (and Qdrant Vector Store1)

**Node Details:**  
- **Recursive Character Text Splitter**  
  - Type: LangChain Text Splitter  
  - Role: Splits large text documents recursively based on characters to optimize chunk size.  
  - Configuration: Default recursive splitting without custom parameters shown.  
  - Input: None directly connected (used as ai_textSplitter input for Default Data Loader1)  
  - Output: Text chunks for embedding.  
  - Edge cases: Very large texts may cause timeouts or memory overhead.

- **Default Data Loader1**  
  - Type: LangChain Document Data Loader  
  - Role: Loads split text chunks into a format suitable for vector embedding.  
  - Configuration: Uses Recursive Character Text Splitter output as input.  
  - Input: Recursive Character Text Splitter output  
  - Output: Document chunks for embedding.  
  - Edge cases: Data format inconsistencies.

- **Embeddings Cohere**  
  - Type: LangChain Embeddings (Cohere)  
  - Role: Converts text chunks into vector embeddings for semantic search.  
  - Configuration: Uses Cohere API credentials; connected to multiple Qdrant nodes.  
  - Input: Default Data Loader1 output  
  - Output: Vector embeddings for Qdrant storage.  
  - Edge cases: API quota limits, auth errors, response delays.

- **Qdrant Vector Store & Qdrant Vector Store1**  
  - Type: LangChain Vector Store Qdrant  
  - Role: Receives embeddings and stores them in Qdrant collections for retrieval.  
  - Configuration: Connected to embeddings and retriever nodes; configured with Qdrant API and collection info.  
  - Input: Embeddings Cohere output  
  - Output: Stored vectors ready for retrieval.  
  - Edge cases: Collection conflicts, connection failures.

---

#### 2.3 Vector Store Management and Retrieval

**Overview:**  
Handles vector search queries, retrieves relevant document chunks, and manages collection lifecycle including deletion of temporary collections.

**Nodes Involved:**  
- Vector Store Retriever  
- Question and Answer Chain  
- qdrant_search (Vector Store Qdrant)  
- Delete Collection (HTTP Request)  

**Node Details:**  
- **Vector Store Retriever**  
  - Type: LangChain Retriever  
  - Role: Retrieves relevant vectors from Qdrant based on semantic queries.  
  - Configuration: Uses Qdrant Vector Store1 as source.  
  - Input: Qdrant Vector Store1 output  
  - Output: Retrieved documents for QA chain.  
  - Edge cases: Empty query results, connection issues.

- **Question and Answer Chain**  
  - Type: LangChain Retrieval QA Chain  
  - Role: Processes retrieved documents to answer questions using AI models.  
  - Configuration: Connected to Vector Store Retriever and DeepSeek Chat Model for language model.  
  - Input: Retriever output and AI model  
  - Output: Extracted answers and information.  
  - Edge cases: Model timeout, incomplete answers.

- **qdrant_search**  
  - Type: LangChain Vector Store Qdrant  
  - Role: Performs search queries on Qdrant collections as an AI tool for the AI Agent.  
  - Configuration: Linked with Embeddings Cohere and AI Agent node.  
  - Input: Embeddings Cohere output  
  - Output: Search results fed to AI Agent.  
  - Edge cases: Search latency, partial matches.

- **Delete Collection**  
  - Type: HTTP Request  
  - Role: Deletes Qdrant collections after processing to prevent clutter.  
  - Configuration: HTTP DELETE request to Qdrant API; executeOnce enabled to run once per workflow run; onError set to continue on failure.  
  - Input: Response node triggers deletion  
  - Output: API response (ignored if failure).  
  - Edge cases: API unavailability, permissions denied.

---

#### 2.4 AI Processing and Question Answering

**Overview:**  
The core AI logic where DeepSeek Chat Model processes queries, the AI Agent orchestrates memory and tool usage, and information extraction is performed from answers.

**Nodes Involved:**  
- AI Agent  
- Simple Memory (Buffer Window)  
- DeepSeek Chat Model  
- Information Extractor  
- Split Out  
- Response (Code)  

**Node Details:**  
- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Central orchestrator coordinating language model, memory, and tools (vector search).  
  - Configuration: Linked to Simple Memory (for conversational context), DeepSeek Chat Model (language model), and qdrant_search (tool for vector search).  
  - Input: Split Out node output (document chunks)  
  - Output: Final response text to Response node.  
  - Edge cases: Agent logic failures, memory overflow.

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains a windowed conversational memory to provide context for the AI Agent.  
  - Configuration: Default buffer size (not explicitly defined).  
  - Input: AI Agent’s memory input  
  - Output: Updated memory state for AI Agent.  
  - Edge cases: Memory size limits, stale context.

- **DeepSeek Chat Model**  
  - Type: LangChain DeepSeek Chat Model  
  - Role: Language model generating natural language responses.  
  - Configuration: API credentials for DeepSeek; connected as ai_languageModel for AI Agent, Information Extractor, and QA Chain.  
  - Input: Queries and context from AI Agent and QA Chain.  
  - Output: Textual responses and extracted information.  
  - Edge cases: API rate limiting, response timeouts.

- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Extracts structured information from AI-generated text.  
  - Configuration: Connected to Question and Answer Chain output and feeds into Split Out node.  
  - Input: QA Chain output  
  - Output: Extracted structured info for further processing.  
  - Edge cases: Extraction errors, ambiguous content.

- **Split Out**  
  - Type: Split Out node  
  - Role: Splits multi-part output into separate items for processing by AI Agent.  
  - Configuration: Default splitting behavior.  
  - Input: Information Extractor output  
  - Output: Individual pieces for AI Agent.  
  - Edge cases: Unexpected data format.

- **Response (Code)**  
  - Type: Code node  
  - Role: Formats or processes AI Agent’s final response for output and triggers cleanup and document creation.  
  - Configuration: Custom scripting to prepare output.  
  - Input: AI Agent output  
  - Output: Triggers Delete Collection and Doc nodes.  
  - Edge cases: Code errors, formatting issues.

---

#### 2.5 Output and Cleanup

**Overview:**  
Generates the summarized document on Google Drive and deletes temporary data from Qdrant to maintain storage hygiene.

**Nodes Involved:**  
- Delete Collection (HTTP Request)  
- Doc (Code)  
- Google Drive (create file)  

**Node Details:**  
- **Delete Collection**  
  - (Described above) Deletes temporary Qdrant collections once processing is complete.

- **Doc (Code)**  
  - Type: Code node  
  - Role: Prepares the final document content and metadata for Google Drive upload.  
  - Configuration: Custom script to structure the summary document.  
  - Input: Response node output  
  - Output: Data for Google Drive create node.  
  - Edge cases: Data formatting or content errors.

- **Google Drive (create)**  
  - Type: Google Drive node  
  - Role: Creates new file on Google Drive with summarized content.  
  - Configuration: OAuth2 credentials; file name, mime type, and folder specified dynamically from Doc node.  
  - Input: Doc node output  
  - Output: Confirmation and metadata of created file.  
  - Edge cases: File write permission errors, quota limits.

---

### 3. Summary Table

| Node Name                     | Node Type                            | Functional Role                        | Input Node(s)               | Output Node(s)                | Sticky Note                      |
|-------------------------------|------------------------------------|-------------------------------------|----------------------------|------------------------------|---------------------------------|
| File Created                  | Google Drive Trigger                | Trigger workflow on new file upload | External                   | Google Drive                 |                                 |
| Google Drive                 | Google Drive                       | Download uploaded file               | File Created               | input                        |                                 |
| input                        | Code                              | Prepare input data for vector store | Google Drive               | Qdrant Vector Store          |                                 |
| Qdrant Vector Store          | LangChain Vector Store Qdrant      | Store embeddings in Qdrant           | input                      | Code                        |                                 |
| Code                        | Code                              | Prepares data for QA Chain           | Qdrant Vector Store        | Question and Answer Chain    |                                 |
| Question and Answer Chain    | LangChain Retrieval QA Chain       | Perform Q&A over retrieved docs      | Vector Store Retriever     | Information Extractor        |                                 |
| Vector Store Retriever       | LangChain Retriever                | Retrieve relevant vectors             | Qdrant Vector Store1       | Question and Answer Chain    |                                 |
| Qdrant Vector Store1         | LangChain Vector Store Qdrant      | Store and retrieve vectors            | Embeddings Cohere          | Vector Store Retriever       |                                 |
| Recursive Character Text Splitter | LangChain Text Splitter            | Recursively split text chunks         | -                         | Default Data Loader1         |                                 |
| Default Data Loader1         | LangChain Document Data Loader     | Load document chunks                  | Recursive Character Text Splitter | Qdrant Vector Store       |                                 |
| Embeddings Cohere            | LangChain Embeddings Cohere        | Generate vector embeddings            | Default Data Loader1       | Qdrant Vector Store / Qdrant Vector Store1 / qdrant_search |                                 |
| qdrant_search               | LangChain Vector Store Qdrant      | Vector search tool for AI Agent      | Embeddings Cohere          | AI Agent                    |                                 |
| AI Agent                    | LangChain Agent                   | Core AI orchestrator                  | Split Out                  | Response                    |                                 |
| Simple Memory               | LangChain Memory Buffer Window     | Memory context for AI Agent          | AI Agent                   | AI Agent                    |                                 |
| DeepSeek Chat Model         | LangChain DeepSeek Chat Model      | Language model for AI processing     | -                         | AI Agent / QA Chain / Information Extractor |                                 |
| Information Extractor       | LangChain Information Extractor    | Extract structured info from AI text | Question and Answer Chain  | Split Out                   |                                 |
| Split Out                   | Split Out                         | Split multi-part output               | Information Extractor      | AI Agent                    |                                 |
| Response                    | Code                              | Format AI Agent output and trigger cleanup | AI Agent                   | Delete Collection / Doc      |                                 |
| Delete Collection           | HTTP Request                      | Delete temporary Qdrant collections  | Response                   | -                          |                                 |
| Doc                         | Code                              | Prepare document for Google Drive    | Response                   | Google Drive (create)        |                                 |
| Google Drive (create)       | Google Drive                     | Create summarized document on Drive  | Doc                        | -                          |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node:**  
   - Type: Google Drive Trigger  
   - Purpose: Start workflow on new file upload  
   - Configure OAuth2 credentials and set trigger on file creation.

2. **Add Google Drive Node (Download File):**  
   - Type: Google Drive  
   - Connect input from the trigger node  
   - Configure to download the file that triggered the workflow.

3. **Add Code Node `input`:**  
   - Connect from Google Drive download node  
   - Script to format the file content into structured input for vector embedding.

4. **Add Qdrant Vector Store Node:**  
   - Connect from `input` node  
   - Configure connection to Qdrant instance with appropriate API credentials  
   - Set to accept input for vector embedding storage.

5. **Add Recursive Character Text Splitter Node:**  
   - No explicit input connection (used by Default Data Loader)  
   - Default parameters to split large text into manageable chunks.

6. **Add Default Data Loader Node:**  
   - Connect ai_textSplitter input to Recursive Character Text Splitter  
   - Prepares chunked documents for embeddings.

7. **Add Embeddings Cohere Node:**  
   - Connect from Default Data Loader output  
   - Configure with Cohere API credentials  
   - Outputs vector embeddings.

8. **Add Qdrant Vector Store1 Node:**  
   - Connect ai_embedding input from Embeddings Cohere node  
   - Configure with Qdrant API to store embeddings.

9. **Add Vector Store Retriever Node:**  
   - Connect ai_vectorStore input from Qdrant Vector Store1  
   - Prepares vector retrieval queries.

10. **Add Question and Answer Chain Node:**  
    - Connect ai_retriever input from Vector Store Retriever  
    - Connect ai_languageModel input to DeepSeek Chat Model (created later)  
    - Configured to perform retrieval-augmented QA.

11. **Add DeepSeek Chat Model Node:**  
    - Language model for AI processing; configure with DeepSeek API credentials.  
    - Connect to AI Agent, Question and Answer Chain, and Information Extractor.

12. **Add Information Extractor Node:**  
    - Connect from Question and Answer Chain output  
    - Extracts structured information from answers.

13. **Add Split Out Node:**  
    - Connect from Information Extractor output  
    - Splits multi-part info for AI Agent.

14. **Add AI Agent Node:**  
    - Connect from Split Out node  
    - Connect ai_languageModel input to DeepSeek Chat Model  
    - Connect ai_tool input to qdrant_search (below)  
    - Connect ai_memory input to Simple Memory node.

15. **Add Simple Memory Node:**  
    - Connect ai_memory output to AI Agent.

16. **Add qdrant_search Node:**  
    - Connect ai_embedding input from Embeddings Cohere  
    - Connect ai_tool output to AI Agent.

17. **Add Response Code Node:**  
    - Connect from AI Agent output  
    - Script to format response and trigger cleanup and document creation.

18. **Add Delete Collection HTTP Request Node:**  
    - Connect from Response node  
    - Configure HTTP DELETE to Qdrant collection API  
    - Set executeOnce to true and onError to continue.

19. **Add Doc Code Node:**  
    - Connect from Response node  
    - Prepare document metadata and content for upload.

20. **Add Google Drive (create) Node:**  
    - Connect from Doc node  
    - Configure OAuth2 credentials  
    - Set file name, mime type, and destination folder dynamically based on Doc node output.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| DeepSeek AI provides a specialized chat model optimized for book summarization and semantic search tasks. | Official DeepSeek website and API documentation (not included here)                                 |
| Qdrant Vector DB is used for efficient vector storage and similarity search in this workflow.             | https://qdrant.tech/                                                                                |
| Cohere embeddings generate vector representations of text chunks for semantic matching.                   | https://cohere.ai/                                                                                  |
| Google Drive integration requires OAuth2 credentials configured in n8n for both triggers and file operations.| n8n Credentials setup for Google Drive OAuth2                                                       |
| The workflow includes error tolerance on collection deletion to avoid workflow interruption.               | Delete Collection node configured with "continue on error" to prevent failure halting workflow.   |
| Recursive Character Text Splitter helps manage large documents by chunking effectively for embeddings.    | Useful for processing large texts without exceeding token limits.                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.