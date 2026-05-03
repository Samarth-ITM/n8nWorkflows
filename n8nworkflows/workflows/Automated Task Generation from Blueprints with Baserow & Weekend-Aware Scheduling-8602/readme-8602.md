Automated Task Generation from Blueprints with Baserow & Weekend-Aware Scheduling

https://n8nworkflows.xyz/workflows/automated-task-generation-from-blueprints-with-baserow---weekend-aware-scheduling-8602


# Automated Task Generation from Blueprints with Baserow & Weekend-Aware Scheduling

### 1. Workflow Overview

This workflow automates the generation of tasks in Baserow based on predefined procedure templates ("blueprints") and assigns them to users with calculated deadlines. It is designed for recurring processes such as project management, HR onboarding, or operational checklists, allowing efficient batch creation of tasks linked to a master template.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives a webhook POST request with parameters for assignee, template, schedule date, and notes.
- **1.2 Configuration Setup:** Defines API credentials, database, and table IDs required for Baserow integration and passes through input parameters.
- **1.3 Template Steps Retrieval:** Queries the Baserow "Details" table for steps linked to the specified master template.
- **1.4 Deadline Calculation:** Calculates task deadlines by adding days to complete each step to the schedule date.
- **1.5 Weekend Adjustment:** Adjusts deadlines falling on weekends to the next Monday.
- **1.6 Task Aggregation and Batch Creation:** Aggregates task data into a batch payload and sends it via HTTP POST to Baserow’s batch API endpoint.
- **1.7 Response Handling:** Sends success or error responses back to the webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  The webhook node triggers the workflow upon receiving a POST request containing all necessary data to generate tasks.

- **Nodes Involved:**  
  - Trigger task creation  
  - Sticky Note1 (documentation)

- **Node Details:**

  - **Trigger task creation**  
    - Type: Webhook  
    - Role: Entry point listening for POST requests at `/create-tasks-for-template`.  
    - Configuration: HTTP method POST, returns response from downstream nodes.  
    - Key variables: Expects `assignee_id`, `template_id`, `schedule_date`, `note` in JSON body.  
    - Connections: Outputs to "Configure settings and ids".  
    - Failure modes: Missing or malformed request body; webhook not triggered if called incorrectly.  
    - Notes: Sticky Note1 explains request body format and usage example with Baserow app builder.

---

#### 1.2 Configuration Setup

- **Overview:**  
  Sets static and dynamic parameters such as API host, tokens, table IDs, and extracts webhook data into structured variables for use later.

- **Nodes Involved:**  
  - Configure settings and ids  
  - Sticky Note2 (documentation)

- **Node Details:**

  - **Configure settings and ids**  
    - Type: Set  
    - Role: Stores API endpoint, authentication token, database and table IDs, plus assignee and scheduling details from webhook.  
    - Configuration: Hardcoded host URL, token placeholder, numeric IDs for database and tables; dynamic values from webhook for assignee, schedule date (defaults to current date if missing), and note.  
    - Expressions: Uses ternary `$if` to default schedule date; direct JSON references from webhook body.  
    - Connections: Outputs to "Get all template steps".  
    - Edge cases: Missing token or invalid IDs cause API failures downstream.  
    - Notes: Sticky Note2 explains meaning of each field and references token creation documentation.

---

#### 1.3 Template Steps Retrieval

- **Overview:**  
  Queries the Baserow "Details" table to fetch all steps linked to the master template specified in the webhook.

- **Nodes Involved:**  
  - Get all template steps

- **Node Details:**

  - **Get all template steps**  
    - Type: Baserow  
    - Role: Retrieves all records from the "Details" table filtered by linked master template ID.  
    - Configuration: Uses `tableId` and `databaseId` from previous node, filters rows where link to master template matches webhook `template_id`. Returns all matching records.  
    - Connections: Outputs to "Calculate deadlines for each step".  
    - Edge cases: Network/API errors, incorrect table or field IDs, no matching template steps found (empty result).  
    - Credentials: Baserow SaaS account with API token.  
    - Version-specific: Uses field filters available in n8n Baserow node version 1.

---

#### 1.4 Deadline Calculation

- **Overview:**  
  Creates task objects by calculating deadlines per step based on the schedule date plus the step’s “Days to complete” value, and prepares other task properties.

- **Nodes Involved:**  
  - Calculate deadlines for each step

- **Node Details:**

  - **Calculate deadlines for each step**  
    - Type: Set  
    - Role: For each template step, creates fields matching the Tasks table schema: Procedure step ID, calculated deadline, assignee, and note.  
    - Configuration:  
      - Procedure step: array containing step ID  
      - Deadline: adds `Days to complete` to schedule date using JavaScript date logic and date-fns `.plus()` method  
      - Assignee: array with assignee ID from webhook  
      - Note: from webhook note field  
    - Expressions: Uses date arithmetic and JSON references from previous node and "Configure settings and ids".  
    - Connections: Outputs to "Avoid scheduling during the weekend".  
    - Edge cases: Missing or invalid `Days to complete` field, date parsing errors.  
    - Notes: Adjust field names here if your Tasks table schema differs.

---

#### 1.5 Weekend Adjustment

- **Overview:**  
  Adjusts calculated deadlines that fall on Saturday or Sunday to the following Monday to avoid weekend scheduling.

- **Nodes Involved:**  
  - Avoid scheduling during the weekend

- **Node Details:**

  - **Avoid scheduling during the weekend**  
    - Type: Code (JavaScript)  
    - Role: Iterates over task deadlines, detects weekend days (Sat=6, Sun=0), adds days to shift deadline to Monday.  
    - Configuration: Custom JS code using `Date` objects and ISO string formatting.  
    - Inputs: Takes all tasks from previous node.  
    - Outputs: Modified list with corrected deadlines.  
    - Edge cases: Timezone differences could affect date calculations; non-date strings cause errors.  
    - Version: Requires n8n v0.154+ for JS code node with ES6+ support.

---

#### 1.6 Task Aggregation and Batch Creation

- **Overview:**  
  Aggregates all tasks into a single array payload and performs a batch HTTP POST to create all tasks in the Baserow Tasks table in one request.

- **Nodes Involved:**  
  - Aggregate tasks for insert  
  - Generate tasks in batch

- **Node Details:**

  - **Aggregate tasks for insert**  
    - Type: Aggregate  
    - Role: Collects all individual task items into an array under the `items` field, preparing for batch insertion.  
    - Configuration: Uses aggregate all item data option.  
    - Connections: Outputs to "Generate tasks in batch".  
    - Edge cases: Empty input array results in empty batch request.

  - **Generate tasks in batch**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Baserow API batch endpoint to insert multiple rows at once.  
    - Configuration:  
      - URL dynamically constructed from API host and Tasks table ID from settings node.  
      - Authorization header with token.  
      - JSON body contains aggregated `items` array.  
      - On error: continues with error output path.  
    - Inputs: Aggregated tasks from previous node.  
    - Outputs: On success, routes to "Success response" node; on error, routes to "Error response" node.  
    - Edge cases: HTTP errors, auth failures, invalid payload, API rate limits.

---

#### 1.7 Response Handling

- **Overview:**  
  Returns a simple text response to the webhook caller indicating success or failure of task creation.

- **Nodes Involved:**  
  - Success response  
  - Error response

- **Node Details:**

  - **Success response**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 response with success message text.  
    - Configuration: "✅ The tasks were created" message.  
    - Inputs: Success output from "Generate tasks in batch".  
    - Edge cases: Response failure if webhook connection lost.

  - **Error response**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP 200 response with failure message text (workflow continues on error).  
    - Configuration: "❌ Something went wrong, please check the log files" message.  
    - Inputs: Error output from "Generate tasks in batch".  
    - Edge cases: Same as above.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                                      | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                       |
|-----------------------------|-------------------------|-----------------------------------------------------|-----------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Trigger task creation        | Webhook                 | Entry point receiving POST request with task params |                             | Configure settings and ids     | Explains required JSON body for webhook trigger with example and screenshot                      |
| Configure settings and ids   | Set                     | Stores API credentials, IDs, and webhook data       | Trigger task creation        | Get all template steps         | Details about each configured field and links to token documentation                            |
| Get all template steps       | Baserow                 | Fetches template steps linked to master template    | Configure settings and ids   | Calculate deadlines for each step |                                                                                                 |
| Calculate deadlines for each step | Set                | Calculates deadlines and prepares task properties   | Get all template steps       | Avoid scheduling during the weekend | Advises to adjust property names to fit your Tasks table schema                                |
| Avoid scheduling during the weekend | Code              | Adjusts deadlines falling on weekend to next Monday | Calculate deadlines for each step | Aggregate tasks for insert  |                                                                                                 |
| Aggregate tasks for insert   | Aggregate               | Aggregates tasks into batch payload for API         | Avoid scheduling during the weekend | Generate tasks in batch     |                                                                                                 |
| Generate tasks in batch      | HTTP Request            | Batch creates tasks in Baserow via API               | Aggregate tasks for insert   | Success response, Error response | Uses HTTP POST to Baserow batch API; continues on error to send failure response                |
| Success response            | Respond to Webhook       | Sends success response text                          | Generate tasks in batch (success) |                            |                                                                                                 |
| Error response              | Respond to Webhook       | Sends error response text                            | Generate tasks in batch (error) |                            |                                                                                                 |
| Sticky Note                 | Sticky Note             | Documentation and instructions                       |                             |                                | Comprehensive overview of workflow purpose, use cases, instructions, and customization options |
| Sticky Note1                | Sticky Note             | Documentation for webhook input format               |                             |                                | Details on triggering the webhook and required JSON body                                       |
| Sticky Note2                | Sticky Note             | Documentation for settings node                       |                             |                                | Explains configuration of API host, tokens, and IDs                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: "Trigger task creation"  
   - HTTP Method: POST  
   - Path: `create-tasks-for-template`  
   - Response Mode: Response Node (will send response from downstream)  

2. **Create Set Node**  
   - Name: "Configure settings and ids"  
   - Add these fields:  
     - API host: `https://api.baserow.io`  
     - Token: your Baserow API token (string)  
     - Database ID: number (your Baserow database)  
     - Detail template table ID: number (table containing template steps)  
     - Link to master template field ID: string (field linking to master table)  
     - Tasks table ID: number (table where tasks are stored)  
     - Assignee ID: expression `={{ $json.body.assignee_id }}` (from webhook)  
     - Schedule date: expression `={{ $if(typeof $json.body.schedule_date == 'undefined',$now.format('yyyy-MM-dd'), $json.body.schedule_date)}}`  
     - Note: expression `={{ $json.body.note }}`  

   - Connect webhook output to this node.

3. **Create Baserow Node**  
   - Name: "Get all template steps"  
   - Credentials: Select Baserow SaaS account with API token  
   - Database ID: `={{ $json["Database ID"] }}`  
   - Table ID: `={{ $json["Detail template table ID"] }}`  
   - Return all rows: true  
   - Add filter to fetch rows where field (link to master template) equals webhook `template_id`:  
     - Field: `={{ $json["Link to master template field ID"] }}`  
     - Operator: `link_row_has`  
     - Value: `={{ $('Trigger task creation').item.json.body.template_id }}`  
   - Connect previous node output to this.

4. **Create Set Node**  
   - Name: "Calculate deadlines for each step"  
   - For each incoming item, assign:  
     - Procedure step (array): `[{{ $json.id }}]`  
     - Deadline (string): calculate by adding `Days to complete` to `Schedule date` using expression:  
       `={{new Date($('Configure settings and ids').item.json['Schedule date']).plus($json['Days to complete'],'days').format('yyyy-MM-dd')}}`  
     - Assignee (array): `[{{ $('Configure settings and ids').item.json['Assignee ID'] }}]`  
     - Note (string): `={{ $('Configure settings and ids').item.json.Note }}`  
   - Connect Baserow node output here.

5. **Create Code Node**  
   - Name: "Avoid scheduling during the weekend"  
   - Paste the provided JavaScript code to adjust deadlines falling on Saturday or Sunday to next Monday.  
   - Connect previous node output here.

6. **Create Aggregate Node**  
   - Name: "Aggregate tasks for insert"  
   - Aggregate mode: aggregate all item data to a single property named `items`.  
   - Connect code node output here.

7. **Create HTTP Request Node**  
   - Name: "Generate tasks in batch"  
   - HTTP Method: POST  
   - URL: `={{ $('Configure settings and ids').item.json['API host'] }}/api/database/rows/table/{{ $('Configure settings and ids').item.json['Tasks table ID'] }}/batch/?user_field_names=true`  
   - Authentication: None (token passed in header)  
   - Headers:  
     - Authorization: `Token {{ $('Configure settings and ids').item.json.Token }}`  
   - Body Content Type: JSON  
   - Body: `={"items": {{ JSON.stringify($json.items) }}}`  
   - On error: continue with error output  
   - Connect aggregate node output here.

8. **Create Respond to Webhook Nodes**  
   - Name one: "Success response"  
     - Respond with: Text  
     - Response body: "✅ The tasks were created"  
     - Connect success output of HTTP Request here.  
   - Name another: "Error response"  
     - Respond with: Text  
     - Response body: "❌ Something went wrong, please check the log files"  
     - Connect error output of HTTP Request here.

9. **Add Sticky Notes** (optional for documentation)  
   - Add a sticky note summarizing the workflow purpose, use cases, and instructions.  
   - Add sticky notes describing webhook input format and configuration settings as per the original workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow automates task creation in Baserow based on templates, suitable for recurring processes like project management and onboarding. | See sticky note at workflow start for detailed overview. |
| Baserow Standard Operating Procedures template is recommended as a base schema to try this workflow. | https://baserow.io/templates/standard-operating-procedures |
| Authentication requires a personal API token from Baserow. | https://baserow.io/user-docs/personal-api-tokens |
| Batch creation of tasks uses the Baserow API endpoint documented here. | https://api.baserow.io/api/redoc/#tag/Database-table-rows/operation/batch_create_database_table_rows |
| Scheduling avoids weekends by shifting deadlines to Monday; can be extended to consider public holidays with external calendar APIs. | Customization suggestion in sticky note. |
| The Baserow application builder supports triggering this webhook using "Send an HTTP request" action from buttons or events. | Example screenshot in sticky note1. |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, respecting all content policies and containing no illegal or protected information. All manipulated data is legal and public.