v1 helper - Find params with affected expressions

https://n8nworkflows.xyz/workflows/v1-helper---find-params-with-affected-expressions-1929


# v1 helper - Find params with affected expressions

### 1. Workflow Overview

This workflow, titled **"v1 helper - Find params with affected expressions"**, is designed as a diagnostic tool to be run **after upgrading to n8n version 1**. Its primary purpose is to scan all active workflows within an n8n instance and identify any node parameters that contain expression extensions affected by breaking changes introduced in n8n v1.

Logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow on demand.
- **1.2 Fetch Active Workflows:** Retrieves all currently active workflows from the n8n instance.
- **1.3 Expression Analysis:** Analyzes each node parameter in the retrieved workflows to find those containing expressions affected by specific v1 changes.
- **1.4 Documentation:** Provides a sticky note with usage instructions and context.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow via manual user interaction.
- **Nodes Involved:**  
  - When clicking "Execute Workflow"
- **Node Details:**
  - **Type:** Manual Trigger (`n8n-nodes-base.manualTrigger`)  
  - **Purpose:** Allows user to manually start the workflow execution.  
  - **Configuration:** No parameters configured; default manual trigger settings.  
  - **Input Connections:** None (starting node).  
  - **Output Connections:** Connected to node "n8n" for subsequent processing.  
  - **Version Requirements:** Compatible with all n8n versions supporting manual triggers.  
  - **Possible Failures:** None expected unless workflow is disabled or n8n service is down.  
  - **Sub-workflow:** None.

#### 1.2 Fetch Active Workflows

- **Overview:** Retrieves all active workflows from the current n8n instance to analyze.
- **Nodes Involved:**  
  - n8n  
- **Node Details:**
  - **Type:** n8n API node (`n8n-nodes-base.n8n`)  
  - **Purpose:** Queries the n8n API to fetch metadata for all active workflows.  
  - **Configuration:**  
    - Filter applied: `activeWorkflows: true` — only retrieves workflows marked as active.  
    - Credentials: Uses an n8n API credential labeled "n8n account" for authentication.  
  - **Input Connections:** Receives trigger from "When clicking Execute Workflow".  
  - **Output Connections:** Passes data to "Find params with affected expressions" for detailed analysis.  
  - **Version Requirements:** Requires n8n instance API access and valid credentials.  
  - **Possible Failures:**  
    - Authentication errors if credentials expire or are invalid.  
    - API rate limiting or network errors.  
    - Empty result if no active workflows exist.  
  - **Sub-workflow:** None.

#### 1.3 Expression Analysis

- **Overview:** Scans each node parameter in all fetched workflows to identify expression extensions known to be affected by n8n v1 breaking changes.
- **Nodes Involved:**  
  - Find params with affected expressions
- **Node Details:**
  - **Type:** Code node (`n8n-nodes-base.code`)  
  - **Purpose:** Executes custom JavaScript to analyze parameters for affected expression extensions.  
  - **Configuration:**  
    - JavaScript code defines:  
      - A list of affected expression extensions: `['beginningOf', 'endOfMonth', 'minus', 'plus']`.  
      - A function to determine if a string is an expression (`={{...}}`).  
      - A search function that recursively scans node parameters for any parameter value that is an expression containing any of the affected extensions.  
    - Returns an array of objects listing:  
      - Workflow name  
      - Node name  
      - Parameter name  
  - **Key Expressions / Variables:**  
    - `AFFECTED_EXTENSIONS` array  
    - `isExpression(value)` checks if the value is an expression string  
    - `containsAny(str, substrings)` checks presence of affected extensions  
    - Recursive `findParamsByTest()` searches all nested parameters  
  - **Input Connections:** Receives workflows data from "n8n" node.  
  - **Output Connections:** None (final output).  
  - **Version Requirements:** Requires n8n version supporting the code node and expression syntax used.  
  - **Possible Failures:**  
    - If workflows data structure changes in future n8n versions, the code may fail parsing.  
    - Large data volumes may cause timeouts or performance issues.  
    - Expression syntax variations might cause false negatives.  
  - **Sub-workflow:** None.

#### 1.4 Documentation

- **Overview:** Provides a sticky note with workflow purpose and usage instructions.
- **Nodes Involved:**  
  - Sticky Note1
- **Node Details:**
  - **Type:** Sticky Note (`n8n-nodes-base.stickyNote`)  
  - **Purpose:** Displays textual information about the workflow: its intended use after upgrading to v1 and a link to relevant n8n v1 changes.  
  - **Configuration:**  
    - Content includes markdown formatted text with a link:  
      `https://github.com/n8n-io/n8n/pull/6435`  
  - **Input/Output Connections:** None (decorative node).  
  - **Possible Failures:** None.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                  | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                                    |
|-------------------------------|-----------------------|---------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger        | Start workflow manually          | None                            | n8n                             | ## v1 Helper ℹ️ This workflow is to be run **after upgrading to n8n v1**. This workflow returns all locations where a node in an active workflow contains a parameter using an expression extension affected by [v1 changes](https://github.com/n8n-io/n8n/pull/6435). For every location, please check that the workflow still behaves as intended. |
| n8n                           | n8n API                | Fetch all active workflows       | When clicking "Execute Workflow" | Find params with affected expressions | Same as above                                                                                                                                   |
| Find params with affected expressions | Code                  | Analyze parameters for affected expressions | n8n                             | None                             | Same as above                                                                                                                                   |
| Sticky Note1                  | Sticky Note            | Workflow description and instructions | None                            | None                             | ## v1 Helper ℹ️ This workflow is to be run **after upgrading to n8n v1**. This workflow returns all locations where a node in an active workflow contains a parameter using an expression extension affected by [v1 changes](https://github.com/n8n-io/n8n/pull/6435). For every location, please check that the workflow still behaves as intended. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it: `v1 helper - Find params with affected expressions`.

2. **Add a Manual Trigger node:**
   - Name it: `When clicking "Execute Workflow"`.
   - No special configuration required; default settings suffice.
   - This node will serve as the workflow starting point.

3. **Add an n8n API node:**
   - Name it: `n8n`.
   - Set node type to `n8n` (n8n API).
   - Configure parameters:
     - Filters: set `activeWorkflows` to `true` to fetch only active workflows.
   - Configure credentials:
     - Use valid n8n API credentials (OAuth2 or API key) with permissions to read workflows.
   - Connect the output of `When clicking "Execute Workflow"` node to the input of this node.

4. **Add a Code node:**
   - Name it: `Find params with affected expressions`.
   - Paste the following JavaScript code in the Code field (JS):
     ```javascript
     const AFFECTED_EXTENSIONS = ['beginningOf', 'endOfMonth', 'minus', 'plus'];

     const isExpression = (value) => typeof value === 'string' && value.startsWith('={{');

     const containsAny = (str, substrings) => {
       for (const substring of substrings) {
         if (str.includes(substring)) return true;
       }
       return false;
     }

     const isAffected = (value) => isExpression(value) && containsAny(value, AFFECTED_EXTENSIONS);

     function findParamsByTest(target, test) {
       const parameterNames = [];

       function search(obj) {
         if (typeof obj === 'object') {
           for (const key in obj) {
             const value = obj[key];

             if (test(value)) {
               parameterNames.push(key);
             } else if (typeof value === 'object') {
               search(value);
             }
           }
         }
       }

       search(target);

       return parameterNames;
     }

     return $input.all().reduce((allLocations, { json: workflow }) => {
       const perWorkflow = workflow.nodes.reduce((allLocationsPerWorkflow, node) => {
         const perNode = findParamsByTest(node.parameters, isAffected).map(
           (parameterName) => {
             return {
               workflowName: workflow.name,
               nodeName: node.name,
               parameterName,
             };
           },
         );

         return [...allLocationsPerWorkflow, ...perNode];
       }, []);

       return [...allLocations, ...perWorkflow];
     }, []);
     ```
   - Connect the output of the `n8n` node to this node.

5. **Add a Sticky Note node:**
   - Name it: `Sticky Note1`.
   - Paste the following content (Markdown supported):
     ```
     ## v1 Helper

     ℹ️ This workflow is to be run **after upgrading to n8n v1**.

     This workflow returns all locations where a node in an active workflow contains a parameter using an **expression extension affected by [v1 changes](https://github.com/n8n-io/n8n/pull/6435)**. For every location, please check that the workflow still behaves as intended.
     ```
   - Position it somewhere visible on the canvas.
   - This node has no input or output connections.

6. **Workflow Activation:**
   - Save the workflow.
   - Activate the workflow if desired; otherwise, run it manually using the manual trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                              |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow is intended to be run **after upgrading to n8n v1** to identify possible breaking changes in expressions. | Workflow description embedded in sticky note and metadata.  |
| The affected expression extensions are based on breaking changes introduced in [n8n v1 PR #6435](https://github.com/n8n-io/n8n/pull/6435). | GitHub pull request detailing changes to expression extensions. |

---

This documentation provides a comprehensive reference to understand, reproduce, and maintain the `v1 helper - Find params with affected expressions` workflow in n8n.