Create Workflow Inventory Dashboard with n8n API and Google Sheets

https://n8nworkflows.xyz/workflows/create-workflow-inventory-dashboard-with-n8n-api-and-google-sheets-9113


# Create Workflow Inventory Dashboard with n8n API and Google Sheets

### 1. Workflow Overview

This workflow automates the creation and maintenance of a centralized inventory dashboard for all n8n workflows by extracting metadata via the n8n API and updating a Google Sheet. It is designed for users who want to monitor and manage their n8n workflows in one place, enabling easy access, status tracking, and documentation.

Logical blocks:

- **1.1 Trigger Setup:** Provides two possible start points — a scheduled trigger for automatic periodic updates and a manual trigger for on-demand execution.
- **1.2 Workflow Retrieval:** Uses the n8n API to fetch the list of all workflows available in the n8n instance.
- **1.3 Workflow Processing Loop:** Iterates over each retrieved workflow to extract specific details.
- **1.4 Data Extraction:** Runs custom JavaScript code to parse and format workflow metadata including name, ID, tags, nodes used, and timestamps.
- **1.5 Google Sheets Update:** Adds or updates rows in a Google Sheet to reflect the current state of each workflow, avoiding duplicates by matching workflow IDs.
- **1.6 Rate Limiting Pause:** Adds a wait time after each sheet update to avoid hitting API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Setup

- **Overview:** Defines two alternative triggers to start the workflow: automatic scheduled execution or manual activation.
- **Nodes Involved:**  
  - Schedule Trigger  
  - When clicking ‘Execute workflow’
- **Node Details:**
  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Configuration: Runs automatically at set intervals (default interval is every minute as per empty interval array)  
    - Inputs: None  
    - Outputs: Connects to “Get All Workflows” node  
    - Failure Modes: Misconfigured intervals may cause unexpected trigger frequency; no authentication needed  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger node  
    - Configuration: Triggered only when user manually executes the workflow  
    - Inputs: None  
    - Outputs: Connects to “Get All Workflows” node  
    - Failure Modes: None (manual trigger)  

#### 1.2 Workflow Retrieval

- **Overview:** Fetches the complete list of workflows from the n8n instance via its API.
- **Nodes Involved:**  
  - Get All Workflows
- **Node Details:**
  - **Get All Workflows**  
    - Type: n8n API node (n8n)  
    - Configuration: No filters applied; retrieves all workflows  
    - Credentials: Requires valid n8n API credentials (OAuth or API key)  
    - Inputs: Trigger nodes (“Schedule Trigger” or “When clicking ‘Execute workflow’”)  
    - Outputs: Connects to “Loop Through Each Workflow” node  
    - Failure Modes: Authentication failure if credentials expired or invalid; API downtime; empty results if no workflows exist  

#### 1.3 Workflow Processing Loop

- **Overview:** Iterates through each workflow retrieved, processing them one batch item at a time for controlled execution.
- **Nodes Involved:**  
  - Loop Through Each Workflow
- **Node Details:**
  - **Loop Through Each Workflow**  
    - Type: SplitInBatches node  
    - Configuration: Default batch size (1 item per batch) to process workflows sequentially  
    - Inputs: “Get All Workflows” node  
    - Outputs: On first output (empty), loops back to itself (used for iteration control); on second output, proceeds to “Extract Workflow Details”  
    - Failure Modes: Large numbers of workflows may cause long execution times; improper batch size may lead to throttling or timeouts  

#### 1.4 Data Extraction

- **Overview:** Parses each workflow JSON object to extract key details: name, ID, tags (concatenated), unique node types used, and timestamps.
- **Nodes Involved:**  
  - Extract Workflow Details
- **Node Details:**
  - **Extract Workflow Details**  
    - Type: Code node (JavaScript)  
    - Configuration: Custom JS script reads the first input item, extracts workflow metadata, concatenates tag names, identifies unique node types, and returns structured JSON fields `nodes` and `info` (contains name, id, tags)  
    - Inputs: “Loop Through Each Workflow” node  
    - Outputs: Connects to “Add/Update Row in Google Sheet” node  
    - Key Expressions: Uses `$input.first().json` to access current workflow data  
    - Failure Modes: If workflow JSON structure changes (e.g., missing tags or nodes array), script may fail; malformed data may cause runtime errors; empty tags array handled safely by map/join  
    - Notes: Customizable code for additional metadata extraction as per sticky note  

#### 1.5 Google Sheets Update

- **Overview:** Appends or updates a row in the specified Google Sheet for each workflow using the extracted metadata, matching on workflow ID to prevent duplicates.
- **Nodes Involved:**  
  - Add/Update Row in Google Sheet
- **Node Details:**
  - **Add/Update Row in Google Sheet**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append or Update (upserts data)  
      - Sheet Name: "Sheet1" (gid=0)  
      - Document ID: Specific Google Sheets spreadsheet ID  
      - Columns mapped: id, link (URL to workflow), tags, nodes, title, Active, Archived, CreatedAt, UpdatedAt  
      - Matching Column: `id` to identify existing rows for update  
    - Credentials: Requires Google Sheets OAuth2 credentials with write access  
    - Inputs: “Extract Workflow Details” node  
    - Outputs: Connects to “Pause to Avoid Rate Limits” node  
    - Failure Modes: Credential expiry or revocation; API quota exceeded; sheet ID or name incorrect; mismatched schema causing update failures; network issues  
    - Notes: Sticky note emphasizes importance of “id” as matching column  

#### 1.6 Rate Limiting Pause

- **Overview:** Introduces a pause after each Google Sheet update to avoid hitting API rate limits.
- **Nodes Involved:**  
  - Pause to Avoid Rate Limits
- **Node Details:**
  - **Pause to Avoid Rate Limits**  
    - Type: Wait node  
    - Configuration: Default wait time (not explicitly set, likely default minimal wait)  
    - Inputs: “Add/Update Row in Google Sheet” node  
    - Outputs: Loops back to “Loop Through Each Workflow” to process next item  
    - Failure Modes: No specific failure expected; infinite loop risk if connections misconfigured  

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                | Input Node(s)              | Output Node(s)                   | Sticky Note                                                                                         |
|-----------------------------|----------------------|-------------------------------|----------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger     | Automatic periodic start       | None                       | Get All Workflows                | Use one of these triggers to run the workflow. Scheduled Start runs automatically.               |
| When clicking ‘Execute workflow’ | Manual Trigger      | Manual start                   | None                       | Get All Workflows                | Use one of these triggers to run the workflow. Manual Start runs on demand.                       |
| Get All Workflows           | n8n API Node          | Retrieve all workflows via API | Schedule Trigger, Manual Trigger | Loop Through Each Workflow       | Add your n8n API credentials here.                                                                |
| Loop Through Each Workflow  | SplitInBatches        | Iterate over workflows         | Get All Workflows           | Extract Workflow Details, Loop Through Each Workflow |                                                                                                   |
| Extract Workflow Details    | Code                  | Extract workflow metadata      | Loop Through Each Workflow  | Add/Update Row in Google Sheet   | Customize code to extract additional details if needed.                                          |
| Add/Update Row in Google Sheet | Google Sheets Node    | Update spreadsheet rows        | Extract Workflow Details    | Pause to Avoid Rate Limits        | CRITICAL: Set 'id' as Matching Column for proper upsert behavior.                                |
| Pause to Avoid Rate Limits  | Wait                  | Rate limit delay               | Add/Update Row in Google Sheet | Loop Through Each Workflow      |                                                                                                   |
| Sticky Note                 | Sticky Note           | Documentation and instructions | None                       | None                            | ## Workflow Inventory in Google Sheets — setup instructions and overview.                        |
| Sticky Note1                | Sticky Note           | Documentation on triggers      | None                       | None                            | ## Trigger Choice — explains scheduled vs manual triggers.                                       |
| Sticky Note2                | Sticky Note           | Google Sheets configuration    | None                       | None                            | ## Google Sheets Configuration — importance of matching column.                                 |
| Sticky Note3                | Sticky Note           | Customization tip for code node | None                       | None                            | ## Customization Tip — how to extend data extraction.                                           |
| Sticky Note7                | Sticky Note           | Contact information            | None                       | None                            | ## Contact me — author contact and support offer (thomas@pollup.net)                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node.
     - Set the interval for automatic execution (default every minute or as desired).
   - Add a **Manual Trigger** node.
     - No parameters needed.
   - You can keep both triggers connected to the next node or delete one depending on use case.

2. **Add n8n API Node to Retrieve Workflows:**
   - Add an **n8n** node.
   - Configure credentials with your n8n API access (OAuth or API key).
   - Set operation to retrieve all workflows without filters.

3. **Add SplitInBatches Node for Iteration:**
   - Add a **SplitInBatches** node.
   - Connect it to the output of the n8n API node.
   - Use default batch size (1) for sequential processing.

4. **Add Code Node to Extract Workflow Details:**
   - Add a **Code** node.
   - Connect it to the second output of the SplitInBatches node.
   - Paste the following JavaScript code (adapt as needed):
     ```js
     let info = {
       name: $input.first().json.name,
       id: $input.first().json.id,
       tags: $input.first().json.tags.map(ar => ar.name).join("\n")
     }
     
     let nodeTypes = {}
     for (const node of $input.first().json.nodes) {
       nodeTypes[node.type] = 1
     }
     let nodes = Object.keys(nodeTypes).join("\n");
     return {nodes, info }
     ```
   - This will extract workflow name, ID, concatenated tag names, and unique node types.

5. **Add Google Sheets Node to Append/Update Rows:**
   - Add a **Google Sheets** node.
   - Configure with Google OAuth2 credentials granting access to your Google Sheets.
   - Set the operation to **Append or Update**.
   - Specify your target Spreadsheet ID and Sheet Name (e.g., "Sheet1").
   - Define columns to map with expressions:
     - `id`: `={{ $json.info.id }}` (set as matching column)
     - `link`: `=https://n8n.pollup.net/workflow/{{ $json.info.id }}`
     - `tags`: `={{ $json.info.tags }}`
     - `nodes`: `={{ $json.nodes }}`
     - `title`: `={{ $json.info.name }}`
     - `Active`: `={{ $('Loop Through Each Workflow').item.json.active }}`
     - `Archived`: `={{ $('Loop Through Each Workflow').item.json.isArchived }}`
     - `CreatedAt`: `={{ $('Loop Through Each Workflow').item.json.createdAt }}`
     - `UpdatedAt`: `={{ $('Loop Through Each Workflow').item.json.updatedAt }}`
   - Ensure `id` is set as the "Matching Column" to correctly update existing rows.

6. **Add Wait Node to Avoid Rate Limits:**
   - Add a **Wait** node.
   - Connect from the Google Sheets node output.
   - Configure a pause duration as needed (default is minimal; increase if hitting rate limits).
   - Connect the wait node back to the first output of the SplitInBatches node to continue iterating.

7. **Connect Triggers to Start:**
   - Connect both the Schedule Trigger and Manual Trigger nodes to the n8n API node that fetches workflows.

8. **Add Sticky Notes for Documentation (Optional):**
   - Add several **Sticky Note** nodes with content explaining workflow purpose, trigger usage, Google Sheets setup, customization tips, and contact info.

9. **Activate the workflow:**
   - Activate and test by manually triggering or waiting for scheduled runs.
   - Monitor Google Sheet for updates reflecting your current workflows.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow Inventory in Google Sheets: Centralizes all n8n workflows for easy management and tracking.        | Sticky Note content in workflow                                  |
| Use either Scheduled Trigger or Manual Trigger as per your update preference; delete unused trigger node.  | Sticky Note1                                                    |
| Critical to set “id” as Matching Column in Google Sheets node to avoid duplicate rows on updates.          | Sticky Note2                                                    |
| Customize the Code node to extract additional workflow details (e.g., node settings, email addresses).     | Sticky Note3                                                    |
| For modifications, help or custom workflows in n8n, Make, or Langchain/Langgraph, contact: thomas@pollup.net | Sticky Note7 / author contact                                  |
| Official n8n documentation for Google Sheets node: https://docs.n8n.io/nodes/n8n-nodes-base.googlesheets/ | Reference for further node configuration                        |
| n8n API documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n/                            | Useful for extending API requests or filtering workflows       |

---

_Disclaimer: The provided content originates exclusively from an automated workflow created with n8n, respecting all current content policies and containing no illegal or protected materials. All data processed is legal and public._