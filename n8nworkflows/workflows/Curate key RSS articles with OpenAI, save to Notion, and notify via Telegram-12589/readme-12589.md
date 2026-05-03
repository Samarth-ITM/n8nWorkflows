Curate key RSS articles with OpenAI, save to Notion, and notify via Telegram

https://n8nworkflows.xyz/workflows/curate-key-rss-articles-with-openai--save-to-notion--and-notify-via-telegram-12589


# Curate key RSS articles with OpenAI, save to Notion, and notify via Telegram

## 1. Workflow Overview

**Workflow title:** Curate key RSS articles with OpenAI, save to Notion, and notify via Telegram  
**Internal workflow name (n8n):** Save key RSS articles to Notion and notify users via Telegram

**Purpose:**  
This workflow pulls articles from multiple RSS feeds, deduplicates them, skips items already stored in a Notion database, uses OpenAI (Chat Completions) to generate a structured analysis and priority score, then **stores only high-priority articles (≥60)** in Notion and **notifies a Telegram chat**. An optional cleanup branch can **archive Notion pages older than 30 days**.

### 1.1 Input Reception & Scheduling
- Entry point is a **Manual Trigger** (intended to be replaced with a Schedule Trigger for automation).

### 1.2 RSS Collection (Parallel)
- Reads three RSS feeds in parallel: TechCrunch, Dev.to, The Verge.
- Merges them into a single stream.

### 1.3 Deduplication & New-Item Detection (Notion-based)
- Removes duplicates across feeds (by `guid` or `link`).
- Pulls all existing Notion pages from the configured database.
- Filters out RSS items whose URL already exists in Notion.

### 1.4 AI Analysis (OpenAI)
- Sends each new article (title + short snippet) to OpenAI Chat Completions.
- Parses JSON-only analysis output and merges it with original article metadata.

### 1.5 Save & Notify
- Filters items where `priority_score >= 60`.
- Creates a Notion page for each qualifying article.
- Sends a Telegram notification with key fields and a link.

### 1.6 Optional Cleanup (Archive >30 days)
- Loads Notion pages, filters by “Added Date” older than 30 days, updates “Status” to “Archived”.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Fan-out
**Overview:** Starts workflow execution and launches both the RSS ingestion branch and the optional cleanup branch.  
**Nodes involved:** Manual Trigger

#### Node: Manual Trigger
- **Type / role:** `n8n-nodes-base.manualTrigger` — manual entry point for testing.
- **Configuration:** No parameters.
- **Connections:**
  - Outputs to: `RSS TechCrunch`, `RSS Dev.to`, `RSS The Verge`, `Get Old Articles (>30 days)`.
- **Edge cases / failures:**
  - None (local trigger).
- **Version notes:** TypeVersion 1.

---

### Block 2 — Collect RSS Feeds (Parallel)
**Overview:** Fetches RSS items from three sources independently; all outputs are later merged.  
**Nodes involved:** RSS TechCrunch, RSS Dev.to, RSS The Verge, Merge

#### Node: RSS TechCrunch
- **Type / role:** `n8n-nodes-base.rssFeedRead` — pulls RSS feed items.
- **Configuration choices:**
  - URL: `https://techcrunch.com/feed/`
  - `ignoreSSL: true` (tolerates SSL issues; can reduce security).
- **Connections:** Outputs to `Merge` (input index 0).
- **Failure types:**
  - Network/timeout, invalid RSS XML, upstream downtime.
  - SSL-related issues mitigated by `ignoreSSL: true`.
- **Version notes:** TypeVersion 1.2.

#### Node: RSS Dev.to
- **Type / role:** RSS reader.
- **Configuration:**
  - URL: `https://dev.to/feed`
  - `ignoreSSL: true`
- **Connections:** Outputs to `Merge` (input index 1).
- **Failure types:** Same as above.
- **Version notes:** TypeVersion 1.2.

#### Node: RSS The Verge
- **Type / role:** RSS reader.
- **Configuration:**
  - URL: `https://www.theverge.com/rss/index.xml`
  - `ignoreSSL: true`
- **Connections:** Outputs to `Merge` (input index 2).
- **Failure types:** Same as above.
- **Version notes:** TypeVersion 1.2.

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines multiple input streams into a single output.
- **Configuration choices:**
  - `numberInputs: 3` (expects three inbound connections).
- **Connections:**
  - Inputs: from the three RSS nodes.
  - Output: `Remove Duplicates`.
- **Edge cases / failures:**
  - If one RSS node fails, execution behavior depends on n8n run settings; some runs may stop before merge completes.
- **Version notes:** TypeVersion 3.2.

---

### Block 3 — Process & Deduplicate
**Overview:** Removes duplicate articles across all feeds using `guid` or `link` as a unique identifier.  
**Nodes involved:** Remove Duplicates

#### Node: Remove Duplicates
- **Type / role:** `n8n-nodes-base.code` — custom JS deduplication.
- **Configuration choices (interpreted):**
  - Iterates through all merged items (`items`).
  - Uses `identifier = item.json.guid || item.json.link`.
  - Keeps first occurrence; drops later duplicates using a `Set`.
  - Logs counts for debugging.
- **Connections:**
  - Input: `Merge`
  - Output: `Get Existing Articles`
- **Edge cases / failures:**
  - Items missing both `guid` and `link` will create `identifier = undefined`, causing accidental deduplication collisions (only one “undefined” retained).
  - Very large feeds could increase memory usage.
- **Version notes:** Code node TypeVersion 2.

---

### Block 4 — Filter New Content (Notion)
**Overview:** Loads existing Notion pages and filters RSS items so only URLs not already present in Notion continue to AI analysis.  
**Nodes involved:** Get Existing Articles, Filter New Articles

#### Node: Get Existing Articles
- **Type / role:** `n8n-nodes-base.notion` — reads all pages from a Notion database.
- **Configuration choices:**
  - Resource: Database Page
  - Operation: Get All
  - `returnAll: true`
  - Database ID placeholder: `YOUR_NOTION_DATABASE_ID`
  - `executeOnce: true` (runs once per workflow execution)
  - `alwaysOutputData: true` (helps downstream nodes run even if no results)
- **Connections:**
  - Input: `Remove Duplicates`
  - Output: `Filter New Articles`
- **Failure types:**
  - Notion auth/permission errors, wrong database ID, rate limits.
  - Schema mismatch doesn’t fail here, but impacts URL extraction downstream.
- **Version notes:** TypeVersion 2.2.

#### Node: Filter New Articles
- **Type / role:** `n8n-nodes-base.code` — compares RSS items to Notion URLs to keep only new ones.
- **Configuration choices (interpreted):**
  - Reads all Notion results: `$("Get Existing Articles").all()`
  - Extracts URLs from multiple possible shapes:
    - `item.json.properties?.URL?.url`
    - `item.json.properties?.URL?.rich_text?.[0]?.plain_text`
    - `item.json.property_url`
    - `item.json.url`
  - Reads deduped RSS items: `$("Remove Duplicates").all()`
  - For each RSS item: `articleUrl = item.json.guid || item.json.link`
  - Filters out if `existingUrls.includes(articleUrl)`
  - Returns `[]` if nothing new.
- **Connections:**
  - Input: `Get Existing Articles`
  - Output: `AI Content Analysis`
- **Edge cases / failures:**
  - If Notion property name isn’t exactly `URL`, extraction may fail → duplicates not detected.
  - Uses `includes()` on an array: O(n) lookup; for large Notion DBs, consider using a `Set` for performance.
  - Same `guid||link` issue as earlier: missing identifiers cause mis-filtering.
- **Version notes:** Code node TypeVersion 2.

---

### Block 5 — AI Analysis (OpenAI)
**Overview:** Sends each new item to OpenAI for structured JSON analysis, then parses and normalizes results into fields used by Notion and Telegram.  
**Nodes involved:** AI Content Analysis, Parse AI Analysis

#### Node: AI Content Analysis
- **Type / role:** `n8n-nodes-base.httpRequest` — calls OpenAI Chat Completions endpoint.
- **Configuration choices:**
  - Method: POST
  - URL: `https://api.openai.com/v1/chat/completions`
  - Authentication: `genericCredentialType` with `httpHeaderAuth` (expects an Authorization header, typically `Bearer <OpenAI_API_Key>`).
  - Body (JSON) built via expression:
    - `model: "gpt-4o-mini"`
    - System message instructs: “return ONLY valid JSON” with fields:
      - `summary`, `category`, `tags` (array), `sentiment`, `priority_score`
    - User message includes sanitized title and up to 500 chars of snippet/description.
    - `max_tokens: 300`, `temperature: 0.3`
- **Connections:**
  - Input: `Filter New Articles`
  - Output: `Parse AI Analysis`
- **Failure types / edge cases:**
  - Missing/invalid OpenAI credential (401).
  - Rate limits (429) and transient errors (5xx).
  - Model availability changes.
  - The prompt requests JSON-only, but the model may still return markdown or extra text; parsing node tries to handle that.
- **Version notes:** TypeVersion 4.2.

#### Node: Parse AI Analysis
- **Type / role:** `n8n-nodes-base.code` — parses AI response and merges with original article fields.
- **Configuration choices (interpreted):**
  - Loops over `items` (each item is an OpenAI response).
  - Extracts `choices[0].message.content`.
  - Attempts JSON extraction via regex:
    - ```json code block``` or first `{ ... }` match.
  - `JSON.parse()`, fallback object on parse errors:
    - summary: `AI analysis failed`, category: `Other`, tags: `['uncategorized']`, sentiment: `Neutral`, priority_score: `50`
  - Re-associates original article by index:
    - `const filterNewArticles = $("Filter New Articles").all();`
    - `const article = filterNewArticles[i].json;`
  - Produces normalized output fields:
    - `title`, `url`, `summary`, `category`, `tags` (comma-joined), `sentiment`, `priority_score` (int), `source`, `published_date`, `original_content`
- **Connections:**
  - Input: `AI Content Analysis`
  - Output: `Priority Filter (≥60)`
- **Edge cases / failures:**
  - **Index alignment risk:** assumes OpenAI results order/count exactly matches `Filter New Articles`. If an upstream node changes batching/ordering or partial failures occur, items can misalign.
  - `choices[0]...` missing will throw; caught only around parsing JSON, not around missing `choices` access (could still error before try/catch if structure differs).
  - `parseInt(aiAnalysis.priority_score) || 50` treats `0` as falsy → becomes `50` unintentionally. (If the AI returns `0`, it will be converted to 50.)
- **Version notes:** Code node TypeVersion 2.

---

### Block 6 — Save & Notify
**Overview:** Filters to high-priority items, writes them to Notion with mapped properties, then sends a Telegram alert.  
**Nodes involved:** Priority Filter (≥60), Save to Notion, Telegram Notification

#### Node: Priority Filter (≥60)
- **Type / role:** `n8n-nodes-base.if` — gates items by numeric priority.
- **Configuration choices:**
  - Condition: `$json.priority_score >= 60`
- **Connections:**
  - Input: `Parse AI Analysis`
  - True output: `Save to Notion`
  - (False output unused)
- **Edge cases / failures:**
  - If `priority_score` is string/non-numeric, comparison may behave unexpectedly; upstream tries to `parseInt`.
- **Version notes:** TypeVersion 1.

#### Node: Save to Notion
- **Type / role:** `n8n-nodes-base.notion` — creates a new Notion database page.
- **Configuration choices:**
  - Operation: Create database page
  - Database ID placeholder: `YOUR_NOTION_DATABASE_ID`
  - Properties mapping (Notion property names must exist and match types):
    - `Title` (rich_text) = `Parse AI Analysis.title`
    - `URL` (url) = `Parse AI Analysis.url`
    - `Summary` (rich_text) = `Parse AI Analysis.summary`
    - `Category` (select) = `Parse AI Analysis.category`
    - `Tags` (rich_text) = `Parse AI Analysis.tags`
    - `Sentiment` (select) = `$json.sentiment`
    - `Priority` (number) = `$json.priority_score`
    - `Source` (rich_text) = `$json.source`
    - `Published` (date) = `$json.published_date` (no time)
    - `Added Date` (date) = `$now`
    - `Status` (select) = `New`
- **Connections:**
  - Input: `Priority Filter (≥60)` (true branch)
  - Output: `Telegram Notification`
- **Failure types / edge cases:**
  - Notion select values must exist (e.g., `Category`, `Sentiment`, `Status` options) or Notion API may reject.
  - Date format issues if `published_date` is not ISO-like.
  - Database schema mismatch (property names/types).
- **Version notes:** TypeVersion 2.2.

#### Node: Telegram Notification
- **Type / role:** `n8n-nodes-base.telegram` — sends a formatted Telegram message.
- **Configuration choices:**
  - `chatId`: placeholder `YOUR_TELEGRAM_CHAT_ID`
  - Message uses Markdown parse mode and references `$('Parse AI Analysis').item.json.*`
  - `parse_mode: Markdown`
- **Connections:**
  - Input: `Save to Notion`
  - No downstream node.
- **Failure types / edge cases:**
  - Bot token / credential missing or invalid.
  - Chat ID wrong, bot not allowed in chat.
  - Markdown formatting breaks if content contains special characters; URLs generally OK.
- **Version notes:** TypeVersion 1.2.

---

### Block 7 — Cleanup (Optional): Archive Notion Pages Older Than 30 Days
**Overview:** Fetches all Notion pages, filters for pages with an “Added Date” older than 30 days, then updates their Status to “Archived”.  
**Nodes involved:** Get Old Articles (>30 days), Filter by Date (>30 days), Archive Old Articles

#### Node: Get Old Articles (>30 days)
- **Type / role:** Notion getAll (same database).
- **Configuration choices:**
  - Get all database pages (`returnAll: true`)
  - `executeOnce: true`, `alwaysOutputData: true`
  - Database ID placeholder: `YOUR_NOTION_DATABASE_ID`
- **Connections:**
  - Input: `Manual Trigger`
  - Output: `Filter by Date (>30 days)`
- **Notes:** Marked OPTIONAL in node notes.
- **Failure types:** Same as other Notion getAll.
- **Version notes:** TypeVersion 2.

#### Node: Filter by Date (>30 days)
- **Type / role:** Code node to select items older than 30 days.
- **Configuration choices (interpreted):**
  - Reads all pages: `$input.all()`
  - Computes `thirtyDaysAgo` = now minus 30 days
  - Extracts date from `item.json.property_added_date?.start`
  - Keeps items where extracted date exists and is older than cutoff
- **Connections:**
  - Input: `Get Old Articles (>30 days)`
  - Output: `Archive Old Articles`
- **Edge cases / failures:**
  - The field path `property_added_date.start` may not match actual Notion response. In many Notion responses, dates live under `properties["Added Date"].date.start`. If path is wrong, nothing will archive.
- **Version notes:** Code node TypeVersion 2.

#### Node: Archive Old Articles
- **Type / role:** Notion update database page (archives by Status).
- **Configuration choices:**
  - Page ID: `={{ $json.id }}`
  - Updates select property: `Status = Archived`
- **Connections:**
  - Input: `Filter by Date (>30 days)`
  - No downstream node.
- **Failure types / edge cases:**
  - `Status` select option `Archived` must exist.
  - Missing `id` in incoming items (if prior node shape differs).
- **Version notes:** TypeVersion 2.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Overview | Sticky Note | Documentation / overview | — | — | ## 📰 Save key RSS articles to Notion and notify users via Telegram<br><br>**Automatically collect, analyze, and organize RSS feed articles using AI**<br><br>### How it works<br><br>1. **Collect**: Fetches articles from multiple RSS feeds (TechCrunch, Dev.to, The Verge)<br>2. **Deduplicate**: Removes duplicate articles across feeds<br>3. **Filter**: Checks against existing Notion database to avoid re-processing<br>4. **Analyze**: Uses OpenAI to analyze content, extract insights, and assign priority scores<br>5. **Save**: Stores high-priority articles (≥60 score) in Notion database<br>6. **Notify**: Sends Telegram alerts for important articles<br>7. **Cleanup**: Optionally archives articles older than 30 days<br><br>### Setup<br><br>1. **Connect RSS Feeds**: Modify RSS nodes with your preferred feeds<br>2. **Configure Notion**: Set your Notion database ID in all Notion nodes<br>3. **Add OpenAI Key**: Configure OpenAI credentials for AI analysis<br>4. **Set Telegram Bot**: Add your Telegram chat ID for notifications<br>5. **Schedule**: Replace Manual Trigger with Schedule Trigger for automation<br><br>### Customization<br><br>• Adjust priority threshold in Priority Filter node (default: ≥60)<br>• Modify AI analysis prompt for different categorization<br>• Change retention period in cleanup section (default: 30 days)<br>• Add more RSS feeds by duplicating RSS nodes |
| Section 1 | Sticky Note | Section label | — | — | ## 1. Collect RSS Feeds<br><br>Fetches articles from configured RSS feeds in parallel |
| Section 2 | Sticky Note | Section label | — | — | ## 2. Process & Deduplicate<br><br>Merges all feeds and removes duplicate articles based on URL/GUID |
| Section 3 | Sticky Note | Section label | — | — | ## 3. Filter New Content<br><br>Compares against Notion database to find only new articles |
| Warning AI | Sticky Note | Credential warning | — | — | ## ⚠️ REQUIRED: OpenAI API Key<br><br>Add your OpenAI API credentials to this node before running the workflow |
| Section 4 | Sticky Note | Section label | — | — | ## 4. AI Analysis<br><br>Uses OpenAI to analyze content, extract summary, categorize, and assign priority scores |
| Section 5 | Sticky Note | Section label | — | — | ## 5. Save & Notify<br><br>Filters high-priority articles, saves to Notion, and sends Telegram notifications |
| Section 6 Optional | Sticky Note | Section label | — | — | ## 6. Cleanup (Optional)<br><br>Automatically archives articles older than 30 days. Enable these nodes if desired. |
| Manual Trigger | Manual Trigger | Workflow entry point | — | RSS TechCrunch; RSS Dev.to; RSS The Verge; Get Old Articles (>30 days) |  |
| RSS TechCrunch | RSS Feed Read | Fetch RSS items | Manual Trigger | Merge | ## 1. Collect RSS Feeds<br><br>Fetches articles from configured RSS feeds in parallel |
| RSS Dev.to | RSS Feed Read | Fetch RSS items | Manual Trigger | Merge | ## 1. Collect RSS Feeds<br><br>Fetches articles from configured RSS feeds in parallel |
| RSS The Verge | RSS Feed Read | Fetch RSS items | Manual Trigger | Merge | ## 1. Collect RSS Feeds<br><br>Fetches articles from configured RSS feeds in parallel |
| Merge | Merge | Combine feed items | RSS TechCrunch; RSS Dev.to; RSS The Verge | Remove Duplicates | ## 2. Process & Deduplicate<br><br>Merges all feeds and removes duplicate articles based on URL/GUID |
| Remove Duplicates | Code | Deduplicate by guid/link | Merge | Get Existing Articles | ## 2. Process & Deduplicate<br><br>Merges all feeds and removes duplicate articles based on URL/GUID |
| Get Existing Articles | Notion | Load existing pages (URL inventory) | Remove Duplicates | Filter New Articles | ## 3. Filter New Content<br><br>Compares against Notion database to find only new articles |
| Filter New Articles | Code | Exclude RSS items already in Notion | Get Existing Articles | AI Content Analysis | ## 3. Filter New Content<br><br>Compares against Notion database to find only new articles |
| AI Content Analysis | HTTP Request | Call OpenAI Chat Completions | Filter New Articles | Parse AI Analysis | ## 4. AI Analysis<br><br>Uses OpenAI to analyze content, extract summary, categorize, and assign priority scores |
| Parse AI Analysis | Code | Parse/normalize AI JSON + article fields | AI Content Analysis | Priority Filter (≥60) | ## 4. AI Analysis<br><br>Uses OpenAI to analyze content, extract summary, categorize, and assign priority scores |
| Priority Filter (≥60) | IF | Gate high-priority items | Parse AI Analysis | Save to Notion | ## 5. Save & Notify<br><br>Filters high-priority articles, saves to Notion, and sends Telegram notifications |
| Save to Notion | Notion | Create Notion page for article | Priority Filter (≥60) | Telegram Notification | ## 5. Save & Notify<br><br>Filters high-priority articles, saves to Notion, and sends Telegram notifications |
| Telegram Notification | Telegram | Send Telegram alert | Save to Notion | — | ## 5. Save & Notify<br><br>Filters high-priority articles, saves to Notion, and sends Telegram notifications |
| Get Old Articles (>30 days) | Notion | Load pages for cleanup | Manual Trigger | Filter by Date (>30 days) | ## 6. Cleanup (Optional)<br><br>Automatically archives articles older than 30 days. Enable these nodes if desired. |
| Filter by Date (>30 days) | Code | Select pages older than 30 days | Get Old Articles (>30 days) | Archive Old Articles | ## 6. Cleanup (Optional)<br><br>Automatically archives articles older than 30 days. Enable these nodes if desired. |
| Archive Old Articles | Notion | Update Status = Archived | Filter by Date (>30 days) | — | ## 6. Cleanup (Optional)<br><br>Automatically archives articles older than 30 days. Enable these nodes if desired. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: “Save key RSS articles to Notion and notify users via Telegram” (or your preferred name).
   - (Optional) Add sticky notes to mirror the sections and guidance.

2. **Add the trigger**
   - Add node: **Manual Trigger**
   - Later, for automation, replace/add **Schedule Trigger** (e.g., every hour).

3. **Add RSS feed readers (3 nodes)**
   - Add **RSS Feed Read** → name: `RSS TechCrunch`
     - URL: `https://techcrunch.com/feed/`
     - Options: enable **Ignore SSL Issues**
   - Add **RSS Feed Read** → name: `RSS Dev.to`
     - URL: `https://dev.to/feed`
     - Options: Ignore SSL Issues
   - Add **RSS Feed Read** → name: `RSS The Verge`
     - URL: `https://www.theverge.com/rss/index.xml`
     - Options: Ignore SSL Issues
   - Connect **Manual Trigger → each RSS node** (three separate connections).

4. **Merge RSS outputs**
   - Add node: **Merge** → name: `Merge`
   - Set **Number of inputs = 3**
   - Connect:
     - `RSS TechCrunch → Merge (Input 1)`
     - `RSS Dev.to → Merge (Input 2)`
     - `RSS The Verge → Merge (Input 3)`

5. **Deduplicate**
   - Add node: **Code** → name: `Remove Duplicates`
   - Paste logic that:
     - Uses a `Set` keyed by `guid || link`
     - Returns only unique items
   - Connect: `Merge → Remove Duplicates`

6. **Configure Notion (existing-article check)**
   - Create Notion credentials in n8n (**Notion API integration**), ensure the integration has access to your database.
   - Add node: **Notion** → name: `Get Existing Articles`
     - Resource: **Database Page**
     - Operation: **Get All**
     - Return All: **true**
     - Database ID: select/paste your database ID
     - (Optional) enable: “Always Output Data”
   - Connect: `Remove Duplicates → Get Existing Articles`

7. **Filter to only new RSS items**
   - Add node: **Code** → name: `Filter New Articles`
   - Implement:
     - Load all Notion pages from `Get Existing Articles`
     - Extract existing URL values (ensure property name matches your database, typically `URL`)
     - Filter RSS items from `Remove Duplicates` where `guid||link` not in existing URLs
   - Connect: `Get Existing Articles → Filter New Articles`

8. **OpenAI analysis (HTTP Request)**
   - Create credentials for OpenAI:
     - Use **HTTP Header Auth** (or equivalent) with header: `Authorization: Bearer <OPENAI_API_KEY>`
   - Add node: **HTTP Request** → name: `AI Content Analysis`
     - Method: POST
     - URL: `https://api.openai.com/v1/chat/completions`
     - Authentication: **Header Auth** (generic)
     - Send Body: **JSON**
     - JSON body should include:
       - model: `gpt-4o-mini`
       - messages: system prompt requesting JSON-only fields, plus user prompt embedding title and snippet
       - max_tokens: `300`, temperature: `0.3`
   - Connect: `Filter New Articles → AI Content Analysis`

9. **Parse AI output + normalize fields**
   - Add node: **Code** → name: `Parse AI Analysis`
   - Implement:
     - Extract `choices[0].message.content`
     - Strip ```json``` blocks if present
     - `JSON.parse` with fallback defaults
     - Merge with original article fields (title/url/source/published/snippet)
     - Output a flat structure containing: `title, url, summary, category, tags, sentiment, priority_score, source, published_date, original_content`
   - Connect: `AI Content Analysis → Parse AI Analysis`

10. **Filter high priority**
    - Add node: **IF** → name: `Priority Filter (≥60)`
    - Condition: **Number** → `{{$json.priority_score}}` **largerEqual** `60`
    - Connect: `Parse AI Analysis → Priority Filter (≥60)`

11. **Save to Notion (create page)**
    - Add node: **Notion** → name: `Save to Notion`
      - Resource: Database Page
      - Operation: Create
      - Database ID: your database
      - Map properties (must exist in Notion with matching types):
        - Title (rich text)
        - URL (url)
        - Summary (rich text)
        - Category (select)
        - Tags (rich text)
        - Sentiment (select)
        - Priority (number)
        - Source (rich text)
        - Published (date)
        - Added Date (date)
        - Status (select)
    - Connect: `Priority Filter (≥60) [true] → Save to Notion`

12. **Telegram notification**
    - Create Telegram credentials by connecting your bot token in n8n’s Telegram node.
    - Add node: **Telegram** → name: `Telegram Notification`
      - Operation: Send Message
      - Chat ID: your target chat ID
      - Parse mode: Markdown
      - Message: include title, priority, category, sentiment, tags, summary, and URL (based on parsed fields)
    - Connect: `Save to Notion → Telegram Notification`

13. **Optional cleanup branch (archive >30 days)**
    - From **Manual Trigger**, add a second branch:
      - **Notion** node: `Get Old Articles (>30 days)` (Database Page → Get All → same database)
      - **Code** node: `Filter by Date (>30 days)`:
        - Compare the Notion “Added Date” to now-30 days
        - Output only old items
      - **Notion** node: `Archive Old Articles`:
        - Operation: Update page
        - Page ID: `{{$json.id}}`
        - Set `Status` select to `Archived`
    - Connect:
      - `Manual Trigger → Get Old Articles (>30 days) → Filter by Date (>30 days) → Archive Old Articles`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace Manual Trigger with Schedule Trigger for automation | Mentioned in Main Overview sticky note |
| OpenAI API key required for “AI Content Analysis” node | Warning AI sticky note |
| Notion database ID must be set in all Notion nodes | Mentioned in Main Overview sticky note |
| Priority threshold default is ≥60 (change in IF node) | Mentioned in Main Overview sticky note |
| Cleanup archives items older than 30 days; optional branch | Mentioned in Main Overview and Section 6 Optional sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.