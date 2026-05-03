Automated Daily Backup of n8n Workflows to GitLab Repositories

https://n8nworkflows.xyz/workflows/automated-daily-backup-of-n8n-workflows-to-gitlab-repositories-4035


# Automated Daily Backup of n8n Workflows to GitLab Repositories

### 1. Workflow Overview

**Purpose:**  
This workflow automates the daily backup of all workflows from a self-hosted n8n instance into a GitLab repository. It ensures that each n8n workflow is stored as a JSON file in GitLab and updated only if there are changes, providing version control and backup.

**Target Use Cases:**  
- Version-controlling n8n workflows for audit and rollback purposes.  
- Automating backup processes to avoid manual exports.  
- Keeping GitLab repositories in sync with the current state of n8n workflows.  

**Logical Blocks:**  
- **1.1 Scheduling & GitLab Repository Info Initialization:** Trigger and initialize repository metadata.  
- **1.2 Retrieve Workflows from n8n:** Fetch all workflows currently in the n8n instance.  
- **1.3 Iterate Over Workflows:** Process each workflow individually through a batch loop.  
- **1.4 Retrieve Corresponding File from GitLab:** Get the stored workflow file from GitLab.  
- **1.5 Compare Workflows:** Check if the local n8n workflow differs from the GitLab version.  
- **1.6 Conditional Actions (Create/Update/Skip/Error):** Based on comparison, create new files, update existing ones, or skip if unchanged.  
- **1.7 Status Handling and Loop Continuation:** Manage status updates and loop control for each workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & GitLab Repository Info Initialization

- **Overview:**  
  This block triggers the workflow on a scheduled basis and retrieves GitLab repository information, storing repo-related global variables used throughout the workflow.

- **Nodes Involved:**  
  - Schedule Trigger  
  - GitLab (Get repository info)  
  - Globals (Set repository metadata)

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule trigger node  
    - Configuration: Runs once daily (interval setting with empty defaults, implying daily run)  
    - Input: None (trigger only)  
    - Output: Triggers downstream nodes  
    - Potential Failures: Scheduler misconfiguration, node downtime  

  - **GitLab (Get repository info)**  
    - Type: GitLab node  
    - Operation: Get repository metadata (owner, name, default branch)  
    - Configuration: Fixed repository name (`n8n-workflows`), owner must be configured properly  
    - Credentials: GitLab API PAT with `api` permission  
    - Input: Trigger from Schedule  
    - Output: Repository details JSON  
    - Potential Failures: Authentication failure, wrong repository or owner, network issues  

  - **Globals (Set repository metadata)**  
    - Type: Set node  
    - Purpose: Extract and set global variables from GitLab repo info for reuse (owner, name, branch, path)  
    - Configuration: Uses expressions referencing GitLab node output fields  
    - Input: GitLab node output  
    - Output: JSON with repository metadata  
    - Potential Failures: Expression evaluation errors if fields are missing  

---

#### 1.2 Retrieve Workflows from n8n

- **Overview:**  
  Retrieves all workflows from the connected n8n instance via its REST API.

- **Nodes Involved:**  
  - Retrieve all workflows (n8n node)

- **Node Details:**  
  - **Retrieve all workflows**  
    - Type: n8n node (n8n API integration)  
    - Operation: List workflows using `/rest/workflows` API  
    - Credentials: n8n API credentials with access to workflows  
    - Input: From Globals node  
    - Output: Array of workflow JSONs  
    - Potential Failures: Authentication errors, API endpoint issues, no workflows found  

---

#### 1.3 Iterate Over Workflows

- **Overview:**  
  Processes all workflows one by one using batch splitting to enable sequential handling.

- **Nodes Involved:**  
  - Loop Over Workflows (SplitInBatches)  
  - Result (NoOp)  
  - Current workflow (NoOp)

- **Node Details:**  
  - **Loop Over Workflows**  
    - Type: SplitInBatches node  
    - Configuration: Default batch size (defaults to 1), iterates over workflows array  
    - Input: Output from Retrieve all workflows  
    - Output: Single workflow per iteration  
    - Potential Failures: Batch handling errors, empty input array  

  - **Result**  
    - Type: NoOp node  
    - Role: Placeholder for collecting final outputs or for workflow design clarity  
    - Input: Loop Over Workflows output (first output)  
    - Output: None further connected  
    - Potential Failures: None  

  - **Current workflow**  
    - Type: NoOp node  
    - Role: Marks the current workflow being processed, used as input for next node  
    - Input: Loop Over Workflows output (second output)  
    - Output: Connected to Get file node  
    - Potential Failures: None  

---

#### 1.4 Retrieve Corresponding File from GitLab

- **Overview:**  
  For each workflow, fetch the matching JSON file from GitLab repository to compare against the current n8n workflow.

- **Nodes Involved:**  
  - Get file (GitLab node)  
  - Extract From File (Extract JSON from binary)  
  - Error output to normal output (NoOp)

- **Node Details:**  
  - **Get file**  
    - Type: GitLab node  
    - Operation: Get file content by path `${workflow_id}.json` from GitLab repo  
    - Configuration: Uses global repo owner/name/branch, file path based on current workflow ID  
    - OnError: Continue with error output enabled  
    - Credentials: GitLab PAT with repository read permissions  
    - Input: Current workflow  
    - Output: File content in binary property or error object  
    - Potential Failures: File not found (404), authentication errors, rate limits  

  - **Extract From File**  
    - Type: ExtractFromFile node  
    - Operation: Converts GitLab file binary content from JSON string to JSON object under `workflow-from-gitlab` field  
    - Input: Output from Get file (file binary)  
    - Output: JSON object of stored workflow  
    - Potential Failures: Malformed JSON, empty file content  

  - **Error output to normal output**  
    - Type: NoOp node  
    - Role: Converts error output path from Get file into normal output for further processing  
    - Input: Error output from Get file  
    - Output: Connected to File status  
    - Potential Failures: None  

---

#### 1.5 Compare Workflows

- **Overview:**  
  Compares the current n8n workflow JSON with the GitLab stored version, determining if they are identical, differ, or if the file is new.

- **Nodes Involved:**  
  - Save each version in a different field (Set)  
  - File status (Code node)  
  - Switch (Conditional routing)

- **Node Details:**  
  - **Save each version in a different field**  
    - Type: Set node  
    - Role: Stores the current n8n workflow JSON as `workflow-from-n8n` and GitLab workflow JSON as `workflow-from-gitlab` for comparison  
    - Input: Output of Extract From File and Current workflow  
    - Output: Prepared data for comparison  
    - Potential Failures: Expression errors if inputs missing  

  - **File status**  
    - Type: Code node (JavaScript)  
    - Role: Compares two workflow objects excluding certain dynamic fields (`updatedAt`, `global`) to decide status: `new`, `diff`, `same`, or `error`  
    - Key Logic: Recursive object comparison ignoring metadata fields  
    - Input: Set node output  
    - Output: Adds `status` field indicating comparison result  
    - Potential Failures: Code errors, unexpected JSON shapes  

  - **Switch**  
    - Type: Switch node  
    - Role: Routes workflows based on `status` field:  
      - `new` → Create file in GitLab  
      - `same` → Mark status same (skip update)  
      - `diff` → Update existing file in GitLab  
      - `error` (fallback) → Mark error status  
    - Input: File status output  
    - Output: Four directional paths for conditional handling  
    - Potential Failures: Misrouted conditions, missing status value  

---

#### 1.6 Conditional Actions (Create/Update/Skip/Error)

- **Overview:**  
  Performs GitLab file creation or update as needed, or sets status accordingly for unchanged or error cases.

- **Nodes Involved:**  
  - Create file (GitLab node)  
  - New file version (GitLab node)  
  - Status same (Set node)  
  - Status diff (Set node)  
  - Status error (Set node)

- **Node Details:**  
  - **Create file**  
    - Type: GitLab node  
    - Operation: Create new file in the repository with the workflow JSON content  
    - Configuration: File path = `${workflow_id}.json`, commit message includes workflow name, author configured as `n8n`  
    - OnError: Continue error output  
    - Input: Switch node `new` output  
    - Output: Success or error info  
    - Potential Failures: File already exists, auth errors, rate limiting  

  - **New file version**  
    - Type: GitLab node  
    - Operation: Edit existing file in GitLab repository with updated workflow JSON  
    - Configuration: File path based on workflow ID, commit message with workflow name  
    - OnError: Continue error output  
    - Input: Switch node `diff` output  
    - Output: Success or error info  
    - Potential Failures: File missing (race condition), auth errors  

  - **Status same**  
    - Type: Set node  
    - Purpose: Sets status `same` for workflows with no changes detected  
    - Input: Switch node `same` output  
    - Output: Connected to End Loop  
    - Potential Failures: None  

  - **Status diff**  
    - Type: Set node  
    - Purpose: Sets status `diff` for workflows updated in GitLab  
    - Input: New file version output  
    - Output: Connected to End Loop  
    - Potential Failures: None  

  - **Status error**  
    - Type: Set node  
    - Purpose: Sets status with error message when failures occur  
    - Input: Switch node `error` output or error outputs from Create/New file version nodes  
    - Output: Connected to End Loop  
    - Potential Failures: None  

---

#### 1.7 Status Handling and Loop Continuation

- **Overview:**  
  Finalizes processing for each workflow by setting status and continuing the batch loop.

- **Nodes Involved:**  
  - End Loop (NoOp)

- **Node Details:**  
  - **End Loop**  
    - Type: NoOp node  
    - Role: Marks end of processing for one workflow and loops back to next batch item  
    - Input: Status same, status diff, or status error nodes  
    - Output: Loop Over Workflows node to continue iteration  
    - Potential Failures: None  

---

### 3. Summary Table

| Node Name                | Node Type              | Functional Role                                  | Input Node(s)                   | Output Node(s)                 | Sticky Note                                         |
|--------------------------|------------------------|-------------------------------------------------|--------------------------------|-------------------------------|----------------------------------------------------|
| Schedule Trigger         | Schedule Trigger        | Initiates workflow daily trigger                 | None                           | GitLab                        |                                                    |
| GitLab                   | GitLab                 | Retrieves GitLab repository metadata             | Schedule Trigger               | Globals                       |                                                    |
| Globals                  | Set                    | Sets global repo variables from GitLab info      | GitLab                        | Retrieve all workflows         |                                                    |
| Retrieve all workflows   | n8n                    | Fetches all workflows from n8n API               | Globals                       | Loop Over Workflows            |                                                    |
| Loop Over Workflows      | SplitInBatches         | Iterates workflows one by one                     | Retrieve all workflows         | Result, Current workflow       |                                                    |
| Result                   | NoOp                   | Placeholder/collector node                         | Loop Over Workflows            | None                         |                                                    |
| Current workflow         | NoOp                   | Marks current workflow for processing             | Loop Over Workflows            | Get file                      |                                                    |
| Get file                 | GitLab                 | Fetches workflow file from GitLab                  | Current workflow              | Extract From File, Error output to normal output |                                                    |
| Extract From File        | ExtractFromFile        | Parses GitLab file JSON from binary                | Get file                      | Save each version in a different field |                                                    |
| Error output to normal output | NoOp                | Converts error output to normal output             | Get file (error output)        | File status                   |                                                    |
| Save each version in a different field | Set         | Stores both workflow versions for comparison      | Extract From File, Current workflow | File status                 |                                                    |
| File status              | Code                   | Compares workflows JSON excluding dynamic fields | Save each version              | Switch                       |                                                    |
| Switch                   | Switch                 | Routes based on comparison status                  | File status                   | Create file, Status same, New file version, Status error |                                                    |
| Create file              | GitLab                 | Creates new workflow file in GitLab                | Switch (new)                  | Status error (on failure)     |                                                    |
| New file version         | GitLab                 | Updates existing workflow file in GitLab           | Switch (diff)                 | Status diff, Status error     |                                                    |
| Status same              | Set                    | Marks workflow as unchanged                         | Switch (same)                 | End Loop                     |                                                    |
| Status diff              | Set                    | Marks workflow as updated                           | New file version              | End Loop                     |                                                    |
| Status error             | Set                    | Marks workflow with error status                    | Switch (error), Create file, New file version | End Loop           |                                                    |
| End Loop                 | NoOp                   | Ends iteration for current workflow and loops      | Status same, Status diff, Status error | Loop Over Workflows      |                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run daily (default interval). No special parameters needed.

2. **Add a GitLab node to get repository info**  
   - Type: GitLab  
   - Operation: Get repository ("repository" resource, "get" operation)  
   - Set Owner to your GitLab username or group name.  
   - Set Repository to your target repo (e.g., `n8n-workflows`).  
   - Credentials: Configure GitLab API credentials with a PAT having `api` scope.  
   - Connect from Schedule Trigger node.

3. **Add a Set node named "Globals"**  
   - Extract and set fields:  
     - `repo.owner` = `{{$json.owner.username}}`  
     - `repo.name` = `{{$json.name}}`  
     - `repo.branch` = `{{$json.default_branch}}`  
     - `repo.path` = `{{$json.web_url}}`  
   - Connect from GitLab node.

4. **Add an n8n node to retrieve all workflows**  
   - Type: n8n  
   - Operation: List workflows (default). No filters needed.  
   - Credentials: Use valid n8n API credentials.  
   - Connect from Globals node.

5. **Add a SplitInBatches node named "Loop Over Workflows"**  
   - Default batch size 1 (process workflows one at a time).  
   - Connect from Retrieve all workflows node.

6. **Add two NoOp nodes:**  
   - "Result" (for output collection, optional) connected to first output of Loop Over Workflows.  
   - "Current workflow" connected to second output of Loop Over Workflows.

7. **Add GitLab node "Get file" to fetch workflow file**  
   - Resource: File, Operation: Get  
   - Owner: `={{$('Globals').first().json.repo.owner}}`  
   - Repository: `={{$('Globals').first().json.repo.name}}`  
   - Branch: `={{$('Globals').first().json.repo.branch}}`  
   - File Path: `={{$json.id}}.json` (workflow ID as filename)  
   - Credentials: Same GitLab PAT credentials  
   - On Error: Continue with error output  
   - Connect from Current workflow node.

8. **Add ExtractFromFile node "Extract From File"**  
   - Operation: fromJson  
   - Binary Property Name: `file-from-gitlab` (matches GitLab node binary output)  
   - Destination Key: `workflow-from-gitlab`  
   - Connect from Get file main output.

9. **Add NoOp node "Error output to normal output"**  
   - Connect from Get file error output to this node, then connect this node to next step.

10. **Add Set node "Save each version in a different field"**  
    - Set two fields:  
      - `workflow-from-gitlab` = `{{$json["workflow-from-gitlab"]}}`  
      - `workflow-from-n8n` = `{{$('Current workflow').item.json}}`  
    - Connect from Extract From File and also from Error output to normal output.

11. **Add Code node "File status"**  
    - JavaScript code compares `workflow-from-n8n` and `workflow-from-gitlab`  
    - Ignores `updatedAt` and `global` fields during comparison.  
    - Adds `status` field: `new`, `same`, `diff`, or `error`  
    - Connect from Set node.

12. **Add Switch node "Switch"**  
    - Condition on `status` field with outputs:  
      - `new`  
      - `same`  
      - `diff`  
      - fallback `error`  
    - Connect from File status node.

13. **Add GitLab node "Create file"** (for `new`)  
    - Resource: File, Operation: Create  
    - Owner, repo, branch same as above (use Globals variables)  
    - File Path: `={{$json.id}}.json`  
    - File Content: `={{JSON.stringify($('Current workflow').item.json, null, 4)}}`  
    - Commit Message: `Create file for workflow {{$json.name}}`  
    - Author: name `n8n`, email `akhil@nuevesolutions.com` (can be customized)  
    - On Error: Continue error output  
    - Credentials: GitLab PAT  
    - Connect from Switch node `new` output.

14. **Add GitLab node "New file version"** (for `diff`)  
    - Resource: File, Operation: Edit  
    - Owner, repo, branch same as above  
    - File Path: `={{$json['workflow-from-n8n'].id}}.json`  
    - File Content: `={{JSON.stringify($json['workflow-from-n8n'], null, 4)}}`  
    - Commit Message: `New file version for workflow {{$json['workflow-from-n8n'].name}}`  
    - On Error: Continue error output  
    - Credentials: GitLab PAT  
    - Connect from Switch node `diff` output.

15. **Add three Set nodes for statuses:**  
    - "Status same" sets `name` and `status` = `same`  
    - "Status diff" sets `name` and `status` = `diff`  
    - "Status error" sets `name` and `status` = `Error : {{ $json.error }}`  
    - Connect from Switch node `same` to Status same, from New file version to Status diff, and from Switch node `error` and Create file error output to Status error.

16. **Add NoOp node "End Loop"**  
    - Connect from Status same, Status diff, and Status error nodes.  
    - Connect output back to Loop Over Workflows node to continue batch processing.

17. **Set all GitLab credentials and n8n API credentials before executing.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                               |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| The workflow uses GitLab Personal Access Token (PAT) with `api` and `write_repository` scopes.  | GitLab API documentation                                     |
| Ignore dynamic fields like `updatedAt` and `global` to reduce false positive diffs.              | Custom code in File status node                              |
| This workflow does not handle merge conflicts or manual GitLab edits separately.                 | Workflow disclaimer                                          |
| Use the workflow with a safe test environment before applying in production.                      | Disclaimer                                                  |
| For more details on GitLab API calls in n8n, refer to https://docs.gitlab.com/ee/api/            | GitLab official API docs                                    |
| Use the n8n REST API `/rest/workflows` to fetch workflows, requiring n8n API credentials.        | n8n API documentation                                       |

---

**Disclaimer:**  
This document describes a fully automated n8n workflow for backing up workflows to GitLab. It respects all current content policies and uses only publicly accessible and legal API calls. Users are responsible for proper credential management and environment testing.