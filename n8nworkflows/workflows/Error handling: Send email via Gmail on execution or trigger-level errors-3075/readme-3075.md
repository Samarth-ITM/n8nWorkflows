Error handling: Send email via Gmail on execution or trigger-level errors

https://n8nworkflows.xyz/workflows/error-handling--send-email-via-gmail-on-execution-or-trigger-level-errors-3075


# Error handling: Send email via Gmail on execution or trigger-level errors

### 1. Workflow Overview

This workflow is designed for comprehensive error handling in n8n automation environments. Its primary purpose is to send detailed email notifications via Gmail whenever an error occurs either during the execution of a workflow or at the trigger level. It extends the standard error notification template by including trigger-level error coverage, providing richer context and actionable information.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures errors from any workflow or trigger using the `Error Trigger` node.
- **1.2 Configuration Setup:** Defines essential configuration parameters such as app URL, recipient email, and sender name.
- **1.3 Context Enrichment:** Constructs URLs and flags to enrich error context and distinguish between execution and trigger errors.
- **1.4 Error Type Branching:** Determines if the error is an execution error or a trigger failure to tailor the notification content.
- **1.5 Error Detail Formatting:** Generates HTML-formatted error details specific to execution errors or trigger failures.
- **1.6 Error Content Merging:** Combines the HTML outputs from the two error types into a single payload.
- **1.7 Email Dispatch:** Sends the composed error notification email via Gmail with detailed subject and body content.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for any error events occurring in workflows or triggers and initiates the error handling process.

- **Nodes Involved:**  
  - `Error Trigger`

- **Node Details:**  
  - **Type:** `Error Trigger` (core n8n node)  
  - **Role:** Captures workflow execution or trigger errors and passes error data downstream.  
  - **Configuration:** Default, no parameters set.  
  - **Expressions/Variables:** Outputs error details in JSON format, including `execution`, `trigger`, and `workflow` error data.  
  - **Connections:** Outputs to `Config` node.  
  - **Version Requirements:** n8n version supporting `Error Trigger` node (generally v0.154+).  
  - **Edge Cases:** If no error occurs, this node does not trigger; if error data is malformed, downstream nodes may fail.  
  - **Sub-workflow:** None.

#### 2.2 Configuration Setup

- **Overview:**  
  Sets static configuration values such as the base URL of the n8n instance, recipient email for notifications, and sender name for emails.

- **Nodes Involved:**  
  - `Config`

- **Node Details:**  
  - **Type:** `Set` node  
  - **Role:** Defines configuration parameters under the `config` namespace using dot notation.  
  - **Configuration:**  
    - `config.appUrl`: Base URL of the n8n app (e.g., `https://YourAccountName.app.n8n.cloud/`)  
    - `config.emailing.sendTo`: Recipient email address for error notifications  
    - `config.emailing.senderName`: Sender name for the Gmail messages  
  - **Expressions/Variables:** Static string values assigned.  
  - **Connections:** Outputs to `Constants` node.  
  - **Version Requirements:** Supports dot notation (n8n v0.154+).  
  - **Edge Cases:** Misconfigured URLs or emails will cause invalid links or failed email delivery.  
  - **Sub-workflow:** None.

#### 2.3 Context Enrichment

- **Overview:**  
  Constructs enriched context data including URLs to the failed workflow and error handling workflow, and flags to identify error type.

- **Nodes Involved:**  
  - `Constants`

- **Node Details:**  
  - **Type:** `Set` node  
  - **Role:** Creates additional JSON fields for workflow URLs, error type flags, and error handling workflow metadata.  
  - **Configuration:**  
    - `workflow.url`: Concatenates `config.appUrl` + `"workflow/"` + failed workflow ID  
    - `workflow.isExecutionError`: Boolean flag indicating if the error is an execution error (true) or trigger error (false)  
    - `errorHandlingWorkflow.id`, `.name`, `.url`: Metadata about this error handling workflow itself  
  - **Expressions/Variables:** Uses expressions to dynamically build URLs and flags based on incoming JSON.  
  - **Connections:** Outputs to both `Merge` and `If` nodes.  
  - **Version Requirements:** Supports dot notation and expressions (n8n v0.154+).  
  - **Edge Cases:** If `execution` or `workflow` data is missing, URLs or flags may be incorrect.  
  - **Sub-workflow:** None.

#### 2.4 Error Type Branching

- **Overview:**  
  Routes the workflow based on whether the error is an execution error or a trigger failure to generate appropriate error details.

- **Nodes Involved:**  
  - `If`

- **Node Details:**  
  - **Type:** `If` node  
  - **Role:** Checks if `workflow.isExecutionError` is true to branch logic.  
  - **Configuration:** Condition: `{{$json.workflow.isExecutionError}} === true`  
  - **Expressions/Variables:** Uses JSON path to evaluate error type flag.  
  - **Connections:**  
    - True branch to `HTML for Execution Error`  
    - False branch to `HTML for Trigger Error`  
  - **Version Requirements:** Supports version 2 conditions (n8n v0.154+).  
  - **Edge Cases:** If flag is missing or malformed, branching may fail or misroute.  
  - **Sub-workflow:** None.

#### 2.5 Error Detail Formatting

- **Overview:**  
  Generates HTML-formatted error details tailored to the error type for inclusion in the notification email.

- **Nodes Involved:**  
  - `HTML for Execution Error`  
  - `HTML for Trigger Error`

- **Node Details:**  

  - **HTML for Execution Error:**  
    - **Type:** `HTML` node  
    - **Role:** Formats execution error details including execution URL, ID, retry info, last executed node, mode, error message, and stack trace.  
    - **Configuration:** Uses HTML template with embedded expressions referencing `$json.execution` fields.  
    - **Connections:** Outputs to `KeepEitherOfHTMLs` node.  
    - **Edge Cases:** Missing execution data leads to incomplete details.

  - **HTML for Trigger Error:**  
    - **Type:** `HTML` node  
    - **Role:** Formats trigger failure details including mode, error message, timestamp, name, description, context, cause details, and stack trace.  
    - **Configuration:** Uses HTML template with embedded expressions referencing `$json.trigger.error` and related fields.  
    - **Connections:** Outputs to `KeepEitherOfHTMLs` node.  
    - **Edge Cases:** Missing trigger error data leads to incomplete details.

- **Version Requirements:** Support for `HTML` node with expressions (n8n v0.154+).  
- **Sub-workflow:** None.

#### 2.6 Error Content Merging

- **Overview:**  
  Combines the HTML outputs from the execution and trigger error formatting nodes into a single payload for email.

- **Nodes Involved:**  
  - `KeepEitherOfHTMLs`  
  - `Merge`

- **Node Details:**  
  - **KeepEitherOfHTMLs:**  
    - **Type:** `Merge` node  
    - **Role:** Combines two inputs using "combine" mode with `includeUnpaired` option to allow either HTML block to pass through if the other is missing.  
    - **Configuration:** Combine by position, include unpaired data.  
    - **Connections:** Inputs from `HTML for Execution Error` and `HTML for Trigger Error`; outputs to `Merge`.  
    - **Edge Cases:** If both inputs are missing, output is empty.

  - **Merge:**  
    - **Type:** `Merge` node  
    - **Role:** Combines enriched context data from `Constants` with the merged HTML error details from `KeepEitherOfHTMLs`.  
    - **Configuration:** Combine by position.  
    - **Connections:** Inputs from `Constants` and `KeepEitherOfHTMLs`; outputs to `Gmail`.  
    - **Edge Cases:** Missing inputs cause incomplete email data.

- **Version Requirements:** Support for `Merge` node v3+.  
- **Sub-workflow:** None.

#### 2.7 Email Dispatch

- **Overview:**  
  Sends the composed error notification email via Gmail with a subject line and body containing detailed error information and links.

- **Nodes Involved:**  
  - `Gmail`

- **Node Details:**  
  - **Type:** `Gmail` node  
  - **Role:** Sends email using Gmail OAuth2 credentials.  
  - **Configuration:**  
    - `sendTo`: Recipient email from `config.emailing.sendTo`  
    - `subject`: Dynamic subject including workflow ID, name, error source (execution or trigger), and error message  
    - `message`: HTML body containing:  
      - Links to failed workflow and error handling workflow  
      - Error details in HTML format  
      - Machine-readable JSON block with enriched error data (`execution`, `trigger`, `workflow`, `errorHandlingWorkflow`)  
    - `options.senderName`: Sender name from `config.emailing.senderName`  
  - **Expressions/Variables:** Uses multiple embedded expressions referencing JSON fields for dynamic content.  
  - **Connections:** Final node; no outputs.  
  - **Version Requirements:** Gmail OAuth2 credentials must be configured and valid. Node version 2.1 used.  
  - **Edge Cases:**  
    - Authentication failures with Gmail OAuth2 credentials  
    - Email sending limits or quota exceeded  
    - Invalid recipient email address  
    - Expression evaluation errors in subject or body  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name               | Node Type        | Functional Role                          | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                                  |
|-------------------------|------------------|----------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Error Trigger           | Error Trigger    | Captures workflow or trigger errors    | (Trigger)                  | Config                      |                                                                                                                              |
| Config                  | Set              | Defines app URL, recipient email, sender name | Error Trigger             | Constants                   | Define your n8n app base url, notifications recipient email, sender name                                                     |
| Constants               | Set              | Builds URLs and flags for error context | Config                     | Merge, If                   |                                                                                                                              |
| If                      | If               | Branches logic by error type            | Constants                  | HTML for Execution Error (true), HTML for Trigger Error (false) |                                                                                                                              |
| HTML for Execution Error| HTML             | Formats execution error details          | If (true)                  | KeepEitherOfHTMLs           |                                                                                                                              |
| HTML for Trigger Error  | HTML             | Formats trigger error details            | If (false)                 | KeepEitherOfHTMLs           |                                                                                                                              |
| KeepEitherOfHTMLs       | Merge            | Combines either execution or trigger HTML | HTML for Execution Error, HTML for Trigger Error | Merge                      |                                                                                                                              |
| Merge                   | Merge            | Combines enriched context with error HTML | Constants, KeepEitherOfHTMLs | Gmail                      |                                                                                                                              |
| Gmail                   | Gmail            | Sends detailed error notification email | Merge                      |                             | Setup your Gmail account credentials here.                                                                                   |
| Sticky Note             | Sticky Note      | Notes on Config node                     |                            |                             | Define your n8n app base url, notifications recipient email, sender name                                                     |
| Sticky Note1            | Sticky Note      | Workflow description and instructions   |                            |                             | See workflow description for detailed usage and configuration instructions including links to related resources and author. |
| Sticky Note2            | Sticky Note      | Gmail credentials setup note             |                            |                             | Setup your Gmail account credentials here.                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Error Trigger` node:**  
   - Type: `Error Trigger`  
   - Purpose: To catch any workflow or trigger errors and start the error handling process.  
   - No special configuration needed.

2. **Create `Config` node:**  
   - Type: `Set`  
   - Enable dot notation.  
   - Assign the following values:  
     - `config.appUrl` = `"https://YourAccountName.app.n8n.cloud/"` (replace with your n8n instance URL)  
     - `config.emailing.sendTo` = `"recipient@gmail.com"` (replace with your notification recipient email)  
     - `config.emailing.senderName` = `"Marvin the Yeoman Warder"` (replace with desired sender name)  
   - Connect `Error Trigger` output to `Config` input.

3. **Create `Constants` node:**  
   - Type: `Set`  
   - Enable dot notation.  
   - Assign the following expressions:  
     - `workflow.url` = `{{$json.config.appUrl + "workflow/" + $json.workflow.id}}`  
     - `workflow.isExecutionError` = `{{Boolean($json.execution)}}`  
     - `errorHandlingWorkflow.id` = `{{$workflow.id}}`  
     - `errorHandlingWorkflow.name` = `{{$workflow.name}}`  
     - `errorHandlingWorkflow.url` = `{{$json.config.appUrl + "workflow/" + $workflow.id}}`  
   - Connect `Config` output to `Constants` input.

4. **Create `If` node:**  
   - Type: `If`  
   - Condition: Check if `{{$json.workflow.isExecutionError}} === true`  
   - Connect `Constants` output to `If` input.

5. **Create `HTML for Execution Error` node:**  
   - Type: `HTML`  
   - HTML content:  
     ```
     <h2>Execution details</h2>
     <p>See execution details at <a href="{{ $json.execution.url }}">{{ $json.execution.url }}</a></p>
     <p>Execution id: {{ $json.execution.id }}</p>
     <p>retryOf: {{ $json.execution.retryOf }}</p>
     <p>lastNodeExecuted: {{ $json.execution.lastNodeExecuted }}</p>
     <p>mode: {{ $json.execution.mode }}</p>
     <p>Message: {{ $json.execution.error.message }}</p>
     <h3>Stack trace</h3>
     <pre>
     {{ $json.execution.error.stack }}
     </pre>
     ```  
   - Connect `If` node's true output to this node.

6. **Create `HTML for Trigger Error` node:**  
   - Type: `HTML`  
   - HTML content:  
     ```
     <h2>Trigger failure details</h2>
     <p>A trigger on main workflow has thrown an error.</p>
     <h3>Mode</h3>
     <p>{{ $json.trigger.mode }}</p>
     <h3>Error</h3>
     <p>Message: {{ $json.trigger.error.message }}</p>
     <p>DateTime: {{ DateTime.fromMillis($json.trigger.error.timestamp) }}</p>
     <p>Name: {{ $json.trigger.error.name }}</p>
     <p>Description: {{ $json.trigger.error.description }}</p>
     <p>Context:<br/>
     <pre>{{ JSON.stringify($json.trigger.error.context, null, 2) }}</pre></p>

     <h3>Cause</h3>
     <p>Message: {{ $json.trigger.error.cause.message }}</p>
     <p>Name: {{ $json.trigger.error.cause.name }}</p>
     <p>Code:{{ $json.trigger.error.cause.code }} </p>
     <p>Status: {{ $json.trigger.error.cause.status }}</p>
     <h3>Stack trace</h3>
     <pre>
     {{ $json.trigger.error.cause.stack }}
     </pre>
     ```  
   - Connect `If` node's false output to this node.

7. **Create `KeepEitherOfHTMLs` node:**  
   - Type: `Merge`  
   - Mode: `combine`  
   - Combine By: `combineByPosition`  
   - Options: Enable `includeUnpaired` to allow either HTML block to pass if the other is missing.  
   - Connect outputs of `HTML for Execution Error` and `HTML for Trigger Error` to inputs of this node.

8. **Create `Merge` node:**  
   - Type: `Merge`  
   - Mode: `combine`  
   - Combine By: `combineByPosition`  
   - Connect `Constants` output to first input, and `KeepEitherOfHTMLs` output to second input.

9. **Create `Gmail` node:**  
   - Type: `Gmail`  
   - Configure Gmail OAuth2 credentials (create or select existing).  
   - Parameters:  
     - `sendTo`: `={{ $json.config.emailing.sendTo }}`  
     - `subject`:  
       ```
       =Workflow {{ $json.workflow.id }} ({{ $json.workflow.name }}) {{ $json.workflow.isExecutionError ? "execution error" : "trigger failure" }}: {{ $json.execution.error.message || $json.trigger.error.message }}
       ```  
     - `message`:  
       ```
       <p>Workflow <a href="{{ $json.workflow.url }}">{{ $json.workflow.id }}</a> ({{ $json.workflow.name }})<br/>
       has triggered the error handling workflow <a href="{{ $json.errorHandlingWorkflow.url }}">{{ $json.errorHandlingWorkflow.id }}</a> ({{ $json.errorHandlingWorkflow.name }})<br/>
       with the error details below.</p>
       {{ $json.html }}
       <h2>Error handling JSON (complete error handling data)</h2>
       <pre>
       {{ JSON.stringify({
         execution: $json.execution,
         trigger: $json.trigger,
         workflow: $json.workflow,
         errorHandlingWorkflow: $json.errorHandlingWorkflow,
       }, null, 2) }}
       </pre>
       ```  
     - `options.senderName`: `={{ $json.config.emailing.senderName }}`  
   - Connect `Merge` output to `Gmail` input.

10. **Activate and test:**  
    - Save the workflow.  
    - Configure your main workflows to use this workflow as their error workflow.  
    - Trigger errors to verify email notifications are sent correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow extends the official n8n error email template by adding trigger-level error notifications.             | https://n8n.io/workflows/696-send-email-via-gmail-on-workflow-error/                                |
| For detailed documentation on the Error Trigger node, see n8n docs.                                                 | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.errortrigger/                    |
| To configure this workflow as an error workflow, set it in your main workflow’s settings under “Error Workflow”.     | https://docs.n8n.io/flow-logic/error-handling/#create-and-set-an-error-workflow                      |
| Author contact and community support available via Olek on n8n community and creators hub.                           | https://community.n8n.io/u/olek/summary and https://n8n.io/creators/olek/                            |
| Gmail OAuth2 credentials must be properly configured to enable email sending without authentication errors.          | Setup via n8n credentials manager with Gmail OAuth2.                                                |

---

This document fully describes the workflow structure, logic, and configuration necessary to understand, reproduce, and maintain the error handling workflow for sending Gmail notifications on both execution and trigger-level errors in n8n.