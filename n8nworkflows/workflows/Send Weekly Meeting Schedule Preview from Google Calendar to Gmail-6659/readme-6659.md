Send Weekly Meeting Schedule Preview from Google Calendar to Gmail

https://n8nworkflows.xyz/workflows/send-weekly-meeting-schedule-preview-from-google-calendar-to-gmail-6659


# Send Weekly Meeting Schedule Preview from Google Calendar to Gmail

### 1. Workflow Overview

This workflow automates sending a weekly preview email of your upcoming Google Calendar events for the next week. It is designed to run every Sunday evening, gather all calendar events scheduled for the upcoming Monday through Sunday, format these events into a clear, user-friendly summary, and send this summary via Gmail to a specified email address.

The workflow is logically divided into the following blocks:

- **1.1 Trigger Setup:** Automatically triggers the workflow every Sunday at 7:00 PM.
- **1.2 Date Calculation:** Calculates the date range (start and end) for the upcoming week.
- **1.3 Calendar Data Retrieval:** Fetches all Google Calendar events for the calculated week.
- **1.4 Event Formatting:** Processes and formats the events into a readable summary.
- **1.5 Email Sending:** Sends the formatted summary as an email via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Setup

- **Overview:**  
  This block initiates the workflow every Sunday at 7 PM server local time, ensuring the weekly schedule preview is sent consistently once per week.

- **Nodes Involved:**  
  - Sunday Evening Trigger (Cron)

- **Node Details:**

  - **Sunday Evening Trigger**  
    - Type: Cron Trigger  
    - Role: Schedules the workflow execution weekly on Sunday evenings.  
    - Configuration:  
      - Mode: Runs every week  
      - Time: 19:00 (7:00 PM)  
      - Weekdays: Sunday only  
    - Inputs: None (trigger node)  
    - Outputs: Emits a single trigger event to start the workflow  
    - Edge Cases:  
      - Server time zone changes may affect trigger time; ensure server time zone aligns with user expectations.  
      - If the node is disabled, workflow will not trigger automatically.  
    - Notes:  
      - Adjust schedule by changing time or weekday parameters.

#### 1.2 Date Calculation

- **Overview:**  
  Calculates the exact ISO date-time range for the upcoming week, from Monday 00:00:00 to Sunday 23:59:59, using Luxon's DateTime utilities.

- **Nodes Involved:**  
  - Calculate Week Dates (Function)

- **Node Details:**

  - **Calculate Week Dates**  
    - Type: Function  
    - Role: Computes the start and end date for the next calendar week.  
    - Configuration:  
      - Uses Luxon DateTime to calculate:  
        - `startDate` as next Monday at 00:00:00  
        - `endDate` as next Sunday at 23:59:59  
      - Returns these as ISO strings for downstream use.  
    - Inputs: Trigger event from Cron node (no data used)  
    - Outputs: JSON with keys `startDate` and `endDate`  
    - Edge Cases:  
      - None significant; assumes server time is correct.  
      - If Luxon library is unavailable or the function environment changes, errors may occur.  
    - Notes:  
      - No user configuration needed.

#### 1.3 Calendar Data Retrieval

- **Overview:**  
  Fetches all Google Calendar events that start within the calculated date range for the upcoming week, retrieving up to 20 upcoming events.

- **Nodes Involved:**  
  - Get Calendar Events (Google Calendar)

- **Node Details:**

  - **Get Calendar Events**  
    - Type: Google Calendar node  
    - Role: Queries Google Calendar API to retrieve events between `startDate` and `endDate`.  
    - Configuration:  
      - Operation: `getAll` (fetch all matching events)  
      - Calendar ID: `primary` (main calendar)  
      - Time Min: Dynamic expression referencing `startDate` from previous node  
      - Time Max: Dynamic expression referencing `endDate` from previous node  
      - Order By: `startTime`  
      - Max Results: 20 (limit to 20 events for performance)  
      - Single Events: true (expands recurring events)  
    - Credentials: Google Calendar OAuth2 API (must be configured)  
    - Inputs: JSON containing `startDate` and `endDate`  
    - Outputs: Array of event objects with details such as start time, end time, summary, location  
    - Edge Cases:  
      - Authentication errors if OAuth token expired or invalid  
      - API quota limits could cause failure  
      - No events found returns empty array (handled downstream)  
      - If calendar ID is incorrect or inaccessible, errors occur  
    - Notes:  
      - Requires proper OAuth credentials and enabled API in Google Cloud Console.

#### 1.4 Event Formatting

- **Overview:**  
  Processes the retrieved events, grouping them by day, sorting them chronologically, and formatting the information into a Markdown-style summary string for email.

- **Nodes Involved:**  
  - Format Meeting Summary (Function)

- **Node Details:**

  - **Format Meeting Summary**  
    - Type: Function  
    - Role: Converts raw event data into a human-readable summary string suitable for email content.  
    - Logic:  
      - If no events, outputs a friendly "no meetings" message.  
      - Groups events by day using the event start date.  
      - Sorts days and events chronologically.  
      - Formats each event with start and end time (or “All Day”), title, and location if present.  
      - Uses Markdown syntax (e.g., `**Day Name**`) for emphasis in emails.  
    - Inputs: Array of event JSON objects from Google Calendar node  
    - Outputs: JSON object with key `summaryEmailContent` containing the formatted string  
    - Edge Cases:  
      - All-day events handled gracefully  
      - Missing fields (e.g., no location, no summary) replaced with defaults  
      - Timezone differences may affect displayed times depending on local environment  
      - If event start or end times are malformed, could cause formatting errors  
    - Notes:  
      - No user configuration required.

#### 1.5 Email Sending

- **Overview:**  
  Sends the formatted meeting summary via Gmail to a specified recipient email address, including a subject line with the upcoming week's date range.

- **Nodes Involved:**  
  - Send Summary Email (Gmail)

- **Node Details:**

  - **Send Summary Email**  
    - Type: Gmail node  
    - Role: Sends email via Gmail API containing the weekly meeting summary.  
    - Configuration:  
      - From Email: Must be the authenticated Gmail address.  
      - To Email: User must replace placeholder with actual recipient email.  
      - Subject: Uses dynamic expressions to include formatted date range for next week (e.g., "Your Upcoming Meeting Schedule: Apr 01 - Apr 07").  
      - Text: Email body includes greeting, the formatted summary from previous node, and a sign-off.  
    - Credentials: Gmail API OAuth2 (must be configured)  
    - Inputs: JSON with `summaryEmailContent` from formatting node  
    - Outputs: Success or error response from Gmail API  
    - Edge Cases:  
      - Authentication failure if OAuth token expired or misconfigured  
      - Incorrect recipient email address causes delivery failure  
      - Gmail API quota limits could disrupt sending  
      - Email formatting may vary depending on client rendering Markdown as plain text  
    - Notes:  
      - User must replace placeholder emails with actual addresses before production use.  
      - Supports rich text but currently sends as plain text with Markdown formatting.

---

### 3. Summary Table

| Node Name           | Node Type             | Functional Role               | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                                                                                                                                                              |
|---------------------|-----------------------|------------------------------|-----------------------|----------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sunday Evening Trigger | Cron                  | Weekly trigger on Sunday 7PM | None                  | Calculate Week Dates  | ### 1. Trigger on Sunday Evening\n\nThis `Cron` node is configured to run automatically every **Sunday at 7:00 PM (19:00)** your server's local time.\n\n**To change the schedule:**\n* Adjust 'Time' to your preferred hour.\n* Adjust 'Weekdays' if you want it to run on a different day. |
| Calculate Week Dates | Function              | Calculate next week's dates  | Sunday Evening Trigger | Get Calendar Events   | ### 2. Calculate Upcoming Week Dates\n\nThis `Function` node uses JavaScript (with Luxon's DateTime library, which n8n uses internally) to calculate the start and end dates for the *upcoming* week (Monday to Sunday).\n\n**No configuration needed here**, it's ready to go.                |
| Get Calendar Events  | Google Calendar       | Retrieve week's calendar events | Calculate Week Dates | Format Meeting Summary | ### 3. Get Upcoming Calendar Events\n\nThis `Google Calendar` node fetches all events within the calculated `startDate` and `endDate` range.\n\n**Setup:**\n1.  **Google Calendar Credential:** Click on 'Credentials' and select 'New Credential'. Choose 'Google Calendar OAuth2 API'. Follow the n8n instructions to connect your Google account. (You might need to enable the Google Calendar API in your Google Cloud Project).\n2.  **Calendar ID:** `primary` usually works for your main calendar. If you want to use a specific calendar, get its ID from Google Calendar settings and paste it here.\n3.  **Time Min/Max:** These are automatically populated with the `startDate` and `endDate` from the previous 'Calculate Week Dates' node. |
| Format Meeting Summary | Function              | Format events into email summary | Get Calendar Events  | Send Summary Email    | ### 4. Format Meeting Summary\n\nThis `Function` node processes the calendar events and formats them into a readable summary string for your email.\n\n**No configuration needed here**, it's pre-programmed to format your events.                                                                |
| Send Summary Email   | Gmail                 | Send summary email via Gmail  | Format Meeting Summary | None                 | ### 5. Send Summary Email\n\nThis `Gmail` node sends the compiled meeting summary to your inbox.\n\n**Setup:**\n1.  **Gmail Credential:** Click 'Credentials' and select 'New Credential'. Choose 'Gmail API'. Follow the n8n instructions to connect your Gmail account.\n2.  **From Email:** Enter your Gmail address (this must be the same account you authenticated).\n3.  **To Email:** **IMPORTANT: Change `YOUR_RECIPIENT_EMAIL@example.com` to your actual email address!**\n4.  **Subject:** Automatically includes the upcoming week's date range.\n5.  **Text:** Uses the `summaryEmailContent` generated by the previous node.\n\nAfter setting up, click 'Execute Workflow' (from the 'Sunday Evening Trigger' node) to test sending an email! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger Node:**  
   - Name: `Sunday Evening Trigger`  
   - Type: Cron Trigger  
   - Parameters:  
     - Mode: Every week  
     - Time: 19:00 (7 PM)  
     - Weekdays: Sunday  
   - This node triggers the workflow weekly.

2. **Add a Function Node to Calculate Week Dates:**  
   - Name: `Calculate Week Dates`  
   - Type: Function  
   - Paste the following JavaScript code into the function editor:

     ```javascript
     const DateTime = require('luxon').DateTime;

     const now = DateTime.now();

     // Calculate next Monday (start of upcoming week)
     const nextMonday = now.startOf('week').plus({ weeks: 1 });

     // Calculate next Sunday (end of upcoming week)
     const nextSunday = nextMonday.plus({ days: 6 }).endOf('day');

     return [{ json: { 
       startDate: nextMonday.toISO(), 
       endDate: nextSunday.toISO() 
     } }];
     ```

   - Connect `Sunday Evening Trigger` output to this node’s input.

3. **Add Google Calendar Node to Fetch Events:**  
   - Name: `Get Calendar Events`  
   - Type: Google Calendar  
   - Operation: `getAll`  
   - Calendar ID: `primary` (or specific calendar ID if preferred)  
   - Time Min: `={{ $json.startDate }}` (expression)  
   - Time Max: `={{ $json.endDate }}` (expression)  
   - Order By: `startTime`  
   - Max Results: 20  
   - Single Events: true  
   - Credentials: Configure Google Calendar OAuth2 credentials with appropriate scopes and Google API enabled.  
   - Connect `Calculate Week Dates` output to this node’s input.

4. **Add Function Node to Format Meeting Summary:**  
   - Name: `Format Meeting Summary`  
   - Type: Function  
   - Paste the following JavaScript code:

     ```javascript
     let summary = "";

     if (items.length === 0) {
       summary = "Good news! You have no meetings scheduled for the upcoming week.\n";
     } else {
       summary = "Here's a summary of your upcoming meetings for the week:\n\n";

       // Group events by day
       const eventsByDay = {};
       for (const item of items) {
         const event = item.json;
         const startDateTime = new Date(event.start.dateTime || event.start.date); // Handle all-day events
         const dateKey = startDateTime.toLocaleDateString('en-US', { weekday: 'long', month: 'long', day: 'numeric' });

         if (!eventsByDay[dateKey]) {
           eventsByDay[dateKey] = [];
         }
         eventsByDay[dateKey].push(event);
       }

       // Sort days chronologically
       const sortedDays = Object.keys(eventsByDay).sort((a, b) => {
         const dateA = new Date(a);
         const dateB = new Date(b);
         return dateA - dateB;
       });

       // Build the summary string
       for (const day of sortedDays) {
         summary += `**${day}**:\n`;
         // Sort events within the day by time
         eventsByDay[day].sort((a, b) => {
           const timeA = new Date(a.start.dateTime || a.start.date);
           const timeB = new Date(b.start.dateTime || b.start.date);
           return timeA - timeB;
         });
         for (const event of eventsByDay[day]) {
           const startTime = event.start.dateTime ? new Date(event.start.dateTime).toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: true }) : 'All Day';
           const endTime = event.end.dateTime ? new Date(event.end.dateTime).toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit', hour12: true }) : '';
           const location = event.location ? ` (at ${event.location})` : '';
           const summaryText = event.summary || 'No Title';

           if (startTime === 'All Day') {
             summary += `- ${summaryText} (All Day)${location}\n`;
           } else {
             summary += `- ${startTime}${endTime ? ` - ${endTime}` : ''}: ${summaryText}${location}\n`;
           }
         }
         summary += "\n"; // Blank line between days
       }
     }

     return [{ json: { summaryEmailContent: summary } }];
     ```

   - Connect `Get Calendar Events` output to this node’s input.

5. **Add Gmail Node to Send the Email:**  
   - Name: `Send Summary Email`  
   - Type: Gmail  
   - Parameters:  
     - From Email: Your authenticated Gmail address (must match OAuth credentials)  
     - To Email: Replace `YOUR_RECIPIENT_EMAIL@example.com` with the actual recipient email address  
     - Subject: Use expression to set date range, e.g.:  
       ```
       Your Upcoming Meeting Schedule: {{ $now.plus({ weeks: 1 }).startOf('week').toFormat('LLL dd') }} - {{ $now.plus({ weeks: 1 }).endOf('week').toFormat('LLL dd') }}
       ```  
     - Text:  
       ```
       Hello!

       {{ $json.summaryEmailContent }}

       Have a productive week!

       Best,
       Your n8n Automation
       ```  
   - Credentials: Configure Gmail API OAuth2 credentials with proper scopes.  
   - Connect `Format Meeting Summary` output to this node’s input.

6. **Test the Workflow:**  
   - Manually trigger the `Sunday Evening Trigger` node to run the workflow immediately for testing.  
   - Verify email delivery and formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| You must replace placeholders for Gmail and recipient emails with your actual email addresses before running in production.                                                                                                                            | Email sending node configuration                                                                                       |
| Google Calendar API must be enabled in your Google Cloud Console project, and OAuth2 credentials configured with proper scopes to allow reading calendar events.                                                                                        | Google Cloud Console, n8n credential setup                                                                             |
| Gmail API OAuth2 credentials must be created and authorized for sending emails on your Gmail account.                                                                                                                                                   | Google Cloud Console, n8n credential setup                                                                             |
| The workflow uses Luxon's DateTime library internally for date calculations, available by default in n8n function nodes.                                                                                                                                | https://moment.github.io/luxon/                                                                                        |
| The email format uses Markdown syntax for bold day headers; Gmail will render this as plain text unless the email client supports Markdown interpretation or you switch to HTML formatting with a different node.                                      | Formatting note                                                                                                         |
| To modify the trigger time or day, adjust the Cron node's time and weekday parameters accordingly.                                                                                                                                                      | Cron node configuration                                                                                                |
| This workflow can be extended by increasing `maxResults` in the Google Calendar node to capture more events or by adding error handling nodes for resilience in case of API failures or credential issues.                                               | Scalability and reliability considerations                                                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.