Meraki Packet Loss and Latency Alerts to Microsoft Teams

https://n8nworkflows.xyz/workflows/workflow-2054-1747220151557.png


# Meraki Packet Loss and Latency Alerts to Microsoft Teams

### 1. Workflow Overview

This workflow automates monitoring of uplink packet loss and latency metrics from Cisco Meraki networks and sends alerts to a Microsoft Teams dispatch channel when thresholds are exceeded. It is designed for network operations or dispatch teams to get timely notifications about network performance issues, specifically for uplinks managed via Meraki Dashboard.

The workflow is logically divided into three main blocks:

- **1.1 Data Retrieval from Meraki API**: Periodically fetches organizations, networks, and uplink stats (loss and latency) from Meraki using API calls authenticated via API key headers.

- **1.2 Data Processing and Filtering**: Organizes and combines the retrieved data, calculates averages over recent time intervals, and filters for networks exceeding defined latency or loss thresholds.

- **1.3 Alerting and State Management**: Checks if alerts for problematic networks have already been sent (using Redis), prevents duplicate alerts, sends new alert messages to Microsoft Teams, and logs active alerts with a TTL to enable repeated notifications if issues persist.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval from Meraki API

- **Overview:**  
  This block fetches all necessary raw data from the Meraki API: organizations the user has access to, all networks per organization, and uplink loss and latency statistics, preparing the foundation for further processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - Get Meraki Organizations  
  - Get Org Name & ID  
  - Get Network IDs  
  - Sets Network Variables  
  - Get Uplink Loss and Latency

- **Node Details:**

  - **Schedule Trigger**  
    - *Type*: Schedule trigger  
    - *Role*: Runs the workflow automatically every 5 minutes from 8am to 5pm, Monday through Friday.  
    - *Configuration*: Cron expression `*/5 8-16 * * 1-5`.  
    - *Connections*: Starts the workflow by triggering "Get Meraki Organizations".  
    - *Edge Cases*: Cron misconfiguration could cause missed or extra runs.

  - **When clicking "Execute Workflow" (Manual Trigger)**  
    - *Type*: Manual trigger  
    - *Role*: Allows manual start for testing or immediate execution.  
    - *Connections*: Also triggers "Get Meraki Organizations".  
    - *Edge Cases*: No input parameters; useful only for manual runs.

  - **Get Meraki Organizations**  
    - *Type*: HTTP Request  
    - *Role*: Calls Meraki API endpoint `/organizations` to retrieve all organizations.  
    - *Configuration*: Uses API key via HTTP header authentication; headers include `Authorization` (API key) and `Accept: application/json`.  
    - *Key expressions*: None, static URL.  
    - *Inputs/Outputs*: Input from triggers; outputs raw organization data JSON.  
    - *Notes*: Requires valid Meraki API key credential.  
    - *Edge Cases*: API key invalid or expired; network errors; API rate limits.  
    - *Sticky Note Content*: Explains API key generation and header usage, with documentation link: https://documentation.meraki.com/General_Administration/Other_Topics/Cisco_Meraki_Dashboard_API

  - **Get Org Name & ID**  
    - *Type*: Set node  
    - *Role*: Simplifies and renames organization data fields (`name` → `CompanyName`, `id` → `OrgID`).  
    - *Configuration*: Extracts and renames fields for clarity.  
    - *Inputs/Outputs*: Input from "Get Meraki Organizations"; output used to get networks for each org.

  - **Get Network IDs**  
    - *Type*: HTTP Request  
    - *Role*: For each organization ID, calls `/organizations/{OrgID}/networks` to get all networks.  
    - *Configuration*: URL uses expression to insert OrgID dynamically. Same header authentication as above.  
    - *Edge Cases*: API rate limits; org with no networks; invalid OrgID.

  - **Sets Network Variables**  
    - *Type*: Set node  
    - *Role*: Simplifies network data fields (`id` → `NetworkID`, `name` → `NetworkName`, `url` → `networkURL`).  
    - *Inputs/Outputs*: Input from "Get Network IDs"; outputs refined network info for merging.

  - **Get Uplink Loss and Latency**  
    - *Type*: HTTP Request  
    - *Role*: For each organization ID, calls `/organizations/{OrgID}/devices/uplinksLossAndLatency?timespan=300` to get uplink stats over the last 5 minutes.  
    - *Configuration*: URL uses OrgID expression; headers as above.  
    - *Edge Cases*: API limits; missing devices; no uplink data.

---

#### 2.2 Data Processing and Filtering

- **Overview:**  
  This block merges network info with uplink metrics, restructures data for easier filtering, computes average packet loss and latency over five timestamps, and filters only networks exceeding threshold values.

- **Nodes Involved:**  
  - Combine latency to its respective Network (Merge)  
  - Makes Latency and Loss Filterable (Set)  
  - Average Latency & Loss over 5m (Code)  
  - Filters Problematic sites (Code)

- **Node Details:**

  - **Combine latency to its respective Network**  
    - *Type*: Merge node  
    - *Role*: Joins network data with uplink latency/loss data by matching `NetworkID` (from networks) and `networkId` (from uplink stats).  
    - *Configuration*: Mode: Combine, join mode: enrichInput1, merge by `NetworkID` and `networkId`.  
    - *Inputs*:  
      - Input 1: network info (from Sets Network Variables)  
      - Input 2: uplink stats (from Get Uplink Loss and Latency)  
    - *Outputs*: Combined dataset with network details and uplink metrics.

  - **Makes Latency and Loss Filterable**  
    - *Type*: Set node  
    - *Role*: Extracts and renames uplink timeseries latency and loss fields (`timeSeries[0..4].lossPercent` and `.latencyMs`) into separate fields per timestamp (`TS0-Loss`, `TS0-Latency`, etc.) for easier processing.  
    - *Inputs*: Combined data from previous Merge.  
    - *Outputs*: Data with flattened latency and loss fields.

  - **Average Latency & Loss over 5m**  
    - *Type*: Code (JavaScript)  
    - *Role*: Calculates average packet loss and latency over the five timestamps for each network entry.  
    - *Logic*: Sums values of `TS0-Loss` through `TS4-Loss` and divides by 5 for average loss; same for latency. Adds `AverageLoss` and `AverageLatency` fields to each item.  
    - *Input*: Filterable latency and loss data.  
    - *Output*: Items enriched with average metrics.  
    - *Edge Cases*: Missing or malformed data could cause NaN results.

  - **Filters Problematic sites**  
    - *Type*: Code (JavaScript)  
    - *Role*: Filters the dataset to pass only networks where average latency > 300ms or average loss > 2%.  
    - *Input*: Items with average metrics.  
    - *Output*: Only problematic sites forwarded.  
    - *Edge Cases*: Networks with missing average fields will be excluded.

  - *Sticky Notes*:  
    - Explains how merging enriches network info with stats.  
    - Describes calculation of averages and filtering based on thresholds.

---

#### 2.3 Alerting and State Management

- **Overview:**  
  This block manages alert state to avoid duplicate notifications, sends alerts to Microsoft Teams for new problems, and logs these alerts into Redis with a 3-hour TTL to allow re-alerting if problems persist.

- **Nodes Involved:**  
  - Check if Alert Exists (Redis)  
  - Create Response (Code)  
  - Merge (combine alerts with Redis check)  
  - Message Techs (Microsoft Teams)  
  - Log the Alert (Redis)

- **Node Details:**

  - **Check if Alert Exists**  
    - *Type*: Redis  
    - *Role*: Checks Redis database for existing alerts keyed by `NetworkName`. Returns the key name if alert exists or `null` otherwise.  
    - *Input*: Problematic sites from filter node.  
    - *Output*: Redis response about alert existence.  
    - *Edge Cases*: Redis connection errors; key not found returns `null`.

  - **Create Response**  
    - *Type*: Code (JavaScript)  
    - *Role*: Converts Redis `null` response into boolean `alertExists` flag (`false` if `null`, else `true`) for filtering later.  
    - *Input*: Output from Redis node.  
    - *Output*: Items with `alertExists` property.

  - **Merge**  
    - *Type*: Merge node  
    - *Role*: Joins problematic sites data with Redis alert existence data by `NetworkName` to identify which sites have existing alerts. Keeps all non-matching items (i.e., new alerts).  
    - *Mode*: Combine, join mode: keepNonMatches, merge by `NetworkName`.  
    - *Inputs*:  
      - Input 1: Redis alert existence data (from Create Response)  
      - Input 2: Problematic sites  
    - *Output*: Sites without existing alerts for forwarding.

  - **Message Techs**  
    - *Type*: Microsoft Teams node  
    - *Role*: Sends alert message to a Teams channel (Dispatch) for each new problematic network. Message includes network name (hyperlinked), average loss, and latency.  
    - *Configuration*: Uses Teams chat ID for Dispatch channel; HTML formatting in message.  
    - *Edge Cases*: Authentication errors; Teams API limits; message formatting issues.

  - **Log the Alert**  
    - *Type*: Redis  
    - *Role*: Stores the alert key (`NetworkName`) in Redis with a TTL of 3 hours (10800 seconds). After expiration, alert will be sent again if issue persists.  
    - *Input*: From Message Techs node.  
    - *Edge Cases*: Redis connection failures.

  - *Sticky Notes*:  
    - Explains Redis usage to prevent duplicate notifications.  
    - Details merging logic to filter only new alerts.  
    - Describes Teams message format and alert logging with TTL.

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                                  | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                                              |
|--------------------------------|--------------------------|-------------------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger         | Periodic workflow start every 5min (Mon-Fri)    | -                                | Get Meraki Organizations         |                                                                                                                                          |
| When clicking "Execute Workflow"| Manual Trigger           | Manual workflow start                            | -                                | Get Meraki Organizations         |                                                                                                                                          |
| Get Meraki Organizations       | HTTP Request             | Fetch Meraki organizations                       | Schedule Trigger, Manual Trigger  | Get Org Name & ID, Get Uplink Loss and Latency | Explains API key and header usage with link to Meraki API docs                                                                           |
| Get Org Name & ID              | Set                      | Extract and rename org name & ID                 | Get Meraki Organizations          | Get Network IDs                 |                                                                                                                                          |
| Get Network IDs                | HTTP Request             | Fetch networks per org                            | Get Org Name & ID                 | Sets Network Variables           |                                                                                                                                          |
| Sets Network Variables         | Set                      | Extract and rename network fields                | Get Network IDs                  | Combine latency to its respective Network |                                                                                                                                          |
| Get Uplink Loss and Latency    | HTTP Request             | Fetch uplink loss and latency stats per org      | Get Meraki Organizations          | Combine latency to its respective Network |                                                                                                                                          |
| Combine latency to its respective Network | Merge                    | Join network info with uplink stats              | Sets Network Variables, Get Uplink Loss and Latency | Makes Latency and Loss Filterable | Explains merging networks with stats                                                                                                    |
| Makes Latency and Loss Filterable | Set                      | Flatten timeseries latency and loss fields       | Combine latency to its respective Network | Average Latency & Loss over 5m |                                                                                                                                          |
| Average Latency & Loss over 5m | Code                     | Calculate average loss and latency over 5 timestamps | Makes Latency and Loss Filterable | Filters Problematic sites       |                                                                                                                                          |
| Filters Problematic sites      | Code                     | Filter networks exceeding latency or loss thresholds | Average Latency & Loss over 5m   | Check if Alert Exists, Merge     | Explains filtering logic based on thresholds                                                                                             |
| Check if Alert Exists          | Redis                    | Check Redis if alert already exists for network  | Filters Problematic sites         | Create Response                 | Redis used to avoid duplicate alerts                                                                                                     |
| Create Response               | Code                     | Convert Redis null to boolean alertExists flag   | Check if Alert Exists             | Merge                          | Describes handling Redis null response                                                                                                  |
| Merge                        | Merge                    | Combine Redis alert state with problematic sites | Create Response, Filters Problematic sites | Message Techs                  | Explains merging to filter new alerts only                                                                                              |
| Message Techs                 | Microsoft Teams          | Send alert message to Teams Dispatch channel     | Merge                           | Log the Alert                   | Sends Teams alert with hyperlink; includes average loss and latency                                                                     |
| Log the Alert                | Redis                    | Log alert in Redis with 3h TTL                    | Message Techs                   | -                             | Logs alert key with TTL to enable re-alerting                                                                                           |
| Sticky Note                   | Sticky Note              | Workflow section explanation                      | -                                | -                             | Pulling in Info: Overview of data retrieval section                                                                                     |
| Sticky Note1                  | Sticky Note              | Workflow section explanation                      | -                                | -                             | Changing data: Data merging and setup explanation                                                                                       |
| Sticky Note2                  | Sticky Note              | Workflow section explanation                      | -                                | -                             | Notify: Alert push and alert state management explanation                                                                              |
| Sticky Note3                  | Sticky Note              | Workflow section explanation                      | -                                | -                             | Meraki API calls and header requirements                                                                                                |
| Sticky Note4                  | Sticky Note              | Workflow section explanation                      | -                                | -                             | Explanation of merging, average calculation, and filtering                                                                             |
| Sticky Note5                  | Sticky Note              | Workflow section explanation                      | -                                | -                             | Explanation of Redis alert checking, merging, Teams notification, and alert logging                                                   |
| Sticky Note6                  | Sticky Note              | Additional use case suggestion                     | -                                | -                             | Suggests replacing Teams node with PSA integration such as ConnectWise Manage                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node:  
     - Set Cron Expression to `*/5 8-16 * * 1-5` (every 5 minutes, Mon-Fri 8am-5pm).  
   - Add a **Manual Trigger** node for manual execution.

2. **Add "Get Meraki Organizations" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.meraki.com/api/v1/organizations`  
   - Headers:  
     - `Authorization`: Use API key credential (HTTP Header Auth).  
     - `Accept`: `application/json`  
   - Authentication: HTTP Header Auth with your Meraki API key.  
   - Connect outputs of both trigger nodes here.

3. **Add "Get Org Name & ID" node:**  
   - Type: Set  
   - Fields:  
     - `CompanyName` = `{{$json["name"]}}`  
     - `OrgID` = `{{$json["id"]}}`  
   - Connect input from "Get Meraki Organizations".

4. **Add "Get Network IDs" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.meraki.com/api/v1/organizations/{{$json.OrgID}}/networks` (expression)  
   - Headers: same as above (Authorization, Accept)  
   - Authentication: HTTP Header Auth (Meraki API key).  
   - Connect input from "Get Org Name & ID".

5. **Add "Sets Network Variables" node:**  
   - Type: Set  
   - Fields:  
     - `NetworkID` = `{{$json["id"]}}`  
     - `NetworkName` = `{{$json["name"]}}`  
     - `networkURL` = `{{$json["url"]}}`  
   - Connect input from "Get Network IDs".

6. **Add "Get Uplink Loss and Latency" node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.meraki.com/api/v1/organizations/{{$json.id}}/devices/uplinksLossAndLatency?timespan=300` (expression)  
   - Headers: same as above  
   - Authentication: HTTP Header Auth (Meraki API key).  
   - Connect input from "Get Meraki Organizations".

7. **Add "Combine latency to its respective Network" node:**  
   - Type: Merge  
   - Mode: Combine, Join Mode: Enrich Input 1  
   - Merge By Fields: `NetworkID` (Input 1) with `networkId` (Input 2)  
   - Input 1: "Sets Network Variables"  
   - Input 2: "Get Uplink Loss and Latency"

8. **Add "Makes Latency and Loss Filterable" node:**  
   - Type: Set  
   - Add fields flattening timeseries loss and latency:  
     - `TS0-Loss` = `{{$json.timeSeries[0].lossPercent}}`  
     - `TS1-Loss` ... up to `TS4-Loss`  
     - `TS0-Latency` = `{{$json.timeSeries[0].latencyMs}}`  
     - `TS1-Latency` ... up to `TS4-Latency`  
   - Input: "Combine latency to its respective Network"

9. **Add "Average Latency & Loss over 5m" node:**  
   - Type: Code  
   - JavaScript code:  
     ```js
     return items.map(item => {
       const lossFields = ['TS0-Loss','TS1-Loss','TS2-Loss','TS3-Loss','TS4-Loss'];
       const latencyFields = ['TS0-Latency','TS1-Latency','TS2-Latency','TS3-Latency','TS4-Latency'];

       let totalLoss = 0;
       let totalLatency = 0;

       for (const f of lossFields) {
         totalLoss += parseFloat(item.json[f]) || 0;
       }
       for (const f of latencyFields) {
         totalLatency += parseFloat(item.json[f]) || 0;
       }

       item.json.AverageLoss = totalLoss / 5;
       item.json.AverageLatency = totalLatency / 5;

       return item;
     });
     ```
   - Input: "Makes Latency and Loss Filterable"

10. **Add "Filters Problematic sites" node:**  
    - Type: Code  
    - JavaScript code:  
      ```js
      return items.filter(item => item.json.AverageLatency > 300 || item.json.AverageLoss > 2);
      ```
    - Input: "Average Latency & Loss over 5m"

11. **Add "Check if Alert Exists" node:**  
    - Type: Redis  
    - Operation: Get  
    - Key: `{{$json.NetworkName}}`  
    - Options: Dot notation enabled  
    - Connect input from "Filters Problematic sites" node.

12. **Add "Create Response" node:**  
    - Type: Code  
    - JS code:  
      ```js
      return items.map(item => {
        item.json.alertExists = item.json.NetworkName !== null;
        return item;
      });
      ```
    - Input: "Check if Alert Exists"

13. **Add "Merge" node:**  
    - Mode: Combine  
    - Join Mode: Keep Non Matches  
    - Merge by Fields: `NetworkName` (Input 1) and `NetworkName` (Input 2)  
    - Input 1: "Create Response"  
    - Input 2: "Filters Problematic sites" (pass as second input)

14. **Add "Message Techs" node:**  
    - Type: Microsoft Teams  
    - Resource: Chat Message  
    - Chat ID: Your Teams Dispatch channel ID  
    - Message:  
      ```html
      <strong>Loss & Latency Alert</strong> <br><br>
      <strong>Network Name:</strong> <a href="{{ $json.networkURL }}">{{ $json.NetworkName }}</a> <br>
      <strong>Average Loss:</strong> {{ $json.AverageLoss }}% <br>
      <strong>Average Latency:</strong> {{ $json.AverageLatency }} <br>
      ```
    - Input: "Merge" node output

15. **Add "Log the Alert" node:**  
    - Type: Redis  
    - Operation: Set  
    - Key: `{{$json.NetworkName}}`  
    - Value: `{{$json.NetworkName}}`  
    - TTL: 10800 seconds (3 hours)  
    - Input: "Message Techs"

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                         |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Setup typically takes 30 minutes to 1 hour, mostly for acquiring Meraki API key and configuring Teams node | Workflow description                                                                                                    |
| Meraki API key must be generated inside Meraki Dashboard; API docs: https://documentation.meraki.com/General_Administration/Other_Topics/Cisco_Meraki_Dashboard_API | Sticky Note3                                                                                                            |
| Teams node can be replaced by PSA ticketing system integrations (e.g., ConnectWise Manage) for automated ticket creation instead of messages | Sticky Note6                                                                                                            |
| Redis is used here to store active alerts with TTL to avoid duplicate notifications; after TTL expires, alert is re-sent if issue persists | Sticky Note5                                                                                                            |
| Tutorial video explaining workflow and setup: https://www.youtube.com/watch?v=JvaN0dNwRNU                    | Workflow description                                                                                                    |

---

This completes the detailed reference and reconstruction guide for the "Meraki Packet Loss and Latency Alerts to Microsoft Teams" workflow.