🦅 Get a bird's-eye view of your n8n instance with the Workflow Dashboard!

https://n8nworkflows.xyz/workflows/---get-a-bird-s-eye-view-of-your-n8n-instance-with-the-workflow-dashboard--2269


# 🦅 Get a bird's-eye view of your n8n instance with the Workflow Dashboard!

### 1. Workflow Overview

This workflow, titled **"🦅 Get a bird's-eye view of your n8n instance with the Workflow Dashboard!"**, is designed to provide a comprehensive dashboard overview of your entire n8n instance. It targets users who manage multiple workflows and want a centralized, visually intuitive summary of workflows, nodes, tags, and webhook endpoints without relying on third-party tools.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Trigger**: Manual or webhook triggers to start the workflow.
- **1.2 Fetching Workflows and Extracting Raw Data**: Nodes that retrieve all workflows and extract relevant arrays and metadata for each workflow.
- **1.3 Workflow Data Processing**: Aggregation, sorting, and structuring of workflows, nodes, tags, and webhooks into respective sections.
- **1.4 JSON Assembly**: Combining all processed data into a single JSON object representing the dashboard data structure.
- **1.5 XML & HTML Dashboard Generation**: Conversion of the JSON summary into XML and an HTML page styled using Bootstrap 5 and XSLT, which is then served via webhook.
- **1.6 Webhook Endpoints for Dashboard and Stylesheet**: Exposing endpoints to serve the dashboard and its XSLT template.
- **1.7 Notes & Documentation**: Embedded sticky notes explaining usage, important considerations, and references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Trigger

**Overview:**  
Provides manual and webhook triggers to start the workflow either via UI interaction or HTTP requests.

**Nodes Involved:**  
- When clicking "Test workflow" (Manual Trigger)  
- Execute Workflow Trigger (Internal trigger node)  
- Request HTML dashboard (Webhook)  
- Request xsl template (Webhook)

**Node Details:**

- **When clicking "Test workflow"**  
  - Type: Manual Trigger  
  - Role: Starts workflow manually for testing or manual refresh.  
  - Inputs: None  
  - Outputs: Triggers fetching workflows  
  - Failures: None expected; manual trigger.

- **Execute Workflow Trigger**  
  - Type: Execute Workflow Trigger  
  - Role: Supports internal workflow executions triggered by other workflows or triggers.  
  - Inputs: Triggered from manual or webhook.  
  - Outputs: Starts the workflow execution chain.  
  - Failures: None typical.

- **Request HTML dashboard**  
  - Type: Webhook (Path: `fb550a01-12f2-4709-ba2d-f71197b68340`)  
  - Role: HTTP endpoint to request the full interactive dashboard page.  
  - Inputs: HTTP GET request  
  - Outputs: Triggers the JSON preparation and dashboard generation.  
  - Failures: No authentication; public access risk.  
  - Notes: Sticky note warns about lack of protection and suggests adding authentication.

- **Request xsl template**  
  - Type: Webhook (Path: `73a91e4d-143d-4168-9efb-6c56f2258aec/dashboard.xsl`)  
  - Role: Serves the XSLT stylesheet required by the dashboard HTML for XML transformation and styling.  
  - Inputs: HTTP GET request  
  - Outputs: Serves XSL content directly.  
  - Failures: None typical.

---

#### 2.2 Fetching Workflows and Extracting Raw Data

**Overview:**  
Fetches all workflows from the n8n instance via API and extracts key attributes and arrays (nodes, tags, webhook paths) for processing.

**Nodes Involved:**  
- n8n-get-workflows  
- get-nodes-via-jmespath  
- workflows-section  
- globals-section

**Node Details:**

- **n8n-get-workflows**  
  - Type: n8n (n8n API node)  
  - Role: Retrieves all workflows from the connected n8n instance.  
  - Parameters: No filters, fetches all workflows.  
  - Credentials: Uses configured n8n API credentials (`n8n account 4`).  
  - Inputs: Trigger node output  
  - Outputs: JSON array of workflows metadata.  
  - Failures: API connection issues, auth failures.

- **get-nodes-via-jmespath**  
  - Type: Set  
  - Role: Extracts and prepares arrays and strings for each workflow:  
    - `nodes_array`: Extracts node types from workflow, transforms to uppercase node names.  
    - `tags_array`: Extracts tag names from workflow.  
    - `instance_url`: Constructs n8n instance URL from environment variables (note for cloud users to hardcode).  
    - `webhook_paths_array`: Extracts webhook paths from nodes of type webhook.  
  - Parameters: Uses JMESPath expressions and JS map functions on JSON data.  
  - Inputs: Output from n8n-get-workflows  
  - Outputs: Enriched workflow JSON with arrays and instance URL.  
  - Failures: Expression errors if workflow JSON is malformed or env vars missing.

- **workflows-section**  
  - Type: Set  
  - Role: Creates a `wf_stats` object per workflow aggregating: unique nodes, counts, formatted timestamps, workflow URLs, active status, trigger counts, tags, and webhook paths.  
  - Key Expressions: Uses JS date formatting (`Luxon`), template literals for URLs and IDs.  
  - Inputs: Enriched workflow JSON from previous node.  
  - Outputs: Workflow stats objects for processing.  
  - Failures: Date parsing errors if timestamps are invalid.

- **globals-section**  
  - Type: Set  
  - Role: Produces a global summary object with:  
    - Total number of workflows  
    - Count of active workflows (filter via JMESPath)  
    - Sum of all trigger counts (reduce function)  
  - Input: Aggregated workflow stats  
  - Output: Single global stats JSON object.  
  - Failures: None typical.

---

#### 2.3 Workflow Data Processing

**Overview:**  
Aggregates, sorts, and maps workflows by nodes, tags, and webhooks, producing summaries for each section.

**Nodes Involved:**  
- nodes-section (Code)  
- tags-section (Code)  
- webhook-section (Code)  
- Sort-workflows  
- Sort-nodes  
- Sort-tags  
- Sort-whooks  
- Aggregate-workflows  
- Aggregate-nodes  
- Aggregate-tags  
- Aggregate-whooks  
- Final-json (Merge)

**Node Details:**

- **nodes-section**  
  - Type: Code  
  - Role: Maps each unique node type to workflows that use it; counts workflows per node.  
  - Logic: Iterates workflows, builds node-to-workflows map, outputs array of node objects with counts and workflow details.  
  - Inputs: Workflow stats items.  
  - Outputs: Array of nodes with usage data.  
  - Failures: JS code failure if input data missing.

- **tags-section**  
  - Type: Code  
  - Role: Maps tags to workflows using them; handles workflows with no tags by grouping under "No Tags".  
  - Logic: Iterates workflows and tags, builds tag-to-workflows map, outputs array of tag objects with counts and workflow details.  
  - Inputs: Workflow stats items.  
  - Outputs: Array of tags with usage data.  
  - Failures: Handles empty tags arrays gracefully.

- **webhook-section**  
  - Type: Code  
  - Role: Maps webhook endpoint paths to workflows that use them; filters out empty or invalid paths.  
  - Logic: Iterates workflows and webhook paths, constructs webhook-to-workflows map with counts and workflow details.  
  - Inputs: Workflow stats items.  
  - Outputs: Array of webhook path objects.  
  - Failures: Warns on invalid webhook paths; skips missing data.

- **Sort-workflows, Sort-nodes, Sort-tags, Sort-whooks**  
  - Type: Sort  
  - Role: Sorts each section by relevant fields:  
    - Workflows: by updated date descending, then name  
    - Nodes: by usage count descending, then node name  
    - Tags: by usage count descending, then tag name  
    - Webhooks: by usage count descending, then path  
  - Inputs: Arrays from previous nodes  
  - Outputs: Sorted arrays for aggregation  
  - Failures: None typical.

- **Aggregate-workflows, Aggregate-nodes, Aggregate-tags, Aggregate-whooks**  
  - Type: Aggregate  
  - Role: Combine all items into single arrays under destination field names:  
    - workflows → `wf_stats`  
    - nodes → `nodes-section`  
    - tags → `tags-section`  
    - webhooks → `whooks-section`  
  - Inputs: Sorted arrays  
  - Outputs: Aggregated arrays for final merge  
  - Failures: None typical.

- **Final-json**  
  - Type: Merge (combine mode)  
  - Role: Combines globals, workflows, nodes, tags, and webhook aggregates into one JSON object representing the entire dashboard data.  
  - Inputs: Globals, aggregated workflows, nodes, tags, webhooks  
  - Outputs: Complete dashboard JSON object  
  - Failures: None typical.

---

#### 2.4 XML & HTML Dashboard Generation

**Overview:**  
Transforms the dashboard JSON into an XML document and then into an HTML page styled via a linked XSLT stylesheet and Bootstrap 5. The HTML is then prepared for HTTP response.

**Nodes Involved:**  
- Prepare JSON object (Execute Workflow)  
- Convert to XML  
- Create HTML  
- Move Binary Data  
- Respond to Webhook

**Node Details:**

- **Prepare JSON object**  
  - Type: Execute Workflow  
  - Role: Runs this workflow itself, effectively refreshing the JSON data.  
  - Inputs: Triggered from webhook request  
  - Outputs: JSON dashboard data for conversion  
  - Failures: Recursive calls must be monitored to avoid loops.

- **Convert to XML**  
  - Type: XML  
  - Role: Converts JSON dashboard data into XML format with `headless: true` (no XML declaration).  
  - Inputs: JSON from Prepare JSON object  
  - Outputs: XML string of dashboard data  
  - Failures: Conversion errors if JSON malformed.

- **Create HTML**  
  - Type: HTML  
  - Role: Wraps XML data with XML stylesheet processing instruction linking to the XSLT template URL (uses environment variable for webhook URL).  
  - Inputs: XML from Convert to XML  
  - Outputs: HTML content with XML and XSL stylesheet declaration  
  - Failures: Missing environment variable may cause broken URLs.

- **Move Binary Data**  
  - Type: Move Binary Data  
  - Role: Converts HTML content in JSON to a binary representation for HTTP response with MIME type `text/xml`.  
  - Inputs: HTML content  
  - Outputs: Binary data for response  
  - Failures: Incorrect MIME type or missing source data.

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends the final XML/HTML page as HTTP response with proper headers (`Content-Type: text/xml`, CORS allowed).  
  - Inputs: Binary data  
  - Outputs: HTTP response to requester  
  - Failures: Network or permission errors.

---

#### 2.5 Webhook Endpoint for XSLT Template

**Overview:**  
Serves the XSLT file which defines the visual structure and styling of the dashboard HTML page using Bootstrap 5.

**Nodes Involved:**  
- Final template (Set)  
- Respond to Webhook2

**Node Details:**

- **Final template**  
  - Type: Set  
  - Role: Holds the entire XSLT stylesheet as a string value `xsl_template`. Includes HTML head with Bootstrap 5 CSS/JS, custom CSS styles, and XSLT templates for transforming JSON XML data into styled HTML.  
  - Inputs: Triggered by `Request xsl template` webhook  
  - Outputs: XSLT string ready to serve  
  - Failures: None typical.

- **Respond to Webhook2**  
  - Type: Respond to Webhook  
  - Role: Serves the XSLT stylesheet string as HTTP response with Content-Type `text/xsl`.  
  - Inputs: XSLT string from Final template  
  - Outputs: HTTP response serving the XSL file  
  - Failures: Network or permission errors.

---

#### 2.6 Notes & Documentation

**Overview:**  
Sticky notes provide guidance, explanations, and important operational notes embedded in the workflow UI.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4

**Node Details:**

- **Sticky Note**  
  - Describes the JSON structure created by the workflow and JS tips on JMESPath, Map, DateTime (Luxon), and Template literals used.  
- **Sticky Note1**  
  - Explains the purpose of the webhook serving the XML dashboard page, the use of XSLT and Bootstrap 5 CDN, and advises saving CSS/JS files locally if desired.  
- **Sticky Note2**  
  - Warns about the webhook being unprotected and publicly accessible; advises adding authentication if required.  
- **Sticky Note3**  
  - Important note for cloud users regarding unavailable environment variables; instructs manual replacement of instance URLs in specific nodes.  
- **Sticky Note4**  
  - Short note indicating to use the webhook endpoint for dashboard access.

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                                              | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                                              |
|----------------------------|--------------------------|--------------------------------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger           | Manual trigger to start workflow                              | None                          | n8n-get-workflows             |                                                                                                                                          |
| Execute Workflow Trigger    | Execute Workflow Trigger  | Internal trigger for workflow execution                       | When clicking "Test workflow" | n8n-get-workflows             |                                                                                                                                          |
| n8n-get-workflows          | n8n                      | Fetch all workflows from n8n instance                         | Execute Workflow Trigger       | get-nodes-via-jmespath        |                                                                                                                                          |
| get-nodes-via-jmespath     | Set                      | Extracts nodes, tags, webhook paths arrays, sets instance URL | n8n-get-workflows             | workflows-section             | Sticky Note3: Cloud users must replace env vars with actual URL in this node                                                             |
| workflows-section          | Set                      | Creates workflow stats object with metadata                   | get-nodes-via-jmespath        | nodes-section, tags-section, globals-section, Sort-workflows, webhook-section |                                                                                                                                          |
| nodes-section              | Code                     | Maps nodes to workflows                                       | workflows-section             | Sort-nodes                   | Sticky Note: Explains JSON sections and JS tips                                                                                         |
| tags-section               | Code                     | Maps tags to workflows                                        | workflows-section             | Sort-tags                    | Sticky Note: Explains JSON sections and JS tips                                                                                         |
| globals-section            | Set                      | Computes global summary statistics                            | workflows-section             | Final-json                   | Sticky Note: Explains JSON sections and JS tips                                                                                         |
| webhook-section            | Code                     | Maps webhook paths to workflows                               | workflows-section             | Sort-whooks                  | Sticky Note: Explains JSON sections and JS tips                                                                                         |
| Sort-workflows             | Sort                     | Sorts workflows by updated date descending, then name         | workflows-section             | Aggregate-workflows          |                                                                                                                                          |
| Sort-nodes                 | Sort                     | Sorts nodes by usage count descending, then node name         | nodes-section                 | Aggregate-nodes              |                                                                                                                                          |
| Sort-tags                  | Sort                     | Sorts tags by usage count descending, then tag name           | tags-section                  | Aggregate-tags               |                                                                                                                                          |
| Sort-whooks                | Sort                     | Sorts webhooks by usage count descending, then path           | webhook-section               | Aggregate-whooks             |                                                                                                                                          |
| Aggregate-workflows        | Aggregate                | Aggregates workflows into array field                         | Sort-workflows                | Final-json                   |                                                                                                                                          |
| Aggregate-nodes            | Aggregate                | Aggregates nodes into array field                             | Sort-nodes                   | Final-json                   |                                                                                                                                          |
| Aggregate-tags             | Aggregate                | Aggregates tags into array field                              | Sort-tags                    | Final-json                   |                                                                                                                                          |
| Aggregate-whooks           | Aggregate                | Aggregates webhook paths into array field                     | Sort-whooks                  | Final-json                   |                                                                                                                                          |
| Final-json                 | Merge (combine)           | Combines all sections (globals, workflows, nodes, tags, webhooks) | globals-section, Aggregate-workflows, Aggregate-nodes, Aggregate-tags, Aggregate-whooks | Prepare JSON object          | Sticky Note: Explains JSON sections and JS tips                                                                                         |
| Prepare JSON object        | Execute Workflow          | Executes current workflow to prepare fresh JSON               | Request HTML dashboard        | Convert to XML               | Sticky Note2: Warns about public access and lack of authentication                                                                      |
| Convert to XML             | XML                      | Converts JSON dashboard data to XML                           | Prepare JSON object           | Create HTML                 |                                                                                                                                          |
| Create HTML                | HTML                     | Wraps XML with XML stylesheet processing instruction          | Convert to XML               | Move Binary Data            | Sticky Note3: Cloud users must replace env var with actual URL                                                                          |
| Move Binary Data           | Move Binary Data          | Converts HTML JSON property to binary for HTTP response       | Create HTML                  | Respond to Webhook           |                                                                                                                                          |
| Respond to Webhook         | Respond to Webhook        | Serves final XML/HTML dashboard page                          | Move Binary Data             | None                        | Sticky Note1: Explains purpose of this webhook and XSLT + Bootstrap usage                                                              |
| Request HTML dashboard     | Webhook                  | HTTP endpoint to serve dashboard                              | None                         | Prepare JSON object          | Sticky Note2: Warns about public access and lack of authentication                                                                      |
| Request xsl template       | Webhook                  | HTTP endpoint to serve XSLT template                          | None                         | Final template              |                                                                                                                                          |
| Final template             | Set                      | Contains entire XSLT stylesheet string                        | Request xsl template          | Respond to Webhook2          | Sticky Note1: Explains purpose of this webhook and XSLT + Bootstrap usage                                                              |
| Respond to Webhook2        | Respond to Webhook        | Serves the XSLT stylesheet                                    | Final template               | None                        |                                                                                                                                          |
| Sticky Note                | Sticky Note              | Workflow explanation and JS tips                              | None                         | None                        | Covers JSON structure and JS tips                                                                                                       |
| Sticky Note1               | Sticky Note              | Explains the purpose of the XML dashboard webhook            | None                         | None                        | Covers XML dashboard serving and XSLT styling                                                                                          |
| Sticky Note2               | Sticky Note              | Warns about public access and missing authentication         | None                         | None                        | Covers security considerations for webhook                                                                                             |
| Sticky Note3               | Sticky Note              | Notes for cloud users about environment variables             | None                         | None                        | Covers environment variable replacements                                                                                               |
| Sticky Note4               | Sticky Note              | Indicates which webhook to use for dashboard requests        | None                         | None                        |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking "Test workflow"`  
   - Purpose: Allows manual execution.

2. **Create Execute Workflow Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Name: `Execute Workflow Trigger`  
   - Purpose: Supports internal workflow invocation.  
   - Connect output of Manual Trigger → input of this node.

3. **Create n8n API Node**  
   - Type: n8n  
   - Name: `n8n-get-workflows`  
   - Credentials: Set up with your n8n API credentials.  
   - Parameters: No filters (fetch all workflows).  
   - Connect output of `Execute Workflow Trigger` → input.

4. **Create Set Node to Extract Arrays**  
   - Type: Set  
   - Name: `get-nodes-via-jmespath`  
   - Parameters:  
     - `nodes_array`: Use JMESPath expression to extract node types and convert to uppercase last segment:  
       `{{$jmespath($json,'nodes[*].type').map(item => (item.split('.').pop().toUpperCase() ))}}`  
     - `tags_array`: Extract tag names: `{{$jmespath($json,'tags[*].name')}}`  
     - `instance_url`: `{{$env["N8N_PROTOCOL"]}}://{{$env["N8N_HOST"]}}` (replace with actual URL if cloud)  
     - `webhook_paths_array`: Extract webhook paths:  
       `{{$jmespath($json, 'nodes[?type==\'n8n-nodes-base.webhook\'].parameters.path | [?@]')}}`  
   - Connect output of `n8n-get-workflows` → input.

5. **Create Set Node to Prepare Workflow Stats Object**  
   - Type: Set  
   - Name: `workflows-section`  
   - Parameters: Use JavaScript/Expression to create `wf_stats` object with:  
     - Unique nodes, total nodes, unique count  
     - Formatted created and updated timestamps using Luxon  
     - Workflow name, id, URL, active status, trigger count, tags, webhook paths  
   - Connect output of `get-nodes-via-jmespath` → input.

6. **Create Code Node for Nodes Mapping**  
   - Type: Code  
   - Name: `nodes-section`  
   - JavaScript: Map nodes to workflows, count usages, output array.  
   - Connect output of `workflows-section` → input.

7. **Create Code Node for Tags Mapping**  
   - Type: Code  
   - Name: `tags-section`  
   - JavaScript: Map tags to workflows, handle "No Tags" group, output array.  
   - Connect output of `workflows-section` → input.

8. **Create Code Node for Webhook Mapping**  
   - Type: Code  
   - Name: `webhook-section`  
   - JavaScript: Map webhook paths to workflows, filter invalid paths, output array.  
   - Connect output of `workflows-section` → input.

9. **Create Set Node for Global Stats**  
   - Type: Set  
   - Name: `globals-section`  
   - JavaScript/Expressions to compute total workflows, active workflows, total triggers.  
   - Connect output of `workflows-section` → input.

10. **Create Sort Nodes for Each Section**  
    - Sort workflows by updated descending, then name: `Sort-workflows`  
    - Sort nodes by count descending, then node name: `Sort-nodes`  
    - Sort tags by count descending, then tag name: `Sort-tags`  
    - Sort webhooks by count descending, then path: `Sort-whooks`  
    - Connect outputs of respective code nodes to these sort nodes.

11. **Create Aggregate Nodes for Each Section**  
    - Aggregate workflows into `wf_stats`: `Aggregate-workflows`  
    - Aggregate nodes into `nodes-section`: `Aggregate-nodes`  
    - Aggregate tags into `tags-section`: `Aggregate-tags`  
    - Aggregate webhooks into `whooks-section`: `Aggregate-whooks`  
    - Connect outputs of sort nodes to respective aggregates.

12. **Create Merge Node to Combine All Sections**  
    - Type: Merge (combine by position)  
    - Name: `Final-json`  
    - Connect outputs of `globals-section`, `Aggregate-workflows`, `Aggregate-nodes`, `Aggregate-tags`, and `Aggregate-whooks` to this node.

13. **Create Execute Workflow Node to Prepare JSON**  
    - Type: Execute Workflow  
    - Name: `Prepare JSON object`  
    - Parameter: Set workflow ID to current workflow ID (`{{$workflow.id}}`)  
    - Connect output of webhook node `Request HTML dashboard` → input.

14. **Create XML Node to Convert JSON to XML**  
    - Type: XML  
    - Name: `Convert to XML`  
    - Parameters: Mode `jsonToxml`, Headless = true  
    - Connect output of `Prepare JSON object` → input.

15. **Create HTML Node to Wrap XML with XSL Declaration**  
    - Type: HTML  
    - Name: `Create HTML`  
    - Parameters:  
      - HTML content includes XML declaration and stylesheet link referencing XSLT webhook path (replace environment variable with actual URL if needed).  
      - Body inserts XML data from previous node.  
    - Connect output of `Convert to XML` → input.

16. **Create Move Binary Data Node**  
    - Type: Move Binary Data  
    - Name: `Move Binary Data`  
    - Mode: jsonToBinary  
    - MIME Type: `text/xml`  
    - Source Key: `html`  
    - Connect output of `Create HTML` → input.

17. **Create Respond to Webhook Node to Serve Dashboard**  
    - Type: Respond to Webhook  
    - Name: `Respond to Webhook`  
    - Response Code: 200  
    - Response Headers: Content-Type `text/xml`, Access-Control-Allow-Origin `*`  
    - Respond with: binary data  
    - Connect output of `Move Binary Data` → input.

18. **Create Webhook Node for Dashboard Request**  
    - Type: Webhook  
    - Name: `Request HTML dashboard`  
    - Path: Unique path (e.g., `fb550a01-12f2-4709-ba2d-f71197b68340`)  
    - Response Mode: Response Node  
    - Connect output → `Prepare JSON object`.

19. **Create Webhook Node for Serving XSLT**  
    - Type: Webhook  
    - Name: `Request xsl template`  
    - Path: Unique path + `/dashboard.xsl` (e.g., `73a91e4d-143d-4168-9efb-6c56f2258aec/dashboard.xsl`)  
    - Response Mode: Response Node

20. **Create Set Node with Full XSLT Template**  
    - Type: Set  
    - Name: `Final template`  
    - Assign full XSLT stylesheet string (includes HTML head, Bootstrap 5 CSS/JS CDN links, custom CSS, and XSLT templates).  
    - Connect output of `Request xsl template` → input.

21. **Create Respond to Webhook Node to Serve XSLT**  
    - Type: Respond to Webhook  
    - Name: `Respond to Webhook2`  
    - Response Code: 200  
    - Response Headers: Content-Type `text/xsl`  
    - Respond with: text  
    - Connect output of `Final template` → input.

22. **Add Sticky Notes**  
    - Create sticky notes in workflow canvas to document usage, important notes about environment variables (especially for cloud users), security considerations, and references.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Reach out to Eduard for help adapting this workflow: https://www.linkedin.com/in/parsadanyan/                                                                                                                                 | Creator contact                                                                                 |
| Important: Cloud users must replace environment variables with actual n8n instance URLs in `get-nodes-via-jmespath` and `Create HTML` nodes.                                                                                  | Sticky Note3; workflow description                                                            |
| The dashboard uses XSLT with Bootstrap 5 loaded via CDN but can be served fully self-contained by hosting Bootstrap files locally.                                                                                            | Sticky Note1                                                                                   |
| The workflow JSON output can be used in BI tools for further visualization and analysis.                                                                                                                                        | Sticky Note                                                                                     |
| Useful JS tips for expressions: unique arrays via `new Set()`, array mapping, reducing sums, DateTime formatting with Luxon, and template literals.                                                                            | Sticky Note                                                                                     |
| Related workflows by Eduard and Yulia available at https://n8n.io/creators/eduard and https://n8n.io/creators/yulia                                                                                                            | Workflow description                                                                           |
| Original article about working with XML and SQL in n8n: https://blog.n8n.io/sql-xml/#how-to-deliver-the-xml-file                                                                                                               | About section card in dashboard                                                                |
| Additional related templates for auto-generating documentation and workflow visualization with Mermaid.js linked in the dashboard About section.                                                                              | About section cards                                                                            |
| Security note: The dashboard webhook is publicly accessible; consider adding authentication or IP restrictions to protect sensitive information.                                                                              | Sticky Note2                                                                                   |

---

This comprehensive analysis and instructions facilitate understanding, modifying, and reproducing the workflow for n8n instance dashboard creation, supporting both human operators and automated agents.