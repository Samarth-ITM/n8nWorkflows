Monitor Domains & IPs on AbuseIPDB Blacklist with Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-domains---ips-on-abuseipdb-blacklist-with-slack-alerts-6836


# Monitor Domains & IPs on AbuseIPDB Blacklist with Slack Alerts

---

### 1. Workflow Overview

This n8n workflow automates the monitoring of specified domains and IP addresses against the AbuseIPDB blacklist service, with immediate Slack notifications when a high-confidence blacklist entry is detected. Its primary use case is to proactively protect email deliverability, sender reputation, and network security by early detection of IPs or domains flagged for abuse.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the blacklist check process.
- **1.2 Asset Definition:** Defines the list of domains and IP addresses to monitor.
- **1.3 AbuseIPDB Querying:** Sends API requests to AbuseIPDB to check blacklist status of each asset.
- **1.4 Blacklist Evaluation:** Conditionally evaluates the abuse confidence score to determine if an alert should be sent.
- **1.5 Alert Dispatch:** Sends a high-priority Slack message if the asset is blacklisted above a threshold.
- **1.6 Documentation/Notes:** Contains extensive sticky notes explaining workflow purpose, use cases, and setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow execution on a fixed schedule, enabling automated, recurring blacklist checks without manual intervention.

- **Nodes Involved:**  
  - Scheduled Check

- **Node Details:**  
  - **Scheduled Check**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a time interval (default configured to trigger every minute/hour/day depending on setup).  
    - Configuration: Uses the built-in scheduling interval rule with no additional filters.  
    - Inputs: None (trigger node)  
    - Outputs: One output connected to the "Define Domains/IPs" node.  
    - Edge Cases: Potential misfire if n8n instance is down at scheduled time; no retry logic configured here.  
    - Version: 1.2  
    - Notes: Acts as the entry point of the workflow.

#### 1.2 Asset Definition

- **Overview:**  
  Defines the fixed list of domains and IP addresses to be monitored. Outputs each asset as an individual item for parallel processing downstream.

- **Nodes Involved:**  
  - Define Domains/IPs

- **Node Details:**  
  - **Define Domains/IPs**  
    - Type: Code (JavaScript)  
    - Role: Creates an array of monitored assets (domains/IPs) and formats them as separate workflow items.  
    - Configuration: Hardcoded array containing `"your-company.com"`, `"mail.your-company.com"`, and `"123.45.67.89"`. Returns each as a JSON object with key `asset`.  
    - Key Expression: Returns `[{ json: { asset: "..." } }, ...]` to split assets.  
    - Inputs: From Scheduled Check  
    - Outputs: To Query Blacklist API  
    - Edge Cases: Hardcoded assets require manual update if assets change; no dynamic input.  
    - Version: 2  
    - Notes: Consider externalizing asset list for scalability.

#### 1.3 AbuseIPDB Querying

- **Overview:**  
  Queries AbuseIPDB’s API for each asset to retrieve abuse confidence scores and blacklist details.

- **Nodes Involved:**  
  - Query Blacklist API

- **Node Details:**  
  - **Query Blacklist API**  
    - Type: HTTP Request  
    - Role: Sends GET requests to AbuseIPDB API endpoint `https://api.abuseipdb.com/api/v2/check` with query parameter `ipAddress` set dynamically to the asset value.  
    - Configuration: URL uses expression to insert `$json.asset`. No additional query parameters or headers shown (assumed to require API key in headers, but not explicitly configured here).  
    - Inputs: From Define Domains/IPs  
    - Outputs: To Is on Blacklist?  
    - Edge Cases: Potential API authentication failure if API key missing/invalid; rate limiting or timeout; malformed asset input causing API errors; asset may be domain or IP but API endpoint expects IP, which could cause errors for domain inputs unless AbuseIPDB supports domain queries.  
    - Version: 4.2  
    - Notes: Verify API key credential configuration in real setup.

#### 1.4 Blacklist Evaluation

- **Overview:**  
  Evaluates the abuse confidence score returned by AbuseIPDB to decide if the asset is significantly blacklisted and requires alerting.

- **Nodes Involved:**  
  - Is on Blacklist?

- **Node Details:**  
  - **Is on Blacklist?**  
    - Type: If (Conditional)  
    - Role: Checks if the `abuseConfidenceScore` from API response is greater than 50.  
    - Configuration: Condition uses expression `={{ $json.abuseConfidenceScore }}` > 50. Uses strict number comparison.  
    - Inputs: From Query Blacklist API  
    - Outputs: True branch to Send High-Priority Alert; false branch is empty (ends workflow).  
    - Edge Cases: Missing or malformed `abuseConfidenceScore` could cause expression failures; score exactly 50 does not trigger alert; no handling for API errors or no data.  
    - Version: 2.2  
    - Notes: Threshold of 50 is arbitrary and can be adjusted.

#### 1.5 Alert Dispatch

- **Overview:**  
  Sends an urgent alert message to a specific Slack user when a blacklist event with high confidence is detected.

- **Nodes Involved:**  
  - Send High-Priority Alert

- **Node Details:**  
  - **Send High-Priority Alert**  
    - Type: Slack (Send Message)  
    - Role: Delivers a formatted message to Slack user ID `123qa` notifying about the blacklisted asset and its confidence score.  
    - Configuration:  
      - Text: Dynamic message including emoji, asset name, and confidence score.  
      - Recipient: Slack user specified by ID mode.  
      - Credentials: Uses Slack API OAuth2 credentials named `temp`.  
    - Inputs: From Is on Blacklist? (true branch)  
    - Outputs: None (terminal node)  
    - Edge Cases: Slack API errors (auth, rate limits, user not found); message formatting errors; credentials must be valid and token active.  
    - Version: 2.3  
    - Notes: Could be extended to notify channels or multiple users.

#### 1.6 Documentation/Notes

- **Overview:**  
  Provides detailed workflow documentation, including problem statement, solution benefits, user personas, scope, and setup instructions, embedded as sticky notes in the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Labels the flow area visually with the title "Flow".  
    - No inputs or outputs.  
  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Contains comprehensive textual documentation describing the workflow’s purpose, problem tackled, solution scope, user roles, and setup steps.  
    - No inputs or outputs.  
    - Content includes:  
      - Problem summary (email deliverability, manual checks inefficiency)  
      - Solution overview (automatic scheduled blacklist checks with alerts)  
      - Target users (SysAdmins, Marketing, Security, IT Support)  
      - Scope limitations (no automatic delisting or root cause analysis)  
      - Detailed setup instructions for monitoring service account creation, asset addition, notifications, and action plans.  
    - Useful for onboarding and operational understanding.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                         | Input Node(s)       | Output Node(s)          | Sticky Note                                                                                      |
|-----------------------|---------------------|---------------------------------------|---------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Scheduled Check       | Schedule Trigger    | Initiate workflow on schedule          | -                   | Define Domains/IPs       |                                                                                                |
| Define Domains/IPs    | Code                | Define assets list for checking        | Scheduled Check     | Query Blacklist API      |                                                                                                |
| Query Blacklist API   | HTTP Request        | Query AbuseIPDB API for blacklist info | Define Domains/IPs  | Is on Blacklist?         |                                                                                                |
| Is on Blacklist?      | If                   | Evaluate confidence score > 50         | Query Blacklist API | Send High-Priority Alert |                                                                                                |
| Send High-Priority Alert | Slack              | Send urgent alert to Slack user        | Is on Blacklist?    | -                       |                                                                                                |
| Sticky Note           | Sticky Note          | Visual label for the workflow area     | -                   | -                       | ## Flow                                                                                        |
| Sticky Note1          | Sticky Note          | Detailed workflow documentation        | -                   | -                       | # Automated Domain/IP Blacklist Monitor problem, solution, users, scope, setup instructions... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Automated Domain/IP Blacklist Monitor with Slack").

2. **Add a "Schedule Trigger" node:**  
   - Set the trigger interval as desired (e.g., every hour or daily).  
   - This node will start the workflow automatically at the configured schedule.

3. **Add a "Code" node named "Define Domains/IPs":**  
   - Set the language to JavaScript.  
   - Insert the following code to define monitored assets:
     ```javascript
     const assetsToCheck = [
         "your-company.com",
         "mail.your-company.com",
         "123.45.67.89"
     ];
     return assetsToCheck.map(asset => ({ json: { asset } }));
     ```
   - Connect the output of "Schedule Trigger" to this node.

4. **Add an "HTTP Request" node named "Query Blacklist API":**  
   - Set the HTTP Method to GET.  
   - For the URL, use the expression:  
     ```
     https://api.abuseipdb.com/api/v2/check?ipAddress={{ $json.asset }}
     ```  
   - **Important:** Configure authentication by adding your AbuseIPDB API key in the headers:  
     - Header key: `Key`  
     - Header value: Your AbuseIPDB API key (stored securely via n8n credentials).  
   - Connect the output of "Define Domains/IPs" to this node.

5. **Add an "If" node named "Is on Blacklist?":**  
   - Configure a condition:  
     - Type: Number  
     - Operation: Greater Than  
     - Value 1 (left side): Expression `={{ $json.abuseConfidenceScore }}`  
     - Value 2 (right side): `50`  
   - Connect the output of "Query Blacklist API" to this "If" node.

6. **Add a "Slack" node named "Send High-Priority Alert":**  
   - Set the Slack node to send a message to a user by ID.  
   - Configure the message text with the expression:  
     ```
     🚨 *URGENT: Blacklist Alert!* 🚨
     Asset *{{ $json.asset }}* has a confidence score of *{{ $json.abuseConfidenceScore }}*. Immediate action is required!
     ```  
   - Configure Slack credentials using OAuth2 with a valid Slack app token that has chat:write permissions.  
   - Enter the Slack user ID (e.g., `123qa`) as the recipient.  
   - Connect the **true** output of "Is on Blacklist?" to this node.

7. **Optionally add Sticky Notes** for documentation and labels directly in the workflow canvas to provide context and instructions to users.

8. **Activate the workflow** once all nodes are connected and credentials are configured.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow does not support automatic delisting or root cause analysis. It is a proactive alerting system only. Manual investigation and action required after alerts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Workflow Scope                                                                                                |
| Slack OAuth2 credentials must be configured with a valid Slack App token that has permissions to send messages to users. See Slack API docs: https://api.slack.com/authentication/oauth-v2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Slack API Authentication                                                                                      |
| AbuseIPDB API requires an API key passed in the HTTP request header named `Key`. Sign up and obtain your key here: https://www.abuseipdb.com/api                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | AbuseIPDB API reference                                                                                       |
| The abuseConfidenceScore threshold of 50 is adjustable based on organizational risk tolerance. Lower values may create more alerts; higher values fewer but more severe alerts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Alert Threshold                                                                                               |
| Domains in asset list may not be supported by AbuseIPDB API (which is IP-focused). Consider resolving domains to IPs before querying or remove domains to avoid API errors.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Asset Input Validation                                                                                        |
| For enhanced robustness, consider adding error handling nodes to catch API errors or timeouts and notify admins accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Workflow Enhancements                                                                                         |
| Monitoring frequency can be tuned in the Schedule Trigger node to balance timely detection and API rate limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Scheduling                                                                                                   |
| Embedded sticky notes provide a comprehensive problem statement, solution benefits, user roles, scope, and detailed setup instructions to facilitate onboarding and maintenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow Documentation                                                                                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---