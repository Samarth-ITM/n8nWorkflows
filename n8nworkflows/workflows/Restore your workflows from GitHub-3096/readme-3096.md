Restore your workflows from GitHub

https://n8nworkflows.xyz/workflows/restore-your-workflows-from-github-3096


# Restore your workflows from GitHub

### 1. Workflow Overview

This workflow automates the restoration of all n8n workflows previously backed up on GitHub. It is designed to fetch workflow JSON files stored in a specified GitHub repository folder and import them into the current n8n instance via the n8n API. This enables users to quickly recover or migrate workflows from GitHub backups.

The workflow is logically divided into these blocks:

- **1.1 Trigger and Initialization:** Manual trigger to start the workflow and setting global variables for GitHub repository details.
- **1.2 Fetch Workflow Files from GitHub:** Retrieve the list of workflow files from the specified GitHub repo folder.
- **1.3 Process Each Workflow File:** Iterate over each file path, fetch its content from GitHub, and convert it from JSON.
- **1.4 Restore Workflows into n8n:** Import each workflow JSON into the n8n instance using the n8n API node.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block initiates the workflow manually and sets global parameters needed for subsequent GitHub API calls, including repository owner, name, and path.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Globals (Set node)  
  - Sticky Note (Instructional note)  
  - Sticky Note3 (Instructional note)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually on demand.  
    - Configuration: No parameters; simply triggers execution.  
    - Input: None  
    - Output: Connects to Globals node.  
    - Edge Cases: None; manual trigger is straightforward.

  - **Globals**  
    - Type: Set  
    - Role: Defines global variables for GitHub repository details.  
    - Configuration:  
      - `repo.owner` set to "BeyondspaceStudio" (default, user should update)  
      - `repo.name` set to "n8n-backup"  
      - `repo.path` set to "workflows"  
    - Input: From manual trigger node.  
    - Output: Connects to "Get all files in given path" HTTP Request node.  
    - Edge Cases: User must update values correctly; otherwise, GitHub API calls will fail or return empty results.

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides setup instructions and usage notes for users.  
    - Content: Explains how to configure the `Globals` node with GitHub repo details.  
    - Input/Output: None (informational only).

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Highlights the `Globals` node for editing.  
    - Content: "Edit this node 👇"  
    - Input/Output: None (informational only).

---

#### 2.2 Fetch Workflow Files from GitHub

- **Overview:**  
  This block fetches the list of all workflow files stored in the configured GitHub repository path.

- **Nodes Involved:**  
  - Get all files in given path (HTTP Request)  
  - Split the result (Split Out)

- **Node Details:**

  - **Get all files in given path**  
    - Type: HTTP Request  
    - Role: Calls GitHub API to list contents of the specified repo folder.  
    - Configuration:  
      - URL dynamically constructed using expressions referencing `repo.owner`, `repo.name`, and `repo.path` from the Globals node.  
      - Authentication: Uses GitHub API credentials (OAuth or token).  
    - Input: Receives repo details from Globals node.  
    - Output: JSON array of files/folders in the repo path.  
    - Edge Cases:  
      - GitHub API rate limits or authentication errors.  
      - Empty folder returns empty array.  
      - Incorrect path or repo details cause 404 or errors.

  - **Split the result**  
    - Type: Split Out  
    - Role: Splits the array of file objects into individual items for sequential processing.  
    - Configuration: Splits on the field `path` from the GitHub API response.  
    - Input: Receives array of files from HTTP Request node.  
    - Output: Sends each file path as a separate item downstream.  
    - Edge Cases: Empty input array results in no further processing.

---

#### 2.3 Process Each Workflow File

- **Overview:**  
  For each file path obtained, this block fetches the file content from GitHub and converts it from JSON format to a JavaScript object.

- **Nodes Involved:**  
  - Get file content from GitHub (GitHub node)  
  - Convert files to JSON (Extract From File)

- **Node Details:**

  - **Get file content from GitHub**  
    - Type: GitHub  
    - Role: Retrieves the raw content of each workflow file from GitHub.  
    - Configuration:  
      - Owner: Hardcoded to "BeyondspaceStudio" (should ideally be dynamic from Globals)  
      - Repository: Hardcoded to "n8n-backup" (should ideally be dynamic)  
      - File Path: Expression referencing the current item's `path` field from the Split node.  
      - Operation: Get file content  
      - Authentication: Uses GitHub API credentials.  
    - Input: Receives file path from Split node.  
    - Output: Raw file content in base64 or JSON format.  
    - Edge Cases:  
      - File not found or access denied errors.  
      - Large files may cause timeouts.  
      - Hardcoded owner/repo may cause mismatch if Globals are changed without updating this node.

  - **Convert files to JSON**  
    - Type: Extract From File  
    - Role: Parses the raw file content into JSON objects representing workflows.  
    - Configuration: Operation set to "fromJson".  
    - Input: Receives raw file content from GitHub node.  
    - Output: Parsed JSON object representing the workflow.  
    - Edge Cases:  
      - Malformed JSON files cause parse errors.  
      - Empty or invalid content leads to failures.

---

#### 2.4 Restore Workflows into n8n

- **Overview:**  
  This block imports each parsed workflow JSON into the current n8n instance using the n8n API node.

- **Nodes Involved:**  
  - Restore n8n Workflows (n8n node)

- **Node Details:**

  - **Restore n8n Workflows**  
    - Type: n8n (n8n API node)  
    - Role: Creates a new workflow in n8n by sending the workflow JSON to the n8n API.  
    - Configuration:  
      - Operation: Create workflow  
      - Workflow Object: Expression serializing the JSON data field from the previous node (`JSON.stringify($json.data)`).  
      - Credentials: Uses n8n API credentials (OAuth2 or API key).  
    - Input: Receives parsed workflow JSON from previous node.  
    - Output: API response confirming workflow creation.  
    - Edge Cases:  
      - API authentication errors.  
      - Workflow JSON incompatible with current n8n version.  
      - Duplicate workflows or naming conflicts.  
      - API rate limits or timeouts.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                         | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                              |
|---------------------------|---------------------|---------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Initiates workflow execution manually | None                        | Globals                        |                                                                                                         |
| Globals                   | Set                 | Sets GitHub repo parameters globally | When clicking ‘Test workflow’ | Get all files in given path    | ## Edit this node 👇                                                                                      |
| Sticky Note               | Sticky Note         | Provides setup instructions           | None                        | None                          | ## Restore from GitHub \nThis workflow will restore all instance workflows from GitHub backups.\n\n### Setup\nOpen `Globals` node and update the values below 👇\n\n- **repo.owner:** your Github username\n- **repo.name:** the name of your repository\n- **repo.path:** the folder to use within the repository.\n\nIf your username was `john-doe` and your repository was called `n8n-backups` and you wanted the workflows to go into a `workflows` folder you would set:\n\n- repo.owner - john-doe\n- repo.name - n8n-backups\n- repo.path - workflows/ |
| Sticky Note3              | Sticky Note         | Highlights Globals node for editing   | None                        | None                          | ## Edit this node 👇                                                                                      |
| Get all files in given path | HTTP Request        | Fetches list of workflow files from GitHub repo path | Globals                      | Split the result              |                                                                                                         |
| Split the result          | Split Out           | Splits file list into individual paths | Get all files in given path  | Get file content from GitHub   |                                                                                                         |
| Get file content from GitHub | GitHub              | Retrieves content of each workflow file | Split the result             | Convert files to JSON          |                                                                                                         |
| Convert files to JSON     | Extract From File   | Parses raw file content into JSON     | Get file content from GitHub | Restore n8n Workflows          |                                                                                                         |
| Restore n8n Workflows     | n8n (API)           | Imports workflows into n8n instance   | Convert files to JSON        | None                          |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Add a Set node**  
   - Name: `Globals`  
   - Purpose: Define global variables for GitHub repository details.  
   - Parameters to set:  
     - `repo.owner` (string): Your GitHub username (e.g., "BeyondspaceStudio")  
     - `repo.name` (string): Your GitHub repository name (e.g., "n8n-backup")  
     - `repo.path` (string): Folder path in repo where workflows are stored (e.g., "workflows")  
   - Connect output of Manual Trigger to this node.

3. **Add an HTTP Request node**  
   - Name: `Get all files in given path`  
   - Purpose: List all files in the GitHub repo path.  
   - HTTP Method: GET  
   - URL: Use expression to build URL dynamically:  
     ```
     https://api.github.com/repos/{{ $json.repo.owner }}/{{ $json.repo.name }}/contents/{{ $json.repo.path }}
     ```  
   - Authentication: Select GitHub API credentials (OAuth2 or Personal Access Token).  
   - Connect output of `Globals` node to this node.

4. **Add a Split Out node**  
   - Name: `Split the result`  
   - Purpose: Split the array of files into individual items.  
   - Field to split out: `path`  
   - Connect output of HTTP Request node to this node.

5. **Add a GitHub node**  
   - Name: `Get file content from GitHub`  
   - Purpose: Retrieve the content of each workflow file.  
   - Resource: File  
   - Operation: Get  
   - Owner: Set to the same value as `repo.owner` (ideally use expression to make dynamic)  
   - Repository: Set to the same value as `repo.name` (ideally dynamic)  
   - File Path: Use expression referencing current item path:  
     ```
     {{$json.path}}
     ```  
   - Authentication: Use GitHub API credentials.  
   - Connect output of Split Out node to this node.

6. **Add an Extract From File node**  
   - Name: `Convert files to JSON`  
   - Purpose: Parse the raw file content into JSON workflow objects.  
   - Operation: From JSON  
   - Connect output of GitHub node to this node.

7. **Add an n8n node (n8n API node)**  
   - Name: `Restore n8n Workflows`  
   - Purpose: Import workflows into the n8n instance.  
   - Operation: Create workflow  
   - Workflow Object: Use expression to stringify the parsed JSON data:  
     ```
     {{ JSON.stringify($json.data) }}
     ```  
   - Credentials: Configure with n8n API credentials (OAuth2 or API key).  
   - Connect output of Extract From File node to this node.

8. **Add Sticky Notes (optional)**  
   - Add notes near the `Globals` node to instruct users to update GitHub repo details.  
   - Add a general note explaining the workflow purpose and setup.

9. **Save and activate the workflow**  
   - Test by manually triggering and verifying workflows are restored from GitHub.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow complements the [Backup Your Workflows to GitHub](https://n8n.io/workflows/2652-backup-your-workflows-to-github/) template by restoring workflows from GitHub backups. | Related template for backup: https://n8n.io/workflows/2652-backup-your-workflows-to-github/      |
| Update the `Globals` node with your GitHub username, repository name, and folder path before running the workflow. | Setup instructions inside Sticky Note node                                                      |
| Required credentials: GitHub API (OAuth2 or Personal Access Token) and n8n API credentials (OAuth2 or API key). | Credential setup is mandatory for GitHub and n8n API nodes                                      |
| For best results, ensure workflow JSON files in GitHub are valid and compatible with your current n8n version. | Prevents import errors during restoration                                                        |
| Check out other templates by the author: [My n8n Templates](https://n8n.io/creators/bangank36/)                | Additional resources and templates                                                               |

---

This documentation provides a complete understanding of the "Restore your workflows from GitHub" workflow, enabling users and AI agents to reproduce, modify, and troubleshoot it effectively.