Text-to-video generator & social media publisher with Fal.ai and Telegram approval

https://n8nworkflows.xyz/workflows/text-to-video-generator---social-media-publisher-with-fal-ai-and-telegram-approval-12128


# Text-to-video generator & social media publisher with Fal.ai and Telegram approval

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Generate short vertical (9:16) cinematic videos from a Telegram text prompt using Fal.ai, send a preview back to Telegram for human approval, and—once approved—auto-generate captions/titles, log the run to Google Sheets, upload the media to Blotato, then publish across multiple social platforms (Instagram, Facebook, LinkedIn, TikTok, YouTube).

**Primary use cases**
- Fast content production pipeline for a brand with a consistent aesthetic (“Hidden Gateway”).
- Human-in-the-loop approval before publishing.
- Multi-channel distribution via a single “media upload once, reuse everywhere” approach (Blotato).

### 1.1 Input Reception (Telegram)
Receives a Telegram message to start the workflow and provides the raw user intent (prompt seed).

### 1.2 AI Prompt Drafting
Uses an LLM agent to convert the user’s Telegram message into a strict-format cinematic 8-second video prompt.

### 1.3 Video Generation (Fal.ai) + Retrieval
Submits the prompt to Fal.ai’s queue endpoint, waits, then fetches the resulting video URL.

### 1.4 Preview + Approval Gate (Telegram)
Sends the generated video to Telegram and waits for a text reply. Approves posting if the reply matches certain keywords.

### 1.5 Refinement Loop (Rejected → Refine → Retry)
If not approved, uses the user’s feedback to revise the original prompt, regenerates, fetches, and sends a new preview (note: the JSON shows only preview-after-retry, not a second approval gate).

### 1.6 Publishing (Captions/Titles → Log → Upload → Post)
On approval, generates caption/hashtags and a short YouTube title via LLM, logs to Google Sheets, uploads the video to Blotato, then posts to the configured platforms.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Telegram)
**Overview:** Starts the workflow when a Telegram bot receives a message; the message text becomes the seed for prompt generation.  
**Nodes involved:** `Telegram Message`

#### Node: Telegram Message
- **Type / role:** `telegramTrigger` — entry point, listens to Telegram updates.
- **Config choices:** Listens to `message` updates only.
- **Key data used later:** `{{$json.message.text}}` (user text), `{{$json.message.chat.id}}` (reply destination).
- **Outputs:** Connected to **Draft Video Prompt**.
- **Potential failures / edge cases:**
  - Telegram credentials misconfigured / webhook not set.
  - Non-text messages (stickers/photos) won’t have `message.text`; downstream agent may generate empty/odd prompt.

---

### Block 2 — AI Prompt Drafting
**Overview:** Converts the user’s message into a brand-aligned cinematic prompt with strict constraints (actor, camera move, palette, no text/fire).  
**Nodes involved:** `Draft Video Prompt`, `OpenRouter Chat Model1`, `Think`

#### Node: Draft Video Prompt
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — LLM agent generating the first Fal.ai-ready prompt.
- **Config choices (interpreted):**
  - Uses a “define” prompt with a structured role/goal/rules block.
  - Hard rules enforce: natural actor calmness, subtle camera move, psychedelic purple/blue neon glow, no on-screen text, no fire imagery.
  - Input: `User Message: {{ $json.message.text }}`
  - Output: **prompt text only** (no JSON, no headings).
- **Model dependency:** Receives a language model via `ai_languageModel` connection from **OpenRouter Chat Model1**.
- **Connections:** Input from **Telegram Message**; output to **Generate Video (Fal.ai)**.
- **Edge cases / failures:**
  - If the LLM includes extra formatting, Fal.ai may still accept it but results may degrade.
  - If `message.text` is missing, the prompt becomes under-specified.

#### Node: OpenRouter Chat Model1
- **Type / role:** `lmChatOpenRouter` — provides the chat model to LangChain agent nodes.
- **Config choices:** Model set to `openai/gpt-5.2` (via OpenRouter).
- **Connections:** Supplies model to **Draft Video Prompt** and **Refine Prompt**.
- **Failure modes:** OpenRouter auth errors, model availability/rate limits, latency/timeouts.

#### Node: Think
- **Type / role:** `toolThink` — LangChain “thinking tool” node.
- **Config / connections:** Connected as an `ai_tool` to **Draft Video Prompt** and **Refine Prompt**.
- **Practical note:** This node doesn’t appear in the main execution path; it is available as a tool to the agents (depending on how the agent uses tools).
- **Failure modes:** Typically none; but if tool wiring is incorrect, agent tool calls could fail.

---

### Block 3 — Video Generation (Fal.ai) + Retrieval
**Overview:** Submits a prompt to Fal.ai’s queue endpoint, waits 1.5 minutes, then fetches the resulting output JSON (expected to include the video URL).  
**Nodes involved:** `Generate Video (Fal.ai)`, `Wait for Video`, `Fetch Video URL`

#### Node: Generate Video (Fal.ai)
- **Type / role:** `httpRequest` — initiates Fal.ai video generation.
- **Config choices:**
  - POST `https://queue.fal.run/fal-ai/veo3/fast`
  - JSON body:
    - `prompt`: `{{ $json.output }}` (from Draft Video Prompt agent)
    - `aspect_ratio`: `9:16`
  - Auth: `httpHeaderAuth` via n8n “genericCredentialType”.
- **Output expectations:** Fal.ai queue typically returns a `response_url` used for polling/fetching results.
- **Connections:** Output to **Wait for Video**.
- **Failure modes / edge cases:**
  - Invalid/expired Fal.ai key (401/403).
  - Fal.ai queue returns error payload or missing `response_url`.
  - Prompt rejected by provider policy.

#### Node: Wait for Video
- **Type / role:** `wait` — simple delay before fetching results.
- **Config choices:** 1.5 minutes.
- **Connections:** Output to **Fetch Video URL**.
- **Edge cases:**
  - Generation may take longer than 1.5 minutes; fetch may return “still processing” or incomplete response.

#### Node: Fetch Video URL
- **Type / role:** `httpRequest` — fetches generation result from Fal.ai.
- **Config choices:**
  - GET `={{ $json.response_url }}`
  - Auth: same header auth
  - **onError:** `continueErrorOutput` (workflow continues even if request errors; error data will be present in output item).
- **Connections:** Output to **Send Preview to Telegram**.
- **Output used later:** `{{$json.video.url}}` is referenced by **Upload to Blotato**.
- **Failure modes / edge cases:**
  - `response_url` missing or not carried forward → expression fails.
  - Response may not contain `video.url` (different schema, still processing, or error).
  - Because errors are continued, downstream nodes may run with missing `video.url` unless guarded.

---

### Block 4 — Preview + Approval Gate (Telegram)
**Overview:** Sends the video to the same Telegram chat for review, then waits for a free-text response. Approves if the response matches common approval phrases.  
**Nodes involved:** `Send Preview to Telegram`, `Wait for Approval`, `Approved?`

#### Node: Send Preview to Telegram
- **Type / role:** `telegram` — sends a video message.
- **Config choices:**
  - Operation: `sendVideo`
  - `chatId`: `={{ $('Telegram Message').item.json.message.chat.id }}`
  - `binaryData`: true
- **Important integration note:** This node is set to send **binary data**, but the upstream **Fetch Video URL** returns a URL in JSON, not a binary file. Unless another node downloads the file into binary, this may fail or send nothing.
- **Connections:** Output to **Wait for Approval**.
- **Failure modes / edge cases:**
  - Telegram node expects binary property (e.g., `data`) but none exists.
  - File too large for Telegram limits if you later add a download step.

#### Node: Wait for Approval
- **Type / role:** `telegram` with `sendAndWait` — sends a prompt message and waits for reply.
- **Config choices:**
  - Message: “Is this video ready to post? Reply 'Great' to post, or describe changes to regenerate.”
  - Response type: `freeText`
  - `chatId`: same chat as trigger.
- **Connections:** Output to **Approved?**
- **Edge cases:**
  - User replies in a different thread/chat; workflow won’t match.
  - Timeouts depend on n8n’s waiting mechanics and Telegram polling/webhook behavior.

#### Node: Approved?
- **Type / role:** `if` — branching gate.
- **Config choices:** Case-insensitive checks against `={{ $json.data.text }}`:
  - equals `great`
  - contains `post it`
  - equals `yes`
- **Outputs:**
  - **True branch (approved):** to **Generate Caption**
  - **False branch (needs changes):** to **Refine Prompt**
- **Edge cases:**
  - Extra punctuation (“Great!”) won’t match equals `great` (but could be solved by contains).
  - Non-English approvals not recognized.

---

### Block 5 — Refinement Loop (Rejected → Refine → Retry)
**Overview:** If the user rejects, the workflow revises the original draft prompt using feedback and regenerates a new video.  
**Nodes involved:** `Refine Prompt`, `Generate Video (Retry)`, `Wait for Retry`, `Fetch Video URL (Retry)`, `Send Preview (Retry)`

#### Node: Refine Prompt
- **Type / role:** LangChain agent — prompt corrector.
- **Config choices:**
  - Inputs:
    - Rejected Prompt: `{{ $('Draft Video Prompt').item.json.output }}`
    - User Feedback: `{{ $json.data.text }}`
  - Output: prompt string only.
  - `hasOutputParser: true` (expects to parse/validate agent output).
- **Model dependency:** from **OpenRouter Chat Model1**.
- **Connections:** Output to **Generate Video (Retry)**.
- **Failure modes:**
  - If the original draft prompt item isn’t accessible (item linking issues), expression may fail.
  - Output parser could throw if the agent returns unexpected format.

#### Node: Generate Video (Retry)
- **Type / role:** `httpRequest` — same Fal.ai endpoint as initial generation.
- **Config choices:** Uses `{{ $json.output }}` from Refine Prompt.
- **Connections:** Output to **Wait for Retry**.
- **Failure modes:** same as initial Fal.ai request.

#### Node: Wait for Retry
- **Type / role:** `wait` — 1.5 minutes.
- **Connections:** Output to **Fetch Video URL (Retry)**.

#### Node: Fetch Video URL (Retry)
- **Type / role:** `httpRequest` — fetches Fal.ai result for retry.
- **Config choices:**
  - URL: `={{ $('Generate Video (Retry)').item.json.response_url }}`
  - Auth: header auth
  - **onError:** `continueErrorOutput`
- **Connections:** Output to **Send Preview (Retry)**
- **Edge cases:** Same schema/timing issues as first fetch.

#### Node: Send Preview (Retry)
- **Type / role:** `telegram` sendVideo (binaryData true).
- **Config choices:** Sends to original chat id.
- **Connections:** No further connections shown.
- **Important logic note:** The refinement path **does not loop back** to “Wait for Approval” in the provided JSON. After sending the retry preview, the workflow ends unless manually extended.

---

### Block 6 — Publishing (Captions/Titles → Log → Upload → Post)
**Overview:** On approval, generates platform-ready text, logs metadata, uploads video to Blotato, and posts to multiple platforms using the returned Blotato media URL.  
**Nodes involved:** `Generate Caption`, `OpenRouter Chat Model`, `Generate Youtube Title`, `Log to Google Sheets`, `Upload to Blotato`, `Post to Instagram`, `Post to Facebook`, `Post to LinkedIn`, `Post to TikTok`, `Post to Youtube`

#### Node: Generate Caption
- **Type / role:** LangChain agent — caption + hashtags generator.
- **Config choices:**
  - Input: `Video prompt → {{ $('Draft Video Prompt').item.json.output }}`
  - Output format:
    - Three short paragraphs separated by line breaks
    - Blank line
    - Hashtags on one line
  - `needsFallback: true` (node can attempt fallback behavior if model fails—depends on node implementation/version).
- **Model dependency:** from **OpenRouter Chat Model**.
- **Connections:** Output to **Generate Youtube Title**.
- **Failure modes:** LLM failures; output not matching required format.

#### Node: OpenRouter Chat Model
- **Type / role:** `lmChatOpenRouter` — model provider for caption/title generation.
- **Config choices:** Model `z-ai/glm-4.5-air:free`.
- **Connections:** Supplies model to **Generate Caption** and **Generate Youtube Title**.
- **Failure modes:** Rate limits for free model tiers; model availability.

#### Node: Generate Youtube Title
- **Type / role:** LangChain agent — creates a short YouTube title.
- **Config choices:**
  - Inputs:
    - Captions: `{{ $json.output }}` (from Generate Caption)
    - Prompt: `{{ $('Draft Video Prompt').item.json.output }}`
  - Output: “3–5 words”.
  - `needsFallback: true`.
- **Connections:** Output to **Log to Google Sheets**.

#### Node: Log to Google Sheets
- **Type / role:** `googleSheets` — append/update logging.
- **Config choices:**
  - Operation: `appendOrUpdate`
  - `documentId` and `sheetName` are present but **not filled** in the JSON (must be selected in UI).
- **Connections:** Output to **Upload to Blotato**.
- **Failure modes:**
  - Missing document/sheet selection.
  - Google OAuth credential issues.
  - Append/update requires a key column configuration in many cases; otherwise behavior may be unexpected.

#### Node: Upload to Blotato
- **Type / role:** `@blotato/...blotato` — uploads media by URL to Blotato.
- **Config choices:**
  - Resource: `media`
  - Media URL: `={{ $('Fetch Video URL').item.json.video.url }}`
- **Connections:** Fans out to all “Post to …” nodes.
- **Output used downstream:** downstream nodes reference `{{ $json.url }}` as the media URL coming back from Blotato.
- **Edge cases:**
  - If the approved path came from a *retry* video, this node still uses **Fetch Video URL** (original), not **Fetch Video URL (Retry)**. That means it may upload/post the wrong video unless adjusted.
  - If Fal.ai response schema differs, `video.url` may be undefined.

#### Node: Post to Instagram
- **Type / role:** Blotato post creation.
- **Config choices:**
  - Account selection: `accountId` is empty in JSON (must be configured).
  - Text: `={{ $('Generate Caption').item.json.output }}`
  - Media: `={{ $json.url }}`
- **Input:** from **Upload to Blotato**.
- **Failure modes:** Missing account linkage; platform auth; media format constraints.

#### Node: Post to Facebook
- **Type / role:** Blotato post creation (facebook).
- **Config choices:**
  - `platform: facebook`
  - Requires `accountId` and `facebookPageId` (both empty in JSON).
  - Text: caption output
  - Media: Blotato media URL
- **Failure modes:** Missing page ID; permissions.

#### Node: Post to LinkedIn
- **Type / role:** Blotato post creation (linkedin).
- **Config choices:** `platform: linkedin`, accountId empty, caption text, media URL.
- **Failure modes:** LinkedIn video constraints; permissions.

#### Node: Post to TikTok
- **Type / role:** Blotato post creation (tiktok).
- **Config choices:** `platform: tiktok`, accountId empty, caption text, media URL.
- **Failure modes:** TikTok upload requirements; review/processing delays.

#### Node: Post to Youtube
- **Type / role:** Blotato post creation (youtube).
- **Config choices:**
  - `platform: youtube`
  - Text: `={{ $('Generate Caption').item.json.output }} #shorts`
  - Media: `={{ $json.url }}`
  - Title: `={{ $('Generate Youtube Title').item.json.output }}`
  - accountId empty.
- **Failure modes:** Title length/validation; Shorts requirements (vertical, <=60s usually—this is 8s so OK); permissions.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Message | telegramTrigger | Entry point from Telegram message | — | Draft Video Prompt | ### 🤖 AI Video Generator & Publisher; This workflow acts as your social media powerhouse. |
| Draft Video Prompt | langchain.agent | Draft cinematic 8s video prompt | Telegram Message | Generate Video (Fal.ai) | ## 1. Generation Phase; Drafts prompt and generates video via Fal.ai |
| Generate Video (Fal.ai) | httpRequest | Submit generation job to Fal.ai queue | Draft Video Prompt | Wait for Video | ## 1. Generation Phase; Drafts prompt and generates video via Fal.ai |
| Wait for Video | wait | Delay before polling Fal.ai result | Generate Video (Fal.ai) | Fetch Video URL | ## 1. Generation Phase; Drafts prompt and generates video via Fal.ai |
| Fetch Video URL | httpRequest | Fetch Fal.ai result (expects video.url) | Wait for Video | Send Preview to Telegram | ## 1. Generation Phase; Drafts prompt and generates video via Fal.ai |
| Send Preview to Telegram | telegram | Send generated video preview to Telegram | Fetch Video URL | Wait for Approval | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Wait for Approval | telegram (sendAndWait) | Ask approval + wait for free-text reply | Send Preview to Telegram | Approved? | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Approved? | if | Branch on approval keywords | Wait for Approval | Generate Caption; Refine Prompt | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Refine Prompt | langchain.agent | Revise prompt using rejection feedback | Approved? (false) | Generate Video (Retry) | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Generate Video (Retry) | httpRequest | Submit retry generation to Fal.ai | Refine Prompt | Wait for Retry | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Wait for Retry | wait | Delay before polling retry result | Generate Video (Retry) | Fetch Video URL (Retry) | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Fetch Video URL (Retry) | httpRequest | Fetch retry Fal.ai result | Wait for Retry | Send Preview (Retry) | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Send Preview (Retry) | telegram | Send retry preview to Telegram | Fetch Video URL (Retry) | — | ## 2. Feedback & Refinement Loop; Wait for human approval. If changes are needed, refine and regenerate. |
| Generate Caption | langchain.agent | Create caption + hashtags | Approved? (true) | Generate Youtube Title | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Generate Youtube Title | langchain.agent | Create short YouTube title | Generate Caption | Log to Google Sheets | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Log to Google Sheets | googleSheets | Log post metadata | Generate Youtube Title | Upload to Blotato | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Upload to Blotato | blotato | Upload media once for reuse | Log to Google Sheets | Post to Instagram; Post to Facebook; Post to LinkedIn; Post to Youtube; Post to TikTok | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Post to Instagram | blotato | Publish to Instagram | Upload to Blotato | — | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Post to Facebook | blotato | Publish to Facebook page | Upload to Blotato | — | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Post to LinkedIn | blotato | Publish to LinkedIn | Upload to Blotato | — | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Post to TikTok | blotato | Publish to TikTok | Upload to Blotato | — | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| Post to Youtube | blotato | Publish YouTube Shorts | Upload to Blotato | — | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| OpenRouter Chat Model | lmChatOpenRouter | LLM provider for captions/title | — | (to agents) Generate Caption; Generate Youtube Title | ## 3. Multi-Channel Publishing; If approved: Generate captions, log to Sheets, and post via Blotato. |
| OpenRouter Chat Model1 | lmChatOpenRouter | LLM provider for draft/refine prompts | — | (to agents) Draft Video Prompt; Refine Prompt | ## 1. Generation Phase; Drafts prompt and generates video via Fal.ai |
| Think | toolThink | Optional agent tool | — | (tool link) Draft Video Prompt; Refine Prompt |  |

Sticky note nodes (comments only, not executed): `Overview`, `Section 1`, `Section 2`, `Section 3`.

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger**
   - Add node: **Telegram Trigger** (`Telegram Message`)
   - Updates: `message`
   - Configure Telegram credentials (bot token) and activate webhook (n8n handles this when active).

2. **Add LLM model for prompt generation**
   - Add node: **OpenRouter Chat Model** (`OpenRouter Chat Model1`)
   - Choose model: `openai/gpt-5.2` (or your preferred)
   - Configure OpenRouter API credential.

3. **Draft the video prompt (agent)**
   - Add node: **AI Agent (LangChain)** (`Draft Video Prompt`)
   - Prompt type: “Define”
   - Paste the role/goal/rules prompt (ensuring it references `{{$json.message.text}}`)
   - Connect **Telegram Message → Draft Video Prompt**
   - Connect **OpenRouter Chat Model1 → Draft Video Prompt** via **AI Language Model** connection.

4. **Submit to Fal.ai**
   - Add node: **HTTP Request** (`Generate Video (Fal.ai)`)
   - Method: POST
   - URL: `https://queue.fal.run/fal-ai/veo3/fast`
   - Body: JSON with:
     - `prompt`: `{{ $json.output }}`
     - `aspect_ratio`: `9:16`
   - Auth: Header Auth (create credential that sets the required Fal.ai header, e.g., `Authorization: Key ...` depending on Fal.ai requirement).
   - Connect **Draft Video Prompt → Generate Video (Fal.ai)**

5. **Wait + fetch result**
   - Add node: **Wait** (`Wait for Video`) set to 1.5 minutes; connect from Fal request.
   - Add node: **HTTP Request** (`Fetch Video URL`)
     - URL: `{{ $json.response_url }}`
     - Auth: same Fal header auth
     - Enable “Continue on Fail” (equivalent to `continueErrorOutput`)
   - Connect **Generate Video (Fal.ai) → Wait for Video → Fetch Video URL**

6. **Send preview to Telegram (important: choose URL vs binary)**
   - Add node: **Telegram** (`Send Preview to Telegram`)
   - Operation: `sendVideo`
   - Chat ID: `{{ $('Telegram Message').item.json.message.chat.id }}`
   - If you keep **binaryData = true**, insert an extra **HTTP Request (Download)** node before this to download `video.url` into binary.
     - Alternatively, switch Telegram node to send by **video URL** (preferred if supported by your Telegram node settings).
   - Connect **Fetch Video URL → Send Preview to Telegram**

7. **Approval message with wait**
   - Add node: **Telegram** (`Wait for Approval`)
   - Operation: `sendAndWait`
   - Message: approval prompt text
   - Response type: `freeText`
   - Chat ID: same expression as above
   - Connect **Send Preview to Telegram → Wait for Approval**

8. **IF approval gate**
   - Add node: **IF** (`Approved?`)
   - Conditions against `{{ $json.data.text }}` (case-insensitive):
     - equals `great`
     - contains `post it`
     - equals `yes`
   - Connect **Wait for Approval → Approved?**

9. **Refinement path (false branch)**
   - Add **OpenRouter Chat Model** already created in step 2 (reuse).
   - Add node: **AI Agent** (`Refine Prompt`)
     - Inputs:
       - Rejected Prompt: `{{ $('Draft Video Prompt').item.json.output }}`
       - User Feedback: `{{ $json.data.text }}`
     - Output: prompt string only
   - Connect **OpenRouter Chat Model1 → Refine Prompt** via AI Language Model.
   - Connect **Approved? (false) → Refine Prompt**
   - Add **HTTP Request** (`Generate Video (Retry)`) same as Fal.ai submit, using `{{ $json.output }}`.
   - Add **Wait** (`Wait for Retry`) 1.5 minutes.
   - Add **HTTP Request** (`Fetch Video URL (Retry)`) using `{{ $('Generate Video (Retry)').item.json.response_url }}`.
   - Add **Telegram sendVideo** (`Send Preview (Retry)`) to same chat.
   - Connect: **Refine Prompt → Generate Video (Retry) → Wait for Retry → Fetch Video URL (Retry) → Send Preview (Retry)**
   - (Optional but recommended) Add another **Wait for Approval** + **Approved?** loop back if you want repeated iterations.

10. **Publishing path (true branch): captions**
   - Add node: **OpenRouter Chat Model** (`OpenRouter Chat Model`)
     - Model: `z-ai/glm-4.5-air:free` (or another)
     - Configure OpenRouter credential.
   - Add node: **AI Agent** (`Generate Caption`)
     - Input: `{{ $('Draft Video Prompt').item.json.output }}`
     - Output format instructions (3 paragraphs + blank line + hashtags)
   - Connect **Approved? (true) → Generate Caption**
   - Connect **OpenRouter Chat Model → Generate Caption** via AI Language Model.

11. **YouTube title**
   - Add node: **AI Agent** (`Generate Youtube Title`)
     - Captions: `{{ $json.output }}`
     - Prompt: `{{ $('Draft Video Prompt').item.json.output }}`
     - Output: 3–5 words
   - Connect **OpenRouter Chat Model → Generate Youtube Title** via AI Language Model.
   - Connect **Generate Caption → Generate Youtube Title**

12. **Google Sheets logging**
   - Add node: **Google Sheets** (`Log to Google Sheets`)
   - Operation: `appendOrUpdate`
   - Select **Document ID** and **Sheet name**
   - Configure Google credentials (OAuth2).
   - Connect **Generate Youtube Title → Log to Google Sheets**
   - (You must define which columns/keys are used for “update”; otherwise treat it as append in practice.)

13. **Upload to Blotato**
   - Add node: **Blotato** (`Upload to Blotato`)
   - Resource: `media`
   - Media URL: `{{ $('Fetch Video URL').item.json.video.url }}`
   - Configure Blotato credentials.
   - Connect **Log to Google Sheets → Upload to Blotato**

14. **Post to social platforms (fan-out)**
   - Add five Blotato post nodes:
     - `Post to Instagram` (platform default/Instagram)
     - `Post to Facebook` (platform facebook + facebookPageId)
     - `Post to LinkedIn` (platform linkedin)
     - `Post to TikTok` (platform tiktok)
     - `Post to Youtube` (platform youtube + title)
   - For each:
     - Set `accountId` (and `facebookPageId` for Facebook).
     - Text: `{{ $('Generate Caption').item.json.output }}` (YouTube adds ` #shorts`)
     - Media URLs: `{{ $json.url }}` (from Upload to Blotato output)
     - YouTube title: `{{ $('Generate Youtube Title').item.json.output }}`
   - Connect **Upload to Blotato → each Post node**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Video Generator & Publisher… Setup Steps: Credentials for OpenAI, Fal.ai, Telegram, Google Sheets, Blotato; select Sheet ID; select Blotato Account IDs.” | From sticky note “Overview” |
| “Generation Phase: Drafts prompt and generates video via Fal.ai” | From sticky note “Section 1” |
| “Feedback & Refinement Loop: Wait for human approval. If changes are needed, refine and regenerate.” | From sticky note “Section 2” |
| “Multi-Channel Publishing: If approved: Generate captions, log to Sheets, and post via Blotato.” | From sticky note “Section 3” |

**Notable implementation gaps to address when modifying/reproducing**
- Telegram preview nodes are configured for **binary** sending, but the workflow fetches **URLs**. Add a download-to-binary step or switch Telegram sendVideo to URL mode.
- The retry branch ends after sending the retry preview (no second approval gate).
- Publishing uploads the **original** video (`Fetch Video URL`) even after a retry; to publish the regenerated video, route/merge the correct “final approved” video URL into `Upload to Blotato`.