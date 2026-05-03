Backup your workflows to GitHub -- in (subfolders)

https://n8nworkflows.xyz/workflows/backup-your-workflows-to-github----in--subfolders--2868


# Backup your workflows to GitHub -- in (subfolders)

### 1. Workflow Overview

This workflow automates backing up all n8n instance workflows to a GitHub repository, organizing them into subfolders based on their tags. It is designed for users who want to safeguard their workflows externally or migrate them between servers. The workflow exports all workflows via the n8n API, then for each workflow:

- Checks if a corresponding file exists in GitHub (using the workflow ID as filename).
- Compares the existing GitHub file content with the current workflow JSON.
- Updates the file if different, creates a new file if missing, or does nothing if identical.

The workflow is logically divided into these blocks:

- **1.1 Trigger and Workflow Export:** Initiates the backup process manually or on schedule, exports workflows via n8n API.
- **1.2 Workflow Processing Loop:** Iterates over each exported workflow, determines GitHub file path based on tags, and prepares data.
- **1.3 GitHub File Retrieval and Comparison:** Retrieves existing GitHub files, compares content with current workflows to detect changes.
- **1.4 GitHub File Update or Creation:** Based on comparison, updates existing files, creates new files, or skips unchanged workflows.
- **1.5 Completion and Return:** Marks the process as done.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Workflow Export

**Overview:**  
This block starts the backup process either manually or on a schedule, then exports all workflows from the n8n instance using the n8n API node.

**Nodes Involved:**  
- Schedule Trigger  
- On clicking 'execute' (Manual Trigger)  
- n8n (n8n API node)  
- Execute Workflow (sub-workflow executor)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow daily at 7 AM.  
  - Config: Interval trigger set to hour 7 daily.  
  - Inputs: None  
  - Outputs: Connects to the n8n API node.  
  - Edge Cases: Timezone considerations may affect trigger time.

- **On clicking 'execute'**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow for immediate backup.  
  - Inputs: None  
  - Outputs: Connects to the n8n API node.  
  - Edge Cases: None significant.

- **n8n (n8n API node)**  
  - Type: n8n API  
  - Role: Retrieves all workflows from the n8n instance.  
  - Config: No filters applied; fetches all workflows.  
  - Credentials: Uses n8n API credentials named "Hirempire".  
  - Inputs: Trigger nodes (manual or schedule)  
  - Outputs: Passes workflows to Execute Workflow node.  
  - Edge Cases: API authentication failure, network issues, large data volume.

- **Execute Workflow**  
  - Type: Execute Workflow  
  - Role: Executes the current workflow for each workflow item individually (mode: each).  
  - Config: Runs the same workflow with the current workflow data as input.  
  - Inputs: Output of n8n API node (list of workflows)  
  - Outputs: Loops back to "Loop Over Items" node for batch processing.  
  - Edge Cases: Recursive execution may cause memory issues if not managed.

---

#### 1.2 Workflow Processing Loop

**Overview:**  
Processes each workflow individually, determines the GitHub repository path based on workflow tags, and prepares data for GitHub operations.

**Nodes Involved:**  
- Loop Over Items  
- tag? (Switch)  
- / (Set node to append slash to tag)  
- Globals (Set node for GitHub repo info and path)  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes workflows in batches to manage memory and API limits.  
  - Config: Default batch size (not explicitly set).  
  - Inputs: Output from Execute Workflow node.  
  - Outputs: Passes each workflow item for further processing.  
  - Edge Cases: Large batch sizes may cause timeouts or memory issues.

- **tag?**  
  - Type: Switch  
  - Role: Checks if the workflow has at least one tag.  
  - Config:  
    - Output "tag" if first tag exists.  
    - Output "none" if no tags.  
  - Inputs: Workflow JSON item.  
  - Outputs:  
    - "tag" branch connects to "/" node.  
    - "none" branch connects directly to Globals node.  
  - Edge Cases: Workflows without tags are handled gracefully.

- **/**  
  - Type: Set  
  - Role: Appends a slash to the first tag name to form a subfolder path.  
  - Config: Sets `tags[0].name` to the tag name plus "/".  
  - Inputs: From "tag?" node "tag" output.  
  - Outputs: Connects to Globals node.  
  - Edge Cases: Tag names with special characters may affect folder naming.

- **Globals**  
  - Type: Set  
  - Role: Defines GitHub repository owner, name, and path (including subfolder by tag).  
  - Config:  
    - `repo.owner`: GitHub username (e.g., "islamnazmi")  
    - `repo.name`: GitHub repository name (e.g., "n8n")  
    - `repo.path`: Folder path in repo, dynamically set as `workflows/{{ $json.tags[0].name }}`  
  - Inputs: From "/" node or directly from "tag?" node "none" output.  
  - Outputs: Passes data to GitHub file retrieval node.  
  - Edge Cases: Incorrect repo info or path may cause GitHub API errors.

---

#### 1.3 GitHub File Retrieval and Comparison

**Overview:**  
Retrieves existing workflow files from GitHub, compares them with current workflows to detect if files are new, changed, or unchanged.

**Nodes Involved:**  
- Get file data (GitHub)  
- If file too large (If)  
- Get File (HTTP Request)  
- Merge Items  
- isDiffOrNew (Code)  
- Check Status (Switch)  

**Node Details:**

- **Get file data**  
  - Type: GitHub  
  - Role: Attempts to get the existing workflow file from GitHub by path and filename (`ID.json`).  
  - Config:  
    - Owner, repo, and file path dynamically set from Globals and workflow ID.  
    - Operation: get file content.  
    - Continue on fail: true (to handle missing files gracefully).  
  - Credentials: GitHub API credentials named "islamnazmi".  
  - Inputs: From Globals node.  
  - Outputs: Passes file data or error info to "If file too large".  
  - Edge Cases: File not found, API rate limits, authentication errors.

- **If file too large**  
  - Type: If  
  - Role: Checks if the file content is empty or if an error exists, indicating a large file or missing content.  
  - Config: Checks if `content` is empty and `error` does not exist.  
  - Inputs: From Get file data.  
  - Outputs:  
    - True: Fetches file via HTTP Request (Get File node).  
    - False: Proceeds to Merge Items node.  
  - Edge Cases: Large files may not be retrievable via GitHub API file endpoint.

- **Get File**  
  - Type: HTTP Request  
  - Role: Downloads the file content directly from the `download_url` provided by GitHub API.  
  - Config: URL set dynamically from JSON `download_url`.  
  - Inputs: From If node (true branch).  
  - Outputs: Passes file content to Merge Items.  
  - Edge Cases: Network errors, invalid URLs.

- **Merge Items**  
  - Type: Merge  
  - Role: Combines data from GitHub file retrieval and current workflow JSON for comparison.  
  - Inputs: From Get File or If node (false branch) and from Execute Workflow Trigger node.  
  - Outputs: Passes combined data to isDiffOrNew node.  
  - Edge Cases: Data mismatch or missing inputs.

- **isDiffOrNew**  
  - Type: Code (JavaScript)  
  - Role: Compares the existing GitHub workflow JSON with the current workflow JSON to determine if the file is the same, different, or new.  
  - Logic:  
    - Orders JSON keys for consistent comparison.  
    - Decodes base64 content if needed.  
    - Sets `github_status` to "same", "different", or "new".  
    - Prepares stringified JSON for updates.  
  - Inputs: Merged data from Merge Items.  
  - Outputs: Passes status to Check Status node.  
  - Edge Cases: JSON parsing errors, base64 decoding issues.

- **Check Status**  
  - Type: Switch  
  - Role: Routes workflow based on `github_status` value.  
  - Config:  
    - "same" → Same file - Do nothing  
    - "different" → File is different  
    - "new" → File is new  
  - Inputs: From isDiffOrNew node.  
  - Outputs: Routes to appropriate next steps.  
  - Edge Cases: Unexpected status values.

---

#### 1.4 GitHub File Update or Creation

**Overview:**  
Updates existing files on GitHub if changed, creates new files if missing, or skips unchanged files.

**Nodes Involved:**  
- Same file - Do nothing (NoOp)  
- File is different (NoOp)  
- File is new (NoOp)  
- Edit existing file (GitHub)  
- Create new file (GitHub)  

**Node Details:**

- **Same file - Do nothing**  
  - Type: No Operation  
  - Role: Placeholder node for workflows that do not require any action.  
  - Inputs: From Check Status node "same" output.  
  - Outputs: Connects to Return node.  
  - Edge Cases: None.

- **File is different**  
  - Type: No Operation  
  - Role: Placeholder before editing existing file.  
  - Inputs: From Check Status node "different" output.  
  - Outputs: Connects to Edit existing file node.  
  - Edge Cases: None.

- **File is new**  
  - Type: No Operation  
  - Role: Placeholder before creating new file.  
  - Inputs: From Check Status node "new" output.  
  - Outputs: Connects to Create new file node.  
  - Edge Cases: None.

- **Edit existing file**  
  - Type: GitHub  
  - Role: Updates the existing workflow file content on GitHub.  
  - Config:  
    - Owner, repo, and file path from Globals and workflow ID.  
    - Operation: edit file.  
    - File content: updated JSON string from isDiffOrNew node.  
    - Commit message: workflow name with status.  
  - Credentials: GitHub API credentials "islamnazmi".  
  - Inputs: From File is different node.  
  - Outputs: Connects to Return node.  
  - Edge Cases: API errors, conflicts, authentication issues.

- **Create new file**  
  - Type: GitHub  
  - Role: Creates a new workflow file in GitHub repository.  
  - Config:  
    - Owner, repo, and file path from Globals and workflow ID.  
    - Operation: create file.  
    - File content: JSON string from isDiffOrNew node.  
    - Commit message: workflow name with status.  
  - Credentials: GitHub API credentials "islamnazmi".  
  - Inputs: From File is new node.  
  - Outputs: Connects to Return node.  
  - Edge Cases: API errors, permission issues.

---

#### 1.5 Completion and Return

**Overview:**  
Marks the completion of processing for each workflow item.

**Nodes Involved:**  
- Return (Set)  

**Node Details:**

- **Return**  
  - Type: Set  
  - Role: Sets a boolean flag `Done` to true indicating completion.  
  - Config: Assigns `Done = true`.  
  - Inputs: From Same file - Do nothing, Edit existing file, and Create new file nodes.  
  - Outputs: None (end of processing).  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role                                   | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                              |
|------------------------|-------------------------|-------------------------------------------------|------------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger         | Triggers workflow daily at 7 AM                  | None                                     | n8n                                    |                                                                                                         |
| On clicking 'execute'  | Manual Trigger           | Manual start of workflow                          | None                                     | n8n                                    |                                                                                                         |
| n8n                   | n8n API                  | Exports all workflows from n8n instance          | Schedule Trigger, On clicking 'execute'  | Execute Workflow                       |                                                                                                         |
| Execute Workflow       | Execute Workflow         | Executes workflow for each exported workflow     | Loop Over Items                          | Loop Over Items                       |                                                                                                         |
| Loop Over Items        | SplitInBatches           | Processes workflows in batches                    | Execute Workflow                        | Execute Workflow (loop), tag?         |                                                                                                         |
| tag?                   | Switch                   | Checks if workflow has tags                        | Execute Workflow Trigger                 | "/" (tag), Globals (none)              |                                                                                                         |
| /                      | Set                      | Appends slash to first tag for folder path       | tag? (tag output)                        | Globals                               |                                                                                                         |
| Globals                | Set                      | Sets GitHub repo owner, name, and path           | /, tag? (none output)                    | Get file data                        |                                                                                                         |
| Get file data          | GitHub                   | Retrieves existing workflow file from GitHub     | Globals                                | If file too large                    |                                                                                                         |
| If file too large      | If                       | Checks if file content is empty or error exists  | Get file data                          | Get File (true), Merge Items (false) |                                                                                                         |
| Get File               | HTTP Request             | Downloads file content from GitHub                | If file too large (true)                | Merge Items                         |                                                                                                         |
| Merge Items            | Merge                    | Combines GitHub file data and current workflow   | Get File, Execute Workflow Trigger      | isDiffOrNew                        |                                                                                                         |
| isDiffOrNew            | Code                     | Compares GitHub file and current workflow JSON   | Merge Items                           | Check Status                      |                                                                                                         |
| Check Status           | Switch                   | Routes based on file comparison status            | isDiffOrNew                          | Same file - Do nothing, File is different, File is new |                                                                                                         |
| Same file - Do nothing | No Operation             | No action needed for identical files              | Check Status (same)                   | Return                            |                                                                                                         |
| File is different      | No Operation             | Placeholder before editing existing file          | Check Status (different)              | Edit existing file                |                                                                                                         |
| File is new            | No Operation             | Placeholder before creating new file              | Check Status (new)                    | Create new file                  |                                                                                                         |
| Edit existing file     | GitHub                   | Updates existing file content on GitHub           | File is different                    | Return                            |                                                                                                         |
| Create new file        | GitHub                   | Creates new file on GitHub                         | File is new                        | Return                            |                                                                                                         |
| Return                 | Set                      | Marks completion of processing                     | Same file - Do nothing, Edit existing file, Create new file | None                              |                                                                                                         |
| Execute Workflow Trigger | Execute Workflow Trigger | Passes workflow data for merging and tag checking | None                                 | Merge Items, tag?                 |                                                                                                         |
| Get File               | HTTP Request             | Downloads file content from GitHub                | If file too large (true)                | Merge Items                         |                                                                                                         |
| Sticky Note            | Sticky Note              | Label: "## Subworkflow"                            | None                                     | None                                |                                                                                                         |
| Sticky Note1           | Sticky Note              | Instructions for setup and usage                   | None                                     | None                                | ## Backup to GitHub \nThis workflow will backup all instance workflows to GitHub.\n\nThe files are saved `ID.json` for the filename.\n\n### Setup\nOpen `Globals` node and update the values below 👇\n\n- **repo.owner:** your Github username\n- **repo.name:** the name of your repository\n- **repo.path:** the folder to use within the repository. If it doesn't exist it will be created.\n\n\nIf your username was `john-doe` and your repository was called `n8n-backups` and you wanted the workflows to go into a `workflows` folder you would set:\n\n- repo.owner - john-doe\n- repo.name - n8n-backups\n- repo.path - workflows/\n\n\nThe workflow calls itself using a subworkflow, to help reduce memory usage. |
| Sticky Note2           | Sticky Note              | Label: "## Main workflow loop"                     | None                                     | None                                |                                                                                                         |
| Sticky Note3           | Sticky Note              | Label: "## Edit this node 👇"                       | None                                     | None                                |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node: Set to trigger daily at 7 AM.  
   - Add a **Manual Trigger** node named "On clicking 'execute'".

2. **Add n8n API Node:**  
   - Add an **n8n API** node named "n8n".  
   - Configure with your n8n API credentials.  
   - No filters; fetch all workflows.  
   - Connect both trigger nodes to this node.

3. **Add Execute Workflow Node:**  
   - Add an **Execute Workflow** node named "Execute Workflow".  
   - Set mode to "Each" to process workflows one by one.  
   - Set workflow ID to current workflow's ID.  
   - Connect output of "n8n" node to this node.

4. **Add Loop Over Items Node:**  
   - Add a **SplitInBatches** node named "Loop Over Items".  
   - Connect output of "Execute Workflow" node to this node.

5. **Add Switch Node for Tags:**  
   - Add a **Switch** node named "tag?".  
   - Configure two outputs:  
     - "tag" if `$json.tags[0]` exists.  
     - "none" if `$json.tags[0]` does not exist.  
   - Connect "Loop Over Items" output to "tag?".

6. **Add Set Node to Append Slash:**  
   - Add a **Set** node named "/".  
   - Set `tags[0].name` to `={{ $json.tags[0].name + '/' }}`.  
   - Connect "tag?" node's "tag" output to this node.

7. **Add Globals Set Node:**  
   - Add a **Set** node named "Globals".  
   - Define three variables:  
     - `repo.owner`: your GitHub username (string).  
     - `repo.name`: your GitHub repository name (string).  
     - `repo.path`: set to `=workflows/{{ $json.tags[0].name }}` (string).  
   - Connect "/" node output and "tag?" node "none" output to this node.

8. **Add GitHub Get File Node:**  
   - Add a **GitHub** node named "Get file data".  
   - Configure with GitHub API credentials.  
   - Operation: get file.  
   - Owner: `={{ $json.repo.owner }}`  
   - Repository: `={{ $json.repo.name }}`  
   - File path: `={{ $json.repo.path }}{{ $('Execute Workflow Trigger').item.json.id }}.json`  
   - Enable "Continue on Fail" and "Always Output Data".  
   - Connect "Globals" node output to this node.

9. **Add If Node to Check File Size:**  
   - Add an **If** node named "If file too large".  
   - Condition: Check if `$json.content` is empty AND `$json.error` does not exist.  
   - Connect "Get file data" output to this node.

10. **Add HTTP Request Node to Download File:**  
    - Add an **HTTP Request** node named "Get File".  
    - Set URL to `={{ $json.download_url }}`.  
    - Connect "If file too large" true output to this node.

11. **Add Merge Node:**  
    - Add a **Merge** node named "Merge Items".  
    - Connect "Get File" output and "If file too large" false output to this node.  
    - Also connect "Execute Workflow Trigger" output to this node (for current workflow JSON).

12. **Add Code Node to Compare Files:**  
    - Add a **Code** node named "isDiffOrNew".  
    - Paste the provided JavaScript code that orders JSON keys and compares content, setting `github_status`.  
    - Connect "Merge Items" output to this node.

13. **Add Switch Node to Check Status:**  
    - Add a **Switch** node named "Check Status".  
    - Configure rules for `github_status` with outputs: "same", "different", "new".  
    - Connect "isDiffOrNew" output to this node.

14. **Add NoOp Nodes for Branches:**  
    - Add three **No Operation** nodes:  
      - "Same file - Do nothing" (for "same")  
      - "File is different" (for "different")  
      - "File is new" (for "new")  
    - Connect "Check Status" outputs accordingly.

15. **Add GitHub Nodes for File Update and Creation:**  
    - Add a **GitHub** node named "Edit existing file":  
      - Operation: edit file  
      - Owner, repo, file path, content, and commit message set dynamically as before.  
      - Credentials: GitHub API.  
      - Connect "File is different" node output to this node.

    - Add a **GitHub** node named "Create new file":  
      - Operation: create file  
      - Same dynamic configuration as above.  
      - Connect "File is new" node output to this node.

16. **Add Return Node:**  
    - Add a **Set** node named "Return".  
    - Set boolean field `Done` to true.  
    - Connect outputs of "Same file - Do nothing", "Edit existing file", and "Create new file" nodes to this node.

17. **Add Execute Workflow Trigger Node:**  
    - Add an **Execute Workflow Trigger** node named "Execute Workflow Trigger".  
    - Set input source to "passthrough".  
    - Connect it to "Merge Items" and "tag?" nodes as per original workflow.

18. **Add Sticky Notes:**  
    - Add sticky notes with instructions and labels as per original workflow for clarity.

19. **Credentials Setup:**  
    - Configure n8n API credentials for the "n8n" node.  
    - Configure GitHub API credentials for all GitHub nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Based on work by [Jonathan](https://n8n.io/creators/jon-n8n) & [Solomon](https://n8n.io/creators/solomon).                         | Original creators of the backup workflow.                                                          |
| The addition is a Set node organizing workflows into subfolders in GitHub based on tags.                                           | Enhances organization by grouping workflows in repo folders.                                       |
| The workflow calls itself as a subworkflow to reduce memory usage during batch processing.                                         | Important for scalability and performance.                                                         |
| Setup instructions are in the "Sticky Note1" node, including how to configure GitHub repo owner, name, and path.                   | See Sticky Note1 content in the workflow for detailed setup.                                       |
| Files are saved in GitHub as `ID.json` named after the workflow ID.                                                                | Ensures unique and consistent file naming.                                                        |
| The workflow handles large files by downloading them via HTTP request if GitHub API file content is empty.                         | Prevents failure on large workflow files.                                                          |
| Commit messages include the workflow name and status (same, different, new) for traceability.                                       | Helps track changes in GitHub commit history.                                                      |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Backup your workflows to GitHub -- in (subfolders)" workflow. It covers all nodes, logic blocks, configuration details, and edge cases to ensure robust operation and ease of modification.