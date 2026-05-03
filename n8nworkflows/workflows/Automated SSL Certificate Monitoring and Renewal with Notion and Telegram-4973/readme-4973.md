Automated SSL Certificate Monitoring and Renewal with Notion and Telegram

https://n8nworkflows.xyz/workflows/automated-ssl-certificate-monitoring-and-renewal-with-notion-and-telegram-4973


# Automated SSL Certificate Monitoring and Renewal with Notion and Telegram

### 1. Workflow Overview

This workflow automates the monitoring and renewal process of SSL certificates using Notion as the data source and Telegram for notifications. It is designed to periodically check the expiration status of SSL certificates for a list of domains recorded in a Notion database, notify responsible parties if certificates are about to expire, trigger automated renewal via a server command, and update the monitoring data accordingly.

The workflow includes the following logical blocks:

- **1.1 Trigger & Input Handling:** Supports three types of triggers — manual execution, scheduled weekly execution, and execution triggered by another workflow. It sets context on how the workflow was triggered.

- **1.2 SSL Data Retrieval:** Fetches all SSL certificate records from a Notion database.

- **1.3 SSL Status Checking:** For each domain, requests current SSL certificate status via an external API, then processes and annotates the results.

- **1.4 Decision Making:** Evaluates if any certificates are near expiration (less than 14 days) and branches flow accordingly.

- **1.5 Notification Preparation & Sending:** Constructs notification messages listing expiring certificates and sends alerts through Telegram.

- **1.6 Renewal Execution:** If certificates are expiring, initiates certificate renewal on the server via SSH and delays further steps to allow renewal processes.

- **1.7 Data Updating:** Updates the Notion database with the latest SSL status and timestamps.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Handling

- **Overview:** Defines how the workflow is started, differentiating manual runs, scheduled runs, and invocations from other workflows. It tags the trigger type for downstream conditional logic.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Weekly Trigger  
  - When Executed by Another Workflow  
  - Set Click Triggered  
  - Set Weekly Triggered  
  - Set anotherWorkflow Triggered  
  - Select Who Triggered

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual start of workflow  
    - Output: Triggers flow to assign trigger type as "click"  
    - Edge Cases: User must manually execute; no input parameters

  - **Weekly Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers workflow weekly  
    - Config: Interval set to 1 week  
    - Edge Cases: Dependent on n8n scheduler running correctly

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Listens for invocation from another workflow  
    - Inputs: Accepts boolean parameter `has_expired_cert`  
    - Edge Cases: If parameter missing or malformed, downstream logic may fail

  - **Set Click Triggered, Set Weekly Triggered, Set anotherWorkflow Triggered**  
    - Type: Set  
    - Role: Tag input data with trigger identification string ("click", "weekly", or "anotherWorkflow")  
    - Output: Adds `triggerBy` field to JSON for flow control

  - **Select Who Triggered**  
    - Type: Merge  
    - Role: Combines all three trigger outputs into one unified flow  
    - Configuration: Number of inputs set to 3 (manual, weekly, anotherWorkflow)  
    - Edge Cases: Ensures only one trigger path proceeds

---

#### 1.2 SSL Data Retrieval

- **Overview:** Retrieves all SSL certificate entries from the Notion database to check their current status.

- **Nodes Involved:**  
  - Get SSL Info

- **Node Details:**

  - **Get SSL Info**  
    - Type: Notion (databasePage:getAll)  
    - Role: Fetches all pages (records) from configured Notion database  
    - Credentials: Uses configured Notion API credentials  
    - Config: Returns all records without pagination  
    - Edge Cases: API rate limiting, Notion API outages, permission errors  

---

#### 1.3 SSL Status Checking

- **Overview:** For each SSL record, queries an external SSL checking API and annotates results with a status indicating certificate health.

- **Nodes Involved:**  
  - Check SSL  
  - Refresh SSL Status

- **Node Details:**

  - **Check SSL**  
    - Type: HTTP Request  
    - Role: Calls `https://ssl-checker.io/api/v1/check/{domain}` for each domain in Notion data  
    - URL Parameter: Injects domain name dynamically from `$json.name`  
    - Edge Cases: Network timeouts, API errors, invalid domain names

  - **Refresh SSL Status**  
    - Type: Code (JavaScript)  
    - Role: Processes API response to categorize SSL status by `days_left`  
    - Logic:  
      - If days_left < 14 → status: "It's about to expire."  
      - Else if days_left ≥ 14 → status: "Normal"  
      - Else → status: "Unavailable"  
    - Outputs annotated JSON for each item  
    - Edge Cases: Missing or malformed API response fields, exceptions in script

---

#### 1.4 Decision Making

- **Overview:** Determines whether any certificates are close to expiration and routes the workflow accordingly.

- **Nodes Involved:**  
  - If (days_left < 14)  
  - If3 (checks array length differences)  
  - If1 (checks trigger type for notification flow)

- **Node Details:**

  - **If**  
    - Type: If Node  
    - Role: Checks if any SSL cert has less than 14 days left (`$('Refresh SSL Status').item.json.result.days_left < 14`)  
    - True branch: Certificates expiring → proceed with renewal and notifications  
    - False branch: Certificates healthy → update statuses accordingly  
    - Edge Cases: Missing `days_left` data, expression evaluation errors

  - **If3**  
    - Type: If Node  
    - Role: Checks if the number of items in the true path does not equal total SSL entries, to handle partial updates  
    - Condition: Compares length of `If` node output array with `Get SSL Info` items length  
    - Edge Cases: Data synchronization issues

  - **If1**  
    - Type: If Node  
    - Role: Checks if the workflow was triggered by another workflow (`triggerBy == "anotherWorkflow"`)  
    - True branch: Send confirmation Telegram message for normal certs  
    - Edge Cases: Missing `triggerBy` field

---

#### 1.5 Notification Preparation & Sending

- **Overview:** Constructs a message listing expiring certificates and sends notifications to Telegram.

- **Nodes Involved:**  
  - Merge  
  - Merge Content  
  - Telegram  
  - Telegram1

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Chooses between branches of If and If3 nodes to combine SSL data for notification  
    - Mode: Choose branch mode  
    - Edge Cases: Ensures correct merging of branches

  - **Merge Content**  
    - Type: Code (JavaScript)  
    - Role: Builds a textual notification listing domains with expiring certificates and remaining days  
    - Output: Text string in `output` field  
    - Edge Cases: Empty lists, message formatting

  - **Telegram**  
    - Type: Telegram  
    - Role: Sends notification message about expiring certificates  
    - Text: Includes workflow name and constructed message from `Merge Content`  
    - Credentials: Requires Telegram API credentials  
    - Edge Cases: Telegram API rate limits, invalid bot tokens

  - **Telegram1**  
    - Type: Telegram  
    - Role: Sends confirmation message that all certificates are normal after update  
    - Text: Fixed message indicating successful update and normal status  
    - Edge Cases: Same as above

---

#### 1.6 Renewal Execution

- **Overview:** Executes certificate renewal on the server via SSH and waits before continuing.

- **Nodes Involved:**  
  - Renew Server Cert  
  - Wait  
  - Execute Workflow

- **Node Details:**

  - **Renew Server Cert**  
    - Type: SSH  
    - Role: Runs `sudo certbot renew` on remote server to renew certificates  
    - Config: Uses private key authentication with specified working directory  
    - Credentials: SSH private key credentials required  
    - Edge Cases: SSH connection failures, permission denied, command errors

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow after renewal command to allow process completion  
    - Config: No explicit delay parameter shown; likely webhook-based wait or default delay  
    - Edge Cases: Timeout or premature continuation

  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Triggers a sub-workflow (ID `WO0anosBgPhhJ0Qi`) named "SSL Check Service" with input indicating expired certs  
    - Inputs: `has_expired_cert` set to true  
    - Edge Cases: Sub-workflow failures, incorrect workflow ID

---

#### 1.7 Data Updating

- **Overview:** Updates Notion database records with new SSL certificate dates, statuses, and check timestamps.

- **Nodes Involved:**  
  - Update SSL Info

- **Node Details:**

  - **Update SSL Info**  
    - Type: Notion (databasePage:update)  
    - Role: Updates each Notion page with new certificate data and status  
    - Fields updated:  
      - Creation Date (valid_from)  
      - Expiration Date (valid_till)  
      - SSL Status (select field, localized name "SSL 狀態")  
      - Last Check At (current timestamp)  
    - Page ID: Dynamically mapped from `Get SSL Info` results  
    - Credentials: Notion API credentials required  
    - Edge Cases: Permission errors, API rate limits, invalid page IDs

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                               | Input Node(s)                            | Output Node(s)                          | Sticky Note                                                                                      |
|-----------------------------|-------------------------|-----------------------------------------------|----------------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Manual workflow start                         |                                        | Set Click Triggered                     |                                                                                                |
| Weekly Trigger              | Schedule Trigger         | Weekly scheduled start                        |                                        | Set Weekly Triggered                    |                                                                                                |
| When Executed by Another Workflow | Execute Workflow Trigger | Triggered by external workflow invocation    |                                        | Set anotherWorkflow Triggered           |                                                                                                |
| Set Click Triggered         | Set                     | Tag workflow execution type as "click"       | When clicking ‘Execute workflow’        | Select Who Triggered                    |                                                                                                |
| Set Weekly Triggered        | Set                     | Tag workflow execution type as "weekly"      | Weekly Trigger                         | Select Who Triggered                    |                                                                                                |
| Set anotherWorkflow Triggered | Set                   | Tag workflow execution type as "anotherWorkflow" | When Executed by Another Workflow      | Select Who Triggered                    |                                                                                                |
| Select Who Triggered        | Merge                   | Consolidate trigger sources                   | Set Click Triggered, Set Weekly Triggered, Set anotherWorkflow Triggered | Get SSL Info                          |                                                                                                |
| Get SSL Info                | Notion                  | Retrieve all SSL certificate entries          | Select Who Triggered                   | Check SSL                             |                                                                                                |
| Check SSL                  | HTTP Request            | Call external SSL checker API for each domain | Get SSL Info                          | Refresh SSL Status                     |                                                                                                |
| Refresh SSL Status          | Code                    | Annotate SSL status based on days left        | Check SSL                            | If                                    |                                                                                                |
| If                         | If                      | Check if any cert expires in less than 14 days | Refresh SSL Status                    | Merge (true branch), If3 (false branch) |                                                                                                |
| If3                        | If                      | Validate if output array length matches input | If                                  | Merge (false branch), Update SSL Info |                                                                                                |
| Merge                      | Merge                   | Merge certificate data from two branches      | If, If3                             | Merge Content                         |                                                                                                |
| Merge Content              | Code                    | Prepare notification message content           | Merge                               | Telegram                             |                                                                                                |
| Telegram                   | Telegram                 | Send notification about expiring certificates | Merge Content                      | Renew Server Cert                     |                                                                                                |
| Renew Server Cert          | SSH                     | Run certbot renew command on server            | Telegram                           | Wait                                |                                                                                                |
| Wait                       | Wait                    | Delay for renewal completion                    | Renew Server Cert                   | Execute Workflow                     |                                                                                                |
| Execute Workflow           | Execute Workflow        | Trigger sub-workflow for SSL checking          | Wait                              |                                        |                                                                                                |
| Update SSL Info            | Notion                  | Update SSL certificate info in Notion           | If3                               | If1                                 |                                                                                                |
| If1                        | If                      | Check if triggered by another workflow          | Update SSL Info                    | Telegram1                           |                                                                                                |
| Telegram1                  | Telegram                 | Send confirmation message of normal SSL status | If1                               |                                        |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers**

   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’".
   - Add a **Schedule Trigger** node named "Weekly Trigger" with interval set to 1 week.
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow" accepting a boolean input parameter `has_expired_cert`.

2. **Set Trigger Tags**

   - Add three **Set** nodes:
     - "Set Click Triggered" sets `triggerBy` = "click", connected from manual trigger.
     - "Set Weekly Triggered" sets `triggerBy` = "weekly", connected from schedule trigger.
     - "Set anotherWorkflow Triggered" sets `triggerBy` = "anotherWorkflow", connected from execute workflow trigger.

3. **Merge Trigger Paths**

   - Add a **Merge** node named "Select Who Triggered" with 3 inputs.
   - Connect the three Set nodes to the Merge inputs.

4. **Retrieve SSL Data**

   - Add a **Notion** node named "Get SSL Info".
   - Configure it to get all pages from the SSL certificates database.
   - Connect "Select Who Triggered" output to "Get SSL Info" input.
   - Set Notion credentials with appropriate API access.

5. **Check SSL Certificates**

   - Add an **HTTP Request** node named "Check SSL".
   - Configure URL as `https://ssl-checker.io/api/v1/check/{{ $json.name }}` (use expression to inject domain name).
   - Connect "Get SSL Info" output to "Check SSL" input.

6. **Process SSL Status**

   - Add a **Code** node named "Refresh SSL Status".
   - Use JavaScript code to annotate each item with `ssl_status` based on `result.days_left`:
     ```
     for (const item of $input.all()) {
       if (item.json.result.days_left < 14) {
         item.json.ssl_status = "It's about to expire.";
       } else if (item.json.result.days_left >= 14) {
         item.json.ssl_status = "Normal";
       } else {
         item.json.ssl_status = "Unavailable";
       }
     }
     return $input.all();
     ```
   - Connect "Check SSL" output to this node.

7. **Check Expiration**

   - Add an **If** node named "If".
   - Condition: Check if `$('Refresh SSL Status').item.json.result.days_left < 14`.
   - Connect "Refresh SSL Status" output to "If" input.

8. **Handle Partial Updates**

   - Add an **If** node named "If3".
   - Condition: Check if length of `If.all()` output array not equal to length of `Get SSL Info` outputs.
   - Connect the false branch of "If" to "If3" input.

9. **Merge Results**

   - Add a **Merge** node named "Merge".
   - Set mode to "Choose Branch".
   - Connect true branch of "If" and both outputs of "If3" to this node.

10. **Prepare Notification Content**

    - Add a **Code** node named "Merge Content".
    - Use code to build notification text listing expiring certificates with days left.
    - Connect "Merge" output to this node.

11. **Send Telegram Notifications**

    - Add a **Telegram** node named "Telegram".
    - Configure message text to include workflow name and notification content.
    - Use Telegram credentials with bot token.
    - Connect "Merge Content" output to this node.

12. **Renew Certificates**

    - Add an **SSH** node named "Renew Server Cert".
    - Configure command: `sudo certbot renew`.
    - Set working directory as needed.
    - Use SSH private key credentials.
    - Connect "Telegram" output to this node.

13. **Wait for Renewal**

    - Add a **Wait** node named "Wait".
    - Connect "Renew Server Cert" output to it.

14. **Execute Sub-Workflow**

    - Add an **Execute Workflow** node named "Execute Workflow".
    - Configure to call the sub-workflow "SSL Check Service" with input `has_expired_cert` = true.
    - Connect "Wait" output to this node.

15. **Update Notion Data**

    - Connect the false branch of "If3" to a **Notion** node named "Update SSL Info".
    - Configure to update the respective Notion page for each SSL certificate:
      - Set `Creation Date` to `result.valid_from`
      - Set `Expiration Date` to `result.valid_till`
      - Set `SSL 狀態` select field to `ssl_status`
      - Set `Last Check At` to current timestamp
    - Use Notion API credentials.

16. **Conditional Confirmation Notification**

    - Add an **If** node named "If1".
    - Condition: Check if `triggerBy` equals `"anotherWorkflow"`.
    - Connect "Update SSL Info" output to "If1".

17. **Send Confirmation Telegram**

    - Add a **Telegram** node named "Telegram1".
    - Configure fixed message confirming all certificates are normal.
    - Connect true branch of "If1" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                          |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| The Telegram nodes require a bot token and chat ID configured in credentials to send messages properly.      | Telegram API documentation: https://core.telegram.org/bots/api                                                            |
| Notion nodes require integration token with access to the specific SSL certificate database.                  | Notion API docs: https://developers.notion.com/docs/getting-started                                                        |
| SSH node requires private key authentication setup for the server hosting certbot, and proper sudo permissions.| Certbot documentation: https://certbot.eff.org/docs/using.html                                                            |
| The external SSL checker API used is `https://ssl-checker.io/api/v1/check/{domain}` — ensure API access rights.| SSL Checker API docs: https://ssl-checker.io/api                                                                           |
| The sub-workflow "SSL Check Service" (ID `WO0anosBgPhhJ0Qi`) must be imported or created separately.          | Ensure this sub-workflow is available in your n8n instance for "Execute Workflow" node to function properly.              |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a low-code automation tool. All content complies with applicable content policies, contains no illegal or offensive material, and manipulates only legal and public data.