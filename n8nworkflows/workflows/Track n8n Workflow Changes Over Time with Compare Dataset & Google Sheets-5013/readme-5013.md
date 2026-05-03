Track n8n Workflow Changes Over Time with Compare Dataset & Google Sheets

https://n8nworkflows.xyz/workflows/track-n8n-workflow-changes-over-time-with-compare-dataset---google-sheets-5013


# Track n8n Workflow Changes Over Time with Compare Dataset & Google Sheets

### 1. Workflow Overview

This workflow is designed to track changes in n8n workflows over time by comparing the JSON definitions of workflows against previously stored versions in Google Sheets. Its main use cases include monitoring workflow changes in a team environment or managed client instances to detect modifications, additions, or removals of workflow nodes and connections, and maintaining a historical log of these changes.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Workflow Retrieval**: Periodically triggers and fetches all workflows from the n8n instance.
- **1.2 Workflow Existence Check & Entry Creation**: Loops through each workflow to check if it already exists in the Google Sheet; if not, creates a new entry.
- **1.3 Workflow State Preparation**: Sets current and previous workflow states from fetched data and sheet data.
- **1.4 Workflow Comparison Subworkflow Execution**: Invokes a subworkflow to compare the current and previous workflow JSONs to identify differences.
- **1.5 Node and Connection Differences Calculation**: Splits nodes and connections to compare them individually and aggregates added, removed, unchanged, or changed elements.
- **1.6 Reporting and Google Sheets Update**: Generates summary reports of changes and updates the Google Sheet with the latest workflow status and history.
- **1.7 Response Handling and Rate Limiting**: Sets response variables and handles Google Sheets rate limiting to ensure smooth operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Workflow Retrieval

- **Overview:**  
  Initiates the process on a schedule and retrieves all workflows from the n8n instance for tracking.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get All Workflows  
  - Loop Over Items

- **Node Details:**

  1. **Schedule Trigger**  
     - Type: Schedule Trigger  
     - Role: Starts the workflow on a defined interval (default daily)  
     - Configuration: Interval set to run once daily (empty object in interval array means default daily)  
     - Inputs: None  
     - Outputs: Connects to "Get All Workflows"  
     - Failure modes: Misconfiguration can cause it not to trigger; no authentication needed

  2. **Get All Workflows**  
     - Type: n8n node (internal API call to n8n)  
     - Role: Retrieves JSON definitions of workflows from the n8n instance  
     - Configuration: Limit set to 25 workflows; includes inactive workflows (activeWorkflows: false); does not return all workflows (returnAll: false)  
     - Credentials: Uses n8n API credentials  
     - Inputs: Trigger from Schedule Trigger  
     - Outputs: Connects to "Loop Over Items"  
     - Failure modes: API authentication errors, rate limits, or network issues

  3. **Loop Over Items**  
     - Type: Split In Batches  
     - Role: Processes workflows one by one through subsequent nodes  
     - Configuration: Default batch size (not explicitly set)  
     - Inputs: From "Get All Workflows"  
     - Outputs: Two outputs: main (empty) and error (connected to "Get Values")  
     - Failure modes: Batch size issues or empty input could cause no iteration

#### 2.2 Workflow Existence Check & Entry Creation

- **Overview:**  
  For each workflow, checks if an entry exists in Google Sheets; if not, creates a new row to track it.

- **Nodes Involved:**  
  - Get Values  
  - Get Row  
  - Row Exists? (If node)  
  - Has UpdatedAt Changed? (If node)  
  - Create New Entry  
  - set non-update response  
  - Set Response

- **Node Details:**

  1. **Get Values**  
     - Type: Set  
     - Role: Extracts and reformats key workflow data (id, name, nodes, connections, updatedAt, status) for further processing  
     - Configuration: Assigns properties by mapping workflow JSON fields into simpler forms; connections are flattened into a unified array with key, node, and index  
     - Inputs: From "Loop Over Items" error output  
     - Outputs: Connects to "Get Row"  
     - Failure modes: Expression errors when parsing connections or missing keys

  2. **Get Row**  
     - Type: Google Sheets  
     - Role: Looks up a row in Google Sheets matching the current workflow id  
     - Configuration: Filters rows where "id" equals current workflow id; targets specific spreadsheet and sheet (documentId and gid=0)  
     - Credentials: Google Sheets OAuth2  
     - Inputs: From "Get Values"  
     - Outputs: Connects to "Row Exists?"  
     - Failure modes: Google API errors, auth failures, rate limits

  3. **Row Exists?**  
     - Type: If  
     - Role: Checks if the Google Sheets query returned any data (row exists)  
     - Configuration: Condition tests if the output JSON is not empty  
     - Inputs: From "Get Row"  
     - Outputs:  
       - True: Connects to "Has UpdatedAt Changed?"  
       - False: Connects to "Create New Entry"  
     - Failure modes: Expression evaluation failure if JSON malformed

  4. **Has UpdatedAt Changed?**  
     - Type: If  
     - Role: Compares the 'updatedAt' timestamp from current workflow with the stored sheet value  
     - Configuration: Checks if timestamps are not equal (dateTime notEquals operator)  
     - Inputs: From "Row Exists?" true output  
     - Outputs:  
       - True: Connects to "Previous State" (to continue comparison)  
       - False: Connects to "set non-update response" (skips update)  
     - Failure modes: Date parsing errors, missing fields

  5. **Create New Entry**  
     - Type: Google Sheets (append)  
     - Role: Creates new row in Google Sheets for a workflow not previously tracked  
     - Configuration: Appends columns including id, name, nodes (stringified), status, history (with timestamp), connections (stringified), last_updated  
     - Credentials: Google Sheets OAuth2  
     - Inputs: From "Row Exists?" false output  
     - Outputs: Connects to "set non-update response"  
     - Failure modes: Sheet write failures, auth errors, rate limits

  6. **set non-update response**  
     - Type: Set  
     - Role: Sets a response indicating no update occurred (is_updated: false)  
     - Configuration: Assigns id and is_updated=false  
     - Inputs: From either "Create New Entry" or "Has UpdatedAt Changed?" false output  
     - Outputs: Connects to "Set Response"  
     - Failure modes: None expected

  7. **Set Response**  
     - Type: Set  
     - Role: Finalizes response data, defaulting is_updated to true if not set  
     - Configuration: Assigns id and is_updated (default true)  
     - Inputs: From "set non-update response"  
     - Outputs: Connects to "GSheets Rate Limit"  
     - Failure modes: None expected

#### 2.3 Workflow State Preparation

- **Overview:**  
  Prepares JSON objects representing current and previous workflow states for comparison.

- **Nodes Involved:**  
  - Previous State (Set)  
  - Current State (Set)  
  - Calculate Template Diffs (Execute Workflow)

- **Node Details:**

  1. **Previous State**  
     - Type: Set  
     - Role: Parses previous workflow data from Google Sheets, converting nodes and connections from JSON strings back to arrays  
     - Configuration: Assigns name, nodes (parsed JSON), connections (parsed JSON), status from sheet data  
     - Inputs: From "Has UpdatedAt Changed?" true output  
     - Outputs: Connects to "Current State"  
     - Failure modes: JSON parse errors if data corrupted

  2. **Current State**  
     - Type: Set  
     - Role: Formats current workflow data from n8n API response, setting arrays for nodes and connections, and status string  
     - Configuration: Assigns name, nodes, connections, status (active/inactive) fields from current workflow JSON  
     - Inputs: From "Previous State"  
     - Outputs: Connects to "Calculate Template Diffs"  
     - Failure modes: Expression failures if JSON fields missing

  3. **Calculate Template Diffs**  
     - Type: Execute Workflow (subworkflow)  
     - Role: Calls the same workflow in "each" mode with waitForSubWorkflow enabled, passing current and previous states for diff calculation  
     - Configuration: Inputs: data object containing current and previous states, jobType = "diff"  
     - Inputs: From "Current State"  
     - Outputs: Connects to "Has Node or Connection Updates?"  
     - Failure modes: Subworkflow execution failures, timeout

#### 2.4 Workflow Comparison Subworkflow Execution

- **Overview:**  
  When triggered with jobType "diff," this block compares nodes and connections between previous and current workflow states, grouping additions, removals, changes, and unchanged items.

- **Nodes Involved:**  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - Switch  
  - Split Nodes Current  
  - Split Nodes Previous  
  - Split Conn Current  
  - Split Conn Previous  
  - Compare Datasets (nodes)  
  - Compare Datasets1 (connections)  
  - Added Nodes, Same Nodes, Different Nodes, Removed Nodes (aggregate)  
  - Added Connections, Same Connections, Different Connections, Removed Connections (aggregate)  
  - Merge Nodes Diff  
  - Merge Connections Diff  
  - Report Nodes  
  - Report Connections  
  - Merge Reports

- **Node Details:**

  1. **When Executed by Another Workflow**  
     - Type: Execute Workflow Trigger  
     - Role: Entry point for subworkflow execution; expects inputs jobType and data object  
     - Configuration: Workflow inputs defined for jobType (string) and data (object)  
     - Inputs: Called by "Calculate Template Diffs" node externally  
     - Outputs: Connects to "Switch"  
     - Failure modes: Input validation errors

  2. **Switch**  
     - Type: Switch  
     - Role: Routes execution based on jobType value; here, only "diff" is used  
     - Configuration: Checks if $json.jobType equals "diff"  
     - Inputs: From "When Executed by Another Workflow"  
     - Outputs: Connects to four nodes in parallel: "Split Nodes Current", "Split Nodes Previous", "Split Conn Current", "Split Conn Previous"  
     - Failure modes: Routing errors if jobType missing or invalid

  3. **Split Nodes Current / Previous**  
     - Type: Split Out  
     - Role: Splits current and previous nodes arrays into individual items for comparison  
     - Configuration: Splits on data.current.nodes and data.previous.nodes respectively  
     - Inputs: From "Switch"  
     - Outputs: To "Compare Datasets" (nodes)  
     - Failure modes: Empty arrays or missing fields cause empty splits

  4. **Split Conn Current / Previous**  
     - Type: Split Out  
     - Role: Splits current and previous connections arrays into individual items for comparison  
     - Configuration: Splits on data.current.connections and data.previous.connections respectively  
     - Inputs: From "Switch"  
     - Outputs: To "Compare Datasets1" (connections)  
     - Failure modes: Empty arrays cause empty splits

  5. **Compare Datasets (nodes) and Compare Datasets1 (connections)**  
     - Type: Compare Datasets  
     - Role: Compares two datasets (previous vs current) by specified fields to classify entries as added, unchanged, changed, or removed  
     - Configuration:  
       - Nodes: merge by fields "type" and "name"  
       - Connections: merge by fields "key", "node", and "index"  
     - Inputs: From respective Split nodes  
     - Outputs: Four outputs each for added, unchanged, changed, removed  
     - Failure modes: Could fail on inconsistent field presence or complex nested structures

  6. **Aggregate Nodes and Connections (Added, Same, Different, Removed)**  
     - Type: Aggregate  
     - Role: Collects all items from each category into arrays  
     - Inputs: From outputs of Compare Datasets nodes  
     - Outputs:  
       - Nodes aggregates connect to "Merge Nodes Diff"  
       - Connections aggregates connect to "Merge Connections Diff"  
     - Failure modes: Empty data leads to empty aggregates

  7. **Merge Nodes Diff and Merge Connections Diff**  
     - Type: Merge  
     - Role: Combines the four aggregated outputs (added, unchanged, changed, removed) into a single JSON object per category  
     - Configuration: Combines by position, includes unpaired items  
     - Inputs: Aggregates for nodes or connections  
     - Outputs: Connects to "Report Nodes" or "Report Connections" respectively  
     - Failure modes: Merge conflicts if data sizes differ

  8. **Report Nodes and Report Connections**  
     - Type: Set  
     - Role: Creates summary JSON objects containing counts and names of added and removed nodes and connections  
     - Configuration: Uses expressions to count lengths and map names/types for user-friendly reporting  
     - Inputs: From merged diffs  
     - Outputs: Connects to "Merge Reports"  
     - Failure modes: Expression evaluation failures if data missing

  9. **Merge Reports**  
     - Type: Merge  
     - Role: Combines node and connection reports into one summary output  
     - Inputs: From "Report Nodes" and "Report Connections"  
     - Outputs: Connects back to "Calculate Template Diffs" node's output in main workflow  
     - Failure modes: Merge errors if inputs missing

#### 2.5 Reporting and Google Sheets Update

- **Overview:**  
  Updates the Google Sheet entry with the latest workflow status, including change history and summaries of node and connection changes.

- **Nodes Involved:**  
  - Has Node or Connection Updates? (If node)  
  - Update Entry  
  - set non-update response  
  - Set Response  
  - GSheets Rate Limit

- **Node Details:**

  1. **Has Node or Connection Updates?**  
     - Type: If  
     - Role: Determines if any nodes or connections have been added or removed (change count > 0)  
     - Configuration: Checks if any of nodes.added_total, nodes.removed_total, connections.added_total, connections.removed_total > 0  
     - Inputs: From "Calculate Template Diffs" subworkflow output  
     - Outputs:  
       - True: Connects to "Update Entry"  
       - False: Connects to "set non-update response"  
     - Failure modes: Expression failures if JSON structure unexpected

  2. **Update Entry**  
     - Type: Google Sheets (appendOrUpdate)  
     - Role: Updates existing Google Sheets row with new status, history, summary, and stringified nodes and connections data  
     - Configuration: Uses matching column "id" to update or append; updates history field with detailed change logs and timestamps  
     - Credentials: Google Sheets OAuth2  
     - Inputs: From "Has Node or Connection Updates?" true output  
     - Outputs: Connects to "Set Response"  
     - Failure modes: Google API errors, auth failures, rate limits

  3. **set non-update response**  
     - (Same as described in 2.2)  
     - Used here to set a false update flag if no changes detected.

  4. **Set Response**  
     - (Same as described in 2.2)  
     - Final response setting for downstream processing.

  5. **GSheets Rate Limit**  
     - Type: Wait  
     - Role: Pauses workflow for 3 seconds to avoid Google Sheets API rate limits  
     - Configuration: Wait time set to 3 seconds  
     - Inputs: From "Set Response"  
     - Outputs: Connects back to "Loop Over Items" to process next workflow  
     - Failure modes: None expected, but excessive delays if many workflows processed

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                                  | Input Node(s)                    | Output Node(s)                         | Sticky Note                                                                                          |
|--------------------------------|---------------------------|------------------------------------------------|---------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger          | Initiate workflow on schedule                   | None                            | Get All Workflows                    | See Sticky Note5 for workflow overview and explanation                                            |
| Get All Workflows              | n8n Node (n8n API)         | Retrieve workflows JSON from n8n instance       | Schedule Trigger                | Loop Over Items                     | See Sticky Note5                                                                                   |
| Loop Over Items                | Split In Batches           | Process workflows one by one                     | Get All Workflows               | Get Values (error output), main empty| See Sticky Note1 and Sticky Note2                                                                 |
| Get Values                    | Set                       | Extract and format workflow data                 | Loop Over Items (error output) | Get Row                           | See Sticky Note1                                                                                   |
| Get Row                       | Google Sheets              | Lookup workflow entry by id in Google Sheet     | Get Values                    | Row Exists?                       | See Sticky Note2                                                                                   |
| Row Exists?                   | If                        | Check if workflow entry exists                    | Get Row                       | Has UpdatedAt Changed? (true), Create New Entry (false) | See Sticky Note2                                                                                   |
| Has UpdatedAt Changed?        | If                        | Compare updatedAt timestamps                      | Row Exists?                   | Previous State (true), set non-update response (false) | See Sticky Note2                                                                                   |
| Create New Entry              | Google Sheets              | Create new Google Sheets row for new workflow    | Row Exists?                   | set non-update response            | See Sticky Note2                                                                                   |
| set non-update response       | Set                       | Set response flag when no update needed          | Create New Entry, Has UpdatedAt Changed? | Set Response                     |                                                                                                   |
| Set Response                 | Set                       | Finalize response data                            | set non-update response        | GSheets Rate Limit                 |                                                                                                   |
| Previous State               | Set                       | Prepare previous workflow state data             | Has UpdatedAt Changed? (true)  | Current State                     |                                                                                                   |
| Current State                | Set                       | Prepare current workflow state data              | Previous State                | Calculate Template Diffs           | See Sticky Note3                                                                                   |
| Calculate Template Diffs      | Execute Workflow           | Call subworkflow to calculate workflow diffs    | Current State                 | Has Node or Connection Updates?    | See Sticky Note3                                                                                   |
| When Executed by Another Workflow | Execute Workflow Trigger | Subworkflow entry point                           | Calculate Template Diffs (subworkflow call) | Switch                        |                                                                                                   |
| Switch                      | Switch                    | Route based on jobType                            | When Executed by Another Workflow | Split Nodes Current, Split Nodes Previous, Split Conn Current, Split Conn Previous |                                                                                                   |
| Split Nodes Current          | Split Out                 | Split current nodes array                         | Switch                       | Compare Datasets (nodes)           | See Sticky Note4                                                                                   |
| Split Nodes Previous         | Split Out                 | Split previous nodes array                        | Switch                       | Compare Datasets (nodes)           | See Sticky Note4                                                                                   |
| Split Conn Current           | Split Out                 | Split current connections array                   | Switch                       | Compare Datasets1 (connections)    | See Sticky Note4                                                                                   |
| Split Conn Previous          | Split Out                 | Split previous connections array                  | Switch                       | Compare Datasets1 (connections)    | See Sticky Note4                                                                                   |
| Compare Datasets             | Compare Datasets          | Compare nodes dataset                             | Split Nodes Current, Split Nodes Previous | Added Nodes, Same Nodes, Different Nodes, Removed Nodes | See Sticky Note4                                                                                   |
| Compare Datasets1            | Compare Datasets          | Compare connections dataset                       | Split Conn Current, Split Conn Previous | Added Connections, Same Connections, Different Connections, Removed Connections | See Sticky Note4                                                                                   |
| Added Nodes                 | Aggregate                 | Aggregate added nodes                             | Compare Datasets              | Merge Nodes Diff                  | See Sticky Note4                                                                                   |
| Same Nodes                  | Aggregate                 | Aggregate unchanged nodes                         | Compare Datasets              | Merge Nodes Diff                  | See Sticky Note4                                                                                   |
| Different Nodes             | Aggregate                 | Aggregate changed nodes                           | Compare Datasets              | Merge Nodes Diff                  | See Sticky Note4                                                                                   |
| Removed Nodes               | Aggregate                 | Aggregate removed nodes                           | Compare Datasets              | Merge Nodes Diff                  | See Sticky Note4                                                                                   |
| Added Connections           | Aggregate                 | Aggregate added connections                       | Compare Datasets1             | Merge Connections Diff           | See Sticky Note4                                                                                   |
| Same Connections            | Aggregate                 | Aggregate unchanged connections                   | Compare Datasets1             | Merge Connections Diff           | See Sticky Note4                                                                                   |
| Different Connections       | Aggregate                 | Aggregate changed connections                     | Compare Datasets1             | Merge Connections Diff           | See Sticky Note4                                                                                   |
| Removed Connections         | Aggregate                 | Aggregate removed connections                     | Compare Datasets1             | Merge Connections Diff           | See Sticky Note4                                                                                   |
| Merge Nodes Diff            | Merge                     | Merge node difference aggregates                  | Added Nodes, Same Nodes, Different Nodes, Removed Nodes | Report Nodes                    | See Sticky Note4                                                                                   |
| Merge Connections Diff      | Merge                     | Merge connection difference aggregates            | Added Connections, Same Connections, Different Connections, Removed Connections | Report Connections               | See Sticky Note4                                                                                   |
| Report Nodes               | Set                       | Build summary report of node changes              | Merge Nodes Diff             | Merge Reports                   | See Sticky Note4                                                                                   |
| Report Connections         | Set                       | Build summary report of connection changes        | Merge Connections Diff       | Merge Reports                   | See Sticky Note4                                                                                   |
| Merge Reports              | Merge                     | Combine node and connection reports                | Report Nodes, Report Connections | Calculate Template Diffs (return) | See Sticky Note4                                                                                   |
| Has Node or Connection Updates? | If                      | Check if any nodes or connections changed          | Calculate Template Diffs     | Update Entry (true), set non-update response (false) |                                                                                                   |
| Update Entry               | Google Sheets              | Update Google Sheet row with change details        | Has Node or Connection Updates? (true) | Set Response                   | See Sticky Note3                                                                                   |
| GSheets Rate Limit         | Wait                      | Pause to avoid exceeding Google Sheets API limits | Set Response                 | Loop Over Items                 |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a `Schedule Trigger` node**  
   - Set the trigger interval to once per day (default daily).  
   - Connect its output to the next node.

2. **Add an `n8n` node named "Get All Workflows"**  
   - Use n8n API credentials.  
   - Configure to retrieve workflows with a limit of 25; include inactive workflows.  
   - Connect Schedule Trigger output to this node.

3. **Add a `Split In Batches` node named "Loop Over Items"**  
   - Use default batch size or small batch size if desired.  
   - Connect "Get All Workflows" output to this node.

4. **Add a `Set` node named "Get Values"**  
   - Connect from the "Loop Over Items" error output (second output).  
   - Assign variables:  
     - id: `{{$json.id}}`  
     - name: `{{$json.name}}`  
     - nodes: map workflow nodes extracting `name` and `type`  
     - connections: flatten nested connections into array with keys `key`, `node`, `index`  
     - updatedAt: `{{$json.updatedAt}}`  
     - status: `{{$json.active ? 'active' : 'inactive'}}`

5. **Add a `Google Sheets` node named "Get Row"**  
   - Use Google Sheets OAuth2 credentials.  
   - Configure to read from the target spreadsheet and sheet.  
   - Use filter to lookup row by column "id" matching `={{ $json.id }}`.  
   - Connect "Get Values" output to this node.

6. **Add an `If` node named "Row Exists?"**  
   - Condition: Check if the output JSON from "Get Row" is not empty.  
   - True output connects to "Has UpdatedAt Changed?"  
   - False output connects to "Create New Entry".

7. **Add an `If` node named "Has UpdatedAt Changed?"**  
   - Condition: Compare current workflow updatedAt (`$('Get All Workflows').item.json.updatedAt`) to sheet's last_updated field (`$json.last_updated`), using "dateTime notEquals".  
   - True output connects to "Previous State"  
   - False output connects to "set non-update response"

8. **Add a `Google Sheets` node named "Create New Entry"**  
   - Append mode.  
   - Map columns: id, name, nodes (stringified JSON), status, history (initial timestamped info), connections (stringified JSON), last_updated.  
   - Connect "Row Exists?" false output here.  
   - Output connects to "set non-update response".

9. **Add a `Set` node named "set non-update response"**  
   - Assign id and `is_updated` = false.  
   - Connect outputs from "Create New Entry" and "Has UpdatedAt Changed?" false output here.  
   - Output connects to "Set Response".

10. **Add a `Set` node named "Set Response"**  
    - Assign id and `is_updated` (default true if not set).  
    - Connect from "set non-update response".  
    - Output connects to "GSheets Rate Limit".

11. **Add a `Wait` node named "GSheets Rate Limit"**  
    - Set to wait 3 seconds to handle Google Sheets API rate limits.  
    - Connect from "Set Response" output.  
    - Output loops back to "Loop Over Items" for next workflow.

12. **Add a `Set` node named "Previous State"**  
    - Input: from "Has UpdatedAt Changed?" true output.  
    - Assign:  
      - name: `$json.name`  
      - nodes: parse JSON string `$json.nodes`  
      - connections: parse JSON string `$json.connections`  
      - status: `$json.status`

13. **Add a `Set` node named "Current State"**  
    - Input: from "Previous State".  
    - Assign:  
      - name: from current workflow JSON  
      - nodes: current workflow nodes array  
      - connections: current workflow connections array  
      - status: active/inactive from current workflow JSON

14. **Add an `Execute Workflow` node named "Calculate Template Diffs"**  
    - Configure to call the same workflow ID (self-call).  
    - Mode: each, waitForSubWorkflow enabled.  
    - Pass inputs: jobType = "diff", data = object containing current and previous states.  
    - Connect from "Current State".  
    - Output connects to "Has Node or Connection Updates?".

15. **Create subworkflow nodes (inside same workflow) for diff calculation:**  
    - Add an `Execute Workflow Trigger` node "When Executed by Another Workflow"  
      - Define inputs: jobType (string), data (object).  
    - Add a `Switch` node to route on jobType == "diff".  
    - Add four `Split Out` nodes:  
      - Split Nodes Current (data.current.nodes)  
      - Split Nodes Previous (data.previous.nodes)  
      - Split Conn Current (data.current.connections)  
      - Split Conn Previous (data.previous.connections)  
    - Add two `Compare Datasets` nodes:  
      - For nodes, merge by fields: type, name  
      - For connections, merge by fields: key, node, index  
    - Add eight `Aggregate` nodes to aggregate added, unchanged, changed, removed datasets separately for nodes and connections.  
    - Add two `Merge` nodes to merge nodes and connections diffs.  
    - Add two `Set` nodes "Report Nodes" and "Report Connections" to create summary JSONs with counts and names.  
    - Add a final `Merge` node "Merge Reports" to combine node and connection reports.  
    - Connect "Merge Reports" output back to main workflow node "Calculate Template Diffs".

16. **Add an `If` node named "Has Node or Connection Updates?"**  
    - Condition: Check if any of the added or removed counts for nodes/connections > 0.  
    - True output connects to "Update Entry".  
    - False output connects to "set non-update response" (reuse existing node).

17. **Add a `Google Sheets` node named "Update Entry"**  
    - Operation: appendOrUpdate, matching column "id".  
    - Update columns with id, name, nodes (stringified), status, history (append new change logs with timestamps), summary (text summary), connections (stringified), last_updated.  
    - Connect from "Has Node or Connection Updates?" true output.  
    - Output connects to "Set Response".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow runs daily to track and report on any changes to workflows on an n8n instance. Useful for team collaboration or managed client instances monitoring. | See Sticky Note5 in workflow JSON |
| The workflow uses the n8n node to retrieve workflows, Google Sheets to store previous workflow states, and Compare Datasets nodes to identify changes in nodes and connections. | Sticky Note1, Sticky Note4 |
| Subworkflow execution is used to modularize the diff calculation logic, enabling easy reuse and scalability. | Sticky Note3 |
| Sample Google Sheet used in the workflow can be found here: https://docs.google.com/spreadsheets/d/1dOHSfeE0W_qPyEWj5Zz0JBJm8Vrf_cWp-02OBrA_ZYc/edit?usp=sharing | Sticky Note5 |
| For further help or community support, join the n8n Discord https://discord.com/invite/XPKeKXeB7d or the n8n Forum https://community.n8n.io/ | Sticky Note5 |
| The Google Sheets node is configured with OAuth2 credentials; ensure proper authentication and API quota management to avoid failures. | General integration note |
| The workflow includes rate limiting (3-second wait) to prevent Google Sheets API quota exceedance. | Node "GSheets Rate Limit" |
| All JSON parsing and stringifying uses n8n expressions and JavaScript within Set nodes; ensure careful syntax to avoid errors. | General best practice |
| Date/time comparisons rely on ISO 8601 format timestamps from n8n workflows' `updatedAt` fields. | General best practice |

---

**Disclaimer:**  
The text above is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.