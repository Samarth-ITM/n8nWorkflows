Get workflows affected by 0.214.3 migration

https://n8nworkflows.xyz/workflows/get-workflows-affected-by-0-214-3-migration-1892


# Get workflows affected by 0.214.3 migration

### 1. Workflow Overview

This workflow is designed to identify n8n workflows potentially affected by a known migration issue introduced in version `0.214.3`. The migration bug caused nodes with multiple outputs—such as `If`, `Switch`, and `Compare Datasets`—to be rewired incorrectly, potentially breaking workflow logic.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives a webhook request to trigger the scan.
- **1.2 Workflow Retrieval:** Fetches all workflows from the n8n instance via the API.
- **1.3 Workflow Analysis:** Parses each workflow's node connections to detect miswired multi-output nodes.
- **1.4 Report Generation and Delivery:** Creates an HTML report listing affected workflows and serves it as the webhook response.
- **1.5 User Guidance and Instructions:** Provides detailed usage notes and operational guidance via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for a webhook call at the path `/affected-workflows` to start the detection process.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (trigger node)  
    - Configuration:  
      - Webhook path: `affected-workflows`  
      - Returns response in `responseNode` mode  
      - Sets HTTP response header `Content-Type` to `text/html; charset=utf-8`  
    - Input connections: None (trigger node)  
    - Output connections: Passes execution to `Get all workflows`  
    - Potential failures: Webhook URL conflicts, permission issues if instance owner rights are missing  
    - Version requirements: None specific  
    - Notes: Must be triggered by the instance owner for full API access  

#### 1.2 Workflow Retrieval

- **Overview:**  
  Retrieves the list of all workflows from the n8n instance through the internal API using an API credential.

- **Nodes Involved:**  
  - `Get all workflows`

- **Node Details:**  
  - **Get all workflows**  
    - Type: n8n (API) node to call n8n internal API  
    - Configuration:  
      - No filters applied; fetches all workflows  
      - Uses n8n API credential (configured with API key)  
    - Input connections: Receives trigger from `Webhook`  
    - Output connections: Sends workflows data to `Parse potentially affected workflows`  
    - Potential failures: Authentication failures if API key is invalid or missing; API rate limits or connectivity issues  
    - Notes: Sticky note instructs to configure this node with valid n8n API credentials  

#### 1.3 Workflow Analysis

- **Overview:**  
  Analyzes each workflow's node connections, focusing on multi-output nodes, to detect any workflows that have potentially been rewired incorrectly due to the migration bug.

- **Nodes Involved:**  
  - `Parse potentially affected workflows`

- **Node Details:**  
  - **Parse potentially affected workflows**  
    - Type: Code (JavaScript) node  
    - Configuration:  
      - Defines a constant `MULTI_OUTPUT_NODES` listing node types with multiple outputs and expected output counts (`compareDatasets` with 4, `switch` with 4, and `if` with 2).  
      - Loops through each workflow's nodes and connections. For each multi-output node, checks if the number of connected outputs matches the expected count.  
      - Workflows with fewer connected outputs than expected are flagged as potentially affected.  
      - Returns an object with an array of affected workflows, including workflow ID, name, active status, and affected node names.  
    - Input connections: Receives all workflows data from `Get all workflows`  
    - Output connections: Sends parsed results to `Generate Report`  
    - Potential failure points:  
      - If community nodes with multiple outputs exist but are not added to `MULTI_OUTPUT_NODES`, false negatives can occur.  
      - Errors in data structure or missing keys may cause runtime errors.  
      - Edge case: Workflows with no multi-output nodes or no connections are skipped.  
    - Notes: One sticky note advises to add any community nodes with multiple outputs to the `MULTI_OUTPUT_NODES` array.

#### 1.4 Report Generation and Delivery

- **Overview:**  
  Generates an HTML report listing all affected workflows and serves it as the HTTP response to the webhook caller.

- **Nodes Involved:**  
  - `Generate Report`  
  - `Serve HTML Report`

- **Node Details:**  
  - **Generate Report**  
    - Type: HTML node  
    - Configuration:  
      - Contains static HTML skeleton for the report page, including styles and a container with a placeholder `<ul>` list element.  
    - Input connections: Receives JSON data from `Parse potentially affected workflows` (though not used directly in HTML)  
    - Output connections: Passes HTML to `Serve HTML Report`  
    - Potential failures: Minimal; mostly static content.  
  - **Serve HTML Report**  
    - Type: Respond to Webhook node  
    - Configuration:  
      - Sets response header `Content-Type` to `text/html; charset=utf-8`  
      - Responds with the HTML content from `Generate Report` node combined with a `<script>` block  
      - The embedded script dynamically reads the JSON data from `Parse potentially affected workflows` node output and populates the `<ul>` with links to affected workflows. Each list item includes workflow ID, name, and affected nodes.  
    - Input connections: Receives HTML content from `Generate Report`  
    - Output connections: None (end of chain)  
    - Potential failures: Client-side JavaScript errors if JSON data is malformed; broken links if instance URL is incorrect or workflows do not exist.  
    - Notes: Sticky note indicates how to access the report URL on the instance.

#### 1.5 User Guidance and Instructions

- **Overview:**  
  Provides detailed operational instructions and warnings to users regarding the workflow’s purpose and usage.

- **Nodes Involved:**  
  - `Sticky Note`  
  - `Sticky Note1`  
  - `Sticky Note2`  
  - `Sticky Note3`

- **Node Details:**  
  - All sticky notes contain markdown-formatted text instructing users:  
    - To run the workflow as the instance owner  
    - To configure the API key credential for the `Get all workflows` node  
    - To add any community multi-output nodes to the analysis list in the code node  
    - How to activate the workflow and access the generated report via URL  
    - What to check in the report (correct outbound connectors)  
  - These are purely informational and do not affect workflow execution.

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                      | Input Node(s)        | Output Node(s)                     | Sticky Note                                                                                                 |
|-------------------------------|----------------------|------------------------------------|----------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Webhook                      | Webhook              | Entry trigger for scan             | None                 | Get all workflows                |                                                                                                             |
| Get all workflows            | n8n API node         | Fetch all workflows from instance  | Webhook              | Parse potentially affected workflows | Configure this node to use your n8n API credential                                                           |
| Parse potentially affected workflows | Code (JavaScript)    | Analyze workflows for migration issues | Get all workflows    | Generate Report                 | In case you have community nodes installed, add them to `MULTI_OUTPUT_NODES`                                |
| Generate Report              | HTML                 | Generates HTML report skeleton     | Parse potentially affected workflows | Serve HTML Report             |                                                                                                             |
| Serve HTML Report            | Respond to Webhook   | Sends generated HTML report as HTTP response | Generate Report      | None                             | Find the generated report at `{YOUR_INSTANCE_URL}/webhooks/affected-workflows`                              |
| Sticky Note                 | Sticky Note          | User instruction and guidance      | None                 | None                            | # ⚠️ When and how to use this workflow [...] **❗️Please ensure to run this workflow as the instance owner❗️** |
| Sticky Note1                | Sticky Note          | Reminder about community nodes     | None                 | None                            | # 👆 In case you have community nodes installed, add them to `MULTI_OUTPUT_NODES`                            |
| Sticky Note2                | Sticky Note          | Reminder to configure API key      | None                 | None                            | # 👆 Configure this node to use your n8n API credential                                                      |
| Sticky Note3                | Sticky Note          | Report access instructions         | None                 | None                            | # 👆 Find the generated report at `{YOUR_INSTANCE_URL}/webhooks/affected-workflows`                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook`  
   - Path: `affected-workflows`  
   - Response Mode: `Response Node`  
   - Set response header `Content-Type` to `text/html; charset=utf-8`  
   - This node is the trigger for the workflow.

2. **Create an n8n API Node to Get Workflows**  
   - Type: n8n (API) node  
   - Name: `Get all workflows`  
   - No filters configured (empty) to fetch all workflows  
   - Set Credentials: Choose or create an n8n API credential using an API key from `Settings > n8n API`  
   - Connect the output of `Webhook` node to this node.

3. **Create a Code Node to Parse Workflows**  
   - Type: Code (JavaScript) node  
   - Name: `Parse potentially affected workflows`  
   - Paste the following JavaScript code (adapt as needed for additional multi-output nodes):
     ```javascript
     const MULTI_OUTPUT_NODES = [
       { type: 'n8n-nodes-base.compareDatasets', outputs: 4 },
       { type: 'n8n-nodes-base.switch', outputs: 4 },
       { type: 'n8n-nodes-base.if', outputs: 2 }
     ];

     const affectedWorkflows = [];

     for (const item of $input.all()) {
       const workflowData = item.json;
       const nodes = workflowData.nodes;
       const connections = workflowData.connections;
       const potentiallyAffectedNodes = [];

       for (const connectionName of Object.keys(connections)) {
         const connection = connections[connectionName];
         const connectionNode = nodes.find(node => node.name === connectionName);
         const matchedMultiOutputNode = MULTI_OUTPUT_NODES.find(n => n.type === connectionNode.type);
         if (matchedMultiOutputNode) {
           const connectedOutputs = connection.main.filter(c => c && c.length > 0);
           if (connectedOutputs.length === 0) continue;
           if (connectedOutputs.length < matchedMultiOutputNode.outputs) {
             potentiallyAffectedNodes.push(connectionName);
           }
         }
       }

       if (potentiallyAffectedNodes.length > 0) {
         affectedWorkflows.push({
           workflowId: workflowData.id,
           workflowName: workflowData.name,
           active: workflowData.active,
           potentiallyAffectedNodes
         });
       }
     }

     return { workflows: affectedWorkflows };
     ```
   - Connect output of `Get all workflows` to this node.

4. **Create an HTML Node to Generate Report Skeleton**  
   - Type: HTML  
   - Name: `Generate Report`  
   - Paste the HTML content for the report page:
     ```html
     <!DOCTYPE html>
     <html>
     <head>
       <meta charset="UTF-8" />
       <title>n8n workflows report</title>
     </head>
     <body>
       <div class="container">
         <h1>Affected workflows:</h1>
         <ul id="list"></ul>
       </div>
     </body>
     </html>

     <style>
     .container {
       background-color: #ffffff;
       text-align: center;
       padding: 16px;
       border-radius: 8px;
     }
     h1 {
       color: #ff6d5a;
       font-size: 24px;
       font-weight: bold;
       padding: 8px;
     }
     h2 {
       color: #909399;
       font-size: 18px;
       font-weight: bold;
       padding: 8px;
     }
     ul {
       list-style: none;
       text-align: left;
       padding: 0;
     }
     li {
       margin: 8px 0;
     }
     a {
       color: #409eff;
       text-decoration: none;
       transition: color 0.2s ease-in-out;
     }
     a:hover {
       color: #ff9900;
     }
     </style>
     ```
   - Connect output of `Parse potentially affected workflows` to this node.

5. **Create a Respond to Webhook Node to Serve the Report**  
   - Type: Respond to Webhook  
   - Name: `Serve HTML Report`  
   - Set response header: `Content-Type` = `text/html; charset=utf-8`  
   - Set "Respond With" to `Text`  
   - For the response body, use an expression to combine the HTML with a script that populates the list dynamically:
     ```javascript
     {{$node["Generate Report"].parameter["html"]}}

     <script>
     const { workflows } = {{ JSON.stringify($node["Parse potentially affected workflows"].json) }};
     const $list = document.getElementById('list');
     workflows.forEach((workflow) => {
       if (!workflow) return;
       const title = `<a target="_blank" href="//${window.location.host}/workflow/${workflow.workflowId}">ID: ${workflow.workflowId}: ${workflow.workflowName} [${workflow.potentiallyAffectedNodes.join(', ')}]</a>`;
       const $listItem = document.createElement('li');
       $listItem.innerHTML = title;
       $list.appendChild($listItem);
     });
     </script>
     ```
   - Connect output of `Generate Report` to this node.

6. **Add Sticky Notes for Instructions (Optional but Recommended)**  
   - Add a sticky note near the `Webhook` and `Get all workflows` nodes with instructions about running as instance owner and configuring the API key.  
   - Add a sticky note near the code node reminding to add community nodes with multiple outputs to the `MULTI_OUTPUT_NODES` array.  
   - Add a sticky note near the `Serve HTML Report` node with instructions on accessing the report URL.

7. **Activate the Workflow**

8. **Usage**  
   - Access the report by visiting `{YOUR_INSTANCE_URL}/webhooks/affected-workflows` in a browser.  
   - Review the list of potentially affected workflows and nodes.  
   - Manually inspect and correct wiring in those workflows as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow specifically addresses issues introduced in n8n version `0.214.3` related to nodes with multiple outputs such as `If`, `Switch`, and `Compare Datasets`.                                                                              | Migration bug context                             |
| To use this workflow, you must run it as the instance owner because it requires access to all workflows via the internal API.                                                                                                                        | User permission requirements                      |
| If you have installed community nodes that have multiple outputs, add their node types and output counts to the `MULTI_OUTPUT_NODES` array in the Code node to ensure they are properly analyzed.                                                      | Extensibility instructions                        |
| The generated HTML report is accessible at `https://{YOUR_INSTANCE_URL}/webhooks/affected-workflows`. It provides clickable links to the affected workflows in the editor UI for easy inspection and fixing.                                          | Report access instructions                         |
| Verify the outbound connectors of affected nodes carefully, as the migration bug may have disconnected or misrouted outputs.                                                                                                                        | Post-report verification step                      |

---

This document provides a complete, detailed analysis and stepwise reconstruction guide for the workflow "Get workflows affected by 0.214.3 migration". It enables users and automation tools to understand, reproduce, and adapt the workflow confidently.