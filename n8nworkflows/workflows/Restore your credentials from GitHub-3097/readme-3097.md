Restore your credentials from GitHub

https://n8nworkflows.xyz/workflows/restore-your-credentials-from-github-3097


# Restore your credentials from GitHub

### 1. Workflow Overview

This workflow automates the restoration of all n8n instance credentials from backups stored in a GitHub repository. It is designed for users who have previously backed up their credentials to GitHub and want to seamlessly import them back into their n8n instance, facilitating migration, recovery, or replication of credentials.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger and Configuration Setup:** Manual trigger to start the workflow and setting global variables for GitHub repository details.
- **1.2 Fetching Credential Files from GitHub:** Retrieving the list of credential files stored in the specified GitHub repository path.
- **1.3 Processing Each Credential File:** Iterating over each file, fetching its content, converting it to JSON, and filtering out files that should be skipped.
- **1.4 Restoring Credentials into n8n:** Importing valid credential JSON data into the n8n instance via the n8n API node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Configuration Setup

- **Overview:**  
  This block initiates the workflow manually and sets the global variables that define the GitHub repository owner, repository name, and path where credentials are stored.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Globals  
  - Sticky Note1  
  - Sticky Note4

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow execution manually.  
    - Configuration: No parameters; triggers on user action.  
    - Inputs: None  
    - Outputs: Connects to the Globals node.  
    - Edge Cases: None typical; manual trigger requires user interaction.

  - **Globals**  
    - Type: Set Node  
    - Role: Defines global variables for GitHub repository details.  
    - Configuration:  
      - `repo.owner` = "BeyondspaceStudio" (default example)  
      - `repo.name` = "n8n-backup"  
      - `repo.path` = "credentials"  
    - Inputs: From manual trigger node.  
    - Outputs: Connects to "Get all files in given path" node.  
    - Edge Cases: Incorrect or missing values here will cause GitHub API calls to fail or fetch wrong data.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides setup instructions and usage notes for the user.  
    - Configuration: Contains detailed instructions on setting `repo.owner`, `repo.name`, and `repo.path`.  
    - Inputs/Outputs: None (informational only).

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Highlights the Globals node for user editing.  
    - Configuration: Simple note "Edit this node 👇".  
    - Inputs/Outputs: None.

---

#### 2.2 Fetching Credential Files from GitHub

- **Overview:**  
  This block queries the GitHub API to retrieve a list of all files located in the specified repository path that contains the credential backups.

- **Nodes Involved:**  
  - Get all files in given path  
  - Split the result

- **Node Details:**

  - **Get all files in given path**  
    - Type: HTTP Request  
    - Role: Calls GitHub REST API to list contents of the credentials folder.  
    - Configuration:  
      - URL dynamically constructed using expressions referencing `repo.owner`, `repo.name`, and `repo.path` from Globals node.  
      - Authentication: Uses predefined GitHub API credentials.  
    - Inputs: From Globals node.  
    - Outputs: JSON array of file metadata (including file paths).  
    - Edge Cases:  
      - Authentication failure if GitHub credentials are invalid.  
      - 404 if repo or path does not exist.  
      - Rate limiting by GitHub API.  

  - **Split the result**  
    - Type: SplitOut  
    - Role: Splits the array of file paths into individual items for processing one by one.  
    - Configuration: Splits on the `path` field of each file object.  
    - Inputs: From "Get all files in given path".  
    - Outputs: One item per file path to the next node.  
    - Edge Cases: Empty folder results in no output items, stopping further processing.

---

#### 2.3 Processing Each Credential File

- **Overview:**  
  For each credential file path, this block fetches the file content from GitHub, converts it from JSON text to a JSON object, and filters out files that should not be restored.

- **Nodes Involved:**  
  - Get file content from GitHub  
  - Convert files to JSON  
  - Check for skipped Credentials  
  - Sticky Note (near Check for skipped Credentials)

- **Node Details:**

  - **Get file content from GitHub**  
    - Type: GitHub Node (Resource: File, Operation: Get)  
    - Role: Retrieves the raw content of each credential file from GitHub.  
    - Configuration:  
      - Owner and repository hardcoded or linked to Globals values.  
      - File path dynamically set from the split file path.  
      - Uses GitHub API credentials.  
    - Inputs: From Split the result node.  
    - Outputs: File content in base64 or raw format.  
    - Edge Cases:  
      - File missing or deleted between listing and fetching.  
      - API rate limits or auth errors.

  - **Convert files to JSON**  
    - Type: ExtractFromFile  
    - Role: Converts the file content (JSON text) into a JSON object for further processing.  
    - Configuration: Operation set to "fromJson".  
    - Inputs: From Get file content from GitHub.  
    - Outputs: JSON object representing the credential data.  
    - Edge Cases:  
      - Malformed JSON causing parse errors.

  - **Check for skipped Credentials**  
    - Type: If Node  
    - Role: Filters out credential files that should not be restored, such as empty JSON files or the n8n account API credential.  
    - Configuration:  
      - Condition 1: Checks if `data` field is empty.  
      - Condition 2: Checks if `data.name` contains "n8n account".  
      - Combines conditions with OR logic.  
    - Inputs: From Convert files to JSON.  
    - Outputs:  
      - True branch: Skipped credentials (no further processing).  
      - False branch: Credentials to restore, passed to next node.  
    - Edge Cases:  
      - False negatives if naming conventions change.  
      - Empty or malformed data fields.

  - **Sticky Note (near Check for skipped Credentials)**  
    - Type: Sticky Note  
    - Role: Explains which credentials are skipped and invites user customization.  
    - Content:  
      - Skips empty JSON files and n8n account API credentials.  
      - Encourages editing this node as needed.

---

#### 2.4 Restoring Credentials into n8n

- **Overview:**  
  This block imports each valid credential JSON object into the n8n instance using the n8n API node.

- **Nodes Involved:**  
  - Restore n8n Credentials

- **Node Details:**

  - **Restore n8n Credentials**  
    - Type: n8n Node (Resource: Credential)  
    - Role: Sends a request to the n8n API to create or update credentials based on the imported JSON data.  
    - Configuration:  
      - `data` parameter is set by stringifying the `data.data` field from the JSON input.  
      - `name` parameter set from `data.name`.  
      - `credentialTypeName` set from `data.type`.  
      - Uses n8n API credentials for authentication.  
    - Inputs: From the False branch of Check for skipped Credentials node.  
    - Outputs: API response confirming credential restoration.  
    - Edge Cases:  
      - API authentication failure.  
      - Conflicts if credential names already exist.  
      - Invalid credential data causing API errors.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                                  | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                      |
|----------------------------|-------------------------|-------------------------------------------------|-------------------------------|------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Starts the workflow manually                     | None                          | Globals                      |                                                                                                 |
| Globals                    | Set                     | Sets GitHub repo owner, name, and path          | When clicking ‘Test workflow’ | Get all files in given path   | "## Edit this node 👇" (Sticky Note4)                                                           |
| Sticky Note1               | Sticky Note             | Provides setup instructions                       | None                          | None                         | Detailed setup instructions for repo.owner, repo.name, repo.path                               |
| Sticky Note4               | Sticky Note             | Highlights Globals node for editing               | None                          | None                         | "## Edit this node 👇"                                                                          |
| Get all files in given path | HTTP Request            | Fetches list of credential files from GitHub    | Globals                       | Split the result             |                                                                                                 |
| Split the result           | SplitOut                | Splits file list into individual file paths     | Get all files in given path   | Get file content from GitHub |                                                                                                 |
| Get file content from GitHub | GitHub                  | Retrieves content of each credential file        | Split the result              | Convert files to JSON        |                                                                                                 |
| Convert files to JSON      | ExtractFromFile          | Converts file content from JSON text to JSON    | Get file content from GitHub  | Check for skipped Credentials |                                                                                                 |
| Check for skipped Credentials | If                      | Filters out empty or unwanted credential files  | Convert files to JSON         | Restore n8n Credentials (False branch) | "## Skip credential\n- The empty json files\n- The n8n account api\n- ...edit this node at will" |
| Restore n8n Credentials    | n8n                      | Imports credentials into n8n instance via API   | Check for skipped Credentials | None                         |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Create a Set Node for Globals**  
   - Name: `Globals`  
   - Add three string fields:  
     - `repo.owner` (e.g., "BeyondspaceStudio")  
     - `repo.name` (e.g., "n8n-backup")  
     - `repo.path` (e.g., "credentials")  
   - Connect the Manual Trigger node to this node.

3. **Create an HTTP Request Node to List Files**  
   - Name: `Get all files in given path`  
   - Method: GET  
   - URL: Use expression to build URL:  
     `https://api.github.com/repos/{{ $json.repo.owner }}/{{ $json.repo.name }}/contents/{{ $json.repo.path }}`  
   - Authentication: Select GitHub API credentials (OAuth2 or Personal Access Token).  
   - Connect Globals node to this node.

4. **Create a SplitOut Node**  
   - Name: `Split the result`  
   - Field to split out: `path` (from the JSON array returned by previous node)  
   - Connect `Get all files in given path` node to this node.

5. **Create a GitHub Node to Get File Content**  
   - Name: `Get file content from GitHub`  
   - Resource: File  
   - Operation: Get  
   - Owner: Set to the same as `repo.owner` (can be hardcoded or use expression)  
   - Repository: Same as `repo.name`  
   - File Path: Use expression referencing the split file path: `={{ $json.path }}`  
   - Credentials: Use the same GitHub API credentials.  
   - Connect `Split the result` node to this node.

6. **Create an ExtractFromFile Node**  
   - Name: `Convert files to JSON`  
   - Operation: fromJson  
   - Connect `Get file content from GitHub` node to this node.

7. **Create an If Node to Filter Skipped Credentials**  
   - Name: `Check for skipped Credentials`  
   - Conditions (OR):  
     - Check if `data` field is empty (empty object or string).  
     - Check if `data.name` contains "n8n account".  
   - Connect `Convert files to JSON` node to this node.

8. **Create an n8n Node to Restore Credentials**  
   - Name: `Restore n8n Credentials`  
   - Resource: Credential  
   - Operation: Create or Update (default)  
   - Parameters:  
     - `data`: Set to `{{ JSON.stringify($json.data.data) }}`  
     - `name`: Set to `{{ $json.data.name }}`  
     - `credentialTypeName`: Set to `{{ $json.data.type }}`  
   - Credentials: Use n8n API credentials with appropriate permissions.  
   - Connect the False output (credentials to restore) of the If node to this node.

9. **Add Sticky Notes (Optional but Recommended)**  
   - Add notes near Globals node explaining how to configure repo details.  
   - Add notes near the If node explaining which credentials are skipped and how to customize.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow complements the [Backup Your Credentials to GitHub](https://n8n.io/workflows/2307-backup-your-credentials-to-github/) template by enabling restoration. | Workflow description and usage context.                                                         |
| For GitHub API authentication, use either OAuth2 or a Personal Access Token with repo read permissions.           | Credential setup requirement.                                                                    |
| For n8n API authentication, ensure the API user has permission to create/update credentials.                      | Credential setup requirement.                                                                    |
| Check out other templates by the author: [My n8n Templates](https://n8n.io/creators/bangank36/)                   | Additional resources and templates.                                                             |
| Skipped credentials include empty JSON files and the "n8n account" API credential by default; customize as needed. | Important for avoiding import errors or overwriting critical credentials.                        |

---

This documentation provides a complete and detailed understanding of the "Restore your credentials from GitHub" workflow, enabling users and automation agents to comprehend, reproduce, and modify the workflow confidently.