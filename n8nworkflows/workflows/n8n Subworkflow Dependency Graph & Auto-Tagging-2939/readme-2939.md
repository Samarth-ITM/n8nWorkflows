n8n Subworkflow Dependency Graph & Auto-Tagging

https://n8nworkflows.xyz/workflows/n8n-subworkflow-dependency-graph---auto-tagging-2939


# n8n Subworkflow Dependency Graph & Auto-Tagging

### 1. Workflow Overview

This workflow is designed to analyze an n8n instance to identify and manage dependencies between workflows, specifically focusing on sub-workflows—workflows that are executed by other workflows. It addresses the common challenge of tracking which workflows call others, verifying the existence of referenced sub-workflows, and automatically tagging sub-workflows based on their callers. The workflow also generates visualizations to provide insights into the usage and relationships of sub-workflows, helping enterprise teams maintain cleaner, more maintainable automation environments.

The workflow logic is organized into the following functional blocks:

- **1.1 Trigger and Initialization**: Defines when and how the workflow starts, including scheduled and manual triggers, and sets the instance URL.
- **1.2 Workflow Retrieval and Dependency Analysis**: Fetches all workflows, identifies which workflows call others, and builds a dependency graph.
- **1.3 Filtering and Verification**: Filters out workflows that are not sub-workflows and verifies that referenced sub-workflows exist.
- **1.4 Caller Counting and New Tag Identification**: Counts the number of callers per sub-workflow and identifies any new callers that are not yet tagged.
- **1.5 Tag Management**: Retrieves existing tags, creates new tags for new callers, and updates workflow tags accordingly.
- **1.6 Visualization**: Generates visual representations of the dependency graph using pie charts and MermaidJS diagrams.
- **1.7 Webhook for On-Demand Visualization**: Provides an HTTP webhook endpoint to view the dependency graph in a browser.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

- **Overview:**  
  This block defines how the workflow is triggered—either on activation, on a weekly schedule, or via an HTTP webhook—and initializes the instance URL used for API calls.

- **Nodes Involved:**  
  - When this workflow is activated  
  - And every Sunday  
  - When viewed in a browser  
  - SET instance_url

- **Node Details:**

  - **When this workflow is activated**  
    - Type: n8nTrigger  
    - Role: Starts the workflow when it is activated manually or deployed.  
    - Configuration: Trigger event set to "activate".  
    - Inputs: None  
    - Outputs: Connects to "GET all workflows".  
    - Edge Cases: None specific; triggers only on activation.

  - **And every Sunday**  
    - Type: scheduleTrigger  
    - Role: Schedules the workflow to run weekly on Sundays.  
    - Configuration: Interval set to every week.  
    - Inputs: None  
    - Outputs: Connects to "GET all workflows".  
    - Edge Cases: Timezone considerations may affect exact trigger time.

  - **When viewed in a browser**  
    - Type: webhook  
    - Role: Provides an HTTP endpoint `/dependency-graph` to trigger the workflow and return the dependency graph visualization.  
    - Configuration: Path set to "dependency-graph", response mode set to "responseNode".  
    - Inputs: HTTP request  
    - Outputs: Connects to "GET all workflows".  
    - Edge Cases: Requires proper webhook registration and accessible URL.

  - **SET instance_url**  
    - Type: set  
    - Role: Stores the n8n instance URL used in API requests.  
    - Configuration: Hardcoded string value `"https://n8n.example.com"` (to be replaced by user).  
    - Inputs: Receives data from "Loop through workflows" node.  
    - Outputs: Connects to "GET all tags".  
    - Edge Cases: Must be updated to the actual instance URL; incorrect URL causes API request failures.

---

#### 1.2 Workflow Retrieval and Dependency Analysis

- **Overview:**  
  This block fetches all workflows from the n8n instance and analyzes their nodes to identify which workflows call others, building a dependency graph that maps sub-workflows to their callers.

- **Nodes Involved:**  
  - GET all workflows  
  - List callers of subworkflows

- **Node Details:**

  - **GET all workflows**  
    - Type: n8n node (n8n)  
    - Role: Retrieves all workflows from the n8n instance via API.  
    - Configuration: No filters; uses n8n API credentials.  
    - Inputs: Trigger nodes ("When this workflow is activated", "And every Sunday", "When viewed in a browser").  
    - Outputs: Connects to "List callers of subworkflows".  
    - Edge Cases: API authentication errors, large number of workflows may cause timeouts.

  - **List callers of subworkflows**  
    - Type: code  
    - Role: Processes all workflows to build a dependency graph object mapping sub-workflows to their callers.  
    - Configuration: JavaScript code iterates workflows, identifies nodes of type `executeWorkflow` or `toolWorkflow`, extracts sub-workflow IDs, and records callers.  
    - Key Expressions: Uses `$input.all()` to process all workflows; builds maps for workflow names and tags; handles self-referencing workflows.  
    - Inputs: All workflows from previous node.  
    - Outputs: Returns an array of objects representing each sub-workflow with its callers and tags.  
    - Edge Cases: Missing or unknown workflow IDs, workflows without nodes, workflows with no callers.

---

#### 1.3 Filtering and Verification

- **Overview:**  
  Filters out workflows that are not sub-workflows (i.e., those without callers) and verifies that referenced sub-workflows exist in the instance.

- **Nodes Involved:**  
  - Exclude uncalled workflows  
  - GET workflow(s)  
  - Exclude missing workflows

- **Node Details:**

  - **Exclude uncalled workflows**  
    - Type: filter  
    - Role: Removes workflows that have zero callers (not sub-workflows).  
    - Configuration: Condition checks if `callers.length > 0`.  
    - Inputs: Output from "List callers of subworkflows".  
    - Outputs: Connects to "GET workflow(s)".  
    - Edge Cases: Workflows with empty or missing callers array.

  - **GET workflow(s)**  
    - Type: n8n node (n8n)  
    - Role: Retrieves detailed information for each sub-workflow by ID to verify existence.  
    - Configuration: Operation "get" with workflowId from input JSON.  
    - Inputs: Filtered sub-workflows from "Exclude uncalled workflows".  
    - Outputs: Connects to "Exclude missing workflows".  
    - Edge Cases: Nonexistent workflow IDs cause errors; node configured to continue on error.

  - **Exclude missing workflows**  
    - Type: filter  
    - Role: Filters out workflows that returned an error (nonexistent or inaccessible).  
    - Configuration: Condition checks that the item does not have an `error` field.  
    - Inputs: Output from "GET workflow(s)".  
    - Outputs: Connects to "Count callers and identify new callers".  
    - Edge Cases: API errors, permission issues.

---

#### 1.4 Caller Counting and New Tag Identification

- **Overview:**  
  Counts the number of callers for each sub-workflow and identifies any new callers that are not yet represented as tags on the sub-workflow.

- **Nodes Involved:**  
  - Count callers and identify new callers  
  - Loop through workflows  
  - SET instance_url

- **Node Details:**

  - **Count callers and identify new callers**  
    - Type: set  
    - Role: Assigns fields including `callers_count` (number of callers) and `new_callers` (callers not yet tagged).  
    - Configuration: Uses expressions to calculate counts and differences between callers and existing tags.  
    - Inputs: Verified sub-workflows from "Exclude missing workflows".  
    - Outputs: Connects to "Loop through workflows".  
    - Edge Cases: Empty callers or tags arrays.

  - **Loop through workflows**  
    - Type: splitInBatches  
    - Role: Processes workflows one by one or in batches for subsequent operations.  
    - Configuration: Default batch size (not explicitly set).  
    - Inputs: Output from "Count callers and identify new callers".  
    - Outputs: Two branches: one to "Combine dependency graph values into labels" and "Format workflow relationship data for rendering"; another to "SET instance_url".  
    - Edge Cases: Large number of workflows may affect performance.

  - **SET instance_url**  
    - (Described in 1.1)  
    - Receives input from "Loop through workflows" and outputs to "GET all tags".

---

#### 1.5 Tag Management

- **Overview:**  
  Manages tags for sub-workflows by retrieving existing tags, removing already existing tags from the new callers list, creating new tags for any remaining new callers, and updating the sub-workflows with the new tags.

- **Nodes Involved:**  
  - GET all tags  
  - Remove existing tags from new_callers list  
  - If any new callers  
  - Split out new callers as new tags  
  - Create new tags  
  - Return original pass through values  
  - Merge  
  - GET all tags again  
  - Create tag id:name dictionary  
  - Retrieve tag ids and names from dictionary  
  - Update workflow tags  
  - Return dependency graph data

- **Node Details:**

  - **GET all tags**  
    - Type: httpRequest  
    - Role: Retrieves all existing tags from the n8n instance.  
    - Configuration: GET request to `/api/v1/tags` with n8n API credentials.  
    - Inputs: From "SET instance_url".  
    - Outputs: Connects to "Remove existing tags from new_callers list".  
    - Edge Cases: API errors, permission issues.

  - **Remove existing tags from new_callers list**  
    - Type: set  
    - Role: Removes tags from the `new_callers` list that already exist in the instance to avoid duplicate tag creation.  
    - Configuration: Uses array difference expressions comparing new callers and existing tag names.  
    - Inputs: Output from "GET all tags".  
    - Outputs: Connects to "If any new callers".  
    - Edge Cases: Empty new_callers or tags arrays.

  - **If any new callers**  
    - Type: if  
    - Role: Checks if there are any new callers remaining to create tags for.  
    - Configuration: Condition checks if `new_callers` array is not empty.  
    - Inputs: Output from "Remove existing tags from new_callers list".  
    - Outputs:  
      - True: Connects to "Split out new callers as new tags".  
      - False: Connects to "Merge".  
    - Edge Cases: Empty arrays.

  - **Split out new callers as new tags**  
    - Type: splitOut  
    - Role: Splits the `new_callers` array into individual items for tag creation.  
    - Configuration: Field to split out is `new_callers`, output field named `new_tag_name`.  
    - Inputs: From "If any new callers" (true branch).  
    - Outputs: Connects to "Create new tags".  
    - Edge Cases: Empty input.

  - **Create new tags**  
    - Type: httpRequest  
    - Role: Creates new tags in the n8n instance for each new caller.  
    - Configuration: POST request to `/api/v1/tags` with body parameter `name` set to `new_tag_name`.  
    - Inputs: From "Split out new callers as new tags".  
    - Outputs: Connects to "Return original pass through values".  
    - Edge Cases: API errors, tag name conflicts.

  - **Return original pass through values**  
    - Type: code  
    - Role: Returns the original input data from "SET instance_url" to maintain data flow.  
    - Configuration: JavaScript code returns all items from "SET instance_url".  
    - Inputs: From "Create new tags".  
    - Outputs: Connects to "Merge".  
    - Edge Cases: None.

  - **Merge**  
    - Type: merge  
    - Role: Combines outputs from the false branch of "If any new callers" and from "Return original pass through values" to unify data flow.  
    - Configuration: Mode "combine" with `includeUnpaired` true, combining by position.  
    - Inputs: From "If any new callers" (false branch) and "Return original pass through values".  
    - Outputs: Connects to "GET all tags again".  
    - Edge Cases: Mismatched input lengths.

  - **GET all tags again**  
    - Type: httpRequest  
    - Role: Retrieves all tags again to get updated tag list after new tags creation.  
    - Configuration: GET request to `/api/v1/tags`.  
    - Inputs: From "Merge".  
    - Outputs: Connects to "Create tag id:name dictionary".  
    - Edge Cases: API errors.

  - **Create tag id:name dictionary**  
    - Type: set  
    - Role: Creates a dictionary object mapping tag IDs to tag names for easy lookup.  
    - Configuration: Uses JavaScript expression to reduce tag array into an object `{id: name}`.  
    - Inputs: From "GET all tags again".  
    - Outputs: Connects to "Retrieve tag ids and names from dictionary".  
    - Edge Cases: Empty tag list.

  - **Retrieve tag ids and names from dictionary**  
    - Type: set  
    - Role: Maps caller workflow IDs to tag IDs using the dictionary, preparing data for tag updates.  
    - Configuration: Uses expressions to map caller IDs to tag IDs, filters out undefined.  
    - Inputs: From "Create tag id:name dictionary".  
    - Outputs: Connects to "Update workflow tags".  
    - Edge Cases: Missing tag IDs.

  - **Update workflow tags**  
    - Type: httpRequest  
    - Role: Updates each sub-workflow's tags via PUT request to the n8n API.  
    - Configuration: PUT request to `/api/v1/workflows/{id}/tags` with JSON body containing the tags array.  
    - Inputs: From "Retrieve tag ids and names from dictionary".  
    - Outputs: Connects to "Return dependency graph data".  
    - Edge Cases: API errors, permission issues.

  - **Return dependency graph data**  
    - Type: set  
    - Role: Prepares and outputs the final dependency graph data with updated tags.  
    - Configuration: Assigns fields `id`, `callers`, `name`, and `callers_count`.  
    - Inputs: From "Update workflow tags".  
    - Outputs: Connects to "Loop through workflows" (for next iteration) and visualization nodes.  
    - Edge Cases: None.

---

#### 1.6 Visualization

- **Overview:**  
  Generates visual representations of the sub-workflow dependency graph, including a pie chart showing the most-called sub-workflows and a MermaidJS graph illustrating workflow relationships.

- **Nodes Involved:**  
  - Combine dependency graph values into labels  
  - Visualize subworkflow dependency graph  
  - Format workflow relationship data for rendering  
  - Visualize dependency graph with MermaidJS

- **Node Details:**

  - **Combine dependency graph values into labels**  
    - Type: aggregate  
    - Role: Aggregates fields `name`, `id`, and `callers_count` from the workflows for charting.  
    - Configuration: Aggregates specified fields from input data.  
    - Inputs: From "Loop through workflows".  
    - Outputs: Connects to "Visualize subworkflow dependency graph" and "Format workflow relationship data for rendering".  
    - Edge Cases: Empty input.

  - **Visualize subworkflow dependency graph**  
    - Type: quickChart  
    - Role: Creates a pie chart image showing the frequency of calls to each sub-workflow.  
    - Configuration: Pie chart with labels from workflow names and data from callers_count; output format PNG, size 600x600.  
    - Inputs: From "Combine dependency graph values into labels".  
    - Outputs: None (final visualization).  
    - Edge Cases: Large number of slices may reduce readability.

  - **Format workflow relationship data for rendering**  
    - Type: code  
    - Role: Converts the dependency graph data into MermaidJS graph syntax for visualization.  
    - Configuration: JavaScript builds a Mermaid `graph TD` string with edges representing caller relationships.  
    - Inputs: From "Loop through workflows".  
    - Outputs: Connects to "Visualize dependency graph with MermaidJS".  
    - Edge Cases: Workflows with no callers produce no edges.

  - **Visualize dependency graph with MermaidJS**  
    - Type: respondToWebhook  
    - Role: Returns an HTML page rendering the MermaidJS graph of workflow dependencies.  
    - Configuration: HTML template includes MermaidJS and Bootstrap; renders graph from Mermaid syntax passed in JSON.  
    - Inputs: From "Format workflow relationship data for rendering".  
    - Outputs: HTTP response to webhook request.  
    - Edge Cases: Browser compatibility, large graphs may be slow to render.

---

#### 1.7 Webhook for On-Demand Visualization

- **Overview:**  
  Provides an HTTP webhook endpoint `/dependency-graph` that triggers the workflow to fetch workflows, analyze dependencies, and return a MermaidJS visualization in the browser.

- **Nodes Involved:**  
  - When viewed in a browser  
  - GET all workflows  
  - List callers of subworkflows  
  - Exclude uncalled workflows  
  - GET workflow(s)  
  - Exclude missing workflows  
  - Count callers and identify new callers  
  - Loop through workflows  
  - Combine dependency graph values into labels  
  - Format workflow relationship data for rendering  
  - Visualize dependency graph with MermaidJS

- **Node Details:**  
  This path reuses nodes from previous blocks to provide an on-demand visualization via HTTP request. The webhook node initiates the flow, and the final node returns the MermaidJS graph as an HTML response.

---

### 3. Summary Table

| Node Name                          | Node Type                | Functional Role                                      | Input Node(s)                             | Output Node(s)                                  | Sticky Note                                                                                                   |
|-----------------------------------|--------------------------|-----------------------------------------------------|------------------------------------------|------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When this workflow is activated   | n8nTrigger               | Trigger on workflow activation                       | None                                     | GET all workflows                              |                                                                                                              |
| And every Sunday                  | scheduleTrigger          | Scheduled weekly trigger                             | None                                     | GET all workflows                              |                                                                                                              |
| When viewed in a browser          | webhook                  | HTTP endpoint for on-demand visualization           | None                                     | GET all workflows                              |                                                                                                              |
| SET instance_url                 | set                      | Stores n8n instance URL                              | Loop through workflows                    | GET all tags                                   | ### Change instance URL                                                                                       |
| GET all workflows                | n8n                      | Retrieves all workflows                              | When this workflow is activated, And every Sunday, When viewed in a browser | List callers of subworkflows                    |                                                                                                              |
| List callers of subworkflows     | code                     | Builds dependency graph of workflows and callers   | GET all workflows                        | Exclude uncalled workflows                      | The script builds a dependency graph of workflows by identifying which workflows call others (via execution nodes) while preserving workflow names, caller relationships, and tags. |
| Exclude uncalled workflows       | filter                   | Filters workflows with no callers                    | List callers of subworkflows             | GET workflow(s)                                | Here we filter out any workflows that are not [sub-workflows](https://docs.n8n.io/flow-logic/subworkflows/) (i.e. executed by other workflows). |
| GET workflow(s)                 | n8n                      | Retrieves workflow details by ID                     | Exclude uncalled workflows               | Exclude missing workflows                       |                                                                                                              |
| Exclude missing workflows        | filter                   | Filters out missing or errored workflows             | GET workflow(s)                         | Count callers and identify new callers         | We verify that the sub-workflows we intend to tag exist in our instance (not old workflow ids left over after importing a workflow from another instance) |
| Count callers and identify new callers | set                      | Counts callers and identifies new callers not tagged | Exclude missing workflows               | Loop through workflows                          |                                                                                                              |
| Loop through workflows           | splitInBatches           | Processes workflows in batches                        | Count callers and identify new callers  | Combine dependency graph values into labels, Format workflow relationship data for rendering, SET instance_url |                                                                                                              |
| GET all tags                    | httpRequest              | Retrieves all existing tags                          | SET instance_url                        | Remove existing tags from new_callers list     |                                                                                                              |
| Remove existing tags from new_callers list | set                      | Removes tags already existing from new callers list | GET all tags                           | If any new callers                             | If a tag is freshly created during an earlier iteration through the list of workflows, then it is removed from the list of new callers (i.e. new tags to create). |
| If any new callers              | if                       | Checks if there are new callers to create tags for  | Remove existing tags from new_callers list | Split out new callers as new tags (true), Merge (false) |                                                                                                              |
| Split out new callers as new tags | splitOut                 | Splits new callers array into individual tag names  | If any new callers (true)               | Create new tags                                |                                                                                                              |
| Create new tags                 | httpRequest              | Creates new tags in n8n instance                      | Split out new callers as new tags       | Return original pass through values             |                                                                                                              |
| Return original pass through values | code                     | Passes original data through                         | Create new tags                        | Merge                                          |                                                                                                              |
| Merge                          | merge                    | Combines branches after tag creation check           | If any new callers (false), Return original pass through values | GET all tags again                              |                                                                                                              |
| GET all tags again             | httpRequest              | Retrieves updated tag list after creation             | Merge                                  | Create tag id:name dictionary                   |                                                                                                              |
| Create tag id:name dictionary  | set                      | Creates dictionary mapping tag IDs to names           | GET all tags again                     | Retrieve tag ids and names from dictionary      |                                                                                                              |
| Retrieve tag ids and names from dictionary | set                      | Maps caller workflow IDs to tag IDs                   | Create tag id:name dictionary          | Update workflow tags                            |                                                                                                              |
| Update workflow tags          | httpRequest              | Updates workflow tags via API                          | Retrieve tag ids and names from dictionary | Return dependency graph data                    |                                                                                                              |
| Return dependency graph data  | set                      | Outputs final dependency graph data                    | Update workflow tags                   | Loop through workflows                          |                                                                                                              |
| Combine dependency graph values into labels | aggregate                | Aggregates workflow names, IDs, and caller counts     | Loop through workflows                 | Visualize subworkflow dependency graph, Format workflow relationship data for rendering | ## Generate chart                                                                                              |
| Visualize subworkflow dependency graph | quickChart               | Generates pie chart of sub-workflow usage              | Combine dependency graph values into labels | None                                           | ### Pie Chart Basic visualization of which sub-workflows are called most often by other workflows             |
| Format workflow relationship data for rendering | code                     | Converts dependency data into MermaidJS graph syntax  | Loop through workflows                 | Visualize dependency graph with MermaidJS       | ## Generate dependency graph                                                                                   |
| Visualize dependency graph with MermaidJS | respondToWebhook         | Returns HTML page rendering MermaidJS graph             | Format workflow relationship data for rendering | None                                           | ### Dependency Graph A visual representation of the relationships between the workflows in your n8n instance  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **n8nTrigger** node named "When this workflow is activated" with event set to "activate".
   - Add a **scheduleTrigger** node named "And every Sunday" with interval set to every week.
   - Add a **webhook** node named "When viewed in a browser" with path `dependency-graph` and response mode set to "responseNode".

2. **Retrieve All Workflows:**

   - Add an **n8n** node named "GET all workflows".
   - Configure it with n8n API credentials.
   - Connect all three trigger nodes to this node.

3. **Build Dependency Graph:**

   - Add a **code** node named "List callers of subworkflows".
   - Paste the JavaScript code that:
     - Iterates all workflows.
     - Identifies nodes of type `executeWorkflow` or `toolWorkflow`.
     - Builds a dependency graph mapping sub-workflows to their callers.
   - Connect "GET all workflows" to this node.

4. **Filter Sub-Workflows:**

   - Add a **filter** node named "Exclude uncalled workflows".
   - Configure condition: `$json.callers.length > 0`.
   - Connect "List callers of subworkflows" to this node.

5. **Verify Sub-Workflow Existence:**

   - Add an **n8n** node named "GET workflow(s)".
   - Configure operation "get" with workflowId from input JSON.
   - Set "On Error" to "continue".
   - Connect "Exclude uncalled workflows" to this node.

6. **Filter Out Missing Workflows:**

   - Add a **filter** node named "Exclude missing workflows".
   - Configure condition: item does not have `error` field.
   - Connect "GET workflow(s)" to this node.

7. **Count Callers and Identify New Callers:**

   - Add a **set** node named "Count callers and identify new callers".
   - Assign:
     - `callers_count` = length of `callers`.
     - `new_callers` = difference between `callers` and existing tags' names.
   - Connect "Exclude missing workflows" to this node.

8. **Process Workflows in Batches:**

   - Add a **splitInBatches** node named "Loop through workflows".
   - Connect "Count callers and identify new callers" to this node.

9. **Set Instance URL:**

   - Add a **set** node named "SET instance_url".
   - Assign a string field `instance_url` with your n8n instance URL (e.g., `https://n8n.example.com`).
   - Connect "Loop through workflows" to this node.

10. **Retrieve All Tags:**

    - Add an **httpRequest** node named "GET all tags".
    - Configure GET request to `{{$json.instance_url}}/api/v1/tags`.
    - Use n8n API credentials.
    - Connect "SET instance_url" to this node.

11. **Remove Existing Tags from New Callers:**

    - Add a **set** node named "Remove existing tags from new_callers list".
    - Assign `new_callers` as difference between current `new_callers` and existing tag names from HTTP response.
    - Connect "GET all tags" to this node.

12. **Check for New Callers:**

    - Add an **if** node named "If any new callers".
    - Condition: `new_callers` array is not empty.
    - Connect "Remove existing tags from new_callers list" to this node.

13. **Split New Callers into Tags:**

    - Add a **splitOut** node named "Split out new callers as new tags".
    - Field to split out: `new_callers`.
    - Destination field name: `new_tag_name`.
    - Connect "If any new callers" (true branch) to this node.

14. **Create New Tags:**

    - Add an **httpRequest** node named "Create new tags".
    - Configure POST request to `{{$json.instance_url}}/api/v1/tags`.
    - Body parameter: `name` = `{{$json.new_tag_name}}`.
    - Use n8n API credentials.
    - Connect "Split out new callers as new tags" to this node.

15. **Return Original Pass Through Values:**

    - Add a **code** node named "Return original pass through values".
    - JavaScript: `return $('SET instance_url').all();`
    - Connect "Create new tags" to this node.

16. **Merge Branches:**

    - Add a **merge** node named "Merge".
    - Mode: combine, include unpaired, combine by position.
    - Connect "If any new callers" (false branch) and "Return original pass through values" to this node.

17. **Retrieve All Tags Again:**

    - Add an **httpRequest** node named "GET all tags again".
    - Same configuration as "GET all tags".
    - Connect "Merge" to this node.

18. **Create Tag ID:Name Dictionary:**

    - Add a **set** node named "Create tag id:name dictionary".
    - Assign `tags` as an object mapping tag IDs to names using reduce on `$json.data`.
    - Pass through `callers` and `id`.
    - Connect "GET all tags again" to this node.

19. **Retrieve Tag IDs and Names from Dictionary:**

    - Add a **set** node named "Retrieve tag ids and names from dictionary".
    - Map caller IDs to tag IDs using the dictionary.
    - Pass through `id`, `callers`, `name`, and `callers_count`.
    - Connect "Create tag id:name dictionary" to this node.

20. **Update Workflow Tags:**

    - Add an **httpRequest** node named "Update workflow tags".
    - Configure PUT request to `{{$json.instance_url}}/api/v1/workflows/{{$json.id}}/tags`.
    - JSON body: `{{$json.tags}}`.
    - Use n8n API credentials.
    - Connect "Retrieve tag ids and names from dictionary" to this node.

21. **Return Dependency Graph Data:**

    - Add a **set** node named "Return dependency graph data".
    - Assign fields `id`, `callers`, `name`, and `callers_count` from previous node.
    - Connect "Update workflow tags" to this node.
    - Connect this node back to "Loop through workflows" for iteration.

22. **Aggregate Data for Visualization:**

    - Add an **aggregate** node named "Combine dependency graph values into labels".
    - Aggregate fields: `name`, `id`, `callers_count`.
    - Connect "Loop through workflows" to this node.

23. **Visualize Subworkflow Dependency Graph (Pie Chart):**

    - Add a **quickChart** node named "Visualize subworkflow dependency graph".
    - Chart type: pie.
    - Labels: workflow names.
    - Data: callers_count.
    - Chart options: 600x600 PNG.
    - Connect "Combine dependency graph values into labels" to this node.

24. **Format Workflow Relationship Data for MermaidJS:**

    - Add a **code** node named "Format workflow relationship data for rendering".
    - JavaScript code to build MermaidJS graph syntax from dependency data.
    - Connect "Loop through workflows" to this node.

25. **Visualize Dependency Graph with MermaidJS:**

    - Add a **respondToWebhook** node named "Visualize dependency graph with MermaidJS".
    - Response type: text/html.
    - Response body: HTML template embedding MermaidJS and rendering the graph.
    - Connect "Format workflow relationship data for rendering" to this node.

---

**Credentials Required:**

- **n8n API Credentials:**  
  Used by HTTP Request nodes and n8n nodes to access workflows and tags via the n8n API. Must have permissions to read workflows, read and write tags.

---

**Important Notes:**

- Replace the placeholder `https://n8n.example.com` in the "SET instance_url" node with your actual n8n instance URL.
- The workflow handles errors gracefully in "GET workflow(s)" by continuing on error to avoid stopping the entire process.
- The webhook node allows on-demand visualization by visiting `https://<your-n8n-instance>/webhook/dependency-graph`.
- The pie chart and MermaidJS visualizations provide complementary views: usage frequency and relationship graph.
- Large n8n instances with many workflows may require adjusting batch sizes or timeout settings.

---

This completes the detailed analysis and reconstruction instructions for the "n8n Subworkflow Dependency Graph & Auto-Tagging" workflow.