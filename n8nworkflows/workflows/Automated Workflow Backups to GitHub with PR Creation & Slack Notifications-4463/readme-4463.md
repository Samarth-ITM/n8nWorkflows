Automated Workflow Backups to GitHub with PR Creation & Slack Notifications

https://n8nworkflows.xyz/workflows/automated-workflow-backups-to-github-with-pr-creation---slack-notifications-4463


# Automated Workflow Backups to GitHub with PR Creation & Slack Notifications

### 1. Workflow Overview

This workflow automates the backup of n8n workflows by pushing their JSON definitions to a GitHub repository, creating a dedicated branch for each backup operation, and opening a Pull Request (PR) to merge changes into the main branch. It integrates with Slack to notify a channel once the PR is created. The workflow is intended for n8n users who want to maintain versioned backups of their workflows on GitHub with automatic notifications.

The workflow consists of the following logical blocks:

- **1.1 Initialization and Variable Setup:** Starting point with manual trigger and setting local variables for GitHub repo and branch naming.
- **1.2 Retrieve Existing GitHub Workflows:** Fetch workflows currently stored in the GitHub repo.
- **1.3 Retrieve Current n8n Workflows:** Fetch all workflows from the n8n instance.
- **1.4 Compare and Find New or Updated Workflows:** Determine which workflows are new or have been updated since last backup.
- **1.5 Branch and Commit Management:** Create a new branch, commit new or updated workflows to the branch.
- **1.6 Pull Request Creation:** Create a PR on GitHub to merge the branch back to main.
- **1.7 Slack Notification:** Notify a Slack channel about the PR creation.
- **1.8 Control and Flow Management:** Conditional branching and merge nodes to ensure proper synchronization and error handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Variable Setup

- **Overview:** Starts the workflow manually, defines local variables including GitHub repo owner, repo name, and the dynamic branch name that includes a timestamp.
- **Nodes Involved:**  
  - Click me to trigger  
  - Define Local Variables  
  - n8n (Fetch workflows from n8n)

- **Node Details:**

  - **Click me to trigger**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Define Local Variables"  
    - Failure modes: None (manual start)

  - **Define Local Variables**  
    - Type: Set  
    - Role: Defines static and dynamic variables for the workflow run, including:  
      - `branch_name`: dynamically built as `workflows_YYYY-MM-DD-hh-mm-ss` (timestamped branch name)  
      - `github_owner`: GitHub username/organization ("duynb92")  
      - `repo_name`: repository name ("n8n-workflows")  
    - Key expressions: Uses `$now.format()` to create the branch name.  
    - Inputs: From manual trigger  
    - Outputs: To n8n node and GitHub retrieval nodes  
    - Edge cases: Incorrect date formatting or variable references can cause errors.

  - **n8n**  
    - Type: n8n node (internal API)  
    - Role: Retrieves all workflows currently saved in the n8n instance.  
    - Credentials: Uses configured n8n API credentials.  
    - Inputs: From "Define Local Variables"  
    - Outputs: To merge nodes for comparison  
    - Edge cases: API auth errors, timeouts, empty data.

---

#### 2.2 Retrieve Existing GitHub Workflows

- **Overview:** Lists files currently stored in the GitHub repository under the `workflows` directory and fetches their content.
- **Nodes Involved:**  
  - Get all workflows on GitHub  
  - Get content for each workflow  
  - Base64decode workflow content

- **Node Details:**

  - **Get all workflows on GitHub**  
    - Type: GitHub (list files)  
    - Role: Lists all workflow files under the `workflows` path in the repo.  
    - Config: Uses variables for owner and repo name.  
    - Credentials: GitHub API credentials.  
    - Inputs: From "Define Local Variables"  
    - Outputs: To "Get content for each workflow"  
    - Edge cases: API rate limits, repo access errors.

  - **Get content for each workflow**  
    - Type: GitHub (get file content)  
    - Role: Fetches the raw content of each workflow file listed.  
    - Config: Uses dynamic path from previous node.  
    - Credentials: GitHub API.  
    - Inputs: From "Get all workflows on GitHub"  
    - Outputs: To "Base64decode workflow content"  
    - Edge cases: File missing or moved, API errors.

  - **Base64decode workflow content**  
    - Type: Set  
    - Role: Decodes the base64 encoded workflow JSON content retrieved from GitHub.  
    - Configuration: Uses expression `$json.content.base64Decode()` to produce JSON workflow definitions.  
    - Inputs: From "Get content for each workflow"  
    - Outputs: To "Find updated workflows" and "Find new workflows" nodes.  
    - Edge cases: Malformed base64 data, decoding failures.

---

#### 2.3 Compare and Find New or Updated Workflows

- **Overview:** Compares workflows fetched from n8n with those in GitHub to find workflows that are new or updated since last backup.
- **Nodes Involved:**  
  - Find new workflows (merge)  
  - Find updated workflows (merge)

- **Node Details:**

  - **Find new workflows**  
    - Type: Merge  
    - Role: Detects workflows present in n8n but not in GitHub (new workflows).  
    - Config: Combines inputs, joins on workflow `id`, keeps entries from n8n not matched in GitHub (keepNonMatches).  
    - Inputs: n8n workflows and GitHub workflows decoded content.  
    - Outputs: To "Wait until branch created1" and "Wait for finished".  
    - Edge cases: Mismatched IDs, empty data arrays.

  - **Find updated workflows**  
    - Type: Merge  
    - Role: Detects workflows that exist on both sides but have different `updatedAt` timestamps.  
    - Config: Combines inputs, joins on `id` and `updatedAt` fields, keeping n8n workflows that differ.  
    - Inputs: n8n workflows and GitHub workflows decoded content.  
    - Outputs: To "Wait for finished" and "Wait until branch created".  
    - Edge cases: Timestamp format mismatch, empty arrays.

---

#### 2.4 Branch and Commit Management

- **Overview:** Creates a new Git branch based on current main commit SHA, commits new and updated workflows separately, and waits for all commits to finish.
- **Nodes Involved:**  
  - If there are any changes in workflows (if)  
  - Get latest commit SHA on main via GitHub API (httpRequest)  
  - Create new branch via GitHub API (httpRequest)  
  - Wait until branch created / Wait until branch created1 (merge)  
  - Filter empty data / Filter empty data1 (filter)  
  - Create new commit to add new workflow (GitHub)  
  - Create new commit to update changed workflow (GitHub)  
  - Wait for GitHub create/edit file finished (merge)  
  - No Operation, do nothing (noOp)

- **Node Details:**

  - **If there are any changes in workflows**  
    - Type: If  
    - Role: Checks if there are any new or updated workflows to process.  
    - Config: OR condition checking if either "Find new workflows" or "Find updated workflows" outputs are non-empty.  
    - Inputs: From "Wait for finished" node.  
    - Outputs: True branch leads to "Get latest commit SHA", False goes to "No Operation".  
    - Edge cases: Empty arrays, false positives.

  - **Get latest commit SHA on main via GitHub API**  
    - Type: HTTP Request  
    - Role: Retrieves the SHA hash of the latest commit on the main branch, required to create a new branch.  
    - Config: Uses GitHub API endpoint for refs on `main` branch with authentication.  
    - Inputs: From "If there are any changes in workflows" true branch.  
    - Outputs: To "Create new branch via GitHub API".  
    - Edge cases: API errors, auth failure.

  - **Create new branch via GitHub API**  
    - Type: HTTP Request  
    - Role: Creates a new branch with the name defined earlier, pointing to the latest commit SHA retrieved.  
    - Config: POST to GitHub refs API with branch name and sha in JSON body.  
    - Inputs: From "Get latest commit SHA on main via GitHub API".  
    - Outputs: To "Wait until branch created" and "Wait until branch created1".  
    - Edge cases: Branch name conflicts, API errors.

  - **Wait until branch created / Wait until branch created1**  
    - Type: Merge  
    - Role: Synchronization points waiting for branch creation before proceeding to commit new or updated workflows respectively.  
    - Inputs: From "Create new branch via GitHub API" and from workflow comparison nodes.  
    - Outputs: To respective Filter nodes.  
    - Edge cases: Timeout if branch creation fails.

  - **Filter empty data / Filter empty data1**  
    - Type: Filter  
    - Role: Filters out empty or invalid workflow items because merge nodes always output data arrays.  
    - Configuration: Checks that workflow `id` is not empty.  
    - Inputs: From branch creation waits and workflow comparison nodes.  
    - Outputs: To commit nodes.  
    - Edge cases: Empty or malformed workflow data.

  - **Create new commit to add new workflow**  
    - Type: GitHub (create file)  
    - Role: Commits new workflow JSON files to the created branch.  
    - Config: Creates files under `workflows/{{workflow id}}.json` with commit message including workflow id.  
    - Inputs: From "Filter empty data".  
    - Outputs: To "Wait for GitHub create/edit file finished".  
    - Edge cases: Commit conflicts, API rate limits.

  - **Create new commit to update changed workflow**  
    - Type: GitHub (edit file)  
    - Role: Updates existing workflow JSON files in the branch.  
    - Config: Edits files under `workflows/{{workflow id}}.json` with updated content and commit message.  
    - Inputs: From "Filter empty data1".  
    - Outputs: To "Wait for GitHub create/edit file finished".  
    - Edge cases: File missing errors, merge conflicts.

  - **Wait for GitHub create/edit file finished**  
    - Type: Merge  
    - Role: Waits for all commit operations to finish before creating a PR.  
    - Inputs: From both commit nodes.  
    - Outputs: To "Create new PR via GitHub API".  
    - Edge cases: Timeout, partial failures.

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Handles the case where no workflows have changed; ends the workflow cleanly.  
    - Inputs: From "If there are any changes in workflows" false branch.  
    - Outputs: None.  
    - Edge cases: None.

---

#### 2.5 Pull Request Creation

- **Overview:** Creates a pull request on GitHub from the newly created branch to the main branch.
- **Nodes Involved:**  
  - Create new PR via GitHub API

- **Node Details:**

  - **Create new PR via GitHub API**  
    - Type: HTTP Request  
    - Role: Creates a pull request with the title "Automated PR for <branch_name>" targeting main branch.  
    - Config: POST to GitHub PRs API with JSON specifying title, head branch, base branch, and empty body.  
    - Inputs: From "Wait for GitHub create/edit file finished".  
    - Credentials: GitHub API.  
    - Outputs: To Slack notification node.  
    - Edge cases: PR creation errors, branch conflicts.

---

#### 2.6 Slack Notification

- **Overview:** Sends a notification message to a Slack channel once the PR has been successfully created.
- **Nodes Involved:**  
  - Slack

- **Node Details:**

  - **Slack**  
    - Type: Slack  
    - Role: Posts a formatted message with PR link and workflow name to a specified Slack channel.  
    - Config: Uses Slack Bot Access Token. Channel ID is fixed (C08NZ6D9CRF). Message includes a firework emoji and clickable PR URL.  
    - Inputs: From "Create new PR via GitHub API".  
    - Outputs: None (end node).  
    - Edge cases: Slack API auth errors, channel permission issues.

---

#### 2.7 Control and Flow Management

- **Overview:** Merge nodes synchronize parallel branches; filter nodes clean data arrays; conditional nodes branch logic.
- **Nodes Involved:**  
  - Wait for finished  
  - Wait for GitHub create/edit file finished  
  - Wait until branch created / Wait until branch created1  
  - Filter empty data / Filter empty data1  
  - If there are any changes in workflows  
  - No Operation, do nothing

- **Node Details:**  
  These nodes ensure proper sequencing and conditional branching, handling cases where no changes exist or ensuring that commits and branch creations complete before continuing. Merge nodes use "chooseBranch" mode to select input data from specific inputs. Filters remove empty or invalid data to prevent errors downstream.

---

### 3. Summary Table

| Node Name                        | Node Type               | Functional Role                                  | Input Node(s)                           | Output Node(s)                              | Sticky Note                                                                                                            |
|---------------------------------|-------------------------|-------------------------------------------------|---------------------------------------|---------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Click me to trigger             | Manual Trigger          | Entry point to start workflow manually           | None                                  | Define Local Variables                       |                                                                                                                        |
| Define Local Variables          | Set                     | Define GitHub repo owner, repo name, branch name| Click me to trigger                   | n8n, Get all workflows on GitHub            |                                                                                                                        |
| n8n                            | n8n API                 | Retrieve all current n8n workflows                | Define Local Variables                | Find new workflows, Find updated workflows  |                                                                                                                        |
| Get all workflows on GitHub    | GitHub                  | List workflow files in GitHub repo                | Define Local Variables                | Get content for each workflow                |                                                                                                                        |
| Get content for each workflow  | GitHub                  | Fetch workflow file content from GitHub           | Get all workflows on GitHub           | Base64decode workflow content                |                                                                                                                        |
| Base64decode workflow content  | Set                     | Decode base64 encoded workflow content             | Get content for each workflow         | Find updated workflows, Find new workflows  |                                                                                                                        |
| Find new workflows             | Merge                   | Find workflows new to n8n (not on GitHub)          | n8n, Base64decode workflow content   | Wait until branch created1, Wait for finished |                                                                                                                        |
| Find updated workflows         | Merge                   | Find workflows updated in n8n since last GitHub backup| n8n, Base64decode workflow content   | Wait for finished, Wait until branch created |                                                                                                                        |
| Wait for finished              | Merge                   | Synchronize new and updated workflows checks       | Find new workflows, Find updated workflows | If there are any changes in workflows        |                                                                                                                        |
| If there are any changes in workflows | If                      | Conditional check to continue only if changes exist| Wait for finished                    | Get latest commit SHA on main, No Operation  |                                                                                                                        |
| Get latest commit SHA on main via GitHub API | HTTP Request           | Get latest main branch commit SHA for new branch   | If there are any changes in workflows | Create new branch via GitHub API             |                                                                                                                        |
| Create new branch via GitHub API | HTTP Request            | Create new GitHub branch for backup                 | Get latest commit SHA on main         | Wait until branch created, Wait until branch created1 |                                                                                                                        |
| Wait until branch created      | Merge                   | Synchronize branch creation before updates          | Create new branch via GitHub API, Find updated workflows | Filter empty data1                            |                                                                                                                        |
| Wait until branch created1     | Merge                   | Synchronize branch creation before new additions    | Create new branch via GitHub API, Find new workflows | Filter empty data                             |                                                                                                                        |
| Filter empty data              | Filter                  | Filter empty entries from new workflows             | Wait until branch created1            | Create new commit to add new workflow        | Empty data can happen because 'Find new workflows' always output data                                                  |
| Filter empty data1             | Filter                  | Filter empty entries from updated workflows          | Wait until branch created             | Create new commit to update changed workflow | Empty data can happen because 'Find updated workflows' always output data                                              |
| Create new commit to add new workflow | GitHub                  | Commit new workflow files to new branch              | Filter empty data                     | Wait for GitHub create/edit file finished    |                                                                                                                        |
| Create new commit to update changed workflow | GitHub                  | Commit updated workflow files to new branch          | Filter empty data1                    | Wait for GitHub create/edit file finished    |                                                                                                                        |
| Wait for GitHub create/edit file finished | Merge                   | Wait for all commit operations to finish             | Create new commit to add new workflow, Create new commit to update changed workflow | Create new PR via GitHub API                   |                                                                                                                        |
| Create new PR via GitHub API   | HTTP Request            | Create a PR from backup branch to main               | Wait for GitHub create/edit file finished | Slack                                         |                                                                                                                        |
| Slack                         | Slack                   | Notify Slack channel with PR creation message       | Create new PR via GitHub API          | None                                         |                                                                                                                        |
| No Operation, do nothing      | NoOp                    | Ends workflow if no changes detected                 | If there are any changes in workflows (false branch) | None                                         |                                                                                                                        |
| Sticky Note                   | Sticky Note             | Setup instructions and links                         | None                                  | None                                         | 1. GitHub credentials: Add your GitHub API credentials in n8n. 2. Slack integration: Connect your Slack Bot token. 3. Update repo details in “Define Local Variables”. 4. n8n API key docs: https://docs.n8n.io/api/authentication/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "Click me to trigger" to serve as the workflow entry point.

2. **Add a Set node** named "Define Local Variables" with the following variables:  
   - `branch_name` (string): Use expression `workflows_{{$now.format("yyyy-MM-dd-hh-mm-ss")}}`  
   - `github_owner` (string): Set to your GitHub username or organization (e.g., "duynb92")  
   - `repo_name` (string): Set to your target repo name (e.g., "n8n-workflows")  
   Connect the manual trigger to this node.

3. **Add an n8n node** named "n8n" configured to use your n8n API credentials to fetch all workflows from your n8n instance. Connect "Define Local Variables" to this node.

4. **Add a GitHub node** named "Get all workflows on GitHub" configured to:  
   - Operation: List files  
   - Resource: File  
   - Owner: Use expression `={{ $json.github_owner }}` from "Define Local Variables"  
   - Repository: Use expression `={{ $json.repo_name }}`  
   - File Path: `workflows`  
   Connect "Define Local Variables" to this node.

5. **Add a GitHub node** named "Get content for each workflow" configured to:  
   - Operation: Get file content  
   - Resource: File  
   - Owner, Repository as above  
   - File Path: Use expression `={{ $json.path }}` from the previous node  
   Connect "Get all workflows on GitHub" to this node.

6. **Add a Set node** named "Base64decode workflow content" with mode "Raw", and define JSON output as expression: `={{ $json.content.base64Decode() }}`. Connect "Get content for each workflow" to this node.

7. **Add two Merge nodes** named "Find new workflows" and "Find updated workflows":  
   - "Find new workflows": Mode "Combine", Join Mode "keepNonMatches" on field `id`, output from input 1 (n8n workflows).  
   - "Find updated workflows": Mode "Combine", Join Mode "keepNonMatches" on fields `id, updatedAt`, output from input 1.  
   Connect "n8n" and "Base64decode workflow content" to both merge nodes according to join logic.

8. **Add a Merge node "Wait for finished"** with mode "chooseBranch" and output "empty", connected to outputs of "Find new workflows" and "Find updated workflows".

9. **Add an If node** named "If there are any changes in workflows" with condition:  
   - OR condition checking if outputs of "Find new workflows" or "Find updated workflows" are not empty arrays.  
   Connect "Wait for finished" to this node.

10. **Add an HTTP Request node** named "Get latest commit SHA on main via GitHub API" configured to GET:  
    - URL: `https://api.github.com/repos/{{ $json.github_owner }}/{{ $json.repo_name }}/git/ref/heads/main`  
    - Authentication: GitHub API credentials  
    Connect the true output of the If node here.

11. **Add an HTTP Request node** named "Create new branch via GitHub API" with POST:  
    - URL: `https://api.github.com/repos/{{ $json.github_owner }}/{{ $json.repo_name }}/git/refs`  
    - Body JSON:  
      ```json
      {
        "ref": "refs/heads/{{ $json.branch_name }}",
        "sha": "{{ $json.object.sha }}"
      }
      ```  
    - Authentication: GitHub API credentials  
    Connect "Get latest commit SHA on main via GitHub API" to this node.

12. **Add two Merge nodes** named "Wait until branch created" and "Wait until branch created1" with mode "chooseBranch", inputs from branch creation node and respective merge nodes for updates and new workflows.

13. **Add two Filter nodes** named "Filter empty data" and "Filter empty data1" to filter out workflows with empty `id`. Connect "Wait until branch created1" to "Filter empty data" and "Wait until branch created" to "Filter empty data1".

14. **Add two GitHub nodes** named "Create new commit to add new workflow" and "Create new commit to update changed workflow":  
    - Add new workflow node: Operation "Create file"  
    - Update workflow node: Operation "Edit file"  
    - Both set file path to `workflows/{{ $json.id }}.json`  
    - Commit messages: "Add new workflow {{ $json.id }}" and "Update workflow {{ $json.id }}" respectively  
    - Branch: Use the branch defined by `branch_name`  
    - Connect "Filter empty data" to adding new workflow commit node, and "Filter empty data1" to update commit node.

15. **Add a Merge node** named "Wait for GitHub create/edit file finished" to wait for both commit operations to complete.

16. **Add an HTTP Request node** named "Create new PR via GitHub API" configured to POST:  
    - URL: `https://api.github.com/repos/{{ $json.github_owner }}/{{ $json.repo_name }}/pulls`  
    - Body JSON:  
      ```json
      {
        "title": "Automated PR for {{ $json.branch_name }}",
        "body": "",
        "head": "{{ $json.branch_name }}",
        "base": "main"
      }
      ```  
    - Authentication: GitHub API credentials  
    Connect "Wait for GitHub create/edit file finished" to this node.

17. **Add a Slack node** named "Slack" configured to:  
    - Post message to your Slack channel (e.g., channel ID "C08NZ6D9CRF")  
    - Message text: `:firework: {{ $workflow.name }} executed succesfully. \nIt created a new <{{ $json.html_url }}|GitHub PR>.`  
    - Use Slack Bot Access Token credentials.  
    Connect "Create new PR via GitHub API" to this node.

18. **Add a No Operation node** named "No Operation, do nothing" connected to the false branch of the "If there are any changes in workflows" node.

19. **Add a Sticky Note** to document setup instructions:  
    - GitHub credentials setup  
    - Slack integration token  
    - Update repo owner, name, and workflow path in variables  
    - n8n API key documentation link: https://docs.n8n.io/api/authentication/

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Setup: Add GitHub API credentials in n8n to authenticate GitHub nodes.                                       | Internal credential configuration                                                                             |
| Slack integration requires a Bot Access Token with permission to post to the target channel.                 | Slack App configuration                                                                                       |
| Update variables `github_owner`, `repo_name`, and workflow directory path in the "Define Local Variables" node.| Essential for correct GitHub repo targeting                                                                  |
| n8n API key is needed to fetch workflows programmatically. See [n8n API Authentication docs](https://docs.n8n.io/api/authentication/). | Documentation link                                                                                            |

---

This completes the comprehensive documentation of the "Automated Workflow Backups to GitHub with PR Creation & Slack Notifications" n8n workflow. It details the nodes, logic, configuration, and instructions to reproduce or modify the workflow reliably.