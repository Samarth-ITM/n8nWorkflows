Build a PDF Search System with Mistral OCR and Weaviate DB

https://n8nworkflows.xyz/workflows/build-a-pdf-search-system-with-mistral-ocr-and-weaviate-db-7339


# Build a PDF Search System with Mistral OCR and Weaviate DB

---

### 1. Workflow Overview

This workflow builds a PDF Search System leveraging Mistral OCR for text extraction, Weaviate as a vector database, and LangChain nodes integrated with Cohere embeddings and reranking models. Its main purpose is to enable users to upload PDF documents, automatically extract and process text, embed the content into vectors, store these in a Weaviate database, and finally enable semantic search and retrieval via a knowledge server endpoint.

The workflow is logically divided into these blocks:

- **1.1 PDF Upload and OCR Extraction:** Handles user PDF uploads and extracts text using Mistral OCR.
- **1.2 Document Preparation and Storage:** Prepares extracted text and metadata, splits text, generates embeddings, and stores vectorized documents in Weaviate.
- **1.3 Semantic Search and Retrieval:** Uses vector-based search with Cohere embeddings and reranking to retrieve relevant documents from Weaviate.
- **1.4 MCP Knowledge Server Integration:** Exposes a webhook endpoint for querying the knowledge base through an external MCP server.

---

### 2. Block-by-Block Analysis

#### 1.1 PDF Upload and OCR Extraction

- **Overview:**  
  This block enables manual uploading of PDF files via a web form, extracts text content from the PDFs using Mistral OCR, and prepares the extracted data with relevant metadata for downstream processing.

- **Nodes Involved:**  
  - Upload PDF  
  - Extract Text from PDF  
  - Prepare Document Data  

- **Node Details:**  

  - **Upload PDF**  
    - Type: Form Trigger  
    - Role: Entry point for users to upload PDF documents.  
    - Configuration: Single PDF file upload, ignoring bot submissions, with a submit button labeled "Upload Document."  
    - Inputs: HTTP form submission  
    - Outputs: Binary file data for the uploaded PDF  
    - Edge Cases: Upload of unsupported file types is prevented by file extension filter; uploading large PDFs may cause timeouts.  
    - Version: 2.2  

  - **Extract Text from PDF**  
    - Type: Mistral AI OCR Node  
    - Role: Extracts text content from uploaded PDF files.  
    - Configuration: Uses the binary property "file" to process the PDF data.  
    - Inputs: Binary data from Upload PDF  
    - Outputs: JSON with extracted text in `extractedText` field  
    - Edge Cases: OCR failures, timeouts, or corrupted PDFs may cause extraction failure; node is configured to retry on failure.  
    - Version: 1  

  - **Prepare Document Data**  
    - Type: Set Node  
    - Role: Assigns document metadata including filename, extracted text content, source tag, and upload timestamp.  
    - Configuration: Sets `filename` from upload metadata, `content` from extracted text, `source` fixed as "uploaded_pdf," and current ISO timestamp.  
    - Inputs: JSON from Extract Text from PDF  
    - Outputs: JSON object ready for vector storage  
    - Edge Cases: Missing fields if prior steps fail; date assignment depends on system clock accuracy.  
    - Version: 3.4  

---

#### 1.2 Document Preparation and Storage

- **Overview:**  
  This block processes the extracted text by splitting it into manageable chunks, generates embeddings using Cohere multilingual models, optionally reranks results, and inserts vectorized documents into the Weaviate vector database.

- **Nodes Involved:**  
  - Text Splitter  
  - Document Loader  
  - Cohere Embeddings  
  - Cohere Reranker  
  - Store in Vector Database  

- **Node Details:**  

  - **Text Splitter**  
    - Type: Recursive Character Text Splitter  
    - Role: Splits long text into overlapping chunks of 600 characters with 200 character overlap, using markdown splitting rules.  
    - Configuration: Chunk size 600, overlap 200, split code "markdown".  
    - Inputs: Not directly connected in JSON, but designed to split large documents before loading.  
    - Outputs: Text chunks for embedding  
    - Edge Cases: Improper splitting if markdown syntax is malformed; overlapping chunks increase redundancy but can slow processing.  
    - Version: 1  

  - **Document Loader**  
    - Type: LangChain Document Default Data Loader  
    - Role: Loads and prepares document chunks for embedding and vector storage.  
    - Configuration: Custom text splitting mode enabled, options empty.  
    - Inputs: Text chunks from Text Splitter  
    - Outputs: Document objects for embedding  
    - Edge Cases: Empty or malformed chunks may cause errors; depends on correct text splitting.  
    - Version: 1.1  

  - **Cohere Embeddings**  
    - Type: LangChain Embeddings (Cohere)  
    - Role: Generates vector embeddings for document chunks using "embed-multilingual-v3.0" model.  
    - Configuration: Model name set to "embed-multilingual-v3.0" for multilingual support.  
    - Inputs: Document chunks from Document Loader  
    - Outputs: Vector embeddings for insertion or search  
    - Edge Cases: API failures, rate limits, model version mismatch can cause failures; embedding model must match retrieval model.  
    - Version: 1  

  - **Cohere Reranker**  
    - Type: LangChain Reranker (Cohere)  
    - Role: Reranks search results to improve relevance based on query context.  
    - Configuration: Default settings, attached as reranker during retrieval.  
    - Inputs: Search results from Weaviate  
    - Outputs: Reranked result set  
    - Edge Cases: Reranking failures lead to fallback on raw results; requires consistent embeddings.  
    - Version: 1  

  - **Store in Vector Database**  
    - Type: LangChain Vector Store (Weaviate)  
    - Role: Inserts embedded document vectors into Weaviate under a specific collection.  
    - Configuration: Mode "insert," collection details dynamically assigned (empty collection value in JSON indicates runtime assignment).  
    - Inputs: Embedded vectors from Cohere Embeddings or Document Loader  
    - Outputs: Confirmation of storage  
    - Edge Cases: DB connection issues, schema mismatches, insertion errors; requires valid Weaviate credentials and collection setup.  
    - Version: 1.3  

---

#### 1.3 Semantic Search and Retrieval

- **Overview:**  
  This block performs vector-based semantic search over stored documents using the same Cohere embedding model and reranker, retrieving relevant documents to answer user queries.

- **Nodes Involved:**  
  - Search Knowledge Base  
  - MCP Knowledge Server  

- **Node Details:**  

  - **Search Knowledge Base**  
    - Type: LangChain Vector Store (Weaviate)  
    - Role: Retrieves vectors/documents from Weaviate based on query embedding, supports reranking.  
    - Configuration: Mode "retrieve-as-tool," collection "KnowledgeDocuments," reranking enabled with Cohere Reranker, minimal metadata included.  
    - Inputs: Query embeddings from Cohere Embeddings or external trigger  
    - Outputs: Retrieved and reranked documents  
    - Edge Cases: Empty search results, DB connectivity errors, inconsistent models between embedding and retrieval cause suboptimal results.  
    - Version: 1.3  

  - **MCP Knowledge Server**  
    - Type: LangChain MCP Trigger (Webhook)  
    - Role: Exposes a webhook endpoint to query the knowledge base externally, acts as a tool callable by AI workflows.  
    - Configuration: Webhook path defined, uses header authentication.  
    - Inputs: Incoming HTTP requests with queries  
    - Outputs: Search results from Search Knowledge Base  
    - Edge Cases: Authentication failures, rate limits, malformed requests; depends on Search Knowledge Base node availability.  
    - Version: 2  

---

#### 1.4 Metadata and Documentation

- **Overview:**  
  Provides user-facing instructions and internal notes to clarify the workflow usage and constraints.

- **Nodes Involved:**  
  - Upload Instructions (Sticky Note)  
  - Sticky Note (Embedding and Rerank)  
  - Sticky Note1 (MCP Server Trigger)  

- **Node Details:**  

  - **Upload Instructions**  
    - Type: Sticky Note  
    - Role: Describes the PDF upload section and the processing flow for new documents.  
    - Content: Explains that PDFs are processed by Mistral OCR and stored in the vector DB.  
    - Position: Near the upload nodes for visibility.  

  - **Sticky Note (Embedding and Rerank)**  
    - Type: Sticky Note  
    - Role: Warns users that embedding and retrieval models must remain consistent to avoid errors.  
    - Content: "You can exchange the models, but you must use the same model for embedding and retrieval and no switching later on."  

  - **Sticky Note1 (MCP Server Trigger)**  
    - Type: Sticky Note  
    - Role: Explains that the MCP Server webhook can be called as a tool in AI workflows.  

---

### 3. Summary Table

| Node Name             | Node Type                                            | Functional Role                          | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                                     |
|-----------------------|-----------------------------------------------------|----------------------------------------|-------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Upload PDF            | Form Trigger                                        | User PDF upload entry point             | —                       | Extract Text from PDF        | ## Manual Document (PDF) Upload Section This section allows users to upload PDF files to the knowledge base.                   |
| Extract Text from PDF  | Mistral AI OCR Node                                | Extract text content from PDFs          | Upload PDF              | Prepare Document Data        | ## Manual Document (PDF) Upload Section This section allows users to upload PDF files to the knowledge base.                   |
| Prepare Document Data  | Set Node                                           | Assign metadata and prepare JSON        | Extract Text from PDF    | Store in Vector Database     | ## Manual Document (PDF) Upload Section This section allows users to upload PDF files to the knowledge base.                   |
| Text Splitter          | Recursive Character Text Splitter                   | Split text into chunks for embedding    | — (connected via ai_textSplitter to Document Loader) | Document Loader             |                                                                                                                                |
| Document Loader        | LangChain Document Loader                           | Load document chunks for embedding      | Text Splitter           | Store in Vector Database (via ai_document) |                                                                                                                                |
| Cohere Embeddings      | LangChain Embeddings (Cohere)                       | Generate embeddings for documents       | Document Loader         | Search Knowledge Base, Store in Vector Database | ## Embedding and Rerank You can exchange the models, but you **must** use the same model for embedding and retrieval and **no switching** later on |
| Cohere Reranker        | LangChain Reranker (Cohere)                         | Rerank retrieved documents              | Search Knowledge Base    | Search Knowledge Base        | ## Embedding and Rerank You can exchange the models, but you **must** use the same model for embedding and retrieval and **no switching** later on |
| Store in Vector Database | LangChain Vector Store (Weaviate)                  | Insert or update vectors in Weaviate    | Prepare Document Data, Document Loader, Cohere Embeddings | —                          |                                                                                                                                |
| Search Knowledge Base  | LangChain Vector Store (Weaviate)                   | Retrieve vectors/documents for query    | Cohere Embeddings, Cohere Reranker | MCP Knowledge Server         |                                                                                                                                |
| MCP Knowledge Server   | LangChain MCP Trigger (Webhook)                     | Webhook for external knowledge queries  | Search Knowledge Base   | —                          | ## MCP Server Trigger You can call this MCP Server as a tool in your AI Workflow                                               |
| Upload Instructions    | Sticky Note                                         | Instructional note for upload section   | —                       | —                          | ## Manual Document (PDF) Upload Section This section allows users to upload PDF files to the knowledge base.                   |
| Sticky Note            | Sticky Note                                         | Instructional note on embedding models  | —                       | —                          | ## Embedding and Rerank You can exchange the models, but you **must** use the same model for embedding and retrieval and **no switching** later on |
| Sticky Note1           | Sticky Note                                         | Instructional note on MCP server trigger| —                       | —                          | ## MCP Server Trigger You can call this MCP Server as a tool in your AI Workflow                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Upload PDF node**  
   - Type: Form Trigger  
   - Configure:  
     - Form Title: "Upload Documents to Knowledge Base"  
     - Add one required file field accepting `.pdf` only  
     - Button label: "Upload Document"  
     - Ignore bot submissions enabled  
   - Save and get webhook URL for external access.

2. **Add Extract Text from PDF node**  
   - Type: Mistral AI  
   - Configure:  
     - Binary Property: `file` (matches upload property)  
   - Link Upload PDF output to this node input.  
   - Enable retry on failure to handle OCR errors.

3. **Add Prepare Document Data node**  
   - Type: Set  
   - Configure to assign:  
     - `filename` = expression: `={{ $('Upload PDF').item.json.file.filename }}`  
     - `content` = `={{ $json.extractedText }}`  
     - `source` = `"uploaded_pdf"` (static string)  
     - `upload_timestamp` = `={{ new Date().toISOString() }}`  
   - Connect Extract Text from PDF output to this node.

4. **Add Store in Vector Database node**  
   - Type: LangChain Vector Store (Weaviate)  
   - Configure:  
     - Mode: `insert`  
     - Set the Weaviate collection name to your vector DB collection (e.g., "KnowledgeDocuments").  
     - Input comes from Prepare Document Data node.  
   - Connect Prepare Document Data output here.

5. **Add Text Splitter node**  
   - Type: Recursive Character Text Splitter  
   - Configure:  
     - Chunk size: 600 characters  
     - Chunk overlap: 200 characters  
     - Split code: `markdown`  
   - This node will be linked as AI text splitter downstream.

6. **Add Document Loader node**  
   - Type: LangChain Document Loader  
   - Configure:  
     - Text splitting mode: Custom  
   - Connect Text Splitter output to this node.

7. **Add Cohere Embeddings node**  
   - Type: LangChain Embeddings (Cohere)  
   - Configure:  
     - Model: `embed-multilingual-v3.0` (must be consistent across embedding and retrieval)  
   - Connect Document Loader output here.

8. **Add Cohere Reranker node**  
   - Type: LangChain Reranker (Cohere)  
   - Use default configuration.  
   - Connect Search Knowledge Base retrieval output to this reranker.

9. **Add Search Knowledge Base node**  
   - Type: LangChain Vector Store (Weaviate)  
   - Configure:  
     - Mode: `retrieve-as-tool`  
     - Collection: `KnowledgeDocuments` (same as insert collection)  
     - Enable reranker, assign Cohere Reranker node.  
     - Include minimal metadata (optional).  
   - Connect Cohere Embeddings output for embedding queries and Cohere Reranker node for reranking.

10. **Add MCP Knowledge Server node**  
    - Type: LangChain MCP Trigger (Webhook)  
    - Configure:  
      - Set webhook path (unique URL segment)  
      - Authentication: Header-based (set header keys/values accordingly)  
    - Connect Search Knowledge Base output to this node.  
    - This node exposes the search functionality as an API endpoint.

11. **Add Sticky Notes**  
    - Upload Instructions: Place near Upload PDF node with instructions about PDF upload and OCR process.  
    - Embedding and Rerank Note: Place near embedding and reranking nodes warning about consistent model usage.  
    - MCP Server Trigger Note: Place near MCP Knowledge Server node explaining external call usage.

12. **Credential Setup**  
    - Configure credentials for:  
      - Mistral OCR node (API key or auth as required)  
      - Cohere Embeddings and Reranker (API keys)  
      - Weaviate Vector Store (endpoint, API key if needed)  
      - MCP Trigger (authentication credentials for headerAuth)  

13. **Finalize connections and test**  
    - Test upload → OCR → storage pipeline with sample PDFs.  
    - Test search queries through MCP webhook to verify retrieval and reranking.  
    - Monitor logs for errors or timeouts and adjust chunk sizes or API limits as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| It is critical that the embedding and retrieval models remain consistent to ensure accuracy. | Sticky note near Cohere Embeddings and Reranker nodes.                                         |
| MCP Server webhook acts as an AI tool endpoint, enabling integration with external workflows. | Sticky note near MCP Knowledge Server node.                                                    |
| PDF upload section processes files via Mistral OCR before vectorization and indexing.         | Sticky note near Upload PDF and Extract Text from PDF nodes.                                   |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---