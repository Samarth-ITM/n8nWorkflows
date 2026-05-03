Clone Nested Folder Structures in Google Drive with Custom Naming

https://n8nworkflows.xyz/workflows/clone-nested-folder-structures-in-google-drive-with-custom-naming-4349


# Clone Nested Folder Structures in Google Drive with Custom Naming

### 1. Workflow Overview

This workflow automates the cloning of nested folder structures within Google Drive, applying customizable naming conventions. It is designed for users who need to replicate complex folder hierarchies programmatically, such as for project templates, backups, or standardized directory setups. The workflow is structured into logical blocks that handle configuration, folder retrieval, iterative processing of folder data, conditional folder creation, and final reporting.

Logical blocks included:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Configuration Setup:** Setting and editing key parameters controlling the cloning process.
- **1.3 Folder Retrieval:** Fetching all folders from the specified Google Drive location.
- **1.4 Iterative Folder Processing:** Parsing and preparing folder data for cloning.
- **1.5 Conditional Folder Creation:** Deciding if and when to create new folders based on logic.
- **1.6 Folder Creation Execution:** Creating folders on Google Drive with custom naming.
- **1.7 Final Reporting:** Summarizing the cloning operation results.
- **1.8 Documentation and Support:** Sticky notes providing configuration guidance, processing info, and troubleshooting tips.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:** Initiates the workflow manually to start the cloning process.
- **Nodes Involved:** `When clicking "Test workflow"`
- **Node Details:**
  - Type: Manual Trigger
  - Configuration: No parameters; activated by manual user action.
  - Input/Output: No input; outputs trigger event to the next node.
  - Version: 1
  - Edge cases: None typical; user must manually trigger.
  - Sub-workflow: None

#### 2.2 Configuration Setup

- **Overview:** Holds configurable variables that dictate how the workflow operates, such as source folder ID, destination folder ID, and naming conventions.
- **Nodes Involved:** `⚙️ CONFIG - Edit the 3 variables above`
- **Node Details:**
  - Type: Code node
  - Configuration: Contains three user-editable variables to be set before execution.
  - Key variables likely include source folder ID, target folder ID, and custom naming patterns.
  - Input: Trigger from manual start node.
  - Output: Passes configuration data downstream.
  - Version: 2
  - Edge cases: Misconfiguration can cause failure in subsequent folder retrieval or creation.
  - Sub-workflow: None
  - Sticky Notes: Related guidance in "Config Help" sticky note.

#### 2.3 Folder Retrieval

- **Overview:** Requests all folder metadata from Google Drive to understand the existing folder structures.
- **Nodes Involved:** `📁 Get All Folders`
- **Node Details:**
  - Type: HTTP Request
  - Configuration: Set to call Google Drive API endpoint for listing folders; uses parameters from config node.
  - Input: Configuration data.
  - Output: JSON data containing folder list.
  - Version: 4.1
  - Edge cases: API rate limits, authentication failures, empty folder list.
  - Credentials: Google Drive OAuth2 (assumed).
  - Sticky Notes: Processing info may reference this node.

#### 2.4 Iterative Folder Processing

- **Overview:** Processes the retrieved folder list, preparing data for conditional creation and cloning logic.
- **Nodes Involved:** `🔄 Process Folders`
- **Node Details:**
  - Type: Code node
  - Configuration: Iterates through folder list, applies logic to determine folder hierarchy and naming.
  - Input: Folder list JSON from previous node.
  - Output: Processed folder data for decision making.
  - Version: 2
  - Edge cases: Parsing errors, unexpected folder metadata formats.
  - Sticky Notes: Processing info sticky note likely explains logic here.

#### 2.5 Conditional Folder Creation

- **Overview:** Decides whether a folder should be created based on criteria, such as existence or naming conflicts.
- **Nodes Involved:** `🔀 Should Create?`
- **Node Details:**
  - Type: If node
  - Configuration: Condition expressions evaluating if a folder needs creation.
  - Input: Processed folder data.
  - Output: If true, proceeds to create folder; else, skips.
  - Version: 2
  - Edge cases: Incorrect conditions could skip needed folders or cause duplicate creation.
  - Sticky Notes: Troubleshooting tips may relate here.

#### 2.6 Folder Creation Execution

- **Overview:** Creates folders on Google Drive with specified names and structure.
- **Nodes Involved:** `✨ Create Folder`
- **Node Details:**
  - Type: HTTP Request
  - Configuration: Calls Google Drive API to create folders with custom names.
  - Input: Folder data from conditional node.
  - Output: API response confirming creation.
  - Version: 4.1
  - Edge cases: API errors, permission denied, duplicate folder names.
  - Credentials: Google Drive OAuth2.
  - Sticky Notes: Troubleshooting sticky note may reference this node.

#### 2.7 Final Reporting

- **Overview:** Summarizes the results of the cloning operation for user review.
- **Nodes Involved:** `📊 Final Report`
- **Node Details:**
  - Type: Code node
  - Configuration: Aggregates responses and statuses into a summary report.
  - Input: Folder creation responses.
  - Output: Report data for display or further use.
  - Version: 2
  - Edge cases: Partial failures, empty results.
  - Sticky Notes: Troubleshooting and processing info may mention expected outputs.

#### 2.8 Documentation and Support

- **Overview:** Provides contextual notes to assist users in configuring and troubleshooting the workflow.
- **Nodes Involved:** `README`, `Config Help`, `Processing Info`, `Troubleshooting`
- **Node Details:**
  - Type: Sticky Notes
  - Configuration: Textual notes with instructions, tips, and troubleshooting advice.
  - Position: Strategically placed near relevant functional nodes.
  - Edge cases: None; informational only.
  - Sticky Notes Content: Not provided in the JSON but critical for user guidance.

---

### 3. Summary Table

| Node Name                   | Node Type        | Functional Role                | Input Node(s)                | Output Node(s)              | Sticky Note                                    |
|-----------------------------|------------------|-------------------------------|-----------------------------|-----------------------------|------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger   | Workflow initiation            |                             | ⚙️ CONFIG - Edit the 3 variables above |                                                |
| ⚙️ CONFIG - Edit the 3 variables above | Code             | Configuration setup            | When clicking "Test workflow" | 📁 Get All Folders           | Config Help (guidance on setting variables)   |
| 📁 Get All Folders            | HTTP Request     | Folder retrieval from Google Drive | ⚙️ CONFIG - Edit the 3 variables above | 🔄 Process Folders          | Processing Info (explains this processing step) |
| 🔄 Process Folders           | Code             | Iterative processing of folder data | 📁 Get All Folders           | 🔀 Should Create?            | Processing Info (describes iteration details) |
| 🔀 Should Create?            | If               | Conditional folder creation decision | 🔄 Process Folders           | ✨ Create Folder             | Troubleshooting (possible logic issues)        |
| ✨ Create Folder             | HTTP Request     | Google Drive folder creation   | 🔀 Should Create?            | 📊 Final Report             | Troubleshooting (API or permission errors)     |
| 📊 Final Report              | Code             | Summarize cloning results      | ✨ Create Folder             |                             | Troubleshooting (review operation outcome)     |
| README                      | Sticky Note      | Documentation                  |                             |                             |                                                |
| Config Help                 | Sticky Note      | Configuration guidance         |                             |                             |                                                |
| Processing Info             | Sticky Note      | Processing explanation         |                             |                             |                                                |
| Troubleshooting             | Sticky Note      | Troubleshooting tips           |                             |                             |                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Name: `When clicking "Test workflow"`
   - No parameters; used to start workflow manually.

2. **Add a Code Node for Configuration**
   - Name: `⚙️ CONFIG - Edit the 3 variables above`
   - Define three editable variables for:
     - Source folder ID (Google Drive folder to clone).
     - Destination folder ID (target folder for clone).
     - Custom naming pattern or flags.
   - Connect Manual Trigger node output to this node input.

3. **Add HTTP Request Node to Get All Folders**
   - Name: `📁 Get All Folders`
   - Configure to call Google Drive API endpoint for listing folders under the source folder ID.
   - Use Google Drive OAuth2 credentials.
   - Set method to GET, with query to list only folders.
   - Connect `⚙️ CONFIG` node output to this node input.

4. **Add Code Node for Iterative Folder Processing**
   - Name: `🔄 Process Folders`
   - Script to iterate over the folder list JSON.
   - Parse folder hierarchy and apply logic for custom naming.
   - Connect `📁 Get All Folders` output to this node input.

5. **Add If Node for Conditional Folder Creation**
   - Name: `🔀 Should Create?`
   - Condition to check if folder already exists or matches criteria.
   - Connect `🔄 Process Folders` output to this node input.
   - True branch leads to folder creation; false branch skips.

6. **Add HTTP Request Node to Create Folder**
   - Name: `✨ Create Folder`
   - Configure to call Google Drive API to create a folder.
   - Use Google Drive OAuth2 credentials.
   - POST method with body defining folder name and parent folder ID.
   - Connect true output of `🔀 Should Create?` to this node input.

7. **Add Code Node for Final Report**
   - Name: `📊 Final Report`
   - Aggregate successful and failed folder creation results.
   - Format a summary report.
   - Connect `✨ Create Folder` output to this node input.

8. **Add Sticky Notes for Documentation**
   - Create sticky notes near respective nodes:
     - `README`: General info.
     - `Config Help`: Instructions on setting variables.
     - `Processing Info`: Explains folder processing logic.
     - `Troubleshooting`: Common issues and fixes.

9. **Configure Credentials**
   - Set up Google Drive OAuth2 credentials accessible by HTTP Request nodes.
   - Validate API scopes include folder read/write permissions.

10. **Test Workflow**
    - Use manual trigger.
    - Review logs and final report to confirm folder cloning.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                  |
|-------------------------------------------------------------------------------------------------|--------------------------------|
| Workflow automates cloning nested Google Drive folders with custom naming.                      | Workflow purpose                |
| Google Drive API documentation: https://developers.google.com/drive/api/v3/reference/folders    | API reference                  |
| Ensure Google Drive OAuth2 credentials have folder read/write permissions.                       | Credential setup                |
| Troubleshooting common API errors: check rate limits, permissions, and folder ID correctness.   | Troubleshooting sticky note     |
| Custom naming can be adapted in the processing code node for flexible folder structure cloning. | Processing Info sticky note     |

---

**Disclaimer:**  
The provided text is derived solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.