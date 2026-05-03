Monitor Kubernetes Services & Pods with Prometheus and Send Alerts to Slack

https://n8nworkflows.xyz/workflows/monitor-kubernetes-services---pods-with-prometheus-and-send-alerts-to-slack-7665


# Monitor Kubernetes Services & Pods with Prometheus and Send Alerts to Slack

### 1. Workflow Overview

This workflow monitors Kubernetes cluster health by querying Prometheus metrics related to services and pods, normalizing and aggregating this data, then sending summarized alerts to a Slack channel every 5 minutes. It targets DevOps teams and Kubernetes administrators who need real-time visibility into pod statuses, service endpoint changes, and critical issues such as pod restarts, pending pods, or pods stuck in failed states.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Periodic execution every 5 minutes to initiate monitoring.
- **1.2 Prometheus Queries for Services and Pods**: Multiple HTTP Request nodes query Prometheus for:
  - Current and 5-minute-ago service endpoints
  - Pods in Running state
  - Containers not ready
  - Pods in CrashLoopBackOff or terminated with error
  - Pods with restart spikes in last 5 minutes
  - Pods in Pending state
- **1.3 Data Mapping and Merging**: Code nodes map raw Prometheus results into structured namespace/service or pod-related objects. Merge nodes combine datasets.
- **1.4 Normalization and Aggregation**: Code nodes clean, classify, and normalize various pod conditions into a consistent JSON schema.
- **1.5 Slack Message Formatting**: A code node summarizes the collected data into a human-readable Slack message with grouped alerts and emojis.
- **1.6 Slack Notification**: Sends the formatted alert message to a Slack channel using a webhook with bearer authentication.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow every 5 minutes to ensure continuous monitoring.
  
- **Nodes Involved:**  
  - 🕒 Every 5 Min Trigger

- **Node Details:**  
  - Type: Schedule Trigger  
  - Configuration: Interval set to every 5 minutes  
  - Input: None (trigger node)  
  - Output: Triggers parallel downstream Prometheus queries  
  - Edge cases: Trigger failure rare; ensure n8n server uptime.  
  
---

#### 1.2 Prometheus Queries for Services and Pods

- **Overview:**  
  Executes multiple Prometheus HTTP queries to fetch Kubernetes cluster health metrics.

- **Nodes Involved:**  
  - Current endpoints  
  - Endpoints 5m ago  
  - Pods Running State  
  - Containers Not Ready  
  - CrashLoopBackOff / Termination Reason  
  - Pod Restart Spike (last 5m)  
  - Pod Pending State

- **Node Details:**  

  - *Current endpoints*  
    - Type: HTTP Request  
    - Role: Queries Prometheus for current available service endpoints, grouped by namespace and service  
    - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=sum by (namespace, service) (kube_endpoint_address_available)`  
    - Input: Trigger node  
    - Output: JSON with `data.result` array  
    - Failure: Network errors, Prometheus downtime, malformed responses
  
  - *Endpoints 5m ago*  
    - Type: HTTP Request  
    - Role: Queries Prometheus for available service endpoints 5 minutes ago (offset) for trend comparison  
    - URL: Similar to above with `offset 5m`  
    - Input/Output: Same as above  
    - Failure: Same as above
  
  - *Pods Running State*  
    - Type: HTTP Request  
    - Role: Queries Prometheus for pods in Running phase, grouped by namespace and pod  
    - URL: `...query=sum(kube_pod_status_phase{phase="Running"}) by (namespace, pod)`  
    - Failure: Same as above
  
  - *Containers Not Ready*  
    - Type: HTTP Request  
    - Role: Queries pods with any containers not ready  
    - URL: `...query=kube_pod_container_status_ready == 0`  
    - Failure: Same as above
  
  - *CrashLoopBackOff / Termination Reason*  
    - Type: HTTP Request  
    - Role: Queries pods in CrashLoopBackOff or terminated due to errors  
    - URL: `...query=kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1`  
    - Failure: Same as above
  
  - *Pod Restart Spike (last 5m)*  
    - Type: HTTP Request  
    - Role: Queries pods with restart count spikes in last 5 minutes  
    - URL: `...query=increase(kube_pod_container_status_restarts_total[5m]) > 0`  
    - Failure: Same as above
  
  - *Pod Pending State*  
    - Type: HTTP Request  
    - Role: Queries pods in Pending phase  
    - URL: `...query=kube_pod_status_phase{phase="Pending"} > 0`  
    - Failure: Same as above

---

#### 1.3 Data Mapping and Merging

- **Overview:**  
  Transforms raw Prometheus JSON results into structured objects keyed by namespace/service or pod attributes. Merges current and past endpoint data as well as all pod metrics into unified streams.

- **Nodes Involved:**  
  - Map Prometheus results into namespace/service  
  - Map Prometheus results into namespace/service1  
  - Merge (endpoints)  
  - Merge All data collected

- **Node Details:**  

  - *Map Prometheus results into namespace/service* (two nodes with similar logic)  
    - Type: Code  
    - Role: Converts Prometheus `data.result` array into `{ "<namespace>/<service>": value }` object for easier downstream comparison  
    - Input: HTTP Request JSON from current and 5m ago endpoints  
    - Output: Structured JSON object  
    - Edge cases: Empty or missing `data.result`; malformed metric fields
  
  - *Merge (endpoints)*  
    - Type: Merge  
    - Role: Combines current and 5m ago mapped endpoint data into a single stream  
    - Input: From both mapping nodes  
    - Output: Combined dataset for comparison  
    - Edge cases: Unequal input sizes, missing data
  
  - *Merge All data collected*  
    - Type: Merge  
    - Role: Combines six inputs: merged endpoints, pods running, containers not ready, CrashLoopBackOff pods, pod restarts, pod pending  
    - Input: Multiple HTTP requests and normalization outputs  
    - Output: Unified data array for normalization and formatting  
    - Edge cases: Partial data, node failures

---

#### 1.4 Normalization and Aggregation

- **Overview:**  
  Cleans, classifies, and transforms mixed raw data from Prometheus queries into a consistent JSON schema representing pod and service health states.

- **Nodes Involved:**  
  - Normalization Node  
  - Normalization

- **Node Details:**  

  - *Normalization Node*  
    - Type: Code  
    - Role: Processes pod restart spike raw data to structured objects with keys: type, namespace, pod, container, value  
    - Input: Pod Restart Spike HTTP Request  
    - Output: Normalized pod_restart objects  
    - Edge cases: Missing container info, numeric parse errors
  
  - *Normalization*  
    - Type: Code  
    - Role: Processes merged combined data with multiple pod states and services; classifies by type (pod_restart, pod_not_ready, pod_pending, pod_waiting_reason, service) and extracts relevant metadata  
    - Input: Merged all data collected node  
    - Output: Array of normalized JSON objects representing health issues or service states  
    - Edge cases: Unexpected metric fields or missing data

---

#### 1.5 Slack Message Formatting

- **Overview:**  
  Aggregates normalized data into grouped alerts with emojis and human-readable descriptions. Summarizes service endpoint changes and pod health issues.

- **Nodes Involved:**  
  - Slack formatter with summary

- **Node Details:**  
  - Type: Code  
  - Role: Groups input items by type (service, pod_not_ready, pod_waiting_reason, pod_restart, pod_pending), detects service endpoint changes between snapshots, builds a formatted Slack message string with sections and emojis, ready for posting  
  - Input: Normalized data array  
  - Output: JSON object with `text` property containing Slack message  
  - Edge cases: No changes or no issues detected (graceful message), empty input arrays, string formatting errors

---

#### 1.6 Slack Notification

- **Overview:**  
  Sends the formatted alert message to a specific Slack channel using an authenticated HTTP POST request.

- **Nodes Involved:**  
  - 📤 Send Alerts to Slack

- **Node Details:**  
  - Type: HTTP Request  
  - Role: Posts the message to Slack API endpoint `chat.postMessage` using bearer token authentication  
  - Parameters:  
    - Channel: `#k8s-alerts`  
    - Text: Taken from previous node's output (`{{$json["text"]}}`)  
    - Headers: Content-Type application/json  
  - Credentials: Bearer token stored in n8n credentials  
  - Input: Slack formatted message  
  - Output: Slack API response  
  - Failure: Authentication errors, network issues, Slack API rate limits

---

### 3. Summary Table

| Node Name                           | Node Type              | Functional Role                               | Input Node(s)                             | Output Node(s)                            | Sticky Note                                                                                                                                                     |
|-----------------------------------|------------------------|----------------------------------------------|------------------------------------------|------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 🕒 Every 5 Min Trigger             | Schedule Trigger       | Initiates workflow every 5 minutes            | None                                     | Current endpoints, Endpoints 5m ago, Pods Running State, Containers Not Ready, CrashLoopBackOff, Pod Restart Spike, Pod Pending State | ⏲ Runs every 5 minutes; kicks off monitoring; ensures continuous health checks of pods & endpoints                                                               |
| Current endpoints                 | HTTP Request           | Queries current service endpoints             | 🕒 Every 5 Min Trigger                   | Map Prometheus results into namespace/service | Runs a Prometheus query for current service endpoints; fetches real-time endpoint data                                                                           |
| Endpoints 5m ago                 | HTTP Request           | Queries service endpoints 5 minutes ago       | 🕒 Every 5 Min Trigger                   | Map Prometheus results into namespace/service1 | Runs a Prometheus query for endpoints 5 minutes ago; useful for comparing trends/changes                                                                        |
| Map Prometheus results into namespace/service | Code                  | Maps Prometheus results to namespace/service | Current endpoints                       | Merge (endpoints)                        | Maps the raw Prometheus query result into structured format: namespace → service → endpoint                                                                     |
| Map Prometheus results into namespace/service1| Code                  | Maps Prometheus results to namespace/service | Endpoints 5m ago                       | Merge (endpoints)                        | Maps the raw Prometheus query result into structured format: namespace → service → endpoint                                                                     |
| Merge (endpoints)                | Merge                  | Merges current and 5-minute-old endpoints    | Map nodes for current and 5m ago       | Merge All data collected                 | Merges current and 5-min old endpoint data; allows comparison for detecting endpoint changes                                                                   |
| Pods Running State               | HTTP Request           | Queries pods in Running state                  | 🕒 Every 5 Min Trigger                   | Merge All data collected                 | Queries Prometheus for pods in Running state; helps verify overall cluster health                                                                              |
| Containers Not Ready             | HTTP Request           | Queries pods with containers not ready        | 🕒 Every 5 Min Trigger                   | Merge All data collected                 | Queries Prometheus for pods stuck in Not Ready state; useful for spotting readiness probe failures                                                             |
| CrashLoopBackOff / Termination Reason | HTTP Request           | Queries pods in CrashLoopBackOff or terminated| 🕒 Every 5 Min Trigger                   | Merge All data collected                 | Queries pods in CrashLoopBackOff or terminated with error; critical for alerting on failed workloads                                                           |
| Pod Restart Spike (last 5m)      | HTTP Request           | Queries pods restarted in last 5 minutes      | 🕒 Every 5 Min Trigger                   | Normalization Node                       | Queries pods that restarted multiple times in last 5 minutes; detects unhealthy pods with restart loops                                                       |
| Normalization Node               | Code                   | Normalizes pod restart data                    | Pod Restart Spike (last 5m)             | Merge All data collected                 | Normalizes pod restart data; ensures consistent format before merging                                                                                         |
| Pod Pending State               | HTTP Request           | Queries pods in Pending state                   | 🕒 Every 5 Min Trigger                   | Merge All data collected                 | Queries pods in Pending phase; identifies pods pending scheduling                                                                                            |
| Merge All data collected        | Merge                  | Merges all collected metrics                   | Merge (endpoints), Pods Running State, Containers Not Ready, CrashLoopBackOff, Normalization Node, Pod Pending State | Normalization                          | Merges all pod health data and endpoints into a single combined dataset                                                                                      |
| Normalization                   | Code                   | Cleans and structures merged data              | Merge All data collected                | Slack formatter with summary             | Cleans and structures merged data; removes duplicates/empty values                                                                                             |
| Slack formatter with summary    | Code                   | Formats monitoring results into Slack message | Normalization                          | 📤 Send Alerts to Slack                  | Converts monitoring results into human-readable Slack message format; adds severity & summary tags                                                            |
| 📤 Send Alerts to Slack          | HTTP Request           | Sends alert message to Slack                    | Slack formatter with summary            | None                                     | Sends the final alert message to Slack channel via webhook; notifies DevOps team instantly                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: "🕒 Every 5 Min Trigger"  
   - Type: Schedule Trigger  
   - Configure to trigger every 5 minutes

2. **Create seven HTTP Request nodes to query Prometheus:**  

   - *Current endpoints*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=sum by (namespace, service) (kube_endpoint_address_available)`  
     - Method: GET  
     - Connect input from trigger node

   - *Endpoints 5m ago*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=sum by (namespace, service) (kube_endpoint_address_available offset 5m)`  
     - Connect input from trigger node

   - *Pods Running State*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=sum(kube_pod_status_phase{phase="Running"}) by (namespace, pod)`  
     - Connect input from trigger node

   - *Containers Not Ready*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=kube_pod_container_status_ready == 0`  
     - Connect input from trigger node

   - *CrashLoopBackOff / Termination Reason*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1`  
     - Connect input from trigger node

   - *Pod Restart Spike (last 5m)*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=increase(kube_pod_container_status_restarts_total[5m]) > 0`  
     - Connect input from trigger node

   - *Pod Pending State*  
     - URL: `http://prometheus-kube-prometheus-prometheus.monitoring:9090/api/v1/query?query=kube_pod_status_phase{phase="Pending"} > 0`  
     - Connect input from trigger node

3. **Create two Code nodes to map Prometheus endpoint results:**  

   - *Map Prometheus results into namespace/service*  
     - JavaScript to iterate over `data.result` and create an object keyed by `namespace/service` with the sum value for current endpoints  
     - Input from *Current endpoints* HTTP node  

   - *Map Prometheus results into namespace/service1*  
     - Same JavaScript logic as above  
     - Input from *Endpoints 5m ago* HTTP node  

4. **Create a Merge node ("Merge (endpoints)")**  
   - Combine the two mapped endpoint nodes as inputs  
   - Use default merge settings (merge by position)  

5. **Create a Merge node ("Merge All data collected")**  
   - Add six inputs connected as follows:  
     - First input: from "Merge (endpoints)"  
     - Second input: from "Pods Running State" HTTP node  
     - Third input: from "Containers Not Ready" HTTP node  
     - Fourth input: from "CrashLoopBackOff / Termination Reason" HTTP node  
     - Fifth input: from the normalization node (created next)  
     - Sixth input: from "Pod Pending State" HTTP node  

6. **Create a Code node ("Normalization Node") for Pod Restart Spike data:**  
   - Input from *Pod Restart Spike (last 5m)* HTTP node  
   - JavaScript to parse `data.result`, normalize pod restart info with fields: type, namespace, pod, container, value  
   - Output to "Merge All data collected" node (input #5)

7. **Create a Code node ("Normalization") to process the merged data:**  
   - Input from "Merge All data collected" node  
   - JavaScript logic to iterate over merged data, classify pod and service issues with types like pod_restart, pod_waiting_reason, pod_not_ready, pod_pending, service, and produce normalized JSON objects  
   - Output to Slack formatter node  

8. **Create a Code node ("Slack formatter with summary"):**  
   - Input from "Normalization" node  
   - JavaScript to group data by type, detect service endpoint changes, build a formatted Slack message with emojis and sections, output JSON with `text` property  

9. **Create an HTTP Request node ("📤 Send Alerts to Slack"):**  
   - Method: POST  
   - URL: `https://slack.com/api/chat.postMessage`  
   - Headers: `Content-Type: application/json`  
   - Authentication: HTTP Bearer Auth with Slack API token credential configured in n8n  
   - Body parameters:  
     - channel: `#k8s-alerts`  
     - text: expression `{{$json["text"]}}` from previous node  
   - Input from Slack formatter node  
   
10. **Connect all nodes appropriately:**  
    - Trigger to all Prometheus query nodes  
    - Prometheus queries to mapping or normalization as detailed  
    - Merge nodes to combine data streams  
    - Normalization to Slack formatting  
    - Slack formatting to Slack HTTP request  

11. **Credential Setup:**  
    - Create and configure HTTP Bearer Auth credential for Slack API token named appropriately (e.g., "Bearer Auth account")  

12. **Test and validate the workflow:**  
    - Run manually or wait for schedule trigger  
    - Confirm Prometheus queries return valid data  
    - Confirm Slack messages post correctly and contain expected grouped alerts  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Runs every 5 minutes via Cron trigger; ensures continuous monitoring of Kubernetes pods and services with Prometheus metrics.                                                | Workflow overview and trigger node                               |
| Prometheus queries cover key Kubernetes metrics: pod restarts, pending pods, not ready pods, service discovery counts.                                                       | Prometheus query nodes and sticky notes                          |
| Data normalization standardizes diverse Prometheus metrics into a uniform JSON schema for easier aggregation and alerting.                                                  | Normalization code nodes                                         |
| Slack alert messages use emojis and grouping for quick identification of issues; service endpoint changes are compared between current and 5-minute-ago snapshots.           | Slack formatter node and message format                          |
| Bearer token authentication is required for Slack API integration; ensure token has permissions to post messages in target channel `#k8s-alerts`.                            | Slack HTTP Request node and credential setup                     |
| Useful for Kubernetes DevOps teams to gain real-time visibility into cluster health and receive immediate alerts on critical pod events.                                     | Project target users                                             |
| Sticky notes in the workflow provide additional explanations and context for nodes, improving maintainability and understanding.                                            | Sticky Notes in workflow JSON                                    |
| For Prometheus metric references and queries, see official kube-prometheus or kube-state-metrics documentation.                                                               | https://github.com/prometheus-community/kube-prometheus         |
| Slack API chat.postMessage documentation for customizing alert messages and authentication details.                                                                          | https://api.slack.com/methods/chat.postMessage                   |

---

This document covers all workflow nodes, logic blocks, configurations, and instructions to reproduce the automated Kubernetes monitoring and Slack alerting workflow in n8n.