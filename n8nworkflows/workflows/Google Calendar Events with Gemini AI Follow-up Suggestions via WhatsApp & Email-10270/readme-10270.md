Google Calendar Events with Gemini AI Follow-up Suggestions via WhatsApp & Email

https://n8nworkflows.xyz/workflows/google-calendar-events-with-gemini-ai-follow-up-suggestions-via-whatsapp---email-10270


# Google Calendar Events with Gemini AI Follow-up Suggestions via WhatsApp & Email

---
### 1. Workflow Overview

This n8n workflow automates the management of meeting schedules and follow-ups, focusing on two main processes:

- **1.1 Create New Google Calendar Events:**  
  Periodically reads meeting requests from a Google Sheet, formats the datetime details, creates corresponding events in a designated Google Calendar, sends confirmation messages via WhatsApp and Email, and updates the sheet status to mark completion.

- **1.2 AI-Powered Follow-up Meeting Suggestions:**  
  On a separate schedule, fetches recent past events from Google Calendar, filters for those needing follow-up, uses an AI Meeting Agent (Google Gemini-powered LangChain agent) to propose suitable future meeting slots based on previous meeting details and calendar availability, formats these suggestions into human-readable messages, and sends them via WhatsApp and Email.

Logical blocks are distinguished by their input sources and targets, trigger schedules, and communication channels:

- **Block A: Create Event from Google Sheet Data**  
  *(Schedule Trigger → Google Sheets → Date Formatting → Event Creation → Notifications → Sheet Update)*

- **Block B: Reminder Event with AI Follow-up Proposals**  
  *(Schedule Trigger → Google Calendar Past Events → Deduplication → Filtering → AI Meeting Agent (Gemini + Availability) → Message Generation → Notifications)*

Additional utility nodes handle batching and waiting to ensure processing efficiency and reliable messaging timing.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Block A: Create Event from Google Sheet Data

**Overview:**  
This block automates creating new Google Calendar events using meeting data pulled from a Google Sheet. It formats event times, creates the calendar event, sends WhatsApp and Email confirmations, and updates the Google Sheet row to mark the event as processed.

**Nodes Involved:**  
- Schedule Trigger  
- Get in sheet (Google Sheets)  
- Loop Over Items (Split in Batches)  
- Date & Time  
- Code  
- Create an event (Google Calendar)  
- Rapiwa (WhatsApp)  
- Send a message1 (Gmail)  
- Update status in sheet (Google Sheets)  
- Wait  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Trigger node, initiates workflow at regular intervals (every few minutes).  
  - *Role:* Starts event creation process automatically.  
  - *Connections:* Outputs to "Get in sheet".  
  - *Edge Cases:* Ensure interval is appropriate to avoid rate limits or missing new entries.

- **Get in sheet**  
  - *Type:* Google Sheets node, reads rows filtered by "status" from a specific spreadsheet and sheet.  
  - *Role:* Fetches meeting requests that require scheduling.  
  - *Key Config:* Document ID and Sheet Name set to the target spreadsheet; filter by "status" column to select unscheduled events.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Outputs to "Loop Over Items".  
  - *Edge Cases:* Sheet structure changes or auth errors.

- **Loop Over Items**  
  - *Type:* SplitInBatches node, processes rows one by one for scalability.  
  - *Role:* Enables sequential processing of meetings.  
  - *Connections:* Main output to "Date & Time" for formatting; secondary output empty.  
  - *Edge Cases:* Large batches may slow execution; handle empty inputs gracefully.

- **Date & Time**  
  - *Type:* DateTime node, formats the "end time" field from the sheet into a custom string format (`yyyy-MM-dd HH:mm:ss ZZZZ`).  
  - *Role:* Prepares datetime for conversion to ISO by next node.  
  - *Inputs:* Uses `{{ $json['end time'] }}` from sheet data.  
  - *Connections:* Output to "Code".  
  - *Edge Cases:* Invalid date inputs.

- **Code**  
  - *Type:* Code node (JavaScript), converts the formatted date string to a precise UTC ISO 8601 string (`YYYY-MM-DDTHH:mm:ssZ`).  
  - *Role:* Ensures event times are in correct format for Google Calendar API.  
  - *Inputs:* Receives all items from "Date & Time".  
  - *Outputs:* Adds a "formatted" ISO string to each item’s JSON.  
  - *Edge Cases:* Malformed dates; time zone considerations.

- **Create an event**  
  - *Type:* Google Calendar node, creates a new event in the specified calendar.  
  - *Role:* Schedules the meeting on Google Calendar.  
  - *Key Config:* Calendar ID, event summary, description, location, color, reminder email 30 minutes before.  
  - *Inputs:* Uses event details and the ISO formatted end time from previous node.  
  - *Credentials:* Google Calendar OAuth2.  
  - *Connections:* Outputs to "Rapiwa" and "Send a message1".  
  - *Edge Cases:* API quota limits, invalid calendar ID, missing required fields.

- **Rapiwa**  
  - *Type:* Rapiwa node, sends WhatsApp message to client number confirming event creation.  
  - *Role:* Client notification via WhatsApp.  
  - *Key Config:* Phone number, template message with event summary and time.  
  - *Credentials:* Rapiwa API.  
  - *Connections:* No downstream nodes.  
  - *Edge Cases:* Message delivery failures, invalid phone number.

- **Send a message1**  
  - *Type:* Gmail node, sends confirmation email about the created event.  
  - *Role:* Alternative notification channel.  
  - *Key Config:* Recipient email, subject and message content dynamically generated from event data.  
  - *Credentials:* Gmail OAuth2.  
  - *Connections:* Outputs to "Update status in sheet".  
  - *Edge Cases:* Email delivery failures, auth issues.

- **Update status in sheet**  
  - *Type:* Google Sheets node, updates the row corresponding to the scheduled event to mark status as "checked" and reminder status as "sent".  
  - *Role:* Prevents duplicate scheduling by marking processed rows.  
  - *Key Config:* Uses matching `row_number` from "Get in sheet".  
  - *Credentials:* Google Sheets OAuth2.  
  - *Connections:* Outputs to "Wait".  
  - *Edge Cases:* Sheet update conflicts, incorrect row matching.

- **Wait**  
  - *Type:* Wait node, delays next iteration or action to manage rate limits and orderly processing.  
  - *Role:* Throttles workflow execution to avoid flooding APIs or notifications.  
  - *Connections:* Loops back to "Loop Over Items".  
  - *Edge Cases:* Misconfigured wait times could slow or overwhelm processing.

---

#### 2.2 Block B: Reminder Event with AI Follow-up Proposals

**Overview:**  
This block automatically pulls recent past meetings from Google Calendar, filters them for follow-up needs, uses an AI Meeting Agent powered by Google Gemini via LangChain to find suitable future slots, converts AI output into friendly messages, and notifies clients via WhatsApp and Email.

**Nodes Involved:**  
- Schedule Trigger1  
- Get Past Events (Google Calendar)  
- Mark as Seen (Remove Duplicates)  
- Loop Over Items1 (Split in Batches)  
- Only Follow Ups (Filter)  
- Meeting Agent (LangChain Agent using Google Gemini)  
- Availability (Google Calendar Tool)  
- OpenAI (Google Gemini LLM Chat)  
- Output (LangChain Output Parser Structured)  
- Generate Message (Set node)  
- Rapiwa1 (WhatsApp)  
- Send a message (Gmail)  
- Wait1  

**Node Details:**

- **Schedule Trigger1**  
  - *Type:* Trigger node, runs on scheduled intervals (every few minutes).  
  - *Role:* Initiates the follow-up suggestion workflow.  
  - *Connections:* Outputs to "Get Past Events".  
  - *Edge Cases:* Interval timing affects freshness of suggestions.

- **Get Past Events**  
  - *Type:* Google Calendar node, retrieves all events in the calendar between now and 4 days ago.  
  - *Role:* Fetches recent meetings eligible for follow-up.  
  - *Key Config:* `timeMin` = now minus 4 days; `timeMax` = now. Returns all matching events.  
  - *Credentials:* Google Calendar OAuth2.  
  - *Connections:* Outputs to "Mark as Seen".  
  - *Edge Cases:* API limits, time zone accuracy.

- **Mark as Seen**  
  - *Type:* RemoveDuplicates node, filters out events already processed in previous workflow runs based on event ID.  
  - *Role:* Ensures idempotency by avoiding duplicate follow-ups.  
  - *Connections:* Outputs to "Loop Over Items1".  
  - *Edge Cases:* Improper deduplication can cause repeats or misses.

- **Loop Over Items1**  
  - *Type:* SplitInBatches node, processes events one at a time for AI analysis.  
  - *Connections:* Outputs to "Only Follow Ups".  
  - *Edge Cases:* Large load could slow processing.

- **Only Follow Ups**  
  - *Type:* Filter node, lets through only events matching specific follow-up criteria (currently placeholder conditions).  
  - *Role:* Focuses AI effort on relevant meetings.  
  - *Connections:* Outputs to "Meeting Agent".  
  - *Edge Cases:* Filter misconfiguration may block or allow unwanted events.

- **Meeting Agent**  
  - *Type:* LangChain Agent node with Google Gemini (PaLM) AI model integration.  
  - *Role:* Analyzes previous meeting details, uses calendar availability tool to find similar future slots, and returns a list of available meeting times.  
  - *Key Config:*  
    - System message instructs AI to consider day, time, duration, future scheduling, and output format.  
    - Input includes meeting summary, start datetime, and duration (calculated).  
    - Uses Google Calendar Tool node as an AI tool to check calendar availability.  
  - *Credentials:* Google Calendar OAuth2 and Google Gemini API.  
  - *Connections:* Outputs to "Generate Message".  
  - *Edge Cases:* API timeouts, malformed AI output, unexpected AI responses.

- **Availability**  
  - *Type:* Google Calendar Tool node, queried by the AI agent to check free slots between provided start and end times.  
  - *Role:* Supplies real calendar availability data to AI.  
  - *Credentials:* Google Calendar OAuth2.  
  - *Edge Cases:* Calendar access errors or inconsistent data.

- **OpenAI**  
  - *Type:* LangChain LLM Chat node with GPT-4.1-mini model (Google Gemini).  
  - *Role:* Underlying language model powering the Meeting Agent’s reasoning.  
  - *Edge Cases:* Model API limits, unstable responses.

- **Output**  
  - *Type:* LangChain Output Parser Structured node, expects and validates AI response JSON schema containing an array of slots with start and end times.  
  - *Role:* Ensures predictable structured output for downstream usage.  
  - *Edge Cases:* Parsing errors if AI output does not match schema.

- **Generate Message**  
  - *Type:* Set node, formats the AI-provided slots into a user-friendly multi-line message excluding weekend slots, with formatted dates and times.  
  - *Role:* Prepares text message content for notifications.  
  - *Key Config:* Uses Luxon DateTime formatting expressions in JS template.  
  - *Connections:* Outputs to "Rapiwa1" and "Send a message".  
  - *Edge Cases:* Empty slot arrays or formatting exceptions.

- **Rapiwa1**  
  - *Type:* Rapiwa node, sends WhatsApp follow-up message with available slots.  
  - *Credentials:* Rapiwa API.  
  - *Edge Cases:* Delivery failures, incorrect phone number.

- **Send a message**  
  - *Type:* Gmail node, sends follow-up email with available slots message.  
  - *Credentials:* Gmail OAuth2.  
  - *Edge Cases:* Email delivery failures.

- **Wait1**  
  - *Type:* Wait node, delays workflow continuation to pace message sending.  
  - *Edge Cases:* Misconfigured delay may stall or flood notifications.

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                               | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                                                     |
|---------------------|-----------------------------------|----------------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                   | Start event creation workflow                 | -                                | Get in sheet                        | ## Schedule Trigger * **Purpose:** To **start the entire workflow automatically** at regular intervals (like a cron job).      |
| Get in sheet         | Google Sheets                     | Fetch meeting data from Google Sheet          | Schedule Trigger                 | Loop Over Items                    | ## Get in sheet * **Purpose:** To **fetch data** from a Google Sheet containing information for meetings that need scheduling.  |
| Loop Over Items      | SplitInBatches                   | Process each meeting request sequentially     | Get in sheet                    | Date & Time                       |                                                                                                                                |
| Date & Time          | DateTime                         | Format event end time into custom string      | Loop Over Items                 | Code                            | ## Date & Time * **Purpose:** To **standardize and format the event end time** from the Google Sheet into a usable format.     |
| Code                | Code                             | Convert formatted date string to ISO UTC      | Date & Time                    | Create an event                 | ## Code * **Purpose:** Convert formatted date into ISO 8601 UTC string for Google Calendar.                                     |
| Create an event      | Google Calendar                  | Create Google Calendar event                   | Code                          | Rapiwa, Send a message1          | ## Create an event * **Purpose:** To **create the new meeting event** in the designated Google Calendar.                       |
| Rapiwa               | Rapiwa (WhatsApp)                | Send WhatsApp confirmation message             | Create an event                | -                               | ## Rapiwa, Send a message * **Purpose:** To **send a confirmation message via WhatsApp** to the client after event creation.   |
| Send a message1      | Gmail                           | Send Email confirmation of event               | Create an event                | Update status in sheet           |                                                                                                                                 |
| Update status in sheet | Google Sheets                   | Mark meeting as processed in Google Sheet      | Send a message1               | Wait                           | ## Update status in sheet * **Purpose:** To **mark the row in the Google Sheet as processed** after event scheduling.          |
| Wait                 | Wait                            | Delay next batch processing                     | Update status in sheet         | Loop Over Items                 |                                                                                                                                |
| Schedule Trigger1    | Schedule Trigger                 | Start follow-up proposal workflow              | -                              | Get Past Events                 |                                                                                                                                |
| Get Past Events      | Google Calendar                 | Retrieve past events from Google Calendar      | Schedule Trigger1             | Mark as Seen                   | ## Get Recent Meetings * **Purpose:** To pull past meetings for follow-up, avoiding duplicates.                                 |
| Mark as Seen         | RemoveDuplicates                | Remove already processed events                 | Get Past Events               | Loop Over Items1               |                                                                                                                                |
| Loop Over Items1     | SplitInBatches                 | Process past events one by one                   | Mark as Seen                  | Only Follow Ups               |                                                                                                                                |
| Only Follow Ups      | Filter                         | Filter events that require follow-up            | Loop Over Items1              | Meeting Agent                 | ## Only Follow Ups * **Purpose:** To **filter the events** so only follow-up meetings proceed.                                  |
| Meeting Agent        | LangChain Agent (Gemini AI)     | Analyze meeting and find available slots        | Only Follow Ups               | Generate Message              | ## Meeting Agent * **Purpose:** AI assistant to find suitable follow-up slots considering previous meeting details.           |
| Availability         | Google Calendar Tool           | Check real Google Calendar availability          | Meeting Agent (used as tool)   | Meeting Agent (used as tool)   |                                                                                                                                |
| OpenAI               | LangChain LLM Chat (Gemini)    | AI language model powering Meeting Agent        | Meeting Agent (ai_languageModel) | Meeting Agent (ai_languageModel) |                                                                                                                                |
| Output               | LangChain Output Parser        | Parse AI output into structured JSON             | Meeting Agent (ai_outputParser) | Meeting Agent (ai_outputParser) |                                                                                                                                |
| Generate Message     | Set                            | Format AI slots into user-friendly message       | Meeting Agent                 | Rapiwa1, Send a message        |                                                                                                                                |
| Rapiwa1              | Rapiwa (WhatsApp)              | Send follow-up WhatsApp message                   | Generate Message              | -                             | ## Rapiwa * **Purpose:** To **send the follow-up message via WhatsApp** to the customer.                                        |
| Send a message       | Gmail                         | Send follow-up Email message                       | Generate Message              | Wait1                         | ## Send a message * **Purpose:** To **send the follow-up message via Email** as a secondary communication channel.             |
| Wait1                | Wait                          | Delay after follow-up message sent                | Send a message                | Loop Over Items1 (implicit)    |                                                                                                                                |
| Sticky Note           | Sticky Note                   | Documentation and explanation                      | -                            | -                             | Various notes describing each major block and node group, including links and usage tips.                                      |

---

### 4. Reproducing the Workflow from Scratch

**Step-by-Step to Rebuild the Workflow:**

1. **Create Schedule Trigger (Create Event Branch):**  
   - Node Type: Schedule Trigger  
   - Configuration: Set interval to every few minutes (e.g., 5 minutes) to poll Google Sheet for new meetings.

2. **Add Google Sheets Node ("Get in sheet"):**  
   - Operation: Read rows from the target spreadsheet and sheet.  
   - Filter: Filter rows where "status" column is empty or not "checked".  
   - Credentials: Google Sheets OAuth2.  
   - Connect Schedule Trigger output to this node.

3. **Add SplitInBatches Node ("Loop Over Items"):**  
   - Purpose: Process rows one at a time for stable API calls.  
   - Connect output of "Get in sheet" to this node.

4. **Add DateTime Node ("Date & Time"):**  
   - Operation: Format date/time.  
   - Input: Use `{{$json["end time"]}}`.  
   - Format: Custom `yyyy-MM-dd HH:mm:ss ZZZZ` to prepare for JS conversion.  
   - Connect from "Loop Over Items".

5. **Add Code Node ("Code"):**  
   - JS code: Convert formatted string to UTC ISO 8601 string.  
   - Input: All items from Date & Time node.  
   - Connect from "Date & Time".

6. **Add Google Calendar Node ("Create an event"):**  
   - Operation: Create event.  
   - Calendar ID: Set to your target calendar.  
   - Event fields: Summary, location, description from JSON, end time from code node output.  
   - Reminders: Email reminder 30 minutes before.  
   - Credentials: Google Calendar OAuth2.  
   - Connect from "Code".

7. **Add Rapiwa Node:**  
   - Send WhatsApp confirmation message with event summary and time.  
   - Credentials: Rapiwa API.  
   - Connect from "Create an event".

8. **Add Gmail Node ("Send a message1"):**  
   - Send email confirmation with dynamic subject and body from event details.  
   - Credentials: Gmail OAuth2.  
   - Connect from "Create an event".

9. **Add Google Sheets Node ("Update status in sheet"):**  
   - Operation: Update row to set "status" = "checked" and "reminder status" = "sent".  
   - Use matching "row_number" from initial sheet fetch.  
   - Credentials: Google Sheets OAuth2.  
   - Connect from "Send a message1".

10. **Add Wait Node ("Wait"):**  
    - Purpose: Delay next iteration for API or rate limit safety.  
    - Connect from "Update status in sheet".  
    - Connect output back to "Loop Over Items" for processing next batch item.

---

11. **Create Schedule Trigger (Reminder Event Branch):**  
    - Node Type: Schedule Trigger  
    - Configuration: Runs every few minutes (e.g., 5 minutes) to check past events.

12. **Add Google Calendar Node ("Get Past Events"):**  
    - Operation: Get all events from calendar between now minus 4 days and now.  
    - Credentials: Google Calendar OAuth2.  
    - Connect from Schedule Trigger.

13. **Add RemoveDuplicates Node ("Mark as Seen"):**  
    - Deduplicate by event ID to avoid reprocessing.  
    - Connect from "Get Past Events".

14. **Add SplitInBatches Node ("Loop Over Items1"):**  
    - Process events one by one.  
    - Connect from "Mark as Seen".

15. **Add Filter Node ("Only Follow Ups"):**  
    - Filter events for those requiring follow-up based on some criteria (customize).  
    - Connect from "Loop Over Items1".

16. **Add LangChain Agent Node ("Meeting Agent"):**  
    - Configure input text with meeting summary, start datetime, and calculated duration.  
    - Set system message instructing AI to find similar future slots considering day/time/duration.  
    - Use Google Calendar Tool node as AI tool for availability checks.  
    - Credentials: Google Gemini (PaLM), Google Calendar OAuth2.  
    - Connect from "Only Follow Ups".

17. **Add Google Calendar Tool Node ("Availability"):**  
    - Used as AI tool by Meeting Agent to query calendar availability.  
    - Credentials: Google Calendar OAuth2.

18. **Add LangChain LLM Chat Node ("OpenAI"):**  
    - Model: GPT-4.1-mini (Google Gemini).  
    - Used internally by Meeting Agent.

19. **Add LangChain Output Parser Node ("Output"):**  
    - Schema: JSON object with `slots` array containing `start` and `end` string fields.  
    - Connect from Meeting Agent.

20. **Add Set Node ("Generate Message"):**  
    - Format AI output slots into a multi-line message excluding weekends and formatting dates/times.  
    - Connect from Meeting Agent.

21. **Add Rapiwa Node ("Rapiwa1"):**  
    - Send WhatsApp message with generated follow-up slots.  
    - Credentials: Rapiwa API.  
    - Connect from "Generate Message".

22. **Add Gmail Node ("Send a message"):**  
    - Send follow-up email with same message content.  
    - Credentials: Gmail OAuth2.  
    - Connect from "Generate Message".

23. **Add Wait Node ("Wait1"):**  
    - Delay to pace follow-up message sending.  
    - Connect from "Send a message".  
    - Connect output back to "Loop Over Items1" for next event.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow uses Rapiwa API for WhatsApp messaging. Rapiwa official site: https://rapiwa.com, dashboard: https://app.rapiwa.com, docs: https://docs.rapiwa.com                                                                                                                                                                                            | Rapiwa WhatsApp API integration                                                                                        |
| Gmail OAuth2 credentials are required for email sending nodes. Ensure proper scopes for sending emails.                                                                                                                                                                                                                                                     | Gmail API setup                                                                                                        |
| Google Gemini (PaLM) API credentials are required for AI agent nodes; ensure access and quota.                                                                                                                                                                                                                                                              | Google PaLM / Gemini API                                                                                                |
| Google Sheets and Google Calendar OAuth2 credentials require appropriate scopes for reading, writing, and calendar management.                                                                                                                                                                                                                            | Google Cloud OAuth2 setup                                                                                              |
| The "Mark as Seen" node is critical to prevent duplicate processing of events across workflow runs. Test thoroughly to avoid duplicate messaging.                                                                                                                                                                                                          | Idempotency safeguard                                                                                                  |
| The AI system message can be customized to match specific business rules, such as preferred meeting hours or maximum look-ahead window.                                                                                                                                                                                                                   | AI agent customization                                                                                                 |
| The Google Sheet must contain columns: title, description, location, color_number, start time, end time, reminder status, status, row_number for correct operation. Use the sample sheet: https://docs.google.com/spreadsheets/d/1DSRrIDvw-Q9NugRK3eT7GMa0kwe0QnGhdySnqq1_kJU/edit?usp=sharing                                                                 | Google Sheet schema requirement                                                                                        |
| Use Wait nodes to manage API rate limits and avoid flooding communication channels. Adjust wait times based on your environment and API limits.                                                                                                                                                                                                            | Rate limiting and pacing                                                                                               |
| The workflow includes extensive documentation as sticky notes directly inside n8n for user reference.                                                                                                                                                                                                                                                      | Embedded workflow documentation                                                                                        |
| Support and community resources: WhatsApp chat (https://wa.me/8801322827799), Discord (https://discord.gg/SsCChWEP), Facebook Group (https://www.facebook.com/groups/spagreenbd), Website (https://spagreen.net), Developer portfolio (https://codecanyon.net/user/spagreen/portfolio)                                                                               | Community and support links                                                                                            |

---

**Disclaimer:** This document is based exclusively on an n8n automated workflow and complies strictly with content policies. All data manipulated is legal and public.