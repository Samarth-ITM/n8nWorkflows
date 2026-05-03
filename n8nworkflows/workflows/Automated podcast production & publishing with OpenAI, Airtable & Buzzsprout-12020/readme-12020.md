Automated podcast production & publishing with OpenAI, Airtable & Buzzsprout

https://n8nworkflows.xyz/workflows/automated-podcast-production---publishing-with-openai--airtable---buzzsprout-12020


# Automated podcast production & publishing with OpenAI, Airtable & Buzzsprout

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Automated podcast production & publishing with OpenAI, Airtable & Buzzsprout  
**Workflow name (in JSON):** Automated Podcast Generation with n8n, OpenAI & Buzzsprout

**Purpose:**  
Automatically turns a newly uploaded raw audio file in Google Drive into a fully produced podcast package: transcription → cleaned transcript → episode metadata (title/description/show notes/tags/publish date) → blog article draft → social posts → AI thumbnail → Buzzsprout publishing → Airtable record updates → Slack notification.

**Typical use cases:**
- Solo creators or teams who want a near “audio-to-published” pipeline.
- Content repurposing for marketing (blog + social) from the same recording.
- Central tracking in Airtable (episode asset hub).

### 1.1 Audio Intake & Preparation
Detect new file in a specific Google Drive folder and download it.

### 1.2 Transcription & Transcript Cleanup
Transcribe audio, then clean/QA the transcript into a structured JSON output.

### 1.3 Episode Metadata Generation
Use cleaned transcript + summary to generate episode title, description, show notes, tags, and suggested publish date.

### 1.4 Blog Article Creation
Convert show notes into a long-form markdown blog draft (returned as JSON).

### 1.5 Store Initial Episode Data in Airtable
Create an Airtable record for the episode draft containing metadata and blog draft.

### 1.6 Social Content Generation
Generate LinkedIn, Twitter thread, Instagram caption, TikTok script; then update the same Airtable record.

### 1.7 Thumbnail Generation
Generate an AI image thumbnail and save the returned URL to Airtable.

### 1.8 Publish Episode on Buzzsprout
Prepare payload, re-download the original audio, upload to Buzzsprout via HTTP multipart, then store Buzzsprout response back in Airtable.

### 1.9 Notify Team
Send a formatted Slack message with key episode publishing details.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Audio Intake & Preparation
**Overview:** Watches a Google Drive folder for newly created files, then downloads the raw audio so it can be transcribed.  
**Nodes involved:** `Trigger: New Audio File`, `Download Raw Audio`

#### Node: Trigger: New Audio File
- **Type / role:** Google Drive Trigger (`n8n-nodes-base.googleDriveTrigger`) — entry point.
- **Configuration (interpreted):**
  - Triggers on **fileCreated**.
  - Watches a **specific folder** (folder ID `1vuilPY22rcrwtfC-yPV0yu4FHeIQmJZD`, labeled “n8n”).
  - Polling schedule: **every minute**.
- **Key data used later:** `webViewLink` is used as a “URL-mode” file reference downstream.
- **Outputs:** Emits file metadata of newly created file(s).
- **Connections:** → `Download Raw Audio`.
- **Credentials:** Google Drive OAuth2.
- **Potential failures / edge cases:**
  - Polling delay: can be up to a minute.
  - Permissions: trigger user must have access to the folder and file.
  - If non-audio files are added, downstream transcription may fail (no filtering present).
  - Duplicate triggers possible if files are created/updated in quick succession depending on Drive behavior.

#### Node: Download Raw Audio
- **Type / role:** Google Drive (`n8n-nodes-base.googleDrive`) — downloads file binary.
- **Configuration:**
  - Operation: **download**
  - `fileId` is set using **URL mode**: `={{ $json.webViewLink }}`
    - This relies on n8n’s Drive node being able to resolve a webViewLink to an ID.
- **Outputs:** Binary field (commonly `data`) containing audio content + file info.
- **Connections:** → `AI Transcription`.
- **Credentials:** Google Drive OAuth2.
- **Potential failures / edge cases:**
  - If `webViewLink` resolution fails, use the file’s `id` instead (more robust).
  - Large files may hit memory/time limits depending on n8n hosting.
  - Google Drive download can fail on transient 5xx errors.

**Sticky note context:**  
“## Audio Intake & Preparation … detects when a new audio file is added … downloads it for processing.”

---

### Block 1.2 — Transcription & Transcript Cleanup
**Overview:** Uses OpenAI transcription for raw audio, then uses GPT to clean and structure the transcript into strict JSON.  
**Nodes involved:** `AI Transcription`, `AI Transcript Cleaning & QA`, `Parse Cleaned Transcript JSON`

#### Node: AI Transcription
- **Type / role:** OpenAI (LangChain) node (`@n8n/n8n-nodes-langchain.openAi`) — audio transcription.
- **Configuration:**
  - Resource: **audio**
  - Operation: **transcribe**
  - Model is not explicitly shown in parameters (depends on node defaults / credential settings).
- **Inputs:** Binary audio from `Download Raw Audio`.
- **Outputs:** Transcript text (referenced downstream as `{{ $json.text }}`).
- **Connections:** → `AI Transcript Cleaning & QA`.
- **Credentials:** OpenAI API.
- **Potential failures / edge cases:**
  - Unsupported audio codec/container; consider normalizing formats.
  - File too large for API limits; may need chunking.
  - Rate limits / quota errors.

#### Node: AI Transcript Cleaning & QA
- **Type / role:** OpenAI (LangChain) — GPT cleanup and structured output.
- **Configuration:**
  - Model: **gpt-4o**
  - System prompt enforces:
    - filler word removal, grammar/punctuation, speaker labels
    - provide a 2-sentence summary
    - provide 3–4 bullet key takeaways
    - provide full cleaned transcript
    - **ENTIRE reply must be ONLY valid JSON** (no markdown/backticks)
  - User content: `={{ $json.text }}` (the raw transcript from prior node)
- **Outputs:** AI response stored under an OpenAI node “output” structure; later parsed.
- **Connections:** → `Parse Cleaned Transcript JSON`.
- **Potential failures / edge cases:**
  - Model occasionally returns non-JSON despite instructions (no guardrails except parsing step).
  - Speaker labeling quality depends on transcript; if single-speaker, may not produce “Speaker 2”.

#### Node: Parse Cleaned Transcript JSON
- **Type / role:** Code (`n8n-nodes-base.code`) — JSON parsing and reshaping.
- **Configuration (logic):**
  - Parses: `JSON.parse($json.output[0].content[0].text)`
  - Returns a single item with `json` equal to parsed content.
- **Inputs:** OpenAI node response object.
- **Outputs:** Clean structured JSON (expected keys: `summary`, `key_takeaways`, `transcript`, etc. based on prompt).
- **Connections:** → `AI Episode Metadata Generator`.
- **Potential failures / edge cases:**
  - Will throw if AI output is not valid JSON (no try/catch here).
  - Assumes the OpenAI response path `output[0].content[0].text` always exists; may break if node output schema changes.

**Sticky note context:**  
“## Transcription & Transcript Cleanup … transcribed … cleaned … JSON parsing prepares the cleaned transcript…”

---

### Block 1.3 — Episode Metadata Generation
**Overview:** Generates SEO-friendly episode metadata from the cleaned transcript and summary, then parses the model response into fields.  
**Nodes involved:** `AI Episode Metadata Generator`, `Parse Episode Metadata JSON`

#### Node: AI Episode Metadata Generator
- **Type / role:** OpenAI (LangChain) — metadata generation.
- **Configuration:**
  - Model: **gpt-4o**
  - System instructions: return ONLY JSON with keys:
    - `episode_title`
    - `episode_description`
    - `show_notes`
    - `episode_tags` (array)
    - `suggested_publish_date` (ISO date or “publish now”)
  - User payload is an object constructed from upstream fields:
    - `summary`: `{{ $json['summary'] }}`
    - `transcript speaker1`: `{{ $json.transcript['Speaker 1'] }}`
    - `transcript speaker 2`: `{{ $json.transcript["Speaker 2"] }}`
    - `key_takeaways`: `{{ $json.key_takeaways }}`
- **Inputs:** Parsed transcript JSON from `Parse Cleaned Transcript JSON`.
- **Outputs:** AI JSON text inside OpenAI node response structure.
- **Connections:** → `Parse Episode Metadata JSON`.
- **Potential failures / edge cases:**
  - If `transcript['Speaker 2']` doesn’t exist, expression returns `undefined` → the model receives missing content.
  - If transcript format is not `{ "Speaker 1": "...", "Speaker 2": "..." }`, the payload may be incomplete.
  - Model may output markdown fences; parsing node tries to remove them.

#### Node: Parse Episode Metadata JSON
- **Type / role:** Code — robust JSON parsing + normalization.
- **Configuration (logic):**
  - Extracts raw text: `item.json.output?.[0]?.content?.[0]?.text || ""`
  - Removes code fences: `.replace(/```json|```/g, "")`
  - `try/catch` JSON.parse; on failure returns `{ error: "JSON parse failed", raw }`
  - Normalizes output fields to:
    - `episode_title` (string|null)
    - `episode_description` (string|null)
    - `show_notes` (string|null)
    - `episode_tags` (array default `[]`)
    - `suggested_publish_date` (string|null)
    - `full_parsed` (original parsed object)
- **Connections:** → `AI Blog Article Generator`.
- **Potential failures / edge cases:**
  - If parse fails, downstream nodes will receive nulls/empties; later steps may still run but produce weak results.
  - If `episode_tags` is not an array, it will be replaced with `[]`.

**Sticky note context:**  
“## Episode Metadata Generation … generate title/description/show notes/tags/publish date … parses AI response into clean fields…”

---

### Block 1.4 — Blog Article Creation
**Overview:** Converts show notes to a blog article in Markdown, returned as JSON; parses and merges with metadata.  
**Nodes involved:** `AI Blog Article Generator`, `Parse Blog Article JSON`

#### Node: AI Blog Article Generator
- **Type / role:** OpenAI (LangChain) — blog drafting.
- **Configuration:**
  - Model: **gpt-4o**
  - Prompt requires:
    - clean Markdown, 700–1000 words, H2/H3
    - author byline + SEO meta description
    - return ONLY JSON: `{ "blog_draft_markdown": "..." }`
  - Input: `{{ $json.show_notes }}`
- **Inputs:** Episode metadata from `Parse Episode Metadata JSON`.
- **Outputs:** JSON text in OpenAI response.
- **Connections:** → `Parse Blog Article JSON`.
- **Potential failures / edge cases:**
  - Model may include invalid JSON or escape issues due to Markdown content.

#### Node: Parse Blog Article JSON
- **Type / role:** Code — parse blog JSON and merge with episode metadata.
- **Configuration (logic):**
  - Extract raw: `$json.output[0].content[0].text`
  - Removes code fences and parses JSON; **throws** on failure (hard stop).
  - Pulls previous metadata explicitly: `$items("Parse Episode Metadata JSON")[0].json`
  - Creates `episode_tags_string`:
    - if array → `join(", ")`, else empty string
  - Returns merged JSON including:
    - all metadata fields
    - `blog_draft_markdown`
    - `episode_tags_string`
- **Connections:** → `Create Episode Draft Record`.
- **Potential failures / edge cases:**
  - Hard failure if blog JSON parsing fails.
  - Depends on node name `"Parse Episode Metadata JSON"` existing (renaming breaks `$items()` reference).
  - Assumes only one item (`[0]`).

**Sticky note context:**  
“## Blog Article Creation … transforms show notes into a polished blog post draft…”

---

### Block 1.5 — Store Initial Episode Data in Airtable
**Overview:** Creates a new Airtable record for the episode with metadata and blog draft.  
**Nodes involved:** `Create Episode Draft Record`

#### Node: Create Episode Draft Record
- **Type / role:** Airtable (`n8n-nodes-base.airtable`) — create record.
- **Configuration:**
  - Base: `appipUxLpxPEKRXrJ` (label cached: “Fake Review Detector”)
  - Table: `tbln0KraNZGYXyHQc` (cached: “podcastData”)
  - Operation: **create**
  - Fields written:
    - `episode_title`, `episode_description`, `show_notes`
    - `episode_tags` (string) = `episode_tags_string`
    - `blog_draft_markdown`
    - `suggested_publish_date`
- **Inputs:** Merged JSON from `Parse Blog Article JSON`.
- **Outputs:** Airtable create response, typically including `id` and `fields`.
- **Connections:** → `AI Social Media Content Generator`.
- **Credentials:** Airtable token.
- **Potential failures / edge cases:**
  - Base/table IDs must match environment.
  - Field types: all are sent as strings; mismatched Airtable field types could reject updates.
  - If your Airtable has required fields not provided, create will fail.

**Sticky note context:**  
“## Store Initial Episode Data in Airtable … first version of the episode record…”

---

### Block 1.6 — Social Content Generation
**Overview:** Generates multi-platform social copy from the blog draft and updates the existing Airtable record.  
**Nodes involved:** `AI Social Media Content Generator`, `Parse Social Content`, `Update Social Content`

#### Node: AI Social Media Content Generator
- **Type / role:** OpenAI (LangChain) — social repurposing.
- **Configuration:**
  - Model: **gpt-4o**
  - Output strictly JSON with keys:
    - `linkedin` (string)
    - `twitter_thread` (array of exactly 5 tweets)
    - `instagram_caption` (string with 5 hashtags)
    - `tiktok_script` (string)
  - Input content: `{{ $json.fields.blog_draft_markdown }}`
    - Note: At this point, `$json` is the Airtable create response, so fields are under `.fields`.
- **Outputs:** JSON text inside OpenAI response.
- **Connections:** → `Parse Social Content`.
- **Potential failures / edge cases:**
  - Model may not return exactly 5 tweets; parser does not validate length.
  - If Airtable response doesn’t include `fields.blog_draft_markdown`, prompt will be empty.

#### Node: Parse Social Content
- **Type / role:** Code — parse AI JSON and merge into unified payload for Airtable update.
- **Configuration (logic):**
  - Parses OpenAI JSON (removing code fences).
  - Pulls previous merged metadata from `$items("Parse Blog Article JSON")[0].json`
  - Pulls Airtable create response from `$items("Create Episode Draft Record")[0].json`
  - Writes:
    - `linkedin_post`, `twitter_thread`, `instagram_caption`, `tiktok_script`
    - `record_id_data`: `record_id.fields.record_Id`
      - **Important:** This assumes Airtable record has a field named `record_Id` already populated.
  - Builds strings:
    - `twitter_thread_string`: join tweets with blank lines
    - `episode_tags_string`: **BUG RISK** — it tries `previousData.episode_tags.join(", ")` but `episode_tags` in previousData is already normalized to an array in Parse Episode Metadata JSON; however Parse Blog Article JSON merges `previousData` which sets `episode_tags` to array, so OK. Still, later it overwrites `episode_tags_string`.
- **Connections:** → `Update Social Content`.
- **Potential failures / edge cases:**
  - If Airtable does not have `record_Id` field, `record_id_data` will be undefined and update will fail.
  - Uses `$items("...")[0]` heavily; multi-item executions may mismatch.
  - If JSON parsing fails, node throws and stops.

#### Node: Update Social Content
- **Type / role:** Airtable — update existing record.
- **Configuration:**
  - Operation: **update**
  - Matching column: `record_Id` (read-only in schema, but used as match key)
  - Values:
    - `record_Id` = `{{ $json.record_id_data }}`
    - `linkedin_post`, `tiktok_script`, `twitter_thread` (string), `instagram_caption`
- **Inputs:** Merged payload from `Parse Social Content`.
- **Outputs:** Airtable update response with updated `fields`.
- **Connections:** → `AI Thumbnail Generator (YouTube Image)`.
- **Potential failures / edge cases:**
  - If `record_Id` is read-only and not populated, matching fails.
  - Airtable update requires record identification; matching config must align with how your table stores record IDs.

**Sticky note context:**  
“## Social Content Generation … creates LinkedIn/Twitter/Instagram/TikTok … updates Airtable…”

---

### Block 1.7 — Thumbnail Generation
**Overview:** Generates an episode thumbnail image via OpenAI Images and stores the image URL in Airtable.  
**Nodes involved:** `AI Thumbnail Generator (YouTube Image)`, `Save Thumbnail URL`

#### Node: AI Thumbnail Generator (YouTube Image)
- **Type / role:** OpenAI (LangChain) — image generation.
- **Configuration:**
  - Resource: **image**
  - Model: **dall-e-2**
  - Prompt includes `{{ $json.fields.episode_title }}`
  - Options:
    - size set to `1024x1024` (note: prompt asks 1280x720 but option is square)
    - returns image URLs: enabled
- **Inputs:** Airtable update response from `Update Social Content` (uses `.fields.episode_title`).
- **Outputs:** Image generation response; later node expects `url` at `$json.url`.
- **Connections:** → `Save Thumbnail URL`.
- **Potential failures / edge cases:**
  - Aspect ratio mismatch: 1024x1024 won’t be 1280x720; adjust size if supported.
  - Output schema may return `data[0].url` depending on node version; this workflow expects `$json.url`.
  - DALL·E 2 limitations and content policy restrictions.

#### Node: Save Thumbnail URL
- **Type / role:** Airtable — update record with thumbnail URL.
- **Configuration:**
  - Operation: **update**
  - Match on `record_Id`
  - `record_Id` is taken from: `{{ $('Update Social Content').item.json.fields.record_Id }}`
  - `youtube_thumbnail_url` = `{{ $json.url }}`
- **Inputs:** Image generation output (for `url`), plus uses a cross-node reference to `Update Social Content`.
- **Outputs:** Airtable updated record.
- **Connections:** → `Prepare Buzzsprout Episode Payload`.
- **Potential failures / edge cases:**
  - If image node doesn’t output `url` at root, field update will store blank.
  - Cross-node item reference assumes same item index; can break with multiple items.

**Sticky note context:**  
“## Thumbnail Generation … generates custom thumbnail … saves image URL into Airtable…”

---

### Block 1.8 — Publish Episode on Buzzsprout
**Overview:** Prepares Buzzsprout payload, re-downloads the original audio, uploads episode via HTTP multipart, then stores Buzzsprout IDs/URLs back into Airtable.  
**Nodes involved:** `Prepare Buzzsprout Episode Payload`, `Download Audio for Buzzsprout Upload`, `Upload Episode`, `Save Buzzsprout Episode Data`

#### Node: Prepare Buzzsprout Episode Payload
- **Type / role:** Set (`n8n-nodes-base.set`) — constructs fields for Buzzsprout request.
- **Configuration:**
  - Sets:
    - `episode_title` = `{{ $json.fields.episode_title }}`
    - `final_description` = `{{ $json.fields.episode_description + "\n\nThumbnail: " + $json.fields.youtube_thumbnail_url }}`
- **Inputs:** Airtable record from `Save Thumbnail URL` (uses `.fields.*`).
- **Outputs:** Simple JSON with `episode_title` and `final_description`.
- **Connections:** → `Download Audio for Buzzsprout Upload`.
- **Potential failures / edge cases:**
  - If `youtube_thumbnail_url` missing, description includes “Thumbnail: ” with blank.
  - If episode_description is empty/null, concatenation may yield `"null\n\nThumbnail..."`.

#### Node: Download Audio for Buzzsprout Upload
- **Type / role:** Google Drive download — obtains binary again for Buzzsprout.
- **Configuration:**
  - Operation: download
  - `fileId` in URL mode: `={{ $('Trigger: New Audio File').item.json.webViewLink }}`
- **Inputs:** Payload from prior Set node, but uses cross-node reference to the trigger for the file link.
- **Outputs:** Binary `data` used by HTTP Request multipart.
- **Connections:** → `Upload Episode`.
- **Potential failures / edge cases:**
  - Same `webViewLink` robustness concern as earlier.
  - Cross-node reference to Trigger assumes same execution item.

#### Node: Upload Episode
- **Type / role:** HTTP Request (`n8n-nodes-base.httpRequest`) — publishes to Buzzsprout.
- **Configuration:**
  - POST to: `https://www.buzzsprout.com/api/{your_podcast_id}/episodes.json`
  - Authentication: header `Authorization: Token token={you_api_key}`
  - Body: multipart/form-data with:
    - `title` = `{{ $('Prepare Buzzsprout Episode Payload').item.json.episode_title }}`
    - `description` = `{{ $('Prepare Buzzsprout Episode Payload').item.json.final_description }}`
    - `file` = binary field `data`
- **Inputs:** Binary audio from `Download Audio for Buzzsprout Upload`.
- **Outputs:** Buzzsprout episode object (expects fields like `id`, `audio_url`, `published_at`, `guid`).
- **Connections:** → `Save Buzzsprout Episode Data`.
- **Potential failures / edge cases:**
  - Must replace `{your_podcast_id}` and `{you_api_key}`.
  - Buzzsprout API errors: 401 auth, 422 validation, upload size limits.
  - Ensure the HTTP node is configured to include binary in multipart correctly (it references `inputDataFieldName: "data"`).

#### Node: Save Buzzsprout Episode Data
- **Type / role:** Airtable update — saves Buzzsprout identifiers/URLs into the record.
- **Configuration:**
  - Operation: update
  - Match on `record_Id`
  - Writes:
    - `guid` = `{{ $json.guid }}`
    - `audio_url` = `{{ $json.audio_url }}`
    - `published_at` = `{{ $json.published_at }}`
    - `buzzsprout_episode_id` = `{{ $json.id }}`
    - `record_Id` = `{{ $('Save Thumbnail URL').item.json.fields.record_Id }}`
- **Inputs:** Buzzsprout API response JSON + cross-node record_Id from Airtable.
- **Outputs:** Updated Airtable record with publishing fields.
- **Connections:** → `Prepare Slack Message`.
- **Potential failures / edge cases:**
  - If Buzzsprout response schema differs, fields may be missing.
  - Matching relies on `record_Id` existing and stable.

**Sticky note context:**  
- “## Prepare Episode for Buzzsprout … prepares final formatted text … retrieves original audio…”  
- “## Publish Episode on Buzzsprout … uploads … stores everything back in Airtable…”

---

### Block 1.9 — Notify Team
**Overview:** Formats episode details and posts a message to Slack for the marketing team.  
**Nodes involved:** `Prepare Slack Message`, `Notify Marketing Team`

#### Node: Prepare Slack Message
- **Type / role:** Set — renames fields for Slack template usage.
- **Configuration:**
  - `Episode Title` = `{{ $json.fields.episode_title }}`
  - `Buzzsprout episode Id` = `{{ $json.fields.buzzsprout_episode_id }}`
  - `Published at` = `{{ $json.fields.published_at }}`
  - `Youtube thumbnail url` = `{{ $json.fields.youtube_thumbnail_url }}`
- **Inputs:** Airtable updated record from `Save Buzzsprout Episode Data`.
- **Outputs:** Simplified JSON keys used by Slack node.
- **Connections:** → `Notify Marketing Team`.
- **Potential failures / edge cases:**
  - If Airtable update response does not include `.fields.*` (depending on Airtable node settings), expressions may be undefined.

#### Node: Notify Marketing Team
- **Type / role:** Slack (`n8n-nodes-base.slack`) — sends message to a channel.
- **Configuration:**
  - Operation: post message to a selected channel (channelId is blank in JSON export; must be set).
  - Message text template contains formatted blocks (currently includes a 🎙️ character even though earlier AI prompts requested no emojis; Slack node itself uses it).
  - Uses fields from prior node like:
    - `{{ $json["Episode Title"] }}`
    - `{{ $json["Buzzsprout episode Id"] }}`
    - `{{ $json["Published at"] }}`
    - `{{ $json["Youtube thumbnail url"] }}`
- **Inputs:** Set node output.
- **Outputs:** Slack API response.
- **Credentials:** Slack API.
- **Potential failures / edge cases:**
  - Missing channel selection causes runtime error.
  - Slack token scope must include chat:write for the channel.
  - If fields are undefined, message will have blanks.

**Sticky note context:**  
“## Notify Team … structured message … so they can begin promotion instantly.”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: New Audio File | googleDriveTrigger | Entry point: detect new file in Drive folder | — | Download Raw Audio | ## Audio Intake & Preparation  \nAutomatically detects when a new audio file is added to Google Drive and downloads it for processing. This begins the podcast workflow by pulling in the raw audio that will later be transcribed and converted into content. |
| Download Raw Audio | googleDrive | Download binary audio for transcription | Trigger: New Audio File | AI Transcription | ## Audio Intake & Preparation  \nAutomatically detects when a new audio file is added to Google Drive and downloads it for processing. This begins the podcast workflow by pulling in the raw audio that will later be transcribed and converted into content. |
| AI Transcription | openAi (LangChain) | Transcribe audio to text | Download Raw Audio | AI Transcript Cleaning & QA | ## Transcription & Transcript Cleanup  \nThe raw audio is transcribed using AI, then cleaned to remove filler words, fix formatting, and produce a structured transcript. The JSON parsing prepares the cleaned transcript for the next steps like episode title generation and notes creation. |
| AI Transcript Cleaning & QA | openAi (LangChain) | Clean transcript + summary/takeaways as strict JSON | AI Transcription | Parse Cleaned Transcript JSON | ## Transcription & Transcript Cleanup  \nThe raw audio is transcribed using AI, then cleaned to remove filler words, fix formatting, and produce a structured transcript. The JSON parsing prepares the cleaned transcript for the next steps like episode title generation and notes creation. |
| Parse Cleaned Transcript JSON | code | Parse AI JSON into fields | AI Transcript Cleaning & QA | AI Episode Metadata Generator | ## Transcription & Transcript Cleanup  \nThe raw audio is transcribed using AI, then cleaned to remove filler words, fix formatting, and produce a structured transcript. The JSON parsing prepares the cleaned transcript for the next steps like episode title generation and notes creation. |
| AI Episode Metadata Generator | openAi (LangChain) | Generate title/description/show notes/tags/publish date | Parse Cleaned Transcript JSON | Parse Episode Metadata JSON | ## Episode Metadata Generation  \nUses the cleaned transcript to generate high-quality metadata — title, description, show notes, tags, and publish date. Then parses the AI response into clean fields for use in Airtable, Buzzsprout, and social media generation. |
| Parse Episode Metadata JSON | code | Parse and normalize metadata JSON | AI Episode Metadata Generator | AI Blog Article Generator | ## Episode Metadata Generation  \nUses the cleaned transcript to generate high-quality metadata — title, description, show notes, tags, and publish date. Then parses the AI response into clean fields for use in Airtable, Buzzsprout, and social media generation. |
| AI Blog Article Generator | openAi (LangChain) | Expand show notes into markdown blog draft (as JSON) | Parse Episode Metadata JSON | Parse Blog Article JSON | ## Blog Article Creation  \nTransforms the show notes into a polished blog post draft in Markdown format. This creates long-form written content that can be published on blogs, websites, or newsletters. |
| Parse Blog Article JSON | code | Parse blog JSON and merge with metadata | AI Blog Article Generator | Create Episode Draft Record | ## Blog Article Creation  \nTransforms the show notes into a polished blog post draft in Markdown format. This creates long-form written content that can be published on blogs, websites, or newsletters. |
| Create Episode Draft Record | airtable | Create initial episode record in Airtable | Parse Blog Article JSON | AI Social Media Content Generator | ## Store Initial Episode Data in Airtable  \nStores the transcript, title, initial notes, tags, and other metadata into Airtable. This creates the first version of the episode record before thumbnails, social posts, and Buzzsprout links are added. |
| AI Social Media Content Generator | openAi (LangChain) | Generate social posts from blog draft | Create Episode Draft Record | Parse Social Content | ## Social Content Generation  \nCreates LinkedIn, Twitter thread, Instagram caption, and TikTok script. Merges the content with previous data and updates the Airtable record with all social-ready text. |
| Parse Social Content | code | Parse social JSON and prepare Airtable update fields | AI Social Media Content Generator | Update Social Content | ## Social Content Generation  \nCreates LinkedIn, Twitter thread, Instagram caption, and TikTok script. Merges the content with previous data and updates the Airtable record with all social-ready text. |
| Update Social Content | airtable | Update Airtable record with social content | Parse Social Content | AI Thumbnail Generator (YouTube Image) | ## Social Content Generation  \nCreates LinkedIn, Twitter thread, Instagram caption, and TikTok script. Merges the content with previous data and updates the Airtable record with all social-ready text. |
| AI Thumbnail Generator (YouTube Image) | openAi (LangChain) | Generate AI thumbnail image URL | Update Social Content | Save Thumbnail URL | ## Thumbnail Generation  \nGenerates a custom thumbnail for the episode using AI. Saves the image URL into Airtable so it’s available for marketing and Buzzsprout descriptions. |
| Save Thumbnail URL | airtable | Save thumbnail URL to Airtable | AI Thumbnail Generator (YouTube Image) | Prepare Buzzsprout Episode Payload | ## Thumbnail Generation  \nGenerates a custom thumbnail for the episode using AI. Saves the image URL into Airtable so it’s available for marketing and Buzzsprout descriptions. |
| Prepare Buzzsprout Episode Payload | set | Build title + final description for Buzzsprout | Save Thumbnail URL | Download Audio for Buzzsprout Upload | ## Prepare Episode for Buzzsprout  \nPrepares the final formatted text for publishing, including cleaned description and thumbnail link. Also retrieves the original audio file again to attach to Buzzsprout. |
| Download Audio for Buzzsprout Upload | googleDrive | Download audio binary for Buzzsprout upload | Prepare Buzzsprout Episode Payload | Upload Episode | ## Prepare Episode for Buzzsprout  \nPrepares the final formatted text for publishing, including cleaned description and thumbnail link. Also retrieves the original audio file again to attach to Buzzsprout. |
| Upload Episode | httpRequest | Publish episode to Buzzsprout via API | Download Audio for Buzzsprout Upload | Save Buzzsprout Episode Data | ## Publish Episode on Buzzsprout  \nUploads the episode with audio, title, and description to Buzzsprout. After Buzzsprout returns the episode URL and ID, these details are stored back in Airtable to keep the episode record complete. |
| Save Buzzsprout Episode Data | airtable | Save Buzzsprout IDs/URLs back into Airtable | Upload Episode | Prepare Slack Message | ## Publish Episode on Buzzsprout  \nUploads the episode with audio, title, and description to Buzzsprout. After Buzzsprout returns the episode URL and ID, these details are stored back in Airtable to keep the episode record complete. |
| Prepare Slack Message | set | Map Airtable fields into Slack-friendly keys | Save Buzzsprout Episode Data | Notify Marketing Team | ## Notify Team  \nSends a structured message to your marketing team with episode title, publish link, thumbnail, and schedule details so they can begin promotion instantly. |
| Notify Marketing Team | slack | Send Slack notification | Prepare Slack Message | — | ## Notify Team  \nSends a structured message to your marketing team with episode title, publish link, thumbnail, and schedule details so they can begin promotion instantly. |
| Sticky Note | stickyNote | Comment | — | — | # How It Works … (see General Notes & Resources) |
| Sticky Note1 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note2 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note3 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note4 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note5 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note6 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note7 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note8 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note9 | stickyNote | Comment | — | — | (comment only) |
| Sticky Note10 | stickyNote | Comment | — | — | (comment only) |

> Note: Sticky Note nodes are included as nodes in the workflow, but they do not participate in execution.

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Google Drive OAuth2 credential (access to the target folder).
   2. OpenAI API credential (access to transcription + GPT + image generation).
   3. Airtable Personal Access Token credential (access to base/table).
   4. Slack credential (bot token with permission to post to the target channel).
   5. Buzzsprout: you will use an API token in the HTTP Request header (not an n8n credential in this workflow).

2) **Add Trigger node**
   1. Add **Google Drive Trigger**.
   2. Event: **File Created**.
   3. Trigger on: **Specific Folder** → select your folder.
   4. Polling: every minute (or adjust).
   5. Connect to next node.

3) **Download the raw audio**
   1. Add **Google Drive** node.
   2. Operation: **Download**.
   3. File reference: use the trigger output.
      - Recommended: use file `id` if available.
      - As in workflow: set File ID in URL mode to `{{$json.webViewLink}}`.
   4. Connect to transcription node.

4) **Transcribe audio**
   1. Add **OpenAI (LangChain)** node.
   2. Resource: **Audio** → Operation: **Transcribe**.
   3. Ensure it consumes the binary from Drive.
   4. Connect to cleanup node.

5) **Clean transcript into strict JSON**
   1. Add **OpenAI (LangChain)** node with model **gpt-4o**.
   2. Provide system prompt enforcing JSON-only output (as in workflow).
   3. User content: `{{$json.text}}`.
   4. Connect to parsing code node.

6) **Parse cleaned transcript JSON**
   1. Add **Code** node.
   2. Parse: `JSON.parse($json.output[0].content[0].text)` and return it as the item JSON.
   3. Connect to metadata generator.

7) **Generate episode metadata**
   1. Add **OpenAI (LangChain)** node model **gpt-4o**.
   2. System prompt requests JSON keys: `episode_title`, `episode_description`, `show_notes`, `episode_tags`, `suggested_publish_date`.
   3. Provide input payload built from:
      - `summary`
      - transcript speaker fields from `$json.transcript`
      - `key_takeaways`
   4. Connect to parsing code node.

8) **Parse episode metadata JSON (with fallback)**
   1. Add **Code** node.
   2. Implement:
      - extract raw text from OpenAI response
      - strip ``` fences
      - try/catch JSON.parse
      - output normalized keys + defaults
   3. Connect to blog generator.

9) **Generate blog article JSON**
   1. Add **OpenAI (LangChain)** node model **gpt-4o**.
   2. Prompt: convert `show_notes` into markdown blog draft; return JSON `{ "blog_draft_markdown": "..." }`.
   3. Connect to blog parsing node.

10) **Parse blog JSON and merge**
   1. Add **Code** node:
      - parse blog JSON (strip fences; throw on failure)
      - fetch previous metadata via `$items("Parse Episode Metadata JSON")[0].json`
      - compute `episode_tags_string`
      - merge into one object
   2. Connect to Airtable create.

11) **Create Airtable episode draft record**
   1. Add **Airtable** node.
   2. Operation: **Create**.
   3. Select Base + Table (your podcast table).
   4. Map fields: title, description, show notes, tags string, blog markdown, suggested publish date.
   5. Connect to social generator.

   **Important:** Ensure your Airtable table contains a usable unique key for later updates.  
   This workflow expects a field named **`record_Id`** that can be used as a matching column. If you don’t have it, either:
   - add a formula field that equals `RECORD_ID()`, or
   - change update nodes to use Airtable’s native `id` instead of matching.

12) **Generate social content**
   1. Add **OpenAI (LangChain)** node model **gpt-4o**.
   2. Prompt: JSON object with linkedin/twitter_thread/instagram_caption/tiktok_script.
   3. Input: `{{$json.fields.blog_draft_markdown}}` from Airtable create response.
   4. Connect to parser node.

13) **Parse social content + prepare update**
   1. Add **Code** node:
      - parse social JSON
      - get merged metadata via `$items("Parse Blog Article JSON")[0].json`
      - get Airtable record data via `$items("Create Episode Draft Record")[0].json`
      - build `twitter_thread_string`
      - produce payload including `record_Id` value to match on
   2. Connect to Airtable update.

14) **Update Airtable with social fields**
   1. Add **Airtable** node.
   2. Operation: **Update**.
   3. Matching column: `record_Id` (or switch to Airtable internal `id` strategy).
   4. Map: linkedin_post, twitter_thread, instagram_caption, tiktok_script.
   5. Connect to thumbnail generation.

15) **Generate thumbnail image**
   1. Add **OpenAI (LangChain)** image node.
   2. Model: `dall-e-2` (or update as desired).
   3. Prompt referencing `{{$json.fields.episode_title}}`.
   4. Configure return URLs.
   5. Connect to Airtable update that stores the URL.

16) **Save thumbnail URL to Airtable**
   1. Add **Airtable** node (Update).
   2. Match record using `record_Id` from the updated Airtable record.
   3. Set `youtube_thumbnail_url` from the image node output (`$json.url` in this workflow; adjust if your image node returns nested URLs).

17) **Prepare Buzzsprout payload**
   1. Add **Set** node.
   2. Create:
      - `episode_title` from Airtable fields
      - `final_description` = description + thumbnail URL appended

18) **Re-download audio for publishing**
   1. Add **Google Drive** node (Download).
   2. Reference the original trigger file (prefer file `id`; workflow uses `webViewLink`).
   3. Connect to HTTP upload.

19) **Upload episode to Buzzsprout**
   1. Add **HTTP Request** node.
   2. Method: POST.
   3. URL: `https://www.buzzsprout.com/api/<PODCAST_ID>/episodes.json`
   4. Headers: `Authorization: Token token=<API_KEY>`
   5. Body type: multipart/form-data:
      - `title` (string)
      - `description` (string)
      - `file` (binary from Drive, binary property name typically `data`)
   6. Connect to Airtable update.

20) **Save Buzzsprout response to Airtable**
   1. Add **Airtable** node (Update).
   2. Match on `record_Id`.
   3. Store: `buzzsprout_episode_id`, `audio_url`, `published_at`, `guid`.

21) **Prepare Slack message**
   1. Add **Set** node to map Airtable fields into display-friendly keys (Episode Title, Published at, etc.).

22) **Notify Slack channel**
   1. Add **Slack** node.
   2. Operation: Post message to channel.
   3. Select channel ID.
   4. Use templated message body referencing fields from Set node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “# How It Works … converts a raw audio recording into a fully published podcast episode … stores everything back in Airtable … notifies on Slack.” | Sticky note “How It Works” summary embedded in the canvas |
| “# Setup Steps … Prepare Airtable … Connect Accounts … Add Trigger … Transcript & AI Generation … Save to Airtable … Generate Thumbnail … Upload to Buzzsprout … Send Slack Notification.” | Sticky note “Setup Steps” embedded in the canvas |

