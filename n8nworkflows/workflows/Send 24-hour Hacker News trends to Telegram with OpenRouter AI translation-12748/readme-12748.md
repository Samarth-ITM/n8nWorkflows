Send 24-hour Hacker News trends to Telegram with OpenRouter AI translation

https://n8nworkflows.xyz/workflows/send-24-hour-hacker-news-trends-to-telegram-with-openrouter-ai-translation-12748


# Send 24-hour Hacker News trends to Telegram with OpenRouter AI translation

## 1. Workflow Overview

**Purpose:** This workflow runs every 4 hours, fetches the last 24 hours of “Ask HN” and “Show HN” posts via the Algolia Hacker News API, recalculates a *trend/velocity score* to detect fast-rising stories (while filtering early “noise”), translates title and story content via **OpenRouter (Gemini model)** into a target language (configured as Chinese), formats a Telegram-friendly HTML message, and sends it to a specified Telegram chat.

**Target use cases:**
- Automated digest of trending HN posts with “momentum” scoring (not just raw points)
- Multi-language delivery of HN summaries to Telegram (personal chat or channel)
- Avoiding immature posts that have not had time to accumulate meaningful engagement

### 1.1 Scheduling & Sliding Window Setup
Runs on a schedule (every 4 hours) and defines a 24-hour lookback window plus Algolia query filters.

### 1.2 Data Fetch (Algolia HN Search API)
Calls Algolia’s `/search_by_date` endpoint with tags and numeric filters.

### 1.3 Trend Scoring & Filtering
Computes `velocity_score` using a gravity-style normalization by age, drops “infant” posts with low early points, sorts, and applies a minimum score threshold.

### 1.4 AI Translation (OpenRouter)
Uses an AI Agent node with an OpenRouter chat model to translate `title` and `story_text` to the chosen language and output strict JSON.

### 1.5 Merge + Message Formatting + Telegram Send
Merges translations back into the original items, builds one combined Telegram message (HTML parse mode), and sends it.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Sliding Window Setup

**Overview:** Triggers execution every 4 hours and prepares Algolia query parameters for a 24-hour lookback window, including basic anti-noise engagement filtering.

**Nodes involved:**
- Schedule Trigger
- Algolia parameters
- Sticky Note (commentary)

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — periodic workflow entry point
- **Configuration:** Runs every **4 hours**.
- **Inputs / outputs:** No inputs; outputs to **Algolia parameters**.
- **Edge cases / failures:** None typical; schedule depends on n8n instance uptime and timezone config (`Etc/UTC` in workflow settings).
- **Version specifics:** typeVersion `1.3`.

#### Node: Algolia parameters
- **Type / role:** `n8n-nodes-base.code` — constructs query parameters dynamically
- **Key configuration choices:**
  - Uses Luxon `DateTime.now()` (available in n8n Code node runtime) to compute:
    - `lookBackHours = 24`
    - `startTime` as UNIX seconds of now minus 24h
  - Sets:
    - `tags: "(ask_hn,show_hn)"`
    - `numericFilters`: `created_at_i>startTime,points>1`
    - `hitsPerPage: 1000`
- **Key variables / expressions:**
  - `DateTime.now()`
  - `now.minus({hours: lookBackHours}).toSeconds()`
  - `numericFilters.join(",")`
- **Inputs / outputs:** Input from **Schedule Trigger**; outputs one item containing query config to **Request Algolia**.
- **Edge cases / failures:**
  - If Luxon `DateTime` is unavailable (older n8n / restricted runtime), code fails.
  - `hitsPerPage: 1000` depends on Algolia constraints; if API changes limit, results may be capped.
- **Version specifics:** typeVersion `2`.

#### Node: Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — documentation overlay
- **Content highlights:**
  - Uses Algolia API: https://hn.algolia.com/api
  - Sliding window: every 4 hours, look back 24h
  - Noise rule concept (discard very new + very low interaction; keep “explosive” posts)
- **Execution impact:** None (non-executable).

---

### Block 2 — Data Fetch (Algolia HN Search API)

**Overview:** Queries Algolia for posts in the last 24 hours matching selected tags and minimum engagement.

**Nodes involved:**
- Request Algolia

#### Node: Request Algolia
- **Type / role:** `n8n-nodes-base.httpRequest` — HTTP GET to Algolia
- **Configuration choices:**
  - **URL:** `http://hn.algolia.com/api/v1/search_by_date`
  - `sendQuery: true`
  - Query parameters sourced from previous Code node:
    - `tags = {{$json.tags}}`
    - `numericFilters = {{$json.numericFilters}}`
    - `hitsPerPage = {{$json.hitsPerPage}}`
- **Inputs / outputs:** Input from **Algolia parameters**; output JSON contains Algolia response (notably `hits`) to **Recalculate popularity score**.
- **Edge cases / failures:**
  - Network/DNS failures, timeouts, rate limiting.
  - Using **HTTP** (not HTTPS). Some environments/proxies may block plain HTTP; consider switching to `https://hn.algolia.com/...`.
  - If Algolia response structure changes (e.g., `hits` missing), downstream code fails.
- **Version specifics:** typeVersion `4.3`.

---

### Block 3 — Trend Scoring & Filtering

**Overview:** Converts raw Algolia hits into ranked trending items by computing a normalized velocity score, discarding immature low-signal posts, and filtering by score threshold.

**Nodes involved:**
- Recalculate popularity score
- Filter

#### Node: Recalculate popularity score
- **Type / role:** `n8n-nodes-base.code` — transforms Algolia `hits[]` into scored items
- **Configuration choices / algorithm:**
  - Reads `items[0].json.hits`
  - For each hit:
    - Compute `ageInHours` from `created_at` to now
    - Extract `points` and `num_comments`
  - **Infancy filtering (“buffer”):**
    - If `ageInHours < 1.0` and `points < 5` → discard (`return null`)
  - **Velocity score (gravity model):**
    - `rawScore = points + (comments * 2)`
    - `gravity = 1.8`
    - `normalizedScore = (rawScore - 1) / (ageInHours + 2)^gravity`
  - Adds:
    - `analysis.age_hours` (2 decimals)
    - `analysis.velocity_score` (4 decimals)
  - Filters nulls, sorts descending by `velocity_score`
  - Outputs each hit as its own n8n item
- **Inputs / outputs:** Input from **Request Algolia**; outputs many items to **Filter**.
- **Edge cases / failures:**
  - If `hits` is undefined or not an array → runtime error.
  - If `created_at` is invalid ISO → Luxon parse issues leading to NaN ages.
  - Extreme values (very new posts) handled by `(ageInHours + 2)` to avoid division by zero, but negative ages (clock skew) could inflate scores.
- **Version specifics:** typeVersion `2`.

#### Node: Filter
- **Type / role:** `n8n-nodes-base.filter` — keeps only sufficiently “trending” items
- **Configuration choices:**
  - Condition: `{{$json.analysis.velocity_score}} > 1`
  - Strict type validation enabled by node’s versioned condition system.
- **Inputs / outputs:** Input from **Recalculate popularity score**; output items to **Translate**.
- **Edge cases / failures:**
  - If `analysis.velocity_score` missing/non-numeric, strict validation may drop items or error depending on runtime behavior.
  - Threshold tuning: `> 1` may be too strict/lenient based on traffic; can result in empty output (no Telegram message content beyond header).
- **Version specifics:** typeVersion `2.3`.

---

### Block 4 — AI Translation (OpenRouter)

**Overview:** Sends each filtered post’s title and story text to an OpenRouter-backed chat model through an AI Agent node, instructing it to return JSON-only translated fields.

**Nodes involved:**
- OpenRouter Chat Model1
- Translate
- Sticky Note1 (commentary)

#### Node: OpenRouter Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides LLM chat model via OpenRouter
- **Configuration choices:**
  - Model: `google/gemini-2.5-flash-lite`
  - No extra options configured
- **Credentials:** OpenRouter API credential `OpenRouter-William`
- **Inputs / outputs (AI port):**
  - Connected to **Translate** via `ai_languageModel` output → Agent’s model input.
- **Edge cases / failures:**
  - OpenRouter auth/credit errors, model availability changes, rate limits.
  - Output format adherence depends on model compliance; downstream parsing expects JSON.
- **Version specifics:** typeVersion `1`.

#### Node: Translate
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent that performs translation
- **Configuration choices:**
  - **Prompt type:** “define”
  - **Input text field:**  
    - `title: {{ JSON.stringify($json.title) }}`
    - `story_text: {{ JSON.stringify($json.story_text) }}`
    - JSON.stringify reduces prompt-breaking from quotes/newlines and makes empties explicit.
  - **System message (core behavior):**
    - “Professional news media translation API”
    - Translate `title` and `story_text` into **Chinese**
    - If `story_text` empty → keep original text
    - Strip HTML; convert `<a href>` to Markdown `[text](url)`
    - Output **ONLY** a valid JSON object (no code fences, no extra text)
  - **Error handling:** `onError: continueRegularOutput` (workflow continues even if model call fails)
- **Inputs / outputs:**
  - Main input from **Filter**
  - Model input from **OpenRouter Chat Model1**
  - Main output to **Merge translation results**
- **Edge cases / failures:**
  - Model returns non-JSON or wraps JSON in markdown fences → handled downstream by sanitization, but not guaranteed.
  - If OpenRouter call fails, output may be missing `item.json.output` (downstream merges may mark translation_error or produce empty translated fields).
- **Version specifics:** typeVersion `3.1`.

#### Node: Sticky Note1
- **Type / role:** `n8n-nodes-base.stickyNote` — translation target reminder
- **Content:** “Replace with your target language… Must be Spanish, French, Chinese, Japanese…”
- **Execution impact:** None.

---

### Block 5 — Merge + Message Formatting + Telegram Send

**Overview:** Because the Agent node output does not preserve the full original item, the workflow re-attaches original fields, formats a single HTML Telegram message listing all trending items, and sends it.

**Nodes involved:**
- Merge translation results
- Combine message templates
- Send a text message
- Sticky Note2 (commentary)

#### Node: Merge translation results
- **Type / role:** `n8n-nodes-base.code` — merges LLM output back with original items
- **Configuration choices / logic:**
  - Retrieves originals with: `$('Filter').all()`  
    - **Important dependency:** the upstream node must be named exactly **“Filter”** (as the comment notes).
  - For each AI result item (mapped by index):
    - Reads `item.json.output` (LLM agent output string)
    - Removes ```json and ``` fences via regex replace
    - Attempts `JSON.parse`
      - On parse failure: sets `{ translation_error: true, raw_ai_response: aiOutputString }`
    - Merges:
      - `...originalData` then `...translatedData`
- **Inputs / outputs:** Input from **Translate**; output items to **Combine message templates**.
- **Edge cases / failures:**
  - **Index alignment risk:** Assumes AI outputs are in the same order and same count as `Filter` items. If an item errors, is retried, or dropped, translations may mismatch originals.
  - If `Filter` produces 0 items, this code returns empty.
  - If AI output contains leading text beyond code fences, JSON parse still fails.
- **Version specifics:** typeVersion `2`.

#### Node: Combine message templates
- **Type / role:** `n8n-nodes-base.code` — builds one combined Telegram HTML message
- **Configuration choices / formatting:**
  - Collects all items: `$input.all()`
  - Escapes HTML entities for safety (`& < >`)
  - Cleans `story_text`:
    - Converts `<p>` and `<br>` to newlines
    - Removes all other HTML tags
    - Decodes some entities (`&quot;`, `&#x27;`, `&amp;`)
    - Truncates to **120 chars**
    - Escapes HTML again
  - Message layout:
    - Header: `<b>🚨 Hacker News 24H Trending Alert</b>`
    - Count of stories
    - For each story:
      - Title as clickable HN item link: `https://news.ycombinator.com/item?id={{objectID}}`
      - Metrics line in `<code>` including velocity score, comments, points
      - Age in hours
      - Optional `<blockquote>` preview if `story_text` exists
  - Outputs single item: `{ json: { text: "<final html>" } }`
- **Inputs / outputs:** Input from **Merge translation results**; output to **Send a text message**.
- **Edge cases / failures:**
  - Telegram message length limit (Telegram has maximum message size). With many items, the combined message may exceed limits and fail. Consider limiting top N items or splitting into multiple messages.
  - The workflow currently uses `data.title` and `data.story_text` for display; if you intended to display translated fields, you must change to use the translated keys returned by the agent (e.g., `data.title` may already be translated if AI returns the same keys; otherwise message remains original).
- **Version specifics:** typeVersion `2`.

#### Node: Send a text message
- **Type / role:** `n8n-nodes-base.telegram` — sends final message to Telegram
- **Configuration choices:**
  - Operation: send text message
  - `chatId: "YourChatID"` (placeholder)
  - `text: {{$json.text}}`
  - `parse_mode: HTML` (required for `<b>`, `<code>`, `<a>`, `<blockquote>`)
- **Credentials:** Telegram bot credential `Telegram-CryoZeroRssBot`
- **Inputs / outputs:** Input from **Combine message templates**; end of workflow.
- **Edge cases / failures:**
  - Wrong Chat ID / bot not admin in channel → 400/403 errors.
  - HTML parse errors if message contains unsupported tags or unescaped characters (partially mitigated by escaping).
- **Version specifics:** typeVersion `1.2`.

#### Node: Sticky Note2
- **Type / role:** `n8n-nodes-base.stickyNote` — Telegram setup instructions
- **Content highlights:**
  - Create bot via `@BotFather`, save token in credentials
  - Get chat ID via `@userinfobot` (personal) or channel suffix `-100xxxxxxx` and bot as admin
  - Ensure bot permission to post
- **Execution impact:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Time-based entry point (every 4 hours) | — | Algolia parameters | # Hacker News Trend Tracking  \nTrack popular via https://hn.algolia.com/api  \n## Sliding Window  \nTimer (Trigger): Run every 4 hours. Lookback: Fetch data from the past 24 hours.  \n- Define a rule—"Posts published less than 1 hour ago and with extremely low interaction (<3 points) are considered noise and should be discarded directly."  \n- If a post receives 20 likes in just 10 minutes (extremely explosive), we should keep it; but if it only gets 1 like after 10 minutes (normal situation), we temporarily ignore it and wait for the next scheduled task (e.g., 4 hours later) to evaluate it when it "matures." |
| Algolia parameters | n8n-nodes-base.code | Build Algolia query params (24h lookback) | Schedule Trigger | Request Algolia | # Hacker News Trend Tracking  \nTrack popular via https://hn.algolia.com/api  \n## Sliding Window  \nTimer (Trigger): Run every 4 hours. Lookback: Fetch data from the past 24 hours.  \n- Define a rule—"Posts published less than 1 hour ago and with extremely low interaction (<3 points) are considered noise and should be discarded directly."  \n- If a post receives 20 likes in just 10 minutes (extremely explosive), we should keep it; but if it only gets 1 like after 10 minutes (normal situation), we temporarily ignore it and wait for the next scheduled task (e.g., 4 hours later) to evaluate it when it "matures." |
| Request Algolia | n8n-nodes-base.httpRequest | Fetch HN posts from Algolia API | Algolia parameters | Recalculate popularity score |  |
| Recalculate popularity score | n8n-nodes-base.code | Compute age + velocity score; discard immature posts; sort | Request Algolia | Filter |  |
| Filter | n8n-nodes-base.filter | Keep items with velocity_score > 1 | Recalculate popularity score | Translate |  |
| OpenRouter Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenRouter | LLM provider (OpenRouter Gemini) | — | Translate (AI model connection) |  |
| Translate | @n8n/n8n-nodes-langchain.agent | Translate title/story_text to Chinese; JSON-only output | Filter | Merge translation results | ### 👆 Replace with your target language  \nMust be Spanish, French, Chinese, Japanese... |
| Merge translation results | n8n-nodes-base.code | Parse AI JSON; merge into original items via $('Filter').all() | Translate | Combine message templates |  |
| Combine message templates | n8n-nodes-base.code | Build single Telegram HTML message digest | Merge translation results | Send a text message |  |
| Send a text message | n8n-nodes-base.telegram | Send digest to Telegram chat/channel | Combine message templates | — | ### 👆 Replace this with your ChatID  \n1. **Credentials**: Create a bot via @BotFather and save the Token in the credentials.  \n2. **Chat ID**:  \n   - For **Personal**: Search @userinfobot to get your ID.  \n   - For **Channel**: Add your bot as an **Administrator** to the channel. The Chat ID is usually the channel link suffix (e.g., -100xxxxxxx).  \n3. **Important**: Ensure the bot has permission to post messages! |
| Sticky Note | n8n-nodes-base.stickyNote | Comment/documentation | — | — | # Hacker News Trend Tracking  \nTrack popular via https://hn.algolia.com/api  \n## Sliding Window  \nTimer (Trigger): Run every 4 hours. Lookback: Fetch data from the past 24 hours.  \n- Define a rule—"Posts published less than 1 hour ago and with extremely low interaction (<3 points) are considered noise and should be discarded directly."  \n- If a post receives 20 likes in just 10 minutes (extremely explosive), we should keep it; but if it only gets 1 like after 10 minutes (normal situation), we temporarily ignore it and wait for the next scheduled task (e.g., 4 hours later) to evaluate it when it "matures." |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ### 👆 Replace with your target language  \nMust be Spanish, French, Chinese, Japanese... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment/documentation | — | — | ### 👆 Replace this with your ChatID  \n1. **Credentials**: Create a bot via @BotFather and save the Token in the credentials.  \n2. **Chat ID**:  \n   - For **Personal**: Search @userinfobot to get your ID.  \n   - For **Channel**: Add your bot as an **Administrator** to the channel. The Chat ID is usually the channel link suffix (e.g., -100xxxxxxx).  \n3. **Important**: Ensure the bot has permission to post messages! |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Hacker News 24H Trend to Telegram & AI Translator**
   - Set timezone as desired (this workflow uses **Etc/UTC**).

2. **Add “Schedule Trigger”**
   - Node type: **Schedule Trigger**
   - Configure rule: **Every 4 hours**
   - This is the entry node.

3. **Add “Algolia parameters” (Code node)**
   - Node type: **Code**
   - Paste logic that:
     - Computes `startTime` = now - 24h (UNIX seconds)
     - Sets `tags="(ask_hn,show_hn)"`
     - Sets `numericFilters="created_at_i>...,points>1"`
     - Sets `hitsPerPage=1000`
   - Connect: **Schedule Trigger → Algolia parameters**

4. **Add “Request Algolia” (HTTP Request)**
   - Node type: **HTTP Request**
   - Method: GET (default)
   - URL: `https://hn.algolia.com/api/v1/search_by_date` (recommended HTTPS)
   - Enable **Send Query Parameters**
   - Add query parameters:
     - `tags` = expression `{{$json.tags}}`
     - `numericFilters` = expression `{{$json.numericFilters}}`
     - `hitsPerPage` = expression `{{$json.hitsPerPage}}`
   - Connect: **Algolia parameters → Request Algolia**

5. **Add “Recalculate popularity score” (Code node)**
   - Node type: **Code**
   - Implement:
     - Read `items[0].json.hits`
     - Compute `analysis.age_hours` and `analysis.velocity_score`
     - Discard if `< 1h` and `< 5 points`
     - Sort by `velocity_score` desc
     - Return as separate items
   - Connect: **Request Algolia → Recalculate popularity score**

6. **Add “Filter”**
   - Node type: **Filter**
   - Condition: `{{$json.analysis.velocity_score}}` **greater than** `1`
   - Connect: **Recalculate popularity score → Filter**
   - Keep the node name as **Filter** unless you also update downstream references.

7. **Add OpenRouter model node**
   - Node type: **OpenRouter Chat Model** (LangChain)
   - Model: `google/gemini-2.5-flash-lite` (or your preferred model available in OpenRouter)
   - Create/Open credentials:
     - Provider: **OpenRouter**
     - API key from OpenRouter dashboard
   - No “main” connection is needed; it will connect via AI ports.

8. **Add “Translate” (AI Agent)**
   - Node type: **AI Agent**
   - Prompt type: **Define**
   - Text input:
     - `title: {{ JSON.stringify($json.title) }}`
     - `story_text: {{ JSON.stringify($json.story_text) }}`
   - System message:
     - Set target language (currently Chinese)
     - Require **JSON-only** output with keys like `title`, `story_text`
     - Strip HTML; convert `<a href>` to Markdown
   - Set **On Error** to “continue” (equivalent to `continueRegularOutput`)
   - Connect main: **Filter → Translate**
   - Connect AI model port: **OpenRouter Chat Model → Translate** (ai_languageModel connection)

9. **Add “Merge translation results” (Code node)**
   - Node type: **Code**
   - Logic:
     - `const originalItems = $('Filter').all();`
     - For each agent output:
       - read `item.json.output`
       - strip code fences
       - JSON.parse with try/catch
       - merge parsed translation object onto original item by index
   - Connect: **Translate → Merge translation results**
   - If you renamed the Filter node, update `$('Filter')` accordingly.

10. **Add “Combine message templates” (Code node)**
    - Node type: **Code**
    - Build a single Telegram HTML message:
      - header + count
      - per item: linked title, metrics line, age, optional blockquote preview
      - HTML escaping + truncation (e.g., 120 chars)
    - Output: one item with `json.text`
    - Connect: **Merge translation results → Combine message templates**

11. **Add “Send a text message” (Telegram node)**
    - Node type: **Telegram**
    - Operation: **Send Message** (text)
    - Credentials:
      - Create bot via `@BotFather`
      - Add token in n8n Telegram credentials
    - Chat ID:
      - Set to your target chat/channel ID (e.g., `-100...` for channels)
    - Text: expression `{{$json.text}}`
    - Additional field: `parse_mode = HTML`
    - Connect: **Combine message templates → Send a text message**

12. **(Optional) Add Sticky Notes**
    - Add notes near blocks for maintenance: Algolia windowing rules, translation language, Telegram chat ID instructions.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Track popular via Algolia HN API | https://hn.algolia.com/api |
| Sliding window design: run every 4 hours, look back 24 hours; discard immature low-engagement posts | From workflow sticky note (trend tracking rationale) |
| Replace translation target language in the agent system message | Change “Translate … into Chinese” to Spanish/French/Japanese/etc. |
| Telegram setup: create bot via @BotFather; get Chat ID via @userinfobot (personal) or channel id suffix (-100…) and add bot as admin | From workflow sticky note (Telegram configuration) |

Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.