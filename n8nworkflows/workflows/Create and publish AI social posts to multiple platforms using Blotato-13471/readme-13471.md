Create and publish AI social posts to multiple platforms using Blotato

https://n8nworkflows.xyz/workflows/create-and-publish-ai-social-posts-to-multiple-platforms-using-blotato-13471


# Create and publish AI social posts to multiple platforms using Blotato

## 1. Workflow Overview

**Workflow name:** AI Social Media Automation for Multiple Platforms using Blotato  
**User-provided title:** Create and publish AI social posts to multiple platforms using Blotato

**Purpose:**  
This workflow lets a user send a YouTube link to a Telegram bot, automatically transcribes the video via Blotato, fact-checks/summarizes the content into carousel-ready bullet points using Blotato “AI Research” (Perplexity query), generates multiple visual formats (images/videos) using several Blotato templates, requests Telegram approval for the Instagram carousel, and then publishes content to multiple social platforms (Instagram, TikTok, Facebook, LinkedIn, Twitter/X) through Blotato.

### 1.1 Input Reception (Telegram)
Receives a Telegram message (expected: a YouTube URL) and starts the automation.

### 1.2 Transcription Job Creation + Polling
Creates a Blotato “source” transcription job, then repeatedly checks until the job is completed.

### 1.3 Fact-checking + Bullet Summary (Perplexity via Blotato) + Polling
Uses Blotato “sourceType=perplexity-query” to validate and summarize the transcript into **5 detailed bullet points** for an Instagram carousel, then polls until the summary is ready.

### 1.4 Multi-template Visual Generation (Parallel branches)
From the generated bullet summary, the workflow triggers **five** visual generation branches (carousel image cards, AI video w/ voice, slideshow, “breaking news” infographic, monocolor carousel). Each branch polls until its media is ready.

### 1.5 Approval & Publishing (Telegram + Blotato posting)
- Instagram branch: sends image to Telegram and waits for approval; if approved, posts to Instagram.
- Other branches: once assets are ready, posts to TikTok, Facebook, LinkedIn, and Twitter/X and sends “Published” notifications in Telegram.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Telegram Trigger (Entry Point)
**Overview:** Starts the workflow when a Telegram message is received. The message text is used as the YouTube URL input.  
**Nodes involved:** `Telegram Trigger`

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point listening to Telegram updates (`message`).
- **Configuration (interpreted):**
  - Update types: `message`
  - Expects `$json.message.text` to contain a URL (YouTube link).
- **Key variables:**
  - `{{$json.message.text}}` (source URL)
  - `{{$json.message.chat.id}}` (reply target)
- **Connections:**
  - Output → `Create Transcription`
- **Credentials:** Telegram API credential `Telegram_Blotato`
- **Edge cases / failures:**
  - Bot not added to chat / privacy settings restrict messages.
  - Non-URL input or unsupported URL may cause downstream Blotato errors.
  - Telegram credential revoked/invalid.

---

### Block 2.2 — Transcription Creation & Completion Polling
**Overview:** Creates a Blotato transcription job for the YouTube URL and polls until the transcription status is completed.  
**Nodes involved:** `Create Transcription`, `Get Transcription`, `If1`, `Wait`

#### Node: Create Transcription
- **Type / role:** `@blotato/...blotato` — creates a “source” job from a URL.
- **Configuration:**
  - Resource: `source`
  - `sourceUrl`: `{{$json.message.text}}`
  - Custom instructions: “create all transcription of this youtube video and return the title in the first line.”
- **Outputs:** returns an object containing an `id` used to poll.
- **Connections:** Output → `Get Transcription`
- **Credentials:** `Blotato account`
- **Edge cases:**
  - URL not reachable, private video, geo-blocked, or too long.
  - Instruction-following may vary; “title in first line” is not strictly guaranteed.

#### Node: Get Transcription
- **Type / role:** Blotato — fetches the created source job.
- **Configuration:**
  - Resource: `source`
  - Operation: `get`
  - `sourceId`: `{{$json.id}}` (from Create Transcription)
- **Connections:** Output → `If1`
- **Edge cases:** API rate limits; job still processing.

#### Node: If1
- **Type / role:** `if` — checks whether transcription is ready.
- **Condition:**
  - `{{$json.status}}` equals `=completed` (note: the right side includes a leading `=`)
- **Connections:**
  - True → `Check with perplexity ai`
  - False → `Wait` (then re-check)
- **Potential issue (important):**
  - The comparison value is **`=completed`** (with an equals sign). If Blotato returns `completed` (without `=`), this condition will never pass, causing an infinite loop.
  - Recommended fix: set right value to `completed`.

#### Node: Wait
- **Type / role:** `wait` — delays polling to avoid hammering API.
- **Configuration:** waits 20 seconds.
- **Connections:** Output → `Get Transcription`
- **Edge cases:** long running jobs; overall execution time limits (n8n cloud/timeouts).

---

### Block 2.3 — Fact-checking & Bullet Summary (Perplexity via Blotato) + Polling
**Overview:** Sends the transcript to a Perplexity-based research query via Blotato, then retrieves and validates the “content” output until it’s completed.  
**Nodes involved:** `Check with perplexity ai`, `Get content`, `If`, `Wait1`

#### Node: Check with perplexity ai
- **Type / role:** Blotato — creates a new source based on a Perplexity query.
- **Configuration:**
  - Resource: `source`
  - Source type: `perplexity-query`
  - Query prompt:  
    `from this transcription, and Check all information and summarize in 5 detailed bullet points for an instagram carousel: the transcription : {{ $json.content }}`
  - Depends on transcription text being available in `{{$json.content}}` from prior `Get Transcription`.
- **Connections:** Output → `Get content`
- **Edge cases:**
  - If transcript `content` is missing/empty, summary will be poor or fail.
  - Perplexity query may produce variable format; downstream templates expect bullet-point-like content.

#### Node: Get content
- **Type / role:** Blotato — polls the Perplexity-query source by id.
- **Configuration:**
  - Resource: `source`
  - Operation: `get`
  - `sourceId`: `{{$json.id}}`
- **Connections:** Output → `If`
- **Edge cases:** rate limits; still “processing”.

#### Node: If
- **Type / role:** `if` — checks if research/summarization is completed.
- **Condition:**
  - `{{$json.status}}` equals `=completed` (same risk as `If1`)
- **Connections:**
  - True → triggers **five parallel Create visual nodes** (`Create visual`, `Create visual1`, `Create visual3`, `Create visual4`, `Create visual5`)
  - False → `Wait1` then poll again
- **Potential issue (important):**
  - Same `=completed` string mismatch risk → infinite polling.
  - Recommended fix: right value should likely be `completed`.

#### Node: Wait1
- **Type / role:** `wait` — delays polling.
- **Configuration:** 20 seconds.
- **Connections:** Output → `Get content`

---

### Block 2.4 — Instagram Carousel (Tweet-card style) + Approval + Publish
**Overview:** Generates a quote-card carousel style visual, polls until done, sends a preview image to Telegram, waits for approval, then posts to Instagram via Blotato and confirms in Telegram.  
**Nodes involved:** `Create visual`, `Get visual`, `If Else`, `Wait4`, `Send a photo message`, `post for approval`, `Create Instagram`, `Send a text message`

#### Node: Create visual
- **Type / role:** Blotato — generates a visual using a template.
- **Configuration:**
  - Resource: `video` (Blotato uses “video” resource for templated renders; output may be images)
  - Prompt: `create instagram carousel with this detailed bullet points : {{ $json.content }}`
  - Template: **Twitter/X style quote cards with minimal style** (`/base/v2/tweet-card/.../v1`)
  - Template inputs include branding:
    - theme: dark
    - handle: `doc.firass`
    - authorName: `Dr. FIRAS`
    - verified: true
    - aspectRatio: 9:16
    - profileImage: `https://www.dr-firas.com/logo.jpg`
- **Connections:** Output → `Get visual`
- **Edge cases:** template expects quotes array string; prompt-to-template mapping may be imperfect.

#### Node: Get visual
- **Type / role:** Blotato — polls the render job.
- **Configuration:**
  - Resource: `video`
  - Operation: `get`
  - `videoId`: `{{ $('Create visual').item.json.item.id }}`
- **Connections:** Output → `If Else`
- **Outputs used downstream:** `item.status`, `item.imageUrls`
- **Edge cases:** `imageUrls` may be empty until done.

#### Node: If Else
- **Type / role:** `if` — checks if render finished.
- **Condition:** `{{$json.item.status}}` equals `done`
- **Connections:**
  - True → `Send a photo message` (preview + approval flow)
  - False → `Wait4` then `Get visual`
- **Edge cases:** status could be `error`; not handled explicitly.

#### Node: Wait4
- **Type / role:** wait 15 seconds (polling delay)
- **Connections:** Output → `Get visual`

#### Node: Send a photo message
- **Type / role:** Telegram sendPhoto — sends first image as preview.
- **Configuration:**
  - File: `{{$json.item.imageUrls[0]}}`
  - Chat: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
- **Connections:** Output → `post for approval`
- **Edge cases:** Telegram needs a direct reachable URL; if Blotato URL is not public or expires, sendPhoto fails.

#### Node: post for approval
- **Type / role:** Telegram `sendAndWait` — asks for approval and waits for user interaction.
- **Configuration:**
  - Message: “Here is the post for approval.”
  - Chat ID from trigger chat
  - Operation: `sendAndWait` (creates a “waiting” execution until reply/interaction)
- **Connections:** Output → `Create Instagram`
- **Edge cases:**
  - If user never responds, execution remains waiting.
  - The workflow does not parse approval text/buttons; it proceeds once the wait completes (behavior depends on n8n Telegram node “sendAndWait” response mode).

#### Node: Create Instagram
- **Type / role:** Blotato — publishes to Instagram.
- **Configuration:**
  - Platform: (not explicitly set; node name suggests Instagram; uses Blotato posting fields)
  - Account: `doc.firass` (accountId `11892`)
  - Post text: `Comment "learn" and I'll DM you more information!`
  - Media URLs: `{{ $('If Else').item.json.item.imageUrls }}`
- **Connections:** Output → `Send a text message`
- **Edge cases:** Instagram requirements (aspect ratio, number of slides, file formats); Blotato account must be connected and authorized.

#### Node: Send a text message
- **Type / role:** Telegram sendMessage — confirmation.
- **Configuration:** text `Published`
- **Connections:** end
- **Edge cases:** Telegram message failures due to chat permissions.

**Sticky note coverage:**  
- “## Template : Tweet Card Carousel with Minimal Style” applies to this block’s template nodes (Create visual / Get visual / polling chain).

---

### Block 2.5 — TikTok AI Video with Voice (Download → Post → Notify)
**Overview:** Generates an AI video with voiceover, polls until done, downloads the file via HTTP request (to make it accessible as a media URL in the flow), posts to TikTok via Blotato, then notifies in Telegram.  
**Nodes involved:** `Create visual1`, `Get visual1`, `If Else1`, `Wait5`, `Get image/video1`, `Create Tiktok`, `Send notification : Published`

#### Node: Create visual1
- **Type / role:** Blotato render using **AI Video with AI Voice** template.
- **Configuration:**
  - Prompt: `create ai video with ai voice with this detailed bullet points : {{ $json.content }}`
  - Template: `/base/v2/ai-story-video/.../v1`
  - Inputs: voiceName “Will (American, friendly)”, aspectRatio 9:16, aiImageModel `replicate/recraft-ai/recraft-v3`
- **Connections:** Output → `Get visual1`
- **Edge cases:** long render times; voice model availability.

#### Node: Get visual1
- **Type / role:** Blotato poll render
- **Configuration:** `videoId: {{ $('Create visual1').item.json.item.id }}`
- **Connections:** Output → `If Else1`
- **Outputs:** `item.status`, `item.mediaUrl`

#### Node: If Else1
- **Type / role:** checks render completion
- **Condition:** `{{$json.item.status}} == done`
- **Connections:**
  - True → `Get image/video1`
  - False → `Wait5` → `Get visual1`

#### Node: Wait5
- Wait 15 seconds; loops back.

#### Node: Get image/video1
- **Type / role:** `httpRequest` — downloads media (GET).
- **Configuration:** URL `{{ $('Get visual1').item.json.item.mediaUrl }}`
- **Connections:** Output → `Create Tiktok`
- **Potential integration nuance:**
  - As configured, it downloads data but the TikTok node uses `{{$json.item.mediaUrl}}` (see below), not the HTTP response. This mismatch can break posting unless Blotato node accepts the same URL without downloading.
  - If the intention was to pass binary, additional configuration is needed (download as file, then upload or host).

#### Node: Create Tiktok
- **Type / role:** Blotato publish to TikTok.
- **Configuration:**
  - Platform: `tiktok`
  - Account: `eliteshicos` (accountId `30526`)
  - Post text: `Comment "learn" and I'll DM you more information!`
  - Media URLs: `{{ $json.item.mediaUrl }}`
- **Connections:** Output → `Send notification : Published`
- **Potential issue:**
  - At this point `$json` comes from `Get image/video1` (HTTP node), which typically does **not** output `item.mediaUrl`. Unless n8n keeps prior JSON merged (it doesn’t by default), this expression can be undefined.
  - Recommended fix: set `postContentMediaUrls` to `{{ $('Get visual1').item.json.item.mediaUrl }}`.

#### Node: Send notification : Published
- Telegram confirmation to the trigger chat.

**Sticky note coverage:**  
- “## Template : AI Video with AI Voice” applies to this block’s template nodes.

---

### Block 2.6 — Facebook Image Slideshow Post (Render → Poll → Publish → Notify)
**Overview:** Generates an “Image Slideshow with Prominent Text”, polls until done, posts to Facebook Page via Blotato, then notifies Telegram.  
**Nodes involved:** `Create visual3`, `Get visual3`, `If Else3`, `Wait9`, `Create Facebook2`, `Send notification : Published2`

#### Node: Create visual3
- **Type / role:** Blotato render (slideshow template)
- **Configuration:**
  - Template: `/base/v2/images-with-text/.../v1` (Image Slideshow with Prominent Text)
  - Prompt: uses `{{$json.content}}`
  - aspectRatio 9:16
- **Connections:** Output → `Get visual3`

#### Node: Get visual3
- Poll render by id; output includes `item.imageUrls` (used as media URLs).

#### Node: If Else3
- Checks `item.status == done`
- True → `Create Facebook2`
- False → `Wait9` → `Get visual3`

#### Node: Create Facebook2
- **Type / role:** Blotato publish to Facebook.
- **Configuration:**
  - Platform: `facebook`
  - Account: “Firass Ben” (accountId `1759`)
  - Facebook Page: “Dr. Firas” (pageId `101603614680195`)
  - Text: `Comment "learn" and I'll DM you more information!`
  - Media: `{{ $('Get visual3').item.json.item.imageUrls }}`
- **Connections:** Output → `Send notification : Published2`
- **Edge cases:** Page permission/auth, media format constraints.

#### Node: Send notification : Published2
- Telegram confirmation

**Sticky note coverage:**  
- “## Template : Image Slideshow with Prominent Text” applies to this block.

---

### Block 2.7 — LinkedIn “Breaking News” Infographic (Render → Poll → Download → Post → Notify)
**Overview:** Generates a “Breaking News” style infographic, polls until done, downloads image via HTTP request, posts to LinkedIn via Blotato, then notifies Telegram.  
**Nodes involved:** `Create visual4`, `Get visual4`, `If Else4`, `Wait12`, `Get image/video2`, `Create Linkedin3`, `Send notification : Published3`

#### Node: Create visual4
- **Type / role:** Blotato render
- **Configuration:**
  - Template: “Generate a TV news broadcast style infographic…” (templateId `8800be71-...`)
  - Prompt uses `{{$json.content}}`
  - Template input: footerText `Dr. FIRAS`
- **Connections:** Output → `Get visual4`

#### Node: Get visual4
- Poll render; output includes `item.imageUrls`.

#### Node: If Else4
- Checks `item.status == done`
- True → `Get image/video2`
- False → `Wait12` → `Get visual4`

#### Node: Get image/video2
- **Type / role:** HTTP GET
- **Configuration:** URL `{{ $json.item.imageUrls[0] }}`
- **Connections:** Output → `Create Linkedin3`
- **Potential nuance:** same as TikTok branch—download may not be used; posting uses `$('Get visual4')...` so it’s safe here.

#### Node: Create Linkedin3
- **Type / role:** Blotato publish to LinkedIn.
- **Configuration:**
  - Platform: `linkedin`
  - Account: “Samuel Amalric” (accountId `1446`) (note: branding mismatch vs Dr. Firas)
  - Text: `Comment "learn" and I'll DM you more information!`
  - Media: `{{ $('Get visual4').item.json.item.imageUrls[0] }}`
- **Connections:** Output → `Send notification : Published3`

#### Node: Send notification : Published3
- Telegram confirmation

**Sticky note coverage:**  
- “## Template : Breaking News” applies to this block.

---

### Block 2.8 — Twitter/X Monocolor Carousel (Render → Poll → Download → Post → Notify)
**Overview:** Generates a “Tutorial Carousel with Monocolor Background”, polls, downloads image, posts to Twitter/X via Blotato, then notifies Telegram.  
**Nodes involved:** `Create visual5`, `Get visual5`, `If Else5`, `Wait15`, `Get image/video3`, `Create Twitter`, `Send notification : Published4`

#### Node: Create visual5
- **Type / role:** Blotato render
- **Configuration:**
  - Template: `/base/v2/tutorial-carousel/.../v1`
  - Inputs: authorName `Dr. FIRAS`, companyName `@doc.firass`, aspectRatio 4:5, profileImage URL set
  - Prompt uses `{{$json.content}}`
- **Connections:** Output → `Get visual5`

#### Node: Get visual5
- Poll render; output includes `item.imageUrls`.

#### Node: If Else5
- Checks `item.status == done`
- True → `Get image/video3`
- False → `Wait15` → `Get visual5`

#### Node: Get image/video3
- HTTP GET `{{ $json.item.imageUrls[0] }}`
- Output → `Create Twitter`

#### Node: Create Twitter
- **Type / role:** Blotato publish to Twitter/X.
- **Configuration:**
  - Platform: `twitter`
  - Account: `Docteur_Firas` (accountId `1289`)
  - Text: `Comment "learn" and I'll DM you more information!`
  - Media: `{{ $json.item.imageUrls }}`
- **Potential issue:** At this point `$json` comes from `Get image/video3` (HTTP) and may not contain `item.imageUrls`.  
  - Recommended fix: use `{{ $('Get visual5').item.json.item.imageUrls }}`.

#### Node: Send notification : Published4
- Telegram confirmation

**Sticky note coverage:**  
- “## Template : Tutorial Carousel with Monocolor Background” applies to this block.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point (Telegram message) | — | Create Transcription | # 🚀 AI Social Media Automation for Multiple Platforms using Blotato… Notion link: https://automatisation.notion.site/AI-Social-Media-Automation-for-Multiple-Platforms-using-Blotato-30a3d6550fd98051adc4eaa6e676478c?source=copy_link |
| Create Transcription | blotato | Create transcript job from YouTube URL | Telegram Trigger | Get Transcription | ## Step 1 – Retrieve the video transcription |
| Get Transcription | blotato | Poll/get transcription result | Create Transcription, Wait | If1 | ## Step 1 – Retrieve the video transcription |
| If1 | if | Check transcription job completion | Get Transcription | Check with perplexity ai, Wait | ## Step 1 – Retrieve the video transcription |
| Wait | wait | Delay before re-polling transcription | If1 (false) | Get Transcription | ## Step 1 – Retrieve the video transcription |
| Check with perplexity ai | blotato | Create fact-check + summary job (Perplexity query) | If1 (true) | Get content | ## Step 2 – Create an optimized prompt using Blotato AI Research |
| Get content | blotato | Poll/get fact-check summary result | Check with perplexity ai, Wait1 | If | ## Step 2 – Create an optimized prompt using Blotato AI Research |
| If | if | Check summary job completion + fan-out | Get content | Create visual; Create visual1; Create visual3; Create visual4; Create visual5; Wait1 | ## Step 2 – Create an optimized prompt using Blotato AI Research |
| Wait1 | wait | Delay before re-polling summary | If (false) | Get content | ## Step 2 – Create an optimized prompt using Blotato AI Research |
| Create visual | blotato | Generate carousel (tweet-card template) | If (true) | Get visual | ## Template : Tweet Card Carousel with Minimal Style |
| Get visual | blotato | Poll/get carousel render | Create visual, Wait4 | If Else | ## Template : Tweet Card Carousel with Minimal Style |
| If Else | if | Check carousel render completion | Get visual | Send a photo message, Wait4 | ## Template : Tweet Card Carousel with Minimal Style |
| Wait4 | wait | Delay before re-polling carousel render | If Else (false) | Get visual | ## Template : Tweet Card Carousel with Minimal Style |
| Send a photo message | telegram | Send carousel preview image to Telegram | If Else (true) | post for approval | ## Template : Tweet Card Carousel with Minimal Style |
| post for approval | telegram | Wait for approval via Telegram | Send a photo message | Create Instagram | ## Template : Tweet Card Carousel with Minimal Style |
| Create Instagram | blotato | Publish carousel to Instagram | post for approval | Send a text message | ## Template : Tweet Card Carousel with Minimal Style |
| Send a text message | telegram | Notify “Published” (Instagram) | Create Instagram | — |  |
| Create visual1 | blotato | Generate AI video w/ voice | If (true) | Get visual1 | ## Template : AI Video with AI Voice |
| Get visual1 | blotato | Poll/get AI video render | Create visual1, Wait5 | If Else1 | ## Template : AI Video with AI Voice |
| If Else1 | if | Check AI video render completion | Get visual1 | Get image/video1, Wait5 | ## Template : AI Video with AI Voice |
| Wait5 | wait | Delay before re-polling AI video render | If Else1 (false) | Get visual1 | ## Template : AI Video with AI Voice |
| Get image/video1 | httpRequest | HTTP GET media URL (video) | If Else1 (true) | Create Tiktok | ## Template : AI Video with AI Voice |
| Create Tiktok | blotato | Publish to TikTok | Get image/video1 | Send notification : Published | ## Template : AI Video with AI Voice |
| Send notification : Published | telegram | Notify “Published” (TikTok) | Create Tiktok | — |  |
| Create visual3 | blotato | Generate image slideshow | If (true) | Get visual3 | ## Template : Image Slideshow with Prominent Text |
| Get visual3 | blotato | Poll/get slideshow render | Create visual3, Wait9 | If Else3 | ## Template : Image Slideshow with Prominent Text |
| If Else3 | if | Check slideshow render completion | Get visual3 | Create Facebook2, Wait9 | ## Template : Image Slideshow with Prominent Text |
| Wait9 | wait | Delay before re-polling slideshow render | If Else3 (false) | Get visual3 | ## Template : Image Slideshow with Prominent Text |
| Create Facebook2 | blotato | Publish to Facebook Page | If Else3 (true) | Send notification : Published2 | ## Template : Image Slideshow with Prominent Text |
| Send notification : Published2 | telegram | Notify “Published” (Facebook) | Create Facebook2 | — |  |
| Create visual4 | blotato | Generate “Breaking News” infographic | If (true) | Get visual4 | ## Template : Breaking News |
| Get visual4 | blotato | Poll/get infographic render | Create visual4, Wait12 | If Else4 | ## Template : Breaking News |
| If Else4 | if | Check infographic render completion | Get visual4 | Get image/video2, Wait12 | ## Template : Breaking News |
| Wait12 | wait | Delay before re-polling infographic render | If Else4 (false) | Get visual4 | ## Template : Breaking News |
| Get image/video2 | httpRequest | HTTP GET first image | If Else4 (true) | Create Linkedin3 | ## Template : Breaking News |
| Create Linkedin3 | blotato | Publish to LinkedIn | Get image/video2 | Send notification : Published3 | ## Template : Breaking News |
| Send notification : Published3 | telegram | Notify “Published” (LinkedIn) | Create Linkedin3 | — |  |
| Create visual5 | blotato | Generate monocolor carousel | If (true) | Get visual5 | ## Template : Tutorial Carousel with Monocolor Background |
| Get visual5 | blotato | Poll/get monocolor carousel render | Create visual5, Wait15 | If Else5 | ## Template : Tutorial Carousel with Monocolor Background |
| If Else5 | if | Check monocolor carousel completion | Get visual5 | Get image/video3, Wait15 | ## Template : Tutorial Carousel with Monocolor Background |
| Wait15 | wait | Delay before re-polling monocolor carousel | If Else5 (false) | Get visual5 | ## Template : Tutorial Carousel with Monocolor Background |
| Get image/video3 | httpRequest | HTTP GET first image | If Else5 (true) | Create Twitter | ## Template : Tutorial Carousel with Monocolor Background |
| Create Twitter | blotato | Publish to Twitter/X | Get image/video3 | Send notification : Published4 | ## Template : Tutorial Carousel with Monocolor Background |
| Send notification : Published4 | telegram | Notify “Published” (Twitter/X) | Create Twitter | — |  |
| Sticky Note | stickyNote | Comment container | — | — | ## Template : AI Video with AI Voice |
| Sticky Note2 | stickyNote | Comment container | — | — | ## Template : Tweet Card Carousel with Minimal Style |
| Sticky Note3 | stickyNote | Comment container | — | — | ## Template : Image Slideshow with Prominent Text |
| Sticky Note4 | stickyNote | Comment container | — | — | ## Template : Breaking News |
| Sticky Note5 | stickyNote | Comment container | — | — | ## Template : Tutorial Carousel with Monocolor Background |
| Sticky Note6 | stickyNote | Comment container | — | — | # 🚀 AI Social Media Automation for Multiple Platforms using Blotato… Notion link: https://automatisation.notion.site/AI-Social-Media-Automation-for-Multiple-Platforms-using-Blotato-30a3d6550fd98051adc4eaa6e676478c?source=copy_link |
| Sticky Note1 | stickyNote | Comment container | — | — | ## Step 1 – Retrieve the video transcription |
| Sticky Note7 | stickyNote | Comment container | — | — | ## Step 2 – Create an optimized prompt using Blotato AI Research |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Add **Telegram API** credential in n8n:
      - Create a bot via BotFather, paste token into n8n Telegram credential.
   2. Add **Blotato API** credential in n8n:
      - Generate API key in Blotato and configure the `blotatoApi` credential.

2) **Add Telegram Trigger**
   1. Create node: **Telegram Trigger**
   2. Updates: `message`
   3. Select your Telegram credential.
   4. This node provides:
      - `{{$json.message.text}}` (expected YouTube URL)
      - `{{$json.message.chat.id}}`

3) **Transcription job (Blotato)**
   1. Create node: **Blotato** → Resource `source` (Create)
   2. Name: `Create Transcription`
   3. Set `sourceUrl` to: `{{ $json.message.text }}`
   4. Add custom instructions: “create all transcription of this youtube video and return the title in the first line.”
   5. Connect: `Telegram Trigger` → `Create Transcription`

4) **Poll transcription until completed**
   1. Create node: **Blotato** → Resource `source`, Operation `get`
      - Name: `Get Transcription`
      - `sourceId`: `{{ $json.id }}`
   2. Create node: **If**
      - Name: `If1`
      - Condition: `{{ $json.status }}` equals **`completed`** (recommended; avoid `=completed`)
   3. Create node: **Wait**
      - Name: `Wait`
      - Amount: `20 seconds`
   4. Wire:
      - `Create Transcription` → `Get Transcription`
      - `Get Transcription` → `If1`
      - `If1 (false)` → `Wait` → `Get Transcription` (loop)
      - `If1 (true)` → next block

5) **Create Perplexity-based fact-check + bullet summary**
   1. Create node: **Blotato** → Resource `source`
      - Name: `Check with perplexity ai`
      - Source type: `perplexity-query`
      - Query:  
        `from this transcription, and Check all information and summarize in 5 detailed bullet points for an instagram carousel:\n the transcription : {{ $json.content }}`
   2. Connect: `If1 (true)` → `Check with perplexity ai`

6) **Poll summary until completed**
   1. Create node: **Blotato** → Resource `source`, Operation `get`
      - Name: `Get content`
      - `sourceId`: `{{ $json.id }}`
   2. Create node: **If**
      - Name: `If`
      - Condition: `{{ $json.status }}` equals **`completed`** (recommended)
   3. Create node: **Wait**
      - Name: `Wait1`
      - Amount: `20 seconds`
   4. Wire:
      - `Check with perplexity ai` → `Get content` → `If`
      - `If (false)` → `Wait1` → `Get content` (loop)

7) **Create 5 visual-generation branches (fan-out from `If (true)`)**
   - From `If (true)`, connect to **all** of the following “Create visual…” nodes in parallel.

   **Branch A — Instagram carousel (tweet-card template)**
   1. Blotato node: Resource `video` (create render), name `Create visual`
      - Prompt: `create instagram carousel with this detailed bullet points : {{ $json.content }}`
      - Template: “Twitter/X style quote cards with minimal style”
      - Inputs: theme/handle/authorName/profileImage/aspectRatio as desired
   2. Blotato node: Resource `video` Operation `get`, name `Get visual`
      - `videoId`: `{{ $('Create visual').item.json.item.id }}`
   3. If node `If Else`: `{{ $json.item.status }}` equals `done`
      - False → Wait node `Wait4` (15s) → `Get visual`
      - True → Telegram sendPhoto `Send a photo message`
         - file: `{{ $json.item.imageUrls[0] }}`
         - chatId: `{{ $('Telegram Trigger').item.json.message.chat.id }}`
   4. Telegram node `post for approval`: operation `sendAndWait`
      - message: “Here is the post for approval.”
      - chatId from trigger
   5. Blotato publish node `Create Instagram`
      - accountId: your Instagram-connected Blotato account
      - postContentText: your caption
      - postContentMediaUrls: `{{ $('If Else').item.json.item.imageUrls }}`
   6. Telegram sendMessage `Send a text message`: “Published”

   **Branch B — TikTok AI video with voice**
   1. Blotato render `Create visual1` using template “AI Video with AI Voice”
   2. Poll with `Get visual1` and `If Else1` (`item.status == done`) + `Wait5` (15s)
   3. (Optional) HTTP Request `Get image/video1` GET `{{ $('Get visual1').item.json.item.mediaUrl }}`
   4. Blotato publish `Create Tiktok`
      - platform: `tiktok`
      - media URL: **use** `{{ $('Get visual1').item.json.item.mediaUrl }}` (recommended)
   5. Telegram notify `Send notification : Published`

   **Branch C — Facebook slideshow**
   1. Blotato render `Create visual3` template “Image Slideshow with Prominent Text”
   2. Poll: `Get visual3` → `If Else3` + `Wait9`
   3. Publish: `Create Facebook2`
      - platform: facebook
      - accountId and facebookPageId from your Blotato connections
      - media: `{{ $('Get visual3').item.json.item.imageUrls }}`
   4. Telegram notify `Send notification : Published2`

   **Branch D — LinkedIn breaking news**
   1. Blotato render `Create visual4` template “Breaking News”
   2. Poll: `Get visual4` → `If Else4` + `Wait12`
   3. (Optional) HTTP Request `Get image/video2`
   4. Publish: `Create Linkedin3`
      - platform: linkedin
      - media: `{{ $('Get visual4').item.json.item.imageUrls[0] }}`
   5. Telegram notify `Send notification : Published3`

   **Branch E — Twitter/X monocolor carousel**
   1. Blotato render `Create visual5` template “Tutorial Carousel with Monocolor Background”
   2. Poll: `Get visual5` → `If Else5` + `Wait15`
   3. (Optional) HTTP Request `Get image/video3`
   4. Publish: `Create Twitter`
      - platform: twitter
      - media: **use** `{{ $('Get visual5').item.json.item.imageUrls }}` (recommended)
   5. Telegram notify `Send notification : Published4`

8) **Activate workflow**
   - Ensure Telegram Trigger webhook is registered (n8n will do this on activation).
   - Test by sending a YouTube URL to the bot.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Social Media Automation for Multiple Platforms using Blotato (By Dr. Firas)” + YouTube reference `@[youtube](HaYFevp7KsU)` | Sticky note branding/info block |
| Full documentation link (Notion): https://automatisation.notion.site/AI-Social-Media-Automation-for-Multiple-Platforms-using-Blotato-30a3d6550fd98051adc4eaa6e676478c?source=copy_link | Setup, API config, platform connection guides, customization tips |
| Blotato referral link: https://blotato.com/?ref=firas | Mentioned as required account and for connecting social platforms |
| Workflow description in note: Trigger → Transcription → Fact-check summary → Template routing → Visual generation → Approval → Multi-platform publishing | High-level behavior statement from sticky note |

---

**Content policy disclaimer (provided by user):**  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.