Host your own Uptime Monitoring with Scheduled Triggers

https://n8nworkflows.xyz/workflows/host-your-own-uptime-monitoring-with-scheduled-triggers-2327


# Host your own Uptime Monitoring with Scheduled Triggers

### 1. Workflow Overview

This workflow implements a lightweight uptime monitoring service for a list of websites, targeting webmasters who want a simple, cost-effective solution without excessive complexity. It periodically checks specified website URLs and sends alerts when any site is down. Additionally, it logs website status information in a Google Sheet to track uptime trends over time.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger & Input Retrieval**: Periodically initiates the workflow and fetches the list of websites to monitor from a Google Sheet.
- **1.2 Site Testing**: Iterates over each website URL and performs an HTTP request to determine if the site is up or down.
- **1.3 Status Calculation**: Compares the current HTTP response status with the previous recorded state to detect status changes.
- **1.4 Routing & Notifications**: Routes the flow based on status changes to send email and Slack alerts if a site is down or if the status changed.
- **1.5 Logging & State Update**: Appends uptime event logs to a dedicated Google Sheet and updates the main site status back in the dashboard sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger & Input Retrieval

- **Overview:**  
  Initiates the workflow on a schedule (every 6 hours) and retrieves the list of websites to monitor from a Google Sheet named "dashboard". This block sets the cadence of uptime checks and provides the input data.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Sites

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every 6 hours.  
    - Configuration: Interval set to 6 hours.  
    - Inputs: None  
    - Outputs: Triggers "Get Sites" node.  
    - Edge Cases: Timezone issues may affect expected trigger time; misconfiguration can cause too frequent or infrequent checks.

  - **Get Sites**  
    - Type: Google Sheets  
    - Role: Reads website list from Google Sheet "dashboard" (gid=0).  
    - Configuration: Uses Google Sheets OAuth2 credentials; reads entire sheet data.  
    - Expressions: Document ID and sheet details are statically set pointing to the URL of the monitoring Google Sheet.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Passes site list to "For Each Site..." node.  
    - Possible Failures: Google API quota limits, OAuth token expiration, incorrect Sheet ID or access permissions.

---

#### 1.2 Site Testing

- **Overview:**  
  Processes each site URL individually by performing an HTTP GET request to determine the site's status code, which indicates if the site is UP or DOWN.

- **Nodes Involved:**  
  - For Each Site... (SplitInBatches)  
  - Perform Site Test

- **Node Details:**

  - **For Each Site...**  
    - Type: SplitInBatches  
    - Role: Iterates over each website retrieved from the Google Sheet, processing them one by one (batch size defaults to 1).  
    - Inputs: List of sites from "Get Sites".  
    - Outputs: Sends each site data to "Perform Site Test".  
    - Edge Cases: Large site lists may cause slow processing; failure in one batch does not stop others.

  - **Perform Site Test**  
    - Type: HTTP Request  
    - Role: Sends an HTTP GET request to the URL in the "Property" field.  
    - Configuration:  
      - URL dynamically set from the site's "Property" (website URL).  
      - Response configured to never error and to return full HTTP response including headers.  
    - Inputs: Single site URL from "For Each Site...".  
    - Outputs: HTTP response passed to "Calculate Status".  
    - Edge Cases: Network timeouts, invalid URLs, SSL errors, server errors, redirects.  
    - Notes: Using "neverError" ensures workflow continues even if site is down.

---

#### 1.3 Status Calculation

- **Overview:**  
  Determines the current site status (UP or DOWN) based on HTTP response code and compares it to the previous status to detect status changes and various state transitions.

- **Nodes Involved:**  
  - Calculate Status

- **Node Details:**

  - **Calculate Status**  
    - Type: Set  
    - Role: Creates new JSON fields to interpret HTTP status and previous status conditions.  
    - Configuration:  
      - Extracts the Date header from HTTP response for timestamp.  
      - Copies "Property" (site URL).  
      - Defines boolean flags for transitions:  
        - UP_FROM_UP: site was UP and still UP (statusCode < 400 and previous Status = 'UP')  
        - DOWN_FROM_DOWN: site was DOWN and still DOWN (statusCode >= 400 and Status = 'DOWN')  
        - UP_FROM_DOWN: site was DOWN but now UP (statusCode < 400 and Status = 'DOWN')  
        - DOWN_FROM_UP: site was UP but now DOWN (statusCode >= 400 and Status = 'UP')  
    - Inputs: HTTP response data and previous status.  
    - Outputs: Flags and data passed to "Status Router".  
    - Edge Cases: Missing or malformed HTTP headers; undefined previous status field.

---

#### 1.4 Routing & Notifications

- **Overview:**  
  Routes the workflow based on site status changes. Sends alerts via email and Slack if the site is down or if a status change is detected. Also forwards data to log nodes.

- **Nodes Involved:**  
  - Status Router (Switch)  
  - Send Email Alert  
  - Send Chat Alert

- **Node Details:**

  - **Status Router**  
    - Type: Switch  
    - Role: Routes based on boolean flags from "Calculate Status" indicating the type of status transition.  
    - Configuration: Four outputs corresponding to:  
      - UP_FROM_UP  
      - UP_FROM_DOWN  
      - DOWN_FROM_DOWN  
      - DOWN_FROM_UP  
    - Inputs: Flags from "Calculate Status".  
    - Outputs: Routes to alert and logging nodes accordingly.  
    - Edge Cases: If flags are missing or not boolean, routing may fail or misroute.

  - **Send Email Alert**  
    - Type: Gmail  
    - Role: Sends an email alert when a site is down or status changes to down.  
    - Configuration:  
      - Recipient: no-reply@example.com (placeholder, should be replaced).  
      - Subject and body dynamically include site URL and status.  
      - Uses Gmail OAuth2 credentials.  
      - Plain text email without attribution.  
    - Inputs: Routed from "Status Router" on DOWN states.  
    - Outputs: Triggers "Send Chat Alert".  
    - Edge Cases: Gmail API limits, invalid credentials, unreachable SMTP server.

  - **Send Chat Alert**  
    - Type: Slack  
    - Role: Sends Slack message to a channel when a site is down or status changes to down.  
    - Configuration:  
      - Channel: "general" channel ID configured via Slack API credentials.  
      - Message includes site URL, date, and status.  
    - Inputs: Triggered after email alert node.  
    - Outputs: None.  
    - Edge Cases: Slack API rate limits or permission errors.

---

#### 1.5 Logging & State Update

- **Overview:**  
  Logs each uptime event into a dedicated Google Sheet per site and updates the main Google Sheet dashboard with the current status of each site.

- **Nodes Involved:**  
  - Log Uptime Event  
  - Update Site Status

- **Node Details:**

  - **Log Uptime Event**  
    - Type: Google Sheets  
    - Role: Appends a new row to a Google Sheet named after the property (site URL) with detailed event information.  
    - Configuration:  
      - Columns: date, period (YYYY-MM), Property, and all status transition booleans.  
      - Operation: Append new row.  
      - Uses Google Sheets OAuth2 credentials.  
    - Inputs: Routed from "Status Router" for all cases.  
    - Outputs: Triggers "Update Site Status".  
    - Edge Cases: Google API quota, sheet access permissions, invalid sheet name if Property is malformed.

  - **Update Site Status**  
    - Type: Google Sheets  
    - Role: Updates or appends the current status of the site in the main dashboard sheet.  
    - Configuration:  
      - Matches on "Property" column to update row.  
      - Updates "Status" column to "UP" or "DOWN" based on boolean flags.  
      - Operation: AppendOrUpdate.  
    - Inputs: From "Log Uptime Event".  
    - Outputs: Loops back to "For Each Site..." to process next site.  
    - Edge Cases: Race conditions if multiple updates happen concurrently, Google Sheets API limits.

---

### 3. Summary Table

| Node Name          | Node Type        | Functional Role                           | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                                |
|--------------------|------------------|-----------------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger | Initiates workflow every 6 hours        | None                  | Get Sites               | ## 1. Setting a Schedule\n[Read more about Scheduling Workflows](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/)\n\nSince we expect downtime to be a rare occurance, our monitor should only check infrequently during the day. We'll use a schedule trigger for this purpose.\n\nOnce the schdule activates, we'll pull a list of sites to check from our google sheet. |
| Get Sites          | Google Sheets    | Retrieves list of websites               | Schedule Trigger       | For Each Site...         | ### 🚨Google Sheet Required!\nYou'll need the following columns:\n* **Property** - the website address to monitor\n* **Status** - either one of "UP" or "DOWN"                 |
| For Each Site...    | SplitInBatches   | Iterates over each site URL              | Get Sites              | Perform Site Test        |                                                                                                                            |
| Perform Site Test   | HTTP Request     | Performs HTTP GET to check site status  | For Each Site...       | Calculate Status         | ## 2. Perform Site Checks\n[Read more about using HTTP requests](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)\n\nn8n makes it easy to communicate with external websites by offering a powerful HTTP request node which can handle GET and POST requests as well as pagination.\n\nHere, we're only interested in the status code of our requests. |
| Calculate Status    | Set              | Calculates status flags and timestamps  | Perform Site Test      | Status Router            |                                                                                                                            |
| Status Router      | Switch           | Routes based on status transitions       | Calculate Status       | Log Uptime Event, Send Email Alert | ## 3. Sending Alerts and Logging Results\n[Read more about using Switch for powerful control flow](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch)\n\nThe switch node is powerful control flow tool that makes your workflows smart. Here, we're able to use Switch to trigger alert notifications whenever we have DOWN status or whenever we get a status change.\n\nWe store the event in our Sites Google Sheet and update the site's status which will be used to calculate our state on the next scheduled run. |
| Send Email Alert    | Gmail            | Sends email alert on DOWN status         | Status Router          | Send Chat Alert          |                                                                                                                            |
| Send Chat Alert     | Slack            | Sends Slack alert on DOWN status         | Send Email Alert       | None                    |                                                                                                                            |
| Log Uptime Event    | Google Sheets    | Logs uptime event data per site          | Status Router          | Update Site Status       |                                                                                                                            |
| Update Site Status  | Google Sheets    | Updates main dashboard sheet status      | Log Uptime Event       | For Each Site...         |                                                                                                                            |
| Sticky Note3        | Sticky Note      | Explanation of scheduling block          | None                  | None                    | ## 1. Setting a Schedule\n[Read more about Scheduling Workflows](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/)\n\nSince we expect downtime to be a rare occurance, our monitor should only check infrequently during the day. We'll use a schedule trigger for this purpose.\n\nOnce the schdule activates, we'll pull a list of sites to check from our google sheet. |
| Sticky Note4        | Sticky Note      | Explanation of HTTP requests block       | None                  | None                    | ## 2. Perform Site Checks\n[Read more about using HTTP requests](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)\n\nn8n makes it easy to communicate with external websites by offering a powerful HTTP request node which can handle GET and POST requests as well as pagination.\n\nHere, we're only interested in the status code of our requests. |
| Sticky Note5        | Sticky Note      | Explanation of alerts and logging block  | None                  | None                    | ## 3. Sending Alerts and Logging Results\n[Read more about using Switch for powerful control flow](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.switch)\n\nThe switch node is powerful control flow tool that makes your workflows smart. Here, we're able to use Switch to trigger alert notifications whenever we have DOWN status or whenever we get a status change.\n\nWe store the event in our Sites Google Sheet and update the site's status which will be used to calculate our state on the next scheduled run. |
| Sticky Note6        | Sticky Note      | General encouragement and support links  | None                  | None                    | ## Try It Out!\n### Thie workflow showcases how you can build a simple website monitoring service using Scheduled Triggers and the HTTP Requests node.\n\n### Need Help?\nJoin the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)!\n\nHappy Hacking! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 6 hours.  
   - No credentials needed.

2. **Create Google Sheets Node "Get Sites"**  
   - Type: Google Sheets  
   - Operation: Read rows from sheet "dashboard" (gid=0).  
   - Set Document ID to your Google Sheet ID containing site list.  
   - Use Google Sheets OAuth2 credentials.  
   - Connect Schedule Trigger output to this node.

3. **Create SplitInBatches Node "For Each Site..."**  
   - Type: SplitInBatches  
   - Batch size defaults to 1 (process one site at a time).  
   - Connect output of "Get Sites" to this node.

4. **Create HTTP Request Node "Perform Site Test"**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: Set via expression from current item property: `{{$json["Property"]}}`  
   - Enable "Never Error" to avoid workflow failure on request errors.  
   - Enable "Full Response" to get headers and status code.  
   - Connect "For Each Site..." second output to this node.

5. **Create Set Node "Calculate Status"**  
   - Type: Set  
   - Add fields:  
     - `date` = `{{$json.headers.date}}`  
     - `Property` = `{{$json.Property}}`  
     - `UP_FROM_UP` = `{{$json.statusCode < 400 && $json.Status === 'UP'}}` (boolean)  
     - `DOWN_FROM_DOWN` = `{{$json.statusCode >= 400 && $json.Status === 'DOWN'}}` (boolean)  
     - `UP_FROM_DOWN` = `{{$json.statusCode < 400 && $json.Status === 'DOWN'}}` (boolean)  
     - `DOWN_FROM_UP` = `{{$json.statusCode >= 400 && $json.Status === 'UP'}}` (boolean)  
   - Connect "Perform Site Test" to this node.

6. **Create Switch Node "Status Router"**  
   - Type: Switch  
   - Add four rules:  
     - Output "UP_FROM_UP": condition `{{$json.UP_FROM_UP}} === true`  
     - Output "UP_FROM_DOWN": condition `{{$json.UP_FROM_DOWN}} === true`  
     - Output "DOWN_FROM_DOWN": condition `{{$json.DOWN_FROM_DOWN}} === true`  
     - Output "DOWN_FROM_UP": condition `{{$json.DOWN_FROM_UP}} === true`  
   - Connect "Calculate Status" output to this node.

7. **Create Gmail Node "Send Email Alert"**  
   - Type: Gmail  
   - Set "Send To" to your alert email address (e.g., your admin email).  
   - Subject: `n8n uptime: {{$json.Property}} is {{$json.DOWN_FROM_UP ? 'DOWN' : 'UP'}}`  
   - Message body:  
     ```
     From: n8n uptime
     Date: {{$json.date}}
     
     {{$json.Property}} is {{$json.DOWN_FROM_UP ? 'DOWN' : 'UP'}}
     ```  
   - Use Gmail OAuth2 credentials.  
   - Connect "Status Router" outputs for DOWN_FROM_UP, DOWN_FROM_DOWN, and UP_FROM_DOWN to this node (as per original routing).

8. **Create Slack Node "Send Chat Alert"**  
   - Type: Slack  
   - Channel: Set to your Slack channel ID (e.g., "general").  
   - Message text:  
     ```
     From: n8n uptime
     Date: {{$json.date}}
     
     {{$json.Property}} is {{$json.DOWN_FROM_UP ? 'DOWN' : 'UP'}}
     ```  
   - Use Slack API credentials.  
   - Connect "Send Email Alert" output to this node.

9. **Create Google Sheets Node "Log Uptime Event"**  
   - Type: Google Sheets  
   - Operation: Append row.  
   - Document ID: same Google Sheet as "Get Sites".  
   - Sheet Name: dynamic, set via expression `{{$json.Property}}` (each site has its own sheet).  
   - Columns to map:  
     - date: `{{$json.date}}`  
     - period: `{{ new Date($json.date).toISOString().slice(0,7) }}` (format YYYY-MM)  
     - Property, UP_FROM_UP, DOWN_FROM_DOWN, UP_FROM_DOWN, DOWN_FROM_UP from current JSON.  
   - Use Google Sheets OAuth2 credentials.  
   - Connect all "Status Router" outputs (including those routed to email alerts) to this node to ensure logging for all cases.

10. **Create Google Sheets Node "Update Site Status"**  
    - Type: Google Sheets  
    - Operation: AppendOrUpdate row (match on "Property" column).  
    - Document ID: same as above.  
    - Sheet Name: "dashboard" (gid=0).  
    - Columns:  
      - Property: `{{$json.Property}}`  
      - Status: `{{$json.DOWN_FROM_UP || $json.DOWN_FROM_DOWN ? 'DOWN' : 'UP'}}`  
    - Use Google Sheets OAuth2 credentials.  
    - Connect "Log Uptime Event" output to this node.

11. **Loop Back**  
    - Connect "Update Site Status" output back to "For Each Site..." to continue processing remaining sites.

12. **Optional: Add Sticky Note Nodes**  
    - Add explanatory sticky notes near each logical block for documentation and clarity.  
    - Include relevant links and instructions as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Workflow uses Google Sheets to store site URLs and their status; columns required are "Property" (URL) and "Status" (UP/DOWN). | Google Sheets configuration                                                                                                   |
| Scheduling interval is set to 6 hours to reduce unnecessary checks, assuming downtime is rare.                             | Schedule Trigger setup                                                                                                        |
| HTTP Request node uses "Never Error" to ensure workflow continues even if a site is unreachable.                            | Node configuration detail                                                                                                     |
| Alerts are sent to both email (via Gmail) and Slack to ensure timely notifications.                                         | Integration details                                                                                                           |
| For customization, Google Sheets nodes can be replaced with Excel or Airtable nodes as needed.                             | Customization hint                                                                                                            |
| Join the n8n community for support: [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)   | Support resources                                                                                                            |
| Documentation for key nodes: Schedule Trigger, HTTP Request, Switch node can be found at n8n official docs linked in sticky notes. | Node documentation references                                                                                                |

---

This completes the full structured reference for the "Host your own Uptime Monitoring with Scheduled Triggers" workflow. It enables advanced users and AI agents to understand, reproduce, and modify the workflow confidently.