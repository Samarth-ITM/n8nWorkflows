Turn GitHub releases into audio updates and newsletters with OpenAI and ElevenLabs

https://n8nworkflows.xyz/workflows/turn-github-releases-into-audio-updates-and-newsletters-with-openai-and-elevenlabs-14973


# Turn GitHub releases into audio updates and newsletters with OpenAI and ElevenLabs

The user wants me to analyze an n8n workflow JSON and produce a comprehensive structured reference document. The workflow is titled "Turn GitHub releases into audio updates and newsletters with OpenAI and ElevenLabs". I need to produce sections: Workflow Overview, Block-by-Block Analysis, Summary Table, Reproducing the Workflow from Scratch, General Notes & Resources.

I'll need to analyze each node in the JSON.

Let's list the nodes:

1. "📋 MAIN — Workflow Overview" (stickyNote)
2. "📝 Note — GitHub Trigger, Parse & Notion Fetch" (stickyNote)
3. "📝 Note — Script, TTS & UploadToURL" (stickyNote)
4. "📝 Note — Subscribers, HTML Build & Gmail Send" (stickyNote)
5. "📝 Note — Slack Post & Notion Write-Back" (stickyNote)
6. "GitHub — On New Release Published" (githubTrigger)
7. "Code — Parse & Categorise Release Notes" (code)
8. "Notion — Query Changelog Database" (notion)
9. "Code — Merge Notion Context + Release Data" (code)
10. "OpenAI — Write Podcast-Style Spoken Script" (openAi)
11. "ElevenLabs — Generate MP3 Audio" (httpRequest)
12. "Code — Store Script Text Before Upload" (code)
13. "Upload a File" (uploadToUrl)
14. "Code — Store Audio URL + Full Release Data" (code)
15. "Google Sheets — Load Subscriber List" (googleSheets)
16. "Aggregate — Bundle All Subscriber Emails" (aggregate)
17. "Code — Build HTML Email Newsletter" (code)
18. "Gmail — Send Audio Newsletter to Subscribers" (gmail)
19. "Slack — Post to #product-updates Channel" (slack)
20. "Notion — Write Back Audio URL to Changelog Page" (notion)

We also have connections. Let's examine the connections to understand flow.

Connections:

- "Upload a File" -> "Code — Store Audio URL + Full Release Data"
- "ElevenLabs — Generate MP3 Audio" -> "Code — Store Script Text Before Upload"
- "GitHub — On New Release Published" -> "Code — Parse & Categorise Release Notes"
- "Notion — Query Changelog Database" -> "Code — Merge Notion Context + Release Data"
- "Code — Build HTML Email Newsletter" -> "Gmail — Send Audio Newsletter to Subscribers"
- "Google Sheets — Load Subscriber List" -> "Aggregate — Bundle All Subscriber Emails"
- "Code — Store Script Text Before Upload" -> "Upload a File"
- "Code — Parse & Categorise Release Notes" -> "Notion — Query Changelog Database"
- "Aggregate — Bundle All Subscriber Emails" -> "Code — Build HTML Email Newsletter"
- "Slack — Post to #product-updates Channel" -> "Notion — Write Back Audio URL to Changelog Page"
- "Code — Merge Notion Context + Release Data" -> "OpenAI — Write Podcast-Style Spoken Script"
- "Code — Store Audio URL + Full Release Data" -> "Google Sheets — Load Subscriber List"
- "OpenAI — Write Podcast-Style Spoken Script" -> "ElevenLabs — Generate MP3 Audio"
- "Gmail — Send Audio Newsletter to Subscribers" -> "Slack — Post to #product-updates Channel"

Thus flow:

1. GitHub trigger (GitHub — On New Release Published)
2. Code — Parse & Categorise Release Notes
3. Notion — Query Changelog Database (queries for matching version)
4. Code — Merge Notion Context + Release Data
5. OpenAI — Write Podcast-Style Spoken Script
6. ElevenLabs — Generate MP3 Audio (HTTP request)
7. Code — Store Script Text Before Upload (preserves script text)
8. Upload a File (UploadToURL)
9. Code — Store Audio URL + Full Release Data
10. Google Sheets — Load Subscriber List
11. Aggregate — Bundle All Subscriber Emails
12. Code — Build HTML Email Newsletter
13. Gmail — Send Audio Newsletter to Subscribers
14. Slack — Post to #product-updates Channel
15. Notion — Write Back Audio URL to Changelog Page

Thus it's a linear pipeline with no branching.

Now, we need to produce sections:

### 1. Workflow Overview

Summarize purpose, target use cases, logical blocks.

Group into logical blocks: maybe:

- Block 1: Trigger & Parsing (GitHub trigger, code parse)
- Block 2: Notion Enrichment (Notion query, merge)
- Block 3: AI Scriptwriting & Audio Generation (OpenAI, ElevenLabs)
- Block 4: Audio Hosting (UploadToURL)
- Block 5: Subscriber Aggregation (Google Sheets, Aggregate)
- Block 6: Email Newsletter Composition (HTML generation, Gmail)
- Block 7: Internal Notification & Write-Back (Slack, Notion update)

### 2. Block-by-Block Analysis

For each block we need overview, nodes involved, node details with type, config, expressions, connections, version requirements, edge cases, sub-workflow references.

### 3. Summary Table

All nodes with columns: Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note

Need to map sticky notes: there are five sticky notes. Each covering a subset of nodes.

Sticky note 1: "📋 MAIN — Workflow Overview" — overall, maybe covers entire workflow. We'll put that content in all nodes? The instruction says: "If a sticky note visually covers multiple nodes, duplicate its content in each affected row." This sticky note is a general overview, maybe doesn't visually cover a specific node, but as a high-level note it might be considered to cover all nodes. We'll include its content in each node's sticky note column.

Sticky note 2: "📝 Note — GitHub Trigger, Parse & Notion Fetch" — covers nodes: GitHub — On New Release Published, Code — Parse & Categorise Release Notes, Notion — Query Changelog Database. Add that note's content to those nodes.

Sticky note 3: "📝 Note — Script, TTS & UploadToURL" — covers nodes: OpenAI — Write Podcast-Style Spoken Script, ElevenLabs — Generate MP3 Audio, Upload a File. Add its content to those.

Sticky note 4: "📝 Note — Subscribers, HTML Build & Gmail Send" — covers: Google Sheets — Load Subscriber List, Aggregate — Bundle All Subscriber Emails, Code — Build HTML Email Newsletter, Gmail — Send Audio Newsletter to Subscribers.

Sticky note 5: "📝 Note — Slack Post & Notion Write-Back" — covers: Slack — Post to #product-updates Channel, Notion — Write Back Audio URL to Changelog Page.

Thus we include sticky note content for each node that is covered.

But also note the main sticky note "📋 MAIN — Workflow Overview" we can include in each node as well? Or we could note that it's a general overview note for entire workflow. We'll include its content in all nodes for completeness.

### 4. Reproducing the Workflow from Scratch

Step-by-step numbered list to rebuild the workflow, node creation, connections, parameter setup, credential config, default values.

Need to include: each node type and key config, connection order, parameter setup, credentials, environment variables, sub-workflow details (none), default values.

### 5. General Notes & Resources

We'll put any notes: branding, external links, etc. The description text contains "Humanize your software updates. Import this template and start broadcasting your product journey with audio." Could be a note. Also there may be no video link; we can mention the workflow description.

Now we need to write the document in English, professional, structured, not raw JSON, preserve sticky notes content with links.

Let's start building the analysis.

### 1. Workflow Overview

Explain overall purpose: automate transformation of GitHub releases into podcast-style audio announcements, then distribute via email and Slack, enrich with Notion, write back.

List logical blocks: 1) Trigger & Parsing (GitHub trigger, parse), 2) Notion Enrichment (query and merge), 3) AI Scriptwriting (OpenAI), 4) Audio Generation (ElevenLabs), 5) Audio Hosting (UploadToURL), 6) Subscriber Aggregation (Google Sheets, Aggregate), 7) Email Newsletter (HTML builder, Gmail), 8) Internal Sync & Write-Back (Slack, Notion update).

We can mention that it's a linear pipeline (no branching), but there are multiple external services.

### 2. Block-by-Block Analysis

We'll group by block.

#### Block 1: GitHub Trigger & Release Parsing

- Overview: Fires when new release published, parses raw markdown into structured categories (features, fixes, improvements), extracts version, date, etc.
- Nodes Involved: 
  - GitHub — On New Release Published
  - Code — Parse & Categorise Release Notes
  - Notion — Query Changelog Database (maybe part of next block)
  - Code — Merge Notion Context + Release Data (though this merges with Notion data; could be part of next block)

But we can separate Block 1 (GitHub trigger + parsing) and Block 2 (Notion enrichment).

Thus block 1 includes: GitHub Trigger, Code Parse. Block 2 includes: Notion Query, Code Merge.

Let's define block 1:

- Overview: The workflow begins when a new release is published on a monitored GitHub repository. The raw release body is parsed into structured categories, version metadata extracted.
- Nodes Involved:
  - GitHub — On New Release Published (githubTrigger)
  - Code — Parse & Categorise Release Notes (code)
- Node details for each.

#### Block 2: Notion Enrichment & Deduplication

- Overview: Queries a Notion changelog database for a page matching the version tag. Merges Notion context (summary) with parsed release data, checks if audio already published to avoid duplicate runs.
- Nodes Involved:
  - Notion — Query Changelog Database
  - Code — Merge Notion Context + Release Data

#### Block 3: AI Script Generation

- Overview: Sends combined content to OpenAI Chat model to rewrite into a conversational, podcast-style spoken script (60–90 seconds). Node config: Chat model, prompts not in JSON but can be inferred. The node uses the "openAi" type, resource=chat.
- Node: OpenAI — Write Podcast-Style Spoken Script

#### Block 4: Audio Generation & Hosting

- Overview: Sends script text to ElevenLabs TTS API to generate MP3 audio, then uploads the audio to UploadToURL for a public CDN link.
- Nodes: ElevenLabs — Generate MP3 Audio, Code — Store Script Text Before Upload, Upload a File, Code — Store Audio URL + Full Release Data.

#### Block 5: Subscriber List Load & Aggregation

- Overview: Reads subscriber email list from Google Sheets, aggregates all emails into a BCC string.
- Nodes: Google Sheets — Load Subscriber List, Aggregate — Bundle All Subscriber Emails

#### Block 6: Email Newsletter Composition & Delivery

- Overview: Builds a rich HTML email with embedded audio player, changelog summary, CTA button, and sends to all subscribers via Gmail (BCC).
- Nodes: Code — Build HTML Email Newsletter, Gmail — Send Audio Newsletter to Subscribers

#### Block 7: Internal Notification & Notion Write-Back

- Overview: Posts a Slack message with audio preview link to #product-updates channel, then updates the Notion changelog page with audio status and URL.
- Nodes: Slack — Post to #product-updates Channel, Notion — Write Back Audio URL to Changelog Page

Now go into each node detail.

#### Node details for each block

We'll need for each node:
- Type and technical role
- Configuration choices (interpreted, not raw JSON)
- Key expressions or variables used
- Input and output connections
- Version-specific requirements
- Edge cases or potential failure types

Let's go through each node.

1. "GitHub — On New Release Published"
   - Type: n8n-nodes-base.githubTrigger
   - Role: Webhook trigger listening for new GitHub release events.
   - Config: owner = YOUR_GITHUB_ORG_OR_USERNAME, repository = YOUR_REPO_NAME, events = ["release"].
   - Output: JSON payload containing action, release details.
   - Connections: Output -> Code — Parse & Categorise Release Notes
   - Version: typeVersion 1
   - Edge Cases: Webhook must be configured; if action is not "published" it may still fire, filtered by Code node.

2. "Code — Parse & Categorise Release Notes"
   - Type: n8n-nodes-base.code
   - Role: Parse GitHub release body, categorize bullet items into features, fixes, improvements, produce plain text changelog.
   - Config: jsCode defined; filters out non-"published" actions; extracts version, releaseName, releaseUrl, author, publishedAt, releaseDate; parses lines; categorizes based on headings; truncates plain text to 2000 chars.
   - Key variables: version, releaseName, releaseUrl, author, publishedAt, releaseDate, repoName, plainChangelog, features, fixes, improvements, other, totalChanges
   - Input: GitHub trigger output
   - Output: Structured JSON item
   - Connections: -> Notion — Query Changelog Database
   - Version: typeVersion 2

3. "Notion — Query Changelog Database"
   - Type: n8n-nodes-base.notion
   - Role: Query Notion database for a page matching version tag.
   - Config: resource=databasePage, operation=getAll, databaseId = YOUR_NOTION_DATABASE_ID, filter: Version equals (likely to use variable $json.version). limit=1, options downloadFiles=false.
   - Input: output from Parse node (the version field used in filter)
   - Output: Notion page object or empty.
   - Connections: -> Code — Merge Notion Context + Release Data
   - Version: typeVersion 2.2

4. "Code — Merge Notion Context + Release Data"
   - Type: n8n-nodes-base.code
   - Role: Merge release data with Notion page context; check if audio already published; if so, returns empty array to stop processing.
   - Config: uses $input.first().json (Notion page) and $('Code — Parse & Categorise Release Notes').item.json; extracts notionPageId, notionSummary, audioAlreadyPublished; if audioAlreadyPublished true, returns [] (stops).
   - Input: Notion query output and previous release data.
   - Output: Combined JSON with notionPageId, notionSummary plus releaseData fields.
   - Connections: -> OpenAI — Write Podcast-Style Spoken Script
   - Version: typeVersion 2

5. "OpenAI — Write Podcast-Style Spoken Script"
   - Type: @n8n/n8n-nodes-langchain.openAi
   - Role: Use OpenAI Chat model to rewrite combined content into a conversational spoken script.
   - Config: resource=chat. Likely uses a prompt stored elsewhere; parameters not shown; likely messages with system and user prompts.
   - Input: JSON with merged content.
   - Output: Chat completion response.
   - Connections: -> ElevenLabs — Generate MP3 Audio
   - Version: typeVersion 1.4

6. "ElevenLabs — Generate MP3 Audio"
   - Type: n8n-nodes-base.httpRequest
   - Role: HTTP POST to ElevenLabs TTS endpoint to generate audio.
   - Config: URL = "https://api.elevenlabs.io/v1/text-to-speech/YOUR_ELEVENLABS_VOICE_ID", method POST, body JSON with text from AI response, model_id, voice_settings; headers include xi-api-key; Accept=audio/mpeg; response as file; timeout 60s.
   - Input: Output from OpenAI node; uses $json.choices[0].message.content as script text.
   - Output: Binary MP3 file.
   - Connections: -> Code — Store Script Text Before Upload
   - Version: typeVersion 4.2

7. "Code — Store Script Text Before Upload"
   - Type: n8n-nodes-base.code
   - Role: Extract spoken script text from AI response before binary overwrites context; preserve release data.
   - Config: gets AI response from OpenAI node; gets releaseData from Merge node; passes through binary and json.
   - Input: Binary MP3 from ElevenLabs, plus context.
   - Output: json with releaseData + spokenScript; binary remains.
   - Connections: -> Upload a File
   - Version: typeVersion 2

8. "Upload a File"
   - Type: n8n-nodes-uploadtourl.uploadToUrl
   - Role: Upload MP3 binary to a hosting service to get a permanent public URL.
   - Config: default, credentials required for uploadtourl API (id "credential-id", name "uploadtourl - Deepanshi").
   - Input: Binary from previous node.
   - Output: JSON with hosted file URL.
   - Connections: -> Code — Store Audio URL + Full Release Data
   - Version: typeVersion 1

9. "Code — Store Audio URL + Full Release Data"
   - Type: n8n-nodes-base.code
   - Role: Extract audio URL from upload response; combine with release data.
   - Config: reads uploadResp fields for url; merges with releaseData from previous node.
   - Input: Upload response JSON.
   - Output: json with all release data fields + audioUrl.
   - Connections: -> Google Sheets — Load Subscriber List
   - Version: typeVersion 2

10. "Google Sheets — Load Subscriber List"
   - Type: n8n-nodes-base.googleSheets
   - Role: Read rows from Google Sheets "Subscribers" sheet.
   - Config: documentId = YOUR_GOOGLE_SHEET_ID, sheetName = "Subscribers".
   - Input: None (triggered by previous node passing data)
   - Output: Items for each subscriber row.
   - Connections: -> Aggregate — Bundle All Subscriber Emails
   - Version: typeVersion 4.5

11. "Aggregate — Bundle All Subscriber Emails"
   - Type: n8n-nodes-base.aggregate
   - Role: Collect all subscriber items into a single item with an array field "subscribers".
   - Config: aggregateAllItemData, destinationFieldName = "subscribers".
   - Input: Items from Google Sheets.
   - Output: One item with subscribers array.
   - Connections: -> Code — Build HTML Email Newsletter
   - Version: typeVersion 1

12. "Code — Build HTML Email Newsletter"
   - Type: n8n-nodes-base.code
   - Role: Generate HTML email body with audio player, changelog table, CTA button; extract BCC list.
   - Config: jsCode constructs htmlBody string, also extracts bccList, emailSubject. Uses releaseData from earlier node, aggregated subscribers.
   - Input: aggregated subscriber list, plus release data from Code — Store Audio URL + Full Release Data.
   - Output: json with htmlBody, bccList, emailSubject plus release data.
   - Connections: -> Gmail — Send Audio Newsletter to Subscribers
   - Version: typeVersion 2

13. "Gmail — Send Audio Newsletter to Subscribers"
   - Type: n8n-nodes-base.gmail
   - Role: Send email via Gmail with HTML body.
   - Config: sendTo = "newsletter@yourdomain.com", subject = {{ $json.emailSubject }}, message = {{ $json.htmlBody }}, options: bccList = {{ $json.bccList }}, replyTo = "user@example.com", senderName = "Product Team", appendAttribution = false.
   - Input: json with email fields.
   - Output: send result.
   - Connections: -> Slack — Post to #product-updates Channel
   - Version: typeVersion 2.1

14. "Slack — Post to #product-updates Channel"
   - Type: n8n-nodes-base.slack
   - Role: Post Slack message to channel.
   - Config: select = channel, channelId = YOUR_SLACK_CHANNEL_ID (cachedResultName "#product-updates"), otherOptions: unfurl_links=false, unfurl_media=false. The message content is likely default or defined elsewhere (maybe in node config but not shown; could use default text or block kit). We'll note that.
   - Input: from Gmail node.
   - Output: Slack post result.
   - Connections: -> Notion — Write Back Audio URL to Changelog Page
   - Version: typeVersion 2.2

15. "Notion — Write Back Audio URL to Changelog Page"
   - Type: n8n-nodes-base.notion
   - Role: Update the Notion changelog page with audio status and URL.
   - Config: operation=update, possibly pageId from notionPageId, updates properties: AudioStatus=Published, AudioURL, AudioPublishedAt. However configuration not shown; we assume these fields are set via node parameters.
   - Input: from Slack node.
   - Output: Update confirmation.
   - Connections: none after (end).
   - Version: typeVersion 2.2

Now summarizing sticky notes.

Sticky notes:

1. "📋 MAIN — Workflow Overview" content is provided. It covers entire architecture, setup requirements.

2. "📝 Note — GitHub Trigger, Parse & Notion Fetch" content.

3. "📝 Note — Script, TTS & UploadToURL" content.

4. "📝 Note — Subscribers, HTML Build & Gmail Send" content.

5. "📝 Note — Slack Post & Notion Write-Back" content.

We need to map these notes to nodes.

Note 2 covers nodes: GitHub — On New Release Published, Code — Parse & Categorise Release Notes, Notion — Query Changelog Database.

Note 3 covers nodes: OpenAI — Write Podcast-Style Spoken Script, ElevenLabs — Generate MP3 Audio, Upload a File.

Note 4 covers: Google Sheets — Load Subscriber List, Aggregate — Bundle All Subscriber Emails, Code — Build HTML Email Newsletter, Gmail — Send Audio Newsletter to Subscribers.

Note 5 covers: Slack — Post to #product-updates Channel, Notion — Write Back Audio URL to Changelog Page.

Main overview note covers all nodes.

Now create the summary table. It should list each node with Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note.

Functional Role: we need a short description for each node.

Let's fill each.

1. GitHub — On New Release Published (githubTrigger) – Entry point; triggers on new release.

2. Code — Parse & Categorise Release Notes (code) – Parse release body; categorize changes.

3. Notion — Query Changelog Database (notion) – Query Notion for matching version.

4. Code — Merge Notion Context + Release Data (code) – Merge Notion page data and release data; check duplication.

5. OpenAI — Write Podcast-Style Spoken Script (openAi) – Generate spoken script.

6. ElevenLabs — Generate MP3 Audio (httpRequest) – Convert script to MP3.

7. Code — Store Script Text Before Upload (code) – Preserve script text before binary overrides.

8. Upload a File (uploadToUrl) – Host MP3 file.

9. Code — Store Audio URL + Full Release Data (code) – Extract hosted audio URL and merge.

10. Google Sheets — Load Subscriber List (googleSheets) – Retrieve subscriber emails.

11. Aggregate — Bundle All Subscriber Emails (aggregate) – Aggregate emails into single list.

12. Code — Build HTML Email Newsletter (code) – Generate HTML email with audio embed.

13. Gmail — Send Audio Newsletter to Subscribers (gmail) – Send newsletter email.

14. Slack — Post to #product-updates Channel (slack) – Post update to Slack.

15. Notion — Write Back Audio URL to Changelog Page (notion) – Update Notion page with audio status.

Now Input/Output connections:

- GitHub — On New Release Published: Input: none (trigger). Output: Code — Parse & Categorise Release Notes.

- Code — Parse & Categorise Release Notes: Input: GitHub — On New Release Published. Output: Notion — Query Changelog Database.

- Notion — Query Changelog Database: Input: Code — Parse & Categorise Release Notes. Output: Code — Merge Notion Context + Release Data.

- Code — Merge Notion Context + Release Data: Input: Notion — Query Changelog Database. Output: OpenAI — Write Podcast-Style Spoken Script.

- OpenAI — Write Podcast-Style Spoken Script: Input: Code — Merge Notion Context + Release Data. Output: ElevenLabs — Generate MP3 Audio.

- ElevenLabs — Generate MP3 Audio: Input: OpenAI — Write Podcast-Style Spoken Script. Output: Code — Store Script Text Before Upload.

- Code — Store Script Text Before Upload: Input: ElevenLabs — Generate MP3 Audio. Output: Upload a File.

- Upload a File: Input: Code — Store Script Text Before Upload. Output: Code — Store Audio URL + Full Release Data.

- Code — Store Audio URL + Full Release Data: Input: Upload a File. Output: Google Sheets — Load Subscriber List.

- Google Sheets — Load Subscriber List: Input: Code — Store Audio URL + Full Release Data. Output: Aggregate — Bundle All Subscriber Emails.

- Aggregate — Bundle All Subscriber Emails: Input: Google Sheets — Load Subscriber List. Output: Code — Build HTML Email Newsletter.

- Code — Build HTML Email Newsletter: Input: Aggregate — Bundle All Subscriber Emails. Output: Gmail — Send Audio Newsletter to Subscribers.

- Gmail — Send Audio Newsletter to Subscribers: Input: Code — Build HTML Email Newsletter. Output: Slack — Post to #product-updates Channel.

- Slack — Post to #product-updates Channel: Input: Gmail — Send Audio Newsletter to Subscribers. Output: Notion — Write Back Audio URL to Changelog Page.

- Notion — Write Back Audio URL to Changelog Page: Input: Slack — Post to #product-updates Channel. Output: none (end).

Now sticky notes mapping:

- Main overview sticky note content for all nodes.

- Note 2 for nodes 1,2,3.

- Note 3 for nodes 5,6,8 (and perhaps Code — Store Script Text Before Upload is not covered; but the note says "Script, TTS & UploadToURL". It covers OpenAI, ElevenLabs, UploadToURL. Possibly also Code — Store Script Text Before Upload is part of that flow but not in note. However we can include note 3 for node 5,6,8.

- Note 4 for nodes 10,11,12,13.

- Note 5 for nodes 14,15.

Better to include note contents in rows.

Now we need to produce the final document.

Proceed to write.

Also check if any environment variables are required.

The description mentions environment variables: NOTION_DATABASE_ID, APPROVER_EMAIL, DISCORD_WEBHOOK_URL, SLACK_CHANNEL_ID. However nodes include static fields referencing YOUR_NOTION_DATABASE_ID, YOUR_SLACK_CHANNEL_ID. We'll document these as placeholders that need environment variables or config.

Now write sections.

Will include details for each node.

Potential edge cases: GitHub webhook may fire for release actions other than "published" (like pre-release, edited), code node filters by action === 'published'. Notion query may return empty results (no matching page) – merge node then uses notionSummary from plainChangelog. If Audio already published, merge node returns empty, stopping workflow. OpenAI may fail due to API limits or key issues. ElevenLabs may fail due to invalid voice ID, timeout, or API key. UploadToURL service may fail; audio URL extraction may need multiple fallback keys. Google Sheets may have no subscribers or missing "Email" column. Gmail may hit sending limits or require OAuth2. Slack may need bot token with chat:write scope and channel ID. Notion update may need pageId to update; may fail if page not found.

Now produce final content.

Ensure not to include raw JSON (except small examples if necessary). Keep output fluent and professional.

Now produce final answer.## 1. Workflow Overview  

**Purpose** – This workflow turns every new GitHub release into a polished, podcast‑style audio announcement and distributes it as a rich HTML newsletter (via Gmail) and an internal Slack message, while writing the result back to a Notion changelog page.  

**Target Use Cases**  

- Product teams that want to surface release notes in an accessible audio format.  
- Marketing or developer‑relations teams that need to broadcast updates to a subscriber list.  
- Organizations that maintain a canonical changelog in Notion and want to keep it in sync with GitHub releases.  

**Logical Blocks**  

| Block | Function | Key Nodes |
|-------|----------|----------|
| **1. GitHub Trigger & Parsing** | Capture the release event, extract metadata, split the markdown into structured categories (features / fixes / improvements). | GitHub Trigger → Code Parse |
| **2. Notion Enrichment & Deduplication** | Look up the corresponding Notion changelog page, merge its richer context, and abort if the audio has already been published. | Notion Query → Code Merge |
| **3. AI Script Generation** | Rewrite the combined text into a natural, spoken‑style script suitable for ~60‑90 s narration. | OpenAI Chat |
| **4. Audio Production & Hosting** | Convert the script to MP3 via ElevenLabs, upload the binary to a CDN, and capture the permanent URL. | ElevenLabs HTTP → Code Store Script → Upload To URL → Code Store URL |
| **5. Subscriber Aggregation** | Pull email addresses from a Google Sheets sheet and bundle them into a BCC list. | Google Sheets → Aggregate |
| **6. Newsletter Composition & Delivery** | Build a fully styled HTML email with an inline audio player and send it through Gmail. | Code HTML → Gmail Send |
| **7. Internal Notification & Write‑Back** | Post a Slack message with the audio link, then update the Notion page (status = Published, audio URL, timestamp). | Slack Post → Notion Update |

The pipeline is strictly linear; every block feeds the next, so a failure in any step halts the entire run.

---

## 2. Block‑by‑Block Analysis  

### Block 1 – GitHub Trigger & Parsing  

**Overview** – The workflow starts when a new release is published on a monitored GitHub repository. The raw release body (markdown) is cleaned, version metadata is extracted, and bullet points are classified into *features*, *fixes*, *improvements* (or “other”).

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **GitHub — On New Release Published** | `githubTrigger` (n8n‑nodes‑base) | Webhook entry point that fires on `release` events. | `owner`: *YOUR_GITHUB_ORG_OR_USERNAME*  <br> `repository`: *YOUR_REPO_NAME*  <br> `events`: `["release"]` | – | No input → JSON payload (`action`, `release`, `repository`) | Webhook not reachable → workflow never runs. Fires for any release action; downstream Code node filters to `published`. |
| **Code — Parse & Categorise Release Notes** | `code` (JS) | Parse the GitHub payload, extract version/date, and split bullet lines into categories. | Inline JS that: <br>• Checks `action === 'published'` (returns empty otherwise) <br>• Strips markdown → plain text (`plainChangelog`) <br>• Classifies lines under headings (`## feat`, `## fix`, …) into `features`, `fixes`, `improvements` <br>• Caps plain text at 2000 chars | `$input.first().json` (GitHub payload) | Output: `{ version, releaseName, releaseUrl, author, publishedAt, releaseDate, repoName, plainChangelog, features, fixes, improvements, other, totalChanges }` | If `action` ≠ `published`, node returns `[]` and the run stops. Empty or malformed markdown → empty category arrays, but workflow continues. |

---

### Block 2 – Notion Enrichment & Deduplication  

**Overview** – The workflow queries a Notion changelog database for a page whose `Version` property matches the release tag. If the page is found, its richer summary is merged. If the page’s `AudioStatus` is already **Published**, the run aborts to avoid duplicate processing.

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **Notion — Query Changelog Database** | `notion` (databasePage getAll) | Look up the Notion page that matches the release version. | `databaseId`: *YOUR_NOTION_DATABASE_ID* (environment variable `NOTION_DATABASE_ID`) <br> `filter`: `Version` equals `{{ $json.version }}` <br> `limit`: 1 | Uses `version` from previous node to build filter. | Input: parsed release data → Output: Notion page object (or empty) | No matching page → `id` undefined; downstream merge node falls back to plain changelog. |
| **Code — Merge Notion Context + Release Data** | `code` (JS) | Combine Notion summary with release data; detect already‑published audio. | Reads `notionResp` and `$('Code — Parse & Categorise Release Notes').item.json`. <br> Extracts `notionPageId`, `notionSummary` (rich text). <br> Checks `AudioStatus === 'Published'`; if true, returns `[]`. | `audioAlreadyPublished` flag; `notionSummary` defaults to first 300 chars of `plainChangelog`. | Input: Notion page + release data → Output: enriched JSON (or empty) | If `AudioStatus` is already “Published”, workflow stops cleanly. Missing `Summary` property → fallback to `plainChangelog`. |

---

### Block 3 – AI Script Generation  

**Overview** – OpenAI’s Chat model receives the merged release content and rewrites it into a warm, conversational script that reads naturally in 60‑90 seconds.

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **OpenAI — Write Podcast‑Style Spoken Script** | `@n8n/n8n‑nodes‑langchain.openAi` (chat) | Generate a podcast‑style spoken script from the merged text. | `resource`: `chat` <br> (Prompt configuration is stored in the node UI – typically a system message like “You are a product‑update narrator…” and a user message containing the release data.) | Consumes `json` from Merge node; produces `choices[0].message.content`. | Input: merged JSON → Output: Chat completion JSON | API rate limits, invalid API key, or model down → node fails. No script returned → downstream TTS node receives empty text. |

---

### Block 4 – Audio Production & Hosting  

**Overview** – The script is turned into an MP3 file by ElevenLabs, then uploaded to a CDN (UploadToURL) so a permanent URL is available for the newsletter and Slack message.

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **ElevenLabs — Generate MP3 Audio** | `httpRequest` (POST) | Call the ElevenLabs TTS endpoint and receive binary MP3 data. | `url`: `https://api.elevenlabs.io/v1/text-to-speech/YOUR_ELEVENLABS_VOICE_ID` <br> `method`: POST <br> `body` (JSON): `{ "text": {{ JSON.stringify($json.choices[0].message.content) }}, "model_id": "eleven_multilingual_v2", "voice_settings": { "stability": 0.5, "similarity_boost": 0.85, "style": 0.2, "use_speaker_boost": true } }` <br> `headers`: `xi-api-key`: *YOUR_ELEVENLABS_API_KEY*, `Content-Type`: `application/json`, `Accept`: `audio/mpeg` <br> `responseFormat`: `file` (binary) <br> `timeout`: 60 s | Uses the script text from OpenAI node. | Input: Chat completion JSON → Output: Binary MP3 (`data`) | Invalid voice ID, API key, or quota exceeded → HTTP error. Timeout if script is very long. |
| **Code — Store Script Text Before Upload** | `code` (JS) | Preserve the spoken script as JSON before the binary MP3 overwrites the item context. | Reads `$('OpenAI — Write Podcast-Style Spoken Script').item.json` and `$('Code — Merge Notion Context + Release Data').item.json`. <br> Returns `{ json: { …releaseData, spokenScript }, binary: $input.first().binary }`. | Keeps `spokenScript` field for downstream reference. | Input: MP3 binary + previous JSON → Output: JSON with `spokenScript` + binary unchanged. | Missing `spokenScript` → downstream HTML may lack transcript. |
| **Upload a File** | `uploadToUrl` (n8n‑nodes‑uploadtourl) | Host the MP3 binary and return a public URL. | No extra parameters; relies on credential `uploadToUrlApi` (named “uploadtourl – Deepanshi”). | – | Input: binary MP3 → Output: JSON containing `url`, `file`, `link` or similar fields. | Upload service down, credential expired → node fails. |
| **Code — Store Audio URL + Full Release Data** | `code` (JS) | Extract the hosted URL from the upload response and merge it with the full release context. | Looks for `uploadResp.url`, `uploadResp.data.url`, `uploadResp.file.url`, `uploadResp.link`. <br> Returns `{ json: { …releaseData, audioUrl } }`. | `audioUrl` becomes the permanent link for the newsletter and Slack. | Input: Upload response → Output: enriched JSON (adds `audioUrl`). | Unexpected response shape → `audioUrl` empty, causing broken embed. |

---

### Block 5 – Subscriber Aggregation  

**Overview** – All subscriber emails are pulled from a Google Sheets sheet and aggregated into a single comma‑separated string that will be used as the BCC list for Gmail.

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **Google Sheets — Load Subscriber List** | `googleSheets` (read) | Read every row from the “Subscribers” sheet. | `documentId`: *YOUR_GOOGLE_SHEET_ID* <br> `sheetName`: `Subscribers` <br> (Column header must be `Email`). | – | Input: none (triggered by previous node) → Output: one item per row, each containing at least `Email`. | Empty sheet → no subscribers → BCC list empty. Missing `Email` column → node error. |
| **Aggregate — Bundle All Subscriber Emails** | `aggregate` (aggregateAllItemData) | Collect all rows into a single item containing a `subscribers` array. | `destinationFieldName`: `subscribers` | – | Input: multiple subscriber items → Output: single item `{ subscribers: […] }`. | If the sheet is empty, `subscribers` will be an empty array. |

---

### Block 6 – Newsletter Composition & Delivery  

**Overview** – A rich HTML email is generated, embedding an `<audio>` tag pointing to the hosted MP3, a changelog summary table, and a CTA button. The email is sent via Gmail to all subscribers (BCC).

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **Code — Build HTML Email Newsletter** | `code` (JS) | Assemble the final HTML email and create the BCC list string. | Uses `releaseData` from `Code — Store Audio URL + Full Release Data` and `subscribers` from Aggregate node. <br> Produces `htmlBody`, `bccList` (comma‑separated), `emailSubject`. | `releaseData.features`, `.fixes`, `.improvements` → table rows; `releaseData.audioUrl` → audio source; `releaseData.releaseUrl` → CTA link. | Input: aggregated subscribers + release data → Output: JSON `{ …releaseData, htmlBody, bccList, emailSubject }`. | No subscribers → `bccList` empty (Gmail may send only to “To” address). HTML generation errors if `audioUrl` missing. |
| **Gmail — Send Audio Newsletter to Subscribers** | `gmail` (send) | Deliver the newsletter email. | `sendTo`: `newsletter@yourdomain.com` (static “To”) <br> `subject`: `={{ $json.emailSubject }}` <br> `message`: `={{ $json.htmlBody }}` (HTML) <br> `options.bccList`: `={{ $json.bccList }}` <br> `options.replyTo`: `user@example.com` <br> `options.senderName`: `Product Team` <br> `options.appendAttribution`: `false` | – | Input: JSON from HTML node → Output: Gmail send result. | OAuth2 token expired, daily sending limit exceeded, or Gmail policy violation (bulk BCC). |

---

### Block 7 – Internal Notification & Notion Write‑Back  

**Overview** – A Slack message containing a link to the MP3 and a brief summary is posted to an internal channel, then the Notion changelog page is updated with `AudioStatus = Published`, the audio URL, and a timestamp.

| Node | Type | Functional Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases |
|------|------|----------------|-------------------|------------------------|----------------|------------|
| **Slack — Post to #product‑updates Channel** | `slack` (postMessage) | Broadcast a block‑kit message with the audio link. | `channelId`: *YOUR_SLACK_CHANNEL_ID* (environment variable `SLACK_CHANNEL_ID`) <br> `otherOptions.unfurl_links`: `false` <br> `otherOptions.unfurl_media`: `false` | Message body is defined in the node UI (typically includes `releaseName`, `version`, audio URL, release link). | Input: from Gmail node → Output: Slack API response. | Bot not in channel, token lacks `chat:write` scope. |
| **Notion — Write Back Audio URL to Changelog Page** | `notion` (updatePage) | Update the Notion page that was queried earlier. | `operation`: `update` <br> `pageId`: derived from `notionPageId` stored in earlier node. <br> Property updates: `AudioStatus` → `Published`, `AudioURL` → `$json.audioUrl`, `AudioPublishedAt` → `{{ $now.toISO() }}`. | Uses `notionPageId` from merge node; writes audio URL and status. | Input: from Slack node → Output: Notion update confirmation. | Missing `notionPageId` (no page found) → update fails; property names must match Notion schema. |

---

## 3. Summary Table  

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| 📋 MAIN — Workflow Overview | stickyNote | Overall workflow description & setup | – | – | **All nodes**: “## 🎙️ Product Update Audio Announcements — Changelog to TTS Audio … (full content)” |
| 📝 Note — GitHub Trigger, Parse & Notion Fetch | stickyNote | Describes GitHub trigger, parsing, and Notion fetch | – | – | GitHub — On New Release Published; Code — Parse & Categorise Release Notes; Notion — Query Changelog Database |
| 📝 Note — Script, TTS & UploadToURL | stickyNote | Describes OpenAI script generation, ElevenLabs TTS, UploadToURL | – | – | OpenAI — Write Podcast‑Style Spoken Script; ElevenLabs — Generate MP3 Audio; Upload a File |
| 📝 Note — Subscribers, HTML Build & Gmail Send | stickyNote | Describes subscriber loading, email composition, Gmail send | – | – | Google Sheets — Load Subscriber List; Aggregate — Bundle All Subscriber Emails; Code — Build HTML Email Newsletter; Gmail — Send Audio Newsletter to Subscribers |
| 📝 Note — Slack Post & Notion Write‑Back | stickyNote | Describes Slack notification and Notion update | – | – | Slack — Post to #product‑updates Channel; Notion — Write Back Audio URL to Changelog Page |
| GitHub — On New Release Published | githubTrigger | Webhook that fires on release events | – | Code — Parse & Categorise Release Notes | **📝 Note — GitHub Trigger, Parse & Notion Fetch** |
| Code — Parse & Categorise Release Notes | code | Parses release body and categorises changes | GitHub — On New Release Published | Notion — Query Changelog Database | **📝 Note — GitHub Trigger, Parse & Notion Fetch** |
| Notion — Query Changelog Database | notion (databasePage getAll) | Retrieves matching Notion changelog page | Code — Parse & Categorise Release Notes | Code — Merge Notion Context + Release Data | **📝 Note — GitHub Trigger, Parse & Notion Fetch** |
| Code — Merge Notion Context + Release Data | code | Merges Notion summary with release data; aborts if audio already published | Notion — Query Changelog Database | OpenAI — Write Podcast‑Style Spoken Script | – |
| OpenAI — Write Podcast‑Style Spoken Script | @n8n/n8n-nodes-langchain.openAi | Generates a conversational script from merged content | Code — Merge Notion Context + Release Data | ElevenLabs — Generate MP3 Audio | **📝 Note — Script, TTS & UploadToURL** |
| ElevenLabs — Generate MP3 Audio | httpRequest (POST) | Calls ElevenLabs TTS API to produce MP3 binary | OpenAI — Write Podcast‑Style Spoken Script | Code — Store Script Text Before Upload | **📝 Note — Script, TTS & UploadToURL** |
| Code — Store Script Text Before Upload | code | Preserves script text before binary overwrites item context | ElevenLabs — Generate MP3 Audio | Upload a File | – |
| Upload a File | uploadToUrl | Hosts MP3 binary and returns a public URL | Code — Store Script Text Before Upload | Code — Store Audio URL + Full Release Data | **📝 Note — Script, TTS & UploadToURL** |
| Code — Store Audio URL + Full Release Data | code | Extracts hosted audio URL and merges with release data | Upload a File | Google Sheets — Load Subscriber List | – |
| Google Sheets — Load Subscriber List | googleSheets | Reads subscriber email list from sheet | Code — Store Audio URL + Full Release Data | Aggregate — Bundle All Subscriber Emails | **📝 Note — Subscribers, HTML Build & Gmail Send** |
| Aggregate — Bundle All Subscriber Emails | aggregate | Bundles all subscriber rows into a single item array | Google Sheets — Load Subscriber List | Code — Build HTML Email Newsletter | **📝 Note — Subscribers, HTML Build & Gmail Send** |
| Code — Build HTML Email Newsletter | code | Composes HTML email with embedded audio player, changelog, CTA | Aggregate — Bundle All Subscriber Emails | Gmail — Send Audio Newsletter to Subscribers | **📝 Note — Subscribers, HTML Build & Gmail Send** |
| Gmail — Send Audio Newsletter to Subscribers | gmail | Sends the HTML newsletter to all subscribers (BCC) | Code — Build HTML Email Newsletter | Slack — Post to #product‑updates Channel | **📝 Note — Subscribers, HTML Build & Gmail Send** |
| Slack — Post to #product‑updates Channel | slack | Posts a Slack message with audio link and summary | Gmail — Send Audio Newsletter to Subscribers | Notion — Write Back Audio URL to Changelog Page | **📝 Note — Slack Post & Notion Write‑Back** |
| Notion — Write Back Audio URL to Changelog Page | notion (updatePage) | Updates the Notion page with AudioStatus, AudioURL, AudioPublishedAt | Slack — Post to #product‑updates Channel | – (end) | **📝 Note — Slack Post & Notion Write‑Back** |

---

## 4. Reproducing the Workflow from Scratch  

Below is a step‑by‑step guide to recreate the entire workflow in a new n8n instance.

1. **Create a new workflow** and name it *Product Update Audio Announcements*.

2. **Add Sticky Notes (optional but recommended)**  
   - Insert the **📋 MAIN — Workflow Overview** sticky note (copy the full content from the JSON).  
   - Insert the four descriptive sticky notes in the order they appear, positioning them near the nodes they describe.

3. **GitHub Trigger**  
   - Add a **GitHub Trigger** node (`githubTrigger`).  
   - Set `owner` to your GitHub org/user.  
   - Set `repository` to the repo name.  
   - Set `events` to `["release"]`.  
   - Save and activate the trigger (n8n will register a webhook).

4. **Parse & Categorise Release Notes**  
   - Add a **Code** node (`n8n-nodes-base.code`).  
   - Paste the JavaScript from the JSON (or adapt the logic). Key points:  
     - Filter on `action === 'published'`.  
     - Extract `version`, `releaseName`, `releaseUrl`, `author`, `publishedAt`.  
     - Convert markdown to plain text (`plainChangelog`).  
     - Categorize bullets into `features`, `fixes`, `improvements`, `other`.  
   - Connect **GitHub Trigger → Code**.

5. **Notion – Query Changelog Database**  
   - Add a **Notion** node (`n8n-nodes-base.notion`).  
   - Operation: `getAll` (database pages).  
   - Set `databaseId` to the ID of your Notion changelog DB (use environment variable `NOTION_DATABASE_ID`).  
   - Add a manual filter: `Version` equals `{{ $json.version }}`.  
   - Set `limit` to `1`.  
   - Connect **Code → Notion Query**.

6. **Merge Notion Context + Release Data**  
   - Add another **Code** node.  
   - Logic:  
     - If the Notion page’s `AudioStatus` property equals **Published**, return `[]` (stop).  
     - Otherwise, combine `releaseData` (from step 4) with `notionPageId` and `notionSummary`.  
   - Connect **Notion Query → Merge Code**.

7. **OpenAI – Podcast‑Style Script**  
   - Add an **OpenAI** node (`@n8n/n8n-nodes-langchain.openAi`).  
   - Resource: `chat`.  
   - In the node UI configure:  
     - **System prompt**: “You are a friendly product‑update narrator. Rewrite the following release notes into a 60‑90 second spoken script. Avoid jargon, markdown, or bullet points.”  
     - **User prompt**: reference the combined data (`{{ $json.notionSummary }} … {{ $json.plainChangelog }}`).  
   - Connect **Merge Code → OpenAI**.

8. **ElevenLabs – Text‑to‑Speech**  
   - Add an **HTTP Request** node.  
   - Method: `POST`.  
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/YOUR_ELEVENLABS_VOICE_ID`.  
   - Body (JSON):  
     ```json
     {
       "text": "{{ JSON.stringify($json.choices[0].message.content) }}",
       "model_id": "eleven_multilingual_v2",
       "voice_settings": {
         "stability": 0.5,
         "similarity_boost": 0.85,
         "style": 0.2,
         "use_speaker_boost": true
       }
     }
     ```  
   - Headers:  
     - `xi-api-key`: your ElevenLabs API key.  
     - `Content-Type`: `application/json`.  
     - `Accept`: `audio/mpeg`.  
   - Response Format: `File` (binary).  
   - Timeout: 60000 ms.  
   - Connect **OpenAI → ElevenLabs**.

9. **Store Script Text Before Upload**  
   - Add a **Code** node.  
   - Logic: Pull the script text from the OpenAI node (`$('OpenAI — Write Podcast‑Style Spoken Script').item.json`) and keep the binary from ElevenLabs (`$input.first().binary`). Return `{ json: { …releaseData, spokenScript }, binary }`.  
   - Connect **ElevenLabs → Code**.

10. **Upload Audio to CDN**  
    - Add an **Upload to URL** node (`n8n-nodes-uploadtourl.uploadToUrl`).  
    - Credentials: create an UploadToURL API credential (name it *uploadtourl – Deepanshi* or any name you prefer).  
    - No extra parameters needed; the node sends the binary from the previous node and returns a JSON containing the public URL.  
    - Connect **Code (step 9) → Upload**.

11. **Store Audio URL & Full Release Data**  
    - Add another **Code** node.  
    - Logic: Extract the hosted URL from the upload response (`url`, `data.url`, `file.url`, or `link`) and merge it with the release data from step 9.  
    - Connect **Upload → Code**.

12. **Load Subscriber List from Google Sheets**  
    - Add a **Google Sheets** node.  
    - Operation: `read` (default).  
    - Document ID: your Google Sheet ID (`YOUR_GOOGLE_SHEET_ID`).  
    - Sheet Name: `Subscribers`.  
    - Ensure the column header is `Email`.  
    - Connect **Code (step 11) → Google Sheets**.

13. **Aggregate Subscriber Emails**  
    - Add an **Aggregate** node.  
    - Settings: `aggregateAllItemData`, destination field `subscribers`.  
    - Connect **Google Sheets → Aggregate**.

14. **Build HTML Email Newsletter**  
    - Add a **Code** node.  
    - Logic:  
      - Build `bccList` from `subscribers[].Email`.  
      - Construct `htmlBody` with `<audio>`, changelog table, CTA button, footer.  
      - Set `emailSubject` to `🎙️ [${version}] ${releaseName} — Listen to the Update`.  
    - Connect **Aggregate → Code**.

15. **Send Newsletter via Gmail**  
    - Add a **Gmail** node.  
    - `To`: a static address (e.g., `newsletter@yourdomain.com`).  
    - `Subject`: `{{ $json.emailSubject }}`.  
    - `Message`: `{{ $json.htmlBody }}` (HTML).  
    - Options:  
      - `bccList`: `{{ $json.bccList }}`.  
      - `replyTo`: your email.  
      - `senderName`: `Product Team`.  
      - `appendAttribution`: `false`.  
    - Authenticate with a Gmail OAuth2 credential.  
    - Connect **Code (step 14) → Gmail**.

16. **Post to Slack**  
    - Add a **Slack** node.  
    - Channel: `#product-updates` (provide `YOUR_SLACK_CHANNEL_ID`).  
    - Compose a message (in the node UI) that includes: version, release name, audio URL (`{{ $json.audioUrl }}`), and link to the GitHub release.  
    - Set `unfurl_links` and `unfurl_media` to `false`.  
    - Authenticate with a Slack Bot token (`chat:write` scope).  
    - Connect **Gmail → Slack**.

17. **Update Notion Changelog Page**  
    - Add a **Notion** node.  
    - Operation: `update`.  
    - Page ID: reference the `notionPageId` stored earlier (you may need to pass it through the data flow; add it to the JSON from step 6).  
    - Property updates:  
      - `AudioStatus` → `Published`.  
      - `AudioURL` → `{{ $json.audioUrl }}`.  
      - `AudioPublishedAt` → `{{ $now.toISO() }}`.  
    - Authenticate with the same Notion integration token used in step 5.  
    - Connect **Slack → Notion Update**.

18. **Activate the Workflow**  
    - Verify all connections, credentials, and placeholders (`YOUR_*`).  
    - Activate the workflow (GitHub webhook will be registered).  
    - Test by creating a dummy release on your repo.

**Required Credentials**  

| Credential | Service | Required Scopes / Details |
|------------|---------|----------------------------|
| GitHub Personal Access Token | GitHub | `repo` (read) |
| Notion Integration Token | Notion | Share the changelog database with the integration |
| OpenAI API Key | OpenAI | Chat completion access |
| ElevenLabs API Key | ElevenLabs | Text‑to‑Speech access |
| UploadToURL API Key | UploadToURL | File hosting |
| Google Sheets OAuth2 | Google Sheets | Read access to the `Subscribers` sheet |
| Gmail OAuth2 | Gmail | Send emails |
| Slack Bot Token | Slack | `chat:write` scope, added to `#product-updates` |

**Environment Variables** (set in n8n → Settings → Environment)

| Variable | Purpose |
|----------|---------|
| `NOTION_DATABASE_ID` | ID of the Notion changelog database (used in step 5) |
| `APPROVER_EMAIL` | (Optional) email for approval notifications |
| `DISCORD_WEBHOOK_URL` | (Optional) if you later add Discord notifications |
| `SLACK_CHANNEL_ID` | Channel ID for the Slack post (step 16) |

---

## 5. General Notes & Resources  

| Note Content | Context or Link |
|--------------|-----------------|
| **Humanize your software updates. Import this template and start broadcasting your product journey with audio.** | Marketing copy from the workflow description. |
| **ElevenLabs voice IDs** – Retrieve your preferred voice ID from <https://elevenlabs.io/voices>. | Required for the TTS step. |
| **UploadToURL service** – Ensure your UploadToURL instance is reachable and returns a JSON field `url` (or `data.url`, `file.url`, `link`) that the workflow expects. | Hosting provider for the MP3 file. |
| **Gmail sending limits** – Gmail enforces a daily sending quota (~500 for consumer accounts, ~2000 for Workspace). Consider using a dedicated newsletter service for large lists. | Important for scalability. |
| **Notion property names** – The workflow assumes properties named `Version`, `Summary`, `AudioStatus`, `AudioURL`, `AudioPublishedAt`. Adjust the node parameters if your Notion schema differs. | Customization note. |
| **Slack message formatting** – The Slack node’s message body is defined in the node UI (not in JSON). Include the audio URL and release link for a rich preview. | Implementation detail. |
| **Error handling** – The workflow has an early exit in the “Merge Notion Context” node if `AudioStatus` is already `Published`. No other explicit error branches exist; any node failure will stop the run. Consider adding an Error Trigger node for resilience. | Operational tip. |
| **Data privacy** – Subscriber emails are processed only as BCC entries. No email is stored beyond the single send. Ensure compliance with GDPR/CCPA when managing the subscriber sheet. | Legal reminder. |
| **Audio duration** – The OpenAI prompt targets a 60‑90 second script. If your release notes are lengthy, the script may exceed that window; adjust the prompt or the `plainChangelog` truncation (currently 2000 chars) accordingly. | Content tuning note. |