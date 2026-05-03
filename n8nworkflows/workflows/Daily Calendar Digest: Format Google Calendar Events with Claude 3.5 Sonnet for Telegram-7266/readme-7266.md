Daily Calendar Digest: Format Google Calendar Events with Claude 3.5 Sonnet for Telegram

https://n8nworkflows.xyz/workflows/daily-calendar-digest--format-google-calendar-events-with-claude-3-5-sonnet-for-telegram-7266


# Daily Calendar Digest: Format Google Calendar Events with Claude 3.5 Sonnet for Telegram

### 1. Workflow Overview

This workflow automates the daily retrieval and summarization of Google Calendar events and sends a formatted digest to a Telegram chat using Claude 3.5 Sonnet AI by Anthropic. It targets users who want a concise, AI-enhanced daily summary of their scheduled events delivered directly to Telegram at a fixed time every morning.

The workflow is logically grouped into these blocks:

- **1.1 Time Trigger:** Initiates the workflow at a scheduled time each day (6:00 AM).
- **1.2 Google Calendar Event Retrieval:** Queries Google Calendar to get all events for the current day.
- **1.3 Event Data Extraction and Aggregation:** Extracts relevant fields (event ID, summary, start time) from each event and aggregates them into a single data string.
- **1.4 AI Processing with Anthropic Claude 3.5 Sonnet:** Uses Claude AI to extract and format event information into a human-readable summary.
- **1.5 Telegram Messaging:** Sends the formatted event summary as a text message to a designated Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Time Trigger

- **Overview:**  
  This block schedules the workflow to run daily at a specific hour (6:00 AM), triggering the start of the event retrieval process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger`  
    - Configuration: Set to trigger once daily at 6:00 AM.  
    - Input: None (start node).  
    - Output: Triggers the "Get many events" node.  
    - Edge Cases: Timezone mismatches could cause unexpected trigger times; ensure server and calendar timezones align.  
    - Version Requirements: Supports n8n v1.2 or higher for interval trigger.

#### 2.2 Google Calendar Event Retrieval

- **Overview:**  
  Fetches all calendar events scheduled for the current day from the user's Google Calendar.

- **Nodes Involved:**  
  - Get many events

- **Node Details:**  
  - **Get many events**  
    - Type: `Google Calendar` node  
    - Configuration:  
      - Operation: Get all events (`getAll`)  
      - Time range: From start of the current day to end of the current day, dynamically calculated via expressions `$now.startOf('day')` and `$now.endOf('day')`.  
      - Calendar selection: Dynamic list, user must select desired calendar.  
    - Credentials: Requires OAuth2 credentials for Google Calendar API.  
    - Input: Triggered by Schedule Trigger node.  
    - Output: List of events with full event data.  
    - Edge Cases:  
      - OAuth token expiration or permission errors.  
      - Empty calendar days result in zero events but workflow continues.  
      - Large event volumes may affect performance or API rate limits.

#### 2.3 Event Data Extraction and Aggregation

- **Overview:**  
  Extracts essential event attributes (ID, summary, start time) and combines all events into a single aggregated data string for further AI processing.

- **Nodes Involved:**  
  - ID, Summary, Time  
  - Combine data  
  - Get string

- **Node Details:**  
  - **ID, Summary, Time**  
    - Type: `Set` node  
    - Configuration: Extracts and assigns three fields per event:  
      - `id`: event ID from Google Calendar JSON  
      - `summary`: event title  
      - `dateTime`: start datetime of the event  
    - Input: Output array from "Get many events" node.  
    - Output: Simplified event objects with only relevant fields.  
    - Edge Cases: Events without summaries or start times may result in incomplete data.  
  - **Combine data**  
    - Type: `Aggregate` node  
    - Configuration: Aggregates all event items into a single JSON object or string for unified processing.  
    - Input: Simplified event data from "ID, Summary, Time" node.  
    - Output: A single aggregated JSON object representing all events.  
    - Edge Cases: Empty input results in empty aggregation.  
  - **Get string**  
    - Type: `Set` node  
    - Configuration: Converts aggregated data into a string field named `data` for input into the AI extractor.  
    - Input: Aggregated data from "Combine data" node.  
    - Output: JSON with a single `data` string field.  
    - Edge Cases: If aggregation fails, this node may receive empty or malformed input.

#### 2.4 AI Processing with Anthropic Claude 3.5 Sonnet

- **Overview:**  
  The aggregated event data string is sent to an AI model (Claude 3.5 Sonnet) which extracts and formats event information into a concise summary emphasizing event names and times.

- **Nodes Involved:**  
  - Anthropic Chat Model1  
  - Event extractor

- **Node Details:**  
  - **Anthropic Chat Model1**  
    - Type: `LangChain Anthropic Chat Model`  
    - Configuration:  
      - Model: `claude-3-5-sonnet-20241022` (Claude Sonnet 3.5)  
      - No additional options set.  
    - Credentials: Requires Anthropic API credentials.  
    - Input: Receives the aggregated event data string.  
    - Output: AI-processed response passed to "Event extractor".  
    - Edge Cases: API rate limits, authentication failures, or unexpected model output could cause errors.  
  - **Event extractor**  
    - Type: `LangChain Information Extractor`  
    - Configuration:  
      - Input text: The `data` string from previous node.  
      - System prompt: Instructs the model to act as an event extractor, focusing on extracting event names and their date/time.  
      - Attributes: Requests extraction of a "Summary" attribute representing all events.  
    - Input: AI output from "Anthropic Chat Model1".  
    - Output: Structured summary text with event details.  
    - Edge Cases: AI output variability may affect extraction accuracy or completeness.

#### 2.5 Telegram Messaging

- **Overview:**  
  Sends the AI-generated summary of daily events as a Telegram message to a specified chat.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**  
  - **Send a text message**  
    - Type: `Telegram` node  
    - Configuration:  
      - Text: Uses expression to insert extracted summary text (`{{$json.output.Summary}}`).  
      - Chat ID: User must specify target Telegram chat ID.  
      - Additional options: Attribution disabled (no sender info appended).  
    - Credentials: Requires Telegram Bot API credentials.  
    - Input: Receives structured summary from "Event extractor".  
    - Output: Sends message to Telegram chat.  
    - Edge Cases: Invalid chat ID, revoked bot permissions, or network errors may cause failures.

---

### 3. Summary Table

| Node Name           | Node Type                                | Functional Role                         | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                                                         |
|---------------------|-----------------------------------------|---------------------------------------|----------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                        | Initiates workflow daily at 6:00 AM  | -                    | Get many events           | ## Set a time<br>Set a time when you want to get a reminder                                                                                         |
| Get many events      | Google Calendar                        | Retrieves all events for the day      | Schedule Trigger     | ID, Summary, Time         | ## Get all events for that day<br>By using 'get many events' and the time schedule, you get events for the current day.<br><br>## Get ID, event, and time<br>You get the event, ID, and time of the events. |
| ID, Summary, Time    | Set                                   | Extracts event ID, summary, start time| Get many events      | Combine data              | (Same as above—part of event data extraction)                                                                                                      |
| Combine data        | Aggregate                             | Aggregates all event data into one   | ID, Summary, Time    | Get string                | ## Combine all data into one<br>By aggregating and editing fields into a string                                                                     |
| Get string           | Set                                   | Prepares aggregated data string       | Combine data         | Event extractor           | (Same as Combine data note)                                                                                                                         |
| Anthropic Chat Model1| LangChain Anthropic Chat Model        | AI model processing of event data     | (No direct input; connected by AI flow) | Event extractor           | (No sticky note)                                                                                                                                     |
| Event extractor      | LangChain Information Extractor       | Extracts and formats event summary    | Get string, Anthropic Chat Model1 | Send a text message       | ## Important extract of data<br><br>And send it via Telegram                                                                                         |
| Send a text message  | Telegram                             | Sends summary text to Telegram chat   | Event extractor      | -                        | (Same as Event extractor note)                                                                                                                     |
| Sticky Note4         | Sticky Note                          | Visual note on scheduling              | -                    | -                        | ## Set a time<br>Set a time when you want to get a reminder                                                                                        |
| Sticky Note5         | Sticky Note                          | Visual note on event retrieval         | -                    | -                        | ## Get all events for that day<br>By using 'get many events' and the time schedule, you get events for the current day.<br><br>## Get ID, event, and time<br>You get the event, ID, and time of the events. |
| Sticky Note6         | Sticky Note                          | Visual note on data aggregation        | -                    | -                        | ## Combine all data into one<br>By aggregating and editing fields into a string                                                                    |
| Sticky Note7         | Sticky Note                          | Visual note on data extraction and sending | -               | -                        | ## Important extract of data<br><br>And send it via Telegram                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: `Schedule Trigger`  
   - Set to trigger daily at 6:00 AM (use interval trigger and specify `triggerAtHour: 6`).  
   - This node has no inputs.

2. **Create Google Calendar node ("Get many events"):**  
   - Type: `Google Calendar`  
   - Set operation to `getAll`.  
   - Set `timeMin` to `={{ $now.startOf('day') }}` and `timeMax` to `={{ $now.endOf('day') }}` to fetch events for the current day.  
   - Select the target calendar from the dropdown (requires OAuth2 credentials for Google Calendar configured in n8n).  
   - Connect Schedule Trigger node output to this node’s input.

3. **Create Set node ("ID, Summary, Time"):**  
   - Type: `Set`  
   - Configure to map:  
     - `id` → `={{ $json["id"] }}`  
     - `summary` → `={{ $json["summary"] }}`  
     - `dateTime` → `={{ $json.start.dateTime }}`  
   - Connect "Get many events" output to this node.

4. **Create Aggregate node ("Combine data"):**  
   - Type: `Aggregate`  
   - Set aggregation mode to `aggregateAllItemData` to combine all event objects into a single JSON.  
   - Connect "ID, Summary, Time" node output to this node.

5. **Create Set node ("Get string"):**  
   - Type: `Set`  
   - Create a new string field `data` with value set to the aggregated data: `={{ $json.data }}`.  
   - Connect "Combine data" output to this node.

6. **Create Anthropic Chat Model node ("Anthropic Chat Model1"):**  
   - Type: `LangChain Anthropic Chat Model`  
   - Select model `claude-3-5-sonnet-20241022` (Claude Sonnet 3.5).  
   - Provide Anthropic API credentials.  
   - Connect this node’s input to the output of "Get string" node, or handle via AI language model connection if supported.

7. **Create Information Extractor node ("Event extractor"):**  
   - Type: `LangChain Information Extractor`  
   - Set input text to `={{ $json.data }}` (the string prepared).  
   - Configure system prompt:  
     ```
     You are event extractor. 
     Your task is get event with date and time.
     ```  
   - Specify attributes to extract: one attribute named "Summary" with description "Get all events".  
   - Connect output of Anthropic Chat Model node to this node's AI language model input.  
   - Connect "Get string" node output to this node’s main input.

8. **Create Telegram node ("Send a text message"):**  
   - Type: `Telegram`  
   - Set text to `={{ $json.output.Summary }}` to use extracted summary text.  
   - Specify `chatId` with your target Telegram chat ID.  
   - Disable attribution (set `appendAttribution` to false).  
   - Provide Telegram Bot API credentials.  
   - Connect output of "Event extractor" node to this node.

9. **Verify all connections:**  
   - Schedule Trigger → Get many events → ID, Summary, Time → Combine data → Get string → Anthropic Chat Model1 (AI input) → Event extractor → Send a text message.

10. **Test workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Check Telegram for the daily summary message.  
    - Troubleshoot any API credential or data format issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow tags include “Time_trigger”, “Calendar”, “Telegram”, and “info_extract” to categorize its functionality.              | Useful for filtering and organizing workflows within n8n projects.                                     |
| The AI model used is Claude 3.5 Sonnet (version 20241022), an advanced language model by Anthropic for conversational tasks. | Anthropic API documentation: https://docs.anthropic.com/                                                |
| Telegram integration requires bot creation and obtaining a valid chat ID where messages will be sent.                         | Telegram Bot API guide: https://core.telegram.org/bots                                                |
| Google Calendar OAuth2 credentials must be properly configured with scopes allowing event read access.                        | Google Calendar API guide: https://developers.google.com/calendar/api/guides/overview                  |
| Time expressions like `$now.startOf('day')` and `$now.endOf('day')` rely on the timezone settings of the n8n instance.       | Confirm n8n timezone settings to ensure correct event day boundaries.                                  |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.