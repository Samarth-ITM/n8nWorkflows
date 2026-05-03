Automatic Self-Hosted Application Updates with Coolify Deployments

https://n8nworkflows.xyz/workflows/automatic-self-hosted-application-updates-with-coolify-deployments-8118


# Automatic Self-Hosted Application Updates with Coolify Deployments

### 1. Workflow Overview

This n8n workflow automates the update process for a self-hosted n8n instance by checking its current version and comparing it against the latest release available on GitHub. If a newer version is detected, it triggers a deployment update via Coolify (or a similar deployment service) to upgrade the instance automatically.

**Target Use Cases:**  
- Self-hosted n8n users wanting automated version checks and zero-touch deployment updates.  
- DevOps teams managing n8n instances with Coolify or compatible deployment APIs.  
- Scheduled or manual update triggers to maintain up-to-date automation environments.

**Logical Blocks:**

- **1.1 Input Triggers:** Manual execution or scheduled timer to start the version check.  
- **1.2 Version Fetching:** HTTP requests to n8n instance settings and GitHub API for latest release info.  
- **1.3 Data Merge & Normalization:** SQL merge and variable setting to unify version data for comparison.  
- **1.4 Version Comparison:** Decision node to evaluate if an update is required.  
- **1.5 Deployment Trigger:** HTTP request to Coolify API to restart or deploy the updated n8n image.  
- **1.6 No Operation:** Placeholder node for the case when no update is needed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Triggers

**Overview:**  
This block initiates the workflow either manually or on a scheduled interval.

**Nodes Involved:**  
- Schedule every time you want start the update check (Schedule Trigger)  
- When clicking ‘Execute workflow’ (Manual Trigger)

**Node Details:**

- **Schedule every time you want start the update check**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow at configured intervals (default empty interval, user to set e.g., every 6 hours).  
  - Configuration: Interval rule set via UI; no inputs; outputs trigger signal to downstream nodes.  
  - Edge Cases: Misconfigured intervals may cause missed or overly frequent runs.

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows immediate manual initiation of the workflow for testing or ad-hoc checks.  
  - Configuration: Default, no parameters.  
  - Edge Cases: None significant; manual initiation.

---

#### 1.2 Version Fetching

**Overview:**  
Fetches current n8n instance version and latest stable release version from GitHub.

**Nodes Involved:**  
- CALL YOUR N8N INSTANCE and retrieve the version (HTTP Request)  
- CALL GITHUB and retrieve the last n8n image stable version (HTTP Request)

**Node Details:**

- **CALL YOUR N8N INSTANCE and retrieve the version**  
  - Type: HTTP Request  
  - Role: Retrieves n8n instance settings from the REST API endpoint `/rest/settings`.  
  - Configuration:  
    - URL set to `https://yourn8ndomain/rest/settings` (replace with actual domain).  
    - No authentication by default, but can add if instance is secured.  
  - Output: JSON containing `data.versionCli` field among others.  
  - Input: Trigger from either manual or schedule trigger.  
  - Edge Cases:  
    - Connection errors if URL incorrect or instance down.  
    - Auth failures if secured endpoint not configured.  
    - Missing or malformed response may cause later failures.

- **CALL GITHUB and retrieve the last n8n image stable version**  
  - Type: HTTP Request  
  - Role: Fetches latest stable n8n release metadata from GitHub API.  
  - Configuration:  
    - URL: `https://api.github.com/repos/n8n-io/n8n/releases/latest`.  
    - Method: GET, no authentication required.  
  - Output: JSON containing release details, notably `name` field formatted like `n8n@1.60.0`.  
  - Input: Triggered alongside n8n instance version fetch.  
  - Edge Cases:  
    - API rate-limiting or downtime.  
    - Unexpected response structure changes.

---

#### 1.3 Data Merge & Normalization

**Overview:**  
Combines the two version data sources into a single dataset and normalizes the version strings for comparison.

**Nodes Involved:**  
- Merge bot data and pass just what we need (Merge)  
- Rename variable and unify format (Set)

**Node Details:**

- **Merge bot data and pass just what we need**  
  - Type: Merge (Combine by SQL)  
  - Role: Joins HTTP outputs into a single JSON object with two fields:  
    - `versionCli` from n8n instance data  
    - `name` from GitHub release data  
  - Configuration:  
    - SQL Query:  
      ```sql
      SELECT
        input1.data->versionCli AS versionCli,
        input2.name AS name
      FROM input1
      LEFT JOIN input2 ON 1=1;
      ```  
    - Requires exactly one item per input.  
  - Input: Outputs from both HTTP requests.  
  - Output: Single item with `versionCli` and `name` keys.  
  - Edge Cases:  
    - Empty inputs if HTTP requests fail.  
    - Multiple items per input cause query failure.

- **Rename variable and unify format**  
  - Type: Set  
  - Role: Extracts and normalizes the version strings into two string variables for comparison:  
    - `actualn8nversion` set to `versionCli` value.  
    - `newn8nversion` extracted by splitting the `name` field at '@' and taking the second part (e.g., `"n8n@1.60.0"` → `"1.60.0"`).  
  - Configuration:  
    - Assignments:  
      - `actualn8nversion = {{$json.versionCli}}`  
      - `newn8nversion = {{$json.name.split('@')[1]}}`  
  - Input: Output from Merge node.  
  - Output: JSON with `actualn8nversion` and `newn8nversion` as strings.  
  - Edge Cases:  
    - Missing or malformed `name` field causing split errors.  
    - Non-string types.

---

#### 1.4 Version Comparison

**Overview:**  
Determines if an update is necessary by comparing current and latest version strings.

**Nodes Involved:**  
- If the 2 versions is not the same make an update of image (IF)

**Node Details:**

- **If the 2 versions is not the same make an update of image**  
  - Type: IF (Condition)  
  - Role: Compares `actualn8nversion` and `newn8nversion` strings for inequality.  
  - Configuration:  
    - Condition: `{{$json.actualn8nversion}} !== {{$json.newn8nversion}}`  
    - Case sensitive: Enabled  
    - Strict type validation: Enabled  
  - Input: Output of Set node with normalized versions.  
  - Outputs:  
    - True branch if versions differ → trigger deployment.  
    - False branch if versions are equal → no operation.  
  - Edge Cases:  
    - Null or missing version variables.  
    - String comparison nuances (e.g., pre-release tags not handled).

---

#### 1.5 Deployment Trigger

**Overview:**  
Triggers the Coolify deployment service to restart/redeploy the updated n8n instance.

**Nodes Involved:**  
- CALL COOLIFY OR SIMILAR SERVICE FOR UPDATE IMAGE (HTTP Request)

**Node Details:**

- **CALL COOLIFY OR SIMILAR SERVICE FOR UPDATE IMAGE**  
  - Type: HTTP Request  
  - Role: Sends a restart/deploy request to the Coolify API endpoint for the target service.  
  - Configuration:  
    - URL: `https://app.coolify.io/api/v1/services/your-services-uuid/restart?latest=true` (replace domain and UUID).  
    - Authentication: HTTP Bearer token credential (valid API token required).  
    - Method: POST or GET (default GET in the node, verify API spec).  
    - Optional: Timeout and retry options configurable.  
  - Input: True branch output from IF node.  
  - Output: API response from Coolify.  
  - Edge Cases:  
    - Invalid or expired API token causing auth failure.  
    - Network errors or Coolify downtime.  
    - Incorrect service UUID or URL causing failed deployment.

---

#### 1.6 No Operation

**Overview:**  
Pass-through node to explicitly do nothing when no update is needed.

**Nodes Involved:**  
- No Operation, do nothing (NoOp)

**Node Details:**

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Placeholder for workflow path when no update is required.  
  - Input: False branch output from IF node.  
  - Output: None (end of path).  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name                                      | Node Type         | Functional Role                              | Input Node(s)                                          | Output Node(s)                                   | Sticky Note                                                                                                                     |
|------------------------------------------------|-------------------|----------------------------------------------|--------------------------------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Sticky • Overview                               | Sticky Note       | Workflow overview and instructions           | —                                                      | —                                               | ## Overview Purpose: Auto-check your n8n version and deploy an update via Coolify if a newer release is available. Pipeline: Manual/Schedule → HTTP n8n settings → HTTP GitHub latest → Merge (SQL) → Set (normalize) → IF compare → Deploy or No-Op. Before running: Replace the n8n URL, Coolify API URL and token, adjust schedule. |
| Sticky • Manual Trigger                         | Sticky Note       | Manual trigger instructions                   | —                                                      | —                                               | ## Manual Trigger Click **Execute workflow** to run tests immediately. No configuration required.                                |
| Sticky • Schedule                               | Sticky Note       | Schedule trigger instructions                 | —                                                      | —                                               | ## Schedule Trigger Runs the check on a timer. Set interval in Rule. Keep both HTTP nodes connected to Merge.                   |
| Sticky • N8N Settings                           | Sticky Note       | N8N settings HTTP request instructions        | —                                                      | —                                               | ## N8N Settings (HTTP Request) Fetch your instance settings. URL: `https://<your-n8n-domain>/rest/settings`. Output must include `data.versionCli`. Add auth if needed. |
| Sticky • GitHub Latest                          | Sticky Note       | GitHub latest release HTTP request instructions | —                                                      | —                                               | ## GitHub Latest (HTTP Request) Gets latest n8n release. URL: `https://api.github.com/repos/n8n-io/n8n/releases/latest`. Uses field `name` (e.g., `n8n@1.60.0`). No auth. |
| Sticky • Merge SQL                              | Sticky Note       | SQL merge instructions to combine inputs     | —                                                      | —                                               | ## Merge (SQL) Reduces inputs to 2 fields: `versionCli` and `name`. Query joins input1 and input2. Requires 1 item per input.    |
| Sticky • Set                                    | Sticky Note       | Set node instructions for normalization       | —                                                      | —                                               | ## Set (Normalize) Creates `actualn8nversion` and `newn8nversion` strings for comparison by extracting and splitting fields.    |
| Sticky • IF                                     | Sticky Note       | IF node instructions for version comparison | —                                                      | —                                               | ## IF (Compare) Condition: `actualn8nversion !== newn8nversion`. True → deploy update. False → do nothing. Case sensitive and strict type enabled. |
| Sticky • Deploy                                 | Sticky Note       | Deploy HTTP request instructions             | —                                                      | —                                               | ## Deploy (HTTP Request) Triggers Coolify deploy. URL: `https://<coolify-domain>/api/v1/services/<APP_UUID>/restart?latest=true`. Auth: HTTP Bearer token. Replace placeholders. |
| Schedule every time you want start the update check | Schedule Trigger  | Scheduled trigger to start update check      | —                                                      | CALL YOUR N8N INSTANCE and retrieve the version; CALL GITHUB and retrieve the last n8n image stable version |                                                                                                                                |
| When clicking ‘Execute workflow’                | Manual Trigger    | Manual trigger to start update check         | —                                                      | CALL GITHUB and retrieve the last n8n image stable version; CALL YOUR N8N INSTANCE and retrieve the version |                                                                                                                                |
| CALL YOUR N8N INSTANCE and retrieve the version | HTTP Request      | Fetch current n8n version from instance      | Schedule; Manual Trigger                               | Merge bot data and pass just what we need       |                                                                                                                                |
| CALL GITHUB and retrieve the last n8n image stable version | HTTP Request      | Fetch latest n8n release from GitHub         | Schedule; Manual Trigger                               | Merge bot data and pass just what we need       |                                                                                                                                |
| Merge bot data and pass just what we need       | Merge             | SQL merge to combine version data             | CALL YOUR N8N INSTANCE and retrieve the version; CALL GITHUB and retrieve the last n8n image stable version | Rename variable and unify format                 |                                                                                                                                |
| Rename variable and unify format                 | Set               | Normalize version strings for comparison      | Merge bot data and pass just what we need              | If the 2 versions is not the same make an update of image |                                                                                                                                |
| If the 2 versions is not the same make an update of image | IF                | Compare actual vs latest version               | Rename variable and unify format                        | CALL COOLIFY OR SIMILAR SERVICE FOR UPDATE IMAGE (true branch); No Operation, do nothing (false branch) |                                                                                                                                |
| CALL COOLIFY OR SIMILAR SERVICE FOR UPDATE IMAGE | HTTP Request      | Trigger deployment restart via Coolify API    | If the 2 versions is not the same make an update of image | —                                               |                                                                                                                                |
| No Operation, do nothing                         | NoOp              | End node if no update needed                   | If the 2 versions is not the same make an update of image | —                                               |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Schedule Trigger** node named `Schedule every time you want start the update check`.  
     - Configure interval as desired (e.g., every 6 hours). Leave empty to configure later.

   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
     - Default settings.

2. **Add HTTP Request Nodes to Fetch Versions:**

   - Add **HTTP Request** node `CALL YOUR N8N INSTANCE and retrieve the version`.  
     - URL: `https://<your-n8n-domain>/rest/settings` (replace with your instance domain).  
     - Method: GET.  
     - Authentication: None by default; add if your instance requires auth.  
     - Connect from both Schedule Trigger and Manual Trigger nodes directly.

   - Add **HTTP Request** node `CALL GITHUB and retrieve the last n8n image stable version`.  
     - URL: `https://api.github.com/repos/n8n-io/n8n/releases/latest`.  
     - Method: GET.  
     - Authentication: None.  
     - Connect from both Schedule Trigger and Manual Trigger nodes.

3. **Merge HTTP Request Outputs:**

   - Add **Merge** node named `Merge bot data and pass just what we need`.  
     - Mode: Combine by SQL.  
     - Query:  
       ```
       SELECT
         input1.data->versionCli AS versionCli,
         input2.name AS name
       FROM input1
       LEFT JOIN input2 ON 1=1;
       ```  
     - Connect the output of `CALL YOUR N8N INSTANCE and retrieve the version` to input 1 of Merge node.  
     - Connect the output of `CALL GITHUB and retrieve the last n8n image stable version` to input 2 of Merge node.

4. **Normalize Version Variables:**

   - Add **Set** node named `Rename variable and unify format`.  
     - Add two string fields:  
       - `actualn8nversion` with value `={{ $json.versionCli }}`  
       - `newn8nversion` with value `={{ $json.name.split('@')[1] }}`  
     - Connect output of Merge node to this Set node.

5. **Add Version Comparison IF Node:**

   - Add **IF** node named `If the 2 versions is not the same make an update of image`.  
     - Condition: String "not equals" comparison.  
     - Left value: `={{ $json.actualn8nversion }}`  
     - Right value: `={{ $json.newn8nversion }}`  
     - Enable case sensitive and strict type validation.  
     - Connect output of Set node to IF node.

6. **Add Deployment HTTP Request Node:**

   - Add **HTTP Request** node named `CALL COOLIFY OR SIMILAR SERVICE FOR UPDATE IMAGE`.  
     - URL: `https://<your-coolify-domain>/api/v1/services/<APP_UUID>/restart?latest=true` (replace placeholders).  
     - Authentication: HTTP Bearer Credential.  
       - Create and assign a credential with your Coolify API token.  
     - Method: GET or POST as required by Coolify API (default GET).  
     - Connect True output (update needed) of IF node to this node.

7. **Add No Operation Node:**

   - Add **NoOp** node named `No Operation, do nothing`.  
     - Connect False output (no update needed) of IF node to this node.

8. **Final Connections:**

   - From **Schedule Trigger** node, connect outputs to both HTTP request nodes (`CALL YOUR N8N INSTANCE...` and `CALL GITHUB...`).  
   - From **Manual Trigger** node, connect outputs similarly to both HTTP request nodes.  
   - Ensure both HTTP request nodes feed into the Merge node.  
   - Merge node connects to Set node; Set node to IF node; IF node branches to Deploy or NoOp node.

9. **Credential Setup:**

   - Create HTTP Bearer Auth credential with your Coolify API token for the deployment HTTP Request node.  
   - If your n8n instance requires authentication for `/rest/settings`, configure appropriate credentials for that HTTP Request node.

10. **Testing:**

    - Use Manual Trigger node to run workflow immediately for testing.  
    - Observe logs and outputs to verify versions are fetched and compared correctly.  
    - Confirm deployment triggers on version mismatch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Before running, replace placeholders for your n8n instance URL, Coolify API domain, UUID, and ensure valid API tokens are configured.       | Sticky • Overview                                                                                     |
| Adjust schedule intervals according to your maintenance window to avoid unnecessary deployment frequency.                                    | Sticky • Schedule                                                                                     |
| Coolify API token must have sufficient permissions to restart services.                                                                      | Sticky • Deploy                                                                                       |
| GitHub API is used unauthenticated; consider adding authentication if hitting rate limits.                                                  | Sticky • GitHub Latest                                                                                |
| The SQL merge requires exactly one item per input; ensure HTTP requests return single items to avoid failures.                              | Sticky • Merge SQL                                                                                   |
| Manual Trigger node allows immediate testing without waiting for scheduled execution.                                                        | Sticky • Manual Trigger                                                                               |
| The comparison is string-based and case sensitive; semantic versioning edge cases (e.g., pre-release versions) are not explicitly handled.  | Sticky • IF                                                                                          |
| Workflow inspired by community shared automation for self-hosted n8n auto-updates with Coolify deployments.                                  | —                                                                                                   |

---

**Disclaimer:**  
The text above is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.