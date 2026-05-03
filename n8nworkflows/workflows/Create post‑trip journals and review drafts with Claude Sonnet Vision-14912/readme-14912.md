Create post‑trip journals and review drafts with Claude Sonnet Vision

https://n8nworkflows.xyz/workflows/create-post-trip-journals-and-review-drafts-with-claude-sonnet-vision-14912


# Create post‑trip journals and review drafts with Claude Sonnet Vision

Good. I've verified all nodes and connections. Now let me write the complete document. 1. Workflow Overview

**Post-Trip Memory & Review Builder with Claude AI Vision** is an end-to-end automation that transforms travel photos and notes into structured memory packages. After a trip ends, it retrieves photos from Google Photos and notes from Google Sheets, uses Claude Sonnet 4 Vision to analyze image content, generate a narrative journal, draft platform-specific reviews, and assemble a highlights reel — then delivers everything via email and Google Drive.

**Target use cases:** Individual travelers or travel platforms that want to automatically produce post-trip journals, social-media-ready reviews, and curated highlight slideshows without manual writing or curation.

**Logical blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | Trip Intake & Validation | Accepts a webhook payload (or batch job), validates required fields and dates, normalizes preferences |
| 2 | Photo & Notes Collection | Fetches photos from Google Photos and trip notes from Google Sheets, extracts EXIF metadata, merges into a unified timeline |
| 3 | AI Photo Analysis & Content Generation | Three parallel Claude Sonnet 4 agents: (a) photo analysis & categorisation, (b) travel journal writing, (c) platform review drafting |
| 4 | Journal, Reviews & Delivery | Parses all AI outputs, generates a PDF journal, builds a highlights reel, saves to Google Drive and Sheets, emails the package, responds to the webhook caller |
| 5 | Daily Batch Processing & Analytics | A scheduled trigger scans a "trips" sheet for trips ended in the last 7 days that have not yet been processed, creates processing job payloads, and logs analytics |

The workflow has **two entry points**: the **Trip Completion Webhook** (real-time) and the **Daily Batch Processing** schedule (cron `0 6 * * *`).

---

## 2. Block-by-Block Analysis

---

### Block 1 — Trip Intake & Validation

**Overview:** Receives an inbound POST request with trip details, validates all required fields and date logic, normalizes photo-source and output-preference settings, and produces a clean `validatedTrip` object for downstream consumption.

**Nodes Involved:** Trip Completion Webhook, Validate Trip Data

#### Trip Completion Webhook
- **Type:** `n8n-nodes-base.webhook` (v2)
- **Role:** HTTP entry point; listens for POST requests at path `trip-completed`.
- **Configuration:** Method = POST, response mode = `responseNode` (response is deferred until the Send Response node fires).
- **Key expressions/variables:** Webhook ID = `trip-memory-builder`.
- **Input:** External HTTP client.
- **Output:** → Validate Trip Data (main, index 0).
- **Edge cases:** If the webhook path is already in use by another workflow, n8n will reject registration; the payload must be valid JSON or n8n will return a 400.

#### Validate Trip Data
- **Type:** `n8n-nodes-base.code` (v2, mode = `runOnceForEachItem`)
- **Role:** Validates the webhook body and produces a canonical `validatedTrip` record.
- **Configuration / logic:**
  - Required fields: `tripId`, `userId`, `userName`, `userEmail`, `tripDetails`.
  - `tripDetails` must contain `destination`, `startDate`, `endDate`.
  - Date checks: valid parse, end ≥ start, end ≤ today, duration ≤ 365 days.
  - Normalises `photoSources` (defaults `includeSharedPhotos = true`, builds date range from trip dates).
  - Normalises `outputPreferences` (default journal style = `narrative`, review platforms = `[tripadvisor, google]`, highlight count capped at 20, privacy = `private`).
  - Generates a unique `processingId` (`PROC-{timestamp}-{random}`).
- **Key expressions/variables:** All output stored under `json.validatedTrip`.
- **Input:** Trip Completion Webhook.
- **Output:** → Fetch Google Photos, Fetch Trip Notes (fan-out).
- **Edge cases:**
  - Throws on missing fields or invalid dates — workflow execution halts unless error handling is added.
  - `daysAfterTripEnd` is calculated at validation time and not rechecked later.

---

### Block 2 — Photo & Notes Collection

**Overview:** Two parallel branches retrieve photos (Google Photos API) and trip notes (Google Sheets). Each branch processes and enriches its data; the results merge into a unified day-by-day timeline ready for AI analysis.

**Nodes Involved:** Fetch Google Photos, Fetch Trip Notes, Extract Photo Metadata, Process Trip Notes, Merge Photos & Notes

#### Fetch Google Photos
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Role:** Calls the Google Photos Library API `mediaItems:search` endpoint.
- **Configuration:**
  - Method: POST, URL: `https://photoslibrary.googleapis.com/v1/mediaItems:search`
  - Authentication: `predefinedCredentialType` — `googlePhotosOAuth2Api`.
  - Body parameters:
    - `albumId` = `{{ $json.validatedTrip.photoSources.googlePhotosAlbumId }}`
    - `pageSize` = `100`
    - `filters` — a `dateFilter` with one range built from `validatedTrip.startDate` / `endDate` (year, month, day extracted via JS `new Date`).
  - Timeout: 30 000 ms.
  - `continueOnFail` = true.
- **Key expressions:** The `filters` field is built with `JSON.stringify` from an inline object whose `startDate`/`endDate` use `new Date(...).getFullYear()`, `.getMonth()+1`, `.getDate()`.
- **Input:** Validate Trip Data.
- **Output:** → Extract Photo Metadata.
- **Edge cases:**
  - If `googlePhotosAlbumId` is null/empty, the API may return an error or no items.
  - Google Photos API has a 10 000 items/day quota; large trips may need pagination (not implemented here — `pageSize` is fixed at 100).
  - `continueOnFail` means an API error will still pass downstream as an error item, not halt the workflow.

#### Fetch Trip Notes
- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Role:** Reads all rows from the `trip_notes` tab of the configured Google Sheet.
- **Configuration:**
  - Operation: Read (default).
  - Sheet name: `trip_notes`.
  - Document ID: `YOUR_GOOGLE_SHEET_ID` (placeholder — must be replaced).
  - Credential: `googleSheetsOAuth2Api`.
  - `continueOnFail` = true.
- **Input:** Validate Trip Data.
- **Output:** → Process Trip Notes.
- **Edge cases:**
  - If the sheet is empty or the document ID placeholder has not been replaced, the node returns an empty dataset.
  - The node reads the entire tab; filtering for the specific trip is done downstream.

#### Extract Photo Metadata
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Parses the Google Photos response, extracts EXIF and structural metadata, sorts photos chronologically, computes aggregate statistics.
- **Configuration / logic:**
  - Retrieves `validatedTrip` from the Validate Trip Data node by reference (`$('Validate Trip Data').item.json.validatedTrip`).
  - Extracts `mediaItems` array from the HTTP response.
  - For each photo: `photoId`, `filename`, `mimeType`, `baseUrl`, `productUrl`, `creationTime` (ISO), `width`, `height`, `location`, camera info (make, model, focal length, aperture, ISO), `downloadUrl` (`baseUrl=w2048-h2048`), `thumbnailUrl` (`baseUrl=w400-h400`).
  - Sorts by `creationTime`.
  - Computes stats: `totalPhotos`, `photosWithLocation`, `locationCoverage` (%), `uniqueDays`, `avgPhotosPerDay`, `dateRange`.
  - If zero photos, returns an object with `error: 'No photos found for this trip'`.
- **Output structure:** `{ validatedTrip, photos, photoStats }`.
- **Input:** Fetch Google Photos.
- **Output:** → Merge Photos & Notes.
- **Edge cases:**
  - Zero-photo trips produce an error flag but don't halt; downstream AI will receive an empty photo list.

#### Process Trip Notes
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Filters raw sheet rows to the current trip, organises notes by day, extracts tags/places/highlights.
- **Configuration / logic:**
  - Retrieves `validatedTrip` from Validate Trip Data node.
  - Filters rows where `note.tripId === validatedTrip.tripId` OR matching `userId` + date range.
  - Sorts by `date`.
  - Groups into `notesByDay` keyed by `YYYY-MM-DD`; each entry has `time`, `content`, `location`, `tags` (comma-split), `mood`, `highlights`.
  - Extracts `allTags`, `mentionedPlaces`, `highlightNotes`.
- **Output structure:** `{ validatedTrip, tripNotes: { byDay, all, stats } }`.
- **Input:** Fetch Trip Notes.
- **Output:** → Merge Photos & Notes.
- **Edge cases:**
  - No matching notes produce an empty `byDay` object — downstream AI works with notes-only or photos-only.

#### Merge Photos & Notes
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Combines photo and note data into a unified day-by-day timeline and assembles the `aiAnalysisInput` payload.
- **Configuration / logic:**
  - Reads from Extract Photo Metadata (`$('Extract Photo Metadata').item.json`) and Process Trip Notes (`$('Process Trip Notes').item.json`).
  - Groups photos by day (`photosByDay`).
  - Creates a sorted `timeline` array of days, each containing `date`, `dayNumber`, `photos`, `photoCount`, `notes`, `noteCount`, `locations`.
  - Builds `aiAnalysisInput = { validatedTrip, timeline, statistics, rawPhotos, rawNotes }`.
- **Input:** Extract Photo Metadata, Process Trip Notes (both feed into this single Code node; the node reads them by reference, not via direct main input — only one main input triggers execution, the other is read by expression).
- **Output:** → Analyze Photos with Claude Vision, Generate Travel Journal Content, Generate Platform Review Drafts (fan-out, three parallel branches).
- **Edge cases:**
  - If the node is triggered only by one branch arriving first, the other may not yet have populated its data. In practice both branches complete before Merge because the upstream branches are short, but in high-latency scenarios stale data could be read.

---

### Block 3 — AI Photo Analysis & Content Generation

**Overview:** Three independent LangChain Agent nodes, each powered by a dedicated Claude Sonnet 4 model node, run in parallel. They produce: (a) a structured photo/visual analysis, (b) a full travel journal, and (c) platform-optimised review drafts. All three outputs converge into Parse All AI Outputs.

**Nodes Involved:** Analyze Photos with Claude Vision, Claude Sonnet 4 with Vision, Generate Travel Journal Content, Claude Sonnet 4 with Vision1, Generate Platform Review Drafts, Claude Sonnet 4 with Vision2

#### Analyze Photos with Claude Vision
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.6, promptType = `define`)
- **Role:** Primary photo analysis agent. Receives the merged timeline data and returns a JSON object with trip analysis, photo categories, highlight selections, day-by-day highlights, missing elements, and quality scores.
- **Configuration:**
  - Prompt: Includes trip info (destination, duration, type, interests), content overview (photo/note counts), and a summarised timeline (one line per day).
  - Task: analyse themes, identify significant moments, detect mood/travel style, categorise photos, pick highlights, create narrative arc, note gaps.
  - System message: *"You are an expert photo analyst and travel storyteller. Respond with valid JSON only — no markdown, no code blocks, no preamble."*
  - Connected LLM: **Claude Sonnet 4 with Vision** (ai_languageModel sub-node).
- **Key expressions in prompt:** `{{ $json.validatedTrip.destination }}`, `{{ $json.validatedTrip.durationDays }}`, `{{ $json.timeline.map(...) }}`, etc.
- **Input:** Merge Photos & Notes.
- **Output:** → Parse All AI Outputs.
- **Edge cases:**
  - Claude may include markdown fences despite the system message; the Parse node strips them.
  - If the JSON is malformed or incomplete, `JSON.parse` in Parse All AI Outputs will throw.

#### Claude Sonnet 4 with Vision
- **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1)
- **Role:** LLM sub-node providing the model for the photo analysis agent.
- **Configuration:** Model = `claude-sonnet-4-20250514`, temperature = 0.4.
- **Credential:** `anthropicApi` (Anthropic account).
- **Connection:** `ai_languageModel` output → Analyze Photos with Claude Vision.

#### Generate Travel Journal Content
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.6, promptType = `define`)
- **Role:** Journal-writing agent. Uses trip details and analysis context to create a structured journal with opening reflection, daily narratives, highlight moments, closing reflection, travel tips, and favorite memories.
- **Configuration:**
  - Prompt references: destination, duration, journal style from `$('Merge Photos & Notes').item.json.validatedTrip`.
  - System message: *"You are a professional travel writer. Create engaging, vivid, and personal journal content. Respond with valid JSON only."*
  - Connected LLM: **Claude Sonnet 4 with Vision1**.
- **Input:** Merge Photos & Notes.
- **Output:** → Parse All AI Outputs.
- **Edge cases:** Same JSON-parsing risks as the other agents.

#### Claude Sonnet 4 with Vision1
- **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1)
- **Role:** LLM sub-node for the journal agent.
- **Configuration:** Model = `claude-sonnet-4-20250514`, temperature = 0.4.
- **Credential:** `anthropicApi`.
- **Connection:** `ai_languageModel` output → Generate Travel Journal Content.

#### Generate Platform Review Drafts
- **Type:** `@n8n/n8n-nodes-langchain.agent` (v1.6, promptType = `define`)
- **Role:** Review-drafting agent. Produces review content tailored to each platform listed in `outputPreferences.reviewPlatforms` (TripAdvisor, Google, Instagram, Blog).
- **Configuration:**
  - Prompt includes trip details and platform list via `$('Merge Photos & Notes')`.
  - Guidelines per platform: TripAdvisor (detailed, pros/cons), Google (concise, 150-300 words), Instagram (visual storytelling, hashtags, 100-150 words), Blog (long-form, SEO).
  - System message: *"You are an expert travel reviewer. Create authentic, platform-appropriate review content. Respond with valid JSON only."*
  - Connected LLM: **Claude Sonnet 4 with Vision2**.
- **Input:** Merge Photos & Notes.
- **Output:** → Parse All AI Outputs.
- **Edge cases:** If `reviewPlatforms` contains an unexpected platform name, Claude will still attempt to generate content but the structure may not match downstream expectations.

#### Claude Sonnet 4 with Vision2
- **Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1)
- **Role:** LLM sub-node for the review agent.
- **Configuration:** Model = `claude-sonnet-4-20250514`, temperature = 0.4.
- **Credential:** `anthropicApi`.
- **Connection:** `ai_languageModel` output → Generate Platform Review Drafts.

---

### Block 4 — Journal, Reviews & Delivery

**Overview:** Parses all three AI outputs into a single `memoryPackage` object, then fans out to four parallel output generators (PDF, highlights, Drive, Sheets), converges into an email, builds an API response, inserts a Wait node, and finally sends the webhook response.

**Nodes Involved:** Parse All AI Outputs, Create Journal PDF Document, Create Highlights Package, Save to Google Drive, Store Metadata in Sheets, Send Email with Memory Package, Build API Response, Wait For While, Send Response

#### Parse All AI Outputs
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Aggregates the three AI agent responses into one unified `memoryPackage`.
- **Configuration / logic:**
  - Helper `parseAIResponse`: extracts text from `response.output.text` or `content[0].text`, strips ````json` / ```` ``` fences, then `JSON.parse`.
  - Reads `photoAnalysisRaw` from Analyze Photos with Claude Vision, `journalRaw` from Generate Travel Journal Content, `reviewsRaw` from Generate Platform Review Drafts.
  - Reads `mergedData` from Merge Photos & Notes.
  - Assembles `memoryPackage` with: trip metadata, `photoAnalysis`, `journal`, `reviews`, `photos`, `notes`, `timeline`, `statistics` (augmented with highlight count and quality score), `generatedAt`, `processingId`, `aiModel`, `status = 'CONTENT_GENERATED'`.
- **Input:** Analyze Photos with Claude Vision, Generate Travel Journal Content, Generate Platform Review Drafts (three main-input connections — all must complete for this node to execute).
- **Output:** → Create Journal PDF Document, Create Highlights Package, Save to Google Drive, Store Metadata in Sheets.
- **Edge cases:**
  - If any of the three AI nodes returned an error or non-parseable text, `JSON.parse` will throw and halt the workflow.
  - `continueOnFail` is not set on this node, so failures are fatal.

#### Create Journal PDF Document
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Role:** Calls an external PDF generation API to render the journal.
- **Configuration:**
  - Method: POST, URL: `https://api.example.com/generate-pdf` (placeholder).
  - Body: `template = "travel-journal"`, `content` = stringified journal JSON, `photos` = first 30 photos stringified, `coverPhoto` = first photo's `downloadUrl`.
  - Timeout: 60 000 ms.
  - `continueOnFail` = true.
- **Input:** Parse All AI Outputs.
- **Output:** → Send Email with Memory Package.
- **Edge cases:**
  - The URL is a placeholder; must be replaced with a real PDF microservice.
  - No authentication is configured — add API key header if the service requires it.
  - If the service is unreachable or returns an error, `continueOnFail` allows the flow to continue but the email will lack a PDF attachment.

#### Create Highlights Package
- **Type:** `n8n-nodes-base.code` (v2, mode = `runOnceForEachItem`)
- **Role:** Builds a structured highlight reel from the AI-selected photos, including slide metadata, soundtrack mood suggestions, and export-format specs.
- **Configuration / logic:**
  - Reads `highlightPhotos` from `memoryPackage.photoAnalysis`.
  - Sorts by `qualityScore` descending, takes top `maxHighlights` (from preferences, default 10).
  - Each slide: `slideNumber`, photo URL/thumbnail/filename, caption, title, description, category, mood, date, location.
  - `soundtrack.mood` derived from `tripAnalysis.overallMood`.
  - `exportFormats` for Instagram Story, Instagram Post, YouTube Shorts, Slideshow Video.
  - Statistics: total highlights, average quality, categories/moods represented.
- **Input:** Parse All AI Outputs.
- **Output:** `{ memoryPackage, highlightReel }` → Send Email with Memory Package.
- **Edge cases:**
  - If `highlightPhotos` is empty (e.g., zero-photo trip), `topHighlights` will be empty; the reel will have zero slides.

#### Save to Google Drive
- **Type:** `n8n-nodes-base.googleDrive` (v3)
- **Role:** Saves the `memoryPackage` as a JSON file to Google Drive.
- **Configuration:**
  - Operation: Upload / Create file (default).
  - File name: `={{ $json.memoryPackage.tripId }}-journal.json`.
  - Drive: "My Drive", folder: root (`/ (Root folder)` — should be changed to the organised folder structure described in the documentation).
  - Credential: `googleDriveOAuth2Api`.
  - `continueOnFail` = true.
- **Input:** Parse All AI Outputs.
- **Output:** → Send Email with Memory Package.
- **Edge cases:**
  - Saving to root is a default placeholder; in production, set folderId to `/Travel Memories/{Year}/{Trip Name}/Journal`.
  - OAuth scope must include `https://www.googleapis.com/auth/drive.file`.

#### Store Metadata in Sheets
- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Role:** Appends a row with generated-content metadata to the `generated_content` tab.
- **Configuration:**
  - Operation: `append`.
  - Column mapping: `autoMapInputData`.
  - Sheet: `generated_content`, document: `YOUR_GOOGLE_SHEET_ID`.
  - Credential: `googleSheetsOAuth2Api`.
  - `continueOnFail` = true.
- **Input:** Parse All AI Outputs.
- **Output:** → Send Email with Memory Package.
- **Edge cases:**
  - If column names in the sheet don't match the JSON keys, `autoMapInputData` may silently drop fields.

#### Send Email with Memory Package
- **Type:** `n8n-nodes-base.emailSend` (v2.1)
- **Role:** Emails the user their memory package with journal PDF and highlights attached.
- **Configuration:**
  - To: `={{ $json.memoryPackage.userEmail }}`.
  - From: `user@example.com` (placeholder).
  - Subject: `=✨ Your {{ $json.memoryPackage.destination }} Travel Memories Are Ready!`.
  - Attachments option references: `journal-document, highlights-package` (these names are symbolic; actual attachment resolution depends on prior node binary data — currently neither the PDF request node nor the highlights Code node produce binary attachments, so this will likely send an email without real attachments unless the PDF service returns binary data).
  - Credential: `smtp`.
- **Input:** Create Journal PDF Document, Create Highlights Package, Save to Google Drive, Store Metadata in Sheets (four inputs — all converge; node fires once all are done).
- **Output:** → Build API Response.
- **Edge cases:**
  - SMTP credential must be configured; if from-address is not authorised, the mail server may reject.
  - The `attachments` field uses symbolic names that require binary data from preceding nodes — this needs verification.

#### Build API Response
- **Type:** `n8n-nodes-base.code` (v2, mode = `runOnceForEachItem`)
- **Role:** Constructs a clean JSON API response summarising the generated content.
- **Configuration / logic:**
  - Reads `memoryPackage` and `highlightReel` from the input item.
  - Builds response with: `success`, `processingId`, `tripId`, `destination`, `generatedAt`, `summary` (photo counts, journal pages, reviews, quality score), `topHighlight`, `deliverables`, `journalPreview` (title, opening line, favourite memories), and a user-friendly `message`.
- **Input:** Send Email with Memory Package.
- **Output:** → Wait For While.
- **Edge cases:** None significant.

#### Wait For While
- **Type:** `n8n-nodes-base.wait` (v1.1)
- **Role:** Pauses execution (production mode) to allow asynchronous operations (e.g., email delivery, Drive propagation) to settle before responding to the webhook caller.
- **Configuration:** No parameters (default behaviour — waits for manual resume or webhook callback; in a production execution it will pause until the workflow is resumed).
- **Input:** Build API Response.
- **Output:** → Send Response.
- **Edge cases:**
  - In test/debug mode the Wait node executes immediately; in production it pauses indefinitely until resumed. This may cause the webhook to time out if not resumed promptly.

#### Send Response
- **Type:** `n8n-nodes-base.respondToWebhook` (v1)
- **Role:** Sends the JSON response back to the original webhook caller.
- **Configuration:**
  - Respond with: `json`.
  - Response body: `={{ JSON.stringify($json, null, 2) }}`.
  - Response header: `Content-Type: application/json`.
- **Input:** Wait For While.
- **Output:** None (end of flow).
- **Edge cases:**
  - If the webhook caller has already disconnected (timeout), the response is lost silently.

---

### Block 5 — Daily Batch Processing & Analytics

**Overview:** A cron-triggered path that runs daily at 06:00, scans the trips sheet for recently ended trips that haven't been processed, creates processing-job payloads, and logs analytics.

**Nodes Involved:** Daily Batch Processing, Find Recently Completed Trips, Filter Trips Needing Processing, Log Batch Processing Analytics

#### Daily Batch Processing
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)
- **Role:** Cron-based trigger.
- **Configuration:** Cron expression `0 6 * * *` (every day at 06:00 server time).
- **Output:** → Find Recently Completed Trips.

#### Find Recently Completed Trips
- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Role:** Reads all rows from the `trips` tab.
- **Configuration:** Sheet name = `trips`, document = `YOUR_GOOGLE_SHEET_ID`, credential = `googleSheetsOAuth2Api`.
- **Output:** → Filter Trips Needing Processing.

#### Filter Trips Needing Processing
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Filters trips to those ended within the last 7 days, not yet processed (`memoryPackageGenerated !== true`), and having photos (`hasPhotos = true`, `photoCount > 0`). Converts qualifying trips into processing-job payloads that mirror the webhook body format.
- **Configuration / logic:**
  - Returns an array of items, each with `json.body` containing `tripId`, `userId`, `userName`, `userEmail`, `tripDetails`, `photoSources`, `notesLocation`, `outputPreferences`, `deliveryOptions`.
  - Default output preferences: narrative style, TripAdvisor + Google + Instagram reviews, 10 highlights.
- **Output:** → Log Batch Processing Analytics.
- **Design note:** The processing jobs are not connected back into the main flow. To actually process them, a Sub-Workflow call or an HTTP call back to the webhook would be needed. Currently this block only logs what *would* be processed.

#### Log Batch Processing Analytics
- **Type:** `n8n-nodes-base.googleSheets` (v4.5)
- **Role:** Appends the filtered job data to the `analytics` tab.
- **Configuration:** Operation = `append`, auto-map columns, sheet = `analytics`, document = `YOUR_GOOGLE_SHEET_ID`, credential = `googleSheetsOAuth2Api`.
- **Output:** None (end of batch flow).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|-----------------|-------------|
| Main Documentation | stickyNote | Workflow overview and setup instructions | — | — | ## Post-Trip Memory & Review Builder with Claude AI — Automatically transforms your travel photos and notes into beautiful journals, highlight reels, and review drafts using Claude's vision and language capabilities. Includes setup steps: import workflow, configure credentials (Anthropic API, Google Photos API, Google Drive API, Google Sheets, SMTP/Gmail, OpenWeatherMap optional), create Google Sheets with tabs (trips, trip_notes, generated_content, analytics), set up Google Drive folder structure (/Travel Memories/{Year}/{Trip Name}/Photos, Journal, Reviews), configure API keys and resource IDs, activate both webhook and scheduled workflows. |
| Section 1: Trip Intake | stickyNote | Visual grouping | — | — | ## 1. Trip Intake & Validation |
| Section 2: Data Collection | stickyNote | Visual grouping | — | — | ## 2. Photo & Notes Collection |
| Section 3: AI Analysis | stickyNote | Visual grouping | — | — | ## 3. AI Photo Analysis & Content Generation |
| Section 4: Output Creation | stickyNote | Visual grouping | — | — | ## 4. Journal, Reviews & Delivery |
| Section 5: Batch Processing | stickyNote | Visual grouping | — | — | ## 5. Daily Batch Processing & Analytics |
| Trip Completion Webhook | webhook | HTTP entry point (POST, responseNode mode) | External client | Validate Trip Data | ## 1. Trip Intake & Validation |
| Validate Trip Data | code | Validates & normalises trip payload | Trip Completion Webhook | Fetch Google Photos, Fetch Trip Notes | ## 1. Trip Intake & Validation |
| Fetch Google Photos | httpRequest | Retrieves photos from Google Photos API | Validate Trip Data | Extract Photo Metadata | ## 2. Photo & Notes Collection |
| Fetch Trip Notes | googleSheets | Reads trip_notes sheet tab | Validate Trip Data | Process Trip Notes | ## 2. Photo & Notes Collection |
| Extract Photo Metadata | code | Parses photo items, extracts EXIF, sorts chronologically | Fetch Google Photos | Merge Photos & Notes | ## 2. Photo & Notes Collection |
| Process Trip Notes | code | Filters & organises notes by day | Fetch Trip Notes | Merge Photos & Notes | ## 2. Photo & Notes Collection |
| Merge Photos & Notes | code | Builds unified day-by-day timeline | Extract Photo Metadata, Process Trip Notes | Analyze Photos with Claude Vision, Generate Travel Journal Content, Generate Platform Review Drafts | ## 2. Photo & Notes Collection |
| Analyze Photos with Claude Vision | @n8n/n8n-nodes-langchain.agent | AI photo analysis & categorisation agent | Merge Photos & Notes | Parse All AI Outputs | ## 3. AI Photo Analysis & Content Generation |
| Claude Sonnet 4 with Vision | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM model sub-node for photo analysis | (connected as ai_languageModel to) Analyze Photos with Claude Vision | (ai_languageModel to) Analyze Photos with Claude Vision | ## 3. AI Photo Analysis & Content Generation |
| Generate Travel Journal Content | @n8n/n8n-nodes-langchain.agent | AI journal-writing agent | Merge Photos & Notes | Parse All AI Outputs | ## 3. AI Photo Analysis & Content Generation |
| Claude Sonnet 4 with Vision1 | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM model sub-node for journal agent | (connected as ai_languageModel to) Generate Travel Journal Content | (ai_languageModel to) Generate Travel Journal Content | ## 3. AI Photo Analysis & Content Generation |
| Generate Platform Review Drafts | @n8n/n8n-nodes-langchain.agent | AI review-drafting agent | Merge Photos & Notes | Parse All AI Outputs | ## 3. AI Photo Analysis & Content Generation |
| Claude Sonnet 4 with Vision2 | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM model sub-node for review agent | (connected as ai_languageModel to) Generate Platform Review Drafts | (ai_languageModel to) Generate Platform Review Drafts | ## 3. AI Photo Analysis & Content Generation |
| Parse All AI Outputs | code | Merges three AI outputs into a memoryPackage | Analyze Photos with Claude Vision, Generate Travel Journal Content, Generate Platform Review Drafts | Create Journal PDF Document, Create Highlights Package, Save to Google Drive, Store Metadata in Sheets | ## 4. Journal, Reviews & Delivery |
| Create Journal PDF Document | httpRequest | Calls external PDF generation API | Parse All AI Outputs | Send Email with Memory Package | ## 4. Journal, Reviews & Delivery |
| Create Highlights Package | code | Builds highlight reel structure | Parse All AI Outputs | Send Email with Memory Package | ## 4. Journal, Reviews & Delivery |
| Save to Google Drive | googleDrive | Saves memoryPackage JSON to Drive | Parse All AI Outputs | Send Email with Memory Package | ## 4. Journal, Reviews & Delivery |
| Store Metadata in Sheets | googleSheets | Appends generated-content metadata row | Parse All AI Outputs | Send Email with Memory Package | ## 4. Journal, Reviews & Delivery |
| Send Email with Memory Package | emailSend | Emails journal PDF & highlights to user | Create Journal PDF Document, Create Highlights Package, Save to Google Drive, Store Metadata in Sheets | Build API Response | ## 4. Journal, Reviews & Delivery |
| Build API Response | code | Constructs JSON webhook response | Send Email with Memory Package | Wait For While | ## 4. Journal, Reviews & Delivery |
| Wait For While | wait | Pauses execution before responding | Build API Response | Send Response | ## 4. Journal, Reviews & Delivery |
| Send Response | respondToWebhook | Returns JSON response to webhook caller | Wait For While | — | ## 4. Journal, Reviews & Delivery |
| Daily Batch Processing | scheduleTrigger | Cron trigger at 06:00 daily | — | Find Recently Completed Trips | ## 5. Daily Batch Processing & Analytics |
| Find Recently Completed Trips | googleSheets | Reads trips sheet tab | Daily Batch Processing | Filter Trips Needing Processing | ## 5. Daily Batch Processing & Analytics |
| Filter Trips Needing Processing | code | Filters trips ended in last 7 days, not yet processed | Find Recently Completed Trips | Log Batch Processing Analytics | ## 5. Daily Batch Processing & Analytics |
| Log Batch Processing Analytics | googleSheets | Appends processing jobs to analytics tab | Filter Trips Needing Processing | — | ## 5. Daily Batch Processing & Analytics |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
- n8n instance (cloud or self-hosted) with the LangChain nodes package installed.
- Anthropic API key with access to `claude-sonnet-4-20250514`.
- Google Cloud project with OAuth2 credentials for: Google Photos Library API, Google Drive API, Google Sheets API.
- SMTP credentials (or Gmail OAuth2) for outbound email.
- A PDF generation microservice endpoint (replace `https://api.example.com/generate-pdf`).
- A Google Sheet with four tabs: `trips`, `trip_notes`, `generated_content`, `analytics`.
- A Google Drive folder structure: `/Travel Memories/{Year}/{Trip Name}/Photos`, `…/Journal`, `…/Reviews`.

---

### Step-by-step rebuild

1. **Create the workflow** and name it *Post-Trip Memory & Review Builder with Claude AI Vision*.

2. **Add Main Documentation sticky note.** Place it on the canvas, set width = 1072, height = 1184. Paste the full documentation content (setup steps, how it works, folder structure).

3. **Add five section sticky notes:**
   - "Section 1: Trip Intake" — color 4, width 480, height 400.
   - "Section 2: Data Collection" — color 4, width 960, height 800.
   - "Section 3: AI Analysis" — color 4, width 1120, height 880.
   - "Section 4: Output Creation" — color 4, width 1280, height 960.
   - "Section 5: Batch Processing" — color 4, width 1520, height 420.

4. **Add Trip Completion Webhook:**
   - Type: Webhook, version 2.
   - HTTP Method: POST.
   - Path: `trip-completed`.
   - Response Mode: `responseNode`.

5. **Add Validate Trip Data:**
   - Type: Code, version 2.
   - Mode: `runOnceForEachItem`.
   - Paste the full JavaScript that validates required fields, date logic, normalises photoSources and outputPreferences, generates a `processingId`.
   - Connect: Trip Completion Webhook → Validate Trip Data.

6. **Add Fetch Google Photos:**
   - Type: HTTP Request, version 4.2.
   - Method: POST, URL: `https://photoslibrary.googleapis.com/v1/mediaItems:search`.
   - Authentication: Predefined credential type → `googlePhotosOAuth2Api`.
   - Send Body: enabled. Add parameters:
     - `albumId` = `={{ $json.validatedTrip.photoSources.googlePhotosAlbumId }}`
     - `pageSize` = `100`
     - `filters` = `={{ JSON.stringify({ dateFilter: { ranges: [{ startDate: { year: new Date($json.validatedTrip.startDate).getFullYear(), month: new Date($json.validatedTrip.startDate).getMonth() + 1, day: new Date($json.validatedTrip.startDate).getDate() }, endDate: { year: new Date($json.validatedTrip.endDate).getFullYear(), month: new Date($json.validatedTrip.endDate).getMonth() + 1, day: new Date($json.validatedTrip.endDate).getDate() } }] } }) }}`
   - Timeout: 30000 ms.
   - Continue on Fail: enabled.
   - Connect: Validate Trip Data → Fetch Google Photos.

7. **Add Fetch Trip Notes:**
   - Type: Google Sheets, version 4.5.
   - Operation: Read (default).
   - Sheet Name: `trip_notes`.
   - Document ID: select your Google Sheet.
   - Credential: Google Sheets OAuth2.
   - Continue on Fail: enabled.
   - Connect: Validate Trip Data → Fetch Trip Notes.

8. **Add Extract Photo Metadata:**
   - Type: Code, version 2.
   - Paste the JavaScript that extracts `mediaItems`, builds photo objects with EXIF data and download/thumbnail URLs, sorts chronologically, computes `photoStats`.
   - Connect: Fetch Google Photos → Extract Photo Metadata.

9. **Add Process Trip Notes:**
   - Type: Code, version 2.
   - Paste the JavaScript that filters rows by trip ID / user + date range, groups by day, extracts tags/places/highlights.
   - Connect: Fetch Trip Notes → Process Trip Notes.

10. **Add Merge Photos & Notes:**
    - Type: Code, version 2.
    - Paste the JavaScript that reads both branches by node reference, groups photos by day, merges into a unified `timeline`, outputs `aiAnalysisInput`.
    - Connect: Extract Photo Metadata → Merge Photos & Notes; Process Trip Notes → Merge Photos & Notes.

11. **Add Claude Sonnet 4 with Vision (model node for photo analysis):**
    - Type: `@n8n/n8n-nodes-langchain.lmChatAnthropic`, version 1.
    - Model: `claude-sonnet-4-20250514`.
    - Temperature: 0.4.
    - Credential: Anthropic API.

12. **Add Analyze Photos with Claude Vision:**
    - Type: `@n8n/n8n-nodes-langchain.agent`, version 1.6.
    - Prompt Type: `define`.
    - Paste the full prompt text (trip info, content overview, timeline summary, task list, response JSON schema).
    - System Message: *"You are an expert photo analyst and travel storyteller. Respond with valid JSON only — no markdown, no code blocks, no preamble."*
    - Connect: Claude Sonnet 4 with Vision → (ai_languageModel) Analyze Photos with Claude Vision.
    - Connect: Merge Photos & Notes → (main) Analyze Photos with Claude Vision.

13. **Add Claude Sonnet 4 with Vision1 (model node for journal):**
    - Same type and settings as step 11, same model and temperature.

14. **Add Generate Travel Journal Content:**
    - Type: `@n8n/n8n-nodes-langchain.agent`, version 1.6.
    - Prompt Type: `define`.
    - Paste the full journal-writing prompt (trip details, create journal sections, response JSON schema).
    - System Message: *"You are a professional travel writer. Create engaging, vivid, and personal journal content. Respond with valid JSON only."*
    - Connect: Claude Sonnet 4 with Vision1 → (ai_languageModel) Generate Travel Journal Content.
    - Connect: Merge Photos & Notes → (main) Generate Travel Journal Content.

15. **Add Claude Sonnet 4 with Vision2 (model node for reviews):**
    - Same type and settings as step 11.

16. **Add Generate Platform Review Drafts:**
    - Type: `@n8n/n8n-nodes-langchain.agent`, version 1.6.
    - Prompt Type: `define`.
    - Paste the full review-drafting prompt (trip details, target platforms, platform guidelines, response JSON schema).
    - System Message: *"You are an expert travel reviewer. Create authentic, platform-appropriate review content. Respond with valid JSON only."*
    - Connect: Claude Sonnet 4 with Vision2 → (ai_languageModel) Generate Platform Review Drafts.
    - Connect: Merge Photos & Notes → (main) Generate Platform Review Drafts.

17. **Add Parse All AI Outputs:**
    - Type: Code, version 2.
    - Paste the JavaScript with `parseAIResponse` helper, reads all three AI node outputs and merged data, assembles `memoryPackage`.
    - Connect: Analyze Photos with Claude Vision → Parse All AI Outputs; Generate Travel Journal Content → Parse All AI Outputs; Generate Platform Review Drafts → Parse All AI Outputs.

18. **Add Create Journal PDF Document:**
    - Type: HTTP Request, version 4.2.
    - Method: POST, URL: `https://api.example.com/generate-pdf` (replace with your PDF service).
    - Send Body: enabled. Parameters: `template = travel-journal`, `content = {{ JSON.stringify($json.memoryPackage.journal) }}`, `photos = {{ JSON.stringify($json.memoryPackage.photos.filter((p, i) => i < 30)) }}`, `coverPhoto = {{ $json.memoryPackage.photos[0]?.downloadUrl }}`.
    - Timeout: 60000 ms.
    - Continue on Fail: enabled.
    - Connect: Parse All AI Outputs → Create Journal PDF Document.

19. **Add Create Highlights Package:**
    - Type: Code, version 2, mode: `runOnceForEachItem`.
    - Paste the JavaScript that builds a highlight reel from `photoAnalysis.highlightPhotos`, sorted by quality, with slide metadata, soundtrack, and export format specs.
    - Connect: Parse All AI Outputs → Create Highlights Package.

20. **Add Save to Google Drive:**
    - Type: Google Drive, version 3.
    - File Name: `={{ $json.memoryPackage.tripId }}-journal.json`.
    - Drive: My Drive. Folder: root (change to your target folder).
    - Credential: Google Drive OAuth2.
    - Continue on Fail: enabled.
    - Connect: Parse All AI Outputs → Save to Google Drive.

21. **Add Store Metadata in Sheets:**
    - Type: Google Sheets, version 4.5.
    - Operation: Append.
    - Sheet: `generated_content`, Document: your Google Sheet.
    - Column Mapping: autoMapInputData.
    - Credential: Google Sheets OAuth2.
    - Continue on Fail: enabled.
    - Connect: Parse All AI Outputs → Store Metadata in Sheets.

22. **Add Send Email with Memory Package:**
    - Type: Email Send, version 2.1.
    - To: `={{ $json.memoryPackage.userEmail }}`.
    - From: `user@example.com` (replace).
    - Subject: `=✨ Your {{ $json.memoryPackage.destination }} Travel Memories Are Ready!`.
    - Options → Attachments: `journal-document, highlights-package`.
    - Credential: SMTP.
    - Connect: Create Journal PDF Document → Send Email; Create Highlights Package → Send Email; Save to Google Drive → Send Email; Store Metadata in Sheets → Send Email.

23. **Add Build API Response:**
    - Type: Code, version 2, mode: `runOnceForEachItem`.
    - Paste the JavaScript that reads `memoryPackage` and `highlightReel` and builds a summary response object.
    - Connect: Send Email with Memory Package → Build API Response.

24. **Add Wait For While:**
    - Type: Wait, version 1.1.
    - No parameters (default).
    - Connect: Build API Response → Wait For While.

25. **Add Send Response:**
    - Type: Respond to Webhook, version 1.
    - Respond With: JSON.
    - Response Body: `={{ JSON.stringify($json, null, 2) }}`.
    - Response Headers: `Content-Type: application/json`.
    - Connect: Wait For While → Send Response.

26. **Add Daily Batch Processing:**
    - Type: Schedule Trigger, version 1.2.
    - Cron expression: `0 6 * * *`.
    - Connect: Daily Batch Processing → Find Recently Completed Trips.

27. **Add Find Recently Completed Trips:**
    - Type: Google Sheets, version 4.5.
    - Sheet: `trips`, Document: your Google Sheet.
    - Credential: Google Sheets OAuth2.
    - Connect: Find Recently Completed Trips → Filter Trips Needing Processing.

28. **Add Filter Trips Needing Processing:**
    - Type: Code, version 2.
    - Paste the JavaScript that filters trips ended within 7 days, not yet processed, with photos; outputs processing-job payloads mirroring the webhook body.
    - Connect: Filter Trips Needing Processing → Log Batch Processing Analytics.

29. **Add Log Batch Processing Analytics:**
    - Type: Google Sheets, version 4.5.
    - Operation: Append.
    - Sheet: `analytics`, Document: your Google Sheet.
    - Column Mapping: autoMapInputData.
    - Credential: Google Sheets OAuth2.
    - No downstream connection (end of batch branch).

30. **Configure all credentials in n8n:**
    - Anthropic API → named e.g. "Anthropic account".
    - Google Photos OAuth2 → with scope `https://www.googleapis.com/auth/photoslibrary.readonly`.
    - Google Sheets OAuth2 → with scope `https://www.googleapis.com/auth/spreadsheets`.
    - Google Drive OAuth2 → with scope `https://www.googleapis.com/auth/drive.file`.
    - SMTP → host, port, user, password, TLS settings.

31. **Replace all placeholder IDs:**
    - `YOUR_GOOGLE_SHEET_ID` in Fetch Trip Notes, Store Metadata in Sheets, Find Recently Completed Trips, Log Batch Processing Analytics, Filter Trips Needing Processing.
    - `https://api.example.com/generate-pdf` in Create Journal PDF Document.
    - `user@example.com` in Send Email with Memory Package.
    - Google Drive folder ID in Save to Google Drive (change from root to your target folder).

32. **Activate the workflow.** Both the webhook and the scheduled trigger will be active.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The batch processing branch creates processing-job payloads but does **not** loop them back into the main flow. To fully automate, add an "Execute Workflow" or "HTTP Request" node after Filter Trips Needing Processing that calls the webhook (or a Sub-Workflow node) for each job. | Design gap in the current workflow |
| Google Photos API has a 10 000 items/day quota and a default page size of 100. For trips with > 100 photos, implement pagination by looping on `nextPageToken`. | Google Photos API documentation |
| The PDF generation endpoint (`api.example.com`) is a placeholder. You must provide a real service (e.g., a custom n8n Code node using a library like `pdf-lib`, or an external API like DocRaptor, PDFShift, etc.). | Requires replacement before production use |
| The `attachments` field in Send Email references symbolic names `journal-document` and `highlights-package`. For these to resolve as real file attachments, the preceding nodes must output binary data (e.g., the HTTP Request node must return the PDF as binary). Verify this with your PDF service. | Email attachment mechanism |
| The Wait For While node will pause the execution in production until resumed. If the webhook caller has a short timeout (e.g., 30 s), the response may never be received. Consider removing this node or setting a short resume timeout. | Production timing consideration |
| All three Claude agents use temperature 0.4 for consistent, structured JSON output. Increasing temperature may yield more creative prose but risks invalid JSON. | Model configuration rationale |
| The `continueOnFail` flag on Fetch Google Photos, Fetch Trip Notes, Create Journal PDF Document, Save to Google Drive, and Store Metadata in Sheets means partial failures do not halt the workflow. Inspect execution logs for individual node errors. | Error handling strategy |
| Claude Sonnet 4 Vision model identifier `claude-sonnet-4-20250514` must be available on your Anthropic account. If using a different model, update all three model nodes and the `aiModel` field in Parse All AI Outputs. | Model availability |
| The Google Drive node saves the memory package as a JSON file, not as a rendered PDF. The PDF is generated by the separate HTTP Request node. The Drive file serves as a data archive. | Output file type distinction |