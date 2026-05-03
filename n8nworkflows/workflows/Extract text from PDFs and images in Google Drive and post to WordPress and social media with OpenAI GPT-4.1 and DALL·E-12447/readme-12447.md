Extract text from PDFs and images in Google Drive and post to WordPress and social media with OpenAI GPT-4.1 and DALL·E

https://n8nworkflows.xyz/workflows/extract-text-from-pdfs-and-images-in-google-drive-and-post-to-wordpress-and-social-media-with-openai-gpt-4-1-and-dall-e-12447


# Extract text from PDFs and images in Google Drive and post to WordPress and social media with OpenAI GPT-4.1 and DALL·E

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow watches Google Drive for newly added **PDFs or images**, extracts their text (PDF parsing or OCR-like extraction via an LLM chain), then uses AI to generate an accompanying image (DALL·E via OpenAI), and finally publishes content to **WordPress** and multiple **social media channels** (Facebook, Telegram, LinkedIn, Discord). A separate branch appears to exist for sending “warning/notification” messages to multiple channels, but it is currently gated behind a **NoOp (“Do nothing”)** node.

**Target use cases:**
- Auto-ingesting documents (PDF/image) from Drive, extracting text, and broadcasting a summary/post.
- Auto-generating a featured image for a blog/social post using OpenAI image generation.
- Syndicating to multiple platforms in parallel.

### Logical blocks (by dependencies)

**1.1 Drive intake & file-type routing**
- Trigger on new Drive files → Switch routes to PDF vs image download.

**1.2 Text extraction**
- PDFs: download → “Extract from File” (PDF text extraction)
- Images: download → LLM chain labeled “Vertex AI extract text” (despite name, it’s a LangChain LLM-chain node)

**1.3 Text-post fanout + image prompt generation**
- A NoOp fanout sends extracted text to Telegram/Twitter/LinkedIn/Facebook and also feeds an “Expert AI image prompt creator”.

**1.4 Image generation & distribution**
- Prompt creator (agent + OpenAI chat model) → OpenAI image generation → image editing → WordPress + social image posts + Discord

**1.5 WordPress media upload & featured image**
- Create WP post → request/edit image → upload media → update metadata → set featured image

**1.6 Notification/warning branch (currently inert)**
- Another NoOp (“Do nothing”) fans out to Telegram/Twitter/Facebook/Gmail/Discord and a custom “Rapiwa” node.

---

## 2. Block-by-Block Analysis

### 2.1 Drive intake & file-type routing

**Overview:**  
Detects new files in Google Drive, then routes execution depending on whether the file is a PDF or an image.

**Nodes involved:**
- **Get PDF or Images** (Google Drive Trigger)
- **Route based on PDF or Image** (Switch)
- **Download PDF** (Google Drive)
- **Download Image** (Google Drive)

#### Node details

**Get PDF or Images**
- **Type/role:** `googleDriveTrigger` — entry point; listens for Drive events (new/updated file).
- **Configuration (interpreted):** Not provided in JSON (empty params), but typically requires:
  - Drive account credentials (OAuth2)
  - Trigger event (e.g., “File created” / “File updated”)
  - Folder scope or query
- **Inputs/outputs:**
  - **Output →** Route based on PDF or Image
- **Failure/edge cases:**
  - OAuth token expiration / insufficient Drive permissions
  - Trigger firing on non-target files (no filtering shown)
  - Missing file metadata (mimeType) may break downstream routing logic

**Route based on PDF or Image**
- **Type/role:** `switch` — branches into PDF vs Image paths.
- **Configuration:** Empty in JSON; in practice must define conditions, commonly:
  - If `mimeType` equals `application/pdf` → PDF branch
  - Else if `mimeType` starts with `image/` → Image branch
- **Inputs/outputs:**
  - **Input ←** Get PDF or Images
  - **Outputs →** Download PDF (output index 0), Download Image (output index 1)
- **Failure/edge cases:**
  - If no conditions are configured, routing may be incorrect or default-only
  - Unsupported file types (e.g., DOCX) may fall through and never get processed

**Download PDF**
- **Type/role:** `googleDrive` — download binary content for PDF.
- **Configuration:** Empty; typically “Download” operation using file ID from trigger.
- **Special setting:** `executeOnce: true` (node runs once in manual executions; in production it still runs per item depending on n8n semantics and execution mode).
- **Inputs/outputs:**
  - **Input ←** Route based on PDF or Image (PDF branch)
  - **Output →** Extract data from PDF
- **Failure/edge cases:**
  - File not found / permission denied
  - Large PDFs can exceed memory limits or download timeouts

**Download Image**
- **Type/role:** `googleDrive` — download binary content for image.
- **Configuration:** Empty; typically “Download” by file ID.
- **Special settings:** `executeOnce: true`, `alwaysOutputData: true`, `retryOnFail: false`
  - `alwaysOutputData` can allow continuation even if the node returns no items (dangerous if downstream expects binary).
- **Inputs/outputs:**
  - **Input ←** Route based on PDF or Image (Image branch)
  - **Output →** Vertex AI extract text
- **Failure/edge cases:**
  - Non-image file routed here will break OCR/extraction
  - If download yields empty binary, downstream LLM chain likely fails

---

### 2.2 Text extraction (PDF path + Image path)

**Overview:**  
Extracts text from PDFs using the Extract From File node; extracts text from images using an LLM chain (named “Vertex AI extract text”) backed by an OpenAI Chat model node.

**Nodes involved:**
- **Extract data from PDF** (Extract From File)
- **Vertex AI extract text** (LangChain LLM chain)
- **OpenAI Model1** (LangChain Chat Model)

#### Node details

**Extract data from PDF**
- **Type/role:** `extractFromFile` — parses file content (PDF) to text/data.
- **Configuration:** Empty; typically set to:
  - Input binary property (commonly `data`)
  - Extraction mode: text
- **Inputs/outputs:**
  - **Input ←** Download PDF
  - **Output →** do nothing (fanout NoOp)
- **Failure/edge cases:**
  - Scanned PDFs (image-only) may yield empty text unless OCR is used
  - Password-protected PDFs will fail
  - Corrupt PDFs cause parsing exceptions

**OpenAI Model1**
- **Type/role:** `lmChatOpenAi` — provides a Chat LLM to LangChain nodes via `ai_languageModel` connection.
- **Configuration:** Empty in JSON; typically includes:
  - Model name (workflow title suggests GPT-4.1)
  - API key credential (OpenAI)
  - Temperature / max tokens
- **Connections:**
  - **Output (ai_languageModel) →** Vertex AI extract text
- **Failure/edge cases:**
  - Invalid API key / quota exhaustion
  - Model not available for the account/region
  - Responses truncated if token limits too low

**Vertex AI extract text**
- **Type/role:** `chainLlm` — LangChain “LLM Chain” used here as OCR/text extraction from image binary.
- **Important note:** Despite its name, it is **not** a Google Vertex node in this JSON; it’s an n8n LangChain chain node.
- **Configuration:** Empty; in practice requires:
  - Prompt template instructing the model to extract text from the image
  - Mapping of input image binary (or image URL) into the chain
- **Inputs/outputs:**
  - **Input ←** Download Image
  - **Language model input ←** OpenAI Model1 (ai_languageModel connection)
  - **Output →** do nothing (fanout NoOp)
- **Failure/edge cases:**
  - If image binary is not passed in the expected field, chain fails or returns nonsense
  - LLMs are not deterministic OCR; formatting and accuracy vary
  - Very large images may exceed limits (size, tokens via base64, etc.)

---

### 2.3 Text-post fanout + image prompt creation

**Overview:**  
A NoOp node (“do nothing”) acts as a hub: it fans out extracted text to multiple text-based social posts and also triggers an AI agent to craft an image-generation prompt.

**Nodes involved:**
- **do nothing** (NoOp)
- **send text post** (Telegram)
- **Create Tweet text (free account)** (Twitter/X)
- **Create Tweet text (premium account)** (Twitter/X)
- **Create profile text post** (LinkedIn)
- **Create page text post** (LinkedIn)
- **Facebook text post** (Facebook Graph API)
- **Expert AI image prompt creator** (LangChain Agent)
- **OpenAI Model** (LangChain Chat Model)

#### Node details

**do nothing**
- **Type/role:** `noOp` — fanout/branching hub; preserves input items.
- **Inputs/outputs:**
  - **Inputs ←** Extract data from PDF OR Vertex AI extract text
  - **Outputs →** multiple nodes in parallel:
    - send text post (Telegram)
    - Create Tweet text (free + premium)
    - Create profile/page text post (LinkedIn)
    - Facebook text post
    - Expert AI image prompt creator
    - Do nothing (second NoOp / notifications branch)
- **Failure/edge cases:**
  - If upstream extraction returns empty text, all text posting nodes may publish empty/invalid content unless guarded

**send text post**
- **Type/role:** `telegram` — sends a text post/message to a chat/channel.
- **Configuration:** Empty; requires Telegram bot token + chat ID.
- **Failure/edge cases:** invalid chat ID, bot not admin in channel, message too long.

**Create Tweet text (free account)** / **Create Tweet text (premium account)**
- **Type/role:** `twitter` — posts tweets (or other Twitter actions).
- **Configuration:** Empty; likely two variants due to API limitations/tiers.
- **Failure/edge cases:**
  - Twitter API access restrictions (free tier often cannot post)
  - Rate limits; auth failures; policy blocks

**Create profile text post** / **Create page text post**
- **Type/role:** `linkedIn` — creates LinkedIn posts (profile vs organization/page).
- **Configuration:** Empty; requires LinkedIn OAuth and correct author URN.
- **Failure/edge cases:** missing permissions (w_member_social, w_organization_social), invalid URNs.

**Facebook text post**
- **Type/role:** `facebookGraphApi` — publishes a text post.
- **Configuration:** Empty; requires page access token and proper permissions.
- **Failure/edge cases:** token expiry, missing publish permissions, page restrictions.

**OpenAI Model**
- **Type/role:** `lmChatOpenAi` — chat model used by the agent.
- **Connections:**
  - **ai_languageModel →** Expert AI image prompt creator
- **Failure/edge cases:** same as OpenAI Model1.

**Expert AI image prompt creator**
- **Type/role:** `agent` (LangChain) — generates a high-quality prompt for image generation based on extracted text.
- **Configuration:** Empty; normally includes:
  - System instructions (e.g., “You are an expert prompt engineer…”)
  - Tool usage (if any) and output schema (prompt string)
- **Inputs/outputs:**
  - **Input ←** do nothing (extracted text)
  - **LLM input ←** OpenAI Model (ai_languageModel)
  - **Output →** Generate an image
- **Failure/edge cases:**
  - If extracted text is too long, the agent may produce weak/partial prompts
  - Output parsing issues if downstream expects a specific field name

---

### 2.4 Image generation & distribution (WordPress + social image posts)

**Overview:**  
Generates an image from the AI prompt, optionally edits it, then publishes to WordPress and multiple social channels as an image post.

**Nodes involved:**
- **Generate an image** (OpenAI image)
- **Edit Image2** (Edit Image)
- **Create WordPress Post** (WordPress)
- **Facebook Image post** (Facebook Graph API)
- **Telegram Image post** (Telegram)
- **Create profile image post** (LinkedIn)
- **Create page image post** (LinkedIn)
- **Post on Discord Channel** (Discord)
- **Do nothing** (NoOp; notifications hub)

#### Node details

**Generate an image**
- **Type/role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI operation node used here for image generation (title indicates DALL·E).
- **Configuration:** Empty; typically includes:
  - Operation: “Generate Image”
  - Model: DALL·E variant supported by account
  - Prompt: from Expert AI image prompt creator output
  - Size/quality/style
- **Inputs/outputs:**
  - **Input ←** Expert AI image prompt creator
  - **Output →** Edit Image2
- **Failure/edge cases:**
  - Content policy blocks
  - Invalid model/endpoint
  - Large images or unsupported sizes

**Edit Image2**
- **Type/role:** `editImage` — post-processes generated image (resize/crop/format).
- **Configuration:** Empty; commonly used to convert to JPG/PNG and enforce dimensions.
- **Inputs/outputs:**
  - **Input ←** Generate an image
  - **Outputs →** multiple parallel:
    - Create WordPress Post
    - Facebook Image post
    - Telegram Image post
    - LinkedIn image posts (profile/page)
    - Post on Discord Channel
    - Do nothing (notification hub)
- **Failure/edge cases:** missing binary data property; unsupported image format.

**Create WordPress Post**
- **Type/role:** `wordpress` — creates a post entry.
- **Configuration:** Empty; typically:
  - Site URL + credentials (Application Password or OAuth/plugin-based)
  - Post title/content/status
  - Potentially uses extracted text + generated image URL later
- **Connections:**
  - **Output →** HTTP Request1 (likely for further WP processing)
- **Failure/edge cases:** auth errors, XML-RPC disabled, REST API permissions, invalid post content.

**Facebook Image post / Telegram Image post / LinkedIn image posts / Post on Discord Channel**
- **Role:** publish the generated/edited image to each platform.
- **Configuration:** Empty; each requires appropriate credentials and expected binary field mapping.
- **Failure/edge cases:** platform-specific file size limits, missing publish scopes, rate limits.

**Do nothing** (at position ~1808)
- **Type/role:** `noOp` — acts as a hub for notifications/actions (see block 2.6).
- **Inputs:** from Edit Image2 and also from earlier NoOp hub.
- **Outputs:** Send a text message / Create Direct Message / Warning Message / Send a message / Post on message / Rapiwa

---

### 2.5 WordPress media upload & featured image

**Overview:**  
After creating the WP post, the workflow appears to prepare/edit an image, upload it to WordPress as media, set metadata, and set it as the featured image.

**Nodes involved:**
- **HTTP Request1** (HTTP Request)
- **Edit Image** (Edit Image)
- **upload media to wp** (HTTP Request)
- **upload image to meta data** (HTTP Request)
- **set featured image** (HTTP Request)

#### Node details

**HTTP Request1**
- **Type/role:** `httpRequest` — likely retrieves an image or WP endpoint call related to the created post.
- **Configuration:** Empty; typically includes:
  - Method/URL (e.g., fetch generated image, or call WP REST)
  - Authentication (basic/app password, token)
- **Inputs/outputs:**
  - **Input ←** Create WordPress Post
  - **Output →** Edit Image
- **Failure/edge cases:** 401/403, wrong content-type, timeout.

**Edit Image**
- **Type/role:** `editImage` — prepares the image for WP media library upload (format/size).
- **Inputs/outputs:**
  - **Input ←** HTTP Request1
  - **Output →** upload media to wp
- **Failure/edge cases:** missing binary, invalid image.

**upload media to wp**
- **Type/role:** `httpRequest` — uploads binary to WordPress media endpoint (`/wp-json/wp/v2/media` commonly).
- **Version:** 4.2 (HTTP Request node)
- **Inputs/outputs:**
  - **Input ←** Edit Image
  - **Output →** upload image to meta data
- **Failure/edge cases:**
  - Incorrect multipart/form-data setup
  - Missing `Content-Disposition` filename
  - WP rejects file type or exceeds upload_max_filesize

**upload image to meta data**
- **Type/role:** `httpRequest` — updates media metadata (alt text, caption, etc.) or attaches media to post.
- **Inputs/outputs:**
  - **Input ←** upload media to wp
  - **Output →** set featured image
- **Failure/edge cases:** wrong media ID parsing from previous response.

**set featured image**
- **Type/role:** `httpRequest` — sets the created media as the post’s featured image (likely updates post with `featured_media`).
- **Inputs/outputs:**
  - **Input ←** upload image to meta data
  - **Output:** none shown
- **Failure/edge cases:** wrong post ID, insufficient permissions to edit posts.

---

### 2.6 Notification / warning branch (currently inert but connected)

**Overview:**  
A second NoOp (“Do nothing”) fans out to multiple messaging endpoints (Telegram, Twitter DM, Facebook warning message, Gmail email, Discord) plus a custom “Rapiwa” node. This looks like an alerting/monitoring pathway, but there is no logic shown to conditionally activate it—it will run whenever it receives input.

**Nodes involved:**
- **Do nothing** (NoOp)
- **Send a text message** (Telegram)
- **Create Direct Message** (Twitter)
- **Warning Message** (Facebook Graph API)
- **Send a message** (Gmail)
- **Post on message** (Discord)
- **Rapiwa** (Custom community node)

#### Node details

**Do nothing**
- **Type/role:** `noOp` hub for notifications.
- **Inputs:** from:
  - do nothing (text hub)
  - Edit Image2 (image hub)
- **Outputs:** to the six action nodes listed below.

**Send a text message (Telegram)** / **Post on message (Discord)** / **Send a message (Gmail)** / **Warning Message (FacebookGraphApi)** / **Create Direct Message (Twitter)**  
- **Role:** send outbound notifications.
- **Configuration:** Empty; requires corresponding credentials and message content expressions.
- **Failure/edge cases:** message formatting, rate limiting, permission scope.

**Rapiwa**
- **Type/role:** `n8n-nodes-rapiwa.rapiwa` — custom/community node (unknown behavior from JSON).
- **Configuration:** Empty.
- **Version-specific requirements:** requires the `n8n-nodes-rapiwa` package installed on the n8n instance.
- **Failure/edge cases:**
  - Node package missing/incompatible → workflow fails to load/execute
  - Unknown auth/config requirements

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Get PDF or Images | Google Drive Trigger | Entry point: watch Drive | — | Route based on PDF or Image |  |
| Route based on PDF or Image | Switch | Route by mime/type | Get PDF or Images | Download PDF; Download Image |  |
| Download PDF | Google Drive | Download PDF binary | Route based on PDF or Image | Extract data from PDF |  |
| Extract data from PDF | Extract From File | Extract text from PDF | Download PDF | do nothing |  |
| Download Image | Google Drive | Download image binary | Route based on PDF or Image | Vertex AI extract text |  |
| OpenAI Model1 | OpenAI Chat Model (LangChain) | LLM provider for OCR chain | — | Vertex AI extract text (ai_languageModel) |  |
| Vertex AI extract text | LangChain LLM Chain | “OCR-like” text extraction from image | Download Image | do nothing |  |
| do nothing | NoOp | Fanout hub (text posts + prompt creation) | Extract data from PDF; Vertex AI extract text | send text post; Create Tweet text (free); Create Tweet text (premium); Create profile text post; Create page text post; Facebook text post; Expert AI image prompt creator; Do nothing |  |
| send text post | Telegram | Publish text message | do nothing | — |  |
| Create Tweet text (free account) | Twitter | Publish tweet (free path) | do nothing | — |  |
| Create Tweet text (premium account) | Twitter | Publish tweet (premium path) | do nothing | — |  |
| Create profile text post | LinkedIn | Publish profile text post | do nothing | — |  |
| Create page text post | LinkedIn | Publish page text post | do nothing | — |  |
| Facebook text post | Facebook Graph API | Publish Facebook text post | do nothing | — |  |
| OpenAI Model | OpenAI Chat Model (LangChain) | LLM provider for prompt agent | — | Expert AI image prompt creator (ai_languageModel) |  |
| Expert AI image prompt creator | LangChain Agent | Create prompt for image generation | do nothing | Generate an image |  |
| Generate an image | OpenAI (LangChain OpenAI node) | Generate image (DALL·E) | Expert AI image prompt creator | Edit Image2 |  |
| Edit Image2 | Edit Image | Prepare generated image | Generate an image | Create WordPress Post; Facebook Image post; Telegram Image post; Create profile image post; Create page image post; Post on Discord Channel; Do nothing |  |
| Create WordPress Post | WordPress | Create WP post | Edit Image2 | HTTP Request1 |  |
| HTTP Request1 | HTTP Request | WP/image follow-up request | Create WordPress Post | Edit Image |  |
| Edit Image | Edit Image | Prepare image for WP upload | HTTP Request1 | upload media to wp |  |
| upload media to wp | HTTP Request | Upload media to WP | Edit Image | upload image to meta data |  |
| upload image to meta data | HTTP Request | Update media metadata/attach | upload media to wp | set featured image |  |
| set featured image | HTTP Request | Set WP featured image | upload image to meta data | — |  |
| Facebook Image post | Facebook Graph API | Publish image post | Edit Image2 | — |  |
| Telegram Image post | Telegram | Publish image message | Edit Image2 | — |  |
| Create profile image post | LinkedIn | Publish profile image post | Edit Image2 | — |  |
| Create page image post | LinkedIn | Publish page image post | Edit Image2 | — |  |
| Post on Discord Channel | Discord | Publish to Discord channel | Edit Image2 | — |  |
| Do nothing | NoOp | Notification hub fanout | do nothing; Edit Image2 | Send a text message; Create Direct Message; Warning Message; Send a message; Post on message; Rapiwa |  |
| Send a text message | Telegram | Notification message | Do nothing | — |  |
| Create Direct Message | Twitter | Send Twitter DM | Do nothing | — |  |
| Warning Message | Facebook Graph API | Send FB warning/notification | Do nothing | — |  |
| Send a message | Gmail | Send email notification | Do nothing | — |  |
| Post on message | Discord | Send Discord message | Do nothing | — |  |
| Rapiwa | Rapiwa (custom) | Custom action/notification | Do nothing | — |  |
| Sticky Note | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note1 | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note2 | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note3 | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note4 | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note5 | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note6 | Sticky Note | Comment container | — | — | (empty) |
| Sticky Note7 | Sticky Note | Comment container | — | — | (empty) |

---

## 4. Reproducing the Workflow from Scratch

Because most node parameters are blank in the provided JSON, the steps below describe the **required structure** and the **typical minimum configuration** needed to make it functional. You will need to decide field mappings (title/content/captions) based on your desired output format.

### 4.1 Create the intake and routing

1. **Add node:** *Google Drive Trigger* → name it **Get PDF or Images**  
   - Set credentials: Google Drive OAuth2  
   - Configure trigger event (e.g., “File Created”)  
   - Restrict to a folder or apply a query filter to only accept PDFs and images.

2. **Add node:** *Switch* → name it **Route based on PDF or Image**  
   - Add rules based on trigger output field (commonly `mimeType`):
     - Rule 1 (Output 1): `mimeType == "application/pdf"`
     - Rule 2 (Output 2): `mimeType` starts with `"image/"`

3. **Connect:** Get PDF or Images → Route based on PDF or Image

### 4.2 Download file content

4. **Add node:** *Google Drive* → **Download PDF**  
   - Operation: Download  
   - File: use expression from trigger item (file ID)
   - Ensure it outputs **binary** (default is usually `data`)

5. **Add node:** *Google Drive* → **Download Image**  
   - Operation: Download  
   - File: expression from trigger item (file ID)  
   - Ensure it outputs binary

6. **Connect:**  
   - Switch (PDF output) → Download PDF  
   - Switch (image output) → Download Image

### 4.3 Extract text (PDF and image)

7. **Add node:** *Extract From File* → **Extract data from PDF**  
   - Input binary property: the PDF binary field from Download PDF (often `data`)  
   - Output: extracted text field (n8n default depends on node)

8. **Add node:** *OpenAI Chat Model (LangChain)* → **OpenAI Model1**  
   - Credentials: OpenAI API key  
   - Model: **gpt-4.1** (as per workflow title)  
   - Set reasonable max tokens

9. **Add node:** *LangChain LLM Chain* → **Vertex AI extract text**  
   - Attach **OpenAI Model1** as the language model (ai connection)  
   - Provide a prompt template like: “Extract all readable text from this image. Return plain text.”  
   - Map the incoming image binary (you may need to convert to an image URL or base64 depending on node support/version)

10. **Connect:**  
   - Download PDF → Extract data from PDF  
   - Download Image → Vertex AI extract text  
   - OpenAI Model1 (ai_languageModel) → Vertex AI extract text (ai input)

### 4.4 Fanout hub for posting and prompt creation

11. **Add node:** *NoOp* → **do nothing** (text hub)

12. **Connect:**  
   - Extract data from PDF → do nothing  
   - Vertex AI extract text → do nothing

13. **Add social text nodes** (configure credentials + message content):
   - Telegram → **send text post**
   - Twitter → **Create Tweet text (free account)**
   - Twitter → **Create Tweet text (premium account)**
   - LinkedIn → **Create profile text post**
   - LinkedIn → **Create page text post**
   - Facebook Graph API → **Facebook text post**

14. **Connect:** do nothing → each of the text-post nodes in parallel.

15. **Add node:** *OpenAI Chat Model (LangChain)* → **OpenAI Model**  
   - Same credential; model gpt-4.1 (or your choice)

16. **Add node:** *LangChain Agent* → **Expert AI image prompt creator**  
   - Provide system instructions to produce a DALL·E-ready prompt from extracted text  
   - Connect **OpenAI Model** to the agent’s AI input

17. **Connect:**  
   - do nothing → Expert AI image prompt creator  
   - OpenAI Model (ai_languageModel) → Expert AI image prompt creator

### 4.5 Generate image and publish image posts

18. **Add node:** *OpenAI (LangChain)* → **Generate an image**  
   - Operation: generate image  
   - Model: DALL·E (select supported model)  
   - Prompt: from the agent output

19. **Add node:** *Edit Image* → **Edit Image2**  
   - Convert/resize for platform requirements (e.g., 1200×630 for sharing, PNG/JPG)

20. **Connect:** Expert AI image prompt creator → Generate an image → Edit Image2

21. **Add platform image-post nodes** and configure:
   - WordPress → **Create WordPress Post** (title/content/status)
   - Facebook Graph API → **Facebook Image post**
   - Telegram → **Telegram Image post**
   - LinkedIn → **Create profile image post**
   - LinkedIn → **Create page image post**
   - Discord → **Post on Discord Channel**

22. **Connect:** Edit Image2 → each of the nodes above in parallel.

### 4.6 WordPress media upload and featured image chain

23. **Add node:** *HTTP Request* → **HTTP Request1**  
   - Use it to fetch/transform the image or to call WP endpoints after post creation (your design choice).  
   - Authenticate if calling WP.

24. **Add node:** *Edit Image* → **Edit Image**  
   - Prepare image for WP media upload (filename, format, size).

25. **Add node:** *HTTP Request* → **upload media to wp**  
   - POST to `/wp-json/wp/v2/media`  
   - Send binary as multipart/form-data  
   - Set auth (Basic with application password is common)

26. **Add node:** *HTTP Request* → **upload image to meta data**  
   - PATCH/POST to media endpoint to set alt text/caption, etc.  
   - Or attach to post via `post` field.

27. **Add node:** *HTTP Request* → **set featured image**  
   - PATCH the post created in step 21: set `featured_media` to uploaded media ID.

28. **Connect:**  
   - Create WordPress Post → HTTP Request1 → Edit Image → upload media to wp → upload image to meta data → set featured image

### 4.7 Notification hub (optional but present)

29. **Add node:** *NoOp* → **Do nothing** (notification hub)

30. **Connect:**  
   - do nothing → Do nothing  
   - Edit Image2 → Do nothing

31. **Add notification nodes** (configure as desired):
   - Telegram → **Send a text message**
   - Twitter → **Create Direct Message**
   - Facebook Graph API → **Warning Message**
   - Gmail → **Send a message**
   - Discord → **Post on message**
   - Custom node → **Rapiwa** (requires installing the community package)

32. **Connect:** Do nothing → each notification node in parallel.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but their content is empty in the provided workflow JSON. | Documentation/comments are not available from sticky notes. |
| The node named “Vertex AI extract text” is actually a LangChain LLM Chain node in this workflow export; it is not a Google Vertex AI node. | Important to avoid credential/setup confusion. |
| The workflow heavily relies on nodes with empty parameters in the export; to run it, you must define: routing conditions, Drive download mapping, extraction prompts, and the post payloads for each social platform. | Implementation requirement. |
| The “Rapiwa” node is a community/custom node (`n8n-nodes-rapiwa`) and must be installed on the n8n instance. | Otherwise the workflow may fail to import/execute. |