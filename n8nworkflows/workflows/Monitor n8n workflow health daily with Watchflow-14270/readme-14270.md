Monitor n8n workflow health daily with Watchflow

https://n8nworkflows.xyz/workflows/monitor-n8n-workflow-health-daily-with-watchflow-14270


# Monitor n8n workflow health daily with Watchflow

# 1. Workflow Overview

This workflow performs a daily health audit of selected active n8n workflows and reports their latest execution status to Watchflow. It is designed for operators who want a centralized monitoring layer for workflow reliability without manually checking each workflow in n8n.

The workflow follows a simple monitoring chain:

- Trigger once per day
- Fetch active workflows tagged `daily`
- Retrieve the latest execution for each workflow
- Evaluate whether the last execution succeeded
- Send either a success ping or a failure event to Watchflow

## 1.1 Daily Trigger

The workflow starts on a schedule and launches the audit automatically once per day.

## 1.2 Workflow Discovery and Execution Lookup

It queries the n8n API for workflows that are both active and tagged `daily`, then fetches the most recent execution for each of those workflows.

## 1.3 Health Evaluation

It checks the status of the latest execution and splits the logic into success and failure paths.

## 1.4 Watchflow Reporting

It sends a health signal to Watchflow:
- `Ping` when the latest execution is successful
- `Fail` when the latest execution is not successful and an error message is available

---

# 2. Block-by-Block Analysis

## 2.1 Daily Trigger

**Overview:**  
This block is the workflow entry point. It runs on a recurring schedule and starts the health scan automatically without manual input.

**Nodes Involved:**  
- Every Day

### Node Details

#### Every Day
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Scheduled trigger node that starts the workflow at a recurring interval.
- **Configuration choices:**  
  The rule uses an interval configuration with default daily behavior. In practice, this is intended to fire once per day.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none, this is the entry point
  - Output: `Get All Workflows`
- **Version-specific requirements:**  
  Uses `typeVersion: 1.1`, which corresponds to the newer Schedule Trigger implementation in n8n.
- **Edge cases or potential failure types:**  
  - Misconfigured timezone on the n8n instance can cause the run time to differ from expectations
  - If the workflow is inactive, the trigger will not run
  - If the interval is edited incorrectly, it may run more or less frequently than intended
- **Sub-workflow reference:**  
  None.

---

## 2.2 Workflow Discovery and Execution Lookup

**Overview:**  
This block uses the n8n API to discover the workflows that should be monitored, then retrieves the latest execution for each one. It forms the data collection layer of the monitoring process.

**Nodes Involved:**  
- Get All Workflows
- Get Last Execution

### Node Details

#### Get All Workflows
- **Type and technical role:** `n8n-nodes-base.n8n`  
  Native n8n API node used to query workflow metadata from the same or another n8n instance.
- **Configuration choices:**  
  - Filters workflows by:
    - tag: `daily`
    - active workflows only: enabled
  - `returnAll` is disabled, so the node will not fetch unlimited results
- **Key expressions or variables used:**  
  No dynamic expression in the parameters shown.
- **Input and output connections:**  
  - Input: `Every Day`
  - Output: `Get Last Execution`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - Invalid or missing n8n API credentials
  - Incorrect base URL for the n8n API
  - Tag mismatch: workflows without the `daily` tag will not be included
  - Pagination risk: because `returnAll` is `false`, some matching workflows may be omitted if there are more results than the node’s default limit
  - API permission issues if the token lacks access to workflow metadata
- **Sub-workflow reference:**  
  None.

#### Get Last Execution
- **Type and technical role:** `n8n-nodes-base.n8n`  
  Native n8n API node querying execution records for each workflow returned by the previous node.
- **Configuration choices:**  
  - Resource: `execution`
  - Limit: `1`
  - Filter by `workflowId` using the current item’s workflow ID
  - `activeWorkflows` option enabled
- **Key expressions or variables used:**  
  - `={{ $json.id }}`  
    Uses the current workflow’s ID from `Get All Workflows` to request its latest execution.
- **Input and output connections:**  
  - Input: `Get All Workflows`
  - Output: `Is Successful?`
- **Version-specific requirements:**  
  Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - If a workflow has never executed, this node may return no item, which prevents downstream reporting for that workflow
  - API permission issues on execution history
  - If executions are pruned aggressively, the “latest” execution might not reflect the full operational history
  - If multiple items are returned unexpectedly, downstream nodes assume one latest execution per workflow
- **Sub-workflow reference:**  
  None.

---

## 2.3 Health Evaluation

**Overview:**  
This block determines whether the latest execution should be treated as healthy or failed. It routes successful executions to the Watchflow ping path and everything else to the failure path.

**Nodes Involved:**  
- Is Successful?

### Node Details

#### Is Successful?
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional branching node that evaluates execution status.
- **Configuration choices:**  
  - Compares the field `status` against the string `success`
  - Uses strict validation and a single `and` condition
- **Key expressions or variables used:**  
  - `={{ $json.status }}`
- **Input and output connections:**  
  - Input: `Get Last Execution`
  - True output: `Watchflow: Ping`
  - False output: `Watchflow: Fail`
- **Version-specific requirements:**  
  Uses `typeVersion: 2`.
- **Edge cases or potential failure types:**  
  - If `status` is missing or null, the condition evaluates false and the workflow goes to the fail path
  - Status values other than exactly `success` will be treated as failures
  - This means statuses like `error`, `crashed`, `waiting`, `running`, or custom future statuses are all grouped as non-success
- **Sub-workflow reference:**  
  None.

---

## 2.4 Watchflow Reporting

**Overview:**  
This block sends the result of the health evaluation to Watchflow. It uses the monitored workflow name to build a Watchflow key and posts either a success heartbeat or a failure alert with execution metadata.

**Nodes Involved:**  
- Watchflow: Ping
- Watchflow: Fail

### Node Details

#### Watchflow: Ping
- **Type and technical role:** `CUSTOM.watchflow`  
  Custom Watchflow integration node that reports a successful health event.
- **Configuration choices:**  
  - Operation defaults to success/ping behavior
  - `name` is taken from the original workflow name
  - `key` is a slug-like value derived from the workflow name
  - `data` includes:
    - `last_execution_id`
    - `finished_at`
- **Key expressions or variables used:**  
  - `={{ $('Get All Workflows').item.json.name.toLowerCase().replace(/\s+/g, '-') }}`
  - `={{ $('Get All Workflows').item.json.name }}`
  - `={ "last_execution_id": "{{ $json.id }}", "finished_at": "{{ $json.stoppedAt }}" }`
- **Input and output connections:**  
  - Input: true branch from `Is Successful?`
  - Output: none
- **Version-specific requirements:**  
  - Uses custom node package `@watchflow/n8n-nodes-watchflow`
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Custom node package not installed on the n8n instance
  - Invalid Watchflow API credentials
  - Workflow names with special characters may produce unstable or non-unique keys after normalization
  - If `stoppedAt` is absent, the payload may contain an empty finish time
  - Cross-node item reference using `$('Get All Workflows').item...` assumes item linkage remains intact
- **Sub-workflow reference:**  
  None.

#### Watchflow: Fail
- **Type and technical role:** `CUSTOM.watchflow`  
  Custom Watchflow integration node that reports a failed health event.
- **Configuration choices:**  
  - Explicit operation: `fail`
  - `name` is taken from the original workflow name
  - `key` is generated from the workflow name in the same way as the ping node
  - `error` field is populated from the execution error message
  - `data` includes:
    - `last_execution_id`
    - `error_details`
- **Key expressions or variables used:**  
  - `={{ $('Get All Workflows').item.json.name.toLowerCase().replace(/\s+/g, '-') }}`
  - `={{ $('Get All Workflows').item.json.name }}`
  - `={{  $json.data.resultData.error.message }}`
  - `={ "last_execution_id": "{{ $json.id }}", "error_details":  {{ $json.data.resultData.error.message }}}`
- **Input and output connections:**  
  - Input: false branch from `Is Successful?`
  - Output: none
- **Version-specific requirements:**  
  - Requires `@watchflow/n8n-nodes-watchflow`
  - Uses `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - If the execution failed without a populated `data.resultData.error.message`, this expression may resolve to undefined and cause payload issues
  - The `error_details` value in `data` is inserted without guaranteed JSON escaping; messages containing quotes or structured text may break payload formatting
  - Non-success statuses that are not actual failures may still go down this branch and attempt to read an error message that does not exist
  - Invalid Watchflow API credentials or service connectivity issues
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Day | `n8n-nodes-base.scheduleTrigger` | Starts the workflow on a daily schedule |  | Get All Workflows | ### ⏰ 1. Daily Health Audit<br>Initiates the global scan for all active workflows. |
| Get All Workflows | `n8n-nodes-base.n8n` | Retrieves active workflows tagged `daily` from the n8n API | Every Day | Get Last Execution | ### 🔍 2. Dynamic Discovery<br>Fetches active workflows and inspects their latest status. |
| Get Last Execution | `n8n-nodes-base.n8n` | Retrieves the latest execution for each discovered workflow | Get All Workflows | Is Successful? | ### 🔍 2. Dynamic Discovery<br>Fetches active workflows and inspects their latest status. |
| Is Successful? | `n8n-nodes-base.if` | Branches based on whether the latest execution status equals `success` | Get Last Execution | Watchflow: Ping / Watchflow: Fail | ### ⚖️ 3. Health Evaluator<br>Determines if the last run was successful. |
| Watchflow: Ping | `CUSTOM.watchflow` | Sends a success heartbeat to Watchflow | Is Successful? |  | ### 📡 4. Watchflow Sync<br>Broadcasts health status (Ping/Fail) to your Watchflow dashboard. |
| Watchflow: Fail | `CUSTOM.watchflow` | Sends a failure event and error details to Watchflow | Is Successful? |  | ### 📡 4. Watchflow Sync<br>Broadcasts health status (Ping/Fail) to your Watchflow dashboard. |
| Sticky Setup | `n8n-nodes-base.stickyNote` | Documentation note for setup prerequisites |  |  | ### 🚀 Setup & Requirements<br><br>1. **Watchflow Node**: Install `@watchflow/n8n-nodes-watchflow` via npm.<br>2. **Watchflow API Key**: Signup at `watchflow.io` -> Settings -> API Key.<br>3. **n8n API Key**: Go to Settings -> n8n API in your instance.<br>4. **n8n API Base URL**: Use `/v1/api` suffix (e.g., `https://n8n.domain.com/v1/api`). |
| Sticky Trigger | `n8n-nodes-base.stickyNote` | Documentation note for the trigger block |  |  | ### ⏰ 1. Daily Health Audit<br>Initiates the global scan for all active workflows. |
| Sticky Fetch | `n8n-nodes-base.stickyNote` | Documentation note for workflow discovery |  |  | ### 🔍 2. Dynamic Discovery<br>Fetches active workflows and inspects their latest status. |
| Sticky Eval | `n8n-nodes-base.stickyNote` | Documentation note for status evaluation |  |  | ### ⚖️ 3. Health Evaluator<br>Determines if the last run was successful. |
| Sticky Watchflow | `n8n-nodes-base.stickyNote` | Documentation note for Watchflow reporting |  |  | ### 📡 4. Watchflow Sync<br>Broadcasts health status (Ping/Fail) to your Watchflow dashboard. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Watchflow Global Health Monitor`.
   - Ensure the workflow will later be activated.

2. **Install the custom Watchflow node package**
   - Install `@watchflow/n8n-nodes-watchflow` in the n8n environment.
   - Restart n8n if required so the custom node appears in the editor.

3. **Prepare credentials**
   - Create an **n8n API credential** for the `n8n` node:
     - Base URL should include `/v1/api`
     - Example: `https://your-n8n-domain.com/v1/api`
     - Use an n8n API key created from your instance settings
   - Create a **Watchflow API credential** for the custom Watchflow node:
     - Obtain the API key from `watchflow.io` under Settings
   - Verify both credentials with a test call if possible.

4. **Add the trigger node**
   - Create a **Schedule Trigger** node.
   - Name it `Every Day`.
   - Configure the trigger rule as a recurring interval intended to run daily.
   - Leave it as the workflow entry point.

5. **Add the workflow discovery node**
   - Create an **n8n** node.
   - Name it `Get All Workflows`.
   - Set it to query workflows.
   - Configure filters:
     - Tag = `daily`
     - Active workflows only = enabled
   - Set `returnAll` to `false` to match the original workflow.
   - Attach the previously created n8n API credential.
   - Connect `Every Day` → `Get All Workflows`.

6. **Add the execution lookup node**
   - Create another **n8n** node.
   - Name it `Get Last Execution`.
   - Set resource to `execution`.
   - Set limit to `1`.
   - Add a workflow filter using an expression referencing the current item:
     - workflow ID = `{{$json.id}}`
   - Enable the option for active workflows if available in your n8n version.
   - Use the same n8n API credential.
   - Connect `Get All Workflows` → `Get Last Execution`.

7. **Add the conditional evaluator**
   - Create an **If** node.
   - Name it `Is Successful?`.
   - Add one condition:
     - Left value: `{{$json.status}}`
     - Operator: equals
     - Right value: `success`
   - Keep strict validation enabled if available.
   - Connect `Get Last Execution` → `Is Successful?`.

8. **Add the success reporting node**
   - Create a **Watchflow** node from the custom package.
   - Name it `Watchflow: Ping`.
   - Configure:
     - Name: `{{ $('Get All Workflows').item.json.name }}`
     - Key: `{{ $('Get All Workflows').item.json.name.toLowerCase().replace(/\s+/g, '-') }}`
     - Data:
       ```text
       { "last_execution_id": "{{ $json.id }}", "finished_at": "{{ $json.stoppedAt }}" }
       ```
   - Use the Watchflow credential.
   - Connect the **true** output of `Is Successful?` → `Watchflow: Ping`.

9. **Add the failure reporting node**
   - Create another **Watchflow** node.
   - Name it `Watchflow: Fail`.
   - Set operation to `fail`.
   - Configure:
     - Name: `{{ $('Get All Workflows').item.json.name }}`
     - Key: `{{ $('Get All Workflows').item.json.name.toLowerCase().replace(/\s+/g, '-') }}`
     - Error: `{{ $json.data.resultData.error.message }}`
     - Data:
       ```text
       { "last_execution_id": "{{ $json.id }}", "error_details": {{ $json.data.resultData.error.message }}}
       ```
   - Use the same Watchflow credential.
   - Connect the **false** output of `Is Successful?` → `Watchflow: Fail`.

10. **Optionally add documentation sticky notes**
    - Add a sticky note for setup requirements with:
      - Watchflow node installation
      - Watchflow API key location
      - n8n API key location
      - n8n API base URL must include `/v1/api`
    - Add sticky notes for:
      - Daily Health Audit
      - Dynamic Discovery
      - Health Evaluator
      - Watchflow Sync

11. **Activate the workflow**
    - Save the workflow.
    - Activate it so the schedule trigger starts running automatically.

12. **Validate behavior**
    - Confirm the n8n API node can list workflows tagged `daily`.
    - Confirm at least one monitored workflow has execution history.
    - Test one successful and one failing monitored workflow if possible.
    - Verify records appear correctly in Watchflow.

## Recommended hardening when rebuilding

To make the workflow safer in production, consider these improvements even though they are not present in the original export:

1. **Enable `returnAll` in `Get All Workflows`**
   - Prevent missing workflows when many are tagged `daily`.

2. **Handle workflows with no executions**
   - Add an If node after `Get Last Execution` to detect empty results and optionally report a “never run” state.

3. **Protect the fail payload**
   - Use JSON-safe construction for `error_details` to avoid broken payloads when the error message contains quotes.

4. **Normalize Watchflow keys more robustly**
   - Replace special characters, trim leading/trailing dashes, and consider appending workflow ID for uniqueness.

5. **Differentiate non-success statuses**
   - Add branching for `error`, `crashed`, `running`, or `waiting` if those statuses matter operationally.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the custom Watchflow node package: `@watchflow/n8n-nodes-watchflow` | n8n custom node requirement |
| Get your Watchflow API key from your Watchflow account settings | `https://watchflow.io` |
| Create an n8n API key from your n8n instance settings | n8n instance configuration |
| The n8n API base URL must include the `/v1/api` suffix | Example: `https://n8n.domain.com/v1/api` |

## Additional implementation observations

- The workflow has **one entry point**: `Every Day`.
- It does **not** invoke any sub-workflows.
- Monitoring scope is limited to workflows that are both:
  - active
  - tagged `daily`
- The workflow checks only the **latest execution**, not aggregate reliability over time.
- Failure reporting assumes the latest non-success execution contains `data.resultData.error.message`. If that assumption is false, the fail branch may itself error or send incomplete data.