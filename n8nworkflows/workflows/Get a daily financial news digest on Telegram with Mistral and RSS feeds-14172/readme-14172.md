Get a daily financial news digest on Telegram with Mistral and RSS feeds

https://n8nworkflows.xyz/workflows/get-a-daily-financial-news-digest-on-telegram-with-mistral-and-rss-feeds-14172


# Get a daily financial news digest on Telegram with Mistral and RSS feeds

# 1. Workflow Overview

This workflow generates a **daily financial news digest** and delivers it to **Telegram**. It collects articles from a curated set of finance RSS feeds, ranks and filters them, sends the selected stories to **NVIDIA NIM** using a **Mistral Large** model to produce a concise digest, then logs the outcome to **Google Sheets**.

Primary use cases:
- Daily market/news briefings for investors, operators, founders, and analysts
- Automated monitoring of multiple finance publications
- Lightweight AI-assisted summarization distributed via Telegram

## 1.1 Scheduled Start and Runtime Configuration
The workflow starts on a daily cron schedule and loads all runtime configuration values from a Set node. This includes RSS feeds, story limits, freshness window, Telegram values, Google Sheet ID, and execution run ID.

## 1.2 Feed Expansion and Iteration
The configured feed list is transformed into one item per feed, then processed in a loop so each RSS source is read individually.

## 1.3 Article Collection, Tagging, and Ranking
Each feed is fetched, normalized, tagged with source metadata, and accumulated using workflow global static data. After all feeds are processed, stories are filtered, deduplicated, scored, and trimmed to the top results.

## 1.4 Branching: Digest vs No-Stories Alert
If ranked stories are available, the workflow proceeds to AI summarization. If not, it sends a Telegram alert indicating that no fresh stories were found.

## 1.5 AI Digest Generation
The selected top stories are converted into a chat-completions payload and sent to NVIDIA NIM with a Mistral model. The response text is extracted as the digest body.

## 1.6 Telegram Delivery
The digest is wrapped in a branded Telegram message payload and posted to the Telegram Bot API.

## 1.7 Logging and Final Status
If Telegram delivery succeeds, the run is logged to the `Digest_Log` sheet and the workflow emits a success status object. If delivery fails, the error is logged to `Error_Log` and the workflow emits a failure status object.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Start and Configuration

### Overview
This block triggers the workflow once per day and defines all adjustable settings in one place. It acts as the operational control panel for feeds, Telegram, freshness logic, and Google Sheets logging.

### Nodes Involved
- `⏰ Schedule Trigger1`
- `📋 RSS Feed Config1`

### Node Details

#### 1. `⏰ Schedule Trigger1`
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point
- **Configuration choices:** Uses a cron expression `0 10 * * *`, so it runs daily at 10:00 according to the workflow/server timezone.
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to `📋 RSS Feed Config1`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases or failures:**
  - Timezone misunderstandings can cause unexpected run time
  - Workflow must be activated for schedule execution
- **Sub-workflow reference:** None

#### 2. `📋 RSS Feed Config1`
- **Type and role:** `n8n-nodes-base.set`; central configuration store
- **Configuration choices:**
  - Defines `rssFeeds` as an array of feed objects with:
    - `url`
    - `tier`
    - `source`
  - Defines:
    - `telegramBotToken`
    - `telegramChatId`
    - `maxStoriesInDigest`
    - `freshnessHours`
    - `googleSheetId`
    - `runId = {{$execution.id}}`
- **Key expressions or variables used:**
  - `={{ $execution.id }}` for `runId`
- **Input and output connections:** Input from `⏰ Schedule Trigger1`; output to `Generate Feed Items`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:**
  - Placeholder values must be replaced before production
  - Invalid RSS URLs will surface later in the feed-reading block
  - Invalid Telegram bot token/chat ID will break delivery later
  - Invalid Google Sheet ID will break logging later
- **Sub-workflow reference:** None

---

## Block 2 — Feed Expansion and Loop Control

### Overview
This block turns the feed configuration array into a stream of items and iterates through them one at a time. It provides the loop structure used by the downstream RSS reader and aggregator.

### Nodes Involved
- `Generate Feed Items`
- `Loop Over Feeds`

### Node Details

#### 3. `Generate Feed Items`
- **Type and role:** `n8n-nodes-base.code`; converts feed config array into one item per feed
- **Configuration choices:**
  - Reads the first input item
  - Extracts `rssFeeds`
  - Returns one output item per feed with:
    - `feedUrl`
    - `feedSource`
    - `feedTier`
- **Key expressions or variables used:**
  - Uses `$input.first().json`
- **Input and output connections:** Input from `📋 RSS Feed Config1`; output to `Loop Over Feeds`
- **Version-specific requirements:** Code node version `2`
- **Edge cases or failures:**
  - If `rssFeeds` is missing or not an array, code execution fails
  - Empty `rssFeeds` means nothing meaningful will be processed
- **Sub-workflow reference:** None

#### 4. `Loop Over Feeds`
- **Type and role:** `n8n-nodes-base.splitInBatches`; loop controller
- **Configuration choices:** Uses default batching behavior to process feed items incrementally
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input from `Generate Feed Items`
  - Output 1 loops to `Read RSS Feed`
  - Receives control back from `Is Processing Done?` to continue iteration
- **Version-specific requirements:** Type version `3`
- **Edge cases or failures:**
  - Miswiring this node breaks the loop
  - If upstream returns no items, loop body never executes
- **Sub-workflow reference:** None

---

## Block 3 — RSS Reading, Normalization, and Aggregation

### Overview
This block fetches each RSS feed, normalizes article fields, and accumulates all feed data until every feed has been processed. It then filters, deduplicates, and ranks the stories.

### Nodes Involved
- `Read RSS Feed`
- `Tag Articles`
- `Aggregate and Rank`
- `Is Processing Done?`

### Node Details

#### 5. `Read RSS Feed`
- **Type and role:** `n8n-nodes-base.rssFeedRead`; fetches feed entries from the current RSS URL
- **Configuration choices:**
  - URL is read dynamically from `{{$json.feedUrl}}`
  - `continueOnFail` is enabled
- **Key expressions or variables used:**
  - `={{ $json.feedUrl }}`
- **Input and output connections:** Input from `Loop Over Feeds`; output to `Tag Articles`
- **Version-specific requirements:** Type version `1.1`
- **Edge cases or failures:**
  - Feed unreachable
  - Invalid XML/RSS structure
  - Rate limits or remote server errors
  - Because `continueOnFail` is enabled, execution continues but may return incomplete/error-shaped data
- **Sub-workflow reference:** None

#### 6. `Tag Articles`
- **Type and role:** `n8n-nodes-base.code`; normalizes article structure and attaches source metadata from the current loop item
- **Configuration choices:**
  - Maps all incoming feed items into a standard schema:
    - `title`
    - `link`
    - `pubDate`
    - `description`
    - `sourceName`
    - `tier`
  - Truncates description to 400 characters
  - Pulls source and tier from `Loop Over Feeds`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$('Loop Over Feeds').item.json.feedSource`
  - `$('Loop Over Feeds').item.json.feedTier`
- **Input and output connections:** Input from `Read RSS Feed`; output to `Aggregate and Rank`
- **Version-specific requirements:** Code node version `2`
- **Edge cases or failures:**
  - If RSS output item schema varies widely, some fields may be empty
  - If `Loop Over Feeds` context is unavailable due to execution changes, metadata resolution could fail
  - Missing `title` or `link` is tolerated but later filtered out
- **Sub-workflow reference:** None

#### 7. `Aggregate and Rank`
- **Type and role:** `n8n-nodes-base.code`; stateful collector and ranking engine
- **Configuration choices:**
  - Uses `$getWorkflowStaticData('global')`
  - Resets static collection state when `runId` changes
  - Appends valid article objects from each feed
  - Tracks processed feed count
  - Returns a temporary `{ waiting: true }` result until all feeds are processed
  - After all feeds are processed:
    - Applies freshness filtering using `freshnessHours`
    - Removes sponsored/promotional titles
    - Deduplicates by link
    - Scores by:
      - source tier
      - article recency
      - finance-related title keywords
    - Applies title-based near-deduplication
    - Sorts descending by score
    - Limits to `maxStoriesInDigest`
  - Clears static collector after final processing
- **Key expressions or variables used:**
  - `$getWorkflowStaticData('global')`
  - `$('📋 RSS Feed Config1').first().json`
- **Input and output connections:** Input from `Tag Articles`; output to `Is Processing Done?`
- **Version-specific requirements:** Code node version `2`
- **Edge cases or failures:**
  - Static data persistence means concurrent executions could interfere if overlapping runs occur
  - Invalid `pubDate` values may parse to `NaN`; in this code they effectively bypass freshness filtering if parsing fails
  - Empty feed outputs are allowed
  - If many feeds fail silently, top story selection may be poor or empty
- **Sub-workflow reference:** None

#### 8. `Is Processing Done?`
- **Type and role:** `n8n-nodes-base.if`; loop completion gate
- **Configuration choices:**
  - Checks whether `waiting != true`
  - If false, returns control to `Loop Over Feeds`
  - If true, proceeds to story availability check
- **Key expressions or variables used:**
  - `={{ $json.waiting }}`
- **Input and output connections:**
  - Input from `Aggregate and Rank`
  - True branch to `Any Stories Today?`
  - False branch back to `Loop Over Feeds`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or failures:**
  - If `Aggregate and Rank` emits malformed waiting state, loop logic can break
- **Sub-workflow reference:** None

---

## Block 4 — Story Availability Decision

### Overview
This block determines whether there are any ranked stories worth summarizing. If not, it sends an informational Telegram message instead of calling the AI model.

### Nodes Involved
- `Any Stories Today?`
- `No Stories Alert`

### Node Details

#### 9. `Any Stories Today?`
- **Type and role:** `n8n-nodes-base.if`; checks top-story count
- **Configuration choices:**
  - Condition: `topStories.length > 0`
- **Key expressions or variables used:**
  - `={{ $json.topStories ? $json.topStories.length : 0 }}`
- **Input and output connections:**
  - Input from `Is Processing Done?`
  - True branch to `Build NIM Payload`
  - False branch to `No Stories Alert`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or failures:**
  - Handles missing `topStories` by treating count as `0`
- **Sub-workflow reference:** None

#### 10. `No Stories Alert`
- **Type and role:** `n8n-nodes-base.httpRequest`; sends Telegram fallback notification
- **Configuration choices:**
  - Posts directly to Telegram Bot API `/sendMessage`
  - Uses raw JSON body with configured `chat_id`
  - Message advises checking feeds or increasing `freshnessHours`
  - `continueOnFail` is enabled
- **Key expressions or variables used:**
  - Bot token: `$('📋 RSS Feed Config1').first().json.telegramBotToken`
  - Chat ID: `$('📋 RSS Feed Config1').first().json.telegramChatId`
- **Input and output connections:** Input from `Any Stories Today?`; no downstream connection
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or failures:**
  - Invalid bot token/chat ID
  - Telegram API response errors
  - No structured error logging is attached to this branch
- **Sub-workflow reference:** None

---

## Block 5 — AI Digest Construction and Generation

### Overview
This block turns top-ranked stories into a structured prompt and sends it to NVIDIA NIM for summarization using Mistral. It then extracts the resulting digest text.

### Nodes Involved
- `Build NIM Payload`
- `Generate Digest (NIM)`
- `Extract Digest`

### Node Details

#### 11. `Build NIM Payload`
- **Type and role:** `n8n-nodes-base.code`; builds chat-completions request payload
- **Configuration choices:**
  - Reads `topStories`
  - Builds a detailed user prompt containing title, source, published date, summary, and link for each selected story
  - Builds a strict system prompt specifying:
    - exact formatting
    - no HTML
    - plain text only
    - short summaries
    - final “Today’s Signal”
  - Uses model `mistralai/mistral-large-3-675b-instruct-2512`
  - Sets:
    - `max_tokens: 1500`
    - `temperature: 0.7`
    - `top_p: 0.9`
  - Outputs:
    - serialized `nimPayload`
    - passthrough `topStories`
    - `totalCollected`
- **Key expressions or variables used:**
  - `$input.first().json.topStories`
  - `$input.first().json.totalCollected`
- **Input and output connections:** Input from `Any Stories Today?`; output to `Generate Digest (NIM)`
- **Version-specific requirements:** Code node version `2`
- **Edge cases or failures:**
  - If `topStories` is empty or malformed, prompt generation may fail
  - Strict formatting is requested but not guaranteed by the model
- **Sub-workflow reference:** None

#### 12. `Generate Digest (NIM)`
- **Type and role:** `n8n-nodes-base.httpRequest`; sends the AI prompt to NVIDIA NIM
- **Configuration choices:**
  - POST to `https://integrate.api.nvidia.com/v1/chat/completions`
  - Sends raw JSON body from `nimPayload`
  - Uses generic credential auth type with `httpHeaderAuth`
  - Adds header `Content-Type: application/json`
  - `continueOnFail` is enabled
- **Key expressions or variables used:**
  - `={{ $json.nimPayload }}`
- **Input and output connections:** Input from `Build NIM Payload`; output to `Extract Digest`
- **Version-specific requirements:** Type version `4.2`
- **Credential requirements:**
  - HTTP Header Auth credential with:
    - Header name: `Authorization`
    - Header value: `Bearer YOUR_NVIDIA_API_KEY`
- **Edge cases or failures:**
  - Invalid API key
  - Model name unavailable
  - Rate limits
  - Timeout or API-side errors
  - Error responses still flow onward because of `continueOnFail`
- **Sub-workflow reference:** None

#### 13. `Extract Digest`
- **Type and role:** `n8n-nodes-base.set`; extracts assistant response text and preserves story metadata
- **Configuration choices:**
  - `digestText` from `choices[0].message.content.trim()`
  - Fallback text: `Digest generation failed`
  - Passes through `topStories` and `totalCollected` from `Build NIM Payload`
- **Key expressions or variables used:**
  - `={{ $json.choices && $json.choices[0] ? $json.choices[0].message.content.trim() : 'Digest generation failed' }}`
  - `={{ $('Build NIM Payload').item.json.topStories }}`
  - `={{ $('Build NIM Payload').item.json.totalCollected }}`
- **Input and output connections:** Input from `Generate Digest (NIM)`; output to `Build Telegram Payload`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:**
  - If response shape differs from OpenAI-style chat-completions schema, extraction falls back
  - Fallback still allows Telegram delivery of an error-like digest body
- **Sub-workflow reference:** None

---

## Block 6 — Telegram Message Assembly and Delivery

### Overview
This block adds branding/header/footer, converts the digest into a Telegram API payload, and sends it. Delivery result is then evaluated for success or failure.

### Nodes Involved
- `Build Telegram Payload`
- `Send to Telegram`
- `Telegram Send OK?`

### Node Details

#### 14. `Build Telegram Payload`
- **Type and role:** `n8n-nodes-base.code`; formats the final Telegram message
- **Configuration choices:**
  - Uses `digestText` from upstream
  - Reads `telegramChatId` and feed count from config
  - Adds branded header:
    - `DAILY FINANCE DIGEST`
    - `Your signal in the financial noise`
  - Adds branded footer:
    - number of source feeds
    - Cordexa Technologies attribution
    - `https://cordexa.tech`
  - Sets `disable_web_page_preview: true`
  - Outputs:
    - serialized `telegramPayload`
    - full `digestText`
    - `topStories`
    - `totalCollected`
- **Key expressions or variables used:**
  - `$('📋 RSS Feed Config1').first().json.telegramChatId`
  - `$('📋 RSS Feed Config1').first().json.rssFeeds.length`
- **Input and output connections:** Input from `Extract Digest`; output to `Send to Telegram`
- **Version-specific requirements:** Code node version `2`
- **Edge cases or failures:**
  - Telegram has message length limits; a long digest may be rejected
  - Branding text can be edited in code, but formatting should remain plain text
- **Sub-workflow reference:** None

#### 15. `Send to Telegram`
- **Type and role:** `n8n-nodes-base.httpRequest`; sends the digest via Telegram Bot API
- **Configuration choices:**
  - POST request to `https://api.telegram.org/bot<TOKEN>/sendMessage`
  - Sends raw JSON body from `telegramPayload`
  - `continueOnFail` is enabled
- **Key expressions or variables used:**
  - URL from config token:
    - `={{ 'https://api.telegram.org/bot' + $('📋 RSS Feed Config1').first().json.telegramBotToken + '/sendMessage' }}`
  - Body:
    - `={{ $json.telegramPayload }}`
- **Input and output connections:** Input from `Build Telegram Payload`; output to `Telegram Send OK?`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases or failures:**
  - Invalid token
  - Unauthorized bot
  - Bot not present in channel/group
  - Invalid chat ID
  - Message too long
  - Telegram formatting/content rejection
- **Sub-workflow reference:** None

#### 16. `Telegram Send OK?`
- **Type and role:** `n8n-nodes-base.if`; splits success vs failure path based on Telegram API response
- **Configuration choices:**
  - Condition: `{{$json.ok === true}} == true`
- **Key expressions or variables used:**
  - `={{ $json.ok === true }}`
- **Input and output connections:**
  - Input from `Send to Telegram`
  - True branch to `Log Digest to Sheets`
  - False branch to `Build Error Log Row`
- **Version-specific requirements:** Type version `2.2`
- **Edge cases or failures:**
  - If Telegram returns an unexpected schema, failures may route to error branch
- **Sub-workflow reference:** None

---

## Block 7 — Success Logging and Completion

### Overview
This block logs successful deliveries to Google Sheets and returns a compact final success object.

### Nodes Involved
- `Log Digest to Sheets`
- `Digest Complete`

### Node Details

#### 17. `Log Digest to Sheets`
- **Type and role:** `n8n-nodes-base.googleSheets`; appends a success log row
- **Configuration choices:**
  - Operation: `append`
  - Sheet name: `Digest_Log`
  - Google Sheet document ID from config
  - Writes:
    - date
    - run ID
    - timestamp
    - stories count
    - first five story titles
    - Telegram status
    - total articles collected
  - `continueOnFail` is enabled
- **Key expressions or variables used:**
  - `={{ new Date().toLocaleDateString('en-US') }}`
  - `={{ $execution.id }}`
  - `={{ $('Build Telegram Payload').item.json.topStories.length }}`
  - `={{ $('Send to Telegram').item.json.ok ? 'SENT' : 'FAILED' }}`
  - `={{ $('📋 RSS Feed Config1').first().json.googleSheetId }}`
- **Input and output connections:** Input from `Telegram Send OK?` true branch; output to `Digest Complete`
- **Version-specific requirements:** Type version `4.5`
- **Credential requirements:**
  - Google Sheets OAuth2 credential
- **Edge cases or failures:**
  - Sheet/tab missing
  - Insufficient permissions
  - Schema mismatch if target columns differ
  - Since `continueOnFail` is true, final success node may run even if logging fails
- **Sub-workflow reference:** None

#### 18. `Digest Complete`
- **Type and role:** `n8n-nodes-base.set`; final success status output
- **Configuration choices:**
  - Sets:
    - `status = Digest delivered`
    - `timestamp = now`
    - `storiesSent = topStories.length`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString() }}`
  - `={{ $('Build Telegram Payload').item.json.topStories.length }}`
- **Input and output connections:** Input from `Log Digest to Sheets`; no downstream node
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:** Minimal; mostly expression-reference related
- **Sub-workflow reference:** None

---

## Block 8 — Error Logging and Failure Completion

### Overview
If Telegram delivery fails, this block builds a structured error record, writes it to Google Sheets, and returns a failure status object.

### Nodes Involved
- `Build Error Log Row`
- `Log Error to Sheets`
- `Digest Failed`

### Node Details

#### 19. `Build Error Log Row`
- **Type and role:** `n8n-nodes-base.set`; prepares an error log record
- **Configuration choices:**
  - Sets:
    - `Timestamp`
    - `Run ID`
    - `Stage = Send to Telegram`
    - `Error Message`
    - `HTTP Status`
    - `Total Articles Collected`
  - Pulls message from Telegram response fields with fallback
- **Key expressions or variables used:**
  - `={{ $json.message || $json.error || $json.description || 'Telegram request failed' }}`
  - `={{ $json.statusCode || '' }}`
  - `={{ $('Build Telegram Payload').item.json.totalCollected || 0 }}`
- **Input and output connections:** Input from `Telegram Send OK?` false branch; output to `Log Error to Sheets`
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:**
  - Telegram error response fields may vary, yielding generic message
- **Sub-workflow reference:** None

#### 20. `Log Error to Sheets`
- **Type and role:** `n8n-nodes-base.googleSheets`; appends an error row
- **Configuration choices:**
  - Operation: `append`
  - Sheet name: `Error_Log`
  - Document ID from config
  - Writes:
    - stage
    - run ID
    - timestamp
    - HTTP status
    - error message
    - total collected
  - `continueOnFail` is enabled
- **Key expressions or variables used:**
  - `={{ $('📋 RSS Feed Config1').first().json.googleSheetId }}`
- **Input and output connections:** Input from `Build Error Log Row`; output to `Digest Failed`
- **Version-specific requirements:** Type version `4.5`
- **Credential requirements:**
  - Google Sheets OAuth2 credential
- **Edge cases or failures:**
  - Missing `Error_Log` tab
  - Access/permission problems
  - Column mismatch
- **Sub-workflow reference:** None

#### 21. `Digest Failed`
- **Type and role:** `n8n-nodes-base.set`; final failure status output
- **Configuration choices:**
  - Sets:
    - `status = Digest delivery failed`
    - `timestamp = now`
- **Key expressions or variables used:**
  - `={{ new Date().toISOString() }}`
- **Input and output connections:** Input from `Log Error to Sheets`; no downstream node
- **Version-specific requirements:** Type version `3.4`
- **Edge cases or failures:** Minimal
- **Sub-workflow reference:** None

---

## Block 9 — Sticky Notes / Embedded Operational Documentation

### Overview
These nodes are not part of the executable logic, but they contain critical setup and operational guidance. They document configuration, credentials, Telegram setup, logging expectations, and activation checks.

### Nodes Involved
- `Overview`
- `Step 1 — Config`
- `Step 2 — Credentials`
- `Optional — Telegram Channel Setup`
- `Step 3 — Delivery`
- `Step 4 — Activate`

### Node Details

#### 22. `Overview`
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level operational description
- **Configuration choices:** Documents purpose, flow, required setup, and Cordexa contact details
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### 23. `Step 1 — Config`
- **Type and role:** `n8n-nodes-base.stickyNote`; pre-test configuration notes
- **Configuration choices:** Explains editable fields in `RSS Feed Config1`, feed object structure, and tier meaning
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### 24. `Step 2 — Credentials`
- **Type and role:** `n8n-nodes-base.stickyNote`; credential setup notes
- **Configuration choices:** Explains HTTP Header Auth for NVIDIA NIM and Google Sheets OAuth2 requirements
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### 25. `Optional — Telegram Channel Setup`
- **Type and role:** `n8n-nodes-base.stickyNote`; Telegram deployment note
- **Configuration choices:** Explains testing in personal chat first, then bot admin setup for channel usage
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### 26. `Step 3 — Delivery`
- **Type and role:** `n8n-nodes-base.stickyNote`; logging/delivery validation note
- **Configuration choices:** Explains required sheet tabs and test expectations for success and failure paths
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

#### 27. `Step 4 — Activate`
- **Type and role:** `n8n-nodes-base.stickyNote`; pre-activation checklist
- **Configuration choices:** Recommends validating success, no-stories, and error paths before activating
- **Input and output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases or failures:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏰ Schedule Trigger1 | Schedule Trigger | Daily workflow entry point |  | 📋 RSS Feed Config1 | ## Step 1 — Config Open `RSS Feed Config1` and update these values before testing:  - `rssFeeds` — add, remove, or replace feed URLs - `maxStoriesInDigest` — number of stories to include in each digest - `freshnessHours` — recency window for article selection - `telegramBotToken` — your Telegram bot token - `telegramChatId` — your chat ID or `@channelname` - `googleSheetId` — target Google Sheet ID for logs  Feed format: - `url` — RSS feed URL - `source` — display name used in the digest - `tier` — source weight used during ranking  Tier meaning: - Tier 1 = highest-priority sources - Tier 2 = good secondary sources - Tier 3 = lower-priority or niche sources  For testing, you can temporarily increase `freshnessHours` so older articles are still eligible. |
| 📋 RSS Feed Config1 | Set | Central runtime configuration | ⏰ Schedule Trigger1 | Generate Feed Items | ## Step 1 — Config Open `RSS Feed Config1` and update these values before testing:  - `rssFeeds` — add, remove, or replace feed URLs - `maxStoriesInDigest` — number of stories to include in each digest - `freshnessHours` — recency window for article selection - `telegramBotToken` — your Telegram bot token - `telegramChatId` — your chat ID or `@channelname` - `googleSheetId` — target Google Sheet ID for logs  Feed format: - `url` — RSS feed URL - `source` — display name used in the digest - `tier` — source weight used during ranking  Tier meaning: - Tier 1 = highest-priority sources - Tier 2 = good secondary sources - Tier 3 = lower-priority or niche sources  For testing, you can temporarily increase `freshnessHours` so older articles are still eligible. |
| Generate Feed Items | Code | Expands feed array into one item per feed | 📋 RSS Feed Config1 | Loop Over Feeds | ## Step 1 — Config Open `RSS Feed Config1` and update these values before testing:  - `rssFeeds` — add, remove, or replace feed URLs - `maxStoriesInDigest` — number of stories to include in each digest - `freshnessHours` — recency window for article selection - `telegramBotToken` — your Telegram bot token - `telegramChatId` — your chat ID or `@channelname` - `googleSheetId` — target Google Sheet ID for logs  Feed format: - `url` — RSS feed URL - `source` — display name used in the digest - `tier` — source weight used during ranking  Tier meaning: - Tier 1 = highest-priority sources - Tier 2 = good secondary sources - Tier 3 = lower-priority or niche sources  For testing, you can temporarily increase `freshnessHours` so older articles are still eligible. |
| Loop Over Feeds | Split In Batches | Iterates through feeds | Generate Feed Items, Is Processing Done? | Read RSS Feed | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Read RSS Feed | RSS Feed Read | Fetches current RSS source | Loop Over Feeds | Tag Articles | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Tag Articles | Code | Normalizes article fields and attaches source metadata | Read RSS Feed | Aggregate and Rank | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Aggregate and Rank | Code | Collects all feed items, filters, scores, deduplicates, and selects top stories | Tag Articles | Is Processing Done? | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Is Processing Done? | If | Decides whether all feeds have been processed | Aggregate and Rank | Any Stories Today?, Loop Over Feeds | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Any Stories Today? | If | Branches between digest generation and no-stories alert | Is Processing Done? | Build NIM Payload, No Stories Alert | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Build NIM Payload | Code | Builds AI request payload from ranked stories | Any Stories Today? | Generate Digest (NIM) | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Generate Digest (NIM) | HTTP Request | Calls NVIDIA NIM chat completions API | Build NIM Payload | Extract Digest | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Extract Digest | Set | Extracts digest text from AI response | Generate Digest (NIM) | Build Telegram Payload | ## Optional — Telegram Channel Setup Use a personal chat ID for testing first.  For a channel: 1. Create the channel 2. Add the bot as an admin 3. Set `telegramChatId` to `@channelname`  All Telegram nodes read the token and chat ID from `📋 RSS Feed Config1`. |
| Build Telegram Payload | Code | Wraps digest in branded Telegram message payload | Extract Digest | Send to Telegram | ## Optional — Telegram Channel Setup Use a personal chat ID for testing first.  For a channel: 1. Create the channel 2. Add the bot as an admin 3. Set `telegramChatId` to `@channelname`  All Telegram nodes read the token and chat ID from `📋 RSS Feed Config1`. |
| Send to Telegram | HTTP Request | Posts digest to Telegram Bot API | Build Telegram Payload | Telegram Send OK? | ## Optional — Telegram Channel Setup Use a personal chat ID for testing first.  For a channel: 1. Create the channel 2. Add the bot as an admin 3. Set `telegramChatId` to `@channelname`  All Telegram nodes read the token and chat ID from `📋 RSS Feed Config1`. |
| Telegram Send OK? | If | Splits success and failure delivery paths | Send to Telegram | Log Digest to Sheets, Build Error Log Row | ## Optional — Telegram Channel Setup Use a personal chat ID for testing first.  For a channel: 1. Create the channel 2. Add the bot as an admin 3. Set `telegramChatId` to `@channelname`  All Telegram nodes read the token and chat ID from `📋 RSS Feed Config1`. |
| No Stories Alert | HTTP Request | Sends fallback Telegram message when no fresh stories qualify | Any Stories Today? |  | ## Optional — Telegram Channel Setup Use a personal chat ID for testing first.  For a channel: 1. Create the channel 2. Add the bot as an admin 3. Set `telegramChatId` to `@channelname`  All Telegram nodes read the token and chat ID from `📋 RSS Feed Config1`. |
| Build Error Log Row | Set | Prepares error record for failed Telegram delivery | Telegram Send OK? | Log Error to Sheets | ## Step 3 — Delivery This workflow writes to Google Sheets and sends the digest to Telegram.  Before going live: - Create a Google Sheet with two tabs:   - `Digest_Log`   - `Error_Log` - Make sure `googleSheetId` is set in `RSS Feed Config1` - Confirm both Sheets nodes point to the same target sheet - Run one manual test and confirm a new row appears in `Digest_Log` - Force one Telegram delivery failure and confirm a row appears in `Error_Log`  Delivery behavior: - `Log Digest to Sheets` records successful runs - `Log Error to Sheets` records Telegram delivery failures - `No Stories Alert` sends a Telegram message when no fresh stories qualify for the digest |
| Log Error to Sheets | Google Sheets | Appends Telegram failure log row to `Error_Log` | Build Error Log Row | Digest Failed | ## Step 3 — Delivery This workflow writes to Google Sheets and sends the digest to Telegram.  Before going live: - Create a Google Sheet with two tabs:   - `Digest_Log`   - `Error_Log` - Make sure `googleSheetId` is set in `RSS Feed Config1` - Confirm both Sheets nodes point to the same target sheet - Run one manual test and confirm a new row appears in `Digest_Log` - Force one Telegram delivery failure and confirm a row appears in `Error_Log`  Delivery behavior: - `Log Digest to Sheets` records successful runs - `Log Error to Sheets` records Telegram delivery failures - `No Stories Alert` sends a Telegram message when no fresh stories qualify for the digest |
| Digest Failed | Set | Final failure status output | Log Error to Sheets |  | ## Step 3 — Delivery This workflow writes to Google Sheets and sends the digest to Telegram.  Before going live: - Create a Google Sheet with two tabs:   - `Digest_Log`   - `Error_Log` - Make sure `googleSheetId` is set in `RSS Feed Config1` - Confirm both Sheets nodes point to the same target sheet - Run one manual test and confirm a new row appears in `Digest_Log` - Force one Telegram delivery failure and confirm a row appears in `Error_Log`  Delivery behavior: - `Log Digest to Sheets` records successful runs - `Log Error to Sheets` records Telegram delivery failures - `No Stories Alert` sends a Telegram message when no fresh stories qualify for the digest |
| Log Digest to Sheets | Google Sheets | Appends success log row to `Digest_Log` | Telegram Send OK? | Digest Complete | ## Step 3 — Delivery This workflow writes to Google Sheets and sends the digest to Telegram.  Before going live: - Create a Google Sheet with two tabs:   - `Digest_Log`   - `Error_Log` - Make sure `googleSheetId` is set in `RSS Feed Config1` - Confirm both Sheets nodes point to the same target sheet - Run one manual test and confirm a new row appears in `Digest_Log` - Force one Telegram delivery failure and confirm a row appears in `Error_Log`  Delivery behavior: - `Log Digest to Sheets` records successful runs - `Log Error to Sheets` records Telegram delivery failures - `No Stories Alert` sends a Telegram message when no fresh stories qualify for the digest |
| Digest Complete | Set | Final success status output | Log Digest to Sheets |  | ## Step 4 — Activate Before activating the schedule:  - Run the workflow manually once - Confirm the digest reaches Telegram - Confirm `Digest_Log` receives a new row - Confirm story count and titles are recorded correctly - Confirm the no-stories branch works when no fresh items qualify - Confirm `Error_Log` receives a row when Telegram delivery fails  Activate the workflow only after both success and error paths behave as expected. |
| Overview | Sticky Note | Embedded workflow documentation |  |  | ## Overview **Who it's for:** Operators, investors, analysts, founders, and solo builders who want a daily finance news digest delivered to Telegram without manually checking multiple RSS feeds.  **What it does:** Runs once per day, reads recent stories from curated finance RSS feeds, ranks the strongest items, uses NVIDIA NIM with Mistral Large 3 to write a concise digest, sends the digest to Telegram, and logs the run to Google Sheets.  **How it works:** 1. `Schedule Trigger1` starts the workflow on a daily cron. 2. `RSS Feed Config1` defines feeds, thresholds, Telegram values, and the target Google Sheet ID. 3. `Generate Feed Items` and `Loop Over Feeds` process each RSS feed one at a time. 4. `Read RSS Feed`, `Tag Articles`, and `Aggregate and Rank` collect, score, filter, deduplicate, and rank stories. 5. `Any Stories Today?` decides whether to generate a digest or send a no-stories alert. 6. `Build NIM Payload` and `Generate Digest (NIM)` create the AI-written digest. 7. `Build Telegram Payload` and `Send to Telegram` deliver the final message. 8. `Log Digest to Sheets` records successful runs, and `Log Error to Sheets` records delivery failures.  **Required setup:** - NVIDIA NIM API key via HTTP Header Auth - Google Sheets OAuth2 credential - Telegram bot token - Telegram chat ID or `@channelname` - A Google Sheet with `Digest_Log` and `Error_Log` tabs  Built by Cordexa Technologies  https://cordexa.tech  cordexatech@gmail.com |
| Step 1 — Config | Sticky Note | Embedded config guidance |  |  | ## Step 1 — Config Open `RSS Feed Config1` and update these values before testing:  - `rssFeeds` — add, remove, or replace feed URLs - `maxStoriesInDigest` — number of stories to include in each digest - `freshnessHours` — recency window for article selection - `telegramBotToken` — your Telegram bot token - `telegramChatId` — your chat ID or `@channelname` - `googleSheetId` — target Google Sheet ID for logs  Feed format: - `url` — RSS feed URL - `source` — display name used in the digest - `tier` — source weight used during ranking  Tier meaning: - Tier 1 = highest-priority sources - Tier 2 = good secondary sources - Tier 3 = lower-priority or niche sources  For testing, you can temporarily increase `freshnessHours` so older articles are still eligible. |
| Step 2 — Credentials | Sticky Note | Embedded credential guidance |  |  | ## Step 2 — Credentials Connect these credentials before going live:  - **HTTP Header Auth** on `Generate Digest (NIM)`   - Header name: `Authorization`   - Value: `Bearer YOUR_NVIDIA_API_KEY`  - **Google Sheets OAuth2** on both Sheets nodes   - `Log Digest to Sheets`   - `Log Error to Sheets`  This workflow does **not** use a Telegram credential node. Telegram delivery uses the bot token stored in `RSS Feed Config1`, and the HTTP Request nodes read that value directly.  Also confirm `Generate Digest (NIM)` still sends: - `Content-Type: application/json` - POST request to `https://integrate.api.nvidia.com/v1/chat/completions` |
| Optional — Telegram Channel Setup | Sticky Note | Embedded Telegram deployment guidance |  |  | ## Optional — Telegram Channel Setup Use a personal chat ID for testing first.  For a channel: 1. Create the channel 2. Add the bot as an admin 3. Set `telegramChatId` to `@channelname`  All Telegram nodes read the token and chat ID from `📋 RSS Feed Config1`. |
| Step 3 — Delivery | Sticky Note | Embedded delivery/logging guidance |  |  | ## Step 3 — Delivery This workflow writes to Google Sheets and sends the digest to Telegram.  Before going live: - Create a Google Sheet with two tabs:   - `Digest_Log`   - `Error_Log` - Make sure `googleSheetId` is set in `RSS Feed Config1` - Confirm both Sheets nodes point to the same target sheet - Run one manual test and confirm a new row appears in `Digest_Log` - Force one Telegram delivery failure and confirm a row appears in `Error_Log`  Delivery behavior: - `Log Digest to Sheets` records successful runs - `Log Error to Sheets` records Telegram delivery failures - `No Stories Alert` sends a Telegram message when no fresh stories qualify for the digest |
| Step 4 — Activate | Sticky Note | Embedded activation checklist |  |  | ## Step 4 — Activate Before activating the schedule:  - Run the workflow manually once - Confirm the digest reaches Telegram - Confirm `Digest_Log` receives a new row - Confirm story count and titles are recorded correctly - Confirm the no-stories branch works when no fresh items qualify - Confirm `Error_Log` receives a row when Telegram delivery fails  Activate the workflow only after both success and error paths behave as expected. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger node** named `⏰ Schedule Trigger1`.
   - Type: `Schedule Trigger`
   - Configure cron expression: `0 10 * * *`
   - This runs once daily at 10:00.

3. **Add a Set node** named `📋 RSS Feed Config1`.
   - Connect `⏰ Schedule Trigger1 -> 📋 RSS Feed Config1`
   - Add these fields:
     - `rssFeeds` as an array of objects:
       - `url`
       - `tier`
       - `source`
     - `telegramBotToken` as string
     - `telegramChatId` as string
     - `maxStoriesInDigest` as number, default `5`
     - `freshnessHours` as number, default `24`
     - `googleSheetId` as string
     - `runId` as expression: `={{ $execution.id }}`
   - Populate the feed array with the finance feeds you want to monitor.

4. **Add a Code node** named `Generate Feed Items`.
   - Connect `📋 RSS Feed Config1 -> Generate Feed Items`
   - Use code that reads `rssFeeds` from the first item and returns one item per feed with:
     - `feedUrl`
     - `feedSource`
     - `feedTier`

5. **Add a Split In Batches node** named `Loop Over Feeds`.
   - Connect `Generate Feed Items -> Loop Over Feeds`
   - Default settings are sufficient.

6. **Add an RSS Feed Read node** named `Read RSS Feed`.
   - Connect `Loop Over Feeds -> Read RSS Feed`
   - Set URL to expression: `={{ $json.feedUrl }}`
   - Enable `Continue On Fail`

7. **Add a Code node** named `Tag Articles`.
   - Connect `Read RSS Feed -> Tag Articles`
   - Write code to normalize all feed items into:
     - `title`
     - `link`
     - `pubDate`
     - `description`
     - `sourceName`
     - `tier`
   - Pull `sourceName` and `tier` from `Loop Over Feeds`.

8. **Add a Code node** named `Aggregate and Rank`.
   - Connect `Tag Articles -> Aggregate and Rank`
   - Use workflow global static data:
     - initialize/reset per `runId`
     - accumulate all articles across loop iterations
     - count processed feeds
   - When not all feeds are processed, return:
     - `waiting: true`
     - `feedsProcessed`
     - `totalFeeds`
   - Once all feeds are processed:
     - filter out missing `title`/`link`
     - filter by `freshnessHours`
     - filter titles containing banned words such as sponsored/promotional terms
     - deduplicate by link
     - score by source tier, recency, and finance keywords
     - deduplicate near-identical titles
     - sort descending
     - slice to `maxStoriesInDigest`
   - Return:
     - `topStories`
     - `totalCollected`
     - optionally `empty: true` if none remain

9. **Add an If node** named `Is Processing Done?`.
   - Connect `Aggregate and Rank -> Is Processing Done?`
   - Condition:
     - left expression: `={{ $json.waiting }}`
     - operator: `not equals`
     - right value: `true`
   - Connect:
     - **false output** back to `Loop Over Feeds`
     - **true output** to the next decision node

10. **Add an If node** named `Any Stories Today?`.
    - Connect `Is Processing Done? -> Any Stories Today?`
    - Condition:
      - left expression: `={{ $json.topStories ? $json.topStories.length : 0 }}`
      - operator: `greater than`
      - right value: `0`

11. **Add an HTTP Request node** named `No Stories Alert`.
    - Connect the **false** branch of `Any Stories Today?` to this node
    - Method: `POST`
    - URL expression:
      `={{ 'https://api.telegram.org/bot' + $('📋 RSS Feed Config1').first().json.telegramBotToken + '/sendMessage' }}`
    - Send body: enabled
    - Content type: `Raw`
    - Raw content type: `application/json`
    - Body should include:
      - `chat_id` from config
      - plain text message explaining that no fresh stories were found and suggesting increasing `freshnessHours`
    - Enable `Continue On Fail`

12. **Add a Code node** named `Build NIM Payload`.
    - Connect the **true** branch of `Any Stories Today?` to this node
    - Build a chat-completions payload using:
      - system prompt with strict digest format
      - user prompt enumerating each story with title, source, published date, summary, and link
    - Use model:
      - `mistralai/mistral-large-3-675b-instruct-2512`
    - Set:
      - `max_tokens` = `1500`
      - `temperature` = `0.7`
      - `top_p` = `0.9`
    - Output fields:
      - `nimPayload` as JSON string
      - `topStories`
      - `totalCollected`

13. **Create NVIDIA credential** before the next node.
    - Credential type: `HTTP Header Auth`
    - Header name: `Authorization`
    - Header value: `Bearer YOUR_NVIDIA_API_KEY`

14. **Add an HTTP Request node** named `Generate Digest (NIM)`.
    - Connect `Build NIM Payload -> Generate Digest (NIM)`
    - Method: `POST`
    - URL: `https://integrate.api.nvidia.com/v1/chat/completions`
    - Authentication: `Generic Credential Type`
    - Generic auth type: `HTTP Header Auth`
    - Select the NVIDIA credential
    - Send headers: enabled
    - Add header:
      - `Content-Type: application/json`
    - Send body: enabled
    - Content type: `Raw`
    - Raw content type: `application/json`
    - Body expression: `={{ $json.nimPayload }}`
    - Enable `Continue On Fail`

15. **Add a Set node** named `Extract Digest`.
    - Connect `Generate Digest (NIM) -> Extract Digest`
    - Create fields:
      - `digestText`:
        `={{ $json.choices && $json.choices[0] ? $json.choices[0].message.content.trim() : 'Digest generation failed' }}`
      - `topStories`:
        `={{ $('Build NIM Payload').item.json.topStories }}`
      - `totalCollected`:
        `={{ $('Build NIM Payload').item.json.totalCollected }}`

16. **Add a Code node** named `Build Telegram Payload`.
    - Connect `Extract Digest -> Build Telegram Payload`
    - Build a final text payload with:
      - plain text header
      - digest body
      - plain text footer
      - attribution
      - feed count
    - Output:
      - `telegramPayload` as JSON string
      - `digestText`
      - `topStories`
      - `totalCollected`
    - Include:
      - `chat_id`
      - `text`
      - `disable_web_page_preview: true`

17. **Add an HTTP Request node** named `Send to Telegram`.
    - Connect `Build Telegram Payload -> Send to Telegram`
    - Method: `POST`
    - URL expression:
      `={{ 'https://api.telegram.org/bot' + $('📋 RSS Feed Config1').first().json.telegramBotToken + '/sendMessage' }}`
    - Body expression: `={{ $json.telegramPayload }}`
    - Content type: `Raw`
    - Raw content type: `application/json`
    - Enable `Continue On Fail`

18. **Add an If node** named `Telegram Send OK?`.
    - Connect `Send to Telegram -> Telegram Send OK?`
    - Condition:
      - left expression: `={{ $json.ok === true }}`
      - operator: `equals`
      - right value: `true`

19. **Prepare Google Sheets destination**.
    - Create a Google Sheet with two tabs:
      - `Digest_Log`
      - `Error_Log`

20. **Create Google Sheets OAuth2 credential**.
    - Connect a Google account with access to the target spreadsheet.

21. **Add a Google Sheets node** named `Log Digest to Sheets`.
    - Connect the **true** branch of `Telegram Send OK?` to this node
    - Operation: `Append`
    - Document ID:
      `={{ $('📋 RSS Feed Config1').first().json.googleSheetId }}`
    - Sheet name: `Digest_Log`
    - Map fields manually:
      - `Date` = current local US date
      - `Run ID` = `{{$execution.id}}`
      - `Timestamp` = current ISO timestamp
      - `Stories Count` = `$('Build Telegram Payload').item.json.topStories.length`
      - `Story 1 Title` through `Story 5 Title`
      - `Telegram Status` = `SENT` if Telegram response `ok`
      - `Total Articles Collected`
    - Enable `Continue On Fail`

22. **Add a Set node** named `Digest Complete`.
    - Connect `Log Digest to Sheets -> Digest Complete`
    - Fields:
      - `status = Digest delivered`
      - `timestamp = {{ new Date().toISOString() }}`
      - `storiesSent = {{ $('Build Telegram Payload').item.json.topStories.length }}`

23. **Add a Set node** named `Build Error Log Row`.
    - Connect the **false** branch of `Telegram Send OK?` to this node
    - Fields:
      - `Timestamp`
      - `Run ID`
      - `Stage = Send to Telegram`
      - `Error Message`
      - `HTTP Status`
      - `Total Articles Collected`
    - Use error fallbacks from the Telegram response body.

24. **Add a Google Sheets node** named `Log Error to Sheets`.
    - Connect `Build Error Log Row -> Log Error to Sheets`
    - Operation: `Append`
    - Document ID:
      `={{ $('📋 RSS Feed Config1').first().json.googleSheetId }}`
    - Sheet name: `Error_Log`
    - Map:
      - `Stage`
      - `Run ID`
      - `Timestamp`
      - `HTTP Status`
      - `Error Message`
      - `Total Articles Collected`
    - Use the same Google Sheets OAuth2 credential
    - Enable `Continue On Fail`

25. **Add a Set node** named `Digest Failed`.
    - Connect `Log Error to Sheets -> Digest Failed`
    - Fields:
      - `status = Digest delivery failed`
      - `timestamp = {{ new Date().toISOString() }}`

26. **Optionally add sticky notes** mirroring the original workflow.
    - Add notes for:
      - overview
      - config checklist
      - credentials
      - Telegram channel setup
      - delivery/logging
      - activation checklist

27. **Manual validation pass**
    - Replace placeholders in `📋 RSS Feed Config1`
    - Run once manually
    - Confirm:
      - RSS feeds return items
      - ranked stories appear
      - NVIDIA NIM returns digest text
      - Telegram receives the message
      - `Digest_Log` gets a row

28. **Test edge branches**
    - Temporarily reduce eligible content or tighten `freshnessHours` to force the no-stories branch
    - Use an invalid Telegram token or chat ID temporarily to force the error branch
    - Confirm `Error_Log` gets a row

29. **Activate the workflow**
    - Only after success, no-stories, and failure paths have all been verified

### Credential Summary
- **HTTP Header Auth**
  - Used by: `Generate Digest (NIM)`
  - Header: `Authorization`
  - Value: `Bearer YOUR_NVIDIA_API_KEY`

- **Google Sheets OAuth2**
  - Used by:
    - `Log Digest to Sheets`
    - `Log Error to Sheets`

- **Telegram**
  - No n8n credential object used
  - Bot token and chat ID are stored in `📋 RSS Feed Config1`

### Input/Output Expectations
- **Workflow input:** none external; schedule-driven
- **Main output on success:** final item from `Digest Complete`
- **Main output on Telegram failure:** final item from `Digest Failed`
- **No-stories path:** sends Telegram alert and stops without final logging node in this version

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Built by Cordexa Technologies | https://cordexa.tech |
| Contact email listed in the workflow note | cordexatech@gmail.com |
| Intended audience: operators, investors, analysts, founders, and solo builders | Context from workflow overview note |
| Required Google Sheet tabs: `Digest_Log` and `Error_Log` | Delivery/logging setup |
| Telegram channel use requires adding the bot as an admin and setting `telegramChatId` to `@channelname` | Telegram deployment note |
| The workflow does not use a Telegram credential node; the bot token is read from configuration | Credential/design note |
| NVIDIA endpoint expected by the workflow | `https://integrate.api.nvidia.com/v1/chat/completions` |

## Additional Implementation Notes
- The workflow has **one entry point**: `⏰ Schedule Trigger1`.
- It has **no sub-workflow nodes** and does **not invoke any child workflow**.
- The most important operational risk is the use of **workflow global static data** in `Aggregate and Rank`; overlapping executions could interfere with each other.
- The no-stories branch currently sends a Telegram alert but does **not** write a Google Sheets log row.
- Telegram message length should be monitored; if the digest becomes too long, consider shortening prompts or splitting messages.