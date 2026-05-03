Get chess.com game reviews by email using Google Gemini or other LLMs

https://n8nworkflows.xyz/workflows/get-chess-com-game-reviews-by-email-using-google-gemini-or-other-llms-14165


# Get chess.com game reviews by email using Google Gemini or other LLMs

# 1. Workflow Overview

This workflow automatically fetches the latest Chess.com game for a specified player, enriches the raw game data with player-specific context, sends that game to an AI model for coaching analysis, and emails the resulting HTML report to a configured address.

Typical use cases:
- Daily self-review for Chess.com players
- Automated post-game coaching summaries
- Lightweight chess improvement reports without needing a private Chess.com API key
- Reusable pattern for “public API → enrich data → LLM analysis → email delivery”

## 1.1 Triggering and Input Configuration

The workflow starts either on a daily schedule or through a manual trigger for testing. Both entry points feed into a configuration node where the Chess.com username and destination email address are defined.

## 1.2 Chess.com Data Retrieval

Using the configured username, the workflow builds a Chess.com public API URL for the current month and fetches that month’s game archive.

## 1.3 Game Selection and Context Enrichment

The returned game list is sorted from newest to oldest. The workflow then enriches each game with perspective-aware metadata such as the player’s color, opponent information, result interpretation, and ECO opening code. A batching node ensures that only the most recent game is analyzed.

## 1.4 AI Coaching Analysis

The latest enriched game is passed into an AI Agent node. That node uses a connected chat model, here Google Gemini, and a strict prompt requesting a structured HTML coaching report tailored to club-level players.

## 1.5 Email Delivery

The HTML analysis is sent via Gmail to the configured email recipient, with a chess-specific subject line based on the reviewed player.

---

# 2. Block-by-Block Analysis

## Block 1 — Entry Points and User Configuration

### Overview
This block defines how the workflow starts and where the core runtime parameters come from. It supports both automated execution and manual testing, then normalizes input into a single configuration object containing the Chess.com username and destination email.

### Nodes Involved
- ⏰ Daily Schedule Trigger
- ▶️ Manual Test Run
- Config  - Username - Email

### Node Details

#### 1. ⏰ Daily Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; entry node for automated runs.
- **Configuration choices:** Configured with an interval rule that currently runs once per day.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Config  - Username - Email`.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Misconfigured schedule may run more or less often than expected.
  - Timezone interpretation depends on the n8n instance settings.
- **Sub-workflow reference:** None.

#### 2. ▶️ Manual Test Run
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry node for ad hoc execution in the editor.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Config  - Username - Email`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Only runs when manually executed in the n8n editor or via supported testing mechanisms.
- **Sub-workflow reference:** None.

#### 3. Config  - Username - Email
- **Type and technical role:** `n8n-nodes-base.set`; static configuration node that creates the workflow’s runtime parameters.
- **Configuration choices:** Assigns two string fields:
  - `username`: placeholder value `uer-name-here`
  - `email`: placeholder value `user@example.com`
- **Key expressions or variables used:** Outputs fixed fields later consumed by downstream expressions.
- **Input and output connections:** Receives input from both trigger nodes; outputs to `Build API URL (Current Month)`.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If the username is not updated, the workflow will query an invalid Chess.com profile.
  - If the email is invalid, the Gmail node may fail or reject delivery.
  - There is no validation layer before downstream use.
- **Sub-workflow reference:** None.

---

## Block 2 — Chess.com API Request Construction and Retrieval

### Overview
This block constructs the Chess.com archive URL for the current month and performs the HTTP request to obtain all games played during that period. It relies entirely on the public Chess.com API and requires no API credential.

### Nodes Involved
- Build API URL (Current Month)
- Fetch Games from Chess.com

### Node Details

#### 4. Build API URL (Current Month)
- **Type and technical role:** `n8n-nodes-base.code`; transforms configuration input into an API endpoint descriptor.
- **Configuration choices:** JavaScript code:
  - Reads `username` from the incoming item
  - Determines current year and month from `new Date()`
  - Builds URL in this pattern:  
    `https://api.chess.com/pub/player/{username}/games/{year}/{month}`
  - Returns one item with `username`, `year`, `month`, and `url`
- **Key expressions or variables used:**
  - `$input.item.json.username`
- **Input and output connections:** Input from `Config  - Username - Email`; output to `Fetch Games from Chess.com`.
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Uses the server’s current date; if the player’s latest game was in a previous month and they have not played this month, no games will be found.
  - Username is inserted directly into the URL; malformed values may cause 404 or API errors.
  - Does not URL-encode special characters.
- **Sub-workflow reference:** None.

#### 5. Fetch Games from Chess.com
- **Type and technical role:** `n8n-nodes-base.httpRequest`; performs the GET request against the public Chess.com endpoint.
- **Configuration choices:**
  - URL comes from expression `{{ $json.url }}`
  - Sends JSON headers manually
  - Custom headers include:
    - `User-Agent`
    - `Accept: application/json`
    - `Accept-Language`
  - `lowercaseHeaders` disabled
- **Key expressions or variables used:**
  - `={{ $json.url }}`
- **Input and output connections:** Input from `Build API URL (Current Month)`; output to `Sort Games (Latest First)`.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - Network failures or Chess.com downtime
  - HTTP 404 if the archive endpoint is unavailable for the user/month
  - Rate limiting or anti-bot restrictions, though this is less likely for low-frequency usage
  - Unexpected response shape if Chess.com changes the API
- **Sub-workflow reference:** None.

---

## Block 3 — Sorting, Enrichment, and Selection of the Most Recent Game

### Overview
This block turns the monthly archive into ordered individual game items, adds player-centric metadata, then restricts processing to the single most recent game. It is the core transformation stage between raw API data and AI-ready context.

### Nodes Involved
- Sort Games (Latest First)
- Add Player Context
- Process Last Game Only

### Node Details

#### 6. Sort Games (Latest First)
- **Type and technical role:** `n8n-nodes-base.code`; sorts the monthly archive and explodes the array into individual items.
- **Configuration choices:** JavaScript code:
  - Reads `games` from the HTTP response
  - Reads `username` from the `Config  - Username - Email` node using node reference syntax
  - Throws an error if `games` is missing or empty
  - Sorts by `end_time` descending
  - Adds `_username` to each game object for later perspective calculations
- **Key expressions or variables used:**
  - `$input.item.json.games`
  - `$("Config  - Username - Email").first().json.username`
- **Input and output connections:** Input from `Fetch Games from Chess.com`; output to `Add Player Context`.
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Throws hard error when no games exist this month
  - If `games` is not an array, the sort call will fail
  - Dependency on another node by name means renaming `Config  - Username - Email` would break the code unless updated
- **Sub-workflow reference:** None.

#### 7. Add Player Context
- **Type and technical role:** `n8n-nodes-base.code`; enriches each game with reviewing-player context.
- **Configuration choices:** JavaScript code:
  - Determines whether the configured player was White or Black
  - Computes:
    - `playerColor`
    - `opponentColor`
    - `playerRating`
    - `opponentUsername`
    - `opponentRating`
    - `gameResult`
    - `resultEmoji`
    - `playerResult`
    - `opponentResult`
    - `ecoCode`
  - Maps Chess.com result codes into Win/Loss/Draw/Unknown
  - Extracts ECO code from `game.eco` URL using regex
- **Key expressions or variables used:**
  - `$input.item.json`
  - `game._username`
  - `game.white.username`
  - `game.black.username`
  - `game.eco.match(/\/([A-E]\d{2})/)`
- **Input and output connections:** Input from `Sort Games (Latest First)`; output to `Process Last Game Only`.
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - If `white`, `black`, or `username` fields are missing, the code will error
  - If the reviewing username is not one of the game participants, the node will incorrectly assume Black
  - Result codes outside the mapped lists become `Unknown`
  - ECO remains `Unknown` if no URL is present or the format differs
- **Sub-workflow reference:** None.

#### 8. Process Last Game Only
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; limits processing to the first item, effectively the newest game after sorting.
- **Configuration choices:** Default batch settings; batch size is implicitly 1.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Main input from `Add Player Context`
  - Second output goes to `AI Chess Coach (Analysis)`
  - First output is unused in this workflow
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - The use of Split in Batches for single-item selection is functional but not the simplest pattern; modifications may accidentally create loops if connected differently
  - If more than one game should be processed later, this node becomes a constraint
- **Sub-workflow reference:** None.

---

## Block 4 — AI-Based Chess Coaching Analysis

### Overview
This block turns a structured chess game object into a coaching report in strict Gmail-compatible HTML. The AI Agent node holds the main prompt and output constraints, while the connected LLM node supplies the underlying model.

### Nodes Involved
- AI Chess Coach (Analysis)
- LLM — Google Gemini

### Node Details

#### 9. AI Chess Coach (Analysis)
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; LLM-powered agent node generating the chess coaching analysis.
- **Configuration choices:**
  - Prompt type is `define`
  - Main text prompt includes:
    - Reviewing player and opponent details
    - Ratings, colors, result, time control, opening code
    - Game URL
    - Result codes
    - Final FEN
    - Full PGN
  - Instructs the model to generate a fixed HTML structure:
    1. Game Summary
    2. Critical Moments
    3. Mistake Categories
    4. What Went Well
    5. Five Key Lessons
    6. Training Plan
    7. Closing Motivation
  - System message defines tone, rating range, allowed HTML tags, and formatting restrictions
- **Key expressions or variables used:**
  - `{{ $json.reviewingUsername }}`
  - `{{ $json.playerColor }}`
  - `{{ $json.playerRating }}`
  - `{{ $json.opponentUsername }}`
  - `{{ $json.opponentColor }}`
  - `{{ $json.opponentRating }}`
  - `{{ $json.resultEmoji }}`
  - `{{ $json.gameResult }}`
  - `{{ $json.time_control }}`
  - `{{ $json.time_class }}`
  - `{{ $json.ecoCode }}`
  - `{{ $json.url }}`
  - `{{ $json.playerResult }}`
  - `{{ $json.opponentResult }}`
  - `{{ $json.fen }}`
  - `{{ $json.pgn }}`
- **Input and output connections:**
  - Main input from `Process Last Game Only`
  - AI language model input from `LLM — Google Gemini`
  - Main output to `Email the Report`
- **Version-specific requirements:** Type version `3.1`; requires compatible LangChain/AI features in the n8n environment.
- **Edge cases or potential failure types:**
  - LLM may still produce non-compliant HTML despite instructions
  - Very large PGN or model-side token limits could truncate output
  - Missing PGN/FEN fields weaken or break analysis quality
  - Model may hallucinate strategic details beyond the PGN if the prompt is not strict enough
  - AI provider authentication or quota issues propagate from the model node
- **Sub-workflow reference:** None.

#### 10. LLM — Google Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`; chat model backend used by the agent node.
- **Configuration choices:** Minimal explicit settings; defaults are used, with credentials attached through `googlePalmApi`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Connected to `AI Chess Coach (Analysis)` via `ai_languageModel`.
- **Version-specific requirements:** Type version `1`; requires valid Google Gemini / PaLM-compatible credentials in n8n.
- **Edge cases or potential failure types:**
  - Invalid or expired credentials
  - Model access restrictions by account or region
  - Quota exhaustion
  - Latency or timeout during long generations
- **Sub-workflow reference:** None.

---

## Block 5 — Email Delivery

### Overview
This final block sends the AI-generated HTML report to the configured recipient using Gmail OAuth2. It uses the LLM output directly as the message body.

### Nodes Involved
- Email the Report

### Node Details

#### 11. Email the Report
- **Type and technical role:** `n8n-nodes-base.gmail`; sends an email through Gmail.
- **Configuration choices:**
  - Recipient comes from the config node:
    - `{{ $('Config  - Username - Email').first().json.email }}`
  - Message body comes from:
    - `{{ $json.output }}`
  - Subject:
    - `♟️ Chess Game Review — {{ $('Process Last Game Only').item.json.reviewingUsername }}`
  - Attribution disabled
- **Key expressions or variables used:**
  - `={{ $('Config  - Username - Email').first().json.email }}`
  - `={{ $json.output }}`
  - `=♟️ Chess Game Review — {{ $('Process Last Game Only').item.json.reviewingUsername }}`
- **Input and output connections:** Input from `AI Chess Coach (Analysis)`; no downstream output.
- **Version-specific requirements:** Type version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Invalid Gmail OAuth2 credentials
  - Gmail sending restrictions or account security blocks
  - If `$json.output` is empty or malformed, the email may be blank or poorly rendered
  - The `sendTo` expression contains a trailing newline in the JSON source; usually harmless, but worth cleaning in production
  - If the upstream batch logic changes, `.item` references may behave unexpectedly
- **Sub-workflow reference:** None.

---

## Non-Executable Documentation Nodes

These are sticky notes used for explanation inside the canvas. They do not participate in execution but should be preserved for maintainability.

### 12. 📖 Workflow Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Describes the workflow purpose, process steps, and author contact.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; informational only.
- **Sub-workflow reference:** None.

### 13. Step 1 Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Instructs user to configure username and email in the config node.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### 14. Steps 2-4 Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Explains fetch, sort, and context enrichment logic.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### 15. Step 5 Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Explains AI analysis role and suggests alternate LLMs.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### 16. Step 6 Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation.
- **Configuration choices:** Explains email delivery and optional downstream channels.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

### 17. 📖 Workflow Overview1
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation showing an example output.
- **Configuration choices:** Contains a sample chess coaching report excerpt.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏰ Daily Schedule Trigger | n8n-nodes-base.scheduleTrigger | Automated daily entry point |  | Config  - Username - Email | # ♟️ AI Chess Game Coach<br>An n8n workflow that automatically fetches your latest Chess.com game, analyzes it with AI, and emails you a personalized coaching report.<br>---<br>## How It Works<br>1. Triggers on a daily schedule (or manually on demand)<br>2. Fetches your most recent game from the Chess.com public API<br>3. Enriches the game data with player context, result, and opening code<br>4. Sends the full game to an AI coach for structured analysis<br>5. Emails you an HTML coaching report covering critical moments, mistakes, strengths, and a training plan<br>---<br>## Created by<br>**Ucartz Online**<br>For any queries or clarifications about this workflow, feel free to reach out.<br>📧 [support@ucartz.com](mailto:support@ucartz.com) |
| ▶️ Manual Test Run | n8n-nodes-base.manualTrigger | Manual test entry point |  | Config  - Username - Email | # ♟️ AI Chess Game Coach<br>An n8n workflow that automatically fetches your latest Chess.com game, analyzes it with AI, and emails you a personalized coaching report.<br>---<br>## How It Works<br>1. Triggers on a daily schedule (or manually on demand)<br>2. Fetches your most recent game from the Chess.com public API<br>3. Enriches the game data with player context, result, and opening code<br>4. Sends the full game to an AI coach for structured analysis<br>5. Emails you an HTML coaching report covering critical moments, mistakes, strengths, and a training plan<br>---<br>## Created by<br>**Ucartz Online**<br>For any queries or clarifications about this workflow, feel free to reach out.<br>📧 [support@ucartz.com](mailto:support@ucartz.com) |
| Config  - Username - Email | n8n-nodes-base.set | Stores configurable username and recipient email | ⏰ Daily Schedule Trigger, ▶️ Manual Test Run | Build API URL (Current Month) | ## Step 1 — Configure<br>Open the **⚙️ Config** node and set:<br>• `username` → your Chess.com handle<br>• `email` → where to send reports<br>💡 Username is case-insensitive. |
| Build API URL (Current Month) | n8n-nodes-base.code | Builds Chess.com monthly archive URL | Config  - Username - Email | Fetch Games from Chess.com | ## Steps 2–4 — Fetch & Enrich<br>**Fetch Games** calls the free Chess.com API (no key needed).<br>**Sort** orders games newest → oldest.<br>**Add Context** determines:<br>- Which color you played<br>- Your result (Win / Loss / Draw)<br>- Opponent info & ECO code<br>All data is passed directly into the AI prompt. |
| Fetch Games from Chess.com | n8n-nodes-base.httpRequest | Retrieves current-month game archive from Chess.com | Build API URL (Current Month) | Sort Games (Latest First) | ## Steps 2–4 — Fetch & Enrich<br>**Fetch Games** calls the free Chess.com API (no key needed).<br>**Sort** orders games newest → oldest.<br>**Add Context** determines:<br>- Which color you played<br>- Your result (Win / Loss / Draw)<br>- Opponent info & ECO code<br>All data is passed directly into the AI prompt. |
| Sort Games (Latest First) | n8n-nodes-base.code | Sorts games newest-first and emits individual items | Fetch Games from Chess.com | Add Player Context | ## Steps 2–4 — Fetch & Enrich<br>**Fetch Games** calls the free Chess.com API (no key needed).<br>**Sort** orders games newest → oldest.<br>**Add Context** determines:<br>- Which color you played<br>- Your result (Win / Loss / Draw)<br>- Opponent info & ECO code<br>All data is passed directly into the AI prompt. |
| Add Player Context | n8n-nodes-base.code | Enriches games with player-perspective metadata | Sort Games (Latest First) | Process Last Game Only | ## Steps 2–4 — Fetch & Enrich<br>**Fetch Games** calls the free Chess.com API (no key needed).<br>**Sort** orders games newest → oldest.<br>**Add Context** determines:<br>- Which color you played<br>- Your result (Win / Loss / Draw)<br>- Opponent info & ECO code<br>All data is passed directly into the AI prompt. |
| Process Last Game Only | n8n-nodes-base.splitInBatches | Restricts flow to the latest single game | Add Player Context | AI Chess Coach (Analysis) | ## Steps 2–4 — Fetch & Enrich<br>**Fetch Games** calls the free Chess.com API (no key needed).<br>**Sort** orders games newest → oldest.<br>**Add Context** determines:<br>- Which color you played<br>- Your result (Win / Loss / Draw)<br>- Opponent info & ECO code<br>All data is passed directly into the AI prompt. |
| AI Chess Coach (Analysis) | @n8n/n8n-nodes-langchain.agent | Generates structured HTML coaching review | Process Last Game Only, LLM — Google Gemini | Email the Report | ## Step 5 — AI Analysis<br>The AI coach receives the full PGN, player context, and ratings, then produces a structured HTML report.<br>**Swap the LLM node** to use:<br>- OpenAI GPT-4o<br>- Anthropic Claude<br>- Mistral, Llama, etc.<br>💡 Better models → better analysis. GPT-4o or Claude Sonnet recommended for 1000+ rated players. |
| LLM — Google Gemini | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Supplies the chat model for the AI agent |  | AI Chess Coach (Analysis) | ## Step 5 — AI Analysis<br>The AI coach receives the full PGN, player context, and ratings, then produces a structured HTML report.<br>**Swap the LLM node** to use:<br>- OpenAI GPT-4o<br>- Anthropic Claude<br>- Mistral, Llama, etc.<br>💡 Better models → better analysis. GPT-4o or Claude Sonnet recommended for 1000+ rated players. |
| Email the Report | n8n-nodes-base.gmail | Sends HTML coaching report via Gmail | AI Chess Coach (Analysis) |  | ## Step 6 — Send Report<br>The HTML report is emailed directly.<br>**Want other channels?**<br>Add nodes after the email step:<br>- **Telegram** — send as message<br>- **Slack** — post to channel<br>- **Notion** — save to database<br>- **Google Drive** — save as file<br>- **Discord** — post to server |
| 📖 Workflow Overview | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Step 1 Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Steps 2-4 Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Step 5 Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| Step 6 Note | n8n-nodes-base.stickyNote | Canvas documentation |  |  |  |
| 📖 Workflow Overview1 | n8n-nodes-base.stickyNote | Canvas documentation with sample output |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `♟️ Chess Game Coach — Auto AI Review via Email`.

2. **Add the schedule trigger**
   - Create a `Schedule Trigger` node.
   - Name it: `⏰ Daily Schedule Trigger`.
   - Configure it to run once per day.
   - If desired, change the interval to every 12 hours or another cadence.

3. **Add a manual trigger**
   - Create a `Manual Trigger` node.
   - Name it: `▶️ Manual Test Run`.
   - No extra configuration is required.

4. **Add the configuration node**
   - Create a `Set` node.
   - Name it: `Config  - Username - Email`.
   - Add two string fields:
     - `username` = your Chess.com username
     - `email` = destination email for reports
   - Connect both trigger nodes to this Set node.

5. **Add the API URL builder**
   - Create a `Code` node.
   - Name it: `Build API URL (Current Month)`.
   - Connect `Config  - Username - Email` to it.
   - Paste this logic in equivalent form:
     - Read `username` from input
     - Compute current year and month
     - Build URL as `https://api.chess.com/pub/player/{username}/games/{year}/{month}`
     - Return one item containing `username`, `year`, `month`, and `url`
   - Practical code structure:
     - `const username = $input.item.json.username;`
     - `const now = new Date();`
     - `const year = now.getFullYear();`
     - `const month = String(now.getMonth() + 1).padStart(2, '0');`

6. **Add the HTTP request node**
   - Create an `HTTP Request` node.
   - Name it: `Fetch Games from Chess.com`.
   - Connect `Build API URL (Current Month)` to it.
   - Set:
     - **Method:** GET
     - **URL:** `{{ $json.url }}`
   - Enable custom headers and set:
     - `User-Agent: Mozilla/5.0 (compatible; n8n-chess-coach-bot/1.0)`
     - `Accept: application/json`
     - `Accept-Language: en-US,en;q=0.9`
   - No authentication is needed.

7. **Add the sort node**
   - Create a `Code` node.
   - Name it: `Sort Games (Latest First)`.
   - Connect `Fetch Games from Chess.com` to it.
   - Implement logic to:
     - Read `games` from the HTTP response
     - Throw an error if the array is missing or empty
     - Sort by `end_time` descending
     - Return each game as its own item
     - Add `_username` from the config node
   - Important: if you keep the exact code style from the original workflow, reference the config node by exact name:
     - `$("Config  - Username - Email").first().json.username`
   - If you rename the config node, update the expression accordingly.

8. **Add the player context enrichment node**
   - Create another `Code` node.
   - Name it: `Add Player Context`.
   - Connect `Sort Games (Latest First)` to it.
   - Implement logic to:
     - Determine whether the configured user played White or Black
     - Extract player and opponent ratings
     - Map game result codes to `Win`, `Loss`, `Draw`, or `Unknown`
     - Add a corresponding emoji
     - Extract ECO code from the Chess.com `eco` URL if available
   - Ensure the node outputs all fields needed later:
     - `reviewingUsername`
     - `playerColor`
     - `opponentColor`
     - `playerRating`
     - `opponentUsername`
     - `opponentRating`
     - `gameResult`
     - `resultEmoji`
     - `playerResult`
     - `opponentResult`
     - `ecoCode`

9. **Add the “latest game only” limiter**
   - Create a `Split In Batches` node.
   - Name it: `Process Last Game Only`.
   - Connect `Add Player Context` to it.
   - Leave the default batch size at 1.
   - Use the batch output that emits the current item onward to the AI step.
   - Do not build a loop back unless you want to process multiple games.

10. **Add the AI Agent node**
    - Create an `AI Agent` node from the LangChain-enabled n8n nodes.
    - Name it: `AI Chess Coach (Analysis)`.
    - Connect the main output from `Process Last Game Only` to it.
    - Set prompt mode to define your own prompt.
    - Put the user prompt text so it includes:
      - Reviewing player
      - Opponent
      - Ratings
      - Result and emojis
      - Time control and time class
      - Opening ECO code
      - Game link
      - Result codes
      - Final FEN
      - Full PGN
    - Instruct it to produce **clean HTML** for Gmail with sections:
      1. Game Summary
      2. Critical Moments
      3. Mistake Categories
      4. What Went Well
      5. Five Key Lessons
      6. Training Plan
      7. Closing Motivation
    - Add a system message that:
      - Sets the AI as an encouraging chess coach for players rated 600–1400
      - Requests practical, big-picture improvement advice
      - Forbids Markdown
      - Restricts output to basic HTML tags such as `h2`, `h3`, `p`, `ul`, `li`, `ol`, `strong`, `em`, `a`, `br`, `hr`, `blockquote`

11. **Add the LLM model node**
    - Create a `Google Gemini Chat Model` node.
    - Name it: `LLM — Google Gemini`.
    - Connect it to the AI Agent node through the `ai_languageModel` connector, not the regular main connector.
    - Configure Google Gemini credentials in n8n.
      - The workflow uses `googlePalmApi` credentials.
      - Use a valid API key/account with model access enabled.
    - You may leave default model settings unless you need a specific model variant or temperature.
    - Alternative models can replace this node if preferred:
      - OpenAI Chat Model
      - Anthropic Chat Model
      - Mistral-compatible chat model
      - Other LangChain-supported providers

12. **Add the Gmail node**
    - Create a `Gmail` node.
    - Name it: `Email the Report`.
    - Connect `AI Chess Coach (Analysis)` to it.
    - Configure Gmail OAuth2 credentials.
    - Set:
      - **To:** `{{ $('Config  - Username - Email').first().json.email }}`
      - **Subject:** `♟️ Chess Game Review — {{ $('Process Last Game Only').item.json.reviewingUsername }}`
      - **Message:** `{{ $json.output }}`
    - Disable attribution/appended branding if you want a cleaner email.

13. **Credential setup**
    - **Google Gemini credentials**
      - Add valid Gemini/Google AI credentials in n8n.
      - Attach them to `LLM — Google Gemini`.
    - **Gmail OAuth2 credentials**
      - Create Gmail OAuth2 credentials in n8n.
      - Ensure the connected Google account has permission to send email.
      - Attach them to `Email the Report`.

14. **Reconnect the full execution chain**
    - `⏰ Daily Schedule Trigger` → `Config  - Username - Email`
    - `▶️ Manual Test Run` → `Config  - Username - Email`
    - `Config  - Username - Email` → `Build API URL (Current Month)`
    - `Build API URL (Current Month)` → `Fetch Games from Chess.com`
    - `Fetch Games from Chess.com` → `Sort Games (Latest First)`
    - `Sort Games (Latest First)` → `Add Player Context`
    - `Add Player Context` → `Process Last Game Only`
    - `Process Last Game Only` → `AI Chess Coach (Analysis)`
    - `LLM — Google Gemini` → `AI Chess Coach (Analysis)` via AI model connector
    - `AI Chess Coach (Analysis)` → `Email the Report`

15. **Optional: recreate the canvas notes**
    - Add sticky notes describing:
      - Overall workflow purpose
      - Configuration instructions
      - Fetch and enrichment steps
      - AI model swapping guidance
      - Email and multi-channel delivery extensions
      - Sample output preview

16. **Test manually**
    - Run the workflow using `▶️ Manual Test Run`.
    - Confirm:
      - The Chess.com API returns a `games` array
      - The sorting node emits items
      - The context node assigns correct color and result
      - The AI output is HTML, not Markdown
      - Gmail renders the report correctly

17. **Activate the workflow**
    - Once validated, activate the workflow so the schedule trigger runs automatically.

## Rebuild Constraints and Expectations
- The workflow expects the Chess.com monthly archive endpoint to include at least one game in the current month.
- The AI node expects these game fields to exist or usually exist:
  - `white`, `black`, `pgn`, `fen`, `url`, `time_control`, `time_class`, `eco`
- No sub-workflow is used in this design.
- If you want to analyze multiple recent games:
  - Remove `Process Last Game Only`, or
  - Replace it with a looping design and aggregate/email multiple reports

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Created by Ucartz Online | Workflow branding |
| Support email: [support@ucartz.com](mailto:support@ucartz.com) | Contact for workflow queries or clarifications |
| Chess.com public API is used and does not require an API key for this archive endpoint | General integration note |
| Suggested LLM alternatives: OpenAI GPT-4o, Anthropic Claude 3.5 Sonnet, Google Gemini 1.5 Pro, Mistral, Llama | Model substitution guidance |
| Suggested downstream channels after email: Telegram, Slack, Notion, Google Drive, Discord | Extension ideas |
| Important operational limitation: only games from the current calendar month are fetched | Retrieval scope note |
| Important processing limitation: only the most recent game is analyzed because of the Split In Batches node | Processing scope note |
| Sample result note is included on the canvas to illustrate expected HTML coaching output | Documentation note |