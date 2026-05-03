Bulk create Google Calendar events from Google Sheets

https://n8nworkflows.xyz/workflows/bulk-create-google-calendar-events-from-google-sheets-14107


# Bulk create Google Calendar events from Google Sheets

# 1. Workflow Overview

This workflow bulk-creates Google Calendar events from rows stored in a Google Sheet. It is designed for cases where a spreadsheet acts as a simple event queue: each row contains event fields such as summary, start time, end time, location, description, attendees, and a processing status.

The workflow reads all rows from a sheet, checks whether each row should still be processed, creates calendar events only for rows marked `pending` or `failed`, then updates the sheet to reflect whether the event creation succeeded or failed. Rows already marked as created are skipped.

## 1.1 Input Reception and Initialization

The workflow starts manually. There is no webhook or scheduled trigger in this version, so it is intended for operator-driven batch execution.

## 1.2 Spreadsheet Read and Status Filtering

The workflow reads all rows from a configured Google Sheet and evaluates each row’s `status` field. Only rows with `status = pending` or `status = failed` continue to event creation. Other rows are ignored.

## 1.3 Calendar Event Creation

For each eligible row, the workflow attempts to create a Google Calendar event using row values for start, end, summary, description, location, and attendees.

## 1.4 Error Detection and Spreadsheet Status Update

After the calendar call, the workflow checks whether the Google Calendar node returned an `error` field. If no error is present, the corresponding sheet row is updated to `created`. If an error exists, the row is updated to `failed`.

---

# 2. Block-by-Block Analysis

## Block 1 — Initialization

### Overview
This block provides the workflow entry point. It exists solely to begin execution manually from the n8n editor or execution interface.

### Nodes Involved
- Manual Trigger

### Node Details

#### Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for ad hoc execution.
- **Configuration choices:** No parameters are configured.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Read Sheet`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No runtime failure in normal use.
  - Workflow will not run automatically because there is no scheduled or event-based trigger.
- **Sub-workflow reference:** None.

---

## Block 2 — Read Google Sheets and Decide Whether to Process

### Overview
This block retrieves spreadsheet rows and filters them by processing status. It ensures that only rows marked as `pending` or `failed` move forward, preventing already-created rows from being recreated.

### Nodes Involved
- Read Sheet
- Check Status Spreadsheets
- No Operation, do nothing

### Node Details

#### Read Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads rows from a Google Sheet document.
- **Configuration choices:**
  - Uses a specific spreadsheet document: `[EXAMPLE] Bulk Create Event Google Calendar`
  - Uses sheet/tab `Sheet1` via `gid=0`
  - Default read operation is used to fetch rows
- **Key expressions or variables used:** No custom expressions in parameters.
- **Input and output connections:**
  - Input: `Manual Trigger`
  - Output: `Check Status Spreadsheets`
- **Version-specific requirements:** Type version `4`; requires Google Sheets OAuth2 credentials compatible with the current n8n Google Sheets node.
- **Edge cases or potential failure types:**
  - OAuth credential failure or expired Google token
  - Missing access to the spreadsheet
  - Wrong sheet selected or deleted tab
  - Unexpected column structure
  - Missing `status`, `start`, `end`, or other expected columns in returned rows
- **Sub-workflow reference:** None.

#### Check Status Spreadsheets
- **Type and technical role:** `n8n-nodes-base.if`; branches rows depending on whether they are still eligible for processing.
- **Configuration choices:**
  - Uses OR logic
  - True if `{{$json.status}}` equals `pending`
  - True if `{{$json.status}}` equals `failed`
  - Any other value goes to the false branch
- **Key expressions or variables used:**
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: `Read Sheet`
  - True output: `Create Event`
  - False output: `No Operation, do nothing`
- **Version-specific requirements:** Type version `2.2`; condition engine uses version 2 syntax with strict validation.
- **Edge cases or potential failure types:**
  - If `status` is missing, null, differently cased, or contains spaces, the row will be treated as false and skipped
  - Values like `Pending`, `PENDING`, or ` created` will not match because the comparison is case-sensitive and strict
- **Sub-workflow reference:** None.

#### No Operation, do nothing
- **Type and technical role:** `n8n-nodes-base.noOp`; terminal placeholder for skipped rows.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: false branch from `Check Status Spreadsheets`
  - Output: none
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - No functional failure; it simply absorbs skipped items
- **Sub-workflow reference:** None.

---

## Block 3 — Create Google Calendar Event

### Overview
This block builds and submits a Google Calendar event from spreadsheet row data. It uses the row fields directly and allows the workflow to continue even if event creation fails.

### Nodes Involved
- Create Event

### Node Details

#### Create Event
- **Type and technical role:** `n8n-nodes-base.googleCalendar`; creates a calendar event in a selected Google Calendar.
- **Configuration choices:**
  - Calendar: `user@example.com`
  - Start time comes from the sheet row
  - End time comes from the sheet row
  - Additional fields:
    - `summary` from sheet row
    - `location` from sheet row
    - `description` from sheet row
    - `attendees` from current item JSON
  - `continueOnFail` is enabled
  - `executeOnce` is enabled
- **Key expressions or variables used:**
  - `={{ $('Read Sheet').item.json.start }}`
  - `={{ $('Read Sheet').item.json.end }}`
  - `={{ $('Read Sheet').item.json.summary }}`
  - `={{ $('Read Sheet').item.json.location }}`
  - `={{ $('Read Sheet').item.json.description }}`
  - `={{ $json.attendees }}`
- **Input and output connections:**
  - Input: true branch from `Check Status Spreadsheets`
  - Output: `Check If Error`
- **Version-specific requirements:** Type version `1`; requires valid Google Calendar OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Invalid or non-ISO datetime values in `start` or `end`
  - `end` before `start`
  - Invalid attendee format
  - Unauthorized calendar access
  - API quota/rate-limit issues
  - Because `continueOnFail` is enabled, downstream logic must inspect the output carefully
  - `executeOnce: true` can be significant: in batch/item workflows, this setting may cause only one execution for the node rather than one per incoming item, depending on node behavior and n8n version. That is a major point to validate during reproduction.
- **Sub-workflow reference:** None.

**Important implementation note:**  
The node references values using `$('Read Sheet').item.json...` for most event fields instead of `$json...`. In many workflows this still resolves per item, but it is more fragile than directly using the current item context. If item linking changes or batched execution behaves unexpectedly, event fields may map incorrectly.

---

## Block 4 — Detect Calendar Errors and Update the Sheet

### Overview
This block interprets the result of event creation and writes the processing outcome back to the source spreadsheet. It updates each processed row to either `created` or `failed`, using `row_number` as the matching key.

### Nodes Involved
- Check If Error
- Update Sheet (Created)
- Update Sheet (Failed)

### Node Details

#### Check If Error
- **Type and technical role:** `n8n-nodes-base.if`; checks whether the previous node output contains an error indicator.
- **Configuration choices:**
  - Tests whether `{{$json.error}}` exists
  - If error exists → failed branch
  - If error does not exist → success branch
- **Key expressions or variables used:**
  - `={{$json.error}}`
- **Input and output connections:**
  - Input: `Create Event`
  - True output: `Update Sheet (Failed)`
  - False output: `Update Sheet (Created)`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - This logic assumes `continueOnFail` produces an `error` field on failed items
  - If the Google Calendar node returns a different error shape in a newer version, this test may fail silently
  - If a successful response unexpectedly contains an `error` property, rows could be misclassified
- **Sub-workflow reference:** None.

#### Update Sheet (Created)
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates a specific row in the spreadsheet to mark successful processing.
- **Configuration choices:**
  - Operation: `update`
  - Spreadsheet: same source spreadsheet
  - Sheet: `Sheet1` / `gid=0`
  - Matching column: `row_number`
  - Values written:
    - `status = created`
    - `row_number = {{ $('Check Status Spreadsheets').item.json.row_number }}`
  - Mapping mode is explicitly defined
- **Key expressions or variables used:**
  - `={{ $('Check Status Spreadsheets').item.json.row_number }}`
- **Input and output connections:**
  - Input: false branch from `Check If Error`
  - Output: none
- **Version-specific requirements:** Type version `4`.
- **Edge cases or potential failure types:**
  - If `row_number` is missing, the update will fail or update nothing
  - Spreadsheet schema mismatch can break mapping
  - If row order changes and `row_number` no longer matches the intended row, status may be written to the wrong record
  - Credential access errors or API limits
- **Sub-workflow reference:** None.

#### Update Sheet (Failed)
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates a specific row in the spreadsheet to mark failed processing.
- **Configuration choices:**
  - Operation: `update`
  - Spreadsheet: same source spreadsheet
  - Sheet: `Sheet1` / `gid=0`
  - Matching column: `row_number`
  - Values written:
    - `status = failed`
    - `row_number = {{ $('Check Status Spreadsheets').item.json.row_number }}`
  - Mapping mode is explicitly defined
- **Key expressions or variables used:**
  - `={{ $('Check Status Spreadsheets').item.json.row_number }}`
- **Input and output connections:**
  - Input: true branch from `Check If Error`
  - Output: none
- **Version-specific requirements:** Type version `4`.
- **Edge cases or potential failure types:**
  - Same as for the success-update node
  - Failures here can cause rows to remain in a retryable state without reflecting the actual calendar error
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual workflow start |  | Read Sheet | ## Bulk Google Calendar Event with Google Sheets<br>### How it works<br>1. **Manual Trigger** – Start workflow manually.<br>2. **Read Sheet** – Reads all rows from the configured Google Sheet.<br>3. **Check Status Spreadsheets** – Determines if each row is pending/failed or already processed:<br>- **Pending/Failed**: Proceed to create a calendar event.<br>- **Already Created**: Mark as duplicate.<br>6. **Create Event** – Uses row data (summary, start/end time, description, location, attendees) to create an event in Google Calendar.<br>7. **Check If Error** – Detects if the event creation succeeded:<br>- Success: Update row status as Created.<br>- Failure: Update row status as Failed.<br>### Setup steps<br>- [ ] Enable Google Sheets and Google Calendar APIs<br>- [ ] Set up credentials to connect to Google Sheets and Google Calendar<br>- [ ] Verify credentials are working properly<br>- [ ] Set up the Google Sheets file |
| Manual Trigger | Manual Trigger | Manual workflow start |  | Read Sheet | ## 1. Initialize workflow<br>Starts the workflow and sets initial variables. |
| Read Sheet | Google Sheets | Read all rows from the source spreadsheet | Manual Trigger | Check Status Spreadsheets | ## Bulk Google Calendar Event with Google Sheets<br>### How it works<br>1. **Manual Trigger** – Start workflow manually.<br>2. **Read Sheet** – Reads all rows from the configured Google Sheet.<br>3. **Check Status Spreadsheets** – Determines if each row is pending/failed or already processed:<br>- **Pending/Failed**: Proceed to create a calendar event.<br>- **Already Created**: Mark as duplicate.<br>6. **Create Event** – Uses row data (summary, start/end time, description, location, attendees) to create an event in Google Calendar.<br>7. **Check If Error** – Detects if the event creation succeeded:<br>- Success: Update row status as Created.<br>- Failure: Update row status as Failed.<br>### Setup steps<br>- [ ] Enable Google Sheets and Google Calendar APIs<br>- [ ] Set up credentials to connect to Google Sheets and Google Calendar<br>- [ ] Verify credentials are working properly<br>- [ ] Set up the Google Sheets file |
| Read Sheet | Google Sheets | Read all rows from the source spreadsheet | Manual Trigger | Check Status Spreadsheets | ## 2. Read Google Sheets |
| Check Status Spreadsheets | If | Allow only pending/failed rows to continue | Read Sheet | Create Event; No Operation, do nothing | ## 2. Read Google Sheets |
| Create Event | Google Calendar | Create a calendar event from row data | Check Status Spreadsheets | Check If Error | ## 3. Create Event to Calendar<br>Uses row data (summary, start/end time, description, location, attendees |
| Check If Error | If | Detect whether event creation returned an error | Create Event | Update Sheet (Created); Update Sheet (Failed) | ## 3. Create Event to Calendar<br>Uses row data (summary, start/end time, description, location, attendees |
| No Operation, do nothing | No Op | End branch for rows that should be skipped | Check Status Spreadsheets |  |  |
| Update Sheet (Created) | Google Sheets | Mark processed row as created | Check If Error |  | ## 4. Update Status to Google Sheets<br>This will update status if success will be `created` and if failed will be `failed` |
| Update Sheet (Failed) | Google Sheets | Mark processed row as failed | Check If Error |  | ## 4. Update Status to Google Sheets<br>This will update status if success will be `created` and if failed will be `failed` |
| Sticky Note | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation |  |  |  |
| Sticky Note6 | Sticky Note | Canvas documentation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: `Bulk Google Calendar Event with Google Sheets`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Leave default settings.
   - This will be the workflow entry point.

3. **Prepare the Google Sheet**
   - Create a Google Sheet with at least these columns:
     - `summary`
     - `start`
     - `end`
     - `description`
     - `location`
     - `attendees`
     - `status`
   - Ensure the Google Sheets node also exposes `row_number` when reading/updating rows.
   - Recommended status values:
     - `pending`
     - `failed`
     - `created`
   - Recommended datetime format:
     - ISO 8601, for example `2026-04-10T09:00:00+07:00`
   - Recommended attendees format:
     - Validate what the Google Calendar node expects in your n8n version. If one string field is used, it may need to contain a valid email or a list transformed before use.

4. **Add a Google Sheets node to read rows**
   - Node type: **Google Sheets**
   - Rename it to: `Read Sheet`
   - Connect `Manual Trigger -> Read Sheet`
   - Configure:
     - Credential: Google Sheets OAuth2
     - Select the spreadsheet document
     - Select the target sheet/tab
     - Use the read operation that returns sheet rows
   - Leave options default unless your sheet requires custom handling.

5. **Configure Google Sheets credentials**
   - Create or select a **Google Sheets OAuth2** credential.
   - Make sure the connected Google account has access to the spreadsheet.
   - Confirm the Google Sheets API is enabled in the Google Cloud project used by the credential.

6. **Add an If node to filter rows**
   - Node type: **If**
   - Rename it to: `Check Status Spreadsheets`
   - Connect `Read Sheet -> Check Status Spreadsheets`
   - Configure conditions:
     - Use **OR**
     - Condition 1:
       - Left value: `{{$json.status}}`
       - Operator: `equals`
       - Right value: `pending`
     - Condition 2:
       - Left value: `{{$json.status}}`
       - Operator: `equals`
       - Right value: `failed`
   - Keep case sensitivity in mind.

7. **Add a No Op node for skipped rows**
   - Node type: **No Op**
   - Rename it to: `No Operation, do nothing`
   - Connect the **false** output of `Check Status Spreadsheets` to this node.

8. **Add a Google Calendar node to create events**
   - Node type: **Google Calendar**
   - Rename it to: `Create Event`
   - Connect the **true** output of `Check Status Spreadsheets` to this node.
   - Configure:
     - Credential: Google Calendar OAuth2
     - Calendar: choose the target calendar
     - Start: `{{$json.start}}`  
       - The source workflow uses `$('Read Sheet').item.json.start`, but using `{{$json.start}}` is usually safer for item-based processing.
     - End: `{{$json.end}}`
     - Additional fields:
       - Summary: `{{$json.summary}}`
       - Description: `{{$json.description}}`
       - Location: `{{$json.location}}`
       - Attendees: map from `{{$json.attendees}}`
   - Enable **Continue On Fail**
     - This is required so failures are passed to downstream logic instead of stopping the workflow.
   - Carefully verify whether to enable **Execute Once**
     - The original workflow has it enabled.
     - In most bulk-processing scenarios, leaving it disabled is safer so each row creates one event.
     - If you copy the original exactly, test whether only one event is created. If so, disable `Execute Once`.

9. **Configure Google Calendar credentials**
   - Create or select a **Google Calendar OAuth2** credential.
   - Ensure the Google account has access to the target calendar.
   - Confirm the Google Calendar API is enabled in the Google Cloud project.

10. **Add an If node to detect errors**
    - Node type: **If**
    - Rename it to: `Check If Error`
    - Connect `Create Event -> Check If Error`
    - Configure condition:
      - Test whether `{{$json.error}}` **exists**
    - Interpretation:
      - **True** = event creation failed
      - **False** = event creation succeeded

11. **Add a Google Sheets node for successful updates**
    - Node type: **Google Sheets**
    - Rename it to: `Update Sheet (Created)`
    - Connect the **false** output of `Check If Error` to this node.
    - Configure:
      - Operation: **Update**
      - Same spreadsheet and sheet as the read node
      - Matching column: `row_number`
      - Set values:
        - `status = created`
        - `row_number = {{$json.row_number}}`
    - If the node requires schema mapping:
      - Include the available sheet columns
      - Mark `row_number` as the match column

12. **Add a Google Sheets node for failed updates**
    - Node type: **Google Sheets**
    - Rename it to: `Update Sheet (Failed)`
    - Connect the **true** output of `Check If Error` to this node.
    - Configure:
      - Operation: **Update**
      - Same spreadsheet and sheet as the read node
      - Matching column: `row_number`
      - Set values:
        - `status = failed`
        - `row_number = {{$json.row_number}}`
    - As above, ensure schema mapping includes `row_number`.

13. **Prefer current-item expressions for update nodes**
    - The source workflow uses:
      - `{{ $('Check Status Spreadsheets').item.json.row_number }}`
    - In a standard item flow, `{{$json.row_number}}` is often simpler and less fragile.
    - If reproducing exactly, use the original cross-node expression.
    - If rebuilding for reliability, prefer current-item expressions after confirming the item passed through unchanged.

14. **Test with sample rows**
    - Add a few rows in the sheet:
      - one with `status = pending`
      - one with `status = failed`
      - one with `status = created`
    - Run the workflow manually.
    - Confirm:
      - only pending/failed rows are processed
      - successful rows become `created`
      - failed rows become `failed`
      - created rows are skipped

15. **Validate data formatting**
    - Confirm `start` and `end` are valid datetime strings acceptable to Google Calendar.
    - Confirm attendee values match the Google Calendar node requirement in your n8n version.
    - If attendees are stored as comma-separated emails in the sheet, you may need an intermediate transformation node not present in this workflow.

16. **Optional hardening improvements**
    - Add a node before event creation to validate required fields.
    - Normalize status values to lowercase.
    - Split or transform attendee strings into the exact structure required by the calendar node.
    - Record the created Google Calendar event ID back into the sheet.
    - Add deduplication logic beyond status, such as matching by summary + start datetime.

### Expected Input/Output Behavior

- **Input source:** Google Sheet rows
- **Required row fields:** `summary`, `start`, `end`, `status`
- **Optional row fields:** `description`, `location`, `attendees`
- **Success path output:** row updated with `status = created`
- **Failure path output:** row updated with `status = failed`

### Sub-workflow Setup

This workflow does **not** use any sub-workflows and does not invoke any external n8n workflow via Execute Workflow nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Enable both Google Sheets API and Google Calendar API before testing. | Google Cloud project used by OAuth credentials |
| The source sheet in the workflow is an example file named `[EXAMPLE] Bulk Create Event Google Calendar`. | Example spreadsheet referenced in the workflow |
| The workflow’s own canvas notes describe the logic as: read rows, process only `pending` or `failed`, create events, then update status to `created` or `failed`. | Internal canvas documentation |
| One canvas note states “Already Created: Mark as duplicate,” but the actual implemented logic does not write any duplicate marker. It only skips rows through a No Op branch. | Important behavior clarification |
| The Google Calendar node has `continueOnFail` enabled, which is necessary for downstream error branching. | Error-handling design |
| The Google Calendar node also has `executeOnce` enabled in the exported workflow. This should be verified carefully because it may reduce bulk behavior depending on n8n version and node execution semantics. | Reproduction and validation note |