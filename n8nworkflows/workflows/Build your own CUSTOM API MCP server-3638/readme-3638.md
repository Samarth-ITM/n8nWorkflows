Build your own CUSTOM API MCP server

https://n8nworkflows.xyz/workflows/build-your-own-custom-api-mcp-server-3638


# Build your own CUSTOM API MCP server

### 1. Workflow Overview

This workflow demonstrates how an organization can build a custom MCP (Machine Conversational Platform) server using n8n to interact with the PayCaptain payroll API. It enables MCP clients to search for employee data, retrieve employee details by ID, and update employee records securely. The workflow includes logic to sanitize sensitive data, restrict updates to allowed fields, and log all operations for audit purposes.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger & Input Reception:** Listens for incoming MCP client requests and extracts operation parameters.
- **1.2 Operation Routing:** Routes requests to one of three custom tools based on the requested operation: Search Employee, Get Employee by ID, or Update Employee.
- **1.3 Search Employee Tool:** Queries the PayCaptain API to search employees matching a query string, filters and sanitizes results.
- **1.4 Get Employee by ID Tool:** Retrieves employee details by ID from PayCaptain API, filters and sanitizes the response.
- **1.5 Update Employee Tool:** Validates and filters update requests, restricts updates to allowed fields, sends update requests to PayCaptain API, and returns success or error responses.
- **1.6 Logging:** Logs all incoming requests and operations to a Google Sheet for audit and traceability.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger & Input Reception

- **Overview:**  
  This block receives incoming MCP client requests via the MCP Server Trigger node and extracts the input parameters: operation type, query string, employee ID, and values for updates.

- **Nodes Involved:**  
  - Paycaptain MCP Server (MCP Trigger)  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - Log Call (Google Sheets)

- **Node Details:**

  - **Paycaptain MCP Server**  
    - Type: MCP Trigger (Langchain integration)  
    - Role: Entry point for MCP client requests via a webhook path.  
    - Configuration: Webhook path set to a unique ID.  
    - Inputs: MCP client requests with JSON payloads.  
    - Outputs: Passes request data downstream.  
    - Edge Cases: Authentication should be enabled before production to prevent unauthorized access.  
    - Notes: See [MCP Server Trigger docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger).

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Receives inputs from the MCP Server Trigger and exposes parameters for routing.  
    - Configuration: Defines inputs `operation`, `query`, `employeeId`, and `values` as workflow inputs.  
    - Inputs: Triggered by MCP Server node.  
    - Outputs: Provides structured input data for routing.  
    - Edge Cases: Input validation is implicit; malformed inputs could cause routing errors.

  - **Log Call**  
    - Type: Google Sheets node  
    - Role: Appends a log entry for each MCP request with timestamp, operation, query, employeeId, and values.  
    - Configuration: Appends to a specific Google Sheet document and sheet tab (Sheet1).  
    - Inputs: Receives data from `When Executed by Another Workflow`.  
    - Outputs: None (terminal node for logging).  
    - Edge Cases: Google Sheets API rate limits or connectivity issues may delay logging.

---

#### 2.2 Operation Routing

- **Overview:**  
  Routes the incoming request to the appropriate tool workflow based on the `operation` parameter: `searchEmployees`, `getEmployeeById`, or `updateEmployee`.

- **Nodes Involved:**  
  - Operation (Switch)

- **Node Details:**

  - **Operation**  
    - Type: Switch node  
    - Role: Routes workflow execution based on the `operation` input value.  
    - Configuration:  
      - Routes to `searchEmployee` if operation equals `searchEmployees`.  
      - Routes to `getEmployeeById` if operation equals `getEmployeeById`.  
      - Routes to `updateEmployee` if operation equals `updateEmployee`.  
    - Inputs: Receives data from `Log Call`.  
    - Outputs: Connects to respective tool workflows.  
    - Edge Cases: Unknown or misspelled operations will not match any route, causing no downstream execution.

---

#### 2.3 Search Employee Tool

- **Overview:**  
  Searches employees via the PayCaptain API using a query string, filters matching employees, sanitizes sensitive fields, and aggregates results.

- **Nodes Involved:**  
  - Search Employees (Tool Workflow)  
  - Get Employees (HTTP Request)  
  - Filter Matches (Filter)  
  - Strip Sensitive Fields (Set)  
  - Aggregate Search Results (Aggregate)

- **Node Details:**

  - **Search Employees**  
    - Type: Tool Workflow (Langchain)  
    - Role: Encapsulates the search employee logic as a reusable tool.  
    - Configuration: Passes `query` and `operation` inputs, no employeeId or values.  
    - Inputs: Routed from `Operation` switch.  
    - Outputs: Returns filtered and sanitized employee data.  
    - Edge Cases: Empty or invalid query strings may return no results.

  - **Get Employees**  
    - Type: HTTP Request  
    - Role: Calls PayCaptain API endpoint `/employees` with pagination to retrieve employee lists.  
    - Configuration:  
      - URL: `https://api.paycaptain.com/employees`  
      - Authentication: HTTP Header Auth with JWT token credential.  
      - Query Parameters: `company=paycaptain`, `page` for pagination.  
      - Pagination: Limits to 3 pages with 1-second interval.  
    - Inputs: Triggered by `Operation` node.  
    - Outputs: List of employees JSON.  
    - Edge Cases: API rate limits, invalid credentials, or network errors.

  - **Filter Matches**  
    - Type: Filter  
    - Role: Filters employees whose concatenated searchable fields contain the query string (case-insensitive).  
    - Configuration:  
      - Concatenates fields: hrEmployeeId, payrollCode, full name, email, NI number, city, job title, grade, department, team.  
      - Compares lowercase concatenated string to lowercase query.  
    - Inputs: From `Get Employees`.  
    - Outputs: Matching employees only.  
    - Edge Cases: No matches found results in empty output.

  - **Strip Sensitive Fields**  
    - Type: Set  
    - Role: Removes sensitive or unnecessary fields from employee data before returning.  
    - Configuration: Explicitly sets allowed fields such as company, hrEmployeeId, payrollCode, name, email, phone, mailing address, location, department, team, job grade, job title, contract type, weekly hours, days worked.  
    - Inputs: From `Filter Matches`.  
    - Outputs: Sanitized employee records.  
    - Edge Cases: Missing fields in source data result in undefined values.

  - **Aggregate Search Results**  
    - Type: Aggregate  
    - Role: Aggregates all filtered employee items into a single response object under `response` field.  
    - Inputs: From `Strip Sensitive Fields`.  
    - Outputs: Aggregated response for MCP client.  
    - Edge Cases: Empty input results in empty aggregated response.

---

#### 2.4 Get Employee by ID Tool

- **Overview:**  
  Retrieves a single employee's details by their employee ID, sanitizes the data, and aggregates the response.

- **Nodes Involved:**  
  - Get Employee (Tool Workflow)  
  - Get Employees1 (HTTP Request)  
  - Filter Matching ID (Filter)  
  - Strip Sensitive Fields1 (Set)  
  - Aggregate Get Response (Aggregate)

- **Node Details:**

  - **Get Employee**  
    - Type: Tool Workflow (Langchain)  
    - Role: Encapsulates logic to get employee details by ID.  
    - Configuration: Passes `employeeId` and `operation` inputs, no query or values.  
    - Inputs: Routed from `Operation` switch.  
    - Outputs: Sanitized employee data for the requested ID.  
    - Edge Cases: Invalid or missing employeeId returns no results.

  - **Get Employees1**  
    - Type: HTTP Request  
    - Role: Calls PayCaptain API `/employees` endpoint with pagination to retrieve employee lists (same as `Get Employees`).  
    - Configuration: Same as `Get Employees`.  
    - Inputs: From `Operation` node.  
    - Outputs: List of employees JSON.  
    - Edge Cases: Same as `Get Employees`.

  - **Filter Matching ID**  
    - Type: Filter  
    - Role: Filters employees to find the one matching the requested `employeeId`.  
    - Configuration: Compares `hrEmployeeId` field to input `employeeId`.  
    - Inputs: From `Get Employees1`.  
    - Outputs: Single matching employee or empty.  
    - Edge Cases: No matching employee found results in empty output.

  - **Strip Sensitive Fields1**  
    - Type: Set  
    - Role: Sanitizes employee data by selecting allowed fields (same fields as in Search Employee).  
    - Inputs: From `Filter Matching ID`.  
    - Outputs: Sanitized employee record.  
    - Edge Cases: Missing fields handled as undefined.

  - **Aggregate Get Response**  
    - Type: Aggregate  
    - Role: Aggregates the single employee record into a response object under `response`.  
    - Inputs: From `Strip Sensitive Fields1`.  
    - Outputs: Aggregated response for MCP client.  
    - Edge Cases: Empty input results in empty response.

---

#### 2.5 Update Employee Tool

- **Overview:**  
  Validates update requests to ensure only allowed fields are modified, sends update requests to PayCaptain API, and returns success or error messages accordingly.

- **Nodes Involved:**  
  - Update Employee (Tool Workflow)  
  - Valid Fields Only (Set)  
  - Has Valid Request? (If)  
  - Update Employee1 (HTTP Request)  
  - Get Success Response (Set)  
  - Get Error Response (Set)

- **Node Details:**

  - **Update Employee**  
    - Type: Tool Workflow (Langchain)  
    - Role: Encapsulates update employee logic.  
    - Configuration: Passes `employeeId`, `values`, and `operation` inputs.  
    - Inputs: Routed from `Operation` switch.  
    - Outputs: Success or error response.  
    - Edge Cases: Missing or invalid `employeeId` or `values` cause errors.

  - **Valid Fields Only**  
    - Type: Set  
    - Role: Filters the `values` object to only include editable fields.  
    - Configuration: Allowed fields include firstname, middlename, lastname, mailing address fields, email, phone, NI number, location, department, team, job grade, job title.  
    - Inputs: From `Operation` node.  
    - Outputs: Filtered `values` object.  
    - Edge Cases: If no valid fields are present, results in empty `values`.

  - **Has Valid Request?**  
    - Type: If  
    - Role: Checks if filtered `values` is not empty to proceed with update.  
    - Configuration: Condition checks if `values` object is not empty.  
    - Inputs: From `Valid Fields Only`.  
    - Outputs:  
      - True: Proceed to update API call.  
      - False: Return error response.  
    - Edge Cases: Empty update requests are rejected.

  - **Update Employee1**  
    - Type: HTTP Request  
    - Role: Sends POST request to PayCaptain API `/employee` endpoint to update employee data.  
    - Configuration:  
      - URL: `https://api.paycaptain.com/employee`  
      - Method: POST  
      - Body: JSON containing `hrEmployeeId` and filtered `values`.  
      - Authentication: HTTP Header Auth with JWT token.  
    - Inputs: From `Has Valid Request?` (true branch).  
    - Outputs: API response.  
    - Edge Cases: API errors, invalid data, or auth failures.

  - **Get Success Response**  
    - Type: Set  
    - Role: Sets a simple `"ok"` response on successful update.  
    - Inputs: From `Update Employee1`.  
    - Outputs: Success message to MCP client.

  - **Get Error Response**  
    - Type: Set  
    - Role: Returns an error message listing allowed editable fields when update request is invalid.  
    - Inputs: From `Has Valid Request?` (false branch).  
    - Outputs: Error message to MCP client.

---

#### 2.6 Logging

- **Overview:**  
  Logs all MCP requests with relevant details to a Google Sheet for audit and traceability.

- **Nodes Involved:**  
  - Log Call (Google Sheets)

- **Node Details:**

  - **Log Call**  
    - Type: Google Sheets node  
    - Role: Appends a row with timestamp, operation, query, employeeId, and values (stringified) to a Google Sheet.  
    - Configuration:  
      - Document ID and Sheet name specified.  
      - Append operation enabled.  
      - Uses OAuth2 credentials for Google Sheets.  
    - Inputs: From `When Executed by Another Workflow`.  
    - Outputs: None.  
    - Edge Cases: Google API limits or connectivity issues may delay logging.

---

### 3. Summary Table

| Node Name                  | Node Type                     | Functional Role                              | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                                       |
|----------------------------|-------------------------------|----------------------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Paycaptain MCP Server      | MCP Trigger                   | Entry point for MCP client requests          |                              | When Executed by Another Workflow | ## 1. Set up an MCP Server Trigger [Read more about the MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| When Executed by Another Workflow | Execute Workflow Trigger       | Receives and exposes input parameters        | Paycaptain MCP Server         | Log Call                      |                                                                                                                                  |
| Log Call                   | Google Sheets                 | Logs all MCP requests for audit               | When Executed by Another Workflow | Operation                    |                                                                                                                                  |
| Operation                  | Switch                       | Routes requests based on operation type       | Log Call                     | Get Employees, Get Employees1, Valid Fields Only |                                                                                                                                  |
| Search Employees           | Tool Workflow (Langchain)    | Searches employees by query                    | Operation                    | Paycaptain MCP Server         | ## 2. Build Your MCP Server from Existing APIs [Read more about the HTTP Request Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Get Employees              | HTTP Request                 | Retrieves employee list from PayCaptain API   | Operation                    | Filter Matches                |                                                                                                                                  |
| Filter Matches             | Filter                       | Filters employees matching query               | Get Employees                | Strip Sensitive Fields        |                                                                                                                                  |
| Strip Sensitive Fields     | Set                          | Sanitizes employee data                         | Filter Matches               | Aggregate Search Results      |                                                                                                                                  |
| Aggregate Search Results   | Aggregate                    | Aggregates search results into response        | Strip Sensitive Fields       |                              |                                                                                                                                  |
| Get Employee               | Tool Workflow (Langchain)    | Retrieves employee details by ID                | Operation                    | Paycaptain MCP Server         |                                                                                                                                  |
| Get Employees1             | HTTP Request                 | Retrieves employee list (for ID filtering)     | Operation                    | Filter Matching ID            |                                                                                                                                  |
| Filter Matching ID         | Filter                       | Filters employee matching requested ID         | Get Employees1               | Strip Sensitive Fields1       |                                                                                                                                  |
| Strip Sensitive Fields1    | Set                          | Sanitizes employee data                         | Filter Matching ID           | Aggregate Get Response        |                                                                                                                                  |
| Aggregate Get Response     | Aggregate                    | Aggregates single employee data into response  | Strip Sensitive Fields1      |                              |                                                                                                                                  |
| Update Employee            | Tool Workflow (Langchain)    | Updates employee details                         | Operation                    | Paycaptain MCP Server         |                                                                                                                                  |
| Valid Fields Only          | Set                          | Filters update values to allowed fields         | Operation                    | Has Valid Request?            |                                                                                                                                  |
| Has Valid Request?         | If                           | Checks if update request has valid fields       | Valid Fields Only            | Update Employee1, Get Error Response |                                                                                                                                  |
| Update Employee1           | HTTP Request                 | Sends update request to PayCaptain API          | Has Valid Request? (true)    | Get Success Response          |                                                                                                                                  |
| Get Success Response       | Set                          | Returns success message                          | Update Employee1             |                              |                                                                                                                                  |
| Get Error Response         | Set                          | Returns error message for invalid update fields | Has Valid Request? (false)   |                              |                                                                                                                                  |
| Sticky Note                | Sticky Note                  | Instructional note on MCP Server Trigger setup |                              |                               | ## 1. Set up an MCP Server Trigger [Read more about the MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| Sticky Note1               | Sticky Note                  | Explanation of building MCP server from APIs   |                              |                               | ## 2. Build Your MCP Server from Existing APIs [Read more about the HTTP Request Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) |
| Sticky Note2               | Sticky Note                  | PayCaptain branding and links                    |                              |                               | ![](https://cdn.prod.website-files.com/6628137b611845ebff95551f/6628137b611845ebff9556e7_paycaptain-logo.svg#full-width) **Website**: https://paycaptain.com **DeveloperHub**: https://developer.paycaptain.com **Good to know:** PayCaptain also sponsors the n8n London Meetups - Definitely check them out! |
| Sticky Note3               | Sticky Note                  | Security reminder to always authenticate MCP server |                              |                               | ### Always Authenticate Your Server! Before going to production, it's always advised to enable authentication on your MCP server trigger. |
| Sticky Note4               | Sticky Note                  | Full workflow description, usage, requirements, and customization tips |                              |                               | ## Try It Out! This n8n demonstrates how any organisation can quickly and easily build and offer MCP servers to their customers or internal staff to improve productivity. This MCP example uses PayCaptain.com as an example and shows how to create an MCP server which can search for and update employee data. ... (full content as in description) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Add an MCP Trigger node named `Paycaptain MCP Server`.  
   - Set the webhook path to a unique identifier (e.g., `5f6728df-d3e8-48bb-9a38-0f2e54c7962c`).  
   - Enable authentication before production.  
   - This node listens for MCP client requests.

2. **Add Execute Workflow Trigger Node**  
   - Add `When Executed by Another Workflow` node.  
   - Configure workflow inputs: `operation` (string), `query` (string), `employeeId` (string), `values` (object).  
   - Connect `Paycaptain MCP Server` output to this node.

3. **Add Google Sheets Node for Logging**  
   - Add `Log Call` node.  
   - Configure to append rows to a Google Sheet document (ID: `1Ls_3pmzIafl1NUAzzflkJgyq1smPW6vfGjbVuVzdkac`, sheet `Sheet1`).  
   - Map columns: `timestamp` (current ISO timestamp), `operation`, `query`, `employeeId`, `values` (stringified).  
   - Use Google Sheets OAuth2 credentials.  
   - Connect `When Executed by Another Workflow` to `Log Call`.

4. **Add Switch Node for Operation Routing**  
   - Add `Operation` switch node.  
   - Configure three outputs with conditions on `operation` input:  
     - `searchEmployee` if `operation` equals `searchEmployees`  
     - `getEmployeeById` if `operation` equals `getEmployeeById`  
     - `updateEmployee` if `operation` equals `updateEmployee`  
   - Connect `Log Call` to `Operation`.

5. **Build Search Employee Tool Workflow**  
   - Create a new workflow named `searchEmployees`.  
   - Inputs: `operation`, `query`, `employeeId`, `values`.  
   - Add HTTP Request node `Get Employees`:  
     - URL: `https://api.paycaptain.com/employees`  
     - Authentication: HTTP Header Auth with JWT token credential.  
     - Query params: `company=paycaptain`, `page` for pagination.  
     - Enable pagination with max 3 pages, 1-second interval.  
   - Add Filter node `Filter Matches`: filter employees whose concatenated searchable fields contain the lowercase query string.  
   - Add Set node `Strip Sensitive Fields`: keep only allowed fields (company, hrEmployeeId, payrollCode, firstName, lastName, email, phone, mailing address fields, location, department, team, jobGrade, jobTitle, jobEffectiveDate, contractType, normalWeeklyHours, daysWorkedPerWeek).  
   - Add Aggregate node `Aggregate Search Results`: aggregate all filtered employees into a single `response` field.  
   - Return aggregated response.

6. **Build Get Employee by ID Tool Workflow**  
   - Create a new workflow named `getEmployeeById`.  
   - Inputs: `operation`, `query`, `employeeId`, `values`.  
   - Add HTTP Request node `Get Employees1` (same config as `Get Employees`).  
   - Add Filter node `Filter Matching ID`: filter employees where `hrEmployeeId` equals input `employeeId`.  
   - Add Set node `Strip Sensitive Fields1`: same allowed fields as in Search Employee.  
   - Add Aggregate node `Aggregate Get Response`: aggregate single employee into `response`.  
   - Return aggregated response.

7. **Build Update Employee Tool Workflow**  
   - Create a new workflow named `updateEmployee`.  
   - Inputs: `operation`, `query`, `employeeId`, `values`.  
   - Add Set node `Valid Fields Only`: filter `values` object to only allowed editable fields: firstname, middlename, lastname, mailingStreet, mailingCity, mailingStateProvince, mailingPostalCode, mailingCountry, email, phone, niNumber, location, department, team, jobGrade, jobTitle.  
   - Add If node `Has Valid Request?`: check if filtered `values` is not empty.  
   - True branch:  
     - Add HTTP Request node `Update Employee1`:  
       - URL: `https://api.paycaptain.com/employee`  
       - Method: POST  
       - Body: JSON with `hrEmployeeId` and filtered `values`.  
       - Authentication: HTTP Header Auth with JWT token.  
     - Add Set node `Get Success Response`: set `response` to `"ok"`.  
   - False branch:  
     - Add Set node `Get Error Response`: set `response` to error message listing allowed editable fields.  
   - Return success or error response accordingly.

8. **Connect Tool Workflows to Main Workflow**  
   - Connect `Operation` switch outputs to respective tool workflows using Tool Workflow nodes:  
     - `searchEmployee` → `Search Employees` tool workflow node  
     - `getEmployeeById` → `Get Employee` tool workflow node  
     - `updateEmployee` → `Update Employee` tool workflow node

9. **Credential Setup**  
   - Create HTTP Header Auth credential with JWT token for PayCaptain API.  
   - Create Google Sheets OAuth2 credential for logging.  
   - Assign credentials to respective nodes.

10. **Testing and Deployment**  
    - Test each operation with sample MCP client queries.  
    - Enable authentication on MCP Server Trigger before production.  
    - Optionally customize employee attributes or logging destination.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow uses PayCaptain.com as an example payroll API for employee data management.                                                                                                                                                                                                                                                                                      | https://paycaptain.com                                                                                            |
| PayCaptain Developer Hub provides API documentation and developer resources.                                                                                                                                                                                                                                                                                                  | https://developer.paycaptain.com                                                                                  |
| MCP Server Trigger documentation explains how to integrate MCP clients like Claude Desktop with n8n MCP servers.                                                                                                                                                                                                                                                               | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop |
| MCP clients such as Claude Desktop can connect to this MCP server to perform employee queries and updates.                                                                                                                                                                                                                                                                     | https://claude.ai/download                                                                                        |
| Always enable authentication on MCP Server Trigger nodes before sharing or deploying to production to secure access.                                                                                                                                                                                                                                                          | Security best practice                                                                                            |
| Google Sheets logging adds auditability but may introduce latency; consider faster logging alternatives if needed.                                                                                                                                                                                                                                                             | Performance consideration                                                                                         |
| PayCaptain sponsors n8n London Meetups, a good community resource for n8n users.                                                                                                                                                                                                                                                                                                | https://paycaptain.com and n8n community events                                                                  |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and customize the MCP server workflow for PayCaptain employee management, while anticipating potential errors and integration considerations.