Generate long-form SEO articles from news with LINE approvals and Google Docs

https://n8nworkflows.xyz/workflows/generate-long-form-seo-articles-from-news-with-line-approvals-and-google-docs-12866


# Generate long-form SEO articles from news with LINE approvals and Google Docs

## 1. Workflow Overview

**Purpose:**  
This workflow automates turning fresh niche news into a long-form, SEO-optimized Google Doc article, with **two human approvals via LINE** (keywords/title, then outline) and a **recursive chapter-writing loop** to improve depth.

**Typical use cases:**
- Daily content production for an automation/AI niche blog
- Human-in-the-loop editorial control for AI-generated content
- Publishing pipeline that archives outputs (Google Docs + Google Sheets history)

### 1.1 News Discovery & Content Strategy (Phase 1)
Triggered on a schedule, the workflow pulls Google News RSS results, parses items, and asks an OpenAI-powered agent to propose keywords + a title. It then sends the proposal to LINE and pauses for approval (wait/resume pattern).

### 1.2 LINE Webhook Approval & Resume Bridge (Phase 2)
A separate webhook listens for the LINE message **“Create Article”**, looks up the stored **execution resume URL** in Google Sheets, and calls it to resume the paused main run.

### 1.3 Outline + Recursive Chapter Writing to Google Docs (Phase 3)
After approval, the workflow generates an outline, requests approval again, then creates a Google Doc, splits the outline into sections, loops through them, writes each chapter (400–600 words) in HTML, appends it to the Doc, logs the final Doc to Sheets, and notifies via LINE.

---

## 2. Block-by-Block Analysis

### Block 1 — News Discovery & AI Strategy (Phase 1)

**Overview:**  
Fetches the latest Google News RSS items for a defined query, extracts titles/links, and uses an AI agent to propose 3 SEO keywords plus a catchy article title. It then stores the execution resume URL for later approval-based resumption and sends the proposal to LINE.

**Nodes Involved:**
- 1. Daily Schedule Trigger
- Workflow Configuration
- 2. Get Latest News
- Parse RSS Feed
- OpenAI Chat Model
- 3. AI: Keyword Selection
- Store Resume Key
- 4. LINE: KW Approval Request
- 5. Wait: KW Approval

#### Node: **1. Daily Schedule Trigger** (Schedule Trigger)
- **Role:** Primary scheduled entry point.
- **Config choices:** Runs on an interval rule (the JSON shows `interval: [{}]`, which typically means it needs to be set in the UI to a real interval).
- **Outputs:** To **Workflow Configuration**.
- **Failure/edge cases:**
  - If schedule is not properly configured, it may never fire.
  - Timezone differences can affect expected firing time.

#### Node: **Workflow Configuration** (Set)
- **Role:** Centralizes parameters used across the workflow.
- **Config choices:** Sets:
  - `rssUrl`: Google News RSS search URL
  - `lineUserId`: LINE recipient user ID
  - `sheetId`: Google Sheets document ID used for resume keys and history
- **Key variables used later:**
  - `{{ $json.rssUrl }}`
  - `{{ $('Workflow Configuration').item.json.lineUserId }}`
  - `{{ $('Workflow Configuration').item.json.sheetId }}`
- **Outputs:** To **2. Get Latest News**.
- **Failure/edge cases:**
  - Wrong `lineUserId` → LINE push fails or sends to wrong user.
  - Wrong `sheetId` → Google Sheets operations fail.

#### Node: **2. Get Latest News** (HTTP Request)
- **Role:** Fetches RSS XML from the configured RSS URL.
- **Config choices:** GET request to `={{ $json.rssUrl }}`.
- **Outputs:** To **Parse RSS Feed**.
- **Failure/edge cases:**
  - RSS endpoint throttling or temporary errors (429/5xx).
  - Response shape differences: this workflow expects `json.data` to contain the raw RSS XML string. If n8n’s HTTP node returns a different structure (or if “Download” options differ), parsing will fail.

#### Node: **Parse RSS Feed** (Code)
- **Role:** Extracts titles and links from the RSS XML and formats them for the AI agent.
- **Config choices (interpreted):**
  - Reads `const rssData = $node["2. Get Latest News"].json.data;`
  - Regex matches `<title>...</title>` and `<link>...</link>`
  - Skips index 0 (RSS feed title) by starting loop at `i = 1`
  - Takes up to 10 items (`Math.min(..., 11)`)
  - Outputs a single object with `newsDataForAI` containing items separated by `---`
- **Outputs:** To **3. AI: Keyword Selection**.
- **Failure/edge cases:**
  - If RSS format changes (CDATA, namespaces, different ordering), regex may mis-extract.
  - If `json.data` is undefined, code throws.
  - Titles/links counts may differ; current code uses `Math.min(titles.length, 11)` but indexes `links[i]`—can throw if fewer links than titles.

#### Node: **OpenAI Chat Model** (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)
- **Role:** Provides the LLM backing model for the keyword-selection agent.
- **Config choices:** Model set to **`gpt-4.1-mini`** using OpenAI credentials (“OpenAi account 4”).
- **Connections:** Feeds the **AI languageModel** input of **3. AI: Keyword Selection**.
- **Failure/edge cases:**
  - Missing/invalid OpenAI credentials.
  - Model availability changes or account rate limits.

#### Node: **3. AI: Keyword Selection** (LangChain Agent)
- **Role:** Produces 3 SEO keywords + reasons + a title from the parsed news list.
- **Config choices:**
  - Prompt includes: `Latest News: {{ $json.newsDataForAI }}`
  - System message enforces output rules and requires final line: `Title: [Your Proposed Title]`
- **Inputs:**
  - Main input from **Parse RSS Feed**
  - LLM from **OpenAI Chat Model**
- **Outputs:** To **Store Resume Key**.
- **Failure/edge cases:**
  - If the agent does not include `Title:` line, later title extraction may fail or create “Untitled Article”.
  - Very long RSS content could exceed token limits; current feed is limited to 10 items which helps.

#### Node: **Store Resume Key** (Google Sheets)
- **Role:** Persists the current execution’s **resume URL** keyed by `lineUserId` so a LINE command can resume the workflow later.
- **Config choices:**
  - Operation: **appendOrUpdate**
  - Matching column: `userId`
  - Writes:
    - `userId` = `{{ $('Workflow Configuration').item.json.lineUserId }}`
    - `resumeurl` = `{{ $execution.resumeUrl }}`
  - Targets a specific sheet tab by `gid=0` (intended to be `Approval_Keys` per sticky note, but here it’s configured by gid).
- **Inputs:** From **3. AI: Keyword Selection**.
- **Outputs:** To **4. LINE: KW Approval Request**.
- **Failure/edge cases:**
  - Google Sheets credentials/scopes missing.
  - Sheet structure mismatch (missing `userId` / `resumeurl` headers) breaks mapping.
  - Multiple users: using `lineUserId` as key supports per-user resume URLs, but only if lookup later uses the same key.

#### Node: **4. LINE: KW Approval Request** (HTTP Request)
- **Role:** Pushes the keyword proposal to LINE as a text message.
- **Config choices:**
  - POST `https://api.line.me/v2/bot/message/push`
  - Body includes:
    - `"to"` = configured LINE user id
    - Message text includes agent output and asks to reply with “Create Article”
  - Authentication: **genericCredentialType → httpHeaderAuth** (expects Authorization header like `Bearer {channel_access_token}`)
- **Outputs:** To **5. Wait: KW Approval**.
- **Failure/edge cases:**
  - Invalid LINE token or missing header → 401/403.
  - LINE push restrictions (user not friended bot, bot not allowed to push).
  - The message includes an emoji; if LINE or downstream parsing expects ASCII only, could be an issue (rare).

#### Node: **5. Wait: KW Approval** (Wait)
- **Role:** Pauses execution until resumed via webhook resume URL.
- **Config choices:**
  - Resume mode: **webhook**
  - Returns response text on resume: “Approved. Generating outline...”
- **Outputs:** To **6. AI: Outline Creation**.
- **Failure/edge cases:**
  - If resume URL is lost/not stored, run remains stuck.
  - If resumed multiple times, behavior depends on n8n wait node semantics (usually one resume per waiting execution).
  - If webhook URL is exposed, anyone with the URL could resume (treat resume URLs as secrets).

---

### Block 2 — LINE Webhook Receiver & Resume Logic (Phase 2)

**Overview:**  
Receives incoming LINE webhook events, checks if the message equals “Create Article”, looks up the resume URL for that user, and calls it to resume the paused workflow run.

**Nodes Involved:**
- LINE Webhook Receiver
- Check if 'Create Article'
- Get Resume URL from Sheets
- Resume Main Workflow

#### Node: **LINE Webhook Receiver** (Webhook)
- **Role:** Secondary entry point for incoming LINE events.
- **Config choices:**
  - POST endpoint path: `line-webhook-receiver`
- **Outputs:** To **Check if 'Create Article'**.
- **Failure/edge cases:**
  - LINE signature verification is not implemented here (recommended for security).
  - Payload structure assumptions (expects `body.events[0].message.text`).
  - If LINE sends non-message events (follow/unfollow, postback), `message.text` may not exist.

#### Node: **Check if 'Create Article'** (IF)
- **Role:** Filters webhook events; only proceeds when the exact text equals `Create Article`.
- **Config choices:**
  - Condition: `{{ $json.body.events[0].message.text }}` equals `"Create Article"`
- **Outputs:** True branch → **Get Resume URL from Sheets** (false branch is unused).
- **Failure/edge cases:**
  - Case sensitivity: “create article” will not match.
  - LINE messages with leading/trailing spaces won’t match unless trimmed.
  - Missing `events[0]` causes expression errors.

#### Node: **Get Resume URL from Sheets** (Google Sheets)
- **Role:** Retrieves the stored resume URL.
- **Config choices:** Operation: **lookup** on the same document id (hardcoded in this node).
- **Important note:** The node configuration in the JSON does not show:
  - which sheet/tab is used
  - which lookup column/value is used  
  These must be set in the UI for a functional lookup (typically lookup by `userId` = LINE sender user id).
- **Outputs:** To **Resume Main Workflow**.
- **Failure/edge cases:**
  - If the lookup key does not match stored `userId`, no row found → resume URL missing.
  - Using a hardcoded documentId here while the rest uses `sheetId` can cause inconsistency if changed.

#### Node: **Resume Main Workflow** (HTTP Request)
- **Role:** Calls the stored resume URL to resume the waiting execution.
- **Config choices:** GET `={{ $json.resumeurl }}`
- **Failure/edge cases:**
  - If `resumeurl` is empty/undefined → invalid URL error.
  - Resume URL expired or execution no longer waiting → 404/410 depending on n8n version/config.
  - If the sheet row contains multiple resume URLs (race), you may resume an older execution unintentionally.

---

### Block 3 — Outline Approval + Deep-Dive Recursive Writing Loop (Phase 3)

**Overview:**  
After keyword approval, the workflow generates an SEO outline, requests approval via LINE again, then creates a Google Doc and writes the article section-by-section. It appends each chapter to the same Doc, logs the final output to Sheets, and notifies via LINE.

**Nodes Involved:**
- OpenAI Chat Model1
- 6. AI: Outline Creation
- 7. LINE: Outline Approval
- 8. Wait: Outline Approval
- Create Google Doc
- Split Outline to Chapters
- Chapter Loop
- OpenAI Chat Model2
- 9. AI: Chapter Writing
- Write Chapter to Docs
- 11. Log Final Article to Sheets
- 12. LINE: Final Notification

#### Node: **OpenAI Chat Model1** (OpenAI Chat Model)
- **Role:** LLM for outline generation.
- **Config choices:** `gpt-4.1-mini` with same OpenAI credential.
- **Connections:** Provides **ai_languageModel** to **6. AI: Outline Creation**.
- **Failure/edge cases:** Same as other OpenAI model nodes.

#### Node: **6. AI: Outline Creation** (LangChain Agent)
- **Role:** Creates a structured outline (H2/H3) from the approved keywords/title proposal.
- **Config choices:**
  - Prompt references keyword selection output:
    - `{{ $('3. AI: Keyword Selection').item.json.output }}`
  - System: “Include 3-5 H2 sections with nested H3 sub-points.”
- **Outputs:** To **7. LINE: Outline Approval**.
- **Failure/edge cases:**
  - If outline doesn’t use `##` headings, downstream splitting may fail (the splitter uses `/((?=##+ ))/`).
  - If keyword-selection output is verbose or malformed, outline quality may degrade.

#### Node: **7. LINE: Outline Approval** (HTTP Request)
- **Role:** Sends the generated outline to LINE and asks for “Create Article” again.
- **Config choices:** POST to LINE push API with message text `📋 Outline is ready:\n\n + $json.output`.
- **Important:** This node’s JSON does **not** show authentication settings (unlike the KW request node). In n8n, it must also include the same Authorization header.
- **Outputs:** To **8. Wait: Outline Approval**.
- **Failure/edge cases:**
  - If authentication not set, request will fail.
  - Outline may exceed LINE message length limits (LINE has constraints; long outlines can be truncated or rejected).

#### Node: **8. Wait: Outline Approval** (Wait)
- **Role:** Pauses again until resumed, confirming outline approval.
- **Config choices:** Resume via webhook; response “Approved. Starting deep-dive writing loop...”
- **Outputs:** To **Create Google Doc**.

#### Node: **Create Google Doc** (Google Docs)
- **Role:** Creates the destination document for the article.
- **Config choices:**
  - Title expression:
    - `{{ $('3. AI: Keyword Selection').item.json.output.split('Title:')[1] || 'Untitled Article' }}`
  - Folder: `default` (root or default drive location)
- **Failure/edge cases:**
  - If AI output does not include `Title:`, doc title becomes “Untitled Article”.
  - `split('Title:')[1]` may include leading whitespace/newlines; you may want `.trim()` to clean it.
  - Google Docs permissions/credentials required.

#### Node: **Split Outline to Chapters** (Code)
- **Role:** Turns the outline text into an array of sections to be written one-by-one.
- **Config choices:**
  - Reads outline from `6. AI: Outline Creation` output
  - Splits with: `outline.split(/(?=##+ )/g)`
  - Produces items: `{ section_content: '...' }` per section
- **Outputs:** To **Chapter Loop**.
- **Failure/edge cases:**
  - If headings are not `## `-prefixed, you may get a single large section.
  - If outline includes `###` without `##`, split may behave unexpectedly.

#### Node: **Chapter Loop** (Split In Batches)
- **Role:** Iterates through chapter items sequentially (batch loop) to avoid generating all chapters at once.
- **Config choices:** Default batch options (batch size not shown; defaults apply unless set).
- **Connections (important):**
  - One output goes to **9. AI: Chapter Writing** (per batch item)
  - Another output goes to **11. Log Final Article to Sheets** (this is the “noItemsLeft/next” style path used when the loop completes)
- **Failure/edge cases:**
  - If batch size too large, you may hit rate limits.
  - If the loop is not correctly wired, it may log final article too early or never.

#### Node: **OpenAI Chat Model2** (OpenAI Chat Model)
- **Role:** LLM for chapter writing.
- **Config choices:** `gpt-4.1-mini`.
- **Connections:** Supplies **ai_languageModel** to **9. AI: Chapter Writing**.

#### Node: **9. AI: Chapter Writing** (LangChain Agent)
- **Role:** Writes one chapter (400–600 words) in HTML for the current outline section.
- **Config choices:**
  - Prompt: “write the full content for this specific chapter: {{ $json.section_content }}”
  - System constraints: professional SEO copywriter, 400–600 words, HTML tags `<p>`, `<strong>`, `<ul>`
- **Outputs:** To **Write Chapter to Docs**.
- **Failure/edge cases:**
  - LLM may not strictly respect word count or tag constraints.
  - If `section_content` is empty, output may be generic filler.

#### Node: **Write Chapter to Docs** (Google Docs - Update/Insert)
- **Role:** Appends/inserts the chapter HTML into the created Google Doc.
- **Config choices:**
  - Operation: **update**
  - Action: **insert**
  - (Document ID and insert location are not visible in the JSON snippet; in the UI, it must reference the Doc created earlier, typically via `{{ $('Create Google Doc').item.json.id }}`.)
- **Outputs:** Back to **Chapter Loop** to continue with the next section.
- **Failure/edge cases:**
  - If doc ID is not correctly mapped, inserts will fail.
  - Google Docs API can reject malformed HTML; you may need to sanitize or insert as plain text.

#### Node: **11. Log Final Article to Sheets** (Google Sheets - Append)
- **Role:** After the loop completes, records article metadata to `Article_History`.
- **Config choices:**
  - Appends columns:
    - `URL` = `https://docs.google.com/document/d/{{ docId }}/edit`
    - `Date` = `{{ $now.format('YYYY-MM-DD') }}`
    - `Title` = `{{ $('Create Google Doc').item.json.name }}`
  - DocumentId: `={{ $('Workflow Configuration').item.json.sheetId }}`
  - Sheet name: `Article_History`
- **Outputs:** To **12. LINE: Final Notification**.
- **Failure/edge cases:**
  - If the loop never reaches completion branch, logging won’t happen.
  - Headers must match (`Title`, `URL`, `Date`) or mapping can fail.

#### Node: **12. LINE: Final Notification** (HTTP Request)
- **Role:** Sends final “deep writing complete” message with title + Google Doc URL.
- **Config choices:** POST to LINE push API with constructed URL from doc id.
- **Important:** Authentication is not shown in JSON; must be configured (Authorization header).
- **Failure/edge cases:**
  - If doc creation failed, `id/name` are missing → invalid message content.
  - LINE push failures (auth, permissions) prevent notification but article may still exist.

---

### Block 4 — Documentation/Annotations (Sticky Notes)

**Overview:**  
Sticky notes describe intent, setup requirements, and phase separation. They don’t affect execution but carry critical operational instructions.

**Nodes Involved:**
- Sticky Note - Overview
- Sticky Note - Phase 1
- Sticky Note - Phase 2
- Sticky Note - Phase 3

#### Node: **Sticky Note - Overview** (Sticky Note)
- **Role:** Describes the full concept, features, and setup steps (LINE webhook URL, Sheets tabs, config fields).

#### Node: **Sticky Note - Phase 1** (Sticky Note)
- **Role:** Explains the news discovery + AI strategy + storing resume URL bridge.

#### Node: **Sticky Note - Phase 2** (Sticky Note)
- **Role:** Explains webhook receiver and resuming logic.

#### Node: **Sticky Note - Phase 3** (Sticky Note)
- **Role:** Explains the chapter-based recursive writing approach and Google Docs appending.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| 1. Daily Schedule Trigger | scheduleTrigger | Scheduled entry point | — | Workflow Configuration | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| Workflow Configuration | set | Stores rssUrl, lineUserId, sheetId | 1. Daily Schedule Trigger | 2. Get Latest News | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| 2. Get Latest News | httpRequest | Fetches RSS XML | Workflow Configuration | Parse RSS Feed | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| Parse RSS Feed | code | Extracts titles/links and formats for AI | 2. Get Latest News | 3. AI: Keyword Selection | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| OpenAI Chat Model | lmChatOpenAi | LLM for keyword agent | — | 3. AI: Keyword Selection (ai_languageModel) | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| 3. AI: Keyword Selection | langchain.agent | Generates keywords + reasons + title | Parse RSS Feed | Store Resume Key | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| Store Resume Key | googleSheets | Saves $execution.resumeUrl keyed by userId | 3. AI: Keyword Selection | 4. LINE: KW Approval Request | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| 4. LINE: KW Approval Request | httpRequest | Pushes keyword proposal to LINE | Store Resume Key | 5. Wait: KW Approval | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| 5. Wait: KW Approval | wait | Pauses until resumed (approval gate) | 4. LINE: KW Approval Request | 6. AI: Outline Creation | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for outline agent | — | 6. AI: Outline Creation (ai_languageModel) | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| 6. AI: Outline Creation | langchain.agent | Builds H2/H3 outline | 5. Wait: KW Approval | 7. LINE: Outline Approval | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| 7. LINE: Outline Approval | httpRequest | Pushes outline to LINE | 6. AI: Outline Creation | 8. Wait: Outline Approval | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| 8. Wait: Outline Approval | wait | Pauses until outline approved | 7. LINE: Outline Approval | Create Google Doc | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| Create Google Doc | googleDocs | Creates target Doc | 8. Wait: Outline Approval | Split Outline to Chapters | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| Split Outline to Chapters | code | Splits outline into per-section items | Create Google Doc | Chapter Loop | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| Chapter Loop | splitInBatches | Iterates sections; triggers finalization when done | Split Outline to Chapters; Write Chapter to Docs | 9. AI: Chapter Writing; 11. Log Final Article to Sheets | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| OpenAI Chat Model2 | lmChatOpenAi | LLM for chapter writing | — | 9. AI: Chapter Writing (ai_languageModel) | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| 9. AI: Chapter Writing | langchain.agent | Writes one chapter in HTML | Chapter Loop | Write Chapter to Docs | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| Write Chapter to Docs | googleDocs | Appends chapter content to the Doc | 9. AI: Chapter Writing | Chapter Loop | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| 11. Log Final Article to Sheets | googleSheets | Writes Title/URL/Date to Article_History | Chapter Loop | 12. LINE: Final Notification | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| 12. LINE: Final Notification | httpRequest | Notifies final doc link | 11. Log Final Article to Sheets | — | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |
| LINE Webhook Receiver | webhook | Receives LINE events | — | Check if 'Create Article' | ## 2️⃣ LINE Webhook & Resume Logic<br>This section acts as the receiver for your mobile commands.<br>- **Trigger**: Listens for the text "Create Article" from your LINE.<br>- **Lookup**: Retrieves the unique Resume URL from Google Sheets.<br>- **Resume**: Signals the main workflow to continue the next phase. |
| Check if 'Create Article' | if | Filters approval command | LINE Webhook Receiver | Get Resume URL from Sheets | ## 2️⃣ LINE Webhook & Resume Logic<br>This section acts as the receiver for your mobile commands.<br>- **Trigger**: Listens for the text "Create Article" from your LINE.<br>- **Lookup**: Retrieves the unique Resume URL from Google Sheets.<br>- **Resume**: Signals the main workflow to continue the next phase. |
| Get Resume URL from Sheets | googleSheets | Looks up resumeurl | Check if 'Create Article' | Resume Main Workflow | ## 2️⃣ LINE Webhook & Resume Logic<br>This section acts as the receiver for your mobile commands.<br>- **Trigger**: Listens for the text "Create Article" from your LINE.<br>- **Lookup**: Retrieves the unique Resume URL from Google Sheets.<br>- **Resume**: Signals the main workflow to continue the next phase. |
| Resume Main Workflow | httpRequest | Calls resume URL to continue waiting execution | Get Resume URL from Sheets | — | ## 2️⃣ LINE Webhook & Resume Logic<br>This section acts as the receiver for your mobile commands.<br>- **Trigger**: Listens for the text "Create Article" from your LINE.<br>- **Lookup**: Retrieves the unique Resume URL from Google Sheets.<br>- **Resume**: Signals the main workflow to continue the next phase. |
| Sticky Note - Overview | stickyNote | Project notes & setup requirements | — | — | # 🤖 AI-Powered SEO Content Factory (LINE Approval & Recursive Loop)<br><br>This high-end workflow automates the editorial pipeline from niche news discovery to long-form, deep-dive SEO articles.<br><br>## 🚀 Key Features<br>1. **Smart Sourcing**: Aggregates trending news via Google News RSS.<br>2. **AI Strategy**: Proposes SEO keywords and titles using OpenAI GPT-4o.<br>3. **Human-in-the-Loop**: Two-stage manual approval via LINE mobile commands.<br>4. **Recursive Loop Writing**: Splits articles into chapters for superior depth (400-600 words per section) instead of a single output.<br>5. **Automated Archiving**: Finalizes content in Google Docs and logs metadata to Sheets.<br><br>## ⚙️ Setup Steps<br>1. **Configuration**: Fill in your `lineUserId` and `sheetId` in the 'Workflow Configuration' node.<br>2. **Google Sheets**: Create a sheet with two tabs: `Approval_Keys` (userId, resumeurl) and `Article_History` (Title, URL, Date).<br>3. **LINE Setup**: Set your n8n Production URL in the LINE Developers Console and enable Webhooks.<br>4. **Activate**: Toggle to 'Active' to start the daily automation. |
| Sticky Note - Phase 1 | stickyNote | Phase annotation | — | — | ## 1️⃣ News Discovery & AI Strategy<br>Monitors your niche and triggers the AI to propose a content strategy.<br>- **RSS**: Fetches the latest niche trends.<br>- **AI**: Generates SEO Keywords and Title candidates.<br>- **Bridge**: Stores the session URL in Google Sheets so you can approve via text message. |
| Sticky Note - Phase 2 | stickyNote | Phase annotation | — | — | ## 2️⃣ LINE Webhook & Resume Logic<br>This section acts as the receiver for your mobile commands.<br>- **Trigger**: Listens for the text "Create Article" from your LINE.<br>- **Lookup**: Retrieves the unique Resume URL from Google Sheets.<br>- **Resume**: Signals the main workflow to continue the next phase. |
| Sticky Note - Phase 3 | stickyNote | Phase annotation | — | — | ## 3️⃣ Deep-Dive Recursive Writing Loop<br>To prevent "AI Fatigue" and surface-level content, this phase loops through the outline:<br>- **Chapter Writing**: Focuses on one specific H2/H3 at a time.<br>- **Google Docs**: Appends content progressively to a single document.<br>- **Staggered Flow**: Ensures a high word count and quality structure. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named:  
   **Generate long-form SEO articles from news with LINE approvals and Google Docs**

2. **Add node: Schedule Trigger**  
   - Name: `1. Daily Schedule Trigger`  
   - Configure a daily time or fixed interval (ensure it’s not blank).

3. **Add node: Set**  
   - Name: `Workflow Configuration`  
   - Add string fields:
     - `rssUrl` (Google News RSS query URL)
     - `lineUserId` (your LINE user ID to push messages to)
     - `sheetId` (Google Sheets document ID)
   - Connect: Trigger → Set

4. **Add node: HTTP Request**  
   - Name: `2. Get Latest News`  
   - Method: GET  
   - URL: `{{ $json.rssUrl }}`  
   - Connect: Set → HTTP Request

5. **Add node: Code**  
   - Name: `Parse RSS Feed`  
   - Paste logic to extract `<title>` and `<link>` entries and output `newsDataForAI`.  
   - Connect: HTTP Request → Code

6. **Add node: OpenAI Chat Model (LangChain)**  
   - Name: `OpenAI Chat Model`  
   - Model: `gpt-4.1-mini`  
   - Credentials: configure OpenAI API key credentials in n8n.

7. **Add node: AI Agent (LangChain Agent)**  
   - Name: `3. AI: Keyword Selection`  
   - Prompt includes `{{ $json.newsDataForAI }}`  
   - System message enforces final line `Title: ...`  
   - Connect main: Parse RSS Feed → Agent  
   - Connect AI model: OpenAI Chat Model → Agent (language model input)

8. **Add node: Google Sheets**  
   - Name: `Store Resume Key`  
   - Credentials: Google (OAuth2/service account with Sheets access)  
   - Operation: `appendOrUpdate`  
   - Match column: `userId`  
   - Values:
     - `userId` = `{{ $('Workflow Configuration').item.json.lineUserId }}`
     - `resumeurl` = `{{ $execution.resumeUrl }}`
   - Select the correct tab (recommended: `Approval_Keys`).  
   - Connect: Keyword Selection → Store Resume Key

9. **Add node: HTTP Request** (LINE push message)  
   - Name: `4. LINE: KW Approval Request`  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/push`  
   - Authentication: Header Auth  
     - Header: `Authorization: Bearer <LINE_CHANNEL_ACCESS_TOKEN>`  
   - Body: JSON including `to` and `messages[0].text` that embeds the AI output and asks to reply “Create Article”.  
   - Connect: Store Resume Key → LINE node

10. **Add node: Wait**  
    - Name: `5. Wait: KW Approval`  
    - Resume: `webhook`  
    - Response on resume: “Approved. Generating outline...”  
    - Connect: LINE KW Request → Wait

11. **Add node: OpenAI Chat Model (LangChain)**  
    - Name: `OpenAI Chat Model1`  
    - Model: `gpt-4.1-mini`  
    - Same OpenAI credentials.

12. **Add node: AI Agent**  
    - Name: `6. AI: Outline Creation`  
    - Prompt references `{{ $('3. AI: Keyword Selection').item.json.output }}`  
    - System message: require 3–5 H2 with nested H3  
    - Connect main: Wait KW Approval → Outline Agent  
    - Connect AI model: OpenAI Chat Model1 → Outline Agent

13. **Add node: HTTP Request** (LINE outline push)  
    - Name: `7. LINE: Outline Approval`  
    - Same LINE POST endpoint  
    - Ensure Authorization header is configured  
    - Message embeds `{{ $json.output }}` and asks to reply “Create Article”  
    - Connect: Outline Agent → LINE Outline Approval

14. **Add node: Wait**  
    - Name: `8. Wait: Outline Approval`  
    - Resume: webhook  
    - Response: “Approved. Starting deep-dive writing loop...”  
    - Connect: LINE Outline Approval → Wait

15. **Add node: Google Docs**  
    - Name: `Create Google Doc`  
    - Operation: Create  
    - Title: `{{ $('3. AI: Keyword Selection').item.json.output.split('Title:')[1] || 'Untitled Article' }}` (recommended to wrap with `.trim()`)  
    - Folder: choose target folder or default  
    - Connect: Wait Outline Approval → Create Doc

16. **Add node: Code**  
    - Name: `Split Outline to Chapters`  
    - Split outline by `##` sections and output items `{ section_content }`  
    - Connect: Create Doc → Split Outline

17. **Add node: Split In Batches**  
    - Name: `Chapter Loop`  
    - Batch size: 1 (recommended for controlled pacing)  
    - Connect: Split Outline → Split In Batches

18. **Add node: OpenAI Chat Model (LangChain)**  
    - Name: `OpenAI Chat Model2`  
    - Model: `gpt-4.1-mini`

19. **Add node: AI Agent**  
    - Name: `9. AI: Chapter Writing`  
    - Prompt uses `{{ $json.section_content }}`  
    - System message requests 400–600 words and HTML tags  
    - Connect main: Chapter Loop → Chapter Writing  
    - Connect AI model: OpenAI Chat Model2 → Chapter Writing

20. **Add node: Google Docs (Update → Insert)**  
    - Name: `Write Chapter to Docs`  
    - Operation: Update  
    - Action: Insert  
    - Document ID: map from `Create Google Doc` (typically `{{ $('Create Google Doc').item.json.id }}`)  
    - Content to insert: map from chapter writing output (typically `{{ $json.output }}`)  
    - Connect: Chapter Writing → Write Chapter to Docs  
    - Connect: Write Chapter to Docs → Chapter Loop (to continue next batch)

21. **Finalize path after loop completion:**  
    - From `Chapter Loop`, connect the “done/no items left” output to logging:
      - **Google Sheets** node `11. Log Final Article to Sheets`
        - Operation: Append
        - Sheet: `Article_History`
        - Document ID: `{{ $('Workflow Configuration').item.json.sheetId }}`
        - Values:
          - Title = `{{ $('Create Google Doc').item.json.name }}`
          - URL = `https://docs.google.com/document/d/{{ $('Create Google Doc').item.json.id }}/edit`
          - Date = `{{ $now.format('YYYY-MM-DD') }}`
      - Then connect to **HTTP Request** node `12. LINE: Final Notification` (same LINE auth) that sends the Doc link.

22. **Create the approval receiver workflow path (second entry):**
    - Add **Webhook** node: `LINE Webhook Receiver`
      - Path: `line-webhook-receiver`
      - Method: POST
    - Add **IF** node: `Check if 'Create Article'`
      - Condition: `{{ $json.body.events[0].message.text }}` equals `Create Article`
    - Add **Google Sheets** node: `Get Resume URL from Sheets`
      - Operation: Lookup
      - Document: the same Google Sheet used in step 8
      - Lookup by `userId` (value should come from LINE event sender user id; you must map it from the webhook payload)
      - Return `resumeurl`
    - Add **HTTP Request** node: `Resume Main Workflow`
      - URL: `{{ $json.resumeurl }}`
    - Connect: Webhook → IF → Sheets Lookup → Resume HTTP

23. **LINE Developer Console configuration**
    - Enable webhooks and set webhook URL to your n8n production URL + `/webhook/line-webhook-receiver`
    - Ensure the bot can **push** messages to your user (bot friendship/permissions).

24. **Google Sheets preparation**
    - Create tabs:
      - `Approval_Keys`: columns `userId`, `resumeurl`
      - `Article_History`: columns `Title`, `URL`, `Date`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI-Powered SEO Content Factory (LINE Approval & Recursive Loop)” including features and setup steps | From Sticky Note - Overview |
| Requires Google Sheets tabs: `Approval_Keys` (userId, resumeurl) and `Article_History` (Title, URL, Date) | From Sticky Note - Overview |
| LINE setup: set n8n Production URL in LINE Developers Console and enable Webhooks | From Sticky Note - Overview |
| Two-stage approval via LINE message “Create Article” | From Sticky Notes Phase 1/2/3 |
| Disclaimer: *Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n…* | User-provided disclaimer context |

