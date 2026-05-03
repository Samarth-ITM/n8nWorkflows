Automated Document Sync Between SharePoint and Google Drive with Supabase

https://n8nworkflows.xyz/workflows/automated-document-sync-between-sharepoint-and-google-drive-with-supabase-10749


# Automated Document Sync Between SharePoint and Google Drive with Supabase

### 1. Workflow Overview

This workflow automates synchronization of documents between Microsoft SharePoint and Google Drive, using Supabase (Postgres) as a metadata store. It is designed to:

- Run on a schedule.
- Retrieve all folders and files recursively from a SharePoint document library.
- Filter out system and temporary files.
- Compare SharePoint file metadata with records stored in Supabase to detect new or updated files.
- For new or updated files, download the file content from SharePoint.
- Rename files according to their SharePoint titles.
- Upload files to Google Drive, ensuring a backup and sync between SharePoint and Google Drive.
- Update Supabase/Postgres metadata to track files processed and mark them as complete.

**Logical blocks:**

- **1.1 Schedule Trigger & Initial Data Retrieval**: Trigger the workflow on schedule and fetch SharePoint folder/file listings and Supabase metadata.
- **1.2 Filter and Normalize Metadata**: Flatten nested SharePoint folder structure, filter unwanted files, and normalize timestamps.
- **1.3 Compare SharePoint and Supabase Datasets**: Identify new or modified files needing processing.
- **1.4 Loop & Process New/Modified Files**: Download files from SharePoint, rename, upload to Google Drive, and update metadata in Supabase/Postgres.
- **1.5 Metadata Management**: Insert or update records in Supabase/Postgres; delete outdated metadata entries if necessary.
- **1.6 Error Handling and Retry Logic**: Continue on errors for downloads and database updates to ensure workflow robustness.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Initial Data Retrieval

**Overview:**  
This block initiates the workflow on a schedule, retrieves existing metadata from Supabase, and fetches full file/folder structure from SharePoint via API.

**Nodes involved:**  
- Schedule Trigger  
- Supabase (Get)  
- Microsoft SharePoint HTTP Request  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow repeatedly at configured intervals (default is 1-minute interval, to be customized).  
  - Configuration: Interval trigger, no special parameters.  
  - Inputs: None (start node).  
  - Outputs: Triggers Supabase and SharePoint HTTP requests.  
  - Edge cases: Misconfiguration can result in too frequent or infrequent runs.

- **Supabase (Get)**  
  - Type: Supabase node (Postgres interface)  
  - Role: Retrieves all records from the `n8n_metadata` table in Supabase to compare with SharePoint data.  
  - Configuration: Operation = getAll, Table = `n8n_metadata`, useCustomSchema = true.  
  - Inputs: Triggered by Schedule Trigger.  
  - Outputs: Passes metadata to timestamp normalization node.  
  - Edge cases: Authentication issues, network failures, or empty table scenarios.

- **Microsoft SharePoint HTTP Request**  
  - Type: HTTP Request  
  - Role: Fetches folders and files recursively from a SharePoint path using SharePoint REST API.  
  - Configuration:  
    - Method: GET  
    - URL: `https://<your-tenant>.sharepoint.com/sites/<your-site>/_api/web/GetFolderByServerRelativeUrl('/sites/<your-site>/<document-library>/<subfolders>')?$expand=Files,Folders,Folders/Files,Folders/Folders/Folders/Files,Folders/Folders/Folders/Folders/Files`  
    - Authentication: Microsoft SharePoint OAuth2  
  - Inputs: Triggered by Schedule Trigger.  
  - Outputs: Raw folder/file data to filtering code node.  
  - Edge cases: API limits, auth token expiration, URL misconfiguration.

---

#### 1.2 Filter and Normalize Metadata

**Overview:**  
Processes raw SharePoint data to flatten folder structure, filter out unwanted file types, and normalize timestamps for consistent comparison.

**Nodes involved:**  
- filter files (Code node)  
- normalize last modified date (Set node)  
- Sticky Note2 (explains Last_modified_date standardization)  

**Node Details:**

- **filter files**  
  - Type: Code (JavaScript)  
  - Role: Recursively traverses nested SharePoint folder data, extracts all files, and filters out temporary/system files by name or extension.  
  - Configuration:  
    - Excluded extensions: `.db`, `.msg`, `.xlsx`, `.xlsm`, `.pptx`  
    - Filters filenames starting with `~$` (Office temp files).  
  - Inputs: SharePoint raw data.  
  - Outputs: Clean list of valid SharePoint file objects.  
  - Edge cases: Unexpected SharePoint data structure, missing keys, large folder trees causing timeout.

- **normalize last modified date**  
  - Type: Set node  
  - Role: Adds “Z” suffix to the `Last_modified_date` field to standardize to UTC ISO string.  
  - Configuration: Assigns `Last_modified_date` = `${Last_modified_date}Z`.  
  - Inputs: Supabase metadata data.  
  - Outputs: Passes normalized metadata to dataset comparison.  
  - Edge cases: Missing or malformed timestamps.

---

#### 1.3 Compare SharePoint and Supabase Datasets

**Overview:**  
Compares SharePoint file metadata with Supabase’s stored metadata to detect new or updated documents requiring processing.

**Nodes involved:**  
- Compare Datasets  
- Sticky Note11 (explains the comparison purpose)  

**Node Details:**

- **Compare Datasets**  
  - Type: Compare Datasets node  
  - Role: Performs fuzzy comparison between SharePoint and Supabase datasets based on matching keys, identifying inserts, updates, deletions.  
  - Configuration:  
    - Merge by fields: SharePoint `sharepoint_file_id` vs Supabase `UniqueId`  
    - Also compares `Last_modified_date` and `Loading Done` flags.  
    - Fuzzy compare enabled.  
  - Inputs: Normalized Supabase metadata and filtered SharePoint file list.  
  - Outputs:  
    - New/modified files: passed to loop processing.  
    - Files to delete: passed to Supabase deletion node.  
  - Edge cases: Missing or inconsistent IDs, timestamp mismatches, large dataset performance.

---

#### 1.4 Loop & Process New/Modified Files

**Overview:**  
Iterates over files identified for upload: downloads from SharePoint, renames locally, uploads to Google Drive, then marks as completed in Postgres.

**Nodes involved:**  
- Loop Over Items2 (Split in Batches)  
- get metadata (Code node)  
- Set metadata (Set node)  
- Insert Document Metadata (Postgres upsert)  
- Microsoft SharePoint HTTP Request1 (file download)  
- rename files (Code node)  
- Upload file (Google Drive upload)  
- Postgres (update Loading Done flag)  
- Sticky Notes explaining each step (Sticky Note3, Sticky Note5, Sticky Note7, Sticky Note4, Sticky Note6)  

**Node Details:**

- **Loop Over Items2**  
  - Type: SplitInBatches  
  - Role: Processes new/modified files one by one or in batches to control load.  
  - Configuration: Default batch options (size unspecified).  
  - Inputs: Files from Compare Datasets node.  
  - Outputs: To get metadata node.  
  - Edge cases: Large batch sizes may cause timeouts.

- **get metadata**  
  - Type: Code  
  - Role: Constructs accessible file URLs from SharePoint data, using either existing `LinkingUrl` or building URL with `ServerRelativeUrl` and `UniqueId` (versioning).  
  - Inputs: Individual file metadata.  
  - Outputs: Enriched metadata with `fileUrl`.  
  - Edge cases: Missing fields, URL construction errors.

- **Set metadata**  
  - Type: Set  
  - Role: Maps and sets key metadata fields (file ID, type, title, URL, modified date, folder name) for downstream processing and database insertion.  
  - Inputs: Output of get metadata.  
  - Outputs: To Insert Document Metadata node.  
  - Edge cases: Improper JSON path expressions, missing fields.

- **Insert Document Metadata**  
  - Type: Postgres (upsert)  
  - Role: Inserts or updates file metadata in the `n8n_metadata` Postgres table, ensuring the latest info is stored.  
  - Configuration: Upsert on `id` field matching `file_id`.  
  - Inputs: Set metadata node output.  
  - Outputs: To Microsoft SharePoint HTTP Request1 for file download.  
  - Edge cases: DB connectivity issues, unique constraint failures.

- **Microsoft SharePoint HTTP Request1**  
  - Type: HTTP Request  
  - Role: Downloads the actual file content binary from SharePoint using `ServerRelativeUrl`.  
  - Configuration:  
    - Method: GET  
    - URL constructed dynamically from current file's `ServerRelativeUrl`.  
    - Response: file content in binary.  
    - OAuth2 authentication with SharePoint credentials.  
    - Retry on failure enabled.  
  - Inputs: Triggered after Insert Document Metadata.  
  - Outputs: To rename files node and back to Loop Over Items2 (error handling).  
  - Edge cases: Download failures, authentication errors, file not found.

- **rename files**  
  - Type: Code  
  - Role: Sets the binary file name to the SharePoint file title for proper naming in upload.  
  - Inputs: Binary data from SharePoint download.  
  - Outputs: To Upload file node.  
  - Edge cases: Binary data missing or corrupted.

- **Upload file**  
  - Type: Google Drive  
  - Role: Uploads the downloaded file to Google Drive under specified folder (`root` by default).  
  - Configuration:  
    - Drive: My Drive  
    - Folder: root or specified folder ID  
    - OAuth2 authentication for Google Drive.  
  - Inputs: Renamed file binary.  
  - Outputs: To Postgres update node.  
  - Edge cases: API quota limits, upload failures.

- **Postgres**  
  - Type: Postgres (update)  
  - Role: Marks the file record with `Loading Done = true` to indicate successful upload and completion.  
  - Configuration: Update operation on `n8n_metadata`, matching `id`.  
  - Inputs: Upload confirmation.  
  - Outputs: None (end of loop).  
  - Edge cases: DB update failures.

---

#### 1.5 Metadata Management (Deletion)

**Overview:**  
Deletes outdated file metadata from Supabase/Postgres when files are removed from SharePoint.

**Nodes involved:**  
- Supabase1 (Delete)  
- Sticky Note9 (explains deletion)  

**Node Details:**

- **Supabase1**  
  - Type: Supabase node (delete operation)  
  - Role: Deletes metadata records for files no longer present in SharePoint.  
  - Configuration:  
    - Table: `n8n_metadata`  
    - Filter: Delete records where `id` matches missing SharePoint files.  
  - Inputs: From Compare Datasets node deletion output.  
  - Outputs: None.  
  - Edge cases: Deletion failures, auth issues.

---

#### 1.6 Error Handling and Notes

**Overview:**  
Workflow uses `continueErrorOutput` on key nodes like SharePoint file download and Postgres update to allow workflow continuation despite individual errors.

**Sticky Notes:**  
- Provide comprehensive explanations for each major block and node.  
- Explain setup steps, filtering rationale, and synchronization logic.  
- Highlight configuration needed for SharePoint tenant URLs, OAuth credentials, and table names.

---

### 3. Summary Table

| Node Name                       | Node Type                | Functional Role                                   | Input Node(s)                    | Output Node(s)                       | Sticky Note                                                                                              |
|--------------------------------|--------------------------|-------------------------------------------------|---------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger         | Triggers workflow on schedule                    | None                            | Supabase, Microsoft SharePoint HTTP Request | ## Schedule Trigger<br>Triggers the workflow on a defined schedule                                      |
| Supabase                      | Supabase                 | Get all metadata records from Supabase           | Schedule Trigger                | normalize last modified date        | ## Supabase (Get)<br>Retrieves existing records from `n8n_metadata`                                   |
| normalize last modified date  | Set                      | Standardizes last modified date format            | Supabase                       | Compare Datasets                   | ### standardizes `Last_modified_date`                                                                  |
| Microsoft SharePoint HTTP Request | HTTP Request           | Fetch all folders and files recursively from SharePoint | Schedule Trigger                | filter files                      | ## Microsoft SharePoint HTTP Request<br>Retrieves all folders and files recursively                    |
| filter files                 | Code                     | Recursively extracts files and filters unwanted  | Microsoft SharePoint HTTP Request | Compare Datasets                   | ## Filter Files<br>Filters out system and temporary files                                              |
| Compare Datasets             | Compare Datasets          | Compares SharePoint and Supabase metadata         | normalize last modified date, filter files | Supabase1, Loop Over Items2         | ## Compare Datasets<br>Compares SharePoint and Supabase data to detect new or updated files            |
| Supabase1                    | Supabase                 | Deletes outdated metadata entries                  | Compare Datasets               | None                             | ### delete file from supabase (metadata table)                                                          |
| Loop Over Items2             | SplitInBatches           | Loops over new or modified files                   | Compare Datasets               | get metadata                      | ### loop over newly added or modified files                                                             |
| get metadata                 | Code                     | Constructs usable file URLs                         | Loop Over Items2               | Set metadata                     |                                                                                                        |
| Set metadata                 | Set                      | Maps key file metadata for DB insertion            | get metadata                  | Insert Document Metadata          | ## extract metadata                                                                                      |
| Insert Document Metadata     | Postgres (Upsert)        | Upserts metadata into Postgres/Supabase            | Set metadata                  | Microsoft SharePoint HTTP Request1 | ## Insert Document Metadata<br>Upserts processed SharePoint file data                                  |
| Microsoft SharePoint HTTP Request1 | HTTP Request         | Downloads file content from SharePoint              | Insert Document Metadata       | rename files, Loop Over Items2 (on error) | ## Microsoft SharePoint HTTP Request1 (Download)<br>Downloads each new or updated file                |
| rename files                | Code                     | Renames binary files to SharePoint file titles      | Microsoft SharePoint HTTP Request1 | Upload file                   | ## Rename Files<br>Renames each downloaded binary file                                                 |
| Upload file                 | Google Drive             | Uploads files to Google Drive                        | rename files                  | Postgres (update)                | ## Upload file (Google Drive)<br>Uploads downloaded files to Google Drive                              |
| Postgres                   | Postgres (Update)         | Marks metadata record as completed (Loading Done)  | Upload file                  | Loop Over Items2 (if more batches) | ## Postgres (Mark Complete)<br>Marks entries as processed                                             |
| Sticky Note                 | Sticky Note              | Overview, setup instructions, and explanations     | None                         | None                            | # SharePoint Sync Overview<br>Runs on a schedule, fetches SharePoint files, filters, syncs metadata... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval according to desired sync frequency (e.g., every hour).  
   - No credentials needed.

2. **Create a Supabase node** to get all records  
   - Type: Supabase  
   - Operation: getAll  
   - Table: `n8n_metadata`  
   - Use custom schema: true  
   - Connect Schedule Trigger output to this node.  
   - Configure Supabase credentials.

3. **Create a Set node** to normalize `Last_modified_date`  
   - Type: Set  
   - Add an assignment: `Last_modified_date` = `{{$json["Last_modified_date"] + "Z"}}`  
   - Include other fields unchanged.  
   - Connect Supabase node output here.

4. **Create a Microsoft SharePoint HTTP Request node** to fetch folders and files  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://<your-tenant>.sharepoint.com/sites/<your-site>/_api/web/GetFolderByServerRelativeUrl('/sites/<your-site>/<document-library>/<subfolders>')?$expand=Files,Folders,Folders/Files,Folders/Folders/Folders/Files,Folders/Folders/Folders/Folders/Files`  
   - Authentication: Microsoft SharePoint OAuth2  
   - Connect Schedule Trigger output here in parallel with Supabase node.

5. **Create a Code node to recursively find and filter files**  
   - Type: Code (JavaScript)  
   - Paste filtering code that recursively extracts files and filters out `.db`, `.msg`, `.xlsx`, `.xlsm`, `.pptx` and files starting with `~$`.  
   - Connect SharePoint HTTP Request output here.

6. **Create a Compare Datasets node**  
   - Type: Compare Datasets  
   - Input 1: Normalize last modified date node (Supabase data)  
   - Input 2: Filter files code node (SharePoint data)  
   - Set merge by fields mapping: SharePoint `sharepoint_file_id` <-> Supabase `UniqueId`, `Last_modified_date` <-> `TimeLastModified`, `Loading Done` <-> `Exists`  
   - Enable fuzzy compare.

7. **Create a Supabase node for deletion**  
   - Type: Supabase  
   - Operation: Delete  
   - Table: `n8n_metadata`  
   - Filter: `id` equals the IDs of files missing from SharePoint (from Compare Datasets' deletion output).  
   - Connect Compare Datasets deletion output here.

8. **Create a SplitInBatches node** (Loop Over Items2)  
   - Type: SplitInBatches  
   - Connect Compare Datasets new/updated files output here.

9. **Create a Code node (get metadata)**  
   - Type: Code  
   - Build file URLs combining `ServerRelativeUrl` and versioning with `UniqueId`.  
   - Connect SplitInBatches output here.

10. **Create a Set node (Set metadata)**  
    - Type: Set  
    - Assign fields: `file_id`, `file_type`, `file_title`, `file_url`, `last_modified_date`, `sharepoint_file_id`, `foldername`.  
    - Use expressions based on previous code node output.  
    - Connect get metadata output here.

11. **Create a Postgres node (Insert Document Metadata)**  
    - Type: Postgres  
    - Operation: Upsert  
    - Table: `n8n_metadata`  
    - Map fields from Set metadata node.  
    - Connect Set metadata output here.  
    - Configure Postgres credentials.

12. **Create a Microsoft SharePoint HTTP Request node (Download)**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: Constructed dynamically using current item's `ServerRelativeUrl`  
    - Response: File (binary)  
    - Authentication: SharePoint OAuth2  
    - Connect Insert Document Metadata output here.  
    - Enable retry on failure.

13. **Create a Code node (rename files)**  
    - Type: Code  
    - Rename binary file to SharePoint file title (`$input.item.binary.data.fileName = $input.first().json.title`).  
    - Connect SharePoint file download output here.

14. **Create a Google Drive node (Upload file)**  
    - Type: Google Drive  
    - Operation: Upload file  
    - Drive: My Drive  
    - Folder: root (or specified folder)  
    - Connect rename files output here.  
    - Configure Google Drive OAuth2 credentials.

15. **Create a Postgres node (Mark complete)**  
    - Type: Postgres  
    - Operation: Update  
    - Table: `n8n_metadata`  
    - Set `Loading Done = true` for current `id`.  
    - Connect Upload file output here.

16. **Connect Postgres update output back to SplitInBatches node** to continue batches.

17. **Add Sticky Notes** for documentation and maintenance ease.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Runs on a schedule, fetches SharePoint folders and files, filters unwanted formats, and syncs metadata.   | Overview sticky note at workflow start.          |
| Only new or modified files are processed; files uploaded to Google Drive for backup and sync.              | Overview sticky note.                             |
| Setup requires connecting SharePoint, Supabase/Postgres, and Google Drive credentials.                     | Overview sticky note.                             |
| Adjust excluded extensions in the filtering code node as needed.                                          | Filter files node note.                           |
| Postgres `n8n_metadata` table tracks files and upload status (`Loading Done` flag).                        | Insert Document Metadata and Postgres nodes notes.|
| OAuth2 credentials needed for SharePoint and Google Drive nodes.                                          | Credential section in nodes.                      |
| Retry enabled on file download to handle transient network errors.                                        | SharePoint HTTP Request1 node.                    |
| The workflow supports recursive folder structures.                                                       | Microsoft SharePoint HTTP Request node and filter files code. |
| Use the workflow to maintain consistent backups of SharePoint files in Google Drive for redundancy.       | Workflow purpose.                                 |

---

This document provides a complete, detailed, and structured reference to understand, reproduce, or maintain the "Automated Document Sync Between SharePoint and Google Drive with Supabase" workflow in n8n.