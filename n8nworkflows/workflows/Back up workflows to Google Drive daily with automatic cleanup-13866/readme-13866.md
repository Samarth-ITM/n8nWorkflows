Back up workflows to Google Drive daily with automatic cleanup

https://n8nworkflows.xyz/workflows/back-up-workflows-to-google-drive-daily-with-automatic-cleanup-13866


# Back up workflows to Google Drive daily with automatic cleanup

# 1. Workflow Overview

This workflow creates a daily backup of all n8n workflows into Google Drive and automatically deletes old backup folders after 15 days.

It is split into two independent scheduled branches:

- **Daily backup branch**: creates a dated folder in Google Drive, fetches all workflows from the n8n REST API, converts each workflow to a JSON file, and uploads the files into the new folder.
- **Cleanup branch**: lists existing backup folders in a target Google Drive parent folder, calculates folder age from the creation date, and deletes folders older than 15 days.

## 1.1 Backup Trigger and Folder Preparation
A schedule trigger runs daily at 04:01. It creates a new folder in a predefined Google Drive parent folder using the current date as the folder name, then stores the new folder ID for later upload steps.

## 1.2 Workflow Export and File Generation
After folder creation, the workflow calls the n8n API to retrieve all workflows, splits the returned array into one item per workflow, and converts each item into an individual `.json` file.

## 1.3 Upload to Google Drive
Each generated JSON file is briefly delayed, then uploaded into the folder created earlier.

## 1.4 Cleanup Trigger and Retention Enforcement
A second schedule trigger runs daily at 03:01. It lists folders/files under the backup parent folder, computes how many days have passed since each item’s `createdTime`, and deletes any folder older than 15 days.

---

# 2. Block-by-Block Analysis

## Block 1 — Documentation and In-Canvas Notes

### Overview
This block contains only sticky notes used to describe the workflow inside the n8n canvas. These notes do not affect execution but provide important operational context.

### Nodes Involved
- `Sticky Note`
- `Sticky Note5`
- `Sticky Note6`

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`; visual documentation node.
- **Configuration choices:** Contains a full description of the workflow purpose, behavior, and setup checklist.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None at runtime; purely informational.
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and role:** `n8n-nodes-base.stickyNote`; labels the cleanup branch.
- **Configuration choices:** Content is `Delete after 15 days`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and role:** `n8n-nodes-base.stickyNote`; labels the daily backup branch.
- **Configuration choices:** Content is `Daily Backup`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## Block 2 — Daily Backup Trigger and Google Drive Folder Creation

### Overview
This block starts the backup process on a daily schedule and creates a destination folder in Google Drive named from the current date. That folder ID is then passed forward so all backup files are uploaded to the correct location.

### Nodes Involved
- `start4h`
- `Create folder`
- `Format data`

### Node Details

#### start4h
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry point for the backup branch.
- **Configuration choices:** Configured to run daily at **04:01**.
- **Key expressions or variables used:** The Schedule Trigger provides date-related fields that are later used by `Create folder`, such as `Month`, `Day of month`, and `Year`.
- **Input and output connections:** No input; outputs to `Create folder`.
- **Version-specific requirements:** Type version 1.2.
- **Edge cases or potential failure types:**
  - Timezone differences between server timezone and expected business timezone.
  - Missed executions if the n8n instance is down at scheduled time.
- **Sub-workflow reference:** None.

#### Create folder
- **Type and role:** `n8n-nodes-base.googleDrive`; creates a folder in Google Drive.
- **Configuration choices:**
  - Resource: **folder**
  - Parent folder: fixed Google Drive folder ID `1HJYupeDiynEpa58lmXas-wRAD3zrpbIf`
  - Drive: `My Drive`
  - Folder name expression: `{{$json.Month}}-{{$json["Day of month"]}}-{{$json.Year}}`
- **Key expressions or variables used:**
  - Uses Schedule Trigger output fields to build the folder name.
- **Input and output connections:** Input from `start4h`; output to `Format data`.
- **Version-specific requirements:** Type version 3; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Invalid or revoked Google Drive OAuth2 credentials.
  - Parent folder ID not found or no permission to create inside it.
  - Non-unique folder names if workflow runs multiple times the same day.
  - Locale-dependent month name output could affect naming consistency.
- **Sub-workflow reference:** None.

#### Format data
- **Type and role:** `n8n-nodes-base.set`; stores selected values from the created folder for later cross-node access.
- **Configuration choices:**
  - Creates field `data` from `{{$json.name}}`
  - Creates field `id` from `{{$json.id}}`
- **Key expressions or variables used:**
  - `{{$json.name}}`
  - `{{$json.id}}`
- **Input and output connections:** Input from `Create folder`; output to `apiN8N`.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If folder creation fails, this node will never execute.
  - If the Google Drive response structure changes, expression references may fail.
- **Sub-workflow reference:** None.

---

## Block 3 — Fetching Workflows from n8n and Splitting Them

### Overview
This block retrieves all workflows from the n8n API and transforms the returned array into one item per workflow. That itemized structure is necessary for generating one JSON file per workflow.

### Nodes Involved
- `apiN8N`
- `Split`

### Node Details

#### apiN8N
- **Type and role:** `n8n-nodes-base.httpRequest`; calls the n8n REST API to list workflows.
- **Configuration choices:**
  - Method defaults to GET.
  - URL: `https://noiton.atendon.com.br/api/v1/workflows`
  - Sends headers:
    - `accept: application/json`
    - `X-N8N-API-KEY: eyJ_YOUR_JWT_TOKEN_HERE`
- **Key expressions or variables used:** No expressions in the URL or headers.
- **Input and output connections:** Input from `Format data`; output to `Split`.
- **Version-specific requirements:** Type version 4.2.
- **Edge cases or potential failure types:**
  - Invalid API key or expired/revoked access.
  - Wrong base URL for the n8n instance.
  - API unavailable, timeout, DNS error, SSL/TLS error.
  - Pagination issue: this node assumes the endpoint returns the full workflow list in `data`. If the instance/API version paginates responses differently, some workflows may be missed.
  - Sensitive value is hardcoded in node configuration; this is a security risk and should be moved to credentials or variables.
- **Sub-workflow reference:** None.

#### Split
- **Type and role:** `n8n-nodes-base.code`; converts the returned array of workflows into separate n8n items.
- **Configuration choices:**
  - JavaScript code reads `$input.first().json.data`
  - Maps each workflow object into `{ json: workflow }`
- **Key expressions or variables used:**
  - `$input.first().json.data`
- **Input and output connections:** Input from `apiN8N`; output to `ConvertToJson`.
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**
  - If the API response does not contain a `data` array, code throws or returns unexpected output.
  - Empty workflow list results in zero output items.
  - Very large workflow counts could increase execution time and memory use.
- **Sub-workflow reference:** None.

---

## Block 4 — JSON File Generation and Upload

### Overview
This block converts each workflow item into a JSON file and uploads it to the Google Drive folder created earlier. It relies on cross-node expressions to reference both the current workflow item and the folder created at the beginning of the backup branch.

### Nodes Involved
- `ConvertToJson`
- `5s`
- `upload file`

### Node Details

#### ConvertToJson
- **Type and role:** `n8n-nodes-base.convertToFile`; converts each workflow JSON object into a file item.
- **Configuration choices:**
  - Operation: `toJson`
  - Mode: `each`
  - File naming enabled with expression `{{$json.name}}.json`
- **Key expressions or variables used:**
  - `{{$json.name}}.json`
- **Input and output connections:** Input from `Split`; output to `5s`.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or potential failure types:**
  - Workflow names with unsupported filename characters may cause upload issues or inconsistent naming.
  - Duplicate workflow names may overwrite or conflict depending on Google Drive behavior and node settings.
  - If a workflow item lacks `name`, filename generation may fail or produce an empty filename.
- **Sub-workflow reference:** None.

#### 5s
- **Type and role:** `n8n-nodes-base.wait`; pauses briefly before upload.
- **Configuration choices:** No explicit wait duration is configured in the provided JSON, so behavior depends on node defaults/version behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `ConvertToJson`; output to `upload file`.
- **Version-specific requirements:** Type version 1.1.
- **Edge cases or potential failure types:**
  - If no delay is effectively configured, this node adds little practical value.
  - Wait nodes can resume via internal execution data; queue or persistence issues can affect continuation in some deployments.
- **Sub-workflow reference:** None.

#### upload file
- **Type and role:** `n8n-nodes-base.googleDrive`; uploads the generated file into the backup folder.
- **Configuration choices:**
  - Google Drive node using binary input from previous step
  - File/folder name expression: `{{$('Split').item.json.name}}`
  - Destination folder ID expression: `{{$('Format data').item.json.id}}`
  - Drive: `My Drive`
- **Key expressions or variables used:**
  - `{{$('Split').item.json.name}}`
  - `{{$('Format data').item.json.id}}`
- **Input and output connections:** Input from `5s`; no downstream output.
- **Version-specific requirements:** Type version 3; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:**
  - If the expression cannot resolve `Format data` item context, uploads fail.
  - If the folder was not created or was deleted, upload fails.
  - Missing binary data from `ConvertToJson` causes upload failure.
  - Duplicate filenames may create duplicates or versioning ambiguity in Drive.
  - OAuth permission issues or Google Drive quota limits.
- **Sub-workflow reference:** None.

---

## Block 5 — Cleanup Trigger, Folder Age Calculation, and Deletion

### Overview
This block runs on a separate schedule, lists items inside the backup root folder, computes the age of each item from its `createdTime`, and deletes any folder older than 15 days.

### Nodes Involved
- `start3h`
- `get folder`
- `Check date`
- `>15d`
- `Delete folder`

### Node Details

#### start3h
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry point for cleanup.
- **Configuration choices:** Configured to run daily at **03:01**.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `get folder`.
- **Version-specific requirements:** Type version 1.2.
- **Edge cases or potential failure types:**
  - Timezone mismatch.
  - Cleanup may run before backup on the same day, which is fine here because retention is 15 days.
- **Sub-workflow reference:** None.

#### get folder
- **Type and role:** `n8n-nodes-base.googleDrive`; lists files/folders in the backup parent folder.
- **Configuration choices:**
  - Resource: `fileFolder`
  - Search scope: within parent folder `1HJYupeDiynEpa58lmXas-wRAD3zrpbIf`
  - Search type: `all`
  - Return all results enabled
  - Requested fields: `*`
- **Key expressions or variables used:** None in main configuration.
- **Input and output connections:** Input from `start3h`; output to `Check date`.
- **Version-specific requirements:** Type version 3; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Lists both files and folders because search type is `all`; if files exist directly in the parent folder, later deletion logic may fail because delete operation expects a folder ID.
  - Large numbers of items may increase runtime.
  - Credentials or permissions issues.
- **Sub-workflow reference:** None.

#### Check date
- **Type and role:** `n8n-nodes-base.dateTime`; computes time difference between item `createdTime` and current date.
- **Configuration choices:**
  - Operation: `getTimeBetweenDates`
  - Start date: `{{ new Date($json.createdTime).toISOString().split('T')[0] }}`
  - End date: `{{ new Date().toISOString().split('T')[0] }}`
- **Key expressions or variables used:**
  - `new Date($json.createdTime).toISOString().split('T')[0]`
  - `new Date().toISOString().split('T')[0]`
- **Input and output connections:** Input from `get folder`; output to `>15d`.
- **Version-specific requirements:** Type version 2.
- **Edge cases or potential failure types:**
  - If `createdTime` is missing or malformed, date parsing fails.
  - Because dates are truncated to `YYYY-MM-DD`, partial-day precision is lost.
  - Timezone normalization to UTC may shift borderline items by one day.
- **Sub-workflow reference:** None.

#### >15d
- **Type and role:** `n8n-nodes-base.if`; filters items older than 15 days.
- **Configuration choices:**
  - Compares `{{$json.timeDifference.days}} > 15`
  - Loose type validation enabled
- **Key expressions or variables used:**
  - `{{$json.timeDifference.days}}`
- **Input and output connections:** Input from `Check date`; true output to `Delete folder`. No false branch connected.
- **Version-specific requirements:** Type version 2.2.
- **Edge cases or potential failure types:**
  - If `timeDifference.days` is missing or non-numeric, condition may behave unexpectedly under loose validation.
  - Current logic deletes only when strictly greater than 15, not equal to 15.
- **Sub-workflow reference:** None.

#### Delete folder
- **Type and role:** `n8n-nodes-base.googleDrive`; deletes folders that meet the retention rule.
- **Configuration choices:**
  - Resource: `folder`
  - Operation: `deleteFolder`
  - Folder ID expression: `{{$('get folder').item.json.id}}`
- **Key expressions or variables used:**
  - `{{$('get folder').item.json.id}}`
- **Input and output connections:** Input from `>15d`; no downstream output.
- **Version-specific requirements:** Type version 3; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:**
  - If listed item is not actually a folder, delete may fail.
  - Folder may already be deleted or inaccessible.
  - Lack of permission to delete.
  - Cross-node item linking must resolve correctly for each candidate item.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| start4h | Schedule Trigger | Starts the daily backup branch at 04:01 |  | Create folder | ## Daily Backup |
| Create folder | Google Drive | Creates a dated backup folder inside the configured parent Google Drive folder | start4h | Format data | ## Daily Backup |
| Format data | Set | Stores the created folder name and ID for later upload targeting | Create folder | apiN8N | ## Daily Backup |
| apiN8N | HTTP Request | Fetches all workflows from the n8n REST API | Format data | Split | ## Daily Backup |
| Split | Code | Splits the API response array into one item per workflow | apiN8N | ConvertToJson | ## Daily Backup |
| ConvertToJson | Convert to File | Converts each workflow JSON object into a `.json` file | Split | 5s | ## Daily Backup |
| 5s | Wait | Pauses before file upload | ConvertToJson | upload file | ## Daily Backup |
| upload file | Google Drive | Uploads each generated JSON file to the created backup folder | 5s |  | ## Daily Backup |
| start3h | Schedule Trigger | Starts the cleanup branch at 03:01 |  | get folder | ## Delete after 15 days |
| get folder | Google Drive | Lists all items under the backup parent folder | start3h | Check date | ## Delete after 15 days |
| Check date | Date & Time | Calculates age difference between item creation date and current date | get folder | >15d | ## Delete after 15 days |
| >15d | If | Filters items older than 15 days | Check date | Delete folder | ## Delete after 15 days |
| Delete folder | Google Drive | Deletes folders older than the retention period | >15d |  | ## Delete after 15 days |
| Sticky Note | Sticky Note | General in-canvas workflow documentation |  |  | ## n8n Workflow Backup to Google Drive  This workflow automatically creates daily backups of all n8n workflows and stores them in Google Drive. It also removes old backups automatically to keep storage organized.  The workflow runs on a schedule and exports all workflows using the n8n API. Each workflow is saved as an individual .json file inside a folder created with the current date.  A second scheduled process checks existing backup folders and deletes folders older than 15 days.  This ensures you always have recent backups while keeping your storage clean.  ## How it works  1. A Schedule Trigger starts the daily backup 2. A folder is created in Google Drive using the current date 3. The n8n API retrieves all workflows 4. Workflows are split into individual items 5. Each workflow is converted into a .json file 6. Files are uploaded to the created Google Drive folder 7. Another schedule lists existing backup folders 8. The workflow calculates how many days old each folder is 9. Folders older than 15 days are deleted  ## Setup steps  1. Connect a Google Drive OAuth credential 2. Set the Google Drive root folder ID for backups 3. Configure access to the n8n API 4. Adjust the schedule times if needed 5. Adjust the retention period (default 15 days) |
| Sticky Note5 | Sticky Note | Labels the cleanup area |  |  | ## Delete after 15 days |
| Sticky Note6 | Sticky Note | Labels the backup area |  |  | ## Daily Backup |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Auto backup n8n workflows to Google Drive with scheduled cleanup`.

2. **Add the first Schedule Trigger for backup**
   - Node type: **Schedule Trigger**
   - Name: `start4h`
   - Configure it to run **every day at 04:01**.
   - Keep in mind the execution uses your n8n instance timezone.

3. **Add a Google Drive node to create the daily folder**
   - Node type: **Google Drive**
   - Name: `Create folder`
   - Resource: **Folder**
   - Operation: **Create folder** (implicit by using folder resource in create mode)
   - Drive: **My Drive**
   - Parent folder: choose the folder where all backups will live.
   - In this workflow, the parent folder ID is `1HJYupeDiynEpa58lmXas-wRAD3zrpbIf`.
   - Set folder name with an expression using schedule fields:
     - `{{$json.Month}}-{{$json["Day of month"]}}-{{$json.Year}}`
   - Attach **Google Drive OAuth2** credentials with permission to create folders and upload files.
   - Connect `start4h -> Create folder`.

4. **Add a Set node to store the folder ID**
   - Node type: **Set**
   - Name: `Format data`
   - Create two fields:
     - `data` = `{{$json.name}}`
     - `id` = `{{$json.id}}`
   - This keeps the created folder ID easy to reference later.
   - Connect `Create folder -> Format data`.

5. **Add an HTTP Request node to query the n8n API**
   - Node type: **HTTP Request**
   - Name: `apiN8N`
   - Method: **GET**
   - URL: your n8n instance API endpoint for workflows, e.g.:
     - `https://your-n8n-domain/api/v1/workflows`
   - Enable headers and add:
     - `accept` = `application/json`
     - `X-N8N-API-KEY` = your n8n API key
   - Connect `Format data -> apiN8N`.
   - Prefer storing the API key outside the node body if possible, for example via environment or credential strategy.

6. **Add a Code node to split the workflow list**
   - Node type: **Code**
   - Name: `Split`
   - Paste logic equivalent to:
     - Read the array from the API response field `data`
     - Return one n8n item per workflow
   - Functional code behavior:
     - `const workflows = $input.first().json.data;`
     - `return workflows.map(workflow => ({ json: workflow }));`
   - Connect `apiN8N -> Split`.

7. **Add a Convert to File node**
   - Node type: **Convert to File**
   - Name: `ConvertToJson`
   - Operation: **To JSON**
   - Mode: **Each**
   - Set filename expression:
     - `{{$json.name}}.json`
   - Enable formatting if desired, as in the source workflow.
   - Connect `Split -> ConvertToJson`.

8. **Add a Wait node**
   - Node type: **Wait**
   - Name: `5s`
   - Place it after file conversion.
   - The provided workflow does not explicitly define a delay value, so if you want deterministic behavior, configure a short wait such as **5 seconds**.
   - Connect `ConvertToJson -> 5s`.

9. **Add a Google Drive upload node**
   - Node type: **Google Drive**
   - Name: `upload file`
   - Configure it to upload the binary file coming from the previous node.
   - Set the file name from the current workflow item:
     - `{{$('Split').item.json.name}}`
   - Set the destination folder ID from the folder created earlier:
     - `{{$('Format data').item.json.id}}`
   - Use the same **Google Drive OAuth2** credential as above.
   - Connect `5s -> upload file`.

10. **Add the second Schedule Trigger for cleanup**
    - Node type: **Schedule Trigger**
    - Name: `start3h`
    - Configure it to run **every day at 03:01**.
    - Connect nothing into it; it is a separate entry point.

11. **Add a Google Drive listing node for existing backup folders**
    - Node type: **Google Drive**
    - Name: `get folder`
    - Resource: **File/Folder**
    - Search/list under the same parent backup folder used in `Create folder`
    - Parent folder ID: `1HJYupeDiynEpa58lmXas-wRAD3zrpbIf`
    - Search scope/type: **all**
    - Enable **Return All**
    - Request fields: `*`
    - Connect `start3h -> get folder`.

12. **Add a Date & Time node to calculate item age**
    - Node type: **Date & Time**
    - Name: `Check date`
    - Operation: **Get Time Between Dates**
    - Start date expression:
      - `{{ new Date($json.createdTime).toISOString().split('T')[0] }}`
    - End date expression:
      - `{{ new Date().toISOString().split('T')[0] }}`
    - Connect `get folder -> Check date`.

13. **Add an If node for retention filtering**
    - Node type: **If**
    - Name: `>15d`
    - Configure one numeric condition:
      - Left value: `{{$json.timeDifference.days}}`
      - Operator: **greater than**
      - Right value: `15`
    - Leave only the true branch connected.
    - Connect `Check date -> >15d`.

14. **Add a Google Drive delete-folder node**
    - Node type: **Google Drive**
    - Name: `Delete folder`
    - Resource: **Folder**
    - Operation: **Delete Folder**
    - Folder ID expression:
      - `{{$('get folder').item.json.id}}`
    - Use the same Google Drive OAuth2 credentials.
    - Connect the **true** output of `>15d -> Delete folder`.

15. **Optionally add sticky notes**
    - Add one general note describing the workflow.
    - Add one note above the backup branch: `Daily Backup`.
    - Add one note above the cleanup branch: `Delete after 15 days`.

16. **Credential setup**
    - **Google Drive OAuth2**
      - Must allow creating folders, listing content, uploading files, and deleting folders.
      - Ensure the authenticated account has access to the configured backup parent folder.
    - **n8n API access**
      - Generate an API key in your n8n instance.
      - Update the HTTP Request node header `X-N8N-API-KEY`.
      - Confirm the API endpoint path matches your n8n version and deployment.

17. **Test the backup branch**
    - Execute from `start4h` manually or temporarily replace with manual testing.
    - Confirm:
      - A dated folder is created.
      - API returns workflow data.
      - One `.json` file per workflow is generated.
      - Files land in the created folder.

18. **Test the cleanup branch**
    - Execute from `start3h`.
    - Verify `get folder` returns items with `createdTime`.
    - Verify `Check date` creates `timeDifference.days`.
    - Verify only items older than 15 days pass the IF node.
    - Test carefully in a non-production folder before enabling deletion.

19. **Activate the workflow**
    - Turn the workflow on once both branches behave correctly.

## Important implementation constraints
- The cleanup branch currently lists **all** items in the backup parent folder. If non-folder files exist there, the delete-folder step may fail.
- The retention rule is **strictly greater than 15** days, not greater than or equal to 15.
- The upload branch depends on cross-node references to `Format data` and `Split`; keep node names unchanged if you reuse the same expressions.
- The API response is assumed to contain a `data` array of workflows.

## Recommended improvements when rebuilding
- Filter cleanup results to folders only.
- Store the n8n API key securely rather than hardcoding it.
- Handle pagination if your API returns partial workflow lists.
- Sanitize file names before upload.
- Configure the Wait node explicitly if a delay is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| n8n Workflow Backup to Google Drive. This workflow automatically creates daily backups of all n8n workflows and stores them in Google Drive. It also removes old backups automatically to keep storage organized. | In-canvas general documentation |
| The workflow runs on a schedule and exports all workflows using the n8n API. Each workflow is saved as an individual `.json` file inside a folder created with the current date. | In-canvas general documentation |
| A second scheduled process checks existing backup folders and deletes folders older than 15 days. | In-canvas general documentation |
| Setup steps noted in the canvas: connect a Google Drive OAuth credential, set the Google Drive root folder ID, configure n8n API access, adjust the schedule times, and adjust the retention period. | In-canvas general documentation |