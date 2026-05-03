Sync Azure DevOps Work Items to GitHub Issues with Google Sheets Tracking

https://n8nworkflows.xyz/workflows/sync-azure-devops-work-items-to-github-issues-with-google-sheets-tracking-8643


# Sync Azure DevOps Work Items to GitHub Issues with Google Sheets Tracking

### 1. Workflow Overview

This workflow automates synchronization between Microsoft Azure DevOps work items and GitHub Issues, tracking the mappings in Google Sheets. It is designed primarily for teams using Azure DevOps for project management and GitHub for issue tracking, providing seamless integration and traceability.

The workflow is logically divided into two main functional blocks:

- **1.1 Story Work Item Processing:**  
  Triggered when a new Story is created or updated in Azure DevOps. It extracts key data from the Story, creates a corresponding GitHub Issue, assigns it to a random collaborator from the GitHub repository, and logs the Azure Story ID, GitHub Issue number, and URL into a Google Sheet for cross-reference.

- **1.2 Task Work Item Processing:**  
  Triggered when a new Task is created or updated in Azure DevOps. It extracts relevant Task data, looks up the parent Story's GitHub Issue via the Google Sheets mapping, retrieves the GitHub Issue details, and updates the Issue body by appending a clickable bullet point link referencing the Task.

This structure ensures that Stories and their associated Tasks in Azure DevOps are mirrored and linked within GitHub Issues, maintaining workload balance via random assignee allocation and centralized tracking through Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Story Work Item Processing

**Overview:**  
This block handles new or updated Story work items from Azure DevOps. It extracts relevant Story data, creates a GitHub Issue, assigns a random collaborator, and logs the linkage in Google Sheets.

**Nodes Involved:**  
- Webhook1  
- Filter Story Data  
- Create Github Issue  
- HTTP Request1  
- Code2  
- HTTP Request2  
- Google Sheets  

**Node Details:**  

- **Webhook1**  
  - *Type:* Webhook  
  - *Role:* Entry point that receives Azure DevOps POST events for Story work items.  
  - *Configuration:* Listens on path `3006842c-5f13-4147-8d78-9ee73ac78e34`, only POST method; responds immediately with message "Azure Devops Event Triggered".  
  - *Connections:* Output to Filter Story Data.  
  - *Edge cases:* Missing or malformed payload could cause downstream failures. Ensure Azure DevOps sends proper webhook data.  

- **Filter Story Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts and simplifies Azure DevOps Story work item data from webhook payload.  
  - *Configuration:* Runs once per item; extracts fields such as ID, URL, Title, WorkItemType, CreatedBy, CreatedDate, Description (default empty string), State, AreaPath, IterationPath.  
  - *Key expressions:* Accesses `$json.body.resource` fields for extraction.  
  - *Connections:* Output to Create Github Issue.  
  - *Edge cases:* Missing expected fields, e.g., Description or CreatedBy, may require default handling.  

- **Create Github Issue**  
  - *Type:* GitHub Node  
  - *Role:* Creates a new GitHub Issue using title and description from filtered Story data.  
  - *Configuration:* Owner set to `usman151710`; repository set to `github-n8n`; authentication via OAuth2 credential named "GitHub - Mohammad Usman". Labels and assignees empty at creation.  
  - *Key expressions:* Title and body derived from filtered Story data.  
  - *Connections:* Output to HTTP Request1.  
  - *Edge cases:* API rate limits, permission errors, or invalid repo/owner names may cause failure.  

- **HTTP Request1**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the list of collaborators from the GitHub repository to determine assignees.  
  - *Configuration:* URL constructed dynamically as `{{$json.repository_url}}/collaborators`. Authentication uses the same GitHub OAuth2 credential. Expects JSON response.  
  - *Connections:* Output to Code2.  
  - *Edge cases:* Network issues, API errors, or insufficient permissions can cause failures.  

- **Code2**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes collaborator list to randomly select one assignee for the new Issue.  
  - *Configuration:* Extracts `login` field from each collaborator; selects one at random; returns object with `assignee` property.  
  - *Connections:* Output to HTTP Request2.  
  - *Edge cases:* Empty collaborator list results in undefined assignee; must handle gracefully.  

- **HTTP Request2**  
  - *Type:* HTTP Request  
  - *Role:* Updates the newly created GitHub Issue, assigning the randomly selected collaborator.  
  - *Configuration:* PATCH method to the GitHub Issue URL from the Create Github Issue node; JSON body specifies `"assignees": [selected assignee]`. Uses GitHub OAuth2 authentication.  
  - *Connections:* Output to Google Sheets.  
  - *Edge cases:* API permission errors, invalid assignee, or network issues may cause failure.  

- **Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends a new row mapping Azure DevOps Story ID, GitHub Issue number, and GitHub Issue URL for tracking.  
  - *Configuration:* Appends to sheet with document ID `1Cr7IRGiS1mB691euOBvDLC_Y56lTDWc4mhAll9-TEII`, sheet `gid=0` (named "IssuesMap"). Columns appended: Url, AzureID, IssueID. Uses OAuth2 credential "Google Sheets Read - Usman".  
  - *Connections:* Terminal node for this branch.  
  - *Edge cases:* Google Sheets API quota limits, permission errors, or document access issues may cause failure.  

---

#### 2.2 Task Work Item Processing

**Overview:**  
This block handles new or updated Task work items from Azure DevOps. It extracts Task data, looks up the parent Story’s GitHub Issue mapping in Google Sheets, retrieves that Issue, and appends a clickable Task link to the Issue body.

**Nodes Involved:**  
- Webhook2  
- Filter Task Data  
- Google Sheets1  
- GitHub2  
- GitHub1  

**Node Details:**  

- **Webhook2**  
  - *Type:* Webhook  
  - *Role:* Entry point that receives Azure DevOps POST events for Task work items.  
  - *Configuration:* Listens on path `cfecf4f6-b82d-4dd0-af5b-bd231130b272`, POST method.  
  - *Connections:* Output to Filter Task Data.  
  - *Edge cases:* Similar to Webhook1; malformed or missing payloads can cause downstream errors.  

- **Filter Task Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts and simplifies Azure DevOps Task work item data.  
  - *Configuration:* Runs once per item; extracts fields: ID, URL, Title, WorkItemType, CreatedBy, CreatedDate, Description (default empty), State, Parent (AzureID of parent Story), ticketUrl (link to Task in Azure DevOps).  
  - *Connections:* Output to Google Sheets1.  
  - *Edge cases:* Missing Parent field or malformed URL can cause issues.  

- **Google Sheets1**  
  - *Type:* Google Sheets Node  
  - *Role:* Searches the Google Sheet mapping to find the GitHub Issue corresponding to the Task’s parent Story AzureID.  
  - *Configuration:* Filters sheet `gid=0` in document `1Cr7IRGiS1mB691euOBvDLC_Y56lTDWc4mhAll9-TEII` for rows where `AzureID` equals Task’s `parent` field. Uses OAuth2 credential "Google Sheets Read - Usman".  
  - *Connections:* Output to GitHub2.  
  - *Edge cases:* No matching row found results in empty output; must handle gracefully.  

- **GitHub2**  
  - *Type:* GitHub Node  
  - *Role:* Retrieves the GitHub Issue details for the parent Story using IssueID found in Google Sheets.  
  - *Configuration:* Owner `usman151710`; repo `github-n8n`; issueNumber from Google Sheets lookup. OAuth2 authentication.  
  - *Connections:* Output to GitHub1.  
  - *Edge cases:* Invalid issueNumber, API permission issues, or network failures may occur.  

- **GitHub1**  
  - *Type:* GitHub Node  
  - *Role:* Edits the GitHub Issue body to append a bullet link referencing the new Task.  
  - *Configuration:* Edits Issue with IssueID from Google Sheets; appends HTML list item with clickable link to Azure DevOps Task URL, including AzureID and Title. Owner and repo same as above; OAuth2 authentication.  
  - *Connections:* Terminal node for this branch.  
  - *Edge cases:* API update conflicts, invalid HTML encoding, or permission issues.  

---

### 3. Summary Table

| Node Name        | Node Type           | Functional Role                                     | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                     |
|------------------|---------------------|----------------------------------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------------------------|
| Webhook1         | Webhook             | Trigger for Azure DevOps Story events              | —                     | Filter Story Data      | 🔔 Trigger for Azure DevOps Story events. Receives POST requests whenever a new Story work item is created/updated in Azure DevOps. |
| Filter Story Data | Code                | Clean and structure Story data from Azure DevOps   | Webhook1              | Create Github Issue    | 🧹 Cleans and structures Story data from Azure DevOps. Extracts fields like: ID, Title, Description, CreatedBy, State, etc. Passes simplified object for GitHub Issue creation. |
| Create Github Issue | GitHub Node        | Create new GitHub Issue from Story data             | Filter Story Data      | HTTP Request1          | 📄 Creates a new GitHub Issue in the repository (based on Story Title and Description from Azure DevOps).       |
| HTTP Request1    | HTTP Request        | Fetch repository collaborators from GitHub         | Create Github Issue    | Code2                  | 📡 Calls GitHub API to fetch repository collaborators. Used to determine possible assignees for the new issue.  |
| Code2            | Code                | Select a random collaborator as assignee            | HTTP Request1          | HTTP Request2          | 🎲 Selects a random collaborator from the list of repository members and prepares the assignee value.           |
| HTTP Request2    | HTTP Request        | Assign selected collaborator to GitHub Issue        | Code2                  | Google Sheets          | ✏️ Updates the GitHub Issue using PATCH request. Assigns the randomly selected collaborator.                    |
| Google Sheets    | Google Sheets       | Append mapping of Azure Story ID, GitHub Issue info | HTTP Request2          | —                      | 📊 Appends mapping between Azure DevOps Story ID, GitHub Issue number, and Issue URL into the Google Sheet. Acts as a central cross-reference. |
| Webhook2         | Webhook             | Trigger for Azure DevOps Task events                 | —                     | Filter Task Data       | 🔔 Trigger for Azure DevOps Task events. Receives POST requests whenever a new Task is created/updated in Azure DevOps. |
| Filter Task Data | Code                | Clean and structure Task data from Azure DevOps     | Webhook2               | Google Sheets1         | 🧹 Cleans and structures Task data from Azure DevOps. Extracts Task ID, Title, Description, Parent Story ID, and Task URL for mapping back to GitHub Issue. |
| Google Sheets1   | Google Sheets       | Lookup parent Story mapping in Google Sheets        | Filter Task Data       | GitHub2                | 🔍 Looks up the parent Story’s mapping in Google Sheets (using AzureID as the key) to find the corresponding GitHub Issue ID. |
| GitHub2          | GitHub Node         | Retrieve GitHub Issue details for parent Story      | Google Sheets1         | GitHub1                | 📥 Retrieves details of the GitHub Issue associated with the parent Story from the mapping found in Sheets.      |
| GitHub1          | GitHub Node         | Append Task link to GitHub Issue body                | GitHub2                | —                      | ➕ Updates the GitHub Issue body. Appends a clickable Task link (with Azure DevOps Task ID and Title) as a bullet point under the parent GitHub Issue. |
| Sticky Note      | Sticky Note         | General workflow purpose description                 | —                      | —                      | 📌 Workflow Purpose: This workflow automates the connection between Microsoft Azure DevOps, GitHub, and Google Sheets to streamline project tracking. Details included above. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook1 Node**  
   - Type: Webhook  
   - Parameters:  
     - HTTP Method: POST  
     - Path: `3006842c-5f13-4147-8d78-9ee73ac78e34`  
     - Response Data: `"Azure Devops Event Triggered"`  

2. **Create Filter Story Data Node**  
   - Type: Code  
   - Parameters:  
     - Run Mode: Run once for each item  
     - JavaScript Code: Extract and build simplified Story object with fields: id, url, title, workItemType, createdBy, createdDate, description (default ""), state, areaPath, iterationPath.  
   - Connect output of Webhook1 to input of this node.  

3. **Create Create Github Issue Node**  
   - Type: GitHub Node (Create Issue)  
   - Parameters:  
     - Owner: `usman151710`  
     - Repository: `github-n8n`  
     - Title: Expression from filtered Story data: `{{$json.title}}`  
     - Body: Expression: `{{$json.description}}`  
     - Labels: empty  
     - Assignees: empty (initially)  
   - Credentials: GitHub OAuth2 with appropriate access scope for issues creation.  
   - Connect output of Filter Story Data to this node.  

4. **Create HTTP Request1 Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Expression: `{{$json.repository_url}}/collaborators` (retrieved from previous GitHub Issue creation response)  
     - Authentication: OAuth2 (GitHub) credentials same as above  
     - Response Format: JSON  
   - Connect output of Create Github Issue to this node.  

5. **Create Code2 Node**  
   - Type: Code  
   - Parameters:  
     - JavaScript code to extract `login` from each collaborator and randomly select one assignee.  
   - Connect output of HTTP Request1 to this node.  

6. **Create HTTP Request2 Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: Expression: `{{$('Create Github Issue').first().json.url}}` (URL of created issue)  
     - Method: PATCH  
     - Body (JSON): `{"assignees": ["{{$json.assignee}}"]}`  
     - Authentication: GitHub OAuth2 credentials  
   - Connect output of Code2 to this node.  

7. **Create Google Sheets Node**  
   - Type: Google Sheets (Append)  
   - Parameters:  
     - Document ID: `1Cr7IRGiS1mB691euOBvDLC_Y56lTDWc4mhAll9-TEII`  
     - Sheet Name: `gid=0` (IssuesMap)  
     - Columns: Url (from GitHub Issue URL), AzureID (from Filter Story Data id), IssueID (from GitHub Issue number)  
   - Credentials: Google Sheets OAuth2 with access rights to the document  
   - Connect output of HTTP Request2 to this node.  

8. **Create Webhook2 Node**  
   - Type: Webhook  
   - Parameters:  
     - HTTP Method: POST  
     - Path: `cfecf4f6-b82d-4dd0-af5b-bd231130b272`  

9. **Create Filter Task Data Node**  
   - Type: Code  
   - Parameters:  
     - Run Mode: Run once for each item  
     - JavaScript code to extract Task fields: id, url, title, workItemType, createdBy, createdDate, description (default ""), state, parent (AzureID of parent Story), ticketUrl (link to Azure DevOps Task URL)  
   - Connect output of Webhook2 to this node.  

10. **Create Google Sheets1 Node**  
    - Type: Google Sheets (Lookup / Filter)  
    - Parameters:  
      - Document ID: same as above  
      - Sheet Name: `gid=0`  
      - Filter: AzureID column equals `{{$json.parent}}` from Task data  
    - Credentials: Google Sheets OAuth2  
    - Connect output of Filter Task Data to this node.  

11. **Create GitHub2 Node**  
    - Type: GitHub Node (Get Issue)  
    - Parameters:  
      - Owner: `usman151710`  
      - Repository: `github-n8n`  
      - Issue Number: `{{$json.IssueID}}` from Google Sheets lookup  
      - Authentication: GitHub OAuth2  
    - Connect output of Google Sheets1 to this node.  

12. **Create GitHub1 Node**  
    - Type: GitHub Node (Edit Issue)  
    - Parameters:  
      - Owner: `usman151710`  
      - Repository: `github-n8n`  
      - Issue Number: `{{$('Google Sheets1').first().json.IssueID}}`  
      - Body: Append to current body an HTML list item with clickable link to Task URL, showing AzureID and Task title, e.g.:  
        ```html
        {{$json.body}}

        <li>
          <a href="{{ $('Filter Task Data').first().json.ticketUrl }}"
             target="_blank" rel="noopener noreferrer">
            [{{ $('Google Sheets1').first().json.AzureID }}] - {{ $('Filter Task Data').first().json.title }}
          </a>
        </li>
        ```  
      - Authentication: GitHub OAuth2  
    - Connect output of GitHub2 to this node.  

Make sure all OAuth2 credentials for GitHub and Google Sheets are configured with the necessary scopes:

- GitHub OAuth2: repo access, issues read/write, collaborator read  
- Google Sheets OAuth2: read/write access to the target spreadsheet  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 📌 Workflow Purpose: Automates syncing Azure DevOps Stories and Tasks to GitHub Issues, assigns collaborators randomly, and tracks mappings in Google Sheets for project traceability.                                                                                                                                                                                                                                              | General workflow overview                                                                           |
| 🔔 Webhook triggers require Azure DevOps to be configured to send POST webhooks to the respective webhook URLs.                                                                                                                                                                                                                                                                                                                    | Azure DevOps webhook setup                                                                          |
| OAuth2 credentials for GitHub must have repository and issue permissions; Google Sheets credentials need edit access to the specified spreadsheet.                                                                                                                                                                                                                                                                               | Credential requirements                                                                             |
| Google Sheets document used as a centralized mapping table between Azure DevOps work item IDs and GitHub Issues: https://docs.google.com/spreadsheets/d/1Cr7IRGiS1mB691euOBvDLC_Y56lTDWc4mhAll9-TEII/edit#gid=0                                                                                                                                                                                                                         | Google Sheets mapping document                                                                      |
| GitHub repository: https://github.com/usman151710/github-n8n                                                                                                                                                                                                                                                                                                                                                                        | Repository used for Issue creation and updates                                                     |
| Random assignee selection balances workload among collaborators but does not account for current workload or availability; consider enhancements if needed.                                                                                                                                                                                                                                                                       | Potential enhancement note                                                                         |
| Error handling is not explicitly defined; consider adding nodes for catching API errors, handling empty collaborator lists, or missing mapping rows to improve reliability.                                                                                                                                                                                                                                                        | Reliability considerations                                                                         |
| The workflow uses HTML in GitHub Issue bodies to embed clickable links for related Tasks to improve visibility and navigation.                                                                                                                                                                                                                                                                                                     | Formatting detail                                                                                  |
| This workflow was created and tagged under the project "Usman - Azure" (created 2025-09-01).                                                                                                                                                                                                                                                                                                                                         | Project metadata                                                                                   |

---

**Disclaimer:**  
The provided description and analysis are based exclusively on a workflow automated with n8n, adhering strictly to current content policies and containing no illegal or protected content. All data handled are legal and public.