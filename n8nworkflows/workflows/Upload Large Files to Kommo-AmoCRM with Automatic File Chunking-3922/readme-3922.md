Upload Large Files to Kommo/AmoCRM with Automatic File Chunking

https://n8nworkflows.xyz/workflows/upload-large-files-to-kommo-amocrm-with-automatic-file-chunking-3922


# Upload Large Files to Kommo/AmoCRM with Automatic File Chunking

### 1. Workflow Overview

This workflow automates the process of uploading large files to Kommo or AmoCRM by splitting files into manageable chunks and uploading them sequentially. It is designed for cases where file size exceeds typical upload limits and supports sending files in chat messages, transaction/contact fields, or notes within the CRM.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Validation:** Receives inputs including file URL, name, and base64 content, and verifies file availability.
- **1.2 File Preparation:** Downloads the file, converts it, and calculates its size.
- **1.3 Session Creation and File Chunking:** Creates a session for upload, checks file size against limits, and splits the file into chunks if needed.
- **1.4 Chunk Upload Loop:** Iterates over file chunks, converts each chunk to a file format, and uploads them sequentially to the CRM.
- **1.5 Error Handling:** Stops workflow with error messages when file is missing or disk space is insufficient.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
This block triggers the workflow, extracts necessary input parameters, and checks if the required file information is present.

**Nodes Involved:**  
- `input` (Execute Workflow Trigger)  
- `hasFile` (If)  
- `No file Error` (Stop and Error)  
- `Convert to File` (Convert to File)

**Node Details:**

- **input**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point to the workflow, triggered externally with parameters.  
  - Configuration: No parameters; expects external input with required fields (drive_url, file_name, file_base64).  
  - Connections: Outputs to `hasFile`.  
  - Edge cases: Missing input parameters will cause downstream errors.

- **hasFile**  
  - Type: If  
  - Role: Checks if the file input exists and is valid.  
  - Configuration: Condition evaluating presence of file data.  
  - Connections: True branch to `Convert to File`, False branch to `No file Error`.  
  - Edge cases: False branch handles missing or invalid file inputs.

- **No file Error**  
  - Type: Stop and Error  
  - Role: Stops the workflow with an error if no file is found.  
  - Configuration: Custom error message indicating missing file input.  
  - Connections: Terminal node.  
  - Edge cases: Triggered on missing file input.

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts the input data into a file format usable for size calculation and further processing.  
  - Configuration: Converts base64 or file input into file object.  
  - Connections: Outputs to `getFileSizeInBytes`.  
  - Edge cases: Conversion failures if input format is incorrect.

---

#### 1.2 File Preparation

**Overview:**  
Calculates the file size and prepares the environment for uploading by creating an upload session.

**Nodes Involved:**  
- `getFileSizeInBytes` (Code)  
- `createSession` (HTTP Request)  
- `Merge` (Merge)

**Node Details:**

- **getFileSizeInBytes**  
  - Type: Code  
  - Role: Calculates the file size in bytes from the converted file object.  
  - Configuration: Custom script to extract size from file metadata.  
  - Connections: Outputs to both `createSession` and `Merge`.  
  - Edge cases: Incorrect file structure may cause script errors.

- **createSession**  
  - Type: HTTP Request  
  - Role: Makes an API call to Kommo/AmoCRM to open an upload session.  
  - Configuration: Uses OAuth2 credentials with the CRM API endpoint for session initialization.  
  - Connections: Outputs to `Merge`.  
  - Edge cases: Auth errors, network timeouts, API limits.

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from file size calculation and session creation for downstream processing.  
  - Configuration: Merges data streams, keeping both inputs.  
  - Connections: Outputs to `isGraterThenMax`.  
  - Edge cases: Mismatched data formats causing merge failures.

---

#### 1.3 Session Creation and File Chunking

**Overview:**  
Determines if the file exceeds the maximum allowed size for upload, handles insufficient disk space errors, and splits the file into chunks when needed.

**Nodes Involved:**  
- `isGraterThenMax` (If)  
- `No free disk space` (Stop and Error)  
- `SplitFileToChunks` (Code)  
- `Convert parts to File` (Convert to File)

**Node Details:**

- **isGraterThenMax**  
  - Type: If  
  - Role: Checks if file size exceeds predefined maximum chunk size.  
  - Configuration: Condition comparing file size to max chunk size parameter.  
  - Connections: True branch to `No free disk space`, False branch to `SplitFileToChunks`.  
  - Edge cases: Misconfiguration of size threshold causing false positives/negatives.

- **No free disk space**  
  - Type: Stop and Error  
  - Role: Stops the workflow with an error indicating insufficient disk space for upload.  
  - Configuration: Custom error message.  
  - Connections: Terminal node.  
  - Edge cases: Triggered when file size is too large for processing.

- **SplitFileToChunks**  
  - Type: Code  
  - Role: Splits the file base64 string into smaller chunks for sequential uploading.  
  - Configuration: Script divides the base64 string into fixed-size parts.  
  - Connections: Outputs to `Convert parts to File`.  
  - Edge cases: Incorrect splitting logic could corrupt file parts.

- **Convert parts to File**  
  - Type: Convert to File  
  - Role: Converts each file chunk from base64 string to file object for upload.  
  - Configuration: Standard conversion node.  
  - Connections: Outputs to `Loop Over File Chunks`.  
  - Edge cases: Conversion failures with malformed chunks.

---

#### 1.4 Chunk Upload Loop

**Overview:**  
Handles uploading each chunk sequentially, looping over the file chunks and calling the upload endpoint for each part.

**Nodes Involved:**  
- `Loop Over File Chunks` (Split In Batches)  
- `loadFile` (HTTP Request)  
- `result` (Limit)  
- `getUrl` (Set)

**Node Details:**

- **Loop Over File Chunks**  
  - Type: Split In Batches  
  - Role: Iterates over each chunk in batches of one, controlling upload flow.  
  - Configuration: Batch size set to 1 to upload chunks sequentially.  
  - Connections: Main output to `result` (completion) and secondary to `getUrl` (next chunk preparation).  
  - Edge cases: Batch misconfiguration can cause upload concurrency issues.

- **loadFile**  
  - Type: HTTP Request  
  - Role: Uploads each file chunk to the Kommo/AmoCRM API.  
  - Configuration: Uses OAuth2 credentials and API endpoint for file upload chunk.  
  - Connections: Outputs back to `Loop Over File Chunks` to continue loop.  
  - Edge cases: Network errors, auth failures, API rate limits.

- **result**  
  - Type: Limit  
  - Role: Limits the total number of chunks processed or acts as a completion handler.  
  - Configuration: Default limit to control workflow flow.  
  - Connections: Terminal for loop completion.  
  - Edge cases: Premature limiting could truncate uploads.

- **getUrl**  
  - Type: Set  
  - Role: Prepares or updates URLs or parameters for the next chunk upload.  
  - Configuration: Sets variables needed for `loadFile`.  
  - Connections: Outputs to `loadFile`.  
  - Edge cases: Incorrect parameter setting may cause upload failures.

---

#### 1.5 Error Handling

**Overview:**  
Provides graceful stops with informative errors when no file is provided or when disk space is insufficient.

**Nodes Involved:**  
- `No file Error` (Stop and Error)  
- `No free disk space` (Stop and Error)

**Node Details:**

- Already detailed in previous blocks.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                               |
|-------------------------|---------------------------|----------------------------------------|-----------------------------|---------------------------|-------------------------------------------|
| input                   | Execute Workflow Trigger  | Entry point, receives inputs           |                             | hasFile                   |                                           |
| hasFile                 | If                        | Checks if file input exists             | input                       | Convert to File, No file Error |                                           |
| No file Error           | Stop and Error             | Stops if no file provided               | hasFile                     |                           |                                           |
| Convert to File         | Convert to File            | Converts input to file object            | hasFile                     | getFileSizeInBytes         |                                           |
| getFileSizeInBytes      | Code                      | Calculates file size                     | Convert to File             | createSession, Merge       |                                           |
| createSession           | HTTP Request              | Creates upload session                   | getFileSizeInBytes          | Merge                     |                                           |
| Merge                   | Merge                     | Merges file size and session data       | getFileSizeInBytes, createSession | isGraterThenMax           |                                           |
| isGraterThenMax         | If                        | Checks if file size exceeds max size    | Merge                       | No free disk space, SplitFileToChunks |                                           |
| No free disk space      | Stop and Error             | Stops if file too large                  | isGraterThenMax             |                           |                                           |
| SplitFileToChunks       | Code                      | Splits file into chunks                  | isGraterThenMax             | Convert parts to File      |                                           |
| Convert parts to File   | Convert to File            | Converts chunks to file objects          | SplitFileToChunks           | Loop Over File Chunks      |                                           |
| Loop Over File Chunks   | Split In Batches           | Loops over chunks to upload sequentially | Convert parts to File       | result, getUrl            |                                           |
| loadFile                | HTTP Request              | Uploads file chunk to CRM                | getUrl                      | Loop Over File Chunks      |                                           |
| result                  | Limit                     | Controls completion of chunk uploads    | Loop Over File Chunks       |                           |                                           |
| getUrl                  | Set                       | Sets parameters for chunk upload        | Loop Over File Chunks       | loadFile                  |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `input` node**  
   - Type: Execute Workflow Trigger  
   - Purpose: Entry point to receive parameters `drive_url`, `file_name`, and `file_base64`.  
   - No parameters needed; expect external trigger.

2. **Create `hasFile` node**  
   - Type: If  
   - Condition: Check if input contains valid file data (e.g., `file_base64` is not empty).  
   - Connect `input` → `hasFile`.  
   - True → `Convert to File`  
   - False → `No file Error`.

3. **Create `No file Error` node**  
   - Type: Stop and Error  
   - Error message: "File input is missing or invalid."  
   - Connect False branch of `hasFile` → `No file Error`.

4. **Create `Convert to File` node**  
   - Type: Convert to File  
   - Converts base64 input to file object for size calculation.  
   - Connect True branch of `hasFile` → `Convert to File`.

5. **Create `getFileSizeInBytes` node**  
   - Type: Code  
   - Script to calculate file size from the file object.  
   - Connect `Convert to File` → `getFileSizeInBytes`.

6. **Create `createSession` node**  
   - Type: HTTP Request  
   - Configure with OAuth2 credentials for Kommo/AmoCRM.  
   - Set API endpoint to open upload session.  
   - Connect `getFileSizeInBytes` → `createSession`.

7. **Create `Merge` node**  
   - Type: Merge  
   - Mode: Merge by Index or similar to combine outputs from `getFileSizeInBytes` and `createSession`.  
   - Connect `getFileSizeInBytes` and `createSession` → `Merge`.

8. **Create `isGraterThenMax` node**  
   - Type: If  
   - Condition: Check if file size > max chunk size (define max chunk size as a parameter).  
   - Connect `Merge` → `isGraterThenMax`.  
   - True → `No free disk space`  
   - False → `SplitFileToChunks`.

9. **Create `No free disk space` node**  
   - Type: Stop and Error  
   - Error message: "File size exceeds allowed maximum or insufficient disk space."  
   - Connect True branch of `isGraterThenMax` → `No free disk space`.

10. **Create `SplitFileToChunks` node**  
    - Type: Code  
    - Script: Split base64 file string into fixed-size chunks.  
    - Connect False branch of `isGraterThenMax` → `SplitFileToChunks`.

11. **Create `Convert parts to File` node**  
    - Type: Convert to File  
    - Converts each chunk base64 string to file object for upload.  
    - Connect `SplitFileToChunks` → `Convert parts to File`.

12. **Create `Loop Over File Chunks` node**  
    - Type: Split In Batches  
    - Batch Size: 1 (upload chunks sequentially).  
    - Connect `Convert parts to File` → `Loop Over File Chunks`.

13. **Create `getUrl` node**  
    - Type: Set  
    - Sets parameters or URLs for uploading current chunk (e.g., file name, session info).  
    - Connect secondary output of `Loop Over File Chunks` → `getUrl`.

14. **Create `loadFile` node**  
    - Type: HTTP Request  
    - Configure with OAuth2 credentials.  
    - API endpoint: Upload chunk to Kommo/AmoCRM.  
    - Connect `getUrl` → `loadFile`.

15. **Connect `loadFile` output back to `Loop Over File Chunks`**  
    - To continue uploading next chunk until all chunks processed.

16. **Create `result` node**  
    - Type: Limit  
    - Limits or marks completion of chunk uploads.  
    - Connect main output of `Loop Over File Chunks` → `result`.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow splits large files into chunks to comply with Kommo/AmoCRM API limits and upload constraints.   | Workflow purpose.                                                                                           |
| Requires OAuth2 credentials configured for Kommo or AmoCRM API access.                                    | Credential setup step.                                                                                      |
| Input parameters required: `drive_url` (file location), `file_name` (name to save as), `file_base64` (file content). | Input requirements.                                                                                        |
| Installation steps include importing the workflow, configuring credentials, setting input parameters, and running the workflow. | Installation instructions.                                                                                  |
| Supports uploading files to chat messages, contact/transaction fields, or notes in Kommo/AmoCRM.          | Use case scenarios.                                                                                        |
| For detailed API limits and chunk size recommendations, consult Kommo/AmoCRM API documentation.           | External reference for chunk size and API usage guidelines.                                                |