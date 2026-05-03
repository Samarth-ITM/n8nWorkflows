Monitor & Debug n8n Workflows with Claude AI Assistant and MCP Server

https://n8nworkflows.xyz/workflows/monitor---debug-n8n-workflows-with-claude-ai-assistant-and-mcp-server-10779


# Monitor & Debug n8n Workflows with Claude AI Assistant and MCP Server

### 1. Workflow Overview

This workflow provides a powerful monitoring and debugging endpoint for n8n instances by querying n8n’s API to gather detailed insights about workflows and their executions. It is designed for use cases such as operational health checks, failure analysis, and performance auditing of n8n workflows.

The workflow operates through a single webhook entry point that accepts POST requests with an action parameter. Based on this action, it routes the request to one of three main functional blocks:

- **1.1 Input Reception & Routing:** Receives HTTP POST requests and routes them by action type.
- **1.2 Data Retrieval via API Calls:** Fetches data from the n8n API about active workflows, recent executions, or execution details with errors.
- **1.3 Data Processing & Response:** Processes retrieved data to compute KPIs and failure metrics, then responds back to the requester.

The logical blocks and their roles:

- **1.1 Input Reception & Routing**  
  - Nodes: Webhook, Route by Action  
  - Role: Accepts external requests, inspects the action parameter, and routes processing accordingly.

- **1.2 Data Retrieval via API Calls**  
  - Nodes: Get Active Workflows, Get Last Executions, Get Executions Details (Status = Error), Extract All Executions, Split Active Executions, Details of Executions, Active Workflows  
  - Role: Performs HTTP requests to the n8n API to retrieve workflows and executions data, then splits and structures this data for processing.

- **1.3 Data Processing & Response**  
  - Nodes: Processing Audit, Respond to Webhook  
  - Role: Analyzes execution data to produce detailed KPIs (e.g., failure rates, execution counts, timing metrics), then sends the results back through the webhook response.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing

- **Overview:**  
  This block receives monitoring requests and decides which data retrieval path to follow based on the `action` field in the request body.

- **Nodes Involved:**  
  - Webhook  
  - Route by Action

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP POST endpoint)  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `8d8ea5d4-986e-46f3-adee-11eda64f8f60` (unique webhook path)  
      - Response Mode: Response node (delays response until Respond to Webhook node)  
      - Accepts JSON body, expects `action`, `limit`, and `workflow_id` parameters.  
    - Inputs: External HTTP POST requests  
    - Outputs: Forwarded JSON data to Route by Action  
    - Failure modes: Invalid/missing action parameter, malformed JSON, webhook timeout if response delayed too long.

  - **Route by Action**  
    - Type: Switch  
    - Configuration:  
      - Routes on the value of `{{$json.body.action}}` with exact string matches:  
        - `get_active_workflows` → output "active_workflows"  
        - `get_workflow_executions` → output "executions_worfklows" (note typo in output key)  
        - `get_execution_details` → output "execution_details"  
      - Case-sensitive, strict string validation.  
    - Inputs: From Webhook node  
    - Outputs: Routes to one of three API calls nodes accordingly  
    - Failure modes: Unknown action value results in no output branch triggered, halting workflow.

#### 2.2 Data Retrieval via API Calls

- **Overview:**  
  This block performs HTTP requests to the n8n API to gather relevant monitoring data, then splits and structures the data for analysis.

- **Nodes Involved:**  
  - Get Active Workflows  
  - Get Last Executions  
  - Get Executions Details (Status = Error)  
  - Extract All Executions  
  - Split Active Executions  
  - Details of Executions  
  - Active Workflows

- **Node Details:**

  - **Get Active Workflows**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://<YOUR_N8N_INSTANCE>/api/v1/workflows` (must be replaced with user instance URL)  
      - Query Parameters:  
        - `active=true` (filters active workflows)  
        - `limit={{ $json.body.workflow_id }}` (uses workflow_id from request body as limit, unusual usage)  
        - `excludePinnedData=true` (exclude pinned data)  
      - Authentication: Predefined credential (named "n8n account") using n8n API credentials  
    - Inputs: From Route by Action (active_workflows branch)  
    - Outputs: JSON with workflows list, forwarded to Split Active Executions  
    - Failure modes: Network issues, auth failures, invalid credentials, API URL misconfiguration.

  - **Get Last Executions**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://<YOUR_N8N_INSTANCE>/api/v1/executions`  
      - Query Parameters:  
        - `includeData=false` (exclude execution data details for efficiency)  
        - `limit={{ $('Webhook').item.json.body.limit }}` (limit of executions based on request body)  
      - Authentication: Same as above  
    - Inputs: From Route by Action (executions_worfklows branch)  
    - Outputs: JSON with executions array, forwarded to Extract All Executions  
    - Failure modes: Same as above; also watch for invalid limit values or missing parameters.

  - **Get Executions Details (Status = Error)**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://<YOUR_N8N_INSTANCE>/api/v1/executions`  
      - Query Parameters:  
        - `includeData=true` (include full execution data)  
        - `limit={{ $('Webhook').item.json.body.limit }}`  
        - `workflowId={{ $('Webhook').item.json.body.workflow_id }}`  
        - `status=error` (filter to error executions only)  
      - Authentication: Same as above  
    - Inputs: From Route by Action (execution_details branch)  
    - Outputs: JSON of error executions, forwarded to Details of Executions  
    - Failure modes: Same as above, plus potential missing workflowId parameter.

  - **Extract All Executions**  
    - Type: SplitOut  
    - Configuration: Splits the `data` array field into individual items for processing.  
    - Inputs: From Get Last Executions  
    - Outputs: Individual execution items forwarded to Processing Audit  
    - Failure modes: If `data` field missing or empty, no outputs.

  - **Split Active Executions**  
    - Type: SplitOut  
    - Configuration: Splits the `data` array field from active workflows response into individual workflows.  
    - Inputs: From Get Active Workflows  
    - Outputs: Individual workflows forwarded to Active Workflows node  
    - Failure modes: Missing or empty `data` field.

  - **Details of Executions**  
    - Type: SplitOut  
    - Configuration: Splits detailed error executions array from Get Executions Details node.  
    - Inputs: From Get Executions Details (Status = Error)  
    - Outputs: Individual error execution items forwarded to Respond to Webhook  
    - Failure modes: Similar to other SplitOut nodes.

  - **Active Workflows**  
    - Type: Set  
    - Configuration: Maps selected workflow properties (`createdAt`, `updatedAt`, `id`, `name`, `isArchived`) from JSON to simplified output format.  
    - Inputs: From Split Active Executions  
    - Outputs: Forwarded to Respond to Webhook  
    - Failure modes: Missing expected fields in workflow JSON.

#### 2.3 Data Processing & Response

- **Overview:**  
  Processes the execution logs to calculate monitoring KPIs, analyze failures, execution modes, timing metrics, and prepare alerts. Ultimately sends the analyzed data back through the webhook response.

- **Nodes Involved:**  
  - Processing Audit  
  - Respond to Webhook

- **Node Details:**

  - **Processing Audit**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Processes all execution items passed to it.  
      - Computes:  
        - Total, successful, failed executions and rates  
        - Unique workflows with failures  
        - Grouped failures by workflow  
        - Execution modes distribution  
        - Workflow-specific metrics including success/failure rates and last execution time  
        - Time-series executions count by hour  
        - Average, max, min execution times  
        - Recent failure details (top 5)  
        - Alerts triggered if failure rate exceeds threshold (currently 10%)  
      - Returns a comprehensive KPI object to output.  
    - Inputs: From Extract All Executions  
    - Outputs: KPI JSON forwarded to Respond to Webhook  
    - Failure modes: Errors in code execution, missing data fields, empty inputs.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration:  
      - Responds with all incoming items (full JSON output from previous node)  
    - Inputs: From Active Workflows or Processing Audit or Details of Executions nodes depending on route  
    - Outputs: HTTP response sent back to original webhook caller  
    - Failure modes: Response timeout, malformed response data.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                            | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                               |
|-------------------------------|-------------------------|-------------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook                 | Entry point receiving monitoring requests | (External HTTP POST)           | Route by Action                | ## 1. Webhook Entry with a switch by action Receives POST requests with action parameter to route monitoring requests      |
| Route by Action               | Switch                  | Routes requests by `action` field          | Webhook                       | Get Active Workflows, Get Last Executions, Get Executions Details (Status = Error) |                                                                                                                           |
| Get Active Workflows          | HTTP Request            | Fetch active workflows from n8n API        | Route by Action               | Split Active Executions        | ## 3. Get all the active workflows of the instance                                                                        |
| Get Last Executions           | HTTP Request            | Fetch latest workflow executions            | Route by Action               | Extract All Executions         | ## 2. Get last executions logs and process them                                                                             |
| Get Executions Details (Status = Error) | HTTP Request | Fetch detailed error executions for workflow | Route by Action               | Details of Executions          | ## 4. Get the detailed executions logs for a specific workflow                                                             |
| Extract All Executions        | SplitOut                | Split executions array into individual items | Get Last Executions           | Processing Audit               |                                                                                                                           |
| Split Active Executions       | SplitOut                | Split active workflows array into items    | Get Active Workflows          | Active Workflows               |                                                                                                                           |
| Details of Executions         | SplitOut                | Split detailed error executions array      | Get Executions Details (Status = Error) | Respond to Webhook            |                                                                                                                           |
| Active Workflows              | Set                     | Simplify and format active workflows data | Split Active Executions       | Respond to Webhook             |                                                                                                                           |
| Processing Audit              | Code                    | Analyze execution data to compute KPIs     | Extract All Executions        | Respond to Webhook             |                                                                                                                           |
| Respond to Webhook            | Respond to Webhook      | Send processed data back as HTTP response  | Active Workflows, Processing Audit, Details of Executions | (HTTP response) | ## 5. Send Response back                                                                                                   |
| Main Overview                | Sticky Note             | Workflow purpose and setup instructions    | -                             | -                             | ## Workflow Monitoring Endpoint ... [YouTube tutorial link](https://www.youtube.com/watch?v=oJzNnHIusZs)                     |
| Webhook Entry                | Sticky Note             | Explains webhook entry and routing         | -                             | -                             | ## 1. Webhook Entry with a switch by action                                                                               |
| API Calls                   | Sticky Note             | Describes API calls for last executions    | -                             | -                             | ## 2. Get last executions logs and process them                                                                           |
| API Calls1                  | Sticky Note             | Describes API calls for active workflows   | -                             | -                             | ## 3. Get all the active workflows of the instance                                                                        |
| API Calls2                  | Sticky Note             | Describes API calls for execution details  | -                             | -                             | ## 4. Get the detailed executions logs for a specific workflow                                                            |
| Response                    | Sticky Note             | Notes about sending the response            | -                             | -                             | ## 5. Send Response back                                                                                                   |
| Video Tutorial              | Sticky Note             | Link to video tutorial on usage             | -                             | -                             | ## [Tutorial](https://www.youtube.com/watch?v=oJzNnHIusZs) @[youtube](oJzNnHIusZs)                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `8d8ea5d4-986e-46f3-adee-11eda64f8f60` (or any unique path)  
   - Response Mode: Response Node  
   - Enable JSON body parsing (default)  

2. **Create Switch Node "Route by Action"**  
   - Connect input from Webhook node  
   - Add three outputs with rules:  
     - If `{{$json.body.action}}` equals `get_active_workflows` → output 1  
     - If equals `get_workflow_executions` → output 2  
     - If equals `get_execution_details` → output 3  
   - Use strict case-sensitive string equals conditions.

3. **Create HTTP Request Node "Get Active Workflows"**  
   - URL: `https://<YOUR_N8N_INSTANCE>/api/v1/workflows` (replace placeholder)  
   - Authentication: Use n8n API credentials (OAuth2 or API key)  
   - Query Parameters:  
     - active = true  
     - limit = `={{ $json.body.workflow_id }}` (Note: This may be intended as a workflow ID filter or limit)  
     - excludePinnedData = true  
   - Connect input from "Route by Action" output 1

4. **Create HTTP Request Node "Get Last Executions"**  
   - URL: `https://<YOUR_N8N_INSTANCE>/api/v1/executions`  
   - Authentication: Same as above  
   - Query Parameters:  
     - includeData = false  
     - limit = `={{ $json.body.limit }}` from webhook body  
   - Connect input from "Route by Action" output 2

5. **Create HTTP Request Node "Get Executions Details (Status = Error)"**  
   - URL: `https://<YOUR_N8N_INSTANCE>/api/v1/executions`  
   - Authentication: Same as above  
   - Query Parameters:  
     - includeData = true  
     - limit = `={{ $json.body.limit }}`  
     - workflowId = `={{ $json.body.workflow_id }}`  
     - status = error  
   - Connect input from "Route by Action" output 3

6. **Create SplitOut Node "Extract All Executions"**  
   - Field to split out: `data`  
   - Connect input from "Get Last Executions"

7. **Create SplitOut Node "Split Active Executions"**  
   - Field to split out: `data`  
   - Connect input from "Get Active Workflows"

8. **Create SplitOut Node "Details of Executions"**  
   - Field to split out: `data`  
   - Connect input from "Get Executions Details (Status = Error)"

9. **Create Set Node "Active Workflows"**  
   - Assign fields:  
     - createdAt: `{{$json.createdAt}}`  
     - updatedAt: `{{$json.updatedAt}}`  
     - id: `{{$json.id}}`  
     - name: `{{$json.name}}`  
     - isArchived: `{{$json.isArchived}}`  
   - Connect input from "Split Active Executions"

10. **Create Code Node "Processing Audit"**  
    - Paste the provided JavaScript code to process executions and compute KPIs  
    - Connect input from "Extract All Executions"

11. **Create Respond to Webhook Node "Respond to Webhook"**  
    - Respond with all incoming items  
    - Connect inputs from:  
      - "Active Workflows" node  
      - "Processing Audit" node  
      - "Details of Executions" node  

12. **Connect all outputs accordingly:**  
    - From "Split Active Executions" → "Active Workflows" → "Respond to Webhook"  
    - From "Extract All Executions" → "Processing Audit" → "Respond to Webhook"  
    - From "Details of Executions" → "Respond to Webhook"

13. **Configure Credentials**  
    - Create or select n8n API credentials with access to your n8n instance API.  
    - Use these credentials in all HTTP Request nodes.

14. **Replace all `<YOUR_N8N_INSTANCE>` placeholders with your actual n8n instance URL.**

15. **Test the workflow**  
    - Send POST requests to the webhook path with JSON body containing:  
      - `action`: one of `get_active_workflows`, `get_workflow_executions`, or `get_execution_details`  
      - `limit`: integer limit for executions (where applicable)  
      - `workflow_id`: workflow ID for specific queries (where applicable)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                      | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow combined with a local MCP Server enables health monitoring of any n8n instance through API endpoints.                                   | Main Overview sticky note                                                                         |
| Setup instructions include replacing instance URLs, configuring API credentials, and setting request body parameters (`limit`, `workflow_id`).   | Main Overview sticky note                                                                         |
| Monitoring can be customized by modifying the Processing Audit node’s failure thresholds and KPIs.                                               | Main Overview sticky note                                                                         |
| Video tutorial explaining workflow usage and setup available on YouTube.                                                                         | [Tutorial Video](https://www.youtube.com/watch?v=oJzNnHIusZs)                                    |
| Sticky notes provide stepwise explanations for webhook entry, API calls, and response handling.                                                  | Various sticky notes attached to nodes                                                           |

---

_Disclaimer: The provided text originates solely from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public._