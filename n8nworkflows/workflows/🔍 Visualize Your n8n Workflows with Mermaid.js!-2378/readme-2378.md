🔍 Visualize Your n8n Workflows with Mermaid.js!

https://n8nworkflows.xyz/workflows/---visualize-your-n8n-workflows-with-mermaid-js--2378


# 🔍 Visualize Your n8n Workflows with Mermaid.js!

### 1. Workflow Overview

This workflow, titled **"Workflow dashboard with mermaid.js"**, serves to visualize n8n workflows as interactive flowcharts rendered with Mermaid.js. It targets users who prefer visual representations of automation processes for documentation, presentations, or better comprehension of workflow structures.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Routing:** Receives HTTP requests via webhook and routes them according to query parameters.
- **1.2 Workflows Data Retrieval:** Fetches workflow lists or a single workflow’s details from the n8n instance using the n8n API node.
- **1.3 Data Aggregation & Preparation:** Aggregates and prepares workflow data for visualization.
- **1.4 Mermaid Diagram Generation:** Converts workflow nodes and connections into a Mermaid.js flowchart syntax string.
- **1.5 Web Response Delivery:** Responds to HTTP requests either with an HTML page listing workflows or with Mermaid.js chart code.
- **1.6 Configuration & Notes:** Holds environment-specific configuration settings and notes for cloud users.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Routing

**Overview:**  
Captures incoming HTTP requests and determines whether to display the main page or generate a Mermaid.js chart for a specific workflow based on query parameters.

**Nodes Involved:**  
- Webhook  
- Switch  
- When clicking ‘Test workflow’ (manual trigger)

**Node Details:**

- **Webhook**  
  - *Type:* Webhook (HTTP trigger)  
  - *Role:* Entry point for HTTP requests to the workflow.  
  - *Configuration:* Path set to a UUID; responds via response node.  
  - *Input/Output:* No input; outputs incoming HTTP request data.  
  - *Failures:* Timeout, invalid requests, missing query parameters.  

- **Switch**  
  - *Type:* Switch (conditional routing)  
  - *Role:* Routes flow based on query parameters in incoming request.  
  - *Configuration:*  
    - Two outputs:  
      - "load page" when no query parameters or query keys are empty.  
      - "has wfid" when query contains `wfid` (workflow ID).  
  - *Input:* Webhook node output.  
  - *Output:*  
    - To "List workflows" node if loading page.  
    - To "Single workflow" node if `wfid` present.  
  - *Edge Cases:* Expression evaluation failure if request data malformed.  

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual testing of the workflow; connected to Switch node.  
  - *Input/Output:* No input; triggers Switch node.  
  - *Use:* Useful during development/testing.  

---

#### 2.2 Workflows Data Retrieval

**Overview:**  
Fetches either the list of all workflows or details of a single workflow via the n8n API node, using stored credentials.

**Nodes Involved:**  
- List workflows  
- Single workflow

**Node Details:**

- **List workflows**  
  - *Type:* n8n API (resource: workflows)  
  - *Role:* Retrieves all workflows from the n8n instance.  
  - *Configuration:* No filters, default options.  
  - *Credentials:* Uses "Ted n8n account" API credential.  
  - *Input:* From Switch "load page" output.  
  - *Output:* Sends workflows data to "Prepare workflow list".  
  - *Failures:* API auth errors, network issues.  

- **Single workflow**  
  - *Type:* n8n API (resource: workflows)  
  - *Role:* Retrieves a specific workflow by ID from query parameter `wfid`.  
  - *Configuration:* Operation "get" with dynamic workflow ID from query.  
  - *Credentials:* Same as above.  
  - *Input:* From Switch "has wfid" output.  
  - *Output:* Sends single workflow JSON to "Code" node.  
  - *Failures:* Invalid workflow ID, API auth, network issues.  

---

#### 2.3 Data Aggregation & Preparation

**Overview:**  
Processes the retrieved workflows data, extracting essential information and aggregating it for further processing.

**Nodes Involved:**  
- Prepare workflow list  
- Aggregate  
- CONFIG

**Node Details:**

- **Prepare workflow list**  
  - *Type:* Set  
  - *Role:* Transforms each workflow item to a simplified object containing only `id` and `name`.  
  - *Input:* From List workflows output.  
  - *Output:* Passes simplified list to Aggregate node.  

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Aggregates the simplified workflow objects into a single array under `wf_data`.  
  - *Input:* From Prepare workflow list.  
  - *Output:* Passes aggregated data to CONFIG node.  

- **CONFIG**  
  - *Type:* Set  
  - *Role:* Sets configuration variables such as `instance_url`, `webhook_name`, and `webhook_path`.  
  - *Parameters:*  
    - `instance_url`: Uses environment variables for protocol and host.  
    - `webhook_name`: Derived dynamically from the Webhook node’s path parameter.  
    - `webhook_path`: Environment variable or default "webhook".  
  - *Input:* From Aggregate node.  
  - *Output:* Passes configuration and workflows data to Send Page node.  
  - *Edge Cases:* Environment variables missing in cloud environment; sticky note includes instructions.  

---

#### 2.4 Mermaid Diagram Generation

**Overview:**  
Transforms the JSON representation of a workflow into a Mermaid.js flowchart syntax string, applying special shapes and formatting disabled nodes with strikethrough.

**Nodes Involved:**  
- Code

**Node Details:**

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:*  
    - Parses workflow nodes and connections.  
    - Filters out "stickyNote" nodes.  
    - Maps node types to Mermaid shapes (rectangles, rhombus, circles, etc.).  
    - Applies strikethrough formatting for disabled nodes.  
    - Builds Mermaid flowchart notation with directional arrows and connection types (main or others).  
    - Handles isolated nodes without connections.  
    - Returns Mermaid chart string in JSON property `mermaidChart`.  
  - *Input:* Single workflow JSON from Single workflow node.  
  - *Output:* Mermaid.js code string to "Respond with Mermaid" node.  
  - *Potential Failures:*  
    - Malformed workflow JSON.  
    - Unsupported node types not mapped.  
    - Expression or code runtime errors.  

---

#### 2.5 Web Response Delivery

**Overview:**  
Sends back either an HTML page listing all workflows with buttons to view flowcharts or the Mermaid.js flowchart text for an individual workflow.

**Nodes Involved:**  
- Send Page  
- Respond with Mermaid

**Node Details:**

- **Send Page**  
  - *Type:* Respond to Webhook  
  - *Role:*  
    - Sends a complete HTML page with embedded Bootstrap 5 and Mermaid.js scripts.  
    - Lists all workflows with clickable buttons to fetch and display Mermaid diagrams on the page dynamically.  
    - Includes styling and links to author’s profiles and blog posts.  
  - *Input:* From CONFIG node.  
  - *Output:* HTTP response HTML.  
  - *Edge Cases:* Large workflow list might slow page load.  

- **Respond with Mermaid**  
  - *Type:* Respond to Webhook  
  - *Role:* Responds with plain text containing Mermaid.js flowchart code.  
  - *Input:* From Code node.  
  - *Output:* HTTP response with Content-Type text/plain.  
  - *Failures:* If Code node fails, response will be missing or error.  

---

#### 2.6 Configuration & Notes

**Overview:**  
Holds static notes and instructions for cloud users and environment-specific configuration.

**Nodes Involved:**  
- Sticky Note3

**Node Details:**

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Provides important instructions for cloud users to update environment-dependent fields manually because cloud does not support environment variables.  
  - *Content Summary:*  
    - Update `instance_url` to actual cloud URL instead of environment variables.  
    - Change `webhook_path` to literal "webhook" instead of environment variable expression.  
  - *Input/Output:* None (informational only).  

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                      | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                      |
|----------------------------|-----------------------|------------------------------------|-------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Webhook                    | Webhook               | Receives HTTP requests             | None                          | Switch                       |                                                                                                |
| Switch                     | Switch                | Routes based on query parameter    | Webhook, When clicking ‘Test workflow’ | List workflows, Single workflow |                                                                                                |
| When clicking ‘Test workflow’ | Manual Trigger        | Manual testing trigger              | None                          | Switch                       |                                                                                                |
| List workflows             | n8n API               | Fetches list of workflows          | Switch                        | Prepare workflow list        |                                                                                                |
| Prepare workflow list      | Set                   | Simplifies workflow data            | List workflows                | Aggregate                    |                                                                                                |
| Aggregate                 | Aggregate             | Aggregates workflow data           | Prepare workflow list         | CONFIG                       |                                                                                                |
| CONFIG                     | Set                   | Sets configuration variables       | Aggregate                    | Send Page                   | Important note for cloud users: update instance_url and webhook_path fields manually            |
| Send Page                  | Respond to Webhook    | Sends HTML page listing workflows  | CONFIG                       | None                        |                                                                                                |
| Single workflow            | n8n API               | Fetches specific workflow by ID    | Switch                        | Code                        |                                                                                                |
| Code                       | Code                  | Generates Mermaid.js flowchart code| Single workflow              | Respond with Mermaid         |                                                                                                |
| Respond with Mermaid       | Respond to Webhook    | Sends Mermaid code as plain text   | Code                         | None                        |                                                                                                |
| Sticky Note3               | Sticky Note           | Instructions for cloud users       | None                         | None                        | ## IMPORTANT NOTE FOR CLOUD USERS - See details in section 2.6                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: Use a unique identifier or UUID for webhook path (e.g., `dd9e2c5d-6c48-428e-aa54-bef9e369d3b0`)  
   - Response Mode: Response Node  
   - Position appropriately on canvas.

2. **Create Switch Node**  
   - Type: Switch  
   - Connect input from Webhook node.  
   - Add two outputs with rules:  
     - Output "load page": Condition checks if query parameters are empty or do not contain `wfid`.  
     - Output "has wfid": Condition checks if query parameters contain `wfid`.  
   - Enable loose type validation.

3. **Create Manual Trigger Node** (Optional)  
   - Name: "When clicking ‘Test workflow’"  
   - Connect output to Switch node.

4. **Create n8n API Node to List Workflows**  
   - Name: "List workflows"  
   - Resource: Workflows  
   - Operation: List (default)  
   - No filters.  
   - Attach appropriate n8n API credentials ("Ted n8n account").  
   - Connect output from Switch "load page" output.

5. **Create Set Node to Prepare Workflow List**  
   - Name: "Prepare workflow list"  
   - Set parameters: For each item, assign `wf_data` as an object with `id` and `name` properties derived from each workflow’s JSON.  
   - Connect input from List workflows node.

6. **Create Aggregate Node to Aggregate Workflows**  
   - Name: "Aggregate"  
   - Aggregate field: `wf_data` (array of simplified workflow objects)  
   - Connect input from Prepare workflow list node.

7. **Create Set Node for Configuration (CONFIG)**  
   - Name: "CONFIG"  
   - Add fields:  
     - `instance_url`: Use expression `{{$env["N8N_PROTOCOL"]}}://{{$env["N8N_HOST"]}}` (or replace with your instance URL for cloud).  
     - `webhook_name`: Expression `{{$('Webhook').params.path}}` (dynamic path from Webhook node).  
     - `webhook_path`: Expression `{{$env["N8N_ENDPOINT_WEBHOOK"] || "webhook"}}` (or "webhook" for cloud).  
   - Connect input from Aggregate node.

8. **Create Respond to Webhook Node for Sending HTML Page**  
   - Name: "Send Page"  
   - Respond With: Text  
   - Response Body: Paste the full provided HTML content (including Bootstrap 5, Mermaid.js scripts, and custom JS to render workflows).  
   - Connect input from CONFIG node.

9. **Create n8n API Node to Get Single Workflow**  
   - Name: "Single workflow"  
   - Operation: Get  
   - Workflow ID: Use expression `={{ $json.query.wfid }}` (dynamic from query parameter).  
   - Attach appropriate credentials.  
   - Connect input from Switch "has wfid" output.

10. **Create Code Node for Mermaid Diagram Generation**  
    - Name: "Code"  
    - Paste the JavaScript code that:  
      - Extracts nodes and connections, filters out sticky notes.  
      - Maps node types to Mermaid shapes.  
      - Formats disabled nodes with strikethrough.  
      - Builds Mermaid flowchart syntax string.  
      - Returns `mermaidChart` string in JSON output.  
    - Connect input from Single workflow node.

11. **Create Respond to Webhook Node to Return Mermaid Code**  
    - Name: "Respond with Mermaid"  
    - Respond With: Text  
    - Response Headers: Set `Content-Type` to `text/plain`.  
    - Response Body: Expression `={{ $json.mermaidChart }}` from Code node output.  
    - Connect input from Code node.

12. **Create Sticky Note Node for Cloud Users**  
    - Name: "Sticky Note3"  
    - Content: Instructions to update `instance_url` and `webhook_path` for cloud environment (replace environment variables with hardcoded values).  
    - Position near CONFIG node for visibility.

13. **Connect Nodes According to the connections outlined in the workflow:**  
    - Webhook → Switch  
    - Switch "load page" → List workflows → Prepare workflow list → Aggregate → CONFIG → Send Page  
    - Switch "has wfid" → Single workflow → Code → Respond with Mermaid  
    - Manual Trigger → Switch (for testing)

14. **Credentials Setup:**  
    - Set up n8n API credentials named "Ted n8n account", configured with appropriate permissions to access workflows.  
    - Ensure environment variables for `N8N_PROTOCOL`, `N8N_HOST`, and `N8N_ENDPOINT_WEBHOOK` exist or replace them with static values if using the cloud.

15. **Test the Workflow:**  
    - Trigger via webhook URL with or without `wfid` query parameter to test both main page rendering and Mermaid diagram generation.  
    - Use manual trigger for testing routing logic.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Important note for cloud users: environment variables are not supported; update `instance_url` and `webhook_path` manually in CONFIG node. | See Sticky Note3 content in workflow; also explained in description section.                        |
| Template built with Mermaid.js, Bootstrap 5, and AJAX to render interactive workflow diagrams.     | Workflow description and embedded HTML page code.                                                  |
| Contact Eduard for customization assistance.                                                       | [LinkedIn](https://www.linkedin.com/in/parsadanyan/)                                              |
| Live examples include multiple flowcharts on a page, special shapes for different node types, and Langchain nodes with styled connections. | Displayed in workflow description images (not embedded here).                                      |
| The HTML page includes author info, links to more templates, and blog post fetching for additional content. | HTML contains scripts pulling blog posts from https://blog.n8n.io/rss/                              |

---

This document provides a structured and detailed reference to understand, recreate, and maintain the "Workflow dashboard with mermaid.js" n8n workflow, facilitating both human and AI agent consumption.