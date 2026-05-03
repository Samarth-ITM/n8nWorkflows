Send deduplicated Kubernetes(EKS/GKE/AKS) error logs from Grafana Loki to Slack

https://n8nworkflows.xyz/workflows/send-deduplicated-kubernetes-eks-gke-aks--error-logs-from-grafana-loki-to-slack-7021


# Send deduplicated Kubernetes(EKS/GKE/AKS) error logs from Grafana Loki to Slack

---

### 1. Workflow Overview

This workflow is designed to monitor Kubernetes error logs collected via Grafana Loki and send deduplicated alerts to a designated Slack channel. It targets DevOps teams or SREs managing Kubernetes clusters (EKS, GKE, AKS) who want timely, concise notifications about unique error events without being overwhelmed by repeated log noise.

The workflow is logically structured into these functional blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every 5 minutes to collect recent errors.
- **1.2 Loki Query:** Fetches error logs from Loki within the last 10 minutes using a specified query.
- **1.3 Log Extraction:** Parses raw Loki log entries to extract structured metadata and relevant log content.
- **1.4 Deduplication:** Filters out duplicate logs in the current batch to avoid repeated Slack alerts.
- **1.5 Slack Notification:** Sends formatted Slack messages for each unique error log.

Each block corresponds to one or more n8n nodes connected sequentially to ensure smooth data flow from querying logs to posting alerts.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow every 5 minutes, ensuring periodic log checks.

- **Nodes Involved:**  
  - 🕒 Every 5 Min Trigger

- **Node Details:**  
  - **Name:** 🕒 Every 5 Min Trigger  
  - **Type:** Schedule Trigger  
  - **Configuration:**  
    - Runs on a fixed interval every 5 minutes (configurable)  
  - **Key Expressions:** None  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Connects to "📥 Query Loki for Error Logs" node  
  - **Version Requirements:** Standard for n8n schedule trigger node  
  - **Potential Failures:** Workflow won’t trigger if n8n is not running or time drift occurs.  
  - **Sticky Note:** "Runs every 5 minutes. You can adjust this interval as needed."

---

#### 1.2 Loki Query

- **Overview:**  
  Sends an HTTP query to the Loki API to retrieve error logs from the last 10 minutes filtered by a regex on the log content.

- **Nodes Involved:**  
  - 📥 Query Loki for Error Logs

- **Node Details:**  
  - **Name:** 📥 Query Loki for Error Logs  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - Method: GET (default implied)  
    - URL: `http://loki-gateway.monitoring.svc.cluster.local:8080/loki/api/v1/query_range` (internal Loki service endpoint)  
    - Query Parameters:  
      - `query`: `{namespace="monitoring"} |= "error"` (filters logs in the monitoring namespace containing "error")  
      - `start`: 10 minutes ago, converted to nanoseconds (`(Date.now() - 10 * 60 * 1000) * 1000000`)  
      - `end`: current time in nanoseconds (`Date.now() * 1000000`)  
    - Sends query parameters as URL params  
  - **Key Expressions:**  
    - Date calculations for start/end parameters using JavaScript expressions  
  - **Input Connections:** From schedule trigger  
  - **Output Connections:** To "🧹 Extract Log Fields"  
  - **Version Requirements:** Compatible with n8n HTTP Request node v4+  
  - **Potential Failures:**  
    - Loki service unreachable/network errors  
    - Incorrect query parameters causing empty or malformed responses  
    - Timezone or timestamp miscalculations affecting log range  
  - **Sticky Note:** "Sends an HTTP request to Loki with last 10 minutes range, error regex, and customizable namespaces."

---

#### 1.3 Log Extraction

- **Overview:**  
  Parses the Loki API response to extract structured log fields such as pod name, namespace, container name, node, timestamp, and cleans the log message content. Filters out non-error levels like debug or info.

- **Nodes Involved:**  
  - 🧹 Extract Log Fields

- **Node Details:**  
  - **Name:** 🧹 Extract Log Fields  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Custom JS code iterates over Loki results array  
    - Extracts log content by parsing `extracteddata="map[...]content:..."` pattern if present  
    - Falls back to raw log line if extraction fails but contains log level  
    - Skips logs with level debug or info (case-insensitive)  
    - Normalizes timestamps from nanoseconds to ISO string  
    - Assigns default "unknown" if metadata fields missing  
  - **Key Expressions:** Complex regex matching and string replacement to clean logs  
  - **Input Connections:** From Loki query node  
  - **Output Connections:** To "🧠 Remove Duplicate Alerts"  
  - **Version Requirements:** Requires n8n code node supporting ES6 syntax (standard)  
  - **Potential Failures:**  
    - Parsing errors if Loki response structure changes  
    - Regex failures or unexpected log formats causing empty outputs  
  - **Sticky Note:** "Parses log entries, extracts metadata, skips null/empty logs automatically."

---

#### 1.4 Deduplication

- **Overview:**  
  Removes duplicate log entries in the current batch by normalizing log messages (masking timestamps, numbers, quoted strings) and filtering repeated logs per pod to reduce Slack noise.

- **Nodes Involved:**  
  - 🧠 Remove Duplicate Alerts

- **Node Details:**  
  - **Name:** 🧠 Remove Duplicate Alerts  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Maintains a Set of normalized log keys combining pod name and normalized log message  
    - Normalization replaces timestamps, numbers with placeholders and masks quoted strings  
    - Deduplicates logs by checking if normalized key already seen  
  - **Key Expressions:** Normalization function with regex replacements and toLowerCase  
  - **Input Connections:** From log extraction node  
  - **Output Connections:** To "📤 Send Alerts to Slack"  
  - **Version Requirements:** Standard code node supporting ES6  
  - **Potential Failures:**  
    - Logic errors in normalization causing false positives/negatives  
    - Large batch sizes causing memory overhead (unlikely in 5-minute window)  
  - **Sticky Note:** "Removes repeated alerts in the current batch to avoid Slack spam."

---

#### 1.5 Slack Notification

- **Overview:**  
  Sends a formatted message for each unique error log to a specified Slack channel using Slack’s API with Bearer token authentication.

- **Nodes Involved:**  
  - 📤 Send Alerts to Slack

- **Node Details:**  
  - **Name:** 📤 Send Alerts to Slack  
  - **Type:** HTTP Request  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://slack.com/api/chat.postMessage`  
    - Authentication: HTTP Bearer Auth with Slack token credential  
    - Headers: Content-Type set to `application/json`  
    - Body Parameters (JSON):  
      - `channel`: `#k8s-alerts` (Slack channel name, configurable)  
      - `text`: Formatted message containing emoji 🚨 and metadata fields: Pod, Namespace, Container, Node, Timestamp, and the log message wrapped in triple backticks  
  - **Key Expressions:** Uses n8n expression syntax to inject JSON fields dynamically into message text  
  - **Input Connections:** From deduplication node  
  - **Output Connections:** None (end node)  
  - **Version Requirements:** HTTP Request node supporting bearer token and JSON body  
  - **Potential Failures:**  
    - Invalid or expired Slack token resulting in 401 Unauthorized  
    - Slack API rate limiting or connectivity issues  
    - Improper channel name causing message post failure  
  - **Sticky Note:** "Posts alerts to Slack with metadata and formatted error logs. Ensure Slack token and channel name are set."

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role         | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                              |
|---------------------------|----------------------|------------------------|---------------------------|--------------------------|--------------------------------------------------------------------------------------------------------|
| 🕒 Every 5 Min Trigger      | Schedule Trigger     | Trigger workflow       | -                         | 📥 Query Loki for Error Logs | Runs every 5 minutes. You can adjust this interval as needed.                                          |
| 📥 Query Loki for Error Logs | HTTP Request         | Fetch error logs       | 🕒 Every 5 Min Trigger     | 🧹 Extract Log Fields      | Sends an HTTP request to Loki with last 10 minutes range, error regex, and customizable namespaces.    |
| 🧹 Extract Log Fields       | Code (JavaScript)    | Parse and filter logs  | 📥 Query Loki for Error Logs | 🧠 Remove Duplicate Alerts | Parses log entries, extracts metadata, skips null/empty logs automatically.                            |
| 🧠 Remove Duplicate Alerts  | Code (JavaScript)    | Deduplicate logs       | 🧹 Extract Log Fields       | 📤 Send Alerts to Slack    | Removes repeated alerts in the current batch to avoid Slack spam.                                      |
| 📤 Send Alerts to Slack     | HTTP Request         | Send Slack messages    | 🧠 Remove Duplicate Alerts  | -                        | Posts alerts to Slack with metadata and formatted error logs. Ensure Slack token and channel name are set. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: 🕒 Every 5 Min Trigger  
   - Set interval to every 5 minutes.

2. **Create HTTP Request Node for Loki Query**  
   - Type: HTTP Request  
   - Name: 📥 Query Loki for Error Logs  
   - Method: GET  
   - URL: `http://loki-gateway.monitoring.svc.cluster.local:8080/loki/api/v1/query_range`  
   - Add Query Parameters:  
     - `query` = `{namespace="monitoring"} |= "error"` (customize namespace or filter as needed)  
     - `start` = `={{ (Date.now() - 10 * 60 * 1000) * 1000000 }}` (10 minutes ago, nanoseconds)  
     - `end` = `={{ Date.now() * 1000000 }}` (current time, nanoseconds)  
   - Connect output of schedule trigger → this node.

3. **Create Code Node for Log Extraction**  
   - Type: Code (JavaScript)  
   - Name: 🧹 Extract Log Fields  
   - Paste provided JS code that:  
     - Iterates Loki results  
     - Extracts 'content:' from `extracteddata` if present  
     - Falls back to raw log line with 'level=' present  
     - Filters out debug/info level logs  
     - Extracts pod, namespace, container, node, and ISO timestamp  
   - Connect output of Loki query → this node.

4. **Create Code Node for Deduplication**  
   - Type: Code (JavaScript)  
   - Name: 🧠 Remove Duplicate Alerts  
   - Paste provided JS code that:  
     - Normalizes logs by removing timestamps, masking numbers and quoted strings  
     - Deduplicates logs per pod by normalized content  
   - Connect output of log extraction → this node.

5. **Create HTTP Request Node for Slack Notification**  
   - Type: HTTP Request  
   - Name: 📤 Send Alerts to Slack  
   - Method: POST  
   - URL: `https://slack.com/api/chat.postMessage`  
   - Authentication: HTTP Bearer Auth (configure Slack token credential)  
   - Headers: Add `Content-Type: application/json`  
   - Body Type: JSON  
   - Body Parameters:  
     - `channel`: `#k8s-alerts` (adjust channel name as needed)  
     - `text`: Use n8n expression to build message:  
       ```
       🚨 *Kubernetes Error Detected* *Pod:* {{$json["pod"]}} *Namespace:* {{$json["namespace"]}} *Container:* {{$json["container"]}} *Node:* {{$json["node"]}} *Timestamp:* {{$json["timestamp"]}} *Log:* ```{{$json["log"]}}```
       ```  
   - Connect output of deduplication node → this node.

6. **Save and Activate Workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| 🔔 Loki Error Logs to Slack (Deduplicated): Queries Loki every 5 minutes, filters, deduplicates, and sends to Slack. | Overview sticky note at workflow start.                        |
| Customize regex or deduplication logic by editing the Loki query node or code nodes accordingly.| See related sticky notes near respective nodes.               |
| Slack token must have `chat:write` permission and the bot/user must be a member of the target channel.| Slack API documentation: https://api.slack.com/methods/chat.postMessage |
| The Loki endpoint URL and namespace filter must match your Kubernetes cluster monitoring setup.  | Modify for EKS/GKE/AKS environments as needed.                |
| Message formatting uses triple backticks for code block display in Slack to improve readability. | Slack message formatting best practices.                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.