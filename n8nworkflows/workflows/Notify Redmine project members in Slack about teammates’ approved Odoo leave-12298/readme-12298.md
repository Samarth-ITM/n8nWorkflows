Notify Redmine project members in Slack about teammates’ approved Odoo leave

https://n8nworkflows.xyz/workflows/notify-redmine-project-members-in-slack-about-teammates--approved-odoo-leave-12298


# Notify Redmine project members in Slack about teammates’ approved Odoo leave

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Notify Redmine project members in Slack about teammates’ approved Odoo leave (tomorrow)  
**Workflow name (in JSON):** “Notify project members when a teammate has an approved leave tomorrow.”

**Purpose:**  
Every weekday at **17:15**, the workflow queries **Odoo (Time Off)** to find **approved leave** records starting “tomorrow” (time-window based), matches the absent employee to a **Redmine user**, determines the **active Redmine projects** they belong to, fetches **project teammates** (deduplicated), maps those teammates to **Slack users**, and sends **targeted Slack DMs**. If the employee has no Redmine account or no relevant project members, the workflow notifies the **employee’s manager** instead (or additionally).

**Main logical blocks (by dependency):**
1.1 **Scheduled start + global variables + datetime**  
1.2 **Reference data fetch (Redmine users, closed projects, Slack users)**  
1.3 **Query Odoo for tomorrow leave records + guard clause**  
1.4 **Per-leave-record enrichment (leave details, employee + manager)**  
1.5 **Redmine user lookup + project membership filtering**  
1.6 **Project teammate expansion via subflow + teammate email resolution**  
1.7 **Message recipient preparation (Slack ID matching, manager inclusion)**  
1.8 **Message sending via “Push message to member” subflow (timeoff/remote)**  
1.9 **Embedded subflows included in the same JSON (callable workflows)**

---

## 2. Block-by-Block Analysis

### 2.1 Scheduled start + global variables + datetime
**Overview:** Triggers the main flow on weekdays at 17:15, initializes configuration placeholders, and computes “today/tomorrow” strings in UTC+7 for downstream queries.

**Nodes involved:**
- Step 1: Schedule the trigger to run every weekday at 5:15 PM.
- Step2: Set Variables
- Step3: Get datetime

**Node details:**

**Node: Step 1: Schedule the trigger to run every weekday at 5:15 PM.**  
- **Type/role:** Schedule Trigger — entry point for the main flow.  
- **Config:** Runs every day interval `1` at **17:15**. (Despite sticky note wording “weekday”, the configuration shown is “days interval = 1”; true weekday-only scheduling would require a weekday rule.)  
- **Outputs to:** Step2: Set Variables  
- **Failure modes:** None typical; mis-scheduling is the main risk (weekdays vs daily).  
- **Version:** scheduleTrigger v1.2.

**Node: Step2: Set Variables**  
- **Type/role:** Set node — intended to store environment constants (URLs, limits, links).  
- **Config choices:** `mode=raw`, but **jsonOutput is currently an empty object**. Yet many downstream nodes reference keys like:
  - `redmine6_Url`, `limit`, `get_prj_closed`
  - `hr_leave_web_search_read`, `hr_leave_web_read`, `hr_employee_web_read`
  - `time_off_approval_link`
- **Outputs to:** Step3: Get datetime  
- **Edge cases/failures:** If these variables are not filled, HTTP nodes will generate invalid URLs / expression errors.  
- **Version:** set v3.4.

**Node: Step3: Get datetime**  
- **Type/role:** Code node — produces date strings for filtering.  
- **Logic:**  
  - Converts “now” to **UTC+7** by adding 7 hours.
  - Computes `today`, `yesterday`, `twoDaysAgo`, `tomorrow` as `YYYY-MM-DD`
  - Also outputs `dayOfWeekYesterday` and `monthYesterday`  
- **Outputs to:** Step4: Get all user in Redmine6  
- **Edge cases:** The “UTC+7 by adding hours” approach can be wrong if the server is not in UTC or DST-related assumptions change.  
- **Version:** code v2.

---

### 2.2 Reference data fetch (Redmine users, closed projects, Slack users)
**Overview:** Loads reference datasets used later for mapping Redmine user IDs → email, excluding closed projects, and mapping email → Slack memberID.

**Nodes involved:**
- Step4: Get all user in Redmine6
- Step5: Get a list of closed projects.
- Step6: Get many users

**Node details:**

**Node: Step4: Get all user in Redmine6**  
- **Type/role:** HTTP Request — fetches Redmine users list.  
- **Config:**  
  - GET: `{{redmine6_Url}}users.json?limit={{limit}}`
  - Auth: **HTTP Header Auth** (credential “API KEY Redmine6 Test”)  
  - `executeOnce=true` (cached per execution), `retryOnFail=true`, `maxTries=2`, `alwaysOutputData=true`  
- **Outputs to:** Step5: Get a list of closed projects.  
- **Edge cases:** pagination (limit may not cover all users), API key permissions, Redmine 6 endpoint differences.  
- **Version:** httpRequest v4.2.

**Node: Step5: Get a list of closed projects.**  
- **Type/role:** HTTP Request — returns projects that should be excluded.  
- **Config:** GET `{{get_prj_closed}}`, same Redmine header auth, retry enabled, `alwaysOutputData=true`.  
- **Outputs to:** Step6: Get many users  
- **Edge cases:** endpoint must return JSON with `.projects` array (expected later in Step20).  
- **Version:** httpRequest v4.2.

**Node: Step6: Get many users**  
- **Type/role:** Slack node — fetches Slack user directory.  
- **Config:** Slack resource `user`, operation `getAll`, `limit=100`. Uses Slack API credential “Slack API Test”.  
- **Outputs to:** Step7: Odoo query  
- **Edge cases:** limit 100 might not include all users; Slack pagination handling depends on node behavior. If your workspace >100 users, increase limit and ensure pagination is supported.  
- **Version:** slack v2.4.

---

### 2.3 Query Odoo for tomorrow leave records + guard clause
**Overview:** Queries Odoo `hr.leave` for leave starting within a specific window (today evening → tomorrow afternoon) and exits if no records.

**Nodes involved:**
- Step7: Get the list of members' leave records for tomorrow.
- Step8: Check if there is a record or not?
- End1

**Node details:**

**Node: Step7: Get the list of members' leave records for tomorrow.**  
- **Type/role:** HTTP Request — Odoo JSON-RPC `web_search_read` on `hr.leave`.  
- **Config:**  
  - POST to `{{hr_leave_web_search_read}}` with JSON-RPC body.
  - Domain includes:
    - company filter `employee_id.company_id in [1]`
    - start date window: `date_from >= "{{today}} 18:54:03"` AND `date_from <= "{{tomorrow}} 16:54:03"`
    - duration thresholds: hours>=4 OR days>=0.5
  - `limit={{limit}}`
  - Auth: HTTP Header Auth (“API KEY Odoo Test”)  
  - `alwaysOutputData=true`, retry enabled.  
- **Outputs to:** Step8  
- **Edge cases:**  
  - The time window is hard-coded (18:54:03 / 16:54:03) and may not match “tomorrow leave” precisely.
  - If Odoo returns unexpected shapes, Step8 and Step9 may fail.
- **Version:** httpRequest v4.2.

**Node: Step8: Check if there is a record or not?**  
- **Type/role:** IF — stops if there are no results.  
- **Condition:** `$json.result.length != 0`  
- **True output:** Step9  
- **False output:** End1  
- **Edge cases:** If Odoo response doesn’t include `.result.length`, expression can error; `alwaysOutputData` on Step7 helps but doesn’t guarantee structure.  
- **Version:** if v2.3.

**Node: End1**  
- **Type/role:** NoOp — explicit end when no leave records.  
- **Version:** noOp v1.

---

### 2.4 Per-leave-record enrichment (leave details, employee + manager, merge context)
**Overview:** Iterates each leave record, fetches full leave detail, extracts normalized dates, fetches employee and manager profiles to obtain work emails, then merges all enriched fields into one item per leave.

**Nodes involved:**
- Step9: Handling and get user information in Odoo 18
- Step10: Loop Over Items
- Step11: Get leave record information
- Step12: Get name record
- Step13: Get employee information in Odoo 18
- Step14: Get work_email
- Step15: Get information for this employee's manager on Odoo 18
- Step16: Get work_email of manager
- Step21: Merge data

**Node details:**

**Node: Step9: Handling and get user information in Odoo 18**  
- **Type/role:** Code — transforms Odoo search results into a list of items.  
- **Outputs fields per item:** `odoo_leave_record_id`, `odoo_employee_id`, `odoo_display_name`, `odoo_user_id`.  
- **Outputs to:** Step10  
- **Edge cases:** Assumes `inputData[0].json.result.records` exists. If missing, will throw.  
- **Version:** code v2.

**Node: Step10: Loop Over Items**  
- **Type/role:** SplitInBatches — iterates leave items.  
- **Connections:**  
  - Batch output → Step11 and also to Step22 (loop that will later drive message sending logic)  
  - Second output also connects to Step21 (index 2) as part of merge wiring  
- **Edge cases:** default batch size (not explicitly set) depends on node defaults; could process multiple at once.  
- **Version:** splitInBatches v3.

**Node: Step11: Get leave record information**  
- **Type/role:** HTTP Request — Odoo JSON-RPC `web_read` for `hr.leave` for a single leave ID.  
- **Config:** POST to `{{hr_leave_web_read}}`; reads detailed fields including `display_name`, `date_from`, `date_to`, etc.  
- **Outputs to:** Step12  
- **Edge cases:** `alwaysOutputData=false` here—if request fails, downstream may not run.  
- **Version:** httpRequest v4.2.

**Node: Step12: Get name record**  
- **Type/role:** Code — extracts `display_name` and normalizes `date_from/date_to` by adding +7 hours, returns formatted `YYYY-MM-DD HH:mm:ss`.  
- **Failure behavior:** Throws explicit errors if result array missing or date fields missing/invalid.  
- **Outputs to:** Step13 and also into Step21 (merge input index 1).  
- **Version:** code v2.

**Node: Step13: Get employee information in Odoo 18**  
- **Type/role:** HTTP Request — Odoo JSON-RPC `web_read` for `hr.employee` for the employee ID.  
- **Outputs to:** Step14  
- **Edge cases:** If employee lacks `work_email` or `parent_id`, later steps fail.  
- **Version:** httpRequest v4.2.

**Node: Step14: Get work_email**  
- **Type/role:** Code — extracts:
  - `odoo_work_email`
  - `odoo_emailTrim` (local-part before `@`, used to query Redmine user by name)
  - `odoo_parent_id` (manager employee id)  
- **Outputs to:** Step15 and Step21 (merge input index 0).  
- **Edge cases:** If `work_email` is null/empty, `.split('@')` will throw.  
- **Version:** code v2.

**Node: Step15: Get information for this employee's manager on Odoo 18**  
- **Type/role:** HTTP Request — reads manager employee record using `odoo_parent_id`.  
- **Outputs to:** Step16  
- **Edge cases:** If employee has no manager (`parent_id` missing), this HTTP call will be invalid.  
- **Version:** httpRequest v4.2.

**Node: Step16: Get work_email of manager**  
- **Type/role:** Code — extracts manager email + local-part.  
- **Outputs to:** Step17 and Step21 (merge input index 4).  
- **Edge cases:** Same as Step14; fails if manager work_email missing.  
- **Version:** code v2.

**Node: Step21: Merge data**  
- **Type/role:** Merge — combines multiple parallel branches into one enriched item.  
- **Mode:** `combineByPosition`, `numberInputs=5`.  
- **Expected inputs (by connections):**
  1) from Step14 (employee email/manager id)  
  2) from Step12 (leave display_name + adjusted dates)  
  3) from Step10 (leave id/employee id base item)  
  4) from Step20 OR Step19.2 (Redmine membership results, later)  
  5) from Step16 (manager email)  
- **Outputs to:** Step10: Loop Over Items (creates a feedback-style topology where merged data is used downstream for per-item operations), ultimately Step10 → Step22 continues.  
- **Edge cases:** `combineByPosition` is fragile: if any branch yields different item counts/order, merges become mismatched.  
- **Version:** merge v3.2.

---

### 2.5 Redmine user lookup + project membership filtering
**Overview:** Finds the employee’s Redmine user (by `name=<email local-part>`), checks if exactly one match exists, then fetches memberships and filters out excluded/closed projects.

**Nodes involved:**
- Step17: Get user info in Redmine 6
- Step18: Check if there is a record or not?
- Step19.1: Get membership list of user in Redmine 6
- Step19.2: Return isAccountRedmine = false
- Step20: Get project IDs that the member is participating in.

**Node details:**

**Node: Step17: Get user info in Redmine 6**  
- **Type/role:** HTTP Request — searches Redmine users by name.  
- **Config:** GET `{{redmine6_Url}}users.json?name={{odoo_emailTrim}}`  
- **Outputs to:** Step18  
- **Edge cases:** Redmine “name” search might return multiple results; depends on Redmine config and user naming.  
- **Version:** httpRequest v4.2.

**Node: Step18:  Check if there is a record or not?**  
- **Type/role:** IF — ensures unambiguous match.  
- **Condition:** `$json.total_count == 1`  
- **True:** Step19.1  
- **False:** Step19.2  
- **Edge cases:** If 0 or >1, considered “no account” path.  
- **Version:** if v2.2.

**Node: Step19.1: Get membership list of user in Redmine 6**  
- **Type/role:** HTTP Request — reads user with memberships included.  
- **Config:** GET `users/<id>.json?include=memberships`  
- **Outputs to:** Step20  
- **Edge cases:** memberships may be absent; API key needs permission.  
- **Version:** httpRequest v4.2.

**Node: Step19.2: Return isAccountRedmine = false**  
- **Type/role:** Code — produces a normalized “no Redmine” object:
  - `isAccountRedmine=false`
  - `redmine_listProjectID=[]`  
- **Outputs to:** Step21 (merge input index 3)  
- **Version:** code v2.

**Node: Step20: Get project IDs that the member is participating in.**  
- **Type/role:** Code — filters memberships to active projects.  
- **Logic:**  
  - Builds an exclusion set containing hard-coded project IDs `[8,9,10,12]` plus IDs from “closed projects” API response.  
  - From memberships, keeps only projects not excluded, returns:
    - `isAccountRedmine=true`
    - `redmine_listProjectID: [{id: ...}, ...]`
    - `redmine_userID`, `redmine_mail`
- **Outputs to:** Step21 (merge input index 3)  
- **Edge cases:** Assumes closed projects response has `.projects`.  
- **Version:** code v2.

---

### 2.6 Project teammate expansion via subflow + teammate email resolution
**Overview:** For each leave item, if the employee has active Redmine projects, calls a sub-workflow to fetch all memberships per project, deduplicate teammate Redmine IDs (excluding the absent person), then maps teammate IDs to emails using the Redmine users list.

**Nodes involved:**
- Step22: Loop Over Items
- Step 23: If redmine_listProjectID != [] ==> true
- Step 24: Call subflow: "Get membership list of user in Redmine'
- Step25: Get email Redmine project Team Member Info

**Node details:**

**Node: Step22: Loop Over Items**  
- **Type/role:** SplitInBatches — iterates enriched leave items.  
- **Outputs:**  
  - Main output 0 → Step26 (a later loop used around message sending)  
  - Output 1 → Step 23 (branch for fetching project members)  
- **Edge cases:** Again depends on batch defaults.  
- **Version:** splitInBatches v3.

**Node: Step 23: If redmine_listProjectID != [] ==> true**  
- **Type/role:** IF — ensures Redmine account exists and project list is non-empty.  
- **Conditions:**  
  - `isAccountRedmine == true`
  - `redmine_listProjectID` is not empty  
- **True:** Step 24 (subflow call)  
- **False:** loops back into Step22 (effectively “skip and continue”)  
- **Version:** if v2.3.

**Node: Step 24: Call subflow: "Get membership list of user in Redmine'**  
- **Type/role:** Execute Workflow — calls external workflow: **“Subflow Get membership list of user in RM”** (`workflowId` shown in JSON).  
- **Config:** `retryOnFail=true`, `maxTries=2`. Inputs mapping is empty (passthrough is not enabled here; actual data passing depends on Execute Workflow node behavior and settings).  
- **Outputs to:** Step25  
- **Edge cases:**  
  - If the subflow expects specific fields and they aren’t passed, it fails.
  - The referenced workflow must exist and be accessible.  
- **Version:** executeWorkflow v1.3.

**Node: Step25: Get email Redmine project Team Member Info**  
- **Type/role:** Code — converts Redmine teammate IDs into teammate emails using the Redmine users list from Step4.  
- **Key dependencies:**  
  - Input: `redmine_projectTeamMemberIds` from subflow output
  - Reference: `Step4: Get all user in Redmine6`.json.users  
- **Output:** `redmine_projectTeamMemberInfo: [{redmine_userID, redmine_mail}, ...]` (mail can be null if user not found in list)  
- **Outputs to:** Step22 (continues loop)  
- **Edge cases:** Redmine user list may not include all users if pagination/limit too low.  
- **Version:** code v2.

---

### 2.7 Message recipient preparation (Slack ID matching, manager inclusion)
**Overview:** Determines whether to notify teammates (if Redmine account exists) or manager only, matches recipient emails to Slack IDs, and constructs message payload objects for the message-sending subflow.

**Nodes involved:**
- Step26: Loop Over Items
- Step 27: If redmine_listProjectID != [] ==> true
- Step28.1: Prepare information about leave schedules for announcement.
- Step28.2: Prepare information about leave schedules for announcement.
- End

**Node details:**

**Node: Step26: Loop Over Items**  
- **Type/role:** SplitInBatches — iterates over items that should result in Slack messages.  
- **Outputs:**  
  - Output 0 → End (when batches complete)
  - Output 1 → Step 27  
- **Version:** splitInBatches v3.

**Node: Step 27: If redmine_listProjectID != [] ==> true**  
- **Type/role:** IF — decides teammate notification vs manager-only.  
- **Condition:** only checks `isAccountRedmine == true` (note: unlike Step23, it does **not** require project list non-empty).  
- **True:** Step28.1 (notify teammates + possibly manager)  
- **False:** Step28.2 (notify manager only)  
- **Edge cases:** If `isAccountRedmine=true` but `redmine_projectTeamMemberInfo` missing/empty, Step28.1 handles empty list by switching to manager-only behavior internally.  
- **Version:** if v2.3.

**Node: Step28.1: Prepare information about leave schedules for announcement.**  
- **Type/role:** Code — builds an array of recipient objects (team members and possibly manager).  
- **Key inputs/expressions:**  
  - `redmine_projectTeamMemberInfo` from upstream merge context
  - Slack directory from `Step6: Get many users` (via `$items("Step6: Get many users")`)
  - Leave fields: `display_name_leave_record`, `date_from`, `date_to`, `odoo_leave_record_id`
  - `time_off_approval_link` from variables
  - Manager email from merged item (`odoo_manager_work_email`)
- **Logic highlights:**  
  - Fuzzy match Slack user by email/local-part; assigns `memberID` (Slack DM channel/user ID) when found.
  - If no teammates found, returns a single record for manager (`isManager: true`).
  - If manager exists but not already in teammate list, append manager notification.  
- **Outputs to:** Step29 (subflow send)  
- **Failure modes:** Missing Slack emails, missing manager email, or missing `time_off_approval_link`.  
- **Version:** code v2.

**Node: Step28.2: Prepare information about leave schedules for announcement.**  
- **Type/role:** Code — manager-only payload builder.  
- **Outputs:** one object `{type:"timeoff", memberID, title, date_from, date_to, link, redmine_mail}`  
- **Outputs to:** Step29  
- **Edge cases:** Same as above; if manager not found in Slack, `memberID` becomes undefined and Slack API will fail downstream unless handled.  
- **Version:** code v2.

**Node: End**  
- **Type/role:** NoOp — end of loop/flow.  
- **Version:** noOp v1.

---

### 2.8 Message sending via subflow “Push message to member” (timeoff/remote)
**Overview:** Calls a separate workflow to post Slack messages. In the embedded subflow implementation (in this JSON), a Switch routes by message type and posts to Slack `chat.postMessage`.

**Nodes involved:**
- Step29: Call subflow: 'Push message to member'
- (Embedded subflow nodes described in block 2.9)

**Node details:**

**Node: Step29: Call subflow: 'Push message to member'**  
- **Type/role:** Execute Workflow — calls external workflow: **“Subflow push message to member”**.  
- **Config:** `mode=each`, `waitForSubWorkflow=true`.  
- **Outputs to:** Step26 (continues batching loop)  
- **Edge cases:** Subflow must exist; also message objects must contain `type`, `memberID`, `title`, `date_from`, `date_to`, `link`.  
- **Version:** executeWorkflow v1.3.

---

### 2.9 Embedded subflows included inside this JSON

This JSON contains nodes that appear to represent **two callable workflows** (subflows) placed in the same export. They are not connected to the main schedule trigger; they start from **Execute Workflow Trigger** nodes.

#### 2.9.1 Subflow: Get membership list of user in Redmine
**Overview:** Receives a user’s list of Redmine project IDs, loops each project to fetch memberships, extracts teammate user IDs, and deduplicates them excluding the originating user.

**Nodes involved:**
- Step1: When Executed by Another Workflow
- Step2: Return redmine list ProjectID of user
- Step3: Loop Over Items
- Step4: Get membership list of user in Redmine 6
- Step5: Get redmine project Team MemberIds
- Step6: Remove duplicate id

**Node details:**

**Step1: When Executed by Another Workflow**  
- **Type/role:** ExecuteWorkflowTrigger — entry point for this subflow.  
- **InputSource:** passthrough (expects caller to pass fields like `redmine_listProjectID` and `redmine_userID`).  
- **Outputs to:** Step2  
- **Version:** executeWorkflowTrigger v1.1.

**Step2: Return redmine list ProjectID of user**  
- **Type/role:** Code — returns `redmine_listProjectID` as items so SplitInBatches can iterate.  
- **Output:** array of `{id: <projectId>}` items.  
- **Outputs to:** Step3  
- **Version:** code v2.

**Step3: Loop Over Items**  
- **Type/role:** SplitInBatches with `batchSize=1` — iterates project IDs.  
- **Outputs:**  
  - Output 0 → Step6 (dedupe aggregator pattern)  
  - Output 1 → Step4 (fetch memberships)  
- **Edge cases:** Dedupe pattern relies on `$items("Step3: Loop Over Items")` later; changes in execution mode can break this.  
- **Version:** splitInBatches v3.

**Step4: Get membership list of user in Redmine 6**  
- **Type/role:** HTTP Request — GET `projects/<id>/memberships.json` using Redmine header auth.  
- **Outputs to:** Step5  
- **Version:** httpRequest v4.2.

**Step5: Get redmine project Team MemberIds**  
- **Type/role:** Code — extracts `memberships[].user.id` into `redmine_projectTeamMemberIds:[{id},...]`.  
- **Outputs to:** Step3 (feeds back into batch loop)  
- **Version:** code v2.

**Step6: Remove duplicate id**  
- **Type/role:** Code — collects all teammate IDs from the loop, deduplicates, removes the originating user ID(s) from `Step1` input (`redmine_userID`), outputs final `redmine_projectTeamMemberIds`.  
- **Outputs:** (This node is not shown connected onward in the JSON connections; in a typical subflow it should be the final output node.)  
- **Edge cases:**  
  - If `redmine_userID` is missing, exclusion may not work as intended.
  - If loop hasn’t produced items yet, dedupe may output empty.  
- **Version:** code v2.

#### 2.9.2 Subflow: Push message to member
**Overview:** Receives a message payload, optionally rate-limits with a Wait, switches by `type` (timeoff/remote), and posts to Slack `chat.postMessage`.

**Nodes involved:**
- Step2: Loop Over Items
- Step3: Wait 1s
- Step4: Switch
- Step5.1: Send a message to the project team members. (timeoff)
- Step5.2: Send a message to the project team members. (remote)
- No Operation, do nothing
- End

**Node details:**

**Step2: Loop Over Items**  
- **Type/role:** SplitInBatches — orchestrates message sends and loop continuation.  
- **Outputs:** to “No Operation” and to “Step3: Wait 1s” (depending on batch output index).  
- **Edge cases:** batch configuration not explicit; rate-limit logic may not behave as expected.  
- **Version:** splitInBatches v3.

**Step3: Wait 1s**  
- **Type/role:** Wait — simple throttling before Slack call.  
- **Outputs to:** Step4  
- **Version:** wait v1.1.

**Step4: Switch**  
- **Type/role:** Switch — routes based on `{{$node['Step1: When Executed by Another Workflow'].json.type}}` equaling `timeoff` or `remote`.  
- **Outputs to:** Step5.1 or Step5.2  
- **Edge cases:** If `type` is neither, no output path: message dropped silently.  
- **Version:** switch v3.4.

**Step5.1 / Step5.2: Send a message to the project team members.**  
- **Type/role:** HTTP Request — Slack Web API `chat.postMessage`.  
- **Config:**  
  - URL: `https://slack.com/api/chat.postMessage`
  - POST body params: `channel={{memberID}}`, `text=...` (formatted message)
  - Auth: **HTTP Bearer Auth** (credential “Bearer Auth account Odoo Bot TEST”)  
  - `onError=continueRegularOutput` (won’t hard-fail subflow), retries enabled.  
- **Outputs to:** Step2: Loop Over Items (continue)  
- **Edge cases:**  
  - If `memberID` undefined/not a valid channel/user, Slack returns error.
  - Token scopes must include `chat:write` and ability to DM users.  
- **Version:** httpRequest v4.2.

**No Operation, do nothing / End**  
- **Type/role:** NoOp — placeholders/terminators.  
- **Version:** noOp v1.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Step 1: Schedule the trigger to run every weekday at 5:15 PM. | Schedule Trigger | Main entry schedule | — | Step2: Set Variables | ## Schedule daily runs at 5:15 PM. \n- Get some information about datetime, the list of users on Redmine, and the list of closed projects.\n- Add necessary variables. |
| Step2: Set Variables | Set | Store URLs/limits/links used everywhere | Step 1: Schedule… | Step3: Get datetime | ## Schedule daily runs at 5:15 PM. … |
| Step3: Get datetime | Code | Compute today/tomorrow strings (UTC+7) | Step2: Set Variables | Step4: Get all user in Redmine6 | ## Schedule daily runs at 5:15 PM. … |
| Step4: Get all user in Redmine6 | HTTP Request | Fetch Redmine users for ID→email mapping | Step3: Get datetime | Step5: Get a list of closed projects. | ## Schedule daily runs at 5:15 PM. … |
| Step5: Get a list of closed projects. | HTTP Request | Fetch closed projects to exclude | Step4: Get all user… | Step6: Get many users | ## Schedule daily runs at 5:15 PM. … |
| Step6: Get many users | Slack | Fetch Slack users directory | Step5: Get…closed projects | Step7: Get the list of members' leave records for tomorrow. | ## Schedule daily runs at 5:15 PM. … |
| Step7: Get the list of members' leave records for tomorrow. | HTTP Request | Query Odoo hr.leave (tomorrow window) | Step6: Get many users | Step8: Check if there is a record or not? | ## Querying data in Odoo … |
| Step8: Check if there is a record or not? | IF | Guard: stop when no leave | Step7 | Step9 / End1 | ## Querying data in Odoo … |
| End1 | NoOp | End when no records | Step8 (false) | — | ## Querying data in Odoo … |
| Step9: Handling and get user information in Odoo 18 | Code | Normalize leave search results to items | Step8 (true) | Step10: Loop Over Items | ## Querying data in Odoo … |
| Step10: Loop Over Items | SplitInBatches | Iterate each leave record | Step9 / Step21 | Step11 / Step22 / Step21 | ## Loop over item … |
| Step11: Get leave record information | HTTP Request | Odoo web_read for leave record | Step10 | Step12 | ## Loop over item … |
| Step12: Get name record | Code | Extract display_name and adjust dates | Step11 | Step13 / Step21 | ## Loop over item … |
| Step13: Get employee information in Odoo 18 | HTTP Request | Odoo web_read for employee | Step12 | Step14 | ## Loop over item … |
| Step14: Get work_email | Code | Extract employee email + manager id | Step13 | Step15 / Step21 | ## Loop over item … |
| Step15: Get information for this employee's manager on Odoo 18 | HTTP Request | Odoo web_read for manager employee | Step14 | Step16 | ## Loop over item … |
| Step16: Get work_email of manager | Code | Extract manager email | Step15 | Step17 / Step21 | ## Loop over item … |
| Step17: Get user info in Redmine 6 | HTTP Request | Find Redmine user by email local-part | Step16 | Step18 | ## Get user info in Redmine … |
| Step18:  Check if there is a record or not? | IF | Ensure unique Redmine match | Step17 | Step19.1 / Step19.2 | ## Get user info in Redmine … |
| Step19.1: Get membership list of user in Redmine 6 | HTTP Request | Get user memberships | Step18 (true) | Step20 | ## Get user info in Redmine … |
| Step19.2: Return isAccountRedmine = false | Code | Fallback when no Redmine user | Step18 (false) | Step21 | ## Get user info in Redmine … |
| Step20: Get project IDs that the member is participating in. | Code | Filter memberships to active projects | Step19.1 | Step21 | ## Get user info in Redmine … |
| Step21: Merge data | Merge | Combine leave + employee + manager + Redmine info | Step14, Step12, Step10, Step20/19.2, Step16 | Step10 | (no sticky note) |
| Step22: Loop Over Items | SplitInBatches | Iterate enriched leave items for project teammate lookup | Step10 / Step25 / Step 23 (false loop) | Step26 / Step 23 | (no sticky note) |
| Step 23: If redmine_listProjectID != [] ==> true | IF | Decide whether to call membership subflow | Step22 | Step 24 / Step22 | (no sticky note) |
| Step 24: Call subflow: "Get membership list of user in Redmine' | Execute Workflow | Expand project teammate IDs | Step 23 (true) | Step25 | (no sticky note) |
| Step25: Get email Redmine project Team Member Info | Code | Map teammate IDs to emails via Redmine users list | Step 24 | Step22 | (no sticky note) |
| Step26: Loop Over Items | SplitInBatches | Iterate message payload items | Step22 / Step29 | End / Step 27 | (no sticky note) |
| Step 27: If redmine_listProjectID != [] ==> true | IF | Choose teammates+manager vs manager-only | Step26 | Step28.1 / Step28.2 | (no sticky note) |
| Step28.1: Prepare information about leave schedules for announcement. | Code | Build recipient list + Slack ID matching | Step 27 (true) | Step29 | (no sticky note) |
| Step28.2: Prepare information about leave schedules for announcement. | Code | Build manager-only payload + Slack ID matching | Step 27 (false) | Step29 | (no sticky note) |
| Step29: Call subflow: 'Push message to member' | Execute Workflow | Send Slack messages (via subflow) | Step28.1 / Step28.2 | Step26 | (no sticky note) |
| End | NoOp | End of processing loop | Step26 (done) | — | (no sticky note) |
| Step1: When Executed by Another Workflow | Execute Workflow Trigger | Subflow entry: Redmine membership list | — | Step2: Return redmine list ProjectID of user | ## Subflow: Get membership list of user in Redmine … |
| Step2: Return redmine list ProjectID of user | Code | Subflow: emit project IDs to iterate | Step1: When Executed… | Step3: Loop Over Items | ## Subflow: Get membership list of user in Redmine … |
| Step3: Loop Over Items | SplitInBatches | Subflow: iterate projects | Step2: Return… / Step5 | Step4 / Step6 | ## Subflow: Get membership list of user in Redmine … |
| Step4: Get membership list of user in Redmine 6 | HTTP Request | Subflow: project memberships | Step3 | Step5 | ## Subflow: Get membership list of user in Redmine … |
| Step5: Get redmine project Team MemberIds | Code | Subflow: extract teammate IDs | Step4 | Step3 | ## Subflow: Get membership list of user in Redmine … |
| Step6: Remove duplicate id | Code | Subflow: dedupe teammate IDs, exclude absent user | Step3 | (not connected) | ## Subflow: Get membership list of user in Redmine … |
| Step2: Loop Over Items | SplitInBatches | Subflow: message send loop | Step5.1 / Step5.2 | No Operation, do nothing / Step3: Wait 1s | ## Subflow: Push message to member … |
| Step3: Wait 1s | Wait | Subflow: throttle | Step2: Loop Over Items | Step4: Switch | ## Subflow: Push message to member … |
| Step4: Switch | Switch | Subflow: route by type | Step3: Wait 1s | Step5.1 / Step5.2 | ## Subflow: Push message to member … |
| Step5.1: Send a message to the project team members. | HTTP Request | Subflow: Slack postMessage (timeoff) | Step4: Switch | Step2: Loop Over Items | ## Subflow: Push message to member … |
| Step5.2: Send a message to the project team members. | HTTP Request | Subflow: Slack postMessage (remote) | Step4: Switch | Step2: Loop Over Items | ## Subflow: Push message to member … |
| No Operation, do nothing | NoOp | Subflow: placeholder | Step2: Loop Over Items | — | ## Subflow: Push message to member … |
| Sticky Note / Sticky Note1 / Sticky Note2 / Sticky Note3 / Sticky Note4 / Sticky Note5 / Sticky Note6 / Sticky Note7 / Sticky Note10 | Sticky Note | Documentation annotations | — | — | (These are notes, not functional nodes) |

> Note: Some sticky notes visually describe blocks; not every node is physically “covered” by a note in JSON. The table includes the intended contextual notes where clearly applicable.

---

## 4. Reproducing the Workflow from Scratch

### 4.1 Create credentials (required first)
1) **Redmine 6 API key**  
   - Create an **HTTP Header Auth** credential in n8n.  
   - Header typically: `X-Redmine-API-Key: <key>` (or whatever your Redmine expects).

2) **Odoo API access**  
   - Create **HTTP Header Auth** for Odoo JSON-RPC endpoint access (as used here).  
   - Ensure headers match your Odoo gateway/proxy requirement (often session cookie or token, but this workflow assumes header-based access).

3) **Slack**  
   - For user directory: create a **Slack API** credential (OAuth token).  
   - For posting messages: create **HTTP Bearer Auth** with a Slack bot token that has `chat:write` and DM permissions.

---

### 4.2 Build the Main Workflow
1) Add **Schedule Trigger** node  
   - Set to run daily at **17:15** (and if you truly want weekdays only, configure weekday rules).

2) Add **Set** node “Step2: Set Variables”  
   - Populate at least these fields (examples of what you must define conceptually):
     - `redmine6_Url` (base URL, ending with `/`)
     - `limit` (integer; ensure large enough for users/leaves)
     - `get_prj_closed` (full endpoint returning `{projects:[...]}`)
     - `hr_leave_web_search_read` (Odoo JSON-RPC endpoint for search_read)
     - `hr_leave_web_read` (Odoo JSON-RPC endpoint for web_read hr.leave)
     - `hr_employee_web_read` (Odoo JSON-RPC endpoint for web_read hr.employee)
     - `time_off_approval_link` (base URL to open the leave approval in Odoo UI)

3) Add **Code** node “Step3: Get datetime”  
   - Implement UTC+7 date computations and output `today` and `tomorrow` as `YYYY-MM-DD`.

4) Add **HTTP Request** “Step4: Get all user in Redmine6”  
   - GET `{{redmine6_Url}}users.json?limit={{limit}}`  
   - Auth: Redmine Header Auth credential.

5) Add **HTTP Request** “Step5: Get a list of closed projects.”  
   - GET `{{get_prj_closed}}`  
   - Auth: Redmine Header Auth.

6) Add **Slack** node “Step6: Get many users”  
   - Resource: User, Operation: Get All, Limit: set high enough for your workspace.

7) Add **HTTP Request** “Step7: Get the list of members' leave records for tomorrow.”  
   - POST to `{{hr_leave_web_search_read}}` using JSON-RPC.  
   - Domain must filter approved leaves starting tomorrow (adapt the time window to your policy).  
   - Auth: Odoo Header Auth.

8) Add **IF** node “Step8”  
   - Check the Odoo response contains at least one record.

9) False branch → add **NoOp** “End1”.

10) True branch → add **Code** “Step9”  
   - Map records into items with `odoo_leave_record_id` and `odoo_employee_id`.

11) Add **SplitInBatches** “Step10” to iterate each leave.

12) Add **HTTP Request** “Step11” (Odoo hr.leave web_read) using `odoo_leave_record_id`.

13) Add **Code** “Step12”  
   - Extract `display_name`, `date_from`, `date_to`; normalize timezone.

14) Add **HTTP Request** “Step13” (Odoo hr.employee web_read) using `odoo_employee_id`.

15) Add **Code** “Step14”  
   - Extract `work_email`, `emailTrim`, and `parent_id`.

16) Add **HTTP Request** “Step15” (Odoo hr.employee web_read) using manager id.

17) Add **Code** “Step16”  
   - Extract manager email.

18) Add **HTTP Request** “Step17” (Redmine users search)  
   - GET `{{redmine6_Url}}users.json?name={{odoo_emailTrim}}`

19) Add **IF** “Step18”  
   - Condition: `total_count == 1`

20) True → **HTTP Request** “Step19.1”  
   - GET `users/<id>.json?include=memberships`

21) False → **Code** “Step19.2”  
   - Output `isAccountRedmine=false` and empty project list.

22) Add **Code** “Step20” after Step19.1  
   - Filter memberships; exclude closed projects and hard-coded project IDs.

23) Add **Merge** “Step21”  
   - Mode: combine by position, **5 inputs**.  
   - Connect: Step14, Step12, Step10, Step20/Step19.2, Step16.

24) Add **SplitInBatches** “Step22”  
   - Iterate merged items.

25) Add **IF** “Step23”  
   - True when Redmine account exists and `redmine_listProjectID` not empty.

26) True branch → add **Execute Workflow** “Step24”  
   - Select workflow: “Subflow Get membership list of user in RM”.

27) Add **Code** “Step25”  
   - Map teammate IDs to emails using `Step4` Redmine users list.

28) Add **SplitInBatches** “Step26” to iterate message targets.

29) Add **IF** “Step27”  
   - True when `isAccountRedmine == true`.

30) True → add **Code** “Step28.1” (teammates + manager logic)  
    - Match Slack memberID by email from Slack directory (Step6).

31) False → add **Code** “Step28.2” (manager only).

32) Add **Execute Workflow** “Step29”  
   - Select workflow: “Subflow push message to member”  
   - Mode: **each**, wait for completion.

33) Add **NoOp** “End” connected from Step26 completion output.

---

### 4.3 Create Sub-workflow: “Subflow Get membership list of user in RM”
1) New workflow → add **Execute Workflow Trigger** (passthrough).
2) Add **Code** node that outputs the incoming `redmine_listProjectID` as individual items.
3) Add **SplitInBatches** with `batchSize=1`.
4) Add **HTTP Request** to GET `projects/<id>/memberships.json` (Redmine auth).
5) Add **Code** to extract `memberships[].user.id` into an array.
6) Add **Code** to:
   - collect all IDs across projects,
   - dedupe,
   - remove the originating `redmine_userID`,
   - output `{ redmine_projectTeamMemberIds: [{id:...}, ...] }`
7) Ensure the final node is connected as the workflow output (so the caller receives `redmine_projectTeamMemberIds`).

---

### 4.4 Create Sub-workflow: “Subflow push message to member”
1) New workflow → add **Execute Workflow Trigger** (expects input fields such as `type`, `memberID`, `title`, `date_from`, `date_to`, `link`).
2) Add **Wait** node (1s) if you need throttling.
3) Add **Switch** on `type`:
   - `timeoff` route
   - `remote` route  
4) For each route, add **HTTP Request**:
   - POST `https://slack.com/api/chat.postMessage`
   - Bearer token auth
   - Body includes `channel={{memberID}}` and formatted `text`.
5) Optionally set `onError=continueRegularOutput` to avoid failing the whole batch when one DM fails.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Subflows don't need to be ‘Active,’ only the main flow… main flow ‘Active’ will be able to call the subflow.” | Sticky note: Subflow Get membership list of user in Redmine |
| “Need to add trigger node ‘When Executed by Another Workflow’” | Sticky note: Subflow Push message to member |
| Consulting/support contact | [Linkedin](https://www.linkedin.com/company/bac-ha-software/posts/?feedView=all) / [Website](https://bachasoftware.com/bhsoft-contacts) |
| Intended outcome summary and suggested extensions (calendar sync, auditing, backup owners, digest, etc.) | Sticky note block describing “Results / Take it further” |

