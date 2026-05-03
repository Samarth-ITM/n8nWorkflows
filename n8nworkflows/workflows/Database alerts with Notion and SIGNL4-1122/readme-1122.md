Database alerts with Notion and SIGNL4

https://n8nworkflows.xyz/workflows/database-alerts-with-notion-and-signl4-1122


# Database alerts with Notion and SIGNL4

### 1. Workflow Overview

This n8n workflow automates monitoring of machine status data stored in a Notion database and manages alerts in SIGNL4 accordingly. It simulates industrial machine monitoring by reading machine "Up" or "Down" status from a Notion table and sends alerts to a SIGNL4 team when a machine goes down. When the machine status changes back to "Up," the workflow automatically resolves the alert in SIGNL4. Additionally, status updates from SIGNL4 (acknowledgements, closures, annotations, escalations) are received via webhook and synchronized back to the Notion database.

The workflow is structured in the following logical blocks:

- **1.1 Database Polling and Alert Sending:** Periodically reads Notion database for machine status changes, sends new alerts to SIGNL4 for down machines, and updates Notion items as "read" to avoid duplicate alerts.
- **1.2 Alert Resolution:** Reads Notion items with machines marked "Up" and "read", then resolves corresponding alerts in SIGNL4 and updates Notion accordingly.
- **1.3 SIGNL4 Webhook Processing:** Receives SIGNL4 event notifications (new alerts, acknowledgements, closures, annotations, etc.), processes the event type, and updates the related Notion database page with the latest alert status.
- **1.4 Notion Trigger (optional, disabled):** A trigger node that could start alerting on new database pages added, currently disabled in this workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Database Polling and Alert Sending

- **Overview:**  
This block periodically polls the Notion database for machines marked as "down" (unchecked "Up" checkbox and not yet read) and sends alerts to SIGNL4 for these machines. After sending alerts, it updates the Notion items to mark them as read, preventing repeated alerts for the same down event.

- **Nodes Involved:**  
  - Interval  
  - Notion Read New  
  - SIGNL4 Alert 2  
  - Notion Update Read

- **Node Details:**

  - **Interval**  
    - Type: Interval Trigger  
    - Role: Triggers the workflow every 20 seconds to check for machine status updates.  
    - Configuration: Interval set to 20 seconds.  
    - Inputs: None (starting node)  
    - Outputs: Connects to Notion Read Open and Notion Read New nodes in parallel.  
    - Edge Cases: Long intervals might delay alerting; short intervals increase API calls and potential rate limits.

  - **Notion Read New**  
    - Type: Notion Database Read (Get All)  
    - Role: Reads all Notion pages where "Read" checkbox is unchecked and "Up" checkbox is unchecked (machine down and alert not sent yet).  
    - Configuration: Filter with AND condition for "Read" equals false and "Up" equals false in the database with ID `0f26823d-f509-43bb-b0e9-e9bb4ab91217`.  
    - Inputs: Triggered by Interval node.  
    - Outputs: Connects to SIGNL4 Alert 2 node.  
    - Edge Cases: Empty results mean no new down machines; errors may occur if Notion API rate limits are hit or invalid database ID.

  - **SIGNL4 Alert 2**  
    - Type: SIGNL4 API - Send Alert  
    - Role: Sends a new alert to SIGNL4 for each machine read from Notion as down.  
    - Configuration:  
      - Message: "Machine Alert: " plus machine name from Notion item.  
      - Title: "n8n Alert"  
      - ExternalId: Notion page ID (used for linking alert to Notion item).  
      - Location: Fixed latitude 52.3992137 and longitude 13.0583823 (can be customized).  
    - Inputs: Items from Notion Read New node.  
    - Outputs: Connects to Notion Update Read node.  
    - Edge Cases: SIGNL4 API authentication failures, network timeouts, or invalid externalId could cause failure.

  - **Notion Update Read**  
    - Type: Notion Database Page Update  
    - Role: Updates the Notion page to set the "Read" checkbox to true, indicating the alert was sent to SIGNL4.  
    - Configuration: Updates page with ID from Notion Read New node, setting "Read" checkbox to true.  
    - Inputs: Triggered by SIGNL4 Alert 2 completion.  
    - Outputs: None further.  
    - Edge Cases: Notion update permission issues, API errors could cause the read flag not to update, leading to duplicate alerts.

---

#### 2.2 Alert Resolution

- **Overview:**  
This block reads Notion entries for machines that are currently "Up" (checkbox checked) and already marked as "Read" (alert sent previously), then resolves the corresponding alerts in SIGNL4 and updates Notion accordingly.

- **Nodes Involved:**  
  - Notion Read Open  
  - SIGNL4 Resolve  
  - Notion Update Final

- **Node Details:**

  - **Notion Read Open**  
    - Type: Notion Database Read (Get All)  
    - Role: Retrieves Notion pages where "Up" checkbox is true and "Read" checkbox is true (machine is running, previously alerted as down).  
    - Configuration: Filter with AND condition for "Up" equals true and "Read" equals true.  
    - Inputs: Triggered by Interval node.  
    - Outputs: Connects to SIGNL4 Resolve node.  
    - Edge Cases: No matching pages means no alerts to resolve; API failures possible.

  - **SIGNL4 Resolve**  
    - Type: SIGNL4 API - Resolve Alert  
    - Role: Resolves the SIGNL4 alert linked to the Notion page (using page ID as externalId).  
    - Configuration: ExternalId set from Notion Read Open node page ID.  
    - Inputs: Notion Read Open output.  
    - Outputs: Connects to Notion Update Final node.  
    - Edge Cases: Resolve failure if alert does not exist, expired, or API error.

  - **Notion Update Final**  
    - Type: Notion Database Page Update  
    - Role: Updates Notion page properties after alert resolution; clears or resets relevant flags (checkbox "Read" unchecked).  
    - Configuration: Updates page by ID from Notion Read Open node; clears "Read" checkbox.  
    - Inputs: Triggered by SIGNL4 Resolve node.  
    - Outputs: None further.  
    - Edge Cases: Update failure may cause desynchronization between Notion and SIGNL4.

---

#### 2.3 SIGNL4 Webhook Processing

- **Overview:**  
This block handles incoming webhook POST requests from SIGNL4 about alert status changes. It parses the event type and status code, derives a human-readable status string, and updates the corresponding Notion page with this status. This keeps Notion in sync with SIGNL4 alert lifecycle changes.

- **Nodes Involved:**  
  - Webhook  
  - Function (parse SIGNL4 status)  
  - Notion Update

- **Node Details:**

  - **Webhook**  
    - Type: HTTP Webhook (POST)  
    - Role: Receives real-time event notifications from SIGNL4 (alerts created, acknowledged, closed, annotated, etc.).  
    - Configuration: Path `95fd62c7-fc8c-4f6f-8441-bbf85a2da81a`, method POST, no additional options.  
    - Inputs: External HTTP calls from SIGNL4.  
    - Outputs: Connects to Function node.  
    - Edge Cases: Webhook must be publicly accessible; failure to receive events causes sync loss.

  - **Function (parse SIGNL4 status)**  
    - Type: Function (JavaScript code)  
    - Role: Interprets SIGNL4 webhook payload, maps alert and event status codes to descriptive strings (e.g., "Acknowledged", "Closed", "New Alert", "No one on duty", "Annotated"), appends username and annotation if present, and sets a boolean `s4Up` flag indicating if alert is closed (machine up).  
    - Key logic:  
      - statusCode + eventType combinations determine alert state.  
      - Annotation messages appended if present.  
      - Username extracted or defaulted to "System".  
    - Inputs: JSON body from Webhook node.  
    - Outputs: Adds `s4Status` and `s4Up` fields to JSON.  
    - Edge Cases: Payload missing expected fields may cause errors; unrecognized status codes default to "Status".

  - **Notion Update**  
    - Type: Notion Database Page Update  
    - Role: Updates the Notion page identified by the alert's externalEventId with the new status string (`s4Status`) from the Function node.  
    - Configuration: Updates "Description" rich_text field with the status and annotation string.  
    - Inputs: Output from Function node.  
    - Outputs: None further.  
    - Edge Cases: Update failure may cause Notion to be out of sync with SIGNL4.

---

#### 2.4 Notion Trigger (Disabled)

- **Overview:**  
This node is configured to trigger the workflow when a new page is added to the monitored Notion database, but it is disabled and not currently active in the workflow.

- **Nodes Involved:**  
  - Notion Trigger

- **Node Details:**

  - **Notion Trigger**  
    - Type: Notion Trigger  
    - Role: Watches for new pages added to the specified Notion database and triggers workflow on such events.  
    - Configuration: Database ID specified, polling every 1 minute.  
    - Status: Disabled.  
    - Edge Cases: If enabled, could trigger alert sending on new machine entries.

---

### 3. Summary Table

| Node Name         | Node Type                 | Functional Role                            | Input Node(s)           | Output Node(s)        | Sticky Note                             |
|-------------------|---------------------------|-------------------------------------------|------------------------|-----------------------|----------------------------------------|
| Interval          | Interval Trigger          | Periodically triggers polling              | -                      | Notion Read Open, Notion Read New |                                        |
| Notion Read New   | Notion Database Read      | Reads machines down and not read           | Interval               | SIGNL4 Alert 2         |                                        |
| SIGNL4 Alert 2     | SIGNL4 API (Send Alert)  | Sends alert to SIGNL4 for down machines    | Notion Read New        | Notion Update Read     |                                        |
| Notion Update Read | Notion Database Update    | Marks Notion items as read after alert sent| SIGNL4 Alert 2         | -                     |                                        |
| Notion Read Open   | Notion Database Read      | Reads machines up and already alerted      | Interval               | SIGNL4 Resolve         |                                        |
| SIGNL4 Resolve     | SIGNL4 API (Resolve Alert)| Resolves alerts in SIGNL4 for machines up  | Notion Read Open       | Notion Update Final    |                                        |
| Notion Update Final| Notion Database Update    | Resets "Read" checkbox after alert resolved| SIGNL4 Resolve         | -                     |                                        |
| Webhook            | HTTP Webhook (POST)       | Receives SIGNL4 alert status updates       | External HTTP calls    | Function               |                                        |
| Function (parse SIGNL4 status) | Function (JS)          | Parses SIGNL4 webhook payload and generates status text | Webhook                | Notion Update          |                                        |
| Notion Update      | Notion Database Update    | Updates Notion page with alert status text | Function                | -                     |                                        |
| Notion Trigger     | Notion Trigger (disabled) | Triggers on new Notion database pages (disabled) | -                      | SIGNL4 Alert           |                                        |
| SIGNL4 Alert       | SIGNL4 API (Send Alert)  | Sends alert on new Notion page (disabled trigger) | Notion Trigger         | -                     |                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Interval Trigger Node:**  
   - Name: `Interval`  
   - Type: Interval  
   - Parameters: Set interval to 20 seconds.  
   - No credentials needed.

2. **Add Notion Credentials:**  
   - Configure Notion API credentials with access to your target database.

3. **Add Notion Read New Node:**  
   - Name: `Notion Read New`  
   - Type: Notion (Get All)  
   - Parameters:  
     - Resource: databasePage  
     - Operation: getAll  
     - Database ID: your Notion database ID (e.g., `0f26823d-f509-43bb-b0e9-e9bb4ab91217`)  
     - Filter: AND condition where "Read" checkbox is false and "Up" checkbox is false.  
   - Connect input from `Interval`.

4. **Add SIGNL4 Credentials:**  
   - Configure SIGNL4 API credentials for alert sending and resolution.

5. **Add SIGNL4 Alert 2 Node:**  
   - Name: `SIGNL4 Alert 2`  
   - Type: SIGNL4 (Send Alert)  
   - Parameters:  
     - Message: `Machine Alert: {{$node["Notion Read New"].json["Name"]}}`  
     - Title: `n8n Alert`  
     - ExternalId: `{{$node["Notion Read New"].json["id"]}}`  
     - Location: Set latitude: 52.3992137, longitude: 13.0583823 (customize as needed)  
   - Connect input from `Notion Read New`.

6. **Add Notion Update Read Node:**  
   - Name: `Notion Update Read`  
   - Type: Notion (Update)  
   - Parameters:  
     - Page ID: `{{$node["Notion Read New"].json["id"]}}`  
     - Update properties: Set "Read" checkbox to true.  
   - Connect input from `SIGNL4 Alert 2`.

7. **Add Notion Read Open Node:**  
   - Name: `Notion Read Open`  
   - Type: Notion (Get All)  
   - Parameters:  
     - Database ID: same as above  
     - Filter: AND condition where "Up" checkbox is true and "Read" checkbox is true.  
   - Connect input from `Interval` (parallel to Notion Read New).

8. **Add SIGNL4 Resolve Node:**  
   - Name: `SIGNL4 Resolve`  
   - Type: SIGNL4 (Resolve Alert)  
   - Parameters:  
     - ExternalId: `{{$node["Notion Read Open"].json["id"]}}`  
   - Connect input from `Notion Read Open`.

9. **Add Notion Update Final Node:**  
   - Name: `Notion Update Final`  
   - Type: Notion (Update)  
   - Parameters:  
     - Page ID: `{{$node["Notion Read Open"].json["id"]}}`  
     - Update properties: Clear "Read" checkbox (uncheck).  
   - Connect input from `SIGNL4 Resolve`.

10. **Create Webhook Node:**  
    - Name: `Webhook`  
    - Type: HTTP Webhook (POST)  
    - Parameters:  
      - Path: a unique string (e.g., `95fd62c7-fc8c-4f6f-8441-bbf85a2da81a`)  
      - HTTP Method: POST  
    - No input connections (starting node).

11. **Add Function Node to Parse SIGNL4 Status:**  
    - Name: `Function`  
    - Type: Function (JavaScript)  
    - Parameters: Paste the logic that:  
      - Reads `alert.statusCode` and `eventType` from webhook JSON body,  
      - Maps to status strings ("Acknowledged", "Closed", "New Alert", "No one on duty", "Annotated"),  
      - Extracts annotation messages and username,  
      - Sets `s4Status` and boolean `s4Up` (true if closed).  
    - Connect input from `Webhook`.

12. **Add Notion Update Node to Sync Status:**  
    - Name: `Notion Update`  
    - Type: Notion (Update)  
    - Parameters:  
      - Page ID: `{{$node["Webhook"].json["body"]["alert"]["externalEventId"]}}`  
      - Update properties: Set "Description" rich_text to `{{$node["Function"].json["s4Status"]}}`  
    - Connect input from `Function`.

13. **(Optional) Add Notion Trigger Node:**  
    - Name: `Notion Trigger`  
    - Type: Notion Trigger  
    - Parameters:  
      - Database ID: your Notion database ID  
      - Event: pageAddedToDatabase  
      - Poll interval: 1 minute  
    - Connect output to `SIGNL4 Alert` (if you want to trigger alerts on new pages).

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow simulates machine data using a Notion table with an "Up" checkbox to mark machine state.| Setup description in the workflow overview.     |
| SIGNL4 alert status codes and event types are used to determine alert lifecycle states in webhook. | Refer to SIGNL4 API documentation for codes.    |
| The workflow uses the Notion page ID as the externalId in SIGNL4 to correlate alerts and database items. | Enables two-way synchronization.                 |
| Alerts include fixed geolocation coordinates (latitude: 52.3992137, longitude: 13.0583823).          | Can be customized to actual machine locations.  |
| Status updates from SIGNL4 are automatically reflected in Notion's "Description" rich_text field.    | Ensures Notion reflects real-time alert states. |
| The disabled Notion Trigger node can be enabled to start alerting on new machine entries if desired.| Disabled by default to avoid duplicate alerts.  |
| SIGNL4 webhook requires exposing the n8n webhook URL publicly and configuring SIGNL4 to send events.| Must ensure correct webhook URL and security.   |
| More info on n8n nodes used: https://docs.n8n.io/nodes/                                              | Official n8n documentation site.                 |
| SIGNL4 official website: https://www.signl4.com/                                                    | For alert system reference and API details.     |

---

This documentation fully describes the workflow’s logic, node configurations, dependencies, and operational behavior to enable effective understanding, reproduction, and troubleshooting.