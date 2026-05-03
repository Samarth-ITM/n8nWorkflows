Weekly reminder on your notion tasks with a deadline

https://n8nworkflows.xyz/workflows/weekly-reminder-on-your-notion-tasks-with-a-deadline-2409


# Weekly reminder on your notion tasks with a deadline

### 1. Workflow Overview

This n8n workflow automates a **weekly reminder system for Notion tasks with deadlines**, targeted at users who manage tasks in Notion but tend to lose track of those with important due dates. The workflow fetches all active tasks (excluding closed ones) from a specified Notion database, categorizes them based on their deadline status (overdue or upcoming), generates an HTML email summarizing these tasks with styling, and sends this email weekly along with a push notification via Pushover.

**Logical Blocks:**

- **1.1 Trigger and Initialization:** Scheduled trigger and manual trigger for testing initiate the workflow and set workflow-wide variables.
- **1.2 Data Fetching and Filtering:** Retrieve tasks from Notion, filter out tasks without deadlines, and sort by closest deadline.
- **1.3 Task Categorization:** Separate tasks into overdue and upcoming groups based on the current date.
- **1.4 HTML Generation:** Create individual HTML snippets per task and aggregate these into group-specific HTML blocks, then compile a full styled HTML email.
- **1.5 Notification Dispatch:** Send the generated email and push notification to the user.
- **1.6 Supportive and Configuration Notes:** Sticky notes provide setup instructions, filtering guidance, and customization tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

**Overview:**  
This block starts the workflow either on a scheduled basis (every Monday at 9 AM) or manually for testing. It also sets essential workflow variables such as email, Notion database URL, logo path, and Pushover user key.

**Nodes involved:**  
- Schedule Trigger  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Set Workflow vars  

**Node Details:**

- **Schedule Trigger**  
  - Type: ScheduleTrigger  
  - Configured for weekly execution every Monday at 9:00 AM.  
  - Input: none (time-based trigger)  
  - Output: triggers to Set Workflow vars node.  
  - Edge cases: time zone mismatches or scheduling not activated.

- **When clicking ‘Test workflow’**  
  - Type: ManualTrigger  
  - Allows manual execution for testing purposes.  
  - Input: manual user action  
  - Output: triggers Set Workflow vars node.  
  - Edge cases: none significant, manual trigger.

- **Set Workflow vars**  
  - Type: Set  
  - Assigns key variables: `logo_path`, `pushover_user_key`, `notion_database_url`, `your_email`.  
  - These must be filled appropriately before running.  
  - Input: trigger from scheduler or manual trigger  
  - Output: feeds Notion node.  
  - Edge cases: missing or incorrect variable values will cause downstream failures.

---

#### 1.2 Data Fetching and Filtering

**Overview:**  
Fetches pages (tasks) from the specified Notion database, filters tasks that have a deadline property set, and sorts them by the closest deadline date.

**Nodes involved:**  
- Notion  
- Filter for deadline  
- Sort by closest deadline  

**Node Details:**

- **Notion**  
  - Type: Notion (databasePage, getAll)  
  - Fetches all pages from the configured database except filtered out ones (e.g., closed tasks filtered via Notion API filters).  
  - Uses Notion API credentials.  
  - Input: workflow vars from Set Workflow vars  
  - Output: all tasks data to Filter for deadline node.  
  - Edge cases: authentication errors, API rate limits, database ID errors.

- **Filter for deadline**  
  - Type: Filter  
  - Condition: keeps only tasks where `property_deadline.start` exists (i.e., tasks having a deadline).  
  - Input: tasks list from Notion node  
  - Output: filtered list to Sort by closest deadline  
  - Edge cases: tasks with missing or malformed deadline property are excluded.

- **Sort by closest deadline**  
  - Type: Sort  
  - Sorts filtered tasks ascending by `property_deadline.start` (earliest deadline first).  
  - Input: filtered tasks  
  - Output: sorted tasks to HTML for Task and Merge node.  
  - Edge cases: improper date formats could cause sorting issues.

---

#### 1.3 Task Categorization

**Overview:**  
Separates tasks into two groups: those with overdue deadlines and those with upcoming deadlines.

**Nodes involved:**  
- Merge  
- If deadline is overdue  
- Aggregate overdue tasks  
- Aggregate due to tasks  

**Node Details:**

- **Merge**  
  - Type: Merge (combine by position)  
  - Combines two outputs from Sort by closest deadline node (one directly, one after HTML for Task) to feed into the If node.  
  - Input: HTML for Task and Sort by closest deadline  
  - Output: merged data to If deadline is overdue node.

- **If deadline is overdue**  
  - Type: If  
  - Condition checks if the task’s deadline is before or equal to current date/time (`$now`).  
  - Input: merged tasks  
  - Output: routes to Aggregate overdue tasks (if true) or Aggregate due to tasks (if false).  
  - Edge cases: timezone mismatches, null deadlines.

- **Aggregate overdue tasks**  
  - Type: Aggregate  
  - Aggregates all tasks flagged as overdue into a single array under `overdue` field.  
  - Input: tasks from If node (true branch)  
  - Output: to HTML overdue List node.

- **Aggregate due to tasks**  
  - Type: Aggregate  
  - Aggregates all tasks flagged as upcoming (due to) into a single array under `due_to` field.  
  - Input: tasks from If node (false branch)  
  - Output: to HTML due to List node.

---

#### 1.4 HTML Generation

**Overview:**  
Generates HTML snippets for each task, then aggregates these snippets into two groups (overdue and due to), and finally compiles a full HTML email template with embedded styling and branding.

**Nodes involved:**  
- HTML for Task  
- HTML overdue List  
- HTML due to List  
- Merge groups  
- Aggregate  
- HTML  

**Node Details:**

- **HTML for Task**  
  - Type: HTML  
  - Generates an HTML block per task showing name (linked), deadline, priority, status, and tags.  
  - Uses expressions to format dates and access properties.  
  - Input: sorted tasks  
  - Output: each task’s HTML to Merge node.  
  - Edge cases: missing properties may cause blank fields or errors.

- **HTML overdue List**  
  - Type: HTML  
  - Creates a header and lists all aggregated overdue tasks’ HTML snippets.  
  - Shows “No overdue tasks. Great!” if none.  
  - Input: aggregate overdue tasks output  
  - Output: to Merge groups node.

- **HTML due to List**  
  - Type: HTML  
  - Creates a header and lists all aggregated due tasks’ HTML snippets.  
  - Input: aggregate due to tasks output  
  - Output: to Merge groups node.

- **Merge groups**  
  - Type: Merge (default mode)  
  - Combines overdue and due to HTML blocks into one dataset.  
  - Input: from HTML overdue List and HTML due to List  
  - Output: to Aggregate node.

- **Aggregate**  
  - Type: Aggregate  
  - Aggregates the merged group HTML blocks into one field `html_groups`.  
  - Input: from Merge groups  
  - Output: complete HTML groups to final HTML node.

- **HTML**  
  - Type: HTML  
  - Generates the final styled email template including logo, title, link to Notion database, and inserts the aggregated HTML groups for tasks.  
  - Uses inline CSS styling, expressions for logo path, Notion URL, and task groups.  
  - Input: aggregated HTML groups  
  - Output: HTML content to Send Email node.

---

#### 1.5 Notification Dispatch

**Overview:**  
Sends the compiled HTML email to the user’s email address and a push notification to their phone via Pushover.

**Nodes involved:**  
- Send Email  
- Pushover  

**Node Details:**

- **Send Email**  
  - Type: EmailSend  
  - Sends email with subject "Weekly Update about Notion Tasks" and the generated HTML content as the body.  
  - From address is fixed (`n8n@unitize.de`).  
  - Recipient email is fetched from workflow variables.  
  - SMTP Credentials required and should be configured.  
  - Input: HTML email content  
  - Output: triggers Pushover node.  
  - Edge cases: SMTP auth failures, email delivery issues.

- **Pushover**  
  - Type: Pushover  
  - Sends a push notification with a fixed message alerting about the weekly update.  
  - Uses user key from workflow variables.  
  - Input: trigger from Send Email  
  - Output: none (end node)  
  - Edge cases: invalid user key, API failures.

---

#### 1.6 Supportive and Configuration Notes (Sticky Notes)

**Overview:**  
Sticky notes provide users with setup instructions, filtering advice, scheduling info, and customization tips.

**Sticky Notes and Context:**

- Schedule info: "Current schedule is every monday at 9 am."
- Fetch/filter/sort tasks: "Currently tasks are filtered by having a deadline and sorted by this."
- HTML template per task: "Generate a template for each task. It displays the headline and some properties."
- Task grouping explanation.
- HTML email styling advice.
- Email and push notification sending setup instructions.
- Dependency notes: need access to Notion and Pushover accounts.
- Workflow variables setup reminder.
- Notion filtering advice to exclude closed tasks by adding status filters (e.g., Status != Closed).

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                    | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                   |
|-------------------------|---------------------|--------------------------------------------------|-----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | ScheduleTrigger     | Starts workflow every Monday at 9 AM             | -                           | Set Workflow vars         | Current schedule is every monday at 9 am.                                                                     |
| When clicking ‘Test workflow’ | ManualTrigger       | Manual trigger for testing the workflow           | -                           | Set Workflow vars         |                                                                                                               |
| Set Workflow vars       | Set                 | Defines workflow variables (email, URLs, keys)   | Schedule Trigger, Manual Trigger | Notion                   | Adjust this node to your needs!                                                                                |
| Notion                 | Notion              | Fetch tasks from Notion database                   | Set Workflow vars            | Filter for deadline       | Adjustment needed: add filters to exclude closed tasks (e.g., Status != Closed).                               |
| Filter for deadline      | Filter              | Filters tasks having a deadline                    | Notion                      | Sort by closest deadline  | Currently tasks are filtered by having a deadline and sorted by this                                          |
| Sort by closest deadline | Sort                | Sorts tasks ascending by deadline date            | Filter for deadline          | HTML for Task, Merge      |                                                                                                               |
| HTML for Task           | HTML                | Generates HTML snippet per task                    | Sort by closest deadline     | Merge                     | Generate a template for each task showing headline and key properties                                         |
| Merge                   | Merge               | Combines task data streams                         | Sort by closest deadline, HTML for Task | If deadline is overdue    |                                                                                                               |
| If deadline is overdue  | If                  | Checks if task deadline is overdue or upcoming    | Merge                       | Aggregate overdue tasks, Aggregate due to tasks | Create groups of tasks to "overdue" and "due to"                                                              |
| Aggregate overdue tasks | Aggregate           | Aggregates overdue tasks into one array           | If deadline is overdue (true) | HTML overdue List         |                                                                                                               |
| Aggregate due to tasks  | Aggregate           | Aggregates upcoming tasks into one array          | If deadline is overdue (false) | HTML due to List          |                                                                                                               |
| HTML overdue List       | HTML                | Generates HTML block listing overdue tasks        | Aggregate overdue tasks      | Merge groups              |                                                                                                               |
| HTML due to List        | HTML                | Generates HTML block listing upcoming tasks       | Aggregate due to tasks       | Merge groups              |                                                                                                               |
| Merge groups            | Merge               | Combines overdue and upcoming task HTML blocks   | HTML overdue List, HTML due to List | Aggregate                 |                                                                                                               |
| Aggregate               | Aggregate           | Aggregates combined HTML groups                    | Merge groups                | HTML                      |                                                                                                               |
| HTML                    | HTML                | Creates final styled email HTML template           | Aggregate                   | Send Email                | Create html email template: style and branding, includes logo and task groups                                |
| Send Email              | EmailSend           | Sends the generated HTML email                     | HTML                        | Pushover                  | Send email and push notification. Setup SMTP and email addresses.                                            |
| Pushover                | Pushover            | Sends push notification to phone                   | Send Email                  | -                         | Use Pushover User Key to receive notifications. See Pushover docs for setup.                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: ScheduleTrigger  
   - Set to run every week on Monday at 9:00 AM.  

2. **Create a Manual Trigger node:**  
   - Type: ManualTrigger  
   - For testing runs.

3. **Create a Set node ("Set Workflow vars"):**  
   - Add variables:  
     - `logo_path` (string): URL or path to logo image  
     - `pushover_user_key` (string): your Pushover user key  
     - `notion_database_url` (string): URL to your Notion database  
     - `your_email` (string): recipient email address  
   - Connect Schedule Trigger and Manual Trigger outputs to this node input.

4. **Create a Notion node:**  
   - Resource: databasePage  
   - Operation: getAll  
   - Database ID: your Notion task database's ID  
   - Set filter to exclude closed or done tasks (e.g., filter where Status != "Closed").  
   - Attach Notion API credentials.  
   - Connect output of Set Workflow vars to this node.

5. **Create a Filter node ("Filter for deadline"):**  
   - Condition: Keep items where the property `deadline.start` exists (is not empty).  
   - Connect Notion node output to this node.

6. **Create a Sort node ("Sort by closest deadline"):**  
   - Sort field: `property_deadline.start` ascending.  
   - Connect Filter node output to Sort node.

7. **Create an HTML node ("HTML for Task"):**  
   - HTML template for each task:  
     ```html
     <div class="task">
       <a href="{{ $json.url }}">
         <h3>{{ $json.name }}"</h3>
       </a>
       <p>
         <strong>Deadline: </strong>{{ $json.property_deadline.start.toDateTime().format('dd.MM.yyyy') }}<br>
         <strong>Prio: </strong>{{ $json.property_prio }}<br>
         <strong>Status: </strong>{{ $json.property_status }}<br>
         <strong>Tags: </strong>{{ $json.property_tags }}
       </p>
     </div>
     ```
   - Connect output of Sort node to this node.

8. **Create a Merge node:**  
   - Mode: Combine by position (default).  
   - Connect Sort node output (original) to input 1, and HTML for Task output to input 2.  

9. **Create an If node ("If deadline is overdue"):**  
   - Condition: Check if `property_deadline.start` is before or equal to now (`$now`).  
   - Connect Merge node output to this node.

10. **Create two Aggregate nodes:**  
    - "Aggregate overdue tasks": aggregates all items from If node **true** output under `overdue`.  
    - "Aggregate due to tasks": aggregates all items from If node **false** output under `due_to`.  
    - Connect If node outputs accordingly.

11. **Create two HTML nodes:**  
    - "HTML overdue List":  
      ```html
      <h2>Tasks which are already <u>overdue</u></h2>
      {{ $if($json.overdue.length > 0, $json.overdue.pluck('html'), 'No overdue tasks. Great!') }}
      ```
      Connect from "Aggregate overdue tasks."

    - "HTML due to List":  
      ```html
      <h2>Tasks with an <u>upcoming</u> deadline</h2>
      {{ $json.due_to.pluck('html') }}
      ```
      Connect from "Aggregate due to tasks."

12. **Create a Merge node ("Merge groups"):**  
    - Connect outputs of both HTML overdue List and HTML due to List nodes.

13. **Create an Aggregate node:**  
    - Aggregate all merged HTML groups into field `html_groups`.  
    - Connect output from Merge groups node.

14. **Create an HTML node ("HTML") for final email template:**  
    - Use the provided full HTML template with inline CSS, embedding:  
      - Logo path: `{{ $('Set Workflow vars').item.json.logo_path }}`  
      - Notion database URL: `{{ $('Set Workflow vars').item.json.notion_database_url }}`  
      - Task groups HTML: `{{ $json.html_groups.pluck('html') }}`  

15. **Create an Email Send node:**  
    - Use SMTP credentials configured with your mail server.  
    - From email: `n8n@unitize.de` (or your preferred sender)  
    - To email: `{{ $('Set Workflow vars').item.json.your_email }}`  
    - Subject: "Weekly Update about Notion Tasks"  
    - HTML body: `={{ $json.html }}`  
    - Connect from final HTML node.

16. **Create a Pushover node:**  
    - Credentials: Your Pushover API credentials.  
    - User Key: `={{ $('Set Workflow vars').item.json.pushover_user_key }}`  
    - Message: "You received a weekly update about your Notion Tasks. Check your mails!"  
    - Priority: 1  
    - Connect from Send Email node.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| You need to have access to your Notion database and API credentials properly configured.              | Workflow prerequisites                                            |
| You must create a Pushover account and generate a User Key to receive push notifications.             | Pushover API setup                                                |
| Adjust the Notion node filters to exclude “Closed” or “Done” tasks to avoid receiving reminders for them. | Customizing task filtering                                        |
| Customize the HTML email template to change styles, add logos, or modify content as needed.           | HTML styling and branding                                         |
| This workflow runs every Monday at 9 AM to provide weekly reminders.                                  | Schedule Trigger info                                             |
| For more help implementing this or other workflows, contact via LinkedIn or [Unitize website](https://www.unitize.de). | Support contacts                                                 |
| To start using n8n, register via affiliate link: [https://n8n.partnerlinks.io/edr9c63lw12z](https://n8n.partnerlinks.io/edr9c63lw12z) | n8n registration affiliate link                                   |

---

This document provides a complete technical reference for the workflow, enabling advanced users and automation agents to understand, reproduce, and customize the workflow effectively.