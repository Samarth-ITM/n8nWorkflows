Create Zoom meeting link from Google Calendar invite

https://n8nworkflows.xyz/workflows/create-zoom-meeting-link-from-google-calendar-invite-1340


# Create Zoom meeting link from Google Calendar invite

### 1. Workflow Overview

This workflow automates the creation of Zoom meeting links based on Google Calendar events scheduled within the next 12 hours from execution time. It filters out events that are either in-person meetings, signal meetings, canceled (marked "transparent"), or already existing Zoom meetings created by Calendly, ensuring only relevant virtual meetings are processed. The workflow includes the following logical blocks:

- **1.1 Trigger and Time Calculation:** Initiates the workflow either manually or daily at 7 AM, calculates the time window (next 12 hours) for fetching calendar events.
- **1.2 Google Calendar Event Retrieval:** Queries Google Calendar for events within the calculated time window.
- **1.3 Event Filtering:** Applies multiple conditions to exclude undesired events like in-person, signal, canceled, or existing Zoom meetings.
- **1.4 Zoom Meeting Creation:** For each filtered event, creates a Zoom meeting with matching details (topic, start time, duration, timezone) via Zoom’s OAuth2 API.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Time Calculation

- **Overview:**  
  Initiates the workflow either manually or automatically once per day at 7 AM and calculates the timestamp 12 hours ahead to define the event retrieval window.

- **Nodes Involved:**  
  - Cron Once a Day  
  - On clicking 'execute'  
  - Date & Time

- **Node Details:**

  - **Cron Once a Day**  
    - Type: Cron Trigger  
    - Configuration: Triggers once daily at 07:00 (local time).  
    - Input/Output: No input; outputs trigger signal to Date & Time node.  
    - Edge Cases: Cron service misconfiguration or n8n server downtime could delay or skip trigger.

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Configuration: Allows manual execution for testing or on-demand runs.  
    - Input/Output: No input; outputs trigger to Date & Time node.  
    - Edge Cases: None significant; manual trigger may cause concurrency if combined with scheduled runs.

  - **Date & Time**  
    - Type: Date & Time calculation node  
    - Configuration: Calculates current timestamp plus 12 hours; outputs ISO string under key `later`.  
    - Expressions Used: `new Date().toISOString()` for current time; adds 12 hours duration (12 * 60 * 60 * 1000 ms).  
    - Input: Trigger from either Cron or Manual nodes.  
    - Output: Provides `later` timestamp used to limit Google Calendar query.  
    - Edge Cases: Timezone differences could affect event inclusion if calendar events are in different zones.  
    - Version Note: Uses standard date calculation; no version-specific requirements.

#### 2.2 Google Calendar Event Retrieval

- **Overview:**  
  Fetches all calendar events scheduled between the current time and 12 hours later, as calculated by the previous node.

- **Nodes Involved:**  
  - Google Calendar

- **Node Details:**

  - **Google Calendar**  
    - Type: Google Calendar API node  
    - Configuration: Retrieves all single events (`singleEvents: true`) from a specific calendar (`calendar` parameter to be replaced with actual calendar ID).  
    - Parameters:  
      - `timeMin`: Current time (ISO string)  
      - `timeMax`: 12 hours later (`Date & Time` node output `later`)  
    - Credentials: Google OAuth2 configured (named "Google Calendar account").  
    - Input: Trigger from `Date & Time`.  
    - Output: List of calendar events to be filtered.  
    - Edge Cases:  
      - Incorrect calendar ID or missing permissions cause API errors.  
      - Large number of events may cause timeouts or pagination issues (not explicitly handled).  
      - Timezone mismatches could exclude relevant events.  
    - Note: The placeholder `REPLACE_WITH_CALENDAR_ID` must be replaced with a valid calendar identifier.

#### 2.3 Event Filtering

- **Overview:**  
  Filters out events that should not have Zoom meetings created, including events marked as "transparent" (likely canceled), those with "signal" or "minute meeting" or "in person" in their summaries, to exclude in-person meetings, signal meetings, and canceled Calendly meetings.

- **Nodes Involved:**  
  - IF Zoom meeting

- **Node Details:**

  - **IF Zoom meeting**  
    - Type: If node (conditional branching)  
    - Configuration: Multiple string conditions combined with AND logic (all must be true to pass):  
      - `transparency` field does **not** contain `"transparent"`  
      - `summary` field does **not** contain `"signal"`  
      - `summary` field does **not** contain `"minute meeting"`  
      - `summary` field does **not** contain `"in person"`  
    - Expressions: Uses expressions accessing event JSON fields from the previous Google Calendar node.  
    - Input: Events from Google Calendar node.  
    - Output: Only events passing all filters continue to Zoom meeting creation.  
    - Edge Cases:  
      - Case sensitivity is not explicitly handled; partial matches only.  
      - If fields are missing (e.g. no `summary`), expressions might fail or exclude event unintentionally.  
      - Events not matching these filters are dropped silently; no logging or alternate path shown.  
    - Notes: Sticky note on this node states filters applied and their rationale.

#### 2.4 Zoom Meeting Creation

- **Overview:**  
  For each filtered Google Calendar event, creates a Zoom meeting using Zoom's API, replicating event details such as topic, start time, duration, and timezone.

- **Nodes Involved:**  
  - Zoom

- **Node Details:**

  - **Zoom**  
    - Type: Zoom API node  
    - Configuration:  
      - Topic: Set dynamically from the event's summary (`{{$node["IF Zoom meeting"].json["summary"]}}`).  
      - Duration: Calculated as the difference between event end and start times in minutes.  
      - Start Time and TimeZone: Taken directly from event's `start.dateTime` and `start.timeZone`.  
      - Authentication: OAuth2 (configured with credential named “Zoom account”).  
      - Additional fields: No extra settings beyond duration and timezone.  
    - Expressions: Dynamically compute meeting details based on filtered event data.  
    - Input: Filtered event from IF Zoom meeting node.  
    - Output: Created Zoom meeting details (e.g., meeting link).  
    - Edge Cases:  
      - OAuth2 token expiration or invalid credentials cause authorization errors.  
      - API rate limits could cause failures on bulk event processing.  
      - Incorrect event time fields may cause Zoom API errors.  
      - Missing or malformed date-time fields cause expression errors.  
    - Version Note: Requires OAuth2 credentials pre-configured.  
    - Sticky Note: None attached.

---

### 3. Summary Table

| Node Name         | Node Type          | Functional Role                         | Input Node(s)         | Output Node(s)      | Sticky Note                                                                                   |
|-------------------|--------------------|---------------------------------------|-----------------------|---------------------|-----------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger     | Manual start of workflow               | —                     | Date & Time         |                                                                                               |
| Cron Once a Day    | Cron Trigger       | Scheduled daily trigger at 7 AM       | —                     | Date & Time         |                                                                                               |
| Date & Time       | Date & Time        | Calculates current time + 12 hours    | On clicking 'execute', Cron Once a Day | Google Calendar |                                                                                               |
| Google Calendar   | Google Calendar API | Retrieves calendar events within timeframe | Date & Time           | IF Zoom meeting     |                                                                                               |
| IF Zoom meeting   | If (conditional)   | Filters out in-person, signal, canceled events | Google Calendar       | Zoom                | Filters out: existing Zoom meetings made by Calendly, in person meetings, signal meetings, canceled Calendly meetings ("transparent") |
| Zoom              | Zoom API           | Creates Zoom meetings matching events | IF Zoom meeting       | —                   |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `On clicking 'execute'` for manual runs.

2. **Create a Cron Trigger node** named `Cron Once a Day`.  
   - Configure it to trigger once daily at 07:00 (local time).

3. **Create a Date & Time node** named `Date & Time`.  
   - Set the value to current time: `={{new Date().toISOString()}}`.  
   - Action: Calculate.  
   - Duration: 12.  
   - Time Unit: hours.  
   - Output property name: `later`.

4. **Connect** both `On clicking 'execute'` and `Cron Once a Day` nodes to the `Date & Time` node (multiple triggers feeding one node).

5. **Create a Google Calendar node** named `Google Calendar`.  
   - Operation: Get All events.  
   - Options:  
     - `timeMin`: `={{new Date().toISOString()}}` (current time)  
     - `timeMax`: `={{$node["Date & Time"].json["later"]}}` (12 hours later)  
     - `singleEvents`: true (to get individual event instances)  
   - Calendar: Replace placeholder with valid Google Calendar ID.  
   - Credential: Configure with a valid Google Calendar OAuth2 credential.

6. **Connect** `Date & Time` node output to `Google Calendar` node input.

7. **Create an If node** named `IF Zoom meeting`.  
   - Configure conditions (all must be true to pass):  
     - `transparency` field does NOT contain `"transparent"`  
     - `summary` field does NOT contain `"signal"`  
     - `summary` field does NOT contain `"minute meeting"`  
     - `summary` field does NOT contain `"in person"`  
   - Use expressions to access these fields from the incoming Google Calendar event data.

8. **Connect** `Google Calendar` node output to `IF Zoom meeting` node.

9. **Create a Zoom node** named `Zoom`.  
   - Authentication: OAuth2 with a valid Zoom account credential.  
   - Parameters:  
     - Topic: Set to `={{$node["IF Zoom meeting"].json["summary"]}}`  
     - Start Time: `={{$node["IF Zoom meeting"].json["start"]["dateTime"]}}`  
     - Duration (minutes): calculated as `(Date.parse(end) - Date.parse(start)) / (60*1000)` using expressions on the event's start and end dateTime fields.  
     - Time Zone: `={{$node["IF Zoom meeting"].json["start"]["timeZone"]}}`  
     - Additional settings: leave defaults or empty.  

10. **Connect** the `IF Zoom meeting` node’s "true" output to the `Zoom` node.

11. Confirm all credentials (Google Calendar OAuth2 and Zoom OAuth2) are properly set up and authorized.

12. Save and activate the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| The calendar ID in the Google Calendar node must be replaced with a valid calendar identifier before use.     | Configuration detail for Google Calendar node                                                          |
| The workflow filters out events marked as "transparent" to exclude canceled meetings, especially from Calendly.| Explanation of filtering logic in the IF Zoom meeting node’s sticky note                                |
| OAuth2 credentials for both Google Calendar and Zoom must be pre-configured with the appropriate scopes.       | Required for API access and authentication                                                             |
| Timezone handling depends on event data from Google Calendar and is passed to Zoom to ensure correct scheduling.| Important for correct meeting scheduling across time zones                                              |
| Manual trigger node allows testing outside the scheduled time.                                                | Useful for debugging or immediate runs                                                                  |

---

This documentation fully details the workflow structure, node configurations, and logic required to replicate, understand, and maintain the automated Zoom meeting creation from Google Calendar invites.