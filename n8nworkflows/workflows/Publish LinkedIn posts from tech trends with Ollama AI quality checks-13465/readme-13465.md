Publish LinkedIn posts from tech trends with Ollama AI quality checks

https://n8nworkflows.xyz/workflows/publish-linkedin-posts-from-tech-trends-with-ollama-ai-quality-checks-13465


# Publish LinkedIn posts from tech trends with Ollama AI quality checks

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Publish LinkedIn posts from tech trends with Ollama AI quality checks  
**Workflow name (in JSON):** Automatically publish LinkedIn posts using trends and AI quality checks

This workflow automates a full LinkedIn content pipeline in two scheduled flows:

### 1.1 Flow 1 — Daily Research (6 AM)
- Fetches trending tech topics from **Hacker News**, **Reddit** (8 subreddits), and **Product Hunt** in parallel.
- Normalizes each source to a common schema, merges results, runs a **7-layer deduplication strategy**, and ranks items with a custom relevance score.
- Uses an **Ollama-powered AI Writer agent** to generate **3 LinkedIn drafts** (different angles).
- Saves drafts into **Google Sheets** as a queue (status: `draft`).

### 1.2 Flow 2 — Smart Publish (Tue–Thu 9:30 AM)
- Reads all rows from the Google Sheets queue.
- Filters unpublished drafts, then an **AI Selector** picks the single best post for today.
- A **Quality Gate** reviews and scores it, then routes to:
  - **approve** → publish to LinkedIn + add hashtags comment
  - **revise** → save revised content then publish + comment
  - **reject** → mark rejected and notify via Telegram
- Updates Google Sheets status and sends Telegram notifications.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / Guidance (Sticky Notes)
**Overview:** Provides operational context, setup checklist, and warnings to customize credentials and prompts.  
**Nodes involved:** Sticky Note, Sticky Note1, Sticky Note8, Sticky Note2, Sticky Note9, Sticky Note10, Sticky Note11, Sticky Note12, Sticky Note6, Sticky Note7, Sticky Note3, Sticky Note4, Sticky Note13, Sticky Note14

#### Node details (each is `n8n-nodes-base.stickyNote`)
- **Technical role:** Non-executing annotation nodes.
- **Key content highlights:**
  - Main note explains two-flow design, setup steps, queue columns, and customization knobs.
  - Warnings:
    - Replace `YOUR_LINKEDIN_PERSON_ID`
    - Edit all 3 AI system prompts (Writer/Selector/Quality Gate)
- **Edge cases:** None (non-executing).

---

### Block B — Flow 1 Trigger & Parallel Trend Fetch
**Overview:** Triggers daily at 6 AM and fetches three sources concurrently.  
**Nodes involved:** ⏰ Daily 6 AM — Research, 🟠 Fetch Hacker News, 🔴 Fetch Reddit, 🟣 Fetch Product Hunt

#### ⏰ Daily 6 AM — Research
- **Type/role:** Schedule Trigger (`scheduleTrigger`) entry point for Flow 1.
- **Config:** Cron `0 6 * * *` (daily at 06:00).
- **Outputs:** Fan-out to the three fetch nodes.
- **Edge cases:** Instance timezone affects actual run time.

#### 🟠 Fetch Hacker News
- **Type/role:** HTTP Request (`httpRequest`) to HN Algolia API.
- **Config:**
  - GET `https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=30`
  - JSON response, timeout 15s
  - `onError: continueRegularOutput` (won’t stop workflow on failures)
- **Output:** Raw Algolia payload (`hits` array) to Normalize HN.
- **Failure modes:** Network timeout, API rate limits, schema changes; errors won’t halt flow but may reduce items.

#### 🔴 Fetch Reddit
- **Type/role:** Code node (`code`) that loops multiple subreddits and requests `/hot.json`.
- **Config choices:**
  - Subreddits: `technology, programming, startups, machinelearning, artificial, SaaS, webdev, entrepreneurship`
  - Each request:
    - GET `https://www.reddit.com/r/${sub}/hot.json?limit=8`
    - Adds `User-Agent: n8n-linkedin-poster/1.0`
    - timeout 10s
  - Filters out stickied posts and truncates `selftext` to 300 chars.
  - Returns one item per post; if nothing, returns `{ noData: true }`.
- **Output:** Already close to normalized, then passed to Normalize Reddit.
- **Failure modes / edge cases:**
  - Reddit can throttle or block anonymous requests (429/403) without OAuth; code silently ignores errors (`catch(e) {}`), potentially yielding `noData`.
  - Some posts have missing `url`; fallback uses permalink.

#### 🟣 Fetch Product Hunt
- **Type/role:** HTTP Request (`httpRequest`) to Product Hunt RSS feed.
- **Config:**
  - GET `https://www.producthunt.com/feed`
  - Response as text, timeout 15s
  - Headers set: `User-Agent: Mozilla/5.0`, `Accept: application/rss+xml,text/xml,*/*`
  - `onError: continueRegularOutput`
- **Output:** Raw XML text to Normalize PH.
- **Failure modes:** Feed format changes, Cloudflare blocking, transient HTML instead of RSS.

---

### Block C — Normalization (Standard Schema per Source)
**Overview:** Converts each source output into a consistent item structure used downstream.  
**Nodes involved:** 🔧 Normalize HN, 🔧 Normalize Reddit, 🔧 Normalize PH

#### 🔧 Normalize HN
- **Type/role:** Code node to map Algolia results → normalized items.
- **Config choices:**
  - Extracts: `title`, `url` (fallback to HN item URL), `discussionUrl`, `score`, `comments`, `author`, `created`
  - Adds: `source: hackernews`, `sourceLabel: Hacker News`, `_normalized: true`
  - If no hits, returns `{ noData: true }`.
- **Output:** Items to Merge (input index 0).
- **Edge cases:** Missing `url` or `num_comments` fields; handled with fallbacks.

#### 🔧 Normalize Reddit
- **Type/role:** Code node that passes through Reddit-shaped items, adds `_normalized`.
- **Config choices:** Filters `noData` items out; otherwise `{...d, _normalized:true}`.
- **Output:** Items to Merge (input index 1).
- **Edge cases:** If upstream returned only `{noData:true}`, outputs a single noData marker.

#### 🔧 Normalize PH
- **Type/role:** Code node parsing RSS XML text into items.
- **Config choices:**
  - Regex extracts `<item>...</item>` blocks (up to 20).
  - Extracts `title`, `link/guid`, `description`, `pubDate`
  - Strips HTML tags from description, truncates 300 chars.
  - Adds: `source: producthunt`, `sourceLabel: Product Hunt`, `_normalized:true`
- **Output:** Items to Merge (input index 2).
- **Failure modes:** XML variations may break regex; may return noData marker.

---

### Block D — Merge, Collect, Deduplicate, Rank
**Overview:** Waits for all sources, consolidates items, removes duplicates using multiple heuristics, and ranks the top trends.  
**Nodes involved:** ⏳ Wait for ALL Sources, 📦 Collect All Items, 🧹 Deduplicate + Rank, 📊 Has Items?, 💤 No Items

#### ⏳ Wait for ALL Sources
- **Type/role:** Merge node (`merge`) configured as “wait for all inputs”.
- **Config:** `numberInputs: 3`
- **Inputs:** Normalized HN/Reddit/PH.
- **Output:** Combined stream to Collect All Items.
- **Edge cases:** If one branch errors but still outputs (due to continue-on-error/noData), merge completes; if a branch outputs nothing at all, execution behavior depends on n8n merge semantics (typically it waits for all connected inputs per run).

#### 📦 Collect All Items
- **Type/role:** Code node aggregating all incoming items into one JSON object.
- **Config choices:**
  - Filters out: `noData`, missing title, not `_normalized`, title length < 5.
  - Produces:
    - `allItems` array with standardized fields
    - `totalCollected`
    - `sourceCounts` by `source`
    - `collectedAt`
- **Output:** Single item to Deduplicate + Rank.
- **Edge cases:** If all are filtered out, returns `totalCollected: 0` and empty `allItems`.

#### 🧹 Deduplicate + Rank
- **Type/role:** Code node implementing 7-layer dedup and custom ranking.
- **Dedup layers (as implemented):**
  1. Normalized URL exact match (`nu`)
  2. “Canonical/external content key” (`ec`) for some domains (special handling for Reddit/HN/ProductHunt)
  3. Normalized title exact match (`nt`)
  4. First 4 significant words match (stopwords removed)
  5. First 3 significant words match
  6. Jaccard similarity of significant words > 0.6
  7. Title containment (one includes the other)
- **Ranking signals:**
  - Combined score and comments across duplicates
  - Cross-source count (strong boost: `(sources-1)*8`)
  - Freshness boost if created within last 12 hours
  - Produces `relevanceScore`, `isMultiSource`, `crossSources`, `combinedScore`, `combinedComments`
- **Outputs:**
  - `topItems` (top 20)
  - `topItemsFormatted` (human-readable list used as AI prompt input)
  - `stats` including dedup rate
  - `itemCount`
- **Failure modes / edge cases:**
  - Aggressive dedup may over-collapse distinct stories with similar titles.
  - Date parsing issues if `created` is not ISO/RFC-compatible.
  - Relevance score is heuristic and may need tuning for your audience.

#### 📊 Has Items?
- **Type/role:** IF node routing based on `itemCount > 0`.
- **True output:** 🤖 AI LinkedIn Writer
- **False output:** 💤 No Items
- **Edge cases:** If `itemCount` missing/non-numeric, strict validation may route unexpectedly (but upstream always sets it).

#### 💤 No Items
- **Type/role:** Code node producing a “no trends” status object.
- **Output:** Ends Flow 1.
- **Purpose:** Avoids downstream AI/Sheets when no data.

---

### Block E — AI Draft Generation (Ollama Writer) + Extraction
**Overview:** Generates 3 draft posts using an AI agent, then robustly extracts JSON posts and formats them as queue rows.  
**Nodes involved:** Ollama Writer, 🤖 AI LinkedIn Writer, 📝 Extract Posts

#### Ollama Writer
- **Type/role:** LangChain Ollama chat model (`lmChatOllama`) used as the agent’s LLM.
- **Config:** Model `mistral`
- **Connections:** Provides `ai_languageModel` input to 🤖 AI LinkedIn Writer.
- **Requirements:** Local Ollama reachable by n8n (commonly `http://localhost:11434` depending on n8n Ollama credential/config).
- **Failure modes:** Model not pulled, Ollama not running, network access from n8n container to host.

#### 🤖 AI LinkedIn Writer
- **Type/role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) to generate drafts.
- **Input text:** Uses `topItemsFormatted` and instructs “Generate 3 LinkedIn post drafts. Each DIFFERENT angle.”
- **System message:** Contains placeholders:
  - `[YOUR NAME]`, `[YOUR EXPERTISE]`, `[YOUR PRODUCT]`
  - Strict output contract: **RAW JSON ONLY**, schema with `posts[]`, `weekTheme`, `contentCalendarNote`
  - Style constraints: 150–250 words, hook first line, no hashtags in body, max 3 emoji, prioritize ⭐ trends.
- **Output:** Agent result (may be nested) to 📝 Extract Posts.
- **Failure modes:** Model may output non-JSON; addressed by Extract Posts fallback parsing.

#### 📝 Extract Posts
- **Type/role:** Code node that:
  - Attempts to locate JSON with `posts` anywhere in the agent output (deep search).
  - Removes markdown fences if present.
  - Fallback: inserts a manual-draft placeholder if parsing fails.
- **Creates queue row fields:**
  - `postId` = `${YYYY-MM-DD}-${i+1}-${angle}`
  - `hashtags` stored as a space-separated string
  - `status: draft`, `createdDate`, `generatedAt`
  - `dedupStats` derived from the dedup node stats (via `$('📊 Has Items?').item.json`)
- **Output:** 1 item per post draft to Google Sheets append.
- **Edge cases:**
  - If the agent returns valid JSON but with different keys, extraction may fail.
  - `sourceData.stats...` assumes Has Items? true-branch item contains dedup stats (it does, from Deduplicate + Rank).

---

### Block F — Save Drafts to Google Sheets Queue
**Overview:** Appends each generated draft as a new row in a Google Sheet “Post Queue”.  
**Nodes involved:** 📋 Save to Queue

#### 📋 Save to Queue
- **Type/role:** Google Sheets node (`googleSheets`) append.
- **Config choices:**
  - Operation: `append`
  - Document: `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - Sheet tab: `gid=0` (cached name “Post Queue”)
  - Mapping mode: “define below”; converts fields to string.
  - Writes key columns: Post ID, Angle, Hook Line, Full Post, Hashtags, Trend Referenced, Word Count, Best Day, Posting Notes, Status, Created Date, Generated At, Dedup Stats.
- **Failure modes:**
  - Auth/permission errors (Google credentials).
  - Wrong sheet ID or gid/tab mismatch.
  - Column names must match exactly; otherwise data may go into wrong columns or be blank.

---

### Block G — Flow 2 Trigger + Load Drafts
**Overview:** Triggers Tue–Thu at 9:30 AM, reads the queue, and prepares candidate drafts for selection.  
**Nodes involved:** ⏰ Tue-Thu 9:30 AM — Publish, 📋 Read ALL Drafts, 📦 Collect All Drafts, 📄 Has Drafts?, 💤 No Drafts

#### ⏰ Tue-Thu 9:30 AM — Publish
- **Type/role:** Schedule Trigger entry point for Flow 2.
- **Config:** Cron `30 9 * * 2,3,4` (Tue/Wed/Thu at 09:30).
- **Edge cases:** Timezone again matters.

#### 📋 Read ALL Drafts
- **Type/role:** Google Sheets read node.
- **Config:** Uses `YOUR_GOOGLE_SHEET_ID` and `YOUR_SHEET_GID` (must be replaced).
- **Output:** All rows to Collect All Drafts.
- **Failure modes:** Same Google auth/sheet reference issues.

#### 📦 Collect All Drafts
- **Type/role:** Code node to filter unpublished drafts and build AI-readable context.
- **Draft inclusion rule:**
  - `Status` in `{draft, pending_review, approved}` (case-insensitive) AND no `Published Date`.
- **Also collects recent published context:**
  - `Status` in `{published, revised_and_published}` to avoid repeating angle/trend.
- **Produces:**
  - `hasDrafts`, `totalDrafts`, `drafts[]`
  - `draftsFormatted` (full text of each draft)
  - `recentPostsContext` (last 5)
  - `todayName`, `todayDate`
- **Edge cases:** If sheet uses different header names, mapping might miss fields (the code tries both “Title Case” and camelCase variants for many fields).

#### 📄 Has Drafts?
- **Type/role:** IF node checking boolean `hasDrafts === true`.
- **True output:** 🧠 AI Post Selector
- **False output:** 💤 No Drafts

#### 💤 No Drafts
- **Type/role:** Code node returning a “no drafts” status object; ends Flow 2.

---

### Block H — AI Selection (Ollama Selector) + Extraction
**Overview:** AI chooses the best draft for today using weighted criteria; extraction node ensures a reliable selected post object for review.  
**Nodes involved:** Ollama Selector, 🧠 AI Post Selector, 🎯 Extract Selection

#### Ollama Selector
- **Type/role:** Ollama chat model for selector agent.
- **Config:** `model: YOUR_MODEL_NAME` (must be replaced with a real local model name).
- **Connections:** `ai_languageModel` → 🧠 AI Post Selector.
- **Failure modes:** Same Ollama availability/model mismatch.

#### 🧠 AI Post Selector
- **Type/role:** LangChain Agent to select one post.
- **Prompt content:** Includes drafts list, today’s date/day, and recent published context.
- **System message:** Weighted selection rubric and strict JSON output contract:
  - Must output `selectedPostId`, reasoning, ratings, alternate backup, etc.
  - Prioritizes status `approved` or `force_publish` (note: upstream filter allows `approved`; `force_publish` is mentioned but not included in the filter list unless you add it).
- **Output:** Agent result → 🎯 Extract Selection.

#### 🎯 Extract Selection
- **Type/role:** Code node to parse `selectedPostId` from possibly nested agent output.
- **Behavior:**
  - Deep searches for JSON containing `selectedPostId`.
  - Fallback: selects the first draft if parsing fails.
  - Outputs the selected post content plus selection metadata.
- **Edge cases:**
  - If AI returns a Post ID not present, it falls back to first draft.
  - If `draftsData.drafts` is empty (shouldn’t happen because IF gate), would error—guarded by the IF.

---

### Block I — AI Quality Gate (Ollama Reviewer) + Decision Routing
**Overview:** Reviews selected post, scores across 5 criteria, decides approve/revise/reject; extraction normalizes outputs; switch routes downstream actions.  
**Nodes involved:** Ollama Reviewer, 🛡️ AI Quality Gate, 🎯 Extract Review, 🔀 Review Decision

#### Ollama Reviewer
- **Type/role:** Ollama chat model for quality gate agent.
- **Config:** `model: YOUR_MODEL_NAME` (replace).
- **Connections:** `ai_languageModel` → 🛡️ AI Quality Gate.

#### 🛡️ AI Quality Gate
- **Type/role:** LangChain Agent that evaluates and optionally rewrites.
- **Prompt content:** Includes selector context + post content.
- **System message:**
  - Criteria: hook/value/authenticity/engagement/brand safety (1–10)
  - Decision rules:
    - approve: avg ≥ 7 and none < 5
    - revise: avg 5–7 or fixable
    - reject: avg < 5 or brandSafety < 5
  - If revise: must output complete revised post.
  - Strict raw JSON output schema.
- **Output:** Agent result → 🎯 Extract Review.
- **Failure modes:** Non-JSON output; handled by extraction fallback.

#### 🎯 Extract Review
- **Type/role:** Code node that:
  - Extracts decision JSON (`approve|revise|reject`) from nested output
  - Fallback default: **approve** with “parse failed” note
  - If decision is `revise` and `revisedPost` exists:
    - sets `fullPost` to revised
    - sets `hookLine` to revised hook (or first line)
    - `wasRevised: true`
- **Also passes through:** `alternatePostId` for rejection notification.
- **Edge cases:** If AI says `revise` but omits `revisedPost`, it will publish the original (decision remains revise, but content unchanged). Consider adding a guard if you want “revise requires revisedPost”.

#### 🔀 Review Decision
- **Type/role:** Switch node routing by `$json.decision`.
- **Rules/outputs:**
  - `approve` → 📤 Publish to LinkedIn
  - `revise` → 💾 Save Revised
  - `reject` → ❌ Mark Rejected
- **Edge cases:** Any unexpected decision string won’t match; with current setup it would produce no output (effectively ending). Extract Review tries to prevent this.

---

### Block J — Publish to LinkedIn + Comment + Update Sheet + Notify
**Overview:** Posts to LinkedIn UGC API, comments hashtags, updates Google Sheet status, and notifies Telegram.  
**Nodes involved:** 📤 Publish to LinkedIn, 💬 Hashtags Comment, ✅ Mark Published, 📲 Published ✅

#### 📤 Publish to LinkedIn
- **Type/role:** HTTP Request to LinkedIn UGC Posts API.
- **Config:**
  - POST `https://api.linkedin.com/v2/ugcPosts`
  - OAuth2 (generic credential type)
  - Headers: `Content-Type: application/json`, `X-Restli-Protocol-Version: 2.0.0`
  - Body includes:
    - `author: urn:li:person:YOUR_LINKEDIN_PERSON_ID` (**must replace**)
    - `shareCommentary.text` set via `JSON.stringify($json.fullPost)`
    - PUBLIC visibility
  - `onError: continueRegularOutput`
- **Output:** LinkedIn response JSON (should include an `id`).
- **Failure modes:**
  - Missing/invalid OAuth scopes (typically `w_member_social`).
  - Incorrect Person URN.
  - LinkedIn API may return 401/403/429; node continues, but downstream expects `$json.id`.

#### 💬 Hashtags Comment
- **Type/role:** HTTP Request to LinkedIn comments API.
- **Config:**
  - POST `https://api.linkedin.com/v2/socialActions/{{encodeURIComponent($json.id)}}/comments`
  - OAuth2
  - Body includes `actor: urn:li:person:YOUR_LINKEDIN_PERSON_ID` (**must replace**)
  - Comment text uses hashtags from `$('🎯 Extract Review').item.json.hashtags` (string)
- **Dependency:** Requires `id` from Publish node; if publish failed and no `id`, URL becomes invalid.
- **Failure modes:** Same OAuth issues; invalid socialActions ID if publish failed; comments endpoint permissions.

#### ✅ Mark Published
- **Type/role:** Google Sheets appendOrUpdate to mark row published.
- **Matching:** Uses `Post ID` as matching column.
- **Writes:**
  - `Status` = `published` or `revised_and_published` based on `wasRevised`
  - `AI Review` summary (selection reason + review summary + score)
  - `LinkedIn URL` = `$('📤 Publish to LinkedIn').item.json.id || 'published'` (note: this stores the **post id**, not a full URL)
  - `Revised Post` if revised
  - `Published Date` = ISO timestamp
- **Edge cases:**
  - If publish failed, LinkedIn ID missing; sheet will store `'published'`, which may be misleading.
  - If duplicate Post IDs exist in sheet, update behavior can be ambiguous.

#### 📲 Published ✅
- **Type/role:** Telegram node sending a Markdown message.
- **Config:** `chatId: YOUR_TELEGRAM_CHAT_ID` (replace) and bot credentials required in n8n.
- **Message includes:** angle, hook, trend, quality score, revision flag, selector reason.
- **Failure modes:** Wrong chat ID, bot not allowed to message chat, Markdown parsing issues if text contains special characters.

---

### Block K — Revise Path: Save Revised then Publish
**Overview:** If quality gate decides “revise”, saves revised content to the sheet before publishing.  
**Nodes involved:** 💾 Save Revised (then flows into 📤 Publish to LinkedIn)

#### 💾 Save Revised
- **Type/role:** Google Sheets appendOrUpdate.
- **Matching:** `Post ID`
- **Writes:**
  - `Status: revised_and_published` (note: this is set **before** actual publish)
  - Updates `Full Post`, `Hook Line`, `AI Review` including issues/score
- **Connections:** Output → 📤 Publish to LinkedIn
- **Edge cases:**
  - Status becomes “revised_and_published” even if publishing later fails.
  - Consider setting an intermediate status like `revised_ready` and only mark published after LinkedIn success.

---

### Block L — Reject Path: Mark Rejected + Notify
**Overview:** Marks the selected post as rejected and sends Telegram alert.  
**Nodes involved:** ❌ Mark Rejected, 📲 Rejected ❌

#### ❌ Mark Rejected
- **Type/role:** Google Sheets appendOrUpdate.
- **Matching:** `Post ID`
- **Writes:** `Status: rejected`, and an `AI Review` rejection summary.
- **Output:** to Telegram reject notification.
- **Failure modes:** Same Sheets auth/schema issues.

#### 📲 Rejected ❌
- **Type/role:** Telegram message notifying rejection, includes backup post ID.
- **Edge cases:** Markdown escaping (issues text may contain characters Telegram interprets).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Workflow overview and setup notes | — | — | ## Automatically publish LinkedIn posts using trends and AI quality checks … (full note includes setup steps, customization, and queue columns) |
| Sticky Note1 | stickyNote | Label: fetch trends in parallel | — | — | ## Fetch trending topics\nHacker News, Reddit, and Product Hunt fetched in parallel. |
| Sticky Note8 | stickyNote | Label: normalize data | — | — | ## Normalize fetched data\nStandardize output format from each source. |
| Sticky Note2 | stickyNote | Label: merge/dedup/rank/generate | — | — | ## Merge, deduplicate, and generate drafts\nWait for all sources, run 7-layer dedup, rank, then AI writes 3 drafts. |
| Sticky Note9 | stickyNote | Label: save drafts | — | — | ## Save drafts to Google Sheets\nAppend generated posts to the queue with status "draft". |
| Sticky Note10 | stickyNote | Label: read/select | — | — | ## Read drafts and select best post\nLoad all drafts, filter unpublished, AI Selector picks the best one for today. |
| Sticky Note11 | stickyNote | Label: quality gate | — | — | ## AI Quality Gate and review\nScore the selected post across 5 criteria, then route to approve, revise, or reject. |
| Sticky Note12 | stickyNote | Label: publish/notify | — | — | ## Publish to LinkedIn and notify\nPublish approved or revised post, add hashtag comment, update sheet, send Telegram notification. |
| Sticky Note6 | stickyNote | Warning: replace LinkedIn person id | — | — | ## ⚠️ Replace YOUR_LINKEDIN_PERSON_ID\nUpdate the Publish and Comment nodes with your LinkedIn Person URN. |
| Sticky Note7 | stickyNote | Warning: edit AI prompts | — | — | ##  **⚠️ Edit all 3 AI system prompts**\n**Update Writer, Selector, and Quality Gate with your name, expertise, and tone.** |
| Sticky Note3 | stickyNote | Label: Flow 1 | — | — | ## **Flow 1 — Daily Research (6 AM):** |
| Sticky Note4 | stickyNote | Label: Flow 2 | — | — | ## **Flow 2 — Smart Publish (Tue–Thu 9:30 AM):** |
| Sticky Note13 | stickyNote | Warning: edit AI prompts | — | — | ##  **⚠️ Edit all 3 AI system prompts**\n**Update Writer, Selector, and Quality Gate with your name, expertise, and tone.** |
| Sticky Note14 | stickyNote | Warning: edit AI prompts | — | — | ##  **⚠️ Edit all 3 AI system prompts**\n**Update Writer, Selector, and Quality Gate with your name, expertise, and tone.** |
| ⏰ Daily 6 AM — Research | scheduleTrigger | Entry point for daily trend research | — | 🟠 Fetch Hacker News; 🔴 Fetch Reddit; 🟣 Fetch Product Hunt | ## **Flow 1 — Daily Research (6 AM):** |
| 🟠 Fetch Hacker News | httpRequest | Fetch HN front page trends | ⏰ Daily 6 AM — Research | 🔧 Normalize HN | ## Fetch trending topics… |
| 🔴 Fetch Reddit | code | Fetch hot posts from multiple subreddits | ⏰ Daily 6 AM — Research | 🔧 Normalize Reddit | ## Fetch trending topics… |
| 🟣 Fetch Product Hunt | httpRequest | Fetch Product Hunt RSS feed | ⏰ Daily 6 AM — Research | 🔧 Normalize PH | ## Fetch trending topics… |
| 🔧 Normalize HN | code | Normalize HN payload to standard schema | 🟠 Fetch Hacker News | ⏳ Wait for ALL Sources | ## Normalize fetched data… |
| 🔧 Normalize Reddit | code | Normalize Reddit items to standard schema | 🔴 Fetch Reddit | ⏳ Wait for ALL Sources | ## Normalize fetched data… |
| 🔧 Normalize PH | code | Parse RSS XML and normalize PH items | 🟣 Fetch Product Hunt | ⏳ Wait for ALL Sources | ## Normalize fetched data… |
| ⏳ Wait for ALL Sources | merge | Wait/merge 3 sources | 🔧 Normalize HN; 🔧 Normalize Reddit; 🔧 Normalize PH | 📦 Collect All Items | ## Merge, deduplicate, and generate drafts… |
| 📦 Collect All Items | code | Aggregate all normalized items into one object | ⏳ Wait for ALL Sources | 🧹 Deduplicate + Rank | ## Merge, deduplicate, and generate drafts… |
| 🧹 Deduplicate + Rank | code | 7-layer dedup + relevance ranking | 📦 Collect All Items | 📊 Has Items? | ## Merge, deduplicate, and generate drafts… |
| 📊 Has Items? | if | Route if there are ranked items | 🧹 Deduplicate + Rank | 🤖 AI LinkedIn Writer; 💤 No Items | ## Merge, deduplicate, and generate drafts… |
| 💤 No Items | code | End Flow 1 when no trends | 📊 Has Items? (false) | — |  |
| Ollama Writer | lmChatOllama | LLM for writer agent | — | 🤖 AI LinkedIn Writer (ai_languageModel) | ##  **⚠️ Edit all 3 AI system prompts**… |
| 🤖 AI LinkedIn Writer | langchain agent | Generate 3 LinkedIn drafts as JSON | 📊 Has Items? (true) + Ollama Writer | 📝 Extract Posts | ## Merge, deduplicate, and generate drafts… |
| 📝 Extract Posts | code | Parse AI JSON, build queue rows | 🤖 AI LinkedIn Writer | 📋 Save to Queue | ## Save drafts to Google Sheets… |
| 📋 Save to Queue | googleSheets | Append drafts to queue sheet | 📝 Extract Posts | — | ## Save drafts to Google Sheets… |
| ⏰ Tue-Thu 9:30 AM — Publish | scheduleTrigger | Entry point for publish runs | — | 📋 Read ALL Drafts | ## **Flow 2 — Smart Publish (Tue–Thu 9:30 AM):** |
| 📋 Read ALL Drafts | googleSheets | Read queue sheet rows | ⏰ Tue-Thu 9:30 AM — Publish | 📦 Collect All Drafts | ## Read drafts and select best post… |
| 📦 Collect All Drafts | code | Filter unpublished drafts + format for AI | 📋 Read ALL Drafts | 📄 Has Drafts? | ## Read drafts and select best post… |
| 📄 Has Drafts? | if | Route if unpublished drafts exist | 📦 Collect All Drafts | 🧠 AI Post Selector; 💤 No Drafts | ## Read drafts and select best post… |
| 💤 No Drafts | code | End Flow 2 when no drafts | 📄 Has Drafts? (false) | — |  |
| Ollama Selector | lmChatOllama | LLM for selector agent | — | 🧠 AI Post Selector (ai_languageModel) | ##  **⚠️ Edit all 3 AI system prompts**… |
| 🧠 AI Post Selector | langchain agent | Select best post ID for today | 📄 Has Drafts? (true) + Ollama Selector | 🎯 Extract Selection | ## Read drafts and select best post… |
| 🎯 Extract Selection | code | Parse selection JSON and output chosen draft | 🧠 AI Post Selector | 🛡️ AI Quality Gate | ## AI Quality Gate and review… |
| Ollama Reviewer | lmChatOllama | LLM for quality gate agent | — | 🛡️ AI Quality Gate (ai_languageModel) | ##  **⚠️ Edit all 3 AI system prompts**… |
| 🛡️ AI Quality Gate | langchain agent | Score + approve/revise/reject, possibly rewrite | 🎯 Extract Selection + Ollama Reviewer | 🎯 Extract Review | ## AI Quality Gate and review… |
| 🎯 Extract Review | code | Normalize decision, choose final post text | 🛡️ AI Quality Gate | 🔀 Review Decision | ## AI Quality Gate and review… |
| 🔀 Review Decision | switch | Route by decision | 🎯 Extract Review | 📤 Publish to LinkedIn; 💾 Save Revised; ❌ Mark Rejected | ## AI Quality Gate and review… |
| 💾 Save Revised | googleSheets | Save revised content back to sheet | 🔀 Review Decision (revise) | 📤 Publish to LinkedIn | ## Publish to LinkedIn and notify… |
| 📤 Publish to LinkedIn | httpRequest | Create LinkedIn UGC post | 🔀 Review Decision (approve) or 💾 Save Revised | 💬 Hashtags Comment | ## Publish to LinkedIn and notify… |
| 💬 Hashtags Comment | httpRequest | Comment hashtags under published post | 📤 Publish to LinkedIn | ✅ Mark Published; 📲 Published ✅ | ## Publish to LinkedIn and notify… |
| ✅ Mark Published | googleSheets | Update status + metadata in sheet | 💬 Hashtags Comment | — | ## Publish to LinkedIn and notify… |
| 📲 Published ✅ | telegram | Notify success on Telegram | 💬 Hashtags Comment | — | ## Publish to LinkedIn and notify… |
| ❌ Mark Rejected | googleSheets | Mark row rejected | 🔀 Review Decision (reject) | 📲 Rejected ❌ | ## AI Quality Gate and review… |
| 📲 Rejected ❌ | telegram | Notify rejection on Telegram | ❌ Mark Rejected | — | ## Publish to LinkedIn and notify… |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1. **Google Sheet**: Create a spreadsheet with a tab (e.g., “Post Queue”) containing at least these headers (exact spelling matters):
   - Post ID, Angle, Hook Line, Full Post, Hashtags, Trend Referenced, Word Count, Best Day, Posting Notes, Status, Created Date, Published Date, LinkedIn URL, AI Review, Revised Post, Dedup Stats, Generated At
2. **Ollama** installed and running locally; pull models you’ll use:
   - Example: `ollama pull mistral`
3. **LinkedIn API App**:
   - App with `w_member_social` scope
   - Obtain your **Person URN**: `urn:li:person:<ID>`
4. **Telegram Bot**:
   - Create bot via @BotFather; get bot token and destination chat ID.
5. In n8n, create credentials:
   - Google Sheets OAuth2
   - LinkedIn OAuth2 (generic OAuth2 API works)
   - Telegram credential
   - Ollama access (depending on node setup; often no auth but must be reachable)

---

### Flow 1 — Daily Research (6 AM)
1. **Add node:** *Schedule Trigger*  
   - Name: `⏰ Daily 6 AM — Research`  
   - Cron: `0 6 * * *`
2. **Add node:** *HTTP Request*  
   - Name: `🟠 Fetch Hacker News`  
   - GET `https://hn.algolia.com/api/v1/search?tags=front_page&hitsPerPage=30`  
   - Response: JSON, Timeout: 15000ms  
   - Error handling: “Continue on Fail” (equivalent to `continueRegularOutput`)
3. **Add node:** *Code*  
   - Name: `🔴 Fetch Reddit`  
   - Paste code that loops subreddits and calls `https://www.reddit.com/r/<sub>/hot.json?limit=8` with a User-Agent, returning items (as in workflow).
4. **Add node:** *HTTP Request*  
   - Name: `🟣 Fetch Product Hunt`  
   - GET `https://www.producthunt.com/feed`  
   - Response: Text, Timeout: 15000ms  
   - Headers: `User-Agent: Mozilla/5.0`, `Accept: application/rss+xml,text/xml,*/*`  
   - Continue on fail enabled
5. **Add normalization Code nodes:**
   - `🔧 Normalize HN` (maps HN hits → `{title,url,discussionUrl,score,comments,source,sourceLabel,created,_normalized}`)  
   - `🔧 Normalize Reddit` (filters noData, adds `_normalized`)  
   - `🔧 Normalize PH` (parse RSS `<item>` blocks, extract title/link/desc/pubDate)
6. **Connect:**
   - Trigger → all three fetch nodes
   - Each fetch → its normalize node
7. **Add node:** *Merge*  
   - Name: `⏳ Wait for ALL Sources`  
   - Mode: “Wait for all inputs” / `numberInputs = 3`  
   - Connect each normalize node to a different merge input.
8. **Add node:** *Code*  
   - Name: `📦 Collect All Items`  
   - Aggregate all merged items into `allItems`, compute `sourceCounts`.
9. **Add node:** *Code*  
   - Name: `🧹 Deduplicate + Rank`  
   - Implement the 7-layer dedup + scoring; output `topItemsFormatted`, `itemCount`, `stats`.
10. **Add node:** *IF*  
   - Name: `📊 Has Items?`  
   - Condition: `{{ $json.itemCount }} > 0`
11. **Add node:** *Code*  
   - Name: `💤 No Items`  
   - Returns a status message; connect from IF “false”.
12. **Add node:** *Ollama Chat Model*  
   - Name: `Ollama Writer`  
   - Model: `mistral` (or your choice)
13. **Add node:** *LangChain Agent*  
   - Name: `🤖 AI LinkedIn Writer`  
   - Connect IF “true” → agent main input  
   - Connect `Ollama Writer` → agent **ai_languageModel** input  
   - Prompt: include `{{ $json.topItemsFormatted }}` and request 3 drafts  
   - System message: set your name/expertise/product/tone and enforce **raw JSON** schema for 3 posts.
14. **Add node:** *Code*  
   - Name: `📝 Extract Posts`  
   - Parse the agent output to get `posts[]`; build queue rows (`postId`, fields, `status:'draft'`, `dedupStats`).
15. **Add node:** *Google Sheets*  
   - Name: `📋 Save to Queue`  
   - Operation: Append  
   - Document ID: your sheet ID  
   - Sheet/tab: your “Post Queue” tab  
   - Map columns from `📝 Extract Posts` outputs.

---

### Flow 2 — Smart Publish (Tue–Thu 9:30 AM)
16. **Add node:** *Schedule Trigger*  
   - Name: `⏰ Tue-Thu 9:30 AM — Publish`  
   - Cron: `30 9 * * 2,3,4`
17. **Add node:** *Google Sheets*  
   - Name: `📋 Read ALL Drafts`  
   - Operation: Read/Get Many rows (default read)  
   - Document ID + sheet gid/tab set correctly.
18. **Add node:** *Code*  
   - Name: `📦 Collect All Drafts`  
   - Filter rows where `Status` in `draft|pending_review|approved` and `Published Date` empty.  
   - Create `draftsFormatted`, `recentPostsContext`, `todayName/todayDate`, and `hasDrafts`.
19. **Add node:** *IF*  
   - Name: `📄 Has Drafts?`  
   - Condition: `{{ $json.hasDrafts }} is true`
20. **Add node:** *Code*  
   - Name: `💤 No Drafts`  
   - Connect from IF “false”.
21. **Add node:** *Ollama Chat Model*  
   - Name: `Ollama Selector`  
   - Model: your local model name (must exist in Ollama)
22. **Add node:** *LangChain Agent*  
   - Name: `🧠 AI Post Selector`  
   - Main input from IF “true”  
   - ai_languageModel from `Ollama Selector`  
   - Prompt: include `{{ $json.draftsFormatted }}` and `{{ $json.recentPostsContext }}`  
   - System message: rubric + strict JSON returning `selectedPostId` and metadata.
23. **Add node:** *Code*  
   - Name: `🎯 Extract Selection`  
   - Parse `selectedPostId`, find the chosen draft in `drafts[]`, output final selection object.
24. **Add node:** *Ollama Chat Model*  
   - Name: `Ollama Reviewer`  
   - Model: your local model name
25. **Add node:** *LangChain Agent*  
   - Name: `🛡️ AI Quality Gate`  
   - Main input from `🎯 Extract Selection`  
   - ai_languageModel from `Ollama Reviewer`  
   - System message: scoring rules + decision rules + revisedPost requirement + raw JSON schema.
26. **Add node:** *Code*  
   - Name: `🎯 Extract Review`  
   - Parse decision; if revise, replace post content with revised; set `wasRevised`.
27. **Add node:** *Switch*  
   - Name: `🔀 Review Decision`  
   - Rules on `{{ $json.decision }}` equals `approve`, `revise`, `reject`.
28. **Revise branch (optional pre-save):**  
   - Add *Google Sheets* node `💾 Save Revised` (appendOrUpdate matching `Post ID`) to update Full Post/Hook/Status/AI Review.
   - Connect `🔀 Review Decision (revise)` → `💾 Save Revised` → `📤 Publish to LinkedIn`
29. **Approve branch:**  
   - Connect `🔀 Review Decision (approve)` → `📤 Publish to LinkedIn`
30. **Add node:** *HTTP Request*  
   - Name: `📤 Publish to LinkedIn`  
   - POST `https://api.linkedin.com/v2/ugcPosts`  
   - OAuth2 credential (LinkedIn)  
   - Headers: `Content-Type: application/json`, `X-Restli-Protocol-Version: 2.0.0`  
   - Body includes your Person URN: `urn:li:person:<YOUR_ID>`  
   - Set `shareCommentary.text` to `{{ JSON.stringify($json.fullPost) }}`
31. **Add node:** *HTTP Request*  
   - Name: `💬 Hashtags Comment`  
   - POST `https://api.linkedin.com/v2/socialActions/{{ encodeURIComponent($json.id) }}/comments`  
   - OAuth2 credential  
   - Body actor = your Person URN; message text from extracted review hashtags.
32. **Add node:** *Google Sheets*  
   - Name: `✅ Mark Published`  
   - Operation: appendOrUpdate  
   - Matching column: `Post ID`  
   - Update `Status`, `Published Date`, `AI Review`, `LinkedIn URL`, `Revised Post` (if revised).
33. **Add node:** *Telegram*  
   - Name: `📲 Published ✅`  
   - Send formatted success message to your chat ID.
34. **Reject branch:**  
   - Add *Google Sheets* node `❌ Mark Rejected` (appendOrUpdate)  
   - Add *Telegram* node `📲 Rejected ❌`  
   - Connect `🔀 Review Decision (reject)` → `❌ Mark Rejected` → `📲 Rejected ❌`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Replace `YOUR_GOOGLE_SHEET_ID` and (for Flow 2 read) `YOUR_SHEET_GID` | Required for Google Sheets nodes to work |
| Replace `YOUR_LINKEDIN_PERSON_ID` in both LinkedIn HTTP nodes | Needed in `author` (ugcPosts) and `actor` (comments) |
| Edit all 3 AI system prompts with your name/expertise/tone | Writer, Selector, Quality Gate prompts contain placeholders |
| Sources used | Hacker News Algolia API (`https://hn.algolia.com/api/v1/search?...`), Reddit hot JSON (`https://www.reddit.com/r/<sub>/hot.json?...`), Product Hunt RSS (`https://www.producthunt.com/feed`) |
| LinkedIn endpoints used | `https://api.linkedin.com/v2/ugcPosts` and `https://api.linkedin.com/v2/socialActions/{id}/comments` |

