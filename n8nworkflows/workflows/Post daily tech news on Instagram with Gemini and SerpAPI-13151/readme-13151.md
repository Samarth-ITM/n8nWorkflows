Post daily tech news on Instagram with Gemini and SerpAPI

https://n8nworkflows.xyz/workflows/post-daily-tech-news-on-instagram-with-gemini-and-serpapi-13151


# Post daily tech news on Instagram with Gemini and SerpAPI

## 1. Workflow Overview

**Purpose:** Automatically create and publish a daily Instagram image post about trending technology news. The workflow searches the web for fresh tech news (via SerpAPI), uses an AI agent (Gemini) to select a story and generate both an **image prompt** and an **Instagram caption**, generates the image with **Google Gemini Imagen**, uploads the image to get a public URL, then publishes the post to Instagram.

**Primary use cases:**
- Daily “tech news” content automation for Instagram
- Scalable social content generation using AI + search grounding
- Hands-free scheduling with consistent brand voice

### 1.1 Scheduling & Execution Start
Runs once per day at a fixed hour.

### 1.2 Workflow Configuration (Topic + Audience)
Defines the search query and content intent used downstream by the AI agent.

### 1.3 News Research + Structured Content Generation (AI Agent)
Searches via SerpAPI, selects a news item, and returns **strict JSON** containing:
- `imagePrompt`
- `caption`
- `newsSource`

### 1.4 Image Generation (Gemini Imagen)
Creates an image from the generated prompt.

### 1.5 Temporary Hosting + Instagram Publishing
Uploads the generated image to a temporary file host (to obtain a URL) and publishes it to Instagram with the AI caption.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduling & Trigger
**Overview:** Starts the workflow automatically every day at 9 AM.

**Nodes involved:**
- Schedule Daily Posts

#### Node: Schedule Daily Posts
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based trigger (entry point).
- **Configuration:** Runs **daily at 09:00** (server/workflow timezone).
- **Key values:** `triggerAtHour: 9`
- **Connections:**
  - **Output:** to **Workflow Configuration**
- **Potential failures / edge cases:**
  - Timezone mismatch (n8n instance timezone vs desired local timezone).
  - If the instance is down at 9:00, the run may be missed (depends on n8n scheduling behavior and hosting).
- **Version notes:** Uses typeVersion `1.3` (no special constraints, but UI differs across n8n versions).

---

### Block 2 — Workflow Configuration (Inputs)
**Overview:** Sets constants used by the AI agent: the news query, content type, and audience.

**Nodes involved:**
- Workflow Configuration

#### Node: Workflow Configuration
- **Type / role:** `n8n-nodes-base.set` — creates/overrides fields for downstream steps.
- **Configuration choices (interpreted):**
  - Creates three string fields:
    - `newsQuery`: `"latest trending technology news"`
    - `contentType`: `"Instagram Image Post"`
    - `targetAudience`: `"tech enthusiasts and general audience"`
- **Key expressions/variables:** None inside this node; it defines values referenced later.
- **Connections:**
  - **Input:** from **Schedule Daily Posts**
  - **Output:** to **AI Agent - News Research & Prompt Generation**
- **Potential failures / edge cases:**
  - Mis-typed field names break downstream expressions (e.g., `newsQuery` must match exactly).
- **Version notes:** typeVersion `3.4` (Set node UI/options vary by version).

---

### Block 3 — AI News Research + Prompt/Citation Generation
**Overview:** Uses a LangChain AI Agent backed by **Google Gemini Chat** and a **SerpAPI tool** to research news, choose a story, and generate structured JSON (image prompt + caption + source).

**Nodes involved:**
- AI Agent - News Research & Prompt Generation
- Google Gemini Chat Model
- SerpAPI News Search Tool
- Structured Output Parser

#### Node: Google Gemini Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` — provides the LLM for the agent and parser.
- **Configuration choices:**
  - Default options (no explicit model/version shown in parameters).
- **Connections:**
  - Provides **AI language model** connection to:
    - **AI Agent - News Research & Prompt Generation**
    - **Structured Output Parser**
- **Potential failures / edge cases:**
  - Missing/invalid Google Gemini credentials.
  - Model availability/region restrictions.
  - Rate limiting / quota exhaustion.
- **Version notes:** typeVersion `1`.

#### Node: SerpAPI News Search Tool
- **Type / role:** `@n8n/n8n-nodes-langchain.toolSerpApi` — tool the agent calls to perform web/news search grounding.
- **Configuration choices:**
  - Default options (no custom query here; the agent decides how to call it).
- **Connections:**
  - Connected via **ai_tool** to **AI Agent - News Research & Prompt Generation**
- **Potential failures / edge cases:**
  - Missing/invalid SerpAPI key.
  - SerpAPI query limits / throttling.
  - Low-quality or irrelevant results depending on query wording.
- **Version notes:** typeVersion `1`.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema output from the agent.
- **Configuration choices:**
  - **Schema type:** manual JSON schema requiring:
    - `imagePrompt` (string)
    - `caption` (string)
    - `newsSource` (string)
  - **autoFix: true** — attempts to repair malformed JSON or mismatched outputs.
- **Key schema requirement:** All three fields are **required**.
- **Connections:**
  - Receives **AI language model** from **Google Gemini Chat Model**
  - Provides **ai_outputParser** to **AI Agent - News Research & Prompt Generation**
- **Potential failures / edge cases:**
  - If the model returns content that cannot be repaired into valid schema, the node/agent run fails.
  - If the agent returns non-JSON or missing required fields repeatedly, auto-fix may not succeed.
- **Version notes:** typeVersion `1.3`.

#### Node: AI Agent - News Research & Prompt Generation
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates tool use (SerpAPI) + LLM reasoning to produce final structured output.
- **Configuration choices (interpreted):**
  - **Prompt text (user instruction):**
    - Searches for news about `{{$('Workflow Configuration').item.json.newsQuery}}`
    - Requests creation of image content and caption for the chosen story
  - **System message:** Instructs agent to:
    1) use SerpAPI to search latest trending news  
    2) select the most engaging story  
    3) generate a detailed image prompt  
    4) generate caption + hashtags  
    - Return **exact JSON** with `imagePrompt`, `caption`, `newsSource`
  - **Output parsing:** `hasOutputParser: true` (parser wired in)
- **Key expressions/variables used:**
  - `{{ $('Workflow Configuration').item.json.newsQuery }}`
- **Connections:**
  - **Inputs:**
    - Main input from **Workflow Configuration**
    - AI Language Model from **Google Gemini Chat Model**
    - AI Tool from **SerpAPI News Search Tool**
    - AI Output Parser from **Structured Output Parser**
  - **Output (main):** to **Generate an image**
- **Output shape (important):**
  - The workflow later references:  
    `$('AI Agent - News Research & Prompt Generation').item.json.output.caption`  
    `$('AI Agent - News Research & Prompt Generation').item.json.output.imagePrompt`  
    This implies the agent node outputs an `output` object containing parsed fields.
- **Potential failures / edge cases:**
  - SerpAPI returns no results; agent may hallucinate or fail schema compliance.
  - Breaking news with paywalled/blocked sources—agent may produce weak `newsSource`.
  - Parser schema mismatch (e.g., agent returns `source` instead of `newsSource`).
  - Prompt injection from search snippets (mitigated somewhat by system instructions, but still a risk).
- **Version notes:** typeVersion `3.1` (agent behavior/options vary by node version).

---

### Block 4 — Image Generation (Gemini Imagen)
**Overview:** Generates the Instagram post image using Gemini’s image generation model based on the AI-produced prompt.

**Nodes involved:**
- Generate an image

#### Node: Generate an image
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — generates an image from a text prompt.
- **Configuration choices:**
  - **Resource:** `image`
  - **Model:** `imagen-3.0-generate-001`
  - **Prompt expression:** uses agent output:  
    `={{ $('AI Agent - News Research & Prompt Generation').item.json.output.imagePrompt }}`
- **Connections:**
  - **Input:** from **AI Agent - News Research & Prompt Generation**
  - **Output:** to **Upload file to tmpfiles**
- **Potential failures / edge cases:**
  - Safety policy blocks some prompts (returns error or empty output).
  - Quota/rate limits for image generation.
  - Output format mismatch (binary vs JSON) depending on node behavior/version.
- **Version notes:** typeVersion `1.1`.

---

### Block 5 — Temporary Hosting + Publish to Instagram
**Overview:** Uploads the generated image to a temporary host to obtain a public URL, then publishes it to Instagram using a community Instagram node.

**Nodes involved:**
- Upload file to tmpfiles
- Post to Instagram

#### Node: Upload file to tmpfiles
- **Type / role:** `n8n-nodes-tmpfiles.tmpfiles` — uploads a file and returns a temporary public URL.
- **Configuration choices:** Default (no parameters set).
- **Connections:**
  - **Input:** from **Generate an image**
  - **Output:** to **Post to Instagram**
- **Expected output fields:**
  - Workflow later uses `{{$json.url}}` in the Instagram node, so this node must output `url`.
- **Potential failures / edge cases:**
  - If the incoming image is not provided as expected (binary vs buffer mismatch), upload fails.
  - Temporary URL expiration: Instagram publishing must occur before the URL expires.
  - tmpfiles service downtime or rate limiting.
- **Version notes:** typeVersion `1`.

#### Node: Post to Instagram
- **Type / role:** `@mookielianhd/n8n-nodes-instagram.instagram` — publishes an image post to Instagram.
- **Configuration choices (interpreted):**
  - **Resource:** `image`
  - **Target node/user:** `"me"` (publishes to authenticated account)
  - **Caption expression:**  
    `={{ $('AI Agent - News Research & Prompt Generation').item.json.output.caption }}`
  - **Image URL expression:**  
    `={{ $json.url }}` (expects URL from tmpfiles node output)
- **Connections:**
  - **Input:** from **Upload file to tmpfiles**
  - **Output:** none (end of workflow)
- **Potential failures / edge cases:**
  - Credential/auth issues (token expiry, missing permissions).
  - Instagram API constraints (account type requirements, media publishing restrictions, rate limits).
  - Image URL not accessible publicly by Instagram (must be reachable by Instagram servers).
  - Caption length/hashtag policies (may truncate or fail depending on API/node behavior).
- **Version notes:** typeVersion `1` (community node; behavior depends on installed package version).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Daily Posts | n8n-nodes-base.scheduleTrigger | Daily trigger at 9 AM | — | Workflow Configuration | ## Schedule/Timer |
| Workflow Configuration | n8n-nodes-base.set | Defines query/audience constants | Schedule Daily Posts | AI Agent - News Research & Prompt Generation | ## Search Query & Target Audience |
| AI Agent - News Research & Prompt Generation | @n8n/n8n-nodes-langchain.agent | Researches news + generates structured JSON for image/caption | Workflow Configuration; (AI) Google Gemini Chat Model; (Tool) SerpAPI News Search Tool; (Parser) Structured Output Parser | Generate an image | # Scrape news  and generate a prompt |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backing the agent and parser | — | (AI) AI Agent - News Research & Prompt Generation; (AI) Structured Output Parser | # Scrape news  and generate a prompt |
| SerpAPI News Search Tool | @n8n/n8n-nodes-langchain.toolSerpApi | Web/news search tool for grounding | — | (AI Tool) AI Agent - News Research & Prompt Generation | # Scrape news  and generate a prompt |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema (imagePrompt, caption, newsSource) | (AI) Google Gemini Chat Model | (AI Parser) AI Agent - News Research & Prompt Generation | # Scrape news  and generate a prompt |
| Generate an image | @n8n/n8n-nodes-langchain.googleGemini | Generates Instagram image via Imagen | AI Agent - News Research & Prompt Generation | Upload file to tmpfiles | ## Generate post image |
| Upload file to tmpfiles | n8n-nodes-tmpfiles.tmpfiles | Uploads image and returns temporary public URL | Generate an image | Post to Instagram | ## Get a temporary URL |
| Post to Instagram | @mookielianhd/n8n-nodes-instagram.instagram | Publishes image post with caption | Upload file to tmpfiles | — | ## Final step - Publish |
| 📖 Setup Guide | n8n-nodes-base.stickyNote | Canvas documentation | — | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas label | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas label | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas label | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas label | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas label | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas label | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Schedule Trigger**
   2. Set rule to **run daily at 09:00**.
   3. This is your workflow entry point.

2) **Add configuration constants**
   1. Add node: **Set** (name it **Workflow Configuration**)
   2. Add fields (strings):
      - `newsQuery` = `latest trending technology news`
      - `contentType` = `Instagram Image Post`
      - `targetAudience` = `tech enthusiasts and general audience`
   3. Connect: **Schedule Trigger → Workflow Configuration**

3) **Add the AI Agent**
   1. Add node: **AI Agent** (LangChain) and name it **AI Agent - News Research & Prompt Generation**
   2. Set the agent prompt text to:
      - `Search for news about: {{ $('Workflow Configuration').item.json.newsQuery }}`
      - `Create image content and caption for this news story.`
   3. In **System Message**, paste instructions requiring tool usage + exact JSON output with:
      - `imagePrompt`, `caption`, `newsSource`
   4. Ensure the agent is configured to use an **Output Parser** (structured).

4) **Attach Gemini Chat model to the agent**
   1. Add node: **Google Gemini Chat Model**
   2. Configure **Google Gemini credentials** (API key / OAuth depending on your n8n setup).
   3. Connect the model via **AI Language Model** connection to:
      - **AI Agent - News Research & Prompt Generation**
      - **Structured Output Parser** (next step)

5) **Add Structured Output Parser**
   1. Add node: **Structured Output Parser**
   2. Set schema type to **Manual**
   3. Paste JSON schema requiring:
      - `imagePrompt` (string), `caption` (string), `newsSource` (string), all required
   4. Enable **Auto-fix**.
   5. Connect:
      - **Google Gemini Chat Model → Structured Output Parser** (AI language model connection)
      - **Structured Output Parser → AI Agent** (AI output parser connection)

6) **Add SerpAPI tool**
   1. Add node: **SerpAPI** tool (LangChain tool node)
   2. Configure **SerpAPI credentials** (API key).
   3. Connect **SerpAPI Tool → AI Agent** using the **AI Tool** connection.

7) **Generate the image (Gemini Imagen)**
   1. Add node: **Google Gemini** (image generation) and name it **Generate an image**
   2. Set **Resource** to **Image**
   3. Select model **imagen-3.0-generate-001**
   4. Set prompt to:
      - `{{ $('AI Agent - News Research & Prompt Generation').item.json.output.imagePrompt }}`
   5. Connect: **AI Agent → Generate an image**

8) **Upload image to temporary hosting**
   1. Add node: **tmpfiles** (Upload file to tmpfiles)
   2. Keep defaults
   3. Connect: **Generate an image → Upload file to tmpfiles**
   4. Confirm this node outputs a field named **`url`** (required downstream).

9) **Post to Instagram**
   1. Install and enable the community node package (if not already): `@mookielianhd/n8n-nodes-instagram`
   2. Add node: **Instagram** (name it **Post to Instagram**)
   3. Configure **Instagram credentials** as required by that node (token/auth flow).
   4. Set:
      - Resource: **image**
      - Node/User: **me**
      - Image URL: `{{ $json.url }}`
      - Caption: `{{ $('AI Agent - News Research & Prompt Generation').item.json.output.caption }}`
   5. Connect: **Upload file to tmpfiles → Post to Instagram**

10) **Test and enable**
   1. Execute manually once to verify:
      - SerpAPI returns results
      - Agent returns valid structured JSON
      - Image generation returns a valid image
      - tmpfiles returns `url`
      - Instagram publishes successfully
   2. Enable the workflow to activate the daily schedule.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How it works: Schedule Trigger (9 AM) → Configuration → AI Agent (SerpAPI + content) → Image Generation (Gemini) → Upload & Post (Instagram)” | From sticky note “📖 Setup Guide” |
| Required Credentials: Google Gemini API (Agent & Image Generation), SerpAPI (News search), Instagram API (Publishing) | From sticky note “📖 Setup Guide” |
| Configuration is done in “Workflow Configuration” node (topics, content type, target audience). | From sticky note “📖 Setup Guide” |
| Testing steps: add credentials → run manually → enable schedule trigger | From sticky note “📖 Setup Guide” |