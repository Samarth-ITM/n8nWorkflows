Automated Event Management with Typeform, Stripe, Google Tools & Slack Notifications

https://n8nworkflows.xyz/workflows/automated-event-management-with-typeform--stripe--google-tools---slack-notifications-10117


# Automated Event Management with Typeform, Stripe, Google Tools & Slack Notifications

### 1. Workflow Overview

This workflow automates event participant management and communication for event organizers using Typeform, Stripe, Google Workspace tools, and Slack. It handles participant registration, payment processing, event scheduling, reminders, and post-event follow-up with surveys.

The workflow is logically divided into three main blocks:

- **1.1 Registration and Onboarding:**  
  Captures participant data from a Typeform registration form, appends or updates participant information in Google Sheets, processes payment via Stripe, sends confirmation emails upon successful payment, adds the event to the participant’s Google Calendar, and notifies the organizer on Slack.

- **1.2 Event Reminders:**  
  On a daily schedule, retrieves upcoming events from Google Sheets with confirmed payments, calculates if reminder emails should be sent based on configuration, and sends reminder emails to participants accordingly.

- **1.3 Post-Event Follow-up:**  
  On a daily schedule, retrieves past events from Google Sheets that have confirmed payments but no follow-up sent, calculates if follow-up emails are due, sends thank-you and survey invitation emails, and updates the Google Sheet to mark the follow-up as sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Registration and Onboarding

**Overview:**  
Manages new event registrations by collecting participant data, updating records, processing payments, confirming registrations via email, adding events to calendars, and notifying organizers.

**Nodes Involved:**  
- Typeform Registration Form  
- Workflow Configuration  
- Add to Participant List (Google Sheets)  
- Process Payment (Stripe)  
- Check Payment Status (If)  
- Send Confirmation Email (Gmail)  
- Add to Calendar (Google Calendar)  
- Notify Organizer (Slack)  
- Sticky Note1 (Documentation)

**Node Details:**

- **Typeform Registration Form**  
  - *Type:* Trigger node (Typeform Trigger)  
  - *Role:* Receives new participant registration submissions from a specified Typeform form.  
  - *Configuration:* Uses a webhook linked to a specific Typeform form ID (placeholder to be replaced).  
  - *Input:* Incoming webhook data from Typeform.  
  - *Output:* Participant form data (e.g., name, email, phone).  
  - *Potential Failures:* Webhook misconfiguration, invalid form ID, connectivity issues.

- **Workflow Configuration**  
  - *Type:* Set node  
  - *Role:* Defines static workflow parameters such as event details, fees, reminder and follow-up timing, and Slack channel ID.  
  - *Configuration:* Assigns placeholders for event name, date, time, location, participation fee, reminder and follow-up days, and Slack channel.  
  - *Input:* Data from Typeform node.  
  - *Output:* JSON object with event configuration values.  
  - *Potential Failures:* Missing or incorrect placeholder values leading to misconfigurations.

- **Add to Participant List**  
  - *Type:* Google Sheets node  
  - *Role:* Appends or updates participant details in a Google Sheet for event tracking.  
  - *Configuration:* Maps participant info (name, email, phone), event details from configuration, sets payment status as "Pending," and logs registration date/time. Uses email as matching column for updates.  
  - *Input:* Data from Workflow Configuration and Typeform node.  
  - *Output:* Confirmation of sheet update.  
  - *Credentials:* Google Sheets OAuth2 credentials required.  
  - *Potential Failures:* Authentication errors, sheet ID or name misconfiguration, rate limits.

- **Process Payment**  
  - *Type:* Stripe node  
  - *Role:* Charges participant for event participation fee.  
  - *Configuration:* Creates a charge with amount derived from participation fee (converted to cents), currency set to JPY, customer and source IDs placeholders, metadata with participant info, and charge description.  
  - *Input:* Participant data from Google Sheets node.  
  - *Output:* Payment charge data including status.  
  - *Credentials:* Stripe API credentials required.  
  - *Potential Failures:* Payment failure (insufficient funds, invalid source), authentication errors, network issues.

- **Check Payment Status**  
  - *Type:* If node  
  - *Role:* Checks if payment status equals "succeeded" to proceed.  
  - *Configuration:* Condition compares payment status string precisely.  
  - *Input:* Stripe charge data.  
  - *Output:* Routes flow based on payment success or failure.  
  - *Potential Failures:* Unexpected or missing payment status field.

- **Send Confirmation Email**  
  - *Type:* Gmail node  
  - *Role:* Sends confirmation email to participant upon successful payment.  
  - *Configuration:* Uses participant email from Typeform data, composes HTML email with participant name, event details (date, time, location), and subject referencing event name.  
  - *Input:* Data from Check Payment Status node on success path.  
  - *Output:* Email sent confirmation.  
  - *Credentials:* Gmail OAuth2 credentials required.  
  - *Potential Failures:* Authentication errors, invalid email addresses, Gmail API quota exceeded.

- **Add to Calendar**  
  - *Type:* Google Calendar node  
  - *Role:* Adds event to participant’s Google Calendar with start/end times, location, and description including participant name.  
  - *Configuration:* Event start time from event date/time config; end time set 2 hours later. Uses primary calendar.  
  - *Input:* Data from Send Confirmation Email node.  
  - *Output:* Calendar event created confirmation.  
  - *Credentials:* Google Calendar OAuth2 credentials required.  
  - *Potential Failures:* Invalid date/time formats, calendar permission errors.

- **Notify Organizer**  
  - *Type:* Slack node  
  - *Role:* Sends notification message to organizer’s Slack channel about new registration.  
  - *Configuration:* Message includes participant name, email, and event name; posts to configured Slack channel ID. OAuth2 authentication.  
  - *Input:* Data from Add to Calendar node.  
  - *Output:* Slack message sent confirmation.  
  - *Credentials:* Slack OAuth2 credentials required.  
  - *Potential Failures:* Slack API permission issues, invalid channel ID.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Provides summary documentation for Registration and Onboarding block.  
  - *Content:* Describes the flow of registration, payment, confirmation, calendar addition, and organizer notification.

---

#### 2.2 Event Reminders

**Overview:**  
On a daily schedule, checks for upcoming events with paid participants, calculates reminder dates, verifies if reminders should be sent today, and sends reminder emails accordingly.

**Nodes Involved:**  
- Daily Reminder Check (Schedule Trigger)  
- Get Upcoming Events (Google Sheets)  
- Calculate Reminder Date (DateTime)  
- Check If Reminder Needed (If)  
- Send Reminder Email (Gmail)  
- Sticky Note3 (Documentation)

**Node Details:**

- **Daily Reminder Check**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers workflow daily at 9 AM to initiate the reminder process.  
  - *Configuration:* Fixed hourly trigger at 9:00.  
  - *Potential Failures:* Workflow execution failures or missed triggers due to downtime.

- **Get Upcoming Events**  
  - *Type:* Google Sheets node  
  - *Role:* Retrieves participant rows with payment status "succeeded" from Google Sheet.  
  - *Configuration:* Filters rows where "Payment Status" = "succeeded". Retrieves all matches, not just first.  
  - *Input:* Trigger from Daily Reminder Check.  
  - *Output:* Participant event records.  
  - *Credentials:* Google Sheets OAuth2 required.  
  - *Potential Failures:* Authentication errors, misconfigured sheet ID/name.

- **Calculate Reminder Date**  
  - *Type:* DateTime node  
  - *Role:* Calculates the reminder date by subtracting configured number of days (default 3) from the event date.  
  - *Configuration:* Uses `reminderDaysBefore` from Workflow Configuration and event’s "Event Date" field.  
  - *Input:* Participant event data from Google Sheets.  
  - *Output:* Date value representing when to send reminder.  
  - *Potential Failures:* Invalid date formats, missing event date.

- **Check If Reminder Needed**  
  - *Type:* If node  
  - *Role:* Compares the current date (`$now`) to the calculated reminder date; proceeds if dates match.  
  - *Configuration:* Checks if today’s date string equals the reminder date.  
  - *Input:* Calculated reminder date.  
  - *Output:* Routes only participants needing reminder today.  
  - *Potential Failures:* Time zone mismatches, incorrect date formatting.

- **Send Reminder Email**  
  - *Type:* Gmail node  
  - *Role:* Sends a reminder email to participants whose event is approaching.  
  - *Configuration:* Uses participant email and name, event name, date, time, and location from configuration; email subject references event name.  
  - *Input:* Participants filtered by If node.  
  - *Output:* Email sent confirmation.  
  - *Credentials:* Gmail OAuth2 credentials required.  
  - *Potential Failures:* Email delivery failures, invalid addresses.

- **Sticky Note3**  
  - *Type:* Sticky Note  
  - *Role:* Documents the Event Reminders block explaining its scheduled trigger and email sending logic.

---

#### 2.3 Post-Event Follow-up

**Overview:**  
On a daily schedule, identifies past events with paid participants who have not yet received follow-up emails, calculates follow-up due date, sends thank-you and survey emails, and updates Google Sheets to mark follow-up as sent.

**Nodes Involved:**  
- Daily Follow-up Check (Schedule Trigger)  
- Get Past Events (Google Sheets)  
- Calculate Follow-up Date (DateTime)  
- Check If Follow-up Needed (If)  
- Send Thank You & Survey (Gmail)  
- Update Follow-up Status (Google Sheets)  
- Sticky Note2 (Documentation)

**Node Details:**

- **Daily Follow-up Check**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers workflow daily at 10 AM to start follow-up process.  
  - *Configuration:* Fixed hourly trigger at 10:00.  
  - *Potential Failures:* Workflow execution failures or missed triggers.

- **Get Past Events**  
  - *Type:* Google Sheets node  
  - *Role:* Retrieves participants with payment status "succeeded" and no follow-up sent.  
  - *Configuration:* Filters rows where "Payment Status" = "succeeded" and "Follow-up Sent" is empty or null.  
  - *Input:* Trigger from Daily Follow-up Check.  
  - *Output:* Participant event records pending follow-up.  
  - *Credentials:* Google Sheets OAuth2 credentials required.  
  - *Potential Failures:* Authentication errors, sheet misconfiguration.

- **Calculate Follow-up Date**  
  - *Type:* DateTime node  
  - *Role:* Calculates the follow-up date by adding configured days (default 2) after the event date.  
  - *Configuration:* Uses `followupDaysAfter` from Workflow Configuration and event’s "Event Date" field.  
  - *Input:* Participant event data.  
  - *Output:* Date representing when follow-up should be sent.  
  - *Potential Failures:* Date parsing errors, missing event date.

- **Check If Follow-up Needed**  
  - *Type:* If node  
  - *Role:* Checks if current date is greater than or equal to follow-up date, indicating follow-up is due.  
  - *Configuration:* Compares `$now` with calculated follow-up date.  
  - *Input:* Follow-up date data.  
  - *Output:* Filters participants due for follow-up email.  
  - *Potential Failures:* Time zone or date comparison issues.

- **Send Thank You & Survey**  
  - *Type:* Gmail node  
  - *Role:* Sends a thank-you email with a survey link to participants.  
  - *Configuration:* Uses participant email and name, event name, and a placeholder URL for survey. Subject references event name.  
  - *Input:* Participants filtered by If node.  
  - *Output:* Email sent confirmation.  
  - *Credentials:* Gmail OAuth2 required.  
  - *Potential Failures:* Email sending errors, invalid survey URL.

- **Update Follow-up Status**  
  - *Type:* Google Sheets node  
  - *Role:* Updates participant row in Google Sheet to mark follow-up as sent with current timestamp.  
  - *Configuration:* Matches row by email; sets "Follow-up Sent" to "Yes" and "Follow-up Date" to current datetime.  
  - *Input:* After sending follow-up email.  
  - *Output:* Sheet updated confirmation.  
  - *Credentials:* Google Sheets OAuth2 required.  
  - *Potential Failures:* Update conflicts, authentication issues.

- **Sticky Note2**  
  - *Type:* Sticky Note  
  - *Role:* Describes the Post-Event Follow-up block functionality.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                        | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                         |
|----------------------------|-------------------------|-------------------------------------|--------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Typeform Registration Form | Typeform Trigger        | Receives participant registrations | —                              | Workflow Configuration          |                                                                                                                     |
| Workflow Configuration      | Set                     | Defines event and workflow settings | Typeform Registration Form      | Add to Participant List         |                                                                                                                     |
| Add to Participant List     | Google Sheets           | Stores participant data             | Workflow Configuration          | Process Payment                 |                                                                                                                     |
| Process Payment             | Stripe                  | Charges participant payment         | Add to Participant List         | Check Payment Status            |                                                                                                                     |
| Check Payment Status        | If                      | Checks payment success              | Process Payment                 | Send Confirmation Email         |                                                                                                                     |
| Send Confirmation Email     | Gmail                   | Sends registration confirmation    | Check Payment Status            | Add to Calendar                |                                                                                                                     |
| Add to Calendar             | Google Calendar         | Adds event to participant calendar | Send Confirmation Email         | Notify Organizer               |                                                                                                                     |
| Notify Organizer            | Slack                   | Notifies organizer of registration | Add to Calendar                | —                              |                                                                                                                     |
| Daily Reminder Check        | Schedule Trigger        | Triggers daily reminder workflow    | —                              | Get Upcoming Events             |                                                                                                                     |
| Get Upcoming Events         | Google Sheets           | Retrieves upcoming paid participants| Daily Reminder Check            | Calculate Reminder Date         |                                                                                                                     |
| Calculate Reminder Date     | DateTime                | Calculates reminder send date       | Get Upcoming Events             | Check If Reminder Needed        |                                                                                                                     |
| Check If Reminder Needed    | If                      | Checks if reminder should be sent   | Calculate Reminder Date         | Send Reminder Email             |                                                                                                                     |
| Send Reminder Email         | Gmail                   | Sends reminder email to participants| Check If Reminder Needed        | —                              |                                                                                                                     |
| Daily Follow-up Check       | Schedule Trigger        | Triggers daily follow-up workflow   | —                              | Get Past Events                |                                                                                                                     |
| Get Past Events             | Google Sheets           | Retrieves past participants needing follow-up | Daily Follow-up Check    | Calculate Follow-up Date        |                                                                                                                     |
| Calculate Follow-up Date    | DateTime                | Calculates follow-up send date      | Get Past Events                | Check If Follow-up Needed       |                                                                                                                     |
| Check If Follow-up Needed   | If                      | Checks if follow-up is due          | Calculate Follow-up Date        | Send Thank You & Survey         |                                                                                                                     |
| Send Thank You & Survey     | Gmail                   | Sends thank-you and survey email    | Check If Follow-up Needed       | Update Follow-up Status         |                                                                                                                     |
| Update Follow-up Status     | Google Sheets           | Marks follow-up as sent             | Send Thank You & Survey         | —                              |                                                                                                                     |
| Sticky Note                 | Sticky Note             | Documentation overview              | —                              | —                              | See detailed notes in Section 5                                                                                    |
| Sticky Note1                | Sticky Note             | Registration and Onboarding notes   | —                              | —                              | Describes Registration and Onboarding block                                                                        |
| Sticky Note2                | Sticky Note             | Post-Event Follow-up notes          | —                              | —                              | Describes Post-Event Follow-up block                                                                               |
| Sticky Note3                | Sticky Note             | Event Reminders notes               | —                              | —                              | Describes Event Reminders block                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger Node**  
   - Type: Typeform Trigger  
   - Configure webhook with your Typeform form ID replacing `<__PLACEHOLDER_VALUE__Typeform Form ID__>`.  
   - This node starts the workflow upon new registrations.

2. **Create Set Node named "Workflow Configuration"**  
   - Provide static values for:  
     - `eventName` (string)  
     - `eventDate` (string, YYYY-MM-DD)  
     - `eventTime` (string, e.g., "18:00")  
     - `eventLocation` (string)  
     - `participationFee` (string representing amount in JPY)  
     - `reminderDaysBefore` (number, default 3)  
     - `followupDaysAfter` (number, default 2)  
     - `organizerSlackChannel` (string Slack channel ID)  
   - Connect output of Typeform Trigger to this node.

3. **Create Google Sheets Node "Add to Participant List"**  
   - Operation: appendOrUpdate  
   - Document ID and Sheet Name: replace placeholders with your Google Sheet details.  
   - Mapping columns: Name, Email (used as match for update), Phone, Registration Date (use current datetime), Event Name, Event Date, Payment Status ("Pending" initially).  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Connect output of Workflow Configuration node here.

4. **Create Stripe Node "Process Payment"**  
   - Operation: Create charge  
   - Amount: participationFee * 100 (to convert to smallest currency unit)  
   - Currency: "jpy" (adjust if needed)  
   - Customer ID and Source ID: replace placeholders with actual Stripe customer and source IDs.  
   - Include metadata with participant email, name, and event name.  
   - Authenticate with Stripe API credentials.  
   - Connect output of Add to Participant List node.

5. **Create If Node "Check Payment Status"**  
   - Condition: Check if payment status equals "succeeded".  
   - Connect output of Process Payment node.

6. **Create Gmail Node "Send Confirmation Email"**  
   - Send to participant email from Typeform data.  
   - Compose HTML message with participant name, event name, date, time, and location from Workflow Configuration.  
   - Subject: includes event name.  
   - Authenticate with Gmail OAuth2 credentials.  
   - Connect true output of If node.

7. **Create Google Calendar Node "Add to Calendar"**  
   - Calendar: primary  
   - Start: Combine eventDate and eventTime from Workflow Configuration as ISO string.  
   - End: Add 2 hours to start time.  
   - Summary: eventName  
   - Location: eventLocation  
   - Description: includes participant name.  
   - Authenticate with Google Calendar OAuth2 credentials.  
   - Connect output of Send Confirmation Email node.

8. **Create Slack Node "Notify Organizer"**  
   - Message text: includes participant name, email, event name.  
   - Channel: organizerSlackChannel from Workflow Configuration.  
   - Authentication: Slack OAuth2 credentials.  
   - Connect output of Add to Calendar node.

9. **Create Schedule Trigger Node "Daily Reminder Check"**  
   - Trigger time: daily at 9:00 AM.

10. **Create Google Sheets Node "Get Upcoming Events"**  
    - Document ID and Sheet Name same as participant list.  
    - Filter rows where "Payment Status" = "succeeded".  
    - Authenticate with Google Sheets OAuth2.  
    - Connect output of Daily Reminder Check.

11. **Create DateTime Node "Calculate Reminder Date"**  
    - Operation: subtract days  
    - Duration: reminderDaysBefore from Workflow Configuration  
    - Date input: event date from Google Sheets data.  
    - Connect output of Get Upcoming Events.

12. **Create If Node "Check If Reminder Needed"**  
    - Condition: Check if today’s date equals calculated reminder date.  
    - Connect output of Calculate Reminder Date.

13. **Create Gmail Node "Send Reminder Email"**  
    - Send to participant email from Google Sheets data.  
    - Message includes participant name, event details, location, and reminder subject.  
    - Authenticate with Gmail OAuth2.  
    - Connect true output of If node.

14. **Create Schedule Trigger Node "Daily Follow-up Check"**  
    - Trigger time: daily at 10:00 AM.

15. **Create Google Sheets Node "Get Past Events"**  
    - Document ID and Sheet Name same as participant list.  
    - Filter rows where "Payment Status" = "succeeded" and "Follow-up Sent" is empty or null.  
    - Authenticate with Google Sheets OAuth2.  
    - Connect output of Daily Follow-up Check.

16. **Create DateTime Node "Calculate Follow-up Date"**  
    - Operation: add days  
    - Duration: followupDaysAfter from Workflow Configuration  
    - Date input: event date from Google Sheets data.  
    - Connect output of Get Past Events.

17. **Create If Node "Check If Follow-up Needed"**  
    - Condition: Check if today’s date is greater or equal to follow-up date.  
    - Connect output of Calculate Follow-up Date.

18. **Create Gmail Node "Send Thank You & Survey"**  
    - Send to participant email from Google Sheets data.  
    - Message includes thank you note, participant name, event name, and survey link placeholder.  
    - Subject includes event name.  
    - Authenticate with Gmail OAuth2.  
    - Connect true output of If node.

19. **Create Google Sheets Node "Update Follow-up Status"**  
    - Operation: update  
    - Matches participant by email.  
    - Sets "Follow-up Sent" to "Yes" and "Follow-up Date" to current datetime.  
    - Authenticate with Google Sheets OAuth2.  
    - Connect output of Send Thank You & Survey node.

20. **Add Sticky Notes**  
    - Add three sticky notes with descriptions of each block (Registration and Onboarding, Event Reminders, Post-Event Follow-up) for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for event organizers, community managers, and businesses hosting workshops, webinars, or conferences requiring participant registration, payment processing, reminders, and follow-ups. It integrates Typeform, Stripe, Google Sheets, Gmail, Google Calendar, and Slack. Please replace all placeholder values (e.g., form IDs, Google Sheet IDs, Stripe customer/source IDs, Slack channel ID, and survey URL) before activation. Credentials must be properly set up for all integrated services. Adjust reminder and follow-up timing by modifying the Workflow Configuration node. Email templates can be customized in respective Gmail nodes. | Workflow overview and setup instructions included in Sticky Note (Node "Sticky Note") in the workflow.          |
| For Stripe integration, ensure you have valid customer and source IDs. Payment currency is currently set to Japanese Yen (JPY) but can be changed in the Process Payment node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Stripe integration details.                                                                                       |
| Google Sheets must contain columns "Name", "Email", "Phone", "Event Date", "Event Name", "Payment Status", "Registration Date", "Follow-up Sent", and "Follow-up Date" for data mapping and filtering to work correctly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Google Sheets schema and required columns.                                                                        |
| Gmail OAuth2 credentials must allow sending emails on behalf of the user. Gmail nodes use HTML content for email body, which can be customized for branding and messaging.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Gmail integration notes.                                                                                           |
| Google Calendar events are created on the primary calendar with a default duration of two hours. Adjust time settings in the Add to Calendar node if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Google Calendar event creation details.                                                                            |
| Slack notifications require OAuth2 credentials with permission to post messages in the specified channel. Replace the Slack channel ID in Workflow Configuration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Slack integration notes.                                                                                           |
| Survey URL placeholder must be replaced with the actual post-event survey link in the "Send Thank You & Survey" node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Post-event survey integration.                                                                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.