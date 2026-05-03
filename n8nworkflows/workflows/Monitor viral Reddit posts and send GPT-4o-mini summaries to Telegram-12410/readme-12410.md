Monitor viral Reddit posts and send GPT-4o-mini summaries to Telegram

https://n8nworkflows.xyz/workflows/monitor-viral-reddit-posts-and-send-gpt-4o-mini-summaries-to-telegram-12410


# Monitor viral Reddit posts and send GPT-4o-mini summaries to Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow runs every day at **8:00 AM**, checks multiple Reddit niches (subreddits) for **potentially viral posts**, uses **GPT-4o-mini** to generate **100–200 word Telegram-formatted summaries**, and sends them to a configured **Telegram chat**.

**Target use cases:**
- Daily monitoring of trending/viral Reddit content across multiple topics
- Automated content briefings for communities, teams, or personal digest channels
- Lightweight “viral radar” for tech/science/gaming/programming communities

### 1.1 Scheduling & Initialization
Runs daily and loads configuration (subreddit list + Telegram destination).

### 1.2 Niche Iteration & Reddit Fetching
Splits the configured niche list into single items and loops through them, fetching the newest posts from each subreddit.

### 1.3 Post Normalization & Viral Filtering
Extracts key post fields, then filters posts using engagement and recency logic.

### 1.4 Aggregation & AI Summarization → Telegram Delivery
Aggregates filtered posts and generates summaries via an AI agent using an OpenAI chat model, then sends results to Telegram.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Configuration

**Overview:**  
Triggers the workflow daily and defines the niche subreddits plus the Telegram chat ID, then prepares the list for iteration.

**Nodes involved:**
- Daily 8 AM Trigger
- Workflow Configuration
- Split Out

#### Node: Daily 8 AM Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point on a time schedule.
- **Configuration choices:** Runs at **8 AM** (daily).
- **Inputs:** None (trigger node).
- **Outputs:** Sends a single empty-ish item into **Workflow Configuration**.
- **Version notes:** v1.3 — schedule rule UI differs slightly by n8n versions.
- **Failure/edge cases:** Instance timezone matters; 8 AM is according to the n8n server timezone unless configured otherwise.

#### Node: Workflow Configuration
- **Type / role:** `Set` — defines runtime constants.
- **Configuration choices:**
  - `niches` (array): `["technology", "programming", "science", "gaming"]`
  - `telegramChatId` (string): placeholder `{{TELEGRAM_CHAT_ID_PLACEHOLDER}}`
  - `includeOtherFields: true` (keeps incoming fields, though typically none are needed here)
- **Key variables/expressions:** none beyond static values.
- **Input:** from **Daily 8 AM Trigger**
- **Output:** to **Split Out**
- **Failure/edge cases:** If `telegramChatId` is missing/invalid, Telegram node will fail later.

#### Node: Split Out
- **Type / role:** `Split Out` — converts an array field into one item per element.
- **Configuration choices:** `fieldToSplitOut = niches`
- **Input:** from **Workflow Configuration** (with `niches` array)
- **Output:** to **Loop Over Niches**
- **Failure/edge cases:** If `niches` isn’t an array (e.g., set as string), this node can error or output nothing.

**Sticky notes applying to this block:**
- **Sticky Note (How it works + Setup steps)**
- **Sticky Note1 (Configuration)**

---

### Block 2 — Niche Iteration & Reddit Data Fetching

**Overview:**  
Loops over each niche/subreddit and pulls the latest 50 posts from Reddit’s “new” category, using OAuth2 credentials.

**Nodes involved:**
- Loop Over Niches
- Get Reddit Viral Posts

#### Node: Loop Over Niches
- **Type / role:** `Split In Batches` — used here as an iteration/loop controller.
- **Configuration choices:** Default options (batch size not explicitly set in JSON shown).
- **Inputs:** from **Split Out** (one item per niche)
- **Outputs:**
  - Output 1 → **Extract Post Data**
  - Output 2 → **Get Reddit Viral Posts**
- **Important behavior note:** In classic looping patterns, `Split In Batches` feeds items forward and the loop is closed by connecting the last node back into `Split In Batches`. Here, **Get Reddit Viral Posts** is connected back into **Loop Over Niches**, forming a loop.
- **Failure/edge cases:**
  - Misconfigured batch size can cause unexpected iteration behavior.
  - If the loop is not balanced correctly, it may stop early or iterate incorrectly.

#### Node: Get Reddit Viral Posts
- **Type / role:** `Reddit` — fetches posts from a subreddit.
- **Configuration choices:**
  - Operation: `getAll`
  - Category filter: `new`
  - Limit: `50`
  - Subreddit: `={{ $json.niches }}`
    - Because of **Split Out**, `$json.niches` is a single subreddit name (e.g., `"technology"`).
- **Credentials:** Reddit OAuth2 (`Reddit account`)
- **Input:** from **Loop Over Niches**
- **Output:** back to **Loop Over Niches** (loop continuation)
- **Failure/edge cases:**
  - OAuth token expiration / missing scopes
  - Reddit rate limiting or transient API errors
  - Subreddit name invalid or banned/private (may return errors/empty)

**Sticky notes applying to this block:**
- **Sticky Note3 (Reddit Data Fetching)**

---

### Block 3 — Post Normalization & Viral Filtering

**Overview:**  
Extracts relevant fields from each Reddit post item, computes a formatted date, then filters for “viral” content using upvote and recency thresholds.

**Nodes involved:**
- Extract Post Data
- Filter
- Aggregate

#### Node: Extract Post Data
- **Type / role:** `Set` — normalizes Reddit post fields for downstream logic.
- **Configuration choices (fields created):**
  - `title` = `{{$json.title}}`
  - `postLink` = `{{$json.url}}`
  - `upvotes` = `{{$json.ups}}`
  - `createdAt` = `{{ new Date($json.created_utc * 1000).toLocaleDateString('en-GB').replace(/\//g, '-') }}`
    - Produces a **string date** like `dd-mm-yyyy`
  - `subreddit` = `{{$json.subreddit}}`
- **Options:** `ignoreConversionErrors: true` (reduces hard failures when type conversions fail)
- **Input:** from **Loop Over Niches**
- **Output:** to **Filter**
- **Failure/edge cases:**
  - Reddit node output shape can differ; if fields like `created_utc` are missing, `createdAt` becomes invalid.
  - `createdAt` is **not a timestamp**; it’s a formatted date string, which can lead to recency logic inaccuracies later.

#### Node: Filter
- **Type / role:** `Filter` — keeps only posts meeting engagement criteria.
- **Configuration choices:** Two conditions combined with **AND** (important):
  1) `upvotes > 500`
  2) A boolean expression that checks:
     - `$json.upvotes > 70` AND
     - post date within last 24 hours, computed by parsing `createdAt`
- **Key expressions:**
  - Condition 1:
    - Left: `={{ $json.upvotes }}`
    - Op: `gt`
    - Right: `500`
  - Condition 2 (boolean “true” check):
    ```js
    {{
      $json.upvotes > 70 &&
      (
        Date.now() -
        new Date(
          $json.createdAt.split('-').reverse().join('-')
        ).getTime()
      ) <= (1 * 24 * 60 * 60 * 1000)
    }}
    ```
- **Input:** from **Extract Post Data**
- **Output:** to **Aggregate**
- **Critical logic note (likely unintended):**
  - The filter uses **AND**, meaning a post must satisfy **both**:
    - `upvotes > 500`
    - AND also `upvotes > 70 within 24h`
  - However, the sticky note describes: “**500+ upvotes OR 70+ upvotes within 24 hours**”.
  - To match the description, the combinator should be **OR**, or the conditions should be refactored.
- **Edge cases / failure types:**
  - The recency check uses only a date (no time), which can misclassify posts around day boundaries.
  - If `createdAt` is malformed, `new Date(...)` becomes invalid → `getTime()` becomes `NaN` → condition becomes false.

#### Node: Aggregate
- **Type / role:** `Aggregate` — combines item data for batch processing.
- **Configuration choices:** `aggregateAllItemData` (collects all incoming items into one aggregated structure).
- **Input:** from **Filter**
- **Output:** to **AI Summarizer**
- **Failure/edge cases:**
  - If filter outputs zero items, aggregation may output nothing; downstream AI/Telegram nodes might not run.
  - Large aggregates could exceed token limits depending on what is passed to the AI node.

**Sticky notes applying to this block:**
- **Sticky Note2 (Quality Filter)**

---

### Block 4 — AI Summarization & Telegram Delivery

**Overview:**  
Uses a LangChain Agent node powered by OpenAI’s GPT-4o-mini to produce Telegram-ready summaries, then sends them to the configured chat.

**Nodes involved:**
- OpenAI Chat Model
- AI Summarizer
- Send to Telegram

#### Node: OpenAI Chat Model
- **Type / role:** `OpenAI Chat Model` (LangChain) — provides the LLM backend.
- **Configuration choices:** Model set to `gpt-4o-mini`
- **Credentials:** OpenAI API (`n8n free OpenAI API credits`)
- **Connections:** Outputs via `ai_languageModel` to **AI Summarizer**
- **Failure/edge cases:**
  - Invalid API key, quota exhaustion, model not available in region/account
  - Network timeouts or OpenAI transient errors

#### Node: AI Summarizer
- **Type / role:** `LangChain Agent` — orchestrates prompt + model to generate summaries.
- **Configuration choices:**
  - **Prompt input (`text`):**
    ```
    Reddit Post Data:

    {{ $json.data.toJsonString() }}
    ```
    This implies the incoming item has a `data` property that can be converted to JSON string.
  - **System message:** instructs the agent to:
    - analyze post data (title/content/upvotes/metadata)
    - explain why viral
    - produce 100–200 word summary
    - “output in telegram formatted text”
  - `promptType: define`
- **Inputs:**
  - Main data from **Aggregate**
  - LLM from **OpenAI Chat Model** connection (`ai_languageModel`)
- **Outputs:** to **Send to Telegram**; expected field: `$json.output`
- **Important mismatch risk:**
  - Depending on how **Aggregate** structures output, `$json.data` may not exist. Some aggregate modes output fields like `aggregatedData` or arrays under another key. If `$json.data` is missing, the expression fails or produces empty prompt content.
- **Failure/edge cases:**
  - Prompt too large (token limit) if aggregated data includes many posts
  - Output formatting: Telegram Markdown can break if the model returns unescaped special characters

#### Node: Send to Telegram
- **Type / role:** `Telegram` — sends the generated summary to a chat.
- **Configuration choices:**
  - Text: `={{ $json.output }}`
  - Chat ID: `={{ $('Workflow Configuration').item.json.telegramChatId }}`
  - Parse mode: `Markdown`
- **Credentials:** Telegram bot API (`Telegram account`)
- **Input:** from **AI Summarizer**
- **Output:** none (terminal action)
- **Failure/edge cases:**
  - Invalid chat ID, bot not allowed in chat/channel
  - Markdown parse errors (Telegram can reject malformed Markdown)
  - Rate limits if many messages are sent quickly

**Sticky notes applying to this block:**
- **Sticky Note4 (Summary Generation)**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily 8 AM Trigger | Schedule Trigger | Time-based entry point | — | Workflow Configuration | ## How it works\n\nThis workflow monitors Reddit for viral posts in your chosen niches and sends AI-generated summaries to Telegram. It runs daily at 8 AM, fetching the latest posts from specified subreddits, filtering for high engagement (500+ upvotes OR 70+ upvotes within 24 hours), and creating concise summaries using GPT-4o-mini.\n\n## Setup steps\n\n1. Configure your niches in the "Workflow Configuration" node (currently: technology, programming, science, gaming)\n2. Add your Telegram chat ID in the same node\n3. Connect your Reddit OAuth2 credentials\n4. Connect your Telegram bot credentials\n5. Adjust the schedule trigger if needed (default: daily at 8 AM)\n6. Test the workflow manually before activating |
| Workflow Configuration | Set | Define niches + Telegram destination | Daily 8 AM Trigger | Split Out | ## Configuration\nSets up niches to monitor and Telegram destination, then splits into individual subreddit checks |
| Split Out | Split Out | Split niches array into items | Workflow Configuration | Loop Over Niches | ## Configuration\nSets up niches to monitor and Telegram destination, then splits into individual subreddit checks |
| Loop Over Niches | Split In Batches | Iteration/loop controller | Split Out, Get Reddit Viral Posts | Extract Post Data, Get Reddit Viral Posts | ## Reddit Data Fetching\nLoops through each niche, fetches the 50 newest posts, and extracts relevant metadata |
| Get Reddit Viral Posts | Reddit | Fetch newest subreddit posts | Loop Over Niches | Loop Over Niches | ## Reddit Data Fetching\nLoops through each niche, fetches the 50 newest posts, and extracts relevant metadata |
| Extract Post Data | Set | Normalize post fields | Loop Over Niches | Filter | ## Quality Filter\nKeeps only viral posts meeting engagement thresholds, then aggregates for batch processing |
| Filter | Filter | Viral/quality gating | Extract Post Data | Aggregate | ## Quality Filter\nKeeps only viral posts meeting engagement thresholds, then aggregates for batch processing |
| Aggregate | Aggregate | Combine filtered items for AI | Filter | AI Summarizer | ## Quality Filter\nKeeps only viral posts meeting engagement thresholds, then aggregates for batch processing |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Provide GPT-4o-mini model | — | AI Summarizer (ai_languageModel) | ## Summary Generation\nUses AI to create concise post summaries in Telegram-formatted text and delivers to your chat |
| AI Summarizer | LangChain Agent | Generate Telegram-formatted summaries | Aggregate; OpenAI Chat Model (ai_languageModel) | Send to Telegram | ## Summary Generation\nUses AI to create concise post summaries in Telegram-formatted text and delivers to your chat |
| Send to Telegram | Telegram | Deliver summary to Telegram chat | AI Summarizer | — | ## Summary Generation\nUses AI to create concise post summaries in Telegram-formatted text and delivers to your chat |
| Sticky Note | Sticky Note | Documentation | — | — | ## How it works\n\nThis workflow monitors Reddit for viral posts in your chosen niches and sends AI-generated summaries to Telegram. It runs daily at 8 AM, fetching the latest posts from specified subreddits, filtering for high engagement (500+ upvotes OR 70+ upvotes within 24 hours), and creating concise summaries using GPT-4o-mini.\n\n## Setup steps\n\n1. Configure your niches in the "Workflow Configuration" node (currently: technology, programming, science, gaming)\n2. Add your Telegram chat ID in the same node\n3. Connect your Reddit OAuth2 credentials\n4. Connect your Telegram bot credentials\n5. Adjust the schedule trigger if needed (default: daily at 8 AM)\n6. Test the workflow manually before activating |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Configuration\nSets up niches to monitor and Telegram destination, then splits into individual subreddit checks |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## Quality Filter\nKeeps only viral posts meeting engagement thresholds, then aggregates for batch processing |
| Sticky Note3 | Sticky Note | Documentation | — | — | ## Reddit Data Fetching\nLoops through each niche, fetches the 50 newest posts, and extracts relevant metadata |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Summary Generation\nUses AI to create concise post summaries in Telegram-formatted text and delivers to your chat |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   - Add node: **Schedule Trigger**
   - Set it to run **daily at 08:00** (server timezone).

2) **Add configuration constants**
   - Add node: **Set** named **Workflow Configuration**
   - Add fields:
     - `niches` (Array) = `["technology","programming","science","gaming"]`
     - `telegramChatId` (String) = your Telegram chat ID (or channel ID)
   - Enable **Include Other Fields** (optional).

3) **Split niches into individual items**
   - Add node: **Split Out**
   - Set **Field to split out** = `niches`
   - Connect: Trigger → Workflow Configuration → Split Out

4) **Add loop controller**
   - Add node: **Split In Batches** named **Loop Over Niches**
   - Keep default options (or set a batch size of 1 for clarity).
   - Connect: **Split Out → Loop Over Niches**

5) **Fetch Reddit posts**
   - Add node: **Reddit** named **Get Reddit Viral Posts**
   - Operation: **Get All**
   - Category filter: **new**
   - Limit: **50**
   - Subreddit field expression: `{{ $json.niches }}`
   - Configure **Reddit OAuth2** credentials.
   - Connect: **Loop Over Niches → Get Reddit Viral Posts**
   - Connect back to loop (to continue batches): **Get Reddit Viral Posts → Loop Over Niches**
     - This forms the loop pattern used in the provided workflow.

6) **Extract/normalize post fields**
   - Add node: **Set** named **Extract Post Data**
   - Add fields (expressions):
     - `title` = `{{ $json.title }}`
     - `postLink` = `{{ $json.url }}`
     - `upvotes` = `{{ $json.ups }}`
     - `createdAt` = `{{ new Date($json.created_utc * 1000).toLocaleDateString('en-GB').replace(/\//g, '-') }}`
     - `subreddit` = `{{ $json.subreddit }}`
   - Option: **Ignore conversion errors** = true
   - Connect: **Loop Over Niches → Extract Post Data**
     - (This assumes your loop node is emitting Reddit post items into this branch.)

7) **Filter for viral posts**
   - Add node: **Filter** named **Filter**
   - Add conditions (as in workflow):
     - `upvotes` **greater than** `500`
     - Boolean expression equals **true**:
       ```js
       {{
         $json.upvotes > 70 &&
         (
           Date.now() -
           new Date($json.createdAt.split('-').reverse().join('-')).getTime()
         ) <= (1 * 24 * 60 * 60 * 1000)
       }}
       ```
   - Important: If you want the behavior described in the note (“500+ OR 70+ within 24h”), set the filter combinator to **OR** instead of AND.
   - Connect: **Extract Post Data → Filter**

8) **Aggregate filtered items**
   - Add node: **Aggregate** named **Aggregate**
   - Mode: **Aggregate All Item Data**
   - Connect: **Filter → Aggregate**

9) **Add OpenAI model node**
   - Add node: **OpenAI Chat Model** (LangChain) named **OpenAI Chat Model**
   - Model: **gpt-4o-mini**
   - Configure **OpenAI API** credentials.

10) **Add AI Agent summarizer**
   - Add node: **AI Agent** (LangChain) named **AI Summarizer**
   - Prompt type: **Define**
   - Text:
     ```
     Reddit Post Data:

     {{ $json.data.toJsonString() }}
     ```
     (Adjust this expression to match whatever field your Aggregate outputs.)
   - System message: paste the workflow’s system instructions (concise summaries, 100–200 words, Telegram-formatted text).
   - Connect LLM: **OpenAI Chat Model → AI Summarizer** via the `ai_languageModel` connection.
   - Connect data: **Aggregate → AI Summarizer**

11) **Send message to Telegram**
   - Add node: **Telegram** named **Send to Telegram**
   - Chat ID expression:
     - `{{ $('Workflow Configuration').item.json.telegramChatId }}`
   - Text expression:
     - `{{ $json.output }}`
   - Additional field: `parse_mode = Markdown`
   - Configure **Telegram bot** credentials (create bot with BotFather, add token).
   - Connect: **AI Summarizer → Send to Telegram**

12) **Test & activate**
   - Run once manually to validate:
     - Reddit credentials work
     - Filter outputs items
     - AI node receives expected input structure
     - Telegram formatting is accepted
   - Activate workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow monitors Reddit for viral posts in your chosen niches and sends AI-generated summaries to Telegram. It runs daily at 8 AM, fetching the latest posts from specified subreddits, filtering for high engagement (500+ upvotes OR 70+ upvotes within 24 hours), and creating concise summaries using GPT-4o-mini.” | Sticky note “How it works” (workflow intent) |
| Setup steps: configure niches + Telegram chat ID, connect Reddit OAuth2 + Telegram bot, adjust schedule, test manually | Sticky note “Setup steps” |
| Potential logic mismatch: Filter is configured with AND but description says OR | Implementation vs documented intent (important for modification) |