Convert Google Tasks to Google Calendar Events via Webhook Integration

https://n8nworkflows.xyz/workflows/convert-google-tasks-to-google-calendar-events-via-webhook-integration-7865


# Convert Google Tasks to Google Calendar Events via Webhook Integration

---

## 1. Workflow Overview

This workflow automates the conversion of Google Tasks into Google Calendar events triggered by incoming webhook requests. It targets scenarios where users want to seamlessly synchronize task deadlines as calendar events, facilitating better schedule management without manual duplication.

The workflow logic is organized into six functional blocks:

- **1.1 Input Reception:** Receives incoming JSON payloads via a webhook containing task details.
- **1.2 Configuration Setup:** Centralizes key variables such as Task List ID, Calendar ID, event duration, and color.
- **1.3 Date Formatting:** Converts task due timestamp (epoch seconds) into a properly formatted datetime string.
- **1.4 Duplicate Detection:** Searches recent calendar events to prevent creating duplicate events for the same task.
- **1.5 Event Creation:** Creates a new Google Calendar event based on the task details and formatted date.
- **1.6 Cleanup:** Optionally deletes the original Google Task after event creation to maintain task list hygiene.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block receives incoming HTTP POST requests containing task information to trigger the workflow.

**Nodes Involved:**  
- Webhook

**Node Details:**  

- **Webhook**
  - Type: Webhook node  
  - Role: Starts the workflow by receiving a POST request at path `/task-to-calendar`.  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `task-to-calendar`  
  - Key Variables:  
    - Expects JSON body with keys:  
      - `TaskName` (string): Task title  
      - `DueDateTimeSeconds` (integer): Due date as Unix timestamp in seconds  
  - Input/Output:  
    - Input: External HTTP POST request  
    - Output: JSON payload forwarded downstream  
  - Edge Cases / Failures:  
    - Missing or malformed JSON payload  
    - Incorrect or missing keys could break downstream expressions  
  - Sticky Note Reference:  
    - Step 1 – Trigger (Webhook) explains example payload and alternative triggers

---

### 2.2 Configuration Setup

**Overview:**  
Defines and centralizes configurable parameters for the workflow such as Google Tasklist ID, target Calendar ID, event duration, and color coding.

**Nodes Involved:**  
- Configuration

**Node Details:**  

- **Configuration**  
  - Type: Set node  
  - Role: Holds static configuration data as workflow variables  
  - Configuration:  
    - `defaultMinutes`: 30 (event duration in minutes)  
    - `eventColor`: 8 (Google Calendar color index)  
    - `calendarId`: "primary" (default target calendar)  
    - `tasklistId`: Placeholder `"REPLACE_WITH_TASKLIST_ID"` (Google Tasks list identifier)  
  - Key Expressions: None; values are static except for tasklistId replacement  
  - Input/Output:  
    - Input: Data from Webhook node  
    - Output: Configuration object for downstream nodes  
  - Edge Cases / Failures:  
    - Forgetting to replace `tasklistId` breaks Google Tasks queries  
  - Sticky Note Reference:  
    - Step 2 – Configuration explains variables and usage

---

### 2.3 Date Formatting

**Overview:**  
Converts the incoming Unix timestamp for task due date into a formatted datetime string; this is used for event start time.

**Nodes Involved:**  
- Google Tasks  
- Date & Time

**Node Details:**  

- **Google Tasks**  
  - Type: Google Tasks node  
  - Role: Retrieves all tasks from specified Google Tasklist, filtering to incomplete tasks  
  - Configuration:  
    - Operation: `getAll`  
    - Tasklist: Value taken dynamically from `Configuration.tasklistId`  
    - Additional: `showCompleted` set to false to exclude completed tasks  
  - Credentials: Google Tasks OAuth2  
  - Input/Output:  
    - Input: Configuration output  
    - Output: List of tasks, each with metadata including title and id  
  - Edge Cases:  
    - Invalid or expired OAuth credentials  
    - Empty task list  
- **Date & Time**  
  - Type: DateTime node  
  - Role: Formats the task due date from epoch seconds to `yyyy-MM-dd HH:mm:ss` string  
  - Configuration:  
    - Operation: `formatDate`  
    - Date input expression: Converts `Webhook.body.DueDateTimeSeconds` from seconds to datetime  
    - Output format: Custom `"yyyy-MM-dd HH:mm:ss"`  
  - Input/Output:  
    - Input: Google Tasks output (indirect) and Webhook data  
    - Output: Formatted date string for event start  
  - Edge Cases:  
    - Missing or invalid `DueDateTimeSeconds` field  
  - Sticky Note Reference:  
    - Step 3 – Date explains formatting and event duration logic

---

### 2.4 Duplicate Detection

**Overview:**  
Checks for existing events matching the task title within a recent time window to prevent duplicate calendar entries.

**Nodes Involved:**  
- Google Calendar9  
- If1

**Node Details:**  

- **Google Calendar9**  
  - Type: Google Calendar node  
  - Role: Searches for calendar events with a title matching the incoming task name within last 5 minutes  
  - Configuration:  
    - Operation: `getAll`  
    - Limit: 3  
    - Query: Matches `Webhook.body.TaskName`  
    - UpdatedMin: Current time minus 5 minutes (recent events)  
    - ShowDeleted: false  
    - Calendar: Uses `Configuration.calendarId` (default "primary")  
  - Credentials: Google Calendar OAuth2  
  - Input/Output:  
    - Input: Date & Time node output  
    - Output: List of matching events  
  - Edge Cases:  
    - OAuth token expiration  
    - No events found (empty output)  
- **If1**  
  - Type: If node  
  - Role: Conditional to check if any existing event matches task summary to avoid duplicates  
  - Configuration:  
    - Conditions:  
      - Checks if event summary exists  
      - Summary equals incoming `TaskName` from webhook  
  - Input/Output:  
    - Input: Output from Google Calendar9  
    - Output:  
      - False branch: No duplicates → proceed to create event  
      - True branch: Duplicate found → skip creation  
  - Edge Cases:  
    - Case sensitivity might cause false negatives/positives  
  - Sticky Note Reference:  
    - Step 4 – Dedupe explains searching and duplicate guard logic

---

### 2.5 Event Creation

**Overview:**  
Creates a Google Calendar event for the task using the formatted start time and configured duration and color.

**Nodes Involved:**  
- Google Calendar

**Node Details:**  

- **Google Calendar**  
  - Type: Google Calendar node  
  - Role: Creates a new event in the configured calendar  
  - Configuration:  
    - Operation: Create event (implicit by parameters)  
    - Start: From `Date & Time.formattedDate`  
    - End: Start time plus `defaultMinutes` from Configuration  
    - Calendar: Uses configured `calendarId`  
    - Additional Fields:  
      - Summary: Task title from Google Tasks node (`title`)  
      - Color: `eventColor` from Configuration  
  - Credentials: Google Calendar OAuth2  
  - Input/Output:  
    - Input: False branch from If1 (no duplicate)  
    - Output: Created event metadata  
  - Edge Cases:  
    - OAuth token expiration  
    - Invalid time format or calendar ID  
  - Sticky Note Reference:  
    - Step 5 – Create Event explains event creation details

---

### 2.6 Cleanup

**Overview:**  
Optionally deletes the original Google Task after successfully creating the calendar event, preventing task list clutter.

**Nodes Involved:**  
- Google Tasks1

**Node Details:**  

- **Google Tasks1**  
  - Type: Google Tasks node  
  - Role: Deletes the original task by its ID  
  - Configuration:  
    - Operation: `delete`  
    - Tasklist ID: Static string `"MTQ4MDk5NDE1Mjc2NjA5MTA5MjE6MDow"` (this appears hardcoded, should be replaced or parameterized)  
    - Task ID: Extracted dynamically from the Google Tasks node (`id`)  
  - Credentials: Google Tasks OAuth2  
  - Input/Output:  
    - Input: Output from Google Calendar node (post event creation)  
    - Output: Confirmation of deletion  
  - Edge Cases:  
    - Hardcoded tasklist ID may cause failure if not matching actual tasklist  
    - Deletion failure if task already removed or permission denied  
  - Sticky Note Reference:  
    - Step 6 – Cleanup explains optional deletion purpose

---

## 3. Summary Table

| Node Name         | Node Type          | Functional Role                   | Input Node(s)       | Output Node(s)       | Sticky Note                                                                                  |
|-------------------|--------------------|---------------------------------|---------------------|----------------------|----------------------------------------------------------------------------------------------|
| Template Notes    | Sticky Note        | Template description             |                     |                      | Describes template purpose and setup steps                                                  |
| Webhook           | Webhook            | Input reception via HTTP POST   |                     | Configuration        | Step 1 – Trigger (Webhook): example payload and trigger explanation                         |
| Configuration     | Set                | Centralized configuration       | Webhook             | Google Tasks         | Step 2 – Configuration: variables for tasklistId, calendarId, event duration, color         |
| Google Tasks      | Google Tasks       | Retrieve tasks from tasklist    | Configuration       | Date & Time          |                                                                                              |
| Date & Time       | DateTime           | Format due date timestamp       | Google Tasks        | Google Calendar9     | Step 3 – Date formatting: converts epoch seconds to formatted date string                   |
| Google Calendar9  | Google Calendar    | Search for duplicates           | Date & Time         | If1                  | Step 4 – Duplicate guard: searches recent events to prevent duplicates                      |
| If1               | If                 | Conditional duplicate detection | Google Calendar9    | Google Calendar      |                                                                                              |
| Google Calendar   | Google Calendar    | Create calendar event           | If1 (false branch)  | Google Tasks1        | Step 5 – Create event: event creation with start, end, color, and summary                   |
| Google Tasks1     | Google Tasks       | Delete original task            | Google Calendar     |                      | Step 6 – Cleanup: optional deletion of Google Task after event creation                     |
| Step 1 – Trigger  | Sticky Note        | Documentation                   |                     |                      | Explains Webhook trigger and sample JSON payload                                           |
| Step 2 – Configuration | Sticky Note    | Documentation                   |                     |                      | Explains configuration variables                                                          |
| Step 3 – Date     | Sticky Note        | Documentation                   |                     |                      | Explains date formatting logic                                                            |
| Step 4 – Dedupe   | Sticky Note        | Documentation                   |                     |                      | Explains duplicate detection logic                                                        |
| Step 5 – Create Event | Sticky Note     | Documentation                   |                     |                      | Explains event creation parameters                                                        |
| Step 6 – Cleanup  | Sticky Note        | Documentation                   |                     |                      | Explains optional cleanup step                                                            |
| Flow Overview     | Sticky Note        | Documentation                   |                     |                      | Summarizes overall flow steps                                                             |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Node Type: Webhook  
   - Name: `Webhook`  
   - HTTP Method: POST  
   - Path: `task-to-calendar`  
   - Purpose: Receive JSON payload with keys `TaskName` and `DueDateTimeSeconds`.

2. **Create Set Node for Configuration**  
   - Node Type: Set  
   - Name: `Configuration`  
   - Add variables:  
     - `defaultMinutes` (Number): 30  
     - `eventColor` (Number): 8  
     - `calendarId` (String): "primary" (or your calendar ID)  
     - `tasklistId` (String): Replace with your Google Tasklist ID  
   - Connect `Webhook` output to `Configuration` input.

3. **Create Google Tasks Node to Retrieve Tasks**  
   - Node Type: Google Tasks  
   - Name: `Google Tasks`  
   - Operation: `getAll`  
   - Tasklist: Expression referencing `Configuration.tasklistId` (`={{ $('Configuration').item.json.tasklistId }}`)  
   - Additional Fields: Set `showCompleted` to `false`  
   - Credentials: Select your Google Tasks OAuth2 credentials  
   - Connect `Configuration` output to `Google Tasks` input.

4. **Create Date & Time Node to Format Due Date**  
   - Node Type: DateTime  
   - Name: `Date & Time`  
   - Operation: `formatDate`  
   - Date input expression: Convert `Webhook.body.DueDateTimeSeconds` from seconds to datetime:  
     `={{ $('Webhook').item.json.body.DueDateTimeSeconds.toDateTime("s") }}`  
   - Custom format: `yyyy-MM-dd HH:mm:ss`  
   - Connect `Google Tasks` output to `Date & Time` input.

5. **Create Google Calendar Node to Search for Duplicate Events**  
   - Node Type: Google Calendar  
   - Name: `Google Calendar9`  
   - Operation: `getAll`  
   - Limit: 3  
   - Options:  
     - Query: `={{ $('Webhook').item.json.body.TaskName }}`  
     - UpdatedMin: `={{ $now.minus(5, 'minutes') }}`  
     - ShowDeleted: `false`  
   - Calendar: Use expression referencing `Configuration.calendarId`  
   - Credentials: Use your Google Calendar OAuth2 credentials  
   - Connect `Date & Time` output to `Google Calendar9` input.

6. **Create If Node for Duplicate Detection**  
   - Node Type: If  
   - Name: `If1`  
   - Conditions:  
     - Check if `summary` field exists and equals `Webhook.body.TaskName` (case-sensitive)  
   - Connect `Google Calendar9` output to `If1` input.

7. **Create Google Calendar Node to Create Event**  
   - Node Type: Google Calendar  
   - Name: `Google Calendar`  
   - Operation: Create event (implicit by parameters)  
   - Start: `={{ $('Date & Time').item.json.formattedDate }}`  
   - End: Calculate end by adding `defaultMinutes` from configuration:  
     `={{ $('Date & Time').item.json.formattedDate.toDateTime().plus($('Configuration').item.json.defaultMinutes, 'minutes').format("yyyy-MM-dd HH:mm:ss") }}`  
   - Calendar: Use `Configuration.calendarId`  
   - Additional Fields:  
     - Summary: Use task title from Google Tasks node (`={{ $('Google Tasks').item.json.title }}`)  
     - Color: `={{ $('Configuration').item.json.eventColor }}`  
   - Credentials: Google Calendar OAuth2  
   - Connect `If1` false branch (no duplicate) to this node.

8. **Create Google Tasks Node to Delete Original Task (Optional Cleanup)**  
   - Node Type: Google Tasks  
   - Name: `Google Tasks1`  
   - Operation: `delete`  
   - Tasklist ID: Set appropriately (replace hardcoded ID with configuration if needed)  
   - Task ID: `={{ $('Google Tasks').item.json.id }}`  
   - Credentials: Google Tasks OAuth2  
   - Connect output of `Google Calendar` node to this node.

9. **Add Sticky Notes for Documentation**  
   - Add sticky notes describing each step as per the original workflow, including example payload, configuration instructions, and logic explanations.

10. **Activate and Test**  
    - Ensure Google OAuth credentials for Tasks and Calendar nodes are properly created and authorized.  
    - Test by sending POST requests with JSON payload e.g. `{ "TaskName": "Pay invoice", "DueDateTimeSeconds": 1735179600 }` to your webhook URL.  
    - Monitor workflow execution and verify calendar event creation and task deletion.

---

## 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires valid Google OAuth2 credentials for both Tasks and Calendar scopes.            | OAuth2 credential setup in n8n credential manager                                                |
| Replace `tasklistId` placeholder with your actual Google Tasklist ID before activation.               | Google Tasks API documentation or Google Tasks web interface                                    |
| The hardcoded Tasklist ID in the cleanup (delete) node should be parameterized for flexibility.       | Modify `Google Tasks1` node to use `Configuration.tasklistId`                                   |
| Example webhook payload: `{ "TaskName": "Pay invoice", "DueDateTimeSeconds": 1735179600 }`             | See Step 1 – Trigger sticky note                                                                |
| For preventing duplicates, adjust the search window (`updatedMin`) and matching logic as needed.      | Duplicate detection logic explained in Step 4 – Dedupe sticky note                               |
| Google Calendar color indexes range from 1 to 11; verify color availability in your calendar settings. | Google Calendar API color reference                                                             |
| n8n version compatibility: DateTime node uses version 2 with `formatDate` operation; Google Calendar nodes use version 1.1. | Verify node versions if errors occur                                                            |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---