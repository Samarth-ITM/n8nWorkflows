Generate, translate and publish AI content to WordPress and socials with GPT-4.1 and Google Sheets

https://n8nworkflows.xyz/workflows/generate--translate-and-publish-ai-content-to-wordpress-and-socials-with-gpt-4-1-and-google-sheets-12571


# Generate, translate and publish AI content to WordPress and socials with GPT-4.1 and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow automatically selects a topic (from Google Sheets or via a user form submission), generates AI content, translates it, publishes it to WordPress, distributes an image-based post to multiple social platforms (Facebook, Telegram, LinkedIn profile/page, Instagram), and finally logs results back to Google Sheets while updating the topic’s status. It also sends internal notifications (Discord, Microsoft Teams, and a custom “Rapiwa” node).

**Typical use cases**
- Automated content pipeline for blogs + social distribution
- Topic queue management in Google Sheets
- AI-assisted research/content generation and localization

### Logical blocks
1.1 **Topic intake (scheduled + form ingestion)**  
1.2 **Topic queue iteration & notifications (batch loop)**  
1.3 **AI research + structured extraction**  
1.4 **Translation**  
1.5 **WordPress publishing**  
1.6 **Image retrieval, processing, WordPress media upload & featured image**  
1.7 **Social publishing (Facebook/Telegram/LinkedIn/Instagram)**  
1.8 **Logging & status updates back to Google Sheets**

---

## 2. Block-by-Block Analysis

### 2.1 Topic intake (scheduled + form ingestion)

**Overview:**  
Provides two independent ways to feed topics into the workflow: a scheduled pull from Google Sheets, and a user form submission flow that uploads a file to Google Drive and records a new topic.

**Nodes involved:**
- Schedule Trigger
- Get a New Topic form sheet
- On form submission (Captures user-submitted content)
- Uploads the submitted file to folder
- Save a New Topic

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` – starts executions on a time schedule.
- **Configuration (interpreted):** Schedule details are not present in JSON (empty parameters), so the schedule must be configured in the UI (cron/interval).
- **Outputs:** Connects to **Get a New Topic form sheet**.
- **Failure modes:** Misconfigured schedule (never fires), timezone mismatches, workflow inactive.

#### Node: Get a New Topic form sheet
- **Type / role:** `Google Sheets` – reads one or more topic rows from a sheet.
- **Configuration:** Not shown (empty parameters). Typically: spreadsheet ID, sheet/tab, read operation, filters (e.g., status=NEW), limit.
- **Inputs:** From **Schedule Trigger**.
- **Outputs:** To **Loop Over Items**.
- **Failure modes:** Google auth/permissions, wrong sheet/tab, rate limits, unexpected schema.

#### Node: On form submission (Captures user-submitted content)
- **Type / role:** `Form Trigger` – entry point for user-submitted topic/content.
- **Configuration:** Uses a webhookId; actual form fields aren’t included in JSON.
- **Outputs:** To **Uploads the submitted file to folder**.
- **Failure modes:** Form not published/accessible, missing required fields, file size limits.

#### Node: Uploads the submitted file to folder
- **Type / role:** `Google Drive` – uploads user-submitted file into a Drive folder.
- **Configuration:** Not shown; typically needs destination folder ID and binary property from form submission.
- **Inputs:** From **On form submission…**
- **Outputs:** To **Save a New Topic**
- **Failure modes:** Drive auth/permissions, invalid folder ID, missing binary data.

#### Node: Save a New Topic
- **Type / role:** `Google Sheets` – appends or updates a topic record based on form submission.
- **Configuration:** Not shown; likely “Append row” with topic/title, description, file link, status.
- **Inputs:** From **Uploads the submitted file to folder**
- **Outputs:** None further (ends this intake path).
- **Failure modes:** Sheet auth, column mismatch, data typing issues.

---

### 2.2 Topic queue iteration & notifications (batch loop)

**Overview:**  
Iterates through topic items from the sheet in batches. For each batch item it sends notifications (Discord, Teams, and Rapiwa). It also loops back to process the next batch.

**Nodes involved:**
- Loop Over Items
- Send a message (Discord)
- Create message (Microsoft Teams)
- Rapiwa
- Check Inputs

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` – controls batching/iteration.
- **Configuration:** Not shown; typically “Batch Size”.
- **Inputs:** From **Get a New Topic form sheet** and later from **Google Sheets Status Update** (re-loop).
- **Outputs:**
  - **Output 1:** to **Send a message**, **Rapiwa**, **Create message** (parallel fan-out)
  - **Output 2:** to **Check Inputs** (loop continuation)
- **Failure modes:** Batch size mis-set (too large), infinite loop if “continue” logic not handled, empty input list.

#### Node: Send a message
- **Type / role:** `Discord` – posts a message to a Discord channel.
- **Configuration:** Not shown; requires Discord bot token and channel ID.
- **Inputs:** From **Loop Over Items**.
- **Outputs:** None.
- **Failure modes:** Invalid bot permissions, rate limiting, message length/format errors.

#### Node: Create message
- **Type / role:** `Microsoft Teams` – sends a Teams message.
- **Configuration:** Not shown; depends on Teams credentials/webhook/channel config.
- **Inputs:** From **Loop Over Items**.
- **Outputs:** None.
- **Failure modes:** Auth, webhook disabled, tenant restrictions.

#### Node: Rapiwa
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` (community/custom node) – custom integration (unknown specifics).
- **Configuration:** Not shown.
- **Inputs:** From **Loop Over Items**.
- **Outputs:** None.
- **Version-specific requirements:** Requires installing the `n8n-nodes-rapiwa` community node package on the n8n instance.
- **Failure modes:** Missing package, incompatible node version, credential/config errors.

#### Node: Check Inputs
- **Type / role:** `IF` – routes items depending on whether required fields are present/valid.
- **Configuration:** Conditions are not present (empty parameters), but it has **two outputs**:
  - **True path (Output 1):** to **Research AI**
  - **False path (Output 2):** to **Loop Over Items** (skip/fallback to continue loop)
- **Inputs:** From **Loop Over Items** (Output 2).
- **Failure modes:** If conditions are blank/misconfigured, everything may go to one branch; missing expected fields can cause later expression failures.

---

### 2.3 AI research + structured extraction

**Overview:**  
Uses a LangChain Agent powered by OpenAI chat model, with a structured output parser, then extracts title/description from the agent’s JSON output.

**Nodes involved:**
- Research AI
- OpenAI
- Structured Output Parser
- Code (Extract Title & Description from JSON)

#### Node: OpenAI
- **Type / role:** `lmChatOpenAi` – language model provider for the agent.
- **Configuration:** Not shown. Typically includes model name (e.g., GPT-4.1), temperature, API key credential.
- **Connections:** Wired as **ai_languageModel** into **Research AI**.
- **Failure modes:** Invalid API key, model not available, quota/rate limits, timeouts.

#### Node: Structured Output Parser
- **Type / role:** LangChain structured output parser – forces/validates JSON schema.
- **Configuration:** Not shown (but it is connected as `ai_outputParser` to Research AI).
- **Failure modes:** Model returns invalid JSON; schema mismatch; parser throws validation errors.

#### Node: Research AI
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` – agent that generates researched content (likely includes prompt/tools, not visible).
- **Configuration:** Not shown; depends heavily on prompt instructions and expected JSON format.
- **Inputs:** From **Check Inputs** (True branch).
- **AI Inputs:** From **OpenAI** and **Structured Output Parser** via AI ports.
- **Outputs:** To **Code (Extract Title & Description from JSON)**.
- **Failure modes:** Hallucinated structure, long responses, tool misconfig, parser failures.

#### Node: Code (Extract Title & Description from JSON)
- **Type / role:** `Code` – transforms the agent output into fields needed downstream.
- **Configuration:** Not shown, but name implies it parses JSON and outputs at least **title** and **description/content**.
- **Inputs:** From **Research AI**.
- **Outputs:** To **Translate to Your Own Language**.
- **Failure modes:** JS errors due to missing properties, invalid JSON, unexpected agent output shape.

---

### 2.4 Translation

**Overview:**  
Translates generated content into a target language (your language), then pauses briefly to avoid downstream rate limits or sequencing issues.

**Nodes involved:**
- Translate to Your Own Language
- Wait 5 second

#### Node: Translate to Your Own Language
- **Type / role:** `Google Translate` – translates text.
- **Configuration:** Not shown; typically source/target languages and the input text field.
- **Inputs:** From **Code (Extract Title & Description from JSON)**.
- **Outputs:** To **Wait 5 second**.
- **Failure modes:** Auth, quota limits, unsupported language codes, very long text truncation.

#### Node: Wait 5 second
- **Type / role:** `Wait` – introduces a delay before publishing.
- **Configuration:** Not shown (but node name suggests 5 seconds).
- **Inputs:** From **Translate to Your Own Language**.
- **Outputs:** To **Create WordPress Post**.
- **Failure modes:** Rare; mostly operational (workflow execution timeouts on very strict hosting setups).

---

### 2.5 WordPress publishing

**Overview:**  
Creates a WordPress post from the translated content, then begins media/social distribution steps.

**Nodes involved:**
- Create WordPress Post

#### Node: Create WordPress Post
- **Type / role:** `WordPress` – creates a post (likely draft/published).
- **Configuration:** Not shown; typically site URL, auth (Application Password / OAuth / basic), post title/content/status/categories/tags.
- **Inputs:** From **Wait 5 second**.
- **Outputs:** To **Get file**.
- **Failure modes:** Auth failure, insufficient permissions, invalid HTML/content, wrong endpoint, WP security plugins blocking REST.

---

### 2.6 Image retrieval, processing, WordPress media upload & featured image

**Overview:**  
Fetches a file from Google Drive, optionally edits it, uploads it to WordPress Media Library via HTTP, updates metadata, and sets it as featured image.

**Nodes involved:**
- Get file
- Edit Image
- upload media to wp
- upload image to meta data
- set featured image

#### Node: Get file
- **Type / role:** `Google Drive` – downloads an image file (binary) used for WP and socials.
- **Configuration:** Not shown; typically file ID from sheet/topic row or from WP post context.
- **Inputs:** From **Create WordPress Post**.
- **Outputs:** Fan-out to:
  - **Edit Image**
  - **Facebook Image post**
  - **Telegram Image post**
  - **Create profile image post**
  - **Create page image post**
  - **Get Image ID for Instagram**
- **Failure modes:** Missing file ID, Drive permission issues, file not found.

#### Node: Edit Image
- **Type / role:** `Edit Image` – transforms image (resize/crop/overlay).
- **Configuration:** Not shown.
- **Inputs:** From **Get file**.
- **Outputs:** To **upload media to wp**.
- **Failure modes:** Unsupported image format, missing binary property name, memory limits.

#### Node: upload media to wp
- **Type / role:** `HTTP Request` – uploads media to WordPress REST API (`/wp-json/wp/v2/media`).
- **Configuration:** Not shown; typically multipart/form-data with binary, auth headers.
- **Inputs:** From **Edit Image**.
- **Outputs:** To **upload image to meta data**.
- **Version:** HTTP Request v4.2 (notable for modern auth/body options).
- **Failure modes:** 401/403 auth, 413 payload too large, incorrect content-type, WP blocks.

#### Node: upload image to meta data
- **Type / role:** `HTTP Request` – updates attachment metadata (title/alt text/caption).
- **Inputs:** From **upload media to wp** (needs returned media ID).
- **Outputs:** To **set featured image**.
- **Failure modes:** Missing media ID, permission issues, invalid payload fields.

#### Node: set featured image
- **Type / role:** `HTTP Request` – updates the created WP post to set `featured_media`.
- **Inputs:** From **upload image to meta data**.
- **Outputs:** None (workflow continues through social/logging path elsewhere).
- **Failure modes:** Missing post ID, wrong endpoint (`/posts/{id}`), permission issues.

---

### 2.7 Social publishing (Facebook/Telegram/LinkedIn/Instagram)

**Overview:**  
Uses the fetched Google Drive image to publish on multiple platforms. Instagram publishing is a two-step process (create container / get media ID, then publish).

**Nodes involved:**
- Facebook Image post
- Telegram Image post
- Create profile image post (LinkedIn)
- Create page image post (LinkedIn)
- Get Image ID for Instagram
- Publish Post to Instagram

#### Node: Facebook Image post
- **Type / role:** `Facebook Graph API` – posts an image to a Facebook destination.
- **Inputs:** From **Get file**.
- **Outputs:** None.
- **Failure modes:** Token expiration, missing permissions (pages_manage_posts, etc.), media upload constraints.

#### Node: Telegram Image post
- **Type / role:** `Telegram` – sends photo/message to a chat/channel.
- **Inputs:** From **Get file**.
- **Outputs:** None.
- **Failure modes:** Bot not admin in channel, chat ID wrong, file too large.

#### Node: Create profile image post
- **Type / role:** `LinkedIn` – posts an image post to a personal profile.
- **Inputs:** From **Get file**.
- **Outputs:** None.
- **Failure modes:** OAuth scopes, LinkedIn API restrictions, image upload steps misconfigured.

#### Node: Create page image post
- **Type / role:** `LinkedIn` – posts an image post to a LinkedIn page.
- **Inputs:** From **Get file**.
- **Outputs:** To **Google Sheets Final Blog** (indicating this is treated as the “final” step before logging).
- **Failure modes:** Page admin permissions, org URN issues, OAuth scopes.

#### Node: Get Image ID for Instagram
- **Type / role:** `HTTP Request` – likely creates an Instagram media container and returns an ID (Graph API).
- **Inputs:** From **Get file**.
- **Outputs:** To **Publish Post to Instagram**.
- **Failure modes:** Missing IG business account linkage, permissions, incorrect container parameters.

#### Node: Publish Post to Instagram
- **Type / role:** `Facebook Graph API` – publishes the Instagram container.
- **Inputs:** From **Get Image ID for Instagram**.
- **Outputs:** None.
- **Failure modes:** Container not ready, publish too early (often requires delay/retry), permission issues.

---

### 2.8 Logging & status updates back to Google Sheets

**Overview:**  
After social posting (notably after LinkedIn page posting), the workflow records the final blog/post data and updates processing status, then loops to handle the next topic.

**Nodes involved:**
- Google Sheets Final Blog
- Google Sheets Status Update
- Loop Over Items (re-entry)

#### Node: Google Sheets Final Blog
- **Type / role:** `Google Sheets` – writes final output record (post URL, title, language, social links, etc.).
- **Configuration:** Not shown.
- **Inputs:** From **Create page image post**.
- **Outputs:** To **Google Sheets Status Update**.
- **Retry:** enabled (`retryOnFail: true`, waits 5s between tries).
- **Failure modes:** Sheet write conflicts, auth, wrong columns; may retry repeatedly.

#### Node: Google Sheets Status Update
- **Type / role:** `Google Sheets` – updates the topic row status (e.g., DONE/POSTED) so it won’t be reprocessed.
- **Inputs:** From **Google Sheets Final Blog**.
- **Outputs:** Back to **Loop Over Items** (re-loop to continue processing remaining topics).
- **Retry:** enabled with 5s wait.
- **Failure modes:** Wrong row identification, concurrent edits, permission issues, schema mismatch.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled workflow entry point | — | Get a New Topic form sheet |  |
| Get a New Topic form sheet | googleSheets | Read next topic(s) from a sheet | Schedule Trigger | Loop Over Items |  |
| Loop Over Items | splitInBatches | Iterate topics in batches | Get a New Topic form sheet; Google Sheets Status Update | Send a message; Rapiwa; Create message; Check Inputs |  |
| Send a message | discord | Notify in Discord | Loop Over Items | — |  |
| Rapiwa | rapiwa | Custom notification/integration | Loop Over Items | — |  |
| Create message | microsoftTeams | Notify in Microsoft Teams | Loop Over Items | — |  |
| Check Inputs | if | Validate required fields & route | Loop Over Items | Research AI (true); Loop Over Items (false) |  |
| Research AI | langchain.agent | Generate researched structured content | Check Inputs | Code (Extract Title & Description from JSON) |  |
| Structured Output Parser | langchain.outputParserStructured | Enforce structured JSON output | — (AI port) | Research AI (AI port) |  |
| OpenAI | langchain.lmChatOpenAi | LLM backend (e.g., GPT-4.1) | — (AI port) | Research AI (AI port) |  |
| Code (Extract Title & Description from JSON) | code | Parse AI JSON to fields | Research AI | Translate to Your Own Language |  |
| Translate to Your Own Language | googleTranslate | Translate generated content | Code (Extract Title & Description from JSON) | Wait 5 second |  |
| Wait 5 second | wait | Delay before publishing | Translate to Your Own Language | Create WordPress Post |  |
| Create WordPress Post | wordpress | Create WP post | Wait 5 second | Get file |  |
| Get file | googleDrive | Download image for WP + socials | Create WordPress Post | Edit Image; Facebook Image post; Telegram Image post; Create profile image post; Create page image post; Get Image ID for Instagram |  |
| Edit Image | editImage | Transform image prior to upload | Get file | upload media to wp |  |
| upload media to wp | httpRequest | Upload media to WP REST API | Edit Image | upload image to meta data |  |
| upload image to meta data | httpRequest | Update attachment metadata | upload media to wp | set featured image |  |
| set featured image | httpRequest | Set featured image on WP post | upload image to meta data | — |  |
| Facebook Image post | facebookGraphApi | Publish image post to Facebook | Get file | — |  |
| Telegram Image post | telegram | Publish image post to Telegram | Get file | — |  |
| Create profile image post | linkedIn | Publish image post to LinkedIn profile | Get file | — |  |
| Create page image post | linkedIn | Publish image post to LinkedIn page | Get file | Google Sheets Final Blog |  |
| Get Image ID for Instagram | httpRequest | Create IG media container / get ID | Get file | Publish Post to Instagram |  |
| Publish Post to Instagram | facebookGraphApi | Publish IG container | Get Image ID for Instagram | — |  |
| Google Sheets Final Blog | googleSheets | Log final published outputs | Create page image post | Google Sheets Status Update |  |
| Google Sheets Status Update | googleSheets | Update topic status; continue loop | Google Sheets Final Blog | Loop Over Items |  |
| On form submission (Captures user-submitted content) | formTrigger | Manual entry point for new topics | — | Uploads the submitted file to folder |  |
| Uploads the submitted file to folder | googleDrive | Save submitted file to Drive | On form submission (Captures user-submitted content) | Save a New Topic |  |
| Save a New Topic | googleSheets | Append new topic to sheet | Uploads the submitted file to folder | — |  |
| Sticky Note | stickyNote | Comment (empty) | — | — |  |
| Sticky Note1 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note2 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note3 (disabled) | stickyNote | Comment (empty) | — | — |  |
| Sticky Note4 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note5 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note6 | stickyNote | Comment (empty) | — | — |  |
| Sticky Note8 | stickyNote | Comment (empty) | — | — |  |

> Sticky notes exist but contain no content in the provided JSON; therefore the “Sticky Note” column remains empty in effect.

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
- Name it: *Automatic Pick a Topic & Publish Translated Content on Multiple Platforms*

2) **Add entry point A: Schedule**
- Add **Schedule Trigger**
- Configure schedule (e.g., daily/weekly or cron)
- Connect to next node

3) **Google Sheets: read next topics**
- Add **Google Sheets** node named **Get a New Topic form sheet**
- Configure credentials (Google OAuth2)
- Operation: *Read / Get Many* (depending on your design)
- Select Spreadsheet + Sheet (topic queue)
- Filter to “unprocessed” topics (commonly a `Status` column = `NEW`)
- Connect to **Split In Batches**

4) **Batch loop**
- Add **Split In Batches** named **Loop Over Items**
- Set batch size (e.g., 1 for one topic per run)
- Connect Output 1 to notification nodes (next step)
- Connect Output 2 to **Check Inputs** (this acts as loop continuation)

5) **Notifications (optional but present)**
- Add **Discord** node **Send a message**, configure bot token + channel, message template using current item fields.
- Add **Microsoft Teams** node **Create message**, configure credentials/webhook/channel, message template.
- Install community node `n8n-nodes-rapiwa`, then add **Rapiwa** node and configure as needed.
- Connect **Loop Over Items (Output 1)** to all three in parallel.

6) **Validate required input**
- Add **IF** node **Check Inputs**
- Configure conditions (examples you likely need):
  - Topic/title is not empty
  - Has a Drive file ID (if required)
  - Status is NEW
- True output → **Research AI**
- False output → **Loop Over Items** (so it skips invalid rows and continues)

7) **LLM + structured output**
- Add **OpenAI Chat Model** node (`lmChatOpenAi`) named **OpenAI**
  - Configure OpenAI credentials (API key)
  - Select model (e.g., GPT-4.1 family if available in your account)
- Add **Structured Output Parser** node
  - Define a schema (fields like `title`, `post_html` or `post_markdown`, `excerpt`, `image_prompt`, `social_caption`, etc.)
- Add **AI Agent** node **Research AI**
  - Set system/user instructions to research and produce content in the schema
  - Connect **OpenAI** to **Research AI** via **ai_languageModel**
  - Connect **Structured Output Parser** to **Research AI** via **ai_outputParser**
- Connect **Check Inputs (true)** → **Research AI**

8) **Extract fields for downstream nodes**
- Add **Code** node: **Code (Extract Title & Description from JSON)**
- Implement JS that reads the agent output and returns normalized fields, e.g.:
  - `title`
  - `content` (HTML for WP)
  - `socialText`
  - `imageFileId` (if comes from sheet)
- Connect **Research AI** → **Code**

9) **Translate**
- Add **Google Translate** node: **Translate to Your Own Language**
- Configure Google credentials (or Translate credentials depending on n8n setup)
- Set target language (e.g., `fr`)
- Translate the fields you need (title + content or captions)
- Connect **Code** → **Translate**

10) **Delay**
- Add **Wait** node: **Wait 5 second**
- Set wait time to 5 seconds
- Connect **Translate** → **Wait**

11) **Create WordPress post**
- Add **WordPress** node: **Create WordPress Post**
- Configure WP credentials (commonly WP Application Password + username, or other supported method)
- Operation: Create Post
- Map:
  - Title ← translated title
  - Content ← translated content (HTML)
  - Status ← `publish` or `draft`
- Connect **Wait** → **Create WordPress Post**

12) **Get image file from Google Drive**
- Add **Google Drive** node: **Get file**
- Operation: Download
- Provide File ID (from sheet row or from Code output)
- Ensure output is **binary**
- Connect **Create WordPress Post** → **Get file**

13) **Edit image**
- Add **Edit Image** node: **Edit Image**
- Configure transformations (resize/crop as needed)
- Connect **Get file** → **Edit Image**

14) **Upload media to WordPress via HTTP**
- Add **HTTP Request** node: **upload media to wp**
- Configure:
  - Method: POST
  - URL: `https://YOUR_SITE/wp-json/wp/v2/media`
  - Auth: same as WP (basic/app password) or header
  - Send binary as multipart form-data (`file`)
- Connect **Edit Image** → **upload media to wp**

15) **Update attachment metadata**
- Add **HTTP Request** node: **upload image to meta data**
- Method: POST (or PUT)
- URL: `https://YOUR_SITE/wp-json/wp/v2/media/{{$json.id}}` (use returned media ID)
- Body: set `alt_text`, `caption`, `title` as desired
- Connect previous → this

16) **Set featured image**
- Add **HTTP Request** node: **set featured image**
- Method: POST (or PUT)
- URL: `https://YOUR_SITE/wp-json/wp/v2/posts/{{$node["Create WordPress Post"].json.id}}`
- Body: `{ "featured_media": <mediaId> }`
- Connect **upload image to meta data** → **set featured image**

17) **Social posting fan-out from the same Drive file**
- From **Get file**, connect in parallel to:
  - **Facebook Graph API** node **Facebook Image post** (publish photo)
  - **Telegram** node **Telegram Image post** (sendPhoto)
  - **LinkedIn** node **Create profile image post**
  - **LinkedIn** node **Create page image post**
  - **HTTP Request** node **Get Image ID for Instagram** (create container)
- Then connect:
  - **Get Image ID for Instagram** → **Facebook Graph API** node **Publish Post to Instagram**

18) **Logging to Google Sheets**
- Add **Google Sheets** node **Google Sheets Final Blog**
  - Append/update a row with WP URL/ID, title, language, timestamps, and social post IDs if you capture them
- Add **Google Sheets** node **Google Sheets Status Update**
  - Update the original topic row status to `DONE` (store WP post ID/URL)
- Connect **Create page image post** → **Google Sheets Final Blog** → **Google Sheets Status Update**

19) **Loop continuation**
- Connect **Google Sheets Status Update** → **Loop Over Items**
- Ensure Split In Batches is configured to continue until all items processed.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist in the workflow but are empty in the provided export. | No links/comments were embedded in sticky notes. |
| The workflow uses a community/custom node `n8n-nodes-rapiwa.rapiwa`. | Ensure the node package is installed on the n8n instance before importing/rebuilding. |
| Several nodes have empty parameters in the provided JSON (Sheets/WP/HTTP/Agent prompts). | You must configure credentials, sheet IDs, WP endpoints, and agent/prompt/schema in the n8n UI to match your environment. |