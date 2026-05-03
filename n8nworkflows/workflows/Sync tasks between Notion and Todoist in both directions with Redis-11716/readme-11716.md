Sync tasks between Notion and Todoist in both directions with Redis

https://n8nworkflows.xyz/workflows/sync-tasks-between-notion-and-todoist-in-both-directions-with-redis-11716


# Sync tasks between Notion and Todoist in both directions with Redis

## 1. Workflow Overview

**Title:** Sync tasks between Notion and Todoist in both directions with Redis

**Purpose:**  
This workflow synchronizes tasks between a Notion database and Todoist, primarily driven by Notion webhook events. It updates, completes/reopens, creates, or deletes Todoist tasks based on changes in Notion, and uses **Redis locks/flags** to prevent loops (ping-pong updates) and duplicate creations due to repeated webhooks.

**Target use cases**
- Teams managing tasks in Notion but executing in Todoist
- Keeping a Notion task database and a Todoist project aligned (including section mapping, due dates, recurrence strings, completion status)
- Preventing sync loops using Redis-based “lock” keys and short-lived “creating” flags

### 1.1 Setup & Helpers (one-time / occasional)
- Generate the JSON config for “Globals” nodes (Notion database ID, Todoist project ID, section mapping, timezone).
- Help activate Todoist OAuth (for a Todoist developer app) using a mini OAuth flow.

### 1.2 Notion Webhook Intake (replaceable trigger)
- A webhook node exists but is disabled in this export; a “Notion Trigger” placeholder node is used so the trigger can be swapped easily.

### 1.3 Redis loop-prevention + Fetch fresh Notion page
- Checks Redis lock for the Notion page ID to prevent processing changes that were caused by Todoist-side updates.
- Fetches the Notion page again because Notion may retry webhook deliveries and payload may be stale.

### 1.4 Mapping Notion → Todoist payload
- A Code node builds the Todoist task payload from Notion properties with fallbacks and edge-case handling (title, due dates, recurrence string formatting, reminders, priority mapping, section mapping, etc.).

### 1.5 Determine action in Todoist
- If Notion task is deleted/obsolete → delete in Todoist (if a Todoist ID exists).
- Else:
  - If Todoist ID exists → retrieve Todoist task with retry logic, then update content/priority/due/section if needed; complete/reopen based on Notion status checkbox.
  - If Todoist ID missing → create a Todoist task (with a Redis “creating_for_<db>” flag to avoid duplicates), then store the created Todoist ID back into Notion.

### 1.6 Lock Todoist task ID in Redis
- After actions that might trigger reciprocal sync, caches `lock_<todoistId>` briefly so other sync paths do not re-trigger.

---

## 2. Block-by-Block Analysis

### Block A — “Start here” prerequisites & instructions (documentation layer)
**Overview:** Sticky notes describing required Notion/Todoist/Redis setup and credential mapping.  
**Nodes involved:** Sticky Note11

#### Node: Sticky Note11
- **Type / role:** Sticky Note (instructions)
- **Content (preserved):**
  - Notion database template: https://steadfast-banjo-d1f.notion.site/17682b476c848086b002de766879aa71  
  - Requires properties (case matters): Name, Status, Priority, Due, Focus, Todoist ID, etc.
  - Todoist project must have sections matching Notion status (except Done/Obsolete)
  - Redis: https://redis.io/try-free/
  - n8n credential docs:
    - Notion: https://docs.n8n.io/integrations/builtin/credentials/notion/
    - Todoist: https://docs.n8n.io/integrations/builtin/credentials/todoist/
    - Redis: https://docs.n8n.io/integrations/builtin/credentials/redis/
- **Edge cases:** Misnamed Notion properties will break mapping code and/or Notion API PATCH paths.

---

### Block B — Config JSON generator for Globals nodes (Setup Helper)
**Overview:** Interactive forms to pick a Notion database and Todoist project, fetch sections, request timezone, and output a JSON config to copy into Globals nodes.  
**Nodes involved:** Sticky Note9, On form submission1, Get Notion Databases1, Prep Dropdown2, Choose Notion Database1, Get Notion Database ID1, Get projects1, Prep Dropdown3, Choose Todoist Project1, Get Todoist Project ID1, Get sections1, Choose Timezone1, Generate config1, Form1

#### Node: Sticky Note9
- **Type / role:** Sticky Note (instructions)
- **Content:** `# 1. Generate config JSON for Globals Nodes`

#### Node: On form submission1
- **Type / role:** Form Trigger (manual UI entry point)
- **Config:** “Setup Helper” form trigger
- **Outputs:** to **Get Notion Databases1**
- **Failure modes:** none typical; requires n8n forms enabled.

#### Node: Get Notion Databases1
- **Type / role:** Notion node; lists databases
- **Operation:** Database → Get All
- **Credentials:** Notion account 2
- **Outputs:** to **Prep Dropdown2**
- **Failure modes:** Notion auth errors; insufficient permissions.

#### Node: Prep Dropdown2
- **Type / role:** Code; converts Notion DB list to dropdown JSON
- **Logic:** builds `[{ option: <name> }, ...]` skipping “Inbox”
- **Output:** `{ options: JSON.stringify(dropDownValues) }`
- **Outputs:** to **Choose Notion Database1**
- **Failure modes:** If Notion response schema changes, `item.json.name` might not exist.

#### Node: Choose Notion Database1
- **Type / role:** Form; dropdown selection
- **Config:** Form JSON uses `{{ $json.options }}`
- **Outputs:** to **Get Notion Database ID1**
- **Failure modes:** If `options` missing/not valid JSON, form rendering fails.

#### Node: Get Notion Database ID1
- **Type / role:** Code; resolve selected DB name to DB ID
- **Key expressions:**
  - `$('Choose Notion Database1').first().json['Select Notion Database']`
  - `$('Get Notion Databases1').all()`
- **Outputs:** to **Get projects1**
- **Failure modes:** If multiple DBs share name; if selection mismatches, ID becomes null.

#### Node: Get projects1
- **Type / role:** HTTP Request; Todoist list projects
- **Endpoint:** `GET https://api.todoist.com/api/v1/projects`
- **Auth:** predefined Todoist credential type
- **Outputs:** to **Prep Dropdown3**
- **Failure modes:** Todoist auth, rate limits.

#### Node: Prep Dropdown3
- **Type / role:** Code; converts Todoist projects to dropdown JSON
- **Uses:** `results = $input.first().json.results`
- **Outputs:** to **Choose Todoist Project1**
- **Failure modes:** If Todoist API returns different shape.

#### Node: Choose Todoist Project1
- **Type / role:** Form; dropdown selection
- **Outputs:** to **Get Todoist Project ID1**

#### Node: Get Todoist Project ID1
- **Type / role:** Code; resolve selected project name to project ID
- **Uses:** `$('Get projects1').first().json.results`
- **Outputs:** to **Get sections1**
- **Failure modes:** Name mismatch => null ID.

#### Node: Get sections1
- **Type / role:** HTTP Request; Todoist list sections for project
- **Endpoint:** `GET https://api.todoist.com/api/v1/sections?project_id=<id>`
- **Outputs:** to **Choose Timezone1**
- **Failure modes:** invalid/missing project_id, auth errors.

#### Node: Choose Timezone1
- **Type / role:** Form; timezone text input (+ HTML help)
- **Help link:** https://www.iana.org/time-zones
- **Outputs:** to **Generate config1**

#### Node: Generate config1
- **Type / role:** Code; generates config JSON
- **Builds:**
  - `database_id` from **Get Notion Database ID1**
  - `project_id` from **Get Todoist Project ID1**
  - `sections[]` from **Get sections1** results
  - `timezone` from the timezone form
- **Outputs:** to **Form1**

#### Node: Form1
- **Type / role:** Form completion page; displays the JSON to copy
- **Completion message:** `{{ $json.toJsonString() }}`

---

### Block C — Todoist OAuth helper (activate developer app/webhook)
**Overview:** A small OAuth callback flow storing client ID/secret/state in workflow static data and exchanging an OAuth code for tokens.  
**Nodes involved:** Sticky Note10, Todoist Webhook Setup Helper1, Generate security token1, Store variables1, Redirect to Auth Page1, OAuth redirect1, Get variables1, Verify security token1, Exchange Tokens1, Respond with success1, Respond with error1

#### Node: Sticky Note10
- **Type / role:** Sticky Note
- **Content:** `# 2. Activate Todoist Webhook`

#### Node: Todoist Webhook Setup Helper1
- **Type / role:** Form Trigger (manual entry point)
- **Fields:** Client ID, Client secret
- **Response mode:** last node
- **Outputs:** to **Generate security token1**
- **Failure modes:** none typical.

#### Node: Generate security token1
- **Type / role:** Crypto; generate random `state`
- **Output field:** `state`
- **Outputs:** to **Store variables1**

#### Node: Store variables1
- **Type / role:** Code; store secrets/state in workflow static data
- **Writes:** `staticData.clientID`, `staticData.clientSecret`, `staticData.state`
- **Outputs:** to **Redirect to Auth Page1**
- **Failure modes:** If static data unavailable (rare) or overwritten accidentally.

#### Node: Redirect to Auth Page1
- **Type / role:** Form completion redirect to Todoist authorize URL
- **Redirect URL:**  
  `https://todoist.com/oauth/authorize?client_id={{ Client ID }}&scope=task:add,data:read_write,data:delete&state={{ state }}`
- **Failure modes:** Incorrect Todoist app configuration, wrong redirect URL in Todoist developer settings.

#### Node: OAuth redirect1
- **Type / role:** Webhook (GET); OAuth callback endpoint
- **Path:** `Todoist-outh`
- **Response mode:** “responseNode”
- **Outputs:** to **Get variables1**
- **Failure modes:** If Todoist redirect URI doesn’t match exactly.

#### Node: Get variables1
- **Type / role:** Code; read staticData into the current item
- **Outputs:** to **Verify security token1**

#### Node: Verify security token1
- **Type / role:** IF; CSRF protection check
- **Condition:** `$('OAuth redirect1').first().json.query.state == $json.state`
- **True →** Exchange Tokens1  
- **False →** Respond with error1

#### Node: Exchange Tokens1
- **Type / role:** HTTP Request; exchange code for token
- **Endpoint:** `POST https://todoist.com/oauth/access_token` (form-urlencoded)
- **Body:** client_id, client_secret, code from callback query
- **onError:** continueErrorOutput
- **Outputs:** success to Respond with success1; error to Respond with error1
- **Failure modes:** invalid code, invalid client credentials.

#### Node: Respond with success1 / Respond with error1
- **Type / role:** Respond to Webhook
- **Bodies:**
  - Success: “Developer App activated successfully. The window can be closed now.”
  - Error: “Something went wrong.”

---

### Block D — Webhook intake placeholder + globals for sync
**Overview:** Provides a replaceable trigger and injects a static config (“GlobalsN”) used by mapping logic; stores key execution metadata for debugging.  
**Nodes involved:** Sticky Note88, Sticky Note90, Sticky Note82, Notion Webhook (disabled), Notion Trigger, GlobalsN, Execution Data

#### Node: Sticky Note88
- **Type / role:** Sticky Note
- **Content:** Notion Webhook instructions; replaceable with emulator nodes if desired.

#### Node: Sticky Note90
- **Type / role:** Sticky Note
- **Content:** “Use Sync Setup Helper Workflow to generate the JSON and paste it in every Globals Nodes”

#### Node: Sticky Note82
- **Type / role:** Sticky Note
- **Content:** “Following nodes reference to this node, so the trigger can easily be replaced”

#### Node: Notion Webhook (disabled)
- **Type / role:** Webhook (POST) intake endpoint
- **Path:** `notion-webhook2`
- **Status:** disabled in this export
- **Output:** to Notion Trigger
- **Failure modes:** If enabled: signature/security not validated here; relies on obscurity/path.

#### Node: Notion Trigger
- **Type / role:** NoOp placeholder “trigger”
- **Role:** single reference point for downstream expressions (`$('Notion Trigger').item.json...`)
- **Output:** to GlobalsN

#### Node: GlobalsN
- **Type / role:** Set (raw JSON); global config
- **Config JSON includes:** `database_id`, `project_id`, `sections[]`
- **executeOnce:** true
- **Output:** to Execution Data
- **Failure modes:** Invalid JSON breaks all downstream mapping; wrong IDs cause Notion/Todoist requests to fail.

#### Node: Execution Data
- **Type / role:** Execution Data; store debug metadata
- **Saved keys:**
  - `database_id`, `project_id`
  - `notion_id` = `$('Notion Trigger').item.json.body.data.parent.id`
  - `source` = `"Notion"`
- **Output:** to Check if Notion ID is locked2

---

### Block E — Redis lock check + fetch current Notion page
**Overview:** Prevents loops by skipping processing if the Notion page is “locked”, and fetches the Notion page fresh.  
**Nodes involved:** Sticky Note92, Sticky Note79, Check if Notion ID is locked2, Only continue if not locked3, Get Notion task2

#### Node: Sticky Note92
- **Type / role:** Sticky Note
- **Content:** “Checking for a cached flag in Redis, to prevent overrides and endless loops.”

#### Node: Check if Notion ID is locked2
- **Type / role:** Redis GET lock
- **Key:** `lock_{{ $json.body.entity.id }}`
- **Reads into:** field `item`
- **Credentials:** Redis account 2
- **Output:** to Only continue if not locked3
- **Failure modes:** Redis connectivity/auth; missing body.entity.id if trigger payload differs.

#### Node: Only continue if not locked3
- **Type / role:** Filter; proceed if lock is not set to known values
- **Logic (OR):**
  - `$json.item != "Executed"` OR `$json.item != "SmallExecuted"`
- **Important:** As written, this OR condition is almost always true (a value cannot equal both at once). If the intention is “not equals Executed AND not equals SmallExecuted”, this should be **AND**.
- **Output:** to Get Notion task2
- **Failure modes:** Expression issues if `$json.item` missing.

#### Node: Sticky Note79
- **Type / role:** Sticky Note
- **Content:** Notion retries webhooks; re-fetch the page to ensure up-to-date data.

#### Node: Get Notion task2
- **Type / role:** Notion; fetch database page by ID
- **Page ID:** `$('Notion Trigger').item.json.body.entity.id`
- **Retry:** enabled (`maxTries: 2`, wait 3000ms)
- **Output:** to Map Notion to Todoist
- **Failure modes:** Notion page not found, permissions, rate limit.

---

### Block F — Map Notion → Todoist payload + store execution debug fields
**Overview:** Converts Notion properties to a Todoist task JSON payload with fallback values and due/recurrence handling. Stores more debugging data.  
**Nodes involved:** Sticky Note87, Sticky Note89, Sticky Note84, Sticky Note83, Map Notion to Todoist, Execution Data10

#### Node: Sticky Note87
- **Type / role:** Sticky Note
- **Content:** mapping includes fallback values for edge cases

#### Node: Map Notion to Todoist
- **Type / role:** Code; main mapping logic
- **Inputs:** Notion page JSON from Get Notion task2 and GlobalsN via `$('GlobalsN').item.json`
- **Outputs (selected):**
  - `id` (from Notion `TodoistID` rich text, else empty)
  - `content` (from Notion `Task Name` title, else `[empty]`)
  - `priority` mapping (Notion priority numeric → Todoist priority string)
  - `section_id` (mapped by Globals sections list; note: the condition in code appears incorrect—see edge cases)
  - `checked` (from Notion `Status.checkbox`)
  - `due_datetime` and/or `due_string`:
    - If Due date exists: set due_datetime; if recurrence exists, build due_string (with time formatting if datetime)
    - If Due end exists: override due_string with “<recurrence> until <end-date>”
    - If Due removed: sets `due_string = "no date"` to satisfy API expectations
  - `reminders` if `due_datetime` exists (push, 0 minutes before)
- **Key edge cases / likely bugs:**
  - **Section mapping condition**:  
    `if (eventData.properties.Section.select.name == ("\n"+ globals.sections[i].Section_name + "\n\n\n") || (globals.sections[i].Section_name) )`  
    The `|| (globals.sections[i].Section_name)` part is always truthy, so it will pick the **first** section_id every time. This likely needs to be:  
    `... == globals.sections[i].Section_name` without the always-true OR.
  - Notion property names must match exactly (e.g., `TodoistID`, `Task Name`, `Section`, `Status`, `Due`, `Recurrence`).
- **Output:** to Execution Data10
- **Failure modes:** Type errors if properties are missing/null; string split on undefined; invalid Notion schema.

#### Node: Sticky Note83
- **Type / role:** Sticky Note
- **Content:** store relevant info in execution data for debugging

#### Node: Execution Data10
- **Type / role:** Execution Data; store extra debug info
- **Saved keys:**
  - `todoist_id` = `$json.id`
  - `task_name` = `$json.content`
- **Output:** to Is Deleted?1

---

### Block G — Decide: deleted/obsolete vs active
**Overview:** If Notion task is archived or in “Obsolete” section, delete in Todoist; otherwise continue with update/create logic.  
**Nodes involved:** Is Deleted?1, Sticky Note93

#### Node: Is Deleted?1
- **Type / role:** IF
- **Condition (OR):**
  - Notion page `archived == true`
  - Notion `properties.Section.select.name == "Obsolete"`
- **True output →** Todoist ID exists2 (delete path)  
- **False output →** Todoist ID exists?2 (update/create path)
- **Failure modes:** If `properties.Section.select` is null (unselected), expression errors.

#### Node: Sticky Note93
- **Type / role:** Sticky Note
- **Content:** Edge case: task moved from Obsolete to Done

---

### Block H — Delete in Todoist when Notion is obsolete/deleted
**Overview:** If Todoist ID exists, delete the Todoist task (continue even if already deleted), then lock the Todoist ID in Redis to prevent loops.  
**Nodes involved:** Sticky Note77, Sticky Note81, Todoist ID exists2, Delete Task in Todoist3, Get todoist ID5, Lock Todoist ID5

#### Node: Sticky Note77
- **Type / role:** Sticky Note
- **Content:** “Delete the task if it does not fit into the filter anymore”

#### Node: Todoist ID exists2
- **Type / role:** Filter; only proceed if `$json.id` not empty
- **Output:** to Delete Task in Todoist3

#### Node: Sticky Note81
- **Type / role:** Sticky Note
- **Content:** continues on error; not necessary to retrieve task beforehand

#### Node: Delete Task in Todoist3
- **Type / role:** Todoist node; delete task
- **Operation:** delete
- **onError:** continueRegularOutput (so workflow proceeds even if delete fails)
- **Credentials:** Todoist account 2
- **Output:** to Get todoist ID5
- **Failure modes:** Task already deleted; permissions; API errors (ignored due to continue).

#### Node: Get todoist ID5
- **Type / role:** Set; normalize `id` for lock
- **Sets:** `id = $('Todoist ID exists2').item.json.id`
- **Output:** to Lock Todoist ID5

#### Node: Lock Todoist ID5
- **Type / role:** Redis SET lock
- **Key:** `lock_{{ $json.id }}`
- **TTL:** 80 seconds (note sticky says 15 seconds; config says 80)
- **Value:** `Executed`
- **Credentials:** Redis account
- **Failure modes:** Redis errors; incorrect TTL expectation.

---

### Block I — Update existing Todoist task (retrieve with retry)
**Overview:** If Notion has a Todoist ID, retrieve the Todoist task via HTTP; if not found, retry a few times; then decide whether to update fields and/or complete/reopen.  
**Nodes involved:** Sticky Note91, Todoist ID exists?2, Set tries2, Get Todoist Task3, Catch known error3, Wait3, Update tries2, If tries left3, Retry limit reached3, Switch

#### Node: Sticky Note91
- **Type / role:** Sticky Note
- **Content:** Retrieve Todoist task; advanced retry logic because “Get many tasks” doesn’t return completed tasks.

#### Node: Todoist ID exists?2
- **Type / role:** IF; check if `$json.id` not empty
- **True →** Set tries2 (retrieve path)  
- **False →** Check if creating flag exists3 (create path)
- **Failure modes:** missing `$json.id` due to mapping issues.

#### Node: Set tries2
- **Type / role:** Set; initialize `tries`
- **Sets:** `tries = $json.tries || 0` (keeps other fields)
- **Output:** to Get Todoist Task3

#### Node: Get Todoist Task3
- **Type / role:** HTTP Request; fetch Todoist task by ID
- **Endpoint:** `GET https://api.todoist.com/api/v1/tasks/{{ $json.id }}`
- **onError:** continueErrorOutput (so errors go to a separate path)
- **Outputs:**
  - Success → Switch
  - Error → Catch known error3
- **Failure modes:** 404 not found, auth/rate limits.

#### Node: Catch known error3
- **Type / role:** IF; detect “could not be found” in `$json.error`
- **True →** Wait3 (retry path)  
- **False →** Status is not Done2 (continues but note: this path is unusual; it uses Notion status to decide creation later)
- **Failure modes:** error shape may not contain `$json.error`.

#### Node: Wait3
- **Type / role:** Wait; delay before retry
- **Output:** to Update tries2
- **Failure modes:** none typical; increases execution time.

#### Node: Update tries2
- **Type / role:** Set; increment tries
- **Sets:** `tries = $('Set tries2').item.json.tries + 1`
- **Output:** to If tries left3

#### Node: If tries left3
- **Type / role:** IF; retry limit check
- **Condition:** `tries < 3`
- **True →** Set tries2 (loop)  
- **False →** Retry limit reached3

#### Node: Retry limit reached3
- **Type / role:** Stop and Error
- **Message:** “Retry limit reached”

#### Node: Switch
- **Type / role:** Switch; only one rule shown
- **Rule:** if `$('Notion Trigger').item.json.body.type == "page.properties_updated"`
- **Output:** to Differences exist and Has been completed?2 (both are connected from Switch output 0)
- **Edge cases:** If other Notion event types occur, rules may not match and downstream update logic may not run.

---

### Block J — Update Todoist fields if differences exist
**Overview:** Compares the mapped Todoist payload with the fetched Todoist task; if any differences exist, sends an update request to Todoist.  
**Nodes involved:** Sticky Note78, Differences exist, Update task in Todoist2, Get todoist ID4, Lock Todoist ID5

#### Node: Sticky Note78
- **Type / role:** Sticky Note
- **Content:** “Update the task if it needs to be changed”

#### Node: Differences exist
- **Type / role:** Filter; checks for any field differences (OR)
- **Compares:**
  - content
  - priority
  - due datetime vs Todoist due.date
  - section_id
  - due string
- **Notable risk:** `$('Map Notion to Todoist').item.json.due_datetime.split('.')[0]` will fail if due_datetime is missing. (There is an `|| null`, but it applies after `.split()`.)
- **Output:** to Update task in Todoist2

#### Node: Update task in Todoist2
- **Type / role:** HTTP Request; update Todoist task
- **Endpoint:** `POST https://api.todoist.com/api/v1/tasks/{{ $json.id }}`
- **Body:** JSON = `$('Map Notion to Todoist').item.json.toJsonString()`
- **Auth:** Todoist credential
- **Retry:** enabled (wait 3000ms)
- **Output:** to Get todoist ID4

#### Node: Get todoist ID4
- **Type / role:** Set; normalize `id` from Get Todoist Task3
- **Sets:** `id = $('Get Todoist Task3').item.json.id`
- **Output:** to Lock Todoist ID5 (locks with value “Executed”)

---

### Block K — Complete/Reopen Todoist based on Notion checkbox + special Notion correction
**Overview:** If Notion says task completed, close it in Todoist; else reopen. Also includes a special Notion update when certain due_string exists (likely an edge-case correction).  
**Nodes involved:** Has been completed?2, Mark as Completed in Todoist2, Mark as Incomplete in Todoist2, Filter, Update Notion, Sticky Note12

#### Node: Sticky Note12
- **Type / role:** Sticky Note
- **Content:** “Todoist Task Checker”

#### Node: Has been completed?2
- **Type / role:** IF; checks mapped `checked == true`
- **Condition:** `$('Map Notion to Todoist').item.json.checked` is true
- **True →** Mark as Completed in Todoist2  
- **False →** Mark as Incomplete in Todoist2

#### Node: Mark as Completed in Todoist2
- **Type / role:** Todoist; close task
- **Task ID:** `$('Get Todoist Task3').item.json.id`
- **Retry:** enabled
- **Outputs:** to Get todoist ID4 **and** Filter
- **Failure modes:** task already closed; API errors.

#### Node: Mark as Incomplete in Todoist2
- **Type / role:** Todoist; reopen task
- **Task ID:** `{{ $json.id }}`
- **Output:** to Get todoist ID4

#### Node: Filter
- **Type / role:** Filter; checks if mapped `due_string` exists
- **Condition:** exists `$('Map Notion to Todoist').item.json.due_string`
- **Output:** to Update Notion
- **Interpretation:** If due_string exists while completing, workflow patches Notion Status checkbox back to false (possibly to resolve a “no date” / recurrence mismatch edge case).

#### Node: Update Notion
- **Type / role:** HTTP Request; PATCH Notion page
- **Endpoint:** `PATCH https://api.notion.com/v1/pages/{{ notionPageId }}`
- **Body:** sets `properties.Status.checkbox = false`
- **Failure modes:** Notion permission errors; schema mismatch if Status is not checkbox.

---

### Block L — Create Todoist task when missing + prevent duplicates + write back to Notion
**Overview:** When Notion has no Todoist ID, create a Todoist task, prevent duplicate creations with a short Redis “creating flag”, then store the new Todoist ID back into Notion and lock the Todoist ID.  
**Nodes involved:** Sticky Note17, Sticky Note86, Status is not Done2, Check if creating flag exists3, Only continue if flag does not exist3, Set creating flag3, Create task in Todoist2, Store Todoist ID, Lock Todoist ID6, Lock Todoist ID5

#### Node: Sticky Note17
- **Type / role:** Sticky Note
- **Content:** “Create task if it does not exist (anymore)”

#### Node: Status is not Done2
- **Type / role:** Filter; only proceed if Notion Status is not “Done”
- **Condition:** `$('Get Notion task2').item.json.properties['Status'].status.name != "Done"`
- **Output:** to Check if creating flag exists3
- **Failure modes:** If Status is not a Notion “status” type, expression breaks.

#### Node: Sticky Note86
- **Type / role:** Sticky Note
- **Content:** Notion may fire multiple webhooks on creation; prevents duplicate Todoist tasks.

#### Node: Check if creating flag exists3
- **Type / role:** Redis GET
- **Key:** `creating_for_{{ $('Notion Trigger').item.json.body.data.parent.id }}`
- **Output field:** `item`
- **Output:** to Only continue if flag does not exist3
- **Failure modes:** Redis errors; parent.id missing.

#### Node: Only continue if flag does not exist3
- **Type / role:** Filter; proceed only if Redis returned empty
- **Condition:** `$json.item` is empty
- **Output:** to Set creating flag3

#### Node: Set creating flag3
- **Type / role:** Redis SET with TTL
- **Key:** `creating_for_<parent.id>`
- **TTL:** 10 seconds
- **Value:** current ISO timestamp
- **Output:** to Create task in Todoist2

#### Node: Create task in Todoist2
- **Type / role:** HTTP Request; create Todoist task
- **Endpoint:** `POST https://api.todoist.com/api/v1/tasks`
- **Body:** mapped JSON from Map Notion to Todoist
- **Outputs:**
  - to Lock Todoist ID5 (lock “Executed”)
  - to Store Todoist ID (write back to Notion)
- **Failure modes:** invalid payload (e.g., due fields), auth/rate limits.

#### Node: Store Todoist ID
- **Type / role:** HTTP Request; PATCH Notion page to store Todoist ID
- **Endpoint:** `PATCH https://api.notion.com/v1/pages/{{ notionPageId }}`
- **Body parameter path:** `properties.TodoistID.rich_text[0].text.content = {{ $json.id }}`
- **Output:** to Lock Todoist ID6
- **Failure modes:** Notion property name mismatch (`TodoistID` vs “Todoist ID”); rich_text array might be empty and index 0 path may fail in Notion.

#### Node: Lock Todoist ID6
- **Type / role:** Redis SET lock (secondary value)
- **Key:** `lock_{{ $json.id }}`
- **TTL:** 80 seconds
- **Value:** `SmallExecuted`
- **Purpose:** indicates a smaller/alternate lock reason (used in lock check node)
- **Failure modes:** Redis errors.

---

### Block M — Sticky note for webhook processing header
**Overview:** A labeling note for the webhook-processing area.  
**Nodes involved:** Sticky Note13, Sticky Note84, Sticky Note85, Sticky Note89, Sticky Note12, Sticky Note16, Sticky Note80, Sticky Note91, Sticky Note78, Sticky Note77

#### Node: Sticky Note13
- **Type / role:** Sticky Note
- **Content:** `#Processing Data Coming from webhooks`

#### Node: Sticky Note16 / Sticky Note80
- **Type / role:** Sticky Notes
- **Content:** “The Todoist ID is being cached in Redis for 15 seconds, so the other Diff sync does not get triggered.”
- **Note:** Actual TTL used in Redis lock nodes is 80 seconds.

#### Node: Sticky Note84
- **Type / role:** Sticky Note
- **Content:** “Store more information, which was not available earlier”

#### Node: Sticky Note85
- **Type / role:** Sticky Note (empty)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Notion Webhook | Webhook | Notion webhook intake (disabled) |  | Notion Trigger | ## Notion Webhook Grab the webhook URL and use it in the Notion automation. If using the NotionWebhookEmulator instead, replace this node with the nodes shown below. |
| Notion Trigger | NoOp | Replaceable trigger anchor | Notion Webhook | GlobalsN | Following nodes reference to this node, so the trigger can easily be replaced |
| GlobalsN | Set | Stores global config (DB/project/sections) | Notion Trigger | Execution Data | ## Set Globals Use Sync Setup Helper Workflow to generate the JSON and paste it in every Globals Nodes |
| Execution Data | Execution Data | Store debug metadata | GlobalsN | Check if Notion ID is locked2 | Store relevant information in highlighted execution data, in case of debugging |
| Check if Notion ID is locked2 | Redis | Read lock for Notion page | Execution Data | Only continue if not locked3 | Checking for a cached flag in Redis, to prevent overrides and endless loops. |
| Only continue if not locked3 | Filter | Skip processing if locked | Check if Notion ID is locked2 | Get Notion task2 | Checking for a cached flag in Redis, to prevent overrides and endless loops. |
| Get Notion task2 | Notion | Fetch latest Notion page | Only continue if not locked3 | Map Notion to Todoist | Notion tries to deliver webhooks again. To ensure the data is up to date, the payload must be retrieved again. |
| Map Notion to Todoist | Code | Build Todoist payload from Notion | Get Notion task2 | Execution Data10 | The mapping is more advanced and also includes fall-back values for edge-cases |
| Execution Data10 | Execution Data | Store extra debug fields | Map Notion to Todoist | Is Deleted?1 | Store more information, which was not available earlier |
| Is Deleted?1 | IF | Decide obsolete/deleted vs active | Execution Data10 | Todoist ID exists2; Todoist ID exists?2 | Edge case: the task has been moved from Obsolete to Done |
| Todoist ID exists2 | Filter | Ensure ID exists before delete | Is Deleted?1 (true) | Delete Task in Todoist3 | Delete the task if it does not fit into the filter anymore |
| Delete Task in Todoist3 | Todoist | Delete task (continue on error) | Todoist ID exists2 | Get todoist ID5 | Continues on error, so it is not necessary to retrieve task beforehand |
| Get todoist ID5 | Set | Normalize id for locking | Delete Task in Todoist3 | Lock Todoist ID5 | Delete the task if it does not fit into the filter anymore |
| Lock Todoist ID5 | Redis | Lock Todoist ID (Executed) | Get todoist ID4 / Get todoist ID5 / Create task in Todoist2 |  | The Todoist ID is being cached in Redis for 15 seconds, so the other Diff sync does not get triggered. |
| Todoist ID exists?2 | IF | Branch update/retrieve vs create | Is Deleted?1 (false) | Set tries2; Check if creating flag exists3 | The mapping between the Notion and Todoist tasks is stored in Notion |
| Set tries2 | Set | Initialize retry counter | Todoist ID exists?2; If tries left3 | Get Todoist Task3 | Retrieve Todoist task... advanced retry logic... |
| Get Todoist Task3 | HTTP Request | Get Todoist task by ID | Set tries2 | Switch; Catch known error3 | Retrieve Todoist task... advanced retry logic... |
| Catch known error3 | IF | Detect “could not be found” | Get Todoist Task3 (error) | Status is not Done2; Wait3 | Retrieve Todoist task... advanced retry logic... |
| Wait3 | Wait | Delay between retries | Catch known error3 (true) | Update tries2 | Retrieve Todoist task... advanced retry logic... |
| Update tries2 | Set | Increment retry counter | Wait3 | If tries left3 | Retrieve Todoist task... advanced retry logic... |
| If tries left3 | IF | Retry limit check | Update tries2 | Set tries2; Retry limit reached3 | Retrieve Todoist task... advanced retry logic... |
| Retry limit reached3 | Stop and Error | Abort after retries exhausted | If tries left3 (false) |  | Retrieve Todoist task... advanced retry logic... |
| Switch | Switch | Handle only certain Notion event types | Get Todoist Task3 (success) | Differences exist; Has been completed?2 |  |
| Differences exist | Filter | Detect changes requiring update | Switch | Update task in Todoist2 | Update the task if it needs to be changed |
| Update task in Todoist2 | HTTP Request | Update Todoist task | Differences exist | Get todoist ID4 | Update the task if it needs to be changed |
| Get todoist ID4 | Set | Normalize id from fetched task | Update task in Todoist2; Mark as Completed in Todoist2; Mark as Incomplete in Todoist2 | Lock Todoist ID5 |  |
| Has been completed?2 | IF | Close/reopen based on Notion checkbox | Switch | Mark as Completed in Todoist2; Mark as Incomplete in Todoist2 |  |
| Mark as Completed in Todoist2 | Todoist | Close task | Has been completed?2 (true) | Get todoist ID4; Filter |  |
| Mark as Incomplete in Todoist2 | Todoist | Reopen task | Has been completed?2 (false) | Get todoist ID4 |  |
| Filter | Filter | If due_string exists, patch Notion status false | Mark as Completed in Todoist2 | Update Notion |  |
| Update Notion | HTTP Request | PATCH Notion page status checkbox false | Filter |  |  |
| Status is not Done2 | Filter | Only create if Notion status != Done | Catch known error3 (false) | Check if creating flag exists3 |  |
| Check if creating flag exists3 | Redis | Prevent duplicate creations (flag) | Todoist ID exists?2 (false); Status is not Done2 | Only continue if flag does not exist3 | In rare cases Notion fires multiple webhooks... prevents duplicates in Todoist. |
| Only continue if flag does not exist3 | Filter | Proceed only if flag absent | Check if creating flag exists3 | Set creating flag3 | In rare cases Notion fires multiple webhooks... prevents duplicates in Todoist. |
| Set creating flag3 | Redis | Set short-lived creating flag | Only continue if flag does not exist3 | Create task in Todoist2 | In rare cases Notion fires multiple webhooks... prevents duplicates in Todoist. |
| Create task in Todoist2 | HTTP Request | Create Todoist task | Set creating flag3 | Lock Todoist ID5; Store Todoist ID | Create task if it does not exist (anymore) |
| Store Todoist ID | HTTP Request | Write created Todoist ID back to Notion | Create task in Todoist2 | Lock Todoist ID6 | The mapping between the Notion and Todoist tasks is stored in Notion |
| Lock Todoist ID6 | Redis | Lock Todoist ID (SmallExecuted) | Store Todoist ID |  | The Todoist ID is being cached in Redis for 15 seconds, so the other Diff sync does not get triggered. |
| Sticky Note11 | Sticky Note | Prerequisites/instructions |  |  | (content as shown in node) |
| Sticky Note9 | Sticky Note | Setup block header |  |  | # 1. Generate config JSON for Globals Nodes |
| Sticky Note10 | Sticky Note | OAuth helper header |  |  | # 2. Activate Todoist Webhook |
| Sticky Note13 | Sticky Note | Webhook processing header |  |  | #Processing Data Coming from webhooks |
| Sticky Note16 | Sticky Note | Redis lock explanation |  |  | The Todoist ID is being cached in Redis for 15 seconds, so the other Diff sync does not get triggered. |
| Sticky Note17 | Sticky Note | Create block header |  |  | Create task if it does not exist (anymore) |
| Sticky Note77 | Sticky Note | Delete block note |  |  | Delete the task if it does not fit into the filter anymore |
| Sticky Note78 | Sticky Note | Update block note |  |  | Update the task if it needs to be changed |
| Sticky Note79 | Sticky Note | Notion refetch rationale |  |  | Notion tries to deliver webhooks again... payload must be retrieved again. |
| Sticky Note80 | Sticky Note | Redis lock explanation |  |  | The Todoist ID is being cached in Redis for 15 seconds, so the other Diff sync does not get triggered. |
| Sticky Note81 | Sticky Note | Delete continues on error |  |  | Continues on error, so it is not necessary to retrieve task beforehand |
| Sticky Note82 | Sticky Note | Trigger replaceability |  |  | Following nodes reference to this node, so the trigger can easily be replaced |
| Sticky Note83 | Sticky Note | Debug info note |  |  | Store relevant information in highlighted execution data, in case of debugging |
| Sticky Note84 | Sticky Note | Debug info note |  |  | Store more information, which was not available earlier |
| Sticky Note85 | Sticky Note | Empty |  |  |  |
| Sticky Note86 | Sticky Note | Duplicate prevention note |  |  | In rare cases Notion fires multiple webhooks... prevents duplicates in Todoist. |
| Sticky Note87 | Sticky Note | Mapping note |  |  | The mapping is more advanced and also includes fall-back values for edge-cases |
| Sticky Note88 | Sticky Note | Notion webhook note |  |  | ## Notion Webhook Grab the webhook URL... |
| Sticky Note89 | Sticky Note | Mapping storage note |  |  | The mapping between the Notion and Todoist tasks is stored in Notion |
| Sticky Note90 | Sticky Note | Globals instructions |  |  | ## Set Globals Use Sync Setup Helper Workflow to generate the JSON and paste it in every Globals Nodes |
| Sticky Note91 | Sticky Note | Retry logic note |  |  | Retrieve Todoist task... advanced retry logic... |
| Sticky Note92 | Sticky Note | Redis loop prevention |  |  | Checking for a cached flag in Redis, to prevent overrides and endless loops. |
| Sticky Note93 | Sticky Note | Edge case note |  |  | Edge case: the task has been moved from Obsolete to Done |
| Sticky Note12 | Sticky Note | Todoist task checker label |  |  | ## Todoist Task Checker |
| Todoist Webhook Setup Helper1 | Form Trigger | Collect Todoist OAuth app credentials |  | Generate security token1 | # 2. Activate Todoist Webhook |
| Generate security token1 | Crypto | Generate OAuth state token | Todoist Webhook Setup Helper1 | Store variables1 | # 2. Activate Todoist Webhook |
| Store variables1 | Code | Persist OAuth vars to static data | Generate security token1 | Redirect to Auth Page1 | # 2. Activate Todoist Webhook |
| Redirect to Auth Page1 | Form | Redirect to Todoist OAuth authorize | Store variables1 |  | # 2. Activate Todoist Webhook |
| OAuth redirect1 | Webhook | OAuth callback endpoint |  | Get variables1 | # 2. Activate Todoist Webhook |
| Get variables1 | Code | Read static data into item | OAuth redirect1 | Verify security token1 | # 2. Activate Todoist Webhook |
| Verify security token1 | IF | Validate returned state | Get variables1 | Exchange Tokens1; Respond with error1 | # 2. Activate Todoist Webhook |
| Exchange Tokens1 | HTTP Request | Exchange code for token | Verify security token1 (true) | Respond with success1; Respond with error1 | # 2. Activate Todoist Webhook |
| Respond with success1 | Respond to Webhook | OAuth success response | Exchange Tokens1 |  | # 2. Activate Todoist Webhook |
| Respond with error1 | Respond to Webhook | OAuth error response | Verify security token1 (false) / Exchange Tokens1 (error) |  | # 2. Activate Todoist Webhook |
| On form submission1 | Form Trigger | Start config helper |  | Get Notion Databases1 | # 1. Generate config JSON for Globals Nodes |
| Get projects1 | HTTP Request | Fetch Todoist projects | Get Notion Database ID1 | Prep Dropdown3 | # 1. Generate config JSON for Globals Nodes |
| Get sections1 | HTTP Request | Fetch Todoist sections | Get Todoist Project ID1 | Choose Timezone1 | # 1. Generate config JSON for Globals Nodes |
| Prep Dropdown3 | Code | Dropdown JSON from projects | Get projects1 | Choose Todoist Project1 | # 1. Generate config JSON for Globals Nodes |
| Choose Todoist Project1 | Form | Select Todoist project | Prep Dropdown3 | Get Todoist Project ID1 | # 1. Generate config JSON for Globals Nodes |
| Get Todoist Project ID1 | Code | Resolve project name → ID | Choose Todoist Project1 | Get sections1 | # 1. Generate config JSON for Globals Nodes |
| Prep Dropdown2 | Code | Dropdown JSON from Notion DBs | Get Notion Databases1 | Choose Notion Database1 | # 1. Generate config JSON for Globals Nodes |
| Choose Notion Database1 | Form | Select Notion database | Prep Dropdown2 | Get Notion Database ID1 | # 1. Generate config JSON for Globals Nodes |
| Get Notion Database ID1 | Code | Resolve DB name → ID | Choose Notion Database1 | Get projects1 | # 1. Generate config JSON for Globals Nodes |
| Choose Timezone1 | Form | Provide timezone | Get sections1 | Generate config1 | # 1. Generate config JSON for Globals Nodes |
| Generate config1 | Code | Build Globals JSON | Choose Timezone1 | Form1 | # 1. Generate config JSON for Globals Nodes |
| Form1 | Form | Output JSON for copy/paste | Generate config1 |  | # 1. Generate config JSON for Globals Nodes |
| Get Notion Databases1 | Notion | Fetch Notion databases | On form submission1 | Prep Dropdown2 | # 1. Generate config JSON for Globals Nodes |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Notion credential (internal integration token) with access to the target database/pages.
   2. Todoist credential (OAuth2 via n8n Todoist credential type).
   3. Redis credential (Redis Cloud or self-host).

2) **Prepare Notion & Todoist structures**
   1. Create/duplicate a Notion database with the required properties (names must match the mapping code’s expectations).
   2. Create a Todoist project with sections corresponding to your Notion sections/status mapping (excluding Done/Obsolete as noted).

3) **(Optional but recommended) Build the Setup Helper block**
   1. Add **Form Trigger** node “On form submission1” (Setup Helper).
   2. Add **Notion** node “Get Notion Databases1” (Database → Get All) connected from the form trigger.
   3. Add **Code** “Prep Dropdown2” to output `options` JSON.
   4. Add **Form** “Choose Notion Database1” using the dropdown JSON.
   5. Add **Code** “Get Notion Database ID1” to resolve selected name → ID.
   6. Add **HTTP Request** “Get projects1” (Todoist GET /projects).
   7. Add **Code** “Prep Dropdown3”, then **Form** “Choose Todoist Project1”.
   8. Add **Code** “Get Todoist Project ID1”.
   9. Add **HTTP Request** “Get sections1” with query `project_id`.
   10. Add **Form** “Choose Timezone1” (text field + link to https://www.iana.org/time-zones).
   11. Add **Code** “Generate config1” producing `{database_id, project_id, sections, timezone}`.
   12. Add **Form** “Form1” (completion) that displays `{{ $json.toJsonString() }}`.

4) **Create the main webhook intake**
   1. Add **Webhook** node “Notion Webhook” (POST, path e.g. `notion-webhook2`).  
      - In this exported workflow it is **disabled**; you can enable it.
   2. Add a **NoOp** node “Notion Trigger” after the webhook.  
      - Downstream expressions will reference this node, so keep its name stable.

5) **Add Globals config node**
   1. Add **Set** node “GlobalsN” with **Mode: Raw JSON** and paste the JSON generated by the Setup Helper.
   2. Enable **Execute once**.
   3. Connect: Notion Trigger → GlobalsN.

6) **Add execution debug capture**
   1. Add **Execution Data** node “Execution Data” to store database_id, project_id, notion_id, source.
   2. Connect: GlobalsN → Execution Data.

7) **Add Redis Notion lock check**
   1. Add **Redis** node “Check if Notion ID is locked2” (GET) key `lock_{{ $json.body.entity.id }}` into property `item`.
   2. Add **Filter** node “Only continue if not locked3”.  
      - **Recommended fix:** use **AND**: `$json.item != "Executed"` AND `$json.item != "SmallExecuted"`.
   3. Connect: Execution Data → Redis GET → Filter.

8) **Fetch the Notion page fresh**
   1. Add **Notion** node “Get Notion task2” (Database Page → Get) pageId `$('Notion Trigger').item.json.body.entity.id`.
   2. Enable retries (e.g., 2 tries, 3000ms wait).
   3. Connect: Only continue if not locked3 → Get Notion task2.

9) **Add mapping code**
   1. Add **Code** node “Map Notion to Todoist” and implement:
      - Read globals from `$('GlobalsN').item.json`
      - Build Todoist payload fields (content/priority/section/due/reminders/etc.)
   2. **Recommended fix:** correct section mapping condition so it matches the selected name properly.
   3. Connect: Get Notion task2 → Map Notion to Todoist.

10) **Store more execution debug**
   1. Add **Execution Data** node “Execution Data10” to save `todoist_id` and `task_name`.
   2. Connect: Map Notion to Todoist → Execution Data10.

11) **Deleted/Obsolete routing**
   1. Add **IF** node “Is Deleted?1”:
      - True if Notion page is archived OR Section == “Obsolete”.
   2. Connect: Execution Data10 → Is Deleted?1.

12) **Delete branch**
   1. Add **Filter** “Todoist ID exists2” (`$json.id` not empty).
   2. Add **Todoist** “Delete Task in Todoist3” (delete) with **Continue on Fail**.
   3. Add **Set** “Get todoist ID5” to set `id` from the earlier branch.
   4. Add **Redis** “Lock Todoist ID5” (SET) key `lock_{{ $json.id }}`, TTL ~80s, value “Executed”.
   5. Connect: Is Deleted?1 (true) → Todoist ID exists2 → Delete → Get todoist ID5 → Lock Todoist ID5.

13) **Update/Create branch (Todoist ID present?)**
   1. Add **IF** “Todoist ID exists?2” (id not empty).
   2. Connect: Is Deleted?1 (false) → Todoist ID exists?2.

14) **Retrieve Todoist task with retry logic**
   1. Add **Set** “Set tries2” (tries = `$json.tries || 0`, include other fields).
   2. Add **HTTP Request** “Get Todoist Task3” GET `/tasks/{{ $json.id }}` with **Continue on Error output**.
   3. Add **IF** “Catch known error3” (`$json.error` contains “could not be found”).
   4. Add **Wait** “Wait3”.
   5. Add **Set** “Update tries2” (tries + 1).
   6. Add **IF** “If tries left3” (tries < 3), else **Stop and Error** “Retry limit reached3”.
   7. Wire the loop: Get Todoist Task3 error → Catch known error3 true → Wait3 → Update tries2 → If tries left3 true → Set tries2 → Get Todoist Task3.

15) **On successful fetch: decide update and completion**
   1. Add **Switch** node “Switch” with rule: Notion Trigger body.type equals `page.properties_updated`.
   2. From Switch output, connect to:
      - **Filter** “Differences exist”
      - **IF** “Has been completed?2”

16) **Update fields if different**
   1. Add **HTTP Request** “Update task in Todoist2” POST `/tasks/{{ $json.id }}` with JSON from `Map Notion to Todoist`.
   2. Add **Set** “Get todoist ID4” to normalize `id` from fetched task.
   3. Add **Redis** “Lock Todoist ID5” (SET, value Executed).
   4. Connect: Differences exist → Update task → Get todoist ID4 → Lock Todoist ID5.

17) **Complete/Reopen**
   1. Add **Todoist** “Mark as Completed in Todoist2” (close) using ID from Get Todoist Task3.
   2. Add **Todoist** “Mark as Incomplete in Todoist2” (reopen) using `$json.id`.
   3. Connect: Has been completed?2 → close or reopen → Get todoist ID4 → Lock Todoist ID5.
   4. Add **Filter** “Filter” to check if mapped due_string exists; if yes, PATCH Notion via **HTTP Request** “Update Notion” to set Status checkbox false.

18) **Create branch (when no Todoist ID)**
   1. Add **Filter** “Status is not Done2” (Notion status.name != Done).
   2. Add **Redis GET** “Check if creating flag exists3” key `creating_for_{{ parent.id }}`.
   3. Add **Filter** “Only continue if flag does not exist3” checking returned item is empty.
   4. Add **Redis SET** “Set creating flag3” TTL 10s.
   5. Add **HTTP Request** “Create task in Todoist2” POST `/tasks` with mapped JSON.
   6. Add **HTTP Request** “Store Todoist ID” PATCH Notion page property `TodoistID` with created id.
   7. Add **Redis SET** “Lock Todoist ID6” TTL 80s value “SmallExecuted”.
   8. Also connect Create task in Todoist2 → Lock Todoist ID5 (Executed) to avoid immediate loops.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Notion database template and required properties (case matters) | https://steadfast-banjo-d1f.notion.site/17682b476c848086b002de766879aa71 |
| Redis Cloud free instance | https://redis.io/try-free/ |
| n8n Notion credentials documentation | https://docs.n8n.io/integrations/builtin/credentials/notion/ |
| n8n Todoist credentials documentation | https://docs.n8n.io/integrations/builtin/credentials/todoist/ |
| n8n Redis credentials documentation | https://docs.n8n.io/integrations/builtin/credentials/redis/ |
| Timezone reference (IANA database) | https://www.iana.org/time-zones |
| Potential logic issue: “Only continue if not locked3” uses OR and likely should use AND | Redis lock block |
| Potential mapping bug: section mapping condition in code is always true due to `|| (globals.sections[i].Section_name)` | Map Notion to Todoist block |
| TTL mismatch: sticky note says 15s but Redis lock nodes use 80s | Redis lock behavior |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.