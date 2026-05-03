Run Apache Airflow DAG and Retrieve XCom Value

https://n8nworkflows.xyz/workflows/run-apache-airflow-dag-and-retrieve-xcom-value-3026


# Run Apache Airflow DAG and Retrieve XCom Value

### 1. Workflow Overview

This workflow enables n8n to programmatically trigger an Apache Airflow DAG run, monitor its execution state, and retrieve the result of a specific task’s XCom return value. It is designed for use cases where Airflow orchestrates workflows and n8n acts as an external controller and consumer of Airflow task results.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives input parameters including DAG ID, task ID, DAG run configuration, and timing controls.
- **1.2 Airflow API Setup:** Sets the Airflow API base URL prefix for subsequent HTTP requests.
- **1.3 Trigger DAG Run:** Sends a POST request to Airflow API to start the DAG run with the given configuration.
- **1.4 Monitor DAG Run State:** Periodically checks the DAG run status until it reaches a terminal state (success or failure) or times out.
- **1.5 Retrieve XCom Result:** Upon successful completion, fetches the XCom return value of the specified task.
- **1.6 Error Handling and Timing Control:** Manages retries, wait intervals, and errors such as DAG run failure or timeout.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives and exposes the input parameters required to trigger and monitor the Airflow DAG run.

- **Nodes Involved:**  
  - `in data`

- **Node Details:**  
  - **Node:** `in data`  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for the workflow; accepts inputs: `dag_id`, `task_id`, `conf`, `wait`, `wait_time`.  
    - Configuration: Defines workflow inputs with expected names and types.  
    - Inputs: External trigger or sub-workflow call.  
    - Outputs: Passes input data downstream.  
    - Edge Cases: Missing or malformed inputs could cause downstream failures. Validation is external to this node.

#### 2.2 Airflow API Setup

- **Overview:**  
  Sets the Airflow API URL prefix used by all subsequent HTTP requests to the Airflow REST API.

- **Nodes Involved:**  
  - `airflow-api`

- **Node Details:**  
  - **Node:** `airflow-api`  
    - Type: Set  
    - Role: Defines a constant string variable `prefix` with the Airflow API base URL (e.g., `https://airflow.example.com`).  
    - Configuration: Hardcoded string assignment for API prefix.  
    - Inputs: Receives input data from `in data`.  
    - Outputs: Provides the API prefix for HTTP request nodes.  
    - Edge Cases: Incorrect URL or protocol will cause HTTP request failures.

#### 2.3 Trigger DAG Run

- **Overview:**  
  Initiates the Airflow DAG run by sending a POST request with the provided configuration.

- **Nodes Involved:**  
  - `Airflow: dag_run`

- **Node Details:**  
  - **Node:** `Airflow: dag_run`  
    - Type: HTTP Request  
    - Role: Sends POST to `/api/v1/dags/{dag_id}/dagRuns` to start a DAG run.  
    - Configuration:  
      - URL dynamically constructed using `airflow-api.prefix` and `in data.dag_id`.  
      - JSON body includes `conf` parameter from input.  
      - Authentication: Basic Auth using stored Airflow credentials.  
    - Inputs: From `airflow-api`.  
    - Outputs: Returns JSON including `dag_run_id` used for monitoring.  
    - Edge Cases: Authentication failure, network errors, invalid DAG ID, or malformed `conf` cause errors.

#### 2.4 Monitor DAG Run State

- **Overview:**  
  Polls the Airflow API to check the state of the triggered DAG run until it completes or fails, with retry and timeout logic.

- **Nodes Involved:**  
  - `Airflow: dag_run - state`  
  - `Switch: state`  
  - `if state == queued`  
  - `Wait`  
  - `count`  
  - `If count > wait_time`  
  - `dag run wait too long`  
  - `dag run fail`

- **Node Details:**  
  - **Node:** `Airflow: dag_run - state`  
    - Type: HTTP Request  
    - Role: GET request to `/api/v1/dags/{dag_id}/dagRuns/{dag_run_id}` to retrieve current DAG run state.  
    - Configuration: URL uses `airflow-api.prefix`, `in data.dag_id`, and `Airflow: dag_run.dag_run_id`.  
    - Authentication: Basic Auth.  
    - Inputs: From `Wait` node (polling loop).  
    - Outputs: JSON with `state` field.  
    - Edge Cases: Network errors, invalid dag_run_id, auth errors.

  - **Node:** `Switch: state`  
    - Type: Switch  
    - Role: Routes flow based on DAG run `state` value (`success`, `queued`, `running`, `failed`).  
    - Configuration: Checks `state` field in JSON response.  
    - Inputs: From `Airflow: dag_run - state`.  
    - Outputs:  
      - `success` → proceed to retrieve XCom result.  
      - `queued` or `running` → increment count and wait.  
      - `failed` → trigger error node.  
    - Edge Cases: Unexpected states fall to fallback output (treated as failure).

  - **Node:** `if state == queued`  
    - Type: If  
    - Role: Checks if state is exactly `queued` to decide wait or fail.  
    - Inputs: From `Airflow: dag_run`.  
    - Outputs:  
      - True → Wait node to delay next poll.  
      - False → Fail node.  
    - Edge Cases: Misinterpretation of states could cause premature failure.

  - **Node:** `Wait`  
    - Type: Wait  
    - Role: Pauses execution for a configurable number of seconds before next poll.  
    - Configuration: Uses `wait` input parameter or defaults to 10 seconds.  
    - Inputs: From `if state == queued` or `If count > wait_time`.  
    - Outputs: Triggers next state check.  
    - Edge Cases: Long wait times increase total workflow duration.

  - **Node:** `count`  
    - Type: Code  
    - Role: Tracks number of polling iterations by incrementing a counter.  
    - Configuration: JavaScript code increments `count` or initializes it to 1.  
    - Inputs: From `Switch: state` for queued/running states.  
    - Outputs: Updated count JSON.  
    - Edge Cases: Counter reset or missing data could break timeout logic.

  - **Node:** `If count > wait_time`  
    - Type: If  
    - Role: Compares current count to max allowed wait time (in iterations).  
    - Configuration: Uses `wait_time` input parameter or defaults to 12.  
    - Inputs: From `count`.  
    - Outputs:  
      - True → triggers `dag run wait too long` error node.  
      - False → triggers `Wait` node to continue polling.  
    - Edge Cases: Incorrect wait_time leads to premature timeout or excessive waiting.

  - **Node:** `dag run wait too long`  
    - Type: Stop and Error  
    - Role: Stops workflow with error message "dag run wait too long" when timeout exceeded.  
    - Inputs: From `If count > wait_time`.  
    - Outputs: None (workflow ends).  
    - Edge Cases: None.

  - **Node:** `dag run fail`  
    - Type: Stop and Error  
    - Role: Stops workflow with error message "dag run fail" when DAG run fails or unexpected state.  
    - Inputs: From `Switch: state` failure output and `if state == queued` false output.  
    - Outputs: None (workflow ends).  
    - Edge Cases: None.

#### 2.5 Retrieve XCom Result

- **Overview:**  
  After successful DAG run completion, retrieves the XCom `return_value` from the specified task.

- **Nodes Involved:**  
  - `Airflow: dag_run - get result`

- **Node Details:**  
  - **Node:** `Airflow: dag_run - get result`  
    - Type: HTTP Request  
    - Role: GET request to `/api/v1/dags/{dag_id}/dagRuns/{dag_run_id}/taskInstances/{task_id}/xcomEntries/return_value` to fetch XCom result.  
    - Configuration: URL uses `airflow-api.prefix`, `in data.dag_id`, `Airflow: dag_run.dag_run_id`, and `in data.task_id`.  
    - Authentication: Basic Auth.  
    - Inputs: From `Switch: state` success output.  
    - Outputs: JSON containing the XCom `value` field with the task’s return value.  
    - Edge Cases: Missing XCom entry, auth errors, or network issues.

---

### 3. Summary Table

| Node Name                  | Node Type              | Functional Role                          | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                   |
|----------------------------|------------------------|----------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------|
| in data                    | Execute Workflow Trigger | Entry point; receives input parameters | External trigger            | airflow-api                   |                                                                                                               |
| airflow-api                | Set                    | Sets Airflow API base URL prefix       | in data                     | Airflow: dag_run              | Update Airflow API Link Prefix in this node to your Airflow server URL (e.g., https://airflow.example.com)     |
| Airflow: dag_run           | HTTP Request           | Triggers Airflow DAG run                | airflow-api                 | if state == queued            | Configure Basic Auth credentials here for Airflow API access                                                  |
| if state == queued         | If                     | Checks if DAG run state is 'queued'    | Airflow: dag_run            | Wait (true), dag run fail (false) |                                                                                                               |
| Wait                      | Wait                   | Waits between DAG run state checks     | if state == queued, If count > wait_time | Airflow: dag_run - state     | Wait time configurable via input parameter `wait`, default 10 seconds                                        |
| Airflow: dag_run - state   | HTTP Request           | Gets current DAG run state              | Wait                       | Switch: state                 | Configure Basic Auth credentials here for Airflow API access                                                  |
| Switch: state             | Switch                 | Routes flow based on DAG run state      | Airflow: dag_run - state    | Airflow: dag_run - get result (success), count (queued/running), dag run fail (failed) |                                                                                                               |
| count                     | Code                   | Counts polling iterations               | Switch: state               | If count > wait_time          |                                                                                                               |
| If count > wait_time       | If                     | Checks if polling exceeded max wait    | count                      | dag run wait too long (true), Wait (false) |                                                                                                               |
| dag run wait too long      | Stop and Error         | Stops workflow on timeout               | If count > wait_time        | None                         |                                                                                                               |
| dag run fail              | Stop and Error         | Stops workflow on DAG run failure       | Switch: state (failed), if state == queued (false) | None                         |                                                                                                               |
| Airflow: dag_run - get result | HTTP Request           | Retrieves XCom return value             | Switch: state (success)     | None                         | Configure Basic Auth credentials here for Airflow API access                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `in data` node**  
   - Type: Execute Workflow Trigger  
   - Add workflow inputs:  
     - `dag_id` (string)  
     - `task_id` (string)  
     - `conf` (string or JSON)  
     - `wait` (number, optional)  
     - `wait_time` (number, optional)  
   - This node serves as the entry point.

2. **Create `airflow-api` node**  
   - Type: Set  
   - Add string field `prefix` with your Airflow API base URL, e.g., `https://airflow.example.com`.  
   - Connect `in data` → `airflow-api`.

3. **Create `Airflow: dag_run` node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `={{ $('airflow-api').item.json.prefix }}/api/v1/dags/{{ $('in data').item.json.dag_id }}/dagRuns`  
   - Body Content Type: JSON  
   - Body: `{ "conf": {{ $('in data').item.json.conf }} }`  
   - Authentication: HTTP Basic Auth  
   - Credentials: Configure with Airflow username/password  
   - Connect `airflow-api` → `Airflow: dag_run`.

4. **Create `if state == queued` node**  
   - Type: If  
   - Condition: Check if `{{$json["state"]}}` equals `"queued"`  
   - Connect `Airflow: dag_run` → `if state == queued`.

5. **Create `Wait` node**  
   - Type: Wait  
   - Duration: Expression `={{ $('in data').item.json.hasOwnProperty('wait') ? $('in data').item.json.wait : 10 }}` seconds  
   - Connect `if state == queued` (true) → `Wait`.

6. **Create `Airflow: dag_run - state` node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $('airflow-api').item.json.prefix }}/api/v1/dags/{{ $('in data').item.json.dag_id }}/dagRuns/{{ $('Airflow: dag_run').item.json.dag_run_id }}`  
   - Authentication: HTTP Basic Auth (same credentials)  
   - Connect `Wait` → `Airflow: dag_run - state`.

7. **Create `Switch: state` node**  
   - Type: Switch  
   - Rules based on `{{$json["state"]}}` with outputs:  
     - `success` (state == "success")  
     - `queued` (state == "queued")  
     - `running` (state == "running")  
     - `failed` (state == "failed")  
   - Connect `Airflow: dag_run - state` → `Switch: state`.

8. **Create `Airflow: dag_run - get result` node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $('airflow-api').item.json.prefix }}/api/v1/dags/{{ $('in data').item.json.dag_id }}/dagRuns/{{ $('Airflow: dag_run').item.json.dag_run_id }}/taskInstances/{{ $('in data').item.json.task_id }}/xcomEntries/return_value`  
   - Authentication: HTTP Basic Auth (same credentials)  
   - Connect `Switch: state` success output → `Airflow: dag_run - get result`.

9. **Create `count` node**  
   - Type: Code  
   - JavaScript code:  
     ```js
     try {
       $('count').first().json.count += 1;
       return { count: $('count').first().json.count };
     } catch (error) {
       return { count: 1 };
     }
     ```  
   - Connect `Switch: state` queued and running outputs → `count`.

10. **Create `If count > wait_time` node**  
    - Type: If  
    - Condition: `{{$json.count}} > {{$('in data').item.json.hasOwnProperty('wait_time') ? $('in data').item.json.wait_time : 12}}`  
    - Connect `count` → `If count > wait_time`.

11. **Create `dag run wait too long` node**  
    - Type: Stop and Error  
    - Error Message: "dag run wait too long"  
    - Connect `If count > wait_time` true output → `dag run wait too long`.

12. **Connect `If count > wait_time` false output → `Wait`**  
    - This loops back to wait before next state check.

13. **Connect `Switch: state` failed output → `dag run fail` node**  
    - Create `dag run fail` node: Stop and Error with message "dag run fail".

14. **Connect `if state == queued` false output → `dag run fail`**  
    - This handles unexpected states after initial DAG run trigger.

15. **Connect `Airflow: dag_run` → `if state == queued`**  
    - Already done in step 4.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                      | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Update the Airflow API Link Prefix in the `airflow-api` node to match your Airflow server URL (e.g., `https://airflow.example.com`).                                                                                                            | Preparation instructions in workflow description                                                 |
| Configure Basic Authentication credentials in all HTTP Request nodes interacting with Airflow API (`Airflow: dag_run`, `Airflow: dag_run - state`, `Airflow: dag_run - get result`). Consider more secure auth methods if available.                | https://airflow.apache.org/docs/apache-airflow-providers-fab/stable/auth-manager/api-authentication.html#basic-authentication |
| Ensure Airflow user has permissions: `can create on DAG Runs`, `can read on DAG Runs`, `can read on XComs`, `can edit on DAGs`, and `can read on DAGs`.                                                                                          | Preparation instructions in workflow description                                                 |
| The workflow expects input parameters: `dag_id`, `task_id`, `conf` (JSON string), `wait` (poll interval in seconds), and `wait_time` (max polling iterations).                                                                                   | Usage instructions                                                                              |
| The workflow returns the XCom `return_value` in the `value` field of the final output JSON.                                                                                                                                                       | Output description                                                                              |
| Airflow API documentation references: DAGRun endpoint https://airflow.apache.org/docs/apache-airflow/2.10.5/stable-rest-api-ref.html#tag/DAGRun and XCom endpoint https://airflow.apache.org/docs/apache-airflow/2.10.5/stable-rest-api-ref.html#tag/XCom |                                                                                                 |

---

This document fully describes the workflow structure, node configurations, logic flow, error handling, and reproduction steps for both human and automation agent use.