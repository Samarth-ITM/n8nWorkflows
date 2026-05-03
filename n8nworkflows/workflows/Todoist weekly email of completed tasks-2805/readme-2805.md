Todoist weekly email of completed tasks

https://n8nworkflows.xyz/workflows/todoist-weekly-email-of-completed-tasks-2805


# Todoist weekly email of completed tasks

### 1. Workflow Overview

This workflow provides Todoist users with a weekly email summary of their completed tasks. It addresses the limitation of Todoist’s native reporting by fetching completed tasks via Todoist’s public API, filtering out unwanted projects, and sending a neatly formatted email grouped by completion day.

The workflow is logically divided into these blocks:

- **1.1 Trigger Block:** Initiates the workflow either on a scheduled weekly basis (Friday afternoon) or manually.
- **1.2 Data Retrieval Block:** Fetches completed tasks from Todoist API for the past week.
- **1.3 Filtering Block:** Optionally excludes tasks from specified Todoist projects.
- **1.4 Formatting Block:** Groups tasks by completion date and formats them into an HTML email body.
- **1.5 Email Sending Block:** Sends the formatted summary email via SMTP or another configured email service.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Block

- **Overview:**  
  Starts the workflow either automatically every Friday afternoon or manually via a button click.

- **Nodes Involved:**  
  - Every Friday afternoon  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **Every Friday afternoon**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow weekly on Fridays at 3 PM.  
    - Configuration: Set to trigger every week on day 5 (Friday) at hour 15 (3 PM).  
    - Inputs: None  
    - Outputs: Connects to "Get completed tasks via Todoist API" node.  
    - Edge cases: Misconfigured timezone could cause unexpected trigger times.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or on-demand runs.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Get completed tasks via Todoist API" node.  
    - Edge cases: None significant; manual trigger depends on user action.

---

#### 1.2 Data Retrieval Block

- **Overview:**  
  Retrieves all completed tasks from the last 7 days using the Todoist Sync API.

- **Nodes Involved:**  
  - Get completed tasks via Todoist API

- **Node Details:**

  - **Get completed tasks via Todoist API**  
    - Type: HTTP Request  
    - Role: Calls Todoist’s sync API endpoint to fetch completed tasks within a date range.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.todoist.com/sync/v9/completed/get_all`  
      - Body Parameters:  
        - `since`: Current date minus 7 days (dynamic expression `{{$now.minus(7, 'days')}}`)  
        - `until`: Current date/time (`{{$now}}`)  
      - Authentication: Uses predefined Todoist API credential.  
    - Inputs: Trigger nodes ("Every Friday afternoon" or "When clicking ‘Test workflow’")  
    - Outputs: Passes JSON response containing completed tasks to the filtering block.  
    - Edge cases:  
      - API rate limits or downtime could cause failures.  
      - Invalid or expired API token leads to authentication errors.  
      - Network timeouts or malformed responses.  
    - Version-specific: Uses HTTP Request node version 4.2.

---

#### 1.3 Filtering Block

- **Overview:**  
  Removes tasks belonging to specified Todoist projects to exclude unwanted data from the summary.

- **Nodes Involved:**  
  - Optional: Ignore specific projects

- **Node Details:**

  - **Optional: Ignore specific projects**  
    - Type: Code (JavaScript)  
    - Role: Filters out tasks whose `project_id` matches any ID in the `ignoredProjects` array.  
    - Configuration:  
      - `ignoredProjects` array is user-editable; default includes one example project ID `['2335544024']`.  
      - Iterates over all tasks, excludes those with matching project IDs.  
    - Key expressions:  
      - Accesses input tasks via `$input.all()[0].json.items`  
      - Returns filtered array of tasks.  
    - Inputs: Output from "Get completed tasks via Todoist API" node.  
    - Outputs: Filtered task list to the formatting block.  
    - Edge cases:  
      - If `ignoredProjects` is empty, no filtering occurs.  
      - If input data structure changes (e.g., missing `items`), code may fail.  
      - Large task lists could impact performance.  
    - Version-specific: Code node version 2.

---

#### 1.4 Formatting Block

- **Overview:**  
  Groups the filtered tasks by their completion date and formats them into an HTML email body.

- **Nodes Involved:**  
  - Format the email body

- **Node Details:**

  - **Format the email body**  
    - Type: Code (JavaScript)  
    - Role: Processes the filtered tasks to group by date and generate an HTML string for email content.  
    - Configuration:  
      - Groups tasks by extracting the date part from `completed_at` timestamp.  
      - Creates an HTML structure with headings for each date and bullet points for tasks.  
      - Returns an object with `emailBody` property containing the HTML string.  
    - Key expressions:  
      - Uses `reduce` to group tasks by date.  
      - Formats HTML with `<h1>`, `<h2>`, and `<ul><li>` tags.  
    - Inputs: Filtered tasks from "Optional: Ignore specific projects".  
    - Outputs: JSON with `emailBody` passed to the email sending node.  
    - Edge cases:  
      - If no tasks, email body will contain only the header "Completed Items" with no lists.  
      - Date parsing errors if `completed_at` is missing or malformed.  
    - Version-specific: Code node version 2.

---

#### 1.5 Email Sending Block

- **Overview:**  
  Sends the formatted weekly summary email using SMTP or another configured email service.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**

  - **Send Email**  
    - Type: Email Send  
    - Role: Sends the email with the subject "Todoist Weekly Review" and the HTML content from the previous node.  
    - Configuration:  
      - Subject: Static text "Todoist Weekly Review"  
      - Email format: HTML content dynamically set from `Format the email body` node's `emailBody` property.  
      - Credentials: Uses SMTP or other email service credentials configured in n8n.  
      - Sender and recipient emails must be set by the user in node parameters.  
    - Inputs: Receives formatted email body from "Format the email body" node.  
    - Outputs: None (end of workflow).  
    - Edge cases:  
      - SMTP authentication failures or misconfiguration.  
      - Email delivery failures or spam filtering.  
      - Missing sender or recipient addresses cause errors.  
    - Version-specific: Email Send node version 2.1.

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                           | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------------|---------------------|-----------------------------------------|-------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Every Friday afternoon         | Schedule Trigger    | Weekly automatic trigger                 | None                          | Get completed tasks via Todoist API |                                                                                                |
| When clicking ‘Test workflow’  | Manual Trigger      | Manual trigger for testing or on-demand | None                          | Get completed tasks via Todoist API |                                                                                                |
| Get completed tasks via Todoist API | HTTP Request       | Fetch completed tasks from Todoist API  | Every Friday afternoon, When clicking ‘Test workflow’ | Optional: Ignore specific projects |                                                                                                |
| Optional: Ignore specific projects | Code                | Filter out tasks from ignored projects  | Get completed tasks via Todoist API | Format the email body           | Modify the `ignoredProjects` array to exclude specific Todoist projects from the summary. See [Todoist Projects API](https://api.todoist.com/rest/v2/projects) |
| Format the email body          | Code                | Group tasks by day and format HTML email| Optional: Ignore specific projects | Send Email                    |                                                                                                |
| Send Email                    | Email Send          | Send the weekly summary email            | Format the email body          | None                           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Schedule Trigger** node named "Every Friday afternoon":  
     - Set interval to weekly.  
     - Configure to trigger on Friday (day 5) at 15:00 (3 PM).

   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" for manual runs.

2. **Create HTTP Request Node:**

   - Add an **HTTP Request** node named "Get completed tasks via Todoist API".  
   - Set method to POST.  
   - Set URL to `https://api.todoist.com/sync/v9/completed/get_all`.  
   - Enable sending body as JSON.  
   - Add body parameters:  
     - `since` = Expression: `{{$now.minus(7, 'days')}}`  
     - `until` = Expression: `{{$now}}`  
   - Set authentication to use a **Todoist API** credential (create this credential beforehand with your Todoist API token).  
   - Connect both trigger nodes ("Every Friday afternoon" and "When clicking ‘Test workflow’") to this node.

3. **Create Filtering Code Node:**

   - Add a **Code** node named "Optional: Ignore specific projects".  
   - Paste the following JavaScript code:

     ```javascript
     // maintain this array with ignored Todoist project_id's
     // empty "[]" it when you don't want to ignore any
     const ignoredProjects = ['2335544024'];

     // Remove ignored projects
     const items = $input.all()[0].json.items;
     var newItems = [];
     for(let j = 0; j < items.length; j++) {
       if(!ignoredProjects.includes(items[j].project_id)) {
         newItems.push(items[j]);
       }
     }

     return newItems;
     ```

   - Connect the output of "Get completed tasks via Todoist API" to this node.

4. **Create Formatting Code Node:**

   - Add a **Code** node named "Format the email body".  
   - Paste the following JavaScript code:

     ```javascript
     const items = $input.all();

     // Group items by day
     const grouped = items.reduce((acc, item) => {
       const date = new Date(item.json.completed_at).toISOString().split('T')[0];
       acc[date] = acc[date] || [];
       acc[date].push(item.json.content);
       return acc;
     }, {});

     // Format the grouped data into an HTML string for the email
     let emailBody = "<h1>Completed Items</h1>";
     for (const [date, contents] of Object.entries(grouped)) {
       emailBody += `<h2>${date}</h2><ul>`;
       contents.forEach(content => {
         emailBody += `<li>${content}</li>`;
       });
       emailBody += `</ul>`;
     }

     return [{ json: { emailBody } }];
     ```

   - Connect the output of "Optional: Ignore specific projects" to this node.

5. **Create Email Send Node:**

   - Add an **Email Send** node named "Send Email".  
   - Set the subject to: `Todoist Weekly Review`.  
   - Set the email format to HTML and use an expression to set the email body:  
     `={{ $('Format the email body').item.json.emailBody }}`  
   - Configure the sender and recipient email addresses as required.  
   - Select or create an SMTP credential or alternative email service credential (e.g., Brevo, Mailjet).  
   - Connect the output of "Format the email body" to this node.

6. **Connect Workflow:**

   - Connect both trigger nodes ("Every Friday afternoon" and "When clicking ‘Test workflow’") to "Get completed tasks via Todoist API".  
   - Connect "Get completed tasks via Todoist API" to "Optional: Ignore specific projects".  
   - Connect "Optional: Ignore specific projects" to "Format the email body".  
   - Connect "Format the email body" to "Send Email".

7. **Test and Activate:**

   - Run the workflow manually using the manual trigger node to verify functionality.  
   - Check your email inbox for the weekly summary.  
   - Activate the workflow to enable scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Todoist API token can be found in Todoist settings under Integrations > Developer.                             | https://app.todoist.com/app/settings/integrations/developer                                        |
| To exclude specific projects, update the `ignoredProjects` array with project IDs.                             | https://api.todoist.com/rest/v2/projects (to find project IDs)                                     |
| SMTP credentials can be replaced with other email services supported by n8n like Brevo or Mailjet.             | n8n documentation on Email Send node credentials                                                   |
| The workflow runs every Friday at 3 PM by default; adjust schedule trigger as needed for different timing.     |                                                                                                    |
| This workflow solves the lack of completed task reports in Todoist’s native app by leveraging their public API.|                                                                                                    |

---

This document fully describes the "Todoist weekly email of completed tasks" workflow, enabling reproduction, modification, and troubleshooting by users and automation agents alike.