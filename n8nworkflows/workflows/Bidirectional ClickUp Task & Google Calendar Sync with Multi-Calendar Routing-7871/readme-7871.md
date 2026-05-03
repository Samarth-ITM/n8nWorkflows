Bidirectional ClickUp Task & Google Calendar Sync with Multi-Calendar Routing

https://n8nworkflows.xyz/workflows/bidirectional-clickup-task---google-calendar-sync-with-multi-calendar-routing-7871


# Bidirectional ClickUp Task & Google Calendar Sync with Multi-Calendar Routing

### 1. Workflow Overview

This workflow provides a **bidirectional synchronization** between ClickUp tasks and multiple Google Calendars, maintaining a mapping of task-event relationships in a Google Sheet. It supports multi-calendar routing based on task lists and spaces, allowing tasks in different ClickUp spaces or lists to sync with different Google Calendars.

The workflow is logically divided into the following blocks:

- **1.1 Configuration & Initialization:** Defines calendar IDs, Google Sheet ID, and other key parameters.
- **1.2 Trigger & Event Routing:** Listens to ClickUp task events and routes them based on event types and task metadata.
- **1.3 Create Events:** Creates new Google Calendar events corresponding to new ClickUp tasks.
- **1.4 Update Events:** Updates existing Calendar events when ClickUp tasks change, with special handling for all-day events.
- **1.5 Delete Events:** Deletes Calendar events and cleans up Google Sheets mappings when ClickUp tasks are deleted.
- **1.6 Calendar & Sheet Lookup:** Retrieves existing events and mappings to coordinate sync operations.
- **1.7 Conditional Logic & Routing:** Uses Switch and If nodes extensively to route tasks to the correct calendar and handle different statuses, spaces, and event types.

---

### 2. Block-by-Block Analysis

#### 2.1 Configuration & Initialization

- **Overview:**  
  This block sets up all necessary configuration variables such as calendar IDs for different categories, the Google Sheet ID used for mapping, and the ClickUp Team ID. It acts as a centralized configuration hub for the workflow.

- **Nodes Involved:**  
  - Configuration (Set)  
  - Template Notes (Sticky Note)

- **Node Details:**

  - *Configuration*  
    - Type: Set node  
    - Purpose: Stores string variables holding calendar IDs for personal, faculdade, tech, internship calendars, Google Sheet ID, sheet tab name, and ClickUp team ID.  
    - Configuration: Values must be replaced with user-specific IDs (placeholders "REPLACE_WITH_...").  
    - Inputs: None (start node)  
    - Outputs: Configuration data used by downstream nodes via expressions.  
    - Edge cases: Missing or incorrect IDs will cause failures in calendar or sheet operations.

  - *Template Notes*  
    - Type: Sticky Note  
    - Purpose: Provides instructions on setup and usage.  
    - Content includes detailed template instructions, setup steps, and notes on mapping.

---

#### 2.2 Trigger & Event Routing

- **Overview:**  
  Listens for ClickUp task events (created, updated, deleted, due date updated) and routes each event type to the appropriate processing branch.

- **Nodes Involved:**  
  - ClickUp Trigger  
  - Configuration (Set)  
  - Switch3 (Switch node routing event types)  
  - Merge (merges branches for further processing)

- **Node Details:**

  - *ClickUp Trigger*  
    - Type: ClickUp Trigger  
    - Purpose: Webhook that fires on ClickUp task events: taskCreated, taskDeleted, taskUpdated, taskDueDateUpdated.  
    - Configuration: Uses OAuth2 authentication, listens to specified events, linked to a webhook ID.  
    - Output: Emits task event data downstream.

  - *Configuration*  
    - Passes configuration data downstream.

  - *Switch3*  
    - Type: Switch  
    - Purpose: Routes incoming ClickUp events based on event type (taskCreated, taskDeleted, taskUpdated, taskDueDateUpdated, taskStatusUpdated).  
    - Configuration: Matches event string exactly to route.  
    - Outputs: Different branches for each event type.  
    - Edge cases: Unknown event types will not match any branch (no output).

  - *Merge*  
    - Type: Merge (choose branch)  
    - Purpose: Combines outputs from multiple branches for further processing.

---

#### 2.3 Create Events (ClickUp → Google Calendar)

- **Overview:**  
  Handles creation of new Google Calendar events upon new ClickUp tasks. It routes tasks to calendars based on the task list, checks for duplicate events, creates events, and appends mapping to Google Sheets.

- **Nodes Involved:**  
  - Switch2 (routes by task list name to calendar)  
  - Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3 (set calendarId for each list)  
  - Edit Fields4 (passes calendarId downstream)  
  - Google Calendar9 (search recent events to avoid duplicates)  
  - If (checks if matching event exists)  
  - Google Calendar10 (creates event)  
  - Google Sheets (appends mapping)  
  - Edit Fields6 (prepares data for mapping sheet deletion)  
  - Google Sheets1 (deletes mapping rows for cleanup)

- **Node Details:**

  - *Switch2*  
    - Type: Switch  
    - Purpose: Routes tasks based on ClickUp task list name ("Tarefas Pessoais", "Matérias em curso", "Cursos de tecnologia", "Estágio").  
    - Output: Each output corresponds to one calendar category.

  - *Edit Fields* nodes (Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3)  
    - Type: Set nodes  
    - Purpose: Assign the appropriate calendarId from configuration based on the task list.  
    - Output: Passes calendarId for querying and creating events.

  - *Edit Fields4*  
    - Passes calendarId for Google Calendar API nodes.

  - *Google Calendar9*  
    - Type: Google Calendar (getAll)  
    - Operation: Searches recent events (last 5 minutes) with the same summary as the task name to avoid duplicates.  
    - Input: Uses calendarId and task name as query.  
    - Output: List of matching events (empty if none found).

  - *If*  
    - Checks if any matching event was found.  
    - If no match, continues to create event.

  - *Google Calendar10*  
    - Type: Google Calendar (create)  
    - Creates event with task name (summary), description, start/end dates converted from milliseconds, and a fixed color.  
    - Calendar chosen by calendarId.

  - *Google Sheets*  
    - Appends a new row to the mapping sheet with clickupTaskId, calendarEventId, and eventName.  
    - Sheet and tab specified by configuration.

  - *Edit Fields6* and *Google Sheets1*  
    - Used in deletion flows for cleanup (described later).

- **Edge cases:**  
  - Duplicate detection may miss events if task name changes or search window is too narrow.  
  - Google Calendar API limits or auth failures can cause errors.  
  - Google Sheets append failures if sheet or tab not found.

---

#### 2.4 Update Events (Keep Calendar in sync with ClickUp changes)

- **Overview:**  
  Updates Google Calendar events when ClickUp tasks are updated or their due dates change. Handles both timed and all-day events with conditional branches and compares existing vs new data to avoid unnecessary updates.

- **Nodes Involved:**  
  - Google Sheets2 (lookup mapping by clickupTaskId)  
  - Google Calendar11..14 (fetch event from calendar)  
  - Switch (routes tasks by status)  
  - Multiple if_allDayEvent* nodes (check if event is all-day)  
  - Multiple If17..If28 nodes (compare fields like colorId, name, description, start/end to decide if update needed)  
  - Google Calendar15..25 (update event with new fields)  
  - Google Sheets3 (updates mapping with new summary if changed)

- **Node Details:**

  - *Google Sheets2*  
    - Filters the mapping sheet by clickupTaskId to get calendarEventId for the task.

  - *Google Calendar11..14*  
    - Retrieves the existing calendar event by event ID and calendar ID (per calendar category).  
    - Configured to continue on error to handle missing events gracefully.

  - *Switch*  
    - Routes tasks based on ClickUp task status (e.g., a fazer, em progresso, bloqueado, cancelado, concluído, etc.).  
    - Each output branch connects to an all-day event check node.

  - *if_allDayEvent, if_allDayEvent1, ... if_allDayEvent5*  
    - If nodes that check if the event is an all-day event by inspecting if `end.date` field is non-empty.  
    - Branches into separate flows for all-day vs timed event updates.

  - *If17..If28*  
    - Series of If nodes that compare task and event fields such as colorId, summary, description, start and end times/dates to detect if an update is necessary.  
    - Conditions use exact or date-only comparisons depending on all-day status.

  - *Google Calendar15..25*  
    - Update operations on Google Calendar events with new summary, description, start/end dates, color, and all-day flag if applicable.

  - *Google Sheets3*  
    - Updates the mapping row with the latest event name if changed.

- **Edge cases:**  
  - Missing mapping rows lead to update failure; fallback to create is recommended but not implemented here.  
  - Timezone handling must be verified to avoid date mismatches.  
  - Recurring events are not explicitly handled.  
  - Google Calendar API rate limiting or network errors can cause update failures.

---

#### 2.5 Delete Events (Remove event and clean mapping)

- **Overview:**  
  Deletes Google Calendar events and removes their mapping rows in the Google Sheet when ClickUp tasks are deleted.

- **Nodes Involved:**  
  - Google Sheets2 (lookup mapping by clickupTaskId)  
  - Google Calendar11..14 (delete events in all calendars)  
  - Edit Fields6 (prepares fields for deletion)  
  - Google Sheets1 (delete the mapping row)

- **Node Details:**

  - *Google Sheets2*  
    - Retrieves mapping row(s) for the task.

  - *Google Calendar11..14*  
    - Delete operations on calendar events in each calendar category.  
    - Configured to continue on error to avoid breaking workflow if event not found.

  - *Edit Fields6*  
    - Prepares data for deleting the mapping row from the Google Sheet.

  - *Google Sheets1*  
    - Deletes the mapping row from the sheet by row number.

- **Edge cases:**  
  - Event may already be deleted in Google Calendar, handled by continue on error.  
  - Mapping row may be missing; deletion will fail silently.  
  - If tasks move lanes/lists, care should be taken to update mappings accordingly.

---

#### 2.6 Calendar & Sheet Lookup

- **Overview:**  
  Performs searches and lookups in Google Calendar and Google Sheets to find existing events or mappings for synchronization decisions.

- **Nodes Involved:**  
  - Google Calendar9 (search events by summary)  
  - Google Sheets2, Google Sheets4..7 (lookup rows by clickupTaskId)  
  - Google Calendar1, 6, 7, 8 (get events by eventId)

- **Node Details:**

  - *Google Calendar9*  
    - Searches recent events on calendar filtering by summary to avoid duplicate creation.

  - *Google Sheets2, 4, 5, 6, 7*  
    - Filter mapping sheet rows by clickupTaskId to find associated calendar events.  
    - Each node corresponds to a different ClickUp space or list (as routed by Switch1).

  - *Google Calendar1, 6, 7, 8*  
    - Get single events by eventId from specific calendars, used to verify event details before updates or deletes.

- **Edge cases:**  
  - Missing mapping rows cause failures or fallback needs.  
  - Multiple rows matching the same clickupTaskId could cause inconsistencies.  
  - Google API errors or quota issues.

---

#### 2.7 Conditional Logic & Routing

- **Overview:**  
  Extensive use of Switch and If nodes to route tasks between calendars based on ClickUp space or list, task status, and to conditionally process events as all-day or timed.

- **Nodes Involved:**  
  - Switch1 (routes by ClickUp space.id)  
  - Switch2 (routes by ClickUp list.name)  
  - Switch (routes by task status)  
  - If nodes (if_allDayEvent*, If17..If28, If1, If)

- **Node Details:**

  - *Switch1*  
    - Routes tasks based on ClickUp space IDs corresponding to personal, faculdade, tech, internship spaces.

  - *Switch2*  
    - Routes tasks based on task list names for calendar ID assignment.

  - *Switch*  
    - Routes based on task status (a fazer, em progresso, bloqueado, cancelado, concluído, etc.) for update logic.

  - *If nodes*  
    - Evaluate task and event field equality to minimize unnecessary updates.  
    - Check if events are all-day or timed to branch processing.

- **Edge cases:**  
  - Hardcoded space and list IDs require updates if ClickUp structure changes.  
  - Case sensitivity is carefully managed but can still cause mismatches if input deviates.  
  - Complex conditional chains increase workflow complexity and maintenance effort.

---

### 3. Summary Table

| Node Name          | Node Type              | Functional Role                            | Input Node(s)                               | Output Node(s)                              | Sticky Note                                                                                           |
|--------------------|------------------------|------------------------------------------|---------------------------------------------|---------------------------------------------|----------------------------------------------------------------------------------------------------|
| Configuration      | Set                    | Stores configuration variables           | ClickUp Trigger                             | Switch3                                     |                                                                                                    |
| Template Notes     | Sticky Note            | Instructions and setup notes              | None                                        | None                                        | Template instructions for setup and usage.                                                         |
| ClickUp Trigger    | ClickUp Trigger        | Triggers on ClickUp task events           | None                                        | Configuration                               |                                                                                                    |
| Switch3            | Switch                 | Routes by ClickUp event type               | Configuration                               | ClickUp1, Google Sheets2, ClickUp, ClickUp |                                                                                                    |
| ClickUp1           | ClickUp                | Gets detailed task info                     | Switch3                                     | If1                                         |                                                                                                    |
| If1                | If                     | Checks if task has due_date/start_date     | ClickUp1                                    | Switch2                                     |                                                                                                    |
| Switch2            | Switch                 | Routes by ClickUp list name                | If1                                         | Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3 |                                                                                                    |
| Edit Fields        | Set                    | Sets calendarId for "Tarefas Pessoais"    | Switch2                                     | Edit Fields4                                |                                                                                                    |
| Edit Fields1       | Set                    | Sets calendarId for "Matérias em curso"   | Switch2                                     | Edit Fields4                                |                                                                                                    |
| Edit Fields2       | Set                    | Sets calendarId for "Cursos de tecnologia"| Switch2                                     | Edit Fields4                                |                                                                                                    |
| Edit Fields3       | Set                    | Sets calendarId for "Estágio"              | Switch2                                     | Edit Fields4                                |                                                                                                    |
| Edit Fields4       | Set                    | Passes calendarId downstream                | Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3 | Google Calendar9                           |                                                                                                    |
| Google Calendar9   | Google Calendar (getAll)| Searches recent events for duplicates     | Edit Fields4                                | If                                          |                                                                                                    |
| If                 | If                     | Checks if duplicate event exists           | Google Calendar9                            | Google Calendar10                           |                                                                                                    |
| Google Calendar10  | Google Calendar (create)| Creates new event in Google Calendar      | If                                          | Google Sheets                               |                                                                                                    |
| Google Sheets      | Google Sheets           | Appends mapping row                        | Google Calendar10                           |                                             |                                                                                                    |
| Google Sheets2     | Google Sheets           | Looks up mapping by clickupTaskId          | Switch3                                     | Google Calendar11..14                       |                                                                                                    |
| Google Calendar11  | Google Calendar (delete)| Deletes event from personal calendar       | Google Sheets2                              | Edit Fields6                                |                                                                                                    |
| Google Calendar12  | Google Calendar (delete)| Deletes event from faculdade calendar      | Google Sheets2                              | Edit Fields6                                |                                                                                                    |
| Google Calendar13  | Google Calendar (delete)| Deletes event from tech calendar           | Google Sheets2                              | Edit Fields6                                |                                                                                                    |
| Google Calendar14  | Google Calendar (delete)| Deletes event from internship calendar     | Google Sheets2                              | Edit Fields6                                |                                                                                                    |
| Edit Fields6       | Set                    | Prepares deletion parameters               | Google Calendar11..14                       | Google Sheets1                              |                                                                                                    |
| Google Sheets1     | Google Sheets           | Deletes mapping row                         | Edit Fields6                                |                                             |                                                                                                    |
| Google Sheets3     | Google Sheets           | Updates mapping row                         | Google Calendar15..25                       |                                             |                                                                                                    |
| Switch              | Switch                 | Routes by task status                       | Google Calendar1, 6, 7, 8                    | if_allDayEvent* nodes                       |                                                                                                    |
| if_allDayEvent* nodes | If                   | Checks if event is all-day                  | Switch                                       | If17..If28 nodes                            |                                                                                                    |
| If17..If28          | If                     | Compares task and event data to decide update | if_allDayEvent* nodes                      | Google Calendar15..25                       |                                                                                                    |
| Google Calendar15..25 | Google Calendar (update)| Updates Google Calendar events             | If17..If28                                  | Google Sheets3                              |                                                                                                    |
| Switch1             | Switch                 | Routes by ClickUp space ID                  | ClickUp                                      | Google Sheets4..7                           |                                                                                                    |
| Google Sheets4..7   | Google Sheets           | Lookup mapping rows by space                | Switch1                                      | Google Calendar1, 6, 7, 8                   |                                                                                                    |
| Google Calendar1, 6, 7, 8 | Google Calendar (get)| Gets event for update validation          | Google Sheets4..7                           | Switch                                      |                                                                                                    |
| Merge               | Merge                  | Combines branches                           | Switch, Google Calendar1, 6, 7, 8            | ClickUp1                                    |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Configuration Node**  
   - Type: Set  
   - Set string variables:  
     - `calendarId_personal` (Google Calendar ID for personal tasks)  
     - `calendarId_faculdade` (Google Calendar ID for faculdade)  
     - `calendarId_tech` (Google Calendar ID for tech courses)  
     - `calendarId_internship` (Google Calendar ID for internship)  
     - `sheetId` (Google Sheet ID for task-event mapping)  
     - `sheetTabName` (Tab name, e.g., "Sheet1")  
     - `clickupTeamId` (Your ClickUp team ID)  

2. **Add ClickUp Trigger Node**  
   - Configure with OAuth2 credentials for ClickUp.  
   - Set to trigger on events: taskCreated, taskDeleted, taskUpdated, taskDueDateUpdated.  
   - Connect output to Configuration node.

3. **Add Switch3 Node**  
   - Route incoming ClickUp event by `$json.event` with cases:  
     - taskCreated  
     - taskDeleted  
     - taskUpdated  
     - taskDueDateUpdated  
     - taskStatusUpdated  
   - Connect Configuration node output to Switch3 input.

4. **Create Branch for `taskCreated`**  
   - Add Switch2 node to route tasks based on `$json.list.name` to "Tarefas Pessoais", "Matérias em curso", "Cursos de tecnologia", "Estágio".  
   - For each output, add corresponding Set node (Edit Fields, Edit Fields1, Edit Fields2, Edit Fields3) to set `calendarId` from configuration variables accordingly.  
   - Connect all Edit Fields* nodes to a Set node (Edit Fields4) to pass the calendarId downstream.

5. **Add Google Calendar9 Node**  
   - Operation: getAll  
   - Calendar: Use calendarId from Edit Fields4  
   - Query: Task name from ClickUp (summary)  
   - UpdatedMin: Current time minus 5 minutes  
   - ShowDeleted: false  

6. **Add If Node**  
   - Condition: Check if any events were returned by Google Calendar9 (to detect duplicates).  
   - If no event found, continue to create event.

7. **Add Google Calendar10 Node**  
   - Operation: create  
   - Calendar: calendarId from Edit Fields4  
   - Set summary, description, start, end from ClickUp task data (convert timestamps to formatted date strings).  
   - Set color (e.g., 8).  

8. **Add Google Sheets Node**  
   - Operation: append  
   - Document ID: from configuration sheetId  
   - Sheet Name: from configuration sheetTabName  
   - Columns: `clickupTaskId` (ClickUp task id), `calendarEventId` (from created event), `eventName` (summary)  

9. **Create Branch for `taskDeleted`**  
   - Add Google Sheets2 node to filter mapping rows by clickupTaskId from event data.  
   - Connect to Google Calendar11..14 nodes (delete operations) each targeting one calendar. Set to continue on error.  
   - Connect those to Edit Fields6 node (set parameters for sheet deletion).  
   - Connect Edit Fields6 to Google Sheets1 node (delete mapping row by row number).

10. **Create Branch for `taskUpdated` and `taskDueDateUpdated`**  
    - Add Google Sheets2 node to lookup mapping row by clickupTaskId.  
    - Connect to Google Calendar11..14 nodes (get event) for verification.  
    - Add Switch node to route by task status (`status.status` from ClickUp) for update logic.  
    - Connect to if_allDayEvent* If nodes to branch by all-day vs timed events.  
    - Add If17..If28 nodes to compare current task and calendar event fields to decide if update is needed.  
    - Connect to Google Calendar15..25 nodes to update events with new data.  
    - Connect to Google Sheets3 node to update mapping row with new summary if changed.

11. **Create Branch for `taskStatusUpdated`**  
    - Connect to ClickUp node to fetch full task details by id.  
    - Connect to If1 node checking if task has due_date or start_date.  
    - Connect to Switch2 and continue as in create flow (to handle status-related updates).

12. **Create Supporting Nodes**  
    - Switch1 node routes ClickUp tasks by space.id to Google Sheets4..7 nodes for mapping lookup.  
    - Google Calendar1,6,7,8 nodes get events for validation in update flows.

13. **Set Credentials**  
    - Assign OAuth2 credentials for ClickUp nodes.  
    - Assign Google API credentials with access to Google Calendar and Sheets for all Google nodes.

14. **Validation and Testing**  
    - Ensure Google Sheet exists with the specified tab and columns (`clickupTaskId`, `calendarEventId`, `eventName`).  
    - Test create, update, and delete flows by creating/modifying/deleting ClickUp tasks.  
    - Monitor for errors such as missing calendar IDs, auth failures, or data format issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Mapping sheet stores `clickupTaskId`, `calendarEventId`, `eventName` to maintain synchronization references.                   | Core synchronization design detail.                                                                         |
| No secrets are included in the JSON; credentials are assigned after import for security.                                        | Security best practice for workflow sharing.                                                                 |
| For stricter duplicate detection, expand Google Calendar search window or include description comparison.                      | Duplicate event prevention tip.                                                                               |
| Timezone handling is critical; verify your OAuth2 credentials handle timezone conversions correctly, or convert dates manually. | Avoids date/time mismatches between ClickUp and Google Calendar.                                            |
| Soft delete alternative: Instead of deleting events, consider changing color or adding "Cancelled" suffix for auditability.    | Alternative deletion approach.                                                                                |
| Recurring events are not handled explicitly; consider adding support if needed.                                                 | Limitation of current workflow.                                                                               |
| This template was originally inspired by [n8n blog and community examples](https://n8n.io/blog/).                               | For further learning and inspiration.                                                                         |

---

This detailed reference document covers the entire workflow structure, logic, nodes, and configuration steps required to understand, reproduce, and maintain the bidirectional sync between ClickUp tasks and multiple Google Calendars with multi-calendar routing.