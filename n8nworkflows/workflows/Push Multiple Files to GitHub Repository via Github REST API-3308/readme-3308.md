Push Multiple Files to GitHub Repository via Github REST API

https://n8nworkflows.xyz/workflows/push-multiple-files-to-github-repository-via-github-rest-api-3308


# Push Multiple Files to GitHub Repository via Github REST API

### 1. Workflow Overview

This n8n workflow automates the process of pushing multiple files simultaneously to a GitHub repository using GitHub's REST API, overcoming the limitation of n8n’s native GitHub node that only supports single-file uploads. Its main use cases include batch updates, such as deploying website contents, updating documentation, or managing configuration files at once, with atomic commit operations to ensure repository consistency.

The workflow’s logic is structured into these primary blocks:

- **1.1 Initialization and Input Setup:** Triggering the workflow manually and setting GitHub credentials, repository details, commit message, and defining file contents.
- **1.2 Repository State Retrieval:** Fetching the latest commit SHA for the target branch and retrieving the base tree SHA necessary for building the new commit.
- **1.3 Git Tree Creation:** Creating a new Git tree that includes the multiple files to be uploaded with their paths and content.
- **1.4 Commit Creation:** Making a new commit referencing the newly created tree and the latest commit as the parent.
- **1.5 Branch Update:** Updating the reference of the target branch to point to the new commit, making the file changes live in the repository.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Input Setup

**Overview:**  
This block prepares all required input data and credentials to interact with GitHub during the workflow execution. It defines the files’ content to be committed and configures repository and authentication information.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set Github Info  
- File 1  
- File 2  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type & Role:* Manual Trigger node to start the workflow on demand.  
  - *Config:* No parameters.  
  - *Input/Output:* No input; outputs trigger next node `Set Github Info`.  
  - *Edge Cases:* None (manual start).

- **Set Github Info**  
  - *Type & Role:* Set node — defines workflow variables including GitHub Token, Username, Repository, Branch, and Commit Message.  
  - *Config:* User must replace placeholders with their actual GitHub Personal Access Token (PAT), repository info, branch name (default "main"), and commit message.  
  - *Expressions:* None, static assignment.  
  - *Connections:* Input from manual trigger; output to `File 1`.  
  - *Edge Cases:* Empty or invalid token leads to authorization errors downstream. Missing or incorrect repo/branch causes API failure.

- **File 1**  
  - *Type & Role:* Set node — contains content for the first file to be uploaded.  
  - *Config:* Static string assigned to “content” field: "This is the content of your file #1."  
  - *Connections:* Input from `Set Github Info`; output to `File 2`.  
  - *Edge Cases:* Content must be properly formatted; empty content results in empty file on GitHub.

- **File 2**  
  - *Type & Role:* Set node — content for the second file to be pushed.  
  - *Config:* Static string content: "This is the content of your file #2."  
  - *Connections:* Input from `File 1`; output to `Get latest commit SHA`.  
  - *Edge Cases:* Same as File 1.

---

#### 2.2 Repository State Retrieval

**Overview:**  
This block fetches the current repository state needed to base the new commit on. It obtains the latest commit SHA for the target branch and then retrieves the tree SHA that represents the current state of files in that commit.

**Nodes Involved:**  
- Get latest commit SHA  
- Get base tree SHA  

**Node Details:**

- **Get latest commit SHA**  
  - *Type & Role:* HTTP Request node — calls GitHub API endpoint `/git/refs/heads/{branch}` to retrieve the reference object that includes the latest commit SHA for the branch.  
  - *Config:*  
    - Method: GET (default)  
    - URL constructed dynamically from variables using expressions: for username, repo, branch.  
    - Authorization: Bearer token header using `${{ Github Token }}` from `Set Github Info`.  
  - *Input:* From `File 2`.  
  - *Output:* JSON with reference info including commit SHA under `.json.object.sha`.  
  - *Edge Cases:*  
    - Invalid token or insufficient permissions → 401/403 errors.  
    - Invalid branch or repo names → 404 errors.  
    - Rate limiting by GitHub API.  
  - *Version Requirements:* n8n v0.174+ recommended for stable HTTP Request node.

- **Get base tree SHA**  
  - *Type & Role:* HTTP Request node — fetches commit details for the latest commit SHA from previous node using `/git/commits/{sha}` endpoint, to obtain base tree SHA used to build the new tree.  
  - *Config:*  
    - Method: GET  
    - URL uses expression `{{ $json.object.sha }}` from previous node’s output.  
    - Authorization header as before.  
  - *Input:* From `Get latest commit SHA`.  
  - *Output:* JSON with commit data including `.json.tree.sha` (base tree SHA).  
  - *Edge Cases:*  
    - Failure if previous SHA is invalid or missing.  
    - API errors as before.

---

#### 2.3 Git Tree Creation

**Overview:**  
This block creates a new Git tree object in GitHub, including the files to add or update, built on the base tree SHA obtained earlier. This step is critical to enable multiple files to be committed in one atomic operation.

**Nodes Involved:**  
- Create new tree  

**Node Details:**

- **Create new tree**  
  - *Type & Role:* HTTP Request node — sends a POST request to `/git/trees` endpoint to create a new tree that references the base tree and contains new/updated file blobs.  
  - *Config:*  
    - URL dynamically set using GitHub info variables.  
    - Method POST, Content-Type JSON.  
    - Body includes:  
      - `base_tree`: the SHA from `Get base tree SHA`.  
      - `tree[]`: array of file objects with fields:  
        - `path`: file path in repo (e.g., "public/file1.txt", "public/file2.txt")  
        - `mode`: "100644" (file mode for blobs)  
        - `type`: "blob"  
        - `content`: file content from `File 1` and `File 2` nodes via expressions.  
  - *Input:* From `Get base tree SHA`.  
  - *Output:* JSON object containing new tree SHA in `.json.sha`.  
  - *Edge Cases:*  
    - File paths must be valid in GitHub (e.g., no forbidden characters).  
    - Large file content could cause API rate limits or payload size issues.  
    - If content expressions fail or are empty, files could be missing or empty.  
    - Authorization errors if token scope insufficient.  
  - *Customization:* Additional files can be added by inserting more `tree[]` entries with proper naming and referencing other Set nodes.

---

#### 2.4 Commit Creation

**Overview:**  
After creating the new Git tree including all intended files, this block creates a new Git commit referencing that tree and the latest commit as the parent.

**Nodes Involved:**  
- Create commit  

**Node Details:**

- **Create commit**  
  - *Type & Role:* HTTP Request node — posts to `/git/commits` to create a commit linked to the new tree and previous commit(s).  
  - *Config:*  
    - URL constructed dynamically with GitHub info.  
    - Method: POST, JSON body specifying:  
      - `message`: Commit message from `Set Github Info`.  
      - `tree`: new tree SHA from `Create new tree`.  
      - `parents`: array with one element, the latest commit SHA from `Get latest commit SHA`.  
    - Authorization and Content-Type headers as usual.  
  - *Input:* From `Create new tree`.  
  - *Output:* Commit object JSON including new commit SHA (`.json.sha`).  
  - *Edge Cases:*  
    - Commit message empty or invalid → possible API rejection.  
    - Improper tree SHA or missing parent commit → errors.  
    - Insufficient scopes or expired token causes failures.

---

#### 2.5 Branch Update

**Overview:**  
The final block updates the branch reference to the new commit SHA, effectively making the pushed files visible in the repository.

**Nodes Involved:**  
- Update branch  

**Node Details:**

- **Update branch**  
  - *Type & Role:* HTTP Request node sends a PATCH request to `/git/refs/heads/{branch}` to update the branch pointer to the new commit.  
  - *Config:*  
    - URL dynamically built with GitHub info.  
    - Method: PATCH.  
    - JSON Body:  
      - `sha`: new commit SHA from `Create commit`.  
      - `force`: false (prevents forced update, safer commit history).  
    - Authorization and Content-Type headers set accordingly.  
  - *Input:* From `Create commit`.  
  - *Output:* Confirmation JSON from GitHub API.  
  - *Edge Cases:*  
    - If branch protected or permission issues, the update may be rejected.  
    - Conflicts if the branch advanced since initial commit fetch — workflow does not currently handle retries/rebase.  
    - Force update false to avoid overwriting divergent history.

---

### 3. Summary Table

| Node Name               | Node Type            | Functional Role                               | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                    |
|-------------------------|----------------------|-----------------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Start workflow manually                        |                             | Set Github Info             |                                                                                                                                               |
| Set Github Info         | Set                  | Define GitHub auth, repo info, branch, message | When clicking ‘Test workflow’ | File 1                     |                                                                                                                                               |
| File 1                  | Set                  | Define content for first file                   | Set Github Info             | File 2                      |                                                                                                                                               |
| File 2                  | Set                  | Define content for second file                  | File 1                     | Get latest commit SHA        |                                                                                                                                               |
| Get latest commit SHA   | HTTP Request         | Get latest commit SHA of branch                 | File 2                     | Get base tree SHA            |                                                                                                                                               |
| Get base tree SHA       | HTTP Request         | Get tree SHA corresponding to latest commit     | Get latest commit SHA       | Create new tree              |                                                                                                                                               |
| Create new tree         | HTTP Request         | Create git tree with the new files               | Get base tree SHA           | Create commit               |                                                                                                                                               |
| Create commit           | HTTP Request         | Create commit referencing new git tree and parent commit | Create new tree          | Update branch               |                                                                                                                                               |
| Update branch           | HTTP Request         | Update branch pointer to new commit              | Create commit              |                             |                                                                                                                                               |
| Sticky Note1            | Sticky Note          | Provides detailed introduction, setup steps, and usage instructions |                             |                             | ## Push Multiple Files to GitHub Repo  A streamlined workflow for uploading multiple files to a GitHub repository via the GitHub REST API. This solution addresses a significant limitation of the native GitHub n8n node, which supports only single-file uploads. This approach enables batch file operations, making it particularly valuable for automation scenarios that require simultaneous uploads of multiple files to your GitHub repositories. Setup Instructions: 1. Create a new GitHub Personal Access Token [here](https://github.com/settings/personal-access-tokens). In the "Repository access" section, select your repository and grant "Read and write" permissions under the "Contents" category.  2. Configure your GitHub information in the "Set GitHub Info" node.  3. Update the "Create New Tree" node with your filenames and content. You can add as many tree entries (files) as needed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: To manually trigger the workflow.

2. **Create Set node to define GitHub info**  
   - Name: "Set Github Info"  
   - Assign the following string fields:  
     - `Github Token`: Your GitHub Personal Access Token (PAT) with repo write permission.  
     - `Github Username`: Your GitHub username or organization name.  
     - `Github Repo`: Repository name where files will be pushed.  
     - `Github Branch`: Target branch name (e.g., "main").  
     - `Github Commit Update Message`: Commit message text (e.g., "Updating file1.txt and file2.txt").  
   - Connect the manual trigger output to this node.

3. **Create Set node for File 1 content**  
   - Name: "File 1"  
   - Assign field `content` with the desired text content for file1.txt.  
   - Connect from "Set Github Info".

4. **Create Set node for File 2 content**  
   - Name: "File 2"  
   - Assign field `content` with the desired text content for file2.txt.  
   - Connect from "File 1".

5. **Create HTTP Request node to get latest commit SHA**  
   - Name: "Get latest commit SHA"  
   - HTTP Method: GET  
   - URL: `https://api.github.com/repos/{{ $('Set Github Info').item.json['Github Username'] }}/{{ $('Set Github Info').item.json['Github Repo'] }}/git/refs/heads/{{ $('Set Github Info').item.json['Github Branch'] }}`  
   - Add Header: Authorization → `Bearer {{ $('Set Github Info').item.json['Github Token'] }}`  
   - Connect from "File 2".

6. **Create HTTP Request node to get base tree SHA**  
   - Name: "Get base tree SHA"  
   - HTTP Method: GET  
   - URL: `https://api.github.com/repos/{{ $('Set Github Info').item.json['Github Username'] }}/{{ $('Set Github Info').item.json['Github Repo'] }}/git/commits/{{ $json.object.sha }}`  
   - Header: Authorization as above  
   - Connect from "Get latest commit SHA".

7. **Create HTTP Request node to create new tree**  
   - Name: "Create new tree"  
   - HTTP Method: POST  
   - URL: `https://api.github.com/repos/{{ $('Set Github Info').item.json['Github Username'] }}/{{ $('Set Github Info').item.json['Github Repo'] }}/git/trees`  
   - Headers:  
     - Authorization: Bearer token as before  
     - Content-Type: application/json  
   - Body Parameters (as key-value pairs):  
     - base_tree: `={{ $json["tree"]["sha"] }}` (expression referencing output from "Get base tree SHA")  
     - Multiple `tree` entries representing files:  
       - `tree[0].path`: "public/file1.txt"  
       - `tree[0].mode`: "100644"  
       - `tree[0].type`: "blob"  
       - `tree[0].content`: `={{ $('File 1').item.json.content }}`  
       - `tree[1].path`: "public/file2.txt"  
       - `tree[1].mode`: "100644"  
       - `tree[1].type`: "blob"  
       - `tree[1].content`: `={{ $('File 2').item.json.content }}`  
   - Connect from "Get base tree SHA".

8. **Create HTTP Request node to create commit**  
   - Name: "Create commit"  
   - HTTP Method: POST  
   - URL: `https://api.github.com/repos/{{ $('Set Github Info').item.json['Github Username'] }}/{{ $('Set Github Info').item.json['Github Repo'] }}/git/commits`  
   - Headers: Authorization + Content-Type (application/json)  
   - Body (JSON specified):  
   ```json
   {
     "message": "{{ $('Set Github Info').item.json['Github Commit Update Message'] }}",
     "tree": "{{ $json.sha }}",
     "parents": ["{{ $('Get latest commit SHA').item.json.object.sha }}"]
   }
   ```  
   - Connect from "Create new tree".

9. **Create HTTP Request node to update branch**  
   - Name: "Update branch"  
   - HTTP Method: PATCH  
   - URL: `https://api.github.com/repos/{{ $('Set Github Info').item.json['Github Username'] }}/{{ $('Set Github Info').item.json['Github Repo'] }}/git/refs/heads/{{ $('Set Github Info').item.json['Github Branch'] }}`  
   - Headers: Authorization + Content-Type (application/json)  
   - Body (JSON):  
   ```json
   {
     "sha": "{{ $json.sha }}",
     "force": false
   }
   ```  
   - Connect from "Create commit".

10. **Final workflow test**  
    - Activate and run the workflow via the manual trigger.  
    - Verify on GitHub that the files `public/file1.txt` and `public/file2.txt` are created/updated with provided content in the specified branch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| To create a GitHub Personal Access Token with appropriate permissions, navigate to GitHub Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens.     | https://github.com/settings/personal-access-tokens                                             |
| This workflow abstracts complex GitHub Git Data API calls to enable atomic multi-file uploads, which is not natively supported by the GitHub node in n8n.                 | Workflow description and use cases in documentation above.                                     |
| Modify the "Create new tree" node to add or remove files by adjusting the `tree[]` array with file paths and contents accordingly.                                         | Dynamic customization guidance section.                                                        |
| The workflow assumes familiarity with Git concepts (commits, trees, refs), and requires a GitHub repository with write access and a valid branch.                         | User prerequisites.                                                                            |
| Sticky note node provides detailed setup instructions and environments info embedded within the workflow for user reference.                                              | Sticky Note1 node content in the workflow.                                                    |

---

This reference document fully covers the workflow structure and configuration to enable advanced users or automation agents to understand, reproduce, and extend the GitHub multi-file push process in n8n.