Automated employee offboarding: lock Redmine & GitLab accounts using Odoo 18

https://n8nworkflows.xyz/workflows/automated-employee-offboarding--lock-redmine---gitlab-accounts-using-odoo-18-11941


# Automated employee offboarding: lock Redmine & GitLab accounts using Odoo 18

## 1. Workflow Overview

**Purpose:**  
This workflow automates employee offboarding by detecting employees whose **expected leaving date is today** (from **Odoo 18** resignations) and then automatically:
- **Locks the user in Redmine 6**, removes them from all groups, and removes project memberships
- **Blocks the user in GitLab** (if found and not already blocked/banned)

**Primary use case:** IT Ops / HR / Security teams wanting same-day access removal after an employee’s last shift/day.

### 1.1 Scheduling & Date Preparation (Steps 1–3)
Runs on a schedule (weekdays at 17:00) and prepares date context + base variables (URLs, API endpoints, limits).

### 1.2 Retrieve Resignations from Odoo (Step 4–5)
Calls Odoo `hr.resignation` search/read endpoint and exits early if no data.

### 1.3 Filter Resignations for “Leaving Today” (Step 6–7)
Removes draft/canceled resignations and keeps only records whose `expected_revealing_date` equals today; if none match, end.

### 1.4 Per-Employee Loop: Fetch Odoo Employee → Identify Accounts (Step 8–12)
Iterates employee IDs, fetches `hr.employee` to obtain `work_email`, derives usernames, and searches Redmine for a matching user.

### 1.5 Redmine 6 Lockdown (Step 13–18 + Step 17 loop)
If exactly one Redmine user is found: lock them, remove all groups, fetch memberships, and delete each membership.

### 1.6 GitLab Blocking (Step 19–23 and Step 24–28)
Looks up the user in GitLab by derived username and blocks them if not already blocked or banned. Two nearly identical GitLab branches exist (see notes in Block 6).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Base Variables (Steps 1–3)
**Overview:** Triggers the workflow on weekdays at 17:00, computes the current date/day-of-week, and defines base configuration variables used across HTTP nodes.

**Nodes involved:**
- `Step1: Schedule Trigger`
- `Step2: Handle get today`
- `Step3: Set Variables`

#### Node: Step1: Schedule Trigger
- **Type / Role:** Schedule Trigger; entry point
- **Config:** Runs weekly on days **Mon–Fri** at **17:00** (minute 0)
- **Output:** Emits an empty item to start the workflow
- **Edge cases:** Instance timezone affects “17:00”; ensure n8n timezone matches intended operations time.

#### Node: Step2: Handle get today
- **Type / Role:** Code; compute date metadata
- **Config choices:** JavaScript builds:
  - `date` formatted as `YYYY-MM-DD`
  - `dayOfWeek` label
- **Important detail:** `getDayOfWeek()` uses `date.getDay()` but its `days` array starts at `"Monday"`. In JavaScript, `getDay()` returns `0 = Sunday`, so the day label is **shifted** (Sunday becomes “Monday”, etc.).  
  This doesn’t appear used downstream, but it is technically incorrect.
- **Output:** `{ date, dayOfWeek }`
- **Edge cases:** None critical unless dayOfWeek is later used.

#### Node: Step3: Set Variables
- **Type / Role:** Set; centralized configuration
- **Config:** Raw JSON output defines:
  - `redmine6_Url` (base URL, expected to end with `/`)
  - `odooUrl` (not used directly in nodes shown)
  - `limit` (used in Odoo resignation query)
  - `web_search_read_api` (Odoo JSON-RPC endpoint for `web_search_read`)
  - `web_read_api` (Odoo JSON-RPC endpoint for `web_read`)
  - `gitlab_Url` (base URL, expected to end with `/`)
- **Edge cases:** If base URLs are missing trailing slashes, concatenated endpoints may break (e.g., `...Url}}users.json`).

---

### Block 2 — Query Odoo Resignations & Early Exit (Steps 4–5)
**Overview:** Pulls resignation records from Odoo 18 and exits immediately if no results.

**Nodes involved:**
- `Step4: Get Employee Resignation in Odoo 18`
- `Step5: If total_count is not equal to 0 = true`
- `END`

#### Node: Step4: Get Employee Resignation in Odoo 18
- **Type / Role:** HTTP Request; Odoo JSON-RPC call
- **Authentication:** `httpHeaderAuth` (sticky note says “Use APIkey Odoo 18”)
- **Method/Body:** POST JSON-RPC `web_search_read` on model `hr.resignation`
- **Key parameters:**
  - `limit: {{ Step3.limit }}`
  - `domain`: `state in [draft, confirm, manager_approved, handed_over, cancel]`
  - Returns fields including `employee_id`, `expected_revealing_date`, `state`, etc.
- **Output shape (important downstream):**
  - Uses `result.records` later in Step6
  - Uses `result.length` in Step5 (potential mismatch; see below)
- **Failure modes:**
  - 401/403 if API key invalid
  - 404 if endpoint is wrong (`web_search_read_api`)
  - Odoo JSON-RPC errors in response payload even if HTTP 200

#### Node: Step5: If total_count is not equal to 0 = true
- **Type / Role:** If; early exit gate
- **Condition:** `{{ $json.result.length }} != 0`
- **Potential issue:** Odoo `web_search_read` often returns `{ records: [...], length: <count> }` (or similar). This workflow later reads `result.records`. If Odoo returns `result.length` the condition works; if it returns `result.records.length` instead, the check may misbehave.
- **True path:** continues to Step6  
- **False path:** goes to `END`
- **Edge cases:** If `$json.result` is undefined due to an Odoo error payload, expression evaluation can fail.

#### Node: END
- **Type / Role:** NoOp; terminator for “no resignations found”
- **Connections:** None beyond end

---

### Block 3 — Filter “Leaving Today” Employees (Steps 6–7)
**Overview:** Filters resignations to those effective today and not in `draft` or `cancel`, producing a list of `{id: employeeId}` items for iteration.

**Nodes involved:**
- `Step6: Check if there is a valid record.`
- `Step7: If hasRecords is empty==> true`
- `END1`

#### Node: Step6: Check if there is a valid record.
- **Type / Role:** Code; filter/transform
- **Logic:**
  - Reads `records = inputData[0].json.result.records`
  - `today = new Date().toISOString().split('T')[0]` (UTC date)
  - Filters out `state === "draft"` and `state === "cancel"`
  - Keeps only `record.expected_revealing_date === today`
  - Outputs `[{ id: record.employee_id.id }, ...]`
- **Key risk:** `toISOString()` uses **UTC** date. If your business day is Asia/Bangkok, after local midnight the UTC date may still be “yesterday”, causing off-by-one-day filtering.
- **alwaysOutputData:** true (so the node outputs even if empty array—helpful for Step7)
- **Failure modes:** If Odoo response doesn’t include `result.records`, code throws.

#### Node: Step7: If hasRecords is empty==> true
- **Type / Role:** If; emptiness gate
- **Condition:** checks `{{ $json }}` is **empty**
- **Interpretation:** If Step6 returns no items, route to `END1`; else continue to loop.
- **Edge cases:** This pattern works because Step6 returns an array of items; but “empty” checks depend on n8n’s item context. If Step6 outputs a single item with empty object instead of no items, logic changes.

#### Node: END1
- **Type / Role:** NoOp; terminator for “no one leaves today”
- **Connections:** Ends the run

---

### Block 4 — Per-Employee Loop: Fetch Employee & Derive Identity (Steps 8–10)
**Overview:** Iterates each employee ID, fetches Odoo employee details (work email), and derives `work_email` local-part and a simplified GitLab “username”.

**Nodes involved:**
- `Step8: Loop Over Items`
- `Step9: Get employee information in Odoo 18`
- `Step10: Handle work_email`

#### Node: Step8: Loop Over Items
- **Type / Role:** SplitInBatches; batching/loop controller
- **Config:** default options (batch size defaults to 1 unless set in UI)
- **Connections:**
  - **Output 1:** unused in this workflow
  - **Output 2:** drives Step9
- **Looping behavior:** Later nodes route back to `Step8` after GitLab blocking to process next employee.
- **Edge cases:** If many employees, ensure batch size and rate limits for Odoo/Redmine/GitLab.

#### Node: Step9: Get employee information in Odoo 18
- **Type / Role:** HTTP Request; Odoo JSON-RPC `web_read`
- **Authentication:** `httpHeaderAuth`
- **Method/Body:** POST, model `hr.employee`, method `web_read`, args `[[ {{ $json.id }} ]]`
- **Fields requested:** `active`, `user_id.display_name`, `work_email`
- **Failure modes:**
  - Missing permissions: cannot read employee data
  - Employee has no work_email → Step10 will fail

#### Node: Step10: Handle work_email
- **Type / Role:** Code; normalize identifiers
- **Logic:**
  - `work_email = inputData[0].json.result[0].work_email.split('@')[0];`
  - `username = work_email.split('.')[0];`
  - returns `{ work_email, username }`
- **Important:** variable name `work_email` actually becomes the **local part** (before `@`), not the full email.
- **Edge cases / failures:**
  - If `result[0].work_email` is null/empty → `.split` throws
  - `username` takes only the segment before the first dot; may not match GitLab user naming conventions (e.g., `john.doe` becomes `john`).

---

### Block 5 — Redmine 6 User Lookup & Lockdown (Steps 11–18 + membership loop)
**Overview:** Searches Redmine by name/email-derived value. If exactly one match, locks the user, removes all groups, fetches memberships, and deletes them one by one.

**Nodes involved:**
- `Step11: Get user info in RM6`
- `Step12:  Check if there is a record or not?`
- `Step13: Update the user's status to "locked" on RM6`
- `Step14: Remove the user from all groups in RM 6`
- `Step15: Get membership list of user in RM 6`
- `Step16: Get memberships_id`
- `Step17: Loop Over memberships_id`
- `Step18: Delete membership in project on RM 6`
- `Code` (loop helper)

#### Node: Step11: Get user info in RM6
- **Type / Role:** HTTP Request; Redmine user search
- **Authentication:** `httpHeaderAuth` (likely `X-Redmine-API-Key`)
- **URL:** `{{ redmine6_Url }}users.json?name={{ Step10.work_email }}`
- **executeOnce:** true (may be unintended in loops; see below)
- **Edge cases:**
  - `executeOnce: true` means it may only execute once per workflow execution depending on n8n behavior/version, which can break per-employee lookups.
  - Redmine search may return multiple users; workflow only handles exactly 1.

#### Node: Step12:  Check if there is a record or not?
- **Type / Role:** If; ensure single match
- **Condition:** `{{ $json.total_count }} == 1`
- **True path:** proceed with Redmine lock flow (Step13...)
- **False path:** jumps to GitLab branch (`Step24: Get user information in Gitlab`)
- **Edge cases:** If `total_count` is 0 or >1, Redmine is skipped entirely.

#### Node: Step13: Update the user's status to "locked" on RM6
- **Type / Role:** HTTP Request; Redmine user update
- **Method:** PUT
- **URL:** `...users/{{ Step12.json.users[0].id }}.json`
- **Body:** `{"user": {"status": 3}}` (Redmine: 3 = locked)
- **Failure modes:** insufficient admin rights; user not found; CSRF not relevant for API-key calls.

#### Node: Step14: Remove the user from all groups in RM 6
- **Type / Role:** HTTP Request; Redmine user update
- **Method:** PUT
- **Body:** `{ "user": { "group_ids": [] } }`
- **executeOnce:** true (same risk as Step11 in loop contexts)
- **Note:** Removing group IDs depends on Redmine API allowing full replacement.
- **Failure modes:** permission errors; Redmine may require including other fields depending on configuration.

#### Node: Step15: Get membership list of user in RM 6
- **Type / Role:** HTTP Request; fetch user with memberships
- **URL:** `...users/{id}.json?include=memberships`
- **executeOnce:** true (risk in loops)
- **Output expectation:** `json.user.memberships` array exists.

#### Node: Step16: Get memberships_id
- **Type / Role:** Code; map memberships to IDs
- **Logic:** reads `inputData[0].json.user.memberships` → outputs `[{ id }, ...]`
- **alwaysOutputData:** true
- **Edge cases:** if user has no memberships, `membershipsProject` may be undefined → code throws unless Redmine returns empty array.

#### Node: Step17: Loop Over memberships_id
- **Type / Role:** SplitInBatches; iterates membership IDs
- **Connections:**
  - Output 2 → `Step18` (delete)
  - Output 1 → `Code` (then to GitLab Step19)
- **Behavior:** Standard pattern: delete each membership, then loop until done, then continue.

#### Node: Step18: Delete membership in project on RM 6
- **Type / Role:** HTTP Request; delete membership
- **Method:** DELETE
- **URL:** `...memberships/{{ $json.id }}.json`
- **onError:** `continueRegularOutput` (won’t stop flow on delete errors)
- **executeOnce:** true (high risk: deletion should run per membership)
- **alwaysOutputData:** true
- **Edge cases:** membership already deleted; permission issues; rate limiting.

#### Node: Code
- **Type / Role:** Code; emits `{}` to continue flow
- **Purpose here:** acts as a simple pass-through after Step17 completes, then triggers GitLab lookup (`Step19`)
- **Output:** `[{}]`

---

### Block 6 — GitLab User Lookup & Blocking (Steps 19–23 and Steps 24–28)
**Overview:** Searches GitLab users with `search=<username>`, checks whether the user is already blocked/banned, and blocks them otherwise. There are **two parallel/duplicated GitLab flows**: one triggered after Redmine processing completion (Step19 path) and one triggered when Redmine does not have exactly one match (Step24 path).

**Nodes involved (branch A):**
- `Step19: Get user information in Gitlab`
- `Step20:  Check if there is a record = true`
- `Step21: Check if the user is not blocked = true`
- `Step22: Check if the user is not deactivate = true`
- `Step23: Block a user in Gitlab`

**Nodes involved (branch B):**
- `Step24: Get user information in Gitlab`
- `Step25:  Check if there is a record = true`
- `Step26: Check if the user is not blocked = true`
- `Step27: Check if the user is not deactivate = true`
- `Step28: Block a user in Gitlab`

#### Node: Step19 / Step24: Get user information in Gitlab
- **Type / Role:** HTTP Request; GitLab user search
- **Authentication:** `httpHeaderAuth` (typically `PRIVATE-TOKEN: <token>`)
- **URL:** `{{ gitlab_Url }}api/v4/users?search={{ Step10.username }}`
- **alwaysOutputData:** true
- **Edge cases / correctness:**
  - GitLab returns an **array** of users. This workflow later reads `.json.state` as if it were an object, which is likely incorrect unless n8n transforms it or GitLab returns a single object (it doesn’t in standard API).
  - You likely need `{{$json[0].id}}` and `{{$json[0].state}}` (first match) or a filter step.

#### Node: Step20 / Step25: Check if there is a record = true
- **Type / Role:** If; existence gate
- **Condition:** `notEmpty({{ Step19.json }})` (or Step24.json)
- **True path:** continues checks
- **False path:** routes back to `Step8` (next employee)
- **Edge cases:** “notEmpty” on an array works, but downstream “id/state” extraction is still problematic.

#### Node: Step21 / Step26: Check if the user is not blocked = true
- **Type / Role:** If; state check
- **Condition:** `{{ Step19.json.state }} != blocked` (and similarly for Step24)
- **Problem:** rightValue is `=blocked` (includes `=`), so it compares against the literal string `=blocked`. Same issue in banned checks.
- **Problem #2:** if Step19.json is an array, `.state` is undefined.
- **False path:** routes back to `Step8`.

#### Node: Step22 / Step27: Check if the user is not deactivate = true
- **Type / Role:** If; banned check
- **Condition:** `{{ Step19.json.state }} != banned` (but configured as `=banned`)
- **False path:** routes back to `Step8`.

#### Node: Step23 / Step28: Block a user in Gitlab
- **Type / Role:** HTTP Request; block user
- **Method:** POST
- **URL:** `.../api/v4/users/{{ Step20.json.id }}/block` (or Step25.json.id)
- **Problem:** Step20 is an If node; its output item is the same payload as input, not a selected user object. If GitLab search returns an array, `id` won’t exist at the top level.
- **Output:** alwaysOutputData true; then routes back to `Step8` to continue loop.
- **Failure modes:** requires admin-level token; may 404 if wrong ID; may 409 if already blocked.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Step1: Schedule Trigger | scheduleTrigger | Start workflow on weekday schedule | — | Step2: Handle get today | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step2: Handle get today | code | Build today date & weekday | Step1: Schedule Trigger | Step3: Set Variables | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step3: Set Variables | set | Define base URLs/endpoints/limit | Step2: Handle get today | Step4: Get Employee Resignation in Odoo 18 | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step4: Get Employee Resignation in Odoo 18 | httpRequest | Query Odoo resignations (web_search_read) | Step3: Set Variables | Step5: If total_count is not equal to 0 = true | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step5: If total_count is not equal to 0 = true | if | Exit early if no resignations | Step4: Get Employee Resignation in Odoo 18 | Step6: Check if there is a valid record.; END | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step6: Check if there is a valid record. | code | Filter resignations to those leaving today | Step5 (true) | Step7: If hasRecords is empty==> true | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step7: If hasRecords is empty==> true | if | End if no employees match today | Step6: Check if there is a valid record. | END1; Step8: Loop Over Items | ## Step 1 --> 4: The flow will automatically run daily at 17:00 and check if any members are working their last shift on the current day. / * Step 5: If there are no records, the flow will end. If there are records, it will filter the records that are NOT in draft or canceled state. |
| Step8: Loop Over Items | splitInBatches | Iterate employees to offboard | Step7 (false); Step23; Step28; Step20 (false); Step21 (false); Step22 (false); Step25 (false); Step26 (false); Step27 (false) | Step9: Get employee information in Odoo 18 | ## Loop Over Items / - Get the information of that member on Odoo, then extract their work email. / - Use that email to find that user on RM6. / + If found, take the user's ID and lock the user. |
| Step9: Get employee information in Odoo 18 | httpRequest | Fetch employee work_email from Odoo | Step8: Loop Over Items | Step10: Handle work_email | ## Loop Over Items / - Get the information of that member on Odoo, then extract their work email. / - Use that email to find that user on RM6. / + If found, take the user's ID and lock the user. |
| Step10: Handle work_email | code | Derive local-part and username | Step9: Get employee information in Odoo 18 | Step11: Get user info in RM6 | ## Loop Over Items / - Get the information of that member on Odoo, then extract their work email. / - Use that email to find that user on RM6. / + If found, take the user's ID and lock the user. |
| Step11: Get user info in RM6 | httpRequest | Search Redmine user by name | Step10: Handle work_email | Step12:  Check if there is a record or not? | ## Loop Over Items / - Get the information of that member on Odoo, then extract their work email. / - Use that email to find that user on RM6. / + If found, take the user's ID and lock the user. |
| Step12:  Check if there is a record or not? | if | Proceed only if exactly one Redmine match | Step11: Get user info in RM6 | Step13… (true); Step24… (false) | ## Loop Over Items / - Get the information of that member on Odoo, then extract their work email. / - Use that email to find that user on RM6. / + If found, take the user's ID and lock the user. |
| Step13: Update the user's status to "locked" on RM6 | httpRequest | Lock Redmine user (status=3) | Step12 (true) | Step14: Remove the user from all groups in RM 6 | ## Lock, remove all group, delete membership in project on RM6 |
| Step14: Remove the user from all groups in RM 6 | httpRequest | Remove all Redmine groups | Step13 | Step15: Get membership list of user in RM 6 | ## Lock, remove all group, delete membership in project on RM6 |
| Step15: Get membership list of user in RM 6 | httpRequest | Fetch memberships to remove | Step14 | Step16: Get memberships_id | ## Lock, remove all group, delete membership in project on RM6 |
| Step16: Get memberships_id | code | Extract membership IDs | Step15 | Step17: Loop Over memberships_id | ## Lock, remove all group, delete membership in project on RM6 |
| Step17: Loop Over memberships_id | splitInBatches | Iterate memberships | Step16; Step18 | Code; Step18: Delete membership in project on RM 6 | ## Lock, remove all group, delete membership in project on RM6 |
| Step18: Delete membership in project on RM 6 | httpRequest | Delete a membership | Step17 (output 2) | Step17: Loop Over memberships_id | ## Lock, remove all group, delete membership in project on RM6 |
| Code | code | Continue after membership loop | Step17 (output 1) | Step19: Get user information in Gitlab | ## Lock, remove all group, delete membership in project on RM6 |
| Step19: Get user information in Gitlab | httpRequest | Search GitLab user | Code | Step20:  Check if there is a record = true | ## Check block a user on Gitlab |
| Step20:  Check if there is a record = true | if | Ensure GitLab search has results | Step19 | Step21 (true); Step8 (false) | ## Check block a user on Gitlab |
| Step21: Check if the user is not blocked = true | if | Skip if already blocked | Step20 (true) | Step22 (true); Step8 (false) | ## Check block a user on Gitlab |
| Step22: Check if the user is not deactivate = true | if | Skip if banned | Step21 (true) | Step23 (true); Step8 (false) | ## Check block a user on Gitlab |
| Step23: Block a user in Gitlab | httpRequest | Block GitLab user | Step22 (true) | Step8: Loop Over Items | ## Check block a user on Gitlab |
| Step24: Get user information in Gitlab | httpRequest | Search GitLab user (fallback path) | Step12 (false) | Step25:  Check if there is a record = true | ## Check block a user on Gitlab |
| Step25:  Check if there is a record = true | if | Ensure GitLab search has results | Step24 | Step26 (true); Step8 (false) | ## Check block a user on Gitlab |
| Step26: Check if the user is not blocked = true | if | Skip if already blocked | Step25 (true) | Step27 (true); Step8 (false) | ## Check block a user on Gitlab |
| Step27: Check if the user is not deactivate = true | if | Skip if banned | Step26 (true) | Step28 (true); Step8 (false) | ## Check block a user on Gitlab |
| Step28: Block a user in Gitlab | httpRequest | Block GitLab user | Step27 (true) | Step8: Loop Over Items | ## Check block a user on Gitlab |
| END | noOp | End when no resignations | Step5 (false) | — |  |
| END1 | noOp | End when no “leaving today” employees | Step7 (true) | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   “Automatically lock Redmine, Gitlab accounts for employees after their last shift”

2. **Add `Schedule Trigger`** node (`Step1: Schedule Trigger`)
   - Configure weekly schedule:
     - Days: Monday–Friday
     - Time: 17:00
   - Ensure n8n instance timezone matches the intended local timezone.

3. **Add `Code`** node (`Step2: Handle get today`)
   - Paste logic that outputs:
     - `date` in `YYYY-MM-DD`
     - `dayOfWeek`
   - (Optional) Fix weekday mapping if you plan to use it.

4. **Add `Set`** node (`Step3: Set Variables`)
   - Mode: **Raw JSON**
   - Add keys:
     - `redmine6_Url` (e.g., `https://redmine.example.com/`)
     - `gitlab_Url` (e.g., `https://gitlab.example.com/`)
     - `limit` (e.g., `100`)
     - `web_search_read_api` (Odoo JSON-RPC endpoint for `web_search_read`)
     - `web_read_api` (Odoo JSON-RPC endpoint for `web_read`)
     - `odooUrl` (optional; not required by current nodes)

5. **Create credentials: `HTTP Header Auth`**
   - **Odoo 18:** header-based API key/session header as required by your Odoo setup
   - **Redmine 6:** typically `X-Redmine-API-Key: <admin_api_key>`
   - **GitLab:** typically `PRIVATE-TOKEN: <admin_token>`

6. **Add `HTTP Request`** node (`Step4: Get Employee Resignation in Odoo 18`)
   - Method: POST
   - URL: `={{ $node['Step3: Set Variables'].json.web_search_read_api }}`
   - Authentication: **HTTP Header Auth** (Odoo credential)
   - Body: JSON-RPC calling:
     - model: `hr.resignation`
     - method: `web_search_read`
     - kwargs:
       - `limit: {{ Step3.limit }}`
       - domain with relevant resignation states
       - specification including `employee_id` and `expected_revealing_date` and `state`

7. **Add `If`** node (`Step5: If total_count is not equal to 0 = true`)
   - Condition: numeric **not equals**  
     - Left: `={{ $json.result.length }}`
     - Right: `0`
   - False output → connect to a **NoOp** node named `END`.

8. **Add `Code`** node (`Step6: Check if there is a valid record.`)
   - Filter:
     - exclude `draft` and `cancel`
     - keep `expected_revealing_date === today`
     - output items `{ id: employee_id.id }`
   - Important: consider using local timezone instead of UTC if needed.

9. **Add `If`** node (`Step7: If hasRecords is empty==> true`)
   - Condition: **empty** check on `={{ $json }}`
   - True → connect to **NoOp** `END1`
   - False → proceed to loop

10. **Add `Split In Batches`** (`Step8: Loop Over Items`)
   - Use default batch size (or set a specific size)
   - Connect Step7 false → Step8 input
   - Use **Output 2** of Step8 to proceed to employee processing nodes.

11. **Add `HTTP Request`** (`Step9: Get employee information in Odoo 18`)
   - Method: POST
   - URL: `={{ $node['Step3: Set Variables'].json.web_read_api }}`
   - Auth: Odoo header auth
   - JSON-RPC `web_read` on `hr.employee` with args `[[ {{ $json.id }} ]]`
   - Request `work_email` (and any other needed fields)

12. **Add `Code`** (`Step10: Handle work_email`)
   - Extract local-part: `work_email.split('@')[0]`
   - Derive `username` from local-part (current logic uses segment before `.`)

13. **Add `HTTP Request`** (`Step11: Get user info in RM6`)
   - GET: `{{ redmine6_Url }}users.json?name={{ Step10.work_email }}`
   - Auth: Redmine header auth

14. **Add `If`** (`Step12: Check if there is a record or not?`)
   - Condition: `{{ $json.total_count }} == 1`
   - True → Redmine lockdown chain
   - False → GitLab lookup fallback chain (Step24...)

15. **Redmine lockdown chain (true path)**
   1. `HTTP Request` (`Step13`) PUT user status locked (status=3)
   2. `HTTP Request` (`Step14`) PUT group_ids to `[]`
   3. `HTTP Request` (`Step15`) GET user with `include=memberships`
   4. `Code` (`Step16`) map `user.memberships` to `{id}`
   5. `SplitInBatches` (`Step17`) loop memberships
   6. `HTTP Request` (`Step18`) DELETE membership by id
      - Set **On Error** to “Continue (regular output)” if you want best-effort cleanup
   7. After Step17 completes, connect its completion output (Output 1) to a small `Code` node that outputs `{}` (named `Code`), then proceed to GitLab Step19.

16. **GitLab blocking chain A (after Redmine)**
   - `HTTP Request` (`Step19`) GET `/api/v4/users?search={{ Step10.username }}`
   - `If` (`Step20`) notEmpty result
   - `If` (`Step21`) state != blocked
   - `If` (`Step22`) state != banned
   - `HTTP Request` (`Step23`) POST `/api/v4/users/{id}/block`
   - After Step23 connect back to `Step8` to continue next employee.

17. **GitLab blocking chain B (when Redmine match is not exactly one)**
   - Same as chain A but starting at `Step24` → `Step28`
   - Connect Step28 back to `Step8`

18. **Activate the workflow** after verifying:
   - Credentials are valid
   - URLs include correct base paths and slashes
   - Odoo response fields match what code nodes expect

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Automatically lock Redmine, Gitlab accounts for employees after their last shift … (full sticky note text preserved) | Workflow purpose and intended capabilities (as described in the sticky note `Sticky Note2`). |
| ## Step 1 --> 4: The flow will automatically run daily at 17:00… | Scheduling + Odoo query intent (sticky note near Steps 1–5). |
| ## Loop Over Items … | Per-employee processing intent (sticky note near Step8–Step11). |
| ## Lock, remove all group, delete membership in project on RM6 | Redmine offboarding block label (sticky note near Steps 13–18). |
| ## Check block a user on Gitlab | GitLab offboarding block label (sticky note near GitLab nodes). |
| ## Use APIkey Odoo 18 | Credential guidance for Odoo (sticky note). |

