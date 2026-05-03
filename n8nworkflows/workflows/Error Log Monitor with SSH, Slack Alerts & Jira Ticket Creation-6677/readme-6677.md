Error Log Monitor with SSH, Slack Alerts & Jira Ticket Creation

https://n8nworkflows.xyz/workflows/error-log-monitor-with-ssh--slack-alerts---jira-ticket-creation-6677


# Error Log Monitor with SSH, Slack Alerts & Jira Ticket Creation

### 1. Workflow Overview

This workflow is designed to monitor application error logs on a remote server via SSH, detect critical and non-critical errors, send Slack alerts accordingly, and automatically create Jira bug tickets for critical errors.  
It targets DevOps, Site Reliability Engineering (SRE), or incident response use cases where automatic error detection and alerting help reduce response times and improve issue tracking.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Configuration Setup:** Initiates the workflow manually or via schedule and sets base parameters.  
- **1.2 Log Retrieval:** Connects to a remote server via SSH and reads recent error logs filtered for critical error levels.  
- **1.3 Log Parsing:** Processes raw log output to extract structured error entries with metadata.  
- **1.4 Error Filtering:** Separates critical errors from non-critical ones.  
- **1.5 Critical Error Handling:** Sends detailed Slack alerts and creates Jira tickets for critical errors.  
- **1.6 Non-Critical Error Handling:** Sends simpler Slack alerts for lower severity errors.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration Setup

- **Overview:** This block triggers the workflow manually or automatically every 5 minutes and sets initial configuration parameters.  
- **Nodes Involved:**  
  - Manual Trigger  
  - Schedule Every 5min  
  - Set Config  
- **Node Details:**

| Node Name       | Details                                                                                   |
|-----------------|-------------------------------------------------------------------------------------------|
| Manual Trigger  | Type: Manual trigger node. Allows manual start for testing or on-demand execution. No parameters required. Outputs trigger event to next node. |
| Schedule Every 5min | Type: Schedule trigger node. Configured to trigger every 5 minutes using interval on "minutes". No credentials needed. Outputs trigger event to Set Config node. |
| Set Config      | Type: Set node to prepare or reset any necessary workflow variables or parameters. Currently empty options, serving as placeholder for config. Receives trigger input and passes on to next node. No expressions or credentials used.|

- **Edge Cases:**  
  - Manual trigger requires user intervention.  
  - Schedule trigger depends on n8n scheduler and must be enabled in instance.  
  - Set Config currently has no parameters but could be extended to include environment variables or thresholds.  

---

#### 1.2 Log Retrieval

- **Overview:** Connects to the remote server using SSH with private key authentication and retrieves the last 50 lines of the application error log filtered for significant errors (CRITICAL, ERROR, FATAL).  
- **Nodes Involved:**  
  - Read Error Logs  
  - Wait For All Logs  
- **Node Details:**

| Node Name       | Details                                                                                   |
|-----------------|-------------------------------------------------------------------------------------------|
| Read Error Logs | Type: SSH node. Runs shell command `tail -n 50 /var/log/application/error.log | grep -E '(CRITICAL|ERROR|FATAL)' || echo 'No critical errors found'`. Uses private key SSH authentication (credential "SSH Password account - test"). Output is STDOUT text of filtered logs. Connects to Wait For All Logs node. Failure modes include SSH connection issues, authentication failure, command timeout, or missing log file. |
| Wait For All Logs | Type: Wait node. Used here likely to ensure all data from SSH node is ready before parsing. No delay configured, simply passes input to next node. |

- **Edge Cases:**  
  - SSH connection failure due to network or key issues.  
  - No critical errors found returns a default message handled downstream.  
  - Log file path or permissions issues may cause empty or error output.

---

#### 1.3 Log Parsing

- **Overview:** Converts raw log lines into structured JSON objects with assigned error levels, timestamps, unique IDs, and raw log text for downstream processing.  
- **Nodes Involved:**  
  - Parse Logs  
- **Node Details:**

| Node Name  | Details                                                                                   |
|------------|-------------------------------------------------------------------------------------------|
| Parse Logs | Type: Code node with JavaScript. Reads stdout from SSH node, splits by newline, filters out empty lines and the "No critical errors found" message. Assigns error level ("CRITICAL" if line contains CRITICAL or FATAL else "ERROR"), generates timestamp (current ISO string), creates unique error ID using current timestamp and index, and prepares JSON array with error details. Outputs one JSON object per error line. Input from Wait For All Logs, output to IF Critical Error node. Possible failures include empty input, unexpected stdout format, or script errors.|

- **Key Expressions:**  
  - `$input.first().json.stdout` - raw log output.  
  - Dynamic error ID: `ERR-${Date.now()}-${index}`.  

- **Edge Cases:**  
  - No logs or only the default message results in empty output array.  
  - Unexpected log format may cause parsing errors.  

---

#### 1.4 Error Filtering

- **Overview:** Determines if each error is critical or not to route to appropriate alert or ticket creation logic.  
- **Nodes Involved:**  
  - IF Critical Error  
- **Node Details:**

| Node Name       | Details                                                                                   |
|-----------------|-------------------------------------------------------------------------------------------|
| IF Critical Error | Type: If node. Checks if `$json.level` equals "CRITICAL". If true, routes to critical error handling; else routes to non-critical error handling. Single field condition, no expressions beyond variable check. |

- **Edge Cases:**  
  - Errors missing or incorrectly labeled "level" property may be misrouted.  
  - Multiple errors processed independently; each triggers separate downstream nodes.

---

#### 1.5 Critical Error Handling

- **Overview:** Sends a detailed Slack alert for critical errors and creates a Jira bug ticket with full error details.  
- **Nodes Involved:**  
  - Send Slack Alert  
  - Create Jira Ticket  
- **Node Details:**

| Node Name       | Details                                                                                   |
|-----------------|-------------------------------------------------------------------------------------------|
| Send Slack Alert | Type: Slack node using webhook. Sends message "🚨 CRITICAL ERROR DETECTED" to Slack channel ID "C0987654". Uses Slack API credential "Slack account - test". No message customization beyond static text. Intended to alert immediately on critical errors. Possible failures: invalid webhook, permission issues, network failure. |
| Create Jira Ticket | Type: Jira node. Creates a new issue in project "BUG" of type "Bug". Summary includes error ID, priority set to "Highest", labels "auto-generated" and "critical-error". Description includes formatted details with error ID, level, timestamp, message, raw log, with markdown formatting. Uses Jira Software Cloud API credentials "Jira SW Cloud - test". Possible failures: API auth errors, invalid project or issue type, rate limits. |

- **Key Expressions:**  
  - Summary: `Critical Error: {{ $json.id }}`  
  - Description includes multiline markdown code blocks embedding error details.  

- **Edge Cases:**  
  - Slack or Jira service downtime or authentication failures impact alerting or ticket creation.  
  - Jira quotas or permission issues may prevent ticket creation.  
  - Simultaneous multiple critical errors lead to multiple tickets.

---

#### 1.6 Non-Critical Error Handling

- **Overview:** Sends a simpler Slack alert for errors that are not critical, indicating the error level.  
- **Nodes Involved:**  
  - Send Non-Critical Alert  
- **Node Details:**

| Node Name           | Details                                                                                   |
|---------------------|-------------------------------------------------------------------------------------------|
| Send Non-Critical Alert | Type: Slack node using webhook. Sends message "⚠️ Error Detected: {{ $json.level }}" to Slack channel ID "C098765". Uses same Slack API credentials as critical alert node. Less detailed, no ticket creation triggered. Possible failures similar to Send Slack Alert node. |

- **Edge Cases:**  
  - If no errors or only critical errors present, this node is not executed.  
  - Slack channel must exist and allow bot posting.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                       | Input Node(s)        | Output Node(s)                    | Sticky Note                                                                                  |
|---------------------|---------------------|------------------------------------|----------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger       | Manual start for testing            | -                    | Set Config                      | For testing                                                                                  |
| Schedule Every 5min | Schedule Trigger     | Scheduled automatic trigger         | -                    | Set Config                      | Runs every 5 minutes automatically                                                          |
| Set Config          | Set                  | Initializes workflow parameters     | Manual Trigger, Schedule Every 5min | Read Error Logs                | Configures basic parameters                                                                 |
| Read Error Logs     | SSH                  | Reads latest error logs via SSH     | Set Config            | Wait For All Logs               | SSH into server and reads error logs                                                        |
| Wait For All Logs   | Wait                 | Synchronizes flow before parsing    | Read Error Logs       | Parse Logs                     | Wait for all error logs read                                                                 |
| Parse Logs          | Code                 | Parses raw logs into structured errors | Wait For All Logs      | IF Critical Error              | JavaScript code to parse and categorize errors                                              |
| IF Critical Error   | If                   | Routes errors based on severity     | Parse Logs            | Send Slack Alert, Create Jira Ticket, Send Non-Critical Alert | Filters critical errors from regular errors                                                  |
| Send Slack Alert    | Slack                 | Sends Slack alert for critical errors | IF Critical Error     | -                              | Sends detailed alert for critical errors                                                    |
| Create Jira Ticket  | Jira                  | Creates Jira bug ticket for critical errors | IF Critical Error     | -                              | Auto-creates bug ticket for critical errors                                                 |
| Send Non-Critical Alert | Slack              | Sends Slack alert for non-critical errors | IF Critical Error     | -                              | Sends simple alert for non-critical errors                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node with default settings (no parameters).  
   - Add a **Schedule Trigger** node, configure it to trigger every 5 minutes by setting the interval field for "minutes" to 5.

2. **Create Configuration Node:**  
   - Add a **Set** node named "Set Config". Leave it empty or add parameters as needed. Connect both triggers (Manual and Schedule) to this node's input.

3. **Create SSH Node to Read Logs:**  
   - Add an **SSH** node named "Read Error Logs".  
   - Configure authentication with a private key credential. Create or select existing SSH private key credential with access to the target server.  
   - Set command to:  
     ```
     tail -n 50 /var/log/application/error.log | grep -E '(CRITICAL|ERROR|FATAL)' || echo 'No critical errors found'
     ```  
   - Connect "Set Config" node output to this SSH node input.

4. **Add Wait Node:**  
   - Add a **Wait** node "Wait For All Logs" with default configuration (no delay). Connect the SSH node to this node.

5. **Add Code Node to Parse Logs:**  
   - Add a **Code** node "Parse Logs".  
   - Paste the following JavaScript code:  
     ```javascript
     const logOutput = $input.first().json.stdout || '';
     const lines = logOutput.split('\n').filter(line => line.trim() !== '' && line !== 'No critical errors found');

     const errors = [];

     lines.forEach((line, index) => {
       let errorLevel = 'ERROR';
       
       if (line.includes('CRITICAL') || line.includes('FATAL')) {
         errorLevel = 'CRITICAL';
       }
       
       const timestamp = new Date().toISOString();
       const errorId = `ERR-${Date.now()}-${index}`;
       
       errors.push({
         id: errorId,
         level: errorLevel,
         message: line.trim(),
         timestamp: timestamp,
         raw_log: line
       });
     });

     return errors.map(error => ({ json: error }));
     ```  
   - Connect "Wait For All Logs" node to this code node.

6. **Add If Node to Filter Critical Errors:**  
   - Add an **If** node "IF Critical Error".  
   - Set condition to check if:  
     - Field: Expression `{{$json.level}}`  
     - Operation: equals  
     - Value: `CRITICAL`  
   - Connect "Parse Logs" node to this node.

7. **Add Slack Node for Critical Alerts:**  
   - Add a **Slack** node "Send Slack Alert".  
   - Select existing Slack API credential or create new.  
   - Set message text to:  
     ```
     🚨 CRITICAL ERROR DETECTED
     ```  
   - Set target channel to the appropriate Slack channel by ID (e.g., "C0987654").  
   - Connect the **true** output of the If node to this node.

8. **Add Jira Node to Create Tickets:**  
   - Add a **Jira** node "Create Jira Ticket".  
   - Configure Jira Software Cloud API credentials.  
   - Set Project to "BUG" (or your project key).  
   - Set Issue Type to "Bug".  
   - Summary:  
     ```
     Critical Error: {{ $json.id }}
     ```  
   - Add labels: `auto-generated`, `critical-error`  
   - Priority: Highest  
   - Description (use markdown):  
     ```
     **Error Details:**

     Error ID: {{ $json.id }}
     Level: {{ $json.level }}
     Timestamp: {{ $json.timestamp }}

     **Error Message:**
     ```
     {{ $json.message }}
     ```

     **Raw Log:**
     ```
     {{ $json.raw_log }}
     ```

     *This ticket was automatically created by the Error Monitoring System.*
     ```  
   - Connect the **true** output of the If node also to this node (parallel to Slack alert).

9. **Add Slack Node for Non-Critical Alerts:**  
   - Add a **Slack** node "Send Non-Critical Alert".  
   - Use the same Slack API credential as above.  
   - Message text:  
     ```
     ⚠️ Error Detected: {{ $json.level }}
     ```  
   - Set channel ID to another appropriate Slack channel (e.g., "C098765").  
   - Connect the **false** output of the If node to this node.

10. **Final Connections and Testing:**  
    - Verify all node connections as per above.  
    - Activate and test with manual trigger initially to confirm SSH connectivity, parsing, Slack messages, and Jira ticket creation.  
    - Enable schedule trigger for automated periodic monitoring.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                         |
|------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| For testing, use the Manual Trigger node to start the workflow on demand.    | Sticky Note on Manual Trigger node                                                     |
| The workflow is designed to run every 5 minutes automatically via Schedule Trigger. | Sticky Note on Schedule Every 5min node                                               |
| SSH node reads the last 50 lines of the error log and filters for critical keywords. | Sticky Note on Read Error Logs node                                                   |
| Code node uses JavaScript to parse and categorize errors into structured JSON. | Sticky Note on Parse Logs node                                                        |
| Wait node ensures all log data is ready before parsing starts.               | Sticky Note on Wait For All Logs node                                                 |
| Critical errors trigger detailed Slack alert and Jira ticket creation.       | Sticky Notes on Send Slack Alert and Create Jira Ticket nodes                         |
| Non-critical errors only trigger a simpler Slack alert.                      | Sticky Note on Send Non-Critical Alert node                                          |
| Slack channel IDs and Jira project keys must be adjusted to your environment.| Credentials and parameters need to be updated accordingly                             |
| SSH private key credential must have server access to /var/log/application/error.log. | Setup required outside n8n                                                            |
| Jira ticket description uses markdown formatting for readability.            | Jira API supports markdown formatting                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.