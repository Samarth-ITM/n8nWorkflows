Monitor Server Uptime & Get Email Alerts with Google Sheets

https://n8nworkflows.xyz/workflows/monitor-server-uptime---get-email-alerts-with-google-sheets-3880


# Monitor Server Uptime & Get Email Alerts with Google Sheets

### 1. Workflow Overview

This workflow automates real-time monitoring of web servers by periodically pinging them, logging their status, and sending email alerts if any server becomes unreachable. It is designed for IT administrators and DevOps teams who want a lightweight, no-code server uptime monitoring solution integrated with Google Sheets and Gmail.

**Logical Blocks:**

- **1.1 Schedule Trigger:** Initiates the workflow every minute to ensure continuous monitoring.
- **1.2 Web Servers List Retrieval:** Reads the list of servers (IP addresses or hostnames) from a Google Sheet.
- **1.3 Servers Alive Check (HTTP Request):** Sends HTTP GET requests to each server to verify availability.
- **1.4 Web Server Alive Log:** Logs successful server pings with timestamps into a dedicated Google Sheet.
- **1.5 Server Down Notification:** Sends an email alert via Gmail if a server is unreachable.
- **1.6 Web Server Down Log:** Logs failed server pings with timestamps into a separate Google Sheet for audit and troubleshooting.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Triggers the entire workflow every minute, enabling near real-time server monitoring.

- **Nodes Involved:**  
  - `1. Schedule Trigger`

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Initiates the workflow on a fixed interval (every 1 minute).  
  - **Configuration:** Interval set to 1 minute using the built-in scheduling options.  
  - **Inputs:** None (trigger node).  
  - **Outputs:** Connects to `2. Web Servers List`.  
  - **Version Requirements:** n8n v0.115+ recommended for stable scheduleTrigger node.  
  - **Potential Failures:** Scheduler misconfiguration or n8n instance downtime could delay or stop triggers.

#### 1.2 Web Servers List Retrieval

- **Overview:**  
  Fetches the list of servers to monitor from a Google Sheet named `Server_List`. Each row corresponds to one server.

- **Nodes Involved:**  
  - `2. Web Servers List`

- **Node Details:**  
  - **Type:** Google Sheets (Read operation)  
  - **Role:** Reads all rows from the `Server_List` sheet in the specified Google Sheets document.  
  - **Configuration:**  
    - Document ID: `1OiwBkf3bs3tcfi5cAIrOl_GrXw2EfQLdcPbh6SaBFKQ`  
    - Sheet Name: `Server_List` (gid: 524060827)  
    - Reads all rows without filters.  
  - **Inputs:** Receives trigger from `1. Schedule Trigger`.  
  - **Outputs:** Sends each server row to `3. Servers Alive Check (HTTP)`.  
  - **Credentials:** Uses Google Sheets OAuth2 credentials.  
  - **Potential Failures:**  
    - Authentication errors if OAuth token expires.  
    - Sheet or document ID changes or access revoked.  
    - Empty or malformed sheet rows causing downstream errors.

#### 1.3 Servers Alive Check (HTTP Request)

- **Overview:**  
  Sends an HTTP GET request to each server URL constructed from the server address read from the sheet. Determines if the server is reachable.

- **Nodes Involved:**  
  - `3. Servers Alive Check (HTTP)`

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Performs HTTP GET requests to `http://{{ $json.Server }}` for each server.  
  - **Configuration:**  
    - URL template: `http://{{ $json.Server }}` (dynamic based on server field)  
    - `continueOnFail` enabled to handle unreachable servers gracefully without stopping workflow.  
  - **Inputs:** Receives server list items from `2. Web Servers List`.  
  - **Outputs:**  
    - On success: Connects to `4. Web Server Alive Log`.  
    - On failure: Connects to `5. Server Down Notification`.  
  - **Potential Failures:**  
    - Network timeouts or DNS resolution failures.  
    - HTTP errors (e.g., 404, 500) treated as failures.  
    - Malformed server addresses causing invalid URLs.  
  - **Notes:** The error handling path is critical to trigger alerts and logging for down servers.

#### 1.4 Web Server Alive Log

- **Overview:**  
  Logs successful server pings with timestamp and status "Alive" into the `Server_Status_Alive` Google Sheet.

- **Nodes Involved:**  
  - `4. Web Server Alive Log`

- **Node Details:**  
  - **Type:** Google Sheets (Append operation)  
  - **Role:** Appends a new row for each successful ping with:  
    - `TimeStamp`: current datetime in `yyyy-MM-dd HH:mm:ss` format  
    - `Server IP Address`: server address from the HTTP request node  
    - `Status`: fixed string "Alive"  
  - **Configuration:**  
    - Document ID: same as above  
    - Sheet Name: `Server_Status_Alive` (gid: 303958634)  
    - Mapping mode: manual column definition  
  - **Inputs:** Receives successful HTTP request items from `3. Servers Alive Check (HTTP)`.  
  - **Outputs:** None (end of success path).  
  - **Potential Failures:**  
    - Google Sheets API quota limits.  
    - Authentication issues.  
    - Schema mismatch if sheet columns are altered.

#### 1.5 Server Down Notification

- **Overview:**  
  Sends an email alert via Gmail when a server ping fails, including server address and failure timestamp.

- **Nodes Involved:**  
  - `5. Server Down Notification`

- **Node Details:**  
  - **Type:** Gmail (Send Email)  
  - **Role:** Sends alert emails to a configured recipient.  
  - **Configuration:**  
    - Recipient: `automation0.n8n@gmail.com` (editable)  
    - Subject: `🔻 Server Down: {{ $json.Server }}: {{ $today.format('yyyy-MM-dd') }}`  
    - Message body includes:  
      - Failure timestamp (`{{$now.format('yyyy-MM-dd HH:mm:ss')}}`)  
      - Server address (`{{ $json["Server"] }}`)  
      - Suggested action text  
    - Email type: plain text  
    - Attribution appended automatically  
  - **Inputs:** Receives failed HTTP request items from `3. Servers Alive Check (HTTP)` error path.  
  - **Outputs:** Connects to `6. Web Server Down Log` for logging.  
  - **Credentials:** Uses Gmail OAuth2 credentials.  
  - **Potential Failures:**  
    - Gmail API quota or authentication errors.  
    - Invalid recipient email address.  
    - Network issues preventing email sending.

#### 1.6 Web Server Down Log

- **Overview:**  
  Logs failed server pings with timestamp and status "Down" into the `Server_Status_Down` Google Sheet for historical tracking.

- **Nodes Involved:**  
  - `6. Web Server Down Log`

- **Node Details:**  
  - **Type:** Google Sheets (Append operation)  
  - **Role:** Appends a new row for each failed ping with:  
    - `TimeStamp`: current datetime in `yyyy-MM-dd HH:mm:ss` format  
    - `Server IP Address`: server address from the HTTP request failure item  
    - `Status`: fixed string "Down"  
  - **Configuration:**  
    - Document ID: same as above  
    - Sheet Name: `Server_Status_Down` (gid: 0)  
    - Mapping mode: manual column definition  
  - **Inputs:** Receives items from `5. Server Down Notification`.  
  - **Outputs:** None (end of failure path).  
  - **Potential Failures:**  
    - Google Sheets API quota or authentication issues.  
    - Sheet schema changes causing mapping errors.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                          | Input Node(s)              | Output Node(s)                       | Sticky Note                                                                                                         |
|-----------------------------|---------------------|----------------------------------------|----------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| 1. Schedule Trigger          | Schedule Trigger    | Triggers workflow every minute         | -                          | 2. Web Servers List                | 📘 Node Descriptions for Your Web Server Monitor Workflow (covers all nodes)                                         |
| 2. Web Servers List          | Google Sheets       | Reads server list from Google Sheet    | 1. Schedule Trigger         | 3. Servers Alive Check (HTTP)      | 📘 Node Descriptions for Your Web Server Monitor Workflow (covers all nodes)                                         |
| 3. Servers Alive Check (HTTP)| HTTP Request        | Pings each server, routes success/fail | 2. Web Servers List         | 4. Web Server Alive Log, 5. Server Down Notification | 📘 Node Descriptions for Your Web Server Monitor Workflow (covers all nodes)                                         |
| 4. Web Server Alive Log      | Google Sheets       | Logs successful pings                   | 3. Servers Alive Check (HTTP) | -                                  | 📘 Node Descriptions for Your Web Server Monitor Workflow (covers all nodes)                                         |
| 5. Server Down Notification  | Gmail               | Sends alert email on server failure    | 3. Servers Alive Check (HTTP) (error path) | 6. Web Server Down Log             | 📘 Node Descriptions for Your Web Server Monitor Workflow (covers all nodes)                                         |
| 6. Web Server Down Log       | Google Sheets       | Logs failed pings                       | 5. Server Down Notification | -                                  | 📘 Node Descriptions for Your Web Server Monitor Workflow (covers all nodes)                                         |
| Sticky Note                 | Sticky Note          | Documentation note                     | -                          | -                                  | 📘 Node Descriptions for Your Web Server Monitor Workflow                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `1. Schedule Trigger`  
   - Set interval to every 1 minute (field: minutesInterval = 1).  
   - No credentials needed.

3. **Add a Google Sheets node to read servers list:**  
   - Name: `2. Web Servers List`  
   - Operation: Read rows from sheet.  
   - Configure credentials with Google Sheets OAuth2.  
   - Document ID: `1OiwBkf3bs3tcfi5cAIrOl_GrXw2EfQLdcPbh6SaBFKQ` (replace with your own).  
   - Sheet Name: `Server_List` (or your sheet name).  
   - Connect `1. Schedule Trigger` output to this node input.

4. **Add an HTTP Request node to ping servers:**  
   - Name: `3. Servers Alive Check (HTTP)`  
   - HTTP Method: GET  
   - URL: `http://{{ $json.Server }}` (dynamic from sheet data)  
   - Enable `continueOnFail` to true (to handle failures without stopping).  
   - Connect `2. Web Servers List` output to this node input.

5. **Add a Google Sheets node to log alive servers:**  
   - Name: `4. Web Server Alive Log`  
   - Operation: Append row.  
   - Use same Google Sheets credentials.  
   - Document ID: same as above.  
   - Sheet Name: `Server_Status_Alive`.  
   - Map columns:  
     - `TimeStamp`: `{{$now.format('yyyy-MM-dd HH:mm:ss')}}`  
     - `Server IP Address`: `{{$node["2. Web Servers List"].json["Server"]}}`  
     - `Status`: `"Alive"`  
   - Connect the success output of `3. Servers Alive Check (HTTP)` to this node.

6. **Add a Gmail node to send down notifications:**  
   - Name: `5. Server Down Notification`  
   - Credentials: Gmail OAuth2 (configure your Gmail account).  
   - Recipient: set to your alert email (e.g., `automation0.n8n@gmail.com`).  
   - Subject: `🔻 Server Down: {{ $json.Server }}: {{ $today.format('yyyy-MM-dd') }}`  
   - Message:  
     ```
     Hi Team,

     At {{$now.format('yyyy-MM-dd HH:mm:ss')}}, the following server failed to respond to ping:

     🔻 Server Down: {{ $json.Server }}

     Please investigate immediately to prevent service disruption.

     Automated Monitoring System
     ```  
   - Email type: Plain text  
   - Connect the error output of `3. Servers Alive Check (HTTP)` to this node.

7. **Add a Google Sheets node to log down servers:**  
   - Name: `6. Web Server Down Log`  
   - Operation: Append row.  
   - Use same Google Sheets credentials.  
   - Document ID: same as above.  
   - Sheet Name: `Server_Status_Down`.  
   - Map columns:  
     - `TimeStamp`: `{{$now.format('yyyy-MM-dd HH:mm:ss')}}`  
     - `Server IP Address`: `{{$node["3. Servers Alive Check (HTTP)"].json["Server"]}}`  
     - `Status`: `"Down"`  
   - Connect the output of `5. Server Down Notification` to this node.

8. **Activate the workflow.**

9. **Prepare your Google Sheets document:**  
   - Sheet 1 (`Server_List`): Column header `Server` with server IPs or hostnames listed below.  
   - Sheet 2 (`Server_Status_Alive`): Columns `TimeStamp`, `Server IP Address`, `Status`.  
   - Sheet 3 (`Server_Status_Down`): Columns `TimeStamp`, `Server IP Address`, `Status`.

10. **Test the workflow:**  
    - Add both reachable and unreachable servers to `Server_List`.  
    - Run the workflow manually or wait for the schedule trigger.  
    - Verify logs and email alerts.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow uses Google Sheets for dynamic server list management, enabling easy scaling without workflow edits.                | Workflow description                                                                                         |
| Gmail OAuth2 credentials must be configured properly to send email alerts; ensure the Gmail account has API access enabled.       | Gmail node configuration                                                                                     |
| Adjust HTTP Request URL template if servers require ports or specific paths (e.g., `http://{{Server}}:8080/status`).               | HTTP Request node configuration                                                                              |
| The workflow logs both successful and failed pings for audit and reporting purposes.                                               | Workflow description                                                                                         |
| The workflow is designed to run every minute but can be adjusted in the Schedule Trigger node as needed.                          | Schedule Trigger node configuration                                                                           |
| For best results, ensure your n8n instance is running continuously and has stable internet access.                                 | General operational advice                                                                                    |
| Google Sheets API quotas may limit the number of requests; monitor usage if scaling to many servers.                              | Google Sheets node potential failure notes                                                                   |
| Sticky Note node in the workflow contains detailed node descriptions and can be used as in-editor documentation.                  | Sticky Note node content                                                                                      |
| Google Sheets document URL example: https://docs.google.com/spreadsheets/d/1OiwBkf3bs3tcfi5cAIrOl_GrXw2EfQLdcPbh6SaBFKQ/edit      | Provided in node configurations                                                                               |

---

This structured documentation provides a comprehensive understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the server monitoring automation effectively.