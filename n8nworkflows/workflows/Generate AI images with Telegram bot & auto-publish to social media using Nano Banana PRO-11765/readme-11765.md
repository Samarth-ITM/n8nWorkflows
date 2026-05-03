Generate AI images with Telegram bot & auto-publish to social media using Nano Banana PRO

https://n8nworkflows.xyz/workflows/generate-ai-images-with-telegram-bot---auto-publish-to-social-media-using-nano-banana-pro-11765


# Generate AI images with Telegram bot & auto-publish to social media using Nano Banana PRO

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow turns a Telegram bot into an AI image generator using **Nano Banana PRO** (via **Kie.ai**). Users can generate images in three modes (text-to-image, image-to-image, multi-image fusion). After generation, the workflow sends the image back to Telegram and can optionally prepare/publish social media posts (Facebook/Instagram/X) via **Blotato**, with optional image compression via **TinyPNG**.

**Target use cases:**
- Product “photoshoot” style image generation from text prompts
- Lifestyle transformation of an uploaded product photo (image-to-image)
- Fusion/collage-like generation from multiple uploaded images
- Optional post caption enhancement + automated publishing

### Logical blocks
1.1 **Entry & Command Routing (Telegram)**  
Receives Telegram messages and routes based on command.

1.2 **Input Collection (Forms in Telegram)**  
Collects prompt + options (aspect ratio, quality) and optionally images.

1.3 **Prompt Enhancement (Optional, OpenAI)**  
If user chooses “Enhance Prompt”, an AI agent rewrites the prompt.

1.4 **Image Prep per Mode (Code)**  
Builds the Kie.ai request body for Nano Banana PRO, including image URLs (Cloudinary for uploaded images).

1.5 **Image Generation Job Orchestration (Kie.ai)**  
Creates generation job, polls status, retrieves output URL, downloads binary, returns to Telegram.

1.6 **Social Sharing Preparation (Optional)**  
Asks user whether to share; processes image optionally (TinyPNG); generates platform captions with OpenAI.

1.7 **Publishing to Social Platforms (Blotato)**  
Publishes to selected platforms (Facebook, Instagram, X).

---

## 2. Block-by-Block Analysis

### 2.1 Entry & Command Routing (Telegram)

**Overview:**  
Listens for Telegram messages, checks the command text, and routes to the appropriate mode (text-to-image, image-to-image, multi-image). Unmatched messages show a menu.

**Nodes involved:**
- Telegram Trigger
- Switch
- Show Menu

#### Node: **Telegram Trigger**
- **Type/role:** `telegramTrigger` — workflow entry via Telegram webhook updates.
- **Config:** listens to `message` updates.
- **Outputs:** to **Switch**.
- **Credentials:** Telegram API credential required (bot token).
- **Failure/edge cases:** webhook misconfiguration, bot privacy mode restricting messages, Telegram downtime.

#### Node: **Switch**
- **Type/role:** `switch` — routes by exact command string.
- **Rules:**  
  - Output 0: `$json.message.text == "/text_to_image"`  
  - Output 1: `$json.message.text == "/image_to_image"`  
  - Output 2: `$json.message.text == "/multi_image"`  
  - Fallback: `extra` → menu
- **Outputs:**  
  - to **Ask for Text Prompt** / **Ask for Image Upload** / **Ask for Multi Images** / **Show Menu**
- **Edge cases:** message has no `.text` (e.g., user sends sticker/photo); in that case it will fall back to menu.

#### Node: **Show Menu**
- **Type/role:** `telegram` — sends a menu message.
- **Config highlights:**  
  - `chatId = $('Telegram Trigger').item.json.message.from.id`
  - Removes keyboard (`replyKeyboardRemove`)
- **Edge cases:** user blocked bot; invalid chatId expression if trigger payload shape differs.

**Sticky note covering this block:**  
“## Entry & Command Routing …”

---

### 2.2 Text-to-Image Mode (Prompt Collection + Optional Enhancement)

**Overview:**  
Collects a text prompt and generation parameters. If “Enhance Prompt” is checked, an AI agent rewrites the prompt before image generation.

**Nodes involved:**
- Ask for Text Prompt
- If
- AI Agent
- OpenAI Chat Model
- ENHANCED TEXT PREP
- STANDARD TEXT PREP

#### Node: **Ask for Text Prompt**
- **Type/role:** `telegram` (`sendAndWait`) — interactive form collection.
- **Form fields:** Image Description (required), Enhance Prompt (checkbox), Aspect Ratio (required dropdown), Quality (required dropdown).
- **Key expressions:** `chatId = $('Telegram Trigger').item.json.message.from.id`
- **Output:** to **If**
- **Edge cases:** user abandons form; Telegram timeouts; form field keys must match downstream code exactly.

#### Node: **If**
- **Type/role:** `if` — checks whether enhancement was selected.
- **Condition:** `$json.data["Enhance Prompt"].length == "1"`  
  (Checkbox returns an array; length 1 means checked.)
- **True branch:** to **AI Agent**
- **False branch:** to **STANDARD TEXT PREP**
- **Edge cases:** if “Enhance Prompt” missing, `.length` may error; here type validation is “loose” but still may fail depending on runtime payload.

#### Node: **AI Agent**
- **Type/role:** LangChain agent — prompt engineer for Nano Banana PRO.
- **Input text template:** includes user prompt + aspect ratio + quality from `$json.data`.
- **System message:** very detailed prompt-engineering rules; outputs only enhanced prompt.
- **Connections:** uses **OpenAI Chat Model** as language model.
- **Edge cases:** token limits for long prompts; agent iteration limit (maxIterations=10); OpenAI rate limits/timeouts.

#### Node: **OpenAI Chat Model**
- **Type/role:** OpenAI chat model for the AI agent.
- **Config:** model `gpt-5`, timeout 180s, built-in web search enabled (high context).
- **Credentials:** OpenAI API key.
- **Edge cases:** model availability, org policy restrictions, network timeouts.

#### Node: **ENHANCED TEXT PREP**
- **Type/role:** `code` — builds sanitized Kie.ai request body from enhanced prompt.
- **Key logic:**  
  - Reads `$('AI Agent').first().json.output`  
  - Reads aspect ratio & quality from `$('Ask for Text Prompt').first().json[...]`
  - Sanitizes punctuation/quotes/newlines and escapes quotes.
  - Outputs `{ json: { body: JSON.stringify(body) } }`
- **Output:** to **HTTP Request** (job creation).
- **Edge cases:** if AI Agent output missing; if form node output path differs; sanitization may double-escape quotes depending on input.

#### Node: **STANDARD TEXT PREP**
- **Type/role:** `code` — same as enhanced prep but uses original prompt from the form.
- **Key logic:** reads from `$('If').first().json.data[...]`
- **Output:** to **HTTP Request**
- **Edge cases:** relies on `If` node payload shape (must contain `.data`).

**Sticky note covering this block:**  
“## Text-to-Image …”

---

### 2.3 Image-to-Image Mode (Upload → Cloudinary → Prep)

**Overview:**  
Collects an uploaded product image and a scene description. Uploads the image to Cloudinary to obtain a public URL, then constructs a Nano Banana PRO request referencing that URL.

**Nodes involved:**
- Ask for Image Upload
- Upload Image (Cloudinary)
- IMAGE-TO-IMAGE PREP
- Image upload failed (Cloudinary) image-image

#### Node: **Ask for Image Upload**
- **Type/role:** `telegram` (`sendAndWait`) — form with file upload.
- **Form fields:** Upload Product Image (required file), Image Description (required), Aspect Ratio, Quality.
- **Output:** to **Upload Image**
- **Edge cases:** file too large; unsupported types; Telegram file retrieval issues.

#### Node: **Upload Image**
- **Type/role:** Cloudinary upload — hosts uploaded file, returns `secure_url`.
- **Config:** file field key `Upload_Product_Image` (must match Telegram binary key mapping).
- **On error:** `continueErrorOutput` and routes to error message node.
- **Outputs:**  
  - Success → **IMAGE-TO-IMAGE PREP**  
  - Error → **Image upload failed (Cloudinary) image-image**
- **Edge cases:** wrong Cloudinary credentials; quota exceeded; binary key mismatch.

#### Node: **IMAGE-TO-IMAGE PREP**
- **Type/role:** `code` — builds request body using Cloudinary `secure_url`.
- **Key logic:**  
  - Reads `$('Ask for Image Upload').first().json` (tries `.data` or direct)  
  - Gets image URL from `$('Upload Image').first().json.secure_url`
  - Builds `image_input: [imageUrl]`
- **Output:** to **HTTP Request**
- **Edge cases:** if Cloudinary output missing; if form data keys differ; aspect ratio/quality default fallback is used.

#### Node: **Image upload failed (Cloudinary) image-image**
- **Type/role:** `telegram` — error notification.
- **Trigger:** Cloudinary node error output.
- **Edge cases:** none (best-effort messaging).

**Sticky note covering this block:**  
“## Image-to-Image …”

---

### 2.4 Multi-Image Fusion Mode (Split → Upload each → Collect URLs → Prep)

**Overview:**  
Collects 2–8 images, splits them into individual items, uploads each to Cloudinary, then aggregates the uploaded URLs into a single Nano Banana PRO request.

**Nodes involved:**
- Ask for Multi Images
- MULTI-IMAGE SPLITTER
- Loop Over Items (Split in Batches)
- Upload an asset from file data (Cloudinary)
- MULTI-IMAGE PREP
- Image upload failed (Cloudinary) multiple-images

#### Node: **Ask for Multi Images**
- **Type/role:** `telegram` (`sendAndWait`) — multi-file form.
- **Form fields:** Upload Images (multiple), Image Description, Aspect Ratio, Quality.
- **Output:** to **MULTI-IMAGE SPLITTER**
- **Edge cases:** Telegram multi-file constraints; users upload fewer than required; mismatched key names.

#### Node: **MULTI-IMAGE SPLITTER**
- **Type/role:** `code` — converts one form submission into N items each containing one binary under key `image`.
- **Key logic:**  
  - Expects binaries named `Upload_Images_0`, `Upload_Images_1`, …  
  - Produces items with `binary: { image: fileBinary }`
  - Note: It writes `fusionDescription: data['Fusion Description']` but the form label is **“Image Description”**. This means `fusionDescription` will likely be `undefined` (bug), though it is not used later.
- **Output:** to **Loop Over Items**
- **Edge cases:** binary keys may differ depending on n8n Telegram node behavior; label mismatch; missing binary leads to silently skipped images.

#### Node: **Loop Over Items** (`splitInBatches`)
- **Type/role:** batching/loop control.
- **Connections:**  
  - Main output 0 (“Done”) → **MULTI-IMAGE PREP**  
  - Output 1 (“Loop”) → **Upload an asset from file data**
- **Behavior:** iterates through split items, uploading each, then finishes.
- **Edge cases:** if batching not configured (batch size defaults), may process all at once; large image sets can slow execution.

#### Node: **Upload an asset from file data**
- **Type/role:** Cloudinary upload for each batch item.
- **Config:** expects binary key `image`.
- **On error:** continue and route to Telegram error node.
- **Notes (sticky on node):** “❌ Image upload failed (Cloudinary)”
- **Outputs:**  
  - Success → back to **Loop Over Items**  
  - Error → **Image upload failed (Cloudinary) multiple-images**

#### Node: **MULTI-IMAGE PREP**
- **Type/role:** `code` — aggregates uploaded `secure_url` values into `image_input` array.
- **Key logic:**  
  - Reads form fields from `$('Ask for Multi Images').first().json`  
  - Collects all Cloudinary items via `$('Loop Over Items').all()` and maps `item.json.secure_url`
- **Output:** to **HTTP Request**
- **Edge cases:** if uploads failed for some images, array may be incomplete; if Loop Over Items has no items (all skipped), Kie.ai call may fail.

#### Node: **Image upload failed (Cloudinary) multiple-images**
- **Type/role:** `telegram` — error notification.

**Sticky note covering this block:**  
“## Multi-Image Fusion …”

---

### 2.5 Image Generation & Delivery (Kie.ai + polling + Telegram send)

**Overview:**  
Creates a generation task at Kie.ai, polls for completion, downloads the resulting image, and sends it to Telegram with a download link.

**Nodes involved:**
- HTTP Request (createTask)
- Wait
- HTTP Request2 (recordInfo)
- If1
- HTTP Request1 (download image binary)
- Send a photo message
- Image generation failed (Kie.ai)

#### Node: **HTTP Request** (Create Image Job)
- **Type/role:** `httpRequest` — POST to Kie.ai createTask.
- **Config highlights:**
  - URL: `https://api.kie.ai/api/v1/jobs/createTask`
  - Method: POST
  - Body: `jsonBody = {{$json.body}}` (comes from *PREP* code nodes)
  - Headers: `Authorization: Bearer <your api key>`, `Content-Type: application/json`
  - `onError: continueErrorOutput`
- **Outputs:**  
  - Success → **Wait**  
  - Error output → **Image generation failed (Kie.ai)** (wired as second connection)
- **Edge cases:** missing/invalid API key; malformed JSON (body is a stringified JSON — if Kie.ai expects an object, this can be an integration mismatch); rate limits.

#### Node: **Wait**
- **Type/role:** wait/pause node.
- **Config:** waits 10 (seconds).
- **Output:** to **HTTP Request2**
- **Edge cases:** fixed wait may be too short; increases polling loops.

#### Node: **HTTP Request2** (recordInfo)
- **Type/role:** `httpRequest` — checks task status.
- **Config:**
  - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
  - Query: `taskId = {{$json.data.taskId}}`
  - Same Authorization header
- **Output:** to **If1**
- **Edge cases:** taskId missing; transient failures; API returns resultJson not parseable.

#### Node: **If1**
- **Type/role:** `if` — checks success state.
- **Condition:** `$json.data.state contains "success"`
- **True branch:** **HTTP Request1** (download)
- **False branch:** **Wait** (poll again)
- **Edge cases:** infinite polling if task never becomes success (no max retry); “failed” state will still loop.

#### Node: **HTTP Request1**
- **Type/role:** `httpRequest` — downloads generated image.
- **URL expression:** `{{ JSON.parse($json.data.resultJson).resultUrls[0] }}`
- **Output:** to **Send a photo message**
- **Edge cases:** resultJson not valid JSON; `resultUrls[0]` missing; file download large.

#### Node: **Send a photo message**
- **Type/role:** `telegram` — sends generated image to user.
- **Config:**
  - `operation: sendPhoto`
  - `binaryData: true` (expects HTTP Request1 produced binary)
  - `caption`: download link using `JSON.parse($json.data.resultJson).resultUrls[0]`
- **Output:** to **Share to Social Media**
- **Edge cases:** Telegram photo size constraints; binary field name mismatch; caption parse error.

#### Node: **Image generation failed (Kie.ai)**
- **Type/role:** `telegram` — error messaging for Kie.ai failures.

**Sticky note covering this block:**  
“## Image Generation & Delivery …”

---

### 2.6 Social Sharing Preparation (User choice → optional compression → caption generation)

**Overview:**  
After delivering the image, asks whether to share to social media, collects post text and platform selection, optionally compresses/downloads the image, and generates platform-specific captions via AI.

**Nodes involved:**
- Share to Social Media
- SHARING PROCESSOR (disabled)
- If2
- Compress Image (TinyPNG)
- Get Image
- JSON FORMATTER
- WITHOUT IMAGE DATA
- Image compression failed (TinyPNG)
- AI Agent1 + OpenAI Chat Model1 + Structured Output Parser

#### Node: **Share to Social Media**
- **Type/role:** `telegram` (`sendAndWait`) — form prompt for sharing.
- **Fields:** Post Content (required), Process Image (checkbox), Share to Social Media (checkbox with Facebook/Instagram/X).
- **Output:** to **SHARING PROCESSOR**
- **Edge cases:** if user selects none, downstream intended default is “share to all” (implemented in SHARING PROCESSOR).

#### Node: **SHARING PROCESSOR** (disabled)
- **Type/role:** `code` — normalizes share form output into:
  - `userPrompt` (post text)
  - `processImage` (boolean)
  - `socialMedia` (lowercase list; defaults to all if none checked)
- **Important:** Node is **disabled**, but connections still reference it (many downstream expressions use `$('SHARING PROCESSOR')...`). In live runs, this will break unless re-enabled or replaced.
- **Output:** to **If2**
- **Failure mode:** with node disabled, **If2** won’t receive expected data; expressions referencing it will fail.

#### Node: **If2**
- **Type/role:** `if` — checks whether image processing is requested.
- **Condition:** `$json.processImage == true`
- **True branch:** **Compress Image**
- **False branch:** **WITHOUT IMAGE DATA**
- **Edge cases:** if `$json.processImage` missing due to SHARING PROCESSOR disabled.

#### Node: **Compress Image** (TinyPNG)
- **Type/role:** `httpRequest` — TinyPNG shrink by URL.
- **Config:**
  - POST `https://api.tinify.com/shrink`
  - Basic auth via n8n credential (TinyPNG API key as username, per TinyPNG convention)
  - JSON body uses Kie.ai output URL: `JSON.parse($('HTTP Request2').item.json.data.resultJson).resultUrls[0]`
  - `onError: continueErrorOutput`
- **Outputs:**  
  - Success → **Get Image**  
  - Error → **Image compression failed (TinyPNG)**
- **Edge cases:** TinyPNG monthly limit; authentication errors; invalid source URL.

#### Node: **Get Image**
- **Type/role:** `httpRequest` — downloads the compressed image from `$json.output.url`.
- **Output:** to **JSON FORMATTER**
- **Edge cases:** URL missing; download errors.

#### Node: **JSON FORMATTER**
- **Type/role:** `code` — merges JSON sharing fields + binary image from **Get Image**.
- **Key logic:** reads `$('SHARING PROCESSOR')...` and `$('Get Image').item.binary`
- **Output:** to **AI Agent1**
- **Edge cases:** SHARING PROCESSOR disabled; binary key mismatch.

#### Node: **WITHOUT IMAGE DATA**
- **Type/role:** `code` — passes only JSON sharing fields (no binary).
- **Output:** to **AI Agent1**
- **Edge cases:** same dependency on SHARING PROCESSOR.

#### Node: **Image compression failed (TinyPNG)**
- **Type/role:** `telegram` — warns compression failed but generation succeeded.

#### Node: **AI Agent1**
- **Type/role:** LangChain agent — creates platform-optimized captions.
- **Inputs:** `userPrompt` + `socialMedia` list.
- **System message:** instructs generating only JSON fields for selected platforms.
- **Has output parser:** yes → **Structured Output Parser**
- **Connections:** uses **OpenAI Chat Model1**; output routed to **Switch1**.
- **Edge cases:** if socialMedia empty/undefined; JSON formatting errors; model deviates from schema.

#### Node: **OpenAI Chat Model1**
- **Type/role:** OpenAI model `gpt-5`, same settings as earlier.
- **Edge cases:** API issues, timeouts.

#### Node: **Structured Output Parser**
- **Type/role:** structured JSON parser enforcing schema keys `facebook`, `instagram`, `x`.
- **Edge cases:** if AI returns invalid JSON; missing required keys.

**Sticky note covering this block:**  
“## Social Sharing Preparation …”

---

### 2.7 Publishing to Social Platforms (Blotato)

**Overview:**  
Based on platform selection, routes to one or more Blotato publish nodes to post the image URL and the generated caption.

**Nodes involved:**
- Switch1
- Facebook
- Instagram
- X
- Note: Facebook (sticky note)

#### Node: **Switch1**
- **Type/role:** `switch` — “fan-out” to all matching selected platforms.
- **Config:** `allMatchingOutputs = true`
- **Conditions:** checks `$('SHARING PROCESSOR').item.json.socialMedia.includes('facebook'/'instagram'/'x')`
- **Outputs:** to **Facebook**, **Instagram**, **X**
- **Edge cases:** SHARING PROCESSOR disabled; `.includes` on undefined; platform casing mismatch.

#### Node: **Facebook** (Blotato)
- **Type/role:** `@blotato/...` — publishes image + text to a Facebook page.
- **Config:** accountId + facebookPageId preselected, caption from `$json.output.facebook`, media URL from Kie.ai output URL.
- **Edge cases:** Blotato auth; page permissions; invalid media URL; Facebook API restrictions.

#### Node: **Instagram** (Blotato)
- **Type/role:** publishes to Instagram.
- **Config:** caption `$json.output.instagram`, media URL same.
- **Edge cases:** IG requires certain media constraints; Blotato account linking.

#### Node: **X** (Blotato)
- **Type/role:** publishes to X/Twitter.
- **Config:** caption `$json.output.x`, media URL same.
- **Edge cases:** X character limits; media upload failures; account permissions.

**Sticky note covering this block:**  
“## Social Media Publishing …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | telegramTrigger | Entry point: receive Telegram messages | — | Switch | ## Entry & Command Routing<br>Receives Telegram commands and routes execution to the correct generation mode (text-to-image, image-to-image, or multi-image fusion). |
| Switch | switch | Route commands to modes | Telegram Trigger | Ask for Text Prompt; Ask for Image Upload; Ask for Multi Images; Show Menu | ## Entry & Command Routing<br>Receives Telegram commands and routes execution to the correct generation mode (text-to-image, image-to-image, or multi-image fusion). |
| Show Menu | telegram | Show available commands | Switch (fallback) | — | ## Entry & Command Routing<br>Receives Telegram commands and routes execution to the correct generation mode (text-to-image, image-to-image, or multi-image fusion). |
| Ask for Text Prompt | telegram | Collect text prompt + options | Switch | If | ## Text-to-Image<br>Collects a text prompt, optionally enhances it if selected by the user, generates an image with Nano Banana PRO, and returns the result to Telegram. |
| If | if | Decide whether to enhance prompt | Ask for Text Prompt | AI Agent; STANDARD TEXT PREP | ## Text-to-Image<br>Collects a text prompt, optionally enhances it if selected by the user, generates an image with Nano Banana PRO, and returns the result to Telegram. |
| AI Agent | langchain agent | Enhance the image prompt | If (true) | ENHANCED TEXT PREP | ## Text-to-Image<br>Collects a text prompt, optionally enhances it if selected by the user, generates an image with Nano Banana PRO, and returns the result to Telegram. |
| OpenAI Chat Model | OpenAI chat model | LLM for prompt enhancement | — | AI Agent (ai_languageModel) | ## Text-to-Image<br>Collects a text prompt, optionally enhances it if selected by the user, generates an image with Nano Banana PRO, and returns the result to Telegram. |
| ENHANCED TEXT PREP | code | Build Kie.ai request from enhanced prompt | AI Agent | HTTP Request | ## Text-to-Image<br>Collects a text prompt, optionally enhances it if selected by the user, generates an image with Nano Banana PRO, and returns the result to Telegram. |
| STANDARD TEXT PREP | code | Build Kie.ai request from original prompt | If (false) | HTTP Request | ## Text-to-Image<br>Collects a text prompt, optionally enhances it if selected by the user, generates an image with Nano Banana PRO, and returns the result to Telegram. |
| Ask for Image Upload | telegram | Collect single image + prompt + options | Switch | Upload Image | ## Image-to-Image<br>Collects a user-uploaded image, prepares it for processing, generates a transformed version using Nano Banana PRO, and returns the result to Telegram. |
| Upload Image | cloudinary | Upload single image to Cloudinary | Ask for Image Upload | IMAGE-TO-IMAGE PREP; Image upload failed (Cloudinary) image-image | ## Image-to-Image<br>Collects a user-uploaded image, prepares it for processing, generates a transformed version using Nano Banana PRO, and returns the result to Telegram. |
| IMAGE-TO-IMAGE PREP | code | Build Kie.ai request with Cloudinary image URL | Upload Image | HTTP Request | ## Image-to-Image<br>Collects a user-uploaded image, prepares it for processing, generates a transformed version using Nano Banana PRO, and returns the result to Telegram. |
| Image upload failed (Cloudinary) image-image | telegram | Error message for single upload failure | Upload Image (error) | — | ## Image-to-Image<br>Collects a user-uploaded image, prepares it for processing, generates a transformed version using Nano Banana PRO, and returns the result to Telegram. |
| Ask for Multi Images | telegram | Collect multiple images + prompt + options | Switch | MULTI-IMAGE SPLITTER | ## Multi-Image Fusion<br>Collects multiple user-uploaded images, prepares them for processing, generates a combined scene using Nano Banana PRO, and returns the result to Telegram. |
| MULTI-IMAGE SPLITTER | code | Split multi-upload into items (one binary each) | Ask for Multi Images | Loop Over Items | ## Multi-Image Fusion<br>Collects multiple user-uploaded images, prepares them for processing, generates a combined scene using Nano Banana PRO, and returns the result to Telegram. |
| Loop Over Items | splitInBatches | Iterate uploads and collect results | MULTI-IMAGE SPLITTER; Upload an asset from file data | MULTI-IMAGE PREP; Upload an asset from file data | ## Multi-Image Fusion<br>Collects multiple user-uploaded images, prepares them for processing, generates a combined scene using Nano Banana PRO, and returns the result to Telegram. |
| Upload an asset from file data | cloudinary | Upload each image item to Cloudinary | Loop Over Items (loop) | Loop Over Items; Image upload failed (Cloudinary) multiple-images | ❌ Image upload failed (Cloudinary)<br>## Multi-Image Fusion<br>Collects multiple user-uploaded images, prepares them for processing, generates a combined scene using Nano Banana PRO, and returns the result to Telegram. |
| Image upload failed (Cloudinary) multiple-images | telegram | Error message for multi-upload failure | Upload an asset from file data (error) | — | ## Multi-Image Fusion<br>Collects multiple user-uploaded images, prepares them for processing, generates a combined scene using Nano Banana PRO, and returns the result to Telegram. |
| MULTI-IMAGE PREP | code | Build Kie.ai request with multiple Cloudinary URLs | Loop Over Items (done) | HTTP Request | ## Multi-Image Fusion<br>Collects multiple user-uploaded images, prepares them for processing, generates a combined scene using Nano Banana PRO, and returns the result to Telegram. |
| HTTP Request | httpRequest | Create Kie.ai image generation task | ENHANCED TEXT PREP; STANDARD TEXT PREP; IMAGE-TO-IMAGE PREP; MULTI-IMAGE PREP | Wait; Image generation failed (Kie.ai) | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| Image generation failed (Kie.ai) | telegram | Error message when Kie.ai task creation fails | HTTP Request (error) | — | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| Wait | wait | Delay between polling attempts | HTTP Request; If1 (false) | HTTP Request2 | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| HTTP Request2 | httpRequest | Poll Kie.ai task status | Wait | If1 | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| If1 | if | Check job success vs keep polling | HTTP Request2 | HTTP Request1; Wait | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| HTTP Request1 | httpRequest | Download generated image binary | If1 (true) | Send a photo message | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| Send a photo message | telegram | Send generated image back to user | HTTP Request1 | Share to Social Media | ## Image Generation & Delivery<br>Creates the image generation job using Kie.ai, checks completion status, retrieves the result, and sends the generated image with its download link back to Telegram. |
| Share to Social Media | telegram | Ask/share form (text + platform selection) | Send a photo message | SHARING PROCESSOR | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| SHARING PROCESSOR | code | Normalize sharing form output (disabled) | Share to Social Media | If2 | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| If2 | if | Branch: compress image or skip | SHARING PROCESSOR | Compress Image; WITHOUT IMAGE DATA | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| Compress Image | httpRequest | TinyPNG shrink-by-URL | If2 (true) | Get Image; Image compression failed (TinyPNG) | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| Image compression failed (TinyPNG) | telegram | Compression failure notification | Compress Image (error) | — | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| Get Image | httpRequest | Download compressed image | Compress Image | JSON FORMATTER | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| JSON FORMATTER | code | Merge share JSON + binary image | Get Image | AI Agent1 | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| WITHOUT IMAGE DATA | code | Pass share JSON only | If2 (false) | AI Agent1 | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| AI Agent1 | langchain agent | Generate platform captions JSON | JSON FORMATTER; WITHOUT IMAGE DATA | Switch1 | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| OpenAI Chat Model1 | OpenAI chat model | LLM for caption generation | — | AI Agent1 (ai_languageModel) | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| Structured Output Parser | structured output parser | Enforce JSON schema for captions | — | AI Agent1 (ai_outputParser) | ## Social Sharing Preparation<br>Collects post content and platform selection, optionally processes the image, and generates platform-optimized captions for social media sharing. |
| Switch1 | switch | Fan-out to chosen platforms | AI Agent1 | Facebook; Instagram; X | ## Social Media Publishing<br>Publishes the generated image and caption to selected platforms (Facebook, Instagram, X) based on the user’s selection. |
| Facebook | blotato | Publish to Facebook page | Switch1 | — | ## Social Media Publishing<br>Publishes the generated image and caption to selected platforms (Facebook, Instagram, X) based on the user’s selection. |
| Instagram | blotato | Publish to Instagram | Switch1 | — | ## Social Media Publishing<br>Publishes the generated image and caption to selected platforms (Facebook, Instagram, X) based on the user’s selection. |
| X | blotato | Publish to X/Twitter | Switch1 | — | ## Social Media Publishing<br>Publishes the generated image and caption to selected platforms (Facebook, Instagram, X) based on the user’s selection. |
| Workflow Overview | stickyNote | Documentation/overview | — | — |  |
| Note: Telegram Trigger | stickyNote | Documentation | — | — |  |
| Note: OpenAI Chat Model | stickyNote | Documentation | — | — |  |
| Note: Upload Image | stickyNote | Documentation | — | — |  |
| Note: Code in JavaScript3 | stickyNote | Documentation | — | — |  |
| Note: If1 | stickyNote | Documentation | — | — |  |
| Note: If2 | stickyNote | Documentation | — | — |  |
| Note: Facebook | stickyNote | Documentation | — | — |  |
| Sticky Note | stickyNote | Template thumbnail image | — | — |  |
| Sticky Note1 | stickyNote | Template thumbnail image | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram entry**
1. Add **Telegram Trigger** node  
   - Updates: `message`  
   - Connect Telegram credentials (bot token).
2. Add **Switch** node  
   - Create three rules on `{{$json.message.text}}` equals:  
     - `/text_to_image` → Output 0  
     - `/image_to_image` → Output 1  
     - `/multi_image` → Output 2  
   - Fallback output: `extra`.
3. Add **Show Menu** (Telegram “Send Message”)  
   - `chatId = {{$('Telegram Trigger').item.json.message.from.id}}`  
   - Send the menu text.  
4. Wire: **Telegram Trigger → Switch**, and fallback to **Show Menu**.

2) **Text-to-Image path**
5. Add **Ask for Text Prompt** (Telegram `sendAndWait`, responseType `customForm`)  
   - Same fields: Image Description (textarea), Enhance Prompt (checkbox), Aspect Ratio (dropdown), Quality (dropdown).  
   - `chatId` as above.
6. Add **If** node  
   - Condition: `{{$json.data["Enhance Prompt"].length}}` equals `1` (string/number tolerated).
7. Add **OpenAI Chat Model** node (LangChain)  
   - Model: `gpt-5` (or your available equivalent)  
   - Timeout: 180000ms  
   - Credentials: OpenAI API key.
8. Add **AI Agent** node  
   - Prompt text: include `{{$json.data['Image Description']}}`, aspect ratio, quality  
   - Paste the system message (prompt-engineering instructions)  
   - Connect model input to **OpenAI Chat Model**.
9. Add **ENHANCED TEXT PREP** (Code)  
   - Build body: `{ model:"nano-banana-pro", input:{ prompt, image_input:[], aspect_ratio, resolution, output_format:"png"}}`  
   - Ensure output JSON has `body` as a string: `JSON.stringify(body)`.
10. Add **STANDARD TEXT PREP** (Code) with same body build but using original prompt from the form.
11. Wire:  
   - **Switch output 0 → Ask for Text Prompt → If**  
   - **If true → AI Agent → ENHANCED TEXT PREP**  
   - **If false → STANDARD TEXT PREP**

3) **Image-to-Image path**
12. Add **Ask for Image Upload** (Telegram `sendAndWait`, customForm)  
   - File field (required), Image Description, Aspect Ratio, Quality.
13. Add **Cloudinary Upload** node (Cloudinary) named **Upload Image**  
   - Operation: uploadFile  
   - File parameter must match Telegram binary key (verify in executions; in this workflow it’s `Upload_Product_Image`).  
   - Credentials: Cloudinary API.
   - Enable “Continue on Fail” if you want error routing.
14. Add **IMAGE-TO-IMAGE PREP** (Code)  
   - Read form data; get `secure_url` from Cloudinary output; set `image_input: [secure_url]`.
15. Add **Image upload failed…** (Telegram send message) for the error output.
16. Wire: **Switch output 1 → Ask for Image Upload → Upload Image → IMAGE-TO-IMAGE PREP** (and error to Telegram failure message).

4) **Multi-image fusion path**
17. Add **Ask for Multi Images** (Telegram `sendAndWait`, customForm)  
   - Multi-file upload field (2–8), Image Description, Aspect Ratio, Quality.
18. Add **MULTI-IMAGE SPLITTER** (Code)  
   - Split into items each with `binary.image`.
   - Ensure you reference the correct incoming binary keys (`Upload_Images_0` etc.); adjust if your Telegram node uses different naming.
19. Add **Loop Over Items** (`Split in Batches`)  
20. Add Cloudinary node **Upload an asset from file data**  
   - Operation uploadFile  
   - File: `image`
21. Add **Image upload failed… multiple-images** (Telegram error message).
22. Add **MULTI-IMAGE PREP** (Code)  
   - Collect all uploaded URLs from the batch loop and build `image_input: [url1,url2,...]`.
23. Wire:  
   - **Switch output 2 → Ask for Multi Images → MULTI-IMAGE SPLITTER → Loop Over Items**  
   - **Loop Over Items (loop) → Upload an asset… → Loop Over Items**  
   - **Loop Over Items (done) → MULTI-IMAGE PREP**

5) **Kie.ai generation + polling**
24. Add **HTTP Request** node (createTask)  
   - POST `https://api.kie.ai/api/v1/jobs/createTask`  
   - JSON body: `{{$json.body}}`  
   - Headers: Authorization Bearer, Content-Type application/json  
   - “Continue on Fail” if you want the error branch.
25. Add **Wait** node: 10 seconds.
26. Add **HTTP Request2** node (recordInfo)  
   - GET `https://api.kie.ai/api/v1/jobs/recordInfo`  
   - Query: `taskId={{$json.data.taskId}}`  
   - Same Authorization header.
27. Add **If1** node  
   - Condition: `{{$json.data.state}}` contains `success`
28. Add **HTTP Request1** node (download result)  
   - URL: `{{ JSON.parse($json.data.resultJson).resultUrls[0] }}`
29. Add **Send a photo message** (Telegram sendPhoto)  
   - chatId: `{{$('Telegram Trigger').first().json.message.chat.id}}`  
   - binaryData: true  
   - caption: download URL.
30. Add **Image generation failed (Kie.ai)** Telegram error node.
31. Wire:  
   - All PREP nodes → **HTTP Request**  
   - **HTTP Request → Wait → HTTP Request2 → If1**  
   - **If1 false → Wait** (poll loop)  
   - **If1 true → HTTP Request1 → Send a photo message**  
   - **HTTP Request error → Image generation failed**

6) **Sharing + publishing (optional)**
32. Add **Share to Social Media** (Telegram `sendAndWait`) with fields (Post Content, Process Image checkbox, Share to Social Media checkbox list).
33. Add **SHARING PROCESSOR** (Code) and **ensure it is enabled**  
   - Output `{userPrompt, processImage:boolean, socialMedia:[facebook|instagram|x]}` with default-to-all behavior.
34. Add **If2** node: `{{$json.processImage}} == true`
35. Add **Compress Image** (HTTP Request to TinyPNG)  
   - POST `https://api.tinify.com/shrink`  
   - Basic auth credential (TinyPNG)  
   - Body uses generated image URL.  
36. Add **Get Image** (HTTP Request) to download compressed binary from `$json.output.url`
37. Add **JSON FORMATTER** (Code) merging share JSON + binary from Get Image
38. Add **WITHOUT IMAGE DATA** (Code) passing only share JSON
39. Add **Image compression failed (TinyPNG)** Telegram node (error output from Compress)
40. Add **OpenAI Chat Model1**, **Structured Output Parser**, **AI Agent1**  
   - AI Agent1 must output platform JSON; connect model + output parser.
41. Add **Switch1** with `allMatchingOutputs=true` and conditions based on selected platforms.
42. Add **Blotato** nodes: Facebook, Instagram, X  
   - Provide Blotato credentials  
   - Set account/page IDs  
   - postContentText from AI output fields  
   - postContentMediaUrls from Kie.ai result URL.
43. Wire:  
   - **Send a photo message → Share to Social Media → SHARING PROCESSOR → If2**  
   - **If2 true → Compress Image → Get Image → JSON FORMATTER → AI Agent1**  
   - **If2 false → WITHOUT IMAGE DATA → AI Agent1**  
   - **AI Agent1 → Switch1 → Blotato nodes**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Template Thumbnail | https://res.cloudinary.com/dfnh7m0iw/image/upload/v1766058388/85944a2b3def45bfe9c1da0a8746f523_1765470283_1_nuqve0.png |
| Template Thumbnail | https://res.cloudinary.com/dfnh7m0iw/image/upload/v1766058389/0431a5b13cf422e56ec88243b33a9b99_1765574830963_myc3ya.png |
| Workflow overview and setup prerequisites (Telegram bot, Kie.ai, Cloudinary, OpenAI; optional TinyPNG and Blotato) | Provided in the workflow’s “Workflow Overview” sticky note content |

### Important implementation warnings (actionable)
- **SHARING PROCESSOR is disabled** but is referenced everywhere downstream (Switch1 conditions, formatter nodes). To make social sharing work, **enable it** or refactor downstream nodes to read directly from `Share to Social Media`.
- **Multi-image splitter label mismatch:** it references `data['Fusion Description']`, but the form field is **“Image Description”**. This doesn’t break generation (since later code uses “Image Description”), but it’s misleading and should be corrected.
- **Polling loop has no max retries:** If Kie.ai returns `failed` or stalls, the workflow can loop indefinitely between **If1 → Wait → HTTP Request2**. Consider adding a counter or timeout.