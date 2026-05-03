ServiceNow Incident Notifications to Slack Workflow

https://n8nworkflows.xyz/workflows/servicenow-incident-notifications-to-slack-workflow-2704


# ServiceNow Incident Notifications to Slack Workflow

### 1. Workflow Overview

This workflow automates the notification of new ServiceNow incidents to a designated Slack channel, targeting IT operations teams and system administrators who require real-time incident updates for efficient response and collaboration.

The workflow is logically divided into the following blocks:

- **1.1 Triggering Mechanism**: Supports both manual and scheduled triggers to initiate the workflow.
- **1.2 Incident Retrieval and Validation**: Calculates a timestamp 5 minutes prior, queries ServiceNow for incidents created since then, and checks if new incidents exist.
- **1.3 Incident Processing and Notification**: Sorts new incidents by incident number and posts detailed notifications to Slack.
- **1.4 Error Handling**: Detects errors in connecting to ServiceNow and posts an alert message to Slack.
- **1.5 No Incident Handling**: Gracefully ends the workflow if no new incidents are found.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering Mechanism

- **Overview:**  
  This block initiates the workflow either manually via a button click or automatically every 5 minutes, ensuring timely and flexible execution.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Run Every 5 Minutes (Schedule Trigger)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the workflow for testing or on-demand runs.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Get 5 Minute Ago Timestamp" node.  
    - Edge Cases: None significant; manual trigger is user-initiated.

  - **Run Every 5 Minutes**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every 5 minutes.  
    - Configuration: Interval set to every 5 minutes.  
    - Inputs: None  
    - Outputs: Connects to "Get 5 Minute Ago Timestamp" node.  
    - Edge Cases: Potential delays if workflow execution exceeds 5 minutes; no concurrency control specified.

---

#### 2.2 Incident Retrieval and Validation

- **Overview:**  
  This block calculates the timestamp for 5 minutes ago, fetches incidents created since then from ServiceNow, and checks if any new incidents exist to determine further processing.

- **Nodes Involved:**  
  - Get 5 Minute Ago Timestamp  
  - Get Incidents from ServiceNow  
  - Check if New Incidents

- **Node Details:**

  - **Get 5 Minute Ago Timestamp**  
    - Type: DateTime  
    - Role: Computes a UTC timestamp exactly 5 minutes before the current time.  
    - Configuration: Subtracts 5 minutes from current UTC time; outputs field named `queryDate`.  
    - Inputs: Trigger from either manual or schedule trigger.  
    - Outputs: Connects to "Get Incidents from ServiceNow".  
    - Edge Cases: Timezone consistency ensured by using UTC; no major failure expected.

  - **Get Incidents from ServiceNow**  
    - Type: ServiceNow  
    - Role: Queries ServiceNow for incidents created on or after the calculated timestamp.  
    - Configuration:  
      - Resource: Incident  
      - Operation: Get All  
      - Query: `sys_created_on>=<queryDate>` (dynamic expression)  
      - Display Value: true (returns human-readable fields)  
      - Authentication: Basic Auth credentials configured.  
    - Inputs: Receives timestamp from previous node.  
    - Outputs:  
      - Main output: Incident data to "Check if New Incidents" node.  
      - Error output: Continues to "Post Error Message if Error with ServiceNow" node on failure.  
    - Edge Cases:  
      - Authentication failures (expired or invalid credentials).  
      - Network timeouts or API errors.  
      - No incidents returned (empty dataset).  
    - Version-specific: Uses `alwaysOutputData` to ensure output even on errors.

  - **Check if New Incidents**  
    - Type: If  
    - Role: Determines if any new incidents were retrieved by checking for existence of `sys_id` in the response.  
    - Configuration: Condition checks if `sys_id` field exists in the JSON data.  
    - Inputs: Incident data from ServiceNow node.  
    - Outputs:  
      - True branch: To "Sort Incidents in Ascending Order" node.  
      - False branch: To "No Incidents, Do Nothing" node.  
    - Edge Cases:  
      - Empty or malformed data could cause condition to fail.  
      - Strict type validation ensures robustness.

---

#### 2.3 Incident Processing and Notification

- **Overview:**  
  This block sorts the new incidents by their incident number in ascending order and posts detailed, formatted notifications to a specified Slack channel, including a direct link to each incident.

- **Nodes Involved:**  
  - Sort Incidents in Ascending Order  
  - Post Incident Details to Slack Channel

- **Node Details:**

  - **Sort Incidents in Ascending Order**  
    - Type: Sort  
    - Role: Orders incidents by their `number` field ascendingly to maintain logical sequence.  
    - Configuration: Sort field set to `number` ascending.  
    - Inputs: Incident data from "Check if New Incidents" node (true branch).  
    - Outputs: Connects to "Post Incident Details to Slack Channel".  
    - Edge Cases:  
      - Missing or malformed `number` fields could affect sorting.

  - **Post Incident Details to Slack Channel**  
    - Type: Slack  
    - Role: Sends a richly formatted Slack message for each incident to a designated channel.  
    - Configuration:  
      - Message Type: Block Kit  
      - Channel: Specified Slack channel ID (configured via credentials and channelId).  
      - Message Content:  
        - Header: "ServiceNow Incident Notification"  
        - Section fields: Incident ID, Description, Severity, Caller, Priority, State, Category, Date Opened (all dynamically populated from incident data).  
        - Action Button: "View Incident" linking directly to the incident in ServiceNow via URL constructed with `sys_id`.  
      - Markdown enabled.  
      - No link to workflow included.  
    - Inputs: Sorted incident data.  
    - Outputs: None (end of processing chain).  
    - Edge Cases:  
      - Slack API rate limits or authentication errors.  
      - Missing incident fields could cause message formatting issues.  
      - URL construction depends on correct `sys_id` presence.

---

#### 2.4 Error Handling

- **Overview:**  
  This block handles errors encountered during the ServiceNow API call by posting an alert message to Slack, ensuring the team is promptly informed of connectivity or authentication issues.

- **Nodes Involved:**  
  - Post Error Message if Error with ServiceNow

- **Node Details:**

  - **Post Error Message if Error with ServiceNow**  
    - Type: Slack  
    - Role: Posts a predefined error alert to Slack if the ServiceNow node fails.  
    - Configuration:  
      - Text: "🚨 Issue connecting to ServiceNow. Please investigate error in n8n. 🚨"  
      - Channel: Same Slack channel as incident notifications.  
      - Markdown enabled.  
      - No link to workflow included.  
    - Inputs: Error output from "Get Incidents from ServiceNow" node.  
    - Outputs: None (end of error handling chain).  
    - Edge Cases:  
      - Slack API failures or invalid credentials.  
      - Multiple errors in quick succession could cause message flooding.

---

#### 2.5 No Incident Handling

- **Overview:**  
  This block gracefully ends the workflow without action if no new incidents are found, avoiding unnecessary notifications.

- **Nodes Involved:**  
  - No Incidents, Do Nothing

- **Node Details:**

  - **No Incidents, Do Nothing**  
    - Type: NoOp (No Operation)  
    - Role: Terminates the workflow path when no new incidents exist.  
    - Configuration: Default no-op with no parameters.  
    - Inputs: False branch from "Check if New Incidents".  
    - Outputs: None.  
    - Edge Cases: None; safe termination.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                              |
|-----------------------------------|---------------------|----------------------------------------|--------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger      | Manual workflow initiation              | None                           | Get 5 Minute Ago Timestamp             |                                                                                                        |
| Run Every 5 Minutes                | Schedule Trigger    | Scheduled workflow initiation every 5 min | None                           | Get 5 Minute Ago Timestamp             | ![n8n](https://uploads.n8n.io/templates/n8n.png) The `Schedule Trigger` node is configured to automatically execute the workflow every 5 minutes. This setup ensures consistent and timely monitoring for new incidents in ServiceNow without requiring manual input. The selected interval strikes a balance between responsiveness and efficient resource usage, making it ideal for real-time incident management workflows. |
| Get 5 Minute Ago Timestamp         | DateTime            | Calculate timestamp 5 minutes ago       | When clicking ‘Test workflow’, Run Every 5 Minutes | Get Incidents from ServiceNow          | ![Servicenow](https://uploads.n8n.io/templates/servicenow.png) This section begins with the `Get 5 Minute Ago Timestamp` node, which calculates a timestamp exactly 5 minutes prior to the current time. This timestamp is used as a reference point for querying incidents created within the last 5 minutes. The `Get Incidents from ServiceNow` node then fetches all incidents created after the calculated timestamp from the ServiceNow system, ensuring only the most recent incidents are retrieved. Finally, the `Check if New Incidents` node evaluates whether any incidents were returned by checking if the `sys_id` field exists in the response. This logic helps determine the next steps in the workflow, ensuring actions are taken only when new incidents are detected. |
| Get Incidents from ServiceNow      | ServiceNow          | Retrieve incidents created since timestamp | Get 5 Minute Ago Timestamp      | Check if New Incidents, Post Error Message if Error with ServiceNow | See above sticky note                                                                                   |
| Check if New Incidents             | If                  | Check if any new incidents exist        | Get Incidents from ServiceNow   | Sort Incidents in Ascending Order, No Incidents, Do Nothing | See above sticky note                                                                                   |
| Sort Incidents in Ascending Order  | Sort                | Sort incidents by incident number ascending | Check if New Incidents          | Post Incident Details to Slack Channel | ![Slack](https://uploads.n8n.io/templates/slack.png) This section begins with the `Sort Incidents in Ascending Order` node, which organizes the retrieved ServiceNow incidents by their incident number in ascending order. This ensures that incidents are processed and displayed in a logical sequence. The sorted incidents are then passed to the `Post Incident Details to Slack Channel` node, which formats and sends a detailed message to a designated Slack channel. The message includes key information such as the incident ID, description, severity, caller, priority, state, category, and the date the incident was opened. A "View Incident" button is also provided, linking directly to the ServiceNow record for quick access. This section ensures clear, organized communication of incident details, enabling efficient team collaboration and resolution. |
| Post Incident Details to Slack Channel | Slack               | Post detailed incident notification to Slack | Sort Incidents in Ascending Order | None                                  | See above sticky note                                                                                   |
| Post Error Message if Error with ServiceNow | Slack               | Post error alert to Slack on ServiceNow connection failure | Get Incidents from ServiceNow (error output) | None                                  | ![Slack](https://uploads.n8n.io/templates/slack.png) This section handles error reporting using the `Post Error Message if Error with ServiceNow` node. If the workflow encounters any issues connecting to ServiceNow, this node sends a predefined error message to a specified Slack channel. Usually this is triggered by expired credentials. The message alerts the team to investigate the issue in n8n, ensuring prompt attention and troubleshooting. By proactively notifying the team of connection errors, this section helps maintain the reliability of the workflow and minimizes disruptions in incident monitoring and reporting. |
| No Incidents, Do Nothing           | NoOp                | End workflow path if no new incidents    | Check if New Incidents (false branch) | None                                  | ![n8n](https://uploads.n8n.io/templates/n8n.png) If a ServiceNow system ID is not found in the ServiceNow node output, it will route to this node which effectively ends the process without doing anything. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters needed.

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Run Every 5 Minutes"  
   - Set interval to every 5 minutes.

3. **Create DateTime Node**  
   - Type: DateTime  
   - Name: "Get 5 Minute Ago Timestamp"  
   - Operation: Subtract from date  
   - Duration: 5  
   - Time Unit: minutes  
   - Magnitude: `={{ $now.toUTC() }}` (current UTC time)  
   - Output Field Name: `queryDate`

4. **Connect both triggers ("When clicking ‘Test workflow’" and "Run Every 5 Minutes") to "Get 5 Minute Ago Timestamp" node.**

5. **Create ServiceNow Node**  
   - Type: ServiceNow  
   - Name: "Get Incidents from ServiceNow"  
   - Resource: Incident  
   - Operation: Get All  
   - Query: `=sys_created_on>= {{ $json.queryDate }}` (dynamic expression referencing previous node output)  
   - Display Value: true  
   - Authentication: Basic Auth (configure with your ServiceNow API credentials)  
   - Enable "Continue on Error" to allow error output.  
   - Enable "Always Output Data" to ensure output even on error.

6. **Connect "Get 5 Minute Ago Timestamp" node to "Get Incidents from ServiceNow".**

7. **Create If Node**  
   - Type: If  
   - Name: "Check if New Incidents"  
   - Condition: Check if `sys_id` field exists in the JSON data (`={{ $json.sys_id }}` exists).  
   - Use strict type validation.

8. **Connect "Get Incidents from ServiceNow" main output to "Check if New Incidents" node.**

9. **Create Sort Node**  
   - Type: Sort  
   - Name: "Sort Incidents in Ascending Order"  
   - Sort Field: `number` ascending.

10. **Connect "Check if New Incidents" true branch to "Sort Incidents in Ascending Order".**

11. **Create Slack Node for Posting Incident Details**  
    - Type: Slack  
    - Name: "Post Incident Details to Slack Channel"  
    - Message Type: Block Kit  
    - Channel: Set to your Slack channel ID (e.g., `C086LRRQZQB`)  
    - Message Blocks: Use the following JSON structure with dynamic expressions referencing the "Get Incidents from ServiceNow" node data:

    ```json
    {
      "blocks": [
        {
          "type": "header",
          "text": {
            "type": "plain_text",
            "text": "ServiceNow Incident Notification",
            "emoji": true
          }
        },
        {
          "type": "section",
          "fields": [
            {
              "type": "mrkdwn",
              "text": "*Incident ID:*\n{{ $('Get Incidents from ServiceNow').item.json.number }}"
            },
            {
              "type": "mrkdwn",
              "text": "*Description:*\n{{ $('Get Incidents from ServiceNow').item.json.short_description }}"
            },
            {
              "type": "mrkdwn",
              "text": "*Severity:*\n{{ $('Get Incidents from ServiceNow').item.json.severity }}"
            },
            {
              "type": "mrkdwn",
              "text": "*Caller:*\n{{ $('Get Incidents from ServiceNow').item.json.caller_id.display_value }}"
            },
            {
              "type": "mrkdwn",
              "text": "*Priority:*\n{{ $('Get Incidents from ServiceNow').item.json.priority }}"
            },
            {
              "type": "mrkdwn",
              "text": "*State:*\n{{ $('Get Incidents from ServiceNow').item.json.incident_state }}"
            },
            {
              "type": "mrkdwn",
              "text": "*Category:*\n{{ $('Get Incidents from ServiceNow').item.json.category }}"
            },
            {
              "type": "mrkdwn",
              "text": "*Date Opened:*\n{{ $('Get Incidents from ServiceNow').item.json.opened_at }}"
            }
          ]
        },
        {
          "type": "actions",
          "elements": [
            {
              "type": "button",
              "text": {
                "type": "plain_text",
                "text": "View Incident",
                "emoji": true
              },
              "url": "https://dev206761.service-now.com/nav_to.do?uri=incident.do?sys_id={{ $('Get Incidents from ServiceNow').item.json.sys_id }}",
              "action_id": "view_incident"
            }
          ]
        }
      ]
    }
    ```

    - Credentials: Configure Slack API credentials with bot token having permission to post in the target channel.

12. **Connect "Sort Incidents in Ascending Order" node to "Post Incident Details to Slack Channel".**

13. **Create Slack Node for Error Handling**  
    - Type: Slack  
    - Name: "Post Error Message if Error with ServiceNow"  
    - Text: "🚨 Issue connecting to ServiceNow. Please investigate error in n8n. 🚨"  
    - Channel: Same Slack channel as above  
    - Markdown enabled  
    - Credentials: Same Slack API credentials.

14. **Connect the error output of "Get Incidents from ServiceNow" node to "Post Error Message if Error with ServiceNow".**

15. **Create NoOp Node**  
    - Type: NoOp  
    - Name: "No Incidents, Do Nothing"  
    - No parameters.

16. **Connect "Check if New Incidents" false branch to "No Incidents, Do Nothing".**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow is ideal for IT operations teams or system administrators using ServiceNow and Slack to streamline incident notifications and improve response times.                                                                                                                                                                                                | Workflow Description                                                                                     |
| Customize Slack message blocks to include additional incident details or modify formatting as needed.                                                                                                                                                                                                                                                               | Slack Block Kit documentation: https://api.slack.com/block-kit                                         |
| Adjust the ServiceNow query parameter `sysparm_query` to filter incidents by other criteria such as priority or state.                                                                                                                                                                                                                                             | ServiceNow REST API documentation: https://developer.servicenow.com/dev.do#!/reference/api/rome/rest/incident |
| Ensure ServiceNow Basic Auth credentials are valid and have appropriate permissions to query incidents.                                                                                                                                                                                                                                                             | ServiceNow API Authentication best practices                                                           |
| Slack Bot token must have permissions to post messages in the target channel.                                                                                                                                                                                                                                                                                        | Slack API OAuth scopes: https://api.slack.com/authentication/oauth-v2#scopes                            |
| The workflow uses UTC time for timestamp calculations to avoid timezone issues.                                                                                                                                                                                                                                                                                      |                                                                                                         |
| Error handling posts alerts to Slack to notify the team of connectivity issues, usually due to expired credentials.                                                                                                                                                                                                                                                 |                                                                                                         |
| The "View Incident" button URL is hardcoded with the ServiceNow instance domain `dev206761.service-now.com`. Update this URL to match your ServiceNow instance domain.                                                                                                                                                                                             |                                                                                                         |

---

This documentation provides a complete, structured reference to understand, reproduce, and customize the "ServiceNow Incident Notifications to Slack Workflow" in n8n. It covers all nodes, their configurations, data flow, error handling, and integration points with ServiceNow and Slack.