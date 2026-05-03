Backup n8n Workflows to Bitbucket

https://n8nworkflows.xyz/workflows/backup-n8n-workflows-to-bitbucket-2787


# Backup n8n Workflows to Bitbucket

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows from a self-hosted n8n instance to a Bitbucket private repository. It is designed for users who want version control, daily automated backups, and easy recovery of their workflows without incurring additional costs. The workflow runs on a daily schedule (default 2 AM), fetches all workflows from n8n, compares each with its counterpart in Bitbucket, and uploads only new or changed workflows. It incorporates rate limiting logic to comply with Bitbucket API constraints and includes error handling with retries.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Initialization:** Starts the workflow daily and sets Bitbucket workspace and repository details.
- **1.2 Workflow Retrieval & Looping:** Fetches all workflows from n8n and processes them one by one.
- **1.3 Bitbucket Comparison:** Retrieves the existing workflow file from Bitbucket to check if it is new or changed.
- **1.4 Conditional Upload:** Decides whether to upload the workflow based on comparison results.
- **1.5 Upload & Rate Limiting:** Uploads workflows to Bitbucket and calculates wait times to avoid API rate limits.
- **1.6 Wait & Loop Continuation:** Waits dynamically before processing the next workflow to respect API limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Initialization

- **Overview:**  
  This block triggers the workflow daily at 2 AM and initializes the Bitbucket workspace and repository configuration.

- **Nodes Involved:**  
  - Run Daily at 2 AM  
  - Set Bitbucket Workspace & Repository

- **Node Details:**

  - **Run Daily at 2 AM**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow daily at 2 AM server time.  
    - Configuration: Set to trigger once daily at hour 2 (2 AM).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Set Bitbucket Workspace & Repository".  
    - Edge Cases: Timezone differences if server timezone is not UTC; ensure server time matches expected schedule.

  - **Set Bitbucket Workspace & Repository**  
    - Type: Set  
    - Role: Defines Bitbucket workspace and repository slugs as workflow variables.  
    - Configuration: Two string fields set:  
      - WorkspaceSlug: User-defined Bitbucket workspace identifier.  
      - RepositorySlug: User-defined Bitbucket repository name.  
    - Inputs: From "Run Daily at 2 AM".  
    - Outputs: Connects to "Get All Workflows".  
    - Edge Cases: Incorrect workspace or repository names will cause HTTP errors downstream.

#### 1.2 Workflow Retrieval & Looping

- **Overview:**  
  Retrieves all workflows from the n8n instance and processes them individually in batches.

- **Nodes Involved:**  
  - Get All Workflows  
  - Loop Workflows

- **Node Details:**

  - **Get All Workflows**  
    - Type: n8n API node  
    - Role: Fetches all workflows from the connected n8n instance using n8n API credentials.  
    - Configuration: No filters; retrieves all workflows.  
    - Credentials: Uses n8n API credential with API key authentication.  
    - Inputs: From "Set Bitbucket Workspace & Repository".  
    - Outputs: Connects to "Loop Workflows".  
    - Edge Cases: API authentication failure, network issues, or empty workflow list.

  - **Loop Workflows**  
    - Type: SplitInBatches  
    - Role: Processes workflows one at a time to manage API calls and rate limits.  
    - Configuration: Default batch size (1 item per batch).  
    - Inputs: From "Get All Workflows".  
    - Outputs: Connects to "Get Existing Worfklow from Bitbucket" and loops back after waiting.  
    - Edge Cases: Large numbers of workflows may increase total runtime; batch size can be adjusted.

#### 1.3 Bitbucket Comparison

- **Overview:**  
  For each workflow, attempts to retrieve the existing version from Bitbucket to determine if it is new or has changed.

- **Nodes Involved:**  
  - Get Existing Worfklow from Bitbucket

- **Node Details:**

  - **Get Existing Worfklow from Bitbucket**  
    - Type: HTTP Request  
    - Role: Fetches the existing workflow file from Bitbucket repository using Bitbucket API.  
    - Configuration:  
      - URL dynamically constructed using workspace and repository slugs, and sanitized workflow name as filename.  
      - Authentication: HTTP Basic Auth with Bitbucket app password credentials.  
      - Response: Full HTTP response including headers.  
      - On error: Continues workflow even if 404 (file not found) occurs.  
    - Inputs: From "Loop Workflows".  
    - Outputs: Connects to "New or Changed?".  
    - Edge Cases: 404 indicates new workflow; other HTTP errors may cause silent failures; network issues.

#### 1.4 Conditional Upload

- **Overview:**  
  Compares the retrieved Bitbucket workflow file with the current n8n workflow JSON to decide if upload is necessary.

- **Nodes Involved:**  
  - New or Changed?

- **Node Details:**

  - **New or Changed?**  
    - Type: If  
    - Role: Checks if the workflow is new (404 error) or content differs from Bitbucket version.  
    - Configuration:  
      - Condition 1: HTTP error status equals 404 (file not found).  
      - Condition 2: Bitbucket file content not equal to current workflow JSON stringified.  
      - Combines conditions with OR logic.  
      - Case insensitive, strict type validation.  
    - Inputs: From "Get Existing Worfklow from Bitbucket".  
    - Outputs:  
      - True branch: Connects to "Upload Workflow to Bitbucket".  
      - False branch: Connects to "Wait to Avoid Rate Limiting".  
    - Edge Cases: JSON string comparison may fail if formatting changes; consider normalization if needed.

#### 1.5 Upload & Rate Limiting

- **Overview:**  
  Uploads the new or changed workflow to Bitbucket and calculates wait time to respect Bitbucket API rate limits.

- **Nodes Involved:**  
  - Upload Workflow to Bitbucket  
  - Calculate Wait Time

- **Node Details:**

  - **Upload Workflow to Bitbucket**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Bitbucket API to upload the workflow JSON as a file.  
    - Configuration:  
      - URL: Bitbucket API endpoint for uploading files to repository src.  
      - Method: POST.  
      - Body parameters:  
        - message: Commit message includes workflow name and current timestamp (format yyyy-MM-dd HH:mm:ss).  
        - filename: Sanitized workflow name as filename with JSON content as value.  
      - Headers: Content-Type set to application/x-www-form-urlencoded.  
      - Authentication: HTTP Basic Auth with Bitbucket credentials.  
      - Response: Full HTTP response.  
    - Inputs: From "New or Changed?".  
    - Outputs: Connects to "Calculate Wait Time".  
    - Edge Cases: API errors, authentication failure, malformed request, network issues.

  - **Calculate Wait Time**  
    - Type: Code  
    - Role: Calculates dynamic wait time based on Bitbucket API rate limit headers from previous request.  
    - Configuration:  
      - Reads rate limit headers `x-ratelimit-remaining` and `x-ratelimit-reset`.  
      - If remaining requests < 100, calculates wait time to spread requests until reset plus 10% buffer.  
      - If remaining < 500, sets wait time to 2 seconds.  
      - Caps wait time at 30 seconds max.  
      - Defaults to 1 second if no header info available.  
    - Inputs: From "Upload Workflow to Bitbucket".  
    - Outputs: Connects to "Wait to Avoid Rate Limiting".  
    - Edge Cases: Missing or malformed headers; fallback to default wait time.

#### 1.6 Wait & Loop Continuation

- **Overview:**  
  Waits for the calculated time to avoid hitting API rate limits, then continues processing the next workflow.

- **Nodes Involved:**  
  - Wait to Avoid Rate Limiting

- **Node Details:**

  - **Wait to Avoid Rate Limiting**  
    - Type: Wait  
    - Role: Pauses workflow execution for the calculated wait time before continuing.  
    - Configuration:  
      - Wait time set dynamically from previous node output `waitTime` or defaults to 1 second.  
    - Inputs: From both "New or Changed?" (false branch) and "Calculate Wait Time".  
    - Outputs: Connects back to "Loop Workflows" to process next workflow.  
    - Edge Cases: Wait node failure is rare; long wait times may slow overall backup duration.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                   |
|----------------------------------|---------------------|----------------------------------------|--------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Run Daily at 2 AM                | Schedule Trigger    | Starts workflow daily at 2 AM           | None                           | Set Bitbucket Workspace & Repository |                                                                                              |
| Set Bitbucket Workspace & Repository | Set                 | Defines Bitbucket workspace & repo slugs | Run Daily at 2 AM              | Get All Workflows                  |                                                                                              |
| Get All Workflows                | n8n API             | Retrieves all workflows from n8n        | Set Bitbucket Workspace & Repository | Loop Workflows                    |                                                                                              |
| Loop Workflows                  | SplitInBatches      | Processes workflows one by one           | Get All Workflows              | Get Existing Worfklow from Bitbucket, Wait to Avoid Rate Limiting |                                                                                              |
| Get Existing Worfklow from Bitbucket | HTTP Request       | Retrieves workflow file from Bitbucket  | Loop Workflows                | New or Changed?                   |                                                                                              |
| New or Changed?                 | If                  | Checks if workflow is new or changed    | Get Existing Worfklow from Bitbucket | Upload Workflow to Bitbucket (true), Wait to Avoid Rate Limiting (false) |                                                                                              |
| Upload Workflow to Bitbucket    | HTTP Request        | Uploads new/changed workflow to Bitbucket | New or Changed? (true)         | Calculate Wait Time               |                                                                                              |
| Calculate Wait Time             | Code                 | Calculates wait time based on rate limits | Upload Workflow to Bitbucket   | Wait to Avoid Rate Limiting       |                                                                                              |
| Wait to Avoid Rate Limiting     | Wait                 | Waits to avoid hitting API rate limits  | New or Changed? (false), Calculate Wait Time | Loop Workflows                   |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger Node**  
   - Add a **Schedule Trigger** node named "Run Daily at 2 AM".  
   - Configure to trigger daily at 2 AM (set `triggerAtHour` to 2).  

2. **Create the Set Node for Bitbucket Configuration**  
   - Add a **Set** node named "Set Bitbucket Workspace & Repository".  
   - Add two string fields:  
     - `WorkspaceSlug`: set to your Bitbucket workspace slug (e.g., "myworkspace").  
     - `RepositorySlug`: set to your Bitbucket repository slug (e.g., "myrepo").  
   - Connect "Run Daily at 2 AM" output to this node.  

3. **Add n8n API Node to Get All Workflows**  
   - Add an **n8n API** node named "Get All Workflows".  
   - Use your n8n API credentials with API key authentication.  
   - Leave filters empty to fetch all workflows.  
   - Connect "Set Bitbucket Workspace & Repository" output to this node.  

4. **Add SplitInBatches Node to Loop Workflows**  
   - Add a **SplitInBatches** node named "Loop Workflows".  
   - Use default batch size (1).  
   - Connect "Get All Workflows" output to this node.  

5. **Add HTTP Request Node to Get Existing Workflow from Bitbucket**  
   - Add an **HTTP Request** node named "Get Existing Worfklow from Bitbucket".  
   - Configure:  
     - Method: GET  
     - URL:  
       ```
       https://api.bitbucket.org/2.0/repositories/{{ $json.WorkspaceSlug }}/{{ $json.RepositorySlug }}/src/main/{{ $json.name.replace(/[^a-zA-Z0-9]/g, '-').toLowerCase() }}
       ```  
       (Use expression referencing "Set Bitbucket Workspace & Repository" node for workspace and repo slugs, and current workflow name sanitized.)  
     - Authentication: HTTP Basic Auth using Bitbucket credentials (username + app password).  
     - Response: Full response enabled.  
     - On Error: Continue on failure (to handle 404).  
   - Connect "Loop Workflows" output to this node.  

6. **Add If Node to Check If Workflow is New or Changed**  
   - Add an **If** node named "New or Changed?".  
   - Configure conditions (OR):  
     - Condition 1: `{{$json.error.status}}` equals `404` (indicates new workflow).  
     - Condition 2: Content from Bitbucket file (`$('Get Existing Worfklow from Bitbucket').item.json.data`) not equal to current workflow JSON stringified (`JSON.stringify($('Loop Workflows').item.json, null, 2)`).  
   - Connect "Get Existing Worfklow from Bitbucket" output to this node.  

7. **Add HTTP Request Node to Upload Workflow to Bitbucket**  
   - Add an **HTTP Request** node named "Upload Workflow to Bitbucket".  
   - Configure:  
     - Method: POST  
     - URL:  
       ```
       https://api.bitbucket.org/2.0/repositories/{{ $json.WorkspaceSlug }}/{{ $json.RepositorySlug }}/src
       ```  
     - Authentication: HTTP Basic Auth with Bitbucket credentials.  
     - Headers: Content-Type = application/x-www-form-urlencoded  
     - Body Parameters:  
       - `message`: Commit message with workflow name and timestamp, e.g., `{{$json.name + ' [' + $now.format('yyyy-MM-dd HH:mm:ss') +']'}}`  
       - Filename: sanitized workflow name as key, value is stringified workflow JSON.  
     - Response: Full response enabled.  
   - Connect "New or Changed?" true output to this node.  

8. **Add Code Node to Calculate Wait Time**  
   - Add a **Code** node named "Calculate Wait Time".  
   - Paste the JavaScript code that parses Bitbucket rate limit headers and calculates wait time accordingly (max 30 seconds, default 1 second).  
   - Connect "Upload Workflow to Bitbucket" output to this node.  

9. **Add Wait Node to Respect Rate Limits**  
   - Add a **Wait** node named "Wait to Avoid Rate Limiting".  
   - Set wait time dynamically using expression: `{{$json.waitTime || 1}}` seconds.  
   - Connect "Calculate Wait Time" output to this node.  
   - Also connect "New or Changed?" false output (no upload needed) to this node.  

10. **Loop Back to Process Next Workflow**  
    - Connect "Wait to Avoid Rate Limiting" output back to "Loop Workflows" to continue processing remaining workflows.  

11. **Credentials Setup:**  
    - Create Bitbucket HTTP Basic Auth credentials with your Bitbucket username and app password (with repository write access).  
    - Create n8n API credentials with your n8n API key for accessing workflows.  

12. **Test & Adjust:**  
    - Run the workflow manually or wait for scheduled trigger.  
    - Verify workflows are backed up to Bitbucket repository with proper commit messages and filenames.  
    - Adjust batch size or wait times if hitting API limits or timeouts.  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Bitbucket API rate limits documentation recommends spreading requests to avoid hitting limits.            | https://support.atlassian.com/bitbucket-cloud/docs/api-request-limits/                              |
| Bitbucket App Password must have repository write access to upload files.                                 | https://support.atlassian.com/bitbucket-cloud/docs/app-passwords/                                  |
| Workflow filenames are sanitized by replacing non-alphanumeric characters with dashes and lowercased.    | Ensures compatibility with Bitbucket file naming conventions.                                      |
| Commit messages include timestamps for easy tracking of backups over time.                               | Timestamp format: yyyy-MM-dd HH:mm:ss                                                              |
| This workflow is ideal for self-hosted n8n users wanting free private repository backups with versioning. |                                                                                                    |

---

This documentation fully describes the workflow structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.