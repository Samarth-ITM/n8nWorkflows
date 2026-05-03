SSL/TLS Certificate Expiry Monitor with Slack Alert

https://n8nworkflows.xyz/workflows/ssl-tls-certificate-expiry-monitor-with-slack-alert-6957


# SSL/TLS Certificate Expiry Monitor with Slack Alert

### 1. Workflow Overview

This workflow automates the monitoring of SSL/TLS certificate expiry dates for a predefined list of domains and proactively sends alerts to a designated Slack channel when certificates are nearing expiration. It is designed to help system administrators, DevOps teams, web developers, and managed service providers avoid unexpected downtime and security risks caused by expired certificates.

The main logical blocks are:

- **1.1 Scheduled Trigger:** Initiates the workflow execution on a customizable periodic schedule (e.g., weekly).
- **1.2 Domain Enumeration:** Defines and outputs the list of domains whose certificates are to be monitored.
- **1.3 Certificate Expiry Retrieval:** Queries an external certificate information API to get the expiry details for each domain.
- **1.4 Expiry Evaluation:** Checks if each certificate is expiring within a critical time window (default 30 days).
- **1.5 Slack Alert Dispatch:** Sends a high-priority alert message to a specific Slack channel if a certificate is close to expiry.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block starts the workflow automatically on a recurring schedule, enabling continuous and unattended monitoring.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Scheduled Trigger  
    - Configuration: Default schedule set to trigger periodically (interval configured but unspecified in detail, typically weekly recommended)  
    - Inputs: None (start node)  
    - Outputs: Triggers the "List Domains to Monitor" node  
    - Potential Failures: Misconfiguration of schedule or n8n service downtime could prevent triggering  
    - Version-specific: Uses typeVersion 1.2, compatible with n8n versions supporting cron/schedule triggers

#### 1.2 Domain Enumeration

- **Overview:**  
  Outputs an array of domains to be monitored, each as a separate JSON item, enabling iteration in subsequent nodes.

- **Nodes Involved:**  
  - List Domains to Monitor (Code Node)

- **Node Details:**  
  - **List Domains to Monitor**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Defines an array `domainsToMonitor` containing domain strings (e.g., `"your-company.com"`, `"mail.your-company.com"`, etc.)  
      - Returns each domain as an individual JSON object `{ domain: "domain-name" }` for iteration  
    - Expressions/Variables: Hardcoded domain list; users must edit this array to include their desired domains  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: List of domain objects sent to next node  
    - Potential Failures: Errors in JavaScript code; empty domain list results in no further processing  
    - Version-specific: Uses typeVersion 2; requires n8n supporting code nodes with JavaScript execution  

#### 1.3 Certificate Expiry Retrieval

- **Overview:**  
  For each domain, this node calls a public API to retrieve certificate information including expiry dates.

- **Nodes Involved:**  
  - Check Certificate Expiry (HTTP Request)

- **Node Details:**  
  - **Check Certificate Expiry**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET (default)  
      - URL: `https://api.certspotter.com/v1/certinfo?domain={{ $json.domain }}` — dynamically inserts domain from input data  
      - No authentication configured (public API assumed)  
      - No additional headers or body parameters  
    - Inputs: List of domain JSON objects from "List Domains to Monitor"  
    - Outputs: JSON response containing certificate info, notably `.result.expires` date  
    - Potential Failures:  
      - API downtime or network errors  
      - Rate limiting by API provider  
      - Unexpected API response format or missing expiry data  
      - Incorrect domain format causing empty or error responses  
    - Version-specific: typeVersion 4.2, requires n8n version supporting latest HTTP Request node  

#### 1.4 Expiry Evaluation

- **Overview:**  
  Evaluates whether each certificate expires within the next 30 days and routes the flow accordingly.

- **Nodes Involved:**  
  - Is Certificate Expiring? (IF)

- **Node Details:**  
  - **Is Certificate Expiring?**  
    - Type: IF Node  
    - Configuration:  
      - Condition checks if certificate expiry date (`new Date($json.result.expires)`) is less than the date 30 days from now (`new Date(Date.now() + 30*24*60*60*1000)`)  
      - Strict type validation and case sensitivity enabled  
    - Inputs: Certificate data from HTTP Request node  
    - Outputs:  
      - True branch: Certificates expiring within 30 days → send Slack alert  
      - False branch: Certificates not expiring soon → no further action  
    - Potential Failures:  
      - Missing or invalid expiry date in API response causing expression errors  
      - Date parsing errors if API returns unexpected formats  
    - Version-specific: typeVersion 2.2  

#### 1.5 Slack Alert Dispatch

- **Overview:**  
  Sends an urgent alert message to a specified Slack user or channel notifying about the impending certificate expiry.

- **Nodes Involved:**  
  - YOUR_SECURITY_ALERT_CHANNEL_ID (Slack node)

- **Node Details:**  
  - **YOUR_SECURITY_ALERT_CHANNEL_ID**  
    - Type: Slack Node  
    - Configuration:  
      - Sends a formatted message including domain name and expiry date (formatted as locale date string)  
      - Message text uses expressions to dynamically insert domain and expiry date  
      - Target user/channel specified by an ID (placeholder `123123` to be replaced by actual Slack user or channel ID)  
      - Uses Slack API credentials configured in n8n (credential name: "temp")  
      - Message includes warning emoji and formatting for urgency  
    - Inputs: From IF node’s true branch (certificates expiring soon)  
    - Outputs: None (end node)  
    - Potential Failures:  
      - Slack API authentication errors (invalid or expired credentials)  
      - Incorrect user/channel ID causing message delivery failure  
      - Network issues or Slack service downtime  
    - Version-specific: typeVersion 2.3  

---

### 3. Summary Table

| Node Name                     | Node Type         | Functional Role                           | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                              |
|-------------------------------|-------------------|-----------------------------------------|--------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger  | Initiate workflow on schedule            | None                     | List Domains to Monitor        |                                                                                                         |
| List Domains to Monitor       | Code              | Provide domain list to monitor            | Schedule Trigger          | Check Certificate Expiry       |                                                                                                         |
| Check Certificate Expiry      | HTTP Request      | Query API for certificate expiry          | List Domains to Monitor   | Is Certificate Expiring?       |                                                                                                         |
| Is Certificate Expiring?      | IF                | Check if certificate expires within 30 days | Check Certificate Expiry  | YOUR_SECURITY_ALERT_CHANNEL_ID |                                                                                                         |
| YOUR_SECURITY_ALERT_CHANNEL_ID| Slack             | Send alert message to Slack channel       | Is Certificate Expiring?  | None                          |                                                                                                         |
| Sticky Note                  | Sticky Note       | Visual flow label                         | None                     | None                          | ## Flow                                                                                                 |
| Sticky Note1                 | Sticky Note       | Detailed workflow explanation and setup instructions | None                     | None                          | ### 💡 SSL/TLS Certificate Expiry Monitor 🗓️... (full detailed explanation and setup guide provided)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Node Type: Schedule Trigger  
   - Configure the schedule interval as desired (e.g., weekly on Monday at 08:00 AM)  
   - Leave other default settings  
   - This node will start the workflow automatically.

3. **Add a Code node named "List Domains to Monitor":**  
   - Node Type: Code (JavaScript)  
   - Paste the following JavaScript code in the code editor:
     ```javascript
     const domainsToMonitor = [
         "your-company.com",
         "mail.your-company.com",
         "api.your-company.com"
     ];

     return domainsToMonitor.map(domain => ({ json: { domain } }));
     ```
   - Replace the array contents with your actual domains to monitor.  
   - Connect the Schedule Trigger's output to this node's input.

4. **Add an HTTP Request node named "Check Certificate Expiry":**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.certspotter.com/v1/certinfo?domain={{ $json.domain }}` (use expression editor to insert domain dynamically)  
   - Leave headers and authentication empty unless your API requires it.  
   - Connect the "List Domains to Monitor" output to this node’s input.

5. **Add an IF node named "Is Certificate Expiring?":**  
   - Node Type: IF  
   - Condition Type: Number comparison  
   - Condition: Check if expiry date is less than 30 days from now  
   - Use the expression for left value:  
     `={{ new Date($json.result.expires) }}`  
   - Use the expression for right value:  
     `={{ new Date(Date.now() + 30 * 24 * 60 * 60 * 1000) }}`  
   - Connect the "Check Certificate Expiry" node output to this node.

6. **Add a Slack node named "YOUR_SECURITY_ALERT_CHANNEL_ID":**  
   - Node Type: Slack  
   - Authentication: Set up Slack API credentials in n8n before this step (OAuth2 recommended)  
   - Message text:
     ```
     🚨 *URGENT: SSL Certificate Expiry Alert!* 🚨

     Domain: *{{ $json.domain }}*
     Expires on: *{{ new Date($json.result.expires).toLocaleDateString() }}*

     Please renew this certificate immediately to avoid downtime!
     ```
   - Target: Specify the Slack user or channel ID in the user/channel ID field (replace placeholder `"123123"`)  
   - Connect the true output branch of the IF node "Is Certificate Expiring?" to this Slack node.

7. **Ensure all nodes are connected in the sequence:**  
   - Schedule Trigger → List Domains to Monitor → Check Certificate Expiry → Is Certificate Expiring? → Slack alert (true branch)  
   - False branch of IF node ends the workflow.

8. **Save and activate the workflow.**

9. **Test the workflow manually:**  
   - Run the workflow and verify Slack alerts are received for any certificates expiring within 30 days.  
   - Adjust domain list, schedule, or alert message as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| This workflow does **not** perform automatic SSL/TLS certificate renewal; it only alerts team members to take action.                                                                                                                                                                                                                                                                                                                                                                                                           | Workflow purpose clarification                                                                     |
| The certificate info API used (`https://api.certspotter.com/v1/certinfo`) is an example; for production, select a reliable certificate monitoring API suitable for your domains.                                                                                                                                                                                                                                                                                                                                              | Certificate API usage                                                                              |
| Slack credentials must be preconfigured in n8n with appropriate permissions to send messages to the target channel or user.                                                                                                                                                                                                                                                                                                                                                                                                     | Slack API setup                                                                                   |
| You can adjust the expiry warning period by modifying the 30-day threshold in the IF node’s expression.                                                                                                                                                                                                                                                                                                                                                                                                                         | Customization tip                                                                                 |
| For more information on Slack node setup in n8n, visit: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                                                                                                                                                                                                                                                                                                                                                                                               | Official n8n Slack node documentation                                                            |
| The workflow includes detailed inline comments and a sticky note block explaining its purpose, target users, and setup instructions for ease of onboarding.                                                                                                                                                                                                                                                                                                                                                                     | User guidance and documentation                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.