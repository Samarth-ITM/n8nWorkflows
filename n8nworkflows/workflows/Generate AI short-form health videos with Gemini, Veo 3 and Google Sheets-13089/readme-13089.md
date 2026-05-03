Generate AI short-form health videos with Gemini, Veo 3 and Google Sheets

https://n8nworkflows.xyz/workflows/generate-ai-short-form-health-videos-with-gemini--veo-3-and-google-sheets-13089


# Generate AI short-form health videos with Gemini, Veo 3 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Create Viral AI Health Videos*  
**Stated title:** *Generate AI short-form health videos with Gemini, Veo 3 and Google Sheets*

This workflow automates an end-to-end pipeline to:
1) collect viral health-related TikTok videos,  
2) store them as “ideas” in Google Sheets with a processing status,  
3) analyze a selected pending idea with Gemini (video analysis),  
4) generate a short-form script split into multiple Veo 3 scene prompts,  
5) render scenes via a Veo 3 API provider (GeminiGenAI), poll until done,  
6) concatenate rendered clips,  
7) publish the final video to Facebook using Blotato,  
8) update the Google Sheet status.

### 1.1 Viral Content Collection (scheduled ingestion)
Runs every 6 hours, scrapes a TikTok profile via Apify, normalizes key fields, appends the result to Google Sheets with `Status = Pending`.

### 1.2 Idea Analysis & Script Generation (scheduled processing)
Runs every 10 hours, pulls one `Pending` row from Sheets, uses Gemini to analyze the video, then uses a LangChain Agent to produce structured output: scenes (each with a Veo3 prompt) + metadata (caption, hashtags).

### 1.3 AI Video Rendering (Veo 3)
Splits scenes into individual jobs, loops through them, calls the Veo 3 generation endpoint, waits, checks status, and collects completed video URLs.

### 1.4 Video Post-processing (concatenate)
Aggregates scene outputs and calls an external “concatenate” endpoint to merge clips.

### 1.5 Publishing & Tracking
Publishes to Facebook via Blotato and marks the original Sheet row as `Done`.

---

## 2. Block-by-Block Analysis

### Block 1 — Viral Content Collection

**Overview:** Collects viral content from TikTok (via Apify actor), transforms fields into a consistent schema, and stores each item as a row in Google Sheets for later processing.

**Nodes involved:**
- Collect Viral Ideas
- Scrape Viral Health Content (Apify)
- Normalize Viral Content Data
- Save Viral Ideas to Google Sheets

#### Node: Collect Viral Ideas
- **Type / role:** Schedule Trigger; starts ingestion.
- **Configuration:** Runs every **6 hours**.
- **Inputs:** None (trigger).
- **Outputs:** To **Scrape Viral Health Content (Apify)**.
- **Edge cases / failures:** n8n schedule misfires (instance downtime), timezone expectations.

#### Node: Scrape Viral Health Content (Apify)
- **Type / role:** Apify node; runs actor and retrieves dataset items.
- **Configuration choices:**
  - **Actor:** “TikTok Scraper (clockworks/tiktok-scraper)” (actor id `GdWCkxBtKWOsKjdch`)
  - **Operation:** “Run actor and get dataset”
  - **Input parameters (key ones):**
    - `profiles`: `["@yeucotheminhh"]`
    - `resultsPerPage`: `20`
    - `profileSorting`: `popular`
    - `proxyCountryCode`: `VN`
    - `shouldDownloadVideos`: `true`
    - various “download X” flags disabled (covers, subtitles, etc.)
- **Credentials:** Apify OAuth2 (“Apify account 3”).
- **Outputs:** To **Normalize Viral Content Data**.
- **Edge cases / failures:**
  - Actor quota limits, dataset size, TikTok anti-bot restrictions, proxy failures.
  - Items may have `mediaUrls` empty (some pinned/ads or missing download), which later breaks when indexing `[0]` in Sheets node.

#### Node: Normalize Viral Content Data
- **Type / role:** Set node; maps raw Apify fields to normalized columns.
- **Configuration choices (field mapping):**
  - `Caption` ← `$json.text`
  - `Date Post` ← `$json.createTimeISO.toDateTime().format('dd/LL/yyyy')`
  - `Username` ← `$json.authorMeta.name`
  - `Link user` ← `$json.authorMeta.profileUrl`
  - `Link video` ← `$json.webVideoUrl`
  - `VideoURL` (array) ← `$json.mediaUrls`
  - `Likes` ← `$json.diggCount`
  - `Shares` ← `$json.shareCount`
  - `View` ← `$json.playCount`
  - `Save` ← `$json.collectCount`
  - `Hasgtag` (array) ← `$json.hashtags`
- **Inputs:** From Apify results (one item per TikTok post).
- **Outputs:** To **Save Viral Ideas to Google Sheets**.
- **Edge cases / failures:**
  - If `createTimeISO` missing/invalid → DateTime conversion fails.
  - If `authorMeta` missing → expression errors.
  - `Hasgtag` stored as array but Sheet schema expects string (see below).

#### Node: Save Viral Ideas to Google Sheets
- **Type / role:** Google Sheets; appends a new row per scraped item.
- **Configuration choices:**
  - **Operation:** Append
  - **Document:** “Health viral content” (Google Sheet ID set in node)
  - **Sheet/tab:** “Trang tính1”
  - **Columns written:**
    - `Caption`, `Date Post`, `Username`, `Link user`, `Link video`
    - `VideoURL` ← `={{ $json.VideoURL[0] }}` (first media URL only)
    - `Likes`, `Shares`, `View`, `Save`
    - `Hasgtag` ← `={{ $json.Hasgtag }}`
    - `Status` hardcoded to `"Pending"`
- **Credentials:** Google Sheets OAuth2 (“GiangXAI”).
- **Inputs:** Normalized Set output.
- **Outputs:** End of this block.
- **Edge cases / failures:**
  - `VideoURL[0]` fails if `VideoURL` is empty array; you may need a fallback.
  - `Hasgtag` is an array; the sheet column type is “string”. It may become JSON text or fail depending on settings. Consider joining to a string (`map(h=>"#"+h.name).join(" ")`).

---

### Block 2 — Idea Analysis & Script Generation

**Overview:** Loads one pending idea from Google Sheets, analyzes its video with Gemini “video analyze”, then uses a LangChain agent to generate a structured multi-scene Veo prompt plan plus caption/hashtags.

**Nodes involved:**
- Process Viral Ideas
- Load Viral Ideas from Sheet
- Analyze Viral Potential & Hook
- Generate AI Video Script (Health Niche)
- Gemini LLM
- Reasoning / Prompt Logic
- Parse Script & Video Parameters

#### Node: Process Viral Ideas
- **Type / role:** Schedule Trigger; starts processing pipeline.
- **Configuration:** Runs every **10 hours**.
- **Outputs:** To **Load Viral Ideas from Sheet**.
- **Edge cases:** Overlap with ingestion; multiple runs may process same row if status isn’t updated early.

#### Node: Load Viral Ideas from Sheet
- **Type / role:** Google Sheets; fetches one row to process.
- **Configuration choices:**
  - **Filter:** `Status` equals `"Pending"`
  - **Option:** `returnFirstMatch: true` (only the first matching row)
- **Credentials:** Google Sheets OAuth2 (“GiangXAI”).
- **Outputs:** To **Analyze Viral Potential & Hook**.
- **Edge cases / failures:**
  - If no pending rows exist: node may output nothing (downstream nodes won’t run).
  - “First match” ordering depends on Sheets API return order; not guaranteed FIFO unless sheet is sorted.

#### Node: Analyze Viral Potential & Hook
- **Type / role:** LangChain Google Gemini node (resource: **video**, operation: **analyze**).
- **Configuration choices:**
  - **Model:** `models/gemini-3-pro-preview`
  - **Video URL:** `={{ $json.VideoURL }}`
  - **Analysis prompt text:** fixed string: “Detailed video description including detailed analysis of setting and characters (no description with watermark if any)”
- **Credentials:** Google PaLM/Gemini (“api google”).
- **Outputs:** To **Generate AI Video Script (Health Niche)** (provides Gemini analysis text in `content.parts[0].text`).
- **Edge cases / failures:**
  - Video URL must be reachable by Gemini; expired or blocked URLs will fail.
  - Large/long videos may exceed model or API constraints.

#### Node: Gemini LLM
- **Type / role:** LangChain chat LLM provider used by the Agent and Output Parser.
- **Configuration choices:**
  - **Model:** `models/gemini-3-pro-preview`
- **Connections:** Connected as `ai_languageModel` to:
  - Generate AI Video Script (Health Niche)
  - Parse Script & Video Parameters
- **Edge cases:** Model availability (“preview”), quotas, safety filters.

#### Node: Reasoning / Prompt Logic
- **Type / role:** LangChain “toolThink” (internal reasoning tool) available to the Agent.
- **Configuration:** Default.
- **Connections:** Provided as `ai_tool` to the Agent.
- **Edge cases:** If agent/tool calling disabled by environment policies, may reduce quality but usually won’t break flow.

#### Node: Parse Script & Video Parameters
- **Type / role:** Structured output parser; enforces JSON schema on agent output.
- **Configuration choices:**
  - **autoFix:** enabled (tries to repair invalid JSON)
  - **Schema example:**
    - `scenes[]`: `{ scene_number, character_name, veo3_prompt }`
    - `video_metadata`: `{ caption, hashtags }`
- **Connections:** Provided as `ai_outputParser` to the Agent.
- **Edge cases / failures:**
  - If the agent output is too malformed, autoFix may still fail.
  - Schema does not enforce minimum/maximum scenes; downstream expects `output.scenes` exists.

#### Node: Generate AI Video Script (Health Niche)
- **Type / role:** LangChain Agent; turns analysis text into scene prompts and metadata for Veo3.
- **Configuration choices:**
  - **Input text:** `={{ $json.content.parts[0].text }}`
  - **Prompt type:** “define”
  - **System message:** extensive constraints:
    - Stylized 3D animation, anthropomorphic food characters, Vietnamese spoken dialogue, no on-screen text/subtitles
    - Strict policy constraints (no minors, no real persons, no specific IP characters, etc.)
    - Requires prompts suitable for shorts/reels
  - **Output parsing:** enabled (uses “Parse Script & Video Parameters”).
- **Inputs:** Output of “Analyze Viral Potential & Hook”.
- **Outputs:** To **Split Ideas into Video Jobs**. Output shape: `output.scenes` and `output.video_metadata`.
- **Edge cases / failures:**
  - If Gemini analysis returns empty, agent may hallucinate; consider a guard/IF node.
  - Prompt is very strict; safety filters may refuse certain health content depending on details.

---

### Block 3 — AI Video Rendering (Veo 3)

**Overview:** Converts structured scenes into individual API calls to Veo 3 generation, waits for rendering, polls status until complete, and routes completion vs retry.

**Nodes involved:**
- Split Ideas into Video Jobs
- Loop Through Video Jobs
- Create AI Video (Veo 3 API)
- Wait for Video Rendering
- Retrieve Rendered Video
- Check Rendering Status

#### Node: Split Ideas into Video Jobs
- **Type / role:** Split Out; iterates over `output.scenes` into separate items.
- **Configuration:**
  - **Field to split:** `output.scenes`
  - **Include:** “allOtherFields” (keeps metadata alongside each scene item)
- **Inputs:** Agent output with `output.scenes`.
- **Outputs:** To **Loop Through Video Jobs**.
- **Edge cases:** If `output.scenes` missing or empty → no items → rendering block does nothing.

#### Node: Loop Through Video Jobs
- **Type / role:** Split In Batches; controls iterative processing of scene items.
- **Configuration:** Defaults (batch size not explicitly set; n8n default is typically 1).
- **Connections (important):**
  - Output 0 → **Create AI Video (Veo 3 API)** (start render for current scene)
  - Output 0 also → **Aggregate Video Assets** (this is unusual; see note below)
  - Input 1 (“continue”) is fed from **Check Rendering Status** (Processing path) to loop again.
- **Edge cases / design caution:**
  - The workflow connects **Loop Through Video Jobs → Aggregate Video Assets** directly. That means aggregation might run before rendered URLs exist (depending on item flow). In practice, you typically aggregate *after* all scenes completed. Consider moving aggregation after the loop ends.

#### Node: Create AI Video (Veo 3 API)
- **Type / role:** HTTP Request; starts a video generation job.
- **Configuration choices:**
  - **POST** `https://api.geminigen.ai/uapi/v1/video-gen/veo`
  - **Content-Type:** `multipart-form-data`
  - Body parameters:
    - `prompt` ← `$('Split Ideas into Video Jobs').item.json['output.scenes'].veo3_prompt`
    - `model` = `veo-3-fast`
    - `resolution` = `720p`
    - `aspect_ratio` = `9:16`
  - **Auth:** Generic credential (HTTP Header Auth) “GeminiGenAI”
- **Outputs:** To **Wait for Video Rendering** (expects a response containing a `uuid` later used).
- **Edge cases / failures:**
  - Provider API rate limits, invalid prompt, timeouts.
  - Expression references `$('Split Ideas into Video Jobs').item...` rather than current item `$json...`. This can misalign in batch/loop contexts; safer is `{{$json.veo3_prompt}}` if each item is a scene object.

#### Node: Wait for Video Rendering
- **Type / role:** Wait node; pauses execution before polling.
- **Configuration:** Default (manual wait duration not set in JSON).
- **Outputs:** To **Retrieve Rendered Video**.
- **Edge cases:** Without configured wait time, behavior depends on node defaults; could be immediate or require manual resume depending on n8n version/config.

#### Node: Retrieve Rendered Video
- **Type / role:** HTTP Request; checks job status / history.
- **Configuration choices:**
  - **GET** `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
  - **Auth:** HTTP Header Auth (“API gemeni GEn AI”)
- **Inputs:** From Wait node; expects `$json.uuid` present (from Create call).
- **Outputs:** To **Check Rendering Status**.
- **Edge cases:**
  - If the create endpoint returns a different property name (not `uuid`), polling breaks.
  - Expired job IDs or provider-side deletion.

#### Node: Check Rendering Status
- **Type / role:** Switch; routes based on status.
- **Rules configured:**
  - **Processing** if `{{ $json.status }}` equals `1`
  - **Completed** if `{{ $json.status }}` equals `2`
  - **Failed** if `{{ $json.status }}` equals `"3"` (string comparison) — inconsistent type vs previous numeric checks
- **Outputs / routing:**
  - One output goes back to **Wait for Video Rendering** (continue polling)  
  - Another output goes to **Loop Through Video Jobs** (continue to next scene)
- **Edge cases / failures:**
  - Type mismatch: if API returns numeric `3`, the “Failed” route may never trigger.
  - Current connections do not show any dedicated “Failed handling” node (logging, retries, mark row failed). Consider adding it.
  - The connection layout suggests polling and looping are intertwined; ensure “Completed” goes to batch “continue” while “Processing” goes to Wait.

---

### Block 4 — Video Post-processing

**Overview:** Collects the rendered video URLs from multiple scenes and sends them to an external concatenation service, producing a final combined video URL.

**Nodes involved:**
- Aggregate Video Assets
- Prepare Video Metadata & Order
- Merge Video Clips

#### Node: Aggregate Video Assets
- **Type / role:** Aggregate; collects multiple `generated_video[0].video_url` values into an array.
- **Configuration:** Aggregates field `generated_video[0].video_url`.
- **Inputs:** From **Loop Through Video Jobs** (but ideally should be from “Completed render outputs”).
- **Outputs:** To **Prepare Video Metadata & Order**.
- **Edge cases:**
  - If the provider returns a different response shape, field path won’t exist.
  - If aggregation runs too early, list may be incomplete.

#### Node: Prepare Video Metadata & Order
- **Type / role:** Code node; rewraps aggregated URLs into a specific JSON payload format.
- **Key logic (interpreted):**
  - Reads aggregated `item.json.video_url`
  - Ensures it is an array
  - Converts to `[{ video_url: <url> }, ...]`
  - Builds final object:
    - `video_urls`: formatted list
    - `id`: `"2323"` (hardcoded)
  - Outputs: `json.output` = `JSON.stringify(finalObject)`
- **Outputs:** To **Merge Video Clips**.
- **Edge cases:**
  - Hardcoded `id` may be expected by the external API; if not, remove or map from sheet row id.
  - If `item.json.video_url` is undefined, code throws or returns empty list.

#### Node: Merge Video Clips
- **Type / role:** HTTP Request; concatenates multiple clips.
- **Configuration choices:**
  - **POST** `https://no-code-architects-toolkit-843211623372.us-central1.run.app/v1/video/concatenate`
  - Sends **JSON body**: `={{ $json.output }}`
  - **Auth:** HTTP Header Auth (“NCA toolkit”)
- **Outputs:** To **Publish Video to Facebook** (uses response as media URL).
- **Edge cases:**
  - Service availability, payload format mismatch (string vs object; here it sends a JSON string as body).
  - Consider sending parsed JSON object instead of string if API expects object.

---

### Block 5 — Publishing & Tracking

**Overview:** Publishes the merged video to Facebook and updates the idea’s status in Google Sheets.

**Nodes involved:**
- Publish Video to Facebook
- Update Status & Results in Sheet

#### Node: Publish Video to Facebook
- **Type / role:** Blotato node; social publishing.
- **Configuration choices:**
  - Platform: `facebook`
  - Account: “Giang VT”
  - Facebook Page: ID `688227101036478`
  - Post text: `={{ $('Generate AI Video Script (Health Niche)').item.json.output.video_metadata.caption }}`
  - Media URLs: `={{ $json.response }}` (from concatenate endpoint)
- **Credentials:** Blotato API (“Blotato GiangxAI”)
- **Outputs:** To **Update Status & Results in Sheet**
- **Edge cases:**
  - If concatenate response field isn’t `response` (or is a nested object), media URL mapping fails.
  - Facebook posting restrictions, video processing delays, invalid page token, rate limits.

#### Node: Update Status & Results in Sheet
- **Type / role:** Google Sheets; updates processed row.
- **Configuration choices:**
  - **Operation:** Update
  - **Match column:** `row_number` (read-only field from “Load Viral Ideas from Sheet”)
  - Writes:
    - `Status` = `"Done"`
    - `row_number` = `={{ $('Load Viral Ideas from Sheet').item.json.row_number }}`
- **Credentials:** Google Sheets OAuth2 (“GiangXAI”)
- **Edge cases:**
  - If `row_number` missing (depends on Sheets node output settings), update won’t match anything.
  - Consider updating earlier to “Processing” to avoid duplicate processing by concurrent runs.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Collect Viral Ideas | n8n-nodes-base.scheduleTrigger | Scheduled trigger for scraping | — | Scrape Viral Health Content (Apify) | ## Viral Content Collection<br>Automatically collects viral health-related content from external sources on a schedule. The data is cleaned, normalized, and stored in Google Sheets to build a reusable database of proven viral ideas. |
| Scrape Viral Health Content (Apify) | @apify/n8n-nodes-apify.apify | Run TikTok scraper actor and return dataset | Collect Viral Ideas | Normalize Viral Content Data | ## Viral Content Collection<br>Automatically collects viral health-related content from external sources on a schedule. The data is cleaned, normalized, and stored in Google Sheets to build a reusable database of proven viral ideas. |
| Normalize Viral Content Data | n8n-nodes-base.set | Normalize scraped fields to sheet schema | Scrape Viral Health Content (Apify) | Save Viral Ideas to Google Sheets | ## Viral Content Collection<br>Automatically collects viral health-related content from external sources on a schedule. The data is cleaned, normalized, and stored in Google Sheets to build a reusable database of proven viral ideas. |
| Save Viral Ideas to Google Sheets | n8n-nodes-base.googleSheets | Append “Pending” idea row to sheet | Normalize Viral Content Data | — | ## Viral Content Collection<br>Automatically collects viral health-related content from external sources on a schedule. The data is cleaned, normalized, and stored in Google Sheets to build a reusable database of proven viral ideas. |
| Process Viral Ideas | n8n-nodes-base.scheduleTrigger | Scheduled trigger for processing pending ideas | — | Load Viral Ideas from Sheet | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Load Viral Ideas from Sheet | n8n-nodes-base.googleSheets | Load first row with Status=Pending | Process Viral Ideas | Analyze Viral Potential & Hook | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Analyze Viral Potential & Hook | @n8n/n8n-nodes-langchain.googleGemini | Gemini video analysis from URL | Load Viral Ideas from Sheet | Generate AI Video Script (Health Niche) | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Gemini LLM | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM provider for agent/parser | — (AI connection) | Generate AI Video Script (Health Niche); Parse Script & Video Parameters | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Reasoning / Prompt Logic | @n8n/n8n-nodes-langchain.toolThink | Optional “think” tool for the agent | — (AI connection) | Generate AI Video Script (Health Niche) | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Parse Script & Video Parameters | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON schema for agent output | — (AI connection) | Generate AI Video Script (Health Niche) | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Generate AI Video Script (Health Niche) | @n8n/n8n-nodes-langchain.agent | Generate scenes + Veo prompts + caption/hashtags | Analyze Viral Potential & Hook | Split Ideas into Video Jobs | ## Idea Analysis & Script Generation<br>Analyzes each viral idea to identify hooks, angles, and engagement patterns. AI models are used to generate structured, short-form video scripts optimized for health content and social media virality. |
| Split Ideas into Video Jobs | n8n-nodes-base.splitOut | Split scenes array into per-scene items | Generate AI Video Script (Health Niche) | Loop Through Video Jobs | ## AI Video Rendering (Veo 3)<br>Converts AI-generated scripts into videos using the Veo 3 API. It handles video creation, rendering status checks, retries, and retrieval of completed video assets automatically. |
| Loop Through Video Jobs | n8n-nodes-base.splitInBatches | Iterate through scenes (batch loop) | Split Ideas into Video Jobs; Check Rendering Status | Aggregate Video Assets; Create AI Video (Veo 3 API) | ## AI Video Rendering (Veo 3)<br>Converts AI-generated scripts into videos using the Veo 3 API. It handles video creation, rendering status checks, retries, and retrieval of completed video assets automatically. |
| Create AI Video (Veo 3 API) | n8n-nodes-base.httpRequest | Start Veo render job via provider API | Loop Through Video Jobs | Wait for Video Rendering | ## AI Video Rendering (Veo 3)<br>Converts AI-generated scripts into videos using the Veo 3 API. It handles video creation, rendering status checks, retries, and retrieval of completed video assets automatically. |
| Wait for Video Rendering | n8n-nodes-base.wait | Pause before polling render status | Create AI Video (Veo 3 API); Check Rendering Status | Retrieve Rendered Video | ## AI Video Rendering (Veo 3)<br>Converts AI-generated scripts into videos using the Veo 3 API. It handles video creation, rendering status checks, retries, and retrieval of completed video assets automatically. |
| Retrieve Rendered Video | n8n-nodes-base.httpRequest | Poll job history/status by uuid | Wait for Video Rendering | Check Rendering Status | ## AI Video Rendering (Veo 3)<br>Converts AI-generated scripts into videos using the Veo 3 API. It handles video creation, rendering status checks, retries, and retrieval of completed video assets automatically. |
| Check Rendering Status | n8n-nodes-base.switch | Route based on render status | Retrieve Rendered Video | Wait for Video Rendering; Loop Through Video Jobs | ## AI Video Rendering (Veo 3)<br>Converts AI-generated scripts into videos using the Veo 3 API. It handles video creation, rendering status checks, retries, and retrieval of completed video assets automatically. |
| Aggregate Video Assets | n8n-nodes-base.aggregate | Collect scene video URLs | Loop Through Video Jobs | Prepare Video Metadata & Order | ## Video Post-processing<br>Processes rendered video assets, including aggregation, ordering, and optional merging of clips. It prepares the final video output ready for publishing. |
| Prepare Video Metadata & Order | n8n-nodes-base.code | Format payload for concatenation API | Aggregate Video Assets | Merge Video Clips | ## Video Post-processing<br>Processes rendered video assets, including aggregation, ordering, and optional merging of clips. It prepares the final video output ready for publishing. |
| Merge Video Clips | n8n-nodes-base.httpRequest | Concatenate clips into final video | Prepare Video Metadata & Order | Publish Video to Facebook | ## Video Post-processing<br>Processes rendered video assets, including aggregation, ordering, and optional merging of clips. It prepares the final video output ready for publishing. |
| Publish Video to Facebook | @blotato/n8n-nodes-blotato.blotato | Publish final video to Facebook page | Merge Video Clips | Update Status & Results in Sheet | ## Publishing & Tracking<br>Publishes completed videos to social platforms and updates Google Sheets with publishing status and results. It ensures every video is tracked from idea to distribution. |
| Update Status & Results in Sheet | n8n-nodes-base.googleSheets | Mark row as Done using row_number | Publish Video to Facebook | — | ## Publishing & Tracking<br>Publishes completed videos to social platforms and updates Google Sheets with publishing status and results. It ensures every video is tracked from idea to distribution. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation block | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation block | — | — |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation block | — | — |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation block | — | — |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation block | — | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Setup / credits note | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **Create Viral AI Health Videos**
   - (Optional) Keep it inactive until credentials are set.

2) **Add Block 1 nodes (collection)**
   1. Add **Schedule Trigger** node named **Collect Viral Ideas**
      - Set interval: every **6 hours**.
   2. Add **Apify** node named **Scrape Viral Health Content (Apify)**
      - Operation: **Run actor and get dataset**
      - Actor: **clockworks/tiktok-scraper** (select from list if available)
      - Provide input JSON:
        - profiles: `["@yeucotheminhh"]`
        - resultsPerPage: `20`
        - proxyCountryCode: `VN`
        - shouldDownloadVideos: `true`
        - (set other flags as desired)
      - Set Apify OAuth2 credentials.
      - Connect: **Collect Viral Ideas → Scrape Viral Health Content (Apify)**
   3. Add **Set** node named **Normalize Viral Content Data**
      - Add fields matching:
        - Caption, Date Post, Username, Link user, Link video, VideoURL, Likes, Shares, View, Save, Hasgtag
      - Use expressions to map from Apify output (as listed in section 2).
      - Connect: **Scrape Viral Health Content (Apify) → Normalize Viral Content Data**
   4. Add **Google Sheets** node named **Save Viral Ideas to Google Sheets**
      - Auth: Google Sheets OAuth2
      - Operation: **Append**
      - Select Spreadsheet: your “Health viral content”
      - Select Sheet: “Trang tính1”
      - Map columns, including `Status` = **Pending**
      - Important: map `VideoURL` carefully; if `VideoURL` is array, use first element or join.
      - Connect: **Normalize Viral Content Data → Save Viral Ideas to Google Sheets**

3) **Add Block 2 nodes (processing + AI script)**
   1. Add **Schedule Trigger** named **Process Viral Ideas**
      - Interval: every **10 hours**
   2. Add **Google Sheets** node named **Load Viral Ideas from Sheet**
      - Operation: **Read / Get Many** (the node UI differs by n8n version; use a read operation that supports filters)
      - Filter: `Status` equals `Pending`
      - Enable: **Return First Match**
      - Ensure the node outputs a `row_number` field (Google Sheets node typically includes it).
      - Connect: **Process Viral Ideas → Load Viral Ideas from Sheet**
   3. Add **Google Gemini (LangChain)** node named **Analyze Viral Potential & Hook**
      - Resource: **video**
      - Operation: **analyze**
      - Model: `models/gemini-3-pro-preview`
      - Video URLs: map from the sheet’s `VideoURL` column
      - Connect: **Load Viral Ideas from Sheet → Analyze Viral Potential & Hook**
   4. Add **Gemini Chat Model (LangChain)** node named **Gemini LLM**
      - Model: `models/gemini-3-pro-preview`
      - Set Google Gemini/PaLM credentials
   5. Add **Tool Think (LangChain)** named **Reasoning / Prompt Logic**
   6. Add **Structured Output Parser (LangChain)** named **Parse Script & Video Parameters**
      - Enable auto-fix
      - Define schema containing `scenes[]` and `video_metadata` (as in section 2).
   7. Add **Agent (LangChain)** node named **Generate AI Video Script (Health Niche)**
      - Input text: map from Gemini analyze output (e.g., `content.parts[0].text`)
      - System message: paste your constraints (Vietnamese dialogue, no on-screen text, stylized food characters, etc.)
      - Attach:
        - Language model: **Gemini LLM**
        - Tool: **Reasoning / Prompt Logic**
        - Output parser: **Parse Script & Video Parameters**
      - Connect: **Analyze Viral Potential & Hook → Generate AI Video Script (Health Niche)**

4) **Add Block 3 nodes (rendering loop)**
   1. Add **Split Out** node named **Split Ideas into Video Jobs**
      - Field to split: `output.scenes`
      - Include all other fields: enabled
      - Connect: **Generate AI Video Script (Health Niche) → Split Ideas into Video Jobs**
   2. Add **Split In Batches** node named **Loop Through Video Jobs**
      - Batch size: set to **1** (recommended) to render one scene at a time.
      - Connect: **Split Ideas into Video Jobs → Loop Through Video Jobs**
   3. Add **HTTP Request** node named **Create AI Video (Veo 3 API)**
      - Method: POST
      - URL: `https://api.geminigen.ai/uapi/v1/video-gen/veo`
      - Body: multipart/form-data
      - Fields: `prompt`, `model=veo-3-fast`, `resolution=720p`, `aspect_ratio=9:16`
      - Credentials: HTTP Header Auth (API key/token from GeminiGenAI or alternative provider)
      - Prompt expression: preferably use the current item’s scene prompt (avoid cross-node item reference).
      - Connect: **Loop Through Video Jobs → Create AI Video (Veo 3 API)**
   4. Add **Wait** node named **Wait for Video Rendering**
      - Configure a delay (e.g., 30–90 seconds) to avoid hammering the status endpoint.
      - Connect: **Create AI Video (Veo 3 API) → Wait for Video Rendering**
   5. Add **HTTP Request** node named **Retrieve Rendered Video**
      - Method: GET
      - URL: `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
      - Auth: HTTP Header Auth
      - Connect: **Wait for Video Rendering → Retrieve Rendered Video**
   6. Add **Switch** node named **Check Rendering Status**
      - Create outputs for:
        - Processing: status == 1
        - Completed: status == 2
        - Failed: status == 3 (ensure consistent numeric comparison)
      - Connect: **Retrieve Rendered Video → Check Rendering Status**
   7. Wire loop logic:
      - **Processing → Wait for Video Rendering** (poll again)
      - **Completed → Loop Through Video Jobs (Continue input)** (advance to next scene)
      - **Failed →** add your own retry/mark-failed handling (recommended)

5) **Add Block 4 nodes (aggregate + concatenate)**
   1. Add **Aggregate** node **Aggregate Video Assets**
      - Aggregate field holding the rendered URL returned by the provider (match your actual response shape)
      - Ensure aggregation happens **after all scenes are completed** (commonly after SplitInBatches finishes).
   2. Add **Code** node **Prepare Video Metadata & Order**
      - Build the payload required by your concatenate endpoint (`video_urls: [{video_url:...}]`).
   3. Add **HTTP Request** node **Merge Video Clips**
      - POST to your concatenate service endpoint
      - Pass JSON body as required
      - Auth: HTTP Header Auth
   - Connect these three in order.

6) **Add Block 5 nodes (publish + sheet update)**
   1. Add **Blotato** node **Publish Video to Facebook**
      - Platform: Facebook
      - Choose account + page
      - Text: map from `output.video_metadata.caption`
      - Media URL: map from concatenate output (ensure it is a direct downloadable video URL)
   2. Add **Google Sheets** node **Update Status & Results in Sheet**
      - Operation: Update
      - Match by `row_number`
      - Set `Status` = Done
      - Optionally also write publish URL, timestamp, platform post id, etc.
   - Connect: **Merge Video Clips → Publish Video to Facebook → Update Status & Results in Sheet**

7) **Credentials to configure**
   - **Apify OAuth2** (actor execution)
   - **Google Sheets OAuth2**
   - **Google Gemini / PaLM API** (for both Gemini nodes)
   - **HTTP Header Auth** for:
     - Veo provider endpoint (`api.geminigen.ai`)
     - Concatenation endpoint (`no-code-architects-toolkit...`)
   - **Blotato API** for publishing

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Setup guide n8n | https://n8n.partnerlinks.io/giangxai |
| Veo 3 API credentials suggestions | https://geminigen.ai/ and https://kie.ai?ref=f8cec88ea15f9ecbff52ccbafa41dd6e |
| Publishing provider | https://blotato.com/?ref=giang9s |
| Workflow intent (end-to-end, no manual intervention once configured) | Sticky note “Workflow Setup Guide” content |

