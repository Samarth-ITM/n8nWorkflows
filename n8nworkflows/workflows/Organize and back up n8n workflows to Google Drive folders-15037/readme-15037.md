Organize and back up n8n workflows to Google Drive folders

https://n8nworkflows.xyz/workflows/organize-and-back-up-n8n-workflows-to-google-drive-folders-15037


# Organize and back up n8n workflows to Google Drive folders

Now I have all the information I need to write the comprehensive document. Let me structure it properly. 1. Workflow Overview

This workflow automatically organizes and backs up every n8n workflow from an instance into categorized JSON files on Google Drive. It connects to the n8n public API, retrieves all workflows, and classifies each one into one of five categories—**Active**, **Template**, **Done**, **Archived**, or **Work-in-Progress**—based on the workflow's native `active`/`isArchived` flags and custom tags ("Template", "Done"). For each category it creates a dated sub-folder under a parent Google Drive folder, converts each workflow to an individual JSON file, and uploads the files in batches of five with a pause between batches to avoid Google Drive rate-limiting.

The logical blocks are:

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Entry & Configuration** | Triggers (scheduled or manual), sets credentials and URLs, creates the top-level dated portfolio folder, fetches all workflows from the n8n API, and splits the result into individual items |
| 2 | **Filter Cascade** | A chain of IF nodes that classifies each workflow into one of the five categories |
| 3 | **Active Workflows Branch** | Sort, convert to JSON, batch-loop, create category folder, upload, and wait |
| 4 | **Archived Workflows Branch** | Same pattern for archived workflows |
| 5 | **Template Workflows Branch** | Same pattern for template-tagged workflows |
| 6 | **Done Workflows Branch** | Same pattern for done-tagged workflows |
| 7 | **Work-in-Progress Branch** | Same pattern for anything not matching the above categories |
| 8 | **Completion** | Collects all finished branches into a single Time Saved node |

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Entry & Configuration

**Overview:** This block provides two entry points (a weekly schedule and a manual trigger), sets the Google Drive parent folder ID and n8n base URL, creates a dated top-level folder on Google Drive, fetches all workflows from the n8n API, and splits the API response into individual workflow items for downstream processing.

**Nodes Involved:** Schedule, onCommand, Configuration, Create folder, Workflows, Split Out

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **Schedule** | Schedule Trigger (v1.3) | Automated weekly trigger | Recurs every week on Friday at 14:00 (2 PM) | Trigger day: `[5]` (Friday), hour: `14` | None (entry point) | Configuration | If the workflow is inactive the schedule will not fire |
| **onCommand** | Manual Trigger (v1) | Manual on-demand trigger | No parameters | — | None (entry point) | Configuration | No edge cases; user must click "Execute Workflow" |
| **Configuration** | Set / Edit Fields (v3.4) | Defines the Google Drive parent folder ID and n8n instance base URL | Two string assignments: `googleFolderId` = `1i5bJY9o8AMlkPQlE42wcEA1SJlT0EYbt`, `n8nUrl` = `https://n8n-api.zmglobalit.com` | `googleFolderId`, `n8nUrl` — referenced by downstream nodes via `$('Configuration').item.json.googleFolderId` and `$('Configuration').item.json.n8nUrl` | Schedule or onCommand | Create folder | Incorrect folder ID → folder creation fails; URL with trailing slash or path → API call malformed |
| **Create folder** | Google Drive (v3) | Creates the top-level dated portfolio folder on Google Drive | Resource: `folder`; Name: `=portfolio_{{ $now.toISO().split('T')[0] }}`; Parent folder ID: from Configuration; Folder color: `#38D0A3` (teal-green) | `$now.toISO().split('T')[0]` extracts the current date; `$('Configuration').item.json.googleFolderId` | Configuration | Workflows | Google Drive OAuth2 credential missing or expired → auth error; parent folder ID invalid → 404; folder already exists → Google Drive may create a duplicate |
| **Workflows** | HTTP Request (v4.4) | Calls the n8n public API to list all workflows | URL: `={{$('Configuration').item.json.n8nUrl}}/api/v1/workflows`; Authentication: Generic Credential Type → Header Auth; Full response enabled | `$('Configuration').item.json.n8nUrl`; Header Auth credential supplies `X-N8N-API-KEY` | Create folder | Split Out | API key missing/revoked → 401/403; wrong base URL → connection refused or 404; n8n Cloud trial account cannot generate API keys; rate limits on self-hosted instances |
| **Split Out** | Split Out (v1) | Splits the `body.data` array from the API response into individual items | Field to split out: `body.data` | `$json.body.data` | Workflows | isActive | Empty `data` array → no items produced; API response structure change → field not found |

---

#### Block 2 — Filter Cascade

**Overview:** A chain of IF (conditional) nodes that routes each workflow into its category. The order is: (1) is it Active? → Active branch; (2) is it Archived? → Archived branch; (3) is it tagged "Template"? → Template branch; (4) is it tagged "Done"? → Done branch; (5) anything else → Work-in-Progress. This block also includes the "isNotArchived" node that separates archived from non-archived inactive workflows.

**Nodes Involved:** isActive, isNotArchived, isTemplate, isDone

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output (True) | Output (False) | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------------|----------------|---------------------------|
| **isActive** | IF (v2.3) | Checks whether the workflow's `active` flag is `true` | Condition: boolean equals `true`; Left value: `={{ $json.active }}`; Type validation: `loose` | `$json.active` | Split Out | sortActive | isNotArchived | Workflows with no `active` field → evaluates as falsy; loose validation may treat string "true" as truthy |
| **isNotArchived** | IF (v2.3) | Checks whether the workflow's `isArchived` flag is `false` | Condition: boolean equals `false`; Left value: `={{ $json.isArchived }}`; Type validation: `strict` | `$json.isArchived` | isActive (false) | isTemplate | sortArchived | Strict type validation means only actual `false` boolean passes; `null` or `undefined` will not match |
| **isTemplate** | IF (v2.3) | Checks whether the workflow's tags contain the string "Template" | Condition: string `contains`; Left value: `={{ $json.tags }}`; Right value: `Template`; Type validation: `loose` | `$json.tags` | isNotArchived (true) | sortTemplates | isDone | Tags field is an array of objects; loose string-contains on an array may not behave as expected in all n8n versions; tags with "template" (lowercase) won't match |
| **isDone** | IF (v2.3) | Checks whether the workflow's tags contain the string "Done" | Condition: string `contains`; Left value: `={{ $('isTemplate').item.json.tags }}`; Right value: `Done`; Type validation: `loose` | `$('isTemplate').item.json.tags` | isTemplate (false) | sortDone | sortInProgress | Same tag-matching concerns as isTemplate; a workflow tagged both "Template" and "Done" will be captured by the Template branch first (higher priority) |

---

#### Block 3 — Active Workflows Branch

**Overview:** Takes all workflows where `active = true`, sorts them alphabetically by name, converts each to a separate JSON file, then uploads them in batches of five to a dated `active-backups_YYYY-MM-DD` sub-folder on Google Drive. A Wait node pauses after each batch to avoid Google Drive rate-limiting.

**Nodes Involved:** sortActive, active, loopActive, createActive, uploadActive, Wait

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **sortActive** | Sort (v1) | Sorts active workflows alphabetically by name | Sort field: `name`, ascending | `$json.name` | isActive (true) | active | Empty list → no output |
| **active** | Convert to File (v1.1) | Converts each workflow item to a JSON binary file | Mode: `each`; Operation: `toJson`; Binary property: `files`; File name: `={{$json.name}}-backup_{{ $now.toISO().split('T')[0] }}.json` | `$json.name`, `$now.toISO().split('T')[0]` | sortActive | loopActive | Very large workflow JSON → binary property may exceed memory limits; special characters in workflow name → potentially invalid file name |
| **loopActive** | Split In Batches (v3) | Processes items in groups of 5 | Batch size: `5` | — | active (items) | Output 0 → createActive; Output 1 → Time Saved | If fewer than 5 items, first batch processes all and loop ends immediately |
| **createActive** | Google Drive (v3) | Creates the category sub-folder once | Resource: `folder`; Name: `=active-backups_{{ $today.toISO().split('T')[0] }}`; Parent: `$('Create folder').item.json.id`; Execute Once: `true` | `$today.toISO().split('T')[0]`, `$('Create folder').item.json.id` | loopActive (batch) | uploadActive | Parent folder ID from "Create folder" must be available; `executeOnce` ensures only one folder is created even though the node receives multiple items |
| **uploadActive** | Google Drive (v3) | Uploads the batch of JSON files to the created folder | Input data field: `files`; File name: `={{$binary.files.fileName}}`; Folder ID: `$('createActive').item.json.id` | `$binary.files.fileName`, `$('createActive').item.json.id` | createActive | Wait | Credential expired → 401; folder not found → 404; file name collision → overwrite behavior depends on Google Drive settings |
| **Wait** | Wait (v1.1) | Pauses execution to avoid triggering Google Drive's upload spam detector | Default parameters (intended ~5-second pause) | — | uploadActive | loopActive | If configured as webhook-based wait without a resume call, execution will hang indefinitely; ensure time-based wait is configured |

---

#### Block 4 — Archived Workflows Branch

**Overview:** Processes workflows where `isArchived = true`. Follows the same sort → convert → batch-loop → create folder → upload → wait pattern as the Active branch, but uploads to an `archive-backups_YYYY-MM-DD` sub-folder.

**Nodes Involved:** sortArchived, archived, loopArchived, createArchived, uploadArchived, Wait4

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **sortArchived** | Sort (v1) | Sorts archived workflows by name | Sort field: `name` | — | isNotArchived (false) | archived | Empty list → no output |
| **archived** | Convert to File (v1.1) | Converts each archived workflow to JSON | Mode: `each`; Operation: `toJson`; Binary property: `files`; File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json` | — | sortArchived | loopArchived | Same file-name concerns as active node |
| **loopArchived** | Split In Batches (v3) | Batches of 5 | Batch size: `5` | — | archived | Output 0 → createArchived; Output 1 → Time Saved | — |
| **createArchived** | Google Drive (v3) | Creates the archive sub-folder once | Resource: `folder`; Name: `=archive-backups_{{ $today.toISO().split('T')[0] }}`; Parent: `$('Create folder').item.json.id`; Execute Once: `true` | — | loopArchived (batch) | uploadArchived | — |
| **uploadArchived** | Google Drive (v3) | Uploads JSON files to the archive folder | Input data field: `files`; Folder ID: `$('createArchived').item.json.id` | — | createArchived | Wait4 | — |
| **Wait4** | Wait (v1.1) | Pause between batches | Default parameters | — | uploadArchived | loopArchived | Same wait configuration concern |

---

#### Block 5 — Template Workflows Branch

**Overview:** Processes workflows whose tags contain "Template". Creates a `template-backups_YYYY-MM-DD` sub-folder and uploads converted JSON files in batches of five.

**Nodes Involved:** sortTemplates, template, loopTemplates, createTemplate, uploadTemplates, Wait1

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **sortTemplates** | Sort (v1) | Sorts template workflows by name | Sort field: `name` | — | isTemplate (true) | template | — |
| **template** | Convert to File (v1.1) | Converts each template workflow to JSON | Mode: `each`; Operation: `toJson`; Binary property: `files`; File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json` | — | sortTemplates | loopTemplates | — |
| **loopTemplates** | Split In Batches (v3) | Batches of 5 | Batch size: `5` | — | template | Output 0 → createTemplate; Output 1 → Time Saved | — |
| **createTemplate** | Google Drive (v3) | Creates the template sub-folder once | Resource: `folder`; Name: `=template-backups_{{ $today.toISO().split('T')[0] }}`; Parent: `$('Create folder').item.json.id`; Execute Once: `true` | — | loopTemplates (batch) | uploadTemplates | — |
| **uploadTemplates** | Google Drive (v3) | Uploads JSON files to the template folder | Input data field: `files`; Folder ID: `$('createTemplate').item.json.id` | — | createTemplate | Wait1 | — |
| **Wait1** | Wait (v1.1) | Pause between batches | Default parameters | — | uploadTemplates | loopTemplates | — |

---

#### Block 6 — Done Workflows Branch

**Overview:** Processes workflows whose tags contain "Done" (but not "Template"). Creates a `done-backups_YYYY-MM-DD` sub-folder and uploads in batches.

**Nodes Involved:** sortDone, done, loopDone, createDone, uploadDone, Wait3

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **sortDone** | Sort (v1) | Sorts done workflows by name | Sort field: `name` | — | isDone (true) | done | — |
| **done** | Convert to File (v1.1) | Converts each done workflow to JSON | Mode: `each`; Operation: `toJson`; Binary property: `files`; File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json` | — | sortDone | loopDone | — |
| **loopDone** | Split In Batches (v3) | Batches of 5 | Batch size: `5` | — | done | Output 0 → createDone; Output 1 → Time Saved | — |
| **createDone** | Google Drive (v3) | Creates the done sub-folder once | Resource: `folder`; Name: `=done-backups_{{ $today.toISO().split('T')[0] }}`; Parent: `$('Create folder').item.json.id`; Execute Once: `true` | — | loopDone (batch) | uploadDone | — |
| **uploadDone** | Google Drive (v3) | Uploads JSON files to the done folder | Input data field: `files`; Folder ID: `$('createDone').item.json.id` | — | createDone | Wait3 | — |
| **Wait3** | Wait (v1.1) | Pause between batches | Default parameters | — | uploadDone | loopDone | — |

---

#### Block 7 — Work-in-Progress Branch

**Overview:** Processes all workflows that are not Active, not Archived, not tagged "Template", and not tagged "Done". These are assumed to be work-in-progress and are uploaded to a `wip_YYYY-MM-DD` sub-folder.

**Nodes Involved:** sortInProgress, work-in-progress, loopWIP, createWIP, uploadInProgress, Wait2

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **sortInProgress** | Sort (v1) | Sorts WIP workflows by name | Sort field: `name` | — | isDone (false) | work-in-progress | — |
| **work-in-progress** | Convert to File (v1.1) | Converts each WIP workflow to JSON | Mode: `each`; Operation: `toJson`; Binary property: `files`; File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json` | — | sortInProgress | loopWIP | — |
| **loopWIP** | Split In Batches (v3) | Batches of 5 | Batch size: `5` | — | work-in-progress | Output 0 → createWIP; Output 1 → Time Saved | — |
| **createWIP** | Google Drive (v3) | Creates the WIP sub-folder once | Resource: `folder`; Name: `=wip_{{ $today.toISO().split('T')[0] }}`; Parent: `$('Create folder').item.json.id`; Execute Once: `true` | — | loopWIP (batch) | uploadInProgress | — |
| **uploadInProgress** | Google Drive (v3) | Uploads JSON files to the WIP folder | Input data field: `files`; Folder ID: `$('createWIP').item.json.id` | — | createWIP | Wait2 | — |
| **Wait2** | Wait (v1.1) | Pause between batches | Default parameters | — | uploadInProgress | loopWIP | — |

---

#### Block 8 — Completion

**Overview:** All five category branches converge at a single Time Saved node that records the estimated time saved by running this automation.

**Nodes Involved:** Time Saved

| Node | Type | Technical Role | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Failure Types |
|------|------|---------------|---------------|-----------------------------|-------|--------|---------------------------|
| **Time Saved** | Time Saved (v1) | Records estimated time saved by the automation | Minutes saved: `10` | — | All loop-done outputs (loopActive, loopTemplates, loopDone, loopWIP, loopArchived) | None (terminal) | Not applicable; purely informational |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|----------------|-----------------|-------------|
| Schedule | Schedule Trigger (v1.3) | Automated weekly trigger (Friday 14:00) | — | Configuration | Global Actions — Start the workflows via Schedule or onCommand |
| onCommand | Manual Trigger (v1) | Manual on-demand execution trigger | — | Configuration | Global Actions — Start the workflows via Schedule or onCommand |
| Configuration | Set (v3.4) | Sets Google Drive parent folder ID and n8n base URL | Schedule, onCommand | Create folder | Configuration — This node sets the Google Drive folder ID and the N8N base URL for the Create Folder and Workflows nodes. |
| Create folder | Google Drive (v3) | Creates top-level dated portfolio folder | Configuration | Workflows | Initial Steps — Create folder, Collect workflows, Split out workflows; Select or create your Google Drive credential, and select your folder color |
| Workflows | HTTP Request (v4.4) | Fetches all workflows from n8n public API | Create folder | Split Out | Initial Steps — Create folder, Collect workflows, Split out workflows; Select Method 1 or Method 2 in the API Key Instructions and modify this node accordingly |
| Split Out | Split Out (v1) | Splits API response array into individual items | Workflows | isActive | Initial Steps — Create folder, Collect workflows, Split out workflows |
| isActive | IF (v2.3) | Checks if workflow is active | Split Out | sortActive (true), isNotArchived (false) | Filters |
| sortActive | Sort (v1) | Sorts active workflows by name | isActive (true) | active | Active — Sort, Convert, Create, Upload |
| active | Convert to File (v1.1) | Converts active workflows to JSON files | sortActive | loopActive | Active — Sort, Convert, Create, Upload |
| loopActive | Split In Batches (v3) | Processes active items in batches of 5 | active, Wait | createActive (batch), Time Saved (done) | Active — Sort, Convert, Create, Upload |
| createActive | Google Drive (v3) | Creates active-backups sub-folder (once) | loopActive (batch) | uploadActive | Active — Sort, Convert, Create, Upload |
| uploadActive | Google Drive (v3) | Uploads active workflow JSON files | createActive | Wait | Active — Sort, Convert, Create, Upload |
| Wait | Wait (v1.1) | Pause between upload batches | uploadActive | loopActive | Active — Sort, Convert, Create, Upload |
| isNotArchived | IF (v2.3) | Checks if workflow is not archived | isActive (false) | isTemplate (true), sortArchived (false) | Filters |
| sortArchived | Sort (v1) | Sorts archived workflows by name | isNotArchived (false) | archived | Archived — Sort, Convert, Create, Upload |
| archived | Convert to File (v1.1) | Converts archived workflows to JSON files | sortArchived | loopArchived | Archived — Sort, Convert, Create, Upload |
| loopArchived | Split In Batches (v3) | Processes archived items in batches of 5 | archived, Wait4 | createArchived (batch), Time Saved (done) | Archived — Sort, Convert, Create, Upload |
| createArchived | Google Drive (v3) | Creates archive-backups sub-folder (once) | loopArchived (batch) | uploadArchived | Archived — Sort, Convert, Create, Upload |
| uploadArchived | Google Drive (v3) | Uploads archived workflow JSON files | createArchived | Wait4 | Archived — Sort, Convert, Create, Upload |
| Wait4 | Wait (v1.1) | Pause between upload batches | uploadArchived | loopArchived | Archived — Sort, Convert, Create, Upload |
| isTemplate | IF (v2.3) | Checks if tags contain "Template" | isNotArchived (true) | sortTemplates (true), isDone (false) | Filters |
| sortTemplates | Sort (v1) | Sorts template workflows by name | isTemplate (true) | template | Templates — Sort, Convert, Create, Upload |
| template | Convert to File (v1.1) | Converts template workflows to JSON files | sortTemplates | loopTemplates | Templates — Sort, Convert, Create, Upload |
| loopTemplates | Split In Batches (v3) | Processes template items in batches of 5 | template, Wait1 | createTemplate (batch), Time Saved (done) | Templates — Sort, Convert, Create, Upload |
| createTemplate | Google Drive (v3) | Creates template-backups sub-folder (once) | loopTemplates (batch) | uploadTemplates | Templates — Sort, Convert, Create, Upload |
| uploadTemplates | Google Drive (v3) | Uploads template workflow JSON files | createTemplate | Wait1 | Templates — Sort, Convert, Create, Upload |
| Wait1 | Wait (v1.1) | Pause between upload batches | uploadTemplates | loopTemplates | Templates — Sort, Convert, Create, Upload |
| isDone | IF (v2.3) | Checks if tags contain "Done" | isTemplate (false) | sortDone (true), sortInProgress (false) | Filters |
| sortDone | Sort (v1) | Sorts done workflows by name | isDone (true) | done | Done — Sort, Convert, Create, Upload |
| done | Convert to File (v1.1) | Converts done workflows to JSON files | sortDone | loopDone | Done — Sort, Convert, Create, Upload |
| loopDone | Split In Batches (v3) | Processes done items in batches of 5 | done, Wait3 | createDone (batch), Time Saved (done) | Done — Sort, Convert, Create, Upload |
| createDone | Google Drive (v3) | Creates done-backups sub-folder (once) | loopDone (batch) | uploadDone | Done — Sort, Convert, Create, Upload |
| uploadDone | Google Drive (v3) | Uploads done workflow JSON files | createDone | Wait3 | Done — Sort, Convert, Create, Upload |
| Wait3 | Wait (v1.1) | Pause between upload batches | uploadDone | loopDone | Done — Sort, Convert, Create, Upload |
| sortInProgress | Sort (v1) | Sorts WIP workflows by name | isDone (false) | work-in-progress | Work-in-progress — Sort, Convert, Create, Upload |
| work-in-progress | Convert to File (v1.1) | Converts WIP workflows to JSON files | sortInProgress | loopWIP | Work-in-progress — Sort, Convert, Create, Upload |
| loopWIP | Split In Batches (v3) | Processes WIP items in batches of 5 | work-in-progress, Wait2 | createWIP (batch), Time Saved (done) | Work-in-progress — Sort, Convert, Create, Upload |
| createWIP | Google Drive (v3) | Creates wip sub-folder (once) | loopWIP (batch) | uploadInProgress | Work-in-progress — Sort, Convert, Create, Upload |
| uploadInProgress | Google Drive (v3) | Uploads WIP workflow JSON files | createWIP | Wait2 | Work-in-progress — Sort, Convert, Create, Upload |
| Wait2 | Wait (v1.1) | Pause between upload batches | uploadInProgress | loopWIP | Work-in-progress — Sort, Convert, Create, Upload |
| Time Saved | Time Saved (v1) | Records estimated time saved (10 minutes) | loopActive, loopTemplates, loopDone, loopWIP, loopArchived (all done outputs) | — | — |

---

### 4. Reproducing the Workflow from Scratch

Below is the complete step-by-step procedure to rebuild this workflow manually in n8n.

---

#### Step 1 — Create the Workflow

1. Create a new workflow and name it **"Workflow Organizer Google Drive - Template"**.

---

#### Step 2 — Add Entry Points

2. Add a **Schedule Trigger** node. Name it **Schedule**. Configure: interval field = `weeks`, trigger at day = `Friday` (index 5), trigger at hour = `14`.
3. Add a **Manual Trigger** node. Name it **onCommand**. No configuration needed.

---

#### Step 3 — Add the Configuration Node

4. Add an **Edit Fields (Set)** node (v3.4). Name it **Configuration**.
5. Add two string assignments:
   - `googleFolderId` → your Google Drive parent folder ID (e.g. `1i5bJY9o8AMlkPQlE42wcEA1SJlT0EYbt`)
   - `n8nUrl` → your n8n instance base URL (e.g. `https://n8n-api.zmglobalit.com`). Do **not** include a trailing slash or any path after the domain/port.
6. Connect **Schedule** → **Configuration** and **onCommand** → **Configuration**.

---

#### Step 4 — Configure Credentials

7. **Google Drive OAuth2 credential:**
   - Go to Credentials → Add Credential → **Google Drive OAuth2 API**.
   - Complete the OAuth2 flow with your Google account.
   - Name it **Google Drive account**.
8. **Header Auth credential (for n8n API key):**
   - Go to Credentials → Add Credential → **Header Auth**.
   - Name: `X-N8N-API-KEY`; Value: your n8n API key (obtained from Settings → API → Create API Key).
   - Name the credential **Header Auth account** (or similar).

---

#### Step 5 — Create the Top-Level Portfolio Folder

9. Add a **Google Drive** node (v3). Name it **Create folder**.
   - Resource: `folder`
   - Folder name: `=portfolio_{{ $now.toISO().split('T')[0] }}`
   - Parent folder ID: `={{ $('Configuration').item.json.googleFolderId }}`
   - Drive: `My Drive`
   - Folder color RGB: `#38D0A3`
   - Credential: **Google Drive account**
10. Connect **Configuration** → **Create folder**.

---

#### Step 6 — Fetch All Workflows

11. Add an **HTTP Request** node (v4.4). Name it **Workflows**.
    - Method: `GET`
    - URL: `={{$('Configuration').item.json.n8nUrl}}/api/v1/workflows`
    - Authentication: `Generic Credential Type`
    - Generic Auth Type: `Header Auth`
    - Credential: your Header Auth credential
    - Options → Response → Full Response: `true`
12. Connect **Create folder** → **Workflows**.

---

#### Step 7 — Split Out Individual Workflows

13. Add a **Split Out** node (v1). Name it **Split Out**.
    - Field to split out: `body.data`
14. Connect **Workflows** → **Split Out**.

---

#### Step 8 — Add Filter Cascade

15. Add an **IF** node (v2.3). Name it **isActive**.
    - Condition combinator: `and`
    - Condition: Boolean equals `true`; Left value: `={{ $json.active }}`; Type validation: `loose`
16. Connect **Split Out** → **isActive**.

17. Add an **IF** node (v2.3). Name it **isNotArchived**.
    - Condition combinator: `and`
    - Condition: Boolean equals `false`; Left value: `={{ $json.isArchived }}`; Type validation: `strict`
18. Connect **isActive** (false output) → **isNotArchived**.

19. Add an **IF** node (v2.3). Name it **isTemplate**.
    - Condition combinator: `and`
    - Condition: String `contains`; Left value: `={{ $json.tags }}`; Right value: `Template`; Type validation: `loose`
20. Connect **isNotArchived** (true output) → **isTemplate**.

21. Add an **IF** node (v2.3). Name it **isDone**.
    - Condition combinator: `and`
    - Condition: String `contains`; Left value: `={{ $('isTemplate').item.json.tags }}`; Right value: `Done`; Type validation: `loose`
22. Connect **isTemplate** (false output) → **isDone**.

---

#### Step 9 — Active Workflows Branch

23. Add a **Sort** node (v1). Name it **sortActive**. Sort field: `name`.
24. Connect **isActive** (true output) → **sortActive**.

25. Add a **Convert to File** node (v1.1). Name it **active**.
    - Mode: `each`
    - Operation: `toJson`
    - Binary property name: `files`
    - File name: `={{$json.name}}-backup_{{ $now.toISO().split('T')[0] }}.json`
26. Connect **sortActive** → **active**.

27. Add a **Split In Batches** node (v3). Name it **loopActive**. Batch size: `5`.
28. Connect **active** → **loopActive** (input).

29. Add a **Google Drive** node (v3). Name it **createActive**.
    - Resource: `folder`
    - Folder name: `=active-backups_{{ $today.toISO().split('T')[0] }}`
    - Parent folder ID: `={{ $('Create folder').item.json.id }}`
    - Drive: `My Drive`
    - Execute Once: `true`
    - Credential: **Google Drive account**
30. Connect **loopActive** (output 0 — batch) → **createActive**.

31. Add a **Google Drive** node (v3). Name it **uploadActive**.
    - Operation: upload (default)
    - Input data field name: `files`
    - File name: `={{$binary.files.fileName}}`
    - Folder ID: `={{ $('createActive').item.json.id }}`
    - Drive: `My Drive`
    - Credential: **Google Drive account**
32. Connect **createActive** → **uploadActive**.

33. Add a **Wait** node (v1.1). Name it **Wait**. Configure a time-based wait of approximately 5 seconds.
34. Connect **uploadActive** → **Wait**.
35. Connect **Wait** → **loopActive** (to process the next batch).

---

#### Step 10 — Archived Workflows Branch

36. Add a **Sort** node. Name it **sortArchived**. Sort field: `name`.
37. Connect **isNotArchived** (false output) → **sortArchived**.

38. Add a **Convert to File** node. Name it **archived**.
    - Mode: `each`; Operation: `toJson`; Binary property: `files`
    - File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json`
39. Connect **sortArchived** → **archived**.

40. Add a **Split In Batches** node. Name it **loopArchived**. Batch size: `5`.
41. Connect **archived** → **loopArchived**.

42. Add a **Google Drive** node. Name it **createArchived**.
    - Resource: `folder`
    - Folder name: `=archive-backups_{{ $today.toISO().split('T')[0] }}`
    - Parent folder ID: `={{ $('Create folder').item.json.id }}`
    - Drive: `My Drive`; Execute Once: `true`; Credential: **Google Drive account**
43. Connect **loopArchived** (output 0) → **createArchived**.

44. Add a **Google Drive** node. Name it **uploadArchived**.
    - Input data field: `files`; File name: `={{$binary.files.fileName}}`; Folder ID: `={{ $('createArchived').item.json.id }}`; Credential: **Google Drive account**
45. Connect **createArchived** → **uploadArchived**.

46. Add a **Wait** node. Name it **Wait4**. Configure ~5 seconds.
47. Connect **uploadArchived** → **Wait4** → **loopArchived**.

---

#### Step 11 — Template Workflows Branch

48. Add a **Sort** node. Name it **sortTemplates**. Sort field: `name`.
49. Connect **isTemplate** (true output) → **sortTemplates**.

50. Add a **Convert to File** node. Name it **template**.
    - Mode: `each`; Operation: `toJson`; Binary property: `files`
    - File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json`
51. Connect **sortTemplates** → **template**.

52. Add a **Split In Batches** node. Name it **loopTemplates**. Batch size: `5`.
53. Connect **template** → **loopTemplates**.

54. Add a **Google Drive** node. Name it **createTemplate**.
    - Resource: `folder`; Folder name: `=template-backups_{{ $today.toISO().split('T')[0] }}`; Parent folder ID: `={{ $('Create folder').item.json.id }}`; Execute Once: `true`; Credential: **Google Drive account**
55. Connect **loopTemplates** (output 0) → **createTemplate**.

56. Add a **Google Drive** node. Name it **uploadTemplates**.
    - Input data field: `files`; File name: `={{$binary.files.fileName}}`; Folder ID: `={{ $('createTemplate').item.json.id }}`; Credential: **Google Drive account**
57. Connect **createTemplate** → **uploadTemplates**.

58. Add a **Wait** node. Name it **Wait1**. Configure ~5 seconds.
59. Connect **uploadTemplates** → **Wait1** → **loopTemplates**.

---

#### Step 12 — Done Workflows Branch

60. Add a **Sort** node. Name it **sortDone**. Sort field: `name`.
61. Connect **isDone** (true output) → **sortDone**.

62. Add a **Convert to File** node. Name it **done**.
    - Mode: `each`; Operation: `toJson`; Binary property: `files`
    - File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json`
63. Connect **sortDone** → **done**.

64. Add a **Split In Batches** node. Name it **loopDone**. Batch size: `5`.
65. Connect **done** → **loopDone**.

66. Add a **Google Drive** node. Name it **createDone**.
    - Resource: `folder`; Folder name: `=done-backups_{{ $today.toISO().split('T')[0] }}`; Parent folder ID: `={{ $('Create folder').item.json.id }}`; Execute Once: `true`; Credential: **Google Drive account**
67. Connect **loopDone** (output 0) → **createDone**.

68. Add a **Google Drive** node. Name it **uploadDone**.
    - Input data field: `files`; File name: `={{$binary.files.fileName}}`; Folder ID: `={{ $('createDone').item.json.id }}`; Credential: **Google Drive account**
69. Connect **createDone** → **uploadDone**.

70. Add a **Wait** node. Name it **Wait3**. Configure ~5 seconds.
71. Connect **uploadDone** → **Wait3** → **loopDone**.

---

#### Step 13 — Work-in-Progress Branch

72. Add a **Sort** node. Name it **sortInProgress**. Sort field: `name`.
73. Connect **isDone** (false output) → **sortInProgress**.

74. Add a **Convert to File** node. Name it **work-in-progress**.
    - Mode: `each`; Operation: `toJson`; Binary property: `files`
    - File name: `={{$json.name}}-backup_{{ $today.toISO().split('T')[0] }}.json`
75. Connect **sortInProgress** → **work-in-progress**.

76. Add a **Split In Batches** node. Name it **loopWIP**. Batch size: `5`.
77. Connect **work-in-progress** → **loopWIP**.

78. Add a **Google Drive** node. Name it **createWIP**.
    - Resource: `folder`; Folder name: `=wip_{{ $today.toISO().split('T')[0] }}`; Parent folder ID: `={{ $('Create folder').item.json.id }}`; Execute Once: `true`; Credential: **Google Drive account**
79. Connect **loopWIP** (output 0) → **createWIP**.

80. Add a **Google Drive** node. Name it **uploadInProgress**.
    - Input data field: `files`; File name: `={{$binary.files.fileName}}`; Folder ID: `={{ $('createWIP').item.json.id }}`; Credential: **Google Drive account**
81. Connect **createWIP** → **uploadInProgress**.

82. Add a **Wait** node. Name it **Wait2**. Configure ~5 seconds.
83. Connect **uploadInProgress** → **Wait2** → **loopWIP**.

---

#### Step 14 — Add the Completion Node

84. Add a **Time Saved** node (v1). Name it **Time Saved**. Minutes saved: `10`.
85. Connect the **done** output (output 1) from each of the five Split In Batches nodes to **Time Saved**:
    - **loopActive** (output 1) → **Time Saved**
    - **loopTemplates** (output 1) → **Time Saved**
    - **loopDone** (output 1) → **Time Saved**
    - **loopWIP** (output 1) → **Time Saved**
    - **loopArchived** (output 1) → **Time Saved**

---

#### Step 15 — Workflow Settings

86. Set the workflow binary mode to **separate**.
87. Set execution order to **v1**.
88. Set the caller policy to **workflows from same owner** (if relevant to your instance).
89. Deactivate "Save manual executions" if desired.
90. Set "Save data success execution" to **none** to avoid storing large execution data.

---

#### Step 16 — Tag Your Workflows

91. Before running, ensure your n8n workflows are properly tagged:
    - Template workflows → add the tag **"Template"**
    - Completed project workflows → add the tag **"Done"**
    - Finished templates should **not** be tagged "Done" (they remain templates with placeholder text)
    - Any workflow that is not active, not archived, and lacks Template/Done tags will be classified as **Work-in-Progress** automatically

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| N8N Cloud users on a trial account cannot generate or use the N8N API key. | Limitation — affects the Workflows (HTTP Request) node |
| If you use Method 1 (Direct Insertion of the API key as a header in the HTTP Request node), the API key will not be listed as a connected credential by the Workflow Credentials Auditor. | https://n8n.io/workflows/14423 |
| The n8n API key can be obtained at Settings → API → Create API Key. Copy and save it securely immediately — it cannot be viewed again after creation. | API Key Instructions |
| Two authentication methods are supported for the Workflows node. Method 1 (Quick Start): set Authentication to None, add a custom header `X-N8N-API-KEY` with the key value. Method 2 (Best Practice, default): set Authentication to Generic Credential Type → Header Auth, and create or select a Header Auth credential with name `X-N8N-API-KEY` and value of your API key. | API Key Instructions |
| Do NOT include any path after the domain/port in the n8nUrl (e.g. no trailing slash, no `/api/v1`). The `/api/v1/workflows` path is appended automatically by the Workflows node expression. | Configuration node guidance |
| A pre-designated Google Drive parent folder (and its ID) is required before running this workflow. | Pre-requisite |
| A parallel GitHub Repository storage version of this workflow exists and is available free upon request from workflows@zmglobalit.com. A consolidated version (for both Google Drive and GitHub) is also available. | Custom builds and alternative versions |
| A custom version of this workflow or the GitHub version costs $15. Contact: workflows@zmglobalit.com | Commercial custom builds |
| Preview the GitHub storage version: | https://github.com/zianamitchell/n8n-workflow-images/blob/main/n8n-workflow-organizer-github.png?raw=1 |
| The Google Drive consolidated version is meant to be used in tandem with this Workflow Organizer Google Drive — both should link to the same parent folder. | Consolidated version note |
| This workflow is tagged "ZMGIT", "Standard-Template", and "v1" in the original instance. | Workflow metadata |
| The Wait nodes are intended to provide a ~5-second pause between upload batches to avoid triggering Google Drive's spam upload detector. Verify the Wait node configuration is set to time-based (not webhook-based) resumption to avoid execution hanging. | Wait node configuration |
| The `isNotArchived` node uses strict type validation, meaning only a boolean `false` value will pass. Null or undefined values will not match the false condition. | Filter cascade nuance |
| The `isTemplate` and `isDone` nodes use loose string-contains matching on the `tags` field. Since tags in the n8n API are returned as an array of objects, the contains operation behavior depends on n8n's array-to-string coercion. Verify that your workflows' tags serialize correctly. | Tag matching nuance |
| Workflow categorization priority: Active > Archived > Template > Done > Work-in-Progress. A workflow that is both active and archived would be captured by the Active branch first (since isActive is evaluated before isNotArchived). | Classification priority |
| The `createActive`, `createTemplate`, `createDone`, `createWIP`, and `createArchived` nodes all have "Execute Once" enabled, ensuring only one sub-folder is created per category per execution regardless of batch count. | Execute Once setting |
| The "Create folder" node assigns the Google Drive folder color `#38D0A3` (teal-green) to the top-level portfolio folder. Sub-category folders do not specify a color. | Folder color customization |