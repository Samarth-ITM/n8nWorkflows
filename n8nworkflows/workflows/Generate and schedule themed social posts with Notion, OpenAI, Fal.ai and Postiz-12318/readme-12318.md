Generate and schedule themed social posts with Notion, OpenAI, Fal.ai and Postiz

https://n8nworkflows.xyz/workflows/generate-and-schedule-themed-social-posts-with-notion--openai--fal-ai-and-postiz-12318


# Generate and schedule themed social posts with Notion, OpenAI, Fal.ai and Postiz

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

# Generate and schedule themed social posts with Notion, OpenAI, Fal.ai and Postiz

## 1. Workflow Overview

This workflow automatically generates a daily, on-brand social media post for “MetrixMate” by:
1) selecting the least-recently-used “Post Theme” from Notion,  
2) generating an image prompt with OpenAI (three theme styles supported),  
3) generating an image with Fal.ai Imagen 4, saving it to Google Drive,  
4) generating a caption (also via OpenAI, with Notion + Drive as tools),  
5) uploading the media to Postiz and creating drafts for multiple social networks (LinkedIn, X, Facebook, Instagram).

### 1.1 Input Reception (Scheduling)
- Runs every day at 12:00.

### 1.2 Data & Strategy (Notion context)
- Pulls Brand Guidelines content (Notion page blocks).
- Pulls a single Post Theme (least recently used) from a Notion database.
- Merges theme + brand context into one payload.

### 1.3 AI Prompting & Image Generation (3 theme routes)
- Switches by theme name: **Expert Advise**, **System**, **Activity**.
- For the chosen route:
  - OpenAI generates an image prompt (JSON output expected).
  - Fal.ai Imagen 4 generates an image (returns an image URL).
  - The image is downloaded and saved to Google Drive.
  - The Notion theme page is updated with “Last Used |date”.

### 1.4 Captioning & Publishing (Postiz distribution)
- Generates a caption using an LLM Agent that can call:
  - Notion Tool (to fetch DB pages as “brand details” context)
  - Google Drive Tool (to download/analyze the image)
- Downloads the Drive image (binary) for upload.
- Uploads the media to Postiz.
- Creates Postiz post drafts for LinkedIn, X, Facebook, Instagram using integration IDs.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Scheduling)

**Overview:** Starts the workflow daily at a fixed time.

**Nodes involved:**
- Daily Trigger

**Node details**

#### 1) Daily Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration:** Runs every day at **12:00** (triggerAtHour: 12).
- **Outputs:** Sends a single item to:
  - **Get Post Theme**
  - **Get Brand Guidelines**
- **Failure/edge cases:**
  - Server timezone matters (n8n instance timezone); “12 PM” may not match your local timezone unless configured.

---

### Block 2 — Data & Strategy (Notion context)

**Overview:** Fetches brand guidelines text and selects today’s theme from Notion, then merges both into one object for downstream AI generation.

**Nodes involved:**
- Get Brand Guidelines
- Format Brand Text
- Get Post Theme
- Merge Data
- Route by Theme

**Node details**

#### 2) Get Brand Guidelines
- **Type / role:** `Notion` — reads a Notion page’s blocks (brand document).
- **Configuration choices:**
  - Resource: **block**
  - Operation: **getAll**
  - Block ID provided via Notion URL: `MetrixMate-Brand-Document-...`
  - `returnAll: true`
- **Output:** Multiple items (one per block), each with `json.content` (used later).
- **Connections:** Output → **Format Brand Text**
- **Failure/edge cases:**
  - Notion auth/permissions (integration must have access to the page).
  - Some blocks may not contain `content` → handled by filtering in Code node.

#### 3) Format Brand Text
- **Type / role:** `Code` — concatenates Notion block contents into a single text blob.
- **Configuration choices:**
  - Joins all incoming `i.json.content` with newline.
  - Outputs `{ content: allText }`.
- **Key variables/logic:**
  - `$input.all()` to collect all blocks.
- **Connections:** Output → **Merge Data** (input index 1)
- **Failure/edge cases:**
  - If Notion returns no blocks with `content`, output becomes empty string; downstream prompts may be lower quality.

#### 4) Get Post Theme
- **Type / role:** `Notion` — selects one theme page from a Notion database.
- **Configuration choices:**
  - Resource: **databasePage**
  - Operation: **getAll**
  - Database: `MM - Theme DB` (ID: `24921223-e79e-8056-95e9-c3554fab09dc`)
  - `limit: 1`
  - Sort: `Last Used |date` ascending (least-recently-used first)
  - Filter type: manual (but no explicit filter conditions shown)
- **Output:** Single theme page item; includes fields like `name`, and custom properties referenced later:
  - `property_creative_brief`
  - `property_tone_voice`
  - `property_example_prompts`
- **Connections:** Output → **Merge Data** (input index 0)
- **Failure/edge cases:**
  - If database is empty, workflow stops (no item to route).
  - If `Last Used |date` is missing/invalid for some pages, Notion sorting can behave unexpectedly.

#### 5) Merge Data
- **Type / role:** `Merge` — combines theme item + brand text.
- **Configuration choices:** Default Merge node settings (in n8n, default is typically “Combine” by position/index).
- **Inputs:**
  - Input 0: **Get Post Theme** (1 item)
  - Input 1: **Format Brand Text** (1 item)
- **Output:** One merged item used by routing and prompt generation.
- **Connections:** Output → **Route by Theme**
- **Failure/edge cases:**
  - If either side produces 0 items, merge may output 0 items depending on merge mode. Verify merge mode if you modify upstream nodes.

#### 6) Route by Theme
- **Type / role:** `Switch` — routes logic based on theme name.
- **Configuration choices:**
  - Rules check: `={{ $json.name }}`
  - Routes:
    1. equals `"Expert Advise"`
    2. equals `"System"`
    3. equals `"Activity"`
- **Outputs:**
  - Output 0 → **Gen Prompt (Expert)**
  - Output 1 → **Gen Prompt (System)**
  - Output 2 → **Gen Prompt (Activity)**
- **Failure/edge cases:**
  - If theme name doesn’t match exactly (typo, different capitalization), it won’t go to any route and workflow effectively ends.
  - Consider adding a default route for unknown themes.

---

### Block 3 — AI Content Generation (Theme-specific prompt + Fal.ai image + Drive save + Notion update)

**Overview:** Depending on the selected theme type, the workflow generates an image prompt (LLM), generates an image (Fal.ai Imagen 4), downloads it, saves to Google Drive, and updates the Notion theme page as “last used”.

**Nodes involved (theme routes):**
- Expert route: OpenAI Chat Model5, Gen Prompt (Expert), Gen Image (Expert), Get Image URL (Expert), Save to Drive (Expert), Update Notion (Expert)
- System route: OpenAI Chat Model4, Gen Prompt (System), Gen Image (System), Get Image URL (System), Save to Drive (System), Update Notion (System)
- Activity route: gpt-4o LLM2, Gen Prompt (Activity), Gen Image (Activity), Get Image URL (Activity), Save to Drive (Activity), Update Notion (Activity)

#### Expert Route

##### 7) OpenAI Chat Model5
- **Type / role:** `lmChatOpenAi` (LangChain) — provides LLM to the Agent.
- **Configuration:** Model `chatgpt-4o-latest`.
- **Connections:** `ai_languageModel` → **Gen Prompt (Expert)**
- **Failure/edge cases:** OpenAI credential missing; model name not available in your account/region.

##### 8) Gen Prompt (Expert)
- **Type / role:** `LangChain Agent` — creates an image prompt for “Expert Advise”.
- **Configuration choices:**
  - Prompt instructs: match sample prompt style; define headline/background; append: `Write MetrixMate at the bottom`
  - Injects Notion properties:
    - `{{ $json.property_creative_brief }}`
    - `{{ $json.property_tone_voice }}`
    - `{{ $json.property_example_prompts }}`
  - Output parser enabled (`hasOutputParser: true`) expecting JSON (but systemMessage is empty here).
- **Output:** `json.output` is used as the Fal prompt.
- **Connections:** Main → **Gen Image (Expert)**
- **Failure/edge cases:**
  - If the agent fails to output valid JSON, `json.output` may be missing or malformed.
  - Prompt mentions “Always add statement …” but doesn’t strictly enforce JSON output with system message; consider adding a strict JSON system message like the System route does.

##### 9) Gen Image (Expert)
- **Type / role:** `HTTP Request` — calls Fal.ai Imagen 4 preview endpoint.
- **Configuration:**
  - POST `https://fal.run/fal-ai/imagen4/preview`
  - JSON body: `{ "prompt": $json.output, "image_model": "imagen4", "aspect_ratio": "1:1" }`
  - Auth: `genericCredentialType` + `httpHeaderAuth` (Fal key typically in Authorization header)
- **Output:** Contains `images[0].url` and `images[0].file_name`
- **Connections:** → **Get Image URL (Expert)**
- **Failure/edge cases:**
  - Fal credits exhausted / rate limits.
  - Endpoint returns error or different structure; expressions expecting `images[0]` will break.

##### 10) Get Image URL (Expert)
- **Type / role:** `HTTP Request` — downloads the generated image by URL.
- **Configuration:** URL `={{ $json.images[0].url }}`
- **Output:** Binary image data typically available to downstream nodes.
- **Connections:** → **Save to Drive (Expert)**
- **Failure/edge cases:**
  - `images[0].url` missing; request fails.
  - Large images/timeouts.

##### 11) Save to Drive (Expert)
- **Type / role:** `Google Drive` — uploads the image to a Drive folder.
- **Configuration:**
  - Name: `={{ $json.images[0].file_name }}`
  - Folder: `n8n Content Folder` (ID `1dlLOiWnqdB2JW4j_pyrB8KehS2mEQ0N-`)
  - Auth: Service Account
  - Input binary field: `data`
- **Output:** Drive file metadata including `id`.
- **Connections:** → **Update Notion (Expert)**
- **Failure/edge cases:**
  - Service account lacks folder access (must be shared with SA email).
  - If prior node doesn’t output binary in `data`, upload fails.

##### 12) Update Notion (Expert)
- **Type / role:** `Notion` — updates the specific theme page to set last-used date.
- **Configuration:**
  - Operation: update databasePage
  - Page URL: `Expert Advise-...`
  - Property set: `Last Used |date` to localized date string:
    - `{{ new Date().toLocaleDateString('en-US', { year:'numeric', month:'long', day:'numeric' }) }}`
- **Connections:** → **Generate Caption (Expert)**
- **Failure/edge cases:**
  - Notion “date” property expects an ISO date; here it’s a formatted string (“January 8, 2026”). If Notion rejects it, update fails.
  - Safer: use ISO string `new Date().toISOString()` and map to a date field format.

#### System Route

##### 13) OpenAI Chat Model4
- **Type / role:** `lmChatOpenAi` — LLM provider.
- **Configuration:** `chatgpt-4o-latest`
- **Connections:** `ai_languageModel` → **Gen Prompt (System)**

##### 14) Gen Prompt (System)
- **Type / role:** `LangChain Agent` — generates a System UI themed prompt with strict rules.
- **Configuration highlights:**
  - Strong `systemMessage` with explicit brand/style constraints and **strict JSON output format**: `{ "prompt": "..." }`
  - Text includes the same Notion properties (creative brief, tone, example prompts).
  - Has output parser enabled.
- **Connections:** → **Gen Image (System)**
- **Failure/edge cases:**
  - If the agent returns JSON with key `prompt`, but workflow expects `$json.output`, you must confirm agent node’s actual output mapping in your n8n version. Current downstream uses `$json.output`.

##### 15) Gen Image (System)
- Same as Expert route but uses `$json.output`.

##### 16) Get Image URL (System)
- Same as Expert route.

##### 17) Save to Drive (System)
- Same Drive folder, same binary field `data`, outputs Drive file `id`.

##### 18) Update Notion (System)
- Updates page “System-...” and sets `Last Used |date` using the same formatted string approach.
- Output → **Generate Caption (System)**

#### Activity Route

##### 19) gpt-4o LLM2
- **Type / role:** `lmChatOpenAi` — LLM provider for Activity route.
- **Configuration:** `chatgpt-4o-latest`, response format “text”.
- **Connections:** `ai_languageModel` → **Gen Prompt (Activity)**

##### 20) Gen Prompt (Activity)
- **Type / role:** `LangChain Agent` — generates metaphorical/activity-style image prompt.
- **Configuration highlights:**
  - Text: asks for “Metaphorical/Activity theme”, organic scene, tied to MetrixMate; end with `Write MetrixMate at the bottom`
  - `systemMessage` is set to `"="` (likely unintended), output parser enabled.
- **Connections:** → **Gen Image (Activity)**
- **Failure/edge cases:**
  - The system message `"="` provides no guardrails; output JSON validity may be inconsistent.

##### 21) Gen Image (Activity)
- Fal.ai Imagen 4 call identical pattern.

##### 22) Get Image URL (Activity)
- Downloads the image URL.

##### 23) Save to Drive (Activity)
- Uploads image to Drive, same folder & binary field.

##### 24) Update Notion (Activity)
- Updates page “Activity-...” with `Last Used |date`.
- Output → **Generate Caption (Activity)**

---

### Block 4 — Captioning & Publishing (Agent tools, Drive download, Postiz upload, multi-channel drafts)

**Overview:** Generates captions using agent nodes that can call Notion/Drive as tools, then uploads the image to Postiz and creates drafts for multiple social networks.

**Nodes involved:**
- Generate Caption (Expert)
- Generate Caption (System)
- Generate Caption (Activity)
- notion / notion1 / notion2 (Notion Tool)
- drive2 / drivefile / drive1 (Google Drive Tool)
- openai / openai1 / openai2 (OpenAI chat models for caption agents)
- Download for Upload
- Download from Drive (System)
- Download from Drive (Activity)
- Upload to Postiz
- Post to Instagram
- Post to Facebook
- Post to Linkedin
- Post to X

**Node details (caption agents + tools)**

#### 25) openai / 26) openai1 / 27) openai2
- **Type / role:** `lmChatOpenAi` — LLM providers for caption agents.
- **Configuration:** Model `gpt-4.1-mini`
- **Connections:** Each connects via `ai_languageModel` to its respective caption agent:
  - openai → Generate Caption (Expert)
  - openai1 → Generate Caption (System)
  - openai2 → Generate Caption (Activity)
- **Failure/edge cases:** Credential/model access issues.

#### 28) notion / 29) notion1 / 30) notion2
- **Type / role:** `Notion Tool` — tool callable by LangChain agent.
- **Configuration:** databasePage → getAll from `MM - Theme DB`.
- **Connections:** Each is attached as `ai_tool` to the respective caption agent.
- **Failure/edge cases:**
  - Potentially expensive: “getAll” can return many pages; consider filtering to brand DB instead of theme DB if intent is “brand details”.
  - Notion Tool permissions.

#### 31) drive2 / 32) drivefile / 33) drive1
- **Type / role:** `Google Drive Tool` — tool callable by LangChain agent.
- **Configuration:** operation download; fileId from the saved Drive file for each route.
- **Notes:** Each has node note: “ignore the unexcited nodes ” (likely meaning “unused”).
- **Connections:** Each attached as `ai_tool` to its caption agent.
- **Failure/edge cases:**
  - Tools may not actually be used unless the agent decides to call them.
  - If fileId expression resolves empty, tool download fails.

#### 34) Generate Caption (Expert)
- **Type / role:** `LangChain Agent` — creates a caption (plain text).
- **Prompt constraints:**
  - “Use the Notion tool … and google drive to analysis the image”
  - Descriptive, a little longer
  - Must end with `matrixmate.com` (note spelling differs from MetrixMate)
  - No extra analysis; plain text; no special characters/formatting
- **Tools/LLM:** uses `openai` + `notion` + `drive2`
- **Connections:** Main → **Download for Upload**
- **Failure/edge cases:**
  - “matrixmate.com” vs “metrixmate.com” inconsistency across prompts/nodes.
  - Agent may not actually call tools; if you require tool usage, add stronger instructions or a structured plan.

#### 35) Generate Caption (System)
- Similar to Expert but routes to **Download from Drive (System)**.
- Uses `openai1` + `notion1` + `drivefile`.

#### 36) Generate Caption (Activity)
- Similar but explicitly “No longer than 20-25 words”; ends with `metrixmate.com` (different spelling from Expert/System instructions).
- Uses `openai2` + `notion2` + `drive1`.
- Routes to **Download from Drive (Activity)**.

**Node details (Drive download for upload)**

#### 37) Download for Upload
- **Type / role:** `Google Drive` — downloads the Expert image as binary for upload.
- **Configuration:** operation download; fileId from `Save to Drive (Expert).item.json.id`
- **Connections:** → **Upload to Postiz**
- **Failure/edge cases:** fileId empty if Expert path didn’t run; in practice, this node only runs when Expert caption node runs.

#### 38) Download from Drive (System)
- Same as above but uses `Save to Drive (System)` file id.
- Output → **Upload to Postiz**

#### 39) Download from Drive (Activity)
- Same as above but uses `Save to Drive (Activity)` file id.
- Output → **Upload to Postiz**

**Node details (Postiz upload + posting)**

#### 40) Upload to Postiz
- **Type / role:** `HTTP Request` — uploads binary media to Postiz.
- **Configuration:**
  - POST `https://api.postiz.com/public/v1/upload`
  - `multipart-form-data`
  - Form field `file` comes from binary property `data`
  - Header includes `Authorization` (must be set via credentials or node parameters)
  - Option: lowercaseHeaders true
- **Output:** Expected to include uploaded media metadata (`id`, `path` used later).
- **Connections:** fans out to:
  - Post to X
  - Post to Linkedin
  - Post to Facebook
  - Post to Instagram
- **Failure/edge cases:**
  - Missing/invalid Authorization token.
  - If binary data field isn’t `data`, upload will fail.
  - API response shape changes can break downstream expressions using `$json.id` and `$json.path`.

#### 41) Post to Instagram / 42) Post to Facebook / 43) Post to Linkedin / 44) Post to X
- **Type / role:** `HTTP Request` — creates a Postiz post draft per network integration.
- **Configuration:**
  - POST `https://api.postiz.com/public/v1/posts`
  - JSON body includes:
    - `type: "draft"`
    - `date: {{ new Date().toISOString() }}`
    - `shortLink: true`
    - `tags: tag1, tag2`
    - `posts[0].integration.id`: network-specific integration ID
    - `posts[0].value[0].content`: `{{ $('Generate Caption (Expert)').first().json.output }}`
    - `image[0].id`: `{{ $json.id }}`
    - `image[0].url/path`: `{{ $json.path }}`
- **Important integration detail:**
  - All four nodes always reference **Generate Caption (Expert)** for content, even if the System or Activity route ran. This is likely a bug: System/Activity runs will post Expert caption or fail if that node didn’t execute.
- **Failure/edge cases:**
  - Wrong integration IDs → drafts fail or go to wrong accounts.
  - Missing caption output or mismatch route.
  - Postiz API auth (these nodes do not visibly set Authorization header; you may need to configure headers/credentials similarly to upload).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation |  |  | ## How it works<br>1. **Daily Trigger:** Starts automatically every day at 12 PM.<br>2. **Context Gathering:** Fetches your 'Brand Guidelines' and the specific 'Post Theme' for the day from Notion.<br>3. **Image Generation:** Uses OpenAI to create a detailed image prompt based on the theme, then generates the image using Fal.ai (Imagen 3/4).<br>4. **Captioning:** Generates an on-brand caption using OpenAI.<br>5. **Publishing:** Uploads the media to Postiz and schedules/posts it to LinkedIn, X, Facebook, and Instagram.<br><br>## Setup steps<br>1. **Notion:** Create two databases: one for 'Brand Guidelines' and one for 'Post Themes'. Update the Notion nodes with your Database IDs.<br>2. **Credentials:** Configure credentials for OpenAI, Fal.ai, Google Drive, Notion, and Postiz.<br>3. **Fal.ai:** Ensure you have credits and the correct model selected in the HTTP nodes.<br>4. **Postiz:** Update the 'Integration IDs' in the final social media nodes with your specific Postiz integration IDs. |
| Sticky Note1 | Sticky Note | Documentation |  |  | ### 1. Data & Strategy<br>Fetches your brand voice and the specific theme for today's post from Notion. |
| Sticky Note2 | Sticky Note | Documentation |  |  | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Sticky Note3 | Sticky Note | Documentation |  |  | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Daily Trigger | Schedule Trigger | Daily entry point | — | Get Post Theme; Get Brand Guidelines | ### 1. Data & Strategy<br>Fetches your brand voice and the specific theme for today's post from Notion. |
| Get Brand Guidelines | Notion | Fetch brand blocks | Daily Trigger | Format Brand Text | ### 1. Data & Strategy<br>Fetches your brand voice and the specific theme for today's post from Notion. |
| Format Brand Text | Code | Join brand blocks into one text | Get Brand Guidelines | Merge Data | ### 1. Data & Strategy<br>Fetches your brand voice and the specific theme for today's post from Notion. |
| Get Post Theme | Notion | Select LRU theme | Daily Trigger | Merge Data | ### 1. Data & Strategy<br>Fetches your brand voice and the specific theme for today's post from Notion. |
| Merge Data | Merge | Combine theme + brand text | Get Post Theme; Format Brand Text | Route by Theme | ### 1. Data & Strategy<br>Fetches your brand voice and the specific theme for today's post from Notion. |
| Route by Theme | Switch | Route to Expert/System/Activity | Merge Data | Gen Prompt (Expert); Gen Prompt (System); Gen Prompt (Activity) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| OpenAI Chat Model5 | OpenAI Chat Model (LangChain) | LLM for Expert prompt agent | — (AI link) | Gen Prompt (Expert) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Gen Prompt (Expert) | LangChain Agent | Generate Expert image prompt | Route by Theme | Gen Image (Expert) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Gen Image (Expert) | HTTP Request | Fal.ai Imagen4 generate | Gen Prompt (Expert) | Get Image URL (Expert) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Get Image URL (Expert) | HTTP Request | Download image by URL | Gen Image (Expert) | Save to Drive (Expert) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Save to Drive (Expert) | Google Drive | Upload image to Drive | Get Image URL (Expert) | Update Notion (Expert) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Update Notion (Expert) | Notion | Mark theme as used | Save to Drive (Expert) | Generate Caption (Expert) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| OpenAI Chat Model4 | OpenAI Chat Model (LangChain) | LLM for System prompt agent | — (AI link) | Gen Prompt (System) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Gen Prompt (System) | LangChain Agent | Generate System UI image prompt | Route by Theme | Gen Image (System) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Gen Image (System) | HTTP Request | Fal.ai Imagen4 generate | Gen Prompt (System) | Get Image URL (System) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Get Image URL (System) | HTTP Request | Download image by URL | Gen Image (System) | Save to Drive (System) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Save to Drive (System) | Google Drive | Upload image to Drive | Get Image URL (System) | Update Notion (System) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Update Notion (System) | Notion | Mark theme as used | Save to Drive (System) | Generate Caption (System) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| gpt-4o LLM2 | OpenAI Chat Model (LangChain) | LLM for Activity prompt agent | — (AI link) | Gen Prompt (Activity) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Gen Prompt (Activity) | LangChain Agent | Generate Activity image prompt | Route by Theme | Gen Image (Activity) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Gen Image (Activity) | HTTP Request | Fal.ai Imagen4 generate | Gen Prompt (Activity) | Get Image URL (Activity) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Get Image URL (Activity) | HTTP Request | Download image by URL | Gen Image (Activity) | Save to Drive (Activity) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Save to Drive (Activity) | Google Drive | Upload image to Drive | Get Image URL (Activity) | Update Notion (Activity) | ### 2. AI Content Generation<br>Three parallel paths handle different content styles (System, Activity, Expert). Each path generates a prompt via OpenAI and an image via Fal.ai. |
| Update Notion (Activity) | Notion | Mark theme as used | Save to Drive (Activity) | Generate Caption (Activity) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| openai | OpenAI Chat Model (LangChain) | LLM for Expert caption agent | — (AI link) | Generate Caption (Expert) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| notion | Notion Tool | Tool for caption agent (DB getAll) | — (tool link) | Generate Caption (Expert) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| drive2 | Google Drive Tool | Tool for caption agent (download) | — (tool link) | Generate Caption (Expert) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Generate Caption (Expert) | LangChain Agent | Create Expert caption | Update Notion (Expert) | Download for Upload | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| openai1 | OpenAI Chat Model (LangChain) | LLM for System caption agent | — (AI link) | Generate Caption (System) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| notion1 | Notion Tool | Tool for caption agent (DB getAll) | — (tool link) | Generate Caption (System) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| drivefile | Google Drive Tool | Tool for caption agent (download) | — (tool link) | Generate Caption (System) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Generate Caption (System) | LangChain Agent | Create System caption | Update Notion (System) | Download from Drive (System) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| openai2 | OpenAI Chat Model (LangChain) | LLM for Activity caption agent | — (AI link) | Generate Caption (Activity) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| notion2 | Notion Tool | Tool for caption agent (DB getAll) | — (tool link) | Generate Caption (Activity) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| drive1 | Google Drive Tool | Tool for caption agent (download) | — (tool link) | Generate Caption (Activity) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Generate Caption (Activity) | LangChain Agent | Create Activity caption | Update Notion (Activity) | Download from Drive (Activity) | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Download for Upload | Google Drive | Download Expert image binary | Generate Caption (Expert) | Upload to Postiz | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Download from Drive (System) | Google Drive | Download System image binary | Generate Caption (System) | Upload to Postiz | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Download from Drive (Activity) | Google Drive | Download Activity image binary | Generate Caption (Activity) | Upload to Postiz | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Upload to Postiz | HTTP Request | Upload media to Postiz | Download for Upload / Download from Drive (System) / Download from Drive (Activity) | Post to X; Post to Linkedin; Post to Facebook; Post to Instagram | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Post to Instagram | HTTP Request | Create IG post draft in Postiz | Upload to Postiz | — | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Post to Facebook | HTTP Request | Create FB post draft in Postiz | Upload to Postiz | — | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Post to Linkedin | HTTP Request | Create LinkedIn post draft in Postiz | Upload to Postiz | — | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |
| Post to X | HTTP Request | Create X post draft in Postiz | Upload to Postiz | — | ### 3. Captioning & Publishing<br>Generates the final caption, updates Notion history, and pushes the content to Postiz for distribution. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Schedule Trigger**
   - Add node: *Schedule Trigger*
   - Set to run daily at **12:00** (adjust timezone in n8n settings if needed).

2) **Fetch brand guidelines from Notion**
   - Add Notion node “Get Brand Guidelines”
   - Resource: **Block** → Operation: **Get All**
   - Use the Brand Guidelines page URL (block/page).
   - Ensure Notion credentials have access.

3) **Format brand text**
   - Add Code node “Format Brand Text”
   - JS: concatenate all incoming `json.content` into one `content` field.

4) **Fetch today’s theme from Notion DB**
   - Add Notion node “Get Post Theme”
   - Resource: **Database Page** → **Get All**
   - Database ID: your Themes DB
   - Set **Limit = 1**
   - Sort by `Last Used |date` ascending.

5) **Merge theme + brand text**
   - Add Merge node “Merge Data”
   - Connect:
     - Get Post Theme → Merge input 0
     - Format Brand Text → Merge input 1
   - Use a merge mode that results in a single combined item (typically “Combine” by position).

6) **Route by theme name**
   - Add Switch node “Route by Theme”
   - Value: `={{ $json.name }}`
   - Add routes:
     - “Expert Advise”
     - “System”
     - “Activity”

7) **For each route: create an image prompt (LLM Agent)**
   - Add an OpenAI Chat Model node for each route (or reuse one):
     - Model: `chatgpt-4o-latest` (prompt generation)
   - Add a LangChain Agent node:
     - “Gen Prompt (Expert)”, “Gen Prompt (System)”, “Gen Prompt (Activity)”
     - Provide route-specific prompt text.
     - Prefer enforcing strict JSON output and ensure downstream uses the correct field (e.g., `output` vs parsed JSON key).

8) **Generate image with Fal.ai**
   - Add HTTP Request nodes “Gen Image (...)”
   - POST `https://fal.run/fal-ai/imagen4/preview`
   - JSON body: `{ prompt: <agentOutput>, image_model: "imagen4", aspect_ratio: "1:1" }`
   - Add Fal.ai credential:
     - Use Header Auth (commonly `Authorization: Key <FAL_KEY>` or Bearer per Fal docs).

9) **Download the image**
   - Add HTTP Request nodes “Get Image URL (...)”
   - URL: `={{ $json.images[0].url }}`

10) **Save image to Google Drive**
   - Add Google Drive nodes “Save to Drive (...)”
   - Operation: Upload
   - Authentication: Service Account (share folder with SA email)
   - Folder: choose your content folder
   - Binary property: `data`
   - Name: `={{ $json.images[0].file_name }}`

11) **Update Notion “Last Used |date”**
   - Add Notion update nodes per route
   - Operation: databasePage → update
   - Target the specific theme page (or update the page returned by Get Post Theme if you redesign)
   - Set `Last Used |date`
   - Recommendation: store ISO date for reliability.

12) **Caption generation (per route)**
   - Add OpenAI Chat Model nodes:
     - Model: `gpt-4.1-mini` (captioning)
   - Add Notion Tool node (optional but used here):
     - databasePage → getAll (consider pointing to a Brand DB instead)
   - Add Google Drive Tool node (optional):
     - download file using the saved file id
   - Add LangChain Agent nodes “Generate Caption (...)”
     - Instruct plain text output, include domain at end, word limits, etc.
     - Attach the OpenAI model as `ai_languageModel` and the tools as `ai_tool`.

13) **Download from Drive for Postiz upload**
   - Add Google Drive download nodes:
     - “Download for Upload” (Expert), “Download from Drive (System)”, “Download from Drive (Activity)”
   - File ID expressions should reference the correct “Save to Drive (...)” node output id.

14) **Upload media to Postiz**
   - Add HTTP Request node “Upload to Postiz”
   - POST `https://api.postiz.com/public/v1/upload`
   - Content type: multipart-form-data
   - Body parameter: `file` from binary `data`
   - Add header `Authorization: <POSTIZ_API_KEY>` (exact scheme per Postiz docs/account).

15) **Create Postiz drafts per network**
   - Add 4 HTTP Request nodes:
     - Post to Instagram / Facebook / Linkedin / X
   - POST `https://api.postiz.com/public/v1/posts`
   - Set integration IDs to your Postiz integration ids.
   - Ensure caption reference matches the active route (do not hardcode Expert unless intended).
   - Use uploaded media response fields (`id`, `path`) consistently.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Fal.ai: Ensure you have credits and the correct model selected in the HTTP nodes.” | From sticky note “Setup steps” |
| “Postiz: Update the 'Integration IDs' in the final social media nodes with your specific Postiz integration IDs.” | From sticky note “Setup steps” |
| Node notes: “ignore the unexcited nodes ” | Present on drive1/drive2/drivefile (Google Drive Tool nodes) |
| Potential inconsistency: captions mention `matrixmate.com` vs `metrixmate.com` | Caption agent prompts differ across routes |
| Potential logic issue: social post nodes reference `Generate Caption (Expert)` for all routes | Affects System/Activity publishing correctness |

If you want, I can also propose a corrected “publishing” design so each route posts its own caption (and optionally its own tags/schedule time) without duplicating nodes.