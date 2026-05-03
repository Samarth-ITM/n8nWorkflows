View Fantasy Football Roster Details with Sleeper API and Telegram Chatbot

https://n8nworkflows.xyz/workflows/view-fantasy-football-roster-details-with-sleeper-api-and-telegram-chatbot-6655


# View Fantasy Football Roster Details with Sleeper API and Telegram Chatbot

### 1. Workflow Overview

This workflow enables a Telegram chatbot that allows users to retrieve detailed information about their fantasy football roster from the Sleeper platform. By typing a command with their Sleeper username into Telegram, users receive a message listing their team players, including player names, positions, and NFL teams.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by a Telegram chatbot message containing the user’s Sleeper username.
- **1.2 Username Extraction & User Identification:** Extracts the username from the chatbot message and retrieves the corresponding Sleeper user ID.
- **1.3 League and Roster Retrieval:** Fetches the first NFL league for the user (currently hardcoded for 2024), retrieves rosters for that league, and isolates the user’s roster.
- **1.4 Player Data Lookup:** Searches an external Airtable database for detailed player information based on the roster’s player IDs.
- **1.5 Response Composition and Delivery:** Combines player data and sends a formatted message back to the user via Telegram.
- **1.6 Informational Notes:** Several sticky notes provide guidance, warnings about configuration, and suggestions for customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for Telegram messages to the chatbot and triggers the workflow.
- **Nodes Involved:** 
  - Send Message to Chatbot

- **Node Details:**

  - **Send Message to Chatbot**
    - Type: Telegram Trigger node
    - Role: Initiates the workflow when a Telegram message is received.
    - Configuration: Listens for "message" updates from Telegram.
    - Credentials: Requires Telegram API credentials configured in n8n.
    - Input: Telegram message events.
    - Output: JSON containing the message text and chat metadata.
    - Potential Failures: Telegram API authentication errors, webhook misconfiguration, message format issues.

---

#### 1.2 Username Extraction & User Identification

- **Overview:** Extracts the Sleeper username from the Telegram message text and obtains the Sleeper user ID via Sleeper API.
- **Nodes Involved:** 
  - Extract Username
  - Get Sleeper User ID

- **Node Details:**

  - **Extract Username**
    - Type: Code node
    - Role: Parses the message text to isolate the username parameter after the command "/team".
    - Configuration: Uses JavaScript to strip the command and trim whitespace.
    - Expressions: `$json.message.text` → extracts text from Telegram message.
    - Input: Telegram message JSON.
    - Output: JSON with extracted `username`.
    - Edge Cases: User submits message without "/team" or with malformed username → may result in empty or invalid username.

  - **Get Sleeper User ID**
    - Type: HTTP Request node
    - Role: Queries Sleeper API `/v1/user/{username}` to retrieve user details by username.
    - Configuration: URL is dynamically constructed using the extracted username.
    - Input: JSON with `username`.
    - Output: JSON containing Sleeper user info including `user_id`.
    - Edge Cases: Invalid username returns 404 or empty results; API rate limits or downtime.

---

#### 1.3 League and Roster Retrieval

- **Overview:** Fetches the user's leagues for the 2024 NFL season, extracts the first league's ID, retrieves all rosters for that league, and finds the user's roster.
- **Nodes Involved:** 
  - Get Leagues
  - Extract League ID
  - Get Rosters
  - Find User Roster

- **Node Details:**

  - **Get Leagues**
    - Type: HTTP Request node
    - Role: Retrieves leagues for the user for NFL 2024 season.
    - Configuration: URL uses user_id from previous node; season year is hardcoded to 2024.
    - Input: JSON with `user_id`.
    - Output: JSON array of league objects.
    - Notes: Sticky Note1 warns the year must be updated annually.
    - Edge Cases: User with no leagues returns empty array; API issues.

  - **Extract League ID**
    - Type: Code node
    - Role: Selects the first league from the returned leagues.
    - Configuration: Extracts `league_id` and `owner_id` from the first league object.
    - Input: JSON array of leagues.
    - Output: JSON with `league_id` and `owner_id`.
    - Edge Cases: Empty leagues input leads to undefined `league_id`; should be handled.

  - **Get Rosters**
    - Type: HTTP Request node
    - Role: Retrieves all rosters for the extracted league.
    - Configuration: URL built with `league_id`.
    - Input: JSON with `league_id`.
    - Output: JSON array of rosters.
    - Edge Cases: League with no rosters returns empty; API errors.

  - **Find User Roster**
    - Type: Code node
    - Role: Finds the roster belonging to the user by matching `owner_id` with user's Sleeper `user_id`.
    - Configuration: Searches roster array, extracts player IDs, and builds an Airtable filter formula.
    - Inputs: Sleeper `user_id` from "Get Sleeper User ID" node; rosters array.
    - Outputs: JSON containing player IDs and Airtable filter formula.
    - Edge Cases: No matching roster throws an error; empty rosters; missing player IDs.

---

#### 1.4 Player Data Lookup

- **Overview:** Uses the player IDs to search an Airtable base for detailed player information.
- **Nodes Involved:** 
  - Search records

- **Node Details:**

  - **Search records**
    - Type: Airtable node
    - Role: Queries Airtable table "Fantasy Players Daily" filtering records by player IDs from roster.
    - Configuration: Uses a dynamic formula filter based on player IDs passed from previous node.
    - Inputs: JSON with `airtable_filter` formula string.
    - Credentials: Airtable API token required.
    - Outputs: Detailed player records matching the roster.
    - Notes: Sticky Note3 explains this requires a synced Airtable table updated by a separate Sleeper NFL Daily Sync workflow.
    - Edge Cases: No matching players found; expired or invalid Airtable credentials; formula syntax errors.

---

#### 1.5 Response Composition and Delivery

- **Overview:** Merges player data and sends a formatted message back to the Telegram chat.
- **Nodes Involved:** 
  - Merge
  - Send a text message

- **Node Details:**

  - **Merge**
    - Type: Merge node
    - Role: Combines the Telegram message trigger data with the player data for final output.
    - Configuration: Default merge mode (likely 'append' or 'pass-through').
    - Inputs: Two inputs - from Telegram trigger and Airtable search.
    - Outputs: Combined dataset for message formatting.
    - Edge Cases: Data mismatch if one input is empty.

  - **Send a text message**
    - Type: Telegram node
    - Role: Sends the roster summary message to the Telegram chat.
    - Configuration: Uses an expression to count players and list their full names, positions, and teams in a single message.
    - Inputs: Combined data from Merge node.
    - Credentials: Telegram API credentials required.
    - Output: Message sent to user’s chat.
    - Notes: Sticky Note2 encourages customization of the message content.
    - Edge Cases: Message too long for Telegram limits; invalid chat ID; Telegram API errors.

---

#### 1.6 Informational Notes

- **Overview:** Several sticky notes provide instructions, warnings, and suggestions.
- **Nodes Involved:** 
  - Sticky Note
  - Sticky Note1
  - Sticky Note2
  - Sticky Note3
  - Sticky Note4
  - Sticky Note5
  - Sticky Note6

- **Node Details:**

  - **Sticky Note** (at workflow top)
    - Content: Explains purpose and Telegram credential requirement; emphasizes correct username casing.
  
  - **Sticky Note1**
    - Content: Warns that the GET leagues URL is hardcoded for 2024 and must be updated yearly.

  - **Sticky Note2**
    - Content: Encourages customization of the Telegram message text.

  - **Sticky Note3**
    - Content: Advises that player_id fields depend on a separate Sleeper NFL Daily Sync workflow to keep Airtable updated; suggests Google Sheets alternative.

  - **Sticky Note4**
    - Content: Notes the logic only pulls the first league; users in multiple leagues require custom selection logic.

  - **Sticky Note5**
    - Content: Lists prerequisites: Telegram chatbot setup and Airtable with player data sync.

  - **Sticky Note6**
    - Content: Suggests hardcoding a username in the Extract Username node if Telegram is not used.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                           | Input Node(s)                  | Output Node(s)            | Sticky Note                                                                                                                               |
|-------------------------|--------------------|-----------------------------------------|-------------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Send Message to Chatbot  | Telegram Trigger    | Workflow entry on Telegram message      | —                             | Extract Username, Merge    | REPLACE_WITH_YOUR_TELEGRAMAPI_CREDENTIAL required                                                                                         |
| Extract Username         | Code               | Parse username from message text        | Send Message to Chatbot        | Get Sleeper User ID        | Can be hardcoded for testing instead of Telegram (Sticky Note6)                                                                           |
| Get Sleeper User ID      | HTTP Request       | Retrieve Sleeper user info by username  | Extract Username               | Get Leagues               | —                                                                                                                                         |
| Get Leagues              | HTTP Request       | Get user's NFL leagues for 2024         | Get Sleeper User ID            | Extract League ID          | Needs update yearly for new season (Sticky Note1)                                                                                        |
| Extract League ID        | Code               | Extract first league's ID                | Get Leagues                   | Get Rosters                | Only first league considered; customize for multiple leagues (Sticky Note4)                                                               |
| Get Rosters              | HTTP Request       | Get all rosters for the league           | Extract League ID             | Find User Roster           | —                                                                                                                                         |
| Find User Roster         | Code               | Find user's roster and build Airtable filter | Get Rosters, Get Sleeper User ID | Search records           | Throws error if no roster found                                                                                                           |
| Search records           | Airtable           | Search Airtable for player details      | Find User Roster              | Merge                     | Requires Airtable credentials; relies on synced player data (Sticky Note3)                                                                |
| Merge                   | Merge              | Combine Telegram trigger and player data | Search records, Send Message to Chatbot | Send a text message   | —                                                                                                                                         |
| Send a text message      | Telegram           | Send roster details back to Telegram    | Merge                        | —                         | REPLACE_WITH_YOUR_TELEGRAMAPI_CREDENTIAL required; message is customizable (Sticky Note2)                                                  |
| Sticky Note              | Sticky Note        | Informational guidance                   | —                             | —                         | Explains purpose and Telegram credential setup                                                                                            |
| Sticky Note1             | Sticky Note        | Warning about hardcoded year             | —                             | —                         | Warns GET leagues URL is fixed for 2024                                                                                                  |
| Sticky Note2             | Sticky Note        | Message customization suggestion         | —                             | —                         | Suggests editing message text                                                                                                             |
| Sticky Note3             | Sticky Note        | Airtable sync dependency note            | —                             | —                         | Explains need for Sleeper NFL Daily Sync workflow or alternative data source                                                              |
| Sticky Note4             | Sticky Note        | League selection limitation note         | —                             | —                         | Notes first league only logic and need for multi-league customization                                                                     |
| Sticky Note5             | Sticky Note        | Prerequisite list                         | —                             | —                         | Lists Telegram and Airtable setup requirements                                                                                            |
| Sticky Note6             | Sticky Note        | Username hardcoding suggestion            | —                             | —                         | Suggests modifying Extract Username node for hardcoded username                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node ("Send Message to Chatbot")**
   - Type: Telegram Trigger
   - Configuration: Listen for "message" updates.
   - Credentials: Set your Telegram API credentials in n8n.
   - Position: Start node.
   - Purpose: Trigger workflow on incoming Telegram messages.

2. **Create Code node ("Extract Username")**
   - Type: Code (JavaScript)
   - Connect input from Telegram Trigger node.
   - Code snippet:
     ```javascript
     const message = $json.message.text || '';
     const username = message.replace('/team ', '').trim();
     return [{ json: { username } }];
     ```
   - Purpose: Extract the Sleeper username from the Telegram command message.
   - Optional: For testing, hardcode username directly here.

3. **Create HTTP Request node ("Get Sleeper User ID")**
   - Connect input from "Extract Username".
   - Method: GET
   - URL: `https://api.sleeper.app/v1/user/{{ $json.username }}`
   - Purpose: Retrieve Sleeper user details by username.

4. **Create HTTP Request node ("Get Leagues")**
   - Connect input from "Get Sleeper User ID".
   - Method: GET
   - URL: `https://api.sleeper.app/v1/user/{{$json["user_id"]}}/leagues/nfl/2024`
   - Note: Change "2024" for future seasons.
   - Purpose: Get user's NFL leagues for the specified year.

5. **Create Code node ("Extract League ID")**
   - Connect input from "Get Leagues".
   - Code snippet:
     ```javascript
     const leagues = $input.all();
     const league = leagues[0].json; // first league only
     return [{ json: { league_id: league.league_id, owner_id: league.user_id } }];
     ```
   - Purpose: Select first league from list.

6. **Create HTTP Request node ("Get Rosters")**
   - Connect input from "Extract League ID".
   - Method: GET
   - URL: `https://api.sleeper.app/v1/league/{{$json["league_id"]}}/rosters`
   - Purpose: Retrieve all rosters for the league.

7. **Create Code node ("Find User Roster")**
   - Connect input from "Get Rosters".
   - Also access data from "Get Sleeper User ID" via expression `$items('Get Sleeper User ID')[0].json.user_id`.
   - Code snippet:
     ```javascript
     const sleeperUserId = $items('Get Sleeper User ID')[0].json.user_id;
     const rosters = $input.all();
     const myRoster = rosters.find(roster => roster.json.owner_id === sleeperUserId);

     if (!myRoster) {
       throw new Error(`No roster found for Sleeper User ID: ${sleeperUserId}`);
     }

     const playerIds = myRoster.json.players || [];
     const formula = `OR(${playerIds.map(id => `{player_id} = \"${id}\"`).join(', ')})`;

     return [{
       json: {
         player_ids: playerIds,
         player_id: playerIds,        // for downstream usage
         airtable_filter: formula,
         sleeper_user_id: sleeperUserId
       }
     }];
     ```
   - Purpose: Identify the user's roster and prepare Airtable filter.

8. **Create Airtable node ("Search records")**
   - Connect input from "Find User Roster".
   - Operation: Search
   - Base: Select your Airtable base containing player data.
   - Table: Select appropriate table.
   - Filter by formula: `={{ $json.airtable_filter }}`
   - Credentials: Set Airtable API Token.
   - Purpose: Retrieve player details matching roster player IDs.

9. **Create Merge node ("Merge")**
   - Connect two inputs:
     - First input from "Search records"
     - Second input from "Send Message to Chatbot" (Telegram Trigger node)
   - Purpose: Combine Telegram context and player data for message formatting.

10. **Create Telegram node ("Send a text message")**
    - Connect input from "Merge".
    - Credentials: Use Telegram API credentials.
    - Chat ID: `={{ $node["Send Message to Chatbot"].json.message.chat.id }}`
    - Text (expression):
      ```
      =You have {{ $items().filter(i => i.json.full_name).length }} players on your roster:

      {{ $items()
        .filter(i => i.json.full_name)
        .map(i => `${i.json.full_name} (${i.json.position} - ${i.json.team})`)
        .join(", ") }}
      ```
    - Purpose: Send formatted roster summary back to the user.

11. **Add Sticky Notes as desired** for guidance and reminders:
    - About Telegram credentials and username casing.
    - About yearly league URL update.
    - Notes on Airtable data dependency.
    - Warning about single league selection logic.
    - Prerequisites for Telegram Bot and Airtable sync.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow requires Telegram credentials configured in n8n credentials panel.                                                                                               | Telegram API integration                                                                                         |
| Sleeper username must be entered exactly as in Sleeper app (case-sensitive).                                                                                              | User input via Telegram                                                                                            |
| The GET leagues URL is hardcoded for NFL 2024 season; update to next season as needed.                                                                                     | API URL: `https://api.sleeper.app/v1/user/{user_id}/leagues/nfl/2024`                                            |
| Player details search relies on Airtable base with synced player data. See "Sleeper NFL Players Daily Sync" workflow for syncing data daily.                             | Airtable integration, Sleeper NFL Daily Sync template                                                            |
| Alternative data sources like Google Sheets can replace Airtable if preferred.                                                                                            | Customization possibility                                                                                         |
| Current logic selects only the first league found; users with multiple leagues must customize to select correct league.                                                  | Multi-league user handling                                                                                        |
| Telegram message text can be customized in the "Send a text message" node to modify the roster summary format.                                                            | User experience customization                                                                                     |
| Telegram chatbot setup requires creating a bot via BotFather or another preferred messaging trigger.                                                                     | Telegram bot creation instructions                                                                                |
| Hardcoding username in "Extract Username" node can be used for manual testing without Telegram.                                                                           | Testing and debugging                                                                                             |
| Official Sleeper API documentation: https://docs.sleeper.app/                                                                                                            | Reference for API endpoints and data structures                                                                  |
| n8n official documentation: https://docs.n8n.io/                                                                                                                         | For node configuration and credentials setup                                                                     |

---

**Disclaimer:** The provided text is generated exclusively from an n8n automated workflow. All data handled is legal and public. The workflow respects content policies and contains no illegal or offensive elements.