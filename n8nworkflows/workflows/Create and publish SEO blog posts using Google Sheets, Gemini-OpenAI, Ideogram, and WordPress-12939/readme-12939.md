Create and publish SEO blog posts using Google Sheets, Gemini/OpenAI, Ideogram, and WordPress

https://n8nworkflows.xyz/workflows/create-and-publish-seo-blog-posts-using-google-sheets--gemini-openai--ideogram--and-wordpress-12939


# Create and publish SEO blog posts using Google Sheets, Gemini/OpenAI, Ideogram, and WordPress

## 1. Workflow Overview

**Workflow title:** Create and publish SEO blog posts using Google Sheets, Gemini/OpenAI, Ideogram, and WordPress

**Purpose:**  
Automate end-to-end SEO blog publishing: pick topics from Google Sheets, verify client/project is active, research keyword intent, generate an HTML article with controlled link rules, create and upload images (with alt text) to WordPress, ensure a WordPress category exists (create if missing), publish the post with featured image + category, then log the live URL back to Google Sheets. Errors are pushed to Discord.

**Target use cases:**
- Scaled content operations for multiple client sites (“PBN_Website_Access” suggests multi-domain support).
- SEO content production with strict anchor-link placement rules and internal-link enrichment via sitemap crawl.
- Automated WordPress media + post publishing.

### 1.1 Input & Triggers (Scheduling + Topic Queue)
- Runs on a schedule, pulls rows from Google Sheets, loops rows in batches.

### 1.2 Client/Project Validation + Website Access Bootstrap
- Checks if project/client is active; fetches WordPress site access data (base URL, auth token, sitemap URL) from an Apps Script endpoint.

### 1.3 Research + Internal Link Discovery
- AI agent performs SERP-style research and produces an outline/gaps report.
- Sitemap is fetched and parsed to provide candidate internal URLs.

### 1.4 Article Writing + Title/Body Extraction
- AI agent writes the HTML article.
- Code node extracts `<h1>` as title and removes it from body to avoid double-title publishing.

### 1.5 Category Classification + WP Category Resolution
- AI classifies category (from a fixed list), searches WordPress categories, creates category if needed, then standardizes `id` for later publishing.

### 1.6 Image Prompting + Image Generation + WP Media Upload
- AI generates structured prompts + alt text for thumbnail and in-content images.
- Ideogram generates images; workflow downloads them as binary and uploads to WordPress media; sets alt text via WP REST.

### 1.7 Merge + Post Assembly + Publish
- Uploaded media results are merged/aggregated.
- AI agent merges image URLs and article body into final HTML.
- Post is published to WordPress with featured media and category.

### 1.8 Logging + Error Notifications
- Live URL appended to a Google Sheet.
- Discord notifications fire on guarded error paths and a final validation gate.

---

## 2. Block-by-Block Analysis

### Block 1 — Input & Triggers
**Overview:** Starts the workflow daily and loads posting tasks from a Google Sheet, iterating through rows.  
**Nodes involved:** `Schedule_Publish`, `Get_Post_Data`, `Post_Loop`, `Sticky Note4`, `Sticky Note10`

#### Node: Schedule_Publish
- **Type / role:** Schedule Trigger; time-based entrypoint.
- **Config choices:** Runs at **06:00** (interval rule with `triggerAtHour: 6`).
- **Connections:** Outputs to `Get_Post_Data`.
- **Edge cases:** Timezone is instance-dependent; ensure n8n instance timezone matches expectation. Missed runs can occur if instance is down.

#### Node: Get_Post_Data
- **Type / role:** Google Sheets node; fetches rows from a configured sheet/tab.
- **Config choices:** Uses OAuth2 credentials “Sheet Systems”. Sheet/tab and document are placeholders (`your_tab_id`, `your_sheet_id`).
- **Connections:** To `Post_Loop`.
- **Edge cases:** Wrong sheet ID/tab ID, missing permissions, empty sheet, rate limits.

#### Node: Post_Loop
- **Type / role:** Split In Batches; loops through rows.
- **Config choices:** Default batch behavior (batch size not explicitly set in JSON).
- **Connections:**
  - Output 0: not used directly.
  - Output 1: to `get_client_status` (this is typical “continue” output usage).
- **Edge cases:** If batch size is default and sheet is large, runtime may be long. Ensure node is configured as intended in UI (batch size, “reset” semantics).

#### Sticky notes
- **Sticky Note4 content:**  
  “### Input & Triggers  
  - Starts the workflow  
  - Collects initial data”
- **Sticky Note10 content:** (applies broadly to many nodes; see Summary Table duplication)  
  Contains full workflow description + setup steps.

---

### Block 2 — Client/Project Validation + Site Access Bootstrap
**Overview:** Ensures the posting job belongs to an active client/project and fetches per-domain WordPress access details via an external endpoint.  
**Nodes involved:** `get_client_status`, `client_active_check`, `PBN_Website_Access`

#### Node: get_client_status
- **Type / role:** Google Sheets; filters the “client status” sheet to validate project.
- **Config choices:**
  - Filters: `Client ID` equals `Domain ID` from `Get_Post_Data`
  - and `Project Status` equals `"Active"`.
  - Always outputs data enabled (even if empty) to allow IF checks.
- **Connections:** To `client_active_check`.
- **Edge cases:** Filter column names must match sheet headers exactly. If no match, output may be empty but node still returns; downstream must handle it.

#### Node: client_active_check
- **Type / role:** IF node; verifies client exists/active by checking `Client ID` existence.
- **Condition:** String **exists** on `{{$json['Client ID']}}`.
- **Connections:**
  - True: `PBN_Website_Access`
  - False: back to `Post_Loop` (skips inactive jobs and continues loop)
- **Edge cases:** If sheet returns multiple rows, ensure the intended row is used. If field naming differs, condition fails and job is skipped.

#### Node: PBN_Website_Access
- **Type / role:** HTTP Request; calls a Google Apps Script endpoint to retrieve WordPress connection details for the target domain.
- **Config choices:**
  - POST to Apps Script URL.
  - Body params: `api_key` and `domain` from `Post_Loop` → `Posting Website`.
  - Always output data enabled.
- **Expected output fields used later:**
  - `Complete URL` (WordPress base URL, must end with `/` or workflow concatenations must be correct)
  - `Basic_Token` (Basic Auth token)
  - `Sitemap Post URL` (sitemap endpoint)
- **Connections:** To `Do the Research on the Topic`.
- **Edge cases / failures:** Apps Script downtime, invalid API key, wrong domain mapping, malformed base URL causing later WP requests to fail.

---

### Block 3 — Research + Internal Link Discovery
**Overview:** Produces an SEO research report and fetches the site sitemap to extract internal URL candidates.  
**Nodes involved:** `Do the Research on the Topic`, `Google Gemini Chat Model1`, `sitemap_crawl( internal_linking )`, `Sticky Note9`

#### Node: Do the Research on the Topic
- **Type / role:** LangChain Agent; generates a structured research report used to guide writing.
- **Config choices:**
  - Inputs: keyword, landing page, critical client instructions from `get_client_status`.
  - System message: detailed multi-section research requirements (intent, competitors, gaps, outline, sources).
  - Prompt type: “define”.
- **Model connection:** Uses `Google Gemini Chat Model1` via `ai_languageModel`.
- **Connections:** To `sitemap_crawl( internal_linking )`.
- **Edge cases:** LLM may produce very long outputs; keep n8n memory limits in mind. Research may include links that later violate “no brand names” rule in writing if not controlled.

#### Node: Google Gemini Chat Model1
- **Type / role:** LLM provider node (Gemini).
- **Config choices:** Default options; uses credentials “AI_Studio_Content_Posting_”.
- **Connections:** Feeds `Do the Research on the Topic` as its language model.
- **Edge cases:** API quota, model availability, safety filters, transient 429/5xx.

#### Node: sitemap_crawl( internal_linking )
- **Type / role:** HTTP Request; retrieves sitemap XML.
- **Config choices:** URL from `PBN_Website_Access` → `Sitemap Post URL`.
- **Connections:** To `write_content`.
- **Edge cases:** Non-XML sitemap, sitemap index vs urlset, compressed sitemap, blocked by WAF, large sitemap causing memory overhead.

#### Sticky Note9 content:
- “Content Research  
  - Fetches URLs and keywords”

---

### Block 4 — AI Article Writing + Title/Body Extraction
**Overview:** Writes the HTML article with strict anchor/internal link rules, then extracts title and body for downstream category and publishing steps.  
**Nodes involved:** `write_content`, `Google Gemini Chat Model For Content Writing1`, `extract_title_body`, `Sticky Note`

#### Node: write_content
- **Type / role:** LangChain Agent; generates the blog HTML.
- **Config choices:**
  - Inputs:
    - Keyword and landing page (note: references `Get_Post_Data` in the text; should likely be `Post_Loop` for per-row consistency—see edge case).
    - Research from `Do the Research on the Topic`.
    - Internal link candidates: parses sitemap XML with regex `<loc>…</loc>`, takes first 200 URLs.
  - System message enforces:
    - 800–1000 words, simple HTML tags only.
    - Start with a question.
    - Anchor keyword appears 2–3 times but hyperlinked exactly once in first 1–3 paragraphs with a specific `<a>` style.
    - Add 1–2 internal links with different anchor text.
    - Avoid brands, avoid “AI terms”, avoid em dashes and newline breaks, include 2+ FAQs at end.
- **Model connection:** Uses `Google Gemini Chat Model For Content Writing1`.
- **Connections:** To `extract_title_body`.
- **Edge cases:**
  - **Row mismatch risk:** prompt uses `$('Get_Post_Data').item.json.Keyword` but the loop is `Post_Loop`. If multiple rows exist, `Get_Post_Data` may refer to the first item rather than the current batch item. Best practice: use `Post_Loop` consistently.
  - Regex sitemap parsing may fail if sitemap is not in expected format or contains namespaces/newlines.
  - LLM might violate tag constraints or hyperlink rule; no post-generation validator exists besides later merge/publish.

#### Node: Google Gemini Chat Model For Content Writing1
- **Type / role:** LLM provider (Gemini) for writing.
- **Config choices:** Uses same Gemini credentials; default options.
- **Connections:** Provides model to `write_content`.
- **Edge cases:** Same as other Gemini model nodes.

#### Node: extract_title_body
- **Type / role:** Code node; extracts `<h1>` content and removes it from body.
- **Logic summary:**
  - Reads `$input.first().json.output`.
  - Removes `\n` and replaces em dashes `—` with spaces.
  - Extracts first `<h1>…</h1>` as `h1Only`.
  - Removes all `<h1>…</h1>` blocks from the content, returning `excludedH1`.
- **Connections:** To `classify_category`.
- **Edge cases:**
  - If `<h1>` missing or malformed, title becomes empty; downstream category classification and publish title will be empty.
  - If the content contains multiple `<h1>` blocks, all are removed from body.
  - HTML with attributes in `<h1>` is handled (`<h1[^>]*>`), which is good.

#### Sticky Note content:
- “AI Content Generation  
  - Generates blog content”

---

### Block 5 — Category Classification + WP Category Resolution
**Overview:** Classifies the post category from a fixed taxonomy list, checks if it exists in WordPress, creates it if missing, and standardizes the category ID for publishing.  
**Nodes involved:** `classify_category`, `OpenAI Chat Model1`, `Structured Output Parser`, `get_category`, `category_exists_check`, `create_category`, `set_category_id`, `error_guard`, `If Error Existed Then Get Notified`, `a6a39619...` (Discord error node)

#### Node: classify_category
- **Type / role:** LangChain Agent; returns JSON `{ "category": "..." }`.
- **Config choices:**
  - Strict list: `business, tech, travel, health, finance, lifestyle, education, food, sports, entertainment`.
  - Prompt examples include an invalid “general” category, but rule says choose strictly from list; this is a prompt inconsistency.
  - `hasOutputParser: true`.
  - On error: continue regular output; always output data enabled.
- **Model connection:** Uses `OpenAI Chat Model1` (via `ai_languageModel`) and `Structured Output Parser` (via `ai_outputParser`).
- **Input:** Blog title from `extract_title_body` (`{{$json.h1Only}}`).
- **Connections:** To `get_category`.
- **Edge cases:**
  - Prompt conflict (“general” example) may cause output outside allowed list; WP search may return nothing.
  - If parser auto-fix fails, output may be missing `output.category`.

#### Node: OpenAI Chat Model1
- **Type / role:** LLM provider (OpenAI).
- **Config choices:** Model set to `gpt-5-mini`.
- **Connections:** Supplies model to both `classify_category` and `Structured Output Parser` (as configured).
- **Edge cases:** API quota, model availability, policy refusals, transient errors.

#### Node: Structured Output Parser
- **Type / role:** Structured output parser for category JSON.
- **Config choices:** Manual schema: `{"category":""}`, auto-fix enabled.
- **Connections:** Feeds parsed output into `classify_category`.
- **Edge cases:** If model returns code fences or extra keys, parser attempts auto-fix but may still fail.

#### Node: get_category
- **Type / role:** HTTP Request; searches WP categories by name.
- **Config choices:**
  - GET: `wp-json/wp/v2/categories?search={{ $json.output.category }}`
  - Uses Basic Auth header from `PBN_Website_Access.Basic_Token`.
  - On error: continue; always output enabled.
- **Connections:** To `category_exists_check`.
- **Edge cases:**
  - WP returns an array, not an object; checking `$json.id` later is fragile unless n8n item mapping picks the first element. In many cases, WP returns `[{id:...}, ...]` and `$json.id` will be undefined.
  - Auth failures (401), blocked REST API, wrong base URL concatenation.

#### Node: category_exists_check
- **Type / role:** IF node; decides whether category already exists.
- **Condition:** Number **exists** on `{{$json.id}}`.
- **Connections:**
  - True: `set_category_id`
  - False: `create_category`
- **Edge cases:** As above, WP category search typically returns an array; this IF may always go false, causing repeated category creation attempts.

#### Node: create_category
- **Type / role:** HTTP Request; creates a WP category.
- **Config choices:**
  - POST to `wp-json/wp/v2/categories`
  - Body: `name = $('classify_category').item.json.output.category`
  - Auth: Basic token.
  - On error: continue.
- **Connections:** To `set_category_id`.
- **Edge cases:** WP may reject duplicates (term exists), insufficient permissions, invalid category name, REST disabled.

#### Node: set_category_id
- **Type / role:** Set node; normalizes the category ID to `json.id`.
- **Config choices:** Sets `id = {{$json.id}}` (string type).
- **Connections:** To `error_guard`.
- **Edge cases:** If prior node output is array or missing `id`, publishes without category.

#### Node: error_guard
- **Type / role:** IF node; ensures `id` is not empty before proceeding.
- **Condition:** String **notEmpty** on `{{$json.id}}`.
- **Connections:**
  - True: `generate_image_prompt`
  - False: `If Error Existed Then Get Notified` and then `Post_Loop` (continue loop)
- **Edge cases:** Misconfiguration upstream leads to false branch; error message depends on the shape of `$json.error`.

#### Node: If Error Existed Then Get Notified
- **Type / role:** Discord node (webhook); sends error details.
- **Config choices:** Mentions user IDs; includes site URL and `error.*` fields.
- **Connections:** Back to `Post_Loop`.
- **Edge cases:** If the failure payload doesn’t include `$json.error`, message will be empty or expression errors could occur.

#### Node: If Error Existed Then Get Notified (a6a39619…)
- **Type / role:** Another Discord webhook node for error notifications.
- **Status:** Present in workflow but **not connected** in the graph.
- **Risk:** Dead node; may confuse maintainers. If intended, connect from relevant error branch(es).

---

### Block 6 — Image Prompting + Ideogram Generation + WP Media Upload + Alt Text
**Overview:** Generates structured prompts for images, calls Ideogram to create images, downloads as binary, uploads to WordPress media endpoints, and sets SEO alt text.  
**Nodes involved:** `generate_image_prompt`, `Image Prompt Generator1`, `Structured Output Parser2`, `DeepSeek Chat Model2`, `Thumbnail Image Generator1`, `Blog Image Generator1`, `Thumbnail Image Binary Conversion1`, `Blog Image Binary Conversion1`, `Thumbnail Uploading1`, `Blog Image Uploading1`, `Add Alt Text in Thumbnail Image1`, `Add Alt Text in Blog Image1`, `merge_data`, `Sticky Note1`

#### Node: generate_image_prompt
- **Type / role:** LangChain Agent; outputs JSON with prompts and alt text.
- **Config choices:**
  - Inputs: title and content from `extract_title_body`.
  - System message requires 3 images (thumbnail + 2 content images) and strict “NO TEXT”.
  - `hasOutputParser: true`.
- **Model connection:** Uses **Image Prompt Generator1** (Gemini) as LLM and **Structured Output Parser2** as parser (per connections).
- **Connections:** To both `Thumbnail Image Generator1` and `Blog Image Generator1`.
- **Edge cases:**
  - **Schema mismatch:** Parser schema example uses keys like `"Thumbnail Prompt"` and `"Image Prompt 1"`, but the agent’s required output format uses `"Thumbnail_Prompt"` and `"Image_Prompt_1"`, etc. Downstream nodes reference `"Thumbnail Prompt"` / `"Image Prompt 1"` and `"Alt Text ..."` with spaces. This mismatch can break prompt extraction unless auto-fix remaps keys perfectly.
  - It claims 3 images but workflow only generates **thumbnail + one blog image**. No node generates “Content Image 2”.

#### Node: Image Prompt Generator1
- **Type / role:** Gemini Chat Model for image prompt agent.
- **Connections:** Supplies model to `generate_image_prompt`.
- **Edge cases:** Same as other Gemini model nodes.

#### Node: Structured Output Parser2
- **Type / role:** Structured parser for image prompt JSON.
- **Config choices:** Auto-fix enabled; schema example contains:
  - `"Thumbnail Prompt"`, `"Alt Text Thumbnail Image"`, `"Image Prompt 1"`, `"Alt Text Image 1"`
- **Connections:** Feeds parsed output to `generate_image_prompt`.
- **Edge cases:** Key naming inconsistencies are the main risk; parser may not reliably rename underscores vs spaces.

#### Node: DeepSeek Chat Model2
- **Type / role:** DeepSeek LLM provider.
- **Status:** Connected to `Structured Output Parser2` as a language model, but parser is typically not an LLM consumer unless configured that way in this node ecosystem.
- **Risk:** Likely unused/miswired or legacy.
- **Edge cases:** If actually used, credential/quota issues.

#### Node: Thumbnail Image Generator1
- **Type / role:** HTTP Request to Ideogram generate endpoint.
- **Config choices:**
  - POST `https://api.ideogram.ai/v1/ideogram-v3/generate`
  - multipart form-data: `prompt`, `rendering_speed=TURBO`, `resolution=1280x704`
  - Header: `Api-Key: your_api_key` (placeholder; should be a credential)
  - `onError=continueRegularOutput`, `alwaysOutputData=true`
- **Prompt expression:** `$('generate_image_prompt').item.json.output['Thumbnail Prompt']`
- **Connections:** To `Thumbnail Image Binary Conversion1`.
- **Edge cases:** Wrong API key, moderation rejections, response shape changes, empty `data[0].url`.

#### Node: Blog Image Generator1
- **Type / role:** HTTP Request to Ideogram generate endpoint (content image).
- **Prompt expression:** `output['Image Prompt 1']`
- **Connections:** To `Blog Image Binary Conversion1`.
- **Edge cases:** Same as thumbnail generator.

#### Node: Thumbnail Image Binary Conversion1 / Blog Image Binary Conversion1
- **Type / role:** HTTP Request; downloads the generated image URL to binary.
- **Config choices:** URL `{{$json.data[0].url}}`.
- **Connections:** To respective upload nodes.
- **Edge cases:** If Ideogram returns no `data[0].url`, node fails. Large images may hit memory limits.

#### Node: Thumbnail Uploading1 / Blog Image Uploading1
- **Type / role:** HTTP Request; uploads binary to WordPress media endpoint.
- **Config choices:**
  - POST to `{{Complete URL}}wp-json/wp/v2/media`
  - Headers:
    - `Authorization: Basic {{Basic_Token}}`
    - `Content-Disposition: attachment; filename=image.jpg`
  - Content type: `binaryData`, input field `data`
  - `onError=continueRegularOutput`, `alwaysOutputData=true`
  - Blog image upload has `retryOnFail=true`
- **Connections:** To alt-text update nodes.
- **Edge cases:** WP file size limits, mime-type detection, auth permissions, REST media disabled, binary field name mismatch if download node doesn’t set `data`.

#### Node: Add Alt Text in Thumbnail Image1 / Add Alt Text in Blog Image1
- **Type / role:** HTTP Request; updates media `alt_text`.
- **Config choices:**
  - POST to `wp-json/wp/v2/media/{{ $json.id }}`
  - Body: `alt_text = ...`
  - Auth header.
  - Retries enabled for blog image alt text.
- **Alt text expressions:**
  - Thumbnail: `output['Alt Text Thumbnail Image']`
  - Blog image: `output['Alt Text Image 1']`
- **Connections:** Both feed into `merge_data` (different input indexes).
- **Edge cases:** If upload returned error object (because `onError=continue`), `$json.id` may be missing but the node still runs. This later breaks merge/publish.

#### Node: merge_data
- **Type / role:** Merge node; combines thumbnail and blog image media responses.
- **Config choices:** Default merge behavior (not explicitly set).
- **Connections:** To `final_validation_check`.
- **Edge cases:** If one branch returns empty, merge may output unexpected item structure; publish step expects aggregated data later.

#### Sticky Note1 content:
- “Image Generation  
  - Creates blog images”

---

### Block 7 — Final Validation + Aggregation + Blog/Image Merge + Publish
**Overview:** Validates error state, aggregates uploaded media, merges article with image URLs, then publishes post to WordPress.  
**Nodes involved:** `final_validation_check`, `If Error Existed Then Get Notified1`, `Aggregate1`, `Blog and Photo Merge1`, `Google Gemini Chat Model6`, `publish_blog`

#### Node: final_validation_check
- **Type / role:** IF node; checks whether an error exists.
- **Condition:** Number **empty** on `{{$json.error.status}}` (i.e., proceed if no error status).
- **Connections:**
  - True (no error): `Aggregate1`
  - False (error present): `If Error Existed Then Get Notified1`
- **Edge cases:** Some failed nodes may not set `error.status`, so this condition may incorrectly treat failures as success. Consider checking existence of required fields (media IDs) instead.

#### Node: If Error Existed Then Get Notified1
- **Type / role:** Discord webhook; sends error details.
- **Connections:** Back to `Post_Loop`.
- **Edge cases:** Same as other Discord node.

#### Node: Aggregate1
- **Type / role:** Aggregate; aggregates all incoming items.
- **Config choices:** `aggregateAllItemData` (collects multiple items into `data` array).
- **Connections:** To `Blog and Photo Merge1`.
- **Edge cases:** Ordering matters: later `publish_blog` uses `Aggregate1.data[0].id` as featured_media, assuming thumbnail is first. If order flips, a content image could become featured.

#### Node: Blog and Photo Merge1
- **Type / role:** LangChain Agent; merges images + “additional details” into the article body and returns final HTML.
- **Config choices:**
  - Text includes:
    - Blog title from `extract_title_body.h1Only`
    - “Blog Images DATA in JSON” uses `{{$json.data[1].guid.rendered}}` (this appears to reference one uploaded image URL; index assumptions are risky)
    - Current existing article: `extract_title_body.excludedH1`
  - System instructions:
    - Return only HTML (no ```html fences).
    - Thumbnail must be at top before article starts (but it also says thumbnail link not included here—contradiction).
    - Mentions 2nd and 3rd images, but workflow only supplies one content image URL.
- **Model connection:** Uses `Google Gemini Chat Model6`.
- **Connections:** To `publish_blog`.
- **Edge cases:** Indexing into `$json.data[1]` may be wrong if aggregate ordering differs or there are only 2 items. Also, prompt contradictions can produce invalid placement.

#### Node: Google Gemini Chat Model6
- **Type / role:** Gemini LLM provider for merge agent.
- **Connections:** Supplies model to `Blog and Photo Merge1`.
- **Edge cases:** Same Gemini API concerns.

#### Node: publish_blog
- **Type / role:** HTTP Request; publishes WordPress post.
- **Config choices:**
  - POST to `wp-json/wp/v2/posts`
  - form-urlencoded body:
    - `title = extract_title_body.h1Only`
    - `content = $json.output` with an attempt to strip ```html fences:
      - `.replace(/^```html\n/, '').replace(/\n```$/, '')`
    - `status=publish`
    - `featured_media = Aggregate1.data[0].id`
    - `categories[] = set_category_id.id`
  - Basic Auth header.
  - Retries enabled (`retryOnFail=true`, `maxTries=2`), onError continue, always output data.
- **Connections:** To `save_live_url`.
- **Edge cases:**
  - If `featured_media` missing or not numeric, WP may reject or publish without featured image.
  - If `categories[]` wrong type or empty, WP may publish uncategorized.
  - Base URL concatenation includes a trailing space in endpoint string (`"posts "` in JSON). If that exists in node config, it can break request.
  - HTML content might still contain forbidden tags or newlines; WP accepts it but rendering may differ.

---

### Block 8 — Logging + Loop Continuation
**Overview:** Writes the published live URL back to Google Sheets and continues processing remaining rows.  
**Nodes involved:** `save_live_url`

#### Node: save_live_url
- **Type / role:** Google Sheets; append logging row.
- **Config choices:**
  - Operation: append.
  - Maps many columns including Domain ID, Live URLs (`{{$json.guid.rendered}}` from WP post response), timestamp, anchor data.
  - Document ID uses a URL placeholder; sheetName uses an ID placeholder.
  - On error: continue; always output data.
- **Connections:** Back to `Post_Loop` to process next item.
- **Edge cases:** Schema must match sheet columns; removed columns in mapping indicate past edits—ensure target sheet headers align. Timestamp format uses `hh-mm` (12-hour, minutes) not `HH:mm`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note10 | Sticky Note | Global workflow notes & setup | — | — | # Automated Blog Publishing Workflow… (full note content in workflow) |
| Sticky Note4 | Sticky Note | Section label: inputs/triggers | — | — | ### Input & Triggers - Starts the workflow - Collects initial data |
| Schedule_Publish | Schedule Trigger | Start workflow on schedule | — | Get_Post_Data | # Automated Blog Publishing Workflow… |
| Get_Post_Data | Google Sheets | Fetch posting rows | Schedule_Publish | Post_Loop | # Automated Blog Publishing Workflow… |
| Post_Loop | Split In Batches | Iterate through rows | Get_Post_Data; (returns from several nodes) | get_client_status | # Automated Blog Publishing Workflow… |
| get_client_status | Google Sheets | Validate client/project active | Post_Loop | client_active_check | # Automated Blog Publishing Workflow… |
| client_active_check | IF | Skip inactive clients | get_client_status | PBN_Website_Access (true); Post_Loop (false) | # Automated Blog Publishing Workflow… |
| PBN_Website_Access | HTTP Request | Fetch WP base URL/auth/sitemap for domain | client_active_check | Do the Research on the Topic | # Automated Blog Publishing Workflow… |
| Sticky Note9 | Sticky Note | Section label: research | — | — | Content Research - Fetches URLs and keywords |
| Google Gemini Chat Model1 | Gemini Chat Model | LLM for research agent | — | Do the Research on the Topic (model input) | # Automated Blog Publishing Workflow… |
| Do the Research on the Topic | LangChain Agent | Produce SEO research report | PBN_Website_Access | sitemap_crawl( internal_linking ) | Content Research - Fetches URLs and keywords |
| sitemap_crawl( internal_linking ) | HTTP Request | Fetch sitemap XML for internal links | Do the Research on the Topic | write_content | # Automated Blog Publishing Workflow… |
| Sticky Note | Sticky Note | Section label: writing | — | — | AI Content Generation - Generates blog content |
| Google Gemini Chat Model For Content Writing1 | Gemini Chat Model | LLM for writing agent | — | write_content (model input) | AI Content Generation - Generates blog content |
| write_content | LangChain Agent | Generate HTML blog article | sitemap_crawl( internal_linking ) | extract_title_body | AI Content Generation - Generates blog content |
| extract_title_body | Code | Extract H1 title + body without H1 | write_content | classify_category | # Automated Blog Publishing Workflow… |
| OpenAI Chat Model1 | OpenAI Chat Model | LLM for category classification | — | classify_category; Structured Output Parser (model input) | # Automated Blog Publishing Workflow… |
| Structured Output Parser | Structured Output Parser | Parse `{category}` JSON | OpenAI Chat Model1 (model link) | classify_category (parser link) | # Automated Blog Publishing Workflow… |
| classify_category | LangChain Agent | Produce category JSON from title | extract_title_body; OpenAI Chat Model1; Structured Output Parser | get_category | # Automated Blog Publishing Workflow… |
| get_category | HTTP Request | Search WP categories | classify_category | category_exists_check | # Automated Blog Publishing Workflow… |
| category_exists_check | IF | Decide create vs reuse category | get_category | set_category_id (true); create_category (false) | # Automated Blog Publishing Workflow… |
| create_category | HTTP Request | Create WP category | category_exists_check (false) | set_category_id | # Automated Blog Publishing Workflow… |
| set_category_id | Set | Normalize category id | create_category / category_exists_check | error_guard | # Automated Blog Publishing Workflow… |
| error_guard | IF | Stop if category id missing | set_category_id | generate_image_prompt (true); If Error Existed Then Get Notified (false) | # Automated Blog Publishing Workflow… |
| If Error Existed Then Get Notified | Discord | Error notification (category gate) | error_guard (false) | Post_Loop | # Automated Blog Publishing Workflow… |
| If Error Existed Then Get Notified (a6a396…) | Discord | Unused error notifier | — | — | # Automated Blog Publishing Workflow… |
| Sticky Note1 | Sticky Note | Section label: images | — | — | Image Generation - Creates blog images |
| Image Prompt Generator1 | Gemini Chat Model | LLM for image prompt agent | — | generate_image_prompt (model input) | Image Generation - Creates blog images |
| Structured Output Parser2 | Structured Output Parser | Parse image prompt JSON | DeepSeek Chat Model2 (model link) | generate_image_prompt (parser link) | Image Generation - Creates blog images |
| DeepSeek Chat Model2 | DeepSeek Chat Model | LLM provider (possibly unused/miswired) | — | Structured Output Parser2 (model input) | Image Generation - Creates blog images |
| generate_image_prompt | LangChain Agent | Create Ideogram prompts + alt text | error_guard; Image Prompt Generator1; Structured Output Parser2 | Thumbnail Image Generator1; Blog Image Generator1 | Image Generation - Creates blog images |
| Thumbnail Image Generator1 | HTTP Request | Ideogram generate thumbnail | generate_image_prompt | Thumbnail Image Binary Conversion1 | Image Generation - Creates blog images |
| Thumbnail Image Binary Conversion1 | HTTP Request | Download thumbnail binary | Thumbnail Image Generator1 | Thumbnail Uploading1 | Image Generation - Creates blog images |
| Thumbnail Uploading1 | HTTP Request | Upload thumbnail to WP media | Thumbnail Image Binary Conversion1 | Add Alt Text in Thumbnail Image1 | Image Generation - Creates blog images |
| Add Alt Text in Thumbnail Image1 | HTTP Request | Set thumbnail alt_text in WP | Thumbnail Uploading1 | merge_data (input 0) | Image Generation - Creates blog images |
| Blog Image Generator1 | HTTP Request | Ideogram generate content image | generate_image_prompt | Blog Image Binary Conversion1 | Image Generation - Creates blog images |
| Blog Image Binary Conversion1 | HTTP Request | Download content image binary | Blog Image Generator1 | Blog Image Uploading1 | Image Generation - Creates blog images |
| Blog Image Uploading1 | HTTP Request | Upload content image to WP media | Blog Image Binary Conversion1 | Add Alt Text in Blog Image1 | Image Generation - Creates blog images |
| Add Alt Text in Blog Image1 | HTTP Request | Set content image alt_text | Blog Image Uploading1 | merge_data (input 1) | Image Generation - Creates blog images |
| merge_data | Merge | Combine thumbnail+image results | Add Alt Text in Thumbnail Image1; Add Alt Text in Blog Image1 | final_validation_check | # Automated Blog Publishing Workflow… |
| final_validation_check | IF | Route success vs error notification | merge_data | Aggregate1 (true); If Error Existed Then Get Notified1 (false) | # Automated Blog Publishing Workflow… |
| If Error Existed Then Get Notified1 | Discord | Error notification (final gate) | final_validation_check (false) | Post_Loop | # Automated Blog Publishing Workflow… |
| Aggregate1 | Aggregate | Collect media items into array | final_validation_check (true) | Blog and Photo Merge1 | # Automated Blog Publishing Workflow… |
| Google Gemini Chat Model6 | Gemini Chat Model | LLM for final HTML merge | — | Blog and Photo Merge1 (model input) | # Automated Blog Publishing Workflow… |
| Blog and Photo Merge1 | LangChain Agent | Insert images into article HTML | Aggregate1; Google Gemini Chat Model6 | publish_blog | # Automated Blog Publishing Workflow… |
| publish_blog | HTTP Request | Publish WP post | Blog and Photo Merge1 | save_live_url | # Automated Blog Publishing Workflow… |
| save_live_url | Google Sheets | Append published URL log row | publish_blog | Post_Loop | # Automated Blog Publishing Workflow… |

---

## 4. Reproducing the Workflow from Scratch (Manual Build in n8n)

1. **Create workflow** named exactly as desired.

2. **Add Schedule Trigger** node (`Schedule_Publish`)
   - Set run time to **daily at 06:00** (confirm timezone in n8n settings).

3. **Add Google Sheets node** (`Get_Post_Data`)
   - Credentials: Google Sheets OAuth2 (account with access).
   - Operation: read/get many rows (configure as in your environment).
   - Set **Document ID** (sheet) and **Sheet/Tab ID**.
   - Ensure output includes columns at least:
     - `Domain ID`, `Posting Website`, `Keyword`, `Landing Page`

4. **Add Split In Batches** (`Post_Loop`)
   - Connect: `Schedule_Publish → Get_Post_Data → Post_Loop`
   - Configure batch size (recommended explicitly, e.g., 1–10).

5. **Add Google Sheets node** (`get_client_status`)
   - Use a “client status” spreadsheet/tab.
   - Add filters:
     - `Client ID` = `{{$('Get_Post_Data').item.json['Domain ID']}}`
     - `Project Status` = `Active`
   - Enable “Always Output Data”.

6. **Add IF node** (`client_active_check`)
   - Condition: `{{$json['Client ID']}}` **exists**
   - True → continue; False → connect back to `Post_Loop` (to skip).

7. **Add HTTP Request** (`PBN_Website_Access`)
   - POST to your Apps Script (or your own service) that returns:
     - `Complete URL`, `Basic_Token`, `Sitemap Post URL`
   - Send body params:
     - `api_key` (store securely; preferably use n8n credentials/env vars)
     - `domain` = `{{$('Post_Loop').item.json['Posting Website']}}`

8. **Add LangChain Agent** (`Do the Research on the Topic`)
   - Paste the system message from the workflow (intent/competitors/gaps/etc.).
   - In input text, pass:
     - keyword, landing page, and critical instructions field.
   - Add **Gemini Chat Model** node (`Google Gemini Chat Model1`) and connect it as the agent’s **Language Model**.
   - Set Gemini credentials.

9. **Add HTTP Request** (`sitemap_crawl( internal_linking )`)
   - GET `{{$('PBN_Website_Access').item.json['Sitemap Post URL']}}`
   - Connect: Research Agent → sitemap fetch.

10. **Add LangChain Agent** (`write_content`)
    - Use the provided system message (HTML-only, anchor link rule, internal links rule).
    - In input text:
      - Provide keyword, landing page, research output, and internal link opportunities by parsing sitemap response.
    - Add Gemini model node (`Google Gemini Chat Model For Content Writing1`) and attach as Language Model.
    - **Important:** Use `Post_Loop` variables (not `Get_Post_Data`) to avoid row mismatch.

11. **Add Code node** (`extract_title_body`)
    - Use logic:
      - read `output` string from writer
      - extract first `<h1>` to `h1Only`
      - remove `<h1>` from body to `excludedH1`

12. **Add Category classifier agent** (`classify_category`)
    - Create OpenAI Chat Model node (`OpenAI Chat Model1`) with model `gpt-5-mini`.
    - Add Structured Output Parser (`Structured Output Parser`) with schema `{"category":""}` and auto-fix.
    - In the agent prompt, ensure the allowed category list is consistent (remove “general” example or include it deliberately).
    - Connect:
      - `extract_title_body → classify_category`
      - `OpenAI Chat Model1 → classify_category (Language Model)`
      - `Structured Output Parser → classify_category (Output Parser)`

13. **Add HTTP Request** (`get_category`)
    - GET: `{{Complete URL}}wp-json/wp/v2/categories?search={{$json.output.category}}`
    - Header: `Authorization: Basic {{Basic_Token}}`

14. **Add IF** (`category_exists_check`)
    - If category exists:
      - Prefer checking `{{$json[0].id}}` if WP returns array.
    - True → set id; False → create category.

15. **Add HTTP Request** (`create_category`)
    - POST `wp-json/wp/v2/categories`
    - Body: `name={{$json.output.category}}`
    - Auth header.

16. **Add Set node** (`set_category_id`)
    - Set `id` to the resolved category id (ensure you pick correct path: `id` from create response or from first array element on search).

17. **Add IF** (`error_guard`)
    - Condition: `{{$json.id}}` not empty
    - False branch → Discord error notifier → back to `Post_Loop`

18. **Add Image prompt agent** (`generate_image_prompt`)
    - Attach Gemini model (`Image Prompt Generator1`) and a structured output parser (`Structured Output Parser2`).
    - **Ensure schema keys match what downstream uses** (either update downstream expressions to underscore keys or change parser schema to those keys).
    - Connect from `error_guard` true.

19. **Add Ideogram thumbnail generation** (`Thumbnail Image Generator1`)
    - HTTP POST to Ideogram endpoint with multipart fields:
      - `prompt` from image prompt output
      - `rendering_speed=TURBO`
      - `resolution=1280x704`
    - Header `Api-Key` from a secure credential (don’t hardcode).
    - Download the returned `data[0].url` via another HTTP Request node to get binary.

20. **Upload thumbnail to WordPress** (`Thumbnail Uploading1`)
    - POST binary to `wp-json/wp/v2/media`
    - Headers: Basic Auth, Content-Disposition.
    - Then POST alt text update to `wp-json/wp/v2/media/{{id}}`.

21. **Repeat for content image** (`Blog Image Generator1` → download → `Blog Image Uploading1` → alt text update)

22. **Merge image branches** (`merge_data`)
    - Merge thumbnail + content image results.

23. **Add validation gate** (`final_validation_check`)
    - Prefer validating required fields exist (e.g., uploaded media IDs), not just `error.status`.

24. **Aggregate results** (`Aggregate1`)
    - Aggregate all merged items into `data[]`.

25. **Add HTML merge agent** (`Blog and Photo Merge1`)
    - Attach Gemini model (`Google Gemini Chat Model6`).
    - Provide title, article body, and uploaded image URLs to insert into HTML.

26. **Publish to WordPress** (`publish_blog`)
    - POST to `wp-json/wp/v2/posts`
    - Send:
      - `title` = extracted h1
      - `content` = final HTML (strip code fences if necessary)
      - `status=publish`
      - `featured_media` = thumbnail media ID
      - `categories[]` = category id
    - Auth header.

27. **Log live URL** (`save_live_url`)
    - Google Sheets append to your logging sheet.
    - Use WP response `guid.rendered` as Live URL.
    - Connect `save_live_url → Post_Loop` to continue.

28. **Discord notifications**
    - Create Discord Webhook credential.
    - Add Discord nodes for errors and connect them to the relevant false/error paths.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automates the complete blog publishing process from topic research to WordPress publication…” plus setup steps (credentials, sheet setup, WP access, schedule, testing). | Provided in the workflow’s global sticky note (“Automated Blog Publishing Workflow”). |
| Disclaimer (French): “Le texte fourni provient exclusivement d’un workflow automatisé…” | Provided by user; applies to the overall content handling and compliance stance. |