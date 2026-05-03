Kubernetes Deployment & Pod Monitoring with Telegram Alerts

https://n8nworkflows.xyz/workflows/kubernetes-deployment---pod-monitoring-with-telegram-alerts-9613


# Kubernetes Deployment & Pod Monitoring with Telegram Alerts

### 1. Workflow Overview

This workflow automates monitoring of Kubernetes cluster workloads, focusing on deployments and pods within a specified namespace. It periodically collects data about pods and deployments, processes this data to assess the health and readiness of workloads, generates a detailed markdown report, and sends Telegram alerts if any workload has zero ready pods. The workflow includes logical blocks for scheduling, Kubernetes API interaction, data processing/report generation, alert condition evaluation, notification delivery, and report saving.

Logical blocks:

- **1.1 Schedule Trigger**: Initiates periodic execution of the workflow.
- **1.2 Kubernetes Configuration Setup**: Prepares Kubernetes access credentials and target namespace.
- **1.3 Data Collection**: Fetches pods and deployments data from the Kubernetes cluster.
- **1.4 Processing & Report Generation**: Parses and analyzes collected data, detects issues, composes a markdown status report.
- **1.5 Alert Evaluation**: Determines if alerts should be sent based on detected issues.
- **1.6 Telegram Notification**: Sends alerts to a Telegram chat if issues are present.
- **1.7 Report Saving**: Saves the generated markdown report as a timestamped file locally.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Triggers the workflow at regular time intervals to automate continuous monitoring.
- **Nodes Involved:** Schedule Trigger
- **Node Details:**
  - Type: Schedule Trigger
  - Configuration: Executes every minute (interval set to 1 minute).
  - Expressions/Variables: None.
  - Inputs: None (trigger node).
  - Outputs: Passes empty initial data to next node.
  - Edge Cases: If execution is delayed, intervals may drift; no authentication involved.
  - Version: 1

#### 1.2 Kubernetes Configuration Setup

- **Overview:** Prepares Kubernetes access by setting kubeconfig content and namespace for API queries.
- **Nodes Involved:** Kubeconfig Setup
- **Node Details:**
  - Type: Code node (JavaScript)
  - Configuration: 
    - User must paste raw kubeconfig content inside the node’s JS code.
    - Namespace is set to 'production' by default.
  - Key Variables:
    - `kubeconfig`: The raw kubeconfig YAML content as a string.
    - `namespace`: Target Kubernetes namespace.
  - Inputs: From Schedule Trigger.
  - Outputs: JSON containing `kubeconfig` and `namespace`.
  - Edge Cases: Missing or invalid kubeconfig will cause downstream failures; user must update credentials/token correctly.
  - Version: 2

#### 1.3 Data Collection

- **Overview:** Concurrently retrieves pods and deployments JSON data from Kubernetes using kubectl executed inside the node.
- **Nodes Involved:** Get Pods, Get Deployments
- **Node Details:**

  - **Get Pods**
    - Type: Execute Command
    - Configuration:
      - Installs curl, downloads kubectl binary v1.34.0 for Linux amd64.
      - Writes kubeconfig content to a temporary file.
      - Executes `kubectl get pods` with JSON output in the specified namespace.
      - Cleans up temporary kubeconfig file.
    - Inputs: From Kubeconfig Setup.
    - Outputs: JSON-formatted pods data in `stdout`.
    - Edge Cases: 
      - Network issues downloading kubectl.
      - Permission or execution errors on temporary file.
      - Invalid kubeconfig or token causing kubectl authorization errors.
      - Kubernetes API server unavailability or namespace errors.
    - Version: 1

  - **Get Deployments**
    - Type: Execute Command
    - Configuration:
      - Same setup as Get Pods but runs `kubectl get deployments` with JSON output.
    - Inputs: From Kubeconfig Setup.
    - Outputs: JSON-formatted deployments data in `stdout`.
    - Edge Cases: Similar to Get Pods, plus any deployment-specific API errors.

- Both nodes run in parallel and send data downstream.

#### 1.4 Processing & Report Generation

- **Overview:** Parses pod and deployment data, correlates pods to workload owners, detects alert conditions, and generates a detailed markdown report with workload status and alerts.
- **Nodes Involved:** Process & Generate Report
- **Node Details:**
  - Type: Code node (JavaScript)
  - Configuration:
    - Accepts inputs from both Get Pods and Get Deployments nodes.
    - Parses JSON output from `stdout` of each input.
    - Groups pods by owner references (Deployments via ReplicaSet, DaemonSets, StatefulSets, Nodes).
    - Computes readiness statistics and detects workloads with zero ready pods.
    - Generates a markdown report including:
      - Summary counts (deployments, other workloads, pods).
      - Detailed sections per deployment and other workloads with pod readiness.
      - Standalone pods section.
      - Alerts section listing problematic workloads.
    - Converts markdown report to base64-encoded binary data with metadata for saving.
  - Key Variables:
    - `podsData`, `deploymentsData`: Parsed Kubernetes API data.
    - `deploymentStatus`, `otherWorkloads`, `standalonePods`: Data structures for grouping.
    - `alerts`: Array of alert objects with workload name, kind, and issue description.
    - Outputs JSON including `markdown`, `hasAlerts` boolean, `alerts` array, and counts, plus binary markdown data.
  - Inputs: From Get Pods and Get Deployments.
  - Outputs: JSON + binary data to Has Alerts? node.
  - Edge Cases:
    - JSON parsing errors if kubectl output is malformed.
    - Missing or incomplete ownership metadata in pods.
    - Edge cases where pods belong to unknown or custom owners.
    - Potential for empty workload lists if namespace is empty or misconfigured.
  - Version: 2

#### 1.5 Alert Evaluation

- **Overview:** Checks if the processed data includes any alerts indicating zero ready pods, branching workflow accordingly.
- **Nodes Involved:** Has Alerts?
- **Node Details:**
  - Type: If node (Boolean condition)
  - Configuration:
    - Condition: `{{$json.hasAlerts}} == true`
  - Inputs: From Process & Generate Report.
  - Outputs:
    - True branch: workflow continues to send Telegram alert.
    - False branch: skips alert, proceeds to save report.
  - Edge Cases: If `hasAlerts` is undefined or non-boolean, may cause branch misrouting.
  - Version: 1

#### 1.6 Telegram Notification

- **Overview:** Sends a formatted alert message with workload issues and full report to a predefined Telegram chat.
- **Nodes Involved:** Send Telegram Alert
- **Node Details:**
  - Type: Telegram node
  - Configuration:
    - Text message composed in Markdown, includes:
      - Alert header with namespace.
      - List of alerts formatted as bullet points.
      - Full markdown report included after separator.
    - Chat ID must be replaced with actual Telegram chat ID.
    - Parse mode set to Markdown for formatting.
  - Inputs: True branch from Has Alerts? node.
  - Outputs: Passes data downstream to Save Report.
  - Credentials: Telegram API credentials configured externally with bot token.
  - Edge Cases:
    - Invalid chat ID or bot token results in Telegram API errors.
    - Markdown formatting errors could cause message rejection.
    - Large message size may exceed Telegram limits.
  - Version: 1

#### 1.7 Report Saving

- **Overview:** Saves the generated markdown report as a timestamped file on local disk or configured storage.
- **Nodes Involved:** Save Report
- **Node Details:**
  - Type: Write Binary File
  - Configuration:
    - Filename pattern: `k8s-report-YYYY-MM-DD-HHmmss.md` based on execution timestamp.
    - Reads base64-encoded markdown data from input.
  - Inputs: Either from Telegram Alert node (on alert) or directly from Has Alerts? false branch.
  - Outputs: None (terminal node).
  - Edge Cases:
    - Write permissions must be granted to target folder.
    - Filename collisions unlikely due to timestamp but possible if run concurrently within same second.
  - Version: 1

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                                | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                      |
|-----------------------|----------------------|-----------------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger     | Initiates periodic workflow execution          | None                          | Kubeconfig Setup               | ## Kubernetes Monitoring Workflow\n\nAutomatically monitors your K8s cluster and sends Telegram alerts when workloads have zero ready pods.\n\nReports are saved as markdown files for every execution. |
| Kubeconfig Setup       | Code                 | Sets kubeconfig content and namespace          | Schedule Trigger              | Get Pods, Get Deployments      | ## CONFIGURATION REQUIRED\n\n1. Paste your kubeconfig content\n2. Set target namespace (default: 'production')\n\nThe workflow will automatically download kubectl during execution.                |
| Get Pods              | Execute Command       | Fetches pods JSON data from Kubernetes         | Kubeconfig Setup              | Process & Generate Report       | ## Data Collection\n\nBoth nodes run in parallel to fetch:\n- All pods\n- All deployments\n\nfrom the specified namespace                                                   |
| Get Deployments        | Execute Command       | Fetches deployments JSON data                   | Kubeconfig Setup              | Process & Generate Report       | ## Data Collection\n\nBoth nodes run in parallel to fetch:\n- All pods\n- All deployments\n\nfrom the specified namespace                                                   |
| Process & Generate Report | Code               | Parses data, detects alerts, generates report  | Get Pods, Get Deployments     | Has Alerts?                    | ## Processing & Alert Detection\n\nGroups pods by owner (Deployment, DaemonSet, StatefulSet, Node)\n\nDetects alerts: workloads with 0 ready pods\n\nGenerates comprehensive markdown report |
| Has Alerts?            | If                   | Branches based on presence of alerts            | Process & Generate Report     | Send Telegram Alert (true), Save Report (false) |                                                                                                                         |
| Send Telegram Alert    | Telegram              | Sends alert message to Telegram chat            | Has Alerts? (true branch)     | Save Report                   | ## TELEGRAM CONFIGURATION\n\n1. Create bot via @BotFather\n2. Get bot token & add as credential\n3. Replace YOUR_TELEGRAM_CHAT_ID with your actual chat ID\n\nFind chat ID: message your bot, then visit:\nhttps://api.telegram.org/bot<TOKEN>/getUpdates |
| Save Report            | Write Binary File     | Saves markdown report file locally               | Has Alerts? (false branch), Send Telegram Alert | None                          | ## Report Output\n\nSaves markdown report with timestamp:\nk8s-report-YYYY-MM-DD-HHmmss.md\n\nExecutes on every run regardless of alert status                                        |
| Sticky Note            | Sticky Note           | Workflow description note                        | None                          | None                          | ## Kubernetes Monitoring Workflow\n\nAutomatically monitors your K8s cluster and sends Telegram alerts when workloads have zero ready pods.\n\nReports are saved as markdown files for every execution. |
| Sticky Note1           | Sticky Note           | Configuration instructions                       | None                          | None                          | ## CONFIGURATION REQUIRED\n\n1. Paste your kubeconfig content\n2. Set target namespace (default: 'production')\n\nThe workflow will automatically download kubectl during execution.                |
| Sticky Note2           | Sticky Note           | Notes on data collection nodes                   | None                          | None                          | ## Data Collection\n\nBoth nodes run in parallel to fetch:\n- All pods\n- All deployments\n\nfrom the specified namespace                                                   |
| Sticky Note3           | Sticky Note           | Notes on processing and alert detection          | None                          | None                          | ## Processing & Alert Detection\n\nGroups pods by owner (Deployment, DaemonSet, StatefulSet, Node)\n\nDetects alerts: workloads with 0 ready pods\n\nGenerates comprehensive markdown report |
| Sticky Note4           | Sticky Note           | Telegram bot setup instructions                   | None                          | None                          | ## TELEGRAM CONFIGURATION\n\n1. Create bot via @BotFather\n2. Get bot token & add as credential\n3. Replace YOUR_TELEGRAM_CHAT_ID with your actual chat ID\n\nFind chat ID: message your bot, then visit:\nhttps://api.telegram.org/bot<TOKEN>/getUpdates |
| Sticky Note5           | Sticky Note           | Report saving information                         | None                          | None                          | ## Report Output\n\nSaves markdown report with timestamp:\nk8s-report-YYYY-MM-DD-HHmmss.md\n\nExecutes on every run regardless of alert status                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**
   - Type: Schedule Trigger
   - Set interval to run every 1 minute.
   - No input required.

2. **Create Code node "Kubeconfig Setup"**
   - Connect Schedule Trigger → Kubeconfig Setup.
   - Paste your Kubernetes kubeconfig YAML content inside the node’s JavaScript code.
   - Set target namespace variable (default: `'production'`).
   - Output JSON with keys `kubeconfig` (string) and `namespace` (string).

3. **Create Execute Command node "Get Pods"**
   - Connect Kubeconfig Setup → Get Pods.
   - Command script:
     - Install `curl` via apk.
     - Write kubeconfig content from input to temporary file `/tmp/kubeconfig-$RANDOM.yaml`.
     - Download kubectl binary version v1.34.0 for Linux amd64.
     - Make kubectl executable.
     - Run `kubectl --kubeconfig=$KUBECONFIG_PATH get pods -n <namespace> -o json`.
     - Remove temporary kubeconfig file.
   - Use expressions to insert kubeconfig and namespace from input JSON.
   - No credentials needed since kubectl uses token from kubeconfig.

4. **Create Execute Command node "Get Deployments"**
   - Connect Kubeconfig Setup → Get Deployments.
   - Use the same command script as Get Pods but replace `get pods` with `get deployments`.
   - Use the same kubeconfig and namespace expression.

5. **Create Code node "Process & Generate Report"**
   - Connect Get Pods → Process & Generate Report.
   - Connect Get Deployments → Process & Generate Report.
   - Code logic:
     - Parse JSON from both nodes.
     - Group pods by owner references (deployments, daemonsets, statefulsets).
     - Compute readiness and restart counts.
     - Detect workloads with zero ready pods and record alerts.
     - Generate markdown report string with sections for deployments, other workloads, standalone pods, and alerts.
     - Convert markdown to base64-encoded binary data with metadata.
   - Output JSON properties: `markdown`, `hasAlerts` (boolean), `alerts` array, counts, namespace.
   - Output binary property: base64 markdown data for saving.

6. **Create If node "Has Alerts?"**
   - Connect Process & Generate Report → Has Alerts?.
   - Condition: Check if `{{$json.hasAlerts}}` equals `true`.
   - True branch proceeds to Send Telegram Alert.
   - False branch proceeds to Save Report.

7. **Create Telegram node "Send Telegram Alert"**
   - Connect Has Alerts? (true) → Send Telegram Alert.
   - Configure Telegram API credentials with bot token.
   - Set chat ID to your Telegram chat ID.
   - Message text (Markdown):
     ```
     🚨 *Kubernetes Alert*

     *Namespace:* {{$json.namespace}}

     {{$json.alerts.map(a => `⚠️ *${a.workload}*: ${a.issue}`).join('\n')}}

     ---

     {{$json.markdown}}
     ```
   - Enable Markdown parse mode.

8. **Create Write Binary File node "Save Report"**
   - Connect Has Alerts? (false) → Save Report.
   - Connect Send Telegram Alert → Save Report (to always save report).
   - File name expression: `k8s-report-{{$now.format('YYYY-MM-DD-HHmmss')}}.md`
   - Data property: binary data from previous node (`data`).
   - Ensure write permissions are granted to target directory.

9. **Add Sticky Notes (optional)**
   - Add notes explaining workflow purpose, configuration, data collection, processing, Telegram setup, and report saving as per original.

10. **Activate and Test Workflow**
    - Paste valid kubeconfig with user token.
    - Set correct Telegram chat ID.
    - Run manually or wait for schedule trigger.
    - Verify report generation and Telegram alert on workload issues.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Kubernetes Monitoring Workflow: Automatically monitors your K8s cluster and sends Telegram alerts when workloads have zero ready pods. Reports saved as markdown files. | General workflow purpose.                                                                                |
| Configuration required: Paste your kubeconfig content and set target namespace (default: 'production'). The workflow downloads kubectl automatically during execution. | Important setup step to access Kubernetes API.                                                          |
| Data collection runs in parallel: fetching both pods and deployments JSON from Kubernetes namespace.  | Workflow efficiency note.                                                                                |
| Processing groups pods by owner references and detects alerts for workloads with zero ready pods. Generates a comprehensive markdown report. | Core logic for alert detection and reporting.                                                           |
| Telegram configuration: Create bot via @BotFather, get bot token, create credential in n8n, replace YOUR_TELEGRAM_CHAT_ID with actual chat ID. Find chat ID by messaging bot then calling Telegram API getUpdates. | Telegram integration setup instructions.                                                                |
| Report output: Saves markdown report with timestamped file name on every execution regardless of alert status. | Report persistence and audit trail.                                                                     |
| Telegram API reference for chat ID retrieval: https://api.telegram.org/bot<TOKEN>/getUpdates          | Useful link for Telegram chat ID discovery.                                                             |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.