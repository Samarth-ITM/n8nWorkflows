Automated Weekly Google Calendar Summary via Email with AI ✨🗓️📧

https://n8nworkflows.xyz/workflows/automated-weekly-google-calendar-summary-via-email-with-ai--------4783


# Automated Weekly Google Calendar Summary via Email with AI ✨🗓️📧

### 1. Workflow Overview

This workflow automates the generation and emailing of a personalized weekly summary of upcoming Google Calendar events. Scheduled to run once per week, it fetches events from a specified Google Calendar for the coming seven days, processes and formats the data using AI, converts the summary into HTML, and sends it via email.

Logical blocks:

- **1.1 Scheduled Trigger and User Context Setup:** Triggers the workflow weekly and sets user-specific locale, timezone, and identity details.
- **1.2 Date-Time Computations:** Calculates precise date-time boundaries for the upcoming week based on user locale and timezone.
- **1.3 Google Calendar Event Retrieval:** Fetches all calendar events between the computed start and end dates.
- **1.4 Event Data Simplification and Aggregation:** Cleans up raw event data by removing extraneous fields and aggregates all events into a single JSON array.
- **1.5 AI-Powered Summary Generation:** Uses Google Gemini AI through an AI agent node to produce a friendly, structured weekly schedule summary highlighting priority events.
- **1.6 Markdown to HTML Conversion:** Converts the AI-generated Markdown summary into HTML format suitable for email.
- **1.7 Email Dispatch:** Sends the formatted summary via SMTP email with a dynamic subject line.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and User Context Setup

- **Overview:** Initiates the workflow once per week and establishes personalized settings such as locale, timezone, user name, and home city for correct formatting and contextual AI processing.
- **Nodes Involved:** `weekly_schedule`, `locale`

##### Node: weekly_schedule
- **Type:** Schedule Trigger
- **Role:** Automatically triggers the workflow every week at 12:00 PM.
- **Configuration:** Weekly interval schedule with trigger at 12:00.
- **Inputs:** None (trigger node).
- **Outputs:** Triggers `locale` node.
- **Edge Cases:** Misconfigured timing may cause unexpected trigger intervals.
- **Sticky Note:** Explains purpose and how to modify trigger time.

##### Node: locale
- **Type:** Set
- **Role:** Defines user-specific parameters for localization and personalization.
- **Configuration:** Sets four string variables:  
  - `users-locale` (e.g., `"en-AU"`)  
  - `users-timezone` (e.g., `"Australia/Sydney"`)  
  - `users-name` (e.g., `"Bob"`)  
  - `users-home-city` (e.g., `"Sydney"`)
- **Inputs:** Triggered by `weekly_schedule`.
- **Outputs:** Passes data to `date_time`.
- **Edge Cases:** Incorrect locale or timezone can cause wrong date/time formatting.
- **Sticky Note:** Emphasizes user action required to configure these values properly.

---

#### 1.2 Date-Time Computations

- **Overview:** Calculates various date and time strings to define the appropriate 7-day window for event retrieval, adjusted for the user’s timezone and locale.
- **Nodes Involved:** `date_time`
  
##### Node: date_time
- **Type:** Set
- **Role:** Generates multiple date/time strings used for filtering calendar events and formatting.
- **Configuration:** Uses JavaScript expressions with `$now` and Luxon DateTime to produce variables such as:  
  - `users-current-date` (localized full date)  
  - `users-current-day-1-minute-before-midnight-iso` (ISO timestamp at end of current day)  
  - `users-current-day-1-minute-before-midnight-plus-7-days-iso` (ISO timestamp at end of day seven days later)  
  - Plus other localized date/time formats for reference.
- **Inputs:** Receives from `locale`.
- **Outputs:** Sends data to `get_next_weeks_events`.
- **Edge Cases:** Errors in timezone or locale expressions could lead to incorrect date boundaries.
- **Sticky Note:** Notes dynamic generation, usually no user modification required.

---

#### 1.3 Google Calendar Event Retrieval

- **Overview:** Connects to Google Calendar API to retrieve all events scheduled within the upcoming week.
- **Nodes Involved:** `get_next_weeks_events`

##### Node: get_next_weeks_events
- **Type:** Google Calendar
- **Role:** Fetches all events between `timeMin` and `timeMax` calculated in `date_time`.
- **Configuration:**  
  - `timeMin` set to ISO string for end of current day (`users-current-day-1-minute-before-midnight-iso`)  
  - `timeMax` set to ISO string for end of day plus 7 days (`users-current-day-1-minute-before-midnight-plus-7-days-iso`)  
  - Calendar ID configured to a specific Google Group calendar (replace with user’s own calendar ID).  
  - `returnAll` enabled to fetch all events in range.
- **Credentials:** Requires Google Calendar OAuth2 credentials.
- **Inputs:** From `date_time`.
- **Outputs:** Passes event data to `simplify_evens_json`.
- **Edge Cases:**  
  - Authentication failure if OAuth2 credentials are invalid or expired.  
  - Empty event list if no events scheduled.  
  - Calendar ID misconfiguration leads to wrong or no data.
- **Sticky Note:** Highlights user action to update calendar ID and credentials.

---

#### 1.4 Event Data Simplification and Aggregation

- **Overview:** Cleans fetched event objects by removing unnecessary metadata fields and compiles all event JSON objects into a single aggregated array.
- **Nodes Involved:** `simplify_evens_json`, `aggregate_events`

##### Node: simplify_evens_json
- **Type:** Code (JavaScript)
- **Role:** Iterates over each event JSON, deleting fields such as `htmlLink`, `etag`, `iCalUID`, `organizer`, `status`, etc.
- **Configuration:** Custom JavaScript code explicitly removes unwanted fields to reduce noise for AI processing.
- **Inputs:** From `get_next_weeks_events`.
- **Outputs:** Passes cleaned events to `aggregate_events`.
- **Edge Cases:** If event JSON structure changes, deletion code may fail or remove unexpected fields.
- **Sticky Note:** Notes purpose and advises modification only if specific fields need retaining or further removal.

##### Node: aggregate_events
- **Type:** Aggregate
- **Role:** Combines all simplified event items into a single JSON array field `eventdata`.
- **Configuration:** Uses “aggregate all item data” mode with destination field `eventdata`.
- **Inputs:** From `simplify_evens_json`.
- **Outputs:** Passes aggregated events to `event_summary_agent`.
- **Edge Cases:** Input must be an array; empty input results in empty `eventdata`.
- **Sticky Note:** Usually no changes needed.

---

#### 1.5 AI-Powered Summary Generation

- **Overview:** Utilizes Google Gemini AI via an agent node to create a clear, friendly, and formatted weekly event summary, accentuating priority events based on keywords.
- **Nodes Involved:** `Google Gemini`, `event_summary_agent`

##### Node: Google Gemini
- **Type:** Language Model (Google Gemini via LangChain)
- **Role:** Provides connection and model interface for the AI agent.
- **Configuration:**  
  - Model name set to `models/gemini-2.5-flash-preview-05-20`.  
  - Credentials set for Google PaLM API.
- **Inputs:** None directly; linked as an AI language model provider.
- **Outputs:** Supplies model to `event_summary_agent`.
- **Edge Cases:** API quota limits, authentication failures, or model deprecation may cause errors.
- **Sticky Note:** User must configure credentials and can change the model name if needed.

##### Node: event_summary_agent
- **Type:** LangChain AI Agent
- **Role:** Sends the aggregated JSON event data and detailed prompt to Google Gemini, instructing it to generate the weekly summary text.
- **Configuration:**  
  - Input text is set to the aggregated `eventdata`.  
  - System message is a comprehensive prompt detailing:  
    - Data structure and relevant fields (`summary`, `start.dateTime`, `end.dateTime`, `description`)  
    - User locale, timezone, name, and city for context  
    - Formatting instructions for greeting, daily grouping, event formatting with time conversion, priority event identification with keywords, weekly insight generation, and output constraints (email body text only).  
- **Inputs:** Receives aggregated events and AI model reference.
- **Outputs:** Produces Markdown text summary passed to `markdown_to_html`.
- **Edge Cases:**  
  - Prompt expression errors could break query.  
  - AI response delays or failures.  
  - Inaccurate timezone handling if locale data is wrong.  
- **Sticky Note:** Suggests customizable prompt tweaks for greeting, formatting, keywords, and insights.

---

#### 1.6 Markdown to HTML Conversion

- **Overview:** Converts AI-generated Markdown summary into HTML format for email compatibility and better presentation.
- **Nodes Involved:** `markdown_to_html`

##### Node: markdown_to_html
- **Type:** Markdown
- **Role:** Converts Markdown string from AI agent into HTML.
- **Configuration:**  
  - Options enable emoji and simple line breaks for better formatting.  
  - Input is the AI agent’s output text.
- **Inputs:** From `event_summary_agent`.
- **Outputs:** Passes HTML content as `email-html` to `send_email`.
- **Edge Cases:** Malformed Markdown could cause conversion errors.
- **Sticky Note:** Notes purpose and that changes are usually unnecessary.

---

#### 1.7 Email Dispatch

- **Overview:** Sends the formatted weekly schedule summary as an email with a dynamic subject line.
- **Nodes Involved:** `send_email`

##### Node: send_email
- **Type:** Email Send
- **Role:** Sends an email containing the HTML summary.
- **Configuration:**  
  - Email body set from `email-html`.  
  - Subject dynamically constructed with next week’s date range (day number to localized date).  
  - SMTP credentials must be configured.  
  - From and To email addresses configurable by user.
- **Inputs:** Receives HTML from `markdown_to_html`.
- **Outputs:** None (final node).
- **Edge Cases:**  
  - SMTP authentication failure.  
  - Invalid email addresses.  
  - Network timeouts.
- **Sticky Note:** Emphasizes required SMTP credentials and user modifications for sender and recipient emails.

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                             | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                              |
|------------------------|----------------------------------|--------------------------------------------|--------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| weekly_schedule        | Schedule Trigger                 | Workflow trigger, runs weekly at 12:00 PM | None                     | locale                     | *Initiates workflow weekly at 12:00 PM. Adjust rule to change schedule.*                               |
| locale                 | Set                             | Defines user locale, timezone, name, city | weekly_schedule          | date_time                  | *Configure user locale, timezone, name, city for personalization and date formatting.*                 |
| date_time              | Set                             | Calculates date/time boundaries for events | locale                   | get_next_weeks_events      | *Generates date/time strings to define 7-day event window; usually no modification needed.*            |
| get_next_weeks_events  | Google Calendar                 | Retrieves events for upcoming week         | date_time                | simplify_evens_json        | *Set your Google Calendar ID and OAuth2 credentials to fetch events.*                                 |
| simplify_evens_json    | Code (JavaScript)               | Cleans event data, removing extraneous fields | get_next_weeks_events    | aggregate_events           | *Removes unneeded event fields to simplify AI processing; modify code only if necessary.*               |
| aggregate_events       | Aggregate                      | Aggregates all events into one JSON array  | simplify_evens_json      | event_summary_agent        | *Combines all events in a single field for AI consumption; usually no changes required.*               |
| Google Gemini          | LangChain AI Language Model     | Provides AI model for summary generation   | None                     | event_summary_agent (as AI model) | *Set Google Gemini (PaLM) API credentials; optionally change model name.*                              |
| event_summary_agent    | LangChain AI Agent              | Generates weekly summary text from events  | aggregate_events, Google Gemini | markdown_to_html         | *Uses a detailed prompt for formatting, priority detection, and insights; customizable prompt.*        |
| markdown_to_html       | Markdown Converter              | Converts Markdown summary to HTML           | event_summary_agent      | send_email                 | *Converts Markdown to HTML with emojis and line breaks enabled; no changes usually needed.*             |
| send_email             | Email Send (SMTP)               | Sends formatted summary via email           | markdown_to_html         | None                      | *Configure SMTP credentials, from/to email addresses, and optionally the subject line.*                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add **Schedule Trigger** node named `weekly_schedule`.
   - Configure to run once per week at 12:00 PM (set interval to 1 week, trigger at hour 12).
   
2. **Set User Locale and Personalization:**
   - Add **Set** node named `locale`.
   - Connect `weekly_schedule` → `locale`.
   - Add these string fields with appropriate values:  
     - `users-locale` (e.g., `en-AU`)  
     - `users-timezone` (e.g., `Australia/Sydney`)  
     - `users-name` (e.g., `Bob`)  
     - `users-home-city` (e.g., `Sydney`)

3. **Calculate Date-Time Boundaries:**
   - Add **Set** node named `date_time`.
   - Connect `locale` → `date_time`.
   - Define multiple fields with expressions using `$now` and Luxon to generate:  
     - `users-current-date` (localized full date string)  
     - `users-current-day-1-minute-before-midnight-iso` (ISO of end of current day)  
     - `users-current-day-1-minute-before-midnight-plus-7-days-iso` (ISO of end of day +7)  
     - Additional date/time formats as needed for reference.
   - Use timezone and locale values from `locale` node.

4. **Fetch Google Calendar Events:**
   - Add **Google Calendar** node named `get_next_weeks_events`.
   - Connect `date_time` → `get_next_weeks_events`.
   - Operation: `getAll`.
   - Set `timeMin` to `={{ $json["users-current-day-1-minute-before-midnight-iso"] }}`.
   - Set `timeMax` to `={{ $json["users-current-day-1-minute-before-midnight-plus-7-days-iso"] }}`.
   - Set `returnAll` to true.
   - Specify the calendar ID for your target Google Calendar.
   - Configure Google Calendar OAuth2 credentials.

5. **Simplify Event JSON Objects:**
   - Add **Code** node named `simplify_evens_json`.
   - Connect `get_next_weeks_events` → `simplify_evens_json`.
   - Implement JavaScript to delete unwanted fields such as `htmlLink`, `etag`, `iCalUID`, `organizer`, `status`, `created`, `updated`, `eventType`, `id`, `reminders`, `creator`.
   - Return the cleaned event array.

6. **Aggregate Events Into One Array:**
   - Add **Aggregate** node named `aggregate_events`.
   - Connect `simplify_evens_json` → `aggregate_events`.
   - Set aggregation mode to "Aggregate all item data".
   - Use destination field name: `eventdata`.

7. **Setup Google Gemini AI Model Node:**
   - Add **Google Gemini** node.
   - Configure with Google PaLM API credentials.
   - Use model `models/gemini-2.5-flash-preview-05-20` or preferred variant.

8. **Configure AI Agent for Summary Generation:**
   - Add **LangChain AI Agent** node named `event_summary_agent`.
   - Connect `aggregate_events` → `event_summary_agent` main input.
   - Connect `Google Gemini` → `event_summary_agent` AI language model input.
   - Pass aggregated events as text input.
   - Set a detailed system message prompt with instructions for greeting, event grouping by day, formatting times in user timezone, marking priority events based on keywords (`important`, `urgent`, etc.), and concluding with weekly insight.
   - Include user locale, timezone, name, and city variables dynamically from prior nodes.

9. **Convert Markdown Summary to HTML:**
   - Add **Markdown** node named `markdown_to_html`.
   - Connect `event_summary_agent` → `markdown_to_html`.
   - Set mode to `markdownToHtml`.
   - Enable emoji and simple line breaks options.
   - Use AI agent output as markdown input.
   - Set destination key: `email-html`.

10. **Send Email with Summary:**
    - Add **Email Send** node named `send_email`.
    - Connect `markdown_to_html` → `send_email`.
    - Select and configure SMTP credentials.
    - Set `fromEmail` and `toEmail` to desired email addresses.
    - Set email subject dynamically:  
      `Next Week Calendar Summary : {{ $now.plus({ days: 1 }).day }}-{{ $now.plus({ days: 7 }).toLocaleString(DateTime.DATE_MED) }}`.
    - Use `email-html` output as HTML body.
   
11. **Activate Workflow and Test:**
    - Activate the workflow.
    - Run manually or wait for scheduled trigger.
    - Verify email receipt and summary content.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow automatically generates and emails a personalized weekly calendar summary using AI.        | Core project purpose.                                                                                             |
| User must configure locale, timezone, calendar ID, AI credentials, and SMTP credentials properly.    | Critical setup steps for correct operation.                                                                     |
| AI prompt is highly customizable for tone, formatting, and priority keyword detection.               | Found in `event_summary_agent` system message prompt.                                                           |
| Google Gemini API usage requires valid PaLM API credentials and may incur usage costs.               | See Google Cloud PaLM documentation for API setup and quotas.                                                   |
| SMTP node requires valid email server credentials; Gmail and other providers may require app passwords or OAuth2 setup. | Refer to your email provider's SMTP instructions.                                                               |
| Workflow subject line dynamically reflects the upcoming week’s date range for clarity.               | Enhances email relevance and tracking.                                                                           |
| See workflow sticky notes inside n8n for detailed explanations and user action points.               | Sticky notes provide in-editor guidance.                                                                         |

---

**Disclaimer:**  
The provided text results exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All processed data is lawful and public.