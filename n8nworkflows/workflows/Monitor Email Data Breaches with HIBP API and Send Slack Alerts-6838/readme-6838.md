Monitor Email Data Breaches with HIBP API and Send Slack Alerts

https://n8nworkflows.xyz/workflows/monitor-email-data-breaches-with-hibp-api-and-send-slack-alerts-6838


# Monitor Email Data Breaches with HIBP API and Send Slack Alerts

### 1. Workflow Overview

This workflow automates the monitoring of corporate email addresses against known data breaches using the HaveIBeenPwned (HIBP) API. It is designed primarily for security teams and IT administrators to proactively detect compromised credentials and alert relevant personnel via Slack for immediate action.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow periodically on a defined schedule (e.g., weekly checks).
- **1.2 Email List Preparation:** Defines the set of corporate email addresses to monitor.
- **1.3 HIBP API Query:** Sends requests to the HIBP API to check if each email address has been compromised.
- **1.4 Breach Detection:** Evaluates the API response to determine if a breach exists for the given email.
- **1.5 Slack Alerting:** Sends urgent notifications to a designated Slack channel when a breach is detected.
- **1.6 Documentation & Guidance:** Sticky notes provide extensive problem context, solution overview, setup instructions, and usage guidelines.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** This block defines when the workflow runs automatically to perform regular breach checks.
- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**
    - Type: Trigger node that initiates the workflow on a time-based schedule.
    - Configuration: Default interval configured with a placeholder (empty object) indicating a schedule should be set by the user (e.g., weekly).
    - Inputs: None (trigger node).
    - Outputs: Connected to "List Emails to Check" node.
    - Edge Cases: If the schedule is not properly configured, the workflow will not run automatically.
    - Notes: User should customize the schedule to suit organizational needs.

#### 2.2 Email List Preparation

- **Overview:** Defines the static list of corporate emails to be checked against breaches.
- **Nodes Involved:**  
  - List Emails to Check

- **Node Details:**

  - **List Emails to Check**
    - Type: Code node executing JavaScript.
    - Configuration: Contains a hardcoded array of emails (`user1@yourcompany.com`, `user2@yourcompany.com`, `admin@yourcompany.com`).
    - Key Expressions: Returns items with JSON property `email` for each address.
    - Inputs: Receives trigger from Schedule Trigger.
    - Outputs: Emits a JSON array of email objects to "Query HIBP API" node.
    - Edge Cases: Requires manual update to the email list for additions/removals.
    - Failure Modes: Syntax errors in code or empty email list would disrupt flow.

#### 2.3 HIBP API Query

- **Overview:** Queries the HaveIBeenPwned API to check each email for breaches.
- **Nodes Involved:**  
  - Query HIBP API

- **Node Details:**

  - **Query HIBP API**
    - Type: HTTP Request node.
    - Configuration:
      - URL: Dynamically constructed to call `https://api.pwnedpasswords.com/range/{{ encodeURIComponent($json.email) }}`.
      - Note: The URL appears to query the Pwned Passwords API range endpoint, which usually accepts partial hashes rather than emails; this is a potential misconfiguration or simplified example. The intended endpoint for emails is `https://haveibeenpwned.com/api/v3/breachedaccount/{account}`.
      - Headers: User must configure `hibp-api-key` header manually with the API key.
    - Inputs: Receives email from "List Emails to Check".
    - Outputs: Passes API response to "Is Breached?" node.
    - Edge Cases:
      - API rate limiting errors if too many requests.
      - Authentication failure if API key is invalid or missing.
      - Incorrect endpoint causing invalid or empty response.
      - Network timeouts.
    - Version Requirements: HTTP Request node version 4.2 or later recommended.

#### 2.4 Breach Detection

- **Overview:** Determines if the API response indicates a breach for the email.
- **Nodes Involved:**  
  - Is Breached?

- **Node Details:**

  - **Is Breached?**
    - Type: If node, used for conditional branching.
    - Configuration:
      - Condition checks if the API response body is not equal to an empty array `[]`.
      - Interpretation: If response body is anything other than an empty array, it is considered a breach.
    - Inputs: Receives HTTP response from "Query HIBP API".
    - Outputs:
      - True branch: Leads to "Send High-Priority Alert".
      - False branch: No further action.
    - Edge Cases:
      - Empty or malformed API response.
      - Unexpected data format from API.
      - Potential false negatives if API changes output format.

#### 2.5 Slack Alerting

- **Overview:** Sends a high-priority alert message to a Slack channel when a breach is detected.
- **Nodes Involved:**  
  - Send High-Priority Alert

- **Node Details:**

  - **Send High-Priority Alert**
    - Type: Slack node for sending messages.
    - Configuration:
      - Text: Dynamic message highlighting the breached email, marked urgent with emoji and formatting.
      - User: Set to send message to a specific Slack user or channel by ID (requires replacing `[YOUR_SECURITY_ALERT_CHANNEL_ID]` with actual ID).
      - Credential: Requires Slack API credentials configured in n8n.
    - Inputs: Triggered only if "Is Breached?" node returns true.
    - Outputs: None (end of branch).
    - Edge Cases:
      - Slack API authentication failure.
      - Invalid or missing channel ID.
      - Slack API rate limits.
      - Message delivery failure.
    - Notes: Critical for alerting security team to enable fast response.

#### 2.6 Documentation & Guidance

- **Overview:** Provides detailed problem context, solution description, target audience, scope, and setup instructions directly in the workflow.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**
    - Type: Visual annotation node.
    - Content: Title "Flow" indicating the start or overview of the workflow.
  - **Sticky Note1**
    - Type: Visual annotation node with extensive markdown content.
    - Content:
      - Explains the problem of credential stuffing attacks.
      - Describes the solution offered by this workflow.
      - Defines the target users (security teams, IT admins, SMEs).
      - Clarifies scope and limitations.
      - Provides detailed step-by-step setup instructions including credential preparation, node configuration, testing, and activation.
    - Purpose: Serves as in-editor documentation and onboarding guide.
    - Position: Placed below main workflow nodes for easy reference.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                         | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                          |
|-----------------------|---------------------|---------------------------------------|-------------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger    | Initiates workflow on schedule        | —                       | List Emails to Check       |                                                                                                    |
| List Emails to Check   | Code                | Defines email list to check            | Schedule Trigger         | Query HIBP API             |                                                                                                    |
| Query HIBP API         | HTTP Request        | Queries HIBP API for breach data       | List Emails to Check     | Is Breached?               |                                                                                                    |
| Is Breached?           | If                  | Checks if breach data indicates compromise | Query HIBP API           | Send High-Priority Alert   |                                                                                                    |
| Send High-Priority Alert | Slack              | Sends urgent Slack alert on breach     | Is Breached? (true branch) | —                         |                                                                                                    |
| Sticky Note            | Sticky Note         | Visual title "Flow"                    | —                       | —                         |                                                                                                    |
| Sticky Note1           | Sticky Note         | Detailed problem, solution, setup guide | —                       | —                         | Explains credential stuffing risks, workflow scope, setup steps, usage tips, and target users.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Set the trigger interval to your desired schedule (e.g., weekly on Monday at 8:00 AM).
   - This node will start the workflow automatically on the set schedule.

3. **Add a Code node named "List Emails to Check":**
   - Paste the following JavaScript code to define the emails to monitor:
     ```javascript
     const emailsToCheck = [
         "user1@yourcompany.com",
         "user2@yourcompany.com",
         "admin@yourcompany.com"
     ];

     return emailsToCheck.map(email => ({
         json: { email: email }
     }));
     ```
   - Connect the Schedule Trigger's output to this node's input.

4. **Add an HTTP Request node named "Query HIBP API":**
   - Set the HTTP Method to GET.
   - Set the URL to:
     ```
     https://haveibeenpwned.com/api/v3/breachedaccount/{{ encodeURIComponent($json.email) }}
     ```
     (Note: Adjusted to correct HIBP API endpoint for email breach checking.)
   - Under the Headers section, add a header:
     - Key: `hibp-api-key`
     - Value: Your HaveIBeenPwned API key (must be stored securely).
   - Set Response Format to JSON.
   - Connect "List Emails to Check" node output to this node input.

5. **Add an If node named "Is Breached?":**
   - Configure condition:
     - Type: Expression
     - Expression: Check if the response body is not empty.
       Example condition:
       - Check if `{{$json.length}}` is greater than 0 (assuming response is an array of breaches).
   - Connect "Query HIBP API" node output to this node input.

6. **Add a Slack node named "Send High-Priority Alert":**
   - Select your Slack credentials configured in n8n.
   - Set the message text to:
     ```
     🚨 *URGENT: Data Breach Alert!* 🚨
     Email address *{{ $json.email }}* was found in a data breach.
     Immediate action is required!
     ```
   - Specify the Slack channel ID or user ID to send alerts to (replace placeholder with actual ID).
   - Connect the True branch of "Is Breached?" node to this Slack node.

7. **Add Sticky Notes for documentation (optional but recommended):**
   - Create one sticky note titled "Flow" above the main nodes.
   - Create another sticky note below the main nodes with detailed problem context, solution explanation, target audience, scope, and setup instructions.

8. **Verify credentials:**
   - Create or update Slack credentials with OAuth2 token.
   - Store the HaveIBeenPwned API key securely in n8n credentials.

9. **Test the workflow:**
   - Manually trigger the workflow.
   - Use known breached email addresses to confirm alerts are sent.
   - Monitor Slack for incoming alerts.

10. **Activate the workflow:**
    - Once verified, enable the workflow to run on the schedule.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The workflow does not automatically reset passwords or remediate breaches; it is solely a detection and alerting mechanism.                                                                                                                                                                                                                                | Important operational limitation.                                                                      |
| The HaveIBeenPwned API requires registration and an API key; free tier limits and rate limiting apply.                                                                                                                                                                                                                                                      | https://haveibeenpwned.com/API/v3                                                                       |
| Slack channel ID can be obtained from the Slack UI by right-clicking the channel name and selecting "Copy Link"; extract the ID from the URL.                                                                                                                                                                                                              | Slack documentation                                                                                     |
| Credential stuffing attacks are a major cybersecurity risk where attackers reuse leaked credentials to gain unauthorized access. This workflow helps mitigate that risk by rapid breach detection.                                                                                                                                                            | Security context explanation.                                                                           |
| Video walkthrough of n8n workflows and credential setup available at [n8n YouTube Channel](https://www.youtube.com/c/n8n_io).                                                                                                                                                                                                                                | User onboarding and training resource.                                                                 |

---

This document provides a comprehensive reference and guide for understanding, reproducing, and maintaining the "Monitor Email Data Breaches with HIBP API and Send Slack Alerts" workflow in n8n. It is designed for both human operators and automation agents to ensure reliability and extensibility.