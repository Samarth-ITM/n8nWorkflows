Automated Wazuh Rule Deployment Pipeline with GitHub, XML Validation & Telegram Alerts

https://n8nworkflows.xyz/workflows/automated-wazuh-rule-deployment-pipeline-with-github--xml-validation---telegram-alerts-7226


# Automated Wazuh Rule Deployment Pipeline with GitHub, XML Validation & Telegram Alerts

### 1. Workflow Overview

This workflow automates the deployment pipeline for Wazuh detection rules triggered by GitHub commits. It is designed to validate and deploy updated XML rules while ensuring correctness via SSH commands and notifying stakeholders through Telegram alerts. The pipeline handles GitHub webhook triggers, extracts changed files, downloads and uploads the rules to a Wazuh manager, validates the rules, deploys them, restarts the Wazuh manager service, and finally sends success or failure notifications.

Logical blocks:

- **1.1 GitHub Trigger & Commit Validation:** Listens for GitHub webhook events and filters commits for deployment eligibility.
- **1.2 Changed Files Extraction & Download:** Extracts list of changed files from the commit and downloads relevant Wazuh rule files.
- **1.3 Upload & Rule Validation:** Uploads downloaded rules to the Wazuh manager via SSH and runs validation commands on the server.
- **1.4 Deployment & Service Restart:** Deploys validated rules, restarts Wazuh manager service, and confirms deployment success.
- **1.5 Notification:** Sends Telegram messages indicating success or failure of the deployment process.
- **1.6 No Operation Branch:** Handles cases where commits are not valid for deployment (e.g., irrelevant changes).

---

### 2. Block-by-Block Analysis

#### 1.1 GitHub Trigger & Commit Validation

**Overview:**  
This block receives webhook events from GitHub when changes occur in the repository. It validates if the commit includes files relevant for deployment, controlling whether the workflow proceeds or stops.

**Nodes Involved:**  
- Github Trigger  
- Valid Commit for Deployment  
- No Operation, do nothing

**Node Details:**  

- **Github Trigger**  
  - Type: GitHub Trigger (Webhook listener)  
  - Role: Entry point triggered on GitHub repository events, e.g., push events.  
  - Configuration: Default webhook settings listening for push events.  
  - Inputs: GitHub webhook payload.  
  - Outputs: Passes event data to commit validation node.  
  - Edge Cases: Webhook misconfiguration, authentication failure, payload schema changes.  

- **Valid Commit for Deployment**  
  - Type: IF node  
  - Role: Checks whether the commit contains files relevant to Wazuh rule deployment.  
  - Configuration: Conditional expressions that evaluate commit content; likely checks for specific paths or file extensions in changed files.  
  - Inputs: GitHub Trigger output.  
  - Outputs:  
    - True branch: Proceed to extract changed files.  
    - False branch: Invoke No Operation node.  
  - Edge Cases: Expression failures due to unexpected payload structure, false negatives if commit files are misclassified.  

- **No Operation, do nothing**  
  - Type: NoOp (No operation)  
  - Role: Terminates workflow gracefully when commits are irrelevant to deployment.  
  - Inputs: False branch from Valid Commit for Deployment.  
  - Outputs: None  
  - Edge Cases: None. Acts as a safe endpoint.

---

#### 1.2 Changed Files Extraction & Download

**Overview:**  
Extracts the list of changed files from the commit and downloads each relevant Wazuh rule file via HTTP requests.

**Nodes Involved:**  
- Extract Changed Files  
- Download Rule

**Node Details:**  

- **Extract Changed Files**  
  - Type: Code (JavaScript) node  
  - Role: Parses the GitHub webhook payload to extract changed file paths from the commit details.  
  - Configuration: Custom JavaScript code processing the webhook JSON to produce a list of changed files.  
  - Inputs: Output from Valid Commit for Deployment node (true branch).  
  - Outputs: List of changed files passed to Download Rule node.  
  - Edge Cases: Parsing errors if payload schema changes, empty or malformed file list.

- **Download Rule**  
  - Type: HTTP Request  
  - Role: Downloads the content of each changed Wazuh rule file from GitHub or another HTTP source.  
  - Configuration: HTTP GET requests targeting raw file URLs corresponding to changed files.  
  - Inputs: List of changed file paths (from Extract Changed Files).  
  - Outputs: File contents forwarded to Upload a file node.  
  - Edge Cases: HTTP errors (404, 403, timeout), rate limiting by GitHub, corrupted downloads.

---

#### 1.3 Upload & Rule Validation

**Overview:**  
Uploads downloaded files to the Wazuh manager server via SSH and performs validation checks on the rules to ensure they are syntactically and semantically correct.

**Nodes Involved:**  
- Upload a file  
- Rule Validation  
- Rule Validation check

**Node Details:**  

- **Upload a file**  
  - Type: SSH  
  - Role: Transfers the downloaded rule files to the Wazuh manager system using SCP or direct file upload commands.  
  - Configuration: SSH credentials configured for the Wazuh manager server, commands or file transfer parameters set appropriately.  
  - Inputs: File content from Download Rule node.  
  - Outputs: Triggers Rule Validation node after upload completion.  
  - Edge Cases: SSH connection failure, authentication errors, insufficient permissions, file write errors on server.

- **Rule Validation**  
  - Type: SSH  
  - Role: Executes validation commands on the Wazuh manager to check rule correctness (e.g., XML syntax validation).  
  - Configuration: SSH credentials reused; runs shell commands/scripts to validate rules.  
  - Inputs: Output of Upload a file node.  
  - Outputs: Passes results to conditional check node.  
  - Edge Cases: Command failures, validation script errors, permission issues.

- **Rule Validation check**  
  - Type: IF node  
  - Role: Analyzes SSH command output to determine if validation passed or failed.  
  - Configuration: Condition evaluates command exit status or output text indicating success or failure.  
  - Inputs: Output from Rule Validation node.  
  - Outputs:  
    - True branch: Proceed to Deploying the Rules node.  
    - False branch: Trigger Rules deployment failed Telegram alert.  
  - Edge Cases: Misinterpretation of SSH output, inconsistent validation messages.

---

#### 1.4 Deployment & Service Restart

**Overview:**  
Deploys validated rules on the Wazuh server and restarts the Wazuh manager service to apply changes.

**Nodes Involved:**  
- Deploying the Rules  
- Restart Wazuh_manager  
- Final Confirmation check

**Node Details:**  

- **Deploying the Rules**  
  - Type: SSH  
  - Role: Executes deployment commands on Wazuh manager, such as copying files into active directories or running deployment scripts.  
  - Configuration: SSH credentials with required permissions; shell commands for deployment.  
  - Inputs: True branch from Rule Validation check.  
  - Outputs: Triggers Restart Wazuh_manager node.  
  - Edge Cases: Deployment command failures, permission issues, partial deployments.

- **Restart Wazuh_manager**  
  - Type: SSH  
  - Role: Restarts the Wazuh manager service to apply new rules.  
  - Configuration: SSH credentials; command to restart Wazuh manager service (e.g., systemctl restart wazuh-manager).  
  - Inputs: Output from Deploying the Rules node.  
  - Outputs: Passes to final confirmation node.  
  - Edge Cases: Service restart failure, SSH connectivity loss.

- **Final Confirmation check**  
  - Type: IF node  
  - Role: Checks success or failure of the restart command or deployment status to decide which notification to send.  
  - Configuration: Condition based on restart SSH command response.  
  - Inputs: Output of Restart Wazuh_manager node.  
  - Outputs:  
    - True branch: Send success Telegram message.  
    - False branch: Send failure Telegram message.  
  - Edge Cases: Misinterpretation of service status, false success/failure detection.

---

#### 1.5 Notification

**Overview:**  
Sends Telegram alerts to notify stakeholders about success or failure of the deployment process.

**Nodes Involved:**  
- ✅ Success Message  
- ❌ Failure Message  
- Rules deployment failed

**Node Details:**  

- **✅ Success Message**  
  - Type: Telegram  
  - Role: Sends a Telegram message indicating successful deployment and restart of Wazuh rules.  
  - Configuration: Telegram bot credentials, chat ID, message content confirming success.  
  - Inputs: True branch of Final Confirmation check.  
  - Outputs: None (workflow end).  
  - Edge Cases: Telegram API rate limits, invalid chat ID, network issues.

- **❌ Failure Message**  
  - Type: Telegram  
  - Role: Sends a Telegram message indicating failure during deployment or restart.  
  - Configuration: Telegram bot credentials, chat ID, message content indicating failure.  
  - Inputs: False branch of Final Confirmation check.  
  - Outputs: None (workflow end).  
  - Edge Cases: Same as Success Message node.

- **Rules deployment failed**  
  - Type: Telegram  
  - Role: Sends a Telegram alert specifically for rule validation failure before deployment.  
  - Configuration: Telegram bot credentials, chat ID, message content for validation failure.  
  - Inputs: False branch of Rule Validation check.  
  - Outputs: None (workflow end).  
  - Edge Cases: Same as above.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                        | Input Node(s)           | Output Node(s)                     | Sticky Note                                                                                               |
|--------------------------|--------------------|-------------------------------------|------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------|
| Github Trigger           | GitHub Trigger     | Receive GitHub webhook events       | –                      | Valid Commit for Deployment       |                                                                                                           |
| Valid Commit for Deployment | IF                 | Validate commit relevance for deploy | Github Trigger         | Extract Changed Files, No Operation, do nothing |                                                                                                           |
| No Operation, do nothing  | NoOp               | Stop workflow on irrelevant commit  | Valid Commit for Deployment (false) | –                                |                                                                                                           |
| Extract Changed Files     | Code (JavaScript)  | Extract changed file list from commit | Valid Commit for Deployment (true) | Download Rule                    |                                                                                                           |
| Download Rule             | HTTP Request       | Download changed rule files          | Extract Changed Files   | Upload a file                    |                                                                                                           |
| Upload a file             | SSH                | Upload rule files to Wazuh manager   | Download Rule          | Rule Validation                  |                                                                                                           |
| Rule Validation           | SSH                | Validate uploaded rules on server    | Upload a file          | Rule Validation check            |                                                                                                           |
| Rule Validation check     | IF                 | Check validation success/failure     | Rule Validation        | Deploying the Rules, Rules deployment failed |                                                                                                           |
| Rules deployment failed   | Telegram           | Alert on rule validation failure     | Rule Validation check (false) | –                              |                                                                                                           |
| Deploying the Rules       | SSH                | Deploy validated rules               | Rule Validation check (true) | Restart Wazuh_manager           |                                                                                                           |
| Restart Wazuh_manager     | SSH                | Restart Wazuh manager service        | Deploying the Rules    | Final Confirmation check          |                                                                                                           |
| Final Confirmation check  | IF                 | Confirm deployment success           | Restart Wazuh_manager  | ✅ Success Message, ❌ Failure Message |                                                                                                           |
| ✅ Success Message        | Telegram           | Notify deployment success            | Final Confirmation check (true) | –                              |                                                                                                           |
| ❌ Failure Message        | Telegram           | Notify deployment failure            | Final Confirmation check (false) | –                              |                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node**  
   - Type: GitHub Trigger  
   - Configure webhook to listen for push events on your Wazuh repository.  
   - No additional parameters needed by default.  
   - This node starts the workflow.

2. **Add IF Node: Valid Commit for Deployment**  
   - Type: IF  
   - Set conditions to check if the commit contains Wazuh rule files (e.g., file path includes `rules/` or XML extensions).  
   - Connect input from GitHub Trigger node.  
   - True branch continues workflow; false branch connects to No Operation.

3. **Add No Operation Node**  
   - Type: NoOp  
   - Connect input from false branch of Valid Commit for Deployment.

4. **Add Code Node: Extract Changed Files**  
   - Type: Code (JavaScript)  
   - Paste code to parse webhook payload and extract the list of changed files (paths).  
   - Connect input from true branch of Valid Commit for Deployment.

5. **Add HTTP Request Node: Download Rule**  
   - Type: HTTP Request  
   - Configure to perform GET requests to retrieve raw content of each changed file from GitHub (or relevant source).  
   - Use URL templates with variables referencing extracted file paths.  
   - Connect input from Extract Changed Files node.

6. **Add SSH Node: Upload a file**  
   - Type: SSH  
   - Configure SSH credentials for Wazuh manager server (host, username, private key/password).  
   - Set command or SCP for uploading downloaded files to the appropriate directory.  
   - Connect input from Download Rule node.

7. **Add SSH Node: Rule Validation**  
   - Type: SSH  
   - Use same SSH credentials.  
   - Configure command to validate Wazuh rules (e.g., XML linting, Wazuh validation script).  
   - Connect input from Upload a file node.

8. **Add IF Node: Rule Validation check**  
   - Type: IF  
   - Condition based on SSH command output or exit status to detect validation success or failure.  
   - Connect input from Rule Validation node.  
   - True branch to Deploying the Rules.  
   - False branch to Rules deployment failed.

9. **Add Telegram Node: Rules deployment failed**  
   - Type: Telegram  
   - Configure Telegram bot credentials and chat ID.  
   - Message content: “Rule validation failed, deployment aborted.”  
   - Connect input from false branch of Rule Validation check.

10. **Add SSH Node: Deploying the Rules**  
    - Type: SSH  
    - Reuse SSH credentials.  
    - Configure deployment commands (copying files to active directories, applying configurations).  
    - Connect input from true branch of Rule Validation check.

11. **Add SSH Node: Restart Wazuh_manager**  
    - Type: SSH  
    - Reuse SSH credentials.  
    - Command to restart Wazuh manager service (e.g., `systemctl restart wazuh-manager`).  
    - Connect input from Deploying the Rules node.

12. **Add IF Node: Final Confirmation check**  
    - Type: IF  
    - Condition based on restart command output or service health check.  
    - Connect input from Restart Wazuh_manager node.  
    - True branch to Success Message.  
    - False branch to Failure Message.

13. **Add Telegram Node: ✅ Success Message**  
    - Type: Telegram  
    - Configure Telegram bot credentials and chat ID.  
    - Message content: “Wazuh rules deployed and manager restarted successfully.”  
    - Connect input from true branch of Final Confirmation check.

14. **Add Telegram Node: ❌ Failure Message**  
    - Type: Telegram  
    - Configure Telegram bot credentials and chat ID.  
    - Message content: “Wazuh rules deployment or restart failed.”  
    - Connect input from false branch of Final Confirmation check.

15. **Connect all nodes as per the logic described in the overview.**  
    - Ensure the flow from GitHub Trigger through to notifications is correctly linked.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Telegram nodes require bot token and chat ID setup for sending messages.                            | Telegram Bot API documentation: https://core.telegram.org/bots/api  |
| SSH nodes require secure credential configuration, including private keys or password authentication. | Ensure Wazuh manager SSH access is correctly configured and secure. |
| GitHub Trigger requires proper webhook setup in the GitHub repository settings.                     | GitHub Webhooks doc: https://docs.github.com/en/developers/webhooks-and-events/webhooks |
| The code node for extracting files must be updated if GitHub webhook payload format changes.       | GitHub push event payload: https://docs.github.com/en/developers/webhooks-and-events/webhook-events-and-payloads#push |
| Validate SSH command outputs thoroughly to avoid false positives or negatives in deployment status. |                                                                     |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.