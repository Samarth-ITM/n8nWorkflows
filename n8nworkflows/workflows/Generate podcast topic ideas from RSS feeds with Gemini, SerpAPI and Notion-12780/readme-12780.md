Generate podcast topic ideas from RSS feeds with Gemini, SerpAPI and Notion

https://n8nworkflows.xyz/workflows/generate-podcast-topic-ideas-from-rss-feeds-with-gemini--serpapi-and-notion-12780


# Generate podcast topic ideas from RSS feeds with Gemini, SerpAPI and Notion

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs daily to collect fresh items from a set of RSS feeds listed in Google Sheets, extracts only content published in the last 24 hours, asks **Google Gemini** (with **SerpAPI** tool access) to generate **5 ranked podcast episode ideas** (title, hook, talking points, thumbnail idea, etc.), stores each idea as a row/page in a **Notion database**, and then sends a **Telegram** notification with a link to the database view.

**Target use cases:**
- Daily “content research → show planning” automation for a podcast/YouTube channel
- Trend monitoring across multiple sources (YouTube RSS / blogs) and producing episode concepts
- Centralizing ideas in Notion and notifying a team via Telegram

### 1.1 Input Reception & Scheduling
Runs every 24 hours and loads the list of RSS feed URLs from Google Sheets.

### 1.2 Per-Feed RSS Scraping & Freshness Filtering
Iterates through each sheet row (each feed), reads RSS items, and keeps only items from the last 24 hours (excluding YouTube Shorts).

### 1.3 Headline Aggregation
Collects titles from filtered items into one combined list for AI analysis.

### 1.4 AI Topic Ideation (Gemini + SerpAPI) + Structured Output
Gemini receives the headline list, can call SerpAPI for external context, and returns ideas in a fixed JSON schema.

### 1.5 Notion Storage + Telegram Notification
Splits the JSON array of ideas into individual items, writes each as a Notion database page, then sends a Telegram message once the process completes.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Scheduling
**Overview:** Triggers daily execution and pulls the RSS feed list from Google Sheets.

**Nodes involved:**
- Execute every 24H
- Get list of RSS channels/sites
- Loop Over Rows In Sheet

#### Node: Execute every 24H
- **Type / role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) — entry point.
- **Configuration choices:** Runs daily with `triggerAtHour: 12` (server/project timezone in n8n settings).
- **Outputs:** Connects to **Get list of RSS channels/sites**.
- **Edge cases / failures:**
  - Timezone mismatch can cause unexpected run times.
  - If n8n instance is down at trigger time, execution may be missed depending on n8n setup.

#### Node: Get list of RSS channels/sites
- **Type / role:** Google Sheets (`n8n-nodes-base.googleSheets`) — reads rows containing RSS feed URLs.
- **Configuration choices (interpreted):**
  - Document selected by **URL** (Google Sheets link).
  - Sheet name: **Sheet1**.
  - Uses OAuth2 credentials (“Google Sheets account 2”).
  - `retryOnFail: true`, waits 2s between tries.
- **Expected input:** Trigger signal only.
- **Expected output:** An item per row, including a field named **`feed_link`** (required by downstream RSS node).
- **Edge cases / failures:**
  - OAuth scope/consent issues, expired refresh token.
  - Sheet not shared with the OAuth account.
  - Column name mismatch (if `feed_link` doesn’t exist, RSS read will fail).

#### Node: Loop Over Rows In Sheet
- **Type / role:** Split In Batches (`n8n-nodes-base.splitInBatches`) — batching/loop mechanism over sheet rows.
- **Configuration choices:** Default batch settings (not explicitly set).
- **Connections:**
  - **Input:** Get list of RSS channels/sites
  - **Outputs:**
    - Main output to **Filter for latest content (Last 24H)** (unusual; see note below)
    - Main output to **Scrape RSS for each link**
- **Important behavior note:**  
  In this workflow, **Loop Over Rows** is used as a loop driver where the RSS node feeds back into it (see Block 2). This pattern can work, but the presence of *two outgoing connections* (to Filter and RSS) is atypical and can be confusing—execution order and item shape should be validated.
- **Edge cases / failures:**
  - If batching isn’t configured, large sheets can produce long executions.
  - If loop termination isn’t properly handled, can lead to incomplete processing.

---

### Block 2 — Per-Feed RSS Scraping & Freshness Filtering
**Overview:** Reads each RSS feed URL, then filters the resulting items to keep only the last 24 hours and excludes “/shorts” links.

**Nodes involved:**
- Scrape RSS for each link
- Filter for latest content (Last 24H)

#### Node: Scrape RSS for each link
- **Type / role:** RSS Feed Read (`n8n-nodes-base.rssFeedRead`) — fetches RSS items for the current feed.
- **Configuration choices:**
  - URL is dynamic: `={{ $json.feed_link }}`
  - `onError: continueErrorOutput` + `alwaysOutputData: true`
  - Retries enabled (maxTries 2) with 5s wait
- **Connections:**
  - **Input:** Loop Over Rows In Sheet
  - **Output:** Feeds back to **Loop Over Rows In Sheet** (loop continuation pattern)
- **Edge cases / failures:**
  - Invalid RSS URL, timeouts, 4xx/5xx.
  - RSS feeds with non-standard fields; downstream filter expects `published` or `pubDate`.
  - Because `continueErrorOutput` is used, failed reads may still emit items, potentially with error metadata that can break filtering unless handled.

#### Node: Filter for latest content (Last 24H)
- **Type / role:** Filter (`n8n-nodes-base.filter`) — keeps only relevant/new items.
- **Configuration choices (interpreted):**
  - Condition 1 (date): `published` if present else `pubDate` must be **after** `now - 24 hours`
    - Left: `={{ $json.published ? $json.published : $json.pubDate }}`
    - Right: `={{ $now.minus(24, 'hours') }}`
  - Condition 2 (string): item `link` **does not contain** `/shorts`
    - Left: `={{ $json.link }}`
    - Right: `/shorts`
- **Connections:**
  - **Input:** Loop Over Rows In Sheet (per current wiring)
  - **Output:** Aggregate Results
- **Edge cases / failures:**
  - Date parsing issues if feed provides an unexpected format; “strict” validation may drop items.
  - Missing `link` can cause expression errors (or evaluate to empty).
  - The filter is connected from the batch node rather than directly from RSS read; ensure the items arriving here are the RSS items (not the sheet rows). If not, the date/link expressions will fail.

---

### Block 3 — Headline Aggregation
**Overview:** Aggregates filtered RSS item titles into one list (`headline_list`) used as the AI prompt input.

**Nodes involved:**
- Aggregate Results

#### Node: Aggregate Results
- **Type / role:** Aggregate (`n8n-nodes-base.aggregate`) — compiles titles into a merged list.
- **Configuration choices:**
  - Aggregates field `title`
  - Renames aggregated output field to `headline_list`
  - `mergeLists: true` to combine results into a single list output context
- **Connections:**
  - **Input:** Filter for latest content (Last 24H)
  - **Output:** AI Agent Prompt For Content Research
- **Edge cases / failures:**
  - If no items pass the filter, `headline_list` may be empty; the AI prompt may become weak/invalid.
  - If RSS items use different field names (no `title`), aggregation returns empty list.

---

### Block 4 — AI Topic Ideation (Gemini + SerpAPI) + Structured Output
**Overview:** Sends the aggregated headlines to an AI agent using Gemini as the model and SerpAPI as a tool, requesting 5 ranked ideas in a strict JSON structure.

**Nodes involved:**
- Google Gemini Chat Model
- SerpAPI
- Structured Output Parser - fixed JSON format
- AI Agent Prompt For Content Research

#### Node: Google Gemini Chat Model
- **Type / role:** LangChain Chat Model for Google Gemini (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — the LLM backend.
- **Configuration choices:**
  - Model: `models/gemini-2.5-pro`
  - Uses Google PaLM/Gemini API credentials
- **Connections:**
  - Provides `ai_languageModel` connection into **AI Agent Prompt For Content Research**
- **Edge cases / failures:**
  - Model availability / API quota.
  - Safety settings or content policy blocks (less likely here, but possible depending on prompt/data).
  - Credential misconfiguration (wrong API key/project).

#### Node: SerpAPI
- **Type / role:** LangChain tool (`@n8n/n8n-nodes-langchain.toolSerpApi`) — enables web search enrichment.
- **Configuration choices:**
  - Device: desktop
  - Google domain: `google.co.in` (India-oriented results)
  - `no_cache` is set via an AI-generated expression override; if it resolves to empty/invalid boolean, the tool could behave unexpectedly.
- **Connections:**
  - Provides `ai_tool` connection into **AI Agent Prompt For Content Research**
- **Edge cases / failures:**
  - SerpAPI quota/limits, invalid API key.
  - Regional domain differences affecting results consistency.

#### Node: Structured Output Parser - fixed JSON format
- **Type / role:** Structured output parser (`@n8n/n8n-nodes-langchain.outputParserStructured`) — enforces JSON schema.
- **Configuration choices:**
  - Schema example includes an object with `"ideas": [ ... ]`
  - Each idea includes: `podcast_idea_title`, `topical_or_contrarian`, `hook`, `channels_talking`, `talking_points`, `thumbnail_idea`, `rank` (number)
- **Connections:**
  - Connected as `ai_outputParser` into **AI Agent Prompt For Content Research**
- **Edge cases / failures:**
  - Model output not matching schema → parser errors.
  - The agent may return markdown/table instead of JSON if prompt compliance fails; parser will reject.

#### Node: AI Agent Prompt For Content Research
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — orchestrates LLM + tools + parsing.
- **Configuration choices (interpreted):**
  - Prompt injects `headline_list` into a bullet-like string:  
    `{{ $json.headline_list.join('\n- ') }}`
  - Requests: analyze “latest 2026 crypto headlines”, use search tools to find “Alpha” angles, output 5 ranked ideas, and “output content ideas as a JSON file”.
  - System message positions the agent as a crypto content expert for a specific YouTube podcast and mentions audience split (33% India).
  - `promptType: define`, `hasOutputParser: true` with the structured parser connected.
- **Connections:**
  - **Input:** Aggregate Results
  - **Output:** Split Out
  - **AI connections:** Receives Gemini model + SerpAPI tool + output parser
- **Edge cases / failures:**
  - If `headline_list` is empty, the prompt becomes low-signal; can yield generic ideas.
  - Tool call failures (SerpAPI) may degrade output or fail execution if not handled.
  - Prompt requests a “table” but also “JSON file”; conflicting instructions can reduce parser compliance—parser mitigates but failures still possible.

---

### Block 5 — Notion Storage + Telegram Notification
**Overview:** Splits the AI JSON array into individual idea items, creates a Notion database page for each, then sends a Telegram notification.

**Nodes involved:**
- Split Out
- Create a database page
- Send a text message

#### Node: Split Out
- **Type / role:** Split Out (`n8n-nodes-base.splitOut`) — turns an array into individual items.
- **Configuration choices:**
  - Field to split: `output.ideas`
  - Includes all other fields
  - `alwaysOutputData: true`
- **Connections:**
  - **Input:** AI Agent Prompt For Content Research
  - **Output:** Create a database page
- **Edge cases / failures:**
  - If the agent output is not in `output.ideas` path (e.g., parser output differs), split fails or outputs nothing.
  - Empty ideas array → no Notion pages created.

#### Node: Create a database page
- **Type / role:** Notion (`n8n-nodes-base.notion`) — creates one database page per idea.
- **Configuration choices (interpreted):**
  - Resource: Database Page
  - Database ID: `2e5ae570623380ef8f32d826eec3f589`
  - Maps Notion properties from `{{$json["output.ideas"].…}}`
  - Sets `Timestamp` to `{{$now.toISO()}}`
  - Retries enabled, wait 3s, `maxTries: 2`
- **Property mapping used:**
  - `podcast_idea_title|title` ← `output.ideas.podcast_idea_title`
  - `topical_or_contrarian|rich_text` ← `output.ideas.topical_or_contrarian`
  - `hook|rich_text` ← `output.ideas.hook`
  - `channels_talking|rich_text` ← `output.ideas.channels_talking`
  - `talking_points|rich_text` ← `output.ideas.talking_points`
  - `thumbnail_idea|rich_text` ← `output.ideas.thumbnail_idea`
  - `rank|number` ← `output.ideas.rank`
  - `Timestamp|date` ← current time
  - There is also an extra mapping key shown as `=rank|number` in addition to `rank|number` (likely a duplication/typo). This may fail or be ignored depending on Notion node behavior.
- **Connections:**
  - **Input:** Split Out
  - **Output:** Send a text message
- **Edge cases / failures:**
  - Notion property names/types must match exactly; mismatches cause runtime errors.
  - Duplicate/invalid property key (`=rank|number`) could cause an API error depending on how the node validates fields.
  - Rate limits if many pages are created at once.

#### Node: Send a text message
- **Type / role:** Telegram (`n8n-nodes-base.telegram`) — sends completion notification.
- **Configuration choices:**
  - Chat ID: `123456789`
  - Text includes timestamp and a Notion database view link:
    - `{{$now.toFormat('DD, HH:mm')}} - Daily Podcast Content Research Completed: https://www.notion.so/2e5ae570623380ef8f32d826eec3f589?v=2e5ae5706233808089e6000cb8dd99f4`
  - `appendAttribution: false`
  - `executeOnce: true` (Telegram node will run only once per execution even if upstream returns multiple items)
- **Connections:**
  - **Input:** Create a database page
  - **Output:** None
- **Edge cases / failures:**
  - If Notion page creation partially fails, Telegram may still send (depending on error handling upstream).
  - Bot permissions / chat ID mismatch.
  - With `executeOnce: true`, you get one message even when multiple ideas are created (intended “completion ping”).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Execute every 24H | Schedule Trigger | Daily workflow trigger | — | Get list of RSS channels/sites | ## How it works… Video walkthrough: https://www.youtube.com/watch?v=IoHayi68Ckk  / Setup steps (Google Sheets, Credentials, Notion DB, Adjustments) |
| Get list of RSS channels/sites | Google Sheets | Load RSS feed URLs from a sheet | Execute every 24H | Loop Over Rows In Sheet | ## 1. Gather & Filter Latest Content… |
| Loop Over Rows In Sheet | Split In Batches | Iterate over sheet rows / drive loop | Get list of RSS channels/sites; Scrape RSS for each link | Filter for latest content (Last 24H); Scrape RSS for each link | ## 1. Gather & Filter Latest Content… |
| Scrape RSS for each link | RSS Feed Read | Fetch RSS items per feed URL | Loop Over Rows In Sheet | Loop Over Rows In Sheet | ## 1. Gather & Filter Latest Content… |
| Filter for latest content (Last 24H) | Filter | Keep items published in last 24h and exclude shorts | Loop Over Rows In Sheet | Aggregate Results | ## 1. Gather & Filter Latest Content… |
| Aggregate Results | Aggregate | Build a list of headlines from filtered items | Filter for latest content (Last 24H) | AI Agent Prompt For Content Research | ## 2. Generate Topics with AI… |
| Google Gemini Chat Model | LangChain Chat Model (Gemini) | LLM used by the agent | — (AI connection) | AI Agent Prompt For Content Research | ## 2. Generate Topics with AI… |
| SerpAPI | LangChain Tool | Web search tool for agent enrichment | — (AI connection) | AI Agent Prompt For Content Research | ## 2. Generate Topics with AI… |
| Structured Output Parser - fixed JSON format | LangChain Output Parser | Forces AI output into defined JSON schema | — (AI connection) | AI Agent Prompt For Content Research | ## 2. Generate Topics with AI… |
| AI Agent Prompt For Content Research | LangChain Agent | Generate ranked podcast ideas using headlines + tools | Aggregate Results | Split Out | ## 2. Generate Topics with AI… |
| Split Out | Split Out | Split JSON `ideas` array into items | AI Agent Prompt For Content Research | Create a database page | ## 3. Store Ideas & Notify… |
| Create a database page | Notion | Create Notion DB page per idea | Split Out | Send a text message | ## 3. Store Ideas & Notify… |
| Send a text message | Telegram | Send completion notification with Notion link | Create a database page | — | ## 3. Store Ideas & Notify… |
| Sticky Note | Sticky Note | Documentation / setup instructions | — | — | (Contains: How it works, setup steps, and video link https://www.youtube.com/watch?v=IoHayi68Ckk) |
| Sticky Note2 | Sticky Note | Documentation for Block 1 | — | — | ## 1. Gather & Filter Latest Content… |
| Sticky Note3 | Sticky Note | Documentation for Block 2 | — | — | ## 2. Generate Topics with AI… |
| Sticky Note4 | Sticky Note | Documentation for Block 3 | — | — | ## 3. Store Ideas & Notify… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   “Generate Podcast Topic Ideas from RSS Feeds using Google Gemini AI, SerAPI, and Notion”.

2. **Add node: Schedule Trigger**
   - Type: **Schedule Trigger**
   - Configure: run **daily**, set **Trigger at hour = 12** (adjust to your timezone needs).
   - This is the workflow entry point.

3. **Add node: Google Sheets (read feed list)**
   - Type: **Google Sheets**
   - Operation: read/get rows (default “Read” behavior in n8n’s Sheets node).
   - Set **Document**: select by URL and paste your sheet URL.
   - Set **Sheet name**: `Sheet1` (or your sheet tab).
   - Ensure your sheet has a column named **`feed_link`** with RSS URLs.
   - Credentials: configure **Google Sheets OAuth2** credential and select it in the node.
   - Connect: **Schedule Trigger → Google Sheets**.

4. **Add node: Split In Batches (loop rows)**
   - Type: **Split In Batches**
   - Keep default options (or set a batch size if you have many feeds).
   - Connect: **Google Sheets → Split In Batches**.

5. **Add node: RSS Feed Read**
   - Type: **RSS Feed Read**
   - URL: expression `{{$json.feed_link}}`
   - Enable retry (optional) and set error handling:
     - Consider enabling “Continue on Fail” style behavior if you want resilience per-feed.
   - Connect: **Split In Batches → RSS Feed Read**.
   - Create loop-back connection: **RSS Feed Read → Split In Batches** (to proceed to next batch item).  
     (In n8n, this is a common looping pattern.)

6. **Add node: Filter (last 24 hours + exclude shorts)**
   - Type: **Filter**
   - Add condition (DateTime → After):
     - Left: `{{$json.published ? $json.published : $json.pubDate}}`
     - Right: `{{$now.minus(24, 'hours')}}`
   - Add condition (String → Not contains):
     - Left: `{{$json.link}}`
     - Right: `/shorts`
   - Connect the data stream containing RSS items into this Filter.  
     (Recommended: **RSS Feed Read → Filter**. If you keep the original wiring, verify the filter actually receives RSS items.)

7. **Add node: Aggregate**
   - Type: **Aggregate**
   - Aggregate field: `title`
   - Output field name: `headline_list`
   - Enable merging lists so you end with a single combined list for the AI prompt.
   - Connect: **Filter → Aggregate**.

8. **Add node: Google Gemini Chat Model**
   - Type: **Google Gemini Chat Model** (LangChain)
   - Model: `models/gemini-2.5-pro`
   - Credentials: configure **Google Gemini (PaLM) API** credential (API key / project as required by your n8n version).

9. **Add node: SerpAPI tool**
   - Type: **SerpAPI Tool** (LangChain)
   - Set:
     - Device: desktop
     - Google domain: `google.co.in` (optional; adjust to your audience)
   - Credentials: configure **SerpAPI** credential with API key.

10. **Add node: Structured Output Parser**
    - Type: **Structured Output Parser** (LangChain)
    - Paste schema example (conceptually):
      - Object with key `ideas` as an array of objects containing:
        `podcast_idea_title`, `topical_or_contrarian`, `hook`, `channels_talking`, `talking_points`, `thumbnail_idea`, `rank` (number).

11. **Add node: AI Agent**
    - Type: **AI Agent** (LangChain)
    - Prompt: include the aggregated headlines via expression:
      - `Analyze ... headlines ... in {{ $json.headline_list.join('\n- ') }} ... Generate 5 ideas ... output as JSON ...`
    - System message: set the crypto-podcast expert context and India audience note.
    - Enable structured output / output parser usage.
    - Connect AI resources:
      - **Gemini Chat Model → Agent** via the node’s **AI Language Model** connector.
      - **SerpAPI Tool → Agent** via **AI Tool** connector.
      - **Structured Output Parser → Agent** via **AI Output Parser** connector.
    - Connect: **Aggregate → AI Agent**.

12. **Add node: Split Out**
    - Type: **Split Out**
    - Field to split out: `output.ideas`
    - Include: “all other fields”
    - Connect: **AI Agent → Split Out**.

13. **Add node: Notion (Create Database Page)**
    - Type: **Notion**
    - Resource: **Database Page**
    - Database: select or paste your **Database ID**
    - Map properties to the idea fields:
      - Title property ← `{{$json["output.ideas"].podcast_idea_title}}`
      - Rich text fields ← corresponding JSON fields
      - Number field `rank` ← `{{$json["output.ideas"].rank}}`
      - Date field Timestamp ← `{{$now.toISO()}}`
    - Credentials: configure **Notion API** credential and ensure the integration has access to the database.
    - Connect: **Split Out → Notion**.
    - Note: avoid duplicate/invalid property keys (the provided workflow shows both `rank|number` and `=rank|number`; keep only the valid one).

14. **Add node: Telegram (Send Message)**
    - Type: **Telegram**
    - Operation: send text message
    - Chat ID: your target chat/channel ID
    - Message text: include timestamp and Notion database link.
    - Set **Execute Once** = true (so you only notify once even after multiple Notion creates).
    - Credentials: configure **Telegram Bot API** credential.
    - Connect: **Notion → Telegram**.

15. **Activate the workflow** and run a manual execution once to validate:
    - Sheet column name `feed_link`
    - RSS node returns `title`, `link`, `published/pubDate`
    - Filter passes items
    - Agent output matches schema
    - Notion properties match types

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically fetches fresh content from specified RSS feeds daily… stores these insights in Notion and sends a Telegram notification.” | Sticky note “How it works” |
| Video walkthrough | https://www.youtube.com/watch?v=IoHayi68Ckk |
| Setup steps include: create Google Sheet with RSS URLs, configure credentials (Sheets, Gemini, SerpAPI, Notion, Telegram), create Notion DB with matching properties, update node pointers/IDs. | Sticky note “Setup steps” |
| Block notes: “1. Gather & Filter Latest Content” / “2. Generate Topics with AI” / “3. Store Ideas & Notify” | Sticky Notes 2–4 |

