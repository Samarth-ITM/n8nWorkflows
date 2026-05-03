Sync Note.com articles to WordPress with OpenAI categorization and tags

https://n8nworkflows.xyz/workflows/sync-note-com-articles-to-wordpress-with-openai-categorization-and-tags-13005


# Sync Note.com articles to WordPress with OpenAI categorization and tags

## 1. Workflow Overview

**Title:** Sync Note.com articles to WordPress with OpenAI categorization and tags

**Purpose:**  
This workflow monitors a note.com RSS feed, fetches full article data via the note.com API, uses OpenAI to select a **WordPress category ID** and **tag IDs**, migrates images to the WordPress media library (featured + inline images), rewrites image URLs inside the article HTML, then publishes the post to WordPress and sets the featured image.

**Target use cases:**
- Automatically syndicating note.com articles to a self-hosted WordPress site
- Keeping WordPress as the canonical hosting for media assets
- Enforcing consistent taxonomy using AI-assisted categorization

### 1.1 Feed Monitoring & Note ID Extraction
Detects new RSS items hourly, extracts the note ID from the RSS item URL.

### 1.2 Article Retrieval from note.com API
Fetches full structured article data (title/body/images/eyecatch).

### 1.3 AI Taxonomy Selection (Category + Tags)
Sends title + first 2000 chars to OpenAI; parses JSON response containing `category_id` and `tag_ids`.

### 1.4 Featured Image Migration
Downloads the eyecatch image and uploads it to WordPress Media Library.

### 1.5 Inline Image Migration + URL Rewriting
Downloads each inline image, uploads to WordPress, builds a mapping old→new URLs, replaces occurrences in article HTML.

### 1.6 WordPress Publishing + Featured Media Assignment
Creates a WordPress post (published) with selected taxonomy, then sets the featured image on the created post.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Feed Monitoring & Note ID Extraction
**Overview:** Watches the note.com RSS feed every hour and derives a `note_id` from each RSS item’s link for API retrieval.  
**Nodes involved:**  
- RSS Feed Trigger - Note.com  
- Extract Note ID  

#### Node: RSS Feed Trigger - Note.com
- **Type / role:** `rssFeedReadTrigger` — polling trigger for RSS.
- **Key configuration:**
  - Feed URL: `={{ $parameter.noteRssFeedUrl }}`
  - Polling: every hour
- **Inputs / outputs:**
  - **Output →** Extract Note ID
- **Edge cases / failures:**
  - Invalid RSS URL, network/DNS failures, rate limits
  - RSS item without `link` field can break downstream parsing
- **Version notes:** Type version `1` (older trigger; behavior depends on n8n RSS trigger implementation but generally stable).

#### Node: Extract Note ID
- **Type / role:** `set` — creates `note_id` used by API call.
- **Key configuration:**
  - Adds field `note_id`:
    - `={{ $json.link.split('/').pop() }}`
- **Inputs / outputs:**
  - **Input:** RSS item JSON
  - **Output →** Get Article from Note.com API
- **Edge cases / failures:**
  - If `$json.link` is missing or not a string, expression fails
  - Trailing slash in link may yield empty `note_id` (e.g., `.../12345/`)
- **Version notes:** Type version `3.4` (Set node with “assignments” UI).

---

### Block 1.2 — Article Retrieval from note.com API
**Overview:** Fetches the full note.com article payload including HTML body and image metadata.  
**Nodes involved:**  
- Get Article from Note.com API  

#### Node: Get Article from Note.com API
- **Type / role:** `httpRequest` — retrieves note.com API JSON.
- **Key configuration:**
  - URL: `=https://note.com/api/v3/notes/{{ $json.note_id }}`
  - Method: default GET
- **Inputs / outputs:**
  - **Input:** `note_id` from Extract Note ID
  - **Output →** AI Categorization (OpenAI)
- **Edge cases / failures:**
  - 404 if note ID invalid or article removed
  - API structure changes (expects `data.name`, `data.body`, `data.eyecatch`, `data.pictures`)
  - Rate limiting / transient 5xx
- **Version notes:** Type version `4.3` (HTTP Request node; response handling differs slightly across versions).

---

### Block 1.3 — AI Taxonomy Selection (Category + Tags)
**Overview:** Uses OpenAI to decide the best WordPress category and tags, then parses the result into structured fields.  
**Nodes involved:**  
- AI Categorization (OpenAI)  
- Parse AI Response  

#### Node: AI Categorization (OpenAI)
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — LLM call for classification.
- **Key configuration:**
  - Model: `gpt-4o-mini`
  - System prompt: instructs to output **JSON only** with numeric `category_id` and array `tag_ids`.
  - User content sent:  
    `Title: {{ $json.data.name }} Body (first 2000 chars): {{ $json.data.body.substring(0, 2000) }}`
- **Inputs / outputs:**
  - **Input:** note.com API response
  - **Output →** Parse AI Response
- **Edge cases / failures:**
  - Non-JSON output despite instructions (markdown fenced blocks, extra text)
  - Missing `data.body` or `substring` called on non-string value
  - Model/auth/quota errors
- **Version notes:** Type version `2.1` (LangChain-based OpenAI node; output shape can differ by version/settings).

#### Node: Parse AI Response
- **Type / role:** `code` — extracts and parses JSON from the OpenAI response.
- **Key configuration (interpreted):**
  - Reads LLM output from: `let rawText = $json.output[0].content[0].text;`
  - Strips ```json fences
  - `JSON.parse(cleanJson)` and returns parsed object
  - On failure returns `{ error, raw, cleaned }`
- **Inputs / outputs:**
  - **Input:** OpenAI node output
  - **Output →** Download Featured Image
- **Edge cases / failures:**
  - If the OpenAI node output schema differs, `$json.output[0].content[0].text` may be undefined
  - If parsing fails, downstream nodes still run but will later break when category/tags are referenced
- **Version notes:** Type version `2` (Code node).

---

### Block 1.4 — Featured Image Migration
**Overview:** Downloads the note.com eyecatch image and uploads it to WordPress media; later used as `featured_media`.  
**Nodes involved:**  
- Download Featured Image  
- Upload Featured Image to WordPress  

#### Node: Download Featured Image
- **Type / role:** `httpRequest` — downloads eyecatch as binary.
- **Key configuration:**
  - URL: `={{ $('Get Article from Note.com API').item.json.data.eyecatch }}`
  - Response format: **file** (binary)
- **Inputs / outputs:**
  - **Input:** Parse AI Response (for sequencing), but it actually references data from “Get Article…”
  - **Output →** Upload Featured Image to WordPress
- **Edge cases / failures:**
  - `data.eyecatch` missing or null
  - Non-image response or blocked hotlinking
- **Version notes:** Type version `4.3`.

#### Node: Upload Featured Image to WordPress
- **Type / role:** `httpRequest` — uploads binary to WordPress media endpoint.
- **Key configuration:**
  - URL: `={{ $parameter.wordpressSiteUrl }}/wp-json/wp/v2/media`
  - Method: POST
  - Authentication: predefined credential type `wordpressApi` (Application Password / WP REST)
  - Sends binary (`contentType: binaryData`), field name `data`
  - Headers:
    - `Content-Disposition: attachment; filename="eyecatch.png"`
    - `Content-Type: image/png`
- **Inputs / outputs:**
  - **Input:** Download Featured Image (binary)
  - **Output →** Prepare Image Array
- **Edge cases / failures:**
  - Wrong credentials / insufficient permissions to upload media
  - Incorrect content-type (if eyecatch is JPG/WebP but forced to PNG)
  - WordPress limits: max upload size, blocked mime types
- **Version notes:** Type version `4.3`.

---

### Block 1.5 — Inline Image Migration + URL Rewriting
**Overview:** Migrates all images referenced by note.com (`data.pictures`), uploads them to WordPress, creates an old→new URL mapping, then rewrites the HTML body to use WordPress-hosted URLs.  
**Nodes involved:**  
- Prepare Image Array  
- Split Images  
- Download Article Images  
- Upload Article Images to WordPress  
- Create URL Mapping  
- Aggregate URL Mappings  
- Replace Image URLs in Content  

#### Node: Prepare Image Array
- **Type / role:** `set` — copies the article pictures array into a split-friendly field.
- **Key configuration:**
  - `temp_pictures = {{ $node["Get Article from Note.com API"].json.data.pictures }}`
- **Inputs / outputs:**
  - **Input:** Upload Featured Image to WordPress (sequencing)
  - **Output →** Split Images
- **Edge cases / failures:**
  - `data.pictures` missing/not an array → split may fail or produce no items
- **Version notes:** Type version `3.4`.

#### Node: Split Images
- **Type / role:** `splitOut` — emits one item per image.
- **Key configuration:**
  - Field to split: `temp_pictures`
- **Inputs / outputs:**
  - **Output →** Download Article Images
- **Edge cases / failures:**
  - Empty list: no items flow forward, which can break later aggregation/posting unless n8n continues with no items (depends on configuration)
- **Version notes:** Type version `1`.

#### Node: Download Article Images
- **Type / role:** `httpRequest` — downloads each inline image as binary.
- **Key configuration:**
  - URL: `={{ $json.url }}`
  - Response format: **file**
- **Inputs / outputs:**
  - **Input:** each split image item (expects `url`)
  - **Output →** Upload Article Images to WordPress
- **Edge cases / failures:**
  - Missing `$json.url`, 403, hotlink protection
- **Version notes:** Type version `4.3`.

#### Node: Upload Article Images to WordPress
- **Type / role:** `httpRequest` — uploads each inline image binary to WP media.
- **Key configuration:**
  - URL: `={{ $parameter.wordpressSiteUrl }}/wp-json/wp/v2/media`
  - Method: POST, auth `wordpressApi`
  - Binary field: `data`
  - Headers:
    - `Content-Disposition: attachment; filename="image_{{ $json.index }}.png"`
    - `Content-Type: image/png`
- **Inputs / outputs:**
  - **Output →** Create URL Mapping
- **Edge cases / failures:**
  - Same issues as featured upload (mime/type mismatch, size limits)
  - `index` may not exist in the split item; filename becomes `image_.png`
- **Version notes:** Type version `4.3`.

#### Node: Create URL Mapping
- **Type / role:** `set` — creates an object with old/new URL pair.
- **Key configuration:**
  - `old_url = {{ $node["Split Images"].json.url }}`
  - `new_url = {{ $json.source_url }}`
- **Inputs / outputs:**
  - **Input:** WordPress upload response (expects `source_url`)
  - **Output →** Aggregate URL Mappings
- **Edge cases / failures:**
  - If WordPress response lacks `source_url`, mapping breaks
  - If old URL isn’t exactly the one used in HTML, replacement won’t occur
- **Version notes:** Type version `3.4`.

#### Node: Aggregate URL Mappings
- **Type / role:** `aggregate` — collects all mapping items into one array.
- **Key configuration:**
  - Mode: aggregate all item data
  - Output field: `url_map` (array of mapping objects)
- **Inputs / outputs:**
  - **Output →** Replace Image URLs in Content
- **Edge cases / failures:**
  - With zero inline images, this node may receive no items (downstream post creation may not run unless you handle empty case)
- **Version notes:** Type version `1`.

#### Node: Replace Image URLs in Content
- **Type / role:** `code` — rewrites HTML body by replacing old URLs with new ones.
- **Key configuration (interpreted):**
  - Reads body HTML: `$node["Get Article from Note.com API"].json.data.body`
  - Iterates over `$json.url_map` and does `split(old).join(new)` for global replacement
  - Outputs `{ final_body: bodyHtml }`
- **Inputs / outputs:**
  - **Output →** Create WordPress Post
- **Edge cases / failures:**
  - If `url_map` missing/undefined, loop may throw
  - If note body contains URL variants (query strings, resized URLs), exact string replacement may miss some
- **Version notes:** Type version `2`.

---

### Block 1.6 — WordPress Publishing + Featured Media Assignment
**Overview:** Creates and publishes the post in WordPress with AI-selected taxonomy and rewritten content, then sets the featured image.  
**Nodes involved:**  
- Create WordPress Post  
- Set Featured Image on Post  

#### Node: Create WordPress Post
- **Type / role:** `wordpress` — creates a WordPress post (REST API wrapper).
- **Key configuration:**
  - Title: from note data: `Get Article... data.name`
  - Content: `={{ $json.final_body }}`
  - Status: `publish`
  - Categories: `={{ $('Parse AI Response').item.json.category_id }}`
  - Tags: `={{ $('Parse AI Response').item.json.tag_ids }}`
  - Ping status: `open`
- **Inputs / outputs:**
  - **Input:** Replace Image URLs in Content
  - **Output →** Set Featured Image on Post
- **Edge cases / failures:**
  - If Parse AI Response returned an error object, categories/tags expressions may be undefined
  - WordPress taxonomy IDs must exist; invalid IDs cause REST errors
  - WordPress auth/permissions issues
- **Version notes:** Type version `1` (older WP node; ensure your n8n supports it and credentials are compatible).

#### Node: Set Featured Image on Post
- **Type / role:** `httpRequest` — updates created post to set `featured_media`.
- **Key configuration:**
  - URL: `={{ $parameter.wordpressSiteUrl }}/wp-json/wp/v2/posts/{{ $json.id }}`
  - Method: POST
  - Query parameter: `featured_media={{ $('Upload Featured Image to WordPress').item.json.id }}`
  - Auth: `wordpressApi`
- **Inputs / outputs:**
  - **Input:** Create WordPress Post (expects `$json.id`)
- **Edge cases / failures:**
  - If featured image upload failed, `id` is missing
  - Some WP setups expect JSON body instead of query string (usually both work, but not always)
- **Version notes:** Type version `4.3`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| RSS Feed Trigger - Note.com | rssFeedReadTrigger | Poll RSS feed hourly | — | Extract Note ID | ## Note.com to WordPress Auto-Publisher … Prerequisites: OpenAI API credentials; WordPress API credentials (Application Password); Your note.com RSS feed URL |
| Extract Note ID | set | Extract note ID from RSS link | RSS Feed Trigger - Note.com | Get Article from Note.com API | ### Configuration Required … (Replace RSS URL; Update WordPress URL; Customize categories/tags; Set credentials) |
| Get Article from Note.com API | httpRequest | Fetch full note data from note.com API | Extract Note ID | AI Categorization (OpenAI) | ## Note.com to WordPress Auto-Publisher … |
| AI Categorization (OpenAI) | @n8n/n8n-nodes-langchain.openAi | Choose WP category/tag IDs via LLM | Get Article from Note.com API | Parse AI Response | ### AI Categorization … Customize the system prompt to match your WordPress taxonomy! |
| Parse AI Response | code | Parse JSON from LLM output | AI Categorization (OpenAI) | Download Featured Image | ### AI Categorization … Customize the system prompt to match your WordPress taxonomy! |
| Download Featured Image | httpRequest | Download eyecatch binary | Parse AI Response | Upload Featured Image to WordPress | ## Note.com to WordPress Auto-Publisher … |
| Upload Featured Image to WordPress | httpRequest | Upload featured image to WP media | Download Featured Image | Prepare Image Array | ## Note.com to WordPress Auto-Publisher … |
| Prepare Image Array | set | Prepare `pictures` array for splitting | Upload Featured Image to WordPress | Split Images | ### Image Processing … ensures images are hosted on YOUR server. |
| Split Images | splitOut | Emit one item per inline image | Prepare Image Array | Download Article Images | ### Image Processing … |
| Download Article Images | httpRequest | Download each inline image binary | Split Images | Upload Article Images to WordPress | ### Image Processing … |
| Upload Article Images to WordPress | httpRequest | Upload each inline image to WP media | Download Article Images | Create URL Mapping | ### Image Processing … |
| Create URL Mapping | set | Build old→new URL pair | Upload Article Images to WordPress | Aggregate URL Mappings | ### Image Processing … |
| Aggregate URL Mappings | aggregate | Collect all URL pairs into array | Create URL Mapping | Replace Image URLs in Content | ### Image Processing … |
| Replace Image URLs in Content | code | Rewrite HTML body with WP URLs | Aggregate URL Mappings | Create WordPress Post | ### Image Processing … |
| Create WordPress Post | wordpress | Publish WP post with taxonomy | Replace Image URLs in Content | Set Featured Image on Post | ## Note.com to WordPress Auto-Publisher … |
| Set Featured Image on Post | httpRequest | Update WP post featured_media | Create WordPress Post | — | ## Note.com to WordPress Auto-Publisher … |
| Sticky Note - Introduction | stickyNote | Documentation/comment | — | — | (self) |
| Sticky Note - AI | stickyNote | Documentation/comment | — | — | (self) |
| Sticky Note - Images | stickyNote | Documentation/comment | — | — | (self) |
| Sticky Note - Config | stickyNote | Documentation/comment | — | — | (self) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow parameters**
   1) Add a workflow parameter named `noteRssFeedUrl` (your note.com RSS URL).  
   2) Add a workflow parameter named `wordpressSiteUrl` (e.g., `https://example.com`).

2. **Add trigger: “RSS Feed Trigger”**
   - Node type: **RSS Feed Read Trigger**
   - Feed URL: `{{$parameter.noteRssFeedUrl}}`
   - Polling: **Every hour**

3. **Add “Extract Note ID” (Set node)**
   - Add field `note_id` (string)
   - Value: `{{$json.link.split('/').pop()}}`
   - Connect: RSS Trigger → Extract Note ID

4. **Add “Get Article from Note.com API” (HTTP Request)**
   - Method: GET
   - URL: `https://note.com/api/v3/notes/{{$json.note_id}}`
   - Connect: Extract Note ID → Get Article…

5. **Add “AI Categorization (OpenAI)”**
   - Node type: **OpenAI (LangChain)**
   - Credentials: configure **OpenAI API** in n8n
   - Model: `gpt-4o-mini`
   - Messages:
     - System: include the category/tag ID lists and require strict JSON output
     - User: `Title: {{$json.data.name}} Body (first 2000 chars): {{$json.data.body.substring(0, 2000)}}`
   - Connect: Get Article… → AI Categorization

6. **Add “Parse AI Response” (Code node)**
   - Paste JS that:
     - reads the LLM output text
     - removes ```json fences
     - parses JSON and returns `{category_id, tag_ids}`
   - Connect: AI Categorization → Parse AI Response

7. **Featured image download/upload**
   1) **Download Featured Image** (HTTP Request)
      - URL: `{{$('Get Article from Note.com API').item.json.data.eyecatch}}`
      - Response: **File**
      - Connect: Parse AI Response → Download Featured Image
   2) **Upload Featured Image to WordPress** (HTTP Request)
      - URL: `{{$parameter.wordpressSiteUrl}}/wp-json/wp/v2/media`
      - Method: POST
      - Authentication: **WordPress API** credential (Application Password / WP REST)
      - Send Body: true; Content Type: **binary**
      - Binary property: `data`
      - Headers:
        - `Content-Disposition: attachment; filename="eyecatch.png"`
        - `Content-Type: image/png`
      - Connect: Download Featured Image → Upload Featured Image

8. **Inline image migration**
   1) **Prepare Image Array** (Set)
      - Field `temp_pictures` (array):
        - `{{$node["Get Article from Note.com API"].json.data.pictures}}`
      - Connect: Upload Featured Image → Prepare Image Array
   2) **Split Images** (Split Out)
      - Field to split: `temp_pictures`
      - Connect: Prepare Image Array → Split Images
   3) **Download Article Images** (HTTP Request)
      - URL: `{{$json.url}}`
      - Response: **File**
      - Connect: Split Images → Download Article Images
   4) **Upload Article Images to WordPress** (HTTP Request)
      - URL: `{{$parameter.wordpressSiteUrl}}/wp-json/wp/v2/media`
      - Method: POST; Auth: WordPress API credential
      - Send binary `data`
      - Headers:
        - `Content-Disposition: attachment; filename="image_{{$json.index}}.png"`
        - `Content-Type: image/png`
      - Connect: Download Article Images → Upload Article Images
   5) **Create URL Mapping** (Set)
      - `old_url`: `{{$node["Split Images"].json.url}}`
      - `new_url`: `{{$json.source_url}}`
      - Connect: Upload Article Images → Create URL Mapping
   6) **Aggregate URL Mappings** (Aggregate)
      - Aggregate all items into field `url_map`
      - Connect: Create URL Mapping → Aggregate URL Mappings
   7) **Replace Image URLs in Content** (Code)
      - Load original HTML from `Get Article… data.body`
      - For each mapping in `url_map`, replace all occurrences
      - Output `{ final_body: ... }`
      - Connect: Aggregate URL Mappings → Replace Image URLs in Content

9. **Create and publish WordPress post**
   - Node type: **WordPress**
   - Operation: create post
   - Title: `{{$node["Get Article from Note.com API"].json.data.name}}`
   - Content: `{{$json.final_body}}`
   - Status: `publish`
   - Categories: `{{$('Parse AI Response').item.json.category_id}}`
   - Tags: `{{$('Parse AI Response').item.json.tag_ids}}`
   - Connect: Replace Image URLs in Content → Create WordPress Post
   - Credentials: configure WordPress credentials used by the WP node

10. **Set featured image on the created post**
   - Node type: **HTTP Request**
   - URL: `{{$parameter.wordpressSiteUrl}}/wp-json/wp/v2/posts/{{$json.id}}`
   - Method: POST
   - Auth: WordPress API credential
   - Query parameter:
     - `featured_media = {{$('Upload Featured Image to WordPress').item.json.id}}`
   - Connect: Create WordPress Post → Set Featured Image on Post

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Note.com to WordPress Auto-Publisher” prerequisites: OpenAI credentials, WordPress API credentials (Application Password), note.com RSS feed URL | Sticky Note - Introduction |
| “Customize the system prompt to match your WordPress taxonomy!” | Sticky Note - AI |
| Images are downloaded from note.com, uploaded to WordPress, and URLs replaced so media is hosted on your server | Sticky Note - Images |
| Configuration required: replace RSS URL, update WordPress URL in HTTP nodes, customize categories/tags in AI prompt, set OpenAI + WordPress credentials | Sticky Note - Config |