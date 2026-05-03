Prevent simultaneous workflow executions with Redis

https://n8nworkflows.xyz/workflows/prevent-simultaneous-workflow-executions-with-redis-2270


# Prevent simultaneous workflow executions with Redis

### 1. Workflow Overview

This workflow prevents simultaneous executions of a scheduled main workflow by using Redis as a distributed lock system. It is designed for scenarios where the main workflow's execution time may exceed its scheduled interval, which could cause overlapping runs and potential data or process conflicts.

The workflow is divided into the following logical blocks:

- **1.1 Schedule Trigger**: Periodically checks whether the main workflow is idle or running.
- **1.2 Redis Status Check**: Reads a Redis key that acts as a flag indicating if the main workflow is currently running or idle.
- **1.3 Condition Evaluation**: Determines the next action based on the Redis key value.
- **1.4 Workflow Execution Control**: Controls the execution of the main workflow and updates the Redis flag accordingly.
- **1.5 Manual Reset**: Provides a manual method to reset the Redis flag to idle, useful for troubleshooting or after server outages.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger Block

- **Overview:**  
  Periodically triggers the workflow every 5 seconds to check the status of the main workflow.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow execution on a fixed interval.  
    - Configuration: Interval set to every 5 seconds.  
    - Inputs: None (trigger node).  
    - Outputs: Connected to "Get Status" node.  
    - Edge Cases: Misconfiguration of interval can cause too frequent or too sparse checks.  
    - Notes: Can be adjusted to any interval depending on workflow requirements (see Sticky Note2).

---

#### 2.2 Redis Status Check Block

- **Overview:**  
  Reads the Redis key to determine if the main workflow is running or idle.

- **Nodes Involved:**  
  - Get Status  
  - Redis Key exists  

- **Node Details:**  
  - **Get Status**  
    - Type: Redis Node (Get Operation)  
    - Role: Retrieves the current status flag from Redis.  
    - Configuration:  
      - Key dynamically set as `workflowStatus_{{ $workflow.id }}`, ensuring uniqueness per workflow.  
      - Property name to store value: `workflowStatus`.  
    - Credentials: Uses Redis credentials configured for the Redis instance.  
    - Inputs: From Schedule Trigger.  
    - Outputs: To "Redis Key exists".  
    - Edge Cases: Redis connection failure, missing key (returns empty).  
    
  - **Redis Key exists**  
    - Type: If Node  
    - Role: Checks if the Redis key exists and is non-empty.  
    - Configuration: Checks if `workflowStatus` is not empty.  
    - Inputs: From "Get Status".  
    - Outputs:  
      - True branch: to "Continue if Idle".  
      - False branch: to "No Operation".  
    - Edge Cases: Expression failures if `$json.workflowStatus` is missing.

---

#### 2.3 Condition Evaluation Block

- **Overview:**  
  Decides whether to continue with execution depending on the Redis status value.

- **Nodes Involved:**  
  - Continue if Idle  
  - No Operation

- **Node Details:**  
  - **Continue if Idle**  
    - Type: Filter Node  
    - Role: Allows workflow continuation only if Redis status equals "idle".  
    - Configuration: Condition is `$json.workflowStatus === "idle"`.  
    - Inputs: From "Redis Key exists" (true branch).  
    - Outputs: To "Set Running".  
    - Edge Cases: Case sensitivity in status string may cause logic errors.  
  - **No Operation**  
    - Type: No Operation Node  
    - Role: Ends workflow path when Redis key does not exist or is empty (assumed idle).  
    - Inputs: From "Redis Key exists" (false branch).  
    - Outputs: To "Set Running".  
    - Notes: Ensures that a missing Redis key still triggers the workflow execution.

---

#### 2.4 Workflow Execution Control Block

- **Overview:**  
  Manages the lifecycle flag in Redis and triggers the main workflow accordingly.

- **Nodes Involved:**  
  - Set Running  
  - Execute Workflow  
  - Set Idle  

- **Node Details:**  
  - **Set Running**  
    - Type: Redis Node (Set Operation)  
    - Role: Sets Redis key to "running" to indicate the workflow is executing.  
    - Configuration: Key is `workflowStatus_{{ $workflow.id }}`, value is `"running"`.  
    - Inputs: From "Continue if Idle" and "No Operation".  
    - Outputs: To "Execute Workflow".  
    - Edge Cases: Redis set failure may lead to duplicate executions.  
    - Sticky Note1: "This updates the flag, indicating, that the workflow is currently running."  
  - **Execute Workflow**  
    - Type: Execute Workflow Node  
    - Role: Calls the main workflow by workflow ID.  
    - Configuration: Workflow ID is set to the main workflow's ID.  
    - Inputs: From "Set Running".  
    - Outputs: To "Set Idle".  
    - Edge Cases: If the main workflow errors, this node is configured to continue output to avoid blocking flag reset.  
    - Sticky Note4: "Set the ID of the main workflow which should be executed."  
  - **Set Idle**  
    - Type: Redis Node (Set Operation)  
    - Role: Sets Redis key to "idle" after main workflow completes.  
    - Configuration: Key is `workflowStatus_{{ $workflow.id }}`, value `"idle"`.  
    - Inputs: From "Execute Workflow".  
    - Outputs: None (workflow end).  
    - Edge Cases: Redis set failure could cause the workflow to appear running indefinitely.  
    - Sticky Note6: "This updates the flag, indicating, that the workflow is currently idle."

---

#### 2.5 Manual Reset Block (for Troubleshooting)

- **Overview:**  
  Allows manual resetting of the Redis status flag to "idle" to recover from errors or outages.

- **Nodes Involved:**  
  - When clicking "Test workflow" (Manual Trigger)  
  - Reset to Idle

- **Node Details:**  
  - **When clicking "Test workflow"**  
    - Type: Manual Trigger  
    - Role: Allows manual activation of the reset process.  
    - Configuration: Disabled by default; enable when needed.  
    - Inputs: None  
    - Outputs: To "Reset to Idle"  
  - **Reset to Idle**  
    - Type: Redis Node (Set Operation)  
    - Role: Resets Redis key to "idle" manually.  
    - Configuration: Key is `workflowStatus_{{ $workflow.id }}`, value `"idle"`.  
    - Inputs: From manual trigger.  
    - Outputs: None  
    - Credentials: Uses same Redis credentials.  
    - Edge Cases: Should only be used when safe to reset, to avoid premature workflow triggers.  
  - Sticky Note: "Unplanned server outage? Need to reset the flag? Disable the schedule trigger, activate these nodes and run the **Reset to Idle** node manually."

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                                  | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                      |
|-------------------------------|---------------------------|-------------------------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger          | Periodically triggers workflow                   | -                       | Get Status              | ## Set Interval\nDefine how frequently the main workflow should run.                                            |
| Get Status                    | Redis                     | Reads Redis flag for workflow status             | Schedule Trigger         | Redis Key exists        | This checks for a dynamic flag (containing the workflow ID) which represents if the workflow is currently running.|
| Redis Key exists              | If                        | Checks if Redis key exists and is non-empty      | Get Status               | Continue if Idle, No Operation | If the flag stored in Redis already exists and indicates that the workflow is still running, another execution will be prevented. In that case this workflow ends here. |
| Continue if Idle              | Filter                    | Continues only if Redis status is "idle"         | Redis Key exists         | Set Running             |                                                                                                                  |
| No Operation                 | No Operation              | Ends flow if Redis key does not exist            | Redis Key exists         | Set Running             |                                                                                                                  |
| Set Running                  | Redis                     | Sets Redis flag to "running"                      | Continue if Idle, No Operation | Execute Workflow      | This updates the flag, indicating, that the workflow is currently running                                       |
| Execute Workflow             | Execute Workflow          | Executes the main workflow                         | Set Running              | Set Idle                | Set Workflow ID\nSet the ID of the main workflow which should be executed                                        |
| Set Idle                    | Redis                     | Sets Redis flag to "idle" after execution         | Execute Workflow         | -                       | This updates the flag, indicating, that the workflow is currently idle                                          |
| When clicking "Test workflow" | Manual Trigger            | Manual trigger to reset Redis flag                | -                       | Reset to Idle           | Unplanned server outage? Need to reset the flag? Disable the schedule trigger, activate these nodes and run the **Reset to Idle** node manually. |
| Reset to Idle               | Redis                     | Manually resets Redis flag to "idle"              | When clicking "Test workflow" | -                    |                                                                                                                  |
| Sticky Note                  | Sticky Note               | Notes for troubleshooting                         | -                       | -                       | Unplanned server outage? Need to reset the flag? Disable the schedule trigger, activate these nodes and run the **Reset to Idle** node manually. |
| Sticky Note1                 | Sticky Note               | Notes on setting "running" flag                   | -                       | -                       | This updates the flag, indicating, that the workflow is currently running                                       |
| Sticky Note2                 | Sticky Note               | Notes on setting schedule interval                | -                       | -                       | ## Set Interval\nDefine how frequently the main workflow should run.                                            |
| Sticky Note3                 | Sticky Note               | Notes on Redis key existence check                | -                       | -                       | If the flag stored in Redis already exists and indicates, that the workflow is still running, another execution will be prevented. In that case this workflow ends here. |
| Sticky Note4                 | Sticky Note               | Notes on setting main workflow ID                  | -                       | -                       | ## Set Workflow ID\nSet the ID of the main workflow which should be executed                                    |
| Sticky Note5                 | Sticky Note               | Notes on checking Redis flag                       | -                       | -                       | This checks for a dynamic flag (containing the workflow ID) which represents if the workflow is currently running.|
| Sticky Note6                 | Sticky Note               | Notes on setting "idle" flag                        | -                       | -                       | This updates the flag, indicating, that the workflow is currently idle                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Configuration: Set interval to trigger every 5 seconds (adjustable as needed).  
   - Position: Start of workflow.

2. **Create "Get Status" node**  
   - Type: Redis node (Get operation)  
   - Credentials: Configure with your Redis instance credentials.  
   - Parameters:  
     - Key: `workflowStatus_{{ $workflow.id }}` (dynamic with workflow id)  
     - Operation: Get  
     - Property name: `workflowStatus` (to store Redis output)  
   - Connect output of "Schedule Trigger" to input of "Get Status".

3. **Create "Redis Key exists" node**  
   - Type: If node  
   - Parameters:  
     - Condition: Check if `{{$json.workflowStatus}}` is not empty (notEmpty operator).  
   - Connect output of "Get Status" to input of this node.

4. **Create "Continue if Idle" node**  
   - Type: Filter node  
   - Parameters:  
     - Condition: Check if `{{$json.workflowStatus}}` equals `"idle"`.  
   - Connect True output of "Redis Key exists" to "Continue if Idle".

5. **Create "No Operation" node**  
   - Type: No Operation node  
   - Connect False output of "Redis Key exists" to "No Operation".

6. **Create "Set Running" node**  
   - Type: Redis node (Set operation)  
   - Credentials: Use Redis credentials.  
   - Parameters:  
     - Key: `workflowStatus_{{ $workflow.id }}`  
     - Value: `"running"`  
     - Operation: Set  
   - Connect output of both "Continue if Idle" and "No Operation" to "Set Running".

7. **Create "Execute Workflow" node**  
   - Type: Execute Workflow node  
   - Parameters:  
     - Workflow ID: Paste the ID of your main workflow here.  
     - On Error: Set to "Continue Regular Output" to ensure flag reset happens even if errors occur.  
   - Connect output of "Set Running" to "Execute Workflow".

8. **Create "Set Idle" node**  
   - Type: Redis node (Set operation)  
   - Credentials: Use Redis credentials.  
   - Parameters:  
     - Key: `workflowStatus_{{ $workflow.id }}`  
     - Value: `"idle"`  
     - Operation: Set  
   - Connect output of "Execute Workflow" to "Set Idle".

9. **(Optional) Create "When clicking 'Test workflow'" node**  
   - Type: Manual Trigger (disabled by default)  
   - Use only for manual reset purposes.

10. **(Optional) Create "Reset to Idle" node**  
    - Type: Redis node (Set operation)  
    - Credentials: Use Redis credentials.  
    - Parameters:  
      - Key: `workflowStatus_{{ $workflow.id }}`  
      - Value: `"idle"`  
      - Operation: Set  
    - Connect output of manual trigger to this node.

**Additional setup instructions:**  
- Replace the Schedule Trigger in your main workflow with an "Execute Workflow" node that calls this Redis lock workflow.  
- Copy your main workflow ID from n8n URL and paste it in the "Execute Workflow" node of this workflow.  
- Ensure Redis credentials are properly configured and accessible by all Redis nodes in this workflow.  
- Adjust the schedule interval according to your timing needs (see Sticky Note2).  
- For troubleshooting, disable the schedule trigger and manually reset the flag using the manual trigger and "Reset to Idle" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Unplanned server outage? Need to reset the flag? Disable the schedule trigger, activate these nodes and run the **Reset to Idle** node manually. | Troubleshooting instructions included as Sticky Note in the workflow.                                  |
| https://docs.n8n.io/nodes/n8n-nodes-base.redis/                                                               | Official n8n Redis node documentation for reference on configuration and operations.                    |
| https://docs.n8n.io/nodes/n8n-nodes-base.executeworkflow/                                                     | Official n8n Execute Workflow node documentation for proper sub-workflow invocation setup.              |
| Workflow ID can be found at the end of the URL when editing a workflow in n8n.                                | Setup instruction for dynamic referencing of main workflow in the Execute Workflow node.                |

---

This document fully describes the "Prevent simultaneous workflow executions with Redis" workflow, enabling expert users and automation agents to understand, reproduce, and maintain it without ambiguity.