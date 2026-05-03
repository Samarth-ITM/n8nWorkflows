Create SEO blog drafts in WordPress from trending topics with Perplexity and HuggingFace

https://n8nworkflows.xyz/workflows/create-seo-blog-drafts-in-wordpress-from-trending-topics-with-perplexity-and-huggingface-12386


# Create SEO blog drafts in WordPress from trending topics with Perplexity and HuggingFace

## 1. Workflow Overview

**Workflow title:** Create SEO blog drafts in WordPress from trending topics with Perplexity and HuggingFace

**Purpose:**  
This workflow discovers a currently trending topic (via Perplexity), queues it in Google Sheets, generates an SEO-oriented blog draft and an image prompt (via Perplexity), stores the generated content back into Sheets, creates a **WordPress draft post**, generates a **featured image** using **HuggingFace FLUX**, uploads it to the WordPress Media Library, and assigns it as the post’s featured image.

**Target use cases:**
- Automated content pipeline for editorial teams (draft-first, human review)
- Rapid trend-response publishing for dev/security/news niches
- Structured content generation with traceability via Google Sheets

### 1.1 Manual Start (Entry Point)
- Workflow is triggered manually from the n8n UI.

### 1.2 Trend Discovery (Perplexity)
- Perplexity retrieves trending topics; workflow selects one item from results.

### 1.3 Topic Queue & Retrieval (Google Sheets)
- Selected topic is appended to a sheet with `is_generated=0`.
- Workflow then pulls topics where `is_generated=0` for generation.

### 1.4 Content + Image Prompt Generation (Perplexity) and Parsing
- Perplexity generates a JSON payload containing blog title, sections, keywords, meta description, and an image prompt.
- A Code node sanitizes and parses the model output into valid JSON.

### 1.5 Persist Generated Content (Google Sheets) + WordPress Draft
- Update the matching row in Sheets with generated fields.
- Create a WordPress post as **draft** with generated title and HTML content.

### 1.6 Featured Image Generation + Upload + Assignment
- Call HuggingFace FLUX inference endpoint to generate an image file.
- Fix the binary metadata (filename/extension) for WordPress compatibility.
- Upload image to WordPress media via REST API.
- Set that image as `featured_media` on the created post.

---

## 2. Block-by-Block Analysis

### Block 1 — Manual Start
**Overview:** Starts the workflow on demand from the n8n editor.  
**Nodes involved:**  
- When clicking 'Execute workflow'

#### Node: When clicking 'Execute workflow'
- **Type / role:** Manual Trigger (`n8n-nodes-base.manualTrigger`) — entry point.
- **Configuration choices:** No parameters; runs only when user executes manually.
- **Connections:**
  - **Out:** → `Find top trending topics`
- **Edge cases / failures:** None (except workflow not executed).

---

### Block 2 — Trend Discovery & Selection
**Overview:** Uses Perplexity to fetch trend results, then randomly selects one topic from returned `search_results`.  
**Nodes involved:**  
- Find top trending topics  
- Select a clean trending topic

#### Node: Find top trending topics
- **Type / role:** Perplexity node (`n8n-nodes-base.perplexity`) — LLM/search-based trend retrieval.
- **Configuration choices (interpreted):**
  - Sends a single message prompt instructing Perplexity to find the most important topic for a given sector in the last 24–48h.
  - The prompt **demands JSON-only output** and **no citations**.
  - Note: The prompt includes `[Sector of choice]` placeholder; no workflow variable injects an actual sector.
- **Key variables/expressions:** None injected; prompt is static text.
- **Connections:**
  - **In:** Manual Trigger
  - **Out:** → `Select a clean trending topic`
- **Edge cases / failures:**
  - Perplexity may return non-JSON or include citations despite instructions.
  - Output structure may not contain `search_results` (the next node expects it).
  - Sector ambiguity: since `[Sector of choice]` isn’t replaced, results can be inconsistent.

#### Node: Select a clean trending topic
- **Type / role:** Code node (`n8n-nodes-base.code`) — selects a random item from results.
- **Configuration choices (interpreted):**
  - Reads `inputData.search_results`.
  - Picks a random index and returns that object as the single output item.
- **Key expressions/variables:**
  - Uses `$input.first().json` to read Perplexity output.
- **Connections:**
  - **In:** `Find top trending topics`
  - **Out:** → `Add topic to Google Sheets`
- **Edge cases / failures:**
  - Throws error if `search_results` missing/empty.
  - Random selection reduces determinism; repeated runs can queue varied topics.

---

### Block 3 — Topic Queue in Google Sheets + Fetch Ungenerated
**Overview:** Appends the chosen topic to Google Sheets, then fetches rows not yet generated (`is_generated = 0`).  
**Nodes involved:**  
- Add topic to Google Sheets  
- Get most recent topics from Google Sheet

#### Node: Add topic to Google Sheets
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — append row.
- **Configuration choices (interpreted):**
  - **Operation:** Append to `Sheet1` in the spreadsheet `YOUR_GOOGLE_SHEET_ID`.
  - Writes:
    - `Topic` = selected topic’s `snippet`
    - `is_generated` = `"0"`
- **Key expressions/variables:**
  - `Topic: ={{ $json.snippet }}`
  - `is_generated: 0`
- **Connections:**
  - **In:** `Select a clean trending topic`
  - **Out:** → `Get most recent topics from Google Sheet`
- **Version-specific notes:** Uses Google Sheets node v4.7 style “columns mapping”.
- **Edge cases / failures:**
  - Credential/auth issues (OAuth) or missing spreadsheet permissions.
  - If `snippet` isn’t present on selected topic object, Topic cell becomes empty.
- **Sticky note:** “Replace YOUR_GOOGLE_SHEET_ID with your own Google Sheet ID.”

#### Node: Get most recent topics from Google Sheet
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — read with filters.
- **Configuration choices (interpreted):**
  - Reads from `Sheet1` and filters rows where `is_generated` equals `"0"`.
  - Returns matching rows; typically includes `row_number` (used later for update).
- **Key expressions/variables:** Filter UI: `is_generated` == `0`.
- **Connections:**
  - **In:** `Add topic to Google Sheets`
  - **Out:** → `Generate content and image prompt using Perplexity AI`
- **Edge cases / failures:**
  - If the sheet has no matching rows, downstream generation will run with empty input (likely fails).
  - Ensure the sheet has an `is_generated` column and values exactly `"0"`.

---

### Block 4 — Content + Image Prompt Generation (Perplexity) and JSON Parsing
**Overview:** Generates structured post content in JSON, then sanitizes/parses the JSON for downstream use.  
**Nodes involved:**  
- Generate content and image prompt using Perplexity AI  
- Parse Content from string to JSON

#### Node: Generate content and image prompt using Perplexity AI
- **Type / role:** Perplexity node (`n8n-nodes-base.perplexity`) — content generation.
- **Configuration choices (interpreted):**
  - **Model:** `sonar-pro`
  - Prompt requests a strict JSON object with keys like `row_number`, `Topic`, `title`, `keywords`, `image_prompt`, `sections`, `additional_elements`.
  - Injects topic from Google Sheets: `Topic: {{ $json.Topic }}`
- **Key expressions/variables:**
  - `{{ $json.Topic }}` and `{{ $json.row_number }}`
- **Connections:**
  - **In:** `Get most recent topics from Google Sheet`
  - **Out:** → `Parse Content from string to JSON`
- **Edge cases / failures:**
  - **Potential prompt bug:** The sample JSON in the prompt is missing a comma after `"Topic": {{ $json.Topic }}` (as written in the prompt body). The model may still output valid JSON, but this increases risk of malformed output.
  - Model may wrap JSON in ```json fences; next node attempts to remove them.
  - If Perplexity returns unexpected schema (missing `sections`, `additional_elements`, etc.), downstream expressions will fail.

#### Node: Parse Content from string to JSON
- **Type / role:** Code node (`n8n-nodes-base.code`) — robust-ish JSON parsing and cleanup.
- **Configuration choices (interpreted):**
  - Reads: `$input.first().json.choices[0].message.content`
  - Trims and removes leading ```json and trailing ``` if present.
  - `JSON.parse()` and returns `{ success, error, data }`.
- **Key expressions/variables:**
  - Assumes OpenAI-like response shape: `choices[0].message.content`
- **Connections:**
  - **In:** `Generate content and image prompt using Perplexity AI`
  - **Out:** → `Update Google Sheet with generated content`
- **Edge cases / failures:**
  - If Perplexity node output doesn’t contain `choices[0].message.content`, this node fails.
  - If content isn’t valid JSON after cleanup, returns `success:false` but does **not** throw; downstream nodes reference `$json.data...` and will fail anyway (expression errors / undefined).
  - If parsed JSON is not an object, returns `success:false`.

---

### Block 5 — Persist Generated Content to Sheets + Create WordPress Draft
**Overview:** Writes generated title/content/SEO fields into the same sheet row, then creates a WordPress post draft.  
**Nodes involved:**  
- Update Google Sheet with generated content  
- Create a post

#### Node: Update Google Sheet with generated content
- **Type / role:** Google Sheets node (`n8n-nodes-base.googleSheets`) — update existing row.
- **Configuration choices (interpreted):**
  - **Operation:** Update row in `Sheet1`, matching on `row_number`.
  - Writes:
    - `title` from `$json.data.title`
    - `content` concatenated from `sections[0..4]` headings + contents
    - `keywords` as comma-joined array
    - `meta_title` = Topic
    - `meta_description` from `additional_elements.meta_description`
    - `row_number` = `$json.data.row_number` (also used as matcher)
- **Key expressions/variables:**
  - Title: `={{ $json.data.title }}`
  - Content: hard-coded concatenation of `sections[0]` through `sections[4]`
  - Keywords: `={{ $json.data.keywords.join() }}`
  - Matcher: `matchingColumns: ["row_number"]`
- **Connections:**
  - **In:** `Parse Content from string to JSON`
  - **Out:** → `Create a post`
- **Edge cases / failures:**
  - If fewer than 5 sections are returned, expressions like `sections[4].heading` will fail.
  - If `row_number` is missing, update won’t match any row or will error.
  - Does **not** set `is_generated` to `1`; therefore the same row may be regenerated in future runs unless handled externally.

#### Node: Create a post
- **Type / role:** WordPress node (`n8n-nodes-base.wordpress`) — creates WP post.
- **Configuration choices (interpreted):**
  - **Resource/operation:** Post → Create
  - `title` = `{{ $json.title }}` (coming from the Google Sheets update output)
  - `content` = `{{ $json.content }}`
  - `status` = `draft`
  - `authorId` = `4` (must exist in WP)
- **Connections:**
  - **In:** `Update Google Sheet with generated content`
  - **Out:** → `Generate featured image using HuggingFace FLUX`
- **Edge cases / failures:**
  - WP credential/auth errors.
  - Author ID invalid → request fails.
  - If `$json.title` / `$json.content` are not present in the incoming item, post may be empty or fail depending on WP configuration.

---

### Block 6 — Image Generation (HuggingFace) + Upload + Set Featured Media
**Overview:** Generates a featured image from the generated `image_prompt`, fixes binary metadata, uploads to WP media endpoint, then assigns it to the created post.  
**Nodes involved:**  
- Generate featured image using HuggingFace FLUX  
- Update name and extension of image  
- Upload image to WP media Library  
- Assign image to an article as a Featured image

#### Node: Generate featured image using HuggingFace FLUX
- **Type / role:** HTTP Request node (`n8n-nodes-base.httpRequest`) — calls HuggingFace inference router.
- **Configuration choices (interpreted):**
  - POST to: `https://router.huggingface.co/hf-inference/models/black-forest-labs/FLUX.1-schnell`
  - Sends JSON body: `{ "inputs": "<image_prompt>" }`
  - Uses response format **file** (stores binary in n8n).
  - Headers:
    - `Authorization: Bearer YOUR_TOKEN_HERE`
    - `Accept: image/png`
- **Key expressions/variables:**
  - Image prompt: `{{ $('Parse Content from string to JSON').item.json.data.image_prompt }}`
    - This is a **cross-node reference**; it does not rely on the current input item’s JSON.
- **Connections:**
  - **In:** `Create a post`
  - **Out:** → `Update name and extension of image`
- **Edge cases / failures:**
  - Invalid/expired HuggingFace token.
  - Model cold start/timeouts or 429 rate limits.
  - Returned content-type may not match `Accept`.
  - Cross-node reference can break if the referenced node didn’t run or produced no item.

#### Node: Update name and extension of image
- **Type / role:** Code node (`n8n-nodes-base.code`) — normalizes binary metadata.
- **Configuration choices (interpreted):**
  - Reads `$input.first().binary.data`.
  - Overwrites:
    - `fileName: generated-image.jpg`
    - `fileExtension: jpg`
    - `mimeType: image/jpeg`
  - Throws if no binary data.
- **Connections:**
  - **In:** `Generate featured image using HuggingFace FLUX`
  - **Out:** → `Upload image to WP media Library`
- **Edge cases / failures:**
  - If HuggingFace returns binary under a different key than `data`, this will fail.
  - Forces JPEG metadata while upstream requests PNG; usually OK for WP upload but may mismatch real bytes.

#### Node: Upload image to WP media Library
- **Type / role:** HTTP Request node (`n8n-nodes-base.httpRequest`) — uploads binary to WordPress REST Media endpoint.
- **Configuration choices (interpreted):**
  - POST to: `https://your-site.com/wp-json/wp/v2/media`
  - Authentication: **predefined credential type** `wordpressApi`
  - Body: `binaryData` from field `data`
  - Headers:
    - `Content-Disposition: attachment; filename=<fileName>.<fileExtension>`
    - `Content-Type: application/octet-stream`
- **Connections:**
  - **In:** `Update name and extension of image`
  - **Out:** → `Assign image to an article as a Featured image`
- **Version-specific notes:** HTTP Request node v4.3 supports response/binary options and credential types as configured.
- **Edge cases / failures:**
  - `your-site.com` placeholder must be replaced.
  - WordPress may reject the upload due to file type, size limits, or permission (needs `upload_files` capability).
  - Content-Type header is generic; some WP setups prefer actual MIME.

#### Node: Assign image to an article as a Featured image
- **Type / role:** HTTP Request node (`n8n-nodes-base.httpRequest`) — updates WordPress post to set featured media.
- **Configuration choices (interpreted):**
  - PUT to: `https://your-site.com/wp-json/wp/v2/posts/<postId>`
  - JSON body: `{ "featured_media": <mediaId> }`
  - `postId` is read from the `Create a post` node output: `$('Create a post').item.json.id`
  - `mediaId` is read from current node input: `$json.id` (WP media creation response)
- **Connections:**
  - **In:** `Upload image to WP media Library`
  - **Out:** none (end)
- **Edge cases / failures:**
  - If media upload response does not contain `id`, assignment fails.
  - If post creation failed or ID not accessible via cross-node reference, URL becomes invalid.
  - Requires WP permission to edit posts.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking 'Execute workflow' | Manual Trigger | Entry point (manual execution) | — | Find top trending topics | ## Publish WordPress Blog Posts from Trending Topics with Perplexity, Google Sheets and HuggingFace (setup + overview) |
| Find top trending topics | Perplexity | Discover trending topic(s) | When clicking 'Execute workflow' | Select a clean trending topic | ### 🔍 Trend Discovery — Perplexity searches for breaking news and viral topics, then selects one for content generation. |
| Select a clean trending topic | Code | Randomly select one topic from results | Find top trending topics | Add topic to Google Sheets | ### 🔍 Trend Discovery — Perplexity searches for breaking news and viral topics, then selects one for content generation. |
| Add topic to Google Sheets | Google Sheets | Append queued topic row | Select a clean trending topic | Get most recent topics from Google Sheet | ### ✍️ Content Generation — Topics are queued in Sheets, then Perplexity generates full blog content with SEO metadata. The JSON is parsed and saved back. |
| Get most recent topics from Google Sheet | Google Sheets | Retrieve rows where `is_generated=0` | Add topic to Google Sheets | Generate content and image prompt using Perplexity AI | ### ✍️ Content Generation — Topics are queued in Sheets, then Perplexity generates full blog content with SEO metadata. The JSON is parsed and saved back. |
| Generate content and image prompt using Perplexity AI | Perplexity | Generate structured blog JSON + image prompt | Get most recent topics from Google Sheet | Parse Content from string to JSON | ### ✍️ Content Generation — Topics are queued in Sheets, then Perplexity generates full blog content with SEO metadata. The JSON is parsed and saved back. |
| Parse Content from string to JSON | Code | Remove fences + parse model output to JSON | Generate content and image prompt using Perplexity AI | Update Google Sheet with generated content | ### ✍️ Content Generation — Topics are queued in Sheets, then Perplexity generates full blog content with SEO metadata. The JSON is parsed and saved back. |
| Update Google Sheet with generated content | Google Sheets | Update row with generated title/content/SEO fields | Parse Content from string to JSON | Create a post | ### ✍️ Content Generation — Topics are queued in Sheets, then Perplexity generates full blog content with SEO metadata. The JSON is parsed and saved back. |
| Create a post | WordPress | Create WordPress draft post | Update Google Sheet with generated content | Generate featured image using HuggingFace FLUX | ### 📝 WordPress Publishing — Creates a draft post with the generated title and content. |
| Generate featured image using HuggingFace FLUX | HTTP Request | Generate image via HuggingFace inference (binary file) | Create a post | Update name and extension of image | ### 🖼️ Featured Image — FLUX generates an image, which is uploaded to WordPress and set as the featured media. |
| Update name and extension of image | Code | Fix binary metadata for WP upload | Generate featured image using HuggingFace FLUX | Upload image to WP media Library | ### 🖼️ Featured Image — FLUX generates an image, which is uploaded to WordPress and set as the featured media. |
| Upload image to WP media Library | HTTP Request | Upload binary image to WP Media Library | Update name and extension of image | Assign image to an article as a Featured image | ### 🖼️ Featured Image — FLUX generates an image, which is uploaded to WordPress and set as the featured media. |
| Assign image to an article as a Featured image | HTTP Request | Set `featured_media` on created WP post | Upload image to WP media Library | — | ### 🖼️ Featured Image — FLUX generates an image, which is uploaded to WordPress and set as the featured media. |
| Sticky Note | Sticky Note | Documentation block | — | — | ## Publish WordPress Blog Posts from Trending Topics with Perplexity, Google Sheets and HuggingFace (setup + overview) |
| Sticky Note1 | Sticky Note | Documentation block | — | — | ### 🔍 Trend Discovery — Perplexity searches for breaking news and viral topics, then selects one for content generation. |
| Sticky Note2 | Sticky Note | Documentation block | — | — | ### ✍️ Content Generation — Topics are queued in Sheets, then Perplexity generates full blog content with SEO metadata. The JSON is parsed and saved back. |
| Sticky Note3 | Sticky Note | Documentation block | — | — | ### 📝 WordPress Publishing — Creates a draft post with the generated title and content. |
| Sticky Note4 | Sticky Note | Documentation block | — | — | ### 🖼️ Featured Image — FLUX generates an image, which is uploaded to WordPress and set as the featured media. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Trigger**
   1. Add node: **Manual Trigger**
   2. Name: `When clicking 'Execute workflow'`

2) **Trend discovery with Perplexity**
   1. Add node: **Perplexity**  
      - Name: `Find top trending topics`
      - Configure prompt to return **JSON only**, no citations (as in workflow).
      - Ensure your Perplexity credentials are set in n8n.
   2. Connect: Manual Trigger → Find top trending topics

3) **Select one topic**
   1. Add node: **Code**
      - Name: `Select a clean trending topic`
      - JS logic:
        - Read `$input.first().json.search_results`
        - Randomly select one element
        - Return `[{ json: selectedTopic }]`
   2. Connect: Find top trending topics → Select a clean trending topic

4) **Google Sheets: append topic**
   1. Add node: **Google Sheets**
      - Name: `Add topic to Google Sheets`
      - **Operation:** Append
      - **Document:** your spreadsheet (replace `YOUR_GOOGLE_SHEET_ID`)
      - **Sheet:** `Sheet1`
      - Map columns:
        - `Topic` = `={{ $json.snippet }}`
        - `is_generated` = `0`
      - Authenticate with Google Sheets OAuth2 credentials.
   2. Connect: Select a clean trending topic → Add topic to Google Sheets

5) **Google Sheets: fetch ungenerated rows**
   1. Add node: **Google Sheets**
      - Name: `Get most recent topics from Google Sheet`
      - **Operation:** Read / Get Many (filtered)
      - Filter: `is_generated` equals `0`
      - Same document/sheet as above
   2. Connect: Add topic to Google Sheets → Get most recent topics from Google Sheet

6) **Generate content JSON with Perplexity**
   1. Add node: **Perplexity**
      - Name: `Generate content and image prompt using Perplexity AI`
      - Model: `sonar-pro`
      - Prompt must:
        - Inject `Topic: {{ $json.Topic }}`
        - Require strict JSON fields including `row_number`, `title`, `keywords[]`, `image_prompt`, `sections[]`, and `additional_elements.meta_description`
        - Avoid markdown; output HTML inside JSON strings
   2. Connect: Get most recent topics from Google Sheet → Generate content and image prompt using Perplexity AI

7) **Parse Perplexity output into JSON**
   1. Add node: **Code**
      - Name: `Parse Content from string to JSON`
      - Implement:
        - Read `choices[0].message.content`
        - Strip ```json fences
        - `JSON.parse()`
        - Return `{ success, error, data }`
   2. Connect: Generate content and image prompt using Perplexity AI → Parse Content from string to JSON

8) **Update Google Sheet row with generated content**
   1. Add node: **Google Sheets**
      - Name: `Update Google Sheet with generated content`
      - **Operation:** Update
      - Match column: `row_number`
      - Map:
        - `row_number` = `={{ $json.data.row_number }}`
        - `title` = `={{ $json.data.title }}`
        - `keywords` = `={{ $json.data.keywords.join() }}`
        - `meta_title` = `={{ $json.data.Topic }}`
        - `meta_description` = `={{ $json.data.additional_elements.meta_description }}`
        - `content` = concatenate section headings/contents (ensure your generator returns enough sections)
   2. Connect: Parse Content from string to JSON → Update Google Sheet with generated content

9) **Create WordPress draft**
   1. Add node: **WordPress**
      - Name: `Create a post`
      - **Operation:** Create Post
      - Title: `={{ $json.title }}`
      - Content: `={{ $json.content }}`
      - Additional fields:
        - Status: `draft`
        - Author ID: set to a valid WP user id (workflow uses `4`)
      - Configure WordPress credentials (application password / OAuth / WP API credential type as supported).
   2. Connect: Update Google Sheet with generated content → Create a post

10) **Generate featured image via HuggingFace FLUX**
   1. Add node: **HTTP Request**
      - Name: `Generate featured image using HuggingFace FLUX`
      - Method: POST
      - URL: `https://router.huggingface.co/hf-inference/models/black-forest-labs/FLUX.1-schnell`
      - Send JSON body:
        - `inputs` = `{{ $('Parse Content from string to JSON').item.json.data.image_prompt }}`
      - Headers:
        - `Authorization: Bearer <your_huggingface_token>`
        - `Content-Type: application/json`
        - `Accept: image/png`
      - Response: **File** (binary)
   2. Connect: Create a post → Generate featured image using HuggingFace FLUX

11) **Fix binary metadata**
   1. Add node: **Code**
      - Name: `Update name and extension of image`
      - Read `$input.first().binary.data`
      - Force filename, extension, mimeType
      - Throw if missing binary
   2. Connect: Generate featured image using HuggingFace FLUX → Update name and extension of image

12) **Upload to WordPress media**
   1. Add node: **HTTP Request**
      - Name: `Upload image to WP media Library`
      - Method: POST
      - URL: `https://<your-site.com>/wp-json/wp/v2/media`
      - Authentication: predefined credential type `wordpressApi`
      - Body content type: **binary data**
      - Input binary field: `data`
      - Headers:
        - `Content-Disposition: attachment; filename={{$binary.data.fileName}}.{{ $binary.data.fileExtension }}`
        - `Content-Type: application/octet-stream`
   2. Connect: Update name and extension of image → Upload image to WP media Library

13) **Assign featured image to the created post**
   1. Add node: **HTTP Request**
      - Name: `Assign image to an article as a Featured image`
      - Method: PUT
      - URL: `https://<your-site.com>/wp-json/wp/v2/posts/{{ $('Create a post').item.json.id }}`
      - Authentication: predefined credential type `wordpressApi`
      - JSON body: `{ "featured_media": {{ $json.id }} }`
   2. Connect: Upload image to WP media Library → Assign image to an article as a Featured image

**Required credentials / secrets to set**
- Perplexity API credentials (in Perplexity nodes)
- Google Sheets OAuth2 credentials
- WordPress API credentials (`wordpressApi`), and site base URL replacement
- HuggingFace token (replace `YOUR_TOKEN_HERE`)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer |
| Replace `YOUR_GOOGLE_SHEET_ID` with your own Google Sheet ID. | Google Sheets nodes configuration |
| Replace `your-site.com` with your WordPress site URL. | WordPress Media/Post REST calls |
| Replace `YOUR_TOKEN_HERE` with your HuggingFace API token. | HuggingFace FLUX HTTP Request |
| Update `authorId` in the WordPress node to match your author. | WordPress “Create a post” node |

