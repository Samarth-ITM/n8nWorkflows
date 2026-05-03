Migrate ClickUp list or task tree to Nextcloud Deck as a one-off import

https://n8nworkflows.xyz/workflows/migrate-clickup-list-or-task-tree-to-nextcloud-deck-as-a-one-off-import-14040


# Migrate ClickUp list or task tree to Nextcloud Deck as a one-off import

# 1. Workflow Overview

This workflow performs a **one-off migration from ClickUp to Nextcloud Deck**. It supports two source modes:

- **View mode**: start from a ClickUp **view ID**, discover the underlying home list, then import the parent tasks visible from that list context.
- **Task-root mode**: start from one ClickUp **root task ID**, discover its home list, scan that list, isolate the root task subtree, and import only that subtree.

The migration target is a **single Nextcloud Deck board**. The workflow creates or reuses:

- **Deck stacks** from ClickUp statuses
- **Deck labels** from ClickUp metadata
- **Deck cards** from ClickUp parent tasks
- **Done state** for completed ClickUp tasks

It also enriches card descriptions with:

- task metadata
- ClickUp description
- checklists
- subtasks
- comments

## 1.1 Entry and Configuration

The workflow starts manually and uses one of two configuration nodes. Both branches are normalized into a single config object consumed downstream.

## 1.2 Target Validation

Before doing ClickUp-heavy work, the workflow validates that the Nextcloud Deck board exists and is accessible with the attached HTTP Basic Auth credential.

## 1.3 Source Branch A: View Mode

This branch uses a ClickUp view ID, calls the view endpoint to discover the real home list ID, paginates that list, and normalizes parent tasks into a shared schema.

## 1.4 Source Branch B: Task-Root Mode

This branch fetches a specific root task, discovers its home list, paginates the list, filters all tasks down to the root subtree, and normalizes that subtree into the same shared schema.

## 1.5 Shared Import Preparation

After both branches merge, the workflow builds required stacks and labels, creates them in Deck, then reads the board and stacks back to obtain authoritative IDs.

## 1.6 Card Enrichment and Creation

Prepared tasks are matched to stack IDs and label IDs, comments are fetched and appended to descriptions, and cards are created in Deck.

## 1.7 Post-Creation Updates and Reporting

The workflow assigns labels to created cards, marks completed cards as done, and produces a summary of successes and failures.

---

# 2. Block-by-Block Analysis

## 2.1 Entry, Setup, and Configuration

### Overview
This block starts the workflow manually and selects the active configuration path. Although the workflow contains both source modes, the current trigger is connected to the **task-root** config node by default.

### Nodes Involved
- Manual Trigger
- Set Config - View
- Set Config - Task Root
- Set Config
- Sticky Note
- Sticky Note - Setup & Security

### Node Details

#### Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger`; manual entry point for one-off execution.
- **Configuration choices:** No parameters. It currently feeds `Set Config - Task Root`.
- **Key expressions or variables:** None.
- **Input / output connections:** No input; outputs to `Set Config - Task Root`.
- **Version-specific requirements:** Type version 1.
- **Edge cases / failures:** None technically, but operationally it can trigger the wrong branch if the user forgets to reconnect it for view mode.
- **Sub-workflow reference:** None.

#### Set Config - View
- **Type / role:** `n8n-nodes-base.set`; provides static configuration for view mode.
- **Configuration choices:**
  - `clickup_base_url = https://api.clickup.com`
  - `clickup_list_id` actually stores a **ClickUp view ID**
  - `clickup_api_token`
  - `nextcloud_base_url`
  - `nextcloud_deck_board_id`
  - `max_pages`
  - `status_stack_map_json`
  - `subtask_title_prefix`
  - `source_mode = view`
- **Key expressions or variables:** Static assignments only.
- **Input / output connections:** No required input in logic terms; outputs to `Set Config`.
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**
  - invalid `status_stack_map_json`
  - placeholders left unchanged
  - confusion because `clickup_list_id` is a legacy field name but expects a **view ID**
- **Sub-workflow reference:** None.

#### Set Config - Task Root
- **Type / role:** `n8n-nodes-base.set`; provides static configuration for task-root mode.
- **Configuration choices:**
  - `clickup_base_url`
  - `clickup_api_token`
  - `nextcloud_base_url`
  - `nextcloud_deck_board_id`
  - `max_pages`
  - `status_stack_map_json`
  - `subtask_title_prefix`
  - `clickup_task_id`
  - `source_mode = task_root`
- **Key expressions or variables:** Static assignments only.
- **Input / output connections:** Receives from `Manual Trigger`; outputs to `Set Config`.
- **Version-specific requirements:** Set node v3.4.
- **Edge cases / failures:**
  - placeholders left unchanged
  - invalid root task ID
  - invalid `status_stack_map_json`
- **Sub-workflow reference:** None.

#### Set Config
- **Type / role:** `n8n-nodes-base.code`; pass-through config normalizer.
- **Configuration choices:** Copies incoming JSON unchanged so all downstream nodes can read from `$('Set Config')`.
- **Key expressions or variables:** `return $input.all().map(item => ({ json: { ...(item.json ?? {}) } }));`
- **Input / output connections:** Receives from either config node; outputs to `Deck - Validate Board`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:** Minimal; only malformed incoming item structures would matter.
- **Sub-workflow reference:** None.

#### Sticky Note
- **Type / role:** sticky note; high-level process explanation.
- **Configuration choices:** Documents both source modes and shared behavior.
- **Key expressions or variables:** None.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note v1.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### Sticky Note - Setup & Security
- **Type / role:** sticky note; setup and publication/security guidance.
- **Configuration choices:** Advises replacing placeholders, reconnecting credentials, and validates that no real secrets are embedded in config values.
- **Key expressions or variables:** None.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note v1.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Nextcloud Target Validation and Branch Routing

### Overview
This block validates access to the target Deck board before spending time on ClickUp requests, then routes execution into view mode or task-root mode depending on `source_mode`.

### Nodes Involved
- Deck - Validate Board
- IF - Task Root Mode

### Node Details

#### Deck - Validate Board
- **Type / role:** `n8n-nodes-base.httpRequest`; validates board existence and credentials.
- **Configuration choices:**
  - GET request to:
    `{{ nextcloud_base_url }}/index.php/apps/deck/api/v1.0/boards/{{ nextcloud_deck_board_id }}`
  - Uses generic credential type with `httpBasicAuth`
  - Sends headers:
    - `OCS-APIRequest: true`
    - `Accept: application/json`
- **Key expressions or variables:**
  - `$('Set Config').first().json.nextcloud_base_url`
  - `$('Set Config').first().json.nextcloud_deck_board_id`
- **Input / output connections:** Receives from `Set Config`; outputs to `IF - Task Root Mode`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - wrong base URL
  - wrong board ID
  - missing Deck app / wrong API path
  - invalid or detached HTTP Basic Auth credential
  - TLS/SSL problems
- **Sub-workflow reference:** None.

#### IF - Task Root Mode
- **Type / role:** `n8n-nodes-base.if`; mode switch.
- **Configuration choices:** Checks whether `{{$json.source_mode}} == "task_root"`.
- **Key expressions or variables:** `={{ $json.source_mode }}`
- **Input / output connections:**
  - Input from `Deck - Validate Board`
  - False branch to `Build Page Numbers` (view mode)
  - True branch to `ClickUp - Get Root Task` (task-root mode)
- **Version-specific requirements:** IF node v2.2 with conditions version 2.
- **Edge cases / failures:**
  - wrong `source_mode` value
  - strict comparison means typo or casing mismatch will route unexpectedly
- **Sub-workflow reference:** None.

---

## 2.3 View Mode Branch

### Overview
This branch starts from a ClickUp view ID, discovers the actual home list ID behind the view, paginates the list, and converts parent tasks into the workflow’s common schema for later Deck import.

### Nodes Involved
- Build Page Numbers
- ClickUp - Get View Tasks Page
- Resolve ClickUp Home List ID
- Build List Page Numbers
- ClickUp - Get List Tasks Page
- Normalize ClickUp Parent Tasks - View
- Sticky Note - View Mode Branch

### Node Details

#### Build Page Numbers
- **Type / role:** `n8n-nodes-base.code`; seed item generator for view discovery.
- **Configuration choices:** Emits one item with:
  - `page = 0`
  - `clickup_list_id` from config
- **Key expressions or variables:** Reads `$('Set Config').first().json`.
- **Input / output connections:** Receives from false output of `IF - Task Root Mode`; outputs to `ClickUp - Get View Tasks Page`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:** If config is missing `clickup_list_id`, downstream URL becomes invalid.
- **Sub-workflow reference:** None.

#### ClickUp - Get View Tasks Page
- **Type / role:** `n8n-nodes-base.httpRequest`; calls ClickUp view tasks endpoint.
- **Configuration choices:**
  - GET `{{ clickup_base_url }}/api/v2/view/{{ clickup_list_id }}/task`
  - Query params:
    - `page`
    - `subtasks=true`
    - `include_closed=true`
  - Headers:
    - `Authorization: {{ clickup_api_token }}`
    - `Accept: application/json`
- **Key expressions or variables:**
  - `$('Set Config').first().json.clickup_base_url`
  - `$('Set Config').first().json.clickup_list_id`
  - `$json.page`
- **Input / output connections:** From `Build Page Numbers`; to `Resolve ClickUp Home List ID`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - invalid token
  - wrong view ID
  - rate limiting
  - network timeouts
  - endpoint schema differences
- **Sub-workflow reference:** None.

#### Resolve ClickUp Home List ID
- **Type / role:** `n8n-nodes-base.code`; extracts actual list ID from view results.
- **Configuration choices:** Scans returned tasks for `list.id` or `list_id`; falls back to config value only if it looks numeric.
- **Key expressions or variables:**
  - `cfg.clickup_list_id`
  - produces `clickup_actual_list_id`, `clickup_actual_list_name`, `visible_task_count`
- **Input / output connections:** From `ClickUp - Get View Tasks Page`; to `Build List Page Numbers`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - no tasks in the view, so list cannot be inferred
  - fallback only works when config is numeric
  - malformed response structure
- **Sub-workflow reference:** None.

#### Build List Page Numbers
- **Type / role:** `n8n-nodes-base.code`; pagination fan-out generator.
- **Configuration choices:** Emits one item per page from `0` to `max_pages - 1`.
- **Key expressions or variables:**
  - `cfg.max_pages`
  - `src.clickup_actual_list_id`
- **Input / output connections:** From `Resolve ClickUp Home List ID`; to `ClickUp - Get List Tasks Page`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - missing `clickup_actual_list_id`
  - very high `max_pages` may cause many API calls
- **Sub-workflow reference:** None.

#### ClickUp - Get List Tasks Page
- **Type / role:** `n8n-nodes-base.httpRequest`; fetches paginated tasks from real ClickUp list.
- **Configuration choices:**
  - GET `{{ clickup_base_url }}/api/v2/list/{{ clickup_actual_list_id }}/task`
  - Query params:
    - `page`
    - `subtasks=true`
    - `include_closed=true`
    - `include_timl=true`
  - Authorization header with ClickUp token
- **Key expressions or variables:**
  - `$json.clickup_actual_list_id`
  - `$json.page`
- **Input / output connections:** From `Build List Page Numbers`; to `Normalize ClickUp Parent Tasks - View`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - rate limits
  - list too large for selected `max_pages`
  - invalid token or list access issues
- **Sub-workflow reference:** None.

#### Normalize ClickUp Parent Tasks - View
- **Type / role:** `n8n-nodes-base.code`; transforms ClickUp list tasks into the shared card schema.
- **Configuration choices:**
  - deduplicates tasks across pages
  - keeps only **parent tasks**
  - maps ClickUp statuses to Deck stacks via `status_stack_map_json`
  - builds markdown descriptions
  - derives labels from custom fields and completion state
  - converts dates to ISO-8601
- **Key expressions or variables:**
  - `$('Set Config - View').first().json`
  - parses `cfg.status_stack_map_json`
  - uses `cfg.subtask_title_prefix`
- **Input / output connections:** From `ClickUp - Get List Tasks Page`; to `Normalize ClickUp Parent Tasks`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - invalid `status_stack_map_json`
  - unexpected custom field structures
  - no parent tasks found
  - long titles are truncated to 255 chars
  - date parsing failures return null
- **Sub-workflow reference:** None.

#### Sticky Note - View Mode Branch
- **Type / role:** sticky note; documents the upper branch.
- **Configuration choices:** Explains why view discovery is needed.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note v1.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.4 Task-Root Mode Branch

### Overview
This branch imports only one ClickUp root task tree. It fetches the root task, discovers its home list, scans paged list results, filters descendants, then normalizes the result into the shared schema.

### Nodes Involved
- ClickUp - Get Root Task
- Resolve Root Task Home List ID
- Build Root Task List Page Numbers
- ClickUp - Get Root Task List Tasks Page
- Filter Root Task Tree
- Normalize ClickUp Task Tree
- Sticky Note - Task Root Branch

### Node Details

#### ClickUp - Get Root Task
- **Type / role:** `n8n-nodes-base.httpRequest`; fetches the root task.
- **Configuration choices:**
  - GET `{{ clickup_base_url }}/api/v2/task/{{ clickup_task_id }}`
  - Authorization header using ClickUp token
- **Key expressions or variables:**
  - `$('Set Config').first().json.clickup_task_id`
- **Input / output connections:** From true output of `IF - Task Root Mode`; to `Resolve Root Task Home List ID`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - invalid task ID
  - task not accessible
  - wrong token
- **Sub-workflow reference:** None.

#### Resolve Root Task Home List ID
- **Type / role:** `n8n-nodes-base.code`; extracts root task metadata and home list.
- **Configuration choices:** Produces root task ID, name, URL, and actual list ID/name.
- **Key expressions or variables:** Uses fields like `task.list.id`, `task.id`, `task.url`.
- **Input / output connections:** From `ClickUp - Get Root Task`; to `Build Root Task List Page Numbers`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - root task missing
  - task has no resolvable home list
- **Sub-workflow reference:** None.

#### Build Root Task List Page Numbers
- **Type / role:** `n8n-nodes-base.code`; generates pages to scan the home list.
- **Configuration choices:** Uses `max_pages` to build paginated requests.
- **Key expressions or variables:** `cfg.max_pages`, `src.clickup_actual_list_id`
- **Input / output connections:** From `Resolve Root Task Home List ID`; to `ClickUp - Get Root Task List Tasks Page`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - missing actual list ID
  - too-low `max_pages` can miss descendants
- **Sub-workflow reference:** None.

#### ClickUp - Get Root Task List Tasks Page
- **Type / role:** `n8n-nodes-base.httpRequest`; fetches pages from the root task’s home list.
- **Configuration choices:**
  - GET list tasks endpoint
  - Query params:
    - `page`
    - `subtasks=true`
    - `include_closed=true`
    - `include_timl=true`
- **Key expressions or variables:** `$json.clickup_actual_list_id`, `$json.page`
- **Input / output connections:** From `Build Root Task List Page Numbers`; to `Filter Root Task Tree`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - list pagination incomplete
  - auth/rate-limit problems
- **Sub-workflow reference:** None.

#### Filter Root Task Tree
- **Type / role:** `n8n-nodes-base.code`; isolates the selected root subtree.
- **Configuration choices:**
  - seeds a task map with the fetched root task
  - adds all paged list tasks
  - repeatedly walks parent relations until all descendants are included
  - outputs one payload containing only subtree tasks
- **Key expressions or variables:** references `$('ClickUp - Get Root Task').first().json`
- **Input / output connections:** From `ClickUp - Get Root Task List Tasks Page`; to `Normalize ClickUp Task Tree`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - root task ID missing
  - descendants absent because `max_pages` was too low
  - malformed parent references
- **Sub-workflow reference:** None.

#### Normalize ClickUp Task Tree
- **Type / role:** `n8n-nodes-base.code`; converts subtree tasks into shared schema.
- **Configuration choices:**
  - same general logic as the view normalization
  - if the root task has direct children, those become cards; otherwise the root task itself becomes the card
  - deeper descendants are embedded into each card description as nested subtasks
- **Key expressions or variables:**
  - `$('Set Config').first().json`
  - `$('ClickUp - Get Root Task').first().json`
  - parses `status_stack_map_json`
- **Input / output connections:** From `Filter Root Task Tree`; to `Normalize ClickUp Parent Tasks`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - invalid `status_stack_map_json`
  - no resulting cards
  - unusual task hierarchy structures
- **Sub-workflow reference:** None.

#### Sticky Note - Task Root Branch
- **Type / role:** sticky note; documents the lower branch.
- **Configuration choices:** Explains the branch flow and intended use.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note v1.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.5 Shared Task Merge and Deck Structure Preparation

### Overview
Both source branches join here. The workflow builds the list of required Deck stacks and labels, attempts to create them, and then switches to read-back mode to get authoritative board metadata.

### Nodes Involved
- Normalize ClickUp Parent Tasks
- Build Stack Items
- Deck - Create Stack
- After Stack Creation - Emit Once
- Build Label Items
- Deck - Create Label
- After Label Creation - Emit Once
- Deck - Get Board After Labels
- Deck - Get Stacks
- Sticky Note - Shared Import Pipeline

### Node Details

#### Normalize ClickUp Parent Tasks
- **Type / role:** `n8n-nodes-base.code`; merge point for both branches.
- **Configuration choices:** No transformation; returns all incoming items unchanged.
- **Key expressions or variables:** `return $input.all();`
- **Input / output connections:** From either normalization node; to `Build Stack Items`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:** Minimal.
- **Sub-workflow reference:** None.

#### Build Stack Items
- **Type / role:** `n8n-nodes-base.code`; computes unique target stacks.
- **Configuration choices:**
  - parses `status_stack_map_json`
  - merges configured statuses with statuses actually seen in tasks
  - normalizes keys and sorts by order
- **Key expressions or variables:**
  - `$('Set Config').first().json`
  - `$('Normalize ClickUp Parent Tasks').all()`
- **Input / output connections:** From `Normalize ClickUp Parent Tasks`; to `Deck - Create Stack`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - invalid `status_stack_map_json`
  - duplicate status keys resolved by map overwrite
- **Sub-workflow reference:** None.

#### Deck - Create Stack
- **Type / role:** `n8n-nodes-base.httpRequest`; creates one Deck stack per item.
- **Configuration choices:**
  - POST `/boards/{boardId}/stacks`
  - JSON body with `title` and `order`
  - batch size 1, interval 100 ms
  - `continueOnFail = true`
  - HTTP Basic Auth
- **Key expressions or variables:** `$json.stack_title`, `$json.order`
- **Input / output connections:** From `Build Stack Items`; to `After Stack Creation - Emit Once`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - duplicate stack titles on rerun
  - insufficient permissions
  - invalid board
  - network/API failures
- **Sub-workflow reference:** None.

#### After Stack Creation - Emit Once
- **Type / role:** `n8n-nodes-base.code`; collapses multiple create results into one item.
- **Configuration choices:** Emits `{ stack_create_attempts: count }`.
- **Key expressions or variables:** `$input.all().length`
- **Input / output connections:** From `Deck - Create Stack`; to `Build Label Items`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:** None significant.
- **Sub-workflow reference:** None.

#### Build Label Items
- **Type / role:** `n8n-nodes-base.code`; computes unique labels needed by tasks.
- **Configuration choices:**
  - labels from OKR, Progress, Priority, Finished
  - normalizes colors to Deck-compatible 6-digit hex
- **Key expressions or variables:** reads `deck_labels` from normalized tasks.
- **Input / output connections:** From `After Stack Creation - Emit Once`; to `Deck - Create Label`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - malformed label colors
  - duplicate titles collapse into one
- **Sub-workflow reference:** None.

#### Deck - Create Label
- **Type / role:** `n8n-nodes-base.httpRequest`; creates board labels.
- **Configuration choices:**
  - POST `/boards/{boardId}/labels`
  - body `{ title, color }`
  - batch size 1, interval 100 ms
  - `continueOnFail = true`
- **Key expressions or variables:** `$json.title`, `$json.color`
- **Input / output connections:** From `Build Label Items`; to `After Label Creation - Emit Once`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - duplicate label titles
  - invalid color
  - auth errors
- **Sub-workflow reference:** None.

#### After Label Creation - Emit Once
- **Type / role:** `n8n-nodes-base.code`; collapses label results to one item.
- **Configuration choices:** Emits `{ label_create_attempts: count }`.
- **Key expressions or variables:** `$input.all().length`
- **Input / output connections:** From `Deck - Create Label`; to `Deck - Get Board After Labels`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:** None significant.
- **Sub-workflow reference:** None.

#### Deck - Get Board After Labels
- **Type / role:** `n8n-nodes-base.httpRequest`; authoritative board read-back.
- **Configuration choices:**
  - GET board endpoint
  - relies on board response to retrieve final `labels`
- **Key expressions or variables:** board ID from `Set Config`
- **Input / output connections:** From `After Label Creation - Emit Once`; to `Deck - Get Stacks`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - board inaccessible after create phase
  - response missing expected labels array
- **Sub-workflow reference:** None.

#### Deck - Get Stacks
- **Type / role:** `n8n-nodes-base.httpRequest`; authoritative stack read-back.
- **Configuration choices:** GET `/boards/{boardId}/stacks`
- **Key expressions or variables:** board ID from `Set Config`
- **Input / output connections:** From `Deck - Get Board After Labels`; to `Prepare Task Work Items`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - missing stacks due to failed creates
  - permissions or URL issues
- **Sub-workflow reference:** None.

#### Sticky Note - Shared Import Pipeline
- **Type / role:** sticky note; explains why board/stacks are re-read after creation.
- **Configuration choices:** Documents rerun-safe strategy.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note v1.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

## 2.6 Task Preparation, Comment Enrichment, and Card Creation

### Overview
This block matches normalized tasks to actual Deck stack IDs and label IDs, builds final card payloads, fetches ClickUp comments, appends them to descriptions, and creates Deck cards.

### Nodes Involved
- Prepare Task Work Items
- ClickUp - Get Task Comments
- Append Comments To Task
- Deck - Create Card

### Node Details

#### Prepare Task Work Items
- **Type / role:** `n8n-nodes-base.code`; composes final per-card work items.
- **Configuration choices:**
  - matches `stack_title` to actual Deck stack ID
  - matches label titles to actual Deck label IDs
  - sets stable `order` within each stack
  - forwards description, due date, and metadata
- **Key expressions or variables:**
  - `$('Normalize ClickUp Parent Tasks').all()`
  - `$('Deck - Get Stacks').all()`
  - `$('Deck - Get Board After Labels').first().json`
- **Input / output connections:** From `Deck - Get Stacks`; to `ClickUp - Get Task Comments`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - throws if no stacks are returned
  - throws if expected stack titles are missing after creation
  - label IDs may be missing if label read-back is incomplete
- **Sub-workflow reference:** None.

#### ClickUp - Get Task Comments
- **Type / role:** `n8n-nodes-base.httpRequest`; fetches comments per normalized parent task.
- **Configuration choices:**
  - GET `/api/v2/task/{task_id}/comment`
  - batch size 1, interval 800 ms
  - `continueOnFail = true`
- **Key expressions or variables:** `$json.task_id`
- **Input / output connections:** From `Prepare Task Work Items`; to `Append Comments To Task`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - comments endpoint unavailable
  - insufficient ClickUp permissions
  - rate limiting
  - because `continueOnFail` is true, downstream descriptions may simply omit comments
- **Sub-workflow reference:** None.

#### Append Comments To Task
- **Type / role:** `n8n-nodes-base.code`; merges comment markdown into card descriptions.
- **Configuration choices:**
  - extracts comments from multiple possible payload shapes
  - includes author and ISO date when available
  - appends a `## Comments` section
  - preserves paired item indexing back to prepared tasks
- **Key expressions or variables:**
  - `$('Prepare Task Work Items').all()`
  - `$input.all()`
- **Input / output connections:** From `ClickUp - Get Task Comments`; to `Deck - Create Card`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - unexpected comment payload formats
  - invalid date values
  - pair mismatch if upstream item order changes unexpectedly
- **Sub-workflow reference:** None.

#### Deck - Create Card
- **Type / role:** `n8n-nodes-base.httpRequest`; creates one Deck card per prepared parent task.
- **Configuration choices:**
  - POST `/boards/{boardId}/stacks/{stackId}/cards`
  - body includes:
    - `title`
    - `type: plain`
    - `order`
    - `description`
    - `duedate`
  - batch size 1, interval 100 ms
  - `continueOnFail = true`
- **Key expressions or variables:**
  - `$json.stack_id`
  - `$json.title`
  - `$json.order`
  - `$json.description`
  - `$json.duedate`
- **Input / output connections:**
  - From `Append Comments To Task`
  - To `Prepare Label Assignments`
  - To `Prepare Completed Card Updates`
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - missing stack ID
  - card payload rejected by Deck
  - permission issues
  - network failures
  - failures do not stop downstream summary logic because of `continueOnFail`
- **Sub-workflow reference:** None.

---

## 2.7 Post-Creation Label Assignment, Completion Updates, and Summary

### Overview
After cards are created, the workflow fans out label assignments and done-state updates. It ends with a summary node that reports how many items succeeded or failed.

### Nodes Involved
- Prepare Label Assignments
- Deck - Assign Label
- Prepare Completed Card Updates
- Deck - Mark Card Done
- Result Summary
- Sticky Note - Card Creation & Finishing

### Node Details

#### Prepare Label Assignments
- **Type / role:** `n8n-nodes-base.code`; creates one item per `(card, label)` pair.
- **Configuration choices:**
  - skips failed card creates
  - resolves source item using paired item index
  - carries `task_id`, `card_id`, `stack_id`, `label_id`, `label_title`
- **Key expressions or variables:**
  - `$('Append Comments To Task').all()`
  - `$('Deck - Get Board After Labels').first().json`
- **Input / output connections:** From `Deck - Create Card`; to `Deck - Assign Label`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - missing `stackId` in card response
  - lost paired-item information
  - label IDs absent from prepared source
- **Sub-workflow reference:** None.

#### Deck - Assign Label
- **Type / role:** `n8n-nodes-base.httpRequest`; attaches one label to one Deck card.
- **Configuration choices:**
  - PUT `/boards/{boardId}/stacks/{stackId}/cards/{cardId}/assignLabel`
  - body `{ labelId }`
  - batch size 1, interval 100 ms
  - `continueOnFail = true`
- **Key expressions or variables:** `$json.stack_id`, `$json.card_id`, `$json.label_id`
- **Input / output connections:** From `Prepare Label Assignments`; to `Result Summary`.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - invalid stack/card/label combination
  - duplicate assignment behavior varies by API
  - auth errors
- **Sub-workflow reference:** None.

#### Prepare Completed Card Updates
- **Type / role:** `n8n-nodes-base.code`; prepares PATCH-style update payloads for completed cards.
- **Configuration choices:**
  - only includes tasks where `clickup_is_complete` is true
  - composes full card payload including `done` timestamp
  - attempts to determine `owner` from created card, source, or `nextcloud_username`
- **Key expressions or variables:**
  - `$('Append Comments To Task').all()`
  - `$('Set Config').first().json.nextcloud_username`
- **Input / output connections:** From `Deck - Create Card`; to `Deck - Mark Card Done`.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - config does not define `nextcloud_username`
  - owner may be undefined if Deck response lacks owner
  - failed card creations are skipped
- **Sub-workflow reference:** None.

#### Deck - Mark Card Done
- **Type / role:** `n8n-nodes-base.httpRequest`; updates existing Deck cards with `done` timestamp.
- **Configuration choices:**
  - PUT `/boards/{boardId}/stacks/{stackId}/cards/{cardId}`
  - body includes title, description, type, order, owner, done, and duedate if present
  - batch size 1, interval 100 ms
  - `continueOnFail = true`
- **Key expressions or variables:**
  - `$json.stack_id`
  - `$json.card_id`
  - `$json.done`
  - fallback owner from `$('Set Config').first().json.nextcloud_username`
- **Input / output connections:** From `Prepare Completed Card Updates`; no downstream connection shown.
- **Version-specific requirements:** HTTP Request v4.2.
- **Edge cases / failures:**
  - owner requirement may fail if owner is null
  - wrong card/stack IDs
  - permission issues
- **Sub-workflow reference:** None.

#### Result Summary
- **Type / role:** `n8n-nodes-base.code`; final reporting node.
- **Configuration choices:**
  - counts attempted and successful card creates
  - counts attempted and successful label assignments
  - optionally counts done updates
  - emits a readable summary string plus numeric metrics
- **Key expressions or variables:**
  - `$('Deck - Create Card').all()`
  - `$('Append Comments To Task').all()`
  - `$('Deck - Assign Label').all()`
  - `$('Prepare Completed Card Updates').all()`
  - `$('Deck - Mark Card Done').all()`
  - config board/source IDs
- **Input / output connections:** From `Deck - Assign Label`; final sink.
- **Version-specific requirements:** Code node v2.
- **Edge cases / failures:**
  - if no label assignments exist, summary still works
  - done metrics are wrapped in try/catch, so absent done flow will not crash summary
- **Sub-workflow reference:** None.

#### Sticky Note - Card Creation & Finishing
- **Type / role:** sticky note; documents the final phase.
- **Configuration choices:** Explains card creation, label assignment, done patching, and `continueOnFail` behavior.
- **Input / output connections:** None.
- **Version-specific requirements:** Sticky note v1.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | n8n-nodes-base.manualTrigger | Manual start for one-off migration |  | Set Config - Task Root | Starts the one-off migration manually. Use this workflow for ad hoc imports, not for a recurring sync. Before running, decide which config node you want active: Set Config - View for importing everything visible in a ClickUp view; Set Config - Task Root for importing one root task tree. This trigger currently feeds the task-root config branch. To run in view mode, connect/activate the view config path instead. |
| Set Config - View | n8n-nodes-base.set | Static config for view-mode import |  | Set Config | Configuration for view mode. Fill placeholders. `clickup_list_id` actually expects a ClickUp view ID. `source_mode` must stay `view`.  |
| Set Config - Task Root | n8n-nodes-base.set | Static config for root-task subtree import | Manual Trigger | Set Config | Configuration for task_root mode. Fill placeholders. `clickup_task_id` is the root task to import. `source_mode` must stay `task_root`. |
| Set Config | n8n-nodes-base.code | Normalized shared config output | Set Config - View, Set Config - Task Root | Deck - Validate Board | Pass-through normalizer for the chosen config node. |
| Deck - Validate Board | n8n-nodes-base.httpRequest | Validates Nextcloud Deck board access | Set Config | IF - Task Root Mode | Validates the Nextcloud Deck target before any ClickUp-heavy work starts. |
| IF - Task Root Mode | n8n-nodes-base.if | Routes to view or task-root branch | Deck - Validate Board | Build Page Numbers, ClickUp - Get Root Task | Routes execution by `source_mode`. |
| Build Page Numbers | n8n-nodes-base.code | Seeds view discovery call with page 0 | IF - Task Root Mode | ClickUp - Get View Tasks Page | Upper branch = View mode. Flow: Set Config - View → Set Config → Validate Deck board → IF false branch → seed page 0 on ClickUp view endpoint → resolve the real home list ID → paginate the list → normalize parent tasks into the shared schema. Why this extra discovery step exists: ClickUp views are not the same thing as ClickUp lists. The workflow first uses the view to discover the underlying list, then imports from the list endpoint. |
| ClickUp - Get View Tasks Page | n8n-nodes-base.httpRequest | Queries ClickUp view tasks endpoint for discovery | Build Page Numbers | Resolve ClickUp Home List ID | Upper branch = View mode. Flow: Set Config - View → Set Config → Validate Deck board → IF false branch → seed page 0 on ClickUp view endpoint → resolve the real home list ID → paginate the list → normalize parent tasks into the shared schema. Why this extra discovery step exists: ClickUp views are not the same thing as ClickUp lists. The workflow first uses the view to discover the underlying list, then imports from the list endpoint. |
| Resolve ClickUp Home List ID | n8n-nodes-base.code | Extracts actual ClickUp list ID from view results | ClickUp - Get View Tasks Page | Build List Page Numbers | Upper branch = View mode. Flow: Set Config - View → Set Config → Validate Deck board → IF false branch → seed page 0 on ClickUp view endpoint → resolve the real home list ID → paginate the list → normalize parent tasks into the shared schema. Why this extra discovery step exists: ClickUp views are not the same thing as ClickUp lists. The workflow first uses the view to discover the underlying list, then imports from the list endpoint. |
| Build List Page Numbers | n8n-nodes-base.code | Builds pagination items for real list import | Resolve ClickUp Home List ID | ClickUp - Get List Tasks Page | Upper branch = View mode. Flow: Set Config - View → Set Config → Validate Deck board → IF false branch → seed page 0 on ClickUp view endpoint → resolve the real home list ID → paginate the list → normalize parent tasks into the shared schema. Why this extra discovery step exists: ClickUp views are not the same thing as ClickUp lists. The workflow first uses the view to discover the underlying list, then imports from the list endpoint. |
| ClickUp - Get List Tasks Page | n8n-nodes-base.httpRequest | Fetches paged list tasks from ClickUp | Build List Page Numbers | Normalize ClickUp Parent Tasks - View | Upper branch = View mode. Flow: Set Config - View → Set Config → Validate Deck board → IF false branch → seed page 0 on ClickUp view endpoint → resolve the real home list ID → paginate the list → normalize parent tasks into the shared schema. Why this extra discovery step exists: ClickUp views are not the same thing as ClickUp lists. The workflow first uses the view to discover the underlying list, then imports from the list endpoint. |
| Normalize ClickUp Parent Tasks - View | n8n-nodes-base.code | Normalizes ClickUp list tasks into shared schema | ClickUp - Get List Tasks Page | Normalize ClickUp Parent Tasks | Upper branch = View mode. Flow: Set Config - View → Set Config → Validate Deck board → IF false branch → seed page 0 on ClickUp view endpoint → resolve the real home list ID → paginate the list → normalize parent tasks into the shared schema. Why this extra discovery step exists: ClickUp views are not the same thing as ClickUp lists. The workflow first uses the view to discover the underlying list, then imports from the list endpoint. |
| ClickUp - Get Root Task | n8n-nodes-base.httpRequest | Fetches selected root task | IF - Task Root Mode | Resolve Root Task Home List ID | Lower branch = Task root mode. Flow: Set Config - Task Root → Set Config → Validate Deck board → IF true branch → fetch one root task → resolve its home list → paginate that list → filter down to the root task and all descendants → normalize into the same shared schema as view mode. Use this when you want one project/task tree instead of everything in a ClickUp view or list. |
| Resolve Root Task Home List ID | n8n-nodes-base.code | Extracts root task metadata and home list | ClickUp - Get Root Task | Build Root Task List Page Numbers | Lower branch = Task root mode. Flow: Set Config - Task Root → Set Config → Validate Deck board → IF true branch → fetch one root task → resolve its home list → paginate that list → filter down to the root task and all descendants → normalize into the same shared schema as view mode. Use this when you want one project/task tree instead of everything in a ClickUp view or list. |
| Build Root Task List Page Numbers | n8n-nodes-base.code | Builds pagination items for root task home list scan | Resolve Root Task Home List ID | ClickUp - Get Root Task List Tasks Page | Lower branch = Task root mode. Flow: Set Config - Task Root → Set Config → Validate Deck board → IF true branch → fetch one root task → resolve its home list → paginate that list → filter down to the root task and all descendants → normalize into the same shared schema as view mode. Use this when you want one project/task tree instead of everything in a ClickUp view or list. |
| ClickUp - Get Root Task List Tasks Page | n8n-nodes-base.httpRequest | Fetches paged tasks from root task home list | Build Root Task List Page Numbers | Filter Root Task Tree | Lower branch = Task root mode. Flow: Set Config - Task Root → Set Config → Validate Deck board → IF true branch → fetch one root task → resolve its home list → paginate that list → filter down to the root task and all descendants → normalize into the same shared schema as view mode. Use this when you want one project/task tree instead of everything in a ClickUp view or list. |
| Filter Root Task Tree | n8n-nodes-base.code | Filters paged list tasks down to root subtree | ClickUp - Get Root Task List Tasks Page | Normalize ClickUp Task Tree | Lower branch = Task root mode. Flow: Set Config - Task Root → Set Config → Validate Deck board → IF true branch → fetch one root task → resolve its home list → paginate that list → filter down to the root task and all descendants → normalize into the same shared schema as view mode. Use this when you want one project/task tree instead of everything in a ClickUp view or list. |
| Normalize ClickUp Task Tree | n8n-nodes-base.code | Normalizes subtree tasks into shared schema | Filter Root Task Tree | Normalize ClickUp Parent Tasks | Lower branch = Task root mode. Flow: Set Config - Task Root → Set Config → Validate Deck board → IF true branch → fetch one root task → resolve its home list → paginate that list → filter down to the root task and all descendants → normalize into the same shared schema as view mode. Use this when you want one project/task tree instead of everything in a ClickUp view or list. |
| Normalize ClickUp Parent Tasks | n8n-nodes-base.code | Merge point for normalized tasks from both branches | Normalize ClickUp Parent Tasks - View, Normalize ClickUp Task Tree | Build Stack Items | Merge point for both branches. This node intentionally does no transformation and simply forwards all incoming items. |
| Build Stack Items | n8n-nodes-base.code | Builds unique Deck stack definitions | Normalize ClickUp Parent Tasks | Deck - Create Stack | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Deck - Create Stack | n8n-nodes-base.httpRequest | Creates stacks in Nextcloud Deck | Build Stack Items | After Stack Creation - Emit Once | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| After Stack Creation - Emit Once | n8n-nodes-base.code | Collapses stack-create batch into one item | Deck - Create Stack | Build Label Items | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Build Label Items | n8n-nodes-base.code | Builds unique Deck label definitions | After Stack Creation - Emit Once | Deck - Create Label | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Deck - Create Label | n8n-nodes-base.httpRequest | Creates labels in Nextcloud Deck | Build Label Items | After Label Creation - Emit Once | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| After Label Creation - Emit Once | n8n-nodes-base.code | Collapses label-create batch into one item | Deck - Create Label | Deck - Get Board After Labels | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Deck - Get Board After Labels | n8n-nodes-base.httpRequest | Reads board metadata and labels after creation | After Label Creation - Emit Once | Deck - Get Stacks | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Deck - Get Stacks | n8n-nodes-base.httpRequest | Reads stack metadata after creation | Deck - Get Board After Labels | Prepare Task Work Items | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Prepare Task Work Items | n8n-nodes-base.code | Resolves stack IDs, label IDs, order, and final card inputs | Deck - Get Stacks | ClickUp - Get Task Comments | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| ClickUp - Get Task Comments | n8n-nodes-base.httpRequest | Fetches ClickUp comments per parent task | Prepare Task Work Items | Append Comments To Task | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Append Comments To Task | n8n-nodes-base.code | Appends comment markdown to each card description | ClickUp - Get Task Comments | Deck - Create Card | Shared pipeline after both branches merge. 1. Build unique Deck stacks from mapped statuses 2. Create stacks 3. Build unique Deck labels from normalized tasks 4. Create labels 5. Re-read board and stacks to get authoritative IDs 6. Prepare work items with stack IDs, label IDs, order, due date, and description 7. Fetch ClickUp comments and append them to the card description. Important design choice: The workflow trusts the read-back data from Deck more than the create responses. That makes reruns safer when stacks or labels already exist. |
| Deck - Create Card | n8n-nodes-base.httpRequest | Creates Deck cards from prepared parent tasks | Append Comments To Task | Prepare Label Assignments, Prepare Completed Card Updates | Final phase. Create one Deck card per parent task; fan out label assignments and apply them; patch completed tasks with a `done` timestamp; emit a result summary. Behavior worth knowing: subtasks are not separate cards; they are embedded in the parent card description. Comment fetches, card creation, label assignment, and done updates use `continueOnFail` where appropriate so one bad item does not sink the entire run. |
| Prepare Label Assignments | n8n-nodes-base.code | Builds one assignment item per card-label pair | Deck - Create Card | Deck - Assign Label | Final phase. Create one Deck card per parent task; fan out label assignments and apply them; patch completed tasks with a `done` timestamp; emit a result summary. Behavior worth knowing: subtasks are not separate cards; they are embedded in the parent card description. Comment fetches, card creation, label assignment, and done updates use `continueOnFail` where appropriate so one bad item does not sink the entire run. |
| Deck - Assign Label | n8n-nodes-base.httpRequest | Assigns labels to created cards | Prepare Label Assignments | Result Summary | Final phase. Create one Deck card per parent task; fan out label assignments and apply them; patch completed tasks with a `done` timestamp; emit a result summary. Behavior worth knowing: subtasks are not separate cards; they are embedded in the parent card description. Comment fetches, card creation, label assignment, and done updates use `continueOnFail` where appropriate so one bad item does not sink the entire run. |
| Prepare Completed Card Updates | n8n-nodes-base.code | Builds update payloads for cards that should be marked done | Deck - Create Card | Deck - Mark Card Done | Final phase. Create one Deck card per parent task; fan out label assignments and apply them; patch completed tasks with a `done` timestamp; emit a result summary. Behavior worth knowing: subtasks are not separate cards; they are embedded in the parent card description. Comment fetches, card creation, label assignment, and done updates use `continueOnFail` where appropriate so one bad item does not sink the entire run. |
| Deck - Mark Card Done | n8n-nodes-base.httpRequest | Marks completed cards done in Deck | Prepare Completed Card Updates |  | Final phase. Create one Deck card per parent task; fan out label assignments and apply them; patch completed tasks with a `done` timestamp; emit a result summary. Behavior worth knowing: subtasks are not separate cards; they are embedded in the parent card description. Comment fetches, card creation, label assignment, and done updates use `continueOnFail` where appropriate so one bad item does not sink the entire run. |
| Result Summary | n8n-nodes-base.code | Emits final migration counts and status | Deck - Assign Label |  | Produces a human-readable migration summary. |
| Sticky Note | n8n-nodes-base.stickyNote | Global workflow description |  |  | ClickUp View/List → Nextcloud Deck (one-off import). Supports `view` and `task_root` source modes. Creates stacks, labels, cards, appends comments, and marks completed tasks done. |
| Sticky Note - Setup & Security | n8n-nodes-base.stickyNote | Setup and security guidance |  |  | Setup checklist for community use. Replace placeholders, reconnect credentials, ensure `status_stack_map_json` stays valid JSON. Security audit note: no real tokens/passwords were found in config values. |
| Sticky Note - View Mode Branch | n8n-nodes-base.stickyNote | Documents view-mode branch |  |  | Upper branch = View mode. |
| Sticky Note - Task Root Branch | n8n-nodes-base.stickyNote | Documents task-root branch |  |  | Lower branch = Task root mode. |
| Sticky Note - Shared Import Pipeline | n8n-nodes-base.stickyNote | Documents shared downstream logic |  |  | Shared pipeline after both branches merge. |
| Sticky Note - Card Creation & Finishing | n8n-nodes-base.stickyNote | Documents final phase behavior |  |  | Final phase. |

---

# 4. Reproducing the Workflow from Scratch

Below is a manual rebuild sequence for n8n.

## 4.1 Create the entry and config nodes

1. **Create a `Manual Trigger` node**
   - Name: `Manual Trigger`

2. **Create a `Set` node**
   - Name: `Set Config - View`
   - Add string fields:
     - `clickup_base_url` = `https://api.clickup.com`
     - `clickup_list_id` = your **ClickUp view ID**
     - `clickup_api_token` = your ClickUp API token
     - `nextcloud_base_url` = your Nextcloud base URL
     - `nextcloud_deck_board_id` = your target board ID
     - `max_pages` = `10` or desired limit
     - `status_stack_map_json` = JSON string such as:
       ```json
       [{"status_key":"to do","stack_title":"To do","order":100},{"status_key":"in progress","stack_title":"In progress","order":200},{"status_key":"complete","stack_title":"Complete","order":300}]
       ```
     - `subtask_title_prefix` = `↳ `
     - `source_mode` = `view`

3. **Create another `Set` node**
   - Name: `Set Config - Task Root`
   - Add string fields:
     - `clickup_base_url` = `https://api.clickup.com`
     - `clickup_api_token` = your ClickUp API token
     - `nextcloud_base_url` = your Nextcloud base URL
     - `nextcloud_deck_board_id` = your target board ID
     - `max_pages` = `10`
     - `status_stack_map_json` = same JSON string as above
     - `subtask_title_prefix` = `↳ `
     - `clickup_task_id` = your root task ID
     - `source_mode` = `task_root`

4. **Create a `Code` node**
   - Name: `Set Config`
   - Paste:
     ```javascript
     return $input.all().map(item => ({ json: { ...(item.json ?? {}) } }));
     ```

5. **Connect your active entry path**
   - For task-root mode: `Manual Trigger` → `Set Config - Task Root` → `Set Config`
   - For view mode instead: connect `Manual Trigger` to `Set Config - View`, then to `Set Config`
   - Only one config branch should be active at execution time.

## 4.2 Validate the target board

6. **Create an `HTTP Request` node**
   - Name: `Deck - Validate Board`
   - Method: `GET`
   - URL:
     ```text
     ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id }}
     ```
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Basic Auth`
   - Attach your Nextcloud credential
   - Headers:
     - `OCS-APIRequest: true`
     - `Accept: application/json`

7. **Create an `If` node**
   - Name: `IF - Task Root Mode`
   - Condition:
     - Left value: `={{ $json.source_mode }}`
     - Operation: `equal`
     - Right value: `task_root`

8. **Connect**
   - `Set Config` → `Deck - Validate Board` → `IF - Task Root Mode`

## 4.3 Build the view-mode branch

9. **Create a `Code` node**
   - Name: `Build Page Numbers`
   - Paste:
     ```javascript
     const cfg = $('Set Config').first().json;
     return [{ json: { page: 0, clickup_list_id: cfg.clickup_list_id } }];
     ```

10. **Create an `HTTP Request` node**
    - Name: `ClickUp - Get View Tasks Page`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.clickup_base_url.replace(/\/$/, '') + '/api/v2/view/' + $('Set Config').first().json.clickup_list_id + '/task' }}
      ```
    - Query params:
      - `page = {{ $json.page }}`
      - `subtasks = true`
      - `include_closed = true`
    - Headers:
      - `Authorization = {{ $('Set Config').first().json.clickup_api_token }}`
      - `Accept = application/json`

11. **Create a `Code` node**
    - Name: `Resolve ClickUp Home List ID`
    - Paste the code from the workflow JSON for this node.

12. **Create a `Code` node**
    - Name: `Build List Page Numbers`
    - Paste the code from the workflow JSON for this node.

13. **Create an `HTTP Request` node**
    - Name: `ClickUp - Get List Tasks Page`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.clickup_base_url.replace(/\/$/, '') + '/api/v2/list/' + $json.clickup_actual_list_id + '/task' }}
      ```
    - Query params:
      - `page = {{ $json.page }}`
      - `subtasks = true`
      - `include_closed = true`
      - `include_timl = true`
    - Headers:
      - `Authorization = {{ $('Set Config').first().json.clickup_api_token }}`
      - `Accept = application/json`

14. **Create a `Code` node**
    - Name: `Normalize ClickUp Parent Tasks - View`
    - Paste the full code from the workflow JSON for this node.

15. **Connect the false branch of the IF**
    - `IF - Task Root Mode` false → `Build Page Numbers`
    - `Build Page Numbers` → `ClickUp - Get View Tasks Page`
    - `ClickUp - Get View Tasks Page` → `Resolve ClickUp Home List ID`
    - `Resolve ClickUp Home List ID` → `Build List Page Numbers`
    - `Build List Page Numbers` → `ClickUp - Get List Tasks Page`
    - `ClickUp - Get List Tasks Page` → `Normalize ClickUp Parent Tasks - View`

## 4.4 Build the task-root branch

16. **Create an `HTTP Request` node**
    - Name: `ClickUp - Get Root Task`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.clickup_base_url.replace(/\/$/, '') + '/api/v2/task/' + $('Set Config').first().json.clickup_task_id }}
      ```
    - Headers:
      - `Authorization = {{ $('Set Config').first().json.clickup_api_token }}`
      - `Accept = application/json`

17. **Create a `Code` node**
    - Name: `Resolve Root Task Home List ID`
    - Paste the code from the workflow JSON for this node.

18. **Create a `Code` node**
    - Name: `Build Root Task List Page Numbers`
    - Paste the code from the workflow JSON for this node.

19. **Create an `HTTP Request` node**
    - Name: `ClickUp - Get Root Task List Tasks Page`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.clickup_base_url.replace(/\/$/, '') + '/api/v2/list/' + $json.clickup_actual_list_id + '/task' }}
      ```
    - Query params:
      - `page = {{ $json.page }}`
      - `subtasks = true`
      - `include_closed = true`
      - `include_timl = true`
    - Headers:
      - `Authorization = {{ $('Set Config').first().json.clickup_api_token }}`
      - `Accept = application/json`

20. **Create a `Code` node**
    - Name: `Filter Root Task Tree`
    - Paste the code from the workflow JSON for this node.

21. **Create a `Code` node**
    - Name: `Normalize ClickUp Task Tree`
    - Paste the code from the workflow JSON for this node.

22. **Connect the true branch of the IF**
    - `IF - Task Root Mode` true → `ClickUp - Get Root Task`
    - `ClickUp - Get Root Task` → `Resolve Root Task Home List ID`
    - `Resolve Root Task Home List ID` → `Build Root Task List Page Numbers`
    - `Build Root Task List Page Numbers` → `ClickUp - Get Root Task List Tasks Page`
    - `ClickUp - Get Root Task List Tasks Page` → `Filter Root Task Tree`
    - `Filter Root Task Tree` → `Normalize ClickUp Task Tree`

## 4.5 Merge both branches into shared normalized task stream

23. **Create a `Code` node**
    - Name: `Normalize ClickUp Parent Tasks`
    - Paste:
      ```javascript
      return $input.all();
      ```

24. **Connect both branch outputs**
    - `Normalize ClickUp Parent Tasks - View` → `Normalize ClickUp Parent Tasks`
    - `Normalize ClickUp Task Tree` → `Normalize ClickUp Parent Tasks`

## 4.6 Create Deck stacks

25. **Create a `Code` node**
    - Name: `Build Stack Items`
    - Paste the code from the workflow JSON.

26. **Create an `HTTP Request` node**
    - Name: `Deck - Create Stack`
    - Method: `POST`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id + '/stacks' }}
      ```
    - Body content type: `Raw`
    - Raw content type: `application/json`
    - Body:
      ```text
      ={{ JSON.stringify({ title: $json.stack_title, order: Number($json.order || 999) }) }}
      ```
    - Headers:
      - `OCS-APIRequest: true`
      - `Accept: application/json`
      - `Content-Type: application/json`
    - Authentication: HTTP Basic Auth
    - Batching:
      - batch size = 1
      - interval = 100 ms
    - Enable `Continue On Fail`

27. **Create a `Code` node**
    - Name: `After Stack Creation - Emit Once`
    - Paste:
      ```javascript
      return [{
        json: {
          stack_create_attempts: $input.all().length
        }
      }];
      ```

28. **Connect**
    - `Normalize ClickUp Parent Tasks` → `Build Stack Items` → `Deck - Create Stack` → `After Stack Creation - Emit Once`

## 4.7 Create Deck labels

29. **Create a `Code` node**
    - Name: `Build Label Items`
    - Paste the code from the workflow JSON.

30. **Create an `HTTP Request` node**
    - Name: `Deck - Create Label`
    - Method: `POST`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id + '/labels' }}
      ```
    - Body:
      ```text
      ={{ JSON.stringify({ title: $json.title, color: $json.color || '808080' }) }}
      ```
    - Raw JSON body, same headers and auth as stack creation
    - Batching: size 1, interval 100 ms
    - Enable `Continue On Fail`

31. **Create a `Code` node**
    - Name: `After Label Creation - Emit Once`
    - Paste:
      ```javascript
      return [{ json: { label_create_attempts: $input.all().length } }];
      ```

32. **Connect**
    - `After Stack Creation - Emit Once` → `Build Label Items` → `Deck - Create Label` → `After Label Creation - Emit Once`

## 4.8 Read authoritative Deck metadata

33. **Create an `HTTP Request` node**
    - Name: `Deck - Get Board After Labels`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id }}
      ```
    - Headers:
      - `OCS-APIRequest: true`
      - `Accept: application/json`
    - Authentication: HTTP Basic Auth

34. **Create an `HTTP Request` node**
    - Name: `Deck - Get Stacks`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id + '/stacks' }}
      ```
    - Same headers and auth

35. **Connect**
    - `After Label Creation - Emit Once` → `Deck - Get Board After Labels` → `Deck - Get Stacks`

## 4.9 Prepare cards and append comments

36. **Create a `Code` node**
    - Name: `Prepare Task Work Items`
    - Paste the code from the workflow JSON.

37. **Create an `HTTP Request` node**
    - Name: `ClickUp - Get Task Comments`
    - Method: `GET`
    - URL:
      ```text
      ={{ $('Set Config').first().json.clickup_base_url.replace(/\/$/, '') + '/api/v2/task/' + $json.task_id + '/comment' }}
      ```
    - Headers:
      - `Authorization = {{ $('Set Config').first().json.clickup_api_token }}`
      - `Accept = application/json`
    - Batching:
      - size 1
      - interval 800 ms
    - Enable `Continue On Fail`

38. **Create a `Code` node**
    - Name: `Append Comments To Task`
    - Paste the code from the workflow JSON.

39. **Connect**
    - `Deck - Get Stacks` → `Prepare Task Work Items` → `ClickUp - Get Task Comments` → `Append Comments To Task`

## 4.10 Create cards

40. **Create an `HTTP Request` node**
    - Name: `Deck - Create Card`
    - Method: `POST`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id + '/stacks/' + $json.stack_id + '/cards' }}
      ```
    - Body:
      ```text
      ={{ JSON.stringify({ title: $json.title, type: 'plain', order: Number($json.order || 999), description: $json.description || '', duedate: $json.duedate || null }) }}
      ```
    - Raw JSON, same Deck auth/headers
    - Batching size 1, interval 100 ms
    - Enable `Continue On Fail`

41. **Connect**
    - `Append Comments To Task` → `Deck - Create Card`

## 4.11 Assign labels to created cards

42. **Create a `Code` node**
    - Name: `Prepare Label Assignments`
    - Paste the code from the workflow JSON.

43. **Create an `HTTP Request` node**
    - Name: `Deck - Assign Label`
    - Method: `PUT`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id + '/stacks/' + $json.stack_id + '/cards/' + $json.card_id + '/assignLabel' }}
      ```
    - Body:
      ```text
      ={{ JSON.stringify({ labelId: Number($json.label_id) }) }}
      ```
    - Raw JSON, same auth/headers
    - Batching size 1, interval 100 ms
    - Enable `Continue On Fail`

44. **Connect**
    - `Deck - Create Card` → `Prepare Label Assignments` → `Deck - Assign Label`

## 4.12 Mark completed cards done

45. **Create a `Code` node**
    - Name: `Prepare Completed Card Updates`
    - Paste the code from the workflow JSON.
    - Optional improvement: add `nextcloud_username` to the config nodes if your Deck API requires it and card create responses do not include owner.

46. **Create an `HTTP Request` node**
    - Name: `Deck - Mark Card Done`
    - Method: `PUT`
    - URL:
      ```text
      ={{ $('Set Config').first().json.nextcloud_base_url.replace(/\/$/, '') + '/index.php/apps/deck/api/v1.0/boards/' + $('Set Config').first().json.nextcloud_deck_board_id + '/stacks/' + $json.stack_id + '/cards/' + $json.card_id }}
      ```
    - Body: use the same payload logic as in the workflow JSON
    - Raw JSON, same auth/headers
    - Batching size 1, interval 100 ms
    - Enable `Continue On Fail`

47. **Connect**
    - `Deck - Create Card` → `Prepare Completed Card Updates` → `Deck - Mark Card Done`

## 4.13 Add final reporting

48. **Create a `Code` node**
    - Name: `Result Summary`
    - Paste the code from the workflow JSON.

49. **Connect**
    - `Deck - Assign Label` → `Result Summary`

## 4.14 Add sticky notes if desired

50. Add sticky notes for:
    - overall workflow description
    - setup/security checklist
    - view branch explanation
    - task-root branch explanation
    - shared import pipeline
    - card creation and finishing phase

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is designed for one-off imports, not recurring synchronization. | Operational note |
| In view mode, the field named `clickup_list_id` actually expects a ClickUp **view ID**. | Important configuration caveat |
| The workflow prefers reading Deck board/stacks back after creation instead of trusting create responses. This makes reruns safer. | Design principle |
| Subtasks are not created as separate Deck cards. They are embedded into parent card descriptions. | Migration behavior |
| ClickUp comments are appended after descriptions, checklists, and subtasks. | Card content behavior |
| `status_stack_map_json` must remain valid JSON. | Configuration requirement |
| Reattach the same Nextcloud HTTP Basic Auth credential to all Deck HTTP nodes after importing the workflow into another n8n instance. | Credential setup |
| A fallback `nextcloud_username` is referenced in done-update logic, but it is not defined in the current config nodes. Add it if your Deck API requires explicit owner values. | Compatibility note |
| The exported workflow contains credential references and metadata, not embedded secret values. | Security note |