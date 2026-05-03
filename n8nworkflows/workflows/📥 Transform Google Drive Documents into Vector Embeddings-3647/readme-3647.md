📥 Transform Google Drive Documents into Vector Embeddings

https://n8nworkflows.xyz/workflows/---transform-google-drive-documents-into-vector-embeddings-3647


# 📥 Transform Google Drive Documents into Vector Embeddings

### 1. Workflow Overview

This workflow automates the ingestion of documents stored in a specified Google Drive folder by converting them into vector embeddings suitable for semantic search or Retrieval-Augmented Generation (RAG) AI agents. It supports multiple document formats (PDF, TXT, JSON), processes them through a text splitting and embedding pipeline using OpenAI’s `text-embedding-3-small` model, and stores the resulting vectors in a Postgres database with the PGVector extension. After successful processing, files are moved to a designated “vectorized” folder to prevent reprocessing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Triggering:** Initiates the workflow either manually or on a scheduled basis and searches the source Google Drive folder for new files.
- **1.2 File Download and Type Routing:** Downloads each file and routes it based on MIME type to the appropriate extraction method.
- **1.3 Content Extraction:** Extracts raw text content from files depending on their format (PDF, plain text, JSON).
- **1.4 Text Splitting and Embedding:** Splits extracted text into manageable chunks and generates vector embeddings using OpenAI.
- **1.5 Vector Storage and File Management:** Inserts embeddings into the PGVector-enabled Postgres database and moves processed files to a “vectorized” folder to avoid duplication.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:** This block starts the workflow either manually or on a schedule and retrieves all files from the configured Google Drive source folder.
- **Nodes Involved:**  
  - `When clicking ‘Test workflow’`  
  - `Schedule Trigger`  
  - `Search Folder`  
  - `Loop Over Items`

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for on-demand processing.  
    - Configuration: No parameters; triggers workflow on user action.  
    - Inputs: None  
    - Outputs: Connects to `Search Folder`  
    - Edge Cases: None specific; user must manually trigger.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at 3 AM by default.  
    - Configuration: Interval set to trigger at hour 3 daily.  
    - Inputs: None  
    - Outputs: Connects to `Search Folder`  
    - Edge Cases: Timezone considerations; ensure server time matches expected timezone.

  - **Search Folder**  
    - Type: Google Drive node (fileFolder resource)  
    - Role: Lists all files in the specified Google Drive folder (source folder).  
    - Configuration:  
      - Folder ID set to the source folder (e.g., `1mBHrP8UzUnfn3dj_3QS1r0XhQQyVPAGX`)  
      - Returns all files found.  
    - Inputs: Trigger from manual or schedule nodes  
    - Outputs: Connects to `Loop Over Items`  
    - Credentials: Google Drive OAuth2  
    - Edge Cases: Folder ID must be correct; API quota limits; empty folder returns no items.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each file item retrieved from the folder.  
    - Configuration: Default batch size (processes items one by one).  
    - Inputs: Files from `Search Folder`  
    - Outputs: Two outputs:  
      - First output: empty (used for flow control)  
      - Second output: connects to `Download File` for processing each file.  
    - Edge Cases: Large number of files may slow processing; batch size can be adjusted.

---

#### 2.2 File Download and Type Routing

- **Overview:** Downloads each file from Google Drive and routes it based on MIME type to the appropriate extraction node.
- **Nodes Involved:**  
  - `Download File`  
  - `Switch`

- **Node Details:**

  - **Download File**  
    - Type: Google Drive node  
    - Role: Downloads the actual file content for processing.  
    - Configuration:  
      - File ID dynamically set from the current item’s `id`.  
      - Operation: `download`  
    - Inputs: From `Loop Over Items` (second output)  
    - Outputs: Connects to `Switch`  
    - Credentials: Google Drive OAuth2  
    - Edge Cases: File may be missing or deleted; download failures; large files may timeout.

  - **Switch**  
    - Type: Switch node  
    - Role: Routes files based on their MIME type to the correct extraction method.  
    - Configuration:  
      - Three outputs based on MIME type:  
        - `pdf` for `application/pdf`  
        - `text` for `text/plain`  
        - `json` for `application/json`  
      - Uses expression `{{$binary["data"].mimeType}}` to determine MIME type.  
    - Inputs: From `Download File`  
    - Outputs:  
      - `pdf` → `Extract from PDF`  
      - `text` → `Extract from Text`  
      - `json` → `Extract from JSON`  
    - Edge Cases: Files with unsupported MIME types are dropped (no default case); MIME type detection depends on Google Drive metadata.

---

#### 2.3 Content Extraction

- **Overview:** Extracts textual content from files according to their format to prepare for embedding.
- **Nodes Involved:**  
  - `Extract from PDF`  
  - `Extract from Text`  
  - `Extract from JSON`

- **Node Details:**

  - **Extract from PDF**  
    - Type: ExtractFromFile node  
    - Role: Extracts text from PDF files.  
    - Configuration: Operation set to `pdf`.  
    - Inputs: From `Switch` (pdf output)  
    - Outputs: Connects to `Postgres PGVector Store`  
    - Edge Cases: Complex PDFs with images or scanned pages may not extract well; no OCR included.

  - **Extract from Text**  
    - Type: ExtractFromFile node  
    - Role: Extracts plain text from text files.  
    - Configuration: Operation set to `text`.  
    - Inputs: From `Switch` (text output)  
    - Outputs: Connects to `Postgres PGVector Store`  
    - Edge Cases: Encoding issues may arise; assumes UTF-8 text.

  - **Extract from JSON**  
    - Type: ExtractFromFile node  
    - Role: Extracts text content from JSON files.  
    - Configuration: Operation set to `fromJson`.  
    - Inputs: From `Switch` (json output)  
    - Outputs: Connects to `Postgres PGVector Store`  
    - Edge Cases: JSON structure must be compatible; complex nested JSON may require custom parsing.

---

#### 2.4 Text Splitting and Embedding

- **Overview:** Splits extracted text into chunks and generates vector embeddings using OpenAI’s embedding model.
- **Nodes Involved:**  
  - `Recursive Character Text Splitter`  
  - `Default Data Loader`  
  - `Embeddings OpenAI`

- **Node Details:**

  - **Recursive Character Text Splitter**  
    - Type: LangChain Text Splitter node  
    - Role: Splits large text into smaller overlapping chunks for better embedding.  
    - Configuration:  
      - Chunk overlap set to 50 characters.  
      - Default splitting options.  
    - Inputs: Not directly connected in the main flow (likely used internally or in sub-workflows).  
    - Outputs: Connects to `Default Data Loader`  
    - Edge Cases: Very short texts may not split; overlapping chunks increase embedding count.

  - **Default Data Loader**  
    - Type: LangChain Document Data Loader  
    - Role: Prepares documents for embedding by loading and formatting text chunks.  
    - Configuration: Default options.  
    - Inputs: From `Recursive Character Text Splitter` (ai_textSplitter output)  
    - Outputs: Connects to `Postgres PGVector Store` (ai_document output)  
    - Edge Cases: Assumes input is properly split text chunks.

  - **Embeddings OpenAI**  
    - Type: LangChain Embeddings node  
    - Role: Generates vector embeddings using OpenAI’s `text-embedding-3-small` model.  
    - Configuration:  
      - Model: `text-embedding-3-small`  
      - No additional options.  
    - Inputs: From `Postgres PGVector Store` (ai_embedding input)  
    - Outputs: Connects back to `Postgres PGVector Store` (ai_embedding output)  
    - Credentials: OpenAI API Key  
    - Edge Cases: API rate limits; invalid API key; network timeouts.

---

#### 2.5 Vector Storage and File Management

- **Overview:** Inserts embeddings into the Postgres PGVector database and moves processed files to a “vectorized” folder.
- **Nodes Involved:**  
  - `Postgres PGVector Store`  
  - `Move File`

- **Node Details:**

  - **Postgres PGVector Store**  
    - Type: LangChain Vector Store node for PGVector  
    - Role: Inserts vector embeddings into the configured Postgres table.  
    - Configuration:  
      - Mode: `insert`  
      - Table name: `n8n_vectors_wfs`  
      - Collection: `n8n_wfs` (used if collection support enabled)  
    - Inputs:  
      - Main input: from extraction nodes (`Extract from PDF`, `Extract from Text`, `Extract from JSON`)  
      - AI embedding input: from `Embeddings OpenAI`  
      - AI document input: from `Default Data Loader`  
    - Outputs: Connects to `Move File`  
    - Credentials: Postgres with PGVector extension  
    - Edge Cases: Database connection failures; duplicate entries; schema mismatches.

  - **Move File**  
    - Type: Google Drive node  
    - Role: Moves processed files to a designated “vectorized” folder to avoid reprocessing.  
    - Configuration:  
      - File ID: dynamically set from the current item’s `id` in `Loop Over Items`.  
      - Drive ID: `My Drive`  
      - Folder ID: target folder for processed files (e.g., `1Re6vg-PZxBoUU6sTRDbGs-77bAJ40u8F`)  
      - Operation: `move`  
    - Inputs: From `Postgres PGVector Store`  
    - Outputs: Connects back to `Loop Over Items` (first output) for flow control.  
    - Credentials: Google Drive OAuth2  
    - Edge Cases: Permission errors; folder ID must exist; file may be locked or deleted.

---

### 3. Summary Table

| Node Name                  | Node Type                                      | Functional Role                                  | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                      |
|----------------------------|------------------------------------------------|-------------------------------------------------|-------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                                 | Manual start of workflow                         | None                          | Search Folder                |                                                                                                                  |
| Schedule Trigger           | Schedule Trigger                               | Scheduled daily trigger at 3 AM                  | None                          | Search Folder                |                                                                                                                  |
| Search Folder             | Google Drive (fileFolder resource)             | Lists files in source Google Drive folder        | When clicking ‘Test workflow’, Schedule Trigger | Loop Over Items             |                                                                                                                  |
| Loop Over Items           | SplitInBatches                                 | Iterates over each file item                      | Search Folder                 | Download File (2nd output), (1st output unused) |                                                                                                                  |
| Download File             | Google Drive                                   | Downloads file content                            | Loop Over Items (2nd output)  | Switch                      |                                                                                                                  |
| Switch                    | Switch                                         | Routes files by MIME type                         | Download File                 | Extract from PDF, Extract from Text, Extract from JSON |                                                                                                                  |
| Extract from PDF          | ExtractFromFile                                | Extracts text from PDF files                      | Switch (pdf output)           | Postgres PGVector Store     |                                                                                                                  |
| Extract from Text         | ExtractFromFile                                | Extracts text from plain text files              | Switch (text output)          | Postgres PGVector Store     |                                                                                                                  |
| Extract from JSON         | ExtractFromFile                                | Extracts text from JSON files                     | Switch (json output)          | Postgres PGVector Store     |                                                                                                                  |
| Recursive Character Text Splitter | LangChain Text Splitter                      | Splits text into chunks for embedding            | Not directly connected in main flow | Default Data Loader         |                                                                                                                  |
| Default Data Loader       | LangChain Document Data Loader                  | Loads and formats text chunks                     | Recursive Character Text Splitter | Postgres PGVector Store     |                                                                                                                  |
| Embeddings OpenAI         | LangChain Embeddings                            | Generates vector embeddings using OpenAI         | Postgres PGVector Store (ai_embedding input) | Postgres PGVector Store     |                                                                                                                  |
| Postgres PGVector Store   | LangChain Vector Store (PGVector)               | Inserts embeddings into Postgres PGVector table  | Extract from PDF/Text/JSON, Embeddings OpenAI, Default Data Loader | Move File                   |                                                                                                                  |
| Move File                 | Google Drive                                   | Moves processed files to “vectorized” folder     | Postgres PGVector Store       | Loop Over Items (1st output) |                                                                                                                  |
| Sticky Note               | Sticky Note                                    | License and author information                    | None                          | None                        | Creative Commons License: CC BY-SA 4.0; Author: AlexK1919; License details: https://creativecommons.org/licenses/by-sa/4.0/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Set up Google Drive OAuth2 credentials in n8n.
   - Set up OpenAI API credentials.
   - Configure Postgres credentials with PGVector extension enabled.

2. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.
   - Add a **Schedule Trigger** node named `Schedule Trigger` configured to run daily at 3 AM (set `triggerAtHour` to 3).

3. **Add Google Drive Search Node:**
   - Add a **Google Drive** node named `Search Folder`.
   - Set resource to `fileFolder`.
   - Set operation to `list`.
   - Set folder ID to your source folder ID (e.g., `1mBHrP8UzUnfn3dj_3QS1r0XhQQyVPAGX`).
   - Connect both trigger nodes to this node.

4. **Add Loop Over Items Node:**
   - Add a **SplitInBatches** node named `Loop Over Items`.
   - Connect `Search Folder` output to `Loop Over Items` input.

5. **Add Google Drive Download Node:**
   - Add a **Google Drive** node named `Download File`.
   - Set operation to `download`.
   - Set file ID to `={{ $json.id }}` (dynamic from current item).
   - Connect second output of `Loop Over Items` to this node.

6. **Add Switch Node for MIME Type Routing:**
   - Add a **Switch** node named `Switch`.
   - Add three rules based on expression `{{$binary["data"].mimeType}}`:
     - Equals `application/pdf` → output `pdf`
     - Equals `text/plain` → output `text`
     - Equals `application/json` → output `json`
   - Connect `Download File` output to `Switch`.

7. **Add Extraction Nodes:**
   - Add **ExtractFromFile** nodes:
     - `Extract from PDF` with operation `pdf`.
     - `Extract from Text` with operation `text`.
     - `Extract from JSON` with operation `fromJson`.
   - Connect `Switch` outputs to corresponding extraction nodes.

8. **Add Postgres PGVector Store Node:**
   - Add a **LangChain Vector Store PGVector** node named `Postgres PGVector Store`.
   - Set mode to `insert`.
   - Set table name to your PGVector-enabled table (e.g., `n8n_vectors_wfs`).
   - Enable collection and set collection name (e.g., `n8n_wfs`).
   - Connect outputs of all three extraction nodes to this node.
   - Configure Postgres credentials.

9. **Add Embeddings OpenAI Node:**
   - Add a **LangChain Embeddings OpenAI** node named `Embeddings OpenAI`.
   - Set model to `text-embedding-3-small`.
   - Connect `Postgres PGVector Store` node’s `ai_embedding` input to this node.
   - Connect this node’s output back to `Postgres PGVector Store` node’s `ai_embedding` output.
   - Configure OpenAI credentials.

10. **Add Recursive Character Text Splitter Node:**
    - Add a **LangChain Text Splitter Recursive Character Text Splitter** node named `Recursive Character Text Splitter`.
    - Set chunk overlap to 50.
    - Connect this node’s output to `Default Data Loader`.

11. **Add Default Data Loader Node:**
    - Add a **LangChain Document Default Data Loader** node named `Default Data Loader`.
    - Connect `Recursive Character Text Splitter` output to this node.
    - Connect this node’s output to `Postgres PGVector Store` node’s `ai_document` input.

12. **Add Move File Node:**
    - Add a **Google Drive** node named `Move File`.
    - Set operation to `move`.
    - Set file ID to `={{ $('Loop Over Items').item.json.id }}` (dynamic from current item).
    - Set drive ID to `My Drive`.
    - Set folder ID to your processed folder ID (e.g., `1Re6vg-PZxBoUU6sTRDbGs-77bAJ40u8F`).
    - Connect `Postgres PGVector Store` output to `Move File`.
    - Connect `Move File` output back to the first output of `Loop Over Items` to continue batch processing.

13. **Final Connections:**
    - Connect `When clicking ‘Test workflow’` and `Schedule Trigger` to `Search Folder`.
    - Connect `Search Folder` to `Loop Over Items`.
    - Connect `Loop Over Items` (2nd output) to `Download File`.
    - Connect `Download File` to `Switch`.
    - Connect `Switch` outputs to extraction nodes.
    - Connect extraction nodes to `Postgres PGVector Store`.
    - Connect `Postgres PGVector Store` to `Move File`.
    - Connect `Move File` back to `Loop Over Items` (1st output).

14. **Optional: Add Sticky Note**
    - Add a **Sticky Note** node with license and author information.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0) license applies.            | License details: https://creativecommons.org/licenses/by-sa/4.0/                                |
| Author: AlexK1919                                                                                   | Author website: https://www.alexk1919.com                                                       |
| Workflow supports multiple file types: PDF, TXT, JSON.                                             | Can be extended with additional extractors for DOCX, Markdown, HTML, or OCR preprocessing.      |
| Embeddings use OpenAI’s `text-embedding-3-small` model.                                            | Requires valid OpenAI API key.                                                                  |
| Postgres database must have PGVector extension enabled.                                            | Used for storing vector embeddings for semantic search and RAG applications.                     |
| Google Drive OAuth2 credentials required for all Google Drive nodes (search, download, move).      | Ensure correct scopes and permissions are granted.                                              |
| Scheduled trigger defaults to 3 AM daily but can be customized.                                    | Adjust schedule as needed in the `Schedule Trigger` node.                                       |
| Files are moved after processing to avoid duplication.                                            | Target folder ID must be set correctly in the `Move File` node.                                 |

---

This structured documentation provides a detailed understanding of the workflow’s architecture, node configurations, and operational logic, enabling users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.