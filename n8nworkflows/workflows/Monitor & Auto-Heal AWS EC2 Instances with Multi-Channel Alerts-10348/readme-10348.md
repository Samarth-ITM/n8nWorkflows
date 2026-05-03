Monitor & Auto-Heal AWS EC2 Instances with Multi-Channel Alerts

https://n8nworkflows.xyz/workflows/monitor---auto-heal-aws-ec2-instances-with-multi-channel-alerts-10348


# Monitor & Auto-Heal AWS EC2 Instances with Multi-Channel Alerts

### 1. Workflow Overview

This workflow automates the monitoring and self-healing of AWS EC2 instances running in a production environment. It periodically checks the health status of EC2 instances, identifies those that are unhealthy for more than 10 minutes, attempts to restart them, and then alerts the DevOps team through multiple channels (email, WhatsApp, and Google Sheets logging). It is designed to ensure high availability and quick incident response with minimal manual intervention.

The workflow logically divides into the following blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Runs on a cron schedule every 5 minutes, triggering the start of the process and retrieving the list of EC2 instances.
- **1.2 Instance Iteration & Status Check:** Iterates over each instance to fetch its current health status.
- **1.3 Health Evaluation & Self-Healing Decision:** Evaluates if an instance is unhealthy and if remediation is needed based on uptime and state.
- **1.4 Automated Restart & Alerting:** Attempts to restart unhealthy instances and sends multi-channel alerts (WhatsApp, email, logging).
- **1.5 Workflow Completion:** Marks the workflow’s end after alert dispatch.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Retrieval

- **Overview:**  
  This block initiates the workflow every 5 minutes via a cron schedule, retrieving the list of all EC2 instances from a custom API endpoint.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get EC2 Instances

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow on a cron schedule  
    - Configuration: Runs every 5 minutes using a cron expression `0 */5 * * *`  
    - Inputs: None  
    - Outputs: Triggers Get EC2 Instances node  
    - Potential Failures: Cron expression misconfiguration; scheduler downtime  

  - **Get EC2 Instances**  
    - Type: HTTP Request  
    - Role: Fetches the list of production EC2 instances from a custom API endpoint (`https://devops.oneclicksales.xyz/ec2`)  
    - Configuration: Simple GET request without authentication or parameters  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Passes instance list to Loop Over Instances node  
    - Potential Failures: API endpoint unavailability, network errors, malformed response  

#### 1.2 Instance Iteration & Status Check

- **Overview:**  
  Splits the list of instances to process each one individually and checks the current health status of each instance.

- **Nodes Involved:**  
  - Loop Over Instances  
  - Check Instance Status

- **Node Details:**

  - **Loop Over Instances**  
    - Type: SplitInBatches  
    - Role: Processes EC2 instances one at a time (batch size = 1 by default)  
    - Configuration: Default options, no batch size explicitly set but implied single-item processing  
    - Inputs: List of instances from Get EC2 Instances  
    - Outputs: Feeds single instance data to Check Instance Status or continues looping if conditions fail  
    - Potential Failures: Large instance list causing delays; improper batch handling  

  - **Check Instance Status**  
    - Type: HTTP Request  
    - Role: Retrieves detailed health status of the current instance from a custom API endpoint (`https://devops.oneclicksales.xyz/ec2status`)  
    - Configuration: Simple GET request, no parameters indicated (likely uses instance info from input)  
    - Inputs: Single instance from Loop Over Instances  
    - Outputs: Passes instance status data to Check Health Status node  
    - Potential Failures: API failures, latency, incorrect instance ID propagation  

#### 1.3 Health Evaluation & Self-Healing Decision

- **Overview:**  
  Evaluates if the instance is currently unhealthy and if it has been so for more than 10 minutes, deciding whether to proceed with remediation.

- **Nodes Involved:**  
  - Check Health Status  
  - Analyze Health Data

- **Node Details:**

  - **Check Health Status**  
    - Type: If Node  
    - Role: Checks if instance state is not "running"  
    - Configuration: Condition triggers if `State.Name` ≠ "running" (string inequality, case sensitive)  
    - Inputs: Instance status from Check Instance Status  
    - Outputs:  
      - True branch: Passes to Analyze Health Data (unhealthy instances)  
      - False branch: Loops back to Loop Over Instances (healthy instances, continue processing)  
    - Potential Failures: Missing or malformed `State.Name` field; strict case sensitivity may cause false negatives  

  - **Analyze Health Data**  
    - Type: Code Node (JavaScript)  
    - Role: Calculates uptime and determines if action is required (unhealthy > 10 minutes)  
    - Configuration:  
      - Extracts `InstanceId`, `State.Name`, `InstanceType`, and `LaunchTime` from input JSON  
      - Calculates uptime in minutes (`now - LaunchTime`)  
      - Determines `isUnhealthy` and `actionRequired` flags  
      - Returns structured object with these values and current timestamp  
    - Inputs: Unhealthy instance data from Check Health Status  
    - Outputs: Passes data to Restart Instance node  
    - Potential Failures: Date parsing errors, missing fields, time zone considerations  

#### 1.4 Automated Restart & Alerting

- **Overview:**  
  Attempts to restart instances flagged for remediation and sends multi-channel alerts to notify the team.

- **Nodes Involved:**  
  - Restart Instance  
  - Send WhatsApp Alert  
  - Send Email Alert  
  - Log to AlertsLog Sheet

- **Node Details:**

  - **Restart Instance**  
    - Type: HTTP Request  
    - Role: Calls a custom API endpoint (`https://devops.oneclicksales.xyz/ec2restart`) to restart the instance  
    - Configuration: POST or GET not explicitly specified (default GET assumed), no parameters shown (likely uses input data)  
    - Inputs: Output from Analyze Health Data  
    - Outputs: Fan-out to alert nodes  
    - Potential Failures: API downtime, restart failures, permission errors  

  - **Send WhatsApp Alert**  
    - Type: HTTP Request  
    - Role: Sends an instant alert via WhatsApp using Twilio API  
    - Configuration:  
      - POST to Twilio Messages endpoint with authentication via Twilio credentials  
      - Message includes instance ID, state, and uptime minutes in formatted text with alert emoji  
      - Uses predefined Twilio credential with Account SID and Auth Token  
    - Inputs: Output from Restart Instance  
    - Outputs: Ends at End Workflow  
    - Potential Failures: Twilio API quota exceeded, authentication errors, invalid phone numbers  

  - **Send Email Alert**  
    - Type: Email Send  
    - Role: Sends a detailed email alert to DevOps team  
    - Configuration:  
      - Subject: "EC2 Health Alert - Instance Restarted"  
      - To: devops-team@yourcompany.com  
      - From: alerts@yourcompany.com  
      - Uses SMTP credentials  
    - Inputs: Output from Restart Instance  
    - Outputs: Leads to End Workflow  
    - Potential Failures: SMTP server issues, email delivery failures  

  - **Log to AlertsLog Sheet**  
    - Type: Google Sheets  
    - Role: Logs alert information into a Google Sheets document for historical tracking  
    - Configuration:  
      - Appends or updates rows in Sheet1 of a specified spreadsheet (ID provided)  
      - Authenticated via Google Service Account credentials  
      - Uses auto-mapping to populate columns with input data fields  
    - Inputs: Output from Restart Instance  
    - Outputs: Leads to End Workflow  
    - Potential Failures: Google API quota limits, authentication errors, spreadsheet permission issues  

#### 1.5 Workflow Completion

- **Overview:**  
  Marks the graceful end of workflow execution after alerts are sent and logs updated.

- **Nodes Involved:**  
  - End Workflow

- **Node Details:**

  - **End Workflow**  
    - Type: No Operation (NoOp)  
    - Role: Terminates the workflow execution cleanly  
    - Configuration: None  
    - Inputs: From Send WhatsApp Alert, Send Email Alert, Log to AlertsLog Sheet  
    - Outputs: None  
    - Potential Failures: None  

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                         | Input Node(s)          | Output Node(s)                                | Sticky Note                                                                                      |
|----------------------|---------------------|---------------------------------------|-----------------------|-----------------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger    | Initiates workflow every 5 minutes    | None                  | Get EC2 Instances                             | ## 📋 Workflow Configuration<br>Runs every 5 minutes via cron schedule<br>Checks all Production EC2 instances<br>Monitors health status<br>Auto-remediation enabled |
| Get EC2 Instances     | HTTP Request        | Retrieves list of EC2 instances       | Schedule Trigger      | Loop Over Instances                           |                                                                                                |
| Loop Over Instances   | SplitInBatches      | Processes instances one-by-one         | Get EC2 Instances     | Check Instance Status (false branch)<br>Check Health Status (true branch) |                                                                                                |
| Check Instance Status | HTTP Request        | Retrieves current health status       | Loop Over Instances   | Check Health Status                           |                                                                                                |
| Check Health Status   | If Node             | Checks if instance is unhealthy       | Check Instance Status | Analyze Health Data (true branch)<br>Loop Over Instances (false branch) | ## ⚙️ Self-Healing Actions<br>Detects unhealthy instances<br>Attempts restart<br>Monitors recovery<br>Alerts team<br>Failsafe: acts only after 10+ minutes unhealthy |
| Analyze Health Data   | Code Node           | Determines if remediation needed      | Check Health Status   | Restart Instance                             |                                                                                                |
| Restart Instance      | HTTP Request        | Attempts to restart unhealthy instance | Analyze Health Data   | Send WhatsApp Alert<br>Send Email Alert<br>Log to AlertsLog Sheet |                                                                                                |
| Send WhatsApp Alert   | HTTP Request        | Sends instant alert via WhatsApp      | Restart Instance      | End Workflow                                 | ## 📧 Alert Notifications<br>Sends detailed report to team via Email, WhatsApp, and Google Sheets<br>Includes Instance ID, status, action taken, timestamp |
| Send Email Alert      | Email Send          | Sends email alert to DevOps team      | Restart Instance      | End Workflow                                 | ## 📧 Alert Notifications<br>Sends detailed report to team via Email, WhatsApp, and Google Sheets<br>Includes Instance ID, status, action taken, timestamp |
| Log to AlertsLog Sheet| Google Sheets       | Logs alert events to Google Sheets    | Restart Instance      | End Workflow                                 | ## 📧 Alert Notifications<br>Sends detailed report to team via Email, WhatsApp, and Google Sheets<br>Includes Instance ID, status, action taken, timestamp |
| End Workflow         | NoOp                | Ends the workflow gracefully           | Send WhatsApp Alert<br>Send Email Alert<br>Log to AlertsLog Sheet | None                                  |                                                                                                |
| Sticky Note 1         | Sticky Note         | Describes workflow schedule & scope   | None                  | None                                         | ## 📋 Workflow Configuration<br>Runs every 5 minutes via cron schedule<br>Checks all Production EC2 instances<br>Monitors health status<br>Auto-remediation enabled |
| Sticky Note 2         | Sticky Note         | Describes alert notification methods  | None                  | None                                         | ## 📧 Alert Notifications<br>Sends detailed report to team via Email, WhatsApp, and Google Sheets<br>Includes Instance ID, status, action taken, timestamp |
| Sticky Note 3         | Sticky Note         | Describes self-healing logic           | None                  | None                                         | ## ⚙️ Self-Healing Actions<br>Detects unhealthy instances<br>Attempts restart<br>Monitors recovery<br>Alerts team<br>Failsafe: acts only after 10+ minutes unhealthy |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set Cron Expression: `0 */5 * * *` to run every 5 minutes  
   - No credentials needed  

2. **Create Get EC2 Instances Node**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://devops.oneclicksales.xyz/ec2`  
   - No authentication  
   - Connect Schedule Trigger output to this node input  

3. **Create Loop Over Instances Node**  
   - Type: SplitInBatches  
   - Default batch size (process one instance at a time)  
   - Connect Get EC2 Instances output to this node input  

4. **Create Check Instance Status Node**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://devops.oneclicksales.xyz/ec2status`  
   - (Assumes instance info is passed in request context or headers if needed)  
   - Connect "false" output of Loop Over Instances (empty batch) to this node input  

5. **Create Check Health Status Node**  
   - Type: If Node  
   - Condition: Check if `State.Name` is NOT equal to `"running"` (case sensitive)  
   - Left value expression: `{{$json.State.Name}}`  
   - Operator: "notEquals"  
   - Right value: `"running"`  
   - Connect Check Instance Status output to this node input  

6. **Connect "True" Branch of Check Health Status to Analyze Health Data Node**  
   - Create Analyze Health Data Node (Code Node)  
   - Paste the following JavaScript code:
     ```javascript
     const instanceId = $input.item.json.InstanceId;
     const instanceState = $input.item.json.State.Name;
     const instanceType = $input.item.json.InstanceType;
     const launchTime = $input.item.json.LaunchTime;

     const now = new Date();
     const launch = new Date(launchTime);
     const uptimeMinutes = (now - launch) / (1000 * 60);

     return {
       instanceId,
       instanceState,
       instanceType,
       uptimeMinutes: Math.round(uptimeMinutes),
       isUnhealthy: instanceState !== 'running',
       actionRequired: instanceState !== 'running' && uptimeMinutes > 10,
       timestamp: now.toISOString()
     };
     ```
   - Connect output of Check Health Status "true" branch to this node  

7. **Create Restart Instance Node**  
   - Type: HTTP Request  
   - Method: GET or POST (confirm with API)  
   - URL: `https://devops.oneclicksales.xyz/ec2restart`  
   - Connect Analyze Health Data output to this node input  

8. **Create Send WhatsApp Alert Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.twilio.com/2010-04-01/Accounts/{{ $credentials.accountSid }}/Messages.json`  
   - Authentication: Twilio API (configure Twilio credentials with Account SID and Auth Token)  
   - Body Parameters (form-urlencoded):  
     - To: `whatsapp:+1234567890` (replace with real number)  
     - From: `whatsapp:+14155238886` (Twilio WhatsApp sandbox number or your number)  
     - Body: `🚨 EC2 Alert: Instance {{ $json.instanceId }} was unhealthy ({{ $json.instanceState }}) and has been restarted. Uptime: {{ $json.uptimeMinutes }} mins`  
   - Connect Restart Instance output to this node  

9. **Create Send Email Alert Node**  
   - Type: Email Send  
   - SMTP Credentials: Setup SMTP credentials for your mail server  
   - From Email: `alerts@yourcompany.com`  
   - To Email: `devops-team@yourcompany.com`  
   - Subject: `EC2 Health Alert - Instance Restarted`  
   - Connect Restart Instance output to this node  

10. **Create Log to AlertsLog Sheet Node**  
    - Type: Google Sheets  
    - Authentication: Google Service Account Credentials with access to target spreadsheet  
    - Operation: Append or Update  
    - Document ID: Your Google Sheets document ID  
    - Sheet Name: `Sheet1`  
    - Mapping Mode: Auto map input data fields to columns  
    - Connect Restart Instance output to this node  

11. **Create End Workflow Node**  
    - Type: No Operation (NoOp)  
    - Connect Send WhatsApp Alert, Send Email Alert, and Log to AlertsLog Sheet outputs to this node  

12. **Wiring Recap:**  
    - Schedule Trigger → Get EC2 Instances → Loop Over Instances  
    - Loop Over Instances (false branch) → Check Instance Status → Check Health Status  
    - Check Health Status:  
      - True → Analyze Health Data → Restart Instance → Send WhatsApp Alert, Send Email Alert, Log to AlertsLog Sheet → End Workflow  
      - False → Loop Over Instances (for next instance)  

13. **Credential Setup:**  
    - Twilio API credentials (Account SID + Auth Token)  
    - SMTP credentials for email sending  
    - Google Service Account credentials for Google Sheets access  

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow runs every 5 minutes using a cron schedule to ensure near-real-time monitoring and remediation.                          | Sticky Note 1: Workflow Configuration                        |
| Alerts are sent via Email, WhatsApp, and logged to Google Sheets to provide comprehensive visibility and audit trail.             | Sticky Note 2: Alert Notifications                            |
| Self-healing logic only triggers restart actions on instances unhealthy for more than 10 minutes to prevent unnecessary restarts. | Sticky Note 3: Self-Healing Actions                           |
| Twilio WhatsApp API requires proper sandbox or production setup; phone numbers must be configured correctly.                       | Twilio API Docs: https://www.twilio.com/docs/whatsapp/api    |
| Google Sheets API requires service account with write permissions to the specified spreadsheet.                                    | Google Sheets API Docs: https://developers.google.com/sheets/api |
| Custom API endpoints (`/ec2`, `/ec2status`, `/ec2restart`) must be accessible and correctly implemented for this workflow to function. | Internal API endpoints (custom, secured)                     |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.