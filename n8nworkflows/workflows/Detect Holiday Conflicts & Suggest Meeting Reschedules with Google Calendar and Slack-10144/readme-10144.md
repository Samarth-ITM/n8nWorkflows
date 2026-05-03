Detect Holiday Conflicts & Suggest Meeting Reschedules with Google Calendar and Slack

https://n8nworkflows.xyz/workflows/detect-holiday-conflicts---suggest-meeting-reschedules-with-google-calendar-and-slack-10144


# Detect Holiday Conflicts & Suggest Meeting Reschedules with Google Calendar and Slack

---

### 1. Workflow Overview

This workflow is designed to help distributed teams detect scheduling conflicts caused by public holidays in multiple countries and suggest alternative meeting times. It automatically checks next week’s calendar events against public holidays of selected countries, identifies conflicts, proposes reschedule dates avoiding holidays and weekends, and sends a consolidated digest message to a Slack channel.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Configuration Setup:** Initiates the process daily and sets key variables such as date ranges, country codes, Google Calendar ID, and Slack channel ID.

- **1.2 Public Holiday Retrieval Loop:** Iterates over each country code to fetch public holiday data from a public API and merges these results filtering only for the upcoming week.

- **1.3 Calendar Event Retrieval:** Fetches all Google Calendar events within the next week’s date range.

- **1.4 Conflict Detection:** Compares calendar events against the aggregated holiday list to find scheduling conflicts.

- **1.5 Conflict Handling & Suggestion Generation:** Checks if conflicts exist; if so, generates alternative reschedule suggestions avoiding holidays and weekends.

- **1.6 Slack Notification:** Formats the conflict and suggestion data into a readable Slack message and posts it to the configured Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration Setup

- **Overview:** Triggers the workflow once daily at 9:00 AM server time and initializes all configuration variables used throughout the workflow.

- **Nodes Involved:**  
  - `Daily Check`  
  - `Workflow Configuration`

- **Node Details:**

  - **Daily Check**  
    - Type: Schedule Trigger  
    - Configuration: Triggers once every day at 09:00 server time.  
    - Inputs: None (start node)  
    - Outputs: Triggers `Workflow Configuration` node.  
    - Edge Cases: Timezone mismatch might cause unexpected trigger times; adjust trigger time as needed.  

  - **Workflow Configuration**  
    - Type: Set  
    - Configuration: Defines variables including:  
      - `currentYear` (current year as number)  
      - `nextWeekStart` (ISO date string, 7 days from now)  
      - `nextWeekEnd` (ISO date string, 14 days from now)  
      - `countryCodes` (array of ISO country codes for holidays)  
      - `slackChannel` (Slack channel ID for posting)  
      - `calendarId` (Google Calendar ID to query events)  
    - Inputs: From `Daily Check`  
    - Outputs: Branches to `Loop Over Items` and `Get Next Week Calendar Events`  
    - Edge Cases: Incorrect country codes or calendar/channel IDs cause failures downstream; update carefully.

#### 1.2 Public Holiday Retrieval Loop

- **Overview:** Iterates over each country code to perform individual API calls fetching public holidays, then merges and filters the results to only keep holidays within the next week.

- **Nodes Involved:**  
  - `Loop Over Items`  
  - `Fetch Public Holidays`  
  - `Merge and Filter Next Week Holidays`

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Configuration: Iterates over the `countryCodes` array from `Workflow Configuration`.  
    - Inputs: From `Workflow Configuration`  
    - Outputs: Feeds each country code individually to `Fetch Public Holidays`  
    - Edge Cases: Large country code arrays could slow the workflow; batching ensures manageable requests.

  - **Fetch Public Holidays**  
    - Type: HTTP Request  
    - Configuration: Calls `https://date.nager.at/api/v3/PublicHolidays/{currentYear}/{countryCode}` for each country code.  
    - Inputs: From `Loop Over Items` (one country code per call)  
    - Outputs: JSON array of holidays for that country.  
    - Edge Cases: No API key needed; if API limit or downtime occurs, holiday data may be missing.

  - **Merge and Filter Next Week Holidays**  
    - Type: Code  
    - Configuration: Merges all holiday arrays from `Fetch Public Holidays` calls, filters holidays occurring between `nextWeekStart` and `nextWeekEnd`, and formats them with country names.  
    - Inputs: Collection of all holiday arrays from loop outputs  
    - Outputs: Single item with `holidays` array containing holiday date, name, and country code  
    - Key Expressions: Uses JavaScript Date comparisons and mapping of country codes to country names  
    - Edge Cases: Empty or malformed API responses might cause missing or incomplete holiday data.

#### 1.3 Calendar Event Retrieval

- **Overview:** Retrieves all Google Calendar events scheduled within the next week.

- **Nodes Involved:**  
  - `Get Next Week Calendar Events`

- **Node Details:**

  - **Get Next Week Calendar Events**  
    - Type: Google Calendar  
    - Configuration: Fetches all events between `nextWeekStart` and `nextWeekEnd` from the specified `calendarId`. Returns all events without limit.  
    - Inputs: From `Workflow Configuration`  
    - Outputs: List of calendar events as separate items  
    - Requirements: Google Calendar OAuth2 credentials must be configured in n8n  
    - Edge Cases: Insufficient permissions, invalid calendar ID, or API rate limits may cause errors or empty results.

#### 1.4 Conflict Detection

- **Overview:** Compares each calendar event’s date with the holidays to detect any scheduling conflicts.

- **Nodes Involved:**  
  - `Detect Holiday Conflicts`

- **Node Details:**

  - **Detect Holiday Conflicts**  
    - Type: Code  
    - Configuration:  
      - Inputs:  
        - 0: The single merged holiday list from `Merge and Filter Next Week Holidays`  
        - 1: The list of calendar events from `Get Next Week Calendar Events`  
      - Logic: Maps holidays for quick date lookup, then iterates over each event to check if its start date matches any holiday. Collects conflict details including event name, time, affected countries, holiday name(s), attendees, and event ID.  
      - Outputs: Single JSON object containing `conflicts` array and `totalConflicts` count.  
    - Edge Cases: Events without start times are skipped; partial or absent holiday data may reduce detection accuracy.

#### 1.5 Conflict Handling & Suggestion Generation

- **Overview:** Checks if any conflicts were found and, if so, generates suggested reschedule dates avoiding weekends and holidays.

- **Nodes Involved:**  
  - `Check If Conflicts Found`  
  - `Generate Reschedule Suggestions`

- **Node Details:**

  - **Check If Conflicts Found**  
    - Type: If  
    - Configuration: Checks if the `conflicts` array from `Detect Holiday Conflicts` is not empty.  
    - Inputs: From `Detect Holiday Conflicts`  
    - Outputs: Continues only on true (conflicts exist) branch  
    - Edge Cases: Empty conflict array stops the workflow from continuing to suggestion generation and Slack posting.

  - **Generate Reschedule Suggestions**  
    - Type: Code  
    - Configuration:  
      - Inputs: From `Check If Conflicts Found`  
      - Logic: For each conflict, finds the next available date (up to 30 days ahead) that is not a weekend or holiday, maintaining the original meeting time. Outputs enhanced conflict objects with suggested reschedule date/time.  
    - Edge Cases: If no suitable reschedule date is found within 30 days, defaults to 30th day.

#### 1.6 Slack Notification

- **Overview:** Formats the conflict and reschedule suggestion data into a Slack message and posts it to the specified Slack channel.

- **Nodes Involved:**  
  - `Format Slack Digest`  
  - `Post Slack Digest`

- **Node Details:**

  - **Format Slack Digest**  
    - Type: Code  
    - Configuration: Builds a multiline Slack message summarizing conflicts, including event details, affected countries, holidays, and reschedule suggestions. If no conflicts, posts a no-conflict message.  
    - Inputs: From `Generate Reschedule Suggestions`  
    - Outputs: Single item with Slack message text in `slackMessage` field  
    - Edge Cases: Formatting errors may cause message issues; empty conflict list handled gracefully.

  - **Post Slack Digest**  
    - Type: Slack  
    - Configuration:  
      - Posts the `slackMessage` to the Slack channel ID defined in `Workflow Configuration` using OAuth2 authentication.  
    - Inputs: From `Format Slack Digest`  
    - Requirements: Slack OAuth2 credentials with chat:write scope and channel access must be configured.  
    - Edge Cases: Incorrect Slack credentials or channel ID will cause posting failures.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                                    | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                  |
|-------------------------------|----------------------------|---------------------------------------------------|-------------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Daily Check                   | Schedule Trigger            | Triggers workflow daily at 09:00                   | None                                | Workflow Configuration             | **Purpose:** Triggers the workflow once every weekday morning.                              |
| Workflow Configuration        | Set                        | Defines key variables used throughout workflow     | Daily Check                        | Loop Over Items, Get Next Week Calendar Events | **Purpose:** Central place to define variables.                                               |
| Loop Over Items               | SplitInBatches             | Iterates over country codes to fetch holidays      | Workflow Configuration              | Fetch Public Holidays, Merge and Filter Next Week Holidays | **Purpose:** Iterates through each country code.                                            |
| Fetch Public Holidays         | HTTP Request               | Calls public holiday API for each country          | Loop Over Items                    | Loop Over Items                    | **Purpose:** Calls Nager.Date API for public holidays.                                       |
| Merge and Filter Next Week Holidays | Code                   | Merges and filters holidays to next week only      | Fetch Public Holidays (all items) | Detect Holiday Conflicts           | **Purpose:** Merges API results and filters to next week only.                              |
| Get Next Week Calendar Events | Google Calendar            | Retrieves all Google Calendar events for next week | Workflow Configuration             | Detect Holiday Conflicts           | **Purpose:** Reads all events in next week’s window from Google Calendar.                    |
| Detect Holiday Conflicts      | Code                       | Detects conflicts between events and holidays      | Merge and Filter Next Week Holidays, Get Next Week Calendar Events | Check If Conflicts Found           | **Purpose:** Compares event dates with holiday dates to find conflicts.                     |
| Check If Conflicts Found      | If                         | Continues workflow only if conflicts exist         | Detect Holiday Conflicts           | Generate Reschedule Suggestions    | **Purpose:** Guards the branch; continues only when conflicts exist.                        |
| Generate Reschedule Suggestions | Code                     | Suggests alternative meeting dates avoiding holidays and weekends | Check If Conflicts Found           | Format Slack Digest               | **Purpose:** Suggests next business day that is not a holiday/weekend.                      |
| Format Slack Digest           | Code                       | Creates a Slack message summarizing conflicts       | Generate Reschedule Suggestions    | Post Slack Digest                 | **Purpose:** Creates a readable Slack message.                                              |
| Post Slack Digest             | Slack                      | Posts the conflict digest message to Slack          | Format Slack Digest                | None                             | **Purpose:** Posts the digest to Slack.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Schedule Trigger** node named `Daily Check`.  
   - Set it to trigger daily at 09:00 (server time).  

2. **Add Configuration Variables:**  
   - Add a **Set** node named `Workflow Configuration`.  
   - Create variables:  
     - `currentYear` (number): `={{ new Date().getFullYear() }}`  
     - `nextWeekStart` (string): `={{ new Date(Date.now() + 7 * 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}`  
     - `nextWeekEnd` (string): `={{ new Date(Date.now() + 14 * 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}`  
     - `countryCodes` (array): `["US", "GB", "DE", "IN", "CN", "KR", "HK"]`  
     - `slackChannel` (string): Slack channel ID (e.g., `C09FB9QQQTX`)  
     - `calendarId` (string): Google Calendar ID to query  

3. **Loop Over Country Codes:**  
   - Add a **SplitInBatches** node named `Loop Over Items`.  
   - Connect `Workflow Configuration` to it.  
   - Configure to iterate over the `countryCodes` array.  

4. **Fetch Public Holidays for Each Country:**  
   - Add an **HTTP Request** node named `Fetch Public Holidays`.  
   - Connect `Loop Over Items` to it.  
   - Set URL to:  
     `=https://date.nager.at/api/v3/PublicHolidays/{{ $('Workflow Configuration').first().json.currentYear }}/{{ $json }}`  
     (where `$json` refers to the current country code in the loop)  
   - No authentication needed.  

5. **Loop Back to Continue Requests:**  
   - Connect `Fetch Public Holidays` back to `Loop Over Items` → ensures all country codes are fetched one by one.  

6. **Merge and Filter Holidays:**  
   - Add a **Code** node named `Merge and Filter Next Week Holidays`.  
   - Connect `Loop Over Items` (main output 0) to this node.  
   - Use JavaScript to combine all holiday arrays, filter those within `nextWeekStart` and `nextWeekEnd`, and map country codes to country names.  

7. **Get Google Calendar Events:**  
   - Add a **Google Calendar** node named `Get Next Week Calendar Events`.  
   - Connect `Workflow Configuration` main output to this node.  
   - Configure:  
     - Calendar ID: Use `calendarId` from `Workflow Configuration`.  
     - Time Min: `={{ $('Workflow Configuration').first().json.nextWeekStart }}T00:00:00Z`  
     - Time Max: `={{ $('Workflow Configuration').first().json.nextWeekEnd }}T23:59:59Z`  
     - Operation: Get All Events  
     - Return All: Yes  
   - Ensure Google OAuth2 credentials are set up.  

8. **Detect Conflicts Between Events and Holidays:**  
   - Add a **Code** node named `Detect Holiday Conflicts`.  
   - Connect `Merge and Filter Next Week Holidays` to input 0 and `Get Next Week Calendar Events` to input 1.  
   - Implement logic to compare event start dates with holidays and output conflict details.  

9. **Check if Conflicts Exist:**  
   - Add an **If** node named `Check If Conflicts Found`.  
   - Connect from `Detect Holiday Conflicts`.  
   - Condition: Check if `conflicts` array is not empty (`notEmpty`).  

10. **Generate Reschedule Suggestions:**  
    - Add a **Code** node named `Generate Reschedule Suggestions`.  
    - Connect the true path of `Check If Conflicts Found`.  
    - For each conflict, find next available weekday (not holiday or weekend), up to 30 days ahead.  

11. **Format Slack Digest Message:**  
    - Add a **Code** node named `Format Slack Digest`.  
    - Connect from `Generate Reschedule Suggestions`.  
    - Format a multi-line Slack message summarizing conflicts and suggestions.  

12. **Post to Slack Channel:**  
    - Add a **Slack** node named `Post Slack Digest`.  
    - Connect from `Format Slack Digest`.  
    - Configure channel to the `slackChannel` variable from `Workflow Configuration`.  
    - Use OAuth2 credentials with `chat:write` permission.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                     | Context or Link                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow helps global/distributed teams avoid scheduling meetings on regional public holidays by suggesting alternative dates.               | Template Overview sticky note                                                                                                  |
| The public holiday API used (`Nager.Date`) requires no API key or authentication and is free to use.                                              | Fetch Public Holidays sticky note                                                                                              |
| Google Calendar OAuth2 credentials must have read access to the calendar specified by `calendarId`.                                              | Get Next Week Calendar Events sticky note                                                                                      |
| Slack OAuth2 credentials must allow posting messages to the specified channel (`chat:write` scope).                                              | Post Slack Digest sticky note                                                                                                  |
| To customize, adjust `countryCodes`, `calendarId`, and `slackChannel` in the `Workflow Configuration` node only.                                | Workflow Configuration sticky note                                                                                             |
| Adjust the date range or suggestion logic by editing `nextWeekStart`, `nextWeekEnd`, or the code in `Generate Reschedule Suggestions`.           | Workflow Configuration & Generate Reschedule Suggestions sticky notes                                                         |
| Runs daily at 09:00 server time by default; can be changed to weekly or other schedule as needed.                                                 | Daily Check sticky note                                                                                                        |
| Slack message uses standard emoji shortcodes and multi-line formatting to enhance readability.                                                   | Format Slack Digest sticky note                                                                                                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---