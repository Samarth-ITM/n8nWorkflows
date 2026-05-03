Daily Import Yesterday's Forex Factory Calendar to Google Sheets with Telegram Alerts

https://n8nworkflows.xyz/workflows/daily-import-yesterday-s-forex-factory-calendar-to-google-sheets-with-telegram-alerts-9357


# Daily Import Yesterday's Forex Factory Calendar to Google Sheets with Telegram Alerts

### 1. Workflow Overview

This workflow automates the daily import of the previous day's Forex Factory economic calendar data into Google Sheets and sends alerts for high-impact news via Telegram. Its primary use case is for traders and analysts who want to monitor and archive Forex economic events categorized by their market impact.

The workflow logic is organized into the following blocks:

- **1.1 Daily Trigger & Date Preparation:** Initiates the workflow daily and prepares date parameters.
- **1.2 Month Name Conversion:** Converts textual month names into numeric values to format API requests.
- **1.3 Fetch Forex Factory Calendar:** Retrieves Forex Factory calendar data for the specified date via a RapidAPI HTTP request.
- **1.4 Impact-Based Filtering & Data Append:**
  - Separates events into high/medium impact and low impact.
  - Appends the categorized events into respective Google Sheets tabs.
- **1.5 Telegram Notification:** Aggregates high impact news and sends a consolidated message to a Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger & Date Preparation

- **Overview:**  
  This block triggers the workflow daily at a set hour and initializes the date context for the API calls.

- **Nodes Involved:**  
  - Schedule Trigger: Daily Triggers

- **Node Details:**

  - **Schedule Trigger: Daily Triggers**  
    - Type: Schedule Trigger  
    - Configuration: Triggers once daily at 08:00 (hour 8)  
    - Key expressions: None explicit; outputs current date/time used downstream  
    - Inputs: None (start node)  
    - Outputs: Provides date/time context including Year, Month name, and Day of month fields (used later)  
    - Edge cases: Workflow will not trigger if n8n instance is down or paused; time zone considerations may affect trigger time.

#### 1.2 Month Name Conversion

- **Overview:**  
  Converts the textual month name from the trigger's date data into a numeric month value for API query parameters.

- **Nodes Involved:**  
  - Convert month name to number

- **Node Details:**

  - **Convert month name to number**  
    - Type: Code (JavaScript)  
    - Configuration: Maps full month name (case-insensitive) to month number (1–12). Outputs JSON with original `Month` and `Month_Number`.  
    - Key expressions: Uses `$input.all()` to process all incoming items, applies mapping with capitalization normalization.  
    - Inputs: From Schedule Trigger node (date info)  
    - Outputs: JSON containing month name and numeric month  
    - Edge cases: If month name is missing or misspelled, `Month_Number` will be `null`, leading to potential API request failure.

#### 1.3 Fetch Forex Factory Calendar

- **Overview:**  
  Fetches the Forex Factory economic calendar data for the specified date using a third-party API.

- **Nodes Involved:**  
  - Get calendar from Forex Factory  
  - Sticky Note (API details)

- **Node Details:**

  - **Get calendar from Forex Factory**  
    - Type: HTTP Request  
    - Configuration:  
      - Uses RapidAPI endpoint `https://forex-factory-scraper1.p.rapidapi.com/get_real_time_calendar_details`.  
      - Query parameters include calendar type, year, month, day, currency (ALL), time format (24h), and timezone (GMT-05:00 Eastern Time).  
      - Headers include the required `x-rapidapi-host` for authentication.  
      - Authenticated via HTTP Header Auth credentials.  
    - Key expressions:  
      - Year from Schedule Trigger: `{{$('Schedule Trigger: Daily Triggers').item.json.Year}}`  
      - Month number from previous node: `{{$json.Month_Number}}`  
      - Day of month from Schedule Trigger: `{{$('Schedule Trigger: Daily Triggers').item.json['Day of month']}}`  
    - Inputs: From "Convert month name to number" node  
    - Outputs: JSON array of calendar events with fields such as News Title, year, date, time, currency, impact, actual, forecast, previous  
    - Version-specific: Uses n8n version supporting HTTP Request v4.2 and generic HTTP header auth  
    - Edge cases: API rate limiting, invalid or null month number, network errors, authentication failure with RapidAPI credentials  
    - Sticky Note attached explains API fields and usage.

#### 1.4 Impact-Based Filtering & Data Append

- **Overview:**  
  Filters the calendar events based on impact severity (high/medium vs. low) and appends the categorized data into two separate Google Sheets tabs.

- **Nodes Involved:**  
  - If  
  - Append to Google Sheets: High Impact  
  - Append to Google Sheets: Low Impact  
  - Filter High Impact News  
  - Aggregate  
  - Sticky Note (Google Sheets Append explanation)

- **Node Details:**

  - **If**  
    - Type: If (conditional split)  
    - Configuration: Checks if event's `impact` field contains "High" or "Medium" (case sensitive).  
    - Inputs: From API data node  
    - Outputs:  
      - True branch: Events with High or Medium impact  
      - False branch: Events with other impact levels (Low, Non-Economic, etc.)  
    - Edge cases: Impact field missing or empty leads to false branch; case sensitivity can cause misses if impact field varies.

  - **Append to Google Sheets: High Impact**  
    - Type: Google Sheets  
    - Configuration:  
      - Appends rows to the "calendar-High/Medium Impact" sheet tab (gid=0).  
      - Columns include date, time, year, actual, impact, currency, forecast, previous, and News Title mapped from JSON fields.  
      - Uses Service Account authentication.  
    - Inputs: True output from "If" node  
    - Outputs: Passes data to "Filter High Impact News" node for further filtering  
    - Edge cases: Google Sheets API quota limits, invalid credentials, sheet not found, malformed data.

  - **Filter High Impact News**  
    - Type: Filter  
    - Configuration: Keeps only events where `impact` contains "High". This filters out Medium from High Impact sheet results for Telegram notification.  
    - Inputs: From "Append to Google Sheets: High Impact" node  
    - Outputs: True branch goes to aggregation, false branch ignored  
    - Edge cases: Case sensitivity or missing impact field.

  - **Aggregate**  
    - Type: Aggregate  
    - Configuration: Aggregates the "News Title" fields from filtered high impact events into a collection for messaging.  
    - Inputs: True output from "Filter High Impact News"  
    - Outputs: Passes aggregated news titles to Telegram node  
    - Edge cases: Empty input leads to empty message.

  - **Append to Google Sheets: Low Impact**  
    - Type: Google Sheets  
    - Configuration:  
      - Appends rows for low impact events to the "calendar-Low Impact" sheet tab (gid=1954835598).  
      - Same columns mapping as High Impact sheet.  
      - Uses Service Account authentication.  
    - Inputs: False output from "If" node  
    - Outputs: None downstream  
    - Edge cases: Same as for High Impact sheet append.

  - **Sticky Note (Google Sheets Append Nodes)**  
    - Provides contextual explanation about the two Google Sheets append nodes and their purpose.

#### 1.5 Telegram Notification

- **Overview:**  
  Sends a Telegram message summarizing the high impact news events for the day.

- **Nodes Involved:**  
  - Telegram  
  - Sticky Note (Telegram usage)

- **Node Details:**

  - **Telegram**  
    - Type: Telegram node  
    - Configuration:  
      - Sends a message to a specified chat ID.  
      - Message content includes the previous day's date and the aggregated high impact news titles.  
      - Uses Telegram API credentials with OAuth2 authentication.  
    - Inputs: From "Aggregate" node  
    - Outputs: None (terminal node)  
    - Edge cases: Invalid chat ID, Telegram API rate limiting, network issues, empty message content  
    - Sticky Note instructs to add your Chat ID for the Telegram node.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                             | Input Node(s)                       | Output Node(s)                  | Sticky Note                                                                                       |
|-------------------------------|---------------------|--------------------------------------------|-----------------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger: Daily Triggers | Schedule Trigger    | Triggers workflow daily at 08:00           | None                              | Convert month name to number    |                                                                                                 |
| Convert month name to number    | Code                | Converts month name to numeric month value | Schedule Trigger: Daily Triggers  | Get calendar from Forex Factory |                                                                                                 |
| Get calendar from Forex Factory | HTTP Request        | Fetches Forex Factory calendar data        | Convert month name to number      | If                             | Explains Forex Factory API fields and usage                                                     |
| If                            | If                  | Splits events by impact (High/Medium vs Low) | Get calendar from Forex Factory   | Append to Google Sheets (High Impact), Append to Google Sheets (Low Impact) |                                                                                                 |
| Append to Google Sheets: High Impact | Google Sheets       | Appends high/medium impact events to sheet | If (True branch)                  | Filter High Impact News         | Explains Google Sheets append nodes purpose                                                    |
| Append to Google Sheets: Low Impact  | Google Sheets       | Appends low impact events to sheet         | If (False branch)                 | None                           | Explains Google Sheets append nodes purpose                                                    |
| Filter High Impact News        | Filter              | Filters only High impact news from high/medium | Append to Google Sheets: High Impact | Aggregate                     |                                                                                                 |
| Aggregate                     | Aggregate           | Aggregates high impact news titles          | Filter High Impact News           | Telegram                       |                                                                                                 |
| Telegram                     | Telegram            | Sends Telegram alert with high impact news  | Aggregate                        | None                           | Add your Chat ID to it. Sends High Impact News in one message.                                 |
| Sticky Note                   | Sticky Note         | Notes about Forex Factory API                | None                            | None                           |                                                                                                 |
| Sticky Note1                  | Sticky Note         | Notes about Google Sheets Append nodes       | None                            | None                           |                                                                                                 |
| Sticky Note2                  | Sticky Note         | Notes about Telegram node setup               | None                            | None                           |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Schedule Trigger: Daily Triggers`
   - Type: Schedule Trigger
   - Set to trigger daily at 08:00 (hour 8)
   - No credentials required.

3. **Add a Code node:**
   - Name: `Convert month name to number`
   - Type: Code
   - Connect from Schedule Trigger node.
   - Paste the JavaScript code to map month names to their numeric equivalent:

     ```javascript
     const items = $input.all();
     const monthMap = {
       January: 1, February: 2, March: 3, April: 4, May: 5, June: 6,
       July: 7, August: 8, September: 9, October: 10, November: 11, December: 12
     };
     const output = items.map(item => {
       const monthName = item.json.Month || "";
       const formattedName = monthName.trim().charAt(0).toUpperCase() + monthName.trim().slice(1).toLowerCase();
       const monthNumber = monthMap[formattedName] || null;
       return {
         json: {
           Month: monthName,
           Month_Number: monthNumber
         }
       };
     });
     return output;
     ```

4. **Add an HTTP Request node:**
   - Name: `Get calendar from Forex Factory`
   - Type: HTTP Request
   - Connect from `Convert month name to number`.
   - Set HTTP Method: GET
   - URL: `https://forex-factory-scraper1.p.rapidapi.com/get_real_time_calendar_details`
   - Authentication: HTTP Header Auth using RapidAPI credentials (setup required)
   - Add header parameter: `x-rapidapi-host` with value `forex-factory-scraper1.p.rapidapi.com`
   - Query parameters:
     - `calendar`: Forex
     - `year`: expression: `{{$('Schedule Trigger: Daily Triggers').item.json.Year}}`
     - `month`: expression: `{{$json.Month_Number}}`
     - `day`: expression: `{{$('Schedule Trigger: Daily Triggers').item.json['Day of month']}}`
     - `currency`: ALL
     - `time_format`: 24h
     - `timezone`: GMT-05:00 Eastern Time (US & Canada)

5. **Add an If node:**
   - Name: `If`
   - Connect from `Get calendar from Forex Factory`.
   - Condition: Check if `impact` field contains "High" OR contains "Medium" (case sensitive).
   - Use combinator OR with two string contains conditions.

6. **Add two Google Sheets nodes:**

   - **High Impact Sheet:**
     - Name: `Append to Google Sheets: High Impact`
     - Connect from `If` node's True output.
     - Operation: Append
     - Document ID: Use your Google Sheets ID for the Forex Factory calendar
     - Sheet Name: `calendar-High/Medium Impact` (gid=0)
     - Columns mapping: Map fields date, time, year, actual, impact, currency, forecast, previous, News Title from JSON.
     - Authentication: Service Account Google API credentials.

   - **Low Impact Sheet:**
     - Name: `Append to Google Sheets: Low Impact`
     - Connect from `If` node's False output.
     - Operation: Append
     - Document ID: Same as above
     - Sheet Name: `calendar-Low Impact` (gid=1954835598)
     - Same columns mapping as high impact.
     - Authentication: Service Account Google API credentials.

7. **Add a Filter node:**
   - Name: `Filter High Impact News`
   - Connect from `Append to Google Sheets: High Impact`.
   - Condition: `impact` contains "High" (case sensitive).

8. **Add an Aggregate node:**
   - Name: `Aggregate`
   - Connect from `Filter High Impact News`.
   - Aggregate the "News Title" field into a list.

9. **Add a Telegram node:**
   - Name: `Telegram`
   - Connect from `Aggregate`.
   - Set Chat ID to your Telegram chat.
   - Message text example:

     ```
     Forex factory Calendar: 
     Date: {{ new Date(Date.now() - 24*60*60*1000).toISOString().split('T')[0] }}

     The High Impact News:
     {{ $json['News Title'] }}
     ```

   - Use Telegram API credentials (OAuth2).

10. **Set node execution order accordingly and test the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Forex Factory API used via RapidAPI provides real-time economic calendar data with detailed fields.  | Sticky Note attached to "Get calendar from Forex Factory" node explains fields: News Title, year, date, etc.     |
| Two Google Sheets tabs are used to separate high/medium and low impact events for better data management. | Sticky Note near Google Sheets nodes explains this data organization.                                            |
| Telegram node requires adding your Telegram chat ID to receive notifications.                         | Sticky Note near Telegram node reminds to configure chat ID.                                                    |

---

**Disclaimer:**  
The text above is exclusively based on an automated workflow developed with n8n, adhering strictly to content policies without any illegal or offensive elements. All data handled is legal and publicly accessible.