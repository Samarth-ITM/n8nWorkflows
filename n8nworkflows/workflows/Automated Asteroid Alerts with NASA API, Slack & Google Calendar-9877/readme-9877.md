Automated Asteroid Alerts with NASA API, Slack & Google Calendar

https://n8nworkflows.xyz/workflows/automated-asteroid-alerts-with-nasa-api--slack---google-calendar-9877


# Automated Asteroid Alerts with NASA API, Slack & Google Calendar

### 1. Workflow Overview

This workflow automates the monitoring of near-Earth asteroids using NASA's Near-Earth Object Web Service API. It runs twice daily to fetch asteroid data for the upcoming 14 days, filters the results based on configurable thresholds for proximity and size, and alerts users via Slack and Google Calendar events about significant asteroid approaches.

The logical workflow is divided into the following blocks:

- **1.1 Scheduling & Date Range Calculation:** Triggers the workflow every 12 hours and calculates the date range for the NASA API query.
- **1.2 Data Retrieval:** Calls NASA's API to retrieve asteroid data for the specified date range.
- **1.3 Filtering & Processing:** Filters the retrieved asteroids according to distance and size thresholds, organizing relevant details.
- **1.4 Conditional Branching:** Checks if any asteroids meet filtering criteria and branches accordingly.
- **1.5 Alert Message Formatting:** Prepares detailed alert messages for Slack and email.
- **1.6 Notification Delivery:** Sends alerts to a selected Slack channel and creates individual Google Calendar events for each significant asteroid.
- **1.7 No-Results Handling:** Handles cases when no asteroids meet the criteria by doing nothing (but can be customized).

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Date Range Calculation

- **Overview:** Initiates the workflow every 12 hours and calculates the current date and a date 14 days ahead in the required format for the NASA API.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Calculate Date Range  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution on a 12-hour interval (twice daily).  
    - Configuration: Interval set to 12 hours.  
    - Input: None (trigger).  
    - Output: Triggers the next node (Calculate Date Range).  
    - Edge cases: Workflow may not trigger if n8n instance is down or scheduler misconfigured.

  - **Calculate Date Range**  
    - Type: Code node (JavaScript)  
    - Role: Calculates and formats today's date and the date 14 days ahead as YYYY-MM-DD strings.  
    - Key expressions:  
      - Uses native JavaScript Date API to create date range variables: `start_date`, `end_date`, `today`, and `next_week`.  
    - Input: Trigger from Schedule Trigger.  
    - Output: Outputs JSON with formatted dates for downstream NASA API node.  
    - Edge cases: Date calculations must consider timezone; uses server time by default.

---

#### 2.2 Data Retrieval

- **Overview:** Fetches near-Earth asteroid data from NASA's API using the calculated date range.
- **Nodes Involved:**  
  - Get an asteroid neo feed  

- **Node Details:**

  - **Get an asteroid neo feed**  
    - Type: NASA API node  
    - Role: Calls NASA’s Near-Earth Object Feed API for asteroid data between the start and end dates.  
    - Configuration: Uses credentials linked to a NASA API key (must be replaced from demo key).  
    - Input: Receives date range from Calculate Date Range.  
    - Output: Sends asteroid data to filtering node.  
    - Version requirements: Requires valid NASA API credentials.  
    - Edge cases: API rate limits, invalid/missing API key, network errors.  
    - Sticky note: Advises to replace `DEMO_KEY` with a personal API key from https://api.nasa.gov/.

---

#### 2.3 Filtering & Processing

- **Overview:** Filters asteroids based on proximity and size thresholds, formats and sorts the results.
- **Nodes Involved:**  
  - Filter and Process Asteroids  

- **Node Details:**

  - **Filter and Process Asteroids**  
    - Type: Code node (JavaScript)  
    - Role: Extracts asteroid data, checks distance and diameter thresholds, and returns filtered, sorted list.  
    - Configuration:  
      - Thresholds set as variables in code:  
        - `MAX_DISTANCE_KM = 75,000,000,000` (test default, very high)  
        - `MIN_DIAMETER_METERS = 1` (test default, very low)  
      - Filters asteroids that approach within max distance and exceed min diameter.  
      - Sorts filtered asteroids by closest approach distance.  
    - Input: Receives asteroid data from NASA API node.  
    - Output: Passes filtered asteroids to conditional check.  
    - Edge cases: Input data structure variations, missing fields, zero filtered results.  
    - Sticky note: Recommends adjusting thresholds to realistic values (e.g., 7.5 million km max distance, 100 meters min diameter).

---

#### 2.4 Conditional Branching

- **Overview:** Determines if any asteroids passed the filter and branches workflow for alerting or no-operation.
- **Nodes Involved:**  
  - Check If Asteroids Found  
  - No Operation, do nothing  

- **Node Details:**

  - **Check If Asteroids Found**  
    - Type: If node  
    - Role: Checks if the number of filtered asteroids > 0.  
    - Configuration: Condition compares the length of input items to zero.  
    - Input: Filtered asteroid list.  
    - Output:  
      - True: Proceed to formatting alert messages.  
      - False: Proceed to No Operation node.  
    - Edge cases: Miscount of items due to upstream issues.

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Ends workflow silently when no asteroids meet criteria.  
    - Edge cases: Can be replaced with notification logic for "All Clear" messages.  
    - Sticky note: Explains this node finishes the "no asteroids" branch silently.

---

#### 2.5 Alert Message Formatting

- **Overview:** Formats detailed alert messages listing all detected asteroids for Slack and email.
- **Nodes Involved:**  
  - Format Alert Messages  

- **Node Details:**

  - **Format Alert Messages**  
    - Type: Code node (JavaScript)  
    - Role: Creates a formatted summary message including asteroid names, approach dates, distances, sizes, speeds, hazard status, and NASA detail URLs.  
    - Configuration:  
      - Uses markdown for Slack message formatting.  
      - Constructs plain-text email body by stripping markdown characters.  
      - Includes hazard indicator icons.  
    - Input: Filtered asteroid data.  
    - Output: JSON object with `slackMessage`, `emailSubject`, `emailBody`, asteroid count, and asteroid details array.  
    - Edge cases: Empty input list should be blocked by previous If node.  
    - Output used by downstream Slack and Split nodes.

---

#### 2.6 Notification Delivery

- **Overview:** Sends alerts to Slack and creates Google Calendar events for each significant asteroid.
- **Nodes Involved:**  
  - Send Slack Alert  
  - Split Out Individual Asteroids  
  - Create an event  

- **Node Details:**

  - **Send Slack Alert**  
    - Type: Slack node  
    - Role: Posts the formatted alert message to a configured Slack channel.  
    - Configuration:  
      - Uses OAuth2 credentials for Slack authentication.  
      - Sends markdown-enabled text to a selected channel (e.g., #asteroid-alerts).  
    - Input: Formatted alert message from code node.  
    - Output: None (end of Slack branch).  
    - Edge cases: Slack authentication failure, permission issues in channel.  
    - Sticky note: Details setup instructions for Slack node including OAuth2 credentials and channel selection.

  - **Split Out Individual Asteroids**  
    - Type: SplitOut node  
    - Role: Splits the asteroid array to process each asteroid individually for calendar event creation.  
    - Configuration:  
      - Splits on `asteroids` field in input JSON.  
      - Includes additional fields: `emailSubject` and `slackMessage`.  
    - Input: Formatted alert messages node.  
    - Output: Each individual asteroid data item to Create an event node.  
    - Edge cases: Empty array input would result in no downstream events.

  - **Create an event**  
    - Type: Google Calendar node  
    - Role: Creates a calendar event for each asteroid with details about approach date, distance, size, speed, and URL.  
    - Configuration:  
      - Uses OAuth2 credentials for Google Calendar.  
      - Event start date set from asteroid’s full approach date/time.  
      - Event summary and description include asteroid data, formatted with localized numbers.  
      - Calendar selected from user account.  
    - Input: Individual asteroid JSON from SplitOut node.  
    - Output: None (end of workflow).  
    - Edge cases: Calendar permission errors, invalid date formats.  
    - Sticky note: Provides instructions for Google Calendar OAuth2 setup and event customization.

---

### 3. Summary Table

| Node Name                    | Node Type            | Functional Role                       | Input Node(s)             | Output Node(s)                         | Sticky Note                                                                                         |
|------------------------------|----------------------|------------------------------------|---------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger     | Initiates workflow every 12 hours  | None                      | Calculate Date Range                   |                                                                                                   |
| Calculate Date Range         | Code                 | Calculates date range for API      | Schedule Trigger          | Get an asteroid neo feed               |                                                                                                   |
| Get an asteroid neo feed     | NASA API             | Fetches asteroid data from NASA    | Calculate Date Range      | Filter and Process Asteroids           | Replace default `DEMO_KEY` with your own NASA API key: https://api.nasa.gov/                      |
| Filter and Process Asteroids | Code                 | Filters asteroids by distance/size | Get an asteroid neo feed  | Check If Asteroids Found               | Adjust filtering thresholds for sensitivity (distance and size)                                  |
| Check If Asteroids Found     | If                   | Checks if filtered asteroids exist | Filter and Process Asteroids | Format Alert Messages / No Operation  |                                                                                                   |
| Format Alert Messages        | Code                 | Formats alert messages for Slack   | Check If Asteroids Found  | Send Slack Alert, Split Out Individual Asteroids |                                                                                                   |
| Send Slack Alert             | Slack                | Sends alert message to Slack       | Format Alert Messages     | None                                  | Setup Slack OAuth2 credentials and channel permissions                                            |
| Split Out Individual Asteroids | SplitOut           | Splits asteroid list for calendar  | Format Alert Messages     | Create an event                       |                                                                                                   |
| Create an event              | Google Calendar      | Creates calendar events per asteroid | Split Out Individual Asteroids | None                               | Setup Google Calendar OAuth2 credentials and select calendar; customize event details             |
| No Operation, do nothing     | NoOp                 | Ends workflow if no asteroids found | Check If Asteroids Found  | None                                  | Can be replaced with "All Clear" message logic                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 12 hours.

2. **Create Calculate Date Range node**  
   - Type: Code  
   - Paste JavaScript code to calculate today’s date and 14 days later, formatted as YYYY-MM-DD.  
   - Connect Schedule Trigger → Calculate Date Range.

3. **Create Get an asteroid neo feed node**  
   - Type: NASA API node  
   - Configure resource: `asteroidNeoFeed`.  
   - Add NASA API credentials with a valid API key (replace demo key).  
   - Connect Calculate Date Range → Get an asteroid neo feed.

4. **Create Filter and Process Asteroids node**  
   - Type: Code  
   - Paste filtering JavaScript code which:  
     - Extracts asteroid data.  
     - Filters by `MAX_DISTANCE_KM` and `MIN_DIAMETER_METERS`.  
     - Sorts filtered results by distance.  
   - Adjust `MAX_DISTANCE_KM` and `MIN_DIAMETER_METERS` as desired.  
   - Connect Get an asteroid neo feed → Filter and Process Asteroids.

5. **Create Check If Asteroids Found node**  
   - Type: If node  
   - Condition: Check if `{{ $items().length }}` > 0.  
   - Connect Filter and Process Asteroids → Check If Asteroids Found.

6. **Create Format Alert Messages node**  
   - Type: Code  
   - Paste JavaScript code that formats a Slack message and email content from asteroid data.  
   - Connect Check If Asteroids Found (true branch) → Format Alert Messages.

7. **Create Send Slack Alert node**  
   - Type: Slack node  
   - Configure Slack OAuth2 credentials.  
   - Select or input the Slack channel ID for alerts.  
   - Set `Text` parameter to `={{ $json.slackMessage }}` and enable markdown.  
   - Connect Format Alert Messages → Send Slack Alert.

8. **Create Split Out Individual Asteroids node**  
   - Type: SplitOut  
   - Field to split: `asteroids`.  
   - Include fields: `emailSubject, slackMessage`.  
   - Connect Format Alert Messages → Split Out Individual Asteroids.

9. **Create Create an event node**  
   - Type: Google Calendar  
   - Configure Google Calendar OAuth2 credentials.  
   - Select calendar to use.  
   - Set event start date to `={{ $json.asteroids.approach_date_full }}`.  
   - Compose summary and description with asteroid info using expressions, e.g.:  
     - Summary: `小惑星接近アラート` (Asteroid Approach Alert)  
     - Description with distance, size, speed, and NASA URL.  
   - Connect Split Out Individual Asteroids → Create an event.

10. **Create No Operation, do nothing node**  
    - Type: NoOp  
    - Connect Check If Asteroids Found (false branch) → No Operation, do nothing.

11. **Add sticky notes** (optional)  
    - Place notes near relevant nodes to document API key replacement, filter thresholds, Slack and Calendar setup instructions, and no-results handling.

12. **Configure workflow settings**  
    - Set timezone (America/New_York).  
    - Enable execution data saving for debugging.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                           |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| Replace the default NASA API DEMO_KEY with your own key to avoid strict rate limits and enable regular data fetching.  | https://api.nasa.gov/                     |
| Adjust filtering criteria in the "Filter and Process Asteroids" node to balance alert sensitivity and noise.           | Sticky note on filtering thresholds      |
| Slack alerts require OAuth2 credentials with permissions to post in the chosen channel.                                 | Slack Setup sticky note                   |
| Google Calendar events are created with detailed asteroid info; OAuth2 credentials must be configured properly.         | Calendar Setup sticky note                |
| The "No Operation" node silently ends the workflow if no asteroids meet criteria; can be replaced by a daily "All Clear" message node. | No Results Note sticky note               |
| Markdown formatting is used in Slack messages to improve readability and highlight hazardous objects with icons.       | Format Alert Messages node code comments |

---

**Disclaimer:** The text and logic described derive exclusively from an automated n8n workflow. All data usage complies with relevant content policies and legal use.