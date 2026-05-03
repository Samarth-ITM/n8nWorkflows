Generate Japanese Twitter posts with GPT-4, quality control, and Notion

https://n8nworkflows.xyz/workflows/generate-japanese-twitter-posts-with-gpt-4--quality-control--and-notion-13003


# Generate Japanese Twitter posts with GPT-4, quality control, and Notion

## 1. Workflow Overview

**Title:** Generate Japanese Twitter posts with GPT-4, quality control, and Notion

This n8n workflow automates Japanese-language Twitter (X) post generation using GPT‑4, applies AI-based quality scoring and cultural/risk analysis, then routes posts for **auto-publishing** or **human approval**, while storing outcomes in **Notion**. It also produces a **weekly analytics email report** based on Notion-stored performance data.

### 1.1 Daily Content Generation (Weekdays 9:00 JST)
- Triggered Monday–Friday at 9:00 (cron).
- Generates 3 Japanese post options via GPT‑4.
- Parses them into individual items for downstream checks.

### 1.2 Quality Scoring + Self-Improvement Loop
- Each post is scored by GPT‑4 across 5 dimensions (100-point scale).
- If score < 70, it enters an improvement loop (up to 3 rewrites), rescoring after each rewrite.

### 1.3 Risk/Sentiment Analysis + Routing
- GPT‑4 checks sentiment and “flame/risk” factors for Japanese cultural suitability.
- Routing outcomes:
  - **Auto-approve:** post immediately and save as published.
  - **Requires approval:** save as pending + email the team.
  - **Auto-reject:** (intended per sticky note) save for review; however, the current JSON does not implement a dedicated reject branch.

### 1.4 Human Approval Webhook
- Separate entry point via `/approval-webhook` to approve/reject/edit content by Notion ID.
- Updates Notion status accordingly (approval vs rejection).
- **Important:** The workflow currently updates Notion but does **not** post approved items to Twitter on webhook approval (missing node/connection).

### 1.5 Weekly Analytics (Mondays 10:00)
- Pulls the last week’s posts from Notion.
- GPT‑4 produces a Markdown report.
- Emails the report.

---

## 2. Block-by-Block Analysis

### Block A — Documentation / Visual Guidance (Sticky Notes)
**Overview:** Sticky notes describe intended behavior, thresholds, and integration expectations. They do not execute logic but are critical to reproduce the intended design.

**Nodes involved:**
- Sticky Note - Main Overview
- Sticky Note - Content Generation
- Sticky Note - Brand Voice
- Sticky Note - Quality Scoring
- Sticky Note - Improvement
- Sticky Note - Risk Analysis
- Sticky Note - Routing
- Sticky Note - Auto Approve
- Sticky Note - Approval Path
- Sticky Note - Webhook
- Sticky Note - Analytics

**Node details (all Sticky Note nodes)**
- **Type/role:** `stickyNote` — documentation only.
- **Configuration choices:** position, size, color, and Markdown content.
- **Connections:** none (not executable).
- **Edge cases:** none at runtime, but notes may describe behavior not implemented (notably: reject handling and webhook-approved Twitter posting).

---

### Block B — Daily Trigger & Content Generation
**Overview:** Runs on weekdays; generates 3 culturally-aware Japanese tweets using GPT‑4 and prepares them as separate n8n items.

**Nodes involved:**
1. Schedule Daily Content Generation
2. Generate Content with GPT-4
3. Parse Generated Content

#### 1) Schedule Daily Content Generation
- **Type/role:** `scheduleTrigger` — starts daily pipeline.
- **Configuration:** Cron `0 9 * * 1-5` (Mon–Fri at 09:00). Note: cron is interpreted in the n8n instance timezone unless configured otherwise; sticky notes assume **JST**.
- **Outputs:** sends one execution signal to three nodes in parallel:
  - Generate Content with GPT-4
  - Get Japanese Cultural Context
  - Get Past 30 Days Posts
- **Edge cases:** timezone mismatch (posting at wrong local time).

#### 2) Generate Content with GPT-4
- **Type/role:** `httpRequest` — calls OpenAI Chat Completions API.
- **Authentication:** “genericCredentialType” with `httpHeaderAuth` (requires an Authorization header in credentials; node itself sets Content-Type only).
- **Request body:** model `gpt-4`, temperature 0.7, max_tokens 1000.
- **Prompt behavior:** system role sets Japan-market expert; user requests **3 posts** returned as a JSON array of objects `{content, hashtags}`; includes date via `$now.format('yyyy年MM月dd日(ddd)')`.
- **Outputs:** raw OpenAI response JSON (`choices[0].message.content` is expected to be JSON text).
- **Edge cases/failures:**
  - Missing/invalid OpenAI credential header (`401`).
  - Model access denied (`403`) or deprecated model naming.
  - Response not valid JSON (breaks downstream parsing).
  - Rate limits (`429`) / timeouts.

#### 3) Parse Generated Content
- **Type/role:** `code` — parses GPT output into 3 separate n8n items.
- **Key logic:**
  - `JSON.parse(response.choices[0].message.content)`
  - Emits items with fields:
    - `content`, `hashtags`, `generatedAt`, `platform: 'twitter'`, `status: 'pending_quality_check'`, `version: i+1`
- **Inputs:** OpenAI response from “Generate Content with GPT-4”.
- **Outputs:** multiple items (one per proposed post) to “AI Quality Scoring”.
- **Edge cases:**
  - If GPT returns markdown fences or non-JSON, JSON.parse throws.
  - If `choices[0]` missing (API error format), expression fails.

---

### Block C — Cultural Context & Brand Voice Learning (Currently Not Consumed)
**Overview:** Collects cultural context and attempts to learn brand voice from the last 30 days of posts. In the current workflow wiring, these outputs are not used to condition the generated content.

**Nodes involved:**
1. Get Japanese Cultural Context
2. Get Past 30 Days Posts
3. Analyze Brand Voice
4. Create Brand Voice Profile

#### 1) Get Japanese Cultural Context
- **Type/role:** `httpRequest` to OpenAI — returns JSON describing season/holidays/business events and “avoid topics”.
- **Trigger:** runs in parallel from the daily schedule trigger.
- **Important:** No downstream connections in the JSON; result is unused.
- **Edge cases:** same OpenAI risks as other calls; unused outputs may confuse operators.

#### 2) Get Past 30 Days Posts (Notion)
- **Type/role:** `notion` with `getAll`.
- **Purpose (intended):** fetch posts for last 30 days to infer tone/hashtags/etc.
- **Important:** no filter is configured in JSON; it retrieves “all” from the configured resource scope, unless your Notion node default includes a database internally (not shown here).
- **Output:** items representing Notion records.
- **Edge cases:** missing Notion credentials, missing database/page scope, pagination volume, property name mismatches.

#### 3) Analyze Brand Voice
- **Type/role:** `code` — computes summary stats from Notion items.
- **Key logic:**
  - Collects content from `p.json.Content || p.json.content`
  - Computes `averageLength`, `emojiCount` (unicode emoji range), question count, hashtag usage
  - Outputs `topHashtags` and `brandVoiceAnalysis`
- **Edge cases:**
  - Notion property naming often differs (rich text objects); the node assumes plain strings/arrays already exist.
  - Emoji regex range may miss some emoji blocks.

#### 4) Create Brand Voice Profile
- **Type/role:** `httpRequest` to OpenAI — turns stats into a JSON “brand voice profile”.
- **Input:** output of “Analyze Brand Voice”.
- **Important:** No downstream connections; profile is unused by “Generate Content with GPT‑4”.
- **Edge cases:** JSON parsing would be needed if later consumed; currently not implemented.

---

### Block D — Quality Scoring
**Overview:** Each generated post is scored by GPT‑4 and merged with the original post data, producing a normalized scoring object.

**Nodes involved:**
1. AI Quality Scoring
2. Parse Quality Scores

#### 1) AI Quality Scoring
- **Type/role:** `httpRequest` to OpenAI — asks for a JSON scoring payload including risk_level and risk_reason.
- **Inputs:** Each post item from “Parse Generated Content” or “Parse Improved Content”.
- **Prompt:** includes `$json.content` and `$json.hashtags.join(', ')`.
- **Temperature:** 0.3 (more deterministic).
- **Output:** OpenAI response with a JSON string in `choices[0].message.content`.
- **Edge cases:**
  - If hashtags missing or not an array, `.join` will fail.
  - GPT may output non-strict JSON (trailing commas, code fences).

#### 2) Parse Quality Scores
- **Type/role:** `code` — parses score JSON and merges it with the original content.
- **Key logic:**
  - `scores = JSON.parse(scoringResponse.choices[0].message.content)`
  - `originalData = $('Parse Generated Content').item.json`
  - Outputs fields: `qualityScore`, sub-scores, `feedback`, `riskLevel`, `riskReason`, `needsImprovement`, `improvementAttempts: 0`
- **Connections:** to “Check if Improvement Needed”.
- **Critical edge case (data lineage bug):**
  - When scoring content that came from **Parse Improved Content**, this node still pulls `$('Parse Generated Content').item.json`, which can mismatch the current item or revert to the initial version.
  - This undermines the improvement loop correctness and can cause wrong content to be evaluated/routed.
- **Other edge cases:** JSON.parse failures; missing `choices`.

---

### Block E — Self-Improvement Loop (Up to 3 Attempts)
**Overview:** If quality score < 70 and attempts < 3, GPT‑4 rewrites the post based on feedback, then the new version is rescored.

**Nodes involved:**
1. Check if Improvement Needed
2. Auto-Improve Content
3. Parse Improved Content
(then loops back into AI Quality Scoring)

#### 1) Check if Improvement Needed
- **Type/role:** `if`
- **Conditions (AND):**
  - `$json.needsImprovement == true`
  - `$json.improvementAttempts < 3`
- **True output:** Auto-Improve Content
- **False output:** Sentiment & Risk Analysis
- **Edge cases:** `improvementAttempts` must exist and be numeric; strict validation is enabled.

#### 2) Auto-Improve Content
- **Type/role:** `httpRequest` to OpenAI — rewrite request.
- **Prompt inputs:** original content, qualityScore, feedback, risk level/reason.
- **Expected output:** JSON `{content, hashtags}`.
- **Edge cases:** same JSON cleanliness issues.

#### 3) Parse Improved Content
- **Type/role:** `code` — parse improved JSON and increment attempts.
- **Key logic:**
  - `improved = JSON.parse(improveResponse.choices[0].message.content)`
  - `originalData = $('Parse Quality Scores').item.json`
  - Outputs: improved content, hashtags, `improvementAttempts + 1`, keeps `version`, stores `previousScore` & `previousFeedback`.
- **Output connection:** AI Quality Scoring (loop).
- **Edge cases:** if the workflow processes multiple items concurrently, referencing `$('Parse Quality Scores').item.json` can still be ambiguous; prefer using current item data rather than cross-node item selection.

---

### Block F — Sentiment & Risk Analysis + Final Decision
**Overview:** Performs a second, dedicated risk/sentiment/cultural appropriateness analysis and computes `finalRiskAssessment` used for routing.

**Nodes involved:**
1. Sentiment & Risk Analysis
2. Merge Risk Analysis
3. Decision Routing

#### 1) Sentiment & Risk Analysis
- **Type/role:** `httpRequest` to OpenAI — returns JSON with sentiment, risk level, risk factors, recommendations, and cultural appropriateness.
- **Input:** comes from “Check if Improvement Needed” false branch (i.e., after satisfactory score or max attempts reached).
- **Temperature:** 0.2.
- **Edge cases:** JSON formatting issues; model refusal; timeouts.

#### 2) Merge Risk Analysis
- **Type/role:** `code` — merges analysis with the quality-scored content and computes routing label.
- **Key logic:**
  - Parses analysis JSON from OpenAI response.
  - `contentData = $('Parse Quality Scores').item.json`
  - Computes:
    - `finalRiskAssessment`:
      - `auto_reject` if analysis risk high
      - `requires_approval` if analysis risk medium OR qualityScore < 70
      - otherwise `auto_approve`
- **Edge cases / design gaps:**
  - Still references `Parse Quality Scores` rather than current content item; can mismatch with improved iterations.
  - Uses risk from sentiment analysis, not necessarily the earlier scoring risk fields.

#### 3) Decision Routing
- **Type/role:** `if`
- **Condition:** `$json.finalRiskAssessment == 'auto_approve'`
- **True:** Save to Notion (Auto-Approved)
- **False:** Save to Notion (Pending Approval)
- **Major gap vs sticky notes:**
  - No explicit branch for `auto_reject`. High-risk content will follow the “false” path and be treated as pending approval, not rejected.

---

### Block G — Auto-Approved Publishing Path
**Overview:** Persists approved content to Notion and posts it to Twitter automatically.

**Nodes involved:**
1. Save to Notion (Auto-Approved)
2. Post to Twitter (Auto-Approved)

#### 1) Save to Notion (Auto-Approved)
- **Type/role:** `notion` — create/update page is implied, but the node is configured with `pageId` in URL mode and empty value; operation is not shown (defaults depend on node).
- **Input:** from Decision Routing true branch.
- **Edge cases:**
  - As configured, this node is incomplete: empty `pageId` likely fails, or it may require `databaseId` with “create” operation for proper storage.
  - Missing mapping to properties (Content, Hashtags, Status, etc.).
- **Output:** to Post to Twitter (Auto-Approved).

#### 2) Post to Twitter (Auto-Approved)
- **Type/role:** `httpRequest` to Twitter API v2 `/2/tweets`.
- **Authentication:** OAuth2 (`oAuth2Api`) via generic credential type.
- **Body:** `text = $json.content + ' ' + hashtags as #tag`.
- **Edge cases:**
  - Twitter/X character limit (280) may be exceeded after adding hashtags.
  - OAuth token scope issues (tweet.write).
  - API policy/rate limits, 429.
  - If hashtags contain spaces/symbols, may generate invalid tags.

---

### Block H — Pending Approval Path (Email Notification)
**Overview:** Saves pending content to Notion and notifies approvers by email.

**Nodes involved:**
1. Save to Notion (Pending Approval)
2. Send Approval Email

#### 1) Save to Notion (Pending Approval)
- **Type/role:** `notion` — similarly underconfigured with `pageId` empty and no visible property mapping.
- **Edge cases:** same as auto-approved Notion node.

#### 2) Send Approval Email
- **Type/role:** `emailSend`
- **Configuration:**
  - To: `user@example.com`
  - From: `user@example.com`
  - Subject includes expressions: qualityScore and riskLevel
- **Edge cases:**
  - SMTP/service credentials missing.
  - Email body is not configured; approvers may not receive content preview unless added.
  - Deliverability issues (SPF/DKIM), or “from” not allowed.

---

### Block I — Approval Webhook Handling (Separate Entry Point)
**Overview:** Receives approval actions and updates Notion accordingly.

**Nodes involved:**
1. Webhook - Approval Dashboard
2. Process Approval Action
3. Check Approval Action
4. Update Notion - Approved
5. Update Notion - Rejected

#### 1) Webhook - Approval Dashboard
- **Type/role:** `webhook`
- **Path:** `approval-webhook` (endpoint: `/approval-webhook`)
- **Input contract (per sticky note):**
  - Query: `action` = approve/reject/edit, `id` = Notion content ID
  - Body for edits: `{ editedContent, approvedBy }`
- **Edge cases:** missing query params; unauthenticated endpoint (should add auth or secret token).

#### 2) Process Approval Action
- **Type/role:** `code`
- **Behavior:**
  - Reads `query.action`, `query.id`, optional `body.editedContent`, `body.approvedBy`
  - Outputs normalized object with timestamp.
- **Edge cases:** webhook payload shape differs depending on n8n webhook settings (raw vs parsed); null-safe access is partly used.

#### 3) Check Approval Action
- **Type/role:** `if`
- **Condition:** `$json.action == 'approve'`
- **True:** Update Notion - Approved
- **False:** Update Notion - Rejected
- **Gap:** `edit` action is not handled explicitly (will go to “Rejected” branch).

#### 4) Update Notion - Approved / 5) Update Notion - Rejected
- **Type/role:** `notion` update operations.
- **Expected behavior:** update a Notion page record by ID and set Status/ApprovedBy/etc.
- **Edge cases:**
  - The node configuration in JSON does not show which page/property is updated; must be configured.
  - No subsequent Twitter posting step after approval (missing).

---

### Block J — Weekly Analytics Reporting
**Overview:** Weekly scheduled job pulls posts from Notion, has GPT‑4 generate a performance report, and emails it.

**Nodes involved:**
1. Weekly Performance Report Trigger
2. Get Past Week Posts
3. Generate Weekly Analytics Report
4. Send Weekly Report Email

#### 1) Weekly Performance Report Trigger
- **Type/role:** `scheduleTrigger`
- **Cron:** `0 10 * * 1` (Mondays 10:00; instance timezone).
- **Edge cases:** timezone mismatch.

#### 2) Get Past Week Posts
- **Type/role:** `notion` getAll.
- **Gap:** no filter to “past week” is shown; it may fetch everything.
- **Edge cases:** volume, pagination, property shapes.

#### 3) Generate Weekly Analytics Report
- **Type/role:** `httpRequest` to OpenAI.
- **Prompt:** sends JSON string of mapped Notion fields:
  - `content: item.json.Content`
  - `qualityScore: item.json['Quality Score']`
  - `engagement: item.json.Engagement || 0`
  - `platform: item.json.Platform`
- **Output:** Markdown report in `choices[0].message.content`.
- **Edge cases:** property names not matching Notion output; OpenAI JSON stringify may become huge (token limits).

#### 4) Send Weekly Report Email
- **Type/role:** `emailSend`
- **Subject:** includes date formatting.
- **Gap:** email body not configured (unless defaults exist); should include GPT output.
- **Edge cases:** mail credentials.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Content Generation | stickyNote | Documentation for daily generation block | — | — | ## 📅 Daily Content Generation Flow … |
| Sticky Note - Quality Scoring | stickyNote | Documentation for scoring model and threshold | — | — | ## 🎯 AI Quality Scoring System … |
| Sticky Note - Improvement | stickyNote | Documentation for rewrite loop | — | — | ## 🔄 Self-Improvement Loop … |
| Sticky Note - Risk Analysis | stickyNote | Documentation for risk detection criteria | — | — | ## ⚠️ Risk & Sentiment Analysis … |
| Sticky Note - Routing | stickyNote | Documentation for routing rules | — | — | ## 🚦 Smart Routing Decision … |
| Sticky Note - Auto Approve | stickyNote | Documentation for auto-approved path | — | — | ## ✅ Auto-Approved Path … |
| Sticky Note - Approval Path | stickyNote | Documentation for human approval path | — | — | ## 👤 Human Approval Path … |
| Sticky Note - Webhook | stickyNote | Documentation for approval webhook contract | — | — | ## 🔗 Approval Webhook … |
| Sticky Note - Analytics | stickyNote | Documentation for weekly reporting | — | — | ## 📊 Weekly Analytics Report … |
| Sticky Note - Brand Voice | stickyNote | Documentation for brand voice learning | — | — | ## 🎨 Brand Voice Learning … |
| Schedule Daily Content Generation | scheduleTrigger | Weekday trigger for daily pipeline | — | Generate Content with GPT-4; Get Japanese Cultural Context; Get Past 30 Days Posts | ## 📅 Daily Content Generation Flow … |
| Generate Content with GPT-4 | httpRequest | OpenAI generation of 3 posts | Schedule Daily Content Generation | Parse Generated Content | ## 📅 Daily Content Generation Flow … |
| Parse Generated Content | code | Split GPT array into items | Generate Content with GPT-4 | AI Quality Scoring | ## 📅 Daily Content Generation Flow … |
| AI Quality Scoring | httpRequest | OpenAI scoring and feedback | Parse Generated Content; Parse Improved Content | Parse Quality Scores | ## 🎯 AI Quality Scoring System … |
| Parse Quality Scores | code | Parse scores + merge with original | AI Quality Scoring | Check if Improvement Needed | ## 🎯 AI Quality Scoring System … |
| Check if Improvement Needed | if | Gate to improvement loop | Parse Quality Scores | Auto-Improve Content (true); Sentiment & Risk Analysis (false) | ## 🔄 Self-Improvement Loop … |
| Auto-Improve Content | httpRequest | OpenAI rewrite based on feedback | Check if Improvement Needed (true) | Parse Improved Content | ## 🔄 Self-Improvement Loop … |
| Parse Improved Content | code | Parse rewrite + increment attempts | Auto-Improve Content | AI Quality Scoring | ## 🔄 Self-Improvement Loop … |
| Sentiment & Risk Analysis | httpRequest | OpenAI sentiment/risk/cultural check | Check if Improvement Needed (false) | Merge Risk Analysis | ## ⚠️ Risk & Sentiment Analysis … |
| Merge Risk Analysis | code | Merge analysis + compute final route | Sentiment & Risk Analysis | Decision Routing | ## ⚠️ Risk & Sentiment Analysis … |
| Decision Routing | if | Route auto-approve vs pending | Merge Risk Analysis | Save to Notion (Auto-Approved) (true); Save to Notion (Pending Approval) (false) | ## 🚦 Smart Routing Decision … |
| Save to Notion (Auto-Approved) | notion | Persist approved content | Decision Routing (true) | Post to Twitter (Auto-Approved) | ## ✅ Auto-Approved Path … |
| Post to Twitter (Auto-Approved) | httpRequest | Publish tweet via Twitter API v2 | Save to Notion (Auto-Approved) | — | ## ✅ Auto-Approved Path … |
| Save to Notion (Pending Approval) | notion | Persist pending content | Decision Routing (false) | Send Approval Email | ## 👤 Human Approval Path … |
| Send Approval Email | emailSend | Notify approvers | Save to Notion (Pending Approval) | — | ## 👤 Human Approval Path … |
| Get Japanese Cultural Context | httpRequest | OpenAI context for season/holidays | Schedule Daily Content Generation | — | ## 📅 Daily Content Generation Flow … |
| Get Past 30 Days Posts | notion | Fetch historic posts for brand analysis | Schedule Daily Content Generation | Analyze Brand Voice | ## 🎨 Brand Voice Learning … |
| Analyze Brand Voice | code | Compute stats/top hashtags | Get Past 30 Days Posts | Create Brand Voice Profile | ## 🎨 Brand Voice Learning … |
| Create Brand Voice Profile | httpRequest | OpenAI brand voice profile JSON | Analyze Brand Voice | — | ## 🎨 Brand Voice Learning … |
| Webhook - Approval Dashboard | webhook | Entry point for approval actions | — | Process Approval Action | ## 🔗 Approval Webhook … |
| Process Approval Action | code | Normalize webhook query/body | Webhook - Approval Dashboard | Check Approval Action | ## 🔗 Approval Webhook … |
| Check Approval Action | if | Branch approve vs reject | Process Approval Action | Update Notion - Approved (true); Update Notion - Rejected (false) | ## 🔗 Approval Webhook … |
| Update Notion - Approved | notion | Update Notion record to approved | Check Approval Action (true) | — | ## 🔗 Approval Webhook … |
| Update Notion - Rejected | notion | Update Notion record to rejected | Check Approval Action (false) | — | ## 🔗 Approval Webhook … |
| Weekly Performance Report Trigger | scheduleTrigger | Weekly trigger for analytics | — | Get Past Week Posts | ## 📊 Weekly Analytics Report … |
| Get Past Week Posts | notion | Fetch posts for analytics | Weekly Performance Report Trigger | Generate Weekly Analytics Report | ## 📊 Weekly Analytics Report … |
| Generate Weekly Analytics Report | httpRequest | OpenAI report generation (Markdown) | Get Past Week Posts | Send Weekly Report Email | ## 📊 Weekly Analytics Report … |
| Send Weekly Report Email | emailSend | Email the weekly report | Generate Weekly Analytics Report | — | ## 📊 Weekly Analytics Report … |
| Sticky Note - Main Overview | stickyNote | Global overview and requirements | — | — | ## AI-Powered Japanese Social Media Content Generator with Quality Control … |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. OpenAI (for HTTP Request nodes):
      - Create an HTTP Header Auth credential that adds: `Authorization: Bearer <OPENAI_API_KEY>`
      - Nodes themselves set `Content-Type: application/json`
   2. Twitter/X OAuth2:
      - Create OAuth2 credentials with scopes allowing tweet creation (e.g., `tweet.write`)
   3. Notion:
      - Create Notion credentials and share the target database with the integration
   4. Email:
      - Configure SMTP (or your email provider) in n8n Email node credentials

2) **Prepare Notion database (recommended schema)**
   - Create a database with properties such as:
     - **Content** (text), **Hashtags** (multi-select or text), **Quality Score** (number),
     - **Risk Level** (select), **Status** (select: pending/published/rejected),
     - **Platform** (select), **Engagement** (number), **Generated At** (date),
     - **Improvement Attempts** (number), **Feedback** (text)
   - You will map these in the Notion nodes later (the provided JSON leaves this incomplete).

3) **Daily flow nodes**
   1. Add **Schedule Trigger** named “Schedule Daily Content Generation”
      - Cron: `0 9 * * 1-5`
      - Ensure instance timezone matches JST if required
   2. Add **HTTP Request** named “Generate Content with GPT-4”
      - POST `https://api.openai.com/v1/chat/completions`
      - Auth: your OpenAI header auth credential
      - Headers: `Content-Type: application/json`
      - Body: model `gpt-4`, temperature `0.7`, max_tokens `1000`
      - Messages: system + user prompt requesting 3 posts and strict JSON array output
   3. Add **Code** named “Parse Generated Content”
      - Parse `choices[0].message.content` as JSON array
      - Output one item per post (content, hashtags, version, timestamps, etc.)
   4. Connect: Schedule Trigger → Generate Content with GPT-4 → Parse Generated Content

4) **Quality scoring + improvement**
   1. Add **HTTP Request** “AI Quality Scoring”
      - POST OpenAI chat completions
      - temperature `0.3`
      - Prompt asks for JSON scores including `total_score`, sub-scores, `feedback`, and `risk_level`
   2. Add **Code** “Parse Quality Scores”
      - JSON.parse scoring output
      - Merge into post item; set `needsImprovement = total_score < 70`, `improvementAttempts = 0`
      - (Recommended fix while rebuilding: merge with the *current* input item instead of referencing “Parse Generated Content”.)
   3. Add **IF** “Check if Improvement Needed”
      - Condition 1: needsImprovement equals true
      - Condition 2: improvementAttempts < 3
   4. Add **HTTP Request** “Auto-Improve Content”
      - POST OpenAI
      - Prompt includes original content + feedback; request strict JSON `{content, hashtags}`
      - temperature `0.7`
   5. Add **Code** “Parse Improved Content”
      - Parse improved JSON
      - Increment improvementAttempts
      - Send back into “AI Quality Scoring”
   6. Connect:
      - Parse Generated Content → AI Quality Scoring → Parse Quality Scores → Check if Improvement Needed
      - IF true → Auto-Improve Content → Parse Improved Content → AI Quality Scoring (loop)

5) **Risk analysis + routing**
   1. Add **HTTP Request** “Sentiment & Risk Analysis”
      - POST OpenAI
      - temperature `0.2`
      - Request JSON with sentiment, risk_level, risk_factors, recommendations, cultural_appropriateness
   2. Add **Code** “Merge Risk Analysis”
      - Parse analysis JSON
      - Merge with scored content
      - Compute `finalRiskAssessment` (auto_approve / requires_approval / auto_reject)
   3. Add **IF** “Decision Routing”
      - Condition: finalRiskAssessment == auto_approve
      - True: auto-approved path
      - False: pending approval path
      - (Recommended when rebuilding: add a dedicated branch for `auto_reject`.)
   4. Connect:
      - IF false branch (from “Check if Improvement Needed”) → Sentiment & Risk Analysis → Merge Risk Analysis → Decision Routing

6) **Notion persistence + Twitter posting**
   1. Add **Notion** “Save to Notion (Auto-Approved)”
      - Operation: Create page in your database (recommended)
      - Map fields: Content, Hashtags, Quality Score, Risk Level, Status=published, etc.
   2. Add **HTTP Request** “Post to Twitter (Auto-Approved)”
      - POST `https://api.twitter.com/2/tweets`
      - OAuth2 auth
      - Body: `text` expression combining content + hashtags
   3. Connect: Decision Routing (true) → Save to Notion (Auto-Approved) → Post to Twitter (Auto-Approved)

7) **Pending approval + email**
   1. Add **Notion** “Save to Notion (Pending Approval)”
      - Operation: Create page in database
      - Status=pending, store risk/quality and full content
   2. Add **Email Send** “Send Approval Email”
      - To/From per your team
      - Subject includes qualityScore and riskLevel
      - (Recommended: include approval links pointing to your webhook endpoint with query params.)
   3. Connect: Decision Routing (false) → Save to Notion (Pending Approval) → Send Approval Email

8) **Approval webhook sub-flow**
   1. Add **Webhook** “Webhook - Approval Dashboard”
      - Path: `approval-webhook`
   2. Add **Code** “Process Approval Action”
      - Read query: `action`, `id`; body: `editedContent`, `approvedBy`
   3. Add **IF** “Check Approval Action”
      - action == approve
   4. Add **Notion** “Update Notion - Approved” (operation: update page by ID)
   5. Add **Notion** “Update Notion - Rejected” (operation: update page by ID)
   6. Connect: Webhook → Process Approval Action → IF → Update nodes
   7. (Recommended: add “Post to Twitter (Approved via Webhook)” after approval, using the stored or edited content.)

9) **Weekly analytics**
   1. Add **Schedule Trigger** “Weekly Performance Report Trigger”
      - Cron: `0 10 * * 1`
   2. Add **Notion** “Get Past Week Posts”
      - Operation: getAll
      - (Recommended: filter on date property “Generated At” or “Published At” for last 7 days.)
   3. Add **HTTP Request** “Generate Weekly Analytics Report”
      - POST OpenAI; prompt asks for Markdown report
   4. Add **Email Send** “Send Weekly Report Email”
      - Subject uses `$now.format('yyyy年MM月dd日')`
      - (Recommended: set email body to the GPT Markdown output.)
   5. Connect: Weekly Trigger → Get Past Week Posts → Generate Weekly Analytics Report → Send Weekly Report Email

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The sticky notes specify: auto-reject on high risk; however, the current routing IF only separates auto-approve vs non-auto-approve. High-risk content is not explicitly rejected in execution logic. | “Sticky Note - Routing” vs actual “Decision Routing” implementation gap |
| Brand voice and cultural context nodes run but are not used to condition the content generation prompt in the current wiring. | “Get Japanese Cultural Context” and “Create Brand Voice Profile” are disconnected outputs |
| Approval webhook supports approve/reject/edit in notes, but logic only checks `approve`; everything else routes to “rejected”. | “Sticky Note - Webhook” vs “Check Approval Action” |
| Webhook approval updates Notion but does not publish to Twitter after approval. | Missing post-to-Twitter step in approval sub-flow |
| Notion “Save to …” nodes show empty `pageId` and no visible field mapping; to function, configure database/page targets and map properties. | Notion node configuration is incomplete in provided workflow JSON |

Disclaimer (as provided): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.