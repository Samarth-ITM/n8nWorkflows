Backup workflows to git repository on Github

https://n8nworkflows.xyz/workflows/backup-workflows-to-git-repository-on-github-2532


# Backup workflows to git repository on Github

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows by exporting each workflow as a JSON file and committing it to a specified GitHub repository. It is designed to run on a schedule and ensures that all workflows are backed up with their current state. The workflow handles three main cases for each workflow:

- **New workflows** that do not yet have corresponding files in the repository are created as new files.
- **Unchanged workflows** are skipped to avoid unnecessary commits.
- **Changed workflows** with modifications compared to the repository version are updated.

The workflow does not currently handle deletions or renaming of workflows, so old files may persist in the repository.

Logical blocks:

- **1.1 Scheduled Trigger and Global Setup**: Initiates workflow runs and sets global repository variables.
- **1.2 Fetching Workflows**: Retrieves all workflows from n8n.
- **1.3 Iteration over Workflows**: Processes workflows one by one.
- **1.4 GitHub File Existence Check**: Checks if a workflow backup file exists in the repo.
- **1.5 Content Comparison and Decision Making**: Compares existing file content vs current workflow JSON.
- **1.6 File Creation or Update**: Creates new files or updates existing files in GitHub accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Global Setup

- **Overview:**  
  This block triggers the workflow execution on a scheduled interval and sets global variables for GitHub repository configuration.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Globals

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `scheduleTrigger`  
    - Role: Starts workflow execution automatically at defined intervals (every N minutes).  
    - Configuration: Interval set on minutes (default unspecified here).  
    - Inputs: None  
    - Outputs: Connects to Globals node.  
    - Potential Failures: None generally unless n8n scheduler is disabled.

  - **Globals**  
    - Type: `set`  
    - Role: Defines global variables for GitHub repository owner, repository name, and backup folder path.  
    - Configuration:  
      - `repo.owner` = GitHub username (e.g., "shashikanth171")  
      - `repo.name` = Repository name (e.g., "n8n-backup")  
      - `repo.path` = Folder path in repo (e.g., "workflows/")  
    - Inputs: Receives trigger from Schedule Trigger  
    - Outputs: Connects to n8n node  
    - Edge cases: Variables must be correct to avoid GitHub API errors.

#### 1.2 Fetching Workflows

- **Overview:**  
  Retrieves all existing workflows from the connected n8n instance using the n8n API credentials.

- **Nodes Involved:**  
  - n8n

- **Node Details:**

  - **n8n**  
    - Type: `n8n` (n8n API node)  
    - Role: Fetches all workflows without any filter.  
    - Configuration: Empty filters, uses configured n8n API credentials.  
    - Inputs: From Globals node  
    - Outputs: Connects to Loop Over Items node  
    - Edge cases: API authentication failure, network timeout.

#### 1.3 Iteration over Workflows

- **Overview:**  
  Splits the array of workflows into individual items for sequential processing.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - Type: `splitInBatches`  
    - Role: Processes workflows one at a time (batch size = 1 by default).  
    - Configuration: Default options; executes once per workflow item.  
    - Inputs: From n8n node  
    - Outputs: Two outputs: first empty (unused), second connected to GitHub node.  
    - Edge cases: Large number of workflows may slow processing.

#### 1.4 GitHub File Existence Check

- **Overview:**  
  Attempts to get the backup file for the current workflow from the GitHub repository to determine if it exists.

- **Nodes Involved:**  
  - GitHub

- **Node Details:**

  - **GitHub**  
    - Type: `github` (GitHub API node)  
    - Role: Fetches file content by path (e.g., `workflows/[workflow_name].json`) in the repo.  
    - Configuration:  
      - Owner, repository, and path dynamically set from Globals and current workflow name.  
      - Operation: `get` file content (no binary)  
      - Continue on fail: true (important to allow processing to continue if file missing)  
    - Inputs: From Loop Over Items (workflow item)  
    - Outputs: Connects to If node  
    - Edge cases:  
      - 404 error if file does not exist (expected for new workflows)  
      - GitHub authentication errors  
      - Rate limit exceeded errors

#### 1.5 Content Comparison and Decision Making

- **Overview:**  
  Determines if the workflow file exists, and if it does, compares content to see if an update is necessary.

- **Nodes Involved:**  
  - If  
  - Code  
  - If1

- **Node Details:**

  - **If**  
    - Type: `if` node  
    - Role: Checks if the GitHub node returned an error (file missing).  
    - Condition: `$json.error` does not exist → file exists; else new file needed.  
    - Inputs: From GitHub node  
    - Outputs:  
      - True (file missing) → Create new file node  
      - False (file exists) → Code node  
    - Edge cases: Expression failures if `$json.error` not reliable.

  - **Code**  
    - Type: `code` (JavaScript)  
    - Role: Decodes the base64 content of the existing repo file to UTF-8 string for comparison.  
    - Key code:  
      ```js
      for (item of items) {
          item.json.content = Buffer.from(item.json.content, 'base64').toString('utf8');
      }
      return items;
      ```  
    - Inputs: From If node (file exists branch)  
    - Outputs: To If1 node  
    - Edge cases: Malformed base64 content could cause errors.

  - **If1**  
    - Type: `if` node  
    - Role: Compares existing file content (`$json.content`) with current workflow JSON string.  
    - Condition:  
      - `$json.content` not equal to workflow JSON string (from Loop Over Items)  
    - Outputs:  
      - True (content changed) → Update file node  
      - False (content unchanged) → Loop Over Items (next workflow)  
    - Edge cases: String comparisons may fail if formatting differs but content is same.

#### 1.6 File Creation or Update

- **Overview:**  
  Creates new backup files or updates existing ones in the GitHub repository with the current workflow JSON.

- **Nodes Involved:**  
  - Create new file and commit  
  - Update file content and commit

- **Node Details:**

  - **Create new file and commit**  
    - Type: `github`  
    - Role: Creates a new file in the repository with workflow JSON content and commits it.  
    - Configuration:  
      - Owner, repo, and path from Globals + workflow name  
      - File content: current workflow JSON string  
      - Commit message: `[N8N Backup] [workflow_name].json`  
    - Inputs: From If node (file missing branch)  
    - Outputs: To Loop Over Items for next item  
    - Edge cases: File creation conflicts if file created concurrently; GitHub API errors.

  - **Update file content and commit**  
    - Type: `github`  
    - Role: Updates existing file content and commits changes.  
    - Configuration: Same as create node, but operation is `edit`.  
    - Inputs: From If1 node (content changed branch)  
    - Outputs: To Loop Over Items for next item  
    - Edge cases: Version conflicts if file changed externally; GitHub rate limits.

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                              | Input Node(s)           | Output Node(s)                | Sticky Note                                      |
|-----------------------------|---------------------------|----------------------------------------------|-------------------------|------------------------------|-------------------------------------------------|
| Schedule Trigger            | scheduleTrigger           | Initiates workflow run on a schedule         | None                    | Globals                      |                                                 |
| Globals                    | set                       | Sets GitHub repo owner, name, and path       | Schedule Trigger        | n8n                         | Set variables                                   |
| n8n                        | n8n                       | Retrieves all workflows from n8n              | Globals                 | Loop Over Items              | Get all workflows                              |
| Loop Over Items            | splitInBatches            | Iterates over workflows one by one            | n8n                     | GitHub (2nd output), empty   |                                                 |
| GitHub                     | github                    | Checks if workflow backup file exists         | Loop Over Items          | If                          | Check if file exists in the repository          |
| If                         | if                        | Branches based on file existence               | GitHub                   | Code (file exists), Create new file (file missing) |                                                 |
| Code                       | code                      | Decodes base64 file content for comparison    | If (file exists)         | If1                         | Convert the file contents to JSON string        |
| If1                        | if                        | Compares repo file content with workflow JSON | Code                     | Update file (changed), Loop Over Items (unchanged) | Check if there are any changes in the workflow |
| Create new file and commit | github                    | Creates new file and commits to GitHub        | If (file missing)        | Loop Over Items             | Create a new file for the workflow               |
| Update file content and commit | github                    | Updates existing file content and commits     | If1 (content changed)    | Loop Over Items             | Workflow changes committed to the repository    |
| Sticky Note                | stickyNote                | Informational notes                            | -                       | -                           | Multiple notes: see below                         |

**Sticky Notes Content (linked to nodes):**

- Schedule Trigger, Globals: "Set variables"
- n8n: "Get all workflows"
- Loop Over Items, GitHub: "Check if file exists in the repository"
- If: "Check if file exists in the repository"
- Code: "Convert the file contents to JSON string"
- If1: "Check if there are any changes in the workflow"
- Create new file and commit: "Create a new file for the workflow"
- Update file content and commit: "Workflow changes committed to the repository"

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Type: `scheduleTrigger`  
   - Set interval (e.g., every 15 minutes or as needed).

3. **Add a Set node named "Globals":**  
   - Connect the Schedule Trigger to Globals.  
   - Add string variables:  
     - `repo.owner`: GitHub username (e.g., "shashikanth171")  
     - `repo.name`: GitHub repository name (e.g., "n8n-backup")  
     - `repo.path`: Path within repo (e.g., "workflows/").

4. **Add an n8n node to fetch all workflows:**  
   - Connect Globals to this n8n node.  
   - Leave filters empty to get all workflows.  
   - Configure credentials to connect to your n8n instance (API access).

5. **Add a SplitInBatches node named "Loop Over Items":**  
   - Connect n8n node to this node.  
   - Use default batch size (1) to process workflows one at a time.

6. **Add a GitHub node to check for existing backup file:**  
   - Connect second output of Loop Over Items to this node.  
   - Set resource to `file` and operation to `get`.  
   - Configure:  
     - Owner: `={{$node["Globals"].json["repo"]["owner"]}}`  
     - Repository: `={{$node["Globals"].json["repo"]["name"]}}`  
     - File Path: `={{$node["Globals"].json["repo"]["path"]}}{{$json["name"]}}.json`  
   - Use GitHub credentials with repo write access.  
   - Enable "Continue On Fail" to handle missing files gracefully.

7. **Add an If node:**  
   - Connect GitHub node to If.  
   - Condition: Check if `$json.error` does not exist → file exists; if exists, go to Code node, else create new file.

8. **Add a Code node to decode base64 content:**  
   - Connect If node's "file exists" output to Code node.  
   - JavaScript code:  
     ```js
     let items = $input.all();
     for (item of items) {
         item.json.content = Buffer.from(item.json.content, 'base64').toString('utf8');
     }
     return items;
     ```

9. **Add a second If node (If1):**  
   - Connect Code node output to If1 node.  
   - Condition: Compare decoded content with current workflow JSON string:  
     - Left: `$json.content`  
     - Right: `={{ $('Loop Over Items').item.json.toJsonString() }}`  
     - Operator: not equals  
   - True output → Update file node  
   - False output → Loop Over Items (to process next workflow)

10. **Add a GitHub node to create new file:**  
    - Connect If node's "file missing" output to this node.  
    - Operation: `create` (default)  
    - Configure similar to the check node but add:  
      - File Content: `={{ $('Loop Over Items').item.json.toJsonString() }}`  
      - Commit Message: `=[N8N Backup] {{ $('Loop Over Items').item.json.name }}.json`

11. **Add a GitHub node to update existing file:**  
    - Connect If1's "content changed" output to this node.  
    - Operation: `edit`  
    - Configure similarly to create node (owner, repo, path, content, commit message).

12. **Connect both create and update GitHub nodes back to Loop Over Items node:**  
    - This loops the process for all workflows.

13. **Configure all GitHub nodes with the same GitHub credentials** that have write access to the repository.

14. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                               |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow source code is maintained at: https://code.swecha.org/shashikanth/n8n-templates/-/tree/main/backup-worflows | Original source repository for this backup workflow           |
| Current limitations: Renamed or deleted workflows in n8n do not remove or rename files in GitHub | Consider adding a cleanup job or manual maintenance           |
| Use GitHub personal access token with repo scope for GitHub credentials in n8n               | GitHub OAuth or PAT with correct scopes is required           |
| This workflow is suitable for scheduled, automated backups of n8n workflows                  | Ideal for versioning and audit trails of workflow changes     |

---

This document fully describes the "Backup workflows to git repository" n8n workflow, enabling users or AI agents to understand, reproduce, and troubleshoot the backup process with GitHub integration.