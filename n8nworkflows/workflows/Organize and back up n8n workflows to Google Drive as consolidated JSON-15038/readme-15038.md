Organize and back up n8n workflows to Google Drive as consolidated JSON

https://n8nworkflows.xyz/workflows/organize-and-back-up-n8n-workflows-to-google-drive-as-consolidated-json-15038


# Organize and back up n8n workflows to Google Drive as consolidated JSON

### 1. Workflow Overview

This workflow automates the organization and backup of all n8n workflows from a given instance by categorizing them into five status-based groups—**Active**, **Archived**, **Template**, **Done**, and **Work-in-Progress (WIP)**—then consolidating each group into a single JSON file and uploading all five files into a date-stamped Google Drive folder. It is designed for users managing large n8n instances who need organized, human-readable, off-platform backups.

The logical flow is organized into the following blocks:

- **1.1 Trigger & Global Actions** — Schedule-based (weekly Friday 2 PM) or manual on-demand start.
- **1.2 Configuration** — Sets the Google Drive parent folder ID and the n8n instance base URL.
- **1.3 Initial Steps** — Creates a date-stamped backup folder in Google Drive, fetches all workflows via the n8n REST API, and splits the result into individual items.
- **1.4 Categorical Filters** — A cascade of IF nodes that routes each workflow into exactly one of five categories based on a priority hierarchy: Active → Archived → Template → Done → WIP.
- **1.5 Active Branch** — Sort, convert to JSON, upload, wait.
- **1.6 Archived Branch** — Sort, convert to JSON, upload, wait.
- **1.7 Template Branch** — Sort, convert to JSON, upload, wait.
- **1.8 Done Branch** — Sort, convert to JSON, upload, wait.
- **1.9 Work-in-Progress Branch** — Sort, convert to JSON, upload, wait.
- **1.10 Completion Reporting** — All branches converge on a Time Saved analytics node.

---

### 2. Block-by-Block Analysis

#### Block 1.1 — Trigger & Global Actions

- **Overview:** Provides two entry points for the workflow. The Schedule trigger runs automatically every Friday at 14:00. The onCommand manual trigger is disabled by default but can be enabled for ad-hoc runs.
- **Nodes Involved:** Schedule, onCommand

| Node | Details |
|------|---------|
| **Schedule** | **Type:** `n8n-nodes-base.scheduleTrigger` (v1.3). Runs weekly on Friday at 14:00 (2 PM server time). **Input:** None (entry point). **Output:** Configuration. **Edge Cases:** If the n8n instance is offline at the scheduled time, the execution is skipped. Timezone depends on server configuration. |
| **onCommand** | **Type:** `n8n-nodes-base.manualTrigger` (v1). Disabled by default (`disabled: true`). Must be manually re-enabled and triggered from the n8n UI. **Input:** None (entry point). **Output:** Configuration. **Edge Cases:** Enabling this node does not disable the Schedule; both can run simultaneously. |

---

#### Block 1.2 — Configuration

- **Overview:** A Set node that defines two critical variables used by downstream nodes: the Google Drive parent folder ID and the n8n instance base URL. These values must be populated before the workflow can run.
- **Nodes Involved:** Configuration

| Node | Details |
|------|---------|
| **Configuration** | **Type:** `n8n-nodes-base.set` (v3.4). Defines two string assignments: `googleFolderId` (default placeholder: "GOOGLE DRIVE FOLDER ID GOES HERE") and `n8nUrl` (default placeholder: "N8N BASE URL GOES HERE"). **Key Expressions:** `$('Configuration').item.json.googleFolderId` and `$('Configuration').item.json.n8nUrl` are referenced by the Folder and Workflows nodes respectively. **Input:** Schedule or onCommand. **Output:** Folder. **Edge Cases:** If placeholders are not replaced, the Folder node will fail (invalid folder ID) and the Workflows node will call a malformed URL. The n8nUrl must NOT include a trailing slash or path segments after the domain/port. |

---

#### Block 1.3 — Initial Steps

- **Overview:** Creates a date-stamped subfolder inside the designated Google Drive parent folder, retrieves all workflows from the n8n API, and splits the API response array into individual items for per-workflow filtering.
- **Nodes Involved:** Folder, Workflows, Split Out

| Node | Details |
|------|---------|
| **Folder** | **Type:** `n8n-nodes-base.googleDrive` (v3), operation: create folder. **Folder name:** `n8n-Workflow-Backups_{{ $today.toISO().split('T')[0] }}` (e.g., `n8n-Workflow-Backups_2026-04-14`). **Parent folder ID:** resolved from `$('Configuration').item.json.googleFolderId`. **Folder color:** `#38D0A3` (teal). **Credential:** Google Drive OAuth2. **Input:** Configuration. **Output:** Workflows. **Edge Cases:** If the parent folder ID is invalid or inaccessible, the node returns a 403/404 error. The created folder's `id` is referenced by all upload nodes via `$('Folder').item.json.id`. |
| **Workflows** | **Type:** `n8n-nodes-base.httpRequest` (v4.4). Sends a GET request to `{{$('Configuration').item.json.n8nUrl}}/api/v1/workflows`. **Authentication:** Generic Credential Type → Header Auth (Method 2, best practice). The Header Auth credential must contain a header named `X-N8N-API-KEY` with the n8n API key as its value. **Response mode:** Full response (includes status, headers, and body). **Input:** Folder. **Output:** Split Out. **Edge Cases:** If the API key is missing or invalid, the request returns 401/403. If the n8nUrl is malformed, the request fails with a connection error. n8n Cloud trial accounts cannot generate API keys. If using Method 1 (direct header insertion), the key won't appear in credential audits. |
| **Split Out** | **Type:** `n8n-nodes-base.splitOut` (v1). Splits the field `body.data` (the array of workflow objects returned by the API) into separate items, one per workflow. **Input:** Workflows. **Output:** isActive. **Edge Cases:** If the API returns an empty `data` array (no workflows), the node outputs zero items and the rest of the workflow is skipped. If the API response structure changes (e.g., field name changes in a future n8n version), the split field path must be updated. |

---

#### Block 1.4 — Categorical Filters

- **Overview:** A cascading chain of four IF nodes that classifies each workflow into exactly one category. The priority order is: Active (takes precedence over everything) → Archived → Template → Done → WIP (default fallback). Each IF node's True branch routes to its respective processing pipeline; the False branch passes to the next IF node in the chain.
- **Nodes Involved:** isActive, isNotArchived, isTemplate, isDone

| Node | Details |
|------|---------|
| **isActive** | **Type:** `n8n-nodes-base.if` (v2.3). Condition: `$json.active` is boolean `true` (loose type validation). **True output →** sortActive. **False output →** isNotArchived. **Key Expression:** `{{ $json.active }}`. **Edge Cases:** A workflow that is both active AND tagged "Template" or "Done" will be classified as Active (priority rule). If the `active` field is absent or null, loose validation may still evaluate to false. |
| **isNotArchived** | **Type:** `n8n-nodes-base.if` (v2.3). Condition: `$json.isArchived` is boolean `false` (strict type validation). **True output →** isTemplate (workflow is NOT archived, continue checking tags). **False output →** sortArchived (workflow IS archived). **Key Expression:** `{{ $json.isArchived }}`. **Edge Cases:** Strict type validation means the field must be an actual boolean `false`, not a falsy string or null. If `isArchived` is null/undefined, the condition fails and the workflow falls to the Archived branch incorrectly. |
| **isTemplate** | **Type:** `n8n-nodes-base.if` (v2.3). Condition: `$json.tags` contains the string `"Template"` (loose type validation, string contains operation). **True output →** sortTemplates. **False output →** isDone. **Key Expression:** `{{ $json.tags }}`. **Edge Cases:** The `tags` field is likely a string or array. If it's an array serialized as a string, "contains" may match substring false positives (e.g., a tag named "NotTemplate"). Tag matching is case-sensitive. |
| **isDone** | **Type:** `n8n-nodes-base.if` (v2.3). Condition: `$('isTemplate').item.json.tags` contains the string `"Done"` (loose type validation, string contains operation). **True output →** sortDone. **False output →** sortInProgress. **Key Expression:** `{{ $('isTemplate').item.json.tags }}` — notably references the isTemplate node's output rather than `$json`. This is functionally equivalent since data flows from isTemplate's false branch, but is an explicit cross-node reference. **Edge Cases:** Same case-sensitivity and substring concerns as isTemplate. If the isTemplate node's output is empty or missing, this reference could fail. |

---

#### Block 1.5 — Active Branch

- **Overview:** Processes all workflows that have `active: true`. Sorts them alphabetically by name, consolidates them into a single JSON file, uploads it to the date-stamped Google Drive folder, and waits to avoid Google's upload rate-limiting.
- **Nodes Involved:** sortActive, active, uploadActive, Wait

| Node | Details |
|------|---------|
| **sortActive** | **Type:** `n8n-nodes-base.sort` (v1). Sorts by the field `name` in ascending order. **Input:** isActive (True). **Output:** active. |
| **active** | **Type:** `n8n-nodes-base.convertToFile` (v1.1). Operation: `toJson`. Converts all incoming items into a single JSON binary. **Binary property name:** `files`. **File name:** `active-backups_{{ $today.toISO().split('T')[0] }}.json` (e.g., `active-backups_2026-04-14.json`). **Input:** sortActive. **Output:** uploadActive. **Edge Cases:** If no active workflows exist, the node produces an empty JSON array file. |
| **uploadActive** | **Type:** `n8n-nodes-base.googleDrive` (v3), operation: upload file. **File name:** derived from binary `{{$binary.files.fileName}}`. **Upload to folder:** `$('Folder').item.json.id`. **Input data field:** `files`. **Credential:** Google Drive OAuth2. **Input:** active. **Output:** Wait. **Edge Cases:** If the Folder node failed earlier, the folder ID reference will be undefined and the upload fails with a 404. |
| **Wait** | **Type:** `n8n-nodes-base.wait` (v1.1). **Parameters:** empty `{}`. **Intended behavior:** Wait 5 seconds before proceeding to avoid Google's spam upload detector. **⚠ Configuration Note:** With empty parameters, the default resume type in n8n Wait v1.1 is `webhook`, which would pause execution indefinitely until an external signal. For the intended 5-second delay behavior, the node should be configured with `resume: "timeInterval"`, `amount: 5`, `unit: "seconds"`. This is a critical configuration gap in the current JSON. **Input:** uploadActive. **Output:** Time Saved. |

---

#### Block 1.6 — Archived Branch

- **Overview:** Processes all workflows that are NOT active but ARE archived (`isArchived: true`). Sorts, converts to JSON, uploads, and waits.
- **Nodes Involved:** sortArchived, archived, uploadArchived, Wait4

| Node | Details |
|------|---------|
| **sortArchived** | **Type:** `n8n-nodes-base.sort` (v1). Sorts by `name` ascending. **Input:** isNotArchived (False). **Output:** archived. |
| **archived** | **Type:** `n8n-nodes-base.convertToFile` (v1.1). Operation: `toJson`. Binary property: `files`. **File name:** `archived-backups_{{ $today.toISO().split('T')[0] }}.json`. **Input:** sortArchived. **Output:** uploadArchived. |
| **uploadArchived** | **Type:** `n8n-nodes-base.googleDrive` (v3), operation: upload file. File name from binary. Folder: `$('Folder').item.json.id`. Input field: `files`. **Credential:** Google Drive OAuth2. **Input:** archived. **Output:** Wait4. |
| **Wait4** | **Type:** `n8n-nodes-base.wait` (v1.1). Same empty-parameters situation as Wait. Intended 5-second delay. **Input:** uploadArchived. **Output:** Time Saved. |

---

#### Block 1.7 — Template Branch

- **Overview:** Processes non-active, non-archived workflows tagged "Template". Sorts, converts to JSON, uploads, and waits.
- **Nodes Involved:** sortTemplates, templates, uploadTemplates, Wait1

| Node | Details |
|------|---------|
| **sortTemplates** | **Type:** `n8n-nodes-base.sort` (v1). Sorts by `name` ascending. **Input:** isTemplate (True). **Output:** templates. |
| **templates** | **Type:** `n8n-nodes-base.convertToFile` (v1.1). Operation: `toJson`. Binary property: `files`. **File name:** `template-backups_{{ $today.toISO().split('T')[0] }}.json`. **Input:** sortTemplates. **Output:** uploadTemplates. |
| **uploadTemplates** | **Type:** `n8n-nodes-base.googleDrive` (v3), operation: upload file. File name from binary. Folder: `$('Folder').item.json.id`. Input field: `files`. **Credential:** Google Drive OAuth2. **Input:** templates. **Output:** Wait1. |
| **Wait1** | **Type:** `n8n-nodes-base.wait` (v1.1). Same empty-parameters situation. Intended 5-second delay. **Input:** uploadTemplates. **Output:** Time Saved. |

---

#### Block 1.8 — Done Branch

- **Overview:** Processes non-active, non-archived, non-Template workflows tagged "Done". Sorts, converts to JSON, uploads, and waits.
- **Nodes Involved:** sortDone, done, uploadDone, Wait2

| Node | Details |
|------|---------|
| **sortDone** | **Type:** `n8n-nodes-base.sort` (v1). Sorts by `name` ascending. **Input:** isDone (True). **Output:** done. |
| **done** | **Type:** `n8n-nodes-base.convertToFile` (v1.1). Operation: `toJson`. Binary property: `files`. **File name:** `done-backups_{{ $today.toISO().split('T')[0] }}.json`. **Input:** sortDone. **Output:** uploadDone. |
| **uploadDone** | **Type:** `n8n-nodes-base.googleDrive` (v3), operation: upload file. File name from binary. Folder: `$('Folder').item.json.id`. Input field: `files`. **Credential:** Google Drive OAuth2. **Input:** done. **Output:** Wait2. |
| **Wait2** | **Type:** `n8n-nodes-base.wait` (v1.1). Same empty-parameters situation. Intended 5-second delay. **Input:** uploadDone. **Output:** Time Saved. |

---

#### Block 1.9 — Work-in-Progress Branch

- **Overview:** The default fallback category. Processes all workflows that are not Active, not Archived, not tagged Template, and not tagged Done. Sorts, converts to JSON, uploads, and waits.
- **Nodes Involved:** sortInProgress, work-in-progress, uploadInProgress, Wait3

| Node | Details |
|------|---------|
| **sortInProgress** | **Type:** `n8n-nodes-base.sort` (v1). Sorts by `name` ascending. **Input:** isDone (False). **Output:** work-in-progress. |
| **work-in-progress** | **Type:** `n8n-nodes-base.convertToFile` (v1.1). Operation: `toJson`. Binary property: `files`. **File name:** `wip-backups_{{ $today.toISO().split('T')[0] }}.json`. **Input:** sortInProgress. **Output:** uploadInProgress. |
| **uploadInProgress** | **Type:** `n8n-nodes-base.googleDrive` (v3), operation: upload file. File name from binary. Folder: `$('Folder').item.json.id`. Input field: `files`. **Credential:** Google Drive OAuth2. **Input:** work-in-progress. **Output:** Wait3. |
| **Wait3** | **Type:** `n8n-nodes-base.wait` (v1.1). Same empty-parameters situation. Intended 5-second delay. **Input:** uploadInProgress. **Output:** Time Saved. |

---

#### Block 1.10 — Completion Reporting

- **Overview:** All five Wait nodes converge into a single Time Saved analytics node that reports estimated time saved per execution.
- **Nodes Involved:** Time Saved

| Node | Details |
|------|---------|
| **Time Saved** | **Type:** `n8n-nodes-base.timeSaved` (v1). Reports 10 minutes saved per execution. **Input:** Wait, Wait1, Wait2, Wait3, Wait4. **Output:** None (terminal). **Edge Cases:** This is a reporting/analytics node; it does not affect workflow data flow. The 10-minute estimate is static and may not reflect actual time saved. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|-----------------|-------------|
| Schedule | scheduleTrigger | Weekly Friday 2 PM trigger | — | Configuration | Global Actions — Start the workflow via Schedule or onCommand |
| onCommand | manualTrigger | Manual on-demand trigger (disabled) | — | Configuration | Global Actions — Start the workflow via Schedule or onCommand |
| Configuration | set | Sets googleFolderId and n8nUrl variables | Schedule, onCommand | Folder | Configuration — This node sets the Google Drive folder ID and the N8N base URL for the Create Folder and Workflows nodes. ; Edit googleFolderId and n8nUrl |
| Folder | googleDrive | Creates date-stamped backup folder in Google Drive | Configuration | Workflows | Select or create your Google Drive credential, and select your folder color ; Initial Steps — Create folder, Collect workflows, Split out workflows |
| Workflows | httpRequest | Fetches all workflows from n8n REST API | Folder | Split Out | Select Method 1 or Method 2 in the API Key Instructions and modify this node accordingly ; Initial Steps — Create folder, Collect workflows, Split out workflows |
| Split Out | splitOut | Splits API response array into individual workflow items | Workflows | isActive | Initial Steps — Create folder, Collect workflows, Split out workflows |
| isActive | if | Filters workflows where active = true | Split Out | sortActive (True), isNotArchived (False) | Filters |
| isNotArchived | if | Filters workflows where isArchived = false (not archived) | isActive (False) | isTemplate (True), sortArchived (False) | Filters |
| isTemplate | if | Filters workflows where tags contain "Template" | isNotArchived (True) | sortTemplates (True), isDone (False) | Filters |
| isDone | if | Filters workflows where tags contain "Done" | isTemplate (False) | sortDone (True), sortInProgress (False) | Filters |
| sortActive | sort | Sorts active workflows alphabetically by name | isActive (True) | active | Active — Sort, Convert, Upload |
| active | convertToFile | Converts active workflows to consolidated JSON file | sortActive | uploadActive | Active — Sort, Convert, Upload |
| uploadActive | googleDrive | Uploads active JSON file to Google Drive | active | Wait | Active — Sort, Convert, Upload |
| Wait | wait | 5-second pause to avoid Google upload rate limiting | uploadActive | Time Saved | |
| sortArchived | sort | Sorts archived workflows alphabetically by name | isNotArchived (False) | archived | Archived — Sort, Convert, Upload |
| archived | convertToFile | Converts archived workflows to consolidated JSON file | sortArchived | uploadArchived | Archived — Sort, Convert, Upload |
| uploadArchived | googleDrive | Uploads archived JSON file to Google Drive | archived | Wait4 | Archived — Sort, Convert, Upload |
| Wait4 | wait | 5-second pause to avoid Google upload rate limiting | uploadArchived | Time Saved | |
| sortTemplates | sort | Sorts template workflows alphabetically by name | isTemplate (True) | templates | Templates — Sort, Convert, Upload |
| templates | convertToFile | Converts template workflows to consolidated JSON file | sortTemplates | uploadTemplates | Templates — Sort, Convert, Upload |
| uploadTemplates | googleDrive | Uploads template JSON file to Google Drive | templates | Wait1 | Templates — Sort, Convert, Upload |
| Wait1 | wait | 5-second pause to avoid Google upload rate limiting | uploadTemplates | Time Saved | |
| sortDone | sort | Sorts done workflows alphabetically by name | isDone (True) | done | Done — Sort, Convert, Upload |
| done | convertToFile | Converts done workflows to consolidated JSON file | sortDone | uploadDone | Done — Sort, Convert, Upload |
| uploadDone | googleDrive | Uploads done JSON file to Google Drive | done | Wait2 | Done — Sort, Convert, Upload |
| Wait2 | wait | 5-second pause to avoid Google upload rate limiting | uploadDone | Time Saved | |
| sortInProgress | sort | Sorts WIP workflows alphabetically by name | isDone (False) | work-in-progress | Work-in-progress — Sort, Convert, Upload |
| work-in-progress | convertToFile | Converts WIP workflows to consolidated JSON file | sortInProgress | uploadInProgress | Work-in-progress — Sort, Convert, Upload |
| uploadInProgress | googleDrive | Uploads WIP JSON file to Google Drive | work-in-progress | Wait3 | Work-in-progress — Sort, Convert, Upload |
| Wait3 | wait | 5-second pause to avoid Google upload rate limiting | uploadInProgress | Time Saved | |
| Time Saved | timeSaved | Reports estimated time saved (10 minutes) | Wait, Wait1, Wait2, Wait3, Wait4 | — | |
| Sticky Note14 | stickyNote | Main workflow introduction, usage guide, and process breakdown | — | — | (Authoring/content note — not tied to operational nodes) |
| Sticky Note15 | stickyNote | API key method selection reminder | — | — | Select Method 1 or Method 2 in the API Key Instructions and modify this node accordingly |
| Sticky Note16 | stickyNote | Configuration editing reminder | — | — | Edit googleFolderId and n8nUrl |
| Sticky Note17 | stickyNote | Configuration node explanation | — | — | Configuration — This node sets the Google Drive folder ID and the N8N base URL for the Create Folder and Workflows nodes. |
| Sticky Note18 | stickyNote | Global trigger actions | — | — | Global Actions — Start the workflow via Schedule or onCommand |
| Sticky Note19 | stickyNote | Active branch label | — | — | Active — Sort, Convert, Upload |
| Sticky Note20 | stickyNote | Templates branch label | — | — | Templates — Sort, Convert, Upload |
| Sticky Note21 | stickyNote | Done branch label | — | — | Done — Sort, Convert, Upload |
| Sticky Note22 | stickyNote | WIP branch label | — | — | Work-in-progress — Sort, Convert, Upload |
| Sticky Note23 | stickyNote | Archived branch label | — | — | Archived — Sort, Convert, Upload |
| Sticky Note24 | stickyNote | Initial steps label | — | — | Initial Steps — Create folder, Collect workflows, Split out workflows |
| Sticky Note25 | stickyNote | Filters label | — | — | Filters |
| Sticky Note26 | stickyNote | Limitations documentation | — | — | LIMITATIONS: 1. N8N Cloud users on a trial account cannot generate or use the N8N API key. 2. If you use Method 1 (Direct Insertion) in the API Key Instructions below your API Key will not be listed as connected to this workflow if you use my [Workflow Credentials Auditor](https://n8n.io/workflows/14423). Same goes for all other credentials directly inserted into a HTTP Request node. |
| Sticky Note27 | stickyNote | Google Drive credential and folder color reminder | — | — | Select or create your Google Drive credential, and select your folder color |
| Sticky Note (ba30e335) | stickyNote | Alternative versions and custom builds | — | — | ALTERNATIVE VERSIONS AND CUSTOM BUILDS — While this version focuses on global status tagging using Google Drive, I have developed a parallel version for Github Repository storage. Need the Github Version? Free upon request. Preview: https://github.com/zianamitchell/n8n-workflow-images/blob/main/n8n-workflow-organizer-consolidated-github.png?raw=1 . Want to backup workflows without consolidating them? Standard Workflow Organizer free upon request for both Google Drive and GitHub. Note: The Google Drive version is meant to work in tandem with this Workflow Organizer Consolidated Google Drive. They should be linked to the same parent folder. Need a Custom build? Custom automation services available. A custom version of this workflow or the Github version is $15. Contact: workflows@zmglobalit.com |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it `Workflow Organizer Consolidated Google Drive - Template`.

2. **Add a Schedule Trigger node.**
   - Type: `Schedule Trigger`
   - Configuration: Weekly on Friday at 14:00 (2 PM)
   - This is the primary entry point.

3. **Add a Manual Trigger node** (optional, for on-demand runs).
   - Type: `Manual Trigger`
   - Name it `onCommand`
   - Disable it by default (click the node → toggle "Disabled")
   - Connect its output to the Configuration node (same as Schedule).

4. **Add a Set node** named `Configuration`.
   - Type: `Set` (v3.4)
   - Add two string assignments:
     - `googleFolderId` → Replace the placeholder with your Google Drive parent folder ID.
     - `n8nUrl` → Replace the placeholder with your n8n base URL (e.g., `https://subdomain.app.n8n.cloud`, `https://n8n.domain.com`, `http://localhost:5678`). Do NOT include trailing slashes or paths.
   - Connect Schedule output → Configuration input.
   - Connect onCommand output → Configuration input.

5. **Create a Google Drive OAuth2 credential.**
   - Go to Credentials → New Credential → Google Drive OAuth2 API.
   - Complete the OAuth2 consent flow and authorize access.

6. **Add a Google Drive node** named `Folder`.
   - Type: `Google Drive` (v3)
   - Resource: `folder`
   - Operation: create
   - Folder name: `=n8n-Workflow-Backups_{{ $today.toISO().split('T')[0] }}`
   - Drive: `My Drive` (select from list)
   - Parent folder ID: `={{ $('Configuration').item.json.googleFolderId }}` (mode: ID)
   - Folder color: `#38D0A3`
   - Credential: Select the Google Drive OAuth2 credential created in step 5.
   - Connect Configuration output → Folder input.

7. **Create a Header Auth credential for the n8n API.**
   - Go to Credentials → New Credential → Header Auth.
   - Name: `X-N8N-API-KEY`
   - Value: Your n8n API key (obtained from Settings → API → Create API Key)
   - Save securely; the key is not visible after creation.

8. **Add an HTTP Request node** named `Workflows`.
   - Type: `HTTP Request` (v4.4)
   - Method: GET (default)
   - URL: `={{$('Configuration').item.json.n8nUrl}}/api/v1/workflows`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `Header Auth`
   - Header Auth: Select the credential created in step 7.
   - Options → Response → Full Response: enabled
   - Connect Folder output → Workflows input.

9. **Add a Split Out node** named `Split Out`.
   - Type: `Split Out` (v1)
   - Field to split out: `body.data`
   - Connect Workflows output → Split Out input.

10. **Add an IF node** named `isActive`.
    - Type: `IF` (v2.3)
    - Condition combinator: AND
    - Condition: `$json.active` → operator: boolean is true
    - Type validation: loose
    - True output → will connect to sortActive
    - False output → will connect to isNotArchived
    - Connect Split Out output → isActive input.

11. **Add an IF node** named `isNotArchived`.
    - Type: `IF` (v2.3)
    - Condition combinator: AND
    - Condition: `$json.isArchived` → operator: boolean is false
    - Type validation: strict
    - True output → will connect to isTemplate
    - False output → will connect to sortArchived
    - Connect isActive (False) → isNotArchived input.

12. **Add an IF node** named `isTemplate`.
    - Type: `IF` (v2.3)
    - Condition combinator: AND
    - Condition: `$json.tags` → operator: string contains → value: `Template`
    - Type validation: loose
    - True output → will connect to sortTemplates
    - False output → will connect to isDone
    - Connect isNotArchived (True) → isTemplate input.

13. **Add an IF node** named `isDone`.
    - Type: `IF` (v2.3)
    - Condition combinator: AND
    - Condition: `$('isTemplate').item.json.tags` → operator: string contains → value: `Done`
    - Type validation: loose
    - True output → will connect to sortDone
    - False output → will connect to sortInProgress
    - Connect isTemplate (False) → isDone input.

14. **Add a Sort node** named `sortActive`.
    - Type: `Sort` (v1)
    - Sort field: `name` (ascending)
    - Connect isActive (True) → sortActive input.

15. **Add a Convert to File node** named `active`.
    - Type: `Convert to File` (v1.1)
    - Operation: `toJson`
    - Binary property name: `files`
    - File name: `=active-backups_{{ $today.toISO().split('T')[0] }}.json`
    - Connect sortActive output → active input.

16. **Add a Google Drive node** named `uploadActive`.
    - Type: `Google Drive` (v3)
    - Operation: upload file
    - File name: `={{$binary.files.fileName}}`
    - Folder ID: `={{ $('Folder').item.json.id }}` (mode: ID)
    - Drive: `My Drive`
    - Input data field name: `files`
    - Credential: Google Drive OAuth2
    - Connect active output → uploadActive input.

17. **Add a Wait node** named `Wait`.
    - Type: `Wait` (v1.1)
    - **Critical:** Set Resume to `Time Interval`, Amount: `5`, Unit: `Seconds`. (The default webhook resume will cause the execution to hang indefinitely.)
    - Connect uploadActive output → Wait input.

18. **Add a Sort node** named `sortArchived`.
    - Type: `Sort` (v1)
    - Sort field: `name` (ascending)
    - Connect isNotArchived (False) → sortArchived input.

19. **Add a Convert to File node** named `archived`.
    - Type: `Convert to File` (v1.1)
    - Operation: `toJson`
    - Binary property name: `files`
    - File name: `=archived-backups_{{ $today.toISO().split('T')[0] }}.json`
    - Connect sortArchived output → archived input.

20. **Add a Google Drive node** named `uploadArchived`.
    - Same configuration pattern as uploadActive (step 16).
    - Credential: Google Drive OAuth2
    - Connect archived output → uploadArchived input.

21. **Add a Wait node** named `Wait4`.
    - Same configuration as Wait (step 17): Resume = Time Interval, 5 seconds.
    - Connect uploadArchived output → Wait4 input.

22. **Add a Sort node** named `sortTemplates`.
    - Type: `Sort` (v1)
    - Sort field: `name` (ascending)
    - Connect isTemplate (True) → sortTemplates input.

23. **Add a Convert to File node** named `templates`.
    - Type: `Convert to File` (v1.1)
    - Operation: `toJson`
    - Binary property name: `files`
    - File name: `=template-backups_{{ $today.toISO().split('T')[0] }}.json`
    - Connect sortTemplates output → templates input.

24. **Add a Google Drive node** named `uploadTemplates`.
    - Same configuration pattern as uploadActive.
    - Credential: Google Drive OAuth2
    - Connect templates output → uploadTemplates input.

25. **Add a Wait node** named `Wait1`.
    - Same configuration as Wait: Resume = Time Interval, 5 seconds.
    - Connect uploadTemplates output → Wait1 input.

26. **Add a Sort node** named `sortDone`.
    - Type: `Sort` (v1)
    - Sort field: `name` (ascending)
    - Connect isDone (True) → sortDone input.

27. **Add a Convert to File node** named `done`.
    - Type: `Convert to File` (v1.1)
    - Operation: `toJson`
    - Binary property name: `files`
    - File name: `=done-backups_{{ $today.toISO().split('T')[0] }}.json`
    - Connect sortDone output → done input.

28. **Add a Google Drive node** named `uploadDone`.
    - Same configuration pattern as uploadActive.
    - Credential: Google Drive OAuth2
    - Connect done output → uploadDone input.

29. **Add a Wait node** named `Wait2`.
    - Same configuration as Wait: Resume = Time Interval, 5 seconds.
    - Connect uploadDone output → Wait2 input.

30. **Add a Sort node** named `sortInProgress`.
    - Type: `Sort` (v1)
    - Sort field: `name` (ascending)
    - Connect isDone (False) → sortInProgress input.

31. **Add a Convert to File node** named `work-in-progress`.
    - Type: `Convert to File` (v1.1)
    - Operation: `toJson`
    - Binary property name: `files`
    - File name: `=wip-backups_{{ $today.toISO().split('T')[0] }}.json`
    - Connect sortInProgress output → work-in-progress input.

32. **Add a Google Drive node** named `uploadInProgress`.
    - Same configuration pattern as uploadActive.
    - Credential: Google Drive OAuth2
    - Connect work-in-progress output → uploadInProgress input.

33. **Add a Wait node** named `Wait3`.
    - Same configuration as Wait: Resume = Time Interval, 5 seconds.
    - Connect uploadInProgress output → Wait3 input.

34. **Add a Time Saved node** named `Time Saved`.
    - Type: `Time Saved` (v1)
    - Minutes saved: `10`
    - Connect all five Wait nodes (Wait, Wait1, Wait2, Wait3, Wait4) outputs → Time Saved input.

35. **Verify all connections:**
    - Schedule → Configuration → Folder → Workflows → Split Out → isActive
    - isActive (True) → sortActive → active → uploadActive → Wait → Time Saved
    - isActive (False) → isNotArchived
    - isNotArchived (True) → isTemplate
    - isNotArchived (False) → sortArchived → archived → uploadArchived → Wait4 → Time Saved
    - isTemplate (True) → sortTemplates → templates → uploadTemplates → Wait1 → Time Saved
    - isTemplate (False) → isDone
    - isDone (True) → sortDone → done → uploadDone → Wait2 → Time Saved
    - isDone (False) → sortInProgress → work-in-progress → uploadInProgress → Wait3 → Time Saved

36. **Activate the workflow** by toggling it to Active in the n8n UI.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| n8n Cloud trial accounts cannot generate or use the n8n API key. This workflow will not function on trial instances. | Limitation — applies to all n8n Cloud trial users |
| If using Method 1 (direct API key insertion in the HTTP Request header), the API key will not appear as a connected credential in credential auditing tools like the Workflow Credentials Auditor. | https://n8n.io/workflows/14423 |
| The isDone node references `$('isTemplate').item.json.tags` instead of `$json.tags`. This is functionally equivalent but uses an explicit cross-node reference. If the isTemplate node's output structure changes, this expression must be updated. | Configuration note for the isDone filter node |
| All five Wait nodes (Wait, Wait1, Wait2, Wait3, Wait4) have empty parameters in the export JSON, which defaults to webhook-based resume. For the intended 5-second delay behavior, each must be reconfigured with Resume = Time Interval, Amount = 5, Unit = Seconds. | Critical configuration correction |
| The classification priority is strict: Active workflows are captured first regardless of tags or archived status. An active workflow tagged "Template" or "Done" will appear in the Active backup file, not in Template or Done. | Logical priority rule |
| Tag matching in isTemplate and isDone uses "string contains" with loose type validation and is case-sensitive. A tag named "template" (lowercase) will not match "Template". | Tag naming convention |
| The Google Drive folder color is set to `#38D0A3` (teal). This can be changed in the Folder node's options. | Visual customization |
| An alternative GitHub Repository storage version of this workflow is available free upon request from the author. | https://github.com/zianamitchell/n8n-workflow-images/blob/main/n8n-workflow-organizer-consolidated-github.png?raw=1 |
| A non-consolidated Workflow Organizer (individual files per workflow) is also available for both Google Drive and GitHub, designed to work in tandem with this consolidated version using the same parent folder. | Author offering |
| Custom builds with advanced logic (client-specific routing, multi-tier tagging, unique project tagging, integration with other platforms) are available for $15. | Contact: workflows@zmglobalit.com |
| The workflow tags on the template itself are: Standard-Template, v1, ZMGIT. | Workflow metadata |
| The Schedule trigger defaults to weekly on Friday at 14:00. Adjust the interval, day, and hour to match your backup cadence requirements. | Scheduling customization |