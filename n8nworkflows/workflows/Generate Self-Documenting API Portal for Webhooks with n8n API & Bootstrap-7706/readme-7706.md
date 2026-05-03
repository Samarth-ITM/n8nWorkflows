Generate Self-Documenting API Portal for Webhooks with n8n API & Bootstrap

https://n8nworkflows.xyz/workflows/generate-self-documenting-api-portal-for-webhooks-with-n8n-api---bootstrap-7706


# Generate Self-Documenting API Portal for Webhooks with n8n API & Bootstrap

### 1. Workflow Overview

This n8n workflow automates the generation of a live, self-documenting API portal for all webhooks defined across active workflows within an n8n instance. It scans available workflows for those that expose webhooks with attached documentation metadata, aggregates this data, and dynamically renders a styled, interactive HTML documentation page using Bootstrap. The result is a central, up-to-date API portal accessible via a webhook endpoint.

**Target Use Cases:**  
- Automatically generate and maintain API documentation for n8n webhook endpoints without manual updates.  
- Provide developers and stakeholders with a live, user-friendly documentation portal reflecting the current state of webhook workflows.  
- Facilitate API discovery and testing with generated cURL commands and request/response schemas.

**Logical Blocks:**  
- **1.1 Input Reception:** Receives the HTTP request to serve the documentation portal.  
- **1.2 Configuration Setup:** Defines global documentation settings like title and version.  
- **1.3 Workflow Retrieval and Aggregation:** Retrieves all active workflows and aggregates their data.  
- **1.4 Documentation Extraction and Filtering:** Filters workflows to identify those with webhook endpoints and attached documentation metadata (API_DOCS).  
- **1.5 Documentation HTML Generation:** Calls a sub-workflow to generate the detailed HTML content for the API documentation accordion.  
- **1.6 Final HTML Assembly and Response:** Combines all parts into a complete HTML page with Bootstrap styling and returns it as the HTTP response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception  
**Overview:**  
This block listens for incoming HTTP requests on the `/api-doc` path and triggers the workflow to generate and serve the API documentation page.

**Nodes Involved:**  
- `Webhook`  
- `Configs`

**Node Details:**  
- **Webhook**  
  - Type: Webhook Trigger  
  - Configuration: Path set to `api-doc`, response mode configured to wait for a response node output.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to `Configs` node.  
  - Edge Cases: Potential issues include misconfigured webhook path or access restrictions.  
- **Configs**  
  - Type: Set Node  
  - Configuration: Assigns global documentation parameters: `name_doc` ("N8N Example Doc"), `version` ("v.0.1"), `description` ("Example description"), and `url_base` ("n8n.io/").  
  - Inputs: From `Webhook` node  
  - Outputs: Connects to `ExecuteSubWorkflow` (in next block)  
  - Edge Cases: Incorrect configuration here affects global documentation appearance.

---

#### 1.2 Configuration Setup & Sub-Workflow Execution  
**Overview:**  
Executes a separate sub-workflow responsible for generating the core HTML content of the documentation page, passing the base URL as input.

**Nodes Involved:**  
- `ExecuteSubWorkflow`

**Node Details:**  
- **ExecuteSubWorkflow**  
  - Type: Execute Workflow Node  
  - Configuration: Calls sub-workflow with ID `y2xOdk7RyRoS2sgF` (named "Documentation"). Waits for the sub-workflow to complete before continuing. Passes `url_base` from `Configs` node as input.  
  - Inputs: From `Configs` node  
  - Outputs: Connects to `MakeFullHTML` node  
  - Edge Cases: Sub-workflow failure or timeout; configured to continue on error but may cause incomplete documentation.  
  - Version: Requires n8n version supporting `waitForSubWorkflow` parameter.

---

#### 1.3 Workflow Retrieval and Aggregation  
**Overview:**  
Retrieves all active workflows from the n8n instance via API and aggregates their node data for filtering.

**Nodes Involved:**  
- `Execute`  
- `GetWorkflows`  
- `Aggregate`

**Node Details:**  
- **Execute**  
  - Type: Execute Workflow Trigger (trigger for sub-workflow)  
  - Configuration: Accepts input parameters, including `url_base`.  
  - Inputs: None (trigger)  
  - Outputs: Connects to `GetWorkflows` node.  
- **GetWorkflows**  
  - Type: n8n API Node  
  - Configuration: Fetches all active workflows, filtered by `activeWorkflows: true`. Uses stored `n8nApi` credentials with appropriate API permissions.  
  - Inputs: From `Execute`  
  - Outputs: Connects to `Aggregate`  
  - Edge Cases: API permission errors, network issues, or rate limits.  
- **Aggregate**  
  - Type: Aggregate Node  
  - Configuration: Aggregates all returned workflow items into a single data structure for downstream processing.  
  - Inputs: From `GetWorkflows`  
  - Outputs: Connects to `FilterWorkflows` node in next block.

---

#### 1.4 Documentation Extraction and Filtering  
**Overview:**  
Filters aggregated workflows to identify those containing webhook nodes alongside an `API_DOCS` set node, extracts JSON documentation metadata, and generates HTML accordion content for each documented endpoint.

**Nodes Involved:**  
- `FilterWorkflows`

**Node Details:**  
- **FilterWorkflows**  
  - Type: Code Node (JavaScript)  
  - Configuration:  
    - Filters workflows to those containing both webhook nodes and a `Set` node named `API_DOCS`.  
    - Parses the JSON metadata from `API_DOCS` nodes.  
    - Locates associated webhook nodes by name.  
    - Constructs Bootstrap accordion HTML for each endpoint, including method badges, endpoint paths, summaries, descriptions, cURL commands, request bodies, and success/error responses.  
    - Uses `url_base` from the `Execute` trigger node or defaults to `https://n8n.io/`.  
  - Inputs: From `Aggregate`  
  - Outputs: Returns one JSON with `htmlContent` property containing the generated accordion HTML.  
  - Edge Cases: JSON parse errors, missing nodes, or malformed metadata may cause partial or no output. Logs errors to console.  
  - Important: Generates the core documentation HTML fragment used downstream.

---

#### 1.5 Final HTML Assembly  
**Overview:**  
Wraps the accordion HTML content into a full HTML document using Bootstrap, includes custom dark theme styling, and provides footer credit. Handles fallback content if documentation generation failed.

**Nodes Involved:**  
- `MakeFullHTML`  
- `RespondHTML`

**Node Details:**  
- **MakeFullHTML**  
  - Type: HTML Node  
  - Configuration:  
    - Constructs a full HTML page embedding the accordion content via `{{$json.htmlContent}}`.  
    - Uses Bootstrap 5 CSS and icons from CDN.  
    - Applies CSS custom properties to create a dark theme.  
    - Includes a footer with author credit and LinkedIn link.  
    - Includes a fallback alert if no HTML content is generated.  
  - Inputs: From `ExecuteSubWorkflow` (which outputs the accordion HTML)  
  - Outputs: Connects to `RespondHTML`  
  - Edge Cases: If `htmlContent` is missing or empty, fallback alert is shown.  
- **RespondHTML**  
  - Type: Respond to Webhook Node  
  - Configuration: Responds with HTTP body as the generated HTML string.  
  - Inputs: From `MakeFullHTML`  
  - Outputs: End of workflow (response sent to client).  
  - Edge Cases: Response errors if HTML content is malformed.

---

#### 1.6 Auxiliary and Testing Nodes  
**Overview:**  
Contains nodes to test the webhook and provide example API_DOCS metadata. Includes sticky notes with documentation and design comments.

**Nodes Involved:**  
- `Webhook1`  
- `API_DOCS`  
- `FakeResponse`  
- `Respond`  
- Sticky Notes: `Sticky Note`, `Sticky Note1`, `Sticky Note2`, `Sticky Note3`

**Node Details:**  
- **Webhook1**  
  - Type: Webhook Trigger  
  - Configuration: Path set to unique ID for testing, response mode to respond node.  
  - Outputs: Connects to `FakeResponse`.  
- **API_DOCS**  
  - Type: Set Node  
  - Configuration: Contains raw JSON output with detailed metadata describing the test webhook endpoint (summary, method, description, request and response samples).  
  - Inputs: From `Webhook1`  
  - Outputs: None (metadata node used only for documentation discovery).  
- **FakeResponse**  
  - Type: Code Node  
  - Configuration: Returns a sample JSON success message mimicking a workflow start response.  
  - Inputs: From `Webhook1`  
  - Outputs: Connects to `Respond`.  
- **Respond**  
  - Type: Respond to Webhook Node  
  - Configuration: Returns incoming items as response, used for the test webhook.  
  - Inputs: From `FakeResponse`  
- **Sticky Notes**  
  - Provide author credits, workflow summary, and setup instructions.  
  - Includes a note with the author LinkedIn link and branding.  
  - One note contains an icon image referencing documentation page creation.

---

### 3. Summary Table

| Node Name        | Node Type                 | Functional Role                       | Input Node(s)           | Output Node(s)       | Sticky Note                                                                                                                    |
|------------------|---------------------------|------------------------------------|------------------------|----------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Webhook          | Webhook Trigger           | Entry point for API documentation   | None                   | Configs               | ## webhook for testing                                                                                                         |
| Configs          | Set Node                  | Define global documentation settings| Webhook                | ExecuteSubWorkflow    |                                                                                                                               |
| ExecuteSubWorkflow| Execute Workflow          | Run sub-workflow generating HTML    | Configs                 | MakeFullHTML          |                                                                                                                               |
| MakeFullHTML     | HTML Node                 | Build full HTML page with Bootstrap | ExecuteSubWorkflow      | RespondHTML           |                                                                                                                               |
| RespondHTML      | Respond to Webhook        | Return full HTML page as HTTP response| MakeFullHTML          | None                 |                                                                                                                               |
| Execute          | Execute Workflow Trigger  | Start of sub-workflow to get workflows| None                 | GetWorkflows          |                                                                                                                               |
| GetWorkflows     | n8n API Node              | Retrieve all active workflows        | Execute                 | Aggregate             |                                                                                                                               |
| Aggregate        | Aggregate Node            | Aggregate all workflow data          | GetWorkflows            | FilterWorkflows       | ## Get Workflow and filter                                                                                                    |
| FilterWorkflows  | Code Node                 | Filter workflows and generate accordion HTML| Aggregate          | ExecuteSubWorkflow (via ExecuteSubWorkflow output) |                                                                                                                               |
| Webhook1         | Webhook Trigger           | Test webhook endpoint                | None                   | FakeResponse          | ## webhook for testing                                                                                                         |
| API_DOCS         | Set Node                  | Provide JSON metadata for docs       | Webhook1                | None                 |                                                                                                                               |
| FakeResponse     | Code Node                 | Return sample success JSON response  | Webhook1                | Respond               |                                                                                                                               |
| Respond          | Respond to Webhook        | Respond to test webhook              | FakeResponse            | None                 |                                                                                                                               |
| Sticky Note      | Sticky Note               | Documentation comment                | None                   | None                 | ## Get Workflow and filter                                                                                                    |
| Sticky Note1     | Sticky Note               | Documentation comment                | None                   | None                 | ## webhook for testing                                                                                                         |
| Sticky Note2     | Sticky Note               | Workflow overview and author info   | None                   | None                 | ### Live n8n API Documentation Generator\n\n**Author:** Matheus Pedrosa | https://www.linkedin.com/in/matheus-pedrosa-custodio/\n**Version:** 1.0\n**Date:** 2025-08-21\n\n## Summary\nThis is a "meta-workflow" that acts as an engine to automatically generate a live, interactive HTML documentation page for your n8n instance's webhooks. It scans all active workflows, finds the ones designated for documentation, and renders a single, professional, dark-themed page.\n\n## Key Feature\nThe core logic relies on a **convention-based discovery**. To include a webhook in the documentation, you simply add a `Set` node named **`API_DOCS`** to that workflow and fill in a JSON object with the endpoint's metadata (summary, request body, responses, etc.). This generator then automatically parses that data to build the documentation page.\n\n## Setup (Quick Checklist)\n1.  **`API_DOCS` Nodes:** In every workflow you want to document, add a `Set` node named `API_DOCS` and provide the necessary JSON metadata (especially the `webhookPath`).\n2.  **`GetWorkflows` Node:** Configure this node with your `n8n API` credentials. It needs permission to read all workflows.\n3.  **`Configs` Node:** Customize the global settings for your documentation page, such as the main title (`name_doc`), description, and version.\n4.  **`Webhook` Trigger:** The URL of this workflow's trigger is the public URL for your final documentation page. |
| Sticky Note3     | Sticky Note               | Visual icon for documentation page  | None                   | None                 | ### Create documentation page\n\n![](https://cdn-icons-png.flaticon.com/512/4028/4028647.png)                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the main webhook trigger node:**  
   - Add a Webhook node named `Webhook`.  
   - Set the path to `api-doc`.  
   - Set Response Mode to `Response Node` (waits for explicit response).  
   - Connect its output to the next node.

2. **Create the global configuration node:**  
   - Add a Set node named `Configs`.  
   - Add key-value pairs:  
     - `name_doc`: string, e.g., "N8N Example Doc"  
     - `version`: string, e.g., "v.0.1"  
     - `description`: string, e.g., "Example description"  
     - `url_base`: string, e.g., "n8n.io/"  
   - Connect input from `Webhook`, output to the sub-workflow executor.

3. **Create Execute Workflow node for documentation generation:**  
   - Add an Execute Workflow node named `ExecuteSubWorkflow`.  
   - Configure it to call the sub-workflow with ID `y2xOdk7RyRoS2sgF` (this is the documentation builder).  
   - Enable "Wait for Sub-Workflow to Finish."  
   - Map `url_base` from `Configs` node JSON.  
   - Connect input from `Configs`, output to HTML builder node.

4. **Create the HTML builder node:**  
   - Add an HTML node named `MakeFullHTML`.  
   - Paste the full HTML template (as in the workflow), inserting dynamic expressions where indicated (`{{$('Configs').item.json.name_doc}}`, `{{$json.htmlContent}}`, etc.).  
   - Connect input from `ExecuteSubWorkflow`, output to the respond node.

5. **Create the final respond node:**  
   - Add a Respond to Webhook node named `RespondHTML`.  
   - Set `Respond With` to `Text`.  
   - Set `Response Body` to expression `{{$json.html}}` from previous node.  
   - Connect input from `MakeFullHTML`. This node will send the final HTML page back to the client.

6. **Build the sub-workflow (`y2xOdk7RyRoS2sgF`) to generate documentation content:**  
   - Create an Execute Workflow Trigger node named `Execute`.  
   - Add a node `GetWorkflows` using the n8n API credentials (ensure credentials are set up).  
     - Configure to filter active workflows only.  
   - Connect `GetWorkflows` output to an Aggregate node named `Aggregate` that aggregates all workflow items.  
   - Connect `Aggregate` output to a Code node named `FilterWorkflows`.  
     - Paste the JavaScript code filtering workflows with webhook + `API_DOCS` nodes and building the accordion HTML.  
   - Return the HTML content in a JSON property, e.g., `htmlContent`.  
   - Connect `FilterWorkflows` output back to the main workflow's `ExecuteSubWorkflow` node.

7. **Create a test webhook workflow for demonstration:**  
   - Add a Webhook node named `Webhook1` with a unique path (UUID or descriptive).  
   - Connect to a Set node `API_DOCS` with raw JSON output describing the documentation metadata (summary, method, requestBody, successResponse, errorResponse).  
   - Connect to a Code node `FakeResponse` returning a sample JSON success response.  
   - Connect to a Respond to Webhook node `Respond` returning the same response.

8. **Add sticky notes for documentation and user guidance:**  
   - Add sticky notes with workflow overview, usage instructions, author info, and any visual aids as per the original workflow.

9. **Credential Setup:**  
   - Configure the `GetWorkflows` node with valid n8n API credentials that have permission to read workflows.  
   - Ensure the sub-workflow ID is accessible and the sub-workflow is active.  
   - Confirm all webhook nodes are active and accessible externally if needed.

10. **Test the workflow:**  
    - Activate all workflows involved.  
    - Access the `/api-doc` endpoint of the main webhook URL to see the generated live API documentation page.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow was authored by Matheus Pedrosa. Connect with him on LinkedIn: https://www.linkedin.com/in/matheus-pedrosa-custodio/                                                                                              | Author credit and professional profile                                                              |
| The workflow uses Bootstrap 5 and Bootstrap Icons from CDN to style the documentation page with a modern, responsive, dark theme.                                                                                              | Styling resources                                                                                     |
| To document a webhook endpoint automatically, add a `Set` node named `API_DOCS` to that workflow with a JSON object specifying metadata such as summary, description, method, request and response schemas.                       | Documentation metadata convention                                                                   |
| The sub-workflow ID `y2xOdk7RyRoS2sgF` is critical as it contains the logic to parse workflows and generate the main HTML content. Replace it with your own if customizing or replicating.                                        | Sub-workflow dependency                                                                              |
| The main documentation page endpoint is accessible at `/api-doc` relative to your n8n public URL.                                                                                                                             | Access instruction                                                                                   |
| Error handling includes fallback HTML content if the documentation generation fails, providing guidance for troubleshooting common issues such as missing `API_DOCS` nodes or inactive workflows.                                | Error resilience and user feedback                                                                  |
| ![](https://cdn-icons-png.flaticon.com/512/4028/4028647.png) — used in sticky note as an icon for documentation page creation.                                                                                                  | Visual aid                                                                                           |

---

**Disclaimer:** The provided description and analysis are derived solely from the n8n workflow JSON and comply with all applicable content policies. All data processed is legal and publicly available.