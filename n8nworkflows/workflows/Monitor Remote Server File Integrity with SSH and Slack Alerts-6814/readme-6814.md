Monitor Remote Server File Integrity with SSH and Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-remote-server-file-integrity-with-ssh-and-slack-alerts-6814


# Monitor Remote Server File Integrity with SSH and Slack Alerts

### 1. Workflow Overview

This workflow is designed to **monitor file integrity on a remote server** by regularly calculating and verifying file checksums over SSH, and sending alerts via Slack if any changes are detected. It targets IT teams, DevOps, system administrators, managed hosting providers, and small businesses who need automated, real-time detection of unauthorized or unexpected modifications to critical server files.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Periodic initiation of the file integrity check.
- **1.2 File List and Known Checksums Setup:** Define which files to monitor and their expected checksums.
- **1.3 Remote Checksum Retrieval:** Connect to the remote server via SSH and calculate the current checksum of each file.
- **1.4 Checksum Comparison:** Compare retrieved checksums against expected values.
- **1.5 Alerting and Termination:** Send Slack alerts if discrepancies are found; otherwise, end workflow quietly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow on a defined schedule to automate regular file integrity audits without manual intervention.

- **Nodes Involved:**  
  - Scheduled Check (Cron Node)

- **Node Details:**

  - **Scheduled Check**  
    - **Type:** Cron trigger  
    - **Role:** Automatically triggers the workflow at specified times (e.g., daily at 3 AM).  
    - **Configuration:** Default empty parameters; schedule to be customized as needed.  
    - **Expressions/Variables:** None.  
    - **Input:** None (trigger node)  
    - **Output:** Passes control to "List Files & Checksums" node.  
    - **Version:** n8n Cron node v1.  
    - **Edge Cases:** Misconfiguration of schedule may cause workflow to not run or run too frequently.  
    - **Sub-workflow:** None.

#### 1.2 File List and Known Checksums Setup

- **Overview:**  
  Provides a static list of file paths and their verified "good" checksums, serving as the reference for integrity verification.

- **Nodes Involved:**  
  - List Files & Checksums (Code Node)

- **Node Details:**

  - **List Files & Checksums**  
    - **Type:** Code node (JavaScript)  
    - **Role:** Defines the files to monitor and their known checksums; outputs an array of objects with file paths and expected checksums.  
    - **Configuration:** The provided example code is a placeholder that should be replaced with logic to output an array like:  
      ```js
      return [
        { json: { path: '/path/to/file1', knownChecksum: 'abc123...' } },
        { json: { path: '/path/to/file2', knownChecksum: 'def456...' } },
        // ...
      ];
      ```  
    - **Expressions/Variables:** The node’s output JSON includes `path` and `knownChecksum` fields per item.  
    - **Input:** From Scheduled Check node.  
    - **Output:** To Get Remote File Checksum node, passing each file entry.  
    - **Version:** v1.  
    - **Edge Cases:** If the file list or known checksums are incorrect or empty, integrity checks become useless or produce false positives.  
    - **Sub-workflow:** None.

#### 1.3 Remote Checksum Retrieval

- **Overview:**  
  Connects via SSH to the remote server and calculates the current SHA256 checksum of each file listed.

- **Nodes Involved:**  
  - Get Remote File Checksum (SSH Node)

- **Node Details:**

  - **Get Remote File Checksum**  
    - **Type:** SSH node  
    - **Role:** Executes a remote shell command to get the SHA256 checksum of the file at the specified path.  
    - **Configuration:** Command set as:  
      ```
      sha256sum {{ $json.path }}
      ```  
      This uses an expression to insert the file path dynamically per item.  
    - **Expressions/Variables:** Uses `{{ $json.path }}` to pass the file path.  
    - **Input:** From List Files & Checksums node.  
    - **Output:** Provides command output (stdout) containing the checksum and filename.  
    - **Version:** v1 SSH node.  
    - **Edge Cases:**  
      - SSH connection errors (wrong credentials, network issues).  
      - Command failure (file missing, permission denied).  
      - Output parsing errors if command output is unexpected.  
    - **Sub-workflow:** None.

#### 1.4 Checksum Comparison

- **Overview:**  
  Compares the known checksum with the checksum retrieved from the server to detect file modifications.

- **Nodes Involved:**  
  - Checksums Match? (If Node)

- **Node Details:**

  - **Checksums Match?**  
    - **Type:** If node  
    - **Role:** Branches workflow based on whether the remote checksum matches the known checksum.  
    - **Configuration:** Condition compares:  
      ```
      $json.knownChecksum === $json.stdout.split(' ')[0]
      ```  
      - `knownChecksum`: Expected checksum from the code node.  
      - `stdout.split(' ')[0]`: Extracts actual checksum from SSH command output.  
    - **Expressions/Variables:** Uses JSON fields and JavaScript string split.  
    - **Input:** From Get Remote File Checksum node.  
    - **Output:**  
      - True branch (checksums match): proceeds to End Workflow node.  
      - False branch (checksums differ): proceeds to Send Alert node.  
    - **Version:** v1.  
    - **Edge Cases:**  
      - If SSH output format changes unexpectedly, parsing may fail.  
      - Expression errors if fields are missing.  
    - **Sub-workflow:** None.

#### 1.5 Alerting and Termination

- **Overview:**  
  Sends Slack alerts on checksum mismatches or ends the workflow quietly if integrity is intact.

- **Nodes Involved:**  
  - Send Alert (Slack Node)  
  - End Workflow (No-Op Node)

- **Node Details:**

  - **Send Alert**  
    - **Type:** Slack node  
    - **Role:** Sends a formatted alert message to a designated Slack channel on file integrity breach.  
    - **Configuration:**  
      - Text contains dynamic fields for file path, known checksum, and current checksum extracted via expressions:  
        ```
        🚨 *File Integrity Alert!* 🚨
        File: *{{ $json.path }}* has been modified!
        Known Checksum: *{{ $json.knownChecksum }}*
        Current Checksum: *{{ $json.stdout.split(' ')[0] }}*
        ```  
      - Sends message to a Slack user or channel specified by ID (`YOUR_SECURITY_ALERT_CHANNEL_ID` placeholder to be replaced).  
      - Requires Slack API credentials.  
    - **Expressions/Variables:** Uses JSON fields and string splitting to extract checksum.  
    - **Input:** From Checksums Match? node (false branch).  
    - **Output:** None (end of alert branch).  
    - **Version:** Slack node v2.3.  
    - **Edge Cases:**  
      - Slack API authentication errors.  
      - Invalid channel/user ID causing message failure.  
      - Message formatting issues.  
    - **Sub-workflow:** None.

  - **End Workflow**  
    - **Type:** No-Op node  
    - **Role:** Ends the workflow quietly if no file changes detected.  
    - **Configuration:** None.  
    - **Input:** From Checksums Match? node (true branch).  
    - **Output:** None.  
    - **Version:** v1.  
    - **Edge Cases:** None.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name              | Node Type       | Functional Role                    | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                  |
|------------------------|-----------------|----------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Scheduled Check        | Cron            | Trigger workflow on schedule     | None                   | List Files & Checksums   |                                                                                                                              |
| List Files & Checksums | Code            | Define files and known checksums | Scheduled Check        | Get Remote File Checksum |                                                                                                                              |
| Get Remote File Checksum| SSH             | Run sha256sum remotely           | List Files & Checksums | Checksums Match?         |                                                                                                                              |
| Checksums Match?       | If              | Compare checksums, branch logic  | Get Remote File Checksum| Send Alert, End Workflow |                                                                                                                              |
| Send Alert             | Slack           | Send Slack notification on alert | Checksums Match? (false)| None                   |                                                                                                                              |
| End Workflow           | No-Op           | End workflow quietly on success  | Checksums Match? (true) | None                    |                                                                                                                              |
| Sticky Note            | Sticky Note     | Visual flow overview             | None                   | None                    | ## Flow                                                                                                                      |
| Sticky Note1           | Sticky Note     | Documentation and explanation    | None                   | None                    | # 🚨 Automated File Integrity Check 🛡️ <br> Problem, solution, users, and setup details in multilingual explanation.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Cron Node:**
   - Name: `Scheduled Check`
   - Purpose: Trigger workflow on desired schedule.
   - Configuration: Set schedule (e.g., every day at 3:00 AM).
   - Save.

3. **Add a Code Node:**
   - Name: `List Files & Checksums`
   - Purpose: Define files to monitor and their known checksums.
   - Configuration: Replace default code with:
     ```js
     return [
       { json: { path: '/path/to/your/file1', knownChecksum: 'your_known_checksum1' } },
       { json: { path: '/path/to/your/file2', knownChecksum: 'your_known_checksum2' } },
       // Add more files as needed
     ];
     ```
   - Connect output of `Scheduled Check` node to input of this node.

4. **Add SSH Node:**
   - Name: `Get Remote File Checksum`
   - Purpose: Run `sha256sum` on remote files.
   - Configuration:
     - Command: `sha256sum {{ $json.path }}`
     - Credentials: Create and select SSH credentials with access to remote server.
   - Connect output of `List Files & Checksums` node to this node.

5. **Add If Node:**
   - Name: `Checksums Match?`
   - Purpose: Compare known and current checksums.
   - Configuration:
     - Condition:  
       - Mode: Expression  
       - Expression:  
         ```
         $json.knownChecksum === $json.stdout.split(' ')[0]
         ```
     - True: Checksums match (proceed to end).  
     - False: Checksums differ (send alert).
   - Connect output of `Get Remote File Checksum` to this node.

6. **Add Slack Node:**
   - Name: `Send Alert`
   - Purpose: Send alert message to Slack on mismatch.
   - Configuration:
     - Authentication: Set Slack API credentials.
     - Channel/User: Set the Slack channel or user ID for alerts (replace placeholder `[YOUR_SECURITY_ALERT_CHANNEL_ID]`).
     - Message text:  
       ```
       🚨 *File Integrity Alert!* 🚨
       File: *{{ $json.path }}* has been modified!
       Known Checksum: *{{ $json.knownChecksum }}*
       Current Checksum: *{{ $json.stdout.split(' ')[0] }}*
       ```
   - Connect false output of `Checksums Match?` node to this node.

7. **Add No-Op Node:**
   - Name: `End Workflow`
   - Purpose: End workflow quietly if no changes.
   - Connect true output of `Checksums Match?` node to this node.

8. **Verify connections:**
   - `Scheduled Check` → `List Files & Checksums` → `Get Remote File Checksum` → `Checksums Match?`
   - `Checksums Match?` true → `End Workflow`
   - `Checksums Match?` false → `Send Alert`

9. **Save and test:**
   - Run manually to verify connection to SSH server and Slack.
   - Update file paths and checksums to real values.
   - Confirm Slack alert triggers on checksum mismatch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is a vital automated security tool for early detection of unauthorized file changes on remote servers, leveraging n8n’s integration capabilities with SSH and Slack for real-time alerting, ideal for small and mid-sized business IT teams.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Workflow purpose and target audience             |
| Ensure SSH user has permission to run `sha256sum` on target files and network connectivity is reliable to avoid false negatives or failures during checksum retrieval.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | SSH Node operational note                         |
| Slack credentials require OAuth2 token or app with chat permissions; Slack channel ID must be correctly set to receive alerts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Slack Node configuration note                     |
| Before deploying, manually verify known checksums by running `sha256sum [file_path]` on your server to capture baseline integrity values.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Setup prerequisite                                |
| Workflow is scalable to multiple files by adding entries in the Code node array; consider performance impact for very large file sets or frequent schedules.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Scalability consideration                         |
| For further customization, consider adding retries on SSH failures or integrating additional notification channels (e.g., email, SMS).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Enhancement suggestion                            |
| Sticky notes within the workflow provide detailed explanations and multilingual instructions for easier comprehension and onboarding of new users.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Documentation embedded in workflow                |

---

**Disclaimer:** The text provided is exclusively sourced from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.