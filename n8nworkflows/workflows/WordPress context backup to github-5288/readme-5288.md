WordPress context backup to github

https://n8nworkflows.xyz/workflows/wordpress-context-backup-to-github-5288


# WordPress context backup to github

---
### 1. Workflow Overview

This workflow automates backing up WordPress blog context data into a GitHub repository as JSON files. It is designed to regularly fetch all WordPress posts, format them, compare them with existing backups in GitHub, and then either create a new backup file or update the existing one if changes are detected.

The workflow consists of the following logical blocks:

- **1.1 Trigger and Input Reception**: Handles manual and scheduled triggers to initiate the backup process.
- **1.2 WordPress Data Retrieval and Formatting**: Fetches all WordPress posts and formats them into JSON.
- **1.3 GitHub File Retrieval and Comparison**: Checks if a backup file exists for the WordPress data in GitHub, downloads it if present, and compares it with the current data.
- **1.4 Decision and File Handling**: Based on comparison, decides to create a new file, update an existing file, or do nothing if no changes are detected.
- **1.5 Workflow Completion**: Marks the workflow as done and returns control.

The workflow uses nested subworkflow logic and manages potential edge cases such as large files, missing tags, and GitHub API failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Reception

**Overview:**  
This block initiates the workflow either manually or on a schedule (daily at 5 PM). It also processes input tags and global repository settings.

**Nodes Involved:**  
- Manual Trigger  
- Schedule Trigger  
- Execute Workflow Trigger  
- tag? (Switch)  
- / (Set)  
- Globals (Set)  
- Sticky Note3  

**Node Details:**  

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Allows manual execution of the workflow.  
  - Configuration: Default manual trigger; no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Get All WP Posts".  
  - Edge cases: None.

- **Schedule Trigger**  
  - Type: Trigger node  
  - Role: Automatically triggers workflow daily at 17:00 (5 PM).  
  - Configuration: Interval trigger set to trigger at hour 17.  
  - Inputs: None  
  - Outputs: Connected to "Get All WP Posts".  
  - Edge cases: Timezone considerations may apply.

- **Execute Workflow Trigger**  
  - Type: Execute Workflow Trigger node  
  - Role: Passes through incoming data for further processing.  
  - Configuration: Input source set to passthrough.  
  - Inputs: Manual Trigger or Schedule Trigger indirectly.  
  - Outputs: Connects to "Merge Items" and "tag?" switch node.  
  - Edge cases: None.  

- **tag? (Switch)**  
  - Type: Switch node  
  - Role: Checks if the input JSON has a tag at `tags[0]`.  
  - Configuration: Two outputs: "tag" if tag exists, "none" if not.  
  - Inputs: From "Execute Workflow Trigger".  
  - Outputs: "tag" output connects to "/" node; "none" output connects to "Globals" node.  
  - Edge cases: Missing or malformed tags.

- **/ (Set)**  
  - Type: Set node  
  - Role: Sets the first tag name with a trailing slash in `tags[0].name`.  
  - Configuration: Expression sets `tags[0].name` to `{{$('Execute Workflow Trigger').item.json.tags[0].name}}/`.  
  - Inputs: From "tag?" switch node (tag output).  
  - Outputs: Connects to "Globals" node.  
  - Edge cases: Relies on presence of tag from previous node.

- **Globals (Set)**  
  - Type: Set node  
  - Role: Defines global repository parameters for GitHub operations.  
  - Configuration: Sets three variables:  
    - `repo.owner` = "boagolden82"  
    - `repo.name` = "n8nback"  
    - `repo.path` = `workflows/{{ $json.tags[0].name }}` (uses the tag name with trailing slash)  
  - Inputs: From "/" node or directly from "tag?" none output.  
  - Outputs: Connects to "Get file data" node for GitHub file retrieval.  
  - Edge cases: If no tag, path may be incomplete.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Visual indicator labeled "Edit this node 👇" to highlight where user customization is expected.  
  - Inputs: None  
  - Outputs: None  

---

#### 2.2 WordPress Data Retrieval and Formatting

**Overview:**  
This block fetches all WordPress posts and applies formatting to prepare data for backup.

**Nodes Involved:**  
- Get All WP Posts  
- Format JSON  
- Execute Workflow  

**Node Details:**  

- **Get All WP Posts**  
  - Type: WordPress node  
  - Role: Retrieves all posts from the WordPress site.  
  - Configuration: Operation set to "getAll" with "returnAll" enabled to fetch all posts.  
  - Credentials: Uses configured Wordpress account credentials.  
  - Inputs: From Manual Trigger or Schedule Trigger.  
  - Outputs: Connected to "Format JSON".  
  - Edge cases: API rate limiting, authentication errors.

- **Format JSON**  
  - Type: Code node  
  - Role: Example code adding a dummy field `myNewField` with value 1 to each post's JSON (placeholder for actual formatting).  
  - Configuration: JavaScript code iterates all input items and appends `myNewField`.  
  - Inputs: From "Get All WP Posts".  
  - Outputs: Connects to "Execute Workflow" node.  
  - Edge cases: Code execution errors.

- **Execute Workflow**  
  - Type: Execute Workflow node  
  - Role: Executes the current workflow for each item separately (mode "each").  
  - Configuration: Uses this workflow's ID to recursively process each post or batch individually.  
  - Inputs: From "Format JSON".  
  - Outputs: Connects back to "Merge Items" in the GitHub comparison block (via subworkflow logic).  
  - Edge cases: Recursive execution may cause complexity or performance issues.

---

#### 2.3 GitHub File Retrieval and Comparison

**Overview:**  
This block fetches the existing backup file from GitHub, downloads its contents, and compares the stored backup with the current WordPress data to detect changes.

**Nodes Involved:**  
- Get file data (GitHub)  
- If file too large (If)  
- Get File (HTTP Request)  
- Merge Items  
- isDiffOrNew (Code)  
- Check Status (Switch)  

**Node Details:**  

- **Get file data**  
  - Type: GitHub node  
  - Role: Attempts to retrieve the backup JSON file from GitHub.  
  - Configuration:  
    - Owner, repository, and file path dynamically set from global settings and current workflow id.  
    - Operation: get file.  
    - Continue on fail: true (allows workflow to continue if file missing).  
  - Credentials: GitHub OAuth credentials required.  
  - Inputs: From "Globals" node.  
  - Outputs: Connects to "If file too large".  
  - Edge cases: File not found, permission errors, API rate limits.

- **If file too large**  
  - Type: If node  
  - Role: Checks if the file content is empty and no error exists.  
  - Configuration:  
    - Condition checks if `content` field is empty and `error` field does not exist.  
  - Inputs: From "Get file data".  
  - Outputs: Two outputs:  
    - True (file too large or missing content) connects to "Get File".  
    - False connects to "Merge Items".  
  - Edge cases: Inconsistent file content or API response.

- **Get File**  
  - Type: HTTP Request node  
  - Role: Downloads the file content using the URL from GitHub API response.  
  - Configuration: URL is extracted from the `download_url` field of the previous node’s JSON.  
  - Inputs: From "If file too large" true output.  
  - Outputs: Connects to "Merge Items".  
  - Edge cases: HTTP errors, large file download issues.

- **Merge Items**  
  - Type: Merge node  
  - Role: Combines the downloaded GitHub file content and current WordPress data for comparison.  
  - Configuration: Default merge combining inputs.  
  - Inputs: From "Get File" and "Execute Workflow Trigger" (passing current data).  
  - Outputs: Connects to "isDiffOrNew" code node.  
  - Edge cases: Input data mismatch.

- **isDiffOrNew**  
  - Type: Code node  
  - Role: Compares the existing GitHub backup with the current WordPress data to determine if they are the same, different, or new.  
  - Configuration:  
    - Orders JSON keys for consistent comparison.  
    - Decodes base64 content when needed.  
    - Sets a status field `github_status` in the output JSON: "same", "different", or "new".  
    - Prepares a stringified JSON for upload if different or new.  
  - Inputs: From "Merge Items".  
  - Outputs: Connects to "Check Status".  
  - Edge cases: JSON parsing errors, base64 decoding errors.

- **Check Status**  
  - Type: Switch node  
  - Role: Routes workflow based on `github_status` ("same", "different", "new").  
  - Configuration: Three outputs mapped to status values.  
  - Inputs: From "isDiffOrNew".  
  - Outputs:  
    - "same" → "Same file - Do nothing"  
    - "different" → "File is different"  
    - "new" → "File is new"  
  - Edge cases: Unexpected status values.

---

#### 2.4 Decision and File Handling

**Overview:**  
This block acts upon the comparison outcome by either doing nothing, updating the existing file, or creating a new file in GitHub.

**Nodes Involved:**  
- Same file - Do nothing (NoOp)  
- File is different (NoOp)  
- File is new (NoOp)  
- Edit existing file (GitHub)  
- Create new file (GitHub)  

**Node Details:**  

- **Same file - Do nothing**  
  - Type: NoOp (no operation) node  
  - Role: Represents the path where no backup update is needed.  
  - Inputs: From "Check Status" (same output).  
  - Outputs: Connects to "Return".  
  - Edge cases: None.

- **File is different**  
  - Type: NoOp node  
  - Role: Placeholder representing detected differences; leads to file update.  
  - Inputs: From "Check Status" (different output).  
  - Outputs: Connects to "Edit existing file".  
  - Edge cases: None.

- **File is new**  
  - Type: NoOp node  
  - Role: Placeholder representing a new backup file is needed.  
  - Inputs: From "Check Status" (new output).  
  - Outputs: Connects to "Create new file".  
  - Edge cases: None.

- **Edit existing file**  
  - Type: GitHub node  
  - Role: Updates an existing file in GitHub repository with new content.  
  - Configuration:  
    - Owner, repo, and file path derived from global variables and current workflow id.  
    - Operation: edit file.  
    - File content: uses stringified JSON from "isDiffOrNew".  
    - Commit message includes the workflow name and github_status.  
  - Credentials: GitHub OAuth credentials.  
  - Inputs: From "File is different".  
  - Outputs: Connects to "Return".  
  - Edge cases: Conflicts, permission errors.

- **Create new file**  
  - Type: GitHub node  
  - Role: Creates a new file in the GitHub repository for backup.  
  - Configuration:  
    - Owner, repo, and file path similar to edit node.  
    - Operation: create file.  
    - File content and commit message similar to edit node.  
  - Credentials: GitHub OAuth credentials.  
  - Inputs: From "File is new".  
  - Outputs: Connects to "Return".  
  - Edge cases: File already exists, permission errors.

---

#### 2.5 Workflow Completion

**Overview:**  
Finalizes the workflow by returning a completion status for downstream processes.

**Nodes Involved:**  
- Return (Set)  

**Node Details:**  

- **Return**  
  - Type: Set node  
  - Role: Sets a boolean field `Done` to true, indicating successful completion.  
  - Configuration: Assigns `Done` = true.  
  - Inputs: From "Create new file", "Edit existing file", and "Same file - Do nothing".  
  - Outputs: None (end of workflow).  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role                          | Input Node(s)                    | Output Node(s)                   | Sticky Note                                |
|------------------------|-------------------------|----------------------------------------|---------------------------------|---------------------------------|--------------------------------------------|
| Manual Trigger         | Trigger                 | Manual start of workflow                | None                            | Get All WP Posts                |                                            |
| Schedule Trigger       | Trigger                 | Scheduled daily trigger at 5 PM         | None                            | Get All WP Posts                |                                            |
| Execute Workflow Trigger | Execute Workflow Trigger | Passes input data through               | Manual Trigger, Schedule Trigger | Merge Items, tag?               |                                            |
| tag?                   | Switch                  | Checks if tag exists in input            | Execute Workflow Trigger        | / (for tag), Globals (for none) |                                            |
| /                      | Set                     | Sets tag name with trailing slash        | tag?                           | Globals                        |                                            |
| Globals                | Set                     | Defines GitHub repo owner, name, path    | / or tag?                     | Get file data                  |                                            |
| Get file data          | GitHub                  | Retrieves backup file from GitHub        | Globals                        | If file too large              |                                            |
| If file too large      | If                      | Checks if file content is empty          | Get file data                  | Get File (true), Merge Items (false) |                                            |
| Get File               | HTTP Request            | Downloads file content from GitHub URL   | If file too large (true)       | Merge Items                   |                                            |
| Merge Items            | Merge                   | Combines GitHub file and current data    | Get File, Execute Workflow Trigger | isDiffOrNew                 |                                            |
| isDiffOrNew            | Code                    | Compares backup and current data          | Merge Items                   | Check Status                  |                                            |
| Check Status           | Switch                  | Routes workflow based on comparison result | isDiffOrNew                 | Same file, File is different, File is new |                                            |
| Same file - Do nothing | NoOp                    | Does nothing if files are identical       | Check Status ("same")          | Return                       |                                            |
| File is different      | NoOp                    | Placeholder for different file handling   | Check Status ("different")     | Edit existing file           |                                            |
| File is new            | NoOp                    | Placeholder for new file handling         | Check Status ("new")           | Create new file              |                                            |
| Edit existing file     | GitHub                  | Updates existing backup file on GitHub    | File is different              | Return                       |                                            |
| Create new file        | GitHub                  | Creates new backup file on GitHub          | File is new                   | Return                       |                                            |
| Return                 | Set                     | Marks workflow completion with Done=true | Same file, Edit existing file, Create new file | None                 |                                            |
| Get All WP Posts       | WordPress               | Retrieves all WordPress posts             | Manual Trigger, Schedule Trigger | Format JSON                 |                                            |
| Format JSON            | Code                    | Formats WP posts data (adds dummy field)  | Get All WP Posts              | Execute Workflow             | 将所有WP文章格式化为JSON文本 (Chinese: Formats all WP posts as JSON text) |
| Execute Workflow       | Execute Workflow        | Recursively processes each post separately | Format JSON                  | Merge Items (subworkflow)    |                                            |
| Sticky Note3           | Sticky Note             | Visual note "Edit this node 👇"           | None                        | None                        |                                            |
| Sticky Note            | Sticky Note             | Visual note "## Subworkflow"               | None                        | None                        |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - No configuration needed.  
   - This node initiates the workflow manually.

2. **Create Schedule Trigger node**  
   - Set to trigger daily at 17:00 (5 PM).

3. **Create Execute Workflow Trigger node**  
   - Set input source to passthrough.  
   - Connect Manual Trigger and Schedule Trigger outputs to this node.

4. **Create Switch node "tag?"**  
   - Configure two outputs:  
     - "tag": Condition checks if `tags[0]` exists.  
     - "none": Condition for when `tags[0]` does not exist.  
   - Connect Execute Workflow Trigger output to this switch.

5. **Create Set node "/"**  
   - Assign `tags[0].name` to expression: `{{$('Execute Workflow Trigger').item.json.tags[0].name}}/`  
   - Connect "tag" output of "tag?" to this node.

6. **Create Set node "Globals"**  
   - Define variables:  
     - `repo.owner` = "boagolden82"  
     - `repo.name` = "n8nback"  
     - `repo.path` = `workflows/{{ $json.tags[0].name }}`  
   - Connect "/" node output and also connect "none" output of "tag?" here.

7. **Create GitHub node "Get file data"**  
   - Operation: get file.  
   - Owner: `={{ $json.repo.owner }}`  
   - Repository: `={{ $json.repo.name }}`  
   - File Path: `={{ $json.repo.path }}{{ $('Execute Workflow Trigger').item.json.id }}.json`  
   - Enable continue on fail and always output data.  
   - Connect "Globals" output here.  
   - Use GitHub OAuth2 credentials.

8. **Create If node "If file too large"**  
   - Condition: Check if `content` is empty AND `error` does not exist.  
   - True output connects to "Get File" node.  
   - False output connects to "Merge Items" node.

9. **Create HTTP Request node "Get File"**  
   - URL: `={{ $json.download_url }}` from previous node.  
   - Connect True output of "If file too large" here.

10. **Create Merge node "Merge Items"**  
    - Default merge mode.  
    - Connect outputs from "Get File" and "Execute Workflow Trigger" (current data) here.

11. **Create Code node "isDiffOrNew"**  
    - Paste the JavaScript code that:  
      - Orders JSON keys,  
      - Decodes base64 if needed,  
      - Compares GitHub backup and current data,  
      - Sets `github_status` to "same", "different", or "new",  
      - Prepares stringified JSON for upload.  
    - Connect from "Merge Items" here.

12. **Create Switch node "Check Status"**  
    - Use `{{$json.github_status}}` to route to three outputs:  
      - "same" → Connect to "Same file - Do nothing"  
      - "different" → "File is different"  
      - "new" → "File is new"

13. **Create NoOp node "Same file - Do nothing"**  
    - Connect "same" output of "Check Status" here.

14. **Create NoOp node "File is different"**  
    - Connect "different" output of "Check Status" here.

15. **Create NoOp node "File is new"**  
    - Connect "new" output of "Check Status" here.

16. **Create GitHub node "Edit existing file"**  
    - Operation: edit file  
    - Owner, repo, filePath same as "Get file data" node.  
    - File Content: `={{$('isDiffOrNew').item.json["n8n_data_stringy"]}}`  
    - Commit message: `={{$('Execute Workflow Trigger').first().json.name}} ({{$json.github_status}})`  
    - Credentials: GitHub OAuth2  
    - Connect "File is different" output here.

17. **Create GitHub node "Create new file"**  
    - Operation: create file  
    - Owner, repo, filePath same as above.  
    - File Content and Commit message same as above.  
    - Credentials: GitHub OAuth2  
    - Connect "File is new" output here.

18. **Create Set node "Return"**  
    - Assignment: Set boolean `Done` = true  
    - Connect outputs of "Same file - Do nothing", "Edit existing file", and "Create new file" here.

19. **Create WordPress node "Get All WP Posts"**  
    - Operation: getAll  
    - Return All: true  
    - Credentials: WordPress API credentials  
    - Connect outputs of Manual Trigger and Schedule Trigger here (parallel to Execute Workflow Trigger).

20. **Create Code node "Format JSON"**  
    - JavaScript code to format all posts, example adding dummy field `myNewField=1`.  
    - Connect "Get All WP Posts" output here.

21. **Create Execute Workflow node**  
    - Mode: each  
    - Workflow ID: current workflow’s ID  
    - Connect "Format JSON" output here.  
    - This node invokes the same workflow recursively to handle data items.

22. **Add Sticky Notes**  
    - Add a sticky note labeled "Edit this node 👇" near the "/" Set node for user guidance.  
    - Add a sticky note labeled "## Subworkflow" near the recursive Execute Workflow node cluster.

---

### 5. General Notes & Resources

| Note Content                                                     | Context or Link                                   |
|-----------------------------------------------------------------|--------------------------------------------------|
| The JSON formatting code in "Format JSON" node currently adds a dummy field (`myNewField = 1`). Customize this node to format WordPress posts as needed. | Node "Format JSON"                                    |
| GitHub API credentials require OAuth2 setup with appropriate repository access. | GitHub nodes require OAuth2 credentials setup.    |
| WordPress API credentials require valid authentication with access to read posts. | WordPress node requires configured credentials.  |
| The workflow uses recursive execution (Execute Workflow node in "each" mode) to process individual posts or batches. Be mindful of performance and recursion limits. | "Execute Workflow" node and subworkflow logic.    |
| The workflow assumes that each WordPress post or dataset has a unique `id` used as the filename for backup JSON in GitHub. | Filename pattern: `{{workflow id}}.json`          |

---

**Disclaimer:**  
The provided workflow is fully automated via n8n integration tool, respecting all content policies, and handles only legal and public data. No illegal or protected content is processed.

---