Generate and publish Instagram Reels with Gemini and Veo

https://n8nworkflows.xyz/workflows/generate-and-publish-instagram-reels-with-gemini-and-veo-13149


# Generate and publish Instagram Reels with Gemini and Veo

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate and publish Instagram Reels with Gemini and Veo

**Purpose:**  
This workflow automatically generates a short vertical video (Reel) from an AI-written prompt, then publishes it to an Instagram Business account on a schedule.

**Typical use cases:**
- Daily automated content posting (motivation, tips, quotes, micro-stories)
- Consistent brand posting cadence without manual editing
- Rapid testing of topics/captions with AI-generated video concepts

### 1.1 Scheduling & Initialization
Runs every day at a configured time, then sets the topic/caption/aspect ratio used downstream.

### 1.2 Prompt Engineering (Gemini)
Uses Google Gemini to transform a high-level topic into a detailed, visually-oriented prompt suitable for a video generation model.

### 1.3 Video Generation (Veo via Gemini node)
Calls Google’s video generation resource (Veo) using the prompt and returns a generated video URL.

### 1.4 Instagram Publishing
Publishes the generated video as an Instagram Reel via Instagram Graph API (through a community node).

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Workflow Configuration
**Overview:**  
Triggers the workflow daily and defines the content parameters (topic, caption, aspect ratio) used by AI and publishing nodes.

**Nodes involved:**
- Schedule Reel Posting
- Workflow Configuration

#### Node: Schedule Reel Posting
- **Type / role:** `Schedule Trigger` (`n8n-nodes-base.scheduleTrigger`) — entry point; time-based execution.
- **Configuration (interpreted):**
  - Runs on an interval rule configured to trigger **daily at 09:00** (`triggerAtHour: 9`). (Minutes/timezone depend on n8n instance settings.)
- **Inputs / outputs:**
  - **Input:** none (trigger node)
  - **Output:** to **Workflow Configuration**
- **Version notes:** typeVersion **1.3**
- **Edge cases / failures:**
  - Timezone mismatch (server timezone vs expected locale).
  - Missed executions if n8n is down at the scheduled time (depends on n8n scheduling behavior and hosting).

#### Node: Workflow Configuration
- **Type / role:** `Set` (`n8n-nodes-base.set`) — defines constants for downstream use.
- **Configuration (interpreted):** sets three fields:
  - `videoTopic` = `"Create an engaging short video about daily motivation"`
  - `caption` = `"Daily motivation to inspire your day! #motivation #inspiration #reels"`
  - `aspectRatio` = `"9:16"`
- **Key expressions / variables used:**
  - Downstream references use `$('Workflow Configuration').item.json.<field>`
- **Inputs / outputs:**
  - **Input:** from **Schedule Reel Posting**
  - **Output:** to **Generate Video Prompt**
- **Version notes:** typeVersion **3.4**
- **Edge cases / failures:**
  - If later nodes expect different aspect ratio formats (e.g., `9:16` vs `portrait`), the video generation call may fail.
  - Caption length/hashtag policies are enforced by Instagram; overly long captions may fail publishing.

---

### Block 2 — Prompt Engineering (Gemini)
**Overview:**  
Generates a detailed, under-200-word visual prompt for a vertical short-form video based on `videoTopic`.

**Nodes involved:**
- Generate Video Prompt

#### Node: Generate Video Prompt
- **Type / role:** Google Gemini via LangChain node (`@n8n/n8n-nodes-langchain.googleGemini`) — text generation.
- **Configuration (interpreted):**
  - Uses a **Gemini chat/messages** style request with a single user message.
  - The message is dynamically built from `videoTopic`.
  - Model selection is present but **not set in the JSON** (`modelId.value` is empty). You must select a model (sticky note recommends `gemini-2.0-flash` for prompt generation).
- **Key expressions / variables used:**
  - Message content expression:
    - `{{ 'Generate a detailed video prompt ... about: ' + $json.videoTopic + ' ...' }}`
  - Relies on `videoTopic` produced by **Workflow Configuration**.
- **Inputs / outputs:**
  - **Input:** from **Workflow Configuration**
  - **Output:** to **Generate Video with Veo**
  - Output includes `message.content` (used downstream as the video prompt).
- **Version notes:** typeVersion **1.1**
- **Edge cases / failures:**
  - Missing/invalid Google credentials or model selection → authentication/model errors.
  - Quota/rate limit errors from Google.
  - If the model returns unexpected structure (rare), `{{$json.message.content}}` downstream could be empty or undefined.
  - Prompt may exceed the requested constraints unless the model follows instructions; consider adding validation if needed.

---

### Block 3 — Video Generation (Veo)
**Overview:**  
Generates the actual video using the prompt produced by Gemini, requesting a vertical aspect ratio, and returns a URL.

**Nodes involved:**
- Generate Video with Veo

#### Node: Generate Video with Veo
- **Type / role:** Google Gemini via LangChain node (`@n8n/n8n-nodes-langchain.googleGemini`) — **video generation** using `resource: "video"` (Veo).
- **Configuration (interpreted):**
  - **Prompt:** taken directly from the previous node’s output:
    - `prompt = {{$json.message.content}}`
  - **Aspect ratio option:** taken from Workflow Configuration:
    - `options.aspectRatio = {{ $('Workflow Configuration').item.json.aspectRatio }}`
  - **Return mode:** `returnAs: "url"` — node outputs a direct URL to the generated video.
  - Model selection is present but **not set** (`modelId.value` empty). Must choose the appropriate Veo-capable model.
- **Key expressions / variables used:**
  - Prompt: `{{ $json.message.content }}`
  - Aspect ratio: `{{ $('Workflow Configuration').item.json.aspectRatio }}`
- **Inputs / outputs:**
  - **Input:** from **Generate Video Prompt**
  - **Output:** to **Publish to Instagram**
  - Expected output field: `url` (used as `videoUrl` for Instagram)
- **Version notes:** typeVersion **1.1**
- **Edge cases / failures:**
  - Veo generation can take minutes; may exceed n8n execution timeout if not configured.
  - Quota, content policy rejections, transient generation errors.
  - Returned URL may expire; publishing should happen promptly after generation.
  - If `url` is missing, Instagram publish will fail due to empty video URL.

---

### Block 4 — Instagram Reel Publishing
**Overview:**  
Publishes the generated video to an Instagram Business account as a Reel with the configured caption.

**Nodes involved:**
- Publish to Instagram

#### Node: Publish to Instagram
- **Type / role:** Instagram community node (`@mookielianhd/n8n-nodes-instagram.instagram`) — publishes media via Instagram Graph API.
- **Configuration (interpreted):**
  - **Operation:** `publish`
  - **Resource:** `reels`
  - **Graph API Version:** `v22.0`
  - **Instagram Business Account ID:** currently a placeholder:
    - `node = "<__PLACEHOLDER_VALUE__Instagram Business Account ID__>"`
  - **Caption:** from Workflow Configuration:
    - `caption = {{ $('Workflow Configuration').item.json.caption }}`
  - **Video URL:** from Veo node output:
    - `videoUrl = {{ $('Generate Video with Veo').item.json.url }}`
- **Inputs / outputs:**
  - **Input:** from **Generate Video with Veo**
  - **Output:** none configured beyond completion (publishing result returned by node)
- **Version notes:** typeVersion **1**
- **Edge cases / failures:**
  - Missing/expired Facebook Graph access token; insufficient permissions.
  - Wrong Instagram Business Account ID or not linked to a Facebook Page.
  - Instagram publishing limits or rate limits.
  - Video URL inaccessible to Meta (must be publicly reachable and downloadable by Graph API).
  - Graph API changes/versioning issues; node pinned to `v22.0`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Reel Posting | n8n-nodes-base.scheduleTrigger | Daily time-based trigger | — | Workflow Configuration | ## 🎬 Automated Instagram Reel Generator  / ### 📖 HOW IT WORKS / **Flow Overview:** / 1. **Schedule Trigger** - Runs daily at 9 AM (customizable) / 2. **Workflow Configuration** - Defines video topic, caption, and aspect ratio / 3. **Generate Video Prompt** - Gemini AI creates detailed visual prompt for video generation / 4. **Generate Video with Veo** - Google Veo generates the actual video from prompt / 5. **Publish to Instagram** - Posts the reel directly to your Instagram account / **Processing Time:** 2-5 minutes per video (Veo generation) / --- / ### ⚙️ SETUP GUIDE / **Step 1: Google Gemini Credentials** / - Get API key: https://ai.google.dev/ / - Add credentials to both "Generate Video Prompt" and "Generate Video with Veo" nodes / - Select model: gemini-2.0-flash recommended for prompt generation / - Select Veo model for video generation / **Step 2: Instagram Credentials** / - Requires Facebook Graph API access token / - Needed permissions: instagram_content_publish, pages_read_engagement / - Get started: https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/get-started / - Add credentials to "Publish to Instagram" node / **Step 3: Configure Content** / - Edit "Workflow Configuration" node / - Set **videoTopic**: Your content theme (e.g., "daily motivation", "tech tips") / - Set **caption**: Instagram caption with hashtags / - Set **aspectRatio**: 9:16 for Reels (default) / **Step 4: Adjust Schedule** / - Edit "Schedule Reel Posting" trigger / - Change time/frequency as needed / - Consider API rate limits (don't post too frequently) / **Step 5: Test Before Activating** / - Click "Test workflow" to run manually / - Verify credentials work / - Check video quality and Instagram post / - Activate workflow when ready / --- / ### ⚠️ IMPORTANT NOTES / - Check Google Gemini API quotas and pricing / - Instagram has daily posting limits / - Ensure workflow timeout > 5 minutes for video generation / - Test thoroughly before scheduling / Docs: https://docs.n8n.io/ |
| Workflow Configuration | n8n-nodes-base.set | Define topic/caption/aspect ratio variables | Schedule Reel Posting | Generate Video Prompt | (same sticky note as above) |
| Generate Video Prompt | @n8n/n8n-nodes-langchain.googleGemini | Generate detailed visual prompt from topic | Workflow Configuration | Generate Video with Veo | (same sticky note as above) |
| Generate Video with Veo | @n8n/n8n-nodes-langchain.googleGemini | Generate video and return URL | Generate Video Prompt | Publish to Instagram | (same sticky note as above) |
| Publish to Instagram | @mookielianhd/n8n-nodes-instagram.instagram | Publish Reel to Instagram Business account | Generate Video with Veo | — | (same sticky note as above) |
| 📋 Setup Guide & How It Works | n8n-nodes-base.stickyNote | Embedded documentation and setup notes | — | — | ## 🎬 Automated Instagram Reel Generator (content includes links above) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n and name it:  
   **“Generate and publish Instagram Reels with Gemini and Veo”**

2) **Add Trigger Node**
   - Add node: **Schedule Trigger**
   - Configure it to run **daily at 09:00**
     - Set the interval rule to trigger at hour **9**
   - This is the workflow entry point.

3) **Add Configuration Node**
   - Add node: **Set** and name it **“Workflow Configuration”**
   - Add fields:
     - `videoTopic` (String): e.g. “Create an engaging short video about daily motivation”
     - `caption` (String): e.g. “Daily motivation to inspire your day! #motivation #inspiration #reels”
     - `aspectRatio` (String): `9:16`
   - Connect: **Schedule Trigger → Workflow Configuration**

4) **Add Gemini Prompt Generation Node**
   - Add node: **Google Gemini (LangChain)** and name it **“Generate Video Prompt”**
   - Credentials:
     - Create/configure **Google AI (Gemini) API key** credentials (from https://ai.google.dev/)
   - Model:
     - Select a text-capable Gemini model (note suggests **gemini-2.0-flash**).
   - Messages:
     - Add one message with content (expression):
       - `Generate a detailed video prompt for a short-form vertical video (9:16 aspect ratio) about: {{$json.videoTopic}}. The prompt should describe visual scenes, camera movements, and mood in detail for video generation AI. Keep it under 200 words and focus on visual storytelling.`
   - Connect: **Workflow Configuration → Generate Video Prompt**

5) **Add Veo Video Generation Node**
   - Add node: **Google Gemini (LangChain)** and name it **“Generate Video with Veo”**
   - Credentials:
     - Reuse the same Google Gemini credentials (or a separate one if required by your setup).
   - Set it to use **video generation**:
     - Resource: **video**
   - Prompt:
     - Use expression: `{{$json.message.content}}`
   - Options:
     - Aspect ratio: expression `{{$('Workflow Configuration').item.json.aspectRatio}}`
   - Return:
     - Return as: **url**
   - Model:
     - Select a **Veo-capable model** (must be chosen; the JSON has it unset).
   - Connect: **Generate Video Prompt → Generate Video with Veo**

6) **Add Instagram Publishing Node**
   - Install/enable the community node if needed: `@mookielianhd/n8n-nodes-instagram`
   - Add node: **Instagram** and name it **“Publish to Instagram”**
   - Credentials:
     - Configure Facebook/Instagram Graph API access token credentials with required permissions:
       - `instagram_content_publish`
       - `pages_read_engagement`
     - Ensure the Instagram account is a **Business** account linked to the correct Facebook Page.
   - Parameters:
     - Resource: **reels**
     - Operation: **publish**
     - Graph API version: **v22.0**
     - Instagram Business Account ID: set your real ID (replace placeholder)
     - Caption: `{{$('Workflow Configuration').item.json.caption}}`
     - Video URL: `{{$('Generate Video with Veo').item.json.url}}`
   - Connect: **Generate Video with Veo → Publish to Instagram**

7) **Execution settings (recommended)**
   - Increase workflow/execution timeout (Veo can take **2–5 minutes**).
   - Consider adding error handling (optional but recommended):
     - Retry logic for Veo generation failures
     - Checks that `url` exists before publishing
     - Notifications (email/Slack) on failure

8) **Test then activate**
   - Use **Test workflow** to confirm:
     - Gemini returns `message.content`
     - Veo returns `url`
     - Instagram publishes successfully
   - Activate the workflow after verification.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Gemini API key setup | https://ai.google.dev/ |
| Instagram Graph API get started (token/permissions/account linking) | https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/get-started |
| n8n documentation | https://docs.n8n.io/ |
| Operational considerations | Veo generation time 2–5 minutes; check Google quotas/pricing; Instagram posting limits; ensure timeout > 5 minutes; test thoroughly before scheduling. |