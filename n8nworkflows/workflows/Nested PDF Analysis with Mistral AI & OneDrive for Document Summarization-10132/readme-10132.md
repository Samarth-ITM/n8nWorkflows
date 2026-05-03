Nested PDF Analysis with Mistral AI & OneDrive for Document Summarization

https://n8nworkflows.xyz/workflows/nested-pdf-analysis-with-mistral-ai---onedrive-for-document-summarization-10132


# Nested PDF Analysis with Mistral AI & OneDrive for Document Summarization

### 1. Workflow Overview

This workflow automates the search, extraction, analysis, and structured summarization of PDF documents stored in a specified OneDrive folder and its nested subfolders (up to 8 levels deep). It is designed to process potentially large document sets, extract text content, and leverage advanced AI models (Mistral AI) for document summarization and data extraction. The workflow outputs structured insights into a data table for further use or reporting.

Logical blocks are grouped as follows:

- **1.1 Initialization & Folder Discovery**: Trigger and search for the main folder, then recursively retrieve subfolders up to 8 levels.
- **1.2 File Filtering & Dataset Comparison**: Identify PDF files among folder items and compare datasets to detect new or changed files.
- **1.3 File Processing Loop**: Sequentially process batches of files under size constraints, downloading and extracting text from PDFs.
- **1.4 AI-Based Document Analysis**: Use Mistral Cloud models and LangChain agents to generate document overviews and extract structured information.
- **1.5 Data Aggregation & Storage**: Aggregate AI outputs and insert structured results into a predefined data table.
- **1.6 Control & Error Handling**: Includes split batches, merges, and conditional checks to ensure workflow robustness.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Folder Discovery

**Overview:**  
This block triggers the workflow on schedule, searches for the main OneDrive folder by name, and retrieves folder contents up to eight nested levels to cover subfolders.

**Nodes Involved:**  
- Schedule Trigger1  
- Search for Main Folder  
- Get items in a folder1 → Get items in a folder9 (eight nodes)  
- If PDF 1 → If PDF 8 (eight conditional nodes)  
- Merge

**Node Details:**

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 20:00 (8 PM)  
  - Configuration: Fixed trigger hour 20  
  - Edge Cases: Trigger timing issues if server time zone mismatches; retries not applicable here.

- **Search for Main Folder**  
  - Type: Microsoft OneDrive Node (search operation)  
  - Role: Locate the main root folder by unique folder name ("PdM Test Folder")  
  - Configuration: Search query set to main folder name  
  - Edge Cases: If multiple folders with similar names, may return multiple results; folder name uniqueness is critical (see sticky note).  
  - Retry enabled on failure

- **Get items in a folder1 through Get items in a folder9**  
  - Type: Microsoft OneDrive (list folder contents)  
  - Role: Recursively fetch subfolders and files at each nested level  
  - Configuration: Uses dynamic folderId based on prior output `$json.id`  
  - Retry enabled on failure  
  - Edge Cases: Limited to 8 levels; if deeper nesting exists, workflow must be extended with additional nodes.

- **If PDF 1 through If PDF 8**  
  - Type: If Node (conditional check)  
  - Role: Filters items by MIME type to select only PDFs (`application/pdf`) at each folder level  
  - Configuration: Checks `$json.file.mimeType` equals `application/pdf`  
  - Edge Cases: Some files might have incorrect MIME types; non-PDFs are excluded.

- **Merge**  
  - Type: Merge Node (multiple inputs)  
  - Role: Combines filtered PDF items from each folder depth into a single stream for processing  
  - Configuration: Number of inputs set to 8 corresponding to the 8 folder levels  
  - Edge Cases: Input data mismatch or missing inputs if folders are empty.

---

#### 2.2 File Filtering & Dataset Comparison

**Overview:**  
This block compares newly found PDF files to existing records in a data table to identify new or updated documents for processing.

**Nodes Involved:**  
- Get row(s) (data table retrieval)  
- Loop Over Items  
- Set File ID 1  
- Compare Datasets  
- Loop Over Items 2  
- Set File ID 2  
- Loop Over Items 3  
- Set File ID 3

**Node Details:**

- **Get row(s)**  
  - Type: Data Table Node (get operation)  
  - Role: Retrieves existing document metadata rows from a data table (ID: "55TlXtWO8i9pkFIr")  
  - Configuration: Limit 5000 rows  
  - Edge Cases: Large data retrieval may impact performance; pagination not shown.

- **Loop Over Items, Loop Over Items 2, Loop Over Items 3**  
  - Type: SplitInBatches  
  - Role: Processes items in manageable batches; prevents resource overload  
  - Configuration: Batch size defaults (not explicitly set)  
  - Edge Cases: Batch size impacts throughput; resetting behavior varies.

- **Set File ID 1, Set File ID 2, Set File ID 3**  
  - Type: Set Node  
  - Role: Assigns or reformats file metadata fields (name, file_id, ID, Path) for downstream processing  
  - Configuration: Uses expressions to format file identifiers and paths; e.g., concatenating folder path segments with file names for unique IDs  
  - Edge Cases: Path splitting assumes a minimum folder depth; missing segments may cause errors.

- **Compare Datasets**  
  - Type: Compare Datasets Node  
  - Role: Compares current folder files to existing data table rows by "name" and "Path" fields to detect new or changed files  
  - Configuration: Merge by fields "name" and "Path"  
  - Edge Cases: Fields must be consistent and normalized; discrepancies in casing or path format may cause false mismatches.

---

#### 2.3 File Processing Loop

**Overview:**  
This block iterates over filtered and verified PDF files, applying a file size filter, downloading files, and extracting text content.

**Nodes Involved:**  
- If Size  
- Loop Over Items 2  
- Loop Over Items 3  
- 2nd Loop Over Items1  
- Download file  
- Extract PDF Text

**Node Details:**

- **If Size**  
  - Type: If Node  
  - Role: Filters files by size, allowing only those under 10,000,000 bytes (~10MB) for processing  
  - Configuration: Checks if `$json.size` < 10,000,000  
  - Edge Cases: Large files are skipped and not processed; consider increasing size limit if needed.

- **2nd Loop Over Items1, Loop Over Items 2, Loop Over Items 3**  
  - Type: SplitInBatches  
  - Role: Further batch processing for controlled execution on downloads and text extraction  
  - Configuration: Default batching options  
  - Edge Cases: Batching helps avoid API rate limits or memory issues.

- **Download file**  
  - Type: Microsoft OneDrive Node (download operation)  
  - Role: Downloads the actual PDF file by file ID for text extraction  
  - Configuration: Uses `$json.ID` for file identifier  
  - Error Handling: On error, node continues workflow execution (continueErrorOutput)  
  - Edge Cases: File access errors, rate limits, or missing files.

- **Extract PDF Text**  
  - Type: Extract From File Node  
  - Role: Extracts textual content from downloaded PDF files  
  - Configuration: Operation set to "pdf"  
  - Error Handling: On error, continues workflow  
  - Edge Cases: Corrupt or scanned-only PDFs may yield no or poor text extraction.

---

#### 2.4 AI-Based Document Analysis

**Overview:**  
This block uses Mistral Cloud AI and LangChain agents to generate structured document summaries and extract relevant metadata.

**Nodes Involved:**  
- Overview (LangChain Agent)  
- Structured Output Parser1  
- Overview LLM Chain  
- Document Information (LangChain Agent)  
- Structured Output Parser  
- Document LLM Chain  
- Mistral Cloud Chat Model

**Node Details:**

- **Mistral Cloud Chat Model**  
  - Type: AI Language Model (Mistral)  
  - Role: Central AI model used by downstream chains and agents  
  - Configuration: Model "mistral-small-latest" with maxTokens 32000, supporting large document contexts  
  - Edge Cases: API rate limits, token limits, network errors.

- **Overview (LangChain Agent)**  
  - Type: LangChain Agent Node  
  - Role: Generates a professional, impartial, markdown-formatted overview of the extracted document text  
  - Configuration: Includes strict instructions for formatting output in three sections: Executive Summary, Key Findings & Data Points, Scope and Context  
  - Input: Document text extracted from PDF  
  - Edge Cases: Input text too large or empty; agent must strictly adhere to output format.

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the Overview text output into a JSON object with fields: summary, key_findings (array), and scope  
  - Configuration: Manual JSON schema enforcing required fields and types; autoFix enabled  
  - Edge Cases: Parsing failures if output format does not exactly match schema.

- **Overview LLM Chain**  
  - Type: LangChain Chain LLM  
  - Role: Validates and reformats the Overview output to ensure correctness and completeness, enforcing empty defaults if missing fields  
  - Configuration: Custom system message instructing strict JSON return with no extra commentary  
  - Edge Cases: AI inconsistencies; fallback to empty fields.

- **Document Information (LangChain Agent)**  
  - Type: LangChain Agent  
  - Role: Extracts structured data including relevant date, location (city, state/country), and personal/professional info from document text  
  - Configuration: System message enforces extraction of a single most relevant date and structured location and personal info objects  
  - Edge Cases: Ambiguous dates or missing data; agent must return empty values if unavailable.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses output of Document Information agent into JSON format matching required schema  
  - Configuration: Manual schema with nested objects for location and personal info, all fields required but can be empty strings  
  - Edge Cases: Parsing errors if AI output is malformed.

- **Document LLM Chain**  
  - Type: LangChain Chain LLM  
  - Role: Validates and reformats extracted document information JSON, ensuring empty defaults if fields are missing  
  - Configuration: Similar to Overview LLM Chain with strict JSON output instruction  
  - Edge Cases: Same as Overview LLM Chain.

---

#### 2.5 Data Aggregation & Storage

**Overview:**  
Aggregates the processed AI outputs and inserts structured data rows into the configured data table for record-keeping and further analysis.

**Nodes Involved:**  
- Merge1  
- Aggregate  
- Insert row

**Node Details:**

- **Merge1**  
  - Type: Merge Node (two inputs)  
  - Role: Combines the outputs of the Overview and Document Information LLM Chains into a single data object for insertion  
  - Configuration: Default merging (no special options)  
  - Edge Cases: Mismatched data or empty inputs.

- **Aggregate**  
  - Type: Aggregate Node  
  - Role: Aggregates all item data into a single batch for insertion  
  - Configuration: Aggregates all incoming data items into one output  
  - Edge Cases: Large volume may affect performance.

- **Insert row**  
  - Type: Data Table Node (insert operation)  
  - Role: Inserts new rows into the data table with mapped columns for Date, Path, Scope, Summary, Location, File_Name, and Key_Findings  
  - Configuration: Column mappings use expressions to extract values from merged AI outputs and downloaded file metadata  
  - Edge Cases: Data mismatches if column names or data formats change; ensure data table schema matches.

---

#### 2.6 Control & Error Handling

**Overview:**  
Nodes managing batch splitting, conditional flows, retries, and error continuation to maintain workflow stability during execution.

**Nodes Involved:**  
- SplitInBatches nodes (Loop Over Items, Loop Over Items 2, Loop Over Items 3, 2nd Loop Over Items1)  
- If Nodes (If PDF 1–8, If Size)  
- Microsoft OneDrive retryOnFail enabled nodes (Get items in a folder*, Search for Main Folder)  
- Download file and Extract PDF Text with continue on error enabled

**Node Details:**

- **SplitInBatches**  
  - Controls batch sizes to avoid timeouts and overloads

- **If Nodes**  
  - Filter and control flow based on file type and size

- **RetryOnFail**  
  - Enables automatic retries on transient OneDrive API errors

- **ContinueErrorOutput**  
  - Allows workflow to proceed despite errors in file download or text extraction, avoiding complete failure

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                  | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                                                                                            |
|-------------------------|----------------------------------|-------------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1       | Schedule Trigger                 | Starts workflow daily at 20:00                   | —                                | Search for Main Folder, Get row(s) | ## Start Here: Enter main folder name; limited to 8 folder depth; folder name must be unique to avoid multiple matches                                               |
| Search for Main Folder  | Microsoft OneDrive (search)      | Finds main folder by name                         | Schedule Trigger1                 | Get items in a folder1             | ## File Search Structure                                                                                                                                              |
| Get items in a folder1  | Microsoft OneDrive (list folder) | Gets first-level folder contents                  | Search for Main Folder            | If PDF 1                          |                                                                                                                                                                      |
| If PDF 1                | If Node                         | Filters PDF files at level 1                      | Get items in a folder1            | Merge, Get items in a folder2      |                                                                                                                                                                      |
| Get items in a folder2  | Microsoft OneDrive (list folder) | Gets second-level folder contents                 | If PDF 1 (false branch)           | If PDF 2                          |                                                                                                                                                                      |
| If PDF 2                | If Node                         | Filters PDF files at level 2                      | Get items in a folder2            | Merge, Get items in a folder3      |                                                                                                                                                                      |
| Get items in a folder3  | Microsoft OneDrive (list folder) | Gets third-level folder contents                  | If PDF 2 (false branch)           | If PDF 3                          |                                                                                                                                                                      |
| If PDF 3                | If Node                         | Filters PDF files at level 3                      | Get items in a folder3            | Merge, Get items in a folder4      |                                                                                                                                                                      |
| Get items in a folder4  | Microsoft OneDrive (list folder) | Gets fourth-level folder contents                 | If PDF 3 (false branch)           | If PDF 4                          |                                                                                                                                                                      |
| If PDF 4                | If Node                         | Filters PDF files at level 4                      | Get items in a folder4            | Merge, Get items in a folder5      |                                                                                                                                                                      |
| Get items in a folder5  | Microsoft OneDrive (list folder) | Gets fifth-level folder contents                  | If PDF 4 (false branch)           | If PDF 5                          |                                                                                                                                                                      |
| If PDF 5                | If Node                         | Filters PDF files at level 5                      | Get items in a folder5            | Merge, Get items in a folder6      |                                                                                                                                                                      |
| Get items in a folder6  | Microsoft OneDrive (list folder) | Gets sixth-level folder contents                  | If PDF 5 (false branch)           | If PDF 6                          |                                                                                                                                                                      |
| If PDF 6                | If Node                         | Filters PDF files at level 6                      | Get items in a folder6            | Merge, Get items in a folder7      |                                                                                                                                                                      |
| Get items in a folder7  | Microsoft OneDrive (list folder) | Gets seventh-level folder contents                | If PDF 6 (false branch)           | If PDF 7                          |                                                                                                                                                                      |
| If PDF 7                | If Node                         | Filters PDF files at level 7                      | Get items in a folder7            | Merge, Get items in a folder9      |                                                                                                                                                                      |
| Get items in a folder9  | Microsoft OneDrive (list folder) | Gets eighth-level folder contents                 | If PDF 7 (false branch)           | If PDF 8                          |                                                                                                                                                                      |
| If PDF 8                | If Node                         | Filters PDF files at level 8                      | Get items in a folder9            | Merge                            |                                                                                                                                                                      |
| Merge                   | Merge Node                      | Combines all PDF file lists from folder levels  | If PDF 1–If PDF 8 (true branches) | If Size                         |                                                                                                                                                                      |
| If Size                 | If Node                         | Filters files under 10MB for processing           | Merge                            | Loop Over Items 2                 |                                                                                                                                                                      |
| Loop Over Items 2       | SplitInBatches                  | Batch processing of filtered files                | If Size                         | Compare Datasets                 |                                                                                                                                                                      |
| Compare Datasets        | Compare Datasets                | Compares current files with existing data         | Loop Over Items 2                 | Loop Over Items 3, Set File ID 2   |                                                                                                                                                                      |
| Loop Over Items 3       | SplitInBatches                  | Batch processing of new/changed files              | Compare Datasets                 | 2nd Loop Over Items1             |                                                                                                                                                                      |
| 2nd Loop Over Items1    | SplitInBatches                  | Batch processing before download                    | Loop Over Items 3                 | Download file                   |                                                                                                                                                                      |
| Download file           | Microsoft OneDrive (download)  | Downloads PDF file for text extraction             | 2nd Loop Over Items1              | Extract PDF Text, 2nd Loop Over Items1 |                                                                                                                                                                      |
| Extract PDF Text        | Extract From File              | Extracts text from PDF                              | Download file                    | Document Information, Overview, 2nd Loop Over Items1 |                                                                                                                                                                      |
| Document Information    | LangChain Agent                | Extracts Structured Metadata (date, location, personal info) | Extract PDF Text                | Document LLM Chain              | ## AI Review: Modify to add fields as needed; update LLM Chain and output parser accordingly                                                                        |
| Document LLM Chain      | LangChain Chain LLM            | Validates and reformats extracted metadata         | Document Information             | Merge1                         |                                                                                                                                                                      |
| Overview                | LangChain Agent                | Generates document overview summary in markdown    | Extract PDF Text                | Structured Output Parser1        |                                                                                                                                                                      |
| Structured Output Parser1 | LangChain Output Parser      | Parses Overview text to JSON                         | Overview                        | Overview LLM Chain              |                                                                                                                                                                      |
| Overview LLM Chain      | LangChain Chain LLM            | Validates and reformats overview JSON               | Structured Output Parser1        | Merge1                         |                                                                                                                                                                      |
| Merge1                  | Merge Node                    | Combines Overview and Document metadata             | Document LLM Chain, Overview LLM Chain | Aggregate                    |                                                                                                                                                                      |
| Aggregate               | Aggregate Node                 | Aggregates all AI outputs for insertion              | Merge1                         | Insert row                    |                                                                                                                                                                      |
| Insert row              | Data Table Insert             | Inserts structured data into data table              | Aggregate                      | 2nd Loop Over Items1            | ## Data Table: Customize columns as needed; verify Insert Row node mapping matches your columns                                                                    |
| Get row(s)              | Data Table Get                | Retrieves existing records for comparison            | Schedule Trigger1               | Loop Over Items                 | ## Data Table                                                                                                                                                        |
| Loop Over Items         | SplitInBatches                | Batch processing of existing records                  | Get row(s)                     | Compare Datasets, Set File ID 1  |                                                                                                                                                                      |
| Set File ID 1           | Set Node                      | Formats file metadata for comparison                  | Loop Over Items                 | Loop Over Items                 |                                                                                                                                                                      |
| Set File ID 2           | Set Node                      | Formats file metadata for comparison                  | Loop Over Items 2               | Loop Over Items 2               |                                                                                                                                                                      |
| Set File ID 3           | Set Node                      | Formats file metadata for comparison                  | Loop Over Items 3               | Loop Over Items 3               |                                                                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 20:00 (8 PM).

2. **Create a Microsoft OneDrive node (Search for Main Folder)**  
   - Operation: Search  
   - Resource: Folder  
   - Query: Enter your main folder name (e.g., "PdM Test Folder")  
   - Enable "Retry On Fail".

3. **Create eight Microsoft OneDrive nodes (Get items in a folder1 to Get items in a folder9)**  
   - Operation: List Folder Contents  
   - Resource: Folder  
   - Folder ID: Set dynamically to `{{$json["id"]}}` from previous node  
   - Enable "Retry On Fail"  
   - Chain each subsequent folder node to the false branch of the preceding If PDF node.

4. **Create eight If nodes (If PDF 1 to If PDF 8)**  
   - Condition: `$json.file.mimeType` equals `"application/pdf"`  
   - Connect each "true" output to the corresponding Merge input and "false" output to the next Get items in a folder node.

5. **Create a Merge node with 8 inputs**  
   - Purpose: Combine PDF files from all folder depths.

6. **Create an If node (If Size)**  
   - Condition: `$json.size` less than `10000000` (10MB)  
   - Connect Merge node output to this.

7. **Create SplitInBatches nodes: Loop Over Items 2, Loop Over Items 3, 2nd Loop Over Items1**  
   - Use default batch sizes  
   - Connect If Size true output to Loop Over Items 2  
   - Connect Loop Over Items 2's first output to Compare Datasets  
   - Connect Compare Datasets third output (new files) to Loop Over Items 3  
   - Connect Loop Over Items 3 first output to 2nd Loop Over Items1.

8. **Create Get row(s) Data Table node**  
   - Operation: Get  
   - Data Table ID: Your target data table (e.g., "55TlXtWO8i9pkFIr")  
   - Limit: 5000  
   - Connect Schedule Trigger1 output to this node.

9. **Create SplitInBatches node (Loop Over Items)**  
   - Connect Get row(s) output to Loop Over Items  
   - Connect Loop Over Items first output to Compare Datasets first input.

10. **Create three Set nodes (Set File ID 1, 2, 3)**  
    - Configure to assign file metadata fields with expressions as in original workflow.  
    - Connect Loop Over Items second output to Set File ID 1, Loop Over Items 2 second output to Set File ID 2, Loop Over Items 3 second output to Set File ID 3.

11. **Create a Compare Datasets node**  
    - Merge by fields "name" and "Path"  
    - Connect Loop Over Items, Loop Over Items 2, Loop Over Items 3 outputs accordingly.

12. **Create Microsoft OneDrive Download file node**  
    - Operation: Download  
    - File ID: `{{$json.ID}}`  
    - On Error: Continue  
    - Connect 2nd Loop Over Items1's second output.

13. **Create Extract From File node**  
    - Operation: PDF text extraction  
    - On Error: Continue  
    - Connect Download file output.

14. **Create LangChain Agent node (Overview)**  
    - Text: Document text from Extract PDF Text node (`{{$json.text}}`)  
    - System Message: As per original, instructing structured, impartial markdown overview with Executive Summary, Key Findings, and Scope.  
    - Prompt Type: Define  
    - Connect Extract PDF Text output.

15. **Create LangChain Structured Output Parser node (Structured Output Parser1)**  
    - Input Schema: JSON schema for summary, key_findings, scope fields (manual schema as per workflow)  
    - AutoFix enabled  
    - Connect Overview output.

16. **Create LangChain Chain LLM node (Overview LLM Chain)**  
    - Text: Reformats Overview output into strict JSON, filling missing fields with empty strings/arrays  
    - System Message: Professional Data Validation and Reformatting instructions  
    - Enable Output Parser  
    - Connect Structured Output Parser1 output.

17. **Create LangChain Agent node (Document Information)**  
    - Text: Extracted document text  
    - System Message: Extract relevant date, location, and personal/professional info with strict role adherence  
    - Enable Output Parser  
    - Connect Extract PDF Text output.

18. **Create LangChain Structured Output Parser node (Structured Output Parser)**  
    - Input Schema: Nested JSON schema for relevant_date, location (city, state_country), personal_professional_info (full_name, title_role, organization)  
    - AutoFix enabled  
    - Connect Document Information output.

19. **Create LangChain Chain LLM node (Document LLM Chain)**  
    - Text: Reformats Document Information output strictly to JSON with empty defaults  
    - System Message: Professional Data Validation instructions  
    - Enable Output Parser  
    - Connect Structured Output Parser output.

20. **Create Merge node (Merge1)**  
    - Connect Overview LLM Chain and Document LLM Chain outputs.

21. **Create Aggregate node**  
    - Aggregate all incoming items into one output  
    - Connect Merge1 output.

22. **Create Data Table Insert row node**  
    - Map columns: Date, Path, Scope, Summary, Location, File_Name, Key_Findings using expressions from merged AI outputs and file metadata.  
    - Connect Aggregate output.

23. **Finalize connections:**  
    - Insert row connects back to 2nd Loop Over Items1 to continue processing files.

24. **Add Sticky Notes** (optional but recommended)  
    - Add notes describing setup, folder structure depth, data table columns, and AI review instructions as per original workflow.

25. **Configure Credentials:**  
    - Microsoft OneDrive OAuth2 credentials for all OneDrive nodes  
    - OpenAI or Mistral Cloud credentials for AI nodes  
    - Ensure API keys and tokens are valid and authorized.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Folder search depth limited to 8 levels; extend workflow with additional nodes for deeper trees | Sticky Note near folder retrieval nodes                                                                     |
| Unique main folder name recommended to avoid duplicate search results                            | Sticky Note near Search for Main Folder node                                                                |
| Data table columns: Summary, Key_Findings, Scope, Date, Location, File_Name, and Path            | Sticky Note near Insert row node; customize as needed                                                      |
| AI agent outputs require strict formatting adherence; modify LLM chains and parsers if fields change | Sticky Note near AI nodes                                                                                   |
| Future extensions suggested: export results to Google Sheets or Excel, automated email summaries | Sticky Notes near end of workflow                                                                            |
| Non-PDF document handling suggested as an extension by replicating similar download and extraction steps | Sticky Note near early file filtering nodes                                                                 |
| For detailed AI prompt structures, refer to LangChain and Mistral AI documentation              | Official docs: https://docs.langchain.com/, https://mistral.ai/docs                                        |

---

_Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._