Service Health Monitoring with Double-Verification & Slack Alerts

https://n8nworkflows.xyz/workflows/service-health-monitoring-with-double-verification---slack-alerts-8130


# Service Health Monitoring with Double-Verification & Slack Alerts

### 1. Workflow Overview

This workflow implements a **Service Health Monitoring** system with a robust double-verification mechanism to reduce false positive alerts. It performs scheduled HTTP status checks on a target service URL, waits and retries if a failure is detected, and only sends alerts to a Slack channel if the service fails twice consecutively. This design ensures alerts are accurate and reduces unnecessary alert fatigue for operations teams.

Logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow at a specified daily time.
- **1.2 First Health Check:** Performs the initial HTTP request to check service status.
- **1.3 Status Evaluation:** Analyzes the HTTP response to detect potential failure.
- **1.4 Retry Delay:** Waits for a defined period before retrying.
- **1.5 Second Health Check:** Repeats the HTTP request to verify the failure.
- **1.6 Failure Confirmation:** Confirms if the failure persists.
- **1.7 Slack Alert:** Sends an alert to a Slack channel if the failure is confirmed.
- **1.8 Documentation Sticky Notes:** Provides visual summaries and setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow once daily at a specified hour to start the health check process.

- **Nodes Involved:**  
  - Check Interval

- **Node Details:**  
  - **Check Interval**  
    - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
    - Configuration: Set to trigger once daily at 09:00 (9 AM).  
    - Input: None (trigger node)  
    - Output: Initiates the workflow flow to "First Check" node.  
    - Edge Cases: If the n8n instance is down or paused at trigger time, the check will be delayed until next trigger.  
    - Version: 1.2

#### 1.2 First Health Check

- **Overview:**  
  Performs the initial HTTP request to check the service’s health status.

- **Nodes Involved:**  
  - First Check

- **Node Details:**  
  - **First Check**  
    - Type: HTTP Request  
    - Configuration: Requests URL `https://httpbin.org/status/400` (placeholder URL for testing).  
    - Options:  
      - Response handling set to never error on HTTP errors to prevent workflow failure on bad status codes.  
      - Full HTTP response is returned, including status code and headers.  
    - Inputs: Triggered by "Check Interval" node.  
    - Outputs: Passes response JSON to "Check Status" node.  
    - Edge Cases: Network timeouts, DNS failures, or non-responsive endpoints could cause timeouts or errors; these are not explicitly handled here.  
    - Version: 4.2

#### 1.3 Status Evaluation

- **Overview:**  
  Evaluates whether the HTTP status code from the first check indicates a failure (status code ≤ 400).

- **Nodes Involved:**  
  - Check Status

- **Node Details:**  
  - **Check Status**  
    - Type: If node  
    - Configuration:  
      - Condition: Check if `$json["statusCode"] ≤ 400`. If true, it indicates a failure or problematic status.  
    - Inputs: Receives HTTP response from "First Check".  
    - Outputs:  
      - If true (failure): Proceeds to "Recheck Delay".  
      - If false (healthy): Stops workflow here (no further action).  
    - Edge Cases:  
      - If the response is malformed or missing `statusCode`, the expression may fail.  
    - Version: 2.2

#### 1.4 Retry Delay

- **Overview:**  
  Waits for a fixed delay period before performing the second health check to confirm failure.

- **Nodes Involved:**  
  - Recheck Delay

- **Node Details:**  
  - **Recheck Delay**  
    - Type: Wait node  
    - Configuration: Waits for 30 seconds before continuing.  
    - Inputs: Triggered only if first check indicates failure.  
    - Outputs: Connects to "Second Check" node.  
    - Edge Cases: Delay may be affected if workflow is paused or system load is high.  
    - Version: 1.1

#### 1.5 Second Health Check

- **Overview:**  
  Performs the second HTTP request to verify the service status after the delay.

- **Nodes Involved:**  
  - Second Check

- **Node Details:**  
  - **Second Check**  
    - Type: HTTP Request  
    - Configuration: Same URL and options as "First Check" (`https://httpbin.org/status/400`), with never-error and full response options enabled.  
    - Inputs: Triggered by "Recheck Delay".  
    - Outputs: Passes response to "Confirm Failure" node.  
    - Edge Cases: Same as first HTTP request node.  
    - Version: 4.2

#### 1.6 Failure Confirmation

- **Overview:**  
  Checks if the second health check also indicates failure (status code ≤ 400) to confirm the issue.

- **Nodes Involved:**  
  - Confirm Failure

- **Node Details:**  
  - **Confirm Failure**  
    - Type: If node  
    - Configuration: Same condition as "Check Status", i.e., `$json["statusCode"] ≤ 400`.  
    - Inputs: Receives second HTTP request response.  
    - Outputs:  
      - If true: Proceeds to "Send Alert to Slack".  
      - If false: Ends the workflow silently (no alert).  
    - Edge Cases: Expression failures if response malformed.  
    - Version: 2.2

#### 1.7 Slack Alert

- **Overview:**  
  Sends a formatted alert message to a configured Slack channel indicating the confirmed service failure.

- **Nodes Involved:**  
  - Send Alert to Slack

- **Node Details:**  
  - **Send Alert to Slack**  
    - Type: Slack node  
    - Configuration:  
      - Sends a message to a specific Slack channel (channel ID set via expression linked to credentials).  
      - Message includes:  
        - Emoji alert icon 🚨  
        - Status code and status message from the HTTP response JSON.  
        - Timestamp extracted from HTTP response headers (`date` header).  
      - Does not include a link back to the workflow in the message.  
    - Credentials: Uses Slack API credentials configured under "Slack Test".  
    - Inputs: Triggered only if failure confirmed twice.  
    - Outputs: Ends workflow after alert sent.  
    - Edge Cases: Slack API authentication failure, rate limiting, channel access issues.  
    - Version: 2.3

#### 1.8 Documentation Sticky Notes

- **Overview:**  
  Provides workflow operators with in-editor documentation and setup instructions for clarity and ease of use.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note**  
    - Type: Sticky Note  
    - Content: Explains how the workflow works in bullet points, including retry logic and alert suppression to prevent false positives.  
  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Setup instructions for customizing URL, Slack token, scheduling, and activation.  
  - Inputs/Outputs: None (decorative only).  
  - Edge Cases: None.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                          | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                      |
|---------------------|---------------------------|----------------------------------------|-----------------------|------------------------|------------------------------------------------------------------------------------------------|
| Check Interval      | Schedule Trigger          | Initiates daily workflow trigger       | None                  | First Check            | Setup steps: Add your service URL(s), configure Slack token, set interval, save & activate.    |
| First Check         | HTTP Request              | Performs initial service health check  | Check Interval        | Check Status           |                                                                                                |
| Check Status        | If                        | Evaluates first check status            | First Check           | Recheck Delay          |                                                                                                |
| Recheck Delay       | Wait                      | Waits 30 seconds before retry           | Check Status          | Second Check           |                                                                                                |
| Second Check        | HTTP Request              | Performs second service health check    | Recheck Delay         | Confirm Failure        |                                                                                                |
| Confirm Failure     | If                        | Confirms failure after second check     | Second Check          | Send Alert to Slack    |                                                                                                |
| Send Alert to Slack | Slack                     | Sends failure alert message to Slack    | Confirm Failure       | None                   |                                                                                                |
| Sticky Note         | Sticky Note               | Explains workflow logic                  | None                  | None                   | How it works: Runs scheduled checks, retries, sends alerts only on double failure, reduces noise |
| Sticky Note1        | Sticky Note               | Provides setup instructions              | None                  | None                   | Setup steps: Add URLs, configure Slack token, set interval, activate workflow                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Check Interval`  
   - Set the trigger to run once daily at 09:00 (hour 9).  
   - Leave other defaults.

3. **Add an HTTP Request node:**  
   - Name: `First Check`  
   - Set the URL to your target service’s health check endpoint (currently `https://httpbin.org/status/400` for testing).  
   - Under Options:  
     - Enable "Never Error" to avoid workflow failure on HTTP errors.  
     - Enable "Full Response" to get status code and headers.  
   - Connect `Check Interval` → `First Check`.

4. **Add an If node:**  
   - Name: `Check Status`  
   - Configure condition:  
     - Expression: `{{$json["statusCode"] <= 400}}`  
   - Connect `First Check` → `Check Status`.  
   - Set the “true” output branch to continue, “false” branch left unconnected (ends workflow).

5. **Add a Wait node:**  
   - Name: `Recheck Delay`  
   - Set wait time to 30 seconds.  
   - Connect `Check Status` (true output) → `Recheck Delay`.

6. **Add a second HTTP Request node:**  
   - Name: `Second Check`  
   - Configure identically to `First Check` (same URL and options).  
   - Connect `Recheck Delay` → `Second Check`.

7. **Add another If node:**  
   - Name: `Confirm Failure`  
   - Same condition as `Check Status`: check if status code ≤ 400.  
   - Connect `Second Check` → `Confirm Failure`.  
   - “True” branch continues, “false” branch ends workflow.

8. **Add a Slack node:**  
   - Name: `Send Alert to Slack`  
   - Set channel via channel ID or name where alerts should be posted.  
   - Configure message text with variables:  
     ```
     🚨 URGENT: Service check failed twice in a row!   
     Status: {{$json["statusCode"]}}
     Message: {{$json["statusMessage"]}}  
     Time: {{$json["headers"]["date"]}}
     ```  
   - Disable “Include Link To Workflow” option for cleaner alerts.  
   - Provide Slack API credentials with bot token authorized for the target channel.  
   - Connect `Confirm Failure` (true output) → `Send Alert to Slack`.

9. **Add Sticky Notes (optional):**  
   - Add notes to explain workflow purpose and setup instructions for ease of maintenance.

10. **Save and activate your workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                              |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| The workflow runs scheduled HTTP health checks with retry and confirmation to reduce noise. | Overview sticky note inside workflow.                                        |
| Setup requires inserting your actual service URL and Slack Bot Token in the respective nodes.| Setup sticky note inside workflow.                                          |
| For Slack credentials, create a Slack App with bot token and appropriate channel permissions.| Slack API documentation: https://api.slack.com/authentication/basics         |
| Adjust the check interval and retry delay as needed based on service criticality and SLA.    | Customization advice.                                                        |
| Use "Never Error" option in HTTP Request nodes to prevent workflow interruption on HTTP errors.| Key to ensure reliable monitoring despite transient failures.               |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created using n8n, a workflow automation platform. It strictly complies with applicable content policies and does not include any illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.