Multi-Format Document Processing for RAG Chatbot with Google Drive & Supabase

https://n8nworkflows.xyz/workflows/multi-format-document-processing-for-rag-chatbot-with-google-drive---supabase-9933


# Multi-Format Document Processing for RAG Chatbot with Google Drive & Supabase

### 1. Workflow Overview

This workflow automates the ingestion and processing of multiple document formats uploaded to a specific Google Drive folder for use in a Retrieval-Augmented Generation (RAG) chatbot system. It monitors newly created files, detects their type, extracts text content accordingly, optionally converts certain documents to Google Docs format, splits large texts for better processing, generates embeddings using OpenAI, and finally stores the processed document vectors in a Supabase vector database. The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Watches a designated Google Drive folder for new file uploads.
- **1.2 File Metadata and Looping:** Captures file metadata, loops through files for individual processing.
- **1.3 File Type Detection and Conditional Processing:** Uses a switch node to route files based on MIME type for appropriate extraction or conversion.
- **1.4 Text Extraction and Processing:** Extracts text from PDFs, Excel, Google Docs, Word documents, and plain text files.
- **1.5 Document Aggregation and Summarization:** Aggregates and summarizes extracted data when necessary.
- **1.6 Text Splitting and Embedding Generation:** Splits large text documents recursively and generates OpenAI embeddings.
- **1.7 Data Insertion:** Inserts processed document vectors into Supabase vectorstore.
- **1.8 Cleanup:** Deletes temporary files in Google Drive after conversion.
- **1.9 Informational and Credit Notes:** Provides workflow comments and credits.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow when new files are created in a specified Google Drive folder.

**Nodes Involved:**  
- File Created

**Node Details:**  
- **File Created**  
  - Type: Google Drive Trigger  
  - Role: Watches for newly created files in a specific Google Drive folder (ID: `1M9h9OnDSWa0kV7_Yj7fdSFHhM57pMPX8`)  
  - Configuration: Trigger on `fileCreated` event, polling every minute, limited to specified folder  
  - Inputs: None (trigger node)  
  - Outputs: Emits metadata about new files (ID, MIME type, owner, timestamps)  
  - Potential Failures: Auth errors if Google Drive OAuth2 credentials expire; rate limiting from Drive API  

---

#### 1.2 File Metadata and Looping

**Overview:**  
Sets file metadata in variables and loops over multiple items for individual processing.

**Nodes Involved:**  
- Set File ID1  
- Loop Over Items

**Node Details:**  
- **Set File ID1**  
  - Type: Set  
  - Role: Extracts and stores `file_id` and `file_type` from the created file metadata for downstream use  
  - Key Expressions:  
    - `file_id` = `{{$('File Created').item.json.id}}`  
    - `file_type` = `{{$('File Created').item.json.mimeType}}`  
  - Inputs: File Created node output  
  - Outputs: Single item with file metadata  
  - Edge Cases: Missing or malformed file metadata may cause failures downstream

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each file individually in batch mode, useful if multiple files triggered simultaneously  
  - Inputs: Output from Set File ID1  
  - Outputs: Single file item streamed to downstream nodes  
  - Edge Cases: Large batch sizes may cause timeouts; single-file operation expected in normal use  

---

#### 1.3 File Type Detection and Conditional Processing

**Overview:**  
Routes files according to MIME type to the appropriate extraction or conversion flow.

**Nodes Involved:**  
- Switch2

**Node Details:**  
- **Switch2**  
  - Type: Switch  
  - Role: Checks `file_type` to route files to PDF extraction, text extraction, Excel extraction, or Google Doc conversion  
  - Conditions & Outputs:  
    - PDF: `application/pdf`  
    - Google Docs: `application/vnd.google-apps.document`  
    - Excel: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`  
    - Windows Word Doc (multiple MIME types)  
  - Inputs: Single file metadata item from Loop Over Items  
  - Outputs: Routes to corresponding extraction or conversion nodes  
  - Edge Cases: Unknown MIME types fall through to fallback output, possibly unhandled  

---

#### 1.4 Text Extraction and Processing

**Overview:**  
Extracts text content from files based on their format.

**Nodes Involved:**  
- Extract PDF Text  
- Extract from Text File  
- Extract from Excel  
- Convert to Google Doc1

**Node Details:**  
- **Extract PDF Text**  
  - Type: ExtractFromFile (PDF)  
  - Role: Extracts text content from PDF files  
  - Inputs: Routed from Switch2 PDF output  
  - Outputs: Text data for embedding  
  - Errors: Corrupt PDFs, large files causing timeouts  

- **Extract from Text File**  
  - Type: ExtractFromFile (Text)  
  - Role: Extracts plain text from Google Docs files  
  - Inputs: Routed from Switch2 Google Docs output  
  - Outputs: Text data for embedding  
  - Edge Cases: Empty documents; unsupported encodings  

- **Extract from Excel**  
  - Type: ExtractFromFile (XLSX)  
  - Role: Extracts text or cell data from Excel spreadsheets  
  - Inputs: Routed from Switch2 Excel output  
  - Outputs: Rows or concatenated data  
  - Connected to: Aggregate1 (aggregation of extracted rows)  
  - Edge Cases: Complex spreadsheets, formulas, or large files may cause extraction issues  

- **Convert to Google Doc1**  
  - Type: HTTP Request (Google Drive API copy file)  
  - Role: Converts Word documents to Google Docs format by copying with MIME type change  
  - Inputs: Routed from Switch2 Windows Doc outputs  
  - Outputs: New Google Doc file ID and name  
  - Credentials: Google Drive OAuth2  
  - Edge Cases: API quota limits, conversion failures, unsupported Word formats  
  - Followed by: Delete File node to remove original file  

---

#### 1.5 Document Aggregation and Summarization

**Overview:**  
Aggregates multiple extracted data items and summarizes text content before embedding.

**Nodes Involved:**  
- Aggregate1  
- Summarize1

**Node Details:**  
- **Aggregate1**  
  - Type: Aggregate  
  - Role: Combines all extracted data items from Excel extraction into one dataset  
  - Inputs: Extract from Excel output  
  - Outputs: Aggregated data to Summarize1  

- **Summarize1**  
  - Type: Summarize  
  - Role: Concatenates aggregated data fields into a single text string for embedding insertion  
  - Inputs: Aggregate1 output  
  - Outputs: Summarized concatenated text for insertion  

---

#### 1.6 Text Splitting and Embedding Generation

**Overview:**  
Splits large text documents recursively for manageable chunks and generates vector embeddings using OpenAI.

**Nodes Involved:**  
- Recursive Character Text Splitter  
- Enhanced Default Data Loader1  
- Embeddings OpenAI1

**Node Details:**  
- **Recursive Character Text Splitter**  
  - Type: LangChain Text Splitter (Recursive Character)  
  - Role: Splits text into chunks of 2000 characters with 200 characters overlap using markdown splitting  
  - Inputs: Text from previous extraction/summarization steps (implied connection)  
  - Outputs: Text chunks for embedding generation  

- **Enhanced Default Data Loader1**  
  - Type: LangChain Document Data Loader  
  - Role: Wraps split text chunks with metadata (file ID, version, creator, timestamps, folder path, file name, extension)  
  - Inputs: Text chunks from Recursive Character Text Splitter  
  - Outputs: Fully prepared document objects for embedding insertion  

- **Embeddings OpenAI1**  
  - Type: LangChain OpenAI Embeddings  
  - Role: Generates vector embeddings for documents  
  - Credentials: OpenAI API key required  
  - Inputs: Document objects from Data Loader  
  - Outputs: Embeddings for vector store insertion  
  - Edge Cases: API rate limits, token limits, network failures  

---

#### 1.7 Data Insertion

**Overview:**  
Inserts processed document embeddings and summarized text into Supabase vectorstore.

**Nodes Involved:**  
- Insert into Supabase Vectorstore1

**Node Details:**  
- **Insert into Supabase Vectorstore1**  
  - Type: LangChain Vector Store (Supabase)  
  - Role: Inserts embeddings and document metadata into Supabase vector database table named `documents`  
  - Credentials: Supabase API key  
  - Inputs: From Summarize1, Extract PDF Text, Extract from Text File, Embeddings OpenAI1, and Enhanced Default Data Loader1 nodes via different input types (`main`, `ai_embedding`, `ai_document`)  
  - Edge Cases: Database connectivity issues, quota limits, schema mismatches  

---

#### 1.8 Cleanup

**Overview:**  
Deletes temporary converted files from Google Drive to maintain storage hygiene.

**Nodes Involved:**  
- Delete File

**Node Details:**  
- **Delete File**  
  - Type: Google Drive  
  - Role: Deletes original files after conversion to Google Docs to prevent duplication  
  - Inputs: Output from Convert to Google Doc1 (file ID)  
  - Credentials: Google Drive OAuth2  
  - Edge Cases: Permissions errors, file already deleted, API rate limits  

---

#### 1.9 Informational and Credit Notes

**Overview:**  
Provides descriptive and credit information about the workflow.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note

**Node Details:**  
- **Sticky Note1**  
  - Type: Sticky Note  
  - Content: Describes the workflow trigger and processing logic in Indonesian language  
  - Position: Top left for overview  

- **Sticky Note**  
  - Type: Sticky Note  
  - Content: Credits Nate Herk as the original author and provides a link to his community:  
    [https://www.skool.com/ai-automation-society-plus/about](https://www.skool.com/ai-automation-society-plus/about)  

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                                         | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                           |
|------------------------------|-----------------------------------|---------------------------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| File Created                 | Google Drive Trigger               | Watches for new files in designated Google Drive folder | None                          | Set File ID1                    | # Watch Trigger (Drive) - Files Created. Multiple files uploaded to monitored folder -> processing   |
| Set File ID1                 | Set                               | Extracts file_id and file_type from trigger output      | File Created                  | Loop Over Items                 | # Watch Trigger (Drive) - Files Created. Multiple files uploaded to monitored folder -> processing   |
| Loop Over Items              | SplitInBatches                    | Iterates over files individually                         | Set File ID1                  | Switch2, Download File1         | # Watch Trigger (Drive) - Files Created. Multiple files uploaded to monitored folder -> processing   |
| Switch2                     | Switch                           | Routes files by MIME type to specific extraction paths  | Loop Over Items               | Extract PDF Text, Extract from Text File, Extract from Excel, Convert to Google Doc1 | # Watch Trigger (Drive) - Files Created. Multiple files uploaded to monitored folder -> processing   |
| Extract PDF Text             | ExtractFromFile (PDF)              | Extracts text from PDF files                             | Switch2 (PDF output)          | Insert into Supabase Vectorstore1 |                                                                                                     |
| Extract from Text File       | ExtractFromFile (Text)             | Extracts text from Google Docs files                     | Switch2 (Text File output)    | Insert into Supabase Vectorstore1 |                                                                                                     |
| Extract from Excel           | ExtractFromFile (XLSX)             | Extracts data from Excel spreadsheets                    | Switch2 (Excel output)        | Aggregate1                     |                                                                                                     |
| Convert to Google Doc1       | HTTP Request (Google Drive API)   | Converts Word docs to Google Docs                        | Switch2 (Windows Doc outputs) | Delete File                   |                                                                                                     |
| Delete File                 | Google Drive                      | Deletes original Word files after conversion            | Convert to Google Doc1        | Loop Over Items                |                                                                                                     |
| Aggregate1                  | Aggregate                        | Aggregates extracted Excel data                          | Extract from Excel            | Summarize1                    |                                                                                                     |
| Summarize1                  | Summarize                       | Concatenates aggregated data for embedding insertion     | Aggregate1                   | Insert into Supabase Vectorstore1 |                                                                                                     |
| Download File1              | Google Drive                     | Downloads files with Google Docs conversion option       | Loop Over Items              | Loop Over Items (feedback)      | # Watch Trigger (Drive) - Files Created. Multiple files uploaded to monitored folder -> processing   |
| Recursive Character Text Splitter | LangChain Text Splitter (Recursive) | Splits large texts into manageable chunks               | (Implied from extraction)    | Enhanced Default Data Loader1   |                                                                                                     |
| Enhanced Default Data Loader1 | LangChain Document Data Loader   | Prepares documents with metadata for embedding           | Recursive Character Text Splitter | Insert into Supabase Vectorstore1 |                                                                                                     |
| Embeddings OpenAI1          | LangChain OpenAI Embeddings       | Generates vector embeddings                               | Enhanced Default Data Loader1 | Insert into Supabase Vectorstore1 |                                                                                                     |
| Insert into Supabase Vectorstore1 | LangChain VectorStore Supabase  | Inserts vectors and metadata into Supabase               | Summarize1, Extract PDF Text, Extract from Text File, Embeddings OpenAI1, Enhanced Default Data Loader1 | None                          |                                                                                                     |
| Sticky Note1                | Sticky Note                      | Workflow overview note                                   | None                         | None                          | # Watch Trigger (Drive) - Files Created. Multiple files uploaded to monitored folder -> processing   |
| Sticky Note                 | Sticky Note                      | Workflow credit note                                    | None                         | None                          | Credits Nate Herk. Link: https://www.skool.com/ai-automation-society-plus/about                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node:**
   - Name: `File Created`
   - Event: `fileCreated`
   - Poll interval: every minute
   - Folder to watch: Google Drive folder ID `1M9h9OnDSWa0kV7_Yj7fdSFHhM57pMPX8`
   - Credentials: Google Drive OAuth2 account

2. **Add Set node:**
   - Name: `Set File ID1`
   - Parameters:  
     - `file_id` = `{{$('File Created').item.json.id}}`  
     - `file_type` = `{{$('File Created').item.json.mimeType}}`
   - Connect `File Created` → `Set File ID1`

3. **Add SplitInBatches node:**
   - Name: `Loop Over Items`
   - Default batch size
   - Connect `Set File ID1` → `Loop Over Items`

4. **Add Switch node:**
   - Name: `Switch2`
   - Field to evaluate: `{{$('Set File ID1').item.json.file_type}}`
   - Rules:  
     - PDF: equals `application/pdf`  
     - Google Docs: equals `application/vnd.google-apps.document`  
     - Excel: equals `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`  
     - Windows Word Doc (3 rules for MIME types):  
       - `application/vnd.openxmlformats-officedocument.wordprocessingml.document`  
       - `application/msword`  
       - `application/vnd.ms-word`
   - Connect `Loop Over Items` → `Switch2`

5. **Add ExtractFromFile nodes for each type:**
   - `Extract PDF Text` (operation: pdf)
   - `Extract from Text File` (operation: text)
   - `Extract from Excel` (operation: xlsx)
   - Connect respective outputs from `Switch2` to these extraction nodes

6. **Add HTTP Request node for conversion:**
   - Name: `Convert to Google Doc1`
   - Method: POST
   - URL: `https://www.googleapis.com/drive/v3/files/{{ $('Set File ID1').item.json.file_id }}/copy`
   - Body parameters:  
     - `name` = `{{ $('Set File ID1').item.json.name }}`  
     - `mimeType` = `application/vnd.google-apps.document`
   - Credentials: Google Drive OAuth2
   - Connect Windows Doc outputs from `Switch2` to this node

7. **Add Google Drive node:**
   - Name: `Delete File`
   - Operation: `deleteFile`
   - File ID: `{{$('Set File ID1').item.json.file_id}}`
   - Credentials: Google Drive OAuth2
   - Connect `Convert to Google Doc1` → `Delete File`

8. **Add Aggregate node:**
   - Name: `Aggregate1`
   - Operation: aggregate all item data
   - Connect `Extract from Excel` → `Aggregate1`

9. **Add Summarize node:**
   - Name: `Summarize1`
   - Field to summarize: concatenate `data` fields
   - Connect `Aggregate1` → `Summarize1`

10. **Add LangChain Recursive Character Text Splitter node:**
    - Name: `Recursive Character Text Splitter`
    - Chunk size: 2000 characters
    - Chunk overlap: 200 characters
    - Split code type: markdown
    - Connect extracted text outputs (implied connection)

11. **Add LangChain Document Data Loader node:**
    - Name: `Enhanced Default Data Loader1`
    - Metadata fields:  
      - file_id, version (v1), creator, created_at, last_modified, folder_path ("DOCUMENTS"), file_name, file_extension  
    - JSON data: Expression to use extracted or concatenated data  
    - Connect `Recursive Character Text Splitter` → `Enhanced Default Data Loader1`

12. **Add LangChain OpenAI Embeddings node:**
    - Name: `Embeddings OpenAI1`
    - Credentials: OpenAI API key
    - Connect `Enhanced Default Data Loader1` → `Embeddings OpenAI1`

13. **Add LangChain VectorStore Supabase node:**
    - Name: `Insert into Supabase Vectorstore1`
    - Mode: insert
    - Table name: `documents`
    - Credentials: Supabase API
    - Connect outputs from:  
      - `Summarize1`  
      - `Extract PDF Text`  
      - `Extract from Text File`  
      - `Embeddings OpenAI1`  
      - `Enhanced Default Data Loader1`

14. **Add Google Drive Download node (optional):**
    - Name: `Download File1`
    - Operation: download with Google Docs conversion to plain text
    - Credentials: Google Drive OAuth2
    - Connect `Loop Over Items` → `Download File1` → back to `Loop Over Items` (feedback loop)

15. **Add Sticky Notes for documentation:**
    - `Sticky Note1` with workflow overview content near input nodes
    - `Sticky Note` crediting Nate Herk with link: https://www.skool.com/ai-automation-society-plus/about

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow triggers on new files uploaded to a designated Google Drive folder and processes them | Workflow Trigger Description (Sticky Note1)                     |
| Original workflow by Nate Herk — Join his community for more AI automation resources           | https://www.skool.com/ai-automation-society-plus/about          |
| Requires Google Drive OAuth2 credentials configured with access to monitored folders           | Credential setup note                                            |
| Requires valid OpenAI API key for embeddings generation                                       | Credential setup note                                            |
| Requires Supabase API key with permissions to write to `documents` vectorstore table           | Credential setup note                                            |
| Supports multiple document formats: PDF, Google Docs, Excel, Word (Office Open XML and legacy) | File type handling note                                          |
| Converts Word documents to Google Docs format for uniform text extraction                      | Conversion node rationale                                        |
| Text splitting uses recursive character method with markdown split code                        | Optimizes input size for embedding; reduces token overflow risk |
| Aggregation and summarization applied for Excel extraction to concatenate cell data            | Data preparation step                                           |
| Deletes original Word files after conversion to avoid duplicates in Google Drive               | Cleanup step to reduce clutter                                   |

---

**Disclaimer:**  
The text provided is derived solely from an automated n8n workflow implementation. The workflow complies fully with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.