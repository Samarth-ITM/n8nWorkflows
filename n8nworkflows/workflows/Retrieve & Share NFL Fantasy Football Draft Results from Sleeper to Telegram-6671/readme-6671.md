Retrieve & Share NFL Fantasy Football Draft Results from Sleeper to Telegram

https://n8nworkflows.xyz/workflows/retrieve---share-nfl-fantasy-football-draft-results-from-sleeper-to-telegram-6671


# Retrieve & Share NFL Fantasy Football Draft Results from Sleeper to Telegram

---

### 1. Workflow Overview

This workflow enables Telegram chatbot users to retrieve their NFL Fantasy Football draft results from the Sleeper platform by simply sending their Sleeper username to the chatbot. It fetches detailed draft pick information including players’ names, positions, teams, and draft round/slot, then formats and sends this data back as a message in Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input Reception:** Listens for Telegram messages and extracts the Sleeper username entered by the user.
- **1.2 Sleeper User and Draft Data Retrieval:** Uses the extracted username to call Sleeper APIs, obtaining the user ID, associated drafts, and draft picks.
- **1.3 Draft Picks Processing and Formatting:** Filters draft picks for the requesting user, enriches the data with league info, and prepares a formatted message.
- **1.4 Telegram Output Delivery:** Sends the formatted draft results back to the user via Telegram.

Supporting the main flow are informational sticky notes providing usage instructions, tips, and warnings.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

**Overview:**  
This block captures incoming Telegram messages from users, extracts the Sleeper username from the message text, and triggers the subsequent API calls.

**Nodes Involved:**  
- Send Message to Chatbot (Telegram Trigger)  
- Extract Username (Code)

**Node Details:**

- **Send Message to Chatbot**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for Telegram messages sent to the chatbot  
  - Configuration: Monitors “message” updates; no user ID restrictions  
  - Inputs: Incoming Telegram message JSON  
  - Outputs: Passes message data to “Extract Username”  
  - Credentials: Requires configured Telegram API credentials (OAuth2 token)  
  - Edge Cases:  
    - Invalid or malformed Telegram message data  
    - Missing or expired Telegram credentials  
  - Sticky Notes: Reminder to set Telegram credentials before running  

- **Extract Username**  
  - Type: Code (JavaScript)  
  - Role: Parses the user’s message text to isolate the Sleeper username  
  - Configuration: Extracts text after the command prefix `/team `, trims whitespace  
  - Expressions: Uses `$json.message.text` to get the raw message string  
  - Inputs: Telegram message JSON from trigger node  
  - Outputs: JSON containing extracted `username` field  
  - Edge Cases:  
    - Empty or incorrectly formatted input (e.g., missing `/team ` prefix) results in empty username  
    - Case sensitivity in username affects API results  
  - Notes: Can be hardcoded if Telegram input is not desired  

#### 2.2 Sleeper User and Draft Data Retrieval

**Overview:**  
Fetches Sleeper user details by username, retrieves the user’s NFL drafts for the 2024 season, and extracts the relevant draft ID for subsequent pick retrieval.

**Nodes Involved:**  
- Get Sleeper User ID (HTTP Request)  
- Get Drafts (HTTP Request)  
- Extract Draft ID (Code)

**Node Details:**

- **Get Sleeper User ID**  
  - Type: HTTP Request  
  - Role: Fetches user info from Sleeper API using the provided username  
  - Configuration: GET request to `https://api.sleeper.app/v1/user/{{ $json.username }}`  
  - Inputs: `{ username }` from “Extract Username” node  
  - Outputs: User JSON including user_id  
  - Edge Cases:  
    - Username not found returns 404 or empty response, causing downstream failures  
    - API rate limits or downtime  
  - Sticky Notes: None specific here  

- **Get Drafts**  
  - Type: HTTP Request  
  - Role: Retrieves NFL drafts associated with the user ID for the 2024 season  
  - Configuration: GET request to `https://api.sleeper.app/v1/user/{{$json["user_id"]}}/drafts/nfl/2024`  
  - Inputs: `{ user_id }` from “Get Sleeper User ID”  
  - Outputs: Array of draft objects  
  - Edge Cases:  
    - Year hardcoded to 2024; requires manual update for future seasons  
    - User with no drafts returns empty array  
  - Sticky Notes: Reminder to update year for future seasons  

- **Extract Draft ID**  
  - Type: Code (JavaScript)  
  - Role: Extracts the first draft’s draft_id and owner_id from drafts list  
  - Configuration: Takes first element of drafts array, outputs draft_id and owner_id  
  - Inputs: Drafts array from “Get Drafts”  
  - Outputs: JSON with `draft_id` and `owner_id`  
  - Edge Cases:  
    - No drafts found leads to undefined properties, potential failures downstream  
    - User in multiple leagues always selects first draft; no user prompt or filtering implemented  
  - Sticky Notes: Warns about single draft selection logic and advises customization if multiple leagues  

#### 2.3 Draft Picks Processing and Formatting

**Overview:**  
Retrieves all draft picks from the identified draft, filters picks by the requesting user, enriches data with league metadata, and formats the results for Telegram output.

**Nodes Involved:**  
- Get Draft Picks (HTTP Request)  
- Return Picked By results from User ID w/ League year and format included (Code)  

**Node Details:**

- **Get Draft Picks**  
  - Type: HTTP Request  
  - Role: Fetches all draft picks for the specified draft_id  
  - Configuration: GET request to `https://api.sleeper.app/v1/draft/{{ $json.draft_id }}/picks`  
  - Inputs: `{ draft_id }` from “Extract Draft ID”  
  - Outputs: Array of draft pick objects  
  - Edge Cases:  
    - Invalid draft_id returns errors or empty picks list  
    - API downtime or rate limiting  
  - Sticky Notes: None  

- **Return Picked By results from User ID w/ League year and format included**  
  - Type: Code (JavaScript)  
  - Role: Filters picks to those made by the user, enriches with league name, scoring type, and season, prepares formatted JSON for output  
  - Configuration:  
    - Uses user_id and username from “Get Sleeper User ID”  
    - Extracts league metadata from “Get Drafts” node JSON  
    - Filters draft picks where `picked_by === user_id`  
    - Adds league and season info fields to each filtered pick  
  - Inputs: Picks array from “Get Draft Picks”, user and draft metadata references  
  - Outputs: Filtered and enriched draft pick JSON array  
  - Edge Cases:  
    - No picks found for user results in empty output  
    - Missing metadata fields default to “Unknown” values  
  - Sticky Notes: None  

#### 2.4 Telegram Output Delivery

**Overview:**  
Formats the draft results into a readable Telegram message and sends it to the user who requested the data.

**Nodes Involved:**  
- Send a text message (Telegram)

**Node Details:**

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends a formatted text message containing draft picks back to the Telegram chat  
  - Configuration:  
    - Message template incorporates season, league name, scoring type  
    - Lists picks with round, pick number, player name, position, and NFL team  
    - Uses data from previous node and chat ID from Telegram trigger message  
  - Inputs: Enriched draft picks JSON array and chat ID from “Send Message to Chatbot” trigger  
  - Outputs: Sends message to Telegram client, no further nodes  
  - Credentials: Requires Telegram API credentials setup  
  - Edge Cases:  
    - Messaging failures due to invalid chat ID or Telegram API errors  
    - Large message content might be truncated by Telegram limits  
  - Sticky Notes: Encourages customizing the message content  

---

### 3. Summary Table

| Node Name                                          | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)                               | Sticky Note                                                                                                      |
|----------------------------------------------------|---------------------|----------------------------------------|-----------------------------|----------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Sticky Note                                        | Sticky Note         | Informational overview and instructions| -                           | -                                            | This chatbot will give you the ability to pull your Sleeper team drafted players & position & team by typing your username into a chatbot in Telegram. Ensure Telegram credentials are set via your n8n credentials panel before running. Be sure to check your Sleeper App for your username & enter your username including upper or lower case letter. If entered incorrectly, no results will present 0 results back.  |
| Send Message to Chatbot                            | Telegram Trigger    | Telegram message listener (entry point)| -                           | Extract Username                             | Reminder to configure Telegram credentials                                                                       |
| Extract Username                                  | Code                | Extracts Sleeper username from message | Send Message to Chatbot      | Get Sleeper User ID                          | Can be hardcoded if Telegram input is not desired                                                               |
| Get Sleeper User ID                               | HTTP Request        | Fetch Sleeper user info by username    | Extract Username            | Get Drafts                                   | -                                                                                                                |
| Sticky Note4                                      | Sticky Note         | Notes on league selection logic        | -                           | -                                            | Please note that this pulls the first Sleeper league found. For users in multiple leagues, customize logic to prompt selection or filter by name. |
| Get Drafts                                        | HTTP Request        | Get user NFL drafts for 2024 season    | Get Sleeper User ID         | Extract Draft ID                             | Please note this GET address will reference 2024 and will probably need to be changed for 2025 after completion of your draft results |
| Extract Draft ID                                  | Code                | Extracts draft_id and owner_id from drafts | Get Drafts                  | Get Draft Picks                              | Please note that this pulls the first Sleeper league found. For users in multiple leagues, customize logic to prompt selection or filter by name. |
| Get Draft Picks                                   | HTTP Request        | Retrieve all picks for draft_id         | Extract Draft ID            | Return Picked By results from User ID w/ League year and format included | This will need update for upcoming years                                                                          |
| Return Picked By results from User ID w/ League year and format included | Code                | Filters and enriches picks by user     | Get Draft Picks             | Send a text message                          | -                                                                                                                |
| Send a text message                               | Telegram            | Sends formatted draft results to user  | Return Picked By results... | -                                            | Message back to your Telegram chatbot. You can change some of the text & information messaging if you want to be creative. |
| Sticky Note1                                      | Sticky Note         | Reminder about API year hardcoding      | -                           | -                                            | Please note this GET address will reference 2024 and will probably need to be changed for 2025 after completion of your draft results |
| Sticky Note2                                      | Sticky Note         | Suggestion to customize Telegram message| -                           | -                                            | Message back to your Telegram chatbot. You can change some of the text & information messaging if you want to be creative. |
| Sticky Note5                                      | Sticky Note         | General instructions and requirements   | -                           | -                                            | # Sleeper NFL Get Draft Results You will need the following to run: Telegram Chatbot via the BotFather or use any Trigger for your messaging you would prefer. An active Sleeper account and team. If you have multiple leagues/drafts this workflow will pull the most recent. This should give you a way to look up your team by username & basic information on players in your rosters using Sleeper App. |
| Sticky Note6                                      | Sticky Note         | Suggests hardcoding username alternative | -                           | -                                            | However, if you want to hardcode a username instead of using Telegram, you can modify this node directly.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node: "Send Message to Chatbot"**  
   - Type: Telegram Trigger  
   - Configure to listen for “message” updates  
   - Set webhook ID as generated by n8n  
   - Add credentials for Telegram API with valid Bot token  
   - No user ID filtering for general access  

2. **Create Code Node: "Extract Username"**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const message = $json.message.text || '';
     const username = message.replace('/team ', '').trim();
     return [{ json: { username } }];
     ```  
   - Connect input from “Send Message to Chatbot” node  

3. **Create HTTP Request Node: "Get Sleeper User ID"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.sleeper.app/v1/user/{{ $json.username }}`  
   - Connect input from “Extract Username” node  

4. **Create HTTP Request Node: "Get Drafts"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.sleeper.app/v1/user/{{$json["user_id"]}}/drafts/nfl/2024`  
   - Connect input from “Get Sleeper User ID” node  
   - Note: Update year manually for future seasons  

5. **Create Code Node: "Extract Draft ID"**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const drafts = $input.all();
     const draft = drafts[0].json;
     return [{ json: { draft_id: draft.draft_id, owner_id: draft.user_id } }];
     ```  
   - Connect input from “Get Drafts”  

6. **Create HTTP Request Node: "Get Draft Picks"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.sleeper.app/v1/draft/{{ $json.draft_id }}/picks`  
   - Connect input from “Extract Draft ID”  

7. **Create Code Node: "Return Picked By results from User ID w/ League year and format included"**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const userNode = $node["Get Sleeper User ID"].json;
     const userId = userNode["user_id"];
     const username = userNode["username"] || "Unknown User";

     const draftData = $node["Get Drafts"].json;

     const leagueName = draftData.metadata?.name || "Unknown League";
     const scoringType = draftData.metadata?.scoring_type || "Unknown Scoring";
     const season = draftData.season || "Unknown Season";

     return items
       .filter(item => item.json.picked_by === userId)
       .map(item => {
         return {
           json: {
             ...item.json,
             username,
             league_name: leagueName,
             scoring_type: scoringType,
             season
           }
         };
       });
     ```  
   - Connect input from “Get Draft Picks”  

8. **Create Telegram Node: "Send a text message"**  
   - Type: Telegram  
   - Text parameter:  
     ```
     Your draft results
     From {{ $json.season }} {{ $json.league_name }} ({{ $json.scoring_type }}) season!

     Here are your picks:
     {{ 
       $input.all().map(item => 
         `• Round ${item.json.round}, Pick ${item.json.draft_slot}: (${item.json.pick_no} overall) ${item.json.metadata.first_name} ${item.json.metadata.last_name} (${item.json.metadata.position} - ${item.json.metadata.team})`
       ).join('\n') 
     }}
     ```  
   - Chat ID: `={{ $node["Send Message to Chatbot"].json.message.chat.id }}`  
   - Credentials: Use Telegram API credentials same as trigger  
   - Connect input from “Return Picked By results from User ID w/ League year and format included”  

9. **(Optional) Add Sticky Note Nodes**  
   - Add informational sticky notes at relevant points to capture instructions and warnings as per the original workflow  

10. **Verify Credentials**  
    - Ensure Telegram API credentials (bot token) are valid and tested  
    - No additional credentials required for Sleeper API (public endpoints)  

11. **Test the Workflow**  
    - Trigger the Telegram bot by sending `/team <SleeperUsername>` message  
    - Verify draft picks are returned correctly  
    - Update the 2024 year in “Get Drafts” for future seasons as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow requires a Telegram chatbot created via BotFather or another Telegram bot creation method.                                                                                                                                     | Telegram BotFather documentation                                                                   |
| Sleeper API endpoints used are public and do not require authentication, but are subject to rate limiting or availability issues.                                                                                                            | Sleeper API docs: https://docs.sleeper.app/                                                       |
| The workflow currently pulls the first NFL draft found for the user in 2024; users in multiple leagues should customize draft selection logic for best results.                                                                             | Sticky Note4 content                                                                               |
| Users can customize the Telegram response message template to include additional data or personalized text.                                                                                                                                  | Sticky Note2 content                                                                               |
| Year in API URLs is hardcoded to 2024; update manually for each new NFL season to ensure correct draft data retrieval.                                                                                                                        | Sticky Note1, Sticky Note                                                                        |
| Hardcoding the username in “Extract Username” node’s code is possible for testing or restricted use without Telegram integration.                                                                                                            | Sticky Note6                                                                                       |
| The workflow’s main use case is for fantasy football enthusiasts wanting quick access to their Sleeper draft data via Telegram chat.                                                                                                        | Overall workflow purpose                                                                           |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---