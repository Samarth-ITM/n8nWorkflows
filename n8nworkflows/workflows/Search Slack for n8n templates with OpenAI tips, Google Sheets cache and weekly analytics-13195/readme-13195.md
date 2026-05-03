Search Slack for n8n templates with OpenAI tips, Google Sheets cache and weekly analytics

https://n8nworkflows.xyz/workflows/search-slack-for-n8n-templates-with-openai-tips--google-sheets-cache-and-weekly-analytics-13195


# Search Slack for n8n templates with OpenAI tips, Google Sheets cache and weekly analytics

## 1. Workflow Overview

**Purpose:**  
A Slack @mention bot that searches the official **n8n template library** for relevant workflows, enriches the response with **OpenAI-generated improvement tips**, uses **Google Sheets as a cache** to reduce repeated API calls, and logs usage to an **analytics sheet**. It also posts a **weekly analytics summary** to Slack and includes an **error-handling path**.

**Target use cases:**
- Teams browsing n8n workflow templates directly from Slack
- Multilingual queries (notably Spanish + Japanese term expansion to English keywords)
- Reducing API calls via caching
- Tracking search usage and effectiveness over time (results count, cache hits, intent)

### 1.1 Trigger & Message Intake (Slack)
Receives Slack app mentions and extracts the raw message payload.

### 1.2 Normalize, Keyword Extraction & Intent Detection
Cleans the mention text, detects intent (**search/help/categories**), extracts keywords from a large known-services list, and expands Japanese terms into English keywords.

### 1.3 Intent Routing
Routes to one of three paths:
- Search flow (with cache → API → AI tips)
- Help response
- Categories response

### 1.4 Cache Layer (Google Sheets)
Checks if a matching response is already cached and uses it when available.

### 1.5 Template Search & AI Enrichment
Calls n8n Templates API, then calls OpenAI:
- If templates exist: generate tips & suggestions based on found templates and user query
- If none: generate alternative search terms and build-from-scratch guidance

### 1.6 Reply to Slack + Log to Google Sheets
Formats a Slack message, posts in Slack, logs analytics, and saves cache entries.

### 1.7 Error Handling
Catches workflow errors and posts a friendly Slack message.

### 1.8 Weekly Analytics Report
Weekly cron reads analytics rows, aggregates stats, and posts a summary to Slack.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Message Intake (Slack)
**Overview:** Listens for Slack `app_mention` events in a specified channel and starts the workflow.  
**Nodes involved:**  
- **Slack Trigger - Bot Mention**

#### Node: Slack Trigger - Bot Mention
- **Type / role:** `Slack Trigger` (`n8n-nodes-base.slackTrigger`) — entry point for Slack events.
- **Configuration (interpreted):**
  - Trigger event: `app_mention`
  - Channel: configured via UI list (currently blank in JSON export; must be set)
- **Key data produced:** Slack event payload typically under `$json.event` (also sometimes directly on `$json` depending on Slack trigger version).
- **Connections:**
  - Output → **Extract Keywords & Detect Intent**
- **Edge cases / failures:**
  - Slack credentials/scopes missing (`app_mentions:read`)
  - Channel not selected (no events received)
  - Slack app event subscription misconfigured (request URL / signing secret)
- **Version notes:** typeVersion `1` (older Slack Trigger interface; ensure compatible with your n8n version).

**Sticky note applied:** “Trigger: Listens for `@bot` mentions…”

---

### Block 2.2 — Normalize, Keyword Extraction & Intent Detection
**Overview:** Removes the bot mention, detects intent (search/help/categories), extracts service keywords, and expands Japanese terms to English search words. Produces a normalized object used throughout downstream nodes.  
**Nodes involved:**  
- **Extract Keywords & Detect Intent**

#### Node: Extract Keywords & Detect Intent
- **Type / role:** `Code` (`n8n-nodes-base.code`) — parsing, classification, keyword extraction.
- **Configuration (interpreted):**
  - Reads Slack payload from `$input.first().json.event || $input.first().json`
  - Strips `<@BOTID>` mention tags via regex
  - Intent detection via regex patterns:
    - `help` patterns: `help|ayuda|ヘルプ|使い方|how to use|...`
    - `categories` patterns: `categories|categorías|カテゴリ|list|popular|...`
    - Default intent: `search`
  - Keyword extraction:
    - Scans against a **large list of known services/tools** (200+)
    - Japanese-to-English mapping dictionary expands matched Japanese terms into multiple English tokens
  - Builds `searchQuery`:
    - If keywords found: first 8 keywords joined
    - Else: uses the full `userRequest`
  - Output JSON fields:
    - `channel`, `threadTs` (uses `event.thread_ts || event.ts`), `userId`
    - `userRequest`, `searchQuery`, `intent`
    - `keywordsFound`, `timestamp`
- **Connections:**
  - Output → **Route by Intent**
- **Edge cases / failures:**
  - Slack payload shape changes (missing `.event`)
  - Empty messages after stripping mention (`userRequest` becomes empty)
  - Keyword explosion (many matches) but the query is capped to 8 keywords
  - Non-Japanese CJK terms won’t be expanded unless explicitly mapped
- **Version notes:** typeVersion `2`

**Sticky note applied:** “Normalize, Extract & Detect Intent…”

---

### Block 2.3 — Intent Routing
**Overview:** Routes execution to search, help, or categories responses based on `intent`.  
**Nodes involved:**  
- **Route by Intent**

#### Node: Route by Intent
- **Type / role:** `Switch` (`n8n-nodes-base.switch`) — branching by rules.
- **Configuration (interpreted):**
  - Switch rules compare `{{$json.intent}}` to:
    1. `"search"`
    2. `"help"`
    3. `"categories"`
- **Connections:**
  - Output 0 (search) → **Check Cache - Google Sheets**
  - Output 1 (help) → **Generate Help Response**
  - Output 2 (categories) → **Generate Categories Response**
- **Edge cases / failures:**
  - If intent has unexpected value, it won’t match any rule; execution may stop (depending on Switch settings).
- **Version notes:** typeVersion `3`

**Sticky note applied:** “Intent Router…”

---

### Block 2.4 — Help / Categories Direct Responses
**Overview:** Produces immediate Slack-ready text for help or category listing and forwards to Slack reply/logging.  
**Nodes involved:**  
- **Generate Help Response**
- **Generate Categories Response**

#### Node: Generate Help Response
- **Type / role:** `Code`
- **Configuration (interpreted):**
  - Creates Spanish help text (usage, commands, examples)
  - Sets `resultCount: 0`, `fromCache: false`
  - Preserves original extracted fields (`...data`)
- **Connections:**
  - Output → **Reply in Slack Thread**
- **Edge cases:**
  - Text language is hardcoded Spanish, regardless of user language
- **Version notes:** typeVersion `2`

#### Node: Generate Categories Response
- **Type / role:** `Code`
- **Configuration (interpreted):**
  - Creates Spanish “popular categories” message
  - Sets `resultCount: 0`, `fromCache: false`
- **Connections:**
  - Output → **Reply in Slack Thread**
- **Edge cases:**
  - Hardcoded categories list may drift from actual n8n template taxonomy
- **Version notes:** typeVersion `2`

---

### Block 2.5 — Cache Layer (Google Sheets)
**Overview:** Reads the “Cache” tab and decides whether to use a cached response or perform a live API search.  
**Nodes involved:**  
- **Check Cache - Google Sheets**
- **Cache Hit?**
- **Use Cached Result**

#### Node: Check Cache - Google Sheets
- **Type / role:** `Google Sheets` — reads cache records.
- **Configuration (interpreted):**
  - Document ID: must be selected (currently blank placeholder)
  - Sheet/tab: `Cache` (must exist)
  - Operation not specified in JSON → default for this node is typically **Read / Get Many** (depends on node UI defaults)
- **Connections:**
  - Output → **Cache Hit?**
- **Edge cases / failures:**
  - Google OAuth not configured
  - Sheet/tab not found
  - Large sheet read may be slow; no server-side filter is configured here
- **Version notes:** typeVersion `4.5`

#### Node: Cache Hit?
- **Type / role:** `IF`
- **Configuration (interpreted):**
  - Condition: checks whether `$json.SearchQuery` **exists**
  - This implies the incoming row structure must have a `SearchQuery` column.
- **Connections:**
  - True → **Use Cached Result**
  - False → **Search n8n Templates API**
- **Critical logic caveat (important):**
  - This checks only for existence of `SearchQuery` in the *current item*, not whether it matches the user’s `searchQuery`, nor whether it’s “recent”.
  - Because the preceding Sheets node likely returns *many rows*, this IF may behave unpredictably (e.g., the first returned row has `SearchQuery`, so it becomes a “hit” even if it’s unrelated).
- **Version notes:** typeVersion `2`

#### Node: Use Cached Result
- **Type / role:** `Code` — merges cache row with Slack context.
- **Configuration (interpreted):**
  - Takes first cache row as `cached`
  - Pulls Slack context from `$('Extract Keywords & Detect Intent').first().json`
  - Produces:
    - `text` = `cached.CachedResponse` (fallback string if missing)
    - `resultCount` from `cached.ResultCount`
    - `fromCache: true`
- **Connections:**
  - Output → **Reply in Slack Thread**
- **Edge cases:**
  - If the sheet returns multiple rows, using `.first()` may not be the best match.
  - CachedResponse may be stale or formatted incorrectly.
- **Version notes:** typeVersion `2`

**Sticky note applied:** “Cache Layer…”

---

### Block 2.6 — Template Search (n8n API) + Result Decision
**Overview:** Queries the n8n Templates search endpoint and decides whether results exist.  
**Nodes involved:**  
- **Search n8n Templates API**
- **Has Results?**

#### Node: Search n8n Templates API
- **Type / role:** `HTTP Request`
- **Configuration (interpreted):**
  - GET `https://api.n8n.io/templates/search`
  - Query parameters:
    - `search` = `Extract Keywords & Detect Intent.searchQuery`
    - `rows` = `5`
  - Timeout: 30s
- **Connections:**
  - Output → **Has Results?**
- **Edge cases / failures:**
  - Network errors/timeouts
  - API changes in response shape (expects `workflows` array)
  - Rate limiting from API endpoint (not handled explicitly)
- **Version notes:** typeVersion `4.2`

#### Node: Has Results?
- **Type / role:** `IF` — checks if any workflows returned.
- **Configuration (interpreted):**
  - Condition: `{{ ($json.workflows || []).length }} > 0`
- **Connections:**
  - True → **AI Generate Tips & Suggestions**
  - False → **AI Suggest Alternatives**
- **Edge cases:**
  - If API returns different property name, this will always evaluate to 0 results.
- **Version notes:** typeVersion `2`

**Sticky note applied:** “Search + AI Tips…”

---

### Block 2.7 — AI Enrichment (OpenAI)
**Overview:** Calls OpenAI Chat Completions to either generate actionable tips (when templates found) or suggest alternative search terms and manual build tips (when none found).  
**Nodes involved:**  
- **AI Generate Tips & Suggestions**
- **AI Suggest Alternatives**

#### Node: AI Generate Tips & Suggestions
- **Type / role:** `HTTP Request` — OpenAI Chat Completions.
- **Configuration (interpreted):**
  - POST `https://api.openai.com/v1/chat/completions`
  - Model: `gpt-4o-mini`
  - `max_tokens: 500`, `temperature: 0.7`
  - System prompt: n8n automation expert; return 3–5 numbered tips; no emojis; match user language.
  - User prompt includes:
    - Original query: `Extract...userRequest`
    - Search keywords: `Extract...searchQuery`
    - Templates found: from current API response `workflows.map(w => w.name).join(', ')`
  - Authentication: `genericCredentialType` using `httpHeaderAuth` (expects an Authorization header, typically `Bearer <OPENAI_API_KEY>`)
  - Timeout: 60s
- **Connections:**
  - Output → **Format Results + Tips**
- **Edge cases / failures:**
  - Missing/invalid OpenAI key → 401
  - Model name not available in account/region
  - Rate limits (429)
  - Response structure missing `choices[0].message.content`
- **Version notes:** typeVersion `4.2`

#### Node: AI Suggest Alternatives
- **Type / role:** `HTTP Request` — OpenAI Chat Completions for “no results” path.
- **Configuration (interpreted):**
  - Same endpoint/model family
  - `max_tokens: 400`
  - System prompt: suggest 3 alternative search terms + 2–3 build tips; no emojis; match user language.
  - User prompt includes query + used keywords.
  - Authentication: same header auth
- **Connections:**
  - Output → **Format No Results + Suggestions**
- **Edge cases / failures:** same as above
- **Version notes:** typeVersion `4.2`

---

### Block 2.8 — Formatting, Slack Reply, and Logging
**Overview:** Builds the final Slack message, posts it, then logs analytics and updates the cache.  
**Nodes involved:**  
- **Format Results + Tips**
- **Format No Results + Suggestions**
- **Reply in Slack Thread**
- **Log to Analytics Sheet**
- **Save to Cache Sheet**

#### Node: Format Results + Tips
- **Type / role:** `Code` — composes Slack markdown message.
- **Configuration (interpreted):**
  - Reads Slack context from `Extract Keywords & Detect Intent`
  - Reads workflows from `Has Results?` node output (expects `workflows`)
  - Builds up to 5 links: `https://n8n.io/workflows/{id}|{name}`
  - Extracts AI tips from current input: `choices[0].message.content`
  - Produces final `text`, `resultCount`, `fromCache:false`
  - Adds footer `_Powered by n8n Template Bot_`
- **Connections:**
  - Output → **Reply in Slack Thread**
- **Edge cases:**
  - If `Has Results?` output item doesn’t contain workflows (because IF nodes can pass data differently), `workflows` may be empty.
  - Slack markdown may break if workflow names contain special characters (rare).
- **Version notes:** typeVersion `2`

#### Node: Format No Results + Suggestions
- **Type / role:** `Code` — composes “no results” Slack message.
- **Configuration (interpreted):**
  - Pulls AI suggestions from `choices[0].message.content`
  - Spanish copy: “No encontre templates exactos…”
  - Produces `resultCount:0`, `fromCache:false`
- **Connections:**
  - Output → **Reply in Slack Thread**
- **Edge cases:**
  - Hardcoded Spanish text may conflict with “reply in user language” intention.
- **Version notes:** typeVersion `2`

#### Node: Reply in Slack Thread
- **Type / role:** `Slack` — posts message back to Slack.
- **Configuration (interpreted):**
  - Posts `{{$json.text}}`
  - Channel ID: `{{$json.channel}}`
  - `mrkdwn: true`
  - **Important:** despite the name, this node does **not** set `thread_ts`. So it will post in-channel, not as a thread reply, unless Slack node defaults are configured elsewhere (they are not shown here).
- **Connections:**
  - Output → **Log to Analytics Sheet** and **Save to Cache Sheet** (parallel)
- **Edge cases / failures:**
  - Missing `chat:write` scope or invalid Slack token
  - Channel ID invalid/not accessible by the app
  - If you intended thread replies, missing `thread_ts` is a functional bug
- **Version notes:** typeVersion `2.2`

#### Node: Log to Analytics Sheet
- **Type / role:** `Google Sheets` — append analytics row.
- **Configuration (interpreted):**
  - Operation: `append`
  - Sheet: `Analytics` tab (must exist)
  - Document ID: must be set
  - **Missing mapping:** No explicit “columns/fields to append” are shown in JSON; depending on node UI defaults, it may append the entire JSON object or require manual field mapping. You should configure it to write:
    - Timestamp, User, Query, Keywords, ResultCount, Intent, FromCache
- **Connections:** none after append
- **Edge cases:**
  - Schema mismatch (headers not matching keys)
  - Data types: booleans may be stored as TRUE/FALSE strings
- **Version notes:** typeVersion `4.5`

#### Node: Save to Cache Sheet
- **Type / role:** `Google Sheets` — append cache entry.
- **Configuration (interpreted):**
  - Operation: `append`
  - Sheet: `Cache` tab
  - Intended fields (per sticky note): SearchQuery, CachedResponse, ResultCount, Timestamp
  - **As with analytics:** ensure correct column mapping is configured.
- **Connections:** none after append
- **Edge cases:**
  - Without a “lookup by SearchQuery”, the cache will grow indefinitely and reads will slow down.
- **Version notes:** typeVersion `4.5`

**Sticky note applied:** “Format, Reply & Log…”

---

### Block 2.9 — Error Handling
**Overview:** On workflow failure, posts a friendly message to Slack so the user gets feedback.  
**Nodes involved:**  
- **Error Trigger**
- **Error Reply in Slack**

#### Node: Error Trigger
- **Type / role:** `Error Trigger` — workflow-level error entry point.
- **Configuration:** default; triggers when the workflow errors.
- **Connections:**
  - Output → **Error Reply in Slack**
- **Edge cases:**
  - Only triggers on errors; it does not catch logical “no results” situations (those are handled separately).
- **Version notes:** typeVersion `1`

#### Node: Error Reply in Slack
- **Type / role:** `Slack` — posts an error message.
- **Configuration (interpreted):**
  - Hardcoded English error text with suggestions (`@bot help`)
  - **ChannelId is blank** in JSON; must be configured, or computed from the error payload (preferred).
- **Connections:** none
- **Edge cases / failures:**
  - Won’t know where to post unless channel is set or derived from error event
  - If error occurs before Slack context exists, you may not have thread/channel data
- **Version notes:** typeVersion `2.2`

**Sticky note applied:** “Error Handling…”

---

### Block 2.10 — Weekly Analytics Summary
**Overview:** Weekly scheduled job reads analytics sheet, aggregates last 7 days, and posts summary to Slack.  
**Nodes involved:**  
- **Weekly Analytics Cron**
- **Read Analytics Data**
- **Generate Weekly Summary**
- **Post Weekly Summary to Slack**

#### Node: Weekly Analytics Cron
- **Type / role:** `Schedule Trigger`
- **Configuration (interpreted):**
  - Runs weekly: `triggerAtDay: [1]` and `triggerAtHour: 9`
  - Note: exact interpretation depends on n8n locale/week start; commonly `1` = Monday.
- **Connections:**
  - Output → **Read Analytics Data**
- **Edge cases:**
  - Timezone: schedule uses workflow timezone setting; verify it matches your team’s timezone.
- **Version notes:** typeVersion `1.2`

#### Node: Read Analytics Data
- **Type / role:** `Google Sheets` — reads analytics tab.
- **Configuration (interpreted):**
  - Reads from `Analytics` tab of selected spreadsheet
  - Operation likely default read/get many
- **Connections:**
  - Output → **Generate Weekly Summary**
- **Edge cases:**
  - Large sheet read can be slow; no filtering is applied at source.
- **Version notes:** typeVersion `4.5`

#### Node: Generate Weekly Summary
- **Type / role:** `Code` — aggregates analytics.
- **Configuration (interpreted):**
  - Filters rows where `new Date(r.Timestamp) >= oneWeekAgo`
  - Counts:
    - total searches (rows)
    - intent breakdown
    - average results per search
    - cache hits (`FromCache === 'true'` or boolean true)
  - Top searches:
    - Uses `(row.Keywords || row.Query || '')` lowercased as the “query key”
    - Top 10 list
  - Outputs Slack markdown text
- **Connections:**
  - Output → **Post Weekly Summary to Slack**
- **Edge cases:**
  - Timestamp parsing fails if Timestamp column is not ISO-like
  - If your analytics columns are named differently, stats will be wrong/empty
- **Version notes:** typeVersion `2`

#### Node: Post Weekly Summary to Slack
- **Type / role:** `Slack` — posts weekly report.
- **Configuration (interpreted):**
  - Posts `{{$json.text}}`
  - Channel must be selected (currently blank placeholder)
  - `mrkdwn: true`
- **Connections:** none
- **Edge cases:**
  - Channel not set → message fails
- **Version notes:** typeVersion `2.2`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | Sticky Note | Documentation / canvas annotation | — | — | # n8n Template Search Bot for Slack (Enhanced)… (features/setup/how it works) |
| Sticky Note - Trigger | Sticky Note | Documentation / canvas annotation | — | — | Trigger: Listens for `@bot` mentions… |
| Sticky Note - Normalize | Sticky Note | Documentation / canvas annotation | — | — | Normalize, Extract & Detect Intent… |
| Sticky Note - Intent Router | Sticky Note | Documentation / canvas annotation | — | — | Intent Router… |
| Sticky Note - Cache | Sticky Note | Documentation / canvas annotation | — | — | Cache Layer… |
| Sticky Note - Search & AI | Sticky Note | Documentation / canvas annotation | — | — | Search + AI Tips… |
| Sticky Note - Reply & Log | Sticky Note | Documentation / canvas annotation | — | — | Format, Reply & Log… |
| Sticky Note - Error Handling | Sticky Note | Documentation / canvas annotation | — | — | Error Handling… |
| Slack Trigger - Bot Mention | Slack Trigger | Receives Slack app mentions | — | Extract Keywords & Detect Intent | Trigger: Listens for `@bot` mentions… |
| Extract Keywords & Detect Intent | Code | Normalize text, detect intent, extract keywords | Slack Trigger - Bot Mention | Route by Intent | Normalize, Extract & Detect Intent… |
| Route by Intent | Switch | Branching by `intent` | Extract Keywords & Detect Intent | Check Cache - Google Sheets; Generate Help Response; Generate Categories Response | Intent Router… |
| Generate Help Response | Code | Create help message | Route by Intent | Reply in Slack Thread | Intent Router… |
| Generate Categories Response | Code | Create categories message | Route by Intent | Reply in Slack Thread | Intent Router… |
| Check Cache - Google Sheets | Google Sheets | Read cache sheet | Route by Intent | Cache Hit? | Cache Layer… |
| Cache Hit? | IF | Decide cache hit vs live search | Check Cache - Google Sheets | Use Cached Result; Search n8n Templates API | Cache Layer… |
| Use Cached Result | Code | Build response from cached row | Cache Hit? | Reply in Slack Thread | Cache Layer… |
| Search n8n Templates API | HTTP Request | Query templates API | Cache Hit? | Has Results? | Search + AI Tips… |
| Has Results? | IF | Check workflows array length | Search n8n Templates API | AI Generate Tips & Suggestions; AI Suggest Alternatives | Search + AI Tips… |
| AI Generate Tips & Suggestions | HTTP Request | OpenAI tips when results exist | Has Results? | Format Results + Tips | Search + AI Tips… |
| AI Suggest Alternatives | HTTP Request | OpenAI alternatives when no results | Has Results? | Format No Results + Suggestions | Search + AI Tips… |
| Format Results + Tips | Code | Slack text formatting with links + AI tips | AI Generate Tips & Suggestions | Reply in Slack Thread | Format, Reply & Log… |
| Format No Results + Suggestions | Code | Slack text formatting (no results) + AI suggestions | AI Suggest Alternatives | Reply in Slack Thread | Format, Reply & Log… |
| Reply in Slack Thread | Slack | Post message to Slack | Use Cached Result / Format Results + Tips / Format No Results + Suggestions / Help / Categories | Log to Analytics Sheet; Save to Cache Sheet | Format, Reply & Log… |
| Log to Analytics Sheet | Google Sheets | Append analytics row | Reply in Slack Thread | — | Format, Reply & Log… |
| Save to Cache Sheet | Google Sheets | Append cache row | Reply in Slack Thread | — | Format, Reply & Log… |
| Error Trigger | Error Trigger | Workflow error entry point | — | Error Reply in Slack | Error Handling… |
| Error Reply in Slack | Slack | Notify Slack on errors | Error Trigger | — | Error Handling… |
| Weekly Analytics Cron | Schedule Trigger | Weekly schedule | — | Read Analytics Data |  |
| Read Analytics Data | Google Sheets | Read analytics rows | Weekly Analytics Cron | Generate Weekly Summary |  |
| Generate Weekly Summary | Code | Aggregate 7-day metrics | Read Analytics Data | Post Weekly Summary to Slack |  |
| Post Weekly Summary to Slack | Slack | Publish weekly report | Generate Weekly Summary | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Slack App + credentials**
   1. Create a Slack app and enable:
      - Event subscriptions: subscribe to `app_mention`
      - OAuth scopes: `app_mentions:read`, `chat:write`
   2. In n8n, create **Slack OAuth2 credentials** (or Slack app credentials as required by your n8n Slack nodes).
   3. Install the Slack app to your workspace/channel.

2) **Create Google Sheet**
   1. Create one spreadsheet with two tabs:
      - **Cache** with headers: `SearchQuery`, `CachedResponse`, `ResultCount`, `Timestamp`
      - **Analytics** with headers: `Timestamp`, `User`, `Query`, `Keywords`, `ResultCount`, `Intent`, `FromCache`
   2. In n8n, configure **Google Sheets OAuth2 credentials**.

3) **Add Slack Trigger**
   1. Add node: **Slack Trigger** named `Slack Trigger - Bot Mention`
   2. Trigger event: `app_mention`
   3. Select the channel to listen to
   4. Connect to next node.

4) **Add Code node: intent + keyword extraction**
   1. Add node: **Code** named `Extract Keywords & Detect Intent`
   2. Paste logic that:
      - Removes `<@...>` mention
      - Computes `intent` (search/help/categories)
      - Extracts keywords from known services list
      - Expands Japanese terms via mapping
      - Outputs: `channel`, `threadTs`, `userId`, `userRequest`, `searchQuery`, `intent`, `keywordsFound`, `timestamp`
   3. Connect Slack Trigger → this Code node.

5) **Add Switch node: Route by intent**
   1. Add node: **Switch** named `Route by Intent`
   2. Create 3 rules on `{{$json.intent}}` equals:
      - `search`
      - `help`
      - `categories`
   3. Connect Code → Switch.

6) **Help branch**
   1. Add **Code** node `Generate Help Response` that sets a Slack-ready `text`
   2. Output should keep original fields and add: `text`, `resultCount: 0`, `fromCache:false`
   3. Connect Switch (help output) → Help Code.

7) **Categories branch**
   1. Add **Code** node `Generate Categories Response` similarly producing `text`
   2. Connect Switch (categories output) → Categories Code.

8) **Search branch: Cache check**
   1. Add **Google Sheets** node `Check Cache - Google Sheets`
      - Document: your spreadsheet
      - Sheet: `Cache`
      - Operation: Read/Get Many
   2. Connect Switch (search output) → this node.
   3. Add **IF** node `Cache Hit?`
      - Recommended configuration (fixing the current logic):
        - Find a row where `SearchQuery` equals `{{$node["Extract Keywords & Detect Intent"].json.searchQuery}}`
        - Optionally ensure timestamp recency (e.g., within last X days)
      - If you keep the original workflow behavior, it only checks `SearchQuery exists`, which is not a true cache match.
   4. Connect Sheets → IF.

9) **Cache hit path**
   1. Add **Code** node `Use Cached Result`
      - Merge the cached row with Slack context
      - Output `text = CachedResponse`, `resultCount`, `fromCache:true`
   2. Connect IF (true) → `Use Cached Result`.

10) **Cache miss path: Search n8n templates**
   1. Add **HTTP Request** node `Search n8n Templates API`
      - GET `https://api.n8n.io/templates/search`
      - Query params:
        - `search` = `{{$node["Extract Keywords & Detect Intent"].json.searchQuery}}`
        - `rows` = `5`
      - Timeout: 30000ms
   2. Connect IF (false) → HTTP node.
   3. Add **IF** node `Has Results?`
      - Condition: `{{ ($json.workflows || []).length }} > 0`
   4. Connect HTTP → `Has Results?`.

11) **OpenAI credentials**
   1. In n8n, create an HTTP Header Auth credential for OpenAI:
      - Header: `Authorization`
      - Value: `Bearer <YOUR_OPENAI_API_KEY>`
   2. (Optional) Also set `Content-Type: application/json` if needed; n8n usually sets it automatically for JSON.

12) **Results exist path: AI tips**
   1. Add **HTTP Request** node `AI Generate Tips & Suggestions`
      - POST `https://api.openai.com/v1/chat/completions`
      - JSON body using:
        - model `gpt-4o-mini`
        - system/user messages as in the workflow
      - Timeout 60000ms
      - Auth: your OpenAI header auth credential
   2. Add **Code** node `Format Results + Tips`
      - Create Slack markdown list of template links
      - Append AI tips from `choices[0].message.content`
      - Output: `text`, `resultCount`, `fromCache:false`
   3. Connect `Has Results?` (true) → OpenAI Tips → Format Results.

13) **No results path: AI alternatives**
   1. Add **HTTP Request** node `AI Suggest Alternatives` (POST chat completions)
   2. Add **Code** node `Format No Results + Suggestions`
      - Output Slack text with suggestions
   3. Connect `Has Results?` (false) → OpenAI Alternatives → Format No Results.

14) **Slack reply node**
   1. Add **Slack** node `Reply in Slack Thread`
      - Channel: `{{$json.channel}}`
      - Text: `{{$json.text}}`
      - Enable markdown
      - Recommended fix: set **thread_ts** to `{{$json.threadTs}}` so it truly replies in thread.
   2. Connect outputs of:
      - `Use Cached Result`
      - `Format Results + Tips`
      - `Format No Results + Suggestions`
      - `Generate Help Response`
      - `Generate Categories Response`
      → into this Slack node.

15) **Logging & caching writes**
   1. Add **Google Sheets** node `Log to Analytics Sheet` (operation: Append)
      - Sheet: `Analytics`
      - Map fields explicitly, e.g.:
        - Timestamp = `{{$json.timestamp}}`
        - User = `{{$json.userId}}`
        - Query = `{{$json.userRequest}}`
        - Keywords = `{{$json.keywordsFound.join(", ")}}`
        - ResultCount = `{{$json.resultCount}}`
        - Intent = `{{$json.intent}}`
        - FromCache = `{{$json.fromCache}}`
   2. Add **Google Sheets** node `Save to Cache Sheet` (operation: Append)
      - Sheet: `Cache`
      - Map:
        - SearchQuery = `{{$json.searchQuery}}`
        - CachedResponse = `{{$json.text}}`
        - ResultCount = `{{$json.resultCount}}`
        - Timestamp = `{{$json.timestamp}}`
   3. Connect `Reply in Slack Thread` → both Sheets append nodes.

16) **Error handling**
   1. Add **Error Trigger** node `Error Trigger`
   2. Add **Slack** node `Error Reply in Slack`
      - Configure a fallback channel, or (better) derive from error context if available.
      - Post a friendly message.
   3. Connect Error Trigger → Error Slack node.

17) **Weekly analytics reporting**
   1. Add **Schedule Trigger** `Weekly Analytics Cron`
      - Weekly at day=1, hour=9 (adjust timezone in workflow settings)
   2. Add **Google Sheets** `Read Analytics Data` (read/get many from `Analytics`)
   3. Add **Code** `Generate Weekly Summary` to aggregate last 7 days and output `text`
   4. Add **Slack** `Post Weekly Summary to Slack`
      - Select a reporting channel
      - Post `{{$json.text}}`
   5. Connect Cron → Read → Code → Slack.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow intends “reply in thread”, but the Slack post node does not set `thread_ts`. Add `thread_ts = {{$json.threadTs}}` to reply properly. | Functional correctness |
| The cache “hit” logic only checks whether `SearchQuery` exists in a sheet row, not whether it matches the user query. Implement a lookup/filter by `searchQuery` and optionally by recency. | Cache correctness/performance |
| OpenAI is called via HTTP Request + header auth; ensure `Authorization: Bearer ...` is set. | OpenAI integration |
| Google Sheets nodes must have explicit column mapping to ensure correct analytics/cache schema. | Data integrity |
| Disclaimer (provided): “Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…” | Compliance/context statement |