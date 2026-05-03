n8n Workflow Manager API

https://n8nworkflows.xyz/workflows/n8n-workflow-manager-api-4166


# n8n Workflow Manager API

---
### 1. Workflow Overview

This workflow, titled **n8n Workflow Manager API**, serves as a flexible API endpoint to manage and interact with n8n workflows remotely via HTTP requests. It supports both **GET** and **POST** methods on a single webhook path (`workflow-manager`), authenticated via header authentication.

**Use Cases:**
- **POST requests** to dynamically execute any target workflow by ID with custom input data.
- **GET requests** to fetch metadata or full details about workflows, filtered by status or specific workflow ID.
- It is designed to act as a centralized API gateway to trigger or inspect n8n workflows programmatically, suitable for integrations such as custom apps or extensions (e.g., Raycast).

The logic is organized into two major blocks, with smaller routing and error handling components:

- **1.1 Webhook Input & Authentication:** Reception and initial routing of HTTP requests with header-based security.
- **1.2 POST Route - Workflow Execution:** Extracts workflow ID and input data from the request, executes the specified workflow, and returns success or error responses.
- **1.3 GET Route - Workflow Info Retrieval:** Handles requests for workflow metadata or full JSON details with filtering by workflow status or ID using the n8n API.
- **1.4 Common Error Handling:** Responds with appropriate JSON messages for different failure or validation scenarios.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Input & Authentication

**Overview:**  
Defines the API entry point that supports multiple HTTP methods (`GET` and `POST`) and requires header-based authentication. It branches requests into POST or GET flows.

**Nodes Involved:**  
- Webhook  
- map parameters  
- Map webhook request to fields  

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point for HTTP requests on path `workflow-manager`  
  - Config: Accepts all HTTP methods, uses header authentication with credentials named "Header Auth account". Response mode is delegated to Respond to Webhook nodes downstream.  
  - Inputs: HTTP request  
  - Outputs: Two outputs: one for GET (goes to `map parameters`), one for POST (goes to `Map webhook request to fields`)  
  - Edge Cases: Authentication failures, invalid HTTP methods, malformed requests.

- **map parameters**  
  - Type: Set  
  - Role: Extracts query parameters `mode`, `workflowId`, and `includedWorkflows` from the GET request.  
  - Key Expressions:  
    - `mode = {{$json.query.mode}}`  
    - `workflowId = {{$json.query.workflowId}}`  
    - `includedWorkflows = {{$json.query.includedWorkflows}}`  
  - Connections: Output to `Does workflowId Exist` node for conditional routing.  
  - Edge Cases: Missing or invalid query parameters.

- **Map webhook request to fields**  
  - Type: Set  
  - Role: Extracts POST request data: `targetWorkflowId` from query param `workflowId` and full request body as `workflowInputData`.  
  - Key Expressions:  
    - `targetWorkflowId = {{$json.query.workflowId}}`  
    - `workflowInputData = {{$json.body}}`  
  - Outputs: To `Execute Workflow` node for dynamic execution.  
  - Notes: Requires corresponding logic in the target workflow to handle `workflowInputData`.  
  - Edge Cases: Missing workflowId, malformed body.

---

#### 2.2 POST Route - Workflow Execution

**Overview:**  
Executes a target workflow specified by `workflowId` from the POST request, passing the request body as input data. Waits for execution completion before responding with success or failure.

**Nodes Involved:**  
- Execute Workflow  
- return succes message  
- return problem executing workflow  
- Return problem handling request  

**Node Details:**

- **Execute Workflow**  
  - Type: Execute Workflow (sub-workflow)  
  - Role: Triggers the workflow specified by `targetWorkflowId` extracted earlier.  
  - Config:  
    - Workflow ID dynamically set via expression from previous node.  
    - Waits for sub-workflow completion before proceeding.  
    - Passes workflow input data (currently empty object due to config, but logically should use input from `workflowInputData` field; this may need adjustment).  
  - Outputs:  
    - Success: to `return succes message` node.  
    - Error: to `return problem executing workflow` node.  
  - Edge Cases: Invalid workflow ID, workflow execution errors, timeout, input data mismatch.

- **return succes message**  
  - Type: Respond to Webhook  
  - Role: Returns the entire execution output to the caller on successful workflow execution.  
  - Output: HTTP response with all incoming items.

- **return problem executing workflow**  
  - Type: Respond to Webhook  
  - Role: Returns a JSON error message `{"status": "problem_executing_workflow"}` on errors from execution.

- **Return problem handling request**  
  - Type: Respond to Webhook  
  - Role: Returns JSON error `{"status": "problem_handling_request"}` used if mapping the POST request fails.

---

#### 2.3 GET Route - Workflow Info Retrieval

**Overview:**  
Retrieves workflows information by invoking the n8n API node. Supports querying a specific workflow by ID or filtering workflows by active/inactive/all status. Returns either full workflow JSON or summarized info based on the `mode` parameter.

**Nodes Involved:**  
- Does workflowId Exist (If)  
- Get specific workflowid  
- Included Workflows (Switch)  
- get all active workflows  
- get all inactive workflows  
- get all workflows  
- full mode (If)  
- Map key workflow info  
- return all workflow info  
- return summarized workflow info  
- return problem getting workflow error  
- Return problem handling get request  

**Node Details:**

- **Does workflowId Exist**  
  - Type: If  
  - Role: Branches logic based on presence of `workflowId` query param.  
  - Condition: Checks if `workflowId` exists and is non-empty.  
  - Outputs:  
    - True: `Get specific workflowid`  
    - False: `Included Workflows`

- **Get specific workflowid**  
  - Type: n8n API (Workflow)  
  - Role: Fetches workflow details by specific ID using n8n API credentials.  
  - Config: `operation = get` with dynamic workflow ID.  
  - Outputs:  
    - Success: `full mode` (to decide output verbosity)  
    - Error: `return problem getting workflow error`

- **Included Workflows**  
  - Type: Switch  
  - Role: Routes based on `includedWorkflows` query param: `inactive`, `active`, `all`, or fallback.  
  - Outputs:  
    - `inactive`: `get all inactive workflows`  
    - `active`: `get all active workflows`  
    - `all`: `get all workflows`  
    - fallback: `get all active workflows` (default)

- **get all active workflows**  
  - Type: n8n API (Workflow)  
  - Role: Fetches all active workflows using API credentials.  
  - Outputs:  
    - Success: `full mode`  
    - Error: `return problem getting workflow error`

- **get all inactive workflows**  
  - Type: n8n API (Workflow)  
  - Role: Fetches all inactive workflows.  
  - Outputs:  
    - Success: `full mode`  
    - Error: `return problem getting workflow error`

- **get all workflows**  
  - Type: n8n API (Workflow)  
  - Role: Fetches all workflows regardless of status.  
  - Outputs:  
    - Success: `full mode`  
    - Error: `return problem getting workflow error`

- **full mode**  
  - Type: If  
  - Role: Decides if output should be full workflow JSON or summarized info based on `mode` query param.  
  - Condition: Checks if `mode` equals `full`.  
  - Outputs:  
    - True: Respond with `return all workflow info` (full JSON)  
    - False: Process summarized info.

- **Map key workflow info**  
  - Type: Set  
  - Role: Maps workflow JSON to a simplified object with key fields: `name`, `id`, `createdAt`, `updatedAt`, `active`.  
  - Outputs: To `return summarized workflow info`.

- **return all workflow info**  
  - Type: Respond to Webhook  
  - Role: Returns full workflow JSON with HTTP 200.  

- **return summarized workflow info**  
  - Type: Respond to Webhook  
  - Role: Returns simplified workflow info with HTTP 200.

- **return problem getting workflow error**  
  - Type: Respond to Webhook  
  - Role: Returns JSON error `{"status": "problem_getting_workflows"}` with HTTP 404.

- **Return problem handling get request**  
  - Type: Respond to Webhook  
  - Role: Returns JSON error `{"status": "problem_handling_request"}` for invalid GET requests.

---

#### 2.4 Common Error Handling

There are dedicated Respond to Webhook nodes to handle errors and validation failures, providing clear JSON error messages for:

- Problems handling the request (`Return problem handling request`)  
- Problems executing target workflows (`return problem executing workflow`)  
- Problems retrieving workflow info (`return problem getting workflow error`)  
- Missing or invalid query parameters (`Return problem handling get request`)

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                                           | Input Node(s)                  | Output Node(s)                                | Sticky Note                                                                                                                                                                                                                                                                               |
|-------------------------------|-------------------------|-----------------------------------------------------------|-------------------------------|-----------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook                 | API entry point, accepts GET/POST with header auth        | -                             | map parameters (GET), Map webhook request to fields (POST) | ## 🚀 Workflow Trigger: Webhook<br>This workflow starts with a **Webhook node** that can be triggered by both `GET` and `POST` HTTP requests.<br>**Authentication:** Header Auth with `Authorization` header and Bearer token.<br>Path: `workflow-manager`                                                    |
| map parameters                | Set                     | Extract GET query params: mode, workflowId, includedWorkflows | Webhook (GET)                  | Does workflowId Exist, Return problem handling get request | ## ▶️ POST Route: Execute a Target Workflow<br>Extracts GET parameters for handling logic.<br>                                                                                                                                                                                              |
| Map webhook request to fields | Set                     | Extract POST query `workflowId` and body as input data     | Webhook (POST)                 | Execute Workflow, Return problem handling request | ## ▶️ POST Route: Execute a Target Workflow<br>Extracts POST request data to run workflows dynamically.<br>Note: Triggered workflow must handle passed input data.<br>                                                                                                                                                               |
| Execute Workflow              | Execute Workflow         | Executes target workflow by ID, waits for completion       | Map webhook request to fields  | return succes message, return problem executing workflow |                                                                                                                                                                                                                                                                                           |
| return succes message         | Respond to Webhook       | Returns successful execution result                         | Execute Workflow               | -                                             |                                                                                                                                                                                                                                                                                           |
| return problem executing workflow | Respond to Webhook   | Returns error if workflow execution fails                   | Execute Workflow (error)       | -                                             |                                                                                                                                                                                                                                                                                           |
| Return problem handling request | Respond to Webhook     | Returns error if POST request mapping fails                 | Map webhook request to fields (error) | -                                    |                                                                                                                                                                                                                                                                                           |
| Does workflowId Exist         | If                      | Checks if workflowId param exists in GET request            | map parameters                 | Get specific workflowid, Included Workflows   |                                                                                                                                                                                                                                                                                           |
| Get specific workflowid       | n8n API (Workflow)       | Fetches workflow details by ID                              | Does workflowId Exist          | full mode, return problem getting workflow error | ## 📄 GET Route: Fetch Workflow Info<br>Uses n8n API to fetch specific workflow by ID.<br>Requires “n8n API” credentials.                                                                                                                                                                   |
| Included Workflows            | Switch                  | Routes to workflows filtered by active/inactive/all status | Does workflowId Exist (false)  | get all inactive workflows, get all active workflows, get all workflows, get all active workflows (fallback) |                                                                                                                                                                                                                                                                                           |
| get all inactive workflows    | n8n API (Workflow)       | Fetches inactive workflows                                  | Included Workflows             | full mode, return problem getting workflow error |                                                                                                                                                                                                                                                                                           |
| get all active workflows      | n8n API (Workflow)       | Fetches active workflows                                    | Included Workflows             | full mode, return problem getting workflow error |                                                                                                                                                                                                                                                                                           |
| get all workflows             | n8n API (Workflow)       | Fetches all workflows                                       | Included Workflows             | full mode, return problem getting workflow error |                                                                                                                                                                                                                                                                                           |
| full mode                    | If                      | Checks if mode param is 'full' to decide output verbosity  | Get specific workflowid, get all workflows, get all active workflows, get all inactive workflows | return all workflow info, Map key workflow info |                                                                                                                                                                                                                                                                                           |
| Map key workflow info         | Set                     | Maps workflow JSON to summarized info                       | full mode (false)              | return summarized workflow info               |                                                                                                                                                                                                                                                                                           |
| return all workflow info      | Respond to Webhook       | Returns full workflow JSON                                  | full mode (true)               | -                                             |                                                                                                                                                                                                                                                                                           |
| return summarized workflow info | Respond to Webhook     | Returns summarized workflow info                            | Map key workflow info          | -                                             |                                                                                                                                                                                                                                                                                           |
| return problem getting workflow error | Respond to Webhook | Returns error if workflow info retrieval fails             | Get specific workflowid (error), get all workflows (error), get all active workflows (error), get all inactive workflows (error) | - |                                                                                                                                                                                                                                                                                           |
| Return problem handling get request | Respond to Webhook  | Returns error if GET request is invalid                     | map parameters (error)         | -                                             |                                                                                                                                                                                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook`  
   - Path: `workflow-manager`  
   - HTTP Methods: Allow all (GET, POST)  
   - Authentication: Header Auth (create new credentials with: Header Name `Authorization`, Header Value `Bearer YOUR_STRONG_SECRET_KEY`)  
   - Response Mode: `responseNode` (response handled by downstream Respond to Webhook nodes)  
   - Position appropriately.

2. **Create Set Node for GET parameters extraction**  
   - Type: Set  
   - Name: `map parameters`  
   - Extract query parameters:  
     - `mode` from `{{$json.query.mode}}` (string)  
     - `workflowId` from `{{$json.query.workflowId}}` (string)  
     - `includedWorkflows` from `{{$json.query.includedWorkflows}}` (string)  
   - Set `Always Output Data` to false  
   - Position to receive GET method output from Webhook.

3. **Create Set Node for POST data extraction**  
   - Type: Set  
   - Name: `Map webhook request to fields`  
   - Extract:  
     - `targetWorkflowId` from `{{$json.query.workflowId}}` (string)  
     - `workflowInputData` from entire body `{{$json.body}}` (object)  
   - Set `On Error` to continue  
   - Position to receive POST method output from Webhook.

4. **Create If Node to check if workflowId exists (for GET)**  
   - Type: If  
   - Name: `Does workflowId Exist`  
   - Condition: Check if `{{$json.workflowId}}` exists and is not empty  
   - Position after `map parameters`.

5. **Create n8n API Node to get specific workflow by ID**  
   - Type: n8n API (Workflow)  
   - Name: `Get specific workflowid`  
   - Operation: `get`  
   - Workflow ID: `={{$json.workflowId}}`  
   - Credentials: Select your configured `n8n API` credentials  
   - Position after `Does workflowId Exist` (true branch).

6. **Create Switch Node to filter included workflows**  
   - Type: Switch  
   - Name: `Included Workflows`  
   - Condition on `{{$json.includedWorkflows}}` with cases:  
     - `inactive` → output `inactive`  
     - `active` → output `active`  
     - `all` → output `all`  
     - fallback → `extra` (renamed fallback)  
   - Position after `Does workflowId Exist` (false branch).

7. **Create n8n API Nodes to get workflows by status**  
   - `get all inactive workflows`: Operation: `getAll`, Filter: activeWorkflows = false  
   - `get all active workflows`: Operation: `getAll`, Filter: activeWorkflows = true  
   - `get all workflows`: Operation: `getAll`, no filter  
   - Credentials: Use same `n8n API` credentials  
   - Position connected respectively from `Included Workflows`.

8. **Create If Node to check if `mode` is 'full'**  
   - Type: If  
   - Name: `full mode`  
   - Condition: Check if `{{$json.mode}}` equals `full`  
   - Connect outputs from all API nodes to this node.

9. **Create Set Node to map key workflow info**  
   - Type: Set  
   - Name: `Map key workflow info`  
   - Assign fields:  
     - `name` = `{{$json.name}}`  
     - `id` = `{{$json.id}}`  
     - `createdAt` = `{{$json.createdAt}}`  
     - `updatedAt` = `{{$json.updatedAt}}`  
     - `active` = `{{$json.active}}` (boolean)  
   - Position on `full mode` (false branch).

10. **Create Respond to Webhook Nodes for GET responses and errors**  
    - `return all workflow info`: Responds with all incoming items, HTTP 200  
    - `return summarized workflow info`: Responds with all incoming items (mapped keys), HTTP 200  
    - `return problem getting workflow error`: Responds with JSON `{"status": "problem_getting_workflows"}`, HTTP 404  
    - `Return problem handling get request`: Responds with JSON `{"status": "problem_handling_request"}`  
    - Position connected accordingly.

11. **Create Execute Workflow Node for POST workflow execution**  
    - Type: Execute Workflow  
    - Name: `Execute Workflow`  
    - Workflow ID: Dynamic expression from `Map webhook request to fields` → `targetWorkflowId`  
    - Set option to wait for sub-workflow completion  
    - Set workflow input data mapping as needed (currently empty object in original, recommend mapping `workflowInputData` as input)  
    - Position after `Map webhook request to fields`.

12. **Create Respond to Webhook Nodes for POST success and error responses**  
    - `return succes message`: responds with all incoming items on success  
    - `return problem executing workflow`: responds with JSON `{"status": "problem_executing_workflow"}` on error  
    - `Return problem handling request`: fallback error response for POST request mapping failure  
    - Position connected accordingly.

13. **Connect all nodes as per the flow:**  
    - Webhook outputs: GET → `map parameters` → `Does workflowId Exist` → (branches)  
    - POST → `Map webhook request to fields` → `Execute Workflow`  
    - Connect error and success nodes as described.  

14. **Setup Credentials:**  
    - Header Authentication Credential for Webhook (`Authorization` header with Bearer token)  
    - n8n API Credential with access to the n8n instance API

15. **Test with sample requests:**  
    - POST with `workflowId` query param and JSON body to execute workflows dynamically  
    - GET with query params (`workflowId`, `includedWorkflows`, `mode`) to fetch workflow info

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow was used as backend for a Raycast extension allowing search and execution of n8n workflows directly from Raycast. See the extension’s code here: [n8n Manager for Raycast GitHub Repository](https://github.com/jwa91/n8n-manager-raycast/)                                                                          | Raycast extension integration example                                                                  |
| For detailed n8n API documentation used in this workflow, see: [https://docs.n8n.io/integrations/apis/n8n-api/](https://docs.n8n.io/integrations/apis/n8n-api/)                                                                                                                                                                  | Official n8n API docs                                                                                   |
| Header Authentication setup steps: Create new credentials in n8n > Header Auth > Name: `Authorization`, Value: `Bearer YOUR_STRONG_SECRET_KEY` (replace with your secure token). This protects the webhook endpoint.                                                                                                              | Authentication setup instructions                                                                       |
| **Important:** When using POST to execute workflows with input data, ensure the target workflow is designed to accept and process the incoming `workflowInputData` (e.g., via a “Workflow data” trigger node or by referencing `$json.workflowInputData`).                                                                           | Integration caveat                                                                                       |
| The workflow includes robust error handling nodes to provide clear JSON responses for common failure modes such as missing parameters, execution errors, or API failures.                                                                                                                                                         | Error handling best practices                                                                            |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.