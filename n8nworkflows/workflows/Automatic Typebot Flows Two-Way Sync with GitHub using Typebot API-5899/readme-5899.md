Automatic Typebot Flows Two-Way Sync with GitHub using Typebot API

https://n8nworkflows.xyz/workflows/automatic-typebot-flows-two-way-sync-with-github-using-typebot-api-5899


# Automatic Typebot Flows Two-Way Sync with GitHub using Typebot API

### 1. Workflow Overview

This workflow provides an automated two-way synchronization system between Typebot flows in a Typebot workspace and their backup JSON files stored in a GitHub repository. It continuously ensures that all Typebot flows are backed up to GitHub, updates changed flows, creates new backups for new flows, and deletes backup files on GitHub if the corresponding Typebot flow is deleted.

The workflow is logically divided into these main blocks:

- **1.1 Initialization & Triggering:** Starting points for manual or scheduled execution and loading global configuration.
- **1.2 Fetching & Preparing Data:** Listing all Typebot flows from the workspace and listing all backup files from GitHub.
- **1.3 Processing Typebot & GitHub Data:** Breaking down lists into individual items for detailed comparison.
- **1.4 Comparing Flows & Files:** Comparing local Typebot flows with their GitHub backup files to detect new, updated, unchanged, or deleted flows.
- **1.5 File Operations on GitHub:** Creating, updating, or deleting files in GitHub according to the comparison results.
- **1.6 Control Flow & Looping:** Managing batch processing and recursive workflow calls to handle all items efficiently and reduce memory usage.
- **1.7 Subworkflow Integration:** The workflow calls itself as a subworkflow to process items in batches to avoid memory overload.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Triggering

- **Overview:**  
  This block initializes the workflow execution either manually or on a schedule, and loads global configuration variables such as GitHub repo details and Typebot workspace info.

- **Nodes Involved:**  
  - On clicking 'execute'  
  - Schedule Trigger  
  - Globals

- **Node Details:**  
  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow for testing or on-demand backup.  
    - Inputs: None  
    - Outputs: Connects to Globals node  
    - Failure Modes: None expected; manual trigger  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Schedules daily backup runs at 7 AM server time.  
    - Configuration: Interval trigger at hour 7  
    - Inputs: None  
    - Outputs: Connects to Globals node  
    - Failure Modes: Scheduler downtime or misconfiguration  
  - **Globals**  
    - Type: Set  
    - Role: Defines global variables required for API calls and repo settings:  
      - `typebot.url` (Typebot API base URL)  
      - `typebot.workspace.id` (workspace ID)  
      - `repo.owner` (GitHub username)  
      - `repo.name` (GitHub repository name)  
    - Inputs: From triggers  
    - Outputs: Connects to List Typebots node  
    - Failure Modes: Misconfigured variables lead to API failures or wrong repo usage

#### 2.2 Fetching & Preparing Data

- **Overview:**  
  Fetches the list of all Typebot flows in the workspace and lists all backup files currently in the GitHub repository.

- **Nodes Involved:**  
  - List Typebots  
  - List files  
  - Split Out  
  - typebot  
  - github

- **Node Details:**  
  - **List Typebots**  
    - Type: HTTP Request  
    - Role: Calls Typebot API to list all flows in the workspace using Bearer token auth.  
    - URL: `${typebot.url}/api/v1/typebots?workspaceId=${typebot.workspace.id}`  
    - Outputs: JSON object containing all Typebot flows  
    - Failure Modes: Authentication errors, network issues, API rate limits  
  - **List files**  
    - Type: GitHub node  
    - Role: Lists all files in the GitHub repository root (backup JSON files).  
    - Auth: OAuth2 GitHub credentials  
    - Inputs: Globals node for repo owner and name  
    - Outputs: Array of files metadata  
    - Failure Modes: GitHub auth errors, repo access errors  
  - **Split Out**  
    - Type: Split Out  
    - Role: Extracts `typebots` array from List Typebots response to separate items for individual processing.  
    - Inputs: List Typebots  
    - Outputs: Individual Typebot flow objects  
  - **typebot**  
    - Type: Set  
    - Role: Adds contextual fields (`origin=typebot`) and re-attaches global config for each Typebot flow item.  
    - Inputs: Split Out  
    - Outputs: Feed into batch processing  
  - **github**  
    - Type: Set  
    - Role: Adds contextual fields (`origin=github`) and re-attaches global config with GitHub files array for batch processing.  
    - Inputs: List files  
    - Outputs: Feed into batch processing

#### 2.3 Processing Typebot & GitHub Data

- **Overview:**  
  Manages the batch splitting and looping over individual items from both Typebot and GitHub sources in parallel, to prepare for comparison.

- **Nodes Involved:**  
  - Loop Over Items  
  - Execute Workflow  
  - Execute Workflow Trigger  
  - Switch

- **Node Details:**  
  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes input items one by one (or in batches) to control memory and API usage.  
    - Configuration: Context variable `done` resets loop when complete  
    - Inputs: Set nodes (typebot, github)  
    - Outputs: Triggers Execute Workflow for each batch  
    - Failure Modes: Batch size mismanagement could cause slow processing or overload  
  - **Execute Workflow**  
    - Type: Execute Workflow  
    - Role: Calls the current workflow recursively as a subworkflow to handle each batch individually.  
    - Inputs: From Loop Over Items  
    - Outputs: Loops back to Loop Over Items for next batch  
  - **Execute Workflow Trigger**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for subworkflow calls; receives batch input and sends to Switch node.  
  - **Switch**  
    - Type: Switch  
    - Role: Directs data flow based on origin field (`typebot` or `github`) to appropriate processing nodes.  
    - Inputs: Execute Workflow Trigger  
    - Outputs:  
      - To Get Typebot for origin=typebot  
      - To isDeleted for origin=github

#### 2.4 Comparing Flows & Files

- **Overview:**  
  Compares each Typebot flow with the corresponding GitHub backup file to detect if the file is new, different, same, or deleted.

- **Nodes Involved:**  
  - Get Typebot  
  - Get file data  
  - Edit Fields  
  - Merge  
  - isDiffOrNew  
  - Check Status  
  - File is new  
  - File is different  
  - File is same  
  - isDeleted  
  - If  
  - Delete a file  
  - Return

- **Node Details:**  
  - **Get Typebot**  
    - Type: HTTP Request  
    - Role: Fetches detailed JSON data for a specific Typebot flow by ID.  
    - Inputs: Switch (origin=typebot)  
    - Outputs: Passes to Get file data and Edit Fields  
    - Failure Modes: API error, 404 if flow deleted  
  - **Get file data**  
    - Type: GitHub node  
    - Role: Retrieves the JSON backup file content from GitHub for matching flow ID.  
    - Inputs: Switch (origin=github)  
    - Outputs: Merges with Edit Fields output  
    - Failure Modes: File not found, GitHub API errors  
    - Configured to continue on fail to handle missing files gracefully  
  - **Edit Fields**  
    - Type: Set  
    - Role: Formats the Typebot flow JSON for comparison, outputting in raw JSON format  
    - Inputs: Get Typebot  
    - Outputs: Merge node  
  - **Merge**  
    - Type: Merge  
    - Role: Combines GitHub file data and Typebot flow data for comparison  
  - **isDiffOrNew**  
    - Type: Code  
    - Role:  
      - Orders JSON keys to normalize both datasets  
      - Compares the original GitHub file content with the current Typebot flow JSON  
      - Determines if the file is 'same', 'different', or 'new'  
      - Prepares formatted JSON string for other nodes  
    - Inputs: Merge outputs  
  - **Check Status**  
    - Type: Switch  
    - Role: Routes flow based on comparison result stored in `github_status` ('new', 'different', 'same')  
  - **File is new**, **File is different**, **File is same**  
    - Type: NoOp  
    - Role: Logical placeholders for branching; trigger relevant GitHub file operations or early return  
  - **isDeleted**  
    - Type: Code  
    - Role: Checks if a GitHub backup file corresponds to any existing Typebot flow; flags if deleted  
  - **If**  
    - Type: If  
    - Role: Executes deletion if the file is flagged as deleted (`isDeleted=true`)  
  - **Delete a file**  
    - Type: GitHub node  
    - Role: Deletes the backup file from GitHub repository for flows deleted in Typebot workspace  
  - **Return**  
    - Type: Set  
    - Role: Marks completion of processing for the current item (sets `Done = true`)

#### 2.5 File Operations on GitHub

- **Overview:**  
  Executes GitHub file create, update, or delete operations based on the comparison results.

- **Nodes Involved:**  
  - Create new file  
  - Edit existing file  
  - Delete a file

- **Node Details:**  
  - **Create new file**  
    - Type: GitHub node  
    - Role: Creates a new JSON file in the GitHub repository for flows not yet backed up.  
    - Inputs: File content prepared in isDiffOrNew node  
    - Commit message includes flow name and status  
  - **Edit existing file**  
    - Type: GitHub node  
    - Role: Updates existing file content with the latest Typebot flow JSON if differences detected.  
    - Same commit message logic as create  
  - **Delete a file**  
    - Covered previously under 2.4  
  - All GitHub nodes use OAuth2 credentials and require write access to the repository.

#### 2.6 Control Flow & Looping

- **Overview:**  
  Controls the looping mechanism to process all flows and files efficiently using batch processing and recursive calls.

- **Nodes Involved:**  
  - Loop Over Items  
  - Execute Workflow

- **Node Details:**  
  - Covered in 2.3; these nodes ensure that the workflow processes the entire dataset in manageable batches, avoiding memory overload and API rate limits.

#### 2.7 Subworkflow Integration

- **Overview:**  
  The workflow invokes itself recursively as a subworkflow to handle batches of items, improving scalability and reliability.

- **Nodes Involved:**  
  - Execute Workflow  
  - Execute Workflow Trigger

- **Node Details:**  
  - The main workflow loops over items and calls itself with smaller input subsets.  
  - Subworkflow entry point is Execute Workflow Trigger, which routes data based on origin.  
  - This design reduces memory usage and allows processing large numbers of flows and files.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                                   | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                                                     |
|-----------------------|-------------------------|--------------------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger          | Manual start trigger                              | None                          | Globals                        |                                                                                                                                |
| Schedule Trigger       | Schedule Trigger        | Scheduled daily trigger at 7 AM                   | None                          | Globals                        |                                                                                                                                |
| Globals               | Set                     | Defines global config vars for API and repo       | On clicking 'execute', Schedule Trigger | List Typebots                 | "Open `Globals` node and update repo.owner, repo.name, typebot.url, workspace.id" [Typebot API docs](https://docs.typebot.io/api-reference/how-to) |
| List Typebots          | HTTP Request            | Lists all Typebot flows in workspace               | Globals                       | List files, Split Out          |                                                                                                                                |
| List files             | GitHub                  | Lists all files in GitHub repo                      | List Typebots                 | github                        |                                                                                                                                |
| Split Out              | Split Out               | Splits Typebot list into individual flow items     | List Typebots                 | typebot                       |                                                                                                                                |
| typebot                | Set                     | Adds origin and repo info to Typebot items         | Split Out                    | Loop Over Items               |                                                                                                                                |
| github                 | Set                     | Adds origin and repo info to GitHub files          | List files                   | Loop Over Items               |                                                                                                                                |
| Loop Over Items        | Split In Batches        | Splits items into batches for processing            | typebot, github              | Execute Workflow              |                                                                                                                                |
| Execute Workflow       | Execute Workflow        | Calls workflow recursively for batch processing    | Loop Over Items              | Loop Over Items               |                                                                                                                                |
| Execute Workflow Trigger | Execute Workflow Trigger | Receives batch input for subworkflow                | Execute Workflow             | Switch                       |                                                                                                                                |
| Switch                 | Switch                  | Routes based on origin (typebot/github)             | Execute Workflow Trigger     | Get Typebot, isDeleted        |                                                                                                                                |
| Get Typebot            | HTTP Request            | Fetches detailed Typebot flow JSON                  | Switch (origin=typebot)      | Get file data, Edit Fields    |                                                                                                                                |
| Get file data          | GitHub                  | Fetches corresponding GitHub backup file            | Switch (origin=github)       | Merge                        |                                                                                                                                |
| Edit Fields            | Set                     | Prepares Typebot flow JSON for comparison           | Get Typebot                  | Merge                        |                                                                                                                                |
| Merge                  | Merge                   | Combines GitHub file and Typebot flow data          | Get file data, Edit Fields   | isDiffOrNew                  |                                                                                                                                |
| isDiffOrNew            | Code                    | Compares flows, flags new/different/same statuses   | Merge                        | Check Status                 |                                                                                                                                |
| Check Status           | Switch                  | Routes based on comparison result                    | isDiffOrNew                  | File is new/different/same   |                                                                                                                                |
| File is new            | NoOp                    | Placeholder for new files                            | Check Status                 | Create new file              |                                                                                                                                |
| File is different      | NoOp                    | Placeholder for updated files                        | Check Status                 | Edit existing file           |                                                                                                                                |
| File is same           | NoOp                    | Placeholder for same files                           | Check Status                 | Return                       |                                                                                                                                |
| Create new file        | GitHub                  | Creates new backup file in GitHub                    | File is new                  | Return                       |                                                                                                                                |
| Edit existing file     | GitHub                  | Updates existing backup file in GitHub               | File is different            | Return                       |                                                                                                                                |
| isDeleted              | Code                    | Checks if GitHub file has no matching Typebot flow  | Switch (origin=github)       | If                          |                                                                                                                                |
| If                     | If                      | Deletes file if flagged as deleted                    | isDeleted                   | Delete a file, Return        |                                                                                                                                |
| Delete a file          | GitHub                  | Deletes a backup file from GitHub                     | If                          | Return                       |                                                                                                                                |
| Return                 | Set                     | Marks completion of processing for an item           | File is same, Delete a file, Create new file, Edit existing file | None                        |                                                                                                                                |
| Sticky Note1           | Sticky Note             | Overview and setup instructions                      | None                        | None                        | "Typebot Backup to GitHub. Setup repo.owner, repo.name, typebot.url, workspace.id. Files named ID.json. Calls itself as subworkflow." |
| Sticky Note            | Sticky Note             | Marks main workflow loop                              | None                        | None                        | "Main workflow loop"                                                                                                           |
| Sticky Note2           | Sticky Note             | Marks subworkflow section                             | None                        | None                        | "Subworkflow"                                                                                                                  |
| Sticky Note4           | Sticky Note             | Indicates editable node                               | None                        | None                        | "Edit this node 👇"                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "On clicking 'execute'"  
   - Type: Manual Trigger

2. **Create Schedule Trigger Node:**  
   - Name: "Schedule Trigger"  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 7 AM.

3. **Create Globals Node:**  
   - Type: Set  
   - Define variables:  
     - `typebot.url` (e.g., "https://typebot.io")  
     - `typebot.workspace.id` (your Typebot workspace ID)  
     - `repo.owner` (your GitHub username)  
     - `repo.name` (your GitHub repo name)  
   - Connect both triggers ("On clicking 'execute'" and "Schedule Trigger") to this node.

4. **Create List Typebots Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `{{ $json.typebot.url }}/api/v1/typebots`  
   - Query Parameter: `workspaceId={{ $json.typebot.workspace.id }}`  
   - Authentication: HTTP Bearer with Typebot API token  
   - Connect Globals output to this node.

5. **Create List files Node:**  
   - Type: GitHub node  
   - Operation: List files  
   - Repository owner: `{{ $json.repo.owner }}`  
   - Repository name: `{{ $json.repo.name }}`  
   - Authentication: OAuth2 GitHub credentials with repo access  
   - Connect List Typebots node output to this node.

6. **Create Split Out Node:**  
   - Type: Split Out  
   - Field to split: `typebots` (from List Typebots response)  
   - Connect List Typebots node output to this node.

7. **Create typebot Set Node:**  
   - Type: Set  
   - Assign:  
     - `origin` = "typebot"  
     - `repo.owner`, `repo.name`, `typebot.url` from Globals  
   - Connect Split Out node output to this node.

8. **Create github Set Node:**  
   - Type: Set  
   - Assign:  
     - `origin` = "github"  
     - `repo.owner`, `repo.name`, `typebot.url` from Globals  
     - `typebots` = files array from List files node  
   - Connect List files node output to this node.

9. **Create Loop Over Items Node:**  
   - Type: Split In Batches  
   - Default batch size (optional)  
   - Connect typebot and github Set nodes outputs to this node (merge inputs).

10. **Create Execute Workflow Node:**  
    - Type: Execute Workflow  
    - Mode: each  
    - Workflow ID: current workflow's ID for recursive calls  
    - Connect Loop Over Items node output to this node.

11. **Create Execute Workflow Trigger Node:**  
    - Type: Execute Workflow Trigger  
    - Input source: passthrough  
    - Entry point for subworkflow calls.

12. **Create Switch Node:**  
    - Type: Switch  
    - Condition: `$json.origin == "typebot"` → output 1  
    - `$json.origin == "github"` → output 2  
    - Connect Execute Workflow Trigger output to this node.

13. **Create Get Typebot Node:**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `{{ $json.typebot.url }}/api/v1/typebots/{{ $json.id }}`  
    - Authentication: HTTP Bearer with Typebot API token  
    - Connect Switch output 1 to this node.

14. **Create Get file data Node:**  
    - Type: GitHub node  
    - Operation: Get file  
    - File path: `{{ $json.id }}.json`  
    - Repository owner and name from JSON  
    - Authentication: OAuth2 GitHub credentials  
    - Continue on fail enabled  
    - Connect Switch output 2 to this node.

15. **Create Edit Fields Node:**  
    - Type: Set  
    - Mode: Raw  
    - JSON Output: `{{ $json.typebot }}`  
    - Connect Get Typebot output to this node.

16. **Create Merge Node:**  
    - Type: Merge  
    - Mode: Merge by position or key  
    - Connect Get file data and Edit Fields outputs to Merge node.

17. **Create isDiffOrNew Node:**  
    - Type: Code  
    - JavaScript logic to:  
      - Order JSON keys in both GitHub file and Typebot flow JSON  
      - Compare for equality  
      - Set `github_status` to `same`, `different`, or `new`  
      - Prepare JSON string for file content if different or new  
    - Connect Merge node output to this node.

18. **Create Check Status Node:**  
    - Type: Switch  
    - Conditions on `github_status`:  
      - "new" → output 1  
      - "different" → output 2  
      - "same" → output 3  
    - Connect isDiffOrNew output to this node.

19. **Create File is new, File is different, File is same Nodes:**  
    - Type: NoOp  
    - Connect Check Status outputs respectively to these nodes.

20. **Create Create new file Node:**  
    - Type: GitHub node  
    - Operation: Create file  
    - File Path: `{{ $json.id }}.json`  
    - File Content: JSON string from isDiffOrNew  
    - Commit Message: `{{ $json.name }} (new)`  
    - Authentication: OAuth2 GitHub credentials  
    - Connect File is new node output to this node.

21. **Create Edit existing file Node:**  
    - Type: GitHub node  
    - Operation: Edit file  
    - Similar config as Create new file  
    - Commit Message: `{{ $json.name }} (different)`  
    - Connect File is different node output to this node.

22. **Create Return Node:**  
    - Type: Set  
    - Set `Done = true` boolean  
    - Connect File is same, Create new file, Edit existing file outputs to this node.

23. **Create isDeleted Node:**  
    - Type: Code  
    - Logic: Checks if GitHub file name (without .json) exists in current Typebot flows  
    - Returns `isDeleted` boolean  
    - Connect Switch output 2 to this node (for GitHub files).

24. **Create If Node:**  
    - Type: If  
    - Condition: `isDeleted == true`  
    - Connect isDeleted output to If node.

25. **Create Delete a file Node:**  
    - Type: GitHub node  
    - Operation: Delete file  
    - File Path: file name from GitHub file list  
    - Commit Message: `${fileName} (deleted)`  
    - Authentication: OAuth2 GitHub credentials  
    - Connect If node true output to this node.

26. **Connect Delete a file and If node false output to Return node.**

27. **Link Return node to Loop Over Items node to continue batch processing.**

28. **Add Sticky Notes:**  
    - Add informative sticky notes for setup instructions, main workflow loop, subworkflow, and editable node markers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow automates backing up Typebot flows to GitHub with two-way sync, including deletion detection.                                                                                                                                 | Workflow purpose summary                                                                                         |
| Setup requires entering GitHub username/repository and Typebot API URL/workspace ID in the Globals node before running.                                                                                                                   | Globals node configuration                                                                                        |
| Files are saved in GitHub repository as `<flow_id>.json` files.                                                                                                                                                                            | File naming convention                                                                                           |
| The workflow uses batch processing and recursive calls (subworkflow) to handle large datasets while managing memory and API rate limits.                                                                                                | Workflow design for scalability                                                                                  |
| Typebot API Documentation: https://docs.typebot.io/api-reference/how-to                                                                                                                                                                   | Official Typebot API docs                                                                                        |
| GitHub OAuth2 credentials must have repository write access for file operations to succeed.                                                                                                                                                 | Credential requirements                                                                                          |
| The workflow includes detailed error handling strategies: GitHub file fetch node continues on fail to handle missing files, and deletion checks avoid deleting existing flows.                                                              | Error handling summary                                                                                           |
| Sticky notes contain helpful setup instructions and mark important workflow sections for ease of understanding.                                                                                                                           | Workflow annotations                                                                                             |

---

**Disclaimer:** The provided text exclusively originates from an automated n8n workflow. It conforms fully with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.