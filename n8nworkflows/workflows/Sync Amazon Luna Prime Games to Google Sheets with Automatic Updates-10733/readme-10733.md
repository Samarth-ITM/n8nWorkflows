Sync Amazon Luna Prime Games to Google Sheets with Automatic Updates

https://n8nworkflows.xyz/workflows/sync-amazon-luna-prime-games-to-google-sheets-with-automatic-updates-10733


# Sync Amazon Luna Prime Games to Google Sheets with Automatic Updates

### 1. Workflow Overview

This workflow automates the synchronization of Amazon Luna Prime games that are "Included with Prime" into a Google Sheet, keeping the sheet updated with new game entries. It also provides optional notifications for newly discovered games.

The workflow is logically divided into three main blocks:

- **1.1 Initialization and Data Retrieval**  
  Sets HTTP request headers and parameters, triggers on schedule, fetches existing data from Google Sheets, and then queries Amazon Luna’s API to retrieve the current list of Prime games.

- **1.2 Data Parsing and Google Sheets Update**  
  Parses the Amazon Luna API response to extract detailed game information, compares the new data to existing sheet rows to identify new entries, and updates or appends game rows in Google Sheets accordingly.

- **1.3 New Game Notifications**  
  Filters for newly added games, downloads associated images, processes the data in batches with delays to respect rate limits, and sends notifications about new games via Discord (configurable to other platforms).

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Data Retrieval

**Overview:**  
This block prepares and executes the HTTP request to Amazon Luna’s API with the necessary headers and parameters. It is triggered on a schedule and begins by fetching existing rows from Google Sheets to enable comparison later.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  
- Edit Fields  
- HTTP Request1  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 5 days at 15:00  
  - Config: Interval set to every 5 days at 15:00 hours  
  - Inputs: None  
  - Outputs: “Get row(s) in sheet” node  
  - Edge cases: Missed triggers if workflow inactive; time zone inconsistencies  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves existing game data from the specified Google Sheet  
  - Config: Uses service account authentication; configured with document ID and sheet name  
  - Inputs: From Schedule Trigger  
  - Outputs: “Edit Fields” node  
  - Edge cases: Authorization errors; empty or missing sheet; rate limits  

- **Edit Fields**  
  - Type: Set (field assignment)  
  - Role: Sets HTTP header parameters required by Amazon Luna API request  
  - Config: Hardcoded headers such as Content-Type, Origin, Referer, x-amz-locale (en_US), x-amz-platform (web), device type (browser), marketplace ID (US), language, and user agent string  
  - Inputs: From “Get row(s) in sheet”  
  - Outputs: “HTTP Request1” node  
  - Edge cases: Incorrect or outdated header values could cause request failure or data format changes  

- **HTTP Request1**  
  - Type: HTTP Request (POST)  
  - Role: Queries Amazon Luna API endpoint to fetch the “Included with Prime” games list  
  - Config: POST to `https://proxy-prod.eu-west-1.tempo.digital.a2z.com/getPage` with JSON body specifying query parameters and client context (browser metadata, device info)  
  - Headers: Supplied dynamically from “Edit Fields” node  
  - Inputs: From “Edit Fields”  
  - Outputs: “parsing” node  
  - Edge cases: Network timeouts, API changes, invalid headers, rate limits  

---

#### 2.2 Data Parsing and Google Sheets Update

**Overview:**  
This block processes the raw API response, extracts relevant game details, identifies new entries by comparing with existing sheet rows, and updates the Google Sheet accordingly.

**Nodes Involved:**  
- parsing (Code)  
- isNew (Code)  
- If1 (If)  
- Append or update row in sheet  
- Sticky Note3 (documentation)

**Node Details:**

- **parsing**  
  - Type: Code (JavaScript)  
  - Role: Walks through the JSON response to extract “GAME_TILE” widgets containing game metadata such as ASIN, slug, title, release year, publishers, genres, product URL, images, and age rating  
  - Inputs: From “HTTP Request1”  
  - Outputs: “isNew”  
  - Key logic: Recursive traversal of nested JSON; JSON.parse on presentationData; maps extracted games to output array  
  - Edge cases: Malformed JSON, missing fields, changes in API response structure causing parsing failure  

- **isNew**  
  - Type: Code (JavaScript)  
  - Role: Flags each game as new or existing by comparing ASIN or title against rows fetched from Google Sheets  
  - Inputs: From “parsing” and “Get row(s) in sheet” (via static reference)  
  - Outputs: “If1”  
  - Logic: Builds a Set of existing keys (ASIN or title normalized to uppercase); marks item as new if key not found or missing  
  - Edge cases: Empty sheet (all considered new), case sensitivity issues, missing ASIN and title fallbacks  

- **If1**  
  - Type: If  
  - Role: Filters items to pass only those flagged as new (`isNew == true`)  
  - Inputs: From “isNew”  
  - Outputs: “Append or update row in sheet” (true branch)  
  - Edge cases: No new items results in no downstream execution  

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Role: Writes game data into Google Sheet; updates existing rows by title or appends new ones  
  - Config: Matching by `title` column; fields mapped include asin, isNew (set to FALSE here), title, genres, releaseYear, image URLs  
  - Inputs: From “If1”  
  - Outputs: “GetImage”  
  - Edge cases: Rate limits, authorization, data mismatches, row matching failures  

- **Sticky Note3**  
  - Content: Describes this block’s purpose: updating Google Sheet with new and existing games, matching by title to avoid duplicates  

---

#### 2.3 New Game Notifications

**Overview:**  
This block handles notification of newly added games. It downloads image assets, processes games in small batches respecting rate limits, adds wait periods, and sends notification messages via Discord.

**Nodes Involved:**  
- GetImage (HTTP Request)  
- Loop Over Items (SplitInBatches)  
- Wait  
- Send a message1 (Discord)  
- Sticky Note2 (documentation)

**Node Details:**

- **GetImage**  
  - Type: HTTP Request (GET)  
  - Role: Downloads the landscape image of new games by URL  
  - Inputs: From “Append or update row in sheet”  
  - Outputs: “Loop Over Items”  
  - Edge cases: Image URL invalid or expired, HTTP errors, bandwidth limits  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes new game items in batches of 6 to avoid hitting rate limits on notifications  
  - Inputs: From “GetImage”  
  - Outputs: Two branches: main (empty) and secondary to “Wait” node  
  - Edge cases: Batch size too large or too small may affect performance or API limits  

- **Wait**  
  - Type: Wait  
  - Role: Inserts a pause between batches to rate limit message sending  
  - Inputs: From “Loop Over Items” secondary output  
  - Outputs: “Send a message1”  
  - Edge cases: Workflow timeout if wait time too long, potential accumulation of delayed messages  

- **Send a message1**  
  - Type: Discord  
  - Role: Sends a notification message to a specified Discord channel about each new game added  
  - Config: Uses OAuth2 Discord credentials, guild and channel IDs set; message includes title, genres, release year pulled from the “If1” node’s JSON  
  - Inputs: From “Wait”  
  - Outputs: Loops back to “Loop Over Items” to continue batch processing  
  - Edge cases: Discord API rate limits, credential expiration, invalid channel or guild IDs  

- **Sticky Note2**  
  - Content: Explains notification section, default Discord usage, and suggests replacing the notification node with alternatives (Telegram, Slack, email, webhooks) while keeping the rest unchanged  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                             | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                               |
|-------------------------|-------------------------|---------------------------------------------|-------------------------|---------------------------|----------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger         | Periodically triggers the workflow          |                         | Get row(s) in sheet       | Section 1 – Amazon request parameters                                                                    |
| Get row(s) in sheet     | Google Sheets            | Fetches existing sheet rows for comparison  | Schedule Trigger         | Edit Fields               | Section 1 – Amazon request parameters                                                                    |
| Edit Fields             | Set                     | Sets HTTP request headers for Amazon Luna   | Get row(s) in sheet      | HTTP Request1             | Section 1 – Amazon request parameters                                                                    |
| HTTP Request1           | HTTP Request            | Fetches “Included with Prime” games data    | Edit Fields              | parsing                   | Section 1 – Amazon request parameters                                                                    |
| parsing                 | Code                    | Parses API response extracting game details | HTTP Request1            | isNew                     |                                                                                                          |
| isNew                   | Code                    | Flags new games not in Google Sheet          | parsing                  | If1                       |                                                                                                          |
| If1                     | If                       | Filters new games (isNew == true)            | isNew                    | Append or update row in sheet | Section 3 – New game notifications                                                                        |
| Append or update row in sheet | Google Sheets      | Updates or appends game rows in Google Sheet| If1                      | GetImage                  | Section 2 – Update Google Sheet                                                                           |
| GetImage                | HTTP Request            | Downloads game landscape images               | Append or update row in sheet | Loop Over Items        | Section 3 – New game notifications                                                                        |
| Loop Over Items         | SplitInBatches          | Processes new games in batches of 6          | GetImage                 | Wait (secondary output)    | Section 3 – New game notifications                                                                        |
| Wait                    | Wait                    | Pauses between batches to respect rate limits| Loop Over Items          | Send a message1           | Section 3 – New game notifications                                                                        |
| Send a message1         | Discord                 | Sends notifications of new games             | Wait                     | Loop Over Items           | Section 3 – New game notifications                                                                        |
| Sticky Note             | Sticky Note             | Workflow overview and disclaimers            |                         |                           | Contains full workflow description, guide link, video link, and important notes                           |
| Sticky Note2            | Sticky Note             | Notification section explanation              |                         |                           | Explains notification logic and alternatives                                                             |
| Sticky Note3            | Sticky Note             | Google Sheets update explanation              |                         |                           | Explains how rows are appended or updated in the sheet                                                   |
| Sticky Note4            | Sticky Note             | Amazon request parameters explanation         |                         |                           | Explains header fields setup for Amazon Luna API request                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every 5 days at 15:00 hours.

2. **Add a Google Sheets node (“Get row(s) in sheet”)**  
   - Operation: Retrieve rows from your designated Google Sheet.  
   - Authentication: Use a Google Service Account credential.  
   - Configure Document ID and Sheet Name pointing to your sheet with existing game data.

3. **Add a Set node (“Edit Fields”)**  
   - Purpose: Define HTTP headers for Amazon Luna API request.  
   - Add fields with fixed values:  
     - Content-Type: `text/plain;charset=UTF-8`  
     - Origin: `https://luna.amazon.it`  
     - Refer: `https://luna.amazon.it/`  
     - x-amz-locale: `en_US`  
     - x-amz-platform: `web`  
     - x-amz-device-type: `browser`  
     - x-amz-marketplace-id: `ATVPDKIKX0DER` (US marketplace)  
     - Accept-Language: `en-US`  
     - User-Agent: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/141.0.0.0 Safari/537.36`  
     - searchContext: `{ "query": "included_with_prime" }.`

4. **Add an HTTP Request node (“HTTP Request1”)**  
   - Method: POST  
   - URL: `https://proxy-prod.eu-west-1.tempo.digital.a2z.com/getPage`  
   - Body Content-Type: JSON  
   - Body:  
     ```json
     {
       "timeout": 10000,
       "searchContext": {
         "query": "included_with_prime",
         "sort": "TITLE_A_TO_Z",
         "filtersList": []
       },
       "featureScheme": "WEB_V1",
       "pageContext": {
         "pageType": "multistate_browse_results",
         "pageId": "default"
       },
       "clientContext": {
         "browserMetadata": {
           "browserClientRole": "browser",
           "browserType": "Chrome",
           "browserVersion": "141.0.0.0",
           "deviceModel": "unknown",
           "deviceType": "unknown",
           "osName": "Windows",
           "osVersion": "11"
         }
       },
       "inputContext": {
         "gamepadTypes": []
       },
       "dynamicFeatures": []
     }
     ```  
   - Headers: Bind dynamically from the “Edit Fields” node outputs.

5. **Add a Code node (“parsing”)**  
   - Paste the provided JavaScript code that recursively parses “GAME_TILE” widgets to extract game metadata.

6. **Add a Code node (“isNew”)**  
   - Paste the provided JavaScript code that compares new items against existing Google Sheet rows to flag new games.

7. **Add an If node (“If1”)**  
   - Condition: Check if `isNew` equals `true` (boolean).  
   - True branch connects to next steps, false branch ends flow for that item.

8. **Add a Google Sheets node (“Append or update row in sheet”)**  
   - Operation: Append or update rows.  
   - Match rows by “title” column.  
   - Map fields: asin, isNew (set to FALSE here), title, genres, releaseYear, imagePortrait, imageLandscape.  
   - Authentication: Use Google Service Account credential.  
   - Document ID and Sheet Name as configured.

9. **Add an HTTP Request node (“GetImage”)**  
   - Method: GET  
   - URL: Use expression `={{ $('isNew').item.json.imageLandscape }}` to download the landscape image for each new game.

10. **Add a SplitInBatches node (“Loop Over Items”)**  
    - Batch size: 6  
    - Input: From “GetImage” node.

11. **Add a Wait node (“Wait”)**  
    - Default wait time (can be left empty for default delay or set as needed).  
    - Connect secondary output of “Loop Over Items” to “Wait”.

12. **Add a Discord node (“Send a message1”)**  
    - OAuth2 Credentials: Set up and select your Discord OAuth2 credentials.  
    - Guild ID and Channel ID: Set to target server and channel for notifications.  
    - Message Content:  
      ```
      New game added on Amazon Luna!
      Title: {{ $('If1').item.json.title }}
      Generes: {{ $('If1').item.json.genres }}
      Release Year: {{ $('If1').item.json.releaseYear }}
      ```  
    - Connect “Wait” output to “Send a message1”.  
    - Connect “Send a message1” output back to “Loop Over Items” to continue batch processing.

13. **Connect all nodes in the following order:**  
    Schedule Trigger → Get row(s) in sheet → Edit Fields → HTTP Request1 → parsing → isNew → If1 → Append or update row in sheet → GetImage → Loop Over Items → Wait → Send a message1 → Loop Over Items

14. **Configure credentials:**  
    - Google Sheets: Create and configure a service account with access to your Google Sheet.  
    - Discord OAuth2: Configure OAuth2 credentials with appropriate permissions to post messages.

15. **Test and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Full Guide: How to fetch and sync Amazon Luna Included with Prime games using n8n, including setup and troubleshooting tips.                                                                                                                                                                                                              | https://paoloronco.it/amazon-luna-fetch-included-with-prime-games/                                      |
| Video walkthrough demonstrating the workflow in action.                                                                                                                                                                                                                                                                                   | https://youtu.be/PS6qdCbc5fU                                                                            |
| Important: Data used in this workflow is sourced from Amazon Luna and remains Amazon’s property. Use only for personal/testing purposes. Do not redistribute publicly without permission. Endpoint and header format may change, requiring updates to the workflow.                                                                        |                                                                                                         |
| Notification node is configured by default for Discord but can be replaced with Telegram, Slack, email, webhooks, or other services without modifying the rest of the workflow.                                                                                                                                                           | See Sticky Note2                                                                                         |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly available.