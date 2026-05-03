Automated Trip Weather Forecasts from Google Calendar to Telegram

https://n8nworkflows.xyz/workflows/automated-trip-weather-forecasts-from-google-calendar-to-telegram-10481


# Automated Trip Weather Forecasts from Google Calendar to Telegram

### 1. Workflow Overview

This workflow automates the process of providing personalized weather forecasts for trips scheduled in a Google Calendar and delivers them via Telegram. It is designed to detect travel-related calendar events, extract relevant trip details such as destination and travel dates, retrieve weather forecasts and alerts for these trips, and send timely notifications to the user. The workflow also includes a scheduled update mechanism that triggers a second weather forecast request exactly one day before the trip starts, ensuring the forecast is as current and accurate as possible.

Logical blocks:

- **1.1 Event Trigger Block:** Listens for new or updated Google Calendar events.
- **1.2 Trip Identification Block:** Filters calendar events to identify trips using keyword matching.
- **1.3 Location Extraction Block:** Extracts travel destinations and date ranges from the trip events.
- **1.4 Forecast URL Construction Block:** Builds API request URLs for weather forecast retrieval.
- **1.5 Initial Weather Forecast Fetch Block:** Retrieves weather forecast data immediately after trip detection.
- **1.6 Forecast Message Formatting Block:** Converts raw weather data into a readable message.
- **1.7 Notification Delivery Block:** Sends the formatted forecast message via Telegram.
- **1.8 Scheduled Forecast Update Block:** Waits until one day before trip start to fetch and send updated forecasts.

---

### 2. Block-by-Block Analysis

#### 1.1 Event Trigger Block
- **Overview:** Captures Google Calendar events when they are created or updated, initiating the workflow.
- **Nodes Involved:** 
  - Event created
  - Event updated
- **Node Details:**

  - **Event created**
    - Type: Google Calendar Trigger
    - Role: Triggers workflow on new calendar events.
    - Configuration: Polls every minute on calendar "bara.razvan@gmail.com", triggers on event creation.
    - Inputs: None (trigger node)
    - Outputs: Passes event data downstream.
    - Failure modes: Google API authentication failure, network issues.
  
  - **Event updated**
    - Type: Google Calendar Trigger
    - Role: Triggers workflow on updated calendar events.
    - Configuration: Polls every minute on same calendar, triggers on event update.
    - Inputs: None (trigger node)
    - Outputs: Passes updated event data downstream.
    - Failure modes: Same as above.

#### 1.2 Trip Identification Block
- **Overview:** Filters calendar events to keep only those containing travel-related keywords.
- **Nodes Involved:** 
  - Identify trips
  - If1
- **Node Details:**

  - **Identify trips**
    - Type: Code node (JavaScript)
    - Role: Filters incoming events for travel-related trips.
    - Configuration: Filters events whose summary or description contains keywords: "trip", "travel", "vacation", "flight", "holiday" (case-insensitive).
    - Inputs: Events from triggers.
    - Outputs: Filtered trip events.
    - Failure modes: Empty or malformed event data; no trips found results in empty output.
  
  - **If1**
    - Type: If node
    - Role: Checks if any filtered trip events are present by verifying sequence count > 0.
    - Configuration: Condition: `$json.sequence > 0`.
    - Inputs: Output from Identify trips.
    - Outputs: Continues workflow only if trips detected; otherwise halts.
    - Failure modes: Expression evaluation errors if input malformed.

#### 1.3 Location Extraction Block
- **Overview:** Extracts trip start/end dates and destination locations from event data.
- **Nodes Involved:** 
  - Extract locations
- **Node Details:**

  - **Extract locations**
    - Type: Code node (JavaScript)
    - Role: Parses event fields to retrieve start and end datetime and infers destination location.
    - Configuration:
      - Tries to extract location from summary by matching "to <location>" pattern.
      - Falls back to description extracting after "to <location>".
      - If no match, uses event location field unless it starts with "from".
      - Defaults to null if no location found.
    - Inputs: Trips filtered by If1.
    - Outputs: Items with JSON containing `startDate`, `endDate`, and `location`.
    - Failure modes: Missing or malformed date/time or location fields; unexpected text formats.
    - Notes: Location extraction logic depends on natural language patterns and may fail for unusual event descriptions.

#### 1.4 Forecast URL Construction Block
- **Overview:** Builds Visual Crossing weather API URLs for each trip's location and date range.
- **Nodes Involved:** 
  - Build interogation URL
- **Node Details:**

  - **Build interogation URL**
    - Type: Code node (JavaScript)
    - Role: Constructs API request URL using extracted location, start date, end date, and API key.
    - Configuration:
      - Encodes location component.
      - Extracts date components (date only, no time).
      - Constructs URL for Visual Crossing Timeline API with metric units and requests daily data plus alerts.
      - API key placeholder `[your API]` must be replaced by user.
    - Inputs: Output from Extract locations.
    - Outputs: JSON with `url` property.
    - Failure modes: Missing location or dates; incorrect API key; URL encoding errors.
    - Notes: User must set their Visual Crossing API key in code.

#### 1.5 Initial Weather Forecast Fetch Block
- **Overview:** Sends HTTP GET request to Visual Crossing API to retrieve weather forecast for trip.
- **Nodes Involved:** 
  - Get Destination Weather forecast
- **Node Details:**

  - **Get Destination Weather forecast**
    - Type: HTTP Request node
    - Role: Fetches weather and alert data from Visual Crossing using constructed URL.
    - Configuration:
      - URL dynamically injected from previous node.
      - Default GET method with no additional options.
    - Inputs: URL from Build interogation URL.
    - Outputs: Raw JSON weather and alerts data.
    - Failure modes: Network failure, API rate limit exceeded, invalid API key, malformed URL.

#### 1.6 Forecast Message Formatting Block
- **Overview:** Processes raw weather data and formats a human-readable summary including alerts.
- **Nodes Involved:** 
  - Format message
- **Node Details:**

  - **Format message**
    - Type: Code node (JavaScript)
    - Role: Parses weather data, creates multi-line forecast summary string.
    - Configuration:
      - Includes date, max/min temperature, precipitation probability/type, weather conditions, and description for each day.
      - Appends any alerts descriptions at the end.
    - Inputs: Raw weather JSON from Get Destination Weather forecast or Get updated Destination Weather Forecast.
    - Outputs: JSON with `forecastSummary` string.
    - Failure modes: Missing expected JSON fields; empty or no alert data.
    - Notes: Produces a text summary suitable for messaging.

#### 1.7 Notification Delivery Block
- **Overview:** Sends the formatted weather forecast summary to the user on Telegram.
- **Nodes Involved:** 
  - Send Forecast
- **Node Details:**

  - **Send Forecast**
    - Type: Telegram node
    - Role: Sends text message containing forecast summary.
    - Configuration:
      - Operation set to "send".
      - Requires Telegram credentials and chat ID configured.
      - Receives `forecastSummary` text from Format message node.
    - Inputs: Formatted message JSON.
    - Outputs: Telegram API response.
    - Failure modes: Incorrect Telegram credentials, invalid chat ID, Telegram API downtime.

#### 1.8 Scheduled Forecast Update Block
- **Overview:** Waits until 24 hours before trip start date to fetch updated weather forecast and resend notification.
- **Nodes Involved:** 
  - Wait
  - Get updated Destination Weather Forecast
- **Node Details:**

  - **Wait**
    - Type: Wait node
    - Role: Pauses execution until one day before trip start datetime.
    - Configuration:
      - Resume at specific time calculated as startDate minus 24 hours.
      - Uses ISO string date-time expression based on `startDate` from Extract locations.
    - Inputs: URL from Build interogation URL (same as initial fetch).
    - Outputs: Triggers Get updated Destination Weather Forecast after wait.
    - Failure modes: Incorrect date calculation; time zone issues; node stuck if date in past.
  
  - **Get updated Destination Weather Forecast**
    - Type: HTTP Request node
    - Role: Re-fetches weather forecast from Visual Crossing one day before trip.
    - Configuration:
      - Uses same URL input as initial forecast fetch.
    - Inputs: URL from Build interogation URL (passed through Wait).
    - Outputs: Raw updated weather JSON.
    - Failure modes: Same as initial fetch.
    - Notes: Ensures user receives updated forecast closer to travel date.

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                         | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                          |
|-----------------------------------|------------------------|---------------------------------------|---------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Event created                     | Google Calendar Trigger| Trigger on event creation              | -                               | Identify trips                    | Trigger: EVENT Created or updated                                                                   |
| Event updated                    | Google Calendar Trigger| Trigger on event update                | -                               | Identify trips                    | Trigger: EVENT Created or updated                                                                   |
| Identify trips                   | Code                   | Filters events to detect trips         | Event created, Event updated     | If1                              | Identify if event is a trip by keywords; then extract destination and period                        |
| If1                             | If                     | Checks if trips exist                   | Identify trips                  | Extract locations                 |                                                                                                    |
| Extract locations               | Code                   | Extracts trip dates and location       | If1                            | Build interogation URL            |                                                                                                    |
| Build interogation URL          | Code                   | Builds Visual Crossing API request URL | Extract locations              | Wait, Get Destination Weather forecast | Get Forecast                                                                                   |
| Wait                           | Wait                   | Delays until one day before trip start | Build interogation URL          | Get updated Destination Weather Forecast | Update the forecast one day before the trip; Wait one day before the trip and request again   |
| Get Destination Weather forecast | HTTP Request           | Fetches initial weather forecast       | Build interogation URL          | Format message                   | Fetches weather forecast and alerts for trip location and dates from Visual Crossing               |
| Get updated Destination Weather Forecast | HTTP Request | Fetches updated weather forecast one day before trip | Wait                        | Format message                   | One day before the trip; fetches weather forecast and alerts for trip location and dates           |
| Format message                 | Code                   | Formats weather data into readable message | Get Destination Weather forecast, Get updated Destination Weather Forecast | Send Forecast                 |                                                                                                    |
| Send Forecast                 | Telegram                | Sends forecast summary via Telegram    | Format message                 | -                               | Sends the trip weather forecast summary to user via Telegram                                       |
| Sticky Note4                   | Sticky Note            | Documentation and workflow explanation | -                             | -                               | How it works: describes overall workflow, step-by-step, and optional notification alternatives      |
| Sticky Note1                   | Sticky Note            | Explains Identify trips block          | -                             | -                               | Identify if event is a trip; then extract destination and period                                   |
| Sticky Note6                   | Sticky Note            | Notes on Get Forecast block             | -                             | -                               | Get Forecast                                                                                       |
| Sticky Note3                   | Sticky Note            | Notes on Send notification block        | -                             | -                               | Make sure Telegram credentials and chat ID are set in the node                                    |
| Sticky Note8                   | Sticky Note            | Notes on forecast update block           | -                             | -                               | Update the forecast one day before the trip; Wait one day before the trip and request again       |
| Sticky Note                    | Sticky Note            | Notes on triggers                      | -                             | -                               | Trigger: EVENT Created or updated                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger Nodes:**
   - Create two Google Calendar Trigger nodes named "Event created" and "Event updated".
   - Configure both to poll every minute for calendar ID "bara.razvan@gmail.com".
   - Set "Event created" to trigger on event creation.
   - Set "Event updated" to trigger on event update.
   - Ensure Google Calendar credentials are configured with required scopes.

2. **Create "Identify trips" Code Node:**
   - Type: Code (JavaScript)
   - Paste the script filtering events for keywords: "trip", "travel", "vacation", "flight", "holiday" in event summary or description (case-insensitive).
   - Connect outputs of both calendar triggers to this node.

3. **Create "If1" Node:**
   - Type: If node
   - Add condition: Check if `$json.sequence > 0`.
   - Connect output of "Identify trips" to "If1".
   - Only continue if condition true.

4. **Create "Extract locations" Code Node:**
   - Type: Code (JavaScript)
   - Script extracts `startDate`, `endDate` from event start/end fields.
   - Tries to extract destination location from event summary, description, or location field, avoiding departure locations starting with "from".
   - Connect "If1" true output to this node.

5. **Create "Build interogation URL" Code Node:**
   - Type: Code (JavaScript)
   - Script builds Visual Crossing weather API URL using location and dates.
   - Replace `[your API]` with actual Visual Crossing API key.
   - Connect output of "Extract locations" to this node.

6. **Create "Wait" Node:**
   - Type: Wait
   - Configure to resume at specific time given by `startDate - 24 hours`.
   - Expression example: `={{ new Date(new Date($('Extract locations').item.json.startDate).getTime() - 24 * 60 * 60 * 1000).toISOString() }}`
   - Connect output of "Build interogation URL" to this node.

7. **Create "Get Destination Weather forecast" HTTP Request Node:**
   - Type: HTTP Request
   - Set method to GET.
   - URL parameter set dynamically from the `url` property of the input JSON.
   - Connect "Build interogation URL" output also directly to this node for initial fetch.

8. **Create "Get updated Destination Weather Forecast" HTTP Request Node:**
   - Type: HTTP Request
   - Same configuration as above.
   - Connect output of "Wait" node to this node.

9. **Create "Format message" Code Node:**
   - Type: Code (JavaScript)
   - Script formats weather data:
     - Includes daily max/min temperatures, precipitation probability and type, conditions, description.
     - Appends any alerts descriptions.
   - Connect outputs of both HTTP Request nodes ("Get Destination Weather forecast" and "Get updated Destination Weather Forecast") to this node.

10. **Create "Send Forecast" Telegram Node:**
    - Type: Telegram
    - Operation: Send message.
    - Configure Telegram credentials with appropriate bot token.
    - Set chat ID to target user or group.
    - Connect output of "Format message" to this node.

11. **Add Sticky Notes (optional but recommended):**
    - Add descriptive sticky notes explaining each logical block and node purpose for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses Visual Crossing Weather API, which provides 1000 free daily requests. Obtain a free API key from https://www.visualcrossing.com/weather-api.                                                               | Visual Crossing API                                                                                                                     |
| The Telegram node requires a bot token and chat ID. Create a Telegram bot via BotFather and get chat ID using tools like @userinfobot.                                                                                        | Telegram bot creation and chat ID retrieval                                                                                             |
| The workflow’s timing depends on accurate date parsing and timezone handling; ensure Google Calendar events have correct timezone info to avoid scheduling errors.                                                             | Timezone considerations                                                                                                                |
| The workflow can be extended by replacing the Telegram node with other messaging nodes like Email, WhatsApp, Slack, or SMS to increase notification coverage.                                                                  | Notification customization                                                                                                             |
| The natural language parsing for locations depends on event summaries/descriptions following expected patterns ("to <location>"). Consider enhancing parsing logic or adding manual overrides for non-standard event formats. | Location extraction logic                                                                                                              |
| Workflow is designed for a single Google Calendar but can be adapted to multiple calendars by duplicating triggers and merging outputs.                                                                                       | Multi-calendar support                                                                                                                 |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.