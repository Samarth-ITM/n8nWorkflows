Create and approve LinkedIn posts from Google Sheets with GPT-5.1 and Unsplash

https://n8nworkflows.xyz/workflows/create-and-approve-linkedin-posts-from-google-sheets-with-gpt-5-1-and-unsplash-12325


# Create and approve LinkedIn posts from Google Sheets with GPT-5.1 and Unsplash

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** Create and approve LinkedIn posts from Google Sheets with GPT-5.1 and Unsplash

**Purpose:**  
Automatically pick one “Ready” content idea from Google Sheets on a schedule, generate a LinkedIn-ready post with OpenAI (GPT‑5.1), enforce basic quality rules, email an HTML preview for approval, then publish to LinkedIn (with or without an image). Image handling supports either a user-provided image URL (converted to a direct download link) or an AI-selected Unsplash image.

**Target use cases:**
- Teams preparing LinkedIn post ideas in a sheet and publishing with a human approval gate.
- Semi-automated content pipelines that still require explicit confirmation before posting.
- Image-enhanced posts where either a custom image is supplied or Unsplash is used.

### 1.1 Input Reception & Scheduling
Runs daily (default) and fetches rows with **Status = Ready**, then limits processing to a single row.

### 1.2 Status Management (In Progress)
Immediately marks the chosen row as **In Progress** to avoid reprocessing.

### 1.3 AI Post Generation & Quality Gate
Generates structured JSON content (post + hashtags + image query), formats it into a final LinkedIn post string, checks constraints (≤3000 chars, ≥3 hashtags). If it fails, it loops back to regenerate.

### 1.4 Email Preview Formatting & Approval Gate
Converts the post into Gmail-compatible HTML, emails a preview, then sends an approval email using Gmail “send and wait”.

### 1.5 Image Handling (Optional)
If “Include Image?” is Yes:
- If a custom image URL exists → convert share URL → download file
- Else → search Unsplash → extract descriptions/links → use AI to pick best image → download file

### 1.6 Publishing & Final Status Update
Publishes text-only or image post to LinkedIn, then updates sheet row to **Posted**. If rejected, updates row to **Rejected**.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Scheduling
**Overview:** Triggers on a schedule, queries Google Sheets for “Ready” rows, and restricts execution to one item per run.  
**Nodes involved:** `Schedule Trigger`, `Get Post Idea`, `Limit to One Post`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration (interpreted):** Runs on a schedule defined by the node’s `rule.interval` (template indicates default daily-at-midnight intent via sticky note, but actual interval object is generic).
- **Outputs:** To `Get Post Idea`.
- **Edge cases / failures:**
  - Misconfigured schedule timezone vs desired posting cadence.
  - Node disabled workflow (`active: false`) means it won’t run until activated.

#### Node: Get Post Idea
- **Type / role:** `n8n-nodes-base.googleSheets` — reads rows from a sheet using a filter.
- **Configuration:**
  - Document: Google Sheet ID `1qUir1c-_ScMnoYVoQ0W41nsv5IpLW6rjK8HUNqvNnAg`
  - Sheet tab: `Automate LinkedIn Social Media Content Creation with AI - Gsheet` (gid-like value `1500824165`)
  - Filter: `Status` equals `"Ready"`
- **Outputs:** Rows matching filter to `Limit to One Post`.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - No matching rows → downstream nodes may receive 0 items (workflow effectively does nothing).
  - Permissions / OAuth expiry.
  - Column name mismatch (must exactly match “Status”).

#### Node: Limit to One Post
- **Type / role:** `n8n-nodes-base.limit` — reduces item count to 1.
- **Configuration:** Default limit behavior (no explicit parameters shown).
- **Inputs:** From `Get Post Idea`.
- **Outputs:** Single row to `Update Post Status`.
- **Edge cases:**
  - If Google Sheets returns items in unexpected order, you may not process the “oldest” or “highest priority” post first.

**Sticky note coverage (context):**
- **“Input & Scheduling”**: Runs daily, fetches one Ready post, sets In Progress; highlights sustainability vs flexibility.

---

### Block 2 — Status Management (In Progress)
**Overview:** Marks the chosen row as “In Progress” using its `row_number`, preventing double-processing.  
**Nodes involved:** `Update Post Status`

#### Node: Update Post Status
- **Type / role:** `n8n-nodes-base.googleSheets` — update operation keyed by `row_number`.
- **Configuration:**
  - Operation: **Update**
  - Matching column: `row_number`
  - Writes: `Status = "In Progress"`
  - Uses expression: `row_number = {{ $json.row_number }}`
- **Inputs:** From `Limit to One Post`.
- **Outputs:** Updated row data to `AI Content Generation`.
- **Credentials:** Google Sheets OAuth2.
- **Edge cases / failures:**
  - Missing `row_number` (if sheet node isn’t configured to include it).
  - Concurrent edits in the sheet causing row mismatch.
  - Protected sheet/range preventing updates.

---

### Block 3 — AI Post Generation & Quality Gate
**Overview:** Uses GPT‑5.1 to generate structured LinkedIn content, formats it, validates quality constraints, and loops to regenerate if validation fails.  
**Nodes involved:** `OpenAI Chat Model`, `Structured Output Parser`, `AI Content Generation`, `Post Formatter`, `Validate Post Quality`

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for the agent.
- **Configuration:**
  - Model: `gpt-5.1`
  - Options: default
- **Connections:** Feeds as `ai_languageModel` into `AI Content Generation`.
- **Credentials:** OpenAI API.
- **Edge cases / failures:**
  - Model availability / account access.
  - Rate limits, timeouts, or content policy blocks.

#### Node: Structured Output Parser
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces JSON schema-like output.
- **Configuration:** Example schema includes fields such as `post_content`, `hashtags[]`, `character_count`, `engagement_tip`, `call_to_action`, `image_prompt`.
- **Connections:** Feeds as `ai_outputParser` into `AI Content Generation`.
- **Important mismatch note:** The **agent prompt** asks for `"image_query"` but this parser example shows `"image_prompt"`. The workflow later references `output.image_prompt` in the Unsplash node, creating a likely runtime expression issue unless the model outputs `image_prompt` anyway.
- **Edge cases:**
  - Parsing fails if model returns non-JSON, trailing commentary, or wrong keys.

#### Node: AI Content Generation
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates the LinkedIn post as structured JSON.
- **Configuration highlights:**
  - Prompt uses expressions from the sheet row:
    - `Topic/Subject`, `Content Type`, `Tone`, `Target Audience`, `Additional Notes`
    - Source: `$('Limit to One Post').item.json[...]`
  - Strict requirements: 150–300 words optimal, short paragraphs, hook, 3–5 hashtags, CTA, no markdown, no em dash character `—`, avoid clichés/jargon.
  - Output requested as JSON with keys including: `post_content`, `hashtags`, `character_count`, `engagement_tip`, `call_to_action`, `image_query`.
  - `hasOutputParser: true` (paired with `Structured Output Parser`)
- **Inputs:** From `Update Post Status` (row context) and LLM/parser via LangChain connections.
- **Outputs:** To `Post Formatter`.
- **Edge cases / failures:**
  - The prohibition of `—` can be violated by the model; could fail downstream formatting or approval expectations.
  - JSON validity issues, especially with quotes/newlines.
  - The prompt asks for “one word image queries separated by commas” but names field `image_query`; downstream uses `image_prompt` in Unsplash request (likely bug).

#### Node: Post Formatter
- **Type / role:** `n8n-nodes-base.code` — normalizes AI output, builds final post string, computes metrics and quality flags.
- **Configuration (logic):**
  - Reads `$input.all()[0].json.output` and attempts JSON.parse if it’s a string; otherwise uses object directly.
  - Builds:
    - `final_post = post_content + "\n\n" + hashtags.join(" ")`
    - `character_count`, `word_count`, `hashtag_count`
    - `passes_quality_check = (character_count <= 3000) && (hashtags.length >= 3)`
    - `warnings[]`
- **Outputs:** To `Validate Post Quality`.
- **Edge cases / failures:**
  - If `output` is missing or shaped differently, parsing throws “Failed to parse AI output”.
  - If `hashtags` is not an array, `.join` may fail.

#### Node: Validate Post Quality
- **Type / role:** `n8n-nodes-base.if` — quality gate with loopback.
- **Condition:** `{{$json.passes_quality_check}}` is `true`.
- **True path:** To `Format Content to HTML`.
- **False path:** Loops back to `AI Content Generation` to regenerate.
- **Edge cases:**
  - Infinite loop potential if the model consistently fails constraints (add a max retry counter in production).
  - If `passes_quality_check` undefined → strict validation may treat as false and loop forever.

**Sticky note coverage (context):**
- **“AI Content Generation”**: GPT‑5.1 generation, quality checks, loops back on failure.

---

### Block 4 — Email Preview Formatting & Approval Gate
**Overview:** Converts the final LinkedIn post to Gmail-safe HTML, sends a preview email, then sends an approval request email and waits for the user’s decision.  
**Nodes involved:** `OpenAI Chat Model1`, `Format Content to HTML`, `Send Post Preview`, `Send email for confirmation`, `Check Email Approval`, `Update Post Status to Rejected`

#### Node: OpenAI Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — LLM provider for HTML formatting.
- **Configuration:** Model `gpt-4.1-mini`.
- **Connections:** Feeds `Format Content to HTML` via `ai_languageModel`.
- **Edge cases:** Similar OpenAI availability/rate limits.

#### Node: Format Content to HTML
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates HTML email body.
- **Configuration:**
  - Input: `{{ $json.final_post }}` from `Post Formatter` output (via `Validate Post Quality` true path).
  - Requirements: inline CSS only, max width 600px, Gmail compatible, no buttons/links, preserve line breaks, simple layout.
  - Returns only HTML.
- **Outputs:** To `Send Post Preview`.
- **Edge cases:**
  - Model may return extra text; Gmail node will still send but formatting may break expectations.
  - HTML could be malformed; Gmail might sanitize aggressively.

#### Node: Send Post Preview
- **Type / role:** `n8n-nodes-base.gmail` — sends preview email.
- **Configuration:**
  - To: `example_email@gmail.com`
  - Subject: “LinkedIn Post Preview - Ready to Publish”
  - Message body: `LinkedIn Post Preview:\n\n{{ $json.output }}`
  - **Potential issue:** The HTML agent returns HTML, but this node references `{{$json.output}}`. Depending on how the agent outputs data, the HTML might be in a different property (often `output` exists, but verify in executions).
- **Outputs:** To `Send email for confirmation`.
- **Edge cases:**
  - Gmail OAuth scopes, sending limits.
  - If HTML is not placed correctly, you may email raw text with HTML tags.

#### Node: Send email for confirmation
- **Type / role:** `n8n-nodes-base.gmail` — **sendAndWait** approval gate.
- **Configuration:**
  - Operation: `sendAndWait`
  - Approval type: `double` (Approve / Decline)
  - To: `example_email@gmail.com`
  - Subject: “Approve LinkedIn Post for Publishing”
  - Body includes topic from sheet: `{{ $('Limit to One Post').first().json['Topic/Subject'] }}`
- **Outputs:** To `Check Email Approval` with approval payload in `$json.data.approved`.
- **Edge cases:**
  - Webhook/approval callback must be reachable; n8n instance must have public URL configured if required.
  - If user never responds, execution remains waiting (execution retention considerations).

#### Node: Check Email Approval
- **Type / role:** `n8n-nodes-base.if` — routes based on approval.
- **Condition:** `{{ $json.data.approved }}` is `true`.
- **True path:** To `Check Image Preference`.
- **False path:** To `Update Post Status to Rejected`.
- **Edge cases:**
  - If approval payload differs (e.g., node version changes), `$json.data.approved` could be undefined.

#### Node: Update Post Status to Rejected
- **Type / role:** `n8n-nodes-base.googleSheets` — marks rejected items.
- **Configuration:** Update `Status = "Rejected"` matching on `row_number = {{ $('Update Post Status').first().json.row_number }}`.
- **Inputs:** From `Check Email Approval` false path.
- **Outputs:** End of that branch.
- **Edge cases:** Same as other Sheets updates; also depends on `Update Post Status` node output still accessible by expression.

**Sticky note coverage (context):**
- **“Approval Workflow”**: format email, wait for response, approved continues, rejected updates sheet.

---

### Block 5 — Image Handling (Optional)
**Overview:** If images are enabled for the row, the workflow either downloads a user-provided image (after converting share URLs) or searches Unsplash and uses AI to pick the best image, then downloads it as a file for LinkedIn upload.  
**Nodes involved:** `Check Image Preference`, `Validate Image URL`, `Convert Share Link to Download Link`, `Get Image in Unsplash`, `Get Result Descriptions and Links`, `GPT-4.1-mini`, `Structured Output Parser1`, `Choose Image`, `Download Image`

#### Node: Check Image Preference
- **Type / role:** `n8n-nodes-base.if` — decides image vs text-only.
- **Condition:** `{{ $('Get Post Idea').item.json['Include Image?'] }}` equals `"Yes"`.
- **True path:** To `Validate Image URL`.
- **False path:** To `Create a post` (text-only publish).
- **Edge cases:**
  - Column value must exactly match “Yes” (case/spacing differences will bypass images).
  - Depends on `Get Post Idea` item context; if multiple items existed earlier, mismatched item pairing can occur (mitigated by Limit node).

#### Node: Validate Image URL
- **Type / role:** `n8n-nodes-base.if` — checks whether user provided an image link.
- **Condition:** `{{ $('Limit to One Post').first().json['Image link for your post'] }}` is not empty.
- **True path:** To `Convert Share Link to Download Link` (custom image).
- **False path:** To `Get Image in Unsplash` (auto image).
- **Edge cases:** If the cell contains whitespace, “notEmpty” may pass; downstream code trims, but only after assuming existence.

#### Node: Convert Share Link to Download Link
- **Type / role:** `n8n-nodes-base.code` — normalizes common share URLs to direct download URLs.
- **Logic:**
  - Reads `Image link for your post`, trims.
  - Google Drive file link → converts to `https://drive.google.com/uc?export=download&id=...`
  - Dropbox `dl=0` → `dl=1`
  - Imgur page → `i.imgur.com/...jpg`
  - OneDrive/SharePoint → strips query and appends `?download=1`
  - Outputs: `{ original_url, direct_download_url }`
- **Outputs:** To `Download Image`.
- **Edge cases:**
  - If the link type doesn’t match these patterns, output is unchanged and may not be directly downloadable.
  - Drive links that are not `/file/d/...` format won’t convert.
  - Some hosts require auth/cookies; download will fail.

#### Node: Get Image in Unsplash
- **Type / role:** `n8n-nodes-base.httpRequest` — searches Unsplash.
- **Configuration:**
  - URL: `https://api.unsplash.com/search/photos`
  - Query params:
    - `query = {{ $('AI Content Generation').first().json.output.image_prompt }}`
    - `per_page = 10`
  - Header: `Authorization` (must be `Client-ID YOUR_ACCESS_KEY` typically)
  - `onError: continueRegularOutput` (execution continues even on error, but downstream may break if results missing)
- **Important mismatch note:** AI agent requests `image_query`, not `image_prompt`. Unless the model returns `image_prompt`, this expression may be undefined.
- **Outputs:** To `Get Result Descriptions and Links`.
- **Edge cases / failures:**
  - Missing/invalid Authorization header value.
  - Rate limiting by Unsplash.
  - If query is empty/undefined → poor or failed results.
  - Because errors are continued, downstream code may throw if `results` absent.

#### Node: Get Result Descriptions and Links
- **Type / role:** `n8n-nodes-base.code` — extracts only relevant fields for AI selection.
- **Configuration (logic):**
  - Reads `json.results` from Unsplash response.
  - Builds `output.results` as a keyed object:
    - `results[0]..results[9]` each with:
      - `download: item.links.download`
      - `alt_description`
      - `description`
- **Outputs:** To `Choose Image`.
- **Edge cases:**
  - If `results` is undefined (Unsplash error) → `results.forEach` throws.

#### Node: GPT-4.1-mini
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model for choosing the best image.
- **Configuration:** `gpt-4.1-mini`
- **Connections:** Provides `ai_languageModel` to `Choose Image`.

#### Node: Structured Output Parser1
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces `{"download_url": "..."}` output.
- **Connections:** Provides `ai_outputParser` to `Choose Image`.

#### Node: Choose Image
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — selects best Unsplash image.
- **Configuration:**
  - Input post content: `{{ $('AI Content Generation').first().json.output.post_content }}`
  - Input images: `{{ JSON.stringify($json.results, null, 2) }}`
  - Output: JSON `{ "download_url": "[the download URL]" }`
  - Uses structured output parser + model.
- **Outputs:** To `Download Image`.
- **Edge cases:**
  - If image list is not present or is an object keyed by strings, model can still reason, but may output an invalid URL.
  - Parser will fail if model outputs extra text.

#### Node: Download Image
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads image as binary file.
- **Configuration:**
  - URL: `{{ $json.direct_download_url || $json.output.download_url }}`
    - Custom image path provides `direct_download_url`
    - Unsplash AI selection provides `output.download_url`
  - Response format: `file` (binary)
- **Outputs:** To `Create a post with image`.
- **Edge cases:**
  - Host blocks direct download (403/404).
  - File too large or not an image.
  - If `Choose Image` output is not under `output.download_url` (varies by agent output), expression may fail.

**Sticky note coverage (context):**
- **“Image Handling”**: checks custom link; otherwise Unsplash + AI selection.

---

### Block 6 — Publishing & Final Status Update
**Overview:** Publishes either a text-only LinkedIn post or a post with an uploaded image, then marks the sheet row as Posted.  
**Nodes involved:** `Create a post`, `Create a post with image`, `Update Post Status to Posted`

#### Node: Create a post
- **Type / role:** `n8n-nodes-base.linkedIn` — publishes text post.
- **Configuration:**
  - Text: `{{ $('Validate Post Quality').item.json.final_post }}`
  - Person URN/id: `lo747ESwEe` (account-specific)
- **Inputs:** From `Check Image Preference` false branch.
- **Outputs:** To `Update Post Status to Posted`.
- **Credentials:** LinkedIn OAuth2.
- **Edge cases:**
  - LinkedIn API permissions, expired token.
  - Incorrect `person` identifier.
  - LinkedIn content policy / rate limits.

#### Node: Create a post with image
- **Type / role:** `n8n-nodes-base.linkedIn` — publishes image post.
- **Configuration:**
  - Text: `{{ $('Validate Post Quality').item.json.final_post }}`
  - Person: `lo747ESwEe`
  - `shareMediaCategory = IMAGE`
  - (Binary from `Download Image` is expected by the LinkedIn node for media upload.)
- **Inputs:** From `Download Image`.
- **Outputs:** To `Update Post Status to Posted`.
- **Edge cases:**
  - If binary property name isn’t what LinkedIn node expects, media upload fails.
  - Unsupported file type or size.

#### Node: Update Post Status to Posted
- **Type / role:** `n8n-nodes-base.googleSheets` — marks successful publication.
- **Configuration:**
  - Update `Status = "Posted"`
  - Match `row_number = {{ $('Update Post Status').first().json.row_number }}`
- **Inputs:** From both publishing nodes.
- **Outputs:** End.
- **Edge cases:** Same as other sheet updates; ensure `Update Post Status` is still available in execution scope.

**Sticky note coverage (context):**
- **“Publishing”**: post with/without image, update status to Posted.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | scheduleTrigger | Scheduled entry point | — | Get Post Idea | ## Input & Scheduling<br>Runs daily at midnight, fetches one "Ready" post from Google Sheet, updates status to "In Progress"<br><br>## Why This Matters:<br><br>**Daily schedule means:**<br>- ✅ More sustainable API usage (not hitting OpenAI 24 times a day)<br>- ✅ Gives you time to prepare posts in advance<br>- ✅ One consistent posting time<br>- ❌ Less flexible if you want multiple posts per day |
| Get Post Idea | googleSheets | Fetch “Ready” rows | Schedule Trigger | Limit to One Post | ## Input & Scheduling<br>Runs daily at midnight, fetches one "Ready" post from Google Sheet, updates status to "In Progress"<br><br>## Why This Matters:<br><br>**Daily schedule means:**<br>- ✅ More sustainable API usage (not hitting OpenAI 24 times a day)<br>- ✅ Gives you time to prepare posts in advance<br>- ✅ One consistent posting time<br>- ❌ Less flexible if you want multiple posts per day |
| Limit to One Post | limit | Restrict to one post/run | Get Post Idea | Update Post Status | ## Input & Scheduling<br>Runs daily at midnight, fetches one "Ready" post from Google Sheet, updates status to "In Progress"<br><br>## Why This Matters:<br><br>**Daily schedule means:**<br>- ✅ More sustainable API usage (not hitting OpenAI 24 times a day)<br>- ✅ Gives you time to prepare posts in advance<br>- ✅ One consistent posting time<br>- ❌ Less flexible if you want multiple posts per day |
| Update Post Status | googleSheets | Set Status=In Progress | Limit to One Post | AI Content Generation | ## Input & Scheduling<br>Runs daily at midnight, fetches one "Ready" post from Google Sheet, updates status to "In Progress"<br><br>## Why This Matters:<br><br>**Daily schedule means:**<br>- ✅ More sustainable API usage (not hitting OpenAI 24 times a day)<br>- ✅ Gives you time to prepare posts in advance<br>- ✅ One consistent posting time<br>- ❌ Less flexible if you want multiple posts per day |
| OpenAI Chat Model | lmChatOpenAi | LLM for post generation | — | AI Content Generation (ai_languageModel) | ## AI Content Generation<br><br>Generates LinkedIn post with GPT-5.1, performs quality checks (3000 char limit, min 3 hashtags). Loops back if quality check fails. |
| Structured Output Parser | outputParserStructured | Enforce structured JSON output for post | — | AI Content Generation (ai_outputParser) | ## AI Content Generation<br><br>Generates LinkedIn post with GPT-5.1, performs quality checks (3000 char limit, min 3 hashtags). Loops back if quality check fails. |
| AI Content Generation | langchain.agent | Generate post JSON from sheet inputs | Update Post Status; OpenAI Chat Model; Structured Output Parser | Post Formatter | ## AI Content Generation<br><br>Generates LinkedIn post with GPT-5.1, performs quality checks (3000 char limit, min 3 hashtags). Loops back if quality check fails. |
| Post Formatter | code | Build `final_post`, compute QC flags | AI Content Generation | Validate Post Quality | ## AI Content Generation<br><br>Generates LinkedIn post with GPT-5.1, performs quality checks (3000 char limit, min 3 hashtags). Loops back if quality check fails. |
| Validate Post Quality | if | Route pass/fail; fail loops regenerate | Post Formatter | Format Content to HTML (true); AI Content Generation (false) | ## AI Content Generation<br><br>Generates LinkedIn post with GPT-5.1, performs quality checks (3000 char limit, min 3 hashtags). Loops back if quality check fails. |
| OpenAI Chat Model1 | lmChatOpenAi | LLM for HTML formatting | — | Format Content to HTML (ai_languageModel) | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Format Content to HTML | langchain.agent | Convert final post to Gmail-safe HTML | Validate Post Quality; OpenAI Chat Model1 | Send Post Preview | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Send Post Preview | gmail | Send preview email | Format Content to HTML | Send email for confirmation | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Send email for confirmation | gmail (sendAndWait) | Approval gate (Approve/Decline) | Send Post Preview | Check Email Approval | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Check Email Approval | if | Route approved vs rejected | Send email for confirmation | Check Image Preference (true); Update Post Status to Rejected (false) | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Update Post Status to Rejected | googleSheets | Set Status=Rejected | Check Email Approval (false) | — | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Check Image Preference | if | Decide image vs text-only | Check Email Approval (true) | Validate Image URL (true); Create a post (false) | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Validate Image URL | if | Detect custom image URL | Check Image Preference | Convert Share Link to Download Link (true); Get Image in Unsplash (false) | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Convert Share Link to Download Link | code | Convert share link to direct download URL | Validate Image URL (true) | Download Image | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Get Image in Unsplash | httpRequest | Search Unsplash (10 images) | Validate Image URL (false) | Get Result Descriptions and Links | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Get Result Descriptions and Links | code | Extract descriptions + download links | Get Image in Unsplash | Choose Image | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| GPT-4.1-mini | lmChatOpenAi | LLM for choosing best Unsplash image | — | Choose Image (ai_languageModel) | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Structured Output Parser1 | outputParserStructured | Enforce `{download_url}` output | — | Choose Image (ai_outputParser) | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Choose Image | langchain.agent | Pick best image based on post + metadata | Get Result Descriptions and Links; GPT-4.1-mini; Structured Output Parser1 | Download Image | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Download Image | httpRequest | Download selected image as file/binary | Choose Image OR Convert Share Link to Download Link | Create a post with image | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Create a post with image | linkedIn | Publish LinkedIn post with image | Download Image | Update Post Status to Posted | ## Publishing<br><br>Posts to LinkedIn (with/without image based on previous steps), updates sheet status to "Posted" |
| Create a post | linkedIn | Publish LinkedIn post (text-only) | Check Image Preference (false) | Update Post Status to Posted | ## Publishing<br><br>Posts to LinkedIn (with/without image based on previous steps), updates sheet status to "Posted" |
| Update Post Status to Posted | googleSheets | Set Status=Posted | Create a post OR Create a post with image | — | ## Publishing<br><br>Posts to LinkedIn (with/without image based on previous steps), updates sheet status to "Posted" |
| Sticky Note4 | stickyNote | Documentation block | — | — | # Create and approve LinkedIn posts from Google Sheets with GPT-5.1 and Unsplash<br><br>This workflow generates professional LinkedIn posts using AI, intelligently selects relevant images from Unsplash, sends content for email approval, and publishes directly to LinkedIn.<br><br>**How it works:**<br>1. Checks Google Sheet daily at midnight for posts marked "Ready"<br>2. Generates engaging LinkedIn content using GPT-4 with custom tone/audience<br>3. Performs automatic quality checks (character limits, hashtags)<br>4. Searches Unsplash for 10 relevant images based on post content<br>5. Uses AI to intelligently select the best image that matches your post's message and tone<br>6. Formats post as HTML with selected image and sends to your email for approval<br>7. If approved: posts to LinkedIn with AI-selected image<br>8. If rejected: updates sheet status to "Rejected" for review<br>9. Updates sheet status to "Posted" upon successful publication<br><br>**Setup Requirements:**<br>- Create a Google Sheet with columns: Topic/Subject, Content Type, Tone, Target Audience, Additional Notes, Image link for your post (optional), Include Image?, Status<br>- Connect OpenAI API with your API key (used for both content generation and image selection)<br>- Connect Unsplash API with your access key<br>- Connect Gmail OAuth2 and update recipient email in approval node<br>- Connect Google Sheets OAuth2 and select your sheet from dropdown<br>- Connect LinkedIn OAuth2 for posting<br><br>**Scheduling Options:**<br>- Default: Runs daily at midnight<br>- Customizable: Change Schedule Trigger node to run hourly, weekly, or at specific times<br>- The "Limit" node ensures only one post processes per run<br><br>**Sheet Usage:**<br>- Set Status to "Ready" to trigger content generation<br>- Status changes: Ready → In Progress → Rejected (if disapproved) OR Posted (if approved)<br>- Add image URL in "Image link for your post" column to use your own image (optional)<br>- If no image URL provided, AI searches Unsplash and selects the most relevant image<br>- Set "Include Image?" to "No" if you want text-only post<br>- Review "Rejected" posts, edit if needed, and change back to "Ready" to retry<br><br>**Image Selection Process:**<br>- Searches Unsplash using keywords generated based on the linkedIn post.<br>- Retrieves 10 professional, high-quality images<br>- Uses AI to analyze each image's description and relevance<br>- Selects the single best image that enhances your post's message<br>- Ensures professional, LinkedIn-appropriate imagery<br><br>**Note:** Workflow includes approval gate - nothing posts without your explicit email confirmation. All images are sourced from Unsplash's professional library and selected by AI for maximum relevance and engagement. |
| Sticky Note5 | stickyNote | Documentation block | — | — | ## Input & Scheduling<br><br>Runs daily at midnight, fetches one "Ready" post from Google Sheet, updates status to "In Progress"<br><br>## Why This Matters:<br><br>**Daily schedule means:**<br>- ✅ More sustainable API usage (not hitting OpenAI 24 times a day)<br>- ✅ Gives you time to prepare posts in advance<br>- ✅ One consistent posting time<br>- ❌ Less flexible if you want multiple posts per day |
| Sticky Note6 | stickyNote | Documentation block | — | — | ## AI Content Generation<br><br>Generates LinkedIn post with GPT-5.1, performs quality checks (3000 char limit, min 3 hashtags). Loops back if quality check fails. |
| Sticky Note7 | stickyNote | Documentation block | — | — | ## Approval Workflow<br><br>Formats post as HTML email, sends for approval, waits for response. If approved → proceeds to publishing. If rejected → updates sheet status to "Rejected". |
| Sticky Note8 | stickyNote | Documentation block | — | — | ## Image Handling<br><br>Checks if image link exists → converts share links to direct downloads OR generates image using Unsplash and AI to select the images. |
| Sticky Note9 | stickyNote | Documentation block | — | — | ## Publishing<br><br>Posts to LinkedIn (with/without image based on previous steps), updates sheet status to "Posted" |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name: “Create and approve LinkedIn posts from Google Sheets with GPT-5.1 and Unsplash”

2) **Add Schedule Trigger** (`Schedule Trigger`)  
   - Set it to run **daily at midnight** (or your preferred cadence/timezone).

3) **Add Google Sheets node** (`Get Post Idea`)  
   - Resource: Google Sheets (Read/Get Many depending on UI)
   - Select your **Document** and **Sheet**
   - Add a filter: **Status** = `Ready`
   - Ensure the output includes a `row_number` field (n8n Sheets node typically provides it when using “Read” operations).

4) **Add Limit node** (`Limit to One Post`)  
   - Limit: `1`
   - Connect: `Schedule Trigger → Get Post Idea → Limit to One Post`

5) **Add Google Sheets Update node** (`Update Post Status`)  
   - Operation: **Update**
   - Match on: `row_number`
   - Set values:
     - `row_number = {{$json.row_number}}`
     - `Status = In Progress`
   - Connect: `Limit to One Post → Update Post Status`

6) **Add OpenAI Chat Model node** (`OpenAI Chat Model`)  
   - Model: `gpt-5.1`
   - Configure OpenAI credentials (API key).

7) **Add Structured Output Parser** (`Structured Output Parser`)  
   - Define a JSON schema example that matches what you want the agent to return.
   - Important: ensure it matches the field names you will reference later (see note about `image_query` vs `image_prompt`).

8) **Add LangChain Agent node** (`AI Content Generation`)  
   - Prompt: use the sheet fields (Topic/Subject, Content Type, Tone, Target Audience, Additional Notes) via expressions from `Limit to One Post`.
   - Require JSON-only output (post_content, hashtags[], etc.).
   - Enable “Use Output Parser”.
   - Connect:
     - Main: `Update Post Status → AI Content Generation`
     - AI Language Model: `OpenAI Chat Model → AI Content Generation`
     - Output Parser: `Structured Output Parser → AI Content Generation`

9) **Add Code node** (`Post Formatter`)  
   - Paste logic to:
     - Parse `output` from the agent
     - Build `final_post`
     - Compute `passes_quality_check` (≤3000 chars and ≥3 hashtags)
   - Connect: `AI Content Generation → Post Formatter`

10) **Add IF node** (`Validate Post Quality`)  
   - Condition: `passes_quality_check` is true
   - True output → continue
   - False output → regenerate
   - Connect:
     - `Post Formatter → Validate Post Quality`
     - `Validate Post Quality (false) → AI Content Generation` (loopback)

11) **Add OpenAI Chat Model node** (`OpenAI Chat Model1`)  
   - Model: `gpt-4.1-mini` (or preferred)
   - Connect credentials.

12) **Add LangChain Agent node** (`Format Content to HTML`)  
   - Prompt: convert `{{$json.final_post}}` into Gmail-compatible HTML (inline CSS, max width 600px, no buttons/links).
   - Connect:
     - Main: `Validate Post Quality (true) → Format Content to HTML`
     - AI Language Model: `OpenAI Chat Model1 → Format Content to HTML`

13) **Add Gmail node** (`Send Post Preview`)  
   - Operation: Send
   - To: your email
   - Subject: “LinkedIn Post Preview - Ready to Publish”
   - Body: send the HTML returned by previous node (verify whether it’s in `$json.output` or another field; adjust expression accordingly).
   - Connect: `Format Content to HTML → Send Post Preview`

14) **Add Gmail node** (`Send email for confirmation`)  
   - Operation: **Send and Wait**
   - Approval type: “double” (Approve/Decline)
   - To: your email
   - Subject/body: include topic from sheet via expression.
   - Connect: `Send Post Preview → Send email for confirmation`

15) **Add IF node** (`Check Email Approval`)  
   - Condition: `{{$json.data.approved}}` is true
   - True → proceed to publish
   - False → reject in sheet
   - Connect: `Send email for confirmation → Check Email Approval`

16) **Add Google Sheets Update node** (`Update Post Status to Rejected`)  
   - Update by `row_number`
   - Set `Status = Rejected`
   - `row_number = {{ $('Update Post Status').first().json.row_number }}`
   - Connect: `Check Email Approval (false) → Update Post Status to Rejected`

17) **Add IF node** (`Check Image Preference`)  
   - Condition: `Include Image?` equals `Yes` (from the original sheet row)
   - Connect: `Check Email Approval (true) → Check Image Preference`

18) **Text-only publish path**
   - Add LinkedIn node (`Create a post`)
     - Text: `{{ $('Validate Post Quality').item.json.final_post }}`
     - Author/person: set your LinkedIn person identifier
   - Connect: `Check Image Preference (false) → Create a post`

19) **Image publish path (custom image vs Unsplash)**
   - Add IF node (`Validate Image URL`)
     - Condition: `Image link for your post` is not empty
   - Connect: `Check Image Preference (true) → Validate Image URL`

20) **Custom image branch**
   - Add Code node (`Convert Share Link to Download Link`)
     - Implement conversions (Drive/Dropbox/Imgur/OneDrive) and output `direct_download_url`
   - Add HTTP Request node (`Download Image`)
     - URL: `{{ $json.direct_download_url }}`
     - Response: file (binary)
   - Connect: `Validate Image URL (true) → Convert Share Link to Download Link → Download Image`

21) **Unsplash branch**
   - Add HTTP Request node (`Get Image in Unsplash`)
     - GET `https://api.unsplash.com/search/photos`
     - Query: `query = <your AI image query field>`, `per_page = 10`
     - Header Authorization: `Client-ID <UNSPLASH_ACCESS_KEY>`
   - Add Code node (`Get Result Descriptions and Links`) to extract descriptions + download links
   - Add OpenAI Chat Model (`GPT-4.1-mini`) and Structured Output Parser (`Structured Output Parser1`) expecting `{download_url}`
   - Add LangChain Agent (`Choose Image`) using post content + extracted image metadata, returning `{download_url}`
   - Reuse HTTP Request (`Download Image`) with URL from the selected download_url
   - Connect: `Validate Image URL (false) → Get Image in Unsplash → Get Result Descriptions and Links → Choose Image → Download Image`

22) **Add LinkedIn node for image publishing** (`Create a post with image`)
   - Text: `{{ $('Validate Post Quality').item.json.final_post }}`
   - Share media category: IMAGE
   - Ensure it uses the binary from `Download Image` as media upload (configure binary property if needed by node UI).
   - Connect: `Download Image → Create a post with image`

23) **Add Google Sheets Update node** (`Update Post Status to Posted`)
   - Update by `row_number`
   - Set `Status = Posted`
   - `row_number = {{ $('Update Post Status').first().json.row_number }}`
   - Connect:
     - `Create a post → Update Post Status to Posted`
     - `Create a post with image → Update Post Status to Posted`

24) **Credentials to configure**
   - **Google Sheets OAuth2**: access to the spreadsheet.
   - **OpenAI API**: for GPT‑5.1 and GPT‑4.1-mini nodes.
   - **Gmail OAuth2**: sending email + send-and-wait approval.
   - **LinkedIn OAuth2**: permission to post on behalf of the selected user.
   - **Unsplash Access Key**: set in Authorization header (`Client-ID ...`).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow description, setup requirements, and behavior summary (including approval gate and sheet columns) | Provided in sticky note “Create and approve LinkedIn posts from Google Sheets with GPT-5.1 and Unsplash” |
| **Important consistency check:** AI prompt requests `image_query` but Unsplash node uses `output.image_prompt` | Likely needs alignment: either change agent output field to `image_prompt` or update Unsplash query expression to use `image_query` |
| Potential infinite regeneration loop if quality check keeps failing | Add retry counter or fallback behavior in production |
| Unsplash HTTP node uses `onError: continueRegularOutput` | Downstream code should defensively handle missing `results` to avoid runtime exceptions |