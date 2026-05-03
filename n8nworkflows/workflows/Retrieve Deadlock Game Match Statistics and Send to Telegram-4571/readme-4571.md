Retrieve Deadlock Game Match Statistics and Send to Telegram

https://n8nworkflows.xyz/workflows/retrieve-deadlock-game-match-statistics-and-send-to-telegram-4571


# Retrieve Deadlock Game Match Statistics and Send to Telegram

### 1. Workflow Overview

This workflow, titled **"Deadlock Match Stats Bot"**, automates the retrieval of recent match statistics from the Deadlock game tracker website for a specific player and sends a formatted summary of the match participants to a Telegram chat. It is designed to respond to Telegram messages (triggered by any incoming message), fetch the latest match data for a predefined player, extract detailed player information from the match, format this information into a human-readable message, and send it back via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input Reception:** Listens for incoming Telegram messages to trigger the workflow.
- **1.2 Profile HTML Fetching:** Retrieves the player’s profile page HTML from DeadlockTracker to find recent match IDs.
- **1.3 Match ID Extraction:** Parses the profile HTML to extract the latest match ID.
- **1.4 Match HTML Fetching:** Requests the match page HTML using the extracted match ID.
- **1.5 Player Information Parsing:** Extracts detailed player information (nicknames, heroes, ranks) from the match HTML.
- **1.6 Message Formatting and Sending:** Formats the player data into a message and sends it back to the Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

- **Overview:**  
  This block triggers the workflow on receipt of any message sent to the configured Telegram bot, obtaining the chat ID for the response.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Trigger node for Telegram messages.  
    - *Configuration:* Uses Telegram API credentials to connect to the bot account. No filters applied; triggers on any message.  
    - *Key Expressions:* None (trigger-only node).  
    - *Input/Output:* No input; outputs the message JSON including chat ID.  
    - *Version:* v1.  
    - *Edge Cases:* Network or auth errors with Telegram API; bot must have correct permissions to receive messages.  
    - *Sub-workflow:* None.

#### 2.2 Profile HTML Fetching

- **Overview:**  
  Fetches the HTML content of the DeadlockTracker player profile page to find the most recent match IDs.

- **Nodes Involved:**  
  - Fetch Profile HTML

- **Node Details:**

  - **Fetch Profile HTML**  
    - *Type:* HTTP Request node.  
    - *Configuration:* GET request to fixed URL `https://deadlocktracker.gg/player/97170387`. No special headers or auth. Response expected as raw string (HTML).  
    - *Key Expressions:* URL hardcoded; response captured as string in body.  
    - *Input/Output:* Input from Telegram Trigger; outputs response body.  
    - *Version:* v1.  
    - *Edge Cases:* HTTP errors (404, 500), site downtime, changes in page structure, rate limiting.  
    - *Sub-workflow:* None.

#### 2.3 Match ID Extraction

- **Overview:**  
  Parses the player profile HTML to extract all match IDs by regex matching, selecting the last (most recent) match ID.

- **Nodes Involved:**  
  - Extract Match ID

- **Node Details:**

  - **Extract Match ID**  
    - *Type:* Function node (JavaScript).  
    - *Configuration:* Uses regex `/\(#(\d{8})\)/g` to find all occurrences of match IDs formatted as (#12345678). Extracts all, then picks the last one. Throws error if none found.  
    - *Key Expressions:* Accesses `$json["body"]` or raw `$json` string; returns an object with `matchId`.  
    - *Input/Output:* Input from Fetch Profile HTML; outputs `{ matchId: string }`.  
    - *Version:* v1.  
    - *Edge Cases:* No matches found (throws error), malformed HTML, page format changes, unexpected regex failures.  
    - *Sub-workflow:* None.

#### 2.4 Match HTML Fetching

- **Overview:**  
  Fetches the HTML of the specific match page using the extracted match ID.

- **Nodes Involved:**  
  - Fetch Match HTML

- **Node Details:**

  - **Fetch Match HTML**  
    - *Type:* HTTP Request node.  
    - *Configuration:* GET request to URL constructed as `https://deadlocktracker.gg/match/{matchId}` dynamically using expression `={{"https://deadlocktracker.gg/match/" + $json["matchId"]}}`. Response as raw string (HTML).  
    - *Key Expressions:* Dynamic URL construction based on previous node output.  
    - *Input/Output:* Input from Extract Match ID; outputs response body.  
    - *Version:* v1.  
    - *Edge Cases:* HTTP errors, invalid or expired match ID, site downtime, structure changes.  
    - *Sub-workflow:* None.

#### 2.5 Player Information Parsing

- **Overview:**  
  Parses the match HTML content to extract player details: nickname, hero played, and rank.

- **Nodes Involved:**  
  - Parse Players

- **Node Details:**

  - **Parse Players**  
    - *Type:* Function node using Node.js and Cheerio (jQuery-like HTML parser).  
    - *Configuration:* Loads HTML, selects all `.player-wrapper` elements, extracts nickname from `.player-name`, hero from `.character-icon` alt attribute (fallback "Неизвестно"), and rank from `.rank-icon` alt attribute (fallback "Без ранга"). Returns an array of player objects.  
    - *Key Expressions:* Uses `$json["body"]` or raw `$json` for HTML input; returns array of objects `{nick, hero, rank}`.  
    - *Input/Output:* Input from Fetch Match HTML; outputs array of player data.  
    - *Version:* v1.  
    - *Edge Cases:* Changes in DOM structure, missing elements, malformed HTML, library loading issues.  
    - *Sub-workflow:* None.

#### 2.6 Message Formatting and Sending

- **Overview:**  
  Converts the array of player info into a multi-line Telegram message and sends it to the original chat.

- **Nodes Involved:**  
  - Format Message  
  - Send Telegram Message

- **Node Details:**

  - **Format Message**  
    - *Type:* Function node.  
    - *Configuration:* Iterates over each player item, formats lines with emoji and labels in Russian: player nick, hero, rank. Prepends a header "📊 Игроки матча:" and joins lines with newlines. Outputs single JSON with `text` property.  
    - *Key Expressions:* Uses `items` array from input; returns formatted text.  
    - *Input/Output:* Input from Parse Players; output to Send Telegram Message.  
    - *Version:* v1.  
    - *Edge Cases:* Empty player list, encoding issues, malformed player data.  
    - *Sub-workflow:* None.

  - **Send Telegram Message**  
    - *Type:* Telegram node.  
    - *Configuration:* Sends message text from Format Message node to the chat ID extracted from original Telegram message (`{{$json["message"]["chat"]["id"]}}`). Uses Telegram API credentials.  
    - *Key Expressions:* Message text and chat ID expressions as above.  
    - *Input/Output:* Input from Format Message; no output.  
    - *Version:* v1.  
    - *Edge Cases:* Telegram API errors, invalid chat ID, message size limits, network issues.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name            | Node Type                 | Functional Role                 | Input Node(s)        | Output Node(s)           | Sticky Note                                              |
|----------------------|---------------------------|--------------------------------|----------------------|--------------------------|----------------------------------------------------------|
| Telegram Trigger      | telegramTrigger           | Triggers workflow on Telegram message | None                 | Fetch Profile HTML        |                                                          |
| Fetch Profile HTML    | httpRequest               | Fetches player profile HTML    | Telegram Trigger     | Extract Match ID          |                                                          |
| Extract Match ID      | function                  | Extracts latest match ID       | Fetch Profile HTML   | Fetch Match HTML          |                                                          |
| Fetch Match HTML      | httpRequest               | Fetches match HTML             | Extract Match ID     | Parse Players             |                                                          |
| Parse Players        | function                  | Parses player info from match HTML | Fetch Match HTML     | Format Message            |                                                          |
| Format Message        | function                  | Formats player info into message | Parse Players        | Send Telegram Message     |                                                          |
| Send Telegram Message | telegram                  | Sends formatted message to Telegram chat | Format Message       | None                     |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Credentials: Set up Telegram API credentials for your bot.  
   - No additional parameters needed. This node listens for any incoming Telegram message.

2. **Create Fetch Profile HTML node:**  
   - Type: HTTP Request  
   - Connect input from Telegram Trigger.  
   - Method: GET  
   - URL: `https://deadlocktracker.gg/player/97170387` (replace with desired player ID if needed)  
   - Response Format: String (to get raw HTML)  
   - No authentication needed.

3. **Create Extract Match ID node:**  
   - Type: Function  
   - Connect input from Fetch Profile HTML.  
   - JavaScript code:  
     ```js
     const html = $json["body"] || $json;
     const matchIdMatch = html.match(/\(#(\d{8})\)/g);
     if (!matchIdMatch) throw new Error("Матчи не найдены");
     const ids = matchIdMatch.map(m => m.match(/\d+/)[0]);
     const lastMatchId = ids[ids.length - 1];
     return [{ matchId: lastMatchId }];
     ```
   - This extracts all match IDs and returns the latest one.

4. **Create Fetch Match HTML node:**  
   - Type: HTTP Request  
   - Connect input from Extract Match ID.  
   - Method: GET  
   - URL: Expression: `={{"https://deadlocktracker.gg/match/" + $json["matchId"]}}`  
   - Response Format: String (raw HTML)  
   - No authentication needed.

5. **Create Parse Players node:**  
   - Type: Function  
   - Connect input from Fetch Match HTML.  
   - Use Node.js with Cheerio (make sure n8n supports this).  
   - JavaScript code:  
     ```js
     const cheerio = require('cheerio');
     const $ = cheerio.load($json["body"] || $json);
     const players = [];

     $('.player-wrapper').each((i, el) => {
       const nick = $(el).find('.player-name').text().trim();
       const hero = $(el).find('.character-icon').attr('alt') || 'Неизвестно';
       const rank = $(el).find('.rank-icon').attr('alt') || 'Без ранга';
       players.push({ nick, hero, rank });
     });

     return players;
     ```
   - Outputs an array of player info objects.

6. **Create Format Message node:**  
   - Type: Function  
   - Connect input from Parse Players.  
   - JavaScript code:  
     ```js
     const lines = items.map(p =>
       `🎮 ${p.nick}\n🧙 Герой: ${p.hero}\n🏆 Ранг: ${p.rank}\n`
     );
     return [{ text: `📊 Игроки матча:\n\n` + lines.join('\n') }];
     ```
   - Output is a single JSON with the full message text.

7. **Create Send Telegram Message node:**  
   - Type: Telegram  
   - Connect input from Format Message.  
   - Credentials: Use the same Telegram API credentials as the trigger.  
   - Parameters:  
     - Resource: `message`  
     - Operation: `send`  
     - Chat ID: Expression `={{$json["message"]["chat"]["id"]}}` (from original Telegram message)  
     - Text: Expression `={{$node["Format Message"].json["text"]}}`

8. **Connect all nodes accordingly:**  
   - Telegram Trigger → Fetch Profile HTML → Extract Match ID → Fetch Match HTML → Parse Players → Format Message → Send Telegram Message.

9. **Activate the workflow** after testing with your Telegram bot.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                             |
|------------------------------------------------------------------------------------------------------|---------------------------------------------|
| The workflow uses Russian text and emojis in its messages to provide a friendly and localized output. | Message formatting in "Format Message" node.|
| Player ID (`97170387`) and URLs are hardcoded; adjust to target different players if needed.          | URL in "Fetch Profile HTML" node.            |
| Cheerio library is used in the function node for HTML parsing; n8n must support `require('cheerio')`. | Node.js environment support in n8n.         |
| Telegram bot must have correct permissions to receive messages and send replies.                      | Telegram Bot API documentation.              |
| Regex pattern expects match IDs as 8-digit numbers enclosed in parentheses and preceded by `#`.       | See `Extract Match ID` node function code.   |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.