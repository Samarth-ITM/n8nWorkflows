Automated Daily Outlook Calendar Meeting Digest

https://n8nworkflows.xyz/workflows/automated-daily-outlook-calendar-meeting-digest-4420


# Automated Daily Outlook Calendar Meeting Digest

### 1. Workflow Overview

This workflow automates the generation and sending of a daily digest email summarizing Microsoft Outlook calendar meetings scheduled for the current day. It is designed for users who want an automated, formatted overview of their daily meetings delivered by email every morning.

The workflow is composed of the following logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow daily at a configured time.
- **1.2 Date Calculation:** Computes the date range boundaries (today and tomorrow) to filter events.
- **1.3 Microsoft Outlook Event Fetch:** Retrieves all calendar events starting today but before tomorrow.
- **1.4 Data Formatting:** Extracts and restructures relevant meeting details from raw Outlook event data.
- **1.5 HTML Email Generation:** Converts the meeting data into a styled HTML email body.
- **1.6 Email Sending:** Sends the generated email to the configured recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block triggers the entire workflow once daily at a specific hour, initiating the calendar data retrieval process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger daily at 8:00 AM (configurable hour)  
    - Key Parameters: Interval set to triggerAtHour=8  
    - Input: N/A (starting node)  
    - Output: Emits event to 'Code' node for date calculations  
    - Edge Cases: Misconfiguration can cause incorrect trigger times or no triggers; time zone considerations may affect expected run time.

  - **Sticky Note1**  
    - Context: Reminder to update the trigger time if needed.  
    - Content: "## Update Time\nconfigure the default time, when this workflow should trigger"

---

#### 1.2 Date Calculation

- **Overview:**  
  Calculates ISO-formatted start and end timestamps for "today" (midnight) and "tomorrow" (midnight next day) to filter events within this 24-hour window.

- **Nodes Involved:**  
  - Code (JavaScript)

- **Node Details:**

  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration: Calculates current date at midnight (`today`) and next date midnight (`tomorrow`) and converts them to ISO strings without milliseconds.  
    - Key Expressions:  
      ```js
      const now = new Date();
      const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
      const tomorrow = new Date(today);
      tomorrow.setDate(today.getDate() + 1);
      const isoDate = (d) => d.toISOString().split('.')[0] + 'Z';
      return [{ json: { today: isoDate(today), tomorrow: isoDate(tomorrow) } }];
      ```  
    - Input: Trigger data from Schedule Trigger  
    - Output: JSON with `today` and `tomorrow` ISO timestamps  
    - Edge Cases: Date/time calculation depends on server timezone; daylight savings changes could affect accuracy.

---

#### 1.3 Microsoft Outlook Event Fetch

- **Overview:**  
  Queries Microsoft Outlook calendar events starting today but before tomorrow using the calculated date range.

- **Nodes Involved:**  
  - Microsoft Outlook

- **Node Details:**

  - **Microsoft Outlook**  
    - Type: Microsoft Outlook node (OAuth2 authentication)  
    - Configuration:  
      - Resource: `event`  
      - Filter: Custom OData filter to select events where start datetime >= today and < tomorrow  
      - Filter expression:  
        `=start/dateTime ge '{{$json.today}}' and start/dateTime lt '{{$json.tomorrow}}'`  
      - Credentials: OAuth2 linked to Outlook account ("Outlook - Jared")  
    - Input: JSON with `today` and `tomorrow` from Code node  
    - Output: List of Outlook events matching the filter  
    - Edge Cases:  
      - Authentication failure or expired token errors  
      - API rate limits or timeouts  
      - Empty result sets if no meetings scheduled  
      - Filter syntax errors can cause no events returned or failures

---

#### 1.4 Data Formatting

- **Overview:**  
  Extracts and reshapes relevant event fields into a simplified JSON structure for downstream processing and email generation.

- **Nodes Involved:**  
  - Edit Fields (Set node)

- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Maps Outlook event properties to new fields:  
      - `id` (event id)  
      - `subject` (event title)  
      - `description` (event body preview)  
      - `meeting_start` and `meeting_end` (start and end datetimes)  
      - `attendees` (array of attendees)  
      - `meeting_organizer` (organizer name)  
      - `meeting_organizer_email` (organizer email)  
      - `meeting_link` (web link to event in Outlook)  
    - Expressions use direct JSON path access, e.g., `{{$json.subject}}`  
    - Input: Raw Outlook event JSON objects  
    - Output: Cleaned and simplified meeting JSON objects  
    - Edge Cases: Missing fields in Outlook data could cause undefined values; empty attendees array possible.

---

#### 1.5 HTML Email Generation

- **Overview:**  
  Generates a fully styled HTML email summarizing each meeting's key details, including time, organizer, description, attendees, and a clickable calendar link.

- **Nodes Involved:**  
  - Generate HTML (Code node)

- **Node Details:**

  - **Generate HTML**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Aggregates all meeting JSON objects from previous node into one array  
      - Defines function `generateMeetingReminderEmail` that:  
        - Escapes HTML special characters to prevent injection  
        - Formats meeting start and end times in US English locale with date and time  
        - Separates required and optional attendees by filtering attendee type  
        - Builds individual meeting cards as HTML tables with styles for readability  
        - Includes meeting subject, time, organizer info, description, attendees, and a button linking to the calendar event  
      - Wraps all meeting cards in a full HTML document with simple styling and header ("Your Meetings Today")  
      - Sets email subject dynamically to include the current date (e.g., "🗓️ Your Meetings Today – Monday, Jun 5")  
    - Key Expressions:  
      Uses `$input.all()`, date formatting functions, and template literals for HTML generation  
    - Input: Array of simplified meeting JSON objects from Edit Fields node  
    - Output: JSON with `html` content and `subject` string for email  
    - Edge Cases:  
      - Meetings without description handled gracefully ("No description provided.")  
      - Empty meeting list results in email with empty content section  
      - Potential encoding issues if attendee names or subjects contain special characters

---

#### 1.6 Email Sending

- **Overview:**  
  Sends the generated HTML email digest to the configured recipient using SMTP credentials.

- **Nodes Involved:**  
  - Send Email (Email Send node)  
  - Sticky Note (reminder for email configuration)

- **Node Details:**

  - **Send Email**  
    - Type: Email Send node  
    - Configuration:  
      - Subject and HTML body taken from previous node outputs (`$json.subject`, `$json.html`)  
      - From email: `notification@nuevesolutions.in` (configurable)  
      - To email: `akhil.nueve@gmail.com` (configurable)  
      - SMTP credentials provided via saved SMTP account  
      - Attribution disabled to avoid n8n footer in email  
    - Input: JSON with email subject and HTML content  
    - Output: Email sending result/status  
    - Edge Cases:  
      - SMTP authentication failure or connection issues  
      - Invalid email addresses causing sending errors  
      - Large email body size may cause sending delays or failures

  - **Sticky Note**  
    - Context: Reminder to update sender and receiver email addresses  
    - Content:  
      ```
      ## Update Email Details
      change email details to your sender and receiver email addresses.
      ```

---

### 3. Summary Table

| Node Name         | Node Type               | Functional Role                      | Input Node(s)      | Output Node(s)     | Sticky Note                                                                                   |
|-------------------|-------------------------|------------------------------------|--------------------|--------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger   | Schedule Trigger        | Initiates workflow daily at 8 AM   | -                  | Code               | ## Update Time<br>configure the default time, when this workflow should trigger               |
| Code              | Code (JavaScript)       | Calculates today's and tomorrow's ISO dates | Schedule Trigger   | Microsoft Outlook  |                                                                                               |
| Microsoft Outlook  | Microsoft Outlook       | Fetches calendar events for the day| Code               | Edit Fields        |                                                                                               |
| Edit Fields       | Set                     | Extracts relevant meeting fields   | Microsoft Outlook  | Generate HTML      |                                                                                               |
| Generate HTML     | Code (JavaScript)       | Creates styled HTML email content  | Edit Fields        | Send Email         |                                                                                               |
| Send Email        | Email Send              | Sends the email digest             | Generate HTML      | -                  | ## Update Email Details<br>change email details to your sender and receiver email addresses.  |
| Sticky Note1      | Sticky Note             | Reminder to update trigger time    | -                  | -                  | ## Update Time<br>configure the default time, when this workflow should trigger               |
| Sticky Note       | Sticky Note             | Reminder to update email addresses | -                  | -                  | ## Update Email Details<br>change email details to your sender and receiver email addresses.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger daily at 8:00 AM (adjust `triggerAtHour` as needed)  
   - No credentials needed  
   - Connect output to next node (Code)

2. **Create Code Node for Date Calculation**  
   - Type: Code (JavaScript)  
   - Parameters: Insert code to calculate today's and tomorrow's ISO dates without milliseconds:  
     ```js
     const now = new Date();
     const today = new Date(now.getFullYear(), now.getMonth(), now.getDate());
     const tomorrow = new Date(today);
     tomorrow.setDate(today.getDate() + 1);

     const isoDate = (d) => d.toISOString().split('.')[0] + 'Z';

     return [
       {
         json: {
           today: isoDate(today),
           tomorrow: isoDate(tomorrow),
         },
       },
     ];
     ```  
   - Connect output to Microsoft Outlook node

3. **Create Microsoft Outlook Node to Fetch Events**  
   - Type: Microsoft Outlook  
   - Credentials: Configure OAuth2 with appropriate Outlook account  
   - Parameters:  
     - Resource: `event`  
     - Filters: Custom filter with expression  
       ```
       =start/dateTime ge '{{$json.today}}' and start/dateTime lt '{{$json.tomorrow}}'
       ```  
   - Connect output to Edit Fields node

4. **Create Set Node (Edit Fields) to Format Events**  
   - Type: Set node  
   - Parameters: Map event properties as:  
     - `id` = `{{$json.id}}`  
     - `subject` = `{{$json.subject}}`  
     - `description` = `{{$json.bodyPreview}}`  
     - `meeting_start` = `{{$json.start.dateTime}}`  
     - `meeting_end` = `{{$json.end.dateTime}}`  
     - `attendees` = `{{$json.attendees}}` (array)  
     - `meeting_organizer` = `{{$json.organizer.emailAddress.name}}`  
     - `meeting_organizer_email` = `{{$json.organizer.emailAddress.address}}`  
     - `meeting_link` = `{{$json.webLink}}`  
   - Connect output to Generate HTML node

5. **Create Code Node (Generate HTML) for Email Content**  
   - Type: Code (JavaScript)  
   - Parameters: Use the provided script to generate a full HTML email summarizing meetings with styling and safe escaping.  
   - The script aggregates all meetings, formats times, organizes attendees by required/optional, and builds an HTML email with a subject line including the current date.  
   - Connect output to Send Email node

6. **Create Email Send Node**  
   - Type: Email Send  
   - Credentials: Configure SMTP credentials for your email server  
   - Parameters:  
     - From Email: Set to your sender address (e.g., `notification@yourdomain.com`)  
     - To Email: Set to your recipient email (e.g., your own email)  
     - Subject: Use expression `{{$json.subject}}`  
     - HTML Body: Use expression `{{$json.html}}`  
     - Disable attribution footer  
   - Connect output to end of workflow

7. **Add Sticky Notes (Optional)**  
   - Add sticky notes near the Schedule Trigger and Send Email nodes with reminders to update trigger time and email addresses.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                  |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow triggers daily to provide a convenient morning summary of scheduled meetings.               | High-level workflow purpose                      |
| Email content uses inlined CSS and tables for maximum compatibility across email clients.           | Email HTML generation best practices             |
| Uses Microsoft Outlook OAuth2 credentials, ensure token refresh is configured properly.               | Authentication requirements                       |
| SMTP credentials must support sending from the configured `fromEmail` address.                        | Email sending infrastructure                      |
| Timezone handling depends on the n8n server environment; verify correctness if meetings appear off.  | Potential date/time edge case                      |
| Reminder to update email addresses and trigger time in sticky notes to personalize the workflow.     | Workflow customization instructions               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.