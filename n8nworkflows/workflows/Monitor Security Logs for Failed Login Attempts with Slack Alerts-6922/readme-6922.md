Monitor Security Logs for Failed Login Attempts with Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-security-logs-for-failed-login-attempts-with-slack-alerts-6922


# Monitor Security Logs for Failed Login Attempts with Slack Alerts

### 1. Workflow Overview

This workflow is designed to monitor security logs for failed login attempts and send alerts via Slack when suspicious activity exceeds a defined threshold. Its primary use case is early detection of potential brute-force or unauthorized login attempts by analyzing recent log data from an external API. The workflow comprises five logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the workflow on a configurable interval.
- **1.2 Log Retrieval:** Fetches recent log entries from a specified external log server API.
- **1.3 Log Processing:** Filters and analyzes logs to identify failed login attempts and aggregates relevant metrics.
- **1.4 Threshold Evaluation:** Determines if the count of failed logins exceeds a predefined limit.
- **1.5 Alert Notification:** Sends a detailed alert message to a Slack user/channel when anomalies are detected.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the entire workflow on a periodic schedule, ensuring that log analysis runs automatically without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger node (cron-like scheduling)  
    - Configuration: Set to trigger every 1 minute (configurable interval)  
    - Key parameters: Interval of 1 minute  
    - Input: None (start node)  
    - Output: Triggers the next node, "Fetch Logs"  
    - Version-specific: Version 1.2 used; no special version requirements  
    - Potential failures: Misconfiguration of interval, system time errors, or disabled workflow preventing execution  
    - Notes: Responsible for the periodic execution of the workflow

#### 1.2 Log Retrieval

- **Overview:**  
  This block fetches recent log data from an external API endpoint to supply raw log entries for analysis.

- **Nodes Involved:**  
  - Fetch Logs

- **Node Details:**

  - **Fetch Logs**  
    - Type: HTTP Request node  
    - Configuration: Performs a GET request to `https://api.yourlogserver.com/logs/recent?limit=100` to retrieve the latest 100 log entries  
    - Key parameters: URL endpoint; no additional authentication or headers configured by default  
    - Input: Triggered by Schedule Trigger node  
    - Output: Provides raw log data JSON to the "Count Failed Logins" node  
    - Version-specific: Version 4.2 used; ensure compatibility with HTTP request features  
    - Potential failures: Network timeouts, API unavailability, authentication failures if required, malformed JSON responses  
    - Notes: This node assumes the API returns a JSON object with a `data` array containing log entries

#### 1.3 Log Processing

- **Overview:**  
  Processes the retrieved logs to isolate failed login attempts, count them, identify unique IP addresses involved, and summarize the findings.

- **Nodes Involved:**  
  - Count Failed Logins

- **Node Details:**

  - **Count Failed Logins**  
    - Type: Code node (JavaScript execution)  
    - Configuration: Custom JavaScript that:  
      - Filters logs where `event === "login_failure"`  
      - Counts total failed attempts  
      - Identifies unique IP addresses from failed attempts  
      - Extracts timestamps of first and last failed attempts  
      - Constructs a summary string with counts and IPs  
    - Key expressions: Uses `$json.data` to access logs; returns an object with keys: `loginFailureCount`, `uniqueIps`, `firstFailedAttemptTime`, `lastFailedAttemptTime`, `summary`  
    - Input: Receives raw logs JSON from "Fetch Logs"  
    - Output: Passes summarized detection results to the next node  
    - Version-specific: Version 2 used; ensure JavaScript supported and `$json` context available  
    - Potential failures: Unexpected log format, missing `data` property, empty logs, JavaScript errors, index out of range if no failed attempts exist  
    - Notes: Core logic for anomaly detection; must be updated if log structure changes

#### 1.4 Threshold Evaluation

- **Overview:**  
  Evaluates whether the number of failed login attempts exceeds a specific threshold to decide if an alert should be sent.

- **Nodes Involved:**  
  - Failed Logins > Threshold?

- **Node Details:**

  - **Failed Logins > Threshold?**  
    - Type: If node (conditional branching)  
    - Configuration: Checks if `loginFailureCount` is greater than 5  
    - Key expression: `{{$json.loginFailureCount}} > 5`  
    - Input: Receives summary data from "Count Failed Logins"  
    - Output:  
      - True branch: Proceeds to "Send Anomaly Alert"  
      - False branch: Ends workflow silently  
    - Version-specific: Version 2.2 used; supports advanced conditional logic  
    - Potential failures: Missing or invalid `loginFailureCount` value; expression errors  
    - Notes: Threshold value is hardcoded (5) but can be parameterized for flexibility

#### 1.5 Alert Notification

- **Overview:**  
  Sends a formatted alert message to a Slack user or channel if suspicious activity is detected.

- **Nodes Involved:**  
  - Send Anomaly Alert

- **Node Details:**

  - **Send Anomaly Alert**  
    - Type: Slack node  
    - Configuration:  
      - Sends a Slack message via webhook with detailed alert content  
      - Message includes summary, first and last failed attempt timestamps  
      - Message text supports Markdown formatting and emoji  
      - Sends message to a specific user by Slack user ID (`yourid12`)  
    - Key expressions: Uses template expressions like `{{ $json.summary }}`, `{{ $json.firstFailedAttemptTime }}`, `{{ $json.lastFailedAttemptTime }}`  
    - Input: Triggered when threshold condition is true  
    - Output: Ends workflow after sending alert  
    - Credentials: Requires Slack API credential configured with appropriate permissions  
    - Version-specific: Version 2.3; ensure Slack API node supports webhook and user ID targeting  
    - Potential failures: Slack auth errors, webhook misconfiguration, user ID invalid, rate limits, connectivity issues  
    - Notes: Alert format customizable; user ID must be valid Slack user or channel

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                    | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                              |
|-------------------------|---------------------|----------------------------------|---------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger    | Initiates workflow periodically  | None                | Fetch Logs             | ## Flow                                                                                               |
| Fetch Logs             | HTTP Request         | Retrieves recent logs from API   | Schedule Trigger    | Count Failed Logins    | ## Flow                                                                                               |
| Count Failed Logins     | Code                 | Filters and counts failed logins | Fetch Logs          | Failed Logins > Threshold? | # 🕵️ Simple Log Anomaly Detector 🧠<br>* Detects login failures and summarizes them.<br>* Core detection logic. |
| Failed Logins > Threshold? | If                  | Checks if failed logins exceed threshold | Count Failed Logins | Send Anomaly Alert     | # 🕵️ Simple Log Anomaly Detector 🧠<br>* Filters anomalies to avoid false positives.<br>* Threshold = 5 attempts. |
| Send Anomaly Alert      | Slack                | Sends alert to Slack on anomalies | Failed Logins > Threshold? | None                   | # 🕵️ Simple Log Anomaly Detector 🧠<br>* Sends formatted alert to Slack user.<br>* Requires Slack API credentials. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure interval to run every 1 minute (or desired frequency)  
   - No additional parameters needed  
   - This node will start the workflow automatically.

2. **Add the Fetch Logs node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.yourlogserver.com/logs/recent?limit=100` (replace with your actual log server endpoint)  
   - No authentication configured by default; add if your API requires it  
   - Connect Schedule Trigger output to this node’s input.

3. **Add the Count Failed Logins node:**  
   - Type: Code (JavaScript) node  
   - Paste the following JavaScript code into the Code field:

     ```javascript
     const failedLogins = $json.data.filter(log => log.event === 'login_failure');
     const uniqueIps = [...new Set(failedLogins.map(log => log.ip))];
     const loginFailureCount = failedLogins.length;

     return [{
         json: {
             loginFailureCount,
             uniqueIps,
             firstFailedAttemptTime: failedLogins[0]?.timestamp,
             lastFailedAttemptTime: failedLogins[loginFailureCount - 1]?.timestamp,
             summary: `Detected ${loginFailureCount} failed login attempts from ${uniqueIps.length} unique IP(s): ${uniqueIps.join(', ')}`
         }
     }];
     ```

   - Connect Fetch Logs output to this node’s input.

4. **Add the Failed Logins > Threshold? node:**  
   - Type: If node  
   - Configure condition:  
     - Left value: Expression `{{$json.loginFailureCount}}`  
     - Operation: Greater than (>)  
     - Right value: 5 (threshold can be adjusted)  
   - Connect Count Failed Logins output to this node’s input.

5. **Add the Send Anomaly Alert node:**  
   - Type: Slack node  
   - Configure with your Slack API credentials (OAuth2 or webhook)  
   - Set target user or channel by Slack user ID or channel name  
   - Message text (Markdown supported):

     ```
     🚨 *SECURITY ALERT: High Volume of Failed Logins Detected!* 🚨
     Summary: *{{ $json.summary }}*
     First attempt: *{{ $json.firstFailedAttemptTime }}*
     Last attempt: *{{ $json.lastFailedAttemptTime }}*
     ```

   - Connect the "true" output from the If node (Failed Logins > Threshold?) to this Slack node.

6. **Save and activate the workflow.**

7. **Additional configuration notes:**  
   - Ensure your log API endpoint returns JSON with a `data` array containing log objects with at least `event`, `ip`, and `timestamp` fields.  
   - Slack credentials must have permission to send messages to the chosen user or channel.  
   - Adjust schedule and threshold as needed for your environment.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                 | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow provides basic SIEM-like functionality focused on failed login detection. It is not a full security solution but valuable for early anomaly detection.                        | Workflow description                                          |
| Alert messages use Markdown and emojis for clarity and urgency in Slack. Customize messages as needed.                                                                                        | Slack node configuration                                     |
| The workflow can be extended for other anomaly patterns by modifying the code node filtering logic and thresholds.                                                                             | Code node customization                                      |
| Recommended for IT, Security, DevOps teams, and SMEs looking for a lightweight log anomaly detector without heavy SIEM tools.                                                                | Target users                                                 |
| Replace the placeholder API URL and Slack user ID with your actual values before production use.                                                                                              | Configuration instructions                                   |
| For improved security, consider adding authentication to the HTTP Request node if the log API is protected.                                                                                   | Security best practices                                       |
| Slack API credentials must be created and configured in n8n with appropriate scopes to allow message posting.                                                                                  | Slack API setup documentation                                |

---

**Disclaimer:** The text provided is exclusively generated from an automated n8n workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.