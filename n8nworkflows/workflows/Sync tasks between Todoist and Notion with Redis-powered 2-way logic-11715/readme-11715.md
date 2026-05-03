Sync tasks between Todoist and Notion with Redis-powered 2-way logic

https://n8nworkflows.xyz/workflows/sync-tasks-between-todoist-and-notion-with-redis-powered-2-way-logic-11715


# Sync tasks between Todoist and Notion with Redis-powered 2-way logic

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync tasks between Todoist and Notion with Redis-powered 2-way logic

This workflow implements **event-driven synchronization from Todoist to Notion**, with **Redis-based locking and “creation flags”** to prevent feedback loops and duplicate creation. It also contains **two helper entrypoints**:
- a **Setup Helper form flow** to generate the JSON configuration used by “Globals” nodes (Notion database ID, Todoist project ID, section mapping, timezone),
- a **Todoist OAuth activation flow** to help activate a Todoist developer app for webhook usage.

Although the title mentions “2-way”, the provided JSON mainly covers **Todoist → Notion** propagation plus loop-prevention that anticipates a second direction (e.g., Notion-driven changes triggering Todoist updates).

### Logical Blocks
1.1 **Initial Setup / Prerequisites (sticky instructions)**  
1.2 **Config Generation Helper (forms) → produces Globals JSON**  
1.3 **Todoist Webhook OAuth Activation Helper**  
1.4 **Todoist Webhook Reception + Redis lock gate**  
1.5 **Fetch fresh Todoist task + retry logic for late webhooks / deletions**  
1.6 **Map Todoist task fields to Notion schema (incl. timezone + recurrence parsing)**  
1.7 **Find matching Notion page (by TodoistID) and branch: create vs update**  
1.8 **Create Notion page (Task vs Subtask variants; due date variants)**  
1.9 **Copy Todoist description into Notion blocks, then overwrite Todoist description with Notion link**  
1.10 **Update Notion page if differences exist (due variants) + completion status sync**  
1.11 **Deletion handling: mark Notion as Obsolete**  
1.12 **Redis locks written to suppress opposite-direction triggers**

---

## 2. Block-by-Block Analysis

### 1.1 Initial Setup / Prerequisites (sticky instructions)
**Overview:** Documents required Notion database structure, Todoist project/sections expectations, Redis requirement, and credential setup. These are essential for the workflow to function.

**Nodes Involved:**  
- Sticky Note6

**Node Details**
- **Sticky Note6 (Sticky Note)**
  - **Role:** Human instructions for prerequisites.
  - **Key content (preserved):**
    - Notion DB template: https://steadfast-banjo-d1f.notion.site/17682b476c848086b002de766879aa71  
    - Required Notion properties (case matters): Name, Status, Priority, Due, Focus, Todoist ID
    - Todoist project must have matching sections
    - Redis setup link: https://redis.io/try-free/
    - n8n credential docs:
      - Notion: https://docs.n8n.io/integrations/builtin/credentials/notion/
      - Todoist: https://docs.n8n.io/integrations/builtin/credentials/todoist/
      - Redis: https://docs.n8n.io/integrations/builtin/credentials/redis/
  - **Failure modes:** If Notion database properties don’t match, later Notion updates/creates fail (400 validation errors).

---

### 1.2 Config Generation Helper (forms) → produces Globals JSON
**Overview:** A form-driven helper flow to select a Notion database and Todoist project, fetch Todoist sections, ask for timezone, then generate a JSON snippet that must be pasted into the Globals nodes in the sync flow.

**Nodes Involved:**  
- Sticky Note7  
- On form submission  
- Get Notion Databases  
- Prep Dropdown  
- Choose Notion Database  
- Get Notion Database ID  
- Get projects  
- Prep Dropdown1  
- Choose Todoist Project  
- Get Todoist Project ID  
- Get sections  
- Choose Timezone  
- Generate config  
- Form

**Node Details**
- **Sticky Note7 (Sticky Note)**
  - Role: Label for this helper block: “# 1. Generate config JSON for Globals Nodes”.

- **On form submission (Form Trigger)**
  - **Role:** Entry point to launch the setup helper.
  - **Config:** Form title “Setup Helper”.
  - **Output:** Form submission data.
  - **Edge cases:** Webhook URL must be reachable; if n8n behind auth, ensure proper exposure.

- **Get Notion Databases (Notion node)**
  - **Role:** Lists all databases accessible by the Notion integration.
  - **Config:** Resource `database`, operation `getAll`.
  - **Credentials:** “Notion account 2”.
  - **Failure modes:** Notion auth, missing permissions to databases.

- **Prep Dropdown (Code)**
  - **Role:** Converts Notion databases list into dropdown options JSON (skips name == “Inbox”).
  - **Output:** `{ options: JSON.stringify([{option: ...}, ...]) }`
  - **Edge cases:** If database names are duplicated, selection later may be ambiguous.

- **Choose Notion Database (Form)**
  - **Role:** Presents dropdown for database selection using generated options.
  - **Config:** JSON-defined form with required dropdown.
  - **Edge cases:** If `options` is not valid JSON, form rendering fails.

- **Get Notion Database ID (Code)**
  - **Role:** Finds Notion DB ID by matching chosen name.
  - **Key expressions:** reads `$('Choose Notion Database').first().json['Select Notion Database']`
  - **Edge cases:** If not found, returns `database_id = null` → later failures.

- **Get projects (HTTP Request)**
  - **Role:** Fetches Todoist projects.
  - **Endpoint:** `GET https://api.todoist.com/api/v1/projects`
  - **Credentials:** “Todoist account 2”
  - **Failure modes:** Todoist token invalid; endpoint version mismatch; rate limits.

- **Prep Dropdown1 (Code)**
  - **Role:** Builds dropdown options from `results` array in projects response.
  - **Output:** `{ options: JSON.stringify([{option: project.name}, ...]) }`

- **Choose Todoist Project (Form)**
  - **Role:** Dropdown to pick Todoist project.

- **Get Todoist Project ID (Code)**
  - **Role:** Match selected project name → `project_id`.
  - **Edge cases:** Same project name duplicates or mismatched response structure.

- **Get sections (HTTP Request)**
  - **Role:** Fetches sections for selected project.
  - **Endpoint:** `GET https://api.todoist.com/api/v1/sections?project_id=...`
  - **Failure modes:** project_id null; Todoist API errors.

- **Choose Timezone (Form)**
  - **Role:** Requests IANA timezone string (e.g., `America/New_York`).
  - **Includes link:** https://www.iana.org/time-zones

- **Generate config (Code)**
  - **Role:** Builds final JSON config:
    - `database_id`, `project_id`, `sections: [{Section_id, Section_name}]`, `timezone`
  - **Dependencies:** `Get sections`, `Get Notion Database ID`, `Get Todoist Project ID`, timezone from input.
  - **Edge cases:** Todoist `results` array missing; produces empty sections mapping.

- **Form (Form completion)**
  - **Role:** Displays “Copy this to the Globals Nodes” with the JSON output.

---

### 1.3 Todoist OAuth activation helper (for webhook developer app)
**Overview:** Helps activate a Todoist developer app by performing OAuth: generates `state`, stores client credentials in workflow static data, redirects to Todoist authorization page, then handles callback and exchanges code for tokens.

**Nodes Involved:**  
- Sticky Note8  
- Todoist Webhook Setup Helper  
- Generate security token  
- Store variables  
- Redirect to Auth Page  
- OAuth redirect  
- Get variables  
- Verify security token  
- Exchange Tokens  
- Respond with success  
- Respond with error

**Node Details**
- **Sticky Note8 (Sticky Note)**
  - Role: “# 2. Activate Todoist Webhook”

- **Todoist Webhook Setup Helper (Form Trigger)**
  - Role: Entry point form to collect Client ID / Client secret.
  - Response mode: “lastNode”.
  - Edge cases: Ensure HTTPS/publicly accessible to complete OAuth.

- **Generate security token (Crypto)**
  - Role: Generates random `state` token.

- **Store variables (Code)**
  - Role: Stores `clientID`, `clientSecret`, `state` into workflow static data.
  - Notes: Uses `$getWorkflowStaticData('global')`.

- **Redirect to Auth Page (Form completion/redirect)**
  - Role: Redirects user to:
    - `https://todoist.com/oauth/authorize?client_id=...&scope=task:add,data:read_write,data:delete&state=...`
  - Failure modes: wrong client_id; mismatch redirect URL registered in Todoist app.

- **OAuth redirect (Webhook, GET, response via responseNode)**
  - Role: OAuth callback endpoint (`path: Todoist-outh`).
  - Expects query parameters `code` and `state`.

- **Get variables (Code)**
  - Role: Retrieves staticData and attaches to item for verification/exchange.

- **Verify security token (IF)**
  - Role: Ensures callback `state` matches stored `state`.
  - True → Exchange Tokens; False → Respond with error.

- **Exchange Tokens (HTTP Request)**
  - Role: `POST https://todoist.com/oauth/access_token` (form-urlencoded) with client_id/client_secret/code.
  - onError: `continueErrorOutput` so it can branch to error response.
  - Failure modes: invalid code, expired code, redirect mismatch.

- **Respond with success / Respond with error (Respond to Webhook)**
  - Role: Returns human-readable success/error pages.

---

### 1.4 Todoist Webhook Reception + Redis lock gate
**Overview:** Receives Todoist webhook events, loads Globals, stores debug execution data, then checks Redis for a short-lived lock keyed by Todoist task ID to prevent endless loops.

**Nodes Involved:**  
- Sticky Note, Sticky Note68, Sticky Note66, Sticky Note65, Sticky Note43, Sticky Note40, Sticky Note54  
- Todoist Webhook1  
- Todoist Trigger1 (NoOp indirection)  
- GlobalsT1  
- Execution Data9  
- Check if Todoist ID is locked1  
- Only continue if not locked1  
- If event is not deleted1

**Node Details**
- **Todoist Webhook1 (Webhook, POST)**
  - Role: Main sync entry point (`path: todoist-webhook`).
  - Output: Webhook body includes `event_name`, `event_data` (incl. task id).
  - Failure modes: Todoist signature verification is not implemented here (if Todoist provides signing, it’s not checked).

- **Sticky Note43**
  - Role: Explains indirection: nodes reference “Todoist Trigger1” so trigger can be replaced.

- **Todoist Trigger1 (NoOp)**
  - Role: Reference/alias node for event payload.

- **GlobalsT1 (Set node in raw JSON mode)**
  - Role: Provides global configuration (`database_id`, `project_id`, section mappings, timezone).
  - Important: Sticky Note66 says to generate JSON via helper workflow and paste it here.
  - Edge cases: Wrong IDs → Notion queries fail; wrong timezone → date conversions wrong.

- **Execution Data9 (Execution Data)**
  - Role: Stores debug metadata: `database_id`, `project_id`, `todoist_id`, `source=Todoist`.
  - Useful for later troubleshooting.

- **Sticky Note40**
  - Role: “Checking for a cached flag in Redis, to prevent overrides and endless loops.”

- **Check if Todoist ID is locked1 (Redis GET)**
  - Role: Reads `lock_<todoist_task_id>` into `$json.item`.
  - Key expression: `lock_{{ $json.body.event_data.id }}`.
  - Failure modes: Redis auth/connection; missing Redis instance.

- **Only continue if not locked1 (Filter)**
  - Role: Passes through only if lock value is empty (string empty check).
  - Condition: `$json.item` is empty.
  - Edge cases: If Redis returns something unexpected (e.g., “null” as string), filter logic may not match intention.

- **Sticky Note54**
  - Role: Notes deletion detection relies on event type: “Using the event type from Todoist is the only option to actually recognize, if a task was deleted”.

- **If event is not deleted1 (IF)**
  - Role: Branch:
    - Not deleted → fetch task from Todoist (fresh) with retry.
    - Deleted → go to Notion lookup/delete handling (`Get Notion Task`).

---

### 1.5 Fetch fresh Todoist task + retry logic
**Overview:** Because webhook payload may be stale and Todoist “get many” endpoints omit completed tasks, the workflow re-fetches the task by ID. If task is deleted and endpoint returns “could not be found”, it retries up to 3 times (to handle overlapping delete events).

**Nodes Involved:**  
- Sticky Note5  
- Set tries  
- Get Todoist Task  
- Catch known error  
- Wait  
- Update tries  
- If tries left  
- Retry limit reached  
- End here1

**Node Details**
- **Sticky Note5**
  - Role: Explains late webhooks + retry rationale.

- **Set tries (Set)**
  - Role: Initializes `tries = $json.tries || 0` while keeping other fields.
  - Input: from “If event is not deleted1” (true branch) or from retry loop.

- **Get Todoist Task (HTTP Request)**
  - Role: GET task details fresh:
    - `GET https://api.todoist.com/api/v1/tasks/<id>`
  - Credentials: Todoist account.
  - `onError: continueErrorOutput` to capture error into output for branching.
  - `retryOnFail: true` (node-level retry).
  - Failure modes: 404/“could not be found”; rate limits; credential mismatch.

- **Catch known error (IF)**
  - Role: Checks if `$json.error` contains “could not be found”.
  - True branch → End here1 (stop further processing, no more retries needed per note) OR Wait path depending on connection:
    - Connections show: output 0 → End here1, output 1 → Wait (meaning: if condition matches? In n8n IF: output[0]=true, output[1]=false).
    - Here it is configured to “contains ‘could not be found’”. If TRUE → End here1. If FALSE → Wait (retry), which is slightly counterintuitive given the sticky note; but the wiring indicates retries happen when error is NOT “could not be found”. This is worth validating in your instance.

- **Wait (Wait)**
  - Role: Pause before retrying (webhook overlap / eventual consistency).
  - Continues to Update tries.

- **Update tries (Set)**
  - Role: increments `tries = Set tries.tries + 1`.

- **If tries left (IF)**
  - Role: checks `tries < 3`; if yes → loop back to Set tries; if no → Stop.

- **Retry limit reached (Stop and Error)**
  - Role: throws “Retry limit reached”.

- **End here1 (NoOp)**
  - Role: terminates branch cleanly without error.

---

### 1.6 Map Todoist task fields to Notion schema
**Overview:** Converts Todoist task JSON into a normalized structure expected by Notion: task name, Todoist ID, priority mapping, section mapping via Globals, due start/end, recurrence parsing, and timezone conversions using Luxon.

**Nodes Involved:**  
- Sticky Note55, Sticky Note24  
- Map Todoist to Notion1  
- Execution Data5

**Node Details**
- **Map Todoist to Notion1 (Code)**
  - Role: Central transformation/mapping.
  - Key outputs:
    - `todoist_id = id`
    - `task_name = content`
    - `status` boolean from `checked` / `is_completed`
    - `priority` mapped 1→4, 2→3, 3→2, 4→1 (string)
    - `section_name` resolved from `globals.sections[]` by `section_id` else “Inbox”
    - `start_date` from `due.date` (if ISO contains `T`, converted to `globals.timezone` using `luxon`)
    - If recurring: parse `due.string` to derive:
      - `recurrence = "every " + ...`
      - `end_date` from “ending YYYY-MM-DD” plus time if relevant
    - `created_at = added_at`
  - Dependencies: requires GlobalsT1 fields (`sections`, `timezone`).
  - Version requirements: Code node uses `require('luxon')`—Luxon must be available in n8n runtime (n8n typically includes it, but not guaranteed in all restricted environments).
  - Edge cases:
    - Todoist API fields differ by plan/version (`checked` vs `is_completed`); mapping logic uses both.
    - `due.string` parsing is heuristic; unusual recurrence phrases may produce empty recurrence.
    - Timezone conversion: code appends `+05:30` explicitly in one place (endtime) which is inconsistent if timezone is not Asia/Kolkata—this is a potential bug if you change timezone.

- **Execution Data5 (Execution Data)**
  - Role: Stores `task_name` now that it is available (from Get Todoist Task).
  - Sticky Note24 explains purpose: store more info for debugging.

---

### 1.7 Find matching Notion page and branch: create vs update
**Overview:** Queries Notion database for page where `TodoistID` equals the Todoist task id. If not found, proceeds to create (with a Redis “creating_for_<id>” flag to prevent duplicate creation). If found, proceed by event type to update content/properties or completion state.

**Nodes Involved:**  
- Sticky Note41, Sticky Note60, Sticky Note38, Sticky Note44  
- Get many database pages1  
- Notion Task not found1  
- Check if creating flag exists  
- Only continue if flag does not exist  
- Set creating flag  
- If (parent task?)  
- Get many database pages (for parent)  
- SwitchT / SwitchT3 (create variants)
- Execution Data7
- Switch by Event1

**Node Details**
- **Get many database pages1 (Notion)**
  - Role: Query Notion DB for existing page with `TodoistID == mapped todoist_id`.
  - Config: `databasePage.getAll`, manual filter, `alwaysOutputData: true`.
  - Failure modes: wrong database_id, property name mismatch (“TodoistID|rich_text” must exist).

- **Sticky Note60**
  - Role: Explains why always output data: uses filter-like query.

- **Notion Task not found1 (IF)**
  - Role: checks if query result is empty.
  - True branch: creation flow + dedupe flags.
  - False branch: existing page path → Execution Data7 → Switch by Event1.

- **Execution Data7**
  - Role: stores `notion_id` from the found page (`$('Notion Task not found1').item.json.id`).
  - Sticky Note42: store more info for debugging.

- **Sticky Note44**
  - Role: explains Redis “creating_for_<TodoistID>” flag prevents duplicate Notion pages.

- **Check if creating flag exists (Redis GET)**
  - Key: `creating_for_<todoist_event_data.id>`
  - Output: `$json.item`

- **Only continue if flag does not exist (Filter)**
  - Passes only if Redis value is empty.

- **Set creating flag (Redis SET)**
  - TTL: 10 seconds.
  - Value: current ISO timestamp.
  - Purpose: suppress parallel webhook executions creating duplicates.

- **If (IF node named “If”)**
  - Role: determines if Todoist task is a **top-level task** vs **subtask**:
    - Condition: `parent_id` notExists.
    - True branch: treat as top-level Task → SwitchT3 create variants.
    - False branch: subtask path → `Get many database pages` (lookup parent Notion page) → SwitchT create variants that include relation.

- **Get many database pages (Notion)**
  - Role: For subtasks, find parent Notion page by `TodoistID == parent_id`.
  - Output used to set relation `properties.Tasks.relation = [{id: $json.id}]` during subtask creation.

- **Switch by Event1 (Switch)**
  - Role: branches by `event_name`:
    - `item:updated` → update properties and/or description handling
    - `item:completed` → completion sync
    - `item:uncompleted` → completion sync
  - Sticky Note54 relevance: event name used to detect deleted events elsewhere.

---

### 1.8 Create Notion page (Task vs Subtask; due date variants)
**Overview:** Creates a Notion database page in different variants depending on whether due start/end exist, and whether it is a task or subtask. Subtasks include a relation to their parent Notion page.

**Nodes Involved:**  
- Sticky Note38, Sticky Note1, Sticky Note2  
- SwitchT3 (Task create variant selector)  
- SwitchT (Subtask create variant selector)  
- Create task in NotionT8 / T7 / T6 (Task variants)  
- Create task in NotionT / T5 / T4 (Subtask variants)  
- Create task in Notion1 (NoOp aggregator)  
- Execution Data6

**Node Details**
- **Sticky Note38**
  - Role: “Create Notion page if it does not exist”.

- **Sticky Note1 / Sticky Note2**
  - Role: label areas “Task” and “Sub Task”.

- **SwitchT3 (Switch)**
  - Role: choose Task creation request based on existence of `start_date` and `end_date`.
  - Rules (interpreting):
    1) start_date notExists → create without Due
    2) end_date notExists → create with Due.start only
    3) end_date exists → create with Due.start and Due.end

- **Create task in NotionT8 / T7 / T6 (HTTP Request to Notion pages endpoint)**
  - Role: POST `https://api.notion.com/v1/pages/` with different property sets.
  - Common properties written:
    - Task Name (title)
    - TodoistID (rich_text)
    - Priority (select)
    - Section (select)
    - Recurrence (rich_text; empty if none)
    - Created (date)
    - Parent database id: `GlobalsT1.database_id`
  - Due handling:
    - T8: no Due at all
    - T7: Due.start
    - T6: Due.start and Due.end
  - Failure modes: Notion validation errors if property types/names mismatch.

- **SwitchT (Switch)**
  - Role: subtask creation variant selector, same due logic, but includes relation `properties.Tasks.relation` using parent Notion page id from `Get many database pages`.
  - Outputs to:
    - Create task in NotionT (no due)
    - Create task in NotionT5 (Due.start)
    - Create task in NotionT4 (Due.start + Due.end)

- **Create task in NotionT / T5 / T4 (HTTP Request)**
  - Role: same as above, plus:
    - `properties.Tasks.relation = [{id: $json.id}]` where `$json.id` is the parent Notion page ID.

- **Create task in Notion1 (NoOp)**
  - Role: Aggregator: all create HTTP nodes flow into this node so later nodes can reference `$('Create task in Notion1').item.json`.
  - Important: downstream expects fields `id` and `url` to exist in this node’s incoming JSON.

- **Execution Data6**
  - Role: stores created `notion_id = $json.id` for debugging.
  - Also triggers `Url` (to build notion:// link) and `Lock Notion ID`.

---

### 1.9 Copy Todoist description into Notion blocks, then overwrite Todoist description with Notion link
**Overview:** On creation and on some updates, the workflow converts Todoist markdown description into Notion block append operations. After copying, it overwrites Todoist description with a Notion deep link so the Todoist task points to the Notion page.

**Nodes Involved:**  
- Sticky Note46, Sticky Note45, Sticky Note33, Sticky Note61, Sticky Note3, Sticky Note47  
- Turn Markdown into Notion Blocks2  
- Handle each block separately2  
- Append Notion Block2  
- Url  
- Update Description in Todoist1  
- Turn Markdown into Notion Blocks3  
- Handle each block separately3  
- Link or Content?1  
- Append Notion Block3  
- Update Description in Todoist6

**Node Details**
- **Turn Markdown into Notion Blocks2 (Code)**
  - Role: Parse full Todoist description into a list of `{type, content}` blocks.
  - Supported markdown:
    - `#`, `##`, `###` headings
    - numbered list (`1.`)
    - bullet list (`- `)
    - paragraphs
  - Splits paragraphs > 2000 chars into chunks to fit Notion limits.
  - Output shape: `{ blocks: [...] }`

- **Handle each block separately2 (Split Out)**
  - Role: splits `blocks` array into separate items for per-block append.

- **Append Notion Block2 (Notion block append)**
  - Role: Appends each block to created page.
  - Block target: `blockId = $('Create task in Notion1').item.json.id`
  - Failure modes: Notion API block append limits; rate limits; invalid block type.

- **Url (Code)**
  - Role: Converts `https://...` Notion URL to `notion://...` deep link.
  - Output: `{ url: modifiedUrl }`

- **Update Description in Todoist1 (HTTP Request)**
  - Role: Updates Todoist task description to markdown link:
    - `description = "[↗ Open in Notion](notion://...)"`.
  - Endpoint: `POST https://api.todoist.com/api/v1/tasks/<id>` with query param `description`.
  - Failure modes: Todoist API change; improper method (Todoist usually PATCH/POST depending on endpoint).

- **Sticky Note45 / Sticky Note46 / Sticky Note33**
  - Roles:
    - Copy description before replacement
    - Overwrite description with Notion URL
    - “Set focussed flag…” note exists but no Focus property update is visible in provided nodes (potentially incomplete feature).

- **Link or Content?1 (Switch)**
  - Role: During updates, checks whether Todoist description has content **after** the first line.
  - If there is extra content → run markdown conversion and append blocks (so the Notion page keeps full content while Todoist keeps only the first line link).

- **Turn Markdown into Notion Blocks3 (Code)**
  - Role: Similar to Blocks2, but **skips the first line** (assumed to be the link line) and converts remaining content.

- **Append Notion Block3 (Notion)**
  - Role: Appends blocks to an existing Notion page.
  - Target blockId: `$('Link or Content?1').item.json.id`
  - **Potential issue:** `Link or Content?1` does not obviously output an `id` field by itself. In many designs, the page id should come from the matched Notion page item. You may need to ensure the item reaching this node contains `id` (the Notion page id).

- **Update Description in Todoist6 (Todoist node)**
  - Role: Sets Todoist description to **first line only**:
    - `description = Get Todoist Task.description.split('\n')[0].trim()`
  - This likely keeps the Notion link line, removing additional markdown lines after syncing to Notion.

---

### 1.10 Update Notion page if differences exist (properties + due variants) and completion sync
**Overview:** When a matching Notion page exists, the workflow compares mapped Todoist fields against Notion properties and updates Notion only if differences exist. It also syncs completion state changes and handles recurring due updates.

**Nodes Involved:**  
- Sticky Note39  
- SwitchT4  
- Differences exist5 / Differences exist3 / Differences exist4  
- Update task in Notion / Update task in Notion4 / Update task in Notion5  
- Create task in Notion Refference1 (NoOp)  
- Require Completion?1  
- Has not been completed?1  
- Mark as Done in Notion1  
- Mark as not Done in Notion1  
- Update Due in Notion when recurring1  
- Sticky Note62, Sticky Note69  
- Lock Notion ID / Lock Todoist ID (lock writes are in next block)

**Node Details**
- **Sticky Note39**
  - Role: “Update Notion page if it needs to be changed”.

- **SwitchT4 (Switch)**
  - Role: Selects which difference-check variant to use based on presence of start/end dates (similar logic to create).
  - Routes to one of Differences exist* nodes.

- **Differences exist5 / 3 / 4 (Filter nodes)**
  - Role: Only pass through if any relevant property differs.
  - Differences exist5 checks: task_name, priority, section_name.
  - Differences exist3 checks: task_name, priority, section_name, start_date, recurrence.
  - Differences exist4 checks: task_name, priority, section_name, end_date, recurrence (but includes a suspicious condition: compares `$json.property_due.start` to itself, which is always equal; likely a mistake and should compare mapped start_date vs notion start).
  - **Edge cases:** Requires the Notion page items to include normalized fields like `$json.property_priority`, `$json.property_section`, `$json.property_due.start/end`, `$json.property_recurrence`. These are not standard Notion node outputs unless you previously mapped them—ensure your Notion “getAll” output structure matches expectations or add a mapping step.

- **Update task in Notion / Update task in Notion4 / Update task in Notion5 (HTTP Request PATCH)**
  - Role: PATCH `https://api.notion.com/v1/pages/<pageId>`
  - Variants:
    - Base update (no Due)
    - Update with Due.start
    - Update with Due.start and Due.end
  - Properties updated: Task Name, TodoistID, Priority, Section, Recurrence (+ Due fields for variants).
  - Failure modes: Notion schema mismatch, rate limits.

- **Create task in Notion Refference1 (NoOp)**
  - Role: Aggregates update flows to a single node so “Lock Notion ID” can run.

- **Require Completion?1 (Switch)**
  - Role: Decides if completion status needs syncing / special recurring handling.
  - Rule 1: if Todoist `checked` != Notion `$json.property_status` (expects that field), then continue to `Has not been completed?1`.
  - Rule 2: if Todoist checked == false → route to `Update Due in Notion when recurring1` (indicates recurring tasks may need due refresh even without completion change).

- **Has not been completed?1 (IF)**
  - Despite name, condition checks boolean `true` on `Get Todoist Task.checked`.
  - True → Mark as Done in Notion1
  - False → Mark as not Done in Notion1

- **Mark as Done in Notion1 / Mark as not Done in Notion1 (Notion update)**
  - Role: toggles `Status|checkbox` property on the Notion page.
  - Note: This assumes a checkbox property named “Status”. Sticky Note6 earlier describes “Status” as a Status-type property, not a checkbox—this is a schema inconsistency. In the provided nodes, it is treated as `Status|checkbox`.
  - You must align Notion database schema to what nodes expect (either change DB or change nodes).

- **Update Due in Notion when recurring1 (HTTP PATCH Notion)**
  - Role: updates only `properties.Due.date.start` to mapped start_date for recurring tasks.

- **Sticky Note62 / Sticky Note69**
  - Role: “Notion ID cached in Redis for 15 seconds…”  
  - In the provided workflow, the actual lock TTL used for “Lock Notion ID” is 100 seconds (not 15). Either the note is outdated or other workflow part is missing.

---

### 1.11 Deletion handling: mark Notion as Obsolete
**Overview:** If Todoist event indicates deletion, the workflow tries to find a matching Notion page by TodoistID and marks its Section as “Obsolete”.

**Nodes Involved:**  
- Sticky Note63, Sticky Note64  
- Get Notion Task  
- Notion Task found  
- Execution Data8  
- Mark as Obsolete in Notion1

**Node Details**
- **Get Notion Task (Notion getAll with limit 1)**
  - Role: Find Notion page where `TodoistID == event_data.id`.
  - `alwaysOutputData: true` so workflow continues even if none found.

- **Notion Task found (Filter)**
  - Role: ensures `$json` is not empty (page exists).

- **Execution Data8**
  - Role: stores `notion_id` and `task_name` for debugging.

- **Mark as Obsolete in Notion1 (Notion update)**
  - Role: sets `Section|select` to “Obsolete”.
  - Assumes “Section” is a select field and includes option “Obsolete”.
  - Failure modes: option missing → Notion validation error.

---

### 1.12 Redis locks written to suppress opposite-direction triggers
**Overview:** After writing to Notion or Todoist, the workflow sets short-lived Redis locks keyed by the affected record ID to prevent the counterpart sync (or re-processing) from triggering back immediately.

**Nodes Involved:**  
- Lock Todoist ID  
- Lock Notion ID  
- Sticky Note62, Sticky Note69, Sticky Note40

**Node Details**
- **Lock Todoist ID (Redis SET)**
  - Key: `lock_<id>` where `<id>` is `$json.id` (from Todoist update nodes)
  - TTL: 100 seconds
  - Value: `SmallExecuted`
  - Used by “Check if Todoist ID is locked1” gate.

- **Lock Notion ID (Redis SET)**
  - Key: `lock_<id>` where `<id>` is `$json.id` (Notion page id)
  - TTL: 100 seconds
  - Value: `Executed`
  - In this workflow JSON, there is no Notion-trigger entrypoint; this lock is likely meant for the opposite direction (Notion → Todoist) in a companion flow.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note6 | Sticky Note | Prerequisites/instructions |  |  | # 0. Start here… (includes Notion template link and credential links) |
| Sticky Note7 | Sticky Note | Label for config generation |  |  | # 1. Generate config JSON for Globals Nodes |
| On form submission | Form Trigger | Start setup helper |  | Get Notion Databases |  |
| Get Notion Databases | Notion | List accessible Notion DBs | On form submission | Prep Dropdown |  |
| Prep Dropdown | Code | Build dropdown values for Notion DB selection | Get Notion Databases | Choose Notion Database |  |
| Choose Notion Database | Form | User selects Notion database | Prep Dropdown | Get Notion Database ID |  |
| Get Notion Database ID | Code | Resolve database_id from selection | Choose Notion Database | Get projects |  |
| Get projects | HTTP Request | List Todoist projects | Get Notion Database ID | Prep Dropdown1 |  |
| Prep Dropdown1 | Code | Build dropdown values for Todoist project selection | Get projects | Choose Todoist Project |  |
| Choose Todoist Project | Form | User selects Todoist project | Prep Dropdown1 | Get Todoist Project ID |  |
| Get Todoist Project ID | Code | Resolve project_id from selection | Choose Todoist Project | Get sections |  |
| Get sections | HTTP Request | List Todoist sections for project | Get Todoist Project ID | Choose Timezone |  |
| Choose Timezone | Form | Ask for IANA timezone | Get sections | Generate config |  |
| Generate config | Code | Build globals JSON | Choose Timezone | Form |  |
| Form | Form (completion) | Display JSON to copy | Generate config |  |  |
| Sticky Note8 | Sticky Note | Label for OAuth activation |  |  | # 2. Activate Todoist Webhook |
| Todoist Webhook Setup Helper | Form Trigger | Collect client credentials |  | Generate security token |  |
| Generate security token | Crypto | Generate OAuth state | Todoist Webhook Setup Helper | Store variables |  |
| Store variables | Code | Persist clientID/secret/state in static data | Generate security token | Redirect to Auth Page |  |
| Redirect to Auth Page | Form (completion redirect) | Redirect to Todoist OAuth | Store variables |  |  |
| OAuth redirect | Webhook (GET) | OAuth callback endpoint |  | Get variables |  |
| Get variables | Code | Read static data for OAuth exchange | OAuth redirect | Verify security token |  |
| Verify security token | IF | Validate OAuth state | Get variables | Exchange Tokens / Respond with error |  |
| Exchange Tokens | HTTP Request | Exchange code for tokens | Verify security token | Respond with success / Respond with error |  |
| Respond with success | Respond to Webhook | OAuth success message | Exchange Tokens |  |  |
| Respond with error | Respond to Webhook | OAuth error message | Verify security token, Exchange Tokens |  |  |
| Sticky Note | Sticky Note | Label: processing webhook data |  |  | #Processing Data Coming from webhooks |
| Sticky Note68 | Sticky Note | Register webhook instructions |  |  | Grab URL and add to Todoist developer app… |
| Sticky Note66 | Sticky Note | Globals setup instructions |  |  | Use Sync Setup Helper Workflow… |
| Todoist Webhook1 | Webhook (POST) | Receive Todoist webhook |  | Todoist Trigger1 | Following nodes reference… (Sticky Note43) |
| Sticky Note43 | Sticky Note | Trigger indirection rationale |  |  | Following nodes reference to this node… |
| Todoist Trigger1 | NoOp | Trigger alias | Todoist Webhook1 | GlobalsT1 |  |
| GlobalsT1 | Set (raw JSON) | Provide config globals | Todoist Trigger1 | Execution Data9 | ## Set Globals… (Sticky Note66) |
| Execution Data9 | Execution Data | Store debug metadata | GlobalsT1 | Check if Todoist ID is locked1 | Store relevant information… (Sticky Note65) |
| Sticky Note65 | Sticky Note | Debug info note |  |  | Store relevant information… |
| Check if Todoist ID is locked1 | Redis (GET) | Loop prevention lock check | Execution Data9 | Only continue if not locked1 | Checking for a cached flag… (Sticky Note40) |
| Sticky Note40 | Sticky Note | Loop prevention rationale |  |  | Checking for a cached flag… |
| Only continue if not locked1 | Filter | Continue only when no lock | Check if Todoist ID is locked1 | If event is not deleted1 |  |
| If event is not deleted1 | IF | Branch deleted vs not deleted | Only continue if not locked1 | Set tries / Get Notion Task | Using event type… (Sticky Note54) |
| Sticky Note54 | Sticky Note | Deletion detection note |  |  | Using the event type… |
| Set tries | Set | Initialize tries counter | If event is not deleted1, If tries left | Get Todoist Task | Todoist webhooks can arrive late… (Sticky Note5) |
| Get Todoist Task | HTTP Request | Fetch fresh task details | Set tries | Map Todoist to Notion1, Catch known error | Todoist webhooks can arrive late… (Sticky Note5) |
| Catch known error | IF | Detect “could not be found” | Get Todoist Task | End here1 / Wait |  |
| Wait | Wait | Delay before retry | Catch known error | Update tries |  |
| Update tries | Set | Increment tries | Wait | If tries left |  |
| If tries left | IF | Retry loop control | Update tries | Set tries / Retry limit reached |  |
| Retry limit reached | Stop and Error | Stop after 3 tries | If tries left |  |  |
| End here1 | NoOp | End branch | Catch known error |  |  |
| Map Todoist to Notion1 | Code | Transform Todoist→Notion model | Get Todoist Task | Execution Data5 | Mapping is more advanced… (Sticky Note55) |
| Sticky Note55 | Sticky Note | Mapping note |  |  | The mapping is more advanced… |
| Execution Data5 | Execution Data | Store task_name (debug) | Map Todoist to Notion1 | Get many database pages1 | Store more information… (Sticky Note24) |
| Sticky Note24 | Sticky Note | Debug info note |  |  | Store more information… |
| Get many database pages1 | Notion | Find Notion page by TodoistID | Execution Data5 | Notion Task not found1 | Retrieve Notion page… (Sticky Note41) |
| Sticky Note41 | Sticky Note | Comparison prep note |  |  | Retrieve Notion page… |
| Notion Task not found1 | IF | Create vs update branch | Get many database pages1 | Check if creating flag exists, Execution Data7 | Previous node always returns… (Sticky Note60) |
| Check if creating flag exists | Redis (GET) | Duplicate creation guard | Notion Task not found1 | Only continue if flag does not exist | Prevent duplicates… (Sticky Note44) |
| Only continue if flag does not exist | Filter | Continue only if no creating flag | Check if creating flag exists | Set creating flag |  |
| Set creating flag | Redis (SET) | Set creating_for_<id> TTL=10 | Only continue if flag does not exist | If |  |
| If | IF | Task vs subtask (parent_id missing) | Set creating flag | SwitchT3 / Get many database pages | ## Task / ## Sub Task |
| Get many database pages | Notion | Find parent Notion page for subtasks | If (false) | SwitchT |  |
| SwitchT3 | Switch | Choose create variant (task) | If (true) | Create task in NotionT8/T7/T6 | **Create Notion page**… (Sticky Note38) |
| SwitchT | Switch | Choose create variant (subtask) | Get many database pages | Create task in NotionT/T5/T4 | **Create Notion page**… (Sticky Note38) |
| Create task in NotionT8 | HTTP Request | Create Notion task (no due) | SwitchT3 | Create task in Notion1 |  |
| Create task in NotionT7 | HTTP Request | Create Notion task (start) | SwitchT3 | Create task in Notion1 |  |
| Create task in NotionT6 | HTTP Request | Create Notion task (start+end) | SwitchT3 | Create task in Notion1 |  |
| Create task in NotionT | HTTP Request | Create Notion subtask (relation, no due) | SwitchT | Create task in Notion1 |  |
| Create task in NotionT5 | HTTP Request | Create Notion subtask (relation, start) | SwitchT | Create task in Notion1 |  |
| Create task in NotionT4 | HTTP Request | Create Notion subtask (relation, start+end) | SwitchT | Create task in Notion1 |  |
| Create task in Notion1 | NoOp | Aggregate create result | Create task in NotionT* | Execution Data6, Turn Markdown into Notion Blocks2 |  |
| Turn Markdown into Notion Blocks2 | Code | Convert description to Notion blocks | Create task in Notion1 | Handle each block separately2 | Copy task description… (Sticky Note46) |
| Handle each block separately2 | Split Out | Iterate blocks | Turn Markdown into Notion Blocks2 | Append Notion Block2 |  |
| Append Notion Block2 | Notion (block) | Append blocks to created page | Handle each block separately2 | Lock Notion ID | Copy task description… (Sticky Note46) |
| Execution Data6 | Execution Data | Store notion_id (debug) | Create task in Notion1 | Url, Lock Notion ID | Store more information… (Sticky Note61) |
| Url | Code | Convert Notion https→notion:// | Execution Data6 | Update Description in Todoist1 | Overwrite description… (Sticky Note45) |
| Update Description in Todoist1 | HTTP Request | Set Todoist description to Notion link | Url | Lock Todoist ID | Overwrite description… (Sticky Note45) |
| Execution Data7 | Execution Data | Store notion_id from existing page | Notion Task not found1 | Switch by Event1 | Store more information… (Sticky Note42) |
| Switch by Event1 | Switch | Branch by Todoist event type | Execution Data7 | Link or Content?1+SwitchT4 / Require Completion?1 |  |
| SwitchT4 | Switch | Choose diff-check variant | Switch by Event1 | Differences exist5/3/4 | **Update Notion page**… (Sticky Note39) |
| Differences exist5 | Filter | Diff check (no due) | SwitchT4 | Update task in Notion |  |
| Differences exist3 | Filter | Diff check (start due) | SwitchT4 | Update task in Notion4 |  |
| Differences exist4 | Filter | Diff check (start+end due) | SwitchT4 | Update task in Notion5 |  |
| Update task in Notion | HTTP Request | Patch Notion properties (no due) | Differences exist5 | Create task in Notion Refference1 |  |
| Update task in Notion4 | HTTP Request | Patch Notion properties (start) | Differences exist3 | Create task in Notion Refference1 |  |
| Update task in Notion5 | HTTP Request | Patch Notion properties (start+end) | Differences exist4 | Create task in Notion Refference1 |  |
| Create task in Notion Refference1 | NoOp | Aggregate update result | Update task in Notion* | Lock Notion ID | Cache Notion ID… (Sticky Note69) |
| Link or Content?1 | Switch | Decide whether extra description lines exist | Switch by Event1 | Turn Markdown into Notion Blocks3 | Copy task description… (Sticky Note47) |
| Turn Markdown into Notion Blocks3 | Code | Convert description excluding first line | Link or Content?1 | Handle each block separately3 | Copy task description… (Sticky Note47) |
| Handle each block separately3 | Split Out | Iterate blocks | Turn Markdown into Notion Blocks3 | Append Notion Block3 |  |
| Append Notion Block3 | Notion (block) | Append blocks to existing page | Handle each block separately3 | Update Description in Todoist6 |  |
| Update Description in Todoist6 | Todoist | Keep only first line in description | Append Notion Block3 | Lock Todoist ID |  |
| Require Completion?1 | Switch | Completion/recurring handling | Switch by Event1 | Has not been completed?1 / Update Due in Notion when recurring1 |  |
| Has not been completed?1 | IF | Checked? then done else not done | Require Completion?1 | Mark as Done in Notion1 / Mark as not Done in Notion1 |  |
| Mark as Done in Notion1 | Notion | Set Status checkbox true | Has not been completed?1 | Lock Notion ID |  |
| Mark as not Done in Notion1 | Notion | Clear Status checkbox | Has not been completed?1 |  |  |
| Update Due in Notion when recurring1 | HTTP Request | Patch Due.start for recurring | Require Completion?1 | Lock Notion ID |  |
| Get Notion Task | Notion | Find page for deletion case | If event is not deleted1 | Notion Task found | **Delete Notion page**… (Sticky Note63) |
| Notion Task found | Filter | Ensure Notion page exists | Get Notion Task | Execution Data8 |  |
| Execution Data8 | Execution Data | Store notion_id and task_name | Notion Task found | Mark as Obsolete in Notion1 | Store more information… (Sticky Note64) |
| Mark as Obsolete in Notion1 | Notion | Set Section select to Obsolete | Execution Data8 | Lock Notion ID | **Delete Notion page**… (Sticky Note63) |
| Lock Todoist ID | Redis (SET) | Write lock_<todoistId> | Update Description in Todoist1, Update Description in Todoist6 |  | Checking for a cached flag… (Sticky Note40) |
| Lock Notion ID | Redis (SET) | Write lock_<notionId> | Append Notion Block2, Execution Data6, Mark as Done, Mark as Obsolete, Create task in Notion Refference1, Update Due… |  | The Notion ID is being cached… (Sticky Note62/69) |
| Sticky Note38 | Sticky Note | Label create block |  |  | **Create Notion page** if it does not exist |
| Sticky Note39 | Sticky Note | Label update block |  |  | **Update Notion page** if it needs to be changed |
| Sticky Note63 | Sticky Note | Label delete/obsolete block |  |  | **Delete Notion page** if Todoist task has been deleted |
| Sticky Note64 | Sticky Note | Debug note |  |  | Store more information… |
| Sticky Note45 | Sticky Note | Description overwrite note |  |  | Overwrite description with Notion URL |
| Sticky Note46 | Sticky Note | Copy description note |  |  | Copy task description initially… |
| Sticky Note47 | Sticky Note | Copy description note (update) |  |  | Copy task description initially… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials in n8n**
   1. Notion credential (integration token) with access to the target database.
   2. Todoist credential (API token or OAuth, depending on your n8n setup).
   3. Redis credential (host/port/password/TLS as required).

2) **Create the Setup Helper (Config generator)**
   1. Add **Form Trigger** node “On form submission” (title “Setup Helper”).
   2. Add **Notion** node “Get Notion Databases” (`Database → Get All`).
   3. Add **Code** node “Prep Dropdown” to build dropdown values from Notion DB list.
   4. Add **Form** node “Choose Notion Database” with dropdown values from `Prep Dropdown`.
   5. Add **Code** node “Get Notion Database ID” to map selected name → database_id.
   6. Add **HTTP Request** “Get projects” (Todoist `GET /api/v1/projects`) using Todoist credentials.
   7. Add **Code** “Prep Dropdown1” to build project dropdown.
   8. Add **Form** “Choose Todoist Project”.
   9. Add **Code** “Get Todoist Project ID”.
   10. Add **HTTP Request** “Get sections” (Todoist `GET /api/v1/sections?project_id=...`).
   11. Add **Form** “Choose Timezone” (text input; include IANA link).
   12. Add **Code** “Generate config” that outputs:
       - database_id, project_id, sections[{Section_id, Section_name}], timezone
   13. Add **Form (completion)** node “Form” to display JSON string of config.

3) **Create the Todoist OAuth Activation Helper**
   1. Add **Form Trigger** “Todoist Webhook Setup Helper” with fields Client ID and Client secret.
   2. Add **Crypto** “Generate security token” action generate, output `state`.
   3. Add **Code** “Store variables” to store clientID/clientSecret/state in workflow static data.
   4. Add **Form (completion redirect)** “Redirect to Auth Page” redirecting to Todoist OAuth authorize URL with scopes and state.
   5. Add **Webhook** “OAuth redirect” (GET, responseMode = responseNode) on path `Todoist-outh`.
   6. Add **Code** “Get variables” to read staticData into the item.
   7. Add **IF** “Verify security token” to compare callback query.state with stored state.
   8. Add **HTTP Request** “Exchange Tokens” `POST https://todoist.com/oauth/access_token` form-urlencoded with client_id/client_secret/code.
   9. Add **Respond to Webhook** nodes “Respond with success” and “Respond with error” and wire them as in the JSON.

4) **Create the main Todoist webhook trigger**
   1. Add **Webhook** “Todoist Webhook1” (POST) path `todoist-webhook`.
   2. Add **NoOp** “Todoist Trigger1” and connect from webhook (this is a replaceable trigger alias).

5) **Add Globals configuration node**
   1. Add **Set** node “GlobalsT1” in **Raw JSON** mode.
   2. Paste the JSON produced by the Setup Helper (database_id, project_id, sections, timezone).
   3. Connect “Todoist Trigger1” → “GlobalsT1”.

6) **Add debug execution data + Redis lock gate**
   1. Add **Execution Data** “Execution Data9” storing `database_id`, `project_id`, `todoist_id`, `source`.
   2. Add **Redis (GET)** “Check if Todoist ID is locked1” key `lock_{{$json.body.event_data.id}}`, output to field `item`.
   3. Add **Filter** “Only continue if not locked1” passing only when `$json.item` is empty.
   4. Add **IF** “If event is not deleted1” where `event_name != item:deleted`.

7) **Not-deleted path: retry loop + fetch fresh task**
   1. Add **Set** “Set tries” initializing `tries`.
   2. Add **HTTP Request** “Get Todoist Task” to `GET /api/v1/tasks/{{ event_data.id }}` with Todoist credentials, `onError=continueErrorOutput`.
   3. Add **IF** “Catch known error” checking `$json.error` contains “could not be found”.
   4. Add **Wait** node and **Set** “Update tries” to increment.
   5. Add **IF** “If tries left” (`tries < 3`) looping back to “Set tries”, else **Stop and Error**.

8) **Map Todoist to Notion**
   1. Add **Code** “Map Todoist to Notion1” implementing the mapping logic (priority/sections/dates/recurrence).
   2. Add **Execution Data** “Execution Data5” to store task_name for debugging.

9) **Find Notion page by TodoistID**
   1. Add **Notion** “Get many database pages1” (`Database Page → Get All`, filter TodoistID equals mapped todoist_id), enable “Always output data”.
   2. Add **IF** “Notion Task not found1” checking if output is empty.

10) **Create path (Notion page does not exist)**
   1. Add **Redis GET** “Check if creating flag exists” key `creating_for_<todoist_id>`.
   2. Add **Filter** “Only continue if flag does not exist”.
   3. Add **Redis SET** “Set creating flag” TTL 10 seconds.
   4. Add **IF** “If” to detect top-level vs subtask via `parent_id`.
   5. For task creation: add **Switch** “SwitchT3” to pick create variant by due presence, then add 3 **HTTP Request** nodes (Notion create page) corresponding to:
      - no due, due.start only, due.start+end
   6. For subtask creation: query parent page (`Get many database pages` by parent_id), then **Switch** “SwitchT” and 3 create HTTP nodes including relation.
   7. Add **NoOp** “Create task in Notion1” as aggregator.
   8. Add **Execution Data6** capturing `notion_id`.
   9. Add description copy pipeline:
      - Code “Turn Markdown into Notion Blocks2” → Split Out → Notion block append “Append Notion Block2”
   10. Add Code “Url” to convert Notion URL to notion:// and HTTP Request “Update Description in Todoist1” to set Todoist description to the Notion link.
   11. Add **Redis SET** locks:
      - “Lock Notion ID” after Notion writes
      - “Lock Todoist ID” after Todoist description update

11) **Update path (Notion page exists)**
   1. Add **Execution Data7** to store notion_id.
   2. Add **Switch** “Switch by Event1” on event_name (updated/completed/uncompleted).
   3. For `item:updated`:
      - Add “Link or Content?1” switch to detect extra description lines, then markdown→blocks pipeline (Blocks3) and “Update Description in Todoist6” (keep first line).
      - Add “SwitchT4” selecting diff-check variants; “Differences exist*” filters; PATCH Notion “Update task in Notion*”.
      - Aggregate with “Create task in Notion Refference1” NoOp and lock Notion ID.
   4. For completion/uncompletion:
      - Add “Require Completion?1” switch and “Has not been completed?1” IF to update Notion checkbox.
      - Add “Update Due in Notion when recurring1” patch if needed.
      - Lock Notion ID after updates.

12) **Deleted path**
   1. Add Notion query “Get Notion Task” by TodoistID (limit 1, alwaysOutputData).
   2. Add Filter “Notion Task found”.
   3. Add Execution Data8.
   4. Add Notion update “Mark as Obsolete in Notion1” (Section select = Obsolete).
   5. Lock Notion ID.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Notion database template referenced by the workflow notes | https://steadfast-banjo-d1f.notion.site/17682b476c848086b002de766879aa71 |
| Redis Cloud free instance | https://redis.io/try-free/ |
| n8n credential docs: Notion | https://docs.n8n.io/integrations/builtin/credentials/notion/ |
| n8n credential docs: Todoist | https://docs.n8n.io/integrations/builtin/credentials/todoist/ |
| n8n credential docs: Redis | https://docs.n8n.io/integrations/builtin/credentials/redis/ |
| IANA timezone list (used in setup helper) | https://www.iana.org/time-zones |

--- 

### Important implementation warnings (based on the provided workflow)
- **Notion “Status” mismatch:** Sticky note says Status-type property with options; nodes update `Status|checkbox`. Align your Notion database schema or adjust the nodes.
- **Potential bug in recurrence/timezone logic:** end time appends `+05:30` explicitly, which conflicts with arbitrary timezones.
- **Append Notion Block3 blockId source looks suspicious:** it references `$('Link or Content?1').item.json.id` though that switch does not inherently produce an `id`. Ensure the Notion page id is present on the item reaching Append Notion Block3.
- **Differences exist4 contains a self-comparison** (`property_due.start` vs itself), effectively a no-op condition; likely meant to compare mapped start_date vs notion start.