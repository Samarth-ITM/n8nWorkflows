Plant Care Reminder with OpenWeather, Sheets and Telegram

https://n8nworkflows.xyz/workflows/plant-care-reminder-with-openweather--sheets-and-telegram-9953


# Plant Care Reminder with OpenWeather, Sheets and Telegram

### 1. Workflow Overview

This workflow automates reminders for plant care actions such as watering and fertilizing, integrating data from Google Sheets, weather forecasts from OpenWeather, and user notifications via Telegram. It targets plant enthusiasts or caretakers who want timely, weather-aware notifications to maintain optimal plant health. The workflow is divided into two main logical blocks:

- **1.1 Main Scheduled Workflow:**  
  Runs daily at 9 AM, reads plant and settings data from Google Sheets, determines due care actions, checks weather conditions to adjust watering reminders, and sends notifications via Telegram.

- **1.2 Webhook Sub-Workflow:**  
  Handles confirmation responses from Telegram inline buttons, updating Google Sheets logs and plant care status accordingly, and returns a confirmation HTML page.

---

### 2. Block-by-Block Analysis

#### 1.1 Main Scheduled Workflow

**Overview:**  
Triggered daily at 9 AM, this block reads plant data and user settings, decides which plants require care today, integrates weather forecasts to delay watering if recent or upcoming rain is detected, and sends personalized Telegram messages with inline buttons for user confirmation.

**Nodes Involved:**  
- Schedule Trigger  
- Read settings  
- Vacation Mode (If)  
- Read plants  
- DecideDue (Code)  
- ¿Pending task? (If)  
- OpenWeather request (HTTP Request)  
- Set DecideTag (Set)  
- Set Weather Tag (Set)  
- Merge  
- WeatherGate (Code)  
- If (Check no pending tasks)  
- Send Final Message No Dues1 (Telegram)  
- Send Dues (Telegram)

---

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow daily at 9:00 AM  
  - Configuration: Trigger hour set to 9  
  - Input: None  
  - Output: Starts workflow chain  
  - Edge cases: Timezone issues if not aligned with user settings

- **Read settings**  
  - Type: Google Sheets  
  - Role: Reads user settings such as vacation_mode and timezone from a dedicated settings sheet  
  - Configuration: Reads sheet with ID and gid for "settings" tab  
  - Credentials: Google Sheets OAuth2  
  - Input: Trigger  
  - Output: Settings data JSON  
  - Edge cases: Missing or malformed settings could break vacation mode logic

- **Vacation Mode (If)**  
  - Type: If  
  - Role: Checks if vacation_mode is "on" to skip plant checks  
  - Condition: vacation_mode equals "on"  
  - Input: Settings data JSON  
  - Outputs: If true, halts plant checks; if false, proceeds to Read plants  
  - Edge cases: Case sensitivity, missing vacation_mode field

- **Read plants**  
  - Type: Google Sheets  
  - Role: Reads plant data including last care dates, frequencies, location, and preferences  
  - Configuration: Reads "plants" sheet by document and gid  
  - Credentials: Google Sheets OAuth2  
  - Input: From Vacation Mode if false  
  - Output: Plants data array JSON  
  - Edge cases: Missing columns, invalid coordinate data

- **DecideDue (Code)**  
  - Type: Code  
  - Role: Determines which plants require care actions today by comparing last action date with frequency, parses frequencies in days/weeks/months, and collects coordinates for weather checks if watering is due and weather_delay is enabled  
  - Key logic:  
    - Parses frequency strings ("7d", "2w", "1m") into days  
    - Computes days elapsed since last action  
    - Builds arrays: due (actions due) and toCheckWeather (unique lat/lon for watering with weather delay)  
  - Input: Plants data  
  - Output: JSON with `today`, `due` array, and `toCheckWeather` array  
  - Edge cases: Invalid date formats, missing frequency, no coordinates for weather check

- **¿Pending task? (If)**  
  - Type: If  
  - Role: Checks if there are any due actions (due array not empty)  
  - Condition: due array not empty  
  - Input: Output of DecideDue  
  - Outputs: If true, proceeds to weather check; if false, sends no-due message  
  - Edge cases: Empty or malformed due array

- **OpenWeather request (HTTP Request)**  
  - Type: HTTP Request  
  - Role: Fetches 16-period weather forecast for each coordinate needing weather verification  
  - Configuration: API endpoint `/data/2.5/forecast`, query parameters include lat, lon, appid (API key placeholder), units=metric, cnt=16  
  - Input: Coordinates from due actions with weather_delay  
  - Output: Weather forecast JSON  
  - Edge cases: Invalid API key, API limits, network timeouts, invalid lat/lon

- **Set DecideTag (Set)**  
  - Type: Set  
  - Role: Tags DecideDue output with source = "decide" for merge identification  
  - Input: DecideDue output (due list)  
  - Output: Tagged JSON  
  - Edge cases: None

- **Set Weather Tag (Set)**  
  - Type: Set  
  - Role: Tags OpenWeather response with source = "weather" for merge identification  
  - Input: OpenWeather request output  
  - Output: Tagged JSON  
  - Edge cases: None

- **Merge**  
  - Type: Merge  
  - Role: Combines DecideDue and OpenWeather outputs into single stream for weather decision logic  
  - Input: Tagged Decide and Weather JSON arrays  
  - Output: Combined input array  
  - Edge cases: Sync issues if one input delayed

- **WeatherGate (Code)**  
  - Type: Code  
  - Role: Analyzes merged data to decide if watering should be skipped or delayed based on recent/upcoming rain and plant thirst level; filters out skipped actions; formats output messages for Telegram  
  - Key logic:  
    - Computes rain in last 24h and forecast next 24h from weather data  
    - Applies thresholds (recentThresh, upcomingThresh) adjusted by thirst_level (low, med, high)  
    - Skips watering if rain thresholds exceeded and plant is outdoor unless indoor  
    - Outputs array of actions with messages and update flags for last_action date  
  - Input: Merged Decide + Weather data  
  - Output: Filtered due actions array or noop message if none  
  - Edge cases: Missing weather data, invalid thirst_level, edge timestamps

- **If (Check no pending tasks)**  
  - Type: If  
  - Role: Checks if WeatherGate output is noop (no actionable tasks)  
  - Condition: $json.noop === true  
  - Outputs: If true, sends final no-due Telegram message; if false, sends due notifications  
  - Edge cases: Edge cases in noop flag missing or false negatives

- **Send Final Message No Dues1 (Telegram)**  
  - Type: Telegram  
  - Role: Sends "No plants due today ✅" message to user chat  
  - Configuration: Uses Telegram bot credentials and chat ID placeholder  
  - Input: If node (no due tasks)  
  - Output: None  
  - Edge cases: Telegram API errors, invalid chat ID

- **Send Dues (Telegram)**  
  - Type: Telegram  
  - Role: Sends notification messages for each due plant care action with inline button "Mark as done" linking to webhook URL for confirmation  
  - Configuration:  
    - Text formatted using HTML parse mode  
    - Inline keyboard button URL dynamically includes plant_id and action parameters  
    - Reply markup disables webpage preview  
  - Input: If node (tasks due)  
  - Output: Telegram messages sent  
  - Edge cases: Telegram API rate limits, broken URLs, invalid placeholders in URL

---

#### 1.2 Webhook Sub-Workflow

**Overview:**  
Triggered by Telegram inline button presses, this block receives confirmation of plant actions, validates input, updates Google Sheets with new last_action dates, appends a log entry, and responds with a formatted HTML confirmation page.

**Nodes Involved:**  
- Webhook  
- Prepare Data (Code)  
- If1 (If)  
- Update Water (Google Sheets)  
- Update Fert (Google Sheets)  
- Append Log (Google Sheets)  
- Edit Fields (Set)  
- HTML (Code)  
- Respond to Webhook

---

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Entry point for confirmation URL from Telegram inline button  
  - Configuration: Path "plant-confirm" with response mode set to responseNode  
  - Input: HTTP request with query parameters `plant` and `action`  
  - Output: Raw query data JSON  
  - Edge cases: Missing or invalid query parameters, unauthorized access

- **Prepare Data (Code)**  
  - Type: Code  
  - Role: Validates query parameters, normalizes action, validates allowed actions, prepares update field name, and date  
  - Input: Webhook query JSON  
  - Output: JSON with validation status, plant id, action, date, and column name to update  
  - Edge cases: Invalid or missing id or action, disallowed actions

- **If1 (If)**  
  - Type: If  
  - Role: Branches based on action type to update correct Google Sheets column  
  - Condition: action equals "water"  
  - Outputs: True updates last_water; False updates last_fert  
  - Edge cases: Unsupported action types not handled here

- **Update Water (Google Sheets)**  
  - Type: Google Sheets  
  - Role: Updates last_water field for given plant ID with current date  
  - Configuration: Update operation on "plants" sheet using matching column "id"  
  - Credentials: Google Sheets OAuth2  
  - Input: If1 true branch  
  - Output: Updated sheet row  
  - Edge cases: Missing plant ID, sheet access errors

- **Update Fert (Google Sheets)**  
  - Type: Google Sheets  
  - Role: Updates last_fert field for given plant ID with current date  
  - Configuration: Similar to Update Water, but updates last_fert  
  - Input: If1 false branch  
  - Output: Updated sheet row  
  - Edge cases: Same as Update Water

- **Append Log (Google Sheets)**  
  - Type: Google Sheets  
  - Role: Logs action with timestamp, plant ID, action type, and message ID in "log" sheet  
  - Configuration: Append operation to "log" sheet  
  - Input: After updating plants sheet  
  - Output: Confirmation of log append  
  - Edge cases: Sheet access errors, data format

- **Edit Fields (Set)**  
  - Type: Set  
  - Role: Normalizes and prepares data fields for HTML response rendering  
  - Input: Prepare Data output  
  - Output: JSON fields for HTML node  
  - Edge cases: None

- **HTML (Code)**  
  - Type: Code  
  - Role: Generates an HTML confirmation page showing action completion details for user  
  - Input: Edit Fields output  
  - Output: JSON containing HTML body  
  - Edge cases: None

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends the generated HTML page as HTTP response to the webhook call  
  - Configuration: 200 status, content-type text/html; charset=utf-8  
  - Input: HTML node output  
  - Output: HTTP response  
  - Edge cases: Response errors, HTTP timeouts

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                                   | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                  |
|-------------------------|-----------------------|-------------------------------------------------|-----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger      | Starts daily workflow at 9:00 AM                  | None                        | Read settings               | ## Principal Workflow                                                                        |
| Read settings           | Google Sheets         | Reads user settings (vacation_mode, timezone)     | Schedule Trigger            | Vacation Mode               |                                                                                              |
| Vacation Mode           | If                    | Skips plant reads if vacation_mode is "on"        | Read settings               | Read plants (false), none (true) |                                                                                              |
| Read plants             | Google Sheets         | Reads plants data                                  | Vacation Mode (false)        | DecideDue                  |                                                                                              |
| DecideDue               | Code                  | Determines due plant actions and coords for weather | Read plants                 | ¿Pending task?              |                                                                                              |
| ¿Pending task?          | If                    | Checks if any plant care actions are due           | DecideDue                   | OpenWeather request (true), Send Final Message No Dues1 (false) |                                                                                              |
| OpenWeather request     | HTTP Request          | Fetches weather forecast for watering locations    | ¿Pending task? (true)        | Set Weather Tag             |                                                                                              |
| Set DecideTag           | Set                   | Tags decide output with source="decide"            | DecideDue                   | Merge                      |                                                                                              |
| Set Weather Tag         | Set                   | Tags weather output with source="weather"          | OpenWeather request          | Merge                      |                                                                                              |
| Merge                   | Merge                 | Combines decide and weather data for gating        | Set DecideTag, Set Weather Tag | WeatherGate               |                                                                                              |
| WeatherGate             | Code                  | Filters due actions based on rain and thirst level | Merge                      | If                        |                                                                                              |
| If                      | If                    | Checks if no due actions after weather gating       | WeatherGate                 | Send Final Message No Dues, Send Dues |                                                                                              |
| Send Final Message No Dues1 | Telegram           | Sends "No plants due today" message                  | If (true)                   | None                      |                                                                                              |
| Send Dues               | Telegram              | Sends actionable plant care messages with buttons   | If (false)                  | None                      |                                                                                              |
| Webhook                 | Webhook               | Receives user confirmation from Telegram button     | None                        | Prepare Data               | ## Sub-Workflow Webhook                                                                      |
| Prepare Data            | Code                  | Validates and normalizes webhook input               | Webhook                     | If1                       |                                                                                              |
| If1                     | If                    | Branches update path based on action type            | Prepare Data                | Update Water (true), Update Fert (false) |                                                                                              |
| Update Water            | Google Sheets         | Updates last_water field in plants sheet             | If1 (true)                  | Append Log                 |                                                                                              |
| Update Fert             | Google Sheets         | Updates last_fert field in plants sheet              | If1 (false)                 | Append Log                 |                                                                                              |
| Append Log              | Google Sheets         | Appends action log entry                              | Update Water, Update Fert   | Edit Fields                |                                                                                              |
| Edit Fields             | Set                   | Prepares data for HTML response                        | Append Log                  | HTML                      |                                                                                              |
| HTML                    | Code                  | Generates HTML confirmation page                      | Edit Fields                 | Respond to Webhook         |                                                                                              |
| Respond to Webhook      | Respond to Webhook    | Sends HTML response to user                            | HTML                       | None                      |                                                                                              |
| Sticky Note             | Sticky Note           | Notes for Sub-Workflow Webhook                         | None                        | None                      | ## Sub-Workflow Webhook                                                                      |
| Sticky Note1            | Sticky Note           | Notes for Principal Workflow                            | None                        | None                      | ## Principal Workflow                                                                        |
| Sticky Note2            | Sticky Note           | Setup & Configuration Guide with checklist and details| None                        | None                      | ## 🌱 Setup & Configuration Guide\n\nContains setup checklist and Google Sheets configuration details with usage notes and image |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 09:00 (hour 9).

2. **Create a Google Sheets node “Read settings”:**  
   - Operation: Read  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: settings sheet GID or tab name  
   - Authenticate using Google Sheets OAuth2 credential.

3. **Create an If node “Vacation Mode”:**  
   - Condition: Check if `vacation_mode` field equals "on" (case sensitive string).  
   - Output true: Stop or do not proceed to plant reading.  
   - Output false: Continue to read plants.

4. **Create a Google Sheets node “Read plants”:**  
   - Operation: Read  
   - Document ID: Same as above  
   - Sheet Name: plants sheet GID or tab name  
   - Authenticate with Google Sheets OAuth2.

5. **Create a Code node “DecideDue”:**  
   - Input: plants data  
   - Logic:  
     - Parse watering and fertilizing frequencies (e.g. "7d", "2w", "1m") into days  
     - Calculate days since last action  
     - Collect actions due today and coordinates for weather check if watering with weather_delay enabled  
   - Output: JSON with `today`, `due` array, and `toCheckWeather` array.

6. **Create an If node “¿Pending task?”:**  
   - Condition: Checks if `due` array is not empty.  
   - True branch: Proceed to weather check.  
   - False branch: Send no due message.

7. **Create an HTTP Request node “OpenWeather request”:**  
   - Method: GET  
   - URL: https://api.openweathermap.org/data/2.5/forecast  
   - Query parameters:  
     - lat, lon: from first element of `toCheckWeather` array (dynamic expression)  
     - appid: Your OpenWeather API key (credential or parameter)  
     - units: metric  
     - cnt: 16  
   - Note: Repeat or loop if multiple locations exist (advanced).

8. **Create two Set nodes “Set DecideTag” and “Set Weather Tag”:**  
   - Assign field `source` as "decide" for DecideDue output.  
   - Assign field `source` as "weather" for OpenWeather output.

9. **Create a Merge node:**  
   - Mode: Merge by incoming data  
   - Inputs: Set DecideTag and Set Weather Tag outputs.

10. **Create a Code node “WeatherGate”:**  
    - Logic:  
      - Combine Decide and Weather data  
      - Calculate rain in past 24h and forecast next 24h  
      - Apply rain thresholds depending on thirst level  
      - Skip watering if rain thresholds exceeded and plant is outdoor  
      - Output filtered due list with messages and update flags.

11. **Create an If node “If”:**  
    - Condition: Check if output has `noop` true (no actionable tasks).  
    - True branch: Connect to “Send Final Message No Dues1” (Telegram).  
    - False branch: Connect to “Send Dues” (Telegram).

12. **Create Telegram nodes:**  
    - “Send Final Message No Dues1”: Send static text "No plants due today ✅" to configured chat ID.  
    - “Send Dues”: Send dynamic messages with HTML parse mode and inline keyboard button linking to webhook for confirmation.

---

**Webhook Sub-Workflow Setup:**

13. **Create a Webhook node:**  
    - Path: plant-confirm  
    - Response mode: responseNode

14. **Create Code node “Prepare Data”:**  
    - Validates query parameters `plant` and `action`, normalizes actions, checks allowed actions, prepares update column name, and today’s date.

15. **Create If node “If1”:**  
    - Checks if action is "water".  
    - True: Connect to Update Water node.  
    - False: Connect to Update Fert node.

16. **Create Google Sheets nodes “Update Water” and “Update Fert”:**  
    - Operation: Update  
    - Document ID and Sheet Name: plants sheet  
    - Matching Column: id  
    - Update `last_water` or `last_fert` with today’s date accordingly.

17. **Create Google Sheets node “Append Log”:**  
    - Operation: Append  
    - Document ID and Sheet Name: log sheet  
    - Columns: ts (timestamp), plant_id, action, message_id (constructed from plant and action)

18. **Create Set node “Edit Fields”:**  
    - Pass id, action, today, and updated column name for rendering.

19. **Create Code node “HTML”:**  
    - Generates confirmation HTML page showing the recorded action and plant id.

20. **Create Respond to Webhook node:**  
    - Sends the HTML response back to the user.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Setup & Configuration Guide: Checklist for Google Sheets setup, Telegram bot credentials, OpenWeather API key, workflow import, manual test, and schedule activation. Includes detailed columns and example values for sheets. Also advises on Telegram HTML parse mode and inline keyboard URL formatting. Contains a helpful image for visual setup.                                                                                                                                                                                                                                       | Refer to Sticky Note2 in the workflow UI                                                        |
| Weather API requires valid latitude and longitude coordinates. The workflow depends on accurate coordinates to fetch relevant weather forecasts and conditionally delay watering.                                                                                                                                                                                                                                                                                                                                                                                                | Notes in OpenWeather request and WeatherGate code nodes                                         |
| Inline button URLs in Telegram messages must be customized with your project URL and chat ID placeholders replaced accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Send Dues node and Setup Guide notes                                                           |
| Vacation mode allows users to temporarily disable notifications, useful for holidays or breaks. Must be set to "on" or "off" in settings sheet.                                                                                                                                                                                                                                                                                                                                                                                                                                    | Vacation Mode node and Setup Guide                                                             |
| The workflow assumes the Google Sheets have proper schema and data types as per configuration, including date formats YYYY-MM-DD and frequency strings. Invalid or inconsistent data may cause errors or skipped notifications.                                                                                                                                                                                                                                                                                                                                                        | General workflow assumptions                                                                    |

---

_Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n integration and automation tool. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available._