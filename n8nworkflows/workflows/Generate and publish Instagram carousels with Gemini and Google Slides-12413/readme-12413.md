Generate and publish Instagram carousels with Gemini and Google Slides

https://n8nworkflows.xyz/workflows/generate-and-publish-instagram-carousels-with-gemini-and-google-slides-12413


# Generate and publish Instagram carousels with Gemini and Google Slides

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate and publish Instagram carousels with Gemini and Google Slides

**Purpose:**  
This workflow automatically generates a 6-slide Instagram carousel (copy + structure) using Google Gemini, injects that content into a Google Slides template, exports slide thumbnails, hosts them as public image URLs (ImgBB), then uploads them to the Instagram Graph API as carousel media, publishes the carousel, and logs/persists the results in Google Sheets.

**Typical use cases:**
- Daily automated educational carousel posting for creators/brands
- Content production pipelines using a design template (Google Slides) + AI copywriting
- Tracking publish status and historical posts in Sheets

### Logical Blocks
**1.1 Scheduling / Entry**  
Daily trigger starts the workflow.

**1.2 AI Content Generation (Gemini + Structured Parsing)**  
Gemini generates a strict JSON payload for slide text + caption, parsed into structured fields.

**1.3 Slide Template Duplication + Text Replacement (Google Drive/Slides)**  
Copies a Google Slides template, replaces placeholder tokens with generated content.

**1.4 Slide Export → Public Image Links (Slides → Binary → ImgBB)**  
Iterates slides, downloads thumbnails, converts binary, uploads each to ImgBB to obtain public URLs.

**1.5 Data Aggregation + Persistence (Google Sheets)**  
Aggregates slide URLs into an array and appends a record (date, caption, slide URLs) into a sheet; reads last row back for publishing inputs.

**1.6 Upload to Instagram (child media + carousel container)**  
Creates 6 child media objects on Meta, then creates a CAROUSEL container referencing them.

**1.7 Processing Status Polling + Publish + Status Update**  
Waits and polls until container is FINISHED, then publishes and updates Google Sheets status.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling / Entry
**Overview:** Starts the workflow once per day at a defined hour.

**Nodes involved:**
- Run daily schedule

**Node details**
- **Run daily schedule**
  - **Type / role:** `Schedule Trigger` — entry point.
  - **Config:** Runs daily at **06:00** (workflow timezone depends on n8n instance settings).
  - **Outputs to:** Generate carousel content
  - **Edge cases:** If instance is down at trigger time, run may be missed (unless n8n queue/trigger reliability features are configured).

---

### 2.2 AI Content Generation (Gemini + Structured Parsing)
**Overview:** Gemini generates carousel copy in a strict JSON structure; the structured parser enforces schema compliance.

**Nodes involved:**
- Generate carousel content
- Google Gemini Chat Model1
- Structured Output Parser1
- Sticky Note (AI content generation)

**Node details**
- **Google Gemini Chat Model1**
  - **Type / role:** LangChain Google Gemini chat model.
  - **Config:** Model `models/gemini-2.5-pro`.
  - **Connection:** Provides `ai_languageModel` input to **Generate carousel content**.
  - **Version notes:** Requires n8n LangChain nodes package versions compatible with `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`.
  - **Failure modes:** Invalid/expired Google AI credentials, model access restrictions, rate limiting.

- **Structured Output Parser1**
  - **Type / role:** LangChain structured output parser enforcing a JSON schema.
  - **Config:** Defines JSON schema with fields:
    - `hook.title`, `hook.description`
    - `mistake.title`, `mistake.content`
    - `why_it_matters.title`, `why_it_matters.content`
    - `value.title`, `value.content`
    - `tip.title`, `tip.tip_1`, `tip.tip_2`, `tip.tip_3`
    - `cta.content`
    - `post_description.caption`
  - **Connection:** Provides `ai_outputParser` to **Generate carousel content**.
  - **Edge cases:** Model returns invalid JSON or missing keys → parser throws an error and stops run.

- **Generate carousel content**
  - **Type / role:** LangChain Agent — prompts Gemini to produce daily content.
  - **Config highlights (interpreted):**
    - Prompt forces **raw JSON only**, no markdown, no extra text.
    - 6-slide structure + post description caption.
    - Style constraints (no emojis, “Spartan casual”, CTA wording rules).
    - **Has Output Parser enabled** → output is stored under `output`.
  - **Key outputs used later:**
    - `$('Generate carousel content').item.json.output.<section>...`
  - **Outputs to:** Duplicate carousel template
  - **Failure modes:** LLM output drift (format violations), timeouts, rate limits.

- **Sticky Note (AI content generation)**
  - **Content:**  
    “## AI content generation • Generate carousel copy with Gemini • Parse structured slide output”
  - Applies visually to the AI block.

---

### 2.3 Slide Template Duplication + Text Replacement
**Overview:** Copies a Google Slides template into the user’s Drive, then replaces placeholder tags with AI-generated text.

**Nodes involved:**
- Duplicate carousel template
- Replace text in template
- Get slides (carousels)
- Sticky Note (Slide creation)

**Node details**
- **Duplicate carousel template**
  - **Type / role:** Google Drive — copies a file (Slides template).
  - **Config:**
    - Operation: **copy**
    - Template file ID: `13N2Fykd9YYG6qvpobbuw4J-igaXztx7jUfI2FW0QTqg`
    - New name: `BakDesign n8n post auto`
  - **Credentials:** Google Drive OAuth2 (explicitly set in workflow)
  - **Outputs:** New file metadata including the copied presentation ID (used downstream).
  - **Failure modes:** Missing Drive permissions, template not shared/accessible, insufficient quota.

- **Replace text in template**
  - **Type / role:** Google Slides — replaces placeholder text.
  - **Config:**
    - Operation: **replaceText**
    - `presentationId`: `={{ $json.id }}` (from copied file)
    - Replacements map placeholders to Gemini output, e.g.:
      - `{{hook}}` → `output.hook.title`
      - `{{hook_description}}` → `output.hook.description`
      - `{{Tip_1}}` → `output.tip.tip_1`, etc.
  - **Outputs to:** Get slides (carousels)
  - **Edge cases:**
    - Placeholder typos (template uses different tokens) → text won’t replace.
    - Text length overflow may visually break slide design (not an execution error, but a content/layout risk).

- **Get slides (carousels)**
  - **Type / role:** Google Slides — fetch slide/page list.
  - **Config:** Operation **getSlides**, `presentationId` from previous node (`={{ $json.presentationId }}` as configured).
  - **Outputs:** Slide array, each with identifiers (notably `objectId`) used for thumbnail export.
  - **Failure modes:** Slides API access not enabled, permission issues, wrong presentationId mapping.

- **Sticky Note (Slide creation)**
  - **Content:**  
    “## Slide creation • Populate a Google Slides template with generated carousel content • Extract slide thumbnails and convert them into image files • Prepare slide images for upload and publishing”
  - Applies to nodes in this block and the export subchain.

---

### 2.4 Slide Export → Public Image Links (Slides → Binary → ImgBB)
**Overview:** Iterates over slides, downloads each thumbnail as binary, converts to a property, and uploads to ImgBB to get a public URL.

**Nodes involved:**
- Loop carousel slides
- get the imgs from slides
- convert
- make a link

**Node details**
- **Loop carousel slides**
  - **Type / role:** Split In Batches — used here as an iterator over slide items.
  - **Config:** Default options (batch size not explicitly set).
  - **Outputs to (two parallel paths):**
    - Aggregate (to gather slide URLs later)
    - get the imgs from slides (to export thumbnail → upload)
  - **Edge cases:** If batch size defaults to 1, it will emit items one by one; if slide count differs from expected 6, downstream “slide 1..6” publishing block may break.

- **get the imgs from slides**
  - **Type / role:** Google Slides — get thumbnail for a slide.
  - **Config:**
    - Resource: **page**
    - Operation: **getThumbnail**
    - Download: **true** (returns binary)
    - `pageObjectId`: `={{ $json.objectId }}`
    - `presentationId`: `={{ $('Replace text in template').item.json.presentationId }}`
  - **Outputs:** Binary thumbnail data for each slide.
  - **Failure modes:** Invalid objectId, API quota, missing permissions.

- **convert**
  - **Type / role:** Extract From File — converts binary to JSON property.
  - **Config:** Operation **binaryToPropery** (binary → `$json.data` buffer-like).
  - **Outputs to:** make a link
  - **Failure modes:** No binary present (if thumbnail download failed), memory constraints for large images.

- **make a link**
  - **Type / role:** HTTP Request — uploads image to ImgBB.
  - **Config:**
    - POST `https://api.imgbb.com/1/upload`
    - Multipart form-data:
      - `image`: `={{ $json.data.toString('base64') }}`
    - Query:
      - `key`: `YOUR_IMGBB_KEY_HERE`
  - **Outputs:** ImgBB response including `data.url` (public image URL).
  - **Outputs to:** Loop carousel slides (feeds iterator continuation)
  - **Failure modes:** Invalid ImgBB key, rate limits, payload too large, network errors.

---

### 2.5 Data Aggregation + Persistence (Google Sheets)
**Overview:** Aggregates the uploaded slide URLs, writes a new row to Google Sheets, reads the sheet, and isolates the last row for publishing.

**Nodes involved:**
- Aggregate
- add data
- get data
- get last row
- Sticky Note (Data preparation)

**Node details**
- **Aggregate**
  - **Type / role:** Aggregate — merges multiple items into one.
  - **Config:** `aggregateAllItemData` (collect all items’ data into a single array).
  - **Output structure:** A single item where `$json.data` is an array of each prior item (expected length = number of slides).
  - **Outputs to:** add data
  - **Edge cases:** If fewer than 6 slide URLs exist, later indexing `[0]..[5]` will fail.

- **add data**
  - **Type / role:** Google Sheets — append a new row.
  - **Config:**
    - Operation: **append**
    - Document: `1xkeUQaFBC2xzCJATkEPTJxhBPn2fY0NhhXHYVtoz8Ws` (sheet gid=0)
    - Columns written:
      - `date`: `={{ $now.toISODate() }}`
      - `caption`: `={{ $('Generate carousel content').item.json.output.post_description.caption }}`
      - `slide_1..slide_6`: from aggregated ImgBB URLs:
        - `={{ $json.data[0].data.url }}` … `={{ $json.data[5].data.url }}`
  - **Outputs to:** get data
  - **Failure modes:** Sheet permissions, schema mismatch (missing columns), Google API quota.

- **get data**
  - **Type / role:** Google Sheets — read rows.
  - **Config:** Reads from same document/sheet; range detection set to automatic.
  - **Outputs to:** get last row
  - **Edge cases:** Large sheet → slower reads; if sorting/filters exist, “last row” may not be the newest logical record.

- **get last row**
  - **Type / role:** Code — selects last item from all rows.
  - **Code logic:** `return [ rows[rows.length - 1] ];`
  - **Outputs to:** slide 1
  - **Failure modes:** Empty sheet → `rows.length - 1` is `-1` resulting in `undefined` and subsequent expression failures.

- **Sticky Note (Data preparation)**
  - **Content:**  
    “## Data preparation • Aggregate carousel metadata • Store slide and post data”
  - Applies to this block.

---

### 2.6 Upload to Instagram (child media + carousel container)
**Overview:** Creates 6 Instagram media objects (one per slide image URL), then creates a carousel container referencing all children.

**Nodes involved:**
- slide 1
- slide 2
- slide 3
- slide 4
- slide 5
- slide 6
- Container HTTP Carousel1
- Sticky Note (Upload images to a container)

**Node details**
- **slide 1**
  - **Type / role:** HTTP Request — create IG media object for first slide.
  - **Config:**
    - POST `https://graph.facebook.com/v24.0/YOUR_INSTAGRAM_ACCOUNT_ID_HERE/media`
    - Auth: **httpHeaderAuth** (Generic credential)
    - Query:
      - `image_url`: `={{ $json.slide_1 }}`
      - `caption`: `={{ $json.caption }}`
  - **Outputs to:** slide 2
  - **Edge cases:** Invalid Instagram Account ID, missing permissions (`instagram_content_publish`), token expired, image URL not publicly accessible.

- **slide 2..slide 6**
  - **Type / role:** HTTP Request — create IG media objects for slides 2–6.
  - **Config differences:** Each uses `image_url` from `$('get last row').item.json.slide_N` and caption from same last-row record.
  - **Outputs chaining:** slide2 → slide3 → slide4 → slide5 → slide6
  - **Failure modes:** Same as slide 1, plus intermittent failures on specific images.

- **Container HTTP Carousel1**
  - **Type / role:** HTTP Request — create CAROUSEL container.
  - **Config:**
    - POST `https://graph.facebook.com/v24.0/YOUR_INSTAGRAM_ACCOUNT_ID_HERE/media`
    - Query:
      - `children`: comma-separated list of media IDs:
        - `={{ $('slide 1').item.json.id }}, {{ $('slide 2').item.json.id }}, ... {{ $('slide 6').item.json.id }}`
      - `caption`: `={{ $('get last row').item.json.caption }}`
      - `media_type`: `CAROUSEL`
  - **Outputs to:** 🔍 Check Processing Status2
  - **Edge cases:**
    - If any slide creation failed, missing IDs break container creation.
    - Instagram may require `is_carousel_item=true` for children in some API patterns; this workflow relies on current behavior—verify with your Graph API version.
    - Caption duplication: caption is sent to each slide creation and also to container; typically caption should be on container only (not always fatal, but can be inconsistent).

- **Sticky Note (Upload images to a container)**
  - **Content:**  
    “## Upload images to a container • Upload each carousel slide to Meta as a media item • Combine slides into a single carousel container • Wait until Meta finishes processing media”
  - Applies to the upload/publish pipeline.

---

### 2.7 Processing Status Polling + Publish + Status Update
**Overview:** Waits briefly, polls the container status until FINISHED, publishes the carousel, then updates the sheet row as published.

**Nodes involved:**
- 🔍 Check Processing Status2
- ⏰ Initial Processing Wait1
- Check publish status
- Is carousel ready
- Retry publish check in 5 seconds
- publish
- update status
- Sticky Note (Publish status check)
- Sticky Note (Publish and update status)

**Node details**
- **🔍 Check Processing Status2**
  - **Type / role:** Set — stores container ID for later polling.
  - **Config:** Sets `container_id = {{$json.id}}` from carousel container creation response.
  - **Outputs to:** ⏰ Initial Processing Wait1
  - **Failure modes:** If container creation failed or returns no `id`, polling cannot proceed.

- **⏰ Initial Processing Wait1**
  - **Type / role:** Wait — introduces delay before first status check.
  - **Config:** No parameters shown (so it uses n8n Wait defaults; in UI it may represent a fixed time or “resume via webhook” depending on configuration—here it behaves as a delay node in the loop design).
  - **Outputs to:** Check publish status
  - **Failure modes:** If configured as “wait for webhook” unintentionally, the workflow will stall.

- **Check publish status**
  - **Type / role:** HTTP Request — checks container processing status.
  - **Config:**
    - GET `https://graph.facebook.com/v24.0/{{ container_id }}`
    - Auth: **httpQueryAuth** (Generic credential type)
    - Query: `fields=status_code`
  - **Outputs to:** Is carousel ready
  - **Important mismatch risk:** Other Meta requests use **httpHeaderAuth**, this node uses **httpQueryAuth**. If your token is only configured for header auth, this request will fail (401/403).

- **Is carousel ready**
  - **Type / role:** IF — checks whether processing is finished.
  - **Condition:** `$json.status_code == "FINISHED"`
  - **True output:** publish  
  - **False output:** Retry publish check in 5 seconds
  - **Edge cases:** Meta can also return `ERROR` or other states; workflow does not explicitly handle those.

- **Retry publish check in 5 seconds**
  - **Type / role:** Wait — delay before polling again.
  - **Config:** Shown as “in 5 seconds” by name; ensure the node is actually configured for 5 seconds in n8n UI.
  - **Outputs to:** Check publish status
  - **Failure modes:** Potential infinite loop if status never becomes FINISHED (no max retries).

- **publish**
  - **Type / role:** HTTP Request — publishes the finished carousel container.
  - **Config:**
    - POST `https://graph.facebook.com/v22.0/YOUR_INSTAGRAM_ACCOUNT_ID_HERE/media_publish`
    - Auth: httpHeaderAuth
    - Query: `creation_id = {{$json.id}}` (id from container status response when FINISHED)
  - **Outputs to:** update status
  - **Failure modes:** Publishing permissions missing, container not actually ready, API version differences (v22.0 vs v24.0 used elsewhere).

- **update status**
  - **Type / role:** Google Sheets — append or update status.
  - **Config:**
    - Operation: **appendOrUpdate**
    - Matching column: `slide_1`
    - Writes:
      - `status`: `"published"`
      - `slide_1`: `={{ $('get data').item.json.slide_1 }}`
    - Same document/sheet as earlier
  - **Risk:** It matches on `slide_1` using a value from `get data` (not from `get last row`). If `get data` returns multiple rows, `$('get data').item` points to the first item by default in many contexts; this can update the wrong row. Ideally it should match using `$('get last row').item.json.slide_1`.
  - **Failure modes:** Wrong row updated, permissions, schema mismatch.

- **Sticky Note (Publish status check)**
  - **Content:**  
    “## Publish status check • Verify publishing status • Retry until completed”
  - Applies to the polling loop nodes.

- **Sticky Note (Publish and update status)**
  - **Content:**  
    “## Publish and update status • Publish the carousel to Instagram • Update publishing status in Google Sheets • Track successful and failed posts”
  - Applies to publish/update nodes.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run daily schedule | Schedule Trigger | Daily workflow entry | — | Generate carousel content | ## Generate and publish Instagram carousels automatically… (Main Guide) |
| Generate carousel content | LangChain Agent | Generate JSON carousel script + caption | Run daily schedule; Google Gemini Chat Model1; Structured Output Parser1 | Duplicate carousel template | ## AI content generation… |
| Google Gemini Chat Model1 | Gemini Chat Model | LLM provider for agent | — | Generate carousel content (ai_languageModel) | ## AI content generation… |
| Structured Output Parser1 | Structured Output Parser | Enforce schema on LLM output | — | Generate carousel content (ai_outputParser) | ## AI content generation… |
| Duplicate carousel template | Google Drive | Copy Slides template into Drive | Generate carousel content | Replace text in template | ## Slide creation… |
| Replace text in template | Google Slides | Replace placeholders with generated copy | Duplicate carousel template | Get slides (carousels) | ## Slide creation… |
| Get slides (carousels) | Google Slides | Retrieve slide list/objectIds | Replace text in template | Loop carousel slides | ## Slide creation… |
| Loop carousel slides | Split In Batches | Iterate slides / control loop | Get slides (carousels); make a link | Aggregate; get the imgs from slides | ## Slide creation… |
| get the imgs from slides | Google Slides | Download slide thumbnail (binary) | Loop carousel slides | convert | ## Slide creation… |
| convert | Extract From File | Convert binary to JSON property | get the imgs from slides | make a link | ## Slide creation… |
| make a link | HTTP Request | Upload to ImgBB, get public URL | convert | Loop carousel slides | ## Slide creation… |
| Aggregate | Aggregate | Combine per-slide URLs into one item | Loop carousel slides | add data | ## Data preparation… |
| add data | Google Sheets | Append new post record | Aggregate | get data | ## Data preparation… |
| get data | Google Sheets | Read all rows | add data | get last row | ## Data preparation… |
| get last row | Code | Select newest row for publishing | get data | slide 1 | ## Data preparation… |
| slide 1 | HTTP Request | Create IG media item #1 | get last row | slide 2 | ## Upload images to a container… |
| slide 2 | HTTP Request | Create IG media item #2 | slide 1 | slide 3 | ## Upload images to a container… |
| slide 3 | HTTP Request | Create IG media item #3 | slide 2 | slide 4 | ## Upload images to a container… |
| slide 4 | HTTP Request | Create IG media item #4 | slide 3 | slide 5 | ## Upload images to a container… |
| slide 5 | HTTP Request | Create IG media item #5 | slide 4 | slide 6 | ## Upload images to a container… |
| slide 6 | HTTP Request | Create IG media item #6 | slide 5 | Container HTTP Carousel1 | ## Upload images to a container… |
| Container HTTP Carousel1 | HTTP Request | Create CAROUSEL container | slide 6 | 🔍 Check Processing Status2 | ## Upload images to a container… |
| 🔍 Check Processing Status2 | Set | Store container_id | Container HTTP Carousel1 | ⏰ Initial Processing Wait1 | ## Publish status check… |
| ⏰ Initial Processing Wait1 | Wait | Initial delay before polling | 🔍 Check Processing Status2 | Check publish status | ## Publish status check… |
| Check publish status | HTTP Request | Poll container status_code | ⏰ Initial Processing Wait1; Retry publish check in 5 seconds | Is carousel ready | ## Publish status check… |
| Is carousel ready | IF | Branch when FINISHED vs retry | Check publish status | publish; Retry publish check in 5 seconds | ## Publish status check… |
| Retry publish check in 5 seconds | Wait | Delay before next poll | Is carousel ready (false) | Check publish status | ## Publish status check… |
| publish | HTTP Request | Publish carousel container | Is carousel ready (true) | update status | ## Publish and update status… |
| update status | Google Sheets | Mark row as published | publish | — | ## Publish and update status… |
| Main Guide | Sticky Note | Comment / setup notes | — | — | ## Generate and publish Instagram carousels automatically… + template link |
| Sticky Note | Sticky Note | Comment block | — | — | ## AI content generation… |
| Sticky Note8 | Sticky Note | Comment block | — | — | ## Slide creation… |
| Sticky Note9 | Sticky Note | Comment block | — | — | ## Data preparation… |
| Sticky Note10 | Sticky Note | Comment block | — | — | ## Upload images to a container… |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Publish status check… |
| Sticky Note11 | Sticky Note | Comment block | — | — | ## Publish and update status… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger**
   1) Add **Schedule Trigger** node named “Run daily schedule”.  
   2) Set interval to **Daily** at **06:00**.

2. **Add AI generation (Gemini via LangChain)**
   1) Add **Google Gemini Chat Model** node, select model **`models/gemini-2.5-pro`**.  
      - Configure Google AI/Gemini credentials (API key or OAuth depending on your n8n setup).
   2) Add **Structured Output Parser** node.  
      - Paste/define schema with keys: `hook`, `mistake`, `why_it_matters`, `value`, `tip`, `cta`, `post_description` as in the workflow.
   3) Add **LangChain Agent** node named “Generate carousel content”.  
      - Prompt type: “Define” (custom prompt).  
      - Paste the provided prompt text (ensure it requests **raw JSON only**).  
      - Enable **Output Parser** and connect:
        - Gemini node → Agent as **AI Language Model**
        - Structured Output Parser → Agent as **AI Output Parser**
   4) Connect **Run daily schedule → Generate carousel content**.

3. **Duplicate the Google Slides template**
   1) Add **Google Drive** node “Duplicate carousel template” with operation **Copy**.  
      - File ID: the template presentation ID (e.g. `13N2Fykd9YYG6qvpobbuw4J-igaXztx7jUfI2FW0QTqg`).  
      - Name: your desired copy name.
      - Configure **Google Drive OAuth2** credentials.
   2) Connect **Generate carousel content → Duplicate carousel template**.

4. **Replace placeholders in Slides**
   1) Add **Google Slides** node “Replace text in template”, operation **Replace text**.  
      - `presentationId`: from copied file (`{{$json.id}}`).  
      - Add text replacements matching your template placeholders, e.g.:
        - `{{hook}}` → `{{ $('Generate carousel content').item.json.output.hook.title }}`
        - `{{hook_description}}` → `...output.hook.description`
        - Continue for all fields through `{{CTA}}`.
      - Configure **Google Slides** credentials (same Google project).
   2) Connect **Duplicate carousel template → Replace text in template**.

5. **Get slides list**
   1) Add **Google Slides** node “Get slides (carousels)” with operation **Get slides**.  
      - `presentationId`: the modified presentation ID from previous node output.
   2) Connect **Replace text in template → Get slides (carousels)**.

6. **Iterate slides and export thumbnails**
   1) Add **Split In Batches** node “Loop carousel slides”.  
      - Keep defaults or set batch size to 1 for clarity.
   2) Connect **Get slides (carousels) → Loop carousel slides**.
   3) Add **Google Slides** node “get the imgs from slides”:
      - Resource: Page
      - Operation: Get thumbnail
      - Download: true
      - `pageObjectId`: `{{$json.objectId}}`
      - `presentationId`: from “Replace text in template”
   4) Add **Extract From File** node “convert”:
      - Operation: binaryToProperty (ensure output becomes `$json.data`).
   5) Add **HTTP Request** node “make a link”:
      - POST `https://api.imgbb.com/1/upload`
      - Multipart form-data: `image` = `{{ $json.data.toString('base64') }}`
      - Query: `key` = your ImgBB API key
   6) Connect:  
      - Loop carousel slides → get the imgs from slides → convert → make a link  
      - make a link → Loop carousel slides (to continue batches)

7. **Aggregate URLs and store in Google Sheets**
   1) Add **Aggregate** node:
      - Mode: aggregate all item data into a single array (aggregateAllItemData).
   2) Connect **Loop carousel slides → Aggregate** (in parallel with thumbnail path).
   3) Add **Google Sheets** node “add data”, operation **Append**:
      - Document ID and Sheet (gid=0)
      - Columns:
        - date: `{{$now.toISODate()}}`
        - caption: `{{ $('Generate carousel content').item.json.output.post_description.caption }}`
        - slide_1..slide_6: `{{ $json.data[0].data.url }}` … `[5]`
   4) Add **Google Sheets** node “get data” to read the sheet.
   5) Add **Code** node “get last row” returning only the last item.
   6) Connect: **Aggregate → add data → get data → get last row**

8. **Create Instagram media objects (6 slides)**
   1) Prepare a Meta/Instagram Graph API credential:
      - Long-lived access token with required permissions for publishing.
      - Choose a consistent auth method (header recommended).
   2) Add **HTTP Request** node “slide 1”:
      - POST `https://graph.facebook.com/v24.0/<IG_ACCOUNT_ID>/media`
      - Query: `image_url={{$json.slide_1}}`, `caption={{$json.caption}}`
      - Auth: Header (Bearer token)
   3) Duplicate for “slide 2” … “slide 6”, changing `image_url` to `$('get last row').item.json.slide_N`.
   4) Chain them: **get last row → slide 1 → slide 2 → … → slide 6**

9. **Create carousel container**
   1) Add **HTTP Request** node “Container HTTP Carousel1”:
      - POST `https://graph.facebook.com/v24.0/<IG_ACCOUNT_ID>/media`
      - Query:
        - `children`: comma-separated IDs from slide 1..6 nodes
        - `caption`: from last row
        - `media_type=CAROUSEL`
   2) Connect **slide 6 → Container HTTP Carousel1**.

10. **Poll processing status until FINISHED**
   1) Add **Set** node “🔍 Check Processing Status2”:
      - Set `container_id = {{$json.id}}`
   2) Add **Wait** node “⏰ Initial Processing Wait1” (configure a short delay, e.g. 5–10s).
   3) Add **HTTP Request** node “Check publish status”:
      - GET `https://graph.facebook.com/v24.0/{{$('🔍 Check Processing Status2').item.json.container_id}}`
      - Query `fields=status_code`
      - Auth: must match your Meta token strategy (header or query), but keep it consistent.
   4) Add **IF** node “Is carousel ready”:
      - Condition: `{{$json.status_code}}` equals `FINISHED`
   5) Add **Wait** node “Retry publish check in 5 seconds” with 5 seconds delay.
   6) Connect loop:
      - Container → Set → Wait → Check publish status → IF
      - IF false → Retry wait → Check publish status

11. **Publish carousel and update sheet**
   1) Add **HTTP Request** node “publish”:
      - POST `https://graph.facebook.com/v22.0/<IG_ACCOUNT_ID>/media_publish`
      - Query: `creation_id={{$json.id}}`
   2) Add **Google Sheets** node “update status”, operation **Append or Update**:
      - Match on a stable key (recommended: use the same “last row” identifier).
      - Set `status = published`
   3) Connect: **IF true → publish → update status**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Generate and publish Instagram carousels automatically…” + setup steps + credits | Sticky note “Main Guide” |
| Template link: Copy this carousel Template I used to your Drive | https://docs.google.com/presentation/d/13N2Fykd9YYG6qvpobbuw4J-igaXztx7jUfI2FW0QTqg/edit |
| Author credit | “Happy setup, Bakdaulet Abdikhan” |

