Process OCR Documents from Google Drive into Searchable Knowledge Base with OpenAI & Pinecone

https://n8nworkflows.xyz/workflows/process-ocr-documents-from-google-drive-into-searchable-knowledge-base-with-openai---pinecone-11653


# Process OCR Documents from Google Drive into Searchable Knowledge Base with OpenAI & Pinecone

---

### 1. Workflow Overview

This workflow automates the ingestion of OCR-processed documents stored in Google Drive into a semantic search knowledge base powered by OpenAI embeddings and Pinecone vector storage. It targets educational content in Arabic extracted from Google Vision JSON files, specifically lesson materials formatted with structured filenames. The workflow consists of four main logical blocks:

- **1.1 Input Reception & Metadata Extraction:** Watches a Google Drive folder for new OCR JSON files, downloads them, and extracts structured lesson metadata from the filename.
- **1.2 Vision JSON Parsing & Text Cleaning:** Parses the downloaded Google Vision JSON, extracts and normalizes Arabic text, then splits it into clean, manageable chunks.
- **1.3 Embeddings Generation & Vector Storage:** Generates vector embeddings for each text chunk using OpenAI and inserts them into a Pinecone vector index with rich metadata.
- **1.4 Archival of Processed Files:** Moves processed files to an archive folder on Google Drive to avoid reprocessing and maintain workspace hygiene.

This modular design enables scalable ingestion of OCR documents into a searchable vector database optimized for Retrieval-Augmented Generation (RAG) applications.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Metadata Extraction  

**Overview:**  
Monitors a specific Google Drive folder for new OCR JSON files, downloads each file, and extracts detailed metadata from the filename. This metadata structures the lesson information for downstream processing.

**Nodes Involved:**  
- Watch Drive Folder (new files)  
- Filename → Lesson Metadata  
- Download file  
- Sticky Note1  

**Node Details:**  

- **Watch Drive Folder (new files):**  
  - Type: Google Drive Trigger  
  - Role: Detects new files created in a specified Google Drive folder every minute.  
  - Configuration: Watches a user-configured folder ID (`YOUR_INPUT_FOLDER_ID`). Trigger set to "fileCreated".  
  - Connections: Output → Filename → Lesson Metadata  
  - Edge Cases: Network failures, permission issues, or no new files can delay trigger.  
  - Credentials: Google Drive OAuth2 required.  

- **Filename → Lesson Metadata:**  
  - Type: Code  
  - Role: Parses the filename to extract structured metadata such as subject, grade, language, lesson number, Arabic title, and content field.  
  - Configuration: Custom JavaScript with regex matching filename patterns like `arabic_g12_ar_lesson01_...`. Cleans filename by removing extensions and suffixes.  
  - Variables Used: `$json.name`, `$binary.data.fileName`, regex groups for subject, grade, lang, lesson, title_ar, field.  
  - Connections: Output → Download file  
  - Edge Cases: Filename mismatch triggers an error with metadata `meta_error: 'filename_not_matched'`.  
  - Notes: Supports Arabic text normalization and tagging for curriculum metadata.  

- **Download file:**  
  - Type: Google Drive  
  - Role: Downloads the actual JSON file using the file ID from the trigger node.  
  - Configuration: Uses dynamic expressions to get file ID and filename from the previous node. Operation set to "download".  
  - Connections: Output → Vision JSON → Clean Text Chunks  
  - Credentials: Google Drive OAuth2 required.  
  - Edge Cases: File not found, permission denied, or download errors.  

- **Sticky Note1:**  
  - Content: Describes this block as "01 – Input & Metadata Extraction" and summarizes its functionality.  

---

#### 2.2 Vision JSON Parsing & Text Cleaning  

**Overview:**  
Processes the downloaded Google Vision JSON to extract the full Arabic text, normalizes characters, removes noise, and splits the text into clean chunks suitable for embedding generation.

**Nodes Involved:**  
- Vision JSON → Clean Text Chunks  
- Sticky Note2  

**Node Details:**  

- **Vision JSON → Clean Text Chunks:**  
  - Type: Code  
  - Role:  
    - Reads base64-encoded JSON binary content from the downloaded file.  
    - Extracts text from Google Vision API response fields (`responses[].fullTextAnnotation.text`, fallback `textAnnotations`).  
    - Normalizes Arabic text by removing diacritics, tatweel, and standardizing characters.  
    - Splits text into sentences and then chunks with configured max length (900 characters) and overlap (150).  
    - Generates metadata based on filename and lesson information.  
  - Configuration: Complex JavaScript logic for text extraction, normalization, chunking, and metadata assembly.  
  - Variables Used: Binary data from downloaded file, filename, custom functions for Arabic text normalization and splitting.  
  - Connections: Output → Insert into Pinecone Vector Store  
  - Edge Cases:  
    - No binary data or invalid JSON triggers error outputs.  
    - Missing fullTextAnnotation text causes specific error with hints.  
    - Empty chunks after cleaning also trigger error.  
  - Notes: Includes mapping for content fields and lesson titles.  

- **Sticky Note2:**  
  - Content: Describes this block as "02 – Vision Parsing & Text Cleaning" focusing on Arabic text extraction and chunking.  

---

#### 2.3 Embeddings Generation & Vector Storage  

**Overview:**  
Generates vector embeddings for each cleaned text chunk using OpenAI’s embedding model and inserts these vectors along with metadata into a Pinecone index, facilitating fast semantic search.

**Nodes Involved:**  
- Generate Embeddings (OpenAI)  
- Default Data Loader  
- Recursive Character Text Splitter  
- Insert into Pinecone Vector Store  
- Sticky Note3  

**Node Details:**  

- **Generate Embeddings (OpenAI):**  
  - Type: Langchain OpenAI Embeddings  
  - Role: Creates vector embeddings for text chunks using OpenAI API.  
  - Configuration: Uses default embedding model parameters; no special options set.  
  - Connections: AI output → Insert into Pinecone Vector Store (embedding input)  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API rate limits, authentication errors, or empty input text.  

- **Default Data Loader:**  
  - Type: Langchain Document Default Data Loader  
  - Role: Prepares documents for ingestion; here used to pass metadata and documents for vector insertion.  
  - Configuration: Metadata fields mapped from previous nodes; disables page splitting.  
  - Connections: AI output → Insert into Pinecone Vector Store (document input)  
  - Edge Cases: Missing metadata or malformed documents can cause insertion issues.  

- **Recursive Character Text Splitter:**  
  - Type: Langchain Recursive Character Text Splitter  
  - Role: Splits large text documents into smaller chunks for embedding.  
  - Configuration: Chunk size 1200 characters with 150 character overlap.  
  - Connections: AI output → Default Data Loader  
  - Edge Cases: Very small or empty documents may produce no chunks.  

- **Insert into Pinecone Vector Store:**  
  - Type: Langchain Pinecone Vector Store  
  - Role: Inserts embeddings and document metadata into Pinecone index.  
  - Configuration: Insert mode; namespace dynamically set from filename metadata; batch size 64 vectors per insert operation. Pinecone index configured with user’s index ID.  
  - Connections: Output → Move File to Archive  
  - Credentials: Pinecone API key required.  
  - Edge Cases: API errors, namespace mismatches, batch size issues.  

- **Sticky Note3:**  
  - Content: Describes the block as "03 – Embeddings & Vector Storage" responsible for RAG ingestion.  

---

#### 2.4 Archival of Processed Files  

**Overview:**  
After successful ingestion, moves the processed Google Drive file into a designated archive folder to prevent duplicate processing and maintain input folder cleanliness.

**Nodes Involved:**  
- Move File to Archive  
- Sticky Note4  

**Node Details:**  

- **Move File to Archive:**  
  - Type: Google Drive  
  - Role: Moves the processed file from the input folder to an archive folder specified by the user.  
  - Configuration: Uses dynamic file ID from trigger; moves file to folder ID `YOUR_ARCHIVE_FOLDER_ID`.  
  - Connections: None (end node)  
  - Credentials: Google Drive OAuth2 required.  
  - Edge Cases: Permission errors, folder not found, or file locking issues.  

- **Sticky Note4:**  
  - Content: Describes this block as "04 – Archive Processed Files" to keep workspace organized.  

---

### 3. Summary Table

| Node Name                    | Node Type                            | Functional Role                                    | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                              |
|------------------------------|------------------------------------|---------------------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------|
| Watch Drive Folder (new files) | Google Drive Trigger               | Detects new OCR JSON files in Google Drive folder | None                             | Filename → Lesson Metadata        | ## 01 – Input & Metadata Extraction This section watches a Google Drive folder for new files, downloads each JSON file, and extracts structured lesson metadata from the filename. It prepares the raw file and metadata for text extraction and RAG ingestion. |
| Filename → Lesson Metadata    | Code                               | Parses filename to extract structured lesson metadata | Watch Drive Folder (new files)   | Download file                    | (See above note for Input & Metadata Extraction block)                                                  |
| Download file                | Google Drive                       | Downloads the OCR JSON file                        | Filename → Lesson Metadata         | Vision JSON → Clean Text Chunks   | (See above note for Input & Metadata Extraction block)                                                  |
| Vision JSON → Clean Text Chunks | Code                             | Extracts and cleans Arabic text from JSON, splits into chunks | Download file                    | Insert into Pinecone Vector Store | ## 02 – Vision Parsing & Text Cleaning Parses Google Vision JSON, extracts full Arabic text, removes noise, normalizes characters, and generates clean text chunks with consistent structure for downstream RAG processing. |
| Recursive Character Text Splitter | Langchain Text Splitter           | Splits large text documents into chunks           | None (not connected in main flow) | Default Data Loader              | ## 03 – Embeddings & Vector Storage Generates vector embeddings for each cleaned text chunk and stores them, along with metadata, in a Pinecone index. This section forms the core of the RAG ingestion process for fast semantic search. |
| Default Data Loader           | Langchain Document Loader          | Loads documents with metadata for vector insertion | Recursive Character Text Splitter | Insert into Pinecone Vector Store | (See above note for Embeddings & Vector Storage block)                                                  |
| Generate Embeddings (OpenAI) | Langchain OpenAI Embeddings        | Generates vector embeddings from text chunks       | None (not connected in main flow) | Insert into Pinecone Vector Store | (See above note for Embeddings & Vector Storage block)                                                  |
| Insert into Pinecone Vector Store | Langchain Pinecone Vector Store | Inserts embeddings and metadata into Pinecone index | Vision JSON → Clean Text Chunks; Default Data Loader; Generate Embeddings (OpenAI) | Move File to Archive            | (See above note for Embeddings & Vector Storage block)                                                  |
| Move File to Archive          | Google Drive                       | Archives processed file to designated folder       | Insert into Pinecone Vector Store | None                            | ## 04 – Archive Processed Files Moves the processed Google Drive file into an archive folder to prevent duplicate ingestion and keep the input directory organized for future uploads. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger Node**  
   - Node type: Google Drive Trigger  
   - Set event to `fileCreated`.  
   - Configure to watch a specific folder by folder ID (`YOUR_INPUT_FOLDER_ID`).  
   - Set polling interval to every minute.  
   - Attach Google Drive OAuth2 credentials.  

2. **Add Code Node "Filename → Lesson Metadata"**  
   - Copy the provided JavaScript code that parses the filename to extract subject, grade, language, lesson number, Arabic title, field, and metadata tags.  
   - Use input from the trigger node (file metadata).  
   - Outputs structured metadata JSON for downstream use.  

3. **Add Google Drive Node "Download file"**  
   - Set operation to `download`.  
   - Use expression to set file ID dynamically from `"Filename → Lesson Metadata"` node output (e.g., `={{ $('Watch Drive Folder (new files)').first().json.id }}`).  
   - Optionally set file name for reference.  
   - Attach Google Drive OAuth2 credentials.  

4. **Add Code Node "Vision JSON → Clean Text Chunks"**  
   - Paste the detailed JavaScript code for reading the downloaded JSON binary, extracting text from Google Vision responses, normalizing Arabic, chunking text into ~900 character segments with 150 overlap, and assembling metadata for each chunk.  
   - Input comes from the "Download file" node's binary data.  
   - Output is an array of chunked documents with metadata.  

5. **Add Langchain Nodes for Text Splitting & Embedding**  
   - Add "Recursive Character Text Splitter" node with chunk size 1200 and overlap 150.  
   - Add "Default Data Loader" node configured to include all relevant metadata fields and set `splitPages` to false.  
   - Add "Generate Embeddings (OpenAI)" node with default embedding options.  
   - Connect nodes properly: Vision JSON → Clean Text Chunks → Insert into Pinecone Vector Store (embedding and document inputs) or via splitter and loader as needed.  

6. **Add Pinecone Node "Insert into Pinecone Vector Store"**  
   - Set mode to `insert`.  
   - Configure Pinecone index with your index name (`YOUR_PINECONE_INDEX`).  
   - Set namespace dynamically from metadata (`={{ $('Filename → Lesson Metadata').item.json.namespace }}`).  
   - Set embedding batch size to 64.  
   - Attach Pinecone API credentials.  

7. **Add Google Drive Node "Move File to Archive"**  
   - Operation: `move`.  
   - Use dynamic file ID from trigger node output.  
   - Set destination folder ID to your archive folder ID (`YOUR_ARCHIVE_FOLDER_ID`).  
   - Attach Google Drive OAuth2 credentials.  

8. **Connect the Workflow**  
   - Connect nodes as:  
     Watch Drive Folder (new files) → Filename → Lesson Metadata → Download file → Vision JSON → Clean Text Chunks → Insert into Pinecone Vector Store → Move File to Archive.  
   - Connect embedding and document input paths inside the Pinecone insert node as per Langchain requirements (embedding from OpenAI node, documents from Default Data Loader).  

9. **Add Sticky Note Nodes**  
   - Add descriptive sticky notes at strategic points to explain blocks, using the content provided in the workflow for clarity and documentation.  

10. **Configure Credentials**  
    - Setup Google Drive OAuth2 with proper scopes for reading, downloading, and moving files.  
    - Setup OpenAI API key for embeddings generation.  
    - Setup Pinecone API key with access to the target index.  

11. **Set Workflow Settings**  
    - Timezone: Africa/Cairo (or your preferred timezone).  
    - Execution order and caller policy as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| The workflow handles Arabic educational documents following a strict filename pattern to extract rich lesson metadata automatically.                 | Filename parsing regex and code in "Filename → Lesson Metadata" node.                |
| The text extraction supports Google Vision API output formats, including asynchronous PDF responses and Document AI JSON structures.                | Inside "Vision JSON → Clean Text Chunks" code logic.                                |
| The system normalizes Arabic text by removing diacritics, tatweel, and normalizing characters to improve embedding quality.                         | Text normalization functions in the code node.                                     |
| Embeddings are generated in batches to optimize Pinecone insertion throughput with a batch size of 64 vectors.                                       | Configured in the "Insert into Pinecone Vector Store" node parameters.               |
| Archiving processed files prevents reprocessing and maintains a clean input folder for ongoing ingestion.                                           | "Move File to Archive" node description.                                            |
| For more on vector databases and RAG workflows, see Pinecone docs: https://www.pinecone.io/docs/ and OpenAI embeddings guide: https://platform.openai.com/docs/guides/embeddings | Useful external resources for understanding vector search and embeddings.           |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---