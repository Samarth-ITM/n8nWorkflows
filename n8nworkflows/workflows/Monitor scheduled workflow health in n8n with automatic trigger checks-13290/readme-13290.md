Monitor scheduled workflow health in n8n with automatic trigger checks

https://n8nworkflows.xyz/workflows/monitor-scheduled-workflow-health-in-n8n-with-automatic-trigger-checks-13290


# Monitor scheduled workflow health in n8n with automatic trigger checks

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Monitor scheduled workflow health in n8n with automatic trigger checks  
**Workflow name (internal):** Monitor - Scheduled Workflow Health

This workflow monitors the “health” of other workflows that are expected to run on a schedule (Schedule Trigger) or via polling-based triggers (e.g., connectors that poll on an interval). It automatically discovers eligible workflows, computes an allowed “maximum age” since last execution, fetches the latest execution per workflow, and raises an error if any workflow appears stale (overdue). The raised error is intended to be captured by an **Error Workflow** for alerting (Slack/email/etc.).

### Logical blocks
1. **1.1 Triggering / Invocation**
   - Runs daily (intended 9am) and supports a manual test run.
2. **1.2 Discovery of monitorable workflows**
   - Fetches all active workflows via n8n API and filters down to those with schedule or polling triggers; computes allowed max age.
3. **1.3 Execution freshness evaluation**
   - For each discovered workflow, fetches its latest execution and determines whether it is stale.
4. **1.4 Alerting outcome**
   - If any stale workflows are found, the workflow fails with a formatted error message; otherwise it ends successfully.

---

## 2. Block-by-Block Analysis

### 2.1 Triggering / Invocation
**Overview:** Starts the monitoring run either on a schedule or manually for testing. Both entry points lead into the same discovery step.

**Nodes involved:**
- **Daily Check at 9am**
- **Test Run**

#### Node: Daily Check at 9am
- **Type / role:** Schedule Trigger — initiates the workflow automatically.
- **Configuration (interpreted):** Runs on an interval defined as “hours”.  
  - Note: despite the node name “Daily Check at 9am”, the configured rule shown is an hourly interval (`field: "hours"`). If you truly want 9am daily, this should be configured as a daily schedule or cron expression (see Section 4).
- **Inputs / outputs:** Entry node → outputs to **Fetch All Active Workflows**.
- **Potential failures / edge cases:** None directly; misconfiguration risk (hourly vs daily).

#### Node: Test Run
- **Type / role:** Manual Trigger — allows interactive runs from the editor.
- **Configuration:** Default manual trigger (no parameters).
- **Inputs / outputs:** Entry node → outputs to **Fetch All Active Workflows**.
- **Potential failures / edge cases:** None.

---

### 2.2 Discovery of monitorable workflows
**Overview:** Queries n8n for workflows, then auto-detects which ones are schedule- or polling-triggered and computes a “max allowed age” (in hours) since last execution, with generous safety margins.

**Nodes involved:**
- **Fetch All Active Workflows**
- **Discover Scheduled Workflows**

#### Node: Fetch All Active Workflows
- **Type / role:** n8n node (n8n API) — lists workflows from the same n8n instance.
- **Configuration (interpreted):**
  - **Resource:** Workflows (default for “n8n” node when listing workflows; the JSON shows empty `filters: {}`).
  - **Filters:** None explicitly set here; it will fetch available workflows the credential can access. The sticky note describes “active workflows”, but the node itself doesn’t show an “active only” filter in the provided JSON—so the returned list may include more and relies on later filtering.
  - **Credentials required:** n8n API credential (see sticky note setup).
- **Key data used later:** Each workflow object’s fields like `id`, `name`, `isArchived`, `tags`, `nodes`, and `shared[0].projectId`.
- **Inputs / outputs:**
  - Input: from **Daily Check at 9am** or **Test Run**
  - Output: to **Discover Scheduled Workflows**
- **Potential failures / edge cases:**
  - Authentication/authorization failures (API key invalid, insufficient permissions).
  - Large instances: many workflows could increase runtime.
  - API pagination behavior (depends on node defaults); if paginated and not auto-handled, you might not scan all workflows.

#### Node: Discover Scheduled Workflows
- **Type / role:** Code node — filters workflows and computes expected freshness thresholds.
- **Configuration (interpreted):**
  - **Optional project scoping:** `PROJECT_ID = ''` (empty means “monitor all projects”).  
    - If set, it checks `wf.shared?.[0]?.projectId` and only includes matches.
  - **Skip tag:** `SKIP_TAG = 'skip-monitoring'`  
    - It lowercases workflow tag names and excludes workflows containing that tag.
  - **Eligibility logic (auto-discovery):**
    - Excludes archived workflows (`wf.isArchived`).
    - Includes workflows that have either:
      - A node of type `n8n-nodes-base.scheduleTrigger`, **or**
      - A “polling trigger”: any node whose type includes `Trigger`, is not one of:
        - manualTrigger
        - executeWorkflowTrigger
        - errorTrigger
        - scheduleTrigger  
        and has `n.parameters?.pollTimes` (used as heuristic for polling-based triggers).
  - **Max age computation (hours):**
    - Default `maxAgeHours = 48`.
    - If schedule trigger exists, inspects the first interval rule: `scheduleTrigger.parameters?.rule?.interval?.[0]`:
      - `cronExpression`: uses `parseCronMaxAge(expression)`
      - `minutes`: estimates based on `minutesInterval || 30` and allows `mins*8` worth of delay (minimum 2 hours), converted to hours.
      - `hours`: sets 48 hours
      - `weeks`: 192 hours (8 days)
      - `months`: 840 hours (35 days)
      - otherwise: 48
    - `parseCronMaxAge()` heuristics:
      - Monthly (day-of-month fixed): 840h
      - Any day-of-week restriction: 48h (covers weekends)
      - Every N days (`*/N` in day-of-month): `max(48, N*36)`
      - Sub-hourly (`*/` in minutes): 48h
      - Daily default: 48h
  - **Output format:** Emits items like:
    - `{ workflowId: wf.id, name: wf.name, maxAgeHours }`
- **Inputs / outputs:**
  - Input: list of workflow JSON objects from **Fetch All Active Workflows**
  - Output: to **Get Latest Execution** (one item per discovered workflow)
- **Potential failures / edge cases:**
  - Workflow JSON shapes can vary across n8n versions (e.g., location of project shares or tags). The code uses optional chaining to reduce crashes, but `wf.shared?.[0]?.projectId` may not match your environment.
  - Only the **first** schedule interval entry is considered (`interval[0]`); complex schedules may be mis-estimated.
  - Polling triggers are detected via `pollTimes`; not all polling triggers may expose this consistently.
  - Cron parsing is heuristic and intentionally permissive; it may not catch all “missed schedule” scenarios (by design, fewer false positives).

---

### 2.3 Execution freshness evaluation
**Overview:** For each discovered workflow, queries the latest execution and compares its start time against the computed `maxAgeHours`.

**Nodes involved:**
- **Get Latest Execution**
- **Check for Stale Workflows**

#### Node: Get Latest Execution
- **Type / role:** n8n node (n8n API) — fetches executions.
- **Configuration (interpreted):**
  - **Resource:** Execution
  - **Operation:** List / search executions
  - **Filter:** `workflowId` set via expression: `={{ $json.workflowId }}`
  - **Limit:** 1 (returns only the latest execution by whatever ordering the API uses for “list”; typically newest first)
  - **Error handling:** `onError: continueRegularOutput` and `alwaysOutputData: true`
    - This ensures the workflow continues even if the execution lookup fails for a given workflow.
- **Inputs / outputs:**
  - Input: from **Discover Scheduled Workflows** (one item per monitored workflow)
  - Output: to **Check for Stale Workflows**
- **Potential failures / edge cases:**
  - Auth/permission issues (can’t read executions).
  - Instances with execution pruning: “no execution found” is expected for never-run workflows or purged histories.
  - API ordering assumptions: if the API doesn’t guarantee newest-first, “latest” might not always be correct (rare, but possible depending on implementation).

#### Node: Check for Stale Workflows
- **Type / role:** Code node — merges discovery config with execution results and determines staleness.
- **Configuration (interpreted):**
  - Reads executions from `$input.all()`.
  - Reads discovery config from `$('Discover Scheduled Workflows').all()`.
  - Builds a map: `workflowId -> latest execution`.
  - For each discovered workflow:
    - If no execution or missing `startedAt`: mark stale with reason `No execution found`.
    - Else compute age in hours from `startedAt` to now.
    - If `ageHours > maxAgeHours`: stale with reason `Last ran Xh ago (max: Yh)` and includes `lastRun`.
    - Otherwise mark OK.
  - Outputs a single summary item:
    - `{ stale: [...], staleCount, okCount, checkedCount }`
- **Inputs / outputs:**
  - Input: execution item(s) from **Get Latest Execution**
  - Output: to **Any Stale?**
- **Potential failures / edge cases:**
  - If multiple executions are returned (unexpected with limit=1), last one written to the map wins.
  - Timezone is irrelevant because it compares timestamps; however, clock skew on server could affect results.
  - If `startedAt` is absent but execution exists (unusual), it is treated as stale.

---

### 2.4 Alerting outcome
**Overview:** If any stale workflows exist, the workflow intentionally errors with a human-readable list. Otherwise, it ends with a no-op “healthy” branch.

**Nodes involved:**
- **Any Stale?**
- **Alert — Workflows Missed Schedule**
- **All Healthy**

#### Node: Any Stale?
- **Type / role:** IF node — branch based on stale count.
- **Configuration (interpreted):**
  - Condition: `{{$json.staleCount}} > 0` (numeric greater-than).
- **Inputs / outputs:**
  - Input: from **Check for Stale Workflows**
  - Output (true): to **Alert — Workflows Missed Schedule**
  - Output (false): to **All Healthy**
- **Potential failures / edge cases:**
  - If `staleCount` is missing or not numeric, strict validation could fail; current upstream always sets it.

#### Node: Alert — Workflows Missed Schedule
- **Type / role:** Stop and Error — intentionally fails the workflow to trigger centralized error handling.
- **Configuration (interpreted):**
  - Error message built from expressions:
    - Header: `{{ $json.staleCount }} scheduled workflow(s) missed their schedule:`
    - Bullet list: `{{ $json.stale.map(...).join('\n') }}`
- **Inputs / outputs:** Input from IF(true). No outputs (execution stops).
- **Potential failures / edge cases:**
  - Very long error message if many stale workflows; some alert channels may truncate.
- **Sub-workflow reference:** Not a node-level subworkflow, but intended integration: **n8n “Error Workflow”** configured in workflow settings to send Slack/email.

#### Node: All Healthy
- **Type / role:** NoOp — placeholder “success” end state.
- **Configuration:** None.
- **Inputs / outputs:** Input from IF(false). No outputs.
- **Potential failures / edge cases:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Test Run | Manual Trigger | Manual entry point for testing | — | Fetch All Active Workflows | ## Monitor — Scheduled Workflow Health (auto-detect stale scheduled/polling workflows; setup: add n8n API credential; tag this workflow `skip-monitoring`; optional `PROJECT_ID`; connect Error Workflow; activate; customizable schedule/SKIP_TAG/max-age margins) |
| Daily Check at 9am | Schedule Trigger | Automated entry point | — | Fetch All Active Workflows | ## Monitor — Scheduled Workflow Health (auto-detect stale scheduled/polling workflows; setup: add n8n API credential; tag this workflow `skip-monitoring`; optional `PROJECT_ID`; connect Error Workflow; activate; customizable schedule/SKIP_TAG/max-age margins) |
| Fetch All Active Workflows | n8n | List workflows from n8n API | Test Run; Daily Check at 9am | Discover Scheduled Workflows | ## Discovery |
| Discover Scheduled Workflows | Code | Filter eligible workflows; compute maxAgeHours | Fetch All Active Workflows | Get Latest Execution | ## Discovery |
| Get Latest Execution | n8n | Fetch most recent execution per workflow | Discover Scheduled Workflows | Check for Stale Workflows | ## Staleness Check |
| Check for Stale Workflows | Code | Compare last execution age vs threshold | Get Latest Execution | Any Stale? | ## Staleness Check |
| Any Stale? | IF | Branch on staleCount > 0 | Check for Stale Workflows | Alert — Workflows Missed Schedule; All Healthy | ## Alert or OK |
| Alert — Workflows Missed Schedule | Stop and Error | Fail workflow with formatted stale list | Any Stale? (true) | — | ## Alert or OK |
| All Healthy | NoOp | Successful end branch | Any Stale? (false) | — | ## Alert or OK |
| Sticky Note | Sticky Note | Documentation panel | — | — |  |
| Sticky Note1 | Sticky Note | “Discovery” label | — | — |  |
| Sticky Note2 | Sticky Note | “Staleness Check” label | — | — |  |
| Sticky Note3 | Sticky Note | “Alert or OK” label | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **Monitor - Scheduled Workflow Health**
   - Workflow settings: keep default execution order (`v1`) unless you have a reason to change.

2. **Add entry nodes**
   1) Add **Manual Trigger** node named **Test Run**.  
   2) Add **Schedule Trigger** node named **Daily Check at 9am**.
      - If you want actual “daily at 9am”, set the schedule accordingly (e.g., daily at 09:00) or use a cron expression like `0 9 * * *`.
      - (The provided JSON currently represents an hourly interval despite the name.)

3. **Add “Fetch All Active Workflows”**
   - Node type: **n8n**
   - Name: **Fetch All Active Workflows**
   - Operation: list workflows (workflows endpoint in the n8n node).
   - **Credentials:** configure an **n8n API** credential that can read workflows and executions on this instance.
   - Connect:
     - **Test Run → Fetch All Active Workflows**
     - **Daily Check at 9am → Fetch All Active Workflows**

4. **Add “Discover Scheduled Workflows” (Code)**
   - Node type: **Code**
   - Name: **Discover Scheduled Workflows**
   - Paste the logic that:
     - optionally filters by `PROJECT_ID`
     - excludes workflows tagged `skip-monitoring`
     - detects schedule triggers and polling triggers
     - computes `maxAgeHours`
     - returns items `{ workflowId, name, maxAgeHours }`
   - Connect: **Fetch All Active Workflows → Discover Scheduled Workflows**
   - Optional configuration changes in the code:
     - Set `PROJECT_ID` to restrict monitoring to a single project.
     - Change `SKIP_TAG` if you prefer another exclusion tag name.

5. **Add “Get Latest Execution”**
   - Node type: **n8n**
   - Name: **Get Latest Execution**
   - Resource: **Execution**
   - Operation: list/search executions
   - Filter: `workflowId = {{$json.workflowId}}`
   - Limit: `1`
   - Enable:
     - **Continue On Fail** (equivalent to `onError: continueRegularOutput`)
     - **Always Output Data**
   - Credentials: same **n8n API** credential (must have access to execution history).
   - Connect: **Discover Scheduled Workflows → Get Latest Execution**

6. **Add “Check for Stale Workflows” (Code)**
   - Node type: **Code**
   - Name: **Check for Stale Workflows**
   - Implement logic to:
     - build a map of `workflowId -> execution`
     - read the discovery list via `$('Discover Scheduled Workflows').all()`
     - compute `stale[]`, `staleCount`, `okCount`, `checkedCount`
     - output one summary item
   - Connect: **Get Latest Execution → Check for Stale Workflows**

7. **Add branching “Any Stale?”**
   - Node type: **IF**
   - Name: **Any Stale?**
   - Condition: number comparison `{{$json.staleCount}} > 0`
   - Connect: **Check for Stale Workflows → Any Stale?**

8. **Add alerting and success end nodes**
   1) Node type: **Stop and Error**
      - Name: **Alert — Workflows Missed Schedule**
      - Error message expression:
        - `{{$json.staleCount}} scheduled workflow(s) missed their schedule:\n{{$json.stale.map(s => '• ' + s.name + ' — ' + s.reason).join('\n')}}`
   2) Node type: **No Operation (NoOp)**
      - Name: **All Healthy**
   - Connect:
     - **Any Stale? (true) → Alert — Workflows Missed Schedule**
     - **Any Stale? (false) → All Healthy**

9. **Instance-level alerting (recommended)**
   - In this workflow’s settings, configure an **Error Workflow**.
   - The Error Workflow can forward the error message (from Stop and Error) to Slack, email, etc.

10. **Exclude this monitor workflow from monitoring**
   - Add tag: **skip-monitoring** to *this* workflow (or change `SKIP_TAG` to match your preference).

11. **Activate**
   - Activate the workflow after verifying the schedule and credentials.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically detects when scheduled or polling-trigger workflows stop running; no hardcoded config needed; computes expected run frequency from trigger settings; raises an error for stale workflows; intended to pair with an Error Workflow for notifications. | Sticky note: “Monitor — Scheduled Workflow Health” |
| Setup: add n8n API credential to both n8n nodes; tag this workflow `skip-monitoring`; optionally set `PROJECT_ID` in the discovery code; connect an Error Workflow; activate. | Sticky note: “Monitor — Scheduled Workflow Health” |
| Customization: change the schedule; change `SKIP_TAG`; max-age calculation uses generous margins (e.g., 48h for daily to survive weekends). | Sticky note: “Monitor — Scheduled Workflow Health” |