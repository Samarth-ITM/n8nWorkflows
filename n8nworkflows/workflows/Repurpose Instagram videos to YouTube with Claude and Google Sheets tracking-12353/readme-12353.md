Repurpose Instagram videos to YouTube with Claude and Google Sheets tracking

https://n8nworkflows.xyz/workflows/repurpose-instagram-videos-to-youtube-with-claude-and-google-sheets-tracking-12353


# Repurpose Instagram videos to YouTube with Claude and Google Sheets tracking

## 1. Workflow Overview

**Purpose:**  
This workflow automates repurposing an Instagram video into a YouTube upload, enhances the title/description using Anthropic Claude, logs results to Google Sheets, and sends a WhatsApp notification when complete.

**Target use cases:**  
- Content creators and social media managers republishing Instagram content to YouTube  
- Agencies managing multiple client accounts and needing consistent, trackable uploads  
- Teams that want AI-assisted YouTube SEO metadata generation and lightweight performance guidance

### 1.1 Logical Blocks (by dependency chain)

1. **Trigger & Configuration**  
   Runs on a schedule, then injects required configuration values (Instagram IDs/tokens, Sheet ID, defaults).

2. **Instagram Ingestion & File Retrieval**  
   Fetches media metadata from Instagram Graph API, then downloads the actual video file.

3. **YouTube Upload Preparation & AI Metadata Optimization**  
   Prepares upload fields, calls Claude (LangChain agent) to generate structured optimized metadata.

4. **Publishing, Post-upload Analysis, Tracking & Notification**  
   Uploads to YouTube, runs a second Claude analysis agent on upload data, logs to Google Sheets, and notifies via WhatsApp.

---

## 2. Block-by-Block Analysis

### Block 1 — Trigger & Configuration

**Overview:**  
Starts the workflow on a time-based schedule and sets reusable configuration fields (placeholders by default) consumed by later nodes via expressions.

**Nodes involved:**  
- Initiates the workflow at preset interval  
- Workflow Configuration

#### Node: “Initiates the workflow at preset interval”
- **Type / role:** `Schedule Trigger` — workflow entry point.
- **Configuration (interpreted):** Triggers daily at **09:00** (server / workflow timezone as configured in n8n).
- **Inputs/Outputs:** No inputs; outputs into **Workflow Configuration**.
- **Edge cases / failures:**
  - Timezone mismatches (instance timezone vs desired business timezone).
  - Missed executions if n8n is offline at trigger time (depends on n8n scheduling behavior).

#### Node: “Workflow Configuration”
- **Type / role:** `Set` — defines configuration and defaults.
- **Configuration choices:**
  - Creates these fields (placeholders in template):
    - `youtubeTitle`
    - `youtubeDescription`
    - `googleSheetId`
    - `instagramMediaId`
    - `instagramAccessToken`
    - `whatsappPhoneNumber`
  - **Include Other Fields = true** (passes through incoming data too).
- **Key variables / expressions used downstream:**
  - Accessed via: `$('Workflow Configuration').first().json.<field>`
- **Outputs:** Main output to **Get Instagram Media**.
- **Edge cases / failures:**
  - Placeholders not replaced: downstream API calls will fail (invalid Instagram token, missing media ID, invalid sheet ID, etc.).
  - In multi-item executions, use of `.first()` means only the first item’s config is used (intentional for single-run, but a limitation for batch runs).

---

### Block 2 — Instagram Ingestion & File Retrieval

**Overview:**  
Pulls Instagram media metadata (URL, caption, type) via Graph API and downloads the actual media as a binary file for later use.

**Nodes involved:**  
- Get Instagram Media  
- Download Instagram Video

#### Node: “Get Instagram Media”
- **Type / role:** `HTTP Request` — calls Instagram Graph API to fetch metadata.
- **Configuration choices:**
  - **URL (expression):**  
    `https://graph.instagram.com/{{instagramMediaId}}?fields=media_url,media_type,caption&access_token={{instagramAccessToken}}`
    pulled from **Workflow Configuration**.
  - Response format: **JSON**
- **Inputs/Outputs:**  
  Input from **Workflow Configuration**; output to **Download Instagram Video**.
- **Edge cases / failures:**
  - 400/401 from invalid/expired `instagramAccessToken`.
  - Incorrect `instagramMediaId` or missing permissions/scopes.
  - Media type not video (e.g., IMAGE, CAROUSEL): `media_url` may not be a downloadable video; the workflow does not validate `media_type`.

#### Node: “Download Instagram Video”
- **Type / role:** `HTTP Request` — downloads the media URL as a file (binary).
- **Configuration choices:**
  - URL: `={{ $json.media_url }}`
  - Response format: **File** (binary output)
- **Inputs/Outputs:**  
  Input from **Get Instagram Media**; output to **Prepare YouTube Upload Data**.
- **Edge cases / failures:**
  - `media_url` may be temporary/expired; download can fail.
  - Large file sizes may hit memory/time limits depending on n8n settings.
  - Some URLs may require headers/cookies; not handled here.

---

### Block 3 — YouTube Upload Preparation & AI Metadata Optimization

**Overview:**  
Builds baseline upload fields (title/description/media URL) and uses Claude (via LangChain agent) to generate SEO-optimized YouTube metadata as structured JSON.

**Nodes involved:**  
- Prepare YouTube Upload Data  
- Generate Video Metadata  
- Anthropic Chat Model  
- Structured Output Parser

#### Node: “Prepare YouTube Upload Data”
- **Type / role:** `Set` — normalizes fields for the AI step.
- **Configuration choices:**
  - `videoTitle` = `caption` OR fallback `Workflow Configuration.youtubeTitle`
    - Expression: `={{ $json.caption || $('Workflow Configuration').first().json.youtubeTitle }}`
  - `videoDescription` = `caption` OR fallback `Workflow Configuration.youtubeDescription`
    - Expression: `={{ $json.caption || $('Workflow Configuration').first().json.youtubeDescription }}`
  - `mediaUrl` = `={{ $json.media_url }}`
  - Include Other Fields = true
- **Inputs/Outputs:**  
  Input from **Download Instagram Video**; output to **Generate Video Metadata**.
- **Edge cases / failures:**
  - Uses caption for both title and description; may be undesired if caption is long or not descriptive.
  - If caption is missing and placeholders weren’t updated, title/description become unusable placeholders.

#### Node: “Generate Video Metadata”
- **Type / role:** `LangChain Agent` — prompts the model to generate optimized metadata and returns structured output.
- **Configuration choices:**
  - Prompt (expression):  
    `Generate optimized metadata... Original title: {{ $json.videoTitle }}. Original description: {{ $json.videoDescription }}`
  - System message: instructs YouTube SEO optimization, char limits, return JSON.
  - `hasOutputParser = true` (expects structured parsing).
- **Model & parser wiring:**
  - Uses **Anthropic Chat Model** via `ai_languageModel` connection.
  - Uses **Structured Output Parser** via `ai_outputParser` connection.
- **Outputs:** Main output to **Upload to YouTube**.
- **Edge cases / failures:**
  - Model returns non-JSON or schema-incompatible output → parser fails.
  - Prompt does not explicitly force exact JSON-only output; parser may still fail if Claude adds prose.
  - Rate limits / auth issues on Anthropic credentials.

#### Node: “Anthropic Chat Model”
- **Type / role:** `LangChain LLM (Anthropic Chat)` — provides Claude to the agent.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929` (as selected in node)
- **Credentials:** `Anthropic account`
- **Outputs/Connections:** Connected to **Generate Video Metadata** on `ai_languageModel`.
- **Edge cases / failures:**
  - Invalid API key, insufficient quota, model not available in region/account.

#### Node: “Structured Output Parser”
- **Type / role:** `Structured Output Parser` — validates/parses model output into fields.
- **Schema (manual):**
  - `optimizedTitle` (string)
  - `optimizedDescription` (string)
  - `tags` (array of strings)
- **Outputs/Connections:** Connected to **Generate Video Metadata** on `ai_outputParser`.
- **Edge cases / failures:**
  - If any field missing or wrong type, parsing fails.
  - Schema does not require fields (no `"required"`), but many parsers still expect the keys; behavior can vary by node version.

---

### Block 4 — Publishing, Post-upload Analysis, Tracking & Notification

**Overview:**  
Uploads the video to YouTube with AI-optimized metadata, generates a structured “performance” assessment, logs the result to Google Sheets, and sends a WhatsApp completion message.

**Nodes involved:**  
- Upload to YouTube  
- Analyze Video Performance  
- Anthropic Chat Model1  
- Structured Output Parser1  
- Log to Google Sheets  
- Send WhatsApp Notification

#### Node: “Upload to YouTube”
- **Type / role:** `YouTube` — uploads a video.
- **Configuration choices:**
  - Operation: `video → upload`
  - Title: `={{ $json.optimizedTitle }}`
  - Description (option): `={{ $json.optimizedDescription }}`
  - Privacy: `private`
- **Inputs/Outputs:**  
  Input from **Generate Video Metadata**; output to **Analyze Video Performance**.
- **Important integration note:**  
  This node typically requires a **binary video file input** (from Download Instagram Video). In the current wiring, the upload node receives data from the AI node; unless the binary data is still present and correctly passed through, the upload may fail due to missing binary property. The workflow does not explicitly map or reference the binary.
- **Edge cases / failures:**
  - Missing/incorrect YouTube OAuth credentials.
  - Upload fails if binary not present, file too large, unsupported codec/container.
  - API quota limits and upload rate limiting.
  - Title/description constraints violated (length, invalid characters).

#### Node: “Analyze Video Performance”
- **Type / role:** `LangChain Agent` — creates an initial analysis/prediction from upload data.
- **Configuration choices:**
  - Prompt (expression):  
    `Analyze the uploaded video performance data. Video ID: {{ $json.id }}. Title: {{ $json.title }}`
  - System message: asks for predictions, optimization opportunities, tracking metrics; return JSON.
  - `hasOutputParser = true`
- **Model & parser wiring:**
  - **Anthropic Chat Model1** connected on `ai_languageModel`
  - **Structured Output Parser1** connected on `ai_outputParser`
- **Outputs:** Main output to **Log to Google Sheets**.
- **Edge cases / failures:**
  - Upload node may not output `id` / `title` in expected shape depending on YouTube node version/response.
  - Parser failures if model returns non-conforming output.

#### Node: “Anthropic Chat Model1”
- **Type / role:** `LangChain LLM (Anthropic Chat)` — Claude model for analysis step.
- **Configuration choices:** same model selection: `claude-sonnet-4-5-20250929`
- **Credentials:** `Anthropic account`
- **Outputs/Connections:** Connected to **Analyze Video Performance** on `ai_languageModel`.
- **Edge cases / failures:** same as other Anthropic model node.

#### Node: “Structured Output Parser1”
- **Type / role:** `Structured Output Parser` — parses the analysis output.
- **Schema (manual):**
  - `performanceScore` (number)
  - `recommendations` (array of strings)
  - `targetAudience` (string)
  - `bestPostingTime` (string)
- **Outputs/Connections:** Connected to **Analyze Video Performance** on `ai_outputParser`.
- **Edge cases / failures:** schema mismatches, missing keys, type issues.

#### Node: “Log to Google Sheets”
- **Type / role:** `Google Sheets` — append or update tracking row.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Document ID: from config  
    `={{ $('Workflow Configuration').first().json.googleSheetId }}`
  - Sheet name: `Sheet1`
  - Mapping mode: `autoMapInputData`
  - Matching columns: `videoId`
- **Credentials:** `Google Sheets OAuth2` (“Google Sheets account”)
- **Inputs/Outputs:**  
  Input from **Analyze Video Performance**; output to **Send WhatsApp Notification**.
- **Critical edge case:**  
  The workflow sets matching column as `videoId`, but upstream nodes reference YouTube’s ID as `id` in the analysis prompt. Unless a `videoId` field is created somewhere (it isn’t explicitly), append/update matching may not work as intended and may create duplicates or fail to match.
- **Edge cases / failures:**
  - Sheet permissions missing; wrong spreadsheet ID.
  - Sheet/tab name mismatch (`Sheet1` not present).
  - Column headers not aligned with incoming keys; automap can silently omit fields.
  - AppendOrUpdate requires the matching column to exist in the sheet.

#### Node: “Send WhatsApp Notification”
- **Type / role:** `WhatsApp` — sends completion message.
- **Configuration choices:**
  - Operation: `send`
  - Recipient: `={{ $('Workflow Configuration').first().json.whatsappPhoneNumber }}`
  - Message body (expression): includes:
    - `optimizedTitle`
    - `videoId`
    - `performanceScore`
    - `recommendations.join('\n- ')`
- **Credentials:** `WhatsApp account`
- **Inputs/Outputs:**  
  Input from **Log to Google Sheets**; terminal node (no outgoing connection).
- **Edge cases / failures:**
  - If `recommendations` is missing or not an array, `.join()` expression fails.
  - References `videoId` which may not exist (as noted above).
  - WhatsApp Business API template/format constraints depending on provider; node may require approved templates in some setups.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Initiates the workflow at preset interval | Schedule Trigger | Time-based entry point | — | Workflow Configuration | ## Content ingestion and preparation / **What:** A schedule trigger initiates the workflow at preset intervals, fetches Instagram media / **Why:** Automation removes manual content transfer effort. |
| Workflow Configuration | Set | Stores config/defaults for reuse | Initiates the workflow at preset interval | Get Instagram Media | ## How It Works / This workflow automates cross-platform content distribution… retrieves Instagram videos… generates metadata… uploads… logs… sends WhatsApp… (dual-model approach claim). |
| Get Instagram Media | HTTP Request | Fetch Instagram media metadata | Workflow Configuration | Download Instagram Video | ## Content ingestion and preparation / **What:** …fetches Instagram media / **Why:** Automation removes manual content transfer effort. |
| Download Instagram Video | HTTP Request | Download media as binary file | Get Instagram Media | Prepare YouTube Upload Data | ## Content ingestion and preparation / **What:** … / **Why:** Automation removes manual content transfer effort. |
| Prepare YouTube Upload Data | Set | Build base title/description/mediaUrl fields | Download Instagram Video | Generate Video Metadata | ## Setup Steps / 1. Configure Instagram credentials / 2. Add Anthropic API key… / 3. Connect YouTube… / 4. Link Google Sheets… / 5. Add WhatsApp Business API credentials |
| Generate Video Metadata | LangChain Agent | Create SEO-optimized title/description/tags (structured) | Prepare YouTube Upload Data | Upload to YouTube | ## Prerequisites / Instagram Business/Creator account with API access / ## Use Cases / Social media agencies… / ## Customization / Modify AI prompts… / ## Benefits / Saves 2-3 hours daily… |
| Anthropic Chat Model | Anthropic Chat Model (LangChain) | LLM provider for metadata agent | — (AI connection) | Generate Video Metadata (ai_languageModel) | ## Setup Steps / 1…5… |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforces JSON schema for metadata | — (AI connection) | Generate Video Metadata (ai_outputParser) | ## Setup Steps / 1…5… |
| Upload to YouTube | YouTube | Upload video with optimized metadata | Generate Video Metadata | Analyze Video Performance | ## Publishing, tracking, and notification / **What:** The workflow uploads processed videos to YouTube / **Why:** Automated publishing ensures consistent posting schedules. |
| Analyze Video Performance | LangChain Agent | Produce initial performance predictions & recommendations | Upload to YouTube | Log to Google Sheets | ## Publishing, tracking, and notification / **What:** … / **Why:** … |
| Anthropic Chat Model1 | Anthropic Chat Model (LangChain) | LLM provider for performance agent | — (AI connection) | Analyze Video Performance (ai_languageModel) | ## Publishing, tracking, and notification / **What:** … / **Why:** … |
| Structured Output Parser1 | Structured Output Parser (LangChain) | Enforces JSON schema for performance analysis | — (AI connection) | Analyze Video Performance (ai_outputParser) | ## Publishing, tracking, and notification / **What:** … / **Why:** … |
| Log to Google Sheets | Google Sheets | Append/update tracking row | Analyze Video Performance | Send WhatsApp Notification | ## Publishing, tracking, and notification / **What:** … / **Why:** … |
| Send WhatsApp Notification | WhatsApp | Notify completion and key metrics | Log to Google Sheets | — | ## Publishing, tracking, and notification / **What:** … / **Why:** … |
| Sticky Note | Sticky Note | Documentation | — | — | ## Prerequisites… / Use Cases… / Customization… / Benefits… |
| Sticky Note1 | Sticky Note | Documentation | — | — | ## Setup Steps / 1…5… |
| Sticky Note2 | Sticky Note | Documentation | — | — | ## How It Works (long description) |
| Sticky Note4 | Sticky Note | Documentation | — | — | ## Publishing, tracking, and notification (what/why) |
| Sticky Note5 | Sticky Note | Documentation | — | — | ## Content ingestion and preparation (what/why) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Intelligent Instagram to YouTube upload with Google Sheets tracking”

2. **Add Trigger**
   1. Add node: **Schedule Trigger**
   2. Set rule to run **daily at 09:00** (adjust timezone if needed).

3. **Add configuration node**
   1. Add node: **Set**; name it **Workflow Configuration**
   2. Add string fields:
      - `youtubeTitle` (default fallback title)
      - `youtubeDescription` (default fallback description)
      - `googleSheetId`
      - `instagramMediaId`
      - `instagramAccessToken`
      - `whatsappPhoneNumber`
   3. Enable **Include Other Fields**.
   4. Connect: **Schedule Trigger → Workflow Configuration**

4. **Fetch Instagram media metadata**
   1. Add node: **HTTP Request**; name: **Get Instagram Media**
   2. Method: GET
   3. URL (expression):  
      `https://graph.instagram.com/{{ $('Workflow Configuration').first().json.instagramMediaId }}?fields=media_url,media_type,caption&access_token={{ $('Workflow Configuration').first().json.instagramAccessToken }}`
   4. Response format: **JSON**
   5. Connect: **Workflow Configuration → Get Instagram Media**

5. **Download the Instagram video**
   1. Add node: **HTTP Request**; name: **Download Instagram Video**
   2. URL (expression): `={{ $json.media_url }}`
   3. Response format: **File**
   4. Connect: **Get Instagram Media → Download Instagram Video**

6. **Prepare fields for AI metadata generation**
   1. Add node: **Set**; name: **Prepare YouTube Upload Data**
   2. Add fields:
      - `videoTitle` = `={{ $json.caption || $('Workflow Configuration').first().json.youtubeTitle }}`
      - `videoDescription` = `={{ $json.caption || $('Workflow Configuration').first().json.youtubeDescription }}`
      - `mediaUrl` = `={{ $json.media_url }}`
   3. Enable **Include Other Fields**
   4. Connect: **Download Instagram Video → Prepare YouTube Upload Data**

7. **Add AI metadata agent (Claude)**
   1. Add node: **LangChain Agent**; name: **Generate Video Metadata**
   2. Set Prompt/Text (expression):  
      `Generate optimized metadata for this video. Original title: {{ $json.videoTitle }}. Original description: {{ $json.videoDescription }}`
   3. Set **System message** to instruct SEO optimization and require JSON output (as in the workflow).
   4. Enable **Structured output** / **hasOutputParser** (node option).

8. **Add Claude chat model for the agent**
   1. Add node: **Anthropic Chat Model**; name: **Anthropic Chat Model**
   2. Select model: `claude-sonnet-4-5-20250929` (or closest available)
   3. Configure **Anthropic API credentials** (API key) in n8n.
   4. Connect model to agent using the **AI Language Model** connection:  
      **Anthropic Chat Model → Generate Video Metadata (ai_languageModel)**

9. **Add structured output parser for metadata**
   1. Add node: **Structured Output Parser**
   2. Schema (manual) with:
      - `optimizedTitle` string
      - `optimizedDescription` string
      - `tags` array of strings
   3. Connect parser to agent using **AI Output Parser** connection:  
      **Structured Output Parser → Generate Video Metadata (ai_outputParser)**

10. **Upload to YouTube**
   1. Add node: **YouTube**; name: **Upload to YouTube**
   2. Resource: `video`, Operation: `upload`
   3. Title: `={{ $json.optimizedTitle }}`
   4. Options → Description: `={{ $json.optimizedDescription }}`
   5. Options → Privacy status: `private`
   6. Configure **YouTube OAuth2 credentials** in n8n and select them in the node.
   7. Connect: **Generate Video Metadata → Upload to YouTube**
   8. **Important:** Ensure the node receives the **binary video file** from the download step. If needed, add a **Merge** node (by key/index) or rewire so the binary from “Download Instagram Video” is preserved into the upload node.

11. **Add post-upload performance analysis agent**
   1. Add node: **LangChain Agent**; name: **Analyze Video Performance**
   2. Text (expression):  
      `Analyze the uploaded video performance data. Video ID: {{ $json.id }}. Title: {{ $json.title }}`
   3. Enable structured output parsing.

12. **Add Claude model and parser for analysis**
   1. Add node: **Anthropic Chat Model**; name: **Anthropic Chat Model1**  
      - Same credentials/model selection
   2. Add node: **Structured Output Parser**; name: **Structured Output Parser1** with schema:
      - `performanceScore` number
      - `recommendations` array of strings
      - `targetAudience` string
      - `bestPostingTime` string
   3. Connect:
      - **Anthropic Chat Model1 → Analyze Video Performance (ai_languageModel)**
      - **Structured Output Parser1 → Analyze Video Performance (ai_outputParser)**
   4. Connect: **Upload to YouTube → Analyze Video Performance**

13. **Log results to Google Sheets**
   1. Add node: **Google Sheets**; name: **Log to Google Sheets**
   2. Operation: `Append or Update`
   3. Document ID (expression):  
      `={{ $('Workflow Configuration').first().json.googleSheetId }}`
   4. Sheet name: `Sheet1`
   5. Matching column: `videoId`
   6. Mapping: Auto-map input data
   7. Configure **Google Sheets OAuth2 credentials** in n8n.
   8. Connect: **Analyze Video Performance → Log to Google Sheets**
   9. Ensure your incoming data includes `videoId` (or change matching column to `id` and ensure the sheet has an `id` column header).

14. **Send WhatsApp notification**
   1. Add node: **WhatsApp**; name: **Send WhatsApp Notification**
   2. Operation: `send`
   3. Recipient phone number (expression):  
      `={{ $('Workflow Configuration').first().json.whatsappPhoneNumber }}`
   4. Message body (expression): include title, ID, score, recommendations (as per workflow).
   5. Configure **WhatsApp Business API credentials** in n8n.
   6. Connect: **Log to Google Sheets → Send WhatsApp Notification**

15. **(Optional) Add sticky notes** with the provided content for operator context.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| **Prerequisites:** Instagram Business/Creator account with API access | Sticky note “Prerequisites” |
| **Use Cases:** Social media agencies managing multiple client accounts | Sticky note “Prerequisites” |
| **Customization:** Modify AI prompts for brand-specific tone, adjust scheduling frequency | Sticky note “Prerequisites” |
| **Benefits:** Saves 2-3 hours daily on manual uploads, ensures consistent posting schedules | Sticky note “Prerequisites” |
| **Setup Steps:** 1) Configure Instagram credentials 2) Add Anthropic API key 3) Connect YouTube 4) Link Google Sheets 5) Add WhatsApp Business API credentials | Sticky note “Setup Steps” |
| **How It Works:** end-to-end automation from Instagram retrieval → AI metadata → YouTube upload → Sheets logging → WhatsApp notification; mentions a “dual-model approach” but the current workflow uses Claude for both AI steps | Sticky note “How It Works” |
| **Publishing, tracking, and notification:** Automated publishing ensures consistent posting schedules | Sticky note “Publishing, tracking, and notification” |
| **Content ingestion and preparation:** Schedule trigger + fetch Instagram media to remove manual transfer | Sticky note “Content ingestion and preparation” |