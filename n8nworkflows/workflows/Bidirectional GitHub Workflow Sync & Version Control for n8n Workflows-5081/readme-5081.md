Bidirectional GitHub Workflow Sync & Version Control for n8n Workflows

https://n8nworkflows.xyz/workflows/bidirectional-github-workflow-sync---version-control-for-n8n-workflows-5081


# Bidirectional GitHub Workflow Sync & Version Control for n8n Workflows

### 1. Workflow Overview

This workflow provides a **bidirectional synchronization and version control system between n8n workflows and a GitHub repository**. Its purpose is to ensure that workflows stored in n8n and those saved as JSON files in GitHub remain consistent, automatically updating the out-of-date source in either system.

**Target Use Cases:**

- Teams or individuals managing n8n workflows who want version control and backup via GitHub.
- Automated synchronization to avoid manual exports/imports.
- Detecting which version (n8n or GitHub) is newer and updating the other accordingly.

**Logical Blocks:**

- **1.1 Trigger & Initialization:** Scheduled trigger and setting GitHub repository details.
- **1.2 GitHub File Listing & Filtering:** Fetch workflows JSON files from GitHub repo and filter out placeholders.
- **1.3 n8n Workflows Fetch:** Retrieve all workflows currently in n8n.
- **1.4 Comparison:** Decode GitHub files, compare datasets from GitHub and n8n workflows, and identify differences.
- **1.5 Conditional Sync Decision:** Decide if n8n or GitHub has the newer version for each workflow.
- **1.6 Update GitHub:** Convert updated n8n workflows to JSON, encode, and commit updates to GitHub.
- **1.7 Update n8n:** Create or update workflows within n8n from GitHub files.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

- **Overview:** Starts workflow execution on a scheduled weekly basis and sets GitHub repository details to be used downstream.
- **Nodes Involved:** `Schedule Trigger`, `Set GitHub Details`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Configuration: Weekly trigger on Mondays
    - Inputs: none (start node)
    - Outputs: triggers `Set GitHub Details`
    - Potential failures: scheduling misconfiguration or instance downtime.
  
  - **Set GitHub Details**
    - Type: Set
    - Configuration: Sets static variables for GitHub account name, repo name, and workflow directory path (empty by default, user must fill)
    - Inputs: from `Schedule Trigger`
    - Outputs: triggers `n8n` node and `List files from repo`
    - Edge cases: empty or incorrect repo details will cause GitHub API calls to fail.

#### 1.2 GitHub File Listing & Filtering

- **Overview:** Lists all files in the GitHub repo workflow directory, then filters out files named "placeholder" to avoid processing them.
- **Nodes Involved:** `List files from repo`, `Filter`, `GitHub`
- **Node Details:**

  - **List files from repo**
    - Type: GitHub
    - Operation: List files in the specified repo path.
    - Inputs: from `Set GitHub Details`
    - Outputs: triggers `Filter`
    - Failures: GitHub API rate limits, authentication issues.
  
  - **Filter**
    - Type: Filter
    - Configuration: Excludes files whose name contains "placeholder"
    - Inputs: from `List files from repo`
    - Outputs: triggers `GitHub`
  
  - **GitHub**
    - Type: GitHub
    - Operation: Get file content for each filtered file
    - Inputs: from `Filter`
    - Outputs: triggers `Decode to json`
    - Failures: file not found, permission errors

#### 1.3 n8n Workflows Fetch

- **Overview:** Retrieves all workflows currently stored in the n8n instance.
- **Nodes Involved:** `n8n`
- **Node Details:**

  - **n8n**
    - Type: n8n API node
    - Operation: List all workflows with no filters
    - Inputs: from `Set GitHub Details`
    - Outputs: triggers `n8n vs GitHub`
    - Potential failures: n8n API errors or permission issues

#### 1.4 Comparison

- **Overview:** Decodes GitHub workflow files, then compares the datasets from GitHub and n8n workflows by workflow `id`.
- **Nodes Involved:** `Decode to json`, `n8n vs GitHub`
- **Node Details:**

  - **Decode to json**
    - Type: Set
    - Configuration: Decodes base64 encoded GitHub file content into JSON
    - Inputs: from `GitHub`
    - Outputs: triggers `n8n vs GitHub`
    - Edge cases: malformed base64 or JSON content causing decode failure.
  
  - **n8n vs GitHub**
    - Type: CompareDatasets
    - Configuration: Compares datasets by matching workflow `id` fields.
    - Inputs: from `n8n` and `Decode to json`
    - Outputs:
      - New/changed in n8n (to JSON file conversion)
      - Only in n8n workflows
      - Differences where GitHub is newer (conditional branch)
      - Only in GitHub workflows
    - Edge cases: mismatched IDs, missing fields
    

#### 1.5 Conditional Sync Decision

- **Overview:** For workflows present in both but with different update timestamps, decide which source is newer and branch accordingly.
- **Nodes Involved:** `If n8n before GitHub`, `Code - InputA`, `Code - InputB`
- **Node Details:**

  - **If n8n before GitHub**
    - Type: If
    - Condition: Checks if n8n workflow `updatedAt` date is before GitHub file's `updatedAt` date
    - Inputs: from `n8n vs GitHub` (difference branch)
    - Outputs:
      - True branch: GitHub is newer → update n8n
      - False branch: n8n is newer → update GitHub
    - Edge cases: invalid or missing date fields
    
  - **Code - InputA** (for n8n newer workflows)
    - Type: Code (Python)
    - Purpose: From workflows only in n8n, extract matching workflow dataset by ID
    - Inputs: from `If n8n before GitHub` (false branch)
    - Outputs: triggers `Edit Fields`
  
  - **Code - InputB** (for GitHub newer workflows)
    - Type: Code (Python)
    - Purpose: From workflows only in GitHub, extract matching workflow dataset by ID
    - Inputs: from `If n8n before GitHub` (true branch)
    - Outputs: triggers `Update workflow in n8n`
    - Edge cases: no matches found, returns empty result

#### 1.6 Update GitHub

- **Overview:** Converts n8n workflows to JSON files, encodes them to base64, and uploads or updates the corresponding files in GitHub.
- **Nodes Involved:** `Edit Fields`, `Json file1`, `To base`, `Edit for Update`, `Update file`
- **Node Details:**

  - **Edit Fields**
    - Type: Set
    - Configuration: Prepares JSON dataset for file creation
    - Inputs: from `Code - InputA`
    - Outputs: triggers `Json file1`
  
  - **Json file1**
    - Type: ConvertToFile
    - Operation: Converts JSON data into formatted JSON file (one per item)
    - Inputs: from `Edit Fields`
    - Outputs: triggers `To base`
  
  - **To base**
    - Type: ExtractFromFile
    - Operation: Converts binary JSON file content to base64 property
    - Inputs: from `Json file1`
    - Outputs: triggers `Edit for Update`
  
  - **Edit for Update**
    - Type: Set
    - Configuration: Sets file name (normalized from workflow name) and commit date
    - Inputs: from `To base`
    - Outputs: triggers `Update file`
  
  - **Update file**
    - Type: GitHub
    - Operation: Edit existing workflow JSON file in GitHub repo with new content and commit message
    - Inputs: from `Edit for Update`
    - Failures: GitHub API errors, branch protection, permission issues

#### 1.7 Update n8n

- **Overview:** Creates new workflows in n8n from GitHub files if they only exist in GitHub. Also updates existing workflows in n8n if GitHub is newer.
- **Nodes Involved:** `Json file`, `To base64`, `Edit for Upload`, `Upload file`, `Create new workflow in n8n`, `Update workflow in n8n`
- **Node Details:**

  - **Json file**
    - Type: ConvertToFile
    - Operation: Converts JSON data to a JSON file for upload
    - Inputs: from `n8n vs GitHub` (new only in GitHub)
    - Outputs: triggers `To base64`
  
  - **To base64**
    - Type: ExtractFromFile
    - Operation: Converts JSON file content to base64 property
    - Inputs: from `Json file`
    - Outputs: triggers `Edit for Upload`
  
  - **Edit for Upload**
    - Type: Set
    - Configuration: Sets file name (normalized workflow name) and commit date
    - Inputs: from `To base64`
    - Outputs: triggers `Upload file`
  
  - **Upload file**
    - Type: GitHub
    - Operation: Create new JSON workflow file in GitHub repo
    - Inputs: from `Edit for Upload`
    - Outputs: none
    - Failures: GitHub API errors, permission issues
  
  - **Create new workflow in n8n**
    - Type: n8n
    - Operation: Create new workflow in n8n from JSON payload
    - Inputs: from `n8n vs GitHub` (only in GitHub branch)
    - Outputs: none
    - Edge cases: workflow JSON invalid, n8n API errors
  
  - **Update workflow in n8n**
    - Type: n8n
    - Operation: Update existing workflow in n8n by ID with new JSON
    - Inputs: from `Code - InputB`
    - Outputs: none
    - Failures: workflow not found, API errors

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                                   | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                     |
|-----------------------|-------------------------|-------------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger        | Starts workflow on weekly schedule               | -                          | Set GitHub Details             |                                                                                                |
| Set GitHub Details     | Set                     | Sets repo details for GitHub API calls           | Schedule Trigger            | n8n, List files from repo      |                                                                                                |
| n8n                   | n8n                     | Retrieves all workflows from n8n                  | Set GitHub Details          | n8n vs GitHub                  |                                                                                                |
| List files from repo   | GitHub                  | Lists files in GitHub workflows directory         | Set GitHub Details          | Filter                        |                                                                                                |
| Filter                | Filter                  | Filters out placeholder files                      | List files from repo        | GitHub                        |                                                                                                |
| GitHub                | GitHub                  | Gets file content for each valid workflow file    | Filter                     | Decode to json                |                                                                                                |
| Decode to json        | Set                     | Decodes base64 GitHub file content to JSON        | GitHub                     | n8n vs GitHub                 |                                                                                                |
| n8n vs GitHub          | CompareDatasets         | Compares workflows from GitHub and n8n by ID      | n8n, Decode to json         | Json file, If n8n before GitHub, Create new workflow in n8n | Sticky Note: Check which is newer (GitHub or n8n workflows)                                    |
| Json file             | ConvertToFile           | Converts differing workflows to JSON files        | n8n vs GitHub              | To base64                     |                                                                                                |
| To base64             | ExtractFromFile         | Encodes JSON files to base64 for upload           | Json file                  | Edit for Upload               |                                                                                                |
| Edit for Upload       | Set                     | Sets file name and commit date for new uploads    | To base64                  | Upload file                  |                                                                                                |
| Upload file           | GitHub                  | Uploads new workflow JSON file to GitHub          | Edit for Upload            | -                            |                                                                                                |
| Create new workflow in n8n | n8n                 | Creates new workflow in n8n from GitHub file      | n8n vs GitHub              | -                            | Sticky Note: IF ONLY IN GITHUB - Create new workflow in n8n                                    |
| If n8n before GitHub   | If                      | Checks if n8n workflow is older than GitHub file  | n8n vs GitHub              | Code - InputB (true), Code - InputA (false) | Sticky Note: Check which is newer; true = GitHub newer, false = n8n newer                      |
| Code - InputB         | Code (Python)           | Extracts GitHub newer workflow dataset            | If n8n before GitHub (true) | Update workflow in n8n        |                                                                                                |
| Update workflow in n8n | n8n                     | Updates existing n8n workflow                      | Code - InputB              | -                            |                                                                                                |
| Code - InputA         | Code (Python)           | Extracts n8n newer workflow dataset                | If n8n before GitHub (false) | Edit Fields                 |                                                                                                |
| Edit Fields           | Set                     | Prepares n8n workflow JSON for GitHub upload      | Code - InputA              | Json file1                   |                                                                                                |
| Json file1            | ConvertToFile           | Converts n8n workflow JSON to file                 | Edit Fields                | To base                      |                                                                                                |
| To base               | ExtractFromFile         | Encodes JSON files to base64 for update            | Json file1                 | Edit for Update              |                                                                                                |
| Edit for Update       | Set                     | Sets file name and commit date for updates         | To base                    | Update file                  |                                                                                                |
| Update file           | GitHub                  | Updates existing workflow JSON file in GitHub      | Edit for Update            | -                            |                                                                                                |
| Sticky Note           | Sticky Note             | Provides logic explanation for comparison block   | -                          | -                            | Check which is newer: If TRUE (GitHub newer, update n8n), If FALSE (n8n newer, update GitHub)  |
| Sticky Note1          | Sticky Note             | Explains logic for workflows only in n8n          | -                          | -                            | IF ONLY IN N8N - Update GitHub with relevant workflows                                         |
| Sticky Note2          | Sticky Note             | Explains logic for workflows only in GitHub       | -                          | -                            | IF ONLY IN GITHUB - Create new workflow in n8n                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node**
   - Type: Schedule Trigger
   - Configure to trigger weekly on Monday.
   - Connect output to next node.

3. **Add a Set node named "Set GitHub Details"**
   - Define three string fields:
     - `github_account_name` (GitHub username or org)
     - `github_repo_name` (GitHub repository name)
     - `repo_workflows_path` (path in repo where workflows JSON files are stored)
   - Connect input from Schedule Trigger.
   - Connect outputs to:
     - n8n node (to fetch workflows)
     - GitHub node (to list files)

4. **Add an n8n node**
   - Operation: List workflows (no filters)
   - Input from "Set GitHub Details"
   - Output to comparison node later.

5. **Add a GitHub node named "List files from repo"**
   - Operation: List files
   - Owner: expression from `github_account_name`
   - Repo: expression from `github_repo_name`
   - File path: expression from `repo_workflows_path`
   - Output to Filter node.

6. **Add a Filter node**
   - Condition: File name does NOT contain "placeholder"
   - Input: from "List files from repo"
   - Output: to GitHub get file content node.

7. **Add a GitHub node named "GitHub"**
   - Operation: Get file content for each file from filtered list.
   - Owner, Repo, File Path from previous nodes.
   - Output to Set node for decoding.

8. **Add a Set node named "Decode to json"**
   - Operation: Set JSON output to base64 decode of file content.
   - Input from GitHub file content node.
   - Output to CompareDatasets node.

9. **Add a CompareDatasets node named "n8n vs GitHub"**
   - InputA: from n8n workflows
   - InputB: from decoded GitHub files
   - Merge by field: workflow `id`
   - Configure outputs for:
     - Only in n8n
     - Only in GitHub
     - Differences (records in both but different)
   - Outputs to:
     - JSON conversion for upload to GitHub
     - Conditional decision node
     - Create new workflows for GitHub-only.

10. **Add a Set node named "Json file"**
    - Converts differing workflows to JSON files.
    - Input from "n8n vs GitHub"
    - Output to base64 encoding.

11. **Add an ExtractFromFile node "To base64"**
    - Converts JSON file content to base64 property.
    - Output to Set node for upload.

12. **Add a Set node "Edit for Upload"**
    - Assign variables:
      - `commitDate`: current date/time formatted `dd-MM-yyyy/H:mm`
      - `fileName`: workflow name normalized (lowercase, spaces to dashes)
    - Output to GitHub upload node.

13. **Add a GitHub node "Upload file"**
    - Operation: Create new file in GitHub repo.
    - Inputs: from Set node.
    - Commit message: `n8n_backup_sync-{{commitDate}}`
    - Branch: main

14. **Add a node "Create new workflow in n8n"**
    - Operation: Create workflow
    - Input: JSON workflow from GitHub-only files.
    - Connect from "n8n vs GitHub" only in GitHub branch.

15. **Add an If node "If n8n before GitHub"**
    - Condition: Compare `updatedAt` timestamps between n8n and GitHub
    - True branch: GitHub newer → update n8n
    - False branch: n8n newer → update GitHub

16. **Add two Code nodes (Python):**
    - **Code - InputB** (true branch): Extract workflow dataset from GitHub newer items for update in n8n.
    - **Code - InputA** (false branch): Extract workflow dataset from n8n newer items for upload to GitHub.

17. **Add an n8n node "Update workflow in n8n"**
    - Operation: Update workflow by ID with JSON dataset.
    - Input from Code - InputB.

18. **Add a Set node "Edit Fields"**
    - Prepare JSON for upload.
    - Input from Code - InputA.
    - Output to ConvertToFile node.

19. **Add a ConvertToFile node "Json file1"**
    - Converts JSON to formatted JSON file.
    - Output to ExtractFromFile.

20. **Add an ExtractFromFile node "To base"**
    - Convert JSON file to base64.
    - Output to Set node.

21. **Add a Set node "Edit for Update"**
    - Sets commitDate and fileName for update.
    - Output to GitHub edit file node.

22. **Add a GitHub node "Update file"**
    - Edit existing file in GitHub repo.
    - Commit message includes commitDate.
    - Input from Set node.

23. **Add Sticky Notes at strategic points explaining:**
    - Logic for deciding newer workflow source.
    - Instructions for handling workflows only found in n8n or GitHub.

24. **Configure all GitHub nodes with appropriate OAuth2 credentials.**

25. **Test the workflow end-to-end and verify error handling for:**
    - Missing or invalid GitHub repo details.
    - GitHub API rate limiting or auth errors.
    - Invalid or missing workflow JSON files.
    - n8n API permission errors.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow synchronizes n8n workflows with GitHub for version control and backup.             | Primary purpose                                                                                         |
| Make sure GitHub OAuth2 credentials have proper repo access and scopes.                          | GitHub API integration                                                                                  |
| Workflow relies on consistent workflow `id` fields and JSON formatting between n8n and GitHub.  | Data consistency requirement                                                                             |
| For help with n8n GitHub nodes, see https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.github/ | Official n8n documentation                                                                              |
| Workflow uses Python code nodes for data filtering; ensure Python environment is enabled.        | n8n environment setup                                                                                   |
| The sticky note "CHECK WHICH IS NEWER" clarifies the core comparison logic in the workflow.     | Visual aid inside the workflow                                                                          |

---

*Disclaimer:* The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.