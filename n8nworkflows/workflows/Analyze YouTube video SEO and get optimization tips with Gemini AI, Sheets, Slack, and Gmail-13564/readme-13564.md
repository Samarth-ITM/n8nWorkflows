Analyze YouTube video SEO and get optimization tips with Gemini AI, Sheets, Slack, and Gmail

https://n8nworkflows.xyz/workflows/analyze-youtube-video-seo-and-get-optimization-tips-with-gemini-ai--sheets--slack--and-gmail-13564


# Analyze YouTube video SEO and get optimization tips with Gemini AI, Sheets, Slack, and Gmail

## 1. Workflow Overview

**Title:** Analyze YouTube video SEO and get optimization tips with Gemini AI, Sheets, Slack, and Gmail

**Purpose:**  
This workflow collects a YouTube video URL via an n8n form, extracts the video ID, retrieves metadata and captions, formats the data for analysis, sends it to **Google Gemini** for SEO optimization recommendations, then outputs the report to **Google Sheets**, **Slack**, and **Gmail**.

**Target use cases**
- Content teams auditing YouTube SEO at scale (title/description/tags/captions)
- Creators wanting fast optimization suggestions for a single video
- Agencies producing repeatable “SEO audit + recommendations” reports

### Logical blocks
**1.1 Input Reception & ID Extraction**
- Receives a URL and derives the YouTube Video ID.

**1.2 YouTube Data Collection**
- Fetches video metadata, then captions/transcript via HTTP requests.

**1.3 Data Preparation**
- Normalizes/structures the collected data into an SEO-focused payload for the LLM.

**1.4 AI SEO Analysis (Gemini)**
- Runs an LLM chain using Google Gemini to generate optimization tips.

**1.5 Reporting & Distribution**
- Parses AI output and distributes it to Sheets, Slack, and Gmail.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & ID Extraction

**Overview:** Collects a YouTube URL through a form trigger and extracts the video identifier needed to query YouTube endpoints.

**Nodes involved:**
- YouTube URL Input
- Extract Video ID

#### Node: **YouTube URL Input**
- **Type / role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) – user-facing entry point.
- **Configuration (interpreted):** Uses default form trigger settings (no explicit parameters shown). Typically this node defines one or more form fields (here not visible in JSON), expected to include a YouTube URL.
- **Input/Output:**
  - **Input:** None (trigger).
  - **Output:** Form submission JSON (e.g., a field containing the URL).
  - **Next node:** Extract Video ID.
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Form field not present or empty → downstream extraction fails.
  - URL variants (youtu.be, shorts, playlist links, additional query params) may not parse correctly unless code handles them.

#### Node: **Extract Video ID**
- **Type / role:** `Code` (`n8n-nodes-base.code`) – transforms input URL into `videoId`.
- **Configuration (interpreted):** Custom JavaScript (not included in JSON) expected to:
  - Read the submitted URL (likely from `$json`).
  - Parse common formats (`watch?v=`, `youtu.be/`, `/shorts/`).
  - Output an object containing `videoId` (and possibly the normalized URL).
- **Input/Output:**
  - **Input:** Data from YouTube URL Input.
  - **Output:** Data with extracted `videoId`.
  - **Next node:** Get Video Metadata.
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Non-YouTube URL or missing ID → results in empty/invalid `videoId`.
  - URLs containing playlist parameters may cause incorrect extraction if not explicitly handled.
  - If code throws, workflow stops unless error handling is configured at workflow level.

---

### 2.2 YouTube Data Collection

**Overview:** Fetches the video’s metadata first, then retrieves captions/transcript. Both are needed for SEO analysis.

**Nodes involved:**
- Get Video Metadata
- Get Video Captions

#### Node: **Get Video Metadata**
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) – calls an external API endpoint to retrieve metadata.
- **Configuration (interpreted):**
  - Parameters are not shown in JSON, so the exact URL and authentication are unspecified.
  - Typically this would call **YouTube Data API v3** endpoint like `videos?part=snippet,contentDetails,statistics&id={{$json.videoId}}&key=...`.
- **Input/Output:**
  - **Input:** `videoId` from Extract Video ID.
  - **Output:** Metadata JSON (title, description, tags, channel, stats, etc., depending on configuration).
  - **Next node:** Get Video Captions.
- **Version notes:** typeVersion **4.2**.
- **Edge cases / failures:**
  - Missing/invalid API key or OAuth credentials → 401/403.
  - Video not found / private / age-restricted → 404 or restricted response.
  - Quota exceeded (YouTube API) → 403 with quota reason.
  - If the node is configured with “Continue On Fail” (not shown), downstream may receive partial/empty data.

#### Node: **Get Video Captions**
- **Type / role:** `HTTP Request` (`n8n-nodes-base.httpRequest`) – retrieves captions/subtitles/transcript.
- **Configuration (interpreted):**
  - Not shown; could be:
    - YouTube Captions API (requires OAuth and permissions), or
    - a third-party transcript endpoint/service.
  - Expected to use `videoId` and/or caption track identifiers from metadata.
- **Input/Output:**
  - **Input:** Output of Get Video Metadata (likely includes `videoId` and possibly caption track info).
  - **Output:** Captions text or structured caption segments.
  - **Next node:** Format SEO Data.
- **Version notes:** typeVersion **4.2**.
- **Edge cases / failures:**
  - Captions disabled/unavailable → empty transcript.
  - Auto-captions may require different retrieval logic.
  - Non-English captions or multiple tracks → must pick a language; otherwise analysis may be inconsistent.
  - Large transcripts can exceed LLM context limits later unless trimmed/summarized.

---

### 2.3 Data Preparation

**Overview:** Converts raw metadata + captions into a single structured payload optimized for SEO analysis by the LLM.

**Nodes involved:**
- Format SEO Data

#### Node: **Format SEO Data**
- **Type / role:** `Code` (`n8n-nodes-base.code`) – data shaping and normalization.
- **Configuration (interpreted):**
  - Script (not included) likely:
    - Extracts title/description/tags/category/stats from metadata response.
    - Extracts transcript text and possibly truncates it.
    - Builds a prompt-ready JSON object (e.g., `seoInput`) for the AI node.
- **Key variables/expressions:** Not visible; typically `$json` and possibly `$node["Get Video Metadata"].json`.
- **Input/Output:**
  - **Input:** Captions response (and potentially metadata through merged structure or referenced node data).
  - **Output:** A consolidated SEO dataset for the AI chain.
  - **Next node:** AI SEO Analysis.
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - Mismatched response shapes from the HTTP nodes → undefined property access.
  - Very long fields (description/transcript) → may need truncation to prevent LLM failure.
  - Missing tags/description → prompt should handle empty strings.

---

### 2.4 AI SEO Analysis (Gemini)

**Overview:** Sends prepared SEO data to a LangChain-based LLM chain using **Google Gemini** to produce optimization recommendations.

**Nodes involved:**
- AI SEO Analysis
- Google Gemini

#### Node: **Google Gemini**
- **Type / role:** `Google Gemini Chat Model` (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) – provides the LLM used by the chain.
- **Configuration (interpreted):**
  - Exact model parameters not shown (model name, temperature, etc.).
  - Requires Google AI / Gemini credentials (API key or configured credential in n8n).
- **Input/Output connections:**
  - Connected to **AI SEO Analysis** via `ai_languageModel`.
- **Version notes:** typeVersion **1**.
- **Edge cases / failures:**
  - Invalid/expired API key → auth errors.
  - Safety filters or blocked content (rare for SEO text, but possible).
  - Token/context overflow if transcript is too long.

#### Node: **AI SEO Analysis**
- **Type / role:** `LangChain LLM Chain` (`@n8n/n8n-nodes-langchain.chainLlm`) – orchestrates prompting and model call.
- **Configuration (interpreted):**
  - Receives the formatted SEO dataset from Format SEO Data.
  - Uses **Google Gemini** as the language model input (connected via `ai_languageModel`).
  - Produces an SEO analysis report (likely text, possibly structured).
- **Input/Output:**
  - **Input:** Formatted SEO payload.
  - **Output:** LLM response (analysis + tips).
  - **Next node:** Parse AI Results.
- **Version notes:** typeVersion **1.4**.
- **Edge cases / failures:**
  - If prompt expects structured JSON but model returns free text → parsing step can break.
  - Rate limits from Gemini API.
  - Missing fields in the formatted SEO payload can reduce output quality.

---

### 2.5 Reporting & Distribution

**Overview:** Converts the LLM output into a structured report and sends it to Google Sheets, Slack, and Gmail in parallel.

**Nodes involved:**
- Parse AI Results
- Save Report to Sheets
- Notify Team on Slack
- Email Report to Creator

#### Node: **Parse AI Results**
- **Type / role:** `Code` (`n8n-nodes-base.code`) – post-processing of AI output.
- **Configuration (interpreted):**
  - Script (not included) likely:
    - Extracts key sections from LLM output (e.g., title suggestions, description rewrite, keyword list, chapter ideas).
    - Produces clean fields for Sheets row insertion and message bodies.
- **Input/Output:**
  - **Input:** AI SEO Analysis output.
  - **Output:** Structured fields used by the three reporting nodes.
  - **Next nodes (parallel):** Save Report to Sheets, Notify Team on Slack, Email Report to Creator.
- **Version notes:** typeVersion **2**.
- **Edge cases / failures:**
  - If AI output format changes, parsing can fail (undefined fields / JSON.parse errors).
  - Special characters/newlines may need escaping for Sheets or Slack formatting.

#### Node: **Save Report to Sheets**
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) – persists report.
- **Configuration (interpreted):**
  - Not shown; typically “Append Row” or “Add row”.
  - Requires Google Sheets OAuth2 credentials in n8n.
  - Expected mapping: video URL/ID + metadata + AI recommendations + timestamp.
- **Input/Output:**
  - **Input:** Parsed report fields.
  - **Output:** Sheets API response (row ID / update info).
- **Version notes:** typeVersion **4.4**.
- **Edge cases / failures:**
  - Wrong spreadsheet ID/sheet tab name → 404.
  - Permission issues → 403.
  - Column mismatch (too many/few fields) → append errors.
  - API quota/limits.

#### Node: **Notify Team on Slack**
- **Type / role:** `Slack` (`n8n-nodes-base.slack`) – sends a notification to a channel/user.
- **Configuration (interpreted):**
  - Not shown; typically “Post Message”.
  - Requires Slack OAuth token/credential.
  - Message likely includes video title + key recommendations + link to the Sheets row/report.
- **Input/Output:**
  - **Input:** Parsed report fields.
  - **Output:** Slack API response (message timestamp, channel).
- **Version notes:** typeVersion **2.2**.
- **Edge cases / failures:**
  - Missing scopes (e.g., `chat:write`) → 403.
  - Wrong channel ID/name → channel_not_found.
  - Message too long (Slack limits) if transcript/tips are pasted in full.

#### Node: **Email Report to Creator**
- **Type / role:** `Gmail` (`n8n-nodes-base.gmail`) – emails the SEO report to the creator.
- **Configuration (interpreted):**
  - Not shown; typically “Send Email”.
  - Requires Google OAuth2 credentials with Gmail scope.
  - Email body likely contains summarized recommendations and/or a link to Sheets.
- **Input/Output:**
  - **Input:** Parsed report fields (recipient email must be available from form input or configuration).
  - **Output:** Gmail API send result (message ID).
- **Version notes:** typeVersion **2.1**.
- **Edge cases / failures:**
  - Recipient missing/invalid.
  - OAuth not authorized for Gmail send scope.
  - Gmail send limits, or message rejected due to content/policy.
  - HTML vs plain text formatting issues if not handled.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| YouTube URL Input | Form Trigger | Entry point: collect YouTube URL | — | Extract Video ID |  |
| Extract Video ID | Code | Parse URL and derive `videoId` | YouTube URL Input | Get Video Metadata |  |
| Get Video Metadata | HTTP Request | Fetch YouTube video metadata | Extract Video ID | Get Video Captions |  |
| Get Video Captions | HTTP Request | Fetch captions/transcript | Get Video Metadata | Format SEO Data |  |
| Format SEO Data | Code | Consolidate metadata + transcript into AI payload | Get Video Captions | AI SEO Analysis |  |
| AI SEO Analysis | LangChain LLM Chain | Run SEO analysis prompt/chain | Format SEO Data; (LM: Google Gemini) | Parse AI Results |  |
| Google Gemini | Gemini Chat Model | LLM provider for the chain | — | AI SEO Analysis (ai_languageModel) |  |
| Parse AI Results | Code | Convert AI output into structured report fields | AI SEO Analysis | Save Report to Sheets; Notify Team on Slack; Email Report to Creator |  |
| Save Report to Sheets | Google Sheets | Store report for tracking | Parse AI Results | — |  |
| Notify Team on Slack | Slack | Team notification | Parse AI Results | — |  |
| Email Report to Creator | Gmail | Email the report | Parse AI Results | — |  |
| Main Overview Sticky | Sticky Note | Canvas annotation (empty content) | — | — |  |
| Data Collection Section | Sticky Note | Canvas annotation (empty content) | — | — |  |
| AI Optimization Section | Sticky Note | Canvas annotation (empty content) | — | — |  |
| Reporting Section | Sticky Note | Canvas annotation (empty content) | — | — |  |

*Note:* All sticky notes have empty content in the provided workflow JSON, so no comments can be replicated into the table.

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger: “YouTube URL Input”**
   - Add node: **Form Trigger**
   - Add a form field like:
     - `youtubeUrl` (Text, required)
     - (Optional) `creatorEmail` (Email, if you want Gmail to send to the submitter)
   - Save the form and copy its public URL if needed.

2. **Add “Extract Video ID” (Code node)**
   - Add node: **Code**
   - Implement logic to parse `$json.youtubeUrl` and output:
     - `videoId`
     - `videoUrl` (optional normalized)
     - `creatorEmail` pass-through if captured
   - Connect: **YouTube URL Input → Extract Video ID**

3. **Add “Get Video Metadata” (HTTP Request)**
   - Add node: **HTTP Request**
   - Configure to call YouTube Data API (common approach):
     - Method: `GET`
     - URL: `https://www.googleapis.com/youtube/v3/videos`
     - Query params:
       - `part=snippet,statistics,contentDetails`
       - `id={{$json.videoId}}`
       - `key=<YOUR_YOUTUBE_API_KEY>` (or use OAuth where applicable)
   - Connect: **Extract Video ID → Get Video Metadata**

4. **Add “Get Video Captions” (HTTP Request)**
   - Add node: **HTTP Request**
   - Choose your transcript strategy:
     - YouTube Captions API (OAuth, more complex), or
     - A transcript service endpoint you control/use.
   - Ensure the request uses `{{$json.videoId}}` (or caption track id if needed).
   - Connect: **Get Video Metadata → Get Video Captions**

5. **Add “Format SEO Data” (Code)**
   - Add node: **Code**
   - Build a concise object for the AI model, e.g.:
     - `title`, `description`, `tags`, `channelTitle`, `publishedAt`
     - `transcript` (truncate to a safe length)
     - `stats` (views, likes, comments if available)
   - Connect: **Get Video Captions → Format SEO Data**

6. **Add “Google Gemini” (Gemini Chat Model)**
   - Add node: **Google Gemini Chat Model**
   - Configure credentials: **Gemini / Google AI API key** in n8n credentials.
   - Set model options as desired (temperature, etc.).

7. **Add “AI SEO Analysis” (LangChain LLM Chain)**
   - Add node: **LangChain LLM Chain**
   - Configure prompt to request output in a predictable structure (recommended), e.g. sections:
     - Issues found
     - Title alternatives (5)
     - Description rewrite
     - Keyword/tag suggestions
     - Hook improvements
     - Chapter suggestions
   - Connect language model:
     - **Google Gemini → AI SEO Analysis** via the model connection (`ai_languageModel`)
   - Connect main data:
     - **Format SEO Data → AI SEO Analysis**

8. **Add “Parse AI Results” (Code)**
   - Add node: **Code**
   - Parse the AI output into fields for:
     - Sheets columns (e.g., `videoId`, `title`, `topIssues`, `recommendedTitle1..n`, `recommendedTags`, `summary`, `createdAt`)
     - Slack message text
     - Email subject/body
   - Connect: **AI SEO Analysis → Parse AI Results**

9. **Add “Save Report to Sheets” (Google Sheets)**
   - Add node: **Google Sheets**
   - Credential: Google OAuth2 with access to the target spreadsheet.
   - Operation: typically **Append row** to a selected sheet.
   - Map columns from Parse AI Results output.
   - Connect: **Parse AI Results → Save Report to Sheets**

10. **Add “Notify Team on Slack” (Slack)**
   - Add node: **Slack**
   - Credential: Slack OAuth with `chat:write`.
   - Operation: **Post message** to a channel.
   - Compose message from parsed fields (keep it short + include link to Sheet/video).
   - Connect: **Parse AI Results → Notify Team on Slack**

11. **Add “Email Report to Creator” (Gmail)**
   - Add node: **Gmail**
   - Credential: Google OAuth2 with Gmail send permission.
   - Operation: **Send**
   - To: `{{$json.creatorEmail}}` (if collected) or a fixed address.
   - Subject/body from parsed fields.
   - Connect: **Parse AI Results → Email Report to Creator**

12. **(Optional) Add Sticky Notes**
   - Add 4 sticky notes to label the canvas sections:
     - Main Overview
     - Data Collection Section
     - AI Optimization Section
     - Reporting Section
   - In the provided workflow, these are present but have **empty content**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text | Provided workflow JSON includes 4 sticky notes with empty `content` fields |
| The workflow’s HTTP request node parameters are not included | To reproduce, you must define the exact YouTube metadata/captions endpoints and authentication method |

If you want, paste the *actual* parameter configurations for the HTTP Request / Sheets / Slack / Gmail / AI nodes (or screenshots), and I can document the exact endpoints, mappings, prompts, and credentials precisely.