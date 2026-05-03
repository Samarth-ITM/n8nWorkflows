Automatically create daily list of todos in Todoist

https://n8nworkflows.xyz/workflows/automatically-create-daily-list-of-todos-in-todoist-2149


# Automatically create daily list of todos in Todoist

### 1. Workflow Overview

This workflow automates the daily management of Todoist tasks by performing two scheduled flows every morning:

- **Flow 1 (5:00 AM):** Cleans the Inbox project by deleting any uncompleted tasks labeled as `daily`.
- **Flow 2 (5:10 AM):** Copies template tasks from a designated "template" project into the Inbox project, setting due dates and repeating days according to task descriptions.

The workflow is logically divided into two blocks:

- **1.1 Morning Cleanup:** Triggered at 5:00 AM, it fetches all tasks from Inbox, filters those labeled `daily`, and deletes them to prepare a fresh slate.
  
- **1.2 Template Task Copying:** Triggered at 5:10 AM, it fetches all tasks from the `template` project, parses task descriptions to extract scheduling info (days of week, due time), filters tasks relevant for the current day, and creates new tasks in Inbox with appropriate due dates and labels.

---

### 2. Block-by-Block Analysis

#### 2.1 Morning Cleanup Block

**Overview:**  
This block triggers daily at 5:00 AM to delete all uncompleted tasks in the Inbox project that have the label `daily`. This cleans up previous tasks to avoid clutter.

**Nodes Involved:**  
- Every day at 5am (Schedule Trigger)  
- Get all tasks from Inbox (Todoist)  
- If list not empty (If)  
- if it has daily label (If)  
- Delete task (Todoist)  
- Sticky Note (explains the label usage)

**Node Details:**

- **Every day at 5am**  
  - Type: Schedule Trigger  
  - Configured to trigger daily at 5:00 AM (note: actually set to 5:10 AM in parameters, see below)  
  - Output: Initiates the cleanup sequence  
  - Potential errors: Trigger misconfiguration, timezone issues

- **Get all tasks from Inbox**  
  - Type: Todoist node (getAll operation)  
  - Configured to retrieve all tasks from Inbox project (ID: 938017196)  
  - Returns all tasks regardless of completion status  
  - Credentials: Todoist account  
  - Edge cases: API auth failure, empty inbox, rate limits

- **If list not empty**  
  - Type: If  
  - Checks if there are any tasks (by verifying the `id` field is not empty)  
  - Avoids errors downstream if no tasks found

- **if it has daily label**  
  - Type: If  
  - Checks if the task's labels array includes `"daily"`  
  - Filters only tasks that were created by the template flow (which adds this label)  
  - Edge case: Missing labels field or malformed data

- **Delete task**  
  - Type: Todoist node (delete operation)  
  - Deletes the task with the given `id`  
  - Credentials: Todoist account  
  - Edge cases: Task already deleted, API failure, network issues

- **Sticky Note**  
  - Explains that every new task created by the other flow has the `daily` label, which is the criterion for deletion here

---

#### 2.2 Template Task Copying Block

**Overview:**  
This block runs daily at 5:10 AM and copies tasks from a designated `template` project into the Inbox. It parses the descriptions of template tasks to determine on which days they should be created and their due times. This allows repeating tasks to be automatically managed.

**Nodes Involved:**  
- Every day at 5:10am (Schedule Trigger)  
- Get all tasks from template project (Todoist)  
- Parse task details (Code)  
- Keep tasks that match today (Filter)  
- Create new task in Inbox (Todoist)  
- Sticky Note2 (explains description parsing and due date logic)  
- Sticky Note3 (setup instructions)  
- Sticky Note (label usage note, shared with first block)

**Node Details:**

- **Every day at 5:10am**  
  - Type: Schedule Trigger  
  - Configured to trigger daily at 5:10 AM  
  - Starts the task copying flow  
  - Edge cases: Trigger misconfiguration, timezone mismatches

- **Get all tasks from template project**  
  - Type: Todoist node (getAll operation)  
  - Fetches all tasks from the template project (ID: 2299363018)  
  - Returns all tasks unfiltered  
  - Credentials: Todoist account  
  - Edge cases: API auth failure, empty template list

- **Parse task details**  
  - Type: Code node (JavaScript)  
  - Runs once per item (task)  
  - Extracts `days` and `due` info from the task's description field, which follows a semi-colon-separated format like `days:mon,tues; due:8am`  
  - Parses due time strings (e.g., "8am", "8:30pm") into UTC DateTime objects using a regex and Luxon DateTime (assumed)  
  - Outputs an object with fields for content, description, days, and due date/time  
  - Edge cases: Invalid time format throws error, missing days or due fields, malformed description string

- **Keep tasks that match today**  
  - Type: Filter node  
  - Checks if the current day of the week (e.g., "mon", "tues") is included in the `days` property extracted from the task description  
  - Only tasks scheduled for the current day proceed  
  - Edge cases: Empty or missing `days` field, timezone differences affecting day calculation

- **Create new task in Inbox**  
  - Type: Todoist node (create operation)  
  - Creates a new task in Inbox (ID: 938017196)  
  - Uses the content, description, and parsed due date/time from previous node  
  - Adds the label `daily`  
  - Credentials: Todoist account  
  - Edge cases: API rate limits, invalid due date format, network errors

- **Sticky Note2**  
  - Explains how due dates and days are extracted from task descriptions with an example

- **Sticky Note3**  
  - Setup instructions for credentials, template project, and node configuration

- **Sticky Note**  
  - Reminder about the `daily` label used across flows

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                                   | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                                  |
|--------------------------------|-----------------------|--------------------------------------------------|-----------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note3                   | Sticky Note           | Setup instructions                               | -                           | -                                | 1. Add Todoist creds 2. Create a `template` list to copy from in Todoist. Add days and due times on each task as necessary. 3. Set the projects to copy from and to write to in each Todoist node |
| Every day at 5am              | Schedule Trigger      | Trigger flow to cleanup Inbox at 5:00 AM         | -                           | Get all tasks from Inbox          |                                                                                                              |
| Get all tasks from Inbox       | Todoist               | Get all tasks from Inbox project                   | Every day at 5am            | If list not empty                 |                                                                                                              |
| If list not empty              | If                    | Check if Inbox contains tasks                       | Get all tasks from Inbox    | if it has daily label             |                                                                                                              |
| if it has daily label          | If                    | Filter tasks that have `daily` label                | If list not empty           | Delete task                      |                                                                                                              |
| Delete task                   | Todoist               | Delete task with `daily` label                      | if it has daily label       | -                                |                                                                                                              |
| Sticky Note                   | Sticky Note           | Explanation of `daily` label usage                  | -                           | -                                | Every new task has `daily` label that gets deleted in the other flow                                          |
| Every day at 5:10am            | Schedule Trigger      | Trigger flow to copy template tasks at 5:10 AM    | -                           | Get all tasks from template project |                                                                                                              |
| Get all tasks from template project | Todoist          | Fetch all tasks from the template project          | Every day at 5:10am         | Parse task details               |                                                                                                              |
| Parse task details             | Code                  | Parse days and due time from task description      | Get all tasks from template project | Keep tasks that match today     |                                                                                                              |
| Keep tasks that match today    | Filter                | Keep only tasks scheduled for the current day      | Parse task details          | Create new task in Inbox          |                                                                                                              |
| Create new task in Inbox       | Todoist               | Create task in Inbox with due date and label       | Keep tasks that match today | -                                |                                                                                                              |
| Sticky Note2                  | Sticky Note           | Explains description format for due dates and days | -                           | -                                | This adds due dates to tasks from description.. For example in the description of a task `days:mon,tues; due:8am` So that it will create a task every Monday and Tuesday that's due at 8am ⏰ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credential**  
   - Add Todoist API credentials in n8n (via OAuth2 or API token).

2. **Create Schedule Trigger Node "Every day at 5am"**  
   - Type: Schedule Trigger  
   - Trigger: Daily at 5:00 AM (Note: original node had 5:10 AM, correct to 5:00 AM for cleanup)  
   - Position accordingly.

3. **Create Todoist Node "Get all tasks from Inbox"**  
   - Operation: getAll  
   - Project ID: Inbox project ID (e.g., 938017196)  
   - Return All: true  
   - Connect input from "Every day at 5am".

4. **Create If Node "If list not empty"**  
   - Condition: Check if `$json["id"]` is not empty (string isNotEmpty)  
   - Input: Connect from "Get all tasks from Inbox".

5. **Create If Node "if it has daily label"**  
   - Condition: Boolean check if `($json["labels"] || []).includes('daily')` equals true  
   - Input: Connect from "If list not empty".

6. **Create Todoist Node "Delete task"**  
   - Operation: delete  
   - Task ID: Set to `{{$json["id"]}}`  
   - Input: Connect from "if it has daily label".

7. **Create Schedule Trigger Node "Every day at 5:10am"**  
   - Type: Schedule Trigger  
   - Trigger: Daily at 5:10 AM  
   - Position accordingly (separate branch).

8. **Create Todoist Node "Get all tasks from template project"**  
   - Operation: getAll  
   - Project ID: Template project ID (e.g., 2299363018)  
   - Return All: true  
   - Credential: Use Todoist credentials  
   - Input: Connect from "Every day at 5:10am".

9. **Create Code Node "Parse task details"**  
   - Mode: Run once for each item  
   - JavaScript code to:  
     - Extract `description` and `content` from input task  
     - Split `description` by `;` and parse keys like `days` and `due`  
     - Parse `due` string time (format like `8am`, `8:30pm`) into UTC datetime (using regex and DateTime library)  
     - Return object with `content`, `description`, `days`, and `due` as DateTime  
   - Input: Connect from "Get all tasks from template project".

10. **Create Filter Node "Keep tasks that match today"**  
    - Condition: Check if `days` string contains today's day abbreviation, e.g. `['sun', 'mon', 'tues', 'wed', 'thurs', 'fri', 'sat'][new Date().getDay()]`  
    - Input: Connect from "Parse task details".

11. **Create Todoist Node "Create new task in Inbox"**  
    - Operation: create  
    - Project ID: Inbox project ID (938017196)  
    - Content: `{{$json.content}}`  
    - Description: `{{$json.description}}`  
    - Due DateTime: `{{$json.due}}` (ensure ISO format)  
    - Labels: Add `daily` label  
    - Input: Connect from "Keep tasks that match today".

12. **Add Sticky Notes** (optional for clarity)  
    - Setup instructions for Todoist creds, template list, project IDs  
    - Explain the description format for repeating tasks (`days:mon,tues; due:8am`)  
    - Note about the `daily` label usage for cleanup.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                  |
|------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow relies on the `daily` label to identify tasks for deletion and recreation.        | Label usage explanation in Sticky Note          |
| Template task descriptions must follow the format: `days:mon,tues; due:8pm` for scheduling.    | See Sticky Note2 for example and parsing details|
| Due time strings support formats like `8am`, `8:30pm` and are parsed into UTC.                  | Implemented in Code Node `Parse task details`   |
| Todoist project IDs must be set correctly in the corresponding nodes for Inbox and Template projects. | Todoist node configuration instructions         |
| The workflow triggers run on n8n server timezone; adjust schedule triggers accordingly.         | Potential timezone issues to consider            |

---

This documentation provides a thorough understanding and reproduction guide for the "Automatically create daily list of todos in Todoist" workflow, enabling effective use, replication, and troubleshooting.