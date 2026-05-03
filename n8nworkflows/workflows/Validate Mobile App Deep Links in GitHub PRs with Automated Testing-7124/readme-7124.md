Validate Mobile App Deep Links in GitHub PRs with Automated Testing

https://n8nworkflows.xyz/workflows/validate-mobile-app-deep-links-in-github-prs-with-automated-testing-7124


# Validate Mobile App Deep Links in GitHub PRs with Automated Testing

### 1. Workflow Overview

This workflow automates the validation of mobile app deep links within GitHub Pull Requests (PRs) by running a custom validation script against the repository and commenting the results back on the PR. It is designed for mobile app developers and QA teams who want to enforce deep link correctness and routing integrity as part of their GitHub PR review process.

Logical blocks:

- **1.1 Input Reception:** Receives webhook events triggered by GitHub PR actions.
- **1.2 Configuration Setup:** Initializes workflow variables and parameters needed for script execution and GitHub API calls.
- **1.3 Deep Link Validation:** Clones the repository, runs a shell script to validate deep links from the Android manifest, and collects the output.
- **1.4 Result Formatting:** Parses the script output and formats it into a Markdown table for GitHub comments.
- **1.5 GitHub Commenting:** Posts the validation results as a comment on the PR.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming GitHub Pull Request webhook POST requests. It acts as the entry point to trigger the validation workflow on PR events.

- **Nodes Involved:**  
  - GitHub PR Webhook

- **Node Details:**  

  - **GitHub PR Webhook**  
    - Type: Webhook node  
    - Role: Receives HTTP POST requests from GitHub when a PR event occurs.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/validate-pr` (custom webhook endpoint)  
      - Response Mode: Last node output is returned as HTTP response.  
    - Key Variables: None; raw webhook payload is passed downstream.  
    - Input: External HTTP POST from GitHub  
    - Output: JSON payload containing PR data, passed to configuration node.  
    - Failures: Missing or invalid webhook payload, webhook authentication failure if GitHub secret is set outside this workflow.  
    - Version: Compatible with n8n webhook v1.

#### 2.2 Configuration Setup

- **Overview:**  
  Sets static and dynamic parameters required for the validation process, such as repository details, script and manifest paths, GitHub token, timeout, and branch name.

- **Nodes Involved:**  
  - CONFIG - Variables

- **Node Details:**  

  - **CONFIG - Variables**  
    - Type: Set node  
    - Role: Defines workflow parameters as variables accessible by subsequent nodes.  
    - Configuration:  
      - `pullRequestNumber`: Default 1 (expected to be overridden by webhook data in practice)  
      - `repoUrl`: Git repository URL (empty placeholder, expected to be set)  
      - `manifestPath`: Path to AndroidManifest.xml in the repo (`deeplink-validator-demo/AndroidManifest.xml`)  
      - `scriptPath`: Path to the validation shell script (`deeplink-validator-demo/validate-links.sh`)  
      - `commentMode`: Comment posting mode on GitHub (`replace`)  
      - `timeout`: Script execution timeout in seconds (`120`)  
      - `githubToken`: GitHub API token (empty placeholder)  
      - `branchName`: Git branch to clone (`main`)  
      - `repositoryFullName`: GitHub repository full name (`reponame`)  
    - Key Variables: All above as JSON keys for template literals.  
    - Input: Output of webhook node  
    - Output: JSON with variables set, passed to script execution.  
    - Failures: Missing or incorrect values, especially `repoUrl` or tokens, cause downstream failures.  
    - Version: Compatible with n8n Set node v1.

#### 2.3 Deep Link Validation

- **Overview:**  
  This block clones the repository to a temporary directory, locates the validation script and Android manifest file, executes the script, and captures its standard output for further processing.

- **Nodes Involved:**  
  - Run Validation Script

- **Node Details:**  

  - **Run Validation Script**  
    - Type: Execute Command node  
    - Role: Runs a shell script that validates deep links by parsing the Android manifest.  
    - Configuration:  
      - Shell script command template:  
        - Cleans `/tmp/validation` directory  
        - Clones the repo from `repoUrl` at specified `branchName` into `/tmp/validation`  
        - Sets `SCRIPT_PATH` and `MANIFEST_PATH` variables relative to cloned repo  
        - Checks existence and permission of the validation script  
        - Executes the validation script with manifest path argument  
        - Exits with error if any step fails  
      - Template expressions used for dynamic paths and URLs (`{{ $json.repoUrl }}`, etc.)  
    - Input: JSON with config variables  
    - Output: Standard output from script execution (expected CSV lines of deep link status)  
    - Failures: Git clone errors (invalid URL, auth failure), missing script or manifest, script execution errors, timeout.  
    - Version: Requires n8n Execute Command v1.  
    - Notes: Must run on n8n host with git, shell, and permissions to clone and execute scripts.

#### 2.4 Result Formatting

- **Overview:**  
  Parses the output from the validation script, which should be CSV lines of deep link and status, converts it into a Markdown table with status icons for GitHub comment.

- **Nodes Involved:**  
  - Format Markdown

- **Node Details:**  

  - **Format Markdown**  
    - Type: Function node (JavaScript)  
    - Role: Transforms raw script output into a readable Markdown table for GitHub comment.  
    - Configuration:  
      - Parses `stdout` string split by new lines  
      - For each line containing a comma, splits into URL and status  
      - Maps status "OK" to ✅ icon, others to ❌ icon  
      - Builds Markdown table string with headers and rows  
      - Returns JSON with `markdownResult` property containing the table  
    - Input: Output JSON from Execute Command node (`stdout`)  
    - Output: JSON with formatted Markdown string  
    - Failures: If script output format is invalid or empty, table will be incomplete or empty. Expression errors possible if `stdout` is missing.  
    - Version: Compatible with n8n Function node v1.

#### 2.5 GitHub Commenting

- **Overview:**  
  Posts the formatted validation report as a comment on the relevant GitHub Pull Request using the GitHub API.

- **Nodes Involved:**  
  - GitHub

- **Node Details:**  

  - **GitHub**  
    - Type: GitHub node  
    - Role: Creates or replaces a comment on the PR with the validation results.  
    - Configuration:  
      - Operation: `createComment` on an issue (PR)  
      - Owner and Repository extracted dynamically by splitting `repositoryFullName` variable  
      - Issue Number set from `pullRequestNumber` variable  
      - Body set to the formatted Markdown result  
      - Comment mode: `replace` (implied by variable, but actual replace logic depends on GitHub node config)  
    - Input: JSON with `markdownResult` from Function node and config variables  
    - Output: API response from GitHub comment creation  
    - Failures: Invalid GitHub token, permission denied, incorrect repo or PR number, network issues.  
    - Credentials: Requires GitHub API credentials with repo access.  
    - Version: Uses GitHub node v1.1.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                  | Input Node(s)          | Output Node(s)          | Sticky Note                       |
|---------------------|--------------------|--------------------------------|-----------------------|------------------------|---------------------------------|
| GitHub PR Webhook    | Webhook            | Receives GitHub PR webhook POST | -                     | CONFIG - Variables      |                                 |
| CONFIG - Variables   | Set                | Sets workflow parameters       | GitHub PR Webhook      | Run Validation Script   |                                 |
| Run Validation Script| Execute Command    | Clones repo & runs validation script | CONFIG - Variables     | Format Markdown         |                                 |
| Format Markdown      | Function           | Formats script output to Markdown | Run Validation Script  | GitHub                 |                                 |
| GitHub              | GitHub API         | Posts validation comment on PR | Format Markdown        | -                      |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `GitHub PR Webhook`  
   - Type: Webhook (n8n-nodes-base.webhook)  
   - HTTP Method: POST  
   - Path: `validate-pr`  
   - Response Mode: `lastNode`  
   - No authentication configured unless required by GitHub webhook secret.

2. **Create Set Node for Configuration:**  
   - Name: `CONFIG - Variables`  
   - Type: Set (n8n-nodes-base.set)  
   - Add variables:  
     - `pullRequestNumber`: Number, default 1 (to be overridden)  
     - `repoUrl`: String, set your Git repo HTTPS URL (e.g., `https://github.com/owner/repo.git`)  
     - `manifestPath`: String, path to AndroidManifest.xml (e.g., `deeplink-validator-demo/AndroidManifest.xml`)  
     - `scriptPath`: String, path to validation shell script (e.g., `deeplink-validator-demo/validate-links.sh`)  
     - `commentMode`: String, `replace`  
     - `timeout`: String or Number, e.g., `120` seconds  
     - `githubToken`: String, your GitHub personal access token or use GitHub credential node  
     - `branchName`: String, e.g., `main`  
     - `repositoryFullName`: String, e.g., `owner/repo`  
   - Connect `GitHub PR Webhook` output to this node’s input.

3. **Create Execute Command Node:**  
   - Name: `Run Validation Script`  
   - Type: Execute Command (n8n-nodes-base.executeCommand)  
   - Command: Use the provided shell script template with embedded expressions:  

     ```bash
     #!/bin/sh
     rm -rf /tmp/validation
     echo "🔄 Cloning repo: {{ $json.repoUrl }}"
     git clone --depth 1 --branch "{{ $json.branchName }}" "{{ $json.repoUrl }}" /tmp/validation || exit 1

     SCRIPT_PATH="/tmp/validation/{{ $json.scriptPath }}"
     MANIFEST_PATH="/tmp/validation/{{ $json.manifestPath }}"

     echo "📁 Validating script path:"
     ls -l "$SCRIPT_PATH" || exit 1

     chmod +x "$SCRIPT_PATH" || exit 1
     echo "🚀 Executing: $SCRIPT_PATH $MANIFEST_PATH"
     sh "$SCRIPT_PATH" "$MANIFEST_PATH" || {
       echo "❌ Script execution failed"
       exit 1
     }
     echo "✅ Script ran successfully"
     ```

   - Connect `CONFIG - Variables` output to this node’s input.

4. **Create Function Node to Format Output:**  
   - Name: `Format Markdown`  
   - Type: Function  
   - Code:  

     ```javascript
     const lines = $node["Run Validation Script"].json["stdout"].split("\n");
     let table = "| Deep Link | Status |\n|-----------|--------|\n";
     for (const line of lines) {
       if (!line.includes(",")) continue;
       const [url, status] = line.split(",");
       if (!url || !status) continue;
       const icon = status.trim() === "OK" ? "✅" : "❌";
       table += `| ${url.trim()} | ${icon} ${status.trim()} |\n`;
     }
     return [{ json: { markdownResult: table } }];
     ```

   - Connect `Run Validation Script` output to this node’s input.

5. **Create GitHub Node to Post Comment:**  
   - Name: `GitHub`  
   - Type: GitHub (n8n-nodes-base.github)  
   - Operation: `createComment`  
   - Parameters:  
     - Owner: Expression splitting `repositoryFullName` (e.g., `={{ $('CONFIG - Variables').item.json.repositoryFullName.split("/")[0] }}`)  
     - Repository: Expression splitting `repositoryFullName` (e.g., `={{ $('CONFIG - Variables').item.json.repositoryFullName.split("/")[1] }}`)  
     - Issue Number: Expression `={{ $('CONFIG - Variables').item.json.pullRequestNumber }}`  
     - Body: Expression `={{ $json.markdownResult }}`  
   - Credentials: Configure with a GitHub API credential with sufficient permissions (repo access to comment on PRs).  
   - Connect `Format Markdown` output to this node’s input.

6. **Connection Workflow:**  
   - Connect nodes in order:  
     `GitHub PR Webhook` → `CONFIG - Variables` → `Run Validation Script` → `Format Markdown` → `GitHub`

7. **Webhook Setup on GitHub:**  
   - Configure GitHub repository to send PR webhook events (Pull Request events) to the n8n webhook URL:  
     `https://<n8n-instance>/webhook/validate-pr`  
   - Ensure webhook secret is configured if needed, and the workflow is active.

8. **Credential Setup:**  
   - GitHub API credential with a token that can comment on PRs.  
   - Host running n8n must have git, bash/sh, and network access to clone repos.

9. **Optional:**  
   - Adjust `pullRequestNumber` and `repoUrl` dynamically by parsing the webhook payload within the workflow for automation rather than static values.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                              |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow requires the validation shell script (`validate-links.sh`) to be present in the repository path specified by `scriptPath`. | Shell script is custom and outside n8n, must be maintained. |
| GitHub webhook configuration must include PR events to trigger validation on PR creation or updates.          | GitHub webhook docs: https://docs.github.com/en/developers/webhooks-and-events/webhooks/webhook-events-and-payloads#pull_request |
| The workflow cleans `/tmp/validation` on each run and clones the repo freshly to avoid stale data.             | Requires write permissions and disk space on host.           |
| The GitHub node uses `createComment` operation; to replace comments, additional logic or GitHub Apps may be required. | See GitHub API for comment editing: https://docs.github.com/en/rest/issues/comments#update-a-issue-comment |
| For security, the GitHub token should have minimum scopes required (e.g., `repo`).                             | Use GitHub personal access tokens with least privilege.      |
| Ensure the host environment running n8n has `git` and shell access for Execute Command node to work correctly. |                                                                |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected elements. All data processed is legal and public.