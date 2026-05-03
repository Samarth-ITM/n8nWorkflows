Monitor S3 Bucket Changes with Automated Integrity Auditing using Mistral AI

https://n8nworkflows.xyz/workflows/monitor-s3-bucket-changes-with-automated-integrity-auditing-using-mistral-ai-7547


# Monitor S3 Bucket Changes with Automated Integrity Auditing using Mistral AI

### 1. Workflow Overview

This workflow, titled **"Monitor S3 Bucket Changes with Automated Integrity Auditing using Mistral AI"**, automates the monitoring and auditing of objects in an AWS S3 bucket to ensure data integrity. It is designed for use cases where continuous verification of file integrity and detection of unauthorized changes or corruptions in S3 storage are required. The workflow leverages cryptographic hashing, AI-based content analysis (Mistral AI), and multiple integrations including AWS S3, SSH for host filesystem operations, MongoDB for data logging, and email for reporting.

The workflow is logically divided into the following functional blocks:

- **1.1 Trigger and Initialization**  
  Initiates the workflow manually or on a schedule and prepares the environment.

- **1.2 S3 Bucket Snapshot and Integrity Hashing**  
  Lists objects in the target S3 bucket, generates cryptographic hashes (MD5/MD6) for integrity snapshots, and compares with previous snapshots.

- **1.3 Audit Snapshot Management**  
  Downloads, converts, and saves audit snapshots, manages snapshot files locally and on host storage via SSH.

- **1.4 Detection and Handling of Suspect Files**  
  Identifies files with potential integrity issues, downloads suspect files, and processes them for further analysis.

- **1.5 AI-Powered Content Analysis**  
  Uses Mistral AI and LangChain to analyze file content, including OCR for PDFs and text extraction for logs and text files, generating audit reports.

- **1.6 Reporting and Notification**  
  Creates audit reports, uploads these to storage, and sends email summaries.

- **1.7 Cleanup and Finalization**  
  Deletes temporary files and folders to maintain a clean host environment.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

- **Overview:**  
  This block starts the workflow either manually or on a scheduled basis and prepares initial data structures.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation for testing or on-demand audits  
    - Configuration: Default, no parameters  
    - Inputs: None  
    - Outputs: Starts "Objects Listing"

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow at configured intervals (cron or fixed rate)  
    - Configuration: Default scheduling parameters (not specified)  
    - Inputs: None  
    - Outputs: Starts "Get previous Audit Snapshot" and "List S3 Objects"

- **Potential Failures:** Scheduling misconfiguration, manual trigger misuse

---

#### 1.2 S3 Bucket Snapshot and Integrity Hashing

- **Overview:**  
  Lists all objects in the AWS S3 bucket, generates cryptographic hashes (MD5/MD6) for current snapshot, and prepares JSON data for comparison.

- **Nodes Involved:**  
  - Objects Listing (AWS S3)  
  - Generate MD5 (Crypto)  
  - Create JSON from AWS S3 API Call (Set)  
  - Generate MD6 (Crypto)  
  - Create JSON from results (Set)  
  - Compare Datasets (Compare)

- **Node Details:**  

  - **Objects Listing**  
    - Type: AWS S3 Node (v2)  
    - Role: Lists current S3 objects for integrity check  
    - Configuration: Connected to AWS credentials, bucket specified, retries enabled with wait time 5s  
    - Inputs: Trigger nodes  
    - Outputs: Feeds into "Generate MD5"

  - **Generate MD5**  
    - Type: Crypto  
    - Role: Generates MD5 hash for each S3 object metadata to create snapshot signature  
    - Configuration: Hash function set to MD5  
    - Inputs: From "Objects Listing"  
    - Outputs: "Create JSON from AWS S3 API Call"

  - **Create JSON from AWS S3 API Call**  
    - Type: Set  
    - Role: Structures the hashed data into JSON format for snapshot  
    - Configuration: Uses expressions to format data  
    - Inputs: From "Generate MD5"  
    - Outputs: "Convert to File"

  - **Generate MD6**  
    - Type: Crypto  
    - Role: Generates MD6 hashes (likely for previous snapshots or suspect files)  
    - Configuration: Hash function MD6 (or similar, per naming)  
    - Inputs: From "List S3 Objects" (previous snapshot)  
    - Outputs: "Create JSON from results"

  - **Create JSON from results**  
    - Type: Set  
    - Role: Structures the MD6 hash data into JSON format  
    - Inputs: From "Generate MD6"  
    - Outputs: "Convert to File1" and "Compare Datasets"

  - **Compare Datasets**  
    - Type: Compare Datasets  
    - Role: Compares current snapshot JSON and previous snapshot JSON to detect changes  
    - Inputs: JSON files from current and previous snapshots  
    - Outputs:  
      - Index 0: Path Extraction (for suspect files)  
      - Index 1: Convert to File2 (stores suspect objects list)  
      - Other outputs: No operations or error handling

- **Potential Failures:** AWS API rate limits, hash generation errors, JSON formatting issues

---

#### 1.3 Audit Snapshot Management

- **Overview:**  
  Downloads previous audit snapshots, converts them to JSON, saves current snapshots, and manages snapshot files on the host machine via SSH.

- **Nodes Involved:**  
  - Get previous Audit Snapshot (SSH)  
  - Audit file to JSON (Extract from File)  
  - Convert to File (Convert to File)  
  - Save Audit Snapshot (SSH)  
  - Replace previous snapshot with current one (S3)  
  - Download current Integrity Snapshot (S3)  
  - Upload Snapshot on Host FS (SSH)

- **Node Details:**  

  - **Get previous Audit Snapshot**  
    - Type: SSH  
    - Role: Downloads the last audit snapshot file from host for comparison  
    - Configuration: SSH credentials and remote path settings  
    - Inputs: From "Schedule Trigger"  
    - Outputs: "Audit file to JSON"

  - **Audit file to JSON**  
    - Type: Extract from File  
    - Role: Parses previous audit snapshot file into JSON data  
    - Configuration: Extraction rules for JSON format  
    - Inputs: From "Get previous Audit Snapshot"  
    - Outputs: "Split Objects"

  - **Convert to File**  
    - Type: Convert to File  
    - Role: Converts current snapshot data to a file format suitable for storage  
    - Inputs: From "Create JSON from AWS S3 API Call"  
    - Outputs: "Save Audit Snapshot"

  - **Save Audit Snapshot**  
    - Type: SSH  
    - Role: Uploads the current snapshot file to host machine for archival  
    - Inputs: From "Convert to File"  
    - Outputs: None

  - **Replace previous snapshot with current one**  
    - Type: S3  
    - Role: Updates the snapshot file in S3 with the latest snapshot  
    - Inputs: From "Convert to File1"  
    - Outputs: None

  - **Download current Integrity Snapshot**  
    - Type: S3  
    - Role: Downloads the current snapshot for further processing  
    - Inputs: From "Save Suspect Objects List"  
    - Outputs: "Upload Snapshot on Host FS"

  - **Upload Snapshot on Host FS**  
    - Type: SSH  
    - Role: Uploads the current snapshot onto the host filesystem  
    - Inputs: From "Download current Integrity Snapshot"  
    - Outputs: None

- **Potential Failures:** SSH connectivity problems, file parsing errors, S3 upload/download failures

---

#### 1.4 Detection and Handling of Suspect Files

- **Overview:**  
  Extracts paths of suspect files from comparison results, downloads these files, removes duplicates, then prepares for content analysis.

- **Nodes Involved:**  
  - Path Extraction (Split Out)  
  - Objects Download (AWS S3)  
  - Remove Duplicates (Remove Duplicates)  
  - Download suspect files (S3)  
  - Boucle (Split In Batches)  
  - Upload objects to MinIO (S3)  
  - Remove them from Host FS (SSH)  
  - List downloaded objects (S3)  
  - Extraction des paths (Split Out)  
  - Only keep PDF, TXT and Logs (Filter)

- **Node Details:**  

  - **Path Extraction**  
    - Type: Split Out  
    - Role: Extracts file paths identified as suspect from comparison results  
    - Inputs: From "Compare Datasets"  
    - Outputs: "Objects Download"

  - **Objects Download**  
    - Type: AWS S3 (v2)  
    - Role: Downloads suspect files from S3 for analysis  
    - Configuration: Uses AWS credentials, retry enabled  
    - Inputs: From "Path Extraction"  
    - Outputs: "Upload objects to MinIO"

  - **Remove Duplicates**  
    - Type: Remove Duplicates  
    - Role: Ensures unique suspect files by removing duplicates  
    - Inputs: From "Only keep PDF, TXT and Logs"  
    - Outputs: "Download suspect files"

  - **Download suspect files**  
    - Type: AWS S3 (legacy)  
    - Role: Downloads filtered suspect files for batch processing  
    - Inputs: From "Remove Duplicates"  
    - Outputs: "Boucle"

  - **Boucle**  
    - Type: Split In Batches  
    - Role: Processes suspect files in manageable batches  
    - Inputs: From "Download suspect files"  
    - Outputs: "Report Creation" and "Switch"

  - **Upload objects to MinIO**  
    - Type: S3  
    - Role: Uploads downloaded suspect files to MinIO storage for further use or backup  
    - Inputs: From "Objects Download"  
    - Outputs: "Remove them from Host FS"

  - **Remove them from Host FS**  
    - Type: SSH  
    - Role: Deletes suspect files from host filesystem to clean up local storage  
    - Inputs: From "Upload objects to MinIO"  
    - Outputs: "List downloaded objects"

  - **List downloaded objects**  
    - Type: S3  
    - Role: Lists files recently downloaded for further processing  
    - Inputs: From "Remove them from Host FS"  
    - Outputs: "Extraction des paths"

  - **Extraction des paths**  
    - Type: Split Out  
    - Role: Extracts file paths again for filtering  
    - Inputs: From "List downloaded objects"  
    - Outputs: "Only keep PDF, TXT and Logs"

  - **Only keep PDF, TXT and Logs**  
    - Type: Filter  
    - Role: Filters files to only keep certain file types relevant for AI analysis  
    - Configuration: Filters by file extension (.pdf, .txt, .log)  
    - Inputs: From "Extraction des paths"  
    - Outputs: "Remove Duplicates"

- **Potential Failures:** S3 download errors, duplicate detection issues, file type mismatch, batch processing limits

---

#### 1.5 AI-Powered Content Analysis

- **Overview:**  
  Analyzes suspect file content using AI, including OCR for PDFs and text extraction for logs/txt files, generating prompts and processing responses for integrity auditing.

- **Nodes Involved:**  
  - Switch  
  - Extract text with OCR (Mistral AI)  
  - Extract .txt / .log (Extract from File)  
  - PDF Prompt Creation (Set)  
  - TXT/LOG Prompt Creation (Set)  
  - AnalyseIA (LangChain LLM Chain)  
  - Mistral Cloud Chat Model1 (Mistral AI Cloud)  
  - Result Message Creation (Set)  
  - Error Message #1 (Set)  
  - Error Message #2 (Set)  
  - Result Saved (SSH)  
  - Wait  
  - Report Creation (SSH)  
  - Getting Report (SSH)  
  - Upload Report to MinIO (S3)  
  - Envoi de mail récapitulatif (Email Send)  

- **Node Details:**  

  - **Switch**  
    - Type: Switch  
    - Role: Routes files based on type for appropriate processing (PDF via OCR, TXT/LOG via text extraction, else error)  
    - Inputs: From "Boucle"  
    - Outputs:  
      - PDF files -> "Extract text with OCR"  
      - TXT/LOG files -> "Extract .txt / .log"  
      - Others -> "Error Message #1"

  - **Extract text with OCR**  
    - Type: Mistral AI (OCR)  
    - Role: Extracts text from PDF files using AI OCR capabilities  
    - Config: Retries enabled with wait, error output continues  
    - Inputs: From "Switch"  
    - Outputs: "PDF Prompt Creation"

  - **Extract .txt / .log**  
    - Type: Extract from File  
    - Role: Extracts raw text from .txt or .log files  
    - Inputs: From "Switch"  
    - Outputs: "TXT/LOG Prompt Creation"

  - **PDF Prompt Creation**  
    - Type: Set  
    - Role: Creates AI prompt specific to PDF content analysis  
    - Inputs: From "Extract text with OCR"  
    - Outputs: "AnalyseIA"

  - **TXT/LOG Prompt Creation**  
    - Type: Set  
    - Role: Creates AI prompt specific to text/log content analysis  
    - Inputs: From "Extract .txt / .log"  
    - Outputs: "AnalyseIA"

  - **AnalyseIA**  
    - Type: LangChain LLM Chain  
    - Role: Performs AI content analysis using Mistral Cloud Chat Model  
    - Inputs: From prompt creation nodes  
    - Outputs: "Result Message Creation"  
    - Config: Retries with backoff on fail

  - **Mistral Cloud Chat Model1**  
    - Type: Mistral AI Cloud language model node  
    - Role: Provides the NLP model for LangChain chain  
    - Outputs: Connected internally to "AnalyseIA"

  - **Result Message Creation**  
    - Type: Set  
    - Role: Formats AI results into messages for storage or reporting  
    - Inputs: From "AnalyseIA"  
    - Outputs: "Result Saved"

  - **Error Message #1 / #2**  
    - Type: Set  
    - Role: Handles errors by creating error messages  
    - Inputs: From "Switch" or "Extract .txt / .log" and "Extract text with OCR" failure outputs  
    - Outputs: "Result Saved"

  - **Result Saved**  
    - Type: SSH  
    - Role: Saves AI analysis results to host filesystem  
    - Inputs: From "Result Message Creation" or error messages  
    - Outputs: "Wait"

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay before next batch processing to manage rate limits or system load  
    - Inputs: From "Result Saved"  
    - Outputs: "Boucle" (batch loop)

  - **Report Creation**  
    - Type: SSH  
    - Role: Generates audit report files from processed analysis data  
    - Inputs: From batch processing  
    - Outputs: "Getting Report"

  - **Getting Report**  
    - Type: SSH  
    - Role: Retrieves the generated report file for upload and emailing  
    - Inputs: From "Report Creation"  
    - Outputs: "Envoi de mail récapitulatif", "Extract from File", "Upload Report to MinIO"

  - **Upload Report to MinIO**  
    - Type: S3  
    - Role: Uploads the audit report to MinIO storage  
    - Inputs: From "Getting Report"  
    - Outputs: "Delete local files"

  - **Envoi de mail récapitulatif**  
    - Type: Email Send  
    - Role: Sends summary email of audit results to configured recipients  
    - Inputs: From "Getting Report"  
    - Outputs: None

- **Potential Failures:** AI model API limits, OCR inaccuracies, file type misclassification, email delivery issues

---

#### 1.6 Reporting and Notification

- **Overview:**  
  Generates, retrieves, uploads audit reports, and sends email notifications with the audit summary.

- **Nodes Involved:**  
  - Report Creation (SSH)  
  - Getting Report (SSH)  
  - Upload Report to MinIO (S3)  
  - Envoi de mail récapitulatif (Email Send)

- **Details:** (See above in AI block; this is the final stage of reporting)

- **Potential Failures:** SSH issues, file upload failure, email sending errors

---

#### 1.7 Cleanup and Finalization

- **Overview:**  
  Removes temporary files and folders created during the workflow to maintain cleanliness on the host.

- **Nodes Involved:**  
  - Delete local files (SSH)  
  - Delete a folder1 (S3)  
  - Create a folder1 (S3)

- **Node Details:**  

  - **Delete local files**  
    - Type: SSH  
    - Role: Deletes suspect or temporary files on host filesystem after upload  
    - Inputs: From "Upload Report to MinIO"  
    - Outputs: "Delete a folder1"

  - **Delete a folder1**  
    - Type: S3  
    - Role: Deletes temporary folders in S3 bucket to clean environment  
    - Inputs: From "Delete local files"  
    - Outputs: "Create a folder1"

  - **Create a folder1**  
    - Type: S3  
    - Role: Recreates necessary folder structure in S3 after cleanup  
    - Inputs: From "Delete a folder1"  
    - Outputs: None

- **Potential Failures:** SSH command errors, S3 permission issues, folder recreation failures

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                               | Input Node(s)                    | Output Node(s)                         | Sticky Note                     |
|--------------------------------|----------------------------------|-----------------------------------------------|---------------------------------|--------------------------------------|--------------------------------|
| When clicking ‘Execute workflow’| Manual Trigger                   | Manual workflow start                         | None                            | Objects Listing                      |                                |
| Schedule Trigger               | Schedule Trigger                 | Timed workflow start                          | None                            | Get previous Audit Snapshot, List S3 Objects |                                |
| Objects Listing               | AWS S3 (v2)                     | Lists current S3 objects                       | When clicking ‘Execute workflow’| Generate MD5                        |                                |
| Generate MD5                  | Crypto                          | Generates MD5 hashes for current snapshot     | Objects Listing                 | Create JSON from AWS S3 API Call    |                                |
| Create JSON from AWS S3 API Call| Set                            | Formats snapshot data as JSON                   | Generate MD5                   | Convert to File                    |                                |
| Convert to File               | Convert to File                 | Converts snapshot JSON to file                  | Create JSON from AWS S3 API Call| Save Audit Snapshot               |                                |
| Save Audit Snapshot           | SSH                            | Saves snapshot file to host via SSH             | Convert to File                | None                             |                                |
| Get previous Audit Snapshot   | SSH                            | Downloads previous snapshot from host           | Schedule Trigger               | Audit file to JSON                |                                |
| Audit file to JSON            | Extract from File               | Parses previous snapshot into JSON              | Get previous Audit Snapshot     | Split Objects                    |                                |
| Split Objects                | Split Out                      | Splits JSON objects for processing              | Audit file to JSON             | Compare Datasets                |                                |
| List S3 Objects              | AWS S3 (v2)                    | Lists previous snapshot objects                  | Schedule Trigger               | Generate MD6                    |                                |
| Generate MD6                 | Crypto                        | Generates MD6 hashes for previous snapshot       | List S3 Objects               | Create JSON from results        |                                |
| Create JSON from results     | Set                           | Formats previous snapshot data as JSON            | Generate MD6                  | Convert to File1, Compare Datasets |                                |
| Convert to File1             | Convert to File               | Converts previous snapshot JSON to file           | Create JSON from results      | Replace previous snapshot with current one |                                |
| Replace previous snapshot with current one | S3                 | Uploads current snapshot to S3 bucket             | Convert to File1              | None                             |                                |
| Compare Datasets             | Compare Datasets              | Compares current and previous snapshots           | Split Objects, Create JSON from results | Path Extraction, Convert to File2 |                                |
| Path Extraction             | Split Out                     | Extracts paths of suspect files                    | Compare Datasets              | Objects Download               |                                |
| Objects Download            | AWS S3 (v2)                   | Downloads suspect files from S3                    | Path Extraction              | Upload objects to MinIO         |                                |
| Upload objects to MinIO     | S3                           | Uploads suspect files to MinIO                      | Objects Download             | Remove them from Host FS        |                                |
| Remove them from Host FS    | SSH                          | Deletes suspect files from host filesystem           | Upload objects to MinIO      | List downloaded objects         |                                |
| List downloaded objects     | S3                           | Lists downloaded suspect files                       | Remove them from Host FS     | Extraction des paths           |                                |
| Extraction des paths        | Split Out                    | Extracts file paths for filtering                     | List downloaded objects      | Only keep PDF, TXT and Logs    |                                |
| Only keep PDF, TXT and Logs| Filter                       | Filters files by extension (.pdf, .txt, .log)          | Extraction des paths         | Remove Duplicates              |                                |
| Remove Duplicates           | Remove Duplicates            | Removes duplicate suspect files                       | Only keep PDF, TXT and Logs  | Download suspect files         |                                |
| Download suspect files      | AWS S3 (legacy)              | Downloads suspect files for batch processing          | Remove Duplicates            | Boucle                       |                                |
| Boucle                     | Split In Batches             | Processes suspect files in batches                     | Download suspect files       | Report Creation, Switch        |                                |
| Switch                     | Switch                      | Routes files based on type for analysis                | Boucle                      | Extract text with OCR, Extract .txt / .log, Error Message #1 |                                |
| Extract text with OCR       | Mistral AI                  | Performs OCR text extraction on PDFs                   | Switch                      | PDF Prompt Creation, Error Message #2 |                                |
| Extract .txt / .log        | Extract from File           | Extracts text from .txt and .log files                  | Switch                      | TXT/LOG Prompt Creation, Error Message #2 |                                |
| PDF Prompt Creation        | Set                        | Creates AI prompt for PDF content analysis               | Extract text with OCR        | AnalyseIA                   |                                |
| TXT/LOG Prompt Creation    | Set                        | Creates AI prompt for text/log content analysis          | Extract .txt / .log          | AnalyseIA                   |                                |
| AnalyseIA                  | LangChain LLM Chain        | Performs AI analysis via Mistral Cloud Chat Model        | PDF Prompt Creation, TXT/LOG Prompt Creation | Result Message Creation    |                                |
| Mistral Cloud Chat Model1  | Mistral AI                 | AI language model node used by AnalyseIA                 | None                        | AnalyseIA                   |                                |
| Result Message Creation    | Set                        | Formats AI results into messages                          | AnalyseIA                   | Result Saved               |                                |
| Error Message #1           | Set                        | Creates error message for unsupported file types          | Switch                      | Result Saved               |                                |
| Error Message #2           | Set                        | Creates error message for extraction/OCR failures         | Extract .txt / .log, Extract text with OCR | Result Saved               |                                |
| Result Saved               | SSH                        | Saves AI analysis results to host filesystem               | Result Message Creation, Error Message #1, Error Message #2 | Wait                        |                                |
| Wait                      | Wait                       | Delays batch processing to manage system load              | Result Saved                | Boucle                     |                                |
| Report Creation           | SSH                        | Generates audit reports from analysis data                   | Boucle                      | Getting Report             |                                |
| Getting Report            | SSH                        | Retrieves report file for upload and emailing                 | Report Creation             | Envoi de mail récapitulatif, Extract from File, Upload Report to MinIO |                                |
| Extract from File         | Extract from File          | Extracts data from report files                               | Getting Report              | Split Out                  |                                |
| Split Out                 | Split Out                 | Splits report data for MongoDB insertion                       | Extract from File           | Ajout à MongoDB            |                                |
| Ajout à MongoDB           | MongoDB                   | Inserts processed data into MongoDB database                    | Split Out                   | None                      |                                |
| Upload Report to MinIO    | S3                        | Uploads audit report to MinIO storage                            | Getting Report              | Delete local files         |                                |
| Delete local files        | SSH                       | Deletes temporary files on host filesystem                       | Upload Report to MinIO      | Delete a folder1           |                                |
| Delete a folder1          | S3                        | Deletes temporary folders in S3 bucket                           | Delete local files          | Create a folder1           |                                |
| Create a folder1          | S3                        | Recreates necessary folder structure in S3 after cleanup        | Delete a folder1            | None                      |                                |
| Envoi de mail récapitulatif | Email Send                | Sends summary email of audit results                            | Getting Report              | None                      |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" with default settings.  
   - Add a **Schedule Trigger** node named "Schedule Trigger" configured with your desired interval.

2. **Setup AWS S3 Object Listing:**  
   - Add an **AWS S3 (v2)** node "Objects Listing" connected to the manual trigger. Configure AWS credentials and specify the S3 bucket to audit.  
   - Add an **AWS S3 (v2)** node "List S3 Objects" connected to the schedule trigger with same bucket and credentials.

3. **Generate Hashes for Snapshots:**  
   - Add a **Crypto** node "Generate MD5" connected to "Objects Listing" to generate MD5 hashes for current objects.  
   - Add a **Set** node "Create JSON from AWS S3 API Call" to format the MD5 hashes into JSON.  
   - Add a **Convert to File** node "Convert to File" to create a file from the JSON.  
   - Add an **SSH** node "Save Audit Snapshot" to upload the snapshot file to a host machine.

4. **Handle Previous Snapshot:**  
   - Add an **SSH** node "Get previous Audit Snapshot" connected to the schedule trigger. Configure SSH credentials and path to previous snapshot.  
   - Add an **Extract from File** node "Audit file to JSON" to parse snapshot file into JSON.  
   - Add a **Split Out** node "Split Objects" to split the JSON array.

5. **Generate Hashes for Previous Snapshot:**  
   - Add an **AWS S3 (v2)** node "List S3 Objects" connected to schedule trigger to list previous snapshot objects.  
   - Add a **Crypto** node "Generate MD6" to generate hashes for previous snapshot objects.  
   - Add a **Set** node "Create JSON from results" to format the hashes.  
   - Add **Convert to File1** node to convert to file for upload.  
   - Add an **S3** node "Replace previous snapshot with current one" to upload current snapshot file.

6. **Compare Snapshots:**  
   - Add a **Compare Datasets** node to compare current and previous snapshot JSON data. Configure inputs accordingly.

7. **Extract and Download Suspect Files:**  
   - Add a **Split Out** node "Path Extraction" to extract suspect file paths.  
   - Add an **AWS S3 (v2)** node "Objects Download" to download suspect files.  
   - Add an **S3** node "Upload objects to MinIO" to upload suspect files to MinIO.  
   - Add an **SSH** node "Remove them from Host FS" to delete suspect files from the host.  
   - Add an **AWS S3 (v2)** node "List downloaded objects" to list files after deletion.  
   - Add a **Split Out** node "Extraction des paths" to extract file paths.  
   - Add a **Filter** node "Only keep PDF, TXT and Logs" to filter specific file types.  
   - Add a **Remove Duplicates** node to ensure uniqueness.  
   - Add an **AWS S3 (legacy)** node "Download suspect files" to download files for batch processing.  
   - Add a **Split In Batches** node "Boucle" to process files in batches.

8. **AI Content Analysis:**  
   - Add a **Switch** node to route files by extension: PDFs to OCR, TXT/LOG to text extraction, else error.  
   - Add a **Mistral AI** node "Extract text with OCR" for PDFs.  
   - Add an **Extract from File** node "Extract .txt / .log" for text and log files.  
   - Add two **Set** nodes "PDF Prompt Creation" and "TXT/LOG Prompt Creation" to create AI prompts.  
   - Add a **LangChain LLM Chain** node "AnalyseIA" configured with Mistral Cloud Chat Model node.  
   - Add a **Mistral Cloud Chat Model** node for AI model access.  
   - Add a **Set** node "Result Message Creation" to format AI results.  
   - Add **Set** nodes "Error Message #1" and "Error Message #2" for error handling.  
   - Add an **SSH** node "Result Saved" to save AI analysis results.  
   - Add a **Wait** node to delay processing between batches.

9. **Report Generation and Notification:**  
   - Add an **SSH** node "Report Creation" to generate audit reports.  
   - Add an **SSH** node "Getting Report" to retrieve reports.  
   - Add an **Extract from File** node to parse report data for MongoDB.  
   - Add a **Split Out** node to split report data.  
   - Add a **MongoDB** node "Ajout à MongoDB" to store audit data.  
   - Add an **S3** node "Upload Report to MinIO" to upload reports to storage.  
   - Add an **Email Send** node "Envoi de mail récapitulatif" to send audit summary emails.

10. **Cleanup:**  
    - Add an **SSH** node "Delete local files" to remove temporary files.  
    - Add an **S3** node "Delete a folder1" to delete temporary S3 folders.  
    - Add an **S3** node "Create a folder1" to recreate necessary folders.

11. **Connect Nodes According to the workflow logic** as detailed in the overview and block-by-block analysis.

12. **Configure Credentials:**  
    - AWS credentials for S3 nodes.  
    - SSH credentials for host filesystem operations.  
    - MongoDB credentials for database insertion.  
    - Mistral AI API credentials for AI nodes.  
    - Email credentials (SMTP or similar) for email sending.

13. **Set Parameters and Expressions:**  
    - Define bucket names, file paths, and filtering criteria as needed.  
    - Configure retry settings and error handling parameters.  
    - Use expressions to link data between nodes, e.g., passing file paths or AI prompts.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                              |
|--------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| The workflow extensively uses SSH nodes to interact with a host filesystem for snapshot and report management. | Requires secure SSH access and appropriate permissions.      |
| AI integration is done via Mistral Cloud Chat Model and LangChain chain node, requiring API credentials.      | LangChain docs: https://docs.langchain.com/                  |
| MongoDB is used to store audit results for long-term tracking and querying.                                  | MongoDB credential setup required.                            |
| Email node is configured to send summary reports; ensure SMTP or equivalent credentials are correctly set up.|                                                              |
| Batch processing and wait nodes are used to throttle AI calls and manage rate limits.                         | Important for stable AI API usage and avoiding timeouts.     |

---

**Disclaimer:** The provided content is derived solely from an n8n workflow designed for legal and public data processing. It adheres strictly to content policies and contains no illegal or protected elements.