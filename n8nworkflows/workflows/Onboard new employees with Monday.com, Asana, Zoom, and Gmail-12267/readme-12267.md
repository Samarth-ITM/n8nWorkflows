Onboard new employees with Monday.com, Asana, Zoom, and Gmail

https://n8nworkflows.xyz/workflows/onboard-new-employees-with-monday-com--asana--zoom--and-gmail-12267


# Onboard new employees with Monday.com, Asana, Zoom, and Gmail

## 1. Workflow Overview

**Purpose:** Automate employee onboarding using **Monday.com** as the source of truth. On a schedule, the workflow finds employees whose status is **“Joined”**, skips those already **“Processed”**, then:
1) creates an onboarding task in **Asana**,  
2) sends a welcome email via **Gmail**,  
3) schedules a **Zoom** intro meeting,  
4) writes the Zoom join link back to **Monday.com**, and  
5) marks the employee as **Processed** to prevent duplicates.

**Primary use cases:**
- HR/People Ops onboarding automation
- Ensuring consistent onboarding steps (task checklist + comms + meeting)
- Preventing reprocessing using a “Processed” marker in Monday.com

### 1.1 Trigger & Employee Retrieval (Monday.com)
Runs every minute and fetches “Joined” employees from a specific Monday board, then filters out already processed items.

### 1.2 Task Creation & Welcome Email (Asana + Gmail)
Creates a standardized onboarding task in Asana (with notes/checklist text), then emails the employee.

### 1.3 Intro Meeting & Monday Updates (Zoom + Monday.com)
Creates a Zoom meeting for the employee, stores the join URL into Monday.com, and marks the employee as processed.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Data Retrieval
**Overview:** Starts on a schedule, pulls Monday.com board items with status “Joined”, and filters out items already marked as processed.

**Nodes Involved:**
- Check for newly joined employees
- Fetch joined employees from Monday.com
- Filter – Skip Already Processed Employees

#### Node: **Check for newly joined employees**
- **Type / Role:** Schedule Trigger — periodically starts the workflow.
- **Config (interpreted):** Runs every **1 minute**.
- **Inputs / Outputs:**  
  - Input: none (trigger)  
  - Output → **Fetch joined employees from Monday.com**
- **Version considerations:** `typeVersion 1.3` (standard schedule trigger behavior).
- **Failure/edge cases:**
  - Excessively frequent polling may hit external API rate limits downstream (Monday/Asana/Gmail/Zoom), especially if the “Joined” list is large.

#### Node: **Fetch joined employees from Monday.com**
- **Type / Role:** Monday.com node — retrieves board items by a column value.
- **Config (interpreted):**
  - **Operation:** Get board items by column value
  - **Board ID:** `5025720609`
  - **Column ID:** `project_status`
  - **Column Value filter:** `Joined`
  - **Return all:** enabled
- **Credentials:** “Monday.com account 3” (Monday.com API token/connection).
- **Outputs:**  
  - Output → **Filter – Skip Already Processed Employees**
- **Key data used later:**
  - `{{$json.id}}` (the Monday item ID)
  - `{{$json.column_values[...]}}` (used throughout for name/email/status)
- **Failure/edge cases:**
  - Auth failure (expired token / revoked access).
  - Wrong boardId/columnId/columnValue → returns empty set.
  - Column structure changes (index positions in `column_values` shift) can break downstream expressions.

#### Node: **Filter – Skip Already Processed Employees**
- **Type / Role:** Filter — removes items already processed.
- **Config (interpreted):**
  - Condition: `{{$json.column_values[7].text}} != "Processed"`
  - Strict type validation enabled (Filter v3 condition model).
- **Inputs / Outputs:**  
  - Input from **Fetch joined employees from Monday.com**  
  - Output → **Create onboarding task in Asana**
- **Failure/edge cases:**
  - **Brittle column indexing:** using `column_values[7]` assumes the “Processed” marker is always at index 7. If Monday columns are reordered/added/removed, this may point to the wrong column and incorrectly process/skip employees.
  - If `column_values[7]` is missing/undefined, the expression may evaluate unexpectedly, potentially letting already-processed items through.

**Sticky note coverage (block context):**
- “Trigger & Data Retrieval”: *Scheduling and employee detection. This workflow runs on a schedule and fetches employees from Monday.com whose status is marked as Joined and not yet processed.*

---

### Block 2 — Task Creation & Welcome Email
**Overview:** For each eligible employee, creates an onboarding task in Asana and sends a personalized welcome email via Gmail.

**Nodes Involved:**
- Create onboarding task in Asana
- Send welcome email

#### Node: **Create onboarding task in Asana**
- **Type / Role:** Asana node — creates a task with standardized onboarding notes.
- **Config (interpreted):**
  - **Authentication:** OAuth2
  - **Workspace:** `1205967598025927`
  - **Task name:** `Onboarding – {{ $json.column_values[1].text }}`
  - **Notes:** onboarding checklist template embedding employee name:
    - Uses `{{$json.column_values[1].text}}` for employee name
  - **Due date:** `{{$now}}` (today; Asana expects a date; n8n provides a timestamp—usually acceptable but can cause formatting issues depending on node behavior)
  - **Assignee:** `1212557659831213`
  - **Project:** `1212555032005130`
- **Inputs / Outputs:**  
  - Input from **Filter – Skip Already Processed Employees**  
  - Output → **Send welcome email**
- **Credentials:** “Asana account 2”.
- **Failure/edge cases:**
  - Asana OAuth token expiration / missing scopes.
  - Invalid workspace/project/assignee IDs.
  - If `column_values[1].text` is empty or not the employee name, task naming and notes become incorrect (again due to column index fragility).
  - Due date formatting: if Asana rejects the provided value, task creation can fail.

#### Node: **Send welcome email**
- **Type / Role:** Gmail node — sends a text welcome email to the employee.
- **Config (interpreted):**
  - **To:** `{{ $('Fetch joined employees from Monday.com').item.json.column_values[6].text }}`
  - **Subject:** `Welcome to the Team,{{ ...column_values[1].text }}! 🎉`
  - **Body:** text email referencing employee name from Monday.
  - **Email type:** text
- **Inputs / Outputs:**  
  - Input from **Create onboarding task in Asana**  
  - Output → **Schedule Zoom intro meeting**
- **Credentials:** “Gmail credentials” (OAuth2).
- **Key expressions:**
  - Uses `$('Fetch joined employees from Monday.com').item.json...` to reference the original Monday item.
- **Failure/edge cases:**
  - Gmail OAuth token expiration / insufficient permissions.
  - Invalid/empty email address in `column_values[6].text`.
  - Subject contains a special character (🎉). Usually fine, but in rare cases can cause encoding issues in some systems.
  - If there are multiple items, relying on `$('Fetch...').item` works in n8n paired-item context; if pairing breaks (e.g., merges, splits, or non-standard item linking), the wrong employee email/name could be used.

**Sticky note coverage (block context):**
- “Task Creation”: *Creates an onboarding checklist task in Asana and sends a personalized welcome email to the new employee.*

---

### Block 3 — Intro Meeting & Update (Zoom + Monday.com)
**Overview:** Schedules a Zoom meeting, then writes the meeting join link back to the employee’s Monday item and marks them processed.

**Nodes Involved:**
- Schedule Zoom intro meeting
- Update Employee Zoom Link
- Mark employee as processed

#### Node: **Schedule Zoom intro meeting**
- **Type / Role:** Zoom node — creates a meeting for the new employee’s intro call.
- **Config (interpreted):**
  - **Authentication:** OAuth2
  - **Topic:** `Welcome & Team Introduction – {{ employee name }}`
    - Name pulled from: `$('Fetch joined employees from Monday.com').item.json.column_values[1].text`
  - **Duration:** 30 minutes
  - **Start time:** `new Date(Date.now() + 86400000).toISOString()` (exactly **24 hours from execution time**)
- **Inputs / Outputs:**  
  - Input from **Send welcome email**  
  - Output → **Update Employee Zoom Link** and → **Mark employee as processed** (both in parallel)
- **Credentials:** “Zoom account”.
- **Failure/edge cases:**
  - Zoom OAuth token expiration / missing scopes.
  - Timezone expectations: ISO string is UTC; the meeting may appear at an unexpected local time unless Zoom/user settings handle it as intended.
  - If Zoom API returns no `join_url` (or changes response shape), downstream Monday update will fail.

#### Node: **Update Employee Zoom Link**
- **Type / Role:** Monday.com node — writes Zoom join URL into a link column.
- **Config (interpreted):**
  - **Operation:** Change a column value on a board item
  - **Board ID:** `5025720609`
  - **Item ID:** `{{ $('Fetch joined employees from Monday.com').item.json.id }}`
  - **Column ID:** `link_mkyxc6hk`
  - **Value (link object):**
    - `url`: `{{$json.join_url}}` (from Zoom node output)
    - `text`: `"Join Intro Call"`
- **Inputs / Outputs:**  
  - Input from **Schedule Zoom intro meeting**
  - No outgoing connections
- **Credentials:** “Monday.com account 3”.
- **Failure/edge cases:**
  - If `join_url` is absent/undefined → expression failure or Monday rejects the value.
  - Column type mismatch: `link_mkyxc6hk` must be a Link column; otherwise Monday rejects JSON.
  - Item reference mismatch if pairing context breaks (could update wrong employee).

#### Node: **Mark employee as processed**
- **Type / Role:** Monday.com node — sets a status/label column to “Processed”.
- **Config (interpreted):**
  - **Operation:** Change a column value on a board item
  - **Board ID:** `5025720609`
  - **Item ID:** `{{ $('Fetch joined employees from Monday.com').item.json.id }}`
  - **Column ID:** `color_mkz3c6wx`
  - **Value:** `{ "label": "Processed" }`
- **Inputs / Outputs:**  
  - Input from **Schedule Zoom intro meeting**
  - No outgoing connections
- **Credentials:** “Monday.com account 3”.
- **Failure/edge cases:**
  - Column must support labels/status with label “Processed” existing; otherwise Monday rejects it.
  - Because it runs in parallel with “Update Employee Zoom Link”, an execution could mark processed even if the Zoom link update fails (no transactional guarantee).

**Sticky note coverage (block context):**
- “Intro Meeting & Update”: *Schedules a 30-minute Zoom introductory meeting; updates Zoom link and marks employee as processed in Monday.com.*

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / overview | — | — | ## Onboard new employees using Zoom, Asana, and Monday.com - Workflow Overview … (setup steps and explanation) |
| Check for newly joined employees | Schedule Trigger | Entry point; periodic polling | — | Fetch joined employees from Monday.com | ### Scheduling and employee detection |
| Fetch joined employees from Monday.com | Monday.com | Query board items where status=Joined | Check for newly joined employees | Filter – Skip Already Processed Employees | ### Scheduling and employee detection |
| Filter – Skip Already Processed Employees | Filter | Prevent duplicate processing | Fetch joined employees from Monday.com | Create onboarding task in Asana | ### Scheduling and employee detection |
| Create onboarding task in Asana | Asana | Create onboarding task + checklist notes | Filter – Skip Already Processed Employees | Send welcome email | ### Task Creation and Welcome Email |
| Send welcome email | Gmail | Email the new employee | Create onboarding task in Asana | Schedule Zoom intro meeting | ### Task Creation and Welcome Email |
| Schedule Zoom intro meeting | Zoom | Create intro meeting | Send welcome email | Update Employee Zoom Link; Mark employee as processed | ### Intro Meeting and Updates |
| Update Employee Zoom Link | Monday.com | Write Zoom join URL to employee item | Schedule Zoom intro meeting | — | ### Intro Meeting and Updates |
| Mark employee as processed | Monday.com | Set Processed status to avoid reruns | Schedule Zoom intro meeting | — | ### Intro Meeting and Updates |
| Trigger & Data Retrieval | Sticky Note | Documentation / block label | — | — | ### Scheduling and employee detection |
| Task Creation | Sticky Note | Documentation / block label | — | — | ### Task Creation and Welcome Email |
| Intro Meeting & Update | Sticky Note | Documentation / block label | — | — | ### Intro Meeting and Updates |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Onboard new employees using Zoom, Asana, and Monday.com”** (or your preferred name).
   - Ensure workflow **Execution Order** is set to **v1** (Workflow settings).

2. **Add Trigger: Schedule Trigger**
   - Node: **Schedule Trigger** named **“Check for newly joined employees”**
   - Set interval: **Every 1 minute**.

3. **Add Monday.com: Get items by column value**
   - Node: **Monday.com** named **“Fetch joined employees from Monday.com”**
   - Credentials: connect Monday.com (API token connection).
   - Operation: **Board Item → Get By Column Value**
   - Configure:
     - **Board ID:** `5025720609` (replace with your board)
     - **Column ID:** `project_status` (replace with your status column ID)
     - **Column Value:** `Joined`
     - **Return All:** enabled
   - Connect: Schedule Trigger → Fetch node.

4. **Add Filter: Skip processed**
   - Node: **Filter** named **“Filter – Skip Already Processed Employees”**
   - Condition (string):  
     - Left value: `{{$json.column_values[7].text}}`  
     - Operator: **not equals**  
     - Right value: `Processed`
   - Connect: Fetch node → Filter node  
   - Note: Prefer using a safer reference (by column ID) if you later modify the workflow; this workflow uses a fixed index.

5. **Add Asana: Create task**
   - Node: **Asana** named **“Create onboarding task in Asana”**
   - Credentials: Asana OAuth2 connection.
   - Workspace: `1205967598025927` (or yours)
   - Operation: **Task → Create**
   - Key fields:
     - **Name:** `Onboarding – {{ $json.column_values[1].text }}`
     - **Notes:** include your checklist template; reference `{{$json.column_values[1].text}}` for employee name
     - **Due on:** `{{$now}}`
     - **Assignee:** `1212557659831213` (or your assignee)
     - **Projects:** add `1212555032005130` (or your project)
   - Connect: Filter node → Asana node.

6. **Add Gmail: Send email**
   - Node: **Gmail** named **“Send welcome email”**
   - Credentials: Gmail OAuth2 connection.
   - Operation: **Send**
   - Configure:
     - **To:** `{{ $('Fetch joined employees from Monday.com').item.json.column_values[6].text }}`
     - **Subject:** `Welcome to the Team,{{ $('Fetch joined employees from Monday.com').item.json.column_values[1].text }}! 🎉`
     - **Message (text):** personalized welcome text (as in workflow)
     - **Email type:** text
   - Connect: Asana node → Gmail node.

7. **Add Zoom: Create meeting**
   - Node: **Zoom** named **“Schedule Zoom intro meeting”**
   - Credentials: Zoom OAuth2 connection.
   - Operation: **Meeting → Create** (as provided by the Zoom node)
   - Configure:
     - **Topic:** `Welcome & Team Introduction – {{ $('Fetch joined employees from Monday.com').item.json.column_values[1].text }}`
     - **Duration:** 30
     - **Start Time:** `{{ new Date(Date.now() + 86400000).toISOString() }}`
   - Connect: Gmail node → Zoom node.

8. **Add Monday.com: Update Zoom link column**
   - Node: **Monday.com** named **“Update Employee Zoom Link”**
   - Credentials: same Monday connection as above.
   - Operation: **Board Item → Change Column Value**
   - Configure:
     - **Board ID:** `5025720609`
     - **Item ID:** `{{ $('Fetch joined employees from Monday.com').item.json.id }}`
     - **Column ID:** `link_mkyxc6hk` (must be a Link column)
     - **Value:**  
       - url: `{{ $json.join_url }}`
       - text: `Join Intro Call`
   - Connect: Zoom node → Update Employee Zoom Link.

9. **Add Monday.com: Mark as processed**
   - Node: **Monday.com** named **“Mark employee as processed”**
   - Operation: **Board Item → Change Column Value**
   - Configure:
     - **Board ID:** `5025720609`
     - **Item ID:** `{{ $('Fetch joined employees from Monday.com').item.json.id }}`
     - **Column ID:** `color_mkz3c6wx` (a Status/Label column)
     - **Value:** label = `Processed` (must exist as an option)
   - Connect: Zoom node → Mark employee as processed (parallel branch).

10. **(Optional) Add documentation sticky notes**
   - Add sticky notes with the provided block labels and the “Workflow Overview” text for maintainability.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Onboard new employees using Zoom, Asana, and Monday.com” overview + setup steps (connect Monday.com, configure Asana, Gmail, Zoom, adjust schedule) | From the “Workflow Overview” sticky note |
| Scheduling and employee detection: runs on a schedule; fetches Joined and not yet processed employees | From “Trigger & Data Retrieval” sticky note |
| Creates an onboarding checklist task in Asana and sends a personalized welcome email | From “Task Creation” sticky note |
| Schedules a 30-minute Zoom meeting; updates Zoom link and marks processed in Monday.com | From “Intro Meeting & Update” sticky note |