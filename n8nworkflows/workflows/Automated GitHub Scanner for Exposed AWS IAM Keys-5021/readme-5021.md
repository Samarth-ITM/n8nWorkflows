Automated GitHub Scanner for Exposed AWS IAM Keys

https://n8nworkflows.xyz/workflows/automated-github-scanner-for-exposed-aws-iam-keys-5021


# Automated GitHub Scanner for Exposed AWS IAM Keys

### 1. Workflow Overview

This workflow, titled **"Automated GitHub Scanner for Exposed AWS IAM Keys"**, is designed to automate the detection of potentially exposed AWS IAM access keys in GitHub repositories associated with an AWS account. It targets security and SecOps teams aiming to continuously monitor and rapidly respond to accidental credential exposures on public or private GitHub repositories.

The workflow consists of the following logical blocks:

- **1.1 Initialization and AWS User Enumeration**  
  Trigger the scan manually, retrieve all AWS IAM users, and prepare for per-user processing.

- **1.2 AWS Access Key Retrieval and Filtering**  
  For each user, fetch their IAM access keys, filtering only those that are active.

- **1.3 GitHub Search Preparation and Query Execution**  
  Generate GitHub search queries for each active access key and query GitHub to find exposed keys in code repositories.

- **1.4 Search Results Aggregation and Compromise Assessment**  
  Aggregate GitHub search results to identify unique exposures and assess whether keys are compromised.

- **1.5 Security Reporting and Notification**  
  Generate detailed security reports based on the findings and format Slack notifications for alerting the security team.

- **1.6 Notification Delivery and Workflow Looping**  
  Send alerts to Slack and continue scanning or prepare for next batch processing.

Optional (disabled) is an automated remediation node to disable compromised keys directly in AWS.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Initialization and AWS User Enumeration

- **Overview:**  
  This block initiates the workflow manually, lists all AWS IAM users, and extracts their usernames to prepare for batch processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - List AWS Users1 (HTTP Request)  
  - Extract AWS Usernames (Code)  
  - Split Users for Processing (Split In Batches)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow on manual execution  
    - *Config:* Default, no parameters  
    - *Input/Output:* No input, output triggers next node  
    - *Failures:* None expected; user-triggered

  - **List AWS Users1**  
    - *Type:* HTTP Request  
    - *Role:* Calls AWS IAM API `ListUsers` to get all IAM users  
    - *Config:* POST to `https://iam.amazonaws.com` with form-urlencoded body (`Action=ListUsers`, `Version=2010-05-08`), using AWS credentials stored in n8n.  
    - *Key Expressions:* None  
    - *Input:* Trigger from manual node  
    - *Output:* JSON response with IAM users list  
    - *Failures:* AWS authentication errors, network issues, API limits  

  - **Extract AWS Usernames**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the AWS API response to extract usernames  
    - *Config:* Extracts array `ListUsersResponse.ListUsersResult.Users`, maps to separate items with `{ username }`  
    - *Input:* JSON from AWS API response  
    - *Output:* Array of items each containing a single username for batch processing  
    - *Failures:* Throws error if response structure is missing or malformed  

  - **Split Users for Processing**  
    - *Type:* Split In Batches  
    - *Role:* Processes usernames in batches for scalability and rate limit management  
    - *Config:* Default batch options  
    - *Input:* Usernames array  
    - *Output:* Batches one username per execution, with two outputs: one for empty batch (no output) and one to continue processing  
    - *Failures:* None expected  

---

#### 2.2 AWS Access Key Retrieval and Filtering

- **Overview:**  
  For each username, this block fetches their IAM access keys and filters only those currently active.

- **Nodes Involved:**  
  - Get User Access Keys (HTTP Request)  
  - Filter Active Keys Only (If)

- **Node Details:**

  - **Get User Access Keys**  
    - *Type:* HTTP Request  
    - *Role:* Calls AWS IAM API `ListAccessKeys` for a specific user  
    - *Config:* POST to `https://iam.amazonaws.com` with form-urlencoded body parameters including the current `username` from batch item, using AWS credentials  
    - *Key Expressions:* Username dynamically set as `={{ $json.username }}`  
    - *Input:* Single username item from batch splitter  
    - *Output:* JSON response containing access key metadata  
    - *Failures:* AWS auth errors, invalid username, network timeouts  

  - **Filter Active Keys Only**  
    - *Type:* If (conditional node)  
    - *Role:* Filters only access keys where `Status` is `Active`  
    - *Config:* Condition checks if the first access key's status equals "Active"  
    - *Key Expressions:* `={{ $json.ListAccessKeysResponse.ListAccessKeysResult.AccessKeyMetadata[0].Status }}`  
    - *Input:* Access key list JSON  
    - *Output:* Passes active keys forward, discards inactive ones  
    - *Failures:* Missing or empty access key list may cause evaluation issues  

---

#### 2.3 GitHub Search Preparation and Query Execution

- **Overview:**  
  Generates GitHub search queries for the active AWS keys and executes search requests against GitHub API to detect exposed keys.

- **Nodes Involved:**  
  - Prepare Github Search (Code)  
  - Rate Limit Wait (Wait)  
  - Search GitHub for Exposed Keys (HTTP Request)

- **Node Details:**

  - **Prepare Github Search**  
    - *Type:* Code (JavaScript)  
    - *Role:* Creates optimized GitHub code search queries targeting the exposed AWS Access Key IDs  
    - *Config:* Extracts accessKeyId and userName from input, forms GitHub search URLs with exact match queries  
    - *Key Expressions:* Uses input JSON to build query string like `"\"${accessKeyId}\" in:file"`  
    - *Input:* Filtered active access key JSON  
    - *Output:* Object with `simpleSearch` (main query), `comprehensiveSearch` (optional extended queries), and recommendation  
    - *Failures:* Handles missing keys gracefully, logs errors on malformed input  

  - **Rate Limit Wait**  
    - *Type:* Wait  
    - *Role:* Throttles requests to GitHub API to avoid hitting rate limits  
    - *Config:* Default wait parameters; webhookId provided but no custom time  
    - *Input:* Search query data  
    - *Output:* Passes forward after delay  
    - *Failures:* Minimal; possible indefinite wait if triggered repeatedly  

  - **Search GitHub for Exposed Keys**  
    - *Type:* HTTP Request  
    - *Role:* Executes the GitHub code search API using the prepared search URL  
    - *Config:* GET request to GitHub search endpoint with authentication via stored GitHub OAuth credentials, 10-second timeout, expects JSON response  
    - *Key Expressions:* URL dynamically set from input `simpleSearch.searchUrl`  
    - *Input:* Search URL from previous node  
    - *Output:* GitHub search results including total count and matched items  
    - *Failures:* GitHub API rate limit errors, auth failures, timeout, or malformed responses  

---

#### 2.4 Search Results Aggregation and Compromise Assessment

- **Overview:**  
  Aggregates all GitHub search results per access key to detect unique exposures and determines if the key is compromised.

- **Nodes Involved:**  
  - Aggregate Search Results (Code)  
  - Check For Compromised Keys (If)  
  - Generate Security Report (Code)

- **Node Details:**

  - **Aggregate Search Results**  
    - *Type:* Code (JavaScript)  
    - *Role:* Collects all search results, removes duplicates by repository and file path, counts total matches, and marks keys as compromised if any matches found  
    - *Config:* Iterates all inputs, flattens items, filters unique by repository fullname and path  
    - *Input:* Array of GitHub search results  
    - *Output:* Single summary object with `accessKeyId`, `userName`, `totalMatches`, `repositories`, and boolean `isCompromised`  
    - *Failures:* Empty or malformed input could cause empty results or errors  

  - **Check For Compromised Keys**  
    - *Type:* If (boolean check)  
    - *Role:* Branches workflow based on whether the access key is compromised (`isCompromised == true`)  
    - *Config:* Condition checks `={{ $json.isCompromised }}` equals true  
    - *Input:* Aggregated search result  
    - *Output:* True branch for compromised keys, false branch for safe keys  
    - *Failures:* If `isCompromised` missing, logic defaults to false/no action  

  - **Generate Security Report**  
    - *Type:* Code (JavaScript)  
    - *Role:* Creates a structured security report object with detailed risk assessment and recommended actions based on compromise status  
    - *Config:* Produces timestamp, risk levels (High/Medium/Low), repository details, and action recommendations for manual review  
    - *Input:* Aggregated search result with compromise status  
    - *Output:* Report object summarizing exposure risk and next steps  
    - *Failures:* Assumes valid input structure; missing data may cause incomplete reports  

---

#### 2.5 Security Reporting and Notification

- **Overview:**  
  Formats the security report into Slack-compatible markdown and sends it as a notification to a configured Slack channel.

- **Nodes Involved:**  
  - Format Slack Alert (Code)  
  - Slack (Slack Node)

- **Node Details:**

  - **Format Slack Alert**  
    - *Type:* Code (JavaScript)  
    - *Role:* Converts security reports into Slack-friendly markdown messages with detailed exposure info and recommended mitigation steps  
    - *Config:* Processes arrays of scan results, handles no-result scenarios, composes alert messages with repository exposure details, risk levels, and action lists  
    - *Input:* Security report JSON objects  
    - *Output:* Object containing `message` string formatted for Slack  
    - *Failures:* Logic handles empty or missing results gracefully; heavy data could cause message truncation issues  

  - **Slack**  
    - *Type:* Slack node  
    - *Role:* Sends formatted alert messages to a specified Slack channel  
    - *Config:* Sends to channel ID `C08Q2H8SRUP` using Slack OAuth2 credentials, message text bound from previous node's `message` field  
    - *Input:* Slack message object  
    - *Output:* Slack API response  
    - *Failures:* Slack auth errors, invalid channel, rate limits, or network issues  

---

#### 2.6 Notification Delivery and Workflow Looping

- **Overview:**  
  Sends Slack alerts and loops back to process next batch of users or keys.

- **Nodes Involved:**  
  - Continue Scanning (No Operation)  
  - Connections looping back to Split Users for Processing

- **Node Details:**

  - **Continue Scanning**  
    - *Type:* No Operation  
    - *Role:* Acts as a placeholder to connect Slack notification back to batch processing start, enabling continuous scanning cycles without interruption  
    - *Config:* No parameters  
    - *Input:* From Slack node  
    - *Output:* Connects to Split Users for Processing to trigger next batch  
    - *Failures:* None expected  

---

#### Optional Node (Disabled):

- **Disable Access Keys** (HTTP Request)  
  - *Role:* Intended to automate disabling compromised AWS IAM access keys immediately after detection  
  - *Status:* Disabled in workflow  
  - *Config:* Sends `UpdateAccessKey` action to AWS IAM API to set status to Inactive for specified key and username  
  - *Security:* Requires `iam:UpdateAccessKey` permission; recommended to add safeguards before enabling  
  - *Note:* Workflow currently only notifies; this node can be enabled for automatic remediation  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                      | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                                   |
|---------------------------|---------------------|-----------------------------------------------------|-------------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Start workflow manually                              | None                                | List AWS Users1                    | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview (covers entire workflow)                     |
| List AWS Users1           | HTTP Request        | Retrieve list of AWS IAM users                       | When clicking ‘Execute workflow’    | Extract AWS Usernames              | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Extract AWS Usernames     | Code (JavaScript)   | Parse and extract usernames from AWS response       | List AWS Users1                     | Split Users for Processing         | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Split Users for Processing | Split In Batches    | Batch process usernames for scalability              | Extract AWS Usernames               | Get User Access Keys (output 1) / (empty batch output 0) | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Get User Access Keys      | HTTP Request        | Fetch IAM access keys for each user                  | Split Users for Processing (output 1) | Filter Active Keys Only            | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Filter Active Keys Only   | If                  | Pass only access keys with status "Active"           | Get User Access Keys               | Prepare Github Search              | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Prepare Github Search     | Code (JavaScript)   | Generate GitHub search queries for AWS keys          | Filter Active Keys Only             | Rate Limit Wait                   | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Rate Limit Wait           | Wait                | Prevent hitting GitHub API rate limits                | Prepare Github Search              | Search GitHub for Exposed Keys     | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Search GitHub for Exposed Keys | HTTP Request    | Execute GitHub code search API                         | Rate Limit Wait                   | Aggregate Search Results          | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Aggregate Search Results  | Code (JavaScript)   | Aggregate and deduplicate GitHub search results       | Search GitHub for Exposed Keys     | Check For Compromised Keys         | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Check For Compromised Keys| If                  | Branch flow if keys are compromised                    | Aggregate Search Results           | Generate Security Report           | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Generate Security Report  | Code (JavaScript)   | Create detailed report and risk assessment             | Check For Compromised Keys (true)  | Format Slack Alert                | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Format Slack Alert        | Code (JavaScript)   | Format report into Slack markdown notification         | Generate Security Report           | Slack                            | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Slack                    | Slack               | Send alert message to Slack channel                     | Format Slack Alert                | Continue Scanning                 | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Continue Scanning         | No Operation        | Loop back to batch processing for continuous scanning | Slack                            | Split Users for Processing         | # 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview                                              |
| Disable Access Keys       | HTTP Request (disabled) | Automate disabling compromised AWS access keys       | (Not connected)                   | None                             | ## 🚨 Automate Disable Access Keys — instructions and security notes for enabling this node                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No special parameters  

2. **Add HTTP Request Node to List AWS IAM Users**  
   - Name: `List AWS Users1`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://iam.amazonaws.com`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Body Parameters:  
     - `Action` = `ListUsers`  
     - `Version` = `2010-05-08`  
   - Authentication: Use AWS credentials stored in n8n with permissions to call IAM ListUsers API  

3. **Add Code Node to Extract Usernames**  
   - Name: `Extract AWS Usernames`  
   - Type: Code  
   - Language: JavaScript  
   - Paste script that extracts `UserName` from `ListUsersResponse` and outputs array of usernames each as a separate item  

4. **Add Split In Batches Node**  
   - Name: `Split Users for Processing`  
   - Type: Split In Batches  
   - Default batch size (1 recommended for granular control)  

5. **Add HTTP Request Node to Get User Access Keys**  
   - Name: `Get User Access Keys`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://iam.amazonaws.com`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Body Parameters:  
     - `Action` = `ListAccessKeys`  
     - `Version` = `2010-05-08`  
     - `UserName` = dynamic expression: `{{$json["username"]}}`  
   - Authentication: AWS credentials same as before  

6. **Add If Node to Filter Active Keys**  
   - Name: `Filter Active Keys Only`  
   - Type: If  
   - Condition: Check if first access key's status equals `"Active"`  
   - Expression example: `{{$json["ListAccessKeysResponse"]["ListAccessKeysResult"]["AccessKeyMetadata"][0]["Status"]}} == "Active"`  

7. **Add Code Node to Prepare GitHub Search Query**  
   - Name: `Prepare Github Search`  
   - Type: Code (JavaScript)  
   - Paste script that extracts access key ID and username from AWS API response and generates GitHub code search URLs with exact match queries  

8. **Add Wait Node to Handle Rate Limiting**  
   - Name: `Rate Limit Wait`  
   - Type: Wait  
   - Default parameters (adjust delay as needed based on GitHub rate limits)  

9. **Add HTTP Request Node to Search GitHub**  
   - Name: `Search GitHub for Exposed Keys`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: dynamic from previous node: `{{$json["simpleSearch"]["searchUrl"]}}`  
   - Authentication: GitHub OAuth credentials configured in n8n  
   - Timeout: 10 seconds  
   - Response format: JSON  

10. **Add Code Node to Aggregate Search Results**  
    - Name: `Aggregate Search Results`  
    - Type: Code  
    - JavaScript to merge results, remove duplicates by repository and file path, count matches, and set compromised flag  

11. **Add If Node to Check for Compromised Keys**  
    - Name: `Check For Compromised Keys`  
    - Type: If  
    - Condition: Boolean check if `isCompromised` is true (`{{$json["isCompromised"]}} == true`)  

12. **Add Code Node to Generate Security Report**  
    - Name: `Generate Security Report`  
    - Type: Code  
    - JavaScript that creates a detailed report object with risk levels, recommended actions, and status based on compromise  

13. **Add Code Node to Format Slack Notification**  
    - Name: `Format Slack Alert`  
    - Type: Code  
    - Script to convert security report into Slack markdown message with exposures, risk indicators, and action items  

14. **Add Slack Node to Send Notification**  
    - Name: `Slack`  
    - Type: Slack  
    - Configure OAuth2 Slack credentials  
    - Channel: Set channel ID where alerts should be posted  
    - Message text: `={{ $json.message }}`  

15. **Add No Operation Node to Loop Back**  
    - Name: `Continue Scanning`  
    - Type: No Operation  
    - Connect output to `Split Users for Processing` to enable continuous batch scanning  

16. **Connect Nodes in the Following Order:**  
    - Manual Trigger → List AWS Users1 → Extract AWS Usernames → Split Users for Processing → Get User Access Keys → Filter Active Keys Only → Prepare Github Search → Rate Limit Wait → Search GitHub for Exposed Keys → Aggregate Search Results → Check For Compromised Keys → Generate Security Report → Format Slack Alert → Slack → Continue Scanning → Split Users for Processing  

17. **Credentials Setup:**  
    - AWS: Configure AWS credentials with permissions for `ListUsers` and `ListAccessKeys`  
    - GitHub: OAuth credentials with scope to access GitHub code search API  
    - Slack: OAuth credentials with permission to post messages in chosen Slack channel  

18. **Optional (Disabled) Automated Remediation Node:**  
    - Add HTTP Request node `Disable Access Keys` configured to call IAM `UpdateAccessKey` with `Status=Inactive` for compromised keys  
    - Connect after detection logic and before notification if automation is desired  
    - Ensure AWS credentials have `iam:UpdateAccessKey` permission and consider safeguards before enabling  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| 🔐🕵️‍♂️ GitHub Scanner for Exposed AWS IAM Keys – Quick Overview: Automates discovery, filtering, risk assessment, alerts via Slack, remediation guidance, and continuous monitoring of exposed AWS IAM keys on GitHub. Helps security teams identify and respond to credential exposures quickly.                                                                                                                                                                                          | Sticky Note covering entire workflow overview                                                           |
| 🚨 Automate Disable Access Keys: Instructions for configuring automatic disabling of compromised AWS keys, prerequisites, security considerations, and integration advice. Enables immediate remediation by disabling exposed keys after detection.                                                                                                                                                                                                                                              | Sticky Note near disabled `Disable Access Keys` node                                                    |
| GitHub Search API documentation for code search: https://docs.github.com/en/rest/search#search-code                                                                                                                                                                                                                                                                                                                                                                                             | Reference for GitHub search API                                                                          |
| AWS IAM API Reference: https://docs.aws.amazon.com/IAM/latest/APIReference/API_ListUsers.html and https://docs.aws.amazon.com/IAM/latest/APIReference/API_ListAccessKeys.html                                                                                                                                                                                                                                                                                                                     | AWS IAM API docs                                                                                        |
| Slack API for posting messages with OAuth tokens: https://api.slack.com/messaging/webhooks                                                                                                                                                                                                                                                                                                                                                                                                      | Slack integration documentation                                                                          |

---

This comprehensive reference enables security engineers and developers to understand, reproduce, and maintain the automated GitHub scanning workflow for exposed AWS IAM keys, with clear functional segmentation, node explanations, and integration points.