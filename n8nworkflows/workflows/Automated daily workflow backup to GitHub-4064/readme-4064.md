Automated daily workflow backup to GitHub

https://n8nworkflows.xyz/workflows/automated-daily-workflow-backup-to-github-4064


# Automated daily workflow backup to GitHub

### 1. Workflow Overview

This workflow automates the daily backup of all n8n workflows to a specified GitHub repository. It is designed for users who want to maintain version-controlled, secure backups of their n8n workflows to safeguard against data loss and facilitate recovery. The workflow leverages both the n8n API to fetch workflows and the GitHub API to manage files in the repository.

The automation is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow on a daily schedule.
- **1.2 Retrieve Existing Backup Files:** Lists current backup files from GitHub to determine whether to create or update workflow backup files.
- **1.3 Fetch and Process n8n Workflows:** Retrieves all workflows from the n8n instance, converts each into JSON, and encodes the data to base64 for GitHub compatibility.
- **1.4 Commit Workflow Backup to GitHub:** Checks for each workflow’s backup file existence and either updates the existing file or uploads a new one, with commit metadata.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

**Overview:**  
Triggers the workflow automatically according to a pre-configured schedule (default: daily).

**Nodes Involved:**  
- Schedule Trigger

**Node Details:**

- **Schedule Trigger**  
  - Type: `n8n-nodes-base.scheduleTrigger`  
  - Role: Initiates the entire backup process at a specified interval.  
  - Configuration: Uses a default daily interval (can be customized).  
  - Input: None (starts the workflow).  
  - Output: Triggers "List files from repo" node.  
  - Edge Cases: Misconfigured schedule or disabled node means no backup runs.

#### 2.2 Retrieve Existing Backup Files from GitHub

**Overview:**  
Connects to GitHub to fetch a list of files currently in the backup repository, enabling the workflow to decide whether to create or update backup files.

**Nodes Involved:**  
- List files from repo  
- Aggregate  
- Sticky Note1 (documentation)

**Node Details:**

- **List files from repo**  
  - Type: `n8n-nodes-base.github`  
  - Role: Lists all files in the specified GitHub repository path.  
  - Configuration:  
    - Owner, Repository, Branch, Path configured by user credentials.  
    - Auth: OAuth2 with GitHub Personal Access Token.  
    - Operation: "list" files in the repo.  
  - Input: Trigger from Schedule Trigger.  
  - Output: Sends the list of files to the Aggregate node.  
  - Edge Cases: Authentication failure, rate limiting, incorrect repo details.

- **Aggregate**  
  - Type: `n8n-nodes-base.aggregate`  
  - Role: Consolidates the list of file names from GitHub into a single array for easier lookup.  
  - Configuration: Aggregates "name" field from incoming items.  
  - Input: GitHub file list items.  
  - Output: Single item containing all file names.  
  - Edge Cases: Empty repo returns empty aggregation, no files to compare.

- **Sticky Note1**  
  - Content: "### Retrieve previous file names from Github"  
  - Purpose: Provides contextual documentation for the block.

#### 2.3 Fetch and Process n8n Workflows

**Overview:**  
Retrieves all workflows from n8n, converts each workflow to a JSON file, and encodes it to base64, preparing the data for GitHub upload/update.

**Nodes Involved:**  
- Retrieve workflows  
- Json file  
- To base64  
- Commit date & file name  
- Sticky Note2 (documentation)

**Node Details:**

- **Retrieve workflows**  
  - Type: `n8n-nodes-base.n8n`  
  - Role: Fetches all workflows from the n8n instance via its API.  
  - Configuration: Uses configured n8n API credentials (API key and URL).  
  - Input: Triggered after file list aggregation.  
  - Output: Workflow data passed individually for processing.  
  - Edge Cases: API auth failure, empty workflow list, network errors.

- **Json file**  
  - Type: `n8n-nodes-base.convertToFile`  
  - Role: Converts workflow JSON data into a file format, enabling binary handling.  
  - Configuration: Mode set to "each" (process each workflow individually), output format JSON with pretty print enabled.  
  - Input: Individual workflow items from Retrieve workflows.  
  - Output: Binary JSON file data to "To base64" node.  
  - Edge Cases: Conversion errors if workflow data is malformed.

- **To base64**  
  - Type: `n8n-nodes-base.extractFromFile`  
  - Role: Converts binary JSON file data into base64 encoded string, required by GitHub API.  
  - Configuration: Operation set to "binaryToProperty" to store base64 string in JSON property.  
  - Input: Binary JSON files from Json file.  
  - Output: Base64-encoded string to "Commit date & file name" node.  
  - Edge Cases: Encoding failure on corrupted binary data.

- **Commit date & file name**  
  - Type: `n8n-nodes-base.set`  
  - Role: Adds metadata for commits: current timestamp and a standardized filename for each workflow backup file.  
  - Configuration:  
    - `commitDate`: formatted as `dd-MM-yyyy/H:mm` of current time.  
    - `fileName`: derived from workflow name (spaces replaced by hyphens, lowercase) and first tag (e.g., `workflow-name-tag.json`).  
  - Input: Base64 workflow data.  
  - Output: Metadata enriched items to "Check if file exists".  
  - Edge Cases: Workflows without tags may cause undefined filename parts; this requires attention.  
  - Version: Uses n8n v3.4 features (latest expression formats).

- **Sticky Note2**  
  - Content: "### Retrieve and process workflows from n8n"  
  - Purpose: Describes the function of this processing block.

#### 2.4 Commit Workflow Backup to GitHub

**Overview:**  
Determines if a backup for each workflow already exists in GitHub. If so, updates the file; if not, creates a new file with the workflow backup.

**Nodes Involved:**  
- Check if file exists  
- Update file  
- Upload file  
- Sticky Note3 (documentation)

**Node Details:**

- **Check if file exists**  
  - Type: `n8n-nodes-base.if`  
  - Role: Conditional node that checks if the computed filename exists in the list of files aggregated from GitHub.  
  - Configuration:  
    - Condition: checks if aggregated file name string contains the current workflow backup filename.  
  - Input: Metadata-enriched workflow backup filename and aggregated file list.  
  - Output:  
    - True branch: passes to "Update file" node.  
    - False branch: passes to "Upload file" node.  
  - Edge Cases: Filename matching depends on exact string matching; discrepancies in naming may cause false negatives.

- **Update file**  
  - Type: `n8n-nodes-base.github`  
  - Role: Updates an existing backup file in the GitHub repo with the new base64 encoded workflow data and a commit message.  
  - Configuration:  
    - Owner, Repository, Branch: user-configured credentials.  
    - File path: derived from the workflow name and first tag (lowercased, hyphenated).  
    - Commit message: includes timestamp.  
    - Auth: OAuth2 with GitHub token.  
  - Input: Workflow base64 content and filename from previous nodes.  
  - Output: Success or failure response from GitHub API.  
  - Edge Cases: File modified concurrently may cause conflicts; API failures or permission errors.

- **Upload file**  
  - Type: `n8n-nodes-base.github`  
  - Role: Creates a new file in the GitHub repo with the workflow backup content and commit message if it doesn't exist yet.  
  - Configuration: Same parameters as "Update file" node.  
  - Input: Workflow base64 content and filename.  
  - Output: Confirmation of file creation.  
  - Edge Cases: Permission denied, file path errors, API limits.

- **Sticky Note3**  
  - Content: "### Commit + edit/create files if needed"  
  - Purpose: Clarifies the commit phase of the process.

---

### 3. Summary Table

| Node Name            | Node Type                 | Functional Role                                  | Input Node(s)           | Output Node(s)           | Sticky Note                          |
|----------------------|---------------------------|-------------------------------------------------|-------------------------|--------------------------|-------------------------------------|
| Schedule Trigger      | n8n-nodes-base.scheduleTrigger | Initiates workflow on daily schedule            | —                       | List files from repo     |                                     |
| List files from repo  | n8n-nodes-base.github     | Lists existing backup files in GitHub repo      | Schedule Trigger        | Aggregate                |                                     |
| Aggregate            | n8n-nodes-base.aggregate  | Aggregates filenames from GitHub list            | List files from repo     | Retrieve workflows       |                                     |
| Retrieve workflows    | n8n-nodes-base.n8n        | Retrieves all workflows from n8n instance        | Aggregate                | Json file                |                                     |
| Json file             | n8n-nodes-base.convertToFile | Converts workflow JSON data into JSON files      | Retrieve workflows       | To base64                |                                     |
| To base64             | n8n-nodes-base.extractFromFile | Converts JSON file binary data to base64 string | Json file                | Commit date & file name  |                                     |
| Commit date & file name | n8n-nodes-base.set       | Adds commit timestamp and standardized filename  | To base64                | Check if file exists     |                                     |
| Check if file exists  | n8n-nodes-base.if         | Condition: Checks if backup file exists in repo  | Commit date & file name  | Update file (true), Upload file (false) |                                     |
| Update file           | n8n-nodes-base.github     | Updates existing workflow backup file on GitHub  | Check if file exists     | —                        |                                     |
| Upload file           | n8n-nodes-base.github     | Uploads new workflow backup file to GitHub        | Check if file exists     | —                        |                                     |
| Sticky Note1          | n8n-nodes-base.stickyNote | Documentation block for GitHub file listing       | —                       | —                        | "### Retrieve previous file names from Github" |
| Sticky Note2          | n8n-nodes-base.stickyNote | Documentation block for workflow retrieval        | —                       | —                        | "### Retrieve and process workflows from n8n" |
| Sticky Note3          | n8n-nodes-base.stickyNote | Documentation block for commit/update phase       | —                       | —                        | "### Commit + edit/create files if needed" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Type: `scheduleTrigger`  
   - Configure interval for daily trigger (default).  
   - Connect its main output to the next node.

2. **Create the GitHub "List files from repo" node**  
   - Type: `github`  
   - Operation: `list` files in repo.  
   - Set parameters:  
     - Owner: Your GitHub username or organization.  
     - Repository: Target repository for backups.  
     - Branch: Branch where backups are stored (e.g., `main`).  
     - Path: Optional subfolder path or leave empty.  
   - Authentication: Configure OAuth2 with your GitHub Personal Access Token credential.  
   - Connect Schedule Trigger to this node.

3. **Create the Aggregate node**  
   - Type: `aggregate`  
   - Configure to aggregate the "name" field from incoming GitHub files into a single list.  
   - Connect "List files from repo" main output to Aggregate input.

4. **Create the Retrieve workflows node**  
   - Type: `n8n`  
   - Operation: Retrieve all workflows from your n8n instance.  
   - Configure n8n API credentials (API URL and API key).  
   - Connect Aggregate output to this node.

5. **Create the Json file node**  
   - Type: `convertToFile`  
   - Operation: `toJson`  
   - Mode: `each` (process each workflow item separately).  
   - Enable pretty JSON formatting.  
   - Connect Retrieve workflows output to this node.

6. **Create the To base64 node**  
   - Type: `extractFromFile`  
   - Operation: `binaryToProperty` to convert binary JSON file to a base64 string stored in item property.  
   - Connect Json file output to this node.

7. **Create the Commit date & file name node**  
   - Type: `set`  
   - Add two fields with expressions:  
     - `commitDate`: Use current date/time formatted as `dd-MM-yyyy/H:mm` (e.g., `={{ $now.format('dd-MM-yyyy/H:mm') }}`).  
     - `fileName`: Generate filename by replacing spaces with hyphens and lowercasing workflow name, then append first workflow tag, e.g., `={{ $('Retrieve workflows').item.json.name.replace(/\s+/g, '-').toLowerCase() }}-{{ $('Retrieve workflows').item.json.tags[0].name }}.json`.  
   - Connect To base64 output to this node.

8. **Create the Check if file exists node**  
   - Type: `if`  
   - Condition: Check if aggregated file names string (from Aggregate) contains the generated `fileName` from Commit date & file name node.  
   - Expression example:  
     - Value1: `={{ $('Aggregate').item.json.name }}`  
     - Value2: `={{ $('Commit date & file name').item.json.fileName }}`  
     - Operation: `contains`  
   - Connect Commit date & file name output to this node.

9. **Create the Update file node**  
   - Type: `github`  
   - Operation: `edit` (update existing file).  
   - Set parameters:  
     - Owner, Repository, Branch, Path as before.  
     - File path: Use filename expression as in Commit date & file name node.  
     - File content: Base64 string from "To base64" node property.  
     - Commit message: Include timestamp, e.g., `backup-{{ $node['Commit date & file name'].json.commitDate }}`.  
   - Authentication: Use same GitHub credential as "List files from repo".  
   - Connect the `true` output of Check if file exists node to this node.

10. **Create the Upload file node**  
    - Type: `github`  
    - Operation: `create` (upload new file).  
    - Set parameters identically to Update file node.  
    - Connect the `false` output of Check if file exists node to this node.

11. **Finalize connections:**  
    - Connect Schedule Trigger → List files from repo → Aggregate → Retrieve workflows → Json file → To base64 → Commit date & file name → Check if file exists → Update file (true) / Upload file (false).

12. **Credentials Setup:**  
    - Create GitHub credentials with OAuth2/Personal Access Token having repo write access.  
    - Create n8n API credentials with your n8n instance URL and API key.

13. **Adjust optional parameters:**  
    - Schedule timing for Schedule Trigger.  
    - GitHub repo owner, repository, branch, and path in all GitHub nodes.  
    - Naming conventions in Commit date & file name node expressions.

14. **Activate the workflow** and verify operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| The backup files are named using the workflow name and the first tag, separated by a hyphen and suffixed with `.json`.                 | Filename convention explained; customize in "Commit date & file name" node.                                                    |
| Ensure your GitHub token has `repo` scope or equivalent fine-grained permissions for API access to create and update files.             | Authentication requirements for GitHub credentials.                                                                            |
| Workflow designed and tested in n8n version 1.92.2.                                                                                    | Template version compatibility.                                                                                                |
| Consider adding error handling nodes or notifications for backup failures to enhance reliability.                                       | Suggested customization for robustness.                                                                                        |
| GitHub API file operations require file content encoded as base64; this is why the workflow includes JSON conversion and base64 encoding. | Technical requirement for GitHub API interaction.                                                                               |
| For more info on n8n API keys, visit: https://docs.n8n.io/integrations/builtin/core-nodes/n8n/                                           | Official n8n API documentation.                                                                                                |
| GitHub Personal Access Token creation guide: https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token | Official GitHub docs for token generation.                                                                                     |

---

This completes the detailed documentation and reference of the "Automated daily workflow backup to GitHub" n8n workflow, enabling thorough understanding, reproduction, and customization.