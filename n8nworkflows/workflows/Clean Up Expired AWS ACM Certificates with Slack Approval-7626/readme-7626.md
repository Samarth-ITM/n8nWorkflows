Clean Up Expired AWS ACM Certificates with Slack Approval

https://n8nworkflows.xyz/workflows/clean-up-expired-aws-acm-certificates-with-slack-approval-7626


# Clean Up Expired AWS ACM Certificates with Slack Approval

### 1. Workflow Overview

This workflow automates the cleanup of expired AWS ACM (AWS Certificate Manager) SSL certificates with human approval via Slack. It is designed primarily for AWS administrators, DevOps teams, and IT admins who want to maintain a secure and tidy AWS environment by removing expired SSL certificates only after explicit Slack-based approval.

**Use cases:**  
- Automated daily auditing and cleanup of expired ACM certificates  
- Slack-based human approval to prevent accidental deletions  
- Visibility and control over SSL certificate lifecycle management  

**Logical blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow on a daily (or custom) schedule.  
- **1.2 AWS ACM Certificates Retrieval:** Fetches all ACM certificates in the configured AWS region.  
- **1.3 Filter Expired Certificates:** Filters the list to keep only those certificates that are expired.  
- **1.4 Slack Notification & Approval:** Sends detailed Slack messages about expired certificates and waits for user reactions (approve/reject).  
- **1.5 ACM Certificate Deletion:** Deletes the expired certificate if approval is received.  
- **1.6 Slack Notification of Deletion:** Notifies IT admins on Slack that the certificate was deleted successfully.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow automatically on a periodic schedule (default daily) to start the certificate cleanup process.

- **Nodes Involved:**  
  - `Daily Schedule Trigger`

- **Node Details:**  
  - **Node:** `Daily Schedule Trigger`  
  - **Type:** Schedule Trigger  
  - **Configuration:** Set to run at a user-defined interval (default is daily)  
  - **Key Parameters:** Interval rule (unspecified in JSON, but typically daily)  
  - **Input:** None (trigger node)  
  - **Output:** Triggers `Get many certificates` node  
  - **Potential Failures:** Scheduling misconfiguration, workflow not activated  
  - **Version:** 1.2

---

#### 1.2 AWS ACM Certificates Retrieval

- **Overview:**  
  Retrieves all AWS ACM certificates in the specified region to analyze their expiration status.

- **Nodes Involved:**  
  - `Get many certificates`

- **Node Details:**  
  - **Node:** `Get many certificates`  
  - **Type:** AWS Certificate Manager node  
  - **Operation:** `getMany` — lists all ACM certificates  
  - **Credentials:** AWS credentials scoped to region `ap-southeast-1`  
  - **Input:** Trigger from `Daily Schedule Trigger`  
  - **Output:** Sends list of certificates to filter node  
  - **Edge Cases:** AWS API throttling, permission errors (`acm:ListCertificates`), network connectivity  
  - **Version:** 1

---

#### 1.3 Filter Expired Certificates

- **Overview:**  
  Filters the retrieved certificates to isolate those that have expired based on their `Status` field.

- **Nodes Involved:**  
  - `Get expired certification only`

- **Node Details:**  
  - **Node:** `Get expired certification only`  
  - **Type:** Filter node  
  - **Filter Condition:** Selects certificates where `Status == "EXPIRED"`  
  - **Input:** List of certificates from `Get many certificates`  
  - **Output:** Only expired certificates forwarded to Slack approval node  
  - **Edge Cases:** No expired certificates found (empty output), `Status` field missing or malformed  
  - **Version:** 2.2

---

#### 1.4 Slack Notification & Approval

- **Overview:**  
  Sends a detailed Slack message about each expired certificate and waits for a reaction from an admin to approve or reject deletion.

- **Nodes Involved:**  
  - `Send message and wait for response`

- **Node Details:**  
  - **Node:** `Send message and wait for response`  
  - **Type:** Slack node (Send & Wait for Reaction)  
  - **Slack User:** Target user ID `U054RMBTVBM` (cached as "trung.tran")  
  - **Message:** Contains certificate domain, ARN, issue & expiration dates formatted with expressions.  
  - **Options:** Waits up to 60 minutes for approval reaction (`:white_check_mark:` to approve, `:x:` to reject)  
  - **Authentication:** OAuth2 via Slack Bot Token  
  - **Input:** Expired certificate data  
  - **Output:** Passes approved certificates to deletion node  
  - **Edge Cases:** Slack API errors, user does not respond within wait time, invalid user ID, message formatting errors  
  - **Note:** This node is currently disabled in the workflow JSON (may be for testing or staging)  
  - **Version:** 2.3

---

#### 1.5 ACM Certificate Deletion

- **Overview:**  
  Deletes the approved expired certificate from AWS ACM.

- **Nodes Involved:**  
  - `Delete a certificate`

- **Node Details:**  
  - **Node:** `Delete a certificate`  
  - **Type:** AWS Certificate Manager node  
  - **Operation:** `delete`  
  - **Parameter:** Uses `CertificateArn` from the approved certificate JSON  
  - **Credentials:** AWS credentials scoped to region `us-east-1` (note different region than retrieval node)  
  - **Input:** Approved certificate data from Slack approval node  
  - **Output:** Triggers Slack notification of deletion  
  - **Edge Cases:** AWS permission errors (`acm:DeleteCertificate`), invalid ARN, regional mismatches, API throttling  
  - **Version:** 1

---

#### 1.6 Slack Notification of Deletion

- **Overview:**  
  Sends a confirmation Slack message to the admin notifying that the expired certificate was deleted successfully.

- **Nodes Involved:**  
  - `Inform IT Admin`

- **Node Details:**  
  - **Node:** `Inform IT Admin`  
  - **Type:** Slack node (Send Message)  
  - **Message:**  
    - Confirmation checkmark emoji  
    - Domain name, ARN, deletion timestamp  
    - Name of user who approved deletion (extracted from Slack approval node JSON)  
  - **Slack User:** Same user ID as before (`U054RMBTVBM`)  
  - **Authentication:** OAuth2 Slack Bot Token  
  - **Input:** Output from `Delete a certificate` node  
  - **Output:** End of workflow  
  - **Edge Cases:** Slack API errors, missing user info in approval JSON  
  - **Version:** 2.3

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                         | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                     |
|------------------------------|-------------------------------|---------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Daily Schedule Trigger        | Schedule Trigger              | Starts workflow on schedule            | None                          | Get many certificates          | ### 1. Schedule Trigger: The workflow starts on a scheduled basis (e.g., daily at 09:00).       |
| Get many certificates         | AWS Certificate Manager       | Fetches all ACM certificates            | Daily Schedule Trigger         | Get expired certification only | ### 2. Get Certificates: Fetches all ACM certificates in the configured AWS region(s).          |
| Get expired certification only| Filter                       | Filters only expired certificates       | Get many certificates          | Send message and wait for response | ### 3. Filter Expired Certificates Only: Checks each certificate and keeps only those expired.   |
| Send message and wait for response | Slack (Send & Wait for Reaction) | Sends Slack approval message and waits | Get expired certification only | Delete a certificate           | ### 4. Notify via Slack and Wait for Approval: Sends Slack message and waits for approval.       |
| Delete a certificate          | AWS Certificate Manager       | Deletes approved expired certificate    | Send message and wait for response | Inform IT Admin            | ### 5. Delete Expired Certificate: Deletes if approved, else ends workflow.                      |
| Inform IT Admin               | Slack (Send Message)          | Notifies IT admin of deletion success  | Delete a certificate           | None                          | ### 6. Notify admin via Slack                                                                    |
| Sticky Note                  | Sticky Note                  | Documentation & explanation node        | None                          | None                          | # Clean Up Expired AWS ACM Certificates with Human Approval (detailed workflow description)     |
| Sticky Note1                 | Sticky Note                  | Documentation for Schedule Trigger      | None                          | None                          | ### 1. Schedule Trigger                                                                        |
| Sticky Note2                 | Sticky Note                  | Documentation for Get Certificates      | None                          | None                          | ### 2. Get Certificates                                                                        |
| Sticky Note3                 | Sticky Note                  | Documentation for Filter Expired Certs  | None                          | None                          | ### 3. Filter Expired Certificates Only                                                       |
| Sticky Note4                 | Sticky Note                  | Documentation for Slack Approval         | None                          | None                          | ### 4. Notify via Slack and Wait for Approval                                                 |
| Sticky Note5                 | Sticky Note                  | Documentation for Certificate Deletion  | None                          | None                          | ### 5. Delete Expired Certificate                                                            |
| Sticky Note6                 | Sticky Note                  | Screenshot for visual aid                 | None                          | None                          | (Screenshot image link)                                                                        |
| Sticky Note7                 | Sticky Note                  | Documentation for Slack notification     | None                          | None                          | ### 6. Notify admin via Slack                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Daily Schedule Trigger`  
   - Set interval to daily or desired cron schedule to run the workflow automatically.

3. **Add AWS Certificate Manager node:**  
   - Name: `Get many certificates`  
   - Operation: `getMany`  
   - Credentials: AWS IAM credentials with permissions: `acm:ListCertificates` scoped to region `ap-southeast-1`.  
   - Connect `Daily Schedule Trigger` output to this node's input.

4. **Add a Filter node:**  
   - Name: `Get expired certification only`  
   - Set filter condition:  
     - Field: `Status`  
     - Operator: Equals  
     - Value: `EXPIRED`  
   - Connect `Get many certificates` output to filter input.

5. **Add Slack node (Send & Wait for Reaction):**  
   - Name: `Send message and wait for response`  
   - Operation: `sendAndWait`  
   - Set target user to the Slack user or channel (example uses user ID `U054RMBTVBM`).  
   - Compose message with details: domain, ARN, issue date, expiration date using expressions, e.g. `{{ $json.DomainName }}`  
   - Set wait time: 60 minutes (can be customized).  
   - Approval reactions:  
     - `:white_check_mark:` to approve (deletion)  
     - `:x:` to reject (skip deletion)  
   - Authentication: Slack OAuth2 with a bot token having `chat:write`, `reactions:read`, and `channels:read` scopes.  
   - Connect filter output to this Slack node.

6. **Add AWS Certificate Manager node for deletion:**  
   - Name: `Delete a certificate`  
   - Operation: `delete`  
   - Parameter: Set `certificateArn` to expression `{{$json.CertificateArn}}` from Slack approval output.  
   - Credentials: AWS IAM credentials with `acm:DeleteCertificate` permission scoped to region `us-east-1` (ensure region consistency or adjust accordingly).  
   - Connect Slack node output to this deletion node.

7. **Add Slack node to notify IT admin:**  
   - Name: `Inform IT Admin`  
   - Operation: `sendMessage`  
   - Target user: same as approval Slack user or channel.  
   - Message content: Confirmation with certificate domain, ARN, deletion timestamp (`{{ $now }}`), and approver's name extracted from Slack approval node output.  
   - Authentication: Slack OAuth2 with bot token.  
   - Connect deletion node output to this Slack notification node.

8. **Activate the workflow.**

9. **Ensure credentials setup:**  
   - AWS credentials with permissions:  
     - `acm:ListCertificates`  
     - `acm:DeleteCertificate`  
   - Slack OAuth2 credentials with bot token scopes:  
     - `chat:write`  
     - `reactions:read`  
     - `channels:read`

10. **Optional Customizations:**  
    - Adjust Slack wait time before auto-checking reactions.  
    - Change Slack target user or channel for approvals and notifications.  
    - Add sub-workflows or error handling nodes to log failures or retries.  
    - Implement dry-run mode by adding IF nodes before deletion for testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Automates expired AWS ACM certificate cleanup with Slack approval ensuring secure and tidy AWS environments.                                                                                                                      | Workflow Description                                                                                   |
| Slack Bot Token requires `chat:write`, `reactions:read`, and `channels:read` permissions to send messages and listen for reactions.                                                                                              | Slack API documentation                                                                                |
| AWS IAM user/role must have `acm:ListCertificates`, `acm:DescribeCertificate` (optional if used), and `acm:DeleteCertificate` permissions scoped to the required region(s).                                                       | AWS IAM Policy Setup                                                                                   |
| Workflow is designed to run daily but scheduling can be customized.                                                                                                                                                               | Node Configuration                                                                                      |
| Slack reactions used for approval are `:white_check_mark:` for approve and `:x:` for reject. Workflow waits 1 hour by default before proceeding.                                                                                  | Slack Node Configuration                                                                                |
| Screenshot of Slack message and workflow layout available: ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-20+at+11.31.13%E2%80%AFAM.png)                                                            | Sticky Note6 content                                                                                     |
| To add logging or dry-run modes, integrate Google Sheets, Notion, DynamoDB or add IF nodes for conditional deletion.                                                                                                              | Customization advice                                                                                     |
| Note that one AWS ACM node uses region `ap-southeast-1` and another `us-east-1`; ensure region consistency per your AWS setup to avoid errors.                                                                                   | Credential and region note                                                                               |

---

**Disclaimer:** The provided text is extracted exclusively from an automated n8n workflow respecting all applicable content policies. It contains no illegal or offensive content. All data handled is legal and publicly accessible.