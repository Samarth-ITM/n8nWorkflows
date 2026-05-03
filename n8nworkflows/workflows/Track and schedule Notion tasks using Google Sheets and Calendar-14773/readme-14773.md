Track and schedule Notion tasks using Google Sheets and Calendar

https://n8nworkflows.xyz/workflows/track-and-schedule-notion-tasks-using-google-sheets-and-calendar-14773


# Track and schedule Notion tasks using Google Sheets and Calendar

# 1. Workflow Overview

This workflow monitors a Notion database for task updates, checks whether each task’s scheduled due date conflicts with an existing Google Calendar event, logs the task in Google Sheets, and then either creates a calendar event or sends a conflict email. It is designed for lightweight task scheduling where Notion acts as the task source, Google Calendar as the scheduling layer, Google Sheets as the audit/status log, and Gmail as the conflict notification channel.

Typical use cases:
- Turning Notion tasks with due dates into calendar events
- Preventing double-booking before event creation
- Keeping a simple operational record of sync success/failure
- Alerting the user when a requested slot is already occupied

## 1.1 Input Reception and Source Monitoring

The workflow starts with a Notion Trigger that watches a specific database for page updates. Each trigger payload contains the full page structure, including the task name, due date, created metadata, and page URL.

## 1.2 Calendar Availability Check

The workflow queries Google Calendar for events in the one-hour window starting at the Notion task’s due date. This determines whether the desired slot is free before any event is created.

## 1.3 Logging and Decision Routing

After the availability lookup, the workflow appends a tracking row to Google Sheets with the task metadata and a default availability value. Then an If node evaluates the availability result from the calendar lookup and routes execution to either the success or conflict branch.

## 1.4 Success Path: Event Creation and Status Update

If the slot is considered available, the workflow creates a one-hour Google Calendar event and updates the corresponding Google Sheets row to mark availability as `True`.

## 1.5 Conflict Path: Notification and Failure Status

If the slot is not available, the workflow sends an HTML email through Gmail explaining the calendar conflict and then updates the Google Sheets row to mark availability as `False`.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Notion Task Intake

**Overview:**  
This block detects updates in the configured Notion database and emits a page payload for downstream processing. It is the only entry point of the workflow.

**Nodes Involved:**  
- Get Task

### Node Details

#### Get Task
- **Type and technical role:** `n8n-nodes-base.notionTrigger`  
  Polling trigger that watches a Notion database for updated pages.
- **Configuration choices:**  
  - Event: `pagedUpdatedInDatabase`
  - Polling frequency: every minute
  - Database: specific Notion database selected by ID
  - `simple: false`, so the full Notion page payload is preserved
- **Key expressions or variables used:**  
  No output expression is configured inside the node, but downstream nodes depend heavily on:
  - `{{$json.id}}`
  - `{{$json.created_by.id}}`
  - `{{$json.created_time}}`
  - `{{$json.url}}`
  - `{{$json.parent.database_id}}`
  - `{{$json.properties['Due date'].date.start}}`
  - `{{$json.properties['Task name'].title[0].text.content}}`
  - `{{$json.properties['Task name'].title[0].plain_text}}`
  - `{{$json.properties.Priority.select.name}}`
- **Input and output connections:**  
  - Input: none, trigger node
  - Output: `Check Availability`
- **Version-specific requirements:**  
  - Type version: `1`
  - Requires a supported Notion integration/credential with access to the selected database
- **Edge cases or potential failure types:**  
  - Notion credential/auth failure
  - Database not shared with the Notion integration
  - Missing or renamed properties such as `Due date`, `Task name`, or `Priority`
  - Empty title arrays causing indexing failures like `title[0]`
  - Null due dates causing downstream date expression failures
  - Trigger noise: any page update may retrigger, potentially creating duplicate log rows or calendar events
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: Calendar Availability Lookup

**Overview:**  
This block checks Google Calendar for any event occupying the one-hour period starting from the task due date. Its purpose is to determine whether the requested time slot can be safely scheduled.

**Nodes Involved:**  
- Check Availability

### Node Details

#### Check Availability
- **Type and technical role:** `n8n-nodes-base.googleCalendar`  
  Calendar query node used here in `resource: calendar` mode to inspect availability in a time range.
- **Configuration choices:**  
  - Calendar: selected target Google Calendar
  - Resource: `calendar`
  - `timeMin` = Notion due date start
  - `timeMax` = one hour after the due date start
- **Key expressions or variables used:**  
  - `={{ $json.properties['Due date'].date.start }}`
  - `={{ $json.properties['Due date'].date.start.toDateTime().plus(1, 'hours').toISO() }}`
- **Input and output connections:**  
  - Input: `Get Task`
  - Output: `Record`
- **Version-specific requirements:**  
  - Type version: `1.3`
  - Requires Google Calendar credentials with access to the chosen calendar
  - Uses n8n date helper methods such as `.toDateTime()` and `.toISO()`
- **Edge cases or potential failure types:**  
  - Invalid or null Notion date
  - Timezone mismatches between Notion date value and Google Calendar expectations
  - Calendar permission issues
  - Unclear semantic behavior of availability output depending on Google Calendar node mode/version
  - If the date is all-day or lacks a time component, the one-hour assumption may be incorrect
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Logging and Conditional Decision

**Overview:**  
This block records the incoming task in Google Sheets and then decides whether to create a calendar event or handle a conflict. It forms the operational control point of the workflow.

**Nodes Involved:**  
- Record
- If

### Node Details

#### Record
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends a row to a Google Sheet to track task processing.
- **Configuration choices:**  
  - Operation: `append`
  - Document: chosen spreadsheet
  - Sheet: `gid=0` / `Sheet1`
  - Mapping mode: manual field mapping
  - Stores:
    - `ID`
    - `Due_Date`
    - `Priority`
    - `Task_Name`
    - `Created_By`
    - `Avaliabilty`
    - `Database_ID`
    - `Created_Time`
  - Default inserted `Avaliabilty`: `"0"`
- **Key expressions or variables used:**  
  - `={{ $('Get Task').item.json.id }}`
  - `={{ $('Get Task').item.json.properties['Due date'].date.start }}`
  - `={{ $('Get Task').item.json.properties.Priority.select.name }}`
  - `={{ $('Get Task').item.json.properties['Task name'].title[0].text.content }}`
  - `={{ $('Get Task').item.json.created_by.id }}`
  - `={{ $('Get Task').item.json.parent.database_id }}`
  - `={{ $('Get Task').item.json.created_time }}`
- **Input and output connections:**  
  - Input: `Check Availability`
  - Output: `If`
- **Version-specific requirements:**  
  - Type version: `4.7`
  - Requires Google Sheets credentials with write access
- **Edge cases or potential failure types:**  
  - Spreadsheet or sheet selection invalid
  - Missing columns in the target sheet
  - Typo in column name `Avaliabilty` must match the sheet exactly
  - Duplicate rows if the same Notion page updates repeatedly
  - Missing Notion properties causing append expressions to fail
- **Sub-workflow reference:**  
  None

#### If
- **Type and technical role:** `n8n-nodes-base.if`  
  Evaluates a boolean condition and routes to success or conflict.
- **Configuration choices:**  
  - Condition type: boolean true
  - Condition source: `$('Check Availability').item.json.available`
  - Strict validation enabled
  - True branch: create event
  - False branch: send conflict email
- **Key expressions or variables used:**  
  - `={{ $('Check Availability').item.json.available }}`
- **Input and output connections:**  
  - Input: `Record`
  - True output: `Create an event`
  - False output: `Send Error`
- **Version-specific requirements:**  
  - Type version: `2.3`
- **Edge cases or potential failure types:**  
  - The Google Calendar node must actually return an `available` boolean in the expected shape
  - If the upstream output shape differs, this condition may evaluate incorrectly or fail
  - Strict type validation may reject string values like `"true"` instead of boolean `true`
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: Success Path

**Overview:**  
When the calendar slot is available, this branch creates the Google Calendar event and marks the corresponding tracking row as successful.

**Nodes Involved:**  
- Create an event
- Update Status

### Node Details

#### Create an event
- **Type and technical role:** `n8n-nodes-base.googleCalendar`  
  Creates a new event in Google Calendar.
- **Configuration choices:**  
  - Start time: from sheet field `Due_Date`
  - End time: one hour after `Due_Date`
  - Calendar: selected target Google Calendar
  - Additional description: task name
- **Key expressions or variables used:**  
  - `={{ $json.Due_Date }}`
  - `={{ $json.Due_Date.toDateTime().plus({ hours: 1 }).toISO() }}`
  - `={{ $json.Task_Name }}`
- **Input and output connections:**  
  - Input: `If` true branch
  - Output: `Update Status`
- **Version-specific requirements:**  
  - Type version: `1.3`
- **Edge cases or potential failure types:**  
  - `Due_Date` must be in a parseable date/time format after coming from Google Sheets append output
  - If the Google Sheets node returns strings in unexpected format, `.toDateTime()` may fail
  - Calendar permission or quota errors
  - Duplicate event creation if the workflow is retriggered for the same task
  - No event title is explicitly set; only description is mapped, so the event may get a default or blank summary depending on node defaults
- **Sub-workflow reference:**  
  None

#### Update Status
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the previously logged row to mark the task as available/synced.
- **Configuration choices:**  
  - Operation: `update`
  - Matching column: `ID`
  - Updates:
    - `ID`
    - `Avaliabilty = True`
- **Key expressions or variables used:**  
  - `={{ $('Get Task').item.json.id }}`
  - `=True`
- **Input and output connections:**  
  - Input: `Create an event`
  - Output: none
- **Version-specific requirements:**  
  - Type version: `4.7`
- **Edge cases or potential failure types:**  
  - Matching depends on the `ID` row inserted earlier
  - If multiple rows share the same ID, update behavior may be ambiguous
  - `True` is entered as a string/expression result and may not be a native boolean in Sheets
  - Typo in `Avaliabilty` must match the actual sheet column
- **Sub-workflow reference:**  
  None

---

## 2.5 Block: Conflict Handling

**Overview:**  
If the time slot is not available, the workflow sends a formatted Gmail alert and updates the tracking row to indicate failure. This provides user visibility without creating a conflicting calendar event.

**Nodes Involved:**  
- Send Error
- Update Status1

### Node Details

#### Send Error
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an HTML email via Gmail to report the scheduling conflict.
- **Configuration choices:**  
  - Recipient: fixed configured email address
  - Subject: `Your Task clashes with existing plans`
  - Message: rich HTML email with:
    - task title
    - requested date/time
    - link back to the Notion page
- **Key expressions or variables used:**  
  - `{{ $('Get Task').item.json.properties['Task name'].title[0].plain_text }}`
  - `{{ $('Get Task').item.json.properties['Due date'].date.start }}`
  - `{{ $('Get Task').item.json.url }}`
- **Input and output connections:**  
  - Input: `If` false branch
  - Output: `Update Status1`
- **Version-specific requirements:**  
  - Type version: `2.2`
  - Requires Gmail credentials configured for sending mail
- **Edge cases or potential failure types:**  
  - Gmail auth or send quota issues
  - HTML rendering differences across email clients
  - Missing Notion title/date fields breaking interpolation
  - Hard-coded recipient means no per-user routing
- **Sub-workflow reference:**  
  None

#### Update Status1
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the logged row to mark availability as `False`.
- **Configuration choices:**  
  - Operation: `update`
  - Matching column: `ID`
  - Updates:
    - `ID`
    - `Avaliabilty = False`
- **Key expressions or variables used:**  
  - `={{ $('Get Task').item.json.id }}`
  - `=False`
- **Input and output connections:**  
  - Input: `Send Error`
  - Output: none
- **Version-specific requirements:**  
  - Type version: `4.7`
- **Edge cases or potential failure types:**  
  - Same matching and duplication concerns as the success update node
  - `False` may be stored as a string depending on Sheets behavior
  - Sheet column naming must exactly match `Avaliabilty`
- **Sub-workflow reference:**  
  None

---

## 2.6 Block: In-Canvas Documentation

**Overview:**  
These sticky notes document the workflow purpose and visually label each processing stage. They do not affect runtime behavior but are useful for operators and maintainers.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation note for the workflow.
- **Configuration choices:**  
  Contains a full description of workflow behavior and setup guidance.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  - Type version: `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels Step 1 as fetch and availability check.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels Step 2 as logging and decision.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels Step 3 as event creation and sync status update.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Labels Step 4 as conflict handling.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get Task | Notion Trigger | Watches a Notion database for updated task pages |  | Check Availability | ## Notion Task → Google Calendar Sync<br>### How it works<br>This workflow automatically syncs tasks from your Notion database to Google Calendar. When a task is created or updated, the workflow checks if the selected time slot is available in your calendar. It logs the task data into Google Sheets for tracking and then uses a conditional check to determine the next step.<br>If the time slot is available, the workflow creates a new calendar event and updates the status as successful. If there is a conflict, it sends an email notification and updates the record as failed. This ensures you never double-book your schedule and always have visibility into sync status.<br>### Setup steps<br>1. Connect your Notion, Google Calendar, Google Sheets, and Gmail accounts.<br>2. Select your Notion database and ensure it includes a date field.<br>3. Configure the Google Calendar node with your target calendar.<br>4. Map the Google Sheets columns correctly (ID, Task Name, Due Date, Status).<br>5. Set the IF node to check if no events exist (availability = true).<br>6. Customize the Gmail node recipient and message if needed.<br>## Step 1: Fetch & Check<br>Trigger + calendar availability check |
| Check Availability | Google Calendar | Checks whether the requested one-hour slot is free | Get Task | Record | ## Step 1: Fetch & Check<br>Trigger + calendar availability check |
| Record | Google Sheets | Logs task metadata to the tracking spreadsheet | Check Availability | If | ## Step 2: Log & Decide<br>Store data and evaluate availability |
| If | If | Routes execution based on calendar availability | Record | Create an event; Send Error | ## Step 2: Log & Decide<br>Store data and evaluate availability |
| Create an event | Google Calendar | Creates a calendar event for the task | If | Update Status | ## Step 3: Create Event<br>Add event and mark as synced |
| Update Status | Google Sheets | Marks the logged task row as successful/available | Create an event |  | ## Step 3: Create Event<br>Add event and mark as synced |
| Send Error | Gmail | Sends a conflict notification email | If | Update Status1 | ## Step 4: Handle Conflict<br>Send alert and mark as failed |
| Update Status1 | Google Sheets | Marks the logged task row as failed/unavailable | Send Error |  | ## Step 4: Handle Conflict<br>Send alert and mark as failed |
| Sticky Note | Sticky Note | Visual documentation |  |  |  |
| Sticky Note1 | Sticky Note | Visual block label |  |  |  |
| Sticky Note2 | Sticky Note | Visual block label |  |  |  |
| Sticky Note3 | Sticky Note | Visual block label |  |  |  |
| Sticky Note4 | Sticky Note | Visual block label |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Track and schedule Notion tasks using Google Sheets and Calendar`.

2. **Add a Notion Trigger node** and name it `Get Task`.
   - Node type: **Notion Trigger**
   - Event: `Page Updated in Database`
   - Polling: every minute
   - Database: select your Notion task database
   - Disable simplified output if needed so the full page payload is available (`simple: false`)
   - Credentials: connect a Notion account/integration with access to the database

3. **Ensure the Notion database contains the expected properties.**
   At minimum, create these fields:
   - `Task name` as a title property
   - `Due date` as a date property
   - `Priority` as a select property  
   The expressions in this workflow assume those names exactly.

4. **Add a Google Calendar node** and name it `Check Availability`.
   - Connect `Get Task` → `Check Availability`
   - Node type: **Google Calendar**
   - Resource: set to the calendar/availability mode used by your n8n version
   - Calendar: choose the target calendar
   - Set `timeMin` to the Notion due date start:
     - `{{$json.properties['Due date'].date.start}}`
   - Set `timeMax` to one hour later:
     - `{{$json.properties['Due date'].date.start.toDateTime().plus(1, 'hours').toISO()}}`
   - Credentials: connect Google Calendar OAuth2 credentials

5. **Confirm the output shape of the Google Calendar availability node.**
   - The later If node expects a boolean field named `available`
   - If your node version instead returns events rather than an `available` boolean, you must adapt the logic, for example by checking whether zero events were returned

6. **Create a Google Sheet** for tracking.
   Add a worksheet with columns exactly matching:
   - `ID`
   - `Created_Time`
   - `Created_By`
   - `Database_ID`
   - `Task_Name`
   - `Priority`
   - `Due_Date`
   - `Avaliabilty`  
   Note: the workflow uses the misspelled column name `Avaliabilty`. Keep it as-is unless you also update every mapping.

7. **Add a Google Sheets node** and name it `Record`.
   - Connect `Check Availability` → `Record`
   - Operation: `Append`
   - Select the spreadsheet document
   - Select the target sheet
   - Use manual column mapping
   - Map values as follows:
     - `ID` → `{{$('Get Task').item.json.id}}`
     - `Due_Date` → `{{$('Get Task').item.json.properties['Due date'].date.start}}`
     - `Priority` → `{{$('Get Task').item.json.properties.Priority.select.name}}`
     - `Task_Name` → `{{$('Get Task').item.json.properties['Task name'].title[0].text.content}}`
     - `Created_By` → `{{$('Get Task').item.json.created_by.id}}`
     - `Avaliabilty` → `0`
     - `Database_ID` → `{{$('Get Task').item.json.parent.database_id}}`
     - `Created_Time` → `{{$('Get Task').item.json.created_time}}`
   - Credentials: connect Google Sheets credentials

8. **Add an If node** and name it `If`.
   - Connect `Record` → `If`
   - Configure a boolean condition
   - Left value:
     - `{{$('Check Availability').item.json.available}}`
   - Operator: `is true`
   - Keep strict type validation if your availability field is a real boolean
   - True output will be the success path
   - False output will be the conflict path

9. **Add a Google Calendar node** and name it `Create an event`.
   - Connect the **true** output of `If` → `Create an event`
   - Operation: create event
   - Calendar: same target calendar
   - Start:
     - `{{$json.Due_Date}}`
   - End:
     - `{{$json.Due_Date.toDateTime().plus({ hours: 1 }).toISO()}}`
   - Additional field `Description`:
     - `{{$json.Task_Name}}`
   - Credentials: same Google Calendar OAuth2 account

10. **Decide whether you also want to set an event title explicitly.**
    - In the provided workflow, only the description is mapped
    - In practice, you may want to set the event summary/title to `{{$json.Task_Name}}` as well

11. **Add a Google Sheets node** and name it `Update Status`.
    - Connect `Create an event` → `Update Status`
    - Operation: `Update`
    - Spreadsheet: same tracking sheet
    - Matching column: `ID`
    - Map:
      - `ID` → `{{$('Get Task').item.json.id}}`
      - `Avaliabilty` → `True`
    - This updates the row inserted by `Record`

12. **Add a Gmail node** and name it `Send Error`.
    - Connect the **false** output of `If` → `Send Error`
    - Operation: send email
    - To: set your recipient email address
    - Subject: `Your Task clashes with existing plans`
    - Message format: HTML
    - Use an HTML body that includes:
      - task name
      - due date
      - Notion URL
    - Key expressions:
      - `{{$('Get Task').item.json.properties['Task name'].title[0].plain_text}}`
      - `{{$('Get Task').item.json.properties['Due date'].date.start}}`
      - `{{$('Get Task').item.json.url}}`
    - Credentials: connect Gmail OAuth2 credentials

13. **Add another Google Sheets node** and name it `Update Status1`.
    - Connect `Send Error` → `Update Status1`
    - Operation: `Update`
    - Spreadsheet: same tracking sheet
    - Matching column: `ID`
    - Map:
      - `ID` → `{{$('Get Task').item.json.id}}`
      - `Avaliabilty` → `False`

14. **Verify the full connection order.**
   - `Get Task` → `Check Availability` → `Record` → `If`
   - `If` true → `Create an event` → `Update Status`
   - `If` false → `Send Error` → `Update Status1`

15. **Optionally add sticky notes** for maintainability.
   Suggested notes:
   - A general workflow description and setup checklist
   - `Step 1: Fetch & Check`
   - `Step 2: Log & Decide`
   - `Step 3: Create Event`
   - `Step 4: Handle Conflict`

16. **Configure credentials explicitly.**
   - **Notion:** integration token with database access
   - **Google Calendar:** OAuth2 with calendar read/write permissions
   - **Google Sheets:** OAuth2 with spreadsheet read/write permissions
   - **Gmail:** OAuth2 with mail send permissions

17. **Test with a sample Notion task.**
   - Create or update a page in the Notion database
   - Ensure it contains:
     - a title in `Task name`
     - a valid date/time in `Due date`
     - a selected `Priority`
   - Check whether:
     - a row is appended to Google Sheets
     - the If condition routes correctly
     - an event is created if free
     - an email is sent if conflicting
     - the sheet row is updated accordingly

18. **Harden the workflow before production use.**
   Recommended improvements:
   - Add a filter so only tasks with non-empty due dates are processed
   - Add deduplication logic to avoid repeated event creation on every page update
   - Use a more reliable conflict check if your Google Calendar node does not return `available`
   - Add explicit error handling for missing task title/date
   - Consider storing the created Google Calendar event ID back in Sheets or Notion

19. **Sub-workflow setup**
   - This workflow does **not** use any Execute Workflow or sub-workflow nodes
   - No sub-workflow parameters are required

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Notion Task → Google Calendar Sync: This workflow automatically syncs tasks from your Notion database to Google Calendar, checks slot availability, logs task data into Google Sheets, creates the event when available, or sends an email when there is a conflict. | General workflow purpose |
| Setup guidance from the canvas note: connect Notion, Google Calendar, Google Sheets, and Gmail accounts; select the Notion database; configure the calendar; map Google Sheets columns; set the If node to use availability; customize the Gmail recipient and message. | Internal workflow documentation |
| Step 1: Fetch & Check | Visual block label |
| Step 2: Log & Decide | Visual block label |
| Step 3: Create Event | Visual block label |
| Step 4: Handle Conflict | Visual block label |

## Additional implementation notes

- The workflow has **one entry point only**: the Notion Trigger node.
- There are **no sub-workflows**.
- The workflow assumes a **one-hour task duration** for every event.
- The Google Sheets column name is intentionally spelled `Avaliabilty`; changing it requires updating all related mappings.
- Because the trigger fires on page updates, this workflow may create duplicate records/events unless additional safeguards are added.
- The success path uses data from the `Record` node output, while status updates still reference `Get Task` for matching the original Notion page ID.
- The core risk area is the assumption that `Check Availability` returns `available` as a boolean. Validate this behavior in your n8n version before relying on the If logic.