Automated Failed Login Detection with Jira Tasks, Slack Alerts & Notion Logging

https://n8nworkflows.xyz/workflows/automated-failed-login-detection-with-jira-tasks--slack-alerts---notion-logging-11220


# Automated Failed Login Detection with Jira Tasks, Slack Alerts & Notion Logging

### 1. Workflow Overview

This workflow automates the detection and handling of failed login attempts reported by an external application. It is designed to improve security monitoring by:

- Receiving and normalizing failed login data.
- Validating the presence of essential information (username and IP).
- Detecting whether there are multiple failed attempts from the same username and IP within a short timeframe.
- Creating Jira security tasks to track each incident.
- Sending detailed Slack notifications to alert security teams.
- Logging the failed attempts into a Notion database for record-keeping.

The workflow is logically divided into these blocks:

**1.1 Input Reception and Normalization**  
Receives failed login events via webhook, normalizes incoming data into a consistent format.

**1.2 Validation and Alerts for Missing Data**  
Verifies presence of username and IP; triggers Slack alert if missing.

**1.3 Multiple Attempts Detection**  
Groups and analyzes attempts within a 5-minute window to detect multiple failed attempts from the same user and IP.

**1.4 Jira Task Creation and Slack Notifications**  
Creates Jira issues for single or multiple failed attempts; sends corresponding Slack alerts with details.

**1.5 Post-Processing: Data Formatting and Storage**  
Formats message content from Slack alerts and logs the structured data into a Notion database for auditing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Normalization

- **Overview:**  
Receives failed login POST requests at a webhook endpoint and normalizes the input data, ensuring consistent structure for downstream processing.

- **Nodes Involved:**  
  - Faield Login Trigger (Webhook)  
  - Normalize Login Event (Function)

- **Node Details:**

  - **Faield Login Trigger**  
    - Type: Webhook  
    - Role: Entry point; listens for HTTP POST requests at path `/failed-login`.  
    - Configuration: POST method, response mode set to last node output.  
    - Input: External HTTP POST from application sending failed login data.  
    - Output: Raw event data forwarded to normalization.  
    - Edge Cases: Missing or malformed HTTP requests; webhook authorization not configured here.

  - **Normalize Login Event**  
    - Type: Function  
    - Role: Converts input payload (object or array) into normalized items with fields: username, ip, timestamp, error, and a composed logMessage.  
    - Configuration: JavaScript code extracts and formats fields, defaults timestamp to current time if missing, sets a generic error message if not provided.  
    - Input: Raw JSON body from webhook.  
    - Output: Array of normalized JSON items, one per login attempt.  
    - Edge Cases: Empty or invalid input body; missing fields handled via defaults.

#### 2.2 Validation and Alerts for Missing Data

- **Overview:**  
Checks presence of both username and IP. If either is missing, immediately sends a Slack alert to notify security personnel.

- **Nodes Involved:**  
  - Check Username & IP present (IF)  
  - Slack Alert - Missing Username/IP

- **Node Details:**

  - **Check Username & IP present**  
    - Type: IF (Boolean condition)  
    - Role: Evaluates if both username and IP are non-empty strings.  
    - Configuration: Expression checks `$json["username"] !== '' && $json["ip"] !== ''`.  
    - Input: Normalized login event.  
    - Output: True branch proceeds to multiple attempts detection; False branch triggers Slack alert.  
    - Edge Cases: Empty strings or missing fields cause false branch.

  - **Slack Alert - Missing Username/IP**  
    - Type: Slack  
    - Role: Sends a formatted notification to Slack channel about missing username or IP in a failed login attempt.  
    - Configuration: Message template includes fallback values for username and IP (`Unknown`) and shows timestamp and error; posts to configured Slack channel using OAuth2 credentials.  
    - Input: Items failing the username/IP check.  
    - Output: Slack message delivery confirmation.  
    - Edge Cases: Slack API failures, invalid credentials, channel missing.

#### 2.3 Multiple Attempts Detection

- **Overview:**  
Analyzes recent failed login attempts to detect multiple attempts from the same username and IP within a 5-minute sliding window.

- **Nodes Involved:**  
  - Detect Multiple Attempts (Function)  
  - IF - Multiple Attempts (IF)  

- **Node Details:**

  - **Detect Multiple Attempts**  
    - Type: Function  
    - Role: Groups all input items by username+IP, sorts by timestamp, and finds the largest cluster of attempts within a 5-minute window. Outputs grouped data flagging multiple attempts or single attempts.  
    - Configuration: Uses JavaScript to implement sliding window logic.  
    - Input: Filtered normalized data (username and IP present).  
    - Output: Items tagged with `multiple: true` if multiple attempts detected; otherwise `multiple: false`.  
    - Edge Cases: Incorrect timestamps, empty input sets, time zone inconsistencies.

  - **IF - Multiple Attempts**  
    - Type: IF (Boolean condition)  
    - Role: Routes flow based on whether multiple attempts were detected (`multiple === true`).  
    - Configuration: Expression compares `$json["multiple"] === true`.  
    - Input: Output from detection function.  
    - Output: True branch for multiple attempts summary creation; False branch for single attempt ticket creation.

#### 2.4 Jira Task Creation and Slack Notifications

- **Overview:**  
Creates Jira tickets for failed login attempts (single or multiple) and sends Slack alerts with the relevant details for security team awareness.

- **Nodes Involved:**  
  - Build Multi-Attempt Summary (Function)  
  - Create Ticket - Single Attempt (Jira)  
  - Create Ticket- Multiple Attempts (Jira)  
  - Format Fields For Single Attempt (Set)  
  - Format Fields for Multiple-attempt (Set)  
  - Slack Alert - Single Attempt  
  - Slack Alert- Multiple Attempts  

- **Node Details:**

  - **Build Multi-Attempt Summary**  
    - Type: Function  
    - Role: Constructs a detailed summary message for multiple attempts including count, timestamps, errors for use in Jira description and Slack messages.  
    - Configuration: Aggregates `attempts` array; formats readable message string.  
    - Input: Items tagged as multiple attempts.  
    - Output: JSON with summaryMessage, username, ip, count, and attempts array.  
    - Edge Cases: Empty attempts array, malformed timestamps.

  - **Create Ticket - Single Attempt**  
    - Type: Jira (Cloud API)  
    - Role: Creates a Jira task for a single failed login attempt.  
    - Configuration: Project and issue type selected from Jira instance; summary includes username; description includes logMessage or first attempt's logMessage.  
    - Input: Single attempt items.  
    - Output: Jira issue key and details.  
    - Credentials: Jira Software Cloud OAuth2.  
    - Edge Cases: Jira API errors, permission issues, project or issue type misconfiguration.

  - **Create Ticket- Multiple Attempts**  
    - Type: Jira (Cloud API)  
    - Role: Creates detailed Jira task for multiple failed attempts with summary description.  
    - Configuration: Similar to single attempt but description uses multi-attempt summary message.  
    - Credentials: Same as above.  
    - Edge Cases: Same as above.

  - **Format Fields For Single Attempt**  
    - Type: Set  
    - Role: Adds fields to structure data for Slack alert after single ticket creation. Includes username, ip, totalAttempts=1, attemptType, attemptList.  
    - Input: Output of single attempt ticket creation.  
    - Output: Structured JSON with formatted fields.

  - **Format Fields for Multiple-attempt**  
    - Type: Set  
    - Role: Similar to single attempt formatting but for multiple attempts; sets totalAttempts count and constructs attemptList string from summary.  
    - Input: Output of multiple attempt ticket creation.  
    - Output: Structured JSON for Slack alert.

  - **Slack Alert - Single Attempt**  
    - Type: Slack  
    - Role: Sends Slack message for single failed login attempt; includes Jira ticket link from issue key.  
    - Configuration: Posts to configured Slack channel; message templates use dynamic fields.  
    - Credentials: Slack OAuth2.  
    - Edge Cases: Slack API failures.

  - **Slack Alert- Multiple Attempts**  
    - Type: Slack  
    - Role: Sends Slack message for multiple failed login attempts with detailed summary and Jira ticket link.  
    - Configuration: Similar to single attempt alert but includes multiple attempt details.  
    - Credentials: Same as above.  
    - Edge Cases: Same as above.

#### 2.5 Post-Processing: Data Formatting and Storage

- **Overview:**  
Formats Slack alert message content to extract structured data fields and logs these details into a Notion database for auditing and tracking.

- **Nodes Involved:**  
  - Format Login Attempt (Code)  
  - Login Attempts Data Store in DB (Notion)  

- **Node Details:**

  - **Format Login Attempt**  
    - Type: Code (JavaScript)  
    - Role: Parses Slack message blocks JSON to extract username, IP, totalAttempts, attemptType, attemptList, and Jira URL.  
    - Configuration: Uses regular expressions on Slack message blocks to extract key-value pairs.  
    - Input: Slack alert message JSON.  
    - Output: Structured JSON with extracted fields and raw text.  
    - Edge Cases: Slack message format changes, missing fields, regex failures.

  - **Login Attempts Data Store in DB**  
    - Type: Notion  
    - Role: Adds a page to a configured Notion database with extracted login attempt data for tracking.  
    - Configuration: Maps fields like UserName, IP, TotalAttempts, AttemptList, AttemptType to Notion database properties.  
    - Credentials: Notion API OAuth2 connection.  
    - Edge Cases: Notion API rate limits, missing database permissions.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                                    | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                   |
|-----------------------------|---------------------|--------------------------------------------------|----------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------|
| Faield Login Trigger         | Webhook             | Receives failed login events from external app. | (Start)                         | Normalize Login Event                 | Receives the failed-login data from your application and starts the workflow.                |
| Normalize Login Event        | Function            | Normalizes input data into consistent format.   | Faield Login Trigger             | Check Username & IP present           | This node cleans and format the login data                                                    |
| Check Username & IP present  | IF                  | Checks if username and IP are present.           | Normalize Login Event            | Detect Multiple Attempts / Slack Alert - Missing Username/IP | ## Security Login Alert Process This process for checks failed login event. If the username or IP address is missing, it immediately sends a Slack alert. If both are present, it checks whether the same user and IP tried to log in multiple times. If it’s only one attempt, it creates a Jira task and sends a Slack message. If there are multiple attempts, it prepares a summary, creates a detailed Jira task, and sends a Slack alert with the summary. |
| Slack Alert - Missing Username/IP | Slack               | Alerts for missing username or IP.                | Check Username & IP present      | -                                    | (See above)                                                                                  |
| Detect Multiple Attempts     | Function            | Detects repeated attempts within 5 minutes.      | Check Username & IP present      | IF - Multiple Attempts               | (See above)                                                                                  |
| IF - Multiple Attempts       | IF                  | Routes based on single or multiple attempts.     | Detect Multiple Attempts         | Build Multi-Attempt Summary / Create Ticket - Single Attempt | (See above)                                                                                  |
| Build Multi-Attempt Summary  | Function            | Builds detailed summary for multiple attempts.   | IF - Multiple Attempts           | Create Ticket- Multiple Attempts      | (See above)                                                                                  |
| Create Ticket - Single Attempt | Jira                | Creates Jira task for single failed login.       | IF - Multiple Attempts           | Format Fields For Single Attempt      | (See above)                                                                                  |
| Create Ticket- Multiple Attempts | Jira                | Creates Jira task for multiple failed attempts.  | Build Multi-Attempt Summary      | Format Fields for Multiple-attempt    | (See above)                                                                                  |
| Format Fields For Single Attempt | Set                 | Adds fields for Slack alert after single ticket. | Create Ticket - Single Attempt   | Slack Alert - Single Attempt          | (See above)                                                                                  |
| Format Fields for Multiple-attempt | Set                 | Adds fields for Slack alert after multiple ticket.| Create Ticket- Multiple Attempts | Slack Alert- Multiple Attempts        | (See above)                                                                                  |
| Slack Alert - Single Attempt | Slack               | Sends Slack notification for single attempt.     | Format Fields For Single Attempt | Format Login Attempt                  | (See above)                                                                                  |
| Slack Alert- Multiple Attempts | Slack               | Sends Slack notification for multiple attempts.  | Format Fields for Multiple-attempt | Format Login Attempt                | (See above)                                                                                  |
| Format Login Attempt         | Code                | Extracts structured data from Slack message.     | Slack Alert - Single Attempt / Slack Alert- Multiple Attempts | Login Attempts Data Store in DB        | ## Format data & Store to Database In this Process format the login attempts data in proper manner and add to the Database            |
| Login Attempts Data Store in DB | Notion              | Logs login attempt data into Notion database.    | Format Login Attempt             | -                                    | (See above)                                                                                  |
| Sticky Note                 | Sticky Note          | Receives the failed-login data from your application and starts the workflow. | -                                | -                                    | Receives the failed-login data from your application and starts the workflow.                |
| Sticky Note1                | Sticky Note          | Describes normalization node function.           | -                                | -                                    | This node cleans and format the login data                                                  |
| Sticky Note2                | Sticky Note          | Explains overall security login alert process.   | -                                | -                                    | Security Login Alert Process explanation                                                    |
| Sticky Note3                | Sticky Note          | Provides detailed workflow overview and setup.   | -                                | -                                    | How It Works and Setup Steps                                                                |
| Sticky Note4                | Sticky Note          | Describes data formatting and storage block.     | -                                | -                                    | Format data & Store to Database explanation                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: `Faield Login Trigger`  
   - HTTP Method: POST  
   - Path: `failed-login`  
   - Response Mode: Last Node Output  
   - Purpose: Receive failed login events from your application.

2. **Add Function Node to Normalize Input:**  
   - Type: Function  
   - Name: `Normalize Login Event`  
   - Code: Normalize single object or array input, extract `username`, `ip`, `timestamp` (default to now), `error` (default 'Login failed'), and create `logMessage`.  
   - Connect output of webhook to this node.

3. **Add IF Node to Check Username and IP Presence:**  
   - Type: IF  
   - Name: `Check Username & IP present`  
   - Condition: Boolean expression: `{{$json["username"] !== '' && $json["ip"] !== ''}}`  
   - Connect output of normalization node here.

4. **Add Slack Node for Alerts on Missing Data:**  
   - Type: Slack  
   - Name: `Slack Alert - Missing Username/IP`  
   - Channel: Choose your security notification channel.  
   - Message: Alert with username, IP (or Unknown), timestamp, and error.  
   - Connect False output of IF node to this Slack node.  
   - Configure Slack OAuth2 credentials.

5. **Add Function Node to Detect Multiple Attempts:**  
   - Type: Function  
   - Name: `Detect Multiple Attempts`  
   - Code: Group attempts by username and IP and detect clusters within 5 minutes window, marking items as `multiple: true` or `false`.  
   - Connect True output of IF node here.

6. **Add IF Node to Check Multiple Attempts:**  
   - Type: IF  
   - Name: `IF - Multiple Attempts`  
   - Condition: Boolean expression: `{{$json["multiple"] === true}}`  
   - Connect output of detection node here.

7. **Add Function Node to Build Summary for Multiple Attempts:**  
   - Type: Function  
   - Name: `Build Multi-Attempt Summary`  
   - Code: Compose a detailed summary message listing all attempts with timestamps and errors.  
   - Connect True output of IF node here.

8. **Add Jira Node to Create Ticket for Single Attempt:**  
   - Type: Jira  
   - Name: `Create Ticket - Single Attempt`  
   - Configure project and issue type.  
   - Summary: `[Security Alert] Failed Login: {{$json.username}}`  
   - Description: Use the first attempt’s `logMessage`.  
   - Connect False output of IF node to this node.  
   - Configure Jira OAuth2 credentials.

9. **Add Jira Node to Create Ticket for Multiple Attempts:**  
   - Type: Jira  
   - Name: `Create Ticket- Multiple Attempts`  
   - Configure project and issue type as above.  
   - Summary: `[Security Alert] Multiple Failed Login Attempts: {{$json.username}}`  
   - Description: Use summary message from previous function node.  
   - Connect output of summary builder to this node.

10. **Add Set Node to Format Fields for Single Attempt Slack Alert:**  
    - Type: Set  
    - Name: `Format Fields For Single Attempt`  
    - Fields: username, ip, totalAttempts = 1, attemptList string with timestamp and error, attemptType = "single".  
    - Connect output of single ticket Jira node.

11. **Add Set Node to Format Fields for Multiple Attempts Slack Alert:**  
    - Type: Set  
    - Name: `Format Fields for Multiple-attempt`  
    - Fields: username, ip, totalAttempts = count, attemptList string from attempts array, attemptType = "multiple".  
    - Connect output of multiple ticket Jira node.

12. **Add Slack Node for Single Attempt Alert:**  
    - Type: Slack  
    - Name: `Slack Alert - Single Attempt`  
    - Channel: Same as before.  
    - Message: Detailed alert including Jira ticket link using `{{$json.key}}`.  
    - Connect output of single attempt format node.

13. **Add Slack Node for Multiple Attempts Alert:**  
    - Type: Slack  
    - Name: `Slack Alert- Multiple Attempts`  
    - Channel: Same as before.  
    - Message: Detailed alert including Jira ticket link using `{{$node["Create Ticket- Multiple Attempts"].json.key}}`.  
    - Connect output of multiple attempt format node.

14. **Add Code Node to Parse Slack Alert Messages:**  
    - Type: Code  
    - Name: `Format Login Attempt`  
    - Code: Extract fields (username, IP, totalAttempts, attemptType, attemptList, Jira URL) by parsing Slack message JSON blocks using regex.  
    - Connect output of both Slack alert nodes here.

15. **Add Notion Node to Store Login Attempt Data:**  
    - Type: Notion  
    - Name: `Login Attempts Data Store in DB`  
    - Database: Select your Notion database for login failures.  
    - Map properties: UserName (title), IP (rich text), TotalAttempts (number), AttemptList (rich text), AttemptType (rich text).  
    - Connect output of format login attempt code node.  
    - Configure Notion OAuth2 credentials.

16. **Add Sticky Notes:** (Optional) Add descriptive sticky notes near logical blocks for documentation and clarity.

17. **Test Workflow:**  
    - Send sample POST requests with failed login data containing various cases: missing username/IP, single attempts, multiple attempts.  
    - Verify Jira tickets, Slack messages, and Notion entries.

18. **Enable Workflow:**  
    - Once verified, activate the workflow to run continuously.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow is designed to improve security response by automating the detection of suspicious login activity and maintaining centralized records in Jira and Notion, while alerting teams via Slack.                                                                      | General workflow objective                                                                              |
| Setup instructions emphasize adding the webhook URL to your application and correctly configuring Jira, Slack, and Notion credentials. Testing with varied data inputs ensures robustness.                                                                                  | Sticky Note3 content                                                                                     |
| Slack message parsing relies on Slack Block Kit JSON structure; changes to Slack message formatting may require updates to the parsing code node.                                                                                                                        | Format Login Attempt node details                                                                        |
| Jira project and issue type IDs are specific to the configured Jira instance; these must be updated to match your environment.                                                                                                                                             | Jira nodes configuration                                                                                 |
| Notion database must have properties matching the field names used in the workflow; otherwise, data storage will fail.                                                                                                                                                    | Notion node configuration                                                                                 |
| Slack API credentials must have permissions to post messages in the selected channel.                                                                                                                                                                                       | Slack node credentials                                                                                     |
| For advanced customization, consider expanding detection window or adding filtering criteria for suspicious patterns.                                                                                                                                                    | Potential workflow enhancements                                                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.