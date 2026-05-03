Weekly Shodan Query - Report Accidents

https://n8nworkflows.xyz/workflows/weekly-shodan-query---report-accidents-1977


# Weekly Shodan Query - Report Accidents

---
### 1. Workflow Overview

This workflow, named **Weekly Shodan Query - Report Accidents**, automates the weekly monitoring of specified IP addresses and their expected open ports to detect unexpected open ports that could indicate security incidents. Running every Monday at 5:00 AM, it performs the following logical blocks:

- **1.1 Scheduled Trigger & Input Acquisition:** Triggered weekly, it fetches the list of monitored IP addresses and their expected ports via HTTP.
- **1.2 IP Address Iteration:** Processes each IP individually to manage API load and rate limits.
- **1.3 Shodan Data Retrieval & Service Extraction:** Queries Shodan’s API for detailed info on each IP and extracts the array of services/ports reported by Shodan.
- **1.4 Unexpected Port Filtering:** Compares Shodan-reported ports against the expected ports; filters out expected ports and retains unexpected ones.
- **1.5 Data Formatting:** For each unexpected port, gathers relevant details and formats them first into an HTML table, then converts it into Markdown format.
- **1.6 Alert Creation in TheHive:** Posts an alert to TheHive for any unexpected ports detected, encapsulating the findings with structured metadata for incident response.

This modular approach enables clear monitoring and reporting of anomalous open ports, facilitating proactive network security management.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Input Acquisition

- **Overview:** Initiates the workflow every Monday at 5:00 AM and retrieves the current list of IPs and their expected ports to monitor.
- **Nodes Involved:** 
  - Every Monday (Schedule Trigger)
  - Get watched IPs & Ports (HTTP Request)
  - For each IP (Split In Batches)

- **Node Details:**

  - **Every Monday**
    - Type: Schedule Trigger
    - Role: Triggers workflow weekly on Mondays at 5:00 AM.
    - Parameters: Interval set to weeks, trigger at day 1 (Monday), hour 5.
    - Inputs: None (trigger node)
    - Outputs: Connected to "Get watched IPs & Ports"
    - Potential Failures: Timezone misconfiguration, scheduler downtime.

  - **Get watched IPs & Ports**
    - Type: HTTP Request
    - Role: Fetches list of IP addresses and arrays of expected ports.
    - Parameters: GET request to `https://internal.users.n8n.cloud/webhook/mock-shodan-ips`
    - Outputs: JSON array of objects like `{ip: "x.x.x.x", ports: [port1, port2]}`
    - Inputs: Triggered by "Every Monday"
    - Outputs: Connected to "For each IP"
    - Edge Cases: API downtime, unexpected JSON structure, empty list.
    - Notes: Should be replaced with organization's actual IPS or database endpoint.
  
  - **For each IP**
    - Type: Split In Batches
    - Role: Iterates over each IP one at a time to manage load and rate limits.
    - Parameters: Batch size = 1
    - Inputs: JSON array from "Get watched IPs & Ports"
    - Outputs: Sends one IP object at a time to "Scan each IP"
    - Edge Cases: Empty or malformed input arrays.
    - Sticky Note (covers this block): Explains importance of batch size 1 for focused analysis and API rate limit management.

#### 2.2 Shodan Data Retrieval & Service Extraction

- **Overview:** For each IP, this block queries Shodan API to retrieve detailed host information and extracts the list of services (ports) found on that IP.
- **Nodes Involved:**
  - Scan each IP (HTTP Request)
  - Split out services (Item Lists)
  - Unexpected port? (Filter)

- **Node Details:**

  - **Scan each IP**
    - Type: HTTP Request
    - Role: Queries Shodan API for host information on the current IP.
    - Parameters:
      - URL templated as `https://api.shodan.io/shodan/host/{{ $json.ip }}`
      - Authentication: HTTP Query Auth with Shodan API Key credential.
    - Inputs: One IP from "For each IP"
    - Outputs: JSON with detailed host data, including `data` field (list of services)
    - Edge Cases: API rate limits, invalid API key, IP not found, network errors.
    - Sticky Note: Describes querying Shodan to identify running services and ports.

  - **Split out services**
    - Type: Item Lists
    - Role: Splits the `data` array (services) from Shodan response into individual items for filtering.
    - Parameters: Field to split out = `data`
    - Inputs: Output from "Scan each IP"
    - Outputs: Single service per item to "Unexpected port?"
    - Edge Cases: Empty or missing `data` field.
    
  - **Unexpected port?**
    - Type: Filter
    - Role: Determines if current port is unexpected by checking if port is NOT included in the expected ports list.
    - Parameters: Boolean condition - checks if port is included in expected ports from original IP object and negates it.
    - Inputs: One service item from "Split out services"
    - Outputs: Items that do NOT match any expected port go forward; others are filtered out.
    - Edge Cases: Null or missing port values, unexpected data types.
  
#### 2.3 Data Formatting

- **Overview:** Organizes the filtered unexpected port data into a structured HTML table and converts it to Markdown for readability and report integration.
- **Nodes Involved:**
  - Set data to post for each port (Set)
  - Convert to table (HTML)
  - Convert to Markdown (Markdown)

- **Node Details:**

  - **Set data to post for each port**
    - Type: Set
    - Role: Prepares a simplified object with relevant fields for reporting.
    - Parameters: Sets fields `ip`, `hostnames` (joined as CSV), `port`, `description`, and raw `data`.
    - Inputs: Filtered unexpected port item from "Unexpected port?"
    - Outputs: Formatted data to "Convert to table"
    - Edge Cases: Missing or empty fields, null hostnames array.

  - **Convert to table**
    - Type: HTML
    - Role: Converts the set JSON data into an HTML table.
    - Parameters: Operation set to "convertToHtmlTable"
    - Inputs: Objects from "Set data to post for each port"
    - Outputs: HTML table string to "Convert to Markdown"
    - Edge Cases: Malformed input data causing invalid HTML.
    - Sticky Note: Explains importance of Markdown table formatting for readability and sharing.

  - **Convert to Markdown**
    - Type: Markdown
    - Role: Converts HTML table to Markdown format, stored under key `markdown`.
    - Parameters: Source HTML taken from previous node output `table`
    - Inputs: HTML table from "Convert to table"
    - Outputs: Markdown formatted text for alert description.
    - Edge Cases: HTML conversion failures, empty inputs.

#### 2.4 Alert Creation in TheHive

- **Overview:** Posts an alert to TheHive platform for each unexpected port detected, including the Markdown report and relevant metadata for incident response.
- **Nodes Involved:**
  - Create TheHive alert (TheHive)
  - For each IP (Split In Batches) — used again to continue processing

- **Node Details:**

  - **Create TheHive alert**
    - Type: TheHive
    - Role: Creates an alert in TheHive with details of unexpected open ports.
    - Parameters:
      - Title: "Unexpected ports for {{ ip }}"
      - Description: Includes Markdown-formatted unexpected port data.
      - Severity: Medium
      - Date: current date/time (`{{$now}}`)
      - Tags: IP address
      - TLP: Amber (Traffic Light Protocol)
      - Status: New
      - Type: "Unexpected open port"
      - Source: "n8n"
      - SourceRef: Unique combination of IP and Unix timestamp.
      - Follow and JSON parameters enabled.
    - Credentials: Uses TheHive API credentials
    - Inputs: Markdown data from "Convert to Markdown"
    - Outputs: Connected back to "For each IP" for further processing or completion.
    - Edge Cases: API authentication failure, network issues, malformed alert data.
    - Sticky Note: Describes integration with TheHive for incident escalation and case management.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                                            | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                       |
|-------------------------|--------------------|------------------------------------------------------------|-------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Every Monday            | Schedule Trigger   | Triggers workflow weekly on Mondays at 5:00 AM             | None                    | Get watched IPs & Ports   | ![Scheduled](https://i.imgur.com/PcOuvAL.png) Explains workflow overview and scheduled trigger                                     |
| Get watched IPs & Ports | HTTP Request       | Fetches list of IPs and expected ports                      | Every Monday            | For each IP               | See above note                                                                                                                   |
| For each IP             | Split In Batches   | Iterates over IPs one at a time                             | Get watched IPs & Ports | Scan each IP              | ![n8n](https://i.imgur.com/lKnBNnH.png) Explains batch processing and iteration                                                  |
| Scan each IP            | HTTP Request       | Queries Shodan API for host info                            | For each IP             | Split out services        | ![Shodan](https://i.imgur.com/q4G3kQf.png) Describes Shodan querying and service extraction                                      |
| Split out services      | Item Lists         | Splits Shodan services array into separate items            | Scan each IP            | Unexpected port?          | See above note                                                                                                                   |
| Unexpected port?        | Filter             | Filters out expected ports, passes unexpected ports        | Split out services      | Set data to post for each port | See above note                                                                                                               |
| Set data to post for each port | Set          | Prepares data fields for reporting                          | Unexpected port?        | Convert to table          | ![Shodan](https://i.imgur.com/tK0RXSK.png) Explains formatting data as Markdown table                                           |
| Convert to table        | HTML               | Converts data into an HTML table                            | Set data to post for each port | Convert to Markdown | See above note                                                                                                                   |
| Convert to Markdown     | Markdown           | Converts HTML table to Markdown                             | Convert to table        | Create TheHive alert      | See above note                                                                                                                   |
| Create TheHive alert    | TheHive            | Posts alert to TheHive platform                             | Convert to Markdown     | For each IP               | ![thehive](https://i.imgur.com/y2Yw1ZP.png) Describes final alert creation and integration with TheHive                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**
   - Name: `Every Monday`
   - Type: Schedule Trigger
   - Set interval: Weekly, trigger at Monday (day=1), hour=5:00 AM
   - No credentials needed
   - Connect output to next node

2. **Create HTTP Request Node:**
   - Name: `Get watched IPs & Ports`
   - Type: HTTP Request
   - Method: GET
   - URL: `https://internal.users.n8n.cloud/webhook/mock-shodan-ips`
   - No authentication (or configure as required)
   - Connect input from `Every Monday`
   - Output goes to next node

3. **Create Split In Batches Node:**
   - Name: `For each IP`
   - Type: Split In Batches
   - Batch size: 1 (process one IP at a time)
   - Input from `Get watched IPs & Ports`
   - Output goes to next node

4. **Create HTTP Request Node for Shodan Query:**
   - Name: `Scan each IP`
   - Type: HTTP Request
   - Method: GET
   - URL: `https://api.shodan.io/shodan/host/{{ $json.ip }}`
   - Authentication: HTTP Query Auth
     - Use Shodan API key credential
   - Input from `For each IP`
   - Output goes to next node

5. **Create Item Lists Node:**
   - Name: `Split out services`
   - Type: Item Lists
   - Field to split out: `data`
   - Input from `Scan each IP`
   - Output goes to next node

6. **Create Filter Node:**
   - Name: `Unexpected port?`
   - Type: Filter
   - Condition: Boolean expression to check if the port from Shodan is NOT included in the expected ports array from the original IP object.
     - Expression example: `={{ !$node["For each IP"].item.json.ports.includes($json.port) }}`
   - Input from `Split out services`
   - Output goes to next node

7. **Create Set Node:**
   - Name: `Set data to post for each port`
   - Type: Set
   - Set fields:
     - `ip` = `={{ $node["Get watched IPs & Ports"].item.json.ip }}`
     - `hostnames` = `={{ $json.hostnames.join(', ') }}`
     - `port` = `={{ $json.port }}`
     - `description` = `={{ $json.description }}`
     - `data` = `={{ $json.data }}`
   - Keep only these fields
   - Input from `Unexpected port?`
   - Output goes to next node

8. **Create HTML Node:**
   - Name: `Convert to table`
   - Type: HTML
   - Operation: convertToHtmlTable
   - Input from `Set data to post for each port`
   - Output goes to next node

9. **Create Markdown Node:**
   - Name: `Convert to Markdown`
   - Type: Markdown
   - HTML parameter: `={{ $json.table }}`
   - Destination key: `markdown`
   - Input from `Convert to table`
   - Output goes to next node

10. **Create TheHive Node:**
    - Name: `Create TheHive alert`
    - Type: TheHive
    - Parameters:
      - Title: `=Unexpected ports for {{ $node["For each IP"].last().json.ip }}`
      - Description: `=Unexpected open ports:\n\n{{ $json.markdown }}`
      - Severity: Medium
      - Date: `={{ $now }}`
      - Tags: `={{ $node["For each IP"].last().json.ip }}`
      - TLP: Amber
      - Status: New
      - Type: Unexpected open port
      - Source: n8n
      - SourceRef: `={{ $node["For each IP"].last().json.ip }}:{{$now.toUnixInteger()}}`
      - Enable Follow: true
      - Enable JSON Parameters: true
    - Credentials: TheHive API credentials
    - Input from `Convert to Markdown`
    - Output connected back to `For each IP` or end as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to be integrated into enterprise security operations for continuous monitoring of network endpoints and rapid incident detection.                                                                                                                                                           | Workflow purpose                                                                                                    |
| The Shodan API key must have sufficient query quota and permissions to retrieve detailed host data.                                                                                                                                                                                                                   | Credential requirement                                                                                              |
| TheHive integration requires API access with permissions to create alerts and incidents; ensure API credentials are securely stored in n8n.                                                                                                                                                                         | Credential requirement                                                                                              |
| Error handling for HTTP requests and API limits should be implemented in production workflows to handle network failures, rate limits, or unexpected data formats.                                                                                                                                                  | Best practice advice                                                                                                |
| The expected input JSON format for IPs and ports must be strictly followed for the workflow to process correctly; e.g., `[{"ip":"116.202.106.35","ports":[5678,80]}, {"ip":"188.114.96.9","ports":[8080,80]}]`                                                                                                         | Input format example                                                                                                |
| Use of batch size 1 in `Split In Batches` nodes is crucial to respect Shodan API rate limits and ensure detailed processing per IP.                                                                                                                                                                                  | Performance consideration                                                                                           |
| Markdown formatting greatly improves the readability of alert reports in TheHive, facilitating faster incident triage.                                                                                                                                                                                               | Reporting best practice                                                                                            |
| Documentation and usage of Shodan API are available at https://developer.shodan.io/                                                                                                                                                                                                                                     | Shodan API documentation                                                                                            |
| TheHive official documentation: https://thehive-project.org/                                                                                                                                                                                                                                                         | TheHive platform documentation                                                                                      |

---

This document provides a complete and detailed understanding of the "Weekly Shodan Query - Report Accidents" n8n workflow and enables replication, modification, and effective troubleshooting.