Keep RAG System Updated with Google Drive File Changes to Supabase Vector DB

https://n8nworkflows.xyz/workflows/keep-rag-system-updated-with-google-drive-file-changes-to-supabase-vector-db-9934


# Keep RAG System Updated with Google Drive File Changes to Supabase Vector DB

### 1. Workflow Overview

This workflow automates the process of keeping a Retrieval-Augmented Generation (RAG) system’s vector database up to date by monitoring Google Drive for file changes. Upon detecting an updated file in a designated folder, it processes the file according to its type, extracts its textual content, splits and summarizes it as needed, then uploads the processed data into a Supabase vector database for AI-powered retrieval.

The workflow’s logic is organized into these main blocks:

- **1.1 Trigger & File Identification:** Detect updated files in a specific Google Drive folder and set key file metadata.
- **1.2 Validation & Cleanup:** Validate file type and freshness, then delete any old vector data related to the file in Supabase.
- **1.3 Version Management:** Increment the document version number via OpenAI.
- **1.4 File Processing & Conversion:** Depending on file type, extract text directly or convert files (e.g., MS Word) to Google Docs for processing.
- **1.5 Text Extraction & Aggregation:** Extract text from PDFs, Excel, plain text, or Google Docs, aggregate content if needed.
- **1.6 Text Splitting & Metadata Enrichment:** Split large texts recursively into chunks and add metadata for vector insertion.
- **1.7 Vector Storage:** Insert the processed documents and embeddings into Supabase vector database for AI retrieval.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger & File Identification

**Overview:**  
This block listens to Google Drive for any file updates in a specified folder and captures file metadata to prepare for downstream processing.

**Nodes Involved:**  
- File Updated  
- Set File ID

**Node Details:**

- **File Updated**  
  - Type: Google Drive Trigger  
  - Configuration: Watches a specific Google Drive folder (ID: `1M9h9OnDSWa0kV7_Yj7fdSFHhM57pMPX8`) for file update events every minute.  
  - Input: Trigger event (no input).  
  - Output: JSON containing file metadata such as `id`, `mimeType`, `name`, `owners`, `createdTime`, `modifiedTime`.  
  - Potential Failures: Google OAuth errors, network timeouts.  
  - Notes: Only files within the specified folder trigger workflow execution.

- **Set File ID**  
  - Type: Set Node  
  - Configuration: Assigns two variables:  
    - `file_id` from the trigger’s file ID  
    - `file_type` from the trigger’s file MIME type  
  - Input: Output from "File Updated".  
  - Output: JSON with assigned variables for downstream nodes.  
  - Edge Cases: If the trigger does not provide these fields, expressions may fail.

---

#### 2.2 Validation & Cleanup

**Overview:**  
This block verifies if the updated file is a Google Doc and has been modified within the last 60 seconds; if not, the workflow stops early. If validated, it deletes any existing Supabase database entries associated with this file to prepare for fresh insertion.

**Nodes Involved:**  
- If  
- Delete Old Doc Rows  
- Limit

**Node Details:**

- **If**  
  - Type: If Node  
  - Configuration: Checks two conditions using AND combinator:  
    1. File MIME type equals `application/vnd.google-apps.document` (Google Docs)  
    2. File was created less than 60 seconds ago (using timestamp difference)  
  - Input: Output from "Set File ID".  
  - Output: True branch continues; False branch ends workflow.  
  - Edge Cases: Date parsing errors, missing fields, or files that are not Google Docs prevent further execution.

- **Delete Old Doc Rows**  
  - Type: Supabase Node (Delete operation)  
  - Configuration: Deletes all rows in the `documents` table where `metadata->>file_id` matches the current `file_id`.  
  - Input: True branch output from "If".  
  - Output: Confirmation of deletion, always outputs data.  
  - Potential Failures: Supabase API authentication errors, query syntax issues.  
  - Credentials: Supabase API configured.

- **Limit**  
  - Type: Limit Node  
  - Configuration: Limits data passed downstream; no specific limits set (default 1).  
  - Input: Output from "Delete Old Doc Rows".  
  - Output: Passes limited data.  
  - Role: Controls flow pacing or avoids parallel execution issues.

---

#### 2.3 Version Management

**Overview:**  
This block increments the document version number using OpenAI, preparing the metadata for the updated document to be stored.

**Nodes Involved:**  
- Set Version  
- Loop Over Items1

**Node Details:**

- **Set Version**  
  - Type: OpenAI (Chat Completion) Node  
  - Configuration: Uses GPT-4o-mini model to parse incoming version string (e.g., "v1") and respond with the next version (e.g., "v2").  
  - Input: Output from "Limit" node; expects current version in `$json.metadata.version`.  
  - Output: JSON with a new field `"version"` indicating incremented version number.  
  - Edge Cases: Incorrect or missing version format in metadata may confuse model.  
  - Credentials: OpenAI API configured.

- **Loop Over Items1**  
  - Type: Split In Batches  
  - Configuration: Splits incoming data into batches; default batch size.  
  - Input: Output from "Set Version".  
  - Output: Passes individual item batches downstream.  
  - Role: Handles processing of multiple items if they arise.

---

#### 2.4 File Processing & Conversion

**Overview:**  
This block routes the workflow based on file type. It processes PDFs, Google Docs, Excel spreadsheets directly, or converts MS Word documents to Google Docs before extraction.

**Nodes Involved:**  
- Switch  
- Extract PDF Text1  
- Extract from Text File1  
- Extract from Excel1  
- Convert to Google Doc2  
- Delete File1  
- Download File

**Node Details:**

- **Switch**  
  - Type: Switch Node  
  - Configuration: Routes based on `file_type` variable to one of six outputs:  
    - PDF (`application/pdf`)  
    - Google Doc (`application/vnd.google-apps.document`)  
    - Excel (`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`)  
    - Windows Doc (three MIME types for MS Word formats)  
  - Input: Output from "Loop Over Items1".  
  - Output: Redirects to appropriate extraction or conversion node.  
  - Edge Cases: Unhandled MIME types default to fallback output (index 2).  
  - Notes: Ensures format compatibility for text extraction.

- **Extract PDF Text1**  
  - Type: Extract From File Node (PDF operation)  
  - Configuration: Extracts text content from PDF files.  
  - Input: From "Switch" PDF output.  
  - Output: Extracted text data.  
  - Edge Cases: Password-protected or corrupted PDFs might fail extraction.

- **Extract from Text File1**  
  - Type: Extract From File Node (Text operation)  
  - Configuration: Extracts text from Google Docs (plain text).  
  - Input: From "Switch" Google Doc output.  
  - Output: Extracted text data.  
  - Edge Cases: Empty or malformed documents.

- **Extract from Excel1**  
  - Type: Extract From File Node (XLSX operation)  
  - Configuration: Extracts text from Excel spreadsheets.  
  - Input: From "Switch" Excel output.  
  - Output: Extracted data rows.  
  - Edge Cases: Complex Excel formulas or unsupported features.

- **Convert to Google Doc2**  
  - Type: HTTP Request Node  
  - Configuration: Uses Google Drive API v3 to copy and convert MS Word files to Google Docs format.  
  - Request URL: `https://www.googleapis.com/drive/v3/files/{{file_id}}/copy`  
  - Method: POST  
  - Body Parameters: Sets name and MIME type for conversion.  
  - Input: From "Switch" Windows Doc outputs.  
  - Output: New Google Doc file metadata.  
  - Edge Cases: API errors, insufficient permissions.  
  - Credentials: Google Drive OAuth2 API.

- **Delete File1**  
  - Type: Google Drive Node (Delete File operation)  
  - Configuration: Deletes the original MS Word file after conversion.  
  - Input: From "Convert to Google Doc2".  
  - Output: Confirmation of deletion.  
  - Edge Cases: Permission denied, file not found.

- **Download File**  
  - Type: Google Drive Node (Download operation)  
  - Configuration: Downloads the file content as plain text for further processing.  
  - Input: From "Loop Over Items1" (alternative path).  
  - Output: File binary content.  
  - Credentials: Google Drive OAuth2 API.

---

#### 2.5 Text Extraction & Aggregation

**Overview:**  
Processes extracted content by aggregating data (for Excel files) and summarizing concatenated text to prepare for embedding.

**Nodes Involved:**  
- Aggregate  
- Summarize

**Node Details:**

- **Aggregate**  
  - Type: Aggregate Node  
  - Configuration: Aggregates all incoming item data into a single item.  
  - Input: From "Extract from Excel1".  
  - Output: Aggregated data object.  
  - Role: Consolidates multiple rows/text segments.

- **Summarize**  
  - Type: Summarize Node  
  - Configuration: Concatenates all values from the field named `data`.  
  - Input: From "Aggregate".  
  - Output: Single concatenated text string under `concatenated_data`.  
  - Role: Prepares text for vector embedding.

---

#### 2.6 Text Splitting & Metadata Enrichment

**Overview:**  
Splits large text documents into smaller chunks with overlap for effective vector indexing, and enriches each chunk with metadata such as file ID, version, creator, timestamps, and file properties.

**Nodes Involved:**  
- Recursive Character Text Splitter1  
- Enhanced Default Data Loader2

**Node Details:**

- **Recursive Character Text Splitter1**  
  - Type: LangChain Recursive Character Text Splitter  
  - Configuration: Splits text into chunks of 2000 characters with 200 characters overlap.  
  - Input: Text from summarization or extraction nodes.  
  - Output: Array of text chunks.  
  - Edge Cases: Very small texts might result in one chunk.

- **Enhanced Default Data Loader2**  
  - Type: LangChain Document Default Data Loader  
  - Configuration:  
    - Metadata fields assigned from variables: `file_id`, version (from OpenAI output), creator name, creation and modification timestamps, folder path ("DOCUMENTS"), file name, and MIME type.  
    - Input JSON data is expression-based, selects `data`, `text`, or `concatenated_data` fields.  
  - Input: From text splitter node.  
  - Output: Documents enriched with metadata for vector insertion.

---

#### 2.7 Vector Storage

**Overview:**  
Inserts the processed document chunks and their embeddings into the Supabase vector database for retrieval.

**Nodes Involved:**  
- Insert into Supabase Vectorstore  
- Embeddings OpenAI2 (connected as AI embedding node)

**Node Details:**

- **Insert into Supabase Vectorstore**  
  - Type: LangChain VectorStore Supabase Node  
  - Configuration: Inserts documents in "insert" mode into the `documents` table of Supabase. Uses a query named `match_documents` for indexing.  
  - Input: Accepts AI documents, embeddings, or raw document data from previous nodes.  
  - Output: Confirmation of insertion.  
  - Credentials: Supabase API configured.  
  - Edge Cases: API or network failures, data schema mismatches.

- **Embeddings OpenAI2**  
  - Type: LangChain Embeddings OpenAI Node  
  - Configuration: Generates vector embeddings for text chunks using OpenAI embeddings service.  
  - Input: Text chunks from splitter or loader.  
  - Output: Embeddings passed to Supabase insertion node.  
  - Credentials: OpenAI API configured.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                          | Input Node(s)          | Output Node(s)                | Sticky Note                                                                                             |
|----------------------------|------------------------------------|----------------------------------------|-----------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| File Updated               | Google Drive Trigger                | Trigger on file update event            | (Trigger)             | Set File ID                  | # Watch Trigger (Drive) - Files Updated<br>## Workflow ini jalan jika ada File yang diupdate/diedit/dimodifikasi.<br># Cara Kerja<br>## Cek file yang barusan diupdate -> cek tipe/format file nya lalu konvert jika perlu -> exkstrak teksnya -> upload ke vectord database |
| Set File ID                | Set                                | Assign file ID and MIME type            | File Updated          | If                          |                                                                                                        |
| If                        | If                                 | Validate file type and freshness        | Set File ID           | Delete Old Doc Rows (True)   |                                                                                                        |
| Delete Old Doc Rows        | Supabase (Delete)                   | Remove old entries for the file         | If                    | Limit                       |                                                                                                        |
| Limit                     | Limit                              | Limit downstream data                    | Delete Old Doc Rows    | Set Version                 |                                                                                                        |
| Set Version               | OpenAI Chat Completion              | Increment document version number       | Limit                  | Loop Over Items1             |                                                                                                        |
| Loop Over Items1            | Split In Batches                   | Batch processing for each item           | Set Version            | Switch, Download File        |                                                                                                        |
| Switch                    | Switch                            | Route processing by file MIME type      | Loop Over Items1       | Extract PDF Text1, Extract from Text File1, Extract from Excel1, Convert to Google Doc2 |                                                                                                        |
| Extract PDF Text1          | Extract From File (PDF)             | Extract text from PDF files              | Switch (PDF output)    | Insert into Supabase Vectorstore |                                                                                                        |
| Extract from Text File1     | Extract From File (Text)            | Extract text from Google Docs            | Switch (Google Doc)    | Insert into Supabase Vectorstore |                                                                                                        |
| Extract from Excel1         | Extract From File (XLSX)            | Extract data from Excel spreadsheets     | Switch (Excel)         | Aggregate                   |                                                                                                        |
| Aggregate                 | Aggregate                         | Aggregate Excel extracted rows           | Extract from Excel1    | Summarize                   |                                                                                                        |
| Summarize                 | Summarize                        | Concatenate aggregated data               | Aggregate              | Insert into Supabase Vectorstore |                                                                                                        |
| Convert to Google Doc2      | HTTP Request (Google Drive API)    | Convert MS Word files to Google Docs     | Switch (Windows Doc)   | Delete File1                |                                                                                                        |
| Delete File1              | Google Drive (Delete File)          | Delete original MS Word file after conversion | Convert to Google Doc2 | -                          |                                                                                                        |
| Download File             | Google Drive (Download)             | Download file content for processing     | Loop Over Items1       | Loop Over Items1            |                                                                                                        |
| Recursive Character Text Splitter1 | LangChain Text Splitter         | Split text into manageable chunks         | Summarize or extract nodes | Enhanced Default Data Loader2 |                                                                                                        |
| Enhanced Default Data Loader2 | LangChain Document Data Loader      | Enrich documents with metadata            | Recursive Character Text Splitter1 | Insert into Supabase Vectorstore |                                                                                                        |
| Embeddings OpenAI2         | LangChain Embeddings OpenAI         | Generate vector embeddings for text       | Enhanced Default Data Loader2 | Insert into Supabase Vectorstore |                                                                                                        |
| Insert into Supabase Vectorstore | LangChain Vectorstore Supabase      | Insert processed documents into vector DB | Extract nodes, Embeddings OpenAI2, Enhanced Default Data Loader2 | -                          |                                                                                                        |
| Sticky Note3              | Sticky Note                      | Explanation note on main workflow trigger | -                     | -                          | # Watch Trigger (Drive) - Files Updated<br>## Workflow ini jalan jika ada File yang diupdate/diedit/dimodifikasi.<br># Cara Kerja<br>## Cek file yang barusan diupdate -> cek tipe/format file nya lalu konvert jika perlu -> exkstrak teksnya -> upload ke vectord database |
| Sticky Note               | Sticky Note                      | Credits for original workflow author     | -                     | -                          | ## CREDITS<br>### This Workflow Is Originally by Nate Herk.<br>[Check his community here](https://www.skool.com/ai-automation-society-plus/about) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger Node ("File Updated"):**  
   - Set event to `fileUpdated`.  
   - Watch folder ID: `1M9h9OnDSWa0kV7_Yj7fdSFHhM57pMPX8`.  
   - Poll every minute.  
   - Connect Google Drive OAuth2 credentials.

2. **Add a Set Node ("Set File ID"):**  
   - Assign two variables:  
     - `file_id` = `{{$node["File Updated"].json["id"]}}`  
     - `file_type` = `{{$node["File Updated"].json["mimeType"]}}`  
   - Connect input from "File Updated".

3. **Add an If Node ("If"):**  
   - Condition 1: `file_type` equals `application/vnd.google-apps.document`.  
   - Condition 2: File creation time within last 60 seconds (expression comparing current timestamp and `createdTime`).  
   - Connect input from "Set File ID".  
   - True branch continues, False branch ends workflow.

4. **Add Supabase Delete Node ("Delete Old Doc Rows"):**  
   - Operation: Delete.  
   - Table: `documents`.  
   - Filter: `metadata->>file_id LIKE '%{{$node["Set File ID"].json["file_id"]}}%'`.  
   - Connect input from "If" True branch.  
   - Configure Supabase API credentials.

5. **Add Limit Node ("Limit"):**  
   - Default limit (1).  
   - Connect input from "Delete Old Doc Rows".

6. **Add OpenAI Chat Completion Node ("Set Version"):**  
   - Model: `gpt-4o-mini`.  
   - System prompt: instruct to increment version number (e.g., "v1" to "v2").  
   - Input message content: `{{$json.metadata.version}}` from previous data.  
   - Enable JSON output.  
   - Connect input from "Limit".  
   - Configure OpenAI credentials.

7. **Add Split In Batches Node ("Loop Over Items1"):**  
   - Default batch size.  
   - Connect input from "Set Version".

8. **Add Switch Node ("Switch"):**  
   - Rules based on `file_type`:  
     - PDF: `application/pdf`  
     - Google Doc: `application/vnd.google-apps.document`  
     - Excel: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`  
     - Three Windows Doc MIME types for MS Word files.  
   - Connect input from "Loop Over Items1".

9. **Add Extract From File Nodes:**  
   - "Extract PDF Text1": operation `pdf`, connects from Switch PDF output.  
   - "Extract from Text File1": operation `text`, connects from Switch Google Doc output.  
   - "Extract from Excel1": operation `xlsx`, connects from Switch Excel output.

10. **Add Aggregate Node ("Aggregate"):**  
    - Aggregates all extracted Excel rows.  
    - Connect input from "Extract from Excel1".

11. **Add Summarize Node ("Summarize"):**  
    - Concatenate field `data`.  
    - Connect input from "Aggregate".

12. **Add HTTP Request Node ("Convert to Google Doc2"):**  
    - POST `https://www.googleapis.com/drive/v3/files/{{file_id}}/copy`.  
    - Body parameters: `name` and `mimeType` set to Google Docs format.  
    - Connect outputs from all Windows Doc Switch outputs.  
    - Configure Google Drive OAuth2 credentials.

13. **Add Google Drive Node ("Delete File1"):**  
    - Operation: Delete File.  
    - File ID from "Convert to Google Doc2" output.  
    - Connect input from "Convert to Google Doc2".

14. **Add Google Drive Node ("Download File"):**  
    - Operation: Download file as plain text.  
    - File ID from "Set File ID".  
    - Connect alternative path from "Loop Over Items1".

15. **Add Recursive Character Text Splitter Node:**  
    - Chunk size 2000, overlap 200.  
    - Connect input from "Summarize" or extraction nodes.

16. **Add Enhanced Default Data Loader Node:**  
    - Metadata fields:  
      - `file_id`, `version` (from OpenAI output), `creator` (file owner), `created_at`, `last_modified`, `folder_path` ("DOCUMENTS"), `file_name`, `file_extension`.  
    - Input data: expression selecting extracted or concatenated text field.  
    - Connect input from text splitter.

17. **Add Embeddings OpenAI Node ("Embeddings OpenAI2"):**  
    - Connect input from data loader.  
    - Configure OpenAI credentials.

18. **Add Supabase Insert Node ("Insert into Supabase Vectorstore"):**  
    - Mode: Insert.  
    - Table: `documents`.  
    - Query Name: `match_documents`.  
    - Connect inputs from extraction nodes and embeddings node as appropriate.  
    - Configure Supabase API credentials.

19. **Add Sticky Note Nodes:**  
    - One for workflow trigger explanation (positioned near start).  
    - One for workflow credits linking to Nate Herk’s community.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow triggers on Google Drive file updates in a specific folder, then processes and updates vector DB accordingly. | Sticky Note near "File Updated" node explains main workflow trigger and logic.                                |
| Credits: This workflow is originally by Nate Herk. Community link: [Skool AI Automation Society Plus](https://www.skool.com/ai-automation-society-plus/about) | Sticky Note near end node for credits.                                                                        |
| Version incrementing leverages OpenAI GPT-4o-mini model with a simple prompt to handle version strings like "v1", "v2", etc. | See "Set Version" node configuration.                                                                         |
| Supabase vector database table used: `documents` with a `match_documents` query for vector matching. | Supabase nodes configured with this table and query name.                                                     |
| File conversion for MS Word documents uses Google Drive API file copy method with MIME type conversion. | HTTP Request node "Convert to Google Doc2".                                                                   |
| Text splitting uses LangChain Recursive Character Text Splitter with chunk size 2000 and 200 overlap. | Node "Recursive Character Text Splitter1".                                                                    |
| The workflow handles multiple file types: Google Docs, PDFs, Excel, MS Word (converted), and plain text extraction. | Switch node routes accordingly.                                                                                |

---

**Disclaimer:**  
The text analyzed derives solely from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected data. All processed information is legal and publicly accessible.