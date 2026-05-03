Scheduled Website Uptime Monitor with Slack Alert

https://n8nworkflows.xyz/workflows/scheduled-website-uptime-monitor-with-slack-alert-6467


# Scheduled Website Uptime Monitor with Slack Alert

### 1. Workflow Overview

This workflow is designed to monitor a website’s uptime by periodically checking its HTTP status and sending a Slack alert if the website is down. It targets use cases where automated, scheduled monitoring of web services is required to ensure availability and prompt incident notification.

The workflow logic is organized into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow execution on a fixed hourly schedule.
- **1.2 Website Status Check:** Performs an HTTP request to the target website to evaluate its availability.
- **1.3 Conditional Branching:** Determines if the website is down based on the HTTP response.
- **1.4 Alert Notification:** Sends a Slack message alerting stakeholders of downtime.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically every hour to perform the uptime check regularly without manual intervention.

- **Nodes Involved:**  
  - Schedule Monitor (Every Hour)

- **Node Details:**  
  - **Schedule Monitor (Every Hour)**
    - Type: Cron Trigger  
    - Configuration: Runs on an hourly schedule (default cron without additional parameters, implying every hour)  
    - Input Connections: None (trigger node)  
    - Output Connections: Connected to “Check Website HTTP Status” node  
    - Key Expressions/Variables: None  
    - Version Requirements: n8n Cron Trigger node, version 1  
    - Potential Failures: Scheduler misconfiguration (unlikely with default), instance downtime preventing trigger  
    - Notes: This node forms the entry point of the workflow.

#### 1.2 Website Status Check

- **Overview:**  
  This block makes an HTTP request to the monitored website to capture its current status, which is the core data used for uptime monitoring.

- **Nodes Involved:**  
  - Check Website HTTP Status

- **Node Details:**  
  - **Check Website HTTP Status**  
    - Type: HTTP Request  
    - Configuration:  
      - Default HTTP method (likely GET)  
      - URL not explicitly stated in JSON but must be configured to target the monitored website  
      - No authentication or custom headers indicated  
      - Response mode likely “On Received” for immediate status code check  
    - Input Connections: Receives trigger from “Schedule Monitor (Every Hour)”  
    - Output Connections: Connects to “If Website is Down” node  
    - Key Expressions/Variables: None specified, but HTTP response status code is critical for downstream logic  
    - Version Requirements: HTTP Request node, version 3  
    - Potential Failures: Network errors, DNS failures, timeouts, invalid URL, unexpected HTTP responses

#### 1.3 Conditional Branching

- **Overview:**  
  This block evaluates the HTTP response status code to decide whether the website is down. It branches the workflow accordingly.

- **Nodes Involved:**  
  - If Website is Down

- **Node Details:**  
  - **If Website is Down**  
    - Type: If Node (Conditional Logic)  
    - Configuration:  
      - Condition likely checks if HTTP response status code is outside the 200-299 range (not detailed in JSON but standard practice)  
      - Typically, condition such as: `HTTP Status Code` != 200 or >= 400  
    - Input Connections: Receives data from “Check Website HTTP Status”  
    - Output Connections:  
      - True branch: Connects to “Send a message” node (indicating downtime)  
      - False branch: No connection (workflow ends silently if website is up)  
    - Key Expressions/Variables: Expression evaluating HTTP response status code  
    - Version Requirements: If node, version 2  
    - Potential Failures: Expression evaluation errors if HTTP response missing or malformed

#### 1.4 Alert Notification

- **Overview:**  
  This block sends a Slack message to a predefined channel or user to alert them about website downtime.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  
  - **Send a message**  
    - Type: Slack node  
    - Configuration:  
      - Uses a webhook ID for Slack integration (preconfigured webhook)  
      - Message content not specified but expected to include downtime alert details  
    - Input Connections: Triggered only if “If Website is Down” condition is true  
    - Output Connections: None (end node)  
    - Key Expressions/Variables: Slack message text could be dynamically constructed based on HTTP status or fixed alert text  
    - Version Requirements: Slack node version 2.3  
    - Potential Failures: Slack webhook errors, network issues, invalid webhook ID, permission errors

---

### 3. Summary Table

| Node Name                | Node Type         | Functional Role         | Input Node(s)                 | Output Node(s)          | Sticky Note                        |
|--------------------------|-------------------|------------------------|------------------------------|-------------------------|----------------------------------|
| Schedule Monitor (Every Hour) | Cron Trigger      | Scheduled trigger       | None                         | Check Website HTTP Status|                                  |
| Check Website HTTP Status | HTTP Request      | Website status check    | Schedule Monitor (Every Hour)| If Website is Down       |                                  |
| If Website is Down       | If                | Conditional check       | Check Website HTTP Status    | Send a message          |                                  |
| Send a message           | Slack             | Slack alert notification| If Website is Down           | None                    |                                  |
| Sticky Note              | Sticky Note       | (No operational role)   | None                         | None                    |                                  |
| Sticky Note1             | Sticky Note       | (No operational role)   | None                         | None                    |                                  |
| Sticky Note2             | Sticky Note       | (No operational role)   | None                         | None                    |                                  |

*Note: Sticky Notes are present but empty; no comments or instructions detected.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Cron Trigger node**  
   - Name: `Schedule Monitor (Every Hour)`  
   - Parameters: Set to trigger every hour (default cron settings suffice)  
   - This node will initiate the workflow hourly.

3. **Add an HTTP Request node**  
   - Name: `Check Website HTTP Status`  
   - Connect the output of `Schedule Monitor (Every Hour)` to this node’s input.  
   - Parameters:  
     - HTTP Method: GET  
     - URL: Set to the website’s URL you want to monitor (e.g., https://example.com)  
     - Response Format: Keep default to capture status code  
     - Authentication: None (unless the site requires it)  
     - Timeout: Consider setting a timeout (e.g., 10 seconds) to avoid hanging requests.

4. **Add an If node**  
   - Name: `If Website is Down`  
   - Connect the output of `Check Website HTTP Status` to this node.  
   - Configure the condition:  
     - Use an expression to check the HTTP response code, for example:  
       - `{{$node["Check Website HTTP Status"].json["statusCode"]}}`  
       - Condition: `Is Not Between` 200 and 299  
     - This condition means the website is considered down if any non-success HTTP status is returned.

5. **Add a Slack node**  
   - Name: `Send a message`  
   - Connect the **true** output (first output) of `If Website is Down` to this node.  
   - Configure Slack credentials:  
     - Use existing Slack webhook credentials or create new ones with proper permissions.  
   - Parameters:  
     - Channel: Specify the Slack channel or user to receive alerts  
     - Message Text: Compose a message indicating the website is down, e.g.,  
       `"Alert: The website is down. Status code: {{$node["Check Website HTTP Status"].json["statusCode"]}}"`  
   - Webhook ID: Use or create a valid Slack webhook.

6. **No connections from the false branch of the If node are necessary**, as no action is required if the website is up.

7. **Optionally, add sticky notes for documentation** within the workflow editor describing each block’s purpose.

8. **Activate the workflow** to start monitoring.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                      |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| This workflow assumes the monitored website responds with standard HTTP status codes.         | General HTTP monitoring best practices              |
| Slack webhook must be preconfigured with appropriate permissions to post messages in the channel.| Slack API documentation: https://api.slack.com/messaging/webhooks |
| Consider adding retry logic or error handling for transient network failures to improve robustness.| n8n documentation on error workflows                 |
| Timeout settings on HTTP requests can prevent workflow hang-ups due to slow or unresponsive sites.| n8n HTTP Request node settings                        |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.