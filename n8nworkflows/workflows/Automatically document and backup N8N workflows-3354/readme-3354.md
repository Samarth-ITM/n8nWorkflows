Automatically document and backup N8N workflows

https://n8nworkflows.xyz/workflows/automatically-document-and-backup-n8n-workflows-3354


# Automatically document and backup N8N workflows

### 1. Workflow Overview

This workflow automates the documentation and backup of selected n8n workflows by performing weekly scans for active workflows tagged with "internal-infra." It extracts key metadata and definitions, generates AI-based summaries, stores backups in a GitHub repository, updates a Notion database with documentation, and sends Slack notifications about new or updated workflows.

Logical blocks:

- **1.1 Scheduled Trigger & Workflow Discovery:** Weekly trigger to find active workflows tagged "internal-infra" and filter those updated within the last 7 days.
- **1.2 Metadata Extraction & Preparation:** Extracts and formats workflow metadata and definitions for further processing.
- **1.3 Notion Page Query & Mapping:** Checks if the workflow already exists in the Notion database and prepares data for update or creation.
- **1.4 AI Summarization:** Uses OpenAI GPT-4o-mini to generate concise summaries describing each workflow’s purpose and operation.
- **1.5 Notion Database Update:** Adds new workflows or updates existing ones in the Notion database with metadata and AI summaries.
- **1.6 GitHub Backup:** Uploads or updates workflow JSON files in a specified GitHub repository as backups.
- **1.7 Slack Notifications:** Sends Slack messages notifying about new workflows added, updates made, or errors in workflow setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Workflow Discovery

- **Overview:**  
  This block triggers the workflow every Monday at 1 AM, then queries the n8n instance for active workflows tagged "internal-infra" and filters those updated in the last 7 days.

- **Nodes Involved:**  
  - Every Monday at 1am (Schedule Trigger)  
  - Get active workflows with internal-infra tag (n8n API node)  
  - Check if updated in last 7 days (If node)

- **Node Details:**

  - **Every Monday at 1am**  
    - Type: Schedule Trigger  
    - Configuration: Runs weekly on Mondays at 1 AM  
    - Inputs: None (trigger only)  
    - Outputs: Triggers next node  
    - Edge cases: Ensure server timezone matches expected schedule; missed runs if n8n is down.

  - **Get active workflows with internal-infra tag**  
    - Type: n8n API node  
    - Configuration: Filters workflows by tag "internal-infra" and active status true  
    - Inputs: Trigger from schedule  
    - Outputs: List of workflows with metadata  
    - Edge cases: API authentication failures; empty results if no workflows match.

  - **Check if updated in last 7 days**  
    - Type: If node  
    - Configuration: Compares each workflow’s updatedAt date to current date minus 7 days  
    - Inputs: Workflows list  
    - Outputs: Passes workflows updated within last 7 days  
    - Edge cases: Date parsing errors; workflows with missing updatedAt fields.

---

#### 2.2 Metadata Extraction & Preparation

- **Overview:**  
  Extracts key fields from each workflow JSON, prepares URLs, filenames, and flags needed for subsequent processing.

- **Nodes Involved:**  
  - Set fields

- **Node Details:**

  - **Set fields**  
    - Type: Set node  
    - Configuration: Assigns fields such as active status, workflow URL (based on n8n host), error workflow flag, workflow name, timestamps, environment ID, full workflow JSON string, and export filename (sanitized)  
    - Expressions:  
      - URL: `"https://<n8n_host_name>/workflow/{{ $json.id }}"` (replace `<n8n_host_name>`)  
      - workflow-export-filename: replaces spaces and dashes with underscores and appends `.json`  
    - Inputs: Workflows filtered by last update  
    - Outputs: Enriched workflow metadata  
    - Edge cases: Missing or invalid fields; ensure n8n host URL is correctly set.

---

#### 2.3 Notion Page Query & Mapping

- **Overview:**  
  Queries Notion database to find if a page for the workflow already exists based on a unique environment ID, then maps data for creation or update.

- **Nodes Involved:**  
  - Get notion page with workflow id (HTTP Request)  
  - Map fields (Set node)

- **Node Details:**

  - **Get notion page with workflow id**  
    - Type: HTTP Request  
    - Configuration: POST request to Notion API `/v1/databases/<your_db_id_here>/query` with a filter on `envId` property containing the workflow’s envId  
    - Headers: Notion-Version `2022-06-28`  
    - Authentication: Notion API credential  
    - Inputs: Workflow metadata from Set fields  
    - Outputs: Notion query results (existing pages)  
    - Edge cases: API rate limits; invalid database ID; network errors.

  - **Map fields**  
    - Type: Set node  
    - Configuration: Wraps input JSON under `input` key for easier reference downstream, includes all other fields unchanged  
    - Inputs: Notion query results  
    - Outputs: Prepared data for Notion update or creation  
    - Edge cases: None significant.

---

#### 2.4 AI Summarization

- **Overview:**  
  Sends the workflow JSON definition to OpenAI GPT-4o-mini to generate a concise summary describing the workflow’s function and operation.

- **Nodes Involved:**  
  - Summarize what the Workflow does (OpenAI node)

- **Node Details:**

  - **Summarize what the Workflow does**  
    - Type: OpenAI (LangChain) node  
    - Configuration: Uses GPT-4o-mini model; prompt instructs to provide 1-2 line concise description and a short paragraph if needed, passing the entire workflow JSON as `<n8nplugin>` block  
    - Inputs: Workflow metadata and definition from Map fields  
    - Outputs: AI-generated summary text in `$json.message.content`  
    - Credentials: OpenAI API key  
    - Edge cases: API rate limits, timeouts, or invalid JSON input; ensure workflow JSON is not too large for prompt size limits.

---

#### 2.5 Notion Database Update

- **Overview:**  
  Depending on whether the workflow is new or existing, adds a new page or updates the existing page in Notion with metadata and AI summary.

- **Nodes Involved:**  
  - Is this a new workflow (to Notion)? (If node)  
  - Add to Notion (databasePage create)  
  - Update in Notion (databasePage update)  
  - Notify internal-infra of push (Slack)  
  - Notify internal-infra of update (Slack)

- **Node Details:**

  - **Is this a new workflow (to Notion)?**  
    - Type: If node  
    - Configuration: Checks if Notion query results array is empty (new workflow) or not (existing workflow)  
    - Inputs: AI summary output  
    - Outputs: Branches to Add or Update nodes  
    - Edge cases: Empty or malformed Notion results.

  - **Add to Notion**  
    - Type: Notion node (databasePage create)  
    - Configuration: Creates a new page with properties mapped from workflow metadata and AI summary, including workflow name, URL, active status, created/updated timestamps, error flag, last update timestamp (now), AI summary text, and envId  
    - Inputs: New workflow data  
    - Credentials: Notion API credential  
    - Outputs: Confirmation of creation  
    - Edge cases: API errors, invalid database ID.

  - **Update in Notion**  
    - Type: Notion node (databasePage update)  
    - Configuration: Updates existing page by page ID with updated active status, workflow updated timestamp, error flag, and AI summary  
    - Inputs: Existing workflow data and AI summary  
    - Credentials: Notion API credential  
    - Outputs: Confirmation of update  
    - Edge cases: Page ID missing or invalid.

  - **Notify internal-infra of push**  
    - Type: Slack node  
    - Configuration: Sends message to configured Slack channel announcing new workflow added to Notion, includes workflow name  
    - Inputs: After Add to Notion  
    - Credentials: Slack Bot token  
    - Edge cases: Slack API errors, invalid channel.

  - **Notify internal-infra of update**  
    - Type: Slack node  
    - Configuration: Sends message to Slack channel announcing workflow update in Notion, includes workflow name  
    - Inputs: After Update in Notion  
    - Credentials: Slack Bot token  
    - Edge cases: Slack API errors.

---

#### 2.6 GitHub Backup

- **Overview:**  
  Uploads or updates the workflow JSON file in a GitHub repository under `N8N_Workflows` folder, using sanitized filenames. Handles errors and notifies Slack on failures.

- **Nodes Involved:**  
  - Check that error workflow has been configured (If node)  
  - Upload changes to repo (GitHub edit file)  
  - Create new file in repo (GitHub create file)  
  - Notify on create file in repo fail (Slack)  
  - Notify on workflow setup error (Slack)

- **Node Details:**

  - **Check that error workflow has been configured**  
    - Type: If node  
    - Configuration: Checks if `errorWorkflow` flag is true or if workflow name equals `_infra: Get a Slack alert when a workflow went wrong` (exempt)  
    - Inputs: After Map fields  
    - Outputs: If true, proceed to upload; if false, send error notification  
    - Edge cases: Missing flag or incorrect logic may block backups.

  - **Upload changes to repo**  
    - Type: GitHub node (edit file)  
    - Configuration: Edits existing file in repo at path `N8N_Workflows/<sanitized_filename>` with workflow JSON content; commit message includes execution ID  
    - Inputs: From Check that error workflow has been configured  
    - Credentials: GitHub OAuth2 token with repo write access  
    - On error: Continues to Create new file in repo node  
    - Edge cases: File not found triggers fallback; API rate limits.

  - **Create new file in repo**  
    - Type: GitHub node (create file)  
    - Configuration: Creates new file in repo at same path with workflow JSON content; commit message "created by N8N"  
    - Inputs: On failure of upload changes  
    - Credentials: GitHub OAuth2 token  
    - On error: Continues to Notify on create file in repo fail  
    - Edge cases: Repository permissions; filename sanitization.

  - **Notify on create file in repo fail**  
    - Type: Slack node  
    - Configuration: Sends warning message to Slack channel if GitHub file creation fails, including workflow URL and name with link  
    - Inputs: On failure of create new file node  
    - Credentials: Slack Bot token  
    - Edge cases: Slack API errors.

  - **Notify on workflow setup error**  
    - Type: Slack node  
    - Configuration: Sends warning if error workflow is not configured for a workflow, includes clickable link to workflow in n8n  
    - Inputs: On false branch of Check that error workflow has been configured  
    - Credentials: Slack Bot token  
    - Edge cases: Slack API errors.

---

### 3. Summary Table

| Node Name                           | Node Type                  | Functional Role                              | Input Node(s)                      | Output Node(s)                                   | Sticky Note                                                                                      |
|-----------------------------------|----------------------------|----------------------------------------------|----------------------------------|-------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Every Monday at 1am               | Schedule Trigger           | Triggers workflow weekly on Monday 1 AM      | None                             | Get active workflows with internal-infra tag    | ## Check weekly for changes on active internal-infra workflows                                  |
| Get active workflows with internal-infra tag | n8n API node              | Retrieves active workflows tagged "internal-infra" | Every Monday at 1am              | Check if updated in last 7 days                   |                                                                                                 |
| Check if updated in last 7 days   | If node                   | Filters workflows updated within last 7 days | Get active workflows with internal-infra tag | Set fields                                       | ## Extract key fields                                                                           |
| Set fields                       | Set node                  | Extracts and prepares workflow metadata       | Check if updated in last 7 days  | Get notion page with workflow id                 | ## Extract key fields                                                                           |
| Get notion page with workflow id | HTTP Request              | Queries Notion database for existing workflow page | Set fields                      | Map fields                                       |                                                                                                 |
| Map fields                      | Set node                  | Wraps data for Notion update/create            | Get notion page with workflow id | Check that error workflow has been configured; Summarize what the Workflow does |                                                                                                 |
| Check that error workflow has been configured | If node                   | Checks if error workflow flag is set           | Map fields                      | Upload changes to repo; Notify on workflow setup error | ## Check workflow setup OK                                                                     |
| Summarize what the Workflow does | OpenAI (LangChain) node   | Generates AI summary of the workflow           | Check that error workflow has been configured | Is this a new workflow (to Notion) ?             | ## Generate documentation in Notion                                                           |
| Is this a new workflow (to Notion) ? | If node                   | Determines if workflow is new or existing in Notion | Summarize what the Workflow does | Add to Notion; Update in Notion                   |                                                                                                 |
| Add to Notion                   | Notion node (create page) | Creates new Notion page for the workflow       | Is this a new workflow (to Notion) ? | Notify internal-infra of push                     | ## Generate documentation in Notion                                                           |
| Update in Notion                | Notion node (update page) | Updates existing Notion page with new data     | Is this a new workflow (to Notion) ? | Notify internal-infra of update                   | ## Generate documentation in Notion                                                           |
| Notify internal-infra of push   | Slack node                | Sends Slack notification for new workflow      | Add to Notion                   | None                                            |                                                                                                 |
| Notify internal-infra of update | Slack node                | Sends Slack notification for updated workflow  | Update in Notion                | None                                            |                                                                                                 |
| Upload changes to repo          | GitHub node (edit file)   | Updates existing workflow file in GitHub repo  | Check that error workflow has been configured | Create new file in repo                           | ## Upload / backup workflow to GitHub repo                                                    |
| Create new file in repo         | GitHub node (create file) | Creates new workflow file in GitHub repo       | Upload changes to repo           | Notify on create file in repo fail                | ## Upload / backup workflow to GitHub repo                                                    |
| Notify on create file in repo fail | Slack node                | Sends Slack warning if GitHub file creation fails | Create new file in repo          | None                                            |                                                                                                 |
| Notify on workflow setup error | Slack node                | Sends Slack warning if error workflow not configured | Check that error workflow has been configured (false branch) | None                                            |                                                                                                 |
| Sticky Note                     | Sticky Note               | Visual note: Extract key fields                 | None                           | None                                            | ## Extract key fields                                                                           |
| Sticky Note1                    | Sticky Note               | Visual note: Check workflow setup OK            | None                           | None                                            | ## Check workflow setup OK                                                                     |
| Sticky Note2                    | Sticky Note               | Visual note: Upload / backup workflow to GitHub repo | None                           | None                                            | ## Upload / backup workflow to GitHub repo                                                    |
| Sticky Note3                    | Sticky Note               | Visual note: Generate documentation in Notion   | None                           | None                                            | ## Generate documentation in Notion                                                           |
| Sticky Note4                    | Sticky Note               | Visual note: Check weekly for changes on active internal-infra workflows | None                           | None                                            | ## Check weekly for changes on active internal-infra workflows                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Name: "Every Monday at 1am"  
   - Type: Schedule Trigger  
   - Configuration: Set to trigger weekly on Mondays at 1 AM

2. **Create n8n API Node to Get Workflows:**  
   - Name: "Get active workflows with internal-infra tag"  
   - Type: n8n API node  
   - Configuration: Filter workflows by tag "internal-infra" and active status true

3. **Create If Node to Filter Recent Updates:**  
   - Name: "Check if updated in last 7 days"  
   - Type: If node  
   - Condition: `$json.updatedAt >= (current date - 7 days)` (ISO string comparison)

4. **Create Set Node to Extract Fields:**  
   - Name: "Set fields"  
   - Type: Set node  
   - Assign fields:  
     - active: `{{$json.active}}` (boolean)  
     - url: `"https://<n8n_host_name>/workflow/{{$json.id}}"` (replace `<n8n_host_name>`)  
     - errorWorkflow: `{{$json.settings?.errorWorkflow ? true : false}}` (boolean)  
     - workflow-name: `{{$json.name}}` (string)  
     - updatedAt: `{{$json.updatedAt}}` (string)  
     - createdAt: `{{$json.createdAt}}` (string)  
     - envId: `"internal-{{$json.id}}"` (string)  
     - workflow-definition: `{{ JSON.stringify($json, null, 2) }}` (string)  
     - workflow-export-filename: `{{$json.name.replace(/ /g, "_").replace(/-/g, "_") + ".json"}}` (string)

5. **Create HTTP Request Node to Query Notion Database:**  
   - Name: "Get notion page with workflow id"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.notion.com/v1/databases/<your_db_id_here>/query`  
   - Headers:  
     - Notion-Version: `2022-06-28`  
   - Body (JSON):  
     ```json
     {
       "filter": {
         "and": [
           {
             "property": "envId",
             "rich_text": { "contains": "{{ $json.envId }}" }
           }
         ]
       }
     }
     ```  
   - Authentication: Use Notion API credential

6. **Create Set Node to Map Fields:**  
   - Name: "Map fields"  
   - Type: Set node  
   - Assign: `input` = `{{$('Set fields').item.json}}`  
   - Include other fields unchanged

7. **Create If Node to Check Error Workflow Setup:**  
   - Name: "Check that error workflow has been configured"  
   - Type: If node  
   - Condition:  
     - `input.errorWorkflow == true` OR  
     - `input['workflow-name'] == '_infra: Get a Slack alert when a workflow went wrong'`  
   - True branch: proceed to GitHub upload  
   - False branch: send Slack warning

8. **Create OpenAI Node for Summarization:**  
   - Name: "Summarize what the Workflow does"  
   - Type: OpenAI (LangChain) node  
   - Model: GPT-4o-mini  
   - Prompt:  
     ```
     Concisely tell me what this N8N plugin does in 1-2 lines, then describe how it does it in no more than a paragraph, but only if that detail was already covered by the first 1-2 lines - we don't want to repeat ourselves.
     <n8nplugin>
     {{ $json.input['workflow-definition'] }}
     </n8nplugin>
     ```

9. **Create If Node to Check if Workflow is New to Notion:**  
   - Name: "Is this a new workflow (to Notion) ?"  
   - Type: If node  
   - Condition: Check if `$('Map fields').item.json.results` array is empty

10. **Create Notion Node to Add New Page:**  
    - Name: "Add to Notion"  
    - Type: Notion node (databasePage create)  
    - Database ID: `<replace_me>`  
    - Properties: Map fields from input including workflow name, URL, active, created/updated dates, error flag, last update (now), AI summary, envId

11. **Create Notion Node to Update Existing Page:**  
    - Name: "Update in Notion"  
    - Type: Notion node (databasePage update)  
    - Page ID: `{{$('Map fields').item.json.results[0].id}}`  
    - Properties: Update active, updatedAt, errorWorkflow, AI summary

12. **Create Slack Nodes for Notifications:**  
    - Name: "Notify internal-infra of push"  
      - Type: Slack node  
      - Message: `Pushed new workflow to Notion: {{$('Map fields').item.json.input['workflow-name']}}`  
      - Channel: `<yourchannelID>`  
    - Name: "Notify internal-infra of update"  
      - Type: Slack node  
      - Message: `Updated workflow in Notion: {{$('Map fields').item.json.input['workflow-name']}}`  
      - Channel: `<yourchannelID>`  
    - Name: "Notify on workflow setup error"  
      - Type: Slack node  
      - Message: `WARNING: Error workflow has NOT been setup for: <{{$json.input.url}}|{{$json.input['workflow-name']}}> (No backup will take place until err-workflow is configured)`  
      - Channel: `<yourchannelID>`  
      - Enable markdown

13. **Create GitHub Node to Upload Changes:**  
    - Name: "Upload changes to repo"  
    - Type: GitHub node (edit file)  
    - Owner: `<yourownernamehere>`  
    - Repository: `<yourreponamehere>`  
    - File Path: `N8N_Workflows/{{ $('Set fields').item.json['workflow-export-filename'].replaceAll(/[\\/:*?"<>|,\t\n#%&']/g, "_") }}`  
    - File Content: `{{ $('Set fields').item.json['workflow-definition'] }}`  
    - Commit Message: `updated by N8N #{{ $execution.id }}`  
    - Authentication: OAuth2  
    - On error: continue to create new file node

14. **Create GitHub Node to Create New File:**  
    - Name: "Create new file in repo"  
    - Type: GitHub node (create file)  
    - Same configuration as upload node, commit message "created by N8N"  
    - On error: continue to notify failure node

15. **Create Slack Node to Notify GitHub Upload Failures:**  
    - Name: "Notify on create file in repo fail"  
    - Type: Slack node  
    - Message: `WARNING: Failed to upload new N8N workflow <{{$json.input.url}}|{{$json.input['workflow-name']}}> to repo`  
    - Channel: `<yourchannelID>`  
    - Enable markdown and include link to workflow

16. **Connect Nodes According to Workflow Logic:**  
    - Schedule Trigger → Get active workflows → Check updated in last 7 days → Set fields → Get notion page → Map fields → Check error workflow →  
      - If true: Summarize workflow → Check if new → Add to Notion or Update in Notion → Notify Slack → Upload to GitHub → Create file fallback → Notify on failure  
      - If false: Notify on workflow setup error

17. **Configure Credentials:**  
    - n8n API credential for "Get active workflows" node  
    - Notion API credential for Notion nodes and HTTP request  
    - OpenAI credential for AI summarization node  
    - GitHub OAuth2 credential with repo write access for GitHub nodes  
    - Slack Bot token credential for Slack nodes

18. **Update Parameters:**  
    - Replace placeholder URLs, database IDs, repository owner/name, Slack channel IDs, and n8n host URL in Set fields node  
    - Ensure all regex replacements in filenames handle special characters safely

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow runs weekly on Mondays at 1 AM to avoid heavy load during business hours.        | Scheduling best practice                                                                              |
| Use a private GitHub repository to secure workflow backups.                                   | Security best practice                                                                               |
| Notion database must have columns matching the specified types: text, checkbox, date/time.    | See setup instructions in workflow description                                                     |
| Slack notifications use bot tokens with appropriate channel permissions.                      | Slack API documentation: https://api.slack.com/bot-users                                          |
| OpenAI model "gpt-4o-mini" is used for concise summarization; can be replaced with alternatives | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4o-mini                                |
| Filename sanitization regex replaces characters invalid in GitHub file paths.                  | Prevents commit errors due to invalid filenames                                                     |
| Error workflow flag must be configured in workflows to allow backups; otherwise, Slack warns. | Ensures workflows have error handling configured before backup                                     |
| Workflow diagram and Notion output screenshots are available in the original project files.   | Visual references for workflow structure and output                                                |

---

This documentation provides a complete, detailed reference to understand, reproduce, and maintain the "Automatically document and backup N8N workflows" process, covering all nodes, logic, configurations, and integration points.