Run Multiple Tasks in Parallel with Asynchronous Processing and Webhooks

https://n8nworkflows.xyz/workflows/run-multiple-tasks-in-parallel-with-asynchronous-processing-and-webhooks-8578


# Run Multiple Tasks in Parallel with Asynchronous Processing and Webhooks

### 1. Workflow Overview

This workflow demonstrates a proof-of-concept implementation for running multiple asynchronous tasks in parallel using n8n. Its primary use case is to orchestrate long-running or delayed processes without blocking the main workflow execution, ideal for batch processing or parallel task management.

**Logical Blocks:**

- **1.1 Entry & Trigger Block:** Handles manual and external trigger inputs to start the workflow asynchronously.
- **1.2 Asynchronous Task Execution Block:** Launches multiple parallel sub-workflows without waiting for their completion.
- **1.3 Wait & Webhook Reception Block:** Uses `Wait` nodes configured with webhooks to pause execution until asynchronous tasks report back.
- **1.4 Result Aggregation Block:** Merges and summarizes results received from asynchronous tasks.

---

### 2. Block-by-Block Analysis

#### 1.1 Entry & Trigger Block

**Overview:**  
This block initiates the workflow either manually or via an external trigger and passes input parameters to downstream nodes.

**Nodes Involved:**  
- Start  
- Call Entry Point

**Node Details:**

- **Start**  
  - *Type*: Manual Trigger  
  - *Role*: Entry point for manual execution.  
  - *Configuration*: No parameters; simply triggers workflow on user command.  
  - *Connections*: Output to "Call 1".  
  - *Edge Cases*: None typical; manual trigger requires human initiation.

- **Call Entry Point**  
  - *Type*: Execute Workflow Trigger  
  - *Role*: Receives input parameters for asynchronous calls, including `wait` time and a webhook URL.  
  - *Configuration*: Defines two expected inputs: a number `wait` and a string `webhook`.  
  - *Key Expressions*: Passes these inputs downstream using expressions, e.g., `{{ $('Call Entry Point').item.json.wait }}`.  
  - *Connections*: Outputs to "Wait Seconds".  
  - *Edge Cases*: Input validation failure if `wait` or `webhook` is missing or invalid.

---

#### 1.2 Asynchronous Task Execution Block

**Overview:**  
This block executes two parallel asynchronous workflows ("Call 1" and "Call 2") without waiting for their completion, enabling concurrent long-running task execution.

**Nodes Involved:**  
- Call 1  
- Call 2

**Node Details:**

- **Call 1**  
  - *Type*: Execute Workflow  
  - *Role*: Starts sub-workflow asynchronously with no waiting (`waitForSubWorkflow=false`).  
  - *Configuration*:  
    - Calls the same workflow by ID (self-reference) to simulate parallel tasks.  
    - Passes parameters: `wait=5` seconds and a webhook URL constructed using `{{$execution.resumeUrl}}/call1`.  
  - *Connections*: Output to "Call 2".  
  - *Edge Cases*:  
    - Must ensure sub-workflow ID is correct and that asynchronous execution is supported.  
    - Misconfigured webhook URLs could cause resume failures.

- **Call 2**  
  - *Type*: Execute Workflow  
  - *Role*: Similar to "Call 1" but with different parameters to simulate another parallel task.  
  - *Configuration*:  
    - `wait=3` seconds, webhook URL: `{{$execution.resumeUrl}}/call2`.  
  - *Connections*: Outputs to "Wait for Webhook 1" and "Wait for Webhook 2".  
  - *Edge Cases*: Same as "Call 1".

---

#### 1.3 Wait & Webhook Reception Block

**Overview:**  
This block uses wait nodes configured to resume on webhooks to asynchronously pause the workflow until each external sub-task signals completion.

**Nodes Involved:**  
- Wait Seconds  
- Wait for Webhook 1  
- Wait for Webhook 2  
- Request Webhook  
- Wait 1 Second

**Node Details:**

- **Wait Seconds**  
  - *Type*: Wait  
  - *Role*: Delays execution for a dynamically defined amount of time (`wait` parameter).  
  - *Configuration*: Wait duration set via expression referencing `Call Entry Point`'s input `wait`.  
  - *WebhookId*: Defined to allow resuming if needed.  
  - *Connections*: Output to "Request Webhook".  
  - *Edge Cases*: Invalid or zero `wait` values may cause immediate continuation.

- **Request Webhook**  
  - *Type*: HTTP Request  
  - *Role*: Sends a POST request to the webhook URL passed in inputs, reporting back results.  
  - *Configuration*:  
    - Sends a JSON body with a `result` field, calculated as `wait * 2`.  
    - Continues workflow on error (`onError: continueErrorOutput`).  
  - *Connections*:  
    - On success, no further action.  
    - On failure, triggers "Wait 1 Second" node to retry or delay.  
  - *Edge Cases*: Possible HTTP errors, webhook URL unreachable, timeouts.

- **Wait 1 Second**  
  - *Type*: Wait  
  - *Role*: Pauses for 1 second, likely used for retry delay after failed HTTP requests.  
  - *Connections*: Output loops back to "Request Webhook".  
  - *Edge Cases*: Infinite loop risk if "Request Webhook" keeps failing.

- **Wait for Webhook 1**  
  - *Type*: Wait (Webhook Resume)  
  - *Role*: Pauses workflow until webhook callback at suffix `/call1` is received.  
  - *Configuration*:  
    - Resume on webhook with suffix "call1".  
    - Limits wait time (minutes).  
  - *Connections*: Output to "Merge" node.  
  - *Edge Cases*: If webhook never received, workflow may timeout or stall.

- **Wait for Webhook 2**  
  - *Type*: Wait (Webhook Resume)  
  - *Role*: Same as above but listens on webhook suffix "call2".  
  - *Connections*: Output to "Merge".  
  - *Edge Cases*: Same as "Wait for Webhook 1".

---

#### 1.4 Result Aggregation Block

**Overview:**  
This block merges the results returned from the asynchronous tasks and performs a summary aggregation.

**Nodes Involved:**  
- Merge  
- Sum

**Node Details:**

- **Merge**  
  - *Type*: Merge  
  - *Role*: Combines incoming data streams from "Wait for Webhook 1" and "Wait for Webhook 2".  
  - *Configuration*: Default merge mode (likely "Wait for All").  
  - *Connections*: Output to "Sum".  
  - *Edge Cases*: Missing inputs from either wait node may cause indefinite waiting.

- **Sum**  
  - *Type*: Summarize  
  - *Role*: Aggregates numeric results by summing the `body.result` fields from merged inputs.  
  - *Configuration*: Sum aggregation on field `body.result`.  
  - *Connections*: Terminal node with no outputs.  
  - *Edge Cases*: Non-numeric or missing `result` fields may cause errors or incorrect sums.

---

### 3. Summary Table

| Node Name          | Node Type                 | Functional Role                           | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                   |
|--------------------|---------------------------|-----------------------------------------|------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Start              | Manual Trigger            | Manual workflow trigger                  | —                      | Call 1                        |                                                                                                              |
| Call Entry Point    | Execute Workflow Trigger  | Receives external inputs for async tasks| —                      | Wait Seconds                  |                                                                                                              |
| Wait Seconds       | Wait                      | Waits dynamically based on input         | Call Entry Point        | Request Webhook               |                                                                                                              |
| Request Webhook    | HTTP Request              | Sends callback to provided webhook URL   | Wait Seconds            | — (success), Wait 1 Second (error) |                                                                                                              |
| Wait 1 Second      | Wait                      | Delay before retrying webhook request     | Request Webhook (error) | Request Webhook               |                                                                                                              |
| Call 1             | Execute Workflow          | Starts async sub-workflow #1              | Start                   | Call 2                       |                                                                                                              |
| Call 2             | Execute Workflow          | Starts async sub-workflow #2              | Call 1                  | Wait for Webhook 1, Wait for Webhook 2 |                                                                                                              |
| Wait for Webhook 1 | Wait (Webhook Resume)     | Waits for async task #1 webhook callback | Call 2                  | Merge                        |                                                                                                              |
| Wait for Webhook 2 | Wait (Webhook Resume)     | Waits for async task #2 webhook callback | Call 2                  | Merge                        |                                                                                                              |
| Merge              | Merge                     | Combines webhook results                   | Wait for Webhook 1, Wait for Webhook 2 | Sum                         |                                                                                                              |
| Sum                | Summarize                 | Aggregates results by summing field        | Merge                   | —                            |                                                                                                              |
| Sticky Note        | Sticky Note               | Documentation and instructions             | —                       | —                            | ## n8n Asynchronous Workflow with Wait Node POC This template contains a two-part workflow designed to demonstrate a proof-of-concept for asynchronous and parallel execution of tasks in n8n. The blog post link: https://n8nplaybook.com/post/2025/09/asynchronous-n8n-workflows-parallel-processing-poc/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**
   - Name: `Start`
   - Type: `Manual Trigger`
   - No parameters.
   - Position it at the left/start of your canvas.

2. **Add Execute Workflow Trigger Node:**
   - Name: `Call Entry Point`
   - Type: `Execute Workflow Trigger`
   - Configure inputs schema with two parameters:  
     - `wait` (Number)  
     - `webhook` (String)  
   - Position below `Start`.
   - Connect output of `Start` to input of this node.

3. **Add Wait Node (Dynamic):**
   - Name: `Wait Seconds`
   - Type: `Wait`
   - Set the amount to an expression: `{{$node["Call Entry Point"].json["wait"]}}`
   - Assign a unique webhook ID (auto-generated by n8n).
   - Connect from `Call Entry Point`.

4. **Add HTTP Request Node:**
   - Name: `Request Webhook`
   - Type: `HTTP Request`
   - Method: POST
   - URL: Expression: `{{$node["Call Entry Point"].json["webhook"]}}`
   - Body parameters JSON:  
     - Parameter name: `result`  
     - Value expression: `{{$node["Call Entry Point"].json["wait"] * 2}}`
   - Set `Continue on Fail` to true.
   - Connect from `Wait Seconds`.

5. **Add Wait Node (1 second delay):**
   - Name: `Wait 1 Second`
   - Type: `Wait`
   - Amount: 1 (second)
   - Connect error output of `Request Webhook` to this node.
   - Connect output of `Wait 1 Second` back to `Request Webhook` for retries.

6. **Add Execute Workflow Node (Call 1):**
   - Name: `Call 1`
   - Type: `Execute Workflow`
   - Select the same workflow ID to call itself asynchronously.
   - Set `Wait for Sub-Workflow` to false.
   - Inputs:  
     - `wait`: 5  
     - `webhook`: Expression: `{{$execution.resumeUrl}}/call1`
   - Connect from `Start`.

7. **Add Execute Workflow Node (Call 2):**
   - Name: `Call 2`
   - Type: `Execute Workflow`
   - Same configuration as `Call 1` but:  
     - `wait`: 3  
     - `webhook`: Expression: `{{$execution.resumeUrl}}/call2`
   - Connect output of `Call 1` to input.

8. **Add Wait Nodes for Webhook Resume:**
   - Name: `Wait for Webhook 1`
     - Type: `Wait`
     - Configure to resume on webhook with suffix `call1`.
     - Limit wait time.
     - Connect output of `Call 2`.
   - Name: `Wait for Webhook 2`
     - Same as above but suffix `call2`.
     - Connect output of `Call 2`.

9. **Add Merge Node:**
   - Name: `Merge`
   - Type: `Merge`
   - No special parameters.
   - Connect outputs of `Wait for Webhook 1` and `Wait for Webhook 2`.

10. **Add Summarize Node:**
    - Name: `Sum`
    - Type: `Summarize`
    - Configure to sum the field `body.result`.
    - Connect from `Merge`.

11. **Add Sticky Note:**
    - Content with workflow purpose, usage, and link to blog post (optional).

12. **Ensure all nodes are positioned logically and connected as per above.**

13. **Credentials:**
    - No special credentials required for this workflow.
    - If HTTP requests go to external endpoints requiring auth, configure accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow is a demonstration of asynchronous and parallel task execution using n8n’s Wait node and webhook resume pattern, suitable for batch processing scenarios.                                                                                 | Workflow purpose                                                                                                    |
| For detailed explanation and concepts behind this proof-of-concept, see the blog post: https://n8nplaybook.com/post/2025/09/asynchronous-n8n-workflows-parallel-processing-poc/                                                                         | https://n8nplaybook.com/post/2025/09/asynchronous-n8n-workflows-parallel-processing-poc/                             |
| The workflow calls itself asynchronously to simulate parallel tasks; make sure to import and link workflows correctly when deploying in your environment.                                                                                              | Setup instructions in sticky note                                                                                   |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It complies strictly with current content policies and includes only legal, public data.