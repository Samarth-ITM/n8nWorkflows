Automatically prune n8n execution history

https://n8nworkflows.xyz/workflows/automatically-prune-n8n-execution-history-2646


# Automatically prune n8n execution history

### 1. Workflow Overview

This workflow is designed to automatically prune old n8n workflow executions, helping maintain an optimized and clutter-free n8n environment. By routinely deleting executions that exceed a configurable retention period (defaulted to 10 days), it reduces storage use and enhances overall system performance.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Captures manual or scheduled triggers to initiate pruning.
- **1.2 Execution Retrieval**: Fetches all current workflow executions from the n8n instance.
- **1.3 Decision Making**: Determines which executions are older than the configured retention period.
- **1.4 Execution Pruning**: Deletes executions identified as expired and leaves others untouched.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block starts the workflow execution either on demand or automatically on a schedule.

**Nodes Involved:**  
- Manual Trigger (“When clicking ‘Test workflow’”)  
- Schedule Trigger

**Node Details:**

- **Manual Trigger (When clicking ‘Test workflow’)**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing or immediate pruning runs.  
  - Configuration: No parameters set; triggers on user action.  
  - Input: None  
  - Output: Connected to the “n8n list execution” node.  
  - Edge cases: None specific; user must manually trigger.  
  - Version: 1

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically starts the workflow daily at 4:44 AM.  
  - Configuration: Set to run once per day at 04:44 (hour and minute specified).  
  - Input: None  
  - Output: Connected to the “n8n list execution” node.  
  - Edge cases: Workflow will not run if n8n instance is offline at trigger time. Scheduling can be adjusted as needed.  
  - Version: 1.2

---

#### 1.2 Execution Retrieval

**Overview:**  
This block retrieves all workflow executions from the n8n instance to evaluate which ones need pruning.

**Nodes Involved:**  
- n8n list execution (two instances: “n8n list execution” and “n8n1” — only “n8n list execution” is connected in the main flow)  

**Node Details:**

- **n8n list execution**  
  - Type: n8n API Node (resource: execution)  
  - Role: Retrieves all executions without filters, returning the complete list.  
  - Configuration:  
    - Resource: execution  
    - Return All: true (fetches all executions)  
    - Filters: none (fetch everything)  
    - Credentials: Uses “n8n account” API credentials for authentication.  
  - Input: From either Manual Trigger or Schedule Trigger nodes.  
  - Output: Connected to the “If” node.  
  - Edge cases: Large numbers of executions may cause longer processing times or timeouts. Authentication failures possible if credentials expire or change.  
  - Version: 1

- **n8n1** (Not connected in main flow)  
  - Appears to be an unused duplicate of the n8n API execution list node.  
  - No input or output connections.  
  - Can be considered redundant and possibly removed.

---

#### 1.3 Decision Making

**Overview:**  
Evaluates each execution’s start time and decides if it exceeds the retention period (older than 10 days) and thus should be deleted.

**Nodes Involved:**  
- If

**Node Details:**

- **If**  
  - Type: If Node (Version 2.2)  
  - Role: Checks if each execution’s `startedAt` timestamp is older than 10 days ago.  
  - Configuration:  
    - Condition Type: DateTime comparison  
    - Left Value: `{{$json.startedAt}}` (execution start timestamp)  
    - Operator: before  
    - Right Value: `{{ new Date(Date.now() - 10 * 24 * 60 * 60 * 1000).toISOString() }}` (10 days ago in ISO string)  
  - Input: Receives all executions from “n8n list execution” node.  
  - Output: Two branches:  
    - True: executions older than 10 days  
    - False: executions within retention period  
  - Edge cases:  
    - If execution data lacks a valid `startedAt` field, comparison fails.  
    - Timezone inconsistencies may affect date evaluation if system time is off.  
    - Expression errors if JavaScript expression syntax changes in future n8n versions.  
  - Version: 2.2

---

#### 1.4 Execution Pruning

**Overview:**  
Deletes executions flagged as old by the “If” node and performs no operation on recent executions.

**Nodes Involved:**  
- delete execution  
- No Operation, do nothing

**Node Details:**

- **delete execution**  
  - Type: n8n API Node (resource: execution, operation: delete)  
  - Role: Deletes an execution with the specified ID.  
  - Configuration:  
    - Resource: execution  
    - Operation: delete  
    - ExecutionId: `{{$json.id}}` (ID of the execution to delete)  
    - Credentials: Uses “n8n account” API credentials.  
  - Input: Receives executions from the True branch of the “If” node.  
  - Output: None (end of branch).  
  - Edge cases:  
    - API authentication failure or insufficient permissions causes deletion failure.  
    - If execution ID is invalid or deleted by another process, deletion will error.  
    - Rate limits on API calls may cause delays or failures if many executions are deleted at once.  
  - Version: 1

- **No Operation, do nothing**  
  - Type: No Operation Node  
  - Role: Placeholder to explicitly do nothing for executions within retention period.  
  - Configuration: None  
  - Input: Receives executions from the False branch of the “If” node.  
  - Output: None (end of branch).  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                       | Input Node(s)                | Output Node(s)               | Sticky Note                                      |
|-----------------------------|---------------------------|------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger            | On-demand workflow start            | None                        | n8n list execution           |                                                 |
| Schedule Trigger             | Schedule Trigger          | Scheduled daily workflow start      | None                        | n8n list execution           |                                                 |
| n8n list execution           | n8n API Node (execution)  | Retrieve all workflow executions    | Manual Trigger, Schedule Trigger | If                          |                                                 |
| If                          | If Node (v2.2)            | Determine if executions exceed retention | n8n list execution           | delete execution (True), No Operation (False) |                                                 |
| delete execution             | n8n API Node (execution)  | Delete executions older than retention period | If (True branch)             | None                        |                                                 |
| No Operation, do nothing     | No Operation              | Placeholder for retained executions | If (False branch)            | None                        |                                                 |
| n8n1                        | n8n API Node (execution)  | Unused duplicate node               | None                        | None                        | Can be removed; not connected in workflow       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Add node type: Manual Trigger  
   - Name it: “When clicking ‘Test workflow’”  
   - No parameters needed.

2. **Create Schedule Trigger node**  
   - Add node type: Schedule Trigger  
   - Name it: “Schedule Trigger”  
   - Configure to run daily at 04:44 AM (set triggerAtHour to 4, triggerAtMinute to 44).

3. **Create n8n API node to list executions**  
   - Add node type: n8n API (resource: execution, operation: list)  
   - Name it: “n8n list execution”  
   - Set parameters:  
     - Resource: execution  
     - Return All: true (to fetch all executions)  
     - Filters: none  
   - Set credentials: Use your configured n8n API credentials (e.g., “n8n account”).

4. **Connect Manual Trigger and Schedule Trigger nodes to the n8n list execution node**  
   - From “When clicking ‘Test workflow’” → to → “n8n list execution”  
   - From “Schedule Trigger” → to → “n8n list execution”

5. **Create If node**  
   - Add node type: If node (use version 2.2)  
   - Name it: “If”  
   - Configure condition:  
     - Condition type: DateTime comparison  
     - Left value: Expression `{{$json.startedAt}}`  
     - Operator: before  
     - Right value: Expression `{{ new Date(Date.now() - 10 * 24 * 60 * 60 * 1000).toISOString() }}`  
   - Connect “n8n list execution” output to “If” node input.

6. **Create delete execution node**  
   - Add node type: n8n API (resource: execution, operation: delete)  
   - Name it: “delete execution”  
   - Set parameters:  
     - ExecutionId: expression `{{$json.id}}`  
   - Set credentials: Use the same n8n API credentials.  
   - Connect “If” node’s True output to this node.

7. **Create No Operation node**  
   - Add node type: No Operation  
   - Name it: “No Operation, do nothing”  
   - Connect “If” node’s False output to this node.

8. **Verify all nodes are connected properly**  
   - Manual Trigger → n8n list execution  
   - Schedule Trigger → n8n list execution  
   - n8n list execution → If  
   - If (True) → delete execution  
   - If (False) → No Operation

9. **Set up credentials**  
   - Ensure your n8n API credentials (“n8n account”) are correctly configured with sufficient permissions to list and delete executions.

10. **Test the workflow**  
    - Use the Manual Trigger to run a test pruning.  
    - Monitor execution logs for errors or API failures.

11. **Optional customization**  
    - Modify the retention period by adjusting the expression in the If node's right value: replace `10 * 24 * 60 * 60 * 1000` with the desired number of days in milliseconds.  
    - Adjust Schedule Trigger timing as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                   |
|----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The retention period expression uses JavaScript Date arithmetic to calculate the threshold date dynamically.  | If you want a different period, change the multiplier accordingly.|
| The workflow uses the n8n API node and requires an API credential with execution read and delete permissions. | Ensure your API credentials are up to date and have correct scopes.|
| No Operation node is used as a clean design pattern for “do nothing” cases to keep the flow explicit.          | This improves readability and future extensibility.              |
| The Schedule Trigger runs daily early morning (04:44 AM) to minimize load during peak hours.                   | Ideal for production environments to avoid interference.         |

---

This structured reference document should provide a clear and detailed understanding of the “Automatically prune n8n execution history” workflow. It also enables easy reproduction, modification, and troubleshooting by advanced users or automation systems.