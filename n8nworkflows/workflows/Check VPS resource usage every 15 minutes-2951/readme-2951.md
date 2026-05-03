Check VPS resource usage every 15 minutes

https://n8nworkflows.xyz/workflows/check-vps-resource-usage-every-15-minutes-2951


# Check VPS resource usage every 15 minutes

### 1. Workflow Overview

This workflow automates monitoring of key VPS (Virtual Private Server) resource metrics—CPU, RAM, and Disk usage—every 15 minutes. It is designed for system administrators, DevOps professionals, and IT support teams who require proactive alerts when resource usage exceeds critical thresholds, helping prevent server performance degradation or downtime.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow execution every 15 minutes.
- **1.2 Resource Usage Checks:** Executes SSH commands to retrieve current CPU, RAM, and Disk usage metrics from the VPS.
- **1.3 Data Aggregation:** Combines the individual resource metrics into a single data payload.
- **1.4 Threshold Evaluation:** Assesses whether any resource usage exceeds the 80% threshold.
- **1.5 Alert Notification:** Sends an email alert if any resource is above the threshold.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block triggers the entire workflow every 15 minutes to perform resource checks automatically.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution on a fixed time interval.  
    - Configuration: Set to trigger every 15 minutes using the interval rule.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Check RAM usage" node to start resource checks.  
    - Version: 1.2  
    - Edge Cases: If the node is disabled or misconfigured, the workflow will not run automatically. Timezone considerations may affect trigger timing.

#### 1.2 Resource Usage Checks

- **Overview:**  
  This block runs SSH commands on the VPS to fetch current CPU, RAM, and Disk usage statistics.

- **Nodes Involved:**  
  - Check RAM usage  
  - Check Disk usage  
  - Check CPU usage

- **Node Details:**  
  - **Check RAM usage**  
    - Type: SSH  
    - Role: Executes a Linux command to calculate RAM usage percentage.  
    - Configuration: Runs `free | awk '/Mem:/ {printf "%.2f", (1 - $7/$2) * 100}'` which calculates RAM used as a percentage.  
    - Inputs: Triggered by "Schedule Trigger" node.  
    - Outputs: Connects to "Check Disk usage" node.  
    - Credentials: Uses SSH Password credentials configured for the VPS.  
    - Version: 1  
    - Edge Cases: SSH connection failures, command syntax errors, or unexpected output format may cause failures.  
  - **Check Disk usage**  
    - Type: SSH  
    - Role: Executes a command to get disk usage percentage of the root partition.  
    - Configuration: Runs `df -h | awk '$NF=="/"{printf "%.2f", $5}'` to extract disk usage on `/`.  
    - Inputs: Triggered by "Check RAM usage" node.  
    - Outputs: Connects to "Check CPU usage" node.  
    - Credentials: Same SSH Password credentials.  
    - Version: 1  
    - Edge Cases: SSH failures, disk mount point changes, or unexpected command output.  
  - **Check CPU usage**  
    - Type: SSH  
    - Role: Runs a command to calculate current CPU usage percentage.  
    - Configuration: Runs `top -bn 1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}'` to get CPU usage.  
    - Inputs: Triggered by "Check Disk usage" node.  
    - Outputs: Connects to "Merge check results" node.  
    - Credentials: Same SSH Password credentials.  
    - Version: 1  
    - Edge Cases: SSH failures, command output format changes, or parsing errors.

#### 1.3 Data Aggregation

- **Overview:**  
  This block merges the outputs from the three SSH nodes into a single JSON object containing CPU, RAM, and Disk usage values.

- **Nodes Involved:**  
  - Merge check results

- **Node Details:**  
  - **Merge check results**  
    - Type: Merge  
    - Role: Combines three separate inputs into one consolidated output.  
    - Configuration: Uses SQL mode with a custom query to join inputs by their node names/IDs, mapping outputs to CPU, Disk, and RAM fields.  
    - Inputs: Receives data from "Check CPU usage" (input1), "Check Disk usage" (input2), and "Check RAM usage" (input3).  
    - Outputs: Connects to "Check results against thresholds" node.  
    - Version: 3  
    - Edge Cases: If any input is missing or delayed, the merge may fail or produce incomplete data. SQL query errors or mismatches in input names can cause issues.

#### 1.4 Threshold Evaluation

- **Overview:**  
  This block evaluates if any resource usage exceeds the 80% threshold, determining whether to send an alert.

- **Nodes Involved:**  
  - Check results against thresholds

- **Node Details:**  
  - **Check results against thresholds**  
    - Type: If  
    - Role: Conditional node that checks if CPU, RAM, or Disk usage is greater than or equal to 80%.  
    - Configuration: Uses numeric conditions combined with "any" logic so that if any resource exceeds threshold, the condition passes.  
    - Inputs: Receives merged resource data.  
    - Outputs: On true, connects to "Send Email" node; on false, no further action.  
    - Version: 1  
    - Edge Cases: If input data is missing or non-numeric, the condition may fail or produce false negatives.

#### 1.5 Alert Notification

- **Overview:**  
  Sends an email alert to notify administrators when resource usage is high.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**  
  - **Send Email**  
    - Type: Email Send  
    - Role: Sends an email notification with resource usage details.  
    - Configuration:  
      - Subject: "System Resource Alert"  
      - To and From Email: Placeholder addresses "change@me.com" (must be updated)  
      - Body: Text message includes CPU, RAM, and Disk usage values formatted to two decimal places.  
    - Inputs: Triggered by "Check results against thresholds" node on true condition.  
    - Credentials: SMTP credentials configured for sending email.  
    - Version: 1  
    - Edge Cases: Email sending failures due to SMTP misconfiguration, invalid email addresses, or network issues.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                  | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                   |
|---------------------------|--------------------|--------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger   | Initiates workflow every 15 min | None                         | Check RAM usage              |                                                                                              |
| Check RAM usage           | SSH                | Retrieves RAM usage via SSH     | Schedule Trigger             | Check Disk usage             |                                                                                              |
| Check Disk usage          | SSH                | Retrieves Disk usage via SSH    | Check RAM usage              | Check CPU usage              |                                                                                              |
| Check CPU usage           | SSH                | Retrieves CPU usage via SSH     | Check Disk usage             | Merge check results          |                                                                                              |
| Merge check results       | Merge              | Combines resource data          | Check CPU usage, Check Disk usage, Check RAM usage | Check results against thresholds |                                                                                              |
| Check results against thresholds | If                 | Evaluates if usage exceeds 80%  | Merge check results          | Send Email                  | Sticky Note2: "Update threshold\nIf needed, you can increase/decrease the 80% threshold in this node individually per resource" |
| Send Email                | Email Send         | Sends alert email               | Check results against thresholds | None                        | Sticky Note1: "Update email addresses\nUpdate From and To email addresses in this node to receive notifications" |
| Sticky Note               | Sticky Note        | Workflow description            | None                         | None                        | "Check VPS resource usage every 15 minutes\nThis workflow checks VPS CPU, RAM and Disk usage every 15 minutes and if any of it exceeds 80% will inform you by email" |
| Sticky Note1              | Sticky Note        | Email configuration reminder   | None                         | None                        | "Update email addresses\nUpdate From and To email addresses in this node to receive notifications" |
| Sticky Note2              | Sticky Note        | Threshold configuration reminder | None                         | None                        | "Update threshold\nIf needed, you can increase/decrease the 80% threshold in this node individually per resource" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 15 minutes (minutesInterval: 15).  
   - No credentials needed.

2. **Create SSH Node for RAM Usage**  
   - Type: SSH  
   - Command: `free | awk '/Mem:/ {printf "%.2f", (1 - $7/$2) * 100}'`  
   - Credentials: Configure SSH Password credentials with VPS access.  
   - Connect Schedule Trigger output to this node input.

3. **Create SSH Node for Disk Usage**  
   - Type: SSH  
   - Command: `df -h | awk '$NF=="/"{printf "%.2f", $5}'`  
   - Credentials: Use same SSH credentials as RAM node.  
   - Connect output of "Check RAM usage" node to this node input.

4. **Create SSH Node for CPU Usage**  
   - Type: SSH  
   - Command: `top -bn 1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}'`  
   - Credentials: Use same SSH credentials.  
   - Connect output of "Check Disk usage" node to this node input.

5. **Create Merge Node to Aggregate Results**  
   - Type: Merge  
   - Mode: SQL  
   - Number of Inputs: 3  
   - SQL Query:  
     ```
     SELECT input1.stdout as CPU, input2.stdout as Disk, input3.stdout as RAM 
     FROM input1 
     LEFT JOIN input2 ON input1.name = input2.id 
     LEFT JOIN input3 ON input1.name = input3.id
     ```  
   - Connect outputs of "Check CPU usage" (input1), "Check Disk usage" (input2), and "Check RAM usage" (input3) accordingly.

6. **Create If Node to Check Thresholds**  
   - Type: If  
   - Conditions: Numeric  
     - CPU >= 80  
     - Disk >= 80  
     - RAM >= 80  
   - Combine operation: Any (true if any condition met)  
   - Connect output of Merge node to this node input.

7. **Create Email Send Node**  
   - Type: Email Send  
   - Subject: "System Resource Alert"  
   - To Email: Set to desired recipient email address.  
   - From Email: Set to sender email address.  
   - Text:  
     ```
     System resources are above the threshold.

     CPU: {{ $json.CPU.toNumber().round(2) }}%
     RAM: {{ $json.RAM.toNumber().round(2) }}%
     Disk: {{ $json.Disk.toNumber().round(2) }}%
     ```  
   - Credentials: Configure SMTP credentials for sending email.  
   - Connect "true" output of If node to this node input.

8. **Optional: Add Sticky Notes**  
   - Add notes for workflow description, threshold update reminder, and email configuration reminder for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is designed for Linux-based VPS servers and requires SSH access with password authentication. | Ensure SSH credentials are securely stored and updated if server details change.               |
| Email addresses in the Send Email node must be updated to valid sender and recipient addresses to receive alerts. | SMTP credentials must be correctly configured to avoid email delivery failures.                |
| Thresholds can be adjusted in the If node to customize sensitivity for alerts per resource.    | Adjust numeric values in the If node conditions as needed.                                     |
| The workflow uses a SQL Merge node with a custom query to combine SSH command outputs.         | Ensure node names and input connections match the query logic to avoid merge errors.           |
| For more advanced monitoring, consider integrating with external monitoring tools or dashboards. | n8n can be extended with additional nodes or sub-workflows for richer alerting and reporting. |

---

This documentation provides a complete, structured reference for understanding, reproducing, and maintaining the "Check VPS resource usage every 15 minutes" workflow in n8n.