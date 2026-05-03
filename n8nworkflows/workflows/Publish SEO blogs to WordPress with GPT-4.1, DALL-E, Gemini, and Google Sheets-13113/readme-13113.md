Publish SEO blogs to WordPress with GPT-4.1, DALL-E, Gemini, and Google Sheets

https://n8nworkflows.xyz/workflows/publish-seo-blogs-to-wordpress-with-gpt-4-1--dall-e--gemini--and-google-sheets-13113


# Publish SEO blogs to WordPress with GPT-4.1, DALL-E, Gemini, and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Publish SEO blogs to WordPress with GPT-4.1, DALL-E, Gemini, and Google Sheets

**Purpose:**  
This is “Part 2” of a larger automation. It is triggered by a separate blog-writing workflow and takes the finished HTML content + metadata, then:
- Fetches per-client project configuration from a master Google Sheet
- Adds internal links and determines a WordPress category via an AI agent (Gemini)
- Conditionally generates 3 images (1 featured/thumbnail + 2 in-content images) using GPT-4.1 for prompts and DALL·E (gpt-image-1) for image generation
- Uploads images to WordPress Media, inserts them into the article HTML, and publishes the post
- Updates a tracking Google Sheet
- Triggers a client reporting sub-workflow and sends Discord notifications (only in the “no-images” branch in the provided JSON)

### 1.1 Entry & Configuration Lookup
Triggered by another workflow, reads project configuration from a “Project Information Sheet”.

### 1.2 Data Preparation & Internal Linking (AI)
Normalizes input fields, loads internal link keyword targets from Google Sheets, filters valid URLs, aggregates them, and uses an AI agent to insert up to 3 internal links and output a category id.

### 1.3 Conditional Branch: With Images vs No Images
Checks whether “Image Creation” is enabled in project config:
- **Yes:** Generate prompts → create images → upload to WP → assemble final HTML with images → publish → set featured image.
- **No:** Publish directly without images → update sheet → send Discord.

### 1.4 Publishing, Tracking, and Reporting
Publishes post to WordPress (two different “publish without images” mechanisms exist), updates Google Sheets, and triggers a reporting sub-workflow (in the images path).

---

## 2. Block-by-Block Analysis

### Block 2.1 — Entry & Project Configuration
**Overview:** Receives blog payload from an upstream workflow and fetches the client/project settings needed to route and publish correctly.

**Nodes involved:**
- When Executed by Another Workflow
- Fetch Project Configuration
- Prepare Client & Blog Data

#### Node: When Executed by Another Workflow
- **Type / role:** `executeWorkflowTrigger` (entrypoint for sub-workflow execution)
- **Configuration (interpreted):**
  - Declares required inputs: `Client ID`, `Blog S.NO.`, `Blog Title`, `Content`, `OnPage SEO`, `Focus Keyword`.
- **Key variables:**
  - Accessed later via: `$('When Executed by Another Workflow').item.json[...]`
- **Outputs / connections:**
  - Main output → **Fetch Project Configuration**
- **Edge cases / failures:**
  - If upstream workflow omits fields or uses different names (e.g., `Blog S No` vs `Blog S.NO.`), downstream expressions may resolve to empty values.

#### Node: Fetch Project Configuration
- **Type / role:** `googleSheets` (read configuration row(s) for the given client)
- **Configuration:**
  - Document URL is a placeholder (`Your_Sheet_URL`)
  - Sheet ID is a placeholder (`Sheet_ID`)
  - Operation is not explicitly shown in parameters; by default, this node typically “read/get many” depending on UI selection. In practice, it must be configured to fetch the correct client row.
- **Key fields expected downstream (from returned row):**
  - `Website URL`, `Blog API`, `Image Creation`, `Image Instructions`, `Color and Font`, `Discord Channel ID`, `Project Manager Discord ID`, `Categories`, `On Page Sheet`, `Project Information Sheet`, `GMB Name`, `Client ID`
- **Outputs / connections:**
  - → **Prepare Client & Blog Data**
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account 3`)
- **Edge cases / failures:**
  - Wrong sheet URL/permissions → 403/404
  - If multiple rows match a client and node returns multiple items, later nodes may behave unpredictably (taking first item implicitly).
  - Missing trailing slash in `Website URL` causes malformed WP endpoints (see publishing nodes).

#### Node: Prepare Client & Blog Data
- **Type / role:** `set` (normalizes data into consistent field names)
- **Configuration:**
  - Creates fields combining upstream trigger input and config row:
    - `Blog S No` ← trigger `Blog S.NO.`
    - `Website URL` ← config `Website URL`
    - `Blog Title` ← trigger `Blog Title`
    - `Content` ← trigger `Content`
    - `Auth Code` ← config `Blog API` (not actually used as auth later)
    - `OnPage SEO` ← trigger `OnPage SEO`
    - `Focus Keyword` ← trigger `Focus Keyword`
- **Outputs / connections:**
  - → **Fetch Internal Link Keywords**
- **Edge cases / failures:**
  - If `Website URL` does not end with `/`, later expressions build `...comwp-json/...` (missing slash). The workflow currently concatenates `{{ Website URL }}wp-json/...` without inserting `/`.

**Sticky note coverage:**  
“## Data Preparation & Internal Linking …” applies to nodes in this area (Fetch Project Configuration, Prepare Client & Blog Data, Fetch Internal Link Keywords, Filter Valid URLs Only, Combine All Keywords & URLs, Add Internal Links to Content, Gemini AI Model, Parse Content & Category).

---

### Block 2.2 — Internal Link Keyword Loading & Filtering
**Overview:** Loads keyword→URL targets from a per-project sheet, filters rows with valid URLs, and aggregates them into a single structure for the AI internal-linking agent.

**Nodes involved:**
- Fetch Internal Link Keywords
- Filter Valid URLs Only
- Combine All Keywords & URLs

#### Node: Fetch Internal Link Keywords
- **Type / role:** `googleSheets` (reads “TGT Keywords” sheet from per-project sheet URL)
- **Configuration:**
  - DocumentId: `$('Fetch Project Configuration').item.json['Project Information Sheet']`
  - Sheet name: `TGT Keywords`
  - `onError: continueRegularOutput` + `alwaysOutputData: true` to avoid breaking pipeline if sheet read fails.
- **Outputs / connections:**
  - → **Filter Valid URLs Only**
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account 3`)
- **Edge cases / failures:**
  - If `Project Information Sheet` is missing/invalid URL, node will error but continue; downstream may receive empty items and AI will have no link targets.

#### Node: Filter Valid URLs Only
- **Type / role:** `filter` (keeps only rows where `Target Pages` contains `http`)
- **Configuration:**
  - Condition: `$json['Target Pages']` contains `"http"`
- **Outputs / connections:**
  - → **Combine All Keywords & URLs**
- **Edge cases / failures:**
  - URLs without `http` (e.g., `https://` missing due to formatting) will be dropped.
  - If column name differs (`Target Page` vs `Target Pages`) filter yields nothing.

#### Node: Combine All Keywords & URLs
- **Type / role:** `aggregate` (collects all rows into a single item)
- **Configuration:**
  - Aggregate mode: “aggregateAllItemData”
  - Includes specified fields: `Keywords`, `Target Pages`
  - Produces an array at `$json.data` with objects containing those fields.
- **Outputs / connections:**
  - → **Add Internal Links to Content**
- **Edge cases / failures:**
  - If earlier nodes produce no rows, `$json.data` may be empty; later expressions that map over `.data` still work but produce empty lists.

---

### Block 2.3 — AI Internal Linking + Category Selection
**Overview:** Uses a LangChain Agent powered by Gemini to insert internal links into existing HTML without rewriting it, and outputs both updated HTML and a numeric category id.

**Nodes involved:**
- Add Internal Links to Content
- Gemini AI Model
- Parse Content & Category

#### Node: Add Internal Links to Content
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` (LLM agent to modify HTML carefully)
- **Configuration:**
  - Prompt includes:
    - Focus Keyword list from aggregated sheet rows: `{{ $json.data.map(item => item.Keywords).join(', ') }}`
    - Target URLs list: `{{ $json.data.map(item => item["Target Pages"]).join(', ') }}`
    - Category list from config: `{{ $('Fetch Project Configuration').item.json.Categories }}`
    - Original Content from trigger
  - System message enforces:
    - “Do not modify content except inserting `<a href>` tags”
    - Up to 3 internal links, no close anchors, no repeated anchors, not from headings/FAQs
    - Output only final HTML
    - Also “give Category ID … only Number”
  - Has output parser enabled.
- **Model binding:**
  - Receives language model from **Gemini AI Model**
- **Outputs / connections:**
  - Main output → **Check If Image Creation Enabled**
  - Structured parsing output → **Parse Content & Category**
- **Edge cases / failures:**
  - Model may output HTML but fail to include category id consistently; structured parser may fail.
  - HTML-only output requirement conflicts with need to return category number; the workflow addresses this by using a structured parser expecting `{ content, category }`, but the system message says “Output ONLY the final HTML.” This is a logical mismatch that can cause parsing failures.

#### Node: Gemini AI Model
- **Type / role:** `lmChatGoogleGemini` (Gemini chat model provider)
- **Configuration:**
  - `modelName` is expression `=ai_model` (expects a workflow/static variable named `ai_model`)
- **Outputs / connections:**
  - Supplies the model to **Add Internal Links to Content**
- **Credentials:** Google PaLM/Gemini (`_n8ncreatortesting`)
- **Edge cases / failures:**
  - If `ai_model` variable is not defined in n8n (or is not a valid Gemini model name), node fails.

#### Node: Parse Content & Category
- **Type / role:** `outputParserStructured` (forces JSON schema)
- **Configuration:**
  - Schema example:
    - `content: string`
    - `category: number`
- **Outputs / connections:**
  - Parsed data is used later by:
    - **Generate Image Prompts with AI** (`$json.output.content`)
    - **Publish Blog without Image** (no-images publishing branch)
- **Edge cases / failures:**
  - If agent output doesn’t match schema, parse fails. There is no `autoFix` enabled here, so failures can stop execution depending on node error settings.

---

### Block 2.4 — Branching: Image Creation Enabled?
**Overview:** Chooses between full image pipeline + featured-image publishing, or direct publishing without images.

**Nodes involved:**
- Check If Image Creation Enabled

#### Node: Check If Image Creation Enabled
- **Type / role:** `if`
- **Configuration:**
  - Checks: `Fetch Project Configuration` → `Image Creation` equals `"Yes"`
- **Outputs / connections:**
  - **True** → **Generate Image Prompts with AI**
  - **False** → **Publish Blog without Image** (HTTP node)
- **Edge cases / failures:**
  - If `Image Creation` contains `"yes"`/`"TRUE"`/`1`, condition fails. Consider normalizing.
  - If config lookup returns multiple rows, it may read a non-target row’s setting.

---

### Block 2.5 — AI Image Prompt Generation (GPT-4.1-mini)
**Overview:** Generates 3 detailed prompts (thumbnail + two content images) plus alt texts, then splits them into 3 items for batch image generation.

**Nodes involved:**
- Generate Image Prompts with AI
- OpenAI GPT Model
- Parse Image Prompts1
- Split into 3 Image Items
- Process Each Image

#### Node: Generate Image Prompts with AI
- **Type / role:** `langchain.agent` (prompt engineering agent)
- **Configuration:**
  - Inputs: blog title, AI-linked content (`$json.output.content`), image instructions, color/font.
  - System message defines strict requirements (thumbnail layout with overlay text; no text in content images; realistic style).
  - Has output parser enabled.
- **Model binding:**
  - Receives model from **OpenAI GPT Model**
- **Outputs / connections:**
  - Main → **Split into 3 Image Items**
  - Parser output via **Parse Image Prompts1** back into the agent (LangChain parser chain).
- **Edge cases / failures:**
  - Prompt may produce invalid JSON, mitigated by parser `autoFix=true` in Parse Image Prompts1.

#### Node: OpenAI GPT Model
- **Type / role:** `lmChatOpenAi`
- **Configuration:**
  - Model: `gpt-4.1-mini`
- **Outputs / connections:**
  - Supplies LLM to **Generate Image Prompts with AI**
  - Supplies LLM to **Parse Image Prompts1** (for auto-fix behavior)
- **Credentials:** OpenAI (`Open AI - Misc for testing`)
- **Edge cases / failures:**
  - OpenAI quota / 401 / model not available in account.

#### Node: Parse Image Prompts1
- **Type / role:** `outputParserStructured`
- **Configuration:**
  - `autoFix: true`
  - Expects keys:
    - `Thumbnail Prompt`
    - `Alt Text Thumbnail Image`
    - `Image Prompt 1`, `Alt Text Image 1`
    - `Image Prompt 2`, `Alt Text Image 2`
- **Outputs / connections:**
  - Acts as the agent’s structured output parser (LangChain connection).
- **Edge cases / failures:**
  - If agent returns prose without those keys, autoFix attempts repair but can still fail.

#### Node: Split into 3 Image Items
- **Type / role:** `code` (JS transformation)
- **Configuration:**
  - Reads: `$input.all()[0].json.output`
  - Produces 3 items with:
    - `imageType`: `thumbnail|image1|image2`
    - `imagePrompt`
    - `altText`
- **Outputs / connections:**
  - → **Process Each Image**
- **Edge cases / failures:**
  - If `json.output` missing (parser failed), code throws.
  - Assumes exactly one input item.

#### Node: Process Each Image
- **Type / role:** `splitInBatches`
- **Configuration:**
  - Default batch behavior (size not specified in JSON; default is usually 1)
- **Outputs / connections:**
  - Output 1 → **Collect All Image Data** (this is unusual; see note below)
  - Output 2 → **Generate Image with DALL-E**
- **Edge cases / failures:**
  - The current wiring suggests the workflow tries to aggregate in parallel with generation. In practice, you typically:
    1) Generate/upload each image
    2) Store metadata
    3) Aggregate after the loop is complete  
    The provided connections may still work depending on execution order but are fragile.

**Sticky note coverage:**  
“## AI Image Generation Pipeline …” applies to: Generate Image Prompts with AI, Parse Image Prompts1, Split into 3 Image Items, Process Each Image, Generate Image with DALL-E, Upload to WordPress Media, Store Image Metadata, Collect All Image Data.

---

### Block 2.6 — Image Generation (DALL·E) → Upload to WordPress Media → Metadata Aggregation
**Overview:** For each image item, generates an image, uploads it to WordPress Media Library, and stores returned media ID/URL + alt text/type for later insertion.

**Nodes involved:**
- Generate Image with DALL-E
- Upload to WordPress Media
- Store Image Metadata
- Collect All Image Data

#### Node: Generate Image with DALL-E
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` (OpenAI image generation)
- **Configuration:**
  - Resource: `image`
  - Model: `gpt-image-1`
  - Prompt: `{{ $json.imagePrompt }}`
  - Size: `1536x1024`
  - `onError: continueErrorOutput` (continues on errors but emits error output)
- **Outputs / connections:**
  - → **Upload to WordPress Media**
- **Credentials:** OpenAI
- **Edge cases / failures:**
  - If OpenAI returns an error item, downstream Upload may not have binary `data`.
  - Image safety/content policy refusal could occur for certain prompts.

#### Node: Upload to WordPress Media
- **Type / role:** `httpRequest` (uploads binary to WP media endpoint)
- **Configuration:**
  - POST to: `{{ Website URL }}wp-json/wp/v2/media`
  - Body: binary data, field name `data`
  - Headers:
    - `Content-Disposition`: filename derived from blog title with non-alphanumerics removed
    - `Content-Type: image/png`
    - `Authorization: Your_API` (placeholder)
  - `onError: continueRegularOutput`
- **Outputs / connections:**
  - → **Store Image Metadata**
- **Edge cases / failures:**
  - **Website URL concatenation**: missing `/` breaks endpoint.
  - WP auth failures (401/403) if token missing/invalid.
  - WP may reject large payloads; server upload limits.
  - The workflow does not set WP media alt text via API; alt is only stored for later HTML insertion.

#### Node: Store Image Metadata
- **Type / role:** `set`
- **Configuration:**
  - Captures:
    - `id` = WP media id (`$json.id`)
    - `url` = `$json.guid.raw`
    - `type` = current item `imageType`
    - `alt` = current item `altText`
- **Outputs / connections:**
  - → **Process Each Image** (loop continuation)
- **Edge cases / failures:**
  - If upload failed and returned no `guid.raw`, later insertion breaks.

#### Node: Collect All Image Data
- **Type / role:** `aggregate` (collects all metadata into arrays)
- **Configuration:**
  - Aggregates fields: `id`, `url`, `type`, `alt`
- **Outputs / connections:**
  - → **Insert Images into Content**
- **Edge cases / failures:**
  - If pipeline wiring causes this to run before all images complete, you may get partial arrays.
  - Downstream assumes the first media id is the thumbnail: `id[0]`.

---

### Block 2.7 — Content Assembly, Cleanup, and WordPress Publishing (Images Enabled)
**Overview:** Inserts images into the HTML via AI, cleans the resulting output, publishes to WordPress, then sets the featured image.

**Nodes involved:**
- Insert Images into Content
- Gemini Content Assembly Model
- Clean HTML Output
- Publish Blog with Featured Image
- Set Thumbnail as Featured Image

#### Node: Insert Images into Content
- **Type / role:** `langchain.agent` (HTML assembly)
- **Configuration:**
  - Inputs:
    - Blog title
    - “Blog Images DATA in JSON” (but actually passes separate lines: `{{ $json.url }}`, `{{ $json.type }}`, `{{ $json.alt }}` from aggregate arrays)
    - Current article: `$('Add Internal Links to Content').item.json.output.content`
  - System message: insert thumbnail at top; insert 2 images in relevant places; do not remove `<a>` tags; keep FAQs last; add “FAQs” heading if missing.
- **Model binding:**
  - Uses **Gemini Content Assembly Model**
- **Outputs / connections:**
  - → **Clean HTML Output**
- **Edge cases / failures:**
  - The “images data” is not passed as valid JSON object; it’s three separate arrays printed. Model may misinterpret; consider passing a single JSON string like `{images:[...]}`.

#### Node: Gemini Content Assembly Model
- **Type / role:** `lmChatGoogleGemini`
- **Configuration:** modelName `=ai_model`
- **Outputs / connections:** provides model to **Insert Images into Content**
- **Edge cases:** same as other Gemini model node (missing `ai_model` variable).

#### Node: Clean HTML Output
- **Type / role:** `code` (sanitizes LLM output)
- **Configuration:**
  - Reads: `$('Insert Images into Content').first().json.output`
  - Removes:
    - A suspicious JS line if present (`let cleaned = raw.replace(...)`)
    - Markdown code fences ```json / ```html
  - Returns `{ cleanedOutput: cleaned }`
- **Outputs / connections:**
  - → **Publish Blog with Featured Image**
- **Edge cases / failures:**
  - If the agent output is not in `json.output`, code fails.
  - Regex may not remove all unwanted wrappers; HTML could still contain extra text.

#### Node: Publish Blog with Featured Image
- **Type / role:** `httpRequest` (creates WP post)
- **Configuration:**
  - POST: `{{ Website URL }}wp-json/wp/v2/posts`
  - Body:
    - `title`: Blog Title
    - `content`: `{{ $json.cleanedOutput }}`
    - `status`: `publish`
    - `categories`: `{{ $('Add Internal Links to Content').item.json.output.category }}`
  - Header: `Authorization: your_api_key` (placeholder)
- **Outputs / connections:**
  - → **Set Thumbnail as Featured Image**
- **Edge cases / failures:**
  - Category expects numeric id; if AI returns string/non-number WP may reject (400).
  - If WP requires `Content-Type: application/json`, n8n HTTP node handles it, but auth must be correct.

#### Node: Set Thumbnail as Featured Image
- **Type / role:** `httpRequest` (updates WP post)
- **Configuration:**
  - POST: `{{ Website URL }}wp-json/wp/v2/posts/{{ $json.id }}`
  - Query param: `featured_media = {{ $('Collect All Image Data').item.json.id[0] }}`
  - Header: `Authorization: your_api_key`
- **Outputs / connections:**
  - → **Publish Blog without Images** (Google Sheets update node; misnamed)
  - → **Trigger Client Reporting Workflow**
- **Edge cases / failures:**
  - Assumes thumbnail is first aggregated media item. If ordering changes, wrong featured image is set.
  - WP may require body instead of query parameter for updates; query often works but not guaranteed across setups.

**Sticky note coverage:**  
“## Content Assembly & Publishing …” applies to Insert Images into Content, Clean HTML Output, Publish Blog with Featured Image, Set Thumbnail as Featured Image, and the Google Sheets update that follows.

---

### Block 2.8 — No-Images Publishing Branch + Tracking + Discord
**Overview:** If image creation is disabled, publishes the internally-linked content directly to WordPress, updates the tracking sheet, and sends a Discord notification.

**Nodes involved:**
- Publish Blog without Image (HTTP)
- Update Sheet with Publish URL (No Images)
- Send Discord Notification

#### Node: Publish Blog without Image
- **Type / role:** `httpRequest` (creates WP post without generated images)
- **Configuration:**
  - POST: `{{ Website URL }}wp-json/wp/v2/posts`
  - Body:
    - `title`: Blog Title
    - `content`: `{{ $json.output.content }}`
    - `status`: publish
    - `categories`: `{{ $json.output.category }}`
  - Header: Authorization placeholder
  - `onError: continueRegularOutput`
- **Inputs:**
  - Comes from **Check If Image Creation Enabled (false path)**, so `$json.output.*` refers to parsed output from internal-linking stage.
- **Outputs / connections:**
  - → **Update Sheet with Publish URL (No Images)**
- **Edge cases / failures:**
  - If internal-link parser failed, `$json.output` may not exist.
  - Endpoint may be wrong if Website URL missing slash.

#### Node: Update Sheet with Publish URL (No Images)
- **Type / role:** `googleSheets` (update tracking row)
- **Configuration:**
  - DocumentId: `{{ $node['Prepare Client & Blog Data'].json['OnPage SEO'] }}` (uses trigger-provided sheet URL)
  - Sheet name: `Content Requirement & Posting`
  - Operation: `update`
  - Matching column: `row_number`
  - Values:
    - `row_number`: `{{ $node['Prepare Client & Blog Data'].json['Blog S No'] }}`
    - `Publish URLs`: `{{ $json.link }}`
    - `Published Date`: `{{ $now.format('yyyy-MM-dd') }}`
- **Outputs / connections:**
  - → **Send Discord Notification**
- **Credentials:** Google Sheets OAuth2 (`Google Sheets account 2`)
- **Edge cases / failures:**
  - Uses `OnPage SEO` field as a sheet URL; elsewhere a different field `On Page Sheet` is used. Inconsistent configuration can cause updates to land in the wrong spreadsheet.

#### Node: Send Discord Notification
- **Type / role:** `discord` (sends message)
- **Configuration:**
  - Mentions Project Manager: `<@{{ Project Manager Discord ID }}>`
  - Message includes brand name (`GMB Name`) and a publish URL from `{{ $json['Publish URLs'] }}`  
    (Note: the sheet update sets `Publish URLs`, so this works only if the node output includes the updated row fields.)
  - Guild: “Project Delivery”
  - ChannelId: expression from config `Discord Channel ID`
- **Credentials:** Discord Bot (`Nathan - Shan`)
- **Edge cases / failures:**
  - If Discord Channel ID is invalid or bot lacks permission → 403.
  - If `Publish URLs` isn’t present in node output (depending on Google Sheets node behavior), message may contain blank link.

**Sticky note coverage:**  
“## Reporting & Notifications …” applies to Update Sheet with Publish URL (No Images) and Send Discord Notification (and conceptually to Trigger Client Reporting Workflow, though Discord is only wired in the no-images branch).

---

### Block 2.9 — Tracking Sheet Update (Images Path) + Reporting Sub-workflow
**Overview:** After publishing with images and setting featured image, it updates a tracking sheet (node name suggests “without images” but it is used here), and triggers a separate reporting workflow.

**Nodes involved:**
- Publish Blog without Images (Google Sheets)
- Trigger Client Reporting Workflow

#### Node: Publish Blog without Images (Google Sheets)
- **Type / role:** `googleSheets` (update tracking row)
- **Configuration:**
  - DocumentId: `{{ $node['Fetch Project Configuration'].json['On Page Sheet'] }}`
  - Sheet: `Content Requirement & Posting`
  - Matching by `row_number`
  - Values:
    - `row_number`: trigger `Blog S.NO.`
    - `Publish URLs`: `{{ $json.guid.rendered }}`
    - `Published Date`: `{{ new Date($json.date).toISOString().split('T')[0] }}`
- **Inputs:**
  - Comes from **Set Thumbnail as Featured Image**, so `$json` at this node is the WP post update response (must contain `guid.rendered` and `date`).
- **Credentials:** Google Sheets OAuth2 (`Sheet Systems`)
- **Edge cases / failures:**
  - WordPress response fields differ by WP version/plugins; may be `link` rather than `guid.rendered`.
  - The node name is misleading; it does not publish anything—only updates Sheets.

#### Node: Trigger Client Reporting Workflow
- **Type / role:** `executeWorkflow` (invokes reporting manager workflow)
- **Configuration:**
  - Workflow: “Plan>Design>Test — Reporting Manager” (id `vL74JIN20ypJ559f`)
  - Inputs passed:
    - `Post URL`: `{{ $('Set Thumbnail as Featured Image').item.json.link }}`
    - `Client ID`: from config
    - `Task Name`: “Blog Publishing”
    - `Task Brief`: includes Focus Keyword + Blog Title
    - Purpose/Benefits: static text
  - `onError: continueRegularOutput`
- **Outputs / connections:** end of workflow
- **Edge cases / failures:**
  - If the reporting workflow expects different input names/types, it may fail silently (since onError continues).
  - If `link` missing from WP response, Post URL becomes empty.

**Sticky note coverage:**  
The “## Reporting & Notifications …” note applies conceptually to this block as well.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Executed by Another Workflow | executeWorkflowTrigger | Entry point; receives blog payload from upstream workflow | — | Fetch Project Configuration |  |
| Fetch Project Configuration | Google Sheets | Loads per-client project configuration | When Executed by Another Workflow | Prepare Client & Blog Data | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Prepare Client & Blog Data | Set | Normalizes input fields (title/content/url/category inputs) | Fetch Project Configuration | Fetch Internal Link Keywords | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Fetch Internal Link Keywords | Google Sheets | Reads keyword→target URL list | Prepare Client & Blog Data | Filter Valid URLs Only | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Filter Valid URLs Only | Filter | Keeps only rows with valid target URLs | Fetch Internal Link Keywords | Combine All Keywords & URLs | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Combine All Keywords & URLs | Aggregate | Aggregates keywords/URLs into one structure | Filter Valid URLs Only | Add Internal Links to Content | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Add Internal Links to Content | LangChain Agent | Inserts internal links and returns category id + updated HTML | Combine All Keywords & URLs | Check If Image Creation Enabled | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Gemini AI Model | Google Gemini Chat Model | Provides Gemini model to internal linking agent | — | Add Internal Links to Content | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Parse Content & Category | Structured Output Parser | Parses `{content, category}` from internal linking agent | — (LangChain parser connection) | (Used implicitly downstream) | ## Data Preparation & Internal Linking<br><br>Receives blog data from blog writing automation, fetches project settings, loads internal link keywords, filters valid URLs, and uses AI to add 3 natural anchor links to content with proper category assignment |
| Check If Image Creation Enabled | IF | Branching: image pipeline vs no-image publishing | Add Internal Links to Content | Generate Image Prompts with AI; Publish Blog without Image |  |
| Generate Image Prompts with AI | LangChain Agent | Produces 3 image prompts + alt texts | Check If Image Creation Enabled (true) | Split into 3 Image Items | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| OpenAI GPT Model | OpenAI Chat Model | Provides GPT-4.1-mini model for prompt generation/parsing | — | Generate Image Prompts with AI; Parse Image Prompts1 | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Parse Image Prompts1 | Structured Output Parser | Parses prompts/alts JSON (auto-fix enabled) | — (LangChain parser connection) | Generate Image Prompts with AI (parser) | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Split into 3 Image Items | Code | Converts one parsed object into 3 items (thumbnail/image1/image2) | Generate Image Prompts with AI | Process Each Image | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Process Each Image | SplitInBatches | Iterates over 3 image items | Split into 3 Image Items; Store Image Metadata | Collect All Image Data; Generate Image with DALL-E | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Generate Image with DALL-E | OpenAI Image | Generates PNG image from prompt | Process Each Image | Upload to WordPress Media | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Upload to WordPress Media | HTTP Request | Uploads image binary to WP media library | Generate Image with DALL-E | Store Image Metadata | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Store Image Metadata | Set | Stores WP media id/url + image type + alt text | Upload to WordPress Media | Process Each Image | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Collect All Image Data | Aggregate | Aggregates metadata arrays for insertion | Process Each Image | Insert Images into Content | ## AI Image Generation Pipeline<br><br>Generates 3 branded images (thumbnail + 2 content images) using AI prompts, creates them with DALL-E, uploads to WordPress media library with alt text, and collects metadata for content insertion |
| Insert Images into Content | LangChain Agent | Inserts thumbnail+2 images into HTML without removing links | Collect All Image Data | Clean HTML Output | ## Content Assembly & Publishing<br><br>Inserts generated images into content at strategic positions, cleans HTML output, publishes to WordPress with categories and featured image, then updates tracking sheets |
| Gemini Content Assembly Model | Google Gemini Chat Model | Provides Gemini model to content assembly agent | — | Insert Images into Content | ## Content Assembly & Publishing<br><br>Inserts generated images into content at strategic positions, cleans HTML output, publishes to WordPress with categories and featured image, then updates tracking sheets |
| Clean HTML Output | Code | Removes code fences/unwanted lines from LLM output | Insert Images into Content | Publish Blog with Featured Image | ## Content Assembly & Publishing<br><br>Inserts generated images into content at strategic positions, cleans HTML output, publishes to WordPress with categories and featured image, then updates tracking sheets |
| Publish Blog with Featured Image | HTTP Request | Creates WP post with final HTML and categories | Clean HTML Output | Set Thumbnail as Featured Image | ## Content Assembly & Publishing<br><br>Inserts generated images into content at strategic positions, cleans HTML output, publishes to WordPress with categories and featured image, then updates tracking sheets |
| Set Thumbnail as Featured Image | HTTP Request | Updates WP post featured_media to thumbnail media ID | Publish Blog with Featured Image | Publish Blog without Images; Trigger Client Reporting Workflow | ## Content Assembly & Publishing<br><br>Inserts generated images into content at strategic positions, cleans HTML output, publishes to WordPress with categories and featured image, then updates tracking sheets |
| Publish Blog without Image | HTTP Request | (No-image branch) Publishes post without generated images | Check If Image Creation Enabled (false) | Update Sheet with Publish URL (No Images) |  |
| Update Sheet with Publish URL (No Images) | Google Sheets | Updates tracking row with published URL (no-image branch) | Publish Blog without Image | Send Discord Notification | ## Reporting & Notifications<br><br>Updates Google Sheets with live blog URL and publish date, sends Discord notification to project manager, and triggers Reporting Manager for client communication via email/Slack/WhatsApp |
| Send Discord Notification | Discord | Notifies project manager in Discord | Update Sheet with Publish URL (No Images) | — | ## Reporting & Notifications<br><br>Updates Google Sheets with live blog URL and publish date, sends Discord notification to project manager, and triggers Reporting Manager for client communication via email/Slack/WhatsApp |
| Publish Blog without Images | Google Sheets | (Images branch) Updates tracking row after publish+featured set | Set Thumbnail as Featured Image | — | ## Content Assembly & Publishing<br><br>Inserts generated images into content at strategic positions, cleans HTML output, publishes to WordPress with categories and featured image, then updates tracking sheets |
| Trigger Client Reporting Workflow | Execute Workflow | Calls external reporting workflow with post URL and task info | Set Thumbnail as Featured Image | — | ## Reporting & Notifications<br><br>Updates Google Sheets with live blog URL and publish date, sends Discord notification to project manager, and triggers Reporting Manager for client communication via email/Slack/WhatsApp |
| Sticky Note | Sticky Note | Documentation | — | — |  |
| Sticky Note1 | Sticky Note | Documentation | — | — |  |
| Sticky Note2 | Sticky Note | Documentation | — | — |  |
| Sticky Note3 | Sticky Note | Documentation | — | — |  |
| Sticky Note4 | Sticky Note | Documentation | — | — |  |
| Sticky Note5 | Sticky Note | Documentation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: *Publish SEO blogs to WordPress with GPT-4.1, DALL-E, Gemini, and Google Sheets*.

2. **Add Trigger: “When Executed by Another Workflow”**
   - Node: **Execute Workflow Trigger**
   - Define inputs:
     - `Client ID` (string)
     - `Blog S.NO.` (string/number)
     - `Blog Title` (string)
     - `Content` (string, HTML)
     - `OnPage SEO` (string, URL to tracking sheet)
     - `Focus Keyword` (string)

3. **Add Google Sheets: “Fetch Project Configuration”**
   - Credential: Google Sheets OAuth2
   - Set Spreadsheet to your master config sheet (URL)
   - Set sheet/tab containing client rows (the workflow uses placeholders `Your_Sheet_URL` / `Sheet_ID`)
   - Configure it to return the row for the current client (commonly by filtering on `Client ID`).
   - Ensure the output row includes at minimum:
     - `Website URL`, `Image Creation`, `Categories`, `Project Information Sheet`, `Discord Channel ID`, `Project Manager Discord ID`, `GMB Name`, `On Page Sheet`, `Blog API`, `Image Instructions`, `Color and Font`

4. **Add Set node: “Prepare Client & Blog Data”**
   - Map fields using expressions:
     - `Blog S No` = trigger `Blog S.NO.`
     - `Website URL` = config `Website URL`
     - `Blog Title` = trigger `Blog Title`
     - `Content` = trigger `Content`
     - `Auth Code` = config `Blog API` (optional)
     - `OnPage SEO` = trigger `OnPage SEO`
     - `Focus Keyword` = trigger `Focus Keyword`

5. **Add Google Sheets: “Fetch Internal Link Keywords”**
   - Credential: Google Sheets OAuth2
   - Spreadsheet URL: from config `Project Information Sheet`
   - Sheet/tab name: `TGT Keywords`
   - Enable “Continue on Fail” (so the workflow still runs if missing)

6. **Add Filter: “Filter Valid URLs Only”**
   - Condition: `Target Pages` contains `http`

7. **Add Aggregate: “Combine All Keywords & URLs”**
   - Aggregate all items into one
   - Include fields: `Keywords`, `Target Pages`

8. **Add Gemini model node: “Gemini AI Model”**
   - Credential: Google Gemini/PaLM
   - Model name: set a fixed model (or define an n8n variable `ai_model` and reference it)

9. **Add LangChain Agent: “Add Internal Links to Content”**
   - Attach **Gemini AI Model** as its language model
   - Prompt: include focus keywords, target URLs, categories list, and the original HTML content
   - Add a **Structured Output Parser** node (“Parse Content & Category”) with schema:
     - `content` (string)
     - `category` (number)
   - Connect parser to the agent (LangChain output parser connection)

10. **Add IF node: “Check If Image Creation Enabled”**
    - Condition: config field `Image Creation` equals `Yes`
    - True → image pipeline
    - False → no-image publishing path

### Images-enabled path

11. **Add OpenAI model node: “OpenAI GPT Model”**
    - Credential: OpenAI
    - Model: `gpt-4.1-mini`

12. **Add LangChain Agent: “Generate Image Prompts with AI”**
    - Attach **OpenAI GPT Model**
    - Provide blog title, HTML content, image instructions, color/font
    - Add Structured Output Parser (“Parse Image Prompts1”) with `autoFix=true` and keys for 3 prompts + 3 alt texts.

13. **Add Code node: “Split into 3 Image Items”**
    - Transform parsed prompts into 3 items: `thumbnail`, `image1`, `image2`.

14. **Add SplitInBatches: “Process Each Image”**
    - Batch size 1 (typical)
    - Loop through the 3 items

15. **Add OpenAI Image node: “Generate Image with DALL-E”**
    - Model: `gpt-image-1`
    - Prompt from `imagePrompt`
    - Size: `1536x1024`
    - Ensure it outputs binary data in a field you will upload (the JSON expects binary field `data`)

16. **Add HTTP Request: “Upload to WordPress Media”**
    - POST `https://your-site/.../wp-json/wp/v2/media`
    - Send binary body (`data`)
    - Headers:
      - `Authorization: Bearer <token>` or Basic Auth (as required by your WP setup)
      - `Content-Disposition: attachment; filename="...".png`
      - `Content-Type: image/png`

17. **Add Set: “Store Image Metadata”**
    - Store `id`, `guid.raw` (or `source_url` depending on WP response), plus `imageType` and `altText`
    - Route back to **SplitInBatches** “next batch” pattern

18. **Add Aggregate: “Collect All Image Data”**
    - Aggregate all `id/url/type/alt` into arrays once all images processed.

19. **Add Gemini model: “Gemini Content Assembly Model”** (or reuse)
    - Attach to next agent.

20. **Add LangChain Agent: “Insert Images into Content”**
    - Feed it the current HTML and an explicit JSON object describing images (recommended)
    - Instruct it to insert thumbnail at top and two images in relevant positions.

21. **Add Code: “Clean HTML Output”**
    - Remove markdown fences/backticks
    - Output `cleanedOutput`.

22. **Add HTTP Request: “Publish Blog with Featured Image”**
    - POST `.../wp-json/wp/v2/posts`
    - JSON body: `title`, `content`, `status=publish`, `categories=<numeric id>`
    - Save returned `id` and `link`.

23. **Add HTTP Request: “Set Thumbnail as Featured Image”**
    - POST `.../wp-json/wp/v2/posts/{post_id}`
    - Set `featured_media` to thumbnail media id (first item in aggregate)

24. **Add Google Sheets update node** (named however you like)
    - Update the tracking row in `Content Requirement & Posting` with publish URL and date.
    - Use either `link` or `guid.rendered` based on your WP response fields.

25. **Add Execute Workflow: “Trigger Client Reporting Workflow”**
    - Point to your reporting workflow
    - Pass `Client ID`, `Post URL`, and task metadata.

### No-images path

26. **Add HTTP Request: “Publish Blog without Image”**
    - POST WP post with `content` and `categories` from the internal-link parsed output.

27. **Add Google Sheets: “Update Sheet with Publish URL (No Images)”**
    - Update the row by `row_number` with `Publish URLs` and `Published Date`.

28. **Add Discord: “Send Discord Notification”**
    - Credential: Discord bot
    - Guild/channel from config
    - Message includes publish URL and mentions manager discord ID.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Automated Blog Publishing to WordPress** (Part 2) description: receives written blog content from blog creation workflow; optional image creation; internal linking; publishing; updates sheet; notifies via Discord; triggers reporting workflow. | From workflow sticky note “Automated Blog Publishing to WordPress” |
| Setup steps mentioned: add credentials (Google Sheets, WordPress tokens, OpenAI, Discord bot, Gemini); configure project sheet columns; set WP auth tokens on all HTTP nodes; link Part 1 to this workflow; test end-to-end. | From sticky note “Automated Blog Publishing to WordPress” |
| Notable implementation risk: WordPress URLs are concatenated as `{{Website URL}}wp-json/...` and require `Website URL` to end with `/`. | Applies to all WP HTTP nodes |
| Logical mismatch: internal linking agent is instructed to output only HTML, but the workflow also expects a numeric category in structured output. Adjust prompt or parser strategy for reliability. | Applies to “Add Internal Links to Content” + “Parse Content & Category” |