Convert blog posts to podcast episodes with GPT-4o, ElevenLabs & Google Drive

https://n8nworkflows.xyz/workflows/convert-blog-posts-to-podcast-episodes-with-gpt-4o--elevenlabs---google-drive-11897


# Convert blog posts to podcast episodes with GPT-4o, ElevenLabs & Google Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Convert newly published blog posts (via an RSS feed) into podcast episodes by rewriting the post into a spoken script with an AI agent (Azure OpenAI GPT model), generating audio via an HTTP call (typically ElevenLabs), uploading the resulting audio to Google Drive, and updating an RSS podcast feed, then notifying a team in Slack. A separate error workflow path emails on failures.

**Typical use cases**
- Automated “blog-to-podcast” repurposing pipeline
- Daily/weekly podcast episode generation from a CMS RSS feed
- Keeping an RSS podcast feed updated with newly generated episodes

### 1.1 Triggering & Content Intake
Runs on a schedule, reads an RSS feed, then prepares to iterate over new items.

### 1.2 Iteration & Script Generation (AI)
Loops through RSS items and uses an AI Agent (connected to an Azure OpenAI Chat Model and a Structured Output Parser) to rewrite each blog post into a podcast-ready script.

### 1.3 Audio Rendering & Storage
Formats the script for TTS, calls an external TTS provider via HTTP to generate audio, then uploads the audio to Google Drive.

### 1.4 Podcast Feed Update & Notification
Formats metadata for RSS, builds the podcast feed entry, updates the RSS file, then notifies the team via Slack.

### 1.5 Error Handling
On workflow errors, triggers an error handler and sends an email via Gmail.

---

## 2. Block-by-Block Analysis

### Block 1 — Triggering & RSS Intake
**Overview:** Starts on a schedule, then reads the RSS feed to retrieve blog post items that will be processed into episodes.

**Nodes involved:**
- Schedule Trigger
- Trigger: New Blog Post
- Loop Over Items

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — workflow entry point on a time schedule.
- **Configuration (interpreted):** No schedule parameters are shown in the JSON snippet (likely default/placeholder). In practice, you must set interval (e.g., every day/hour).
- **Connections:** Outputs to **Trigger: New Blog Post**.
- **Edge cases / failures:**
  - Misconfigured schedule => never runs or runs too often.
  - Timezone mismatch between server and intended schedule.

#### Node: Trigger: New Blog Post
- **Type / role:** `RSS Feed Read` — fetches items from an RSS feed.
- **Configuration (interpreted):** Parameters are empty in provided JSON; you must set the RSS URL and (optionally) “Return All” / “Limit”.
- **Connections:** Outputs to **Loop Over Items**.
- **Edge cases / failures:**
  - Invalid RSS URL, SSL issues, timeouts.
  - Feed returns items without expected fields (title/link/content).
  - Duplicate processing if the node is not configured to track “new” items (RSS Read is not inherently a “new item trigger”; it reads the feed at runtime).

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — iterates over RSS items in batches (looping).
- **Configuration (interpreted):** Parameters empty; you must set **Batch Size** (commonly 1 to process one post at a time).
- **Connections:** Uses **output index 1** to send items to **AI Agent Rewrite to Podcast Script** (per the workflow connections).
  - Output index 0 is unused in this workflow.
- **Edge cases / failures:**
  - Incorrect batch size can cause long runtimes or memory use.
  - If no items exist, downstream nodes won’t run.

---

### Block 2 — Script Generation (AI Agent + Structured Output)
**Overview:** Uses an AI agent to rewrite blog content into a podcast script, with a structured output parser to enforce predictable JSON-like outputs.

**Nodes involved:**
- AI Agent Rewrite to Podcast Script
- Azure OpenAI Chat Model
- Structured Output Parser

#### Node: Azure OpenAI Chat Model
- **Type / role:** `Azure OpenAI Chat Model` (LangChain) — provides the LLM backend.
- **Configuration (interpreted):** Parameters are empty in JSON; you must configure:
  - Azure OpenAI resource, deployment name (e.g., GPT-4o / GPT-4.1), API version, endpoint, credentials.
- **Connections:** Connected to the AI Agent via the special **ai_languageModel** port.
- **Version-specific notes:** Node typeVersion `1`—ensure your n8n supports `@n8n/n8n-nodes-langchain` Azure chat model node.
- **Edge cases / failures:**
  - Azure auth/permission errors (401/403), wrong deployment name, quota/rate limits (429).
  - Token limit errors for long blog posts—may require summarization or chunking.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured output parser — forces the agent’s response into a defined structure.
- **Configuration (interpreted):** Empty parameters in JSON; normally you define a schema (fields like `title`, `script`, `description`, etc.).
- **Connections:** Connected to the AI Agent via **ai_outputParser**.
- **Edge cases / failures:**
  - If the model does not comply with the schema, parsing fails.
  - Schema too strict for real-world content variability.

#### Node: AI Agent Rewrite to Podcast Script
- **Type / role:** LangChain Agent — orchestrates prompt + model + parsing to produce a podcast script.
- **Configuration (interpreted):** Empty parameters in JSON; typically contains:
  - System prompt: rewrite blog post into conversational spoken script, add intro/outro, etc.
  - Mapped inputs from RSS item fields (title/content/link).
  - Output aligned with Structured Output Parser schema.
- **Inputs:** From **Loop Over Items** (main), plus LLM via **Azure OpenAI Chat Model**, plus parser via **Structured Output Parser**.
- **Output:** To **Format Data For Audio**.
- **Edge cases / failures:**
  - Missing blog content fields => weak script or prompt failures.
  - Hallucinated facts if prompt doesn’t require fidelity to source.
  - Inconsistent output fields leading to downstream mapping issues.

---

### Block 3 — Audio Rendering & Google Drive Upload
**Overview:** Converts the generated script into the request payload required by the TTS provider (via HTTP), receives audio, and uploads it to Google Drive.

**Nodes involved:**
- Format Data For Audio
- Generate Audio
- Upload file

#### Node: Format Data For Audio
- **Type / role:** `Set` node — shapes data for the TTS request (text, voice settings, file naming).
- **Configuration (interpreted):** Empty parameters in JSON; should map:
  - Script text (from AI output) into something like `text`
  - Voice/model identifiers (ElevenLabs voice ID, stability, similarity, etc.)
  - Output filename (e.g., slug + date + `.mp3`)
- **Connections:** Outputs to **Generate Audio**.
- **Edge cases / failures:**
  - Large text might exceed TTS limits (character limits).
  - Missing fields cause HTTP body to be malformed.

#### Node: Generate Audio
- **Type / role:** `HTTP Request` — calls an external API to generate audio (commonly ElevenLabs).
- **Configuration (interpreted):** Empty parameters in JSON; must specify:
  - Method (usually `POST`)
  - URL (e.g., ElevenLabs TTS endpoint)
  - Auth header / API key
  - Request body (text + voice settings)
  - Response handling: receive binary audio (MP3) into n8n binary data
- **Connections:** Outputs to **Upload file**.
- **Edge cases / failures:**
  - 401/403 if API key missing/invalid.
  - 429 rate limiting; 5xx provider failures.
  - Response not treated as binary => upload step fails.
  - Text length limits causing 400 errors.

#### Node: Upload file
- **Type / role:** `Google Drive` — uploads the audio file to Drive.
- **Configuration (interpreted):** Empty parameters in JSON; must set:
  - Operation: upload
  - Target folder
  - Binary property name (e.g., `data`)
  - File name and MIME type (`audio/mpeg`)
- **Connections:** Outputs to **Format Data For RSS**.
- **Edge cases / failures:**
  - OAuth permission errors, expired refresh token.
  - Uploading empty binary data if previous step failed silently.
  - File naming collisions (overwrites) depending on Drive settings.

---

### Block 4 — RSS Episode Metadata, Feed Build, Update, Slack Notify
**Overview:** Builds a podcast RSS entry (or full feed), updates the RSS file, then posts a Slack notification.

**Nodes involved:**
- Format Data For RSS
- Podcast Feed Builder
- Update RSS File
- Notify Team

#### Node: Format Data For RSS
- **Type / role:** `Set` node — prepares episode metadata for RSS (title, enclosure URL, pubDate, description, etc.).
- **Configuration (interpreted):** Empty parameters in JSON; typically maps:
  - Episode title (from blog title or AI output)
  - Audio file URL (may require a public link; Google Drive links are not automatically direct MP3 enclosure URLs)
  - GUID, duration, description, image
- **Input:** From **Upload file** (which provides Drive file metadata).
- **Output:** To **Podcast Feed Builder**.
- **Edge cases / failures:**
  - Using non-direct or non-public URL for RSS `<enclosure>` breaks podcast apps.
  - Missing required RSS/iTunes tags.

#### Node: Podcast Feed Builder
- **Type / role:** `Code` node — constructs RSS XML or updates a feed structure.
- **Configuration (interpreted):** Code not included in JSON; expected to:
  - Generate an `<item>` block or full RSS XML with iTunes tags
  - Insert the new item in correct order
  - Output the updated feed content or file payload
- **Connections:** Outputs to **Update RSS File**.
- **Edge cases / failures:**
  - XML escaping issues (invalid RSS).
  - Incorrect date formats (RFC-822) causing validation issues.
  - Duplicates if GUID strategy not stable.

#### Node: Update RSS File
- **Type / role:** `Function` node — performs file update logic (write to disk, update in Drive, or build final payload).
- **Configuration (interpreted):** Function code not included; likely:
  - Appends/overwrites RSS feed file content
  - Possibly interacts with storage (Drive/S3/Git) indirectly (but no explicit node connection shown besides Slack)
- **Connections:** Outputs to **Notify Team**.
- **Edge cases / failures:**
  - If it writes to local filesystem: not persistent in many n8n deployments.
  - If it expects a previous RSS file to exist but it doesn’t, it may crash.
  - Encoding issues.

#### Node: Notify Team
- **Type / role:** `Slack` — sends a message to a Slack channel/user that a new episode was created.
- **Configuration (interpreted):** Empty parameters in JSON; must set:
  - Resource/operation: post message
  - Channel, message text (include episode title, Drive link, feed link)
  - Slack credentials (bot token)
- **Edge cases / failures:**
  - Missing scopes (e.g., `chat:write`), wrong channel ID, revoked token.
  - Slack rate limiting.

---

### Block 5 — Error Handling (Email)
**Overview:** If the workflow errors, it triggers an error workflow branch and sends an email.

**Nodes involved:**
- Error Handler Trigger
- Send a message

#### Node: Error Handler Trigger
- **Type / role:** `Error Trigger` — starts when another workflow execution fails (configured in workflow settings in n8n).
- **Configuration (interpreted):** Empty parameters; must be linked in n8n’s error workflow settings or used as a dedicated error workflow.
- **Connections:** Outputs to **Send a message**.
- **Edge cases / failures:**
  - Not enabled/selected as error workflow => never fires.
  - Limited context if execution data is not retained.

#### Node: Send a message
- **Type / role:** `Gmail` — sends an email notification about the error.
- **Configuration (interpreted):** Empty parameters in JSON; must set:
  - To/Subject/Body (include execution ID, error message, node name)
  - Gmail OAuth2 credentials
- **Connections:** From **Error Handler Trigger** only.
- **Edge cases / failures:**
  - OAuth token expiration, restricted scopes, Google security policy blocks.
  - Sending to invalid addresses.

---

### Sticky Notes (documentation nodes)
There are multiple sticky note nodes present but **their contents are empty strings** in the provided workflow export:
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note7

They do not add visible documentation content to replicate.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Scheduled entry point | — | Trigger: New Blog Post |  |
| Trigger: New Blog Post | RSS Feed Read | Fetch blog posts from RSS | Schedule Trigger | Loop Over Items |  |
| Loop Over Items | Split In Batches | Iterate through RSS items | Trigger: New Blog Post | AI Agent Rewrite to Podcast Script (output 1) |  |
| AI Agent Rewrite to Podcast Script | LangChain Agent | Rewrite blog post into podcast script | Loop Over Items; Azure OpenAI Chat Model (ai); Structured Output Parser (ai) | Format Data For Audio |  |
| Azure OpenAI Chat Model | Azure OpenAI Chat Model (LangChain) | LLM backend for agent | — | AI Agent Rewrite to Podcast Script (ai_languageModel) |  |
| Structured Output Parser | Structured Output Parser (LangChain) | Enforce structured AI output | — | AI Agent Rewrite to Podcast Script (ai_outputParser) |  |
| Format Data For Audio | Set | Prepare TTS payload & filename | AI Agent Rewrite to Podcast Script | Generate Audio |  |
| Generate Audio | HTTP Request | Call TTS provider to generate MP3 | Format Data For Audio | Upload file |  |
| Upload file | Google Drive | Upload generated audio | Generate Audio | Format Data For RSS |  |
| Format Data For RSS | Set | Prepare episode metadata for RSS | Upload file | Podcast Feed Builder |  |
| Podcast Feed Builder | Code | Build RSS XML / episode entry | Format Data For RSS | Update RSS File |  |
| Update RSS File | Function | Persist/update RSS feed content | Podcast Feed Builder | Notify Team |  |
| Notify Team | Slack | Notify channel of new episode | Update RSS File | — |  |
| Error Handler Trigger | Error Trigger | Entry point on errors | — | Send a message |  |
| Send a message | Gmail | Email error notification | Error Handler Trigger | — |  |
| Sticky Note | Sticky Note | Canvas note (empty) | — | — |  |
| Sticky Note1 | Sticky Note | Canvas note (empty) | — | — |  |
| Sticky Note2 | Sticky Note | Canvas note (empty) | — | — |  |
| Sticky Note3 | Sticky Note | Canvas note (empty) | — | — |  |
| Sticky Note4 | Sticky Note | Canvas note (empty) | — | — |  |
| Sticky Note5 | Sticky Note | Canvas note (empty) | — | — |  |
| Sticky Note7 | Sticky Note | Canvas note (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name: “Turn blog content into podcast episodes with AI voice and Drive storage” (or your choice).
   - Ensure the workflow is saved.

2) **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Configure schedule (e.g., every hour / every day at 08:00).
   - Connect: **Schedule Trigger → Trigger: New Blog Post**

3) **Add RSS Feed Read**
   - Node: **RSS Feed Read** (name it “Trigger: New Blog Post”)
   - Set **Feed URL** to your blog RSS (e.g., `https://example.com/feed.xml`).
   - Decide item limit (e.g., 10–50) and whether to fetch full content if supported.
   - Connect: **Trigger: New Blog Post → Loop Over Items**

4) **Add Split In Batches**
   - Node: **Split In Batches** (name “Loop Over Items”)
   - Set **Batch Size = 1** (recommended for sequential TTS generation).
   - Connect from **output 1** (the “continue” output) to the AI agent:
     - **Loop Over Items (output 1) → AI Agent Rewrite to Podcast Script**

5) **Add Azure OpenAI Chat Model**
   - Node: **Azure OpenAI Chat Model**
   - Credentials: configure Azure OpenAI (endpoint, key, API version).
   - Set **Deployment name** to your chat model deployment (e.g., GPT-4o class).
   - Connect to agent via AI port:
     - **Azure OpenAI Chat Model (ai_languageModel) → AI Agent Rewrite to Podcast Script**

6) **Add Structured Output Parser**
   - Node: **Structured Output Parser**
   - Define a schema you want back, for example:
     - `episodeTitle` (string)
     - `episodeScript` (string)
     - `episodeDescription` (string)
     - `explicit` (boolean)
   - Connect to agent via AI port:
     - **Structured Output Parser (ai_outputParser) → AI Agent Rewrite to Podcast Script**

7) **Add AI Agent**
   - Node: **AI Agent** (name “AI Agent Rewrite to Podcast Script”)
   - Configure prompt/instructions:
     - Input: RSS `title`, `content`/`contentSnippet`/`link` (whichever the RSS node provides)
     - Ask for a spoken-word script, include intro/outro, keep factual alignment with source.
     - Ensure it outputs fields matching the structured parser.
   - Connect: **AI Agent → Format Data For Audio**

8) **Add Set node: Format Data For Audio**
   - Node: **Set** (name “Format Data For Audio”)
   - Create fields needed by TTS HTTP request, e.g.:
     - `text` = the script (e.g., `{{$json.episodeScript}}`)
     - `voiceId` / `modelId` (if needed)
     - `fileName` = sanitized title + date + `.mp3`
   - Connect: **Format Data For Audio → Generate Audio**

9) **Add HTTP Request: Generate Audio**
   - Node: **HTTP Request** (name “Generate Audio”)
   - Configure for your TTS provider (commonly ElevenLabs):
     - Method: `POST`
     - URL: ElevenLabs TTS endpoint (voice-specific)
     - Headers: `xi-api-key: <secret>`, `Content-Type: application/json`, `Accept: audio/mpeg`
     - Body: JSON using the `text` and voice settings from previous node
     - Response: **download binary** (store as binary property like `data`)
   - Connect: **Generate Audio → Upload file**

10) **Add Google Drive: Upload file**
   - Node: **Google Drive** (name “Upload file”)
   - Credentials: Google Drive OAuth2.
   - Operation: **Upload**
   - Folder: pick target folder for episodes.
   - Binary Property: match HTTP node output (e.g., `data`)
   - File Name: use `{{$json.fileName}}` or similar
   - Connect: **Upload file → Format Data For RSS**

11) **Add Set node: Format Data For RSS**
   - Node: **Set** (name “Format Data For RSS”)
   - Map fields required to build the RSS `<item>`:
     - `title`, `description`, `pubDate`, `guid`
     - `enclosureUrl` (important: must be a direct public URL to the MP3)
   - Note: Google Drive share links usually are not valid enclosure URLs; you may need an alternate hosting/CDN or a Drive “direct download” pattern plus public permissions.
   - Connect: **Format Data For RSS → Podcast Feed Builder**

12) **Add Code node: Podcast Feed Builder**
   - Node: **Code** (name “Podcast Feed Builder”)
   - Implement logic to generate/update RSS XML:
     - Build `<item>` with enclosure, iTunes tags, etc.
     - Insert into existing feed content or create a new feed.
   - Output should pass feed content forward (e.g., `rssXml`).
   - Connect: **Podcast Feed Builder → Update RSS File**

13) **Add Function node: Update RSS File**
   - Node: **Function** (name “Update RSS File”)
   - Implement persistence strategy (choose one):
     - Update a file in Google Drive (requires adding Drive “Update file” node—currently not present in this workflow)
     - Commit to GitHub
     - Upload to S3
     - Write to local FS (not recommended in container/serverless)
   - In the provided workflow, this node directly leads to Slack notification, so it likely outputs a “success” status and/or updated feed location.
   - Connect: **Update RSS File → Notify Team**

14) **Add Slack: Notify Team**
   - Node: **Slack** (name “Notify Team”)
   - Credentials: Slack bot token with `chat:write`.
   - Operation: Post message to channel.
   - Message: include episode title, audio link, RSS feed link.
   - No outgoing connection required.

15) **Add Error Handling workflow path**
   - Node: **Error Trigger** (name “Error Handler Trigger”)
   - Node: **Gmail** (name “Send a message”)
   - Configure Gmail OAuth2 credentials and email template:
     - Include error details from the error trigger payload (execution id, workflow name, failing node).
   - Connect: **Error Handler Trigger → Send a message**
   - In n8n settings: set this workflow (or a separate one containing these nodes) as the **Error Workflow**, depending on your architecture.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided export. | No additional on-canvas documentation to preserve. |
| RSS + Google Drive warning: podcast apps require a publicly reachable, direct MP3 URL in `<enclosure>`. | Consider hosting audio on a podcast/CDN host or ensure Drive link is publicly accessible and truly direct-download. |
| RSS Read is not a true “new item trigger” by itself; it fetches the feed each run. Deduplication must be handled (GUID store, data store, DB, etc.). | Prevent re-generating episodes for the same post. |
| Code/Function node logic is missing from the export, so RSS construction and persistence must be implemented intentionally. | Implement robust XML escaping, GUID strategy, and storage. |