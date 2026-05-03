Repurpose YouTube videos into blogs and social posts with GPT-4o, WordPress, LinkedIn, X and Instagram

https://n8nworkflows.xyz/workflows/repurpose-youtube-videos-into-blogs-and-social-posts-with-gpt-4o--wordpress--linkedin--x-and-instagram-12264


# Repurpose YouTube videos into blogs and social posts with GPT-4o, WordPress, LinkedIn, X and Instagram

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow repurposes a single YouTube video into omnichannel content (Turkish long-form blog + LinkedIn post + X/Twitter thread + Instagram Reels concept/caption). It supports a human-in-the-loop review cycle via Slack webhooks, then publishes only the approved platforms and logs the result to Notion.

**Primary use cases:**
- Creators/marketing teams converting video content into multi-platform written/social assets
- Workflows requiring **manual approval**, **revision loops**, and selective publishing per platform
- Basic publishing audit trail in Notion + Slack success notification

### 1.1 Input Reception & Transcript Retrieval
- Accepts a `youtube_url` (manual trigger input)
- Calls Supadata transcript API
- Stops if transcript is missing

### 1.2 Data Normalization & AI Content Generation
- Normalizes transcript + initializes approval flags and revision counters
- Uses OpenAI (GPT-4o) to generate a **strict JSON object** containing all content variants
- Parses AI output into a structured object (`content_data`)

### 1.3 Pre-Publish Creative Steps & Review Dispatch
- (Placeholder) calls Gemini nodes for blog image / reels generation steps
- Sends a Slack “review” message summarizing content and showing approval state

### 1.4 Approval Handling (Webhook-driven)
- Receives Slack interactive action payload via webhook
- Interprets action: publish / revise / cancel / approve_* per platform
- On revise: asks for feedback and receives it via another webhook; increments revision count and regenerates unapproved content

### 1.5 Conditional Publishing to Platforms
- If action is `publish`, checks per-platform approved flags and publishes:
  - WordPress draft post
  - LinkedIn post
  - X/Twitter 5-tweet thread
  - Instagram (via community node; configuration incomplete in provided workflow)

### 1.6 Logging & Completion
- Writes a Notion database page marking which platforms were approved/published
- Sends Slack success notification

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Transcript Retrieval
**Overview:** Starts with a YouTube URL, fetches transcript text from an external API, and halts execution if transcript is missing.

**Nodes involved:**
- ▶️ Start
- 📝 Fetch Transcript
- ✔️ Check Transcript
- ❌ Error Stop

#### ▶️ Start
- **Type / role:** Manual Trigger; workflow entry point for testing/manual runs.
- **Configuration:** No parameters; expects the operator to provide `youtube_url` in the input JSON.
- **Key variables:** `$json.youtube_url`
- **Outputs:** To 📝 Fetch Transcript
- **Failure modes:** Missing `youtube_url` causes downstream API request to fail or return empty transcript.

#### 📝 Fetch Transcript
- **Type / role:** HTTP Request; calls Supadata YouTube transcript endpoint.
- **Configuration choices:**
  - `GET https://api.supadata.ai/v1/youtube/transcript`
  - Query params:
    - `url = {{$json.youtube_url}}`
    - `text = true` (requests raw text output)
  - Header:
    - `x-api-key = sd_cf253...` (hardcoded in workflow)
- **Inputs:** From ▶️ Start
- **Outputs:** To ✔️ Check Transcript
- **Edge cases / failures:**
  - 401/403 if API key invalid/expired
  - 429 rate limiting
  - Transcript unavailable (no captions, region restrictions, etc.)
  - Network timeouts
  - **Security note:** API key is embedded in the node; best practice is to move it to credentials/environment variables.

#### ✔️ Check Transcript
- **Type / role:** IF; validates transcript presence.
- **Condition:** String `{{$json.content}}` **is not empty**
- **Outputs:**
  - **True** → 🔧 Prepare Data
  - **False** → ❌ Error Stop
- **Failure modes:** If Supadata response schema changes (e.g., transcript not in `content`), this check may incorrectly fail.

#### ❌ Error Stop
- **Type / role:** Stop and Error; terminates execution with message.
- **Error message:** `❌ No transcript found`
- **Inputs:** From ✔️ Check Transcript (false branch)

---

### Block 2 — Data Normalization & AI Content Generation
**Overview:** Prepares a consistent working object (transcript, url, flags), generates a structured JSON content package with GPT-4o, then parses it into `content_data`.

**Nodes involved:**
- 🔧 Prepare Data
- 🧙 AI Content
- 🔍 Parse JSON

#### 🔧 Prepare Data
- **Type / role:** Set; normalizes fields and initializes approval workflow state.
- **Key assignments:**
  - `transcript_text = {{$json.content}}`
  - `video_url = {{$('▶️ Start').item.json.youtube_url}}`
  - `revision_count = 0`
  - `blog_approved/linkedin_approved/twitter_approved/instagram_approved = false`
- **Input:** From ✔️ Check Transcript (true)
- **Output:** To 🧙 AI Content
- **Edge cases:**
  - If multiple items are ever introduced, using `$('▶️ Start').item...` may bind to an unexpected item index.

#### 🧙 AI Content
- **Type / role:** OpenAI (LangChain) node; generates multi-platform content in one response.
- **Model:** `gpt-4o`
- **Important options:**
  - `maxTokens: 4000`
  - `temperature: 0.7`
  - `textFormat: json_object` (attempts to enforce JSON-only output)
- **Prompt logic highlights:**
  - If `revision_count > 0`, instructs to “Regenerate only unapproved platforms.”
  - Outputs required JSON keys:
    - Blog: `blog_title`, `blog_meta_description`, `blog_content` (1000+ words, Turkish), `blog_keywords`
    - `linkedin_post`
    - `twitter_thread` (array of 5 tweets labeled “1/5”…)
    - `instagram_reels` (nested script structure + caption + duration)
    - `image_prompts` (blog image, reels thumbnail)
- **Inputs:** From 🔧 Prepare Data (or 🔧 Prep Revision in revision loop)
- **Outputs:** To 🔍 Parse JSON
- **Failure modes / edge cases:**
  - Model may still return invalid JSON (despite json_object mode), causing parse failure downstream
  - Token limit may truncate 1000+ word blog; consider raising max tokens or using chunking
  - If transcript is very long, prompt may exceed context window (depends on n8n node + model limits)

#### 🔍 Parse JSON
- **Type / role:** Set; parses the model text into an object field.
- **Key expression:**
  - `content_data = {{ JSON.parse($json.output[0].content[0].text) }}`
- **Input:** From 🧙 AI Content
- **Output:** To 🎨 Blog Image
- **Failure modes:**
  - Any slight JSON formatting issue throws an exception and stops workflow
  - Output path assumptions (`output[0].content[0].text`) are node-version dependent; changes in OpenAI node structure can break this.

---

### Block 3 — Pre-Publish Creative Steps & Review Dispatch
**Overview:** Runs placeholder “creative” generation steps (blog image / reels) and posts a Slack review message summarizing outputs and approval states.

**Nodes involved:**
- 🎨 Blog Image
- 💬 Slack Approval

#### 🎨 Blog Image
- **Type / role:** Google Gemini (LangChain) node; intended to generate a blog image or prompt refinement.
- **Configuration status:** Incomplete in provided JSON:
  - `modelId` is empty
  - `messages.values` contains an empty object (no prompt)
- **Input:** From 🔍 Parse JSON
- **Output:** To 💬 Slack Approval
- **Failure modes:**
  - Will fail at runtime without a selected model and a valid prompt/messages structure
  - Credential requirements not shown; likely needs Gemini/Google credentials configured in n8n

#### 💬 Slack Approval
- **Type / role:** Slack node; sends a formatted review message.
- **Text content (key dynamics):**
  - Shows attempt number: `revision_count + 1`
  - Displays video URL
  - Displays blog title, LinkedIn excerpt, first tweet, Instagram hook
  - Displays ✅ markers if flags are already approved
- **Input:** From 🎨 Blog Image
- **Output:** **No connection** in the provided workflow to a “wait for approval” step.
- **Critical integration gap:**
  - There is a separate webhook node **🔔 Wait Approval** intended to receive Slack interactive actions, but **this Slack message does not define interactive buttons** in node configuration.
  - As-is, sending text alone won’t generate Slack `actions[0].action_id` payloads.
- **Failure modes:**
  - Missing Slack credentials / channel configuration (not visible here)
  - Slack formatting errors are unlikely, but payload size limits apply if you expand content

---

### Block 4 — Approval Handling (Webhook-driven + Revision Loop)
**Overview:** Waits for Slack action callbacks, interprets user decision (approve platform / publish / revise / cancel), optionally gathers feedback and regenerates content.

**Nodes involved:**
- 🔔 Wait Approval
- 🔄 Process Action
- 📤 Publish?
- 🔄 Revise?
- ❌ Cancel?
- 💬 Ask Feedback
- 📥 Get Feedback
- 🔧 Prep Revision
- 🚫 Cancel Notify
- 🛑 Stop

#### 🔔 Wait Approval
- **Type / role:** Webhook; receives approval action payload (intended from Slack interactive components).
- **Path:** `approval-webhook`
- **Response mode:** `lastNode`
- **Input:** External HTTP request from Slack
- **Output:** To 🔄 Process Action
- **Failure modes / edge cases:**
  - Slack signature verification is not configured here (recommended)
  - Payload schema assumptions: expects `body.actions[0].action_id`

#### 🔄 Process Action
- **Type / role:** Code; converts Slack action into workflow state updates.
- **Key logic:**
  - `action = $input.item.json.body.actions[0].action_id`
  - Pulls previous state from `$('🔧 Prepare Data').item.json`
  - Creates `result` object with:
    - approval flags (carried over)
    - `action`: normalized (`approve_*` → `approve_platform`, otherwise keeps original)
    - `content_data`: from 🔍 Parse JSON
    - transcript/video_url/revision_count
  - Sets flags true for `approve_blog`, `approve_linkedin`, `approve_twitter`, `approve_instagram`
- **Outputs:** To three IF nodes:
  - 📤 Publish?
  - 🔄 Revise?
  - ❌ Cancel?
- **Important edge case:** If Slack sends a different payload shape (no `actions`), node will throw (cannot read property).
- **Logic gap:** Approve actions set flags but map action to `approve_platform`, yet there is **no branch** handling `approve_platform` to loop back and wait for additional approvals or to update the Slack message.

#### ❌ Cancel?
- **Type / role:** IF; checks if action equals `cancel`.
- **Condition:** `{{$json.action}} == "cancel"`
- **True output:** 🚫 Cancel Notify
- **Edge cases:** If action names differ (e.g., “cancel_workflow”), it won’t match.

#### 🚫 Cancel Notify
- **Type / role:** Slack node; sends “Cancelled”.
- **Output:** 🛑 Stop
- **Failure modes:** Slack auth/channel issues.

#### 🛑 Stop
- **Type / role:** Stop and Error
- **Message:** `Cancelled by user`

#### 🔄 Revise?
- **Type / role:** IF; checks if action equals `revise`.
- **True output:** 💬 Ask Feedback

#### 💬 Ask Feedback
- **Type / role:** Slack node; asks reviewer to provide feedback text.
- **Output:** Not connected to 📥 Get Feedback directly (feedback is collected via separate webhook).

#### 📥 Get Feedback
- **Type / role:** Webhook; receives feedback message payload.
- **Path:** `feedback-webhook`
- **Response mode:** `lastNode`
- **Output:** 🔧 Prep Revision
- **Payload assumption:** Uses `$json.body.event.text` later, implying Slack Events API-like payload.

#### 🔧 Prep Revision
- **Type / role:** Set; prepares revision inputs for AI regeneration.
- **Key assignments:**
  - `revision_count = {{ $('🔄 Process Action').item.json.revision_count + 1 }}`
  - `revision_feedback = {{ $json.body.event.text }}`
  - Carry forward transcript, video_url, and approval flags from 🔄 Process Action
- **Output:** 🧙 AI Content (re-generation)
- **Edge cases:**
  - Feedback payload shape mismatch will break `revision_feedback`
  - The AI prompt references only `revision_count`, not `revision_feedback`; so feedback is currently **not actually used** in AI generation (unless transcript includes it elsewhere). This is a functional gap.

#### 📤 Publish?
- **Type / role:** IF; checks if action equals `publish`.
- **True output:** Parallel fan-out to platform checks:
  - ✔️ Blog?
  - ✔️ LinkedIn?
  - ✔️ Twitter?
  - ✔️ Instagram?
- **Edge cases:** If publish is triggered before flags are set, nothing may publish.

---

### Block 5 — Platform Publishing (Conditional)
**Overview:** Publishes only platforms that are approved (boolean flags), then converges to Notion logging.

**Nodes involved:**
- ✔️ Blog? → 📝 WordPress
- ✔️ LinkedIn? → 💼 LinkedIn
- ✔️ Twitter? → 🐦 Tweet 1/5 → 🐦 Tweet 2/5 → 🐦 Tweet 3/5 → 🐦 Tweet 4/5 → 🐦 Tweet 5/5
- ✔️ Instagram? → 🎥 Gen Reels → ⏱️ Wait → 📸 Instagram1

#### ✔️ Blog?
- **Type / role:** IF; gate WordPress publishing.
- **Condition:** `{{$json.blog_approved}} == true`
- **True output:** 📝 WordPress

#### 📝 WordPress
- **Type / role:** WordPress node; creates a post (configured as draft).
- **Key configuration:**
  - `title = {{content_data.blog_title}}`
  - `tags = {{content_data.blog_keywords.join(',')}}`
  - `status = draft`
- **Notably missing:** The actual blog body/content field is not mapped in the shown parameters (likely omitted by node UI export or not configured). As written, it may create an empty draft with title + tags only.
- **Output:** 📊 Notion
- **Failure modes:** WordPress auth errors, insufficient permissions, tag formatting issues.

#### ✔️ LinkedIn?
- **Type / role:** IF gate for LinkedIn
- **Condition:** `{{$json.linkedin_approved}} == true`
- **True output:** 💼 LinkedIn

#### 💼 LinkedIn
- **Type / role:** LinkedIn node; publishes a post.
- **Key configuration:**
  - `text = {{content_data.linkedin_post}}`
  - `visibility = PUBLIC`
- **Output:** 📊 Notion
- **Failure modes:** OAuth expiration, API permission limitations, rate limits.

#### ✔️ Twitter?
- **Type / role:** IF gate for X/Twitter
- **Condition:** `{{$json.twitter_approved}} == true`
- **True output:** 🐦 Tweet 1/5

#### 🐦 Tweet 1/5 … 🐦 Tweet 5/5
- **Type / role:** Twitter nodes; publish a reply-chain thread.
- **Key configuration:**
  - Tweet 1 text: `twitter_thread[0]`
  - Tweet 2 replies to Tweet 1 via `inReplyToStatusId = {{$('🐦 Tweet 1/5').item.json.id_str}}`
  - Tweet 3 replies to Tweet 2, etc.
- **Final output:** 🐦 Tweet 5/5 → 📊 Notion
- **Failure modes / edge cases:**
  - X API access level may not allow posting
  - Rate limits
  - `id_str` path differences depending on node version/API response
  - If a tweet exceeds character limits, posting fails mid-thread (leaving partial thread)

#### ✔️ Instagram?
- **Type / role:** IF gate for Instagram
- **Condition:** `{{$json.instagram_approved}} == true`
- **True output:** 🎥 Gen Reels

#### 🎥 Gen Reels
- **Type / role:** Google Gemini (LangChain); intended to generate Reels assets/plan.
- **Configuration status:** Incomplete (empty modelId, empty messages)
- **Output:** ⏱️ Wait
- **Failure modes:** Same as 🎨 Blog Image—will not run without model and prompt.

#### ⏱️ Wait
- **Type / role:** Wait node; pauses execution (often used for asynchronous external processing).
- **Output:** 📸 Instagram1
- **Edge cases:** If used to wait for an external callback, it needs configuration (not shown). As-is, it may behave as a manual resume wait depending on n8n settings/version.

#### 📸 Instagram1
- **Type / role:** Community Instagram node (`@mookielianhd/n8n-nodes-instagram.instagram`); configured with resource `image`.
- **Configuration:** Minimal; no caption/media URL mapping shown.
- **Failure modes / requirements:**
  - Requires installing the community node package on the n8n instance
  - Requires Instagram credentials and likely a business/creator account depending on API method
  - Missing required fields will cause runtime errors

---

### Block 6 — Logging & Completion
**Overview:** Writes a Notion record with publishing outcomes and sends a Slack success message.

**Nodes involved:**
- 📊 Notion
- 🎉 Success

#### 📊 Notion
- **Type / role:** Notion node; creates a database page.
- **Resource:** `databasePage`
- **Database:** `2ce9e1e0-0586-80ad-bb90-fbaa84df66ef`
- **Properties mapped:**
  - Title: `content_data.blog_title`
  - URL: `video_url`
  - Status select: `"Published"`
  - Checkboxes for each platform: `blog_approved`, `linkedin_approved`, `twitter_approved`, `instagram_approved`
- **Input:** From each publishing branch (WordPress/LinkedIn/Tweet5)
- **Output:** 🎉 Success
- **Edge cases:** Notion property names must match exactly (`Video Title|title`, etc.). Any schema change breaks the node.

#### 🎉 Success
- **Type / role:** Slack node; posts a summary message listing approved platforms.
- **Text logic:** Conditional lines based on flags.
- **Input:** From 📊 Notion
- **Failure modes:** Slack auth/channel issues.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ▶️ Start | manualTrigger | Entry point; provides `youtube_url` | — | 📝 Fetch Transcript | ## 1️⃣ Input & Transcript<br><br>This section starts the workflow.<br><br>- Paste a YouTube video URL<br>- Fetch the video transcript using an external API<br>- Stop the workflow if no transcript is found<br><br>The transcript is normalized into a single text field for downstream AI processing. |
| 📝 Fetch Transcript | httpRequest | Fetch YouTube transcript via Supadata API | ▶️ Start | ✔️ Check Transcript | ## 1️⃣ Input & Transcript<br><br>This section starts the workflow.<br><br>- Paste a YouTube video URL<br>- Fetch the video transcript using an external API<br>- Stop the workflow if no transcript is found<br><br>The transcript is normalized into a single text field for downstream AI processing. |
| ✔️ Check Transcript | if | Validate transcript exists | 📝 Fetch Transcript | 🔧 Prepare Data / ❌ Error Stop | ## 1️⃣ Input & Transcript<br><br>This section starts the workflow.<br><br>- Paste a YouTube video URL<br>- Fetch the video transcript using an external API<br>- Stop the workflow if no transcript is found<br><br>The transcript is normalized into a single text field for downstream AI processing. |
| ❌ Error Stop | stopAndError | Halt if no transcript | ✔️ Check Transcript (false) | — | ## 1️⃣ Input & Transcript<br><br>This section starts the workflow.<br><br>- Paste a YouTube video URL<br>- Fetch the video transcript using an external API<br>- Stop the workflow if no transcript is found<br><br>The transcript is normalized into a single text field for downstream AI processing. |
| 🔧 Prepare Data | set | Normalize transcript + init flags/counters | ✔️ Check Transcript (true) | 🧙 AI Content | ## 1️⃣ Input & Transcript<br><br>This section starts the workflow.<br><br>- Paste a YouTube video URL<br>- Fetch the video transcript using an external API<br>- Stop the workflow if no transcript is found<br><br>The transcript is normalized into a single text field for downstream AI processing. |
| 🧙 AI Content | openAi (langchain) | Generate omnichannel content JSON | 🔧 Prepare Data / 🔧 Prep Revision | 🔍 Parse JSON |  |
| 🔍 Parse JSON | set | Parse AI JSON into `content_data` | 🧙 AI Content | 🎨 Blog Image |  |
| 🎨 Blog Image | googleGemini (langchain) | Intended image/prompt generation (incomplete) | 🔍 Parse JSON | 💬 Slack Approval |  |
| 💬 Slack Approval | slack | Send review summary to Slack | 🎨 Blog Image | — |  |
| 🔔 Wait Approval | webhook | Receive Slack interactive action payload | — (external) | 🔄 Process Action | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 🔄 Process Action | code | Interpret action + update approval flags | 🔔 Wait Approval | 📤 Publish? / 🔄 Revise? / ❌ Cancel? | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| ❌ Cancel? | if | Branch if user cancels | 🔄 Process Action | 🚫 Cancel Notify | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 🚫 Cancel Notify | slack | Notify cancellation | ❌ Cancel? | 🛑 Stop | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 🛑 Stop | stopAndError | Stop execution after cancel | 🚫 Cancel Notify | — | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 🔄 Revise? | if | Branch if revision requested | 🔄 Process Action | 💬 Ask Feedback | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 💬 Ask Feedback | slack | Ask reviewer for feedback | 🔄 Revise? | — | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 📥 Get Feedback | webhook | Receive feedback text payload | — (external) | 🔧 Prep Revision | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 🔧 Prep Revision | set | Increment revision + carry flags and inputs | 📥 Get Feedback | 🧙 AI Content | ## 2️⃣ Approval Handling<br><br>This section handles the approval logic.<br><br>Depending on the configuration, the workflow can:<br>- Automatically continue to publishing<br>- Pause and wait for manual approval<br>- Cancel or request revisions<br><br>This keeps the workflow flexible for both solo creators and teams. |
| 📤 Publish? | if | Branch into publishing fan-out | 🔄 Process Action | ✔️ Blog? / ✔️ LinkedIn? / ✔️ Twitter? / ✔️ Instagram? |  |
| ✔️ Blog? | if | Publish WordPress only if approved | 📤 Publish? | 📝 WordPress | ## 3️⃣ Blog & LinkedIn Publishing<br><br>This section publishes long-form and professional content.<br><br>- Generates a blog post from the transcript<br>- Creates a LinkedIn post based on the same content<br>- Publishes or saves drafts depending on your platform settings<br><br>Useful for SEO and professional audience distribution. |
| 📝 WordPress | wordpress | Create WP draft post | ✔️ Blog? | 📊 Notion | ## 3️⃣ Blog & LinkedIn Publishing<br><br>This section publishes long-form and professional content.<br><br>- Generates a blog post from the transcript<br>- Creates a LinkedIn post based on the same content<br>- Publishes or saves drafts depending on your platform settings<br><br>Useful for SEO and professional audience distribution. |
| ✔️ LinkedIn? | if | Publish LinkedIn only if approved | 📤 Publish? | 💼 LinkedIn | ## 3️⃣ Blog & LinkedIn Publishing<br><br>This section publishes long-form and professional content.<br><br>- Generates a blog post from the transcript<br>- Creates a LinkedIn post based on the same content<br>- Publishes or saves drafts depending on your platform settings<br><br>Useful for SEO and professional audience distribution. |
| 💼 LinkedIn | linkedIn | Publish LinkedIn post | ✔️ LinkedIn? | 📊 Notion | ## 3️⃣ Blog & LinkedIn Publishing<br><br>This section publishes long-form and professional content.<br><br>- Generates a blog post from the transcript<br>- Creates a LinkedIn post based on the same content<br>- Publishes or saves drafts depending on your platform settings<br><br>Useful for SEO and professional audience distribution. |
| ✔️ Twitter? | if | Publish thread only if approved | 📤 Publish? | 🐦 Tweet 1/5 | ## 4️⃣ X (Twitter) Threads<br><br>This section creates short-form content for X (Twitter).<br><br>- Breaks the main idea into multiple tweets<br>- Publishes them as a thread<br>- Each tweet is generated automatically from the original video content<br><br>Ideal for content amplification and reach. |
| 🐦 Tweet 1/5 | twitter | Post first tweet | ✔️ Twitter? | 🐦 Tweet 2/5 | ## 4️⃣ X (Twitter) Threads<br><br>This section creates short-form content for X (Twitter).<br><br>- Breaks the main idea into multiple tweets<br>- Publishes them as a thread<br>- Each tweet is generated automatically from the original video content<br><br>Ideal for content amplification and reach. |
| 🐦 Tweet 2/5 | twitter | Reply to previous tweet | 🐦 Tweet 1/5 | 🐦 Tweet 3/5 | ## 4️⃣ X (Twitter) Threads<br><br>This section creates short-form content for X (Twitter).<br><br>- Breaks the main idea into multiple tweets<br>- Publishes them as a thread<br>- Each tweet is generated automatically from the original video content<br><br>Ideal for content amplification and reach. |
| 🐦 Tweet 3/5 | twitter | Reply to previous tweet | 🐦 Tweet 2/5 | 🐦 Tweet 4/5 | ## 4️⃣ X (Twitter) Threads<br><br>This section creates short-form content for X (Twitter).<br><br>- Breaks the main idea into multiple tweets<br>- Publishes them as a thread<br>- Each tweet is generated automatically from the original video content<br><br>Ideal for content amplification and reach. |
| 🐦 Tweet 4/5 | twitter | Reply to previous tweet | 🐦 Tweet 3/5 | 🐦 Tweet 5/5 | ## 4️⃣ X (Twitter) Threads<br><br>This section creates short-form content for X (Twitter).<br><br>- Breaks the main idea into multiple tweets<br>- Publishes them as a thread<br>- Each tweet is generated automatically from the original video content<br><br>Ideal for content amplification and reach. |
| 🐦 Tweet 5/5 | twitter | Reply to previous tweet; completes thread | 🐦 Tweet 4/5 | 📊 Notion | ## 4️⃣ X (Twitter) Threads<br><br>This section creates short-form content for X (Twitter).<br><br>- Breaks the main idea into multiple tweets<br>- Publishes them as a thread<br>- Each tweet is generated automatically from the original video content<br><br>Ideal for content amplification and reach. |
| ✔️ Instagram? | if | Publish IG flow only if approved | 📤 Publish? | 🎥 Gen Reels | ## 5️⃣ Instagram Content<br><br>This section handles Instagram content.<br><br>- Generates an Instagram caption<br>- Optionally generates visual or reel ideas<br>- Publishes or prepares content for Instagram<br><br>Designed for high-engagement social platforms. |
| 🎥 Gen Reels | googleGemini (langchain) | Intended reels generation (incomplete) | ✔️ Instagram? | ⏱️ Wait | ## 5️⃣ Instagram Content<br><br>This section handles Instagram content.<br><br>- Generates an Instagram caption<br>- Optionally generates visual or reel ideas<br>- Publishes or prepares content for Instagram<br><br>Designed for high-engagement social platforms. |
| ⏱️ Wait | wait | Pause before IG publish step | 🎥 Gen Reels | 📸 Instagram1 | ## 5️⃣ Instagram Content<br><br>This section handles Instagram content.<br><br>- Generates an Instagram caption<br>- Optionally generates visual or reel ideas<br>- Publishes or prepares content for Instagram<br><br>Designed for high-engagement social platforms. |
| 📸 Instagram1 | instagram (community) | Publish IG image (incomplete) | ⏱️ Wait | — | ## 5️⃣ Instagram Content<br><br>This section handles Instagram content.<br><br>- Generates an Instagram caption<br>- Optionally generates visual or reel ideas<br>- Publishes or prepares content for Instagram<br><br>Designed for high-engagement social platforms. |
| 📊 Notion | notion | Log outcome to Notion DB | 📝 WordPress / 💼 LinkedIn / 🐦 Tweet 5/5 | 🎉 Success | ## 6️⃣ Logging & Completion<br><br>This section finalizes the workflow.<br><br>- Logs publishing results (for example in Notion)<br>- Tracks which platforms were used<br>- Marks the workflow execution as successful<br><br>Helps with content tracking and auditing. |
| 🎉 Success | slack | Notify success summary | 📊 Notion | — | ## 6️⃣ Logging & Completion<br><br>This section finalizes the workflow.<br><br>- Logs publishing results (for example in Notion)<br>- Tracks which platforms were used<br>- Marks the workflow execution as successful<br><br>Helps with content tracking and auditing. |
| Sticky Note | stickyNote | Documentation | — | — | ### What this workflow does<br><br>This workflow transforms a YouTube video into:<br>- A blog post<br>- LinkedIn post<br>- X (Twitter) threads<br>- Instagram caption<br><br>It supports both automatic publishing and manual approval before publishing.<br><br>### How it works<br><br>1. Fetches the YouTube transcript<br>2. Generates content using AI<br>3. (Optional) waits for human approval<br>4. Publishes content to selected platforms<br>5. Logs the publishing result<br><br>### Setup steps<br><br>1. Add your credentials (AI, transcript API, social platforms)<br>2. Paste a YouTube video URL<br>3. Choose your publishing mode<br>4. Execute the workflow |
| Sticky Note1 | stickyNote | Documentation | — | — | (see content above; informational note) |
| Sticky Note2 | stickyNote | Documentation | — | — | (see content above; informational note) |
| Sticky Note3 | stickyNote | Documentation | — | — | (see content above; informational note) |
| Sticky Note4 | stickyNote | Documentation | — | — | (see content above; informational note) |
| Sticky Note5 | stickyNote | Documentation | — | — | (see content above; informational note) |
| Sticky Note6 | stickyNote | Documentation | — | — | (see content above; informational note) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create “▶️ Start” (Manual Trigger)**
- Node type: *Manual Trigger*
- Run data must include:  
  - `youtube_url` (string), e.g. `https://www.youtube.com/watch?v=...`

2) **Add “📝 Fetch Transcript” (HTTP Request)**
- Node type: *HTTP Request*
- Method: GET  
- URL: `https://api.supadata.ai/v1/youtube/transcript`
- Send query parameters:
  - `url` → expression `{{$json.youtube_url}}`
  - `text` → `true`
- Send headers:
  - `x-api-key` → your Supadata key (store in credentials/env var if possible)
- Connect: ▶️ Start → 📝 Fetch Transcript

3) **Add “✔️ Check Transcript” (IF)**
- Condition (String):
  - Value1: `{{$json.content}}`
  - Operation: *is not empty*
- Connect: 📝 Fetch Transcript → ✔️ Check Transcript

4) **Add “❌ Error Stop” (Stop and Error)**
- Error message: `❌ No transcript found`
- Connect: ✔️ Check Transcript (false) → ❌ Error Stop

5) **Add “🔧 Prepare Data” (Set)**
- Add fields:
  - `transcript_text` (string) = `{{$json.content}}`
  - `video_url` (string) = `{{$('▶️ Start').item.json.youtube_url}}`
  - `revision_count` (number) = `0`
  - `blog_approved` (boolean) = `false`
  - `linkedin_approved` (boolean) = `false`
  - `twitter_approved` (boolean) = `false`
  - `instagram_approved` (boolean) = `false`
- Connect: ✔️ Check Transcript (true) → 🔧 Prepare Data

6) **Add “🧙 AI Content” (OpenAI / LangChain)**
- Node type: *@n8n/n8n-nodes-langchain.openAi*
- Credentials: configure **OpenAI API** credential in n8n and select it
- Model: `gpt-4o`
- Options:
  - Max tokens: `4000` (increase if needed)
  - Temperature: `0.7`
  - Response format: **JSON object**
- Prompt: replicate the workflow prompt (include transcript, revision logic)
- Connect: 🔧 Prepare Data → 🧙 AI Content

7) **Add “🔍 Parse JSON” (Set)**
- Create field:
  - `content_data` (object) = `{{ JSON.parse($json.output[0].content[0].text) }}`
- Connect: 🧙 AI Content → 🔍 Parse JSON

8) **(Fix/Complete) “🎨 Blog Image” (Gemini)**
- Node type: *@n8n/n8n-nodes-langchain.googleGemini*
- Select a Gemini model (e.g., `gemini-1.5-pro` depending on your setup)
- Provide a message/prompt, e.g. “Using `content_data.image_prompts.blog_image`, generate an image prompt or brief.”
- Connect: 🔍 Parse JSON → 🎨 Blog Image  
  (Or remove this node if not needed.)

9) **Add “💬 Slack Approval” (Slack message)**
- Node type: *Slack*
- Credentials: Slack OAuth/token configured
- Configure channel/target as needed
- Message text: build a summary using expressions from `content_data` and flags
- **Important:** If you want interactive approvals, configure Slack interactivity (buttons) via Slack Block Kit and point actions to your webhook endpoints.
- Connect: 🎨 Blog Image → 💬 Slack Approval

10) **Add “🔔 Wait Approval” (Webhook)**
- Node type: *Webhook*
- Path: `approval-webhook`
- Response mode: `Last Node`
- In Slack app settings, configure **Interactivity Request URL** to:  
  `https://<your-n8n-host>/webhook/approval-webhook`

11) **Add “🔄 Process Action” (Code)**
- Node type: *Code* (Run once per item)
- Paste the JS from the workflow (adapt payload paths if your Slack payload differs)
- Connect: 🔔 Wait Approval → 🔄 Process Action

12) **Add branching IF nodes**
- “📤 Publish?”: IF where `{{$json.action}} == "publish"`
- “🔄 Revise?”: IF where `{{$json.action}} == "revise"`
- “❌ Cancel?”: IF where `{{$json.action}} == "cancel"`
- Connect: 🔄 Process Action → all three IF nodes

13) **Revision feedback capture**
- Add “💬 Ask Feedback” (Slack) posting “REVISION REQUESTED”
- Add “📥 Get Feedback” (Webhook) with path `feedback-webhook`
- Configure Slack Events or a slash-command style submission to call this webhook
- Add “🔧 Prep Revision” (Set) to:
  - increment `revision_count`
  - store `revision_feedback`
  - carry over transcript/video_url/flags
- Connect: 🔄 Revise? (true) → 💬 Ask Feedback
- Connect: 📥 Get Feedback → 🔧 Prep Revision → 🧙 AI Content

14) **Cancel path**
- Add “🚫 Cancel Notify” (Slack) and “🛑 Stop” (Stop and Error)
- Connect: ❌ Cancel? (true) → 🚫 Cancel Notify → 🛑 Stop

15) **Publish fan-out**
- From 📤 Publish? (true), connect to:
  - ✔️ Blog? (IF `blog_approved == true`) → 📝 WordPress
  - ✔️ LinkedIn? → 💼 LinkedIn
  - ✔️ Twitter? → thread nodes
  - ✔️ Instagram? → IG flow nodes

16) **WordPress**
- Node type: *WordPress*
- Credentials: WordPress API credentials
- Operation: create post
- Set at least:
  - Title = `{{content_data.blog_title}}`
  - Content/body = `{{content_data.blog_content}}` (**add this; missing in provided workflow**)
  - Status = draft/publish as desired
  - Tags = join keywords

17) **LinkedIn**
- Node type: *LinkedIn*
- Credentials: LinkedIn OAuth
- Text = `{{content_data.linkedin_post}}`
- Visibility = PUBLIC

18) **X/Twitter thread**
- Node type: *Twitter/X*
- Credentials: X API
- Create 5 tweet nodes; set reply chaining using previous tweet ID
- Texts from `content_data.twitter_thread[i]`

19) **Instagram**
- Install the community node package `@mookielianhd/n8n-nodes-instagram`
- Configure Instagram credentials required by that node
- Decide whether you post **image** or **reels**; map caption/media URL accordingly from your generated assets pipeline
- Note: the provided workflow does not supply media URLs/files; you must add storage/upload steps.

20) **Logging**
- Add “📊 Notion” (Notion → Create Database Page)
- Create/select the database and ensure properties match:
  - `Video Title` (title)
  - `Video URL` (url)
  - `Status` (select)
  - platform checkboxes
- Connect the end of each publishing branch to 📊 Notion

21) **Completion notification**
- Add “🎉 Success” (Slack)
- Connect: 📊 Notion → 🎉 Success

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow transforms a YouTube video into a blog post, LinkedIn post, X (Twitter) threads, and Instagram caption; supports automatic publishing and manual approval before publishing. | Sticky note (“What this workflow does”) |
| Setup steps: add credentials (AI, transcript API, social platforms), paste YouTube URL, choose publishing mode, execute. | Sticky note (“Setup steps”) |
| Several nodes are **incomplete** as provided: both Gemini nodes have no model/prompt; Instagram node has no media/caption mapping; Slack approval message has no interactive buttons despite webhook approval logic. | Implementation caveat based on node configurations |
| Security: Supadata API key is hardcoded in HTTP headers; move to credentials or environment variables for safer operations. | Best practice for deployment |

