Automated Repository Migration from GitLab Groups to Gitea Organizations

https://n8nworkflows.xyz/workflows/automated-repository-migration-from-gitlab-groups-to-gitea-organizations-4947


# Automated Repository Migration from GitLab Groups to Gitea Organizations

### 1. Workflow Overview

This workflow automates the migration of repositories from GitLab groups to Gitea organizations. It is designed for DevOps teams or system administrators who want to transfer multiple projects hosted under a GitLab group into corresponding repositories within a Gitea organization, preserving issues, labels, milestones, releases, and wiki data.

The workflow is logically structured into these blocks:

- **1.1 Input Initialization:** Manual trigger and setup of essential configuration variables.
- **1.2 GitLab Projects Retrieval:** Fetching the list of repositories from a specified GitLab group.
- **1.3 Iterative Processing:** Looping over each GitLab project to process migration one by one.
- **1.4 Repository Existence Check on Gitea:** For each project, verifying if the repository already exists in Gitea.
- **1.5 Conditional Handling:** Based on existence check results, either migrating the repository or stopping on unexpected errors.
- **1.6 Repository Migration Execution:** Invoking the Gitea API to migrate the repository with preservation of metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

**Overview:**  
This block starts the workflow manually and sets up the necessary parameters such as access tokens and group/organization identifiers.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Setup (CHANGE ME)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters, triggers workflow execution on user command.  
  - Input: None  
  - Output: Triggers the "Setup (CHANGE ME)" node.  
  - Edge cases: None.  
  - Sub-workflow: None.

- **Setup (CHANGE ME)**  
  - Type: Set  
  - Role: Defines configuration variables for API authentication and target group/org names.  
  - Configuration: Sets three string variables:
    - `gitlabAccessToken`: GitLab personal access token.
    - `gitlabGroupPathName`: The GitLab group path name to fetch projects from.
    - `giteaOrganizationPathName`: The target Gitea organization path name for repo migration.  
  - Input: Triggered by manual start node.  
  - Output: Passes variables to the "Gitlab: Get projects" node.  
  - Edge cases: Ensure tokens and path names are accurate; invalid tokens will cause API errors downstream.  
  - Sub-workflow: None.

---

#### 1.2 GitLab Projects Retrieval

**Overview:**  
Fetches all projects under the specified GitLab group, handling pagination to ensure complete retrieval.

**Nodes Involved:**  
- Gitlab: Get projects

**Node Details:**  

- **Gitlab: Get projects**  
  - Type: HTTP Request  
  - Role: Calls GitLab API to list all projects in the specified group.  
  - Configuration:  
    - URL built dynamically with `gitlabGroupPathName` variable.  
    - Uses HTTP header authentication with provided GitLab token.  
    - Pagination enabled with page size of 20; continues fetching until all pages are retrieved.  
  - Input: Receives variables from "Setup (CHANGE ME)".  
  - Output: Sends array of projects to "Loop Over Items" node.  
  - Edge cases:  
    - Authentication failures if token invalid.  
    - API rate limits on GitLab.  
    - Pagination header missing or malformed response.  
  - Sub-workflow: None.

---

#### 1.3 Iterative Processing

**Overview:**  
Splits the array of GitLab projects into individual items to process each repository migration independently.

**Nodes Involved:**  
- Loop Over Items

**Node Details:**  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes one project at a time from the fetched list; enables controlled sequential processing.  
  - Configuration: Default options; no batch size specified which defaults to 1.  
  - Input: Array of projects from "Gitlab: Get projects" or from migration retries.  
  - Output: Individual project item passed downstream to "Gitea: Search for repo".  
  - Edge cases:  
    - Large project lists may extend processing time.  
    - Failure in one batch does not halt entire workflow due to error handling downstream.  
  - Sub-workflow: None.

---

#### 1.4 Repository Existence Check on Gitea

**Overview:**  
For each GitLab project, checks if the repository already exists in the target Gitea organization to avoid duplicate migration.

**Nodes Involved:**  
- Gitea: Search for repo

**Node Details:**  

- **Gitea: Search for repo**  
  - Type: HTTP Request  
  - Role: Queries Gitea API for the existence of the repo by organization and repo path (name).  
  - Configuration:  
    - URL dynamically built with `giteaOrganizationPathName` and current project path.  
    - Authentication via HTTP header with Gitea user token.  
    - On error: configured to continue with error output rather than stopping the workflow.  
  - Input: Single project item from "Loop Over Items".  
  - Output: Successful response routes back to "Loop Over Items" for next project; error responses forwarded to "Switch error codes".  
  - Edge cases:  
    - 404 means repo does not exist, triggering migration.  
    - Other error codes require special handling to avoid incorrect flow.  
    - Network or auth errors.  
  - Sub-workflow: None.

---

#### 1.5 Conditional Handling

**Overview:**  
Routes the workflow depending on the HTTP status code from the existence check: proceed with migration if repo not found (404), or stop with error otherwise.

**Nodes Involved:**  
- Switch error codes  
- Stop and Error

**Node Details:**  

- **Switch error codes**  
  - Type: Switch  
  - Role: Branches on the `error.status` field in the JSON response of the previous node.  
  - Configuration:  
    - Checks specifically for HTTP 404 error code.  
    - Routes 404 to "Gitea: Migrate repo".  
    - All other errors routed to "Stop and Error".  
  - Input: Error output from "Gitea: Search for repo".  
  - Output:  
    - 404 → "Gitea: Migrate repo"  
    - Others → "Stop and Error"  
  - Edge cases:  
    - Unexpected or missing error codes can cause fallback route to stop workflow.  
  - Sub-workflow: None.

- **Stop and Error**  
  - Type: Stop and Error  
  - Role: Terminates workflow with an error message on unexpected status codes.  
  - Configuration:  
    - Error message: "Unexpected error code when checking if Gitea repo already exists".  
  - Input: From "Switch error codes" node on non-404 errors.  
  - Output: None (workflow stops).  
  - Edge cases: None.  
  - Sub-workflow: None.

---

#### 1.6 Repository Migration Execution

**Overview:**  
Executes the migration by instructing Gitea to import the repository from GitLab, preserving repository metadata.

**Nodes Involved:**  
- Gitea: Migrate repo

**Node Details:**  

- **Gitea: Migrate repo**  
  - Type: HTTP Request  
  - Role: Calls Gitea API to import the GitLab repository into the organization.  
  - Configuration:  
    - URL: `https://gitea.example.com/api/v1/repos/migrate`  
    - Method: POST  
    - Body: JSON constructed dynamically from current project metadata and setup variables including:
      - `auth_token`: GitLab access token (used by Gitea for migration).  
      - `clone_addr`: GitLab repo URL.  
      - `description`, `repo_name`, and other repo attributes.  
      - Flags to preserve issues, labels, milestones, LFS, pull requests, releases, and wiki.  
      - `private` set to true.  
      - `repo_owner`: Gitea organization path name.  
      - `service`: fixed as "gitlab".  
    - Authentication: HTTP header authentication with Gitea user credentials.  
  - Input: From the "Switch error codes" node on 404 (repo not found).  
  - Output: Routes back to "Loop Over Items" to continue processing next project.  
  - Edge cases:  
    - API authentication failure.  
    - Invalid or inaccessible GitLab repo URL.  
    - Rate limiting or API errors from Gitea.  
  - Sub-workflow: None.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                | Input Node(s)                   | Output Node(s)                | Sticky Note               |
|----------------------------|---------------------|-----------------------------------------------|--------------------------------|------------------------------|---------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Entry point to start workflow manually         | None                           | Setup (CHANGE ME)             |                           |
| Setup (CHANGE ME)           | Set                 | Sets configuration variables for tokens and group/org names | When clicking ‘Execute workflow’ | Gitlab: Get projects          |                           |
| Gitlab: Get projects        | HTTP Request        | Fetches all projects in GitLab group with pagination | Setup (CHANGE ME)              | Loop Over Items               |                           |
| Loop Over Items             | SplitInBatches      | Iterates over each project individually         | Gitlab: Get projects, Gitea: Migrate repo | Gitea: Search for repo       |                           |
| Gitea: Search for repo      | HTTP Request        | Checks if repository exists in Gitea org       | Loop Over Items                | Loop Over Items, Switch error codes |                           |
| Switch error codes          | Switch              | Routes based on error code from repo search    | Gitea: Search for repo (error) | Gitea: Migrate repo, Stop and Error |                           |
| Stop and Error              | Stop and Error      | Stops workflow on unexpected error codes       | Switch error codes             | None                         |                           |
| Gitea: Migrate repo         | HTTP Request        | Calls Gitea API to migrate the repository       | Switch error codes (404 path)  | Loop Over Items               |                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Start workflow manually.

2. **Create a Set Node**  
   - Name: `Setup (CHANGE ME)`  
   - Add three string parameters:  
     - `gitlabAccessToken` (GitLab personal access token)  
     - `gitlabGroupPathName` (GitLab group path)  
     - `giteaOrganizationPathName` (Gitea organization path)  
   - Connect output of Manual Trigger to this node.

3. **Create an HTTP Request Node**  
   - Name: `Gitlab: Get projects`  
   - Method: GET  
   - URL: `https://gitlab.com/api/v4/groups/{{ $json.gitlabGroupPathName }}/projects`  
   - Query Parameter: `per_page=20`  
   - Enable Pagination: Yes, use response headers `x-next-page`, `x-page`, `x-total-pages` to paginate until complete.  
   - Authentication: HTTP Header Auth with GitLab token (`gitlabAccessToken`).  
   - Connect output of `Setup (CHANGE ME)` to this node.

4. **Create a SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Default batch size (1) to process each project one by one.  
   - Connect output of `Gitlab: Get projects` to this node.

5. **Create an HTTP Request Node**  
   - Name: `Gitea: Search for repo`  
   - Method: GET  
   - URL: `https://gitea.example.com/api/v1/repos/{{ $json.giteaOrganizationPathName }}/{{ $json.path }}`  
   - Authentication: HTTP Header Auth with Gitea user token.  
   - Set onError to "Continue on Error" to handle 404s gracefully.  
   - Connect output of `Loop Over Items` to this node.

6. **Create a Switch Node**  
   - Name: `Switch error codes`  
   - Condition: Check if `{{$json.error.status}} == 404`  
   - Outputs:  
     - Output 1 (named 404): Connect to `Gitea: Migrate repo`  
     - Default (fallback): Connect to `Stop and Error`  
   - Connect error output of `Gitea: Search for repo` to this node.

7. **Create a Stop and Error Node**  
   - Name: `Stop and Error`  
   - Error message: `"Unexpected error code when checking if Gitea repo already exists"`  
   - Connect fallback output of `Switch error codes` to this node.

8. **Create an HTTP Request Node**  
   - Name: `Gitea: Migrate repo`  
   - Method: POST  
   - URL: `https://gitea.example.com/api/v1/repos/migrate`  
   - Authentication: HTTP Header Auth with Gitea user token.  
   - Body Type: JSON  
   - Body (JSON) parameters:  
     ```json
     {
       "auth_token": "{{ $('Setup (CHANGE ME)').item.json.gitlabAccessToken }}",
       "clone_addr": "{{ $('Loop Over Items').item.json.http_url_to_repo }}",
       "description": "{{ $('Loop Over Items').item.json.description }}",
       "issues": true,
       "labels": true,
       "lfs": true,
       "milestones": true,
       "mirror": false,
       "private": true,
       "pull_requests": true,
       "releases": true,
       "repo_name": "{{ $('Loop Over Items').item.json.path }}",
       "repo_owner": "{{ $('Setup (CHANGE ME)').item.json.giteaOrganizationPathName }}",
       "service": "gitlab",
       "wiki": true
     }
     ```  
   - Connect output of `Switch error codes` (404 path) to this node.  
   - Connect output of this node back to `Loop Over Items` to continue processing.

9. **Configure Credentials**  
   - Create HTTP Header Auth credentials for GitLab using `gitlabAccessToken`.  
   - Create HTTP Header Auth credentials for Gitea with valid user token.  
   - Attach appropriate credentials to respective HTTP Request nodes.

10. **Save and Execute**  
    - Trigger workflow manually.  
    - Monitor execution for errors or blocked migrations.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow requires valid API tokens for both GitLab and Gitea with appropriate scopes enabled.  | Ensure tokens have access to group projects and repo creation permissions. |
| The GitLab API pagination uses response headers to fetch all projects; adjust `per_page` if needed. | GitLab API Docs: https://docs.gitlab.com/ee/api/    |
| The Gitea migration endpoint expects the `auth_token` as the GitLab token to fetch source repo.     | Gitea API Docs: https://try.gitea.io/api/swagger    |
| Error handling is designed to stop only on unexpected errors, allowing seamless migration if repo absent. |                                                        |
| Replace placeholder URLs and tokens before running.                                                 |                                                     |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.