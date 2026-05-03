Pattern for Multiple Triggers Combined to Continue Workflow

https://n8nworkflows.xyz/workflows/pattern-for-multiple-triggers-combined-to-continue-workflow-2857


# Pattern for Multiple Triggers Combined to Continue Workflow

### 1. Workflow Overview

This workflow demonstrates a pattern to coordinate two separate workflow executions—referred to as the **Primary** and **Secondary** executions—allowing an external asynchronous process to resume and continue the primary workflow with additional input. It is designed for scenarios where a workflow initiates an external process (e.g., a Telegram conversation or external webhook) and then waits for a callback containing context to resume execution.

The workflow is logically divided into these blocks:

- **1.1 Primary Workflow Trigger and Execution:** Starts the main workflow, sets context, initiates an external process, and waits for a callback.
- **1.2 Wait Node for Resuming Execution:** Pauses the primary workflow until resumed by an external trigger via a webhook call.
- **1.3 Secondary Workflow Trigger and Execution:** Represents the external independent process that triggers a separate workflow execution, passing back context to resume the primary workflow.
- **1.4 Combining Inputs and Continuing Execution:** After resuming, combines data from both primary and secondary executions for further processing.
- **1.5 Simulated External Independent Process:** A demonstration sub-workflow that simulates the external process triggering the secondary workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Primary Workflow Trigger and Execution

- **Overview:**  
  This block initiates the primary workflow execution manually, sets initial context variables, and triggers an external independent process by sending a POST request with the resume URL.

- **Nodes Involved:**  
  - `When clicking ‘Test workflow’` (Manual Trigger)  
  - `Set Primary Execution Context` (Set)  
  - `HTTP Request - Initiate Independent Process` (HTTP Request)  

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the primary workflow manually for testing.  
    - Configuration: No parameters; triggers workflow on manual action.  
    - Input: None  
    - Output: Triggers `Set Primary Execution Context`  
    - Edge Cases: None  

  - **Set Primary Execution Context**  
    - Type: Set  
    - Role: Defines context variables for the primary execution, including a simulated external workflow ID and a context string.  
    - Configuration:  
      - `someContextItem`: A string value `"this value is only available / in-scope from the primary execution's previous-nodes"`  
      - `simulatedExternalProcessWorkflowId`: Workflow ID string `"21cea9f6-d55f-4c47-b6a2-158cce1811cd"` (must be updated for actual use)  
    - Input: From manual trigger  
    - Output: To `HTTP Request - Initiate Independent Process`  
    - Edge Cases: Workflow ID must be updated to match target environment.  

  - **HTTP Request - Initiate Independent Process**  
    - Type: HTTP Request  
    - Role: Initiates the external independent process by sending a POST request to the external workflow’s webhook URL, passing the current execution’s resume URL as context.  
    - Configuration:  
      - URL: Constructed dynamically using the `simulatedExternalProcessWorkflowId` from previous node.  
      - Method: POST  
      - Body: JSON with parameter `resumeUrlForWaitingExecution` set to `$execution.resumeUrl` (the webhook URL to resume this workflow).  
    - Input: From `Set Primary Execution Context`  
    - Output: To `Wait` node  
    - Edge Cases:  
      - Requires the external workflow to be active and accessible at the specified URL.  
      - Network errors or incorrect URL will cause failure.  
      - The embedded workflow ID must be updated to actual external workflow ID.  

---

#### 2.2 Wait Node for Resuming Execution

- **Overview:**  
  This node pauses the primary workflow execution and waits for a webhook POST call to resume. The webhook URL is exposed as `$execution.resumeUrl` and passed to the external process.

- **Nodes Involved:**  
  - `Wait` (Wait Node)  

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses execution until resumed by a webhook call.  
    - Configuration:  
      - Resume mode: `webhook`  
      - HTTP Method: POST  
      - Webhook ID: Fixed UUID for this node’s webhook endpoint.  
    - Input: From `HTTP Request - Initiate Independent Process`  
    - Output: To `This Node Can Access Primary and Secondary`  
    - Edge Cases:  
      - Only one resume call is accepted; multiple calls will be rejected.  
      - If no resume call is received, workflow remains paused indefinitely.  
      - Webhook URL must be correctly passed and accessible externally.  

---

#### 2.3 Secondary Workflow Trigger and Execution

- **Overview:**  
  This block represents the secondary workflow triggered independently by an external process. It receives the resume URL and other data, then calls back to the primary workflow’s wait node to resume execution.

- **Nodes Involved:**  
  - `Receive Input from External, Independent Process` (Webhook)  
  - `HTTP Request - Resume Other Workflow Execution` (HTTP Request)  

- **Node Details:**

  - **Receive Input from External, Independent Process**  
    - Type: Webhook  
    - Role: Entry point for the secondary workflow, triggered externally with data including the resume URL.  
    - Configuration:  
      - HTTP Method: POST  
      - Webhook Path: Unique UUID path  
    - Input: External POST request containing JSON with `resumeUrlForWaitingExecution` and other data (e.g., joke data).  
    - Output: To `HTTP Request - Resume Other Workflow Execution`  
    - Edge Cases:  
      - Must receive valid JSON with required parameters.  
      - If missing or malformed data, subsequent nodes may fail.  

  - **HTTP Request - Resume Other Workflow Execution**  
    - Type: HTTP Request  
    - Role: Calls the resume webhook URL of the primary workflow’s wait node, passing along data received from the secondary trigger.  
    - Configuration:  
      - URL: Derived from `resumeUrlForWaitingExecution` in the webhook body, replacing environment variable `$env.WEBHOOK_URL` with localhost URL for Docker compatibility.  
      - Method: POST  
      - Body: Passes parameters `jokeFromIndependentProcess`, `setupFromIndependentProcess`, and `deliveryFromIndependentProcess` extracted from the webhook input JSON.  
    - Input: From `Receive Input from External, Independent Process`  
    - Output: None (end of this branch)  
    - Edge Cases:  
      - URL replacement assumes Docker environment; may need adjustment for other setups.  
      - If resume URL is invalid or inaccessible, resume will fail.  
      - Data extraction expressions must match input JSON structure.  

---

#### 2.4 Combining Inputs and Continuing Execution

- **Overview:**  
  After the primary workflow is resumed, this block combines data from the original primary execution context and the data received from the secondary execution to continue processing.

- **Nodes Involved:**  
  - `This Node Can Access Primary and Secondary` (Set)  

- **Node Details:**

  - **This Node Can Access Primary and Secondary**  
    - Type: Set  
    - Role: Merges data from the primary execution’s context and the data received via the resume webhook call to create a combined data set.  
    - Configuration:  
      - Sets `somethingFromPrimaryExecution` from `Set Primary Execution Context` node’s `someContextItem`.  
      - Sets `somethingFromSecondaryExecution` from the resumed webhook data field `jokeFromIndependentProcess`.  
    - Input: From `Wait` node (resumed execution)  
    - Output: None (end of main workflow)  
    - Edge Cases:  
      - Assumes the resume webhook call includes the expected fields; missing fields may cause undefined values.  

---

#### 2.5 Simulated External Independent Process (Separate Workflow)

- **Overview:**  
  This block simulates an external independent process that triggers the secondary workflow. It demonstrates how an external system might send data including the resume URL to the secondary workflow.

- **Nodes Involved:**  
  - `Demo "Trigger" Callback Setup` (Set)  
  - `HTTP Request - Get A Random Joke` (HTTP Request)  
  - `Simulate some Consumed Service Time` (Wait)  
  - `Simulate Event that Hits the 2nd Trigger/Flow` (HTTP Request)  
  - `Respond to Webhook` (Respond to Webhook)  
  - `Webhook` (Webhook, disabled)  

- **Node Details:**

  - **Demo "Trigger" Callback Setup**  
    - Type: Set  
    - Role: Sets the target workflow ID for the secondary trigger webhook.  
    - Configuration:  
      - `triggerTargetWorkflowId`: UUID of the secondary workflow webhook (`3064395b-378c-4755-9634-ce40cc4733a6`)  
    - Input: From `Respond to Webhook`  
    - Output: To `HTTP Request - Get A Random Joke`  
    - Edge Cases: Workflow ID must be updated for actual environment.  

  - **HTTP Request - Get A Random Joke**  
    - Type: HTTP Request  
    - Role: Fetches a random programming joke from an external API to simulate data for the callback.  
    - Configuration:  
      - URL: `https://v2.jokeapi.dev/joke/Programming`  
      - Query Parameters: blacklist flags to exclude inappropriate jokes, type set to `single`  
    - Input: From `Demo "Trigger" Callback Setup`  
    - Output: To `Simulate some Consumed Service Time`  
    - Edge Cases: API downtime or network issues may cause failure.  

  - **Simulate some Consumed Service Time**  
    - Type: Wait  
    - Role: Simulates delay to mimic processing time in the external system.  
    - Configuration: Wait for 2 seconds.  
    - Input: From `HTTP Request - Get A Random Joke`  
    - Output: To `Simulate Event that Hits the 2nd Trigger/Flow`  
    - Edge Cases: None  

  - **Simulate Event that Hits the 2nd Trigger/Flow**  
    - Type: HTTP Request  
    - Role: Sends a POST request to the secondary workflow webhook, passing the resume URL and joke data to trigger the secondary workflow.  
    - Configuration:  
      - URL: Constructed dynamically using `triggerTargetWorkflowId` from `Demo "Trigger" Callback Setup`  
      - Body: Includes `resumeUrlForWaitingExecution` and `joke` fields from previous nodes  
    - Input: From `Simulate some Consumed Service Time`  
    - Output: None  
    - Edge Cases: Requires the secondary workflow webhook to be active and accessible.  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends a response to the initial webhook call to acknowledge receipt.  
    - Input: From `Webhook` (disabled)  
    - Output: To `Demo "Trigger" Callback Setup`  
    - Edge Cases: The `Webhook` node is disabled; this is for demonstration only.  

  - **Webhook** (Disabled)  
    - Type: Webhook  
    - Role: Intended as an entry point for the simulated external process; disabled in this workflow.  
    - Edge Cases: Disabled; no active use.  

---

### 3. Summary Table

| Node Name                                   | Node Type            | Functional Role                                      | Input Node(s)                          | Output Node(s)                                | Sticky Note                                                                                                               |
|---------------------------------------------|----------------------|-----------------------------------------------------|--------------------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                | Manual Trigger       | Primary workflow start                              | None                                 | Set Primary Execution Context                  |                                                                                                                           |
| Set Primary Execution Context                | Set                  | Define primary execution context variables          | When clicking ‘Test workflow’         | HTTP Request - Initiate Independent Process    | ## Update Me                                                                                                              |
| HTTP Request - Initiate Independent Process | HTTP Request         | Initiate external independent process with resumeUrl| Set Primary Execution Context         | Wait                                          |                                                                                                                           |
| Wait                                         | Wait                 | Pause primary workflow until resumed via webhook    | HTTP Request - Initiate Independent Process | This Node Can Access Primary and Secondary    | These are the nodes that combine the `Secondary` execution back to the `Primary` execution via the `resumeUrl`.           |
| This Node Can Access Primary and Secondary  | Set                  | Combine data from primary and secondary executions  | Wait                                 | None                                          |                                                                                                                           |
| Receive Input from External, Independent Process | Webhook           | Secondary workflow trigger receiving resumeUrl      | None                                 | HTTP Request - Resume Other Workflow Execution | ## Secondary Trigger From Independent Process<br>When something runs the workflow through this trigger, it is a completely separate execution. |
| HTTP Request - Resume Other Workflow Execution | HTTP Request       | Resume primary workflow by calling wait webhook     | Receive Input from External, Independent Process | None                                          |                                                                                                                           |
| Demo "Trigger" Callback Setup                 | Set                  | Setup target workflow ID for secondary trigger      | Respond to Webhook                   | HTTP Request - Get A Random Joke               | ## Update Me                                                                                                              |
| HTTP Request - Get A Random Joke              | HTTP Request         | Fetch random joke for simulation                     | Demo "Trigger" Callback Setup        | Simulate some Consumed Service Time            |                                                                                                                           |
| Simulate some Consumed Service Time           | Wait                 | Simulate delay in external process                   | HTTP Request - Get A Random Joke     | Simulate Event that Hits the 2nd Trigger/Flow |                                                                                                                           |
| Simulate Event that Hits the 2nd Trigger/Flow | HTTP Request        | Trigger secondary workflow with resumeUrl and joke  | Simulate some Consumed Service Time  | None                                          |                                                                                                                           |
| Respond to Webhook                            | Respond to Webhook   | Respond to webhook calls                              | Webhook (disabled)                   | Demo "Trigger" Callback Setup                   |                                                                                                                           |
| Webhook (disabled)                            | Webhook              | Disabled webhook node (demo only)                    | None                                 | Respond to Webhook                             |                                                                                                                           |
| Sticky Note8                                  | Sticky Note          | Explains independent async process concept          | None                                 | None                                          | ## Independent "Async" Process<br>This could be anything that eventually triggers another workflow and passes through something (e.g. resumeUrl) that identifies the original workflow execution that needs to be joined. For instance, this could be a Telegram conversation where the trigger is watching for a message containing a "reply" to something that was originally sent out via Telegram. |
| Sticky Note                                   | Sticky Note          | Warns only one resume call will work                 | None                                 | None                                          | ## Only One Item Will Work<br>If the previous steps could result in multiple initiations via the `Secondary Trigger` below, **only the first one** will resume the workflow. Others will be rejected. |
| Sticky Note1                                  | Sticky Note          | Explains secondary trigger concept                   | None                                 | None                                          | ## Secondary Trigger From Independent Process<br>When something runs the workflow through this trigger, it is a completely separate execution. By passing through the resumeUrl from the **Primary Execution**, it is possible to join back into it via the "webhook callback" to the `Wait` node. * Note: This trigger could be anything that would support input including the `resumeUrl`, not just a webhook. The `Webhook` node is just used to demonstrate a separate trigger. |
| Sticky Note2                                  | Sticky Note          | Labels primary trigger/execution                      | None                                 | None                                          | ## Primary Trigger/Execution                                                                                              |
| Sticky Note3                                  | Sticky Note          | Highlights nodes combining secondary and primary     | None                                 | None                                          | These are the nodes that combine the `Secondary` execution back to the `Primary` execution via the `resumeUrl`.           |
| Sticky Note4                                  | Sticky Note          | Labels main workflow nodes                            | None                                 | None                                          | # Main Workflow - Keep these together in the same workflow instance                                                      |
| Sticky Note5                                  | Sticky Note          | Labels simulated external independent process nodes | None                                 | None                                          | # Simulated External Independent Process<br>Cut/Paste these nodes into a separate workflow instance<br>Then activate the trigger<br>Then activate the workflow |
| Sticky Note6                                  | Sticky Note          | Reminder to update workflow IDs                       | None                                 | None                                          | ## Update Me                                                                                                              |
| Sticky Note7                                  | Sticky Note          | Instruction to execute node to test                   | None                                 | None                                          | ## Execute This Node to Test                                                                                              |
| Sticky Note9                                  | Sticky Note          | Reminder to update workflow IDs                       | None                                 | None                                          | ## Update Me                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Primary Workflow:**

   1.1 Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.

   1.2 Add a **Set** node named `Set Primary Execution Context` connected from the manual trigger.  
       - Add two fields:  
         - `someContextItem` (string): `"this value is only available / in-scope from the primary execution's previous-nodes"`  
         - `simulatedExternalProcessWorkflowId` (string): Set to the workflow ID of your external independent process workflow (must be updated).

   1.3 Add an **HTTP Request** node named `HTTP Request - Initiate Independent Process` connected from the Set node.  
       - Method: POST  
       - URL: `http://127.0.0.1:5678/webhook/{{ $json.simulatedExternalProcessWorkflowId }}` (replace with actual external workflow webhook URL)  
       - Body Parameters:  
         - `resumeUrlForWaitingExecution`: Set to expression `$execution.resumeUrl` (this is the webhook URL to resume this workflow)  
       - Send Body: true  

   1.4 Add a **Wait** node named `Wait` connected from the HTTP Request node.  
       - Resume Mode: `webhook`  
       - HTTP Method: POST  
       - This node will pause execution until resumed by a webhook call.

   1.5 Add a **Set** node named `This Node Can Access Primary and Secondary` connected from the Wait node.  
       - Add two fields:  
         - `somethingFromPrimaryExecution`: Expression referencing `Set Primary Execution Context` node’s `someContextItem`  
         - `somethingFromSecondaryExecution`: Expression referencing the resumed webhook data field `jokeFromIndependentProcess` (from Wait node’s JSON body)  

2. **Create the Secondary Workflow (External Independent Process):**

   2.1 Add a **Webhook** node named `Receive Input from External, Independent Process`.  
       - HTTP Method: POST  
       - Path: Unique webhook path (e.g., UUID)  
       - This node receives the resume URL and other data from the external process.

   2.2 Add an **HTTP Request** node named `HTTP Request - Resume Other Workflow Execution` connected from the webhook node.  
       - Method: POST  
       - URL: Expression to replace `$env.WEBHOOK_URL` with `http://127.0.0.1:5678` in the received `resumeUrlForWaitingExecution` field:  
         `{{$json.body.resumeUrlForWaitingExecution.replace($env.WEBHOOK_URL, 'http://127.0.0.1:5678')}}`  
       - Body Parameters:  
         - `jokeFromIndependentProcess`: Extract from webhook JSON body `joke`  
         - `setupFromIndependentProcess`: Extract from webhook JSON body `setup` (if available)  
         - `deliveryFromIndependentProcess`: Extract from webhook JSON body `delivery` (if available)  
       - Send Body: true  

3. **Create the Simulated External Independent Process Workflow (Optional Demo):**

   3.1 Add a **Webhook** node (disabled) named `Webhook` (for demo only).

   3.2 Add a **Respond to Webhook** node connected from the webhook node.

   3.3 Add a **Set** node named `Demo "Trigger" Callback Setup` connected from the Respond to Webhook node.  
       - Add field `triggerTargetWorkflowId` with the webhook path/ID of the secondary workflow (`Receive Input from External, Independent Process`).

   3.4 Add an **HTTP Request** node named `HTTP Request - Get A Random Joke` connected from the Set node.  
       - URL: `https://v2.jokeapi.dev/joke/Programming`  
       - Query Parameters:  
         - `blacklistFlags`: `nsfw,religious,political,racist,sexist,explicit`  
         - `type`: `single`  

   3.5 Add a **Wait** node named `Simulate some Consumed Service Time` connected from the HTTP Request node.  
       - Wait time: 2 seconds  

   3.6 Add an **HTTP Request** node named `Simulate Event that Hits the 2nd Trigger/Flow` connected from the Wait node.  
       - Method: POST  
       - URL: `http://127.0.0.1:5678/webhook/{{ $json.triggerTargetWorkflowId }}`  
       - Body Parameters:  
         - `resumeUrlForWaitingExecution`: From previous webhook JSON body  
         - `joke`: From the random joke API response  

4. **Credential Setup:**

   - No special credentials are required for this workflow unless the external HTTP endpoints require authentication.  
   - Ensure the environment variable `WEBHOOK_URL` is set correctly if using the URL replacement in the secondary workflow’s HTTP Request node.  
   - For local Docker setups, the replacement to `http://127.0.0.1:5678` is necessary to route webhook calls internally. Adjust accordingly for other environments.

5. **Activation and Testing:**

   - Activate the primary workflow.  
   - Run the manual trigger node `When clicking ‘Test workflow’`.  
   - The workflow will initiate the external process and pause at the `Wait` node.  
   - The external process (simulated or real) must POST to the secondary workflow webhook with the resume URL and data.  
   - The secondary workflow resumes the primary workflow via the resume URL, passing data back.  
   - The primary workflow continues, combining data from both executions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This pattern is NOT suitable for workflows handling multiple items because only the first resume webhook call will be accepted; subsequent calls will be rejected.                                                               | Workflow limitation                                                                                                 |
| Workflow IDs embedded in `Set` nodes labeled **Update Me** must be changed to match your environment for the demo to work.                                                                                                       | Important configuration step                                                                                        |
| The `Resume Other Workflow Execution` node uses `$env.WEBHOOK_URL` replacement to convert URLs for Docker localhost calls. Adjust this logic if running outside Docker or in different environments.                                | Environment-specific configuration                                                                                 |
| The external independent process can be any system capable of triggering the secondary workflow and passing the resume URL, e.g., Telegram conversations, web services, or other workflows.                                       | Integration flexibility                                                                                             |
| Video explanation and detailed blog post available at: https://n8n.io/blog/async-workflows-with-resume-webhook-callbacks (example link, replace with actual if available)                                                        | Additional learning resource                                                                                        |
| The workflow demonstrates a powerful pattern for asynchronous workflow coordination using n8n’s Wait node with webhook resume functionality, enabling complex multi-step processes involving external callbacks.                   | Conceptual overview                                                                                                |

---

This document provides a complete, structured reference to understand, reproduce, and modify the "Pattern for Multiple Triggers Combined to Continue Workflow" in n8n, including all nodes, logic blocks, and integration considerations.