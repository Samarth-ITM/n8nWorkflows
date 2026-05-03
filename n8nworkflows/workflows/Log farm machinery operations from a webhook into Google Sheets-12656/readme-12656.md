Log farm machinery operations from a webhook into Google Sheets

https://n8nworkflows.xyz/workflows/log-farm-machinery-operations-from-a-webhook-into-google-sheets-12656


# Log farm machinery operations from a webhook into Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Log farm machinery operations from a webhook into Google Sheets

**Purpose:**  
This workflow receives farm machinery operation logs via an HTTP **POST webhook** (typically from an HTML form or data collection app), then writes the data into **three Google Sheets tabs**:
- `main_logs` (one row per operation)
- `fuel` (one row per fuel entry)
- `breakdowns` (one row per breakdown/downtime event)

**Target use cases:**
- Centralized daily/shift operation logging (operator, machinery, field, times, outputs)
- Separately tracking multiple fuel fills within one operation
- Separately tracking multiple breakdown/downtime events within one operation

### Logical Blocks
**1.1 Input Reception (Webhook)**
- Receives JSON payload from external app/form.

**1.2 Configuration & Branching**
- Stores spreadsheet ID and fans out to 3 branches (main logs, fuel, breakdowns).

**1.3 Main Operation Logging**
- Appends one row per operation into `main_logs`.

**1.4 Fuel Entries Expansion & Logging**
- Splits `fuelEntries[]` into multiple items, appends to `fuel`.

**1.5 Breakdown Expansion & Logging**
- Splits `breakdowns[]` into multiple items, appends to `breakdowns`.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (Webhook)

**Overview:**  
Accepts a POST request containing farm operation details in JSON. The webhook output is passed downstream for logging.

**Nodes Involved:**
- Webhook Trigger

#### Node: Webhook Trigger
- **Type / Role:** `n8n-nodes-base.webhook` — entry point; receives HTTP requests.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `farm-operations-logger` (final URL depends on n8n instance)
  - **Response Mode:** `lastNode` (the response returned to caller will come from the last executed node in the run; in a branched flow this can be ambiguous—see edge cases).
  - Expects `Content-Type: application/json` (implied by sticky note; not enforced unless you add checks).
- **Key data used:** Downstream nodes reference payload as `$json.body...`, meaning the webhook is expected to expose the parsed JSON under `body`.
- **Connections:**
  - **Output →** Workflow Configuration
- **Version requirements:** node typeVersion `2.1`
- **Edge cases / failures:**
  - Invalid JSON / wrong content-type may cause missing `$json.body` fields downstream.
  - With **responseMode=lastNode** and **three parallel branches**, the “last node” is timing-dependent; the caller may get inconsistent responses or timeouts if a branch is slow.
  - No validation: missing required fields (e.g., `body.id`) will lead to blank cells or expression errors in later nodes.

---

### 2.2 Configuration & Branching

**Overview:**  
A central Set node stores the Google Sheets spreadsheet ID (intended as a single place to update), and splits execution into three parallel branches.

**Nodes Involved:**
- Workflow Configuration

#### Node: Workflow Configuration
- **Type / Role:** `n8n-nodes-base.set` — adds/overrides fields; used here as a configuration anchor and a branching hub.
- **Configuration (interpreted):**
  - Adds field: `spreadsheetId = "YOUR_SPREADSHEET_ID_HERE"`
  - `includeOtherFields: true` (keeps the incoming webhook payload alongside the new config field)
- **Key expressions / variables:**
  - Downstream nodes sometimes reference this node explicitly via:  
    `$('Workflow Configuration').item.json.body...`
- **Connections:**
  - **Input ←** Webhook Trigger
  - **Outputs →** Main Log Capture Node, Expand Fuel Entries, Expand Breakdowns (three parallel outputs)
- **Version requirements:** typeVersion `3.4`
- **Edge cases / failures:**
  - Spreadsheet ID is *not actually used* by the Google Sheets nodes in this workflow (they have `documentId` hardcoded to `YOUR_SPREADSHEET_ID_HERE`). If you only change it here and forget to change the Google Sheets nodes, writes will fail.
  - If webhook payload is large, parallel writes increase chance of partial success (some sheets updated, others fail).

---

### 2.3 Main Operation Logging (main_logs)

**Overview:**  
Appends a single row per operation into the `main_logs` sheet, mapping key fields from the webhook payload.

**Nodes Involved:**
- Main Log Capture Node

#### Node: Main Log Capture Node
- **Type / Role:** `n8n-nodes-base.googleSheets` — append a row to Google Sheets.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Spreadsheet (Document):** set by **Document ID** (currently placeholder `YOUR_SPREADSHEET_ID_HERE`)
  - **Sheet/tab name:** `main_logs`
  - **Mapping mode:** “define below” with explicit schema; matches on column `id` (but since operation is append, matchingColumns is effectively irrelevant unless switching operation later).
- **Key expressions / variables used (selected):**
  - `id`: `={{ $json.body.id }}`
  - `date`: `={{ $json.body.date }}`
  - `timestamp`: `={{ $json.body.timestamp }}`
  - `operator`: `={{ $json.body.operator }}`
  - `supervisor`: `={{ $json.body.supervisor }}`
  - `start time`: `={{ $json.body.startTime }}`
  - `end time`: `={{ $json.body.endTime }}`
  - `total hours`: `={{ $json.body.totalHours }}`
  - `machinery`: `={{ $json.body.machinery }}`
  - `field`: `={{ $json.body.field }}`
  - `operation`: `={{ $json.body.operation }}`
  - `value/units`: `={{ $json.body.operationOutput.value }} {{ $json.body.operationOutput.unit }}`
  - `fuel entries`: `={{ $json.body.fuelEntries }}`
  - `breakdowns`: `={{ $json.body.breakdowns }}`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output:** none (end of branch)
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Version requirements:** typeVersion `4.7`
- **Edge cases / failures:**
  - If `body.operationOutput` is missing, `operationOutput.value/unit` expressions may evaluate to `undefined` strings or fail depending on n8n expression behavior.
  - Writing arrays/objects directly (`fuelEntries`, `breakdowns`) typically results in `[object Object]` or JSON-ish rendering; if you expect readable content, you may want `JSON.stringify(...)`.
  - Google Sheets errors: invalid spreadsheet ID, missing tab `main_logs`, insufficient permissions, quota limits.

---

### 2.4 Fuel Entries Expansion & Logging (fuel)

**Overview:**  
Transforms an array of fuel entries (`body.fuelEntries[]`) into separate items so each entry becomes one row in the `fuel` sheet.

**Nodes Involved:**
- Expand Fuel Entries
- Fuel Capture Log

#### Node: Expand Fuel Entries
- **Type / Role:** `n8n-nodes-base.code` — converts one webhook item into multiple items (one per fuel entry).
- **Configuration (interpreted):**
  - Iterates over all incoming items.
  - Extracts:
    - `logId = item.json.body.id`
    - `fuelEntries = item.json.body.fuelEntries || []`
  - Emits per fuel entry:
    - `id`
    - `machineryHours: fuelEntry.machineryHours`
    - `litres: fuelEntry.litres`
    - `attendant: fuelEntry.attendant`
- **Key variables:**
  - Uses `$input.all()`
  - Outputs `items.push({ json: {...}})`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Fuel Capture Log
- **Version requirements:** typeVersion `2`
- **Edge cases / failures:**
  - If `body` or `body.id` is missing, `logId` becomes undefined.
  - If `fuelEntries` is not an array, loop behavior may fail.
  - Output schema doesn’t include supplier or start/end hours, but the next node expects those (see next node mismatch).

#### Node: Fuel Capture Log
- **Type / Role:** `n8n-nodes-base.googleSheets` — appends rows to `fuel` sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Sheet/tab name:** `fuel`
  - **Spreadsheet (Document):** placeholder `YOUR_SPREADSHEET_ID_HERE`
  - Explicit schema with columns: `id, mach-start, mach-end, litre, attendant, supplier`
- **Key expressions / variables used:**
  - `id`: `={{ $json.id }}`
  - **But other fields are pulled from the Workflow Configuration node, index [0]:**
    - `litre`: `={{ $('Workflow Configuration').item.json.body.fuelEntries[0].litres }}`
    - `mach-end`: `={{ $('Workflow Configuration').item.json.body.fuelEntries[0].machineryHoursEnd }}`
    - `supplier`: `={{ $('Workflow Configuration').item.json.body.fuelEntries[0].supplier }}`
    - `attendant`: `={{ $('Workflow Configuration').item.json.body.fuelEntries[0].attendant }}`
    - `mach-start`: `={{ $('Workflow Configuration').item.json.body.fuelEntries[0].machineryHoursStart }}`
- **Connections:**
  - **Input ←** Expand Fuel Entries
  - **Output:** none (end of branch)
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Version requirements:** typeVersion `4.7`
- **Edge cases / failures (important):**
  - **Data mismatch bug:** This node ignores the expanded per-entry fields and always writes **fuelEntries[0]** from the original payload. If multiple fuel entries exist, every appended row will duplicate the first entry’s details (only `id` changes per expanded item).
  - The Code node outputs `litres` and `machineryHours`, but this Sheets node expects `mach-start/mach-end/supplier`. As written, these values will be undefined unless present in `fuelEntries[0]`.
  - If `fuelEntries` is empty, referencing `[0]` yields undefined → blank cells or expression issues.
  - Missing `fuel` tab, permissions, or invalid document ID will fail the append.

---

### 2.5 Breakdown Expansion & Logging (breakdowns)

**Overview:**  
Transforms `body.breakdowns[]` into separate items and logs each breakdown event as its own row in the `breakdowns` sheet.

**Nodes Involved:**
- Expand Breakdowns
- Capture Breakdowns

#### Node: Expand Breakdowns
- **Type / Role:** `n8n-nodes-base.code` — converts one operation into multiple breakdown rows.
- **Configuration (interpreted):**
  - Extracts:
    - `logId = item.json.body.id`
    - `breakdowns = item.json.body.breakdowns || []`
  - For each breakdown, outputs:
    - `id`
    - `breakdown` (full object)
    - `type`, `startTime`, `endTime`, `totalDowntime`
- **Connections:**
  - **Input ←** Workflow Configuration
  - **Output →** Capture Breakdowns
- **Version requirements:** typeVersion `2`
- **Edge cases / failures:**
  - Missing `body.id` leads to undefined `id`.
  - If breakdown objects don’t have expected keys, mapped values become blank.

#### Node: Capture Breakdowns
- **Type / Role:** `n8n-nodes-base.googleSheets` — appends rows to `breakdowns` sheet.
- **Configuration (interpreted):**
  - **Operation:** Append
  - **Sheet/tab name:** `breakdowns`
  - **Spreadsheet (Document):** placeholder `YOUR_SPREADSHEET_ID_HERE`
  - Columns mapped from expanded item:
    - `id`: `={{ $json.id }}`
    - `type`: `={{ $json.type }}`
    - `start time`: `={{ $json.startTime }}`
    - `end time`: `={{ $json.endTime }}`
    - `total downtime`: `={{ $json.totalDowntime }}`
    - `notes`: `={{ $json.breakdown.notes }}`
- **Connections:**
  - **Input ←** Expand Breakdowns
  - **Output:** none (end of branch)
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account`)
- **Version requirements:** typeVersion `4.7`
- **Edge cases / failures:**
  - If `breakdown.notes` is missing, notes column will be blank.
  - Missing tab `breakdowns`, invalid spreadsheet ID, permission/quota errors.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Trigger | Webhook | Receives POSTed JSON payload from external form/app | — | Workflow Configuration | ## WEBHOOK TRIGGER; Receives POST requests from data collection app; Path: farm-operations-logger; Accepts JSON payload with operation details; → Passes to Workflow Configuration |
| Workflow Configuration | Set | Stores spreadsheetId config + branches into 3 logging paths | Webhook Trigger | Main Log Capture Node; Expand Fuel Entries; Expand Breakdowns | ## WORKFLOW CONFIGURATION; Central node that: - Stores spreadsheet ID for reference - Splits data flow to 3 branches: 1. Main log (direct) 2. Fuel entries (via expansion) 3. Breakdowns (via expansion) → Update spreadsheetId value here! |
| Expand Fuel Entries | Code | Explodes `fuelEntries[]` into individual items | Workflow Configuration | Fuel Capture Log | ## EXPAND FUEL ENTRIES; Code node that loops through fuelEntries array; Creates one item per fuel entry with log ID; Handles multiple fuel fills per operation; → Each fuel entry becomes separate row |
| Fuel Capture Log | Google Sheets | Appends fuel rows into `fuel` tab | Expand Fuel Entries | — | ## FUEL CAPTURE LOG; Writes to "fuel" sheet tab; Columns: id, mach-start, mach-end, litre, attendant, supplier; ⚠️ Update Document to your spreadsheet; ⚠️ Ensure "fuel" tab exists |
| Expand Breakdowns | Code | Explodes `breakdowns[]` into individual items | Workflow Configuration | Capture Breakdowns | ## EXPAND BREAKDOWNS; Code node that loops through breakdowns array; Creates one item per breakdown with log ID; Includes: type, start/end times, downtime, notes; → Each breakdown becomes separate row |
| Capture Breakdowns | Google Sheets | Appends breakdown rows into `breakdowns` tab | Expand Breakdowns | — | ## CAPTURE BREAKDOWNS; Writes to "breakdowns" sheet tab; Columns: id, type, start time, end time, total downtime, notes; ⚠️ Update Document to your spreadsheet; ⚠️ Ensure "breakdowns" tab exists |
| Main Log Capture Node | Google Sheets | Appends main operation row into `main_logs` tab | Workflow Configuration | — | ## MAIN LOG CAPTURE NODE; Writes to "main_logs" sheet tab; Captures primary operation data: - Operator, machinery, field, operation - Start/end times, total hours - Operation output (value/units) ⚠️ Update Document to your spreadsheet; ⚠️ Ensure "main_logs" tab exists |
| Sticky Note | Sticky Note | Workspace comment (project overview & checklist) | — | — | ## OPERATOR-SPECIFIC FARM OPERATIONS LOGGER; Captures detailed farm machinery operations data via webhook and logs to three Google Sheets tabs: main operations, fuel consumption, and equipment breakdowns. SETUP CHECKLIST: □ Create Google Sheet with 3 tabs: main_logs, fuel, breakdowns □ Add column headers to each tab (see documentation) □ Import workflow into n8n □ Set up Google Sheets OAuth2 credentials □ Update spreadsheet ID in Workflow Configuration node □ Update Document field in all 3 Google Sheets nodes □ Test webhook with sample payload □ Activate workflow □ Update data collection app with Production URL WEBHOOK PATH: GET THIS FROM WEBHOOK TRIGGER METHOD: POST CONTENT-TYPE: application/json |
| Sticky Note1 | Sticky Note | Workspace comment for webhook block | — | — | ## WEBHOOK TRIGGER; Receives POST requests from data collection app; Path: farm-operations-logger; Accepts JSON payload with operation details; → Passes to Workflow Configuration |
| Sticky Note2 | Sticky Note | Workspace comment for configuration block | — | — | ## WORKFLOW CONFIGURATION; Central node that: - Stores spreadsheet ID for reference - Splits data flow to 3 branches: 1. Main log (direct) 2. Fuel entries (via expansion) 3. Breakdowns (via expansion) → Update spreadsheetId value here! |
| Sticky Note3 | Sticky Note | Workspace comment for fuel expansion | — | — | ## EXPAND FUEL ENTRIES; Code node that loops through fuelEntries array; Creates one item per fuel entry with log ID; Handles multiple fuel fills per operation; → Each fuel entry becomes separate row |
| Sticky Note4 | Sticky Note | Workspace comment for fuel sheet logging | — | — | ## FUEL CAPTURE LOG; Writes to "fuel" sheet tab; Columns: id, mach-start, mach-end, litre, attendant, supplier; ⚠️ Update Document to your spreadsheet; ⚠️ Ensure "fuel" tab exists |
| Sticky Note5 | Sticky Note | Workspace comment for main log sheet | — | — | ## MAIN LOG CAPTURE NODE; Writes to "main_logs" sheet tab; Captures primary operation data: - Operator, machinery, field, operation - Start/end times, total hours - Operation output (value/units) ⚠️ Update Document to your spreadsheet; ⚠️ Ensure "main_logs" tab exists |
| Sticky Note6 | Sticky Note | Workspace comment for breakdown expansion | — | — | ## EXPAND BREAKDOWNS; Code node that loops through breakdowns array; Creates one item per breakdown with log ID; Includes: type, start/end times, downtime, notes; → Each breakdown becomes separate row |
| Sticky Note7 | Sticky Note | Workspace comment for breakdown sheet logging | — | — | ## CAPTURE BREAKDOWNS; Writes to "breakdowns" sheet tab; Columns: id, type, start time, end time, total downtime, notes; ⚠️ Update Document to your spreadsheet; ⚠️ Ensure "breakdowns" tab exists |

---

## 4. Reproducing the Workflow from Scratch

1) **Create the Google Sheet**
   1. Create a spreadsheet in Google Drive.
   2. Add **three tabs** named exactly:
      - `main_logs`
      - `fuel`
      - `breakdowns`
   3. Add headers matching the columns used by the Google Sheets nodes:
      - `main_logs`: `id, timestamp, date, operator, supervisor, start time, end time, total hours, machinery, field, operation, value/units, fuel entries, breakdowns`
      - `fuel`: `id, mach-start, mach-end, litre, attendant, supplier`
      - `breakdowns`: `id, type, start time, end time, total downtime, notes`

2) **Create credentials in n8n**
   1. Go to **Credentials → New**.
   2. Create **Google Sheets OAuth2** credentials.
   3. Authenticate with a Google account that has edit access to the spreadsheet.

3) **Add the Webhook entry node**
   1. Add node: **Webhook**.
   2. Set:
      - **HTTP Method:** POST
      - **Path:** `farm-operations-logger`
      - **Response Mode:** Last Node (match workflow; optional—see note below)
   3. Save the workflow so the webhook URLs are generated.

4) **Add “Workflow Configuration” (Set node)**
   1. Add node: **Set**.
   2. Enable **Include Other Fields**.
   3. Add field:
      - `spreadsheetId` (String) = your spreadsheet ID (the long ID from the Google Sheets URL)

5) **Connect Webhook → Workflow Configuration**

6) **Main logs branch**
   1. Add node: **Google Sheets** (Append).
   2. Set credentials: your **Google Sheets OAuth2**.
   3. Set **Document/Spreadsheet**: select by ID (paste spreadsheet ID).
   4. Set **Sheet Name**: `main_logs`
   5. Configure columns mapping (use expressions) to match:
      - `id` → `$json.body.id`
      - `timestamp` → `$json.body.timestamp`
      - `date` → `$json.body.date`
      - `operator` → `$json.body.operator`
      - `supervisor` → `$json.body.supervisor`
      - `start time` → `$json.body.startTime`
      - `end time` → `$json.body.endTime`
      - `total hours` → `$json.body.totalHours`
      - `machinery` → `$json.body.machinery`
      - `field` → `$json.body.field`
      - `operation` → `$json.body.operation`
      - `value/units` → `$json.body.operationOutput.value + " " + $json.body.operationOutput.unit`
      - `fuel entries` → `$json.body.fuelEntries`
      - `breakdowns` → `$json.body.breakdowns`
   6. Connect **Workflow Configuration → Main Log Capture Node**.

7) **Fuel branch (expand then append)**
   1. Add node: **Code** named “Expand Fuel Entries”.
   2. Paste logic that iterates `body.fuelEntries[]` and outputs one item per entry (as in your workflow).
   3. Add node: **Google Sheets** named “Fuel Capture Log” (Append).
   4. Set **Document**: your spreadsheet ID; **Sheet Name**: `fuel`.
   5. Map columns.
      - To correctly use the expanded items, map from `$json` (output of Code). For example:
        - `id` → `$json.id`
        - `litre` → `$json.litres`
        - `attendant` → `$json.attendant`
        - plus any other fields you output (you may need to adjust Code to emit `machineryHoursStart`, `machineryHoursEnd`, `supplier` to match your sheet).
   6. Connect **Workflow Configuration → Expand Fuel Entries → Fuel Capture Log**.

8) **Breakdowns branch (expand then append)**
   1. Add node: **Code** named “Expand Breakdowns”.
   2. Paste logic to iterate `body.breakdowns[]` and output one item per breakdown.
   3. Add node: **Google Sheets** named “Capture Breakdowns” (Append).
   4. Set **Document**: your spreadsheet ID; **Sheet Name**: `breakdowns`.
   5. Map:
      - `id` → `$json.id`
      - `type` → `$json.type`
      - `start time` → `$json.startTime`
      - `end time` → `$json.endTime`
      - `total downtime` → `$json.totalDowntime`
      - `notes` → `$json.breakdown.notes`
   6. Connect **Workflow Configuration → Expand Breakdowns → Capture Breakdowns**.

9) **Test**
   1. Use the webhook **Test URL** with a sample JSON payload matching the expected structure (`body.id`, `body.fuelEntries[]`, `body.breakdowns[]`, etc.).
   2. Verify that:
      - One row is added to `main_logs`
      - N rows added to `fuel` (one per fuel entry)
      - M rows added to `breakdowns` (one per breakdown)

10) **Activate & use Production URL**
   1. Activate the workflow.
   2. Copy the webhook **Production URL** into the HTML form / app.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create Google Sheet with 3 tabs: `main_logs`, `fuel`, `breakdowns` and add column headers | From sticky note “OPERATOR-SPECIFIC FARM OPERATIONS LOGGER” |
| Update spreadsheet ID in Workflow Configuration node **and** update Document field in all Google Sheets nodes | From sticky notes + workflow behavior (Document IDs are hardcoded placeholders) |
| Webhook path: `farm-operations-logger`, method POST, content-type `application/json` | From sticky notes |
| Branching design logs main operation + expands arrays into separate rows | Workflow architecture note |
| Known issue: Fuel Capture Log references `fuelEntries[0]` from Workflow Configuration rather than using the expanded item | Derived from node expressions; impacts correctness for multiple fuel entries |