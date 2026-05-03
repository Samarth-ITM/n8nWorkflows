Retry on fail except for known error

https://n8nworkflows.xyz/workflows/retry-on-fail-except-for-known-error-2719


# Retry on fail except for known error

### 1. Workflow Overview

This workflow snippet implements a custom retry mechanism with advanced error handling for n8n nodes that do not support built-in retry logic or where retries need to be conditional based on error types. It is designed to:

- Attempt execution of a target node multiple times (default max 3 tries).
- Detect and handle known errors differently from unexpected errors.
- Retry only on unexpected errors, while immediately branching on known errors.
- Stop execution with an error message when the retry limit is reached.

The workflow is structured into the following logical blocks:

- **1.1 Initialization and Input Setup:** Starts the workflow and initializes the retry counter.
- **1.2 Main Execution Node:** The node to be retried, which can fail and trigger error handling.
- **1.3 Error Detection and Branching:** Filters errors into known and unknown categories.
- **1.4 Retry Loop Control:** Updates retry count, waits between retries, and decides whether to retry or stop.
- **1.5 Success and Known Error Handling:** Separate paths for successful execution and known error cases.
- **1.6 Termination:** Stops the workflow with an error if retry limit is exceeded.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Input Setup

- **Overview:**  
  This block starts the workflow manually and initializes the retry counter variable `tries` to zero or retains its current value if already set.

- **Nodes Involved:**  
  - Manual Trigger  
  - Set tries

- **Node Details:**

  - **Manual Trigger**  
    - Type: Trigger node  
    - Role: Starts the workflow manually for testing or manual execution.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Set tries" node.  
    - Edge Cases: None.

  - **Set tries**  
    - Type: Set node  
    - Role: Initializes or retains the retry counter `tries`.  
    - Configuration: Sets `tries` to `{{$json.tries || 0}}` — if `tries` exists, use it; otherwise, start at 0.  
    - Inputs: From Manual Trigger or from retry loop.  
    - Outputs: Connects to "Replace Me" node (main execution node).  
    - Edge Cases: If `tries` is not a number or missing, defaults to 0.

---

#### 2.2 Main Execution Node

- **Overview:**  
  This node represents the core operation that may fail and needs retry logic applied. It is a placeholder to be replaced by the user with their actual node.

- **Nodes Involved:**  
  - Replace Me

- **Node Details:**

  - **Replace Me**  
    - Type: NoOp node (placeholder)  
    - Role: Placeholder for the user’s actual node that requires retry logic.  
    - Configuration: Error branch enabled (`onError: continueErrorOutput`) to allow error handling downstream.  
    - Inputs: From "Set tries" node.  
    - Outputs: Two outputs — main (success) and error (failure).  
      - Success output connects to "Success" node.  
      - Error output connects to "Catch known error" node.  
    - Edge Cases: Must be replaced by a node with error output enabled; otherwise, error handling will not work.

---

#### 2.3 Error Detection and Branching

- **Overview:**  
  This block filters errors from the main execution node into known errors (expected, handled differently) and unknown errors (trigger retry).

- **Nodes Involved:**  
  - Catch known error (If node)  
  - Known Error (NoOp)  
  - Wait (Wait node)

- **Node Details:**

  - **Catch known error**  
    - Type: If node  
    - Role: Checks if the error message contains a known error substring (default: "could not be found").  
    - Configuration: Condition checks if `$json.error` contains "could not be found" (case sensitive).  
    - Inputs: From error output of "Replace Me".  
    - Outputs:  
      - True: Connects to "Known Error" node (handle known error).  
      - False: Connects to "Wait" node (trigger retry wait).  
    - Edge Cases:  
      - If error message format differs or is missing, condition may fail.  
      - Case sensitivity may cause misses; can be adjusted.

  - **Known Error**  
    - Type: NoOp node  
    - Role: Placeholder for actions to take on known errors (e.g., logging, alternative flows).  
    - Inputs: From "Catch known error" true branch.  
    - Outputs: None (end of known error path).  
    - Edge Cases: User can customize this node to handle known errors as needed.

  - **Wait**  
    - Type: Wait node  
    - Role: Pauses workflow before retrying to avoid immediate retry loops.  
    - Configuration: Default wait time 5 seconds (configurable).  
    - Inputs: From "Catch known error" false branch (unexpected errors).  
    - Outputs: Connects to "Update tries" node.  
    - Edge Cases: Wait duration can be adjusted; too short may cause rapid retries, too long delays processing.

---

#### 2.4 Retry Loop Control

- **Overview:**  
  This block increments the retry counter, checks if retry limit is reached, and either loops back to retry or stops the workflow.

- **Nodes Involved:**  
  - Update tries (Set node)  
  - If tries left (If node)  
  - Set tries (Set node, reused)  
  - Retry limit reached (Stop and Error node)

- **Node Details:**

  - **Update tries**  
    - Type: Set node  
    - Role: Increments the `tries` counter by 1.  
    - Configuration: Sets `tries` to `={{ $('Set tries').item.json.tries + 1 }}` (adds 1 to current tries).  
    - Inputs: From "Wait" node.  
    - Outputs: Connects to "If tries left" node.  
    - Edge Cases: If `tries` is not numeric, expression may fail.

  - **If tries left**  
    - Type: If node  
    - Role: Checks if the current `tries` count is less than the max allowed (default 3).  
    - Configuration: Condition `{{$json.tries}} < 3`.  
    - Inputs: From "Update tries".  
    - Outputs:  
      - True: Connects back to "Set tries" node to loop retry.  
      - False: Connects to "Retry limit reached" node to stop workflow.  
    - Edge Cases: Max tries can be adjusted; if set too low, may stop prematurely.

  - **Set tries** (reused)  
    - Role: Resets or updates the `tries` variable for the next retry loop iteration.  
    - Inputs: From "If tries left" true branch.  
    - Outputs: Connects to "Replace Me" node to retry main execution.  
    - Edge Cases: Same as initial "Set tries" node.

  - **Retry limit reached**  
    - Type: Stop and Error node  
    - Role: Stops the workflow with an error message when retry limit is exceeded.  
    - Configuration: Error message set to "Retry limit reached".  
    - Inputs: From "If tries left" false branch.  
    - Outputs: None (workflow stops).  
    - Edge Cases: Workflow halts here; user can customize error message.

---

#### 2.5 Success and Known Error Handling

- **Overview:**  
  Defines the paths for successful execution and known error handling.

- **Nodes Involved:**  
  - Success (NoOp)  
  - Known Error (NoOp, also part of error detection block)

- **Node Details:**

  - **Success**  
    - Type: NoOp node  
    - Role: Placeholder for actions to take when the main execution node succeeds.  
    - Inputs: From success output of "Replace Me".  
    - Outputs: None (end of success path).  
    - Edge Cases: User can customize this node to continue workflow after success.

  - **Known Error**  
    - See above in error detection block.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                           | Input Node(s)         | Output Node(s)               | Sticky Note                                                                                      |
|--------------------|---------------------|-----------------------------------------|-----------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger     | Manual Trigger      | Starts the workflow                      | None                  | Set tries                   |                                                                                                |
| Set tries          | Set                 | Initializes or updates retry counter    | Manual Trigger, If tries left | Replace Me                 | Define counter for tries                                                                        |
| Replace Me         | NoOp                | Placeholder for main execution node     | Set tries              | Success, Catch known error  | Replace this by the Node which retrieves the admired data. Enable error branch in Node Settings and connect the outputs like in this example |
| Success            | NoOp                | Handles successful execution             | Replace Me             | None                       | Continue here, if the request succeeded                                                        |
| Catch known error  | If                  | Filters known vs unknown errors          | Replace Me (error output) | Known Error, Wait          | Set filter: Filter by status code or error message                                             |
| Known Error        | NoOp                | Handles known errors                      | Catch known error (true) | None                       | Continue here, if the request failed                                                           |
| Wait               | Wait                | Waits before retrying                     | Catch known error (false) | Update tries               | Set Wait: Change duration if needed, default is 5s                                            |
| Update tries       | Set                 | Increments retry counter                  | Wait                   | If tries left               | Update counter for tries                                                                       |
| If tries left      | If                  | Checks if retry limit reached             | Update tries            | Set tries (true), Retry limit reached (false) | Set max tries: Change if needed, default is 3                                                  |
| Retry limit reached| Stop and Error      | Stops workflow when retry limit exceeded | If tries left (false)   | None                       | Stop here, if all tries have failed                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `Manual Trigger`  
   - Purpose: To start the workflow manually.

2. **Create a Set node to initialize tries**  
   - Name: `Set tries`  
   - Parameters:  
     - Add field `tries` (Number) with value expression: `{{$json.tries || 0}}`  
   - Connect `Manual Trigger` output to `Set tries` input.

3. **Create the main execution node placeholder**  
   - Name: `Replace Me`  
   - Type: The actual node you want to retry (e.g., HTTP Request, API node).  
   - Enable error output branch in node settings.  
   - Connect `Set tries` output to `Replace Me` input.

4. **Create a NoOp node for success path**  
   - Name: `Success`  
   - Connect `Replace Me` main output (success) to `Success` input.

5. **Create an If node to catch known errors**  
   - Name: `Catch known error`  
   - Condition:  
     - Type: String contains  
     - Left value: `{{$json.error}}`  
     - Right value: `"could not be found"` (adjust as needed)  
     - Case sensitive: true (adjust if needed)  
   - Connect `Replace Me` error output to `Catch known error` input.

6. **Create a NoOp node for known errors**  
   - Name: `Known Error`  
   - Connect `Catch known error` true output to `Known Error`.

7. **Create a Wait node for retry delay**  
   - Name: `Wait`  
   - Set wait time to 5 seconds (adjustable).  
   - Connect `Catch known error` false output to `Wait`.

8. **Create a Set node to update tries counter**  
   - Name: `Update tries`  
   - Add field `tries` (Number) with expression: `={{ $('Set tries').item.json.tries + 1 }}`  
   - Connect `Wait` output to `Update tries`.

9. **Create an If node to check if tries left**  
   - Name: `If tries left`  
   - Condition:  
     - Type: Number less than  
     - Left value: `{{$json.tries}}`  
     - Right value: `3` (max retries, adjustable)  
   - Connect `Update tries` output to `If tries left`.

10. **Connect If node outputs:**  
    - True output: Connect to `Set tries` node input (loop retry).  
    - False output: Connect to a new Stop and Error node.

11. **Create a Stop and Error node for retry limit reached**  
    - Name: `Retry limit reached`  
    - Error message: `"Retry limit reached"`  
    - Connect `If tries left` false output to this node.

12. **Connect `Set tries` output to `Replace Me` input**  
    - This completes the retry loop.

13. **Optional:** Add sticky notes for instructions and configuration hints.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Replace the “Replace Me” node with your actual node that requires retry logic.                      | Sticky Note near "Replace Me" node                                                               |
| Enable error output branch on the main execution node to allow error handling.                      | Sticky Note near "Replace Me" node                                                               |
| Default max tries is 3; adjust in "If tries left" node condition as needed.                         | Sticky Note near "If tries left" node                                                            |
| Default wait time between retries is 5 seconds; adjust in "Wait" node parameters.                   | Sticky Note near "Wait" node                                                                      |
| Filter known errors by matching error message substring in "Catch known error" node; customize as needed. | Sticky Note near "Catch known error" node                                                        |
| Workflow stops with error message "Retry limit reached" after max retries.                          | Sticky Note near "Retry limit reached" node                                                      |
| Continue workflow after success or known error by customizing "Success" and "Known Error" nodes.   | Sticky Notes near "Success" and "Known Error" nodes                                              |
| This snippet is designed to be copied into existing workflows and integrated with your nodes.      | Description section                                                                              |

---

This documentation provides a complete understanding of the workflow’s structure, logic, and configuration to enable reproduction, modification, and troubleshooting.