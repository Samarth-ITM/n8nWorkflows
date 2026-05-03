Backup your credentials to GitHub

https://n8nworkflows.xyz/workflows/backup-your-credentials-to-github-2307


# Backup your credentials to GitHub

---
### 1. Workflow Overview

This workflow automates backing up all n8n instance credentials to a GitHub repository, storing each credential as a separate JSON file named by its ID. It leverages the n8n CLI to export all credentials in decrypted form, then iteratively checks if corresponding files exist in the target GitHub repo and path. For each credential file, it updates the file if it exists and differs, creates it if new, or skips if identical.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Export Credentials:** Initiates the backup by manual execution or scheduled trigger and exports all credentials via CLI.
- **1.2 Parse & Loop Credentials:** Parses exported credentials JSON and loops over each credential item.
- **1.3 Globals Setup:** Holds GitHub repo configuration parameters.
- **1.4 Per-Credential GitHub File Check:** For each credential, checks if an existing file is present on GitHub.
- **1.5 File Comparison & Decision:** Compares local credential data with GitHub file content to decide update/create/skip.
- **1.6 GitHub File Operations:** Updates or creates files on GitHub accordingly.
- **1.7 Workflow Recursion & Return:** Uses a sub-workflow call to handle batch processing and returns completion status.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Export Credentials

- **Overview:** Starts the workflow either manually or via schedule and exports all credentials decrypted using the n8n CLI command.
- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger  
  - Execute Command  
  - JSON formatting

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of backup  
    - Config: No parameters  
    - Input: None  
    - Output: Triggers Execute Command  
    - Failures: None expected

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers export every 2 hours  
    - Config: Interval set to every 2 hours  
    - Input: None  
    - Output: Triggers Execute Command  
    - Failures: None expected

  - **Execute Command**  
    - Type: Execute Command  
    - Role: Runs CLI command to export all decrypted credentials using `npx n8n export:credentials --all --decrypted`  
    - Config: Command string set accordingly  
    - Input: Trigger node output  
    - Output: Raw CLI stdout containing exported credentials JSON string  
    - Failure: Command failure, no valid output, CLI errors

  - **JSON formatting**  
    - Type: Code node  
    - Role: Parses and beautifies CLI output JSON string into structured JSON array of credential objects  
    - Config: Extracts JSON string from stdout using regex, parses, and returns array of JSON objects each wrapped as item  
    - Input: CLI output JSON string  
    - Output: Array of credential JSON objects for next processing  
    - Failures: Invalid JSON in CLI output, regex fails to find JSON, JSON parse errors

#### 1.2 Parse & Loop Credentials

- **Overview:** Splits exported credentials into individual items and iteratively processes each using sub-workflow calls to reduce memory usage.
- **Nodes Involved:**  
  - Loop Over Items  
  - Execute Workflow

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits large array of credentials into batches for iterative processing  
    - Config: Default batch size (not specified, assumed 1 per iteration)  
    - Input: Parsed credentials array  
    - Output: Single credential item per batch iteration  
    - Failures: None expected

  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Calls this workflow recursively for each credential item to handle it independently  
    - Config: Mode set to "each" so each item triggers one execution  
    - Input: Single credential item  
    - Output: Feeds back to Loop Over Items for next batch  
    - Failures: Workflow invocation failure, recursion limit exceeded

#### 1.3 Globals Setup

- **Overview:** Holds static configuration for the GitHub repository owner, repository name, and path within repo.
- **Nodes Involved:**  
  - Globals

- **Node Details:**

  - **Globals**  
    - Type: Set  
    - Role: Defines GitHub repository configuration parameters as variables  
    - Config:  
      - repo.owner: GitHub username (default "john-doe")  
      - repo.name: GitHub repo (default "n8n-backup")  
      - repo.path: Repo folder path (default "credentials/")  
    - Input: Credential item from sub-workflow trigger  
    - Output: Configuration object for GitHub API nodes  
    - Failures: Misconfiguration leads to GitHub API errors

#### 1.4 Per-Credential GitHub File Check

- **Overview:** For each credential, checks if a file named `<credential_id>.json` exists in the configured GitHub path.
- **Nodes Involved:**  
  - Get file data  
  - If file too large  
  - Get File  
  - Merge Items

- **Node Details:**

  - **Get file data**  
    - Type: GitHub node (File - Get)  
    - Role: Attempts to fetch the file from GitHub using owner, repo, path + credential ID  
    - Config: Dynamic path using repo.path + credential ID + ".json"  
    - Credentials: GitHub API OAuth2  
    - Input: Global config + credential ID  
    - Output: File metadata or error if not found  
    - Failures: 404 file not found (expected), API auth errors, rate limits

  - **If file too large**  
    - Type: If node  
    - Role: Checks if file content is empty and no error (used to detect large files or failure)  
    - Config: Condition checks content empty and error not present  
    - Input: Output of Get file data  
    - Output: Branch true (file too large or missing), false (file exists with content)  
    - Failures: Logic misfires if API response format changes

  - **Get File**  
    - Type: HTTP Request  
    - Role: Downloads the raw file content using the file's download URL from GitHub API  
    - Config: URL dynamically from JSON field `download_url`  
    - Input: If file too large true branch  
    - Output: Raw file content for comparison  
    - Failures: HTTP errors, large file truncation, unauthorized access

  - **Merge Items**  
    - Type: Merge  
    - Role: Combines results of file content and local credential JSON for comparison  
    - Config: Default merge mode (likely merge by index)  
    - Input: Outputs from Get File or If file too large false branch and Globals with credential data  
    - Output: Combined JSON for next step  
    - Failures: Input mismatches

#### 1.5 File Comparison & Decision

- **Overview:** Compares local credential JSON with GitHub file content to determine if the file is "same", "different", or "new".
- **Nodes Involved:**  
  - isDiffOrNew  
  - Check Status  
  - Same file - Do nothing  
  - File is different  
  - File is new

- **Node Details:**

  - **isDiffOrNew**  
    - Type: Code node  
    - Role: Decodes GitHub file content (base64), parses JSON, orders keys to normalize, compares with local JSON  
    - Config: JavaScript code that compares JSON stringified versions for equality; sets github_status to "same", "different", or "new" accordingly  
    - Input: Merged GitHub file JSON + local credential JSON  
    - Output: Annotated JSON with status and new content stringified  
    - Failures: JSON parse errors, base64 decode failures, unexpected input schema

  - **Check Status**  
    - Type: Switch node  
    - Role: Routes flow based on github_status string: "same", "different", or "new"  
    - Config: 3 outputs mapped to each status string  
    - Input: From isDiffOrNew  
    - Output: Connects to appropriate next node branch  
    - Failures: Unexpected status values

  - **Same file - Do nothing**  
    - Type: NoOp  
    - Role: Terminates flow for identical files without changes  
    - Input: Check Status "same" branch  
    - Output: Connects to Return node  
    - Failures: None

  - **File is different**  
    - Type: NoOp  
    - Role: Placeholder for flow continuation for differing files to update GitHub  
    - Input: Check Status "different" branch  
    - Output: Connects to Edit existing file node  
    - Failures: None

  - **File is new**  
    - Type: NoOp  
    - Role: Placeholder for flow continuation for new files to create on GitHub  
    - Input: Check Status "new" branch  
    - Output: Connects to Create new file node  
    - Failures: None

#### 1.6 GitHub File Operations

- **Overview:** Updates existing files or creates new files on GitHub repository with the exported credential data.
- **Nodes Involved:**  
  - Edit existing file  
  - Create new file  
  - Return

- **Node Details:**

  - **Edit existing file**  
    - Type: GitHub node (File - Edit)  
    - Role: Updates existing credential file content with new JSON string  
    - Config: Uses repo config, dynamic file path and content from isDiffOrNew node, commit message includes workflow name and status  
    - Credentials: GitHub API OAuth2  
    - Input: Credential item + updated content  
    - Output: Success confirmation  
    - Failures: API errors, authorization, rate limiting, file lock conflicts

  - **Create new file**  
    - Type: GitHub node (File - Create)  
    - Role: Creates a new credential file in repo path with credential JSON content  
    - Config: Similar to Edit existing file node with "create" operation  
    - Credentials: GitHub API OAuth2  
    - Input: Credential item + new content  
    - Output: Success confirmation  
    - Failures: API errors, authorization, path not existing (repo will create folder), rate limits

  - **Return**  
    - Type: Set node  
    - Role: Marks the workflow or sub-workflow execution as done with a boolean `Done: true`  
    - Config: Sets Done=true  
    - Input: From Edit existing file/Create new file or Same file - Do nothing  
    - Output: Ends processing for this credential  
    - Failures: None

#### 1.7 Workflow Recursion & Return

- **Overview:** Uses recursive workflow execution to process credentials one by one, reducing memory footprint, and finalizes execution.
- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Globals  
  - Merge Items  
  - Execute Workflow

- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for recursive execution with single credential input  
    - Config: Pass-through input source  
    - Input: Credential item from batch processing  
    - Output: Triggers Globals and Merge Items nodes  
    - Failures: Recursive calls exceeding limits

  - **Globals** (reused)  
    - See above

  - **Merge Items**  
    - Type: Merge  
    - Role: Combines global config and credential input for downstream nodes  
    - Config: Default merge  
    - Input: Globals and Execute Workflow Trigger outputs  
    - Output: Feeds GitHub Get file data node  
    - Failures: Input mismatch

  - **Execute Workflow** (loop)  
    - See above in 1.2

---

### 3. Summary Table

| Node Name              | Node Type                    | Functional Role                              | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                     |
|------------------------|------------------------------|----------------------------------------------|--------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'   | Manual Trigger               | Manual start of backup workflow              | None                           | Execute Command                 |                                                                                                |
| Schedule Trigger        | Schedule Trigger             | Scheduled start every 2 hours                 | None                           | Execute Command                 |                                                                                                |
| Execute Command         | Execute Command              | Export all credentials via CLI                | On clicking 'execute', Schedule Trigger | JSON formatting                 |                                                                                                |
| JSON formatting         | Code                        | Parses CLI JSON output into array             | Execute Command                | Loop Over Items                 |                                                                                                |
| Loop Over Items         | SplitInBatches              | Iteratively process each credential           | JSON formatting                | Execute Workflow                |                                                                                                |
| Execute Workflow        | Execute Workflow            | Recursive call to process each credential     | Loop Over Items                | Loop Over Items                |                                                                                                |
| Execute Workflow Trigger| Execute Workflow Trigger    | Entry for recursive workflow with single credential | Loop Over Items (recursive)    | Globals, Merge Items           |                                                                                                |
| Globals                | Set                         | Holds GitHub repo configuration               | Execute Workflow Trigger       | Get file data                  |                                                                                                |
| Merge Items            | Merge                       | Combines globals and credential input         | Globals, Execute Workflow Trigger | Get file data                |                                                                                                |
| Get file data          | GitHub                      | Tries to get existing credential file         | Merge Items                   | If file too large              |                                                                                                |
| If file too large      | If                           | Checks if file content missing or empty       | Get file data                 | Get File (true), Merge Items (false) |                                                                                                |
| Get File               | HTTP Request                | Downloads raw file content from GitHub        | If file too large             | Merge Items                   |                                                                                                |
| Merge Items            | Merge                       | Combines file content and local credential    | Get File, If file too large (false branch) | isDiffOrNew             |                                                                                                |
| isDiffOrNew            | Code                        | Compares file content with local data          | Merge Items                   | Check Status                  |                                                                                                |
| Check Status           | Switch                      | Routes flow: same, different, or new           | isDiffOrNew                  | Same file - Do nothing, File is different, File is new |                                                                                                |
| Same file - Do nothing | NoOp                        | Ends flow when file unchanged                   | Check Status ("same")          | Return                        |                                                                                                |
| File is different      | NoOp                        | Continues flow for file update                   | Check Status ("different")     | Edit existing file            |                                                                                                |
| File is new            | NoOp                        | Continues flow for file creation                 | Check Status ("new")           | Create new file               |                                                                                                |
| Edit existing file     | GitHub                      | Updates existing file on GitHub                  | File is different             | Return                        |                                                                                                |
| Create new file        | GitHub                      | Creates new file on GitHub                        | File is new                  | Return                        |                                                                                                |
| Return                 | Set                         | Marks completion of credential processing        | Same file - Do nothing, Edit existing file, Create new file | None                          |                                                                                                |
| Sticky Note            | Sticky Note                 | Labels sub-workflow section                      | None                         | None                         | ## Subworkflow                                                                                  |
| Sticky Note1           | Sticky Note                 | Explains backup workflow setup                   | None                         | None                         | ## Backup to GitHub ... [Setup instructions and usage](https://n8n.io/creators/solomon/)        |
| Sticky Note2           | Sticky Note                 | Labels main workflow loop                         | None                         | None                         | ## Main workflow loop                                                                           |
| Sticky Note3           | Sticky Note                 | Prompts user to edit Globals node                 | None                         | None                         | ## Edit this node 👇                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: Manual start of the backup workflow

2. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval: Every 2 hours

3. **Create Execute Command node**  
   - Type: Execute Command  
   - Parameter: `npx n8n export:credentials --all --decrypted`  
   - Connect outputs of Manual Trigger and Schedule Trigger to this node

4. **Create Code node "JSON formatting"**  
   - Type: Code  
   - Paste JS code to extract JSON string from CLI output, parse it, and return an array of credential JSON objects  
   - Connect Execute Command output to this node

5. **Create SplitInBatches node "Loop Over Items"**  
   - Type: SplitInBatches  
   - Default batch size (if not set, will process one item per iteration)  
   - Connect JSON formatting output to this node

6. **Create Execute Workflow node**  
   - Type: Execute Workflow  
   - Mode: each (process each credential separately)  
   - Select current workflow for recursion  
   - Connect Loop Over Items output to this node

7. **Create Execute Workflow Trigger node**  
   - Type: Execute Workflow Trigger  
   - Input source: passthrough  
   - This node serves as entry for recursive execution  
   - Connect Execute Workflow output back to Loop Over Items node to continue batch processing

8. **Create Set node "Globals"**  
   - Type: Set  
   - Add variables:  
     - repo.owner (string): your GitHub username, e.g. "john-doe"  
     - repo.name (string): your GitHub repository name, e.g. "n8n-backup"  
     - repo.path (string): path inside repo, e.g. "credentials/"  
   - Connect Execute Workflow Trigger output to this node

9. **Create Merge node**  
   - Type: Merge  
   - Merge the output of Globals node and Execute Workflow Trigger node (credential data)  
   - Connect both to this merge node

10. **Create GitHub node "Get file data"**  
    - Type: GitHub (resource: file, operation: get)  
    - Owner: `={{ $json.repo.owner }}`  
    - Repository: `={{ $json.repo.name }}`  
    - File Path: `={{ $json.repo.path }}{{ $('Execute Workflow Trigger').item.json.id }}.json`  
    - Credentials: GitHub OAuth2 credentials  
    - Connect Merge node output to this node  
    - Enable "Continue on Fail" and "Always Output Data" to handle missing files gracefully

11. **Create If node "If file too large"**  
    - Type: If  
    - Condition: Check if `$json.content` is empty AND `$json.error` does not exist (indicates file missing or too large)  
    - Connect Get file data output to this node

12. **Create HTTP Request node "Get File"**  
    - Type: HTTP Request  
    - URL: `={{ $json.download_url }}` (from GitHub API file metadata)  
    - Connect If node "true" output (file missing/too large) to this node

13. **Create Merge node "Merge Items"**  
    - Type: Merge  
    - Merge outputs from Get File node and If node "false" branch (file exists with content)  
    - Connect both to this node

14. **Create Code node "isDiffOrNew"**  
    - Type: Code  
    - Paste JavaScript to decode base64 GitHub file content, parse JSON, order keys, compare with local credential JSON, and set github_status to "same", "different", or "new"  
    - Connect Merge Items output to this node

15. **Create Switch node "Check Status"**  
    - Type: Switch  
    - Input: `{{$json.github_status}}`  
    - Rules:  
      - Output 0: "same"  
      - Output 1: "different"  
      - Output 2: "new"  
    - Connect isDiffOrNew output to this node

16. **Create NoOp nodes**  
    - "Same file - Do nothing" (for "same")  
    - "File is different" (for "different")  
    - "File is new" (for "new")  
    - Connect Switch outputs accordingly

17. **Create GitHub node "Edit existing file"**  
    - Type: GitHub (resource: file, operation: edit)  
    - Owner, Repository, File Path: same as Get file data  
    - File Content: `={{$('isDiffOrNew').item.json["n8n_data_stringy"]}}`  
    - Commit Message: `={{$('Execute Workflow Trigger').first().json.name}} ({{$json.github_status}})`  
    - Credentials: GitHub OAuth2  
    - Connect "File is different" NoOp to this node

18. **Create GitHub node "Create new file"**  
    - Type: GitHub (resource: file, operation: create)  
    - Same configuration as Edit existing file, but operation set to create  
    - Connect "File is new" NoOp to this node

19. **Create Set node "Return"**  
    - Type: Set  
    - Assign boolean field `Done = true`  
    - Connect outputs from Same file - Do nothing, Edit existing file, and Create new file to this node

20. **Final Connections**  
    - Loop Over Items output triggers Execute Workflow (recursive)  
    - Execute Workflow Trigger starts Globals and Merge Items nodes  
    - GitHub Get file data follows Merge Items  
    - File checking, comparison, and GitHub update nodes follow accordingly  
    - Return node marks completion

21. **Credentials Setup**  
    - Create GitHub OAuth2 credential with proper access token allowing file read/write in target repo  
    - Ensure n8n instance has access to run CLI commands (for Execute Command node)

22. **Optional**  
    - Add sticky notes with instructions and documentation for maintenance and setup clarity

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is based on Jonathan's [workflow backup template](https://n8n.io/workflows/1534-back-up-your-n8n-workflows-to-github/) | Original template inspiration                                                                   |
| ⚠ Credentials are stored decrypted by default; ensure secure storage or modify CLI command to export encrypted.  | Security advisory                                                                               |
| More templates and creator info: [https://n8n.io/creators/solomon/](https://n8n.io/creators/solomon/)             | Creator's other workflow templates                                                             |
| Workflow uses recursive sub-workflow calls to minimize memory usage during batch processing of credentials.       | Performance and memory optimization note                                                       |
| GitHub API rate limits and auth errors are possible failure points; ensure GitHub credentials have correct scopes. | Operational consideration                                                                       |

---

This documentation enables comprehensive understanding, reproduction, and modification of the "Backup your credentials to GitHub" workflow. It details the flow’s logic, node configurations, potential failure modes, and step-by-step construction guidance.