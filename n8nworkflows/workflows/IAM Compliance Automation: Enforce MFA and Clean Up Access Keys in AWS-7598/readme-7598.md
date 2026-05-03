IAM Compliance Automation: Enforce MFA and Clean Up Access Keys in AWS

https://n8nworkflows.xyz/workflows/iam-compliance-automation--enforce-mfa-and-clean-up-access-keys-in-aws-7598


# IAM Compliance Automation: Enforce MFA and Clean Up Access Keys in AWS

### 1. Workflow Overview

This workflow automates AWS IAM compliance by enforcing Multi-Factor Authentication (MFA) usage and cleaning up access keys for non-compliant users. It targets DevOps, Security, and Cloud Engineers who need continuous, automated monitoring and remediation of IAM user security posture within AWS accounts.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger:** Automatically initiates the workflow daily to ensure ongoing compliance checks.
- **1.2 IAM User Retrieval:** Fetches all IAM users from the AWS account.
- **1.3 MFA Device Verification:** Checks each user for the presence of MFA devices.
- **1.4 Non-Compliant User Filtering:** Filters users without MFA devices.
- **1.5 Slack Notification:** Sends alerts to Slack for users lacking MFA.
- **1.6 Access Key Retrieval:** Retrieves access keys for non-MFA users.
- **1.7 Access Key Parsing:** Extracts and flattens access key metadata.
- **1.8 Active Key Filtering:** Filters out inactive access keys.
- **1.9 Approval and Deactivation:** Requests approval via Slack before deactivating active access keys for non-compliant users.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** Triggers the workflow automatically once per day to enable daily compliance enforcement without manual input.
- **Nodes Involved:**  
  - `Daily scheduler`
- **Node Details:**  
  - **Daily scheduler**  
    - Type: Schedule Trigger  
    - Configuration: Default daily interval trigger (every 24 hours)  
    - Inputs: None (start node)  
    - Outputs: Connects to `Get many users` node  
    - Edge Cases: Scheduler downtime or misconfiguration could delay compliance checks.

#### 1.2 IAM User Retrieval

- **Overview:** Retrieves a complete list of all IAM users in the AWS account for evaluation.
- **Nodes Involved:**  
  - `Get many users`
  - `Sticky Note2`
- **Node Details:**  
  - **Get many users**  
    - Type: AWS IAM Node  
    - Configuration: Uses `ListUsers` API, `returnAll` enabled to fetch all users  
    - Credential: AWS credential scoped to `us-east-1` region  
    - Inputs: Triggered by `Daily scheduler`  
    - Outputs: Provides list of IAM users to `Get IAM User MFA Devices`  
    - Edge Cases: AWS API rate limits, permission errors if credentials lack `iam:ListUsers` rights  
  - **Sticky Note2**  
    - Content: Explains purpose of this node for retrieving all active IAM users.

#### 1.3 MFA Device Verification

- **Overview:** For each user, calls AWS API to list MFA devices, identifying users without MFA.
- **Nodes Involved:**  
  - `Get IAM User MFA Devices`
  - `Sticky Note3`
- **Node Details:**  
  - **Get IAM User MFA Devices**  
    - Type: HTTP Request Node  
    - Configuration: Calls `ListMFADevices` API with `UserName` parameter dynamically injected from each user item  
    - Credential: AWS credential, same as above  
    - Inputs: From `Get many users` node  
    - Outputs: Passes MFA device info to filter node  
    - Edge Cases: API failures, incorrect username interpolation, permission errors without `iam:ListMFADevices` scope  
  - **Sticky Note3**  
    - Content: Clarifies that MFA device presence is critical for compliance.

#### 1.4 Non-Compliant User Filtering

- **Overview:** Filters out users who have at least one MFA device, keeping only those without MFA for further processing.
- **Nodes Involved:**  
  - `Filter out IAM user with MFA device`
- **Node Details:**  
  - **Filter out IAM user with MFA device**  
    - Type: Filter Node  
    - Configuration: Checks if the `MFADevices` array in the AWS API response is empty (no MFA devices)  
    - Inputs: Output from `Get IAM User MFA Devices`  
    - Outputs: Two branches:  
      - True (no MFA devices): proceeds to warning and access key retrieval  
      - False (has MFA): filtered out  
    - Edge Cases: Empty or malformed response, JSON path errors

#### 1.5 Slack Notification

- **Overview:** Sends a Slack alert for each non-MFA user, warning about compliance violation and urging MFA enablement.
- **Nodes Involved:**  
  - `Send warning message(s)`
  - `Sticky Note5`
- **Node Details:**  
  - **Send warning message(s)**  
    - Type: Slack Node  
    - Configuration: Sends a templated message with username and creation date to a configured Slack channel (`it-support` channel ID)  
    - Credential: Slack OAuth2 with `chat:write` permission  
    - Inputs: From true branch of `Filter out IAM user with MFA device`  
    - Outputs: Continues to `Get User Access Key(s)` node for key retrieval  
    - Edge Cases: Slack API errors, invalid channel, missing OAuth scopes  
  - **Sticky Note5**  
    - Content: Explains messaging purpose and Slack alert details.

#### 1.6 Access Key Retrieval

- **Overview:** Fetches all access keys for each IAM user without MFA to identify programmatic access credentials.
- **Nodes Involved:**  
  - `Get User Access Key(s)`
  - `Sticky Note4`
- **Node Details:**  
  - **Get User Access Key(s)**  
    - Type: HTTP Request Node  
    - Configuration: Calls `ListAccessKeys` API using username from `Get many users` node  
    - Credential: AWS credentials, same as prior nodes  
    - Inputs: Follows after Slack warning messages  
    - Outputs: Passes access key data to the parsing code node  
    - Edge Cases: API failures, permissions missing `iam:ListAccessKeys`  
  - **Sticky Note4**  
    - Content: Describes fetching all access keys for non-MFA users.

#### 1.7 Access Key Parsing

- **Overview:** Extracts and flattens access key metadata for each user to facilitate filtering and deactivation steps.
- **Nodes Involved:**  
  - `Parse the list of user access key(s)`
- **Node Details:**  
  - **Parse the list of user access key(s)**  
    - Type: Code Node (JavaScript)  
    - Configuration: Iterates over all incoming data, extracts `UserName`, `AccessKeyId`, `Status`, and converts `CreateDate` from timestamp to ISO string  
    - Inputs: From `Get User Access Key(s)` node  
    - Outputs: Provides structured access key items to filtering node  
    - Edge Cases: Missing or empty access key lists; emits warning if none found  
    - Code snippet:  
      ```js
      const items = await $input.all();
      const results = [];
      for (const item of items) {
        const accessKeys = item.json?.ListAccessKeysResponse?.ListAccessKeysResult?.AccessKeyMetadata || [];
        for (const key of accessKeys) {
          results.push({
            json: {
              UserName: key.UserName,
              AccessKeyId: key.AccessKeyId,
              Status: key.Status,
              CreateDate: new Date(key.CreateDate * 1000).toISOString(),
            }
          });
        }
      }
      return results.length > 0 ? results : [{ json: { warning: 'No access keys found in input data' } }];
      ```
      
#### 1.8 Active Key Filtering

- **Overview:** Filters out inactive access keys, leaving only active keys for potential deactivation.
- **Nodes Involved:**  
  - `Filter out inactive keys`
- **Node Details:**  
  - **Filter out inactive keys**  
    - Type: Filter Node  
    - Configuration: Checks `Status` property equals `"Active"`  
    - Inputs: Output of the parsing code node  
    - Outputs: Passes active keys to Slack approval message node  
    - Edge Cases: No active keys found; workflow gracefully continues without error.

#### 1.9 Approval and Deactivation

- **Overview:** Sends an approval request to a designated Slack user before deactivating each active access key. Upon approval, it calls AWS API to disable the key.
- **Nodes Involved:**  
  - `Send message and wait for response`
  - `Deactivate Access Key(s)`
  - `Sticky Note6`
- **Node Details:**  
  - **Send message and wait for response**  
    - Type: Slack Node  
    - Configuration: Sends an interactive message to a specific Slack user, asking for approval to deactivate the access key; waits up to 60 minutes for response; uses double approval type  
    - Credential: Slack OAuth2  
    - Inputs: From `Filter out inactive keys` node  
    - Outputs: On approval, proceeds to `Deactivate Access Key(s)`  
    - Edge Cases: Slack API failures, timeout without approval, user ignoring request  
  - **Deactivate Access Key(s)**  
    - Type: HTTP Request Node  
    - Configuration: Calls `UpdateAccessKey` API to set the access key status to `Inactive` for the specified user and key  
    - Credential: AWS credentials  
    - Inputs: From Slack approval node  
    - Outputs: None (terminal node)  
    - Edge Cases: AWS API failure, permission issues without `iam:UpdateAccessKey`  
  - **Sticky Note6**  
    - Content: Describes the deactivation step and its security impact.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                            | Input Node(s)                   | Output Node(s)                        | Sticky Note                                               |
|----------------------------|---------------------|------------------------------------------|--------------------------------|-------------------------------------|-----------------------------------------------------------|
| Daily scheduler            | Schedule Trigger    | Triggers workflow daily                   | None                           | Get many users                      | ### 1. Schedule Workflow<br>Triggers the workflow automatically once per day to ensure continuous IAM compliance monitoring without manual intervention. |
| Get many users             | AWS IAM Node        | Retrieves all IAM users                   | Daily scheduler                | Get IAM User MFA Devices            | ### 2. 👥 Get All IAM Users<br>Uses the `ListUsers` API to retrieve all active IAM users in the AWS account. These users will be evaluated for MFA compliance. |
| Get IAM User MFA Devices   | HTTP Request        | Gets MFA device info per user             | Get many users                 | Filter out IAM user with MFA device | ### 3. 🔐 Get IAM User MFA Devices<br>Calls `ListMFADevices` for each user to check if they have at least one MFA device enabled. This is a critical step in identifying users who are not following best security practices. |
| Filter out IAM user with MFA device | Filter             | Keeps only users without MFA              | Get IAM User MFA Devices       | Send warning message(s), Get User Access Key(s) |                                                           |
| Send warning message(s)    | Slack               | Sends Slack alerts for non-MFA users     | Filter out IAM user with MFA device | Get User Access Key(s)              | ### 4. 💬 Send Warning Messages<br>Sends real-time Slack alerts for each non-compliant user, including their username and account creation date. This provides visibility and prompts action before access is revoked. |
| Get User Access Key(s)     | HTTP Request        | Retrieves access keys for non-MFA users  | Send warning message(s)        | Parse the list of user access key(s) | ### 5. 🔎 Get User Access Key(s)<br>For each user without MFA, calls the `ListAccessKeys` API to retrieve all associated access keys that may allow programmatic access to AWS. |
| Parse the list of user access key(s) | Code                | Extracts and flattens access key metadata | Get User Access Key(s)         | Filter out inactive keys            |                                                           |
| Filter out inactive keys   | Filter              | Keeps only active access keys             | Parse the list of user access key(s) | Send message and wait for response |                                                           |
| Send message and wait for response | Slack               | Requests approval in Slack to disable key | Filter out inactive keys       | Deactivate Access Key(s)            |                                                           |
| Deactivate Access Key(s)   | HTTP Request        | Deactivates active access keys            | Send message and wait for response | None                              | ### 6. 🔒 Deactivate Access Key(s)<br>Uses the `UpdateAccessKey` API to set the status of each active access key to `Inactive`. This immediately blocks unauthorized programmatic access for non-MFA users. |
| Sticky Note                | Sticky Note         | Project overview and instructions         | None                          | None                               | # Automated AWS IAM Compliance Workflow for MFA Enforcement and Access Key Deactivation ... [See full content in workflow] |
| Sticky Note1               | Sticky Note         | Describes scheduler node                   | None                          | None                               | ### 1. Schedule Workflow<br>Triggers the workflow automatically once per day to ensure continuous IAM compliance monitoring without manual intervention. |
| Sticky Note2               | Sticky Note         | Describes Get many users node              | None                          | None                               | ### 2. 👥 Get All IAM Users<br>Uses the `ListUsers` API to retrieve all active IAM users in the AWS account. These users will be evaluated for MFA compliance. |
| Sticky Note3               | Sticky Note         | Describes Get IAM User MFA Devices node    | None                          | None                               | ### 3. 🔐 Get IAM User MFA Devices<br>Calls `ListMFADevices` for each user to check if they have at least one MFA device enabled. This is a critical step in identifying users who are not following best security practices. |
| Sticky Note4               | Sticky Note         | Describes Get User Access Key(s) node      | None                          | None                               | ### 5. 🔎 Get User Access Key(s)<br>For each user without MFA, calls the `ListAccessKeys` API to retrieve all associated access keys that may allow programmatic access to AWS. |
| Sticky Note5               | Sticky Note         | Describes Send warning message(s) node     | None                          | None                               | ### 4. 💬 Send Warning Messages<br>Sends real-time Slack alerts for each non-compliant user, including their username and account creation date. This provides visibility and prompts action before access is revoked. |
| Sticky Note6               | Sticky Note         | Describes Deactivate Access Key(s) node    | None                          | None                               | ### 6. 🔒 Deactivate Access Key(s)<br>Uses the `UpdateAccessKey` API to set the status of each active access key to `Inactive`. This immediately blocks unauthorized programmatic access for non-MFA users. |
| Sticky Note7               | Sticky Note         | Screenshot visual aid                       | None                          | None                               | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-19+at+10.41.21%E2%80%AFPM.png) |
| Sticky Note8               | Sticky Note         | Screenshot visual aid                       | None                          | None                               | ![](https://wisestackai.s3.ap-southeast-1.amazonaws.com/Screenshot+2025-08-20+at+10.20.57%E2%80%AFAM.png) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Daily scheduler`  
   - Type: Schedule Trigger  
   - Set to trigger once every 24 hours (default daily interval)  

2. **Create an AWS IAM node to list users**  
   - Name: `Get many users`  
   - Type: AWS IAM  
   - Operation: ListUsers  
   - Set `Return All` to true  
   - Connect AWS credentials with permissions: `iam:ListUsers`  
   - Connect output from `Daily scheduler` to this node  

3. **Create an HTTP Request node to list MFA devices per user**  
   - Name: `Get IAM User MFA Devices`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://iam.amazonaws.com/?Action=ListMFADevices&UserName={{ $json.UserName }}&Version=2010-05-08`  
   - Authentication: Use AWS credentials with `iam:ListMFADevices` permission  
   - Connect output from `Get many users` to this node  

4. **Create a Filter node to keep only users without MFA**  
   - Name: `Filter out IAM user with MFA device`  
   - Condition: Check if the returned `MFADevices` array is empty (i.e., no MFA devices) by using expression:  
     `={{ $json.ListMFADevicesResponse.ListMFADevicesResult.MFADevices.length === 0 }}`  
   - Connect output from `Get IAM User MFA Devices` to this filter  

5. **Create a Slack node to send warning messages for non-MFA users**  
   - Name: `Send warning message(s)`  
   - Type: Slack  
   - Authentication: Configure Slack OAuth2 with `chat:write` permission  
   - Channel: Select appropriate Slack channel (e.g., `it-support`)  
   - Message text:  
     ```
     ⚠️ Security Warning
     The system has detected that user {{ $('Get many users').item.json.UserName }}, created on {{ $('Get many users').item.json.CreateDate.toDateTime('s') }}, does not have an MFA (Multi-Factor Authentication) device enabled.
     Please enable MFA immediately to comply with security best practices.
     ```  
   - Connect "true" output branch from the filter to this node  

6. **Create an HTTP Request node to list access keys for non-MFA users**  
   - Name: `Get User Access Key(s)`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://iam.amazonaws.com/?Action=ListAccessKeys&UserName={{ $json.UserName }}&Version=2010-05-08`  
   - Authentication: AWS credentials with `iam:ListAccessKeys` permission  
   - Connect output from `Send warning message(s)` to this node  

7. **Create a Code node to parse access key data**  
   - Name: `Parse the list of user access key(s)`  
   - Type: Code (JavaScript)  
   - JavaScript code:  
     ```js
     const items = await $input.all();
     const results = [];
     for (const item of items) {
       const accessKeys = item.json?.ListAccessKeysResponse?.ListAccessKeysResult?.AccessKeyMetadata || [];
       for (const key of accessKeys) {
         results.push({
           json: {
             UserName: key.UserName,
             AccessKeyId: key.AccessKeyId,
             Status: key.Status,
             CreateDate: new Date(key.CreateDate * 1000).toISOString(),
           }
         });
       }
     }
     return results.length > 0 ? results : [{ json: { warning: 'No access keys found in input data' } }];
     ```  
   - Connect output from `Get User Access Key(s)` to this node  

8. **Create a Filter node to keep only active keys**  
   - Name: `Filter out inactive keys`  
   - Condition: Check if `Status` equals `"Active"`  
   - Connect output from parsing node to this filter  

9. **Create a Slack node to request approval for key deactivation**  
   - Name: `Send message and wait for response`  
   - Type: Slack  
   - Authentication: Slack OAuth2 with message sending and interactive message permissions  
   - Configure to send a direct message to a specific user (e.g., security admin)  
   - Message template:  
     ```
     ⚠️ *Access Key Deactivation Request*
     User *`{{ $json.UserName }}`* does not have MFA enabled.
     They have active access key(s) that may pose a security risk.
     Do you approve disabling the access key *`{{ $json.AccessKeyId }}`*?
     ```  
   - Set operation to "Send and Wait for Response" with a 60-minute timeout and double approval if desired  
   - Connect output from `Filter out inactive keys` here  

10. **Create an HTTP Request node to deactivate the access key**  
    - Name: `Deactivate Access Key(s)`  
    - Type: HTTP Request  
    - Method: GET  
    - URL:  
      ```
      https://iam.amazonaws.com/?Action=UpdateAccessKey&UserName={{ $json.UserName }}&AccessKeyId={{ $json.AccessKeyId }}&Status=Inactive&Version=2010-05-08
      ```  
    - Authentication: AWS credentials with `iam:UpdateAccessKey` permission  
    - Connect output from Slack approval node to this node  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow is designed to help maintain continuous AWS IAM security compliance by enforcing MFA and disabling programmatic access for users not following best practices. | Workflow Description |
| Video walkthrough available: [Watch the video](https://www.youtube.com/watch?v=ZggCRl8z_gQ) | YouTube Video Link embedded in main Sticky Note |
| AWS IAM permissions required: `iam:ListUsers`, `iam:ListMFADevices`, `iam:ListAccessKeys`, `iam:UpdateAccessKey`. Configure AWS credentials accordingly. | AWS IAM Permissions |
| Slack Bot token must have `chat:write` permission and support interactive messaging for approval flow. | Slack Integration Requirements |
| You can customize by adding whitelists, delaying deactivation, or integrating audit logging (Google Sheets, Airtable, databases). | Customization suggestions in main Sticky Note |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.