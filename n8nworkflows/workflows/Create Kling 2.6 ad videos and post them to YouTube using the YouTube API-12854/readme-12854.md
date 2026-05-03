Create Kling 2.6 ad videos and post them to YouTube using the YouTube API

https://n8nworkflows.xyz/workflows/create-kling-2-6-ad-videos-and-post-them-to-youtube-using-the-youtube-api-12854


# Create Kling 2.6 ad videos and post them to YouTube using the YouTube API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow generates ad-style videos using **Kling 2.6 via FAL.AI**, prepares **YouTube SEO metadata using an AI agent**, then **uploads the video to YouTube** through the YouTube API. It is designed for a user to submit a prompt, automatically produce a video, wait for generation completion, fetch the resulting video asset, and publish it.

**Main logical blocks**
1. **1.1 Prompt Intake (Manual Entry Point)**  
   Collects the initial creative prompt from a Form trigger.
2. **1.2 Video Prompting / Creative Direction (LLM chain)**  
   Uses an OpenRouter chat model to generate videography instructions/structured prompt content for Kling.
3. **1.3 Video Generation Request (FAL.AI HTTP)**  
   Sends a generation request to FAL.AI (Kling 2.6) and stores returned job data.
4. **1.4 Persistence + Wait for Completion**  
   Stores job/metadata in Google Sheets, then waits 5 minutes to allow generation to finish.
5. **1.5 YouTube SEO (AI Agent + Structured Output)**  
   AI agent generates title/description/tags; structured output parser enforces a schema-like output.
6. **1.6 Keywords Extraction + Video Retrieval**  
   Code node extracts keywords and prepares downstream fields; HTTP calls fetch credentials/URLs and download the video.
7. **1.7 Publish to YouTube**  
   Uploads the downloaded binary video to YouTube.

---

## 2. Block-by-Block Analysis

### 2.1 Prompt Intake (Manual Entry Point)

**Overview:** Starts the workflow when a user submits a prompt via an n8n Form. This is the single explicit entry point.

**Nodes involved:**
- **Type Prompt** (Form Trigger)

**Node details**
- **Type Prompt**
  - **Type / role:** `n8n-nodes-base.formTrigger` — interactive trigger to collect user input.
  - **Configuration (interpreted):** Uses an n8n-hosted form endpoint (webhook-backed). In this exported JSON, the form fields are not shown (parameters are empty), so fields must be configured in the UI.
  - **Key variables:** Output depends on form fields; typically you’d expect something like `{{$json.prompt}}` or similar.
  - **Connections:**
    - **Output →** `Videography` (main)
  - **Edge cases / failures:**
    - Missing expected field names will break downstream prompt building.
    - Multiple submissions create multiple parallel runs (ensure idempotency if needed).
  - **Version notes:** TypeVersion `2.2` (Form trigger behavior differs slightly across n8n versions; confirm field schema export/import).

---

### 2.2 Video Prompting / Creative Direction (LLM chain)

**Overview:** Transforms the raw user prompt into a higher-quality “videography” prompt suitable for Kling/FAL.AI, using OpenRouter as the LLM backend.

**Nodes involved:**
- **AI_Brain** (OpenRouter Chat Model)
- **Videography** (LangChain LLM Chain)

**Node details**
- **AI_Brain**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — provides a chat model via OpenRouter for downstream LangChain nodes.
  - **Configuration:** Not visible in JSON (empty parameters). In practice: model name (e.g., a GPT-like model), temperature, max tokens, and OpenRouter credential.
  - **Connections:**
    - **AI languageModel →** `Videography`
  - **Edge cases / failures:**
    - Missing/invalid OpenRouter API key.
    - Model not available or rate-limited.
    - Token limits if prompt is long.

- **Videography**
  - **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — an LLM “chain” to produce a composed output (often a refined prompt + parameters).
  - **Configuration:** Not visible (empty parameters). Expected to include a prompt template that uses form inputs.
  - **Key expressions:** Typically references form fields like `{{$json.<field>}}`.
  - **Connections:**
    - **Input:** `Type Prompt`
    - **LLM:** `AI_Brain`
    - **Output →** `Make FAL.AI Request`
  - **Edge cases / failures:**
    - If the chain output isn’t aligned with what the FAL.AI request expects (missing fields, wrong structure), the HTTP request may fail downstream.

---

### 2.3 Video Generation Request (FAL.AI HTTP)

**Overview:** Sends a request to FAL.AI to generate a Kling 2.6 video based on the videography prompt. The response usually includes a job ID or a result URL, which must be persisted for later retrieval.

**Nodes involved:**
- **Make FAL.AI Request** (HTTP Request)

**Node details**
- **Make FAL.AI Request**
  - **Type / role:** `n8n-nodes-base.httpRequest` — calls FAL.AI API endpoint to start generation.
  - **Configuration:** Not shown (empty parameters). Typically includes:
    - Method: `POST`
    - URL: FAL.AI Kling endpoint
    - Auth header (FAL key) or credential
    - Body: prompt/settings from `Videography` output
  - **Connections:**
    - **Input:** `Videography`
    - **Output →** `Store Data`
  - **Edge cases / failures:**
    - 401/403 if FAL.AI credentials missing/invalid.
    - 429 rate limits.
    - 400 if payload schema doesn’t match the model endpoint requirements.
    - Long-running jobs: response may only be a job handle; requires polling or delayed fetch (handled later with wait + credential fetch).

---

### 2.4 Persistence + Wait for Completion

**Overview:** Stores generation metadata (likely job ID, prompt, timestamps) to Google Sheets, then waits 5 minutes before continuing to SEO + retrieval steps.

**Nodes involved:**
- **Store Data** (Google Sheets)
- **Wait 5 mins** (Wait)

**Node details**
- **Store Data**
  - **Type / role:** `n8n-nodes-base.googleSheets` — persists data in a spreadsheet (logging / job tracking).
  - **Configuration:** Not shown. Typically requires:
    - Spreadsheet ID, Sheet name/range
    - Operation: append row / update row
    - Mapping columns to values from FAL response and prompt
  - **Connections:**
    - **Input:** `Make FAL.AI Request`
    - **Output →** `Wait 5 mins`
  - **Edge cases / failures:**
    - Google auth expired/invalid.
    - Sheet/range mismatch.
    - Concurrency issues (multiple runs appending simultaneously).
  - **Version notes:** TypeVersion `4.5` (ensure compatibility with your n8n version; Google node has had multiple auth modes).

- **Wait 5 mins**
  - **Type / role:** `n8n-nodes-base.wait` — pauses execution to allow async video generation to complete.
  - **Configuration:** Not shown, but implied 5 minutes by node name.
  - **Connections:**
    - **Input:** `Store Data`
    - **Output →** `YT Video SEO`
  - **Edge cases / failures:**
    - If generation takes longer than 5 minutes, subsequent fetch/download may fail (404/not-ready).
    - If n8n restarts, Wait nodes depend on execution persistence settings.

---

### 2.5 YouTube SEO (AI Agent + Structured Output)

**Overview:** Generates YouTube upload metadata (title/description/tags/category/hashtags, etc.) using an AI agent powered by OpenRouter. Structured Output enforces a machine-parseable format.

**Nodes involved:**
- **AI Brain** (OpenRouter Chat Model)
- **Structured Output** (LangChain structured output parser)
- **YT Video SEO** (LangChain agent)

**Node details**
- **AI Brain**
  - **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — LLM provider for the SEO agent.
  - **Connections:**
    - **AI languageModel →** `YT Video SEO`
  - **Edge cases:** Same as other OpenRouter model node (auth, rate limits, model availability).

- **Structured Output**
  - **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — forces the agent’s response into a defined structure (schema).
  - **Configuration:** Not shown; in UI this typically defines JSON schema fields like `title`, `description`, `tags`, etc.
  - **Connections:**
    - **AI outputParser →** `YT Video SEO`
  - **Edge cases / failures:**
    - Parser errors if the agent returns non-conforming text/JSON.
    - If schema is too strict, minor formatting deviations will fail the run.

- **YT Video SEO**
  - **Type / role:** `@n8n/n8n-nodes-langchain.agent` — AI agent that produces SEO metadata.
  - **Configuration (interpreted):**
    - Has **retryOnFail = true**, **maxTries = 2**, **waitBetweenTries = 3000ms**.
    - Uses the attached `AI Brain` and `Structured Output`.
  - **Connections:**
    - **Input:** `Wait 5 mins`
    - **Output →** `Get Keywords`
  - **Edge cases / failures:**
    - Still may fail after retries if output parser rejects results.
    - Prompt injection risk if user prompt is used verbatim; consider sanitization and strict schema.

---

### 2.6 Keywords Extraction + Video Retrieval

**Overview:** Extracts keywords/tags from the SEO output, then calls an endpoint to retrieve video credentials/URLs and downloads the generated video.

**Nodes involved:**
- **Get Keywords** (Code)
- **Fetch Video Credentials** (HTTP Request)
- **Download Video** (HTTP Request)

**Node details**
- **Get Keywords**
  - **Type / role:** `n8n-nodes-base.code` — transforms the SEO agent output into fields needed for upload and/or retrieval.
  - **Configuration:** Not shown. Typically:
    - Parse structured SEO output fields
    - Build YouTube tags array, title, description
    - Potentially carry forward FAL job ID / output URL
  - **Connections:**
    - **Input:** `YT Video SEO`
    - **Output →** `Fetch Video Credentials`
  - **Edge cases / failures:**
    - JavaScript errors if expected fields don’t exist.
    - If Structured Output schema changes, this code must be updated.
  - **Version notes:** TypeVersion `2` (n8n Code node runtime differs from older Function node).

- **Fetch Video Credentials**
  - **Type / role:** `n8n-nodes-base.httpRequest` — retrieves a signed URL / final file URL for the generated video (or polls job status).
  - **Configuration:** Not visible. Common patterns:
    - Call FAL.AI “get result” endpoint with job ID
    - Retrieve `video_url` or signed download URL
  - **Connections:**
    - **Input:** `Get Keywords`
    - **Output →** `Download Video`
  - **Edge cases / failures:**
    - If job not finished yet, API may return “processing” state.
    - Signed URLs can expire; downloading must happen quickly after retrieval.

- **Download Video**
  - **Type / role:** `n8n-nodes-base.httpRequest` — downloads the video file as binary data for YouTube upload.
  - **Configuration:** Not shown, but should be set to:
    - GET the `video_url`
    - “Download” / “Response: File” mode to produce binary output
    - Proper binary property name (e.g., `data`)
  - **Connections:**
    - **Input:** `Fetch Video Credentials`
    - **Output →** `Post on YouTube`
  - **Edge cases / failures:**
    - Large file download timeouts.
    - 403/404 if signed URL expired or not accessible.
    - Incorrect binary property name will cause YouTube upload to fail.

---

### 2.7 Publish to YouTube

**Overview:** Uploads the downloaded video to YouTube and applies the generated SEO metadata.

**Nodes involved:**
- **Post on YouTube** (YouTube)

**Node details**
- **Post on YouTube**
  - **Type / role:** `n8n-nodes-base.youTube` — uploads a video via YouTube Data API.
  - **Configuration:** Not visible. Typically includes:
    - Operation: Upload video
    - Binary property containing the file from `Download Video`
    - Title/Description/Tags from `Get Keywords`
    - Privacy status (public/unlisted/private)
  - **Connections:**
    - **Input:** `Download Video`
    - **Output:** none
  - **Edge cases / failures:**
    - OAuth scope/consent missing for upload.
    - Upload quotas exceeded.
    - Tags must be an array of strings; title length limits; description length limits.
  - **Version notes:** TypeVersion `1` (confirm current YouTube node supports your desired metadata fields).

---

### 2.8 Notes / Comments (Sticky Notes)

**Overview:** A single sticky note exists but contains no content.

**Nodes involved:**
- **Sticky Note1** (Sticky Note)

**Node details**
- **Sticky Note1**
  - **Type / role:** `n8n-nodes-base.stickyNote` — canvas annotation.
  - **Content:** *(empty)*
  - **Connections:** none
  - **Edge cases:** none

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Type Prompt | Form Trigger | Manual prompt intake / workflow entry | — | Videography |  |
| AI_Brain | OpenRouter Chat Model | LLM backend for videography chain | — | Videography (AI languageModel) |  |
| Videography | LangChain LLM Chain | Generate refined Kling/videography prompt | Type Prompt; AI_Brain | Make FAL.AI Request |  |
| Make FAL.AI Request | HTTP Request | Start Kling video generation on FAL.AI | Videography | Store Data |  |
| Store Data | Google Sheets | Persist job metadata/logging | Make FAL.AI Request | Wait 5 mins |  |
| Wait 5 mins | Wait | Delay to allow async rendering | Store Data | YT Video SEO |  |
| AI Brain | OpenRouter Chat Model | LLM backend for YouTube SEO agent | — | YT Video SEO (AI languageModel) |  |
| Structured Output | Structured Output Parser | Enforce schema for SEO output | — | YT Video SEO (AI outputParser) |  |
| YT Video SEO | LangChain Agent | Generate YouTube title/description/tags | Wait 5 mins; AI Brain; Structured Output | Get Keywords |  |
| Get Keywords | Code | Normalize/derive keywords & fields for upload/retrieval | YT Video SEO | Fetch Video Credentials |  |
| Fetch Video Credentials | HTTP Request | Retrieve final video URL / signed URL from provider | Get Keywords | Download Video |  |
| Download Video | HTTP Request | Download video as binary for upload | Fetch Video Credentials | Post on YouTube |  |
| Post on YouTube | YouTube | Upload video + apply SEO metadata | Download Video | — |  |
| Sticky Note1 | Sticky Note | Canvas annotation | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “Type Prompt” node**
   - Node type: **Form Trigger**
   - Add at least one form field, e.g. **prompt** (text area).
   - Save the form and note the generated form URL (optional for distribution).

2. **Create OpenRouter model for videography**
   - Node type: **OpenRouter Chat Model** (`lmChatOpenRouter`)
   - Configure credentials: **OpenRouter API key**
   - Choose a chat model (your preference) and set temperature/max tokens as desired.
   - Name it **AI_Brain**.

3. **Create “Videography” chain**
   - Node type: **LangChain LLM Chain** (`chainLlm`)
   - Connect:
     - **Type Prompt → Videography** (main)
     - **AI_Brain → Videography** (AI languageModel)
   - In the chain prompt template, reference the form field, e.g. `{{$json.prompt}}`, and instruct the model to output a Kling-ready prompt (optionally include duration, style, camera movement, etc.).

4. **Create “Make FAL.AI Request”**
   - Node type: **HTTP Request**
   - Method: `POST`
   - URL: your FAL.AI Kling 2.6 endpoint
   - Auth: Header bearer token (FAL key) or credential method you use
   - Body: map fields from **Videography** output (prompt + Kling parameters)
   - Connect **Videography → Make FAL.AI Request**.

5. **Create “Store Data” in Google Sheets**
   - Node type: **Google Sheets**
   - Credentials: Google OAuth2 / Service Account (whichever you use)
   - Operation: usually **Append** row
   - Map columns for at least: prompt, job_id, created_at, status (based on FAL response)
   - Connect **Make FAL.AI Request → Store Data**.

6. **Create “Wait 5 mins”**
   - Node type: **Wait**
   - Mode: time-based delay = **5 minutes**
   - Connect **Store Data → Wait 5 mins**.

7. **Create OpenRouter model for SEO**
   - Node type: **OpenRouter Chat Model**
   - Configure OpenRouter credentials
   - Name it **AI Brain** (distinct from AI_Brain).

8. **Create “Structured Output”**
   - Node type: **Structured Output Parser**
   - Define schema fields you want, for example:
     - `title` (string)
     - `description` (string)
     - `tags` (array of strings)
     - (optional) `hashtags` (array), `categoryId` (string), `language` (string)
   - This schema must match what downstream Code/YouTube expects.

9. **Create “YT Video SEO” agent**
   - Node type: **AI Agent**
   - Connect:
     - **Wait 5 mins → YT Video SEO** (main)
     - **AI Brain → YT Video SEO** (AI languageModel)
     - **Structured Output → YT Video SEO** (AI outputParser)
   - In agent instructions: ask for YouTube metadata for the generated ad video.
   - Set retries:
     - Enable retry on fail
     - Max tries: 2
     - Wait between tries: 3000 ms

10. **Create “Get Keywords”**
    - Node type: **Code**
    - Implement logic to:
      - Read structured SEO fields (title/description/tags)
      - Normalize tags to YouTube-compatible format
      - Carry forward job_id/result handles from earlier steps (ensure they are still present in the item)
    - Connect **YT Video SEO → Get Keywords**.

11. **Create “Fetch Video Credentials”**
    - Node type: **HTTP Request**
    - Call the “get result” / “status” endpoint for the FAL job using job_id.
    - Extract final `video_url` (or signed URL).
    - Connect **Get Keywords → Fetch Video Credentials**.

12. **Create “Download Video”**
    - Node type: **HTTP Request**
    - Method: `GET` to the retrieved `video_url`
    - Enable binary download (store response as **binary**)
    - Set binary property name (e.g. `data`)
    - Connect **Fetch Video Credentials → Download Video**.

13. **Create “Post on YouTube”**
    - Node type: **YouTube**
    - Credentials: YouTube OAuth2 with scopes for upload (YouTube Data API)
    - Operation: **Upload video**
    - Video binary property: match the one from **Download Video** (e.g. `data`)
    - Map metadata fields:
      - Title from Code output
      - Description from Code output
      - Tags array from Code output
      - Privacy status as desired
    - Connect **Download Video → Post on YouTube**.

14. **(Optional) Add a Sticky Note**
    - Node type: **Sticky Note**
    - In this workflow it is present but empty; you can omit it.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky Note1 is empty. | No additional operational notes were provided inside the workflow canvas. |
| The workflow relies on external services (OpenRouter, FAL.AI, Google Sheets, YouTube API). Ensure credentials and quota limits are configured. | OpenRouter + FAL.AI API keys; Google OAuth; YouTube Data API OAuth + quota. |
| The 5-minute wait is a fixed delay and may be insufficient for longer renders. Consider replacing with polling (loop until status=completed) for reliability. | Reliability improvement suggestion for async rendering workflows. |