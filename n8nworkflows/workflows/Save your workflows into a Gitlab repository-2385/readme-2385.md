Save your workflows into a Gitlab repository

https://n8nworkflows.xyz/workflows/save-your-workflows-into-a-gitlab-repository-2385


# Save your workflows into a Gitlab repository

### 1. Workflow Overview

This n8n workflow automates the process of backing up all workflows from an n8n instance into a GitLab repository. It is designed to synchronize workflows by retrieving them, comparing their current states with the saved versions in GitLab, and then conditionally creating or updating files in the repository based on detected changes. The workflow provides detailed status outputs for each workflow processed, identifying if it is new, unchanged, modified, or if an error occurred during processing.

The workflow logic is organized into the following blocks:

- **1.1 Initialization and Configuration:** Receives manual trigger and sets global parameters for GitLab repository details.
- **1.2 Workflows Retrieval:** Fetches all workflows from the n8n instance.
- **1.3 Iteration Over Workflows:** Processes each workflow individually in batch mode.
- **1.4 File Retrieval & Comparison:** Attempts to fetch corresponding workflow files from GitLab, parses them, and compares them to the current workflow to determine status.
- **1.5 Conditional File Operations:** Based on the comparison status, creates new files, updates existing files, or skips unchanged workflows.
- **1.6 Status Reporting:** Aggregates and outputs the status of each workflow processed.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Configuration

**Overview:**  
This block is triggered manually and sets up the global parameters needed for GitLab operations, including repository owner, name, branch, and directory path.

**Nodes Involved:**  
- When clicking "Test Workflow"  
- Globals

**Node Details:**

- **When clicking "Test Workflow"**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution of the workflow.  
  - Configuration: Default, no parameters.  
  - Inputs: None  
  - Outputs: Triggers "Globals" node.  
  - Edge cases: None.

- **Globals**  
  - Type: Set  
  - Role: Defines global variables with repository details.  
  - Configuration: Sets four string fields:  
    - `repo.owner`: GitLab user/team slug (e.g., "owner-slug")  
    - `repo.name`: GitLab repository slug (e.g., "repo-slug")  
    - `repo.branch`: Target branch for commits (e.g., "branch-slug")  
    - `repo.path`: Target directory path inside the repo, must end with "/" (e.g., "path/")  
  - Inputs: From manual trigger  
  - Outputs: Feeds into "Retrieve all workflows"  
  - Edge cases: Ensure correct slug and path formatting to avoid GitLab API errors.

---

#### 1.2 Workflows Retrieval

**Overview:**  
Retrieves all workflows from the connected n8n instance to prepare for backup.

**Nodes Involved:**  
- Retrieve all workflows

**Node Details:**

- **Retrieve all workflows**  
  - Type: n8n API  
  - Role: Fetches a list of all workflows available in the n8n instance.  
  - Configuration: No filters applied; retrieves all workflows.  
  - Credentials: Requires an n8n API credential with read access.  
  - Inputs: From "Globals" node  
  - Outputs: Sends batch of workflows to "Loop Over Workflows"  
  - Edge cases: Possible API rate limits or access issues.

---

#### 1.3 Iteration Over Workflows

**Overview:**  
Processes workflows one by one to handle individual backup operations sequentially.

**Nodes Involved:**  
- Loop Over Workflows  
- Current workflow  
- End Loop

**Node Details:**

- **Loop Over Workflows**  
  - Type: SplitInBatches  
  - Role: Iterates through each workflow in batches of one item (default).  
  - Configuration: Default batch size (1) to process workflows individually.  
  - Inputs: From "Retrieve all workflows"  
  - Outputs: Sends current batch item to "Result" and "Current workflow"  
  - Edge cases: Large numbers of workflows might increase runtime.

- **Current workflow**  
  - Type: NoOp  
  - Role: Placeholder node to pass current workflow data downstream for GitLab operations.  
  - Inputs: From "Loop Over Workflows"  
  - Outputs: Feeds into "Get file" node  
  - Edge cases: None.

- **End Loop**  
  - Type: NoOp  
  - Role: Marks end of batch processing, feeding back to "Loop Over Workflows" to continue iteration.  
  - Inputs: From status nodes ("Status new", "Status diff", "Status same", "Status error")  
  - Outputs: Back to "Loop Over Workflows"  
  - Edge cases: None.

---

#### 1.4 File Retrieval & Comparison

**Overview:**  
For each workflow, tries to retrieve its JSON file from GitLab, then compares the saved version with the current workflow object, ignoring certain runtime or metadata fields, to detect changes.

**Nodes Involved:**  
- Get file  
- Extract From File  
- Error output to normal output  
- Save each version in a different field  
- File status  
- Sticky Note (Check file)

**Node Details:**

- **Get file**  
  - Type: GitLab  
  - Role: Fetches the workflow JSON file from GitLab repository if it exists.  
  - Configuration:  
    - Operation: Get file by path  
    - File Path: Combination of `repo.path` and workflow id with `.json` extension  
    - Branch: From `repo.branch`  
    - Owner and repository from globals  
    - On error: Continue (do not stop workflow on error)  
  - Credentials: GitLab API credential required  
  - Inputs: From "Current workflow"  
  - Outputs: Success to "Extract From File", error to "Error output to normal output"  
  - Edge cases: File not found error handled gracefully; other errors logged as status "error".

- **Extract From File**  
  - Type: ExtractFromFile  
  - Role: Parses the binary file content retrieved from GitLab into JSON and stores it in `workflow-from-gitlab` field.  
  - Inputs: From "Get file"  
  - Outputs: To "Save each version in a different field"  
  - Edge cases: Invalid JSON content may cause parsing errors.

- **Error output to normal output**  
  - Type: NoOp  
  - Role: Converts error output from "Get file" to normal output for further processing.  
  - Inputs: From "Get file" error path  
  - Outputs: To "File status"  
  - Edge cases: Ensures workflow continues even if GitLab file retrieval fails.

- **Save each version in a different field**  
  - Type: Set  
  - Role: Stores two versions of the workflow in the item JSON for comparison:  
    - `workflow-from-gitlab`: The saved workflow JSON from GitLab  
    - `workflow-from-n8n`: The current workflow JSON from n8n instance  
  - Inputs: From "Extract From File"  
  - Outputs: To "File status"  
  - Edge cases: None.

- **File status**  
  - Type: Code  
  - Role: Compares the two workflow objects ignoring the fields `updatedAt` and `global` to determine if the workflow is new, unchanged, modified, or in error.  
  - Configuration: Custom JavaScript code runs once per item; sets `status` field with values: `"new"`, `"same"`, `"diff"`, or `"error"`.  
  - Inputs: From "Save each version in a different field" or "Error output to normal output"  
  - Outputs: To "Switch"  
  - Edge cases: Potential JS execution errors; handles "file not found" as `"new"` status.

- **Sticky Note (Check file)**  
  - Type: Sticky Note  
  - Role: Documentation within the canvas explaining the logic of getting the file, error handling, and status assignment.

---

#### 1.5 Conditional File Operations

**Overview:**  
Takes appropriate action based on the workflow status: creates new files for new workflows, updates files for modified workflows, does nothing for unchanged, and logs errors encountered.

**Nodes Involved:**  
- Switch  
- Create file  
- New file version  
- Status new  
- Status diff  
- Status same  
- Status error  
- Status error (Set node)  
- Status new (Set node)  
- Status diff (Set node)  
- Status same (Set node)  
- End Loop  
- Sticky Note1 (Save the data)

**Node Details:**

- **Switch**  
  - Type: Switch  
  - Role: Routes items based on `status` field (`new`, `same`, `diff`, `error`).  
  - Inputs: From "File status"  
  - Outputs: Four outputs connected to "Create file", "Status same", "New file version", and "Status error" respectively.  
  - Edge cases: Any unknown or fallback status routed to "error" path.

- **Create file**  
  - Type: GitLab  
  - Role: Creates a new file in GitLab repository for workflows marked as `"new"`.  
  - Configuration:  
    - Operation: Create file  
    - File content: JSON stringified current workflow with pretty formatting  
    - Commit message indicates creation with workflow name  
    - Commit author set to n8n with noreply email  
    - Branch, owner, repo, and file path derived from globals and workflow id  
    - On error: Continue (does not stop workflow)  
  - Inputs: From "Switch" (new)  
  - Outputs: Success to "Status new", error to "Status error"  
  - Edge cases: GitLab API errors such as permission denied, rate limits.

- **New file version**  
  - Type: GitLab  
  - Role: Updates existing file in GitLab for workflows marked as `"diff"`.  
  - Configuration:  
    - Operation: Edit file  
    - File content: JSON stringified current workflow  
    - Commit message indicates update with workflow name  
    - Commit author set as above  
    - Branch, owner, repo, and file path derived similarly  
    - On error: Continue  
  - Inputs: From "Switch" (diff)  
  - Outputs: Success to "Status diff", error to "Status error"  
  - Edge cases: Similar to Create file node.

- **Status new, diff, same, error (Set nodes)**  
  - Type: Set  
  - Role: Standardizes output status object with fields:  
    - `name`: Workflow name from current workflow context  
    - `status`: Status string (e.g., "new", "diff", "same", or "Error : <error message>")  
  - Inputs: From respective preceding nodes (Create file, New file version, Switch output, or error paths)  
  - Outputs: Each leads to "End Loop" to continue processing remaining workflows  
  - Edge cases: None.

- **End Loop**  
  - As described earlier, loops back to process next workflow.

- **Sticky Note1 (Save the data)**  
  - Documentation explaining this block is responsible for saving workflows as new or updated files, or ignoring them if unchanged or errored.

---

#### 1.6 Status Reporting

**Overview:**  
Outputs the final aggregated list of statuses after all workflows have been processed.

**Nodes Involved:**  
- Result

**Node Details:**

- **Result**  
  - Type: NoOp  
  - Role: Collects and outputs the final status summary for all workflows processed in the batch iteration.  
  - Inputs: From "Loop Over Workflows" (main output after each batch)  
  - Outputs: Endpoint of the workflow for viewing results  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                                      | Input Node(s)                     | Output Node(s)                                | Sticky Note                                                                                                           |
|-----------------------------|-------------------------|-----------------------------------------------------|----------------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking "Test Workflow" | Manual Trigger          | Entry point to start the workflow manually          | None                             | Globals                                       |                                                                                                                       |
| Globals                     | Set                     | Sets global GitLab repo parameters                   | When clicking "Test Workflow"     | Retrieve all workflows                        |                                                                                                                       |
| Retrieve all workflows      | n8n API                 | Retrieves all workflows from n8n instance            | Globals                         | Loop Over Workflows                           |                                                                                                                       |
| Loop Over Workflows         | SplitInBatches          | Iterates over workflows one by one                    | Retrieve all workflows            | Result, Current workflow                       |                                                                                                                       |
| Current workflow            | NoOp                    | Passes current workflow item                          | Loop Over Workflows              | Get file                                      |                                                                                                                       |
| Get file                   | GitLab                  | Retrieves workflow JSON file from GitLab              | Current workflow                 | Extract From File (success), Error output to normal output (error) | See Sticky Note: "Check file" — explains file fetching and error handling                                              |
| Extract From File           | ExtractFromFile          | Parses GitLab file content into JSON                   | Get file                        | Save each version in a different field        |                                                                                                                       |
| Error output to normal output | NoOp                  | Converts error output to normal for processing         | Get file (error)                 | File status                                   |                                                                                                                       |
| Save each version in a different field | Set           | Saves both GitLab and n8n workflow versions for comparison | Extract From File                | File status                                   |                                                                                                                       |
| File status                | Code                    | Compares workflow versions, sets workflow status       | Save each version..., Error output to normal output | Switch                                       | See Sticky Note: "Check file" — explains comparison logic                                                            |
| Switch                     | Switch                  | Routes workflow based on status                         | File status                    | Create file (new), Status same, New file version (diff), Status error |                                                                                                                       |
| Create file                | GitLab                  | Creates new workflow file in GitLab                     | Switch (new)                    | Status new (success), Status error (failure) |                                                                                                                       |
| New file version           | GitLab                  | Updates existing workflow file in GitLab                | Switch (diff)                   | Status diff (success), Status error (failure) |                                                                                                                       |
| Status new                 | Set                     | Sets status output for new file creation                | Create file                    | End Loop                                      |                                                                                                                       |
| Status diff                | Set                     | Sets status output for updated file                      | New file version               | End Loop                                      |                                                                                                                       |
| Status same                | Set                     | Sets status output for unchanged workflow                | Switch                        | End Loop                                      |                                                                                                                       |
| Status error               | Set                     | Sets status output for errored workflows                 | Switch, Create file (error), New file version (error) | End Loop                                      |                                                                                                                       |
| End Loop                   | NoOp                    | Marks end of processing for one workflow batch          | Status new, Status diff, Status same, Status error | Loop Over Workflows                           |                                                                                                                       |
| Result                     | NoOp                    | Outputs final status list                                | Loop Over Workflows            | None                                          |                                                                                                                       |
| Sticky Note (Check file)   | Sticky Note             | Documentation on file retrieval and comparison logic    | None                          | None                                          | Content: "## Check file\nGet the file.\nUse error output as normal output.\nSome code to analyse the file and set a status." |
| Sticky Note1 (Save the data) | Sticky Note           | Documentation on saving workflow files                   | None                          | None                                          | Content: "## Save the data\nSave the data as new or edited file, ignored or note as error."                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Add a **Manual Trigger** node named "When clicking \"Test Workflow\"".
   - No special parameters.

2. **Create Globals Node:**
   - Add a **Set** node named "Globals".
   - Configure to set four string fields:  
     - `repo.owner` (e.g., "owner-slug")  
     - `repo.name` (e.g., "repo-slug")  
     - `repo.branch` (e.g., "branch-slug")  
     - `repo.path` (e.g., "path/") — ensure trailing slash  
   - Connect output of manual trigger to this node.

3. **Create Retrieve all workflows Node:**
   - Add an **n8n API** node named "Retrieve all workflows".
   - Operation: List workflows without filters.
   - Credential: Select an n8n API credential with read access.
   - Connect output of "Globals" to this node.

4. **Add Loop Over Workflows Node:**
   - Add a **SplitInBatches** node named "Loop Over Workflows".
   - Default batch size (1).
   - Connect output of "Retrieve all workflows" to this node.

5. **Create Result Node:**
   - Add a **NoOp** node named "Result".
   - Connect first output of "Loop Over Workflows" (batch completion) to this node.

6. **Create Current workflow Node:**
   - Add a **NoOp** node named "Current workflow".
   - Connect second output of "Loop Over Workflows" (current item) to this node.

7. **Add Get file Node:**
   - Add a **GitLab** node named "Get file".
   - Operation: Get file.
   - Parameters:  
     - Owner: `={{ $('Globals').first().json.repo.owner }}`  
     - Repository: `={{ $('Globals').first().json.repo.name }}`  
     - Branch/Reference: `={{ $('Globals').first().json.repo.branch }}`  
     - File path: `={{ $('Globals').first().json.repo.path }}{{ $json.id }}.json`  
   - Set `onError` to "continueErrorOutput" to allow workflow continuation on errors.
   - Set "Execute Once" to true.
   - Credential: Select GitLab API credential.
   - Connect output of "Current workflow" to this node.

8. **Add Extract From File Node:**
   - Add an **ExtractFromFile** node named "Extract From File".
   - Operation: From JSON.
   - Binary Property Name: `file-from-gitlab`.
   - Destination Key: `workflow-from-gitlab`.
   - Connect main output of "Get file" to this node.

9. **Add Error output to normal output Node:**
   - Add a **NoOp** node named "Error output to normal output".
   - Connect error output of "Get file" to this node.

10. **Add Save each version in a different field Node:**
    - Add a **Set** node named "Save each version in a different field".
    - Add two fields:  
      - `workflow-from-gitlab` (type: Object) set to `={{ $json["workflow-from-gitlab"] }}`  
      - `workflow-from-n8n` (type: Object) set to `={{ $('Current workflow').item.json }}`  
    - Connect output of "Extract From File" to this node.
    - Connect output of "Error output to normal output" also to this node (merging both paths).

11. **Add File status Node:**
    - Add a **Code** node named "File status".
    - Mode: Run Once For Each Item.
    - Paste the provided JavaScript code that compares `workflow-from-n8n` and `workflow-from-gitlab` ignoring `updatedAt` and `global` fields, setting `status` as `"new"`, `"same"`, `"diff"`, or `"error"`.
    - Connect output of "Save each version in a different field" to this node.

12. **Add Switch Node:**
    - Add a **Switch** node named "Switch".
    - Add rules for `status` field with outputs:  
      - `"new"` → output 0  
      - `"same"` → output 1  
      - `"diff"` → output 2  
      - fallback renamed `"error"` → output 3  
    - Connect output of "File status" to this node.

13. **Add Create file Node:**
    - Add a **GitLab** node named "Create file".
    - Operation: Create file.
    - Parameters:  
      - Owner, repository, branch, file path from Globals as before.  
      - File content: `={{ JSON.stringify($('Current workflow').item.json, null, 4) }}`  
      - Commit message: `"Create file for workflow {{ $('Current workflow').item.json.name }}"`  
      - Commit author: Name "n8n", Email "noreply-n8n@mipih.fr"  
    - Set `onError` to "continueErrorOutput".
    - Execute once: true.
    - Credential: GitLab API credential.
    - Connect output 0 ("new") of "Switch" to this node.

14. **Add New file version Node:**
    - Add a **GitLab** node named "New file version".
    - Operation: Edit file.
    - Parameters:  
      - Owner, repo, branch, file path from Globals.  
      - File content: `={{ JSON.stringify($json['workflow-from-n8n'], null, 4) }}`  
      - Commit message: `"New file version for workflow {{ $json['workflow-from-n8n'].name }}"`  
      - Commit author same as above.  
    - Set `onError` to "continueErrorOutput".
    - Execute once: true.
    - Credential: GitLab API credential.
    - Connect output 2 ("diff") of "Switch" to this node.

15. **Add Status new Node:**
    - Add a **Set** node named "Status new".
    - Fields:  
      - `name`: `={{ $('Current workflow').item.json.name }}`  
      - `status`: `"new"`  
    - Include: None (only set these fields).
    - Connect success output of "Create file" to this node.

16. **Add Status diff Node:**
    - Add a **Set** node named "Status diff".
    - Fields:  
      - `name`: `={{ $('Current workflow').item.json.name }}`  
      - `status`: `"diff"`  
    - Connect success output of "New file version" to this node.

17. **Add Status same Node:**
    - Add a **Set** node named "Status same".
    - Fields:  
      - `name`: `={{ $('Current workflow').item.json.name }}`  
      - `status`: `"same"`  
    - Connect output 1 ("same") of "Switch" to this node.

18. **Add Status error Node:**
    - Add a **Set** node named "Status error".
    - Fields:  
      - `name`: `={{ $('Current workflow').item.json.name }}`  
      - `status`: `"=Error : {{ $json.error }}"`  
    - Connect output 3 ("error") of "Switch", and error outputs of "Create file" and "New file version" to this node.

19. **Add End Loop Node:**
    - Add a **NoOp** node named "End Loop".
    - Connect outputs of "Status new", "Status diff", "Status same", and "Status error" to this node.
    - Connect output of "End Loop" back to "Loop Over Workflows" to process next batch.

20. **Connect Loop Over Workflows first output to Result Node:**
    - This outputs the final aggregated list of workflow statuses after all batches processed.

21. **Add Sticky Notes (optional):**
    - Add two sticky notes for documentation:  
      - Near "Get file" and "File status": "Check file\nGet the file.\nUse error output as normal output.\nSome code to analyse the file and set a status."  
      - Near Switch and GitLab file operations: "Save the data\nSave the data as new or edited file, ignored or note as error."

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This template is inspired by community workflows for GitHub backup: [Save your workflows into a GitHub repository](https://n8n.io/workflows/817-save-your-workflows-into-a-github-repository/) by hikerspath and [Back Up Your n8n Workflows To Github](https://n8n.io/workflows/1534-back-up-your-n8n-workflows-to-github/) by jon-n8n. | Original inspirations for GitLab adaptation.                                                                        |
| Error handling on GitLab nodes is designed not to stop workflow execution but to mark the workflow status as error in results. | Important for resilience during GitLab API failures or network issues.                                              |
| Fields ignored during workflow comparison: `updatedAt` and `global` to avoid false positives due to runtime metadata changes. | Prevents unnecessary commits when only runtime or last update timestamps differ.                                    |
| Ensure GitLab credentials have sufficient permissions to read, create, and edit files in the target repository and branch.   | Credential setup prerequisite.                                                                                       |
| The repository path should end with a slash `/` to correctly form file paths.                                                | Prevents path concatenation errors.                                                                                  |

---

This completes the detailed analysis and documentation of the "Save your workflows into a Gitlab repository" workflow. The instructions and explanations provided support both manual recreation and automated understanding/modification of the workflow.