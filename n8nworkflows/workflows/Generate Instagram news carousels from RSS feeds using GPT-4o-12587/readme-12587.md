Generate Instagram news carousels from RSS feeds using GPT-4o

https://n8nworkflows.xyz/workflows/generate-instagram-news-carousels-from-rss-feeds-using-gpt-4o-12587


# Generate Instagram news carousels from RSS feeds using GPT-4o

## 1. Workflow Overview

**Purpose:**  
This workflow generates a daily Instagram carousel (up to 10 slides) from an RSS feed. For each RSS item, it fetches the article HTML to extract an image, uses **GPT‑4o** to rewrite the content into a “viral” category/headline/caption, renders each slide image via either **Gotenberg (free)** or **APITemplate (paid)**, uploads the slide images to **Google Drive** to obtain a public URL, then creates and publishes an **Instagram carousel** via the **Facebook Graph API**.

**Target use cases:**
- Automated “daily news brief” carousels (tech, sports, finance, etc.)
- Social content pipelines where AI writes hooks and captions, and images are programmatically rendered and posted

### 1.1 Input Reception & Configuration
User submits a form with RSS URL, brand name, primary color, and image engine choice.

### 1.2 RSS Ingestion & Article Scraping
Read RSS, limit to 10 items, fetch each article HTML, extract `og:image`, and keep only token-efficient text fields.

### 1.3 AI Content Generation (GPT‑4o)
For each cleaned article item, GPT‑4o returns JSON: `category`, `headline`, `caption`.

### 1.4 Slide Composition (HTML or APITemplate)
Transforms AI output into either:
- **HTML (with brand color)** → screenshot-rendered by Gotenberg, or
- **APITemplate** image generation API → returns downloadable image URL

### 1.5 File Hosting (Google Drive) → Instagram Carousel Publish
Uploads each generated slide image to Drive, shares publicly, creates IG carousel item containers, bundles them, and publishes.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Workflow Setup
**Overview:** Collects configuration values (RSS feed, branding, engine selection) via an n8n Form Trigger and starts the pipeline.

**Nodes involved:**
- `SETUP FORM`

#### Node: SETUP FORM
- **Type / role:** `Form Trigger` — workflow entry point; collects user inputs.
- **Key configuration:**
  - Form title: “AI News Carousel Setup”
  - Fields (required):
    - `RSS_FEED_URL` (text)
    - `BRAND_NAME` (text)
    - `PRIMARY_COLOR` (text; expected hex like `#FF5733`)
    - `IMAGE_ENGINE` (dropdown): `FREE_GOTENBERG` or `PAID_APITEMPLATE`
  - Custom CSS theme applied (cyberpunk dark UI).
- **Outputs:** Triggers workflow with submitted values in JSON.
- **Connections:** `SETUP FORM` → `News Source`
- **Edge cases / failures:**
  - Invalid RSS URL (downstream RSS read will fail).
  - Invalid hex color (still inserted into CSS; may render incorrectly).
- **Version notes:** Form Trigger v2.3 features such as custom CSS and field definitions.

---

### Block 2 — RSS Ingestion & Article Scraping
**Overview:** Reads the RSS feed, limits to 10 items, downloads each article HTML, and extracts a representative image (Open Graph) while keeping payload small for the AI call.

**Nodes involved:**
- `News Source`
- `Limit to Top 10`
- `Get Article HTML`
- `Cleaner`

#### Node: News Source
- **Type / role:** `RSS Feed Read` — fetches items from a feed URL.
- **Key configuration:**
  - URL: `{{ $json.RSS_FEED_URL }}` (from form submission)
- **Connections:** `SETUP FORM` → `News Source` → `Limit to Top 10`
- **Edge cases / failures:**
  - Feed unreachable / invalid XML → node error.
  - Some feeds return fewer than 10 items (fine; downstream loops over available items).

#### Node: Limit to Top 10
- **Type / role:** `Limit` — restricts processing to first 10 RSS entries.
- **Key configuration:** `maxItems: 10`
- **Connections:** `News Source` → `Limit to Top 10` → `Get Article HTML`
- **Edge cases:** If RSS returns 0 items, downstream nodes receive no items.

#### Node: Get Article HTML
- **Type / role:** `HTTP Request` — downloads each article page HTML.
- **Key configuration:**
  - URL: `{{ $json.link }}`
  - Response format: **text**
  - Redirects: enabled (default redirect handling is configured)
- **Connections:** `Limit to Top 10` → `Get Article HTML` → `Cleaner`
- **Edge cases / failures:**
  - Paywalls / bot protections (403/503), Cloudflare challenges.
  - Large HTML payloads (performance impact), timeouts.
  - Some RSS items may have missing `link`.

#### Node: Cleaner
- **Type / role:** `Code` — merges RSS text with scraped image, strips HTML from the data passed forward.
- **Key logic/config choices:**
  - Reads RSS items via: `$('News Source').all()`
  - Reads current input (HTML responses) via: `$input.all()`
  - Extracts `og:image` using regex:
    - `<meta property="og:image" content="...">`
  - Default image fallback:
    - `https://images.unsplash.com/photo-1620712943543-bcc4688e7485?q=80&w=1080&auto=format&fit=crop`
  - Output JSON per item:
    - `title`
    - `content` (prefers `contentSnippet` else `content`)
    - `link`
    - `scraped_image`
  - Intentionally does **not** pass full HTML forward to reduce tokens/cost.
- **Connections:** `Get Article HTML` → `Cleaner` → `AI Analyst`
- **Edge cases / failures:**
  - Regex may miss images if site uses different tags (`og:image:secure_url`, lazy-loaded metadata, etc.).
  - Index alignment assumption: it pairs RSS item `i` with scraped HTML item `i`. If any HTTP request fails mid-list, alignment can break and merge wrong title/image.
  - Missing `contentSnippet` and `content` may yield empty summaries.

---

### Block 3 — AI Content Generation (GPT‑4o)
**Overview:** For each cleaned RSS item, asks GPT‑4o to produce a viral category, short headline, and “hot take” caption in JSON-only format.

**Nodes involved:**
- `AI Analyst`

#### Node: AI Analyst
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat model call.
- **Key configuration:**
  - Model: `chatgpt-4o-latest`
  - Prompt instructs:
    - Inputs: `{{ $json.title }}`, `{{ $json.content }}`
    - Outputs JSON only with keys: `category`, `headline`, `caption`
    - Constraints: category max 2 words; headline max 7 words; caption max 20 words
- **Connections:** `Cleaner` → `AI Analyst` → `The Setup`
- **Credentials:** `OpenAi account` (OpenAI API credential)
- **Edge cases / failures:**
  - Model may wrap JSON in code fences despite instruction (downstream code handles this).
  - Rate limits / quota exhaustion.
  - Empty/malformed `content` reduces output quality.

---

### Block 4 — Image Engine Selection & Slide Rendering
**Overview:** Converts AI output into slide render inputs, then routes items to either Gotenberg (HTML screenshot) or APITemplate (image API). Finally downloads the resulting image file for upload.

**Nodes involved:**
- `The Setup`
- `Loop Over Items`
- `Route by Engine`
- `Designer` (Gotenberg path)
- `Generate Image` (APITemplate path)
- `Download Image`

#### Node: The Setup
- **Type / role:** `Code` — parses AI JSON, applies branding (color/date/footer), and generates HTML as binary for Gotenberg.
- **Key configuration / variables:**
  - Reads form data: `$('SETUP FORM').first().json`
  - Uses:
    - `PRIMARY_COLOR` default `#3B82F6`
    - `BRAND_NAME` in footer
  - Attempts to read Cleaner items: `$('Cleaner').all()` for fallback title/image
  - Parses AI response:
    - Supports `item.json.output[0].content[0].text`, or `item.json.output` string, or `item.json.text`
    - Strips ```json fences and extracts first `{...}` for parsing
    - Fallback if parsing fails:
      - `{ headline: fallbackTitle, category: "NEWS UPDATE", caption: "Link in bio." }`
  - Creates 1080×1350 HTML layout with:
    - Background image from `scraped_image` (or default)
    - Category text colored with `PRIMARY_COLOR`
    - Date badge (formatted like `JAN 19 • 2026` in uppercase style)
  - Outputs:
    - `json`: merged data with `headline`, `category`, `caption`, `image_url`
    - `binary.index_html`: HTML file for screenshot
- **Connections:** `AI Analyst` → `The Setup` → `Loop Over Items`
- **Edge cases / failures:**
  - If `Cleaner` items count doesn’t match AI items, fallback image/title may mismatch.
  - Invalid image URL can break background rendering (Gotenberg still may render with blank background).
  - Any parsing errors fall back to generic text.

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — iterates items in batches, enabling sequential slide creation/publishing.
- **Key configuration:** batch size `10`
- **Connections:**
  - Input: `The Setup`
  - Outputs:
    - Output 1 → `Carousel Prep` (runs after carousel items created; see publishing loop behavior below)
    - Output 2 → `Route by Engine` (main per-item processing path)
- **Behavior note:** This node is also part of a loop because `Create Container` connects back into `Loop Over Items`.
- **Edge cases / failures:**
  - Loop wiring means execution order matters; misconfiguration can cause incomplete batches or premature caption bundling.

#### Node: Route by Engine
- **Type / role:** `Switch` — routes each item to selected image engine.
- **Key configuration:**
  - If `{{ $("SETUP FORM").first().json.IMAGE_ENGINE }}` equals:
    - `FREE_GOTENBERG` → output “Gotenberg” → `Designer`
    - `PAID_APITEMPLATE` → output “APITemplate” → `Generate Image`
- **Connections:** `Loop Over Items` → `Route by Engine` → (`Designer` or `Generate Image`)
- **Edge cases / failures:**
  - If `IMAGE_ENGINE` is missing/unexpected, no route matches → item is dropped (no slide produced).

#### Node: Designer (Gotenberg path)
- **Type / role:** `HTTP Request` — sends HTML binary to Gotenberg Chromium screenshot endpoint.
- **Key configuration:**
  - URL: `http://gotenberg:3000/forms/chromium/screenshot/html`
  - Method: `POST`
  - Body: `multipart/form-data` with:
    - `index.html` from binary field `index_html`
    - width `1080`, height `1350`
    - format `jpeg`
    - waitDelay `1s`
- **Connections:** `Route by Engine` → `Designer` → `Wait for Processing`
- **Requirements:**
  - Gotenberg reachable at hostname `gotenberg` (commonly via Docker network).
- **Edge cases / failures:**
  - Gotenberg not running / DNS not resolvable → connection error.
  - Rendering timeouts for slow-loading backgrounds.
  - If HTML uses remote images that block/hang, screenshot may be blank or delayed.

#### Node: Generate Image (APITemplate path)
- **Type / role:** `HTTP Request` — calls APITemplate to generate a rendered slide image.
- **Key configuration:**
  - Endpoint: `https://rest.apitemplate.io/v2/create-image`
  - Method: `POST`, JSON body with overrides:
    - `headline`, `category`, `caption`
    - `background_image.src`: `{{ $json.image_url.split(',')[0] }}`
    - `footer`: brand name from form
    - `date`: `MMM DD` uppercase (runtime)
  - Query parameter: `template_id=PASTE_YOUR_TEMPLATE_ID_HERE`
  - Header: `X-API-KEY=PASTE_YOUR_KEY_HERE`
- **Connections:** `Route by Engine` → `Generate Image` → `Download Image`
- **Edge cases / failures:**
  - Missing/invalid API key or template ID → 401/404.
  - Template layer names must match (`headline`, `category`, etc.) or overrides won’t apply.
  - Background image URL invalid/unreachable → render failures.

#### Node: Download Image
- **Type / role:** `HTTP Request` — downloads the produced image file into binary.
- **Key configuration:**
  - URL: `{{ $json.download_url || $json.url }}`
  - Response format: **file**
- **Connections:**  
  - From APITemplate: `Generate Image` → `Download Image` → `Wait for Processing`  
  - (Gotenberg bypasses this and goes directly `Designer` → `Wait for Processing`)
- **Edge cases / failures:**
  - Some APITemplate responses might differ (neither `download_url` nor `url`) → expression resolves to empty.
  - Large images / slow host → timeout.

---

### Block 5 — Google Drive Hosting & Instagram Publishing
**Overview:** Ensures each slide has a publicly accessible URL (via Drive sharing), creates Instagram carousel item containers, then bundles and publishes the carousel post.

**Nodes involved:**
- `Wait for Processing`
- `Upload file`
- `Share file`
- `Create Container`
- `Carousel Prep`
- `Carousel Bundle`
- `Publish Carousel`

#### Node: Wait for Processing
- **Type / role:** `Wait` — introduces a delay / synchronization point before uploading.
- **Key configuration:** `amount: 1` (unit depends on node defaults; typically seconds in n8n Wait)
- **Connections:** `Designer` → `Wait for Processing` → `Upload file` and `Download Image` → `Wait for Processing` → `Upload file`
- **Edge cases / failures:**
  - If set too low, Drive upload may happen before binary is ready (rare, but possible depending on upstream response timing).
  - If Wait is configured in “Resume via webhook” mode, it would require external resume; here it appears to be simple delay (but confirm in UI).

#### Node: Upload file
- **Type / role:** `Google Drive` — uploads the binary image file.
- **Key configuration:**
  - Drive: “My Drive”
  - Folder: `Viral News` (ID `172cYWt7ryoD6QqNpnC_e8HwpRQrGUiah`)
  - Operation implied: upload (default for this node when binary present)
- **Connections:** `Wait for Processing` → `Upload file` → `Share file`
- **Credentials:** Google Drive OAuth2
- **Edge cases / failures:**
  - Missing binary input (if upstream didn’t produce file) → upload fails.
  - Folder permission issues.
  - Large upload rate limits.

#### Node: Share file
- **Type / role:** `Google Drive` — makes the uploaded file publicly readable.
- **Key configuration:**
  - Operation: `share`
  - File ID: `{{ $json.id }}`
  - Permissions: `reader`, `anyone`, discovery allowed
- **Connections:** `Upload file` → `Share file` → `Create Container`
- **Edge cases / failures:**
  - Org policies may block public sharing.
  - If public sharing disabled, Instagram won’t be able to fetch the image URL.

#### Node: Create Container
- **Type / role:** `Facebook Graph API` — creates an Instagram carousel *item* container (one per slide).
- **Key configuration:**
  - Edge: `media`
  - Node (IG Business Account ID): `REPLACE_WITH_YOUR_ID`
  - Query:
    - `image_url = https://lh3.googleusercontent.com/d/{{ $('Upload file').item.json.id }}`
    - `is_carousel_item = true`
  - API version: `v23.0`
  - Method: `POST`
- **Connections:** `Share file` → `Create Container` → `Loop Over Items` (loop back)
- **Critical dependency:**
  - Uses Google Drive “lh3.googleusercontent.com/d/<fileId>” pattern. This only works if the file is publicly accessible.
- **Edge cases / failures:**
  - Wrong IG Business Account ID → Graph API error.
  - Image URL not publicly reachable → container creation fails.
  - Instagram has aspect ratio/size constraints; invalid images can be rejected.
  - Loop-back wiring means container creation happens per item; ensure no infinite loop (Split in Batches controls it).

#### Node: Carousel Prep
- **Type / role:** `Code` — prepares the final carousel “children” list and the final caption (including headline bullets).
- **Key logic/config choices:**
  - Gets container IDs: `$('Create Container').all()` → takes up to 10 IDs
  - Builds `children_string` as comma-separated IDs for the carousel bundle
  - Builds a “DAILY TECH BRIEF” caption:
    - Extracts headline from AI output JSON (parses the same fenced JSON patterns)
    - Falls back to original `item.json.title` if parsing fails
    - Adds date (MMM DD), bullet list, CTA “Link in Bio…”, hashtags
- **Connections:** `Loop Over Items` (output 1) → `Carousel Prep` → `Carousel Bundle`
- **Edge cases / failures:**
  - If fewer than expected containers exist (some slides failed), caption list and children list may not match intended 10.
  - Assumes `AI Analyst` items correspond to slide order.

#### Node: Carousel Bundle
- **Type / role:** `Facebook Graph API` — creates the carousel container referencing all child media containers.
- **Key configuration:**
  - Edge: `media`
  - Node (IG Business Account ID): `REPLACE_WITH_YOUR_ID`
  - Query:
    - `media_type = CAROUSEL`
    - `children = {{ $json.children_string }}`
    - `caption = {{ $json.caption }}`
  - API version: `v23.0`, method `POST`
- **Connections:** `Carousel Prep` → `Carousel Bundle` → `Publish Carousel`
- **Edge cases / failures:**
  - Any invalid child container ID breaks bundling.
  - Caption length/hashtag constraints (Graph API may reject overly long captions).

#### Node: Publish Carousel
- **Type / role:** `Facebook Graph API` — publishes the created carousel container to Instagram.
- **Key configuration:**
  - Edge: `media_publish`
  - Node (IG Business Account ID): `REPLACE_WITH_YOUR_ID`
  - Query: `creation_id = {{ $json.id }}`
  - API version: `v23.0`, method `POST`
- **Connections:** `Carousel Bundle` → `Publish Carousel`
- **Edge cases / failures:**
  - Publishing too soon after creating containers can sometimes fail if media isn’t finished processing on IG side (a longer wait or retry may be needed).
  - Permissions/scopes missing from Graph API credential.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| SETUP FORM | Form Trigger | Collect runtime settings (RSS, brand, color, engine) | — | News Source | ## 1. Input & Scraping\nFetches RSS feed, limits to top 10, and extracts clean data. |
| News Source | RSS Feed Read | Read RSS items from provided URL | SETUP FORM | Limit to Top 10 | ## 1. Input & Scraping\nFetches RSS feed, limits to top 10, and extracts clean data. |
| Limit to Top 10 | Limit | Restrict processing to 10 items | News Source | Get Article HTML | ## 1. Input & Scraping\nFetches RSS feed, limits to top 10, and extracts clean data. |
| Get Article HTML | HTTP Request | Download article HTML for image extraction | Limit to Top 10 | Cleaner | ## 1. Input & Scraping\nFetches RSS feed, limits to top 10, and extracts clean data. |
| Cleaner | Code | Extract og:image and merge with RSS text (token efficient) | Get Article HTML | AI Analyst | ## 1. Input & Scraping\nFetches RSS feed, limits to top 10, and extracts clean data. |
| AI Analyst | OpenAI (LangChain) | Generate viral category/headline/caption in JSON | Cleaner | The Setup | ## 2. AI Content Generation\nUses GPT-4o to analyze the news content and generate viral hooks, headlines, and captions. |
| The Setup | Code | Parse AI JSON, apply branding, generate HTML binary slide | AI Analyst | Loop Over Items | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Loop Over Items | Split In Batches | Batch/loop control for up to 10 slides | The Setup; Create Container | Carousel Prep; Route by Engine | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Route by Engine | Switch | Choose Gotenberg vs APITemplate based on form input | Loop Over Items | Designer; Generate Image | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Designer | HTTP Request | Render HTML to JPEG via Gotenberg screenshot API | Route by Engine | Wait for Processing | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Generate Image | HTTP Request | Generate image via APITemplate using overrides | Route by Engine | Download Image | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Download Image | HTTP Request | Download produced image as binary file | Generate Image | Wait for Processing | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Wait for Processing | Wait | Small delay/sync before upload | Designer; Download Image | Upload file | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Upload file | Google Drive | Upload slide image file to Drive folder | Wait for Processing | Share file | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Share file | Google Drive | Make uploaded file public (anyone reader) | Upload file | Create Container | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Create Container | Facebook Graph API | Create IG carousel item container from public image URL | Share file | Loop Over Items | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Carousel Prep | Code | Collect container IDs and build final caption | Loop Over Items | Carousel Bundle | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Carousel Bundle | Facebook Graph API | Create CAROUSEL media container with children + caption | Carousel Prep | Publish Carousel | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Publish Carousel | Facebook Graph API | Publish the carousel container to Instagram | Carousel Bundle | — | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |
| Sticky Note | Sticky Note | Documentation | — | — | ## Viral News Agent (Instagram)\nThis workflow acts as a fully autonomous \"News Agency.\" It scrapes RSS feeds, uses AI to write viral scripts, designs carousel slides, and auto-posts to Instagram.\n\n## How it works\n1. **Scrape:** Monitors RSS feeds (Tech, Sports, etc.) for breaking news.\n2. **Analyze:** GPT-4o extracts viral hooks and writes a 10-slide script.\n3. **Design:** Generates images using either **Gotenberg (Free)** or **APITemplate (Paid)**.\n4. **Publish:** Uploads the carousel to Instagram Business.\n\n## Setup Steps\n1. **Credentials:** Connect OpenAI, Google Drive, and Facebook Graph API.\n2. **Instagram ID:** Open the 3 Facebook nodes (\"Create Container\", \"Bundle\", \"Publish\") and replace the default ID with your **Instagram Business Account ID**.\n3. **Image Engine:** In the \"Generate Image\" node, add your API Key if using the Paid mode. If using Free, ensure Docker is running Gotenberg. |
| Sticky Note1 | Sticky Note | Section header | — | — | ## 1. Input & Scraping\nFetches RSS feed, limits to top 10, and extracts clean data. |
| Sticky Note2 | Sticky Note | Section header | — | — | ## 2. AI Content Generation\nUses GPT-4o to analyze the news content and generate viral hooks, headlines, and captions. |
| Sticky Note3 | Sticky Note | Section header | — | — | ## 3. Image Engine\nIterates through each news item. Based on user selection, it routes data to either Gotenberg (Free) or APITemplate (Paid) to generate the slide. |
| Sticky Note4 | Sticky Note | Section header | — | — | ## 4. Instagram Publishing\nUploads images, bundles the carousel, and publishes to Instagram. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named: “Viral news AI agent for Instagram”.

2. **Add Form Trigger node** named `SETUP FORM`
   - Title: “AI News Carousel Setup”
   - Description: “Enter your details to generate today's viral carousels.”
   - Fields (all required):
     - `RSS_FEED_URL` (text)
     - `BRAND_NAME` (text)
     - `PRIMARY_COLOR` (text)
     - `IMAGE_ENGINE` (dropdown): `FREE_GOTENBERG`, `PAID_APITEMPLATE`
   - (Optional) Paste the provided custom CSS for theming.

3. **Add RSS Feed Read** named `News Source`
   - URL: `={{ $json.RSS_FEED_URL }}`
   - Connect: `SETUP FORM` → `News Source`

4. **Add Limit** named `Limit to Top 10`
   - Max items: `10`
   - Connect: `News Source` → `Limit to Top 10`

5. **Add HTTP Request** named `Get Article HTML`
   - URL: `={{ $json.link }}`
   - Response format: **Text**
   - Allow redirects (default is fine)
   - Connect: `Limit to Top 10` → `Get Article HTML`

6. **Add Code node** named `Cleaner`
   - Paste logic to:
     - read `$('News Source').all()`
     - read `$input.all()` (HTML)
     - extract `og:image` via regex
     - output `{ title, content, link, scraped_image }` without HTML
   - Connect: `Get Article HTML` → `Cleaner`

7. **Add OpenAI node (LangChain)** named `AI Analyst`
   - Credentials: configure **OpenAI API** credential in n8n
   - Model: `chatgpt-4o-latest`
   - Prompt: include the RSS title/content and request JSON-only output with `category`, `headline`, `caption`
   - Connect: `Cleaner` → `AI Analyst`

8. **Add Code node** named `The Setup`
   - Implement:
     - read form values from `$('SETUP FORM').first().json`
     - parse AI JSON (strip code fences)
     - fallback values if parsing fails
     - generate HTML (1080×1350) using `PRIMARY_COLOR`, `BRAND_NAME`, background image
     - output binary file `index_html` (`index.html`)
   - Connect: `AI Analyst` → `The Setup`

9. **Add Split In Batches** named `Loop Over Items`
   - Batch size: `10`
   - Connect: `The Setup` → `Loop Over Items`

10. **Add Switch** named `Route by Engine`
   - Rule 1: if `={{ $("SETUP FORM").first().json.IMAGE_ENGINE }}` equals `FREE_GOTENBERG` → output “Gotenberg”
   - Rule 2: equals `PAID_APITEMPLATE` → output “APITemplate”
   - Connect: `Loop Over Items` → `Route by Engine`

11. **Gotenberg branch (free)**
   - Add HTTP Request named `Designer`
     - Method: POST
     - URL: `http://gotenberg:3000/forms/chromium/screenshot/html`
     - Content type: `multipart/form-data`
     - Form fields:
       - `index.html` from binary property `index_html`
       - width `1080`, height `1350`, format `jpeg`, waitDelay `1s`
   - Connect: `Route by Engine (Gotenberg)` → `Designer`

12. **APITemplate branch (paid)**
   - Add HTTP Request named `Generate Image`
     - Method: POST
     - URL: `https://rest.apitemplate.io/v2/create-image`
     - Query parameter: `template_id` = your template id
     - Header: `X-API-KEY` = your API key
     - Body: JSON overrides for `headline`, `category`, `caption`, `background_image`, `footer`, `date`
   - Add HTTP Request named `Download Image`
     - URL: `={{ $json.download_url || $json.url }}`
     - Response format: **File**
   - Connect: `Route by Engine (APITemplate)` → `Generate Image` → `Download Image`

13. **Add Wait** named `Wait for Processing`
   - Amount: `1` (keep minimal delay; increase if IG/Drive needs time)
   - Connect:
     - `Designer` → `Wait for Processing`
     - `Download Image` → `Wait for Processing`

14. **Add Google Drive node** named `Upload file`
   - Credentials: configure **Google Drive OAuth2**
   - Upload to a chosen folder (e.g., “Viral News”)
   - Ensure it uploads the incoming binary image (from Gotenberg/APITemplate)
   - Connect: `Wait for Processing` → `Upload file`

15. **Add Google Drive node** named `Share file`
   - Operation: Share
   - File ID: `={{ $json.id }}`
   - Permissions: anyone + reader
   - Connect: `Upload file` → `Share file`

16. **Add Facebook Graph API node** named `Create Container`
   - Credentials: configure **Facebook Graph API** with permissions for Instagram publishing
   - Edge: `media`
   - Node: set to your **Instagram Business Account ID**
   - Method: POST
   - Query parameters:
     - `image_url = https://lh3.googleusercontent.com/d/{{ $('Upload file').item.json.id }}`
     - `is_carousel_item = true`
   - Connect: `Share file` → `Create Container`

17. **Close the loop for per-slide creation**
   - Connect: `Create Container` → `Loop Over Items`
   - This enables processing the next item in the batch after each container is created.

18. **Add Code node** named `Carousel Prep`
   - Build:
     - `children_string` from `$('Create Container').all().map(i=>i.json.id).slice(0,10).join(',')`
     - `caption` from parsed AI headlines + date + hashtags
   - Connect: `Loop Over Items` (the other output) → `Carousel Prep`

19. **Add Facebook Graph API node** named `Carousel Bundle`
   - Edge: `media`
   - Node: your **Instagram Business Account ID**
   - Method: POST
   - Query parameters:
     - `media_type = CAROUSEL`
     - `children = {{ $json.children_string }}`
     - `caption = {{ $json.caption }}`
   - Connect: `Carousel Prep` → `Carousel Bundle`

20. **Add Facebook Graph API node** named `Publish Carousel`
   - Edge: `media_publish`
   - Node: your **Instagram Business Account ID**
   - Method: POST
   - Query parameter:
     - `creation_id = {{ $json.id }}`
   - Connect: `Carousel Bundle` → `Publish Carousel`

21. **Infrastructure/Creds checklist**
   - **OpenAI credential** added and selected in `AI Analyst`
   - **Google Drive OAuth2** credential added and selected in `Upload file` + `Share file`
   - **Facebook Graph API** credential added and selected in all 3 IG nodes
   - If using **FREE_GOTENBERG**: ensure Gotenberg is running and resolvable as `gotenberg:3000` from n8n (commonly via Docker Compose network).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow acts as a fully autonomous “News Agency.” It scrapes RSS feeds, uses AI to write viral scripts, designs carousel slides, and auto-posts to Instagram. | Sticky note (overview) |
| Replace `REPLACE_WITH_YOUR_ID` in: “Create Container”, “Carousel Bundle”, “Publish Carousel” with your Instagram Business Account ID. | Facebook Graph API configuration |
| Paid engine setup requires APITemplate `X-API-KEY` and `template_id`. Endpoint: `https://rest.apitemplate.io/v2/create-image` | APITemplate branch |
| Free engine requires Gotenberg endpoint: `http://gotenberg:3000/forms/chromium/screenshot/html` (commonly Docker-based). | Gotenberg branch |
| Google Drive public URL pattern used: `https://lh3.googleusercontent.com/d/<FILE_ID>`; requires “anyone can read” sharing. | Instagram container image_url source |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.