Form to Meeting: Google Calendar, Zoom, Gmail & Slack Booking Automation

https://n8nworkflows.xyz/workflows/form-to-meeting--google-calendar--zoom--gmail---slack-booking-automation-10104


# Form to Meeting: Google Calendar, Zoom, Gmail & Slack Booking Automation

### 1. Workflow Overview

This n8n workflow automates the booking process from form submission to meeting scheduling and team notification. It is designed for scenarios where customers or users submit booking requests via a Google Form, and the system must:

- Validate requested meeting times against a Google Calendar to prevent conflicts.
- If the requested time slot is available:
  - Create a Google Calendar event including the customer as an attendee.
  - Schedule a Zoom meeting linked to the calendar event.
  - Send a confirmation email to the customer with meeting details.
  - Notify a teammate or team channel on Slack with booking information.
- If the requested time slot is unavailable:
  - Send an email to the customer asking to select another time.

**Logical blocks:**

- **1.1 Input Reception:** Receive and parse booking form submissions.
- **1.2 Configuration and Data Extraction:** Normalize webhook data and extract key booking details.
- **1.3 Availability Checking:** Query Google Calendar to check if the requested time slot is free.
- **1.4 Conditional Branch (Availability):** Branch logic based on calendar availability.
- **1.5 Booking Confirmation Flow:** Create calendar event, Zoom meeting, send confirmation email, notify Slack.
- **1.6 Unavailability Notification:** Send email informing the customer that the slot is taken.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives incoming POST requests from Google Forms submissions and merges the data for downstream processing.

- **Nodes Involved:**  
  - Google Forms Webhook1  
  - Merge Form Data1

- **Node Details:**

  - **Google Forms Webhook1**  
    - Type: Webhook node  
    - Role: Entry point for booking requests; receives POST submissions at path `/google-forms-booking`.  
    - Config: HTTP method POST, response mode set to last node output.  
    - Inputs: External HTTP POST from Google Forms.  
    - Outputs: Raw JSON payload containing form fields under `body.*`.  
    - Edge cases: Invalid payloads, incorrect HTTP method, missing fields.

  - **Merge Form Data1**  
    - Type: Merge node  
    - Role: Passes through incoming data, preparing it for the next step.  
    - Config: Default merge parameters (simple pass-through).  
    - Inputs: From Google Forms Webhook1.  
    - Outputs: Unmodified form data.  
    - Edge cases: None significant, acts as a simple relay.

#### 1.2 Configuration and Data Extraction

- **Overview:**  
  This block normalizes form data, sets configuration parameters, and extracts booking details including date/time conversion to ISO strings.

- **Nodes Involved:**  
  - Workflow Configuration1  
  - Extract Booking Details1

- **Node Details:**

  - **Workflow Configuration1**  
    - Type: Set node  
    - Role: Defines reusable variables such as the calendar ID, Slack channel ID, and teammate email.  
    - Config: Sets fixed string values for `calendarId`, `slackChannel`, and `teammateEmail`.  
    - Inputs: From Merge Form Data1.  
    - Outputs: Single item with configuration fields included.  
    - Edge cases: Misconfigured calendar or channel IDs.

  - **Extract Booking Details1**  
    - Type: Set node  
    - Role: Extracts and formats customer name, email, booking date/time, and converts local Japan time to ISO8601 strings with timezone offset.  
    - Config: Uses expressions to parse `$json.body` fields and creates `bookingStartISO` and `bookingEndISO` (1-hour duration).  
    - Inputs: From Workflow Configuration1.  
    - Outputs: Single item enriched with booking details.  
    - Edge cases: Invalid or missing date/time fields; incorrect time zone assumptions.

#### 1.3 Availability Checking

- **Overview:**  
  Queries the Google Calendar to check if the requested booking period overlaps with existing events.

- **Nodes Involved:**  
  - Check Calendar Availability1

- **Node Details:**

  - **Check Calendar Availability1**  
    - Type: Google Calendar node  
    - Role: Retrieves all events overlapping the requested time slot.  
    - Config: Query between `bookingStartISO` and `bookingEndISO` for the configured calendar ID.  
    - Inputs: From Extract Booking Details1.  
    - Outputs: List of calendar events during the requested slot (empty if free).  
    - Edge cases: API rate limits, authentication errors, empty or malformed responses.

#### 1.4 Conditional Branch (Availability)

- **Overview:**  
  Branches workflow execution depending on whether the time slot is free or occupied.

- **Nodes Involved:**  
  - Is Time Slot Available?1

- **Node Details:**

  - **Is Time Slot Available?1**  
    - Type: If node  
    - Role: Checks if the calendar query returned an empty array (slot free) or not (slot busy).  
    - Config: Condition tests if `$json` (calendar events) is empty.  
    - Inputs: From Check Calendar Availability1.  
    - Outputs:  
      - True branch: slot free  
      - False branch: slot busy  
    - Edge cases: Unexpected data formats; empty responses are critical.

#### 1.5 Booking Confirmation Flow (Slot Free)

- **Overview:**  
  When the slot is free, creates the calendar event and Zoom meeting, sends confirmation email, and notifies teammate on Slack.

- **Nodes Involved:**  
  - Create Calendar Event1  
  - Create Zoom Meeting1  
  - Send Confirmation Email1  
  - Notify Teammate on Slack1

- **Node Details:**

  - **Create Calendar Event1**  
    - Type: Google Calendar node  
    - Role: Creates a calendar event with the customer as an attendee, includes summary and description.  
    - Config: Uses `bookingStartISO` and `bookingEndISO`; event summary includes customer name; description includes customer details.  
    - Inputs: From Is Time Slot Available?1 (true).  
    - Outputs: Event data including attendee info.  
    - Edge cases: Calendar API errors, invalid calendar ID, attendee email invalid.

  - **Create Zoom Meeting1**  
    - Type: Zoom node  
    - Role: Creates a Zoom meeting linked to the booking time with 60-minute duration.  
    - Config: Topic includes customer name; start time uses `bookingStartISO`; authentication via OAuth2.  
    - Inputs: From Create Calendar Event1.  
    - Outputs: Zoom meeting info including `join_url`.  
    - Edge cases: Zoom API errors, OAuth token expiry, invalid time formats.

  - **Send Confirmation Email1**  
    - Type: Gmail node  
    - Role: Sends a confirmation email to the customer with booking date/time and Zoom join URL.  
    - Config: Email subject and body use localized Japan time (`Asia/Tokyo`) display; pulls Zoom join URL from previous node output.  
    - Inputs: From Create Zoom Meeting1.  
    - Outputs: Email sent confirmation.  
    - Edge cases: SMTP errors, invalid customer email, template expression failures.

  - **Notify Teammate on Slack1**  
    - Type: Slack node  
    - Role: Sends a formatted Slack message to a configured channel notifying of the new booking with customer and Zoom details.  
    - Config: Uses OAuth2 authentication; channel ID from configuration; message includes localized date/time and Zoom link.  
    - Inputs: From Send Confirmation Email1.  
    - Outputs: Slack message post confirmation.  
    - Edge cases: Slack API errors, invalid channel ID, OAuth token issues.

#### 1.6 Unavailability Notification (Slot Busy)

- **Overview:**  
  Sends an email to the customer informing them that the selected time slot is already booked.

- **Nodes Involved:**  
  - Send Unavailable Email1

- **Node Details:**

  - **Send Unavailable Email1**  
    - Type: Gmail node  
    - Role: Sends an email to the customer informing that the requested time slot is unavailable and requests a new booking time.  
    - Config: Uses localized Japan time for readability; simple message template.  
    - Inputs: From Is Time Slot Available?1 (false).  
    - Outputs: Email sent confirmation.  
    - Edge cases: Email sending failures, invalid email address, expression errors.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                               | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                   |
|---------------------------|-------------------|-----------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Google Forms Webhook1      | Webhook           | Receives booking submissions from Google Form | External HTTP POST           | Merge Form Data1             | Title: Automated Booking → Calendar + Zoom + Email + Slack. Captures booking, parses date/time, checks calendar, branches on availability. |
| Merge Form Data1           | Merge             | Passes through form data for processing       | Google Forms Webhook1        | Workflow Configuration1      | Purpose: Entry point for bookings. Receives POST payload (name, email, date, time). Outputs raw form fields.  |
| Workflow Configuration1    | Set               | Sets config variables (calendarId, Slack channel, teammate email) | Merge Form Data1             | Extract Booking Details1     | Purpose: Normalizes/forwards webhook data; acts as buffer for single item flow.                              |
| Extract Booking Details1   | Set               | Extracts and formats booking details, converts date/time to ISO strings | Workflow Configuration1      | Check Calendar Availability1 | Purpose: Maps form fields to variables; builds ISO start/end times with JST to ISO conversion.               |
| Check Calendar Availability1 | Google Calendar | Queries calendar for events overlapping booking time | Extract Booking Details1     | Is Time Slot Available?1     | Purpose: Queries calendar for events overlapping bookingStartISO–bookingEndISO; empty if free.               |
| Is Time Slot Available?1   | If                | Branches based on calendar availability       | Check Calendar Availability1 | Create Calendar Event1, Send Unavailable Email1 | Purpose: Branch on availability; empty = slot free, else busy.                                              |
| Create Calendar Event1     | Google Calendar   | Creates calendar event with guest as attendee | Is Time Slot Available?1 (true) | Create Zoom Meeting1         | Purpose: Creates event with guest; summary and description include guest info.                               |
| Create Zoom Meeting1       | Zoom              | Creates Zoom meeting for booking time          | Create Calendar Event1       | Send Confirmation Email1     | Purpose: Creates Zoom meeting with topic, startTime, 60-minute duration.                                    |
| Send Confirmation Email1   | Gmail             | Sends confirmation email with meeting details  | Create Zoom Meeting1         | Notify Teammate on Slack1    | Purpose: Sends email with localized date/time and Zoom join URL.                                            |
| Notify Teammate on Slack1  | Slack             | Notifies team channel of new booking            | Send Confirmation Email1     | —                           | Purpose: Posts booking summary to Slack with guest info and Zoom link.                                     |
| Send Unavailable Email1    | Gmail             | Sends unavailability email to customer          | Is Time Slot Available?1 (false) | —                           | Purpose: Notifies guest slot is taken, asks to rebook another time.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node:**  
   - Name: `Google Forms Webhook1`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `google-forms-booking`  
   - Response Mode: Last node output  
   - Purpose: Receives booking submissions from Google Forms.

2. **Create a Merge Node:**  
   - Name: `Merge Form Data1`  
   - Type: Merge  
   - Connect input from `Google Forms Webhook1`  
   - Default parameters (simple pass-through).  
   - Purpose: Pass form data downstream.

3. **Create a Set Node for Configuration:**  
   - Name: `Workflow Configuration1`  
   - Type: Set  
   - Connect input from `Merge Form Data1`  
   - Add fields:  
     - `calendarId` (string): Your Google Calendar ID (e.g., `c_...@group.calendar.google.com`)  
     - `slackChannel` (string): Slack channel ID for notifications (e.g., `C09GN4NCTAP`)  
     - `teammateEmail` (string): Email address of teammate for notifications  
   - Purpose: Centralized configuration variables.

4. **Create a Set Node to Extract Booking Details:**  
   - Name: `Extract Booking Details1`  
   - Type: Set  
   - Connect input from `Workflow Configuration1`  
   - Add fields with expressions:  
     - `customerName`: `={{ $json.body.name }}`  
     - `customerEmail`: `={{ $json.body.email }}`  
     - `bookingDate`: `={{ $json.body.date }}`  
     - `bookingTime`: `={{ $json.body.time }}`  
     - `bookingStartISO`: `={{ new Date($json.body.date + 'T' + $json.body.time + ':00+09:00').toISOString() }}`  
     - `bookingEndISO`: `={{ new Date(new Date($json.body.date + 'T' + $json.body.time + ':00+09:00').getTime() + 60*60*1000).toISOString() }}`  
   - Purpose: Extract and convert booking times from form data.

5. **Create a Google Calendar Node to Check Availability:**  
   - Name: `Check Calendar Availability1`  
   - Type: Google Calendar  
   - Connect input from `Extract Booking Details1`  
   - Operation: Get All Events  
   - Calendar: Use `calendarId` from `Workflow Configuration1`  
   - Time Min: `={{ $json.bookingStartISO }}`  
   - Time Max: `={{ $json.bookingEndISO }}`  
   - Purpose: Check for existing events overlapping booking time.

6. **Create an If Node to Branch on Availability:**  
   - Name: `Is Time Slot Available?1`  
   - Type: If  
   - Connect input from `Check Calendar Availability1`  
   - Condition: Check if `$json` (calendar events) is empty (slot free)  
   - True output: Slot free branch  
   - False output: Slot busy branch

7. **Create Google Calendar Node to Create Event (Free Slot):**  
   - Name: `Create Calendar Event1`  
   - Type: Google Calendar  
   - Connect input from `Is Time Slot Available?1` (true)  
   - Operation: Create Event  
   - Calendar: Use `calendarId` from `Workflow Configuration1`  
   - Start: `={{ $('Extract Booking Details1').first().json.bookingStartISO }}`  
   - End: `={{ $('Extract Booking Details1').first().json.bookingEndISO }}`  
   - Summary: `=Booking with {{ $('Extract Booking Details1').first().json.customerName }}`  
   - Attendees: `={{ [$('Extract Booking Details1').first().json.customerEmail] }}`  
   - Description: `=Customer: {{ $('Extract Booking Details1').first().json.customerName }} ({{ $('Extract Booking Details1').first().json.customerEmail }})`  
   - Purpose: Schedule calendar event with customer attendee.

8. **Create Zoom Node to Create Meeting:**  
   - Name: `Create Zoom Meeting1`  
   - Type: Zoom  
   - Connect input from `Create Calendar Event1`  
   - Authentication: OAuth2 (configure credentials for Zoom)  
   - Topic: `=Booking with {{ $('Extract Booking Details1').first().json.customerName }}`  
   - Start Time: `={{ $('Extract Booking Details1').first().json.bookingStartISO }}`  
   - Duration: 60 minutes  
   - Time Zone: UTC (Zoom expects UTC ISO string)  
   - Purpose: Create Zoom meeting for booking.

9. **Create Gmail Node to Send Confirmation Email:**  
   - Name: `Send Confirmation Email1`  
   - Type: Gmail  
   - Connect input from `Create Zoom Meeting1`  
   - Send To: `={{ $('Extract Booking Details1').first().json.customerEmail }}`  
   - Subject: `=Booking Confirmed: {{ new Date($('Extract Booking Details1').first().json.bookingStartISO).toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' }) }}`  
   - Message:  
     ```
     Hi {{ $('Extract Booking Details1').first().json.customerName }},

     Your booking has been confirmed!

     Date & Time: {{ new Date($('Extract Booking Details1').first().json.bookingStartISO).toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' }) }}

     Zoom Meeting Link: {{ $json.join_url }}

     Looking forward to meeting you!

     Best regards
     ```  
   - Purpose: Send booking confirmation with Zoom link.

10. **Create Slack Node to Notify Teammate:**  
    - Name: `Notify Teammate on Slack1`  
    - Type: Slack  
    - Connect input from `Send Confirmation Email1`  
    - Authentication: OAuth2 (configure Slack credentials)  
    - Channel: Use `slackChannel` from `Workflow Configuration1`  
    - Message:  
      ```
      🎉 New booking confirmed!

      *Customer:* {{ $('Extract Booking Details1').first().json.customerName }} ({{ $('Extract Booking Details1').first().json.customerEmail }})
      *Date & Time:* {{ new Date($('Extract Booking Details1').first().json.bookingStartISO).toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' }) }}
      *Zoom Link:* {{ $json.join_url }}
      ```  
    - Purpose: Notify team of new booking.

11. **Create Gmail Node to Send Unavailable Email (Busy Slot):**  
    - Name: `Send Unavailable Email1`  
    - Type: Gmail  
    - Connect input from `Is Time Slot Available?1` (false)  
    - Send To: `={{ $('Extract Booking Details1').first().json.customerEmail }}`  
    - Subject: `Time Slot Unavailable`  
    - Message:  
      ```
      Hi {{ $('Extract Booking Details1').first().json.customerName }},

      Thank you for your booking request. Unfortunately, the requested time slot on {{ new Date($('Extract Booking Details1').first().json.bookingStartISO).toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' }) }} is not available.

      Please submit a new booking request with an alternative time.
      ```  
    - Purpose: Inform customer slot is busy and request a new booking.

12. **Credentials Setup:**  
    - Google Calendar: OAuth2 credentials with access to the target calendar.  
    - Zoom: OAuth2 credentials with permissions to create meetings.  
    - Gmail: OAuth2 or app password credentials to send emails.  
    - Slack: OAuth2 credentials with permission to post messages in the specified channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow assumes input times are in Asia/Tokyo timezone and converts them to ISO8601 for API compatibility.                                                                                                                   | Timezone handling critical for accurate scheduling.                                                |
| All user-editable variables (calendarId, slackChannel, teammateEmail) are centralized in the Set node (`Workflow Configuration1`) for easy maintenance.                                                                            | Simplifies updates without node-by-node edits.                                                    |
| Credentials must be stored securely in n8n and referenced by each node requiring authentication (no hardcoded secrets in nodes).                                                                                                    | Security best practices for API keys and OAuth tokens.                                            |
| Slack messages use rich formatting and mention the Slack channel by ID. Ensure the bot has necessary permissions.                                                                                                                  | Slack API setup required.                                                                          |
| Zoom meeting creation uses OAuth2 and requires API scopes for meeting management.                                                                                                                                                   | Zoom OAuth2 credential setup needed.                                                              |
| Email templates use JavaScript expressions to format dates localized for Japan using `toLocaleString('ja-JP', { timeZone: 'Asia/Tokyo' })`.                                                                                        | Ensures user-friendly datetime display.                                                          |
| The workflow branches explicitly on calendar availability by checking if the returned events list is empty, which is essential to avoid double bookings.                                                                            | Critical logical check to prevent scheduling conflicts.                                           |
| The Slack and Email nodes depend on the Zoom meeting creation node output to include the `join_url` link. This requires the Zoom API to respond without errors for the notification steps to function properly.                      | Potential failure point if Zoom API call fails.                                                   |
| For advanced customization, consider adding error handling nodes after external API calls to catch auth errors or timeouts and notify administrators.                                                                              | Improves robustness and operational visibility.                                                  |
| Workflow tested with n8n version supporting Google Calendar v1.3, Zoom node v1, Gmail node v2.1, Slack node v2.3, Webhook v2.1, and Set node v3.4.                                                                                | Version compatibility note.                                                                       |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.