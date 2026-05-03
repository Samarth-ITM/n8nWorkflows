Send n8n automation errors to a Monday.com board

https://n8nworkflows.xyz/workflows/send-n8n-automation-errors-to-a-monday-com-board-2074


# Send n8n automation errors to a Monday.com board

### 1. Workflow Overview

This workflow serves as an error handler specifically designed to capture and log automation errors from any n8n workflow into a Monday.com board. Its main use case is centralized error tracking, which facilitates troubleshooting and monitoring of workflow failures.

The workflow is logically divided into the following blocks:

- **1.1 Error Capture:** Detects and captures errors occurring in other workflows.
- **1.2 Monday.com Item Creation:** Creates a new item in a Monday.com board to log the error occurrence.
- **1.3 Timestamp Generation:** Obtains the current date and time for the error event.
- **1.4 Stack Trace Extraction:** Processes the error stack trace for safe storage.
- **1.5 Monday.com Item Update:** Populates the newly created Monday.com item with detailed error information.

---

### 2. Block-by-Block Analysis

#### 1.1 Error Capture

- **Overview:**  
This block listens for any errors triggered within n8n workflows, serving as the entry point for the error logging process.

- **Nodes Involved:**  
  - `Error Trigger`

- **Node Details:**

  - **Node Name:** Error Trigger  
  - **Type:** `Error Trigger` (system node)  
  - **Role:** Captures errors from any workflow that designates this workflow as its error handler.  
  - **Configuration:** Default settings; no user parameters required.  
  - **Key Expressions/Variables:** Accesses error details via `$('Error Trigger').item.json.execution.error`.  
  - **Inputs:** None (event-driven).  
  - **Outputs:** Connects to `Monday` node.  
  - **Version:** n8n standard error trigger.  
  - **Failure Modes:** No direct failures; however, if not properly assigned as an error workflow in other workflows, errors will not be captured here.  
  - **Sub-workflow:** None.

#### 1.2 Monday.com Item Creation

- **Overview:**  
Creates a new item in the designated Monday.com board to represent the error event, using the error execution ID as the item name.

- **Nodes Involved:**  
  - `Monday`

- **Node Details:**

  - **Node Name:** Monday  
  - **Type:** `Monday.com` node  
  - **Role:** Creates a new board item for the error log.  
  - **Configuration:**  
    - Operation: `create` board item  
    - Board ID: `1382091189` (to be customized)  
    - Group ID: `topics` (to be customized)  
    - Item Name: Expression concatenates empty string with error execution ID, i.e., the unique execution identifier from the error trigger (`={{ "".concat($('Error Trigger').last().json.execution.id) }}`).  
  - **Key Expressions/Variables:** Uses `$['Error Trigger'].last().json.execution.id` for item naming.  
  - **Inputs:** From `Error Trigger`.  
  - **Outputs:** Connects to `Date & Time` node.  
  - **Credentials:** Uses Monday.com credential (OAuth2).  
  - **Failure Modes:** Authentication failures, incorrect board or group IDs, permission issues, or API rate limits.  
  - **Sub-workflow:** None.

#### 1.3 Timestamp Generation

- **Overview:**  
Generates the current date and time in ISO format, which will be logged alongside error details.

- **Nodes Involved:**  
  - `Date & Time`

- **Node Details:**

  - **Node Name:** Date & Time  
  - **Type:** `Date & Time` node  
  - **Role:** Provides current timestamp for error logging.  
  - **Configuration:** Default; no parameters changed. Outputs ISO 8601 date string in `currentDate`.  
  - **Key Expressions/Variables:** Outputs accessible via `$('Date & Time').last().json.currentDate`.  
  - **Inputs:** From `Monday` node (after item creation).  
  - **Outputs:** Connects to `Code` node.  
  - **Failure Modes:** Unlikely to fail unless internal system clock error occurs.  
  - **Sub-workflow:** None.

#### 1.4 Stack Trace Extraction

- **Overview:**  
Extracts and escapes the stack trace string from the error details to prepare it for safe insertion into the Monday.com long-text field.

- **Nodes Involved:**  
  - `Code`

- **Node Details:**

  - **Node Name:** Code  
  - **Type:** `Code` node (JavaScript)  
  - **Role:** Accesses error stack trace, escapes it, and returns it in a property named `stack`.  
  - **Configuration:**  
    - JavaScript code snippet:
      ```javascript
      console.log($('Error Trigger').last().json.execution);
      str = escape($('Error Trigger').last().json.execution.error.stack);
      return { "stack": str };
      ```
  - **Key Expressions/Variables:** Uses `$['Error Trigger'].last().json.execution.error.stack`.  
  - **Inputs:** From `Date & Time` node.  
  - **Outputs:** Connects to `UPDATE` node.  
  - **Failure Modes:** If `error.stack` is undefined or null, `escape()` may fail or produce unexpected output. Logging via `console.log()` is for debug and can be removed in production.  
  - **Sub-workflow:** None.

#### 1.5 Monday.com Item Update

- **Overview:**  
Updates the newly created Monday.com board item with detailed error information: workflow name, escaped stack trace, error message, and timestamp.

- **Nodes Involved:**  
  - `UPDATE`

- **Node Details:**

  - **Node Name:** UPDATE  
  - **Type:** `Monday.com` node  
  - **Role:** Updates multiple columns of the Monday.com item with error details.  
  - **Configuration:**  
    - Operation: `changeMultipleColumnValues` for the item created by the `Monday` node.  
    - Board ID: `1382091189` (same as creation).  
    - Item ID: Derived dynamically from the output of the `Monday` node (`={{ $('Monday').last().json.id }}`).  
    - Column values (as JSON string):  
      - `column_id_for_workflow_name (text)`: workflow name from `Error Trigger`.  
      - `column_id_for_error_stack (long text)`: escaped stack trace from `Code` node.  
      - `column_id_for_error_message (text)`: error message from `Error Trigger`.  
      - `column_id_for_date (text)`: current date from `Date & Time` node.  
    - Note: The field IDs are placeholders and must be replaced with actual Monday board column IDs.  
  - **Key Expressions/Variables:**  
    - Item ID: `$('Monday').last().json.id`  
    - Workflow name: `$('Error Trigger').item.json.workflow.name`  
    - Stack trace: `$('Code').last().json.stack`  
    - Error message: `$('Error Trigger').item.json.execution.error.message`  
    - Date: `$('Date & Time').last().json.currentDate`  
  - **Inputs:** From `Code` node.  
  - **Outputs:** None (end of workflow).  
  - **Credentials:** Monday.com OAuth2 credentials.  
  - **Failure Modes:** Incorrect column IDs, invalid JSON in column values, API limits, authentication failure.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name    | Node Type            | Functional Role                    | Input Node(s)   | Output Node(s) | Sticky Note                                                                                          |
|--------------|----------------------|----------------------------------|-----------------|----------------|----------------------------------------------------------------------------------------------------|
| Error Trigger| Error Trigger        | Captures errors from workflows   | None            | Monday         |                                                                                                    |
| Monday       | Monday.com           | Creates new Monday.com item      | Error Trigger   | Date & Time    | CREATE ERROR ITEM                                                                                   |
| Date & Time  | Date & Time          | Generates current timestamp      | Monday          | Code           |                                                                                                    |
| Code         | Code (JavaScript)    | Extracts and escapes stack trace | Date & Time     | UPDATE         | GET STACKTRACE                                                                                      |
| UPDATE       | Monday.com           | Updates Monday item with details | Code            | None           | POPULUATE MONDAY ITEM <br> Edit column IDs for your board. <br> See [Monday column IDs instructions](https://support.monday.com/hc/en-us/articles/360000225709-Board-item-column-and-automation-or-integration-ID-s) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Error Trigger node**  
   - Type: `Error Trigger`  
   - Purpose: To catch errors from other workflows.  
   - No input connections; event-driven node.

2. **Add a Monday.com node (named `Monday`)**  
   - Type: `Monday.com`  
   - Credentials: Select or create your Monday.com OAuth2 credential.  
   - Operation: Create a new board item (`resource: boardItem`)  
   - Parameters:  
     - Board ID: Enter your Monday.com board ID (e.g., `1382091189`)  
     - Group ID: Enter your board’s group ID (e.g., `topics`)  
     - Name: Use expression `={{ "".concat($('Error Trigger').last().json.execution.id) }}` to name the item with the error execution ID.  
   - Connect the output of `Error Trigger` to this node.

3. **Add a Date & Time node**  
   - Type: `Date & Time`  
   - Configuration: Default, to produce current timestamp.  
   - Connect output of `Monday` node to this node.

4. **Add a Code node (named `Code`)**  
   - Type: `Code` (JavaScript)  
   - Code:  
     ```javascript
     // Optional debug
     console.log($('Error Trigger').last().json.execution);
     // Escape stack trace for safe storage
     str = escape($('Error Trigger').last().json.execution.error.stack);
     return { "stack": str };
     ```  
   - Connect output of `Date & Time` node to this node.

5. **Add a Monday.com node (named `UPDATE`)**  
   - Type: `Monday.com`  
   - Credentials: Use the same Monday.com OAuth2 credential.  
   - Operation: Update multiple column values of an existing board item (`changeMultipleColumnValues`).  
   - Parameters:  
     - Board ID: Same as creation node (`1382091189`)  
     - Item ID: Expression: `={{ $('Monday').last().json.id }}` (ID of the newly created item)  
     - Column Values: JSON string with your actual column IDs replacing placeholders:  
       ```json
       {
         "column_id_for_workflow_name (text)": "{{ $('Error Trigger').item.json.workflow.name }}",
         "column_id_for_error_stack (long text)": "{{ $('Code').last().json.stack }}",
         "column_id_for_error_message (text)": "{{ $('Error Trigger').item.json.execution.error.message }}",
         "column_id_for_date (text)": "{{ $('Date & Time').last().json.currentDate }}"
       }
       ```  
   - Connect output of `Code` node to this node.

6. **Set this workflow as the error workflow for any other workflows you want to monitor**  
   - In those workflows, under settings, select this workflow as the error handler.  
   - Optionally, add “Stop and Error” nodes in workflows to send custom error messages to this log.

7. **Verify Monday board configuration**  
   - Your Monday board must have columns with correct types: text for timestamp and error message, long text for stack trace.  
   - Retrieve column IDs following Monday.com’s instructions: https://support.monday.com/hc/en-us/articles/360000225709-Board-item-column-and-automation-or-integration-ID-s  
   - Replace placeholder column IDs in the UPDATE node accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| To trigger error logging, select this automation as the error workflow on any automation you want to monitor.                   | Workflow setup instructions                                                                                        |
| For detailed logging, insert Stop and Error nodes in workflows to send specific messages to the Monday.com board.               | Best practice for error granularity                                                                                |
| Monday.com column IDs must be retrieved and replaced in the UPDATE node for correct functionality.                              | https://support.monday.com/hc/en-us/articles/360000225709-Board-item-column-and-automation-or-integration-ID-s    |
| Monday.com credentials require OAuth2 setup with appropriate scopes to create and update items on the target board.             | Credential setup                                                                                                   |
| This workflow logs error execution ID as the item name for uniqueness and traceability.                                          | Naming convention                                                                                                  |

---

This structured reference fully describes the workflow logic, configuration, and setup necessary for both human users and automation agents to understand, reproduce, and maintain the error logging integration between n8n and Monday.com.