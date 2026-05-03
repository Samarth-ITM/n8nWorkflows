Daily MLB Pitcher vs. Batter Matchup Analysis with Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/daily-mlb-pitcher-vs--batter-matchup-analysis-with-google-sheets-and-telegram-7252


# Daily MLB Pitcher vs. Batter Matchup Analysis with Google Sheets and Telegram

### 1. Workflow Overview

This n8n workflow automates a daily analysis of Major League Baseball (MLB) pitcher vs. batter matchups. Its main purpose is to:

- Retrieve the day’s MLB game schedule including probable pitchers and full lineups.
- Batch query detailed season statistics for all involved players.
- Construct comprehensive matchup rows combining pitcher and batter statistics.
- Filter and prioritize the top matchups based on pitcher ERA and batter OPS.
- Output the refined data to a Google Sheet and send a Telegram message highlighting the top 21 batters by OPS.

The workflow is structured into logical blocks:

- **1.1 Schedule & Player Data Retrieval:** Fetch daily games and extract all relevant player IDs.
- **1.2 Player Stats Aggregation:** Retrieve season stats for all identified players.
- **1.3 Matchup Construction:** Combine schedule and stats to create detailed pitcher-batter matchup rows.
- **1.4 Matchup Filtering & Sorting:** Apply ERA and OPS thresholds, select top pitchers and batters, and order data by game time.
- **1.5 Data Formatting & Sheet Update:** Enforce column order, data types, and update Google Sheets.
- **1.6 Telegram Notification:** Format and send a Telegram message with the top batters.
- **1.7 Scheduled Triggers & Maintenance:** Schedule workflow runs and clear the sheet daily.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule & Player Data Retrieval

- **Overview:**  
  Fetches the daily MLB schedule with probable pitchers and lineups, then extracts all unique player IDs involved in today's games.

- **Nodes Involved:**  
  - `11:02 - 8:02` (Schedule Trigger)  
  - `2. Get Daily Games` (HTTP Request)  
  - `3. Extract All Player IDs` (Code)

- **Node Details:**

  - **`11:02 - 8:02`**  
    - Type: Schedule Trigger  
    - Config: Cron expression `02 11-20 * * *` (server time), triggers hourly at minute 2 within hours 11 to 20.  
    - Role: Starts daily data retrieval process periodically during game hours.  
    - Failure: None typical; relies on server time correctness.

  - **`2. Get Daily Games`**  
    - Type: HTTP Request  
    - Config:  
      - URL: `https://statsapi.mlb.com/api/v1/schedule`  
      - Query: `sportId=1`, `date=today` (ISO), `hydrate=probablePitcher,lineups`  
      - Response: JSON  
    - Role: Retrieves today’s MLB game schedule with pitchers and lineups hydrated.  
    - Edge cases: Empty or delayed schedule data, network errors, API rate limits.

  - **`3. Extract All Player IDs`**  
    - Type: Code (JavaScript)  
    - Config:  
      - Extracts all player IDs from the schedule’s games, including probable pitchers and all lineup players from both teams.  
      - Outputs original schedule JSON with a concatenated string of all player IDs as `playerIdsString`.  
    - Variables: Uses `items[0].json.dates[0]?.games` as input, outputs `playerIdsString`.  
    - Failure: If no games or players found, returns empty which halts downstream processing.  
    - Notes: Node name is hardcoded in subsequent expressions (`$('3. Extract All Player IDs')`).

#### 1.2 Player Stats Aggregation

- **Overview:**  
  Queries season statistics for all players identified in the previous block.

- **Nodes Involved:**  
  - `4. Get Batched Player Stats` (HTTP Request)

- **Node Details:**

  - **`4. Get Batched Player Stats`**  
    - Type: HTTP Request  
    - Config:  
      - URL: `https://statsapi.mlb.com/api/v1/people`  
      - Query: `personIds` set to `{{$json.playerIdsString}}` from previous node, `hydrate=stats(group=[pitching,hitting,fielding],type=[season])`  
      - Fetches season stats grouped by pitching, hitting, and fielding for all players.  
    - Failure: API limits, invalid or empty personIds, network errors.

#### 1.3 Matchup Construction

- **Overview:**  
  Combines schedule data with player stats to create detailed matchup rows for each pitcher versus opposing batters.

- **Nodes Involved:**  
  - `5. Create Final Matchup Rows` (Code)

- **Node Details:**

  - **`5. Create Final Matchup Rows`**  
    - Type: Code (JavaScript)  
    - Config:  
      - Inputs: Player stats JSON (current node input), and schedule JSON (via reference to `3. Extract All Player IDs`).  
      - Logic: Iterates over games; for each game, creates matchup rows for home pitcher vs. away batters and away pitcher vs. home batters.  
      - Extracts key stats: ERA, strikeouts (SO) for pitchers; batting average (AVG), OPS, home runs (HR), hits, runs batted in (RBI) for batters.  
      - Outputs array of matchup objects, each representing one pitcher-batter pair with associated stats and metadata.  
    - Variables: Uses `.stats` arrays filtered by group display names for pitching/hitting.  
    - Failure: Empty inputs (no games or players), missing stats fields.  
    - Notes: Adds pitcherId and batterId to each row for update keying later.

#### 1.4 Matchup Filtering & Sorting

- **Overview:**  
  Filters matchups by minimum pitcher ERA and batter OPS, selects top 9 pitchers by ERA and top 3 batters by OPS per pitcher, and sorts final list by game start time.

- **Nodes Involved:**  
  - `6.  Filter for Top Matchups` (Code)

- **Node Details:**

  - **`6.  Filter for Top Matchups`**  
    - Type: Code (JavaScript)  
    - Config:  
      - Filters out rows with pitcher ERA ≤ 3.33 or invalid data.  
      - Sorts to find top 9 unique pitchers by ERA descending.  
      - For each pitcher, selects top 3 unique batters by OPS descending.  
      - Converts game start time to Eastern Time format (hh:mm AM/PM ET).  
      - Orders output objects according to a fixed header order.  
    - Failure: Incorrect or missing ERA/OPS, time parsing errors.  
    - Notes: Uses timezone 'America/New_York' explicitly for time formatting.

#### 1.5 Data Formatting & Sheet Update

- **Overview:**  
  Enforces column order, correct data typing (numbers, rounded averages), sorts rows, and appends or updates the Google Sheet with the filtered matchup data.

- **Nodes Involved:**  
  - `Column Order` (Code)  
  - `7. Update Your Sheet` (Google Sheets)

- **Node Details:**

  - **`Column Order`**  
    - Type: Code  
    - Config:  
      - Fixes column order to a defined list including gameDate, pitcher/batter info, and stats.  
      - Converts numeric fields to numbers, rounds batterAVG and batterOPS to three decimal places.  
      - Formats game start time to hh:mm AM/PM ET, cleaning trailing ":00".  
      - Sorts enriched data by game start time, pitcher ID, and opponent name.  
    - Failure: Date parsing issues, unexpected data formats.

  - **`7. Update Your Sheet`**  
    - Type: Google Sheets node  
    - Config:  
      - Operation: appendOrUpdate by `batterId` key  
      - Uses OAuth2 credentials for Google Sheets access  
      - Sheet and Document IDs configured in node parameters  
    - Failure: Authentication errors, API rate limits, invalid sheet names.

#### 1.6 Telegram Notification

- **Overview:**  
  Composes a Telegram message listing the top 21 batters by OPS and sends it to a specified chat via a Telegram bot.

- **Nodes Involved:**  
  - `8.  21 Hitters` (Code)  
  - `9. sendToTelegramChatbot` (Telegram)

- **Node Details:**

  - **`8.  21 Hitters`**  
    - Type: Code  
    - Config:  
      - Aggregates all batter records from input.  
      - Cleans and ensures batterHits is a number, filters invalid entries.  
      - Sorts batters descending by OPS.  
      - Selects top 21 batters.  
      - Constructs a Markdown-formatted Telegram message listing batters by rank, name, and OPS.  
    - Failure: No valid batters found, malformed input data.

  - **`9. sendToTelegramChatbot`**  
    - Type: Telegram node  
    - Config:  
      - Text: `{{$json.message}}` from previous node  
      - Chat ID: User-configured Telegram chat ID  
      - Credentials: Telegram API token for bot authentication  
    - Failure: Authentication errors, invalid chat ID, Telegram API errors.

#### 1.7 Scheduled Triggers & Maintenance

- **Overview:**  
  Defines scheduled triggers for workflow runs and clears the Google Sheet daily at 9:00 AM and 9:15 AM ET to maintain fresh data.

- **Nodes Involved:**  
  - `9am Clear` (Schedule Trigger)  
  - `Clear your Sheet` (Google Sheets)

- **Node Details:**

  - **`9am Clear`**  
    - Type: Schedule Trigger  
    - Config: Fires twice daily at 9:00 and 9:15 AM ET.  
    - Role: Initiates sheet clearing workflow.

  - **`Clear your Sheet`**  
    - Type: Google Sheets node  
    - Config: Clears entire configured sheet tab before daily data append.  
    - Credentials: Same Google Sheets OAuth2 as update node.  
    - Failure: Permissions or API errors.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                         | Input Node(s)                 | Output Node(s)               | Sticky Note                                        |
|---------------------------|--------------------|---------------------------------------|------------------------------|-----------------------------|---------------------------------------------------|
| 9am Clear                 | Schedule Trigger   | Trigger daily sheet clearing at 9 AM |                              | Clear your Sheet             |                                                   |
| Clear your Sheet          | Google Sheets      | Clears Google Sheet tab before update | 9am Clear                    |                             |                                                   |
| 11:02 - 8:02             | Schedule Trigger   | Hourly trigger during game hours      |                              | 2. Get Daily Games           |                                                   |
| 2. Get Daily Games        | HTTP Request       | Fetches MLB schedule with pitchers & lineups | 11:02 - 8:02               | 3. Extract All Player IDs    |                                                   |
| 3. Extract All Player IDs | Code               | Extracts all player IDs from schedule | 2. Get Daily Games           | 4. Get Batched Player Stats | "Keep node name \"3. Extract All Player IDs\" exact (used by $())" |
| 4. Get Batched Player Stats | HTTP Request     | Fetches season stats for all players  | 3. Extract All Player IDs    | 5. Create Final Matchup Rows |                                                   |
| 5. Create Final Matchup Rows | Code            | Combines schedule and stats into matchup rows | 4. Get Batched Player Stats | 6.  Filter for Top Matchups  |                                                   |
| 6.  Filter for Top Matchups | Code             | Filters and selects top pitcher-batter matchups | 5. Create Final Matchup Rows | Column Order                | "ET conversion is in code; cron uses server time" |
| Column Order              | Code               | Enforces column order and numeric typing | 6.  Filter for Top Matchups  | 7. Update Your Sheet         |                                                   |
| 7. Update Your Sheet      | Google Sheets      | Appends or updates data in Google Sheet | Column Order                | 8.  21 Hitters              |                                                   |
| 8.  21 Hitters            | Code               | Formats top 21 batters message for Telegram | 7. Update Your Sheet         | 9. sendToTelegramChatbot     |                                                   |
| 9. sendToTelegramChatbot  | Telegram           | Sends formatted message to Telegram chat | 8.  21 Hitters              |                             |                                                   |
| Sticky Note               | Sticky Note        | Visual annotation: "Hits"              |                              |                             | "Hits"                                            |
| Notes: MLB Hits           | Sticky Note        | Workflow overview and setup notes      |                              |                             | See notes in section 5 below                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Triggers:**

   - Node: `11:02 - 8:02`  
     - Type: Schedule Trigger  
     - Cron expression: `02 11-20 * * *` (server time)  
     - Purpose: Run hourly at minute 2 from 11:00 to 20:00 daily.

   - Node: `9am Clear`  
     - Type: Schedule Trigger  
     - Times: 9:00 and 9:15 AM Eastern Time (adjust cron accordingly for your server timezone)  
     - Purpose: Clear Google Sheet for fresh data daily.

2. **Fetch Daily Games:**

   - Node: `2. Get Daily Games`  
     - Type: HTTP Request  
     - URL: `https://statsapi.mlb.com/api/v1/schedule`  
     - Query Parameters:  
       - `sportId=1` (MLB)  
       - `date={{ $now.toFormat('yyyy-MM-dd') }}` (today’s date ISO format)  
       - `hydrate=probablePitcher,lineups` (include pitchers & lineups)  
     - Connect output of `11:02 - 8:02` to this node.

3. **Extract Player IDs:**

   - Node: `3. Extract All Player IDs`  
     - Type: Code  
     - JavaScript code: Extracts all pitcher and batter IDs from the schedule JSON, concatenates them into a comma-separated string `playerIdsString` and passes the original data forward.  
     - Connect output of `2. Get Daily Games` to this node.

4. **Fetch Player Stats:**

   - Node: `4. Get Batched Player Stats`  
     - Type: HTTP Request  
     - URL: `https://statsapi.mlb.com/api/v1/people`  
     - Query Parameters:  
       - `personIds={{$json.playerIdsString}}`  
       - `hydrate=stats(group=[pitching,hitting,fielding],type=[season])`  
     - Connect output of `3. Extract All Player IDs` to this node.

5. **Create Matchup Rows:**

   - Node: `5. Create Final Matchup Rows`  
     - Type: Code  
     - JavaScript code: Merges schedule and stats data to produce matchup rows for each pitcher vs opposing batters with detailed stats.  
     - Uses reference to `3. Extract All Player IDs` node for schedule data.  
     - Connect output of `4. Get Batched Player Stats` to this node.

6. **Filter and Select Top Matchups:**

   - Node: `6.  Filter for Top Matchups`  
     - Type: Code  
     - JavaScript code:  
       - Filters for pitcher ERA > 3.33.  
       - Selects top 9 pitchers by ERA and top 3 opposing batters by OPS each.  
       - Converts game start times to Eastern Time hh:mm AM/PM format.  
     - Connect output of `5. Create Final Matchup Rows` to this node.

7. **Enforce Column Order and Format Data:**

   - Node: `Column Order`  
     - Type: Code  
     - JavaScript code:  
       - Orders columns as per workflow spec.  
       - Converts certain fields to numbers and rounds AVG/OPS to 3 decimals.  
       - Sorts rows by game time, pitcher ID, and opponent name.  
     - Connect output of `6.  Filter for Top Matchups` to this node.

8. **Update Google Sheet:**

   - Node: `7. Update Your Sheet`  
     - Type: Google Sheets  
     - Operation: `appendOrUpdate`  
     - Key Column: `batterId`  
     - Credentials: Google Sheets OAuth2 with access to desired document and sheet tab.  
     - Connect output of `Column Order` to this node.

9. **Format Telegram Message:**

   - Node: `8.  21 Hitters`  
     - Type: Code  
     - JavaScript code:  
       - Aggregates all batter data.  
       - Sorts batters by OPS descending.  
       - Selects top 21 batters.  
       - Constructs Markdown message listing batters with rank, name, and OPS.  
     - Connect output of `7. Update Your Sheet` to this node.

10. **Send Telegram Message:**

    - Node: `9. sendToTelegramChatbot`  
      - Type: Telegram  
      - Text: `{{$json.message}}` (from preceding node)  
      - Credentials: Telegram Bot API token  
      - Chat ID: Your Telegram chat or group ID  
      - Connect output of `8.  21 Hitters` to this node.

11. **Clear Google Sheet Daily:**

    - Node: `Clear your Sheet`  
      - Type: Google Sheets  
      - Operation: `clear` sheet tab before daily data append  
      - Credentials: Same as update node  
      - Connect output of `9am Clear` to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| MLB "Hits" Workflow — Overview: Pulls daily MLB schedule with probable pitchers and lineups, batches season stats for all players, builds pitcher vs batter matchup rows, filters for ERA > 3.33, selects top 9 pitchers and top 3 batters each (27 rows), sorts by ET start time, writes to Google Sheets, and sends Telegram message with Top 21 batters by OPS. Setup includes Google Sheets OAuth2, Telegram Bot credentials, optional threshold tweaks, and runs triggered hourly and daily clearing at 9 AM. Key nodes and their roles are documented. Note: If lineups or schedule are not ready, downstream data may be empty (expected). ET conversion is done in code; triggers use server time. | Workflow sticky note inside n8n (node: Notes: MLB Hits)   |
| Google Sheets OAuth2 setup is required with access to your target spreadsheet and tab. Ensure API limits and permissions are correctly configured.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Google Sheets API documentation                           |
| Telegram Bot setup requires creating a bot with BotFather, acquiring the API token, and getting the chat ID where messages will be sent. Use the Telegram node with these credentials.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Telegram Bot API documentation                            |
| Timezone conversions use `America/New_York` explicitly to reflect Eastern Time for MLB games. Cron triggers use server time; adjust accordingly if your server is in a different timezone.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Timezone best practices                                   |
| Node name `3. Extract All Player IDs` must not be changed as it is referenced by code nodes via `$()` expressions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Important naming convention in n8n workflows              |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies with current content policies and contains no illegal or offensive material. All data processed is legal and publicly available.