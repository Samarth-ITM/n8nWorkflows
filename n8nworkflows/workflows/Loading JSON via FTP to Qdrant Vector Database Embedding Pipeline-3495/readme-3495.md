Loading JSON via FTP to Qdrant Vector Database Embedding Pipeline

https://n8nworkflows.xyz/workflows/loading-json-via-ftp-to-qdrant-vector-database-embedding-pipeline-3495


# Loading JSON via FTP to Qdrant Vector Database Embedding Pipeline

### 1. Workflow Overview

This workflow automates the bulk ingestion of structured JSON articles from an FTP server into a Qdrant vector database, enabling semantic search, Retrieval-Augmented Generation (RAG), or AI assistant memory use cases. It is designed to process pre-cleaned JSON files containing metadata and text chunks, embedding them with OpenAI embeddings and storing them efficiently for future querying.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Trigger and FTP file listing to identify JSON files for processing.
- **1.2 File Iteration and Download:** Batch processing of each JSON file by downloading it from FTP.
- **1.3 Data Parsing and Chunking:** Convert binary JSON into document format and optionally split text chunks.
- **1.4 Embedding and Storage:** Generate vector embeddings using OpenAI and insert them into Qdrant.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and lists all JSON files available in the specified FTP directory for processing.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- List all the files (FTP List)  
- Sticky Note (FTP file listing explanation)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution on user command.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Triggers "List all the files" node.  
  - Failures: None expected.  
  - Notes: Entry point for manual testing.

- **List all the files**  
  - Type: FTP node (List operation)  
  - Role: Lists all JSON files in the FTP directory `Oracle/AI/embedding/svenska`.  
  - Configuration:  
    - Operation: List  
    - Path: `Oracle/AI/embedding/svenska`  
  - Inputs: Trigger from Manual Trigger node.  
  - Outputs: List of file metadata objects (including file names).  
  - Failures: FTP connection/authentication errors, path not found, empty directory.  
  - Credentials: FTP account configured.  
  - Notes: Sticky note explains recursive listing of .json files.

- **Sticky Note (FTP file listing explanation)**  
  - Type: Sticky Note  
  - Content: Describes the FTP list operation and recursive listing of JSON files.

---

#### 1.2 File Iteration and Download

**Overview:**  
This block processes each file individually by batching the list and downloading each file in binary format from FTP.

**Nodes Involved:**  
- Loop over one item (SplitInBatches)  
- Downloading item (FTP Download)  
- Sticky Note1 (Batch iteration explanation)  
- Sticky Note2 (Download explanation)

**Node Details:**

- **Loop over one item**  
  - Type: SplitInBatches  
  - Role: Iterates over each file item from the FTP list, processing one file per batch.  
  - Configuration: Default batch size = 1 (implied).  
  - Inputs: List of files from "List all the files".  
  - Outputs: Single file metadata per iteration to "Downloading item".  
  - Failures: None expected, but large lists may cause performance delays.

- **Downloading item**  
  - Type: FTP node (Download operation)  
  - Role: Downloads the current file as binary data for processing.  
  - Configuration:  
    - Path: `Oracle/AI/embedding/svenska/{{ $json.name }}` (dynamic expression to select file)  
    - Binary Property Name: `binary.data`  
  - Inputs: Single file metadata from "Loop over one item".  
  - Outputs: Binary file content passed downstream.  
  - Failures: FTP connection issues, file not found, permission errors.  
  - Credentials: Same FTP account as listing node.

- **Sticky Note1 (Batch iteration explanation)**  
  - Describes the batching mechanism for processing each file individually.

- **Sticky Note2 (Download explanation)**  
  - Explains the file download node configuration and dynamic path usage.

---

#### 1.3 Data Parsing and Chunking

**Overview:**  
This block converts the downloaded binary JSON file into a document format suitable for embedding and optionally splits large text chunks based on the "chunk_id" separator.

**Nodes Involved:**  
- Default Data Loader  
- Character Text Splitter  
- Sticky Note3 (Parsing and splitting explanation)

**Node Details:**

- **Default Data Loader**  
  - Type: Document Default Data Loader (LangChain node)  
  - Role: Parses binary JSON into structured document objects compatible with embedding.  
  - Configuration:  
    - Data Type: binary  
  - Inputs: Binary JSON file from "Downloading item".  
  - Outputs: Document objects representing the JSON content.  
  - Failures: Parsing errors if JSON is malformed or binary data corrupted.

- **Character Text Splitter**  
  - Type: Character Text Splitter (LangChain node)  
  - Role: Splits documents into smaller chunks based on the `"chunk_id"` separator string.  
  - Configuration:  
    - Separator: `"chunk_id"` (used as a delimiter to split text chunks)  
  - Inputs: Document objects from "Default Data Loader".  
  - Outputs: Smaller chunked documents for embedding.  
  - Failures: Incorrect splitting if the separator is missing or malformed.

- **Sticky Note3 (Parsing and splitting explanation)**  
  - Details the purpose of the Default Data Loader and the optional text splitter node.

---

#### 1.4 Embedding and Storage

**Overview:**  
This block generates vector embeddings for the text chunks using OpenAI embeddings and inserts them into the Qdrant vector database in batches.

**Nodes Involved:**  
- Embeddings OpenAI  
- Qdrant Vector Store  
- Sticky Note4 (Embedding and storage explanation)

**Node Details:**

- **Embeddings OpenAI**  
  - Type: LangChain OpenAI Embeddings node  
  - Role: Converts text chunks into vector embeddings using OpenAI's embedding model (e.g., text-embedding-ada-002).  
  - Configuration: Default options (uses configured OpenAI API credentials).  
  - Inputs: Chunked documents from "Character Text Splitter".  
  - Outputs: Embedded vectors with metadata.  
  - Failures: API authentication errors, rate limits, network timeouts.

- **Qdrant Vector Store**  
  - Type: LangChain Qdrant Vector Store node  
  - Role: Inserts embedded vectors into the Qdrant collection for semantic search.  
  - Configuration:  
    - Mode: Insert  
    - Collection: `sv_lang_data`  
    - Embedding Batch Size: 100  
  - Inputs: Embedded vectors from "Embeddings OpenAI" (ai_embedding), also accepts documents directly (ai_document) and raw binary (main).  
  - Outputs: Loops back to "List all the files" to restart or continue processing.  
  - Failures: Qdrant API errors, network issues, batch size limits.  
  - Credentials: Qdrant API credentials configured.

- **Sticky Note4 (Embedding and storage explanation)**  
  - Explains batch size, collection settings in Qdrant, embedding model dimension (1536), and API usage.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                         | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                      |
|-------------------------|---------------------------------------|---------------------------------------|---------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                        | Workflow start trigger                 | None                      | List all the files       |                                                                                                |
| List all the files       | FTP (List)                            | List JSON files on FTP server          | When clicking ‘Test workflow’ | Loop over one item       | ### Fetch JSON File List\n**Node:** FTP (all files)\n**Operation:** List\n**Path:** <file path>\n\nRecursively lists all .json files prepared for embedding. |
| Loop over one item       | SplitInBatches                       | Batch iteration over file list          | List all the files         | Downloading item         | ### Iterate Over Files\n**Node:** Loop Over Items\n\nBatches each file path individually for processing. |
| Downloading item         | FTP (Download)                       | Download individual JSON file          | Loop over one item         | Qdrant Vector Store      | ### Download Each File\n**Node:** FTP (1 file download)\n\nDownloads the current file in binary form using:\n```\nPath = file_path/{{ $json.name }}\n``` |
| Default Data Loader      | Document Default Data Loader (LangChain) | Parse binary JSON into document format | Qdrant Vector Store (ai_document), Character Text Splitter | Qdrant Vector Store (ai_document) | ### Parse JSON Document (Default Data Loader)\n**Node:** Default Data Loader\n**Loader Type**: binary\n- Converts JSON structure into a document format compatible with embedding. |
| Character Text Splitter  | Character Text Splitter (LangChain)  | Split documents into smaller chunks    | Default Data Loader        | Embeddings OpenAI        | ### Split into Smaller Chunks\n**Node:** Character Text Splitter\n**Split by:** "chunk_id" or custom logic based on chunk formatting\n\nOptional node if chunk size normalization is required before embedding. |
| Embeddings OpenAI       | OpenAI Embeddings (LangChain)         | Generate vector embeddings              | Character Text Splitter    | Qdrant Vector Store (ai_embedding) | ### Store in Vector DB\n**Node:** Qdrant Vector Store\n**Batch Size:** 100\n\n**Collection:** <collection_name>\nSends cleaned text chunks to OpenAI to get embeddings (1536 dim for text-embedding-ada-002) |
| Qdrant Vector Store     | Qdrant Vector Store (LangChain)       | Insert embeddings into Qdrant DB       | Downloading item (main), Embeddings OpenAI (ai_embedding), Default Data Loader (ai_document) | List all the files       | ### Store in Vector DB\n**Node:** Qdrant Vector Store\n**Batch Size:** 100\n\n**Collection:** <collection_name>\nSends cleaned text chunks to OpenAI to get embeddings (1536 dim for text-embedding-ada-002) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Start the workflow manually.

2. **Add an FTP node for listing files**  
   - Name: `List all the files`  
   - Operation: `List`  
   - Path: `Oracle/AI/embedding/svenska`  
   - Credentials: Configure with your FTP account credentials.  
   - Connect output of Manual Trigger to this node.

3. **Add a SplitInBatches node**  
   - Name: `Loop over one item`  
   - Purpose: Process each file one at a time.  
   - Connect input from `List all the files`.  
   - Use default batch size (1).

4. **Add an FTP node for downloading files**  
   - Name: `Downloading item`  
   - Operation: `Download`  
   - Path: `=Oracle/AI/embedding/svenska/{{ $json.name }}` (use expression to dynamically select file)  
   - Binary Property Name: `binary.data`  
   - Credentials: Same FTP credentials as listing node.  
   - Connect input from `Loop over one item`.

5. **Add a Document Default Data Loader node (LangChain)**  
   - Name: `Default Data Loader`  
   - Data Type: `binary`  
   - Connect input from `Downloading item` (ai_document input).  
   - This node parses the binary JSON into document objects.

6. **Add a Character Text Splitter node (LangChain)**  
   - Name: `Character Text Splitter`  
   - Separator: `"chunk_id"` (used to split text chunks)  
   - Connect input from `Default Data Loader` (ai_textSplitter input).  
   - Optional: Use this node if chunk size normalization or splitting is required.

7. **Add an OpenAI Embeddings node (LangChain)**  
   - Name: `Embeddings OpenAI`  
   - Credentials: Configure with your OpenAI API credentials.  
   - Connect input from `Character Text Splitter` (ai_embedding input).  
   - This node generates vector embeddings for each chunk.

8. **Add a Qdrant Vector Store node (LangChain)**  
   - Name: `Qdrant Vector Store`  
   - Mode: `Insert`  
   - Collection: `sv_lang_data` (or your target Qdrant collection)  
   - Embedding Batch Size: `100`  
   - Credentials: Configure with your Qdrant API credentials.  
   - Connect inputs:  
     - From `Downloading item` (main input)  
     - From `Embeddings OpenAI` (ai_embedding input)  
     - From `Default Data Loader` (ai_document input)  
   - Connect output back to `List all the files` to loop or continue processing.

9. **Configure Qdrant Collection Settings (outside n8n):**  
   - Ensure your Qdrant collection is created with:  
     ```json
     {
       "vectors": {
         "size": 1536,
         "distance": "Cosine"
       }
     }
     ```  
   - This matches the OpenAI text-embedding-ada-002 vector size and similarity metric.

10. **Optional: Add Sticky Notes**  
    - Add sticky notes near nodes to document purpose and configuration for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed to be modular and adaptable; FTP can be replaced with other file sources like Google Drive, Notion, or S3, and OpenAI embeddings can be swapped with local models such as Ollama.                                           | Workflow description                                                                              |
| Qdrant collection configuration must match embedding vector size (1536) and use cosine distance for optimal semantic search performance.                                                                                                          | Qdrant Vector Store node sticky note                                                             |
| This pipeline is ideal for feeding vector databases for semantic search, RAG systems, or AI assistant memory, supporting pre-cleaned JSON with rich metadata and chunking.                                                                           | Workflow description                                                                              |
| For troubleshooting FTP issues, verify credentials, network access, and correct file paths. For OpenAI embedding errors, check API keys, rate limits, and network connectivity. For Qdrant, ensure API endpoint and collection exist and are reachable. | General operational considerations                                                               |
| The workflow uses LangChain nodes for document loading, splitting, embedding, and vector store integration, leveraging n8n’s LangChain node ecosystem for AI workflows.                                                                               | Node types and ecosystem context                                                                 |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Loading JSON via FTP to Qdrant Vector Database Embedding Pipeline" workflow in n8n.