Error Handling System with PostgreSQL Logging and Rate-Limited Notifications

https://n8nworkflows.xyz/workflows/error-handling-system-with-postgresql-logging-and-rate-limited-notifications-3882


# Error Handling System with PostgreSQL Logging and Rate-Limited Notifications

### 1. Workflow Overview

This workflow is designed to **log all errors into a PostgreSQL database** while **limiting notification alerts** (e.g., emails or push notifications) to avoid spamming when many errors happen in quick succession. It is ideal for scenarios where scheduled tasks or services produce bursts of errors that would otherwise overwhelm alert channels.

**Target Use Cases:**
- Centralized error logging with persistence.
- Rate-limiting of alerts to a configurable interval (default 5 minutes).
- Integration as a standalone primary error handler or as a sub-workflow called by other error-handling workflows.
- Optional cleanup of error logs, useful in development environments.

**Logical Blocks:**

- **1.1 Error Event Reception:** Captures error events triggered by any workflow error.
- **1.2 Logging Errors to PostgreSQL:** Inserts detailed error information into the configured database table.
- **1.3 Rate-Limiting Check:** Queries the database to count error logs from the last 5 minutes to decide whether to send notifications.
- **1.4 Conditional Execution & Cleanup:** If no recent logs exist, proceeds to cleanup the execution context and allows subsequent error handling logic.
- **1.5 (Disabled) Notification Setup:** Sample email and push notification nodes (disabled by default) demonstrating how to alert on errors.
- **1.6 (Optional) Database Log Cleanup:** Manual trigger node allowing deletion of all logs in the error table, primarily for development.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Error Event Reception

- **Overview:**  
  Detects when an error occurs in any workflow execution and triggers this workflow accordingly.

- **Nodes Involved:**  
  - `Error Trigger`

- **Node Details:**

  - **Error Trigger**  
    - Type: Error Trigger node (special n8n trigger node)  
    - Configuration: No parameters; automatically triggers on any workflow error.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to `Insert Log` and `Count for 5 minutes` nodes.  
    - Edge Cases: Must ensure this workflow is activated and linked properly to catch errors. Inactive or misconfigured triggers will prevent error capture.

---

#### Block 1.2: Logging Errors to PostgreSQL

- **Overview:**  
  Inserts a record into the PostgreSQL table `N8Err` with detailed error information, including stack trace, message, workflow name, execution URL, and last node executed.

- **Nodes Involved:**  
  - `Insert Log`

- **Node Details:**

  - **Insert Log**  
    - Type: PostgreSQL node  
    - Configuration:  
      - Operation: Insert into table `N8Err` under schema `p1gq6ljdsam3x1m`.  
      - Columns populated:  
        - `created_at`: current timestamp  
        - `title`: workflow name from error context  
        - `URL`: URL of the execution with error  
        - `Stack`: error stack trace  
        - `Message`: error message  
        - `LastNode`: last node executed before error  
        - `json`: entire error JSON stringified for full context  
      - Uses expressions to extract data from the error trigger’s JSON payload.  
    - Inputs: From `Error Trigger`  
    - Outputs: Connects to `Count for 5 minutes` node.  
    - Credentials: Requires properly configured PostgreSQL credentials.  
    - Edge Cases:  
      - Database connection errors must be handled externally or workflow may fail.  
      - Invalid or missing error properties could cause expression evaluation failures.  
      - Table and schema must exist with correct permissions.  
      - Large error stack traces may cause payload size issues if extremely large.

---

#### Block 1.3: Rate-Limiting Check

- **Overview:**  
  Queries the database to count how many error logs have been created in the last 5 minutes. This count determines if the workflow should continue with alert notifications or skip them to avoid flooding.

- **Nodes Involved:**  
  - `Count for 5 minutes`  
  - `If there is no logs in 5 minutes`

- **Node Details:**

  - **Count for 5 minutes**  
    - Type: PostgreSQL node  
    - Configuration:  
      - Executes SQL query:  
        ```sql
        SELECT count(*) FROM p1gq6ljdsam3x1m."N8Err" WHERE created_at >= $1;
        ```  
      - Query parameter `$1` is dynamically set to current time minus 5 minutes using expression `{{$now.minus(5, 'minutes').toString()}}`.  
    - Inputs: From `Insert Log`  
    - Outputs: Connects to `If there is no logs in 5 minutes`  
    - Credentials: PostgreSQL credentials required.  
    - Edge Cases:  
      - Timezone mismatches between database and workflow may affect query accuracy.  
      - Database connectivity issues need to be considered.

  - **If there is no logs in 5 minutes**  
    - Type: If node (conditional logic)  
    - Configuration:  
      - Checks if the count result from the previous node is less than or equal to 0 (`{{$json.count}} <= 0`).  
      - If true (no logs in last 5 minutes), proceeds to cleanup.  
      - If false (logs exist), terminates without proceeding further.  
    - Inputs: From `Count for 5 minutes`  
    - Outputs:  
      - True branch: Connects to `CleanUp execution. See below if you will prepend this workflow`  
      - False branch: No connections (workflow ends).  
    - Edge Cases:  
      - If count field does not exist or query returns unexpected format, condition may fail.  
      - Loose type validation is enabled to reduce false negatives.

---

#### Block 1.4: Conditional Execution & Cleanup

- **Overview:**  
  If no recent errors were logged, this block cleans up the execution context and allows subsequent error handling logic to execute.

- **Nodes Involved:**  
  - `CleanUp execution. See below if you will prepend this workflow`  
  - `Insert your error handling logic after this`

- **Node Details:**

  - **CleanUp execution. See below if you will prepend this workflow**  
    - Type: Code node  
    - Configuration:  
      - JavaScript code returns empty array `return [];` to clear current execution data.  
      - Effectively terminates the current execution branch cleanly when prepended.  
    - Inputs: From `If there is no logs in 5 minutes` (true branch)  
    - Outputs: Connects to `Insert your error handling logic after this`  
    - Edge Cases: None significant; code is simple.

  - **Insert your error handling logic after this**  
    - Type: No Operation (NoOp) node  
    - Configuration: Placeholder for users to add their own error handling logic after cleanup.  
    - Inputs: From `CleanUp execution`  
    - Outputs: None (end node)  
    - Edge Cases: None.

---

#### Block 1.5: (Disabled) Notification Setup Example

- **Overview:**  
  Demonstrates how to send email and push notifications on errors, disabled by default to avoid spamming.

- **Nodes Involved:**  
  - `Principal E-Mail` (disabled)  
  - `Fallback E-Mail` (disabled)  
  - `Push mobile notification` (disabled)  
  - `Call this Sample - Prepend to your error catcher` (disabled)

- **Node Details:**

  - **Principal E-Mail**  
    - Type: Email Send node  
    - Configuration:  
      - Sends plain text email with error URL, last executed node, error message, and stack trace.  
      - From and To emails configured (`suporte@ideias.casa` and `davimesquita@gmail.com`).  
      - Subject includes workflow name and error indicator.  
    - Disabled: true  
    - Credentials: SMTP credentials (`SMTP Resent`) required.  
    - Inputs: From `Call this Sample - Prepend to your error catcher`  
    - Outputs: On error, continues to `Fallback E-Mail`.

  - **Fallback E-Mail**  
    - Type: Email Send node  
    - Configuration similar to `Principal E-Mail` but uses a different sender email (`contato@ideias.casa`) and SMTP credentials.  
    - Disabled: true  
    - Inputs: From `Principal E-Mail` error output.

  - **Push mobile notification**  
    - Type: Pushover node  
    - Configuration:  
      - Sends push notification with error details including workflow name, URL, last node, message, and stack.  
      - User key configured statically.  
    - Disabled: true  
    - Credentials: Pushover API credentials required.  
    - Inputs: From `Call this Sample - Prepend to your error catcher`.

  - **Call this Sample - Prepend to your error catcher**  
    - Type: Execute Workflow node  
    - Disabled: true  
    - Configuration: Placeholder for referencing this workflow as a sub-workflow. No workflow ID set by default.  
    - Outputs: Connects to `Principal E-Mail` and `Push mobile notification`.

---

#### Block 1.6: (Optional) Database Log Cleanup

- **Overview:**  
  Provides a manual trigger to delete all error logs from the database and reset the sequence, intended for development or testing environments.

- **Nodes Involved:**  
  - `Sometimes... just cleanup`  
  - `Truncate Log Database`

- **Node Details:**

  - **Sometimes... just cleanup**  
    - Type: Manual Trigger node  
    - Configuration: No parameters; user manually triggers this cleanup.  
    - Inputs: None (trigger)  
    - Outputs: Connects to `Truncate Log Database`.

  - **Truncate Log Database**  
    - Type: PostgreSQL node  
    - Configuration:  
      - Operation: Delete all rows from table `N8Err` and restart sequences to reset primary keys.  
      - Schema and table explicitly set.  
    - Inputs: From `Sometimes... just cleanup`  
    - Credentials: PostgreSQL credentials required.  
    - Edge Cases:  
      - Should never be run in production without caution, as it deletes all error logs irreversibly.

---

### 3. Summary Table

| Node Name                                   | Node Type           | Functional Role                          | Input Node(s)                      | Output Node(s)                                      | Sticky Note                                                                                                   |
|---------------------------------------------|---------------------|----------------------------------------|----------------------------------|----------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Error Trigger                               | Error Trigger       | Captures workflow errors               | None                             | Insert Log, Count for 5 minutes                     |                                                                                                               |
| Insert Log                                  | PostgreSQL          | Inserts error record into DB            | Error Trigger                   | Count for 5 minutes                                |                                                                                                               |
| Count for 5 minutes                         | PostgreSQL          | Counts errors logged in last 5 minutes | Insert Log                      | If there is no logs in 5 minutes                    |                                                                                                               |
| If there is no logs in 5 minutes            | If                  | Checks if error count ≤ 0 in last 5 min | Count for 5 minutes             | CleanUp execution (true), End (false)               |                                                                                                               |
| CleanUp execution. See below if you will prepend this workflow | Code                | Clears execution data for workflow prepending | If there is no logs in 5 minutes | Insert your error handling logic after this          |                                                                                                               |
| Insert your error handling logic after this | NoOp                 | Placeholder for user error handling     | CleanUp execution               | None                                               |                                                                                                               |
| Principal E-Mail                            | Email Send          | Sends primary email notification       | Call this Sample - Prepend       | Fallback E-Mail                                    | Disabled by default                                                                                           |
| Fallback E-Mail                            | Email Send          | Sends fallback email if primary fails   | Principal E-Mail (error output)  | None                                               | Disabled by default                                                                                           |
| Push mobile notification                    | Pushover            | Sends push notification on error        | Call this Sample - Prepend       | None                                               | Disabled by default                                                                                           |
| Call this Sample - Prepend to your error catcher | Execute Workflow    | Example sub-workflow call for notifications | None                         | Principal E-Mail, Push mobile notification          | Disabled by default                                                                                           |
| Sometimes... just cleanup                   | Manual Trigger      | Manual trigger to clean up error logs   | None                          | Truncate Log Database                              |                                                                                                               |
| Truncate Log Database                       | PostgreSQL          | Deletes all error logs from DB           | Sometimes... just cleanup       | None                                               | # Database Cleanup: Useful in DEV, but DO NOT run in production                                              |
| See below to prepend this at your error handling | Execute Workflow Trigger | Sample trigger to prepend this workflow | None                          | Insert Log, Count for 5 minutes                      |                                                                                                               |
| Sticky Note                                 | Sticky Note         | Contains workflow description and setup | None                          | None                                               | See detailed workflow overview and DDL instructions with author credits and setup notes                      |
| Sticky Note1                                | Sticky Note         | Error handling sample label              | None                          | None                                               | # Error handling sample                                                                                        |
| Sticky Note2                                | Sticky Note         | Database cleanup warning                  | None                          | None                                               | # Database Cleanup: Useful in DEV, but DO NOT run in production                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create PostgreSQL Table**  
   Execute the following DDL in your PostgreSQL database before starting:  
   ```sql
   create table p1gq6ljdsam3x1m."N8Err"
   (
       id         serial primary key,
       created_at timestamp,
       updated_at timestamp,
       created_by varchar,
       updated_by varchar,
       nc_order   numeric,
       title      text,
       "URL"      text,
       "Stack"    text,
       json       json,
       "Message"  text,
       "LastNode" text
   );

   alter table p1gq6ljdsam3x1m."N8Err" owner to postgres;

   create index "N8Err_order_idx" on p1gq6ljdsam3x1m."N8Err" (nc_order);
   ```

2. **Add an `Error Trigger` node**  
   - Type: Error Trigger  
   - No parameters needed. This node will capture any workflow error.

3. **Add a `PostgreSQL` node named `Insert Log`**  
   - Set operation to "Insert"  
   - Select schema: `p1gq6ljdsam3x1m`  
   - Select table: `N8Err`  
   - Map columns with expressions:  
     - `created_at`: `{{$now}}`  
     - `title`: `{{$json["workflow"]["name"]}}`  
     - `URL`: `{{$json["execution"]["url"]}}`  
     - `Stack`: `{{$json["execution"]["error"]["stack"]}}`  
     - `Message`: `{{$json["execution"]["error"]["message"]}}`  
     - `LastNode`: `{{$json["execution"]["lastNodeExecuted"]}}`  
     - `json`: `{{ JSON.stringify($json) }}` (store full JSON context)  
   - Connect `Error Trigger` output to this node.  
   - Use valid PostgreSQL credentials.

4. **Add a `PostgreSQL` node named `Count for 5 minutes`**  
   - Set operation to "Execute Query"  
   - SQL query:  
     ```sql
     SELECT count(*) FROM p1gq6ljdsam3x1m."N8Err" WHERE created_at >= $1;
     ```  
   - Query replacement for `$1`: `{{$now.minus(5, 'minutes').toString()}}` (dynamic timestamp 5 minutes ago)  
   - Connect `Insert Log` output to this node.  
   - Use same PostgreSQL credentials.

5. **Add an `If` node named `If there is no logs in 5 minutes`**  
   - Condition: check if `{{$json.count}} <= 0` (number type)  
   - Connect `Count for 5 minutes` output to this node.

6. **Add a `Code` node named `CleanUp execution. See below if you will prepend this workflow`**  
   - JavaScript code:  
     ```js
     return [];
     ```  
   - Connect `If` node's "true" output to this node.

7. **Add a `NoOp` node named `Insert your error handling logic after this`**  
   - Placeholder node for your custom logic.  
   - Connect `CleanUp execution` output to this node.

8. *(Optional)* **Add notification nodes for alerts (disabled by default):**  
   - `Principal E-Mail` and `Fallback E-Mail`: Configure SMTP credentials, sender and receiver emails, subject and body expressions referencing error details.  
   - `Push mobile notification`: Set up Pushover credentials and message text.  
   - Connect these nodes in sequence with fallback logic as shown.  
   - Disable these nodes unless you want to send notifications.

9. *(Optional)* **Add a `Manual Trigger` node named `Sometimes... just cleanup`**  
   - Connect it to a `PostgreSQL` node named `Truncate Log Database` configured to delete all rows from `N8Err` and restart sequences.  
   - Use carefully; intended for development only.

10. **Connections:**  
    - `Error Trigger` → `Insert Log` → `Count for 5 minutes` → `If there is no logs in 5 minutes`  
    - `If` node (true) → `CleanUp execution` → `Insert your error handling logic after this`  
    - `If` node (false) → no output (ends)  
    - Optional notification nodes connected from a separate `Execute Workflow` node if used.

11. **Credentials:**  
    - PostgreSQL credentials: must be configured with access to the `p1gq6ljdsam3x1m.N8Err` table.  
    - SMTP credentials (if email notifications enabled).  
    - Pushover API credentials (if push notifications enabled).

12. **Final Notes:**  
    - Ensure the workflow is enabled and error triggers are active.  
    - Adjust time interval (5 minutes) in the SQL query as needed.  
    - Insert your own alerting or remediation steps after the cleanup node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Workflow created by Davi Saranszky Mesquita                                                                                                    | https://www.linkedin.com/in/mesquitadavi/                 |
| PostgreSQL table `N8Err` must be created with the exact schema for this workflow to function properly                                           | See DDL in Workflow Overview and Sticky Note node content |
| This workflow is designed primarily for logging and rate-limiting notifications; actual alert sending is left to user customization            | Notification nodes are disabled by default                 |
| Database cleanup node should only be used in development environments to avoid data loss                                                       | Sticky Note2 in workflow                                   |
| The workflow can be integrated as a primary error handler or as a sub-workflow called from other workflows for modular error handling           | Refer to Execute Workflow nodes and notes                  |
| For best integration, adjust SMTP and Pushover credentials and recipient settings according to your environment                                  | SMTP and Pushover credential setup required                |
| Link to workflow video and instructions is not provided but can be requested from author                                                      | Contact via LinkedIn profile                                |

---

This completes the detailed analysis and documentation of the "Error Handling System with PostgreSQL Logging and Rate-Limited Notifications" workflow.