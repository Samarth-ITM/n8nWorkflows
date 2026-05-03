Summarize QA testing news and send Telegram updates with Gemini and OpenAI

https://n8nworkflows.xyz/workflows/summarize-qa-testing-news-and-send-telegram-updates-with-gemini-and-openai-13832


# Summarize QA testing news and send Telegram updates with Gemini and OpenAI

# 1. Workflow Overview

This workflow collects QA-related news from multiple RSS feeds every 3 hours, keeps only articles published today, removes duplicates, generates short AI summaries, and posts the results to a Telegram channel.

Its main use case is automated news monitoring for QA engineers, test automation teams, or technical communities that want a lightweight digest of relevant articles without manually checking several blogs and feeds.

## 1.1 Source Ingestion

The workflow starts on a schedule, defines a list of RSS feed URLs, then iterates through them one by one and reads their feed items.

## 1.2 Normalization and Date Filtering

For each feed item, it extracts the relevant article fields, computes the current date, formats the article publication date, and filters to keep only items published today.

## 1.3 Deduplication

The filtered articles are deduplicated twice:
1. within the current execution
2. against previous workflow executions

This prevents repeated Telegram updates for the same article.

## 1.4 Batch Processing and AI Summarization

The remaining articles are processed in batches of 5. Each article waits 10 seconds before AI processing. Gemini is used as the primary summarizer. If Gemini fails or returns an empty summary, the workflow falls back to OpenAI GPT-4o-mini.

## 1.5 Telegram Delivery

The generated summary, along with the article title and link, is sent to Telegram. The OpenAI branch includes an additional item loop for controlled delivery.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Trigger and Feed Source Preparation

### Overview
This block starts the workflow every 3 hours and prepares the list of RSS feed URLs to process. It converts a static array of feed URLs into individual items so they can be fetched one at a time.

### Nodes Involved
- `Schedular`
- `Input Feeds List`
- `Fetch one url at a time`
- `Loop Over Feed URL's`
- `No Operation, do nothing`

### Node Details

#### Schedular
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based entry point.
- **Configuration choices:** Runs every 3 hours using an hourly interval rule.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry node → `Input Feeds List`
- **Version-specific requirements:** Uses typeVersion `1.3`.
- **Edge cases or potential failure types:** Usually stable; failures would mostly relate to disabled workflow status or instance scheduling issues.
- **Sub-workflow reference:** None.

#### Input Feeds List
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a single item containing an array of RSS feed URLs.
- **Configuration choices:** Defines `urls` as an array with:
  - `https://www.thetesttribe.com/feed/`
  - `https://medium.com/feed/tag/qa-testing`
  - `https://www.cypress.io/feed.xml`
  - `https://dev.to/feed`
- **Key expressions or variables used:** Static array assignment.
- **Input and output connections:** `Schedular` → `Fetch one url at a time`
- **Version-specific requirements:** Uses Set node typeVersion `3.4`.
- **Edge cases or potential failure types:** Wrong array syntax or malformed URL values would break downstream feed retrieval.
- **Sub-workflow reference:** None.

#### Fetch one url at a time
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Splits the `urls` array into separate items, one URL per item.
- **Configuration choices:** `fieldToSplitOut = urls`
- **Key expressions or variables used:** Uses the `urls` field created by the previous Set node.
- **Input and output connections:** `Input Feeds List` → `Loop Over Feed URL's`
- **Version-specific requirements:** Uses typeVersion `1`.
- **Edge cases or potential failure types:** Fails if `urls` is missing or not an array.
- **Sub-workflow reference:** None.

#### Loop Over Feed URL's
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the feed URLs.
- **Configuration choices:** Default options; batch size is not explicitly set.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  Input from `Fetch one url at a time`  
  Main output to `Read Feed`  
  Loop completion output to `No Operation, do nothing`
- **Version-specific requirements:** Uses typeVersion `3`.
- **Edge cases or potential failure types:** Misuse of loop-back patterns can cause partial processing if downstream connections are altered.
- **Sub-workflow reference:** None.

#### No Operation, do nothing
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Placeholder node marking the loop’s completion path.
- **Configuration choices:** Default.
- **Key expressions or variables used:** None.
- **Input and output connections:** Receives loop completion from `Loop Over Feed URL's`
- **Version-specific requirements:** TypeVersion `1`.
- **Edge cases or potential failure types:** None functionally.
- **Sub-workflow reference:** None.

---

## 2.2 Block: Feed Reading and Data Extraction

### Overview
This block fetches articles from each RSS feed URL and extracts only the fields required for filtering, summarization, and delivery.

### Nodes Involved
- `Read Feed`
- `Extract Data`

### Node Details

#### Read Feed
- **Type and technical role:** `n8n-nodes-base.rssFeedRead`  
  Reads RSS/Atom feed contents from the current URL item.
- **Configuration choices:** URL is dynamic: `{{ $json.urls }}`
- **Key expressions or variables used:** `={{ $json.urls }}`
- **Input and output connections:**  
  Input from `Loop Over Feed URL's`  
  Output to `Extract Data` and back to `Loop Over Feed URL's`
- **Version-specific requirements:** TypeVersion `1.2`.
- **Edge cases or potential failure types:**
  - Invalid feed URL
  - Feed timeout
  - Non-standard RSS/Atom formatting
  - Network/DNS issues
- **Sub-workflow reference:** None.

#### Extract Data
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes feed output into a predictable structure.
- **Configuration choices:** Keeps:
  - `title = $json.title`
  - `link = $json.link`
  - `contentSnippet = $json.contentSnippet`
  - `pubDate = $json.isoDate`
  Dot notation is enabled and conversion errors are ignored.
- **Key expressions or variables used:**
  - `={{ $json.title }}`
  - `={{ $json.link }}`
  - `={{ $json.contentSnippet }}`
  - `={{ $json.isoDate }}`
- **Input and output connections:** `Read Feed` → `Get Current Date`
- **Version-specific requirements:** TypeVersion `3.4`.
- **Edge cases or potential failure types:**
  - Missing `contentSnippet` on some feeds
  - Missing `isoDate`
  - Empty title or link fields
- **Sub-workflow reference:** None.

---

## 2.3 Block: Date Formatting and Filtering

### Overview
This block computes the current date, formats both current and publication dates, and keeps only articles published today.

### Nodes Involved
- `Get Current Date`
- `Get Feed Published Date`
- `Formatted Current Date`
- `Filter by today's news`

### Node Details

#### Get Current Date
- **Type and technical role:** `n8n-nodes-base.dateTime`  
  Adds current date/time metadata while preserving incoming fields.
- **Configuration choices:** `includeInputFields = true`
- **Key expressions or variables used:** None explicit in parameters.
- **Input and output connections:** `Extract Data` → `Get Feed Published Date`
- **Version-specific requirements:** TypeVersion `2`.
- **Edge cases or potential failure types:** Rare; mainly timezone expectations versus feed timezone.
- **Sub-workflow reference:** None.

#### Get Feed Published Date
- **Type and technical role:** `n8n-nodes-base.dateTime`  
  Formats the publication date from the feed into a comparable date field.
- **Configuration choices:**
  - Operation: `formatDate`
  - Input date: `{{ $json.pubDate }}`
  - Output field: `publishedDate`
  - Includes original fields
- **Key expressions or variables used:** `={{ $json.pubDate }}`
- **Input and output connections:** `Get Current Date` → `Formatted Current Date`
- **Version-specific requirements:** TypeVersion `2`.
- **Edge cases or potential failure types:**
  - Invalid or missing `pubDate`
  - Locale/timezone mismatches
- **Sub-workflow reference:** None.

#### Formatted Current Date
- **Type and technical role:** `n8n-nodes-base.dateTime`  
  Formats the generated current date into a comparable output field.
- **Configuration choices:**
  - Operation: `formatDate`
  - Input date: `{{ $json.currentDate }}`
  - Output field: `currentDate`
  - Includes original fields
- **Key expressions or variables used:** `={{ $json.currentDate }}`
- **Input and output connections:** `Get Feed Published Date` → `Filter by today's news`
- **Version-specific requirements:** TypeVersion `2`.
- **Edge cases or potential failure types:** If `currentDate` is absent due to upstream changes, the filter breaks.
- **Sub-workflow reference:** None.

#### Filter by today's news
- **Type and technical role:** `n8n-nodes-base.filter`  
  Keeps only items where `publishedDate` equals `currentDate`.
- **Configuration choices:**
  - Case-insensitive enabled
  - Condition type: dateTime `equals`
  - Left: `{{ $json.publishedDate }}`
  - Right: `{{ $json.currentDate }}`
- **Key expressions or variables used:**
  - `={{$json.publishedDate}}`
  - `={{ $json.currentDate }}`
- **Input and output connections:** `Formatted Current Date` → `Remove Duplicates`
- **Version-specific requirements:** TypeVersion `2.3`.
- **Edge cases or potential failure types:**
  - Timezone skew can exclude valid “today” items
  - Feeds without publication date will be dropped
  - Exact equality may be too strict depending on formatting
- **Sub-workflow reference:** None.

---

## 2.4 Block: Deduplication

### Overview
This block removes repeated articles both inside the same run and across previous executions using a combined `title + link` identity.

### Nodes Involved
- `Remove Duplicates`
- `Remove Duplicates Comparing with Prev Exec`

### Node Details

#### Remove Duplicates
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Removes duplicate items within the current execution.
- **Configuration choices:** Default options; no custom key shown.
- **Key expressions or variables used:** None explicit.
- **Input and output connections:** `Filter by today's news` → `Remove Duplicates Comparing with Prev Exec`
- **Version-specific requirements:** TypeVersion `2`.
- **Edge cases or potential failure types:**
  - Deduplication behavior depends on node defaults
  - If feed items differ in non-key fields, duplicates may still pass depending on implementation
- **Sub-workflow reference:** None.

#### Remove Duplicates Comparing with Prev Exec
- **Type and technical role:** `n8n-nodes-base.removeDuplicates`  
  Removes items already seen in previous workflow runs.
- **Configuration choices:**
  - Operation: `removeItemsSeenInPreviousExecutions`
  - Dedupe value: `{{ $json.title }}{{ $json.link }}`
- **Key expressions or variables used:** `={{ $json.title }}{{ $json.link }}`
- **Input and output connections:** `Remove Duplicates` → `Loop over filtered news`
- **Version-specific requirements:** TypeVersion `2`.
- **Edge cases or potential failure types:**
  - If article title changes slightly, the same link may bypass history filtering
  - If execution history is unavailable or pruned, old items may reappear
- **Sub-workflow reference:** None.

---

## 2.5 Block: Article Batching and Rate-Limit Control

### Overview
This block batches filtered items for downstream processing and inserts a delay before AI calls and Telegram posting. The wait is a critical rate-control mechanism.

### Nodes Involved
- `Loop over filtered news`
- `No Operation, do nothing1`
- `Wait for 10 seconds`

### Node Details

#### Loop over filtered news
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Processes filtered news in batches of 5.
- **Configuration choices:** `batchSize = 5`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Input from `Remove Duplicates Comparing with Prev Exec`  
  Main output to `Wait for 10 seconds`  
  Completion output to `No Operation, do nothing1`
- **Version-specific requirements:** TypeVersion `3`.
- **Edge cases or potential failure types:** Incorrect loop rewiring can stop batch iteration.
- **Sub-workflow reference:** None.

#### No Operation, do nothing1
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Placeholder node for loop completion.
- **Configuration choices:** Default.
- **Key expressions or variables used:** None.
- **Input and output connections:** Receives completion path from `Loop over filtered news`
- **Version-specific requirements:** TypeVersion `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Wait for 10 seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds a fixed pause before summarization and delivery.
- **Configuration choices:** Wait amount is `10` seconds.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Input from `Loop over filtered news`  
  Outputs to `Gemini` and back to `Loop over filtered news`
- **Version-specific requirements:** TypeVersion `1.1`.
- **Edge cases or potential failure types:**
  - Removing this node may increase Telegram rate-limit risk
  - Wait nodes can create resume-state dependence in some hosting setups
- **Sub-workflow reference:** None.

---

## 2.6 Block: Primary AI Summarization with Gemini

### Overview
This block attempts to summarize each article using Google Gemini. The result is normalized into a simple `aiSummary` field for downstream checks.

### Nodes Involved
- `Gemini`
- `Set Gemini AI Summary`

### Node Details

#### Gemini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini`  
  Calls Gemini as the primary LLM summarizer.
- **Configuration choices:**
  - Model: `models/gemini-2.5-flash`
  - Temperature: `0.4`
  - Max output tokens: `900`
  - Error handling: `continueRegularOutput`
- **Prompt behavior:** Requests a 3-bullet concise QA-focused summary based on:
  - `title`
  - `contentSnippet`
- **Key expressions or variables used:**
  - `{{$json.title}}`
  - `{{$json.contentSnippet}}`
- **Input and output connections:** `Wait for 10 seconds` → `Set Gemini AI Summary`
- **Version-specific requirements:** TypeVersion `1.1`; requires Google Gemini/PaLM credentials configured in n8n.
- **Edge cases or potential failure types:**
  - Authentication failure
  - Quota/rate-limit issues
  - Empty `contentSnippet` causing poor summaries
  - Model output structure changes could break extraction
  - Node intentionally continues regular output on error so fallback can work
- **Sub-workflow reference:** None.

#### Set Gemini AI Summary
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts plain text summary from Gemini response.
- **Configuration choices:**
  - Sets `aiSummary = $json.content.parts[0].text`
  - Includes other fields
- **Key expressions or variables used:** `={{ $json.content.parts[0].text }}`
- **Input and output connections:** `Gemini` → `Check AI Summary or error`
- **Version-specific requirements:** TypeVersion `3.4`.
- **Edge cases or potential failure types:**
  - If Gemini returns a different structure, the expression fails or becomes empty
  - `content.parts[0]` may not exist on errors
- **Sub-workflow reference:** None.

---

## 2.7 Block: AI Fallback Decision and OpenAI Summarization

### Overview
This block checks whether Gemini produced a usable summary. If the summary is empty or an error field exists, the article is routed to OpenAI GPT-4o-mini as fallback.

### Nodes Involved
- `Check AI Summary or error`
- `OpenAI`
- `Set OpenAI summary`

### Node Details

#### Check AI Summary or error
- **Type and technical role:** `n8n-nodes-base.if`  
  Routes items based on Gemini result quality.
- **Configuration choices:**
  - OR logic
  - Condition 1: `aiSummary` is empty
  - Condition 2: `error` exists
  - Loose validation enabled
- **Key expressions or variables used:**
  - `={{ $json.aiSummary }}`
  - `={{ $json.error }}`
- **Input and output connections:**  
  Input from `Set Gemini AI Summary`  
  True branch → `OpenAI`  
  False branch → `Send News To Telegram Channel`
- **Version-specific requirements:** TypeVersion `2.3`.
- **Edge cases or potential failure types:**
  - A malformed Gemini response may produce neither `aiSummary` nor `error`, but still trigger fallback if summary is empty
  - If Gemini output is low quality but non-empty, fallback will not occur
- **Sub-workflow reference:** None.

#### OpenAI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  Secondary LLM summarizer used as fallback.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: `500`
  - System role: `You are a senior QA automation engineer`
  - User prompt references:
    - title from `Wait for 10 seconds`
    - link from `Wait for 10 seconds`
- **Key expressions or variables used:**
  - `{{ $('Wait for 10 seconds').item.json.title }}`
  - `{{ $('Wait for 10 seconds').item.json.link }}`
- **Important note:** The prompt says `Content:` but actually passes the article `link`, not the article body/snippet.
- **Input and output connections:** `Check AI Summary or error` → `Set OpenAI summary`
- **Version-specific requirements:** TypeVersion `2.1`; requires OpenAI credentials.
- **Edge cases or potential failure types:**
  - Authentication/quota issues
  - Since the prompt uses the URL instead of article text, summaries may be weak or generic
  - Response schema changes could affect parsing downstream
- **Sub-workflow reference:** None.

#### Set OpenAI summary
- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts the text summary from the OpenAI response into `aiSummary`.
- **Configuration choices:**
  - Sets `aiSummary = $json.output[0].content[0].text`
  - Includes other fields
- **Key expressions or variables used:** `={{$json.output[0].content[0].text}}`
- **Input and output connections:**  
  `OpenAI` → `Send News To Telegram Channel1` and `Loop Over Items`
- **Version-specific requirements:** TypeVersion `3.4`.
- **Edge cases or potential failure types:**
  - Expression fails if output schema differs
  - Duplicate downstream connections may create redundant sends or confusing loop behavior
- **Sub-workflow reference:** None.

---

## 2.8 Block: Telegram Delivery

### Overview
This block sends the generated article digest to Telegram. There are two Telegram nodes: one for the successful Gemini path and one for the OpenAI fallback path.

### Nodes Involved
- `Send News To Telegram Channel`
- `Loop Over Items`
- `Send News To Telegram Channel1`
- `No Operation, do nothing2`

### Node Details

#### Send News To Telegram Channel
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message for the Gemini-success branch.
- **Configuration choices:**
  - Uses a fixed `chatId` placeholder: `123456789`
  - Message text includes:
    - title from `Wait for 10 seconds`
    - link from `Wait for 10 seconds`
    - `aiSummary` from current item
  - Parse mode: HTML
  - Attribution disabled
  - Web preview enabled
- **Key expressions or variables used:**
  - `{{ $('Wait for 10 seconds').item.json.title }}`
  - `{{ $('Wait for 10 seconds').item.json.link }}`
  - `{{ $json.aiSummary }}`
- **Input and output connections:** False branch of `Check AI Summary or error`
- **Version-specific requirements:** TypeVersion `1.2`; requires Telegram Bot credentials.
- **Edge cases or potential failure types:**
  - Invalid bot token
  - Wrong chat ID
  - Bot not added to target group/channel
  - HTML parse issues if summary contains unsupported markup
- **Sub-workflow reference:** None.

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Additional batching loop used on the OpenAI fallback delivery branch.
- **Configuration choices:** `batchSize = 3`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Input from `Set OpenAI summary` and from `Send News To Telegram Channel1`  
  Main completion output to `No Operation, do nothing2`  
  Batch output to `Send News To Telegram Channel1`
- **Version-specific requirements:** TypeVersion `3`.
- **Edge cases or potential failure types:**
  - The branch wiring is unusual and may be redundant because `Set OpenAI summary` also directly connects to `Send News To Telegram Channel1`
  - Can lead to confusion when modifying or debugging delivery order
- **Sub-workflow reference:** None.

#### Send News To Telegram Channel1
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends Telegram messages for the OpenAI fallback path.
- **Configuration choices:**
  - Fixed `chatId` placeholder: `123456789`
  - Message text includes title, link, and `aiSummary`
  - Parse mode: HTML
  - Attribution disabled
  - Web preview enabled
- **Key expressions or variables used:**
  - `{{ $('Wait for 10 seconds').item.json.title }}`
  - `{{ $('Wait for 10 seconds').item.json.link }}`
  - `{{ $json.aiSummary }}`
- **Input and output connections:**  
  Input from `Set OpenAI summary` and `Loop Over Items`  
  Output back to `Loop Over Items`
- **Version-specific requirements:** TypeVersion `1.2`; requires Telegram credentials.
- **Edge cases or potential failure types:**
  - Same Telegram auth/chat issues as the other Telegram node
  - Because of loop wiring, duplicate sends are possible if reconfigured incorrectly
- **Sub-workflow reference:** None.

#### No Operation, do nothing2
- **Type and technical role:** `n8n-nodes-base.noOp`  
  End marker for the OpenAI item loop completion.
- **Configuration choices:** Default.
- **Key expressions or variables used:** None.
- **Input and output connections:** Receives completion path from `Loop Over Items`
- **Version-specific requirements:** TypeVersion `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.9 Block: Documentation Sticky Notes

### Overview
These nodes are visual in-canvas notes. They do not affect execution, but they contain critical operational context, setup requirements, and architectural descriptions.

### Nodes Involved
- `Sticky Note`
- `Sticky Note1`
- `Date Formatting and Relevance`
- `Sticky Note2`
- `Sticky Note3`
- `Sticky Note4`
- `Sticky Note5`

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Large overview note describing the full workflow purpose, setup, AI logic, and scheduling.
- **Input and output connections:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels source ingestion section.
- **Input and output connections:** None.

#### Date Formatting and Relevance
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels date filtering section.
- **Input and output connections:** None.

#### Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels deduplication section.
- **Input and output connections:** None.

#### Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels AI summarization section.
- **Input and output connections:** None.

#### Sticky Note4
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Critical operational note about rate limiting and Gemini error handling.
- **Input and output connections:** None.

#### Sticky Note5
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:** Labels Telegram delivery section.
- **Input and output connections:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedular | Schedule Trigger | Runs workflow every 3 hours |  | Input Feeds List | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Input Feeds List | Set | Defines RSS source URL array | Schedular | Fetch one url at a time | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Fetch one url at a time | Split Out | Splits URL array into one URL per item | Input Feeds List | Loop Over Feed URL's | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Loop Over Feed URL's | Split In Batches | Iterates feed URLs | Fetch one url at a time, Read Feed | Read Feed, No Operation, do nothing | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| No Operation, do nothing | No Op | Marks end of feed URL loop | Loop Over Feed URL's |  | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Read Feed | RSS Feed Read | Fetches feed entries from each URL | Loop Over Feed URL's | Extract Data, Loop Over Feed URL's | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Extract Data | Set | Normalizes feed item fields | Read Feed | Get Current Date | ## Section 1: Sources & ingestion<br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Get Current Date | Date & Time | Adds current date/time | Extract Data | Get Feed Published Date | ## Section 2: Date filtering<br>Normalize publication dates and compare with current date to keep only recent articles. |
| Get Feed Published Date | Date & Time | Formats article publication date | Get Current Date | Formatted Current Date | ## Section 2: Date filtering<br>Normalize publication dates and compare with current date to keep only recent articles. |
| Formatted Current Date | Date & Time | Formats current date for comparison | Get Feed Published Date | Filter by today's news | ## Section 2: Date filtering<br>Normalize publication dates and compare with current date to keep only recent articles. |
| Filter by today's news | Filter | Keeps only articles published today | Formatted Current Date | Remove Duplicates | ## Section 2: Date filtering<br>Normalize publication dates and compare with current date to keep only recent articles. |
| Remove Duplicates | Remove Duplicates | Removes duplicates within current execution | Filter by today's news | Remove Duplicates Comparing with Prev Exec | ## Section 3: Content Refining<br>Remove duplicate articles within the run and across previous executions using title + link. |
| Remove Duplicates Comparing with Prev Exec | Remove Duplicates | Removes items seen in previous executions | Remove Duplicates | Loop over filtered news | ## Section 3: Content Refining<br>Remove duplicate articles within the run and across previous executions using title + link. |
| Loop over filtered news | Split In Batches | Processes filtered articles in batches of 5 | Remove Duplicates Comparing with Prev Exec, Wait for 10 seconds | Wait for 10 seconds, No Operation, do nothing1 | ## CRITICAL: Rate Limiting & Credentials<br>### Wait Node: Do not remove the 10-second wait node; it is essential to prevent Telegram from blocking the bot during high-volume news cycles.<br>### AI Error Handling: The Gemini node is set to "Continue Regular Output" on error to allow the OpenAI fallback to function correctly. |
| No Operation, do nothing1 | No Op | Marks end of filtered article loop | Loop over filtered news |  | ## CRITICAL: Rate Limiting & Credentials<br>### Wait Node: Do not remove the 10-second wait node; it is essential to prevent Telegram from blocking the bot during high-volume news cycles.<br>### AI Error Handling: The Gemini node is set to "Continue Regular Output" on error to allow the OpenAI fallback to function correctly. |
| Wait for 10 seconds | Wait | Delays each item before AI processing | Loop over filtered news | Loop over filtered news, Gemini | ## CRITICAL: Rate Limiting & Credentials<br>### Wait Node: Do not remove the 10-second wait node; it is essential to prevent Telegram from blocking the bot during high-volume news cycles.<br>### AI Error Handling: The Gemini node is set to "Continue Regular Output" on error to allow the OpenAI fallback to function correctly. |
| Gemini | Google Gemini | Primary AI summarizer | Wait for 10 seconds | Set Gemini AI Summary | ## Section 4: AI summarization<br>Manages the dual-AI logic. If the primary Gemini summary is empty or errors out, the "If" node routes the task to OpenAI. |
| Set Gemini AI Summary | Set | Extracts Gemini text response into aiSummary | Gemini | Check AI Summary or error | ## Section 4: AI summarization<br>Manages the dual-AI logic. If the primary Gemini summary is empty or errors out, the "If" node routes the task to OpenAI. |
| Check AI Summary or error | If | Routes to fallback when Gemini fails or returns empty | Set Gemini AI Summary | OpenAI, Send News To Telegram Channel | ## Section 4: AI summarization<br>Manages the dual-AI logic. If the primary Gemini summary is empty or errors out, the "If" node routes the task to OpenAI. |
| OpenAI | OpenAI | Fallback AI summarizer | Check AI Summary or error | Set OpenAI summary | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |
| Set OpenAI summary | Set | Extracts OpenAI response into aiSummary | OpenAI | Send News To Telegram Channel1, Loop Over Items | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |
| Send News To Telegram Channel | Telegram | Sends Gemini-based summary to Telegram | Check AI Summary or error |  | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |
| Loop Over Items | Split In Batches | Additional batching loop for OpenAI branch | Set OpenAI summary, Send News To Telegram Channel1 | Send News To Telegram Channel1, No Operation, do nothing2 | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |
| Send News To Telegram Channel1 | Telegram | Sends OpenAI-based summary to Telegram | Set OpenAI summary, Loop Over Items | Loop Over Items | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |
| No Operation, do nothing2 | No Op | Marks end of OpenAI delivery loop | Loop Over Items |  | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |
| Sticky Note | Sticky Note | Visual workflow overview and setup guidance |  |  | ## QA News Intelligence Engine<br>This workflow automatically transforms raw QA RSS feeds into curated, AI-summarized updates sent to Telegram every 3 hours.<br><br>### How it works:<br><br>Collects news from multiple RSS sources (The Test Tribe, Medium, Cypress, and Dev.to).<br><br>Normalizes data and filters for specific keywords like "Playwright," "Quality," and "Testing."<br><br>Uses a two-step process to remove duplicates within the current execution and compares items against previous executions.<br><br>Processes news in batches of 5. It first attempts to use Gemini to create a 3-bullet technical summary.<br><br>If Gemini fails or returns an empty result, the workflow automatically switches to OpenAI (GPT-4o-mini) to generate the summary.<br><br>Sends the title, link, and AI summary to a Telegram group with a 10-second delay to manage rate limits.<br><br>### How to set up:<br><br>Credentials: Connect your Telegram Bot, Google Gemini (PaLM), and OpenAI API accounts.<br><br>Telegram: Enter your specific Chat ID in the Telegram nodes.<br><br>Scheduling: The trigger is set to 3 hours but can be adjusted in the "Schedule Trigger" node. |
| Sticky Note1 | Sticky Note | Visual note for source ingestion section |  |  | ## Section 1: Sources & ingestion<br><br>Collect RSS feeds, split URLs, and extract article data (title, link, snippet) for processing. |
| Date Formatting and Relevance | Sticky Note | Visual note for date filtering section |  |  | ## Section 2: Date filtering<br><br>Normalize publication dates and compare with current date to keep only recent articles. |
| Sticky Note2 | Sticky Note | Visual note for deduplication section |  |  | ## Section 3: Content Refining<br>Remove duplicate articles within the run and across previous executions using title + link. |
| Sticky Note3 | Sticky Note | Visual note for AI summarization section |  |  | ## Section 4: AI summarization<br>Manages the dual-AI logic. If the primary Gemini summary is empty or errors out, the "If" node routes the task to OpenAI. |
| Sticky Note4 | Sticky Note | Critical operational note |  |  | ## CRITICAL: Rate Limiting & Credentials<br><br>### Wait Node: Do not remove the 10-second wait node; it is essential to prevent Telegram from blocking the bot during high-volume news cycles.<br><br>### AI Error Handling: The Gemini node is set to "Continue Regular Output" on error to allow the OpenAI fallback to function correctly. |
| Sticky Note5 | Sticky Note | Visual note for delivery section |  |  | ## Section 5: Delivery<br>Batch process final articles and send summaries to Telegram with delay to avoid rate limits. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it `QA News Feed`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Name: `Schedular`
   - Configure it to run every 3 hours.

3. **Add a Set node for feed URLs**
   - Name: `Input Feeds List`
   - Create one field:
     - `urls` as an array
   - Set it to:
     - `https://www.thetesttribe.com/feed/`
     - `https://medium.com/feed/tag/qa-testing`
     - `https://www.cypress.io/feed.xml`
     - `https://dev.to/feed`
   - Connect `Schedular` → `Input Feeds List`.

4. **Add a Split Out node**
   - Name: `Fetch one url at a time`
   - Field to split out: `urls`
   - Connect `Input Feeds List` → `Fetch one url at a time`.

5. **Add a Split In Batches node**
   - Name: `Loop Over Feed URL's`
   - Leave default batch behavior
   - Connect `Fetch one url at a time` → `Loop Over Feed URL's`.

6. **Add a No Op node**
   - Name: `No Operation, do nothing`
   - Connect the loop completion output of `Loop Over Feed URL's` to this node.

7. **Add an RSS Feed Read node**
   - Name: `Read Feed`
   - URL: `{{ $json.urls }}`
   - Connect the main batch output of `Loop Over Feed URL's` → `Read Feed`.

8. **Close the feed URL loop**
   - Connect `Read Feed` back to `Loop Over Feed URL's` on its loop input.
   - This allows all feed URLs to be processed sequentially.

9. **Add a Set node to normalize feed data**
   - Name: `Extract Data`
   - Keep these fields:
     - `title = {{ $json.title }}`
     - `link = {{ $json.link }}`
     - `contentSnippet = {{ $json.contentSnippet }}`
     - `pubDate = {{ $json.isoDate }}`
   - Enable:
     - Dot notation
     - Ignore conversion errors
   - Connect `Read Feed` → `Extract Data`.

10. **Add a Date & Time node**
    - Name: `Get Current Date`
    - Include input fields: enabled
    - Connect `Extract Data` → `Get Current Date`.

11. **Add a Date & Time node for publication date formatting**
    - Name: `Get Feed Published Date`
    - Operation: `Format Date`
    - Input date: `{{ $json.pubDate }}`
    - Output field: `publishedDate`
    - Include input fields: enabled
    - Connect `Get Current Date` → `Get Feed Published Date`.

12. **Add another Date & Time node for current date formatting**
    - Name: `Formatted Current Date`
    - Operation: `Format Date`
    - Input date: `{{ $json.currentDate }}`
    - Output field: `currentDate`
    - Include input fields: enabled
    - Connect `Get Feed Published Date` → `Formatted Current Date`.

13. **Add a Filter node**
    - Name: `Filter by today's news`
    - Condition:
      - `publishedDate` equals `currentDate`
    - Use expressions:
      - Left: `{{ $json.publishedDate }}`
      - Right: `{{ $json.currentDate }}`
    - Connect `Formatted Current Date` → `Filter by today's news`.

14. **Add a Remove Duplicates node**
    - Name: `Remove Duplicates`
    - Keep default configuration
    - Connect `Filter by today's news` → `Remove Duplicates`.

15. **Add a second Remove Duplicates node**
    - Name: `Remove Duplicates Comparing with Prev Exec`
    - Operation: `Remove Items Seen In Previous Executions`
    - Dedupe value: `{{ $json.title }}{{ $json.link }}`
    - Connect `Remove Duplicates` → `Remove Duplicates Comparing with Prev Exec`.

16. **Add a Split In Batches node for article processing**
    - Name: `Loop over filtered news`
    - Batch size: `5`
    - Connect `Remove Duplicates Comparing with Prev Exec` → `Loop over filtered news`.

17. **Add another No Op node**
    - Name: `No Operation, do nothing1`
    - Connect the loop completion output of `Loop over filtered news` to this node.

18. **Add a Wait node**
    - Name: `Wait for 10 seconds`
    - Wait amount: `10` seconds
    - Connect the main output of `Loop over filtered news` → `Wait for 10 seconds`.

19. **Close the filtered article loop**
    - Connect `Wait for 10 seconds` back to `Loop over filtered news`.

20. **Add a Google Gemini node**
    - Name: `Gemini`
    - Model: `models/gemini-2.5-flash`
    - Temperature: `0.4`
    - Max output tokens: `900`
    - Set node error handling to continue regular output.
    - Prompt:
      - Summarize the article in 3 concise bullet points
      - Use:
        - `{{ $json.title }}`
        - `{{ $json.contentSnippet }}`
      - Request raw text only, no JSON, no code block formatting
    - Connect `Wait for 10 seconds` → `Gemini`.

21. **Configure Gemini credentials**
    - Add Google Gemini / PaLM API credentials in n8n.
    - Assign them to the `Gemini` node.

22. **Add a Set node to extract Gemini output**
    - Name: `Set Gemini AI Summary`
    - Field:
      - `aiSummary = {{ $json.content.parts[0].text }}`
    - Include other fields: enabled
    - Connect `Gemini` → `Set Gemini AI Summary`.

23. **Add an If node**
    - Name: `Check AI Summary or error`
    - Use OR logic
    - Condition 1:
      - `{{ $json.aiSummary }}` is empty
    - Condition 2:
      - `{{ $json.error }}` exists
    - Enable loose type validation
    - Connect `Set Gemini AI Summary` → `Check AI Summary or error`.

24. **Add a Telegram node for the Gemini-success branch**
    - Name: `Send News To Telegram Channel`
    - Operation: send message
    - Chat ID: replace `123456789` with your Telegram group or channel ID
    - Message text:
      - `Title : {{ $('Wait for 10 seconds').item.json.title }}`
      - `Link: {{ $('Wait for 10 seconds').item.json.link }}`
      - `Gist of the news: {{ $json.aiSummary }}`
    - Parse mode: `HTML`
    - Append attribution: disabled
    - Connect the false output of `Check AI Summary or error` → `Send News To Telegram Channel`.

25. **Configure Telegram credentials**
    - Create or connect Telegram Bot credentials in n8n.
    - Make sure the bot is a member of the target group/channel and has posting rights.

26. **Add an OpenAI node for fallback**
    - Name: `OpenAI`
    - Model: `gpt-4o-mini`
    - Max tokens: `500`
    - Add a system message:
      - `You are a senior QA automation engineer`
    - Add a user message requesting a 3-bullet QA summary.
    - Use expressions:
      - Title: `{{ $('Wait for 10 seconds').item.json.title }}`
      - Content currently configured as: `{{ $('Wait for 10 seconds').item.json.link }}`
    - Connect the true output of `Check AI Summary or error` → `OpenAI`.

27. **Configure OpenAI credentials**
    - Add OpenAI API credentials in n8n.
    - Assign them to the `OpenAI` node.

28. **Add a Set node to extract OpenAI output**
    - Name: `Set OpenAI summary`
    - Field:
      - `aiSummary = {{ $json.output[0].content[0].text }}`
    - Include other fields: enabled
    - Connect `OpenAI` → `Set OpenAI summary`.

29. **Add a second Telegram node for the fallback branch**
    - Name: `Send News To Telegram Channel1`
    - Chat ID: same Telegram target
    - Message text:
      - `Title : {{ $('Wait for 10 seconds').item.json.title }}`
      - `Link: {{ $('Wait for 10 seconds').item.json.link }}`
      - `Gist of the news: {{ $json.aiSummary }}`
    - Parse mode: `HTML`
    - Append attribution: disabled

30. **Add another Split In Batches node**
    - Name: `Loop Over Items`
    - Batch size: `3`

31. **Add a final No Op node**
    - Name: `No Operation, do nothing2`

32. **Wire the OpenAI delivery branch exactly as in the workflow**
    - Connect `Set OpenAI summary` → `Send News To Telegram Channel1`
    - Connect `Set OpenAI summary` → `Loop Over Items`
    - Connect `Loop Over Items` → `Send News To Telegram Channel1`
    - Connect `Send News To Telegram Channel1` → `Loop Over Items`
    - Connect the completion output of `Loop Over Items` → `No Operation, do nothing2`

33. **Add optional sticky notes**
    - Add notes for:
      - source ingestion
      - date filtering
      - deduplication
      - AI summarization
      - delivery
      - rate-limit warning
      - overall setup guidance

34. **Activate the workflow**
    - Turn the workflow on so the schedule trigger can execute.

35. **Validate with a test execution**
    - Confirm:
      - feeds are reachable
      - dates are being formatted correctly
      - only today’s posts are kept
      - duplicate filtering works across runs
      - Gemini summarizes normally
      - OpenAI fallback activates when Gemini output is empty or errors
      - Telegram messages are delivered successfully

## Reproduction Notes and Constraints

1. **Chat ID must be replaced**
   - `123456789` is a placeholder and will not work unless it matches your target.

2. **Gemini fallback logic depends on node error mode**
   - The Gemini node must stay configured to continue regular output on error.

3. **Do not remove the wait node**
   - The 10-second delay is intentionally used to reduce Telegram blocking risk.

4. **Execution-history deduplication requires workflow history**
   - If old executions are deleted or unavailable, previously sent news may be resent.

5. **Current OpenAI prompt uses article link instead of article body**
   - To improve fallback quality, you may want to replace the link expression with `contentSnippet`.

6. **Date equality is strict**
   - If your server timezone differs from the feed timezone, some same-day posts may be missed. Adjust formatting or comparison logic if needed.

7. **The workflow has one real entry point**
   - Only `Schedular` starts the workflow automatically.
   - There are no sub-workflows or Execute Workflow nodes in this design.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| QA News Intelligence Engine: transforms raw QA RSS feeds into curated, AI-summarized Telegram updates every 3 hours. | Workflow overview note |
| Setup requires Telegram Bot, Google Gemini (PaLM), and OpenAI credentials. | Environment preparation |
| Telegram nodes require a valid target Chat ID. | Telegram delivery configuration |
| The trigger interval is currently set to 3 hours and can be adjusted in the schedule node. | Scheduling behavior |
| Critical operational rule: do not remove the 10-second wait node because it helps prevent Telegram rate-limit issues. | Rate limiting |
| Gemini must remain configured with regular-output continuation on error so the OpenAI fallback works correctly. | AI fallback reliability |
| The sticky-note overview states the workflow filters for keywords like “Playwright,” “Quality,” and “Testing,” but no keyword filter node exists in the actual workflow JSON. | Important design discrepancy |
| The OpenAI fallback prompt labels the field as “Content” but actually passes the article link, not the snippet or body text. | Important implementation caveat |