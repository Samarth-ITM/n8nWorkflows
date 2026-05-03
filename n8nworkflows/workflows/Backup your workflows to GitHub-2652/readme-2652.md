Backup your workflows to GitHub

https://n8nworkflows.xyz/workflows/backup-your-workflows-to-github-2652


# Backup your workflows to GitHub

### 1. Workflow Overview

This workflow automates the backup of all n8n instance workflows to a GitHub repository. It is designed for users who want to secure their workflows externally or migrate them between servers. The workflow exports each n8n workflow via the n8n API, then checks if a corresponding JSON file exists in GitHub based on workflow ID. Depending on the comparison result, it updates, creates, or ignores the file in the repository.

The workflow is divided into the following logical blocks:

- **1.1 Triggering and Initialization:** Manual or scheduled start of the workflow, retrieving all workflows from n8n.
- **1.2 Iteration and GitHub File Retrieval:** Looping over each workflow, fetching the corresponding file from GitHub.
- **1.3 File Size Check and Content Retrieval:** Handling large files and downloading content if available.
- **1.4 Diff Check and Status Determination:** Comparing current workflow JSON with GitHub version to classify as same, different, or new.
- **1.5 GitHub File Management:** Creating new files or editing existing ones based on status.
- **1.6 Workflow Control and Completion:** Managing loop continuation and signaling completion.

A subworkflow is used to execute the main workflow logic in batches to optimize memory usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering and Initialization

- **Overview:**  
  This block initiates the workflow via manual execution or a scheduled trigger, then fetches all workflows from the n8n instance through the API node.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger  
  - n8n (n8n API Node)  

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual triggering of the workflow in the n8n editor.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None  
    - Outputs: Connected to the n8n API node.  
    - Edge cases: Manual trigger depends on user action; no failures expected.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every 2 hours.  
    - Configuration: Interval set to 2 hours.  
    - Inputs: None  
    - Outputs: Connected to the n8n API node.  
    - Edge cases: Scheduler downtime or misconfiguration may skip triggers.

  - **n8n**  
    - Type: n8n API node  
    - Role: Retrieves all workflows from the n8n instance (no filters applied).  
    - Configuration: Default filters (empty), uses stored n8n API credentials.  
    - Inputs: From manual or schedule trigger.  
    - Outputs: Passes workflows data to the next node.  
    - Credentials: Uses `n8nApi` credential (n8n account).  
    - Edge cases: API authentication failure, network issues, or empty workflow list.

---

#### 2.2 Iteration and GitHub File Retrieval

- **Overview:**  
  This block loops over each workflow item, sets global GitHub repo parameters, and attempts to fetch the corresponding file from GitHub using the workflow ID as filename.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Globals (Set node)  
  - Merge Items  
  - Loop Over Items (SplitInBatches)  
  - Get file data (GitHub node)  

- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Triggers a subworkflow with the current item passed through.  
    - Configuration: Input source set to passthrough (passes data as is).  
    - Inputs: Receives workflows list from n8n API node via Merge Items.  
    - Outputs: Sends each workflow item to Globals and Merge Items nodes.  
    - Edge cases: Subworkflow must exist and be accessible; failure if subworkflow missing.

  - **Globals**  
    - Type: Set Node  
    - Role: Defines GitHub repository details (owner, repo name, path).  
    - Configuration: Hardcoded values for `repo.owner` (e.g., `john-doe`), `repo.name` (e.g., `n8n-backup`), and `repo.path` (e.g., `workflows/`).  
    - Inputs: Receives data from Execute Workflow Trigger.  
    - Outputs: Feeds into Get file data node.  
    - Edge cases: Incorrect repository details cause GitHub API failures.

  - **Merge Items**  
    - Type: Merge Node  
    - Role: Combines data from Globals and Execute Workflow Trigger to provide complete context for GitHub file retrieval.  
    - Inputs: One input from Globals, one from Execute Workflow Trigger.  
    - Outputs: Sends merged data to Loop Over Items.  
    - Edge cases: Mismatched or empty data could cause missing fields.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes workflows in batches to reduce memory load.  
    - Configuration: Default batch size (not explicitly set, defaults implied).  
    - Inputs: Receives workflows from n8n API node or Execute Workflow node.  
    - Outputs: Loops back to Execute Workflow node or ends batch.  
    - Edge cases: Very large number of workflows may still cause memory issues.

  - **Get file data**  
    - Type: GitHub Node (file get operation)  
    - Role: Attempts to retrieve the workflow JSON file from GitHub by path and filename (`repo.path` + workflow ID + `.json`).  
    - Configuration:  
      - Owner, repo, path are set dynamically from merged data.  
      - Operation: Get file.  
      - Continue On Fail: Enabled (to carry on even if file does not exist).  
      - Always Output Data: Enabled (to handle missing files gracefully).  
    - Inputs: From Globals node.  
    - Outputs: Passes to "If file too large" node.  
    - Credentials: Uses GitHub API credentials.  
    - Edge cases: Missing file causes 404; other API errors possible (rate limit, auth).

---

#### 2.3 File Size Check and Content Retrieval

- **Overview:**  
  Checks if the retrieved file is too large or empty and, if needed, fetches the file content via HTTP. Ensures the content is loaded correctly before diffing.

- **Nodes Involved:**  
  - If file too large (IF node)  
  - Get File (HTTP Request)  
  - Merge Items  

- **Node Details:**

  - **If file too large**  
    - Type: IF Node  
    - Role: Checks if the GitHub file content is empty or missing (`content` is empty string or does not exist, and no error present).  
    - Configuration: Conditions check if `$json.content` is empty and `$json.error` does not exist.  
    - Inputs: From Get file data node.  
    - Outputs:  
      - True branch: Calls Get File HTTP Request node to fetch content.  
      - False branch: Proceeds directly to Merge Items.  
    - Edge cases: Unexpected response formats could cause logic failure.

  - **Get File**  
    - Type: HTTP Request Node  
    - Role: Downloads the raw file content from the `download_url` provided by GitHub API.  
    - Configuration: URL is dynamically taken from `$json.download_url`.  
    - Inputs: From True branch of If file too large node.  
    - Outputs: Connects to Merge Items.  
    - Edge cases: Network errors, invalid URLs, or access restrictions.

  - **Merge Items**  
    - Type: Merge Node  
    - Role: Combines the content from the HTTP Request and the original file metadata for further processing.  
    - Inputs: From If file too large (False branch) and Get File node.  
    - Outputs: Passes merged data to the diff check code node.  
    - Edge cases: Mismatched data or missing fields.

---

#### 2.4 Diff Check and Status Determination

- **Overview:**  
  Compares the content of the workflow file on GitHub and the current n8n workflow JSON to determine if the file is the same, different, or new.

- **Nodes Involved:**  
  - isDiffOrNew (Code Node)  
  - Check Status (Switch Node)  

- **Node Details:**

  - **isDiffOrNew**  
    - Type: Code Node (JavaScript)  
    - Role:  
      - Orders JSON keys alphabetically for normalized comparison.  
      - Decodes base64 content if present, parses JSON.  
      - Compares GitHub file content and current workflow JSON.  
      - Assigns status: `"same"`, `"different"`, or `"new"`.  
      - Adds formatted stringified JSON for changed or new workflows.  
    - Configuration: Custom JavaScript code provided inline.  
    - Inputs: Receives two inputs combined from Merge Items: file content and current workflow JSON.  
    - Outputs: Annotates data with `github_status` and passes to Check Status.  
    - Edge cases: Parsing errors if JSON malformed, base64 decoding errors.

  - **Check Status**  
    - Type: Switch Node  
    - Role: Routes flow based on `github_status` field (`same`, `different`, `new`).  
    - Configuration: Three rules matching string values of `github_status`.  
    - Inputs: From isDiffOrNew output.  
    - Outputs: Branches to different no-op or GitHub edit/create nodes.  
    - Edge cases: Unexpected status values would break routing.

---

#### 2.5 GitHub File Management

- **Overview:**  
  Depending on the diff status, this block either skips update, edits the existing file, or creates a new file in GitHub.

- **Nodes Involved:**  
  - Same file - Do nothing (NoOp)  
  - File is different (NoOp)  
  - File is new (NoOp)  
  - Edit existing file (GitHub)  
  - Create new file (GitHub)  

- **Node Details:**

  - **Same file - Do nothing**  
    - Type: No Operation Node  
    - Role: Terminates flow for unchanged workflows, no GitHub action.  
    - Inputs: From Check Status branch `same`.  
    - Outputs: Returns control to Return node.  

  - **File is different**  
    - Type: No Operation Node  
    - Role: Placeholder before editing GitHub file.  
    - Inputs: From Check Status branch `different`.  
    - Outputs: Connected to Edit existing file node.  

  - **File is new**  
    - Type: No Operation Node  
    - Role: Placeholder before creating new GitHub file.  
    - Inputs: From Check Status branch `new`.  
    - Outputs: Connected to Create new file node.  

  - **Edit existing file**  
    - Type: GitHub Node (edit file operation)  
    - Role: Updates existing workflow file in GitHub with new JSON content.  
    - Configuration:  
      - Owner, repo, path set dynamically from Globals and Execute Workflow Trigger data.  
      - File content is the normalized workflow JSON string from `isDiffOrNew`.  
      - Commit message includes workflow name and status.  
      - Operation: Edit file.  
    - Inputs: From File is different.  
    - Credentials: GitHub API credentials.  
    - Edge cases: Conflicts if file changed in GitHub meanwhile, API errors.

  - **Create new file**  
    - Type: GitHub Node (create file operation)  
    - Role: Creates a new workflow JSON file in GitHub.  
    - Configuration: Similar to Edit existing file but operation is create file.  
    - Inputs: From File is new.  
    - Credentials: GitHub API credentials.  
    - Edge cases: Duplicate file creation, API permissions.

---

#### 2.6 Workflow Control and Completion

- **Overview:**  
  Manages loop continuation, subworkflow execution, and marks completion.

- **Nodes Involved:**  
  - Execute Workflow (Subworkflow executor)  
  - Return (Set node)  

- **Node Details:**

  - **Execute Workflow**  
    - Type: Execute Workflow Node  
    - Role: Invokes the main workflow recursively for each batch item (batch processing).  
    - Configuration:  
      - Mode: each (processes items one by one).  
      - Workflow ID: Current workflow ID (calls itself).  
      - Inputs: Items from Loop Over Items node.  
    - Inputs: From Loop Over Items second output (after batching).  
    - Outputs: Loops back to Loop Over Items to continue processing.  
    - Edge cases: Recursive calls could lead to stack overflow if very large.

  - **Return**  
    - Type: Set Node  
    - Role: Signals completion of workflow processing by setting `Done` to true.  
    - Configuration: Sets boolean field `Done` = true.  
    - Inputs: Connected from NoOp nodes and GitHub file operations.  
    - Outputs: End of flow.  

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                               | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                       |
|-------------------------|----------------------------|-----------------------------------------------|-------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------|
| On clicking 'execute'    | Manual Trigger             | Manual start of workflow                       | None                                | n8n                                   |                                                                                                 |
| Schedule Trigger         | Schedule Trigger           | Scheduled start every 2 hours                  | None                                | n8n                                   |                                                                                                 |
| n8n                     | n8n API Node              | Retrieve all workflows                         | On clicking 'execute', Schedule Trigger | Loop Over Items                       |                                                                                                 |
| Loop Over Items          | SplitInBatches             | Batch processing of workflow items             | n8n, Execute Workflow                | Execute Workflow (batch), (loop back) |                                                                                                 |
| Execute Workflow Trigger | Execute Workflow Trigger   | Passes workflow item to subworkflow            | Merge Items                         | Globals, Merge Items                   | ## Subworkflow                                                                                   |
| Globals                  | Set Node                  | Set GitHub repo details                         | Execute Workflow Trigger            | Get file data                         | ## Backup to GitHub \nThis workflow will backup all instance workflows to GitHub. ...             |
| Get file data            | GitHub Node (get file)     | Retrieve workflow JSON file from GitHub        | Globals                            | If file too large                     |                                                                                                 |
| If file too large        | IF Node                   | Check if file content is missing or empty      | Get file data                      | Get File (true), Merge Items (false) |                                                                                                 |
| Get File                 | HTTP Request              | Download raw file content via URL               | If file too large                  | Merge Items                          |                                                                                                 |
| Merge Items              | Merge Node                | Combine original and downloaded file data      | Get File, If file too large, Execute Workflow Trigger | isDiffOrNew                        |                                                                                                 |
| isDiffOrNew              | Code Node (JS)            | Compare GitHub and n8n JSON, assign status     | Merge Items                       | Check Status                        |                                                                                                 |
| Check Status             | Switch Node               | Route based on comparison status                | isDiffOrNew                      | Same file - Do nothing, File is different, File is new |                                                                                                 |
| Same file - Do nothing   | NoOp Node                 | Skip update, workflow unchanged                 | Check Status                     | Return                              |                                                                                                 |
| File is different        | NoOp Node                 | Placeholder before editing file                  | Check Status                     | Edit existing file                   |                                                                                                 |
| File is new              | NoOp Node                 | Placeholder before creating file                 | Check Status                     | Create new file                    |                                                                                                 |
| Edit existing file       | GitHub Node (edit file)    | Update existing GitHub file                       | File is different                | Return                              |                                                                                                 |
| Create new file          | GitHub Node (create file)  | Create new file in GitHub                         | File is new                     | Return                              |                                                                                                 |
| Return                  | Set Node                  | Mark completion of processing                     | Same file - Do nothing, Edit existing file, Create new file | None                               |                                                                                                 |
| Execute Workflow         | Execute Workflow          | Subworkflow call to handle batch processing      | Loop Over Items (batch output)    | Loop Over Items                     |                                                                                                 |
| Sticky Note              | Sticky Note               | Label "Subworkflow"                              | None                            | None                               | ## Subworkflow                                                                                   |
| Sticky Note1             | Sticky Note               | Setup instructions and backup overview          | None                            | None                               | ## Backup to GitHub \nThis workflow will backup all instance workflows to GitHub. ...             |
| Sticky Note2             | Sticky Note               | Label "Main workflow loop"                       | None                            | None                               | ## Main workflow loop                                                                            |
| Sticky Note3             | Sticky Note               | Instruction to edit Globals node                 | None                            | None                               | ## Edit this node 👇                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Add node of type `Manual Trigger`. No parameters needed.

2. **Create Schedule Trigger node:**  
   - Add node of type `Schedule Trigger`.  
   - Set interval to every 2 hours.

3. **Create n8n API node:**  
   - Add node of type `n8n`.  
   - Select credential for n8n API (e.g., `n8n account`).  
   - No filters applied (empty).  
   - Connect outputs of Manual Trigger and Schedule Trigger to this node.

4. **Add SplitInBatches node:**  
   - Add node `SplitInBatches` named "Loop Over Items".  
   - Connect output of n8n API node to this node.  
   - Default batch size is fine.

5. **Add Execute Workflow node:**  
   - Add node `Execute Workflow`.  
   - Set mode to `each`.  
   - Set workflow ID to current workflow ID (calls itself).  
   - Connect second output of "Loop Over Items" to this node.  
   - Connect output of Execute Workflow back to first input of "Loop Over Items" (to continue looping).

6. **Add Execute Workflow Trigger node:**  
   - Add node `Execute Workflow Trigger`.  
   - Set input source to `passthrough`.  
   - Connect first output of "Loop Over Items" to this node.

7. **Add Set node named Globals:**  
   - Add node `Set`.  
   - Add three string fields:  
     - `repo.owner` (e.g., `john-doe`)  
     - `repo.name` (e.g., `n8n-backup`)  
     - `repo.path` (e.g., `workflows/`)  
   - Connect output of Execute Workflow Trigger to Globals.

8. **Add Merge node:**  
   - Add node `Merge`.  
   - Connect one input from Execute Workflow Trigger.  
   - Connect second input from Globals.  
   - Merge mode default (append).

9. **Add GitHub node named Get file data:**  
   - Add node `GitHub`.  
   - Set operation: `Get file`.  
   - Set owner: expression to `{{$json.repo.owner}}`.  
   - Set repository: expression to `{{$json.repo.name}}`.  
   - Set file path: expression to `{{$json.repo.path}}{{$json.id}}.json`.  
   - Enable "Continue On Fail" and "Always Output Data".  
   - Connect output of Globals to this node.

10. **Add IF node named If file too large:**  
    - Add IF node.  
    - Condition: `$json.content` is empty AND `$json.error` does not exist.  
    - Connect output of Get file data to this node.

11. **Add HTTP Request node named Get File:**  
    - Add HTTP Request node.  
    - Set URL: expression `{{$json.download_url}}`.  
    - Connect True output of If file too large to this node.

12. **Add Merge node:**  
    - Add Merge node.  
    - Connect False output of If file too large and output of Get File to this Merge node.

13. **Add Code node named isDiffOrNew:**  
    - Add Code node.  
    - Paste provided JavaScript code to compare JSON content and assign `github_status`.  
    - Connect output of Merge node to this node.

14. **Add Switch node named Check Status:**  
    - Add Switch node.  
    - Add rules for string values: `same`, `different`, `new` based on `$json.github_status`.  
    - Connect output of isDiffOrNew to Check Status.

15. **Add No Operation nodes:**  
    - Add three NoOp nodes named "Same file - Do nothing", "File is different", and "File is new".  
    - Connect Check Status outputs to these respectively.

16. **Add GitHub nodes for creating and editing files:**  
    - Add GitHub node "Edit existing file":  
      - Operation: `Edit file`.  
      - Owner, repo, path set dynamically as before.  
      - File content from `n8n_data_stringy` field.  
      - Commit message from workflow name and status.  
      - Connect from "File is different" NoOp node.  
    - Add GitHub node "Create new file":  
      - Operation: `Create file`.  
      - Same configuration as edit node.  
      - Connect from "File is new" NoOp node.

17. **Add Set node named Return:**  
    - Add Set node.  
    - Set boolean field `Done` = true.  
    - Connect outputs from "Same file - Do nothing", "Edit existing file", and "Create new file" to this node.

18. **Configure Credentials:**  
    - Ensure n8n API credentials are set for the n8n API node.  
    - Ensure GitHub API credentials are configured for GitHub nodes.

19. **Add Sticky Notes:**  
    - Add sticky notes as per original for documentation and clarity:  
      - Backup overview and setup instructions near Globals node.  
      - Label for Subworkflow near Execute Workflow Trigger node.  
      - Label for Main workflow loop near Loop Over Items and n8n API node.  
      - Instruction to edit Globals near Globals node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| This workflow is based on [Jonathan's](https://n8n.io/creators/jon-n8n/) work. Check out his templates for inspiration.                    | Workflow description                                                       |
| The workflow calls itself as a subworkflow to reduce memory usage during batch processing.                                                 | Workflow design detail                                                     |
| Make sure to update the Globals node with your GitHub username, repository name, and folder path before running.                         | Critical setup instruction                                                 |
| Folder path in GitHub (`repo.path`) will be created if it doesn't exist. Filename format is `ID.json` based on workflow ID.              | GitHub repo structure convention                                          |
| For more templates from the author, visit [https://n8n.io/creators/solomon/](https://n8n.io/creators/solomon/)                            | Additional resources and templates                                         |
| To avoid GitHub rate limits, ensure your GitHub API credentials have appropriate scopes and consider usage frequency accordingly.         | Integration consideration                                                  |
| The diffing algorithm normalizes JSON keys to avoid false positives due to key order changes.                                             | Diff checking detail                                                       |
| The workflow handles large files by downloading the raw content via URL to ensure full data retrieval.                                    | Edge case handling                                                         |

---

This documentation should provide a clear understanding of the workflow's structure, how it functions, and enable accurate reproduction or modification.