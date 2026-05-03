Backup Clockify to Github based on monthly reports

https://n8nworkflows.xyz/workflows/backup-clockify-to-github-based-on-monthly-reports-3147


# Backup Clockify to Github based on monthly reports

### 1. Workflow Overview

This workflow automates the backup of Clockify workspace data into a private GitHub repository by generating monthly detailed reports. It is designed to run daily and maintain versioned backups for the current month and the previous two months by default. The workflow fetches detailed time entry reports from Clockify’s Reports API, compares them with existing GitHub files, and updates or creates files only if changes are detected.

**Logical Blocks:**

- **1.1 Trigger and Initialization:** Schedule trigger and workspace identification.
- **1.2 Global Variables and Scope Setup:** Define workspace ID, GitHub repo details, and month indexes for backup scope.
- **1.3 Monthly Report Preparation:** Calculate date ranges and report file names for each month in scope.
- **1.4 Report Retrieval and Existence Check:** Fetch detailed monthly reports from Clockify and check if corresponding files exist in GitHub.
- **1.5 Data Comparison and Decision:** Compare newly fetched report data with existing GitHub file content.
- **1.6 GitHub File Operations:** Create or update report files in GitHub based on comparison results.
- **1.7 Error Handling:** Manage errors such as missing files or API failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block triggers the workflow daily and retrieves the first available Clockify workspace to use its ID for subsequent API calls.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get first workspace

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution daily at 5:00 AM.  
    - Configuration: Runs once per day at hour 5.  
    - Inputs: None (trigger node)  
    - Outputs: Triggers "Get first workspace" node.  
    - Edge Cases: Trigger misconfiguration could cause missed runs.

  - **Get first workspace**  
    - Type: Clockify API node  
    - Role: Retrieves the first workspace available in the Clockify account.  
    - Configuration: Limits results to 1 workspace.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: Passes workspace data to "Globals" node.  
    - Credentials: Requires Clockify API credentials with workspace read permissions.  
    - Edge Cases: No workspace found, API authentication failure, rate limits.

---

#### 2.2 Global Variables and Scope Setup

- **Overview:**  
  Sets global variables including workspace ID, GitHub repository owner and name, and defines the backup scope as an array of month indexes (0=current month, 1=last month, 2=two months ago).

- **Nodes Involved:**  
  - Globals  
  - Set month indexes  
  - Split Out indexes

- **Node Details:**

  - **Globals**  
    - Type: Set node  
    - Role: Defines key global variables for the workflow.  
    - Configuration:  
      - `workspace_id`: Set from the workspace data JSON (`$json.id`).  
      - `github_repo.owner`: Empty string placeholder to be filled by user.  
      - `github_repo.name`: Empty string placeholder to be filled by user.  
    - Inputs: Workspace data from "Get first workspace".  
    - Outputs: To "Set month indexes".  
    - Edge Cases: Missing or incorrect repo details will cause GitHub API failures.

  - **Set month indexes**  
    - Type: Set node  
    - Role: Defines the array `[0,1,2]` representing months to back up.  
    - Configuration: Assigns `monthIndex` array `[0,1,2]`.  
    - Inputs: From "Globals".  
    - Outputs: To "Split Out indexes".  
    - Edge Cases: Modifying this array changes backup scope.

  - **Split Out indexes**  
    - Type: Split Out node  
    - Role: Splits the array of month indexes into individual items for iteration.  
    - Configuration: Splits on field `monthIndex`.  
    - Inputs: From "Set month indexes".  
    - Outputs: To "Set intervals".  
    - Edge Cases: Empty array would halt downstream processing.

---

#### 2.3 Monthly Report Preparation

- **Overview:**  
  For each month index, calculates the report file name and the start and end dates for the monthly report.

- **Nodes Involved:**  
  - Set intervals

- **Node Details:**

  - **Set intervals**  
    - Type: Set node  
    - Role: Calculates date ranges and report file names for each month index.  
    - Configuration:  
      - `reportName`: `"detailed_report_YYYY-MM"` based on current date minus `monthIndex` months.  
      - `startDate`: First day of the month in `YYYY-MM-DD` format.  
      - `endDate`: Last day of the month in `YYYY-MM-DD` format.  
    - Inputs: Individual month index from "Split Out indexes".  
    - Outputs: To "Get detailed monthly report".  
    - Expressions: Uses `$now.minus($json.monthIndex, 'month')` and date formatting.  
    - Edge Cases: Date calculations must handle month boundaries correctly.

---

#### 2.4 Report Retrieval and Existence Check

- **Overview:**  
  Retrieves detailed monthly reports from Clockify API and checks if a corresponding report file exists in the GitHub repository.

- **Nodes Involved:**  
  - Get detailed monthly report  
  - Check if file exists in GitHub

- **Node Details:**

  - **Get detailed monthly report**  
    - Type: HTTP Request node  
    - Role: Calls Clockify Reports API to get detailed time entries for the month.  
    - Configuration:  
      - POST request to `https://reports.api.clockify.me/v1/workspaces/{workspace_id}/reports/detailed`  
      - JSON body includes `dateRangeStart`, `dateRangeEnd`, pagination, and export type.  
      - Uses Clockify API credentials.  
    - Inputs: From "Set intervals".  
    - Outputs: To "Check if file exists in GitHub".  
    - Edge Cases: API rate limits, invalid dates, authentication errors.

  - **Check if file exists in GitHub**  
    - Type: GitHub node  
    - Role: Checks if the monthly report file exists in the GitHub repository.  
    - Configuration:  
      - Operation: Get file  
      - File path: `reports/{reportName}`  
      - Repository owner and name from Globals.  
      - On error: Continue with error output (to handle 404).  
      - Uses GitHub credentials with read permissions.  
    - Inputs: From "Get detailed monthly report".  
    - Outputs:  
      - Success: To "Point to new data" and "Extract from File" (for comparison).  
      - Error (404): To "Check for 404 error message".  
    - Edge Cases: Network errors, permission issues, file not found (404).

---

#### 2.5 Data Comparison and Decision

- **Overview:**  
  Compares the newly fetched report data with the existing file content from GitHub to determine if an update is needed.

- **Nodes Involved:**  
  - Point to new data  
  - Extract from File  
  - Compare Datasets

- **Node Details:**

  - **Point to new data**  
    - Type: Set node  
    - Role: Extracts the `timeentries` array from the Clockify API response and assigns it to `data`.  
    - Configuration: Sets `data` to `$('Get detailed monthly report').item.json.timeentries`.  
    - Inputs: From "Check if file exists in GitHub" (success path).  
    - Outputs: To "Compare Datasets" (first input).  
    - Edge Cases: Missing or empty `timeentries` array.

  - **Extract from File**  
    - Type: Extract From File node  
    - Role: Parses the existing GitHub file content (JSON) to extract data for comparison.  
    - Configuration: Operation "fromJson".  
    - Inputs: From "Check if file exists in GitHub" (success path).  
    - Outputs: To "Compare Datasets" (second input).  
    - Edge Cases: Malformed JSON in GitHub file.

  - **Compare Datasets**  
    - Type: Compare Datasets node  
    - Role: Compares new data (`data`) with existing data (`data`) from GitHub file.  
    - Configuration: Merges datasets by the field `data`.  
    - Inputs:  
      - First input: New data from "Point to new data".  
      - Second input: Existing data from "Extract from File".  
    - Outputs:  
      - True path: To "Update file in GitHub" (data changed).  
      - False path: No output (no changes).  
      - Third output: Not used.  
    - Edge Cases: Comparison failures if data structures differ.

---

#### 2.6 GitHub File Operations

- **Overview:**  
  Creates new report files or updates existing ones in GitHub based on whether the report data is new or changed.

- **Nodes Involved:**  
  - Check for 404 error message  
  - Skip empty reports  
  - Create file in GitHub  
  - Update file in GitHub  
  - Stop and Error

- **Node Details:**

  - **Check for 404 error message**  
    - Type: If node  
    - Role: Checks if the error from "Check if file exists in GitHub" contains "could not be found" (404).  
    - Configuration: String contains check on `$json.error`.  
    - Inputs: From "Check if file exists in GitHub" (error path).  
    - Outputs:  
      - True: To "Skip empty reports" (file does not exist).  
      - False: To "Stop and Error" (other errors).  
    - Edge Cases: Other error messages cause workflow to stop.

  - **Skip empty reports**  
    - Type: Filter node  
    - Role: Filters out reports with empty `timeentries` arrays to avoid creating empty files.  
    - Configuration: Checks that `$json.timeentries` is not empty.  
    - Inputs: From "Check for 404 error message" (true path).  
    - Outputs:  
      - True: To "Create file in GitHub".  
      - False: No output (skips empty reports).  
    - Edge Cases: Empty reports are ignored to avoid clutter.

  - **Create file in GitHub**  
    - Type: GitHub node  
    - Role: Creates a new report file in GitHub for the month.  
    - Configuration:  
      - Operation: Create file  
      - File path: `reports/{reportName}`  
      - File content: JSON stringified `timeentries` array.  
      - Commit message: "Create report"  
      - Uses GitHub credentials with write permissions.  
    - Inputs: From "Skip empty reports".  
    - Outputs: None (end).  
    - Edge Cases: Permission errors, file creation conflicts.

  - **Update file in GitHub**  
    - Type: GitHub node  
    - Role: Updates existing report file in GitHub if data has changed.  
    - Configuration:  
      - Operation: Edit file  
      - File path: `reports/{reportName}`  
      - File content: JSON stringified new data (`data`).  
      - Commit message: "Update report"  
      - Uses GitHub credentials with write permissions.  
    - Inputs: From "Compare Datasets" (true path).  
    - Outputs: None (end).  
    - Edge Cases: Conflicts if file changed externally, permission errors.

  - **Stop and Error**  
    - Type: Stop and Error node  
    - Role: Stops workflow execution and reports error messages for unexpected failures.  
    - Configuration: Error message set dynamically from `$json.error`.  
    - Inputs: From "Check for 404 error message" (false path).  
    - Outputs: None (terminates workflow).  
    - Edge Cases: Ensures workflow fails visibly on unexpected errors.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                                | Input Node(s)               | Output Node(s)                        | Sticky Note                                                                                         |
|----------------------------|-----------------------|-----------------------------------------------|-----------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger      | Triggers workflow daily at 5:00 AM             | None                        | Get first workspace                  | ## Set trigger<br>By default this workflow runs once a day.                                       |
| Get first workspace        | Clockify API          | Retrieves first Clockify workspace              | Schedule Trigger            | Globals                             | ## Set Globals<br>- Define the repository owner (username / organization) and repository name<br>- By default the fist available Clockify workspace ID is set. This can be overridden here. |
| Globals                   | Set                   | Sets workspace ID and GitHub repo details       | Get first workspace         | Set month indexes                   | ## Set Globals<br>- Define the repository owner (username / organization) and repository name<br>- By default the fist available Clockify workspace ID is set. This can be overridden here. |
| Set month indexes          | Set                   | Defines months to back up (0=current,1,last,2) | Globals                     | Split Out indexes                   | ## Set Scope (optional)<br>By default the last three moths are being backed up.<br>_0 = current month, 1 = last month, etc._ |
| Split Out indexes          | Split Out             | Splits month indexes array into individual items| Set month indexes           | Set intervals                      | ## Set Scope (optional)<br>By default the last three moths are being backed up.<br>_0 = current month, 1 = last month, etc._ |
| Set intervals              | Set                   | Calculates report names and date ranges         | Split Out indexes           | Get detailed monthly report         | ## Set Scope (optional)<br>By default the last three moths are being backed up.<br>_0 = current month, 1 = last month, etc._ |
| Get detailed monthly report| HTTP Request          | Fetches detailed monthly report from Clockify   | Set intervals               | Check if file exists in GitHub       | A detailed report is being retrieved for every month across all entries in the workspace.          |
| Check if file exists in GitHub | GitHub              | Checks if report file exists in GitHub           | Get detailed monthly report | Point to new data, Extract from File, Check for 404 error message | The reports are created or updated in GitHub.<br>**It is essential to back up previous months as well, as values like tags may still change over time.** |
| Point to new data          | Set                   | Extracts new report data for comparison          | Check if file exists in GitHub | Compare Datasets                  | The reports are created or updated in GitHub.<br>**It is essential to back up previous months as well, as values like tags may still change over time.** |
| Extract from File          | Extract From File     | Parses existing GitHub file content               | Check if file exists in GitHub | Compare Datasets                  | The reports are created or updated in GitHub.<br>**It is essential to back up previous months as well, as values like tags may still change over time.** |
| Compare Datasets           | Compare Datasets      | Compares new and existing report data             | Point to new data, Extract from File | Update file in GitHub           | The reports are created or updated in GitHub.<br>**It is essential to back up previous months as well, as values like tags may still change over time.** |
| Update file in GitHub      | GitHub                | Updates existing report file in GitHub            | Compare Datasets            | None                              | The reports are created or updated in GitHub.<br>**It is essential to back up previous months as well, as values like tags may still change over time.** |
| Check for 404 error message| If                    | Checks if GitHub file not found (404)             | Check if file exists in GitHub (error) | Skip empty reports, Stop and Error |                                                                                                   |
| Skip empty reports         | Filter                | Filters out empty reports to avoid empty files    | Check for 404 error message | Create file in GitHub              |                                                                                                   |
| Create file in GitHub      | GitHub                | Creates new report file in GitHub                   | Skip empty reports          | None                              |                                                                                                   |
| Stop and Error             | Stop and Error        | Stops workflow on unexpected errors                | Check for 404 error message | None                              |                                                                                                   |
| Sticky Note3               | Sticky Note           | Instructions for Globals node                       | None                       | None                              | ## Set Globals<br>- Define the repository owner (username / organization) and repository name<br>- By default the fist available Clockify workspace ID is set. This can be overridden here. |
| Sticky Note4               | Sticky Note           | Instructions for Schedule Trigger                   | None                       | None                              | ## Set trigger<br>By default this workflow runs once a day.                                       |
| Sticky Note                | Sticky Note           | Instructions for month indexes and scope           | None                       | None                              | ## Set Scope (optional)<br>By default the last three moths are being backed up.<br>_0 = current month, 1 = last month, etc._ |
| Sticky Note1               | Sticky Note           | Explains detailed report retrieval                   | None                       | None                              | A detailed report is being retrieved for every month across all entries in the workspace.          |
| Sticky Note2               | Sticky Note           | Explains GitHub file creation/update importance     | None                       | None                              | The reports are created or updated in GitHub.<br>**It is essential to back up previous months as well, as values like tags may still change over time.** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to run daily at 5:00 AM.

2. **Add a Clockify node to get first workspace:**  
   - Type: Clockify API node  
   - Resource: Workspace  
   - Limit: 1  
   - Connect input from Schedule Trigger.  
   - Configure Clockify API credentials with read access.

3. **Add a Set node named "Globals":**  
   - Inputs: From "Get first workspace".  
   - Assign variables:  
     - `workspace_id` = `{{$json["id"]}}` (workspace ID from previous node)  
     - `github_repo.owner` = `""` (to be filled with your GitHub username or org)  
     - `github_repo.name` = `""` (to be filled with your GitHub repository name)

4. **Add a Set node named "Set month indexes":**  
   - Inputs: From "Globals".  
   - Assign variable:  
     - `monthIndex` = `[0, 1, 2]` (array of months to back up)

5. **Add a Split Out node named "Split Out indexes":**  
   - Inputs: From "Set month indexes".  
   - Field to split out: `monthIndex`

6. **Add a Set node named "Set intervals":**  
   - Inputs: From "Split Out indexes".  
   - Assign variables using expressions:  
     - `reportName` = `"detailed_report_{{ $now.minus($json.monthIndex, 'month').format('yyyy-MM') }}"`  
     - `startDate` = `"{{$now.minus($json.monthIndex, 'month').startOf('month').format('yyyy-MM-dd')}}"`  
     - `endDate` = `"{{$now.minus($json.monthIndex, 'month').endOf('month').format('yyyy-MM-dd')}}"`

7. **Add an HTTP Request node named "Get detailed monthly report":**  
   - Inputs: From "Set intervals".  
   - Method: POST  
   - URL: `https://reports.api.clockify.me/v1/workspaces/{{$json.workspace_id}}/reports/detailed`  
   - Body (JSON):  
     ```json
     {
       "dateRangeStart": "{{$json.startDate}}T00:00:00Z",
       "dateRangeEnd": "{{$json.endDate}}T23:59:59.999Z",
       "detailedFilter": {
         "page": 1,
         "pageSize": 50
       },
       "exportType": "json"
     }
     ```  
   - Authentication: Use Clockify API credentials.

8. **Add a GitHub node named "Check if file exists in GitHub":**  
   - Inputs: From "Get detailed monthly report".  
   - Operation: Get file  
   - Owner: `{{$json.github_repo.owner}}` (from Globals)  
   - Repository: `{{$json.github_repo.name}}` (from Globals)  
   - File path: `reports/{{$json.reportName}}`  
   - Credentials: GitHub OAuth2 with read permissions.  
   - Set "On Error" to "Continue on Error" to catch 404.

9. **Add a Set node named "Point to new data":**  
   - Inputs: From "Check if file exists in GitHub" (success path).  
   - Assign variable:  
     - `data` = `{{$json["timeentries"]}}` (new report data)

10. **Add an Extract From File node named "Extract from File":**  
    - Inputs: From "Check if file exists in GitHub" (success path).  
    - Operation: fromJson (parses existing GitHub file content)

11. **Add a Compare Datasets node named "Compare Datasets":**  
    - Inputs:  
      - First: From "Point to new data" (new data)  
      - Second: From "Extract from File" (existing data)  
    - Merge by field: `data`  
    - Outputs:  
      - True: Connect to "Update file in GitHub"  
      - False: No output (no changes)

12. **Add a GitHub node named "Update file in GitHub":**  
    - Inputs: From "Compare Datasets" (true path).  
    - Operation: Edit file  
    - Owner, Repository, File path: same as "Check if file exists in GitHub"  
    - File content: `{{ JSON.stringify($json.data, null, 2) }}`  
    - Commit message: "Update report"  
    - Credentials: GitHub OAuth2 with write permissions.

13. **Add an If node named "Check for 404 error message":**  
    - Inputs: From "Check if file exists in GitHub" (error path).  
    - Condition: Check if `$json.error` contains "could not be found".  
    - True path: To "Skip empty reports"  
    - False path: To "Stop and Error"

14. **Add a Filter node named "Skip empty reports":**  
    - Inputs: From "Check for 404 error message" (true path).  
    - Condition: `$json.timeentries` is not empty.  
    - True path: To "Create file in GitHub"  
    - False path: No output (skip empty reports)

15. **Add a GitHub node named "Create file in GitHub":**  
    - Inputs: From "Skip empty reports".  
    - Operation: Create file  
    - Owner, Repository, File path: same as above  
    - File content: `{{ JSON.stringify($json.timeentries, null, 2) }}`  
    - Commit message: "Create report"  
    - Credentials: GitHub OAuth2 with write permissions.

16. **Add a Stop and Error node named "Stop and Error":**  
    - Inputs: From "Check for 404 error message" (false path).  
    - Error message: `{{$json.error}}` (dynamic error message)

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| It is essential to back up previous months as well, as values like tags may still change over time.           | Sticky Note on GitHub file creation/update nodes                                                      |
| Create a **private** GitHub repository and ensure credentials have read/write permissions for the workflow.   | Setup instructions in workflow description                                                           |
| By default, the workflow backs up the current month and previous two months (3 months total).                  | Sticky Note on month indexes and scope                                                                |
| The workflow runs daily at 5:00 AM by default but can be adjusted in the Schedule Trigger node.                | Sticky Note on schedule trigger                                                                        |
| Clockify API credentials require access to workspace data and reports API.                                    | Credential setup notes                                                                                 |
| GitHub credentials require repository access with permissions to create and edit files.                       | Credential setup notes                                                                                 |
| The workflow uses the Clockify Reports API endpoint for detailed time entries, paginated with page size 50.    | Node configuration details                                                                             |
| The workflow uses JSON stringification with indentation (2 spaces) when writing files to GitHub for readability.| GitHub node file content configuration                                                                |

---

This documentation provides a complete, structured understanding of the "Backup Clockify to Github based on monthly reports" workflow, enabling users and automation agents to comprehend, reproduce, and maintain it effectively.