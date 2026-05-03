Prevent Concurrent Workflow Runs Using Redis

https://n8nworkflows.xyz/workflows/prevent-concurrent-workflow-runs-using-redis-3976


# Prevent Concurrent Workflow Runs Using Redis

### 1. Workflow Overview

This workflow is designed to prevent concurrent executions of the same long-running job by using Redis as a locking mechanism. It ensures that only one instance of a workflow runs at a time for a given job key, protecting data integrity and avoiding rate-limit breaches. Additionally, it tracks and exposes live progress states (like "working", "loading", "finishing") which can be polled to monitor job status in real time.

**Target Use Cases:**  
- Avoiding parallel runs triggered by webhooks, cron jobs, or nested workflow calls  
- Implementing lightweight locking without full queue systems  
- Tracking progress of long jobs with live status updates  
- Rate-limit protection for external API calls  
- Filtering burst callback spam (e.g., Telegram bots)  
- Ensuring maintenance tasks don’t overlap  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives trigger inputs (`action`, `key`, `value`, `timeout`) from other workflows.  
- **1.2 Action Routing:** Uses a Switch node to route the flow based on action type (`get`, `set`, `unset`).  
- **1.3 Redis Interaction:** Performs Redis operations (get, set with TTL, delete) on keys named `process_status_<key>`.  
- **1.4 Lock Checking and Control Flow:** Uses If nodes to decide whether a lock exists and whether to continue, stop, or throw an error.  
- **1.5 Long Job Simulation & Progress Updates:** Simulates long jobs with Wait nodes, updating Redis keys to reflect progress states.  
- **1.6 Workflow Completion:** Removes the lock at job completion, allowing future executions.  
- **1.7 Status Checking:** Provides a path to query current progress and handle output states accordingly.  

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
Receives input parameters from another workflow, including the action to perform, the job key identifier, optional value, and timeout duration.

**Nodes Involved:**  
- When Executed by Another Workflow

**Node Details:**  
- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point listening for external calls with inputs `action`, `value`, `key`, `timeout`.  
  - Configuration: Defines expected inputs, no default values (except TTL default later applied).  
  - Inputs: External trigger  
  - Outputs: Passes inputs to Set Timeout node  
  - Failure Modes: Missing or malformed inputs may cause routing issues downstream.  
  - Version: 1.1  

- **Set Timeout**  
  - Type: Set  
  - Role: Assigns a default timeout of 600 seconds if none provided, preserving other inputs.  
  - Configuration: Sets numeric field `timeout` to 600 if not specified.  
  - Inputs: Data from trigger node  
  - Outputs: Passes augmented data to Switch node for action routing  
  - Failure Modes: Expression failures if inputs malformed.  
  - Version: 3.4

---

#### 2.2 Action Routing

**Overview:**  
Decides which Redis operation to execute based on the `action` input (`get`, `set`, `unset`).

**Nodes Involved:**  
- Switch  

**Node Details:**  
- **Switch**  
  - Type: Switch  
  - Role: Routes workflow according to the exact string value of `$json.action`.  
  - Configuration: 3 outputs renamed to `get`, `set`, and `unset` based on strict equality checks.  
  - Inputs: From Set Timeout  
  - Outputs: Directs flow to Redis Get Key, Set Key, or UnSet Key nodes respectively  
  - Failure Modes: Unknown or missing action causes no path selected; no fallback configured.  
  - Version: 3.2  

---

#### 2.3 Redis Interaction

**Overview:**  
Performs the core Redis operations for locking and status tracking: reading, writing (with TTL), and deleting keys.

**Nodes Involved:**  
- Get Key  
- Set Key  
- UnSet Key  

**Node Details:**  
- **Get Key**  
  - Type: Redis  
  - Role: Retrieves the value of the key `process_status_<key>` from Redis.  
  - Configuration: Operation is `get`; key dynamically constructed from `key` input; output stored in property `output`.  
  - Credentials: Uses stored Redis credentials.  
  - Inputs: From Switch node's `get` output  
  - Outputs: Passes data to If nodes for lock checking  
  - Failure Modes: Connection errors, key not found (outputs null), credential issues.  
  - Version: 1  

- **Set Key**  
  - Type: Redis  
  - Role: Sets the Redis key with the specified `value` and TTL (timeout).  
  - Configuration: Operation `set` with TTL enabled; key name built dynamically; TTL defaults to 600 seconds unless overridden.  
  - Credentials: Redis credentials required.  
  - Inputs: From Switch node's `set` output  
  - Outputs: Passes to `set continue` node to mark success  
  - Failure Modes: Redis write failures, TTL misconfiguration, credential errors.  
  - Version: 1  

- **UnSet Key**  
  - Type: Redis  
  - Role: Deletes the Redis key, releasing the lock.  
  - Configuration: Operation `delete`; key built dynamically; no TTL needed.  
  - Credentials: Redis credentials required.  
  - Inputs: From Switch node's `unset` output  
  - Outputs: Passes to `set continue` node  
  - Failure Modes: Redis connection or permission errors.  
  - Version: 1  

- **set continue**  
  - Type: Set  
  - Role: Adds a JSON field `ok` with value `"true"` to indicate success downstream.  
  - Inputs: From Set Key and UnSet Key  
  - Outputs: To further logic or termination nodes  
  - Failure Modes: Minimal  

---

#### 2.4 Lock Checking and Control Flow

**Overview:**  
Determines if a lock exists by checking Redis key outputs and decides whether to proceed or stop execution with a clear error.

**Nodes Involved:**  
- Is Workflow Active (multiple instances)  
- If (multiple instances)  
- Stop and Error (multiple instances)  

**Node Details:**  
- **Is Workflow Active, Is Workflow Active1, Is Workflow Active2, Is Workflow Active3**  
  - Type: Execute Workflow  
  - Role: Calls the same workflow recursively with `action=get` to check the lock status of a given key.  
  - Configuration: Fixed workflowId reference to self template; input `key` is set to a constant `"some_workflow_key"` in examples (to be replaced by actual key in practice); action is always `"get"`.  
  - Inputs: Various points in flow to check lock presence.  
  - Outputs: To corresponding If nodes for null or string comparisons.  
  - Failure Modes: Recursive calls may fail if credentials or workflows misconfigured.  
  - Version: 1.2  

- **If, If1, If2**  
  - Type: If  
  - Role: Check if Redis output is empty (null), indicating no lock.  
  - Configuration: Condition checks if the `output` field from Redis is empty (null).  
  - Inputs: From Redis Get Key or Execute Workflow nodes  
  - Outputs: On true (no lock) continues workflow; on false, triggers Stop and Error node.  
  - Failure Modes: Expression errors if output missing or malformed.  
  - Version: 2.2  

- **Stop and Error, Stop and Error1**  
  - Type: Stop and Error  
  - Role: Stops workflow execution with error message `"Already Executing"`.  
  - Configuration: Custom error message to clearly indicate concurrent execution blocked.  
  - Inputs: From If nodes on false branch (lock present).  
  - Failure Modes: None beyond normal workflow stop.  
  - Version: 1  

---

#### 2.5 Long Job Simulation & Progress Updates

**Overview:**  
Simulates a long-running process using Wait nodes and updates Redis key values to represent progress stages (`started`, `loading`, `finishing`, `working`).

**Nodes Involved:**  
- Wait, Wait1, Wait2, Wait3  
- Set Workflow Active1, Set Workflow "started", Set Workflow "loading", Set Workflow "finishing"  
- Set Workflow Finished1, Set Workflow Finished2  

**Node Details:**  
- **Wait, Wait1, Wait2, Wait3**  
  - Type: Wait  
  - Role: Pauses workflow for testing or simulating time-consuming steps.  
  - Configuration: No delay specified in JSON, assumed default (can be customized).  
  - Inputs: Sequenced from progress update nodes.  
  - Outputs: Next progress update or workflow finish nodes.  
  - Failure Modes: Timeout or webhook failures in edge cases.  
  - Version: 1.1  

- **Set Workflow Active1, Set Workflow "started", Set Workflow "loading", Set Workflow "finishing"**  
  - Type: Execute Workflow  
  - Role: Calls self workflow to set Redis key values representing current progress stage.  
  - Configuration: Each sets `value` to respective status string (`working`, `started`, `loading`, `finishing`), `action`=`set`, `key` fixed as `"some_workflow_key"` in example.  
  - Inputs: From If and Wait nodes in sequential order.  
  - Outputs: Chain to Wait nodes or final unset nodes.  
  - Failure Modes: Recursive call failures or incorrect parameter propagation.  
  - Version: 1.2  

- **Set Workflow Finished1, Set Workflow Finished2**  
  - Type: Execute Workflow  
  - Role: Calls workflow to unset lock key, marking job finished.  
  - Configuration: `action`=`unset`, `key`=`some_workflow_key`.  
  - Inputs: From Wait nodes signaling job completion.  
  - Outputs: Ends the long job simulation.  
  - Failure Modes: Failure to delete lock can cause deadlocks.  
  - Version: 1.2  

---

#### 2.6 Workflow Completion

**Overview:**  
Removes the Redis lock key to allow subsequent workflow executions.

**Nodes Involved:**  
- UnSet Key  
- Set Workflow Finished, Set Workflow Finished1, Set Workflow Finished2 (all Execute Workflow nodes invoking unset action)

**Node Details:**  
- Already covered in Redis Interaction and Long Job Simulation blocks.  
- Key action is `unset`, which deletes the Redis key and clears the lock.

---

#### 2.7 Status Checking

**Overview:**  
Allows polling the current workflow execution status by reading the Redis key and routing based on the stored progress value.

**Nodes Involved:**  
- Is Workflow Active3  
- Switch1  

**Node Details:**  
- **Is Workflow Active3**  
  - Type: Execute Workflow  
  - Role: Calls self workflow to get the current Redis key value for status.  
  - Inputs: Triggered externally or from internal logic to fetch status.  
  - Outputs: Passes `output` to Switch1 node.  

- **Switch1**  
  - Type: Switch  
  - Role: Routes based on the Redis status value (`started`, `loading`, `finished`, or others).  
  - Configuration: Renamed outputs for each known status; fallback output for unknown states.  
  - Inputs: From Is Workflow Active3  
  - Outputs: Can be linked to UI or other logic to display or act based on status.  

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                                 | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                 |
|-------------------------------|-----------------------------|------------------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger    | Entry point receiving external workflow inputs | Manual Trigger (optional)       | Set Timeout                   |                                                                                            |
| Set Timeout                   | Set                         | Assigns default TTL and prepares inputs         | When Executed by Another Workflow | Switch                       |                                                                                            |
| Switch                       | Switch                      | Routes flow based on action input                | Set Timeout                    | Get Key, Set Key, UnSet Key    |                                                                                            |
| Get Key                      | Redis                       | Reads Redis key status                            | Switch (get)                  | Is Workflow Active, If2        |                                                                                            |
| Set Key                      | Redis                       | Sets Redis key with value and TTL                 | Switch (set)                  | set continue                  |                                                                                            |
| UnSet Key                    | Redis                       | Deletes Redis key (removes lock)                  | Switch (unset)                | set continue                  |                                                                                            |
| set continue                 | Set                         | Marks successful Redis operation                   | Set Key, UnSet Key            | Various downstream             |                                                                                            |
| Is Workflow Active           | Execute Workflow            | Calls workflow recursively to check lock status  | Get Key                      | If2                          |                                                                                            |
| If2                         | If                          | Checks if Redis output is empty (lock absent)     | Is Workflow Active            | Set Workflow Active, Stop and Error |                                                                                            |
| Set Workflow Active          | Execute Workflow            | Sets lock key to "working" status                  | If2 (true)                   | No Operation, do nothing      |                                                                                            |
| Stop and Error               | Stop and Error              | Stops workflow if lock exists (error)             | If2 (false)                  | None                         |                                                                                            |
| No Operation, do nothing     | NoOp                        | Placeholder to chain workflow                       | Set Workflow Active           | Wait                         |                                                                                            |
| Wait                        | Wait                        | Simulates long running process                      | No Operation, do nothing      | Set Workflow Finished1        |                                                                                            |
| Set Workflow Finished1       | Execute Workflow            | Unsets lock key signaling job completion           | Wait                         | None                         |                                                                                            |
| Is Workflow Active1          | Execute Workflow            | Checks if lock key exists                            | Various                      | If                           |                                                                                            |
| If                          | If                          | Checks if Redis output is empty (lock absent)      | Is Workflow Active1           | Set Workflow Active1, Stop and Error |                                                                                            |
| Set Workflow Active1         | Execute Workflow            | Sets lock key to "working"                           | If (true)                    | No Operation, do nothing      |                                                                                            |
| Stop and Error1              | Stop and Error              | Stops workflow if lock exists                        | If (false)                   | None                         |                                                                                            |
| Set Workflow "started"       | Execute Workflow            | Sets Redis status to "started"                       | If1 (true)                   | Wait1                        |                                                                                            |
| Wait1                       | Wait                        | Simulates step delay                                  | Set Workflow "started"        | Set Workflow "loading"         |                                                                                            |
| Set Workflow "loading"       | Execute Workflow            | Updates Redis status to "loading"                     | Wait1                        | Wait2                        |                                                                                            |
| Wait2                       | Wait                        | Simulates step delay                                  | Set Workflow "loading"        | Set Workflow "finishing"       |                                                                                            |
| Set Workflow "finishing"     | Execute Workflow            | Updates Redis status to "finishing"                   | Wait2                        | Wait3                        |                                                                                            |
| Wait3                       | Wait                        | Simulates step delay                                  | Set Workflow "finishing"      | Set Workflow Finished2         |                                                                                            |
| Set Workflow Finished2       | Execute Workflow            | Removes Redis lock key signaling final completion    | Wait3                        | None                         |                                                                                            |
| Is Workflow Active2          | Execute Workflow            | Checks lock status                                   | Various                      | If1                         |                                                                                            |
| If1                         | If                          | Checks if Redis output is empty (lock absent)      | Is Workflow Active2           | Set Workflow "started", Stop and Error1 |                                                                                            |
| Is Workflow Active3          | Execute Workflow            | Fetches current status for polling                    | External or internal trigger  | Switch1                      |                                                                                            |
| Switch1                     | Switch                      | Routes based on current Redis status value            | Is Workflow Active3           | (started, loading, finished, extra) |                                                                                            |
| Set Workflow Finished        | Execute Workflow            | Unsets lock key to mark workflow end                  | Wait                         | None                         |                                                                                            |
| Sticky Notes (multiple)      | Sticky Note                 | Provides descriptive comments and guidance             | N/A                          | N/A                          | Multiple notes provide explanations on blocks, Redis logic, usage examples, and testing    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Execute Workflow Trigger node**  
   - Name: `When Executed by Another Workflow`  
   - Parameters: Define inputs `action`, `value`, `key`, `timeout`.  
   - Purpose: Entry point for external workflow calls.

2. **Add a Set node**  
   - Name: `Set Timeout`  
   - Assign numeric field `timeout` with default `600` if not provided.  
   - Pass through all other input fields unchanged.

3. **Add a Switch node**  
   - Name: `Switch`  
   - Configure three outputs:  
     - `get`: if `$json.action === "get"`  
     - `set`: if `$json.action === "set"`  
     - `unset`: if `$json.action === "unset"`  

4. **Add Redis node for GET operation**  
   - Name: `Get Key`  
   - Operation: `get`  
   - Key: `process_status_{{$json.key}}` (dynamic key)  
   - Credentials: Select your Redis credentials.

5. **Add Redis node for SET operation**  
   - Name: `Set Key`  
   - Operation: `set`  
   - Key: `process_status_{{$json.key}}`  
   - Value: `{{$json.value}}`  
   - TTL: `{{$json.timeout}}` seconds  
   - Enable expiration.

6. **Add Redis node for DELETE operation**  
   - Name: `UnSet Key`  
   - Operation: `delete`  
   - Key: `process_status_{{$json.key}}`

7. **Connect Switch outputs**:  
   - `get` → `Get Key`  
   - `set` → `Set Key`  
   - `unset` → `UnSet Key`

8. **Add a Set node**  
   - Name: `set continue`  
   - Add field `ok` with value `"true"`  
   - Connect outputs of `Set Key` and `UnSet Key` to this node.

9. **Add Execute Workflow nodes to call self recursively for lock checks**  
   - Create a workflow call node named `Is Workflow Active` (and variants) calling this same workflow with inputs:  
     - `action`: `"get"`  
     - `key`: Use the job key dynamically  
   - These nodes are used to check lock state at various points.

10. **Add If nodes to check if Redis output is empty**  
    - Condition: `$json.output` is empty (null)  
    - True branch: No lock present, proceed  
    - False branch: Lock present, stop with error

11. **Add Stop and Error nodes**  
    - Name: `Stop and Error`  
    - Error message: `"Already Executing"`  
    - Connected to false branches of If nodes to fail workflow when concurrent execution detected.

12. **Add No Operation (NoOp) node**  
    - Used as a pass-through placeholder for chaining.

13. **Add Wait nodes**  
    - Add several Wait nodes (`Wait`, `Wait1`, `Wait2`, `Wait3`) to simulate job progress delays.

14. **Add Execute Workflow nodes to update Redis status during job**  
    - Set status to `"working"`, `"started"`, `"loading"`, `"finishing"` at different job steps.  
    - Call self workflow with `action="set"`, `key`, and corresponding `value` to update Redis.

15. **Add Execute Workflow nodes to unset key for job completion**  
    - Call self with `action="unset"` and `key` to remove lock.

16. **Add final status polling nodes**  
    - Execute Workflow node `Is Workflow Active3` to fetch current Redis key value.  
    - Switch node `Switch1` to route based on status (`started`, `loading`, `finished`, or fallback).

17. **Wire all nodes according to the logical flow described in section 1 and 2.**

18. **Configure Redis credentials in all Redis nodes.**

19. **Test workflow with manual trigger or external calls providing `action`, `key`, and optional `value` and `timeout`.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow template helps prevent concurrent runs by setting a Redis lock and tracking progress status with TTL keys.                                                      | Workflow description and use cases                                                                                  |
| Customize the TTL in the `Set Timeout` node to match typical job execution duration.                                                                                           | Customization tips                                                                                                  |
| Replace the Stop and Error nodes with Slack or email notifications, or push triggers into a queue to avoid failing parallel calls.                                           | Customization tips                                                                                                  |
| Build a status endpoint by calling this workflow with `action=get` to display real-time job progress to users.                                                                | Customization tips                                                                                                  |
| Use this workflow to protect against Telegram callback spam bursts, API rate-limit violations, and overlapping maintenance windows.                                           | Additional use cases                                                                                                |
| Redis credentials must be configured in n8n beforehand and reachable from the workflow runtime environment.                                                                   | Prerequisites                                                                                                       |
| The workflow uses recursive calls to itself for status checking and updating, so ensure correct input parameter passing and looping control to avoid infinite recursion.      | Execution considerations                                                                                            |
| Sticky notes in the workflow visually explain the Redis logic, usage examples, and testing instructions for easier understanding and modification.                            | Sticky notes content                                                                                                |
| Redis key names are prefixed with `process_status_` followed by the user-provided key, enabling multiple independent locks.                                                  | Redis key naming convention                                                                                         |