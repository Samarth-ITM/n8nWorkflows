Generate portfolio screenshots and Upwork copy with Firecrawl, ScreenshotOne, OpenAI and Google Workspace

https://n8nworkflows.xyz/workflows/generate-portfolio-screenshots-and-upwork-copy-with-firecrawl--screenshotone--openai-and-google-workspace-12952


# Generate portfolio screenshots and Upwork copy with Firecrawl, ScreenshotOne, OpenAI and Google Workspace

## 1. Workflow Overview

**Workflow name (n8n):** `006. ScreenshotOne - Portfolio Screenshots (Custom Sections) - WITH SETUP GUIDE`  
**Provided title:** Generate portfolio screenshots and Upwork copy with Firecrawl, ScreenshotOne, OpenAI and Google Workspace

**Purpose:**  
This workflow accepts a website URL via an n8n Form, renders the site with **Firecrawl** (JS-rendered HTML), uses **OpenAI** to (1) analyze the site and select meaningful section IDs to screenshot and (2) generate **Upwork portfolio copy**, then takes a set of screenshots via **ScreenshotOne**, uploads them to **Google Drive**, logs results to **Google Sheets**, and sends a **Telegram** notification.

**Primary use cases:**
- Automating portfolio asset creation (screenshots + written case study copy)
- Quickly capturing “hero / full page / key sections / mobile” screenshots for a website
- Maintaining a portfolio log with links to assets and a standard description format

### 1.1 Input Reception (Form Trigger)
Receives the website URL from a hosted n8n form.

### 1.2 Website Rendering (Firecrawl)
Calls Firecrawl to retrieve *rendered HTML* (including JS-executed content) so IDs/sections can be accurately detected.

### 1.3 AI Site Analysis (OpenAI)
Extracts IDs from rendered HTML, asks OpenAI to filter them down to “real content sections” worth screenshotting, and returns normalized analysis fields.

### 1.4 Screenshot Plan Generation (ScreenshotOne URL builder)
Builds a list of screenshot “jobs” (hero, fullpage, per-section, mobile) and constructs ScreenshotOne API URLs.

### 1.5 Screenshot Execution Loop + Upload (rate-limited)
Processes screenshot jobs one-by-one, filters failures, uploads successful screenshots to Google Drive, and waits 3 seconds between iterations to reduce rate limits.

### 1.6 Aggregate Results + AI Upwork Copy
Aggregates Drive links + analysis into a single object, asks OpenAI to generate Upwork JSON, parses and normalizes the response.

### 1.7 Persistence + Notification
Appends a row to Google Sheets and sends a Telegram message summarizing analysis, Upwork copy, and screenshot links.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form Trigger)
**Overview:** Collects a website URL from users and initiates the workflow.  
**Nodes involved:** `Portfolio Form`

#### Node: Portfolio Form
- **Type / role:** `Form Trigger` (entry point)
- **Configuration (interpreted):**
  - Form title: “Portfolio Screenshot Generator”
  - One required field: **Website URL** (accepts `example.com` or full URL)
  - Response text after submit: “✅ Portfolio generation started! ... Telegram notification...”
- **Key variables / outputs:**
  - Output JSON contains field label as key: `$json["Website URL"]`
- **Connections:**
  - **Out →** `Firecrawl Scrape`
- **Potential failures / edge cases:**
  - Invalid URL input (missing domain, typo). The workflow partially mitigates by auto-prefixing `https://` later.
- **Version notes:** Form Trigger `typeVersion 2.2`

---

### Block 2 — Website Rendering (Firecrawl)
**Overview:** Uses Firecrawl to render and return HTML so section IDs can be extracted reliably even on JS-heavy sites.  
**Nodes involved:** `Firecrawl Scrape`

#### Node: Firecrawl Scrape
- **Type / role:** `HTTP Request` to Firecrawl `/v1/scrape`
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.firecrawl.dev/v1/scrape`
  - Body (JSON) includes:
    - `url`: normalized from form input; if it doesn’t start with `http`, prefixes `https://`
    - `formats: ["html"]`
    - `onlyMainContent: false`
    - `waitFor: 3000`
    - `timeout: 30000`
  - Headers:
    - `Authorization` (intended to be Firecrawl API key bearer header)
    - `Content-Type: application/json`
  - Timeout option: 60s
- **Key expressions:**
  - URL normalization:
    - `{{ $json['Website URL'].startsWith('http') ? $json['Website URL'] : 'https://' + $json['Website URL'] }}`
- **Connections:**
  - **In ←** `Portfolio Form`
  - **Out →** `Prepare AI Request`
- **Potential failures / edge cases:**
  - Missing/invalid Firecrawl auth header → 401/403
  - Timeouts for heavy pages
  - Firecrawl response format differences (`data.html` vs `html`) are handled downstream
- **Version notes:** HTTP Request `typeVersion 4.2`

---

### Block 3 — AI Site Analysis Preparation (extract IDs + build OpenAI request)
**Overview:** Extracts candidate `id="..."` attributes from rendered HTML, filters likely section IDs, truncates/sanitizes HTML, and prepares a structured OpenAI chat request.  
**Nodes involved:** `Prepare AI Request`

#### Node: Prepare AI Request
- **Type / role:** `Code` (JavaScript transformer)
- **Configuration (interpreted):**
  - Reads:
    - Firecrawl HTML from the previous node
    - Original form input from `Portfolio Form`
  - Normalizes:
    - `siteUrl` (ensures protocol)
    - `siteName` (domain-derived)
  - Extracts IDs from HTML via regex:
    - `html.match(/id=["']([^"']+)["']/gi)`
  - Filters out “utility/framework” IDs with `excludePatterns`:
    - excludes `root, app, __next, __nuxt, gatsby, modal, tooltip, dropdown, nav, header, footer, cookie, popup...` etc.
    - excludes IDs starting with numbers, too short/long
  - Creates OpenAI request body:
    - model: `gpt-4o-mini`
    - temperature: `0.2`
    - max_tokens: `2000`
    - system prompt instructs to output **only valid JSON**
    - user prompt includes:
      - detected IDs list
      - required JSON schema including `section_ids`
      - first 25KB of cleaned HTML
- **Key outputs:**
  - `$json.requestBody` (for OpenAI)
  - `siteUrl`, `siteName`, `detectedIds`, `firecrawlSuccess`
- **Connections:**
  - **In ←** `Firecrawl Scrape`
  - **Out →** `AI Site Analysis`
- **Potential failures / edge cases:**
  - If HTML is empty, detected IDs become empty; AI may return poor results
  - Regex may capture many IDs; request payload size risk (mitigated by HTML truncation to 25KB)
  - Escaping/cleaning could still yield malformed prompt content in rare cases
- **Version notes:** Code node `typeVersion 2`

---

### Block 4 — AI Site Analysis Call + Parsing
**Overview:** Calls OpenAI chat completions to classify the site and pick section IDs; parses the response JSON and applies fallbacks if parsing fails.  
**Nodes involved:** `AI Site Analysis`, `Parse AI Response`

#### Node: AI Site Analysis
- **Type / role:** `HTTP Request` to OpenAI Chat Completions
- **Configuration (interpreted):**
  - POST `https://api.openai.com/v1/chat/completions`
  - Body: `{{ $json.requestBody }}`
  - Authentication: `predefinedCredentialType` = `openAiApi` (n8n OpenAI credential)
  - Timeout: 60s
- **Connections:**
  - **In ←** `Prepare AI Request`
  - **Out →** `Parse AI Response`
- **Potential failures / edge cases:**
  - OpenAI credential missing/invalid → 401
  - Rate limiting → 429 (no explicit retry configured here)
  - Model name not available in the account/region
- **Version notes:** HTTP Request `typeVersion 4.2`

#### Node: Parse AI Response
- **Type / role:** `Code` (parser/normalizer)
- **Configuration (interpreted):**
  - Reads:
    - OpenAI response: `response.choices[0].message.content`
    - Previous data from `Prepare AI Request` (siteUrl, siteName, detectedIds)
  - Parses JSON by extracting the first `{ ... }` block via regex and `JSON.parse`
  - On parse failure, uses a safe default analysis object
  - Normalizes `sectionIds`:
    - supports string or array
    - trims and removes leading `#`
  - Fallback behavior:
    - if AI returned no sections but detected IDs exist → use up to first 10 detected IDs
  - Converts booleans/values to strings suitable for Sheets logging
- **Key outputs:**
  - `siteUrl`, `siteName`
  - analysis fields: `blocks_count`, `blocks_list`, `has_blog`, `languages`, `site_topic`, etc.
  - `sectionIds` (final list used for screenshots)
  - `detectedIds`, `firecrawlSuccess`
- **Connections:**
  - **In ←** `AI Site Analysis`
  - **Out →** `Site Configuration`
- **Potential failures / edge cases:**
  - OpenAI response shape changes or missing `choices[0]` → runtime error
  - Model returns non-JSON despite prompt → parse fallback triggers (works but reduces quality)
- **Version notes:** Code node `typeVersion 2`

---

### Block 5 — Screenshot Job Configuration (ScreenshotOne)
**Overview:** Builds a list of screenshot tasks (hero, full page, per-section, mobile) and constructs ScreenshotOne “take screenshot” API URLs with consistent parameters.  
**Nodes involved:** `Site Configuration`

#### Node: Site Configuration
- **Type / role:** `Code` (screenshot plan builder)
- **Configuration (interpreted):**
  - Inputs: AI analysis output (`siteUrl`, `siteName`, `sectionIds`)
  - Hardcoded ScreenshotOne API key in code:
    - `const API_KEY = 'uSYTkX5PDESZnQ';` (should be replaced)
  - Generates screenshot plan:
    - Always: `hero` (1920x1080)
    - Always: `fullpage` (full_page=true, 1920 width)
    - For each sectionId:
      - adds screenshot with `scroll_into_view: #<sectionId>`
      - sets `error_on_selector_not_found: true` (forces failure if selector doesn’t exist)
    - Always at end: `mobile` (375x812, viewport_mobile=true)
  - Global ScreenshotOne parameters:
    - `block_cookie_banners=true`, `block_chats=true`, `block_ads=true`
    - `delay=3`
    - `device_scale_factor=2`
    - `format=png`
  - Produces per-item output objects containing:
    - `apiUrl` (fully formed ScreenshotOne request URL)
    - naming metadata (`filename`, `description`, etc.)
    - `aiAnalysis` attached for later aggregation
- **Connections:**
  - **In ←** `Parse AI Response`
  - **Out →** `Loop Over Items` (SplitInBatches)
- **Potential failures / edge cases:**
  - Invalid ScreenshotOne key → API returns error (handled later by filtering)
  - `error_on_selector_not_found=true` can cause many section screenshots to be skipped if IDs are wrong/dynamic
  - Some sites block bots; screenshots may return error pages (filtered by mimeType check later)
- **Version notes:** Code node `typeVersion 2`

---

### Block 6 — Screenshot Execution Loop + Filtering + Drive Upload + Rate Limiting
**Overview:** Iterates through screenshot jobs, downloads the image, filters out failed/non-image responses, uploads successful images to Google Drive, and throttles requests with a 3-second wait.  
**Nodes involved:** `Loop Over Items`, `Take Screenshot`, `Merge Screenshot Data`, `Upload to Google Drive`, `Wait 3s`

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` (loop controller)
- **Configuration (interpreted):**
  - Batch size is not explicitly set in JSON (defaults typically to 1 item per batch in UI usage; behavior should be verified in your n8n version).
  - Two outputs are used:
    - Output 0: “done” branch → aggregation
    - Output 1: per-item branch → take screenshot
- **Connections:**
  - **In ←** `Site Configuration`
  - **Out(0) →** `Aggregate All Results` (when loop completes)
  - **Out(1) →** `Take Screenshot` (for each screenshot config item)
- **Potential failures / edge cases:**
  - If configured batch size > 1, downstream index-based merging may mismatch screenshots vs configs
- **Version notes:** `typeVersion 3`

#### Node: Take Screenshot
- **Type / role:** `HTTP Request` (download screenshot file)
- **Configuration (interpreted):**
  - URL: `{{ $json.apiUrl }}`
  - Response format: **file** (binary)
  - Timeout: 120s
  - Retries: enabled (`retryOnFail: true`), `maxTries: 3`, wait 5s between tries
  - `onError: continueRegularOutput` so the workflow continues even if ScreenshotOne fails
  - `neverError: true` to avoid throwing on non-2xx (captures response for inspection)
- **Connections:**
  - **In ←** `Loop Over Items`
  - **Out →** `Merge Screenshot Data`
- **Potential failures / edge cases:**
  - 401/403 invalid key
  - 429 rate limiting (partly mitigated by Wait node after upload)
  - Some failures may return JSON or HTML; handled next
- **Version notes:** HTTP Request `typeVersion 4.2`

#### Node: Merge Screenshot Data
- **Type / role:** `Code` (validator + metadata merger)
- **Configuration (interpreted):**
  - Mode: `runOnceForEachItem`
  - Pulls original config via: `$('Site Configuration').item.json`
  - Validates success:
    - Must contain binary
    - Must not contain error markers in JSON (`is_successful === false`, `error_code`, `error_message`)
    - Binary mimeType must start with `image/`
  - If invalid → returns `[]` (drops item)
  - If valid → outputs combined JSON + binary for upload
- **Connections:**
  - **In ←** `Take Screenshot`
  - **Out →** `Upload to Google Drive`
- **Potential failures / edge cases:**
  - If batch size is not 1, `$('Site Configuration').item` may not match the current item
  - mimeType may be missing for some responses → could drop valid images depending on n8n binary metadata
- **Version notes:** Code node `typeVersion 2`

#### Node: Upload to Google Drive
- **Type / role:** `Google Drive` (file upload)
- **Configuration (interpreted):**
  - Upload name: `{{ $json.filename }}`
  - Drive: “My Drive”
  - **Folder ID is blank** in the workflow (must be set)
  - Uses Google Drive OAuth2 credential
- **Connections:**
  - **In ←** `Merge Screenshot Data`
  - **Out →** `Wait 3s`
- **Potential failures / edge cases:**
  - Missing/invalid OAuth → auth error
  - Folder ID empty/invalid → upload may go to root or fail (depends on node behavior/version)
  - Large files or API quota issues
- **Version notes:** Google Drive `typeVersion 3`

#### Node: Wait 3s
- **Type / role:** `Wait` (throttling)
- **Configuration (interpreted):**
  - Wait amount: `3` seconds
- **Connections:**
  - **In ←** `Upload to Google Drive`
  - **Out →** `Loop Over Items` (continues loop)
- **Potential failures / edge cases:**
  - Long queue times if many screenshots (by design)
- **Version notes:** Wait `typeVersion 1.1`

---

### Block 7 — Aggregation of Outputs (build one consolidated record)
**Overview:** After the loop finishes, collects Drive URLs and analysis metadata, computes success counts, and prepares dynamic columns for Sheets.  
**Nodes involved:** `Aggregate All Results`

#### Node: Aggregate All Results
- **Type / role:** `Code` (aggregator)
- **Configuration (interpreted):**
  - Reads:
    - All items from its input (which are Drive upload results coming from the loop completion branch)
    - All configs from `Site Configuration`
    - All successful merged items from `Merge Screenshot Data`
  - Builds `screenshots[]` array with:
    - `name`, `description`, `filename`, `driveUrl (webViewLink)`, `driveId`
  - Computes:
    - `screenshots_attempted` = number of config items
    - `screenshots_successful` = number of uploaded items
  - Creates a `sheetsRow` object with:
    - site metadata, timestamp
    - AI fields
    - screenshot method label: `'Firecrawl Rendered HTML'`
    - `detected_ids`, `section_ids_used`
  - Adds dynamic columns:
    - `img_<n>_<name> = driveUrl`
- **Connections:**
  - **In ←** `Loop Over Items` (done branch)
  - **Out →** `Prepare Upwork Request`
- **Potential failures / edge cases:**
  - **Index mismatch risk:** it maps `items.map((item, index) => mergeItems[index])`. If any screenshots were dropped earlier, indices can diverge (Drive uploads count equals successful screenshots, but mergeItems also only includes successful ones; still, any reordering could misalign metadata).
  - If Drive node doesn’t return `webViewLink` (permissions/settings), links may be missing.
- **Version notes:** Code node `typeVersion 2`

---

### Block 8 — Upwork Copy Generation (OpenAI) + Parsing
**Overview:** Uses the consolidated result to request a strict JSON Upwork portfolio entry, then parses and enforces constraints (title length, exactly 5 skills, etc.).  
**Nodes involved:** `Prepare Upwork Request`, `AI Generate Upwork`, `Parse Upwork Response`

#### Node: Prepare Upwork Request
- **Type / role:** `Code` (prompt builder)
- **Configuration (interpreted):**
  - Creates `upworkRequestBody` for OpenAI:
    - model: `gpt-4o-mini`
    - temperature: `0.4`
    - max_tokens: `800`
    - system rules:
      - title max 50 chars
      - description max 600 chars
      - exactly 5 skills
      - first-person developer voice
      - JSON only, no markdown
  - Passes through all aggregated data and attaches `upworkRequestBody`
- **Connections:**
  - **In ←** `Aggregate All Results`
  - **Out →** `AI Generate Upwork`
- **Potential failures / edge cases:**
  - If screenshotCount is 0, description quality may degrade but still works
- **Version notes:** Code node `typeVersion 2`

#### Node: AI Generate Upwork
- **Type / role:** `HTTP Request` to OpenAI chat completions
- **Configuration (interpreted):**
  - POST `https://api.openai.com/v1/chat/completions`
  - Body: `{{ $json.upworkRequestBody }}`
  - Uses OpenAI credential (`openAiApi`)
- **Connections:**
  - **In ←** `Prepare Upwork Request`
  - **Out →** `Parse Upwork Response`
- **Potential failures / edge cases:**
  - Rate limits (429), transient failures
- **Version notes:** HTTP Request `typeVersion 4.2`

#### Node: Parse Upwork Response
- **Type / role:** `Code` (JSON parser + constraints)
- **Configuration (interpreted):**
  - Parses the AI message content as JSON (same `{...}` extraction approach)
  - Fallback Upwork object on parse error
  - Ensures skills:
    - converts string → array if needed
    - truncates to 5
    - pads with “Web Development” to reach 5
  - Builds final result object:
    - includes analysis fields, screenshot info, Upwork fields
    - copies all dynamic `img_...` keys from previous data for Google Sheets
  - **Note:** It references some fields that may not exist in this workflow (`ai_found_anchors`, `anchors_checked`, `aiFoundAnchors`, `allAnchors`). These will likely be `undefined` unless present upstream.
- **Connections:**
  - **In ←** `AI Generate Upwork`
  - **Out →** `Save to Google Sheets`
- **Potential failures / edge cases:**
  - If OpenAI returns invalid JSON: fallback triggers
  - If `response.choices[0]` missing: runtime error
- **Version notes:** Code node `typeVersion 2`

---

### Block 9 — Save + Notify
**Overview:** Appends a row to a Google Sheet and sends a Telegram message with analysis, Upwork copy, and screenshot links.  
**Nodes involved:** `Save to Google Sheets`, `Telegram Notification`

#### Node: Save to Google Sheets
- **Type / role:** `Google Sheets` (append row)
- **Configuration (interpreted):**
  - Operation: `append`
  - Spreadsheet is set to a specific document ID (must be changed for your environment if needed)
  - Sheet: `Sheet1` (`gid=0`)
  - Mapping mode: auto-map input data to defined schema columns
  - Large predefined schema includes many `img_*` columns to accommodate various sites/section patterns.
- **Connections:**
  - **In ←** `Parse Upwork Response`
  - **Out →** `Telegram Notification`
- **Potential failures / edge cases:**
  - Document ID not accessible by the OAuth account → 403/404
  - Column explosion: if your output keys don’t match the schema, some fields may not write as expected
  - Sheet headers: sticky note claims first row auto-created on first run; actual behavior depends on node configuration and n8n version (often you must have headers or map explicitly)
- **Version notes:** Google Sheets `typeVersion 4.5`

#### Node: Telegram Notification
- **Type / role:** `Telegram` (send message)
- **Configuration (interpreted):**
  - Sends Markdown message including:
    - site name + URL
    - analysis summary
    - Firecrawl detected/used IDs
    - screenshot success counts
    - Upwork title/role/skills/description
    - list of screenshot links generated from `$json.screenshots`
  - `disable_web_page_preview: true`
  - **Chat ID is not visible in the provided JSON** (likely set in credentials or node UI fields); sticky note indicates it must be set.
- **Connections:**
  - **In ←** `Save to Google Sheets`
  - **Out:** none (end)
- **Potential failures / edge cases:**
  - Invalid bot token / missing chat ID → send fails
  - Markdown formatting issues if AI text contains reserved characters
- **Version notes:** Telegram `typeVersion 1.2`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Main Guide | Sticky Note | Documentation / setup guide |  |  | # 📋 Portfolio Screenshot Generator - Setup Guide … Firecrawl: https://firecrawl.dev ; ScreenshotOne: https://screenshotone.com ; OpenAI: https://platform.openai.com/api-keys |
| Sticky Note - Firecrawl | Sticky Note | Documentation / Firecrawl setup |  |  | ## 🔑 STEP 1: Firecrawl API Key … https://firecrawl.dev … replace with your key in the Authorization header |
| Sticky Note - ScreenshotOne | Sticky Note | Documentation / ScreenshotOne setup |  |  | ## 🔑 STEP 2: ScreenshotOne API Key … https://screenshotone.com … `const API_KEY = '';` |
| Sticky Note - OpenAI | Sticky Note | Documentation / OpenAI setup |  |  | ## 🔑 STEP 3: OpenAI Credentials … https://platform.openai.com/api-keys … Model used: gpt-4o-mini |
| Sticky Note - Google Drive | Sticky Note | Documentation / Drive setup |  |  | ## 📁 STEP 4: Google Drive Setup … ⚠️ CHANGE FOLDER ID … drive.google.com/drive/folders/**YOUR_ID** |
| Sticky Note - Google Sheets | Sticky Note | Documentation / Sheets setup |  |  | ## 📊 STEP 5: Google Sheets Setup … ⚠️ CHANGE DOCUMENT ID … docs.google.com/spreadsheets/d/**YOUR_ID**/edit |
| Sticky Note - Telegram | Sticky Note | Documentation / Telegram setup |  |  | ## 📱 STEP 6: Telegram Setup … ⚠️ CHANGE CHAT ID … api.telegram.org/bot**TOKEN**/getUpdates |
| Sticky Note - Customization | Sticky Note | Documentation / tuning options |  |  | ## 🎨 CUSTOMIZATION OPTIONS … Screenshot settings … AI model … Rate limiting |
| Sticky Note - Flow | Sticky Note | Documentation / flow + error handling |  |  | ## 🔄 WORKFLOW FLOW … “Form → Firecrawl → AI Analysis → Screenshots → Upload → AI Upwork → Save → Notify” … retries + filtering |
| Sticky Note - Trigger | Sticky Note | Documentation / form URL usage |  |  | ## 🚀 TRIGGER … Share this URL to allow others to submit websites… |
| Portfolio Form | Form Trigger | Collect website URL |  | Firecrawl Scrape | (Trigger note) Form URL after activating; share with others |
| Firecrawl Scrape | HTTP Request | Render site and fetch HTML | Portfolio Form | Prepare AI Request | (Firecrawl note) Add API key in Authorization header |
| Prepare AI Request | Code | Extract IDs, build OpenAI analysis request | Firecrawl Scrape | AI Site Analysis | (OpenAI note) Uses gpt-4o-mini, credential required |
| AI Site Analysis | HTTP Request | OpenAI site/section analysis | Prepare AI Request | Parse AI Response | (OpenAI note) Uses OpenAI credential |
| Parse AI Response | Code | Parse/normalize analysis + section IDs | AI Site Analysis | Site Configuration | (Flow note) continues with fallbacks if parsing fails |
| Site Configuration | Code | Build ScreenshotOne job list + URLs | Parse AI Response | Loop Over Items | (ScreenshotOne note) Replace `API_KEY` in code |
| Loop Over Items | Split In Batches | Iterate screenshot jobs | Site Configuration; Wait 3s | Aggregate All Results; Take Screenshot | (Flow note) loop one-by-one to reduce rate limits |
| Take Screenshot | HTTP Request | Download screenshot binary | Loop Over Items | Merge Screenshot Data | (Flow note) retries (3 attempts), continues on error |
| Merge Screenshot Data | Code | Filter failed screenshots; attach metadata | Take Screenshot | Upload to Google Drive | (Flow note) filters failures out |
| Upload to Google Drive | Google Drive | Upload image to Drive folder | Merge Screenshot Data | Wait 3s | (Google Drive note) set Folder ID; OAuth required |
| Wait 3s | Wait | Throttle requests | Upload to Google Drive | Loop Over Items | (Customization note) adjust delay if rate limited |
| Aggregate All Results | Code | Consolidate analysis + Drive links for all screenshots | Loop Over Items | Prepare Upwork Request | (Flow note) builds final dataset even with partial failures |
| Prepare Upwork Request | Code | Build OpenAI request for Upwork entry | Aggregate All Results | AI Generate Upwork | (OpenAI note) model configurable |
| AI Generate Upwork | HTTP Request | OpenAI generates Upwork JSON | Prepare Upwork Request | Parse Upwork Response | (OpenAI note) OpenAI credential required |
| Parse Upwork Response | Code | Parse/validate Upwork JSON; enforce 5 skills | AI Generate Upwork | Save to Google Sheets | (Flow note) fallback used on parse errors |
| Save to Google Sheets | Google Sheets | Append row with results | Parse Upwork Response | Telegram Notification | (Google Sheets note) change Document ID; OAuth required |
| Telegram Notification | Telegram | Send final summary + links | Save to Google Sheets |  | (Telegram note) configure bot token + chat ID |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Form Trigger node** named **“Portfolio Form”**:
   - Title: `Portfolio Screenshot Generator`
   - Description: `Enter website URL to analyze and generate portfolio screenshots with Upwork description.`
   - Field:
     - Label: `Website URL`
     - Required: true
     - Placeholder: `example.com or https://example.com`
   - Response on submit: `✅ Portfolio generation started! You will receive a Telegram notification when ready.`

3. **Add an HTTP Request node** named **“Firecrawl Scrape”**:
   - Method: `POST`
   - URL: `https://api.firecrawl.dev/v1/scrape`
   - Send headers: on
     - `Authorization: Bearer <YOUR_FIRECRAWL_API_KEY>` (or whatever Firecrawl requires for your plan)
     - `Content-Type: application/json`
   - Body type: JSON
   - Body (use expression for `url`):
     - `url`: `{{ $json['Website URL'].startsWith('http') ? $json['Website URL'] : 'https://' + $json['Website URL'] }}`
     - `formats`: `["html"]`
     - `onlyMainContent`: `false`
     - `waitFor`: `3000`
     - `timeout`: `30000`
   - Timeout option: ~60s
   - **Connect:** `Portfolio Form → Firecrawl Scrape`

4. **Add a Code node** named **“Prepare AI Request”**:
   - Paste logic to:
     - Normalize `siteUrl` from form input
     - Extract `siteName` from domain
     - Read HTML from Firecrawl response (`data.html` fallback)
     - Extract and filter IDs
     - Truncate/sanitize HTML to ~25KB
     - Build `requestBody` for OpenAI chat completions (model `gpt-4o-mini`)
   - **Connect:** `Firecrawl Scrape → Prepare AI Request`

5. **Add an HTTP Request node** named **“AI Site Analysis”**:
   - Method: `POST`
   - URL: `https://api.openai.com/v1/chat/completions`
   - Authentication: **Predefined credential type** → `OpenAI API`
     - Create credential in **Settings → Credentials** (OpenAI API key from https://platform.openai.com/api-keys)
   - Body JSON: `{{ $json.requestBody }}`
   - Timeout: ~60s
   - **Connect:** `Prepare AI Request → AI Site Analysis`

6. **Add a Code node** named **“Parse AI Response”**:
   - Parse `choices[0].message.content` as JSON (extract `{...}` region)
   - Normalize `sectionIds` (remove leading `#`)
   - Fallback to detected IDs if AI returns none
   - Output analysis fields + `sectionIds`
   - **Connect:** `AI Site Analysis → Parse AI Response`

7. **Add a Code node** named **“Site Configuration”**:
   - Build screenshot jobs array:
     - hero (desktop), fullpage (desktop), each `sectionId` via `scroll_into_view=#id`, mobile
   - Construct ScreenshotOne API URLs like:
     - `https://api.screenshotone.com/take?access_key=...&url=...&viewport_width=...&...`
   - Set parameters such as blocking ads/cookies/chats and `delay=3`
   - **Important:** Replace `API_KEY` with your ScreenshotOne key (get from https://screenshotone.com)
   - Output one item per screenshot config.
   - **Connect:** `Parse AI Response → Site Configuration`

8. **Add a Split In Batches node** named **“Loop Over Items”**:
   - Configure batch size to **1** (recommended to keep config/item alignment stable).
   - **Connect:** `Site Configuration → Loop Over Items`

9. **Add an HTTP Request node** named **“Take Screenshot”**:
   - URL: `{{ $json.apiUrl }}`
   - Response: **File** (binary)
   - Timeout: 120s
   - Enable retries (3) and 5s between tries
   - Set “Continue On Fail” behavior (or equivalent) so failures don’t stop the workflow
   - **Connect:** `Loop Over Items (items output) → Take Screenshot`

10. **Add a Code node** named **“Merge Screenshot Data”**:
    - Validate:
      - binary exists
      - no error flags in JSON
      - binary mimeType starts with `image/`
    - If invalid, return no items (drop)
    - If valid, output JSON metadata + binary
    - **Connect:** `Take Screenshot → Merge Screenshot Data`

11. **Add a Google Drive node** named **“Upload to Google Drive”**:
    - Operation: Upload
    - File name: `{{ $json.filename }}`
    - Folder ID: set to your destination folder
    - Credentials: create **Google Drive OAuth2** credential and select it
    - **Connect:** `Merge Screenshot Data → Upload to Google Drive`

12. **Add a Wait node** named **“Wait 3s”**:
    - Wait: 3 seconds
    - **Connect:** `Upload to Google Drive → Wait 3s`
    - **Connect loop:** `Wait 3s → Loop Over Items` (to process next screenshot)

13. **Add a Code node** named **“Aggregate All Results”**:
    - Triggered when loop finishes (the “done” output of SplitInBatches)
    - Combine:
      - site info + AI analysis
      - Drive `webViewLink` for each uploaded screenshot
      - counts attempted vs successful
      - dynamic `img_<n>_<name>` columns
    - **Connect:** `Loop Over Items (done output) → Aggregate All Results`

14. **Add a Code node** named **“Prepare Upwork Request”**:
    - Build `upworkRequestBody` for OpenAI with constraints (title/desc/skills)
    - Attach to the aggregated object
    - **Connect:** `Aggregate All Results → Prepare Upwork Request`

15. **Add an HTTP Request node** named **“AI Generate Upwork”**:
    - POST `https://api.openai.com/v1/chat/completions`
    - Auth: OpenAI credential (same as earlier)
    - Body: `{{ $json.upworkRequestBody }}`
    - **Connect:** `Prepare Upwork Request → AI Generate Upwork`

16. **Add a Code node** named **“Parse Upwork Response”**:
    - Parse JSON from AI response
    - Enforce exactly 5 skills
    - Truncate title/role/description to max lengths
    - Pass through screenshots + dynamic `img_...` keys
    - **Connect:** `AI Generate Upwork → Parse Upwork Response`

17. **Add a Google Sheets node** named **“Save to Google Sheets”**:
    - Operation: Append
    - Document: your spreadsheet ID
    - Sheet: your target sheet (e.g., `Sheet1`)
    - Map fields (auto-map or explicit mapping). Ensure headers exist if required by your setup.
    - Credentials: **Google Sheets OAuth2**
    - **Connect:** `Parse Upwork Response → Save to Google Sheets`

18. **Add a Telegram node** named **“Telegram Notification”**:
    - Credential: Telegram API (bot token from @BotFather)
    - Set Chat ID (your own or a group/channel)
    - Message text: include site info, analysis, Upwork fields, and links from `$json.screenshots`
    - Enable Markdown + disable link previews if desired
    - **Connect:** `Save to Google Sheets → Telegram Notification`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Firecrawl API key is required; Firecrawl is used to render JavaScript pages. | https://firecrawl.dev |
| ScreenshotOne access key must be added in the screenshot configuration Code node. | https://screenshotone.com |
| OpenAI API credential is required for “AI Site Analysis” and “AI Generate Upwork”. | https://platform.openai.com/api-keys |
| Google Drive OAuth2 credential required; set a destination Folder ID. | Drive folder ID from `drive.google.com/drive/folders/<ID>` |
| Google Sheets OAuth2 credential required; update spreadsheet Document ID. | Spreadsheet ID from `docs.google.com/spreadsheets/d/<ID>/edit` |
| Telegram bot token + chat ID required; chat ID can be found via getUpdates. | `https://api.telegram.org/bot<TOKEN>/getUpdates` |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.