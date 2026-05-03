Mark outdated workflow nodes on canvas and send a summary with Gmail (add-on)

https://n8nworkflows.xyz/workflows/mark-outdated-workflow-nodes-on-canvas-and-send-a-summary-with-gmail--add-on--2477


# Mark outdated workflow nodes on canvas and send a summary with Gmail (add-on)

### 1. Workflow Overview

This workflow is an **add-on** designed to enhance an existing parent workflow that identifies outdated built-in nodes across all workflows in a single n8n instance. Its primary purpose is to mark these outdated nodes visually on the workflow canvas, facilitate their update by adding the latest node versions nearby, and send a summary email listing all modified workflows.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives workflow data with outdated nodes from the parent workflow.
- **1.2 Settings Initialization:** Loads configuration parameters controlling symbol usage, scope of updates, and canvas modifications.
- **1.3 Workflow Fetching:** Retrieves full workflow definitions for processing.
- **1.4 Workflow Modification:** Renames outdated nodes, adds updated versions to the canvas as configured, and updates connection references.
- **1.5 Workflow Update:** Pushes modified workflows back to the n8n instance.
- **1.6 Output Preparation and Notification:** Generates links to updated workflows and sends an email summary to notify the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block serves as the entry point and reference for incoming data, which includes workflows with outdated nodes and their metadata. It acts as a stable starting point for subsequent processing.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Start Reference  

- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: `Execute Workflow Trigger`  
    - Role: Entry trigger node invoked by the parent workflow when it completes outdated node detection.  
    - Config: Default, no parameters configured.  
    - Input/Output: No input, outputs data received from the parent workflow.  
    - Edge Cases: Trigger failure if parent workflow does not pass expected data format.  
    - Notes: This node initiates the add-on workflow automatically.

  - **Start Reference**  
    - Type: `No Operation` (noOp)  
    - Role: Acts as a stable reference point for the incoming data from the trigger, allowing easy modification of logic upstream without affecting downstream nodes.  
    - Config: None.  
    - Input/Output: Receives data from Execute Workflow Trigger, outputs unchanged data downstream.  
    - Edge Cases: None.

#### 2.2 Settings Initialization

- **Overview:**  
  Loads configurable parameters that determine how nodes are marked and updated, and the behavior of node addition on the canvas.

- **Nodes Involved:**  
  - Settings  

- **Node Details:**

  - **Settings**  
    - Type: `Set`  
    - Role: Defines key workflow parameters for symbol usage, update filtering, and canvas editing.  
    - Configuration:  
      - `instanceBaseUrl`: Base URL of the n8n instance (default: `http://localhost:5432`).  
      - `symbol`: Emoji/symbol to prepend to outdated node names (default: ⚠️).  
      - `onlyMajorChanges`: Boolean flag to include only major version changes (default: true).  
      - `addNodesToCanvas`: Boolean flag to add updated nodes visually on canvas (default: true).  
    - Input/Output: Receives data from Start Reference, outputs settings for downstream use.  
    - Edge Cases: Incorrect base URL or missing settings may cause failures in URL generation or logic decisions.

#### 2.3 Workflow Fetching

- **Overview:**  
  Fetches the complete definition of each workflow needing updates, based on IDs received from the input data.

- **Nodes Involved:**  
  - Get Workflow  

- **Node Details:**

  - **Get Workflow**  
    - Type: `n8n` node (API interaction)  
    - Role: Retrieves full workflow JSON by workflow ID using n8n API credentials.  
    - Configuration: Uses the workflow ID from the input item (`Start Reference`).  
    - Input/Output: Input is workflow metadata, output is full workflow JSON.  
    - Credentials: Requires n8n API credentials with read permissions.  
    - Edge Cases:  
      - Authentication failure if credentials are invalid.  
      - Workflow not found if ID is invalid or deleted.  
      - API timeout or connectivity issues.

#### 2.4 Workflow Modification

- **Overview:**  
  Processes each fetched workflow JSON to rename outdated nodes, add updated nodes near them on the canvas if configured, and update all relevant node connections and references.

- **Nodes Involved:**  
  - Modify Workflow (if required)  

- **Node Details:**

  - **Modify Workflow (if required)**  
    - Type: `Code` (JavaScript)  
    - Role: Implements the core logic for node renaming and canvas node addition.  
    - Configuration:  
      - Runs once per input item (one workflow).  
      - Reads settings from the "Settings" node for symbol, filter flags, and canvas addition.  
      - Loops over each outdated node:  
        - Skips minor updates if `onlyMajorChanges` is true.  
        - Checks if node name is already marked to avoid double processing.  
        - Renames outdated node by prepending the symbol.  
        - Adds a new node with latest version next to the old one if `addNodesToCanvas` is true.  
        - Updates all connection keys and references to renamed nodes accordingly.  
      - Outputs the modified workflow JSON or `null` if no changes needed.  
    - Input/Output: Receives full workflow JSON, outputs updated workflow JSON.  
    - Edge Cases:  
      - If the workflow structure changes significantly between versions, node cloning may fail or cause invalid workflows.  
      - Expression errors if expected fields are missing.  
      - No output if no changes are required, which will skip update calls downstream.

#### 2.5 Workflow Update

- **Overview:**  
  Updates the workflows in the n8n instance with the modified JSON reflecting renamed and added nodes.

- **Nodes Involved:**  
  - Update Workflow  

- **Node Details:**

  - **Update Workflow**  
    - Type: `n8n` node (API interaction)  
    - Role: Pushes the edited workflow JSON back to the n8n instance via API.  
    - Configuration: Uses workflow ID and full workflow JSON from the previous step.  
    - Credentials: Uses n8n API credentials with update permissions.  
    - Input/Output: Receives updated workflow JSON, outputs confirmation with updated workflow metadata.  
    - Edge Cases:  
      - API authentication or permission errors.  
      - Conflicts if workflow was changed concurrently elsewhere.  
      - Failures if JSON is malformed or invalid.

#### 2.6 Output Preparation and Notification

- **Overview:**  
  Prepares an HTML formatted list of updated workflows with links, then sends an email summary to a configured recipient.

- **Nodes Involved:**  
  - Prepare Output  
  - Send Summary  

- **Node Details:**

  - **Prepare Output**  
    - Type: `Set`  
    - Role: Constructs clickable HTML links for each updated workflow using the base URL and workflow ID, to be embedded in the email.  
    - Configuration: Uses an expression to create anchor tags with workflow names and URLs dynamically.  
    - Input/Output: Receives updated workflow metadata, outputs an array of formatted workflow links.  
    - Edge Cases:  
      - Invalid or missing base URL results in broken links.

  - **Send Summary**  
    - Type: `Gmail`  
    - Role: Sends an email with the list of workflows containing outdated nodes that were updated.  
    - Configuration:  
      - Subject: "Outdated n8n Workflow Nodes".  
      - Message body: HTML unordered list generated from the "Prepare Output" node.  
      - Recipient: Configured via Gmail OAuth2 credentials.  
      - Attribution disabled.  
      - Executes only once per workflow run.  
    - Credentials: Requires valid Gmail OAuth2 credentials with send email permissions.  
    - Input/Output: Receives formatted workflow list, outputs email send confirmation.  
    - Edge Cases:  
      - Authentication failure with Gmail.  
      - Email delivery issues or invalid recipient address.

---

### 3. Summary Table

| Node Name                 | Node Type                    | Functional Role                                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                              |
|---------------------------|------------------------------|-----------------------------------------------------|-------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger   | Execute Workflow Trigger      | Entry trigger invoked by parent workflow             |                         | Start Reference          |                                                                                                                          |
| Start Reference           | No Operation (noOp)           | Stable reference for incoming data                    | Execute Workflow Trigger | Settings                 | The following nodes start referencing from here, so it is easily possible to change the logic prior to this node.        |
| Settings                  | Set                          | Loads configuration parameters                        | Start Reference          | Get Workflow             | Minimum requirement: - Set your instanceBaseUrl                                                                         |
| Get Workflow              | n8n (API)                    | Retrieves full workflow JSON by ID                    | Settings                 | Modify Workflow (if required) |                                                                                                                          |
| Modify Workflow (if required) | Code (JavaScript)          | Renames outdated nodes, adds updated nodes, updates connections | Get Workflow             | Update Workflow          | Each workflow is being processed and modified if needed. Depending on the settings an icon is being prepended to the name of outdated nodes. In addition a newer version is being added close by, so it can be replaced quicker by the user. |
| Update Workflow           | n8n (API)                    | Updates the workflow JSON in n8n instance             | Modify Workflow (if required) | Prepare Output           | URL's are generated for each affected workflow                                                                           |
| Prepare Output            | Set                          | Creates HTML links for updated workflows              | Update Workflow          | Send Summary             |                                                                                                                          |
| Send Summary              | Gmail                        | Sends email summary with list of updated workflows    | Prepare Output           |                         | Minimum requirement: - Update mail recipient                                                                              |
| Sticky Note4              | Sticky Note                  | Instruction note about connecting parent workflow     |                         |                         | ## Download the main workflow and connect it's output to this workflow - Download this workflow and follow the belonging instructions: [https://n8n.io/workflows/2301-check-if-workflows-contain-build-in-nodes-that-are-not-of-the-latest-version/](https://n8n.io/workflows/2301-check-if-workflows-contain-build-in-nodes-that-are-not-of-the-latest-version/) - Add an "Execute Workflow" node and configure it, so it calls this workflow. ![Image](https://i.imgur.com/y0vPhYz.png#full-width) |
| Sticky Note               | Sticky Note                  | Notes the workflow is called by another workflow      |                         |                         | This workflow is called by another workflow which provides a list of all workflows with major and minor node updates      |
| Sticky Note1              | Sticky Note                  | Notes reference node for ease of logic change         |                         |                         | The following nodes start referencing from here, so it is easily possible to change the logic prior to this node.        |
| Sticky Note2              | Sticky Note                  | Example input data sample                              |                         |                         | ## Example input data\n\n```\n[\n  {\n    \"workflow\": \"Workflow Nodes Update\",\n    \"Id\": \"dFJpQTFg3QPH6Ol9\",\n    \"outdated_nodes\": [\n      {\n        \"name\": \"If\",\n        \"type\": \"n8n-nodes-base.if\",\n        \"version\": 2,\n        \"latestVersion\": 2.2\n      }\n    ]\n  }\n]\n``` |
| Sticky Note5              | Sticky Note                  | Reminder to update settings                            |                         |                         | ## Update settings\nMinimum requirement:\n- Set your instanceBaseUrl                                                    |
| Sticky Note6              | Sticky Note                  | Explains URL generation for affected workflows        |                         |                         | URL's are generated for each affected workflow                                                                           |
| Sticky Note7              | Sticky Note                  | Setup reminder for Gmail node                          |                         |                         | ## Setup Gmail\nMinimum requirement:\n- Update mail recipient                                                           |
| Sticky Note8              | Sticky Note                  | Explanation of modification logic                      |                         |                         | Each workflow is being processed and modified if needed. Depending on the settings an icon is being prepended to the name of outdated nodes. In addition a newer version is being added close by, so it can be replaced quicker by the user. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add an `Execute Workflow Trigger` node named `Execute Workflow Trigger`.  
   - No parameters required. This node will receive input from the parent workflow.

2. **Add a Reference Node:**  
   - Add a `No Operation (noOp)` node named `Start Reference`.  
   - Connect `Execute Workflow Trigger` → `Start Reference`.

3. **Add Settings Node:**  
   - Add a `Set` node named `Settings`.  
   - Connect `Start Reference` → `Settings`.  
   - Set the following variables (assignments):  
     - `instanceBaseUrl` (string): your n8n instance URL, e.g., `http://localhost:5432`  
     - `symbol` (string): emoji or symbol to prepend to outdated nodes, default `⚠️`  
     - `onlyMajorChanges` (boolean): set to `true` to process only major version changes  
     - `addNodesToCanvas` (boolean): set to `true` to add updated nodes visually on the canvas

4. **Add Workflow Fetch Node:**  
   - Add an `n8n` node named `Get Workflow`.  
   - Set operation to `get`.  
   - Set workflowId to an expression: `={{ $('Start Reference').item.json.Id }}` to dynamically get workflow IDs from input.  
   - Configure with valid n8n API credentials that have workflow read permissions.  
   - Connect `Settings` → `Get Workflow`.

5. **Add Workflow Modification Node:**  
   - Add a `Code` node named `Modify Workflow (if required)`.  
   - Set mode to `runOnceForEachItem`.  
   - Use the following logic (adapted from the workflow):  
     - Read `symbol`, `onlyMajorChanges`, and `addNodesToCanvas` from `Settings`.  
     - Iterate over `outdated_nodes` from input data.  
     - If `onlyMajorChanges` is true, skip nodes with same major version number.  
     - Rename outdated nodes by prefixing with `symbol` if not already prefixed.  
     - If `addNodesToCanvas` is true, add a copy of the latest version node shifted in position.  
     - Update connection keys and references to reflect renamed nodes.  
     - Return updated workflow JSON or `null` if no changes.  
   - Connect `Get Workflow` → `Modify Workflow (if required)`.

6. **Add Workflow Update Node:**  
   - Add an `n8n` node named `Update Workflow`.  
   - Set operation to `update`.  
   - Set workflowId to expression: `={{ $json.id }}`.  
   - Set workflowObject to expression: `={{ JSON.stringify($json) }}`.  
   - Configure with valid n8n API credentials with update permissions.  
   - Connect `Modify Workflow (if required)` → `Update Workflow`.

7. **Add Prepare Output Node:**  
   - Add a `Set` node named `Prepare Output`.  
   - Set the `name` field to expression:  
     ```js
     =<a href={{ $('Settings').item.json.instanceBaseUrl }}/workflow/{{ $json.id }}>{{ $json.name }}</a>
     ```  
   - Connect `Update Workflow` → `Prepare Output`.

8. **Add Email Notification Node:**  
   - Add a `Gmail` node named `Send Summary`.  
   - Configure Gmail OAuth2 credentials with email sending rights.  
   - Set:  
     - Subject: `Outdated n8n Workflow Nodes`  
     - Message (HTML):  
       ```html
       These workflows contain outdated nodes:<br>
       <ul>
       {{ $('Prepare Output').all().pluck('json').pluck('name').map(item => "<li>"+item+"</li>").join('') }}
       </ul>
       ```  
     - Disable attribution.  
     - Check "Execute Once".  
   - Connect `Prepare Output` → `Send Summary`.

9. **Configure Credentials:**  
   - Create and assign:  
     - n8n API credentials with proper permissions for `Get Workflow` and `Update Workflow` nodes.  
     - Gmail OAuth2 credentials for sending the summary email.

10. **Parent Workflow Integration:**  
    - In the parent workflow (the node updater), add an `Execute Workflow` node at the end.  
    - Configure it to call this add-on workflow, passing the list of workflows with outdated nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is an add-on to the main template “Check if workflows contain build-in nodes that are not of the latest version.” It requires the parent workflow to send data to it.                                                                                                                                                          | https://n8n.io/workflows/2301-check-if-workflows-contain-build-in-nodes-that-are-not-of-the-latest-version/                           |
| Beware that major node version updates can cause migration failures due to structural changes. Always verify updated nodes manually or in a test environment before applying changes to production.                                                                                                                                          | Warning note in description                                                                                                          |
| It is recommended to run this workflow on a test environment first and migrate stable workflows to production via git integration or manual export/import.                                                                                                                                                                                 | Disclaimer in description                                                                                                            |
| The symbol (default ⚠️) used to mark outdated nodes is also used to detect already processed nodes to avoid double updates. You can customize this symbol in the `Settings` node.                                                                                                                                                           | Settings node assignments                                                                                                            |
| Screenshot showing the addition of new nodes slightly shifted on the canvas to the right and above the outdated node for easy replacement.                                                                                                                                                                                                  | https://i.imgur.com/yRI0adF.png                                                                                                     |
| Setup instructions include: cloning this add-on workflow, setting the `instanceBaseUrl` in the Settings node, configuring the Gmail node recipient, cloning and configuring the parent workflow, and linking the parent to this add-on with an Execute Workflow node.                                                                        | See Sticky Note4 content and Setup section in description                                                                           |
| Example input data format expected from the parent workflow is a JSON array of objects with `workflow` name, `Id`, and `outdated_nodes` array with node details like name, type, version, latestVersion.                                                                                                                                       | Sticky Note2 content                                                                                                                 |

---

This structured documentation enables thorough understanding, reproduction, and safe modification of the workflow, both for advanced human users and AI-driven automation agents.