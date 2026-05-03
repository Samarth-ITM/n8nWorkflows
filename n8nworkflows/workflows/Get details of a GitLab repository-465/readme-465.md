Get details of a GitLab repository

https://n8nworkflows.xyz/workflows/get-details-of-a-gitlab-repository-465


# Get details of a GitLab repository

### 1. Workflow Overview

This workflow is designed to retrieve detailed information about a specific GitLab repository. It is intended for users who want to quickly fetch repository metadata from GitLab by manually triggering the workflow. The workflow primarily includes two logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 GitLab Repository Retrieval:** Fetching repository details from GitLab via the GitLab API node.

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, allowing the user to trigger the process on-demand.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Role:** Starts the workflow when the user manually clicks "Execute" in n8n.  
  - **Configuration:** Default manual trigger with no additional parameters.  
  - **Input:** None (trigger node).  
  - **Output:** Passes an empty data object to the next node to initiate processing.  
  - **Version:** Compatible with n8n v1 and later.  
  - **Potential Failures:** None expected; user must manually activate.  
  - **Sub-workflow:** None.

#### 1.2 GitLab Repository Retrieval

- **Overview:**  
  This block connects to GitLab using stored credentials to fetch details for a specified repository in a given owner’s namespace.

- **Nodes Involved:**  
  - Gitlab

- **Node Details:**  
  - **Name:** Gitlab  
  - **Type:** GitLab API node  
  - **Role:** Performs an API call to GitLab to retrieve repository metadata.  
  - **Configuration:**  
    - Owner (Namespace/User): `shaligramshraddha`  
    - Resource: `repository`  
    - Operation: `get`  
    - Repository Name: `trial`  
    - Credentials: Uses a pre-configured GitLab API credential named `new`.  
  - **Input:** Receives trigger from manual trigger node.  
  - **Output:** Outputs detailed repository information including metadata such as description, visibility, creation date, etc.  
  - **Version:** Requires GitLab API node version compatible with the GitLab API v4.  
  - **Potential Failures:**  
    - Authentication errors if credentials are invalid or expired.  
    - Repository not found if the owner or repository name is incorrect.  
    - API rate limiting by GitLab if excessive requests are made in a short time.  
    - Network timeouts or connectivity issues.  
  - **Sub-workflow:** None.

### 3. Summary Table

| Node Name          | Node Type          | Functional Role                    | Input Node(s)       | Output Node(s) | Sticky Note                         |
|--------------------|--------------------|----------------------------------|---------------------|----------------|-----------------------------------|
| On clicking 'execute' | Manual Trigger     | Initiates the workflow manually  | —                   | Gitlab         |                                   |
| Gitlab             | GitLab API         | Fetches GitLab repository details| On clicking 'execute'| —              | Requires valid GitLab API credentials |

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a new node of type **Manual Trigger**.  
   - Leave all settings as default since no inputs are needed.  
   - Name it `On clicking 'execute'`.

2. **Create GitLab Node:**  
   - Add a new node of type **GitLab**.  
   - Configure with the following parameters:  
     - Owner: `shaligramshraddha`  
     - Resource: `repository`  
     - Operation: `get`  
     - Repository: `trial`  
   - Assign credentials: Create or select existing **GitLab API credential** (OAuth2 or Personal Access Token) with sufficient permissions to read repository data. Name the credential `new` or any preferred name and select it.  
   - Name the node `Gitlab`.

3. **Connect Nodes:**  
   - Link the output of the Manual Trigger (`On clicking 'execute'`) to the input of the `Gitlab` node.

4. **Save and Execute:**  
   - Save the workflow.  
   - Click "Execute Workflow" to manually start and retrieve repository data.

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| The GitLab node requires a valid API credential with read access to the repository data. Ensure token scopes are set properly. | GitLab Personal Access Token documentation: https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html |
| This simple workflow can be extended by adding error handling nodes or data processing nodes after the GitLab node. | n8n documentation: https://docs.n8n.io/nodes/n8n-nodes-base.gitlab/ |

---

This documentation provides a clear, reproducible guide suitable for advanced users and AI agents to understand, replicate, and extend the workflow for fetching GitLab repository details.