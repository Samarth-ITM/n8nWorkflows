Create AI social media carousels from Google Sheets and auto-publish with Blotato

https://n8nworkflows.xyz/workflows/create-ai-social-media-carousels-from-google-sheets-and-auto-publish-with-blotato-13605


# Create AI social media carousels from Google Sheets and auto-publish with Blotato

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Create Pro-Level Social Media Carousels & Auto-Publish with Blotato  
**Purpose:** Turn one Google Sheets row (product + brand inputs) into a **3-slide, visually consistent** social media carousel using AI (Claude + Gemini image generation), **upload the slides to Blotato**, and **publish** to Instagram/Facebook/X (optionally scheduled).

**Primary use cases**
- E-commerce product marketing, agencies managing multiple brands, creators publishing carousel content at scale.
- Automated “spreadsheet-to-published-post” production with low per-post cost.

### 1.1 Logical Blocks (by dependency)

1. **Step 1 — Fetch Product Data & Prepare Content**  
   Schedule → read one eligible row → mark “Processing” → optionally scrape missing description/images → normalize payload.

2. **Step 2 — AI Creative Direction & Image Preparation**  
   Claude generates a strict JSON plan (visual system + copy + per-slide prompts) → fetch product/brand images as Base64 → create Gemini request items.

3. **Step 3 — Generate Slides with Visual Consistency Loop**  
   Sequential loop over slides → for slide 2+ attach previous slide images as references → Gemini generates slide → convert to file → upload to Blotato → persist slide URLs in workflow static memory.

4. **Step 4 — Publish**  
   Sort slide URLs → build caption + scheduling time → route by platform list → wait (throttling) → publish via Blotato nodes.

> Note: Despite the initial description promising Google Sheet updates to Published/Failed, the provided workflow JSON **does not include nodes** that write back “Published/Failed” or “Post URL”.

---

## 2. Block-by-Block Analysis

### Block 1 — Fetch Product Data & Prepare Content

**Overview:** Pulls the next row scheduled for “today”, marks it as Processing to prevent duplicates, and optionally scrapes product metadata via Jina AI if description or images are missing.

**Nodes involved:**  
- Schedule Trigger  
- Get Rows (Google Sheets)  
- Set Processing (Google Sheets update)  
- Has Description? (IF)  
- Scrape Product (Jina AI)  
- Merge Data (Code)

#### 2.1 Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow entry point, periodic execution.
- **Config (interpreted):** Runs every **30 minutes**.
- **Outputs:** Triggers `Get Rows`.
- **Failure modes:** n8n downtime, timezone expectations (schedule uses server timezone).

#### 2.2 Get Rows
- **Type / role:** `n8n-nodes-base.googleSheets` — reads a single candidate row.
- **Config (interpreted):**
  - Sheet: `carousel workflow` → `Sheet1`
  - Filters:
    - **Post Date** equals `{{$now.toFormat('yyyy-MM-dd')}}` (today)
    - **Status** filter UI also references “Status”, but the JSON shows a mixed/odd filter structure; effective behavior depends on n8n node implementation.
  - `returnFirstMatch: true` → only one row is returned.
- **Key expressions:**
  - `{{ $now.toFormat('yyyy-MM-dd') }}`
- **Input:** from Schedule Trigger.
- **Output:** to `Set Processing`.
- **Failure modes / edge cases:**
  - OAuth permission issues (Sheets), missing columns, sheet not found.
  - Filter misconfiguration could return unintended rows (e.g., not excluding already-processed rows if Status filtering is not correctly set).

#### 2.3 Set Processing
- **Type / role:** Google Sheets “update row” — locks the row.
- **Config (interpreted):**
  - Operation: **Update**
  - Matching column: `row_number`
  - Sets: `Status = "Processing"`
  - Uses `row_number = {{$json.row_number}}` from the fetched row.
- **Input:** from Get Rows.
- **Output:** to `Has Description?`
- **Failure modes / edge cases:**
  - Missing `row_number` in the incoming item (depends on Sheets node output).
  - Concurrent executions could still race if multiple triggers happen and the row selection isn’t atomic at the sheet level.

#### 2.4 Has Description?
- **Type / role:** `n8n-nodes-base.if` — decides whether scraping is required.
- **Config (interpreted):**
  - Condition checks whether **Product Description** and/or **Product Image(s) URL** are empty using expressions referencing `$('Get Rows').item.json[...]`.
  - “OR” combinator.
- **Outputs:**
  - **True** branch → `Scrape Product`
  - **False** branch → `Merge Data`
- **Failure modes / edge cases:**
  - Logic is slightly redundant (two emptiness checks overlap).
  - If *either* description or images are empty, scraping runs; if scraping fails, Merge Data attempts fallback but may still produce empty fields.

#### 2.5 Scrape Product
- **Type / role:** `n8n-nodes-base.jinaAi` — scrapes product page and extracts metadata/content.
- **Config (interpreted):**
  - URL: `{{ $('Get Rows').item.json['Product URL'] }}`
  - Image captioning enabled.
- **Credentials:** `jinaAiApi`
- **Input:** IF true branch.
- **Output:** to `Merge Data`
- **Failure modes / edge cases:**
  - Invalid product URL, anti-bot blocks, timeouts.
  - Scrape output schema differences (metadata keys vary by site).

#### 2.6 Merge Data
- **Type / role:** `n8n-nodes-base.code` — normalizes row + optional scrape into a single clean payload.
- **Config (interpreted):**
  - Builds `productDescription` from sheet value, else fallback to scraped metadata:
    - `og:description` → `twitter:description` → `description`
  - Builds a *single* fallback image from metadata (`og:image*`, `twitter:image*`, etc.) or regex extraction from HTML content.
  - Fixes protocol-relative (`//...`) and relative (`/...`) URLs using product URL origin.
  - Parses `Product Image(s) URL` as comma-separated links (split using regex that respects `http(s)://` boundaries).
  - Normalizes `socials` to lowercase array: `sheet['Socials'].split(',')...`
- **Key outputs (JSON):**
  - `rowNumber`, `brandLogo`, `productURL`, `productDescription`, `productImages[]`, `specification`, `postDate`, `postHour`, `socials[]`
- **Inputs:**
  - From `Scrape Product` (if ran) or directly from `Has Description?` (false branch).
  - References `$('Get Rows').first().json` and optionally `$('Scrape Product').first().json`.
- **Output:** to `Basic LLM Chain`
- **Failure modes / edge cases:**
  - If `Socials` is empty/null, `.split(',')` can throw (in current code it assumes existence).
  - Scrape errors are swallowed (`try/catch`), which can hide root causes and leave empty description/images.
  - Product image URLs may be non-image or blocked; later Base64 fetch may fail.

---

### Block 2 — AI Creative Direction & Image Preparation

**Overview:** Claude generates a strict structured JSON plan (visual identity + slide copy + prompts). Then the workflow fetches brand/product images and encodes them to Base64 for Gemini.

**Nodes involved:**  
- Basic LLM Chain  
- Anthropic Chat Model  
- Structured Output Parser  
- Parse AI Response (Code)  
- Fetch Images to Base64 (Code)  
- Prepare Nano Banana Items (Code)

#### 2.7 Basic LLM Chain
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — orchestrates prompt + model + structured parsing.
- **Config (interpreted):**
  - Prompt instructs: **EXACTLY 3 slides**, visual system first, per-slide imagePrompt ≥ 80 words, and strict JSON output.
  - Uses a **Structured Output Parser** to force schema compliance.
- **Connections:**
  - Input: `Merge Data`
  - AI Language Model input: `Anthropic Chat Model`
  - Output parser input: `Structured Output Parser`
  - Output: to `Parse AI Response`
- **Failure modes / edge cases:**
  - If model returns invalid JSON, parser may fail and stop execution.
  - Token limits: `maxTokensToSample: 4096` may be tight given the long system prompt; Claude “thinking” also consumes budget depending on provider behavior.

#### 2.8 Anthropic Chat Model
- **Type / role:** `lmChatAnthropic` — Claude model provider.
- **Config (interpreted):**
  - Model: `claude-sonnet-4-5-20250929`
  - Options: `thinking: true`, `maxTokensToSample: 4096`
- **Credentials:** `anthropicApi`
- **Failure modes:**
  - API auth, quota, model availability/name changes, latency/timeouts.

#### 2.9 Structured Output Parser
- **Type / role:** `outputParserStructured` — validates/enforces the JSON schema.
- **Config (interpreted):**
  - Provides a JSON schema example matching:
    - `caption`, `hashtags`, `visualSystem{...}`, `slides[{slideNumber, headline, body, layoutApproach, imagePrompt}]`
- **Failure modes / edge cases:**
  - If the model deviates (extra keys, wrong types), parsing fails.
  - If Claude returns `hashtags` not as a string (e.g., array), downstream concatenation may look wrong.

#### 2.10 Parse AI Response
- **Type / role:** Code — extracts parsed LLM output and merges with product assets and routing info.
- **Config (interpreted):**
  - Reads `const parsed = $input.first().json.output;`
  - Emits: `caption`, `hashtags`, `slides`, plus `productImages`, `brandLogo`, `rowNumber`, `socials` from `Merge Data`.
- **Input:** from Basic LLM Chain.
- **Output:** to `Fetch Images to Base64`
- **Failure modes:**
  - If the chain output path differs (not `.json.output`), this breaks.

#### 2.11 Fetch Images to Base64
- **Type / role:** Code — downloads images and Base64-encodes them.
- **Config (interpreted):**
  - For each product image URL: HTTP GET `encoding: arraybuffer` → Base64 string.
  - Same for `brandLogo`.
  - Outputs: `productImagesBase64[]`, `brandLogoBase64`
- **Failure modes / edge cases:**
  - Broken URLs, non-200 responses, large images causing memory pressure.
  - Some servers block hotlinking; may require headers or user-agent.
  - No MIME sniffing: later nodes assume JPEG/PNG.

#### 2.12 Prepare Nano Banana Items
- **Type / role:** Code — creates one n8n item per slide formatted for Gemini `generateContent`.
- **Config (interpreted):**
  - Builds `parts[]`:
    - First part: `{ text: prompt }` (augmented when multiple product images exist).
    - Then inline images: each product image as `mime_type: image/jpeg`
    - Then brand logo: `mime_type: image/png`
  - Adds `meta` per slide and a shared `generationConfig`:
    - `responseModalities: ["TEXT","IMAGE"]`
    - `imageConfig: aspectRatio 1:1, imageSize "2K"`
- **Output:** multiple items → `Loop Over Items`
- **Failure modes / edge cases:**
  - MIME mismatch if product images are PNG/WEBP; currently hard-coded to JPEG.
  - If Base64 array includes nulls, code filters for prompt logic but still iterates and skips null entries correctly.

---

### Block 3 — Generate Slides with Visual Consistency Loop

**Overview:** Iterates slides sequentially, adds prior generated slides as visual references for slide 2+, calls Gemini image generation, converts the returned Base64 to a binary file, uploads to Blotato, and stores the resulting media URLs in global static data for later iterations.

**Nodes involved:**  
- Loop Over Items (Split in Batches)  
- Add Previous Slides (Code)  
- Nano Banana Pro (HTTP Request)  
- Edit Fields1 (Set)  
- Convert to File  
- Blotato Upload  
- Save Slide to Memory (Code)

#### 2.13 Loop Over Items
- **Type / role:** `splitInBatches` — batching/loop controller.
- **Config (interpreted):**
  - Default batch settings (not explicitly set), used as a loop driver.
- **Connections (important):**
  - Input: from `Prepare Nano Banana Items`
  - Output 0 (main): to `Add Previous Slides` (per-item processing)
  - Output 1 (done): to `Prepare Post Payload` (after all slides processed)
- **Failure modes / edge cases:**
  - If batch size defaults to something >1, sequential consistency could be weakened. Typically you want **batch size = 1** for strict sequential generation.

#### 2.14 Add Previous Slides
- **Type / role:** Code — enforces consistency by attaching previous slides as inline reference images.
- **Config (interpreted):**
  - Uses `$getWorkflowStaticData('global')`:
    - If `slideNumber === 1`, resets `staticData.slideUrls = []`.
    - For slide 2+, downloads each stored URL → Base64 → appends to `parts[]` as `inline_data`.
  - Injects two prompt blocks:
    - **Rendering instructions** (product dominance, logo placement, typography size, avoid list, etc.)
    - **Consistency instructions** for slide 2+.
  - Outputs `contents: [{ parts }]` and `generationConfig`.
- **Input:** from Loop Over Items.
- **Output:** to `Nano Banana Pro`
- **Failure modes / edge cases:**
  - If Blotato URLs require auth or expire, GET download for references may fail.
  - Static data persists across executions unless reset; it is reset at slide 1, which is correct provided slide 1 always runs first.

#### 2.15 Nano Banana Pro
- **Type / role:** HTTP Request — calls Gemini image generation endpoint.
- **Config (interpreted):**
  - POST `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent`
  - Auth: `googlePalmApi` credential
  - Body: `{ contents: $json.contents, generationConfig: $json.generationConfig }`
  - Timeout: very high (`1000000`)
  - `retryOnFail: true`
- **Output:** to `Edit Fields1`
- **Failure modes / edge cases:**
  - API quota, model access, response schema changes.
  - Large request payloads (multiple inline images + previous slides) can hit size limits.

#### 2.16 Edit Fields1
- **Type / role:** Set — extracts returned Base64 image from Gemini response.
- **Config (interpreted):**
  - Sets `data = {{ $json.candidates[0].content.parts[0].inlineData.data }}`
- **Input:** from Nano Banana Pro.
- **Output:** to Convert to File.
- **Failure modes / edge cases:**
  - Response shape may differ (e.g., inline_data vs inlineData, part index not 0, image returned in a different part).
  - If Gemini returns text-only due to safety filters, `inlineData` may be missing.

#### 2.17 Convert to File
- **Type / role:** `convertToFile` — converts Base64 string to binary.
- **Config (interpreted):**
  - Operation: `toBinary`
  - Source property: `data`
- **Output:** to `Blotato Upload`
- **Failure modes:**
  - Invalid Base64 string, memory usage for large images.

#### 2.18 Blotato Upload
- **Type / role:** Blotato node — uploads generated image as media asset.
- **Config (interpreted):**
  - Resource: `media`
  - `useBinaryData: true` (uploads the binary from Convert to File)
- **Credentials:** `blotatoApi`
- **Output:** to `Save Slide to Memory`
- **Failure modes / edge cases:**
  - Auth/plan limits, file size limits, transient upload failures.

#### 2.19 Save Slide to Memory
- **Type / role:** Code — stores Blotato media URLs for reference + final publishing.
- **Config (interpreted):**
  - Reads uploaded URL: `$('Blotato Upload').first().json.url`
  - Pushes into `staticData.slideUrls`
  - Returns `meta` + `mediaUrl` for later sorting/publishing.
- **Output:** back to `Loop Over Items` (continues loop)
- **Failure modes:**
  - If Blotato upload response doesn’t contain `.url`, the chain breaks.

---

### Block 4 — Publish

**Overview:** Collects all generated slide URLs, builds the platform-specific payload (caption, scheduling time), routes by selected platforms, optionally waits, and publishes via Blotato.

**Nodes involved:**  
- Prepare Post Payload (Code)  
- Switch  
- Wait / Wait1 / Wait2  
- instagram / facebook / X (Blotato publish nodes)

#### 2.20 Prepare Post Payload
- **Type / role:** Code — aggregates per-slide outputs into one publish payload.
- **Config (interpreted):**
  - Sorts all items by `meta.slideNumber`
  - `mediaUrls`: joined as comma-separated string
  - Caption: `caption + hashtags`
  - Scheduling:
    - Reads `Post Hour` from sheet:
      - `now` → immediate (empty scheduledTime)
      - supports `2pm`, `14:30`, `14` etc.
    - Builds ISO-like string: `YYYY-MM-DDTHH:MM:00` (no timezone suffix)
- **Output:** to Switch
- **Failure modes / edge cases:**
  - If Post Hour parsing fails (NaN), scheduledTime becomes `YYYY-MM-DDTNaN:NaN:00`.
  - No timezone handling: publishing time interpretation depends on Blotato/account settings.

#### 2.21 Switch
- **Type / role:** Switch/router — fans out to the platforms requested in `socials[]`.
- **Config (interpreted):**
  - `allMatchingOutputs: true` (can publish to multiple platforms)
  - Outputs:
    - `instagram` if `{{$json.socials.includes('instagram')}}`
    - `facebook` if `{{$json.socials.includes('facebook')}}`
    - `X` if `{{$json.socials.includes('x')}}`
- **Output:** to Wait/Wait1/Wait2
- **Failure modes:**
  - If `socials` missing or not an array, `.includes()` errors.

#### 2.22 Wait / Wait1 / Wait2
- **Type / role:** Wait — throttling/delay before publish (10 units; in Wait node this is typically seconds).
- **Config:** `amount: 10`
- **Outputs:**
  - Wait → instagram
  - Wait1 → facebook
  - Wait2 → X
- **Failure modes:** minimal; but if Wait is configured for webhook resume in some modes, it can pause execution unexpectedly. (Here it appears to be a simple delay.)

#### 2.23 instagram (Blotato publish)
- **Type / role:** Blotato node — publish carousel to Instagram.
- **Config (interpreted):**
  - `accountId: 20209`
  - `postContentText: {{$json.caption}}`
  - `postContentMediaUrls: {{$json.mediaUrls.split(',')}}`
  - `scheduledTime: {{$json.scheduledTime || ''}}`
- **Failure modes:** invalid accountId, media URL accessibility, platform restrictions.

#### 2.24 facebook (Blotato publish)
- **Type / role:** Blotato node — publish to Facebook Page.
- **Config (interpreted):**
  - Platform: `facebook`
  - `accountId: 13039`
  - `facebookPageId: 703622779508585`
  - Text/media/scheduledTime as above
- **Failure modes:** page permissions, incorrect page ID, media constraints.

#### 2.25 X (Blotato publish)
- **Type / role:** Blotato node — publish to X/Twitter.
- **Config (interpreted):**
  - Platform: `twitter`
  - `accountId: 9184`
  - Text/media/scheduledTime as above
- **Failure modes:** X API/media limitations, auth revocation, scheduling support differences.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Time-based workflow entry | — | Get Rows | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Get Rows | googleSheets | Fetch next row for today | Schedule Trigger | Set Processing | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Set Processing | googleSheets | Mark row as Processing (lock) | Get Rows | Has Description? | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Has Description? | if | Decide whether to scrape missing product data | Set Processing | Scrape Product; Merge Data | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Scrape Product | jinaAi | Scrape product page metadata/content | Has Description? (true) | Merge Data | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Merge Data | code | Merge sheet + scrape into normalized payload | Has Description? (false) / Scrape Product | Basic LLM Chain | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Basic LLM Chain | langchain.chainLlm | Generate structured creative plan (3 slides) | Merge Data | Parse AI Response | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Anthropic Chat Model | langchain.lmChatAnthropic | Claude model provider | — (AI connection) | Basic LLM Chain | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Structured Output Parser | langchain.outputParserStructured | Enforce JSON schema | — (AI connection) | Basic LLM Chain | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Parse AI Response | code | Extract LLM JSON + attach assets references | Basic LLM Chain | Fetch Images to Base64 | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Fetch Images to Base64 | code | Download and Base64-encode images | Parse AI Response | Prepare Nano Banana Items | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Prepare Nano Banana Items | code | Build one Gemini request per slide | Fetch Images to Base64 | Loop Over Items | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Loop Over Items | splitInBatches | Sequential slide generation loop controller | Prepare Nano Banana Items / Save Slide to Memory | Add Previous Slides (loop); Prepare Post Payload (done) | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Add Previous Slides | code | Attach prior slide refs + add consistency prompt blocks | Loop Over Items | Nano Banana Pro | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Nano Banana Pro | httpRequest | Call Gemini image generation API | Add Previous Slides | Edit Fields1 | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Edit Fields1 | set | Extract returned Base64 image into `data` | Nano Banana Pro | Convert to File | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Convert to File | convertToFile | Convert Base64 to binary file | Edit Fields1 | Blotato Upload | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Blotato Upload | blotato | Upload slide image to Blotato media storage | Convert to File | Save Slide to Memory | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Save Slide to Memory | code | Persist slide URL in global static memory | Blotato Upload | Loop Over Items | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Prepare Post Payload | code | Aggregate slide URLs + caption + schedule time | Loop Over Items (done) | Switch | # 🔹 Step 4: Publish |
| Switch | switch | Route publish to requested platforms | Prepare Post Payload | Wait; Wait1; Wait2 | # 🔹 Step 4: Publish |
| Wait | wait | Delay/throttle before Instagram publish | Switch (instagram output) | instagram | # 🔹 Step 4: Publish |
| instagram | blotato | Publish carousel to Instagram | Wait | — | # 🔹 Step 4: Publish |
| Wait1 | wait | Delay/throttle before Facebook publish | Switch (facebook output) | facebook | # 🔹 Step 4: Publish |
| facebook | blotato | Publish carousel to Facebook Page | Wait1 | — | # 🔹 Step 4: Publish |
| Wait2 | wait | Delay/throttle before X publish | Switch (X output) | X | # 🔹 Step 4: Publish |
| X | blotato | Publish carousel to X/Twitter | Wait2 | — | # 🔹 Step 4: Publish |
| Sticky Note | stickyNote | Comment block label | — | — | # 🔹 Step 1: Fetch Product Data & Prepare Content |
| Sticky Note1 | stickyNote | Comment block label | — | — | # 🔹 Step 2: AI Creative Direction & Image Preparation |
| Sticky Note2 | stickyNote | Comment block label | — | — | # 🔹 Step 3: Generate Slides with Visual Consistency Loop |
| Sticky Note3 | stickyNote | Comment block label | — | — | # 🔹 Step 4: Publish |
| Sticky Note5 | stickyNote | Full workflow notes/resources | — | — | Social Media Carousel Generator AI-Powered Design + Auto-Publish — links: https://youtu.be/A_QT-9qUkxc, https://anthropic.com, https://ai.google.dev, https://blotato.com/?ref=nocodehack, https://www.linkedin.com/in/ing-seif/, https://youtube.com/@nocodehack |

---

## 4. Reproducing the Workflow from Scratch

1. **Create workflow**
   - Name it: “Create Pro-Level Social Media Carousels & Auto-Publish with Blotato”.

2. **Add Schedule Trigger**
   - Node: *Schedule Trigger*
   - Interval: every **30 minutes**.

3. **Add Google Sheets: Get Rows**
   - Node: *Google Sheets* → operation to **read rows** (or “Get Many”) with filters.
   - Credential: Google Sheets OAuth2.
   - Target spreadsheet + sheet tab.
   - Filter rows where:
     - `Post Date` equals today (`{{$now.toFormat('yyyy-MM-dd')}}`)
     - (Recommended) `Status` is empty (to avoid re-processing).
   - Option: **Return first match**.

4. **Add Google Sheets: Set Processing**
   - Node: *Google Sheets* → **Update**
   - Match row by `row_number`
   - Set `Status` = `Processing`
   - Map `row_number` from the fetched row.

5. **Add IF: Has Description?**
   - Node: *IF*
   - Condition: run scraping when either:
     - `Product Description` is empty, OR
     - `Product Image(s) URL` is empty
   - True output → Scraper; False output → Merge step.

6. **Add Jina AI: Scrape Product**
   - Node: *Jina AI*
   - URL: `{{$('Get Rows').item.json['Product URL']}}`
   - Enable image captioning (optional).
   - Credential: Jina AI API key.

7. **Add Code: Merge Data**
   - Node: *Code*
   - Implement:
     - Fallback description and image extraction from scraper metadata/content.
     - Normalize fields: `brandLogo`, `productDescription`, `productImages[]`, `specification`, `postHour`, `socials[]`, `rowNumber`.
   - Connect both:
     - IF False → Merge Data
     - Scrape Product → Merge Data

8. **Add LangChain: Basic LLM Chain**
   - Node: *Basic LLM Chain*
   - Prompt: instruct **exactly 3 slides**, strict JSON structure, include visual system + prompts.
   - Enable **structured output parsing**.

9. **Add Anthropic Chat Model**
   - Node: *Anthropic Chat Model*
   - Choose Claude model (e.g., Sonnet).
   - Set max output tokens (ensure enough room for JSON).
   - Connect it to the LLM Chain via the AI “languageModel” connection.

10. **Add Structured Output Parser**
   - Node: *Structured Output Parser*
   - Provide example schema with keys:
     - `caption`, `hashtags`, `visualSystem`, `slides[]` (3 objects).
   - Connect to the LLM Chain via the AI “outputParser” connection.

11. **Add Code: Parse AI Response**
   - Node: *Code*
   - Extract the parser output and pass forward:
     - `caption`, `hashtags`, `slides`, plus `productImages`, `brandLogo`, `rowNumber`, `socials`.

12. **Add Code: Fetch Images to Base64**
   - Node: *Code*
   - For each product image URL and logo URL:
     - HTTP GET as arraybuffer
     - Convert to Base64
   - Output `productImagesBase64[]`, `brandLogoBase64`.

13. **Add Code: Prepare Nano Banana Items**
   - Node: *Code*
   - Create **one item per slide** with:
     - `meta.slideNumber`, `meta.caption`, `meta.hashtags`, etc.
     - `parts[]` starting with `{text: imagePrompt}` plus inline product images and logo as `inline_data`.
     - `generationConfig` with square aspect ratio.

14. **Add Split In Batches: Loop Over Items**
   - Node: *Split In Batches*
   - Set **Batch size = 1** (recommended for strict sequential consistency).
   - Connect from Prepare Nano Banana Items → Loop Over Items.

15. **Add Code: Add Previous Slides**
   - Node: *Code*
   - Use `$getWorkflowStaticData('global')`:
     - On slide 1: reset `slideUrls=[]`
     - For slide 2+: download stored `slideUrls` and append as inline reference images.
   - Add prompt blocks for rendering rules + consistency rules.
   - Output `contents` and `generationConfig`.

16. **Add HTTP Request: Nano Banana Pro (Gemini)**
   - Node: *HTTP Request*
   - Method: POST
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent`
   - Auth: Google AI / PaLM credential type (`googlePalmApi` in n8n).
   - Headers: `Content-Type: application/json`
   - Body JSON: `{ "contents": {{$json.contents}}, "generationConfig": {{$json.generationConfig}} }`
   - Enable retry on fail; set suitable timeout.

17. **Add Set: Edit Fields**
   - Node: *Set*
   - Extract the returned base64 image into `data` from the Gemini response path.

18. **Add Convert to File**
   - Node: *Convert to File*
   - Operation: `toBinary`
   - Source property: `data`

19. **Add Blotato: Upload media**
   - Node: *Blotato* → resource `media`
   - Upload from binary (`useBinaryData = true`)
   - Credential: Blotato API key.

20. **Add Code: Save Slide to Memory**
   - Node: *Code*
   - Read uploaded `url`, push to `staticData.slideUrls`.
   - Output `meta.slideNumber` + `mediaUrl`.
   - Connect back to Split In Batches to continue loop.

21. **Add Code: Prepare Post Payload** (connect from Split In Batches “done” output)
   - Sort by `slideNumber`
   - Build:
     - `mediaUrls` list/CSV
     - `caption = caption + hashtags`
     - `scheduledTime` based on `Post Hour` parsing (`now` → blank)

22. **Add Switch**
   - Node: *Switch*
   - Conditions:
     - If socials includes `instagram` → output “instagram”
     - If socials includes `facebook` → output “facebook”
     - If socials includes `x` → output “X”
   - Enable “all matching outputs”.

23. **Add Wait nodes** (optional throttling)
   - Add one Wait after each Switch output (e.g., 10 seconds).

24. **Add Blotato publish nodes**
   - Instagram:
     - accountId = your Blotato Instagram account
     - text = caption
     - media urls = array
     - scheduledTime optional
   - Facebook:
     - platform = facebook
     - accountId + facebookPageId
   - X:
     - platform = twitter
     - accountId
   - Connect each from its Wait node.

25. **(Recommended additions not present in JSON) Add Google Sheets update for completion**
   - On success: set `Status = Published` and write `Post URL` from Blotato response.
   - On error: use an Error Trigger or catch branch to set `Status = Failed` with details.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Video link referenced in workflow notes | https://youtu.be/A_QT-9qUkxc |
| Anthropic (Claude API) | https://anthropic.com |
| Google AI / Gemini | https://ai.google.dev |
| Blotato (ref link) | https://blotato.com/?ref=nocodehack |
| Contact (LinkedIn) | https://www.linkedin.com/in/ing-seif/ |
| YouTube channel | https://youtube.com/@nocodehack |
| Sticky note mentions “Telegram Bot — notifications (optional)” | Not implemented in the provided nodes |
| Sheet status update to Published/Failed + Post URL is described but not implemented | Add Google Sheets update nodes after publishing and in error handling paths |