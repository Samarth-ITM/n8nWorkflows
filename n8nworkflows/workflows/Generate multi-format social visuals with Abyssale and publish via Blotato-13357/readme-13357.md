Generate multi-format social visuals with Abyssale and publish via Blotato

https://n8nworkflows.xyz/workflows/generate-multi-format-social-visuals-with-abyssale-and-publish-via-blotato-13357


# Generate multi-format social visuals with Abyssale and publish via Blotato

## 1. Workflow Overview

**Workflow name:** Automate multi-format design creation in Abyssale and publish via Blotato  
**Stated title (user):** Generate multi-format social visuals with Abyssale and publish via Blotato

**Purpose:**  
This workflow automates a full “social campaign asset pipeline”:
1) a user sends a product image + caption via Telegram,  
2) AI analyzes the image and generates a strict prompt,  
3) AtlasCloud (NanoBanana) edits/creates a new product image,  
4) AtlasCloud background remover produces a transparent PNG,  
5) Abyssale generates multiple social formats from a design template (dynamic form-driven editing),  
6) Blotato uploads each asset and publishes it to the matching platform.

**Target use cases:**
- Product marketing teams generating consistent multi-format creatives from one source image and caption.
- Agencies producing client social assets at scale from an Abyssale template.
- Automated “edit + cutout + resize + publish” pipelines.

### 1.1 Input Reception (Telegram)
Receives an image from Telegram, downloads it, and converts it to a public URL usable by external AI services.

### 1.2 AI Image Understanding + Prompt Engineering
OpenAI Vision analyzes the reference image; a LangChain agent produces a constrained JSON prompt for NanoBanana.

### 1.3 AI Image Generation + Polling (AtlasCloud NanoBanana)
Submits an asynchronous image generation request, polls until completed, and extracts the generated image URL.

### 1.4 Background Removal + Polling (AtlasCloud)
Runs a second async job to remove background, polls until completed, downloads the final PNG, and replies on Telegram.

### 1.5 Abyssale Template Intake + Dynamic Form
A user submits a Template ID via an n8n Form Trigger; the workflow pulls the Abyssale design, extracts editable layers, and dynamically generates an n8n Form to collect replacements (text/colors/files) plus a caption.

### 1.6 Abyssale Multi-format Generation + Blotato Publishing
Builds Abyssale element payloads per format, generates images for all formats, routes each format to a matching Blotato “upload media” + “create post” path.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram image intake and public hosting
**Overview:** Receives a Telegram photo message, downloads the file, uploads it to tmpfiles.org, and produces a public direct-download URL.

**Nodes involved:**  
- Telegram Trigger  
- Get a file  
- Upload Image to tmpfiles  
- Extract Public URL  
- Prepare AI Agent Input

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — entry point listening for Telegram messages.
- **Configuration (interpreted):**
  - Updates: `message`
  - Additional: `download: true` (ensures media is downloaded and attached as binary to the execution).
- **Key data used:**
  - `message.photo[]` (photo sizes)
  - `message.caption` (user description)
  - `message.chat.id`, `message.message_id`
- **Outputs:** Passes message JSON + downloaded binary (photo) to “Get a file”.
- **Potential failures:**
  - Missing `message.photo` if the user sends text-only or non-photo content.
  - Missing `caption` if user doesn’t add a caption (later used as description).
  - Telegram credential/auth issues.

#### Node: Get a file
- **Type / role:** `telegram` (resource: file) — retrieves Telegram file metadata/binary via file_id.
- **Configuration:**
  - `fileId` expression selects the highest-resolution photo:  
    `{{$json.message.photo[$json.message.photo.length - 1].file_id}}`
- **Input:** Telegram Trigger output.
- **Output:** File data suitable for upload; flows to tmpfiles upload.
- **Potential failures:**
  - Expression fails if `message.photo` is undefined/empty.
  - Telegram API rate limits or invalid file_id.

#### Node: Upload Image to tmpfiles
- **Type / role:** `httpRequest` — uploads the binary file to tmpfiles.org to obtain a public URL.
- **Configuration:**
  - POST `https://tmpfiles.org/api/v1/upload`
  - multipart/form-data body: field `file` from binary input field named `data`.
  - Response format: JSON.
- **Input:** File binary from “Get a file”.
- **Output:** JSON containing `data.url` (tmpfiles link).
- **Potential failures:**
  - Binary field name mismatch (expects `data`).
  - tmpfiles downtime/limits; large file rejection.

#### Node: Extract Public URL
- **Type / role:** `code` — converts tmpfiles page URL into direct download URL.
- **Logic:**
  - Reads `response.data.url`
  - Replaces `tmpfiles.org/` with `tmpfiles.org/dl/`
  - Adds `publicImageUrl` to JSON.
- **Output:** Same response + `publicImageUrl` used downstream.
- **Potential failures:**
  - tmpfiles API response shape changes.
  - If `data.url` missing, `replace()` throws.

#### Node: Prepare AI Agent Input
- **Type / role:** `set` — normalizes fields needed for AI.
- **Assignments:**
  - `image_url` = `{{$json.publicImageUrl}}`
  - `description` = `{{ $('Telegram Trigger').item.json.message.caption }}`
  - `chatId` = `{{ $('Telegram Trigger').item.json.message.chat.id }}`
  - `messageId` = `{{ $('Telegram Trigger').item.json.message.message_id }}`
- **Output:** Single item with these keys.
- **Potential failures / edge cases:**
  - `caption` can be null → downstream prompt quality issues.
  - Cross-node reference uses `$('Telegram Trigger').item` which assumes same execution item alignment.

---

### Block 2 — Reference image analysis (OpenAI Vision)
**Overview:** Uses OpenAI Vision to produce a structured YAML description of the reference image (product/character details).

**Nodes involved:**  
- OpenAI Vision: Analyze Reference Image

#### Node: OpenAI Vision: Analyze Reference Image
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (resource: image, operation: analyze).
- **Configuration:**
  - Model: `chatgpt-4o-latest`
  - `imageUrls` = `{{ $json.image_url }}`
  - Prompt instructs: output **YAML only** with product/character fields and color hex codes.
- **Input:** `Prepare AI Agent Input` output.
- **Output:** A text payload (YAML) in the node’s standard field (commonly `content` in n8n AI nodes).
- **Potential failures:**
  - OpenAI credential/auth issues.
  - Non-image URL or blocked URL retrieval (tmpfiles access).
  - Model returning non-YAML or partial YAML (prompt tries to prevent this, but not guaranteed).

---

### Block 3 — Prompt generation (LangChain agent + parser)
**Overview:** Combines the user caption and the vision analysis to produce a strict JSON object containing `image_prompt` for NanoBanana.

**Nodes involved:**  
- OpenAI Chat Model  
- Structured Output Parser  
- Generate NanoBanana Prompt

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — language model provider for the agent.
- **Configuration:**
  - Model: `gpt-5.2`
- **Connections:** Connected to agent via `ai_languageModel`.
- **Potential failures:**
  - Model availability / account permissions.
  - Higher latency/cost.

#### Node: Structured Output Parser
- **Type / role:** `outputParserStructured` — enforces JSON schema output.
- **Schema:**
  - Object with property `image_prompt` (string).
- **Connections:** Connected to agent via `ai_outputParser`.
- **Potential failures:**
  - If the model output can’t be parsed into the schema, the node/agent run fails.

#### Node: Generate NanoBanana Prompt
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — prompt-engineering agent.
- **Configuration highlights:**
  - System message is a long “product prompt engineer” contract:
    - photorealistic, studio lighting, **transparent background**, isolated cutout
    - “single product only” rule unless explicitly requested otherwise
    - strict preservation of product structure/colors/branding
    - output must be **JSON only** with key `image_prompt`, max 120 words
  - User message merges:
    - User description from Telegram caption
    - Reference image description from vision analysis (`{{ $json.content }}` assumes prior node outputs `content`)
- **Input:** Output of OpenAI Vision node.
- **Output:** Parsed structured JSON accessible as `$json.output.image_prompt` (as used later).
- **Potential failures / edge cases:**
  - If OpenAI Vision output field name isn’t `content`, the prompt will be missing reference description.
  - Strict schema parsing can fail on any deviation (extra keys, prose).

---

### Block 4 — NanoBanana image generation (AtlasCloud) + polling
**Overview:** Calls AtlasCloud NanoBanana edit model asynchronously, then polls until the job completes and extracts the generated image URL.

**Nodes involved:**  
- Create NanoBanana Image  
- Extract Prediction ID  
- Wait Before Polling  
- Check Image Status  
- Check If Completed  
- Extract Final Image URL

#### Node: Create NanoBanana Image
- **Type / role:** `httpRequest` — submits generation request.
- **Configuration:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
  - JSON body includes:
    - `model: "google/nano-banana-pro/edit"`
    - `images: [ $("Prepare AI Agent Input").item.json.image_url ]` (reference image)
    - `prompt: $json.output.image_prompt` (from agent output)
    - `output_format: "png"`, `aspect_ratio: "1:1"`, `resolution: "1k"`
    - async: `enable_sync_mode: false`
  - Header: `Authorization: Bearer <__PLACEHOLDER_VALUE__AtlasCloud API Key__>`
- **Input:** Output from “Generate NanoBanana Prompt”.
- **Output:** AtlasCloud response with a `data.id` prediction/job id.
- **Potential failures:**
  - Missing/invalid AtlasCloud API key placeholder.
  - If `$json.output.image_prompt` doesn’t exist (agent failure), request contains null prompt.
  - Model name changes or account access restrictions.

#### Node: Extract Prediction ID
- **Type / role:** `code` — captures job id + Telegram routing data.
- **Logic:**
  - `predictionId = response.data.id`
  - carries `chatId`, `messageId` from “Prepare AI Agent Input”
- **Output:** `{ predictionId, chatId, messageId }`
- **Potential failures:** response shape changes or missing `data.id`.

#### Node: Wait Before Polling
- **Type / role:** `wait` — pauses before polling job status.
- **Configuration:** No explicit wait time configured (uses node defaults / manual resume style depending on n8n version/settings).
- **Connections:** Loops into “Check Image Status”.
- **Risk:** If not configured as a timed wait, it may not delay as intended (or may require webhook resume). Consider setting a fixed delay (e.g., 3–10 seconds).

#### Node: Check Image Status
- **Type / role:** `httpRequest` — polls prediction status.
- **Configuration:**
  - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{predictionId}}`
  - Auth header with AtlasCloud key.
- **Output:** Response containing `data.status` and outputs when ready.
- **Potential failures:** rate limits (polling too frequently), expired job id.

#### Node: Check If Completed
- **Type / role:** `if` — branches based on completion.
- **Condition:** `{{$json.data.status}} equals "completed"`
- **True path:** to “Extract Final Image URL”
- **False path:** back to “Wait Before Polling” (poll loop)
- **Edge cases:** statuses like `failed`, `canceled`, `queued` are not explicitly handled (will loop forever unless n8n execution times out).

#### Node: Extract Final Image URL
- **Type / role:** `code` — extracts first output image URL.
- **Logic:** `item.json?.data?.outputs?.[0] || null`
- **Output:** `{ imageUrl }`
- **Edge cases:** if outputs array empty, `imageUrl` becomes null; next node will fail.

---

### Block 5 — Background removal (AtlasCloud) + polling + Telegram reply
**Overview:** Uses AtlasCloud background remover asynchronously, polls until complete, downloads the transparent PNG, and replies to the original Telegram message with the final image.

**Nodes involved:**  
- image background remover  
- Extract Prediction ID1  
- Wait Before Polling1  
- Check Image Status1  
- Check If Completed1  
- Extract Final Image URL1  
- Download Final Image  
- Send Photo to Telegram

#### Node: image background remover
- **Type / role:** `httpRequest` — submits background removal job.
- **Configuration:**
  - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
  - Body:
    - `model: "atlascloud/image-background-remover"`
    - `image: $json.imageUrl` (from previous block)
    - async: `enable_sync_mode: false`
- **Output:** Job id in `data.id`.
- **Potential failures:** null `imageUrl`, invalid API key, model endpoint changes.

#### Node: Extract Prediction ID1
- **Type / role:** `code` — same logic as earlier: captures `data.id`, plus Telegram `chatId/messageId`.
- **Note:** Relies on `Prepare AI Agent Input` item access again.

#### Node: Wait Before Polling1
- **Type / role:** `wait` — second polling loop wait node.
- **Configuration:** also not explicitly timed.

#### Node: Check Image Status1
- **Type / role:** `httpRequest` — polls background removal job.
- **URL:** `.../prediction/{{predictionId}}` (uses Extract Prediction ID1)
- **Output:** status + outputs.

#### Node: Check If Completed1
- **Type / role:** `if`
- **Condition:** `data.status == "completed"`
- **False path:** loops back to Wait Before Polling1
- **Missing failure handling:** `failed` not handled.

#### Node: Extract Final Image URL1
- **Type / role:** `code` — extracts `outputs[0]` as `imageUrl`.
- **Output:** `{ imageUrl }`

#### Node: Download Final Image
- **Type / role:** `httpRequest` — downloads the final image as binary.
- **Configuration:** `responseFormat: file`
- **Input:** `imageUrl`
- **Output:** Binary file to send to Telegram.
- **Potential failures:** expired CDN URL, large file, non-200 responses.

#### Node: Send Photo to Telegram
- **Type / role:** `telegram` — sends the generated image back to the user.
- **Configuration:**
  - Operation: `sendPhoto`
  - `chatId` from “Prepare AI Agent Input”
  - `reply_to_message_id` set to original Telegram message id
  - `binaryData: true` (sends downloaded binary)
- **Potential failures:** Telegram size limits, invalid chatId/messageId, missing binary property.

---

### Block 6 — Abyssale template selection (Form Trigger) and design introspection
**Overview:** A separate entry point: user submits an Abyssale Template ID. The workflow fetches the design and extracts editable elements into a normalized list.

**Nodes involved:**  
- On form submission  
- Get a design  
- Extract Layers

#### Node: On form submission
- **Type / role:** `formTrigger` — second entry point.
- **Configuration:**
  - Form title: “Abyssale Generator”
  - One field: “Template ID”
- **Output:** `{ "Template ID": "..." }`

#### Node: Get a design
- **Type / role:** `n8n-nodes-abyssale.abyssale` — reads Abyssale design by ID.
- **Configuration:**
  - Resource: `design`
  - `designId` = `{{ $json['Template ID'] }}`
- **Credentials:** Abyssale API credential.
- **Output:** Design object containing `template_id`, `formats`, `elements`, etc.
- **Potential failures:** invalid template id, auth issues, template not accessible.

#### Node: Extract Layers
- **Type / role:** `code` — converts Abyssale design elements into a simpler list for form building.
- **Logic:**
  - For each `design.elements`, takes first attribute (`attributes[0]`) and uses its `values['facebook-post']` as “currentValue”.
  - Outputs `{ templateId: design.template_id, fields: [...] }`
- **Edge cases:**
  - Assumes every element has `attributes[0]` and includes `values['facebook-post']`.
  - If the template lacks a `facebook-post` format, placeholders/defaults may be wrong or undefined.

---

### Block 7 — Dynamic form generation and user-provided replacements
**Overview:** Builds a dynamic n8n Form definition from the extracted Abyssale layers; users can edit text, colors, and upload replacement images/logos; also collects a caption for publishing.

**Nodes involved:**  
- Build Form Fields  
- Form  
- Convert Uploads to Base64

#### Node: Build Form Fields
- **Type / role:** `code` — generates JSON form schema for the “Form” node.
- **Key behaviors:**
  - For image/logo fields:
    - Adds an HTML preview block showing current image URL
    - Adds a file upload field (accepts png/jpg/jpeg/svg/webp)
  - For text fields:
    - Adds text input with default/placeholder stripped of HTML
  - For container fields:
    - Adds text input keyed as `${name} (${attributeId})`
    - Special-case `background_color`: validates hex (`#RRGGBB` or `#RRGGBBAA`) else uses default
  - Adds an extra field: **“caption for social media”**
- **Output:** `{ templateId, formFields, totalFields }`
- **Potential failures:** unexpected element types, missing `currentValue`.

#### Node: Form
- **Type / role:** `form` — presents the generated form and returns submitted values (and binaries).
- **Configuration:**
  - `defineForm: json`
  - `jsonOutput: {{ $json.formFields }}`
  - Custom CSS theme (dark background + styling)
- **Output:** JSON of user entries + binary uploads.
- **Potential failures:** large file uploads, user submits empty fields (handled by defaults later).

#### Node: Convert Uploads to Base64
- **Type / role:** `code` — converts any uploaded binaries into base64 data URIs for Abyssale payload.
- **Logic:**
  - Iterates over `$input.first().binary` keys
  - Uses `helpers.getBinaryDataBuffer` to base64 encode
  - Produces `base64Images` map keyed by binary field name
- **Edge cases:** memory usage for large images; no binary present is fine.

---

### Block 8 — Build Abyssale payload(s) and generate images per format
**Overview:** Merges design defaults + user overrides into an Abyssale “elements” payload, then generates one output per format in the template.

**Nodes involved:**  
- Build Abyssale Elements  
- Generate Images  
- Switch

#### Node: Build Abyssale Elements
- **Type / role:** `code` — constructs request objects for each Abyssale format.
- **Inputs:**
  - Submitted form JSON + `base64Images`
  - Design object fetched from “Get a design” via cross-node reference.
- **Key logic:**
  - Uses `design.elements[*].attributes[0].values['facebook-post']` as default for all formats.
  - For each element:
    - `container`: reads user input `${name} (${attr.id})`; validates `background_color` hex.
    - `text`: `payload` from user input or default.
    - `image/logo`: if user uploaded, uses `image_encoded` (raw base64 without data URI prefix); else uses `image_url` default.
  - Returns one item per format with:
    - `url: https://api.abyssale.com/banner-builder/{templateId}/generate`
    - `body: { template_format_name: f.id, elements }`
    - `formatName: "{id} ({w}x{h})"`
- **Potential failures / edge cases:**
  - Assumes the binary field naming convention: `${name}__upload_new_` (must match Form’s generated upload field internal key).
  - If Abyssale expects different keys for encoded images or element naming mismatches, generation fails.
  - Default values always taken from `facebook-post` even for other formats; may be acceptable but can be inaccurate if formats differ.

#### Node: Generate Images
- **Type / role:** `httpRequest` — calls Abyssale generate endpoint for each format item.
- **Configuration:**
  - POST to `{{$json.url}}`
  - JSON body `{{$json.body}}`
  - Authentication: predefined credential type `AbyssaleApi`
- **Output:** Abyssale response typically includes generated image file URL(s) and format metadata (the workflow expects `file.cdn_url` and `format.id` later).
- **Potential failures:** Abyssale API errors (invalid template_format_name, element payload invalid, auth).

#### Node: Switch
- **Type / role:** `switch` — routes each generated format to the correct Blotato publishing path.
- **Rules (by `{{$json.format.id}}`):**
  - facebook-post
  - facebook-reel
  - **Potential issue:** one rule checks `rightValue: "=instagram-post"` (note the leading `=`). This likely prevents matching `instagram-post`.
  - instagram-story
  - tiktok-post
  - twitter-post
  - linkedin-feed
  - pinterest-pins
- **Outputs:** 8 branches to “Upload media”, “Upload media1”, … “Upload media7”.
- **Potential failures:** if `format.id` not present or mismatched, items won’t route to any publishing node.

---

### Block 9 — Blotato media upload and publishing per platform
**Overview:** For each format, uploads the generated image to Blotato as media, then creates a platform-specific post with the caption from the dynamic form.

**Nodes involved (8 parallel paths):**  
- Upload media → Create facebook-post  
- Upload media1 → Create facebook-reel  
- Upload media2 → Create instagram-post  
- Upload media3 → Create instagram-story  
- Upload media4 → Create tiktok-post  
- Upload media5 → Create twitter-post  
- Upload media6 → Create linkedin-feed  
- Upload media7 → Create pinterest-pins

#### Node: Upload media (and Upload media1..7)
- **Type / role:** `@blotato/n8n-nodes-blotato.blotato` (resource: media) — uploads media by URL.
- **Configuration:** `mediaUrl = {{ $json.file.cdn_url }}`
- **Input:** Abyssale generation response for that format.
- **Output:** Blotato media object, including a URL used by “Create …” nodes (these nodes use `{{$json.url}}`).
- **Potential failures:** invalid `file.cdn_url`, Blotato auth, unsupported URL access from Blotato.

#### Node: Create facebook-post
- **Type / role:** Blotato post creation.
- **Configuration:**
  - Platform: facebook
  - AccountId: 1759 (“Firass Ben”)
  - Facebook Page: 101603614680195 (“Dr. Firas”)
  - Text: `{{ $('Form').item.json['caption for social media'] }}`
  - Media URLs: `{{ $json.url }}`
- **Potential failures:** missing permissions for page posting, invalid media URL, caption empty.

#### Node: Create facebook-reel
- **Configuration differences:**
  - Options: `facebookMediaType: reel`
  - Same caption + media expression as above.

#### Node: Create instagram-post
- **Platform behavior:** uses Blotato defaults for Instagram feed post (platform not explicitly set in parameters here; relies on node defaults for Instagram when account is Instagram-capable).
- **Caption:** from Form.
- **Media:** `{{$json.url}}`
- **Important upstream note:** Switch rule typo may prevent reaching this node.

#### Node: Create instagram-story
- **Options:** `instagramMediaType: reel` (this is unusual naming: “story” node configured as reel; confirm Blotato semantics).
- **Caption/media:** same pattern.

#### Node: Create tiktok-post
- **Platform:** tiktok
- **AccountId:** 2079 (“elitecybzcs”)

#### Node: Create twitter-post
- **Platform:** twitter

#### Node: Create linkedin-feed
- **Platform:** linkedin
- **AccountId:** 1446 (“Samuel Amalric”)

#### Node: Create pinterest-pins
- **Platform:** pinterest
- **Board:** 1146658823815436667

**Shared edge cases across Create nodes:**
- `$('Form').item...` references assume the Form execution item context remains accessible; if multiple concurrent executions occur, keep an eye on item linking.
- Different platforms have different media aspect constraints; ensure Abyssale formats match platform requirements.
- API failures: rate limits, account disconnected, missing required fields (e.g., Pinterest may require title/description depending on Blotato settings).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry: receive Telegram photo message | — | Get a file | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Get a file | telegram (file) | Fetch Telegram file/binary | Telegram Trigger | Upload Image to tmpfiles | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Upload Image to tmpfiles | httpRequest | Upload binary to tmpfiles.org | Get a file | Extract Public URL | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Extract Public URL | code | Convert tmpfiles URL to direct public URL | Upload Image to tmpfiles | Prepare AI Agent Input | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Prepare AI Agent Input | set | Normalize fields for AI + keep chat routing info | Extract Public URL | OpenAI Vision: Analyze Reference Image | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| OpenAI Vision: Analyze Reference Image | langchain OpenAI (image analyze) | Analyze reference image to YAML | Prepare AI Agent Input | Generate NanoBanana Prompt | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for agent | — | Generate NanoBanana Prompt (ai_languageModel) | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Structured Output Parser | outputParserStructured | Enforce JSON schema `{image_prompt}` | — | Generate NanoBanana Prompt (ai_outputParser) | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Generate NanoBanana Prompt | langchain agent | Produce strict NanoBanana prompt JSON | OpenAI Vision: Analyze Reference Image | Create NanoBanana Image | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Create NanoBanana Image | httpRequest | Submit AtlasCloud NanoBanana edit job | Generate NanoBanana Prompt | Extract Prediction ID | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| Extract Prediction ID | code | Capture prediction ID for polling | Create NanoBanana Image | Wait Before Polling | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| Wait Before Polling | wait | Delay/loop control before polling | Extract Prediction ID / Check If Completed (false) | Check Image Status | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| Check Image Status | httpRequest | Poll AtlasCloud prediction status | Wait Before Polling | Check If Completed | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| Check If Completed | if | Branch when status == completed | Check Image Status | Extract Final Image URL / Wait Before Polling | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| Extract Final Image URL | code | Extract final image URL from outputs | Check If Completed (true) | image background remover | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| image background remover | httpRequest | Submit background removal job | Extract Final Image URL | Extract Prediction ID1 | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Extract Prediction ID1 | code | Capture bg-removal prediction ID | image background remover | Wait Before Polling1 | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Wait Before Polling1 | wait | Delay/loop control before polling | Extract Prediction ID1 / Check If Completed1 (false) | Check Image Status1 | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Check Image Status1 | httpRequest | Poll bg-removal prediction status | Wait Before Polling1 | Check If Completed1 | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Check If Completed1 | if | Branch when status == completed | Check Image Status1 | Extract Final Image URL1 / Wait Before Polling1 | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Extract Final Image URL1 | code | Extract bg-removed image URL | Check If Completed1 (true) | Download Final Image | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Download Final Image | httpRequest | Download final PNG as binary | Extract Final Image URL1 | Send Photo to Telegram | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Send Photo to Telegram | telegram (sendPhoto) | Reply in Telegram with final image | Download Final Image | — | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| On form submission | formTrigger | Entry: user provides Abyssale Template ID | — | Get a design | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Get a design | abyssale | Fetch Abyssale design by ID | On form submission | Extract Layers | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Extract Layers | code | Normalize template elements into fields list | Get a design | Build Form Fields | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Build Form Fields | code | Build dynamic Form schema + caption field | Extract Layers | Form | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Form | form | Display dynamic form; collect text/files/caption | Build Form Fields | Convert Uploads to Base64 | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Convert Uploads to Base64 | code | Convert uploaded files to base64 | Form | Build Abyssale Elements | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Build Abyssale Elements | code | Build Abyssale generate payload per format | Convert Uploads to Base64 | Generate Images | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Generate Images | httpRequest (Abyssale auth) | Generate images for each format | Build Abyssale Elements | Switch | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Switch | switch | Route by `format.id` to platform pipelines | Generate Images | Upload media… (8 branches) | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media | blotato (media) | Upload Facebook Post asset to Blotato | Switch | Create facebook-post | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create facebook-post | blotato (post) | Publish Facebook post | Upload media | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media1 | blotato (media) | Upload Facebook Reel asset to Blotato | Switch | Create facebook-reel | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create facebook-reel | blotato (post) | Publish Facebook reel | Upload media1 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media2 | blotato (media) | Upload Instagram Post asset to Blotato | Switch | Create instagram-post | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create instagram-post | blotato (post) | Publish Instagram post | Upload media2 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media3 | blotato (media) | Upload Instagram Story asset to Blotato | Switch | Create instagram-story | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create instagram-story | blotato (post) | Publish Instagram story/reel (per config) | Upload media3 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media4 | blotato (media) | Upload TikTok asset to Blotato | Switch | Create tiktok-post | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create tiktok-post | blotato (post) | Publish TikTok post | Upload media4 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media5 | blotato (media) | Upload Twitter asset to Blotato | Switch | Create twitter-post | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create twitter-post | blotato (post) | Publish tweet | Upload media5 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media6 | blotato (media) | Upload LinkedIn asset to Blotato | Switch | Create linkedin-feed | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create linkedin-feed | blotato (post) | Publish LinkedIn feed post | Upload media6 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Upload media7 | blotato (media) | Upload Pinterest asset to Blotato | Switch | Create pinterest-pins | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Create pinterest-pins | blotato (post) | Publish Pinterest pin | Upload media7 | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Sticky Note | stickyNote | Comment block | — | — | # 🔹 Step 1: Generate the NanoBanana Image Prompt |
| Sticky Note1 | stickyNote | Comment block | — | — | # 🔹 Step 2: Generate and Retrieve the Edited Product Image |
| Sticky Note2 | stickyNote | Comment block | — | — | # 🔹 Step 3: Remove the Background and Create Transparent Cutout |
| Sticky Note3 | stickyNote | Comment block | — | — | # 🔹 Step 4: Generate Multi-Platform Social Media Designs with **[Abyssale.com](https://abyssale.cello.so/wXzGGKBcrTl)** |
| Sticky Note5 | stickyNote | Workspace notes/links/credits | — | — | # 🚀  Generate social media designs in Abyssale and publish to Blotato  \n## (By Dr. Firas)\n\n@[youtube](T9gNqXRH8nE)\n\n# 📘 Documentation  \n📎 https://automatisation.notion.site/Automate-multi-format-design-creation-in-Abyssale-and-publish-via-Blotato-3053d6550fd98094a291f106966ca092?source=copy_link\n\nLinks mentioned: https://abyssale.cello.so/wXzGGKBcrTl • https://blotato.com/?ref=firas • https://www.atlascloud.ai?ref=8QKPJE |

---

## 4. Reproducing the Workflow from Scratch

### A) Credentials you must create in n8n
1. **Telegram API** credential (bot token)  
   - Name example: “Telegram abyssale”
2. **OpenAI** credential  
   - Name example: “OpenAi account”
3. **Abyssale API** credential  
   - Must be usable by both the Abyssale node and HTTP Request node with “predefined credential type”.
4. **Blotato API** credential  
   - Name example: “Blotato account”
5. **AtlasCloud API key**  
   - This workflow uses a literal header placeholder in HTTP nodes. Store it securely (recommended: n8n credential or environment variable) and reference it via expression.

---

### B) Build the Telegram → AI → Cutout pipeline (Entry point 1)
1. Add **Telegram Trigger**
   - Updates: `message`
   - Additional field: enable **download**
2. Add **Telegram** node “Get a file”
   - Resource: **file**
   - `fileId` expression: pick last photo size `message.photo[...].file_id`
   - Connect: Telegram Trigger → Get a file
3. Add **HTTP Request** “Upload Image to tmpfiles”
   - POST `https://tmpfiles.org/api/v1/upload`
   - Body: multipart-form-data
   - Add body parameter:
     - name: `file`
     - type: binary (form binary data)
     - binary field name: `data`
   - Response: JSON
   - Connect: Get a file → Upload Image to tmpfiles
4. Add **Code** “Extract Public URL”
   - Parse response JSON and create `publicImageUrl` by replacing to `/dl/`
   - Connect: Upload Image to tmpfiles → Extract Public URL
5. Add **Set** “Prepare AI Agent Input”
   - Set:
     - `image_url` = `{{$json.publicImageUrl}}`
     - `description` = `{{ $('Telegram Trigger').item.json.message.caption }}`
     - `chatId` = `{{ $('Telegram Trigger').item.json.message.chat.id }}`
     - `messageId` = `{{ $('Telegram Trigger').item.json.message.message_id }}`
   - Connect: Extract Public URL → Prepare AI Agent Input
6. Add **OpenAI (LangChain)** node “OpenAI Vision: Analyze Reference Image”
   - Resource: Image, Operation: Analyze
   - Model: `chatgpt-4o-latest`
   - `imageUrls` = `{{ $json.image_url }}`
   - Prompt: YAML-only spec (product/character schema)
   - Connect: Prepare AI Agent Input → OpenAI Vision node
7. Add **OpenAI Chat Model** node
   - Model: `gpt-5.2`
8. Add **Structured Output Parser**
   - Manual schema: object with `image_prompt` string
9. Add **LangChain Agent** “Generate NanoBanana Prompt”
   - Prompt type: “define”
   - System message: strict “transparent background cutout, preserve branding, single product” contract
   - User text should reference:
     - Telegram description
     - Vision analysis output (ensure you use the correct field from the Vision node; in this workflow it assumes `$json.content`)
   - Connect:
     - OpenAI Vision node → Agent (main)
     - OpenAI Chat Model → Agent (ai_languageModel)
     - Structured Output Parser → Agent (ai_outputParser)
10. Add **HTTP Request** “Create NanoBanana Image”
   - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
   - Header: `Authorization: Bearer <YOUR_ATLASCLOUD_KEY>`
   - JSON body includes:
     - model `google/nano-banana-pro/edit`
     - images: `[ {{ $('Prepare AI Agent Input').item.json.image_url }} ]`
     - prompt: `{{ $json.output.image_prompt }}`
     - output_format png, aspect_ratio 1:1, resolution 1k
     - enable_sync_mode false
   - Connect: Agent → Create NanoBanana Image
11. Add **Code** “Extract Prediction ID”
   - Output `{predictionId, chatId, messageId}`
   - Connect: Create NanoBanana Image → Extract Prediction ID
12. Add **Wait** “Wait Before Polling”
   - Configure a fixed delay (recommended) like 5–10 seconds to avoid rate limiting.
   - Connect: Extract Prediction ID → Wait Before Polling
13. Add **HTTP Request** “Check Image Status”
   - GET `https://api.atlascloud.ai/api/v1/model/prediction/{{predictionId}}`
   - Same Authorization header
   - Connect: Wait Before Polling → Check Image Status
14. Add **IF** “Check If Completed”
   - Condition: `{{ $json.data.status }}` equals `completed`
   - True → next extraction
   - False → back to Wait Before Polling (poll loop)
   - Connect: Check Image Status → IF, then IF(false) → Wait
15. Add **Code** “Extract Final Image URL”
   - Extract `data.outputs[0]` to `imageUrl`
   - Connect: IF(true) → Extract Final Image URL
16. Add **HTTP Request** “image background remover”
   - POST `https://api.atlascloud.ai/api/v1/model/generateImage`
   - Header Authorization
   - JSON body:
     - model `atlascloud/image-background-remover`
     - image: `{{ $json.imageUrl }}`
     - enable_sync_mode false
   - Connect: Extract Final Image URL → background remover
17. Repeat polling pattern for background remover:
   - **Code** Extract Prediction ID1 → **Wait** → **HTTP** Check Image Status1 → **IF** Check If Completed1 → loop
18. Add **Code** “Extract Final Image URL1”
   - Extract outputs[0] to `imageUrl`
19. Add **HTTP Request** “Download Final Image”
   - GET `{{ $json.imageUrl }}`
   - Response format: file/binary
20. Add **Telegram** node “Send Photo to Telegram”
   - Operation: sendPhoto
   - chatId: `{{ $('Prepare AI Agent Input').item.json.chatId }}`
   - reply_to_message_id: `{{ $('Prepare AI Agent Input').item.json.messageId }}`
   - binaryData: true
   - Connect: Download Final Image → Send Photo to Telegram

---

### C) Build the Abyssale → Blotato pipeline (Entry point 2)
1. Add **Form Trigger** “On form submission”
   - Title: “Abyssale Generator”
   - Field: “Template ID”
2. Add **Abyssale** node “Get a design”
   - Resource: design
   - designId: `{{ $json['Template ID'] }}`
   - Connect: Form Trigger → Get a design
3. Add **Code** “Extract Layers”
   - Map `design.elements` into `{name,type,attribute,currentValue}` using `values['facebook-post']`
   - Output `templateId` + `fields`
4. Add **Code** “Build Form Fields”
   - Build JSON form schema:
     - preview HTML + upload field for image/logo
     - text fields for text/container
     - add “caption for social media”
5. Add **Form** node
   - Define form from JSON: `jsonOutput = {{ $json.formFields }}`
   - Optional: apply custom CSS theme
6. Add **Code** “Convert Uploads to Base64”
   - Convert all binary uploads to base64 data URIs in `base64Images`
7. Add **Code** “Build Abyssale Elements”
   - Cross-reference the design from “Get a design”
   - Build `elements` object:
     - images/logos: choose `image_encoded` if uploaded else default `image_url`
     - text: `payload`
     - container: attribute keyed fields; validate background_color hex
   - Output one item per format with `url` + `body` including `template_format_name`
8. Add **HTTP Request** “Generate Images”
   - POST each `{{$json.url}}`
   - Body: `{{$json.body}}`
   - Authentication: Abyssale predefined credential
9. Add **Switch**
   - Value to test: `{{ $json.format.id }}`
   - Rules for: facebook-post, facebook-reel, instagram-post, instagram-story, tiktok-post, twitter-post, linkedin-feed, pinterest-pins
   - **Fix recommended:** ensure the instagram-post rule compares to `instagram-post` (no leading `=`).
10. For each Switch output, add:
   - **Blotato** node “Upload mediaX” (resource: media), `mediaUrl = {{ $json.file.cdn_url }}`
   - Then **Blotato** node “Create …” with:
     - `postContentText = {{ $('Form').item.json['caption for social media'] }}`
     - `postContentMediaUrls = {{ $json.url }}` (from Upload media output)
     - Configure platform/account/page/board options as needed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate social media designs in Abyssale and publish to Blotato (By Dr. Firas) | Sticky note workspace header |
| YouTube reference: `@[youtube](T9gNqXRH8nE)` | Mentioned in sticky note (video id: `T9gNqXRH8nE`) |
| Notion documentation link | https://automatisation.notion.site/Automate-multi-format-design-creation-in-Abyssale-and-publish-via-Blotato-3053d6550fd98094a291f106966ca092?source=copy_link |
| Abyssale referral link | https://abyssale.cello.so/wXzGGKBcrTl |
| Blotato referral link | https://blotato.com/?ref=firas |
| AtlasCloud referral link | https://www.atlascloud.ai?ref=8QKPJE |
| Operational caution: polling loops don’t handle `failed` status | Add explicit handling to avoid infinite loops/timeouts |
| Operational caution: Wait nodes have no explicit delay configured | Set a delay (e.g., 5–10s) to avoid tight polling and rate limits |

Disclaimer (from user): Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.