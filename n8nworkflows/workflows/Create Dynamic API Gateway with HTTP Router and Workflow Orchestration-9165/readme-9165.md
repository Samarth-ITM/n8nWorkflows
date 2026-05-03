Create Dynamic API Gateway with HTTP Router and Workflow Orchestration

https://n8nworkflows.xyz/workflows/create-dynamic-api-gateway-with-http-router-and-workflow-orchestration-9165


# Create Dynamic API Gateway with HTTP Router and Workflow Orchestration

### 1. Workflow Overview

This workflow implements a **Dynamic API Gateway** that routes incoming HTTP requests via a single webhook URL to specific subflows based on the request's query parameter `action` and HTTP method. It supports multiple HTTP methods (GET, POST, DELETE, PATCH, PUT, HEAD) and centralizes routing, error handling, and response management.

The workflow logic is divided into the following blocks:

- **1.1 HTTP Method Detection:** Branches that hardcode and identify the HTTP method used in the incoming request, since the webhook node does not expose it directly.
- **1.2 Input Reception and Validation:** A webhook node receives all requests, then the workflow checks if the required query parameter `action` exists.
- **1.3 Routes Configuration:** Defines mappings of action names to allowed HTTP methods and subflow IDs to execute.
- **1.4 Route Resolution:** Matches the incoming request's action and method against the configured routes, deciding if a subflow should be run or an error should be returned.
- **1.5 Execution and Response:** Executes the resolved subflow if valid and returns its output; otherwise, returns appropriate HTTP error responses.

This design provides a centralized, maintainable, and extensible API gateway mechanism in n8n.

---

### 2. Block-by-Block Analysis

#### 1.1 HTTP Method Detection

**Overview:**  
Since n8n’s Webhook node does not expose the HTTP method directly, this block uses separate Set nodes to hardcode and tag each HTTP method for incoming requests. These method-specific branches are then aggregated into a single reference.

**Nodes Involved:**  
- GET  
- POST  
- DELETE  
- PATCH  
- PUT  
- HEAD  
- Aggregate Method Branches  
- Sticky Note: Method detection (explanatory)

**Node Details:**

- **GET, POST, DELETE, PATCH, PUT, HEAD**  
  - Type: Set  
  - Role: Each sets a JSON property `method` with the respective HTTP method string.  
  - Configuration: Raw mode with JSON output explicitly setting `"method": "<METHOD>"`.  
  - Input: Connected directly from the Webhook node ("Universal Receiver") with respective method filtering.  
  - Output: All connect to "Aggregate Method Branches".  
  - Edge cases: None significant; assumes branching logic correctly captures the HTTP method.  

- **Aggregate Method Branches**  
  - Type: Set  
  - Role: Consolidates the output from all method branches into one unified JSON object for further processing.  
  - Configuration: Raw mode, passes through the input JSON unchanged (`={{ $json }}`).  
  - Input: Receives from all method branches.  
  - Output: Connects to "Query param exists?".  
  - Edge cases: None, as it just passes data forward.

- **Sticky Note: Method detection**  
  - Content summarizes the purpose of method branches and aggregation.

---

#### 1.2 Input Reception and Validation

**Overview:**  
Receives any HTTP request on a universal webhook path, validates the presence of the `action` query parameter, and routes accordingly.

**Nodes Involved:**  
- Universal Receiver (Webhook)  
- Query param exists? (If)  
- [Error] Required query param missing (Respond To Webhook)  
- Sticky Note: Method detection1 (overall flow explanation)

**Node Details:**

- **Universal Receiver**  
  - Type: Webhook  
  - Role: Entry point for all incoming HTTP requests to path `/universalReceiver`.  
  - Configuration: Accepts all HTTP methods (GET, POST, DELETE, PATCH, PUT, HEAD) with response mode set to respond via a node.  
  - Input: External HTTP calls.  
  - Output: Branches out to method detection nodes (GET, POST, etc.).  
  - Edge cases: Webhook misconfiguration, unexpected HTTP methods (though all standard ones are covered).  
  - Note: Webhook ID is fixed for this endpoint.

- **Query param exists?**  
  - Type: If (Condition node)  
  - Role: Checks if the query parameter `action` exists in the incoming request.  
  - Configuration: Condition tests existence of `query.action` in the JSON of "Universal Receiver" output.  
  - Input: From "Aggregate Method Branches".  
  - Output: True branch leads to "Routes Config"; false branch leads to error response node.  
  - Edge cases: Missing or malformed query parameters.

- **[Error] Required query param missing**  
  - Type: Respond To Webhook  
  - Role: Returns HTTP 400 with JSON error if `action` param is missing.  
  - Configuration: Response code 400, response body JSON `{ "error": "Required information not provided!" }`.  
  - Input: False output from "Query param exists?".  
  - Output: Terminal node (responds to client).  
  - Edge cases: None beyond missing parameter.

- **Sticky Note: Method detection1**  
  - Provides an extensive explanation of the entire flow's logic, especially the method branching and routing concept.

---

#### 1.3 Routes Configuration

**Overview:**  
Defines the routing table that maps `action` names to allowed HTTP methods and subflow IDs which represent workflows to execute.

**Nodes Involved:**  
- Routes Config (Set)  
- Sticky Note: Routes config

**Node Details:**

- **Routes Config**  
  - Type: Set  
  - Role: Contains static JSON array `routes` with objects defining:  
    - `action`: string action name  
    - `allowedMethods`: array of HTTP methods allowed for this action  
    - `subflowId`: numerical ID of the workflow to invoke  
  - Configuration: Raw JSON mode with hardcoded route definitions, e.g.:  
    ```json
    {
      "routes": [
        { "action": "doSomething", "allowedMethods": ["GET"], "subflowId": -1 },
        { "action": "doSomethingElse", "allowedMethods": ["GET", "POST"], "subflowId": -1 },
        { "action": "deleteSomething", "allowedMethods": ["DELETE"], "subflowId": -1 }
      ]
    }
    ```  
  - Input: True output from "Query param exists?".  
  - Output: Connects to "Resolve".  
  - Edge cases: Invalid or missing route definitions; subflow IDs must be valid for execution.  
  - Note: This is easily modified or replaced by dynamic data sources (e.g., database).

- **Sticky Note: Routes config**  
  - Explains purpose of the route configuration and alternative ways to define routes.

---

#### 1.4 Route Resolution

**Overview:**  
Evaluates the incoming request's `action` and HTTP method against the configured routes, determines if a matching route exists, and prepares execution details or error information.

**Nodes Involved:**  
- Resolve (Code)  
- Route is OK? (If)  
- Sticky Note: Resolver

**Node Details:**

- **Resolve**  
  - Type: Code (JavaScript)  
  - Role:  
    - Reads the `method` from aggregated method branches.  
    - Reads `action` from query parameters.  
    - Loads routes from the "Routes Config".  
    - Determines if the action exists, if the method is allowed, and identifies the subflow ID to execute.  
    - Sets appropriate error codes (400, 404, 405, 500) if no match or configuration errors occur.  
    - Returns an object with keys: `action`, `method`, `error`, `status`, `allow` (header), `workflowIdToExecute`, `body`.  
  - Configuration: Runs once per item, robust logic with detailed conditions and fallback.  
  - Input: From "Routes Config".  
  - Output: Connects to "Route is OK?" conditional node.  
  - Edge cases: Missing action, route not found, method not allowed, missing subflow ID.  
  - Potential failure: Expression errors if any input is malformed.

- **Route is OK?**  
  - Type: If  
  - Role: Checks if a valid `workflowIdToExecute` exists (i.e., route resolution succeeded).  
  - Configuration: Tests if `workflowIdToExecute` exists (non-empty).  
  - Input: From "Resolve".  
  - Output:  
    - True branch leads to "Execute Workflow".  
    - False branch leads to "Status = 405?" conditional node.  
  - Edge cases: Missing workflow ID or unresolved route.

- **Sticky Note: Resolver**  
  - Explains the route resolution logic and its role in deciding subflow execution.

---

#### 1.5 Execution and Response

**Overview:**  
Executes the resolved subflow based on routing or returns appropriate error responses based on resolution results.

**Nodes Involved:**  
- Execute Workflow  
- Success (Respond To Webhook)  
- [Error] Method Not Allowed (Respond To Webhook)  
- Status = 405? (If)  
- Error - Not OK (Respond To Webhook)  
- Error - Subflow (Respond To Webhook)  
- Sticky Note: Execute & respond

**Node Details:**

- **Execute Workflow**  
  - Type: Execute Workflow  
  - Role: Runs the subflow identified by `workflowIdToExecute` from the route resolution.  
  - Configuration:  
    - Mode: each item (multiple executions if multiple items).  
    - Wait for subworkflow completion before continuing.  
    - Workflow ID is dynamic, read from previous node output.  
    - No inputs passed explicitly to subflow; empty inputs.  
  - Input: True output from "Route is OK?".  
  - Output:  
    - Success branch leads to "Success" response node.  
    - Error branch leads to "Error - Subflow" node.  
  - Edge cases: Subflow execution failure, timeout, invalid workflow ID.  
  - Version requirement: n8n version supporting Execute Workflow node version 1.3 or higher.

- **Success**  
  - Type: Respond To Webhook  
  - Role: Returns HTTP 200 with the JSON output from the executed subflow.  
  - Configuration: Response code 200, body set to the JSON output of subworkflow execution.  
  - Input: Success output from "Execute Workflow".  
  - Output: Terminal response.  
  - Edge cases: Subflow returns invalid JSON or empty result.

- **Status = 405?**  
  - Type: If  
  - Role: Checks if the error status from route resolution equals 405 (Method Not Allowed).  
  - Configuration: Compares `status` field from "Resolve" output to 405.  
  - Input: False output from "Route is OK?".  
  - Output:  
    - True branch to "[Error] Method Not Allowed".  
    - False branch to "Error - Not OK".  
  - Edge cases: Other error codes than 405.

- **[Error] Method Not Allowed**  
  - Type: Respond To Webhook  
  - Role: Returns HTTP 405 with JSON error and `Allow` header listing allowed methods.  
  - Configuration:  
    - Response code 405.  
    - Response header `Allow` set dynamically from `allow` property in "Resolve" output.  
    - Response body includes the error message from resolve node.  
  - Input: True output from "Status = 405?".  
  - Output: Terminal.  
  - Edge cases: Missing or empty Allow header.

- **Error - Not OK**  
  - Type: Respond To Webhook  
  - Role: Generic error response for errors other than 405.  
  - Configuration:  
    - Response code dynamically set from `status` in "Resolve" output or defaults to 400.  
    - JSON body includes error message.  
  - Input: False output from "Status = 405?".  
  - Output: Terminal.  
  - Edge cases: None particular.

- **Error - Subflow**  
  - Type: Respond To Webhook  
  - Role: Returns HTTP 500 when the executed subflow errors out unexpectedly.  
  - Configuration: Response code 500 with JSON error message.  
  - Input: Error output from "Execute Workflow".  
  - Output: Terminal.  
  - Edge cases: Subflow failure conditions.

- **Sticky Note: Execute & respond**  
  - Describes execution of the matched subflow and response handling.

---

### 3. Summary Table

| Node Name                    | Node Type              | Functional Role                                              | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                       |
|------------------------------|------------------------|--------------------------------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------|
| Universal Receiver            | Webhook                | Entry point for all HTTP requests with multiple methods      | External HTTP                 | GET, POST, DELETE, PATCH, PUT, HEAD | Sticky: Method detection1 (Overall flow explanation)             |
| GET                          | Set                    | Hardcodes HTTP method as GET                                 | Universal Receiver            | Aggregate Method Branches      | Sticky: Method detection                                         |
| POST                         | Set                    | Hardcodes HTTP method as POST                                | Universal Receiver            | Aggregate Method Branches      | Sticky: Method detection                                         |
| DELETE                       | Set                    | Hardcodes HTTP method as DELETE                              | Universal Receiver            | Aggregate Method Branches      | Sticky: Method detection                                         |
| PATCH                        | Set                    | Hardcodes HTTP method as PATCH                               | Universal Receiver            | Aggregate Method Branches      | Sticky: Method detection                                         |
| PUT                          | Set                    | Hardcodes HTTP method as PUT                                 | Universal Receiver            | Aggregate Method Branches      | Sticky: Method detection                                         |
| HEAD                         | Set                    | Hardcodes HTTP method as HEAD                                | Universal Receiver            | Aggregate Method Branches      | Sticky: Method detection                                         |
| Aggregate Method Branches    | Set                    | Aggregates method info from all branches                      | GET, POST, DELETE, PATCH, PUT, HEAD | Query param exists?          | Sticky: Method detection                                         |
| Query param exists?           | If                     | Checks if `action` query param exists                         | Aggregate Method Branches     | Routes Config, [Error] Required query param missing |                                                                  |
| [Error] Required query param missing | Respond To Webhook | Returns 400 error if `action` param missing                   | Query param exists? (false)   | None                          |                                                                  |
| Routes Config                | Set                    | Defines routing table: action, allowed methods, subflow IDs  | Query param exists? (true)    | Resolve                       | Sticky: Routes config                                            |
| Resolve                     | Code                   | Resolves action & method against routes, prepares execution info | Routes Config                | Route is OK?                  | Sticky: Resolver                                                |
| Route is OK?                 | If                     | Checks if a valid subflow ID was found                        | Resolve                      | Execute Workflow, Status = 405? | Sticky: Resolver                                                |
| Execute Workflow             | Execute Workflow       | Runs the resolved subflow workflow                            | Route is OK? (true)           | Success, Error - Subflow      | Sticky: Execute & respond                                       |
| Success                     | Respond To Webhook     | Returns 200 response with subflow output                      | Execute Workflow (success)    | None                          | Sticky: Execute & respond                                       |
| Status = 405?                | If                     | Checks if error status is 405                                 | Route is OK? (false)          | [Error] Method Not Allowed, Error - Not OK | Sticky: Execute & respond                                       |
| [Error] Method Not Allowed   | Respond To Webhook     | Returns 405 error with Allow header                           | Status = 405? (true)          | None                          | Sticky: Execute & respond                                       |
| Error - Not OK               | Respond To Webhook     | Returns generic error response for other errors              | Status = 405? (false)         | None                          | Sticky: Execute & respond                                       |
| Error - Subflow              | Respond To Webhook     | Returns 500 error if subflow execution fails                  | Execute Workflow (error)      | None                          | Sticky: Execute & respond                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Universal Receiver":**  
   - Type: Webhook  
   - Path: `universalReceiver`  
   - HTTP Methods: Select all: GET, POST, DELETE, PATCH, PUT, HEAD  
   - Response Mode: "Respond with a node"  
   - Position it on the left side for clarity.

2. **Create Method Detection Branches:**  
   For each HTTP method (GET, POST, DELETE, PATCH, PUT, HEAD):  
   - Add a **Set** node named with the HTTP method (e.g., "GET").  
   - Set mode: raw JSON output: `{"method": "<METHOD>"}` (replace `<METHOD>` with the actual HTTP method string).  
   - Connect "Universal Receiver" node output for the matching HTTP method to this Set node.

3. **Aggregate Method Branches:**  
   - Create a **Set** node named "Aggregate Method Branches".  
   - Mode: raw JSON output: `={{ $json }}` (passes input unchanged).  
   - Connect all method Set nodes (GET, POST, etc.) outputs to this node.

4. **Check Query Parameter Exists:**  
   - Add an **If** node named "Query param exists?".  
   - Condition: Check if `$('Universal Receiver').item.json.query?.action` exists.  
   - Connect output of "Aggregate Method Branches" to this node.

5. **Create Error Response for Missing Param:**  
   - Add a **Respond To Webhook** node named "[Error] Required query param missing".  
   - Response code: 400  
   - Response body: `{ "error": "Required information not provided!" }`  
   - Connect false output of "Query param exists?" to this node.

6. **Define Routes Configuration:**  
   - Add a **Set** node named "Routes Config".  
   - Raw JSON output with structure:  
     ```json
     {
       "routes": [
         {"action": "doSomething", "allowedMethods": ["GET"], "subflowId": -1},
         {"action": "doSomethingElse", "allowedMethods": ["GET", "POST"], "subflowId": -1},
         {"action": "deleteSomething", "allowedMethods": ["DELETE"], "subflowId": -1}
       ]
     }
     ```  
   - Connect true output of "Query param exists?" to this node.

7. **Create Route Resolution Code Node:**  
   - Add a **Code** node named "Resolve".  
   - JavaScript code:  
     ```js
     const method = $('Aggregate Method Branches').item.json.method;
     const action = $("Universal Receiver").item.json.query?.action;
     const cfg = $("Routes Config").item.json;
     const routes = Array.isArray(cfg?.routes) ? cfg.routes : [];
     const body = $("Universal Receiver").item.json.query?.body || null;

     const routeByAction = routes.find(r => String(r.action || '').toLowerCase() === String(action || '').toLowerCase());
     const routeByActionAndMethod = routes.find(r => {
       const actionFound = String(r.action || '').toLowerCase() === String(action || '').toLowerCase();
       const methodAllowed = Array.isArray(r.allowedMethods) && r.allowedMethods.includes(method);
       return (actionFound && methodAllowed)
     });

     let error = null;
     let status = 200;
     let allowHeader = null;
     let workflowId = null;

     if (!action) {
       error = 'MISSING_ACTION';
       status = 400;
     } else if (!routeByAction) {
       error = 'ROUTE_NOT_FOUND';
       status = 404;
     } else if (!routeByActionAndMethod) {
       error = 'METHOD_NOT_ALLOWED';
       status = 405;
       allowHeader = Array.isArray(routeByAction.allowedMethods) ? routeByAction.allowedMethods.join(', ') : '';
     } else {
       workflowId = routeByActionAndMethod.subflowId ?? null;
       if (workflowId == null) {
         error = 'MISSING_SUBFLOW_ID';
         status = 500;
       }
     }

     return {
       action,
       method,
       error,
       status,
       allow: allowHeader,
       workflowIdToExecute: workflowId,
       body
     };
     ```  
   - Connect output of "Routes Config" to this node.

8. **Check if Route is OK:**  
   - Add an **If** node named "Route is OK?".  
   - Condition: Check if `workflowIdToExecute` exists (non-empty).  
   - Connect output of "Resolve" to this node.

9. **Execute Workflow Node:**  
   - Add an **Execute Workflow** node named "Execute Workflow".  
   - Mode: "each item" with wait for subworkflow enabled.  
   - Workflow ID: Use expression `={{ $('Resolve').item.json.workflowIdToExecute }}` (dynamic).  
   - Workflow inputs: empty object `{}`.  
   - Connect true output of "Route is OK?" to this node.

10. **Success Response Node:**  
    - Add a **Respond To Webhook** node named "Success".  
    - Response code: 200  
    - Respond with JSON: output of subworkflow (use expression: `={{ $json }}`).  
    - Connect success output of "Execute Workflow" to this node.

11. **Error Response for Subflow Failure:**  
    - Add a **Respond To Webhook** node named "Error - Subflow".  
    - Response code: 500  
    - Response body: `{ "error": "Unexpected error in the executed subflow!" }`  
    - Connect error output of "Execute Workflow" to this node.

12. **Check if Status is 405:**  
    - Add an **If** node named "Status = 405?".  
    - Condition: Check if `status` from "Resolve" equals 405.  
    - Connect false output of "Route is OK?" to this node.

13. **Method Not Allowed Response:**  
    - Add a **Respond To Webhook** node named "[Error] Method Not Allowed".  
    - Response code: 405  
    - Response header: `Allow` set to `={{ $('Resolve').item.json.allow || '' }}`  
    - Response body: JSON with error message from resolve node.  
    - Connect true output of "Status = 405?" to this node.

14. **Generic Error Response:**  
    - Add a **Respond To Webhook** node named "Error - Not OK".  
    - Response code: dynamic from `={{ $('Resolve').item.json.status || 400 }}`  
    - Response body: JSON with error message from resolve node.  
    - Connect false output of "Status = 405?" to this node.

15. **Connect Nodes as per above steps:**  
    - Universal Receiver → method branches (GET, POST, etc.)  
    - Method branches → Aggregate Method Branches → Query param exists?  
    - Query param exists? true → Routes Config → Resolve → Route is OK?  
    - Route is OK? true → Execute Workflow → Success / Error - Subflow  
    - Route is OK? false → Status = 405? → [Error] Method Not Allowed / Error - Not OK  
    - Query param exists? false → [Error] Required query param missing

16. **Credentials:**  
    - No special credentials are required for this workflow unless the subflows executed require them.  
    - Ensure that the subflows identified by subflow IDs exist and have proper credentials configured.

17. **Testing:**  
    - Deploy the workflow and test by sending HTTP requests to `/universalReceiver` with query parameter `action` set to one of the configured routes and using supported HTTP methods.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow centralizes API routing, enabling a single webhook to handle multiple HTTP methods and actions dynamically.                         | Workflow design explanation                                   |
| Subflow execution is dynamic based on configuration, allowing easy extension by adding routes and workflows.                                       | Extendability best practice                                   |
| For broader integration, route definitions can be replaced by a database or external service instead of static JSON in the "Routes Config" node. | Scalability and maintainability advice                         |
| Error handling covers missing parameters, unknown routes, method not allowed, and subflow execution failures with appropriate HTTP status codes.  | API error handling standard                                   |
| Sticky notes in the workflow provide rich explanations for each logical block.                                                                     | Inline documentation best practice                            |
| See n8n documentation on [Execute Workflow](https://docs.n8n.io/nodes/n8n-nodes-base.executeWorkflow/) and [Respond to Webhook](https://docs.n8n.io/nodes/n8n-nodes-base.respondToWebhook/) nodes for details on configuration. | Official n8n node references                                 |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.