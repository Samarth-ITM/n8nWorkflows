Sync GitHub Workflows to n8n After Pull Request Merges

https://n8nworkflows.xyz/workflows/sync-github-workflows-to-n8n-after-pull-request-merges-4500


# Sync GitHub Workflows to n8n After Pull Request Merges

### 1. Workflow Overview

This workflow automates the synchronization of GitHub repository workflows into an n8n instance immediately after a pull request (PR) is merged. Its primary use case is to ensure that any new or modified n8n workflow files committed to a GitHub repository are automatically imported or updated inside n8n without manual intervention.

The workflow operates in several logical blocks:

- **1.1 GitHub Event Trigger and Validation:** Listens to GitHub pull request events and filters for merged PRs only.
- **1.2 Retrieve Commit and File Change Details:** Fetches detailed commit data for the merged PR to identify changed files.
- **1.3 Identify Workflow Files Changed:** Filters files changed by their modification status (added or modified).
- **1.4 Fetch Workflow JSON Content:** Downloads the content of each changed workflow file from GitHub.
- **1.5 Decode and Import Workflows in n8n:** Decodes the base64 content of workflows and either creates new workflows or updates existing ones in n8n.
- **1.6 Setup Instructions (Non-executable):** Provides documentation and setup notes for configuring the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 GitHub Event Trigger and Validation

**Overview:**  
This block triggers the workflow on GitHub pull request events and ensures that only merged pull requests proceed further.

**Nodes Involved:**  
- Github Trigger - When there is new pull request  
- Define Local Variables  
- Check if trigger event is pull request merged

**Node Details:**

- **Github Trigger - When there is new pull request**  
  - *Type:* GitHub Trigger  
  - *Role:* Listens for GitHub `pull_request` webhook events on the specified repository.  
  - *Configuration:*  
    - Owner: `duynb92` (configured via credential and parameter)  
    - Repository: `n8n-workflows`  
    - Event: `pull_request`  
    - Credential: GitHub API access token with repo and admin:repo_hook scopes  
  - *Inputs:* Webhook from GitHub on pull request events  
  - *Outputs:* JSON payload containing pull request event details  
  - *Potential failures:*  
    - Webhook misconfiguration  
    - Invalid or expired GitHub credentials  
    - Network issues or GitHub API downtime  

- **Define Local Variables**  
  - *Type:* Set  
  - *Role:* Initializes static variables for repository owner and name for reuse  
  - *Configuration:*  
    - `github_owner`: `"duynb92"`  
    - `repo_name`: `"n8n-workflows"`  
  - *Inputs:* From GitHub Trigger  
  - *Outputs:* Contains local variables accessible downstream  
  - *Edge cases:* None critical; variables must be kept in sync with the actual repository  

- **Check if trigger event is pull request merged**  
  - *Type:* Filter  
  - *Role:* Allows only events where the pull request is closed and merged to continue  
  - *Configuration:*  
    - Checks if `body.action` equals `"closed"`  
    - Checks if `pull_request.merged` boolean is true  
  - *Inputs:* Output of Define Local Variables  
  - *Outputs:* Passes only merged PR events  
  - *Potential failure:* Expression errors if event payload structure changes; false negatives if GitHub event schema evolves  

---

#### 2.2 Retrieve Commit and File Change Details

**Overview:**  
Fetches detailed information about the commit merged in the pull request, especially the list of files changed.

**Nodes Involved:**  
- Fetch merged commit details via GitHub API  
- Get files changes details

**Node Details:**

- **Fetch merged commit details via GitHub API**  
  - *Type:* HTTP Request  
  - *Role:* Requests commit details from GitHub REST API for the merged commit SHA  
  - *Configuration:*  
    - URL constructed dynamically using local variables and merged commit SHA from event:  
      `https://api.github.com/repos/{{github_owner}}/{{repo_name}}/commits/{{merge_commit_sha}}`  
    - Authentication: GitHub API credential (OAuth or PAT)  
  - *Inputs:* From filter confirming merged PR  
  - *Outputs:* JSON including commit details and `files` array with changed file info  
  - *Edge cases:*  
    - API rate limits  
    - Invalid commit SHA (unlikely)  
    - Network errors  

- **Get files changes details**  
  - *Type:* Split Out  
  - *Role:* Splits the array of changed files into individual items for parallel processing  
  - *Configuration:*  
    - Field to split out: `files`  
  - *Inputs:* Commit details JSON  
  - *Outputs:* One item per changed file with file metadata (e.g., filename, status)  
  - *Potential failure:* Empty or missing `files` property leads to no further processing  

---

#### 2.3 Identify Workflow Files Changed

**Overview:**  
Separates files changed into two categories: workflows that are newly added and workflows that are modified.

**Nodes Involved:**  
- Filter modified workflows  
- Filter created workflows

**Node Details:**

- **Filter modified workflows**  
  - *Type:* Filter  
  - *Role:* Passes only files whose status is `"modified"`  
  - *Configuration:* Checks `$json.status === "modified"`  
  - *Inputs:* Split file items  
  - *Outputs:* Modified workflow files only  
  - *Edge cases:* Other statuses (removed, renamed) are ignored here  

- **Filter created workflows**  
  - *Type:* Filter  
  - *Role:* Passes only files whose status is `"added"`  
  - *Configuration:* Checks `$json.status === "added"`  
  - *Inputs:* Split file items  
  - *Outputs:* Newly added workflow files only  
  - *Edge cases:* Does not handle renamed or removed files  

---

#### 2.4 Fetch Workflow JSON Content

**Overview:**  
Downloads the raw JSON file content of the identified workflow files from the GitHub repository.

**Nodes Involved:**  
- Fetch workflow content from Git1 (for modified files)  
- Fetch workflow content from Git (for created files)

**Node Details:**

- **Fetch workflow content from Git1 & Fetch workflow content from Git**  
  - *Type:* GitHub  
  - *Role:* Retrieves file content at the path specified by the filename field from GitHub repository  
  - *Configuration:*  
    - Owner and repo dynamically set from local variables  
    - File path: `filename` property from previous nodes  
    - Operation: `get` file content  
    - Output: raw file content as base64-encoded string in `content` field  
    - Credential: GitHub API token  
  - *Inputs:* Filtered files (added or modified)  
  - *Outputs:* File content JSON objects  
  - *Edge cases:*  
    - File deleted before fetch leads to 404 error  
    - Access denied if token lacks read permissions  
    - Large files might cause timeout issues  

---

#### 2.5 Decode and Import Workflows in n8n

**Overview:**  
Decodes the base64-encoded workflow JSON content and imports it into n8n either by creating new workflows or updating existing ones.

**Nodes Involved:**  
- Decode workflow content to json1 (for created workflows)  
- Decode workflow content to json (for modified workflows)  
- Create new workflow in n8n  
- Update workflow in n8n

**Node Details:**

- **Decode workflow content to json & Decode workflow content to json1**  
  - *Type:* Set  
  - *Role:* Decodes the base64 `content` field to JSON string that represents the workflow object  
  - *Configuration:*  
    - Uses expression: `{{$json.content.base64Decode()}}` to decode  
    - Outputs decoded JSON as new item data  
  - *Inputs:* Raw content from GitHub file fetch nodes  
  - *Outputs:* Workflow JSON object ready for import  
  - *Edge cases:*  
    - Malformed base64 content causes decoding errors  
    - Invalid JSON structure leads to downstream errors  

- **Create new workflow in n8n**  
  - *Type:* n8n node (n8n API)  
  - *Role:* Creates a new workflow in n8n using the decoded workflow JSON  
  - *Configuration:*  
    - Operation: `create`  
    - Workflow object: uses expression `{{$json.toJsonString()}}` from decoded content  
    - Credential: n8n API credentials with create permissions  
  - *Inputs:* Decoded new workflow JSON  
  - *Outputs:* Confirmation and metadata of created workflow  
  - *Potential failures:*  
    - API auth failures  
    - Workflow validation errors (malformed or incompatible workflow JSON)  
    - Network issues  

- **Update workflow in n8n**  
  - *Type:* n8n node (n8n API)  
  - *Role:* Updates an existing workflow in n8n with the decoded workflow JSON  
  - *Configuration:*  
    - Operation: `update`  
    - Workflow ID: extracted from decoded JSON `id` field  
    - Workflow object: full JSON string of workflow  
    - Credential: n8n API credentials with update permissions  
  - *Inputs:* Decoded modified workflow JSON  
  - *Outputs:* Confirmation and metadata of updated workflow  
  - *Edge cases:*  
    - Workflow ID missing or incorrect causes update failure  
    - API permission issues  
    - Conflicts if workflow is locked or in use  

---

#### 2.6 Setup Instructions (Non-executable)

**Overview:**  
Provides important setup notes and documentation for users to configure the workflow successfully.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* Sticky Note (visual only)  
  - *Role:* Contains setup instructions and recommendations for credentials and webhook configuration  
  - *Content Highlights:*  
    - Connect GitHub with PAT (Personal Access Token) classic with `repo` and `admin:repo_hook` scopes  
    - Configure n8n API credentials in n8n nodes  
    - Set repository owner and name in Define Local Variables node  
    - Make sure GitHub repository webhook for `pull_request` events points to this workflow's webhook URL  
  - *Inputs/Outputs:* N/A  
  - *Link:* https://docs.n8n.io/api/authentication/  

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                             | Input Node(s)                            | Output Node(s)                           | Sticky Note                                                                                                                       |
|-------------------------------------|---------------------|--------------------------------------------|----------------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Github Trigger - When there is new pull request | GitHub Trigger      | Trigger on pull request events              | Webhook (GitHub)                       | Define Local Variables                   | Part of GitHub integration setup; requires PAT with `repo` and `admin:repo_hook` scopes.                                         |
| Define Local Variables               | Set                 | Define repo owner and name variables        | Github Trigger                         | Check if trigger event is pull request merged | Must update variables if repo or owner changes.                                                                                   |
| Check if trigger event is pull request merged | Filter              | Allow only merged PR events                  | Define Local Variables                 | Fetch merged commit details via GitHub API | Validates event payload; relies on GitHub PR event structure.                                                                     |
| Fetch merged commit details via GitHub API | HTTP Request        | Get commit details and changed files        | Check if trigger event is pull request merged | Get files changes details                | GitHub API rate limits or network issues possible.                                                                                |
| Get files changes details            | Split Out           | Split commit files into individual items    | Fetch merged commit details via GitHub API | Filter modified workflows, Filter created workflows | Empty files list aborts further execution.                                                                                        |
| Filter modified workflows            | Filter              | Pass only modified files                      | Get files changes details              | Fetch workflow content from Git1         | Ignores added or removed files.                                                                                                  |
| Filter created workflows             | Filter              | Pass only added files                         | Get files changes details              | Fetch workflow content from Git           | Ignores modified or removed files.                                                                                                |
| Fetch workflow content from Git1     | GitHub              | Fetch content of modified workflow files     | Filter modified workflows              | Decode workflow content to json           | 404 if file not found; token permission issues possible.                                                                          |
| Fetch workflow content from Git      | GitHub              | Fetch content of new workflow files          | Filter created workflows               | Decode workflow content to json1          | Same as above.                                                                                                                    |
| Decode workflow content to json      | Set                 | Decode base64 file content to JSON for modified workflows | Fetch workflow content from Git1       | Update workflow in n8n                    | Decoding or JSON parse errors possible.                                                                                          |
| Decode workflow content to json1     | Set                 | Decode base64 file content to JSON for new workflows | Fetch workflow content from Git        | Create new workflow in n8n                 | Same as above.                                                                                                                    |
| Update workflow in n8n               | n8n (API)           | Update existing workflow with new content   | Decode workflow content to json        | -                                       | Workflow ID must exist; API auth required.                                                                                       |
| Create new workflow in n8n           | n8n (API)           | Create new workflow from JSON content       | Decode workflow content to json1       | -                                       | API auth required.                                                                                                                |
| Sticky Note                        | Sticky Note          | Setup instructions and notes                 | -                                    | -                                       | [Setup instructions and credential notes](https://docs.n8n.io/api/authentication/)                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node:**  
   - Type: GitHub Trigger  
   - Event: `pull_request`  
   - Owner: `duynb92`  
   - Repository: `n8n-workflows`  
   - Credential: GitHub API with Personal Access Token (classic) having `repo` and `admin:repo_hook` scopes enabled  
   - Position: Start node to receive webhook events.

2. **Create Set Node "Define Local Variables":**  
   - Assign `github_owner` = `"duynb92"`  
   - Assign `repo_name` = `"n8n-workflows"`  
   - Connect output of GitHub Trigger to this node.

3. **Create Filter Node "Check if trigger event is pull request merged":**  
   - Condition 1: `{{$json.body.action}}` equals `"closed"`  
   - Condition 2: `{{$json.body.pull_request.merged}}` is boolean true  
   - Connect output of "Define Local Variables" to this filter node.

4. **Create HTTP Request Node "Fetch merged commit details via GitHub API":**  
   - Method: GET  
   - URL: `https://api.github.com/repos/{{$json.github_owner}}/{{$json.repo_name}}/commits/{{$json.body.pull_request.merge_commit_sha}}`  
   - Authentication: GitHub API credentials (same as trigger)  
   - Connect from "Check if trigger event is pull request merged".

5. **Create Split Out Node "Get files changes details":**  
   - Field to split out: `files`  
   - Connect from "Fetch merged commit details via GitHub API".

6. **Create Filter Node "Filter modified workflows":**  
   - Condition: `$json.status` equals `"modified"`  
   - Connect from "Get files changes details".

7. **Create Filter Node "Filter created workflows":**  
   - Condition: `$json.status` equals `"added"`  
   - Connect from "Get files changes details".

8. **Create GitHub Node "Fetch workflow content from Git1" (for modified workflows):**  
   - Operation: Get file content  
   - Owner: Use expression `{{$json.github_owner}}` from local variables  
   - Repository: Use expression `{{$json.repo_name}}`  
   - File Path: Use `$json.filename` from filtered file item  
   - Credential: GitHub API credentials  
   - Connect from "Filter modified workflows".

9. **Create GitHub Node "Fetch workflow content from Git" (for created workflows):**  
   - Same configuration as step 8  
   - Connect from "Filter created workflows".

10. **Create Set Node "Decode workflow content to json" (for modified workflows):**  
    - Mode: Raw  
    - JSON Output Expression: `{{$json.content.base64Decode()}}`  
    - Connect from "Fetch workflow content from Git1".

11. **Create Set Node "Decode workflow content to json1" (for created workflows):**  
    - Same as step 10  
    - Connect from "Fetch workflow content from Git".

12. **Create n8n Node "Update workflow in n8n":**  
    - Operation: Update  
    - Workflow ID: Use expression `{{$json.id}}` from decoded JSON  
    - Workflow Object: Use expression `{{$json.toJsonString()}}`  
    - Credential: n8n API credentials with update permission  
    - Connect from "Decode workflow content to json".

13. **Create n8n Node "Create new workflow in n8n":**  
    - Operation: Create  
    - Workflow Object: Use expression `{{$json.toJsonString()}}`  
    - Credential: n8n API credentials with create permission  
    - Connect from "Decode workflow content to json1".

14. **Add a Sticky Note node for Setup Instructions:**  
    - Content:  
      ```
      ## Setup

      1. Connect GitHub: Use the GitHub Trigger node and configure GitHub API credentials.
         Recommended: Use GitHub PAT (Personal Access Token) classic with `repo` and `admin:repo_hook` scopes.
      2. Connect n8n API: Provide your n8n API credentials in the n8n nodes. See https://docs.n8n.io/api/authentication/
      3. Set repository variables: Update `github_owner` and `repo_name` in the Define Local Variables node.
      4. Enable webhook: Ensure your GitHub repository has a webhook for `pull_request` events pointing to this workflow.
      ```
    - Position it for visibility in the design canvas.

15. **Validate and Activate the Workflow:**  
    - Ensure all credentials are valid and permissions are sufficient.  
    - Test with a pull request merge event to confirm workflows are synchronized correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                          |
|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Use GitHub PAT (Personal Access Token) classic with `repo` and `admin:repo_hook` scopes for full webhook and repo access. | GitHub credential setup recommendation                                  |
| n8n API credentials must be configured with appropriate permissions for workflow creation and updates.                     | https://docs.n8n.io/api/authentication/                                 |
| The workflow assumes all workflow files are valid n8n JSON files; malformed files can cause errors during import.           | Consider validating workflow JSON externally or adding error handling.  |
| The workflow currently only handles `added` and `modified` file statuses; deleted workflows are not processed.              | Enhancement opportunity for sync completeness                            |
| GitHub webhook for `pull_request` events must be set up in the GitHub repository settings pointing to this workflow URL.   | GitHub repository webhook configuration                                 |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow designed with n8n, complying strictly with content policies, containing no illegal or offensive elements. All processed data are legal and public.