CI Artifact Completeness Gate (Git Push, Sentry Artifact Verification, Commit)

https://n8nworkflows.xyz/workflows/ci-artifact-completeness-gate--git-push--sentry-artifact-verification--commit--11828


# CI Artifact Completeness Gate (Git Push, Sentry Artifact Verification, Commit)

### 1. Workflow Overview

This workflow, named **CI Artifact Completeness Gate**, is designed as a critical quality gate within a CI/CD pipeline. Its primary purpose is to automatically verify that all required build artifacts for a given release are uploaded and complete in Sentry before allowing code merges in GitHub. This ensures code is merged only when the associated debugging and symbol files are present, preventing incomplete builds from propagating.

The workflow is triggered by GitHub Push events and performs the following logical blocks:

- **1.1 GitHub Push Trigger:** Listens for pushes to a repository and extracts commit and repository details needed for verification.
- **1.2 Retrieve Sentry Releases:** Queries the Sentry API to list all releases for a given project.
- **1.3 Retrieve Artifacts for Release:** Fetches all artifact files uploaded to Sentry for the specific release version associated with the commit.
- **1.4 Verify Artifacts Completeness:** Runs custom JavaScript logic to ensure required artifacts are present (prioritizing dSYM files or the combination of ProGuard and mapping.txt files).
- **1.5 Validate and Prepare Repository Data:** Checks verification result, prepares repository and commit data for status update.
- **1.6 Update GitHub Commit Status:** Posts a status update back to GitHub to indicate success or failure of the artifact completeness gate.

Additionally, sticky notes within the workflow provide detailed explanations and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 GitHub Push Trigger

- **Overview:**  
  Initiates the workflow on every push event to the GitHub repository, capturing commit SHA and repository metadata used for downstream queries and status updates.

- **Nodes Involved:**  
  - `GithubPushTrigger`

- **Node Details:**  
  - **Type:** GitHub Trigger  
  - **Configuration:**  
    - Trigger on `push` events for a specified owner and repository.  
    - Uses a webhook with a unique webhookId.  
    - Credentials linked to a GitHub account with API access.  
  - **Key Expressions:**  
    - Extracts commit SHA (`body.after`) and repository full name (`body.repository.full_name`) from the push event payload.  
  - **Input/Output:**  
    - Input: External GitHub push webhook call.  
    - Output: JSON payload with commit and repo details.  
  - **Failures:**  
    - Possible webhook connection issues, authentication failure with GitHub API.  
    - Missing or malformed push event payload may cause failures downstream.  
  - **Sticky Note:**  
    - "GithubPushTrigger & Check Sentry Artifacts Releases" explaining initial trigger and Sentry releases retrieval.

#### 2.2 Retrieve Sentry Releases

- **Overview:**  
  Queries Sentry API to get a list of all releases for the specified organization and project, which is required to find artifact files for the release corresponding to the pushed commit.

- **Nodes Involved:**  
  - `Check Sentry Artifacts Releases`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - GET request to `https://sentry.io/api/0/projects/org_slug/proj_slug/releases/` (replace org_slug and proj_slug with actual values).  
    - Authentication via predefined Sentry credentials using a bearer token.  
    - Headers include `Authorization: Bearer YOUR_TOKEN_HERE`.  
  - **Key Expressions:**  
    - URL is static but must be customized with organization and project slugs.  
  - **Input/Output:**  
    - Input: Trigger output (start of workflow).  
    - Output: Array of releases from Sentry API.  
  - **Failures:**  
    - Authentication errors with Sentry API.  
    - Network timeouts or invalid URL.  
    - Empty or malformed responses.  
  - **Sticky Note:**  
    - Describes the purpose of this step in the workflow.

#### 2.3 Retrieve Artifacts for Release

- **Overview:**  
  Fetches the list of uploaded artifact files for the specific release version identified by the commit from the Sentry API.

- **Nodes Involved:**  
  - `Check Sentry Artifacts files`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - GET request to dynamic URL: `https://sentry.io/api/0/projects/org_slug/proj_slug/releases/{{ $json.version }}/files/`  
    - Uses the release version interpolated from the previous node’s JSON output.  
    - Authentication same as previous Sentry node.  
  - **Key Expressions:**  
    - URL dynamically builds on release version using `{{ $json.version }}` expression.  
  - **Input/Output:**  
    - Input: Release list data.  
    - Output: Array of artifacts (files) for the release.  
  - **Failures:**  
    - If release version is missing or incorrect, request will fail or return empty.  
    - Authentication and network issues as before.  
  - **Sticky Note:**  
    - Describes the retrieval of artifact files and relationship with verification.

#### 2.4 Verify Artifacts Completeness

- **Overview:**  
  Custom JavaScript logic to check completeness of the release artifacts. It verifies presence of dSYM files as priority, or alternatively both ProGuard and mapping.txt files.

- **Nodes Involved:**  
  - `Verify Artifacts`

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Processes input array of artifact files.  
    - Checks filenames for:  
      - `.dSYM` files (case insensitive)  
      - `proguard.txt` and `mapping.txt` files.  
    - Logic:  
      - Success if dSYM found.  
      - Else success if both proguard.txt and mapping.txt found.  
      - Else failure with description of missing files.  
  - **Key Expressions:**  
    - Uses regex tests on filenames to identify artifacts.  
    - Returns an object with `status` (success/failure) and `description`.  
  - **Input/Output:**  
    - Input: Array of artifact file JSON from previous node.  
    - Output: Verification result JSON.  
  - **Failures:**  
    - No input or empty array causes failure with descriptive message.  
    - Expression errors if input structure unexpected.  
  - **Sticky Note:**  
    - Explains the verification logic and artifact requirements.

#### 2.5 Validate and Prepare Repository Data

- **Overview:**  
  Checks the verification result status; if successful, it prepares the repository commit SHA and repo name for the status update. If failure, the workflow stops here.

- **Nodes Involved:**  
  - `Artifacts Validation and Get Repository Data`

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Extracts commit SHA and repo full name from the initial GitHub push trigger node.  
    - Checks if verification status is 'success'.  
    - If success, returns combined object with input data and repository details.  
    - If failure, returns empty to stop workflow.  
  - **Key Expressions:**  
    - Accesses previous nodes’ JSON using `$node` references.  
  - **Input/Output:**  
    - Input: Verification result and initial trigger data.  
    - Output: Repository and commit data if success; empty otherwise.  
  - **Failures:**  
    - Missing or malformed data may cause expression errors.  
  - **Sticky Note:**  
    - Describes enforcement of gate and continuation logic.

#### 2.6 Update GitHub Commit Status

- **Overview:**  
  Sends a POST request to GitHub API to update the commit status to success, allowing the pull request to display the status badge indicating artifact verification passed.

- **Nodes Involved:**  
  - `Update Status`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - POST request to: `https://api.github.com/repos/{{ $json.repoDetail.repoFullName }}/statuses/{{ $json.repoDetail.repoCommitSha }}`  
    - Body parameters include:  
      - `state`: `"success"` (hardcoded)  
      - `description`: `"Artifacts successfully verified."`  
    - Uses GitHub API credentials with appropriate OAuth token.  
  - **Key Expressions:**  
    - Dynamic URL built from `repoFullName` and `repoCommitSha`.  
  - **Input/Output:**  
    - Input: Repository and commit data from validation node.  
    - Output: Response from GitHub API (status update confirmation).  
  - **Failures:**  
    - Authentication failure, invalid commit SHA, or network errors.  
  - **Sticky Note:**  
    - Explains the final step confirming artifact gate success on GitHub.

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                          | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                                                           |
|-------------------------------------|---------------------|----------------------------------------|----------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| GithubPushTrigger                   | GitHub Trigger      | Start workflow on GitHub push event    | (Webhook)                        | Check Sentry Artifacts Releases     | "GithubPushTrigger & Check Sentry Artifacts Releases" explains initial trigger and Sentry releases retrieval                        |
| Check Sentry Artifacts Releases      | HTTP Request        | Retrieve Sentry releases list           | GithubPushTrigger                | Check Sentry Artifacts files        | Describes purpose of retrieving Sentry release metadata                                                                              |
| Check Sentry Artifacts files         | HTTP Request        | Retrieve artifacts file list for release| Check Sentry Artifacts Releases | Verify Artifacts                   | Explains retrieval of artifact files and relation to verification                                                                    |
| Verify Artifacts                    | Code                | Verify artifact completeness logic      | Check Sentry Artifacts files     | Artifacts Validation and Get Repository Data | Describes verification logic checking for dSYM or ProGuard+mapping files                                                            |
| Artifacts Validation and Get Repository Data | Code         | Validate status and prepare repo data   | Verify Artifacts                | Update Status                      | Explains enforcement of CI gate and passing commit data for status update                                                           |
| Update Status                      | HTTP Request        | Update GitHub commit status to success  | Artifacts Validation and Get Repository Data | (End)                        | Details posting success status to GitHub commit to allow PR merge                                                                  |
| Sticky Note10                      | Sticky Note         | Documentation and setup instructions   | (No input)                      | (No output)                      | "How It Works" overview and setup steps                                                                                             |
| Sticky Note3                       | Sticky Note         | Documentation for initial trigger & releases | (No input)                      | (No output)                      | Explains GitHub trigger and Sentry release retrieval                                                                                 |
| Sticky Note                       | Sticky Note          | Documentation for artifact retrieval & verification | (No input)                      | (No output)                      | Describes artifact retrieval and verification logic                                                                                 |
| Sticky Note2                      | Sticky Note          | Documentation for validation & status update | (No input)                      | (No output)                      | Explains validation enforcement and GitHub status update                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a GitHub Push Trigger Node**  
   - Type: GitHub Trigger  
   - Configure to listen for `push` events on the target repository and owner.  
   - Set up webhook in GitHub to call n8n webhook URL.  
   - Provide GitHub API credentials with repository access.

2. **Add HTTP Request Node: Check Sentry Artifacts Releases**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://sentry.io/api/0/projects/org_slug/proj_slug/releases/` (replace `org_slug` and `proj_slug` with your actual Sentry organization and project slug)  
   - Authentication: Predefined Credential with Sentry API token (Bearer token).  
   - Headers: Add `Authorization: Bearer YOUR_TOKEN_HERE` (use credential).  
   - Connect input from GitHub Push Trigger node.

3. **Add HTTP Request Node: Check Sentry Artifacts files**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://sentry.io/api/0/projects/org_slug/proj_slug/releases/{{ $json.version }}/files/`  
     - Use expression to interpolate release version dynamically from previous response.  
   - Authentication: Same Sentry API credentials.  
   - Connect input from the previous Sentry releases node.

4. **Add Code Node: Verify Artifacts**  
   - Type: Code  
   - JavaScript logic to process the array of artifact files:  
     - Check if files include any `.dSYM` file (priority).  
     - Else check if both `proguard.txt` and `mapping.txt` files exist (fallback).  
     - Return JSON object with `status: "success"` or `"failure"` and descriptive message.  
   - Connect input from the Check Sentry Artifacts files node.

5. **Add Code Node: Artifacts Validation and Get Repository Data**  
   - Type: Code  
   - Extract from `GithubPushTrigger` node the commit SHA and repo full name (`body.after` and `body.repository.full_name`).  
   - Check if the verification status from the previous code node is `"success"`.  
     - If yes, return combined object with `inputData` and `repoDetail` (commit SHA and repo name).  
     - If no, return empty to stop execution.  
   - Connect input from Verify Artifacts node.

6. **Add HTTP Request Node: Update Status**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.github.com/repos/{{ $json.repoDetail.repoFullName }}/statuses/{{ $json.repoDetail.repoCommitSha }}`  
     - Use expression to insert repo full name and commit SHA from previous node.  
   - Body Parameters:  
     - `state`: `"success"`  
     - `description`: `"Artifacts successfully verified."`  
   - Authentication: Use GitHub API credentials.  
   - Connect input from Artifacts Validation and Get Repository Data node.

7. **Optionally add Sticky Note nodes** for documentation inline in the workflow about each stage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow functions as a critical CI/CD Quality Gate, automatically verifying that all essential build artifacts are uploaded to Sentry before allowing code merges. It requires GitHub and Sentry API credentials set up properly. | Sticky Note10 content                                                                                              |
| Update the Sentry API URLs in the HTTP Request nodes with your organization and project slugs. Make sure to use a valid bearer token credential for Sentry API.                                                              | Sticky Note10 Setup Steps                                                                                           |
| The final GitHub Commit Status update allows the Pull Request to show a green checkmark, signaling successful artifact verification.                                                                                        | Sticky Note2                                                                                                       |
| The verification logic prioritizes dSYM files for iOS/macOS debug symbols, or alternatively checks for both ProGuard and mapping.txt files for Android builds.                                                                | Sticky Note                                                                                                        |
| GitHub Push Trigger node requires a webhook registered in GitHub repository settings. Credentials must have appropriate access rights to read repository and post statuses.                                                  | GitHub documentation: https://docs.github.com/en/developers/webhooks-and-events/webhooks/about-webhooks            |
| Sentry API documentation for Releases and Files endpoints: https://docs.sentry.io/api/releases/                                                                                                                               | Sentry API docs                                                                                                    |

---

**Disclaimer:** The provided content is derived solely from an automated n8n workflow and adheres strictly to content policies. All processed data is lawful and publicly accessible.