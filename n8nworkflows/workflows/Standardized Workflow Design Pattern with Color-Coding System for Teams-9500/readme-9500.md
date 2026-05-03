Standardized Workflow Design Pattern with Color-Coding System for Teams

https://n8nworkflows.xyz/workflows/standardized-workflow-design-pattern-with-color-coding-system-for-teams-9500


# Standardized Workflow Design Pattern with Color-Coding System for Teams

---

### 1. Workflow Overview

This workflow, titled **"Standardized Workflow Design Pattern with Color-Coding System for Teams"**, serves as a visual and structural template for designing clean, maintainable, and standardized workflows in n8n. It focuses on establishing best practices for workflow architecture, including normalized input handling, data gathering, business logic segmentation, and comprehensive color-coded annotation for easy team collaboration and operational clarity.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception & Normalization**: Handles multiple trigger types (manual, webhook, sub-workflow) and normalizes their outputs into a common format for downstream processes.
- **1.2 Data Collection & Defaults Setting**: Gathers required external data from sources like Airtable and Google Sheets, and sets default parameters for the workflow.
- **1.3 Business Logic Processing**: Encapsulates distinct business logic implementations, including multiple versions and development stages, managed via a Switch node to route execution.
- **1.4 Workflow Metadata & Documentation**: Uses sticky notes extensively to document design patterns, color coding, and operational guidelines for the team.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization

**Overview:**  
This block consolidates various input triggers into a normalized data structure, enabling consistent downstream processing regardless of how the workflow was triggered.

**Nodes Involved:**  
- Manual (Manual Trigger)  
- General Webhook (Webhook)  
- Triggered from other Workflow (Execute Workflow Trigger)  
- Normalize Manual Trigger (Set)  
- Normalize Webhook Trigger (Set)  
- Trigger | Normalized (Set)  

**Node Details:**

- **Manual**  
  - *Type:* Manual Trigger  
  - *Role:* Enables manual triggering of the workflow for testing or direct execution.  
  - *Configuration:* Default settings, no parameters.  
  - *Connections:* Outputs to "Normalize Manual Trigger".  
  - *Edge Cases:* Manual triggers require user interaction; no input validation needed here.  

- **General Webhook**  
  - *Type:* Webhook  
  - *Role:* Listens for HTTP requests to trigger the workflow.  
  - *Configuration:* Path set to a unique webhook ID for secure and unique invocation.  
  - *Connections:* Outputs to "Normalize Webhook Trigger".  
  - *Edge Cases:* Incoming webhook requests may fail due to network or authentication issues; ensure webhook URL is correctly shared and active.  

- **Triggered from other Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Invokes this workflow from another workflow, passing specified inputs.  
  - *Configuration:* Defines inputs `someInput` and `someOtherInput` which can be passed downstream.  
  - *Connections:* Outputs directly to "Trigger | Normalized".  
  - *Edge Cases:* Requires correct credential and workflow ID configuration for seamless invocation; missing inputs may cause downstream issues.  

- **Normalize Manual Trigger**  
  - *Type:* Set  
  - *Role:* Passes data from Manual trigger while preserving all existing fields (`includeOtherFields: true`) without modification, acting as a normalization step.  
  - *Connections:* Inputs from "Manual"; outputs to "Trigger | Normalized".  
  - *Edge Cases:* No assignments; minimal risk, but unexpected manual input structure could propagate.  

- **Normalize Webhook Trigger**  
  - *Type:* Set  
  - *Role:* Similar to "Normalize Manual Trigger", passes webhook data forward in a normalized form, including all fields.  
  - *Connections:* Inputs from "General Webhook"; outputs to "Trigger | Normalized".  
  - *Edge Cases:* Possible missing or malformed webhook payloads; no explicit validation here.  

- **Trigger | Normalized**  
  - *Type:* Set  
  - *Role:* Acts as the universal normalized input node for all trigger types; strips binary data for safety and consistency (`stripBinary: true`).  
  - *Connections:* Inputs from normalization nodes; outputs to "Set Defaults".  
  - *Edge Cases:* Binary stripping may remove attachments; confirm if binary data is needed downstream.  

---

#### 1.2 Data Collection & Defaults Setting

**Overview:**  
Collects external data from Airtable and Google Sheets and applies default values for required parameters to ensure the workflow has all necessary input data.

**Nodes Involved:**  
- Set Defaults (Set)  
- Search records (Airtable)  
- Get Data From Google Sheets (Google Sheets)  
- Input | Combined (Merge)  

**Node Details:**

- **Set Defaults**  
  - *Type:* Set  
  - *Role:* Defines default values for `googleDocId` and `googleSheetId` to be used downstream if no overrides occur.  
  - *Parameters:*  
    - `googleDocId` = "docABC"  
    - `googleSheetId` = "sheetID123"  
  - *Connections:* Outputs to both "Search records" and "Get Data From Google Sheets".  
  - *Edge Cases:* Defaults can be overridden by upstream input; absence of defaults may cause API calls to fail.  

- **Search records**  
  - *Type:* Airtable  
  - *Role:* Searches records in a specified Airtable base and table.  
  - *Configuration:*  
    - Base: "Testing" (app4JwaENq0JKC1sm)  
    - Table: "User Date Dummy" (tblzRTsHDIBzqUnxY)  
    - Operation: Search with default options (no filters specified).  
  - *Credentials:* Airtable API token configured under "Airtable | Testing Base".  
  - *Connections:* Outputs to "Input | Combined".  
  - *Edge Cases:* Possible API limits or auth errors; empty results must be handled downstream.  

- **Get Data From Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Retrieves data from a specified Google Sheet document and sheet name.  
  - *Configuration:*  
    - Document ID and Sheet Name are dynamically set using expressions from input JSON (`{{$json.googleDocId}}` and `{{$json.googleSheetId}}`).  
  - *Credentials:* OAuth2 credentials configured under "n8n Google Sheets - OAuth2.0 v01".  
  - *Connections:* Outputs to "Input | Combined".  
  - *Edge Cases:* OAuth token expiry or permissions errors possible; invalid IDs cause errors.  

- **Input | Combined**  
  - *Type:* Merge  
  - *Role:* Combines the outputs of Airtable search and Google Sheets data retrieval into a single dataset using "combineAll" mode, aggregating all inputs.  
  - *Connections:* Inputs from "Search records" and "Get Data From Google Sheets"; outputs to "Switch".  
  - *Edge Cases:* Merging empty datasets results in empty output; ensure both sources are queried correctly.  

---

#### 1.3 Business Logic Processing

**Overview:**  
Executes distinct business logic paths based on the workflow execution mode (production, development, or fallback). Contains multiple logic implementations including legacy and in-development versions.

**Nodes Involved:**  
- Switch  
- Business Logic Single or Multi step (Code)  
- Business Logic Single or Multi step1 (Code)  
- Some Old Version (NoOp)  

**Node Details:**

- **Switch**  
  - *Type:* Switch  
  - *Role:* Routes workflow execution based on the current execution mode (`{{$execution.mode}}`), distinguishing between "production", "test" (development), and fallback ("extra").  
  - *Rules:*  
    - Output "Production" if mode equals "production" (case-insensitive)  
    - Output "Development" if mode equals "test" (case-insensitive)  
    - Otherwise, fallback output "extra"  
  - *Connections:* Outputs to three nodes corresponding to each mode.  
  - *Edge Cases:* Missing or unexpected execution mode values result in fallback path.  

- **Business Logic Single or Multi step**  
  - *Type:* Code  
  - *Role:* Placeholder for core business logic in production mode; currently returns a static JSON result.  
  - *Code:* Returns `{ result: "some use case specific output" }`  
  - *Connections:* Input from "Switch" (Production output).  
  - *Edge Cases:* Code node assumes static output; production logic needs to be implemented by users.  

- **Business Logic Single or Multi step1**  
  - *Type:* Code  
  - *Role:* Placeholder for development mode business logic; returns a similar static JSON.  
  - *Code:* Returns `{ result: "some use case specific output" }`  
  - *Connections:* Input from "Switch" (Development output).  
  - *Edge Cases:* Same as above; serves as a development sandbox.  

- **Some Old Version**  
  - *Type:* NoOp  
  - *Role:* Placeholder for legacy or old business logic preserved for reference; no operation performed.  
  - *Connections:* Input from "Switch" (fallback output).  
  - *Edge Cases:* Node does nothing; may be removed or replaced.  

---

#### 1.4 Workflow Metadata & Documentation

**Overview:**  
Provides extensive visual documentation using sticky notes with color coding for node states, functional guidelines, and best practices to facilitate team understanding and maintenance.

**Nodes Involved:**  
- Sticky Note (near triggers)  
- Sticky Note1 (start here instructions)  
- Sticky Note5 (data input gathering)  
- Sticky Note6 (Business Logic A)  
- Sticky Note7 (key objective and references)  
- Sticky Note8 (Business Logic B in development)  
- Sticky Note9 (color coding legend)  
- Sticky Note10 (Business Logic C old version)  

**Node Details:**

- **Sticky Note**  
  - Details the trigger normalization concept and convergence into a normalized node for universal downstream reference.  

- **Sticky Note1**  
  - Provides best practices for visual structuring, visual hierarchy, color coding usage, and design patterns. Encourages use as a template starting point.  

- **Sticky Note5**  
  - Explains the data collection phase, emphasizing setting defaults and gathering all required inputs—even if simple or empty in minimal workflows.  

- **Sticky Note6**  
  - Indicates "Business Logic A" is a working setup, representing stable business code.  

- **Sticky Note7**  
  - States the key objective of the workflow and suggests linking or referencing the triggering system for clarity.  

- **Sticky Note8**  
  - Marks "Business Logic B" as currently under development but with a working foundation.  

- **Sticky Note9**  
  - Provides a detailed legend of the color coding system used throughout the workflow:  
    - Green: Operational (stable)  
    - Red: Error / Not operational  
    - Orange: Partly operational (warnings or improvements needed)  
    - Yellow: Work in progress  
    - Blue: User input required at runtime  
    - Dark grey: On hold or deprecated  
    - White: Neutral / inherits meaning  

- **Sticky Note10**  
  - Notes "Business Logic C" as an old version not in use, kept temporarily for reference or versioning purposes.  

---

### 3. Summary Table

| Node Name                       | Node Type                     | Functional Role                             | Input Node(s)                       | Output Node(s)                            | Sticky Note                                                                                                      |
|--------------------------------|-------------------------------|--------------------------------------------|-----------------------------------|------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Manual                         | Manual Trigger                | Manual workflow trigger                     | -                                 | Normalize Manual Trigger                  |                                                                                                                 |
| General Webhook                | Webhook                      | HTTP webhook trigger                        | -                                 | Normalize Webhook Trigger                 |                                                                                                                 |
| Triggered from other Workflow  | Execute Workflow Trigger      | Triggered from another workflow             | -                                 | Trigger | Normalized                       | Make a note for triggering workflow(s) somewhere for easy debugging later.                                       |
| Normalize Manual Trigger       | Set                          | Normalize data from Manual trigger          | Manual                            | Trigger | Normalized                       | Fields are normalized with the Sub-workflow trigger as defining the "normalization" standard                      |
| Normalize Webhook Trigger      | Set                          | Normalize data from Webhook trigger         | General Webhook                   | Trigger | Normalized                       |                                                                                                                 |
| Trigger | Normalized           | Set                          | Universal normalized trigger data           | Normalize Manual Trigger, Normalize Webhook Trigger, Triggered from other Workflow | Set Defaults                             |                                                                                                                 |
| Set Defaults                  | Set                          | Set default parameters for external data   | Trigger | Normalized                   | Search records, Get Data From Google Sheets |                                                                                                                 |
| Search records                | Airtable                     | Search records in Airtable base & table    | Set Defaults                     | Input | Combined                          |                                                                                                                 |
| Get Data From Google Sheets   | Google Sheets                | Retrieve data from Google Sheets            | Set Defaults                     | Input | Combined                          |                                                                                                                 |
| Input | Combined             | Merge                        | Combine Airtable and Google Sheets data     | Search records, Get Data From Google Sheets | Switch                                |                                                                                                                 |
| Switch                       | Switch                       | Route execution based on workflow mode      | Input | Combined                   | Business Logic Single or Multi step, Business Logic Single or Multi step1, Some Old Version |                                                                                                                 |
| Business Logic Single or Multi step | Code                    | Production mode business logic placeholder  | Switch (Production output)        | -                                        |                                                                                                                 |
| Business Logic Single or Multi step1 | Code                   | Development mode business logic placeholder | Switch (Development output)       | -                                        |                                                                                                                 |
| Some Old Version             | NoOp                         | Legacy or deprecated business logic placeholder | Switch (fallback output)         | -                                        |                                                                                                                 |
| Sticky Note                  | Sticky Note                  | Documentation: Trigger normalization        | -                                 | -                                        | ## Triggers - Normalized Inputs related to Sub-Workflow Trigger; All triggers converge into a normalized node.   |
| Sticky Note1                 | Sticky Note                  | Documentation: Best practices & template usage | -                               | -                                        | # START HERE - Best practices for visual structuring & visual hierarchy; Color coding explained.                  |
| Sticky Note5                 | Sticky Note                  | Documentation: Data input gathering          | -                               | -                                        | ## Collect or gather all data inputs needed; Place to set any defaults.                                          |
| Sticky Note6                 | Sticky Note                  | Documentation: Business Logic A status       | -                               | -                                        | ## Business Logic A - working setup                                                                           |
| Sticky Note7                 | Sticky Note                  | Documentation: Workflow key objective        | -                               | -                                        | # Key Objective of overall Workflow; Link or reference to triggering system                                    |
| Sticky Note8                 | Sticky Note                  | Documentation: Business Logic B status       | -                               | -                                        | ## Business Logic B - currently in development; with working setup                                             |
| Sticky Note9                 | Sticky Note                  | Documentation: Color coding legend            | -                               | -                                        | ## Colorcoding for Nodes; Green=Operational; Red=Error; Orange=Partly Operational; Yellow=WIP; Blue=User Input  |
| Sticky Note10                | Sticky Note                  | Documentation: Business Logic C status       | -                               | -                                        | ## Business Logic C - Old Version, not used anymore; kept for reference or versioning                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Name: "Manual"  
   - Default settings, no parameters.  

2. **Create General Webhook Node**  
   - Node Type: Webhook  
   - Name: "General Webhook"  
   - Set unique webhook ID path (e.g., "f960a6c6-8916-44bd-aa90-b247d331afb8").  

3. **Create Execute Workflow Trigger Node**  
   - Node Type: Execute Workflow Trigger  
   - Name: "Triggered from other Workflow"  
   - Define inputs: `someInput`, `someOtherInput`.  

4. **Create Set Node for Normalize Manual Trigger**  
   - Node Type: Set  
   - Name: "Normalize Manual Trigger"  
   - Enable "Include Other Fields" (true).  
   - No explicit assignments.  
   - Connect "Manual" → "Normalize Manual Trigger".  

5. **Create Set Node for Normalize Webhook Trigger**  
   - Node Type: Set  
   - Name: "Normalize Webhook Trigger"  
   - Enable "Include Other Fields" (true).  
   - No explicit assignments.  
   - Connect "General Webhook" → "Normalize Webhook Trigger".  

6. **Create Set Node for Trigger | Normalized**  
   - Node Type: Set  
   - Name: "Trigger | Normalized"  
   - Enable "Include Other Fields" (true).  
   - Enable option to strip binary data (`stripBinary: true`).  
   - Connect outputs:  
     - "Normalize Manual Trigger" → "Trigger | Normalized"  
     - "Normalize Webhook Trigger" → "Trigger | Normalized"  
     - "Triggered from other Workflow" → "Trigger | Normalized"  

7. **Create Set Node for Set Defaults**  
   - Node Type: Set  
   - Name: "Set Defaults"  
   - Assign variables:  
     - `googleDocId` = "docABC"  
     - `googleSheetId` = "sheetID123"  
   - Connect "Trigger | Normalized" → "Set Defaults".  

8. **Create Airtable Node for Search Records**  
   - Node Type: Airtable  
   - Name: "Search records"  
   - Configure base: Select or enter Airtable base ID "app4JwaENq0JKC1sm" ("Testing").  
   - Configure table: Select or enter table ID "tblzRTsHDIBzqUnxY" ("User Date Dummy").  
   - Operation: "Search" with default options.  
   - Set credentials for Airtable API token.  
   - Connect "Set Defaults" → "Search records".  

9. **Create Google Sheets Node for Data Retrieval**  
   - Node Type: Google Sheets  
   - Name: "Get Data From Google Sheets"  
   - Document ID: Set with expression `{{$json.googleDocId}}`.  
   - Sheet Name: Set with expression `{{$json.googleSheetId}}`.  
   - Set OAuth2 credentials for Google Sheets.  
   - Connect "Set Defaults" → "Get Data From Google Sheets".  

10. **Create Merge Node for Input | Combined**  
    - Node Type: Merge  
    - Name: "Input | Combined"  
    - Mode: "combineAll" to merge all incoming data.  
    - Connect outputs:  
      - "Search records" → "Input | Combined"  
      - "Get Data From Google Sheets" → "Input | Combined"  

11. **Create Switch Node for Execution Mode Routing**  
    - Node Type: Switch  
    - Name: "Switch"  
    - Define rules:  
      - If `{{$execution.mode}}` equals "production" (case-insensitive), route to output "Production".  
      - If `{{$execution.mode}}` equals "test" (case-insensitive), route to output "Development".  
      - Fallback output named "extra".  
    - Connect "Input | Combined" → "Switch".  

12. **Create Code Node for Business Logic Production**  
    - Node Type: Code  
    - Name: "Business Logic Single or Multi step"  
    - JavaScript code:  
      ```javascript
      return [
        { json: { result: "some use case specific output" } }
      ];
      ```  
    - Connect "Switch" output "Production" → this node.  

13. **Create Code Node for Business Logic Development**  
    - Node Type: Code  
    - Name: "Business Logic Single or Multi step1"  
    - JavaScript code identical to production node (placeholder).  
    - Connect "Switch" output "Development" → this node.  

14. **Create NoOp Node for Legacy Business Logic**  
    - Node Type: NoOp  
    - Name: "Some Old Version"  
    - Connect "Switch" output "extra" → this node.  

15. **Add Sticky Notes for Documentation**  
    - Create multiple Sticky Note nodes with the content provided in the workflow, positioned near the relevant node groups for clarity.  
    - Include color coding legend and usage instructions as per "Sticky Note9" and "Sticky Note1".  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Color coding system for nodes: Green=Operational, Red=Error, Orange=Partial, Yellow=WIP, Blue=User Input, Dark Grey=Deprecated, White=Neutral. | Explained in Sticky Note9 in the workflow for team wide visual clarity.                          |
| Workflow serves as a reusable template for standardized and maintainable workflow design patterns. | Emphasized in Sticky Note1 as a starting point and best practice guide.                         |
| Trigger normalization is a key architectural pattern to unify diverse trigger inputs.               | Described in Sticky Note near triggers block.                                                  |
| Business logic nodes currently placeholders; users should replace with actual use-case implementations. | Noted in respective code nodes and sticky notes for business logic blocks.                      |
| Credentials required: Airtable API token and Google Sheets OAuth2 configured with appropriate scopes. | Critical for external data integration nodes to operate correctly.                              |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal, offensive, or protected elements. All data processed are legal and public.

---