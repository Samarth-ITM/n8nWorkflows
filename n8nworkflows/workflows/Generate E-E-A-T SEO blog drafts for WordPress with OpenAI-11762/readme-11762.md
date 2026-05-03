Generate E-E-A-T SEO blog drafts for WordPress with OpenAI

https://n8nworkflows.xyz/workflows/generate-e-e-a-t-seo-blog-drafts-for-wordpress-with-openai-11762


# Generate E-E-A-T SEO blog drafts for WordPress with OpenAI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate E-E-A-T SEO blog drafts for WordPress with OpenAI  
**Purpose:** Collect blog requirements via an n8n Form, generate SEO metadata + outline (E-E-A-T included) with OpenAI, validate the AI output, write a full HTML article, create a WordPress **draft** post, generate a featured image with OpenAI, upload it to WordPress media, assign it to the draft, then return a success/error response.

**Typical use cases**
- Scaling SEO content production (especially for local/real-estate niches)
- Producing consistent E-E-A-T structured drafts for editorial review
- Automating draft + featured image generation for WordPress

### Logical blocks
**1.1 Input Reception & Normalization**
- Form collects topic/keywords/audience/tone/word count/author credibility notes.
- Set node normalizes fields into predictable keys for downstream expressions.

**1.2 SEO Planning (Metadata + Outline) + Validation Gate**
- OpenAI produces strict JSON: meta title/description, slug, outline, FAQs, E-E-A-T notes, image prompt.
- IF node validates the presence of essential fields; on failure, workflow responds with error and stops.

**1.3 Longform Writing + Packaging**
- OpenAI writes the full article body as clean HTML using the outline/metadata.
- Code node compiles final post object (title, slug, HTML body, prompts).

**1.4 WordPress Draft + Image Generation + Media Linking + Response**
- WordPress node creates a draft post.
- OpenAI image generation creates binary image data.
- HTTP Request uploads image to WP media endpoint.
- HTTP Request updates the post to set the featured image.
- Respond-to-webhook returns success.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Normalization

**Overview:** Collects structured inputs from a hosted n8n form and maps them into standardized JSON keys to reduce downstream expression brittleness.

**Nodes involved**
- **Form** (n8n-nodes-base.formTrigger)
- **Store Values** (n8n-nodes-base.set)

#### Node: Form
- **Type / role:** Form Trigger; entry point that hosts a form and starts the workflow on submission.
- **Key configuration:**
  - **Path:** `plots-eeat-blog` (the form endpoint path inside n8n’s form system)
  - **Title:** “Create E-E-A-T WordPress Blog (Example)”
  - **Response mode:** `responseNode` (the workflow must end with a Respond to Webhook node to reply)
  - **Fields collected:**
    - Topic / Working Title (required)
    - Primary (Focus) Keyword (required)
    - Secondary Keywords (comma-separated)
    - Target Audience (required)
    - Search Intent (required dropdown: Informational, Commercial, Transactional, Local)
    - Tone of Voice (required dropdown: Friendly expert, Professional, Investor focused, Sales-driven)
    - Preferred Word Count (required number)
    - Author / Brand Name (required)
    - Author Credentials / Experience (required textarea)
    - Real Experience / Project Notes (optional textarea)
- **Outputs:** Passes a single item containing form answers under their displayed labels.
- **Potential failures / edge cases:**
  - Missing required fields prevents submission.
  - If field labels change, downstream expressions referencing `Topic / Working Title`, etc., will break.

#### Node: Store Values
- **Type / role:** Set node; normalizes form fields into stable internal keys used throughout the workflow.
- **Key configuration choices:**
  - Creates constants and mapped fields:
    - `WP_URL` = `https://example.com`
    - `topic` = from form “Topic / Working Title”
    - `focus_keyword` = from “Primary (Focus) Keyword”
    - `secondary_keywords` = from “Secondary Keywords (comma-separated)”
    - `target_audience`, `search_intent`, `tone`, `word_count`, `author_name`, `author_experience`, `experience_notes`
- **Key expressions / variables:**
  - Example: `={{ $json['Primary (Focus) Keyword'] }}`
  - Word count stored as **number**: `={{ $json['Preferred Word Count (e.g. 1500)'] }}`
- **Connections:**
  - **Input:** Form
  - **Output:** AI: Create Metadata + Outline
- **Potential failures / edge cases:**
  - Expression errors if form labels change or values are null (less likely due to required fields, except optional fields).
  - If `word_count` is empty or non-numeric, later prompts may degrade or validation logic may fail indirectly.

---

### 2.2 SEO Planning (Metadata + Outline) + Validation Gate

**Overview:** Generates SEO-critical planning artifacts (metadata, slug, outline, FAQs, E-E-A-T signals) in strict JSON and validates minimum viability before proceeding to longform writing.

**Nodes involved**
- **AI: Create Metadata + Outline** (@n8n/n8n-nodes-langchain.openAi)
- **Validate Output** (n8n-nodes-base.if)
- **Respond: Error** (n8n-nodes-base.respondToWebhook)

#### Node: AI: Create Metadata + Outline
- **Type / role:** OpenAI (LangChain) chat node; produces a structured JSON plan.
- **Model:** `gpt-4o-mini`
- **Important configuration choices:**
  - **jsonOutput:** enabled (expects JSON output mapped to `message.content`)
  - **Max tokens:** 2048
  - System-like constraints in prompt:
    - “You MUST return ONLY valid JSON. No markdown, no explanations.”
    - Provides an explicit JSON schema (metaTitle, metaDescription, focusKeyword, secondaryKeywords, h1, slug, outline[], faqQuestions, eeatNotes, imagePrompt)
    - Adds topical constraints: example.com sells BMRDA-approved plots near Mysore Road, Bangalore.
- **Key expressions / variables used:**
  - Injects normalized fields: `{{ $json.topic }}`, `{{ $json.word_count }}`, etc.
- **Connections:**
  - **Input:** Store Values
  - **Output:** Validate Output
- **Potential failures / edge cases:**
  - Model may still return invalid JSON or partially structured output (rare but possible).
  - If the node returns a different structure, downstream references like `$json.message.content.metaTitle` will fail.
  - Token limit may truncate JSON for very large outlines/notes.

#### Node: Validate Output
- **Type / role:** IF node; acts as a quality gate.
- **Validation rules (must all pass):**
  - `metaTitle` is not empty: `={{ $json.message.content.metaTitle }}`
  - `metaDescription` is not empty: `={{ $json.message.content.metaDescription }}`
  - `outline` length ≥ 1: `={{ $json.message.content.outline }}`
- **Connections:**
  - **Input:** AI: Create Metadata + Outline
  - **True output:** AI: Write Full Article
  - **False output:** Respond: Error
- **Potential failures / edge cases:**
  - If `message.content` is not an object (e.g., model returns string), expressions may error or evaluate empty.
  - Outline present but malformed (e.g., not an array) can fail strict type validation.

#### Node: Respond: Error
- **Type / role:** Respond to Webhook; returns a JSON error message to the form submitter.
- **Response body:**  
  `{"message":"❌ There was a problem generating metadata / outline. Please tweak the inputs and try again."}`
- **Connections:**
  - **Input:** Validate Output (false branch)
  - **Output:** ends execution for the webhook response path
- **Potential failures / edge cases:**
  - If workflow errors earlier (before reaching this node), user may not get a clean response depending on n8n error handling configuration.

---

### 2.3 Longform Writing + Packaging

**Overview:** Writes the full HTML body using the validated outline/metadata and then compiles all post fields into one payload for WordPress + image generation.

**Nodes involved**
- **AI: Write Full Article** (@n8n/n8n-nodes-langchain.openAi)
- **Compile Post** (n8n-nodes-base.code)

#### Node: AI: Write Full Article
- **Type / role:** OpenAI (LangChain) chat node; generates the complete article body in HTML.
- **Model:** `gpt-4o`
- **Important configuration choices:**
  - **Max tokens:** 4096
  - Output constraints:
    - “Write the full blog post body in clean HTML only”
    - Allowed tags: `<p>, <h2>, <h3>, <ul>, <li>, <strong>, <em>`
    - Must include FAQ section and an E-E-A-T note block at bottom
    - Must not include meta title/description or JSON in output
  - Injects the metadata JSON from the previous AI node:  
    `{{ $json.message.content | json }}`
  - Pulls additional context from **Store Values** using cross-node expressions:
    - `{{ $('Store Values').first().json.target_audience }}`
    - `{{ $('Store Values').first().json.search_intent }}`
    - etc.
- **Connections:**
  - **Input:** Validate Output (true branch), carrying the metadata JSON in `message.content`
  - **Output:** Compile Post
- **Potential failures / edge cases:**
  - HTML may include disallowed tags or formatting if the model deviates.
  - Output may be shorter/longer than requested word count.
  - If metadata JSON is large, token limits could truncate the article.

#### Node: Compile Post
- **Type / role:** Code node; merges metadata + HTML body into a single normalized object for downstream nodes.
- **Key logic (interpreted):**
  - Reads:
    - `meta` from **AI: Create Metadata + Outline** → `message.content`
    - `body` from **AI: Write Full Article** → `message.content`
  - Returns an item with:
    - `title` = `meta.metaTitle`
    - `h1` = `meta.h1` fallback to `meta.metaTitle`
    - `slug`, `metaDescription`, `focusKeyword`, `secondaryKeywords`, `imagePrompt`
    - `bodyHtml` = generated HTML
- **Connections:**
  - **Input:** AI: Write Full Article
  - **Output:** Create WP Draft
- **Potential failures / edge cases:**
  - If either upstream AI node returns unexpected structure, `.first().json.message.content` may be undefined and throw.
  - If `slug` is empty, WP may auto-generate a slug; could diverge from intended SEO.

---

### 2.4 WordPress Draft + Image Generation + Media Linking + Response

**Overview:** Creates the draft post in WordPress, generates a featured image, uploads it to WP media library, attaches it as the post’s featured image, and returns a success response.

**Nodes involved**
- **Create WP Draft** (n8n-nodes-base.wordpress)
- **Generate Featured Image** (@n8n/n8n-nodes-langchain.openAi) — image resource
- **Upload to WordPress** (n8n-nodes-base.httpRequest)
- **Assign Featured Image** (n8n-nodes-base.httpRequest)
- **Respond: Success** (n8n-nodes-base.respondToWebhook)

#### Node: Create WP Draft
- **Type / role:** WordPress node; creates a new post.
- **Key configuration choices:**
  - **Title:** `={{ $json.h1 }}`
  - **Additional fields:**
    - `slug` = `={{ $json.slug }}`
    - `status` = `draft`
    - `content` = `={{ $json.bodyHtml }}`
- **Credentials:** WordPress credential (typically Application Password or OAuth; sticky notes specify Application Password).
- **Connections:**
  - **Input:** Compile Post
  - **Output:** Generate Featured Image
- **Potential failures / edge cases:**
  - Auth failures (wrong application password, insufficient permissions).
  - WP REST API disabled, blocked by security plugin/WAF, or wrong site URL.
  - If slug conflicts, WP may modify it automatically.

#### Node: Generate Featured Image
- **Type / role:** OpenAI image generation node (LangChain OpenAI resource=image); returns binary image data.
- **Key configuration:**
  - **Prompt:** “Featured image for a blog about: {{ $json.title }}. {{ $json.imagePrompt }}. Realistic, high quality, no text, landscape composition.”
  - **Size:** `1024x1792` (portrait-ish; note: this is not typical “landscape” ratio—prompt says landscape but size is vertical)
- **Connections:**
  - **Input:** Create WP Draft (so it can use compiled post fields)
  - **Output:** Upload to WordPress
- **Potential failures / edge cases:**
  - API quota/limits, content policy blocks, or timeouts.
  - Returned data not in expected binary field name for the next node.

#### Node: Upload to WordPress
- **Type / role:** HTTP Request; uploads binary image to WordPress media endpoint.
- **Key configuration choices:**
  - **POST** `https://example.com/wp-json/wp/v2/media`
  - **Authentication:** predefined credential type `wordpressApi`
  - **Body:** binary (`contentType: binaryData`)
  - **Binary field name:** `data`
  - **Headers:** `Content-Disposition: attachment; filename="featured-image.jpg"`
- **Connections:**
  - **Input:** Generate Featured Image
  - **Output:** Assign Featured Image
- **Potential failures / edge cases:**
  - If the image binary is not in `data`, upload will fail.
  - WordPress may reject upload due to filetype restrictions, size limits, or permissions.
  - Some hosts block `/wp-json/` routes or require additional headers.

#### Node: Assign Featured Image
- **Type / role:** HTTP Request; updates the created post to set the featured media.
- **Key configuration:**
  - **POST** `https://example.com/wp-json/wp/v2/posts/{postId}`
  - `postId` pulled from: `$('Create WP Draft').first().json.id`
  - **Authentication:** predefined credential type `wordpressApi`
  - **Body parameters:** currently configured but effectively empty (`parameters: [ {} ]`)
- **Critical note (integration issue):**
  - To assign the featured image, WordPress expects `featured_media` set to the **media ID** returned by the media upload endpoint.
  - In the current configuration, no `featured_media` field is sent, so the featured image assignment will likely **not** work as intended unless n8n implicitly injects something (unlikely).
- **Connections:**
  - **Input:** Upload to WordPress
  - **Output:** Respond: Success
- **Potential failures / edge cases:**
  - Missing `featured_media` leads to “success” response without actually setting featured image.
  - Auth/permissions issues (edit_posts capability needed).
  - If the post ID is undefined, URL expression breaks.

#### Node: Respond: Success
- **Type / role:** Respond to Webhook; returns JSON success message to the form submitter.
- **Response body:**  
  `{"message":"✅ Draft created on example.com with AI content and featured image."}`
- **Connections:**
  - **Input:** Assign Featured Image
  - **Output:** ends execution for webhook response
- **Potential failures / edge cases:**
  - Message may be misleading if featured image wasn’t actually assigned (see note above).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form | n8n-nodes-base.formTrigger | Entry point form submission | — | Store Values | ## 1.Form & Input Handling<br><br>The workflow starts with a form that collects the topic, keywords, audience, tone, and author experience. The Store Values node standardises this data so every following step receives clean and predictable input. |
| Store Values | n8n-nodes-base.set | Normalize form fields into stable keys | Form | AI: Create Metadata + Outline | ## 1.Form & Input Handling<br><br>The workflow starts with a form that collects the topic, keywords, audience, tone, and author experience. The Store Values node standardises this data so every following step receives clean and predictable input. |
| AI: Create Metadata + Outline | @n8n/n8n-nodes-langchain.openAi | Generate SEO JSON plan (metadata/outline/FAQs/E-E-A-T) | Store Values | Validate Output | ## 2.SEO Planning & Structure<br><br>This section builds the SEO foundation: metadata, headings, article outline, trust signals, and validation checks. The workflow stops early if essential SEO elements are missing. |
| Validate Output | n8n-nodes-base.if | Gatekeeper to ensure minimum viable SEO fields exist | AI: Create Metadata + Outline | AI: Write Full Article (true), Respond: Error (false) | ## 2.SEO Planning & Structure<br><br>This section builds the SEO foundation: metadata, headings, article outline, trust signals, and validation checks. The workflow stops early if essential SEO elements are missing. |
| AI: Write Full Article | @n8n/n8n-nodes-langchain.openAi | Generate full HTML article using outline/metadata | Validate Output (true) | Compile Post | ## 3.Write → Package → Create Draft → Generate Image<br><br>Using the approved structure, the workflow writes the full article, packages all post data, creates a WordPress draft, and generates a featured image so the post is content-complete.<br><br>## IMPORTANT<br><br>Generate an Application Password in WordPress for an administrator user.<br><br>Name your application: n8n-auto-blog  <br>Click Generate.<br><br>You will receive a password like:<br>ABCD EFGH IJKL MNOP<br><br>Save this password and add it to the WordPress credentials in n8n for this workflow to function correctly. |
| Compile Post | n8n-nodes-base.code | Merge metadata + HTML into a single post payload | AI: Write Full Article | Create WP Draft | ## 3.Write → Package → Create Draft → Generate Image<br><br>Using the approved structure, the workflow writes the full article, packages all post data, creates a WordPress draft, and generates a featured image so the post is content-complete.<br><br>## IMPORTANT<br><br>Generate an Application Password in WordPress for an administrator user.<br><br>Name your application: n8n-auto-blog  <br>Click Generate.<br><br>You will receive a password like:<br>ABCD EFGH IJKL MNOP<br><br>Save this password and add it to the WordPress credentials in n8n for this workflow to function correctly. |
| Create WP Draft | n8n-nodes-base.wordpress | Create WordPress post draft | Compile Post | Generate Featured Image | ## 3.Write → Package → Create Draft → Generate Image<br><br>Using the approved structure, the workflow writes the full article, packages all post data, creates a WordPress draft, and generates a featured image so the post is content-complete.<br><br>## IMPORTANT<br><br>Generate an Application Password in WordPress for an administrator user.<br><br>Name your application: n8n-auto-blog  <br>Click Generate.<br><br>You will receive a password like:<br>ABCD EFGH IJKL MNOP<br><br>Save this password and add it to the WordPress credentials in n8n for this workflow to function correctly. |
| Generate Featured Image | @n8n/n8n-nodes-langchain.openAi | Generate featured image binary via OpenAI | Create WP Draft | Upload to WordPress | ## 3.Write → Package → Create Draft → Generate Image<br><br>Using the approved structure, the workflow writes the full article, packages all post data, creates a WordPress draft, and generates a featured image so the post is content-complete.<br><br>## IMPORTANT<br><br>Generate an Application Password in WordPress for an administrator user.<br><br>Name your application: n8n-auto-blog  <br>Click Generate.<br><br>You will receive a password like:<br>ABCD EFGH IJKL MNOP<br><br>Save this password and add it to the WordPress credentials in n8n for this workflow to function correctly. |
| Upload to WordPress | n8n-nodes-base.httpRequest | Upload generated image to WP media library | Generate Featured Image | Assign Featured Image | ## 4.Image Upload, Linking & Final Response<br><br>The featured image is uploaded to WordPress and assigned to the post. A clear success or error response is returned so the user always knows the final result. |
| Assign Featured Image | n8n-nodes-base.httpRequest | Update post to set featured media | Upload to WordPress | Respond: Success | ## 4.Image Upload, Linking & Final Response<br><br>The featured image is uploaded to WordPress and assigned to the post. A clear success or error response is returned so the user always knows the final result. |
| Respond: Success | n8n-nodes-base.respondToWebhook | Return success JSON to form submitter | Assign Featured Image | — | ## 4.Image Upload, Linking & Final Response<br><br>The featured image is uploaded to WordPress and assigned to the post. A clear success or error response is returned so the user always knows the final result. |
| Respond: Error | n8n-nodes-base.respondToWebhook | Return error JSON to form submitter | Validate Output (false) | — | ## 2.SEO Planning & Structure<br><br>This section builds the SEO foundation: metadata, headings, article outline, trust signals, and validation checks. The workflow stops early if essential SEO elements are missing. |
| 📝 Blog Form | n8n-nodes-base.stickyNote | Documentation / context (non-executing) | — | — | ## About the Workflow<br><br>This workflow automatically creates a complete, SEO-ready WordPress blog post using AI  from idea to published draft.<br><br>You fill out a simple form with your topic, keywords, audience, tone, word count, and author details. The workflow then plans the article, generates SEO metadata, writes the content in clean HTML, creates a featured image, and publishes everything as a WordPress draft.<br><br>Its goal is to save time while maintaining high standards for SEO quality, structure, and E-E-A-T. The result is a fully packaged blog post ready for review and publishing.<br><br>Ideal for:<br>• SEO teams  <br>• Content writers and agencies  <br>• Real estate and local business websites  <br>• Website owners scaling content production  <br><br>Before running the workflow, generate a WordPress Application Password for an administrator user and store it in the WordPress credentials in n8n. Once set up, the process runs end-to-end with no manual publishing steps. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Documentation / block label (non-executing) | — | — | ## 1.Form & Input Handling<br><br>The workflow starts with a form that collects the topic, keywords, audience, tone, and author experience. The Store Values node standardises this data so every following step receives clean and predictable input. |
| Sticky Note17 | n8n-nodes-base.stickyNote | Documentation / block label (non-executing) | — | — | ## 2.SEO Planning & Structure<br><br>This section builds the SEO foundation: metadata, headings, article outline, trust signals, and validation checks. The workflow stops early if essential SEO elements are missing. |
| Sticky Note18 | n8n-nodes-base.stickyNote | Documentation / block label (non-executing) | — | — | ## 3.Write → Package → Create Draft → Generate Image<br><br>Using the approved structure, the workflow writes the full article, packages all post data, creates a WordPress draft, and generates a featured image so the post is content-complete.<br><br>## IMPORTANT<br><br>Generate an Application Password in WordPress for an administrator user.<br><br>Name your application: n8n-auto-blog  <br>Click Generate.<br><br>You will receive a password like:<br>ABCD EFGH IJKL MNOP<br><br>Save this password and add it to the WordPress credentials in n8n for this workflow to function correctly. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Documentation / block label (non-executing) | — | — | ## 4.Image Upload, Linking & Final Response<br><br>The featured image is uploaded to WordPress and assigned to the post. A clear success or error response is returned so the user always knows the final result. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n.

2) **Add trigger: “Form”**
   - Node type: **Form Trigger**
   - Path: `plots-eeat-blog`
   - Title: “Create E-E-A-T WordPress Blog (Example)”
   - Description: “Fill this form to generate an SEO + E-E-A-T optimized blog post for example.com.”
   - Add fields (matching labels exactly, because expressions will reference them):
     - Topic / Working Title (required)
     - Primary (Focus) Keyword (required)
     - Secondary Keywords (comma-separated)
     - Target Audience (who is this for?) (required)
     - Search Intent (dropdown, required): Informational/Commercial/Transactional/Local
     - Tone of Voice (dropdown, required): Friendly expert/Professional/Investor focused/Sales-driven
     - Preferred Word Count (number, required)
     - Author / Brand Name (required)
     - Author Credentials / Experience (for E-E-A-T) (textarea, required)
     - Real Experience / Project Notes (bullet style) (textarea, optional)
   - Set **Response mode** to **Response node**.

3) **Add “Store Values” (Set node)** and connect **Form → Store Values**
   - Add fields:
     - `WP_URL` (string): `https://example.com`
     - `topic` (string): expression referencing the form label
     - `focus_keyword`, `secondary_keywords`, `target_audience`, `search_intent`, `tone`
     - `word_count` (number)
     - `author_name`, `author_experience`, `experience_notes`

4) **Add “AI: Create Metadata + Outline”** and connect **Store Values → AI: Create Metadata + Outline**
   - Node type: **OpenAI (LangChain)**
   - Model: `gpt-4o-mini`
   - Enable **JSON output**
   - Max tokens: `2048`
   - Message content: paste a strict instruction to return ONLY JSON with keys:
     - `metaTitle`, `metaDescription`, `focusKeyword`, `secondaryKeywords`, `h1`, `slug`, `outline[]`, `faqQuestions[]`, `eeatNotes{...}`, `imagePrompt`
   - Inject variables using expressions from Store Values (topic, keywords, audience, intent, tone, word_count, author fields).

5) **Add “Validate Output” (IF node)** and connect **AI: Create Metadata + Outline → Validate Output**
   - Conditions (AND):
     - `metaTitle` not empty
     - `metaDescription` not empty
     - `outline` array length ≥ 1
   - True path continues; False path returns error.

6) **Add “Respond: Error”** and connect **Validate Output (false) → Respond: Error**
   - Node type: **Respond to Webhook**
   - Respond with: JSON
   - Body: error message string (as in the workflow).

7) **Add “AI: Write Full Article”** and connect **Validate Output (true) → AI: Write Full Article**
   - Node type: **OpenAI (LangChain)**
   - Model: `gpt-4o`
   - Max tokens: `4096`
   - Prompt requirements:
     - Output **HTML only**, no wrapper tags
     - Use the metadata JSON (insert from the Validate Output input item)
     - Pull `target_audience`, `search_intent`, `tone`, etc. from **Store Values** via `$('Store Values').first().json...`
     - Include FAQ section and E-E-A-T note block
     - Restrict HTML tags to allowed list

8) **Add “Compile Post” (Code node)** and connect **AI: Write Full Article → Compile Post**
   - In JS:
     - Read metadata from **AI: Create Metadata + Outline** (`.first().json.message.content`)
     - Read body HTML from **AI: Write Full Article** (`.first().json.message.content`)
     - Return one item with `h1`, `slug`, `bodyHtml`, `imagePrompt`, etc.

9) **Configure WordPress credentials in n8n**
   - In WordPress admin: **Users → Profile → Application Passwords**
   - Create an application password (e.g., `n8n-auto-blog`)
   - In n8n: create **WordPress API** credentials (Application Password-based, depending on your n8n credential type).
   - Ensure the WP user has rights to create/edit posts and upload media.

10) **Add “Create WP Draft” (WordPress node)** and connect **Compile Post → Create WP Draft**
   - Operation: create post
   - Title: `{{$json.h1}}`
   - Content: `{{$json.bodyHtml}}`
   - Status: `draft`
   - Slug: `{{$json.slug}}`
   - Select the WordPress credentials created above.

11) **Add “Generate Featured Image” (OpenAI Image)** and connect **Create WP Draft → Generate Featured Image**
   - Resource: **Image**
   - Prompt: use `{{$json.title}}` and `{{$json.imagePrompt}}`
   - Size: `1024x1792` (or adjust to a true landscape ratio if desired)

12) **Add “Upload to WordPress” (HTTP Request)** and connect **Generate Featured Image → Upload to WordPress**
   - Method: POST
   - URL: `https://example.com/wp-json/wp/v2/media`
   - Authentication: **WordPress API** predefined credential
   - Send body: **binary**
   - Binary property: `data` (must match where the image node stores the binary)
   - Header: `Content-Disposition: attachment; filename="featured-image.jpg"`

13) **Add “Assign Featured Image” (HTTP Request)** and connect **Upload to WordPress → Assign Featured Image**
   - Method: POST
   - URL: `https://example.com/wp-json/wp/v2/posts/{{$('Create WP Draft').first().json.id}}`
   - Authentication: WordPress API credential
   - Body: **must include** `featured_media` = the uploaded media ID.
     - Use expression referencing Upload to WordPress response ID (commonly `{{$json.id}}` if the media endpoint returns `{ id: ... }`).
     - Example body field: `featured_media: {{$('Upload to WordPress').first().json.id}}`
   - (This is the key adjustment needed for reliable featured image assignment.)

14) **Add “Respond: Success”** and connect **Assign Featured Image → Respond: Success**
   - Node type: Respond to Webhook
   - Respond with JSON message confirming draft creation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow automatically creates a complete, SEO-ready WordPress blog post using AI from idea to published draft. | Sticky note “📝 Blog Form” |
| Before running the workflow, generate a WordPress Application Password for an administrator user and store it in the WordPress credentials in n8n. | Sticky note “📝 Blog Form” + “3.Write → Package → Create Draft → Generate Image” |
| IMPORTANT: Name your application `n8n-auto-blog`, generate the password, store it in n8n credentials. | Sticky note “3.Write → Package → Create Draft → Generate Image” |
| Image upload + featured image assignment should set `featured_media` to the uploaded media ID; current “Assign Featured Image” body is empty and may not actually attach the image. | Derived from node configuration of “Assign Featured Image” |

If you want, I can propose a corrected configuration for **Assign Featured Image** (exact body parameters and which field to map from the media upload response) based on the actual response your WordPress returns during upload.