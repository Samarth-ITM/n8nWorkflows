Monitor & Alert on Inactive AWS IAM Users with Slack Notifications

https://n8nworkflows.xyz/workflows/monitor---alert-on-inactive-aws-iam-users-with-slack-notifications-7500


# Monitor & Alert on Inactive AWS IAM Users with Slack Notifications

### 1. Workflow Overview

This workflow automates the monitoring and alerting of inactive AWS IAM users, targeting SRE/DevOps teams, security/compliance officers, and MSPs managing AWS accounts. It runs weekly to identify IAM users who have had no activity for over 90 days, then alerts a designated Slack channel with details for review and action.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Kicks off the workflow on a weekly basis.
- **1.2 IAM User Retrieval:** Lists all IAM users and fetches detailed information about each.
- **1.3 Data Filtering:** Filters out users without valid activity data or service-linked accounts.
- **1.4 Inactivity Check:** Compares last activity timestamps against the 90-day inactivity threshold.
- **1.5 Notification:** Sends Slack alerts for users identified as inactive.
- **1.6 No Operation:** A pass-through node for users not meeting inactivity criteria (does nothing).
- **1.7 Documentation:** Sticky notes provide detailed guidance and instructions within the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Initiates the workflow on a weekly interval to ensure regular checks.
- **Nodes Involved:** 
  - Weekly scheduler
  - Sticky Note1

- **Node Details:**

  - **Weekly scheduler**
    - Type: Schedule Trigger
    - Role: Automatically triggers the workflow weekly.
    - Configuration: Set to trigger every week (default weekly interval).
    - Inputs: None (trigger node)
    - Outputs: Connects to "Get many users".
    - Edge Cases: Scheduler misconfiguration could result in no runs; ensure timezone alignment.

  - **Sticky Note1**
    - Type: Sticky Note
    - Role: Describes the trigger block purpose.
    - Configuration: Contains text explaining the scheduled or manual trigger.
    - Inputs/Outputs: None.

#### 1.2 IAM User Retrieval

- **Overview:** Retrieves all IAM users and enriches each with detailed user attributes.
- **Nodes Involved:** 
  - Get many users
  - Get user
  - Custom HTTP Request (disabled)
  - Sticky Note2
  - Sticky Note3

- **Node Details:**

  - **Get many users**
    - Type: AWS IAM Node (ListUsers operation)
    - Role: Retrieves all IAM users in the AWS account.
    - Configuration: Returns all users in one call (`returnAll: true`), no filters.
    - Credentials: AWS credential scoped to `us-east-1` region (required for IAM).
    - Inputs: From Weekly scheduler.
    - Outputs: Connects to "Get user" and optionally to the disabled "Custom HTTP Request".
    - Edge Cases: AWS permission errors if IAM read permissions missing; network failures.

  - **Get user**
    - Type: AWS IAM Node (GetUser operation)
    - Role: Fetches detailed info per user such as ARN, creation date, password last used.
    - Configuration: Uses dynamic expression to supply username from each user item (`={{ $json.UserName }}`).
    - Credentials: Same AWS credential as above.
    - Inputs: From "Get many users".
    - Outputs: Connects to "Filter bad data".
    - Edge Cases: Potential API throttling; user not found if deleted between calls.

  - **Custom HTTP Request (enable to try)**
    - Type: HTTP Request (disabled)
    - Role: Alternative method to fetch user details via raw AWS IAM API call.
    - Configuration: Uses AWS predefined credential, URL dynamically built with username.
    - Disabled: This node is disabled; included as an example or for testing.
    - Inputs: From "Get many users".
    - Outputs: Connects to "Filter bad data".
    - Edge Cases: Disabled by default; if enabled, requires precise AWS SigV4 signing.

  - **Sticky Note2**
    - Type: Sticky Note
    - Role: Explains the purpose of the "Get many users" node.
  
  - **Sticky Note3**
    - Type: Sticky Note
    - Role: Explains the "Get user" node details and alternative method via HTTP request.

#### 1.3 Data Filtering

- **Overview:** Removes users without valid `PasswordLastUsed` data to avoid false positives and excludes service-linked accounts or malformed records.
- **Nodes Involved:** 
  - Filter bad data
  - Sticky Note4

- **Node Details:**

  - **Filter bad data**
    - Type: Filter Node
    - Role: Ensures each user has a valid `PasswordLastUsed` field.
    - Configuration: Keeps only users where `PasswordLastUsed` exists (strict type validation).
    - Inputs: From "Get user" or "Custom HTTP Request".
    - Outputs: Connects to "IAM user inactive for more than 90 days?".
    - Edge Cases: Users without any login activity (`PasswordLastUsed` null) are excluded here, which might miss some inactive users; can be customized.

  - **Sticky Note4**
    - Type: Sticky Note
    - Role: Explains the rationale behind filtering out irrelevant data.

#### 1.4 Inactivity Check

- **Overview:** Identifies users who have been inactive for more than 90 days by comparing their last password usage date with the current date minus 90 days.
- **Nodes Involved:** 
  - IAM user inactive for more than 90 days?
  - Sticky Note5

- **Node Details:**

  - **IAM user inactive for more than 90 days?**
    - Type: If Node
    - Role: Condition node checking if `PasswordLastUsed` is before the current date minus 90 days.
    - Configuration:
      - Left value: `={{ $json.PasswordLastUsed.toDateTime('s') }}`
      - Right value: `={{ $now.minus(90, 'days') }}`
      - Operator: DateTime "before"
      - Combines conditions with AND (only one condition here).
    - Inputs: From "Filter bad data".
    - Outputs: 
      - True branch: To "Send a message".
      - False branch: To "No Operation, do nothing".
    - Edge Cases: Null or malformed `PasswordLastUsed` would not reach here due to prior filter; time zone differences could affect comparisons.

  - **Sticky Note5**
    - Type: Sticky Note
    - Role: Explains the inactivity check logic.

#### 1.5 Notification

- **Overview:** Sends a Slack message to alert designated users about inactive IAM accounts requiring review.
- **Nodes Involved:** 
  - Send a message
  - Sticky Note6

- **Node Details:**

  - **Send a message**
    - Type: Slack Node
    - Role: Posts a formatted message to Slack alerting about inactive IAM users.
    - Configuration:
      - Message text includes user ARN, username, last activity timestamp formatted to seconds.
      - Uses Slack OAuth2 credentials with a predefined webhook and user selection.
      - Message mentions urgency and instructions for remediation.
    - Inputs: From "IAM user inactive for more than 90 days?" (true branch).
    - Outputs: None.
    - Edge Cases: Slack API rate limits, invalid OAuth token, or revoked permissions can cause failures.

  - **Sticky Note6**
    - Type: Sticky Note
    - Role: Describes the Slack notification purpose and message content.

#### 1.6 No Operation

- **Overview:** Placeholder node for cases where users do not meet inactivity criteria; effectively does nothing.
- **Nodes Involved:** 
  - No Operation, do nothing
  - Sticky Note7

- **Node Details:**

  - **No Operation, do nothing**
    - Type: NoOp Node
    - Role: Ends workflow path for active users with no further action.
    - Inputs: From "IAM user inactive for more than 90 days?" (false branch).
    - Outputs: None.

  - **Sticky Note7**
    - Type: Sticky Note
    - Role: Visual indicator for the "No Operation" node with screenshot for reference.

#### 1.7 Documentation

- **Overview:** Provides detailed instructions, context, and notes for workflow users.
- **Nodes Involved:** 
  - Sticky Note (large, top-left)

- **Node Details:**

  - **Sticky Note**
    - Type: Sticky Note
    - Role: Comprehensive documentation embedded in the workflow.
    - Content Highlights:
      - Workflow purpose, audience, and detailed steps.
      - AWS IAM credential setup requirements (region must be `us-east-1`).
      - Required AWS IAM permissions and Slack bot permissions.
      - Notes on data accuracy and customization tips.
      - Suggestions for audit logging, escalation, multi-account support, and auto-remediation.
      - Warnings about common pitfalls such as null login dates.
    - Inputs/Outputs: None.

---

### 3. Summary Table

| Node Name                          | Node Type            | Functional Role                           | Input Node(s)                | Output Node(s)                | Sticky Note                                                                                       |
|-----------------------------------|----------------------|-----------------------------------------|-----------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Weekly scheduler                  | Schedule Trigger     | Triggers workflow weekly                 | None                        | Get many users               | ### 1. Trigger Workflow Start the workflow on a schedule or manually when you want to check IAM users. |
| Get many users                   | AWS IAM Node         | Lists all IAM users                      | Weekly scheduler            | Get user, Custom HTTP Request | ### 2. List Users Call the `ListUsers` API to retrieve all IAM user names available in the account. |
| Get user                        | AWS IAM Node         | Gets detailed info per user              | Get many users              | Filter bad data              | ### 3. Get User Details For each user, call the `GetUser` API to fetch details including ARN and last password use. |
| Custom HTTP Request (enable to try) | HTTP Request (disabled) | Alternative method to get user details | Get many users              | Filter bad data              | ### 3. Get User Details For each user, call the `GetUser` API to fetch details including ARN and last password use. |
| Filter bad data                 | Filter Node          | Removes users without valid activity data | Get user, Custom HTTP Request | IAM user inactive for more than 90 days? | ### 4. Filter Out Irrelevant Data Remove service-linked accounts or entries without valid activity info. |
| IAM user inactive for more than 90 days? | If Node              | Checks if user inactive > 90 days      | Filter bad data             | Send a message (true), No Operation (false) | ### 5. Identify Inactive Users Compare last activity date with today minus 90 days to find inactive accounts. |
| Send a message                 | Slack Node            | Sends Slack notification for inactive users | IAM user inactive for more than 90 days? (true) | None                        | ### 6. Send Slack Notification Post message to Slack listing inactive user details and instructions. |
| No Operation, do nothing         | NoOp Node            | Ends path for active users (no action)  | IAM user inactive for more than 90 days? (false) | None                        | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-17+at+1.32.23%E2%80%AFPM.png) |
| Sticky Note                     | Sticky Note          | Documentation & instructions             | None                        | None                        | Full embedded documentation with setup, notes, and customization guidance.                       |
| Sticky Note1                    | Sticky Note          | Explains trigger step                    | None                        | None                        | ### 1. Trigger Workflow Start the workflow on a schedule or manually when you want to check IAM users. |
| Sticky Note2                    | Sticky Note          | Explains "Get many users" node           | None                        | None                        | ### 2. List Users Call the `ListUsers` API to retrieve all IAM user names available in the account. |
| Sticky Note3                    | Sticky Note          | Explains "Get user" node                  | None                        | None                        | ### 3. Get User Details For each user, call the `GetUser` API to fetch detailed attributes including ARN, Path, UserId, Creation date, and last password use. |
| Sticky Note4                    | Sticky Note          | Explains filtering logic                  | None                        | None                        | ### 4. Filter Out Irrelevant Data Remove service-linked accounts or entries without valid activity information to avoid noise. |
| Sticky Note5                    | Sticky Note          | Explains inactivity check                 | None                        | None                        | ### 5. Identify Inactive Users Compare the last activity date (`PasswordLastUsed` or key usage) against today minus 90 days. |
| Sticky Note6                    | Sticky Note          | Explains Slack notification step         | None                        | None                        | ### 6. Send Slack Notification Post a message to Slack listing the inactive user(s), including ARN, username, and last activity date, with instructions for review. |
| Sticky Note7                    | Sticky Note          | Visual aid for no operation node          | None                        | None                        | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-17+at+1.32.23%E2%80%AFPM.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**
   - Type: Schedule Trigger
   - Parameters: Set to trigger every 1 week (default weekly interval).
   - Position: Start node.
   - No credentials required.

2. **Create an AWS IAM Node (ListUsers)**
   - Type: AWS IAM Node
   - Operation: `listUsers`
   - Parameters: Return all users (`returnAll: true`).
   - Credentials: AWS credential with region `us-east-1` configured with read-only IAM permissions.
   - Connect Schedule Trigger output to this node.

3. **Create an AWS IAM Node (GetUser)**
   - Type: AWS IAM Node
   - Operation: `getUser`
   - Parameters: UserName set dynamically using expression `={{ $json.UserName }}` from incoming data.
   - Credentials: Same AWS credential as above.
   - Connect output of "Get many users" to this node.

4. **(Optional) Create an HTTP Request Node (disabled)**
   - Type: HTTP Request
   - Parameters: URL set to `https://iam.amazonaws.com/?Action=GetUser&UserName={{ $json.UserName }}&Version=2010-05-08`
   - Authentication: AWS predefined credential (`us-east-1`).
   - Disable this node by default.
   - Connect "Get many users" output here as an alternative to "Get user".

5. **Create a Filter Node**
   - Type: Filter
   - Conditions: Keep only records where `PasswordLastUsed` exists and is a valid date.
   - Connect outputs of both "Get user" and the optional HTTP Request node to this filter node.

6. **Create an If Node**
   - Type: If
   - Condition: Check if `PasswordLastUsed` date is before current date minus 90 days.
     - Left Value: `={{ $json.PasswordLastUsed.toDateTime('s') }}`
     - Operator: DateTime "before"
     - Right Value: `={{ $now.minus(90, 'days') }}`
   - Connect output of Filter node to this If node.

7. **Create a Slack Node**
   - Type: Slack
   - Operation: Send a message (post to channel or user)
   - Parameters:
     - Text:
       ```
       =:warning: *Inactive IAM User Detected* :warning:
       
       The following IAM user has been inactive for more than *90 days*:
       
       *User ARN:* `{{ $json.Arn }}`
       *User Name:* `{{ $json.UserName }}`
       *Last Activity:* {{ $json.PasswordLastUsed.toDateTime('s') }}
       
       Please review this account and take appropriate action (disable access keys, remove user, or re-activate if still needed).
       ```
     - User: Select Slack user by ID or username (e.g., “U054RMBTVBM”)
     - Authentication: OAuth2 credential for Slack bot with permission to post in the channel.
   - Connect the If node’s "true" output to this Slack node.

8. **Create a No Operation Node**
   - Type: NoOp (No operation)
   - Connect the If node’s "false" output here.

9. **Add Sticky Notes (Optional but recommended)**
   - Add sticky notes describing each block and node for clarity and maintainability following the original content.

10. **Configure Credentials**
    - AWS Credential:
      - Service: IAM
      - Region: us-east-1 (mandatory for IAM API)
      - Permissions: Minimum `iam:ListUsers`, `iam:GetUser`.
      - Optionally add `iam:ListAccessKeys`, `iam:GetAccessKeyLastUsed` for enhanced accuracy.
    - Slack OAuth2 Credential:
      - Bot token with permission to post messages in the target Slack channel.

11. **Test Workflow**
    - Manually trigger or wait for scheduled execution.
    - Verify Slack messages for inactive users.
    - Check for errors in AWS API calls or Slack delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| ⚠️ AWS SigV4 for IAM must be scoped to `us-east-1`. Create the AWS credential in n8n with region `us-east-1` even if other services run in different regions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Credential setup note                                                                                         |
| AWS IAM minimum permissions: `iam:ListUsers`, `iam:GetUser`, optionally `iam:ListAccessKeys`, `iam:GetAccessKeyLastUsed` for better inactivity detection.                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | AWS permissions needed                                                                                         |
| Slack bot must have permission to post messages in the target channel.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Slack credential requirement                                                                                   |
| `PasswordLastUsed` is null if the user never signed into the AWS console; consider adding calls to `GetAccessKeyLastUsed` for full activity coverage.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Data accuracy note                                                                                            |
| Customize the inactivity threshold by adjusting the date comparison (e.g., 60, 120, or 180 days).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Customization tip                                                                                              |
| Consider adding audit logging by appending results to Google Sheets or a database with user details and timestamp.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Enhancement suggestion                                                                                        |
| For escalation, add logic to mention `@security` or create tickets if users remain inactive beyond another cycle.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Escalation suggestion                                                                                          |
| Multi-account or multi-region support can be implemented by iterating over multiple AWS credentials, but IAM API region remains `us-east-1`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Multi-account advice                                                                                           |
| Exclude known service accounts by filtering on user tags or a static list.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Filtering enhancement                                                                                          |
| Network egress to `iam.amazonaws.com` is required.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Network requirement                                                                                           |
| Original workflow documentation embedded as a sticky note inside the workflow for easy reference.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Embedded documentation                                                                                        |

---

**Disclaimer:**  
The provided content is extracted exclusively from an automated n8n workflow and complies fully with content policies. No illegal, offensive, or protected elements are included. All data handled is public and legal.