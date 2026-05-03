Automatic Workflow & Credentials Backup to GitHub with Change Detection

https://n8nworkflows.xyz/workflows/automatic-workflow---credentials-backup-to-github-with-change-detection-4609


# Automatic Workflow & Credentials Backup to GitHub with Change Detection

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows and credentials by exporting them periodically, detecting any changes via hashing, and committing updated backups to a GitHub repository. It is designed to run on a scheduled basis (hourly) and only push changes when differences are detected to optimize storage and version control usage.

The workflow is logically divided into two main parallel blocks:

- **1.1 Workflow Backup Block**: Handles the export, hashing, change detection, reading, and GitHub backup of n8n workflows.
- **1.2 Credential Backup Block**: Performs a similar process for n8n credentials, exporting, hashing, detecting changes, reading, and backing up to GitHub.

Each block involves the following logical steps:  
- Setting file paths for export  
- Executing export commands (workflows or credentials)  
- Hashing exported data for change detection  
- Comparing new hash with the previous version's hash  
- If changed, reading the data and pushing it to GitHub with a timestamped commit message

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Backup Block

- **Overview:**  
  This block exports all n8n workflows to a JSON file, computes a hash to detect changes since the last backup, and if changes exist, uploads the new backup file to a GitHub repository with a timestamped commit.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Workflow Path  
  - Get Old Workflow Hash (sub-workflow execution)  
  - Execute Workflow Backup  
  - Get New Workflow Hash (sub-workflow execution)  
  - If Workflow Updated (conditional check)  
  - Read Workflow Data  
  - Extract Workflow Data  
  - GitHub Workflow Backup

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every hour to automate backups.  
    - Config: Runs every hour on the hour (interval = hours).  
    - Inputs: None.  
    - Outputs: Triggers two Set nodes in parallel for workflow and credential paths.  
    - Failure Modes: None typical; ensure n8n instance uptime.  

  - **Set Workflow Path**  
    - Type: Set  
    - Role: Defines the file path `/data/shared/WorkFlowBackup.json` where workflows will be exported.  
    - Config: Single assignment `filePath` to fixed string.  
    - Inputs: From Schedule Trigger.  
    - Outputs: To "Get Old Workflow Hash".  
    - Failure Modes: None expected; path must be writable by n8n.

  - **Get Old Workflow Hash**  
    - Type: Execute Workflow  
    - Role: Calls a sub-workflow (ID: `U7Jh9njKLEjGLVIl` named "n8n Backup Changes") to retrieve the previous hash of the workflow backup JSON file.  
    - Config: Passes `filePath` from Set Workflow Path node as input parameter.  
    - Inputs: From Set Workflow Path.  
    - Outputs: To "Execute Workflow Backup".  
    - Failure Modes: Sub-workflow errors, file not found, or misconfigured sub-workflow inputs.

  - **Execute Workflow Backup**  
    - Type: Execute Command  
    - Role: Runs shell command to export all workflows to the specified JSON file.  
    - Config: Command is `n8n export:workflow --all --output=<filePath>`.  
    - Inputs: From Get Old Workflow Hash.  
    - Outputs: To "Get New Workflow Hash".  
    - Failure Modes: Command execution failure, permission issues, disk space errors.

  - **Get New Workflow Hash**  
    - Type: Execute Workflow  
    - Role: Calls the same sub-workflow as "Get Old Workflow Hash" to compute the new hash of the freshly exported workflow JSON file.  
    - Config: Uses the same `filePath` input from Set Workflow Path.  
    - Inputs: From Execute Workflow Backup.  
    - Outputs: To "If Workflow Updated".  
    - Failure Modes: Same as Get Old Workflow Hash.

  - **If Workflow Updated**  
    - Type: If  
    - Role: Compares new workflow hash vs old workflow hash to detect changes.  
    - Config: Condition triggers if hashes are not equal (`notEquals`).  
    - Inputs: From Get New Workflow Hash.  
    - Outputs: If true, routes to "Read Workflow Data", else ends branch.  
    - Failure Modes: Expression evaluation errors if hash values missing.

  - **Read Workflow Data**  
    - Type: Read File  
    - Role: Reads the exported workflow JSON file content if changes were detected.  
    - Config: Reads file from `filePath` in Set Workflow Path.  
    - Inputs: From If Workflow Updated (true branch).  
    - Outputs: To "Extract Workflow Data".  
    - Failure Modes: File not found, read permissions.

  - **Extract Workflow Data**  
    - Type: Extract From File  
    - Role: Extracts text content from the workflow backup file (assumed JSON text).  
    - Config: Operation is "text" extraction.  
    - Inputs: From Read Workflow Data.  
    - Outputs: To "GitHub Workflow Backup".  
    - Failure Modes: File content unreadable or corrupt.

  - **GitHub Workflow Backup**  
    - Type: GitHub (File resource)  
    - Role: Uploads the workflow backup file content to a GitHub repository under a timestamped filename and commit message.  
    - Config:  
      - Owner: `nickmehr-hamed`  
      - Repository: `https://github.com/nickmehr-hamed/n8n`  
      - File path: `backups/WorkFlow Backup <ISO timestamp>.json`  
      - Commit message: `WorkFlow Backup <ISO timestamp>`  
      - File content: Extracted text from previous node  
      - Credentials: GitHub API with OAuth token  
    - Inputs: From Extract Workflow Data.  
    - Outputs: None (end).  
    - Failure Modes: GitHub API errors, auth failures, rate limits.

---

#### 2.2 Credential Backup Block

- **Overview:**  
  This block performs an analogous process for n8n credentials, exporting all credentials, hashing to detect changes, and pushing updated backups to GitHub.

- **Nodes Involved:**  
  - Set Credential Path  
  - Get Old Credential Hash (sub-workflow execution)  
  - Execute Credential Backup  
  - Get New Credential Hash (sub-workflow execution)  
  - If Credential Updated (conditional check)  
  - Read Credential Data  
  - Extract Credential Data  
  - GitHub Credential Backup

- **Node Details:**

  - **Set Credential Path**  
    - Type: Set  
    - Role: Defines the file path `/data/shared/CredentialBackup.json` for credential backup export.  
    - Config: Single assignment `filePath` with fixed string.  
    - Inputs: From Schedule Trigger.  
    - Outputs: To "Get Old Credential Hash".  
    - Failure Modes: None expected.

  - **Get Old Credential Hash**  
    - Type: Execute Workflow  
    - Role: Calls the same sub-workflow `U7Jh9njKLEjGLVIl` to retrieve previous credential backup hash.  
    - Config: Passes `filePath` from Set Credential Path.  
    - Inputs: From Set Credential Path.  
    - Outputs: To "Execute Credential Backup".  
    - Failure Modes: Same as workflow hash nodes.

  - **Execute Credential Backup**  
    - Type: Execute Command  
    - Role: Runs shell command to export all credentials to the specified JSON file.  
    - Config: Command: `n8n export:credentials --all --output=<filePath>`  
    - Inputs: From Get Old Credential Hash.  
    - Outputs: To "Get New Credential Hash".  
    - Failure Modes: Command failure, permission errors.

  - **Get New Credential Hash**  
    - Type: Execute Workflow  
    - Role: Calls the sub-workflow to compute new hash of exported credential file.  
    - Config: Uses `filePath` from Set Credential Path.  
    - Inputs: From Execute Credential Backup.  
    - Outputs: To "If Credential Updated".  
    - Failure Modes: Same as above hash nodes.

  - **If Credential Updated**  
    - Type: If  
    - Role: Checks if the new hash differs from old hash to detect changes.  
    - Config: Condition triggers if hashes are not equal (`notEquals`).  
    - Inputs: From Get New Credential Hash.  
    - Outputs: If true, routes to "Read Credential Data", else ends branch.  
    - Failure Modes: Expression errors if hash missing.

  - **Read Credential Data**  
    - Type: Read File  
    - Role: Reads the credential backup JSON file content if changes were detected.  
    - Config: Reads file from `filePath` in Set Credential Path node.  
    - Inputs: From If Credential Updated (true branch).  
    - Outputs: To "Extract Credential Data".  
    - Failure Modes: File read errors, permissions.

  - **Extract Credential Data**  
    - Type: Extract From File  
    - Role: Extracts text content from the credential backup file.  
    - Config: Operation is "text" extraction.  
    - Inputs: From Read Credential Data.  
    - Outputs: To "GitHub Credential Backup".  
    - Failure Modes: Corrupted file content.

  - **GitHub Credential Backup**  
    - Type: GitHub (File resource)  
    - Role: Uploads credential backup content to GitHub repository with timestamped filename and commit message.  
    - Config:  
      - Owner: `nickmehr-hamed`  
      - Repository: `https://github.com/nickmehr-hamed/n8n`  
      - File path: `backups/ Credential Backup <ISO timestamp>.json` (note space after slash)  
      - Commit message: `Credential Backup <ISO timestamp>` (note space in commit message)  
      - File content: Extracted text from previous node  
      - Credentials: GitHub API OAuth token  
    - Inputs: From Extract Credential Data.  
    - Outputs: None (end).  
    - Failure Modes: GitHub API errors, authentication, rate limits.

---

### 3. Summary Table

| Node Name                    | Node Type               | Functional Role                                | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                 |
|------------------------------|-------------------------|------------------------------------------------|------------------------------|--------------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger        | Initiates workflow hourly                      | None                         | Set Workflow Path, Set Credential Path |                                                                                             |
| Set Workflow Path             | Set                     | Defines workflow export file path              | Schedule Trigger             | Get Old Workflow Hash           |                                                                                             |
| Get Old Workflow Hash         | Execute Workflow        | Gets previous workflow backup hash             | Set Workflow Path            | Execute Workflow Backup         | Calls sub-workflow "n8n Backup Changes"                                                    |
| Execute Workflow Backup       | Execute Command         | Exports all workflows to JSON                   | Get Old Workflow Hash        | Get New Workflow Hash           |                                                                                             |
| Get New Workflow Hash         | Execute Workflow        | Gets new workflow backup hash                   | Execute Workflow Backup      | If Workflow Updated             | Calls sub-workflow "n8n Backup Changes"                                                    |
| If Workflow Updated           | If                      | Checks if workflow backup has changed           | Get New Workflow Hash        | Read Workflow Data (if true)    |                                                                                             |
| Read Workflow Data            | Read File               | Reads workflow backup JSON file                 | If Workflow Updated          | Extract Workflow Data           |                                                                                             |
| Extract Workflow Data         | Extract From File       | Extracts text content from workflow backup     | Read Workflow Data           | GitHub Workflow Backup          |                                                                                             |
| GitHub Workflow Backup        | GitHub                  | Uploads workflow backup to GitHub repository   | Extract Workflow Data        | None                           |                                                                                             |
| Set Credential Path           | Set                     | Defines credential export file path             | Schedule Trigger             | Get Old Credential Hash         |                                                                                             |
| Get Old Credential Hash       | Execute Workflow        | Gets previous credential backup hash            | Set Credential Path          | Execute Credential Backup       | Calls sub-workflow "n8n Backup Changes"                                                    |
| Execute Credential Backup     | Execute Command         | Exports all credentials to JSON                  | Get Old Credential Hash      | Get New Credential Hash         |                                                                                             |
| Get New Credential Hash       | Execute Workflow        | Gets new credential backup hash                  | Execute Credential Backup    | If Credential Updated           | Calls sub-workflow "n8n Backup Changes"                                                    |
| If Credential Updated         | If                      | Checks if credential backup has changed          | Get New Credential Hash      | Read Credential Data (if true)  |                                                                                             |
| Read Credential Data          | Read File               | Reads credential backup JSON file                 | If Credential Updated        | Extract Credential Data         |                                                                                             |
| Extract Credential Data       | Extract From File       | Extracts text content from credential backup    | Read Credential Data         | GitHub Credential Backup        |                                                                                             |
| GitHub Credential Backup      | GitHub                  | Uploads credential backup to GitHub repository  | Extract Credential Data      | None                           |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Set to trigger every 1 hour.  
   - Connect output to two parallel Set nodes (Workflow Path and Credential Path).

2. **Create Set Workflow Path Node**  
   - Type: Set  
   - Add single field `filePath` with value `/data/shared/WorkFlowBackup.json`.  
   - Connect input from Schedule Trigger.  
   - Connect output to "Get Old Workflow Hash".

3. **Create Set Credential Path Node**  
   - Type: Set  
   - Add single field `filePath` with value `/data/shared/CredentialBackup.json`.  
   - Connect input from Schedule Trigger.  
   - Connect output to "Get Old Credential Hash".

4. **Create Get Old Workflow Hash Node**  
   - Type: Execute Workflow  
   - Workflow to execute: ID or name of sub-workflow "n8n Backup Changes" (ID: `U7Jh9njKLEjGLVIl`).  
   - Pass parameter `filePath` from Set Workflow Path node.  
   - Connect input from Set Workflow Path.  
   - Connect output to "Execute Workflow Backup".

5. **Create Execute Workflow Backup Node**  
   - Type: Execute Command  
   - Command: `n8n export:workflow --all --output={{ $('Set Workflow Path').item.json.filePath }}`  
   - Connect input from Get Old Workflow Hash.  
   - Connect output to "Get New Workflow Hash".

6. **Create Get New Workflow Hash Node**  
   - Type: Execute Workflow  
   - Same sub-workflow as old hash node.  
   - Pass parameter `filePath` from Set Workflow Path node.  
   - Connect input from Execute Workflow Backup.  
   - Connect output to "If Workflow Updated".

7. **Create If Workflow Updated Node**  
   - Type: If  
   - Condition: If hash from "Get New Workflow Hash" is not equal to hash from "Get Old Workflow Hash".  
   - Connect input from Get New Workflow Hash.  
   - True output to "Read Workflow Data".  
   - False output ends branch.

8. **Create Read Workflow Data Node**  
   - Type: Read File  
   - File to read from: `filePath` from Set Workflow Path node.  
   - Connect input from If Workflow Updated (true branch).  
   - Connect output to "Extract Workflow Data".

9. **Create Extract Workflow Data Node**  
   - Type: Extract From File  
   - Operation: Text extraction.  
   - Connect input from Read Workflow Data.  
   - Connect output to "GitHub Workflow Backup".

10. **Create GitHub Workflow Backup Node**  
    - Type: GitHub  
    - Owner: `nickmehr-hamed`  
    - Repository: `n8n` (full URL: https://github.com/nickmehr-hamed/n8n)  
    - Resource: File  
    - File path: `backups/WorkFlow Backup <current ISO datetime>.json` (Use expression to format datetime, replacing 'T' with space and slicing to minutes)  
    - Commit message: `WorkFlow Backup <current ISO datetime>` (same formatting as file path)  
    - File content: Use extracted text from previous node  
    - Credentials: GitHub API OAuth2 token (configure credential in n8n)  
    - Connect input from Extract Workflow Data.

11. **Create Get Old Credential Hash Node**  
    - Type: Execute Workflow  
    - Same sub-workflow as workflow hash nodes.  
    - Pass parameter `filePath` from Set Credential Path node.  
    - Connect input from Set Credential Path.  
    - Connect output to "Execute Credential Backup".

12. **Create Execute Credential Backup Node**  
    - Type: Execute Command  
    - Command: `n8n export:credentials --all --output={{ $('Set Credential Path').item.json.filePath }}`  
    - Connect input from Get Old Credential Hash.  
    - Connect output to "Get New Credential Hash".

13. **Create Get New Credential Hash Node**  
    - Type: Execute Workflow  
    - Same sub-workflow as above.  
    - Pass parameter `filePath` from Set Credential Path node.  
    - Connect input from Execute Credential Backup.  
    - Connect output to "If Credential Updated".

14. **Create If Credential Updated Node**  
    - Type: If  
    - Condition: Hashes not equal between new and old credential hashes.  
    - Connect input from Get New Credential Hash.  
    - True output to "Read Credential Data".  
    - False output ends branch.

15. **Create Read Credential Data Node**  
    - Type: Read File  
    - File path: `filePath` from Set Credential Path node.  
    - Connect input from If Credential Updated (true).  
    - Connect output to "Extract Credential Data".

16. **Create Extract Credential Data Node**  
    - Type: Extract From File  
    - Operation: Text extraction.  
    - Connect input from Read Credential Data.  
    - Connect output to "GitHub Credential Backup".

17. **Create GitHub Credential Backup Node**  
    - Type: GitHub  
    - Owner: `nickmehr-hamed`  
    - Repository: `n8n` (URL: https://github.com/nickmehr-hamed/n8n)  
    - Resource: File  
    - File path: `backups/ Credential Backup <current ISO datetime>.json` (note space after slash)  
    - Commit message: `Credential Backup <current ISO datetime>` (with space)  
    - File content: Extracted text from previous node  
    - Credentials: GitHub API OAuth2 token  
    - Connect input from Extract Credential Data.

18. **Ensure Sub-Workflow "n8n Backup Changes" Exists**  
    - This sub-workflow should accept a `filePath` input and output a hash of the file contents (likely SHA256).  
    - Configure it to read the file, extract text, hash it, and return the hash under `item.json.hash`.  
    - This sub-workflow is called four times for old and new hashes for workflows and credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow is designed to minimize GitHub commits by only uploading backups when changes are detected via hash comparison. | Core design principle to reduce storage and version noise.                                          |
| GitHub OAuth credentials must be configured in n8n with sufficient permissions to create/update files in the repository. | See n8n docs on GitHub OAuth2 credential setup.                                                     |
| The sub-workflow "n8n Backup Changes" is essential for hashing; ensure it is reliable and accessible by ID.      | This sub-workflow performs file read and SHA256 hashing.                                            |
| The export commands (`n8n export:workflow` and `n8n export:credentials`) require the n8n CLI environment to be accessible for execution. | Ensure n8n instance has CLI access and proper permissions.                                          |
| Timestamp formatting in the GitHub backup nodes uses expressions to replace 'T' with a space and limit to minutes. | Example: `.toISOString().replace('T', ' ').slice(0, 16)`                                            |
| The workflow handles both workflows and credentials backups in parallel branches triggered by the same schedule. | Enables consistent backup intervals for both resources.                                            |

---

**Disclaimer:**  
The content provided is extracted from an automated n8n workflow. It fully complies with all applicable content policies and contains no illegal or protected data. All data handled is public and lawful.