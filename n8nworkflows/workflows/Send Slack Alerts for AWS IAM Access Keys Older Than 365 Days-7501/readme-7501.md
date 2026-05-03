Send Slack Alerts for AWS IAM Access Keys Older Than 365 Days

https://n8nworkflows.xyz/workflows/send-slack-alerts-for-aws-iam-access-keys-older-than-365-days-7501


# Send Slack Alerts for AWS IAM Access Keys Older Than 365 Days

### 1. Workflow Overview

This workflow automates the detection and notification of AWS IAM access keys that are older than 365 days, helping DevOps, SREs, and security/compliance teams enforce key rotation policies. It runs on a weekly schedule, fetches all IAM users and their access keys, filters out inactive keys, identifies those older than one year, and sends alerts via Slack for remediation.

**Logical Blocks:**

- **1.1 Schedule Trigger:** Initiates the workflow on a weekly basis.
- **1.2 Fetch IAM Users:** Retrieves all IAM users from AWS.
- **1.3 Retrieve Access Keys per User:** For each user, fetches their access keys metadata.
- **1.4 Filter Active Keys:** Removes keys that are not currently active.
- **1.5 Check Key Age:** Determines if keys are older than 365 days.
- **1.6 Notify via Slack:** Sends alert messages about outdated keys.
- **1.7 No Operation (Fallback):** Ends workflow silently if no outdated keys are detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block starts the workflow automatically on a weekly basis to ensure regular checks.

- **Nodes Involved:**  
  - Weekly scheduler

- **Node Details:**  
  - **Weekly scheduler**  
    - Type: Schedule Trigger  
    - Configuration: Runs every week (interval set to weeks)  
    - Expressions/Variables: None  
    - Input: None (start node)  
    - Output: Connects to “Get many users” node  
    - Edge Cases: Scheduler misconfiguration could cause no runs or excessive runs. Timezone settings may affect trigger timing.  
    - Version: 1.2  

---

#### 2.2 Fetch IAM Users

- **Overview:**  
  Retrieves the full list of IAM users from the AWS account.

- **Nodes Involved:**  
  - Get many users

- **Node Details:**  
  - **Get many users**  
    - Type: AWS IAM Node  
    - Configuration: `listUsers` operation with `returnAll` enabled to get all users in one call  
    - Credentials: AWS credential configured for `us-east-1` region  
    - Input: Triggered by “Weekly scheduler”  
    - Output: Passes user data to “Get User Access Key(s)” via main connection  
    - Edge Cases: AWS API throttling, credential expiration, network issues, permission errors (`iam:ListUsers`)  
    - Version: 1  

---

#### 2.3 Retrieve Access Keys per User

- **Overview:**  
  For each IAM user, fetches their associated access keys metadata by calling AWS IAM API directly.

- **Nodes Involved:**  
  - Get User Access Key(s)

- **Node Details:**  
  - **Get User Access Key(s)**  
    - Type: HTTP Request node (using AWS signed request)  
    - Configuration:  
      - URL dynamically constructed with `UserName` from JSON input:  
        `https://iam.amazonaws.com/?Action=ListAccessKeys&UserName={{ $json.UserName }}&Version=2010-05-08`  
      - Uses predefined AWS credentials (`us-east-1`) for signing  
      - No additional query parameters or body  
    - Input: Receives user list from “Get many users” node  
    - Output: Generates access keys list for each user, sent to “Filter out inactive keys”  
    - Edge Cases: API rate limits, invalid user names, expired credentials, network errors, permission errors (`iam:ListAccessKeys`)  
    - Version: 4.2  

---

#### 2.4 Filter Active Keys

- **Overview:**  
  Filters out any access keys that are not active, ensuring only active keys are checked for age.

- **Nodes Involved:**  
  - Filter out inactive keys

- **Node Details:**  
  - **Filter out inactive keys**  
    - Type: Filter node  
    - Configuration: Filters records where:  
      `ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].Status === "Active"`  
    - Input: Access key metadata from “Get User Access Key(s)”  
    - Output: Passes only active keys to “Access Key Older Than 365 days” node  
    - Edge Cases: Missing `Status` field, empty keys list, inconsistent API response structures  
    - Version: 2.2  

---

#### 2.5 Check Key Age

- **Overview:**  
  Evaluates whether the access key creation date is older than 365 days from the current date.

- **Nodes Involved:**  
  - Access Key Older Than 365 days

- **Node Details:**  
  - **Access Key Older Than 365 days**  
    - Type: If node  
    - Configuration: Checks if:  
      `ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].CreateDate.toDateTime('s') < $today.minus(365, 'days')`  
    - Input: Active keys from filter  
    - Output:  
      - True branch: keys older than 365 days → “Send a message” node  
      - False branch: keys younger or equal to 365 days → “No Operation, do nothing” node  
    - Edge Cases: Invalid or missing date, expression evaluation errors, timezone discrepancies  
    - Version: 2.2  

---

#### 2.6 Notify via Slack

- **Overview:**  
  Sends a Slack message to notify about access keys older than 365 days, including user name and exact key age.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  
  - **Send a message**  
    - Type: Slack node  
    - Configuration:  
      - Message text uses expressions to extract user name and calculate key age in days:  
        `"=:warning: Access key for {{ $json.ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].UserName }} is {{ $json.ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].CreateDate.toDateTime('s').diffTo($now, 'days').ceil().abs() }} days old — rotate soon."`  
      - Sends direct message to user ID `U054RMBTVBM` (cached user `trung.tran`)  
      - Authentication: Slack OAuth2 with configured credentials  
    - Input: Keys older than 365 days from “Access Key Older Than 365 days”  
    - Output: None (end node)  
    - Edge Cases: Slack API rate limits, invalid user ID, expired tokens, network issues  
    - Version: 2.3  

---

#### 2.7 No Operation (Fallback)

- **Overview:**  
  Ends the workflow safely when no keys older than the threshold are found, preventing errors or unnecessary actions.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**  
  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Configuration: None  
    - Input: False branch of the “Access Key Older Than 365 days” node  
    - Output: None (end node)  
    - Edge Cases: None expected  
    - Version: 1  

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                         | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                                             |
|----------------------------|--------------------------|---------------------------------------|--------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Weekly scheduler           | Schedule Trigger          | Triggers workflow weekly               | None                     | Get many users              | ### 1. Schedule Workflow<br>Run the workflow on a weekly schedule to automatically check all IAM users.                 |
| Get many users             | AWS IAM                  | Retrieves all IAM users                | Weekly scheduler         | Get User Access Key(s)      | ### 2. Retrieve IAM Users and Access Keys<br>Fetch all IAM users and list their associated access keys.                 |
| Get User Access Key(s)     | HTTP Request             | Fetches access keys for each user     | Get many users           | Filter out inactive keys    | ### 2. Retrieve IAM Users and Access Keys<br>Fetch all IAM users and list their associated access keys.                 |
| Filter out inactive keys   | Filter                   | Filters only active access keys       | Get User Access Key(s)   | Access Key Older Than 365 days | ### 3. Filter Keys<br>Remove inactive keys and keep only those that are active.                                         |
| Access Key Older Than 365 days | If                       | Checks if key is older than 365 days  | Filter out inactive keys | Send a message; No Operation | ### 4. Check Key Age<br>Identify keys that are older than 365 days based on their creation date.                         |
| Send a message             | Slack                    | Sends Slack alert about old keys      | Access Key Older Than 365 days (true) | None                        | ### 5. Notify via Slack<br>Send a Slack message with details of any outdated keys for review and action.                 |
| No Operation, do nothing   | No Operation (NoOp)      | Ends workflow quietly if no old keys  | Access Key Older Than 365 days (false) | None                        |                                                                                                                         |
| Sticky Note                | Sticky Note              | Documentation and overview            | None                     | None                        | # AWS IAM Access Key Rotation Reminder Automation Workflow... [Full content covering workflow purpose, setup, customization] |
| Sticky Note1               | Sticky Note              | Schedule description                  | None                     | None                        | ### 1. Schedule Workflow<br>Run the workflow on a weekly schedule to automatically check all IAM users.                 |
| Sticky Note2               | Sticky Note              | Fetch users and keys description      | None                     | None                        | ### 2. Retrieve IAM Users and Access Keys<br>Fetch all IAM users and list their associated access keys.                 |
| Sticky Note3               | Sticky Note              | Filter active keys explanation         | None                     | None                        | ### 3. Filter Keys<br>Remove inactive keys and keep only those that are active.                                         |
| Sticky Note4               | Sticky Note              | Key age check explanation              | None                     | None                        | ### 4. Check Key Age<br>Identify keys that are older than 365 days based on their creation date.                         |
| Sticky Note5               | Sticky Note              | Slack notification explanation        | None                     | None                        | ### 5. Notify via Slack<br>Send a Slack message with details of any outdated keys for review and action.                 |
| Sticky Note7               | Sticky Note              | Workflow screenshot image              | None                     | None                        | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-17+at+1.56.54%E2%80%AFPM.png)               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Weekly scheduler” node:**  
   - Type: Schedule Trigger  
   - Set interval to weeks (runs once per week)  
   - Position: Left-most (start node)

2. **Add “Get many users” node:**  
   - Type: AWS IAM  
   - Operation: ListUsers (default)  
   - Set `returnAll` to true to get all users  
   - Credentials: Use AWS credentials configured for `us-east-1` region with permissions `iam:ListUsers`  
   - Connect input from “Weekly scheduler” main output

3. **Add “Get User Access Key(s)” node:**  
   - Type: HTTP Request  
   - Set URL expression to:  
     `https://iam.amazonaws.com/?Action=ListAccessKeys&UserName={{ $json.UserName }}&Version=2010-05-08`  
   - Authentication: Use AWS credentials (same as above) for signing requests  
   - Connect input from “Get many users” output

4. **Add “Filter out inactive keys” node:**  
   - Type: Filter  
   - Condition:  
     - Field: `ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].Status`  
     - Operator: Equals  
     - Value: `Active`  
   - Connect input from “Get User Access Key(s)” output

5. **Add “Access Key Older Than 365 days” node:**  
   - Type: If node  
   - Condition:  
     - Left value (expression): `{{ $json.ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].CreateDate.toDateTime('s') }}`  
     - Operator: Before  
     - Right value (expression): `{{ $today.minus(365, 'days') }}`  
   - Connect input from “Filter out inactive keys” output

6. **Add “Send a message” node:**  
   - Type: Slack  
   - Configure Slack OAuth2 credentials with posting permissions  
   - Message text (expression):  
     `=:warning: Access key for {{ $json.ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].UserName }} is {{ $json.ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].CreateDate.toDateTime('s').diffTo($now, 'days').ceil().abs() }} days old — rotate soon.`  
   - Recipient: Set user by ID `U054RMBTVBM` or your target Slack user/channel  
   - Connect input from “Access Key Older Than 365 days” node’s true branch

7. **Add “No Operation, do nothing” node:**  
   - Type: NoOp  
   - Connect input from “Access Key Older Than 365 days” node’s false branch

8. **Arrange nodes in logical order:**  
   `Weekly scheduler → Get many users → Get User Access Key(s) → Filter out inactive keys → Access Key Older Than 365 days → [Send a message / No Operation]`

9. **Add sticky notes:** (optional but recommended)  
   - Add notes describing each block for clarity and future maintenance.

10. **Save and activate workflow:**  
    - Test the workflow with sample data or in dry-run mode if available.  
    - Ensure AWS credentials have correct permissions: `iam:ListUsers`, `iam:ListAccessKeys`.  
    - Ensure Slack OAuth2 is scoped to send messages to the intended channel or user.  
    - Adjust schedule timing as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed specifically for AWS IAM access key rotation monitoring and Slack alerting. It can be extended to support multiple AWS accounts by duplicating or looping through AWS credentials. It also supports customization such as adjusting the age threshold, adding logging to Google Sheets or databases, or automating key deactivation post-approval.                                                                                                                                                                                                                 | Workflow customization ideas                                                                      |
| IAM permissions required: `iam:ListUsers` and `iam:ListAccessKeys`. Ensure the AWS credential used has these permissions and is scoped to the `us-east-1` region, as IAM API signing requires this region.                                                                                                                                                                                                                                                                                                                                                                                                            | AWS permissions and region requirements                                                          |
| Slack bot credentials require `chat:write` scope and permission to send messages to the target user or channel. Use OAuth2 authentication with a Slack app installed into your workspace.                                                                                                                                                                                                                                                                                                                                                                                                                         | Slack OAuth2 bot setup instructions                                                             |
| Screenshot of workflow UI and node connections is available here for visual reference: ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-17+at+1.56.54%E2%80%AFPM.png)                                                                                                                                                                                                                                                                                                                                                                                                                  | Visual workflow layout                                                                            |

---

**Disclaimer:** The provided text originates exclusively from an automation workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.