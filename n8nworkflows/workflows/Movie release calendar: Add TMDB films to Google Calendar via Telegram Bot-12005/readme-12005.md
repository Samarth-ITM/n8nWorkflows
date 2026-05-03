Movie release calendar: Add TMDB films to Google Calendar via Telegram Bot

https://n8nworkflows.xyz/workflows/movie-release-calendar--add-tmdb-films-to-google-calendar-via-telegram-bot-12005


# Movie release calendar: Add TMDB films to Google Calendar via Telegram Bot

## 1. Workflow Overview

**Purpose:**  
This workflow fetches upcoming movie releases from **TMDB** every day at noon, posts each *new* movie to a **Telegram chat** with an inline button, and—when the button is pressed—creates a corresponding **Google Calendar** event on the movie’s release date.

**Target use cases:**
- Personal or team “movie release calendar” automation
- Avoiding duplicate notifications/events by tracking movies in an n8n **Data Table**
- Quick opt-in event creation via Telegram inline buttons (callback queries)

### 1.1 Scheduled Fetch & Configuration
Runs daily, injects required secrets (TMDB token + Telegram chat ID), calls TMDB Discover endpoint for upcoming releases.

### 1.2 Movie Fan-out + Deduplication + Telegram Message
Splits TMDB results into individual items, checks whether each movie is already stored, stores new ones, and sends a Telegram message with an “Add to calendar” button.

### 1.3 Telegram Callback → Lookup → Google Calendar Event
Listens for inline button callbacks, reads the movie ID from callback data, retrieves the stored movie row from the Data Table, and creates a Google Calendar event.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Fetch & Configuration

**Overview:**  
Triggers daily at noon, sets static configuration (TMDB bearer token and Telegram chat ID), then requests upcoming movies from TMDB sorted by release date.

**Nodes Involved:**
- Every Noon
- Config TMDB token and Telegram chat ID
- Get movies

#### Node: **Every Noon**
- **Type / Role:** Schedule Trigger — starts the workflow on a time schedule.
- **Configuration (interpreted):** Runs every day at **12:00** (server/project timezone).
- **Connections:**
  - **Output →** Config TMDB token and Telegram chat ID
- **Potential failures / edge cases:**
  - Timezone mismatch (noon may not be local noon if n8n timezone differs).
  - If n8n is down at trigger time, execution may be missed (depends on n8n setup).

#### Node: **Config TMDB token and Telegram chat ID**
- **Type / Role:** Set node — injects required configuration into the JSON payload.
- **Configuration (interpreted):**
  - Outputs raw JSON:
    - `tmdb_access_token`: `"token"` (placeholder; must be replaced)
    - `telegram_chat_id`: `"id"` (placeholder; must be replaced)
- **Key variables/expressions:** None (static JSON).
- **Connections:**
  - **Input ←** Every Noon
  - **Output →** Get movies
- **Potential failures / edge cases:**
  - Leaving placeholders unchanged causes TMDB auth failure and Telegram send failure.
  - Chat ID format: Telegram chat IDs can be numeric (often negative for groups). Storing as a string usually works, but inconsistent formatting can break delivery.

#### Node: **Get movies**
- **Type / Role:** HTTP Request — calls TMDB “discover movie” endpoint.
- **Configuration (interpreted):**
  - **Method:** (implicit default) GET
  - **URL:** `https://api.themoviedb.org/3/discover/movie`
  - **Headers:**
    - `Authorization: Bearer {{$json.tmdb_access_token}}`
  - **Query parameters:**
    - `sort_by = primary_release_date.asc`
    - `primary_release_date.gte = {{$today.toFormat('yyyy-MM-dd')}}`
      - Fetches movies with release date **>= today**
- **Key expressions/variables:**
  - `$today.toFormat('yyyy-MM-dd')`
  - `Bearer {{ $json.tmdb_access_token }}`
- **Connections:**
  - **Input ←** Config TMDB token and Telegram chat ID
  - **Output →** Separate movies
- **Potential failures / edge cases:**
  - **401/403** if TMDB token is invalid/expired or not a v4 token.
  - Pagination: TMDB returns paged results; this workflow requests only the default first page unless further parameters are added.
  - Region/language not specified: results may not match desired market without adding `region`, `with_release_type`, etc.
  - Some movies may have missing/empty `release_date`.

---

### Block 2 — Movie Fan-out + Deduplication + Telegram Message

**Overview:**  
Takes the TMDB results array, processes each movie individually, checks a Data Table to avoid duplicates, stores new movies, and posts them to Telegram with an inline callback button containing the movie ID.

**Nodes Involved:**
- Separate movies
- If new
- Add movie data to table
- Ask to add to calendar

#### Node: **Separate movies**
- **Type / Role:** Split Out — converts an array field into one item per array element.
- **Configuration (interpreted):**
  - Splits the `results` array from TMDB response into multiple items.
- **Connections:**
  - **Input ←** Get movies
  - **Output →** If new
- **Potential failures / edge cases:**
  - If TMDB response doesn’t include `results` (API error payload), this node can produce 0 items or fail depending on runtime behavior.
  - If `results` is empty, nothing is sent downstream (expected).

#### Node: **If new**
- **Type / Role:** Data Table — checks existence of a row (dedup gate).
- **Configuration (interpreted):**
  - **Operation:** `rowNotExists`
  - **Data Table:** `movies` (internal id `o7x3ZT6nOpxZqIBJ`)
  - **Filter condition:** column/key `imdb_id` equals `{{$json.id}}`
    - Note: despite the column name `imdb_id`, the value used is **TMDB movie id** (`$json.id`).
- **Connections:**
  - **Input ←** Separate movies
  - **If row does not exist → Output →** Add movie data to table
- **Potential failures / edge cases:**
  - Data Table not found / permission issues.
  - Type mismatch: table schema defines `imdb_id` as number; TMDB `id` is numeric—OK, but callback data arrives as string later (see Block 3).
  - Naming confusion: field called `imdb_id` actually stores TMDB IDs; can mislead maintenance.

#### Node: **Add movie data to table**
- **Type / Role:** Data Table — inserts a new row for each new movie.
- **Configuration (interpreted):**
  - **Operation:** (implied create/insert) with explicit column mapping:
    - `movie = {{$json.title}}`
    - `imdb_id = {{$json.id}}` (TMDB id stored here)
    - `release = {{$json.release_date}}`
  - **Schema present in node:**
    - `imdb_id` (number)
    - `movie` (string)
    - `release` (dateTime)
  - **Matching columns:** `["movie"]` (declared, but for inserts this mainly impacts how n8n tries to match/upsert depending on operation semantics; here it’s still worth verifying behavior in your n8n version).
- **Connections:**
  - **Input ←** If new
  - **Output →** Ask to add to calendar
- **Potential failures / edge cases:**
  - If `release_date` is missing or not parseable as dateTime, later calendar creation may fail.
  - Duplicate handling depends on Data Table behavior; if two different movies share same title and “matching columns” influences dedupe/upsert, you could get unexpected collisions.

#### Node: **Ask to add to calendar**
- **Type / Role:** Telegram node — sends a message with an inline keyboard button.
- **Configuration (interpreted):**
  - **Chat ID:** `{{$('Config TMDB token and Telegram chat ID').item.json.telegram_chat_id}}`
  - **Text:**
    - Title and release date from the *Split Out* item:
      - `{{$('Separate movies').item.json.title}}`
      - `{{$('Separate movies').item.json.release_date}}`
    - Includes overview: `{{$('Separate movies').item.json.overview}}`
  - **Reply Markup:** Inline keyboard with one button:
    - Button text: “Add to calendar”
    - `callback_data = {{$('Separate movies').item.json.id}}` (TMDB movie id)
  - **Credentials:** Telegram API credential required.
- **Connections:**
  - **Input ←** Add movie data to table
  - **Output:** None (terminal for this branch)
- **Potential failures / edge cases:**
  - Telegram auth errors (bad bot token).
  - Chat ID invalid, bot not started by user, or bot lacks permission in group.
  - Telegram `callback_data` limit (64 bytes). TMDB numeric ID is safe.
  - Uses cross-node item references (`$('Separate movies').item...`): if execution/item pairing changes, you can accidentally reference wrong item. It works here because the pipeline is linear per item, but it’s something to watch when adding merges/parallelism.

---

### Block 3 — Telegram Callback → Lookup → Google Calendar Event

**Overview:**  
Waits for Telegram callback queries, extracts the movie id from the callback data, retrieves the stored movie record from the Data Table, and creates a 2-hour Google Calendar event starting at the stored release date.

**Nodes Involved:**
- Add to calendar button pressed
- Get movie from callback data
- Create an event

#### Node: **Add to calendar button pressed**
- **Type / Role:** Telegram Trigger — webhook-based trigger for Telegram updates.
- **Configuration (interpreted):**
  - Listens for `callback_query` updates only (button presses).
  - Requires Telegram Trigger webhook to be registered/active.
- **Connections:**
  - **Output →** Get movie from callback data
- **Potential failures / edge cases:**
  - Webhook not set (workflow inactive, wrong webhook URL, reverse proxy issues).
  - Telegram bot privacy/group settings can prevent updates in some chats.
  - Callback data may be missing or manipulated; should be validated.

#### Node: **Get movie from callback data**
- **Type / Role:** Data Table — fetches a row using the callback data as identifier.
- **Configuration (interpreted):**
  - **Operation:** `get`
  - **Data Table:** `movies`
  - **Filter:** `imdb_id` equals `{{$json.callback_query.data}}`
    - This compares a numeric column to a string value (Telegram sends strings). n8n often coerces automatically, but not guaranteed across versions/config.
- **Connections:**
  - **Input ←** Add to calendar button pressed
  - **Output →** Create an event
- **Potential failures / edge cases:**
  - No matching row found (e.g., table cleared, movie not inserted, or type mismatch). Downstream calendar node may receive empty input.
  - If multiple rows match (shouldn’t if IDs are unique), you may create multiple events.

#### Node: **Create an event**
- **Type / Role:** Google Calendar — creates a calendar event.
- **Configuration (interpreted):**
  - **Calendar:** `user@example.com` (selected calendar)
  - **Start:** `{{$json.release.toDateTime()}}`
  - **End:** `{{$json.release.toDateTime().plus(2, 'hour')}}` (2-hour duration)
  - **Summary:** `Movie "{{$json.movie}}"`
  - **Description:** `Movie "{{$json.movie}}"`
  - **Credentials:** Google Calendar OAuth2 credential required.
- **Connections:**
  - **Input ←** Get movie from callback data
  - **Output:** None
- **Potential failures / edge cases:**
  - OAuth token expired/invalid; missing consent scopes.
  - `release` not parseable to DateTime (format mismatch) → expression error.
  - Timezone behavior: `toDateTime()` may interpret date/time in server timezone; TMDB `release_date` is date-only (no time). This often becomes midnight; confirm expected time.
  - Duplicate event creation: pressing the button multiple times will create multiple events unless you add dedupe logic (e.g., store “added_to_calendar” flag).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Noon | Schedule Trigger | Daily trigger at 12:00 | — | Config TMDB token and Telegram chat ID | **Add Upcoming Movies to Your Google Calendar** (workflow description/requirements) |
| Config TMDB token and Telegram chat ID | Set | Stores TMDB token + Telegram chat ID for downstream use | Every Noon | Get movies | **1. Add your TMDB token and Telegram ID to the config.** |
| Get movies | HTTP Request | Fetch upcoming movies from TMDB Discover | Config TMDB token and Telegram chat ID | Separate movies | **2. Upcoming movies are requested from TMDB, saved, and each movie is sent separately to Telegram with callback button.** |
| Separate movies | Split Out | Split TMDB `results` array into individual movie items | Get movies | If new | **2. Upcoming movies are requested from TMDB, saved, and each movie is sent separately to Telegram with callback button.** |
| If new | Data Table | Check if movie ID already exists (dedupe gate) | Separate movies | Add movie data to table | **2. Upcoming movies are requested from TMDB, saved, and each movie is sent separately to Telegram with callback button.** |
| Add movie data to table | Data Table | Insert new movie into table (id/title/release) | If new | Ask to add to calendar | **2. Upcoming movies are requested from TMDB, saved, and each movie is sent separately to Telegram with callback button.** |
| Ask to add to calendar | Telegram | Send movie info + inline “Add to calendar” button | Add movie data to table | — | **2. Upcoming movies are requested from TMDB, saved, and each movie is sent separately to Telegram with callback button.** |
| Add to calendar button pressed | Telegram Trigger | Receive callback_query events (button presses) | — | Get movie from callback data | **3. Bot receives callbacks from buttons to add selected movie to calendar** |
| Get movie from callback data | Data Table | Retrieve movie row using callback data (movie id) | Add to calendar button pressed | Create an event | **3. Bot receives callbacks from buttons to add selected movie to calendar** |
| Create an event | Google Calendar | Create calendar event on release date | Get movie from callback data | — | **3. Bot receives callbacks from buttons to add selected movie to calendar** |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Data Table**
   1. In n8n, create a Data Table named **`movies`**.
   2. Add columns:
      - `imdb_id` (Number) *(stores TMDB movie id in this workflow)*
      - `movie` (String)
      - `release` (DateTime)

2) **Create node: “Every Noon”**
   1. Add **Schedule Trigger** node.
   2. Set it to run **daily at 12:00** (triggerAtHour = 12).

3) **Create node: “Config TMDB token and Telegram chat ID”**
   1. Add **Set** node.
   2. Mode: **Raw JSON**.
   3. Paste and edit:
      - `tmdb_access_token`: your TMDB v4 API read access token (bearer token)
      - `telegram_chat_id`: your Telegram user/chat ID (where messages should be sent)

4) **Create node: “Get movies”**
   1. Add **HTTP Request** node.
   2. Method: **GET**
   3. URL: `https://api.themoviedb.org/3/discover/movie`
   4. Add query parameters:
      - `sort_by`: `primary_release_date.asc`
      - `primary_release_date.gte`: expression `{{$today.toFormat('yyyy-MM-dd')}}`
   5. Add header:
      - `Authorization`: expression `Bearer {{$json.tmdb_access_token}}`

5) **Create node: “Separate movies”**
   1. Add **Split Out** node.
   2. Field to split out: `results`

6) **Create node: “If new” (dedupe check)**
   1. Add **Data Table** node.
   2. Select table: **movies**
   3. Operation: **Row Not Exists**
   4. Filter condition:
      - Key/column: `imdb_id`
      - Value: expression `{{$json.id}}`

7) **Create node: “Add movie data to table”**
   1. Add **Data Table** node.
   2. Select table: **movies**
   3. Configure to insert/create a row with mappings:
      - `movie`: `{{$json.title}}`
      - `imdb_id`: `{{$json.id}}`
      - `release`: `{{$json.release_date}}`

8) **Create node: “Ask to add to calendar”**
   1. Add **Telegram** node (Send Message).
   2. Configure **Telegram API credentials** (Bot token).
   3. Chat ID: expression `{{$('Config TMDB token and Telegram chat ID').item.json.telegram_chat_id}}`
   4. Message text (example equivalent):
      - `"{{$('Separate movies').item.json.title}}" is released at {{$('Separate movies').item.json.release_date}}\n\n{{$('Separate movies').item.json.overview}}`
   5. Reply Markup: **Inline Keyboard**
      - One button:
        - Text: `Add to calendar`
        - Callback data: expression `{{$('Separate movies').item.json.id}}`

9) **Create node: “Add to calendar button pressed”**
   1. Add **Telegram Trigger** node.
   2. Configure Telegram credentials (same bot).
   3. Updates to listen for: **callback_query**
   4. Ensure your n8n instance is reachable publicly (or via Telegram-compatible webhook routing), then activate workflow to register webhook.

10) **Create node: “Get movie from callback data”**
   1. Add **Data Table** node.
   2. Table: **movies**
   3. Operation: **Get**
   4. Filter:
      - `imdb_id` equals expression `{{$json.callback_query.data}}`

11) **Create node: “Create an event”**
   1. Add **Google Calendar** node (Create Event).
   2. Configure **Google Calendar OAuth2** credentials.
   3. Select calendar (e.g., your primary calendar).
   4. Start: expression `{{$json.release.toDateTime()}}`
   5. End: expression `{{$json.release.toDateTime().plus(2, 'hour')}}`
   6. Summary: `Movie "{{$json.movie}}"`
   7. Description: `Movie "{{$json.movie}}"`

12) **Connect nodes in two branches**
   - Branch A (daily fetch):
     1. Every Noon → Config TMDB token and Telegram chat ID → Get movies → Separate movies → If new → Add movie data to table → Ask to add to calendar
   - Branch B (button callback):
     1. Add to calendar button pressed → Get movie from callback data → Create an event

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Add Upcoming Movies to Your Google Calendar** — Fetch upcoming releases from TMDB daily, dedupe via n8n Data Table, send each movie to Telegram with inline button, on press create Google Calendar event. **Requirements:** TMDB API access token, Telegram bot token and user ID, Google Calendar OAuth credentials, one n8n data table for movie metadata. | Sticky note (overall workflow description) |
| **1. Add your TMDB token and Telegram ID to the config.** | Sticky note (configuration step) |
| **2. Upcoming movies are requested from TMDB, saved, and each movie is sent separately to Telegram with callback button.** | Sticky note (TMDB → table → Telegram block) |
| **3. Bot receives callbacks from buttons to add selected movie to calendar** | Sticky note (Telegram callback → Calendar block) |