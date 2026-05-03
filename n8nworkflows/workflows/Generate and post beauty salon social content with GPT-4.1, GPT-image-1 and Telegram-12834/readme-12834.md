Generate and post beauty salon social content with GPT-4.1, GPT-image-1 and Telegram

https://n8nworkflows.xyz/workflows/generate-and-post-beauty-salon-social-content-with-gpt-4-1--gpt-image-1-and-telegram-12834


# Generate and post beauty salon social content with GPT-4.1, GPT-image-1 and Telegram

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate and post beauty salon social content with GPT-4.1, GPT-image-1 and Telegram

**Purpose:**  
This workflow automatically generates daily (or event-driven) beauty-salon social posts, converts each post into an image-generation prompt, generates a photorealistic image, then distributes the content to Telegram and optionally other channels (WordPress, Facebook, X, LinkedIn) and Google Drive.

**Typical use cases**
- Daily automated content for a beauty salon (Telegram-first).
- Turning incoming ideas (Google Sheets / Airtable / RSS / DB triggers) into ready-to-publish posts with matching visuals.
- Multi-channel distribution pipeline for marketing teams.

### 1.1 Content Source & Triggering
Multiple entry points decide *when* to generate a new post (schedule, manual run, RSS, Google Sheets row added, etc.).

### 1.2 AI Text Generation (Post)
A LangChain Agent (“GENERATE TEXT”) writes the final Telegram post using:
- a configured LLM (OpenAI GPT‑4.1)
- a web-search tool node (SearchApi tool) for up-to-date research
- short-term memory (Buffer Window)

### 1.3 AI Prompt Generation (Image Prompt)
A second LangChain Agent (“GENERATE PROMPT”) converts the post into a detailed image prompt (no text/logos), using a configured LLM (OpenAI GPT‑4.1).

### 1.4 Image Generation
Primary path uses OpenAI’s **gpt-image-1** via the OpenAI Image node.  
The workflow also contains example/optional HTTP Request nodes for other image providers (Replicate, Ideogram, Clipdrop, Runway, etc.), plus an OpenAI Images HTTP node + “Convert to Binary” utility for providers returning base64.

### 1.5 Publishing & Distribution
The generated image output is split and fanned out to:
- Telegram photo post (caption = generated text)
- WordPress post (placeholder config)
- Facebook Graph API post (placeholder config)
- X/Twitter post (placeholder config)
- LinkedIn post (requires Person ID)
- Google Drive upload (filename derived from an agent output field)

---

## 2. Block-by-Block Analysis

### Block 2.1 — Triggers (Entry Points)

**Overview:**  
Any one of several triggers can start the workflow and pass a “reference” (topic/link/feed item/etc.) into the post generator.

**Nodes involved:**
- When clicking Execute workflow (Manual Trigger)
- Schedule Trigger
- Google Sheets Trigger
- RSS Feed Trigger
- Meta Reference (Facebook Trigger)
- Airtable Trigger
- Postgres Trigger

#### Node: When clicking Execute workflow
- **Type / role:** Manual Trigger; dev/testing entry point.
- **Config:** No parameters.
- **Output:** Single empty item.
- **Connections:** → `GENERATE TEXT` (main)
- **Edge cases:** None (only runs when manually executed).

#### Node: Schedule Trigger
- **Type / role:** Schedule-based automation.
- **Config choices:** Triggers at **09:00** (interval rule with `triggerAtHour: 9`).
- **Connections:** → `GENERATE TEXT`
- **Edge cases:** Server timezone differences; ensure n8n timezone matches business needs.

#### Node: Google Sheets Trigger
- **Type / role:** Polling trigger for new rows in Google Sheets.
- **Config choices:**
  - Event: **rowAdded**
  - Watches column: **“Links for articles to refer”**
  - Range monitored: **A2:A10** (A1 notation)
  - Poll every **1 minute**
  - Sheet: `Sheet1` (gid=0)
  - DocumentId is placeholder (`YOUR_AWS_SECRET_KEY_HERE-y-c`) and must be replaced with a real spreadsheet.
- **Connections:** → `GENERATE TEXT`
- **Edge cases / failures:**
  - Google auth/permissions errors
  - Range too small (A2:A10 only catches first 9 additions)
  - Column name mismatch breaks detection
  - Polling quotas / rate limits

#### Node: RSS Feed Trigger
- **Type / role:** Poll RSS and trigger on new feed items.
- **Config choices:** Poll every minute.
- **Connections:** → `GENERATE TEXT`
- **Edge cases:** Invalid RSS URL (not configured in JSON), duplicate item detection behavior, feed downtime.

#### Node: Meta Reference (Facebook Trigger)
- **Type / role:** Webhook trigger for Meta/Facebook.
- **Config choices:** Webhook-based; options empty; needs proper webhook setup in Meta.
- **Connections:** → `GENERATE TEXT`
- **Edge cases:** Webhook verification issues, permissions/scopes, payload shape variance.

#### Node: Airtable Trigger
- **Type / role:** Polling trigger for Airtable changes.
- **Config choices:** baseId/tableId empty placeholders; pollTimes default.
- **Connections:** → `GENERATE TEXT`
- **Edge cases:** Missing base/table config, Airtable rate limits.

#### Node: Postgres Trigger
- **Type / role:** DB trigger on Postgres table changes.
- **Config choices:** schema = `public`, tableName empty placeholder.
- **Connections:** → `GENERATE TEXT`
- **Edge cases:** Missing table selection, DB connectivity, permissions, high-frequency events.

**Sticky notes relevant to this block:**
- “## 1. Choose Your Trigger …” (explains options and when to use each)
- “# Daily AI social content for beauty brands … Section 1 — Content sources & triggers”

---

### Block 2.2 — AI Text Generation (Telegram Post)

**Overview:**  
Generates a short Telegram-ready beauty-salon post (≤ 1024 chars) based on the incoming reference and web research, with a fixed address line appended.

**Nodes involved:**
- GENERATE TEXT (LangChain Agent)
- OPENAI WRITES POSTS (Chat Model)
- Simple Memory (Memory Buffer Window)
- Search google in SearchApi (Tool)

#### Node: OPENAI WRITES POSTS
- **Type / role:** `lmChatOpenAi` language model provider for the agent.
- **Config choices:** Model = **gpt-4.1**.
- **Credentials:** OpenAI API credential (named “n8n free OpenAI API credits”).
- **Connections:** Provides `ai_languageModel` → `GENERATE TEXT`.
- **Edge cases:** Invalid/expired OpenAI key, model availability, rate limiting.

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow`; short-term conversational memory for the agent.
- **Config choices:** Defaults (window size not specified in JSON).
- **Connections:** `ai_memory` → `GENERATE TEXT`.
- **Edge cases:** Memory can leak prior context into future posts if multiple runs share the same execution context (depends on n8n/agent behavior). Consider disabling memory for strict one-shot generation.

#### Node: Search google in SearchApi
- **Type / role:** SearchApi tool node exposed to the agent as a callable tool.
- **Config choices:**
  - Query comes from the agent: `{{$fromAI('Query', '', 'string')}}`
  - Other search settings left default.
- **Connections:** `ai_tool` → `GENERATE TEXT`.
- **Edge cases:** Missing SearchApi credentials/config (not shown in JSON), API limits, unexpected SERP formats.

#### Node: GENERATE TEXT
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates research + generation.
- **Config choices (interpreted):**
  - **User text prompt:**  
    Uses an incoming field `{{$json.name}}` as “main reference article link”.  
    It also instructs to “update facts to 2025–2026” and to “cross-check with Tavily internet research”.
  - **System message:** A detailed brand voice + constraints for Telegram posts, including:
    - Start with real-time research (mentions Tavily, but actual connected tool is SearchApi)
    - Beauty salon audience (women 20–45)
    - Hook + short paragraphs + mini-lists
    - Evidence-based, minimal emojis
    - 3–6 hashtags
    - Final address line: “Salon address: Example Street 10, City…”
  - **Prompt type:** “define” (explicit system prompt usage)
- **Inputs:**
  - Main item from a trigger (varies by trigger)
  - AI language model from `OPENAI WRITES POSTS`
  - AI tool from `Search google in SearchApi`
  - AI memory from `Simple Memory`
- **Outputs:** `json.output` (final post text) used downstream.
- **Connections:** → `GENERATE PROMPT`
- **Edge cases / failures:**
  - `{{$json.name}}` may not exist for many triggers (Manual, Schedule, etc.), causing weak prompts or empty reference. You may need a normalization step (Set node) per trigger.
  - Instruction says “Tavily tool” but no Tavily node is connected; the agent will instead use SearchApi (unless you add Tavily).
  - Telegram 1024-char constraint is only in prompt; no hard enforcement node.

**Sticky notes relevant:**
- “## 2. Connect Chat Models”
- “## 3. Customize Prompts”
- “# Detail prompts for posts and images up to your beauty salon”
- “# Choose Your Favourite Chat Model”
- “# Daily AI social content … Section 2 — AI post generation”

---

### Block 2.3 — AI Visual Prompt Generation

**Overview:**  
Turns the generated post into a single high-quality image prompt for photorealistic marketing visuals, explicitly banning text/logos in the image.

**Nodes involved:**
- GENERATE PROMPT (LangChain Agent)
- OPENAI WRITES PROMPTS (Chat Model)
- Database (Memory Buffer Window) *(connected only to this agent as memory; name is misleading)*

#### Node: OPENAI WRITES PROMPTS
- **Type / role:** `lmChatOpenAi` model provider.
- **Config choices:** Model = **gpt-4.1**.
- **Connections:** `ai_languageModel` → `GENERATE PROMPT`
- **Edge cases:** Same OpenAI credential/model issues as above.

#### Node: Database
- **Type / role:** `memoryBufferWindow` used as the agent’s memory.
- **Config choices:** Defaults.
- **Connections:** `ai_memory` → `GENERATE PROMPT`
- **Edge cases:** Same memory bleed considerations; also the node name “Database” can confuse maintainers.

#### Node: GENERATE PROMPT
- **Type / role:** LangChain Agent; generates an image prompt based on `{{$json.output}}` from the text agent.
- **Config choices (interpreted):**
  - User instruction: “Create a highly detailed, photorealistic visual concept…” from the Telegram post.
  - System message: Detailed conversion rules; output must be **one single prompt** in English, no quotes, no repetition of post text, no text/logos in scene.
- **Inputs:**
  - From `GENERATE TEXT`: `json.output` (the post)
  - From `OPENAI WRITES PROMPTS`: LLM
  - Memory from `Database`
- **Outputs:** `json.output` (the image prompt string).
- **Connections:** → `OPENAI GENERATES IMAGE`
- **Edge cases:**
  - The later Google Drive Upload expects `$('Image Prompt Agent').item.json.output.title` but there is no node named “Image Prompt Agent”, and `GENERATE PROMPT` outputs a string (no `title`). This will break the upload filename unless fixed.

**Sticky notes relevant:**
- “# Detail prompts for posts and images up to your beauty salon”
- “## 3. Customize Prompts”

---

### Block 2.4 — Image Generation (Primary + Optional Alternatives)

**Overview:**  
Generates an image using OpenAI’s image model and prepares it for downstream distribution. The workflow includes multiple alternative provider nodes as examples/templates.

**Nodes involved (primary path):**
- OPENAI GENERATES IMAGE
- Split Out1

**Nodes involved (optional/alternative templates present but not connected to main path):**
- Generate Image (HTTP Request to OpenAI Images API)
- Convert to Binary
- Replicate API
- Imagen Google API
- HuggingFace API
- Kling Images
- Runway Images
- Leonardo Images
- APITemplate.io
- Ideogram API
- Clipdrop API
- Nano Banana
- (Plus “Hugging Face Inference Model” which is an LLM node, but positioned among image options)

#### Node: OPENAI GENERATES IMAGE
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` with resource = **image**.
- **Config choices:**
  - Model: **gpt-image-1**
  - Prompt expression:  
    `IMPORTANT! DO NOT WRITE TEXT ON THE PICTURE! Create a perfect visual for:\n{{ $json.output }}`
  - Uses `json.output` from `GENERATE PROMPT`.
- **Credentials:** OpenAI API credential.
- **Connections:** → `Split Out1`
- **Edge cases:**
  - Image policy/content moderation rejections
  - Large prompt length or invalid prompt characters
  - API rate limits / cost

#### Node: Split Out1
- **Type / role:** Split Out; attempts to split a field into separate items for fan-out posting.
- **Config choices:** Field to split out = `choices[0].message.content`
- **Connections:** → Create a post, Facebook1, X1, LinkedIn1, Upload, Telegram
- **Critical issue / edge case:**
  - `choices[0].message.content` is a **chat-completions style** field, but the upstream node is an **image generation** node. The image node output will not match this structure.
  - As wired, this split will likely output nothing or error, preventing downstream posting.
  - If the intent is to fan out *the image + caption*, you typically do **not** split the content like this; you pass a single item containing binary image + text fields.

#### Node: Generate Image (HTTP Request)
- **Type / role:** Direct HTTP call to `POST https://api.openai.com/v1/images/generations`
- **Config choices:**
  - Body params: model=gpt-image-1, prompt=`{{$json.output.prompt.replace(/\"/g, '')}}`, size=1024x1024
  - Auth: Header auth via generic credential
- **Connections:** → Convert to Binary
- **Edge cases:**
  - Expression expects `$json.output.prompt` but `GENERATE PROMPT` outputs `$json.output` (string). This would fail unless you wrap prompt into `output.prompt`.
  - Requires correct OpenAI auth header (`Authorization: Bearer ...`).
- **Sticky note:** “## 4. Connect Image Generation Model … Convert to Binary…”

#### Node: Convert to Binary
- **Type / role:** Convert base64 image payload into binary file data.
- **Config choices:** Source property `data[0].b64_json`
- **Connections:** Not connected to publish nodes in this workflow JSON.
- **Edge cases:** If provider returns URL instead of base64, this conversion will fail; requires adjusting source property.

#### Other image/provider template nodes (not connected)
- **Replicate API, Ideogram API, Clipdrop API, Runway Images, Kling Images, Leonardo Images, Imagen Google API, HuggingFace API, Nano Banana, APITemplate.io**
- **Role:** Examples/placeholders for alternative image generation pipelines.
- **Edge cases:** Each requires provider-specific auth headers, correct request body, and usually a second “download image” step + “Convert to Binary”.

**Sticky notes relevant:**
- “# Customize generation model and output format”
- “# Choose Your Favourite Image Generation Model”
- “## ⚠️To Use Custom Models with HTTP requests …”
- “## 4. Connect Image Generation Model”

---

### Block 2.5 — Posting & Distribution

**Overview:**  
Publishes the generated image (with caption) to Telegram and optionally posts to other platforms and uploads to Google Drive.

**Nodes involved:**
- Telegram
- Create a post (WordPress)
- Facebook1 (Facebook Graph API)
- X1 (Twitter/X)
- LinkedIn1
- Upload (Google Drive)

> Note: In the JSON, these nodes are fed by **Split Out1**, but Split Out1 is very likely misconfigured for an image output. Publishing may not work until the data shape is fixed.

#### Node: Telegram
- **Type / role:** Telegram node to send a photo message.
- **Config choices:**
  - Operation: `sendPhoto`
  - chatId: `123456789` (placeholder)
  - `binaryData: true`
  - Binary property: `data`
  - Caption: `{{ $('GENERATE TEXT').item.json.output }}`
- **Inputs:** Expects binary image in `item.binary.data` from upstream.
- **Connections:** None (end node).
- **Edge cases:**
  - Missing/invalid Telegram credentials (not shown in JSON)
  - chatId wrong
  - Binary not present due to upstream mismatch (most likely current problem)

#### Node: Create a post (WordPress)
- **Type / role:** WordPress node to create content in a WP site.
- **Config choices:** Additional fields empty; core fields not configured in JSON.
- **Inputs:** Would need title/content/media mapping; currently unspecified.
- **Edge cases:** Auth, missing required fields, media upload complexity.

#### Node: Facebook1 (Facebook Graph API)
- **Type / role:** POST action via Graph API.
- **Config choices:** `httpRequestMethod: POST`, options empty.
- **Edge cases:** Token scopes, media upload requires multi-step endpoints, payload not configured.

#### Node: X1 (Twitter)
- **Type / role:** Twitter/X node.
- **Config choices:** Additional fields empty; not configured for tweet text/media.
- **Edge cases:** X API tier limitations, media upload requires correct node operation and fields.

#### Node: LinkedIn1
- **Type / role:** LinkedIn community management post.
- **Config choices:**
  - Text: `{{ $json['choices[0].message.content'] }}`
  - Person: `[CONFIGURE_YOUR_LINKEDIN_PERSON_ID]`
- **Critical issue:** Text mapping uses chat-completions field that likely doesn’t exist.
- **Edge cases:** Requires correct LinkedIn app permissions and person URN/ID.

#### Node: Upload (Google Drive)
- **Type / role:** Upload generated image to a folder.
- **Config choices:**
  - Drive: “My Drive”
  - Folder: `AI Image Generation` (folder ID configured)
  - Name: `{{ $('Image Prompt Agent').item.json.output.title }}.png`
- **Critical issue:** Node reference `Image Prompt Agent` does not exist; `output.title` not produced.
- **Edge cases:** Missing Google Drive credentials, folder permissions, binary property not mapped.

**Sticky notes relevant:**
- “# Post to social media”
- “## 5. Link Your Social Media & CMS”
- “# Post to Wordpress”
- “# Upload to Drive”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking Execute workflow | Manual Trigger | Manual execution entry point | — | GENERATE TEXT | ## 1. Choose Your Trigger … / # Daily AI social content… |
| Schedule Trigger | Schedule Trigger | Daily scheduled run (9 AM) | — | GENERATE TEXT | ## 1. Choose Your Trigger … / # Daily AI social content… |
| Google Sheets Trigger | Google Sheets Trigger | Trigger on new row in sheet | — | GENERATE TEXT | ## 1. Choose Your Trigger … / # Daily AI social content… |
| RSS Feed Trigger | RSS Feed Read Trigger | Trigger on new RSS items | — | GENERATE TEXT | ## 1. Choose Your Trigger … / # Daily AI social content… |
| Meta Reference | Facebook Trigger | Meta webhook entry point | — | GENERATE TEXT | ## 1. Choose Your Trigger … |
| Airtable Trigger | Airtable Trigger | Trigger on Airtable changes | — | GENERATE TEXT | ## 1. Choose Your Trigger … |
| Postgres Trigger | Postgres Trigger | Trigger on Postgres row changes | — | GENERATE TEXT | ## 1. Choose Your Trigger … |
| Simple Memory | Memory Buffer Window | Memory for text agent | — | GENERATE TEXT (ai_memory) | # Choose Your Favourite Chat Model / # Daily AI social content… |
| Search google in SearchApi | SearchApi Tool | Web research tool for agent | — | GENERATE TEXT (ai_tool) | # Choose Your Favourite Chat Model |
| OPENAI WRITES POSTS | OpenAI Chat Model (LangChain) | LLM for text agent (gpt-4.1) | — | GENERATE TEXT (ai_languageModel) | ## 2. Connect Chat Models |
| GENERATE TEXT | LangChain Agent | Generates Telegram post text | Triggers + Memory + Tool + LLM | GENERATE PROMPT | # Detail prompts… / ## 3. Customize Prompts |
| Database | Memory Buffer Window | Memory for prompt agent | — | GENERATE PROMPT (ai_memory) | # Choose Your Favourite Chat Model |
| OPENAI WRITES PROMPTS | OpenAI Chat Model (LangChain) | LLM for prompt agent (gpt-4.1) | — | GENERATE PROMPT (ai_languageModel) | ## 2. Connect Chat Models |
| GENERATE PROMPT | LangChain Agent | Generates image prompt | GENERATE TEXT + LLM + Memory | OPENAI GENERATES IMAGE | # Detail prompts… / ## 3. Customize Prompts |
| OPENAI GENERATES IMAGE | OpenAI Image (LangChain) | Generates image with gpt-image-1 | GENERATE PROMPT | Split Out1 | # Customize generation model… |
| Split Out1 | Split Out | Fans out items to channels (currently mismatched field) | OPENAI GENERATES IMAGE | Create a post, Facebook1, X1, LinkedIn1, Upload, Telegram | # Post to social media |
| Telegram | Telegram | Send photo + caption to Telegram | Split Out1 | — | # Post to social media |
| Create a post | WordPress | Publish to WordPress (not configured) | Split Out1 | — | # Post to Wordpress |
| Facebook1 | Facebook Graph API | Post to Facebook (not configured) | Split Out1 | — | # Post to social media |
| X1 | Twitter/X | Post to X (not configured) | Split Out1 | — | # Post to social media |
| LinkedIn1 | LinkedIn | Post to LinkedIn (text mapping likely wrong) | Split Out1 | — | # Post to social media |
| Upload | Google Drive | Upload image to Drive folder | Split Out1 | — | # Upload to Drive |
| Generate Image | HTTP Request | Alt: OpenAI images API via HTTP | — | Convert to Binary | ## 4. Connect Image Generation Model |
| Convert to Binary | Convert to File | Convert base64 to binary image | Generate Image | — | ## 4. Connect Image Generation Model |
| Replicate API | HTTP Request | Template: Replicate image gen | — | — | # Choose Your Favourite Image Generation Model |
| Imagen Google API | HTTP Request | Template: Google Imagen endpoint | — | — | # Choose Your Favourite Image Generation Model |
| HuggingFace API | HTTP Request | Template: HF endpoint example | — | — | # Choose Your Favourite Image Generation Model |
| Kling Images | HTTP Request | Template: Kling image API | — | — | # Choose Your Favourite Image Generation Model |
| Runway Images | HTTP Request | Template: Runway text-to-image | — | — | # Choose Your Favourite Image Generation Model |
| Leonardo Images | HTTP Request | Template: Leonardo API | — | — | # Choose Your Favourite Image Generation Model |
| APITemplate.io | APITemplate.io | Template-based image generation | — | — | # Choose Your Favourite Image Generation Model |
| Ideogram API | HTTP Request | Template: Ideogram generate | — | — | # Choose Your Favourite Image Generation Model |
| Clipdrop API | HTTP Request | Template: Clipdrop text-to-image | — | — | # Choose Your Favourite Image Generation Model |
| Nano Banana | HTTP Request | Template: BananaAPI generate | — | — | # Choose Your Favourite Image Generation Model |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | Alternative chat model (gpt-4.1-mini) | — | — | # Choose Your Favourite Chat Model |
| Mistral Cloud Chat Model | Mistral Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| OpenRouter Chat Model | OpenRouter Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| Ollama Chat Model | Ollama Chat Model (LangChain) | Alternative local chat model | — | — | # Choose Your Favourite Chat Model |
| xAI Grok Chat Model | xAI Grok Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| DeepSeek Chat Model | DeepSeek Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| AWS Bedrock Chat Model1 | AWS Bedrock Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| Groq Chat Model1 | Groq Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| Anthropic Chat Model | Anthropic Chat Model (LangChain) | Alternative chat model (Claude) | — | — | # Choose Your Favourite Chat Model |
| Google Gemini Chat Model | Google Gemini Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| Azure OpenAI Chat Model | Azure OpenAI Chat Model (LangChain) | Alternative chat model | — | — | # Choose Your Favourite Chat Model |
| Hugging Face Inference Model | Hugging Face Inference (LangChain) | Model endpoint reference (positioned near image options) | — | — | # Choose Your Favourite Image Generation Model |
| Sticky Note nodes (all) | Sticky Note | Documentation only | — | — | (not applicable) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create triggers (choose at least one)**
1. Add **Manual Trigger** node named “When clicking Execute workflow”.
2. Add **Schedule Trigger** node named “Schedule Trigger” and set it to run daily at **09:00** (or your timezone).
3. (Optional) Add **Google Sheets Trigger**:
   - Credentials: Google OAuth2 with access to the target spreadsheet.
   - Event: *Row Added*; configure sheet + column to watch + a sufficiently large range (avoid A2:A10).
4. (Optional) Add **RSS Feed Trigger**; configure RSS URL and polling interval.
5. (Optional) Add **Airtable Trigger**; configure base/table and credentials.
6. (Optional) Add **Postgres Trigger**; configure DB credentials, schema/table.
7. (Optional) Add **Facebook Trigger** (Meta webhook) and complete Meta webhook verification.

2) **Add the Text Agent block**
8. Add **OpenAI Chat Model (LangChain)** node named “OPENAI WRITES POSTS”:
   - Model: `gpt-4.1`
   - Credentials: OpenAI API key.
9. Add **Memory Buffer Window** named “Simple Memory” (default settings are fine initially).
10. Add **SearchApi Tool** node named “Search google in SearchApi”:
   - Configure SearchApi credentials (per your SearchApi setup).
   - Leave the query as agent-driven (`$fromAI('Query', ...)`).
11. Add **AI Agent** node named “GENERATE TEXT”:
   - System message: paste your brand rules and Telegram constraints.
   - User message: include the incoming reference (ensure you map the correct field; do not assume `$json.name` for every trigger).
12. Connect:
   - Trigger(s) → **GENERATE TEXT** (main)
   - **OPENAI WRITES POSTS** → **GENERATE TEXT** (ai_languageModel)
   - **Simple Memory** → **GENERATE TEXT** (ai_memory)
   - **Search google in SearchApi** → **GENERATE TEXT** (ai_tool)

3) **Add the Image Prompt Agent block**
13. Add **OpenAI Chat Model (LangChain)** node named “OPENAI WRITES PROMPTS”:
   - Model: `gpt-4.1`
   - Credentials: OpenAI API key.
14. Add **Memory Buffer Window** node (name it “Prompt Memory” to avoid confusion).
15. Add **AI Agent** node named “GENERATE PROMPT”:
   - User message: “Create a highly detailed photorealistic visual concept that matches: {{ $json.output }} …”
   - System message: rules banning text/logos and requiring a single English prompt.
16. Connect:
   - **GENERATE TEXT** → **GENERATE PROMPT** (main)
   - **OPENAI WRITES PROMPTS** → **GENERATE PROMPT** (ai_languageModel)
   - Prompt Memory → **GENERATE PROMPT** (ai_memory)

4) **Add Image Generation**
17. Add **OpenAI (Image)** node named “OPENAI GENERATES IMAGE”:
   - Resource: Image
   - Model: `gpt-image-1`
   - Prompt: use `{{ $json.output }}` from “GENERATE PROMPT” plus a “no text in image” instruction.
18. Connect **GENERATE PROMPT** → **OPENAI GENERATES IMAGE**.

5) **Prepare binary for Telegram/Drive (recommended fix)**
19. Ensure the image node outputs a binary file property usable by Telegram/Drive. If it returns URLs/base64 instead:
   - Add an **HTTP Request** to download the image (if URL)
   - Or add **Convert to File** (if base64) with correct source property
20. Avoid using **Split Out** unless you truly have an array to split.

6) **Posting**
21. Add **Telegram** node:
   - Operation: Send Photo
   - chatId: your real chat/channel ID
   - Binary property name: the property holding the image (commonly `data`)
   - Caption: `{{ $('GENERATE TEXT').item.json.output }}`
22. Add **Google Drive** node “Upload”:
   - Operation: Upload
   - Folder ID: target folder
   - File name: derive from a safe field (e.g. a timestamp) because the provided `Image Prompt Agent…output.title` mapping will not exist unless you create it.
23. (Optional) Add and configure:
   - WordPress “Create a post” (title/content/status)
   - Facebook Graph API post (proper endpoint + media upload flow)
   - X post (tweet text + media)
   - LinkedIn post (person ID + proper media share flow)

7) **Connect distribution**
24. Connect **OPENAI GENERATES IMAGE** (or your binary conversion node) → Telegram / Drive / other channels.
25. Test via **Manual Trigger**, then enable the schedule trigger.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| [made with ❤️ by N8ner 👈 click! Feel free to message me!](https://community.n8n.io/u/n8ner/badges) | Author credit sticky note |
| Workflow sections explanation (Sources → AI post → Visual prompt → Image gen → Distribution) | Sticky note “Daily AI social content for beauty brands” |
| “You only need to configure the platforms that matter… the rest can stay inactive.” | Sticky note “Link Your Social Media & CMS” |
| Image generation swap guidance (Clipdrop/Ideogram/Replicate/Runway/Leonardo etc.) | Sticky note “Connect Image Generation Model” |

