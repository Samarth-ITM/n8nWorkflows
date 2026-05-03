Pattern for Parallel Sub-Workflow Execution Followed by Wait-For-All Loop

https://n8nworkflows.xyz/workflows/pattern-for-parallel-sub-workflow-execution-followed-by-wait-for-all-loop-2536


# Pattern for Parallel Sub-Workflow Execution Followed by Wait-For-All Loop

### 1. Workflow Overview

This workflow demonstrates a design pattern for executing multiple sub-workflows asynchronously in parallel within n8n, while ensuring the parent workflow waits until all sub-workflows have completed before proceeding. It addresses a common challenge in n8n: running sub-workflows concurrently but synchronizing the continuation of the main workflow until all parallel tasks finish.

The workflow is logically divided into two main blocks:

- **1.1 Parallel Sub-Workflow Execution Block:**  
  Generates multiple input items, splits them for parallel processing, and triggers corresponding sub-workflows asynchronously via HTTP requests with callback URLs.

- **1.2 Wait-for-All Completion Block:**  
  Implements a webhook-based callback mechanism that waits for asynchronous responses from all sub-workflows, accumulates their completion status, and loops until all sub-workflows have reported back, then continues the main workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Parallel Sub-Workflow Execution Block

**Overview:**  
This block prepares multiple input items, splits them into batches (single items), and triggers parallel executions of a sub-workflow by sending HTTP POST requests containing unique identifiers and a callback URL for asynchronous completion notification.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Simulate Multi-Item for Parallel Processing (Code)  
- Loop Over Items (SplitInBatches)  
- Initialize finishedSet (Set)  
- Start Sub-Workflow via Webhook (HTTP Request)  
- Sticky Note (named "Start Multiple Sub-Workflows Asynchronously")

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual workflow execution during testing.  
  - *Configuration:* No parameters; triggers workflow start manually.  
  - *Inputs:* None  
  - *Outputs:* Passes trigger event to next node.  
  - *Failure Modes:* None typical, manual trigger.  

- **Simulate Multi-Item for Parallel Processing**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates an array of objects representing items to process in parallel (simulated requests).  
  - *Configuration:* Returns three objects, each with unique `requestId` values (`req4567`, `req8765`, `req1234`).  
  - *Key Expressions:* Static JavaScript array return.  
  - *Inputs:* From manual trigger node.  
  - *Outputs:* Emits array of items for splitting.  
  - *Failure Modes:* Syntax errors in code.  

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Splits input array into individual items to process in parallel, enabling asynchronous handling.  
  - *Configuration:* Default batch size (1) to split items one by one.  
  - *Inputs:* Array from the simulation code node.  
  - *Outputs:* Each item triggers subsequent node independently.  
  - *Failure Modes:* Misconfiguration of batch size could affect concurrency; ensure batch size = 1 for parallelism.  

- **Initialize finishedSet**  
  - *Type:* Set  
  - *Role:* Initializes an empty array variable `finishedSet` to keep track of completed sub-workflows (executed once).  
  - *Configuration:* Sets `finishedSet` to an empty array `[]`.  
  - *Inputs:* From the Loop Over Items (only once due to `executeOnce: true`).  
  - *Outputs:* Passes initialized context to next nodes.  
  - *Failure Modes:* If not executed once, could reset tracking prematurely.  

- **Start Sub-Workflow via Webhook**  
  - *Type:* HTTP Request  
  - *Role:* Sends an HTTP POST request to asynchronously trigger the sub-workflow, passing each item's `requestId` and the parent workflow’s resume URL as a callback.  
  - *Configuration:*  
    - URL: Uses environment variable `WEBHOOK_URL` appended with `/webhook/parallel-subworkflow-target`.  
    - Method: POST  
    - Body: JSON with parameter `requestItemId` set from current item `requestId`.  
    - Headers: Includes `callbackurl` header set to `$execution.resumeUrl` (the parent workflow's webhook URL to resume execution upon callback).  
  - *Inputs:* Single item per batch.  
  - *Outputs:* Response from sub-workflow trigger.  
  - *Failure Modes:* Network errors, invalid webhook URL or credentials, missing environment variable, timeout.  

- **Sticky Note ("Start Multiple Sub-Workflows Asynchronously")**  
  - *Type:* Sticky Note  
  - *Role:* Documentation reminder about internal callback URL setup in Kubernetes environment for sub-workflow callbacks.  
  - *Content:* Emphasizes configuring the n8n instance’s internal base URL correctly for callbacks.

---

#### 2.2 Wait-for-All Completion Block

**Overview:**  
This block waits for asynchronous callbacks from each sub-workflow via a webhook, updates a tracking list of completed sub-workflows, checks if all are done, and only then allows the main workflow to continue further.

**Nodes Involved:**  
- Webhook Callback Wait (Wait)  
- Update finishedSet (Code)  
- If All Finished (If)  
- Initialize finishedSet (Set) [shared node with previous block]  
- Acknowledge Finished (RespondToWebhook)  
- Continue Workflow (noop)  
- Sticky Note1 ("Pseudo-Synchronously Wait for All Sub-Workflows to finish")

**Node Details:**

- **Webhook Callback Wait**  
  - *Type:* Wait (Webhook)  
  - *Role:* Suspends workflow execution until a POST request is received on a specific webhook ID, resuming workflow upon callback from sub-workflows.  
  - *Configuration:*  
    - Webhook ID: Static UUID identifying this wait webhook.  
    - Resume: Waits for webhook POST requests.  
    - Response Mode: Responds with a node response.  
  - *Inputs:* From If node (when sub-workflows are not all finished).  
  - *Outputs:* Provides webhook payload to update finished set.  
  - *Failure Modes:* Missed webhook call, webhook URL misconfiguration, delayed callbacks.  

- **Update finishedSet**  
  - *Type:* Code (JavaScript)  
  - *Role:* Adds the `finishedItemId` from the webhook callback to the `finishedSet` array if not already present.  
  - *Configuration:*  
    - Reads current `finishedSet` array from `If All Finished` node JSON context.  
    - Reads finished item ID from webhook body.  
    - Pushes new ID to array if unique.  
  - *Key Expressions:* Uses `$('If All Finished').first().json` to access context; updates array.  
  - *Inputs:* From Webhook Callback Wait node.  
  - *Outputs:* Updated JSON with appended `finishedSet`.  
  - *Failure Modes:* Expression failure if `finishedSet` is undefined; potential race conditions (though mitigated by design).  

- **If All Finished**  
  - *Type:* If  
  - *Role:* Checks if the number of unique finished sub-workflow IDs equals or exceeds the total number of items originally started.  
  - *Configuration:*  
    - Condition: `finishedSet.length >= number_of_items_in_Simulate Multi-Item for Parallel Processing`  
  - *Inputs:* From Initialize finishedSet and Acknowledge Finished nodes.  
  - *Outputs:*  
    - True branch: Continue Workflow (noop) node (workflow continuation).  
    - False branch: Webhook Callback Wait node (continue waiting).  
  - *Failure Modes:* Condition misconfiguration; inaccurate counts due to missing callbacks.  

- **Acknowledge Finished**  
  - *Type:* RespondToWebhook  
  - *Role:* Sends HTTP response acknowledging receipt of callback and triggers the If node to check completion.  
  - *Inputs:* From Update finishedSet node.  
  - *Outputs:* To If All Finished node.  
  - *Failure Modes:* Response failures could cause sub-workflow retries.  

- **Continue Workflow (noop)**  
  - *Type:* NoOp (No operation)  
  - *Role:* Placeholder node signaling all sub-workflows finished; workflow can proceed with further steps if necessary.  
  - *Inputs:* True branch from If All Finished.  
  - *Outputs:* None configured in this template.  

- **Sticky Note1 ("Pseudo-Synchronously Wait for All Sub-Workflows to finish")**  
  - *Type:* Sticky Note  
  - *Role:* Documentation explaining this block handles waiting for all sub-workflows to complete before continuing the main workflow.

---

#### 2.3 Sub-Workflow (Referenced)

**Overview:**  
A separate sub-workflow (not fully included here but referenced in the main workflow) is triggered asynchronously via HTTP POST. It responds immediately and later sends a callback POST request to the main workflow’s webhook with its unique identifier to signal completion.

**Nodes Involved (from main workflow references):**  
- Webhook (to receive start request)  
- Respond to Webhook (acknowledge start)  
- Wait node (simulate processing delay)  
- Call Resume on Parent Workflow (HTTP Request to callback URL)

**Node Details:**

- **Webhook**  
  - *Type:* Webhook  
  - *Role:* Receives POST requests from the main workflow to start processing an individual item.  
  - *Configuration:* Path `/parallel-subworkflow-target`, method POST, responds with Respond to Webhook node.  
  - *Inputs:* External HTTP requests from main workflow.  
  - *Outputs:* Data to Respond to Webhook node and further processing.  

- **Respond to Webhook**  
  - *Type:* RespondToWebhook  
  - *Role:* Sends immediate HTTP response to acknowledge receipt of processing request.  
  - *Outputs:* To Wait node for simulated processing delay.  

- **Wait**  
  - *Type:* Wait  
  - *Role:* Simulates asynchronous processing time.  
  - *Outputs:* To Call Resume on Parent Workflow node.  

- **Call Resume on Parent Workflow**  
  - *Type:* HTTP Request  
  - *Role:* Sends POST request to the parent workflow’s callback URL, signaling that this sub-workflow instance has finished processing.  
  - *Parameters:*  
    - URL from header `callbackurl`.  
    - Body includes `finishedItemId` set to the original `requestItemId` from the webhook payload.  
    - Retries on failure with delay to handle possible race conditions.  
  - *Failure Modes:* Network issues, race conditions mitigated by retry and delay.

- **Sticky Note2 ("Sub-Workflow")**  
  - *Role:* Instruction to cut and paste this sub-workflow into a separate n8n workflow and activate it.

---

### 3. Summary Table

| Node Name                          | Node Type            | Functional Role                                        | Input Node(s)                               | Output Node(s)                              | Sticky Note                                                                                              |
|-----------------------------------|----------------------|-------------------------------------------------------|---------------------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger       | Entry point to start workflow manually                 | None                                        | Simulate Multi-Item for Parallel Processing |                                                                                                        |
| Simulate Multi-Item for Parallel Processing | Code (JavaScript)   | Generates array of items to process in parallel        | When clicking ‘Test workflow’                | Loop Over Items                              |                                                                                                        |
| Loop Over Items                   | SplitInBatches        | Splits items for parallel asynchronous processing      | Simulate Multi-Item for Parallel Processing | Initialize finishedSet, Start Sub-Workflow via Webhook |                                                                                                        |
| Initialize finishedSet            | Set                   | Initializes empty array to track finished sub-workflows | Loop Over Items                              | If All Finished                              |                                                                                                        |
| Start Sub-Workflow via Webhook    | HTTP Request          | Triggers sub-workflow asynchronously with callback URL | Loop Over Items                              | Loop Over Items (loop back; may be unused) | Sticky Note: "Start Multiple Sub-Workflows Asynchronously" - Notes on internal callback URL configuration |
| Webhook Callback Wait             | Wait (Webhook)        | Waits for asynchronous callbacks from sub-workflows   | If All Finished (false branch)               | Update finishedSet                           | Sticky Note1: "Pseudo-Synchronously Wait for All Sub-Workflows to finish"                               |
| Update finishedSet                | Code (JavaScript)     | Adds finished sub-workflow ID to tracking array        | Webhook Callback Wait                        | Acknowledge Finished                         |                                                                                                        |
| Acknowledge Finished              | RespondToWebhook      | Sends HTTP response acknowledging callback receipt     | Update finishedSet                           | If All Finished                              |                                                                                                        |
| If All Finished                  | If                    | Checks if all sub-workflows finished                    | Initialize finishedSet, Acknowledge Finished | Continue Workflow (noop) [true], Webhook Callback Wait [false] |                                                                                                        |
| Continue Workflow (noop)          | NoOp                  | Placeholder node after all sub-workflows complete      | If All Finished (true branch)                | None                                        |                                                                                                        |
| Webhook                         | Webhook                | Sub-workflow entry point receiving start requests      | None (external HTTP)                         | Respond to Webhook                           | Sticky Note2: "Sub-Workflow - Cut/Paste this into a separate workflow and activate it"                  |
| Respond to Webhook               | RespondToWebhook       | Sub-workflow immediate acknowledgment                   | Webhook                                      | Wait                                         |                                                                                                        |
| Wait                            | Wait                   | Sub-workflow processing delay simulation                | Respond to Webhook                           | Call Resume on Parent Workflow               |                                                                                                        |
| Call Resume on Parent Workflow   | HTTP Request          | Sub-workflow calls back to main workflow to signal done | Wait                                         | None                                        | Notes on retry and delay to mitigate race conditions                                                  |
| Sticky Note3                    | Sticky Note            | General note describing the main/parent workflow        | None                                         | None                                        | "Main/Parent Workflow - starts multiple sub-workflows in parallel and loops until all report back"     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters needed.  

2. **Create Code node:**  
   - Name: `Simulate Multi-Item for Parallel Processing`  
   - Set JavaScript code to return array of objects with unique IDs:  
     ```js
     return [
       {requestId: 'req4567'},
       {requestId: 'req8765'},
       {requestId: 'req1234'}
     ];
     ```  

3. **Connect manual trigger to code node.**

4. **Create SplitInBatches node:**  
   - Name: `Loop Over Items`  
   - Parameters: Default batch size = 1 (ensures single-item batches for parallel processing).  

5. **Connect code node to SplitInBatches node.**

6. **Create Set node:**  
   - Name: `Initialize finishedSet`  
   - Parameters: Assign variable `finishedSet` as an empty array `[]`.  
   - Set execution mode to `Execute Once` (only first item).  

7. **Connect SplitInBatches node to Set node.**

8. **Create HTTP Request node:**  
   - Name: `Start Sub-Workflow via Webhook`  
   - Parameters:  
     - Method: POST  
     - URL: `{{$env.WEBHOOK_URL}}/webhook/parallel-subworkflow-target` (requires environment variable `WEBHOOK_URL` to be set)  
     - Body Parameters: JSON with parameter `requestItemId` = `{{$json.requestId}}`  
     - Header Parameters: Include `callbackurl` header set to `{{$execution.resumeUrl}}`  
   - Credentials: None unless webhook requires.  

9. **Connect SplitInBatches node to HTTP Request node.**

10. **Create Wait node (Webhook):**  
    - Name: `Webhook Callback Wait`  
    - Configure to wait for POST webhook requests, with response mode `responseNode`.  
    - Webhook ID: auto-generated or set static (to match sub-workflow callback URL).  

11. **Create Code node:**  
    - Name: `Update finishedSet`  
    - JavaScript code:  
      ```js
      let json = $('If All Finished').first().json;
      if (!json.finishedSet) json.finishedSet = [];
      let finishedItemId = $('Webhook Callback Wait').item.json.body.finishedItemId;
      if (!json.finishedSet.includes(finishedItemId)) json.finishedSet.push(finishedItemId);
      return [json];
      ```  
    - This updates the finishedSet with new callback ID if not already included.  

12. **Connect Wait node to Update finishedSet node.**

13. **Create RespondToWebhook node:**  
    - Name: `Acknowledge Finished`  
    - Send HTTP 200 response to acknowledge the callback.  

14. **Connect Update finishedSet node to RespondToWebhook node.**

15. **Create If node:**  
    - Name: `If All Finished`  
    - Condition: Check if length of `finishedSet` array >= total number of initial items (from simulation code node).  
    - Expression example:  
      ``` 
      {{$json.finishedSet.length}} >= {{$node["Simulate Multi-Item for Parallel Processing"].all().length}} 
      ```  
    - True branch: Connect to next processing node (NoOp).  
    - False branch: Loop back to `Webhook Callback Wait` to keep waiting.  

16. **Connect `Initialize finishedSet` node output to If node input.**  
    Connect `Acknowledge Finished` node output to If node input (for repeated checks).

17. **Create NoOp node:**  
    - Name: `Continue Workflow (noop)`  
    - This node serves as placeholder indicating all sub-workflows finished.  

18. **Connect If node’s true output to NoOp node.**

19. **(Optional) Add Sticky Notes for clarity:**  
    - Add notes near the parallel execution block explaining setup and internal callback URL.  
    - Add notes near wait-for-all block explaining the pseudo-synchronous waiting pattern.  
    - Add notes for sub-workflow describing that it must be cut/pasted as a separate activated workflow.  

20. **Create the Sub-Workflow separately:**  

    - **Webhook node:**  
      - Path: `/parallel-subworkflow-target`  
      - Method: POST  

    - **RespondToWebhook node:**  
      - Immediately acknowledge the start request.  

    - **Wait node:**  
      - Simulate processing delay.  

    - **HTTP Request node:**  
      - Call back to main workflow’s resume URL (passed in header `callbackurl`) with POST body including `finishedItemId` (same as `requestItemId` received).  
      - Configure retry on fail with delay (e.g., 3 seconds) to mitigate race conditions.  

    - Connect nodes in order: Webhook → RespondToWebhook → Wait → HTTP Request.  

21. **Deploy and activate both workflows.**  
    - Set environment variable `WEBHOOK_URL` in main workflow environment to point to your n8n instance’s accessible base URL.  
    - Ensure the sub-workflow path `/parallel-subworkflow-target` matches the HTTP request URL.  
    - Verify that callback URLs are accessible internally from sub-workflows to parent workflow.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                   | Context or Link                                                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow pattern addresses a frequent n8n community question on running sub-workflows in parallel and waiting for all to complete before continuing.                                     | [n8n Community Discussion on Parallel Execution](https://community.n8n.io/t/is-it-possible-to-run-a-part-of-the-workflow-in-parallel/60221)                                                        |
| The approach uses in-workflow webhook waiting and resume URLs, avoiding external message queues or databases, achieving concurrency and synchronization fully within n8n.                      | See related threads about waiting and parallel execution: [Waiting for things to finish](https://community.n8n.io/t/node-does-not-wait-for-predecessor-node-always-need-a-merge-node/60773)          |
| When running in Kubernetes (k8s), internal callback URLs must be configured to use service names and internal ports for intra-cluster communication.                                            | Sticky Note in workflow mentions this to ensure callbacks reach the parent workflow correctly.                                                                                                       |
| The sub-workflow must be created and activated separately. It receives initial requests, responds immediately, then performs processing and finally calls back to the parent workflow.          | Sticky Note in workflow states: "Cut/Paste this into a separate workflow, and activate it!!!"                                                                                                         |
| Retry and delay on callback HTTP request mitigate race conditions where multiple sub-workflows complete simultaneously and attempt to resume the parent workflow concurrently.                   | Implemented in sub-workflow’s HTTP Request node: `retryOnFail` with `waitBetweenTries` set to 3000 ms.                                                                                                |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and modifying the parallel asynchronous sub-workflow execution pattern with wait-for-all synchronization in n8n.