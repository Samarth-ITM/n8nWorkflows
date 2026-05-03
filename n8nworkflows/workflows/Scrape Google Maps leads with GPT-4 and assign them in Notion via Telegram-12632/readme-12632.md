Scrape Google Maps leads with GPT-4 and assign them in Notion via Telegram

https://n8nworkflows.xyz/workflows/scrape-google-maps-leads-with-gpt-4-and-assign-them-in-notion-via-telegram-12632


# Scrape Google Maps leads with GPT-4 and assign them in Notion via Telegram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Scrape Google Maps leads with GPT-4 and assign them in Notion via Telegram

**Purpose:**  
This workflow automates lead generation from Google Maps, enriches each lead with an AI-written “icebreaker” based on reviews, stores the lead in a Notion CRM, and notifies a Telegram sales group with a “⚡️ Take Lead” button. When an agent clicks the button, the workflow assigns the lead to that agent in Notion and updates the Telegram message to show who claimed it.

**Primary use cases**
- Local business prospecting (restaurants, salons, clinics, etc.) from Google Maps
- Team-based lead claiming from a shared Telegram channel/group
- Centralized CRM tracking and assignment in Notion

### Logical blocks
**1.1 Fetch & Prepare Leads (Outscraper → Normalize)**  
Manual run triggers an Outscraper Google Maps search; results are cleaned/normalized and review snippets are extracted.

**1.2 Deduplicate & Enrich (Notion check → GPT analysis)**  
Each lead is checked against the Notion “Active Deals” database. New leads are enriched by GPT with a targeted pitch based on review context.

**1.3 Save & Notify (Notion create → Telegram card)**  
New enriched leads are created in Notion and sent to Telegram as formatted HTML messages with an inline “Take Lead” button.

**1.4 Lead Claim Handling (Telegram callback → Notion assignment → message update)**  
A Telegram callback trigger captures button clicks, matches the clicking user to a Notion “Agents” database, updates the lead’s “Assigned Manager”, edits the original message, and answers the callback (toast).

---

## 2. Block-by-Block Analysis

### 2.1 Fetch & Prepare Leads (Step 1)
**Overview:**  
Fetches businesses from Google Maps via Outscraper using a configurable query, then cleans key fields (phone, website, email) and extracts a short review context for downstream AI prompting.

**Nodes involved:**
- Manual Trigger
- 📝 CONFIGURATION
- 1. Fetch Google Maps Data
- 2. Clean & Normalize

#### Node: Manual Trigger
- **Type / role:** Manual Trigger; entry point for the scraping pipeline.
- **Configuration:** No parameters.
- **Inputs / outputs:**  
  - Output → 📝 CONFIGURATION
- **Edge cases:** None (manual start only).

#### Node: 📝 CONFIGURATION (Set)
- **Type / role:** Set node; central configuration for query, limits, and integration values.
- **Key configuration:**
  - `LIMIT_RESULTS` (number): default `5`
  - `SEARCH_QUERY` (string): default `"Restaurants in Tbilisi"`
  - `OUTSCRAPER_API_KEY` (string): must be filled
  - `TELEGRAM_CHAT_ID` (string): must be filled (target group/chat)
  - `AI_TONE` (string): guidance for AI style
  - `MY_SERVICES` (string): services list the AI should pitch
- **Expressions/variables used:** referenced downstream via `$('📝 CONFIGURATION').first().json...`
- **Inputs / outputs:**  
  - Input ← Manual Trigger  
  - Output → 1. Fetch Google Maps Data
- **Edge cases / failures:**
  - Missing `OUTSCRAPER_API_KEY` → Outscraper request will fail (401/403).
  - Missing `TELEGRAM_CHAT_ID` → Telegram send will fail later.

#### Node: 1. Fetch Google Maps Data (HTTP Request)
- **Type / role:** HTTP Request; calls Outscraper Maps Search API v2.
- **Configuration choices:**
  - URL: `https://api.app.outscraper.com/maps/search-v2`
  - Query params (expressions from config):
    - `query = {{$json.SEARCH_QUERY}}`
    - `limit = {{$json.LIMIT_RESULTS}}`
    - `drop_duplicates = true`
    - `async = false`
    - `reviewsLimit = 1`
    - `reviewsSort = most_relevant`
  - Header:
    - `X-API-KEY = {{$json.OUTSCRAPER_API_KEY}}`
- **Inputs / outputs:**  
  - Input ← 📝 CONFIGURATION  
  - Output → 2. Clean & Normalize
- **Version notes:** node typeVersion `4.3` (n8n HTTP Request).
- **Edge cases / failures:**
  - API key invalid/rate limits → non-2xx response.
  - Response shape variations (array vs nested arrays/objects) are handled later in code, but extreme deviations could still break normalization.
  - If `reviewsLimit=1`, downstream review context may be minimal.

#### Node: 2. Clean & Normalize (Code)
- **Type / role:** Code node; transforms Outscraper output into normalized lead items.
- **Main transformations:**
  - Phone normalization: `clean_phone = phone.replace(/[^0-9]/g,'')`
  - Website normalization: uses `site` or `website`, trimmed; empty → `null`
  - Email normalization: trimmed; empty → `null`
  - Ensures `name` (fallback “Unnamed Lead”), `reviews` numeric fallback `0`
  - Review context building from `reviews_data`:
    - Keeps entries with `review_text.length > 5`
    - Formats as `(rating★) "text"`
    - Takes up to 10 lines
    - Fallback: `"No text reviews."`
  - Filters items with `entry.reviews < MAX_REVIEWS` (MAX_REVIEWS=50000) — effectively keeps most leads.
- **Input handling:** attempts to parse `item.json.data` or `item.json` and supports:
  - arrays
  - nested arrays (flattens)
  - object maps with entries containing `name` or `place_id`
- **Outputs:** returns `newItems` list (each item: `{ json: entry }`).
- **Inputs / outputs:**  
  - Input ← 1. Fetch Google Maps Data  
  - Output → 🔄 Loop Items
- **Edge cases / failures:**
  - If Outscraper returns an unexpected structure, some leads may be dropped.
  - `clean_phone` could become empty string; later duplicate search uses it (may cause false matches or no matches depending on Notion filter behavior).
  - `website` might be null, but later Telegram message still builds a link.

---

### 2.2 CRM Check & AI Analysis (Step 2)
**Overview:**  
Iterates through leads one-by-one, searches the Notion CRM for duplicates (by phone), and only for new leads calls GPT to generate a short targeted pitch (“icebreaker”).

**Nodes involved:**
- 🔄 Loop Items
- 🔎 Search Duplicate
- 🛡️ Restore Data
- Found?
- 🚫 Duplicate / Skip
- 🤖 AI Icebreaker
- 🔗 Merge Final

#### Node: 🔄 Loop Items (Split In Batches)
- **Type / role:** SplitInBatches; processes leads sequentially and loops until exhausted.
- **Configuration:** default options (batch size not explicitly set; n8n default applies).
- **Inputs / outputs:**  
  - Input ← 2. Clean & Normalize  
  - Output (loop/next batch) → 🔎 Search Duplicate and 🛡️ Restore Data (see connections)
  - Receives loop-back from:
    - 🚫 Duplicate / Skip
    - 📢 Notify Team
- **Edge cases / failures:**
  - If no items, downstream nodes won’t run.
  - Ensure loop-back paths always return to this node to continue; this workflow does so for both duplicate and successful notify paths.

#### Node: 🔎 Search Duplicate (Notion - getAll databasePage)
- **Type / role:** Notion node; checks whether lead already exists in “Active Deals (CRM)” by phone number.
- **Configuration choices:**
  - Resource: Database Page
  - Operation: Get All
  - Database: “Active Deals (CRM)” (selected via UI list; databaseId value not embedded here)
  - Filter (manual):
    - Property `Phone|phone_number` **equals** `{{$json.clean_phone}}`
  - `alwaysOutputData: true` (important for IF handling)
- **Inputs / outputs:**  
  - Input ← 🔄 Loop Items  
  - Output → 🛡️ Restore Data (input 2)
- **Edge cases / failures:**
  - Notion credentials missing/expired → auth errors.
  - If `clean_phone` is empty, the filter may match nothing (best case) or behave unexpectedly depending on Notion API behavior.
  - If the Notion database schema differs (property names/types), filtering fails.

#### Node: 🛡️ Restore Data (Merge - combineByPosition)
- **Type / role:** Merge; recombines the original lead item with the Notion search result item by position.
- **Why it exists:** Notion “Get All” output differs from the original lead payload; this node restores the original lead data alongside duplicate-check results.
- **Configuration:** Mode `combine` + `combineByPosition`.
- **Inputs / outputs:**  
  - Input 1 ← 🔄 Loop Items (original lead)  
  - Input 2 ← 🔎 Search Duplicate (search result)  
  - Output → Found?
- **Edge cases / failures:**
  - If batch sizes/positions mismatch (rare in this pattern), wrong items could merge. The current flow is 1 lead → 1 search call, so it should align.

#### Node: Found? (IF)
- **Type / role:** IF; decides if lead is duplicate.
- **Condition:** boolean check:
  - `value1 = {{ !!$json.id }}` equals `true`
  - Interprets presence of an `id` on the merged JSON as “duplicate found”.
- **Inputs / outputs:**  
  - Input ← 🛡️ Restore Data  
  - **True** → 🚫 Duplicate / Skip  
  - **False** → 🤖 AI Icebreaker (and also sends a second connection to 🔗 Merge Final input 2)
- **Edge cases / failures:**
  - Potential ambiguity: `id` could come from the lead itself if the lead payload contains an `id` field. This workflow stores lead identifier in `place_id`, so typically safe—but if Outscraper returns `id`, it could cause false duplicates.
  - Better practice would be checking `{{$items('🔎 Search Duplicate').length > 0}}` or similar.

#### Node: 🚫 Duplicate / Skip (NoOp)
- **Type / role:** NoOp; explicit branch end for duplicates.
- **Inputs / outputs:**  
  - Input ← Found? (true)  
  - Output → 🔄 Loop Items (to continue loop)
- **Edge cases:** None.

#### Node: 🤖 AI Icebreaker (@n8n/n8n-nodes-langchain.openAi)
- **Type / role:** OpenAI (LangChain) chat node; generates a personalized “Context/Problem -> Specific Solution” pitch.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Messages: single prompt assembling:
    - `AI_TONE` from config
    - `MY_SERVICES` from config
    - Lead fields (name, city, rating, reviews)
    - `reviews_context`
    - Task + output format constraints
- **Key expressions:**
  - `{{ $('📝 CONFIGURATION').first().json.AI_TONE }}`
  - `{{ $('📝 CONFIGURATION').first().json.MY_SERVICES }}`
  - `{{ $json.reviews_context }}`
- **Inputs / outputs:**  
  - Input ← Found? (false)  
  - Output → 🔗 Merge Final (input 1)
- **Version notes:** node typeVersion `1` for this LangChain OpenAI node; ensure your n8n instance includes the LangChain nodes package.
- **Edge cases / failures:**
  - Missing OpenAI credentials, quota exceeded, model unavailable.
  - If `reviews_context` is “No text reviews.” the output may be generic.
  - Prompt injection risk is low but possible if reviews contain malicious text; consider stricter system messages if needed.

#### Node: 🔗 Merge Final (Merge - combineByPosition)
- **Type / role:** Merge; combines original lead data with AI output (so later nodes can access both lead fields and `message.content`).
- **Configuration:** Mode `combine` + `combineByPosition`.
- **Inputs / outputs:**  
  - Input 1 ← 🤖 AI Icebreaker (AI message output)  
  - Input 2 ← Found? (false branch sends the original lead forward into this merge)  
  - Output → 💾 Create New Lead
- **Edge cases / failures:**
  - If either branch produces no item, merge may output nothing.
  - Position mismatch is unlikely because both are produced from the same IF false path per lead.

---

### 2.3 Save to Notion & Alert Telegram (Step 3)
**Overview:**  
Creates a new page in Notion “Active Deals” with enriched fields, then formats and sends a Telegram message containing lead details and a “Take Lead” callback button. Loops to process the next lead.

**Nodes involved:**
- 💾 Create New Lead
- 🎨 Prepare Message
- 📢 Notify Team

#### Node: 💾 Create New Lead (Notion - create databasePage)
- **Type / role:** Notion node; inserts lead into CRM.
- **Configuration choices:**
  - Resource: Database Page
  - Operation: Create
  - Database: “Active Deals (CRM)”
  - Properties set:
    - `Company Name|title` = lead name
    - `Phone|phone_number` = `clean_phone`
    - `Email|email` = `email`
    - `Website / Social|url` = `website`
    - `Address|rich_text` = `full_address || ''`
    - `Lead ID|rich_text` = `place_id || ''`
    - `Lead Source|select` = `Cold Outreach`
    - `Pipeline Stage|status` = `Cold Outreach`
    - `Icebreaker|rich_text` = `{{$json.message.content}}` (from AI node after merge)
- **Inputs / outputs:**  
  - Input ← 🔗 Merge Final  
  - Output → 🎨 Prepare Message
- **Edge cases / failures:**
  - Database schema mismatch (property names differ) will cause Notion API errors.
  - `clean_phone` empty may violate your CRM expectations.
  - If AI output path failed and `message.content` missing, expression errors can occur.

#### Node: 🎨 Prepare Message (Code)
- **Type / role:** Code node; builds HTML Telegram message + embeds Notion page ID for callback.
- **Key logic:**
  - Random intro line from 4 options.
  - Reads:
    - `sourceData = $('🔗 Merge Final').item.json`
    - `notionId` and `notionUrl` from `$('💾 Create New Lead').item.json`
  - Escapes HTML for name and AI text.
  - Creates message containing:
    - city, company name, rating/reviews
    - strategy (AI icebreaker)
    - phone in `<code>`
    - links to website and maps (`link || location_link`)
    - link to Notion page
  - Outputs:
    - `telegram_message`
    - `lead_id` = Notion page id
- **Inputs / outputs:**  
  - Input ← 💾 Create New Lead  
  - Output → 📢 Notify Team
- **Edge cases / failures:**
  - If `sourceData.website` is null/empty, `<a href="null">Site</a>` can be malformed; consider conditionally hiding.
  - If `city`, `rating`, or `link` fields are missing from Outscraper, message may contain “undefined”.
  - Telegram HTML is strict; malformed tags/URLs can cause send failures.

#### Node: 📢 Notify Team (Telegram - sendMessage)
- **Type / role:** Telegram node; posts the lead card to a group/chat and attaches an inline keyboard for claiming.
- **Configuration choices:**
  - Chat ID: `{{ $('📝 CONFIGURATION').first().json.TELEGRAM_CHAT_ID }}`
  - Text: `{{ $json.telegram_message }}`
  - Parse mode: HTML
  - Inline keyboard: one button
    - Text: `⚡️ Take Lead`
    - `callback_data = {{ "take_lead_" + $json.lead_id }}`
  - `appendAttribution: false`
- **Inputs / outputs:**  
  - Input ← 🎨 Prepare Message  
  - Output → 🔄 Loop Items (to continue processing next lead)
- **Edge cases / failures:**
  - Bot not in group or lacking permission → send fails.
  - Wrong chat ID → send fails.
  - Telegram limits: message length, HTML parsing errors, invalid URLs.
  - Callback data length limit (Telegram ~64 bytes): Notion page IDs are usually safe, but keep in mind if format changes.

---

### 2.4 Handle Button Clicks (Assignment) (Step 4)
**Overview:**  
Listens for Telegram callback queries, identifies the clicking agent in Notion using their Telegram ID, assigns the corresponding Notion user to the lead, edits the original Telegram message to show who took it, and returns a callback toast.

**Nodes involved:**
- Telegram Callback
- 🕵️‍♂️ Find Agent
- Agent Exists?
- 📝 Assign Lead
- ✅ Update Chat
- 🔔 Show Toast
- ❌ Error Toast

#### Node: Telegram Callback (Telegram Trigger)
- **Type / role:** Telegram Trigger; second entry point for callback button clicks.
- **Configuration:**
  - Updates: `callback_query`
- **Inputs / outputs:**  
  - Output → 🕵️‍♂️ Find Agent
- **Edge cases / failures:**
  - Webhook not set / Telegram trigger not active in production.
  - If callback payload structure differs, downstream expressions must handle it (this workflow references `callback_query`).

#### Node: 🕵️‍♂️ Find Agent (Notion - getAll databasePage)
- **Type / role:** Notion node; finds agent configuration record by Telegram ID.
- **Configuration choices:**
  - Database: “Agents Configuration”
  - Operation: Get All (return all)
  - Filter:
    - `Telegram ID|number` equals `parseInt($json.from?.id || $json.callback_query?.from?.id)`
- **Inputs / outputs:**  
  - Input ← Telegram Callback  
  - Output → Agent Exists?
- **Edge cases / failures:**
  - Filter assumes Telegram ID is stored as a Notion number property.
  - If multiple agent records match, this node returns multiple items; downstream uses `$('🕵️‍♂️ Find Agent').item...` which may behave unexpectedly if more than one item exists.
  - Credentials/schema mismatch.

#### Node: Agent Exists? (IF)
- **Type / role:** IF; validates that the found agent record includes a Notion user in a People property.
- **Condition:**
  - `{{ ($json.properties['Notion User']?.people?.length || 0) > 0 }}` equals `true`
- **Inputs / outputs:**  
  - True → 📝 Assign Lead  
  - False → ❌ Error Toast
- **Edge cases / failures:**
  - If Notion property name differs (“Notion User”), condition fails.
  - If multiple items from Find Agent, IF runs per item; you may need to enforce single record.

#### Node: 📝 Assign Lead (Notion - update databasePage)
- **Type / role:** Notion node; assigns the lead page to the agent.
- **Configuration choices:**
  - Page ID: extracted from callback data:
    - `{{ $('Telegram Callback').item.json.callback_query.data.replace('take_lead_', '') }}`
  - Operation: Update databasePage
  - Sets:
    - `Assigned Manager|people` to list of people IDs:
      ```js
      ($('🕵️‍♂️ Find Agent').item.json.properties['Notion User'].people || []).map(u => u.id)
      ```
- **Inputs / outputs:**  
  - Input ← Agent Exists? (true)  
  - Output → ✅ Update Chat
- **Edge cases / failures:**
  - If callback_data is tampered or missing prefix, pageId becomes invalid → Notion update fails.
  - If “Assigned Manager” property doesn’t exist or isn’t People type, update fails.

#### Node: ✅ Update Chat (Telegram - editMessageText)
- **Type / role:** Telegram node; edits the original lead card message to show who took it.
- **Configuration choices:**
  - Operation: `editMessageText`
  - Chat ID: `{{ $('Telegram Callback').item.json.callback_query.message.chat.id }}`
  - Message ID: `{{ $('Telegram Callback').item.json.callback_query.message.message_id }}`
  - New text:
    - Original message text: `{{ $('Telegram Callback').item.json.message.text }}`
    - Adds: `✅ <b>Taken by:</b> {{ agent name }}`
  - Parse mode: HTML
  - Agent name from Notion title:
    - `{{ $('🕵️‍♂️ Find Agent').item.json.properties['Name'].title[0].plain_text }}`
- **Inputs / outputs:**  
  - Input ← 📝 Assign Lead  
  - Output → 🔔 Show Toast
- **Edge cases / failures:**
  - `$('Telegram Callback').item.json.message.text` may not exist; typically callback queries provide `callback_query.message.text`. If missing, edit may set blank/wrong text.
  - If agent Name title is empty, expression `[0]` may fail.
  - Telegram edit restrictions: old messages, missing rights, or message not editable.

#### Node: 🔔 Show Toast (Telegram - answerCallbackQuery)
- **Type / role:** Telegram node; answers callback to remove loading state in Telegram UI.
- **Configuration:** Operation `answerCallbackQuery` (no custom text set here).
- **Inputs / outputs:**  
  - Input ← ✅ Update Chat  
  - Output: none
- **Edge cases:** If not answered quickly, Telegram may show timeout; best practice is always answering even on error (this workflow does via ❌ Error Toast).

#### Node: ❌ Error Toast (Telegram - answerCallbackQuery)
- **Type / role:** Telegram node; answers callback when agent is not configured/found.
- **Configuration:** Operation `answerCallbackQuery` (no custom error text configured).
- **Inputs / outputs:**  
  - Input ← Agent Exists? (false)  
  - Output: none
- **Edge cases:** Since no text is provided, user may not understand the failure; consider setting an error message.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Sticky | Sticky Note | Documentation / overview | — | — | # AI Sales Coach System (Lead Gen)… (overall explanation + setup steps) |
| Sticky Note 1 | Sticky Note | Documentation for step 1 | — | — | ## Step 1: Fetch & Prepare Data… |
| Sticky Note 2 | Sticky Note | Documentation for step 2 | — | — | ## Step 2: CRM Check & AI Analysis… |
| Sticky Note 3 | Sticky Note | Documentation for step 3 | — | — | ## Step 3: Save to Notion & Alert Telegram… |
| Sticky Note 4 | Sticky Note | Documentation for step 4 | — | — | ## Step 4: Handle Button Clicks (Assignment)… |
| Manual Trigger | Manual Trigger | Entry point for scraping pipeline | — | 📝 CONFIGURATION | ## Step 1: Fetch & Prepare Data… |
| 📝 CONFIGURATION | Set | Central parameters (query, keys, tone) | Manual Trigger | 1. Fetch Google Maps Data | ## Step 1: Fetch & Prepare Data… |
| 1. Fetch Google Maps Data | HTTP Request | Calls Outscraper Maps Search API | 📝 CONFIGURATION | 2. Clean & Normalize | ## Step 1: Fetch & Prepare Data… |
| 2. Clean & Normalize | Code | Normalizes fields + extracts review context | 1. Fetch Google Maps Data | 🔄 Loop Items | ## Step 1: Fetch & Prepare Data… |
| 🔄 Loop Items | Split In Batches | Iterates through leads + loop control | 2. Clean & Normalize; 🚫 Duplicate / Skip; 📢 Notify Team | 🔎 Search Duplicate; 🛡️ Restore Data | ## Step 2: CRM Check & AI Analysis… |
| 🔎 Search Duplicate | Notion | Deduplicate by phone in Active Deals | 🔄 Loop Items | 🛡️ Restore Data | ## Step 2: CRM Check & AI Analysis… |
| 🛡️ Restore Data | Merge | Re-attach original lead to Notion result | 🔄 Loop Items; 🔎 Search Duplicate | Found? | ## Step 2: CRM Check & AI Analysis… |
| Found? | IF | Branch: duplicate vs new lead | 🛡️ Restore Data | 🚫 Duplicate / Skip; 🤖 AI Icebreaker; 🔗 Merge Final | ## Step 2: CRM Check & AI Analysis… |
| 🚫 Duplicate / Skip | NoOp | Skips duplicates and continues loop | Found? | 🔄 Loop Items | ## Step 2: CRM Check & AI Analysis… |
| 🤖 AI Icebreaker | OpenAI (LangChain) | Generates personalized pitch from reviews | Found? (false) | 🔗 Merge Final | ## Step 2: CRM Check & AI Analysis… |
| 🔗 Merge Final | Merge | Combines lead + AI output | 🤖 AI Icebreaker; Found? (false) | 💾 Create New Lead | ## Step 3: Save to Notion & Alert Telegram… |
| 💾 Create New Lead | Notion | Creates new CRM entry in Active Deals | 🔗 Merge Final | 🎨 Prepare Message | ## Step 3: Save to Notion & Alert Telegram… |
| 🎨 Prepare Message | Code | Builds HTML Telegram card + callback payload | 💾 Create New Lead | 📢 Notify Team | ## Step 3: Save to Notion & Alert Telegram… |
| 📢 Notify Team | Telegram | Sends lead card with “Take Lead” button | 🎨 Prepare Message | 🔄 Loop Items | ## Step 3: Save to Notion & Alert Telegram… |
| Telegram Callback | Telegram Trigger | Entry point for button clicks | — | 🕵️‍♂️ Find Agent | ## Step 4: Handle Button Clicks (Assignment)… |
| 🕵️‍♂️ Find Agent | Notion | Finds agent record by Telegram ID | Telegram Callback | Agent Exists? | ## Step 4: Handle Button Clicks (Assignment)… |
| Agent Exists? | IF | Ensures agent has mapped Notion User | 🕵️‍♂️ Find Agent | 📝 Assign Lead; ❌ Error Toast | ## Step 4: Handle Button Clicks (Assignment)… |
| 📝 Assign Lead | Notion | Updates lead page: Assigned Manager people | Agent Exists? (true) | ✅ Update Chat | ## Step 4: Handle Button Clicks (Assignment)… |
| ✅ Update Chat | Telegram | Edits original message to show assignee | 📝 Assign Lead | 🔔 Show Toast | ## Step 4: Handle Button Clicks (Assignment)… |
| 🔔 Show Toast | Telegram | Answers callback query (success) | ✅ Update Chat | — | ## Step 4: Handle Button Clicks (Assignment)… |
| ❌ Error Toast | Telegram | Answers callback query (agent not found) | Agent Exists? (false) | — | ## Step 4: Handle Button Clicks (Assignment)… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- New workflow, name it as desired (e.g., the provided title).
- Ensure **Settings → Execution Order** is `v1` (matches the workflow).

2) **Add documentation sticky notes (optional but recommended)**
- Add 5 Sticky Notes with the provided texts:
  - “Main Sticky” (system overview + setup steps)
  - “Step 1…”, “Step 2…”, “Step 3…”, “Step 4…”

### A) Scrape + enrich pipeline (entry point 1)

3) **Manual Trigger**
- Add node: **Manual Trigger**
- No configuration.

4) **Set: 📝 CONFIGURATION**
- Add node: **Set**
- Add fields:
  - Number: `LIMIT_RESULTS` (e.g., 5)
  - String: `SEARCH_QUERY` (e.g., “Restaurants in Tbilisi”)
  - String: `OUTSCRAPER_API_KEY` (paste your Outscraper key)
  - String: `TELEGRAM_CHAT_ID` (target group/chat id)
  - String: `AI_TONE` (your desired style)
  - String: `MY_SERVICES` (your offer list)
- Connect: Manual Trigger → 📝 CONFIGURATION

5) **HTTP Request: 1. Fetch Google Maps Data**
- Add node: **HTTP Request**
- Method: GET (default)
- URL: `https://api.app.outscraper.com/maps/search-v2`
- Enable **Send Query Parameters**:
  - `query` = expression `{{$json.SEARCH_QUERY}}`
  - `limit` = expression `{{$json.LIMIT_RESULTS}}`
  - `drop_duplicates` = `true`
  - `async` = `false`
  - `reviewsLimit` = `1`
  - `reviewsSort` = `most_relevant`
- Enable **Send Headers**:
  - `X-API-KEY` = expression `{{$json.OUTSCRAPER_API_KEY}}`
- Connect: 📝 CONFIGURATION → 1. Fetch Google Maps Data

6) **Code: 2. Clean & Normalize**
- Add node: **Code**
- Paste the logic that:
  - normalizes website/email/name/reviews/clean_phone
  - builds `reviews_context` from `reviews_data`
  - returns a list of items (`newItems`)
- Connect: 1. Fetch Google Maps Data → 2. Clean & Normalize

7) **Split In Batches: 🔄 Loop Items**
- Add node: **Split In Batches**
- Keep defaults (or set batch size = 1 if you want strict sequential behavior).
- Connect: 2. Clean & Normalize → 🔄 Loop Items

8) **Notion: 🔎 Search Duplicate**
- Add node: **Notion**
- Credentials: connect your Notion integration
- Resource: **Database Page**
- Operation: **Get All**
- Database: select your **Active Deals (CRM)** database
- Filters (Manual):
  - Property: `Phone` (type phone_number)
  - Condition: `equals`
  - Value: expression `{{$json.clean_phone}}`
- Turn on **Always Output Data** (important for consistent branching)
- Connect: 🔄 Loop Items → 🔎 Search Duplicate

9) **Merge: 🛡️ Restore Data**
- Add node: **Merge**
- Mode: **Combine**
- Combine by: **Position**
- Connect:
  - 🔄 Loop Items → 🛡️ Restore Data (Input 1)
  - 🔎 Search Duplicate → 🛡️ Restore Data (Input 2)

10) **IF: Found?**
- Add node: **IF**
- Condition (Boolean):
  - Value 1: `{{ !!$json.id }}`
  - Value 2: `true`
- Connect: 🛡️ Restore Data → Found?

11) **NoOp: 🚫 Duplicate / Skip**
- Add node: **NoOp**
- Connect: Found? (true) → 🚫 Duplicate / Skip
- Connect: 🚫 Duplicate / Skip → 🔄 Loop Items (this is the loop-back)

12) **OpenAI (LangChain): 🤖 AI Icebreaker**
- Add node: **OpenAI (LangChain) / Message Model** (node name depends on your n8n build; use the LangChain OpenAI node)
- Credentials: set OpenAI API key in n8n credentials
- Model: `gpt-4o-mini` (or choose an equivalent)
- Prompt/message content should include:
  - `AI_TONE`, `MY_SERVICES` from 📝 CONFIGURATION
  - lead fields + `reviews_context`
  - required output format “Context/Problem -> Specific Solution”
- Connect: Found? (false) → 🤖 AI Icebreaker

13) **Merge: 🔗 Merge Final**
- Add node: **Merge**
- Mode: **Combine**
- Combine by: **Position**
- Connect:
  - 🤖 AI Icebreaker → 🔗 Merge Final (Input 1)
  - Found? (false) → 🔗 Merge Final (Input 2)

14) **Notion: 💾 Create New Lead**
- Add node: **Notion**
- Resource: **Database Page**
- Operation: **Create**
- Database: **Active Deals (CRM)**
- Map properties (must exist with matching types):
  - Company Name (Title) = `{{$json.name}}`
  - Phone (Phone) = `{{$json.clean_phone}}`
  - Email (Email) = `{{$json.email}}`
  - Website / Social (URL) = `{{$json.website}}`
  - Address (Rich text) = `{{$json.full_address || ''}}`
  - Lead ID (Rich text) = `{{$json.place_id || ''}}`
  - Lead Source (Select) = `Cold Outreach`
  - Pipeline Stage (Status) = `Cold Outreach`
  - Icebreaker (Rich text) = `{{$json.message.content}}`
- Connect: 🔗 Merge Final → 💾 Create New Lead

15) **Code: 🎨 Prepare Message**
- Add node: **Code**
- Build:
  - `telegram_message` (HTML formatted)
  - `lead_id` = created Notion page `id`
- Pull data from:
  - `$('🔗 Merge Final').item.json`
  - `$('💾 Create New Lead').item.json.id` and `.url`
- Connect: 💾 Create New Lead → 🎨 Prepare Message

16) **Telegram: 📢 Notify Team**
- Add node: **Telegram**
- Credentials: create a bot via **@BotFather**, add token to n8n Telegram credentials
- Operation: sendMessage
- Chat ID: `{{ $('📝 CONFIGURATION').first().json.TELEGRAM_CHAT_ID }}`
- Text: `{{$json.telegram_message}}`
- Additional fields:
  - Parse Mode: HTML
  - Reply Markup: Inline Keyboard
  - Button:
    - Text: `⚡️ Take Lead`
    - Callback data: `{{ 'take_lead_' + $json.lead_id }}`
- Connect: 🎨 Prepare Message → 📢 Notify Team
- Connect loop-back: 📢 Notify Team → 🔄 Loop Items

### B) Telegram claim handler (entry point 2)

17) **Telegram Trigger: Telegram Callback**
- Add node: **Telegram Trigger**
- Updates: `callback_query`
- Ensure webhook is registered (n8n will handle this when activating).
- This is a separate entry point (no connection from manual branch required).

18) **Notion: 🕵️‍♂️ Find Agent**
- Add Notion node:
  - Resource: Database Page
  - Operation: Get All
  - Database: **Agents Configuration**
  - Filter:
    - `Telegram ID` (Number) equals `{{ parseInt($json.from?.id || $json.callback_query?.from?.id) }}`
  - Return All: true
- Connect: Telegram Callback → 🕵️‍♂️ Find Agent

19) **IF: Agent Exists?**
- Add IF node:
  - Condition: `{{ ($json.properties['Notion User']?.people?.length || 0) > 0 }}` is true
- Connect: 🕵️‍♂️ Find Agent → Agent Exists?

20) **Notion: 📝 Assign Lead**
- Add Notion node:
  - Resource: Database Page
  - Operation: Update
  - Page ID: `{{ $('Telegram Callback').item.json.callback_query.data.replace('take_lead_', '') }}`
  - Set `Assigned Manager` (People):
    - Value expression: map people IDs from “Notion User” property:
      `{{ ($('🕵️‍♂️ Find Agent').item.json.properties['Notion User'].people || []).map(u => u.id) }}`
- Connect: Agent Exists? (true) → 📝 Assign Lead

21) **Telegram: ✅ Update Chat (editMessageText)**
- Add Telegram node:
  - Operation: editMessageText
  - Chat ID: `{{ $('Telegram Callback').item.json.callback_query.message.chat.id }}`
  - Message ID: `{{ $('Telegram Callback').item.json.callback_query.message.message_id }}`
  - Text: original + attribution line (HTML)
  - Parse mode: HTML
- Connect: 📝 Assign Lead → ✅ Update Chat

22) **Telegram: 🔔 Show Toast (answerCallbackQuery)**
- Add Telegram node:
  - Operation: answerCallbackQuery
- Connect: ✅ Update Chat → 🔔 Show Toast

23) **Telegram: ❌ Error Toast (answerCallbackQuery)**
- Add Telegram node:
  - Operation: answerCallbackQuery
  - (Optionally set an error text like “You are not registered as an agent.”)
- Connect: Agent Exists? (false) → ❌ Error Toast

24) **Credentials checklist**
- Notion: OAuth/token credential connected; integration has access to both databases.
- OpenAI: API key credential configured for the LangChain OpenAI node.
- Telegram: Bot token credential; bot added to target group and allowed to post/edit messages.
- Outscraper: API key placed in 📝 CONFIGURATION.

25) **Activate workflow**
- Activate to enable Telegram Trigger webhooks.
- Test:
  - Run Manual Trigger to post leads.
  - Click “⚡️ Take Lead” to verify Notion assignment and message edit.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Duplicate the companion template (link in description). Connect your Notion account and select the ‘Active Deals’ and ‘Agents’ databases in the relevant nodes.” | Mentioned in Main Sticky; template link not included in JSON. |
| “Outscraper: Get a free API key and add it to the CONFIGURATION node.” | Main Sticky setup step. |
| “Telegram: Create a bot via @BotFather, add it to your group, and get the Chat ID.” | Main Sticky setup step. |
| “OpenAI: Add your API key for the AI analysis.” | Main Sticky setup step. |