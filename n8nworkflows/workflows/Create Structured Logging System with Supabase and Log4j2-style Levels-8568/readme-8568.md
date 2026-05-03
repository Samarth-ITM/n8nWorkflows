Create Structured Logging System with Supabase and Log4j2-style Levels

https://n8nworkflows.xyz/workflows/create-structured-logging-system-with-supabase-and-log4j2-style-levels-8568


# Create Structured Logging System with Supabase and Log4j2-style Levels

### 1. Workflow Overview

This workflow implements a **centralized structured logging system** for n8n, inspired by Log4j2’s logging levels (TRACE, DEBUG, INFO, WARN, ERROR, FATAL). It provides a reusable sub-workflow that logs structured events directly into a Supabase PostgreSQL database, enabling scalable and queryable log storage. The workflow is designed to capture logs from other workflows or error triggers, making it ideal for monitoring, debugging, and analytics across automation stacks.

The workflow’s logic is organized into these main blocks:

- **1.1 Supabase Logging Sub-Workflow**: Receives log entries with structured fields and inserts them into Supabase.
- **1.2 Usage Examples**: Demonstrates how to create logs of different levels (INFO, FATAL) and includes a sample error-triggered logging pattern.
- **1.3 Configuration and Documentation**: Contains detailed instructions and SQL setup scripts, embedded as sticky notes for easy reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Supabase Logging Sub-Workflow

**Overview:**  
This block serves as the core logging mechanism. It acts as a sub-workflow designed to accept log data fields and store them as records in the Supabase `logs` table.

**Nodes Involved:**  
- When Log Traced  
- Create Log

**Node Details:**

- **When Log Traced**  
  - *Type & Role:* `executeWorkflowTrigger` — Entry point node for the sub-workflow, waiting for external triggers with specified log data inputs.  
  - *Configuration:* Accepts inputs: `workflow_name`, `node_name`, `execution_id`, `log_level`, `message`, and `metadata` (object).  
  - *Inputs:* External trigger call with log details.  
  - *Outputs:* Passes input data downstream unmodified.  
  - *Version:* v1.1  
  - *Potential Failures:* Missing expected inputs or malformed metadata object may cause processing errors.

- **Create Log**  
  - *Type & Role:* `supabase` node — Inserts structured log data into Supabase database table.  
  - *Configuration:*  
    - Table: `logs`  
    - Data mapping: `autoMapInputData` (automatically maps input fields to table columns).  
  - *Credentials:* Connects via configured Supabase API credentials.  
  - *Inputs:* Receives all log fields from the trigger node.  
  - *Outputs:* Confirmation of database insert operation.  
  - *Version:* v1  
  - *Potential Failures:* Supabase connection/authentication issues, table schema mismatch, or data validation errors.

---

#### 2.2 Usage Examples for Logging

**Overview:**  
This block shows how to create and send logs of different severity levels to the logging sub-workflow, including a generic error handler that logs FATAL errors.

**Nodes Involved:**  
- Log Info  
- Call Logger SubWorkflow 2  
- Sticky Note3  
- Sticky Note2  
- Error Trigger (disabled)  
- Log Fatal (disabled)  
- Call Logger SubWorkflow 1 (disabled)

**Node Details:**

- **Log Info** (disabled)  
  - *Type & Role:* `code` node — Prepares a structured INFO-level log message.  
  - *Configuration:* Generates a log entry object with:  
    - `workflow_name`: current workflow’s name  
    - `node_name`: previous node’s name or “Unknown node”  
    - `execution_id`: current execution id  
    - `log_level`: `"INFO"`  
    - `message`: placeholder text `"Insert your message here"`  
    - `metadata`: sample JSON object with attributes.  
  - *Inputs:* None (runs independently).  
  - *Outputs:* Structured log JSON object.  
  - *Version:* v2  
  - *Potential Failures:* JavaScript runtime errors if context variables unavailable.

- **Call Logger SubWorkflow 2** (disabled)  
  - *Type & Role:* `executeWorkflow` — Calls the logging sub-workflow asynchronously to insert the log.  
  - *Configuration:*  
    - Workflow ID linked to the logging sub-workflow.  
    - Inputs mapped explicitly from the log object fields: `workflow_name`, `node_name`, `execution_id`, `log_level`, `message`, `metadata`.  
    - Does not wait for sub-workflow completion (`waitForSubWorkflow`: false).  
  - *Inputs:* Receives log data from the `Log Info` node.  
  - *Outputs:* None relevant (fire-and-forget).  
  - *Version:* v1.2  
  - *Potential Failures:* Sub-workflow ID mismatch, mapping errors, or credential issues.

- **Sticky Note3**  
  - *Type & Role:* Documentation node explaining usage of DEBUG/INFO level logging for tracing messages during workflow execution.

- **Sticky Note2**  
  - *Type & Role:* Documentation node describing usage example of logging a FATAL error from an error handler.

- **Error Trigger** (disabled)  
  - *Type & Role:* `errorTrigger` — Captures workflow errors for logging purposes.  
  - *Configuration:* Disabled by default; intended for use as a generic error handler trigger.  
  - *Inputs:* Automatic error event data.  
  - *Outputs:* Passes error info downstream.  
  - *Version:* v1  
  - *Potential Failures:* None when disabled; when enabled, may fail if error data is incomplete.

- **Log Fatal** (disabled)  
  - *Type & Role:* `code` node — Constructs a structured FATAL-level log entry from error trigger data.  
  - *Configuration:* Generates log object with:  
    - `workflow_name`, `node_name`, `execution_id` from error context or fallback values.  
    - `log_level`: `"FATAL"`  
    - `message`: error message or "Unknown error" default.  
    - `metadata`: entire error event JSON.  
  - *Inputs:* Receives error data from `Error Trigger`.  
  - *Outputs:* Structured log object.  
  - *Version:* v2  
  - *Potential Failures:* Missing error context or runtime issues.

- **Call Logger SubWorkflow 1** (disabled)  
  - *Type & Role:* `executeWorkflow` — Calls the logging sub-workflow to record FATAL error logs.  
  - *Configuration:* Similar to Call Logger SubWorkflow 2, but triggered by error events.  
  - *Inputs:* Receives log data from `Log Fatal`.  
  - *Outputs:* None relevant.  
  - *Version:* v1.2

---

#### 2.3 Configuration and Documentation

**Overview:**  
This block contains detailed setup instructions and background information embedded as sticky notes to assist users in preparing the Supabase environment and understanding the workflow usage.

**Nodes Involved:**  
- Sticky Note (Large, left)  
- Sticky Note1 (center top)

**Node Details:**

- **Sticky Note**  
  - *Type & Role:* `stickyNote` — Contains comprehensive instructions for:  
    - Creating Supabase enumerated type `log_level_type` with values (TRACE, DEBUG, INFO, WARN, ERROR, FATAL).  
    - Creating the `logs` table schema with appropriate columns and constraints.  
    - Providing SQL script examples.  
    - Guidance for creating Supabase credentials in n8n.  
  - *Inputs/Outputs:* None (pure documentation).  
  - *Version:* v1

- **Sticky Note1**  
  - *Type & Role:* `stickyNote` — Labels the sub-workflow section for better visual grouping and clarity.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                             | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                                                                |
|--------------------------|-------------------------|---------------------------------------------|--------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note              | stickyNote              | Documentation for Supabase setup and usage  |                          |                           | Contains detailed instructions for Supabase enum, table creation, and credential setup                                                     |
| Sticky Note1             | stickyNote              | Label for sub-workflow section               |                          |                           | ## SUB-WORKFLOW                                                                                                                            |
| When Log Traced          | executeWorkflowTrigger  | Sub-workflow entry point for receiving logs |                          | Create Log                |                                                                                                                                            |
| Create Log               | supabase                | Inserts log entry into Supabase `logs` table | When Log Traced          |                           | Requires configured Supabase API credentials                                                                                            |
| Sticky Note2             | stickyNote              | Usage example: FATAL log from error handler |                          |                           | Generic error handler workflow example                                                                                                    |
| Error Trigger            | errorTrigger            | Captures workflow errors for logging        |                          | Log Fatal                 | Disabled by default                                                                                                                        |
| Log Fatal                | code                    | Constructs FATAL-level log from error data   | Error Trigger            | Call Logger SubWorkflow 1  | Disabled by default                                                                                                                        |
| Call Logger SubWorkflow 1| executeWorkflow         | Calls logging sub-workflow for FATAL logs   | Log Fatal                |                           | Disabled by default; async call to logging sub-workflow                                                                                   |
| Sticky Note3             | stickyNote              | Usage example for DEBUG/INFO messages        |                          |                           | Instructions on customizing log level, message, and metadata                                                                              |
| Log Info                 | code                    | Prepares INFO-level log entry                 |                          | Call Logger SubWorkflow 2  | Disabled by default                                                                                                                        |
| Call Logger SubWorkflow 2| executeWorkflow         | Calls logging sub-workflow for INFO logs    | Log Info                 |                           | Disabled by default; async call to logging sub-workflow                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Supabase Credentials in n8n**  
   - Go to Credentials → Create new → Supabase API.  
   - Enter Supabase URL and API key. Save as e.g. `Supabase account`.

2. **Set Up Supabase Database**  
   - Create enumerated type `log_level_type` with values: TRACE, DEBUG, INFO, WARN, ERROR, FATAL.  
   - Create table `logs` with columns:  
     - `id`: `bigserial` primary key  
     - `created_at`: timestamp with time zone, default now()  
     - `workflow_name`: text, NOT NULL  
     - `node_name`: text, NOT NULL  
     - `execution_id`: text, NOT NULL  
     - `log_level`: `log_level_type`, NOT NULL  
     - `message`: text, NOT NULL  
     - `metadata`: jsonb, nullable

3. **Create Logging Sub-Workflow**

   - Create a new workflow named e.g. `Logger Sub-Workflow`.
   - Add an **Execute Workflow Trigger** node named `When Log Traced`.  
     - Add inputs: `workflow_name` (string), `node_name` (string), `execution_id` (string), `log_level` (string), `message` (string), `metadata` (object).  
   - Add a **Supabase** node named `Create Log`.  
     - Set operation to Insert.  
     - Select Table: `logs`.  
     - Set Data to Send: `autoMapInputData` (auto maps all input fields).  
     - Assign Supabase API credentials created earlier.  
   - Connect `When Log Traced` → `Create Log`.
   - Save and activate this workflow.

4. **Create Example Usage Workflow**

   - Create a new workflow for testing logging.
   - Add a **Code** node named `Log Info`.  
     - Set mode: Run Once For Each Item.  
     - Use the following JavaScript code:
       ```javascript
       return {
           workflow_name: $workflow.name,
           node_name: $prevNode.name || "Unknown node",
           execution_id: $execution.id,
           log_level: "INFO",
           message: "Insert your message here",
           metadata: {
               attr1: "value1",
               attr2: 123
           }
       };
       ```
   - Add an **Execute Workflow** node named `Call Logger SubWorkflow 2`.  
     - Select the Logger Sub-Workflow created above.  
     - Disable "Wait For Sub-Workflow".  
     - Map inputs explicitly:  
       - `workflow_name`: `={{ $json.workflow_name }}`  
       - `node_name`: `={{ $json.node_name }}`  
       - `execution_id`: `={{ $json.execution_id }}`  
       - `log_level`: `={{ $json.log_level }}`  
       - `message`: `={{ $json.message }}`  
       - `metadata`: `={{ $json.metadata }}`  
   - Connect `Log Info` → `Call Logger SubWorkflow 2`.

5. **(Optional) Setup Error Logging**

   - Add an **Error Trigger** node named `Error Trigger` (enable it).  
   - Add a **Code** node named `Log Fatal`.  
     - Mode: Run Once For Each Item.  
     - Use JavaScript code:
       ```javascript
       const e = $json; // error object
       return {
           workflow_name: e.workflow?.name ?? $workflow.name,
           node_name: e.execution?.lastNodeExecuted ?? $prevNode.name,
           execution_id: e.execution?.id ?? $execution.id,
           log_level: "FATAL",
           message: e.execution?.error?.message ?? "Unknown error",
           metadata: e
       };
       ```
   - Add an **Execute Workflow** node named `Call Logger SubWorkflow 1`.  
     - Select Logger Sub-Workflow.  
     - Disable "Wait For Sub-Workflow".  
     - Map inputs similarly to step 4.  
   - Connect `Error Trigger` → `Log Fatal` → `Call Logger SubWorkflow 1`.

6. **Add Sticky Notes**  
   - Add sticky notes to document Supabase setup, sub-workflow purpose, and usage examples as per original workflow.

7. **Test the Workflow**  
   - Trigger the example workflow manually.  
   - Verify that logs appear in Supabase `logs` table with correct fields and timestamps.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow template is inspired by Log4j2 logging levels and designed for centralized logging in n8n using Supabase as backend.                                                                                                          | Workflow Purpose                                                                                         |
| Supabase credential setup documentation: https://docs.n8n.io/integrations/builtin/credentials/supabase/                                                                                                                                      | Credential Setup                                                                                         |
| SQL scripts for enum and table creation provided in sticky note for easy database preparation.                                                                                                                                                 | Database Setup                                                                                           |
| Usage examples demonstrate how to integrate logging calls in workflows and error handlers for scalable monitoring.                                                                                                                           | Usage Examples                                                                                           |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting all applicable content policies with no illegal or protected elements. All data handled is legal and public.