Sync workflow schedules between Google Sheets and Google Calendar

https://n8nworkflows.xyz/workflows/sync-workflow-schedules-between-google-sheets-and-google-calendar-14397


# Sync workflow schedules between Google Sheets and Google Calendar

# 1. Workflow Overview

This workflow scans n8n workflows on a schedule, extracts schedulable trigger configurations, compares the current schedule state with a persisted state in Google Sheets, and synchronizes recurring representations of those schedules into Google Calendar.

Its main purpose is operational visibility: each active n8n workflow with a supported schedule becomes a recurring Google Calendar event, allowing teams to inspect automation cadence in a calendar view. It also tracks synchronization state in Google Sheets to detect when a workflow schedule is new, changed, or unchanged.

## 1.1 Scheduled Execution Entry

The main pipeline starts every 30 minutes via a Schedule Trigger. It then calls the n8n API to list workflows.

## 1.2 Workflow Schedule Extraction and Normalization

Fetched workflows are parsed to identify supported trigger nodes (`scheduleTrigger` and `cron`), convert their scheduling configuration into normalized fields, and generate a human-readable schedule string.

## 1.3 State Lookup and Change Detection

The workflow snapshots the current parsed values, reads the existing persisted state from Google Sheets, manually merges live and saved data by `WorkflowID`, and determines whether each workflow should be created in Calendar, updated, or skipped.

## 1.4 Calendar Creation / Update Writeback

For `create` and `update` actions, the workflow builds Google Calendar recurrence payloads for supported schedule types, creates recurring events, aggregates resulting event IDs, and writes the new canonical sync state back to Google Sheets.

## 1.5 Old Calendar Event Deletion on Update

For `update` actions, the workflow also runs a parallel delete branch that splits previously stored Calendar event IDs and deletes old events from Google Calendar.

## 1.6 Disconnected Maintenance Sub-flow

A separate, disabled webhook-based branch exists for manual Calendar cleanup. It is not connected to the scheduled sync pipeline.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Execution Entry

**Overview:**  
This block launches the main synchronization cycle every 30 minutes. It is the only active entry point for the main workflow.

**Nodes Involved:**  
- Schedule Trigger  
- Get many workflows

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry trigger.
- **Configuration choices:** Configured with an interval rule that fires at minute 30. The sticky notes describe it as every 30 minutes (`*/30 * * * *`), but the raw node configuration shows only `triggerAtMinute: 30`. In practice, confirm the exact recurrence after importing, because the UI representation may resolve this into hourly-at-:30 rather than every 30 minutes depending on node version and editor behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to **Get many workflows**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Misconfigured interval semantics could cause a different execution cadence than intended. Always verify schedule behavior in the n8n UI after import.
- **Sub-workflow reference:** None.

#### Get many workflows
- **Type and technical role:** `n8n-nodes-base.n8n`; calls the n8n API to list workflows.
- **Configuration choices:** Uses filter `activeWorkflows: true`, despite some sticky notes saying active and inactive workflows are fetched. The actual node configuration fetches only active workflows.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Schedule Trigger**; output to **Code: parsing**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - n8n API authentication failure
  - insufficient API permissions
  - API endpoint/version mismatch
  - self-hosted instance network issues
- **Sub-workflow reference:** None.

---

## 2.2 Workflow Schedule Extraction and Normalization

**Overview:**  
This block extracts schedulable trigger information from each workflow and normalizes it into a standard schema. It also produces a human-readable schedule string used later as the primary comparison key.

**Nodes Involved:**  
- Code: parsing  
- Code: save current values

### Node Details

#### Code: parsing
- **Type and technical role:** `n8n-nodes-base.code`; parses workflow JSON and emits one item per schedulable trigger.
- **Configuration choices:**  
  - Includes `scheduleTrigger` and `cron`
  - Excludes `webhook`, `errorTrigger`, and `manualTrigger` unless the internal constant `INCLUDE_MANUAL` is changed to `true`
  - Supports many variants of scheduleTrigger schema and attempts inference when `field` is missing
  - Converts schedules into normalized fields: `freq`, `interval`, `atHour`, `atMinute`, `byDay`, `byMonthDay`, `timezone`
  - Builds a human string in Italian, such as:
    - `Ogni giorno 09:30`
    - `Ogni lun, mer 07:00`
    - `Ogni mese (giorno 15) 08:00`
    - `Ogni ora :45`
    - `Cron: ...`
- **Key expressions or variables used:**  
  - `$input.all()`
  - workflow-level accessors:
    - `wf.id`
    - `wf.name`
    - `wf.tags`
    - `wf.active`
    - `wf.settings?.timezone`
    - `wf.activeVersion?.nodes || wf.nodes`
  - output fields:
    - `WorkflowID`
    - `Workflow`
    - `Tags`
    - `Active`
    - `triggerType`
    - `freq`
    - `interval`
    - `atHour`
    - `atMinute`
    - `byDay`
    - `byMonthDay`
    - `timezone`
    - `schedule`
    - `Schedule-RAW`
- **Input and output connections:** Input from **Get many workflows**; output to **Code: save current values**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Unexpected workflow JSON shape
  - new n8n trigger schema variants not covered by parser
  - malformed trigger config causing inferred `unknown`
  - workflows with multiple schedulable triggers produce multiple output rows
  - cron schedules are parsed but later mostly skipped for Calendar creation
- **Sub-workflow reference:** None.

#### Code: save current values
- **Type and technical role:** `n8n-nodes-base.code`; snapshots live values into `current_*` fields before sheet data overwrites the base names.
- **Configuration choices:** Uses nullish checks (`??`) rather than `||` so valid numeric zero values like `0` are preserved.
- **Key expressions or variables used:**  
  - `current_schedule`
  - `current_freq`
  - `current_atHour`
  - `current_atMinute`
  - `current_byDay`
  - `current_byMonthDay`
  - `current_interval`
  - `current_WorkflowID`
- **Input and output connections:** Input from **Code: parsing**; output to **Sheets:Lookup-ExistOnCalendar**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Mainly expression or script mistakes if fields are renamed upstream.
- **Sub-workflow reference:** None.

---

## 2.3 State Lookup and Change Detection

**Overview:**  
This block reads the current persisted sync state from Google Sheets, merges it with the freshly parsed live schedule data, and determines whether each workflow requires Calendar creation, update, or no action.

**Nodes Involved:**  
- Sheets:Lookup-ExistOnCalendar  
- Code manual merge  
- Code: detect changes  
- Switch

### Node Details

#### Sheets:Lookup-ExistOnCalendar
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from the sheet used as canonical sync state.
- **Configuration choices:**  
  - Uses Google Sheets document `YOUR_SPREADSHEET_ID`
  - Uses tab `n8n Scheduling` with internal `sheetName` value `607954075`
  - Uses **service account** authentication
  - `alwaysOutputData: true` ensures the branch continues even when no rows exist
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Code: save current values**; output to **Code manual merge**.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - spreadsheet not shared with service account
  - missing sheet/tab
  - wrong spreadsheet ID
  - schema drift if expected columns are absent
- **Sub-workflow reference:** None.

#### Code manual merge
- **Type and technical role:** `n8n-nodes-base.code`; manually joins live items with sheet rows by `WorkflowID`.
- **Configuration choices:**  
  - Reads all live workflow items via `$('Code: save current values').all()`
  - Builds a map from sheet rows keyed by `WorkflowID`
  - Emits merged items preserving `current_*` fields while also restoring saved values into generic field names
  - Explicitly propagates `Calendar_EventID`, which is critical for the delete-on-update path
- **Key expressions or variables used:**  
  - `$('Code: save current values').all()`
  - merged fields:
    - `schedule`
    - `freq`
    - `atHour`
    - `atMinute`
    - `byDay`
    - `byMonthDay`
    - `interval`
    - `On Calendar`
    - `Calendar_EventID`
  - `_matchDebug`
- **Input and output connections:** Input from **Sheets:Lookup-ExistOnCalendar**; output to **Code: detect changes**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - duplicate rows in Sheets for same `WorkflowID` cause last-write-wins behavior
  - if current items are absent, output is empty
  - if sheet column names change, merge silently degrades
- **Sub-workflow reference:** None.

#### Code: detect changes
- **Type and technical role:** `n8n-nodes-base.code`; compares live schedule state vs saved sheet state and sets `action`.
- **Configuration choices:**  
  - Uses the human-readable `schedule` string as the canonical diff value
  - Unsupported types are defined by regexes:
    - minutely
    - every N minutes
    - cron
  - Decision logic:
    - new or unsynced workflow + supported schedule -> `create`
    - existing synced workflow + changed supported schedule -> `update`
    - unchanged -> `skip`
    - unsupported -> `skip`
- **Key expressions or variables used:**  
  - `current_schedule`
  - `schedule`
  - `On Calendar`
  - `WorkflowID`
  - output fields:
    - `action`
    - `scheduleChanged`
    - `changedFields`
    - `_debug`
- **Input and output connections:** Input from **Code manual merge**; output to **Switch**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - if two semantically equal schedules produce different human strings, false updates occur
  - unsupported schedules remain stale in Calendar when a previously supported schedule changes to unsupported
  - blank/malformed workflow IDs are skipped
- **Sub-workflow reference:** None.

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`; routes items according to `action`.
- **Configuration choices:** Three named outputs:
  - `create`
  - `update`
  - `skip`
- **Key expressions or variables used:** `={{ $json.action }}`
- **Input and output connections:**  
  - Input from **Code: detect changes**
  - `create` output -> **Remove Duplicates**
  - `update` output -> **Remove Duplicates** and **Code: split eventIds for delete**
  - `skip` output -> terminal
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - typos in `action` would route nowhere
  - strict type validation may reject unexpected value types
- **Sub-workflow reference:** None.

---

## 2.4 Calendar Creation / Update Writeback

**Overview:**  
This block handles both new schedules and updated schedules. It deduplicates items, transforms supported human-readable schedules into Google Calendar recurrence payloads, creates the recurring events, and persists resulting event IDs back into Google Sheets.

**Nodes Involved:**  
- Remove Duplicates  
- Code: RRULE  
- Create an event  
- Code: post-create  
- Sheets:OnCalendar=YES

### Node Details

#### Remove Duplicates
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`; defensive deduplication before Calendar creation.
- **Configuration choices:** Compares selected fields `WorkflowID,current_schedule`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Switch** create/update branches; output to **Code: RRULE**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - if a workflow intentionally has multiple distinct triggers, deduplication only collapses identical `(WorkflowID, current_schedule)` pairs
  - if two rows share ID and schedule but differ in other metadata, those differences are discarded
- **Sub-workflow reference:** None.

#### Code: RRULE
- **Type and technical role:** `n8n-nodes-base.code`; builds Calendar payloads for supported recurrence patterns.
- **Configuration choices:**  
  - Default timezone: `Europe/Rome`
  - Event duration: 5 minutes
  - Skips workflows whose name contains `Workflow Scheduling Extraction`
  - Skips schedules matching minute-level patterns
  - Supported parsed human schedule types:
    - daily
    - weekly
    - weekly with day names
    - monthly
    - hourly
  - Unsupported schedules are silently dropped
  - Hourly schedules are represented as one daily recurring event at `00:MM`, not 24 separate hourly events
- **Key expressions or variables used:**  
  - `$input.all()`
  - output object `calendarPayload`:
    - `summary`
    - `description`
    - `start`
    - `end`
    - `timezone`
    - `repeatFrequency`
    - `repeatUntil`
- **Input and output connections:** Input from **Remove Duplicates**; output to **Create an event**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - the parser depends on Italian schedule strings produced upstream; changing upstream wording breaks this node
  - weekly day-specific schedules are reduced to weekly recurrence frequency only; day specificity is not explicitly passed as BYDAY in this implementation
  - monthly parsing ignores explicit day-of-month from string when constructing payload beyond frequency and start date choice
  - unsupported schedules produce no output, which can make debugging appear like silent loss
- **Sub-workflow reference:** None.

#### Create an event
- **Type and technical role:** `n8n-nodes-base.googleCalendar`; creates recurring Google Calendar events.
- **Configuration choices:**  
  - Uses calendar `YOUR_GOOGLE_CALENDAR_ID`
  - Uses OAuth2 credential `Google Calendar OAuth2`
  - Maps event fields from `calendarPayload`
  - `color: 3`
  - `visibility: private`
  - `useDefaultReminders: false`
  - recurrence fields set through:
    - `repeatFrecuency`
    - `repeatUntil`
- **Key expressions or variables used:**  
  - `={{$json.calendarPayload.start}}`
  - `={{$json.calendarPayload.end}}`
  - `={{$json.calendarPayload.summary}}`
  - `={{$json.calendarPayload.description}}`
  - `={{$json.calendarPayload.repeatUntil}}`
  - `={{$json.calendarPayload.repeatFrequency}}`
- **Input and output connections:** Input from **Code: RRULE**; output to **Code: post-create**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - expired/revoked OAuth token
  - invalid calendar ID
  - recurrence parameter compatibility may differ across node versions
  - timezone not explicitly mapped into separate node field, depending on Google Calendar node behavior
  - API rate limiting if large batches run
- **Sub-workflow reference:** None.

#### Code: post-create
- **Type and technical role:** `n8n-nodes-base.code`; aggregates created event IDs by workflow.
- **Configuration choices:**  
  - Reads original RRULE input items with `$('Code: RRULE').all()`
  - Assumes index alignment between created Calendar items and RRULE items
  - Groups by `WorkflowID`
  - Writes `Calendar_EventID` as a space-separated string
  - Sets `On Calendar` to `YES`
  - Sets `calendarLastSync` to current ISO date
- **Key expressions or variables used:**  
  - `$('Code: RRULE').all()`
  - output fields:
    - `WorkflowID`
    - `Workflow`
    - `Tags`
    - `triggerType`
    - `schedule`
    - `freq`
    - `atHour`
    - `atMinute`
    - `byDay`
    - `byMonthDay`
    - `interval`
    - `On Calendar`
    - `Calendar_EventID`
    - `calendarLastSync`
- **Input and output connections:** Input from **Create an event**; output to **Sheets:OnCalendar=YES**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - if a previous node changes item order, cross-reference by index may fail
  - if event creation partially fails, some event IDs may be missing
  - the space-separated event ID format must stay compatible with the delete splitter
- **Sub-workflow reference:** None.

#### Sheets:OnCalendar=YES
- **Type and technical role:** `n8n-nodes-base.googleSheets`; persists canonical synced state back to Google Sheets.
- **Configuration choices:**  
  - Operation: `appendOrUpdate`
  - Matching column: `WorkflowID`
  - Writes all normalized fields plus:
    - `On Calendar = YES`
    - `Calendar_EventID`
    - `calendarLastSync = {{$today}}`
  - Uses same spreadsheet/tab as lookup
  - Authentication: service account
- **Key expressions or variables used:** Many direct mappings from `$json`, including `{{$today}}`.
- **Input and output connections:** Input from **Code: post-create**; no downstream output.
- **Version-specific requirements:** Type version `4.7`.
- **Edge cases or potential failure types:**  
  - if `WorkflowID` is not unique in the sheet, updates may be inconsistent
  - if schema changes, append/update may fail
  - service account must retain edit access
- **Sub-workflow reference:** None.

---

## 2.5 Old Calendar Event Deletion on Update

**Overview:**  
When a workflow schedule changes, this block removes previously created Calendar events using stored event IDs from Sheets. It runs in parallel with the create branch so new events can be generated while stale ones are deleted.

**Nodes Involved:**  
- Code: split eventIds for delete  
- Delete old event

### Node Details

#### Code: split eventIds for delete
- **Type and technical role:** `n8n-nodes-base.code`; expands stored `Calendar_EventID` strings into one item per event ID.
- **Configuration choices:**  
  - Reads `Calendar_EventID`
  - Splits by whitespace using `/\s+/`
  - Emits `{ calendarEventIdToDelete: eid }`
  - If the field is empty, returns no items
- **Key expressions or variables used:** `Calendar_EventID`
- **Input and output connections:** Input from **Switch** update output; output to **Delete old event**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - sticky note says comma-separated, but actual code expects whitespace-separated IDs
  - malformed or concatenated IDs may fail deletion
- **Sub-workflow reference:** None.

#### Delete old event
- **Type and technical role:** `n8n-nodes-base.googleCalendar`; deletes stale Calendar events.
- **Configuration choices:**  
  - Operation: `delete`
  - Event ID: `={{ $json.calendarEventIdToDelete }}`
  - Calendar: `YOUR_GOOGLE_CALENDAR_ID`
  - `continueOnFail: true`
  - `sendUpdates: none`
- **Key expressions or variables used:** `={{ $json.calendarEventIdToDelete }}`
- **Input and output connections:** Input from **Code: split eventIds for delete**; terminal.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - event already deleted (`410 Gone`) is tolerated
  - invalid or stale event IDs
  - OAuth expiration
  - permission mismatch on target calendar
- **Sub-workflow reference:** None.

---

## 2.6 Disconnected Maintenance Sub-flow

**Overview:**  
This is a separate cleanup branch intended for manual Calendar maintenance. It is not connected to the main synchronization process and the webhook node is disabled.

**Nodes Involved:**  
- Webhook  
- Get many events2  
- Delete an event

### Node Details

#### Webhook
- **Type and technical role:** `n8n-nodes-base.webhook`; manual HTTP entry point.
- **Configuration choices:**  
  - Path: `delete-calendar`
  - Disabled
- **Key expressions or variables used:** None in visible configuration.
- **Input and output connections:** No upstream input; output to **Get many events2**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - disabled node means endpoint is inactive
  - if enabled, request payload expectations are undocumented in node config
- **Sub-workflow reference:** None.

#### Get many events2
- **Type and technical role:** `n8n-nodes-base.googleCalendar`; fetches many events from Calendar.
- **Configuration choices:**  
  - Operation: `getAll`
  - Calendar: `YOUR_GOOGLE_CALENDAR_ID`
  - `returnAll: true`
  - `timeMin: 2026-01-16T00:00:00`
  - `timeMax: 2100-12-31T00:00:00`
  - `recurringEventHandling: first`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **Webhook**; output to **Delete an event**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - hardcoded date window may miss events outside range
  - the note claims retrieval by ID or query, but actual config is a broad get-all over a date range
  - large result sets may be slow
- **Sub-workflow reference:** None.

#### Delete an event
- **Type and technical role:** `n8n-nodes-base.googleCalendar`; deletes fetched events.
- **Configuration choices:**  
  - Operation: `delete`
  - Event ID: `={{ $json.id }}`
  - `sendUpdates: none`
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** `={{ $json.id }}`
- **Input and output connections:** Input from **Get many events2**; terminal.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - broad upstream search may delete more than intended if manually enabled without filters
  - invalid/deleted IDs
  - OAuth issues
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Main scheduled entry point |  | Get many workflows | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### A · Fetch<br>Retrieves all n8n workflows via the n8n REST API. Both active and inactive workflows are fetched; filtering to scheduleTrigger-only happens in **Code: parsing**.<br>**Schedule Trigger**<br>Fires every 30 min (cron `*/30 * * * *`).<br>Also supports manual execution from the n8n UI.<br>Entry point for the entire sync pipeline. |
| Get many workflows | n8n-nodes-base.n8n | Fetch active workflows from n8n API | Schedule Trigger | Code: parsing | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### A · Fetch<br>Retrieves all n8n workflows via the n8n REST API. Both active and inactive workflows are fetched; filtering to scheduleTrigger-only happens in **Code: parsing**.<br>**Get many workflows**<br>Calls `GET /api/v1/workflows` on the local n8n instance.<br>Returns all workflows (active + inactive).<br>No filter applied here — filtering is done in Code: parsing. |
| Code: parsing | n8n-nodes-base.code | Parse workflow triggers into normalized schedule records | Get many workflows | Code: save current values | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### B · Parse & Lookup<br>Normalises each workflow's trigger configuration into a human-readable schedule string. Then reads the current Sheets state (schedule, On Calendar, Calendar_EventID) for every WorkflowID so it can be compared downstream.<br>**Code: parsing**<br>- Filters to workflows that have a `scheduleTrigger` node<br>- Excludes system / internal workflows by name<br>- Normalises trigger config → human schedule string<br>- Extracts: WorkflowID, name, tags, triggerType, schedule, freq, atHour, atMinute, byDay, byMonthDay, interval |
| Code: save current values | n8n-nodes-base.code | Preserve live values before sheet merge | Code: parsing | Sheets:Lookup-ExistOnCalendar | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### B · Parse & Lookup<br>Normalises each workflow's trigger configuration into a human-readable schedule string. Then reads the current Sheets state (schedule, On Calendar, Calendar_EventID) for every WorkflowID so it can be compared downstream.<br>**Code: save current values**<br>Snapshots all parsed fields into `current_*` prefixed copies<br>(e.g. `current_schedule`, `current_freq`, …).<br>Preserves the live state before it is overwritten by the Sheets<br>merge so change detection can compare fresh vs. persisted data. |
| Sheets:Lookup-ExistOnCalendar | n8n-nodes-base.googleSheets | Read persisted sync state from Google Sheets | Code: save current values | Code manual merge | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### B · Parse & Lookup<br>Normalises each workflow's trigger configuration into a human-readable schedule string. Then reads the current Sheets state (schedule, On Calendar, Calendar_EventID) for every WorkflowID so it can be compared downstream.<br>**Sheets: Lookup ExistOnCalendar**<br>Reads every row in the "n8n Scheduling" tab of the spreadsheet.<br>Returns per-workflow: `schedule`, `On Calendar`, `Calendar_EventID`, `calendarLastSync`.<br>Uses a Google Service Account (does not expire — no OAuth re-auth needed). |
| Code manual merge | n8n-nodes-base.code | Join live workflow rows with saved sheet rows by WorkflowID | Sheets:Lookup-ExistOnCalendar | Code: detect changes | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### C · Change Detection<br>Merges live workflow data with Sheets state. Compares `current_schedule` (live) vs `schedule` (Sheets) and assigns an **action**: `create` (new) · `update` (changed) · `skip` (unchanged or unsupported type). Cron and minutely schedules are always skipped.<br>**Code manual merge**<br>Joins live workflow items with Sheets rows on WorkflowID.<br>Propagates from Sheets: `schedule`, `On Calendar`, `Calendar_EventID`.<br>`current_*` fields are kept intact from Code: save current values.<br>Critical: `Calendar_EventID` must flow through here for the delete branch to work. |
| Code: detect changes | n8n-nodes-base.code | Decide create/update/skip action | Code manual merge | Switch | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### C · Change Detection<br>Merges live workflow data with Sheets state. Compares `current_schedule` (live) vs `schedule` (Sheets) and assigns an **action**: `create` (new) · `update` (changed) · `skip` (unchanged or unsupported type). Cron and minutely schedules are always skipped.<br>**Code: detect changes**<br>Compares `current_schedule` (live) vs `schedule` (Sheets).<br>Uses only the human string — not granular fields — to avoid false positives from stale rows.<br>- Empty saved schedule or no Calendar entry → **create**<br>- Strings differ → **update** (unless new schedule is unsupported → skip)<br>- Strings equal → **skip**<br>- cron / minutely / unsupported type → always **skip** |
| Switch | n8n-nodes-base.switch | Route items by action | Code: detect changes | Remove Duplicates; Code: split eventIds for delete | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### C · Change Detection<br>Merges live workflow data with Sheets state. Compares `current_schedule` (live) vs `schedule` (Sheets) and assigns an **action**: `create` (new) · `update` (changed) · `skip` (unchanged or unsupported type). Cron and minutely schedules are always skipped.<br>**Switch**<br>Routes items by the `action` field:<br>- `out0` → **create**  (new workflow, no Calendar event yet)<br>- `out1` → **update**  (schedule changed; two parallel branches: D + E)<br>- `out2` → **skip**    (no change; terminal — no downstream nodes) |
| Remove Duplicates | n8n-nodes-base.removeDuplicates | Prevent duplicate event creation | Switch | Code: RRULE | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### D · Create / Update Path<br>Builds a Google Calendar recurring event payload (RRULE) and creates the event. On **update** this branch runs in parallel with the Delete branch (E). After creation, writes Calendar_EventID + On Calendar=YES back to Sheets.<br>**Remove Duplicates**<br>Deduplicates by WorkflowID before event creation.<br>Defensive node: prevents accidental double-creation if a workflow<br>appears multiple times in the current batch (e.g. after a manual re-run). |
| Code: RRULE | n8n-nodes-base.code | Build Google Calendar recurrence payloads | Remove Duplicates | Create an event | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### D · Create / Update Path<br>Builds a Google Calendar recurring event payload (RRULE) and creates the event. On **update** this branch runs in parallel with the Delete branch (E). After creation, writes Calendar_EventID + On Calendar=YES back to Sheets.<br>**Code: RRULE**<br>Converts `current_schedule` string → Google Calendar event payload with RRULE recurrence.<br>Supported types: daily · weekly (with or without specific days) · monthly · hourly.<br>Hourly → 1 daily event at 00:MM (not 24 events — avoids API rate limiting).<br>Unsupported: cron, minutely → item silently dropped (returns empty array). |
| Create an event | n8n-nodes-base.googleCalendar | Create recurring Calendar event | Code: RRULE | Code: post-create | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### D · Create / Update Path<br>Builds a Google Calendar recurring event payload (RRULE) and creates the event. On **update** this branch runs in parallel with the Delete branch (E). After creation, writes Calendar_EventID + On Calendar=YES back to Sheets.<br>**Create an event**<br>Google Calendar node. Creates a recurring event on `YOUR_GOOGLE_ACCOUNT@gmail.com`.<br>Uses `repeatFrequency` + `repeatUntil = null` for infinite recurrence.<br>Returns the full GCal event object including the `id` used downstream.<br>OAuth credential: `Google Calendar OAuth2` (expires periodically). |
| Code: post-create | n8n-nodes-base.code | Aggregate created event IDs and prepare sheet row | Create an event | Sheets:OnCalendar=YES | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### D · Create / Update Path<br>Builds a Google Calendar recurring event payload (RRULE) and creates the event. On **update** this branch runs in parallel with the Delete branch (E). After creation, writes Calendar_EventID + On Calendar=YES back to Sheets.<br>**Code: post-create**<br>Aggregates created event IDs by WorkflowID (handles batch creates).<br>Adds `On Calendar = YES`, `Calendar_EventID`, `calendarLastSync` (ISO timestamp).<br>Prepares the exact row structure expected by Sheets: OnCalendar=YES. |
| Sheets:OnCalendar=YES | n8n-nodes-base.googleSheets | Upsert canonical synced state into Sheets | Code: post-create |  | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### D · Create / Update Path<br>Builds a Google Calendar recurring event payload (RRULE) and creates the event. On **update** this branch runs in parallel with the Delete branch (E). After creation, writes Calendar_EventID + On Calendar=YES back to Sheets.<br>**Sheets: OnCalendar=YES**<br>`appendOrUpdate` operation matching on the **WorkflowID** column.<br>Writes all schedule fields + `On Calendar=YES` + `Calendar_EventID` + `calendarLastSync`.<br>Service Account credential (never expires).<br>This is the canonical state store — downstream runs read from here. |
| Code: split eventIds for delete | n8n-nodes-base.code | Split saved Calendar event IDs into individual delete items | Switch | Delete old event | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### E · Delete Old Event<br>Runs in parallel with branch D on **update**. Deletes the previous Calendar event(s) identified by Calendar_EventID from Sheets. `continueOnFail = true` — HTTP 410 (already gone) does not halt execution.<br>**Code: split eventIds for delete**<br>Splits the `Calendar_EventID` field (comma-separated string) into one item per ID.<br>Required because a workflow can accumulate multiple event IDs across runs.<br>Feeds individual IDs to Delete old event. |
| Delete old event | n8n-nodes-base.googleCalendar | Delete previous Calendar events during update | Code: split eventIds for delete |  | ## 🕒 n8n – Workflow Scheduling Extraction<br>Scans all active n8n workflows every **30 min** · syncs schedules as recurring events on Google Calendar `YOUR_GOOGLE_ACCOUNT@gmail.com` · state store: Google Sheets `n8n Scheduling` tab<br>### E · Delete Old Event<br>Runs in parallel with branch D on **update**. Deletes the previous Calendar event(s) identified by Calendar_EventID from Sheets. `continueOnFail = true` — HTTP 410 (already gone) does not halt execution.<br>**Delete old event**<br>Deletes the previous Calendar event(s) when a schedule update occurs.<br>`continueOnFail = true` — HTTP 410 "Resource has been deleted" is tolerated and does not halt execution.<br>Terminal node — no downstream connections. |
| Webhook | n8n-nodes-base.webhook | Disabled manual cleanup entry point |  | Get many events2 | ### F · Webhook Sub-flow  *(disconnected – manual trigger)*<br>Standalone maintenance endpoint. Accepts an HTTP POST with event IDs, fetches the matching Calendar events, and hard-deletes them. Not connected to the main 30-min pipeline.<br>**Webhook**<br>HTTP POST trigger for manual Calendar cleanup.<br>Not connected to the main 30-min pipeline.<br>Accepts event IDs in the request body. |
| Get many events2 | n8n-nodes-base.googleCalendar | Fetch Calendar events in maintenance branch | Webhook | Delete an event | ### F · Webhook Sub-flow  *(disconnected – manual trigger)*<br>Standalone maintenance endpoint. Accepts an HTTP POST with event IDs, fetches the matching Calendar events, and hard-deletes them. Not connected to the main 30-min pipeline.<br>**Get many events2**<br>Retrieves Calendar events by ID or search query.<br>Feeds the event list into Delete an event. |
| Delete an event | n8n-nodes-base.googleCalendar | Delete Calendar events in maintenance branch | Get many events2 |  | ### F · Webhook Sub-flow  *(disconnected – manual trigger)*<br>Standalone maintenance endpoint. Accepts an HTTP POST with event IDs, fetches the matching Calendar events, and hard-deletes them. Not connected to the main 30-min pipeline.<br>**Delete an event**<br>Hard-deletes Calendar events by ID.<br>Used only in the manual webhook-triggered maintenance flow. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Sync n8n Workflow Schedules to Google Calendar`.
   - Keep it active only after credentials and IDs are fully configured.

2. **Create the main trigger**
   - Add a **Schedule Trigger** node.
   - Configure it to run every 30 minutes.
   - After importing or configuring, verify in the UI that the schedule truly means every 30 minutes, not only at minute 30 of each hour.

3. **Add the n8n API listing node**
   - Add an **n8n** node named `Get many workflows`.
   - Configure it to fetch workflows.
   - Enable filter `activeWorkflows: true`.
   - Create an **n8n API** credential:
     - In n8n settings, generate an API key.
     - Store it in a credential such as `n8n API`.
   - Connect **Schedule Trigger -> Get many workflows**.

4. **Add the parser code node**
   - Add a **Code** node named `Code: parsing`.
   - Paste the parsing logic that:
     - iterates through all input workflows
     - extracts nodes from `wf.activeVersion?.nodes || wf.nodes`
     - includes `scheduleTrigger` and `cron`
     - excludes `webhook`, `errorTrigger`, and `manualTrigger` unless manually enabled
     - emits one item per schedulable trigger
     - outputs fields:
       - `WorkflowID`
       - `Workflow`
       - `Tags`
       - `Active`
       - `triggerType`
       - `freq`
       - `interval`
       - `atHour`
       - `atMinute`
       - `byDay`
       - `byMonthDay`
       - `timezone`
       - `schedule`
       - `Schedule-RAW`
   - Connect **Get many workflows -> Code: parsing**.

5. **Add the live-state snapshot node**
   - Add a **Code** node named `Code: save current values`.
   - Configure it to duplicate parsed fields into `current_*` versions:
     - `current_schedule`
     - `current_freq`
     - `current_atHour`
     - `current_atMinute`
     - `current_byDay`
     - `current_byMonthDay`
     - `current_interval`
     - `current_WorkflowID`
   - Use null-safe handling so zeros are not lost.
   - Connect **Code: parsing -> Code: save current values**.

6. **Prepare the Google Sheet**
   - Create a spreadsheet and note its ID.
   - Create a tab named `n8n Scheduling`.
   - Add columns at minimum:
     - `WorkflowID`
     - `Workflow`
     - `Tags`
     - `triggerType`
     - `schedule`
     - `Calendar_EventID`
     - `On Calendar`
     - `calendarLastSync`
     - `freq`
     - `atHour`
     - `atMinute`
     - `byDay`
     - `byMonthDay`
     - `interval`

7. **Create Google Service Account credential for Sheets**
   - In Google Cloud Platform, create a service account.
   - Enable Google Sheets API.
   - Download the JSON key.
   - In n8n, create a **Google Service Account** credential.
   - Share the spreadsheet with the service account email as **Editor**.

8. **Add the Google Sheets lookup node**
   - Add a **Google Sheets** node named `Sheets:Lookup-ExistOnCalendar`.
   - Set authentication to **Service Account**.
   - Select your spreadsheet and the `n8n Scheduling` tab.
   - Leave it as a sheet read node returning rows.
   - Enable `alwaysOutputData` if available.
   - Connect **Code: save current values -> Sheets:Lookup-ExistOnCalendar**.

9. **Add the manual merge code node**
   - Add a **Code** node named `Code manual merge`.
   - Configure it to:
     - read all current items from `$('Code: save current values').all()`
     - build a map of sheet rows by `WorkflowID`
     - merge saved values back into each live item
     - preserve:
       - `current_*`
     - inject saved values into:
       - `schedule`
       - `freq`
       - `atHour`
       - `atMinute`
       - `byDay`
       - `byMonthDay`
       - `interval`
       - `On Calendar`
       - `Calendar_EventID`
   - Connect **Sheets:Lookup-ExistOnCalendar -> Code manual merge**.

10. **Add the change-detection code node**
    - Add a **Code** node named `Code: detect changes`.
    - Implement logic:
      - compare `current_schedule` with saved `schedule`
      - if no saved state or no `On Calendar`, set `action = create` for supported schedules
      - if saved state exists and schedule changed, set `action = update`
      - if unchanged, set `action = skip`
      - if schedule is unsupported (`cron`, minutely), set `action = skip`
    - Output at least:
      - `action`
      - `scheduleChanged`
      - `changedFields`
    - Connect **Code manual merge -> Code: detect changes**.

11. **Add the Switch node**
    - Add a **Switch** node named `Switch`.
    - Switch on `{{$json.action}}`.
    - Create three outputs:
      - `create`
      - `update`
      - `skip`
    - Route:
      - `create` to creation path
      - `update` to both creation path and deletion path
      - `skip` nowhere
    - Connect **Code: detect changes -> Switch**.

12. **Add deduplication**
    - Add a **Remove Duplicates** node named `Remove Duplicates`.
    - Compare selected fields:
      - `WorkflowID`
      - `current_schedule`
    - Connect:
      - **Switch create -> Remove Duplicates**
      - **Switch update -> Remove Duplicates**

13. **Add the recurrence payload builder**
    - Add a **Code** node named `Code: RRULE`.
    - Configure constants such as:
      - default timezone `Europe/Rome`
      - event duration `5`
    - Implement support for schedule strings:
      - daily
      - weekly
      - weekly with named days
      - monthly
      - hourly
    - Skip:
      - cron
      - minutely
      - unsupported formats
    - Build a `calendarPayload` object with:
      - `summary`
      - `description`
      - `start`
      - `end`
      - `timezone`
      - `repeatFrequency`
      - `repeatUntil`
    - Also add a workflow-name exclusion so the sync workflow itself is not recreated in Calendar.
    - Connect **Remove Duplicates -> Code: RRULE**.

14. **Create Google Calendar OAuth2 credential**
    - In Google Cloud, enable Google Calendar API.
    - Create OAuth client credentials.
    - In n8n, create a **Google Calendar OAuth2** credential.
    - Authorize access to the target calendar account.
    - Note that this credential may need periodic reconnection.

15. **Add the Calendar create node**
    - Add a **Google Calendar** node named `Create an event`.
    - Operation: create event.
    - Choose the target calendar ID.
    - Set:
      - `start = {{$json.calendarPayload.start}}`
      - `end = {{$json.calendarPayload.end}}`
      - `summary = {{$json.calendarPayload.summary}}`
      - `description = {{$json.calendarPayload.description}}`
      - `repeatFrequency = {{$json.calendarPayload.repeatFrequency}}`
      - `repeatUntil = {{$json.calendarPayload.repeatUntil}}`
      - `color = 3`
      - `visibility = private`
      - `useDefaultReminders = false`
    - Connect **Code: RRULE -> Create an event**.

16. **Add the post-create aggregation node**
    - Add a **Code** node named `Code: post-create`.
    - Configure it to:
      - read created Google Calendar results
      - cross-reference the original `Code: RRULE` items using `$('Code: RRULE').all()`
      - group events by `WorkflowID`
      - produce one row per workflow
      - join all created event IDs into `Calendar_EventID`
      - set `On Calendar = YES`
      - set `calendarLastSync` to today’s date
    - Connect **Create an event -> Code: post-create**.

17. **Add the Google Sheets upsert node**
    - Add a **Google Sheets** node named `Sheets:OnCalendar=YES`.
    - Authentication: Service Account.
    - Operation: `appendOrUpdate`.
    - Spreadsheet: your sheet.
    - Tab: `n8n Scheduling`.
    - Match on `WorkflowID`.
    - Map fields:
      - `WorkflowID`
      - `Workflow`
      - `Tags`
      - `triggerType`
      - `schedule`
      - `Calendar_EventID`
      - `On Calendar` = `YES`
      - `calendarLastSync` = current date
      - `freq`
      - `atHour`
      - `atMinute`
      - `byDay`
      - `byMonthDay`
      - `interval`
    - Connect **Code: post-create -> Sheets:OnCalendar=YES**.

18. **Add the delete-splitter node**
    - Add a **Code** node named `Code: split eventIds for delete`.
    - Configure it to:
      - read `Calendar_EventID`
      - split by whitespace
      - emit one item per event ID under `calendarEventIdToDelete`
    - Connect **Switch update -> Code: split eventIds for delete**.

19. **Add the Calendar delete node**
    - Add a **Google Calendar** node named `Delete old event`.
    - Operation: delete event.
    - Calendar: same target calendar.
    - Event ID: `{{$json.calendarEventIdToDelete}}`
    - Set `sendUpdates = none`.
    - Enable `continueOnFail = true`.
    - Connect **Code: split eventIds for delete -> Delete old event**.

20. **Optional: add the maintenance sub-flow**
    - Add a **Webhook** node named `Webhook`.
      - Path: `delete-calendar`
      - Leave it disabled if you only want the main sync.
    - Add a **Google Calendar** node named `Get many events2`.
      - Operation: `getAll`
      - Calendar: target calendar
      - `returnAll = true`
      - define a time window
      - recurring handling: first
    - Add a **Google Calendar** node named `Delete an event`.
      - Operation: delete
      - Event ID: `{{$json.id}}`
      - `sendUpdates = none`
      - enable continue behavior on errors
    - Connect:
      - **Webhook -> Get many events2 -> Delete an event**
    - This branch is independent and should remain disconnected from the main flow.

21. **Add documentation sticky notes if desired**
    - You may replicate the large overview note and section notes for maintainability.
    - Include setup notes, credential dependencies, and limits.

22. **Configure required IDs and credentials**
    - Replace:
      - `YOUR_GOOGLE_CALENDAR_ID`
      - `YOUR_SPREADSHEET_ID`
      - any credential placeholder names
    - Confirm:
      - n8n API credential works
      - Google Service Account has spreadsheet access
      - Google Calendar OAuth2 has calendar access

23. **Test with manual execution**
    - Run the workflow manually.
    - Confirm:
      - rows appear in Google Sheets
      - supported schedules create recurring Google Calendar events
      - unsupported schedules are skipped
      - re-running without changes produces `skip`
      - changing a workflow schedule causes `update` and old event deletion

24. **Activate the workflow**
    - Once validated, activate the workflow.
    - Monitor OAuth expiration for Google Calendar.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Full documentation for setup and operation | https://paoloronco.notion.site/n8n-Workflow-Scheduling-Extraction-Setup-Docs-330f0ba27c3280ef99b2c5e8e7dfd497 |
| Main purpose: scans active n8n workflows every 30 minutes, syncs schedules to Google Calendar, stores state in Google Sheets tab `n8n Scheduling` | Workflow overview note |
| Required credentials mentioned in notes: n8n API key, Google Calendar OAuth2, Google Service Account | Workflow overview note |
| Setup note includes optional Workflow Settings error workflow ID | Workflow overview note |
| Known limits: hourly schedules are represented as one daily event at `00:MM`, not 24 hourly events | Workflow overview note |
| Known limits: cron and minutely schedules are skipped and never appear on Calendar | Workflow overview note |
| Known limits: disconnected webhook sub-flow is maintenance only | Workflow overview note |
| Operational note: Google Calendar OAuth2 may expire periodically and block create/delete branches | Workflow overview note |

## Important consistency notes

- The sticky notes and actual node configurations are not perfectly aligned in several places:
  - **Get many workflows** is documented as fetching active and inactive workflows, but the node actually filters to `activeWorkflows: true`.
  - **Code: parsing** sticky text says it filters to `scheduleTrigger`, but the code also includes `cron`.
  - **Code: split eventIds for delete** sticky text says comma-separated IDs, but the code splits by whitespace.
  - **Schedule Trigger** description says every 30 minutes, while raw config should be rechecked in the n8n UI to confirm exact timing semantics.

These discrepancies should be corrected if you intend to use this workflow as a long-term production reference.