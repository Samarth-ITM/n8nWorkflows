Kubernetes RCA and alerting using Gemini, Loki, Prometheus, Slack

https://n8nworkflows.xyz/workflows/kubernetes-rca-and-alerting-using-gemini--loki--prometheus--slack-7953


# Kubernetes RCA and alerting using Gemini, Loki, Prometheus, Slack

### 1. Workflow Overview

This workflow automates the Root Cause Analysis (RCA) and alerting process for Kubernetes clusters by integrating data from multiple monitoring and logging sources: Gemini (AI-based analysis), Loki (logging), Prometheus (metrics), and Slack (alerting). It is designed to periodically gather cluster health data, analyze incidents such as pod readiness, restarts, and crash loops, and generate human-readable insights using Gemini AI to send actionable alerts to Slack channels.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Data Collection:** Periodic triggering and parallel queries to Loki, Prometheus, and SSH to gather logs and metrics about the Kubernetes cluster state.
- **1.2 Data Processing and Filtering:** Code nodes and conditionals to parse, transform, and filter data, including excluding irrelevant pods and mapping metrics into useful contextual information.
- **1.3 Data Merging and RCA Prompt Building:** Merging various processed data streams, building a comprehensive prompt for Gemini AI to analyze cluster state and suggest root causes.
- **1.4 AI Processing (Gemini):** Sending the RCA prompt to Gemini AI and receiving the analysis.
- **1.5 Documentation and Formatting:** Enhancing the AI output with Kubernetes documentation, formatting the final alert message.
- **1.6 Alert Dispatching:** Sending the formatted RCA alert message to Slack.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Data Collection

**Overview:**  
This block triggers the workflow on a schedule and concurrently executes multiple HTTP and SSH requests to collect metrics and logs from monitoring systems and Kubernetes nodes.

**Nodes Involved:**  
- Schedule Trigger1  
- Loki  
- SSH1  
- PromQL: Current endpoints  
- PromQL: Endpoints 5m ago  
- Pods Not Ready  
- Pod Restart Spike (last 5m)  
- Pods Pending  
- CrashLoopBackOff

**Node Details:**  

- **Schedule Trigger1**  
  - *Type:* scheduleTrigger  
  - *Role:* Initiates the workflow run at scheduled intervals (default or configured schedule not specified here).  
  - *Config:* No parameters configured, triggers all downstream nodes simultaneously.  
  - *Connections:* Outputs to Loki, PromQL queries, Pods Not Ready, CrashLoopBackOff, Pod Restart Spike, and Pods Pending.  
  - *Failures:* Misconfiguration or downtime in source systems will affect downstream nodes.

- **Loki**  
  - *Type:* HTTP Request  
  - *Role:* Queries Loki for logs relevant to cluster state.  
  - *Config:* HTTP request configured with Loki API endpoint and query parameters (exact query not shown).  
  - *Input:* Trigger from Schedule Trigger1.  
  - *Output:* Feeds SSH1 node.  
  - *Failures:* Authentication failure, network timeout, or malformed queries.

- **SSH1**  
  - *Type:* SSH  
  - *Role:* Executes remote commands on Kubernetes nodes for additional data collection or log retrieval.  
  - *Config:* SSH credentials and commands configured (not detailed).  
  - *Input:* Loki node output.  
  - *Output:* Condition node to exclude terminated pods.  
  - *Failures:* SSH connectivity issues, command errors.

- **PromQL: Current endpoints**  
  - *Type:* HTTP Request  
  - *Role:* Queries Prometheus for current endpoint metrics of Kubernetes services.  
  - *Config:* Prometheus HTTP API with PromQL query for current endpoints.  
  - *Input:* Schedule Trigger1.  
  - *Output:* Maps results into namespace/service.  
  - *Failures:* Query errors, Prometheus downtime.

- **PromQL: Endpoints 5m ago**  
  - *Type:* HTTP Request  
  - *Role:* Queries Prometheus for endpoint metrics from 5 minutes ago to detect changes.  
  - *Config:* PromQL query with time offset.  
  - *Input:* Schedule Trigger1.  
  - *Output:* Maps results into namespace/service1.  
  - *Failures:* Same as above.

- **Pods Not Ready**  
  - *Type:* HTTP Request  
  - *Role:* Queries Kubernetes API or Prometheus for pods that are not ready.  
  - *Config:* HTTP request to relevant API.  
  - *Input:* Schedule Trigger1.  
  - *Output:* Pods Not Ready State code node.  
  - *Failures:* API errors, data unavailability.

- **Pod Restart Spike (last 5m)**  
  - *Type:* HTTP Request  
  - *Role:* Detects spikes in pod restarts over the last 5 minutes.  
  - *Config:* Prometheus HTTP request with appropriate PromQL.  
  - *Input:* Schedule Trigger1.  
  - *Output:* Pod Restart Count State code node.  
  - *Failures:* Query failures, timeouts.

- **Pods Pending**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves pods currently in the pending state.  
  - *Config:* Kubernetes API or Prometheus query.  
  - *Input:* Schedule Trigger1.  
  - *Output:* Pods Pending State code node.  
  - *Failures:* API failures.

- **CrashLoopBackOff**  
  - *Type:* HTTP Request  
  - *Role:* Detects pods in CrashLoopBackOff state.  
  - *Config:* Query to Kubernetes or Prometheus.  
  - *Input:* Schedule Trigger1.  
  - *Output:* Termination State code node.  
  - *Failures:* Query or API failures.

---

#### 2.2 Data Processing and Filtering

**Overview:**  
Processes raw data from collection nodes, filters irrelevant information, maps metrics into contextual namespace/service info, and excludes terminated pods.

**Nodes Involved:**  
- If(Excludes Pods which are terminated already)  
- Loki Error Logs  
- Map Prometheus results into namespace/service  
- Map Prometheus results into namespace/service1  
- Pods Not Ready State  
- Pods Pending State  
- Termination State  
- Pod Restart Count State  
- Endpoints

**Node Details:**  

- **If(Excludes Pods which are terminated already)**  
  - *Type:* If condition  
  - *Role:* Filters out pods that have already been terminated from the SSH node data to avoid false positives.  
  - *Input:* SSH1 output.  
  - *Output:* Loki Error Logs node (true branch).  
  - *Edge cases:* Incorrect filtering may lead to missed alerts or noise.

- **Loki Error Logs**  
  - *Type:* Code  
  - *Role:* Processes the Loki logs to extract error-related information relevant for RCA.  
  - *Input:* If node output.  
  - *Output:* Merge2 node.  
  - *Failures:* Parsing errors, unexpected log formats.

- **Map Prometheus results into namespace/service** & **Map Prometheus results into namespace/service1**  
  - *Type:* Code  
  - *Role:* Transforms Prometheus query results to associate metrics with corresponding Kubernetes namespaces and services for context.  
  - *Input:* PromQL: Current endpoints and PromQL: Endpoints 5m ago outputs respectively.  
  - *Output:* Merge node.  
  - *Failures:* Mapping logic errors, missing labels.

- **Pods Not Ready State**  
  - *Type:* Code  
  - *Role:* Processes pods not ready data into a usable state object.  
  - *Input:* Pods Not Ready HTTP request output.  
  - *Output:* Merge2 node.  
  - *Failures:* Data format issues.

- **Pods Pending State**  
  - *Type:* Code  
  - *Role:* Processes pods pending data into structured form.  
  - *Input:* Pods Pending HTTP request output.  
  - *Output:* Merge2 node.  
  - *Failures:* Data inconsistencies.

- **Termination State**  
  - *Type:* Code  
  - *Role:* Processes CrashLoopBackOff data into termination state info.  
  - *Input:* CrashLoopBackOff HTTP request output.  
  - *Output:* Merge2 node.  
  - *Failures:* Incorrect state interpretation.

- **Pod Restart Count State**  
  - *Type:* Code  
  - *Role:* Processes restart spike data into counts per pod/namespace.  
  - *Input:* Pod Restart Spike HTTP request output.  
  - *Output:* Merge2 node.  
  - *Failures:* Data aggregation errors.

- **Endpoints**  
  - *Type:* Code  
  - *Role:* Combines endpoint data from merged Prometheus mappings for further RCA.  
  - *Input:* Merge node output.  
  - *Output:* Merge2 node.  
  - *Failures:* Data merging conflicts.

---

#### 2.3 Data Merging and RCA Prompt Building

**Overview:**  
Aggregates all processed data streams and builds a comprehensive prompt for Gemini AI to perform root cause analysis.

**Nodes Involved:**  
- Merge  
- Merge2  
- Build Prompt for Gemini

**Node Details:**  

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines mapped Prometheus current and historical endpoint data into a single stream.  
  - *Input:* Map Prometheus results into namespace/service and Map Prometheus results into namespace/service1.  
  - *Output:* Endpoints node.  
  - *Failures:* Synchronization issues if data streams are out of sync.

- **Merge2**  
  - *Type:* Merge  
  - *Role:* Merges all processed states: pods not ready, pending, termination states, Loki error logs, endpoints, pod restart counts into one dataset.  
  - *Input:* Outputs from Pods Not Ready State, Pods Pending State, Termination State, Loki Error Logs, Endpoints, Pod Restart Count State.  
  - *Output:* Build Prompt for Gemini.  
  - *Failures:* Data format mismatches.

- **Build Prompt for Gemini**  
  - *Type:* Code  
  - *Role:* Constructs a detailed text prompt combining all collected and processed cluster state data for Gemini AI to analyze.  
  - *Input:* Merge2 output.  
  - *Output:* Google Gemini1 node.  
  - *Failures:* Prompt formatting errors, incomplete data.

---

#### 2.4 AI Processing (Gemini)

**Overview:**  
Sends the RCA prompt to Gemini AI for natural language analysis, receiving insights and recommendations.

**Nodes Involved:**  
- Google Gemini1  
- Kubernetes Documentation

**Node Details:**  

- **Google Gemini1**  
  - *Type:* HTTP Request  
  - *Role:* Calls Gemini AI API with the prepared prompt to generate root cause analysis and suggestions.  
  - *Input:* Build Prompt for Gemini output.  
  - *Output:* Kubernetes Documentation node.  
  - *Config:* Requires Gemini API credentials and endpoint.  
  - *Failures:* API authentication failure, rate limits, malformed prompt.

- **Kubernetes Documentation**  
  - *Type:* Code  
  - *Role:* Enhances Gemini output by integrating Kubernetes documentation references or additional context for clarity.  
  - *Input:* Gemini response.  
  - *Output:* Formatting the Output to send to Slack.  
  - *Failures:* Code errors, missing documentation resources.

---

#### 2.5 Documentation and Formatting

**Overview:**  
Formats the enriched AI output into a user-friendly message suitable for Slack alerts.

**Nodes Involved:**  
- Formatting the Output to send to Slack  
- Batch

**Node Details:**  

- **Formatting the Output to send to Slack**  
  - *Type:* Code  
  - *Role:* Structures the final message text, adding formatting like markdown, sections, or emojis for readability in Slack.  
  - *Input:* Kubernetes Documentation node output.  
  - *Output:* Batch node.  
  - *Failures:* Formatting bugs causing unreadable messages.

- **Batch**  
  - *Type:* Code  
  - *Role:* Potentially batches multiple messages or prepares the payload for Slack API.  
  - *Input:* Formatting node output.  
  - *Output:* 📤 Send Alerts to Slack node.  
  - *Failures:* Payload size limits, batching logic issues.

---

#### 2.6 Alert Dispatching

**Overview:**  
Sends the final RCA alert message to Slack for immediate notification and action.

**Nodes Involved:**  
- 📤 Send Alerts to Slack

**Node Details:**  

- **📤 Send Alerts to Slack**  
  - *Type:* HTTP Request  
  - *Role:* Calls Slack API to post the formatted alert message to a specified channel or user.  
  - *Input:* Batch node output.  
  - *Config:* Slack OAuth2 credentials or webhook URL configured.  
  - *Failures:* Slack API errors, authentication failure, message rejection.

---

### 3. Summary Table

| Node Name                        | Node Type          | Functional Role                             | Input Node(s)                         | Output Node(s)                     | Sticky Note |
|---------------------------------|--------------------|--------------------------------------------|-------------------------------------|----------------------------------|-------------|
| Schedule Trigger1                | scheduleTrigger    | Periodic trigger to start data collection  | -                                   | Loki, PromQL: Current endpoints, PromQL: Endpoints 5m ago, Pods Not Ready, CrashLoopBackOff, Pod Restart Spike, Pods Pending |             |
| Loki                            | HTTP Request       | Query logs from Loki                        | Schedule Trigger1                    | SSH1                             |             |
| SSH1                            | SSH                | Execute remote commands on Kubernetes nodes| Loki                                | If(Excludes Pods which are terminated already)     |             |
| If(Excludes Pods which are terminated already) | If                | Filter out terminated pods                   | SSH1                                | Loki Error Logs                  |             |
| Loki Error Logs                 | Code               | Parse Loki logs for error info              | If                                 | Merge2                          |             |
| PromQL: Current endpoints       | HTTP Request       | Query current endpoints from Prometheus    | Schedule Trigger1                    | Map Prometheus results into namespace/service |             |
| PromQL: Endpoints 5m ago        | HTTP Request       | Query endpoints from 5 minutes ago          | Schedule Trigger1                    | Map Prometheus results into namespace/service1 |             |
| Map Prometheus results into namespace/service | Code               | Map Prometheus data to namespace/service    | PromQL: Current endpoints           | Merge                           |             |
| Map Prometheus results into namespace/service1 | Code               | Map Prometheus data (5m ago) to namespace/service | PromQL: Endpoints 5m ago            | Merge                           |             |
| Merge                          | Merge              | Combine current and historical endpoint data | Map Prometheus results into namespace/service, Map Prometheus results into namespace/service1 | Endpoints                      |             |
| Endpoints                     | Code               | Process merged endpoint data                 | Merge                              | Merge2                          |             |
| Pods Not Ready                 | HTTP Request       | Query pods not ready                          | Schedule Trigger1                    | Pods Not Ready State            |             |
| Pods Not Ready State           | Code               | Process pods not ready data                   | Pods Not Ready                     | Merge2                          |             |
| Pods Pending                  | HTTP Request       | Query pods pending                            | Schedule Trigger1                    | Pods Pending State             |             |
| Pods Pending State            | Code               | Process pods pending data                     | Pods Pending                      | Merge2                          |             |
| CrashLoopBackOff              | HTTP Request       | Query pods in CrashLoopBackOff state         | Schedule Trigger1                    | Termination State              |             |
| Termination State             | Code               | Process CrashLoopBackOff data                 | CrashLoopBackOff                  | Merge2                          |             |
| Pod Restart Spike (last 5m)   | HTTP Request       | Query pod restart spikes                      | Schedule Trigger1                    | Pod Restart Count State        |             |
| Pod Restart Count State       | Code               | Process pod restart spikes                     | Pod Restart Spike (last 5m)        | Merge2                          |             |
| Merge2                        | Merge              | Merge all processed pod and log state data   | Pods Not Ready State, Pods Pending State, Termination State, Loki Error Logs, Endpoints, Pod Restart Count State | Build Prompt for Gemini        |             |
| Build Prompt for Gemini       | Code               | Build AI prompt for RCA                        | Merge2                            | Google Gemini1                 |             |
| Google Gemini1                | HTTP Request       | Send prompt to Gemini AI and receive analysis | Build Prompt for Gemini            | Kubernetes Documentation       |             |
| Kubernetes Documentation      | Code               | Add Kubernetes doc references to AI output   | Google Gemini1                    | Formatting the Output to send to Slack |             |
| Formatting the Output to send to Slack | Code               | Format final alert message for Slack          | Kubernetes Documentation          | Batch                          |             |
| Batch                        | Code               | Batch messages or prepare payload             | Formatting the Output to send to Slack | 📤 Send Alerts to Slack         |             |
| 📤 Send Alerts to Slack         | HTTP Request       | Post RCA alert message to Slack                | Batch                             | -                              |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Schedule Trigger1` with default or specific cron settings to run at desired intervals.

2. **Add HTTP Request nodes** for data sources:  
   - `Loki`: Configure HTTP request with Loki API endpoint and query to fetch relevant Kubernetes logs.  
   - `PromQL: Current endpoints`: Configure with Prometheus API and PromQL query for current endpoints.  
   - `PromQL: Endpoints 5m ago`: Configure with PromQL query offset 5 minutes.  
   - `Pods Not Ready`: Configure HTTP request to retrieve pods not ready from Kubernetes or Prometheus.  
   - `Pod Restart Spike (last 5m)`: Configure to query pod restart spikes from Prometheus.  
   - `Pods Pending`: Configure to query pods in pending state.  
   - `CrashLoopBackOff`: Configure to query pods in CrashLoopBackOff state.

3. **Create an SSH node** named `SSH1` with SSH credentials to Kubernetes nodes; configure commands to retrieve necessary diagnostics or logs.

4. Connect `Schedule Trigger1` output to all above nodes accordingly.

5. Connect `Loki` output to `SSH1` node.

6. Add an **If node** `If(Excludes Pods which are terminated already)` to filter out terminated pods based on SSH1 output; configure condition expression accordingly.

7. Connect the If node’s true output to a **Code node** `Loki Error Logs` to parse and extract error logs.

8. Add **Code nodes**:  
   - `Map Prometheus results into namespace/service` connected to `PromQL: Current endpoints`.  
   - `Map Prometheus results into namespace/service1` connected to `PromQL: Endpoints 5m ago`.  
   - `Pods Not Ready State` connected to `Pods Not Ready`.  
   - `Pods Pending State` connected to `Pods Pending`.  
   - `Termination State` connected to `CrashLoopBackOff`.  
   - `Pod Restart Count State` connected to `Pod Restart Spike (last 5m)`.

9. Create a **Merge node** `Merge` to combine outputs of the two Prometheus mapping nodes.

10. Connect `Merge` to a **Code node** `Endpoints` to process combined endpoint data.

11. Create a **Merge node** `Merge2` to merge outputs of:  
    - `Pods Not Ready State`, `Pods Pending State`, `Termination State`, `Loki Error Logs`, `Endpoints`, and `Pod Restart Count State`.

12. Connect `Merge2` to a **Code node** `Build Prompt for Gemini` to compose a detailed prompt combining all collected data.

13. Add an HTTP Request node `Google Gemini1` configured with Gemini API credentials and endpoint; connect it to `Build Prompt for Gemini`.

14. Add a **Code node** `Kubernetes Documentation` connected to `Google Gemini1` to enrich AI output with Kubernetes references.

15. Add a **Code node** `Formatting the Output to send to Slack` connected to `Kubernetes Documentation` to format the message.

16. Add a **Code node** `Batch` connected to the formatter to prepare the message payload.

17. Add an HTTP Request node `📤 Send Alerts to Slack` configured with Slack credentials (OAuth2 or webhook URL) and message post endpoint; connect it to `Batch`.

18. Verify all connections and set appropriate error handling, retries, and timeouts.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow integrates Google Gemini AI for Root Cause Analysis of Kubernetes cluster incidents. | AI component enhances automated troubleshooting. |
| Slack is used as the alerting channel for operational notifications.         | Real-time alerts support DevOps response.       |
| Prometheus and Loki are the primary monitoring and logging backends.         | Common open-source Kubernetes observability stack. |
| SSH node allows running custom commands on Kubernetes nodes for deeper insights. | Useful for gathering logs or diagnostics not exposed via APIs. |
| Ensure all API credentials and tokens for Loki, Prometheus, Gemini, and Slack are securely stored in n8n credentials. | Security best practice.                           |
| The workflow’s modular design supports adding more metrics or logs by extending the data collection and processing blocks. | Extensible for future monitoring needs.          |

---

**Disclaimer:**  
The text provided is exclusively generated from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.