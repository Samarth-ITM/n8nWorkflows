Create WordPress posts and Telegram updates from links with BrowserAct and Gemini

https://n8nworkflows.xyz/workflows/create-wordpress-posts-and-telegram-updates-from-links-with-browseract-and-gemini-12364


# Create WordPress posts and Telegram updates from links with BrowserAct and Gemini

## 1. Workflow Overview

**Title:** Create WordPress posts and Telegram updates from links with BrowserAct and Gemini  
**Workflow name (in JSON):** Auto-generate Wordpress posts and social media updates With BrowserAct

**Purpose:**  
A Telegram-bot-driven publishing pipeline: a user sends a message to a Telegram bot. If the message contains a valid link, the workflow scrapes the target page via BrowserAct, uses Gemini/OpenRouter (Gemini models) to (1) clean and restructure the article into an SEO-ready WordPress HTML post, (2) generate a custom feature image, and (3) publish the post to WordPress and broadcast a Telegram channel photo + caption linking to the new article. If the input is chat or lacks a link, it replies conversationally and/or asks for a URL.

### 1.1 Input Reception & Intent Routing
- Trigger on Telegram message, classify intent into `Article_Request`, `Chat`, or `NoData`, then route.

### 1.2 Article Extraction (BrowserAct)
- For `Article_Request`, run a BrowserAct workflow (template) to extract structured article segments from the URL.

### 1.3 Content Engineering (Article → WordPress + Draft Telegram Copy)
- Convert raw extracted segments into:
  - `web_article` (title + cleaned content),
  - `telegram_post` (draft summary),
  - `images` (unique image URLs).

### 1.4 Visual Pipeline (Analyze scraped visuals → prompt → generate image → upload to WP)
- Analyze scraped images (as references), generate a prompt, generate a new 16:9 image, upload as WordPress media.

### 1.5 SEO HTML Assembly & Publishing
- Build SEO-optimized HTML with image embeds, publish to WordPress, retrieve public post link.

### 1.6 Telegram Channel Broadcast
- Create a final Telegram caption under 1024 chars and post photo+caption to a target channel/chat.

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Trigger + Input Classification + Routing
**Overview:** Receives Telegram messages and classifies them into “article request with link” vs chat vs missing data, then routes execution accordingly.

**Nodes involved:**
- User Sends Message to Bot
- Validate user Input
- Validate inputs (LM)
- Structured Output Parser
- Check For Input Type
- Process Initialization Alert

#### Node: **User Sends Message to Bot**
- **Type/Role:** `telegramTrigger` — workflow entry point for Telegram updates.
- **Configuration:** Listens to `message` updates.
- **Outputs:** Main → **Validate user Input**
- **Failure cases:** Telegram trigger credential issues; webhook not registered; bot not receiving messages.

#### Node: **Validate inputs**
- **Type/Role:** `lmChatGoogleGemini` — provides the LLM to the downstream Agent/Parser chain.
- **Configuration:** Uses Google Gemini credentials (model unspecified here; it’s the “chat model” for the classification agent).
- **Connections:** AI language model → **Validate user Input** and **Structured Output Parser**
- **Failure cases:** Gemini API auth/quota; model availability.

#### Node: **Structured Output Parser**
- **Type/Role:** `outputParserStructured` — forces the classifier to output valid JSON.
- **Config:** `autoFix: true`; schema example: `{"Type":"Article_Request","Link":"extracted_link"}`
- **Connections:** AI output parser → **Validate user Input**
- **Failure cases:** If the LLM output is too malformed, autofix may still fail.

#### Node: **Validate user Input**
- **Type/Role:** `agent` — classifies user text.
- **Key config:**  
  - **Input text:** `{{$json.message.text}}`
  - **System message:** strict rules to output only JSON:
    - If intent is article processing AND contains URL → `{"Type":"Article_Request","Link":"..."}`  
    - If chat → `{"Type":"Chat","Link":"Null"}`
    - If missing link / insufficient → `{"Type":"NoData","Link":"Null"}`
  - `hasOutputParser: true` (uses Structured Output Parser).
- **Outputs:** Main → **Check For Input Type**
- **Edge cases:**
  - URL extraction errors if user posts multiple links or non-HTTP formats.
  - Telegram messages without `text` (e.g., media-only) can break expression usage.

#### Node: **Check For Input Type**
- **Type/Role:** `switch` — routes by `{{$json.output.Type}}`.
- **Rules/Outputs:**
  1. `Article_Request` → **Process Initialization Alert** AND **Extract Data from Target Site**
  2. `Chat` → **Conversational Agent**
  3. `NoData` → **Conversational Agent**
- **Failure cases:** If `output.Type` missing, no branch matches.

#### Node: **Process Initialization Alert**
- **Type/Role:** `telegram` — sends a “please wait” message to the user.
- **Config:**
  - Chat ID: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
  - Text: `Ok, I will do it please give me a moment.`
  - Parse mode: HTML
- **Failure cases:** Missing chat id in trigger payload; Telegram send errors.

---

### Block 2 — Conversational Fallback (Chat / NoData)
**Overview:** If the user is chatting or didn’t provide a link, respond conversationally or request a URL.

**Nodes involved:**
- Chat Model
- Conversational Agent
- Answer the User

#### Node: **Chat Model**
- **Type/Role:** `lmChatGoogleGemini` — language model for the conversational agent.
- **Connections:** AI language model → **Conversational Agent**
- **Failure cases:** Gemini API auth/quota.

#### Node: **Conversational Agent**
- **Type/Role:** `agent` — generates a single plain-text response.
- **Input text:** `INput type : {{ $json.output.Type }} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
- **System message behavior:**
  - If `NoData`: ask for the article link.
  - If `Chat`: respond normally.
  - Output as raw text; no markdown/code blocks/tags.
- **Outputs:** Main → **Answer the User**
- **Edge cases:** If trigger message has no `text`, expression fails.

#### Node: **Answer the User**
- **Type/Role:** `telegram` — sends agent output back to user.
- **Text:** `={{ $json.output }}`
- **Chat ID:** from trigger node
- **Parse mode:** HTML (even though agent is told to avoid tags)
- **Failure cases:** Telegram send errors; if `$json.output` is not a string.

---

### Block 3 — BrowserAct Extraction
**Overview:** For valid links, BrowserAct runs a remote scraping/extraction workflow and returns structured article segments.

**Nodes involved:**
- Extract Data from Target Site

#### Node: **Extract Data from Target Site**
- **Type/Role:** `browserAct` — executes a BrowserAct “WORKFLOW” template.
- **Key config:**
  - `workflowId: "69150243121694958"`
  - Timeout: 7200 seconds
  - Input mapping: `input-Target_Link = {{ $json.output.Link }}`
- **Outputs:** Main → **Analyze Input & Generate Article**
- **Failure cases:**
  - Wrong workflowId or missing template in BrowserAct.
  - BrowserAct timeouts (slow pages, anti-bot, login walls).
  - Extraction schema mismatch (returns unexpected JSON).

---

### Block 4 — Content Engineering (Scraped segments → cleaned article + draft social + image list)
**Overview:** Cleans BrowserAct output, deduplicates, merges into a coherent article, and extracts image URLs.

**Nodes involved:**
- Generate Script (LM)
- Analyze Input & Generate Article
- Structured Output4

#### Node: **Generate Script**
- **Type/Role:** `lmChatGoogleGemini` — LLM for the content-processing agent.
- **Model:** `models/gemini-2.5-pro`
- **Connections:** AI language model → **Analyze Input & Generate Article** and **Structured Output4**
- **Failure cases:** API auth/quota; model availability.

#### Node: **Structured Output4**
- **Type/Role:** `outputParserStructured` — enforces JSON object with keys `web_article`, `telegram_post`, `images`.
- **Config:** `autoFix: true`
- **Connections:** AI output parser → **Analyze Input & Generate Article**
- **Failure cases:** Autofix may fail if the LLM returns non-JSON or truncates.

#### Node: **Analyze Input & Generate Article**
- **Type/Role:** `agent` — transforms raw JSON array of extracted segments into publishable structures.
- **Input text:** `={{ $json.output.string }}`
  - Assumes BrowserAct returns something like `output.string` containing a JSON array string.
- **System message highlights:**
  - Remove duplicates/irrelevant segments
  - Merge body text into a smooth news-style article
  - Produce:
    - `web_article.title`
    - `web_article.content` (multi-paragraph)
    - `telegram_post` (Telegram HTML limited tags)
    - `images` (unique URLs)
- **Outputs (fan-out):**
  - Main → **Images Analayze**
  - Main → **Synchronize Paths**
  - Main → **Synchronize Parallel Executions**
- **Edge cases:**
  - If BrowserAct output isn’t at `$json.output.string`, this breaks.
  - If `images` is empty, later image-analysis steps may fail.
  - Telegram HTML constraints: this produces draft only; later step refines.

---

### Block 5 — Visual Synthesis (Analyze scraped image references → prompt → generate new image → upload to WordPress)
**Overview:** Uses scraped image URLs as reference to analyze visual style, then generates an original feature image prompt and renders a new image via Gemini image model, finally uploading it to WordPress media.

**Nodes involved:**
- Analyze image (LM)
- Images Analayze (agent)
- Structured Output1
- Generate Prompt (LM)
- Generate Image Prompt (agent)
- Structured Output2
- Generate an image
- Upload Image To Wordpress

#### Node: **Analyze image**
- **Type/Role:** `lmChatGoogleGemini` — LLM for thumbnail/visual analysis agent.
- **Model:** `models/gemini-2.5-pro`
- **Connections:** AI language model → **Images Analayze** and **Structured Output1**
- **Failure cases:** API limits; image-URL access issues (hotlink protection).

#### Node: **Structured Output1**
- **Type/Role:** `outputParserStructured` — schema example is a list wrapper; intended to structure the analysis result.
- **Config:** `autoFix: true`
- **Connections:** AI output parser → **Images Analayze**
- **Note:** The schema example (`[{ "output": "the ai output"}]`) does not match how later nodes reference the output (they use `$json.output` directly). Ensure the actual agent output shape matches your downstream expressions.

#### Node: **Images Analayze**
- **Type/Role:** `agent` — “forensic” image description generator.
- **Input text:** A multimodal payload including:
  - Text instruction: “Analyze this image…”
  - `image_url` block using `{{ $json.output.images }}`
- **Outputs:** Main → **Generate Image Prompt**
- **Edge cases:**
  - `{{ $json.output.images }}` must be a URL or list in the exact format expected by the Gemini multimodal node/agent. If `images` is an array, you may need to pick one (e.g., first image).
  - Misspelling in node name (“Analayze”) doesn’t affect function but can confuse maintenance.

#### Node: **Generate Prompt**
- **Type/Role:** `lmChatGoogleGemini` — LLM for prompt engineering agent.
- **Model:** `models/gemini-2.5-pro`
- **Connections:** AI language model → **Generate Image Prompt** and **Structured Output2**

#### Node: **Structured Output2**
- **Type/Role:** `outputParserStructured` — enforces `{ "Prompt": "..." }`
- **Config:** `autoFix: true`
- **Connections:** AI output parser → **Generate Image Prompt**

#### Node: **Generate Image Prompt**
- **Type/Role:** `agent` — generates a single detailed prompt for “Nano Banana Pro” image generation.
- **Input text:**
  - Reference description: `{{ $json.output }}`
  - Main web article: `{{ $('Analyze Input & Generate Article').item.json.output.web_article }}`
- **Constraints encoded:**
  - No text/letters in image
  - Landscape 16:9
  - Style tags (UE5/Octane/8k, etc.)
  - Returns only JSON: `{ "Prompt": "..." }`
- **Outputs:** Main → **Generate an image**
- **Edge cases:** If upstream image analysis output is not plain text at `$json.output`, prompt may degrade.

#### Node: **Generate an image**
- **Type/Role:** `googleGemini` (LangChain Gemini) — image generation.
- **Model:** `models/gemini-3-pro-image-preview`
- **Prompt:** `{{ $json.output.Prompt }}`
- **Output:** Binary property `data`
- **Outputs:** Main → **Upload Image To Wordpress**
- **Failure cases:** Model access not enabled; policy restrictions; prompt invalid; binary output missing.

#### Node: **Upload Image To Wordpress**
- **Type/Role:** `httpRequest` — uploads generated image to WordPress Media Library via REST API.
- **Config highlights:**
  - URL: `https://("YourWordpressAdress.com")/wp-json/wp/v2/media` (placeholder)
  - Method: POST
  - Body: binaryData from field `data`
  - Headers:
    - `Content-Disposition: attachment; filename="upload_<timestamp>.png"`
    - `Content-Type: {{ $json.mimeType }}`
  - Auth: predefined `wordpressApi` credential
  - `allowUnauthorizedCerts: true` (use carefully)
- **Outputs:** Main (index 1) → **Waiting for Required Inputs** and → **Synchronize Parallel Executions**
- **Critical setup requirement:** Replace the placeholder domain with your real WordPress domain.
- **Failure cases:**
  - Wrong WordPress base URL; blocked REST API; invalid credentials/app password.
  - `mimeType` missing; binary field name mismatch.
  - Large upload / server limits.

---

### Block 6 — SEO HTML Assembly (WordPress structure + embed uploaded image link) + Publish
**Overview:** Waits for both article content and uploaded image URL, then generates final HTML and publishes a WordPress post.

**Nodes involved:**
- Synchronize Parallel Executions
- OpenRouter
- Structured Output
- Generate Web Structure
- Publish Post via WordPress

#### Node: **Synchronize Parallel Executions**
- **Type/Role:** `merge` — used to join paths (configured as `chooseBranch`).
- **Role in graph:** Acts as a synchronization point for:
  - Article processing output
  - Image upload output
- **Output:** Main → **Generate Web Structure**
- **Edge cases:** `chooseBranch` does not truly “wait for both”; it forwards data from one branch depending on which input arrives/exists. If you intended a true join, use Merge mode “Combine”/“Wait” patterns (or “Merge by Index/Key”), or redesign with “Wait for both” logic.

#### Node: **OpenRouter**
- **Type/Role:** `lmChatOpenRouter` — LLM provider for the HTML/SEO structuring.
- **Model:** `google/gemini-2.5-pro`
- **Connections:** AI language model → **Generate Web Structure** and **Structured Output**
- **Failure cases:** OpenRouter auth; model routing errors; rate limiting.

#### Node: **Structured Output**
- **Type/Role:** `outputParserStructured` — enforces:
  - `wordpress_title`
  - `wordpress_body_html`
- **Config:** `autoFix: true`
- **Connections:** AI output parser → **Generate Web Structure**

#### Node: **Generate Web Structure**
- **Type/Role:** `agent` — transforms cleaned article into SEO HTML and injects image(s).
- **Input text:**
  - Title/content from `Analyze Input & Generate Article`
  - `image link: {{ $('Upload Image To Wordpress').first().json.link }}`
- **System instructions:**
  - SEO formatting with `<h2>`, `<h3>`, `<p>`, `<ul>`, `<blockquote>`, `<strong>`
  - Insert first image right after first paragraph using `<img ... style="width:100%; height:auto;" />`
  - Output only JSON: `{wordpress_title, wordpress_body_html}`
- **Outputs:** Main → **Publish Post via WordPress**
- **Edge cases:**
  - If image upload response doesn’t include `.json.link`, HTML injection fails.
  - If images list is expected but only one link is passed, distribution rule may be partially unmet.

#### Node: **Publish Post via WordPress**
- **Type/Role:** `wordpress` — creates a WordPress post.
- **Config:**
  - Title: `{{ $json.output.wordpress_title }}`
  - Content: `{{ $json.output.wordpress_body_html }}`
  - `alwaysOutputData: true` (helps downstream even on some non-fatal issues)
- **Outputs:** Main → **Synchronize Paths** (index 1)
- **Failure cases:** WP credentials, permissions, REST API disabled, HTML sanitation, post defaults (status may default to draft/publish depending on node defaults—verify in UI).

---

### Block 7 — Telegram Caption Finalization + Channel Post
**Overview:** After the WordPress post link is known and the image is uploaded, generate a Telegram caption (<=1024 chars) and send photo+caption to a Telegram channel/chat.

**Nodes involved:**
- Synchronize Paths
- OpenRouter1
- Structured Output3
- Generate Telegram Post
- Waiting for Required Inputs
- Send a photo And caption

#### Node: **Synchronize Paths**
- **Type/Role:** `merge` (`chooseBranch`) — intended to combine “article ready” and “WP published link ready” paths.
- **Inputs:**
  - From **Analyze Input & Generate Article** (index 0)
  - From **Publish Post via WordPress** (index 1)
- **Output:** Main → **Generate Telegram Post**
- **Edge cases:** As above, `chooseBranch` may not guarantee both payloads are present.

#### Node: **OpenRouter1**
- **Type/Role:** `lmChatOpenRouter` — LLM provider for Telegram caption generation.
- **Model:** `google/gemini-2.5-pro`
- **Connections:** AI language model → **Generate Telegram Post** and **Structured Output3**

#### Node: **Structured Output3**
- **Type/Role:** `outputParserStructured` — enforces:
  - `{ "telegram_caption": "..." }`
- **Config:** `autoFix: true`
- **Connections:** AI output parser → **Generate Telegram Post**

#### Node: **Generate Telegram Post**
- **Type/Role:** `agent` — creates final Telegram caption.
- **Input text:**
  - Draft: `{{ $('Analyze Input & Generate Article').first().json.output.telegram_post }}`
  - Link: `{{ $('Publish Post via WordPress').first().json.link }}`
- **System instructions:**
  - Telegram HTML only (`<b>`, `<i>`, `<a href="...">`)
  - No unsupported tags; use `\n`
  - Must be <1024 chars (caption limit)
  - “Use emojis” (explicitly requested)
  - Output JSON `{ telegram_caption }`
- **Outputs:** Main → **Waiting for Required Inputs**
- **Failure cases:** Missing WP post `link`; caption exceeds limit.

#### Node: **Waiting for Required Inputs**
- **Type/Role:** `merge` — default merge behavior; used here as a convergence point before sending.
- **Inputs:**
  - From **Generate Telegram Post**
  - From **Upload Image To Wordpress** (index 1)
- **Output:** Main → **Send a photo And caption**
- **Edge cases:** Default merge behavior may not behave as a true “wait for both” depending on execution/data shape. Validate with test runs.

#### Node: **Send a photo And caption**
- **Type/Role:** `telegram` — sends photo post to a channel/chat.
- **Config:**
  - Operation: `sendPhoto`
  - File URL: `{{ $('Upload Image To Wordpress').first().json.media_details.sizes.large.source_url }}`
  - Caption: `{{ $('Generate Telegram Post').first().json.output.telegram_caption }}`
  - Parse mode: HTML
  - **Chat ID:** placeholder string: `parameters.chatId==@Channel ID (Use Channel ID or Chat ID)`
  - `executeOnce: true`
- **Critical setup requirement:** Replace chatId with your actual Telegram Channel ID (often negative numeric ID) or `@channelusername`.
- **Failure cases:**
  - Invalid chatId / bot not admin in channel
  - `large` size missing in WP media response; use `full` or other available size
  - Caption HTML invalid (bad quotes in `<a href>`)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point: receives Telegram messages | — | Validate user Input | ### 🚦 Step 1: Intelligent Routing — The workflow acts as a chat bot first… |
| Validate user Input | langchain.agent | Classify intent + extract link | User Sends Message to Bot; Validate inputs (LM); Structured Output Parser | Check For Input Type | ### 🚦 Step 1: Intelligent Routing — The workflow acts as a chat bot first… |
| Validate inputs | lmChatGoogleGemini | LLM for input classification | — | (AI) Validate user Input; (AI) Structured Output Parser | ### 🚦 Step 1: Intelligent Routing — The workflow acts as a chat bot first… |
| Structured Output Parser | outputParserStructured | Enforce JSON for classifier | Validate inputs (LM) | (AI parser) Validate user Input | ### 🚦 Step 1: Intelligent Routing — The workflow acts as a chat bot first… |
| Check For Input Type | switch | Route: Article_Request vs Chat vs NoData | Validate user Input | Process Initialization Alert + Extract Data from Target Site; Conversational Agent | ### 🚦 Step 1: Intelligent Routing — The workflow acts as a chat bot first… |
| Process Initialization Alert | telegram | Notify user “please wait” | Check For Input Type | — | ### ✍️ Step 2: Content Engineering — Running in parallel… |
| Extract Data from Target Site | browserAct | Scrape/extract content from URL | Check For Input Type | Analyze Input & Generate Article | ## ⚡ Workflow Overview & Setup — Requirements, BrowserAct template, docs links… |
| Generate Script | lmChatGoogleGemini | LLM for content processor | — | (AI) Analyze Input & Generate Article; (AI) Structured Output4 | ### ✍️ Step 2: Content Engineering — Running in parallel… |
| Structured Output4 | outputParserStructured | Enforce article JSON structure | Generate Script (LM) | (AI parser) Analyze Input & Generate Article | ### ✍️ Step 2: Content Engineering — Running in parallel… |
| Analyze Input & Generate Article | langchain.agent | Clean, merge, dedupe; produce article + images + draft telegram | Extract Data from Target Site; Generate Script (LM); Structured Output4 | Images Analayze; Synchronize Paths; Synchronize Parallel Executions | ### ✍️ Step 2: Content Engineering — Running in parallel… |
| Analyze image | lmChatGoogleGemini | LLM for image analysis | — | (AI) Images Analayze; (AI) Structured Output1 | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Structured Output1 | outputParserStructured | Enforce/repair image-analysis output | Analyze image (LM) | (AI parser) Images Analayze | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Images Analayze | langchain.agent | Forensic description of image(s) | Analyze image (LM); Structured Output1; Analyze Input & Generate Article (data) | Generate Image Prompt | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Generate Prompt | lmChatGoogleGemini | LLM for prompt generation | — | (AI) Generate Image Prompt; (AI) Structured Output2 | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Structured Output2 | outputParserStructured | Enforce `{Prompt}` JSON | Generate Prompt (LM) | (AI parser) Generate Image Prompt | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Generate Image Prompt | langchain.agent | Build Nano Banana Pro prompt from article + reference style | Images Analayze; Generate Prompt (LM); Structured Output2 | Generate an image | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Generate an image | googleGemini (image) | Generate new feature image (binary) | Generate Image Prompt | Upload Image To Wordpress | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Upload Image To Wordpress | httpRequest | Upload generated image to WP media | Generate an image | Waiting for Required Inputs; Synchronize Parallel Executions | ACTION REQUIRED: Replace the Google search URL with your live site domain. Find: > ("YourWordpressAddress.com") Replace with: > your-actual-site.com |
| Synchronize Parallel Executions | merge (chooseBranch) | Converge article + upload branches before HTML generation | Analyze Input & Generate Article; Upload Image To Wordpress | Generate Web Structure | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| OpenRouter | lmChatOpenRouter | LLM for WordPress HTML structuring | — | (AI) Generate Web Structure; (AI) Structured Output | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Structured Output | outputParserStructured | Enforce `{wordpress_title, wordpress_body_html}` | OpenRouter | (AI parser) Generate Web Structure | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Generate Web Structure | langchain.agent | SEO HTML formatting + image embedding | Synchronize Parallel Executions; OpenRouter; Structured Output | Publish Post via WordPress | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Publish Post via WordPress | wordpress | Create WP post and get link | Generate Web Structure | Synchronize Paths | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Synchronize Paths | merge (chooseBranch) | Converge for Telegram caption generation | Analyze Input & Generate Article; Publish Post via WordPress | Generate Telegram Post | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| OpenRouter1 | lmChatOpenRouter | LLM for Telegram caption | — | (AI) Generate Telegram Post; (AI) Structured Output3 | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Structured Output3 | outputParserStructured | Enforce `{telegram_caption}` | OpenRouter1 | (AI parser) Generate Telegram Post | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Generate Telegram Post | langchain.agent | Final Telegram caption (<1024 chars) | Synchronize Paths; OpenRouter1; Structured Output3 | Waiting for Required Inputs | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Waiting for Required Inputs | merge | Final convergence before sendPhoto | Generate Telegram Post; Upload Image To Wordpress | Send a photo And caption | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Send a photo And caption | telegram | Post image+caption to channel | Waiting for Required Inputs | — | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Chat Model | lmChatGoogleGemini | LLM for fallback chat | — | (AI) Conversational Agent | ### 💬 Step 2-2: Conversational Fallback — If no link is present… |
| Conversational Agent | langchain.agent | Respond to chat or ask for URL | Check For Input Type; Chat Model | Answer the User | ### 💬 Step 2-2: Conversational Fallback — If no link is present… |
| Answer the User | telegram | Send fallback response to user | Conversational Agent | — | ### 💬 Step 2-2: Conversational Fallback — If no link is present… |
| Documentation | stickyNote | Project notes | — | — | ## ⚡ Workflow Overview & Setup — Requirements, BrowserAct template, docs links… |
| Step 1 Explanation | stickyNote | Comment | — | — | ### 🚦 Step 1: Intelligent Routing — The workflow acts as a chat bot first… |
| Step 2b Explanation | stickyNote | Comment | — | — | ### ✍️ Step 2: Content Engineering — Running in parallel… |
| Step 3 Explanation | stickyNote | Comment | — | — | ### 🖼️ Step 3: Visual Synthesis& Upload — Analyzes scraped visuals… |
| Step 4 Explanation | stickyNote | Comment | — | — | ### 📢 Step 4: Dual-Channel Publishing — Once the WordPress blog post is live… |
| Step 4 Explanation1 | stickyNote | Comment | — | — | ### 💬 Step 2-2: Conversational Fallback — If no link is present… |
| Sticky Note | stickyNote | Action required note | — | — | ACTION REQUIRED: Replace the Google search URL with your live site domain… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it (e.g.) “Auto-generate WordPress posts and social updates with BrowserAct”.

2) **Add Telegram Trigger**
   - Node: **Telegram Trigger**
   - Event: `message`
   - Credentials: Telegram bot token
   - This is the entry node.

3) **Add intent classification chain**
   1. Node: **Google Gemini Chat Model** (LangChain) (name: “Validate inputs”)  
      - Credentials: Google Gemini (PaLM) API  
   2. Node: **Structured Output Parser** (LangChain)  
      - Schema example: `{"Type":"Article_Request","Link":"extracted_link"}`
      - Enable `Auto-fix`
   3. Node: **AI Agent** (LangChain) (name: “Validate user Input”)  
      - Text: `{{$json.message.text}}`  
      - System message: the provided classifier rules (Article_Request/Chat/NoData)  
      - Enable Output Parser and connect it to the structured output parser.
   4. Connect: **Telegram Trigger → Validate user Input**

4) **Add Switch routing**
   - Node: **Switch** (name: “Check For Input Type”)
   - Condition 1: `{{$json.output.Type}} equals "Article_Request"`
   - Condition 2: `{{$json.output.Type}} equals "Chat"`
   - Condition 3: `{{$json.output.Type}} equals "NoData"`
   - Connect: **Validate user Input → Switch**

5) **Article_Request branch: acknowledge user**
   - Node: **Telegram** (send message) (name: “Process Initialization Alert”)
   - Chat ID: `{{ $('User Sends Message to Bot').item.json.message.chat.id }}`
   - Text: “Ok, I will do it please give me a moment.”
   - Connect from Switch output “Article_Request” to this node.

6) **Article_Request branch: BrowserAct extraction**
   - Node: **BrowserAct** (name: “Extract Data from Target Site”)
   - Type: WORKFLOW
   - Workflow ID: `69150243121694958` (or your own)
   - Map input `Target_Link` to `{{ $json.output.Link }}`
   - Timeout: 7200
   - Credentials: BrowserAct API key
   - Connect from Switch output “Article_Request” to this node (in parallel with the alert if desired).

7) **Content processing (clean article + images + draft Telegram post)**
   1. Node: **Google Gemini Chat Model** (name: “Generate Script”)  
      - Model: `models/gemini-2.5-pro`
   2. Node: **Structured Output Parser** (name: “Structured Output4”)  
      - Schema includes: `web_article {title, content}`, `telegram_post`, `images[]`
      - Auto-fix on
   3. Node: **AI Agent** (name: “Analyze Input & Generate Article”)  
      - Text: point to the BrowserAct payload that contains the extracted JSON array string (in this workflow: `{{$json.output.string}}`)
      - System message: “Content Processor and Editor…” (as provided)
      - Has output parser enabled (Structured Output4)
   4. Connect: **BrowserAct → Analyze Input & Generate Article**  
      - Ensure the LLM/Parser connections are correctly wired in the LangChain “AI” ports.

8) **Visual pipeline: analyze scraped image(s) → prompt → generate image**
   1. Add **Gemini Chat Model** (name: “Analyze image”) model `models/gemini-2.5-pro`
   2. Add **Structured Output Parser** (name: “Structured Output1”) (auto-fix on)  
      - Adjust schema to match what you actually want downstream (recommended: output plain text field).
   3. Add **AI Agent** (name: “Images Analayze”)  
      - Provide a multimodal message:
        - text: “Analyze this image in extreme detail.”
        - image url(s): from the article’s `images` list (often you should use `{{ $('Analyze Input & Generate Article').first().json.output.images[0] }}` rather than a whole array)
      - System message: “advanced image analysis… return raw descriptive text”
   4. Add **Gemini Chat Model** (name: “Generate Prompt”) model `models/gemini-2.5-pro`
   5. Add **Structured Output Parser** (name: “Structured Output2”) schema `{ "Prompt": "..." }`
   6. Add **AI Agent** (name: “Generate Image Prompt”)  
      - Input: reference image description + main web article
      - Must output `{Prompt: ...}`
   7. Add **Gemini Image Generation** node (name: “Generate an image”)  
      - Model: `models/gemini-3-pro-image-preview`
      - Prompt: `{{ $json.output.Prompt }}`
      - Output binary property: `data`
   8. Connect: **Analyze Input & Generate Article → Images Analayze → Generate Image Prompt → Generate an image**

9) **Upload generated image to WordPress (media)**
   - Node: **HTTP Request** (name: “Upload Image To Wordpress”)
   - Method: POST  
   - URL: `https://your-actual-site.com/wp-json/wp/v2/media`
   - Authentication: WordPress API credentials (Application Password / basic auth depending on n8n credential type)
   - Send body: binary data, field name `data`
   - Headers:
     - `Content-Disposition: attachment; filename="upload_{{ $now.toMillis() }}.png"`
     - `Content-Type: {{ $json.mimeType }}`
   - Connect: **Generate an image → Upload Image To Wordpress**

10) **Assemble SEO WordPress HTML (needs article + uploaded image link)**
   1. Add **Merge** (name: “Synchronize Parallel Executions”)  
      - Prefer a true join strategy; the current workflow uses `chooseBranch` (may not wait for both).
   2. Add **OpenRouter Chat Model** (name: “OpenRouter”) model `google/gemini-2.5-pro`
   3. Add **Structured Output Parser** (name: “Structured Output”) schema `{wordpress_title, wordpress_body_html}`
   4. Add **AI Agent** (name: “Generate Web Structure”)  
      - Input: article title/content + image link from WP media response (commonly `...json.link`)
      - Output JSON `{wordpress_title, wordpress_body_html}`
   5. Connect:
      - **Analyze Input & Generate Article → Synchronize Parallel Executions**
      - **Upload Image To Wordpress → Synchronize Parallel Executions**
      - **Synchronize Parallel Executions → Generate Web Structure**

11) **Publish to WordPress**
   - Node: **WordPress** (name: “Publish Post via WordPress”)
   - Title: `{{ $json.output.wordpress_title }}`
   - Content: `{{ $json.output.wordpress_body_html }}`
   - Credentials: WordPress
   - Connect: **Generate Web Structure → Publish Post via WordPress**

12) **Generate final Telegram caption (needs WP link + draft telegram text)**
   1. Add **Merge** (name: “Synchronize Paths”)  
      - Inputs from:
        - **Analyze Input & Generate Article**
        - **Publish Post via WordPress**
   2. Add **OpenRouter Chat Model** (name: “OpenRouter1”) model `google/gemini-2.5-pro`
   3. Add **Structured Output Parser** (name: “Structured Output3”) schema `{telegram_caption}`
   4. Add **AI Agent** (name: “Generate Telegram Post”)  
      - Input includes:
        - Draft summary: from `Analyze Input & Generate Article` output
        - Published post link: from WordPress post node output (usually `link`)
      - Enforce <1024 chars and Telegram HTML tags only
   5. Connect: **Synchronize Paths → Generate Telegram Post**

13) **Send Telegram channel photo + caption (needs caption + WP media URL)**
   1. Add **Merge** (name: “Waiting for Required Inputs”)  
      - Inputs: **Generate Telegram Post** and **Upload Image To Wordpress**
   2. Add **Telegram** node (name: “Send a photo And caption”) operation `sendPhoto`
      - Chat ID: set to your channel ID (e.g., `@yourchannel` or numeric ID)
      - File: use WP media URL, e.g. `{{ $('Upload Image To Wordpress').first().json.media_details.sizes.large.source_url }}`
      - Caption: `{{ $('Generate Telegram Post').first().json.output.telegram_caption }}`
      - Parse mode: HTML
   3. Connect: **Waiting for Required Inputs → Send a photo And caption**

14) **Chat/NoData branch response**
   - Add **Gemini Chat Model** (name: “Chat Model”)
   - Add **AI Agent** (name: “Conversational Agent”) with system message rules (ask for link if NoData; respond if Chat)
   - Add **Telegram** (name: “Answer the User”) to send back `{{$json.output}}`
   - Connect Switch outputs “Chat” and “NoData” → **Conversational Agent → Answer the User**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Requirements: Credentials for BrowserAct, WordPress, Telegram, OpenRouter, Google Gemini | From “Documentation” sticky note |
| Mandatory: BrowserAct API template “Telegram and WordPress Post Architect” must exist in your BrowserAct account | From “Documentation” sticky note |
| Docs: How to find BrowserAct API key & Workflow ID | https://docs.browseract.com |
| Docs: How to connect n8n to BrowserAct | https://docs.browseract.com |
| Docs: How to use & customize BrowserAct templates | https://docs.browseract.com |
| ACTION REQUIRED: Replace `("YourWordpressAdress.com")` in the WordPress media upload URL with your real domain | Sticky note near “Upload Image To Wordpress” |
| ACTION REQUIRED: Replace Telegram `chatId` placeholder in “Send a photo And caption” with a real channel/chat id | Node configuration currently contains a placeholder string |
| Merge nodes are configured as `chooseBranch` in key places; this may not truly wait for both branches | Reliability note based on current node configuration |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.