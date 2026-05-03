Proxmox System Monitor - VM Status, Host Resources & Temperature Alerts via Telegram

https://n8nworkflows.xyz/workflows/proxmox-system-monitor---vm-status--host-resources---temperature-alerts-via-telegram-9733


# Proxmox System Monitor - VM Status, Host Resources & Temperature Alerts via Telegram

### 1. Workflow Overview

This workflow automates monitoring of a Proxmox VE server, reporting virtual machine (VM) statuses, host resource usage, and temperature alerts every 15 minutes. It is designed for system administrators who want automated, consolidated status updates via Telegram chat. The workflow encompasses these logical blocks:

- **1.1 Scheduling and Initialization:** Triggers the workflow periodically and sets configuration variables.
- **1.2 Authentication:** Logs into the Proxmox API to obtain session credentials.
- **1.3 Data Collection:** Retrieves VM list, recent node tasks, node resource status, and sensor temperatures via SSH.
- **1.4 Data Processing:** Analyzes collected data to compute VM states, resource usage, temperature warnings, and uptime.
- **1.5 Message Generation:** Formats the data into a rich HTML message.
- **1.6 Telegram Notification:** Sends the formatted report to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Initialization

**Overview:**  
This block triggers the workflow every 15 minutes and initializes key variables such as Proxmox connection details and Telegram chat ID.

**Nodes Involved:**  
- Schedule Every 15min  
- Set Variables

**Node Details:**

- **Schedule Every 15min**  
  - *Type & Role:* Schedule Trigger node. Initiates workflow execution every 15 minutes.  
  - *Configuration:* Interval set to 15 minutes.  
  - *Connections:* Outputs to "Set Variables".  
  - *Edge Cases:* If n8n is offline or overloaded, scheduled triggers may be delayed or missed.

- **Set Variables**  
  - *Type & Role:* Set node. Defines static configuration parameters used in the workflow.  
  - *Configuration:* Assigns Proxmox IP, port, node name, Telegram chat ID, Proxmox username, and password as string variables.  
  - *Key Variables:*  
    - `PROXMOX_IP`: e.g., 192.168.1.100  
    - `PROXMOX_PORT`: e.g., 8006  
    - `PROXMOX_NODE`: e.g., pve  
    - `TELEGRAM_CHAT_ID`: Telegram chat identifier  
    - `PROXMOX_USER`: Proxmox login user, e.g., root@pam  
    - `PROXMOX_PASSWORD`: Proxmox password (plaintext here, consider secure storage)  
  - *Connections:* Outputs to "Proxmox Login" node.  
  - *Edge Cases:* Incorrect variable values will cause authentication or API failures. Password stored in plaintext could be a security risk.

---

#### 2.2 Authentication

**Overview:**  
Logs into Proxmox API to obtain an authentication ticket and CSRF token necessary for subsequent API calls.

**Nodes Involved:**  
- Proxmox Login  
- Prepare Auth

**Node Details:**

- **Proxmox Login**  
  - *Type & Role:* HTTP Request node. Performs POST request to Proxmox API `/access/ticket` endpoint for authentication.  
  - *Configuration:*  
    - URL dynamically constructed from variables (`PROXMOX_IP` and `PROXMOX_PORT`).  
    - Sends form-urlencoded body containing `username` and `password`.  
    - Allows unauthorized SSL certificates (useful if using self-signed certs).  
  - *Input:* Receives variables from "Set Variables".  
  - *Output:* JSON response containing `ticket` and `CSRFPreventionToken`.  
  - *Edge Cases:*  
    - Authentication failure due to wrong credentials or network issues.  
    - SSL errors if certificate is invalid and allowUnauthorizedCerts is false.  
    - API changes in Proxmox could break endpoint or response format.

- **Prepare Auth**  
  - *Type & Role:* Set node. Extracts authentication tokens and reassigns essential variables for downstream use.  
  - *Configuration:* Maps `ticket` and `csrf` tokens from "Proxmox Login" output. Passes through `PROXMOX_IP`, `PROXMOX_PORT`, and `PROXMOX_NODE` variables.  
  - *Input:* From "Proxmox Login".  
  - *Output:* To "API - VM List".  
  - *Edge Cases:* Expression failures if expected fields missing in login response.

---

#### 2.3 Data Collection

**Overview:**  
Retrieves detailed data from Proxmox API and system sensors: VM list, recent tasks, node status, and temperatures via SSH.

**Nodes Involved:**  
- API - VM List  
- API - Node Tasks  
- API - Node Status  
- SSH - Get Sensors

**Node Details:**

- **API - VM List**  
  - *Type & Role:* HTTP Request node. Fetches the list of VMs for the specified Proxmox node.  
  - *Configuration:*  
    - GET request to `/nodes/{node}/qemu`.  
    - Uses authentication cookies and CSRF tokens from "Prepare Auth".  
    - Allows unauthorized certs.  
  - *Input:* From "Prepare Auth".  
  - *Output:* Passes data to "API - Node Tasks".  
  - *Edge Cases:* API response errors, invalid auth tokens, network issues.

- **API - Node Tasks**  
  - *Type & Role:* HTTP Request node. Retrieves recent tasks (e.g., VM stop/shutdown events) from node logs.  
  - *Configuration:*  
    - GET request to `/nodes/{node}/tasks?limit=100`.  
    - Uses auth tokens.  
  - *Input:* From "API - VM List".  
  - *Output:* To "API - Node Status".  
  - *Edge Cases:* Large task history may slow response, auth expiration.

- **API - Node Status**  
  - *Type & Role:* HTTP Request node. Fetches current node resource usage and uptime.  
  - *Configuration:*  
    - GET request to `/nodes/{node}/status`.  
    - Uses auth tokens.  
  - *Input:* From "API - Node Tasks".  
  - *Output:* To "SSH - Get Sensors".  
  - *Edge Cases:* API errors, unexpected data formats.

- **SSH - Get Sensors**  
  - *Type & Role:* SSH node. Executes shell commands on the Proxmox host to retrieve temperature sensor data and uptime info.  
  - *Configuration:*  
    - Runs `sensors` command filtered for temperature lines and `uptime` command.  
    - Requires SSH Password credential configured separately with Proxmox host access.  
  - *Input:* From "API - Node Status".  
  - *Output:* To "Process Data".  
  - *Edge Cases:*  
    - SSH connection failure (wrong credentials, network).  
    - Missing or uninstalled `lm-sensors` package on Proxmox host.  
    - Command output format changes.

---

#### 2.4 Data Processing

**Overview:**  
Aggregates and analyzes all collected data to compute VM counts/statuses, detect recent VM stops, calculate CPU/memory usage, and evaluate temperature warnings.

**Nodes Involved:**  
- Process Data

**Node Details:**

- **Process Data**  
  - *Type & Role:* Code node (JavaScript). Complex logic to parse and analyze data from API calls and SSH output.  
  - *Key Logic:*  
    - Validates presence of VM, node status, and task data, throwing errors if missing.  
    - Parses SSH output to extract temperature readings and detects if current temperature meets or exceeds predefined high thresholds.  
    - Computes VM totals, running and stopped count.  
    - Identifies VMs stopped within the last 15 minutes using task logs.  
    - Calculates CPU usage percentage, memory used/total/percentage, and uptime in days/hours.  
    - Returns a structured JSON object summarizing all details and a temperature warning flag.  
  - *Input:* From "SSH - Get Sensors".  
  - *Output:* To "Generate Formatted Message".  
  - *Edge Cases:*  
    - Parsing errors if SSH output format changes.  
    - Missing API data fields.  
    - Time synchronization issues affecting stop time calculations.

---

#### 2.5 Message Generation

**Overview:**  
Creates a well-formatted HTML message including VM stats, host resource usage, temperature info, and alerts suitable for Telegram.

**Nodes Involved:**  
- Generate Formatted Message

**Node Details:**

- **Generate Formatted Message**  
  - *Type & Role:* Code node (JavaScript). Converts processed data into HTML with bold tags, code formatting, and emojis for warnings.  
  - *Logic:*  
    - Adds a fire emoji if temperature warning present.  
    - Lists recently stopped VMs with their stop times.  
    - Formats timestamp in user-friendly locale string.  
    - Creates sections with visual separators and clear labels.  
  - *Input:* From "Process Data".  
  - *Output:* To "Send Telegram Report".  
  - *Edge Cases:*  
    - HTML formatting issues if data contains unexpected characters.  
    - Date formatting dependent on environment locale.

---

#### 2.6 Telegram Notification

**Overview:**  
Sends the generated HTML report to the configured Telegram chat using the Telegram Bot API.

**Nodes Involved:**  
- Send Telegram Report

**Node Details:**

- **Send Telegram Report**  
  - *Type & Role:* Telegram node. Sends a message to a Telegram chat using a configured bot.  
  - *Configuration:*  
    - Text set from previous node output (`message`).  
    - Uses Telegram chat ID from "Set Variables".  
    - Message parsed as HTML (parse_mode = HTML).  
    - Attribution disabled.  
    - Credentials: Telegram API token configured separately.  
  - *Input:* From "Generate Formatted Message".  
  - *Output:* Final node.  
  - *Edge Cases:*  
    - Telegram API rate limits or downtime.  
    - Invalid bot token or chat ID causing send failure.  
    - Message size limits.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                          | Input Node(s)         | Output Node(s)              | Sticky Note                                                                                             |
|---------------------|----------------------|----------------------------------------|-----------------------|----------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Every 15min | Schedule Trigger     | Triggers workflow every 15 minutes     | —                     | Set Variables              | ## Proxmox Monitoring Workflow Monitors your Proxmox VE server every 15 minutes and sends automated reports via Telegram. Tracks VM status, host resources, temperature sensors, and recently stopped VMs. |
| Set Variables       | Set                  | Defines connection and chat variables  | Schedule Every 15min   | Proxmox Login              | ## CONFIGURATION REQUIRED 1. Set your Proxmox IP, port, and node name 2. Enter Proxmox username (with realm, e.g. root@pam) 3. Add your Proxmox password 4. Set your Telegram Chat ID Requires: lm-sensors installed on Proxmox host |
| Proxmox Login       | HTTP Request         | Authenticates with Proxmox API         | Set Variables          | Prepare Auth               | ## Authentication Flow Logs into Proxmox API to obtain: - Session ticket (valid 15 minutes) - CSRF prevention token Both are required for subsequent API calls |
| Prepare Auth        | Set                  | Extracts auth tokens and passes variables | Proxmox Login          | API - VM List              |                                                                                                       |
| API - VM List       | HTTP Request         | Retrieves list of VMs on Proxmox node  | Prepare Auth           | API - Node Tasks           | ## Data Collection Gathers data from Proxmox API: - VM list and status - Recent task log (stop/shutdown events) - Node resources (CPU, memory, uptime) Runs sequentially to ensure authenticated requests |
| API - Node Tasks    | HTTP Request         | Gets recent node tasks (e.g., VM stops)| API - VM List          | API - Node Status          |                                                                                                       |
| API - Node Status   | HTTP Request         | Fetches node CPU, memory, uptime status| API - Node Tasks       | SSH - Get Sensors          |                                                                                                       |
| SSH - Get Sensors   | SSH                  | Runs sensors and uptime commands via SSH | API - Node Status      | Process Data               | ## SSH Temperature Check Connects via SSH to read temperature sensors using the sensors command. Requires SSH Password credential configured with your Proxmox host details. |
| Process Data       | Code                  | Parses and analyzes data, computes stats | SSH - Get Sensors      | Generate Formatted Message  | ## Data Processing Analyzes all collected data: - Counts running/stopped VMs - Detects VMs stopped in last 15 minutes - Calculates resource usage percentages - Evaluates temperature thresholds - Formats uptime statistics Generates structured report with HTML formatting for Telegram |
| Generate Formatted Message | Code           | Creates HTML formatted message for Telegram | Process Data           | Send Telegram Report       |                                                                                                       |
| Send Telegram Report | Telegram             | Sends formatted report to Telegram chat | Generate Formatted Message | —                        | ## TELEGRAM CONFIGURATION 1. Create bot via @BotFather 2. Get bot token and add as credential 3. Chat ID already set in Set Variables node Messages use HTML parse mode for formatting |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Schedule Every 15min`  
   - Set interval to every 15 minutes.

3. **Add a Set node:**  
   - Name: `Set Variables`  
   - Add string fields for:  
     - `PROXMOX_IP` (e.g., "192.168.1.100")  
     - `PROXMOX_PORT` (e.g., "8006")  
     - `PROXMOX_NODE` (e.g., "pve")  
     - `TELEGRAM_CHAT_ID` (your Telegram chat ID)  
     - `PROXMOX_USER` (e.g., "root@pam")  
     - `PROXMOX_PASSWORD` (your Proxmox password)  
   - Connect `Schedule Every 15min` → `Set Variables`.

4. **Add an HTTP Request node:**  
   - Name: `Proxmox Login`  
   - Method: POST  
   - URL: `https://{{ $json.PROXMOX_IP }}:{{ $json.PROXMOX_PORT }}/api2/json/access/ticket`  
   - Content Type: Form-urlencoded  
   - Body parameters:  
     - `username`: `={{ $json.PROXMOX_USER }}`  
     - `password`: `={{ $json.PROXMOX_PASSWORD }}`  
   - Options: Allow unauthorized certificates (for self-signed SSL)  
   - Connect `Set Variables` → `Proxmox Login`.

5. **Add a Set node:**  
   - Name: `Prepare Auth`  
   - Extract fields from `Proxmox Login` output:  
     - `ticket` = `={{ $json.data.ticket }}`  
     - `csrf` = `={{ $json.data.CSRFPreventionToken }}`  
     - Pass through `PROXMOX_IP`, `PROXMOX_PORT`, `PROXMOX_NODE` from `Set Variables`.  
   - Connect `Proxmox Login` → `Prepare Auth`.

6. **Add an HTTP Request node:**  
   - Name: `API - VM List`  
   - Method: GET  
   - URL: `https://{{ $json.PROXMOX_IP }}:{{ $json.PROXMOX_PORT }}/api2/json/nodes/{{ $json.PROXMOX_NODE }}/qemu`  
   - Headers:  
     - `Cookie`: `PVEAuthCookie={{ $json.ticket }}`  
     - `CSRFPreventionToken`: `={{ $json.csrf }}`  
   - Allow unauthorized certificates.  
   - Connect `Prepare Auth` → `API - VM List`.

7. **Add an HTTP Request node:**  
   - Name: `API - Node Tasks`  
   - Method: GET  
   - URL: `https://{{ $json.PROXMOX_IP }}:{{ $json.PROXMOX_PORT }}/api2/json/nodes/{{ $json.PROXMOX_NODE }}/tasks?limit=100`  
   - Headers same as above, using auth from `Prepare Auth`.  
   - Connect `API - VM List` → `API - Node Tasks`.

8. **Add an HTTP Request node:**  
   - Name: `API - Node Status`  
   - Method: GET  
   - URL: `https://{{ $json.PROXMOX_IP }}:{{ $json.PROXMOX_PORT }}/api2/json/nodes/{{ $json.PROXMOX_NODE }}/status`  
   - Headers same as above.  
   - Connect `API - Node Tasks` → `API - Node Status`.

9. **Add an SSH node:**  
   - Name: `SSH - Get Sensors`  
   - Command:  
     ```
     # Temperature sensors
     sensors 2>/dev/null | grep -E 'Package|Core' || echo 'No temperature data available'

     # Detailed uptime
     uptime
     ```  
   - Credentials: Configure SSH password credentials for Proxmox host (host IP, user, password).  
   - Connect `API - Node Status` → `SSH - Get Sensors`.

10. **Add a Code node:**  
    - Name: `Process Data`  
    - Paste the JavaScript code that:  
      - Parses SSH output and API data  
      - Computes VM counts, resource usage, temperature warnings  
      - Formats uptime  
      - Returns a JSON summary object  
    - Connect `SSH - Get Sensors` → `Process Data`.

11. **Add a Code node:**  
    - Name: `Generate Formatted Message`  
    - JavaScript code to format the structured data into an HTML message with sections, emojis, and formatting tags.  
    - Connect `Process Data` → `Generate Formatted Message`.

12. **Add a Telegram node:**  
    - Name: `Send Telegram Report`  
    - Text: `={{ $json.message }}`  
    - Chat ID: `={{ $('Set Variables').first().json.TELEGRAM_CHAT_ID }}`  
    - Additional Fields:  
      - `parse_mode`: `HTML`  
      - `appendAttribution`: false  
    - Credentials: Configure Telegram Bot API credentials with your bot token.  
    - Connect `Generate Formatted Message` → `Send Telegram Report`.

13. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                             |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| Requires `lm-sensors` package installed on the Proxmox host to provide temperature sensor data.                  | Installation: `apt install lm-sensors` on Proxmox host                                    |
| Telegram bot creation instructions: use @BotFather to create the bot and obtain the API token.                   | https://core.telegram.org/bots#6-botfather                                                |
| The Proxmox API session ticket is valid for 15 minutes; workflow runs every 15 minutes to refresh tokens.        | Proxmox API documentation: https://pve.proxmox.com/pve-docs/api-viewer/index.html         |
| SSH credentials must be configured securely in n8n credentials; avoid plaintext passwords in workflow variables. | n8n credentials management documentation                                                  |
| The workflow uses HTML parse mode in Telegram messages to allow rich formatting with bold, italics, and code blocks. | Telegram message formatting: https://core.telegram.org/bots/api#formatting-options         |

---

**Disclaimer:**  
The provided content is exclusively from an automated workflow created with n8n, a workflow automation tool. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.