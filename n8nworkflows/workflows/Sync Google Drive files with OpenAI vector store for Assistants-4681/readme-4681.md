Sync Google Drive files with OpenAI vector store for Assistants

https://n8nworkflows.xyz/workflows/sync-google-drive-files-with-openai-vector-store-for-assistants-4681


# Sync Google Drive files with OpenAI vector store for Assistants

### 1. Workflow Overview

This workflow synchronizes files stored in a specified Google Drive folder with an OpenAI-based vector store, intended for use in AI Assistants. It ensures that new or updated files on Google Drive are uploaded to the vector store, outdated files are deleted, and unchanged files are ignored. The workflow continuously compares Google Drive files with those already indexed, managing creations, updates, and deletions accordingly.

**Logical Blocks:**

- **1.1 Input Reception and Triggering:** Manual or scheduled trigger to start synchronization.
- **1.2 Fetching and Preparing Data:** Listing files from both Google Drive and the OpenAI vector store, preparing datasets for comparison.
- **1.3 Comparing Datasets:** Comparing Google Drive files with vector store files to determine actions.
- **1.4 Filtering and Looping Over Files:** Filtering out files to ignore and iterating over files batch-wise for processing.
- **1.5 File Processing & Vector Store Interaction:** Downloading, uploading, updating, or deleting files in the vector store based on comparison results.
- **1.6 Workflow Termination:** Ending loops and the overall workflow cleanly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:** Starts the sync process either manually or on a schedule.
- **Nodes Involved:** `"\"Test workflow\" clicked"`, `Schedule trigger`, `Set variables`

| Node Name                | Details                                                                                                                                                                                                                                                                                 |
|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **"Test workflow" clicked** | - Type: Manual Trigger<br>- Role: Allows manual start of the workflow.<br>- Config: Default manual trigger.<br>- Inputs: None<br>- Outputs: Triggers `Set variables`.<br>- Failures: N/A.<br>- Sub-workflow: None.                                                                        |
| **Schedule trigger**        | - Type: Schedule Trigger<br>- Role: Automatically triggers the workflow at configured intervals.<br>- Config: Uses default scheduling parameters (unspecified in JSON).<br>- Inputs: None<br>- Outputs: Triggers `Set variables`.<br>- Failures: Scheduling misconfiguration.<br>- Sub-workflow: None. |
| **Set variables**           | - Type: Set Node<br>- Role: Initializes variables and prepares data flow.<br>- Config: Empty parameters (likely sets defaults or resets context).<br>- Inputs: From triggers.<br>- Outputs: Feeds into file listing nodes.<br>- Failures: Expression errors if variables undefined.<br>- Sub-workflow: None. |

---

#### 2.2 Fetching and Preparing Data

- **Overview:** Retrieves current files from Google Drive and the vector store, splitting and setting them up for comparison.
- **Nodes Involved:** `List vector store files [OpenAI]`, `List files from folder`, `Split vector store files`, `Set vector store file`, `Set Google Drive file`

| Node Name                        | Details                                                                                                                                                                                                                                                                                                                                                                                                             |
|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **List vector store files [OpenAI]** | - Type: HTTP Request<br>- Role: Fetches all files currently stored in the OpenAI vector store.<br>- Config: Uses API endpoint for vector store file listing.<br>- Inputs: From `Set variables`.<br>- Outputs: Sends data to `Split vector store files`.<br>- Failures: HTTP errors (auth failures, rate limits, connectivity).<br>- Sub-workflow: None.                                                                 |
| **List files from folder**            | - Type: Google Drive Node<br>- Role: Lists all files in a specific Google Drive folder.<br>- Config: Uses Google Drive credentials, folder ID parameter.<br>- Inputs: From `Set variables`.<br>- Outputs: Sends data to `Set Google Drive file`.<br>- Failures: Authentication, quota exceeded, folder not found.<br>- Sub-workflow: None.                                                                                       |
| **Split vector store files**          | - Type: Split Out<br>- Role: Splits vector store files list into individual records for processing.<br>- Config: Default split.<br>- Inputs: From `List vector store files [OpenAI]`.<br>- Outputs: Sends each record to `Set vector store file`.<br>- Failures: None typical.<br>- Sub-workflow: None.                                                                                                                  |
| **Set vector store file**             | - Type: Set Node<br>- Role: Prepares individual vector store file data for comparison.<br>- Config: Maps or extracts key fields.<br>- Inputs: From `Split vector store files`.<br>- Outputs: Sent to `Compare Google Drive files with vector store files` (second input).<br>- Failures: Expression failures if fields missing.<br>- Sub-workflow: None.                                                                 |
| **Set Google Drive file**             | - Type: Set Node<br>- Role: Prepares individual Google Drive file data for comparison.<br>- Config: Maps or extracts key fields.<br>- Inputs: From `List files from folder`.<br>- Outputs: Sent to `Compare Google Drive files with vector store files` (first input).<br>- Failures: Expression failures if fields missing.<br>- Sub-workflow: None.                                                                 |

---

#### 2.3 Comparing Datasets

- **Overview:** Compares Google Drive files against vector store files to classify files into create, update, delete, or ignore categories.
- **Nodes Involved:** `Compare Google Drive files with vector store files`, `Set Google Drive file to create in vector store`, `Set Google Drive file to update in vector store`, `Set Google Drive file to delete from vector store`, `Set Google Drive file to ignore`

| Node Name                                  | Details                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|--------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Compare Google Drive files with vector store files** | - Type: Compare Datasets Node<br>- Role: Compares two datasets to identify differences and matches.<br>- Config: Uses key fields such as file ID, timestamps for comparison.<br>- Inputs: Google Drive files (1st), vector store files (2nd).<br>- Outputs: Routes outputs to four Set nodes categorizing files:<br>  - Create new vector store files<br>  - Update existing vector store files<br>  - Delete files from vector store<br>  - Ignore files<br>- Failures: Data schema mismatch, missing keys.<br>- Sub-workflow: None. |
| **Set Google Drive file to create in vector store**           | - Type: Set Node<br>- Role: Marks files identified as new to be created in vector store.<br>- Config: Sets flags or metadata for creation.<br>- Inputs: From comparison node.<br>- Outputs: Merges with other categories.<br>- Failures: Expression errors.<br>- Sub-workflow: None.                                                                                                                                                                                                                                                         |
| **Set Google Drive file to update in vector store**           | - Type: Set Node<br>- Role: Marks files identified as updated that require vector store update.<br>- Config: Sets update flags or metadata.<br>- Inputs: From comparison node.<br>- Outputs: Merges with other categories.<br>- Failures: Expression errors.<br>- Sub-workflow: None.                                                                                                                                                                                                                                                       |
| **Set Google Drive file to delete from vector store**         | - Type: Set Node<br>- Role: Marks files identified as outdated to be deleted from vector store.<br>- Config: Sets deletion flags or metadata.<br>- Inputs: From comparison node.<br>- Outputs: Merges with other categories.<br>- Failures: Expression errors.<br>- Sub-workflow: None.                                                                                                                                                                                                                                                   |
| **Set Google Drive file to ignore**                            | - Type: Set Node<br>- Role: Marks files that do not require action (unchanged).<br>- Config: Sets ignore flags or metadata.<br>- Inputs: From comparison node.<br>- Outputs: Goes to merge node to exclude from processing.<br>- Failures: Expression errors.<br>- Sub-workflow: None.                                                                                                                                                                                                                                                      |

---

#### 2.4 Filtering and Looping Over Files

- **Overview:** Merges categorized files, filters out ignored files, and iterates over the actionable files in batches for processing.
- **Nodes Involved:** `Merge Google Drive files`, `Filter out files to ignore`, `Loop over files`, `Aggregate vector store files`

| Node Name                | Details                                                                                                                                                                                                                                                                                                                                                                                                                          |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Merge Google Drive files** | - Type: Merge Node<br>- Role: Combines the separate categorized Google Drive file sets (create, update, delete, ignore) into one dataset.<br>- Config: Uses merge mode to union datasets with indexing.<br>- Inputs: From the four Set nodes for create, ignore, update, delete.<br>- Outputs: Passes merged data to filter for ignoring files.<br>- Failures: Data format conflicts.<br>- Sub-workflow: None.                                         |
| **Filter out files to ignore** | - Type: Filter Node<br>- Role: Excludes files marked as ignore from further processing.<br>- Config: Filters on ignore flag or property.<br>- Inputs: From merged dataset.<br>- Outputs: Sends actionable files to `Loop over files`.<br>- Failures: Expression errors if flags missing.<br>- Sub-workflow: None.                                                                                                                                                 |
| **Loop over files**          | - Type: Split In Batches<br>- Role: Processes files in manageable batches for efficiency.<br>- Config: Batch size unspecified (default or configured).<br>- Inputs: From filter node.<br>- Outputs: Two outputs: one for aggregation after processing, another for conditional checks.<br>- Failures: Batch size misconfiguration.<br>- Sub-workflow: None.                                                                                                          |
| **Aggregate vector store files** | - Type: Aggregate Node<br>- Role: Aggregates processed results for finalization.<br>- Config: Aggregate mode (probably collect all results).<br>- Inputs: From loop.<br>- Outputs: Connects to end workflow.<br>- Failures: Data loss if aggregation fails.<br>- Sub-workflow: None.                                                                                                                                      |

---

#### 2.5 File Processing & Vector Store Interaction

- **Overview:** Handles downloading files from Google Drive, uploading to vector store, managing updates and deletions in the store, and controlling flow based on file status.
- **Nodes Involved:** `Does file need to be created in vector store?`, `Download new or updated file`, `Upload new or updated file`, `Add uploaded file to vector store [OpenAI]`, `Search for existing file in vector store [OpenAI]`, `Does file need to be updated or deleted from vector store?`, `Delete outdated file or previous version`, `Has file been created or updated?`, `End loop`

| Node Name                                | Details                                                                                                                                                                                                                                                                                                                                                                                                                           |
|------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Does file need to be created in vector store?** | - Type: If Node<br>- Role: Determines if current file requires creation in vector store.<br>- Config: Condition checks create flag.<br>- Inputs: From loop node.<br>- Outputs: True → Download file; False → Search existing vector store file.<br>- Failures: Logic errors if flags misassigned.<br>- Sub-workflow: None.                                                                                                               |
| **Download new or updated file**            | - Type: Google Drive Node (Download File)<br>- Role: Downloads file content from Google Drive for processing.<br>- Config: Uses file ID from input.<br>- Inputs: From create or update checks.<br>- Outputs: Sends data to upload node.<br>- Failures: Download errors (file missing, permission issues).<br>- Sub-workflow: None.                                                                                                 |
| **Upload new or updated file**               | - Type: OpenAI Node (Langchain OpenAI)<br>- Role: Uploads the downloaded file content to the OpenAI vector store.<br>- Config: Uses Langchain OpenAI node with configured credentials.<br>- Inputs: From download node.<br>- Outputs: Sends response to add file node.<br>- Failures: API errors, timeout, auth errors.<br>- Sub-workflow: None.                                                                                   |
| **Add uploaded file to vector store [OpenAI]** | - Type: HTTP Request<br>- Role: Adds the uploaded file metadata and vectors to OpenAI vector store.<br>- Config: HTTP API call to vector store add endpoint.<br>- Inputs: From upload node.<br>- Outputs: Routes to switch checking update status.<br>- Failures: HTTP errors.<br>- Sub-workflow: None.                                                                                                                        |
| **Search for existing file in vector store [OpenAI]** | - Type: HTTP Request<br>- Role: Searches vector store for existing file matching Google Drive file.<br>- Config: Uses HTTP API with search parameters.<br>- Inputs: From if node (creation false branch).<br>- Outputs: Sends results to switch node for update/delete decision.<br>- Failures: HTTP errors.<br>- Sub-workflow: None.                                                                                               |
| **Does file need to be updated or deleted from vector store?** | - Type: Switch Node<br>- Role: Routes files needing update or deletion.<br>- Config: Switch conditions on update/delete flags.<br>- Inputs: From search node.<br>- Outputs: Update → Download file; Delete → Delete HTTP request.<br>- Failures: Logic errors.<br>- Sub-workflow: None.                                                                                                                                        |
| **Delete outdated file or previous version** | - Type: HTTP Request<br>- Role: Executes deletion of outdated files from vector store.<br>- Config: HTTP DELETE or equivalent.<br>- Inputs: From switch node.<br>- Outputs: Ends loop.<br>- Failures: HTTP errors, permission denied.<br>- Sub-workflow: None.                                                                                                                                                               |
| **Has file been created or updated?**         | - Type: Switch Node<br>- Role: Checks if file upload or update succeeded.<br>- Config: Switch on upload success or update flags.<br>- Inputs: From add uploaded file node.<br>- Outputs: True → End loop; False → Delete outdated file.<br>- Failures: Logic errors.<br>- Sub-workflow: None.                                                                                                                                |
| **End loop**                                  | - Type: NoOp Node<br>- Role: Marks the end of batch processing loop.<br>- Config: None.<br>- Inputs: From success or delete nodes.<br>- Outputs: Loops back to `Loop over files` or ends.<br>- Failures: None.<br>- Sub-workflow: None.                                                                                                                                                                                        |

---

#### 2.6 Workflow Termination

- **Overview:** Finalizes the workflow after all batches and file processing steps complete.
- **Nodes Involved:** `End workflow`

| Node Name         | Details                                                                                                                                                                      |
|-------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **End workflow**      | - Type: NoOp Node<br>- Role: Marks overall workflow completion.<br>- Config: None.<br>- Inputs: From aggregate node.<br>- Outputs: None.<br>- Failures: None.<br>- Sub-workflow: None. |

---

### 3. Summary Table

| Node Name                              | Node Type                | Functional Role                              | Input Node(s)                                       | Output Node(s)                                      | Sticky Note                                                                                       |
|--------------------------------------|--------------------------|----------------------------------------------|----------------------------------------------------|----------------------------------------------------|-------------------------------------------------------------------------------------------------|
| "Test workflow" clicked               | Manual Trigger           | Manual start trigger                         | None                                               | Set variables                                      |                                                                                                 |
| Schedule trigger                     | Schedule Trigger          | Scheduled start trigger                      | None                                               | Set variables                                      |                                                                                                 |
| Set variables                       | Set Node                 | Initializes variables                        | "Test workflow" clicked, Schedule trigger          | List vector store files [OpenAI], List files from folder |                                                                                                 |
| List vector store files [OpenAI]    | HTTP Request             | Lists existing vector store files           | Set variables                                      | Split vector store files                            |                                                                                                 |
| List files from folder              | Google Drive             | Lists files in Google Drive folder           | Set variables                                      | Set Google Drive file                              |                                                                                                 |
| Split vector store files            | Split Out                | Splits vector store files into individual   | List vector store files [OpenAI]                   | Set vector store file                              |                                                                                                 |
| Set vector store file               | Set Node                 | Prepares vector store file data              | Split vector store files                            | Compare Google Drive files with vector store files |                                                                                                 |
| Set Google Drive file               | Set Node                 | Prepares Google Drive file data              | List files from folder                             | Compare Google Drive files with vector store files |                                                                                                 |
| Compare Google Drive files with vector store files | Compare Datasets         | Compares file lists to categorize changes   | Set Google Drive file, Set vector store file       | Set Google Drive file to create, ignore, update, delete |                                                                                                 |
| Set Google Drive file to create in vector store | Set Node                 | Marks files to create in vector store        | Compare Google Drive files with vector store files | Merge Google Drive files                            |                                                                                                 |
| Set Google Drive file to ignore     | Set Node                 | Marks files to ignore                         | Compare Google Drive files with vector store files | Merge Google Drive files                            |                                                                                                 |
| Set Google Drive file to update in vector store | Set Node                 | Marks files to update in vector store        | Compare Google Drive files with vector store files | Merge Google Drive files                            |                                                                                                 |
| Set Google Drive file to delete from vector store | Set Node                 | Marks files to delete from vector store      | Compare Google Drive files with vector store files | Merge Google Drive files                            |                                                                                                 |
| Merge Google Drive files            | Merge Node               | Merges categorized file lists                | Set Google Drive file to create, ignore, update, delete | Filter out files to ignore                          |                                                                                                 |
| Filter out files to ignore          | Filter Node              | Filters out ignored files                     | Merge Google Drive files                           | Loop over files                                    |                                                                                                 |
| Loop over files                    | Split In Batches          | Processes files in batches                    | Filter out files to ignore                         | Aggregate vector store files (1), Does file need to be created in vector store? (2) |                                                                                                 |
| Aggregate vector store files       | Aggregate Node            | Aggregates processed file data                | Loop over files (1)                                | End workflow                                      |                                                                                                 |
| Does file need to be created in vector store? | If Node                  | Checks if file requires creation             | Loop over files (2)                               | Download new or updated file (True), Search for existing file in vector store [OpenAI] (False) |                                                                                                 |
| Download new or updated file       | Google Drive              | Downloads file content                         | Does file need to be created in vector store? (True), Does file need to be updated or deleted from vector store? (Update output) | Upload new or updated file                         |                                                                                                 |
| Upload new or updated file         | Langchain OpenAI          | Uploads file content to OpenAI vector store  | Download new or updated file                       | Add uploaded file to vector store [OpenAI]        |                                                                                                 |
| Add uploaded file to vector store [OpenAI] | HTTP Request             | Adds file vectors to OpenAI vector store      | Upload new or updated file                         | Has file been created or updated?                  |                                                                                                 |
| Search for existing file in vector store [OpenAI] | HTTP Request             | Searches for matching files in vector store  | Does file need to be created in vector store? (False) | Does file need to be updated or deleted from vector store? |                                                                                                 |
| Does file need to be updated or deleted from vector store? | Switch Node              | Routes files for update or deletion           | Search for existing file in vector store [OpenAI] | Download new or updated file (Update), Delete outdated file or previous version (Delete) |                                                                                                 |
| Delete outdated file or previous version | HTTP Request             | Deletes files from vector store                | Does file need to be updated or deleted from vector store? (Delete) | End loop                                          |                                                                                                 |
| Has file been created or updated?  | Switch Node              | Checks if upload/update succeeded             | Add uploaded file to vector store [OpenAI]         | End loop (True), Delete outdated file (False)     |                                                                                                 |
| End loop                          | NoOp Node                | Marks end of loop iteration                     | Has file been created or updated?, Delete outdated file or previous version | Loop over files                                   |                                                                                                 |
| End workflow                     | NoOp Node                | Marks end of entire workflow                    | Aggregate vector store files                        | None                                               |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `"Test workflow" clicked`.
   - Add a **Schedule Trigger** node named `Schedule trigger` with desired schedule settings (e.g., every hour or daily).

2. **Add Set Variables node:**
   - Add a **Set** node named `Set variables` with no parameters configured initially.
   - Connect outputs of both trigger nodes to this node.

3. **List Existing Vector Store Files:**
   - Add an **HTTP Request** node named `List vector store files [OpenAI]`.
   - Configure it to call the OpenAI vector store API endpoint that lists stored files.
   - Use appropriate authentication (e.g., API key).
   - Connect output of `Set variables` to this node.

4. **List Google Drive Files:**
   - Add a **Google Drive** node named `List files from folder`.
   - Configure it to list files from the target folder by folder ID.
   - Use the Google Drive OAuth2 credentials.
   - Connect output of `Set variables` to this node as well.

5. **Prepare Vector Store Files for Comparison:**
   - Add a **Split Out** node named `Split vector store files` connected to `List vector store files [OpenAI]`.
   - Add a **Set** node named `Set vector store file` connected to the split node.
   - Map required fields such as file ID, name, modification date.

6. **Prepare Google Drive Files for Comparison:**
   - Add a **Set** node named `Set Google Drive file` connected to `List files from folder`.
   - Map required fields such as file ID, name, modification date.

7. **Compare Datasets:**
   - Add a **Compare Datasets** node named `Compare Google Drive files with vector store files`.
   - Connect `Set Google Drive file` as the first input and `Set vector store file` as the second input.
   - Configure key matching fields (e.g., file ID) to identify create, update, delete, or ignore files.

8. **Categorize Files:**
   - Add four **Set** nodes:
     - `Set Google Drive file to create in vector store`
     - `Set Google Drive file to ignore`
     - `Set Google Drive file to update in vector store`
     - `Set Google Drive file to delete from vector store`
   - Connect outputs of the compare node to each set node accordingly.
   - Set flags or metadata to mark each file’s category.

9. **Merge Categorized Files:**
   - Add a **Merge** node named `Merge Google Drive files`.
   - Connect all four categorized Set nodes to this merge node.

10. **Filter Out Ignored Files:**
    - Add a **Filter** node named `Filter out files to ignore`.
    - Configure it to exclude files marked with the ignore flag.
    - Connect output of `Merge Google Drive files` to this filter.

11. **Batch Process Files:**
    - Add a **Split In Batches** node named `Loop over files`.
    - Configure batch size as needed (e.g., 5 or 10).
    - Connect output of filter to this node.

12. **Aggregate Results:**
    - Add an **Aggregate** node named `Aggregate vector store files`.
    - Connect the first output of `Loop over files` to it.

13. **Decide File Creation:**
    - Add an **If** node named `Does file need to be created in vector store?`.
    - Configure it to check the creation flag on the current file.
    - Connect second output of `Loop over files` to this node.

14. **Download File:**
    - Add a **Google Drive** node named `Download new or updated file`.
    - Configure it to download file content using the file ID.
    - Connect true output of If node to this node.

15. **Upload File to OpenAI Vector Store:**
    - Add a **Langchain OpenAI** node named `Upload new or updated file`.
    - Configure it to upload the file content to OpenAI vector store using appropriate credentials.
    - Connect output of download node to this node.

16. **Add Uploaded File Metadata:**
    - Add an **HTTP Request** node named `Add uploaded file to vector store [OpenAI]`.
    - Configure it to POST file metadata and vectors to vector store API.
    - Connect output of upload node to this node.

17. **Search Vector Store for Existing File:**
    - Add an **HTTP Request** node named `Search for existing file in vector store [OpenAI]`.
    - Configure it to search vector store by file ID or name.
    - Connect false output of `Does file need to be created in vector store?` If node to this node.

18. **Decide Update or Delete:**
    - Add a **Switch** node named `Does file need to be updated or deleted from vector store?`.
    - Configure cases for update and delete flags.
    - Connect output of search node to this switch node.

19. **Handle Update:**
    - Connect update case output of switch node back to `Download new or updated file` node for processing.

20. **Handle Delete:**
    - Add an **HTTP Request** node named `Delete outdated file or previous version`.
    - Configure to delete the file from vector store using DELETE method.
    - Connect delete case output of switch node to this node.

21. **Check Upload/Update Success:**
    - Add a **Switch** node named `Has file been created or updated?`.
    - Configure to check the success of upload or update operations.
    - Connect output of `Add uploaded file to vector store [OpenAI]` to this node.

22. **End Loop or Delete Outdated:**
    - Connect true output of this switch node to a **NoOp** node named `End loop`.
    - Connect false output to the `Delete outdated file or previous version` node.

23. **Loop Back:**
    - Connect `End loop` node output back to `Loop over files` to process next batch.

24. **Complete Workflow:**
    - Connect output of `Aggregate vector store files` node to a **NoOp** node named `End workflow`.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow uses the [OpenAI vector store](https://platform.openai.com/docs/guides/embeddings) API to manage indexed files. | Reference for OpenAI embeddings and vector store API usage.                                                             |
| Google Drive OAuth2 credentials must be configured properly with access to the target folder.                            | Google Cloud Console - Create OAuth2 credentials for Drive API.                                                         |
| Batch processing reduces API load and helps avoid rate limits.                                                          | Recommended batch size configurable in `Loop over files` node.                                                          |
| The workflow gracefully handles create, update, delete, and ignore actions to keep the vector store in sync.             |                                                                                                                        |
| For error handling, consider adding error workflow or retries on HTTP Request nodes to handle API failures or timeouts. | n8n docs on error workflows: https://docs.n8n.io/integrations/builtin/core-nodes/error-trigger/                          |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.