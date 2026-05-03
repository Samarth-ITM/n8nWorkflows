Merge Multiple Excel Files into a Single Multi-Sheet File with Summary Page

https://n8nworkflows.xyz/workflows/merge-multiple-excel-files-into-a-single-multi-sheet-file-with-summary-page-8373


# Merge Multiple Excel Files into a Single Multi-Sheet File with Summary Page

### 1. Workflow Overview

This workflow automates the merging of multiple Excel (.xlsx) files from a specified disk folder into a single multi-sheet Excel workbook. Each original file is converted into a separate worksheet, and an additional summary worksheet provides metadata and record counts for all merged files. The key use case is to consolidate disparate Excel files into one comprehensive file for easier analysis and reporting.  

The workflow is logically divided into three main blocks:  

- **1.1 Workflow Initiation and File Reading**: Manual trigger followed by reading all Excel files from disk and splitting them for individual processing.  
- **1.2 Data Extraction and Processing**: Extraction of raw Excel data into JSON, cleaning and aggregation into unified data structures.  
- **1.3 Generate and Save Multi-Worksheet Excel File**: Creation of the merged multi-sheet Excel file including a summary sheet, and saving it back to disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Initiation and File Reading

**Overview:**  
This block starts the workflow manually and reads all `.xlsx` files from a designated folder on disk, then splits the files for sequential processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Read XLXS Files from Disk  
- Read each XLXS  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual workflow execution  
  - Configuration: Default, no parameters  
  - Input: None  
  - Output: Triggers downstream nodes  
  - Edge cases: None typical; user must manually trigger  
  - Sub-workflow: None  

- **Read XLXS Files from Disk**  
  - Type: Read/Write File  
  - Role: Reads all `.xlsx` files from `n8n_files/` directory  
  - Configuration: File selector set to `n8n_files/*.xlsx` (reads all Excel files)  
  - Input: Trigger from manual trigger  
  - Output: Binary data of each Excel file  
  - Edge cases: Missing files, file permission errors, unreadable files  
  - Version: v1 node, standard file read operation  

- **Read each XLXS**  
  - Type: Split In Batches  
  - Role: Processes each file individually, splitting the batch of files into single items  
  - Configuration: Default options, processes one file at a time  
  - Input: Output from file reader node (list of files)  
  - Output: Each file forwarded individually for extraction  
  - Edge cases: Large number of files may cause performance bottlenecks  

---

#### 2.2 Data Extraction and Processing

**Overview:**  
For each individual Excel file, extract the data into JSON format, clean empty rows, and aggregate all files’ data for further processing.

**Nodes Involved:**  
- XLSX to Json List  
- Mulipte Json to Single Json  
- Collect and Process Data  

**Node Details:**  

- **XLSX to Json List**  
  - Type: Extract From File  
  - Role: Converts Excel file contents into JSON data  
  - Configuration:  
    - Operation: `xlsx`  
    - Options: rawData=true, headerRow=true, includeEmptyCells=false  
  - Input: Single Excel file binary data from batch splitter  
  - Output: JSON array representing rows of the sheet  
  - Edge cases: Corrupt Excel file, empty sheets, mixed data types  
  - Version: v1 node  

- **Mulipte Json to Single Json**  
  - Type: Aggregate  
  - Role: Aggregates multiple JSON data items into a single combined JSON object  
  - Configuration: Aggregate all item data into one  
  - Input: JSON arrays from previous extraction  
  - Output: Single combined JSON array  
  - Edge cases: Empty input arrays, inconsistent schemas  

- **Collect and Process Data**  
  - Type: Code (JavaScript)  
  - Role:  
    - Iterates over all files’ JSON data  
    - Extracts file names and generates sheet names (sanitizing file names)  
    - Cleans out empty rows in data  
    - Creates an array of objects containing: sheetName, data, originalFileName, and recordCount  
  - Key expressions:  
    - Sheet name derived by stripping path and extension from file name  
    - Filtering rows with no meaningful data  
  - Input: Aggregated JSON data from previous node  
  - Output: Single JSON object with `allFiles` array describing each file's data and metadata  
  - Edge cases: Files with no data, improperly formatted rows  

---

#### 2.3 Generate and Save Multi-Worksheet Excel File

**Overview:**  
Creates a new Excel workbook with one worksheet per original file and an additional summary worksheet listing file metadata. Saves the resulting file to disk.

**Nodes Involved:**  
- Create Multi-Sheet Excel  
- Save XLXS to Disk  

**Node Details:**  

- **Create Multi-Sheet Excel**  
  - Type: Code (JavaScript)  
  - Role:  
    - Uses external `xlsx` library to create an Excel workbook  
    - For each file in `allFiles`, creates a worksheet from JSON data or an empty sheet if no data  
    - Sanitizes sheet names for Excel compatibility (max 31 characters, no special chars)  
    - Adds a summary sheet with generation time, total files, and file metadata  
    - Converts workbook to a base64-encoded buffer for n8n binary output  
    - Generates timestamped file name  
  - Key expressions:  
    - Sheet name sanitization regex `[\\[\\]\\*\\/\\\\\\?\\:]` replaced with underscores  
    - Summary sheet data structured as 2D array  
  - Input: JSON object with `allFiles` array containing data and metadata  
  - Output: Binary Excel file with metadata in JSON and binary fields  
  - Version: Requires external `xlsx` npm module enabled in n8n environment  
  - Edge cases: No files to process (throws error), large files may cause memory issues  
  - Additional notes:  
    - Requires modification to docker-compose or Dockerfile to allow `xlsx` external module  
    - See sticky note for setup instructions  

- **Save XLXS to Disk**  
  - Type: Read/Write File  
  - Role: Writes the generated Excel file to disk under `n8n_files/output/` with dynamic filename  
  - Configuration:  
    - Operation: write  
    - File name expression: `n8n_files/output/{{$json.fileName}}` (uses generated fileName)  
  - Input: Binary Excel data from previous node  
  - Output: Confirmation of file write operation  
  - Edge cases: File permission errors, disk space issues  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                              |
|-------------------------|---------------------|-----------------------------------------------|-----------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Workflow entry point                           | None                        | Read XLXS Files from Disk  | ## 🚀 1. Workflow Initiation and File Reading: Manual trigger to start workflow                         |
| Read XLXS Files from Disk          | Read/Write File     | Reads all Excel files from disk folder        | When clicking ‘Execute workflow’ | Read each XLXS            | ## 🚀 1. Workflow Initiation and File Reading: Reads `.xlsx` files from `n8n_files/`                      |
| Read each XLXS                    | Split In Batches    | Splits files into individual batch items      | Read XLXS Files from Disk    | Collect and Process Data; XLSX to Json List | ## 🚀 1. Workflow Initiation and File Reading: Processes files one by one                              |
| XLSX to Json List                | Extract From File   | Extracts JSON data from single Excel file     | Read each XLXS              | Mulipte Json to Single Json | ## 📊 2. Data Extraction and Processing: Extract data from single XLSX file                             |
| Mulipte Json to Single Json      | Aggregate           | Aggregates JSON data from multiple files      | XLSX to Json List           | Read each XLXS             | ## 📊 2. Data Extraction and Processing: Summarizes JSON data into one                                 |
| Collect and Process Data          | Code                | Cleans and prepares all files' data/metadata  | Read each XLXS              | Create Multi-Sheet Excel   | ## 📝 3. Generate and Save Multi-Worksheet Excel File: Collects and cleans all file data               |
| Create Multi-Sheet Excel          | Code                | Creates multi-sheet Excel file with summary   | Collect and Process Data    | Save XLXS to Disk          | ## 📝 3. Generate and Save Multi-Worksheet Excel File: Merges data, creates summary, requires `xlsx` module enabled |
| Save XLXS to Disk                | Read/Write File     | Saves the final Excel file to output folder   | Create Multi-Sheet Excel    | None                      | ## 📝 3. Generate and Save Multi-Worksheet Excel File: Writes output file to `n8n_files/output/`          |
| 工作流启动与文件读取 (Sticky Note)  | Sticky Note         | Describes block 1: Workflow Initiation & Reading | N/A                        | N/A                       | ## 🚀 1. Workflow Initiation and File Reading: Manual trigger, read files, batch split                  |
| 数据提取与处理 (Sticky Note)       | Sticky Note         | Describes block 2: Data Extraction & Processing | N/A                        | N/A                       | ## 📊 2. Data Extraction and Processing: Extract and aggregate JSON data                               |
| 生成与保存Excel文件 (Sticky Note)   | Sticky Note         | Describes block 3: Generate and save Excel file | N/A                        | N/A                       | ## 📝 3. Generate and Save Multi-Worksheet Excel File: Data collection, merging, summary, saving; requires Dockerfile update |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manual start of workflow  
   - No special configuration needed  

2. **Create Read/Write File node**  
   - Name: `Read XLXS Files from Disk`  
   - Operation: Read  
   - File selector: `n8n_files/*.xlsx`  
   - Reads all Excel files from the designated directory  
   - Connect output of Manual Trigger to this node  

3. **Create Split In Batches node**  
   - Name: `Read each XLXS`  
   - Default batch size (1) to process files one at a time  
   - Connect from `Read XLXS Files from Disk` node output  

4. **Create Extract From File node**  
   - Name: `XLSX to Json List`  
   - Operation: `xlsx`  
   - Options: Enable `rawData`, `headerRow`, disable `includeEmptyCells`  
   - Connect output of `Read each XLXS` to this node  

5. **Create Aggregate node**  
   - Name: `Mulipte Json to Single Json`  
   - Operation: Aggregate all item data (select appropriate aggregation to combine JSON arrays)  
   - Connect output of `XLSX to Json List` node to this node  

6. **Connect output of Aggregate node back to the input of Split In Batches node (`Read each XLXS`)**  
   - This loop ensures all files are processed and aggregated correctly  

7. **Create Code node**  
   - Name: `Collect and Process Data`  
   - JavaScript code:  
     - Iterate over all inputs  
     - Extract file names, sanitize for sheet names  
     - Clean empty rows  
     - Create array `allFiles` with objects: `sheetName`, `data`, `originalFileName`, `recordCount`  
   - Connect output of `Read each XLXS` node to this node  

8. **Create Code node**  
   - Name: `Create Multi-Sheet Excel`  
   - JavaScript code:  
     - Import `xlsx` module  
     - Create new workbook  
     - For each file in `allFiles`, create worksheet with sanitized sheet name  
     - If no data, create placeholder worksheet with info  
     - Append summary worksheet with metadata (file names, record counts, timestamp)  
     - Write workbook to buffer, convert to base64  
     - Generate timestamped file name  
     - Return JSON and binary data for the Excel file  
   - Connect output of `Collect and Process Data` node to this node  
   - **Important:** Enable external `xlsx` module in n8n environment:  
     - Add `NODE_FUNCTION_ALLOW_EXTERNAL=xlsx` to `.env` or docker-compose.yml  
     - Adjust Dockerfile to install `xlsx` npm package and set environment variables (see sticky note 3)  

9. **Create Read/Write File node**  
   - Name: `Save XLXS to Disk`  
   - Operation: Write  
   - File name: `n8n_files/output/{{$json.fileName}}` (dynamic filename from previous node)  
   - Connect output of `Create Multi-Sheet Excel` to this node  

10. **Verify all connections:**  
    - Manual Trigger → Read XLXS Files from Disk → Read each XLXS →  
      (parallel to) Collect and Process Data → Create Multi-Sheet Excel → Save XLXS to Disk  
    - Read each XLXS → XLSX to Json List → Mulipte Json to Single Json → Read each XLXS (loop back)  

11. **Test the workflow:**  
    - Place `.xlsx` files in `n8n_files/` folder  
    - Execute workflow manually  
    - Check output file in `n8n_files/output/` folder with proper multi-sheet Excel file and summary sheet  

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| To run `Create Multi-Sheet Excel` node, enable external module `xlsx` in n8n environment:                               | See sticky note 3 for detailed Dockerfile and environment variable modifications                |
| Modify `docker-compose.yml` or `.env` file to include: `NODE_FUNCTION_ALLOW_EXTERNAL=xlsx`                              | Required for external module usage in Function/Code nodes                                        |
| Replace `image: n8nio/n8n:latest` with `build: .` and create Dockerfile with npm install of `xlsx` module              | Dockerfile example provided in sticky note 3                                                    |
| Sheet names sanitized to max 31 characters and disallow special characters as per Excel rules                          | Important for Excel compatibility                                                                |
| Input Excel files must be `.xlsx` format and placed in `n8n_files/` folder                                              | Workflow depends on these files being present and accessible                                     |
| Output merged Excel saved to `n8n_files/output/` folder                                                                 | Ensure write permissions exist for this folder                                                  |
| Summary worksheet includes file count, generation timestamp, sheet names, original file names, and record counts       | Useful for auditing and verification                                                            |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.