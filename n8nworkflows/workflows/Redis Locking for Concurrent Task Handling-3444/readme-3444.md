Redis Locking for Concurrent Task Handling

https://n8nworkflows.xyz/workflows/redis-locking-for-concurrent-task-handling-3444


# Redis Locking for Concurrent Task Handling

### 1. Workflow Overview

This workflow implements a **Redis-based locking mechanism** to ensure that concurrent or duplicate executions of a workflow do not cause conflicts, race conditions, or redundant processing. It is designed primarily for scenarios where multiple triggers might overlap—such as webhook calls, scheduled runs, or retries—and where only one workflow instance should run at a time to maintain data integrity and efficient API usage.

**Target use cases include:**
- Preventing duplicate database updates
- Avoiding overlapping synchronization processes between tools
- Managing rate-limited API calls by serializing workflow execution
- Implementing idempotency in automation tasks

**Logical blocks:**

- **1.1 Input Reception:** Receives and parses incoming webhook data, extracting variables used to generate a unique lock identifier.
- **1.2 Redis Lock Status Check:** Queries Redis to determine if a lock key already exists, indicating another workflow instance is running.
- **1.3 Lock Acquisition Attempt:** Attempts to set a Redis lock key with a short TTL (time-to-live) to claim exclusive execution rights.
- **1.4 Lock Existence Evaluation and Retry Logic:** Evaluates the lock acquisition result, deciding whether to proceed, wait and retry, or skip duplicate execution.
- **1.5 Main Business Logic Switch:** Routes execution to one of three predefined workflow branches depending on the incoming data.
- **1.6 Lock Release:** Deletes the Redis lock key to free the lock once the workflow task completes.
- **1.7 Workflow Termination:** Ends workflow execution gracefully when duplicate or concurrent runs are detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the webhook payload, parses JSON data, extracts relevant action variables (var1, var2, var3), and constructs a unique `lockValue` string used as the Redis lock key.

- **Nodes Involved:**  
  - Incoming Webhook Data  
  - Fetch Webhook Data & Declare lockValue  

- **Node Details:**

  - **Incoming Webhook Data**  
    - Type: Webhook  
    - Role: Entry point receiving external HTTP POST requests on a specific path.  
    - Configuration: Uses a fixed webhook path (`94d08900-4816-4c74-962a-aacff5077d5d`).  
    - Input: External HTTP request with JSON payload.  
    - Output: Raw webhook data forwarded to next node.  
    - Edge cases: Missing or malformed webhook payload may cause parsing errors downstream.  
    - Version requirements: Webhook v2 node used.

  - **Fetch Webhook Data & Declare lockValue**  
    - Type: Code (JavaScript)  
    - Role: Parses the webhook JSON payload, extracts variables (`var1`, `var2`, `var3`), and creates a `lockValue` string combining these variables with hyphens.  
    - Configuration: Custom JS code accessing the previous node’s `body.payload`.  
    - Key expressions: `const payload = JSON.parse($('Incoming Webhook Data').first().json["body"]["payload"]);`  
      Constructs `lockValue` as `${var1}-${var2}-${var3}`.  
    - Input: Raw JSON from webhook node.  
    - Output: JSON object with extracted vars and `lockValue`.  
    - Edge cases: Payload missing expected fields may cause runtime exceptions; no try/catch implemented.  
    - Version requirements: Code node v2.

#### 2.2 Redis Lock Status Check

- **Overview:**  
  Checks Redis for the existence of a lock key (`xyz-lock`) to determine if another instance currently holds the lock.

- **Nodes Involved:**  
  - Check Redis Lock  
  - redisLock existence boolean  

- **Node Details:**

  - **Check Redis Lock**  
    - Type: Redis  
    - Role: Performs Redis GET operation for key `xyz-lock` to fetch current lock value.  
    - Configuration: Key hardcoded as `xyz-lock`. Operation: `get`.  
    - Credentials: Uses Redis credentials named "Geoffrey Redis".  
    - Input: Lock value from previous code node is not used here; this node queries Redis directly.  
    - Output: Key value or empty if not set.  
    - Edge cases: Redis connection errors, timeouts, or authentication failures. If Redis unavailable, lock check fails silently.  
    - Version requirements: Redis node v1.

  - **redisLock existence boolean**  
    - Type: IF  
    - Role: Checks if Redis GET returned empty (lock does not exist) or not.  
    - Configuration: Condition tests if property `Element` (which holds Redis GET result) is empty string.  
    - Input: Output from "Check Redis Lock".  
    - Output: Two branches: lock does not exist (true), lock exists (false).  
    - Edge cases: If Redis response format changes, condition may fail.  
    - Version requirements: IF node v2.2.

#### 2.3 Lock Acquisition Attempt

- **Overview:**  
  Attempts to acquire the Redis lock by setting the key with a TTL if the lock was free.

- **Nodes Involved:**  
  - Acquire Redis Lock  
  - redisLock acquired booleans  

- **Node Details:**

  - **Acquire Redis Lock**  
    - Type: Redis  
    - Role: Sets the Redis key `xyz-lock` with a TTL of 180 seconds and value derived from the `lockValue` generated earlier.  
    - Configuration: Operation: `set` with TTL enabled (`expire: true`). Value taken from code node output `lookupVariable`.  
    - Input: Flow from IF node confirming lock availability.  
    - Output: Confirmation of successful set operation.  
    - Edge cases: Redis errors when setting key, race conditions where two workflows set simultaneously, TTL handling issues.  
    - Version requirements: Redis node v1.

  - **redisLock acquired booleans**  
    - Type: IF  
    - Role: Tests if the Redis lock key was successfully set (lock acquired) or if it failed (lock still held by someone else).  
    - Configuration: Condition checks if Redis GET result is not empty for existence.  
    - Input: Output from previous `redisLock existence boolean` node.  
    - Output: Branches for lock acquired (true) or retry required (false).  
    - Edge cases: False positives if Redis key expires just after check.  
    - Version requirements: IF node v2.2.

#### 2.4 Lock Existence Evaluation and Retry Logic

- **Overview:**  
  If lock acquisition fails, the workflow waits and retries; if a duplicate request is detected, it skips execution.

- **Nodes Involved:**  
  - Poll for lock  
  - duplicateWebhook boolean  
  - END  

- **Node Details:**

  - **Poll for lock**  
    - Type: Wait  
    - Role: Pauses workflow execution before retrying lock acquisition, implementing retry delay.  
    - Configuration: Defaults (likely fixed wait time; exact duration not specified).  
    - Input: Triggered when lock is held by another workflow instance.  
    - Output: Continues to duplicate request check.  
    - Edge cases: Excessive waits may cause workflow queueing; no max retry count shown.  
    - Version requirements: Wait node v1.1.

  - **duplicateWebhook boolean**  
    - Type: IF  
    - Role: Compares incoming lock value with currently held lock to detect duplicates; decides to skip or continue.  
    - Configuration: Checks if `lockValue` equals Redis key value.  
    - Input: Output from "Poll for lock" and Redis GET results.  
    - Output: True branch (duplicate detected, skip), False branch (no duplicate, retry lock check).  
    - Edge cases: Comparison may fail if Redis value and lockValue formats differ.  
    - Version requirements: IF node v2.2.

  - **END**  
    - Type: NoOp  
    - Role: Terminates workflow for duplicate requests to avoid reprocessing.  
    - Configuration: None.  
    - Input: From duplicateWebhook boolean true branch.  
    - Output: None.  
    - Edge cases: None.

#### 2.5 Main Business Logic Switch

- **Overview:**  
  Routes execution to one of three possible business logic branches based on the lock value or other conditions.

- **Nodes Involved:**  
  - Workflow Switch  
  - Workflow 1  
  - Workflow 2  
  - Workflow 3  
  - Discard Redis Lock  

- **Node Details:**

  - **Workflow Switch**  
    - Type: Switch  
    - Role: Routes flow into one of three branches named Workflow 1, 2, or 3 depending on conditions.  
    - Configuration: Conditions are placeholders (empty strings); likely intended for customization to route based on data.  
    - Input: From successful lock acquisition.  
    - Output: Three outputs linked to "Workflow 1", "Workflow 2", and "Workflow 3".  
    - Edge cases: Without configured conditions, all inputs may route to default or none.  
    - Version requirements: Switch node v3.2.

  - **Workflow 1, Workflow 2, Workflow 3**  
    - Type: Set  
    - Role: Placeholder nodes representing separate business logic branches.  
    - Configuration: Empty options; to be replaced with real tasks.  
    - Input: From "Workflow Switch".  
    - Output: Each connects to "Discard Redis Lock" node to release lock after processing.  
    - Edge cases: Nodes have no operational logic; real workflows should handle failures and outputs.  
    - Version requirements: Set node v3.4.

  - **Discard Redis Lock**  
    - Type: Redis  
    - Role: Deletes the Redis lock key to release the lock, allowing other workflow instances to proceed.  
    - Configuration: Operation `delete` on key `n8n-rca-lock` (note: key name differs from earlier `xyz-lock`, which may be an inconsistency).  
    - Input: From each Workflow branch after completion.  
    - Output: None specified.  
    - Edge cases: If deletion fails, lock may remain, causing deadlock. Key name discrepancy may cause bugs.  
    - Version requirements: Redis node v1.

#### 2.6 Workflow Termination and Documentation

- **Overview:**  
  Provides clarifying annotations and terminates the workflow gracefully on duplicate detection.

- **Nodes Involved:**  
  - Sticky Note (multiple)  
  - END  

- **Node Details:**

  - **Sticky Notes**  
    - Type: Sticky Note  
    - Role: Provide explanations about workflow purpose, specific node functions, and usage tips.  
    - Configuration: Various notes attached in workspace.  
    - Input/Output: None, purely informational.  
    - Edge cases: None.

  - **END**  
    - As described above.

---

### 3. Summary Table

| Node Name                          | Node Type      | Functional Role                            | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                  |
|-----------------------------------|----------------|-------------------------------------------|--------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Incoming Webhook Data              | Webhook        | Receives external webhook HTTP requests  | None                           | Fetch Webhook Data & Declare lockValue |                                                                                              |
| Fetch Webhook Data & Declare lockValue | Code           | Parses webhook payload, extracts vars, creates lockValue | Incoming Webhook Data           | Check Redis Lock                   |                                                                                              |
| Check Redis Lock                  | Redis          | Reads Redis key to check if lock exists  | Fetch Webhook Data & Declare lockValue | redisLock existence boolean       |                                                                                              |
| redisLock existence boolean        | IF             | Checks if lock key is empty (no lock)    | Check Redis Lock               | Acquire Redis Lock, redisLock acquired booleans |                                                                                              |
| redisLock acquired booleans        | IF             | Checks if lock was acquired successfully  | redisLock existence boolean    | Acquire Redis Lock, Poll for lock |                                                                                              |
| Acquire Redis Lock                 | Redis          | Attempts to set Redis lock key with TTL  | redisLock existence boolean, redisLock acquired booleans | Workflow Switch                  | Sticky Note1: "Attempts to acquire a lock using Redis by setting a key with expiration."     |
| Poll for lock                    | Wait           | Waits before retrying lock acquisition   | redisLock acquired booleans    | duplicateWebhook boolean          |                                                                                              |
| duplicateWebhook boolean           | IF             | Checks for duplicate lock values          | Poll for lock                  | END (if duplicate), Check Redis Lock (if not) | Sticky Note2: "Skips execution when duplicate request is received."                          |
| Workflow Switch                   | Switch         | Routes to one of three workflow branches | Acquire Redis Lock             | Workflow 1, Workflow 2, Workflow 3 |                                                                                              |
| Workflow 1                       | Set            | Placeholder for business logic branch 1 | Workflow Switch                | Discard Redis Lock                |                                                                                              |
| Workflow 2                       | Set            | Placeholder for business logic branch 2 | Workflow Switch                | Discard Redis Lock                |                                                                                              |
| Workflow 3                       | Set            | Placeholder for business logic branch 3 | Workflow Switch                | Discard Redis Lock                |                                                                                              |
| Discard Redis Lock                | Redis          | Deletes Redis lock key to release lock   | Workflow 1, Workflow 2, Workflow 3 | None                            | Sticky Note3: "Deletes the Redis lock key to release the lock."                             |
| END                             | NoOp           | Terminates workflow on duplicate detection | duplicateWebhook boolean (true branch) | None                            |                                                                                              |
| Sticky Note                     | Sticky Note    | Explains Redis lock workflow concept     | None                          | None                            | "This workflow demonstrates Redis-based locking to prevent concurrent execution of workflows."|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook (v2)  
   - Parameters: Set path to `94d08900-4816-4c74-962a-aacff5077d5d`.  
   - Purpose: Receive incoming requests that trigger the workflow.

2. **Create Code Node to Parse Webhook Data**  
   - Type: Code (v2)  
   - Parameters:  
     ```javascript
     const payload = JSON.parse($('Incoming Webhook Data').first().json["body"]["payload"]);
     const var1 = payload.var1;
     const var2 = payload.var2;
     const var3 = payload.var3;
     return {
       var1: var1,
       var2: var2,
       var3: var3,
       lockValue: `${var1}-${var2}-${var3}`
     };
     ```  
   - Connect output of Webhook node to this Code node.

3. **Create Redis Node for Lock Check**  
   - Type: Redis (v1)  
   - Operation: `get`  
   - Key: `xyz-lock`  
   - Credentials: Configure Redis credentials pointing to your Redis instance.  
   - Connect Code node output to this node.

4. **Create IF Node to Check Lock Existence**  
   - Type: IF (v2.2)  
   - Condition: Check if Redis GET result (`Element`) is empty string (meaning no lock).  
   - Connect Redis GET node output to this IF node.

5. **Create Redis Node to Acquire Lock**  
   - Type: Redis (v1)  
   - Operation: `set`  
   - Key: `xyz-lock`  
   - Value: Use expression referencing Code node output `lockValue` (e.g., `={{ $json.lockValue }}`).  
   - TTL: 180 seconds (3 minutes)  
   - Expire: true  
   - Credentials: Same Redis credentials.  
   - Connect "lock does not exist" output from IF node to this node.

6. **Create IF Node to Confirm Lock Acquisition**  
   - Type: IF (v2.2)  
   - Condition: Check that Redis GET result from prior check is not empty (lock acquired).  
   - Connect original IF node’s "lock exists" output to this node  
   - Connect this IF node’s True output to the Redis set node (to retry acquiring lock), False output to Wait node.

7. **Create Wait Node for Retry**  
   - Type: Wait (v1.1)  
   - Configure desired wait time (e.g., 10 seconds).  
   - Connect IF node’s False output (lock not acquired) to this node.

8. **Create IF Node to Detect Duplicate Requests**  
   - Type: IF (v2.2)  
   - Condition: Compare incoming lockValue with Redis key value to detect duplicates.  
   - Connect Wait node output to this node.

9. **Create NoOp Node for Workflow End (on duplicate)**  
   - Type: NoOp  
   - Connect IF node’s True branch (duplicate detected) to this node.

10. **Create Switch Node to Route Business Logic**  
    - Type: Switch (v3.2)  
    - Configure three outputs (Workflow 1, 2, 3) with conditions defined by your use case (currently placeholders).  
    - Connect Redis set node output (lock acquired) to this node.

11. **Create Three Set Nodes for Business Logic Stubs**  
    - Type: Set (v3.4)  
    - Configure each with business-specific parameters or replace with real workflow nodes.  
    - Connect Switch node outputs to these three nodes respectively.

12. **Create Redis Node to Discard Lock**  
    - Type: Redis (v1)  
    - Operation: `delete`  
    - Key: `xyz-lock` (note: fix key name to be consistent)  
    - Credentials: Same Redis credentials.  
    - Connect outputs of all three business logic nodes to this node.

13. **Connect Duplicate IF node’s False branch to Redis lock check node**  
    - So that retries loop back to check lock again.

14. **Add Sticky Notes**  
    - Add informational sticky notes to document workflow purpose, key steps, and tips.

15. **Configure Redis Credentials**  
    - Create and assign Redis credentials with connection details to your Redis instance.

16. **Test the Workflow**  
    - Trigger webhook with test payloads.  
    - Monitor Redis keys to ensure locking behavior works as expected.  
    - Adjust TTL and retry delays as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow demonstrates Redis-based locking to prevent concurrent execution of workflows.                                                    | Sticky Note at top-left of workflow.                                                                |
| Attempts to acquire a lock using Redis by setting a key with expiration.                                                                        | Sticky Note near "Acquire Redis Lock" node.                                                        |
| Skips execution when duplicate request is received.                                                                                            | Sticky Note near "duplicateWebhook boolean" node.                                                  |
| Deletes the Redis lock key to release the lock.                                                                                                | Sticky Note near "Discard Redis Lock" node.                                                        |
| Recommended for use cases requiring idempotency or duplicate processing prevention.                                                             | Workflow description in initial comments.                                                          |
| Requires a Redis instance accessible from n8n; Upstash, Redis Cloud, or self-hosted Redis can be used.                                          | Setup guide section in workflow description.                                                       |
| Ensure consistent Redis key naming (`xyz-lock` used mostly; `n8n-rca-lock` in discard node is inconsistent and should be corrected).            | Observation from node analysis; key names must be consistent to avoid deadlocks.                    |
| Customize switch node conditions to route to actual business logic workflows or sub-workflows relevant to your use case.                        | "Workflow Switch" node placeholders.                                                                |
| For larger workflows, consider externalizing business logic into separate sub-workflows and invoke them here for modularity and readability.    | General best practice for maintainability.                                                         |

---

This comprehensive analysis enables full understanding, reproduction, and modification of the Redis locking workflow to prevent concurrent task execution in n8n.